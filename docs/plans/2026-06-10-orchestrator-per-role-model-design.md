# Design: Per-Role Model + Effort Designation for orchestrator-driven-development

**Date**: 2026-06-10
**Status**: Approved
**Scope**: `skills/orchestrator-driven-development/` (SKILL.md + 6 templates). No other skills touched, no new dependencies.

## Problem

When the orchestrator-driven-development skill generates session files, every role
(Executor, Reviewer, QA) is dispatched on whatever model the orchestrator session
happens to run. The user wants to assign a model — and a reasoning effort — per role,
e.g. `sonnet` for the Executor, `opus` at high effort for the Reviewer.

## Capability constraint (the load-bearing fact)

The orchestrator dispatches roles via the **Agent tool**, which exposes a `model`
parameter but **no effort/thinking parameter and no version pinning**:

| Knob | Agent tool support | Consequence |
|------|--------------------|-------------|
| Model (per role) | ✅ `model` param | Family-level only: `opus` / `sonnet` / `haiku`, each resolving to the latest in its family (opus→4.8, sonnet→4.6, haiku→4.5). |
| Effort (per role) | ❌ none | Must be expressed by injecting a **thinking-directive keyword** into the role's dispatch prompt. Soft lever (raises reasoning budget; not a hard guarantee). |
| Version pinning | ❌ none | `opus` always points at the newest opus; 4.7 cannot be pinned. |

The orchestrator file is a **prompt pasted into a fresh session**, not running code, so
"set the model" is implemented by baking `model: <choice>` into each role's dispatch
block — it takes effect when that session spawns the subagent.

## Design

### Effort → thinking-keyword mapping (single source of truth, lives in SKILL.md)

| Effort | Keyword injected into dispatch prompt |
|--------|----------------------------------------|
| `minimal` | (none) |
| `low` | `think` |
| `medium` | `think hard` |
| `high` | `ultrathink` |

### Roles and defaults (pre-filled in the interactive menu)

| Role | Default model | Default effort | Rationale |
|------|---------------|----------------|-----------|
| Executor | `sonnet` | medium | High-volume mechanical coding; fast and capable. |
| Reviewer | `opus` | high | Judgment-heavy; catches subtle bugs — worth the strongest model. |
| QA | `sonnet` | medium | Test execution and reproduction. |

- **Executor-for-fixes** reuses Executor's assignment; **Reviewer-for-verify** reuses Reviewer's.
- The **orchestrator session's own model** cannot be set by the skill (it is the session
  the user opens manually). The generated `orchestrator.md` adds a one-line recommendation
  ("run this coordinator session on `opus`").

### Interaction

In the skill's generation flow, use `AskUserQuestion` to collect model + effort for the
three roles, with the default table pre-filled so the user can accept in one tap or
customize. If skipped, the defaults apply — generation never blocks. Model choices are
constrained to `{opus, sonnet, haiku}` by the menu (built-in validation).

### Change surface (6 files)

1. **`SKILL.md`** — new *Step 2.5: Assign Models & Effort to Roles*: instruct the
   AskUserQuestion collection (with default table) and embed the effort→keyword mapping
   as the source of truth. Step 3 threads the result into the templates.
2. **`templates/orchestrator-template.md`** —
   (a) new `## Model & Effort Assignments` table;
   (b) every "Dispatching X" block states `Use the Agent tool with model: {{ROLE_MODEL}}`
       and `prepend {{ROLE_THINKING}} to the dispatch prompt`;
   (c) new placeholders: `{{EXECUTOR_MODEL}}`, `{{EXECUTOR_THINKING}}`,
       `{{REVIEWER_MODEL}}`, `{{REVIEWER_THINKING}}`, `{{QA_MODEL}}`, `{{QA_THINKING}}`.
3. **`templates/executor-template.md` / `reviewer-template.md` / `qa-template.md`** —
   a one-line "Recommended model/effort" note at the top (standalone files run in a
   manually-opened session, so this is documentation, not automation).
4. **`templates/progress-template.json`** — new `model_assignments` object persisting the
   three role choices.
5. **`templates/resume-template.md`** — new recovery step: read `model_assignments` from
   `progress.json` and re-dispatch with the same models/effort (otherwise self-healing
   restart silently drops the user's choices).

### Edge cases

- Skipped menu → defaults applied; generation never blocks.
- Generated files explicitly note: **model = hard guarantee; effort = soft lever; version
  not pinnable**.

### Verification

This skill is markdown templates, not runnable code. Verify by generating session files
for a sample plan and confirming: (a) each dispatch block carries the correct `model:` and
thinking keyword; (b) `resume.md` reads `model_assignments` back. Follow `writing-skills`
verification guidance.
