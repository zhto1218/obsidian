# Promotion Reference Cases Pre-Generation Design

## Goal

在现有“每日内容生成”链路前，新增“同行创业参考”能力，为 AI 生成提供可选的真实同行案例灵感输入。

目标链路：

1. 用户先完成店铺基础配置
2. 用户进入某个店铺的当日内容任务
3. 用户上传 3 到 9 张素材并完成素材分析
4. 系统拉取同城同业优先的真实同行案例候选
5. 用户在“同行创业参考”步骤中可选 0 到 3 条案例
6. 生成接口将选中的案例作为额外灵感输入传给模型
7. 用户跳过该步骤时，系统仍按默认逻辑完成生成

本次能力重点是“真实同行案例为生成前提供灵感输入”，不是做独立资讯产品，也不是替代现有热点能力。

## Scope

### In Scope

- 在 `promotions` 内容任务链路中新增“同行创业参考”候选池、任务关联和生成输入
- 新增真实同行案例抓取、缓存、刷新和任务关联接口
- 在前端内容任务流中新增“同行创业参考”步骤
- 支持 0 到 3 条案例选择，不选也能生成
- 将选中的案例转成结构化 prompt 输入，并入现有生成链路
- 调整现有生成校验，使旧 `hotspot_ids` 不再成为生成前的强制门槛
- 增加最小测试覆盖、配置项和文档更新

### Out of Scope

- 不做独立“资讯中心”或长期案例库浏览页面
- 不做收藏、历史复用、分享、案例详情页
- 不做案例自动打标签、人工运营后台
- 不做平台真实发布数据回流
- 不做“AI 伪造同行案例”兜底

## Product Decisions

### 1. Feature Role

“同行创业参考”只承担一个角色：给 AI 提供额外灵感输入。

它不是强制前置条件，也不是用户必须读完的资讯页。用户不选择任何案例时，生成仍然要成功。

### 2. Data Authenticity

本次只接受“真实同行案例”语义。

如果抓取失败、没有候选、候选过少，允许页面展示空态或失败态，但不允许悄悄回退成“AI 编造的同行案例”。真实性优先级高于填满卡片数量。

### 3. Selection Rule

- 最多可选 3 条
- 最少可选 0 条
- 已选满 3 条后，其余卡片不可再选
- 允许用户直接点击“跳过，直接生成”

### 4. Position in Flow

该步骤必须位于“素材分析完成”之后、“正式生成”之前。

原因：

- 在素材分析前展示，用户上下文不足，选择会偏盲
- 在生成后展示，已经失去“生成前灵感输入”的意义
- 放进店铺基础配置会污染长期资料与一次性任务输入的边界

## Architecture

### Module Boundary

这项能力继续放在 `backend/app/modules/promotions/` 下，但继续沿用“内容任务链路”和“店铺配置链路”分治，不把任务级逻辑塞回店铺配置服务。

建议新增或扩展以下文件：

- `backend/app/modules/promotions/content_entity.py`
- `backend/app/modules/promotions/content_schema.py`
- `backend/app/modules/promotions/content_repository.py`
- `backend/app/modules/promotions/content_service.py`
- `backend/app/modules/promotions/generation_service.py`
- `backend/app/modules/promotions/content_controller.py`
- `backend/app/modules/promotions/reference_case_service.py`
- `backend/app/infrastructure/reference_cases/provider.py`

职责边界：

- `content_controller.py` 只负责任务级接口映射
- `content_service.py` 负责任务详情、案例刷新、生成触发编排
- `reference_case_service.py` 负责案例抓取、缓存、归一化和降级
- `generation_service.py` 负责将选中案例转成生成上下文
- `content_repository.py` 负责案例候选表、任务关联表和任务字段扩展

### Frontend Boundary

前端不要把该步骤继续堆进当前 [promotion 页面](/Users/zhto/.codex/worktrees/ad8b/ai-marketing-assistant/frontend/src/pages/promotion/index.tsx)。

当前页面的主职责是店铺长期配置，而“同行创业参考”属于某次内容任务的一次性输入。两者放在同一个大页面中，后续会让状态、请求和回退逻辑持续耦合。

