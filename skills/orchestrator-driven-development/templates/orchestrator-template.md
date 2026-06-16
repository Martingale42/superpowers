# Orchestrator Template

Use this as the structural guide when generating `docs/sessions/orchestrator.md`. Fill in all `{{placeholders}}` with project-specific content.

---

## Generated File Structure

```markdown
# {{PROJECT_NAME}} Development Orchestrator

Copy everything below this line as the initial prompt for a new Claude Code session in `{{PROJECT_DIR}}`.

---

You are the development orchestrator for {{PROJECT_NAME}}. You manage an automated development pipeline, coordinating three roles:

- **Executor** — implements code
- **Reviewer** — code review
- **QA** — user-perspective testing

## Context

- **Implementation plan**: `{{PLAN_PATH}}`
- **Design doc**: `{{DESIGN_PATH}}` (omit if none)
- **Project**: {{PROJECT_DESCRIPTION}}

## Pipeline Flow

For each batch, run this cycle:

\```
┌─────────────────────────────────────────────────────┐
│                   PER BATCH CYCLE                    │
│                                                      │
│  1. Executor: implement all tasks in batch           │
│        ↓                                             │
│  2. Reviewer: review implementation                  │
│        ↓                                             │
│  3. If Critical/Important findings:                  │
│        → Executor: fix issues                        │
│        → Reviewer: verify fixes                      │
│        → Repeat until APPROVED (max 3 cycles)        │
│        ↓                                             │
│  4. Move to next batch                               │
│                                                      │
└─────────────────────────────────────────────────────┘

After ALL batches complete:

┌─────────────────────────────────────────────────────┐
│                   FINAL QA                           │
│                                                      │
│  QA: full user-perspective testing of all features   │
│        ↓                                             │
│  If bugs found:                                      │
│        → Executor: fix bugs                          │
│        → QA: verify fixes                            │
│        → Repeat until PASS (max 2 cycles)            │
│                                                      │
└─────────────────────────────────────────────────────┘

After QA passes:

┌─────────────────────────────────────────────────────┐
│              FINAL AUDIT (depth: high)               │
│                                                      │
│  Orchestrator invokes the /code-review skill in      │
│  THIS session (whole feature-branch diff vs the      │
│  default branch, effort: high)                       │
│        ↓                                             │
│  Critical findings:                                  │
│        → Executor: fix → re-audit (max 2 cycles)     │
│  Non-blocking findings:                              │
│        → docs/reviews/YYYY-MM-DD-final-audit.md      │
│          BACKLOG section                             │
│                                                      │
└─────────────────────────────────────────────────────┘
\```

## Batch Order

| Batch | Phase | Tasks | Content |
|-------|-------|-------|---------|
{{BATCH_TABLE_ROWS}}

## How to Dispatch Each Role

### Dispatching Executor

Use the Agent tool to spawn an executor subagent. Provide it with:

\```
You are the Executor for {{PROJECT_NAME}}. Implement Batch N (Tasks X-Y).

Context:
- Read `{{PLAN_PATH}}` for detailed task specs
- Read `{{DESIGN_PATH}}` for architecture context (if exists)

Rules:
{{PROJECT_RULES}}
- Follow the plan's exact file paths, public APIs, and definitions
- After each task: run the verification command, then commit with the specified message
- If a task is blocked, document the blocker in a comment and skip to the next task

Verification commands:
{{VERIFICATION_COMMANDS}}

When done, output:
- List of completed tasks with commit hashes
- List of any skipped/blocked tasks with reasons
- Final test output
\```

### Dispatching Reviewer

Use the Agent tool to spawn a reviewer subagent. Provide it with:

\```
You are the Code Reviewer for {{PROJECT_NAME}}. Review Batch N (Tasks X-Y).

Context:
- Read `{{DESIGN_PATH}}` for expected APIs (if exists)
- Read `{{PLAN_PATH}}` for task specs

Process:
1. Run: git log --oneline -20 (see batch commits)
2. For each changed file, review for:
   - Correctness (matches plan?)
   - Error handling (errors have context, no silent failures)
   - Security (injection, traversal, auth bypass)
   - API conformance (matches design doc?)
   - YAGNI (no over-engineering?)
   - Test acceptance logic (tolerances/assertions: "what wrong implementation would still pass these tests?")
   - Generated artifacts & doc sync (regenerated code AND its docs reflect the post-fix convention)
   - Coverage domain vs accepted domain (tests cover the full input domain the API accepts)
   - Quantitative claims (numeric precision/perf claims backed by a test or measurement in the diff)
3. Run: {{VERIFICATION_COMMANDS_ONELINE}}
4. Write review report to `docs/reviews/YYYY-MM-DD-batch-N-review.md`
5. Commit the review report

Report format:
- Verdict: APPROVED / APPROVED WITH NOTES / CHANGES REQUESTED
- Findings categorized as Critical / Important / Minor
- Each finding has: [file:line] description
- Verification results
{{EXTRA_REVIEW_SECTIONS}}

Output your verdict and a summary of findings.
\```

### Dispatching Executor for Fixes

If the reviewer returns CHANGES REQUESTED:

\```
You are the Executor for {{PROJECT_NAME}}. Fix the issues from the Batch N code review.

Review report: `docs/reviews/YYYY-MM-DD-batch-N-review.md`

Fix all Critical and Important issues listed in the review. For each fix:
1. Read the finding
2. Open the file, understand the context
3. Fix the issue
4. Run: {{VERIFICATION_COMMANDS_ONELINE}}
5. Commit: "fix(scope): description of fix"

Do NOT fix Minor issues unless trivial. Focus on correctness and security.

When done, output list of fixed issues with commit hashes.
\```

### Dispatching Reviewer for Fix Verification

\```
You are the Code Reviewer for {{PROJECT_NAME}}. Verify that Batch N fixes address the review findings.

Previous review: `docs/reviews/YYYY-MM-DD-batch-N-review.md`

1. Check each Critical/Important finding — is it actually fixed?
2. Run: {{VERIFICATION_COMMANDS_ONELINE}}
3. Append a "Fix Verification" section to the existing review report
4. Update verdict to APPROVED if all Critical/Important issues are resolved
5. Commit updated report

Output your updated verdict.
\```

### Dispatching QA (after all batches)

\```
You are the QA Tester for {{PROJECT_NAME}}. Run a full user-perspective test of all implemented features.

Context:
- Read `{{DESIGN_PATH}}` to understand user journeys (if exists)
- Read `{{PLAN_PATH}}` to know what's implemented

Process:
1. Build and run existing tests: {{BUILD_AND_TEST_COMMAND}}
2. {{QA_TEST_APPROACH}}
3. Test edge cases: {{EDGE_CASES}}
4. Write QA report to `docs/qa/YYYY-MM-DD-full-qa.md`
5. Commit report and any test code

Report format:
- Test results table: Test | Input | Expected | Actual | Status
- Bugs found with reproduction steps
- Each bug has: severity, steps, expected, actual, code location

Output: verdict (PASS/FAIL), number of tests, number of bugs found.
\```

### Dispatching Executor for QA Bug Fixes

If QA returns FAIL (max 2 QA fix cycles), use the Agent tool to spawn an executor subagent:

\```
You are the Executor for {{PROJECT_NAME}}. Fix the bugs from the QA report.

QA report: `docs/qa/YYYY-MM-DD-full-qa.md`

Fix all Critical and High severity bugs. After each fix, run: {{VERIFICATION_COMMANDS_ONELINE}}, then commit: "fix(scope): description of fix".

When done, output list of fixed bugs with commit hashes.
\```

### Dispatching QA for Fix Verification

Use the Agent tool to spawn a QA subagent:

\```
You are the QA Tester for {{PROJECT_NAME}}. Verify that the fixes address the bugs in `docs/qa/YYYY-MM-DD-full-qa.md`.

Re-run the reproduction steps for each reported bug, append a "Fix Verification" section to the report, update the verdict, and commit.

Output your updated verdict (PASS/FAIL).
\```

## Final Audit (after QA passes)

This stage runs in THIS orchestrator session — it is NOT a subagent dispatch. A fresh whole-branch pass catches what per-batch reviews miss (cross-batch interactions, test acceptance logic, stale doc claims).

1. **Run the audit**: invoke the `/code-review` skill (Skill tool) at effort `high`, scoped to the **full feature-branch diff vs the default branch (`main`/`master`)** — not just the last batch. Record this step as `final_audit` in progress.json.
2. **Critical/P0 findings** → dispatch an executor to fix (step `audit_fix`), then re-run the audit (step `audit_verify`; max 2 audit fix cycles). Still failing after 2 cycles → STOP and ask the user.

\```
You are the Executor for {{PROJECT_NAME}}. Fix the Critical findings from the final audit.

Audit findings: [paste the Critical/P0 findings with file:line references]

Fix all Critical findings. After each fix, run: {{VERIFICATION_COMMANDS_ONELINE}}, then commit: "fix(scope): description of fix".

When done, output list of fixed findings with commit hashes.
\```

3. **Non-blocking findings** (High/Medium/Low that don't block merge) → write or append `docs/reviews/YYYY-MM-DD-final-audit.md` with a `## BACKLOG` section listing them, and commit it.

