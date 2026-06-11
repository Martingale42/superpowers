# Orchestrator Effort Alignment + Review Hardening Implementation Plan

> **For Claude:** Use superpowers:executing-plans, superpowers:subagent-driven-development, or superpowers:orchestrator-driven-development to implement this plan.

**Goal:** Replace the deprecated thinking-keyword effort mechanism with hard per-role `model`/`effort` via generated `.claude/agents/` definition files, harden the reviewer checklist (6→10 items), and add a Final Audit pipeline gate that invokes `/code-review`.

**Architecture:** All changes live in `skills/orchestrator-driven-development/` (SKILL.md + 6 existing templates + 1 new template) plus a CHANGELOG entry. The skill is a generator: templates are structural guides that the skill instantiates per-project. Design doc: `docs/plans/2026-06-11-orchestrator-effort-review-redesign-design.md` — read it first; it contains the verified facts (Agent tool has no effort param; `.claude/agents/*.md` frontmatter `effort:` is a hard override; `think`/`think hard` are no-ops; `ultracode` is session-only and not an effort level).

**Tech Stack:** Markdown templates, JSON template. No test suite — verification is `grep` consistency checks and JSON parsing via `uv run python`.

**Tests:** Deferred — this is documentation/template work; verification is grep + JSON parse checks per task.

---

### Task 1: Rewrite SKILL.md Step 2.5 (effort semantics + 4-question shape)

**Files:**
- Modify: `skills/orchestrator-driven-development/SKILL.md`

**Implementation:**

Replace the entire "Step 2.5: Assign Models & Effort to Roles" section (lines ~39–87):

