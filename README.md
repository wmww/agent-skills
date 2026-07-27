# Tess's agent skills
Grab bag of my personal skills for coding agents

- [`gui-testing`](gui-testing/) — run a GUI program in a throwaway headless Wayland
  session and drive it with screenshots, clicks, typing and drags
- [`timegap`](timegap/) — timestamp a program's output with
  [`tg`](https://codeberg.org/sphi/timegap) to find which step is slow or where it stalls

## Install

Claude Code:

```sh
claude plugin marketplace add wmww/agent-skills
claude plugin install <skill>@wmww-agent-skills
```

Codex:

```sh
python3 ~/.codex/skills/.system/skill-installer/scripts/install-skill-from-github.py \
  --url https://github.com/wmww/agent-skills/tree/main/<skill>
chmod +x ~/.codex/skills/gui-testing/guibox   # gui-testing only: the installer drops the exec bit
```
