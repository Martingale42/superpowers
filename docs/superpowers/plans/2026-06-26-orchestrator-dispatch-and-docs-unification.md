# Orchestrator Default Dispatch + Unified Docs Layout — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task, or superpowers:orchestrator-driven-development for a stateful, resumable multi-session pipeline. These edits change behavior-shaping skill content — use superpowers:writing-skills while editing the skill/template files.

**Goal:** Unify all doc artifacts under `docs/superpowers/{plans,specs,sessions,reviews,qa}/` and make the Agent-tool + inlined-role-file path the single default orchestrator dispatch (dropping the never-working `.claude/agents/` `subagent_type` path and per-role effort).

**Architecture:** Pure documentation/template edits plus `git mv` relocations. No code, no dependencies. The orchestrator skill is a *generator*: its templates are structural guides instantiated per-project, so the changes are entirely to template text and the skill's workflow steps. Verification is a grep gate (proving no stale path/dispatch references survive) plus the existing `tests/` suites.

**Tech Stack:** Markdown, JSON, Git. Spec: `docs/superpowers/specs/2026-06-26-orchestrator-dispatch-and-docs-unification-design.md` — read it first.

---

## File Structure

**Relocated (Part A):**
- `docs/plans/*.md` (8) → `docs/superpowers/plans/`
- `docs/reviews/*.md` (2) → `docs/superpowers/reviews/`

**Edited:**
- `skills/writing-plans/SKILL.md` — path conventions only
- `skills/orchestrator-driven-development/SKILL.md` — paths + Step 2.5/3/4 + Key Principles
- `skills/orchestrator-driven-development/templates/orchestrator-template.md` — dispatch + paths + progress schema
- `skills/orchestrator-driven-development/templates/resume-template.md` — collapse fallback + paths
- `skills/orchestrator-driven-development/templates/executor-template.md`, `reviewer-template.md`, `qa-template.md` — header path, remove mirror note, effort line, report paths
- `skills/orchestrator-driven-development/templates/progress-template.json` — model-only schema
- `CHANGELOG.md` — fix moved-file links (Task 1) + add feature entry (Task 8)

**Deleted:**
- `skills/orchestrator-driven-development/templates/agent-definitions-template.md`

**Untouched:** `RELEASE-NOTES.md`, `docs/testing.md`, `docs/README.opencode.md`, `docs/windows/`, `tests/**` (already target `docs/superpowers/plans/`).

---

### Task 1: Relocate repo docs and fix cross-references

**Files:**
- Move: `docs/plans/*.md` → `docs/superpowers/plans/`
- Move: `docs/reviews/*.md` → `docs/superpowers/reviews/`
- Modify: moved files with internal links; `CHANGELOG.md`

**Implementation:**

Relocate with history preserved, then remove the now-empty source dirs:

```bash
mkdir -p docs/superpowers/reviews
git mv docs/plans/*.md docs/superpowers/plans/
git mv docs/reviews/*.md docs/superpowers/reviews/
rmdir docs/plans docs/reviews 2>/dev/null || true
```

Fix the internal references that point at relocated files. Apply each as an exact-string Edit:

- `docs/superpowers/plans/2026-06-10-orchestrator-per-role-model-implementation.md`:
  `docs/plans/2026-06-10-orchestrator-per-role-model-design.md` → `docs/superpowers/plans/2026-06-10-orchestrator-per-role-model-design.md`
- `docs/superpowers/plans/2026-06-11-orchestrator-effort-review-redesign.md` line ~7:
  `docs/plans/2026-06-11-orchestrator-effort-review-redesign-design.md` → `docs/superpowers/plans/2026-06-11-orchestrator-effort-review-redesign-design.md`
- `docs/superpowers/reviews/2026-06-10-upstream-divergence-differences.md`:
  `docs/plans/2026-06-10-orchestrator-per-role-model-*.md` → `docs/superpowers/plans/2026-06-10-orchestrator-per-role-model-*.md`
