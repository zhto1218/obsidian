# Promotion Store Nearby Discovery Design

## Goal

为推广店铺配置补充“发现周边”接口，供前端在“餐厅在哪里”步骤中，基于用户刚选择的定位信息，调用 AI 发现周边候选标签。

本次能力是 `backend/app/modules/promotions/` 现有店铺配置链路的增量扩展，不新建独立模块，不引入新表，不落库保存 AI 推荐结果。

## Scope

### In Scope

- 在现有店铺模块下新增店铺子资源接口
- 前端提交 `address`、`longitude`、`latitude`
- 服务端先更新当前店铺位置，再调用 AI 发现周边候选
- 返回 `name`、`distance_km`、`selected`
- 使用现有 `OPENAI_COMPAT_CHEAP_*` 配置直连 OpenAI 兼容接口
- 增加结构校验、超时、异常日志和最小测试覆盖

### Out of Scope

- 地图服务或真实 POI 检索
- 对 AI 发现结果做数据库持久化
- 前端“已选周边标签”的提交与保存
- 基于真实地理坐标做精确测距

## API Contract

### Endpoint

- `POST /mkt/api/v1/promotion/stores/{store_id}/nearby-discovery`

该接口挂在店铺下，`store_id` 必须属于当前登录用户所在组织。

### Request Body

```json
{
  "address": "杭州市西湖区",
  "longitude": 120.123456,
  "latitude": 30.123456
}
```

约束：

- `address` 必填，最大长度与现有店铺地址字段约束保持一致
- `longitude`、`latitude` 必填，使用浮点数接收
- 本次请求的定位信息会写回当前店铺

### Response Body

```json
{
  "items": [
    {
      "name": "地铁站",
      "distance_km": 0.3,
      "selected": false
    },
    {
      "name": "购物中心",
      "distance_km": 0.5,
      "selected": false
    },
    {
      "name": "商业街",
      "distance_km": 0.8,
      "selected": false
    }
  ]
}
```

说明：

- `distance_km` 返回数值类型，前端自行格式化为 `0.3km`
- `selected` 统一由后端补为 `false`
- 当前版本不返回推荐理由，不让 AI 决定勾选状态

## Service Flow

服务编排放在 `backend/app/modules/promotions/service.py`，控制器只做 HTTP 映射。

处理流程：

1. 根据 `store_id + org_id` 查询店铺，不存在则返回 `404`
2. 将请求里的 `address`、`longitude`、`latitude` 更新回当前店铺
3. 复用现有店铺进度计算逻辑，刷新 `setup_step`、`last_completed_step`、`setup_completed`
4. 基于店铺当前信息构造 AI 提示词
5. 调用 OpenAI 兼容 `chat/completions` 接口
6. 解析并清洗模型返回结果
7. 按距离升序返回结果给前端

实现上优先复用现有 `update_promotion_store` 的数据归一与状态刷新逻辑，避免为“仅更新位置”复制第二套 repository 写入流程。

## AI Integration

### Reuse

复用现有店铺名称推荐所使用的 OpenAI 兼容配置和调用模式：

- `OPENAI_COMPAT_CHEAP_API_KEY`
- `OPENAI_COMPAT_CHEAP_BASE_URL`
- `OPENAI_COMPAT_CHEAP_MODEL_ID`

继续使用 `httpx` 同步调用，保留超时、异常捕获和 `loguru` 日志记录方式。

### Prompt Inputs

提示词输入包含：

- `industry_type`
- 本次更新后的 `address`
- 本次更新后的 `longitude`
- 本次更新后的 `latitude`
- `store_name`，如已存在

模型角色定义为“本地生活商家选址分析助手”，要求根据商家行业和定位信息，输出该位置附近适合作为宣传卖点或投放标签的周边候选。

### Required Model Output

模型必须返回严格 JSON：

```json
{
  "items": [
    { "name": "购物中心", "distance_km": 0.5 },
    { "name": "商业街", "distance_km": 0.8 },
    { "name": "地铁站", "distance_km": 0.3 }
  ]
}
```

不允许输出 Markdown，不允许输出额外说明文本。

## Result Normalization

后端不直接信任模型输出，必须做结构校验和清洗。

清洗规则：

- 最多保留 10 条
- `name` 为空、空白、重复或过长时丢弃
- `distance_km` 不是正数时丢弃
- 统一补充 `selected=false`
- 最终按 `distance_km` 升序返回
- 清洗后少于 3 条时，视为 AI 结果不可用

距离由 AI 返回近似值，后端只负责格式归一，不做地图精确测距。

## Error Handling

控制器层错误语义：

- 店铺不存在或不属于当前组织：`404 Store not found`
- 请求参数缺失或类型不合法：沿用 FastAPI 默认 `422`
- AI 调用失败、超时、返回非预期 JSON、清洗后少于 3 条：`503 Nearby discovery unavailable`

建议在服务层新增独立异常类型，例如 `NearbyDiscoveryUnavailableError`，避免与店名推荐异常混淆。

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

- 成功调用 `/nearby-discovery`，返回 `items`
- 成功调用后，店铺中的 `address`、`longitude`、`latitude` 已被更新
- 店铺不存在时返回 `404`
- 服务抛出 AI 不可用异常时返回 `503`

### Service Tests

- 能正确解析模型 JSON
- 会补 `selected=false`
- 会按 `distance_km` 升序排序
- 非法项会被过滤
- 过滤后不足 3 条时抛出不可用异常

## Data Model Impact

本次不新增数据库表，不修改 `mkt_promotion_store` 表结构。

店铺位置仍写入现有字段：

- `address`
- `longitude`
- `latitude`

AI 发现结果仅作为一次性响应返回，不持久化到数据库。
