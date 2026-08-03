---
name: ship
description: "Use ONLY when the user explicitly invokes /ship or says 'ship it' / 'ship this' for a task discussed in the session — Zehui's fully autonomous implement pipeline that ends in git push. NEVER trigger from generic phrasing like 'implement this', 'build X', or 'fix Y'; without an explicit /ship-style trigger, do not use this skill."
---

# /ship — discussion → pushed commit

Turn the current session's discussion into shipped code in one autonomous run:

spec → plan → adversarial plan review → Workflow implementation → multi-lens
review + fixes → lint/format/tests → docs sync → commit → push

**Announce at start:** "Shipping: <topic> — running the full pipeline on
branch <branch>."

## Hard rules

1. **Explicit trigger only.** If you cannot point to the user explicitly
   invoking /ship (or "ship it"/"ship this"), stop and do nothing.
2. **Fully autonomous.** The only permitted interactions:
   - one batched AskUserQuestion round IF the session has no real discussion
     of the task (cold invocation);
   - adversarial-plan-review's first-run transport prompt;
   - adversarial-plan-review's step-4a genuine-uncertainty consultations.
   Never ask to proceed, to confirm a stage, or to choose between options —
   make the call and log it.
3. **Current branch, as-is.** Never switch or create branches, never suggest
   worktrees (the user manages those), main is fine.
4. **Never commit on failure.** Lint + format + tests must be green first.
   Any hard failure at any stage → stop, no commit, report the stage, the
   cause, and where the artifacts live.
5. **Scope-changing deviations discovered mid-implementation → stop and
   report** rather than improvising scope creep.

## Stage −1 — Resume dispatch

Resolve the repo root first — `git rev-parse --show-toplevel` (not a repo →
STOP: "/ship needs a git repo") — and work from it everywhere. Derive
`<slug>` (kebab-case topic) and take the highest existing `<version>` of
`plans/<version>-<slug>.md`. That plan file already on disk → follow
**references/resume.md**: it decides fresh-run vs resume, the re-entry
stage, and the resume side-effects on later stages. No plan on disk →
fresh run, continue to Stage 0.

A dirty tree with no resume match is still a hard STOP at Stage 0: that's
the user's work, never /ship's to touch.

## Stage 0 — Preflight

```bash
git rev-parse --show-toplevel   # repo root from Stage −1; cd here — all stages run from the repo root
git status --porcelain          # non-empty (ignoring .scratch/, the review skill's own snapshots) AND no resume match → STOP: tell the user to commit/stash first (never stash for them)
git branch --show-current       # record <branch>; empty (detached HEAD) → STOP and report
uv run --no-project --with openai python "$HOME/.claude/skills/adversarial-plan-review/scripts/first_run.py" --check
```

- Transport check exit codes: **0** → ready. **2** → run
  adversarial-plan-review's first-run UX (its Setup step 2) now — an
  allowed ask; that branch may write `.env` and append to `.gitignore` in
  the repo root. **Anything else**, or `uv` missing → STOP and report the
  output verbatim; never proceed on an unknown transport state.
- Use the same interpreter form for every inner-skill python snippet in the
  whole pipeline — `uv run --no-project --with openai python …` — so
  Stage 3 actually uses the interpreter this check validated.
- Detect the repo's verification commands now (repo CLAUDE.md first, then
  package.json scripts / Makefile / pyproject / justfile). None found →
  note "no test infra"; Stage 6 runs whatever subset exists.

## Stage 1 — Size call (never ask)

**Small** iff ALL of: expected change ≤ ~3 files; no new subsystem or
dependency; no schema or public-API surface change. Otherwise **standard**.

- small → rounds ceiling **3**, no separate spec file (plan header carries
  Goal / Why / Success criteria)
- standard → rounds ceiling **5**, separate spec file

Log one line: `Size call: <small|standard> — <reason> → ceiling <N>`.

## Stage 2 — Spec + plan

Source material = the session's discussion. Cold invocation (no real prior
discussion) → ONE batched AskUserQuestion (purpose, constraints, success
criteria), then fully autonomous.

1. Derive `<slug>` (kebab-case topic). `<version>` starts at `v1`; if
   `plans/<version>-<slug>.md` exists, increment (a resume never bumps the
   version).
