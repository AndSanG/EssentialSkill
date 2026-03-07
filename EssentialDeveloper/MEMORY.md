# EssentialFeed Project Memory

## Project Overview
This is a memory from an iOS app following strict TDD + Clean Architecture + Modular design.
Language: Swift | Framework: UIKit | Persistence: CoreData | Network: URLSession. 
The specific stack is not important, the focus is on the architecture and the TDD process.

## Key Architectural Patterns
- Dependency injection via protocols (HTTPClient, FeedLoader, FeedStore, FeedImageDataLoader)
- Composition Root pattern: ViewControllers are generic, composed in app layer
- Separate modules: EssentialFeed (domain+infra), EssentialFeediOS (UI), Prototype (visual sandbox)
- Interface Segregation: e.g., FeedImageDataLoaderTask split from FeedImageDataLoader

## Important Files
- Domain: EssentialFeed/Feed Feature/FeedLoader.swift, FeedImage.swift
- Remote: EssentialFeed/Feed API/RemoteFeedLoader.swift, FeedItemsMapper.swift, HTTPClient.swift
- Cache: EssentialFeed/Feed Cache/LocalFeedLoader.swift, FeedStore.swift, FeedCachePolicy.swift
- Infrastructure: EssentialFeed/Feed Cache/Infrastructure/CoreData/CoreDataFeedStore.swift
- UI: EssentialFeediOS/Feed UI/Controllers/FeedViewController.swift

## Test Structure
- EssentialFeedTests: unit tests (Feed API, Feed Cache)
- EssentialFeedAPIEndToEndTests: real network calls
- EssentialFeedCacheIntegrationTests: LocalFeedLoader + CoreDataFeedStore
- EssentialFeediOSTests: UI tests

## CI
- GitHub Actions with separate macOS and iOS schemes
- Thread sanitizer enabled
- Random test order enabled

## Detailed Guides
- [TDD Project Script](tdd-project-script.md): Step-by-step guide for building a project from scratch using this architecture
