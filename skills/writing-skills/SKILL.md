---
name: writing-skills
description: Use when creating new skills, editing existing skills, or verifying skills work before deployment
---

# Writing Skills

## Overview

**Writing skills applies iterative refinement to process documentation.**

**Personal skills live in agent-specific directories (`~/.claude/skills` for Claude Code, `~/.codex/skills` for Codex)**

You identify failure scenarios, establish baseline behavior, write the skill (documentation), verify improvement, and refine to close loopholes.

**Core principle:** If you didn't observe an agent struggle without the skill, you don't know if the skill teaches the right thing.

**REQUIRED BACKGROUND:** You MUST understand pragmatic-testing before using this skill. That skill defines when and how to validate—apply the same judgment here: test skills when tests add value, not as ritual.

**Official guidance:** For Anthropic's official skill authoring best practices, see anthropic-best-practices.md. This document provides additional patterns and guidelines.

## What is a Skill?

A **skill** is a reference guide for proven techniques, patterns, or tools. Skills help future Claude instances find and apply effective approaches.

**Skills are:** Reusable techniques, patterns, tools, reference guides

**Skills are NOT:** Narratives about how you solved a problem once

## Skill Development Cycle

| Phase | Skill Creation |
|-------|----------------|
| **Identify need** | Observe agent struggling with a task type |
| **Baseline** | Note how agent fails without guidance |
| **Draft skill** | Write documentation addressing observed failures |
| **Validate** | Test with representative scenarios |
| **Refine** | Close loopholes, improve clarity |
| **Deploy** | Make available for use |

## When to Test Skills

Apply pragmatic-testing principles to skill validation:

**MUST validate (non-negotiable):**
- Discipline-enforcing skills (rules that must be followed)
- Skills with complex decision trees
- Skills that override default agent behavior

**CAN defer validation:**
- Simple reference documentation
- Technique guides with clear examples
- Skills derived from well-tested processes

**Validation approaches:**
- Run representative scenarios with subagents
- Academic review for clarity and completeness
- Real-world usage observation and iteration

## Testing Different Skill Types

### Discipline-Enforcing Skills (rules/requirements)

**Examples:** verification-before-completion, designing-before-coding

**Validate with:**
- Pressure scenarios: Does agent comply under time stress?
- Edge cases: Does agent know when rules apply?
- Identify rationalizations and add explicit counters

**Success criteria:** Agent follows rule when it matters

### Technique Skills (how-to guides)

**Examples:** condition-based-waiting, root-cause-tracing

**Validate with:**
- Application scenarios: Can agent apply correctly?
- Variation scenarios: Does agent handle edge cases?
- Gap testing: Are instructions complete?

**Success criteria:** Agent successfully applies technique

### Pattern Skills (mental models)

**Examples:** reducing-complexity, information-hiding

**Validate with:**
- Recognition scenarios: Does agent know when pattern applies?
- Counter-examples: Does agent know when NOT to apply?

**Success criteria:** Agent correctly identifies when/how to apply

### Reference Skills (documentation/APIs)

**Examples:** API docs, command references

**Validate with:**
- Retrieval scenarios: Can agent find information?
- Application scenarios: Can agent use what's found?

**Success criteria:** Agent finds and correctly applies reference

## SKILL.md Structure

### Frontmatter (Required)

```yaml
---
name: skill-name
description: "Use when [specific trigger conditions]. [What it provides]."
---
```

**Description requirements:**
- Must describe WHEN to use (trigger conditions)
- Include keywords for Claude Search Optimization (CSO)
- Do NOT summarize the workflow (that's for the body)

### Body Structure

1. **Overview** — What this skill does (2-3 sentences)
2. **Core workflow** — Steps or decision process
3. **Key principles** — Non-obvious guidance
4. **Integration** — How it connects to other skills
5. **Red flags** — What to avoid

## File Organization

### Self-Contained Skill
```
defense-in-depth/
  SKILL.md    # Everything inline
```
When: All content fits, no heavy reference needed

### Skill with Reference Material
```
systematic-debugging/
  SKILL.md           # Overview + workflow
  root-cause-tracing.md    # Detailed technique
  defense-in-depth.md      # Detailed technique
```
When: Reference material too large for inline (>100 lines)

### Skill with Tools
```
pptx/
  SKILL.md       # Overview + workflows
  scripts/       # Executable tools
  references/    # Heavy documentation
```
When: Includes reusable scripts or extensive reference

## Cross-Referencing Other Skills

**Use explicit requirement markers:**
- ✅ Good: `**REQUIRED SUB-SKILL:** Use superpowers:systematic-debugging`
- ✅ Good: `**REQUIRED BACKGROUND:** You MUST understand pragmatic-testing`
- ❌ Bad: `See skills/testing/...` (unclear if required)
- ❌ Bad: `@skills/...` (force-loads, burns context)

**Why no @ links:** `@` syntax force-loads files immediately, consuming context before needed.

## Quality Checklist

Before deploying a skill:

- [ ] Description includes clear trigger conditions
- [ ] Description includes relevant keywords for CSO
- [ ] Body is under 500 lines (split if larger)
- [ ] Examples are concrete, not abstract
- [ ] Terminology is consistent throughout
- [ ] Cross-references use explicit markers
- [ ] Validated with representative scenarios (if discipline-enforcing)

## Common Rationalizations for Skipping Validation

| Excuse | Reality |
|--------|---------|
| "Skill is obviously clear" | Clear to you ≠ clear to agents. Test if it's discipline-enforcing. |
| "It's just a reference" | References can have gaps. Do a retrieval test. |
| "No time to test" | For reference skills, deploy and iterate. For discipline skills, validate first. |
| "I'm confident it's good" | Apply pragmatic judgment—critical skills need validation. |

## Integration

**Complementary skills:**
- **pragmatic-testing** — Validation philosophy (REQUIRED BACKGROUND)
- **brainstorming** — For designing new skills collaboratively

**This skill does NOT require:**
- Rigid test-first methodology
- 100% scenario coverage
- Formal test infrastructure
