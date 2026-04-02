# Promotion Platform Suggestions Design

## Goal

为推广店铺配置链路补充“投放平台推荐”接口，服务端根据当前店铺已保存的 `target_audience` 和 `featured_services`，调用 AI 返回适合的投放平台候选。

本次能力是 `backend/app/modules/promotions/` 现有店铺配置链路的增量扩展，不新建独立模块，不新增数据库表，不把 AI 推荐结果落库到店铺或内容任务。

## Scope

### In Scope

- 在现有店铺模块下新增店铺子资源接口
- 服务端读取当前店铺已保存配置作为提示词输入
- 复用当前 promotion 店铺 AI 文本能力的 OpenAI 兼容配置与调用方式
- 只在 `douyin` 和 `xiaohongshu` 两个平台中做推荐
- 返回平台编码和简短推荐理由
- 对 AI 返回结果做结构校验、去重和截断
- 增加异常日志和最小测试覆盖

### Out of Scope

- 直接写回 `mkt_promotion_store`
- 修改 `mkt_promotion_content_task.selected_platforms`
- 扩展更多平台枚举
- 返回排序分数、投放预算建议或详细人群画像
- 前端平台选择页交互改造

## API Contract

### Endpoint

- `POST /mkt/api/v1/promotion/stores/{store_id}/platform-suggestions`

该接口挂在店铺下，`store_id` 必须属于当前登录用户所在组织。

### Request Body

无请求体。

接口直接使用当前店铺已保存信息作为输入源，不接受前端额外传入平台范围、自由文本或覆盖字段。

### Response Body

```json
{
  "suggestions": [
    {
      "platform": "douyin",
      "reason": "适合短视频展示门店服务和氛围"
    },
    {
      "platform": "xiaohongshu",
      "reason": "适合客群种草和场景分享"
    }
  ]
}
```

说明：

- `suggestions` 返回对象数组
- `platform` 当前仅允许 `douyin` 或 `xiaohongshu`
- `reason` 为简短中文短句，供前端直接展示
- 顺序按服务端清洗后的推荐顺序保留

## Service Flow

服务编排放在 `backend/app/modules/promotions/service.py`，控制器只做 HTTP 映射。

处理流程：

1. 根据 `store_id + org_id` 查询店铺，不存在则返回 `404`
2. 读取店铺当前信息，重点使用 `target_audience` 和 `featured_services` 组织提示词
3. 调用 OpenAI 兼容文本模型生成平台候选
4. 解析模型返回 JSON
5. 过滤非法平台、空理由、重复项
6. 返回 `{"suggestions": [...]}` 给前端

本次接口不修改店铺数据，不新增 repository 写入逻辑。

## AI Integration

### Reuse

复用当前 promotion 店铺 AI 文本能力使用的配置与调用方式：

- `OPENAI_COMPAT_CHEAP_API_KEY`
- `OPENAI_COMPAT_CHEAP_BASE_URL`
- `OPENAI_COMPAT_CHEAP_MODEL_ID`
- `OPENAI_COMPAT_CHEAP_TIMEOUT_SECONDS`

继续使用同步 `httpx` + `chat/completions`。

### Prompt Inputs

提示词输入优先使用：

- `target_audience`
- `featured_services`

为避免平台推荐脱离店铺语境，也补充以下辅助信息：

- `industry_type`
- `store_name`
- `city_name`

### Required Model Output

模型必须返回严格 JSON：

```json
{
  "suggestions": [
    {
      "platform": "douyin",
      "reason": "适合短视频展示门店服务和氛围"
    }
  ]
}
```

限制：

- `platform` 仅允许 `douyin`、`xiaohongshu`
- `reason` 必须为简短中文短句
- 不允许输出 Markdown
- 不允许输出额外解释文本

## Result Normalization

后端不直接信任模型输出，必须做结构校验和清洗。

清洗规则：

- 只接受 `suggestions` 数组
- 非对象项丢弃
- `platform` 不在允许范围内则丢弃
- 空理由或超长理由丢弃
- 完全重复的平台去重
- 最多返回 2 个候选
- 清洗后至少保留 1 个有效候选，否则视为 AI 结果不可用

## Error Handling

控制器层错误语义：

- 店铺不存在或不属于当前组织：`404 Store not found`
- 文本模型配置缺失、AI 调用失败、超时、返回非预期 JSON、清洗后无有效候选：`503 Platform suggestions unavailable`

服务层建议新增独立异常类型：

- `PlatformSuggestionUnavailableError`

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

- 成功调用 `/platform-suggestions`，返回 `suggestions`
- 店铺不存在时返回 `404`
- 服务抛出 AI 不可用异常时返回 `503`

### Service Tests

- 能正确解析模型 JSON
- 非法平台会被过滤
- 重复平台会去重
- 返回顺序按首次出现顺序保留
- 有效候选为空时抛出 `PlatformSuggestionUnavailableError`
- 店铺信息会进入请求提示词

## Data Model Impact

本次不新增数据库表，不修改 `mkt_promotion_store` 表结构。

继续复用现有字段作为推荐输入：

- `industry_type`
- `store_name`
- `city_name`
- `target_audience`
- `featured_services`

AI 推荐结果仅作为一次性响应返回，不持久化到数据库。
