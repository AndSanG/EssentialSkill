# TDD iOS Project Build Script
## A Reusable Guide for: Remote API → Local Cache → Decoupled UI

This script is intentionally generic so it can be applied to any feature domain (Feed, Orders, Profile, etc.).
Replace `[Feature]` with your domain noun (e.g., Feed, Product, Post).

---

## Session Start

> Use this section at the beginning of every new session — including mid-phase handoffs.

1. Read `CLAUDE.md` — hard constraints for this project.
2. Run `git log --oneline` — identify the last committed test.
3. Find that test's step in this script and confirm the current phase.
4. State which step comes next. Do not write any code until confirmed.

---

## Introduction for Agents

This document is a phase-by-phase execution guide. Read it fully before taking any action. Work through one phase at a time and do not advance until the exit criteria for the current phase is met.

**Why TDD matters specifically for AI agents:** A failing test is a hard contract you cannot hallucinate past. Without a failing test first, an AI agent will generate code that looks correct but satisfies no verifiable requirement. The test is not bureaucracy — it is the only mechanism that forces the agent to prove its output is right. Every time you skip the Red phase, you remove that proof.

### Core Principles to Enforce Throughout
1. **Domain Isolation** — Domain models must never depend on API (`Codable`) or database (`CoreData`/`Realm`) details. Keep infrastructure DTOs inside infrastructure modules.
2. **Dependency Inversion** — High-level modules depend on abstractions (protocols), not concrete types. Prefer constructor injection.
3. **TDD** — Test behavior through public interfaces, not private internals. Use Spies to capture interactions; never preset return values.
4. **Decoupled UI** — The UI must not know where data comes from. Wire dependencies exclusively in the Composition Root.
5. **No Global State** — Hide third-party singletons (e.g., `URLSession.shared`) behind protocols and inject them.
6. **Concurrency** — Treat thread-safety as a first-class requirement. Design for it; prove it in tests.

---

## Project Requirements

Before starting Phase 0, gather the project requirements. Look for a requirements document in the project (e.g., `requirements.md`, `PRD.md`, `brief.md`, or similar). If none exists, ask the user for the following before proceeding:

- **Project name** — the app's name, used for target and type naming
- **Platform** — iOS, macOS, or both
- **API base URL** — the remote endpoint to load data from
- **Auth** — None, API Key, OAuth, Bearer Token
- **Core use cases** — the minimum user stories (e.g., "User sees a feed", "User can pull to refresh")
- **Primary entities** — the domain models (e.g., `FeedItem`, `UserProfile`)
- **Offline behavior** — read-through cache, full offline, or none
- **Cache invalidation policy** — e.g., 7-day max age, manual refresh clears cache
- **UI surfaces** — screens to build (e.g., Feed List, Item Detail)
- **Non-functional constraints** — e.g., thread-safety, accessibility, image loading performance

### Stack Decisions

These must also be resolved before Phase 0, as they cascade into architecture and test strategy across every phase.

> Note: Phases 0–3 (domain, networking, cache) are fully architecture and UI-framework agnostic. The decisions below only affect Phase 4 onward.

**Minimum iOS deployment target / Swift version**
The deployment target determines which app lifecycle and language features are available. Ask the user or read it from the project:
- iOS 12 or below → AppDelegate only, no SceneDelegate
- iOS 13 → SceneDelegate available, multi-window on iPad
- iOS 14+ → SwiftUI App lifecycle available (`@main`, `WindowGroup`)
- iOS 18 (current default) → SwiftUI lifecycle is the standard; AppDelegate/SceneDelegate are legacy

**UI Framework — UIKit or SwiftUI**
This is the most consequential decision. It affects Phase 4 (prototype), Phase 5 (production UI), and Phase 6 (composition root entry point).

| | UIKit | SwiftUI |
|---|---|---|
| Views | `UIViewController` + `UIView` subclasses | `struct` conforming to `View` |
| Lists | `UITableViewController` with DataSource/Delegate, manual cell reuse | `List` with `ForEach`, automatic reuse |
| State updates | Explicit (`tableView.reloadData()`, `label.text = ...`) | Automatic via `@State`, `@ObservedObject` |
| Navigation | `UINavigationController`, push/pop imperatively | `NavigationStack`, declarative |
| Entry point | `AppDelegate` / `SceneDelegate` → `UIWindow` + `rootViewController` | `@main App` → `WindowGroup` |
| Composition Root | `SceneDelegate.scene(_:willConnectTo:)` | `App.body` or `@UIApplicationDelegateAdaptor` |
| Test strategy (Phase 5) | Drive `UIViewController` with `XCTest` + DSL helpers, `FakeUIRefreshControl` → **Phase 5.2 Path A** | Drive `View` logic through observable state; UI tests via `ViewInspector` or snapshot tests → **Phase 5.2 Path B** |

**Minimum iOS Deployment Target — Decision Required**

Ask the user for the minimum deployment target before choosing SwiftUI or UIKit. The answer determines which APIs are available and which path to follow in Phase 5.

