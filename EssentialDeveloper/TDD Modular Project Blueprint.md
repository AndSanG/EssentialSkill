TDD Modular Project Blueprint: From API to UI
This blueprint outlines the steps to build a robust, modular, and test-driven application. It follows the architectural patterns seen in the EssentialFeed project, emphasizing decoupling, dependency inversion, and cross-platform readiness.

Phase 1: Project Setup & Domain Definition
Goal: Define what the app does without committing to any infrastructure (Networking, DB, UI).

Initialize Project: Create a new Workspace and a main Framework target (e.g., AppNameCore).
Define Domain Models: Create simple struct types representing your business data.
Example: FeedImage, NewsItem, Product.
Define Feature Gateways (Protocols): Create protocols that define the operations needed for the feature.
Example: FeedLoader, ImageDataLoader.
Phase 2: Remote API Integration (Networking)
Goal: Connect to the outside world using TDD.

Infrastructure Abstraction: Create an HTTPClient protocol to wrap networking libraries (URLSession, Alamofire).
Implement Remote Loader: Create a concrete implementation of your domain protocol (e.g., RemoteFeedLoader).
Methodology: Use TDD with a HTTPClientSpy.
Steps:
Test "does not request data on init".
Test "requests data from URL on load".
Test "delivers connectivity error on client error".
Test "delivers invalid data error on non-200 responses".
Decouple API Models: Create private Decodable models in the API layer and map them to Domain models to prevent API changes from breaking the app.
Create API End-to-End Tests: Verify the integration against a real backend/staging environment.
Phase 3: Local Caching & Persistence
Goal: Enable offline capabilities and optimize performance.

Define Store Protocol: Create a protocol for storage operations (Save, Retrieve, Delete).
Example: FeedStore.
Implement Local Loader: Create a concrete implementation (e.g., LocalFeedLoader) that uses the Store protocol.
Use Cases:
Save: Handle save logic and cache policies (e.g., expiration).
Load: Retrieve data from the store and validate if it's still fresh.
Infrastructure Implementation: Create concrete Store implementations (e.g., CoreDataFeedStore, RealmFeedStore).
Cache Integration Tests: Use real disk operations to ensure data is correctly persisted.
Phase 4: UI Prototyping (Rapid Feedback)
Goal: Discover UI requirements without the complexity of the full architecture.

Create Prototype Target: Build a standalone App target.
Hardcode Data: Use static data to verify layout, animations (shimmering), and user interactions.
Iterate Fast: Focus on "How it looks" and "How it feels" before "How it's connected".
Phase 5: UI Foundation & Composition
Goal: Build a reusable, platform-independent UI layer.

Modularize UI: Create a separate UI Framework (e.g., AppNameiOS).
Implement Generic Controllers: Create ViewControllers that don't know about specific loaders (Dependency Injection).
Use Composition Roots: Create a Composer (or Factory) in the main App module to connect UI components with Loader implementations (API or Cache).
Patterns: Use Adapters to convert Loader outputs into ViewModel/UI updates.
UI Testing: Use Snapshot tests and Unit tests for ViewControllers by spying on their collaborators.
How to use this as a Prompt
When starting a new project, you can prompt the AI with:

"Acting as an architect following the EssentialFeed paradigm, let's start [Phase X]. Our core requirement is [Feature Description]. Please guide me through the TDD steps, starting with the protocol definition and the first failing test."