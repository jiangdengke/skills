# Skills

这是一个自用的 Codex skills 仓库。

## 目录结构

```text
skills/
  git-conventional-commit/
```

## 安装技能

推荐直接在 Codex 里安装：

```text
使用 $skill-installer 安装 https://github.com/jiangdengke/skills/tree/main/skills/git-conventional-commit
```

如果你已经在 Codex 里，也可以直接说：

```text
帮我用 $skill-installer 安装 https://github.com/jiangdengke/skills/tree/main/skills/git-conventional-commit
```

如果不通过 Codex，也可以手动安装：

```bash
git clone https://github.com/jiangdengke/skills.git
mkdir -p ~/.codex/skills
cp -R skills/skills/git-conventional-commit ~/.codex/skills/
```

安装完成后，重启 Codex，新的 skill 才会被自动发现。
