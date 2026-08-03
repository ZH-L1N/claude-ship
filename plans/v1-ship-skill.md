# /ship Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the `/ship` personal skill — one explicit command that runs Zehui's full implement pipeline (spec → plan → adversarial review → Workflow implementation → multi-lens review → verify → commit → push) autonomously — plus a small orchestrated-mode edit to the existing adversarial-plan-review skill.

**Architecture:** A thin orchestrator SKILL.md at `~/.claude/skills/ship/` with no scripts; all durable state lives in the artifacts the pipeline already produces (spec, plan, fixes md + sidecars, git). One companion edit teaches adversarial-plan-review to accept orchestrated invocations.

**Tech Stack:** Markdown skill files; the existing adversarial-plan-review scripts (env-configured); Claude Code Workflow tool.

## Global Constraints

- Spec of record: `~/.claude/skills/ship/docs/superpowers/specs/2026-08-02-ship-design.md`
- `~/.claude/skills` is **not a git repo** → all commit steps in this plan are omitted (noted per task).
- Skill must be gated to explicit invocation only (`/ship`, "ship it", "ship this") — it ends in `git push`.
- Rounds ceiling: 3 (small) / 5 (standard), via `ADVERSARIAL_MAX_ROUNDS`; never ask the user.
- Never recommend subagent-driven-development anywhere in produced files (user preference).
- Model policy in produced SKILL.md: Fable = large/complex/critical, Opus = most work, Sonnet = easy parallel fan-out.

---

### Task 1: Create the /ship SKILL.md

**Files:**
- Create: `~/.claude/skills/ship/SKILL.md`

**Interfaces:**
- Produces: the `/ship` skill; references `adversarial-plan-review` orchestrated mode (defined in Task 2) and env var `ADVERSARIAL_MAX_ROUNDS`.

- [ ] **Step 1: Write the file with exactly this content**

````markdown
---
name: ship
description: "Zehui's end-to-end implement pipeline: capture spec+plan from the session's discussion, adversarial plan review, Workflow implementation (Fable/Opus), multi-lens code review with fixes, verify, commit, push — fully autonomous on the current branch. ONLY invoke on an explicit trigger: the user typed /ship or said 'ship it' / 'ship this'. NEVER trigger from generic phrasing like 'implement this', 'build X', or 'fix Y' — this pipeline ends in git push."
---

# /ship — discussion → pushed commit

Turn the current session's discussion into shipped code in one autonomous run:

spec → plan → adversarial plan review → Workflow implementation → multi-lens
review + fixes → lint/format/tests → commit → push

**Announce at start:** "Shipping: <topic> — running the full pipeline on branch <branch>."

## Hard rules

1. **Explicit trigger only.** If you cannot point to the user explicitly invoking
   /ship (or "ship it"/"ship this"), stop and do nothing.
2. **Fully autonomous.** The only permitted interactions:
   - one batched AskUserQuestion round IF the session has no real discussion of
     the task (cold invocation);
   - adversarial-plan-review's first-run transport prompt;
   - adversarial-plan-review's step-4a genuine-uncertainty consultations.
   Never ask to proceed, to confirm a stage, or to choose between options —
   make the call and log it.
3. **Current branch, as-is.** Never switch or create branches, never suggest
   worktrees (the user manages those), main is fine.
4. **Never commit on failure.** Lint + format + tests must be green first. Any
   hard failure at any stage → stop, no commit, report (see Failure & resume).
5. **Scope-changing deviations discovered mid-implementation → stop and
   report** rather than improvising scope creep.

## Stage 0 — Preflight

```bash
git rev-parse --show-toplevel   # not a repo → STOP: "/ship needs a git repo"
git status --porcelain          # non-empty → STOP: tell the user to commit/stash first (never stash for them)
git branch --show-current       # record <branch>; empty (detached HEAD) → STOP and report
python "$HOME/.claude/skills/adversarial-plan-review/scripts/first_run.py" --check
```

- Transport check exits 2 → run adversarial-plan-review's first-run UX (its
  Setup step 2) now. This is an allowed ask.
- Detect the repo's verification commands now (repo CLAUDE.md first, then
  package.json scripts / Makefile / pyproject / justfile). None found → note
  "no test infra" and Stage 6 runs whatever subset exists.

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
   `plans/<version>-<slug>.md` exists, increment — unless resuming (see
   Failure & resume; never bump the version for a resume).
