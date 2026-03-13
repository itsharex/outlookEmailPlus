# TODO-00008｜对外验证码与邮件读取开放 API — 实施待办清单

- **文档状态**: P0 已完成，P1 已完成，P2 已完成基础能力闭环，仍有平台化增强项未完成
- **版本**: V1.4
- **日期**: 2026-03-12
- **对齐 PRD**: `docs/PRD/PRD-00008-对外验证码与邮件读取开放API.md`
- **对齐 FD**: `docs/FD/FD-00008-对外验证码与邮件读取开放API.md`
- **对齐 TDD**: `docs/TDD/TDD-00008-对外验证码与邮件读取开放API.md`
- **对齐测试文档**: `docs/TEST/TEST-00008-对外验证码与邮件读取开放API-测试文档.md`
- **对齐安全架构**: `docs/FD/AD-00008-对外开放API安全架构设计.md`

---

## 1. 当前阶段判断

### 1.1 当前结论

- [x] P0 主体能力已落地
- [x] 当前 `/api/external/*` 已具备受控私有接入闭环
- [x] `X-API-Key` 鉴权、设置页配置、开放路由、开放 controller、external service、基础测试已经存在
- [x] P1 公网模式防护已落地：`public_mode`、IP 白名单、限流、高风险端点禁用、能力声明已实现
- [x] P2 `wait-message` 首版解耦已落地：`mode=async` + `/api/external/probe/{probe_id}` + scheduler 后台轮询
- [x] 当前文档口径应更新为“默认仍按受控私有接入使用，但已具备受限公网模式能力”
- [x] P1 动态上游真实探测摘要已实现，`health` / `account-status` 已返回最近探测结果
- [x] P2 多 API Key、Key 级邮箱授权、调用方级配额审计已实现基础闭环

### 1.2 当前版本可做与不可做

- [x] 可用于单实例、本地部署、多调用方受控接入的 API 接入
- [x] 可用于验证码读取、验证链接提取、邮件排查、自检联调
- [x] 可在开启 `public_mode` 后配合白名单、限流和高风险端点禁用，作为受限公网接口使用
- [ ] 不应直接作为公网开放平台对外宣传或默认部署
- [ ] 不应把 `/api/external/messages/{message_id}/raw` 与 `/api/external/wait-message` 视为公网默认可放开的接口

---

## 2. 前置准备

### 2.1 文档与范围确认

- [x] PRD 已完成
- [x] FD 已完成
- [x] TDD 已完成
- [x] API 文档已完成
- [x] BUG 文档已完成
- [x] 当前阶段目标已调整为：
  - 完成 P0/P1 已实现能力的文档与验证收口
  - 明确 P2 已完成与未完成边界
  - 将剩余事项收敛到真实分页语义、高风险接口分级与后续平台化增强项

### 2.2 基线验证

- [x] 运行现有测试：`python -m unittest discover -s tests -v`
- [x] 单独运行开放接口专项测试：
  - `python -m unittest tests.test_external_api -v`
  - `python -m unittest tests.test_settings_external_api_key -v`
  - `python -m unittest tests.test_verification_extractor_options -v`
  - `python -m unittest tests.test_ui_settings_external_api_key -v`
- [x] 记录当前通过数量，作为后续安全改造的回归基线

---

## 3. P0 收口任务

### 3.1 配置与鉴权闭环

**目标**：确认当前受控私有接入能力真正可交付，而不是只有代码存在。  
**涉及文件**：`outlook_web/security/auth.py`、`outlook_web/repositories/settings.py`、`outlook_web/controllers/settings.py`、`static/js/main.js`、`templates/index.html`

- [x] `external_api_key` 已写入 `settings` 默认项
- [x] `get_external_api_key()` / 脱敏展示能力已存在
- [x] `api_key_required()` 已使用 `X-API-Key` + `secrets.compare_digest()`
- [x] 设置页已支持录入、保存、清空 `external_api_key`
- [x] 手动验证“脱敏值不会被保存回数据库”（实测：提交脱敏值后原 key 仍有效）
- [x] 手动验证“清空后所有 `/api/external/*` 统一返回 `API_KEY_NOT_CONFIGURED`”（实测：5 个端点均返回 403）
- [x] 手动验证“错误 key / 缺失 key / 正确 key”三条路径响应与文档一致（实测：401/401/200 均正确）

### 3.2 开放接口闭环