2. **Standard tasks:** draft the spec now but write it to the scratchpad,
   NOT the repo — adversarial-plan-review's preflight requires a clean tree
   outside `plans/`. It lands in the repo at Stage 3 exit. Content:
   what/why, constraints, success criteria, out-of-scope.
3. Write `plans/<version>-<slug>.md` per superpowers:writing-plans
   conventions: header (Goal, Architecture, size call), bite-sized tasks
   with exact files, interfaces, and verification steps. Small tasks: fold
   Goal / Why / Success criteria into this header.

## Stage 3 — Adversarial plan review (orchestrated mode)

Invoke the `adversarial-plan-review` skill in **orchestrated mode**: supply
`<slug>`, `<version>`, and the ceiling; skip its interactive Setup step 3;
auto-resume when prior sidecars exist. Prefix every `evaluate_exit` call
with `ADVERSARIAL_MAX_ROUNDS=<ceiling>`.

Its step-4a genuine-uncertainty consultations stay live — the one
mid-pipeline pause /ship permits. Everything else auto-resolves per its
"Orchestrated mode exception": exit-time soft-blocks → open HIGH findings
are a pipeline failure, open medium/low auto-defer and proceed; a step-7a
plan-bloat trigger → "Switch to consistency-only mode", logged, never a
stall.

| Review exit | /ship action |
|---|---|
| approved / resolved / resolved-with-deferrals / planner-locked (no open highs) | proceed |
| ceiling-hit, cost-capped, or planner-locked with open HIGH findings | STOP — report the open highs; do not implement |

(The inner skill prints these as `reason.value` with underscores —
`ceiling_hit`, `cost_capped`, `planner_locked`, `resolved_with_deferrals`.)

After a proceeding exit (standard tasks): move the spec draft from the
scratchpad to `docs/superpowers/specs/YYYY-MM-DD-<topic>.md`, updating it
if review rounds changed the design.

## Stage 4 — Implement (Workflow tool)

This skill's instruction to call Workflow is the explicit opt-in.

1. Read the final plan; map its tasks into dependency chains.
2. Build a Workflow script:
   - Coupled/sequential tasks (the common case): ONE implementation agent
     runs the whole chain in order — coherence beats parallelism. Fan out
     only for genuinely independent tasks touching disjoint files; if
     parallel agents would touch the same file, serialize instead.
   - **Models:** Fable for large/complex/critical tasks; Opus for most
     implementation; Sonnet for easy mechanical fan-out. Match effort
     tiers the same way.
   - Each agent prompt includes: the plan task text verbatim, the spec
     summary, the absolute repo root ("all work happens under <root>"),
     "follow the repo's CLAUDE.md", "use TDD where the repo has test
     infra", "never run git write commands (add/commit/stash/checkout/
     restore) — the orchestrator owns git", and a required structured
     return `{files_changed, tests_added, deviations, notes}` via schema.
3. Any agent reporting a scope-changing deviation → STOP (hard rule 5).
4. Exit check before Stage 5: every agent returned, and `git diff` +
   untracked files is non-empty and touches the files the plan named. An
   errored agent, an empty diff, or an untouched plan task → hard failure,
   STOP — never let a no-op sail through to a "shipped" commit.

## Stage 5 — Multi-lens review + fix loop

Shared budget with Stage 6: **3 fix rounds total** (one round = one
confirmed-findings → fix agents → re-review pass). Stage 5 may spend at
most 2 — the third belongs to Stage 6.

Review the working diff (`git diff` + untracked files) via Workflow:

1. Parallel lenses, each returning structured findings: correctness,
   security, test-coverage, simplification/altitude, and
   **plan-conformance** (does the diff implement the plan?).
2. Every finding is adversarially verified by an independent skeptic
   prompted to refute it; only confirmed findings survive.
3. Confirmed findings → fix agents (Opus; Fable for the gnarly ones) →
   re-review only the changed areas.
4. Repeat until zero confirmed findings or Stage 5's 2-round share is
   spent. Unfixed leftovers go in the final report — never silently
   dropped. Leftover confirmed HIGH-impact correctness/security findings →
   treat as failure: STOP, no commit.

