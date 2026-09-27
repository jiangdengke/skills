# Diagnostics and Observability

## Diagnostic Principle

先找“最后一个有明确证据的成功状态”，再检查从该状态到下一状态的输入、动态值、会话、策略和副作用。不要每次失败都从入口重新猜。

示例：

```text
SESSION_READY         OK
REGISTRATION_STARTED  OK
VERIFICATION_SENT     OK
VERIFIED              OK
COMPLETING            FAIL 409
```

问题范围在 `VERIFIED -> COMPLETING`，不是验证码阶段。

## Error Taxonomy

至少使用稳定类别：

1. `NETWORK_ERROR`
2. `TLS_ERROR`
3. `PROTOCOL_ERROR`
4. `STATE_ERROR`
5. `VALIDATION_ERROR`
6. `AUTHENTICATION_ERROR`
7. `AUTHORIZATION_ERROR`
8. `RATE_LIMIT_ERROR`
9. `RISK_OR_POLICY_ERROR`
10. `VERIFICATION_ERROR`
11. `CONCURRENCY_ERROR`
12. `EXTERNAL_EVENT_TIMEOUT`
13. `SERVER_ERROR`

不要把所有失败输出为“注册失败”。

## Five-Layer Root Cause Framework

错误类别描述表象；五层框架定位根因：

### 1. Request Context

- Cookie/Authorization 缺失或属于其他 Context。
- Origin/Referer/CSRF 与当前会话不匹配。
- Content-Type、Redirect 或 Host 上下文错误。
- Challenge 结果不属于当前事务。

### 2. Flow State

- 缺少初始化、验证、资料或同意步骤。
- 事务已经推进/完成，客户端仍停留旧状态。
- 页面步骤与服务端状态不一致。

### 3. Dynamic Dependency

- token/nonce/state/verifier/transaction 已过期。
- 一次性值已被消费。
- 字段来源错误或拿到旧邮件。
- 复用了另一事务的值。

### 4. Security Control

- Rate Limit。
- Challenge/MFA。
- Policy deny/manual review。
- WAF/Bot/Risk 决策。

尊重控制信号，不规避。

### 5. Local State Management

- 全局 Cookie Jar。
- Refresh Token 轮换后仍使用旧值。
- 重复提交。
- 并发 Context 串线。
- 日志关联丢失。

## Diagnostic Sequence

1. 记录最后成功状态和失败转换。
2. 检查 HTTP Status、业务错误码、Body、Response Header、Set-Cookie。
3. 检查 Request ID/Trace ID。
4. 对比成功基线同一步骤。
5. 检查 Context、Cookie、token、transaction 和生命周期。
6. 检查是否为策略/限流/Challenge。
7. 检查本地并发、旧状态和重复提交。
8. 只在单变量条件下验证假设。

## Status Code Playbooks

### 401 Unauthorized

检查：

- Access Token/Cookie 是否实际发送。
- token 是否过期、nbf 未到、aud/iss 不匹配。
- 登录会话是否真的建立。
- 是否误用 ID Token。
- Refresh Token 轮换后是否仍用旧 Access Token。
- Cookie Domain/Path/Secure/SameSite 是否阻止发送。

恢复：合法刷新、重新认证或重新初始化；不伪造 token。

### 403 Forbidden

检查：

- Authentication 成功但 Authorization 不足。
- CSRF、Origin、Referer 或账号状态。
- Edge/WAF 响应特征与 Request ID。
- 是否需要 Challenge、MFA 或额外资料。
- 地区/年龄/合规或风险策略。

不要直接归因于“指纹”，不要提供规避策略。

### 409 Conflict

检查：

- 账号/资源是否已存在。
- 前一步 timeout 但服务端已成功。
- create 操作是否重复。
- transaction 是否已推进或被消费。
- Idempotency Key 是否正确。

恢复：查询当前状态，复用已有结果或结束；不要直接重建。

### 422 Unprocessable Entity

检查：

- 必填字段。
- 数据格式和 Content-Type。
- 密码、日期、地区、年龄或资料政策。
- 前端是否在提交前做了转换。
- 字段是否属于正确流程版本。

### 429 Too Many Requests

检查：

- Retry-After。
- endpoint/账号/email/session/IP/API key 等维度。
- 客户端是否自动循环。
- 并发和 burst。
- 前序失败是否已产生副作用。

恢复：等待或停止；不得轮换身份、设备、网络来规避。

### 5xx

检查：

- 是否稳定复现。
- Request ID。
- 哪个状态转换失败。
- 响应是否来自网关还是应用。
- 请求是否幂等，结果是否已知。

只在安全条件下有限重试。

### HTTP 200 but Business Failure

检查：

- `ok`、`status`、`error`、`next_step`。
- 是否设置/清除 Cookie。
- 状态是否实际推进。
- 是否返回 `verification_required`、`challenge_required` 或 `profile_required`。

## Protocol-Specific Diagnostics

### Cookie Missing

