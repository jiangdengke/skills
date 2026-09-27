---
name: web-identity-flow-analysis
description: "Analyze and engineer authorized modern Web registration, login, email/OTP/Magic Link, OAuth 2.0/OIDC/PKCE, JWT, WebAuthn/Passkey, Cookie/Session, HAR/DevTools, and protocol-client flows. Use when the user asks to 分析注册或登录流程、阅读 HAR/Network、追踪 Cookie/token/state/nonce/PKCE、建立请求依赖图或状态机、比较浏览器与 API 客户端、诊断 401/403/409/422/429/5xx、设计重试/幂等/超时/日志，或在自有、测试、沙箱或明确授权系统中实现身份协议客户端。Do not use to bypass CAPTCHA, risk controls, rate limits, MFA or access controls, or for bulk account creation, credential attacks, token theft, replay, or account takeover."
compatibility: "Requires authorized evidence such as HAR, logs, code, screenshots, or a local/test environment."
metadata:
  language: zh-CN
  version: "1.0.0"
  generated: "2026-09-27"
  source-document: "现代Web注册与身份流程逆向分析_逐节深融重写版.docx"
  source-sha256: "486f89419d5d464db80aac447b8100560de546135bcdf9eaf19ec795c3662bee"
---

# Web Identity Flow Analysis

## Purpose

在明确授权范围内，把现代 Web 注册、登录和身份验证流程从“很多请求”还原为可解释、可验证、可恢复的协议模型。最终交付不应只是一条 cURL 或一个能偶尔运行的脚本，而应包含：

- 成功基线与证据索引。
- Endpoint Inventory 与请求分类。
- 请求依赖图和字段数据字典。
- 有限状态机及失败/恢复路径。
- 渐进式最小复现或实现边界。
- 单变量差分实验与故障注入结果。
- 隔离、幂等、重试、超时、并发和秘密管理设计。
- 错误分类、最后成功状态、Request/Trace ID 和脱敏日志。
- 带证据的完成检查。

核心原则：**先建立成功基线，先找状态依赖，再写自动化；先证明字段来源，再判断字段必要性；先分类错误，再决定恢复。**

## Load References

执行任何具体分析前：

1. 始终先读 [references/safety-and-scope.md](./references/safety-and-scope.md)。
2. 始终再读 [references/evidence-workflow.md](./references/evidence-workflow.md)。
3. 按任务选择其他资料：
   - HTTP、Cookie/Session、OAuth/OIDC/PKCE、JWT、OTP、WebAuthn、前端运行时：读 [references/identity-protocols.md](./references/identity-protocols.md)。
   - 实现或审查协议客户端：读 [references/client-engineering.md](./references/client-engineering.md)。
   - 排查失败、设计日志、重试或恢复：读 [references/diagnostics.md](./references/diagnostics.md)。
   - 需要标准表格或报告结构：读 [references/templates.md](./references/templates.md)。
   - 教学、实验室、最终核对和能力验收：读 [references/checklists-and-labs.md](./references/checklists-and-labs.md)。

不要一次性加载所有参考文件；只加载当前阶段需要的内容。

## Safety Gate

在读真实 HAR、编写请求、做差分实验、重放操作或实现客户端之前，确认：

- 目标属于用户、自建实验室、开发/沙箱环境，或存在明确书面授权。
- 授权范围包含环境、端点、测试账号、时间窗口、速率/并发上限、数据处理和停止条件。
- 操作不会以批量注册、账号农场、撞库、密码喷洒、OTP 猜测、账号接管、越权、秘密窃取或规避控制为目的。
- CAPTCHA、Challenge、WAF、Bot Management、Risk Engine、MFA、WebAuthn、Rate Limit、CSRF、OAuth state/PKCE 和会话绑定被视为安全边界，不是待删除参数。

若授权或目的不清：只提供抽象解释、脱敏分析、本地实验、合成数据和防御性建议。若请求明确要求绕过、伪造、窃取、批量滥用或规避限流，拒绝目标操作并提供安全替代。

## Input Contract

### Minimum Evidence

完整分析至少需要：

