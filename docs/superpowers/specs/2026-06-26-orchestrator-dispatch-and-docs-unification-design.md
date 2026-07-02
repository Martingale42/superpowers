# Design: Unified `docs/superpowers/` layout + Agent-tool dispatch as default

**Date:** 2026-06-26
**Status:** Approved (brainstorming) — ready for implementation plan
**Scope:** `skills/orchestrator-driven-development/`, `skills/writing-plans/`, repo `docs/` reorg, `CHANGELOG.md`
**Type:** Fork-specific (`Martingale42/superpowers`); not an upstream PR

---

## Problem

Two defects in the current fork:

1. **Fragmented docs layout.** Plan/spec/session/review/QA artifacts are scattered across
   `docs/plans/`, `docs/superpowers/{plans,specs}/`, `docs/sessions/`, `docs/reviews/`, and
   `docs/qa/`. Upstream v5 moved brainstorming/plans output under `docs/superpowers/`, but the
   fork's older plans and the orchestrator's runtime outputs never followed, leaving two parallel
   conventions.

2. **The orchestrator's primary dispatch path never worked.** The skill generates
   `.claude/agents/orchestrator-{executor,reviewer,qa}.md` with `model:` + `effort:` frontmatter and
   dispatches via `subagent_type: "orchestrator-<role>"`, treating both as harness-enforced "hard
   settings." In practice this never resolved/enforced reliably when the orchestrator prompt is pasted
   into a fresh Claude Code session. The Agent tool exposes a `model` parameter but **no** `effort`
   parameter, so per-role *effort* was never enforceable. The intended tier-2 fallback (Agent tool
   `model` param + inlined standalone role file) is the only path that maps onto what the harness can
   actually do.

## Goals

- One unified artifact root: `docs/superpowers/{plans,specs,sessions,reviews,qa}/`.
- A single, working orchestrator dispatch path: the Agent tool with a `model` parameter, inlining the
  standalone role file as the role definition.
