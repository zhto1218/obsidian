# Promotion Daily Content Generation Backend Design

## Goal

在现有 `backend/app/modules/promotions/` 店铺配置链路之上，新增“每日上传图片并生成图文”的后端能力。

目标链路：

1. 用户先完成店铺创建与基础资料配置
2. 在某个店铺下创建一条“每日内容任务”
3. 上传 3 到 9 张图片并调整顺序
4. 基于“城市 + 行业”拉取当天热点资讯，用户勾选 1 到 3 条热点
5. 用户勾选当天要生成的平台
6. 后端异步生成多平台图文草稿
7. 用户审核、编辑后确认，将结果保存为“待使用草稿”

本次只做后端，不改前端页面，不做平台直发。

## Scope

### In Scope

- 继续复用 `promotions` 模块，不新起独立业务域
- 为店铺补充稳定的 `city_name` 城市字段，支持热点按“城市 + 行业”查询
- 新增内容任务、任务素材、热点缓存、任务热点关系、平台草稿五类持久化数据
- 新增图片上传、热点刷新、任务生成、草稿编辑、草稿确认、历史查询接口
- 复用 MinIO 做图片存储
- 复用 Celery 做异步生成
- 基于视觉模型做图片理解，基于现有 OpenAI 兼容接口做文案生成
- 增加最小测试覆盖、配置项、README 和 `.env.example` 文档更新

### Out of Scope

- 前端页面、前端接口接入与原型还原
- 平台账号绑定、一键发布、平台审核回执
- 自动定时调度每日任务
- 发布后数据分析、投放报表
- 文生图、自动补图、封面设计

## Architecture

### Module Boundary

本次能力继续挂在 `backend/app/modules/promotions/` 下，但不继续把新逻辑堆进现有店铺文件。

建议新增或拆分为以下文件：

- `backend/app/modules/promotions/content_entity.py`
- `backend/app/modules/promotions/content_schema.py`
- `backend/app/modules/promotions/content_repository.py`
- `backend/app/modules/promotions/content_service.py`
- `backend/app/modules/promotions/hotspot_service.py`
- `backend/app/modules/promotions/generation_service.py`
- `backend/app/modules/promotions/content_controller.py`
- `backend/app/modules/promotions/tasks.py`

现有文件继续保留职责：

- `controller.py` 继续负责店铺配置、店名推荐、周边发现
- `service.py` 继续负责店铺基础服务
- `repository.py` 继续负责店铺主表读写

外部依赖适配建议放在基础设施层：

- `backend/app/infrastructure/hotspots/provider.py`：热点数据源适配层，支持外部 HTTP provider 与 OpenAI web search 两种模式
- `backend/app/infrastructure/media/minio_storage.py`：继续复用图片存储

模块协作方式：

- `content_controller.py` 只做 HTTP 映射和鉴权隔离
- `content_service.py` 负责任务、素材、热点选择、确认等业务编排
- `generation_service.py` 负责 AI 生成链路
- `content_repository.py` 负责新增表的建表与读写
- `hotspot_service.py` 负责城市提取、缓存命中、热点 provider 调用与归一化

### Persistence Strategy

遵循仓库当前约束，不使用 Alembic。

新增表与字段使用 repository 中的原生 SQL `CREATE TABLE IF NOT EXISTS`、`CREATE INDEX IF NOT EXISTS`、`COMMENT ON` 方式初始化，与现有 `mkt_promotion_store` 保持一致。

所有新增表：

- 使用小写下划线命名
- 时间字段统一存 UTC 无时区时间
- 补齐中文注释
- 补齐必要索引和唯一约束

## Data Model

### 1. Promotion Store Extension

现有表：`mkt_promotion_store`

新增字段：

- `city_name VARCHAR(64)`：店铺所在城市名称

用途：

- 作为“城市 + 行业热点”抓取的稳定条件
- 创建内容任务时写入任务快照

兼容策略：

- 优先使用 `store.city_name`
- 若为空，可尝试从 `address` 做轻量城市提取
- 若从 `address` 成功提取，应回写 `city_name`
- 若仍无法得到城市，则热点接口与生成接口返回 `400`

### 2. Content Task

表名：`mkt_promotion_content_task`

作用：

