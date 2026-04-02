# Promotion Content Angle Suggestions Design

## 背景

当前 promotion 每日内容任务链路已经支持：

- 上传任务素材
- 基于多图进行任务级视觉分析，产出 `analysis_summary` 与 `analysis_tags`
- 基于热点与参考案例生成内容主稿和平台草稿

前端新增了“内容切入角度”卡片，除了需要后端返回基于任务素材和分析结果的角度推荐，还需要满足两件事：

- 用户当前选择的角度要持久化，页面刷新或回到任务详情后仍能恢复
- `generate` 接口必须显式接收最终选择的角度，而不是只隐式依赖任务状态

因此，本次设计不再只是新增一个只读推荐接口，而是补齐“推荐、选择落库、生成透传”三段闭环。

## 目标

为 `content-task` 增加内容切入角度能力，满足以下目标：

1. 提供基于任务素材和分析结果的角度推荐接口
2. 持久化任务当前已选角度
3. 在 `generate` 接口入参中显式传入最终已选角度
4. 在生成文案时优先参考用户已选角度，同时保留热点和参考案例的辅助作用

## 非目标

- 不新增独立角度表或关联表
- 不引入新的异步任务
- 不把角度推荐和任务图片分析合并为一个接口
- 不在本次修改中持久化“历史推荐列表”

## 核心设计结论

### 1. 角度状态归属 `content-task`

角度是针对“某一天、某一批素材、某一次生成”的内容策划决策，不是店铺长期配置。因此状态归属应放在 `mkt_promotion_content_task`，而不是店铺表。

### 2. 选中状态需要持久化，但 `generate` 仍要显式传参

只做持久化不够，因为 `generate` 会变成依赖隐式任务状态；只做 `generate` 传参也不够，因为页面无法稳定恢复当前选择。

因此两者同时保留：

- `selected_angles` 持久化到任务表，负责页面恢复和任务快照
- `generate` 请求显式提交 `selected_angles`，负责表达“本次最终生成使用的角度”

生成前，服务端以 `generate` 入参为准覆盖任务上的 `selected_angles`，确保最终任务快照与本次生成输入一致。

### 3. 当前 UI 按单选设计，接口按数组建模

截图中的交互是单选，但现有生成链路内部使用 `creative_angles: list[str]`。为减少后续接口变更成本，本次统一使用数组字段 `selected_angles`，但当前校验规则限制最多 1 个值。

这样可以兼顾：

- 当前前端单选交互
- 后端与现有 `creative_angles` 语义兼容
- 未来若改为多选，无需修改字段类型

## 数据模型设计

### Task 新增字段

在 `mkt_promotion_content_task` 新增：

- `selected_angles JSONB NOT NULL DEFAULT '[]'::jsonb`

字段语义：

- 存储当前任务已选择的内容切入角度
- 当前数量约束为 `0..1`
- 内容为中文短语数组，例如 `["山景度假"]`

字段注释：

- `已选内容切入角度`

### Task 响应补充字段

以下响应都补充 `selected_angles: list[str]`：

- `PromotionContentTaskPublic`
- `PromotionContentTaskDetailPublic`

这样前端不需要额外查询就能恢复任务的已选角度状态。

## API 设计

## 1) 推荐接口

### Endpoint

`POST /mkt/api/v1/promotion/stores/{store_id}/content-tasks/{task_id}/angle-suggestions`

### Response

```json
{
  "suggestions": [
    {
      "name": "山景度假",
      "reason": "突出山景阳台与放松度假氛围，适合强化目的地休闲心智",
      "selected": true
    },
    {
      "name": "禅意美学",
      "reason": "强调原木、留白与安静空间感，适合走审美和治愈表达",
      "selected": false
    }
  ]
}
```

### 推荐接口行为

1. 校验 `store_id` 与 `task_id` 属于当前组织范围
2. 读取任务、店铺和任务素材
3. 校验任务下至少存在 1 张素材
4. 校验任务已有任务级分析结果：`analysis_summary` 非空，或 `analysis_tags` 非空
5. 调用文本模型生成 1 到 4 个角度候选
6. 服务端做去重、截断和默认选中处理
7. 若任务当前 `selected_angles` 为空，或其值不在本次候选中，则将首个候选回写到 `selected_angles`
8. 返回候选列表，并根据任务最终 `selected_angles` 标记 `selected`

### 推荐接口输出约束

- `suggestions` 返回 1 到 4 个候选
- `name` 为中文短语，建议控制在 2 到 8 个字
- `reason` 为一句中文解释，建议控制在 18 到 40 个字
- 最终仅 1 个候选标记为 `selected=true`

### 推荐接口为何允许写入

该接口虽然名义上是推荐，但当前 UI 需要“打开即有一个默认选中值”，并且该默认值需要被持久化以支持恢复。因此这里允许在没有有效已选角度时，把首个候选回写到任务。

