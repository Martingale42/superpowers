# Changelog

Fork-specific changes for `Martingale42/superpowers` that are not (yet) in upstream
`obra/superpowers`. This branch (`fork-main`) is the working fork: it combines the
**pragmatic-testing philosophy** (the `non-tdd` line) with **upstream v5.1.0**, merged in
from the `main` base. Upstream releases are tracked separately in `RELEASE-NOTES.md`; this
file is kept distinct so it never conflicts on an upstream merge.

Format loosely follows [Keep a Changelog](https://keepachangelog.com/).

## [Unreleased] — 2026-07-03

### Added — live done-signal gate after the final audit (orchestrator + writing-plans)

Lesson from a real milestone: its only end-to-end test was env-gated (`#[ignore]`),
every offline gate was green, and the headline feature shipped compiled-but-never-executed.
The pipeline had silently promoted "missing env is a SKIP" (a test-layer convention)
into "offline green = done" (a completion definition). Fixes, in layers:

- **`writing-plans`**: plan header gains a **Done signal** field (acceptance command(s),
  env-gated ones marked with their env needs). New rule: an env-gated done signal MUST
  come with an *offline rehearsal* task (externals stubbed, real entry point exercised) —
  the live run validates the environment, never the wiring.
- **`orchestrator-driven-development` QA role**: runs the offline done-signal rehearsal as
  part of full QA (a rehearsal failure is a FAIL, not a skip); verdict vocabulary gains
  `OFFLINE-PASS-LIVE-PENDING`, which explicitly does NOT satisfy the live gate.
- **`orchestrator-driven-development` pipeline**: new **LIVE GATE** stage after the final
  audit (generated only when the plan declares an env-gated done signal). Env preflight
  runs at session start (user prepares env in parallel with the batches) and again at the
  gate. The orchestrator opens a **draft PR** after the audit; the PR leaves draft only on
  gate PASS or an explicit, reason-recorded user waiver (`live_gate_status:
  "WAIVED: <reason>"`, disclosed in the PR body and final summary). Gate failures are
  triaged env-vs-code; code defects get max 2 executor fix cycles (env failures don't
  count); exhaustion triggers a **Live-Gate Exhaustion SOP** — freeze
  (`"BLOCKED: <report>"`, PR stays draft), immutable failure report, backlog row, then a
  STOP with a 4-option user menu (extend / escalate-to-plan / waive / park). A resumed
  session that finds BLOCKED re-presents the menu instead of retrying. progress.json gains
  `live_gate_status` / `live_gate_attempts`; `current_step` gains
  `live_gate` / `live_fix` / `live_verify`. Core principle now stated in both skills:
  **SKIP ≠ PASS** — tests may skip on missing env; the pipeline may not.

## [Unreleased] — 2026-06-26

### Changed — orchestrator default dispatch + unified docs layout

- **Agent-tool dispatch is now the only path.** The orchestrator dispatches each role with the
  Agent tool's `model` parameter, prepending the standalone role file's body as the role
  definition. The `.claude/agents/orchestrator-{executor,reviewer,qa}.md` generation and all
  `subagent_type` dispatch are removed — they never resolved reliably in Claude Code.
- **Per-role effort dropped.** The Agent tool has no effort parameter; subagents inherit the
  orchestrator session's effort. Step 2.5 now collects per-role *model* only (plus Final Audit
  depth). Recommend running the coordinator at `/effort high`.
- **Unified doc layout** under `docs/superpowers/{plans,specs,sessions,reviews,qa}/`. Existing
  `docs/plans/` and `docs/reviews/` relocated; `writing-plans` and `orchestrator-driven-development`
  now emit the unified paths.
- Removed `templates/agent-definitions-template.md`; `progress.json` `model_assignments` is
  model-only.

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
