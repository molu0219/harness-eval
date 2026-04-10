---
name: harness-eval
description: General evaluation framework for AI development systems — scores design quality across 6 dimensions regardless of implementation
---

# /harness-eval

Evaluate any AI development system's design effectiveness. Not "did it work this time" but "is this design capable of working well?"

Applicable to: Claude Code harness, Cursor rules, Windsurf, Aider, custom CLAUDE.md setups, or any AI-assisted development configuration.

## Step 0: Parse Arguments

`/harness-eval` → evaluate current project's AI development system
`/harness-eval <path>` → evaluate a specific system at given path
`/harness-eval compare <path1> <path2>` → compare two systems

---

## Step 1: Discover System Components

Read all AI instruction files in the target. Don't assume any specific structure — discover what exists.

Look for (in order):
1. **System prompts / instructions**: CLAUDE.md, .cursorrules, .windsurfrules, .aider*, rules/*.md, any file referenced by AI config
2. **Automation**: hooks, scripts, pre/post processors, CI integrations
3. **State management**: task tracking files (SPEC.md, TODO.md, etc.), progress files
4. **Knowledge**: architecture docs, decision logs, graphify output, wiki
5. **Configuration**: settings.json, .claude/, .cursor/, tool configs

For each component found, note:
- **Load timing**: always (every API call), per-session (once), per-event (conditional), on-demand (explicit read)
- **Size**: bytes
- **Purpose**: instruction, enforcement, state, reference

---

## Step 1.5: Flow Simulation (開發流程模擬)

> 在評估維度之前，先把開發流程走一遍。不是真的跑 code，而是追蹤：每一步會發生什麼、呼叫什麼、context 裡有什麼、消耗多少 token。

### 模擬方法

畫出完整的開發生命週期，在每個轉換點標記：

```
[Phase] → (觸發什麼) → {context 狀態} → ~token 消耗
```

### 標準生命週期

以下是任何 AI 開發系統都會經過的 phases。對每個 phase，從系統的設計文件（CLAUDE.md, hooks, settings）推導會發生什麼：

#### Phase 0: Session Start
- AI 載入系統指令（instruction files + rules）→ 記錄 token 消耗
- hooks 觸發？輸出什麼？→ 加到 context
- AI 要讀幾個檔案才能開始工作？→ 列出
- **檢查**: 從 session start 到第一行 code 有幾步？每步必要嗎？

#### Phase 1: Development
- AI 寫 code → 哪些 hooks 觸發？多頻繁？
- 每次 Edit/Write 的 hook overhead 是多少 token？
- 有沒有 hook 在這個 phase 跑了但沒輸出？（idle cost）
- **檢查**: hook 頻率和 code 產出的比例合理嗎？

#### Phase 2: Task Completion
- AI 完成 task → 需要幾步 ceremony？
- 每步是否有前一步的輸出作為輸入？（鏈路完整性）
- hook 在哪些步驟觸發？block 還是 warn？
- **檢查**: ceremony 步驟之間有沒有循環依賴？

#### Phase 3: Verification (feature group)
- 全部任務完成 → 誰觸發 Evaluator？怎麼觸發？
- Evaluator spawn → AI 寫回 {task tracking file} → {completion gate hook} 再觸發 → 會不會無限迴圈？
- Review 同理
- **檢查**: 觸發鏈是否收斂（最終到 archive 結束），還是可能發散？

#### Phase 4: Session End / Compact
- AI compact → 保留什麼？丟失什麼？
- 下個 session 能恢復多少？怎麼恢復？
- **檢查**: compact 前後的資訊差距是否在可接受範圍？

### Flow 檢查清單

| 檢查 | 問什麼 | Fail 條件 |
|------|--------|----------|
| **Step Count** | 從 session start 到第一行 code 幾步？ | >5 步 = 過重 |
| **Circular Dependency** | hook A 觸發 → AI 動作 → hook A 再觸發 → ... 會不會無限迴圈？ | 任何無收斂的迴圈 |
| **Dead End** | 有沒有 phase 的輸出沒有任何下游消費？ | 有輸出但沒人讀 |
| **Missing Link** | phase N 的下一步需要什麼輸入？上一步有沒有產出？ | 需要 X 但沒人產出 X |
| **Token Proportionality** | 一個 task 的 ceremony token 佔總 token 多少？ | ceremony > 30% of total |
| **Tool Appropriateness** | 每步呼叫的 tool/hook 是最適合的嗎？ | 用 UserPromptSubmit hook 做應該在 PostToolUse 做的事 |
| **Observability Coverage** | 每個 phase 有沒有留下可追蹤的 log/artifact？ | 有 phase 無任何記錄 = fail |

### Observability Coverage 追蹤

對每個 Phase，問三個問題：
1. **這個 phase 發生了什麼？** → 有沒有 structured event/artifact 記錄？
2. **出問題時能 trace back 嗎？** → 有沒有足夠資訊重現或診斷？
3. **記錄是自動的還是靠 AI 記得寫？** → 自動 = 可靠, 手動 = 可能遺漏

| Phase | 該有什麼 log | 自動/手動 | 沒有時的後果 |
|-------|-------------|----------|-------------|
| **0. Boot** | boot event（什麼專案、什麼 model、什麼模式）| 自動（hook）| 不知道 session 何時開始、用什麼設定 |
| **1. Development** | git commits（什麼改了、為什麼）| 手動（AI commit）| 不知道中間做了什麼決策 |
| **2. Task Complete** | task completion artifact（做了什麼、驗證方式）| 手動但有 gate | 跳過 artifact → {completion gate hook} block |
| **3. Verification** | verification log（Evaluator/Review PASS/FAIL）| 手動但有 gate | 跳過 → {completion gate hook} warn |
| **4. Session End** | handoff note / decision log / retro event | 手動但有 check | {session hook} 檢查 handoff；{compact hook} 強制 decision write |
| **Hook 觸發** | hook debug log + timing event | 自動（hook utils）| 不知道 hook 花了多久、有沒有報錯 |

評分：
- 所有 phase 都有自動化 log = A
- 關鍵 phase (2,3) 有 gate 保護, 其他有手動 + check = B
- 有 phase 完全沒記錄 = C
- 多數 phase 沒記錄 = F

### Token Flow 追蹤

對一個典型 task（寫一個 function + test），估算每個 phase 的 token 消耗：

```
Phase 0: Boot
  Always-loaded: {N} tokens (CLAUDE.md + rules)
  Hook output: {M} tokens ({context injection hook})
  File reads: {K} tokens (SPEC + optional)
  → Subtotal: {X} tokens before first code

Phase 1: Development (~20 API calls for simple task)
  Per-call fixed: {N} tokens × 20 calls = {Y}
  Hook outputs: ~{Z} tokens total (most idle)

Phase 2: Completion
  Ceremony: verify + log + mark = ~{C} tokens
  Hook: {completion gate hook} block/action prompt = ~{H} tokens

Phase 3: Verification (if >3 tasks)
  Evaluator spawn: ~{E} tokens
  Reviewer spawn: ~{R} tokens

Total: {GRAND TOTAL} tokens per task
Overhead ratio: (Phase 0 + hooks + ceremony) / total = {RATIO}%
```

### Flow Simulation 輸出

```
FLOW: Session Start → [N steps] → First Code → [20 edits, 2 hooks/edit] → Task Done → [3 ceremony] → Feature Group Complete → [action prompt → Evaluator → Review → Archive] → Compact

Token breakdown:
  Boot:       {N} tokens (X%)
  Per-call:   {N} tokens (X%)  ← largest, unavoidable
  Hooks:      {N} tokens (X%)
  Ceremony:   {N} tokens (X%)
  Verify:     {N} tokens (X%)

Issues found:
  ✓ No circular dependency
  ✓ No dead ends
  ⚠ {completion gate hook} action prompt → AI writes {task tracking file} → hook re-triggers (但有收斂: 第二次不會有新 action prompt 因為 Evaluator result 已存在)
```

Flow Simulation 的結果會影響 Step 2 的評分：
- Circular dependency → Robustness 降級
- Step count 過多 → Overhead 降級
- Token proportionality 過高 → Overhead 降級
- Missing link → Adherence 降級（流程斷裂 = 規則無法執行）
- Dead end → Context Efficiency 降級（產出沒人讀 = 浪費）
- Observability gap → Adherence 降級（無法事後驗證流程是否正確執行）

---

## Step 1.7: Assumption Stress Test

> "Every component in a harness encodes an assumption about what the model can't do on its own, and those assumptions are worth stress testing." — Anthropic Engineering

對每一個元件（hook, rule, instruction, ceremony step），問：

```
這個元件假設 AI 不能自己做什麼？這個假設在當前 model 下還成立嗎？
```

### Method

1. 列出所有元件
2. 對每個元件寫出它的 **capability assumption**（假設 AI 不能做 X，所以需要這個元件來做 X）
3. 用當前 model 的已知能力驗證：這個假設還成立嗎？
4. 如果不成立 → 這個元件是 **stale scaffolding**，應該移除或降級

### 常見 stale scaffolding 模式

| 模式 | 假設 | 什麼時候不再成立 |
|------|------|-----------------|
| 每次 prompt 注入進度 | AI 記不住上一步做了什麼 | Model context window 夠大，且 AI 自己管狀態 |
| 強制讀取特定檔案 | AI 不會自己去找需要的資訊 | Model 有 tool use，會主動 Read |
| 步驟分解提醒 | AI 不能自己拆解任務 | Model planning 能力提升 |
| Sprint decomposition | AI 不能維持長任務一致性 | Model 長 context 能力提升（Opus 4.6 已移除 sprint） |
| 強制 TDD 流程 | AI 不寫 test | Model 自帶 coding best practices |

### 觸發時機

- 每次 model 升級後跑一次
- eval 發現 Overhead 分數下降時
- 元件的 idle rate > 50% 時

### 輸出

```
ASSUMPTION TEST:
  ✓ {completion gate hook} — 假設: AI 會跳過 completion artifact 直接標完成。成立: 無 hard gate 時確實會跳
  ✓ {permission deny list} — 假設: AI 會嘗試危險操作。成立: 模型會嘗試 rm -rf 等
  ⚠ {tech stack detection hook} — 假設: AI 不掃 dependency files。部分成立: AI 會讀但不一定主動掃
  ✗ {component X} — 假設: [寫出假設]。不成立: [說明為何 model 現在能自己做]。建議移除
```

---

## Step 2: Evaluate Six Dimensions

### Dimension 1: Endurance (持久力)

> AI 在這個系統下能持續有效工作多久才會開始退化？

Evaluate the design's impact on session longevity:

| Criteria | Question | Score |
|----------|----------|-------|
| **Context Budget** | 系統指令佔多少 context？剩多少給實際工作？ | always-loaded bytes / model context window. <5% = A, 5-10% = B, 10-20% = C, >20% = F |
| **Growth Bound** | 隨著工作進行，注入的 context 會不會無限增長？ | 有上限機制 (cache/archive/compact) = A, 線性增長但慢 = C, 無限制增長 = F |
| **Recovery Design** | Context 重置後，系統能恢復多少狀態？ | full reset + structured handoff = A, compact 保留部分 = B, 無機制 = F |
| **Recovery Chain** | 恢復機制的每一步有觸發鏈嗎？ | 每步有明確觸發 = A, 靠 AI 記得 = C, 斷鏈 = F |
| **Continuity Design** | 跨 session 時，工作能不能銜接？ | 有 handoff + state file = A, 靠 git history 推斷 = C, 完全斷裂 = F |
| **Degradation Curve** | 品質退化是漸進的還是懸崖式的？ | 有分級機制 (lite/full) 漸進退化 = A, 直到崩潰才處理 = F |

**Calculation**: Context Budget 是最關鍵指標。

Model context windows (input):
- Claude Opus: ~200K tokens
- Claude Sonnet: ~200K tokens  
- GPT-4: ~128K tokens
- Cursor (effective): ~20-40K tokens per completion

Formula: `context_budget_score = always_loaded_tokens / effective_context_window`

### Dimension 2: Context Efficiency (空間效率)

> 在任何時刻，context 裡的東西是不是 AI 當下需要的？而且是 AI 自己不知道的？

| Criteria | Question | Score |
|----------|----------|-------|
| **Redundancy** | 注入的資訊裡，有多少是 AI 自己已經知道的？ | 0% 冗餘 = A, <20% = B, 20-50% = C, >50% = F |
| **Map vs Manual** | 指令檔是目錄（指向深層文件）還是百科（自己講完全部）？ | ~100 行 index 指向細節 = A, 混合 = C, 巨型單檔 = F |
| **Signal-to-Noise** | always-loaded 內容在當前任務實際用得到多少？ | >80% 相關 = A, 50-80% = C, <50% = F |
| **Timeliness** | 資訊是在需要時才注入，還是提前塞？ | 大部分按需載入 = A, 混合 = C, 全部 always-loaded = F |
| **Freshness** | 有沒有機制清理過時資訊？ | 自動清理 (archive/expire) = A, 手動清理 = C, 永遠累積 = F |
| **Layering** | 靜態規則、動態狀態、參考文件有沒有分層？ | 3+ 層明確分離 = A, 2 層 = C, 全部混在一起 = F |
| **Deduplication** | 同一概念有沒有在多處重複**定義**？ | 無重複定義 = A, 少量 = C, 嚴重 = F |

**Redundancy 是最重要的 criteria**。其他都在問「放了什麼」，這個在問「該不該放」。

**How to assess Redundancy — Ownership Tracing**:

對每一條動態注入（hook output）和每一段 auto-loaded instruction，追蹤完整鏈路：

```
誰創建這個資訊？ → 誰修改它？ → 誰需要它？ → 誰傳遞它？
```

分類規則：
1. **Creator = Consumer** → 中間的 deliverer 是冗餘。例: AI 改 SPEC → hook 注入 SPEC 摘要 → AI 讀摘要。AI 是 creator 也是 consumer，hook 是冗餘 deliverer
2. **已在 context** → 重複注入。例: CLAUDE.md 說「讀 SPEC」→ hook 也說「讀 SPEC」
3. **Creation chain 推導** → 如果 A 的存在必然伴隨 B，那告知 B 存在是冗餘。例: project-init 建 SPEC 時同時建 DECISION → 有 SPEC 就有 DECISION → 告知 DECISION 存在是冗餘
4. **AI 無法自己得知** → 有價值。例: tech stack 偵測（AI 不掃 package.json）

Redundancy = (type 1 + type 2 + type 3) / 總注入量

**How to assess Map vs Manual**:
巨型指令檔的三個死因（引用 Anthropic Engineering）：
1. 擠佔 context — 留給實際工作的空間變少
2. 無法維護 — 越大越難改，越難改越過時
3. 無法機械驗證 — 純文字指令不能被 lint 檢查

Map 的特徵：主檔案 ≤ 150 行，用 progressive disclosure 指向 rules/, docs/, SPEC。
Manual 的特徵：主檔案 > 300 行，包含完整流程、範例、template。

**How to assess Signal-to-Noise**: 
Read all always-loaded content. For each paragraph, ask: "在一個只改一個 CSS class 的任務裡，這段話有用嗎？" Count useful vs total. If most content is only relevant to complex multi-agent scenarios but loads for every task → low signal.

**How to assess Layering**:
- Layer 1 (always): Core behavior rules that apply to every interaction
- Layer 2 (conditional): Task-specific context loaded when detected (e.g., has SPEC → load SPEC rules)
- Layer 3 (on-demand): Reference docs the AI reads only when it needs them
- If all three exist and are clearly separated → A. If everything is in one file → F.

**How to assess Deduplication**:
Distinguish between **definition** and **reference**:
- **Definition**: teaches the concept (what to do, how to do it, rules). Example: coding-style.md's "Error Handling" section with 4 bullet points
- **Reference**: mentions the concept in passing or points to the definition. Example: code-review.md severity example "缺 error handling 導致 crash"

Only count as duplicate if the same concept has **2+ definitions**. References are fine — they're how you avoid duplication.

Grep alone will over-count (matches keywords in both definitions and references). Manual classification needed.

### Dimension 3: Overhead (系統消耗)

> 這個系統本身消耗多少資源？相對於它帶來的價值，值不值得？

| Criteria | Question | Score |
|----------|----------|-------|
| **Fixed Tax** | 每次 API call 的固定 token 消耗 | <2000 tokens = A, 2000-5000 = B, 5000-10000 = C, >10000 = F |
| **Variable Tax** | 每次人類輸入後，系統附加多少 token？ | <50 tokens = A, 50-200 = C, >200 = F |
| **Ceremony** | 完成一個任務需要多少「流程步驟」而非「寫 code」？ | <3 steps = A, 3-5 = C, >5 = F |
| **Boot Sequence** | Session 開始時需要讀多少檔案才能開始工作？ | 0-1 file = A, 2-3 = B, 4+ = C, 讀完還要跑命令 = F |
| **Idle Components** | 有多少元件在大多數情況下跑了但沒有產出？ | <20% = A, 20-50% = C, >50% = F |
| **Trigger Placement** | 每個提醒/檢查放在最接近動作發生的觸發點嗎？ | 全部精準 = A, 多數精準 = B, 有錯放 = C, 嚴重錯放 = F |

**How to assess Trigger Placement**:
每個提醒映射到最佳觸發點:
- {task tracking file} 改動相關 → PostToolUse Edit {task file}（不是每次 prompt）
- 程式碼品質 → PostToolUse Edit/Write code（不是 session 開始）
- Session 級別 → UserPromptSubmit 首次（不是每次）
- 離開前 → Stop hook 或 PreCompact

如果提醒放在 UserPromptSubmit（每次 prompt）但動作只在 Edit {task file} 時發生 → 錯放 = 浪費。

**How to assess Ceremony**:
Trace the path from "task done" to "officially done":
- Just commit → 0 ceremony
- Commit + update task file → 1 step
- Commit + update task + write log + run verify + mark complete → 4 steps
- Commit + log + mark + spawn evaluator + spawn reviewer + archive → 6 steps

More ceremony = more consistent quality, but also more overhead. Score based on whether the ceremony is proportional to the task complexity (one-size-fits-all ceremony = bad).

### Dimension 4: Adherence (遵守度)

> 系統規則實際上能被遵守嗎？有沒有執行機制？

| Criteria | Question | Score |
|----------|----------|-------|
| **Enforceability** | 規則是機械化執行（lint/hook block）還是軟建議？ | 關鍵規則有 hard gate + error 含修復指令 = A, hard gate 但無修復指令 = B, 全是 soft = F |
| **Observability** | 能不能事後知道規則有沒有被遵守？ | 有 artifact/log 可驗證 = A, 只能看 code = C, 無法驗證 = F |
| **Consistency** | 規則之間有沒有矛盾？ | 0 矛盾 = A, 有但可判斷優先級 = C, 有且無法解決 = F |
| **Proportionality** | 重要規則有強執行，不重要的放輕？ | 分級明確 = A, 全部同等重要 = C, 重要的反而沒 enforce = F |
| **Escape Hatch** | 當規則和現實衝突時，有沒有合理的跳出機制？ | 有明確的 override/PAUSE 條件 = A, 只能違反 = C, 完全沒辦法 = F |
| **Cross-Reference** | 指令檔說的規則和 hook 實際檢查的是同一件事嗎？ | 完全一致 = A, 有少量 phantom reference = C, 嚴重不一致 = F |

**How to assess Cross-Reference**:
1. 從指令檔（CLAUDE.md/rules）提取所有引用 hook 行為的語句
2. 對每條：hook 實際上做不做這件事？（phantom reference = 說了但不做）
3. 從 hooks 提取所有輸出：指令檔有沒有告訴 AI 如何回應？（unhandled output = 做了但沒說）
4. settings.json 的 deny/ask 和 CLAUDE.md 的禁止行為列表是否一致？

**How to assess Enforceability — Tool Chain Mapping**:

對系統中的每一條規則，追蹤它的 enforcement tool chain：

| Enforcement Tool | 預期遵守率 | 例子 |
|-----------------|-----------|------|
| permission deny list | ~100% | rm -rf, force push, deploy |
| permission ask list | ~100% | git push, package install |
| hook exit 2 (block) | ~100% | 完成任務無 log artifact, hardcoded secret |
| hook warn + fix instruction | ~80-90% | 改 code 無 test, diff 過大 |
| hook warn (無 fix instruction) | ~60-70% | 歸檔提醒、過期 decision |
| spawned Reviewer post-check | ~90% | 重造輪子, test 覆蓋 |
| rules/instruction file soft text | ~40-70% | 系統思維, trade-off reasoning |
| 無任何 enforcement | ~20-40% | 完全靠 AI 自覺 |

**Method**:
1. 列出 rules/CLAUDE.md 中所有規則（每個 `-` bullet 或 checklist item）
2. 對每條規則標記其 enforcement tool（deny/block/warn/reviewer/soft/none）
3. 計算加權遵守率：`Σ(每條規則的預期遵守率) / 總規則數`
4. 找出 **high-value rules with low enforcement**（重要但只有 soft 的規則）

**Scoring**:
- 加權遵守率 > 80% = A (關鍵規則都有 hard enforcement)
- 60-80% = B (多數有 enforcement)
- 40-60% = C (混合)
- < 40% = F (大多是 soft)

**重要**: 不是 hard rules 越多越好。是**重要的規則要有 hard enforcement，不重要的可以 soft**。一個系統只有 10 條 hard rules 但都覆蓋關鍵操作 = A。100 條 hard rules 但 AI 被擋得無法工作 = F（over-enforcement）。

### Dimension 5: Robustness (穩健度)

> 系統本身會不會壞？壞了會怎樣？

| Criteria | Question | Score |
|----------|----------|-------|
| **Graceful Degradation** | 一個元件壞了，其他還能用嗎？ | 元件獨立，壞了跳過 = A, 有依賴但能 fallback = C, 連鎖崩潰 = F |
| **Error Isolation** | 一個 hook 報錯會不會影響整個 session？ | 錯誤被 catch，session 繼續 = A, 會中斷流程 = F |
| **State Consistency** | Source of truth 有幾個？會不會衝突？ | 單一 source of truth = A, 多個但有同步 = C, 多個且會 drift = F |
| **Empty State** | 什麼都沒有時（新 repo, 無 SPEC, 無 DECISION）能正常工作嗎？ | 完全正常 = A, 部分功能可用 = C, 報錯或卡住 = F |
| **Mid-Session Change** | 人類中途改了 code/需求/config，系統能適應嗎？ | 有偵測+調整機制 = A, 靠 AI 判斷 = C, 會衝突 = F |
| **Repo as Record** | AI 需要的所有知識都在版控的檔案裡嗎？ | 全部在 repo/config = A, 部分在外部 (Slack/Docs) = C, 關鍵知識不在 repo = F |
| **Entropy Defense** | 有沒有機制防止 AI 複製 repo 裡的壞模式？ | 有 golden rules + 背景掃描 = A, 有 lint = B, 沒有 = F |
| **Dependency Liveness** | 所有被引用的元件還活著嗎？ | 0 dead ref = A, 有但非關鍵 = C, 關鍵依賴死了 = F |

**How to assess Repo as Record**:
"不在倉庫裡的東西，對智能體來說不存在" — Anthropic Engineering
1. 列出 AI 做決策需要的所有知識（架構、規範、計劃、決策記錄）
2. 每項：在 repo 的版控檔案裡嗎？還是只在 Slack/Google Docs/人腦裡？
3. 全域指令（~/.claude/）算 repo 嗎？→ 如果有版控的 harness repo 且 install/symlink 機制 = 算。如果手寫沒備份 = 不算。

**How to assess Entropy Defense**:
AI 會複製 repo 中已有的模式——包括壞模式。
1. 有沒有 lint/structural test 保護不變量？(不只是 style，是結構：「每個 API 必須有 error response」)
2. 有沒有背景任務掃描偏差？（{session hook} 偵測重複修改、stale decisions）
3. lint/hook error message 有沒有內嵌修復指令，讓 AI 能自我糾正？

**How to assess Dependency Liveness**:
1. hooks: grep source/bash 呼叫的其他 script → 檢查存在且非空
2. 指令檔: grep 引用的 docs/hooks/路徑 → 檢查存在
3. settings.json: 註冊的 hook 路徑 → 檢查存在且非空
4. docs/: 被指令檔引用的 on-demand 檔案 → 檢查存在

**How to assess Graceful Degradation**:
Draw the dependency graph of all components. For each:
- If it fails, what else breaks?
- Is there a `|| true`, `2>/dev/null`, or fallback?
- Does it block (exit 2) or warn (echo)?

Single point of failure (one component failure kills the session) = automatic F.

### Dimension 6: Collaboration (協作品質)

> 人類和 AI 之間的互動設計好不好？

| Criteria | Question | Score |
|----------|----------|-------|
| **Right Info at Right Time** | 系統會不會在 AI 需要時才給資訊？ | 有狀態感知的注入 = A, 固定注入 = C, 不注入讓 AI 自己找 = F |
| **Noise Level** | 多少系統輸出是人類/AI 不需要看的？ | 只在有 actionable info 時輸出 = A, 有 cache 防重複 = B, 每次都輸出 = F |
| **Autonomy Gradient** | 什麼時候該自動做、什麼時候該問人類？ | 有明確分級（deny/ask/allow）= A, 模糊 = C, 全自動或全手動 = F |
| **Transparency** | AI 在做什麼，人類看得到嗎？ | 有 progress tracking + event log = A, 有但分散 = C, 黑箱 = F |
| **Multi-Agent Design** | 多個 AI agent 之間能不能協調？ | 有協調協議 + 衝突解決 = A, 簡單分工 = C, 沒考慮 = F |
| **Agent Readability** | codebase 對 AI 可讀嗎？用的技術 AI 熟悉嗎？ | 用 "boring" tech (API 穩定、訓練集覆蓋) + worktree 可隔離 = A, 混合 = C, 用 AI 不熟的框架 = F |
| **Failure Analysis** | 出問題時，系統引導「缺什麼 context/tool/constraint」還是「再試一次」？ | 有明確的失敗診斷流程 = A, 靠 AI 判斷 = C, 無引導 = F |
| **Config Alignment** | 權限設定和行為指令一致嗎？ | settings deny/ask 和 CLAUDE.md 禁止行為完全對應 = A, 有差異但不矛盾 = B, 有矛盾 = F |

**How to assess Agent Readability**:
"優先選擇 boring 技術" — API 穩定、訓練集覆蓋好的框架 AI 更容易正確使用。
1. 主要框架是 AI 訓練集常見的嗎？(React/Express/FastAPI = boring ✓, 自研框架 = risky)
2. 有沒有 opaque upstream 的 wrapper？(有時重新實現子集比包裝不透明的上游行為更划算)
3. 應用能不能按 git worktree 啟動隔離實例？(AI 可以啟動自己的測試環境)

**How to assess Failure Analysis**:
"人類時間是最稀缺的資源。出問題時，答案不是更努力，而是缺什麼 context/tool/constraint"
1. 系統有沒有 PAUSE 機制（遇到不確定 → 停下來問）？
2. hook block 時有沒有告訴 AI 具體缺什麼 + 怎麼修？
3. 重複失敗時有沒有升級機制（而不是一直重試）？

**How to assess Config Alignment**:
1. 列出 {permission config} 的 deny 清單 → {instruction file} 禁止行為有沒有對應？
2. 列出 {instruction file} 的禁止行為 → {permission config} 有沒有用 deny 或 ask 支撐？
3. 如果 instruction file 說「不 push」但 permission config 沒有封鎖 → 軟規則無硬支撐
4. 如果 permission config deny 了某操作但 instruction file 沒提 → AI 不理解為什麼被擋

---

## Step 2.5: Move Impact Check

> 當 eval 在 Step 4 建議搬移內容時，必須先跑此檢查。

搬移前三問:
1. **誰引用？** — grep 內容關鍵字 across all files，找到所有引用者
2. **目標有觸發嗎？** — 如果目標是 docs/（on-demand），AI 什麼時候知道要讀？沒觸發 = 等於不存在
3. **來源更新了嗎？** — 搬移後，原本引用這段內容的地方（CLAUDE.md, hooks）是否已更新？

Failure pattern: 為了優化 Token/Space 把內容移到 docs/，但沒有觸發機制讓 AI 知道什麼時候讀 → 內容等於不存在。

---

## Step 3: Score and Generate Report

### Scoring Rules

For each dimension, average the criteria scores:
- A (Excellent) = 4, B (Good) = 3, C (Adequate) = 2, D (Poor) = 1, F (Failing) = 0, N/A = skip

Dimension score: average of criteria → map back to letter grade
- ≥3.5 = A, ≥2.5 = B, ≥1.5 = C, ≥0.5 = D, <0.5 = F

**Overall** = weighted average:
- Robustness ×2 (if system breaks, nothing else matters)
- Endurance ×1.5 (determines maximum useful work)
- Adherence ×1.5 (determines quality of work)
- Context Efficiency ×1, Overhead ×1, Collaboration ×1

### Output Format

```
═══════════════════════════════════════════════════════════
  AI DEVELOPMENT SYSTEM EVALUATION
  System: {name}
  Components: {N} instruction files, {M} hooks, {K} state files
  Total always-loaded: {bytes} bytes (~{tokens} tokens)
═══════════════════════════════════════════════════════════

┌─────────────────────┬───────┬─────────────────────────────┐
│ Dimension           │ Score │ Key Finding                 │
├─────────────────────┼───────┼─────────────────────────────┤
│ 1. Endurance        │ {A-F} │ {one-line}                  │
│ 2. Context Eff.     │ {A-F} │ {one-line}                  │
│ 3. Overhead         │ {A-F} │ {one-line}                  │
│ 4. Adherence        │ {A-F} │ {one-line}                  │
│ 5. Robustness       │ {A-F} │ {one-line}                  │
│ 6. Collaboration    │ {A-F} │ {one-line}                  │
├─────────────────────┼───────┼─────────────────────────────┤
│ OVERALL             │ {A-F} │                             │
└─────────────────────┴───────┴─────────────────────────────┘
```

### Per-Dimension Detail

For each dimension, output:
1. Criteria table with scores and evidence
2. Strongest point (what the design does well)
3. Weakest point (biggest design flaw)
4. One specific, actionable fix

### Comparative Notes

If evaluating a system with hooks + state management (e.g., harness):
- Compare to baseline: "same project with just a CLAUDE.md and no hooks"
- Is the added complexity worth the added control?

If evaluating a minimal system (e.g., just .cursorrules):
- Note what's missing that would be valuable
- Note what's gained by simplicity (lower overhead, less to break)

---

## Step 4: Recommendations

Max 5, prioritized by:
1. Robustness fix (system is broken)
2. Endurance fix (AI can't sustain work)  
3. Overhead reduction (wasting money/context)
4. Adherence improvement (quality at risk)
5. Context/Collaboration polish (nice to have)

Each recommendation:
- What to change (specific)
- Why (which criteria it improves)
- Trade-off (what you give up)

---

## Reference: System Archetypes

Common AI development system patterns and their typical scores:

| Archetype | Example | Endurance | Context | Overhead | Adherence | Robustness | Collab |
|-----------|---------|-----------|---------|----------|-----------|------------|--------|
| **Bare** | No config at all | A | F | A | F | A | F |
| **Single-file** | One CLAUDE.md | B | C | B | D | A | D |
| **Rules-based** | .cursorrules + rules/ | B | B | B | D | A | C |
| **Hooks-enhanced** | CLAUDE.md + hooks | C | B | C | B | C | B |
| **Full harness** | SPEC + hooks + multi-agent | D | B | D | A | C | A |
| **Over-engineered** | Everything + kitchen sink | F | D | F | B | D | B |

The sweet spot depends on project complexity:
- Solo weekend project → Single-file or Rules-based
- Team project, 1-3 months → Hooks-enhanced
- Complex multi-agent, long-running → Full harness
- If your overhead score is worse than your adherence score → you're over-engineered
