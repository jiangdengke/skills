# Checklists and Labs

## Final Analysis Checklist

每项都填写“答案 + 证据位置”，不能只打勾。

### Network Layer

- [ ] DNS 是否正常，目标是否经过 CDN/WAF/网关？
- [ ] 使用 HTTP/1.1、HTTP/2 还是 HTTP/3？
- [ ] TLS、SNI、ALPN 和证书是否正常？
- [ ] 是否跨多个域名或身份域？
- [ ] 网络、TLS 和 HTTP 错误是否分层？
- [ ] connect/read/overall timeout 是否区分？

### Browser and Session Layer

- [ ] 第一个 Session Cookie 在哪个响应产生？
- [ ] Cookie Domain/Path/Secure/HttpOnly/SameSite/Expiry 是什么？
- [ ] 客户端是否使用正确 Cookie Jar？
- [ ] Cookie 与服务端 Session 是否区分？
- [ ] CSRF token 在哪里产生、保存和消费？
- [ ] Origin/Referer/Sec-Fetch-* 的上下文是什么？
- [ ] localStorage/sessionStorage/IndexedDB/页面内存是否保存流程状态？
- [ ] Service Worker 是否拦截请求？
- [ ] Redirect Chain 是否完整？

### Registration and Verification Layer

- [ ] 哪个请求初始化 registration transaction？
- [ ] transaction ID 在哪里返回和保存？
- [ ] 哪一步发送验证？
- [ ] 哪一步确认验证？
- [ ] OTP/Magic Link 是否与 transaction、Session 和联系方式绑定？
- [ ] 重发后旧 code 是否失效？
- [ ] 邮件如何通过 Message-ID、时间和上下文关联？
- [ ] 哪一步真正创建/激活账号？
- [ ] 哪一步仅完成验证但还未创建账号？

### Authentication Layer

- [ ] 最终登录态使用 Cookie 还是 token？
- [ ] Access/Refresh/ID Token 是否区分？
- [ ] JWT 是否完成签名、iss、aud 和时间验证？
- [ ] OAuth state 在哪里生成和校验？
- [ ] PKCE verifier/challenge 在哪里生成和保存？
- [ ] OIDC nonce 如何与 ID Token 绑定？
- [ ] Authorization Code 是否一次性并已消费？
- [ ] WebAuthn/Passkey 是否保留浏览器/认证器边界？
- [ ] 登录态建立后是否验证受保护资源？

### Front-End Layer

- [ ] 关键 Body 在哪个函数构造？
- [ ] 动态字段由用户、服务端、随机数、环境还是第三方产生？
- [ ] 是否使用 WebCrypto；其目标是 Hash、HMAC、加密、签名还是 KDF？
- [ ] SPA 页面步骤是否与服务端状态分开？
- [ ] Source Map/Call Stack 是否提供字段来源证据？
- [ ] 浏览器自动生成 Header 是否被误当业务字段？

### Risk and Policy Layer

- [ ] 是否出现 WAF/Bot/Risk/Challenge 响应？
- [ ] Challenge 在哪个状态出现，合法完成方式是什么？
- [ ] 429 的 Retry-After、维度和恢复时间是否记录？
- [ ] 403 是否区分权限、CSRF、账号状态和策略拒绝？
- [ ] 是否遵守速率、并发和停止条件？
- [ ] 是否避免设备/TLS/浏览器信号伪装？

### Engineering and Observability

- [ ] 每个状态有明确进入/退出条件？
- [ ] 请求依赖图、字段字典和状态机一致？
- [ ] 每个事务的 Cookie/token/transaction/日志完全隔离？
- [ ] 哪些错误可重试，依据是什么？
- [ ] 哪些请求非幂等？
- [ ] timeout 后未知结果如何确认？
- [ ] 是否保存最后成功状态和 Request/Trace ID？
- [ ] 日志是否白名单化并脱敏？
- [ ] 差分测试是否一次只改一个变量？
- [ ] 故障注入是否覆盖主要失败路径？

## Graduation Questions

