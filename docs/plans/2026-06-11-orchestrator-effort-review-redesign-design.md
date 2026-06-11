# Orchestrator Effort 對齊 + Review 強化 — 設計文件

**日期**: 2026-06-11
**範圍**: `skills/orchestrator-driven-development/`（SKILL.md + templates/）
**狀態**: 已核准

## 背景與動機

兩個獨立的問題促成這次改版：

1. **Effort 機制已過時。** 現有 Step 2.5 把 effort 映射為 thinking keyword（`think` / `think hard` / `ultrathink`）prepend 到 dispatch prompt。經官方文件驗證：`think` / `think hard` 現在是普通文字（no-op），只有 `ultrathink` 仍是單次 per-message toggle。同時 Claude Code 已統一用 `/effort`（low/medium/high/xhigh/max）控制推理預算，且 **subagent 定義檔的 frontmatter 支援 `effort:` 欄位**，可硬性覆蓋 session effort。「Model 硬、Effort 軟」的前提不再成立——effort 也可以是硬的。

2. **Review 有系統性盲區。** 一次 orchestrator 完工的實作（tenet-prob）在事後用 `/code-review` 深審，找到 1 Critical + 4 High + 5 Medium，全部是 pipeline reviewer 漏掉的。漏網型態高度集中：
   - 測試**接受邏輯**本身沒人審（容差逃生門把等效容差放寬數個數量級、代數恆真的死斷言）；
   - **生成物的文件**與修復不同步（照舊文件重新生成會精準復刻舊 bug）；
   - 測試**覆蓋域**遠窄於 API **接受域**（建構子收 ν>0，抽樣測試只蓋 ν∈[2.5,50]）；
   - 文件中的**量化宣稱**（精度、ulp）未經實證。

   結論：「A 級的程式碼配 B 級的測試，整體就是 B。」reviewer 的 checklist 沒指向這些地方，且單一 reviewer 缺乏多角度 finder + 對抗驗證的結構。

## 已驗證的事實（設計依據）

| 事實 | 來源 |
|------|------|
| Agent tool 只接受 `model` 參數，無 effort 參數 | 官方 sub-agents 文件 |
| `.claude/agents/*.md` frontmatter 支援 `effort:`（low/medium/high/xhigh/max），覆蓋 session effort；未設定則繼承 | 官方 sub-agents 文件 |
| `/effort` 檔位：opus/sonnet/haiku 均為 low/medium/high/xhigh/max | 使用者實測 |
| `ultracode` 是 Claude Code 的 session-only 設定（xhigh + dynamic workflows），**不是** effort level，不能寫進 frontmatter | 官方 model-config 文件 |
| `ultrathink` 是唯一仍被識別的 prompt keyword，單次 toggle；`think` / `think hard` 為 no-op | 官方 model-config 文件 + 使用者實測 |
| `/code-review ultra` 為使用者觸發、雲端計費，Claude 不能自行啟動 | harness 規範 |

## 決策摘要

| 決策點 | 選擇 |
|--------|------|
| Per-role effort 控制機制 | **生成 subagent 定義檔**（frontmatter `model:` + `effort:`，硬設定） |
| Review 強化程度 | **硬化 per-batch checklist + 新增 Final Audit gate** |
| Standalone 角色檔（01/02/03） | **保留完整內容**，加同源註記 |

## Section 1 — Effort 語意重建（SKILL.md Step 2.5 改寫）

- 刪除 effort→thinking keyword 對照表與所有 `{{*_THINKING}}` placeholder。
- Effort 取值統一為 `low / medium / high / xhigh / max`；原 `minimal` 廢除，`low` 為地板。
- `ultracode` 排除在角色指派之外；SKILL.md 加註：它是 session-only 設定、不可指派給 subagent，亦不建議用於 orchestrator session 本身（協調者不應自行開 workflow）。
- `ultrathink` 不再被 pipeline 使用；留一句註記說明它仍是使用者可手動使用的單次 toggle，與本 skill 的生成物無關。
- Step 2.5 的 AskUserQuestion 從 3 題擴為 **4 題**（仍在 4 題上限內）：Executor、Reviewer、QA 各一題 + 「Final Audit 深度」一題。
- 新 curated combos（Option 1 = 預設，使用者跳過該題時套用）：

| Role | Option 1 (Recommended) | Option 2 | Option 3 | Option 4 |
|------|------------------------|----------|----------|----------|
| Executor | `sonnet` · high | `sonnet` · xhigh | `opus` · high | `haiku` · medium |
| Reviewer | `opus` · xhigh | `opus` · high | `sonnet` · xhigh | `sonnet` · high |
| QA | `sonnet` · high | `sonnet` · xhigh | `opus` · high | `haiku` · medium |
| Final Audit | `/code-review` high | `/code-review` max | skip | — |

  預設值較舊版上調一檔：effort 現在是硬設定、真的生效，且 Claude Code 各模型 session 預設即為 high。

## Section 2 — 生成 subagent 定義檔（新模板 + Step 3 擴充）