After each round, append one line to /ship's own log,
`plans/fixs/<version>-<slug>-ship-review.md` (create if missing — NEVER
the adversarial-review fixes md, which its `regenerate_fixes_md` rewrites
wholesale): `ship-review round <n>: <confirmed> confirmed / <fixed> fixed /
<left> left, per lens: <counts>`. On exit append a final marker:
`review-loop: complete` (zero confirmed findings) or `review-loop:
budget-spent (<n> rounds, <left> non-HIGH deferred)`. Resume dispatch and
Stage 8 read this log — without it a later session cannot prove the review
happened.

## Stage 6 — Verification gate

Re-detect the repo's lint, format, and test commands now — the plan may
have added the repo's first test infra, so Stage 0's detection is a floor,
not a cap — then run them. Red → fix within the remaining shared budget;
still red → STOP, no commit, report the failing output verbatim.

Fixes must make the code green, never the gate blind: never weaken or
delete tests, skip assertions, or suppress lint rules to pass. Changing a
test expectation is legitimate only when the plan itself changed that
behavior. Stage 6 fix diffs get a correctness spot-review before commit
(within the shared budget) — they are the only code that would otherwise
ship unreviewed.

## Stage 7 — Docs sync + commit + push

1. **Docs sync**: re-read the final diff and update any repo markdown it
   invalidates or extends — `README.md`, the repo's `CLAUDE.md` (commands,
   architecture, conventions tables), and affected pages under `docs/`.
   Touch only statements the diff actually changes — no rewrites, no new
   doc files unless the plan called for one. If the repo's format/lint
   covers markdown, re-run it on these edits — Stage 6's gate closed
   before they existed. Nothing affected → skip and note "docs sync: none
   affected" in the Stage 8 report.
2. **Time-sense stamp** (before `git add`): run `date +"%Y-%m-%d %H:%M"`
   and splice its real output — never hand-write or guess a time; the
   time-sense hook silently skips malformed or invented stamps. Append to
   `plans/TIMELINE.md`: `- <timestamp> | <version> | shipped | <topic>`,
   bootstrapping the file from
   `~/Documents/GitHub/claude-time-sense/templates/TIMELINE.md` if
   missing; if THIS topic already has a `| <version> | shipped |` line
   from a prior attempt, update it in place instead of appending. Date the
   plan's status line: `**Status: implemented <timestamp>.**`. Shipping
   with deferrals or unfixed findings → mark each in the code or plan doc
   with `TEMP(<YYYY-MM-DD from the same date output>, until:
   <target-version/condition>): <reason>` (real digits — a placeholder
   never matches the scanner) and append `| temp: <note>` as the **last**
   field of the ledger line. All of it is part of the commit.
3. `git add` exactly what the pipeline produced: code, tests, `plans/…`,
   `plans/fixs/…`, the spec file, and any docs-sync edits.
4. One conventional commit (follow the repo's message convention; standard
   Claude co-author footer). A pre-commit hook failure is a red gate:
   treat it like Stage 6 (fix within the remaining budget or STOP) — never
   `--no-verify`. A pipeline artifact path that turns out gitignored →
   report it, never `git add -f`.
5. `git push`. Rejected because the remote moved → `git pull --rebase`
   once, push once more. Rebase reports conflicts → `git rebase --abort`
   (commit intact, tree clean), leave the commit local and report. No
   remote or no upstream → leave the commit local and report; never create
   remotes or set upstreams unasked. Still failing → leave the commit
   local and report.

## Stage 8 — Final report

- What shipped + commit hash (+ pushed / local-only)
- Size call + ceiling used
- Adversarial review: rounds, exit status, accepted/rejected/deferred, cost
- Review loop: findings confirmed/fixed/deferred per lens
- Verification: commands run + results
- Docs sync: files updated, or "none affected"
- Artifact paths: spec, plan, fixes md, ship-review log
- Deferred items (review deferrals + unfixed low-priority findings)

## Never

- Never open a PR, create a branch, or touch worktrees.
- Never auto-trigger from conversation that lacks an explicit /ship.
- Never bypass a red verification gate.
- Never force-push, `git add -f`, or `git commit --no-verify`.
- Never ask the user to confirm a stage transition.
