---
name: orchestrator-driven-development
description: "Use when user picks orchestrator execution option after writing a plan. Generates session files (orchestrator, resume, executor, reviewer, QA, progress.json) in docs/sessions/ for a multi-agent pipeline with review gates and QA."
---

# Orchestrator-Driven Development

## Overview

Generate a complete set of session files that power a multi-agent orchestration pipeline. The orchestrator runs in a parallel session, dispatching executor/reviewer/QA as subagents in a deterministic cycle with review gates.

**Announce at start:** "I'm using the orchestrator-driven-development skill to generate session files."

## When This Runs

This skill is invoked when the user picks option 3 (Orchestrator) from the writing-plans execution handoff. By this point, a plan already exists at `docs/plans/YYYY-MM-DD-<feature>.md`.

## Workflow

### Step 1: Gather Project Context

Read and extract from the codebase:

1. **Plan file** — task list, phases, verification commands, commit conventions
2. **Design doc** — if referenced in the plan header
3. **Project-specific rules** — linting, formatting, coding conventions (e.g., `no unwrap()` for Rust, `bun run build` for frontend)
4. **Verification commands** — build, test, lint, format commands from the plan or codebase
5. **Project name** — from the plan header or directory name

### Step 2: Define Batch Order

Group plan tasks into batches following these rules:

- **Group by phase** — each phase in the plan becomes a batch
- **Batch table** — number, phase name, task range, short description
- **Ordering** — follow the plan's phase order (which may differ from task number order)
- If the plan has no phases, group into batches of 3-5 related tasks

### Step 3: Generate Session Files

Create all files in `<project-root>/docs/sessions/`:

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

### Step 4: Commit and Hand Off

1. Commit all session files: `git add docs/sessions/ && git commit -m "docs: add orchestrator session files"`
2. Tell the user:

```
Session files generated in docs/sessions/.

To start the orchestrator:
1. Open a new Claude Code session in this project directory
2. Paste the content of docs/sessions/orchestrator.md as the initial prompt

If the orchestrator session gets interrupted:
1. Open a new session
2. Paste the content of docs/sessions/resume.md as the initial prompt
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
```

## Key Principles

- **Templates are structural guides, not literal substitution** — read the template for format, fill in project-specific content based on what you gathered in Step 1
- **Language-agnostic** — adapt verification commands and coding rules to the project's stack
- **Self-healing via progress.json** — the orchestrator can resume from any interruption
- **Standalone role files are backup** — the orchestrator dispatches roles as subagents, but the standalone files let users run roles independently
- **resume.md points to orchestrator.md** — keeps one source of truth for the pipeline rules
