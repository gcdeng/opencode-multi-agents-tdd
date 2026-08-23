# TDD Run Report

## Summary

- Run ID: `<run-id>`
- Input: `<spec-or-ticket-reference>`
- Status: `<COMPLETED|BLOCKED|FAILED>` (BLOCKED only when baseline or rollback safety failed)
- Started At: `<timestamp>`
- Finished At: `<timestamp>`

## Overall Result

- Tasks: `<number>`
- Passed Tasks: `<number>`
- Failed Tasks: `<number>`
- Blocked Tasks: `<number>`
- Test Cases: `<number>`
- Passed Test Cases: `<number>`
- Failed Test Cases: `<number>`
- Blocked Test Cases: `<number>`

## Task Results

### `<TASK-ID>: <title>`

- Status: `<PASSED|FAILED|BLOCKED>`
- Public Seam: `<public-seam>`
- Test Cases: `<passed>/<total>` passed
- Commit: `<short-hash|skipped: not a git repository|error: <reason>>`

| Test Case | Status | Test Writer Attempts | Implementor Attempts | Max Attempts (each) |
|---|---:|---:|---:|---:|
| `<TC-ID>` | `<PASSED|FAILED|BLOCKED>` | `<number>` | `<number>` | `5` |

Repeat the task section for every task.

## Files Changed

- `<path>`

## Test Results

| Check | Status | Command |
|---|---|---|
| Baseline Test | `<PASSED|FAILED>` | `<command>` |
| Focused Tests | `<PASSED|FAILED>` | `<command>` |
| Task Regression | `<PASSED|FAILED|NOT RUN>` | `<command>` |
| Full Regression | `<PASSED|FAILED|NOT RUN>` | `<command>` |
| Typecheck | `<PASSED|FAILED|NOT RUN>` | `<command>` |
| Build | `<PASSED|FAILED|NOT RUN>` | `<command>` |

## Failures

### `<TC-ID>`

- Status: `<FAILED|BLOCKED>`
- Failure Reason: `<TEST_AUTHORING_FAILED|IMPLEMENTATION_FAILED>`
- Test Writer Attempts Used: `<number>`
- Implementor Attempts Used: `<number>`
- Max Attempts (each): `5`
- Last Command: `<command>`
- Last Failure: `<failure-summary>`
- Next Action: Resolve the failure, then run `/tdd-run rerun-failed-only` or ask the orchestrator to retry failed cases

For an implementation failure, also record whether the production changes were rolled back. For a valid test that passed before implementation, record result `already_green`.

Use `None.` when there are no failures.

## Review Required

- Review production code changes.
- Review tests at the confirmed public seams.
- Confirm no implementation-coupled tests were added.
- Confirm known limitations and required follow-up work.

## Known Limitations

- `<known limitation or None>`
