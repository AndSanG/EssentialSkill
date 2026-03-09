---
name: tdd
description: Run the TDD iOS project build script. Use when starting a new session, resuming a phase, or advancing to the next TDD phase. Guides through Phase 0 (Plan) → Phase 1 (Foundation) → Phase 2 (API) → Phase 3 (Cache) → Phase 4 (Prototype) → Phase 5 (UI) → Phase 6 (Composition Root).
disable-model-invocation: true
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
---

Follow the TDD runbook in [runbook.md](runbook.md) exactly. Read it fully before taking any action.

## On every invocation

1. Run `git log --oneline` to identify the last committed test.
2. Read [runbook.md](runbook.md) — locate that test's step and confirm the current phase.
3. State which step comes next. **Do not write any code until the user confirms.**

## Arguments

`$ARGUMENTS` can be a phase name or number to jump directly to that section (e.g., `phase-2`, `phase 3`).

If no argument is given, start from the Session Start protocol above.

## Hard rules (never violate)

- No production code without a failing test first (Red phase is mandatory).
- No commit without a passing test.
- No advancing to the next phase until every test in the current phase has its own commit.
- Tests must be in a separate test target — not embedded in the production module.
- The Domain framework targets macOS only — no iOS simulator required for domain tests.
- Use Swift Testing unless the deployment target requires XCTest (see runbook prerequisites table).
- Use async/await for all new protocol methods unless the deployment target is iOS 14 or below.