- 表示一次用户主动创建的“每日内容生成任务”
- 一个店铺同一天允许创建多条任务
- 任务是素材、热点、平台草稿的总聚合根

关键字段：

- `id UUID PRIMARY KEY`
- `org_id UUID NOT NULL`
- `store_id UUID NOT NULL`
- `task_date DATE NOT NULL`
- `city_name_snapshot VARCHAR(64) NOT NULL`
- `industry_type_snapshot VARCHAR(32) NOT NULL`
- `status VARCHAR(32) NOT NULL`
- `hotspot_fetch_status VARCHAR(24) NOT NULL`
- `hotspot_fetch_error TEXT`
- `analysis_summary TEXT`
- `analysis_tags JSONB NOT NULL DEFAULT '[]'::jsonb`
- `selected_platforms JSONB NOT NULL DEFAULT '[]'::jsonb`
- `generation_error_code VARCHAR(64)`
- `generation_error_message TEXT`
- `generating_started_at TIMESTAMP`
- `generated_at TIMESTAMP`
- `confirmed_at TIMESTAMP`
- `created_at TIMESTAMP`
- `updated_at TIMESTAMP`
- `deleted_at TIMESTAMP`

状态定义：

- `draft`：任务已创建，但还没发起生成
- `generating`：已提交生成，Celery 正在处理
- `review_pending`：所有目标平台草稿生成成功，等待人工审核
- `review_pending_partial`：部分平台成功，部分平台失败
- `confirmed`：已确认，作为待使用草稿保留
- `failed`：整条任务生成失败，没有可审核草稿

热点抓取状态：

- `pending`
- `ready`
- `stale`
- `failed`

索引建议：

- `idx_mkt_promotion_content_task_org_store_date` on `(org_id, store_id, task_date desc)`
- `idx_mkt_promotion_content_task_status` on `(status)`

说明：

- 不对 `(store_id, task_date)` 做唯一约束，满足“同店同日允许多次生成”
- `analysis_summary` 和 `analysis_tags` 直接服务于后续前端“AI 图库理解结果”展示

### 3. Content Asset

表名：`mkt_promotion_content_asset`

作用：

- 保存任务下每一张图片素材及其顺序
- 保存图片理解后的摘要

关键字段：

- `id UUID PRIMARY KEY`
- `task_id UUID NOT NULL`
- `image_url VARCHAR(512) NOT NULL`
- `sort_order INTEGER NOT NULL`
- `source_type VARCHAR(24) NOT NULL DEFAULT 'upload'`
- `analysis_summary TEXT`
- `created_at TIMESTAMP`
- `updated_at TIMESTAMP`
- `deleted_at TIMESTAMP`

索引建议：

- `idx_mkt_promotion_content_asset_task_id` on `(task_id)`
- `idx_mkt_promotion_content_asset_task_sort_order` on `(task_id, sort_order)`

说明：

- 第一版只支持用户上传图片，`source_type` 先保留为扩展位
- 删除素材只做软删，不在本期实现 MinIO 物理删除

### 4. Hotspot Cache

表名：`mkt_promotion_hotspot`

作用：

- 缓存“城市 + 行业 + 日期”的热点结果
- 避免同城同业在同一天重复调用三方热点服务

关键字段：

- `id UUID PRIMARY KEY`
- `city_name VARCHAR(64) NOT NULL`
- `industry_type VARCHAR(32) NOT NULL`
- `hot_date DATE NOT NULL`
- `title VARCHAR(128) NOT NULL`
- `summary TEXT`
- `source_name VARCHAR(64)`
- `source_url VARCHAR(512)`
- `score DOUBLE PRECISION NOT NULL DEFAULT 0`
- `published_at TIMESTAMP`
- `created_at TIMESTAMP`
- `updated_at TIMESTAMP`

唯一约束建议：

- `uniq_mkt_promotion_hotspot_city_industry_date_title_source`

索引建议：

- `idx_mkt_promotion_hotspot_city_industry_date` on `(city_name, industry_type, hot_date)`

说明：

- 第一版热点本质是当天候选池，不做长期趋势表
- `score` 用于前端排序和默认推荐

### 5. Task Hotspot Relation

表名：`mkt_promotion_task_hotspot`

作用：

- 保存某个任务最终勾选了哪些热点

关键字段：

