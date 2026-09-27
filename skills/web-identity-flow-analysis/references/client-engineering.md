# Protocol Client Engineering

本文件用于在自有、测试或明确授权环境中，把已理解的身份流程实现为可测试、可恢复、可观测的客户端。先完成依赖图、字段字典和状态机，再写客户端。

## Architecture

```text
RegistrationOrchestrator
        |
RegistrationStateMachine
        |
IdentityServiceClient
        |
HTTPTransport
        |
TLS / Network

Side components:
CookieJar
TokenStore
VerificationAdapter
RetryPolicy
ErrorClassifier
IdempotencyStore
Clock
Logger / Metrics / Tracer
```

## Layer Responsibilities

### HTTPTransport

负责：

- URL、Method、Header、Body 编码。
- TLS 和证书配置。
- 连接池。
- Redirect。
- Cookie Jar。
- connect/read timeout。
- 低层请求/响应度量。

不负责：

- “发送验证码”“完成注册”等业务动作。
- 状态机转换。
- 业务重试决策。
- 直接读取用户验证码或处理 OAuth 流程。

抽象接口：

```python
class Transport:
    def request(self, method, url, *, headers=None, body=None, timeout=None):
        ...
```

### IdentityServiceClient

以语义动作封装协议端点：

```python
class IdentityClient:
    def initialize_session(self, context): ...
    def start_registration(self, context, identity_input): ...
    def send_verification(self, context): ...
    def confirm_verification(self, context, verification): ...
    def complete_registration(self, context, profile): ...
    def inspect_session(self, context): ...
```

职责：

- 将 Context 状态映射为请求。
- 解析响应 DTO。
- 提取 Cookie、transaction、业务错误和 Request ID。
- 不自行编排整个流程。

### State Machine

职责：

- 验证动作前置条件。
- 应用成功/失败转换。
- 保存最后成功状态。
- 管理一次性值的生成、消费和失效。
- 决定等待、回退、冷却、重新初始化或终止。

不要只存 `failed=True`；至少保存：

```text
error_category
error_code
http_status
request_id
retryable
recommended_action
result_known
```

### Orchestrator

负责：

- 按状态机编排动作。
- 等待 OTP、Magic Link、OAuth 回调、用户同意或认证器。
- 选择浏览器、官方 SDK 或协议执行路径。
- 管理总体 deadline、取消、清理和最终验证。

Orchestrator 不应直接拼 HTTP Header，也不应跳过状态机修改 Context。

### ErrorClassifier

将响应、异常和当前状态转换为稳定错误类型：

```python
class ErrorClassifier:
    def classify(self, event, context):
        if event.http_status == 429:
            return RateLimitError(...)
        if event.http_status in (502, 503, 504):
            return TransientServerError(...)
        if event.http_status == 401:
            return AuthenticationError(...)
        ...
```

字符串消息只作为补充证据，不作为唯一分类规则。

## RegistrationContext

每个并发事务必须独占 Context：

```text
context_id
run_id
current_state
last_successful_state
created_at
overall_deadline

cookie_jar
transaction_id
csrf_token
oauth_state
oauth_nonce
pkce_verifier
token_store
verification_state

retry_counters
idempotency_keys
request_ids
last_error
redaction_map
```

### Isolation Rules

- 不共享全局 HTTP Session/Cookie Jar。
- 不共享 OAuth state/nonce/verifier。
- 不共享 OTP、Magic Link 或 authorization code。
- 不共享可变 TokenStore。
- 日志必须包含 context_id，不能靠线程名猜事务。
- 共享邮箱必须有 Message-ID/时间/事务关联和消费锁。
- 连接池可共享，但会话状态不能共享。

## State Machine

推荐主状态：

```text
INIT
SESSION_READY
REGISTRATION_STARTED
VERIFICATION_REQUIRED
VERIFICATION_SENT
WAITING_EXTERNAL_EVENT
VERIFIED
PROFILE_REQUIRED
COMPLETING
AUTHENTICATED
DONE
COOLDOWN
CHALLENGE_REQUIRED
FAILED_RETRYABLE
FAILED_FINAL
```

### Transition Example

```text
Current: VERIFICATION_SENT
Event: verification result submitted

correct + transaction/session match
  -> VERIFIED
  -> consume OTP

expired
  -> VERIFICATION_REQUIRED or FAILED_FINAL
  -> invalidate old code

too many attempts
  -> COOLDOWN or FAILED_FINAL

network timeout
  -> remain VERIFICATION_SENT with result_known=false
  -> inspect state before retry
```

### State Invariants

- `AUTHENTICATED` 必须有有效 session/token 证据。
- `DONE` 必须包含受保护资源或等价最终检查。
- `VERIFIED` 不等于账号已创建。
- `SESSION_READY` 不等于已登录。
- 一次性值被消费后必须从 Context 移除或标记失效。
- `FAILED_RETRYABLE` 必须带有最大尝试和 deadline。

## Idempotency

### Questions Before Retry

1. 请求有副作用吗？
2. 客户端是否知道服务端最终结果？
3. 服务器是否公开支持 Idempotency Key？
4. Key 的作用域和保留时间是什么？
5. 当前凭证、transaction、challenge 是否仍有效？
6. 重复执行会发送邮件、创建对象或消费一次性值吗？

### Safe Patterns

