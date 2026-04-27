---
name: git-conventional-commit
description: "Generate Git commit messages in Chinese using emoji plus Conventional Commit format. Use when the user asks Codex to analyze `git diff`, write a commit message, split unrelated changes into separate commits, commit code with a team convention, or produce messages like `:sparkles: feat(scope): 描述`. Trigger on requests such as `生成 commit message`, `帮我提交代码`, `分析 git diff`, `按照约定式提交`, `写中文 commit`, or `use emoji conventional commits`."
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

## Output Contract

- Always emit the subject in the exact shape `<emoji> <type>(<scope>): <description>` or `<emoji> <type>: <description>`.
- Always use the emoji shortcodes from the reference file, such as `:sparkles:` instead of a Unicode emoji.
- Always keep description, body, and footer in Chinese.
- Always preserve technical nouns in English when translation would sound unnatural.
- Never explain why a type was chosen unless the user explicitly asks for explanation after the message has been produced.
- Never wrap the final message in code fences.
- Never add extra prose before or after the commit message block.

## Safety Checks

- Never merge unrelated changes into one message just to avoid multiple commits.
- Never echo filenames unless they are essential to the semantic meaning of the change.
- Never describe low-level code mechanics when a higher-level intent can be expressed clearly.
- Follow repository-specific commit conventions over this skill only when the user or repository rules explicitly require it.
