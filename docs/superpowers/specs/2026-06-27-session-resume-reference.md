# Session Resume Reference — orchestrator dispatch + docs unification

> 用途：記錄如何「連回」產出本批工作的 Claude Code session，方便日後在相同脈絡下接續做
> 新的 design 或改動，而不必從頭重建上下文。

## 連回本 session 的指令

Claude Code 的 session 是**綁定專案目錄**的。連回前務必先進到本專案根目錄：

```bash
cd /home/cy/Code/LLMDev/skills/superpowers
```

然後擇一執行：

```bash
# 1) 直接連回「本 session」（用 session id 精準指定）
claude --resume 01HdzAyC8W76qXFkYp4imn1C

# 2) 互動式挑選最近的 session（在清單中找這一條）
claude --resume          # 或 claude -r

# 3) 接續本目錄「最近一次」的 session（若這是最後操作的 session）
claude --continue        # 或 claude -c
```

- **Session id：** `01HdzAyC8W76qXFkYp4imn1C`
- **Session URL：** <https://claude.ai/code/session_01HdzAyC8W76qXFkYp4imn1C>
- **專案目錄：** `/home/cy/Code/LLMDev/skills/superpowers`

> 註：`--resume <id>` 直接跳回指定 session；`-r` 不帶 id 會列出近期 session 供挑選；
> `-c` 接續本目錄最近一次對話。三者都需在上面的專案目錄下執行。

## 連回時的狀態快照（2026-06-27）

- **分支：** `feat/orchestrator-default-dispatch-and-docs-unification`（領先 `main` 10 個 commit；
  完成後若已合併或捨棄，請改連回 `main` 或當時的工作分支）。
- **這批工作做了什麼：**
  1. 把 orchestrator 的派發機制改為「Agent 工具 `model` 參數 ＋ 前置注入獨立角色檔內容」，
     移除從未在 Claude Code 成功運作的 `.claude/agents/` + `subagent_type` 路徑。
  2. 取消每角色 effort（保留每角色 model；子代理沿用 orchestrator session 的 effort）。
  3. 將文件統一到 `docs/superpowers/{plans,specs,sessions,reviews,qa}/`。
- **本批的設計與計畫文件（接續工作的起點）：**
  - Spec：`docs/superpowers/specs/2026-06-26-orchestrator-dispatch-and-docs-unification-design.md`
  - Plan：`docs/superpowers/plans/2026-06-26-orchestrator-dispatch-and-docs-unification.md`
  - 變更摘要：`CHANGELOG.md` 的 `## [Unreleased] — 2026-06-26` 條目。

## 日後接續的建議起手式

連回後，若要做**新的** design 或改動，直接描述需求即可——例如
「沿用上次的 orchestrator 設計脈絡，新增 X / 修改 Y」。系統會在需要時自動帶起
`brainstorming` → `writing-plans` 的流程；新的 spec 會落在 `docs/superpowers/specs/`、
plan 落在 `docs/superpowers/plans/`，與本批一致。