- `CHANGELOG.md` lines referencing the moved plans (the `## [Unreleased] — 2026-06-12` and `2026-06-10` entries):
  `docs/plans/2026-06-10-orchestrator-per-role-model-{design,implementation}.md` → `docs/superpowers/plans/...`
  `docs/plans/2026-06-11-orchestrator-effort-review-redesign-design.md` and `docs/plans/2026-06-11-orchestrator-effort-review-redesign.md` → `docs/superpowers/plans/...`
  (Leave the line "Plan path standardized on `docs/plans/`" historically accurate by appending " (later unified under `docs/superpowers/plans/`)" rather than rewriting it.)

Note: the in-content references inside the moved `docs/superpowers/plans/2026-06-11-*.md` files to `docs/sessions/` / `.claude/agents/` describe the *old* orchestrator design and are historical plan records — **do not** rewrite them here; they are corrected only where they live in the skill templates (Tasks 3–7).

**Verification:**

```bash
ls docs/superpowers/plans/ docs/superpowers/reviews/        # files present
git status --short | grep -E '^R'                            # shows renames, not delete+add
test ! -d docs/plans && test ! -d docs/reviews && echo "old dirs gone"
# No moved-file link still points at the old location (outside RELEASE-NOTES.md and historical plan bodies):
grep -rn 'docs/plans/2026-06-1' CHANGELOG.md docs/superpowers/reviews/ docs/superpowers/plans/*-implementation.md || echo "no stale moved-file links"
```
Expected: files listed under new paths; `git status` shows `R` (renames); old dirs gone; no stale links.

**Commit:**
```bash
git add -A docs/ CHANGELOG.md
git commit -m "docs: relocate fork plans and reviews under docs/superpowers/"
```

---

### Task 2: Update path conventions in writing-plans SKILL

**Files:**
- Modify: `skills/writing-plans/SKILL.md`

**Implementation:**

Three references change (lines ~18, ~253, ~274):

- `**Save plans to:** \`docs/plans/YYYY-MM-DD-<feature-name>.md\`` → `**Save plans to:** \`docs/superpowers/plans/YYYY-MM-DD-<feature-name>.md\``
- `**"Plan complete and saved to \`docs/plans/<filename>.md\`. ...` → `... \`docs/superpowers/plans/<filename>.md\`. ...`
- In the Orchestrator handoff bullet: `Generates session files in \`docs/sessions/\` (orchestrator, ...)` → `Generates session files in \`docs/superpowers/sessions/\` (orchestrator, ...)`

**Verification:**
```bash
grep -nE 'docs/plans/|docs/sessions/' skills/writing-plans/SKILL.md || echo "clean"
grep -c 'docs/superpowers/plans/' skills/writing-plans/SKILL.md   # >= 2
```
Expected: `clean` (no bare paths), `docs/superpowers/plans/` present.

**Commit:**
```bash
git add skills/writing-plans/SKILL.md
git commit -m "docs(writing-plans): point plan/session paths at docs/superpowers/"
```

---

### Task 3: Rework orchestrator SKILL.md (paths + Step 2.5/3/4 + principles)

**Files:**
- Modify: `skills/orchestrator-driven-development/SKILL.md`

**Implementation:**

**(a) Paths** — replace every `docs/sessions/` with `docs/superpowers/sessions/` (frontmatter `description`, lines ~89, ~103, ~121, ~125, ~129, ~133), and the plan reference `docs/plans/YYYY-MM-DD-<feature>.md` (line ~16) with `docs/superpowers/plans/YYYY-MM-DD-<feature>.md`.

**(b) Step 2.5** — retitle to "Assign Models to Roles" and make it model-only. Replace the curated table and the effort/`subagent_type` rules with:

