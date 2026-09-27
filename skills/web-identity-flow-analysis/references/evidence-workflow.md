# Evidence-Driven Workflow

## Core Mental Model

现代注册或登录通常不是一个接口，而是一条有前置条件、输出依赖、一次性值、外部事件和安全门控的状态链。典型抽象：

```text
Open entry
  -> initialize anonymous browser/session state
  -> start registration or authentication transaction
  -> submit identifier/profile
  -> policy/risk/format/duplicate checks
  -> verification required
  -> external event: OTP, Magic Link, consent, authenticator
  -> verification confirmed
  -> account created or activated
  -> authenticated session/token established
  -> protected resource proves usable identity state
```

账号创建、验证完成、会话建立和资源授权不是同一个事实。不要用单一 `success=true` 代替这些状态。

## Analysis Layers

固定使用五层模型：

1. **Network/Transport**：DNS、TCP/QUIC、TLS、HTTP 版本、代理、连接、timeout。
2. **Browser/Session**：Cookie、Session、Storage、Origin/Site、CORS、CSRF、Redirect。
3. **Identity/Credential**：Session Authentication、JWT、OAuth/OIDC/PKCE、Token、WebAuthn。
4. **Business/Verification**：Registration Transaction、OTP、Magic Link、资料、账号创建/激活。
5. **Risk/Policy**：WAF、Bot、Risk Engine、Rate Limit、Challenge、地区/年龄/合规策略。

同时追踪六类状态：网络连接、浏览器状态、注册事务、身份状态、凭证状态、风险/策略状态。

## Baseline First

### Why

没有成功基线就没有可靠参照。旧 Cookie、Storage、缓存、重发验证码、刷新页面或多次尝试会污染字段来源与状态顺序。

### Baseline Procedure

1. 建立 `run_id` 和干净 Profile。
2. 记录浏览器/客户端版本、环境和开始时间。
3. 在每个用户动作前后标记时间。
4. 保存完整 HAR，包括 Redirect、Response Header 和 Set-Cookie。
5. 保存关键 Cookie/Storage 差异，不保存秘密原值。
6. 记录 Console 和关键屏幕状态。
7. 分别确认账号对象、验证状态、登录态和资源访问。

### Baseline Questions

- 哪一步首次产生匿名 Session？
- 哪一步首次出现 Registration Transaction？
- 哪一步触发验证？
- 哪一步证明验证成功？
- 哪一步真正创建/激活账号？
- 哪一步建立 authenticated session/token？
- 哪个受保护资源证明登录态可用？

## Request Classification

不要按文件名直觉分类；按作用和副作用分类。

| Class | Typical contents | Analysis action |
|---|---|---|
| Static | JS/CSS/font/image | 保留来源信息，通常不进入主状态机 |
| Telemetry | analytics/performance/error reporting | 与主链分离，避免误认为必要请求 |
| Configuration | runtime config/flags/localization | 记录是否决定 endpoint、client_id 或功能分支 |
| Identity main path | session/start/send/verify/complete | 进入 Inventory、依赖图和状态机 |
| OAuth/OIDC redirect | authorize/callback/token/userinfo | 保存完整时序与 state/nonce/PKCE 关联 |
| Risk/Challenge | initialization/proof/result | 建模为门控，不绕过 |
| Protected resource | account/profile/home API | 用于验证最终身份/授权状态 |

## Endpoint Inventory Rules

每个主链请求使用稳定编号，如 `E01`、`E02`，并记录：

- Role：语义角色，而不是只写 URL。
- Trigger：页面加载、用户动作、回调、timer、Service Worker。
- Pre-state。
- Method/Host/Path。
- Request type：document/fetch/xhr/websocket 等。
- Inputs：Header/Body/Cookie/Storage/用户输入。
- Outputs：Response、Set-Cookie、Redirect、Storage。
- Side effect。
- Dependency。
- Retry safety。
- Evidence。

