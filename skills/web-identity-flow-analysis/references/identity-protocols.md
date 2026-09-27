# Identity Protocol Reference

本文件是按需速查，不替代证据工作流。任何概念都必须回答：属于哪一层、谁产生、在哪里使用、生命周期多长、失败如何改变状态。

## Network and HTTP

### Request Path

```text
DNS -> TCP or QUIC -> TLS -> HTTP -> CDN/WAF -> identity/business service
```

排错先定位层级：

- DNS/TCP/TLS 未成功时，不要修改 JSON 业务参数。
- HTTP 已返回业务错误时，进入 Header、Body、状态和凭证分析。
- 浏览器协议版本与脚本不同不自动等于根因；必须有差分证据。

### TLS

关注：TLS version、cipher suites、SNI、ALPN、证书链、Session Resumption 和扩展。连接特征可能是风险信号之一，但看到 403 不能直接断言为 TLS 指纹问题。授权排障优先使用真实浏览器、官方 SDK 或标准协议栈，而不是伪造特征。

### HTTP Versions

- HTTP/1.1：文本化语义、连接复用能力有限。
- HTTP/2：二进制帧、Stream、多路复用、HPACK。
- HTTP/3：QUIC、Stream、QPACK。

业务语义与传输组织分开判断。Method、Header、Body 可不变，底层版本不同。

### Methods and Side Effects

Method 不能单独证明幂等性：

- GET 通常读取，但仍需验证是否有副作用。
- POST 常用于动作或创建，可能非幂等。
- OPTIONS 常是 CORS preflight。
- 发送验证码、创建账号、消费一次性凭证不能因为“请求可发”就安全重试。

### Status Codes

| Status | Typical meaning | Identity-flow questions |
|---|---|---|
| 200 | HTTP 成功 | Body 是否业务失败？状态是否改变？ |
| 201 | 创建成功 | 创建了哪个对象？登录态是否已建立？ |
| 202 | 已接受 | 是否异步？如何轮询/回调？ |
| 204 | 成功无 Body | Header/Cookie 是否改变？ |
| 302/303 | 跳转 | Method 如何变化？完整 Redirect Chain？ |
| 307/308 | 保留 Method/Body | 是否跨域、是否携带凭证？ |
| 400 | 格式/业务参数 | Content-Type、字段和协议版本 |
| 401 | 无有效认证 | Cookie/token/过期/aud/登录态 |
| 403 | 理解但拒绝 | 权限、CSRF、账号状态、边缘策略、额外验证 |
| 409 | 状态冲突 | 前一步是否已成功、重复创建、事务已推进 |
| 410 | 已失效 | 一次性或过期资源 |
| 422 | 语义校验 | 必填、格式、密码/地区/年龄政策 |
| 425 | 过早/重放风险 | 是否需重新建立合法时序 |
| 429 | 限流 | Retry-After、维度、无意义重试 |
| 5xx | 服务/网关异常 | Request ID、是否可安全重试、结果是否未知 |

HTTP 成功不等于业务成功；同时读取业务错误码、Body、Set-Cookie 和状态变化。

### Header Categories

1. 内容协商：Accept、Accept-Encoding、Accept-Language、Content-Type。
2. 来源与导航：Origin、Referer、Sec-Fetch-*。
3. Client Hints：User-Agent、Sec-CH-UA-*。
4. 身份与状态：Authorization、Cookie、CSRF、自定义 transaction/session token。
5. 缓存：Cache-Control、ETag、If-None-Match。
6. 追踪：traceparent、tracestate、x-request-id、x-correlation-id。
7. 协议特定：WebSocket/SSE/streaming。

不要机械复制全部 Header，也不要迷信 Header 顺序；判断字段的产生者、语义、请求上下文和生命周期。

## Cookie, Session, Storage, Origin

### Cookie

分析 `Set-Cookie` 时记录：

- Host-only/Domain。
- Path。
- Secure。
- HttpOnly。
- SameSite。
- Expires/Max-Age。
- `__Secure-` / `__Host-` 前缀。

协议客户端应使用真正的 Cookie Jar，正确处理 Domain、Path、Secure、Expiry、Redirect 和同名 Cookie。

### Session

