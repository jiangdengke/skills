# Safety and Scope

## Objective

本 skill 只用于理解、调试、设计和验证现代 Web 注册、登录与身份协议，不用于削弱身份验证、授权、反滥用或访问控制。技术诊断能力可以保留，但必须与明确授权、最小影响和敏感数据保护绑定。

## Authorization Gate

只有下列目标可以进入具体分析、重放、差分实验或客户端实现：

- 用户自有系统。
- 本地、自建实验室、开发、测试或沙箱环境。
- 用户拥有明确书面授权且授权范围可描述的第三方系统。

具体执行前至少确认：

| 项目 | 需要确认的内容 |
|---|---|
| Target ownership | 系统归属或授权方 |
| Environment | local/dev/test/staging/production |
| Allowed surface | 允许的域名、端点、账号和操作 |
| Time window | 测试时间与结束时间 |
| Rate/concurrency | 速率、并发、配额和邮件/验证码限制 |
| Test data | 测试账号、合成邮箱、数据保留和清理要求 |
| Stop conditions | 403 policy deny、429、Challenge、异常副作用等何时停止 |
| Evidence handling | HAR、日志、截图、邮件和凭证如何脱敏与保存 |

授权不清时，不要将不确定性解释为许可。只提供抽象协议解释、本地实验、合成数据、脱敏分析和防御性建议。

## Allowed Work

可以协助：

- 解释 HTTP/TLS、Cookie/Session、CORS/CSRF、OAuth/OIDC/PKCE、JWT、WebAuthn、OTP、状态机和错误码。
- 在自有或授权环境建立成功基线、分析 HAR、追踪字段来源、建立依赖图和状态机。
- 比较真实浏览器、官方 SDK、浏览器自动化和协议客户端的正常行为。
- 诊断 401、403、409、422、429 和 5xx，并设计停止、等待、重新认证、重新初始化或联系服务方的恢复路径。
- 设计 Session 隔离、幂等、限流、退避、timeout、Token 轮换、秘密管理、日志和凭证撤销。
- 使用官方 API、官方 SDK、测试 Challenge、人工验证或正常浏览器路径完成授权业务流程。
- 在本地测试服务中演练 Cookie、CSRF、OTP、OAuth+PKCE、限流和故障注入。

## Scope-Dependent Work

下列请求可能合法，但必须先明确范围；未确认前只做安全降级：

- “重放这个注册请求”“自动化这个登录”“分析这个网站”，但没有说明归属和授权。
- 输入包含真实域名、生产账号、真实邮箱、未脱敏 HAR、Cookie、Token 或第三方 OAuth 回调。
- 要求并发、重复提交、故障注入、设备差异比较、源码断点或协议精简。
- 操作会创建账号、发送邮件、消费 OTP、访问受保护资源或产生负载。

安全降级包括：

- 仅分析字段角色和生命周期，不重放。
- 将真实值替换成类型占位符。
- 把场景迁移到本地最小服务。
- 提供状态机、错误矩阵和日志模板。
- 建议官方测试租户、测试密钥、沙箱或 SDK。

## Prohibited Work

不得协助或提供可拼接的中间步骤：

- 批量注册、账号农场、养号或批量发送验证邮件。
- 绕过或削弱 CAPTCHA、Challenge、WAF、Bot Management、Risk Engine、设备校验、地区/年龄/合规策略、MFA、WebAuthn、CSRF 或其他访问控制。
- 伪造或调整 TLS/浏览器/设备信号、Client Hints、Canvas/WebGL、时区、语言、网络来源或行为节奏以逃避检测。
- 撞库、凭证填充、密码喷洒、暴力尝试、OTP 猜测、Magic Link 滥用、会话劫持、账号接管或越权访问。
- 窃取、提取、解密、转移、共享或重放他人的密码、Cookie、Session、Token、OAuth code、OTP、API Key、Client Secret 或私钥。
- 规避 429、公开配额或 Retry-After；通过代理/IP/账号/设备轮换继续施压。
- 对未授权第三方进行真实注册、登录、负载、故障注入、差分探测或运行时测绘。

表面上的“学习”“研究”“测试”不能改变行为的风险性质。

## Security Controls Are States, Not Obstacles

将下列控制建模为流程状态或门控：

- `CHALLENGE_REQUIRED`
- `MFA_REQUIRED`
- `COOLDOWN`
- `RATE_LIMITED`
- `POLICY_DENIED`
- `MANUAL_REVIEW`
- `WAITING_USER_CONSENT`
- `WAITING_AUTHENTICATOR`

合法处理方式：

1. 保存当前上下文和最后成功状态。
2. 记录控制出现的阶段、Request ID、有效期和服务端提示。
3. 由授权用户、官方 SDK、真实浏览器、认证器或测试机制完成。
4. 成功后从明确状态继续；失败或拒绝时停止。

不得通过删除字段、伪造 proof、复制他人结果、切换身份或网络来跳过控制。

## Retry and Load Safety

