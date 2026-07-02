# Per-Role Model + Effort Designation — Implementation Plan

> **For Claude:** Use superpowers:executing-plans, superpowers:subagent-driven-development, or superpowers:orchestrator-driven-development to implement this plan.

**Goal:** Let the user assign a model and reasoning effort per role (Executor, Reviewer, QA) when the orchestrator-driven-development skill generates session files.

**Architecture:** The orchestrator dispatches roles via the Agent tool (model-only, no effort param). So the skill (a) interactively collects model+effort per role at generation time, (b) bakes `model:` into each dispatch block, (c) injects a thinking-directive keyword to express effort, and (d) persists the assignments in `progress.json` so self-healing resume keeps them.

**Tech Stack:** Markdown skill + templates. No runnable code, no dependencies.

**Design doc:** `docs/superpowers/plans/2026-06-10-orchestrator-per-role-model-design.md`

**Base dir for all paths below:** `skills/orchestrator-driven-development/`

---

### Task 1: Add effort mapping + role-assignment step to SKILL.md

**Files:**
- Modify: `skills/orchestrator-driven-development/SKILL.md`

**Implementation:**

Insert a new step between Step 2 (Define Batch Order) and Step 3 (Generate Session Files),
numbered **Step 2.5: Assign Models & Effort to Roles**. It must:

1. Instruct: use `AskUserQuestion` to collect a model + effort for each of Executor,
   Reviewer, QA, pre-filling the defaults below so the user can accept in one tap.
2. Include the default table:

   | Role | Default model | Default effort |
   |------|---------------|----------------|
   | Executor | `sonnet` | medium |
   | Reviewer | `opus` | high |
   | QA | `sonnet` | medium |

3. Include the effort→keyword mapping (single source of truth):

   | Effort | Thinking keyword injected into dispatch prompt |
   |--------|------------------------------------------------|
   | `minimal` | (none) |
   | `low` | `think` |
   | `medium` | `think hard` |
   | `high` | `ultrathink` |

4. State the constraints explicitly: model is restricted to `{opus, sonnet, haiku}`
   (family-level, latest version); effort is a soft lever via thinking keyword, not a hard
   guarantee; if the user skips the question, apply defaults — never block generation.
5. State that Executor-for-fixes reuses the Executor assignment and Reviewer-for-verify
   reuses the Reviewer assignment.

Also update Step 3's "When generating each file" bullet list to add: "Substitute the model
and thinking-keyword placeholders from the Step 2.5 assignments."

**Verification:**

Run: `grep -n "Step 2.5" skills/orchestrator-driven-development/SKILL.md`
Expected: matches the new step heading. Re-read the step and confirm both tables and the
constraint paragraph are present.

**Commit:**
```bash
git commit -m "feat(orchestrator): collect per-role model/effort before generating session files"
```

---

### Task 2: Wire model + effort into orchestrator-template.md

**Files:**
- Modify: `skills/orchestrator-driven-development/templates/orchestrator-template.md`

**Implementation:**

1. After the `## Batch Order` section (before `## How to Dispatch Each Role`), add:

   ```markdown
   ## Model & Effort Assignments

   | Role | Model | Effort | Thinking keyword |
   |------|-------|--------|------------------|
   | Executor | {{EXECUTOR_MODEL}} | {{EXECUTOR_EFFORT}} | {{EXECUTOR_THINKING}} |
   | Reviewer | {{REVIEWER_MODEL}} | {{REVIEWER_EFFORT}} | {{REVIEWER_THINKING}} |
   | QA | {{QA_MODEL}} | {{QA_EFFORT}} | {{QA_THINKING}} |

   - **Model is a hard setting** passed to the Agent tool. **Effort is a soft lever** —
     the thinking keyword is prepended to the dispatch prompt to raise the reasoning budget.
   - Executor-for-fixes uses the Executor row; Reviewer-for-verify uses the Reviewer row.
   - Recommended: run THIS coordinator session on `opus` (it cannot self-assign its model).
   ```

2. In each "Dispatching X" subsection, change the opening line from
   "Use the Agent tool to spawn an … subagent. Provide it with:" to spell out the model and
   keyword. Example for Executor:

   ```markdown
   Use the Agent tool with `model: {{EXECUTOR_MODEL}}` to spawn an executor subagent.
   Prepend `{{EXECUTOR_THINKING}}` to the prompt below (omit if blank):
   ```

   Apply the matching placeholders: Reviewer + Reviewer-verify blocks use
   `{{REVIEWER_MODEL}}`/`{{REVIEWER_THINKING}}`; Executor-for-fixes uses
   `{{EXECUTOR_MODEL}}`/`{{EXECUTOR_THINKING}}`; QA block uses `{{QA_MODEL}}`/`{{QA_THINKING}}`.