以下 15 个问题必须指向同一组基线、依赖图、状态机和日志证据：

1. 流程一共有几个服务端/客户端状态？
2. 哪个请求初始化会话？
3. 哪个请求真正创建或激活账号？
4. 哪个请求发送验证码或验证消息？
5. 哪个请求确认验证码、链接或外部证明？
6. 哪些请求必须按顺序执行，为什么？
7. 哪些 Cookie 必须保留，其作用域是什么？
8. 哪些 token/code 是一次性的？
9. 哪些 Cookie/token 代表最终登录态？
10. 哪些字段来自服务端？
11. 哪些字段由前端、浏览器、认证器或第三方动态生成？
12. 哪些错误可以安全重试？
13. 哪些请求必须保证幂等或先查询状态？
14. 登录态建立后如何确认账号和权限确实有效？
15. 流程变化时应该从基线、分类、字段字典、状态机或前端追踪中的哪个阶段重新开始？

任何问题不能用证据回答时，标记为未完成，不宣称流程已完全逆向。

## Common Mistakes

1. 把注册理解成一个 POST。
2. 只看 Request Body，不看 Set-Cookie。
3. 不保存 Redirect Chain。
4. 看到 JWT 就认为它能伪造。
5. 把 ID Token 当 Access Token。
6. 把 state、nonce、PKCE 混为一谈。
7. 把 Cookie 和 Session 当同一个东西。
8. 用全局 Cookie Jar 跑多个事务。
9. 所有错误都自动重试。
10. 429 后立即重试或轮换身份规避。
11. OTP 错误后循环猜测。
12. 不记录服务端 Request ID。
13. 只看最终错误，不看最后成功状态。
14. 把所有 403 归因于“指纹”。
15. 把 CORS 当服务器授权机制。
16. 复制一条 cURL 就认为逆向完成。
17. 不分析前端字段来源。
18. 不检查 Storage。
19. 忽略 Service Worker。
20. 不区分 302、303、307、308。
21. 不区分 connect/read/operation timeout。
22. 日志打印密码、OTP、Cookie 或 token。
23. 不处理 token 过期和轮换。
24. 不做状态机，只堆 if/else。
25. 把“能跑一次”当作“工程稳定”。

## Safe Learning Labs

不要拿生产第三方身份系统当练习靶场。搭建最小本地服务：

```text
/signup
/api/session/init
/api/registration/start
/api/verification/send
/api/verification/confirm
/api/registration/complete
/api/session/check
/protected/profile
```

使用本地测试 inbox、合成账号和测试 token。

### Lab 1 — Cookie and Session

目标：

- 服务端创建匿名 Session。
- 浏览器收到 Cookie。
- 后续请求必须带 Cookie。

验证：

- Cookie 在哪里设置。
- Domain/Path/Expiry。
- 删除 Cookie 后的状态。
- 服务端 Session 过期后“Cookie 仍在”为什么无效。

### Lab 2 — CSRF

目标：GET 初始化时生成 CSRF，POST 必须同时带 Session Cookie + CSRF。

验证 Session ID、CSRF Token 和 Transaction ID 的差异。测试缺失、错误、过期和跨会话值。

### Lab 3 — OTP and Mail Correlation

目标：

- OTP 有效期。
- 尝试次数。
- 重发后旧 OTP 失效。
- OTP 与 transaction 绑定。
- 本地 inbox 的 MIME/Message-ID/时间关联。

故障：旧邮件、解析失败、过期、并发任务误取。

### Lab 4 — OAuth + PKCE + OIDC

使用本地或测试 Authorization Server：

- 生成 state、nonce、verifier/challenge。
- 执行 authorization code flow。
- 验证 state、PKCE、nonce、iss、aud、exp。
- 区分 Access/Refresh/ID Token。

故障：state mismatch、wrong verifier、reused code、expired code、nonce mismatch。

### Lab 5 — Rate Limit and Error Taxonomy

让服务端返回 429 + Retry-After。客户端应：