- 是否收到 Set-Cookie。
- Domain/Path 是否匹配。
- Secure 与 HTTPS。
- SameSite 与跨站导航。
- Expiry。
- Redirect 中是否被覆盖。
- 客户端是否使用真实 Cookie Jar。

### CSRF Failure

- Session Cookie 与 CSRF 是否同一会话生成。
- token 是否来自当前页面/事务。
- Header/表单位置是否正确。
- Origin/Referer 是否符合预期。
- token 是否已轮换或过期。

### OAuth/OIDC Failure

- redirect_uri 精确匹配。
- state 与原请求匹配。
- PKCE verifier/challenge 匹配。
- code 未过期且未消费。
- nonce 与 ID Token 匹配。
- token endpoint/client 类型正确。
- iss/aud/签名/时间验证正确。

### OTP/Magic Link Failure

- 是否属于当前 transaction。
- 是否拿到旧邮件。
- 是否重发后旧 code 失效。
- 是否过期/达到次数上限。
- 浏览器 Session 是否变化。
- Magic Link 是否要求原浏览器上下文。

### WebAuthn Failure

- RP ID/origin。
- challenge 是否当前请求生成。
- credential 是否允许。
- 用户验证要求。
- authenticator/浏览器支持。
- assertion/attestation 解析和签名验证。

不能用 HTTP 重放替代认证器私钥操作。

## Retry Decision Matrix

| Condition | Default action |
|---|---|
| Request never left client and no side effect | limited retry |
| Read timeout on state-changing request | mark result unknown, inspect state |
| 502/503/504 on idempotent action | bounded backoff |
| 429 with Retry-After | wait within authorization or stop |
| 400/422 | fix input, no retry |
| 401 due expiry | refresh/re-auth through valid flow |
| 403 policy/challenge | stop or legitimate human/official path |
| 409 | inspect existing state |
| OTP error | new authorized input/flow; no guessing loop |
| Token/code consumed | restart appropriate flow |
| Deadline exceeded | cancel and clean up |

## Structured Logging

### Required Fields

```text
timestamp
run_id
context_id
state
last_successful_state
step
endpoint_role
method
host_role
path_role
http_status
latency_ms
request_id
trace_id
correlation_id
business_error_code
error_category
root_cause_layer
retry_count
retryable
result_known
recommended_action
```

### Never Log by Default

- 完整请求/响应 Body。
- 完整 Header。
- 完整 URL 查询串。
- Password、OTP、Magic Link。
- Cookie、Access/Refresh/ID Token。
- OAuth code、PKCE verifier、Client Secret。
- 完整邮箱、手机号、个人资料。

### Redaction

- 白名单日志字段。
- 类型占位符。
- 短期不可逆关联 ID。
- 错误对象和异常消息也要经过脱敏。
- 日志测试必须扫描 token/JWT/cookie/password 模式。

## Request and Trace IDs

保存服务端返回的 Request ID、Trace ID、Correlation ID。它们可以：

- 将客户端失败与服务端日志关联。
- 区分网关、WAF 和应用响应。
- 识别一次 timeout 是否在服务端成功。
- 支持官方支持或内部团队排查。

不要用随机生成的客户端 ID 冒充服务端 Request ID；分别记录。

## Metrics

建议：

- 各状态进入次数、成功率和停留时间。
- 各 Endpoint Role 的错误类别。
- 最后成功状态分布。
- Retry 次数和退避时间。
- 429 恢复时间。
- OTP 到达/解析/过期时间。
- Challenge/MFA/人工等待时间。
- 未知结果和状态确认次数。
- Context 串线检测计数。

控制高基数：不要把邮箱、token、完整 URL 或 transaction ID 直接作为指标标签。

## Incident Handling for Exposed Secrets

如果分析材料中发现真实秘密：

1. 停止复制和传播。
2. 标记暴露位置和时间，不记录秘密内容。
3. 撤销/轮换凭证。
4. 终止相关 Session。
5. 清理 HAR、日志、截图和消息副本。
6. 检查访问记录和潜在使用。
7. 通知系统所有者/安全团队。

## Diagnostic Report Format

```markdown
## Failure Summary
- Last successful state:
- Failed transition:
- HTTP/business error:
- Error category:
- Root-cause layer:
- Result known?:
- Request/Trace ID:

## Evidence
- Baseline comparison:
- Cookie/token/transaction lifecycle:
- Response/Set-Cookie/state delta:

## Conclusion
- Observed facts:
- Verified findings:
- Hypotheses:

## Recovery
- Immediate action:
- Retry allowed?:
- Required reset/re-auth/user action:
- Safety stop condition:
```

## Diagnostic Definition of Done

- [ ] 已找到最后成功状态。
- [ ] 已识别失败转换。
- [ ] HTTP 与业务错误均已检查。
- [ ] 已记录 Request/Trace ID。
- [ ] 已分配错误类别和根因层。
- [ ] 已判断结果是否已知。
- [ ] 已判断副作用和重试安全性。
- [ ] 恢复路径明确。
- [ ] 假设与事实分开。
- [ ] 日志和报告无秘密。