- `id UUID PRIMARY KEY`
- `task_id UUID NOT NULL`
- `hotspot_id UUID NOT NULL`
- `sort_order INTEGER NOT NULL DEFAULT 0`
- `created_at TIMESTAMP`

唯一约束建议：

- `uniq_mkt_promotion_task_hotspot_task_hotspot`

索引建议：

- `idx_mkt_promotion_task_hotspot_task_id` on `(task_id)`

### 6. Platform Draft

表名：`mkt_promotion_platform_draft`

作用：

- 保存每个平台各自的一份生成结果与人工编辑结果

关键字段：

- `id UUID PRIMARY KEY`
- `task_id UUID NOT NULL`
- `platform_code VARCHAR(32) NOT NULL`
- `title TEXT`
- `body TEXT`
- `asset_ids JSONB NOT NULL DEFAULT '[]'::jsonb`
- `review_status VARCHAR(24) NOT NULL DEFAULT 'pending_review'`
- `edited_title TEXT`
- `edited_body TEXT`
- `edited_asset_ids JSONB NOT NULL DEFAULT '[]'::jsonb`
- `error_message TEXT`
- `created_at TIMESTAMP`
- `updated_at TIMESTAMP`

唯一约束建议：

- `uniq_mkt_promotion_platform_draft_task_platform`

状态定义：

- `pending_review`
- `confirmed`
- `failed`

说明：

- 原始生成结果和人工编辑结果分开保存，避免覆盖原稿
- `asset_ids` 和 `edited_asset_ids` 都按顺序保存 UUID 字符串数组

## API Contract

所有接口都挂在店铺资源下，继续使用当前组织隔离逻辑。

### 1. Create Content Task

- `POST /mkt/api/v1/promotion/stores/{store_id}/content-tasks`

请求体：

```json
{
  "task_date": "2026-03-27"
}
```

约束：

- `task_date` 可选，默认取当前上海时区日期
- 若店铺无法得到 `city_name`，返回 `400`

响应：

- 返回任务基础信息
- 初始状态为 `draft`

### 2. List Content Tasks

- `GET /mkt/api/v1/promotion/stores/{store_id}/content-tasks`

支持筛选：

- `task_date`
- `status`
- `limit`
- `offset`

### 3. Get Content Task Detail

- `GET /mkt/api/v1/promotion/stores/{store_id}/content-tasks/{task_id}`

返回内容：

- 任务基础信息
- 素材列表
- 当前热点候选列表
- 已勾选热点 ID 列表
- 平台草稿列表

### 4. Upload Content Asset

- `POST /mkt/api/v1/promotion/stores/{store_id}/content-tasks/{task_id}/assets`

请求：

- `multipart/form-data`
- 字段：`file`
- 可选字段：`sort_order`

约束：

- 仅接受图片类型
- 单任务最多 9 张
- 上传成功后写入 MinIO，落库 `image_url`
- 若 MinIO 未配置，返回 `503`

### 5. Reorder Content Assets

- `PATCH /mkt/api/v1/promotion/stores/{store_id}/content-tasks/{task_id}/assets/reorder`

请求体：

```json
{
  "asset_ids": [
    "11111111-1111-1111-1111-111111111111",
    "22222222-2222-2222-2222-222222222222"
  ]
}
```

约束：

- 必须覆盖当前任务下全部未删除素材
- 后端按数组顺序重写 `sort_order`

### 6. Delete Content Asset

- `DELETE /mkt/api/v1/promotion/stores/{store_id}/content-tasks/{task_id}/assets/{asset_id}`

行为：

- 仅软删
- 若任务已 `confirmed`，返回 `409`

### 7. Analyze Content Assets

- `POST /mkt/api/v1/promotion/stores/{store_id}/content-tasks/{task_id}/analysis`

响应体：

```json
{
  "task_id": "22222222-2222-2222-2222-222222222222",
  "analysis_summary": "这是一间位于山间的精品民宿，整体适合拍照与度假。",
  "analysis_tags": ["山景房", "原木风", "落地窗"]
}
```

行为：

- 同步读取当前任务下全部已上传图片
- 一次性将全部图片发送给视觉模型，生成任务级 `analysis_summary` 和 `analysis_tags`
- 只回写任务表，然后立即返回结果

约束：

