# Conventional Commit Guide

Use this reference when the type, emoji, scope, or message shape is ambiguous.

## Required subject format

- `<emoji> <type>: <description>`
- `<emoji> <type>(<scope>): <description>`

## Emoji and type mapping

- `:sparkles: feat`: 新功能
- `:bug: fix`: 修复缺陷
- `:building_construction: build`: 构建相关
- `:wrench: chore`: 日常任务或配置变更
- `:bookmark: chore(release)`: 版本发布或打 tag
- `:construction_worker: ci`: 持续集成配置
- `:memo: docs`: 文档更新
- `:high_voltage: perf`: 性能优化
- `:recycling_symbol: refactor`: 代码重构
- `:fast_reverse_button: revert`: 回退代码
- `:lipstick: style`: 代码格式或样式调整
- `:white_check_mark: test`: 测试相关
- `:globe_with_meridians: i18n`: 国际化

## Scope heuristics

- Use a scope when one module clearly owns the change, such as `auth`, `parser`, `ui`, `api`, `db`, or a package name.
- Omit the scope when the change spans several unrelated modules.
- Avoid vague scopes such as `misc`, `common`, or `core` unless the repository already uses them consistently.

## Description heuristics

- Write the description in Chinese.
- Use an imperative phrase such as `补充`, `修复`, `优化`, `重构`, `移除`, or `更新`.
- Keep it within 80 characters.
- Avoid repeating the type meaning inside the description unless it adds real clarity.
- Focus on change intent, not file-by-file implementation details.

## Body heuristics

- Add a body only when the subject alone cannot explain the intent.
- Use Chinese bullet lines starting with `- `.
- Explain why the change is needed or what meaningful work was included.
- Keep each body line within 150 characters.

## Footer heuristics

- Use `破坏性变更:` for incompatible changes.
- Use `关联:` for linked issues.
- Use `修复:` for issues closed by the change when the repository does not require English keywords.

## Examples

- `:sparkles: feat(auth): 增加刷新令牌轮换机制`
- `:bug: fix(api): 修复工作区名称为空时的响应异常`
- `:recycling_symbol: refactor(cli): 简化命令分发流程`
- `:memo: docs: 补充本地开发环境说明`
- `:white_check_mark: test(cache): 增加过期缓存淘汰测试`
- `:wrench: chore(deps): 更新 okhttp 到 5.1.0`
- `:bookmark: chore(release): 发布 v1.0.17`
- `:globe_with_meridians: i18n(ui): 补充登录页中文文案`

## Release heuristics

- 发正式版本时使用 SemVer：`vX.Y.Z`。
- 默认发 PATCH 版本，除非 diff 明确包含用户可见新功能或破坏性变更。
- 新功能使用 MINOR，破坏性变更使用 MAJOR。
- Beta 或 RC 版本使用 `vX.Y.Z-beta.1`、`vX.Y.Z-rc.1` 等后缀。
- Flutter 应用版本保持 `X.Y.Z+build`，build number 必须单调递增。
- 发布提交优先使用 `:bookmark: chore(release): 发布 vX.Y.Z`。
- tag message 和 GitHub Release body 尽量使用中文。

## Multiple commit example

```text
:sparkles: feat(auth): 增加刷新令牌轮换机制

- 减少长期令牌泄漏后的复用风险
- 补充令牌更新链路的过期校验


:white_check_mark: test(auth): 增加刷新令牌轮换测试

- 覆盖成功轮换与重复使用令牌的失败场景
```