这是默认选中态落库，不代表用户最终确认；最终生成仍以 `generate` 请求中的 `selected_angles` 为准。

## 2) 选择持久化接口

### Endpoint

`PATCH /mkt/api/v1/promotion/stores/{store_id}/content-tasks/{task_id}/angle-selections`

### Request

```json
{
  "selected_angles": ["山景度假"]
}
```

### Response

返回更新后的任务摘要，至少包含：

```json
{
  "id": "task-id",
  "selected_angles": ["山景度假"]
}
```

实际实现可直接复用 `PromotionContentTaskPublic`，保持和其他任务更新接口一致。

### 选择持久化规则

- 当前要求 `selected_angles` 恰好为 1 个值
- 空数组、多个值、空字符串都返回 `400`
- 单个角度值建议限制最大长度，例如 16 或 20 字
- 本接口只做结构和长度校验，不反查“是否来自最近一次 AI 候选”

不反查候选集的原因：

- 当前不持久化历史推荐列表
- 如果每次保存都重新调用 AI 验证，会导致接口不稳定且成本高
- 最终前端仍应以推荐接口返回的候选为主，后端只保证结构合法

## 3) Generate 接口改造

### Endpoint

仍然使用：

`POST /mkt/api/v1/promotion/stores/{store_id}/content-tasks/{task_id}/generate`

### Request 变更

新增字段：

```json
{
  "platforms": ["xiaohongshu"],
  "hotspot_ids": [],
  "reference_case_ids": [],
  "selected_angles": ["山景度假"]
}
```

### Generate 接口规则

- `selected_angles` 当前要求恰好 1 个值
- 服务端在生成前先把请求中的 `selected_angles` 回写到任务表
- 如果任务当前已保存的 `selected_angles` 与请求不一致，以请求为准覆盖
- 后续生成链路使用请求中的 `selected_angles` 作为显式输入

这样可以保证：

- 页面持久化状态和最终生成输入一致
- `generate` 行为不依赖隐式任务状态
- 后续回查任务时可以知道本次生成基于哪个角度

## 模型调用设计

### 推荐模型

推荐接口沿用现有 `OPENAI_COMPAT_CHEAP_*` 配置。

提示词至少包含：

- 店铺名称
- 行业
- 城市
- 地址
- 目标客群
- 特色服务
- 任务分析摘要 `analysis_summary`
- 任务分析标签 `analysis_tags`
- 素材数量

模型输出 JSON 结构：

```json
{
  "suggestions": [
    {
      "name": "山景度假",
      "reason": "突出山景阳台与放松度假氛围"
    }
  ]
}
```

### 生成模型

现有生成链路中已经存在 `creative_angles` 概念，由热点压缩而来。本次改造后，生成提示词中应显式区分两类信息：

- 用户已选角度：来自 `generate.selected_angles`
- 热点辅助角度：来自现有热点压缩逻辑

建议生成提示词增加两段：

- `已选切入角度：山景度假`
- `热点辅助角度：xxx；yyy`

语义上，用户已选角度优先级高于热点辅助角度。

实现上可采用以下任一方式：

1. 直接把 `selected_angles` 作为新的 prompt 段落单独传入
2. 把 `selected_angles` 放在 `creative_angles` 前面做去重拼接

本次推荐采用方式 1，因为语义更清晰，后续排障时也更容易看出“用户选择”和“系统推断”分别是什么。

## 输出清洗规则

### 推荐接口清洗

- 忽略空名称或空理由
- 按 `name` 去重，保留首次出现项
- 最多保留前 4 项
- 若模型返回超过 4 项，截断
- 若清洗后为空，视为接口不可用

### 选中态计算

- 如果任务 `selected_angles` 命中候选，则按该值打标
- 如果未命中，则使用首项作为默认选中并回写任务
- 返回时只有一个候选 `selected=true`

## 错误处理

### 推荐接口

#### 404

任务不存在或不属于当前组织 / 店铺范围时返回：

`Task not found`

#### 400

当任务不满足推荐前置条件时返回：

- `Task requires at least 1 asset before angle suggestions`
- `Task requires analysis before angle suggestions`

#### 503

以下场景统一返回：

`Angle suggestions unavailable`

包括：

- `OPENAI_COMPAT_CHEAP_API_KEY` 或 `OPENAI_COMPAT_CHEAP_MODEL_ID` 缺失
- 文本模型请求失败
- 模型返回结构异常
- 输出清洗后为空

### 选择持久化接口 / Generate 接口

#### 400

以下场景返回 `400`：

- `selected_angles` 为空
- `selected_angles` 超过 1 个值
- 角度为空字符串
- 角度长度超限

#### 404

任务不存在时返回：