| Deployment target | SwiftUI verdict | Key constraint | ViewModel mechanism |
|---|---|---|---|
| **iOS 17+** | ✅ Full SwiftUI | `@Observable`, `NavigationStack`, `ContentUnavailableView` all available | `@Observable` (`import Observation`) in domain framework |
| **iOS 16** | ✅ SwiftUI with one swap | `@Observable` unavailable; `NavigationStack` available | `ObservableObject` + `@Published` (`import Combine`) in domain framework |
| **iOS 15** | ⚠️ SwiftUI with gaps | No `NavigationStack`; use deprecated `NavigationView`; `.searchable` limited | `ObservableObject` + `@Published`; avoid `NavigationStack` |
| **iOS 14** | ⚠️ SwiftUI limited | App lifecycle available; significant view API gaps | `ObservableObject` + `@Published`; expect manual workarounds |
| **iOS 13** | ❌ Prefer UIKit | SwiftUI immature; major gaps in List, navigation, state | UIKit → Phase 5.2 Path A |
| **iOS 12 or below** | ❌ UIKit only | SwiftUI not available | UIKit → Phase 5.2 Path A |

Record the chosen deployment target here before proceeding:

> **Chosen minimum deployment target:** ___________

> If swapping `@Observable` for `ObservableObject`, ViewModels stay in the domain framework via `import Combine`. The rest of the architecture — protocols, Composition Root, TDD phases — is unchanged.

**UI Architecture Pattern**
The domain model (Phases 0–3) is shared across all patterns — this decision only shapes how the UI layer is structured in Phase 5. Ask the user which pattern to use. If there is no preference, default to **MVVM**.

| Pattern | Description | Phase 5 shape |
|---|---|---|
| **MVC** | View controller owns model + view logic. Simple, but grows large. | `[Feature]ViewController` handles loading, rendering, and user events directly |
| **MVVM** *(default)* | ViewModel holds state and business logic; View binds to it. Testable without UIKit/SwiftUI. | `[Feature]ViewModel` (observable) + lightweight View/ViewController; ViewModel tested in isolation |
| **MVP** | Presenter drives a passive View via a protocol. Common in UIKit TDD. | `[Feature]Presenter` + `[Feature]ViewProtocol`; view is a dumb protocol implementation |
| **VIPER** | Module split into View, Interactor, Presenter, Entity, Router. High separation, high boilerplate. | Full module per feature; use only if the team is already familiar with it |

The Composition Root (Phase 6) wires the chosen pattern together — the ViewModel, Presenter, or Interactor is created and injected there, not inside the view.

> **SwiftUI + MVVM ViewModel placement:** When using SwiftUI with MVVM and the deployment target is **iOS 17+**, ViewModels should use `@Observable` (from `import Observation`, not `import SwiftUI`). This keeps them platform-agnostic and allows the entire test suite to run on macOS without a simulator. Place ViewModels inside the domain framework — they have no UI dependencies and belong there. For iOS 16 or below, use `ObservableObject` + `@Published` instead — see the deployment target table above.

Do not begin Phase 0 until all of the above are known.

---

## PHASE 0 — Plan

**Goal:** Define the app's use cases and architecture without committing to any framework.

Use cases are defined in BDD style, domain models are identified and separated from DTOs, module boundaries and dependency directions are decided, and protocol contracts are established for the API client, loaders, store, and presentation layer. A dependency diagram is produced before any implementation begins.

### Steps
1. Define the minimum set of use cases in BDD style (Given / When / Then).
2. Identify domain models vs. API/data-transfer models (DTOs).
3. Decide module boundaries and dependency directions.
4. Define protocols for the API client, feature loaders, cache store, and presentation adapters.
5. Create a dependency diagram for the first feature slice.

**Artifacts:** Use case list, module map + dependency diagram, public contracts per module.

**Exit Criteria:** The system can be fully described in terms of modules and contracts without naming any specific framework (no URLSession, CoreData, SwiftUI, etc.).

---

## PHASE 1 — Project Foundation

**Goal:** An empty project with the right structure before writing a single line of logic.

An Xcode iOS app is created alongside a **macOS framework** target for domain logic. The domain framework targets macOS so all domain and networking tests run natively on the Mac — no simulator required, no boot time, no device selection. A unit test target is added against the macOS framework. Random test order and code coverage are enabled.

### Steps
1. Create an Xcode iOS **App** target: `[AppName]`.
2. Add a **macOS Framework** target: `[AppName]` (same name, different platform). Set deployment target to match the macOS version in use. Mark it as the domain + infrastructure module — no UIKit, no SwiftUI imports allowed here.
   > **Decision:** The domain framework targets **macOS only** (not cross-platform iOS+macOS). This is intentional: macOS-only means the test suite runs natively on the Mac without launching a simulator, giving the fastest possible CI feedback loop. The iOS app target links this same framework at build time — no duplication.