- 任务下至少要有 1 张图片，否则返回 `400`
- 若任务已 `confirmed`，返回 `409`
- 若视觉模型配置缺失、调用失败或返回结构异常，返回 `503`

说明：

- 这条接口用于前端“AI 图片理解结果”卡片的同步展示
- 它和 `POST /promotion/stores/{store_id}/image-tags` 不同；后者面向店铺配置阶段，只返回标签，不返回任务级摘要

### 8. Refresh Hotspots

- `POST /mkt/api/v1/promotion/stores/{store_id}/content-tasks/{task_id}/hotspots/refresh`

请求体：

```json
{
  "force_refresh": false
}
```

响应体：

```json
{
  "fetch_status": "ready",
  "message": null,
  "items": [
    {
      "id": "33333333-3333-3333-3333-333333333333",
      "title": "春季限定菜单热度上涨",
      "summary": "当季野菜、春笋相关话题热度明显提升",
      "source_name": "hotspot-provider",
      "source_url": "https://example.com/hotspot/1",
      "score": 92.5
    }
  ]
}
```

行为：

- 优先返回当日缓存
- `force_refresh=true` 时跳过缓存重新拉取
- 三方失败但有历史缓存时，返回 `fetch_status=stale`
- 三方失败且无缓存时，返回 `fetch_status=failed` 和空数组
- 同步更新任务上的 `hotspot_fetch_status`、`hotspot_fetch_error`

### 9. Start Generation

- `POST /mkt/api/v1/promotion/stores/{store_id}/content-tasks/{task_id}/generate`

请求体：

```json
{
  "platforms": ["dianping", "douyin", "xiaohongshu"],
  "hotspot_ids": [
    "33333333-3333-3333-3333-333333333333"
  ]
}
```

平台枚举：

- `dianping`
- `douyin`
- `xiaohongshu`
- `meituan`

校验规则：

- 图片素材数量必须在 3 到 9 张之间
- `platforms` 至少 1 个，最多 4 个
- 若当前任务已有热点候选，`hotspot_ids` 必须在 1 到 3 个之间
- 若热点抓取失败或当天没有候选，允许传空数组继续生成
- 已 `confirmed` 或正在 `generating` 的任务返回 `409`

行为：

- 保存 `selected_platforms`
- 用本次请求覆盖任务热点关联
- 清空旧的错误摘要
- 若任务此前已存在草稿且未确认，先软删旧草稿再生成
- 任务状态改成 `generating`
- 投递 Celery 任务

响应：

- 返回任务当前状态与轮询所需 `task_id`

### 10. Edit Platform Draft

- `PATCH /mkt/api/v1/promotion/stores/{store_id}/content-tasks/{task_id}/drafts/{draft_id}`

请求体：

```json
{
  "title": "小城探店｜这家融合菜餐厅适合约会",
  "body": "这里填写人工修改后的正文",
  "asset_ids": [
    "11111111-1111-1111-1111-111111111111"
  ]
}
```

行为：

- 写入 `edited_title`、`edited_body`、`edited_asset_ids`
- 不覆盖原始生成内容
- 若任务已 `confirmed`，返回 `409`

### 11. Confirm Content Task

- `POST /mkt/api/v1/promotion/stores/{store_id}/content-tasks/{task_id}/confirm`

行为：

- 任务必须处于 `review_pending` 或 `review_pending_partial`
- 至少要有 1 条非失败平台草稿
- 将任务状态更新为 `confirmed`
- 将所有成功草稿状态更新为 `confirmed`
- 写入 `confirmed_at`

说明：

- `confirmed` 之后任务只读
- 需要新内容时，重新创建一条新任务，而不是继续改已确认任务

## Hotspot Fetching Design

### Config

新增配置项：

- `HOTSPOT_PROVIDER_MODE=http`
- `HOTSPOT_PROVIDER_API_URL`
- `HOTSPOT_PROVIDER_API_KEY`
- `HOTSPOT_PROVIDER_OPENAI_MODEL_ID`
- `HOTSPOT_PROVIDER_OPENAI_BASE_URL`
- `HOTSPOT_PROVIDER_TIMEOUT_SECONDS=15`

建议新增 `backend/.env.example`，把这些变量写入模板；根目录 `README.md` 和 `backend/README.md` 也要同步更新。

### Provider Behavior

