# Multi-Agent TDD Workflow 設計報告

## 1. 設計目標

建立一個可恢復、可追蹤、嚴格遵守 TDD 的流程：

1. 使用者提供 `spec`。
2. `tdd-plan` skill 讀取 spec。
3. 產生可審查的 task 與 test case 文件。
4. 使用者切換至 `tdd-orchestrator` agent。
5. Orchestrator 逐一處理 task。
6. 每個 task 依序執行多個 vertical slices：
   - 選擇一個 test case
   - `tdd-test-writer` 撰寫一個測試
   - 確認測試先失敗
   - `tdd-implementor` 進行最小實作
   - 執行 focused test
7. 單一 test case 的 test-writing 與 implementation 階段各最多允許 5 次 attempts。
8. 前一個 task 完成後，才能開始下一個。
9. 全部完成後產生結果報告，交由使用者 review。

核心原則：

- 不依賴 agent 對話記憶，所有狀態寫入文件。
- 不允許 task 平行執行，避免互相影響。
- 測試與實作由不同 agent 負責。
- 測試一律透過事先確認的 public seam 驗證行為。
- 失敗必須保留完整診斷資訊。
- 遇到無法安全判斷的問題時停止，而不是自行猜測。

## 2. 建議的元件

### Skills

```text
tdd-plan
tdd-run
```

### Agents

```text
tdd-orchestrator
tdd-test-writer
tdd-implementor
```

職責應明確分離：

| 元件               | 職責                                              |
| ------------------ | ------------------------------------------------- |
| `tdd-plan`         | 讀取 spec 或 tickets，建立 tasks，設計 test cases |
| `tdd-run`          | 載入計畫並執行整個流程                            |
| `tdd-orchestrator` | 控制狀態、呼叫 subagent、執行測試、處理失敗       |
| `tdd-test-writer`  | 只撰寫或修改測試檔                                |
| `tdd-implementor`  | 只修改 production code，讓測試通過                |

`skill` 負責流程規則，`agent` 負責實際工作。

## 3. Artifact Directory

建議所有生成文件集中放在：

```text
.tdd/
├── tasks/
│   ├── TASK-001.md
│   └── TASK-002.md
├── state.yaml
└── final-report.md
```

### 文件用途

- `tasks/TASK-XXX.md`：單一 task 的完整規格，包含 acceptance criteria 與 test cases。
- `state.yaml`：orchestrator 的持久化狀態、test case status 與 retry 計數器。
- `final-report.md`：每次 workflow 執行後產生的使用者 review 報告。

## 4. `tdd-plan` Skill

### 使用方式

`tdd-plan` 接受 spec、ticket 或兩者作為 input，並自動解析輸入類型。

```text
/tdd-plan <input>
```

skill 應解析使用者提供的 input，判斷其中是 spec、ticket 或兩者的內容。若 input 無法讀取或內容不足，skill 應停止並要求使用者提供有效 input。

### 執行流程

1. 讀取使用者提供的 spec 或 ticket source。
2. 讀取專案結構與測試框架。
3. 如果存在，讀取 `CONTEXT.md` 與相關 ADR，沿用專案的 domain vocabulary。
4. 找出既有測試命令。
5. 如果輸入是 ticket，保留 ticket 的 task scope 與 `Blocked by` dependencies；只有在 ticket 明顯大到無法放入一次新的 implementation context 時才重新拆解，並記錄原因。
6. 如果輸入是 spec，將需求拆成最小可交付 task。
7. 為每個 task 建立 test cases。
8. 為每個 task 明確定義 public seam 與 observable behavior。
9. 建立 task dependency graph；ticket 的 `Blocked by` 可直接作為 dependencies。
10. 產生：

- `.tdd/tasks/TASK-XXX.md`（每個 task 一份）

11. 執行靜態驗證。
12. 停止，等待使用者自行啟動 `tdd-run`。

這個 skill 不應直接修改 production code，也不應自動啟動 orchestrator。

### Ticket 輸入規則