建议新增独立内容任务页，例如：

- `frontend/src/pages/promotion-content/index.tsx`
- `frontend/src/pages/promotion-content/index.config.ts`

并在业务组件层拆出：

- `frontend/src/components/business/promotion/PromotionReferenceCasesStep.tsx`
- `frontend/src/components/business/promotion/PromotionReferenceCaseCard.tsx`
- `frontend/src/components/business/promotion/PromotionReferenceCasesEmptyState.tsx`

公共按钮、头部、空态继续复用现有基础组件，不在页面层新造平行基础控件。

## Persistence Strategy

继续遵循仓库约束，不使用 Alembic。

数据库结构调整统一在 `content_repository.py` 中通过原生 SQL 初始化：

- `CREATE TABLE IF NOT EXISTS`
- `CREATE INDEX IF NOT EXISTS`
- `COMMENT ON`

所有新增字段和新表保持：

- 小写下划线命名
- UTC 时间存储
- 中文注释完整
- 必要索引和唯一约束齐全

## Data Model

### 1. Content Task Extension

现有表：`mkt_promotion_content_task`

新增字段：

- `reference_case_fetch_status VARCHAR(24) NOT NULL DEFAULT 'pending'`
- `reference_case_fetch_error TEXT`

状态值沿用热点抓取语义：

- `pending`
- `ready`
- `stale`
- `failed`

用途：

- 支持页面区分“未加载、已就绪、使用陈旧缓存、刷新失败”四种状态
- 支持生成失败重试时保留上一次候选池状态

说明：

- 不复用 `hotspot_fetch_status`，避免“热点候选”和“同行案例候选”语义混淆

### 2. Reference Case Candidate

表名：`mkt_promotion_reference_case`

作用：

- 缓存“城市 + 行业 + 日期”的真实同行案例候选池
- 为某个内容任务提供可引用的案例列表

关键字段：

- `id UUID PRIMARY KEY`
- `org_id UUID NOT NULL`
- `city_name VARCHAR(64) NOT NULL`
- `industry_type VARCHAR(32) NOT NULL`
- `reference_date DATE NOT NULL`
- `source_platform VARCHAR(32) NOT NULL`
- `title VARCHAR(128) NOT NULL`
- `summary TEXT`
- `inspiration_point TEXT`
- `metric_label VARCHAR(64)`
- `source_name VARCHAR(64)`
- `source_url VARCHAR(512)`
- `score DOUBLE PRECISION NOT NULL DEFAULT 0`
- `published_at TIMESTAMP`
- `created_at TIMESTAMP`
- `updated_at TIMESTAMP`
- `deleted_at TIMESTAMP`

字段含义：

- `summary`：说明这个同行案例实际做了什么
- `inspiration_point`：明确告诉用户和模型“可借鉴点是什么”
- `metric_label`：例如 `2.3 万赞`、`热度 Top 5`、`1.8 万互动`
- `score`：同城同业相关性、发布时间新鲜度、传播热度综合分

唯一约束建议：

- `uniq_mkt_promotion_reference_case_org_city_industry_date_title_source`

索引建议：

- `idx_mkt_promotion_reference_case_city_industry_date`
- `idx_mkt_promotion_reference_case_score`

### 3. Task Reference Case Relation

表名：`mkt_promotion_task_reference_case`

作用：

- 保存某个内容任务最终选择了哪些同行案例

关键字段：

- `id UUID PRIMARY KEY`
- `org_id UUID NOT NULL`
- `task_id UUID NOT NULL`
- `reference_case_id UUID NOT NULL`
- `sort_order INTEGER NOT NULL DEFAULT 0`
- `created_at TIMESTAMP`

唯一约束建议：

- `uniq_mkt_promotion_task_reference_case_task_case`

索引建议：

- `idx_mkt_promotion_task_reference_case_task_id`

## Provider Strategy

### Fetching Rule

案例来源策略：

1. 优先同城同业
2. 同城不足时，允许扩到同省同业
3. 同省仍不足时，允许全国同业，但要降低 `score`