`Task not found`

服务层需要通过 `loguru` 记录失败阶段、`store_id`、`task_id` 和错误原因，但不得输出敏感信息。

## 代码落点

### Schema

在 `backend/app/modules/promotions/content_schema.py` 新增或修改：

- `PromotionContentTaskPublic.selected_angles`
- `PromotionContentTaskGenerateRequest.selected_angles`
- `PromotionContentAngleSuggestion`
- `PromotionContentAngleSuggestionResponse`
- `PromotionContentTaskAngleSelectionRequest`

### Entity

在 `backend/app/modules/promotions/content_entity.py` 的 `PromotionContentTask` 增加：

- `selected_angles: list[str] = Field(default_factory=list, sa_type=JSONB)`

### Repository / DDL

在 promotion content task 建表与兼容 SQL 中增加：

- `selected_angles JSONB NOT NULL DEFAULT '[]'::jsonb`
- 对应列注释 `已选内容切入角度`
- 向后兼容补列 SQL：`ALTER TABLE IF EXISTS mkt_promotion_content_task ADD COLUMN IF NOT EXISTS selected_angles JSONB NOT NULL DEFAULT '[]'::jsonb;`

### Service

在 `backend/app/modules/promotions/content_service.py` 新增：

- `PromotionContentAngleSuggestionUnavailableError`
- 角度提示词构造函数
- 推荐响应解析与清洗函数
- `generate_content_task_angle_suggestions(...)`
- `update_content_task_selected_angles(...)`
- `validate_selected_angles(...)`

并改造：

- `trigger_content_generation(...)` 增加 `selected_angles` 入参
- 生成前回写任务 `selected_angles`

### Controller

在 `backend/app/modules/promotions/content_controller.py` 新增：

- `POST /{task_id}/angle-suggestions`
- `PATCH /{task_id}/angle-selections`

并改造：

- `POST /{task_id}/generate` 的请求体映射

### Generation Service

在 `backend/app/modules/promotions/generation_service.py` 改造主稿生成提示词，显式接收并使用用户已选角度。

## 对现有链路的影响

- 任务表新增一个 JSONB 字段
- 任务公共响应增加 `selected_angles`
- 新增一个选择持久化接口
- `generate` 请求体新增 `selected_angles`
- 主稿生成提示词会显式包含用户已选角度

不变部分：

- 不改 Celery 架构
- 不改热点与参考案例抓取流程
- 不改平台草稿的存储结构

## 测试设计

### Service 层

新增测试覆盖：

1. 推荐成功时返回候选并补充默认选中态
2. 推荐成功且任务已有已选角度时，按持久化值打标
3. 推荐成功但任务已有角度不在候选中时，首项覆盖回写
4. 重复名称去重并截断到 4 项
5. 无素材时报 `ValueError`
6. 无分析结果时报 `ValueError`
7. 文本模型配置缺失时报不可用异常
8. `validate_selected_angles` 对空数组、多值、空串、超长值报错
9. `update_content_task_selected_angles` 能成功回写任务
10. `trigger_content_generation` 会以请求中的 `selected_angles` 覆盖任务并传入生成逻辑

### API 层

新增测试覆盖：

1. 推荐接口成功返回 200
2. 推荐接口任务不存在返回 404
3. 推荐接口缺少素材返回 400
4. 推荐接口缺少分析结果返回 400
5. 推荐接口服务不可用返回 503
6. 选择持久化接口成功返回 200
7. 选择持久化接口非法 `selected_angles` 返回 400
8. `generate` 缺少或非法 `selected_angles` 返回 400
9. `generate` 成功时会把 `selected_angles` 持久化到任务

## 验证计划

最小验证命令：

```bash
cd backend
uv run pytest -q tests/modules/test_promotion_content_service.py tests/api/routes/test_promotion_content_tasks.py
uv run ruff check app tests
```

## 风险与取舍

### 为什么不持久化整份推荐候选

这会引入“推荐快照”和“最终选择”两套状态，复杂度明显上升，而当前需求只要求记住最终已选角度。先持久化 `selected_angles`，足够满足页面恢复和生成追溯。

### 为什么 `generate` 还要传 `selected_angles`

因为生成是高价值动作，需要明确输入来源。显式传参比隐式依赖任务状态更可读，也更便于排查“为什么这次文案按这个角度生成”。

### 为什么字段用数组而不是单字符串

当前 UI 是单选，但后端已有 `creative_angles: list[str]` 语义，数组兼容性更好。当前通过校验把数量限制在 1，既满足现状，也为未来扩展预留空间。

## 后续演进

如果后续前端需要“最近一次 AI 候选仍可恢复展示”，可以在下一轮需求中补充 `last_angle_suggestions` 快照字段或单独的任务推荐快照表。但这不属于本次最小闭环。
