# LOG-00008｜对外验证码与邮件读取开放 API — 执行日志

## 目标

- 在不破坏现有内部接口的前提下，实现 `/api/external/*` 开放接口（仅 `X-API-Key` 鉴权）
- 按既有测试用例驱动落地实现，并通过全量回归测试

## 执行记录（2026-03-08）

### 1) Settings 接入 external_api_key + API Key 鉴权

- 新增 settings 仓库能力：
  - `get_external_api_key()`：读取并自动解密（兼容明文历史数据）
  - `get_external_api_key_masked()`：脱敏展示
- 扩展 `GET /api/settings` 返回：
  - `external_api_key_set`
  - `external_api_key_masked`
- 扩展 `PUT /api/settings` 支持保存/清空 `external_api_key`（加密存储，不回显明文）
- 新增 `api_key_required()`：
  - Header 仅接受 `X-API-Key`
  - 未配置 `external_api_key` → `403 API_KEY_NOT_CONFIGURED`
  - 缺失/错误 → `401 UNAUTHORIZED`
- 数据库初始化补齐默认 setting：`external_api_key=""`（幂等）

验证：
- `python -m unittest tests.test_settings_external_api_key -v` ✅

### 2) 参数化验证码/验证链接提取器

- 在 `verification_extractor.py` 新增 `extract_verification_info_with_options()`：
  - `code_regex`：自定义正则优先
  - `code_length`：如 `6-6`（仅数字验证码）
  - `code_source`：`subject/content/html/all`
  - 验证链接优先关键词：`verify/confirm/activate/validation`
- 返回增强字段：
  - `verification_link`
  - `match_source`
  - `confidence`

验证：
- `python -m unittest tests.test_verification_extractor_options -v` ✅

### 3) 开放 API（/api/external/*）落地

- 新增 `outlook_web/services/external_api.py` 作为开放接口编排层：
  - `ok()/fail()` 统一响应结构
  - 自定义异常统一映射错误码与 HTTP 状态码
  - Graph → IMAP(New) → IMAP(Old) 回退（列表/详情）
  - `wait-message` 轮询实现（最大超时 120 秒，sleep 可被 mock）
  - 访问审计：写入 `audit_logs(resource_type="external_api")`
- 注册路由：
  - `outlook_web/routes/system.py`：`/api/external/health|capabilities|account-status`
  - `outlook_web/routes/emails.py`：`/api/external/messages|messages/latest|messages/{id}|raw|verification-code|verification-link|wait-message`
- 控制器实现：
  - `outlook_web/controllers/system.py`
  - `outlook_web/controllers/emails.py`
- 补齐 RAW 内容：
  - Outlook OAuth IMAP（XOAUTH2）详情返回 `raw_content`
  - 标准 IMAP(Generic) 详情返回 `raw_content`

验证：
- `python -m unittest tests.test_external_api -v` ✅

### 4) 全量回归

验证：
- `python -m unittest discover -s tests -v` ✅（253 tests）

---

## 执行记录（2026-03-09）

### 5) 对齐 OpenAPI/PRD：System 接口返回字段补齐

- `GET /api/external/capabilities`
  - 补齐 `version` 字段（与 OpenAPI `CapabilitiesData.required=[service, version, features]` 对齐）
- `GET /api/external/account-status`
  - 补齐 `account_type` / `provider` / `can_read`
  - 增强 `email` 参数校验：为空或不包含 `@` → `400 INVALID_PARAM`
- 补齐 OpenAPI 文档：`HealthData` 增加 `version`（与 PRD “健康检查返回版本号” 对齐）

验证：
- `python -m unittest tests.test_external_api.ExternalApiSystemTests -v` ✅
- `python -m unittest tests.test_external_api -v` ✅
- `python -m unittest tests.test_settings_external_api_key -v` ✅
- `python -m unittest tests.test_verification_extractor_options -v` ✅
- `python -m unittest discover -s tests -v` ✅（253 tests）

### 6) 前端设置页集成 external_api_key（UI 配置闭环）

- `templates/index.html`
  - 设置页新增“对外开放 API Key”输入框（不回显明文，仅用于写入/清空）
- `static/js/main.js`
  - `loadSettings()` 使用 `*_api_key_masked` 回填脱敏值，并记录 `dataset.maskedValue/isSet`
  - `saveSettings()` 避免把脱敏占位符写回 DB；外部 API Key 支持清空禁用
- `outlook_web/controllers/settings.py`
  - `PUT /api/settings` 对 `gptmail_api_key` / `external_api_key` 增加“脱敏占位符不覆盖”保护
  - `gptmail_api_key` 支持清空（与前端行为一致）

验证：
- `python -m unittest discover -s tests -v` ✅（256 tests）

### 7) 手工冒烟测试（启动服务验证端到端链路）

启动方式（禁用调度器自启 + 使用临时 DB，避免污染本地 `data/`）：