首版返回目标：

- 候选数理想值 6 条
- 最少允许 0 条
- 不要求为了凑数而牺牲真实性

### Provider Boundary

建议新增 `backend/app/infrastructure/reference_cases/provider.py`，与现有热点 provider 平行。

provider 输出统一格式：

```json
{
  "items": [
    {
      "source_platform": "xiaohongshu",
      "title": "山顶早餐观景房内容走红",
      "summary": "某民宿通过清晨山景早餐场景内容获得高互动。",
      "inspiration_point": "把服务包装成具体体验场景，而不是只罗列配置。",
      "metric_label": "2.3万赞",
      "source_name": "小红书",
      "source_url": "https://example.com/post/1",
      "score": 93,
      "published_at": "2026-03-29T08:00:00"
    }
  ]
}
```

### Failure Policy

- provider 失败时记录摘要日志，不向控制器裸抛底层 SDK 异常
- 如果有旧缓存，返回 `stale`
- 如果无缓存，返回 `failed`
- 无论是 `stale` 还是 `failed`，都不阻断最终生成流程

建议日志字段：

- `module=promotions`
- `stage=reference_case_fetch`
- `task_id`
- `city_name`
- `industry_type`
- `provider_mode`
- `error`

## API Design

### 1. Refresh Reference Cases

接口：

- `POST /promotion/stores/{store_id}/content-tasks/{task_id}/reference-cases/refresh`

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
  "items": [],
  "message": null
}
```

说明：

- `items` 返回候选池
- `message` 用于向前端表达“使用缓存”“今日暂无候选”“刷新失败已降级”等摘要

### 2. Task Detail

接口：

- `GET /promotion/stores/{store_id}/content-tasks/{task_id}`

新增响应字段：

- `reference_case_fetch_status`
- `reference_case_fetch_error`
- `reference_cases`

`reference_cases` 类型建议为：

- `id`
- `source_platform`
- `title`
- `summary`
- `inspiration_point`
- `metric_label`
- `source_name`
- `source_url`
- `score`
- `published_at`

### 3. Generate Task

接口：

- `POST /promotion/stores/{store_id}/content-tasks/{task_id}/generate`

请求体从：

```json
{
  "platforms": ["dianping"],
  "hotspot_ids": []
}
```

扩展为：

```json
{
  "platforms": ["dianping"],
  "hotspot_ids": [],
  "reference_case_ids": []
}
```

校验策略：

- `reference_case_ids` 允许为空
- 非空时数量必须在 1 到 3 之间
- 所有 id 必须属于当前任务候选池
- 去重后再写入关联表

兼容策略：

- `hotspot_ids` 保持兼容，但不再因为候选池存在就强制用户必选
- 生成输入允许“无热点、无案例、只有素材”的最小可运行路径

## Generation Strategy

### Prompt Input Structure

现有生成输入至少包含：

- 店铺快照
- 素材分析摘要
- 已上传图片
- 平台类型

新增“同行创业参考”后，prompt 结构增加独立一段：

```text
同行案例参考：
1. 标题：……
   案例事实：……
   可借鉴点：……
   热度摘要：……
