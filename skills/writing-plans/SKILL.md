---
name: writing-plans
description: Use when you have a spec or requirements for a multi-step task, before touching code
---

# Writing Plans

## Overview

Write comprehensive implementation plans assuming the engineer has zero context for our codebase. Document everything they need: which files to touch, what to implement, how to verify. Give them the whole plan as bite-sized tasks. DRY. YAGNI. Frequent commits.

Assume they are a skilled developer, but know almost nothing about our toolset or problem domain.

**Announce at start:** "I'm using the writing-plans skill to create the implementation plan."

**Context:** This should be run in a dedicated worktree (created by brainstorming skill).

**Save plans to:** `docs/plans/YYYY-MM-DD-<feature-name>.md`

## Bite-Sized Task Granularity

**Each task is a focused unit of work (5-15 minutes):**
- Implement one logical component
- Add tests for critical paths (when valuable)
- Verify the implementation works
- Commit

**NOT rigid TDD steps.** Tasks focus on delivering working functionality with appropriate verification.

## Plan Document Header

**Every plan MUST start with this header:**

```markdown
# [Feature Name] Implementation Plan

> **For Claude:** Use superpowers:executing-plans or superpowers:subagent-driven-development to implement this plan.

**Goal:** [One sentence describing what this builds]

**Architecture:** [2-3 sentences about approach]

**Tech Stack:** [Key technologies/libraries]

---
```

## Task Structure

```markdown
### Task N: [Component Name]

**Files:**
- Create: `exact/path/to/file.py`
- Modify: `exact/path/to/existing.py`
- Test: `tests/exact/path/to/test.py` (if tests add value)

**Implementation:**

[Clear description of what to implement]

```python
# Key code or pseudocode showing the approach
def function(input):
    return expected
```

**Verification:**

Run: `[exact command]`
Expected: [what success looks like]

**Tests (if applicable):**

```python
def test_specific_behavior():
    result = function(input)
    assert result == expected
```

**Commit:**

```bash
git add [files]
git commit -m "feat: [description]"
```
```

## When to Include Tests in Plan

Apply pragmatic-testing judgment when planning:

| Situation | Include Tests? |
|-----------|---------------|
| Public API | ✅ Yes |
| Core business logic | ✅ Yes |
| Internal helpers | ⏳ Optional |
| Exploratory/uncertain design | ❌ Defer |
| FFI/memory safety critical | ✅ Yes |

**In the plan, note:**
```markdown
**Tests:** Required for public API
```
or
```markdown
**Tests:** Deferred - exploratory implementation, add when design stabilizes
```

## Task Types

### Implementation Task (most common)

```markdown
### Task 3: Add Order Validation

**Files:**
- Create: `src/validation/order.py`
- Test: `tests/validation/test_order.py`

**Implementation:**

Validate orders before submission:
- Check quantity > 0
- Check price > 0 for limit orders
- Verify instrument exists

```python
def validate_order(order: Order) -> ValidationResult:
    errors = []
    if order.quantity <= 0:
        errors.append("Quantity must be positive")
    # ... more validation
    return ValidationResult(valid=len(errors) == 0, errors=errors)
```

**Verification:**

Run: `pytest tests/validation/test_order.py -v`
Expected: All tests pass

**Commit:**
```bash
git commit -m "feat: add order validation"
```
```

### Integration Task

```markdown
### Task 5: Connect Parser to Data Pipeline

**Files:**
- Modify: `src/pipeline/data_handler.py`

**Implementation:**

Wire the new parser into the existing pipeline:
1. Import the parser
2. Replace the old parsing call
3. Handle the new return type

**Verification:**

Run: `python -m pytest tests/integration/ -v`
Expected: Integration tests pass

Manual check: Feed sample data through pipeline, verify output format.

**Commit:**
```bash
git commit -m "feat: integrate new parser into pipeline"
```
```

### Refactoring Task

```markdown
### Task 7: Extract Common Validation Logic

**Files:**
- Create: `src/validation/common.py`
- Modify: `src/validation/order.py`
- Modify: `src/validation/position.py`

**Implementation:**

Extract duplicate validation helpers to shared module:
- `validate_positive_number(value, field_name)`
- `validate_not_empty(value, field_name)`

**Verification:**

Run: `pytest tests/validation/ -v`
Expected: All existing tests still pass

**Commit:**
```bash
git commit -m "refactor: extract common validation helpers"
```
```

## Remember

- Exact file paths always
- Clear implementation description (code when helpful)
- Exact verification commands with expected output
- Tests when they add value (not as ritual)
- DRY, YAGNI, frequent commits
- Reference relevant skills with explicit markers

## Execution Handoff

After saving the plan, offer execution choice:

**"Plan complete and saved to `docs/plans/<filename>.md`. Two execution options:**

**1. Subagent-Driven (this session)** — I dispatch fresh subagent per task, review between tasks, fast iteration

**2. Parallel Session (separate)** — Open new session with executing-plans, batch execution with checkpoints

**Which approach?"**

**If Subagent-Driven chosen:**
- **REQUIRED SUB-SKILL:** Use superpowers:subagent-driven-development
- Stay in this session
- Fresh subagent per task + code review

**If Parallel Session chosen:**
- Guide them to open new session in worktree
- **REQUIRED SUB-SKILL:** New session uses superpowers:executing-plans
