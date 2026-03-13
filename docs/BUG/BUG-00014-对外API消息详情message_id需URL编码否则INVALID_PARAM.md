# BUG-00014 对外 API 消息详情 `message_id` 需 URL 编码，否则返回 `INVALID_PARAM`

> 关联需求: `PRD-00008` / `FD-00008` / `TDD-00008`
> 记录时间: 2026-03-12
> 影响范围: `/api/external/messages/{message_id}`、`/api/external/messages/{message_id}/raw`
> 当前状态: 已记录，后续评估是否需要在服务端兼容更多未编码输入

---

## 一、问题概述

| 编号 | 标题 | 严重程度 | 类型 |
|------|------|---------|------|
| **D-1** | Outlook/Graph 风格 `message_id` 含特殊字符时，若调用方不做 URL 编码，详情/RAW 接口会返回 `INVALID_PARAM` | 🟡 中 | 接口可用性 / 接入兼容性 |

---

## 二、现象描述

对外接口 `/api/external/messages/{message_id}` 与 `/api/external/messages/{message_id}/raw` 的 `message_id` 取自上游邮件系统。

实际 Outlook/Graph 返回的 `message_id` 往往包含：

- `=`
- `/`
- `+`
- 其他 URI 保留字符

当调用方把该值直接拼进 URL 路径，而没有先做 URL 编码时，服务端会把路径错误切分，最终返回：

```json
{
  "success": false,
  "code": "INVALID_PARAM",
  "message": "email 参数无效",
  "data": null
}
```

---

## 三、复现方式

### 3.1 先获取一条消息

```bash
curl -H "X-API-Key: your-api-key" \
  "http://localhost:5000/api/external/messages?email=user@outlook.com&top=1"
```

返回示例中的 `id` 类似：

```text
AQMkADAwATM0MDAAMi05...AAAHv5VJgAAAA=
```

### 3.2 直接把原值拼到路径中

```bash
curl -H "X-API-Key: your-api-key" \
  "http://localhost:5000/api/external/messages/AQMkADAwATM0MDAAMi05...AAAHv5VJgAAAA=?email=user@outlook.com"
```

### 3.3 实际结果

- 请求可能落到错误路径
- 或被 Flask / 路由层按未编码保留字符解析
- 最终返回 `400 INVALID_PARAM`

### 3.4 正确调用方式

调用方应先对 `message_id` 做 URL 编码，再拼入路径：

```bash
curl -H "X-API-Key: your-api-key" \
  "http://localhost:5000/api/external/messages/AQMkADAwATM0MDAAMi05...%3D?email=user@outlook.com"
```

---

## 四、影响分析

### 4.1 对接入方的影响

- 如果调用方直接拿 `messages` 接口返回的 `id` 拼路径，请求会偶发失败
- 问题更容易出现在 Outlook/Graph 邮件账号
- 接入方会误判为“服务不稳定”或“详情接口偶发失效”

### 4.2 对系统本身的影响

- 当前不是服务异常，也不是邮件读取链路异常
- 本质是路径参数含保留字符时，调用方未按 URL 规范编码
- 但从产品体验看，当前错误提示不够直观，容易误导排查方向

---

## 五、当前结论

当前阶段先按“接入规范要求”处理：

1. `message_id` 作为路径参数时，调用方必须先做 URL 编码
2. 文档与联调说明中应明确这一点
3. 后续再评估是否需要在服务端增加更强兼容性或更明确的错误提示

---

## 六、后续可选改进

可选方向：

1. 在接口文档中明确标注：`message_id` 作为 path param 时必须 URL encode
2. 在服务端捕获这类常见未编码场景，返回更明确的错误提示，而不是泛化为 `INVALID_PARAM`
3. 如果后续要进一步降低接入门槛，可评估改造为：
   - 继续保留当前 path 风格
   - 同时支持 query 形式的 `message_id`
   - 或引入更稳定的内部消息定位方案

当前不建议为此立即改路径协议，因为这会影响现有对外接口稳定性。