3. Add a **macOS Unit Testing Bundle** target: `[AppName]Tests`, with **Target to be Tested** set to the macOS framework. Tests run on Mac, no simulator needed.
4. Set test execution to **random order** and enable **code coverage**.
5. Add `.gitignore`, commit: `"Initial commit"`.

> **Why macOS for the domain framework?**
> The domain and networking layers have no platform-specific dependencies. Targeting macOS lets the test suite run in seconds on the local machine without launching a simulator. The iOS app target links the same framework at build time, so there is no duplication.

**Artifacts:** Xcode project with iOS app target + macOS framework target + macOS test target, CI-ready scheme.

**Exit Criteria:** Project builds cleanly. Domain test suite runs on Mac (no simulator) with zero tests, random order, and coverage enabled.

---

## PHASE 2 — Remote Data / API Layer

**Goal:** A tested, protocol-driven networking layer that delivers domain models. No URLSession coupling in tests.

A protocol-driven networking layer is built from scratch using TDD. An HTTPClient protocol is injected into the RemoteLoader so tests never touch URLSession directly. All JSON mapping lives in a dedicated mapper type, keeping the loader focused on business rules.

### 2.1 Define the Use Case Contract

A [Feature]Loader protocol and a [DomainModel] struct are defined in the [AppName] framework. The protocol exposes a single load(completion:) method. No implementation yet — just the types.

- Create the `DomainModel` value type (struct, all properties, no framework imports).
- Create `[Feature]Loader` protocol using the concurrency style that matches the chosen deployment target:
  - **iOS 15+** → `async throws -> [DomainModel]` (Swift 5.5 / Xcode 13; back-deploys to iOS 15 natively, iOS 13/14 via embedded Swift runtime with limitations)
  - **iOS 14 or below** → `load(completion:)` returning `Result<[DomainModel], Error>`
  - Prefer `async/await` whenever the deployment target allows it. Completion handlers are only used when the floor is iOS 14 or lower.
- **Commit:** `"Define [Feature]Loader protocol and [DomainModel] value type"`

### 2.2 Drive the RemoteLoader with Tests

Remote[Feature]Loader is implemented using strict TDD with an injected HTTPClient protocol. A LoaderSpy captures requests and JSON mapping is kept in a separate [Feature]ItemsMapper type. Memory leak detection is added using the pattern that matches the chosen testing framework:
- **Swift Testing** → `weak` + `defer` inline in each test body, or `deinit` on the `@Suite` struct (see Cross-Cutting Concerns — Memory Leak Detection)
- **XCTest** → `addTeardownBlock { [weak sut] in XCTAssertNil(sut) }` helper on `XCTestCase`

Tests are written one at a time — each must fail first, then pass.

Write tests in this order (one failing test → make it pass → next):
1. `init_doesNotRequestData` — loader does not call the client on init.
2. `load_requestsDataFromURL` — calling `load()` triggers one request to the correct URL.
3. `loadTwice_requestsDataFromURLTwice` — can call `load()` more than once.
4. `load_deliversConnectivityErrorOnClientError` — client error → `.connectivity` error.
5. `load_deliversInvalidDataErrorOnNon200Response` — non-200 → `.invalidData` error.
6. `load_deliversInvalidDataErrorOn200ResponseWithInvalidJSON` — 200 + bad JSON → `.invalidData`.
7. `load_deliversNoItemsOn200ResponseWithEmptyJSONList` — 200 + `{"items":[]}` → empty array.
8. `load_deliversItemsOn200ResponseWithJSONItems` — 200 + valid JSON → domain models.
9. `load_doesNotDeliverResultAfterSUTHasBeenDeallocated` — memory/lifetime safety.

**Key decisions at this stage:**
- Inject `HTTPClient` as a protocol (not URLSession directly).
- Use a `Spy` (capture values), not a `Stub` (preset return values).
- Map JSON → `Remote[Feature]Item` (private DTO) → `DomainModel`. Never expose JSON types.
- Extract a `[Feature]ItemsMapper` static type for all mapping logic.

**Commits (one per passing test):**
```
"[Feature]Loader does not request data upon creation"
"Load requests data from URL"
"Deliver connectivity error on client error"
"Deliver invalidData error on non-200 HTTP response"
"Deliver invalidData error on 200 with invalid JSON"
"Deliver empty items on 200 with empty JSON list"
"Deliver items on 200 with valid JSON items"
"Do not deliver result after instance is deallocated"
```

### 2.3 Implement URLSessionHTTPClient

URLSessionHTTPClient is implemented conforming to HTTPClient. URLProtocol stubs are used for tests — URLSession is never subclassed. Tests cover the URL, HTTP method, error delivery, and successful data delivery.

- Drive with URLProtocol-based stubs (not URLSession subclassing).
- Tests: requests correct URL/method, delivers error on failure, delivers data+response on success.
- **Commit:** `"Make URLSessionHTTPClient conform to HTTPClient protocol"`