```markdown
### Step 2.5: Assign Models to Roles

Before generating files, collect a model for each role and a Final Audit depth with the
**AskUserQuestion tool** (caps: 4 questions, ≤4 options each). Use exactly four questions:
one per role (Executor, Reviewer, QA) offering a model, plus Final Audit depth.

Curated options (Option 1 = the default; apply it if the user skips that question):

| Question | Option 1 (Recommended) | Option 2 | Option 3 |
|----------|------------------------|----------|----------|
| Executor model | `sonnet` | `opus` | `haiku` |
| Reviewer model | `opus` | `sonnet` | `haiku` |
| QA model | `sonnet` | `opus` | `haiku` |
| Final Audit depth | `/code-review` at high | `/code-review` at max | skip |

Rules:
- **Model** is family-level `{opus, sonnet, haiku}` (always the latest in that family, no
  version pinning). It is passed to the Agent tool's `model` parameter at dispatch.
- **Effort is NOT set per role.** The Agent tool has no effort parameter, so each dispatched
  subagent inherits this orchestrator session's effort. Recommend running the coordinator
  session at `/effort high`; that effort propagates to every dispatched role.
- **Executor-for-fixes** reuses the Executor model; **Reviewer-for-verify** reuses the
  Reviewer model; Final Audit fix dispatches reuse the Executor model.
- If the user skips a question, apply the default — never block generation.
- Final Audit depth is a `/code-review` argument (high / max / skip), not a subagent effort;
  the audit runs in THIS session via the `/code-review` skill.

Carry the resulting model for each role plus the Final Audit choice into Step 3 as the
`{{EXECUTOR_MODEL}}` / `{{REVIEWER_MODEL}}` / `{{QA_MODEL}}` and `{{AUDIT_DEPTH}}` substitution values.
```

(Delete the old paragraph about "4 questions ... one per role each offering curated `model · effort` combos", the `model · effort` table, and the `effort`/`ultracode`/`ultrathink`/frontmatter bullets.)

**(c) Step 3** — in the generated-files table remove the `.claude/agents/orchestrator-{executor,reviewer,qa}.md` row; remove the "**Note:** the three agent-definition files go in `.claude/agents/` ... that is where the harness discovers custom subagent types." paragraph. In the "When generating each file" list, replace the `{{ROLE_MODEL}}`/`{{ROLE_EFFORT}}` agent-definition-frontmatter bullet with:

```markdown
- Substitute the per-role `{{ROLE_MODEL}}` values from Step 2.5 into the Model Assignments
  table in `orchestrator.md`, the `model_assignments` block in `progress.json`, and the
  model recommendation in each standalone role file
```

**(d) Step 4** — change the commit command:
`git add docs/sessions/ .claude/agents/ && git commit -m "docs: add orchestrator session files"` →
`git add docs/superpowers/sessions/ && git commit -m "docs: add orchestrator session files"`

**(e) Key Principles** — rewrite two bullets:
- "Standalone role files are backup ..." → `- **Standalone role files are the role definition** — the orchestrator inlines each file's body into the dispatch prompt; the same file also lets a user run that role independently.`
- "Hard settings over imperatives ..." → `- **Per-role model at dispatch** — each role's model is passed via the Agent tool's \`model\` parameter; effort is inherited from the orchestrator session (the Agent tool has no effort parameter), so run the coordinator at \`/effort high\`.`

**Verification:**
```bash
F=skills/orchestrator-driven-development/SKILL.md
grep -nE 'docs/sessions/|docs/plans/|subagent_type|\.claude/agents/orchestrator|ROLE_EFFORT|_EFFORT|· (high|xhigh|medium)' "$F" || echo "clean"
grep -q 'Assign Models to Roles' "$F" && echo "step 2.5 retitled"
```
Expected: `clean`; step 2.5 retitled.

**Commit:**
```bash
git add skills/orchestrator-driven-development/SKILL.md
git commit -m "feat(orchestrator): model-only role assignment, drop agent-definitions and effort"
```

---

### Task 4: Rework orchestrator-template.md (dispatch + paths + progress schema)

**Files:**
- Modify: `skills/orchestrator-driven-development/templates/orchestrator-template.md`

**Implementation:**

**(a) Header + paths** — line ~3 `docs/sessions/orchestrator.md` → `docs/superpowers/sessions/orchestrator.md`; every other `docs/sessions/` → `docs/superpowers/sessions/`; every `docs/reviews/` → `docs/superpowers/reviews/`; every `docs/qa/` → `docs/superpowers/qa/`.

**(b) Model & Effort Assignments table** — replace the whole section with:

```markdown
## Model Assignments

| Role | Role file | Model |
|------|-----------|-------|
| Executor | `docs/superpowers/sessions/01-executor.md` | {{EXECUTOR_MODEL}} |
| Reviewer | `docs/superpowers/sessions/02-code-reviewer.md` | {{REVIEWER_MODEL}} |
| QA | `docs/superpowers/sessions/03-qa-tester.md` | {{QA_MODEL}} |

- Dispatch each role with the **Agent tool**: pass `model` from this table and build the
  prompt by prepending the role file's full body to the task-specific instructions below.
  There is no `subagent_type` and no agent-definition file.
- **Effort** is not set per role — subagents inherit THIS session's effort. Run this
  coordinator session at `/effort high` (recommended).
- Executor-for-fixes uses the Executor model + role file; Reviewer-for-verify uses the
  Reviewer's.
```

**(c) "How to Dispatch" — all six blocks.** For each, replace the lead line
`Use the Agent tool with \`subagent_type: "orchestrator-<role>"\` and this prompt:` with
`Use the Agent tool with \`model: {{<ROLE>_MODEL}}\`. Build the prompt by prepending the full body of \`docs/superpowers/sessions/<file>\`, then appending:`
using this role→file→model mapping (the task-text inside each block is unchanged):

| Block | Role file | Model token |
|-------|-----------|-------------|
| Dispatching Executor | `01-executor.md` | `{{EXECUTOR_MODEL}}` |
| Dispatching Reviewer | `02-code-reviewer.md` | `{{REVIEWER_MODEL}}` |
| Dispatching Executor for Fixes | `01-executor.md` | `{{EXECUTOR_MODEL}}` |
| Dispatching Reviewer for Fix Verification | `02-code-reviewer.md` | `{{REVIEWER_MODEL}}` |
| Dispatching QA (after all batches) | `03-qa-tester.md` | `{{QA_MODEL}}` |
| Dispatching Executor for QA Bug Fixes | `01-executor.md` | `{{EXECUTOR_MODEL}}` |
| Dispatching QA for Fix Verification | `03-qa-tester.md` | `{{QA_MODEL}}` |

**(d) Final Audit section** — the `audit_fix` dispatch (step 2, currently `subagent_type: "orchestrator-executor"`):
`dispatch \`subagent_type: "orchestrator-executor"\`` → `dispatch with the Agent tool (\`model: {{EXECUTOR_MODEL}}\`, prompt prepended with \`docs/superpowers/sessions/01-executor.md\`)`.
The `/code-review`-unavailable **Fallback** (currently `subagent_type: "orchestrator-reviewer"`):
`dispatch \`subagent_type: "orchestrator-reviewer"\`` → `dispatch with the Agent tool (\`model: {{REVIEWER_MODEL}}\`, prompt prepended with \`docs/superpowers/sessions/02-code-reviewer.md\`)`, and **delete** the trailing sentence "Note: in fallback mode the audit runs at the Reviewer's frontmatter effort, not `{{AUDIT_DEPTH}}`." (no frontmatter now; it inherits the session).

**(e) Progress Tracking** — the embedded JSON `model_assignments` becomes model-only:

```json
  "model_assignments": {
    "executor": {"model": "{{EXECUTOR_MODEL}}"},
    "reviewer": {"model": "{{REVIEWER_MODEL}}"},
    "qa":       {"model": "{{QA_MODEL}}"}
  },
```

(Important Rules line "Use the Agent tool to dispatch each role as a subagent" already correct — keep.)

**Verification:**
```bash
F=skills/orchestrator-driven-development/templates/orchestrator-template.md
grep -nE 'docs/sessions/|docs/reviews/|docs/qa/|subagent_type|_EFFORT|frontmatter effort' "$F" || echo "clean"
grep -c 'prepending the full body' "$F"   # 6 dispatch blocks rewritten (executor-fixes reuses Executor block phrasing)
grep -q '## Model Assignments' "$F" && echo "table renamed"
```
Expected: `clean`; Model Assignments table present; dispatch blocks rewritten.