如果请求没有状态副作用，也可能是必要读取，但必须说明它如何影响后续决策。

## Request Dependency Graph

### Required Edge Types

图中的边必须说明依赖类型：

- `sequence`：时序要求。
- `data`：前一步输出字段被后一步使用。
- `session-binding`：必须属于相同 Cookie/Session/浏览器上下文。
- `one-time-consumption`：Authorization Code、OTP、Magic Link 等消费后失效。
- `external-event`：用户点击邮件、输入 OTP、完成同意、使用认证器。
- `time-window`：过期、冷却、等待。
- `policy-gate`：Challenge、MFA、人工审核或政策条件。
- `optional-branch`：Profile、Consent 或额外验证分支。

### Node Format

```text
E02 START_REGISTRATION
Pre-state: SESSION_READY
Inputs: anon_session, csrf, user_identifier
Action: create registration transaction
Outputs: transaction_id, updated session
Side effects: transaction created
Post-state: REGISTRATION_STARTED
Evidence: HAR run-01 request 18, response body, Set-Cookie
```

### Graph Quality Checks

- [ ] 每个关键值有来源箭头和使用箭头。
- [ ] Redirect Chain 未被折叠丢失。
- [ ] Cookie Domain/Path/SameSite 约束被记录。
- [ ] 一次性值的消费位置明确。
- [ ] 外部事件和等待态明确。
- [ ] 成功、失败、冷却和恢复分支明确。
- [ ] 页面步骤没有被错误当作服务端状态。

## Field Data Dictionary

### Source Classes

- 用户输入。
- 前端常量/运行时配置。
- 服务端返回。
- 客户端随机生成。
- 浏览器/环境派生。
- 第三方组件或认证器返回。

### Lifecycle Investigation

对未知字段按此顺序调查：

1. 首次在哪里出现？
2. 谁生成？
3. 保存在哪里？
4. 哪些请求引用？
5. 刷新页面、换标签、重新建会话后是否变化？
6. 自然等待后是否失效？
7. 成功消费后能否复用？
8. 缺失、错误或过期时的失败特征是什么？

### Lifecycle Categories

- Request-level。
- Page/load-level。
- Transaction-level。
- Authentication-flow-level。
- Session-level。
- Account-level。
- Long-lived configuration。

### Do Not Infer from Shape Alone

- 长字符串不等于“加密”。
- JWT 可解码不等于签名可信或可伪造。
- 名为 `token` 的字段不一定是登录凭证。
- 名为 `nonce` 的字段不一定是 OIDC nonce。
- 时间戳变化不等于它是安全 proof。

只有调用栈、协议时序、消费位置和差分实验能提高结论强度。

## Evidence Grades

| Grade | Definition | Example |
|---|---|---|
| Observed | 直接证据 | Response `Set-Cookie` 后下个请求自动携带 |
| Verified | 单变量实验反复证明 | 删除同一 CSRF 值稳定返回特定错误 |
| Strong inference | 多证据一致 | 值从 start response 返回，仅 verify/complete 使用 |
| Hypothesis | 格式/命名推测 | 仅因长度猜测是 AES 输出 |

报告必须将 Hypothesis 与 Observed 分开。

## State Machine Construction

### State Evidence

进入新状态至少使用一种强证据：

- 服务端业务状态字段。
- Set-Cookie 或 token 建立/变化。
- OTP/Magic Link/Authorization Code 被成功消费。
- 账号/资料对象可查询。
- 受保护资源可访问。
- 服务端日志/Request ID 明确确认。

HTTP 200 或页面跳转本身不是充分证据。

### Transition Fields

每个转换记录：

- Current State。
- Event/Trigger。
- Preconditions。
- Action/Endpoint Role。
- Success Evidence。
- Next State。
- Side Effect。
- Failure State。
- Retryable。
- Recovery。
- Evidence/Confidence。

### Network Uncertainty

网络 timeout 不代表服务端没有执行。对创建、发送、确认、消费等动作：