### 2.4 End-to-End API Tests
- Add a separate `[AppName]APIEndToEndTests` target.
- Hit the real staging/prod API (or a controlled fixture server).
- Validate at least: empty list → 0 items; known fixture → expected models.
- Use an `ephemeral` URLSession configuration to avoid caching.
- **Commit:** `"Add end-to-end API tests in a separate target"`

### 2.5 CI Setup
- Add GitHub Actions workflow (`.github/workflows/ci.yml`).
- Enable **Thread Sanitizer** on the CI scheme.
- Run all test targets.
- Add CI badge to README.
- **Commit:** `"Add CI scheme + GitHub Actions config"`

**Artifacts:** `HTTPClient` protocol, `Remote[Feature]Loader`, `[Feature]ItemsMapper`, `URLSessionHTTPClient`, end-to-end test target, CI workflow.

**Exit Criteria:** Remote use case can load domain models deterministically from a captured HTTP response. All mapping and error paths are covered by tests. CI is green.

---

## PHASE 3 — Local Cache Layer

> **Decision Gate:** Phase 3 is only required if offline support is in scope for this deliverable. If the requirements do not call for offline reading, cached data, or cache invalidation, skip Phase 3 entirely and proceed to Phase 4. Document the decision in the README under "What I Would Improve or Add Next".

**Goal:** A tested, protocol-driven persistence layer. The use case (LocalLoader) never knows about CoreData or Codable.

A [Feature]Store protocol abstracts all persistence operations. The LocalLoader implements save, load, and validate use cases against that protocol, with a pure cache policy type enforcing expiry rules. Concrete store implementations (Codable, CoreData) are validated against a shared spec.

### 3.1 Define the FeedStore / Cache Contract

A [Feature]Store protocol is defined with retrieve, insert, and deleteCached operations using Swift.Result for all completion handlers. No implementation yet.

- Create `[Feature]Store` protocol with `retrieve`, `insert`, `deleteCached[Feature]` methods.
- Use `Result` typealiases for all completion types.
- **Commit:** `"Extract [Feature]Store protocol from test spy"`

### 3.2 Drive Local[Feature]Loader — Save Use Case
Tests in order:
1. `init_doesNotMessageStoreUponCreation`
2. `save_requestsCacheDeletion`
3. `save_doesNotRequestInsertionOnDeletionError`
4. `save_requestsNewCacheInsertionOnSuccessfulDeletion`
5. `save_requestsCacheInsertionWithTimestampOnSuccessfulDeletion`
6. `save_failsOnDeletionError`
7. `save_failsOnInsertionError`
8. `save_succeedsOnSuccessfulCacheInsertion`
9. `save_doesNotDeliverResultAfterSUTDeallocated`

**Key decisions:**
- Save = Delete old → Insert new (with a timestamp).
- Store receives `Local[Feature]Item` DTOs, not domain models.
- Add `Local[Feature]Item` as a separate data-transfer type.

**Commits:** One per test.

### 3.3 Drive Local[Feature]Loader — Load Use Case

The LocalFeedLoader load use case is driven with TDD. The loader reads from a [Feature]Store and applies a max-age policy (e.g., 7 days). Deletion is not performed on load — that is handled by the separate validate use case. Domain models are delivered on success.

Tests in order:
1. `load_doesNotMessageStoreUponCreation`
2. `load_requestsCacheRetrieval`
3. `load_failsOnRetrievalError`
4. `load_deliversNoItemsOnEmptyCache`
5. `load_deliversCachedItemsOnLessThanMaxAgeOldCache`
6. `load_deliversNoItemsOnMaxAgeOldCache`
7. `load_deliversNoItemsOnMoreThanMaxAgeOldCache`
8. `load_doesNotDeliverResultAfterSUTDeallocated`

### 3.4 Drive Local[Feature]Loader — ValidateCache Use Case
- Separate from load: validate deletes stale/corrupt caches without delivering results.
- Tests mirror the load cache-age policy but assert on deletion side-effects.
- **Commit:** `"Extract Validate[Feature]Cache use case"`

### 3.5 Extract Cache Policy

Cache expiry logic is extracted into a static [Feature]CachePolicy type. It takes a timestamp and the current date and returns a Bool. It has no state and no side effects, keeping all expiry rules in one verifiable place.

- Create `[Feature]CachePolicy` as a pure, static, side-effect-free type.
- All cache expiry logic lives here. No dates in the loader.
- **Commit:** `"Extract cache validation policy into [Feature]CachePolicy"`

### 3.6 Implement CodableFeedStore
- Store/retrieve using `Codable` + `FileManager`.
- Drive with the full `[Feature]StoreSpecs` protocol (shared spec pattern).
- Use a test-specific store URL (not production path) in setUp/tearDown.
- Dispatch operations on a serial background queue.
- **Commit:** `"Make CodableFeedStore conform to [Feature]Store"`

### 3.7 Implement CoreData[Feature]Store

