# TDD iOS Project Build Script
## A Reusable Guide for: Remote API → Local Cache → Decoupled UI

This script is intentionally generic so it can be applied to any feature domain (Feed, Orders, Profile, etc.).
Replace `[Feature]` with your domain noun (e.g., Feed, Product, Post).

---

## Introduction for Agents

This document is a phase-by-phase execution guide. Read it fully before taking any action. Work through one phase at a time and do not advance until the exit criteria for the current phase is met.

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
| Test strategy (Phase 5) | Drive `UIViewController` with `XCTest` + DSL helpers, `FakeUIRefreshControl` | Drive `View` logic through observable state; UI tests via `ViewInspector` or snapshot tests |

**UI Architecture Pattern**
The domain model (Phases 0–3) is shared across all patterns — this decision only shapes how the UI layer is structured in Phase 5. Ask the user which pattern to use. If there is no preference, default to **MVVM**.

| Pattern | Description | Phase 5 shape |
|---|---|---|
| **MVC** | View controller owns model + view logic. Simple, but grows large. | `[Feature]ViewController` handles loading, rendering, and user events directly |
| **MVVM** *(default)* | ViewModel holds state and business logic; View binds to it. Testable without UIKit/SwiftUI. | `[Feature]ViewModel` (observable) + lightweight View/ViewController; ViewModel tested in isolation |
| **MVP** | Presenter drives a passive View via a protocol. Common in UIKit TDD. | `[Feature]Presenter` + `[Feature]ViewProtocol`; view is a dumb protocol implementation |
| **VIPER** | Module split into View, Interactor, Presenter, Entity, Router. High separation, high boilerplate. | Full module per feature; use only if the team is already familiar with it |

The Composition Root (Phase 6) wires the chosen pattern together — the ViewModel, Presenter, or Interactor is created and injected there, not inside the view.

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

An Xcode iOS project named [AppName] is created with a separate framework target for domain logic, a unit test target, random test order enabled, and code coverage turned on.

### Steps
1. Create an Xcode project with a `[AppName]` framework target (no UI, platform-agnostic).
2. Create a `[AppName]Tests` unit test target.
3. Set test execution to **random order** and enable **code coverage**.
4. Add `.gitignore`, commit: `"Initial commit"`.

**Artifacts:** Xcode project with separate framework + test targets, CI-ready test scheme.

**Exit Criteria:** Project builds cleanly. Test suite runs (with zero tests) in random order with coverage enabled.

---

## PHASE 2 — Remote Data / API Layer

**Goal:** A tested, protocol-driven networking layer that delivers domain models. No URLSession coupling in tests.

A protocol-driven networking layer is built from scratch using TDD. An HTTPClient protocol is injected into the RemoteLoader so tests never touch URLSession directly. All JSON mapping lives in a dedicated mapper type, keeping the loader focused on business rules.

### 2.1 Define the Use Case Contract

A [Feature]Loader protocol and a [DomainModel] struct are defined in the [AppName] framework. The protocol exposes a single load(completion:) method. No implementation yet — just the types.

- Create `[Feature]Loader` protocol with a `load(completion:)` method returning `Result<[DomainModel], Error>`.
- Create the `DomainModel` value type (struct, all properties, no framework imports).
- **Commit:** `"Define [Feature]Loader protocol and [DomainModel] value type"`

### 2.2 Drive the RemoteLoader with Tests

Remote[Feature]Loader is implemented using strict TDD with an injected HTTPClient protocol. A LoaderSpy captures requests and JSON mapping is kept in a separate [Feature]ItemsMapper type. Memory leak detection is added to XCTestCase. Tests are written one at a time — each must fail first, then pass.

