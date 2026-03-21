# Registration Worker Integration and External Mail Pool API

[中文文档](./注册与邮箱池接口文档.md) | [English Version](./registration-mail-pool-api.en.md)

## Overview

This document describes the externally exposed mail-pool APIs for registration workers and script-based integrations.

**Service purpose**: provide controlled mailbox claiming, result callbacks, and pool status visibility for registration workflows.

**Data format**: all requests and responses use JSON.

**CORS**: cross-origin requests are supported.

**Current contract**: only `/api/external/pool/*` is available in the current version. The old anonymous `/api/pool/*` endpoints have been removed.

---

## Authentication and General Rules

### Getting an API Key

Contact the system administrator to obtain an API key and send it in the request header:

```text
X-API-Key: YOUR_API_KEY
```

**Test environment**: contact the administrator for a test key. Rate limit: 100 requests per minute.

### Preconditions

Before calling the APIs, confirm all of the following:

1. `pool_external_enabled=true` is enabled on the server
2. Your API key is enabled and has `pool_access=true`
3. If public mode is enabled, your caller must also satisfy the IP whitelist, feature switches, and rate limits

### Calling Model

- These are service-to-service APIs
- No browser login session is required
- No cookies are required
- No CSRF token is required
- All requests use `X-API-Key`

### Standard Response Format

All endpoints follow the same response structure:

```json
{
  "success": true,
  "data": { "...": "..." },
  "message": "Operation completed successfully"
}
```

Failure response:

```json
{
  "success": false,
  "code": "ERROR_CODE",
  "message": "Error description"
}
```

### Time Field Format

All time fields use ISO 8601 format: `YYYY-MM-DDTHH:MM:SSZ`

---

## API Endpoint List

### Quick Start

Registration worker integrations usually only need these four endpoints:

| Endpoint | Purpose | Required |
| --- | --- | --- |
| `POST /api/external/pool/claim-random` | Claim a mailbox | Yes |
| `POST /api/external/pool/claim-complete` | Submit task completion | Yes |
| `POST /api/external/pool/claim-release` | Release a mailbox | Yes |
| `GET /api/external/pool/stats` | View pool status | Optional |

---

## 1. Claim a Mailbox

### Basics

```text
POST /api/external/pool/claim-random
Authentication required: Yes
```

### Request Example

```bash
curl -X POST https://api.example.com/api/external/pool/claim-random \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "caller_id": "register-worker-1",
    "task_id": "job-20260317-0001",
    "provider": "outlook"
  }'
```

### Parameters

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| `caller_id` | string | Yes | Caller identifier. Use a stable business or machine identifier such as `register-worker-1` or `register-cluster-a`. |
| `task_id` | string | Yes | Unique ID of the current registration task, such as `job-20260317-0001` or `order-928371`. |
| `provider` | string | No | Mail provider filter. Set `outlook` to claim Outlook mailboxes only. |

### Usage Recommendations

- Use a stable `caller_id` to identify a worker instance, host, or node
- Make `task_id` unique for every single job
- If you run different task types, prefer explicit provider filtering to reduce accidental claims

### Success Response Example

```json
{
  "success": true,
  "data": {
    "account_id": 12,
    "email": "demo@outlook.com",
    "claim_token": "clm_xxxxx",
    "lease_expires_at": "2026-03-17T10:00:00Z"
  },
  "message": "Mailbox claimed successfully"
}
```

### Response Fields

| Field | Type | Description |
| --- | --- | --- |
| `account_id` | integer | Account ID. Required when reporting completion or release. |
| `email` | string | Mailbox address. |
| `claim_token` | string | Claim token. Required in follow-up callbacks. |
| `lease_expires_at` | string | Lease expiration time. Submit completion or release before it expires. |

### Failure Response Example

```json
{
  "success": false,
  "code": "NO_AVAILABLE_ACCOUNT",
  "message": "No eligible mailbox is currently available in the pool"
}
```

---

## 2. Complete a Task and Submit the Result

### Basics

```text
POST /api/external/pool/claim-complete
Authentication required: Yes
```

### Request Example

```bash
curl -X POST https://api.example.com/api/external/pool/claim-complete \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "account_id": 12,
    "claim_token": "clm_xxxxx",
    "caller_id": "register-worker-1",
    "task_id": "job-20260317-0001",
    "result": "success",
    "detail": "Registration completed successfully"
  }'
```

