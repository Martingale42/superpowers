# Upstream Update — Recommendation & Analysis

**Date:** 2026-06-10
**Author:** automated review (Claude)
**Companion:** factual differences in `2026-06-10-upstream-divergence-differences.md`.
**Question:** should we update our fork (`Martingale42/superpowers`) to upstream
(`obra/superpowers`) v5.1.0, and if so, how?

---

## TL;DR

**Recommendation: YES — update now, via a one-time merge of `upstream/main` into `main`.**

The cost is small and bounded (**5 files to resolve by hand**, all in the blast radius of
our own `adde5d4` "pragmatic-testing" rewrite). The benefit is large and compounding:
multi-harness support (OpenCode / Codex / Copilot), a major brainstorming overhaul,
18+ bug fixes, subagent context isolation, and the v5.1.0 worktree-skills rewrite. We are a
**full major version behind** and the gap widens every release.

Confidence is high because the merge was **empirically trial-run** (isolated worktree, then
aborted): 5 conflicts, everything else auto-merges — including our two fork-only skills
(`orchestrator-driven-development`, `pragmatic-testing`), which the merge does not touch.

---

## The real decision underneath

Our fork is not a few cosmetic tweaks — it is a **testing-philosophy fork**. `adde5d4`
replaced upstream's TDD mandate with `pragmatic-testing` and rewrote the surrounding skills
(`writing-skills`, `writing-plans`, `subagent-driven-development`, `systematic-debugging`,
`README`) to match. **All 5 merge conflicts fall inside exactly that rewrite.** Meanwhile
upstream has *doubled down* on TDD (v5.1.0 worktree skills are "TDD-validated"; it kept
`test-driven-development`).

So "update" really asks two things:
1. **Tactical:** take upstream's 155 commits of improvements. (Clear yes.)
2. **Strategic:** keep carrying the pragmatic-testing overlay against an upstream moving the
   opposite direction. Upstream's contributor guidelines explicitly **won't accept
   "compliance rewrites of skill content"**, so our philosophy change is a **permanent fork
   divergence** — a recurring reconciliation tax at every future sync, not a one-time cost.

This report recommends proceeding on (1) now and making (2) a conscious, documented choice.

## Benefits of updating