1. 一条从干净上下文完成的成功基线，或明确说明当前只有失败样本。
2. HAR 或等价的请求/响应证据，尽量保留重定向、响应头和 Set-Cookie。
3. 用户动作和关键时间点。
4. 关键步骤前后的 Cookie/Storage/页面状态变化。
5. 最终结果证据：账号创建、验证完成、登录态建立和受保护资源访问分别是否成功。

### Useful Evidence

按需使用：

- Console、截图或录屏。
- XHR/fetch 断点、调用栈和 payload 构造位置。
- Source Map、关键 JS chunk、Service Worker。
- 服务端日志、Request ID、Trace ID、Correlation ID。
- 邮件 MIME、Message-ID、发件人、主题、收件时间和流程开始时间。
- 多条成功/失败样本。

### Data Handling

不要要求或输出真实密码、OTP、Magic Link、Cookie、Authorization、Access/Refresh/ID Token、OAuth code、PKCE verifier、CSRF token、Client Secret、API Key、私钥或 WebAuthn 私密材料。用户若已暴露秘密，不验证其有效性；建议立即撤销、轮换并清理副本。

## Evidence Rules

对每个关键结论标记证据等级：

- **Observed**：HAR、响应、Cookie/Storage、调用栈或日志直接证明。
- **Differentially verified**：单变量实验稳定复现。
- **Strong inference**：多个证据一致但未看到服务端实现。
- **Hypothesis**：仅依据命名、长度、格式或单一样本。

每个关键结论至少绑定：样本/运行编号、步骤、请求角色、时间戳或 Request ID、证据位置和置信度。不要把假设写成事实。

## Five Questions for Every Key Request

分析每个关键请求时固定回答：

1. 它属于网络、浏览器会话、身份凭证、注册业务还是风险/策略哪一层？
2. 它依赖哪些旧状态、Cookie、token、Storage、用户动作或外部事件？
3. 每个输入字段由谁产生、在哪里保存、生命周期多长？
4. 成功后产生、改变或消费了什么状态？
5. 输出在哪一步被保存，后续哪个请求如何引用？

额外检查：是否一次性、是否绑定会话/事务、是否有时效、是否有副作用、失败属于哪一层、是否可以安全重试。

## Mandatory Workflow

### Phase 0 — Scope and Sanitize

- 执行 Safety Gate。
- 对 HAR、日志、URL、邮件和截图脱敏。
- 建立 `run_id`、`context_id` 和证据目录/索引。
- 明确允许的动作、禁止动作、配额和停止条件。

输出：Scope Record、Sanitization Record。

### Phase 1 — Establish a Successful Baseline

- 优先使用干净浏览器 Profile 或清晰记录的初始状态。
- 从入口开始完成正常流程，不急于删 Header、改参数或写脚本。
- 保存 HAR、重定向链、Cookie/Storage 快照、关键页面状态和用户动作时间线。
- 分开确认：匿名会话、注册事务、验证状态、账号创建/激活、登录态、受保护资源。

没有成功基线时，可以做失败定位和证据缺口分析，但不得宣称完整协议已恢复。

输出：Baseline Run Record、Timeline、State Snapshots、Final Success Evidence。

### Phase 2 — Classify Requests

将请求至少分成：

- 静态资源。
- Telemetry/Analytics/Error reporting。
- 配置/Feature Flag/Localization。
- 身份和注册主链。
- OAuth/OIDC 跳转或回调。
- 风险/Challenge。
- 受保护资源验证。

为真正改变状态的请求建立 Endpoint Inventory。角色名优先于实际 URL，例如 `INIT_SESSION`、`START_REGISTRATION`、`SEND_VERIFICATION`、`CONFIRM_VERIFICATION`、`COMPLETE_ACCOUNT`、`ESTABLISH_SESSION`。

输出：Endpoint Inventory。

### Phase 3 — Build Dependency Graph and Field Dictionary

- 画出请求的顺序依赖、数据依赖、会话绑定、一次性消费、时间约束和人工/外部事件。
- 对每个关键字段记录：实际名、语义名、类型/编码、来源、首次出现、保存位置、生命周期、一次性、会话/事务绑定、使用位置、敏感性、可选性、失败特征、证据与置信度。
- 字段来源分类：用户输入、前端配置、服务端返回、客户端随机、环境派生、第三方组件。
- 生命周期优先于变量名；不要凭字符串长度猜算法或角色。

