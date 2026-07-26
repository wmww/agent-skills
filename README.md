# Tess's agent skills
Grab bag of my personal skills for coding agents

- [`gui-testing`](gui-testing/) — run a GUI program in a throwaway headless Wayland
  session and drive it with screenshots, clicks, typing and drags

## Install

### Claude Code

```sh
claude plugin marketplace add wmww/agent-skills
claude plugin install gui-testing@wmww-agent-skills
```

The same works as `/plugin marketplace add …` and `/plugin install …` inside a
session. Later, `claude plugin update gui-testing` pulls new commits.

Or skip the plugin system and symlink the skill into your personal skills dir:

```sh
git clone git@github.com:wmww/agent-skills ~/agent-skills
mkdir -p ~/.claude/skills
ln -s ~/agent-skills/gui-testing ~/.claude/skills/gui-testing
```

### Codex

Codex reads the same `SKILL.md`, from `~/.codex/skills/` instead. Its built-in
`skill-installer` skill can fetch one straight from GitHub — just ask Codex to
install `https://github.com/wmww/agent-skills/tree/main/gui-testing`, or run it
yourself:

```sh
python3 ~/.codex/skills/.system/skill-installer/scripts/install-skill-from-github.py \
  --url https://github.com/wmww/agent-skills/tree/main/gui-testing
chmod +x ~/.codex/skills/gui-testing/guibox   # the installer drops the exec bit
```

Restart Codex afterwards to pick the skill up. Cloning and symlinking into
`~/.codex/skills/` works too, and keeps the exec bit.
