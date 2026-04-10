---
name: harness-eval
description: General evaluation framework for AI development systems — scores design quality across 6 dimensions regardless of implementation
---

# /harness-eval

評估任何 AI 開發系統的設計效能。不是「這次有沒有跑通」，而是「這個設計有沒有能力持續跑好？」

適用於：Claude Code harness、Cursor rules、Windsurf、Aider、自訂 CLAUDE.md 設定，或任何 AI 輔助開發配置。

## 步驟零：解析參數

`/harness-eval` → 評估當前專案的 AI 開發系統
`/harness-eval <path>` → 評估指定路徑的系統
`/harness-eval compare <path1> <path2>` → 比較兩個系統

### 評估範圍（重要）

評估必須嚴格限定在**目標系統**，不是整個 Claude Code 環境。

| 呼叫方式 | 評估範圍 |
|---------|---------|
| `/harness-eval` | 當前專案目錄：`./CLAUDE.md`、`./hooks/`、`./rules/`、`./settings.json`、`./.claude/` |
| `/harness-eval <path>` | 嚴格限定在 `<path>` 內的檔案 |
| `/harness-eval compare <p1> <p2>` | 各路徑獨立評估 |

**全域 `~/.claude/` 檔案不在評估範圍內**，除非：
- 目標系統本身就是一個會安裝到 `~/.claude/` 的 harness repo（例如透過 `install.sh` 或 symlink），AND
- 你要評估的是整個 harness 系統

這種情況下，在元件清單裡將全域安裝的元件標記為 **[shared infrastructure]**，與 **[project-level]** 分開列出。

> 為什麼重要：不限定範圍的話，`/harness-eval` 會抓到使用者全域的 CLAUDE.md 和 rules，導致「always-loaded」token 數虛增，把多個系統混為一談。評出來的是「整個 Claude Code 環境」，而不是「這個 harness 本身」。

---

## 步驟一：探索系統元件

讀取**評估範圍內**（步驟零定義）的所有 AI 指令檔案。不要預設任何特定結構——探索實際存在的內容。