Cookie 可能只是服务端 Session 的索引。拥有 Cookie 不自动意味着已登录；服务端 Session 可能过期、撤销、进入其他阶段或与额外条件绑定。

### Storage

- localStorage：同源持久化，JS 可读。
- sessionStorage：标签页/浏览上下文生命周期。
- IndexedDB：结构化前端状态、SDK 缓存和离线数据。
- 页面内存：state/nonce/verifier 等可能根本没有写入持久化 Storage。

Network 只显示网络结果；字段来源可能在 Storage、页面配置、内存、Service Worker 或第三方 SDK。

### Origin, Site, CORS, CSRF

- Origin = scheme + host + port。
- Same-Origin Policy 限制浏览器脚本访问。
- Site/SameSite 与 Origin 不完全相同。
- CORS 是浏览器执行的跨源读取控制，不等于服务器授权。
- CSRF 防护证明敏感操作来自预期会话/站点上下文，不是登录凭证。

常见 CSRF 组合：Session Cookie + CSRF token + Origin/Referer/SameSite。只复制 Body 通常不够。

## Authentication and Authorization

- Authentication：你是谁。
- Authorization：你能做什么。

已登录仍可能无某项权限；403 不自动代表登录失败。

### Session vs Token Authentication

Session-based：客户端持 Cookie，服务端持状态。

Token-based：客户端持 token，服务端验证或查询 token。

现实系统常为混合模式，不要假设“登录一定是 JWT”或“只有 Cookie”。

## JWT, JWS, JWE, JWK

JWT 常为 `header.payload.signature`。

- Header：alg、kid、typ。
- Claims：iss、sub、aud、exp、iat、nbf、jti。
- Signature：证明完整性和签发方。
- JWS：签名/完整性。
- JWE：加密/机密性。
- JWKS：公钥集合，kid 用于选钥。

Base64URL 解码只是未验证读取，不等于签名有效、声明可信或 token 可伪造。验证至少考虑算法、签名、iss、aud、时间条件和密钥轮换。

## Access, Refresh, and ID Tokens

| Token | Main role | Misuse to avoid |
|---|---|---|
| Access Token | 访问资源 API | 不应被当成身份资料随意解析 |
| Refresh Token | 换取新 Access Token | 不用于普通 API |
| ID Token | OIDC 向 Client 描述认证结果 | 不是通用资源凭证 |

记录每类 token 的 audience、issuer、scope、expiry、rotation、storage、revocation 和并发更新行为。

## OAuth 2.0 Authorization Code

角色：Resource Owner、Client、Authorization Server、Resource Server。

简化时序：

```text
Client -> Authorization endpoint
User authenticates/consents
Authorization server -> short-lived one-time code
Client -> token endpoint with code (+ verifier)
Authorization server -> access/refresh/id tokens
Client -> resource server
```

Authorization Code 不是 Access Token；通常短期、一次性、与 client/redirect/PKCE 绑定。

### OAuth Fields

- `client_id`：客户端身份。
- `redirect_uri`：回调约束。
- `scope`：请求权限。
- `state`：授权请求/回调关联与 CSRF 防护。
- `code`：一次性兑换材料。

## PKCE

```text
random code_verifier
  -> SHA-256 + Base64URL
code_challenge
  -> authorize request
authorization code
  -> token request with original verifier
```

- state 绑定授权请求和回调。
- PKCE 绑定授权码兑换。
- 两者职责不同。

不要记录原始 verifier；记录其生命周期和保存位置即可。

## OpenID Connect

OIDC 在 OAuth 2.0 上增加身份层：

- ID Token。
- UserInfo Endpoint。
- 标准 Claims。
- `nonce`：把 ID Token 与当前认证请求绑定并降低重放风险。

state、nonce、verifier 必须分别追踪生成、存储、传递和校验。

## WebAuthn and Passkey

WebAuthn 更接近“证明持有与 RP 绑定的私钥”，不是共享密码。

### Registration Ceremony

Server challenge -> `navigator.credentials.create()` -> authenticator creates key pair -> private key stays protected -> public credential returned to server。

### Authentication Ceremony

Server challenge -> `navigator.credentials.get()` -> authenticator signs -> server verifies with stored public key。

关注：RP ID、Credential ID、Challenge、Attestation、Assertion、CBOR、COSE Key、Signature Counter。