热点服务的输入参数：

- `city_name`
- `industry_type`
- `task_date`

返回结果最少包含：

- `title`
- `summary`
- `source_name`
- `source_url`
- `score`
- `published_at`

provider 模式：

- `HOTSPOT_PROVIDER_MODE=http`：沿用原有外部 HTTP provider，使用 `HOTSPOT_PROVIDER_API_URL` 和 `HOTSPOT_PROVIDER_API_KEY`
- `HOTSPOT_PROVIDER_MODE=openai_web_search`：调用 OpenAI Responses API 的 web search 工具，使用 `OPENAI_API_KEY` 和 `HOTSPOT_PROVIDER_OPENAI_MODEL_ID`

`openai_web_search` 模式约束：

- 优先使用 `HOTSPOT_PROVIDER_OPENAI_BASE_URL`，未配置时回退到 `OPENAI_BASE_URL`，再回退到官方 `https://api.openai.com/v1`
- 请求里携带 `city_name`、`industry_type`、`task_date`，并把用户位置近似设置为 `CN + city_name + Asia/Shanghai`
- 对行业代码做中文映射，例如 `restaurant -> 餐饮`、`homestay -> 民宿`、`beauty -> 美业`，提高中文网页检索命中率
- 要求模型只输出标准 JSON，字段仍然落在 `title / summary / source_name / source_url / score / published_at`
- 返回结果仍需经过本地归一化、去重、排序与缓存写入，不能绕过现有热点缓存层

### Cache Strategy

缓存键逻辑：

- `city_name + industry_type + hot_date`

刷新逻辑：

1. 先查 `mkt_promotion_hotspot`
2. 有缓存且未强制刷新，直接返回
3. 无缓存或强制刷新，则调用三方热点服务
4. 将结果归一化后 upsert 到缓存表
5. 返回最新候选列表

说明：

- 这里的“三方热点服务”既可以是原有 HTTP provider，也可以是 OpenAI web search provider
- provider 切换只影响拉取方式，不影响缓存键、状态机和下游生成链路

降级逻辑：

- 三方调用失败但缓存存在：返回旧缓存，状态 `stale`
- 三方调用失败且没有缓存：返回空数组，状态 `failed`

这里不使用 `503` 直接打断页面链路，目的是允许用户在热点不可用时继续走“仅店铺资料 + 图片”生成。

## Generation Design

### Config

图片理解需要可识别图片的模型，新增配置项：

- `OPENAI_VISION_MODEL_ID`
- `OPENAI_VISION_TIMEOUT_SECONDS=30`

继续复用已有配置：

- `OPENAI_API_KEY`
- `OPENAI_BASE_URL`
- `OPENAI_COMPAT_CHEAP_API_KEY`
- `OPENAI_COMPAT_CHEAP_BASE_URL`
- `OPENAI_COMPAT_CHEAP_MODEL_ID`

Celery 队列建议新增配置：

- `PROMOTION_CONTENT_QUEUE=promotion_content`

### Celery Wiring

在 `backend/app/infrastructure/celery_app.py` 中：

- 将 `app.modules.promotions.tasks` 加入 `include`
- 新增 `promotion_content` 队列
- 为内容生成任务配置 `task_routes`

第一版不加 Beat 定时任务，仍由用户主动触发。

### Worker Flow

`POST /generate` 只负责校验与入队，真正生成由 Celery Worker 完成。

同步分析接口 `POST /analysis` 会一次性将当前任务图片发送给视觉模型，只回写任务级结果；Worker 仍逐张分析以保留素材级摘要供后续主稿生成使用。

Worker 执行步骤：

1. 读取任务、店铺快照、素材、已选热点
2. 对每张图片调用视觉模型，生成 `analysis_summary`
3. 聚合全部图片摘要，得到任务级 `analysis_summary` 和 `analysis_tags`
4. 将热点候选压缩为 1 到 3 个“创作角度”
5. 结合店铺资料、图片摘要、热点角度生成一份通用主稿
6. 按平台策略改写为各自平台草稿
7. 将每个平台结果写入 `mkt_promotion_platform_draft`
8. 根据成功平台数量更新任务状态

状态流转：

- 全部成功：`generating -> review_pending`
- 部分成功：`generating -> review_pending_partial`
- 全部失败：`generating -> failed`