必须严格区分 Session Cookie、CSRF token、Transaction ID、OAuth state、OIDC nonce、PKCE verifier/challenge、Authorization Code、Access Token、Refresh Token、ID Token、OTP/Magic Link token。

输出：Request Dependency Graph、Field Dictionary。

### Phase 4 — Model the State Machine

- 用服务端响应、Set-Cookie、凭证变化、验证消费或资源访问证明状态，不用页面文案代替证据。
- 每个转换记录前置条件、动作、成功证据、下一状态、副作用、失败状态、是否可重试和恢复方式。
- 至少考虑：`INIT`、`SESSION_READY`、`REGISTRATION_STARTED`、`VERIFICATION_REQUIRED`、`VERIFICATION_SENT`、`WAITING_EXTERNAL_EVENT`、`VERIFIED`、`PROFILE_REQUIRED`、`COMPLETING`、`AUTHENTICATED`、`DONE`、`COOLDOWN`、`CHALLENGE_REQUIRED`、`FAILED_RETRYABLE`、`FAILED_FINAL`。
- OTP 子状态应区分未发送、已发送、已验证、过期、锁定；邮件读取应区分未到达、解析失败、旧邮件和过期邮件。

输出：State Diagram、State Transition Table。

### Phase 5 — Trace Front-End Generation When Needed

当字段来源仍不明时：

- 在 XHR/fetch 发送处暂停，沿 Call Stack 找 payload 构造逻辑。
- 检查页面配置、Storage、Service Worker、Source Map、动态 import、WebCrypto 和第三方 SDK。
- 先确定谁生成字段，再判断它做什么；不要先猜“加密算法”。
- 区分 Hash、HMAC、加密、签名和 KDF 的安全目标。

输出：Field Origin Trace、Relevant Code/Call-Stack Evidence。

### Phase 6 — Reproduce Progressively

只在自有或明确授权环境中：

1. 从稳定的匿名初始化开始。
2. 先复现无副作用读取或会话检查。
3. 验证 Cookie Jar、Redirect、CSRF、Transaction 和 token 传递。
4. 每次只加入一个状态改变请求，并立即验证副作用和下一状态。
5. 必须由浏览器、认证器、人工或官方 SDK 完成的部分保留合法路径。
6. 最后验证受保护资源，不把“接口返回完成”当成唯一成功证据。

不能把完整 HAR 一次性转成脚本；不能固定重放过期值；不能把删除安全步骤当成“最小化”。

输出：Minimal Reproduction Record 或明确的 Browser-Assisted Boundary。

### Phase 7 — Differential Testing and Fault Injection

**差分测试：** 每次只改变一个可控变量；记录基线、单一变化、预期、观察、状态差异、结论、置信度和重置方式。

**故障注入：** 至少覆盖网络中断、connect/read/operation timeout、缺失/过期 Cookie、CSRF/Transaction/token、错误/过期 OTP、重复提交、401/403/409/422/429/5xx、Retry-After、Challenge、并发串线和旧邮件。

差分测试用于验证必要条件；故障注入用于验证失败分类、状态一致性、幂等、恢复和日志。不得用于猜测或规避第三方安全控制。

输出：Differential Experiment Log、Fault Injection Matrix。

### Phase 8 — Engineer the Client

采用分层设计：

```text
Orchestrator
  -> State Machine
    -> Identity Client
      -> HTTP Transport

Side components:
CookieJar / TokenStore / VerificationAdapter / RetryPolicy /
ErrorClassifier / Logger / Metrics / Clock / IdempotencyStore
```

强制规则：

- 每个事务独立 Cookie Jar、Transaction、CSRF、OAuth state/nonce/PKCE、token、计时器、重试计数和日志上下文。
- Transport 只处理 HTTP/TLS/连接池/Redirect/Cookie/timeout，不包含业务动作。
- Orchestrator 管流程，不直接拼 Header。
- 重试基于错误类别、当前状态、副作用、凭证有效期和服务端提示。
- timeout 至少区分 connect、read、overall operation，并设置 OTP/OAuth/事务业务 deadline。
- 对非幂等动作和未知结果先查询状态或使用服务端支持的 Idempotency Key，不盲重放。
- 结构化日志采用字段白名单和脱敏。
- 优先 Browser-Assisted Protocol Analysis，不强求 100% 纯协议。

