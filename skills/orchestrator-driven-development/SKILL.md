---
name: orchestrator-driven-development
description: "Use when user picks orchestrator execution option after writing a plan. Generates session files (orchestrator, resume, executor, reviewer, QA, progress.json) in docs/superpowers/sessions/ for a multi-agent pipeline with review gates, QA, and a final audit."
---

# Orchestrator-Driven Development

## Overview

Generate a complete set of session files that power a multi-agent orchestration pipeline. The orchestrator runs in a parallel session, dispatching executor/reviewer/QA as subagents in a deterministic cycle with review gates.

**Announce at start:** "I'm using the orchestrator-driven-development skill to generate session files."

## When This Runs

This skill is invoked when the user picks option 3 (Orchestrator) from the writing-plans execution handoff. By this point, a plan already exists at `docs/superpowers/plans/YYYY-MM-DD-<feature>.md`.

## Workflow

### Step 1: Gather Project Context

Read and extract from the codebase:

1. **Plan file** — task list, phases, verification commands, commit conventions
2. **Design doc** — if referenced in the plan header
3. **Project-specific rules** — linting, formatting, coding conventions (e.g., `no unwrap()` for Rust, `bun run build` for frontend)
4. **Verification commands** — build, test, lint, format commands from the plan or codebase
5. **Project name** — from the plan header or directory name
6. **Done signal** — the plan header's acceptance command(s); note which are env-gated
   and what env they need (these become `{{DONE_SIGNAL_COMMANDS}}` /
   `{{LIVE_ENV_REQUIREMENTS}}` and decide whether the live gate is generated)

### Step 2: Define Batch Order

Group plan tasks into batches following these rules:

- **Group by phase** — each phase in the plan becomes a batch
- **Batch table** — number, phase name, task range, short description
- **Ordering** — follow the plan's phase order (which may differ from task number order)
- If the plan has no phases, group into batches of 3-5 related tasks

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

### Step 3: Generate Session Files

Create the session files in `<project-root>/docs/superpowers/sessions/`:

| File | Template | Purpose |
|------|----------|---------|
| `orchestrator.md` | `orchestrator-template.md` | Main orchestrator prompt — pipeline flow, dispatch templates, orchestration loop, progress tracking |
| `resume.md` | `resume-template.md` | Recovery prompt — reads progress.json, points to orchestrator.md for full rules |
| `01-executor.md` | `executor-template.md` | Standalone executor for ad-hoc use |
| `02-code-reviewer.md` | `reviewer-template.md` | Standalone reviewer for ad-hoc use |
| `03-qa-tester.md` | `qa-template.md` | Standalone QA tester for ad-hoc use |
| `progress.json` | `progress-template.json` | Initial state: batch 1, step execute |

**When generating each file:**
- Read the corresponding template for structure and format
- Fill in project-specific content: project name, plan paths, batch order, rules, verification commands
- Adapt dispatch prompts to the project's language/framework (Rust → cargo, JS → npm/bun, Python → pytest, etc.)
- Substitute the per-role `{{ROLE_MODEL}}` values from Step 2.5 into the Model Assignments
  table in `orchestrator.md`, the `model_assignments` block in `progress.json`, and the
  model recommendation in each standalone role file
- Substitute `{{AUDIT_DEPTH}}` and `{{MAIN_BRANCH}}` (the repo's default branch, e.g.
  `main` or `master`) in `orchestrator.md`
- If the user chose **skip** for Final Audit, omit all audit content at generation time
  per the generation notes in `orchestrator-template.md` and `resume-template.md` —
  keep only the `audit_status` field in progress.json, with the rule that the
  orchestrator sets it to `"SKIPPED"` when QA passes
- Substitute `{{DONE_SIGNAL_COMMANDS}}` and `{{LIVE_ENV_REQUIREMENTS}}` from the plan
  header's **Done signal** field. If the plan declares NO env-gated done signal, omit
  all live-gate content per the generation notes in `orchestrator-template.md` and
  `resume-template.md` (the `live_gate_status`/`live_gate_attempts` fields stay in
  progress.json at their null/0 defaults)

### Step 4: Commit and Hand Off

1. Commit all session files: `git add docs/superpowers/sessions/ && git commit -m "docs: add orchestrator session files"`
2. Tell the user:

```
Session files generated in docs/superpowers/sessions/.

To start the orchestrator:
1. Open a new Claude Code session in this project directory
2. Paste the content of docs/superpowers/sessions/orchestrator.md as the initial prompt

If the orchestrator session gets interrupted:
1. Open a new session
2. Paste the content of docs/superpowers/sessions/resume.md as the initial prompt
```

## Pipeline Architecture (for reference)

The generated orchestrator follows this cycle:

```
Per Batch:
  Executor (implement) → Reviewer (review)
    → if CHANGES_REQUESTED: Executor (fix) → Reviewer (verify) → loop (max 3)
    → if still CHANGES_REQUESTED after 3: STOP, ask user
    → if APPROVED: next batch

After All Batches:
  QA (full test) → if FAIL: Executor (fix) → QA (verify) → loop (max 2)
    → if still FAIL after 2: STOP, ask user
    → if PASS: done

After QA PASS:
  Final Audit (/code-review on whole branch) → if Critical: Executor fix → re-audit (max 2)
    → if still Critical after 2: STOP, ask user
    → else: non-blocking findings → final-audit BACKLOG

After Final Audit PASS (only if the plan declares an env-gated done signal):
  Live Gate: open DRAFT PR → env preflight → run the done signal
    → env missing: STOP, ask user (provide env, or explicit waiver)
    → FAIL (code defect): Executor fix → re-run (max 2; env/infra failures don't count)
    → still FAIL after 2: Exhaustion SOP — freeze (BLOCKED), failure report,
      backlog row, then STOP with a 4-option user menu
      (extend / escalate-to-plan / waive / park)
    → PASS or recorded waiver: flip PR to ready — pipeline complete
```

## Key Principles

- **Templates are structural guides, not literal substitution** — read the template for format, fill in project-specific content based on what you gathered in Step 1
- **Language-agnostic** — adapt verification commands and coding rules to the project's stack
- **Self-healing via progress.json** — the orchestrator can resume from any interruption
- **Standalone role files are the role definition** — the orchestrator inlines each file's body into the dispatch prompt; the same file also lets a user run that role independently.
- **resume.md points to orchestrator.md** — keeps one source of truth for the pipeline rules
- **Per-role model at dispatch** — each role's model is passed via the Agent tool's `model` parameter; effort is inherited from the orchestrator session (the Agent tool has no effort parameter), so run the coordinator at `/effort high`.
- **Review the tests, not just the code** — the reviewer checklist targets test acceptance logic and doc claims, and the Final Audit re-checks the whole branch with fresh eyes
- **SKIP ≠ PASS** — tests may skip on missing env; the pipeline may not. An env-gated done signal is a *gate*: the PR stays draft until the gate passes or the user explicitly waives it (waiver reason recorded in `live_gate_status` and disclosed in the PR body and final summary)