Write tests in this order (one failing test → make it pass → next):
1. `test_init_doesNotRequestData` — loader does not call the client on init.
2. `test_load_requestsDataFromURL` — calling `load()` triggers one request to the correct URL.
3. `test_loadTwice_requestsDataFromURLTwice` — can call `load()` more than once.
4. `test_load_deliversConnectivityErrorOnClientError` — client error → `.connectivity` error.
5. `test_load_deliversInvalidDataErrorOnNon200Response` — non-200 → `.invalidData` error.
6. `test_load_deliversInvalidDataErrorOn200ResponseWithInvalidJSON` — 200 + bad JSON → `.invalidData`.
7. `test_load_deliversNoItemsOn200ResponseWithEmptyJSONList` — 200 + `{"items":[]}` → empty array.
8. `test_load_deliversItemsOn200ResponseWithJSONItems` — 200 + valid JSON → domain models.
9. `test_load_doesNotDeliverResultAfterSUTHasBeenDeallocated` — memory/lifetime safety.

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

**Goal:** A tested, protocol-driven persistence layer. The use case (LocalLoader) never knows about CoreData or Codable.

A [Feature]Store protocol abstracts all persistence operations. The LocalLoader implements save, load, and validate use cases against that protocol, with a pure cache policy type enforcing expiry rules. Concrete store implementations (Codable, CoreData) are validated against a shared spec.

### 3.1 Define the FeedStore / Cache Contract

A [Feature]Store protocol is defined with retrieve, insert, and deleteCached operations using Swift.Result for all completion handlers. No implementation yet.

- Create `[Feature]Store` protocol with `retrieve`, `insert`, `deleteCached[Feature]` methods.
- Use `Result` typealiases for all completion types.
- **Commit:** `"Extract [Feature]Store protocol from test spy"`

### 3.2 Drive Local[Feature]Loader — Save Use Case
Tests in order:
1. `test_init_doesNotMessageStoreUponCreation`
2. `test_save_requestsCacheDeletion`
3. `test_save_doesNotRequestInsertionOnDeletionError`
4. `test_save_requestsNewCacheInsertionOnSuccessfulDeletion`
5. `test_save_requestsCacheInsertionWithTimestampOnSuccessfulDeletion`
6. `test_save_failsOnDeletionError`
7. `test_save_failsOnInsertionError`
8. `test_save_succeedsOnSuccessfulCacheInsertion`
9. `test_save_doesNotDeliverResultAfterSUTDeallocated`

**Key decisions:**
- Save = Delete old → Insert new (with a timestamp).
- Store receives `Local[Feature]Item` DTOs, not domain models.
- Add `Local[Feature]Item` as a separate data-transfer type.

**Commits:** One per test.

### 3.3 Drive Local[Feature]Loader — Load Use Case

The LocalFeedLoader load use case is driven with TDD. The loader reads from a [Feature]Store and applies a max-age policy (e.g., 7 days). Deletion is not performed on load — that is handled by the separate validate use case. Domain models are delivered on success.

Tests in order:
1. `test_load_doesNotMessageStoreUponCreation`
2. `test_load_requestsCacheRetrieval`
3. `test_load_failsOnRetrievalError`
4. `test_load_deliversNoItemsOnEmptyCache`
5. `test_load_deliversCachedItemsOnLessThanMaxAgeOldCache`
6. `test_load_deliversNoItemsOnMaxAgeOldCache`
7. `test_load_deliversNoItemsOnMoreThanMaxAgeOldCache`
8. `test_load_doesNotDeliverResultAfterSUTDeallocated`

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

[Feature]ViewController is driven with TDD against [Feature]Loader and [Feature]ImageDataLoader protocols. A LoaderSpy, FakeUIRefreshControl, and DSL helpers keep tests decoupled from UIKit internals. The iOS 17+ viewIsAppearing lifecycle is followed throughout.

### 5.1 Add iOS Framework Target
- Create `[AppName]iOS` framework target.
- Platform-agnostic targets (domain + infra) support macOS too.
- Update CI schemes: separate macOS and iOS CI schemes.
- **Commit:** `"Add [AppName]iOS framework target"`

