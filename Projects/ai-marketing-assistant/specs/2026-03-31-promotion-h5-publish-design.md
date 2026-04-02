# Promotion H5 Publish Design

## Goal

为 promotion 内容草稿链路补充 H5 发布能力：

- 抖音：服务端生成 H5 发布跳转链接，前端直接拉起抖音 App 发布页
- 小红书：服务端生成 JS SDK 所需签名配置与分享内容，前端加载小红书 SDK 并拉起小红书 App 草稿发布页

本次能力面向现有 `backend/app/modules/promotions/` 已生成的平台草稿，不做账号绑定，不做后台静默代发，不确认最终发布成功，只负责“发起发布”。

## Scope

### In Scope

- 在现有 promotion 草稿链路下新增“发起 H5 发布”接口
- 抖音发布会话生成
- 小红书分享会话生成
- 服务端统一读取草稿标题、正文、素材 URL
- 新增轻量发布会话表用于追踪本次发布请求
- 前端根据平台类型分别执行跳转或 SDK 调用
- 补充最小 API / service 测试

### Out of Scope

- 抖音 OAuth 账号绑定
- 小红书个人号绑定
- 后台静默代发
- 发布成功后的真实作品回执同步
- 作品数据回流与统计
- 发布失败自动重试
- 通用发布中心

## Existing Context

当前 active promotion 内容链路已具备：

- 内容任务创建
- 素材上传与图片理解
- 角度推荐与选择
- 多平台草稿生成

相关入口集中在：

- `backend/app/modules/promotions/content_controller.py`
- `backend/app/modules/promotions/content_service.py`
- `backend/app/modules/promotions/generation_service.py`

平台草稿已经按 `platform_code` 落在 `mkt_promotion_platform_draft`，具备：

- `title`
- `body`
- `asset_ids`
- `edited_title`
- `edited_body`
- `edited_asset_ids`

因此本次不新建“发布前草稿”概念，而是直接消费现有平台草稿。

## Approaches

### Approach A: Publish Endpoints Under Promotion Drafts

在 promotion 草稿接口下为不同平台新增发布会话接口，第三方平台差异下沉到独立 provider。

优点：

- 延续现有 active promotion 内容链路
- 前端改造最小，直接在草稿卡片上接“发布”
- 第三方平台细节与业务控制器解耦

缺点：

- 当前阶段能力只服务 promotion 草稿，暂不具备仓库级通用复用性

### Approach B: Expand Existing `content_service.py`

将发布逻辑直接并入 `content_service.py`。

优点：

- 初始文件更少

缺点：

- `content_service.py` 已承担任务、素材、角度、生成编排，继续扩展会加速膨胀
- 后续补发布记录、回调、重试时会继续失控

### Approach C: New Generic Publish Center

抽象成独立发布中心模块，供 promotion 与其他内容链路共享。

优点：

- 长期复用性最好

缺点：

- 超出本次需求范围
- 当前没有第二个 active 内容发布来源，容易过度设计

### Recommendation

采用 Approach A：

- API 入口继续放在 promotion 草稿下面
- 第三方平台适配逻辑抽到独立 publish service/provider
- 会话追踪单独存表，避免后续扩展时重构路由和状态模型

## API Contract

### Douyin Publish Session

Endpoint:

- `POST /mkt/api/v1/promotion/stores/{store_id}/content-tasks/{task_id}/drafts/{draft_id}/publish/douyin-session`

Request Body:

```json
{
  "redirect_url": "https://your-h5-domain/publish/result",
  "client_context": {
    "source": "promotion_h5"
  }
}
```

Response Body:

```json
{
  "platform": "douyin",
  "mode": "redirect",
  "publish_url": "snssdk1128://...",
  "session_id": "2f3ab5c6-5d70-4fc8-a45a-7a1cc73a8f32",
  "expires_at": "2026-03-31T16:00:00Z"
}
```

说明：

- `publish_url` 为前端直接跳转的抖音发布链接或 schema
- `session_id` 供前端埋点和后续状态页使用
- 本次不返回用户账号信息

### Xiaohongshu Publish Session

Endpoint:

- `POST /mkt/api/v1/promotion/stores/{store_id}/content-tasks/{task_id}/drafts/{draft_id}/publish/xiaohongshu-session`

Request Body:

```json
{
  "redirect_url": "https://your-h5-domain/publish/result",
  "client_context": {
    "source": "promotion_h5"
  }
}
```

Response Body:

```json
{
  "platform": "xiaohongshu",
  "mode": "sdk",
  "sdk_script_url": "https://fe-static.xhscdn.com/biz-static/goten/xhs-1.0.1.js",
  "session_id": "b38d3740-8e3f-4c58-b2bc-65445ff02e80",
  "verify_config": {
    "app_key": "xxx",
    "nonce": "xxx",
    "timestamp": 1760000000,
    "signature": "xxx"
  },
  "share_info": {
    "type": "normal",
    "title": "标题",
    "content": "正文",
    "images": [
      "https://example.com/a.jpg",
      "https://example.com/b.jpg"
    ]
  },
  "expires_at": "2026-03-31T16:00:00Z"
}
```

说明：

- 前端负责加载 SDK 并调用 `window.xhs.share(...)`
- `verify_config.signature` 的计算规则以你们已持有的小红书官方文档为准
- `sdk_script_url` 默认来自服务端配置，避免前端硬编码

## Service Flow

统一服务编排放在新文件：

- `backend/app/modules/promotions/publish_service.py`

第三方平台适配层放在：

- `backend/app/infrastructure/platform_publish/douyin.py`
- `backend/app/infrastructure/platform_publish/xiaohongshu.py`

