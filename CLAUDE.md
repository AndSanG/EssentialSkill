# Project Rules

## TDD Process

Run `/tdd` and follow the runbook exactly.

- **Never skip the Red phase.** No production code without a failing test first.
- **Never commit without a passing test.** The red → green → refactor → commit cycle is non-negotiable.
- **Never move to the next phase** until every test in the current phase has its own commit.

## Resume Protocol

When starting a new session or switching phases, follow the **Session Start** section at the top of the runbook.

## Stack Decisions

Decisions already made for this project are recorded in the runbook under each phase and in the Stack Decisions section. Do not re-open closed decisions — follow what is written.

## Hard Constraints

- Tests must be in a **separate test target** — not embedded in the production module.
- The **Domain framework targets macOS only** — no iOS simulator required for domain tests.
- Use **Swift Testing** unless the deployment target requires XCTest (see runbook prerequisites table).
- Use **async/await** for all new protocol methods unless the deployment target is iOS 14 or below.
- Default branch is always **master**.