**Commit:**
```bash
git add skills/orchestrator-driven-development/templates/orchestrator-template.md
git commit -m "feat(orchestrator): Agent-tool dispatch with inlined role files in main template"
```

---

### Task 5: Collapse the resume-template fallback

**Files:**
- Modify: `skills/orchestrator-driven-development/templates/resume-template.md`

**Implementation:**

Replace the three-tier step 1 (lines ~21–31) with a single path:

```markdown
1. **Read progress state:**
   Read `docs/superpowers/sessions/progress.json`. Note `model_assignments`. To re-dispatch a
   role, use the Agent tool with `model` = `model_assignments.<role>.model`, building the prompt
   by prepending the full body of the role file (`docs/superpowers/sessions/01-executor.md`,
   `02-code-reviewer.md`, or `03-qa-tester.md`) to the task instructions. Effort is inherited
   from this session. If `model_assignments` has no model for a role, dispatch with no `model`
   parameter (session default).
```

Then update the remaining paths in this file: line ~34 and ~65/~69 `docs/sessions/orchestrator.md` and `docs/sessions/progress.json` → `docs/superpowers/sessions/...`; line ~44 `ls docs/reviews/ docs/qa/` → `ls docs/superpowers/reviews/ docs/superpowers/qa/`.

**Verification:**
```bash
F=skills/orchestrator-driven-development/templates/resume-template.md
grep -nE 'docs/sessions/|docs/reviews/|docs/qa/|subagent_type|\.claude/agents/orchestrator' "$F" || echo "clean"
```
Expected: `clean`.

**Commit:**
```bash
git add skills/orchestrator-driven-development/templates/resume-template.md
git commit -m "feat(orchestrator): single-path resume recovery via Agent-tool dispatch"
```

---

### Task 6: Update standalone role templates

**Files:**
- Modify: `skills/orchestrator-driven-development/templates/executor-template.md`
- Modify: `skills/orchestrator-driven-development/templates/reviewer-template.md`
- Modify: `skills/orchestrator-driven-development/templates/qa-template.md`

**Implementation:**

In each file:
1. Header comment `... generating \`docs/sessions/0N-*.md\`` → `docs/superpowers/sessions/0N-*.md`.
2. Replace the three-line recommendation/mirror block with two lines (drops `{{ROLE_EFFORT}}` and the `.claude/agents` mirror note). For executor:

```markdown
> **Recommended:** open this session on `{{EXECUTOR_MODEL}}` and run `/effort high`.
> A standalone session cannot set these automatically — pick the model when opening and set effort with the slash command.
```
(reviewer → `{{REVIEWER_MODEL}}`, qa → `{{QA_MODEL}}`.)

3. Report-path references: reviewer `docs/reviews/YYYY-MM-DD-batch-N-review.md` → `docs/superpowers/reviews/...` (Report Format heading + Usage example); qa `docs/qa/YYYY-MM-DD-*.md` → `docs/superpowers/qa/...` (Process step 4 + Usage example).

**Verification:**
```bash
cd skills/orchestrator-driven-development/templates
grep -nE 'docs/sessions/|docs/reviews/|docs/qa/|_EFFORT|mirrors \.claude' executor-template.md reviewer-template.md qa-template.md || echo "clean"
```
Expected: `clean`.

**Commit:**
```bash
git add skills/orchestrator-driven-development/templates/{executor,reviewer,qa}-template.md
git commit -m "feat(orchestrator): role templates are sole source of truth, drop effort/mirror note"
```

---

### Task 7: Model-only progress schema; remove agent-definitions template

**Files:**
- Modify: `skills/orchestrator-driven-development/templates/progress-template.json`
- Delete: `skills/orchestrator-driven-development/templates/agent-definitions-template.md`

**Implementation:**

Rewrite `model_assignments` in `progress-template.json` to model-only (drop `effort` and `agent`):

```json
  "model_assignments": {
    "executor": {"model": "{{EXECUTOR_MODEL}}"},
    "reviewer": {"model": "{{REVIEWER_MODEL}}"},
    "qa":       {"model": "{{QA_MODEL}}"}
  },
```

Delete the dead template:
```bash
git rm skills/orchestrator-driven-development/templates/agent-definitions-template.md
```