- Honest effort handling: per-role *model* preserved; per-role *effort* removed (subagents inherit the
  orchestrator session's effort).
- Remove the dead `.claude/agents/` generation entirely.

## Non-goals

- Tuning any skill's triggering description or behavior beyond the items above.
- Changing any non-orchestrator skill's behavior.
- Opening a PR to upstream `obra/superpowers`.
- Editing `RELEASE-NOTES.md` (upstream release history; would conflict on merge).

---

## Part A — Unified `docs/superpowers/{plans,specs,sessions,reviews,qa}/`

### Target structure

```
docs/
├── superpowers/
│   ├── plans/      # writing-plans output  (already exists; absorbs docs/plans/*)
│   ├── specs/      # brainstorming output   (already exists; unchanged)
│   ├── sessions/   # orchestrator session files (NEW convention; runtime output)
│   ├── reviews/    # reviewer + final-audit reports (NEW; absorbs docs/reviews/*)
│   └── qa/         # QA reports (NEW convention; runtime output)
├── testing.md          # unchanged — general project doc
├── README.opencode.md  # unchanged — general project doc
└── windows/            # unchanged — general project doc
```

`sessions/` and `qa/` have no committed files in this repo (they are downstream runtime outputs);
they are established as conventions only.

### File moves (this repo, via `git mv` to preserve history)

| From | To | Count |
|------|----|-------|
| `docs/plans/*.md` | `docs/superpowers/plans/` | 8 |
| `docs/reviews/*.md` | `docs/superpowers/reviews/` | 2 |

`docs/superpowers/plans/` and `docs/superpowers/specs/` already exist (upstream convention) and keep
their current files. Empty `docs/plans/` and `docs/reviews/` directories are removed after the move.

### Convention edits (paths the skills emit to downstream projects)

| File | Change |
|------|--------|
| `skills/writing-plans/SKILL.md` | `docs/plans/` → `docs/superpowers/plans/` (save path, "Plan complete" handoff line, orchestrator handoff `docs/sessions/` → `docs/superpowers/sessions/`) |
| `skills/orchestrator-driven-development/SKILL.md` | description + body: `docs/sessions/` → `docs/superpowers/sessions/`; the plan-path reference `docs/plans/...` → `docs/superpowers/plans/...` |
| `skills/orchestrator-driven-development/templates/*` | every `docs/sessions/` → `docs/superpowers/sessions/`, `docs/reviews/` → `docs/superpowers/reviews/`, `docs/qa/` → `docs/superpowers/qa/` |

Already on `docs/superpowers/` and unchanged: `skills/brainstorming/`, `skills/requesting-code-review/`,
`skills/subagent-driven-development/`, `tests/**` (the test scripts already create and reference
`docs/superpowers/plans/`).

### Cross-reference fixes

- Internal references inside the **moved** fork docs that point at sibling moved files
  (e.g. `docs/superpowers/plans/2026-06-10-...-implementation.md` referencing its design doc;
  `docs/superpowers/plans/2026-06-11-...` referencing `docs/sessions/`).
- `CHANGELOG.md` fork entries that link to relocated files (`docs/plans/2026-06-10-...`,
  `docs/plans/2026-06-11-...`) → `docs/superpowers/plans/...`.
- `docs/superpowers/reviews/2026-06-10-upstream-divergence-differences.md` reference to
  `docs/plans/2026-06-10-...` → `docs/superpowers/plans/...`.

### Explicitly untouched

`RELEASE-NOTES.md` — upstream release history. Its `docs/plans/` mentions are historical narrative of
upstream's own past changes, and it already names `docs/superpowers/{plans,specs}/` as the current
target. Editing it risks conflicts on the next upstream merge.

---

## Part B — Agent-tool dispatch as the default

### Background: the three tiers today

1. **Primary (removed):** generate `.claude/agents/orchestrator-<role>.md` (model + effort frontmatter),
   dispatch via `subagent_type`. Never worked reliably.
2. **Fallback (becomes the default):** Agent tool `model` param + inline the standalone role file's
   content; effort inherits the session.
3. **Last resort (kept as graceful degradation):** no `model` param; pure session defaults.

### New dispatch model — read-and-prepend

At each dispatch the orchestrator:

1. Reads the relevant standalone role file — `docs/superpowers/sessions/01-executor.md`,
   `02-code-reviewer.md`, or `03-qa-tester.md`.
2. Builds the subagent prompt = **(role file body) + (task-specific instructions)**.
3. Calls the **Agent tool** with `model: <role model from progress.json `model_assignments`>` and that
   prompt. No `subagent_type`.

The standalone role file is the **single source of truth** for role content (rules, the 10-item reviewer
checklist, QA categories). It is no longer a "backup" — it is the role definition that gets injected.

*Rejected alternative:* baking each full role body directly into the dispatch blocks in `orchestrator.md`.
Rejected because it duplicates the role definition and bloats `orchestrator.md`; read-and-prepend keeps
one copy.

### Effort handling

- Per-role **effort** is removed. The Agent tool cannot set it; subagents inherit the orchestrator
  session's effort.
- Per-role **model** is preserved via the Agent tool `model` parameter (family-level `{opus, sonnet,
  haiku}`, latest in family — unchanged selection rules).
- `orchestrator.md` recommends the operator run the coordinator session at `/effort high`; this effort
  propagates to all dispatched subagents.
- The standalone role files keep a model recommendation; their per-role `/effort {{ROLE_EFFORT}}` line is
  replaced with a single fixed recommendation (`/effort high`) and the operator-sets-effort note.

### File-by-file changes

**`skills/orchestrator-driven-development/SKILL.md`**
- **Step 2.5** — collapse to model-only. Four `AskUserQuestion`s: Executor model, Reviewer model, QA
  model, Final Audit depth. Drop all effort columns/rows from the curated option table; drop the
  effort-is-a-hard-setting bullets. Keep model defaults (Executor `sonnet`, Reviewer `opus`, QA `sonnet`,
  Audit `/code-review` at high). Add: subagents inherit session effort; recommend session `/effort high`.
- **Step 3** — remove the `.claude/agents/orchestrator-{executor,reviewer,qa}.md` row from the generated-
  files table and the "harness discovers custom subagent types" note. The standalone session files
  (`01/02/03-*.md`) and `progress.json` are the only generated artifacts besides `orchestrator.md` /
  `resume.md`. Drop the `{{ROLE_EFFORT}}` / agent-definition substitution bullet.
- **Step 4** — commit command `git add docs/sessions/ .claude/agents/ ...` → `git add
  docs/superpowers/sessions/ && git commit ...` (no `.claude/agents/`).
- **Key Principles** — rewrite "Hard settings over imperatives" (model enforced via Agent param; effort is
  session-inherited) and "Standalone role files are backup" (they are now the canonical, inlined role
  definition).

**`skills/orchestrator-driven-development/templates/orchestrator-template.md`**
- **Model & Effort Assignments** table → **Model Assignments**: drop the Effort column; replace the
  "Agent definition" column with "Role file" pointing at `docs/superpowers/sessions/0N-*.md`. Rewrite the
  surrounding bullets (no `subagent_type`, no keyword toggles; dispatch = Agent tool `model` + inlined role
  file; effort inherited).
- **How to Dispatch** (all six blocks — executor, reviewer, executor-fixes, reviewer-verify, QA,
  QA-fixes, QA-verify): replace `Use the Agent tool with subagent_type: "orchestrator-<role>"` with
  `Use the Agent tool with model: {{ROLE_MODEL}}, prompting with the contents of
  docs/superpowers/sessions/0N-<role>.md followed by:` + the existing task text.
- **Final Audit** — the `audit_fix` dispatch (currently `subagent_type: "orchestrator-executor"`) and the
  `/code-review`-unavailable fallback (currently `subagent_type: "orchestrator-reviewer"`) switch to Agent
  tool + inlined role file. Remove "runs at the Reviewer's frontmatter effort" (no frontmatter now;
  inherits session). Audit report paths `docs/reviews/...` → `docs/superpowers/reviews/...`.
- **Progress Tracking** — `docs/sessions/progress.json` → `docs/superpowers/sessions/progress.json`; the
  embedded `model_assignments` example drops `effort` and `agent`, keeping `model`.
- **Start** — `docs/sessions/progress.json` → `docs/superpowers/sessions/progress.json`.

**`skills/orchestrator-driven-development/templates/resume-template.md`**
- Collapse the three-tier recovery in step 1 to: read `model_assignments.<role>.model`, dispatch with the
  Agent tool `model` param + inlined `docs/superpowers/sessions/0N-*.md`. Keep "if no model recorded →
  dispatch with no model param (session default)" as the single graceful-degradation case. Remove the
  `.claude/agents/orchestrator-<role>.md` / `subagent_type` tier.
- All `docs/sessions/` → `docs/superpowers/sessions/`; `docs/reviews/ docs/qa/` listing →
  `docs/superpowers/reviews/ docs/superpowers/qa/`.

**`skills/orchestrator-driven-development/templates/executor-template.md`, `reviewer-template.md`, `qa-template.md`**
- Header comment `... generating docs/sessions/0N-*.md` → `docs/superpowers/sessions/0N-*.md`.
- Remove the "Content mirrors `.claude/agents/orchestrator-<role>.md` — update both" note (no second copy
  exists now).
- Replace the `> Recommended: open on {{ROLE_MODEL}}, then /effort {{ROLE_EFFORT}}` block with a model
  recommendation + fixed `/effort high` suggestion (no per-role effort placeholder).
- Report-path references `docs/reviews/` / `docs/qa/` → `docs/superpowers/reviews/` / `docs/superpowers/qa/`.

**`skills/orchestrator-driven-development/templates/progress-template.json`**
- `model_assignments` drops `effort` and `agent` per role; keeps `{"model": "{{ROLE_MODEL}}"}`.

**Removed file**
- `skills/orchestrator-driven-development/templates/agent-definitions-template.md` — `git rm`.

---

## Verification

1. **Grep sweep (must be empty in `skills/`, RELEASE-NOTES.md excepted):**
   - `docs/plans/`, `docs/sessions/`, `docs/reviews/`, `docs/qa/` (bare, non-`superpowers` forms)
   - `subagent_type`, `.claude/agents/orchestrator`
   - `{{EXECUTOR_EFFORT}}`, `{{REVIEWER_EFFORT}}`, `{{QA_EFFORT}}`, `agent-definitions-template`
2. **Moved-doc integrity:** `git mv` used (history preserved); no broken relative links in moved docs or
   `CHANGELOG.md` (grep for the old paths repo-wide and confirm only intended survivors remain).
3. **Existing tests:** run the `tests/` suites that touch these paths (they already target
   `docs/superpowers/plans/`, so they should pass unchanged) and the skill-triggering tests for the
   orchestrator skill.
4. **Self-consistency:** `progress-template.json`, the `orchestrator-template.md` embedded
   `model_assignments` example, and `resume-template.md`'s recovery logic all agree on the same
   `model`-only schema.

Implementation uses the **writing-skills** skill (these are behavior-shaping skill edits).

## Risks & rollback

- **Risk:** a residual `docs/sessions/` or `subagent_type` reference is missed in one template, leaving a
  broken instruction. *Mitigation:* the grep sweep in Verification is the gate; it covers every form.
- **Risk:** the next upstream merge reintroduces `docs/plans/` references. *Mitigation:* CHANGELOG fork
  entry documents the unified layout as fork policy; `RELEASE-NOTES.md` left untouched to minimize merge
  surface.
- **Rollback:** the whole change is doc/template text + `git mv`; revert the commit to restore.

## Out of scope

Triggering-description tuning, non-orchestrator skill behavior, upstream PR, any change to how
`/code-review` itself works.
