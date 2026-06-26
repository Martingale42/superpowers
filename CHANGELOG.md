# Changelog

Fork-specific changes for `Martingale42/superpowers` that are not (yet) in upstream
`obra/superpowers`. This branch (`fork-main`) is the working fork: it combines the
**pragmatic-testing philosophy** (the `non-tdd` line) with **upstream v5.1.0**, merged in
from the `main` base. Upstream releases are tracked separately in `RELEASE-NOTES.md`; this
file is kept distinct so it never conflicts on an upstream merge.

Format loosely follows [Keep a Changelog](https://keepachangelog.com/).

## [Unreleased] — 2026-06-16

### Merged — `fork-main` = non-tdd philosophy + upstream v5.1.0

Created `fork-main` by merging `main` (upstream v5.1.0 + the harness-neutral orchestrator)
into the `non-tdd` pragmatic-testing fork. Resolution policy: **upstream-wins-on-structure,
fork-wins-on-philosophy**.

- **Adopted from upstream v5.1.0** (the fork held stale v4.3.1 copies): `using-git-worktrees`
  (worktree data-loss/consent fixes #940/#991/#999/#238), `brainstorming` (zero-dependency
  visual-companion server), subagent context isolation, `finishing-a-development-branch`,
  `requesting-code-review` (named agent → `general-purpose`), `dispatching-parallel-agents`,
  `using-superpowers` (+ harness reference files), multi-harness support, governance docs,
  and release tooling (`bump-version.sh`).
- **Kept from the fork (pragmatic philosophy):** `pragmatic-testing` (replaces
  `test-driven-development`, which stays deleted with its anti-patterns reference),
  `systematic-debugging` (TDD coupling removed), and the full
  `orchestrator-driven-development` superset (per-role model/effort, `subagent_type`
  dispatch, Final Audit gate, 10-item reviewer checklist, audit self-healing resume).
- **Hand-merged (upstream structure + pragmatic content):**
  - `README.md` — upstream's multi-harness install matrix + the fork's pragmatic workflow
    lines and "Pragmatic Testing Approach" appendix.
  - `writing-plans/SKILL.md` — upstream's File Structure / Scope Check / No Placeholders /
    Self-Review sections + the fork's pragmatic task structure, "When to Include Tests"
    table, and Task Types examples.
  - `subagent-driven-development/SKILL.md` + `implementer-prompt.md` — upstream's Model
    Selection, four-status protocol, Code-Organization and escalation sections + the fork's
    pragmatic Testing Expectations (no rigid test-first).
- **`writing-skills/SKILL.md`** kept at the fork's lean, 500-line-compliant version; upstream
  never changed this skill in v5, so `main`'s 655-line copy is just the pre-fork, TDD-framed
  original.
- Plan path standardized on `docs/plans/` (later unified under `docs/superpowers/plans/`). The 8 files upstream deleted in v5 stay deleted
  (`lib/skills-core.js`, `agents/code-reviewer.md`, `commands/*.md`, `.codex/INSTALL.md`,
  `docs/README.codex.md`, `tests/opencode/test-skills-core.sh`).
- **Deferred follow-up:** residual TDD wording in `using-superpowers` ("Rigid (TDD…)") and
  `verification-before-completion` ("Red-Green") is shared from the merge base (not a
  conflict) and left for a separate pragmatic-alignment pass.

## [Unreleased] — 2026-06-12

### Changed

- **Per-role effort is now a hard setting for `orchestrator-driven-development`.**
  Generation writes three subagent definitions — `.claude/agents/orchestrator-{executor,reviewer,qa}.md`
  — whose frontmatter carries both model and `effort: low/medium/high/xhigh/max`, enforced
  by the harness on every dispatch. The thinking-keyword mechanism from 2026-06-10 is
  **removed**: `think`/`think hard` are no-ops in current Claude Code, and `ultrathink`
  survives only as a user-facing one-shot toggle the pipeline does not use.
  - Dispatch is via `subagent_type` with a documented three-tier fallback: agent
    definition → Agent-tool `model` parameter + inlined standalone role file (effort
    unenforced) → bare dispatch (pre-feature sessions).
  - New defaults: Executor `sonnet`/high, Reviewer `opus`/xhigh, QA `sonnet`/high.
  - Commits `2f1b6f6`, `f56857f`, `7fd3a76`, `a1e54a6`.

### Added

- **Final Audit gate.** After QA passes, the orchestrator runs `/code-review` over the
  whole branch diff vs the default branch: Critical findings trigger executor fix cycles
  (max 2), non-blocking findings land in a final-audit BACKLOG instead of blocking
  completion. Depth is configurable (high / max / skip) via a fourth Step 2.5 question;
  progress/resume gained the matching audit steps. Commits `a361225`, `5379334`, `a5105d9`.
- **Reviewer checklist hardened 6 → 10** — new items target test acceptance logic,
  generated-artifact doc sync, coverage vs the accepted domain, and quantitative claims.
  Motivated by a post-hoc audit of an orchestrator-built feature that found **1 Critical
  + 4 High** the pipeline missed — all in test tolerance logic and artifact docs.
  Commits `f56857f` (agent body), `cc82da4` (standalone template).
- New `templates/agent-definitions-template.md` (source for the three generated agent
  files) and explicit QA fix-cycle dispatch blocks in the orchestrator template.
  Commits `f56857f`, `58202a4`.

### Docs

- Design and implementation plan:
  `docs/superpowers/plans/2026-06-11-orchestrator-effort-review-redesign-design.md` and
  `docs/superpowers/plans/2026-06-11-orchestrator-effort-review-redesign.md`.

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

- Design and implementation plan: `docs/superpowers/plans/2026-06-10-orchestrator-per-role-model-{design,implementation}.md`.
- **Upstream divergence review** (`docs/superpowers/reviews/2026-06-10-upstream-*.md`): the fork is
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