### Parameters

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| `account_id` | integer | Yes | Account ID returned by the claim operation. |
| `claim_token` | string | Yes | Claim token returned by the claim operation. |
| `caller_id` | string | Yes | Must match the `caller_id` used when claiming. |
| `task_id` | string | Yes | Must match the `task_id` used when claiming. |
| `result` | string | Yes | Task result. See the available values below. |
| `detail` | string | No | Additional details such as failure reason, target site, or risk-control behavior. |

### Allowed `result` Values and Final Status Mapping

| `result` value | Meaning | Final mailbox status | Typical use case |
| --- | --- | --- | --- |
| `success` | Registration succeeded and the account was consumed | `used` | Task completed successfully |
| `verification_timeout` | Verification code was never received | `cooldown` | No code arrived for a long time; retry later |
| `provider_blocked` | The provider blocked or restricted the account | `frozen` | Provider risk control, suspension, or limitation |
| `credential_invalid` | Credentials are no longer valid | `retired` | Mailbox password or credentials are invalid |
| `network_error` | Temporary network or infrastructure problem | `available` | Safe to return to the pool and retry quickly |

### Callback Rules

- Report `success` only after the task is truly finished
- If the task is cancelled or never really starts, use `claim-release` instead of `claim-complete`
- Do not map every failure to `network_error`, or bad mailboxes will keep going back to the pool

### Success Response Example

```json
{
  "success": true,
  "data": {
    "account_id": 12,
    "pool_status": "used"
  },
  "message": "Task result submitted successfully"
}
```

### Failure Response Example

```json
{
  "success": false,
  "code": "TOKEN_MISMATCH",
  "message": "The claim_token does not match"
}
```

---

## 3. Release a Mailbox

### Basics

```text
POST /api/external/pool/claim-release
Authentication required: Yes
```

### Request Example

```bash
curl -X POST https://api.example.com/api/external/pool/claim-release \
  -H "X-API-Key: YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "account_id": 12,
    "claim_token": "clm_xxxxx",
    "caller_id": "register-worker-1",
    "task_id": "job-20260317-0001",
    "reason": "Task cancelled"
  }'
```

### Parameters

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| `account_id` | integer | Yes | Account ID returned by the claim operation. |
| `claim_token` | string | Yes | Claim token returned by the claim operation. |
| `caller_id` | string | Yes | Must match the original claim request. |
| `task_id` | string | Yes | Must match the original claim request. |
| `reason` | string | No | Reason for releasing the mailbox. |

### Typical Release Scenarios

- The job was cancelled
- The job never actually started
- An upstream dependency is missing and the mailbox should be returned without being counted as a failure

### Success Response Example

```json
{
  "success": true,
  "data": {
    "account_id": 12,
    "pool_status": "available"
  },
  "message": "Mailbox released back to the pool"
}
```

Notes:

- This endpoint is for pool observation only and does not change state
- Use it for monitoring, capacity checks, and validation, not for high-frequency polling

---

## 4. View Pool Status

### Basics

```text
GET /api/external/pool/stats
Authentication required: Yes
```

### Request Example

```bash
curl -X GET https://api.example.com/api/external/pool/stats \
  -H "X-API-Key: YOUR_API_KEY"
```

### Success Response Example

```json
{
  "success": true,
  "data": {
    "pool_counts": {
      "available": 850,
      "claimed": 120,
      "used": 20,
      "cooldown": 5,
      "frozen": 3,
      "retired": 2
    }
  },
  "message": "Query completed successfully"
}
```

---

## Error Handling

### HTTP Status Codes

| Status Code | Description |
| --- | --- |
| 200 | Request succeeded |
| 400 | Invalid request parameters |
| 401 | Unauthorized, missing or invalid API key |
| 403 | Forbidden |
| 404 | Resource not found |
| 429 | Rate limit exceeded |
| 500 | Internal server error |

### Common Error Responses

#### Missing API Key

```json
{
  "success": false,
  "code": "UNAUTHORIZED",
  "message": "Send X-API-Key: YOUR_API_KEY in the request header"
}
```

#### Rate Limit Exceeded

```json
{
  "success": false,
  "code": "RATE_LIMIT_EXCEEDED",
  "message": "Rate limit exceeded. Please retry later"
}
```

#### Feature Disabled