- Ticket 的 acceptance criteria 作為 task scope。
- Ticket 的 `Blocked by` 作為 task dependencies。
- 同時提供 spec 與 ticket 時，ticket 優先定義 task scope，spec 提供補充上下文。
- spec 與 ticket 衝突時，以 ticket 為 task scope，並在 plan 或 final report 中記錄衝突。

### Task 拆解規則

每個 task 必須：

- 能在一個短迭代內完成。
- 有明確 acceptance criteria。
- 有至少一個對應測試。
- 有明確且經使用者確認的 public seam。
- 每個 test case 都能形成一個獨立的 vertical slice。
- 不與其他 task 共享不必要的修改範圍。
- 有清楚的前置依賴。
- 能獨立判斷完成或失敗。

## 5. Task 文件格式

每個 task 使用一份獨立文件，例如 `.tdd/tasks/TASK-001.md`。這份文件同時包含 task specification 與該 task 的所有 test cases，讓每一輪 TDD 只需要讀取目前 task 文件。

一個 task 可以包含多個相關的 test cases。test case 代表一個測試情境，也是實際執行的 implementation 與 retry 單位；每個 test case 都是一個 vertical slice。`tdd-plan` 可以先列出候選 test cases，但 `tdd-test-writer` 不得一次寫完所有測試，必須依序一次寫一個。

```markdown
---
id: TASK-001
title: Validate payment request
order: 1
dependencies: []
---

# TASK-001: Validate payment request

### Goal

Reject payment requests with invalid amount or currency.

### Dependencies

- None

### Scope

- `src/payment/validation.ts`
- `src/payment/validation.test.ts`

### Acceptance Criteria

- Amount must be greater than zero.
- Currency must be one of the supported currencies.
- Invalid input returns a typed validation error.
- Valid input is accepted.

### Public Seam

`validatePaymentRequest(request)`

### Observable Behavior

Returns either an accepted result or a validation error through the public function result.

### Test Cases

#### TC-001: Reject zero amount

- status: PENDING
- max_attempts: 5
- test_location: `src/payment/validation.test.ts`

**Given:** A payment request with amount `0`.

**When:** The validation function is called.

**Then:** A validation error is returned and identifies `amount`.

#### TC-002: Reject unsupported currency

- status: PENDING
- max_attempts: 5
- test_location: `src/payment/validation.test.ts`

**Given:** A payment request with an unsupported currency.

**When:** The validation function is called.

**Then:** A validation error is returned and identifies `currency`.

#### TC-003: Accept valid payment

- status: PENDING
- max_attempts: 5
- test_location: `src/payment/validation.test.ts`

**Given:** A payment request with a positive amount and supported currency.

**When:** The validation function is called.

**Then:** The request is accepted.

### Completion Definition

- All mapped tests pass.
- Full regression suite passes.
- No unrelated files are modified.
```

每個 task file 內的 test case 至少包含：

- ID
- `status: PENDING`
- `max_attempts: 5`
- test location
- given
- when
- then
- 預期 test file 路徑

本 workflow 只允許 unit test，不寫 integration 或 e2e test case。

### 為什麼採用一 task 一檔案

- orchestrator 每輪只需載入目前 task 的完整上下文。
- 減少跨文件 ID 對照錯誤。
- 使用者可以獨立 review、修改或重跑單一 task。
- 失敗時容易定位對應的 specification、test cases 與結果。
- 未來可以讓不同 task 使用不同 agent session，而不改變文件契約。

task 的 order 與 dependencies 放在各自的 front matter，test case 的 `max_attempts` 放在 task 文件內。orchestrator 啟動時讀取所有 task 的 front matter，建立執行順序；每一輪只讀取 `.tdd/tasks/<current-task-id>.md` 的完整內容。

## 6. Task Metadata 與 Runtime Settings

task file 的 front matter（`id`、`title`、`order`、`dependencies`）是 task metadata 的 canonical source。測試命令由 `tdd-run` 根據專案設定自動偵測，並記錄在 `final-report.md`。

Plan skill 必須驗證：

