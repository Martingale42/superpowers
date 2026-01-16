---
name: pragmatic-testing
description: "Use when writing tests or discussing testing strategy. Provides context-appropriate testing guidance that prioritizes validation over ritual. Replaces rigid TDD with pragmatic approaches suited to exploratory development, trading systems, and rapid iteration cycles."
---

# Pragmatic Testing

## Overview

Testing validates correctness; it is not a design methodology.

**Core principle:** Write tests when they add value, not as a ritual gate before implementation.

**Announce at start:** "I'm using the pragmatic-testing skill to guide our testing approach."

## When to Write Tests

```
MUST have tests (non-negotiable):
┌─────────────────────────────────────────────────────────────┐
│ ✅ Public APIs that other modules depend on                 │
│ ✅ Core business logic (order processing, position calc)    │
│ ✅ FFI boundaries (memory safety verification)              │
│ ✅ Bug fixes (regression test proving the fix)              │
│ ✅ Security-critical code paths                             │
└─────────────────────────────────────────────────────────────┘

CAN defer tests (add after stabilization):
┌─────────────────────────────────────────────────────────────┐
│ ⏳ Exploratory prototypes and spikes                        │
│ ⏳ Internal helper functions                                │
│ ⏳ Experimental features under active iteration             │
│ ⏳ Code that will likely change significantly               │
└─────────────────────────────────────────────────────────────┘

SKIP tests (low value):
┌─────────────────────────────────────────────────────────────┐
│ ❌ Pure wrapper/delegate functions                          │
│ ❌ Config classes without validation logic                  │
│ ❌ Trivial getters/setters                                  │
│ ❌ Code paths impossible to reach without modifying prod    │
└─────────────────────────────────────────────────────────────┘
```

## The Pragmatic Cycle

Unlike TDD's rigid RED-GREEN-REFACTOR, use context-appropriate approaches:

### For Exploratory Development

```
1. SPIKE: Implement a working prototype
2. EVALUATE: Does the approach work? Is it worth keeping?
3. STABILIZE: If keeping, add tests for critical paths
4. REFACTOR: Improve design with test safety net
```

### For Known Requirements

```
1. IMPLEMENT: Build the feature
2. IDENTIFY: What are the critical paths and edge cases?
3. TEST: Write tests covering those paths
4. VERIFY: Ensure tests catch real bugs, not just coverage
```

### For Bug Fixes

```
1. UNDERSTAND: Reproduce and understand the root cause
2. FIX: Implement the correction
3. VERIFY: Manually confirm the fix works
4. PROTECT: Add regression test to prevent recurrence
```

## Test Value Assessment

Before writing a test, ask:

| Question | If NO, then... |
|----------|----------------|
| Will this test catch real bugs? | Skip or simplify |
| Will this test survive refactoring? | Make it less brittle |
| Does this test document behavior? | Consider if docs suffice |
| Is this logic complex enough to warrant testing? | Maybe skip |

## Testing Strategies by Context

### Trading Systems

| Component | Strategy | Rationale |
|-----------|----------|-----------|
| Order logic | Comprehensive unit tests | Core correctness |
| Position calculations | Property-based tests | Mathematical invariants |
| Market data parsing | Golden file tests | Venue format stability |
| Strategy signals | Backtest validation | Real market behavior |
| Risk checks | Boundary tests | Safety critical |

### Rust/Python FFI

| Concern | Test Type | Example |
|---------|-----------|---------|
| Memory leaks | Valgrind/ASAN | Run under sanitizers |
| Double-free | Lifecycle tests | Create → use → drop once |
| Reference cycles | GC verification | Check Python GC can collect |
| Panic across FFI | Abort verification | Ensure no unwinding |

### Adapters (Exchange Integration)

| Layer | Strategy |
|-------|----------|
| HTTP client | Mock server responses |
| WebSocket | Recorded message playback |
| Parsing | Golden files from real venue |
| Order flow | Integration test with testnet |

## Anti-Patterns to Avoid

| Anti-Pattern | Problem | Better Approach |
|--------------|---------|-----------------|
| Testing mocks, not behavior | Verifies nothing real | Use real collaborators when possible |
| 100% coverage goal | Tests low-value code | Focus on critical paths |
| Test-per-method | Brittle to refactoring | Test behaviors, not implementations |
| Complex test setup | Hard to maintain | Simplify or use fixtures |
| Testing implementation details | Breaks on refactor | Test observable behavior |

## When Someone Says "Write Tests First"

This skill explicitly permits implementation before tests when:

1. **Exploring a new problem space** — You don't know what the right abstraction is yet
2. **Rapid prototyping** — Speed of iteration matters more than safety
3. **Learning a new API/library** — Understanding comes before testing
4. **The design is uncertain** — Tests would just be rewritten anyway

After the code stabilizes, add tests for the parts worth keeping.

## Integration with Other Skills

**When superpowers:writing-plans generates TDD steps:**
- Reinterpret "Write failing test first" as "Implement, then test critical paths"
- Keep the verification steps, but test after implementation

**When superpowers:systematic-debugging suggests "Write failing test first":**
- Optional step, not mandatory
- Prioritize understanding root cause first
- Add regression test after fix is confirmed working

**When superpowers:subagent-driven-development dispatches implementers:**
- Subagents should follow pragmatic-testing, not TDD
- Tests are expected for public APIs and critical paths
- Exploratory tasks may defer testing

## Verification Checklist

Before marking implementation complete:

- [ ] Critical paths have tests (or documented reason for deferral)
- [ ] Edge cases identified and either tested or acknowledged
- [ ] No obvious bugs when running manually
- [ ] If tests exist, they all pass
- [ ] If performance-critical, benchmarks exist or planned

## Red Flags

**Never:**
- Skip tests for public APIs that others will depend on
- Leave known bugs without regression tests
- Ignore FFI memory safety testing
- Claim "tests deferred" without a plan to add them

**Always:**
- Test bug fixes with regression tests
- Test core business logic
- Document why tests are deferred when they are
- Add tests when code stabilizes

## The Bottom Line

```
Tests serve the code, not the other way around.

Write tests when they provide value.
Defer tests when exploration is more valuable.
Never skip tests for code that must be correct.
```