```json
{
  "success": false,
  "code": "FEATURE_DISABLED",
  "message": "Feature external_pool is currently disabled"
}
```

#### API Key Has No Pool Permission

```json
{
  "success": false,
  "code": "FORBIDDEN",
  "message": "The current API key is not allowed to access the external pool"
}
```

#### IP Not Allowed in Public Mode

```json
{
  "success": false,
  "code": "IP_NOT_ALLOWED",
  "message": "The current IP is not in the allowlist"
}
```

#### No Mailbox Available

```json
{
  "success": false,
  "code": "NO_AVAILABLE_ACCOUNT",
  "message": "No eligible mailbox is currently available in the pool"
}
```

#### `claim_token` Does Not Match

```json
{
  "success": false,
  "code": "TOKEN_MISMATCH",
  "message": "The claim_token does not match"
}
```

---

## Usage Limits

### Rate Limits

- Only enforced when public mode is enabled
- Applied as a unified per-IP minute bucket across claim / release / complete / stats
- Default limit is `60` requests per minute, and the effective value comes from `external_api_rate_limit_per_minute`

### Lease Timeout

After claiming a mailbox, you must submit a completion or release request before `lease_expires_at`. Expired claims are automatically returned to the pool.

### Recommended Retry Strategy

- When receiving `429`, wait 1 second before retrying
- When receiving `NO_AVAILABLE_ACCOUNT`, wait 5 to 10 seconds before retrying
- Exponential backoff is recommended

---

## Business Flow

```text
Import mailbox (add_to_pool=true)
         ↓
Mailbox enters the pool (status=available)
         ↓
Registration worker calls claim-random
         ↓
Registration worker performs the task
         ↓
    ┌────┴────┐
    ↓         ↓
Success/Fail  Abort midway
    ↓         ↓
claim-complete  claim-release
    ↓         ↓
Status updated   Mailbox returns to pool
```

---

## FAQ

### Q1: The account was imported, but the registration worker cannot claim it

**Possible reasons**:

1. `add_to_pool=true` was not set during import.
2. The account status is not `active` or it is not in `available`.

**Solution**: check the import parameters and make sure the account was added to the pool correctly.

Also check:

3. Whether `pool_external_enabled` is enabled
4. Whether the current API key has `pool_access=true`
5. Whether the current IP is included in the allowlist when public mode is enabled

### Q2: Why do I get a parameter mismatch error during callback

**Reason**: `account_id`, `claim_token`, `caller_id`, and `task_id` must exactly match the values returned by the claim operation.

**Solution**: the registration worker must store the original claim response and send it back without modification.

### Q3: What happens if I forget to submit a callback after claiming

**Impact**: the mailbox remains in `claimed` status and cannot be allocated to other tasks until the lease expires or it is released.

**Recommendation**:

- Call `claim-complete` for both successful and failed tasks.
- Call `claim-release` when abandoning the task midway.

### Q4: What happens if all failures are reported as `network_error`

**Impact**: invalid mailboxes may keep returning to the pool and get assigned repeatedly, wasting resources.

**Recommendation**: choose the correct `result` value based on the real failure reason.

### Q5: Why can’t the old `/api/pool/*` endpoints be used anymore?

**Reason**: the old anonymous endpoints were removed. The current version only exposes controlled external APIs for pool operations.

**Migration**:

- `/api/pool/claim-random` → `/api/external/pool/claim-random`
- `/api/pool/claim-complete` → `/api/external/pool/claim-complete`
- `/api/pool/claim-release` → `/api/external/pool/claim-release`
- `/api/pool/stats` → `/api/external/pool/stats`

Also migrate the request header to `X-API-Key`.

---

## Contact

- Feedback: [Submit an Issue]

---

## Changelog

### v1.0.0 (2026-03-17)

- Initial release
- Support for mailbox claim, completion callback, release, and status query

---

## Migration Notes

Anonymous `/api/pool/*` endpoints have been removed. Use the controlled external endpoints instead:

- `/api/external/pool/claim-random`
- `/api/external/pool/claim-complete`
- `/api/external/pool/claim-release`
- `/api/external/pool/stats`

If you still have old worker scripts, complete the following migration:

1. Move the path to `/api/external/pool/*`
2. Replace the auth header with `X-API-Key`
3. Add handling for `403` and `429`
4. Verify `pool_external_enabled` and `pool_access` before rollout
