# Changelog

Fork-specific changes for `Martingale42/superpowers` that are not (yet) in upstream
`obra/superpowers`. Upstream releases are tracked separately in `RELEASE-NOTES.md`; this
file is kept distinct so it never conflicts on an upstream merge.

Format loosely follows [Keep a Changelog](https://keepachangelog.com/).

## [Unreleased] — 2026-06-10

### Added

- **Per-role model & effort designation for `orchestrator-driven-development`.**
  When the skill generates session files it now asks — via `AskUserQuestion`, one question
  per role — which model and reasoning effort the **Executor**, **Reviewer**, and **QA** run
  at, then bakes the model into every Agent-tool dispatch and injects a thinking-directive
  keyword to express effort. Choices persist in `progress.json` and are restored on a
  self-healing resume.
  - Model is a **hard** setting limited to `{opus, sonnet, haiku}` (family-level, latest
    version, no pinning); effort is a **soft** lever via thinking keyword
    (`minimal`→none, `low`→`think`, `medium`→`think hard`, `high`→`ultrathink`).
  - Defaults: Executor `sonnet`/medium, Reviewer `opus`/high, QA `sonnet`/medium.
    Executor-for-fixes reuses the Executor row; Reviewer-for-verify reuses the Reviewer row.
  - Files: `skills/orchestrator-driven-development/SKILL.md` (new Step 2.5) and all six
    templates. Commits `a811f3e`, `76a8726`, `138de2a`, `3a4abad`.

### Fixed

- **AskUserQuestion shape pinned** — Step 2.5 now prescribes exactly one question per role
  (≤4 questions, ≤4 curated `model · effort` options each, default first and marked
  `(Recommended)`), so generation cannot exceed the tool's caps or behave inconsistently
  across runs.
- **`minimal` effort no longer leaks `(none)`** — the thinking placeholder substitutes to an
  **empty string** rather than the literal text `(none)`, keeping it out of dispatch prompts
  and `progress.json`.
- **Resume gained a second fallback** — a session created before this feature existed (no
  `model_assignments` and no assignments table) now dispatches each role with **no `model`
  parameter**, instead of pointing at a table that wouldn't exist.
- Corrected the implementation-plan note that wrongly called `progress-template.json` invalid
  JSON (placeholders sit inside string values, so it parses as-is); switched its check to
  `uv run python`. Commit `81b7bf1`.

### Docs

- Design and implementation plan: `docs/plans/2026-06-10-orchestrator-per-role-model-{design,implementation}.md`.
- **Upstream divergence review** (`docs/reviews/2026-06-10-upstream-*.md`): the fork is
  155 commits / one major version behind upstream (`v4.3.1` vs `v5.1.0`). An isolated trial
  merge shows the update surfaces **exactly 5 conflicts**, all inside the local
  pragmatic-testing rewrite; fork-only skills are untouched. Recommendation: merge, with
  upstream-wins-on-structure resolution.

### Pending (intentionally not done this session)

- **Upstream merge to v5.1.0** — analyzed and recommended, but **deferred** until the
  per-role feature is validated in real use. See the update-recommendation review for the
  step-by-step merge plan.
- **Push to `origin/main`** — local `main` is ahead of `origin/main` by these commits; push
  is held pending the user's hands-on validation.
