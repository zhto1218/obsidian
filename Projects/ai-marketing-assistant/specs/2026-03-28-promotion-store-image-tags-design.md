# Promotion Store Image Tags Design

## Goal

为推广店铺配置补充“图片生成标签”接口，供前端在店铺已上传菜品/环境图后，直接基于店铺当前保存的 `images` 调用视觉模型，返回一组可用于后续内容生成或表单回填的标签。

本次能力是 `backend/app/modules/promotions/` 现有店铺配置链路的增量扩展，不新建独立模块，不新增数据库表，不把 AI 标签结果落库保存。

## Scope

### In Scope

- 在现有店铺模块下新增店铺子资源接口
- 服务端读取当前店铺已保存的 `images`
- 逐张调用视觉模型生成标签
- 聚合、去重、截断后返回 `tags`
- 使用现有视觉模型配置直连 OpenAI 兼容接口
- 增加结构校验、超时、异常日志和最小测试覆盖

### Out of Scope

- 本次请求上传新图片
- 把图片标签持久化到 `mkt_promotion_store`
- 返回每张图各自的标签明细
- 前端标签勾选、编辑、保存交互

## API Contract

### Endpoint

- `POST /mkt/api/v1/promotion/stores/{store_id}/image-tags`

该接口挂在店铺下，`store_id` 必须属于当前登录用户所在组织。

### Request Body

无请求体。

接口直接使用当前店铺已保存的 `images` 作为输入源，不接受前端额外传入图片 URL 列表。

### Response Body

```json
{
  "tags": ["环境氛围", "朋友聚餐", "招牌菜", "夜宵"]
}
```

说明：

- `tags` 返回字符串数组
- 标签顺序按聚合后的首次出现顺序保留
- 当前版本不返回推荐理由，不返回图片级明细

## Service Flow

服务编排放在 `backend/app/modules/promotions/service.py`，控制器只做 HTTP 映射。

处理流程：

1. 根据 `store_id + org_id` 查询店铺，不存在则返回 `404`
2. 读取当前店铺 `images`，过滤空字符串
3. 若没有可用图片，返回 `400`
4. 遍历图片 URL，逐张调用视觉模型
5. 解析并清洗每次模型返回的 `tags`
6. 聚合去重并截断最终标签列表
7. 返回 `{"tags": [...]}` 给前端

实现上不修改店铺数据，不新增 repository 写入逻辑。

## AI Integration

### Reuse

复用现有视觉模型配置和同步 `httpx` 调用方式：

- `OPENAI_API_KEY`
- `OPENAI_BASE_URL`
- `OPENAI_VISION_MODEL_ID`
- `OPENAI_VISION_TIMEOUT_SECONDS`

继续使用 `chat/completions`，并要求模型返回 JSON。

### Prompt Inputs

提示词输入包含：

- `industry_type`
- `store_name`
- `address`
- `target_audience`
- 当前图片 URL

模型角色定义为“本地生活商家图片标签助手”，要求根据商家信息和单张图片内容返回适合营销场景的中文短标签。

### Required Model Output

模型必须返回严格 JSON：

```json
{
  "tags": ["环境氛围", "朋友聚餐", "招牌菜"]
}
```

不允许输出 Markdown，不允许输出额外说明文本。

## Result Normalization

后端不直接信任模型输出，必须做结构校验和清洗。

清洗规则：

- 只接受 `tags` 数组
- 非字符串项丢弃
- 空字符串、空白字符串丢弃
- 过长标签丢弃
- 单张图返回的重复标签去重
- 最终聚合结果跨图片去重
- 最多返回 8 个标签
- 聚合后若没有任何有效标签，视为 AI 结果不可用

## Error Handling

控制器层错误语义：

- 店铺不存在或不属于当前组织：`404 Store not found`
- 店铺没有已保存图片：`400 Store images are required`
- 视觉模型配置缺失、AI 调用失败、超时、返回非预期 JSON、清洗后无有效标签：`503 Store image tags unavailable`

建议在服务层新增独立异常类型，例如：

- `StoreImagesRequiredError`
- `StoreImageTagUnavailableError`

避免与店名推荐、周边发现异常混淆。

## Logging

新日志使用：

```python
logger.bind(module="promotions")
```

至少记录：

- `store_id`
- `image_url`
- 调用阶段，例如 `request`、`parse`
- 失败原因摘要

不要记录：

- 完整 API key
- 完整 token
- 超长原始模型正文

## Tests

按 TDD 增加最小测试覆盖。

### API Tests

- 成功调用 `/image-tags`，返回 `tags`
- 店铺不存在时返回 `404`
- 店铺没有图片时返回 `400`
- 服务抛出 AI 不可用异常时返回 `503`

### Service Tests

- 能正确解析模型 JSON
- 非法标签会被过滤
- 跨图片标签会去重
- 多张图标签会按首次出现顺序聚合
- 店铺没有图片时抛出 `StoreImagesRequiredError`
- 清洗后没有有效标签时抛出不可用异常

## Data Model Impact

本次不新增数据库表，不修改 `mkt_promotion_store` 表结构。

继续复用现有字段：

- `images`

AI 生成标签仅作为一次性响应返回，不持久化到数据库。
