# Resume Template

Use this as the structural guide when generating `docs/sessions/resume.md`. Fill in all `{{placeholders}}` with project-specific content.

---

## Generated File Structure

```markdown
# Resume Orchestrator (Self-Healing)

Use this prompt when the orchestrator session's context exploded or was interrupted.
Copy everything below `---` into a new Claude Code session in `{{PROJECT_DIR}}`.

---

You are the development orchestrator for {{PROJECT_NAME}}. **Your previous session was interrupted.** Resume from where it left off.

## Recovery Steps

1. **Read progress state:**
   Read `docs/sessions/progress.json`

2. **Read the orchestrator rules:**
   Read `docs/sessions/orchestrator.md` for the full pipeline flow, dispatch templates, and orchestration logic. Follow those rules exactly.

3. **Read the plans** (for context):
   - `{{PLAN_PATH}}`
   - `{{DESIGN_PATH}}` (if exists)

4. **Read recent git history:**
   `git log --oneline -30`

5. **Read any existing review/QA reports:**
   `ls docs/reviews/ docs/qa/ 2>/dev/null`

6. **Determine where to resume** based on `progress.json`:

| `current_step` | What to do |
|----------------|------------|
| `execute` | Executor was in progress. Check git log — if commits exist for current batch tasks, executor may have partially finished. Dispatch executor for remaining tasks only. |
| `review` | Review was in progress or not started. Check if review report exists. If not, dispatch reviewer. If exists but incomplete, dispatch reviewer again. |
| `fix` | Fixes were in progress. Check if fix commits exist since review report. Dispatch executor for remaining Critical/Important issues. |
| `fix_verify` | Fix verification was in progress. Dispatch reviewer to verify fixes. |
| `qa` | QA was in progress. Check if QA report exists. If not, dispatch QA. |
| `qa_fix` | QA bug fixes in progress. Check QA report for remaining bugs. Dispatch executor. |
| `qa_verify` | QA re-verification. Dispatch QA to verify fixes. |
| `done` | Everything is complete. Nothing to do. |

7. **Resume the orchestration loop** from the determined point, following the rules in `docs/sessions/orchestrator.md`.

## Start

Read `docs/sessions/progress.json`, then read `docs/sessions/orchestrator.md`. Determine current state, announce where you're resuming from, and continue the pipeline.
```
