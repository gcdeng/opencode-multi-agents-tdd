---
name: tdd-run
description: Executes `.tdd/tasks/` as a sequential Red-Green TDD workflow with separate test-writer and implementor agents, five attempts per phase per test case, persistent state, and a Markdown final report. Use when the user wants to run the planned TDD work.
---

# TDD Run

You are running a sequential TDD workflow. Run a strict Red-Green loop: one behavior-focused test at a time against a confirmed public seam, a meaningful Red before implementing, the smallest change to turn it green, then the next slice. Do not refactor inside the loop.

The skill accepts an optional recovery mode:

- `continue`: resume from `.tdd/state.yaml`; completed, failed, and blocked items remain skipped according to their state.
- `rerun-failed-only`: retry failed test cases and test cases blocked by an implementation failure. Reset those cases to `PENDING`, clear their failure reason, and reset both attempt counters before processing. Do not reset passed cases.
- `reset-all`: start the workflow from the task plan again, retaining task files but recreating runtime state.

If the previous workflow has failed or blocked items and no mode was supplied, ask the user to choose a recovery mode before changing state. Do not silently reset state.

## Startup

1. Confirm `.tdd/tasks/` exists.
2. Load `.tdd/state.yaml`; create it from `references/state-template.yaml` if absent.
3. Read task front matter to validate the task contract, order, and dependencies. Read only the current task file in full during a task.
4. Detect the repository's test, focused-test, typecheck, and build commands; record the exact commands in the final report.
5. Detect git: if the working directory is not a git worktree, record `skipped: not a git repository` in the report and disable task checkpoints. Otherwise inspect the repository's contribution guidance and recent history once to learn its commit-message convention; record the detection in the report.
6. Run the baseline test. If it fails, mark the workflow `BLOCKED`, write the final report, and stop.

For `rerun-failed-only`, apply the reset before selecting tasks:

1. Reset `FAILED` test cases to `PENDING`, clear `failure_reason`, and set `writer_attempts` and `implementor_attempts` to `0`.
2. Reset `BLOCKED` test cases in the same task when they were blocked by an implementation failure in that task; do not reset dependency-blocked tasks until their dependency becomes `PASSED`.
3. Recompute task statuses from their test-case statuses and dependencies. Preserve all `PASSED` cases and their existing production changes.
4. Persist the reset state before running any agent or command, then continue with the normal Red-Green workflow.

## Safety

- Do not access the network unless the user explicitly authorizes it for the current task.
- Do not read secrets, credentials, `.env` files, or equivalent secret stores.
- Do not run destructive commands, including `git reset --hard`, broad deletion, or force operations.
- Do not push, create pull requests, or change remote repository state.
- Before changing files, verify that the change is within the active task scope.
- If the scope of a rollback cannot be determined safely, do not guess: mark the test case and task `BLOCKED`, record the reason, and stop the affected workflow path.
- Keep agent results structured: changed files, test/seam identity, outcome, and remaining concerns.

## Role

Only you execute test, typecheck, build, and git commands, and write `.tdd/state.yaml` and `.tdd/final-report.md`; you do not write production code or tests. When delegating, remind `tdd-test-writer` and `tdd-implementor` not to run tests or write to `.tdd/`.

## Execution

Process tasks in dependency order; never run tasks in parallel. Within a task, process test cases in file order, one at a time.

### Phase 1: Test Writing (Red Confirmation)

1. Set the task and test case status to `WRITING_TEST`; initialize `writer_attempts: 0`.
2. Delegate the active test case to `tdd-test-writer`.
3. Run the focused test.
   - Passes: verify it is a real behavior assertion, not tautological. If valid, mark it `PASSED` with result `already_green`; do not manufacture a Red failure.
   - Fails for an invalid reason (syntax, import, environment, does not verify acceptance criteria, unrelated): treat as authoring failure.
4. On authoring failure: increment `writer_attempts`. If `< 5`, retry with a new `tdd-test-writer` invocation (same test identity, replace the invalid version). If `== 5`, mark the test case `FAILED` with reason `TEST_AUTHORING_FAILED` and move on.
5. On a meaningful production-behavior Red: set `TEST_RED`, initialize `implementor_attempts: 0`, and proceed to Phase 2.

### Phase 2: Implementation

6. Record the production files in scope.
7. Set the test case status to `IMPLEMENTING` and persist it. Invoke `tdd-implementor`; increment `implementor_attempts` (first invocation = 1).
8. Run the focused test.
   - Passes: mark the test case `PASSED` and select the next test case.
   - Fails and `implementor_attempts < 5`: retry with a new `tdd-implementor` invocation.
    - Fails at attempt 5: restore only the production changes made for this test case. If that scope cannot be determined safely, mark the test case and task `BLOCKED` and record the rollback safety failure. Otherwise mark it `FAILED` with reason `IMPLEMENTATION_FAILED`, and mark the remaining test cases in the task `BLOCKED`.

### Task Completion

After all runnable test cases in a task are processed, run task regression, typecheck, and build. Mark the task `PASSED` only when every test case passed and all checks pass; otherwise `FAILED`. Block tasks that depend on a failed task, but continue independent tasks.

When a task is `PASSED`, create a git checkpoint before starting the next task (skip if checkpoints were disabled in Startup):

1. Derive the files changed by this task from the before-task and after-task repository state. Stage only the task's production and test files; never `git add -A`, unrelated pre-existing changes, or `.tdd/`.
2. Commit using the repository's convention learned in Startup. Record the short hash in the final report.
3. If staging or committing fails, keep the task `PASSED`, record the error, and continue. Do not retry by staging broader files or bypassing hooks.

## State Rules

- Persist every state transition immediately.
- Store only the current task/test case, statuses, attempt counts, and failure reasons. The attempt limit is always 5.
- Record command results, failure summaries, changed files, and rollback details in the final report. Derive changed files from the actual diff, not agent output.
- Never alter test assertions to make a failing test pass.

## Completion

After all runnable tasks are processed, run full regression once. Write `.tdd/final-report.md` from `references/final-report-template.md`, replacing all placeholders and listing every failed or blocked test case under Failures. Set the workflow status `COMPLETED` when every runnable task and final check passed, otherwise `FAILED`. Present the report to the user.

Report field sources (task files follow `tdd-plan`'s `references/task-template.md`):

- Task heading `<TASK-ID>: <title>`: use the task file's front-matter `id` and `title`; if `title` is absent, fall back to the first line of the Goal section.
- `Public Seam`: from the task file's Public Seam section.
- Table rows: each test case's `id` from the task file, with `status`, attempt counts, and `max_attempts: 5` from the run.
