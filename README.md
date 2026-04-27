# Skills

Personal Codex skills repository.

## Layout

```text
skills/
  git-conventional-commit/
```

## Install a skill

After pushing this repository to GitHub, install a skill with:

```bash
python3 ~/.codex/skills/.system/skill-installer/scripts/install-skill-from-github.py \
  --repo <owner>/<repo> \
  --path skills/git-conventional-commit
```

Or install from a GitHub tree URL:

```bash
python3 ~/.codex/skills/.system/skill-installer/scripts/install-skill-from-github.py \
  --url https://github.com/<owner>/<repo>/tree/main/skills/git-conventional-commit
```

Restart Codex after installing new skills.