CoreData[Feature]Store is implemented conforming to [Feature]Store. Shared store specs are extracted into a [Feature]StoreSpecs protocol so both Codable and CoreData implementations are tested against the same contract.

- Adopt the same `[Feature]StoreSpecs` — pass all the same tests.
- Use `/dev/null` store URL in tests to keep them in-memory and fast.
- Use a private background `NSManagedObjectContext`.
- Prove side-effects run serially.
- **Commit:** `"Make CoreDataFeedStore conform to [Feature]Store"`
- **Commit:** `"Delete CodableFeedStore in favor of CoreDataFeedStore"` (if only one is needed)

### 3.8 Cache Integration Tests
- Add `[AppName]CacheIntegrationTests` target.
- Test `Local[Feature]Loader` + `CoreData[Feature]Store` together end-to-end.
- Use a real temporary file URL, clean up in setUp/tearDown.
- Add to CI scheme.

**Artifacts:** `[Feature]Store` protocol, `CoreData[Feature]Store`, `[Feature]CachePolicy`, `Local[Feature]Loader` (save/load/validate), cache integration test target.

**Exit Criteria:** Cached data loads correctly and invalidation rules are provably enforced by tests. Both Codable and CoreData stores pass the shared `[Feature]StoreSpecs` contract.

---

## PHASE 4 — UI Prototyping (Sandbox)

**Goal:** A throw-away visual sandbox to nail the design before writing any testable UI code.

A [AppName]Prototype app target is created with a UITableViewController and a [Feature]Cell driven by hardcoded data. A shimmer animation simulates image loading. The goal is fast visual iteration — no tests, no real loaders.

### 4.1 Create Prototype Target
- New app target: `[AppName]Prototype`.
- Storyboard with a table view and a custom cell layout.
- Hardcoded data array — no networking, no cache.

### 4.2 Iterate Visually
- Add shimmering/skeleton animation to cells.
- Add `UIRefreshControl`.
- Add fade-in animation for image loading simulation.
- Add app icon.
- **Commit:** `"Add prototype with [Feature] cell layout and shimmer animation"`

**Artifacts:** Prototype app target with hardcoded UI, shimmer animation, refresh control.

**Exit Criteria:** UI can be previewed and iterated with hardcoded/stubbed data with no real loaders involved.

---

## PHASE 5 — Decoupled Production UI

**Goal:** A testable, framework-independent UI that never knows where data comes from.

Before starting this phase, confirm the two decisions made in Stack Decisions:
- **UI Framework:** UIKit → follow **Path A**. SwiftUI → follow **Path B**.
- **Deployment target:** determines which APIs and ViewModel mechanism are available (see deployment target table in Stack Decisions).

### 5.1 Add iOS Framework Target *(optional)*

> **SwiftUI note:** For SwiftUI apps, this intermediate iOS framework target is often unnecessary. Views can live directly in the app target if no cross-platform reuse is planned. Skip this step if using SwiftUI and the app target is the only UI consumer. Proceed to 5.2.

- Create `[AppName]iOS` framework target (iOS only — SwiftUI/UIKit allowed here).
- The domain framework (macOS) remains UI-free; this iOS framework depends on it via protocols only.
- Update CI schemes: one macOS scheme for domain + infra tests (no simulator), one iOS scheme for UI tests (simulator required).
- **Commit:** `"Add [AppName]iOS framework target"`

### 5.2 Drive [Feature]ViewController with Tests

**Choose the path that matches your UI framework:**

---

#### Path A — UIKit (`UITableViewController`)

Tests in order:
1. `loadActions_requestFeedFromLoader` — load triggered on `viewIsAppearing` and pull-to-refresh.
2. `loadingIndicator_isVisibleWhileLoadingFeed` — refresh control visible during load.
3. `loadingIndicator_isHiddenAfterLoadingCompletes` — hidden on both success and error.
4. `load_rendersSuccessfullyLoadedFeed` — renders correct number of cells after success.
5. `load_doesNotAlterCurrentRenderingStateOnLoadError` — error doesn't clear existing feed.
6. `feedImageView_loadsImageURLWhenVisible` — image load starts on cell display.
7. `feedImageView_cancelsImageLoadWhenNotVisibleAnymore` — cancel on cell reuse.
8. `feedImageViewLoadingIndicator_isVisibleWhileLoadingImage` — shimmer while loading.
9. `feedImageView_rendersImageLoadedFromURL` — image rendered on success.
10. `feedImageView_showsRetryOnImageLoadError` — retry button on error.
11. `feedImageView_showsRetryOnInvalidImageData` — retry on corrupt data.
12. `feedImageView_retriesImageLoadOnRetryAction` — retry triggers a new load.
13. `feedImageView_preloadsImageURLWhenNearVisible` — prefetch on `willDisplay`.
14. `feedImageView_cancelsImageURLPreloadingWhenNotNearVisibleAnymore` — cancel on `didEndDisplaying`.

