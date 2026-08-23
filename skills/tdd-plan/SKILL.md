---
name: tdd-plan
description: Creates executable TDD task files from a spec or user-provided ticket input. Use when the user asks to plan work, turn requirements into tasks and test cases, define public seams, or prepare a TDD workflow.
---

# TDD Plan

Create the TDD plan. The output is task files for `tdd-run`; production code is outside this skill's scope.

## Input

Accept `/tdd-plan <input>`. The input may contain a spec, tickets, or both.

If both are present, use ticket content as task scope and spec content as context. Record conflicts in the task file.

If the input cannot be read or does not contain enough information to define a
task, stop and ask the user for a valid spec or ticket. Do not invent scope.

## Process

1. Read the complete input.
2. Explore the repository. Read `CONTEXT.md` and relevant ADRs when present.
3. Identify the project's test framework and test commands.
4. If the input is a ticket, preserve its vertical-slice scope and `Blocked by` dependencies. Re-split it only when it is clearly too large to fit one fresh implementation context, and record the reason.
5. If the input is a spec, split it into small, independently verifiable vertical tasks.
6. For every task, define the public seam and observable behavior. Do not write tests against private internals.
7. Define candidate test cases, ordered from the smallest tracer bullet to broader behavior. Each test case must be executable as one Red-to-Green slice.
8. Set `max_attempts: 5` for every test case.
9. Write one file per task under `.tdd/tasks/TASK-XXX.md`.
10. Validate unique IDs, dependencies, ordering, public seams, acceptance criteria, at least one test case per task, and test-case coverage.
11. Stop after generating and validating the plan. Do not start `tdd-run` automatically.

## Task File Contract

Write every task file from `references/task-template.md`, keeping its structure and field names. Every task file must contain YAML front matter with `id`, `title`, `order`, and `dependencies`, then:

- Goal and user-visible behavior
- Acceptance criteria
- Public seam
- Observable behavior
- Ordered test cases
- Completion definition

Only unit tests are allowed. Do not write integration or e2e test cases. Each test case must contain `id`, `status: PENDING`, `max_attempts: 5`, `given`, `when`, `then`, and the expected test location.

## Final Report Alignment

`tdd-run` mirrors task-file fields into `.tdd/final-report.md`. Keep names and values identical so the mapping is direct:

- `id` + `title` -> report task heading `<TASK-ID>: <title>`
- `Public Seam` section -> `Public Seam` field in Task Results
- test case `id` / `status` / `max_attempts` -> row in the report's task table
- Status vocabulary matches `state.yaml`: `PENDING` / `PASSED` / `FAILED` / `BLOCKED`
