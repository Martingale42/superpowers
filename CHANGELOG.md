# Changelog

Fork-specific additions for `Martingale42/superpowers` on top of upstream
`obra/superpowers`. **This branch (`main`) tracks upstream v5.1.0** and adds only the
orchestrator workflow. The pragmatic-testing customizations and the Claude-specific
per-role model/effort feature live on the `non-tdd` branch, which keeps its own
changelog. Upstream releases are tracked in `RELEASE-NOTES.md`.

Format loosely follows [Keep a Changelog](https://keepachangelog.com/).

## [Unreleased] — 2026-06-10

`main` was realigned from the previous fork state onto upstream v5.1.0, re-applying
the orchestrator workflow as a clean, harness-neutral addition.

### Added

- **`orchestrator-driven-development` skill** — generates session files for a
  separate, resumable, multi-agent execution pipeline (executor → reviewer → QA)
  with per-step state tracked in `progress.json` (self-healing resume via
  `resume.md`). Clean, harness-neutral version; the Claude-specific per-role
  model/effort feature is intentionally **not** here (it lives on `non-tdd`).
- **writing-plans execution-handoff wiring** — `skills/writing-plans/SKILL.md` now
  lists `orchestrator-driven-development` in the plan header and offers it as a
  **third execution option** (after Subagent-Driven and Inline Execution), with an
  "If Orchestrator chosen" block. Appended to upstream v5.1.0's two-option structure
  rather than replacing it.

### Notes

- The handoff wiring was initially **dropped** when `main` was reset onto upstream
  (only the skill's own files were carried over, so the skill existed but was never
  offered after planning). It was traced back to a single file —
  `skills/writing-plans/SKILL.md` — and re-applied. This is the only cross-skill
  edit needed for the orchestrator to be discoverable.
- At upstream-PR time this wiring needs care: upstream v5.1.0 removed "integration
  sections" from skills and rejects bundled edits to tuned skills, so discoverability
  would have to be solved a different way than re-adding the handoff block.
- `non-tdd` preserves the full prior fork (pragmatic-testing rewrite, per-role
  model/effort feature, upstream-divergence review reports) and its own CHANGELOG.