**Key decisions (UIKit):**
- `[Feature]ViewController` conforms to `UITableViewDataSource` / `UITableViewDelegate`.
- It depends on `[Feature]Loader` and `FeedImageDataLoader` protocols (not concrete types).
- Use `FakeUIRefreshControl` in tests (records `isRefreshing` state without UIKit behavior).
- Decouple tests from specific UI controls with DSL helper methods.
- Extract `FeedImageDataLoaderTask` for cancellation (Interface Segregation).

---

#### Path B — SwiftUI (`View` + `@Observable` ViewModel)

The ViewModel is already fully tested in the domain framework (see Phase 2 + 5.2 Path B).
SwiftUI views are thin wrappers over observable state — the ViewModel tests *are* the UI behavior tests.

| UIKit concern | SwiftUI equivalent |
|---|---|
| `UIViewController` + `FakeUIRefreshControl` | `@Observable` ViewModel with `isLoading: Bool` |
| `tableView.reloadData()` | Automatic on `@Observable` property change |
| Pull-to-refresh test | Test `load()` is called; `.refreshable` modifier is declarative, no fake needed |
| Cell visibility / prefetch | `onAppear` / `task` modifiers; test ViewModel methods directly |
| Shimmer while image loads | `CachedAsyncImage` / `AsyncImage` phase handling in view |
| Retry button | Bound to ViewModel `load()` — tested at ViewModel level |

**Tests to drive the ViewModel (one failing test → make it pass):**

1. `init_doesNotLoadCharacters` — no side-effects on creation.
2. `load_requestsCharactersFromLoader` — `load()` triggers exactly one loader call.
3. `load_setsIsLoadingDuringRequest` — `isLoading` is `true` while in-flight.
4. `load_clearsIsLoadingOnSuccess` — `isLoading` is `false` after success.
5. `load_clearsIsLoadingOnFailure` — `isLoading` is `false` after failure.
6. `load_deliversItemsOnSuccess` — `items` reflects the loaded results.
7. `load_setsErrorMessageOnFailure` — `errorMessage` is non-nil on failure.
8. `load_clearsErrorBeforeReloading` — stale error cleared when `load()` is called again.
9. `load_doesNotDeliverResultAfterSUTDeallocated` — memory safety.
10. *(If paginated)* `loadNextPage_appendsItemsOnSuccess` — page 2 appended to page 1 results.
11. *(If paginated)* `loadNextPage_doesNothingWhenOnLastPage` — no call when `hasNextPage` is false.
12. *(If search)* Debounce: cancel previous `Task` before starting a new one on input change. See Cross-Cutting Concerns — Search Debounce.

**Key decisions (SwiftUI):**
- ViewModel uses `@Observable` (`import Observation`, not SwiftUI) — stays in the domain framework.
- Views import `SwiftUI` and the domain framework; they have zero knowledge of loaders or infrastructure.
- Image caching: use a `CachedAsyncImage` wrapper backed by `NSCache` in the UI layer — not in the domain.
- No `FeedImageDataLoader` protocol needed unless image loading has business rules (retry, cancel, prefetch).

**Image caching note:**
> Image data loading is a UI-layer concern. Memory-level caching (`NSCache<NSURL, UIImage>`) lives in the UI target, not the domain framework. If image loading has business rules (cancellation, retry, prefetch on `willDisplay`), extract a separate `[Feature]ImageDataLoader` protocol and inject it the same way as the main loader.

### 5.3 Move to Production
- Move `[Feature]ViewController` (UIKit) or verify `[Feature]View` (SwiftUI) lives in the correct target with no infrastructure imports.
- Verify the UI framework / app target has no import of any infrastructure module.
- **Commit:** `"Move [Feature]View to production target"`

**Artifacts:** `[AppName]iOS` framework, `[Feature]ViewController`, `[Feature]ImageDataLoader` protocol, UI-level test suite.

**Exit Criteria:** UI can be previewed/prototyped with stubbed loaders and swapped to real implementations by composition only. No UI type imports any infrastructure module.

---

## PHASE 6 — Composition Root (Main App Target)

**Goal:** Wire everything together in one place. Nothing else knows the full graph.

In the app target's composition root, CoreData[Feature]Store, Local[Feature]Loader, Remote[Feature]Loader, and [Feature]ViewController are wired together using the Decorator and Composite patterns. No UI or domain type imports any infrastructure type.

### 6.1 Create the Composition

> **Cache dependency note:** The Composite and Decorator patterns below only apply if Phase 3 (local cache) was implemented. If Phase 3 was skipped, inject the remote loader directly — the composition simplifies to a single loader per use case.

**With Phase 3 (cache + remote fallback):**

In the main app `AppDelegate` or `SceneDelegate`:
```swift
let store = CoreData[Feature]Store(storeURL: ...)
let localLoader = Local[Feature]Loader(store: store, currentDate: Date.init)
let remoteLoader = Remote[Feature]Loader(url: apiURL, client: URLSessionHTTPClient())

// Compose: try cache first, fall back to remote, then save to cache
let feedViewController = [Feature]ViewComposer.composedWith(
    feedLoader: FeedLoaderWithFallbackComposite(
        primary: localLoader,
        fallback: RemoteFeedLoaderCacheDecorator(decoratee: remoteLoader, cache: localLoader)
    ),
    imageLoader: ...
)
```