- 新增模板 `templates/agent-definitions-template.md`，Step 3 據此生成三個檔案到 `.claude/agents/`：
  - `orchestrator-executor.md`
  - `orchestrator-reviewer.md`
  - `orchestrator-qa.md`

  固定名稱（`.claude/agents/` 是 project-level，跨專案不衝突）。
- Frontmatter：`name`、`description`、`model: {{ROLE_MODEL}}`、`effort: {{ROLE_EFFORT}}`。
- **Body 即角色的 system prompt**：角色靜態內容（規則、驗證指令、報告格式、review checklist）從 orchestrator.md 的 dispatch block 移入 agent 定義檔。
- orchestrator.md dispatch block 縮短為：`subagent_type: "orchestrator-<role>"` + batch 專屬指令（batch 編號、task 範圍、報告路徑）。刪除所有 keyword prepend 規則。
- Step 4 commit 範圍擴為 `git add docs/sessions/ .claude/agents/`。
- Dispatch fallback：若 `subagent_type` 解析失敗（agent 檔被刪），退回 Agent tool `model` 參數 + 角色內容內嵌進 prompt（此時 effort 無法強制，僅繼承 session）。
- Standalone 角色檔（01/02/03）保留完整內容，改動兩處：
  1. 建議行改為「開新 session 後執行 `/effort {{ROLE_EFFORT}}`、模型選 `{{ROLE_MODEL}}`」；
  2. 加註內容與對應 `.claude/agents/` 檔同源，修改 checklist 時兩邊同步。

## Section 3 — Reviewer checklist 硬化（6 條 → 10 條）

新增四條，寫入 reviewer 模板、agent 定義檔、orchestrator dispatch block 精簡版：

7. **測試接受邏輯** — 用審程式碼的強度審容差/比較邏輯：每個 tolerance 是否貼緊實際需要的誤差？有無無條件逃生門？有無代數上恆真的死斷言？註解對容差的描述是否屬實？核心提問：「**什麼樣的錯誤實作仍能通過這些測試？**」
8. **生成物與文件同步** — 修復若觸及生成的程式碼/表格，生成器與生成物的文件、註解必須反映修復後的慣例。
9. **覆蓋域 vs 接受域** — 測試覆蓋範圍必須對齊公開 API 實際接受的輸入域；蓋不到的域要嘛 API 拒絕、要嘛測試補上。
10. **量化宣稱要有實證** — docs/註解中的精度、效能等數字宣稱，必須有 diff 內的測試或量測支撐，否則列為 finding。

## Section 4 — Final Audit gate（新 pipeline 階段）

QA PASS 之後、宣告完成之前：

```
QA PASS
  ↓
FINAL AUDIT: orchestrator 在自己 session 內呼叫 /code-review skill
  （scope: feature branch vs main 的完整 diff；effort 取 Step 2.5 第 4 題）
  ↓
- Critical/P0 findings → dispatch executor-fix → audit verify（max 2 輪）
- 非阻斷 findings → docs/reviews/YYYY-MM-DD-final-audit.md 的 BACKLOG 節
- 2 輪後仍有 P0 → STOP 問使用者
  ↓
done
```

- Fallback：orchestrator 環境若無 `/code-review` skill，改派 reviewer subagent 做 whole-branch audit，prompt 內嵌多角度 finder 指引（演算法數學、測試接受邏輯、文件宣稱、API 契約）+ 逐項對抗驗證。
- Step 2.5 選 skip 則整段省略。
- 明文標註：`/code-review ultra` 為使用者觸發、雲端計費，orchestrator 不得自行呼叫；audit 深度上限為 local `max`。

## Section 5 — progress.json / resume.md 對應更新

- `model_assignments` 每角色改為 `{"model", "effort", "agent": "orchestrator-<role>"}`；刪除 `"thinking"` 欄位。
- `current_step` 新增 `final_audit` / `audit_fix` / `audit_verify`；新增頂層 `audit_status` 欄位（`null` / `"PASS"` / `"SKIPPED"`）。
- resume.md fallback 鏈改為三層：
  1. `.claude/agents/orchestrator-*.md` 存在 → 以 `subagent_type` 派發（model/effort 由 frontmatter 強制）；
  2. agent 檔不存在但 `model_assignments` 在 → Agent tool `model` 參數派發、角色內容內嵌（effort 無法強制）；
  3. 兩者皆無（舊版 session）→ 無 model 參數派發。
- resume.md 的 step 對照表新增 audit 三列。
- SKILL.md Pipeline Architecture 圖加入 Final Audit；fork CHANGELOG 補一條。

## 不做的事（YAGNI）

- 不在 pipeline 中使用 `ultrathink` prepend（已被 frontmatter effort 取代；混用兩種機制徒增規則）。
- 不做 per-batch `/code-review`（成本高且會把 review 報告灌進 orchestrator context，違背委派初衷）。
- 不為 Final Audit 引入獨立的第四個 agent 定義檔（audit 走 `/code-review` skill 或 reviewer fallback）。

## 設計原則

把「祈使句層級」的控制換成「harness 強制」的控制：effort 從 prompt keyword 變 frontmatter 硬設定；review 盲區從「希望 reviewer 想到」變 checklist 明文 + 結構性的多角度終審。可靠性從模型自覺移到機制。
