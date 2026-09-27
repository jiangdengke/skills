# Skills

这是一个自用的 AI Agent Skills 仓库，用来保存常用工作流的提示词、约束和工具调用规范。每个 `skills/<name>/` 目录都是一个独立 skill，可以按需安装到 Pi、Codex、Cursor 等支持 Skills 的工具中。

## 当前 Skills

| Skill | 作用 | 适合场景 |
| --- | --- | --- |
| `git-conventional-commit` | 根据 Git diff 生成中文提交信息，格式为 emoji shortcode + Conventional Commit | 生成 commit message、拆分提交、提交代码、发版、打 tag |
| `grill-me` | 像评审/面试官一样持续追问方案设计，逐步压测决策、边界和风险 | 方案评审、架构设计、RFC/PRD 打磨、提前发现遗漏 |
| `smart-search-cli` | 通过本地 `smart-search` CLI 做联网搜索、网页抓取、文档检索和 Deep Research | 查最新信息、核验网页/事实、检索官方文档、做带来源的调研 |
| `web-identity-flow-analysis` | 在授权范围内分析现代 Web 注册、登录与身份流程，并设计可恢复的协议客户端 | HAR/DevTools 分析、Cookie/Token 追踪、OAuth/OIDC/PKCE、状态机、错误诊断与客户端工程 |

## 目录结构

```text
skills/
  git-conventional-commit/
    SKILL.md
    agents/
    references/
  grill-me/
    SKILL.md
  smart-search-cli/
    SKILL.md
    agents/
    references/
  web-identity-flow-analysis/
    SKILL.md
    references/
```

## 如何使用

### `git-conventional-commit`

用于把当前仓库的 Git 改动整理成团队约定的中文提交信息。

它会：

- 读取 `git status`、暂存区和未暂存区 diff。
- 判断改动类型，例如 `feat`、`fix`、`docs`、`chore`、`refactor`。
- 按 `<emoji> <type>(<scope>): <中文描述>` 输出提交信息。
- 在改动不属于同一意图时，建议拆成多个 commit。
- 在发版场景下生成中文 release commit、tag message 或 release notes。

示例用法：

```text
帮我分析当前 git diff 并生成中文 commit message
帮我按照约定式提交提交代码
帮我把当前改动拆成几个 commit message
帮我发布一个新版本并生成 release 文案
```

输出示例：

```text
:sparkles: feat(auth): 支持用户登录态自动续期
```

### `grill-me`

用于让 Agent 对一个计划或设计进行连续追问，直到双方对关键决策、依赖关系、风险和边界条件达成清晰共识。

它会：

- 一次只问一个问题，避免同时抛出过多分支。
- 沿着设计决策树逐步追问，例如目标、约束、替代方案、回滚方案、测试策略等。
- 每个问题都给出推荐答案，帮助你更快判断合理选择。
- 如果问题可以通过查看代码库回答，优先让 Agent 自己查代码，而不是直接问你。

示例用法：

```text
我准备重构登录模块，grill me
这是我的缓存设计方案，帮我 grill 一下
我想写一个技术方案，帮我像评审一样追问
帮我压力测试一下这个产品设计
```

### `smart-search-cli`

用于让 Agent 通过本地 `smart-search` 命令执行可复现的网页研究流程。

它会：

- 用 `smart-search search` 做快速联网搜索。
- 用 `smart-search fetch` 抓取 URL 正文作为证据。
- 用 Context7 / Exa / Zhipu 等能力检索官方文档、可信来源或中文时效信息。
- 用 `smart-search deep` / `research` 处理需要拆解、交叉验证和证据整理的深度调研任务。
- 要求关键结论尽量基于已抓取正文，而不是只依赖搜索摘要。

示例用法：

```text
帮我查一下今天 OpenAI Responses API 有什么新变化
帮我核验这个链接里的说法：https://example.com/source
深度搜索一下最近的比特币行情
帮我找 React useEffect cleanup 的官方文档
```

