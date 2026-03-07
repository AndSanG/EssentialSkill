# TDD iOS Project Build Script
## A Reusable Guide for: Remote API → Local Cache → Decoupled UI

This script is derived from the EssentialFeed git history. It is intentionally generic
so it can be applied to any feature domain (Feed, Orders, Profile, etc.).
Replace `[Feature]` with your domain noun (e.g., Feed, Product, Post).

---

## PHASE 0 — Project Foundation

**Goal:** An empty project with the right structure before writing a single line of logic.

### Steps
1. Create an Xcode project with a `[AppName]` framework target (no UI, platform-agnostic).
2. Create a `[AppName]Tests` unit test target.
3. Set test execution to **random order** and enable **code coverage**.
4. Add `.gitignore`, commit: `"Initial commit"`.

**Prompt:**
> "Create an Xcode iOS project named [AppName] with a separate framework target for the domain logic. No UI yet. Add a unit test target. Enable random test order and code coverage. Commit as 'Initial commit'."

---

## PHASE 1 — Remote Data / API Layer

**Goal:** A tested, protocol-driven networking layer that delivers domain models. No URLSession coupling in tests.

### 1.1 Define the Use Case Contract
- Create `[Feature]Loader` protocol with a `load(completion:)` method returning `Result<[DomainModel], Error>`.
- Create the `DomainModel` value type (struct, all properties, no framework imports).
- **Commit:** `"Define [Feature]Loader protocol and [DomainModel] value type"`

**Prompt:**
> "In TDD, define a [Feature]Loader protocol and a [DomainModel] struct in the [AppName] framework. The protocol has a single load(completion:) method. No implementation yet—just the types."

### 1.2 Drive the RemoteLoader with Tests
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

**Prompt:**
> "Using strict TDD, implement Remote[Feature]Loader with an injected HTTPClient protocol. Drive development with the failing tests listed above, one at a time. Use a LoaderSpy that captures requests. Keep JSON mapping in a separate [Feature]ItemsMapper type. Add memory leak detection in XCTestCase."

### 1.3 Implement URLSessionHTTPClient
- Drive with URLProtocol-based stubs (not URLSession subclassing).
- Tests: requests correct URL/method, delivers error on failure, delivers data+response on success.
- **Commit:** `"Make URLSessionHTTPClient conform to HTTPClient protocol"`

**Prompt:**
> "Implement URLSessionHTTPClient conforming to HTTPClient. Use URLProtocol stubs for tests—never subclass URLSession. Drive with TDD: test the URL, HTTP method, error delivery, and successful data delivery."

### 1.4 End-to-End API Tests
- Add a separate `[AppName]APIEndToEndTests` target.
- Hit the real staging/prod API (or a controlled fixture server).
- Validate at least: empty list → 0 items; known fixture → expected models.
- Use an `ephemeral` URLSession configuration to avoid caching.
- **Commit:** `"Add end-to-end API tests in a separate target"`

### 1.5 CI Setup
- Add GitHub Actions workflow (`.github/workflows/ci.yml`).
- Enable **Thread Sanitizer** on the CI scheme.
- Run all test targets.
- Add CI badge to README.
- **Commit:** `"Add CI scheme + GitHub Actions config"`

---

## PHASE 2 — Local Cache Layer

**Goal:** A tested, protocol-driven persistence layer. The use case (LocalLoader) never knows about CoreData or Codable.

### 2.1 Define the FeedStore / Cache Contract
- Create `[Feature]Store` protocol with `retrieve`, `insert`, `deleteCached[Feature]` methods.
- Use `Result` typealiases for all completion types.
- **Commit:** `"Extract [Feature]Store protocol from test spy"`

**Prompt:**
> "Define a [Feature]Store protocol with retrieve, insert, and deleteCached operations. Use Swift.Result for all completion handlers. No implementation yet."

### 2.2 Drive Local[Feature]Loader — Save Use Case
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

### 2.3 Drive Local[Feature]Loader — Load Use Case
Tests in order:
1. `test_load_doesNotMessageStoreUponCreation`
2. `test_load_requestsCacheRetrieval`
3. `test_load_failsOnRetrievalError`
4. `test_load_deliversNoItemsOnEmptyCache`
5. `test_load_deliversCachedItemsOnLessThanMaxAgeOldCache`
6. `test_load_deliversNoItemsOnMaxAgeOldCache`
7. `test_load_deliversNoItemsOnMoreThanMaxAgeOldCache`
8. `test_load_doesNotDeliverResultAfterSUTDeallocated`

**Prompt:**
> "Drive LocalFeedLoader load use case with TDD. The loader reads from a [Feature]Store and applies a max-age policy (e.g., 7 days). Do not delete on load—that is a separate 'validate' use case. Deliver domain models on success."

