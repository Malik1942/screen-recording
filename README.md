# screen-recording

**An agent skill for script-driven product capture.** Works with Claude Code, Codex and Cursor — a plain `SKILL.md` skill plus one Python script, nothing agent-specific inside.

Give your coding agent a shot list and it records clean product footage — a native app in the iOS Simulator, a website in a driven browser, or a Figma prototype — using a staging checklist that burns one-time dialogs off-camera, a per-take protocol that verifies every end frame, and an auto-zoom derivation step that turns *measured* interaction times into a camera plan and cursor cues. The one rule everything hangs on: **trim from measured times, never from planned ones** — tap-injection latency makes a 15-second plan into a 60-second take.

## What's inside

| Path | What |
|---|---|
| `SKILL.md` | Triggers, the capture loop, non-negotiables |
| `references/capture-protocol.md` | Surface selection, staging checklists (sim / web / Figma proto), take protocol, HiDPI resolution maths, failure modes |
| `scripts/autoplan.py` | Measured tap times → auto-zoom camera plan + cursor/ripple arguments |

## Requirements

- macOS. iOS Simulator capture needs Xcode (`DEVELOPER_DIR=/Applications/Xcode.app/Contents/Developer`); website/Figma capture needs `screencapture` (built in) and one-time Screen Recording permission for your terminal.
- `python3` for autoplan.
- An agent that reads `SKILL.md` skills — [Claude Code](https://claude.com/claude-code), [Codex](https://developers.openai.com/codex), or [Cursor](https://cursor.com).

## Install

**curl** — installs for every supported agent (one real copy, the rest symlinked):

```bash
curl -fsSL https://raw.githubusercontent.com/Malik1942/screen-recording/main/install.sh | bash
```

Narrow to specific agents or scope to one repo:

```bash
./install.sh --claude --cursor
```

```bash
./install.sh --project
```

**git:**

```bash
git clone https://github.com/Malik1942/screen-recording.git ~/.claude/skills/screen-recording
```

| Agent | Personal scope | Project scope |
|---|---|---|
| Claude Code | `~/.claude/skills/` | `<repo>/.claude/skills/` |
| Codex | `~/.agents/skills/` | `<repo>/.agents/skills/` |
| Cursor | `~/.cursor/skills/` or `~/.agents/skills/` | `<repo>/.cursor/skills/` or `<repo>/.agents/skills/` |

## How to prompt

```text
Record the three scenes in my shot list from the iOS Simulator — clean status bar, English locale.
```

```text
Capture my website's checkout flow at 2560x1440 for a product film.
```

```text
The take came out 0 bytes / shows a permission dialog / is in the wrong language — fix the setup and re-roll.
```

Expect the skill to demand a per-scene contract before rolling: start state, the interaction to show, the payoff to read, and the data that must exist. Scenes missing those come back for rework — that's the point.

## Companion: product-film

This skill produces inputs for **[product-film](https://github.com/Malik1942/product-film)** — the editing/compositing/scoring pipeline (measure with its `diffscan`, render zoom + cursor with its `composite2`, then stitch, score, and audit). Capture itself stands alone; install both for the full recordings-to-scored-film pipeline.

## License

MIT — see [LICENSE](LICENSE).

---

Built by [Malik Zhang](https://malikzhang.com).