| Benefit | Why it matters |
|---------|----------------|
| **Multi-harness support** | OpenCode, Codex plugin mirror, Copilot CLI. If any teammate/tooling uses these, we're currently blind to them. |
| **Brainstorming overhaul (+1,275)** | The skill we rely on most for design work; zero-dep server replaces vendored `node_modules` (smaller, safer). Merges cleanly. |
| **18+ bug fixes** | Windows owner-PID false positives, OpenCode caching/ESM, the SDD integration test that silently passed while broken, bash 5.3+. |
| **Subagent context isolation** | Applied to all delegation skills in v5.0.2 — directly relevant to our orchestrator skill's dispatch model. |
| **Worktree-skills rewrite** | Consent-before-create, native-tool preference, provenance-based cleanup — fixes real data-loss-adjacent bugs (#940, #999). |
| **Governance** | Contributor guidelines, PR/issue templates, CoC — useful if the fork ever takes external contributions. |

## Costs & risks

| Cost / risk | Severity | Notes |
|-------------|----------|-------|
| Resolve 5 conflicting files | **Low–Medium** | Bounded and known (see policy below). Semantically meaningful (testing philosophy), not trivial whitespace. |
| Ongoing pragmatic-testing reconciliation | **Medium (recurring)** | Every future upstream sync will re-conflict on testing-related skills. This is the structural tax of the philosophy fork. |
| Re-validate `orchestrator-driven-development` | **Low** | Our fork-only skill must adopt v5 conventions: dispatch `general-purpose` (it already does, via the Agent tool), and the context-isolation principle. The per-role model/effort work just landed — re-check after merge. |
| Re-apply fork features lost in conflict resolution | **Low** | The orchestrator hand-off option we added to `writing-plans` and the pragmatic-testing judgment table must survive the resolution. |
| History rewrite if rebasing | **Medium** | Avoided by choosing *merge* over *rebase* (see options). |

## Options considered

1. **Merge `upstream/main` → `main`  ✅ recommended.**
   One reconciliation commit, 5 conflicts, fork features preserved, no history rewrite.
   `origin/main` fast-forwards for collaborators.
2. **Rebase our 2 commits onto `upstream/main`.**
   Linear history, but replays our large rewrites onto a new base (same 5 conflicts as
   *rebase* conflicts) **and rewrites already-published history** → force-push required.
   More disruptive for no real gain here.
3. **Cherry-pick selectively** (take brainstorming + harness + fixes, skip conflicting files).
   Lowest immediate risk, but leaves us permanently and *invisibly* diverged, and is fiddly
   to track commit-by-commit across 155 commits. Reasonable only if we decide to drop the
   pragmatic-testing fork and re-derive it.
4. **Stay on v4.3.1 (do nothing).**
   Not recommended. Drift compounds; we miss bug fixes and all multi-harness work; each
   future release makes the eventual merge harder.

## Recommended resolution policy (per conflicting file)

Default stance: **upstream wins on structure and core-infra skills; we re-apply only our
deliberate fork features on top.**

| File | Resolution |
|------|-----------|
| `README.md` | Take upstream's v5 README; re-add our fork-specific notes (orchestrator skill, pragmatic-testing stance). |
| `skills/writing-skills/SKILL.md` | **Adopt upstream's v5 version.** It's actively-maintained core infra; our `adde5d4` mostly *trimmed* it. Only keep a fork change if it was load-bearing — verify before discarding. |
| `skills/writing-plans/SKILL.md` | Base on upstream v5; **re-apply two fork features**: (a) our pragmatic-testing judgment table, (b) the orchestrator hand-off execution option (`8ade195`). |
| `skills/subagent-driven-development/SKILL.md` | **Take upstream's v5** (it carries real fixes: "no pause every 3 tasks", context isolation, test repair); re-apply our pragmatic-testing references on top. |
| `skills/subagent-driven-development/implementer-prompt.md` | Same as above — upstream base, re-apply pragmatic-testing wording. |

**Post-merge follow-ups (not conflicts, but required for consistency):**
- Re-wire any upstream skill that references `test-driven-development` to reference
  `pragmatic-testing` (or consciously keep both). Grep after merge:
  `grep -rl "test-driven-development" skills/ | grep -v test-driven-development/`.
- Re-validate `orchestrator-driven-development` (incl. the new per-role model/effort work)
  against v5.1.0: confirm it dispatches `general-purpose`, honors the context-isolation
  principle, and that `writing-plans`'s orchestrator option still points at it.
- Decide the strategic question: keep the pragmatic-testing fork long-term, or converge to
  upstream TDD. Document the choice so the next sync isn't a surprise.

## Suggested execution sequence

```bash
# 0. Safety: branch off current main first
git checkout main
git switch -c chore/sync-upstream-v5.1.0

# 1. Merge upstream (expect 5 conflicts)
git merge --no-ff upstream/main

# 2. Resolve the 5 files per the policy table above, then:
git add -A && git commit        # completes the merge

# 3. Consistency pass
grep -rl "test-driven-development" skills/ | grep -v 'test-driven-development/'   # rewire refs
#    re-validate orchestrator-driven-development + writing-plans hand-off

# 4. Verify nothing fork-only was lost
test -d skills/orchestrator-driven-development && test -d skills/pragmatic-testing && echo "fork skills intact"

# 5. Run upstream's tests if applicable, then open a PR fork main <- this branch for review
```

Do this on a dedicated branch (step 0) so the reconciliation is reviewable before it lands
on `main`, and so an abort costs nothing.

## Verdict

**Update. Merge `upstream/main` now on a sync branch, resolve the 5 known files with
upstream-wins-on-structure, and re-apply the handful of fork features.** Then make a
deliberate call on whether the pragmatic-testing philosophy fork is worth its recurring
reconciliation tax — that, not the merge mechanics, is the real long-term decision.