**Fallback** — if the `/code-review` skill is unavailable in this session: spawn a reviewer subagent with a whole-branch audit prompt covering the full diff vs the default branch from multiple finder angles — algorithmic/mathematical correctness, test acceptance logic (tolerances, assertions that can't fail), documentation claims vs actual behavior, API contracts — and require per-finding adversarial verification (try to disprove each finding against the code) before reporting.

**NEVER invoke `/code-review ultra`** — ultra is user-triggered and billed; the local depth cap for this audit is `max`.

## Orchestration Logic

Implement this as a loop:

\```python
for batch in BATCHES:
    # 1. Execute
    executor_result = dispatch_executor(batch)

    # 2. Review
    review_result = dispatch_reviewer(batch)

    # 3. Fix cycle (max 3 iterations)
    attempts = 0
    while review_result.verdict == "CHANGES_REQUESTED" and attempts < 3:
        fix_result = dispatch_executor_fixes(batch, review_result)
        review_result = dispatch_reviewer_verify(batch)
        attempts += 1

    if review_result.verdict == "CHANGES_REQUESTED":
        STOP — ask user for guidance

    # 4. Announce batch complete
    print(f"Batch {batch.number} APPROVED. Moving to next batch.")

# After all batches
qa_result = dispatch_qa()

qa_attempts = 0
while qa_result.verdict == "FAIL" and qa_attempts < 2:
    fix_result = dispatch_executor_fixes_from_qa(qa_result)
    qa_result = dispatch_qa_verify()
    qa_attempts += 1

if qa_result.verdict == "FAIL":
    STOP — ask user for guidance

print("ALL BATCHES COMPLETE + QA PASSED")

# Final audit (runs in THIS session via the /code-review skill)
audit_result = run_code_review_skill(scope="branch", effort="high")      # step: final_audit
audit_attempts = 0
while audit_result.has_critical and audit_attempts < 2:
    dispatch_executor_fixes_from_audit(audit_result)                     # step: audit_fix
    audit_result = run_code_review_skill(scope="branch", effort="high")  # step: audit_verify
    audit_attempts += 1
if audit_result.has_critical:
    STOP — ask user for guidance
write_backlog(audit_result.non_blocking)

print("FINAL AUDIT PASSED — pipeline complete")
\```

## Important Rules

- **Use the Agent tool** to dispatch each role as a subagent
- **Wait for each subagent to complete** before dispatching the next (sequential, not parallel)
- **Read subagent output carefully** to determine next action
- **Max 3 fix cycles per batch** — if review still fails after 3 rounds, stop and ask the user
- **Max 2 QA fix cycles** — if QA still fails after 2 rounds, stop and ask the user
- **Max 2 final-audit fix cycles** — if Critical findings persist after 2 rounds, stop and ask the user
- **NEVER invoke `/code-review ultra`** — it is user-triggered and billed; the local audit depth cap is `max`
- **Do not rely on thinking keywords for effort** — `think`/`think hard` are no-ops in current Claude Code and `ultrathink` is a one-shot user toggle; do not prepend them to dispatch prompts. `ultracode` is a Claude Code session-only setting (xhigh + dynamic workflows), not an effort level — do not use it for roles, and do not run the orchestrator session under it (the coordinator should not spawn its own workflows).
- **Announce progress** between each step so the user can follow along:
  - "Starting Batch 1 execution..."
  - "Batch 1 execution complete. Starting review..."
  - "Review found 2 Critical issues. Dispatching fixes..."
  - "Fixes verified. Batch 1 APPROVED."
- **All subagents run in the SAME repo** — they commit directly to the working branch
- **After each batch approval**, briefly summarize what was built

## Progress Tracking (Self-Healing)

After EVERY step (execute, review, fix, verify, QA, audit), update `docs/sessions/progress.json`:

\```json
{
  "current_batch": 1,
  "current_step": "review",
  "step_detail": "Reviewer dispatched, awaiting result",
  "batches_completed": [],
  "batches_in_progress": {
    "batch": 1,
    "phase": "Phase 1",
    "tasks": "1-5",
    "executor_done": true,
    "executor_commits": ["abc1234", "def5678"],
    "review_verdict": null,
    "review_report": null,
    "fix_attempts": 0,
    "approved": false
  },
  "qa_status": null,
  "audit_status": null,
  "audit_attempts": 0,
  "last_updated": "YYYY-MM-DDTHH:MM:SSZ",
  "notes": "Any context needed for resume"
}
\```

**This file is your memory.** Always read it at session start. Always update it after each step. This enables a new session to resume seamlessly.

### Update rules:
- `current_step` is one of: `execute`, `review`, `fix`, `fix_verify`, `qa`, `qa_fix`, `qa_verify`, `final_audit`, `audit_fix`, `audit_verify`, `done`
- After batch approval: move batch from `batches_in_progress` to `batches_completed`, increment `current_batch`
- After QA pass: set `qa_status` to `"PASS"`
- Record the initial audit as `final_audit`, executor fixes as `audit_fix`, and each re-audit as `audit_verify`. Increment `audit_attempts` on each `audit_fix` cycle (it persists the 2-cycle cap across interruptions). After the audit passes: set `audit_status` to `"PASS"`
- Include commit hashes in `executor_commits` so reviewer knows what to diff

## Start

1. Check if `docs/sessions/progress.json` exists
   - If YES: read it and **resume from where it left off** (this is a self-healing restart)
   - If NO: create it, start fresh with Batch 1
2. Read the implementation plan
3. Begin execution. Announce each step as you go.
```
