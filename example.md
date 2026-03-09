# Example Session

## Prompt

1. First read the PDF with requirements example is Discogs
2. Then run `/tdd`

## Output

No git history and no `CLAUDE.md`. This is a fresh start.

Per the runbook's Session Start and Project Requirements sections, I need to gather all requirements before Phase 0. Most are clear from the PDF, but a few things need confirmation:

### Ready to confirm before Phase 0

**App name** — What should it be called? (e.g. DiscogsSearch, Vinylogy, Discogs)

**Offline / cache support?** — The challenge doesn't mention it. Per the runbook, Phase 3 (local cache) can be skipped if not required. I'd recommend skipping it and noting it in "What I'd add next." Agree?

**Discogs API token** — Do you have one? (needed for the `Authorization` header — get it at discogs.com/settings/developers)

**Static analyzer** — SwiftLint via SPM for the bonus requirement. Confirmed?

### Already resolved by the challenge spec

| Decision | Value |
|---|---|
| Platform | iOS 17+, SwiftUI, SPM |
| Architecture | MVVM + Clean Architecture |
| ViewModel | `@Observable` (iOS 17+) |
| Testing | Swift Testing (Xcode 16+) |
| Auth | `Authorization` header with Discogs token |
| Pagination | 30 items/page |