- task ID 唯一。
- test case ID 唯一。
- 所有 task file 都有合法 front matter。
- 所有 dependency 都對應到存在的 task file。
- dependency graph 沒有 cycle。
- task order 符合 dependencies。
- 每個 task 至少有一個 test case。
- 每個 test case 的 `max_attempts` 不得超過 5，且預設為 5。
- 專案中存在可偵測的測試命令，或使用者已提供測試命令。

## 7. Orchestration State Machine

### Workflow 狀態

```text
PLANNED
  ↓
RUNNING
  ↓
COMPLETED
```

異常狀態：

```text
BLOCKED
FAILED
```

### Task 狀態

```text
PENDING
  ↓
IN_PROGRESS
    ↓
    PASSED
```

異常狀態：

```text
BLOCKED
FAILED
```

### Task 狀態說明

| 狀態        | 含義                                                                         |
| ----------- | ---------------------------------------------------------------------------- |
| PENDING     | 尚未開始，等待前置 task 完成                                                 |
| IN_PROGRESS | 正在處理 test cases 或執行 task-level regression / typecheck / build          |
| PASSED      | 所有 test cases 通過，regression、typecheck 與 build 通過                   |
| BLOCKED     | baseline 失敗、rollback 無法安全完成，或 dependency 失敗                  |
| FAILED      | 有 test case 失敗，但 workflow 繼續執行其他 task                             |

### Test Case 狀態

```text
PENDING
  ↓
WRITING_TEST     (test writer 階段，最多 5 次重寫)
  ↓
TEST_RED         (確認有效 Red)
  ↓
IMPLEMENTING     (implementor 階段，最多 5 次實作)
    ↓
    PASSED
```

異常終止狀態：

```text
TEST_AUTHORING_FAILED   (test writer 重寫 5 次仍無有效 Red)
IMPLEMENTATION_FAILED   (implementor 實作 5 次仍不通過)
```

### Test Case 狀態說明

| 狀態                  | 含義                                                                    |
| --------------------- | ----------------------------------------------------------------------- |
| PENDING               | 尚未開始，等待前置 test case 完成                                       |
| WRITING_TEST          | test writer 正在撰寫/重寫測試，累計 `writer_attempts`                    |
| TEST_RED              | 測試已確認有效 Red（測試失敗且失敗原因是 production behavior 尚未實作） |
| IMPLEMENTING          | implementor 正在實作，累計 `implementor_attempts`                       |
| PASSED                | 測試通過，進入下一個 test case                                          |
| FAILED                | 超過 retry 上限；implementation failure 會 rollback 並 block 同 task cases |
| BLOCKED               | 被 implementation failure 或 failed dependency 阻擋                     |

### `state.yaml`

```yaml
version: 2
workflow_status: RUNNING
current_task: TASK-001
current_test_case: TC-001

tasks:
  TASK-001:
    status: IN_PROGRESS
    test_cases:
      TC-001:
        status: WRITING_TEST
        writer_attempts: 2
        implementor_attempts: 0

  TASK-002:
    status: PENDING
    test_cases:
      TC-003:
        status: PENDING
        writer_attempts: 0
        implementor_attempts: 0
```

每次狀態轉移都應立即寫入 `state.yaml`，避免 agent 中斷後失去進度。`state.yaml` 只保存恢復執行所需的狀態；command 結果與詳細 failure 寫入 final report。使用者中止 session 時不建立 `ABORTED` 狀態；若狀態與檔案結果無法安全判斷，應先人工檢查再恢復。

- `writer_attempts`：test writer 在 Red Confirmation 階段的重寫次數（max 5）。
- `implementor_attempts`：implementor 在實作階段的嘗試次數（max 5）。
- `attempts: 0` 表示尚未呼叫對應 agent。

## 8. `tdd-run` Skill

### Recovery Mode

`tdd-run` 支援三種執行模式：

- `continue`：讀取 `.tdd/state.yaml` 繼續；已完成、失敗與 blocked 項目依現有狀態處理。
- `rerun-failed-only`：重試 `FAILED` test cases，以及同一 task 中因 implementation failure 而被 block 的 test cases；保留所有 `PASSED` cases。
- `reset-all`：保留 task files，但重新建立 runtime state，重跑整份 task plan。