2. **Standard tasks:** draft the spec now but write it to the scratchpad, NOT
   the repo — adversarial-plan-review's preflight requires a clean tree
   outside `plans/`. It lands in the repo at Stage 3 exit. Content: what/why,
   constraints, success criteria, out-of-scope.
3. Write `plans/<version>-<slug>.md` per superpowers:writing-plans
   conventions: header (Goal, Architecture, size call), bite-sized tasks with
   exact files, interfaces, and verification steps. Small tasks: fold
   Goal / Why / Success criteria into this header.

## Stage 3 — Adversarial plan review (orchestrated mode)

Invoke the `adversarial-plan-review` skill in **orchestrated mode**: supply
`<slug>`, `<version>`, and the ceiling; skip its interactive Setup step 3;
auto-resume when prior sidecars exist. Prefix every `evaluate_exit` call with
`ADVERSARIAL_MAX_ROUNDS=<ceiling>`.

Its step-4a genuine-uncertainty consultations stay live — that is the one
mid-pipeline pause /ship permits. Exit-time soft-blocks are auto-resolved per
that skill's "Orchestrated mode" notes: open HIGH findings → pipeline failure;
open medium/low → auto-defer and proceed.

| Review exit | /ship action |
|---|---|
| approved / resolved / resolved-with-deferrals / planner-locked (no open highs) | proceed |
| ceiling-hit or cost-capped with open HIGH findings | STOP — report the open highs; do not implement |

After exit (standard tasks): move the spec draft from scratchpad to
`docs/superpowers/specs/YYYY-MM-DD-<topic>.md`, updating it if review rounds
changed the design.

## Stage 4 — Implement (Workflow tool)

This skill's instruction to call Workflow is the explicit opt-in.

1. Read the final plan; map its tasks into dependency chains.
2. Build a Workflow script:
   - Coupled/sequential tasks (the common case): ONE implementation agent runs
     the whole chain in order — coherence beats parallelism. Fan out only for
     genuinely independent tasks touching disjoint files; if parallel agents
     would touch the same file, serialize instead.
   - **Models:** Fable for large/complex/critical tasks; Opus for most
     implementation; Sonnet for easy mechanical fan-out. Match effort tiers
     the same way.
   - Each agent prompt includes: the plan task text verbatim, the spec
     summary, "follow the repo's CLAUDE.md", "use TDD where the repo has test
     infra", and a required structured return
     `{files_changed, tests_added, deviations, notes}` via schema.
3. Any agent reporting a scope-changing deviation → STOP (hard rule 5).

## Stage 5 — Multi-lens review + fix loop

Shared budget with Stage 6: **3 fix rounds total.**

Review the working diff (`git diff` + untracked files) via Workflow:

1. Parallel lenses, each returning structured findings: correctness, security,
   test-coverage, simplification/altitude, and **plan-conformance** (does the
   diff implement the plan?).
2. Every finding is adversarially verified by an independent skeptic prompted
   to refute it; only confirmed findings survive.
3. Confirmed findings → fix agents (Opus; Fable for the gnarly ones) →
   re-review only the changed areas.
4. Repeat until zero confirmed findings or the budget is spent. Unfixed
   leftovers go in the final report — never silently dropped. Leftover
   confirmed HIGH-impact correctness/security findings → treat as failure:
   STOP, no commit.

## Stage 6 — Verification gate

Run the repo's lint, format, and test commands (from Stage 0 detection).
Red → fix within the remaining shared budget; still red → STOP, no commit,
report the failing output verbatim.

## Stage 7 — Commit + push

1. `git add` exactly what the pipeline produced: code, tests, `plans/…`,
   `plans/fixs/…`, the spec file.
2. One conventional commit (follow the repo's message convention; standard
   Claude co-author footer).
3. `git push`. Rejected because the remote moved → `git pull --rebase` once,
   push once more. Still failing → leave the commit local and report.

## Stage 8 — Final report

- What shipped + commit hash (+ pushed / local-only)
- Size call + ceiling used
- Adversarial review: rounds, exit status, accepted/rejected/deferred, cost
- Review loop: findings confirmed/fixed/deferred per lens
- Verification: commands run + results
- Artifact paths: spec, plan, fixes md
- Deferred items (review deferrals + unfixed low-priority findings)

## Failure & resume

Hard failure at any stage → stop, never commit, report the stage, the cause,
and where the artifacts live.

Re-invoking /ship for the same topic resumes from durable state — check in
this order and enter at the first match:

1. Commit exists but unpushed → Stage 7 (push only).
2. Working diff exists and the review loop is incomplete → Stage 5.
3. `plans/<version>-<slug>.md` exists with sidecars in `plans/fixs/` →
   Stage 3 (auto-resume mid-review).
4. Plan exists, no sidecars → Stage 3 from round 1.
5. Nothing on disk → full pipeline from Stage 0.

## Never

- Never open a PR, create a branch, or touch worktrees.
- Never auto-trigger from conversation that lacks an explicit /ship.
- Never bypass a red verification gate.
- Never ask the user to confirm a stage transition.
````

- [ ] **Step 2: Verify the file parses and is gated**

Run:

```bash
python3 -c "
import yaml, pathlib
text = pathlib.Path('$HOME/.claude/skills/ship/SKILL.md').read_text()
fm = text.split('---')[1]
meta = yaml.safe_load(fm)
assert meta['name'] == 'ship', meta
assert 'ONLY invoke on an explicit trigger' in meta['description']
print('frontmatter OK:', meta['name'])
"
```

Expected: `frontmatter OK: ship`

(No commit — not a git repo.)

---

### Task 2: Orchestrated-mode edit to adversarial-plan-review

**Files:**
- Modify: `~/.claude/skills/adversarial-plan-review/SKILL.md` (Setup step 3, ~line 108; soft-block flow in Loop step 7, ~line 426)

**Interfaces:**
- Consumes: `/ship` supplies slug, version, `ADVERSARIAL_MAX_ROUNDS` (Task 1).
- Produces: "orchestrated mode" behavior that Task 1's Stage 3 references.

- [ ] **Step 1: Edit Setup step 3 — add the orchestrated-mode exception**

Old string (exact):

```markdown
3. **Interactively ask the user** for the plan **slug** and **version** (e.g., `optical-lcoe`, `v0.0.5`). Always ask — do not guess from context, do not accept args. Deliberate paths prevent accidental overwrites.
```

New string:

```markdown
3. **Interactively ask the user** for the plan **slug** and **version** (e.g., `optical-lcoe`, `v0.0.5`). Always ask — do not guess from context, do not accept args. Deliberate paths prevent accidental overwrites.

   **Orchestrated mode exception:** when this skill is invoked by the `/ship` orchestrator with an explicit slug, version, and rounds ceiling (`ADVERSARIAL_MAX_ROUNDS`), skip this interactive ask and use the supplied values verbatim. Resume detection (step 6) also auto-selects **Resume** when prior sidecars exist instead of asking. Everything else — scope boundary, git prohibition, step-4a uncertainty consultations — is unchanged.
```

- [ ] **Step 2: Edit the step-7 soft-block flow — add orchestrated-mode auto-resolution**

Old string (exact):

```markdown
If `decision.needs_soft_block` is True, run the §5.4.1 soft-block flow:
```

New string:

```markdown
If `decision.needs_soft_block` is True, run the §5.4.1 soft-block flow — **unless in orchestrated mode** (invoked by `/ship`), in which case do not ask: any open HIGH finding → exit with the underlying reason (`ceiling_hit` / `cost_capped` / `planner_locked`) and report the open highs to the orchestrator (it stops its pipeline); otherwise auto-populate `Deferral(item_id, severity, reason="auto-deferred by /ship at exit", target_version="backlog")` for every open medium/low item, stash them on `state.deferrals_at_exit`, re-run step 6 persistence, and promote via `escalate_to_resolved_with_deferrals(decision, deferrals)`. Interactive invocations proceed with the flow below:
```

- [ ] **Step 3: Verify both edits landed**

Run:

```bash
grep -c "Orchestrated mode exception\|unless in orchestrated mode" "$HOME/.claude/skills/adversarial-plan-review/SKILL.md"
```

Expected: `2`

(No commit — not a git repo.)

---

### Task 3: Cross-file consistency validation

**Files:**
- Read-only check of: `~/.claude/skills/ship/SKILL.md`, `~/.claude/skills/adversarial-plan-review/SKILL.md`, the spec.

**Interfaces:**
- Consumes: Tasks 1–2 outputs.

- [ ] **Step 1: Verify referenced paths and names exist**

```bash
test -f "$HOME/.claude/skills/adversarial-plan-review/scripts/first_run.py" && echo first_run.py OK
grep -q "ADVERSARIAL_MAX_ROUNDS" "$HOME/.claude/skills/ship/SKILL.md" && echo env var OK
grep -q "ADVERSARIAL_MAX_ROUNDS" "$HOME/.claude/skills/adversarial-plan-review/SKILL.md" && echo env var OK in inner skill
```