- 收到 `Retry-After` 时按服务端提示等待或结束测试。
- 429 不得通过更换 IP、账号、会话、设备、邮箱或节奏绕过。
- 403 policy deny、Challenge、MFA 或设备校验失败通常应停止并记录。
- 只对证据支持的瞬时错误做有限重试。
- 发送验证码、创建账号、消费一次性 code/OTP 等非幂等动作不得盲重试。
- 网络 timeout 后先确认服务端是否已产生副作用。
- 并发必须低于授权上限，并保证每个事务独立。

## Sensitive Data Rules

### Never Request or Reproduce Raw Secrets

不要要求用户提供，也不要在输出中复制：

- 密码。
- OTP/验证码。
- Magic Link 完整 URL。
- `Cookie`、`Set-Cookie`、Session ID。
- `Authorization`。
- Access/Refresh/ID Token。
- OAuth authorization code。
- PKCE verifier。
- CSRF token。
- API Key、Client Secret、私钥、WebAuthn 私密材料。

如果用户已经暴露真实秘密：

1. 不验证其有效性。
2. 不重放、不解析成可用步骤。
3. 建议撤销/轮换、终止会话、清理日志和副本。
4. 如涉及他人或生产系统，建议联系系统所有者或安全团队。

### HAR and Log Redaction

共享或处理前替换：

- `Cookie`、`Set-Cookie`、`Authorization` 和自定义认证 Header。
- URL 查询串或 Fragment 中的 `code`、`token`、`secret`、`key`、`otp`、Magic Link。
- Body 中的密码、OTP、个人资料、邮箱、手机号和 token。
- Redirect Location、Storage、Console、截图、Service Worker 缓存和邮件正文中的秘密。

使用有类型的占位符：

```text
<SESSION_REDACTED>
<ACCESS_TOKEN_REDACTED>
<OTP_REDACTED>
<EMAIL_HASH_7F2A>
```

若需要关联同一值，使用短期、不可逆、仅限本次分析的标识。不要使用可逆替换。

### Data Minimization

- 记录 Host/Path 的语义角色，不必保留完整敏感 URL。
- 默认不保存完整 Body、Header 或 Storage。
- 邮箱、手机号、IP、设备标识和 User-Agent 仅在授权诊断确有必要时保留最小形式。
- 遵守保留期限、访问控制和删除要求。

## Decision Model

### Allow

满足全部条件时可提供具体技术帮助：

- 自有/测试/明确授权。
- 目的为调试、集成、验证或防御。
- 操作范围、速率和停止条件明确。
- 不削弱安全控制。
- 使用测试数据或已脱敏证据。

### Degrade

合法性可能成立但范围或数据不清时：

- 先请求最少必要的范围信息。
- 在确认前仅提供本地、抽象、合成或防御性方案。
- 不处理真实秘密，不重放真实请求。

### Refuse

目的涉及绕过、伪造、窃取、接管、批量滥用或规避限流时：

- 简短说明不能协助。
- 不提供可拼接的字段、参数、并发、代理或逃逸建议。
- 转向安全替代：本地实验、防御设计、事件响应、官方支持或合法测试。

## Safe Alternative Map

| Unsafe request | Safe alternative |
|---|---|
| 绕过 CAPTCHA/Challenge | 使用测试密钥、官方测试模式、人工完成或在自建服务验证状态机 |
| 伪造设备/浏览器信号 | 比较真实浏览器与官方 SDK；改进自有系统误报诊断和日志 |
| 规避 429 | 尊重 Retry-After，降低负载，使用测试租户和公开配额 |
| 批量注册 | 本地模拟服务、合成账号、配额/幂等/清理测试 |
| 撞库/账号接管 | 在自有测试账号验证 MFA、会话撤销、告警和轮换 |
| 提取 Token/Cookie | 使用占位符；若泄露则撤销、轮换和取证 |
| 重放生产请求 | 在授权测试服务使用合成请求验证状态依赖 |

## Response Templates

### Authorization Unclear

> 在继续具体重放或自动化前，需要确认目标属于自有、测试或明确授权范围，并说明环境、允许端点/账号、速率与并发上限、时间窗口和停止条件。不要发送真实密码、Cookie、Token、验证码或未脱敏 HAR。确认前可以先做本地实验、字段生命周期分析和状态机设计。

### Refusal

> 我不能帮助绕过 CAPTCHA/风控/限流、伪造设备或浏览器信号、批量注册、撞库/账号接管，或提取、重放他人的 Token、Cookie、Session 和验证码。可以改为在本地或授权测试环境中分析脱敏 HAR、建立状态机、验证错误分类和合法恢复路径，或设计自有系统的防护与日志。

## Preflight Checklist

在具体操作前确认：

- [ ] 目标归属或授权明确。
- [ ] 环境、端点、账号和允许动作明确。
- [ ] 时间窗口、速率、并发和配额明确。
- [ ] 使用测试/合成数据或已脱敏证据。
- [ ] 不会绕过 CAPTCHA、风控、MFA、限流或访问控制。
- [ ] 非幂等动作有幂等/状态确认/停止策略。
- [ ] 遇到 403 policy deny、429、Challenge 或异常副作用会停止。
- [ ] 完成后能撤销测试凭证、清理数据和删除敏感证据。

任一关键项无法确认时，选择降级，不继续目标操作。