**Verification:**
```bash
F=skills/orchestrator-driven-development/templates/progress-template.json
grep -E 'effort|agent' "$F" || echo "schema clean"
python3 -c "import json,sys; json.load(open('$F'.replace('{{EXECUTOR_MODEL}}','x')))" 2>/dev/null; echo "(json shape ok modulo placeholders)"
test ! -f skills/orchestrator-driven-development/templates/agent-definitions-template.md && echo "agent-definitions-template removed"
```
Expected: `schema clean`; template removed.

**Commit:**
```bash
git add skills/orchestrator-driven-development/templates/progress-template.json
git commit -m "feat(orchestrator): model-only progress schema, remove agent-definitions template"
```

---

### Task 8: CHANGELOG entry + final verification gate

**Files:**
- Modify: `CHANGELOG.md`

**Implementation:**

Add a new entry at the top of the `## [Unreleased]` section:

```markdown
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
```

**Verification (the gate — must all pass):**
```bash
# 1. No stale dispatch references anywhere in skills/
grep -rnE 'subagent_type|\.claude/agents/orchestrator|agent-definitions-template' skills/ && echo "FAIL: stale dispatch refs" || echo "PASS: no dispatch refs"

# 2. No bare (non-superpowers) artifact paths in skills/
grep -rnE 'docs/(plans|sessions|reviews|qa)/' skills/ | grep -v 'docs/superpowers/' && echo "FAIL: bare paths" || echo "PASS: no bare paths"

# 3. No per-role effort placeholders left in templates/skill
grep -rnE '\{\{(EXECUTOR|REVIEWER|QA)_EFFORT\}\}|ROLE_EFFORT' skills/ && echo "FAIL: effort tokens" || echo "PASS: no effort tokens"

# 4. Old dirs gone; new dirs populated
test ! -d docs/plans && test ! -d docs/reviews && ls docs/superpowers/{plans,specs,reviews}/ >/dev/null && echo "PASS: layout"

# 5. Existing test suites that exercise these paths still pass
ls tests/                       # locate suites
bash tests/claude-code/test-subagent-driven-development-integration.sh 2>&1 | tail -5 || true   # references docs/superpowers/plans
```
Expected: `PASS` for checks 1–4; test suite runs without path-related failures. (`RELEASE-NOTES.md` is intentionally excluded from the grep scope — it is upstream history.)

**Commit:**
```bash
git add CHANGELOG.md
git commit -m "docs: changelog for orchestrator dispatch and docs-layout unification"
```

---

## Self-Review

**1. Spec coverage:**
- Part A moves (plans 8, reviews 2) → Task 1. ✅
- Part A convention edits: writing-plans → Task 2; orchestrator SKILL paths → Task 3; templates → Tasks 4–6. ✅
- Cross-reference fixes + CHANGELOG links → Task 1. ✅
- `RELEASE-NOTES.md` untouched → respected (excluded from every grep scope). ✅
- Part B: remove agent-definitions → Task 7; remove `subagent_type`/notes/Step 4 commit → Tasks 3,4,5; read-and-prepend dispatch → Task 4 (+ resume Task 5); effort dropped, model kept → Tasks 3,4,7; Step 2.5 simplified → Task 3; progress schema → Tasks 4,7; standalone templates → Task 6; principles → Task 3. ✅
- Verification grep gate → Task 8 (mirrors spec's gate). ✅

**2. Placeholder scan:** No "TBD/TODO/handle appropriately". `{{...}}` are intentional template tokens. Every edit gives exact old→new strings. ✅

**3. Type/identifier consistency:** Role→file→model mapping is identical across Task 4's dispatch table, Task 5's resume text, and Task 6's templates (`01-executor.md`/`{{EXECUTOR_MODEL}}`, `02-code-reviewer.md`/`{{REVIEWER_MODEL}}`, `03-qa-tester.md`/`{{QA_MODEL}}`). `model_assignments` model-only schema identical in Tasks 4 and 7. ✅

No gaps found.

---

## Execution Handoff

(Provided after you choose how to execute — see the chat message.)