控制器仍放在：

- `backend/app/modules/promotions/content_controller.py`

### Common Flow

1. 校验 `store_id + task_id + draft_id` 归属关系
2. 校验草稿存在且未软删
3. 校验草稿标题、正文不为空
4. 读取草稿关联素材
5. 优先使用编辑后的标题、正文、素材
6. 创建 `mkt_promotion_publish_session`
7. 调用对应平台 provider 生成发布会话
8. 更新会话快照并返回给前端

### Draft Content Resolution

字段优先级固定为：

- 标题：`edited_title` > `title`
- 正文：`edited_body` > `body`
- 素材：`edited_asset_ids` > `asset_ids`

若最终标题或正文为空，直接视为不可发布。

## Platform Behavior

### Douyin

服务端负责：

- 准备标题、正文、素材 URL
- 调用抖音 H5 发布能力生成跳转链接
- 返回 `publish_url`

前端负责：

- 点击按钮
- 获取 `publish_url`
- 直接跳转

本次不要求用户先在本系统绑定抖音账号，也不校验最终抖音 App 当前登录账号与系统用户的关系。

### Xiaohongshu

服务端负责：

- 准备标题、正文、图片 URL
- 生成 `verify_config`
- 返回 `sdk_script_url + verify_config + share_info`

前端负责：

- 动态加载小红书 JS SDK
- 等待 `window.xhs.ready()`
- 调用 `window.xhs.share({ shareInfo, verifyConfig })`

本次默认依赖你们已持有的小红书官方 H5 SDK 接入资质与签名规则文档。

## Data Model

新增表：

- `mkt_promotion_publish_session`

建议字段：

- `id UUID PRIMARY KEY`
- `org_id UUID NOT NULL`
- `store_id UUID NOT NULL`
- `task_id UUID NOT NULL`
- `draft_id UUID NOT NULL`
- `platform_code VARCHAR(32) NOT NULL`
- `session_status VARCHAR(24) NOT NULL`
- `redirect_url VARCHAR(512) NULL`
- `payload_snapshot JSONB NOT NULL DEFAULT '{}'::jsonb`
- `error_message TEXT NULL`
- `expired_at TIMESTAMP NOT NULL`
- `created_at TIMESTAMP NOT NULL`
- `updated_at TIMESTAMP NOT NULL`
- `deleted_at TIMESTAMP NULL`

字段说明：

- `session_status` 推荐值：`created`、`ready`、`opened`、`completed`、`failed`、`expired`
- `payload_snapshot` 保存第三方返回的摘要，不保存敏感秘钥
- 本次不单独保存“真实作品ID”，后续若接回执再扩展

建表方式：

- 按仓库当前约束使用原生 SQL bootstrap
- 不依赖 Alembic

## Error Handling

控制器层错误语义：

- `404 Draft not found`
- `409 Draft is not publishable`
- `503 Publish session unavailable`

典型失败场景：

- 草稿平台与调用平台不匹配
- 草稿标题/正文为空
- 没有关联素材
- 抖音 `client_token` 或发布链接生成失败
- 小红书签名配置缺失
- 小红书签名生成失败

日志要求：

- 使用 `logger.bind(module="promotions")`
- 至少记录 `store_id`、`task_id`、`draft_id`、`platform_code`、`stage`
- 不记录完整 access token、secret、signature 明文

## Configuration

### Douyin

- `DOUYIN_OPEN_CLIENT_KEY`
- `DOUYIN_OPEN_CLIENT_SECRET`
- `DOUYIN_OPEN_BASE_URL`
- `DOUYIN_OPEN_H5_PUBLISH_EXPIRE_SECONDS`

### Xiaohongshu

- `XHS_H5_APP_KEY`
- `XHS_H5_APP_SECRET`
- `XHS_H5_SDK_SCRIPT_URL`
- `XHS_H5_SHARE_EXPIRE_SECONDS`

### Shared

- `PUBLISH_RESULT_BASE_URL`
- `PUBLISH_SESSION_TTL_SECONDS`

新增配置后需同步更新：

- 读取位置
- `.env.example`
- `README.md`
- `backend/README.md`

## Frontend Impact

前端在 promotion 草稿页为每个已生成的平台草稿增加“发布”入口：

- 抖音草稿：按钮文案可为“发布到抖音”
- 小红书草稿：按钮文案可为“发布到小红书”

处理差异：

- 抖音：收到 `mode=redirect` 后直接跳转
- 小红书：收到 `mode=sdk` 后加载 SDK 并调用分享方法

前端不负责：

- 自己拼第三方签名
- 自己选择草稿内容优先级
- 自己硬编码 SDK URL

## Testing

### API Tests

- 抖音发布会话创建成功
- 小红书发布会话创建成功
- 草稿不存在返回 `404`
- 草稿平台不匹配返回 `409`
- 第三方 provider 失败返回 `503`

### Service Tests

- 优先读取编辑后的标题/正文/素材
- 抖音 provider 返回 `publish_url`
- 小红书 provider 返回 `verify_config + share_info`
- 发布会话表写入成功
- 过期时间按配置计算正确

## Risks

- 小红书 H5 SDK 签名规则若与当前内部资料不一致，会导致前端无法拉起 App
- 抖音和小红书最终发布动作都发生在宿主 App 内，服务端无法天然确认“用户一定已成功发布”
- 用户取消发布或切换 App 登录账号，不在本次服务端强校验范围内

## Non-Goals for This Iteration

- 统一发布记录中心
- 发布成功回执同步
- 平台账号绑定
- 作品数据统计回流
- 自动重试与失败补偿