前一輪存在 `FAILED` 或 `BLOCKED` 項目且使用者未指定 mode 時，orchestrator 必須先詢問，不得默默重置 state。

`rerun-failed-only` 的 reset 規則：

1. 將要重試的 test case 設為 `PENDING`。
2. 清除 `failure_reason`，並將 `writer_attempts` 與 `implementor_attempts` 歸零。
3. 保留所有 `PASSED` test cases 與既有 production changes。
4. 依賴失敗而被 block 的 task，等 dependency `PASSED` 後才可重新執行。
5. 將 reset 後的 state 立即寫入，再進入正常 Red-Green 流程。

### 啟動前檢查

Orchestrator 啟動後先執行：

1. 確認存在 `.tdd/tasks/` 與 `.tdd/state.yaml`。
2. 讀取所有 task file 的 front matter，建立合法的執行順序。
3. 偵測或讀取測試命令。
4. 執行 baseline test。
5. 如果 baseline 已失敗，立即停止。

Baseline test 已失敗時不能把問題歸咎於目前 task，否則會污染診斷結果。

## 9. 單一 Task 的完整流程

一個 task 可以包含多個 test cases，但必須逐一執行。每個 test case 都是一個 vertical slice：一個測試、一個最小實作、一次 focused verification。

### Phase A: Select Test Case

Orchestrator 從目前 task 選擇下一個 `pending` test case，並確認：

- test case 使用已確認的 public seam。
- test case 描述的是外部可觀察行為。
- test case 尚未完成，且 `attempts < max_attempts`。

### Phase B: Test Writing

Orchestrator 呼叫 `tdd-test-writer`，每次只提供一個 test case：

- task 文件
- 目前 test case
- 已確認的 public seam
- production code 相關檔案
- 既有測試慣例
- 禁止修改 production code 的規則

`test-writer` 只能：

- 新增或修改 test files。
- 使用既有 test framework。
- 覆蓋 acceptance criteria。
- 只透過已確認的 public seam 測試 observable behavior。
- 使用來自 spec、worked example 或獨立已知結果的 expected values。
- 避免 implementation-coupled、tautological 與 side-channel tests。
- 確保測試 deterministic。

`test-writer` 不得：

- 修改 production code。
- 為了讓測試通過而放寬 assertion。
- 修改 task 或 acceptance criteria。
- 執行大範圍重構。

### Phase C: Red Confirmation

Orchestrator 執行目前 test case 的 focused test 並確認有效 Red。

期望結果：

```text
測試失敗，且失敗原因是 production behavior 尚未實作
```

以下情況視為 **authoring failure**，觸發 test writer 重寫：

- 測試直接通過且不是有效的行為 assertion。若行為已由前一 slice 實作，記錄 `already_green` 並視為通過。
- 測試因 syntax、import 或 environment error 失敗。
- 測試沒有驗證 acceptance criteria。
- 測試 failure 與 task 無關。

處理流程：

1. `writer_attempts++`
2. 若 `writer_attempts < 5`：回 Phase B 讓 test writer 更新同一個 test identity；不得留下上一版失效測試
3. 若 `writer_attempts == 5`：標記 test case 為 `FAILED`，原因 `TEST_AUTHORING_FAILED`，記錄失敗摘要，繼續下一個 test case

這一步可避免流程變成「先寫一個永遠通過的測試，再假裝完成」。

### Phase D: Implementation

Orchestrator 呼叫 `tdd-implementor`，提供：

- task specification
- acceptance criteria
- 目前的 test case ID
- 測試失敗輸出
- 修改範圍限制

`implementor` 只能修改 production code，不得修改測試檔案。若測試錯誤，應標記 `TEST_AUTHORING_FAILED`，交由 test-writer 或使用者修正。

實作原則：

1. 先做最小實作讓測試通過。
2. 不提前實作未要求的功能。
3. 不修改測試 assertion 以掩蓋錯誤。
4. 保持既有 API 與 coding convention。
5. 不在這個 loop 中進行 refactor。

### Phase E: Focused Verification