- 无副作用读取：证据支持时有限重试。
- 创建动作：使用服务端支持的 Idempotency Key，或先查询现有状态。
- 发送验证码：不要自动重发；需要状态、冷却和用户意图。
- 消费 OTP/code：结果未知时先检查事务状态。
- 409 duplicate：查询现有对象/事务，不自动重新创建。

不要自行添加未经服务端支持的 `Idempotency-Key` 并假设有效。

## Retry Policy

重试决策是函数：

```text
(error category, current state, side effect, result known,
 credential lifetime, server hint, attempt count, deadline)
```

### Typical Matrix

| Error | Default | Conditions |
|---|---|---|
| DNS temporary | limited retry | no side effect yet |
| Connect timeout | limited retry | request likely not sent; verify transport semantics |
| Read timeout | inspect first | server may have completed side effect |
| 502/503/504 | conditional retry | idempotent/result known/server hint |
| 429 | wait/stop | honor Retry-After and authorized quota |
| 400/422 | no retry | fix input/protocol |
| 401 expired token | re-auth/refresh | only through valid flow |
| 403 policy deny | stop | record evidence; do not evade |
| 409 conflict | inspect state | previous action may have succeeded |
| OTP error | user/new flow | never loop guesses |
| Challenge required | wait/official path | no bypass |

### Backoff

Use bounded exponential backoff with jitter only when retry is allowed:

```text
delay = min(max_delay, base * 2^attempt) + jitter
```

Server `Retry-After` and operation deadline override generic backoff.

## Timeouts and Deadlines

至少区分：

- Connect Timeout。
- Read Timeout。
- Overall Operation Timeout。

业务 deadline：

- OTP 等待。
- Magic Link 等待。
- OAuth 回调。
- Challenge/用户同意。
- 整个注册事务。

Timeout 后记录当前状态、是否有副作用、结果是否已知、是否可查询、恢复入口和剩余 deadline。

## Concurrency and Backpressure

并发不是“开大量线程”。控制：

- 授权速率和公开配额。
- 每事务 Session 隔离。
- 邮箱/验证资源限制。
- 连接池大小。
- 内存和 CPU。
- 队列长度和 backpressure。
- 日志/指标基数。
- 取消和清理。

不要使用 IP、账号、设备、邮箱或 Session 轮换规避限制。

## Token Store

TokenStore 应区分：

- Access Token。
- Refresh Token。
- ID Token。
- Session Cookie。
- OAuth state/nonce/verifier。

记录 metadata 而非秘密日志：issuer、audience、scope、expires_at、received_at、rotation_version、source endpoint。

Refresh Token 轮换时必须原子更新；并发请求不能继续使用已失效旧值。

## Verification Adapter

将外部验证抽象为接口：

```python
class VerificationAdapter:
    def wait_for_result(self, context, deadline): ...
```

实现可以是：

- 本地测试 inbox。
- 人工输入。
- 官方测试 Challenge。
- OAuth callback listener。
- 浏览器/认证器交互。

适配器必须返回关联证据，不只返回一个字符串：Message-ID、received_at、transaction correlation、result type。

## Browser-Assisted Boundary

保留浏览器路径的典型情况：

- Passkey/WebAuthn。
- 第三方身份提供方登录和用户同意。
- Challenge/MFA。
- 复杂 Service Worker 或前端内存状态。
- 频繁变化且无稳定公开 API 的流程。

浏览器与协议客户端共享同一状态机和 Context 语义，但执行器不同。

## Secret Management

禁止日志记录：密码、OTP、Access/Refresh Token、Session Cookie、Client Secret、authorization code、PKCE verifier 和私钥。

使用：

- 环境或秘密管理系统。
- 最小权限。
- 短期测试凭证。
- 轮换和撤销。
- 内存中最短必要生命周期。
- 结构化日志字段白名单。

## Observability Interface

每个请求事件至少包含：

```text
timestamp
run_id
context_id
state
last_successful_state
endpoint_role
method
host_role
path_role
http_status
latency_ms
request_id / trace_id
error_category
business_error_code
retry_count
result_known
recommended_action
```

不要默认记录完整 URL、Header、Body 或 Storage。

## Testing Strategy

### Unit Tests

- Error classification。
- State transitions/invariants。
- Token expiry/rotation。
- Retry decision。
- Redaction。
- Cookie/transaction isolation。

### Integration Tests

- 从干净上下文完成正常流程。
- 过期 Cookie/token。
- OTP 重发和旧 code。
- 429 + Retry-After。
- Timeout 后状态确认。
- 409 duplicate。
- OAuth state/nonce/PKCE mismatch。
- Browser-to-protocol handoff。

### Fault Tests

- 网络中断。
- 响应丢失。
- 服务端 5xx。
- Challenge/MFA。
- 并发 Context 串线检测。
- 日志秘密扫描。

## Engineering Definition of Done

- [ ] 每个事务有独立 Context。
- [ ] Transport 与业务分离。
- [ ] 状态机验证所有动作前置条件。
- [ ] 一次性值有消费/失效管理。
- [ ] 非幂等操作结果未知时不会盲重试。
- [ ] 429/Retry-After 和策略拒绝被尊重。
- [ ] timeout 分层且有业务 deadline。
- [ ] Token 角色与轮换清晰。
- [ ] Browser-Assisted 边界明确。
- [ ] 结构化日志可定位最后成功状态。
- [ ] 日志和错误中无秘密。
- [ ] 正常路径和主要失败路径均测试。