**Without Phase 3 (remote only):**

```swift
let client = URLSessionHTTPClient(session: URLSession(configuration: .default))
let loader = Remote[Feature]Loader(baseURL: apiURL, client: client)
let viewModel = [Feature]ViewModel(loader: loader)
```

- The View / ViewController has no idea it's talking to CoreData or the network.
- **Commit:** `"Compose [Feature]View with all collaborators in the Composition Root"`

**Artifacts:** Composition Root in app target, `FeedLoaderWithFallbackComposite`, `RemoteFeedLoaderCacheDecorator`, wired app/scene delegate.

**Exit Criteria:** The app runs end-to-end with no UI or domain type importing infrastructure types. Full feature slice is exercisable from the main target.

---

## README — Progressive Documentation

The README is a living document. **Never write it all at once.** Add each section at the exact moment the implementation it describes exists. No section is written beforehand, and nothing is left as a catch-up task at the end.

| Section | Write it when |
|---|---|
| **How to run the project** | Phase 1 — the moment the project builds and tests run |
| **Architecture and reasoning** | Phase 2 — once the first real module boundary is enforced (HTTPClient + RemoteLoader) |
| **How Dependency Injection works** | Phase 2 — when the first protocol is injected via constructor |
| **What was tested and why** | Updated incrementally — one bullet per phase as each test suite is completed |
| **Observability and security choices** | Phase 2 (networking decisions) and Phase 5 (UI layer choices); append as each concern is addressed |
| **What you would improve or add next** | Updated at the end of each phase with concrete next steps; reflects the actual current state of the project |

**Rule:** If a README section refers to something that doesn't exist in the codebase yet, it must not be written yet.

**Test count rule:** Keep test counts in README in sync with the actual test suite at all times. If a test is added or removed, update the count in the same commit. Stale counts erode trust in the README.

---

## Artifact Naming

Never name artifacts after phases (e.g., `Phase0Plan.md`, `PHASE1_SETUP.md`). Use names that describe their content:

| Instead of | Use |
|---|---|
| `PHASE0_PLAN.md` | `BDDRequirements.md` |
| `PHASE0_ARCHITECTURE.md` | `ModuleMap.md` or `DependencyDiagram.md` |
| `PHASE1_SETUP.md` | Part of `README.md` |
| `PHASE2_NETWORKING.md` | Inline in source as protocol + type comments |

---

## CROSS-CUTTING CONCERNS (Apply Throughout)

### Testing Framework — Swift Testing

**Prerequisites:** Swift Testing requires **Xcode 16+** (Swift 6.0). It is not available in Xcode 15.

| Test location | Minimum requirement | Notes |
|---|---|---|
| macOS-hosted (domain framework) | Xcode 16 | Swift runtime bundled with toolchain — no iOS deployment target concern |
| iOS simulator / device | Xcode 16 + iOS 16+ deployment target | Swift Testing runtime must be present on the simulated OS |

> If the project's minimum deployment target is iOS 15 or lower, use XCTest for any tests that run on the simulator or device. macOS-hosted domain tests can still use Swift Testing regardless of the app's deployment target.

Use **Swift Testing** (`import Testing`) for all domain, networking, and ViewModel tests. Do not use XCTest for these layers.

| XCTest | Swift Testing |
|---|---|
| `class FooTests: XCTestCase` | `@Suite struct FooTests` |
| `func testSomething()` | `@Test func something()` |
| `XCTAssertEqual(a, b)` | `#expect(a == b)` |
| `XCTAssertNil(x)` | `#expect(x == nil)` |
| `XCTAssertNotNil(x)` | `#require(x != nil)` |
| `XCTUnwrap(x)` | `try #require(x)` |
| `addTeardownBlock { }` | `defer { }` inside the test body, or `deinit` of the `@Suite` struct |

### Memory Leak Detection
Swift Testing has no `addTeardownBlock`. Use `weak` capture + `defer` inside each test, or track via `deinit` of an `@Suite` struct:

```swift
// Inline in test body
@Test func load_doesNotLeakLoader() {
    var sut: RemoteCharacterLoader? = RemoteCharacterLoader(...)
    weak var weakSUT = sut
    sut = nil
    #expect(weakSUT == nil, "Expected instance to be deallocated — potential memory leak")
}

// Or: shared helper for use in makeSUT factory
func assertDeallocated(_ instance: AnyObject?, _ message: String = "Potential memory leak") {
    #expect(instance == nil, Comment(rawValue: message))
}
```

Call the factory helper at the end of every test that creates a `sut`.

### Test Naming Convention
Swift Testing does **not** require a `test` prefix. The `@Test` macro handles discovery. Use a descriptive function name that reads as a sentence fragment; the display name string is optional but useful for parameterised tests.