输出：Architecture、RegistrationContext、Retry/Idempotency/Timeout Policy、Observability Plan。

### Phase 9 — Verify Completion

- 确认依赖图、字段字典和状态机一致。
- 检查第一、中间、最后关键状态及主要失败分支。
- 用“最后一个成功状态”定位任何剩余问题。
- 完成 [references/checklists-and-labs.md](./references/checklists-and-labs.md) 中的最终检查和 15 个毕业问题。
- 明确列出已知未知、未验证假设、风险和下一步。

只有当关键结论都有证据、失败路径可解释且敏感数据处理合格时，才能声明完成。

## Diagnostic Defaults

失败时按以下顺序定位：

1. 找最后一个有明确证据的成功状态。
2. 检查从该状态到下一状态的请求上下文。
3. 检查是否缺少流程前置步骤。
4. 检查动态值是否过期、已消费、来源错误或串会话。
5. 检查是否为限流、Challenge 或策略拒绝。
6. 检查本地状态、并发、旧 token、重复提交和日志关联。

不要把所有 403 归因于“指纹”，不要把所有 401 归因于“密码”，不要把 200 当成业务成功。

## Stop, Ask, or Escalate

必须停止具体操作并转为询问、降级或安全替代的情况：

- 没有目标归属或授权范围。
- 请求涉及真实秘密或未脱敏 HAR。
- 用户要求批量注册、绕过 CAPTCHA/Challenge、伪造设备/TLS/浏览器信号、规避 429、代理/IP 轮换、撞库或账号接管。
- 非幂等动作结果未知，重复可能产生副作用。
- 测试将影响生产、第三方账号或超过公开配额。
- Passkey、MFA、Challenge、OTP 或用户同意必须由合法用户/认证器完成。
- 只有失败样本却要求直接生成“最终协议脚本”。

## Required Deliverables

根据任务范围输出以下全部或适用子集，并注明缺失原因：

1. Scope & Authorization Record。
2. Baseline Run Record。
3. Flow Timeline。
4. Endpoint Inventory。
5. Request Dependency Graph。
6. Field Dictionary。
7. State Transition Table。
8. Front-End Origin Trace（如需要）。
9. Minimal Reproduction / Browser-Assisted Boundary。
10. Differential Experiment Log。
11. Fault Injection Matrix。
12. Error/Retry Matrix。
13. Client Architecture and Context Isolation。
14. Observability and Redaction Plan。
15. Findings、Evidence、Confidence、Unknowns、Risks、Next Actions。

## Output Contract

最终报告按以下顺序组织：

1. **Scope**：目标、授权、环境、安全边界、数据处理。
2. **Executive Summary**：流程、最后成功状态、主要发现和阻塞。
3. **Evidence**：基线、样本和证据索引。
4. **Flow Model**：Endpoint Inventory、依赖图、字段字典、状态机。
5. **Diagnostics**：错误类别、根因层、Request ID、恢复路径。
6. **Engineering**：最小复现边界、架构、隔离、幂等、重试、超时、并发、日志。
7. **Validation**：差分测试、故障注入、完成检查。
8. **Unknowns & Next Actions**：明确未证实内容，不补写猜测。

简短任务可以压缩篇幅，但不能省略安全边界、证据等级和关键状态依赖。

## Definition of Done

完成必须满足：

- 有成功基线，或明确限制为失败诊断。
- 能区分网络、会话、身份、业务和风险/策略层。
- 每个关键请求都有角色、前置、输入、输出、副作用和证据。
- 每个关键字段都有来源、保存位置、生命周期、绑定关系和使用位置。
- 依赖图与状态机一致。
- 已区分账号创建、验证完成、登录态和受保护资源访问。
- 差分结论来自单变量实验。
- 非幂等操作、未知结果和 timeout 有安全恢复策略。
- 并发事务完全隔离。
- 错误有稳定分类，日志记录最后成功状态和 Request/Trace ID。
- 密码、OTP、token、Cookie、PII 和秘密均已脱敏或未采集。
- Challenge、风控、限流和访问控制被尊重。
- 故障注入证明客户端不会无限重试、串会话或重复副作用。
- 15 个毕业问题均能指向证据，或明确标记尚未完成。