3. In the `## Progress Tracking` JSON example, add a `"model_assignments"` key mirroring
   Task 4's shape so the orchestrator knows to keep it updated.

**Verification:**

Run: `grep -n "{{EXECUTOR_MODEL}}\|{{REVIEWER_MODEL}}\|{{QA_MODEL}}\|Model & Effort" skills/orchestrator-driven-development/templates/orchestrator-template.md`
Expected: the assignments table heading plus model placeholders in every dispatch block
(Executor, Executor-fix, Reviewer, Reviewer-verify, QA).

**Commit:**
```bash
git commit -m "feat(orchestrator): bake per-role model and thinking keyword into dispatch blocks"
```

---

### Task 3: Persist + restore assignments (progress.json + resume)

**Files:**
- Modify: `skills/orchestrator-driven-development/templates/progress-template.json`
- Modify: `skills/orchestrator-driven-development/templates/resume-template.md`

**Implementation:**

1. In `progress-template.json`, add a top-level key:

   ```json
   "model_assignments": {
     "executor": { "model": "{{EXECUTOR_MODEL}}", "effort": "{{EXECUTOR_EFFORT}}", "thinking": "{{EXECUTOR_THINKING}}" },
     "reviewer": { "model": "{{REVIEWER_MODEL}}", "effort": "{{REVIEWER_EFFORT}}", "thinking": "{{REVIEWER_THINKING}}" },
     "qa":       { "model": "{{QA_MODEL}}",       "effort": "{{QA_EFFORT}}",       "thinking": "{{QA_THINKING}}" }
   },
   ```

   (keep valid JSON — mind trailing commas).

2. In `resume-template.md`, add a recovery step after "Read progress state": read
   `model_assignments` from `progress.json` and re-dispatch every role with the same model
   and thinking keyword. Note: if `model_assignments` is missing (older session), fall back
   to the defaults in `orchestrator.md`.

**Verification:**

Run: `uv run python -c "import json; json.load(open('skills/orchestrator-driven-development/templates/progress-template.json')); print('VALID')"`
Expected: `VALID`. The placeholders sit inside JSON string values (e.g. `"model": "{{EXECUTOR_MODEL}}"`), so the template parses as valid JSON as-is — no substitution needed.
Also: `grep -n "model_assignments" skills/orchestrator-driven-development/templates/progress-template.json skills/orchestrator-driven-development/templates/resume-template.md`
Expected: key present in the JSON template and a restore step referencing it in resume.

**Commit:**
```bash
git commit -m "feat(orchestrator): persist per-role assignments for self-healing resume"
```

---

### Task 4: Document recommended model/effort in standalone role files

**Files:**
- Modify: `skills/orchestrator-driven-development/templates/executor-template.md`
- Modify: `skills/orchestrator-driven-development/templates/reviewer-template.md`
- Modify: `skills/orchestrator-driven-development/templates/qa-template.md`

**Implementation:**

Each standalone file runs in a session the user opens manually, so the model cannot be set
automatically — add a one-line note near the top of the generated-file body:

- executor: `> **Recommended:** run this session on {{EXECUTOR_MODEL}} (effort: {{EXECUTOR_EFFORT}}).`
- reviewer: `> **Recommended:** run this session on {{REVIEWER_MODEL}} (effort: {{REVIEWER_EFFORT}}).`
- qa: `> **Recommended:** run this session on {{QA_MODEL}} (effort: {{QA_EFFORT}}).`

**Verification:**

Run: `grep -n "Recommended:" skills/orchestrator-driven-development/templates/executor-template.md skills/orchestrator-driven-development/templates/reviewer-template.md skills/orchestrator-driven-development/templates/qa-template.md`
Expected: one recommendation line per file.

**Commit:**
```bash
git commit -m "docs(orchestrator): note recommended model/effort in standalone role files"
```

---

### Task 5: Dry-run verification

**Files:** none (verification only)

**Implementation:**

Mentally (or on a scratch copy) generate session files for a tiny sample plan with a chosen
assignment (e.g. Executor=sonnet/medium, Reviewer=opus/high, QA=sonnet/medium). Confirm:
- Every dispatch block in `orchestrator.md` carries the right `model:` and keyword
  (Reviewer block shows `model: opus` + `ultrathink`; Executor shows `model: sonnet` +
  `think hard`).
- `progress.json` `model_assignments` matches.
- `resume.md` restore step reads it back.
- No placeholder leaks (`{{...}}`) remain after substitution.

Follow `writing-skills` verification guidance for the final review.

**Verification:**

Run: `grep -rn "{{" skills/orchestrator-driven-development/templates/ | grep -i "model\|effort\|thinking"`
Expected: placeholders exist ONLY in the templates (intended), and you can trace each to a
Step 2.5 assignment. No orphan placeholders without a substitution source.

**Commit:** none (no file changes) — or a docs touch-up commit if the dry-run surfaces gaps.