注意：`smart-search-cli` 是 skill，不是 MCP Server。它依赖本机已经安装并配置好的 `smart-search` CLI：

```bash
npm install -g @konbakuyomu/smart-search@latest
smart-search setup
smart-search doctor --format json
```

API Key 建议通过 `smart-search setup` 或 `smart-search config set KEY VALUE` 保存，不要提交到这个仓库。

### `web-identity-flow-analysis`

用于在自有、测试、沙箱或明确授权的系统中，把现代 Web 注册、登录和身份验证流程还原为可解释、可验证、可恢复的协议模型。

它会：

- 先执行授权、安全边界和敏感数据脱敏检查。
- 从成功基线和 HAR 出发，建立 Endpoint Inventory、请求依赖图、字段数据字典和状态机。
- 区分 Cookie/Session、CSRF、OAuth state、OIDC nonce、PKCE、Access/Refresh/ID Token、OTP 和 Magic Link。
- 用单变量差分测试与故障注入验证必要条件、错误分类、幂等、重试和恢复。
- 指导 Transport、Identity Client、State Machine、Orchestrator、Error Classifier 等客户端分层设计。
- 明确禁止绕过 CAPTCHA、风控、限流、MFA、访问控制，以及批量注册、凭证攻击或账号接管。

示例用法：

```text
帮我分析这个注册 HAR，画出请求依赖图和状态机
比较浏览器成功但 API 客户端 403 的差异
审查这个 OAuth/OIDC/PKCE 登录客户端的状态与重试设计
为自建身份系统设计 OTP、幂等、错误分类和故障注入测试
```

## 安装 Skills

### 使用 Codex 安装

推荐直接在 Codex 里按需安装：

```text
使用 $skill-installer 安装 https://github.com/jiangdengke/skills/tree/main/skills/git-conventional-commit
使用 $skill-installer 安装 https://github.com/jiangdengke/skills/tree/main/skills/grill-me
使用 $skill-installer 安装 https://github.com/jiangdengke/skills/tree/main/skills/smart-search-cli
使用 $skill-installer 安装 https://github.com/jiangdengke/skills/tree/main/skills/web-identity-flow-analysis
```

如果你已经在 Codex 里，也可以直接说：

```text
帮我用 $skill-installer 安装 https://github.com/jiangdengke/skills/tree/main/skills/git-conventional-commit
帮我用 $skill-installer 安装 https://github.com/jiangdengke/skills/tree/main/skills/grill-me
帮我用 $skill-installer 安装 https://github.com/jiangdengke/skills/tree/main/skills/smart-search-cli
帮我用 $skill-installer 安装 https://github.com/jiangdengke/skills/tree/main/skills/web-identity-flow-analysis
```

### 手动安装到 Codex

```bash
git clone https://github.com/jiangdengke/skills.git
mkdir -p ~/.codex/skills
cp -R skills/skills/git-conventional-commit ~/.codex/skills/
cp -R skills/skills/grill-me ~/.codex/skills/
cp -R skills/skills/smart-search-cli ~/.codex/skills/
cp -R skills/skills/web-identity-flow-analysis ~/.codex/skills/
```

### 手动安装到 Pi

```bash
git clone https://github.com/jiangdengke/skills.git
mkdir -p ~/.agents/skills
cp -R skills/skills/web-identity-flow-analysis ~/.agents/skills/
```

### 手动安装到 Cursor

```bash
git clone https://github.com/jiangdengke/skills.git
mkdir -p ~/.cursor/skills
cp -R skills/skills/git-conventional-commit ~/.cursor/skills/
cp -R skills/skills/grill-me ~/.cursor/skills/
cp -R skills/skills/smart-search-cli ~/.cursor/skills/
cp -R skills/skills/web-identity-flow-analysis ~/.cursor/skills/
```

安装或更新完成后，重启对应的 AI 工具，新的 skill 才会被自动发现。