```powershell
$env:SECRET_KEY="test-secret-key-32bytes-minimum-0000000000000000"
$env:LOGIN_PASSWORD="testpass123"
$env:SCHEDULER_AUTOSTART="false"
$env:DATABASE_PATH=(Join-Path $env:TEMP "outlookEmail-manual-smoke.db")
python -m flask --app web_outlook_app run --host 127.0.0.1 --port 5055
```

验证步骤（均通过）：
- 登录：`POST /login` → 200
- 获取 CSRF：`GET /api/csrf-token` → 200
- 写入对外 API Key：`PUT /api/settings {"external_api_key":"abc123"}` → 200
- 读取设置脱敏：`GET /api/settings` → `external_api_key_set=true` 且 `external_api_key_masked` 非明文
- 外部健康检查：`GET /api/external/health`（Header `X-API-Key: abc123`）→ 200 `code=OK`

## 执行记录（2026-03-10）

### 8) 对齐 PRD 语义偏差并补强回归用例

- 修复 `verification-code` / `verification-link`
  - 未显式传入 `since_minutes` 时，默认按 PRD 使用最近 `10` 分钟窗口
  - 避免命中过期历史验证码/历史验证链接
- 修复 `wait-message`
  - 由“存在匹配邮件即返回”改为“仅当调用开始后出现的新邮件才返回”
  - 保留超时上限 `120` 秒与同步轮询实现
- 修复 Graph 链路 RAW 内容
  - 新增 Graph MIME 读取函数，优先使用 `/me/messages/{id}/$value`
  - `messages/{id}` 与 `messages/{id}/raw` 在 Graph 成功时返回真实 MIME 内容，而非仅正文
- 补齐审计日志闭环
  - 新增 `/api/external/messages/{id}/raw`
  - 新增 `/api/external/health`
  - 新增 `/api/external/capabilities`
  - 新增 `/api/external/account-status`
- 补充回归测试
  - `messages/latest` 筛选与最新命中
  - `verification-*` 默认 10 分钟窗口
  - `wait-message` 仅返回新邮件
  - Graph RAW 内容与外部审计日志写入

验证：
- `python -m unittest tests.test_external_api -v` ✅
- `python -m unittest tests.test_settings_external_api_key -v` ✅
- `python -m unittest tests.test_verification_extractor_options -v` ✅
- `python -m unittest discover -s tests -v` ✅（256 tests）

## 执行记录（2026-03-12）

### 9) P1 公网安全控制层已实现并完成回归确认

- 已落地公网模式控制相关配置与默认值：
  - `external_api_public_mode`
  - `external_api_ip_whitelist`
  - `external_api_rate_limit_per_minute`
  - `external_api_disable_wait_message`
  - `external_api_disable_raw_content`
- 已新增独立守卫层 `outlook_web/security/external_api_guard.py`
  - `public_mode=false` 时完全透传，保持 P0 行为
  - `public_mode=true` 时启用 IP 白名单、分钟级限流、高风险端点禁用
- `capabilities` 已补齐：
  - `public_mode`
  - `restricted_features`
- 设置页已支持公网模式配置读写与说明文案

验证：
- `python -m unittest tests.test_external_api -v` ✅
  - 覆盖私有模式透传、白名单拒绝/放行、限流命中、高风险端点禁用、P1 设置读写

### 10) P2 首版异步探测已实现并完成回归确认

- `wait-message` 已支持双模式：
  - `mode=sync`：保持原有阻塞等待
  - `mode=async`：创建探测任务并立即返回 `202`
- 已新增：
  - `/api/external/probe/<probe_id>`
  - `external_probe_cache` 状态表
  - scheduler 中的后台探测轮询与过期清理任务
- 当前 P2 的实现边界：
  - 已完成首版异步探测闭环
  - 尚未完成多 API Key、Key 级邮箱授权、调用方级配额审计
  - `health` / `account-status` 仍未升级为真实上游探测

验证：
- `python -m unittest tests.test_external_api -v` ✅
  - 覆盖 probe 创建、状态查询、后台轮询命中、超时、错误处理

### 11) 文档口径与当前实现重新对齐

- 更新 `TODO-00008` 阶段判断：
  - `P0 已完成`
  - `P1 已完成`
  - `P2 部分完成`
- 保留安全边界说明：
  - 当前仍不应将本项目按“默认公网开放平台”宣传
  - 单 `external_api_key` 仍具备读取本实例全部已配置邮箱的权限边界

### 12) 最新全量验证基线

验证：
- `python -m unittest tests.test_external_api -v` ✅（82 tests）
- `python -m unittest tests.test_settings_external_api_key -v` ✅（4 tests）
- `python -m unittest tests.test_ui_settings_external_api_key -v` ✅（2 tests）
- `python -m unittest tests.test_verification_extractor_options -v` ✅（16 tests）
- `python -m unittest discover -s tests -v` ✅（333 tests）