简单 HTTP 重放不能代替认证器私钥操作。将 WebAuthn 建模为浏览器/认证器边界。

## OTP and Magic Link

### OTP Security Properties

- 随机。
- 短有效期。
- 尝试次数限制。
- 重发限制。
- 与联系方式和 transaction 绑定。
- 成功后立即失效。

OTP 通常证明在短时间窗口内控制某个联系方式，不自动证明完整身份。

### OTP States

```text
NOT_SENT -> SENT -> VERIFIED
               -> EXPIRED
               -> LOCKED
               -> RESENT (old code invalidated)
```

“错误验证码”“过期验证码”“拿错旧邮件”“事务不匹配”“已成功后重复提交”应为不同错误。

### Mail Correlation

邮件处理不应只用正则找六位数字。记录：

- multipart text/plain 与 text/html。
- 标题、正文、链接参数。
- 流程开始时间与收件时间。
- 发件人、主题、Message-ID。
- 当前 transaction/context_id。
- 共享邮箱的去重、锁和消费规则。

区分：暂无新邮件、已到但解析失败、旧邮件、过期邮件、并发任务误取。

### Magic Link

可能包含或关联用户、transaction、过期、一次性状态和签名，有时还要求在原始浏览器会话打开。不要将链接 token 当作普通长期凭证。

## Password Handling

HTTPS 保护传输；服务端仍需安全密码存储。常见密码 KDF：Argon2id、bcrypt、scrypt、PBKDF2。密码存储应慢、可调成本、有随机 salt；SHA-256 不能直接替代密码 KDF。

客户端哈希不能代替服务端密码存储设计；是否有前端变换必须以实际协议和调用证据为准。

## Front-End Runtime

### SPA and Requests

页面视觉步骤不等于服务端状态。React/Vue/Svelte/Next 等应用可能只更新路由和内存状态。

fetch 特点：

- `credentials` 影响 Cookie 行为。
- 4xx/5xx 不自动 reject；需检查 `response.ok` 和业务 Body。

XHR 仍常见于旧项目和 SDK。

### Service Worker

Service Worker 可直接响应缓存、修改或转发请求。页面行为和 Network 预期不一致时检查注册状态、fetch handler 和缓存。

### Bundler and Source Map

不必反混淆所有 JS；定位：

1. 哪个函数发请求。
2. 哪些字段在这里生成。
3. 请求前读取哪些状态。
4. 响应如何改变状态。

### Dynamic Fields

字段可能来自：服务端、浏览器自动、前端随机、用户输入、第三方组件。先找产生者，再解释角色。

## Risk and Edge Controls

### Layers

- WAF：协议和攻击模式。
- Bot Management：自动化/异常客户端判断。
- Risk Engine：账号、网络、设备、行为和策略综合决策。

可能输出 Allow、Deny、Additional Verification、Rate Limit、Manual Review。

### Rate Limit

常见算法：Fixed Window、Sliding Window、Token Bucket、Leaky Bucket。维度可能是 IP、Session、Account、Email、Endpoint、Device、API Key 或组合。

429 排障记录：阶段、主体、限流维度线索、Retry-After、恢复时间和是否存在无意义重试。不得通过轮换身份或网络规避。

### Challenge

记录出现阶段、提供方、结果有效期、会话/事务绑定和失败后状态。合法自动化使用测试机制、人工交互、真实浏览器或官方支持流程。

### Browser/Device Signals

信号可能包括 User-Agent、Client Hints、Locale、Timezone、Viewport、Canvas/WebGL、网络和历史行为。用于解释环境差异和误报，不用于伪装。

## Protocol Distinctions Quick Check

分析结束前确认：

- [ ] Cookie 与 Session 未混淆。
- [ ] Authentication 与 Authorization 未混淆。
- [ ] CSRF token 与登录凭证未混淆。
- [ ] state、nonce、PKCE verifier 未混淆。
- [ ] Authorization Code、Access Token、Refresh Token、ID Token 未混淆。
- [ ] JWT 解码与验证未混淆。
- [ ] OTP、账号创建、登录态未混淆。
- [ ] CORS 与服务器授权未混淆。
- [ ] Challenge 被视为安全门控。
- [ ] 页面步骤与服务端状态未混淆。