Expected: all three OK lines.

- [ ] **Step 2: Cold-read the new SKILL.md against the spec**

Fresh-eyes checklist (fix inline if any fail):
- Every spec pipeline stage 0–8 appears with matching behavior (ceilings, exit-status table, 3-round shared budget, push-retry rule).
- Exit-status names match the inner skill's End-report vocabulary (approved / resolved / resolved-with-deferrals / planner-locked / ceiling-hit / cost-capped).
- No placeholder text (TBD/TODO), no mention of subagent-driven-development.
- Description contains no generic trigger words that could auto-fire.

---

### Task 4: Dogfood smoke test

**Files:**
- Create: scratch repo under the session scratchpad (`<scratchpad>/ship-dogfood/repo` + `<scratchpad>/ship-dogfood/origin.git`)

**Interfaces:**
- Consumes: the installed /ship skill (Tasks 1–3 complete).

- [ ] **Step 1: Build the scratch repo with a local bare origin**

```bash
S=<scratchpad>/ship-dogfood
mkdir -p $S && cd $S
git init --bare origin.git
git init repo && cd repo
printf 'def hello():\n    return "hello"\n' > hello.py
printf 'from hello import hello\n\ndef test_hello():\n    assert hello() == "hello"\n' > test_hello.py
printf '# Scratch\nRun tests: `python3 -m pytest -q`\n' > README.md
git add -A && git commit -m "init" && git remote add origin ../origin.git && git push -u origin main
```

Expected: clean push to the local bare origin.

- [ ] **Step 2: Run /ship on a tiny discussed task**

In-session, state the task ("add a `slugify(text)` function: lowercase, spaces→hyphens, strip non-alphanumerics — with tests"), then invoke `/ship`. Observe:
- Size call = small, ceiling 3, no separate spec file
- Plan written to `repo/plans/v1-slugify.md`, adversarial review runs without asking for slug/version
- Implementation + review loop + `pytest` green
- Commit lands and pushes to `origin.git`
- Final report contains all Stage 8 sections

Expected: pipeline completes; `git -C $S/origin.git log --oneline` shows the new commit.

- [ ] **Step 3: Resume smoke test**

In a second scratch run, interrupt after the plan file is written (before review completes), then re-invoke `/ship`: it must resume at Stage 3 with the SAME version (no v2 bump, no re-draft).

Note (conscious cap): the no-commit-on-red path is verified by inspection of the Stage 5/6 STOP rules rather than a rigged live run; a full failure-injection dogfood is a follow-up if wanted.

---

### Task 5: Memory entry

**Files:**
- Create: `/Users/zehui/.claude/projects/-Users-zehui-Documents-GitHub/memory/ship-skill.md`
- Modify: `/Users/zehui/.claude/projects/-Users-zehui-Documents-GitHub/memory/MEMORY.md`

- [ ] **Step 1: Write the memory file**

```markdown
---
name: ship-skill
description: /ship personal skill — full autonomous implement pipeline; explicit trigger only
metadata:
  type: project
---

`/ship` (global, `~/.claude/skills/ship/`) runs Zehui's end-to-end implement
pipeline: spec+plan capture → [[adversarial-plan-review]] orchestrated mode
(ceiling 5 std / 3 small via ADVERSARIAL_MAX_ROUNDS) → Workflow implementation
(Fable/Opus) → multi-lens review + fixes (3-round budget) → verify → commit →
push on the current branch. Fully autonomous; explicit trigger only ("/ship",
"ship it"). Built 2026-08-02; spec + plan live in the skill folder.
```

- [ ] **Step 2: Add the MEMORY.md index line**

```markdown
- [/ship skill](ship-skill.md) — full autonomous implement pipeline; explicit trigger only; orchestrates adversarial-plan-review + Workflow
```

---

## Self-review notes

- Spec coverage: stages 0–8, gating, companion edit, resume, dogfood — all mapped to Tasks 1–4; memory capture added as Task 5 (not in spec; low-cost convention).
- Spec amendments discovered during planning (spec-write timing via scratchpad; orchestrated auto-resolution of exit-time soft-blocks) are reflected in Task 1 Stage 2/3 text and Task 2 Step 2, and back-ported to the spec doc.
- Placeholder scan: none.
- Consistency: `ADVERSARIAL_MAX_ROUNDS`, exit-status vocabulary, `plans/fixs/` path, and Deferral field names (`item_id, severity, reason, target_version`) match the inner skill's current text.
