# Promotion Target Audience Suggestions Design

## Goal

为推广店铺配置第四步补充“目标客群推荐”接口，服务端根据当前店铺已保存的信息调用 AI，返回一组可供前端勾选的目标客群候选项。

本次能力是 `backend/app/modules/promotions/` 现有店铺配置链路的增量扩展，不新建独立模块，不新增数据库表，不直接把 AI 推荐结果写回 `mkt_promotion_store.target_audience`。

## Scope

### In Scope

- 在现有店铺模块下新增店铺子资源接口
- 服务端读取当前店铺已保存的信息作为提示词输入
- 调用现有 OpenAI 兼容文本模型生成目标客群候选
- 对 AI 返回结果做结构校验、清洗、去重和截断
- 返回候选列表供前端展示和勾选
- 增加异常日志和最小测试覆盖

### Out of Scope

- 本次接口直接写回 `target_audience`
- 修改现有 `PATCH /promotion/stores/{store_id}` 保存逻辑
- 新增数据库字段保存“AI 推荐历史”
- 返回推荐理由、排序分数或行业画像说明
- 前端第四步页面交互改造

## API Contract

### Endpoint

- `POST /mkt/api/v1/promotion/stores/{store_id}/target-audience-suggestions`

该接口挂在店铺下，`store_id` 必须属于当前登录用户所在组织。

### Request Body

无请求体。

接口直接使用当前店铺已保存信息作为输入源，不接受前端额外传入客群标签、自由文本或覆盖字段。

### Response Body

```json
{
  "suggestions": ["家庭聚餐", "情侣约会", "商务宴请", "朋友聚会"]
}
```

说明：

- `suggestions` 返回字符串数组
- 顺序按服务端清洗后的推荐顺序保留
- 当前版本不返回 `recommended` 标记，不返回推荐理由
- 前端是否高亮“推荐”标签，仍由自身页面策略决定

## Service Flow

服务编排放在 `backend/app/modules/promotions/service.py`，控制器只做 HTTP 映射。

处理流程：

1. 根据 `store_id + org_id` 查询店铺，不存在则返回 `404`
2. 读取店铺当前信息，组织提示词输入
3. 调用 OpenAI 兼容文本模型生成目标客群候选
4. 解析模型返回 JSON
5. 清洗、去重、截断推荐项
6. 返回 `{"suggestions": [...]}` 给前端

本次接口不修改店铺数据，不新增 repository 写入逻辑。

## AI Integration

### Reuse

复用当前 promotion 店铺 AI 文本能力使用的配置与调用方式：

- `OPENAI_COMPAT_CHEAP_API_KEY`
- `OPENAI_COMPAT_CHEAP_BASE_URL`
- `OPENAI_COMPAT_CHEAP_MODEL_ID`

继续使用同步 `httpx` + `chat/completions`。

### Prompt Inputs

提示词输入包含当前店铺的已有信息，优先使用：

- `industry_type`
- `store_name`
- `city_name`
- `address`
- `target_audience`
- `featured_services`
- `booking_channels`
- `images` 数量

其中：

- `target_audience` 若已有值，作为“已有用户认知”参与提示词，但接口仍返回新的候选列表
- `images` 不直接传图片内容，仅可作为“已有素材数量”辅助判断店铺成熟度

### Required Model Output

模型必须返回严格 JSON：

```json
{
  "suggestions": ["家庭聚餐", "情侣约会", "朋友聚会"]
}
```

不允许输出 Markdown，不允许输出额外说明文本。

## Result Normalization

后端不直接信任模型输出，必须做结构校验和清洗。

清洗规则：

- 只接受 `suggestions` 数组
- 非字符串项丢弃
- 空字符串、空白字符串丢弃
- 超过 12 个字符的客群描述丢弃
- 完全重复项去重
- 最多返回 8 个候选
- 清洗后至少保留 3 个有效候选，否则视为 AI 结果不可用

当前版本不做行业词典强校验，避免把合理但不在静态枚举中的新客群误杀。

## Error Handling

控制器层错误语义：

- 店铺不存在或不属于当前组织：`404 Store not found`
- 文本模型配置缺失、AI 调用失败、超时、返回非预期 JSON、清洗后有效候选不足：`503 Target audience suggestions unavailable`

服务层建议新增独立异常类型：

- `TargetAudienceSuggestionUnavailableError`

不新增 `400` 类参数错误，因为该接口无请求体，且输入全部来自已有店铺信息；即使店铺资料不完整，也允许 AI 基于现有信息尽量生成。

## Logging

新日志使用：

```python
logger.bind(module="promotions")
```

至少记录：

- `store_id`
- 调用阶段，例如 `request`、`parse`
- 失败原因摘要

不要记录：

- 完整 API key
- 完整 token
- 超长原始模型正文

## Tests

按 TDD 增加最小测试覆盖。

### API Tests

- 成功调用 `/target-audience-suggestions`，返回 `suggestions`
- 店铺不存在时返回 `404`
- 服务抛出 AI 不可用异常时返回 `503`

### Service Tests

- 能正确解析模型 JSON
- 非法项会被过滤
- 重复项会去重
- 返回顺序按首次出现顺序保留
- 候选过少时抛出 `TargetAudienceSuggestionUnavailableError`
- 店铺信息会进入请求提示词

## Data Model Impact

本次不新增数据库表，不修改 `mkt_promotion_store` 表结构。

继续复用现有字段作为推荐输入：

- `industry_type`
- `store_name`
- `city_name`
- `address`
- `target_audience`
- `featured_services`
- `booking_channels`
- `images`

AI 推荐结果仅作为一次性响应返回，不持久化到数据库。