**目标**：确认所有 P0 接口都能按文档工作。  
**涉及文件**：`outlook_web/routes/emails.py`、`outlook_web/routes/system.py`、`outlook_web/controllers/emails.py`、`outlook_web/controllers/system.py`、`outlook_web/services/external_api.py`

- [x] `/api/external/messages`
- [x] `/api/external/messages/latest`
- [x] `/api/external/messages/{message_id}`
- [x] `/api/external/messages/{message_id}/raw`
- [x] `/api/external/verification-code`
- [x] `/api/external/verification-link`
- [x] `/api/external/wait-message`
- [x] `/api/external/health`
- [x] `/api/external/capabilities`
- [x] `/api/external/account-status`
- [x] 再核对一次返回结构与 OpenAPI 是否完全一致
- [x] 再核对一次错误码与 HTTP status 是否完全一致

### 3.3 P0 设计实现关注点

- [ ] `messages` 接口当前 `has_more=False` 为固定值，后续是否要补真实分页语义
- [x] `health` 当前已返回“服务进程 + DB 可用 + 最近一次上游探测摘要”，但仍不等价于持续实时健康探测
- [x] `account-status` 当前已执行轻量真实拉信探测并返回最近探测结果，但尚未升级为独立 probe 服务
- [ ] `raw` 当前直接复用详情 service 裁剪字段，后续如做风险分级，优先在 controller 入口处理而不是新建第二套读取链路

### 3.4 P0 测试收口

**涉及文件**：`tests/test_external_api.py`、`tests/test_settings_external_api_key.py`、`tests/test_verification_extractor_options.py`、`tests/test_ui_settings_external_api_key.py`

- [x] 开放接口测试文件已存在
- [x] 设置项测试文件已存在
- [x] 提取器参数化测试文件已存在
- [x] UI 接线测试文件已存在
- [x] 增加“OpenAPI 返回字段抽样校验”测试
- [x] 增加“`raw` 仅返回裁剪字段”测试
- [x] 增加“`wait-message` 不命中旧消息”测试复核

---

## 4. P1 公网模式安全收敛

### 4.1 先做设计，再动代码

**目标**：把“受控私有接入”与“半开放公网部署”彻底分层。  
**核心主线**：鉴权收敛。

- [x] `public_mode=false` 时保持当前行为
- [x] `public_mode=true` 时新增白名单、限流、高风险接口禁用
- [x] 安全控制已落在独立 guard 模块，由 controller 入口组合调用
- [ ] 明确高风险接口分级清单：
  - `/api/external/messages/{message_id}/raw`
  - `/api/external/wait-message`

### 4.2 配置项与落库

**建议涉及文件**：`outlook_web/db.py`、`outlook_web/repositories/settings.py`、`outlook_web/controllers/settings.py`、`static/js/main.js`、`templates/index.html`

- [x] 增加 `external_api_public_mode`
- [x] 增加 `external_api_ip_whitelist`
- [x] 增加 `external_api_disable_wait_message`
- [x] 增加 `external_api_disable_raw_content`
- [x] 增加 `external_api_rate_limit_per_minute`
- [x] 设置页补充公网模式说明文案，避免误开

### 4.3 安全控制实现

**建议涉及文件**：`outlook_web/security/auth.py`、`outlook_web/controllers/emails.py`、`outlook_web/controllers/system.py`

- [x] 通过独立 `external_api_guards()` 守卫组合入口实现
- [x] 接入来源 IP 白名单校验
- [x] 接入高风险接口禁用判断
- [x] 接入基础限流能力
- [x] `api_key_required()` 仍是必经入口

### 4.4 自检接口升级

**建议涉及文件**：`outlook_web/controllers/system.py`、`outlook_web/services/external_api.py`

- [x] `/api/external/capabilities` 返回 `public_mode`
- [x] `/api/external/capabilities` 返回 `restricted_features`
- [x] `/api/external/health` 返回 `upstream_probe_ok`、`last_probe_at`、`last_probe_error`
- [x] `/api/external/account-status` 增加最近一次探测结果字段

### 4.5 P1 测试

**建议涉及文件**：`tests/test_external_api.py`

- [x] 增加 `public_mode=false` 回归测试
- [x] 增加白名单拒绝测试
- [x] 增加 `wait-message` 禁用测试
- [x] 增加 `raw` 禁用测试
- [x] 增加限流命中测试

---

## 5. P2 `wait-message` 解耦

### 5.1 先回答清楚的设计问题