- 不无限重试。
- 尊重等待时间。
- 记录状态和 Request ID。
- 区分 429、网络错误和策略拒绝。

### Lab 6 — Idempotency and Unknown Results

模拟创建请求成功但响应丢失：

- 客户端标记 `result_known=false`。
- 通过查询或 Idempotency Key 确认。
- 不重复创建。

### Lab 7 — Session Isolation

并发两个 Context，主动注入错误共享 Cookie/token，确保测试能检测串线并阻止继续。

### Lab 8 — Observability and Redaction

验证日志包含状态、Request ID、错误类别和恢复建议，同时自动扫描并确认不存在密码、JWT、Cookie、OTP 和秘密。

## Learning Levels

### Level 1 — HTTP and Browser

能力：Method、Status、Header、Redirect、Cookie、Origin/CORS/CSRF、DevTools Network。

验收：从 HAR 区分静态、遥测、配置、身份主链和风险请求。

### Level 2 — Identity Protocols

能力：Session、JWT/JWS/JWK、OAuth Code、PKCE、OIDC、state/nonce、OTP/Magic Link。

验收：解释每类 token 的角色、生成者、使用者和生命周期。

### Level 3 — Front-End Tracing

能力：fetch/XHR、SPA、Source Map、Breakpoint、Call Stack、Service Worker、WebCrypto。

验收：从关键请求追到 payload 构造和字段来源。

### Level 4 — Protocol Client Engineering

能力：Cookie Jar、Session Isolation、State Machine、Retry/Backoff、Idempotency、Logging、Redaction、Error Taxonomy。

验收：客户端可测试、可恢复、可排障，且失败不会扩大副作用。

### Level 5 — Advanced Network and Security

能力：TLS/SNI/ALPN、HTTP/2/3、CDN/WAF/Bot/Risk、WebAuthn。

验收：能把网络、会话、身份、前端、风险和日志放入同一因果链，并知道流程变化后从哪里重建基线。

## Source Coverage Map

此 skill 将原文 18 篇映射为：

| Original area | Skill file |
|---|---|
| 网络与 HTTP | identity-protocols.md |
| 浏览器状态与会话 | identity-protocols.md / evidence-workflow.md |
| 认证与身份协议 | identity-protocols.md |
| 邮箱、OTP、账号生命周期 | identity-protocols.md / client-engineering.md |
| 前端 JS 与执行模型 | identity-protocols.md / evidence-workflow.md |
| 反滥用与边缘安全 | safety-and-scope.md / diagnostics.md |
| HAR 与逆向方法 | evidence-workflow.md / templates.md |
| 协议客户端工程 | client-engineering.md |
| 错误与可观测性 | diagnostics.md |
| 浏览器自动化 vs 协议 | evidence-workflow.md / client-engineering.md |
| 安全实验室 | 本文件 |
| 六阶段工作流 | SKILL.md / evidence-workflow.md |
| 代码设计思维 | client-engineering.md |
| 25 个错误 | 本文件 |
| 学习路线 | 本文件 |
| 最终检查表 | 本文件 |
| 术语 | identity-protocols.md |
| 最终结论 | SKILL.md Definition of Done |

## Final Definition of Done

- [ ] Safety Gate 已通过。
- [ ] 有成功基线或明确限制为失败诊断。
- [ ] Endpoint Inventory 完整。
- [ ] 请求依赖图显示顺序、数据、会话、一次性、时间和外部事件。
- [ ] 字段字典覆盖所有关键动态值。
- [ ] 状态机有成功、失败、等待、冷却和终态。
- [ ] 最小复现从干净 Context 开始，不固定重放一次性值。
- [ ] 浏览器/认证器/人工边界明确。
- [ ] 差分测试遵循单变量。
- [ ] 故障注入覆盖未知结果和非幂等副作用。
- [ ] 错误类别、根因层、Request ID 和最后成功状态可追踪。
- [ ] 并发事务隔离。
- [ ] 日志和报告无秘密。
- [ ] 15 个毕业问题有证据。
- [ ] 未验证内容标为 Hypothesis/Unknown。
