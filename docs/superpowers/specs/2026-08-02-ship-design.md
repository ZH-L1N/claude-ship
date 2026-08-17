# /ship — end-to-end implement-workflow shortcut (design)

- **Date:** 2026-08-02
- **Status:** approved (design reviewed inline by Zehui)
- **Owner:** Zehui
- **Location when built:** `~/.claude/skills/ship/SKILL.md` (personal skill, all machines via ~/.claude)

## Problem

After a task has been discussed in-session (details, questions, constraints), turning
that discussion into shipped code today requires manually driving many steps: write
the spec, write the plan, run `/adversarial-plan-review`, orchestrate implementation,
review the diff, fix findings, verify, commit, push. Each step needs prompting and
each hand-off is a place to forget conventions (rounds ceiling, folders, models).

`/ship` is one explicit command that runs the whole pipeline autonomously.

## Decisions (from design Q&A)

| Decision | Choice |
|---|---|
| Autonomy | Fully autonomous end-to-end. The only permitted mid-pipeline pause: adversarial-plan-review's genuine-uncertainty user consultation (rare, load-bearing). No pause before implementation, commit, or push. Setup-time asks are separate and allowed: one clarifying round if invoked with no prior discussion (Stage 2), and the reviewer transport first-run prompt (Stage 0). |
| Spec folder | `docs/superpowers/specs/YYYY-MM-DD-<topic>.md` in the target repo |
| Spec for small tasks | Skipped as a separate file — folded into the plan's header section |
| Name | `/ship` |
| Rounds ceiling | Auto-decided: 5 standard, 3 small. Never asked. |
| Branch | Whatever is checked out — main included. Never asks. User manages worktrees themselves. |
| Trigger | Explicit invocation only (`/ship` or "ship it/this"). Description gated so casual "implement this" never auto-fires it. |

## Architecture

A **thin orchestrator skill** — one SKILL.md, no scripts. All durable state lives in
files the pipeline already produces (spec, plan, fixes md + JSON sidecars, git), so
re-invoking `/ship` after an interruption resumes from what exists on disk.

Rejected alternatives: script-backed state machine (YAGNI — file artifacts already
persist state); chaining superpowers executing-plans/finishing-a-development-branch
(assume worktrees + human checkpoints, contradicting full autonomy).

## Pipeline

### Stage 0 — Preflight (stop early, stop loud)
- Must be a git repo; otherwise stop.
- Working tree must be clean (`git status --porcelain` empty). adversarial-plan-review
  refuses on changes outside `plans/` anyway; /ship checks first with a clearer message.
- Record current branch; never switch, never ask.
- Verify reviewer transport for adversarial-plan-review (OPENAI_API_KEY or Codex CLI).
  If unconfigured, the inner skill's first-run flow handles it (may ask once — allowed).

### Stage 1 — Size call (automatic, logged, never asked)
**Small** ≈ touches ≤2–3 files, no new subsystem/dependency, no schema or public API
surface change. Small → ceiling 3, no separate spec file. Standard → ceiling 5,
separate spec. The call and its one-line justification are logged in the final report.

### Stage 2 — Capture spec + plan
- Source material: the discussion already in this session. If invoked cold (no prior
  discussion), run ONE batched clarifying round (AskUserQuestion), then go autonomous.
- Standard task: draft the spec (what/why, constraints, success criteria,
  out-of-scope) in the scratchpad, NOT the repo — adversarial-plan-review's
  preflight requires a clean tree outside `plans/`. It lands at
  `docs/superpowers/specs/YYYY-MM-DD-<topic>.md` right after Stage 3 exits
  (updated if review rounds changed the design).
- Plan → `plans/<version>-<slug>.md` per superpowers:writing-plans conventions
  (bite-sized stages, exact files, verification steps). Small task: plan's header
  carries the spec content (Goal / Why / Success criteria).
- Slug derived from topic; version starts at `v1` and auto-increments if
  `plans/<version>-<slug>.md` already exists.

### Stage 3 — Adversarial plan review
- Invoke the `adversarial-plan-review` skill in **orchestrated mode** (companion edit,
  below): slug, version, and `ADVERSARIAL_MAX_ROUNDS` (3 or 5) supplied by /ship;
  interactive setup asks skipped; prior sidecars → auto-resume.
