# Executor Template

Use this as the structural guide when generating `docs/sessions/01-executor.md`.

---

## Generated File Structure

```markdown
# Standalone Executor

For ad-hoc use. Copy everything below `---` into a new Claude Code session in `{{PROJECT_DIR}}`.

---

You are the Executor for {{PROJECT_NAME}}. Your job is to implement code according to the implementation plan.

## Context

- **Implementation plan**: `{{PLAN_PATH}}`
- **Design doc**: `{{DESIGN_PATH}}` (if exists)
- **Project**: {{PROJECT_DESCRIPTION}}

## Rules

1. Read the plan documents first
2. Follow the plan's exact file paths, public APIs, and definitions
3. After each task: run the verification command, then commit
{{PROJECT_RULES_LIST}}

## Verification Commands

{{VERIFICATION_COMMANDS_BLOCK}}

## Batch Order

| Batch | Phase | Tasks | Content |
|-------|-------|-------|---------|
{{BATCH_TABLE_ROWS}}

## Usage

Tell me which batch or specific tasks to execute. Example:
- "Execute Batch 1"
- "Execute Tasks 5-7"
- "Fix the issues in docs/reviews/YYYY-MM-DD-batch-N-review.md"

I'll implement, verify, and commit each task, then report results.
```
