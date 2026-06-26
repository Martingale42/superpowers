# Upstream Divergence — Factual Differences

**Date:** 2026-06-10
**Author:** automated review (Claude)
**This report:** documents *what* differs between our fork and upstream. The
companion report `2026-06-10-upstream-update-recommendation.md` covers *whether to update*.

## Scope & method

- **Remotes:** `origin` = `Martingale42/superpowers` (our fork), `upstream` = `obra/superpowers`.
- **`upstream` fetched:** 2026-06-10 (`git fetch upstream --prune`).
- **Comparison baseline:** `origin/main` (the fork's published state) vs `upstream/main`.
  The 6 local commits adding *per-role model/effort designation* (in
  `skills/orchestrator-driven-development/` + `docs/`) are deliberately **excluded** from
  this comparison so they don't distort "what upstream brings." They touch only
  fork-only files and have no bearing on the merge.
- **Merge base:** `e4a2375` (`Merge pull request #524`). The histories split there.

## Headline numbers

| Metric | Value |
|--------|-------|
| Commits upstream has, we don't | **155** |
| Commits we have, upstream doesn't | **2** |
| File delta (`origin/main` → `upstream/main`) | **128 files, +12,733 / −3,229** |
| Our version | `v4.3.1` (`v4.3.1-4-g8ade195`) |
| Upstream version | **`v5.1.0`** (a full major version ahead) |
| Upstream commit types | fix ×18, docs ×10, feat ×5, chore ×3, refactor ×2, test ×1, plus releases/merges/misc |

## A. What OUR fork has that upstream lacks (2 commits)

Both are local additions; upstream never received them.

### A1. `orchestrator-driven-development` skill — `8ade195`
- 7 new files (`SKILL.md` + 6 templates), +660 lines. Generates a multi-agent
  orchestration session (executor/reviewer/QA pipeline).
- Upstream has **no** equivalent skill. Confirmed: `skills/orchestrator-driven-development`
  does not exist in `upstream/main`.
- Also edited `skills/writing-plans/SKILL.md` (+12) to add the orchestrator hand-off option.
- **Now extended** by the 6 local commits for per-role model/effort (see the design/impl
  docs under `docs/superpowers/plans/2026-06-10-orchestrator-per-role-model-*.md`).

### A2. "Replace TDD with pragmatic-testing" — `adde5d4`
Despite the subject, this is a **broad philosophy rewrite**, not a rename. It touched 9
files (+794 / −1,648):

| File | Change | Note |
|------|--------|------|
| `skills/pragmatic-testing/SKILL.md` | **+186 (new skill)** | replaces the TDD mandate with context-appropriate testing |
| `skills/test-driven-development/SKILL.md` | **−371 (deleted)** | |
| `skills/test-driven-development/testing-anti-patterns.md` | **−299 (deleted)** | |
| `skills/writing-skills/SKILL.md` | 680 lines reworked (net trim) | |
| `skills/systematic-debugging/SKILL.md` | 375 lines reworked | |
| `skills/subagent-driven-development/SKILL.md` | 268 lines reworked | |
| `skills/subagent-driven-development/implementer-prompt.md` | 25 lines | |
| `skills/writing-plans/SKILL.md` | +175 | pragmatic-testing judgment table |
| `README.md` | 63 lines | |

This is the source of most merge friction (Section D), because upstream independently
rewrote several of these same files for v5.

**Net skill-set delta** (directory level):
- Only in our fork: `orchestrator-driven-development`, `pragmatic-testing`
- Only in upstream: `test-driven-development` (we intentionally removed it)
- Upstream added **no brand-new skill directories** vs our fork — its value is in
  *enhancements* to existing skills + tooling + multi-harness support.

## B. What UPSTREAM has that we lack (the 155 commits)

Grouped by theme. None of these exist in our fork.

### B1. Multi-harness support (largest theme)
- **OpenCode** — native skills, one-line install, bootstrap content caching (15 regression
  tests), ESM fixes, path fixes.
- **OpenAI Codex plugin mirror** — `sync-to-codex-plugin` tooling that mirrors superpowers
  into the Codex plugin marketplace; committed Codex plugin manifest; named-agent dispatch docs.
- **Copilot CLI** — tool mapping, platform detection, install instructions (v5.0.7).
- **Gemini CLI** — work-in-progress on upstream branches (not yet in `main`).
- (Cursor support already landed in our v4.3.1.)

### B2. Brainstorming overhaul — **+1,275 lines across 8 files**
The single biggest skill change. "Brainstorming-skill-scaling" plus a **zero-dependency
brainstorm server** (HTTP + WebSocket `server.js`, file watching, 30-min idle auto-exit,
liveness check), replacing vendored `node_modules`. We merge this **cleanly** (we never
touched brainstorming).

### B3. Skill enhancements (clean adopts — we didn't touch these)
- `using-superpowers` (+174): context-isolation principle, steering improvements.
- `requesting-code-review` consolidation: persona/checklist/dispatch unified into
  `code-reviewer.md`; dispatches `Task (general-purpose)` instead of a named agent.
- `using-git-worktrees` + `finishing-a-development-branch` **rewrite**: native-worktree
  detection (`GIT_DIR != GIT_COMMON`), consent before creating worktrees, provenance-based
  cleanup, detached-HEAD handling. (PRI-974)

### B4. Removals (v5.1.0)
- Legacy slash commands `/brainstorm`, `/execute-plan`, `/write-plan` removed (were deprecated stubs).
- `superpowers:code-reviewer` **named agent removed** — folded into `requesting-code-review`.
- "Integration sections" stripped from skills (pre-native-skills legacy).

### B5. Infrastructure / governance
- **Contributor guidelines for AI agents** in `CLAUDE.md`/`AGENTS.md` (after a 94%
  AI-slop PR rejection audit); PR template, issue templates, Code of Conduct.
- New-harness PRs now require a session transcript + auto-trigger acceptance test.
- `bump-version.sh`, `.version-bump.json`, release-notes automation.

### B6. Bug fixes (18 `fix:` commits + more)
- Windows owner-PID lifecycle (false positives across users), cross-platform PID monitoring.
- OpenCode bootstrap caching, ESM, install path.
- Subagent-driven-development integration test (3 independent bugs that silently bailed).
- Subagent context isolation across all delegation skills (v5.0.2).
- Copilot/Codex sessionStart context injection; bash 5.3+ fix.

## C. Version timeline (upstream, since our split)

| Release | Highlights |
|---------|-----------|
| v5.0.2 | Subagent context isolation; zero-dep brainstorm server |
| v5.0.4 | Review-loop refinements; OpenCode one-line install |
| v5.0.5 | Brainstorm server ESM fix; Windows PID fix |
| v5.0.6 | Inline self-review; brainstorm server restructure; owner-PID fixes |
| v5.0.7 | Copilot CLI support; OpenCode fixes |
| **v5.1.0** | Legacy slash commands removed; code-reviewer agent removed; worktree skills rewrite; contributor guidelines; Codex plugin mirror; code-review consolidation; SDD fixes |

## D. Conflict surface (authoritative)

An **isolated trial merge** (`git merge --no-commit --no-ff upstream/main` in a throwaway
worktree off `origin/main`, then aborted) produced **exactly 5 content conflicts**:

| Conflicting file | Why | Our side | Upstream side |
|------------------|-----|----------|---------------|
| `README.md` | both rewrote | `adde5d4` (63) | v5 README updates (+123/−98) |
| `skills/writing-skills/SKILL.md` | both rewrote | `adde5d4` (680) | v5 (+572/−112) |
| `skills/writing-plans/SKILL.md` | both rewrote | `adde5d4` (+175) + `8ade195` (+12) | v5 (+119/−157) |
| `skills/subagent-driven-development/SKILL.md` | both rewrote | `adde5d4` (268) | v5 (+198/−146) |
| `skills/subagent-driven-development/implementer-prompt.md` | both touched | `adde5d4` (25) | v5 |

**Auto-resolved (no manual work):**
- `skills/systematic-debugging/SKILL.md` — both modified, but git merged non-overlapping hunks cleanly.
- `skills/test-driven-development/*` — **no delete/modify conflict**: upstream never modified
  TDD after the merge base, so our deletion applies cleanly.
- `brainstorming` (+1,275), `using-superpowers` (+174), `requesting-code-review`, all
  worktree-skill rewrites, all Codex/OpenCode/Windows tooling — merge cleanly.
- `orchestrator-driven-development` and `pragmatic-testing` — fork-only, untouched by the merge.

**Bottom line:** updating is a 5-file manual resolution, all concentrated in the skills our
`adde5d4` "pragmatic-testing" commit rewrote. Everything else integrates automatically.