每次 implementation 後：

1. 執行目前 test case 的 focused test。
2. 保存簡單狀態至 `state.yaml`，詳細測試結果寫入 final report。
3. 通過後將目前 test case 標記為 `PASSED`，再選擇下一個 test case。
4. 第 5 次 attempt 仍失敗時，rollback 該 test case 的 production changes；若 rollback 範圍無法安全判斷，將目前 test case 與 task 標記為 `BLOCKED` 並停止該 workflow path。若 rollback 成功，將目前 test case 標記為 `FAILED`，並將同一 task 剩餘 test cases 標記為 `BLOCKED`。

所有 runnable test cases 處理完成後，執行 task-level regression、typecheck 與 build。全部通過且所有 test cases 為 `PASSED` 才將 task 標記為 `PASSED`；有任何 `FAILED` 或 `BLOCKED` test case 則標記為 task `FAILED`。

### Phase F: Final Checks

所有 runnable tasks 完成後，執行一次 full regression。任何 task、test case 或 final check 失敗時，workflow 標記為 `FAILED`。Refactor 留給 workflow 外的人工 review。

## 10. 五次失敗規則（兩階段各五次）

每個 test case 有兩組獨立計數器：

| 階段                            | Agent           | 計數器                     | 上限 | 失敗後標記              |
| ------------------------------- | --------------- | -------------------------- | ---: | ----------------------- |
| Test Writing (Red Confirmation) | tdd-test-writer | `writer_attempts`          |    5 | `TEST_AUTHORING_FAILED` |
| Implementation                  | tdd-implementor | `implementor_attempts`     |    5 | `IMPLEMENTATION_FAILED` |

兩階段各自獨立計數，互不影響。

計數器定義：

```text
writer_attempts = 0..5  (test writer 重寫測試的次數)
implementor_attempts = 0..5  (implementor 實作的次數)
```

Test Writing 階段：

```text
test writer 完成一次測試撰寫/重寫
→ focused test
→ 若無有效 Red → attempts++，重試
→ 若 attempts = 5 → TEST_AUTHORING_FAILED
```

Implementation 階段：

```text
implementor 完成一次修改
→ focused test
→ 若不通過 → attempts++，重試
→ 若 attempts = 5 → IMPLEMENTATION_FAILED
```

第五次仍失敗時：

```text
test case status = FAILED
failure_reason = TEST_AUTHORING_FAILED | IMPLEMENTATION_FAILED
remaining test cases in task = BLOCKED
implementation changes = ROLLED_BACK
```

Test authoring failure 後可繼續同一 task 的下一個 test case；implementation failure 必須 rollback 並 block 同一 task 的剩餘 test cases。Independent tasks 可以繼續，依賴 failed task 的 tasks 標記為 `BLOCKED`。所有失敗項目會列在最終的 `.tdd/final-report.md`。

`BLOCKED` 用於 baseline 失敗、rollback 無法安全完成，或 task dependency 失敗。

## 11. Task 順序與隔離

為符合「一個 task 做完才能接下一個」：

- 只允許一個 active task。
- `tdd-test-writer` 和 `tdd-implementor` 不平行執行。
- 下一個 task 必須等其 dependencies 完成；failed dependency 會使 task `BLOCKED`。
- 每個 task 通過後保存 task-level git checkpoint；checkpoint 只包含該 task 的 production/test 變更。
- 有 `FAILED` 或 `BLOCKED` test case 的 task 標記為 `FAILED`；independent task 可以繼續。

建議 task completion checkpoint：

```yaml
status: PASSED
test_cases:
  TC-001:
    status: PASSED
    writer_attempts: 1
    implementor_attempts: 1
```

`status` 可為 `PASSED`、`FAILED` 或 `BLOCKED`；`FAILED` task 不會中斷 independent tasks。

Checkpoint 規則：若 repository 是 git worktree，orchestrator 會依 repository 的 contribution guidance 與近期 commit history 使用既有 commit message convention，並只 stage 該 task 的 production/test 變更。`.tdd/` 與無關的既有變更不得加入 checkpoint。非 git worktree 會跳過 checkpoint；stage 或 commit 失敗時保留 task `PASSED`，並將原因記錄在 final report，不擴大 stage 範圍或繞過 hooks。

