---
name: git-conventional-commit
description: "Generate Git commit messages and release messages in Chinese using emoji plus Conventional Commit format. Use when the user asks Codex to analyze `git diff`, write a commit message, split unrelated changes into separate commits, commit code with a team convention, publish a release, bump versions, create tags, or produce messages like `:sparkles: feat(scope): 描述`. Trigger on requests such as `生成 commit message`, `帮我提交代码`, `分析 git diff`, `按照约定式提交`, `写中文 commit`, `发版`, `发布 release`, or `打 tag`."
---

# Git Conventional Commit

## Overview

Analyze the diff, extract the dominant change intent, and generate one or more commit messages in the exact format `<emoji> <type>(<scope>): <description>`.

Use Chinese for description, body, and footer. Preserve technical terms such as API, Hook, CLI, Database, and JWT in English when translation would reduce clarity.

## Workflow

1. Inspect repository state first.
   - Run `git status --short`.
   - Inspect staged changes with `git diff --cached --stat` and `git diff --cached` when they exist.
   - Inspect unstaged changes with `git diff --stat` and `git diff` when nothing is staged or when the user asks for all current edits.
   - Ignore unrelated workspace noise that the user did not ask to include.

2. Group changes by intent.
   - Keep one commit per coherent purpose.
   - Split unrelated features, bug fixes, refactors, docs updates, or config changes into separate commit messages.
   - If several files support the same user-visible intent, keep them in one commit.
   - If the diff is broad but still centers on one purpose, write one message around the dominant purpose instead of listing file edits.

3. Choose emoji, type, and optional scope.
   - Map the type to the required emoji using [references/conventional-commit-guide.md](./references/conventional-commit-guide.md).
   - Use a scope only when one module, package, or surface is clearly dominant, such as `auth`, `parser`, `ui`, or `api`.
   - Keep `type` and `scope` lowercase.
   - Omit the scope when it would be vague or misleading.

4. Write the description.
   - Use Chinese.
   - Use an imperative sentence fragment.
   - Keep it within 80 characters.
   - Do not end it with a Chinese or English period.
   - Summarize intent, not implementation details.

5. Add a body only when it materially improves understanding.
   - Explain why the change exists or what meaningful work was done.
   - Use list items prefixed with `- `.
   - Keep each body line within 150 characters.
   - Keep body text in Chinese, while leaving technical terms in English when appropriate.

6. Add a footer only when necessary.
   - Use a Chinese footer for breaking changes or issue references.
   - Prefer `破坏性变更:` for incompatible changes.
   - Prefer `关联:` or `修复:` for issue references when the repository does not require another footer style.

7. Produce the final output according to user intent.
   - If the user asks for a commit message or asks to analyze a diff, output only the commit message content.
   - If multiple commits are needed, separate them with exactly two blank lines.
   - Do not output explanations, introductions, Markdown code fences, or confirmation text together with the messages.
   - If the user explicitly asks to run `git commit`, use the generated message format directly when composing the command.

## Release Workflow

Use this workflow when the user asks to publish a release, bump a version, create a tag, or produce release notes.

1. Inspect release state first.
   - Run `git status --short` and avoid including unrelated untracked files such as build outputs or local documents unless the user explicitly asks.
   - Check existing tags with `git tag --sort=-v:refname --list 'v*'`.
   - Inspect release automation when present, such as GitHub Actions workflows that publish releases on tag pushes.

2. Confirm the release channel before choosing a version.
   - When the user asks to publish a release, create a release tag, bump a version for release, or says `发版` / `打 tag`, stop and ask whether this should be a stable release or a prerelease.
   - Ask with explicit options: stable release (`vX.Y.Z`) or prerelease (`vX.Y.Z-beta.1`, `vX.Y.Z-rc.1`, or a similar SemVer suffix).
   - Do not infer a stable release just because the user did not mention beta, RC, test, or prerelease.
   - Only skip this question when the user's current request already unambiguously specifies both the release channel and the target version/tag.

3. Choose the next version with Semantic Versioning after the release channel is confirmed.
   - Use MAJOR for breaking changes or incompatible data/API changes.
   - Use MINOR for user-facing features that remain backward compatible.
   - Use PATCH for bug fixes, dependency/build fixes, release-only version bumps, and small compatibility adjustments.
   - Stable release tags must use `vX.Y.Z`; pre-release tags must use `vX.Y.Z-rc.1`, `vX.Y.Z-beta.1`, or a similar SemVer suffix.

4. Keep app version files aligned.
   - For Flutter apps, keep `pubspec.yaml` in the shape `X.Y.Z+build`.
   - Increment the build number monotonically; for simple stable app releases, prefer `X.Y.Z+Z` only when that matches the repository's existing convention.
   - If the version file, latest tag, and requested release disagree, stop and ask the user which version to publish.

5. Write release commit messages in Chinese.
   - For release-only commits, prefer `:bookmark: chore(release): 发布 vX.Y.Z`.
   - Include a Chinese body when it helps explain what is included in the release.
   - Example:
     ```text
     :bookmark: chore(release): 发布 v1.0.17

     - 更新应用版本到 1.0.17+17
     - 同步本次发布所需的构建与依赖配置
     ```

6. Write tag and release messages in Chinese.
   - Prefer annotated tags: `git tag -a vX.Y.Z`.
   - Tag message should be Chinese, for example:
     ```text
     发布 v1.0.17

     - 更新应用版本到 1.0.17+17
     - 同步 Flutter 依赖锁定结果
     ```
   - GitHub Release title should be `vX.Y.Z`.
   - Manual GitHub Release bodies should be Chinese and use this structure when enough context exists:
     ```markdown
     ## 更新内容
     - ...

     ## 验证
     - ...

     ## 产物
     - ...
     ```
   - GitHub Release pages do not automatically display annotated tag messages. When release automation exists, make the workflow pass a notes body or `--notes-file` explicitly.
   - Prerelease tags such as `vX.Y.Z-alpha.1`, `vX.Y.Z-beta.1`, or `vX.Y.Z-rc.1` must create or edit GitHub Releases with the prerelease flag, for example `gh release create ... --prerelease` or `gh release edit ... --prerelease`.
   - Existing GitHub Releases must be edited to sync title, notes, and prerelease status before or while uploading assets. Do not only run `gh release upload` against an existing release.
   - Prefer copying the annotated tag message into the GitHub Release body, then append artifact information such as APK names under `## 产物`.
   - If the repository already generates release notes automatically, pushing the tag is enough unless the user asks for a manual release body.

## Output Contract

- Always emit the subject in the exact shape `<emoji> <type>(<scope>): <description>` or `<emoji> <type>: <description>`.
- Always use the emoji shortcodes from the reference file, such as `:sparkles:` instead of a Unicode emoji.
- Always keep description, body, and footer in Chinese.
- For release commits, prefer `:bookmark: chore(release): 发布 vX.Y.Z` and keep tag/release messages in Chinese.
- For prerelease tags, ensure the GitHub Release is marked as a prerelease and uses the Chinese release body instead of generic automation text.
- Always preserve technical nouns in English when translation would sound unnatural.
- Never explain why a type was chosen unless the user explicitly asks for explanation after the message has been produced.
- Never wrap the final message in code fences.
- Never add extra prose before or after the commit message block.

## Safety Checks

- Never merge unrelated changes into one message just to avoid multiple commits.
- Never echo filenames unless they are essential to the semantic meaning of the change.
- Never describe low-level code mechanics when a higher-level intent can be expressed clearly.
- Follow repository-specific commit conventions over this skill only when the user or repository rules explicitly require it.