### Platform Strategy

平台改写规则在后端内置固定模板，第一版支持：

- `dianping`：偏真实探店、菜品/环境/服务/价格结构化表达
- `douyin`：偏强钩子、短句、高转化 CTA
- `xiaohongshu`：偏种草与生活方式表达
- `meituan`：偏福利、套餐、到店转化信息

所有模型输出要求严格 JSON，不返回 Markdown。

### Retry Semantics

单任务允许重复发起生成，但仅限未确认任务。

重复生成规则：

- 先记录本次新的 `platforms` 和 `hotspot_ids`
- 软删旧平台草稿
- 保留素材与热点缓存
- 重新走整条异步生成链路

已确认任务不允许重新生成；如果要生成新一轮内容，应重新创建新任务。

## Error Handling

### HTTP Layer

- 店铺、任务、素材、草稿不属于当前组织：`404`
- 请求体缺字段或类型错误：`422`
- 城市缺失、素材数不足、平台数非法：`400`
- 任务状态不允许当前操作，如已确认后继续改：`409`
- MinIO 未配置、视觉模型不可用或文案模型不可用：`503`

### Worker Layer

图片理解失败：

- 记录 `generation_error_code=asset_analysis_failed`
- 记录 `generation_error_message`
- 整体置 `failed`

主稿生成失败：

- 记录 `generation_error_code=master_draft_failed`
- 整体置 `failed`

单个平台改写失败：

- 对应平台草稿记 `review_status=failed`
- 继续其他平台
- 若至少一个平台成功，任务置 `review_pending_partial`

热点抓取失败：

- 不阻断任务创建
- 不阻断生成接口
- 任务记录 `hotspot_fetch_status=failed`

## Logging

新增日志统一使用：

```python
logger.bind(module="promotions")
```

至少记录：

- `store_id`
- `task_id`
- `platform_code`
- `stage`，例如 `upload`、`hotspot_fetch`、`asset_analysis`、`master_draft`、`platform_rewrite`
- 失败原因摘要

不要记录：

- 完整 API key
- 完整 token
- 超长模型原文
- 用户原始图片二进制

## Tests

### API Tests

- 创建内容任务成功，返回 `draft`
- 店铺没有城市信息时创建任务返回 `400`
- 图片上传成功并能查询到素材列表
- 超过 9 张素材时上传返回 `400`
- 同步图片理解接口会返回任务级 `analysis_summary`、`analysis_tags`
- 同步图片理解在视觉模型不可用时返回 `503`
- 热点刷新在 `ready / stale / failed` 三种状态下返回正确结构
- 生成接口会校验素材数、平台数并将任务置为 `generating`
- 编辑草稿成功写入 `edited_*`
- 确认任务后状态变成 `confirmed`

### Service Tests

- 城市提取逻辑优先使用 `city_name`，其次尝试解析 `address`
- 同步图片理解会写回任务级摘要/标签
- 同步图片理解在没有素材时返回 `400`
- 热点缓存命中时不会重复调用三方服务
- 热点归一化会过滤空标题和重复项
- 热点失败时会返回空数组或旧缓存并更新任务抓取状态
- provider 模式切换会正确走到 HTTP provider 或 OpenAI web search provider
- OpenAI web search 响应会被正确解析成标准热点 item 列表
- 视觉模型结果会写回素材 `analysis_summary`
- 平台草稿部分成功时任务状态为 `review_pending_partial`

### Task Tests

- Celery 任务在全部平台成功时写入草稿并更新任务状态
- Celery 任务在单个平台失败时保留成功草稿
- Celery 任务在整条链路失败时记录错误码和错误摘要

## Documentation Impact

本次实现需要同步更新：

- `backend/app/core/config.py`
- `backend/.env.example`
- `README.md`
- `backend/README.md`

前端本次不接真实接口，因此：

- 不修改 `frontend/docs/UI_RULES.md`
- 不修改 `frontend/docs/ARCHITECTURE_RULES.md`
- 保留当前 `frontend/src/pages/promotion/index.tsx` 原型态页面

## Non-Goals For This Iteration

- 热点自动定时抓取
- 多城市多门店批量任务
- 平台账号授权和发布回执
- 任务审批流
- 任务分享、导出 PDF、导出图片包
