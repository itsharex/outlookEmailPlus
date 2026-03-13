# TDD-00008｜对外验证码与邮件读取开放 API — 技术设计细节文档

- **文档编号**: TDD-00008
- **创建日期**: 2026-03-08
- **版本**: V1.0
- **状态**: 草案
- **对齐 PRD**: `docs/PRD/PRD-00008-对外验证码与邮件读取开放API.md`
- **对齐 FD**: `docs/FD/FD-00008-对外验证码与邮件读取开放API.md`
- **对齐 OpenAPI**: `docs/FD/OPENAPI-00008-对外验证码与邮件读取开放API.yaml`
- **前置依赖**: `docs/TDD/TDD-00005-多邮箱统一管理.md`（账号结构、Graph/IMAP 链路、模块化 Blueprint 架构）

---

## 目录

1. [文档目的](#1-文档目的)
2. [设计原则与硬约束](#2-设计原则与硬约束)
3. [总体技术架构与数据流](#3-总体技术架构与数据流)
4. [文件变更清单](#4-文件变更清单)
5. [设置与鉴权技术细节](#5-设置与鉴权技术细节)
6. [Route 层技术细节](#6-route-层技术细节)
7. [Controller 层技术细节](#7-controller-层技术细节)
8. [External Service 层技术细节](#8-external-service-层技术细节)
9. [邮件读取与回退链路细节](#9-邮件读取与回退链路细节)
10. [验证码与验证链接提取细节](#10-验证码与验证链接提取细节)
11. [统一响应与错误码映射](#11-统一响应与错误码映射)
12. [系统自检接口实现细节](#12-系统自检接口实现细节)
13. [前端设置页改造细节](#13-前端设置页改造细节)
14. [兼容性与回归保障](#14-兼容性与回归保障)
15. [测试策略与测试用例](#15-测试策略与测试用例)
16. [实施顺序建议](#16-实施顺序建议)

---

## 1. 文档目的

本 TDD 描述 PRD-00008「对外验证码与邮件读取开放 API」的**完整技术实现细节**，重点回答：

- 如何在不破坏现有内部接口的前提下，新增一套 `X-API-Key` 鉴权的开放接口
- 如何把现有内部邮件读取能力抽象成可复用的开放 Service，而不是继续堆叠在 controller 中
- 如何实现验证码与验证链接的**可配置提取规则**（`code_regex` / `code_length` / `code_source`）
- 如何保证开放接口、设置页、审计日志、健康检查三者闭环
- 如何让 OpenAPI 草稿中的字段、错误码、返回结构与真实代码实现保持一致

---

## 2. 设计原则与硬约束

### 2.1 API 与模块边界硬约束

- **新增路由前缀固定**：所有开放接口统一以 `/api/external` 开头
- **不修改现有内部接口 URL**：
  - `GET /api/emails/<email_addr>`
  - `GET /api/emails/<email_addr>/extract-verification`
  - `GET /api/email/<email_addr>/<message_id>`
  保持不变
- **开放接口不依赖 Session**：只能走 `X-API-Key`，不能走 `login_required`
- **开放接口响应结构统一**：固定为 `success/code/message/data`
- **开放接口首版仅开放读能力**：不新增外部删除、移动、已读等写操作

### 2.2 数据与安全硬约束

- **不新增数据库表**：复用 `settings`、`accounts`、`groups`、`audit_logs`
- **`external_api_key` 建议加密存储**：使用现有 `encrypt_data()` / `decrypt_data()`
- **设置页不回显明文 API Key**：仅返回 `*_set`、`*_masked`
- **日志中不允许输出明文 API Key**
- **查询参数中不接受 `api_key`**：根项目不沿用示例项目的 query 传 key 方案
- **首版不以公网开放为目标验收**：只保证本地化部署、单可信调用方、受控访问环境下的可用性
- **首版不内建开放平台级防护**：来源白名单、调用方配额、多 API Key 隔离不在本次实现闭环内

### 2.3 向后兼容原则

- `outlook_web/controllers/emails.py` 中现有内部 API 行为与响应结构不变
- `outlook_web/controllers/settings.py` 中现有设置项行为不变，仅扩展开放 API Key 字段
- Graph → IMAP(New) → IMAP(Old) 的读取回退顺序保持不变
- 现有 `verification_extractor.py` 默认提取逻辑不破坏，仅在外部接口场景补充参数化入口

### 2.4 对公网暴露的阶段性约束

- 当前 `/api/external/*` 的设计定位为**受控私有接入接口**，不等价于“可直接公网开放的通用 API 平台”
- `wait-message` 使用同步轮询，请求线程会被占用；该接口在公网模式下应视为高风险接口
- `/api/external/messages/{message_id}/raw` 会返回高敏感原始内容；该接口在公网模式下应默认限制
- 若后续引入公网模式，应在应用层增加：
  - `public_mode` 配置
  - 来源 IP 白名单
  - 高风险接口禁用或降级
  - 动态上游探测结果返回

---

## 3. 总体技术架构与数据流

### 3.1 获取验证码主链路

```text
客户端
  ↓ GET /api/external/verification-code
Route: outlook_web/routes/emails.py
  ↓
Security: api_key_required
  ↓
Controller: api_external_get_verification_code()
  ↓
Repository: accounts_repo.get_account_by_email(email)
  ↓
Service: external_api.get_latest_message_for_external(...)
  ↓
Service: list_messages_for_external(...) 内部执行 Graph / IMAP 回退
  ├─ graph_service.get_emails_graph(...)
  ├─ imap_service.get_emails_imap_with_server(..., IMAP_SERVER_NEW)
  └─ imap_service.get_emails_imap_with_server(..., IMAP_SERVER_OLD)
  ↓
Service: get_message_detail_for_external(...) 内部执行详情读取回退
  ↓
Service: verification_extractor.extract_verification_info_with_options(...)
  ↓
Service: ok()/fail() 统一包装
  ↓
Audit: external_api.audit_external_api_access(...)
  ↓
返回 JSON
```

### 3.2 人工排查详情链路

```text
客户端
  ↓ GET /api/external/messages/{message_id}?email=...
api_key_required
  ↓
Controller 校验 email/message_id
  ↓
Service 获取 account
  ↓
Service 读取详情（Graph 优先，IMAP 回退）
  ↓
Service 组装 MessageDetail
  ↓
返回统一响应
```

### 3.3 设置页配置链路

```text
前端 settings 页面
  ↓ GET /api/settings
controllers/settings.py
  ↓
repositories/settings.py
  ↓
返回 external_api_key_set / external_api_key_masked

前端保存 settings
  ↓ PUT /api/settings
controllers/settings.py
  ↓
encrypt_data(external_api_key)
  ↓
repositories/settings.py.set_setting('external_api_key', encrypted)
```

---

## 4. 文件变更清单

| 文件 | 变更类型 | 说明 |
|---|---|---|
| `outlook_web/security/auth.py` | **修改** | 新增 `api_key_required()` |
| `outlook_web/repositories/settings.py` | **修改** | 新增 `get_external_api_key()`、`get_external_api_key_masked()` |
| `outlook_web/controllers/settings.py` | **修改** | 扩展 `external_api_key` 读写与脱敏展示 |
| `outlook_web/db.py` | **修改** | 初始化默认 `external_api_key` setting |
| `outlook_web/routes/emails.py` | **修改** | 注册开放消息与验证接口 |
| `outlook_web/routes/system.py` | **修改** | 注册开放系统接口 |
| `outlook_web/controllers/emails.py` | **修改** | 新增开放 API 控制器函数 |
| `outlook_web/controllers/system.py` | **修改** | 新增开放健康检查/能力/账号状态函数 |
| `outlook_web/services/external_api.py` | **新增** | 外部接口专用 service 层 |
| `outlook_web/services/verification_extractor.py` | **修改** | 增强参数化提取入口 |
| `templates/index.html` | **修改** | 设置页增加开放 API Key 配置区块 |
| `static/js/main.js` | **修改** | 读取/保存开放 API Key |
| `tests/test_external_api.py` | **新增** | 契约/集成测试 |
| `tests/test_settings_external_api_key.py` | **新增/可选** | 设置项测试 |

### 4.1 文件级真实技术改造点

为了避免 TDD 只停留在抽象分层，这里按“文件 -> 需要真实修改什么”进一步展开。

| 文件 | 真实修改点 | 技术细节 |
|---|---|---|
| `outlook_web/db.py` | 初始化默认设置项 | 在 `init_db()` 的 settings 初始化逻辑中补 `external_api_key`，保持幂等，不增加 schema 版本 |
| `outlook_web/repositories/settings.py` | 封装开放 API Key 读取能力 | 新增 `get_external_api_key()`、`get_external_api_key_masked()`，读取时统一走 `decrypt_data()` |
| `outlook_web/security/auth.py` | 新增外部鉴权装饰器 | 增加 `api_key_required()`，只接受 `X-API-Key`，用 `secrets.compare_digest()` 做常量时间比较 |
| `outlook_web/controllers/settings.py` | 扩展设置读写 | `GET /api/settings` 返回 `external_api_key_set`、`external_api_key_masked`；`PUT /api/settings` 支持加密保存与清空 |
| `outlook_web/routes/emails.py` | 注册外部消息与验证接口 | 在现有 emails blueprint 下追加 `/api/external/messages*`、`/verification-*`、`/wait-message` |
| `outlook_web/routes/system.py` | 注册外部系统接口 | 追加 `/api/external/health`、`/capabilities`、`/account-status` |
| `outlook_web/controllers/emails.py` | 增加外部 controller 入口 | 增加参数解析、调用 external service、统一错误响应、外部审计日志 |
| `outlook_web/controllers/system.py` | 增加外部自检接口 | 对外返回轻量健康状态、能力列表、账号状态，不复用登录态接口 |
| `outlook_web/services/external_api.py` | 新建编排层 | 承担账号校验、Graph/IMAP 回退、列表过滤、详情读取、验证码提取结果组装、等待轮询 |
| `outlook_web/services/verification_extractor.py` | 增强提取器参数化能力 | 增加 `extract_verification_info_with_options()`，支持 `code_regex` / `code_length` / `code_source` |
| `templates/index.html` | 增加设置页入口 | 新增 `settingsExternalApiKey` 输入框和 `externalApiKeyHint` 提示文案区域 |
| `static/js/main.js` | 对接设置页读写 | `loadSettings()` 回填脱敏值；`saveSettings()` 通过 `dataset.maskedValue` / `dataset.isSet` 避免把占位符写回 DB |
| `tests/test_external_api.py` | 覆盖主链路测试 | 覆盖鉴权、列表、详情、RAW、验证码、验证链接、wait-message、system 接口 |
| `tests/test_settings_external_api_key.py` | 覆盖设置与加密兼容 | 覆盖设置保存、脱敏回显、明文/加密兼容与清空行为 |

### 4.2 文件之间的技术依赖

这些文件不是独立修改，而是一条完整技术链路：

```text
db.py
  ↓ 提供 external_api_key 默认配置
repositories/settings.py
  ↓ 提供读取 / 解密 / 脱敏能力
security/auth.py
  ↓ 读取 external_api_key 做鉴权
controllers/settings.py
  ↓ 提供管理员配置入口
routes/*.py + controllers/*.py
  ↓ 暴露 /api/external/*
services/external_api.py
  ↓ 编排 graph / imap / extractor
tests/*
  ↓ 验证整条链路
```

关键结论：

- `api_key_required()` 不能脱离 `repositories/settings.py` 独立实现。
- 外部 controller 不能脱离 `services/external_api.py` 直接堆读取逻辑。
- 前端设置页必须和 `controllers/settings.py` 的脱敏规则一致，否则会把占位符误写回数据库。

---

## 5. 设置与鉴权技术细节

### 5.1 `settings` 表 Key 设计

复用 `settings` 表，不新增 schema。新增 key：

| Key | 默认值 | 存储方式 | 说明 |
|---|---|---|---|
| `external_api_key` | `""` | 建议加密 | 对外开放接口使用的 API Key |

### 5.2 `db.py` 初始化逻辑

文件：`outlook_web/db.py`

在 `init_db()` 的默认 settings 初始化阶段增加：

```python
cursor.execute(
    """
    INSERT OR IGNORE INTO settings (key, value)
    VALUES ('external_api_key', '')
    """
)
```

说明：
- 不修改 `DB_SCHEMA_VERSION`
- 因为只是新增默认配置，不涉及 schema 变更
- 对既有数据库幂等生效

### 5.3 `repositories/settings.py` 技术设计

新增函数：

```python
def get_external_api_key() -> str:
    """
    获取对外 API Key。
    - 若值为空，返回空字符串
    - 若值使用 enc: 前缀加密，自动解密
    - 若值为历史明文（兼容老数据），直接返回明文
    """


def get_external_api_key_masked() -> str:
    """
    返回脱敏展示值。
    规则：前 4 位 + 若干 * + 后 4 位；长度不足时返回全 *。
    """
```

建议实现：

```python
from outlook_web.security.crypto import decrypt_data


def get_external_api_key() -> str:
    value = get_setting("external_api_key", "")
    if not value:
        return ""
    try:
        # decrypt_data() 内部已兼容 enc: 加密值和历史明文值
        return decrypt_data(value)
    except Exception:
        return ""
```

真实实现约束：

- 不在 repository 层重复判断 `enc:` 前缀，直接复用 `decrypt_data()`
- `decrypt_data()` 已兼容：
  - `enc:` 开头的加密值
  - 历史明文值
- 解密失败时返回空字符串，避免外部接口鉴权出现 500

补充说明：

- 当前 `controllers/settings.py` 实际使用的是 `get_external_api_key()` + 本地 `_mask_secret_value()`
- `get_external_api_key_masked()` 仍建议保留在 repository 层，供后续其他调用点复用

### 5.4 `api_key_required()` 技术设计

文件：`outlook_web/security/auth.py`

新增：

```python
import secrets

from outlook_web.repositories.settings import get_external_api_key


def api_key_required(f):
    @wraps(f)
    def decorated_function(*args, **kwargs):
        api_key = (request.headers.get("X-API-Key") or "").strip()
        if not api_key:
            return jsonify({
                "success": False,
                "code": "UNAUTHORIZED",
                "message": "API Key 缺失或无效",
                "data": None,
            }), 401

        stored_key = get_external_api_key()
        if not stored_key:
            return jsonify({
                "success": False,
                "code": "API_KEY_NOT_CONFIGURED",
                "message": "系统未配置对外 API Key",
                "data": None,
            }), 403

        if not secrets.compare_digest(str(api_key), str(stored_key)):
            return jsonify({
                "success": False,
                "code": "UNAUTHORIZED",
                "message": "API Key 缺失或无效",
                "data": None,
            }), 401

        return f(*args, **kwargs)
    return decorated_function
```

### 5.5 鉴权实现注意点

- 不复用 `build_error_payload()`，因为开放接口返回结构已在 OpenAPI 中固定为简化结构
- 不跳转登录页面
- 不读取 query 参数中的 `api_key`
- 所有开放接口 controller 必须直接使用 `@api_key_required`
- 使用 `secrets.compare_digest()`，避免直接字符串比较带来的时序差异
- 鉴权失败统一返回简化错误结构，不携带内部 trace 细节

### 5.6 公网模式扩展设计（v1.1 已实现）

为避免当前开放接口被误用为“可直接公网暴露”的能力，v1.1 已补充以下配置项：

| 配置项 | 默认值 | 说明 |
|---|---|---|
| `external_api_public_mode` | `false` | 是否启用公网模式 |
| `external_api_ip_whitelist` | `[]` | 允许访问 external API 的 IP / CIDR 列表 |
| `external_api_disable_wait_message` | `false` | 公网模式下可动态禁用长轮询接口 |
| `external_api_disable_raw_content` | `false` | 公网模式下可动态禁用 RAW 内容接口 |
| `external_api_rate_limit_per_minute` | `60` | 公网模式下对单 IP 或单 API Key 的分钟级限流 |

建议实现约束：

1. `public_mode=false` 时，保持当前受控私有接入行为。
2. `public_mode=true` 时，开放接口应额外经过：
   - IP 白名单检查
   - 高风险接口禁用检查
   - 请求频率限制
3. `api_key_required()` 继续保留为必经入口，不与公网模式互斥。

当前实现中已通过独立守卫层落地，形态为：

```python
def external_api_guards(feature: str | None = None):
    """
    在 public_mode 启用时执行：
    1. 白名单校验
    2. 高风险接口禁用
    3. 限流校验
    """
```

错误码建议扩展：

| code | status | 含义 |
|---|---|---|
| `IP_NOT_ALLOWED` | 403 | 来源 IP 不在白名单 |
| `FEATURE_DISABLED` | 403 | 当前模式下接口被禁用 |
| `RATE_LIMIT_EXCEEDED` | 429 | 请求频率超限 |

---

## 6. Route 层技术细节

### 6.1 `outlook_web/routes/emails.py`

在现有内部路由后追加：

```python
bp.add_url_rule(
    "/api/external/messages",
    view_func=emails_controller.api_external_get_messages,
    methods=["GET"],
)
bp.add_url_rule(
    "/api/external/messages/latest",
    view_func=emails_controller.api_external_get_latest_message,
    methods=["GET"],
)
bp.add_url_rule(
    "/api/external/messages/<path:message_id>",
    view_func=emails_controller.api_external_get_message_detail,
    methods=["GET"],
)
bp.add_url_rule(
    "/api/external/messages/<path:message_id>/raw",
    view_func=emails_controller.api_external_get_message_raw,
    methods=["GET"],
)
bp.add_url_rule(
    "/api/external/verification-code",
    view_func=emails_controller.api_external_get_verification_code,
    methods=["GET"],
)
bp.add_url_rule(
    "/api/external/verification-link",
    view_func=emails_controller.api_external_get_verification_link,
    methods=["GET"],
)
bp.add_url_rule(
    "/api/external/wait-message",
    view_func=emails_controller.api_external_wait_message,
    methods=["GET"],
)
```

说明：
- 统一放在 `emails` Blueprint 内，避免额外新建 Blueprint
- 与内部邮件能力聚合，减少初始化改动
- 后续如引入公网模式，不调整路由 URL，只在 controller / security 层增加额外约束

### 6.2 `outlook_web/routes/system.py`

追加：

```python
bp.add_url_rule(
    "/api/external/health",
    view_func=system_controller.api_external_health,
    methods=["GET"],
)
bp.add_url_rule(
    "/api/external/capabilities",
    view_func=system_controller.api_external_capabilities,
    methods=["GET"],
)
bp.add_url_rule(
    "/api/external/account-status",
    view_func=system_controller.api_external_account_status,
    methods=["GET"],
)
```

---

## 7. Controller 层技术细节

### 7.1 `controllers/emails.py` 新增函数清单

```python
@api_key_required
def api_external_get_messages() -> Any: ...

@api_key_required
def api_external_get_latest_message() -> Any: ...

@api_key_required
def api_external_get_message_detail(message_id: str) -> Any: ...

@api_key_required
def api_external_get_message_raw(message_id: str) -> Any: ...

@api_key_required
def api_external_get_verification_code() -> Any: ...

@api_key_required
def api_external_get_verification_link() -> Any: ...

@api_key_required
def api_external_wait_message() -> Any: ...
```

### 7.2 Controller 当前真实职责

1. 解析并校验参数
2. 调用 `services/external_api.py`
3. 写审计日志并返回统一响应

后续如引入公网模式，可扩展为四件事：

1. 执行 `@api_key_required`
2. 执行 `enforce_external_api_public_controls()`
3. 解析参数并调用 service
4. 审计并返回统一响应

### 7.3 当前真实参数解析辅助函数

当前真实实现不是文档早期草稿中的 `_parse_external_args()`，而是：

```python
def _parse_external_common_args(*, default_since_minutes: int | None = None) -> dict:
    email_addr = (request.args.get("email") or "").strip()
    if not email_addr or "@" not in email_addr:
        raise external_api_service.InvalidParamError("email 参数无效")

    folder = (request.args.get("folder") or "inbox").strip().lower() or "inbox"
    if folder not in {"inbox", "junkemail", "deleteditems"}:
        raise external_api_service.InvalidParamError("folder 参数无效")

    def _int_arg(name: str, default: int) -> int:
        raw = request.args.get(name, None)
        if raw is None or raw == "":
            return default
        try:
            return int(raw)
        except Exception as exc:
            raise external_api_service.InvalidParamError(f"{name} 参数无效") from exc

    skip = _int_arg("skip", 0)
    top = _int_arg("top", 20)
    if skip < 0:
        raise external_api_service.InvalidParamError("skip 参数无效")
    if top < 1 or top > 50:
        raise external_api_service.InvalidParamError("top 参数无效")

    since_minutes_raw = request.args.get("since_minutes", None)
    since_minutes = default_since_minutes
    if since_minutes_raw not in (None, ""):
        try:
            since_minutes = int(since_minutes_raw)
        except Exception as exc:
            raise external_api_service.InvalidParamError("since_minutes 参数无效") from exc
        if since_minutes < 1:
            raise external_api_service.InvalidParamError("since_minutes 参数无效")

    return {
        "email": email_addr,
        "folder": folder,
        "skip": skip,
        "top": top,
        "from_contains": (request.args.get("from_contains", "") or "").strip(),
        "subject_contains": (request.args.get("subject_contains", "") or "").strip(),
        "since_minutes": since_minutes,
    }
```

补充说明：

- `verification-code`、`verification-link` 会调用 `_parse_external_common_args(default_since_minutes=10)`，默认只看最近 10 分钟邮件。
- `messages`、`messages/latest`、`message_detail`、`raw`、`wait-message` 调用 `_parse_external_common_args()`，默认不强制 `since_minutes`。
- `code_length`、`code_regex`、`code_source`、`timeout_seconds`、`poll_interval` 由各自 controller 单独解析，不放进通用解析器。

### 7.4 参数校验约束

| 参数 | 校验规则 |
|---|---|
| `email` | 必填，必须含 `@` |
| `folder` | `inbox/junkemail/deleteditems` |
| `top` | `1-50` |
| `skip` | `>=0` |
| `since_minutes` | `>=1` |
| `timeout_seconds` | `1-120` |
| `poll_interval` | `>=1` 且 `<= timeout_seconds` |
| `code_source` | `subject/content/html/all` |
| `code_length` | 匹配 `^\d+-\d+$` |

### 7.5 审计日志真实实现约定

当前 controller 并不直接调用 `log_audit(...)`，而是统一调用：

```python
external_api_service.audit_external_api_access(
    action="external_api_access",
    email_addr=email_addr,
    endpoint="/api/external/messages",
    status="ok",
    details={"method": method, "count": len(filtered)},
)
```

`audit_external_api_access()` 在 `services/external_api.py` 中再统一做两件事：

1. 组装 `{"endpoint": ..., "status": ..., ...}` 的安全日志对象
2. 调用 `log_audit(action, "external_api", email_addr, details_text)`

真实实现约束：

- controller 成功和失败分支都会记审计日志
- 失败时一般只写 `code` 与异常类型，不写敏感请求体
- service 层已明确标注“避免日志中输出敏感信息（如 API Key）”

---

## 8. External Service 层技术细节

### 8.1 新增文件：`outlook_web/services/external_api.py`

该模块负责承接开放接口通用逻辑，避免 `controllers/emails.py` 继续膨胀。

### 8.2 模块函数清单（真实实现）

```python
def ok(data, message: str = "success") -> dict:
    return {"success": True, "code": "OK", "message": message, "data": data}


def fail(code: str, message: str, *, data=None) -> dict:
    return {"success": False, "code": code, "message": message, "data": data}


def require_account(email_addr: str) -> dict: ...
def list_messages_for_external(... ) -> tuple[list[dict], str]: ...
def filter_messages(... ) -> list[dict]: ...
def get_latest_message_for_external(... ) -> dict: ...
def get_message_detail_for_external(... ) -> dict: ...
def get_verification_result(... ) -> dict: ...
def wait_for_message(... ) -> dict: ...
def audit_external_api_access(... ) -> None: ...
```

需要明确：

- 当前没有单独的 `get_message_raw_for_external()`，`/raw` controller 直接复用 `get_message_detail_for_external()` 后裁剪 `raw_content`
- 当前没有拆成 `extract_verification_code_for_external()` / `extract_verification_link_for_external()` 两个 service，controller 都走 `get_verification_result()`
- 当前 `fail()` 只返回字典，HTTP status 由 controller 决定

### 8.3 `list_messages_for_external()` 设计

输入：
- `email_addr`
- `folder`
- `skip`
- `top`

补充说明：

- `from_contains`
- `subject_contains`
- `since_minutes`

以上三个过滤条件不在该函数签名里处理，而是由外层配合 `filter_messages()` 处理。

真实返回：

```python
([message_summary, ...], "Graph API")
```

实现步骤：

1. `require_account(email_addr)` 做账号存在性和 email 合法性校验
2. 若 `account_type == "imap"`，直接走 `get_emails_imap_generic()`
3. 若是 Outlook 账号，按 `Graph API -> IMAP(New) -> IMAP(Old)` 回退
4. 使用 `_build_message_summary()` 统一映射字段
5. 返回 `(emails, method_label)`，过滤由 controller 再调用 `filter_messages()`

### 8.4 `get_latest_message_for_external()`

真实实现：

- 直接调用 `list_messages_for_external(email_addr=..., folder=..., skip=0, top=20)`
- 再调用 `filter_messages(...)`
- 若过滤后为空，抛 `MailNotFoundError`
- 为避免不同读取链路返回顺序不稳定，再按 `timestamp` 倒序排序后返回第一封

### 8.5 `get_message_detail_for_external()`

真实实现分两条链路：

1. `account_type == "imap"`：调用 `get_email_detail_imap_generic()`
2. Outlook 账号：`graph_service.get_email_detail_graph()` 优先，失败后回退 `imap_service.get_email_detail_imap_with_server(..., IMAP_SERVER_NEW/OLD)`

统一返回结构：

```python
{
    "id": ...,
    "email_address": email_addr,
    "from_address": ...,
    "to_address": ...,
    "subject": ...,
    "content": ...,
    "html_content": ...,
    "raw_content": ...,
    "timestamp": ...,
    "created_at": ...,
    "has_html": True/False,
    "method": "Graph API" | "IMAP (New)" | "IMAP (Old)",
}
```

补充约束：

- Graph 详情读取成功时，会额外调用 `graph_service.get_email_raw_graph()` 尝试补 `raw_content`
- IMAP 分支无独立 HTML body 时，`content` 与 `raw_content` 可能来自同一份正文
- message detail 与 raw 接口实际共享同一底层 service，避免双份读取逻辑

### 8.6 `wait_for_message()`

当前真实逻辑不是“只要有最新邮件就返回”，而是“只返回本次请求开始之后出现的匹配邮件”。核心伪码：

```python
baseline_timestamp = int(time.time())
start = time.time()
while time.time() - start < timeout_seconds:
    latest_message = get_latest_message_for_external(...)
    if int(latest_message.get("timestamp") or 0) >= baseline_timestamp:
        return latest_message
    time.sleep(poll_interval)
raise MailNotFoundError("等待超时，未检测到匹配邮件")
```

注意：
- 最大超时 120 秒
- 默认 30 秒
- 不使用异步任务，不引入 scheduler
- 若当前轮次没有匹配邮件，内部会捕获 `MailNotFoundError` 继续轮询
- 返回结果是消息摘要，不是完整 message detail

### 8.7 `wait-message` 解耦演进设计（v1.2 首版已落地）

当前同步轮询实现适用于：

- 本地化部署
- 低并发
- 单可信调用方

但不适用于：

- 单 worker 公网暴露
- 高并发等待场景
- 需要稳定 SLA 的对外 API

当前已落地的两层形态为：

```text
后台轮询器 / worker
  ↓ 周期性拉取最近邮件
  ↓ 归一化并写入缓存表 / 状态表

Web API
  ↓ 查询最近状态
  ↓ 只负责返回，不负责 sleep 轮询
```

建议增加的抽象：

| 抽象 | 作用 |
|---|---|
| `external_probe_cache` | 缓存按条件命中的探测状态与结果 |
| `poll_pending_probes()` | 后台轮询 pending 探测并写回状态 |
| `create_probe()` / `get_probe_status()` | 创建异步探测并查询最新状态 |

演进策略建议：

1. 当前已在公网模式支持禁用 `wait-message`
2. 当前已引入后台探测状态与 `/api/external/probe/{probe_id}`
3. 后续再决定是否完全收敛同步模式

---

## 9. 邮件读取与回退链路细节

### 9.1 当前真实列表读取链路

当前没有单独暴露 `_read_emails_with_fallback()` 私有函数，回退逻辑直接写在 `list_messages_for_external()` 中。

#### Outlook 账号链路

```python
graph_result = graph_service.get_emails_graph(
    account["client_id"],
    account["refresh_token"],
    folder,
    skip,
    top,
    proxy_url,
)
```

成功：
- 直接返回
- `method = "Graph API"`

失败：
- 收集 `graph_error`
- 若是 `ProxyError` / `ConnectionError`，直接中止并抛 `ProxyError`

#### Outlook IMAP 回退链路

按顺序：

1. `IMAP_SERVER_NEW = "outlook.live.com"`
2. `IMAP_SERVER_OLD = "outlook.office365.com"`

成功时返回 `method = "IMAP (New)"` 或 `"IMAP (Old)"`

#### 通用 IMAP 账号链路

若 `account_type == "imap"`，则不走 Graph / Outlook XOAUTH2 IMAP 回退，而是直接走：

```python
get_emails_imap_generic(
    email_addr=email_addr,
    imap_password=account.get("imap_password", "") or "",
    imap_host=account.get("imap_host", "") or "",
    imap_port=account.get("imap_port", 993) or 993,
    folder=folder,
    provider=account.get("provider", "_default") or "_default",
    skip=skip,
    top=top,
)
```

### 9.2 当前真实详情读取链路

实现顺序：

1. `account_type == "imap"` 时，走 `get_email_detail_imap_generic()`
2. Outlook 账号时，先 `graph_service.get_email_detail_graph()`
3. Graph 成功后额外尝试 `graph_service.get_email_raw_graph()`
4. Graph 失败后依次回退 `imap_service.get_email_detail_imap_with_server(..., IMAP_SERVER_NEW/OLD)`
5. 三条链路都失败则抛 `MailNotFoundError`

### 9.3 非 Outlook IMAP 兼容说明

当前开放 API 第一版目标是 Outlook 邮箱，但根项目已存在 `account_type == "imap"` 分支。

因此当前真实实现已经做到：
- 开放 API 仍允许对 `account_type == "imap"` 的账号调用
- 列表读取时复用 `get_emails_imap_generic()`
- 详情读取时复用 `get_email_detail_imap_generic()`
- system `account-status` 中 `preferred_method` 返回 `imap_generic`

这样与 PRD 的“本地化部署邮件读取服务”目标更一致，也不把开放接口写死为仅支持 Outlook OAuth

---

## 10. 验证码与验证链接提取细节

### 10.1 当前提取器能力

`outlook_web/services/verification_extractor.py` 已具备：

- `smart_extract_verification_code()`
- `fallback_extract_verification_code()`
- `extract_links()`
- `extract_email_text()`
- `extract_verification_info()`

### 10.2 新增参数化入口

新增：

```python
def extract_verification_info_with_options(
    email: Dict[str, Any],
    *,
    code_regex: str | None = None,
    code_length: str | None = None,
    code_source: str = "all",
    prefer_link_keywords: list[str] | None = None,
) -> Dict[str, Any]:
```

### 10.3 `code_source` 实现

| 值 | 取值内容 |
|---|---|
| `subject` | 仅主题 |
| `content` | 仅纯文本正文 |
| `html` | 仅 HTML 内容 |
| `all` | 主题 + 纯文本 + HTML |

真实实现要点：

```python
subject = email.get("subject", "")
content = _extract_content_text_without_subject(email)
html_content = email.get("body_html") or email.get("html_content") or ""

if code_source == "subject":
    source_text = subject
elif code_source == "content":
    source_text = content
elif code_source == "html":
    source_text = html_content
else:
    source_text = f"{subject} {content} {html_content}".strip()
```

### 10.4 `code_regex` 实现

若传入 `code_regex`：
- 先 `re.compile(code_regex)`
- 编译失败抛 `ValueError("code_regex 参数无效")`
- 编译成功后直接用于优先提取

### 10.5 `code_length` 实现

格式示例：`4-8`、`6-6`

实现：

```python
m = re.match(r"^(\d+)-(\d+)$", code_length)
if not m:
    raise ValueError("code_length 参数无效")
min_len = int(m.group(1))
max_len = int(m.group(2))
if min_len > max_len:
    raise ValueError("code_length 参数无效")
pattern = rf"\b\d{{{min_len},{max_len}}}\b"
```

### 10.6 验证链接优先级实现

默认优先关键词：

```python
DEFAULT_LINK_KEYWORDS = [
    "verify",
    "confirmation",
    "confirm",
    "activate",
    "validation",
]
```

算法：
1. 提取全部链接
2. 遍历关键词
3. 返回第一个包含关键词的链接
4. 若无命中，返回 `links[0]`
5. 若无链接，提取器返回 `verification_link=None`，由 controller 映射为 `VERIFICATION_LINK_NOT_FOUND`

---

## 11. 统一响应与错误码映射

### 11.1 成功响应

```python
{
    "success": True,
    "code": "OK",
    "message": "success",
    "data": {...},
}
```

### 11.2 失败响应

```python
{
    "success": False,
    "code": "ACCOUNT_NOT_FOUND",
    "message": "邮箱账号不存在",
    "data": None,
}
```

### 11.3 自定义异常真实定义

`services/external_api.py` 当前已定义以下轻量异常：

```python
class ExternalApiError(Exception):
    code = "INTERNAL_ERROR"
    status = 500

class InvalidParamError(ExternalApiError):
    code = "INVALID_PARAM"
    status = 400

class AccountNotFoundError(ExternalApiError):
    code = "ACCOUNT_NOT_FOUND"
    status = 404

class MailNotFoundError(ExternalApiError):
    code = "MAIL_NOT_FOUND"
    status = 404

class VerificationCodeNotFoundError(ExternalApiError):
    code = "VERIFICATION_CODE_NOT_FOUND"
    status = 404

class VerificationLinkNotFoundError(ExternalApiError):
    code = "VERIFICATION_LINK_NOT_FOUND"
    status = 404

class ProxyError(ExternalApiError):
    code = "PROXY_ERROR"
    status = 502

class UpstreamReadFailedError(ExternalApiError):
    code = "UPSTREAM_READ_FAILED"
    status = 502
```

### 11.4 错误码与 HTTP 映射

| code | HTTP | 触发条件 |
|---|---|---|
| `UNAUTHORIZED` | 401 | 缺少/错误 API Key |
| `API_KEY_NOT_CONFIGURED` | 403 | 系统未配置开放 API Key |
| `INVALID_PARAM` | 400 | 参数校验失败 |
| `ACCOUNT_NOT_FOUND` | 404 | 账号不存在 |
| `MAIL_NOT_FOUND` | 404 | 无匹配邮件 |
| `VERIFICATION_CODE_NOT_FOUND` | 404 | 邮件存在但无验证码 |
| `VERIFICATION_LINK_NOT_FOUND` | 404 | 邮件存在但无验证链接 |
| `PROXY_ERROR` | 502 | Graph 代理连接失败 |
| `UPSTREAM_READ_FAILED` | 502 | Graph/IMAP 全失败 |
| `INTERNAL_ERROR` | 500 | 未分类异常 |

---

## 12. 系统自检接口实现细节

### 12.1 `api_external_health()`

文件：`outlook_web/controllers/system.py`

当前真实实现：

```python
@api_key_required
def api_external_health():
    conn = create_sqlite_connection()
    try:
        db_ok = True
        try:
            conn.execute("SELECT 1").fetchone()
        except Exception:
            db_ok = False

        data = {
            "status": "ok",
            "service": "outlook-email-plus",
            "version": APP_VERSION,
            "server_time_utc": utcnow().isoformat() + "Z",
            "database": "ok" if db_ok else "error",
        }
        return jsonify(external_api_service.ok(data))
    finally:
        conn.close()
```

补充约束：

- 当前 `health` 的语义偏向“服务进程与数据库可用性检查”
- 当前不把它定义为“Graph / IMAP 上游真实可读性证明”
- 即使数据库探测结果是 `error`，只要接口本身未抛异常，当前实现仍返回 HTTP 200，差异体现在 `data.database`
- 若后续进入公网模式，应增加 `public_mode`、`restricted_features`、`upstream_probe_ok` 等字段

### 12.2 `api_external_capabilities()`

当前真实返回固定能力列表：

```python
FEATURES = [
    "message_list",
    "message_detail",
    "raw_content",
    "verification_code",
    "verification_link",
    "wait_message",
]
```

并附带：

- `service`
- `version`

后续增强建议：

```python
{
    "service": "outlook-email-plus",
    "version": APP_VERSION,
    "features": [...],
    "public_mode": False,
    "restricted_features": [],
}
```

约束：

- 私有模式下 `restricted_features` 可为空
- 公网模式下应显式列出：
  - `wait_message`
  - `raw_content`
  等被禁用或降级的能力

### 12.3 `api_external_account_status()`

当前真实实现：
1. 校验 `email`
2. 查询 `accounts_repo.get_account_by_email(email)`
3. 若不存在，返回 `ACCOUNT_NOT_FOUND`
4. 存在则返回：
   - `email`
   - `exists`
   - `account_type`
   - `provider`
   - `group_id`
   - `status`
   - `last_refresh_at`
   - `preferred_method`
   - `can_read`

注意：
- 不在该接口中真正拉信
- 该接口是自检接口，不是链路测试接口
- `can_read` 不是固定 `True`，而是按真实账号条件计算：
  - `status == disabled` 时为 `False`
  - `account_type == "imap"` 时要求 `imap_host` 和 `imap_password` 都存在
  - Outlook 账号要求 `client_id` 和 `refresh_token` 都存在
- 后续如需更强自检，应补充 `probe_method`、`probe_ok`、`probe_error`、`last_probe_at`

---

## 13. 前端设置页改造细节

### 13.1 `templates/index.html`

在系统设置页新增区块：

- `settingsExternalApiKey`：密码输入框
- `externalApiKeyHint`：显示“已配置/未配置”与脱敏值
- 保存按钮继续复用现有 `saveSettings()`

当前真实 UI 约束：

- 输入框使用密码类型，避免直接暴露屏幕内容
- 不单独新增新的保存接口，继续走 `/api/settings`
- 当前未单独增加“随机生成对外 API Key”按钮，因此 TDD 不把它列为必做项

### 13.2 `static/js/main.js`

#### 13.2.1 `loadSettings()`

在现有 settings 拉取成功后追加：

```javascript
const externalApiKeyInput = document.getElementById('settingsExternalApiKey');
const externalApiKeyHint = document.getElementById('externalApiKeyHint');
if (externalApiKeyInput) {
  const masked = data.settings.external_api_key_masked || '';
  externalApiKeyInput.value = masked;
  externalApiKeyInput.dataset.maskedValue = masked;
  externalApiKeyInput.dataset.isSet = data.settings.external_api_key_set ? 'true' : 'false';
}
if (externalApiKeyHint) {
  if (data.settings.external_api_key_set) {
    externalApiKeyHint.textContent = `已设置：${data.settings.external_api_key_masked || ''}（请求头：X-API-Key；清空后保存可禁用对外接口）`;
  } else {
    externalApiKeyHint.textContent = '未设置（设置后可通过 /api/external/* 对外开放接口读取邮件与验证码）';
  }
}
```

#### 13.2.2 `saveSettings()`

```javascript
const externalApiKeyEl = document.getElementById('settingsExternalApiKey');
const externalApiKey = externalApiKeyEl ? externalApiKeyEl.value.trim() : '';
const externalApiKeyMasked = externalApiKeyEl ? (externalApiKeyEl.dataset.maskedValue || '') : '';
const externalApiKeyIsSet = externalApiKeyEl ? externalApiKeyEl.dataset.isSet === 'true' : false;

if (!(externalApiKeyIsSet && externalApiKey && externalApiKey === externalApiKeyMasked)) {
  settings.external_api_key = externalApiKey;
}
```

实现注意点：

- 必须避免把脱敏占位符直接写回 DB
- 允许用户通过“清空 + 保存”禁用 external API
- 提示文案要明确告知 Header 使用方式：`X-API-Key`

---

## 14. 兼容性与回归保障

### 14.1 内部接口不变

以下接口不允许因开放接口改造而改变：

- `GET /api/emails/<email_addr>`
- `GET /api/email/<email_addr>/<message_id>`
- `GET /api/emails/<email_addr>/extract-verification`
- `GET /api/settings`
- `PUT /api/settings`

### 14.2 加密兼容策略

`external_api_key` 读取逻辑必须兼容：
- 新数据：加密值
- 历史手工写入：明文值
- 空值：未配置

### 14.3 回退链路不变

开放接口与内部接口应共享相同的 Graph/IMAP 回退策略，避免两套行为分叉。

---

## 15. 测试策略与测试用例

### 15.1 当前已存在的测试文件

| 文件 | 说明 |
|---|---|
| `tests/test_external_api.py` | 开放接口主测试 |
| `tests/test_settings_external_api_key.py` | 设置项与加密兼容测试 |
| `tests/test_verification_extractor_options.py` | 提取器参数化测试 |
| `tests/test_ui_settings_external_api_key.py` | 设置页 DOM / 前端脚本接线检查 |

### 15.2 鉴权测试

- 未传 `X-API-Key` → 401 `UNAUTHORIZED`
- 未配置 `external_api_key` → 403 `API_KEY_NOT_CONFIGURED`
- 错误 key → 401 `UNAUTHORIZED`
- 正确 key → 进入 controller
- `public_mode + IP 不在白名单` → 403 `IP_NOT_ALLOWED`
- `public_mode + wait-message 已禁用` → 403 `FEATURE_DISABLED`
- `public_mode + 请求频率超限` → 429 `RATE_LIMIT_EXCEEDED`

### 15.3 `messages` 接口测试

- 正常返回列表
- `folder=spam` → 400
- 账号不存在 → 404
- Graph 成功时 `method=Graph API`
- Graph 失败 IMAP 成功时返回成功
- Graph 代理错误时返回 502 `PROXY_ERROR`
- `account_type == "imap"` 时走 `get_emails_imap_generic()`

### 15.4 `messages/latest` 测试

- 命中最新邮件
- 无邮件 → 404 `MAIL_NOT_FOUND`

### 15.5 `messages/{id}` / `raw` 测试

- 正常获取详情
- 缺失 email → 400
- message_id 无效 → 404/502（视底层行为映射）
- `raw` 接口仅返回 `id/email_address/raw_content/method`

### 15.6 `verification-code` 测试

- 默认 4-8 位数字提取成功
- `code_length=6-6` 生效
- `code_regex` 生效
- `code_regex` 非法 → 400
- 邮件存在但无验证码 → 404 `VERIFICATION_CODE_NOT_FOUND`

### 15.7 `verification-link` 测试

- 命中 `verify` 关键词链接
- 无关键词时回退首个外链
- 无链接 → 404 `VERIFICATION_LINK_NOT_FOUND`

### 15.8 `wait-message` 测试

- 第一轮命中直接返回
- 多轮轮询后命中
- 超时返回 404 `MAIL_NOT_FOUND`
- `timeout_seconds > 120` → 400
- `poll_interval > timeout_seconds` → 400
- 仅当消息 `timestamp >= 请求开始时间` 时才返回，旧消息不会误命中

补充说明：

- 当前测试验证的是受控环境中的功能正确性
- 已覆盖公网模式下的接口分级、白名单与限流基础场景
- 尚未覆盖的重点转为：
  - 真实上游探测字段
  - 更高并发下的阻塞/异步混合场景
  - 多 API Key 与范围授权模型

### 15.9 系统接口测试

- `health` 返回 `status/service/database/server_time_utc`
- `capabilities` 返回固定 feature 列表
- `account-status` 存在/不存在路径正确

---

## 16. 实施顺序建议

### 16.1 当前已完成的落地顺序

1. `db.py` 初始化 `external_api_key`
2. `repositories/settings.py` 增加读取/脱敏函数
3. `security/auth.py` 增加 `api_key_required`
4. `controllers/settings.py` 与设置页对接

### 16.2 当前已完成的服务与路由落地

1. 新增 `services/external_api.py`
2. 在 `services/external_api.py` 内落地列表/详情/验证码/等待/审计编排
3. 增强 `verification_extractor.py`
4. 注册 `/api/external/messages*`
5. 注册 `/api/external/verification-*`
6. 注册 `/api/external/health|capabilities|account-status`

### 16.3 当前已完成的验证与文档落地

1. 补充 `tests/test_external_api.py`
2. 补充 `tests/test_settings_external_api_key.py`
3. 补充 `tests/test_verification_extractor_options.py`
4. 补充 `tests/test_ui_settings_external_api_key.py`
5. 同步 PRD / FD / API 文档 / TDD

### 16.4 后续实现重点

1. 对照 OpenAPI 持续校验返回结构与错误码
2. 视上线模式决定是否补充 README 的对外接入示例
3. 评估是否需要把 `wait-message` 从同步轮询迁移为后台探测模型

### 16.5 后续阶段建议（非首版）

1. 增加 `public_mode` 配置项与 `restricted_features`
2. 增加来源 IP 白名单
3. 增加外部接口限流
4. 将 `wait-message` 逐步迁移为后台轮询或异步等待模型
5. 如需多外部调用方，再设计多 API Key 与邮箱范围授权

### 16.6 参考项目经验沉淀

本项目后续演进建议参考但不照搬以下方向：

| 参考项目 | 可借鉴点 | 不照搬点 |
|---|---|---|
| `exeample/outlookEmail` | 最小外部 API 闭环、独立 API 文档 | query 参数传 key、明文 API Key |
| `exeample/BillionMail` | 白名单 / Fail2Ban 这类外围防护思路 | 其整体邮件系统架构与本项目不同 |
| `exeample/MailAggregator_Pro` | 后台轮询与 Web 接口解耦、健康状态动态暴露 | 其异步框架与当前 Flask 架构不同 |

---

## 结论

本 TDD 的核心落点是：

- **把开放 API 做成一层薄而稳定的外壳**，底层仍复用现有内部邮件读取能力；
- **把验证码/链接提取从“页面能力”升级为“服务能力”**，并支持参数化配置；
- **把设置、鉴权、审计、健康检查一起落地**，让开放接口从第一版就具备可配置、可接入、可排查的完整闭环；
- **通过新增 `services/external_api.py` 控制复杂度**，避免继续把复杂逻辑塞回 controller；
- **后续继续沿“鉴权收敛 + 轮询解耦”两条主线演进**，而不是直接把当前实现定义为公网开放平台。