### 2.4 Drive Local[Feature]Loader — ValidateCache Use Case
- Separate from load: validate deletes stale/corrupt caches without delivering results.
- Tests mirror the load cache-age policy but assert on deletion side-effects.
- **Commit:** `"Extract Validate[Feature]Cache use case"`

### 2.5 Extract Cache Policy
- Create `[Feature]CachePolicy` as a pure, static, side-effect-free type.
- All cache expiry logic lives here. No dates in the loader.
- **Commit:** `"Extract cache validation policy into [Feature]CachePolicy"`

**Prompt:**
> "Extract the cache expiry logic into a static [Feature]CachePolicy type. It takes a timestamp and the current date and returns a Bool. It has no state and no side effects."

### 2.6 Implement CodableFeedStore
- Store/retrieve using `Codable` + `FileManager`.
- Drive with the full `[Feature]StoreSpecs` protocol (shared spec pattern).
- Use a test-specific store URL (not production path) in setUp/tearDown.
- Dispatch operations on a serial background queue.
- **Commit:** `"Make CodableFeedStore conform to [Feature]Store"`

### 2.7 Implement CoreData[Feature]Store
- Adopt the same `[Feature]StoreSpecs` — pass all the same tests.
- Use `/dev/null` store URL in tests to keep them in-memory and fast.
- Use a private background `NSManagedObjectContext`.
- Prove side-effects run serially.
- **Commit:** `"Make CoreDataFeedStore conform to [Feature]Store"`
- **Commit:** `"Delete CodableFeedStore in favor of CoreDataFeedStore"` (if only one is needed)

**Prompt:**
> "Implement CoreData[Feature]Store conforming to [Feature]Store. Extract shared store specs into a [Feature]StoreSpecs protocol so both Codable and CoreData implementations can be tested against the same contract."

### 2.8 Cache Integration Tests
- Add `[AppName]CacheIntegrationTests` target.
- Test `Local[Feature]Loader` + `CoreData[Feature]Store` together end-to-end.
- Use a real temporary file URL, clean up in setUp/tearDown.
- Add to CI scheme.

---

## PHASE 3 — UI Prototyping (Sandbox)

**Goal:** A throw-away visual sandbox to nail the design before writing any testable UI code.

### 3.1 Create Prototype Target
- New app target: `[AppName]Prototype`.
- Storyboard with a table view and a custom cell layout.
- Hardcoded data array — no networking, no cache.

### 3.2 Iterate Visually
- Add shimmering/skeleton animation to cells.
- Add `UIRefreshControl`.
- Add fade-in animation for image loading simulation.
- Add app icon.
- **Commit:** `"Add prototype with [Feature] cell layout and shimmer animation"`

**Prompt:**
> "Create a [AppName]Prototype target with a UITableViewController and a [Feature]Cell. Use hardcoded data. Add a shimmer animation to cells to simulate image loading. No tests needed here."

---

## PHASE 4 — Decoupled Production UI

**Goal:** A testable, framework-independent UI that never knows where data comes from.

### 4.1 Add iOS Framework Target
- Create `[AppName]iOS` framework target.
- Platform-agnostic targets (domain + infra) support macOS too.
- Update CI schemes: separate macOS and iOS CI schemes.
- **Commit:** `"Add [AppName]iOS framework target"`

### 4.2 Drive [Feature]ViewController with Tests
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

**Prompt:**
> "Drive [Feature]ViewController with TDD. It depends on [Feature]Loader and [Feature]ImageDataLoader protocols. Use a LoaderSpy, a FakeUIRefreshControl, and DSL helpers to decouple tests from UIKit internals. Follow the iOS17+ viewIsAppearing lifecycle."

### 4.3 Move to Production
- Move `[Feature]ViewController` from test target to `[AppName]iOS` framework.
- Verify the UI framework has no import of any infrastructure module.
- **Commit:** `"Move [Feature]ViewController to production [AppName]iOS target"`

---

## PHASE 5 — Composition Root (Main App Target)

**Goal:** Wire everything together in one place. Nothing else knows the full graph.

### 5.1 Create the Composition
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

**Prompt:**
> "In the app target's composition root, wire together CoreData[Feature]Store, Local[Feature]Loader, Remote[Feature]Loader, and [Feature]ViewController using the Decorator and Composite patterns. No UI or domain type should import infrastructure types."

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

---

## PROMPT TEMPLATES

### Generic Project Start Prompt
> "I want to build [AppName] following strict TDD and Clean Architecture.
> The app [description of what it does].
> Start with Phase 1: define the [Feature]Loader protocol and [DomainModel] struct in a platform-agnostic framework target. No UI, no networking yet. Just the types and a failing test."

### Phase Transition Prompt
> "Phase [N] is complete. All tests pass. Now move to Phase [N+1]: [goal].
> Start with the first failing test: [test name]."

### Refactor Prompt (Safe Refactor)
> "All tests pass. Refactor [type/method] to [goal] without changing behavior. Tests must remain green."