### 5.2 Drive [Feature]ViewController with Tests
Tests in order:
1. `test_loadActions_requestFeedFromLoader` — load triggered on `viewIsAppearing` and pull-to-refresh.
2. `test_loadingIndicator_isVisibleWhileLoadingFeed` — refresh control visible during load.
3. `test_loadingIndicator_isHiddenAfterLoadingCompletes` — hidden on both success and error.
4. `test_load_rendersSuccessfullyLoadedFeed` — renders correct number of cells after success.
5. `test_load_doesNotAlterCurrentRenderingStateOnLoadError` — error doesn't clear existing feed.
6. `test_feedImageView_loadsImageURLWhenVisible` — image load starts on cell display.
7. `test_feedImageView_cancelsImageLoadWhenNotVisibleAnymore` — cancel on cell reuse.
8. `test_feedImageViewLoadingIndicator_isVisibleWhileLoadingImage` — shimmer while loading.
9. `test_feedImageView_rendersImageLoadedFromURL` — image rendered on success.
10. `test_feedImageView_showsRetryOnImageLoadError` — retry button on error.
11. `test_feedImageView_showsRetryOnInvalidImageData` — retry on corrupt data.
12. `test_feedImageView_retriesImageLoadOnRetryAction` — retry triggers a new load.
13. `test_feedImageView_preloadsImageURLWhenNearVisible` — prefetch on `willDisplay`.
14. `test_feedImageView_cancelsImageURLPreloadingWhenNotNearVisibleAnymore` — cancel on `didEndDisplaying`.

**Key decisions:**
- `[Feature]ViewController` conforms to `UITableViewDataSource` / `UITableViewDelegate`.
- It depends on `[Feature]Loader` and `FeedImageDataLoader` protocols (not concrete types).
- Use `FakeUIRefreshControl` in tests (records `isRefreshing` state without UIKit behavior).
- Decouple tests from specific UI controls with DSL helper methods.
- Extract `FeedImageDataLoaderTask` for cancellation (Interface Segregation).

### 5.3 Move to Production
- Move `[Feature]ViewController` from test target to `[AppName]iOS` framework.
- Verify the UI framework has no import of any infrastructure module.
- **Commit:** `"Move [Feature]ViewController to production [AppName]iOS target"`

**Artifacts:** `[AppName]iOS` framework, `[Feature]ViewController`, `[Feature]ImageDataLoader` protocol, UI-level test suite.

**Exit Criteria:** UI can be previewed/prototyped with stubbed loaders and swapped to real implementations by composition only. No UI type imports any infrastructure module.

---

## PHASE 6 — Composition Root (Main App Target)

**Goal:** Wire everything together in one place. Nothing else knows the full graph.

In the app target's composition root, CoreData[Feature]Store, Local[Feature]Loader, Remote[Feature]Loader, and [Feature]ViewController are wired together using the Decorator and Composite patterns. No UI or domain type imports any infrastructure type.

### 6.1 Create the Composition
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
- The ViewController has no idea it's talking to CoreData or the network.
- **Commit:** `"Compose [Feature]ViewController with all collaborators in the Composition Root"`

**Artifacts:** Composition Root in app target, `FeedLoaderWithFallbackComposite`, `RemoteFeedLoaderCacheDecorator`, wired app/scene delegate.

**Exit Criteria:** The app runs end-to-end with no UI or domain type importing infrastructure types. Full feature slice is exercisable from the main target.

---

## CROSS-CUTTING CONCERNS (Apply Throughout)

### Memory Leak Detection
Add to `XCTestCase` base extension:
```swift
func trackForMemoryLeaks(_ instance: AnyObject, file: StaticString = #file, line: UInt = #line) {
    addTeardownBlock { [weak instance] in
        XCTAssertNil(instance, "Instance should have been deallocated. Potential memory leak.", file: file, line: line)
    }
}
```
Use in every factory method.

### Test Naming Convention
`test_[action]_[expectedOutcome]`
Examples:
- `test_load_deliversConnectivityErrorOnClientError`
- `test_save_failsOnDeletionError`
- `test_feedImageView_cancelsImageLoadWhenNotVisibleAnymore`

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