1. **Delete** the effort→thinking-keyword table (`minimal`/`low`/`medium`/`high` → keywords) and every mention of `{{*_THINKING}}` placeholders and keyword prepending.
2. **New question shape:** AskUserQuestion asks **4 questions** (at the tool's cap): Executor, Reviewer, QA (each `model · effort` combos), plus **Final Audit depth**. Keep the existing guidance about not exceeding 4 questions / 4 options and the auto-added Other choice.
3. **New curated combo table** (Option 1 = default, applied if the user skips that question):

| Role | Option 1 (Recommended) | Option 2 | Option 3 | Option 4 |
|------|------------------------|----------|----------|----------|
| Executor | `sonnet` · high | `sonnet` · xhigh | `opus` · high | `haiku` · medium |
| Reviewer | `opus` · xhigh | `opus` · high | `sonnet` · xhigh | `sonnet` · high |
| QA | `sonnet` · high | `sonnet` · xhigh | `opus` · high | `haiku` · medium |
| Final Audit | `/code-review` at high | `/code-review` at max | skip | — |

4. **New rules block** replacing the old one:
   - Model restricted to `{opus, sonnet, haiku}`, family-level latest (unchanged).
   - Effort is one of `low / medium / high / xhigh / max` — **also a hard setting now**: both model and effort are written into the generated `.claude/agents/orchestrator-<role>.md` frontmatter, which the harness enforces for every dispatch of that subagent type.
   - `ultracode` is a Claude Code **session-only setting** (xhigh + dynamic workflows), not an effort level: it cannot be assigned to roles and is not recommended for the orchestrator session itself (the coordinator should not spawn its own workflows).
   - `ultrathink` is a user-facing one-shot prompt toggle; the pipeline does **not** use it (frontmatter effort replaced keyword prepending; `think`/`think hard` are no-ops in current Claude Code).
   - Executor-for-fixes reuses Executor; Reviewer-for-verify reuses Reviewer; Final Audit fix dispatches reuse Executor (unchanged spirit).
   - Skipped questions → defaults; never block generation (unchanged).
   - Orchestrator session's own model/effort still cannot be set here; `orchestrator.md` records a recommendation (`opus`, `/effort high`).
5. Closing line: carry `(model, effort)` per role plus the Final Audit choice into Step 3 as substitution values for `{{EXECUTOR_MODEL}}`/`{{EXECUTOR_EFFORT}}`, `{{REVIEWER_*}}`, `{{QA_*}}`, and `{{AUDIT_DEPTH}}`.

**Verification:**

Run: `grep -n "THINKING\|think hard\|ultrathink\|minimal" skills/orchestrator-driven-development/SKILL.md`
Expected: `ultrathink` appears only in the one explanatory note saying the pipeline does not use it; no `{{*_THINKING}}`, no `think hard` as a mapped keyword, no `minimal` effort level.

**Commit:**
```bash
git add skills/orchestrator-driven-development/SKILL.md
git commit -m "feat(orchestrator): replace thinking keywords with hard effort levels in Step 2.5"
```

---

### Task 2: Create agent-definitions-template.md

**Files:**
- Create: `skills/orchestrator-driven-development/templates/agent-definitions-template.md`

**Implementation:**

New template, same style as the others (a "Generated File Structure" guide). It instructs the skill to generate **three files into `<project-root>/.claude/agents/`**: `orchestrator-executor.md`, `orchestrator-reviewer.md`, `orchestrator-qa.md`. Fixed names — `.claude/agents/` is project-level so there is no cross-project collision.

Each generated file has frontmatter + body (body = the subagent's system prompt; role-static content lives HERE, not in dispatch prompts):

````markdown
# Agent Definitions Template

Use this as the structural guide when generating the three files in
`<project-root>/.claude/agents/`. Frontmatter `model` and `effort` are HARD
settings enforced by the harness on every dispatch of that subagent type.

---

## orchestrator-executor.md

```markdown
---
name: orchestrator-executor
description: Implements plan tasks for the {{PROJECT_NAME}} orchestrator pipeline
model: {{EXECUTOR_MODEL}}
effort: {{EXECUTOR_EFFORT}}
---

You are the Executor for {{PROJECT_NAME}}.

Context:
- Implementation plan: `{{PLAN_PATH}}`
- Design doc: `{{DESIGN_PATH}}` (if exists)

Rules:
{{PROJECT_RULES}}
- Follow the plan's exact file paths, public APIs, and definitions
- After each task: run the verification command, then commit with the specified message
- If a task is blocked, document the blocker in a comment and skip to the next task

Verification commands:
{{VERIFICATION_COMMANDS}}

When done, output: completed tasks with commit hashes, skipped/blocked tasks
with reasons, final test output.
```

## orchestrator-reviewer.md

```markdown
---
name: orchestrator-reviewer
description: Code reviewer for the {{PROJECT_NAME}} orchestrator pipeline
model: {{REVIEWER_MODEL}}
effort: {{REVIEWER_EFFORT}}
---

You are the Code Reviewer for {{PROJECT_NAME}}.

Context:
- Design doc: `{{DESIGN_PATH}}` (if exists)
- Implementation plan: `{{PLAN_PATH}}`

Review checklist — for each changed file:
1. Correctness — does the code do what the plan says?
2. Error handling — errors have context, no silent failures
3. Security — injection, traversal, auth bypass
4. API conformance — matches design doc?
5. YAGNI — no over-engineering
6. Tests present and meaningful
7. Test acceptance logic — review tolerances/comparison logic with the same
   rigor as code: is each tolerance close to the error actually needed? Any
   unconditional escape hatches? Any assertions that are algebraically always
   true (dead)? Do comments describing tolerances tell the truth? Ask: "what
   wrong implementation would still pass these tests?"
8. Generated artifacts & doc sync — if a fix touches generated code/tables,
   the generator AND the generated artifact's docs/comments must reflect the
   post-fix convention (stale docs cause faithful regeneration of old bugs)
9. Coverage domain vs accepted domain — tests must cover the full input
   domain the public API accepts; either the API rejects what is untested or
   the tests expand to cover it
10. Quantitative claims — numeric precision/performance claims in docs or
    comments must be backed by a test or measurement in the diff; otherwise
    file a finding

Verification commands:
{{VERIFICATION_COMMANDS}}

Report format (write to the path given in your dispatch prompt, then commit):
- Verdict: APPROVED / APPROVED WITH NOTES / CHANGES REQUESTED
- Findings categorized Critical / Important / Minor, each with [file:line]
- Verification results
```

## orchestrator-qa.md

```markdown
---
name: orchestrator-qa
description: User-perspective QA tester for the {{PROJECT_NAME}} orchestrator pipeline
model: {{QA_MODEL}}
effort: {{QA_EFFORT}}
---

You are the QA Tester for {{PROJECT_NAME}}. Test like a real user — find bugs
and edge cases.

Context:
- Design doc: `{{DESIGN_PATH}}` (if exists)
- Implementation plan: `{{PLAN_PATH}}`

Process:
1. Build and run existing tests: {{BUILD_AND_TEST_COMMAND}}
2. {{QA_TEST_APPROACH}}
3. Test edge cases: {{EDGE_CASES}}
4. Write the QA report to the path given in your dispatch prompt; commit
   report and any test code.

Report format: results table (Test | Input | Expected | Actual | Status);
each bug has severity, reproduction steps, expected, actual, file:line.
```
````

Also include a short note in the template header: valid `effort` values are `low/medium/high/xhigh/max`; the value must be valid for the chosen model; `ultracode` is never valid here.

**Verification:**

Run: `grep -c "^name: orchestrator-" skills/orchestrator-driven-development/templates/agent-definitions-template.md`
Expected: `3`

**Commit:**
```bash
git add skills/orchestrator-driven-development/templates/agent-definitions-template.md
git commit -m "feat(orchestrator): add agent-definitions template with hard model/effort frontmatter"
```

---

### Task 3: Rework orchestrator-template.md dispatch blocks to subagent_type

**Files:**
- Modify: `skills/orchestrator-driven-development/templates/orchestrator-template.md`

**Implementation:**

1. **"Model & Effort Assignments" section** — replace table and notes with:

```markdown
## Model & Effort Assignments

| Role | Agent definition | Model | Effort |
|------|------------------|-------|--------|
| Executor | `.claude/agents/orchestrator-executor.md` | {{EXECUTOR_MODEL}} | {{EXECUTOR_EFFORT}} |
| Reviewer | `.claude/agents/orchestrator-reviewer.md` | {{REVIEWER_MODEL}} | {{REVIEWER_EFFORT}} |
| QA | `.claude/agents/orchestrator-qa.md` | {{QA_MODEL}} | {{QA_EFFORT}} |

- Model AND effort are **hard settings** enforced by the agent definition's frontmatter —
  dispatch with `subagent_type`, never prepend thinking keywords.
- Executor-for-fixes uses the Executor agent; Reviewer-for-verify uses the Reviewer agent.
- Fallback: if a `subagent_type` fails to resolve (agent file deleted), dispatch with the
  Agent tool's `model` parameter and inline the role content from the standalone role file
  (effort cannot be enforced in this mode — it inherits this session).
- Recommended: run THIS coordinator session on `opus` with `/effort high` — it cannot
  self-assign its own model or effort.
```

2. **Every dispatch block** ("Dispatching Executor", "Dispatching Reviewer", "Dispatching Executor for Fixes", "Dispatching Reviewer for Fix Verification", "Dispatching QA"): change the lead-in to `Use the Agent tool with subagent_type: "orchestrator-<role>"` and **delete all "Prepend {{*_THINKING}} … (omit if blank)" sentences**. Slim each prompt body down to batch-specific content only (role-static rules/checklists/report formats now live in the agent definition):
   - Executor: batch number, task range, reminder to read plan/design.
   - Reviewer: batch number/task range, `git log --oneline -20` to find batch commits, the report path `docs/reviews/YYYY-MM-DD-batch-N-review.md`, run `{{VERIFICATION_COMMANDS_ONELINE}}`, `{{EXTRA_REVIEW_SECTIONS}}`, output verdict + summary.
   - Fixes / Fix-verify / QA: same trimming pattern; keep report paths and max-cycle semantics.

**Verification:**

Run: `grep -n "THINKING\|Prepend" skills/orchestrator-driven-development/templates/orchestrator-template.md`
Expected: no matches (progress.json block is fixed in Task 5's companion change below — if grep still hits the progress block here, that is handled in Task 4/5).

Run: `grep -c "subagent_type" skills/orchestrator-driven-development/templates/orchestrator-template.md`
Expected: ≥ 5 (one per dispatch block).

**Commit:**
```bash
git add skills/orchestrator-driven-development/templates/orchestrator-template.md
git commit -m "feat(orchestrator): dispatch roles via subagent_type, drop thinking keywords"
```

---

### Task 4: Add Final Audit gate to orchestrator-template.md

**Files:**
- Modify: `skills/orchestrator-driven-development/templates/orchestrator-template.md`

**Implementation:**

1. **Pipeline Flow diagram** — after the FINAL QA box, add:

```
┌─────────────────────────────────────────────────────┐
│              FINAL AUDIT {{AUDIT_DEPTH_NOTE}}        │
│                                                      │
│  Orchestrator invokes the /code-review skill in     │
│  THIS session (whole feature-branch diff vs main,   │
│  effort: {{AUDIT_DEPTH}})                            │
│        ↓                                             │
│  Critical findings:                                  │
│        → Executor: fix → re-audit (max 2 cycles)    │
│  Non-blocking findings:                              │
│        → docs/reviews/YYYY-MM-DD-final-audit.md     │
│          BACKLOG section                             │
│                                                      │
└─────────────────────────────────────────────────────┘
```

   If the user chose **skip** in Step 2.5, the generating skill omits this section entirely.

2. **New "Final Audit" dispatch section** after "Dispatching QA":
   - Invoke the `/code-review` skill (Skill tool) at effort `{{AUDIT_DEPTH}}`, scoped to the full branch diff vs the main branch.
   - Critical/P0 findings → dispatch `orchestrator-executor` to fix → re-run the audit (max 2 cycles); still failing → STOP, ask user.
   - Non-blocking findings → write/append `docs/reviews/YYYY-MM-DD-final-audit.md` with a BACKLOG section; commit it.
   - **Fallback** if the `/code-review` skill is unavailable in this session: dispatch `orchestrator-reviewer` with a whole-branch audit prompt embedding multi-angle finder guidance (algorithmic math, test acceptance logic, doc claims, API contracts) plus per-finding adversarial verification before reporting.
   - **Never invoke `/code-review ultra`** — it is user-triggered and billed; local depth caps at `max`.

3. **Orchestration Logic pseudocode** — after the QA loop, add:

```python
# Final audit (skip if AUDIT_DEPTH == skip)
audit_result = run_code_review_skill(scope="branch", effort=AUDIT_DEPTH)
audit_attempts = 0
while audit_result.has_critical and audit_attempts < 2:
    dispatch_executor_fixes_from_audit(audit_result)
    audit_result = run_code_review_skill(scope="branch", effort=AUDIT_DEPTH)
    audit_attempts += 1
if audit_result.has_critical:
    STOP — ask user for guidance
write_backlog(audit_result.non_blocking)
```

4. **Important Rules** — add: "Max 2 final-audit fix cycles", and the ultra prohibition.

5. **Progress Tracking** — in the embedded progress.json example: `model_assignments` entries become `{"model": "…", "effort": "…", "agent": "orchestrator-<role>"}` (delete `"thinking"`); add top-level `"audit_status": null`; extend the `current_step` enum in "Update rules" with `final_audit`, `audit_fix`, `audit_verify`; add "After audit pass: set `audit_status` to `PASS` (or `SKIPPED`)".

**Verification:**

Run: `grep -n "THINKING\|thinking" skills/orchestrator-driven-development/templates/orchestrator-template.md`
Expected: no matches.

Run: `grep -c "final_audit\|audit_status\|AUDIT_DEPTH" skills/orchestrator-driven-development/templates/orchestrator-template.md`
Expected: ≥ 5.

**Commit:**
```bash
git add skills/orchestrator-driven-development/templates/orchestrator-template.md
git commit -m "feat(orchestrator): add Final Audit gate invoking /code-review after QA"
```

---

### Task 5: Update progress-template.json and resume-template.md

**Files:**
- Modify: `skills/orchestrator-driven-development/templates/progress-template.json`
- Modify: `skills/orchestrator-driven-development/templates/resume-template.md`

**Implementation:**

`progress-template.json`:
- Each `model_assignments` entry: `{"model": "{{ROLE_MODEL}}", "effort": "{{ROLE_EFFORT}}", "agent": "orchestrator-<role>"}` — remove `"thinking"`.
- Add `"audit_status": null` after `"qa_status"`.

`resume-template.md` — Recovery Step 1 becomes the three-tier fallback chain:

```markdown
1. **Read progress state:**
   Read `docs/sessions/progress.json`. Note `model_assignments`. To re-dispatch a role:
   (1) If `.claude/agents/orchestrator-<role>.md` exists, dispatch with
       `subagent_type: "orchestrator-<role>"` — model and effort are enforced by its
       frontmatter.
   (2) If the agent file is missing but `model_assignments` has the role, dispatch with
       the Agent tool's `model` parameter and inline the role content from the standalone
       role file (`docs/sessions/0N-*.md`); effort cannot be enforced in this mode.
   (3) If neither exists (session generated before this feature), dispatch with no
       `model` parameter — the session defaults.
```

Also extend the resume step table with three rows:

| `current_step` | What to do |
|----------------|------------|
| `final_audit` | Audit was in progress. Re-invoke `/code-review` (or the reviewer-audit fallback) on the whole branch. |
| `audit_fix` | Audit fixes in progress. Check the final-audit report for unresolved Critical findings; dispatch executor. |
| `audit_verify` | Re-run the audit to verify fixes (counts toward the 2-cycle cap). |

**Verification:**

Run: `uv run python -c "import json,re; s=open('skills/orchestrator-driven-development/templates/progress-template.json').read(); json.loads(s); print('valid JSON')"`
Expected: `valid JSON`

Run: `grep -n "thinking" skills/orchestrator-driven-development/templates/progress-template.json skills/orchestrator-driven-development/templates/resume-template.md`
Expected: no matches.

**Commit:**
```bash
git add skills/orchestrator-driven-development/templates/progress-template.json skills/orchestrator-driven-development/templates/resume-template.md
git commit -m "feat(orchestrator): record agent names in progress state, three-tier resume fallback"
```

---

### Task 6: Update standalone role templates (executor / reviewer / qa)

**Files:**
- Modify: `skills/orchestrator-driven-development/templates/executor-template.md`
- Modify: `skills/orchestrator-driven-development/templates/reviewer-template.md`
- Modify: `skills/orchestrator-driven-development/templates/qa-template.md`

**Implementation:**

All three templates:
1. Replace the recommendation blockquote with:
   ```markdown
   > **Recommended:** open this session on `{{ROLE_MODEL}}`, then run `/effort {{ROLE_EFFORT}}`.
   > A standalone session cannot set these automatically — pick the model when opening and
   > set effort with the slash command.
   ```
2. Add one line after it: `> Content mirrors .claude/agents/orchestrator-<role>.md — when editing the checklist or rules, update both.`

`reviewer-template.md` only:
3. Extend the Review Checklist from 6 to 10 items — same wording as the reviewer body in Task 2 (test acceptance logic; generated artifacts & doc sync; coverage domain vs accepted domain; quantitative claims).

**Verification:**

Run: `grep -c "^[0-9]*\." skills/orchestrator-driven-development/templates/reviewer-template.md | head -1` and `grep -n "/effort" skills/orchestrator-driven-development/templates/*-template.md`
Expected: reviewer checklist shows items 1–10; each of the three role templates contains exactly one `/effort {{…}}` recommendation.

**Commit:**
```bash
git add skills/orchestrator-driven-development/templates/executor-template.md skills/orchestrator-driven-development/templates/reviewer-template.md skills/orchestrator-driven-development/templates/qa-template.md
git commit -m "feat(orchestrator): /effort recommendations and 10-item reviewer checklist in standalone templates"
```

---

### Task 7: Update SKILL.md Steps 3–4, pipeline diagram, key principles

**Files:**
- Modify: `skills/orchestrator-driven-development/SKILL.md`

**Implementation:**

1. **Step 3 file table** — add a row:
   `| .claude/agents/orchestrator-{executor,reviewer,qa}.md | agent-definitions-template.md | Subagent definitions with hard model/effort frontmatter |`
   and note the output directory difference (`.claude/agents/`, not `docs/sessions/`).
2. **Step 3 "When generating each file"** — replace the thinking-keyword substitution bullet with: substitute `{{ROLE_MODEL}}`/`{{ROLE_EFFORT}}` into frontmatter and assignment tables; substitute `{{AUDIT_DEPTH}}`; if Final Audit = skip, omit the audit sections from orchestrator.md and set nothing in progress.json beyond `"audit_status": null` → generation note: write `"SKIPPED"` only when the pipeline reaches that point, or document that the orchestrator sets it.
3. **Step 4** — commit command becomes `git add docs/sessions/ .claude/agents/ && git commit -m "docs: add orchestrator session files"`.
4. **Pipeline Architecture** — append after QA:
   ```
   After QA PASS:
     Final Audit (/code-review on whole branch) → if Critical: Executor fix → re-audit (max 2)
       → if still Critical after 2: STOP, ask user
       → else: done (non-blocking findings → final-audit BACKLOG)
   ```
5. **Key Principles** — add: "Hard settings over imperatives — model and effort live in agent-definition frontmatter enforced by the harness, not in prompt keywords" and "Review the tests, not just the code — the reviewer checklist targets test acceptance logic and doc claims, and the Final Audit re-checks the whole branch with fresh eyes".

**Verification:**

Run: `grep -n "agent-definitions-template\|\.claude/agents/\|Final Audit" skills/orchestrator-driven-development/SKILL.md`
Expected: hits in Step 3 table, Step 4 commit command, Pipeline Architecture, Key Principles.

**Commit:**
```bash
git add skills/orchestrator-driven-development/SKILL.md
git commit -m "feat(orchestrator): wire agent definitions and Final Audit into generation steps"
```

---

### Task 8: CHANGELOG entry + repo-wide consistency sweep

**Files:**
- Modify: `CHANGELOG.md`

**Implementation:**

1. Add a new `## [Unreleased] — 2026-06-11` section above the 2026-06-10 one (or fold into the existing Unreleased block with a dated subsection — match the file's existing style) covering:
   - **Changed:** per-role effort is now a hard setting via generated `.claude/agents/orchestrator-{executor,reviewer,qa}.md` frontmatter (`effort: low/medium/high/xhigh/max`); thinking-keyword mechanism removed (`think`/`think hard` are no-ops in current Claude Code; only `ultrathink` remains as a user-facing one-shot toggle, unused by the pipeline). New defaults: Executor `sonnet`/high, Reviewer `opus`/xhigh, QA `sonnet`/high.
   - **Added:** Final Audit gate (post-QA `/code-review` over the whole branch, max 2 fix cycles, BACKLOG for non-blocking findings; configurable high/max/skip); reviewer checklist hardened 6→10 (test acceptance logic, generated-artifact doc sync, coverage vs accepted domain, quantitative claims) — motivated by a post-hoc audit that found 1 Critical + 4 High the pipeline missed, all in test tolerance logic and artifact docs.
   - Reference the design doc path.
2. **Consistency sweep** across the whole skill directory:

```bash
grep -rn "THINKING\|think hard\|minimal" skills/orchestrator-driven-development/
grep -rn "ultrathink" skills/orchestrator-driven-development/
```

Expected: BOTH commands hit only SKILL.md's single Step 2.5 explanatory note (which
intentionally contains `ultrathink` and the "`think`/`think hard` are no-ops" mention —
that note is mandated by Task 1 and must NOT be removed as a straggler). Any hit outside
that note is a straggler — fix it.

**Verification:**

Run: the two grep commands above plus `uv run python -c "import json; json.loads(open('skills/orchestrator-driven-development/templates/progress-template.json').read()); print('ok')"`
Expected: clean greps, `ok`.

**Commit:**
```bash
git add CHANGELOG.md skills/orchestrator-driven-development/
git commit -m "docs: changelog for effort alignment and review hardening"
```