### 13) P1 未完成项补齐：`health` / `account-status` 真实上游探测摘要

- 新增 `external_upstream_probes` 表，持久化最近一次实例级 / 账号级上游探测结果
- `GET /api/external/health`
  - 新增 `upstream_probe_ok`
  - 新增 `last_probe_at`
  - 新增 `last_probe_error`
  - 返回实例级最近一次真实上游读取探测摘要，不主动发起 live probe
- `GET /api/external/account-status`
  - 新增 `upstream_probe_ok`
  - 新增 `probe_method`
  - 新增 `last_probe_at`
  - 新增 `last_probe_error`
  - 对目标账号执行轻量 `top=1` 列表读取探测，并缓存账号级最近结果
- 当前实现边界：
  - 属于轻量真实探测，不读取邮件详情正文
  - 保留短 TTL 缓存，避免重复请求导致接口过慢
  - 尚未升级为独立动态探测服务或更复杂的多维诊断模型

验证：
- `python -m unittest tests.test_external_api -v` ✅（83 tests）
- `python -m unittest tests.test_settings_external_api_key -v` ✅（4 tests）

### 14) P2 权限模型平台化闭环：多 API Key、邮箱范围授权、调用方级审计

- 新增 `external_api_keys` 表
  - 支持多个对外 API Key 独立存储
  - Key 明文按现有敏感配置加密逻辑存储
  - 每个 Key 支持 `name`、`enabled`、`allowed_emails`
  - 保留 `last_used_at`
- 新增 `external_api_consumer_usage_daily` 表
  - 以 `consumer_key + usage_date + endpoint` 聚合统计当日调用量
  - 记录 `total_count / success_count / error_count / last_used_at`
- `api_key_required()` 已改为：
  - 优先匹配多 Key 表中的启用 Key
  - 未命中时回退 legacy `settings.external_api_key`
  - 认证成功后写入调用方上下文（`consumer_id / consumer_name / consumer_key / allowed_emails`）
- 外部接口已接入 Key 级邮箱范围授权：
  - `/api/external/messages*`
  - `/api/external/verification-*`
  - `/api/external/wait-message`
  - `/api/external/account-status`
  - `/api/external/probe/<probe_id>`
  - 范围外统一返回 `403 EMAIL_SCOPE_FORBIDDEN`
- 审计与统计已补齐调用方维度：
  - `audit_logs.details` 增加 `consumer_id / consumer_name / consumer_key / consumer_source`
  - 每次外部调用同步写入 `external_api_consumer_usage_daily`
- 后台设置接口已扩展：
  - `GET /api/settings` 返回 `external_api_keys`、`external_api_keys_count`、`external_api_multi_key_set`
  - `PUT /api/settings` 支持全量替换 `external_api_keys`
  - 每个 Key 返回脱敏值与当日调用统计，便于后台审计

验证：
- `python -m unittest tests.test_external_api -v` ✅（91 tests）
- `python -m unittest tests.test_settings_external_api_key -v` ✅（6 tests）
- `python -m unittest tests.test_module_boundaries -v` ✅（3 tests）
- `python -m unittest discover -s tests -v` ✅（344 tests）

### 15) P2 收口修复：设置事务提交、多 Key 设置页入口、禁用 Key 语义修正

- `PUT /api/settings` 已改为事务式提交：
  - 先完成所有字段校验与变更收集
  - 再统一开启事务写入
  - 任一字段校验失败或写库失败时整单回滚，避免“返回失败但部分配置已落库”
- 修复多 Key `enabled` 的字符串布尔值解析：
  - `"false"` / `"0"` / `"off"` 现在会被正确识别为禁用
  - 仓库层 `replace_external_api_keys()` 也同步收口，避免绕过 controller 时再次误判
- 修正 `api_key_required()` 语义：
  - 当 legacy `external_api_key` 为空，且所有多 Key 都处于禁用状态时，统一返回 `403 API_KEY_NOT_CONFIGURED`
  - 若仍存在其他启用中的 Key，则错误 Key 继续返回 `401 UNAUTHORIZED`
- 设置页现已补齐多 Key 管理入口：
  - 后台“系统设置”新增 `external_api_keys` JSON 文本框
  - 支持直接维护 `id / name / api_key / enabled / allowed_emails`
  - 保留脱敏 `api_key` 占位值即可表示“不修改该 Key”

验证：
- `python -m unittest tests.test_settings_external_api_key -v` ✅（8 tests）
- `python -m unittest tests.test_external_api -v` ✅（92 tests）
- `python -m unittest tests.test_ui_settings_external_api_key -v` ✅（2 tests）
- `python -m unittest tests.test_module_boundaries -v` ✅（3 tests）
- `python -m unittest discover -s tests -v` ✅（347 tests）