```swift
// Preferred: no "test" prefix, reads naturally
@Test func init_doesNotRequestData() { }
@Test func load_requestsDataFromURL() { }
@Test func load_deliversConnectivityErrorOnClientError() { }
@Test func load_deliversNoItemsOn200WithEmptyJSONList() { }

// With display name (good for parameterised or complex cases)
@Test("delivers .invalidData on non-200 response", arguments: [199, 201, 300, 400, 500])
func load_deliversInvalidDataErrorOnNon200Response(statusCode: Int) { }
```

**Pattern:** `[subject]_[expectedOutcome]` or `[subject]_[condition]_[expectedOutcome]`

### Spy Pattern (not Stub)
Always capture interactions, never preset return values. Complete captured closures in tests to control timing.

### One Assert Per Test
Each test method verifies exactly one behavior. If you need two assertions, write two tests.

### Access Control
Order: `public` → `internal` (implicit) → `private`. Minimize public surface. Protocols are public; implementations are internal.

### Result Types
Use `Swift.Result` throughout. No custom enums for success/failure. Use typealiases in protocols for readability.

### No Global State
Avoid global mutable state. Hide third-party singletons (e.g., `URLSession.shared`) behind protocols and inject them. Never reference global state in production types.

### Concurrency
Treat thread-safety as a first-class requirement. Enable Thread Sanitizer (TSan) on CI. Dispatch store operations on a serial background queue. Prove side-effects run serially in tests.

**CoreData / legacy stores:** dispatch operations on a private serial `NSManagedObjectContext` or a `DispatchQueue`. This is the pattern Phase 3 follows.

**New code (Swift 5.9+):** prefer structured concurrency. Use `async/await` for loader methods and `actor` isolation for shared mutable state instead of manual queue management. Do not mix `DispatchQueue`-based and `async/await`-based code in the same type — pick one model per boundary and be consistent.

> **`@Observable` + `Task`:** When a ViewModel stores a `Task` reference for cancellation, mark it `@ObservationIgnored` to prevent the observation system from tracking it as state. See Search Debounce below.

### Search Debounce
If the feature includes user-driven search or filtering, debounce input in the ViewModel using `Task` cancellation — no third-party library required:

```swift
@ObservationIgnored private var searchTask: Task<Void, Never>?

func onSearchTextChanged(_ text: String) {
    searchTask?.cancel()
    searchTask = Task {
        try? await Task.sleep(for: .milliseconds(300))
        guard !Task.isCancelled else { return }
        load()
    }
}
```

- Cancel the previous `Task` before creating a new one on every input change.
- Test debounce by verifying that rapid successive calls result in only one loader invocation.
- Debounce lives in the ViewModel, not the View.

### Mandatory Commit Discipline

> This is a hard rule. Context resets between sessions. Without commits, work is invisible and unverifiable.

**Every passing test must be committed before writing the next test.** The red → green → refactor → commit cycle is non-negotiable:

1. Write one failing test. Run it — confirm it fails.
2. Write the **minimum** production code to make it pass. Run it — confirm it passes.
3. **Refactor** — clean up the implementation without changing behavior. Run tests again — confirm they still pass. This is where duplication is removed, names are clarified, and structure is improved. Do not skip this step; passing code is not finished code.
4. **Commit.** Use the commit messages prescribed in each phase.
5. Only then write the next failing test.

**Enforcement checklist before moving to the next phase:**
- [ ] Every test listed in the phase has its own commit.
- [ ] No test exists in the codebase without a corresponding passing implementation in the same or earlier commit.
- [ ] The git log reads as a narrative of behavior, not a dump of files.

If you are resuming work in a new session:
1. Read `CLAUDE.md` — it defines the hard constraints for this project.
2. Run `git log --oneline` — compare the log against the phase checklist in this file.
3. Identify the last committed test and locate the matching step in this script.
4. Do not write new tests until you have confirmed all previous ones are committed.

---

## PROMPT TEMPLATES

### Generic Project Start Prompt
```
I want to build [AppName] following strict TDD and Clean Architecture.
The app [description of what it does].
Start with Phase 0: define the use cases in BDD style and produce a module map with dependency diagram. No code yet.
```

### Phase Transition Prompts
```
Phase 0 Exit Criteria met. Let's start Phase 1: Project Foundation.
```
```
Phase 1 Exit Criteria met. Let's start Phase 2: Remote Data / API Layer.
```
```
Phase 2 Exit Criteria met. Let's start Phase 3: Local Cache Layer.
```
```
Phase 3 Exit Criteria met. Let's start Phase 4: UI Prototyping.
```
```
Phase 4 Exit Criteria met. Let's start Phase 5: Decoupled Production UI.
```
```
Phase 5 Exit Criteria met. Let's start Phase 6: Composition Root.
```

### Refactor Prompt (Safe Refactor)
```
All tests pass. Refactor [type/method] to [goal] without changing behavior. Tests must remain green.
```