1. 标记结果未知。
2. 使用查询、后续状态、幂等键或服务端日志确认。
3. 只有证明未执行后才重试。

## Front-End Origin Tracing

当 HAR 只能显示结果时：

1. 定位关键 fetch/XHR。
2. 在发送处暂停。
3. 读取 Call Stack。
4. 向上追到 payload 构造和状态读取。
5. 记录输入来自用户、配置、响应、Storage、随机数、WebCrypto 还是第三方 SDK。
6. 检查 Service Worker 是否拦截或修改请求。
7. 使用 Source Map 或 chunk 搜索定位代码；不必反混淆整个应用。

判断 WebCrypto 时先分类安全目标：

- Hash：摘要。
- HMAC：共享密钥完整性/来源验证。
- Encryption：机密性。
- Signature：私钥持有与完整性。
- KDF：密码或密钥派生。

## Minimal Reproduction

### Meaning of Minimal

最小不是请求最少，而是：

- 每个保留步骤都有必要角色。
- 每个字段有来源。
- 删除任何关键步骤都会失去必要状态。
- 不固定复制一次性或过期值。
- 不删除安全控制。
- 副作用有恢复策略。

### Progressive Order

1. 初始化匿名会话。
2. 无副作用读取/检查。
3. Cookie/Redirect/CSRF/Transaction 连续性。
4. 单个状态改变请求。
5. 外部事件或浏览器边界。
6. 完成/会话建立。
7. 受保护资源验证。

每增加一步，立即检查：响应、Set-Cookie、状态变化、副作用、错误路径和日志。

## Differential Testing

### Discipline

- 一次只改一个变量。
- 使用可比初始状态。
- 记录修改前预期。
- 实验后重置会话/事务，避免污染。
- 结论限定于观察范围。
- 不对第三方安全控制做规避性探测。

### Safe Variables in Authorized Environments

- 可选字段是否省略。
- Content-Type 是否符合协议。
- 合法请求顺序。
- 过期测试 token。
- 幂等请求的重复提交。
- 页面刷新、换标签页或自然等待对生命周期的影响。

### Experiment Record

```text
Experiment ID
Baseline Run
Single Change
Expected Result
Observed HTTP/Business Result
State Delta
Side Effect
Conclusion
Evidence Grade
Reset/Revert
```

## Fault Injection

最低覆盖：

- DNS/连接失败。
- connect/read/overall timeout。
- 响应丢失导致结果未知。
- Cookie 缺失、过期或串线。
- CSRF/Transaction/token 缺失、错误、过期或已消费。
- OTP 错误、过期、旧邮件、重复消费、次数上限。
- 重复提交非幂等动作。
- 401/403/409/422/429/500/502/503/504。
- Retry-After。
- Challenge/MFA/人工验证。
- 并发 Context 串线和旧 token。

故障注入验收：

- 错误分类正确。
- 状态停留/回退/终止正确。
- 不无限重试。
- 不重复副作用。
- 日志记录最后成功状态、Request ID 和恢复建议。
- 敏感数据不泄漏。

## Browser-Assisted Protocol Analysis

推荐抽象顺序：

1. 手工真实浏览器完成正常基线。
2. 浏览器自动化使基线可重复。
3. 提炼请求角色、字段字典和状态机。
4. 将稳定、授权、适合程序调用的部分下沉到协议客户端。
5. 对 Passkey、第三方 SDK、Challenge、用户同意和复杂前端状态保留浏览器/人工路径。

“100% 纯协议”不是默认成功标准；可验证、可维护和尊重安全边界更重要。

## Completion Evidence

完成时所有结论应能回到：

- 同一成功基线。
- Endpoint Inventory。
- 请求依赖图。
- 字段数据字典。
- 状态转换表。
- 差分/故障实验。
- 结构化日志和 Request ID。

如果这些产物互相矛盾，回到最早出现矛盾的阶段重新分析，不用补丁式猜测填空。