```

注意事项：

- 不把 `reference_case` 伪装成 `hotspot`
- 不把用户未选择的候选也一并塞进 prompt
- 不要求模型模仿某个品牌名或直接照抄标题
- 该输入仅用于帮助模型选择选题角度、叙事切口和表达方式

### Suggested Helper

在 `generation_service.py` 中新增辅助方法，例如：

- `build_reference_case_angles(reference_cases: list[Any]) -> list[str]`

输出目标：

- 每条控制在 1 行内
- 优先保留 `title + inspiration_point + metric_label`
- 总条数最多 3 条

这样可以保持 prompt 简洁，不把整张卡片全文原封不动塞给模型。

## Frontend Design

### Page Flow

建议内容任务流按以下步骤组织：

1. `materials`：上传并管理素材
2. `analysis`：展示素材分析结果
3. `reference-cases`：同行创业参考
4. `platforms`：选择目标平台
5. `generating`：提交生成
6. `drafts`：查看和编辑草稿

其中 `reference-cases` 是可跳过步骤，不是流程门槛。

### UI Structure

页面结构按已确认的线框执行：

- 顶部标题：`同行创业参考`
- 右上辅助文案：`0 已选 / 最多 3 条`
- 卡片信息层级：
  - 来源平台标签
  - 标题
  - 案例事实
  - 可借鉴点
  - 热度摘要
  - 勾选控件
- 底部按钮：
  - 次级：`返回`
  - 主级：未选中时为 `跳过，直接生成`；有选中时为 `选好了，开始生成`

### Frontend State

建议页面维护以下状态：

- `task`
- `assets`
- `referenceCases`
- `selectedReferenceCaseIds`
- `referenceCaseFetchStatus`
- `referenceCaseFetchError`
- `isRefreshingReferenceCases`
- `selectedPlatforms`
- `isGenerating`

状态规则：

- 首次进入该步骤自动拉取一次候选
- 手动刷新时不清空当前已选，除非后端明确返回 id 已失效
- 生成失败时保留已选案例和平台，允许一键重试

### Empty, Loading, Error

必须覆盖 4 种显示状态：

1. `loading`：展示骨架或轻量加载提示
2. `ready + items`：展示卡片
3. `ready + no items`：展示“暂无可引用同行参考”
4. `failed/stale`：展示失败提示，但底部仍允许直接生成

禁止做法：

- 只做成功态
- 失败时整页白屏
- 为了“看起来完整”而回退到 AI 生成假案例

## Validation Rules

### Backend Validation

- 任务必须已有 3 到 9 张素材
- 平台数量必须在 1 到 4 之间
- `reference_case_ids` 数量不得超过 3
- 非当前任务候选的 `reference_case_ids` 必须拒绝
- `hotspot_ids` 可继续做合法性校验，但不再做“存在候选就必须至少选 1 条”的限制

### Frontend Validation

- 第 4 个案例不可被选中
- 当已选满 3 条时，其余卡片置灰或点击无效
- “跳过，直接生成”必须提交空 `reference_case_ids`
- “选好了，开始生成”必须提交当前已选 id 列表

## Testing and Verification

### Backend

至少新增以下测试：

- 刷新案例成功并正确归一化字段
- provider 失败但存在缓存时返回 `stale`
- provider 失败且无缓存时返回 `failed`
- `reference_case_ids=[]` 时仍可触发生成
- `reference_case_ids` 为 1 到 3 条时可正常入库
- `reference_case_ids` 超过 3 条时拒绝
- 非候选池 id 拒绝
- 生成 prompt 中包含选中的案例角度

建议最小命令：

```bash
cd backend
uv run pytest -q tests/modules/test_promotion_generation_service.py
uv run pytest -q tests/modules/test_promotion_reference_case_service.py
```

### Frontend

至少完成：

- 组件交互自查
- `npm run lint`

如果实现包含新页面注册、路由配置或 Taro 页面入口调整，再升级验证到：

```bash
cd frontend
npm run build:h5
```

## Documentation Updates

实现时需要同步更新：

- `README.md`
- `backend/.env.example` 或当前实际环境模板
- 前端内容任务相关 README 或说明文档

新增配置建议包括：

- `REFERENCE_CASE_PROVIDER_MODE`
- `REFERENCE_CASE_PROVIDER_API_URL`
- `REFERENCE_CASE_PROVIDER_API_KEY`
- `REFERENCE_CASE_PROVIDER_TIMEOUT_SECONDS`
- `REFERENCE_CASE_PROVIDER_OPENAI_MODEL_ID`

## Rollout Notes

建议按以下顺序实施：

1. 后端数据模型与 refresh 接口
2. 生成接口扩展与兼容校验调整
3. prompt 注入与后端测试
4. 前端内容任务页与参考卡片步骤
5. 文档、配置与最小验证收尾

这样可以先打通“可跳过但可用”的后端能力，再接前端页面，不会因为 UI 未完成卡住核心能力。
