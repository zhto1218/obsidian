# Promotion Content Task Scene Images Design

## 背景

当前 promotion 每日内容任务链路已经支持：

- 创建 `content-task`
- 上传素材到任务资产列表
- 对任务素材做图片分析
- 基于热点、案例和已选角度生成平台草稿

前端新增了“选择生成场景 + 补充描述 + 生成多张图片”的交互。该能力不是独立图片中心，而是当前 `content-task` 的补图入口，因此生成结果必须直接进入任务素材链路，而不是停留在临时 URL。

## 目标

为 `content-task` 增加“按场景批量生成图片”接口，满足以下目标：

1. 前端一次传入多个场景和一段公共补充提示词
2. 后端为每个场景生成 1 张图片
3. 生成结果直接保存为当前任务的 assets
4. 前端可以立即拿到新增素材并进入后续分析、排序和生成流程

## 非目标

- 不新增异步任务、轮询状态或回调机制
- 不新增独立图片表
- 不把场景定义持久化到数据库
- 不在本次实现中支持“每个场景单独一段提示词”
- 不修改现有任务分析、平台草稿生成和发布逻辑

## 核心设计结论

### 1. 接口归属 `content-task`

本次能力服务的是当前每日任务素材池，而不是店铺长期素材库。因此接口归属放在：

`/promotion/stores/{store_id}/content-tasks/{task_id}`

而不是放到店铺基础配置接口下。

### 2. 生成结果直接落成 task assets

用户已经明确要求生成结果进入 daily task 后续链路。因此本次不返回“仅供预览的临时图片”，而是统一：

1. 调用即梦生图
2. 把结果上传到 MinIO
3. 写入 `mkt_promotion_content_asset`

这样后续 `analysis`、`assets/reorder`、`generate`、`publish` 都不需要新增分支判断。

### 3. 接口采用同步串行生成

当前页面交互是用户点击“生成 3 张图片”后直接等待结果，数量也较小，因此优先选择同步接口，逐场景串行调用生图服务。

这样做的取舍：

- 优点：前端集成简单，接口语义直接，复用现有资产返回结构
- 缺点：单次请求耗时较长

当前需求规模下，这个取舍是合理的；如果后续扩展到更多张数或需要中断恢复，再演进为异步任务。

### 4. 默认按“整批成功或整批失败”处理

单次请求中的多个场景应视为一次完整操作。为了避免用户点一次按钮后只落入半批素材，本次采用以下原则：

- 所有场景都生成成功并完成上传后，才统一写入数据库
- 任一场景失败，则整个请求返回失败，不写入任何新 asset

外部生图和 MinIO 上传本身不可回滚，因此“整批成功或整批失败”限定在数据库落库层面。即便某些远端图片已生成成功，只要本次请求最终失败，也不会在任务资产中留下半状态。

## API 设计

### Endpoint

`POST /mkt/api/v1/promotion/stores/{store_id}/content-tasks/{task_id}/scene-images`

### Request

```json
{
  "scenes": ["温馨卧室", "舒适客厅", "户外庭院"],
  "prompt": "希望风格更温馨，色调偏暖，画面干净自然"
}
```

### Request 字段

- `scenes: list[str]`
  - 必填
  - 至少 1 个，最多 6 个
  - 每个值去首尾空格后不能为空
  - 允许前端传中文场景名，不要求提前在后端枚举
- `prompt: str | null`
  - 可空
  - 表示整批图片共用的补充描述
  - 建议最大长度限制为 200 字

### Response

```json
{
  "generated_count": 3,
  "assets": [
    {
      "id": "uuid",
      "org_id": "uuid",
      "task_id": "uuid",
      "image_url": "https://cdn.example.com/a.png",
      "sort_order": 2,
      "source_type": "ai_generated",
      "analysis_summary": null,
      "created_at": "2026-04-01T07:00:00",
      "updated_at": "2026-04-01T07:00:00"
    }
  ]
}
```

### Response 约束

- `generated_count` 等于本次成功写入的图片数量
- `assets` 只返回本次新增素材，不返回任务下全部素材
- 每张新增 asset 的 `source_type` 固定为 `ai_generated`

### 错误语义

- `404 Store not found`：店铺不存在或不属于当前组织
- `404 Task not found`：任务不存在或不属于当前店铺
- `409 Task is read-only`：任务已确认，不允许继续补图
- `400`：场景列表为空、超限、包含空字符串，或 prompt 超长
- `503 Scene image generation unavailable`：即梦未配置、调用失败或 MinIO 不可用

## 服务设计

### Controller

在 `backend/app/modules/promotions/content_controller.py` 新增接口，职责保持与现有 controller 一致：

- 读取路径参数与请求体
- 调用 `content_service`
- 将领域异常映射为 HTTP 状态码
- 返回 response model