- The inner skill's genuine-uncertainty / open-question consultations remain live.
- Exit statuses `approved`, `resolved`, `resolved-with-deferrals`, `planner-locked`
  proceed to implementation. `ceiling-hit` with open HIGH findings → stop and report
  (do not implement a plan with unresolved high-severity holes); open medium/low →
  proceed, carry them into the final report as deferred.

### Stage 4 — Implement via Workflow tool
- Decompose the plan into tasks with explicit dependencies → Workflow script
  (`pipeline()` per dependency chain; `parallel()` only for genuinely independent tasks).
- Model policy: **Fable** for large/complex/critical tasks, **Opus**
  for most implementation work, **Sonnet** for highly parallel easy/mechanical work.
  Effort tiers matched likewise.
- Worktree isolation (`isolation: 'worktree'`) only when parallel agents would touch
  overlapping files.
- Implementing agents follow the target repo's CLAUDE.md; TDD where the repo has test
  infrastructure.

### Stage 5 — Multi-lens review + fix loop
- Workflow review pattern over the working diff: parallel lenses — correctness,
  security, test-coverage, simplification/altitude, plan-conformance — each producing
  structured findings; every finding adversarially verified (independent skeptic
  prompted to refute) before it counts.
- Confirmed findings → fix agents → re-review only the changed areas.
- Loop until zero confirmed findings or **3 fix rounds**; anything still open is
  listed in the final report, never silently dropped.

### Stage 6 — Verification gate
- Run the repo's lint, format, and tests (per global CLAUDE.md: never commit on
  failure). Failures get bounded fix attempts (within the 3-round budget above);
  still red → **stop, no commit**, report with the failing output.

### Stage 7 — Commit + push
- Conventional commit message per repo convention, standard Claude co-author footer.
- Commit on the current branch; push to origin. If push is rejected (remote moved),
  one `git pull --rebase` + retry; still failing → leave the commit local and report.

### Stage 8 — Final report
What shipped, commit hash, size call + ceiling used, adversarial-review stats
(rounds/status/cost), review-loop stats (findings confirmed/fixed/deferred),
verification results, artifact paths (spec, plan, fixes md).

## Failure handling

Any hard failure at any stage → stop the pipeline, never commit, report exactly where
it stopped and where the durable artifacts live. Re-invoking `/ship` for the same
topic resumes: existing plan → skip drafting; existing sidecars → resume review;
implemented-but-unreviewed diff → resume at Stage 5.

## Companion edit: adversarial-plan-review SKILL.md

Two additions, both scoped to invocations coming from `/ship`:

1. **Setup step 3:** skip the interactive slug/version ask; take the ceiling from
   `ADVERSARIAL_MAX_ROUNDS`; auto-choose "resume" when prior sidecars exist.
2. **Loop step 7 soft-blocks:** don't ask at exit. Open HIGH findings → exit with
   the underlying reason and report to the orchestrator (which stops the pipeline);
   open medium/low → auto-defer (`reason="auto-deferred by /ship at exit"`,
   `target_version="backlog"`) and promote to resolved-with-deferrals.

Everything else — scope boundary, git prohibition, step-4a uncertainty
consultations — unchanged. Manual invocations keep the current always-ask behavior.

## Gating

`/ship` ends in `git push`, so its description must restrict triggering to explicit
invocation ("/ship", "ship it", "ship this") — mirroring the existing
skill-install-gating convention. It must never fire from generic phrasing like
"implement this" or "build X".

## Out of scope (v1)

- PR creation / branch management / worktrees (user handles these; direct push only)
- Multi-repo or monorepo-subproject awareness beyond cwd
- Script-based state machine or resume sidecars of its own
- CI watching after push

## Testing

Dogfood: after building, run `/ship` on a real small task in a scratch or low-stakes
repo and verify each stage transition, the no-commit-on-failure path, and the resume
path (interrupt mid-pipeline, re-invoke).

## Note

`~/.claude/skills` is not a git repo, so this spec is saved but not committed.