查找（依序）：
1. **系統提示 / 指令**：CLAUDE.md、.cursorrules、.windsurfrules、.aider*、rules/*.md、AI 設定引用的任何檔案
2. **自動化**：hooks、scripts、前後處理器、CI 整合
3. **狀態管理**：任務追蹤檔案（SPEC.md、TODO.md 等）、進度檔案
4. **知識庫**：架構文件、決策日誌、graphify 輸出、wiki
5. **設定**：settings.json、.claude/、.cursor/、工具設定

對每個找到的元件，記錄：
- **載入時機**：always（每次 API 呼叫）、per-session（每次 session 一次）、per-event（條件性）、on-demand（明確讀取）
- **大小**：bytes
- **用途**：instruction、enforcement、state、reference

如果系統會安裝到全域位置（例如 `~/.claude/`），追蹤安裝機制，評估**來源檔案**，而不是安裝後的副本。

---

## 步驟 1.5：開發流程模擬

> 在評估維度之前，先把開發流程走一遍。不是真的跑 code，而是追蹤：每一步會發生什麼、呼叫什麼、context 裡有什麼、消耗多少 token。

### 模擬方法

畫出完整的開發生命週期，在每個轉換點標記：

```
[Phase] → (觸發什麼) → {context 狀態} → ~token 消耗
```

### 標準開發生命週期

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

#### Phase 3: Verification（feature group）
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
| **步驟數量** | 從 session start 到第一行 code 幾步？ | >5 步 = 過重 |
| **循環依賴** | hook A 觸發 → AI 動作 → hook A 再觸發 → ... 會不會無限迴圈？ | 任何無收斂的迴圈 |
| **死路** | 有沒有 phase 的輸出沒有任何下游消費？ | 有輸出但沒人讀 |
| **缺失鏈路** | phase N 的下一步需要什麼輸入？上一步有沒有產出？ | 需要 X 但沒人產出 X |
| **Token 比例** | 一個 task 的 ceremony token 佔總 token 多少？ | ceremony > 30% of total |
| **工具適切性** | 每步呼叫的 tool/hook 是最適合的嗎？ | 用 UserPromptSubmit hook 做應該在 PostToolUse 做的事 |
| **可觀察性覆蓋** | 每個 phase 有沒有留下可追蹤的 log/artifact？ | 有 phase 無任何記錄 = fail |

### 可觀察性覆蓋追蹤

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

### 開發流程模擬輸出

```
FLOW: Session Start → [N steps] → First Code → [20 edits, 2 hooks/edit] → Task Done → [3 ceremony] → Feature Group Complete → [action prompt → Evaluator → Review → Archive] → Compact

Token 分佈：
  Boot:       {N} tokens (X%)
  Per-call:   {N} tokens (X%)  ← 最大，無可避免
  Hooks:      {N} tokens (X%)
  Ceremony:   {N} tokens (X%)
  Verify:     {N} tokens (X%)

發現問題：
  ✓ 無循環依賴
  ✓ 無死路
  ⚠ {completion gate hook} action prompt → AI 寫 {task tracking file} → hook 再觸發（但有收斂: 第二次不會有新 action prompt 因為 Evaluator result 已存在）
```

開發流程模擬的結果會影響步驟二的評分：
- 循環依賴 → 穩健度降級
- 步驟數量過多 → 系統消耗降級
- Token 比例過高 → 系統消耗降級
- 缺失鏈路 → 遵守度降級（流程斷裂 = 規則無法執行）
- 死路 → Context 空間效率降級（產出沒人讀 = 浪費）
- 可觀察性缺口 → 遵守度降級（無法事後驗證流程是否正確執行）

---

## 步驟 1.7：假設壓力測試

> "Every component in a harness encodes an assumption about what the model can't do on its own, and those assumptions are worth stress testing." — Anthropic Engineering

對每一個元件（hook, rule, instruction, ceremony step），問：

```
這個元件假設 AI 不能自己做什麼？這個假設在當前 model 下還成立嗎？
```

### 方法

1. 列出所有元件
2. 對每個元件寫出它的**能力假設**（假設 AI 不能做 X，所以需要這個元件來做 X）
3. 用當前 model 的已知能力驗證：這個假設還成立嗎？
4. 如果不成立 → 這個元件是**過時鷹架**，應該移除或降級

### 常見過時鷹架模式

| 模式 | 假設 | 什麼時候不再成立 |
|------|------|-----------------|
| 每次 prompt 注入進度 | AI 記不住上一步做了什麼 | Model context window 夠大，且 AI 自己管狀態 |
| 強制讀取特定檔案 | AI 不會自己去找需要的資訊 | Model 有 tool use，會主動 Read |
| 步驟分解提醒 | AI 不能自己拆解任務 | Model planning 能力提升 |
| Sprint 拆解 | AI 不能維持長任務一致性 | Model 長 context 能力提升（Opus 4.6 已移除 sprint） |
| 強制 TDD 流程 | AI 不寫 test | Model 自帶 coding best practices |

### 觸發時機

- 每次 model 升級後跑一次
- eval 發現系統消耗分數下降時
- 元件的 idle rate > 50% 時

### 輸出

```
假設測試：
  ✓ {completion gate hook} — 假設: AI 會跳過 completion artifact 直接標完成。成立: 無 hard gate 時確實會跳
  ✓ {permission deny list} — 假設: AI 會嘗試危險操作。成立: 模型會嘗試 rm -rf 等
  ⚠ {tech stack detection hook} — 假設: AI 不掃 dependency files。部分成立: AI 會讀但不一定主動掃
  ✗ {component X} — 假設: [寫出假設]。不成立: [說明為何 model 現在能自己做]。建議移除
```

---

## 步驟二：評估六個維度

### 維度一：持久力

> AI 在這個系統下能持續有效工作多久才會開始退化？

評估設計對 session 持久性的影響：

| 指標 | 評估問題 | 評分標準 |
|------|----------|---------|
| **Context 預算** | 系統指令佔多少 context？剩多少給實際工作？ | always-loaded bytes / model context window. <5% = A, 5-10% = B, 10-20% = C, >20% = F |
| **使用率上限** | 系統有沒有機制讓長任務的 active context 保持在品質退化閾值以下？ | 有 compact/archive 機制讓 active context 不超過 ~40% = A, 線性增長但慢 = C, 沒有機制 = F |
| **增長上限** | 隨著工作進行，注入的 context 會不會無限增長？ | 有上限機制 (cache/archive/compact) = A, 線性增長但慢 = C, 無限制增長 = F |
| **恢復設計** | Context 重置後，系統能恢復多少狀態？ | full reset + structured handoff = A, compact 保留部分 = B, 無機制 = F |
| **恢復鏈路** | 恢復機制的每一步有觸發鏈嗎？ | 每步有明確觸發 = A, 靠 AI 記得 = C, 斷鏈 = F |
| **連續性設計** | 跨 session 時，工作能不能銜接？ | 有 handoff + state file = A, 靠 git history 推斷 = C, 完全斷裂 = F |
| **退化曲線** | 品質退化是漸進的還是懸崖式的？ | 有分級機制 (lite/full) 漸進退化 = A, 直到崩潰才處理 = F |

**計算**：Context 預算是最關鍵指標。

Model context windows（輸入）：
- Claude Opus: ~200K tokens
- Claude Sonnet: ~200K tokens  
- GPT-4: ~128K tokens
- Cursor（有效）: ~20-40K tokens per completion

公式：`context_budget_score = always_loaded_tokens / effective_context_window`

**使用率上限說明**：根據實際觀察，無論固定佔比多小，當 active context 超過視窗的 ~40% 時，輸出品質就會明顯下降（進入「Dumb Zone」）。一個只佔 5% 的 always-loaded overhead 但沒有 compact 機制的系統，在複雜任務中期仍然會撞牆。評估系統是否有機制（歸檔已完成工作、compact 完成的功能群組、結構化 handoff 開新 session）來防止 active context 達到這個上限。

### 維度二：Context 空間效率

> 在任何時刻，context 裡的東西是不是 AI 當下需要的？而且是 AI 自己不知道的？

| 指標 | 評估問題 | 評分標準 |
|------|----------|---------|
| **冗餘度** | 注入的資訊裡，有多少是 AI 自己已經知道的？ | 0% 冗餘 = A, <20% = B, 20-50% = C, >50% = F |
| **索引 vs 百科** | 指令檔是目錄（指向深層文件）還是百科（自己講完全部）？ | ~100 行 index 指向細節 = A, 混合 = C, 巨型單檔 = F |
| **信噪比** | always-loaded 內容在當前任務實際用得到多少？ | >80% 相關 = A, 50-80% = C, <50% = F |
| **及時性** | 資訊是在需要時才注入，還是提前塞？ | 大部分按需載入 = A, 混合 = C, 全部 always-loaded = F |
| **新鮮度** | 指令檔是每次 agent 失敗就更新的活文件，還是寫完就放著的靜態文件？ | 每次 agent 出現新的失敗類別就更新 = A, 偶爾由人類更新 = C, 從來不更新 = F |
| **分層** | 靜態規則、動態狀態、參考文件有沒有分層？ | 3+ 層明確分離 = A, 2 層 = C, 全部混在一起 = F |
| **去重複** | 同一概念有沒有在多處重複**定義**？ | 無重複定義 = A, 少量 = C, 嚴重 = F |

**冗餘度是最重要的指標**。其他都在問「放了什麼」，這個在問「該不該放」。

**如何評估冗餘度 — 所有權追蹤**：

對每一條動態注入（hook output）和每一段 auto-loaded instruction，追蹤完整鏈路：

```
誰創建這個資訊？ → 誰修改它？ → 誰需要它？ → 誰傳遞它？
```

分類規則：
1. **Creator = Consumer** → 中間的 deliverer 是冗餘。例: AI 改 SPEC → hook 注入 SPEC 摘要 → AI 讀摘要。AI 是 creator 也是 consumer，hook 是冗餘 deliverer
2. **已在 context** → 重複注入。例: CLAUDE.md 說「讀 SPEC」→ hook 也說「讀 SPEC」
3. **Creation chain 推導** → 如果 A 的存在必然伴隨 B，那告知 B 存在是冗餘。例: project-init 建 SPEC 時同時建 DECISION → 有 SPEC 就有 DECISION → 告知 DECISION 存在是冗餘
4. **AI 無法自己得知** → 有價值。例: tech stack 偵測（AI 不掃 package.json）

冗餘度 = (type 1 + type 2 + type 3) / 總注入量

**如何評估索引 vs 百科**：
巨型指令檔的三個死因（引用 Anthropic Engineering）：
1. 擠佔 context — 留給實際工作的空間變少
2. 無法維護 — 越大越難改，越難改越過時
3. 無法機械驗證 — 純文字指令不能被 lint 檢查

索引的特徵：主檔案 ≤ 150 行，用 progressive disclosure 指向 rules/、docs/、SPEC。
百科的特徵：主檔案 > 300 行，包含完整流程、範例、template。

**如何評估信噪比**：
讀所有 always-loaded 內容。對每個段落，問：「在一個只改一個 CSS class 的任務裡，這段話有用嗎？」計算有用 vs 總數。如果大部分內容只在複雜多 agent 場景下有用，但每個任務都載入 → 信噪比低。

**如何評估分層**：
- 第一層（always）：適用於每次互動的核心行為規則
- 第二層（conditional）：偵測到時載入的任務特定 context（例：有 SPEC → 載入 SPEC 規則）
- 第三層（on-demand）：AI 只在需要時讀取的參考文件
- 三層都存在且清楚分離 → A。全部在一個檔案裡 → F。

**如何評估去重複**：
區分**定義**和**引用**：
- **定義**：教導概念（做什麼、怎麼做、規則）。例：coding-style.md 的「Error Handling」段落含 4 個 bullet
- **引用**：順帶提及概念或指向定義。例：code-review.md 嚴重度範例「缺 error handling 導致 crash」

只有同一概念有 **2+ 個定義**才算重複。引用沒問題——那正是避免重複的方式。

只用 Grep 會過度計算（同時匹配定義和引用中的關鍵字）。需要手動分類。

### 維度三：系統消耗

> 這個系統本身消耗多少資源？相對於它帶來的價值，值不值得？

| 指標 | 評估問題 | 評分標準 |
|------|----------|---------|
| **固定成本** | 每次 API call 的固定 token 消耗 | <2000 tokens = A, 2000-5000 = B, 5000-10000 = C, >10000 = F |
| **變動成本** | 每次人類輸入後，系統附加多少 token？ | <50 tokens = A, 50-200 = C, >200 = F |
| **儀式步驟** | 完成一個任務需要多少「流程步驟」而非「寫 code」？ | <3 steps = A, 3-5 = C, >5 = F |
| **啟動流程** | Session 開始時需要讀多少檔案才能開始工作？ | 0-1 file = A, 2-3 = B, 4+ = C, 讀完還要跑命令 = F |
| **閒置元件** | 有多少元件在大多數情況下跑了但沒有產出？ | <20% = A, 20-50% = C, >50% = F |
| **觸發點設置** | 每個提醒/檢查放在最接近動作發生的觸發點嗎？ | 全部精準 = A, 多數精準 = B, 有錯放 = C, 嚴重錯放 = F |

**如何評估觸發點設置**：
每個提醒映射到最佳觸發點：
- {task tracking file} 改動相關 → PostToolUse Edit {task file}（不是每次 prompt）
- 程式碼品質 → PostToolUse Edit/Write code（不是 session 開始）
- Session 級別 → UserPromptSubmit 首次（不是每次）
- 離開前 → Stop hook 或 PreCompact

如果提醒放在 UserPromptSubmit（每次 prompt）但動作只在 Edit {task file} 時發生 → 錯放 = 浪費。

**如何評估儀式步驟**：
追蹤從「task done」到「officially done」的路徑：
- 只 commit → 0 步驟
- Commit + 更新任務檔案 → 1 步
- Commit + 更新任務 + 寫 log + 跑 verify + 標完成 → 4 步
- Commit + log + 標記 + spawn evaluator + spawn reviewer + 歸檔 → 6 步

儀式步驟越多 = 品質越一致，但消耗也越高。評分根據儀式是否與任務複雜度相稱（一刀切的儀式 = 差）。

### 維度四：遵守度

> 系統規則實際上能被遵守嗎？有沒有執行機制？

| 指標 | 評估問題 | 評分標準 |
|------|----------|---------|
| **可執行性** | 規則是機械化執行（lint/hook block）還是軟建議？ | 關鍵規則有 hard gate + error 含修復指令 = A, hard gate 但無修復指令 = B, 全是 soft = F |
| **錯誤修復指引** | Hook/linter block 時，錯誤訊息有沒有告訴 AI 具體怎麼修？ | 所有 block 訊息都含具體修復指令 = A, 部分有 = B, block 但沒有指引 = F |
| **階段分離** | 系統有沒有強制分開「規劃（研究/設計）」和「執行（寫 code）」兩個階段？ | 有明確的先規劃再執行工作流 + 人類確認點 = A, AI 自己決定什麼時候開始寫 = C, 完全沒有分離 = F |
| **可觀察性** | 能不能事後知道規則有沒有被遵守？ | 有 artifact/log 可驗證 = A, 只能看 code = C, 無法驗證 = F |
| **一致性** | 規則之間有沒有矛盾？ | 0 矛盾 = A, 有但可判斷優先級 = C, 有且無法解決 = F |
| **比例性** | 重要規則有強執行，不重要的放輕？ | 分級明確 = A, 全部同等重要 = C, 重要的反而沒 enforce = F |
| **逃生出口** | 當規則和現實衝突時，有沒有合理的跳出機制？ | 有明確的 override/PAUSE 條件 = A, 只能違反 = C, 完全沒辦法 = F |
| **交叉對應** | 指令檔說的規則和 hook 實際檢查的是同一件事嗎？ | 完全一致 = A, 有少量 phantom reference = C, 嚴重不一致 = F |

**如何評估交叉對應**：
1. 從指令檔（CLAUDE.md/rules）提取所有引用 hook 行為的語句
2. 對每條：hook 實際上做不做這件事？（phantom reference = 說了但不做）
3. 從 hooks 提取所有輸出：指令檔有沒有告訴 AI 如何回應？（unhandled output = 做了但沒說）
4. settings.json 的 deny/ask 和 CLAUDE.md 的禁止行為列表是否一致？

**如何評估可執行性 — 工具鏈映射**：

對系統中的每一條規則，追蹤它的 enforcement tool chain：

| Enforcement 工具 | 預期遵守率 | 例子 |
|-----------------|-----------|------|
| permission deny list | ~100% | rm -rf, force push, deploy |
| permission ask list | ~100% | git push, package install |
| hook exit 2 (block) | ~100% | 完成任務無 log artifact, hardcoded secret |
| hook warn + fix instruction | ~80-90% | 改 code 無 test, diff 過大 |
| hook warn（無 fix instruction）| ~60-70% | 歸檔提醒、過期 decision |
| spawned Reviewer post-check | ~90% | 重造輪子, test 覆蓋 |
| rules/instruction file soft text | ~40-70% | 系統思維, trade-off reasoning |
| 無任何 enforcement | ~20-40% | 完全靠 AI 自覺 |

**方法**：
1. 列出 rules/CLAUDE.md 中所有規則（每個 `-` bullet 或 checklist item）
2. 對每條規則標記其 enforcement tool（deny/block/warn/reviewer/soft/none）
3. 計算加權遵守率：`Σ(每條規則的預期遵守率) / 總規則數`
4. 找出**高價值但低執行力的規則**（重要但只有 soft 的規則）

**評分**：
- 加權遵守率 > 80% = A（關鍵規則都有 hard enforcement）
- 60-80% = B（多數有 enforcement）
- 40-60% = C（混合）
- < 40% = F（大多是 soft）

**重要**：不是 hard rules 越多越好。是**重要的規則要有 hard enforcement，不重要的可以 soft**。一個系統只有 10 條 hard rules 但都覆蓋關鍵操作 = A。100 條 hard rules 但 AI 被擋得無法工作 = F（過度執行）。

**如何評估錯誤修復指引**：
只說「違規」的 block 訊息會讓 AI 猜修法，通常陷入迴圈。說「違規：請改做 X」的 block 訊息是自我修復系統。檢查每一條 `exit 2` 路徑：有沒有印出具體的可執行指令？好範例：`🚫 任務標完成但無 Log 記錄。請補：- HH:MM {tid} done. 做了什麼. 驗證方式.`。差範例：`🚫 BLOCK: validation failed`。

**如何評估階段分離**：
系統有沒有明確的規劃階段，先產出計畫（optionally 讓人類確認）再開始寫 code？好的階段分離跡象：有明確的計畫文件（SPEC.md、功能清單）、執行前有人類確認點、AI 不能在計畫通過前寫 code。沒有分離的跡象：AI 在同一個 turn 裡切換規劃和寫 code、沒有任何 artifact 記錄計畫、沒有執行前確認點。Boris Tane（Cloudflare）：「永遠不要讓 agent 在你審查並批准書面計畫之前寫 code。規劃與執行的分離是我做的最重要的事。」

### 維度五：穩健度

> 系統本身會不會壞？壞了會怎樣？

| 指標 | 評估問題 | 評分標準 |
|------|----------|---------|
| **優雅降級** | 一個元件壞了，其他還能用嗎？ | 元件獨立，壞了跳過 = A, 有依賴但能 fallback = C, 連鎖崩潰 = F |
| **錯誤隔離** | 一個 hook 報錯會不會影響整個 session？ | 錯誤被 catch，session 繼續 = A, 會中斷流程 = F |
| **狀態一致性** | Source of truth 有幾個？會不會衝突？ | 單一 source of truth = A, 多個但有同步 = C, 多個且會 drift = F |
| **空狀態** | 什麼都沒有時（新 repo, 無 SPEC, 無 DECISION）能正常工作嗎？ | 完全正常 = A, 部分功能可用 = C, 報錯或卡住 = F |
| **中途變更** | 人類中途改了 code/需求/config，系統能適應嗎？ | 有偵測+調整機制 = A, 靠 AI 判斷 = C, 會衝突 = F |
| **版控即紀錄** | AI 需要的所有知識都在版控的檔案裡嗎？ | 全部在 repo/config = A, 部分在外部 (Slack/Docs) = C, 關鍵知識不在 repo = F |
| **熵防禦** | 有沒有機制防止 AI 複製壞模式，且定期清理 AI 生成的低品質 code？ | 有 lint + 結構測試 + 定期 GC agent 清理 AI slop = A, 只有 lint = B, 沒有 = F |
| **依賴存活性** | 所有被引用的元件還活著嗎？ | 0 dead ref = A, 有但非關鍵 = C, 關鍵依賴死了 = F |

**如何評估版控即紀錄**：
「不在倉庫裡的東西，對智能體來說不存在」— Anthropic Engineering
1. 列出 AI 做決策需要的所有知識（架構、規範、計劃、決策記錄）
2. 每項：在 repo 的版控檔案裡嗎？還是只在 Slack/Google Docs/人腦裡？
3. 全域指令（~/.claude/）算 repo 嗎？→ 如果有版控的 harness repo 且 install/symlink 機制 = 算。如果手寫沒備份 = 不算。

**如何評估熵防禦**：
兩個不同問題，都需要處理：

**問題 A：AI 複製壞模式** — AI 會模仿 repo 裡已有的東西，包括既有的 bug 和反模式。
1. 有沒有 lint/structural test 保護不變量？（不只是 style，是結構：「每個 API 必須有 error response」）
2. 有沒有背景任務掃描偏差？（{session hook} 偵測重複修改、stale decisions）
3. lint/hook error message 有沒有內嵌修復指令，讓 AI 能自我糾正？

**問題 B：AI 生成的 slop 累積** — LLM 生成的 code 傾向於重複實現已有功能、過度設計、可讀性低。這種技術債的累積方式和人類寫的不一樣。
1. 有沒有定期跑的「垃圾回收」agent 或任務，掃描重複實現、dead code、過度設計的模式？
2. 清理吞吐量有沒有跟生成吞吐量成比例？（每天產生 100 行 code，清理也應該每天跑，不是每季跑）
3. 清理跑次有沒有記錄（找到什麼、刪了什麼）？

只處理問題 A → B。兩個都處理 → A。

**如何評估依賴存活性**：
1. hooks：grep source/bash 呼叫的其他 script → 檢查存在且非空
2. 指令檔：grep 引用的 docs/hooks/路徑 → 檢查存在
3. settings.json：註冊的 hook 路徑 → 檢查存在且非空
4. docs/：被指令檔引用的 on-demand 檔案 → 檢查存在

**如何評估優雅降級**：
畫出所有元件的依賴圖。對每個元件：
- 如果它失敗，還有什麼會壞？
- 有沒有 `|| true`、`2>/dev/null` 或 fallback？
- 它是 block（exit 2）還是 warn（echo）？

單點故障（一個元件失敗讓整個 session 崩潰）= 自動 F。

### 維度六：協作品質

> 人類和 AI 之間的互動設計好不好？

| 指標 | 評估問題 | 評分標準 |
|------|----------|---------|
| **對的時間給對的資訊** | 系統會不會在 AI 需要時才給資訊？ | 有狀態感知的注入 = A, 固定注入 = C, 不注入讓 AI 自己找 = F |
| **噪音程度** | 多少系統輸出是人類/AI 不需要看的？ | 只在有 actionable info 時輸出 = A, 有 cache 防重複 = B, 每次都輸出 = F |
| **自主分級** | 什麼時候該自動做、什麼時候該問人類？ | 有明確分級（deny/ask/allow）= A, 模糊 = C, 全自動或全手動 = F |
| **透明度** | AI 在做什麼，人類看得到嗎？ | 有 progress tracking + event log = A, 有但分散 = C, 黑箱 = F |
| **多 Agent 設計** | 多個 AI agent 之間能不能協調？ | 有協調協議 + 衝突解決 = A, 簡單分工 = C, 沒考慮 = F |
| **Agent 可讀性** | codebase 對 AI 可讀嗎？用的技術 AI 熟悉嗎？ | 用 "boring" tech（API 穩定、訓練集覆蓋）+ worktree 可隔離 = A, 混合 = C, 用 AI 不熟的框架 = F |
| **失敗分析** | 出問題時，系統引導「缺什麼 context/tool/constraint」還是「再試一次」？ | 有明確的失敗診斷流程 = A, 靠 AI 判斷 = C, 無引導 = F |
| **設定對齊** | 權限設定和行為指令一致嗎？ | settings deny/ask 和 CLAUDE.md 禁止行為完全對應 = A, 有差異但不矛盾 = B, 有矛盾 = F |

**如何評估 Agent 可讀性**：
「優先選擇 boring 技術」— API 穩定、訓練集覆蓋好的框架 AI 更容易正確使用。
1. 主要框架是 AI 訓練集常見的嗎？（React/Express/FastAPI = boring ✓, 自研框架 = risky）
2. 有沒有 opaque upstream 的 wrapper？（有時重新實現子集比包裝不透明的上游行為更划算）
3. 應用能不能按 git worktree 啟動隔離實例？（AI 可以啟動自己的測試環境）

**如何評估失敗分析**：
「人類時間是最稀缺的資源。出問題時，答案不是更努力，而是缺什麼 context/tool/constraint」
1. 系統有沒有 PAUSE 機制（遇到不確定 → 停下來問）？
2. hook block 時有沒有告訴 AI 具體缺什麼 + 怎麼修？
3. 重複失敗時有沒有升級機制（而不是一直重試）？

**如何評估設定對齊**：
1. 列出 {permission config} 的 deny 清單 → {instruction file} 禁止行為有沒有對應？
2. 列出 {instruction file} 的禁止行為 → {permission config} 有沒有用 deny 或 ask 支撐？
3. 如果 instruction file 說「不 push」但 permission config 沒有封鎖 → 軟規則無硬支撐
4. 如果 permission config deny 了某操作但 instruction file 沒提 → AI 不理解為什麼被擋

---

## 步驟 2.5：搬移影響檢查

> 當 eval 在步驟四建議搬移內容時，必須先跑此檢查。

搬移前三問：
1. **誰引用？** — grep 內容關鍵字 across all files，找到所有引用者
2. **目標有觸發嗎？** — 如果目標是 docs/（on-demand），AI 什麼時候知道要讀？沒觸發 = 等於不存在
3. **來源更新了嗎？** — 搬移後，原本引用這段內容的地方（CLAUDE.md, hooks）是否已更新？

失敗模式：為了優化 Token/Space 把內容移到 docs/，但沒有觸發機制讓 AI 知道什麼時候讀 → 內容等於不存在。

---

## 步驟三：評分與報告

### 評分規則

對每個維度，對各指標分數求平均：
- A（優秀）= 4, B（良好）= 3, C（及格）= 2, D（差）= 1, F（不及格）= 0, N/A = 跳過

維度分數：各指標平均 → 映射回字母等級
- ≥3.5 = A, ≥2.5 = B, ≥1.5 = C, ≥0.5 = D, <0.5 = F

**總分** = 加權平均：
- 穩健度 ×2（系統崩潰，其他都無意義）
- 持久力 ×1.5（決定最大有效工作量）
- 遵守度 ×1.5（決定工作品質）
- Context 空間效率 ×1, 系統消耗 ×1, 協作品質 ×1

### 輸出格式

```
═══════════════════════════════════════════════════════════
  AI 開發系統評估報告
  系統：{name}
  元件：{N} 個指令檔案, {M} 個 hooks, {K} 個狀態檔案
  總 always-loaded：{bytes} bytes (~{tokens} tokens)
═══════════════════════════════════════════════════════════

┌─────────────────────┬───────┬─────────────────────────────┐
│ 維度                │ 評分  │ 關鍵發現                    │
├─────────────────────┼───────┼─────────────────────────────┤
│ 1. 持久力           │ {A-F} │ {一行說明}                  │
│ 2. Context 空間效率 │ {A-F} │ {一行說明}                  │
│ 3. 系統消耗         │ {A-F} │ {一行說明}                  │
│ 4. 遵守度           │ {A-F} │ {一行說明}                  │
│ 5. 穩健度           │ {A-F} │ {一行說明}                  │
│ 6. 協作品質         │ {A-F} │ {一行說明}                  │
├─────────────────────┼───────┼─────────────────────────────┤
│ 總分                │ {A-F} │                             │
└─────────────────────┴───────┴─────────────────────────────┘
```

### 各維度詳細說明

對每個維度，輸出：
1. 各指標評分表（含分數與佐證）
2. 最強點（設計做得好的地方）
3. 最弱點（最大的設計缺陷）
4. 一個具體可行的修復建議

### 比較說明

如果評估有 hooks + 狀態管理的系統（例如 harness）：
- 與基準比較：「同一個專案只有 CLAUDE.md 沒有 hooks」
- 增加的複雜度值得換來的控制力嗎？

如果評估最小化系統（例如只有 .cursorrules）：
- 指出缺少但有價值的部分
- 指出簡單帶來的好處（更低消耗、更少可壞之物）

---

## 步驟四：建議

最多 5 條，優先順序：
1. 穩健度修復（系統損壞）
2. 持久力修復（AI 無法持續工作）
3. 消耗降低（浪費金錢/context）
4. 遵守度提升（品質有風險）
5. Context/協作優化（錦上添花）

每條建議：
- 改什麼（具體說明）
- 為什麼（改善哪個指標）
- 代價（放棄什麼）

---

## 參考：系統類型

常見 AI 開發系統模式及其典型分數：

| 類型 | 範例 | 持久力 | Context | 消耗 | 遵守度 | 穩健度 | 協作 |
|------|------|--------|---------|------|--------|--------|------|
| **裸機** | 完全無設定 | A | F | A | F | A | F |
| **單檔案** | 一個 CLAUDE.md | B | C | B | D | A | D |
| **規則型** | .cursorrules + rules/ | B | B | B | D | A | C |
| **Hook 增強** | CLAUDE.md + hooks | C | B | C | B | C | B |
| **完整 harness** | SPEC + hooks + multi-agent | D | B | D | A | C | A |
| **過度工程** | 全部 + 廚房水槽 | F | D | F | B | D | B |

最佳點取決於專案複雜度：
- 個人週末專案 → 單檔案或規則型
- 團隊專案，1-3 個月 → Hook 增強
- 複雜多 agent、長期運行 → 完整 harness
- 如果你的消耗分數比遵守度分數差 → 你已經過度工程了