### Schema

在 `backend/app/modules/promotions/content_schema.py` 新增：

- `PromotionContentTaskSceneImageGenerateRequest`
- `PromotionContentTaskSceneImageGenerateResponse`

复用现有 `PromotionContentAssetPublic` 作为单张素材响应模型，避免重复定义结构。

### Service

在 `backend/app/modules/promotions/content_service.py` 新增同步服务方法，例如：

- `generate_content_task_scene_images(...)`

推荐流程：

1. 校验店铺存在
2. 校验任务存在且归属当前店铺
3. 校验任务可写
4. 校验 `scenes` 与 `prompt`
5. 读取店铺上下文并构造每个场景的完整 prompt
6. 调用即梦生成图片
7. 将返回的 URL 或 base64 上传到 MinIO
8. 统一创建 task assets
9. 返回新增 assets

### 基础设施复用

无需新增新的基础设施服务，直接复用：

- `app.infrastructure.media.jimeng.JimengImageService`
- `app.infrastructure.media.minio_storage.MinioImageStorage`
- `content_repository.create_asset`

## Prompt 设计

每个场景独立生成 1 张图，但会共享同一段公共补充提示词。

提示词应至少包含以下上下文：

- 店铺行业
- 店铺名称
- 城市
- 地址
- 当前场景
- 公共补充提示词

提示词还应加入稳定的风格约束，降低模型跑偏概率，例如：

- 面向本地生活商家营销展示
- 真实感室内或场景摄影风格
- 画面干净、主体明确
- 不要水印、不要文字、不要海报排版、不要拼贴

对于民宿、餐饮、美业等行业，可在服务层基于 `store.industry_type` 追加一句轻量行业语境，但不引入单独模板表。

示意 prompt：

```text
你是本地生活商家营销素材助手。请生成一张适合商家宣传使用的高质量真实感图片。
店铺行业：homestay。
店铺名称：山野小院。
城市：杭州市。
地址：西湖区龙井路 18 号。
当前场景：温馨卧室。
补充描述：希望风格更温馨，色调偏暖，画面干净自然。
要求：真实摄影感，空间整洁，构图自然，不要文字，不要水印，不要拼贴，不要人物特写。
```

## 数据落库设计

### 资产表

不新增表，直接复用现有 `mkt_promotion_content_asset`。

新增素材写入策略：

- `task_id`：当前任务
- `image_url`：MinIO 公网 URL
- `source_type`：`ai_generated`
- `analysis_summary`：默认 `null`
- `sort_order`：追加到现有素材列表末尾

### 排序规则

若任务当前已有 `N` 张未删除素材，则本次新素材的 `sort_order` 依次为：

- `N`
- `N + 1`
- `N + 2`

这样不影响已有素材顺序，也兼容现有 `assets/reorder` 接口。

## 异常处理与日志

### 即梦依赖检查

当 `JIMENG_API_URL` 或 `JIMENG_API_KEY` 未配置时，直接返回 `503`，不进入生成流程。

### MinIO 依赖检查

当 MinIO 未配置、上传失败或返回异常时，返回 `503`。

### 日志要求

日志统一使用 `loguru`，建议 `bind(module="promotions")`，至少记录：

- `store_id`
- `task_id`
- `scene`
- 当前阶段，例如 `generate` / `upload` / `persist`
- 失败原因

不记录完整 prompt，只记录长度或摘要，避免日志过长。

## 测试设计

遵循 TDD，至少补充以下测试：

### API 测试

1. 成功传入多个场景后，返回 `200`，并写入对应数量的新增 assets
2. 任务不存在时返回 `404`
3. 已确认任务调用时返回 `409`
4. `scenes` 为空、包含空字符串或超过 6 个时返回 `400`
5. 生图或上传失败时返回 `503`

### Service 测试

1. prompt 中包含当前场景和公共补充提示词
2. 生成结果按顺序追加 `sort_order`
3. 当任一场景失败时，不创建任何 asset
4. URL 结果与 base64 结果都能被正确上传并落库

## 文档与配置同步

本次能力复用现有配置项，不新增环境变量。需要同步更新：

- `backend/README.md` 中 promotion daily task 接口说明

如果后续发现环境模板缺失 `JIMENG_*` 或 `MINIO_*` 示例，再单独补充 `.env.example`，但不是本次设计的必需前提。

## 实施建议

建议按以下顺序实施：

1. 先补 schema 和 API 测试，验证目标接口不存在或失败
2. 再补 service 测试，固定 prompt 和落库行为
3. 实现 `content_service` 和 `content_controller`
4. 更新 README
5. 跑最小测试与 `ruff`

这样可以保证新增接口沿着现有 `content-task` 主链路落地，而不是绕出一条新的临时图片分支。