## 12. Agent Contract

### `tdd-orchestrator`

允許：

- 讀取所有 workflow 文件。
- 呼叫 subagents。
- 執行測試、typecheck、build。
- 更新 `.tdd/state.yaml` 與測試結果摘要。
- 產生 final report。

禁止：

- 直接實作 production code。
- 自行修改測試來消除 failure。
- 跳過 Red phase。
- 執行被 failed dependency block 的 task。
- 平行啟動多個 implementation task。

### `tdd-test-writer`

輸入：

```text
- task 文件（含 acceptance criteria、public seam、test cases）
- 目前 test case（id、given/when/then、writer_attempts）
- 已確認的 public seam
- production code 相關檔案
- 既有測試慣例與 test framework
- 禁止修改 production code 的規則
- 目前 writer_attempts 次數
```

輸出：

```text
目前 test case 對應的 test file
test name and public seam
expected red behavior
```

### `tdd-implementor`

輸入：

```text
- task 文件
- 目前 test case（id、given/when/then、implementor_attempts）
- 已確認的 public seam
- 測試失敗輸出（focused test 失敗訊息）
- production code 相關檔案
- 目前 implementor_attempts 次數
```

輸出：

```text
目前 test case 對應的 production files
implementation summary
remaining concerns
```

## 13. 最終結果報告

完成或停止時，orchestrator 必須產生 `.tdd/final-report.md`，並將相同內容回報給使用者。

內容建議：

```markdown
# TDD Run Report

## Summary

- Run ID: `run-20260822-001`
- Input: `<spec-or-ticket-reference>`
- Status: `COMPLETED`
- Started At: `2026-08-22T10:00:00Z`
- Finished At: `2026-08-22T10:45:00Z`

## Overall Result

- Tasks: 2
- Passed Tasks: 2
- Failed Tasks: 0
- Test Cases: 5
- Passed Test Cases: 5
- Failed Test Cases: 0

## Task Results

### TASK-001: Validate payment request

- Status: `PASSED`
- Public Seam: `validatePaymentRequest(request)`
- Test Cases: 2/2 passed

| Test Case | Status | Test Writer Attempts | Implementor Attempts | Max Attempts (each) |
| --------- | ------ | -------------------: | --------------------: | ------------------: |
| TC-001    | PASSED |                    1 |                    1 |                   5 |
| TC-002    | PASSED |                    1 |                    3 |                   5 |

### TASK-002: Create payment request

- Status: `PASSED`
- Public Seam: `createPaymentRequest(input)`
- Test Cases: 1/1 passed

| Test Case | Status | Test Writer Attempts | Implementor Attempts | Max Attempts (each) |
| --------- | ------ | -------------------: | --------------------: | ------------------: |
| TC-003    | PASSED |                    1 |                    2 |                   5 |

## Files Changed

- `src/payment/validation.ts`
- `src/payment/validation.test.ts`

## Test Results

| Check           | Status | Command                  |
| --------------- | ------ | ------------------------ |
| Baseline Test   | PASSED | `npm test`               |
| Focused Tests   | PASSED | `<focused-test-command>` |
| Task Regression | PASSED | `<task-test-command>`    |
| Full Regression | PASSED | `npm test`               |
| Typecheck       | PASSED | `npm run typecheck`      |
| Build           | PASSED | `npm run build`          |

## Failures

None.

## Review Required

- Review production code changes.
- Review test coverage at the confirmed public seams.
- Confirm no implementation-coupled tests were added.
- Consider whether additional edge cases are required.

## Known Limitations

- `<known limitation>`
```

失敗時的 test case 應保留清楚的 attempts 結果：

```markdown
## Failures

### TC-001: Reject zero amount

- Status: `FAILED`
- Test Writer Attempts Used: 5
- Implementor Attempts Used: 0
- Max Attempts (each): 5
- Last Command: `<focused-test-command>`
- Last Failure: `<failure-summary>`
- Next Action: Resolve the failure, then run `/tdd-run rerun-failed-only` or ask the orchestrator to retry failed cases
```

