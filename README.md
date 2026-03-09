# EssentialSkill — TDD iOS Architecture Runbook

A reusable, agent-executable TDD script for building iOS apps following the pattern:
**Remote API → Local Cache → Decoupled UI**

---

## What this is

This repo contains a Claude Code **skill** (`/tdd`) that guides an AI agent — or a human — through building a production-quality iOS app using strict Test-Driven Development and Clean Architecture. The runbook is intentionally generic: replace `[Feature]` with your domain noun (Feed, Orders, Profile, etc.) and it applies to any feature.

---

## How to use the `/tdd` skill

### On this project (already configured)

```
/tdd
```

The skill reads your git log, finds the last committed test, and tells you exactly which step comes next. No code is written until you confirm.

To jump to a specific phase:

```
/tdd phase-2
/tdd phase 5
```

---

### On a new project

**Option A — Personal (available in all your projects)**

```bash
cp -r .claude/skills/tdd ~/.claude/skills/tdd
```

`/tdd` is now available in every project you open with Claude Code.

**Option B — Per project (committed to the repo)**

```bash
mkdir -p /path/to/new-project/.claude/skills
cp -r .claude/skills/tdd /path/to/new-project/.claude/skills/tdd
```

Then add a `CLAUDE.md` to the new project pointing to the TDD rules, or copy this project's `CLAUDE.md` and adapt it.

---

## The flow

```
/tdd
 │
 ├── Session Start
 │     Read CLAUDE.md → git log → locate last committed test → confirm next step
 │
 ├── PHASE 0 — Plan
 │     BDD use cases · module map · dependency diagram · protocol contracts
 │     Exit: system described in modules + contracts, no framework names
 │
 ├── PHASE 1 — Project Foundation
 │     iOS App target · macOS Framework target (domain, no UIKit) · macOS Test target
 │     Random order · coverage enabled · .gitignore · initial commit
 │     Exit: builds clean, domain tests run on Mac (no simulator)
 │
 ├── PHASE 2 — Remote API Layer
 │     2.1  [Feature]Loader protocol + DomainModel
 │     2.2  Remote[Feature]Loader — 9 tests, one commit each (TDD)
 │     2.3  URLSessionHTTPClient — URLProtocol stubs
 │     2.4  End-to-end API test target
 │     2.5  CI — GitHub Actions + Thread Sanitizer
 │     Exit: all mapping + error paths covered, CI green
 │
 ├── PHASE 3 — Local Cache Layer  ← skip if offline not required
 │     3.1  [Feature]Store protocol
 │     3.2  Local[Feature]Loader — Save (9 tests)
 │     3.3  Local[Feature]Loader — Load (8 tests)
 │     3.4  ValidateCache use case
 │     3.5  [Feature]CachePolicy — pure, static, side-effect-free
 │     3.6  CodableFeedStore
 │     3.7  CoreData[Feature]Store + shared StoreSpecs
 │     3.8  Cache integration test target
 │     Exit: invalidation rules provably enforced, shared spec passes both stores
 │
 ├── PHASE 4 — UI Prototype (Sandbox)
 │     Throw-away target · hardcoded data · shimmer · refresh control · app icon
 │     Exit: UI iterable with stubbed data, no real loaders involved
 │
 ├── PHASE 5 — Decoupled Production UI
 │     Path A (UIKit): 14 ViewController tests via FakeUIRefreshControl + DSL helpers
 │     Path B (SwiftUI): 9–12 @Observable ViewModel tests in domain framework
 │     Exit: UI swappable by composition only, no infrastructure imports in UI layer
 │
 └── PHASE 6 — Composition Root
       Wire CoreDataStore + LocalLoader + RemoteLoader + View in one place
       Decorator + Composite patterns for cache-with-fallback
       Exit: app runs end-to-end, no UI/domain type imports infrastructure
```

---

## TDD cycle (non-negotiable)

```
Write one failing test  →  confirm it fails (Red)
                        ↓
Write minimum code to pass  →  confirm it passes (Green)
                        ↓
Refactor  →  tests still green
                        ↓
Commit
                        ↓
Write the next failing test
```

Every passing test gets its own commit before the next test is written.

---

## Core architecture principles

| Principle | Rule |
|---|---|
| Domain Isolation | Domain models never import `Codable`, `CoreData`, or `Realm` |
| Dependency Inversion | Depend on protocols, not concrete types. Constructor injection only |
| TDD | Test behavior through public interfaces. Spies capture — never preset return values |
| Decoupled UI | UI never knows where data comes from. Wire in Composition Root only |
| No Global State | Hide singletons behind protocols and inject them |
| Concurrency | Thread-safety is a first-class requirement. TSan enabled on CI |

---

## Stack decisions baked into the runbook

| Decision | Rule |
|---|---|
| Testing framework | Swift Testing (`import Testing`) — requires Xcode 16+ |
| Domain framework platform | macOS only — domain tests run natively, no simulator |
| Concurrency style | `async/await` for iOS 15+, completion handlers for iOS 14 and below |
| ViewModel mechanism | `@Observable` for iOS 17+, `ObservableObject`+`@Published` for iOS 16 and below |
| Image caching | UI-layer concern (`NSCache`) — not in domain framework |
| Search debounce | `Task` cancellation in ViewModel — no third-party library |

---

## Files in this repo

| File | Purpose |
|---|---|
| `CLAUDE.md` | Hard constraints Claude Code enforces on every session |
| `tdd-project-script.md` | Source of truth for the runbook (human-readable) |
| `.claude/skills/tdd/SKILL.md` | Skill entry point — frontmatter + session protocol |
| `.claude/skills/tdd/runbook.md` | Full phase-by-phase execution guide (used by the skill) |
