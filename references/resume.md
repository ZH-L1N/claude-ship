# /ship resume dispatch (Stage −1 detail)

Read this when `plans/<version>-<slug>.md` for the current topic already
exists on disk. It decides fresh-run vs resume and the re-entry stage.
Never bump the plan version for a resume.

## Is it a resume candidate?

The plan file alone is not enough — the ship must be unfinished:

- Probe for an unpushed topic commit FIRST (`git log @{u}..HEAD --oneline`
  when an upstream exists; no upstream → every local commit is unpushed).
  A stamped-implemented plan whose commit hasn't pushed is check 1's exact
  state — the implemented status alone never disqualifies.
- Plan stamped `**Status: implemented …**` AND its commit already pushed →
  finished ship. Treat as a fresh run (Stage 2 bumps the version) with
  Stage 0's clean-tree STOP fully armed.
- Plan with no `**Status: implemented …**` line → always a candidate.

Every resume re-reads the size call and rounds ceiling from the plan
header instead of re-running Stage 1.

## Wrecked-git-state guard (before any check)

Probe via git, not literal `.git/` paths (worktrees relocate them):
`test -e "$(git rev-parse --git-path rebase-merge)"` — likewise
`rebase-apply` and `MERGE_HEAD`; `--git-path` exits 0 either way, so test
the printed path. Any hit → a rebase or merge is in progress → STOP loudly
and tell the user to resolve it; never resolve conflicts inside /ship.

## Entry checks (first match wins)

1. Unpushed commit whose message references the topic → Stage 7 **step 5
   only** (push). Steps 1–4 already ran in the prior session — never
   re-stamp TIMELINE.md or create a second commit. Uncommitted work on top
   of it → fall through to check 2.
2. Working diff outside the pipeline's own pre-implementation artifacts —
   `plans/`, `.scratch/`, `docs/superpowers/specs/`, and
   `.env`/`.gitignore` (the first-run branch may touch those) → Stage 6 if
   the ship-review log ends in a `review-loop:` marker; otherwise
   Stage 5 — when in doubt, re-review rather than skip review on a guess.
   A diff confined to those paths is not implementation output — fall
   through.
3. Plan + sidecars in `plans/fixs/` → Stage 3 (auto-resume mid-review).
4. Plan, no sidecars → Stage 3 from round 1.

## Resume side-effects on later stages

- Stage 0 still runs, but its clean-tree STOP is skipped — the diff is the
  pipeline's own output. A dirty tree with NO resume match is still a hard
  STOP: that's the user's work, never /ship's to touch.
- Stage 3 exit (standard tasks): the scratchpad spec draft from Stage 2 is
  session-scoped and gone. If the spec file already exists in the repo,
  leave it; otherwise re-draft it from the reviewed plan.
- Stage 5: rounds already spent = the highest `ship-review round <n>` in
  the ship-review log; 0 remaining → straight to Stage 6.
