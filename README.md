# claude-ship

`/ship` — a Claude Code skill that turns the current session's discussion
into shipped code in one autonomous run:

spec → plan → adversarial plan review → Workflow implementation →
multi-lens review + fixes → lint/format/tests → docs sync → commit → push

Explicit trigger only (`/ship`, "ship it"); it never fires from generic
"implement this" phrasing.

## Layout

- `SKILL.md` — the pipeline contract (hard rules, Stages −1…8, Never list)
- `references/resume.md` — Stage −1 resume dispatch detail, read only when
  a plan for the topic already exists
- `plans/` — this skill's own implementation plan + `TIMELINE.md` ledger
- `docs/superpowers/specs/` — design spec

## Install

Clone (or symlink) into Claude Code's personal skills directory:

```bash
git clone git@github.com:ZH-L1N/claude-ship.git ~/.claude/skills/ship
```

## Companions

- [adversarial-plan-review](https://github.com/Exowatt-Labs/adversarial-plan-review)
  — the Stage 3 reviewer loop; /ship drives it in orchestrated mode. Its
  scripts need `uv run --no-project --with openai python …`.
- [claude-time-sense](https://github.com/ZH-L1N/claude-time-sense) — the
  `plans/TIMELINE.md` ledger grammar Stage 7 stamps on every ship.