Workflow 不會因 test case 失敗而停止；所有失敗項目都會列在 Failures 區塊，並在最後回報給使用者。

報告必須區分：

- agent 自動完成的內容。
- 測試實際驗證的內容。
- 尚未驗證的內容。
- 需要使用者決策的內容。

## 14. 人工 Review

Workflow 完成後，使用者檢查：

- production diff。
- test diff。
- final report。
- known limitations。
- 是否需要補充 test cases。

## 15. 安全與可靠性

工具契約包含以下限制：

- 不存取 network、secrets、`.env` 或 credentials，除非使用者明確授權目前 task 的狹義例外。
- 不執行 destructive commands。
- task 通過後可自動建立 task-level git checkpoint；只 stage 該 task 的 production/test 變更，不 stage `.tdd/` 或其他既有變更。
- 不自動 push 或建立 PR。
- 不使用 `git reset --hard`。
- 修改檔案前確認 scope。
- 所有 command、exit code 與測試結果摘要都保存至 final report；完整 stdout/stderr 僅保留在目前 session。
- 所有 agent 回報必須結構化，不只依賴自然語言。
- rollback 範圍無法安全判斷時標記 `BLOCKED`，不得猜測或擴大 rollback 範圍。

## 16. 建議的最小可行版本

第一版建議先實作：

1. `tdd-plan` skill。
2. `.tdd/tasks/TASK-XXX.md`。
3. `tdd-orchestrator` agent。
4. `tdd-test-writer` agent。
5. `tdd-implementor` agent。
6. focused test、task regression 與 full regression。
7. 每個 test case 有兩階段各 5 次：test writer 重寫 5 次、implementor 實作 5 次。
8. `state.yaml` resume 與測試結果摘要。
9. 產生 `.tdd/final-report.md` 並回報給使用者。

## 17. 完整流程

```text
User provides spec or ticket
      |
      v
Run tdd-plan skill
      |
      v
Generate task files
      |
      v
Switch to tdd-orchestrator
      |
      v
Validate baseline
      |
      v
For each task, sequentially:
      |
      +--> select next test case
      |        |
      |        v
+--> tdd-test-writer
       |        |
       |        v
       |     Confirm RED
       |        |
       +--  no  +-- test_writer_attempts < 5 --> retry test writer
       |        |
       |        +-- test_writer_attempts = 5 --> Mark test case FAILED (TEST_AUTHORING_FAILED), continue
       |        |
       |        v
       +--> tdd-implementor
                |
                v
           Run focused test
                |
        pass? --+-- no & implementor_attempts < 5 --> retry
                |
                +-- no & implementor_attempts = 5 --> Rollback; mark FAILED and block remaining cases
                 |
                 v
   Mark test case PASSED
         |
         v
   More test cases?
      | yes
      +-------> next slice
      |
      | no
      v
 Run task regression
      |
      v
Run full regression
      |
      v
Write .tdd/final-report.md
      |
      v
User reviews diff and report
```

## 結論

最重要的設計決策是：

1. `tdd-plan` 可直接讀取 spec、ticket 或兩者。
2. `to-tickets` 可作為 ticket 的輸入準備工具。
3. 以 `.tdd/tasks/TASK-XXX.md` 作為單一 task 的完整 workflow contract。
4. 以簡化的 `state.yaml` 保存可恢復狀態、test case status 與兩個 retry 計數器；詳細結果寫入 final report。
5. 由 orchestrator 嚴格控制 task 順序。
6. `test-writer` 與 `implementor` 必須分離。
7. 實作前強制驗證測試為 Red。
8. 每個 test case 有兩階段獨立上限：test writer 重寫 5 次，implementor 實作 5 次；implementation 第 5 次失敗時 rollback 並 block 同 task 的剩餘 cases，independent tasks 仍可繼續。
9. Final report 保留人工 review。
10. Agent 不自動做高風險的猜測或重構；git checkpoint 僅由 orchestrator 依既定 scope 執行。