- [x] 后台轮询结果当前落在 SQLite `external_probe_cache`
- [x] 当前缓存的是“按条件命中的探测结果”
- [x] 新模型保留现有 `/api/external/wait-message` 路径，并通过 `mode=async` 切换
- [x] 保留同步模式兼容，异步模式改为“创建探测 + 查询状态”

### 5.2 可能的实现拆分

**建议涉及文件**：`outlook_web/services/external_api.py`、`outlook_web/services/scheduler.py`、`outlook_web/db.py`

- [x] 已设计并落地 `external_probe_cache` 状态表
- [x] 已新增后台探测任务
- [x] 已将异步模式下的“拉信”与“HTTP 请求等待”解耦
- [ ] 同步模式仍保留 `sleep` 轮询，未完全收敛为纯状态查询模型

### 5.3 P2 风险点

- [x] 已通过 `external_probe_cache` 定义状态模型，避免直接搬运同步逻辑
- [x] 已先定义状态存储，再暴露异步接口
- [ ] 当前 `sync` / `async` 双语义并存，后续如继续平台化需明确长期收敛策略

---

## 6. 设计实现问题清单

### 6.1 现在必须定的设计点

- [x] 外部安全控制位于独立 guard 层，`auth.py` 保留鉴权，controller 入口调用组合守卫
- [x] 高风险接口当前返回 `403 FEATURE_DISABLED`
- [x] `/api/external/capabilities` 承担“当前模式声明”
- [x] `account-status` 已增加轻量真实拉信探测
  - 当前实现：使用 `top=1` 列表读取做轻量真实探测，并缓存最近探测结果
  - 当前边界：尚未升级为独立 probe 服务或更强上游诊断模型

### 6.2 已完成但暂不继续平台化扩张的点

- [x] 多 API Key
- [x] Key 级邮箱范围授权
- [x] 调用方配额审计后台
  - 当前实现：`/api/settings` 已暴露多 Key 配置与当日调用统计，设置页已提供 JSON 文本框管理入口，审计日志包含 `consumer_id/consumer_name`
  - 当前实现：设置保存已改为事务式提交；任一字段校验失败时不会出现“部分成功、部分失败”
- [ ] 开发者门户
- [ ] Webhook 推送式开放接口

---

## 7. 文档与交付待办

### 7.1 文档同步

- [x] PRD / FD / TDD / API 文档已成套存在
- [x] P1 已实现并已同步更新：
  - `docs/PRD/PRD-00008-对外验证码与邮件读取开放API.md`
  - `docs/FD/FD-00008-对外验证码与邮件读取开放API.md`
  - `docs/TDD/TDD-00008-对外验证码与邮件读取开放API.md`
  - `docs/FD/AD-00008-对外开放API安全架构设计.md`
  - `docs/api.md`
- [x] P2 首版异步探测已同步到 API 文档与实现文档
- [x] P2 收口修复已同步到 API 文档与设置页实现文档
- [ ] 如后续继续重构为纯状态查询模型，再单独补第二版解耦设计稿

### 7.2 执行日志

- [x] 已补充执行日志，记录 P1/P2 实现与验证结果
- [ ] 每个阶段结束后记录：
  - 已做功能
  - 未做功能
  - 风险
  - 回滚点

---

## 8. 发布前检查

- [x] P0 测试全量通过
- [x] 手动联调通过（QQ 邮箱 IMAP 读取成功；Outlook 因网络 SSL 受阻）
- [x] 文档与代码口径一致
- [x] README / API 文档未把当前版本描述为公网开放平台（已核查）
- [x] `raw` 与 `wait-message` 的风险说明对外可见（已在 api.md 中补充独立风险段落）
- [x] 确认当前发布状态属于：P0 收口完成，P1 完成，P2 基础能力闭环完成，但平台化增强项仍未完成

---

## 9. 阶段性结论

当前最合理的后续顺序不是继续平铺接口，而是聚焦剩余真实待办：

1. 先补齐 **`messages.has_more` 的真实分页语义**，避免调用方误把当前固定值当成完整分页能力。
2. 再收紧 **`/raw` 与 `/wait-message` 的高风险接口分级文档和部署口径**，保持“受控私有优先、受限公网谨慎开放”的边界。
3. 最后如确实需要继续平台化，再沿 **硬性 quota、开发者门户、套餐化治理** 主线推进，而不是重复建设已完成的多 Key / 范围授权 / 审计能力。

这份 TODO 的核心目的不是重复文档，而是约束后续实现顺序：

- **先收口，再加固**
- **先分层，再扩展**
- **先解决安全边界，再谈公网能力**
