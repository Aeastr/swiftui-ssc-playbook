# How it looks in practice

The playbook describes what each layer *is*. This file describes how the code inside each layer is actually organized — how a store file is laid out top to bottom, how services stay flat, how environment wiring works, the patterns that repeat across every file.

---

## File anatomy of a store

Stores follow a consistent internal layout. Here's the skeleton, using `SocialStore` as the reference:

```swift
@MainActor
@Observable
final class SocialStore {
    // MARK: - Dependencies
    private let authService: AuthService
    private let friendsService: FriendsService
    private let profileService: ProfileService

    // MARK: - State
    var loadState: CommunityLoadState = .loading
    var friends: [CommunityProfile] = []
    var incomingRequests: [FriendRequestItem] = []
    // ... more observable state

    private var friendshipsRealtimeTask: Task<Void, Never>?
    private var isRefreshingFriendshipData = false

    // MARK: - Lifecycle
    init(authService: AuthService = .shared, ...) { ... }

    // MARK: - Search
    func filteredFriends(query: String) -> [CommunityProfile] { ... }

    // MARK: - Refresh
    func refreshAll(mode:query:searchScope:) async { ... }
    func runPeopleSearchIfNeeded(query:searchScope:) async { ... }
    private func updateSearchContext(query:searchScope:) { ... }

    // MARK: - Friendship
    func requestFriend(_ userID: UUID) async { ... }
    func respondToFriendRequest(_ userID: UUID, action:) async { ... }
    func unfriend(_ profile: CommunityProfile) async { ... }

    // MARK: - Friendship Realtime
    func startFriendshipsRealtimeListener() { ... }
    func stopFriendshipsRealtimeListener() { ... }
    private func scheduleRealtimeRefresh() async { ... }
    private func refreshFriendshipData() async { ... }

    // MARK: - Profile
    private func fetchCurrentProfile() async -> Profile? { ... }
    private func fetchProfiles(for ids: [UUID]) async -> [UUID: CommunityProfile] { ... }

    // MARK: - Friendship Data
    private func fetchAllFriends() async throws -> [CommunityProfile] { ... }
    private func fetchPendingRequestsWithProfiles() async throws -> (...) { ... }

    // MARK: - Error Helpers
    private func presentFailure(_ error:title:description:source:) { ... }
}

// Supporting types
enum CommunityLoadState { ... }
enum CommunityRefreshMode { ... }
struct CommunityViewError: Error, Identifiable { ... }

// Environment key
private struct SocialStoreKey: EnvironmentKey { ... }
extension EnvironmentValues { ... }
```

The order matters. Dependencies first, then observable state, then init, then public methods grouped by feature, then private helpers, then error handling at the bottom. Supporting types and the environment key come after the class.

### Grouping by feature, not by access level

Functions aren't grouped as "all public, then all private." They're grouped by **what part of the feature they belong to**. The Search section has its public method. The Friendship section has `requestFriend`, `respondToFriendRequest`, and `unfriend` together. The Realtime section has start, stop, and the private debounce/refresh helpers right next to each other.

Private helpers sit directly below the public methods they support. `scheduleRealtimeRefresh` and `refreshFriendshipData` live in the Realtime section, not in some distant "Private Methods" section, because that's where you'll look for them when you're working on realtime logic.

`MissionsStore` follows the same structure — Dependencies, State, Lifecycle, then feature sections like Activity, Cross-Feature Inputs, Refresh, Activity Formatting, Profile, Error Helpers — each containing whatever mix of public and private methods that feature area needs.

### State layout

Dependencies (private, injected services) come first. Then public observable state that views read. Then private backing state like task handles and re-entrancy guards. This separation makes it easy to scan a store and immediately see what it exposes vs. what's internal plumbing.

```swift
// MARK: - Dependencies
private let authService: AuthService
private let friendsService: FriendsService

// MARK: - State
var loadState: CommunityLoadState = .loading      // views read this
var friends: [CommunityProfile] = []               // views read this
var unfriendingProfileIDs: Set<UUID> = []           // views read this

private var friendshipsRealtimeTask: Task<Void, Never>?  // internal plumbing
private var isRefreshingFriendshipData = false            // re-entrancy guard
```

---

## File anatomy of a service

Services are much flatter. They don't have the feature-grouped MARK sections that stores do because they're not orchestrating features — they're just wrapping external calls. A service is usually a list of functions that each do one thing.

```swift
final class FriendsService: Sendable {
    static let shared = FriendsService()

    private let client: SupabaseClient
    private let authService: AuthService

    private init(client: SupabaseClient = ..., authService: AuthService = .shared) { ... }

    func fetchFriends(query:limit:offset:) async throws -> [CommunityProfile] { ... }
    func searchPeople(query:limit:offset:) async throws -> [CommunityProfile] { ... }
    func requestFriend(userID:) async throws { ... }
    func respondToFriendRequest(userID:accept:) async throws { ... }
    func removeFriend(userID:) async throws { ... }
    func fetchFriendshipStatus(with:) async throws -> FriendshipStatus { ... }
    func blockUser(userID:) async throws { ... }
    func unblockUser(userID:) async throws { ... }
    // ...
}

// MARK: - RPC Params
private struct FetchFriendsParams: Encodable, Sendable { ... }
private struct SearchPeopleParams: Encodable, Sendable { ... }

// Environment key
private struct FriendsServiceKey: EnvironmentKey { ... }
```

No MARK sections inside the class body for simple services. The functions are just listed in logical order — reads before writes, related operations near each other. RPC parameter structs and record types go after the class, before the environment key.

Services use `static let shared` and a `private init` — singleton pattern with dependency injection through default parameters so tests can substitute.

### When a service gets observable

Most services aren't `@Observable`. But some publish live state that views need to watch — `NavigationService` holds the current route, `CurrentLocationProvider` holds the latest coordinate. These get `@MainActor @Observable` because they're closer to "live system state the UI reads" than "call-and-return boundary."

The distinction: if a view would need to poll the service to detect changes, it should probably be observable. If the view just calls a function and gets a result, it shouldn't.

---

## File anatomy of a coordinator

Coordinators look like a hybrid. They have observable state like a store, but they don't have feature-grouped sections because they serve one purpose: consolidating signals into policy.

```swift
@MainActor
@Observable
final class RuntimePolicyCoordinator {
    // Published signal state
    var isLowPowerModeEnabled: Bool = ...
    var thermalState: ProcessInfo.ThermalState = ...
    var isLowDataModeEnabled: Bool = false
    var isExpensiveNetwork: Bool = false
    var isNetworkAvailable: Bool = true

    // Computed policy outputs
    var policyLevel: PolicyLevel { ... }
    var shouldEnableSocialRealtime: Bool { ... }
    var shouldEnableMapRealtime: Bool { ... }
    var mapSyncInterval: Duration { ... }

    // Private observation infrastructure
    private var observer: NSObjectProtocol?
    private var thermalObserver: NSObjectProtocol?
    private let pathMonitor = NWPathMonitor()

    init() { /* set up observers */ }
    deinit { /* tear down observers */ }
}
```

The layout is: input signals at the top, computed policy outputs in the middle, observation plumbing at the bottom. No MARK sections needed — the file is short enough that the structure is self-evident.

---

## Environment wiring

Every shared dependency has the same three-piece pattern at the bottom of its file:

```swift
// 1. Private key type
private struct SocialStoreKey: EnvironmentKey {
    static let defaultValue = SocialStore()
}

// 2. EnvironmentValues extension
extension EnvironmentValues {
    var socialStore: SocialStore {
        get { self[SocialStoreKey.self] }
        set { self[SocialStoreKey.self] = newValue }
    }
}

// 3. Usage in views
@Environment(\.socialStore) private var socialStore
```

The key struct is always private. The environment property name always matches the type's semantic role — `socialStore`, `authService`, `runtimePolicyCoordinator`. This is the same name you'll use in the `@Environment` declaration in views, so consistency matters.

One thing worth being explicit about: `defaultValue` means different things depending on how you use it. It's the same mechanism, but you're choosing what role it plays.

For **stores** in Rally, `defaultValue` is a fallback — the real instance is owned by a view via `@State` and explicitly passed down with `.environment(...)`. The default is there so environment lookup always resolves, even in previews or if injection is accidentally missed.

For **services** in Rally, `defaultValue = AuthService.shared` is the canonical instance. Nobody owns it with `@State`, nobody injects it — the default is the real thing, because these are app-scoped singletons with no lifecycle to manage.

Either approach works for either type. The important thing is knowing which choice you made and being consistent about it.

---

## Doc comments

Non-trivial store and coordinator functions use a structured doc comment format. If a method is a one-liner or its name already says everything (like a simple getter), don't force the full block on it — use it where the intent, orchestration, or service dependencies aren't obvious from the signature alone.

```swift
/// Refreshes all core community data for initial load and pull-to-refresh.
/// - What: Reloads friends, requests, profile, and dependent people-search state.
/// - How: Chooses load state by mode, performs concurrent fetches, then commits results.
/// - Why: Keeps social state coherent after app/tab entry and manual refresh.
/// - Service Calls:
///   - FriendsService.fetchFriends
///   - ProfileService.fetchProfile
/// - Parameters:
///   - mode: Initial or user-triggered refresh mode.
///   - query: Current search query.
```

The first line is a one-sentence summary. Then What/How/Why break down the intent, mechanism, and reasoning. Service Calls lists every external dependency the function touches (or `None` if it's pure logic). Parameters list what goes in.

Services generally don't need this level of documentation — their functions are straightforward enough that the signature tells you what's happening. A service method called `fetchFriends(query:limit:offset:)` doesn't need a paragraph explaining what it does.

---

## Error handling

Stores centralize error presentation through a private helper:

```swift
private func presentFailure(
    _ error: Error,
    title: String,
    description: String,
    source: String,
    updateLoadState: Bool = true
) {
    if error.isExpectedError() { return }

    let viewError = CommunityViewError(
        title: title,
        description: description,
        technicalDetails: "Source: \(source)\nType: ..."
    )

    if updateLoadState {
        loadState = .failed(viewError)
    }
    presentedError = viewError
}
```

Every intent method in the store wraps its service calls in do/catch and routes failures through this helper. The `updateLoadState` parameter matters — mutation failures (like a failed friend request) set `presentedError` for an alert but don't blow away the loaded content. Only full-refresh failures transition `loadState` to `.failed`.

Services don't centralize errors. They throw, and the calling store decides what to do. A service function either succeeds or throws — it doesn't set UI state.

---

## Logging

Services log with a bracketed prefix:

```swift
print("[Auth] signIn response - user: \(session.user.id)")
print("[Friends] fetchFriends — query: \(query ?? "nil"), limit: \(limit)")
print("[FriendsRealtime] subscribed channel=\(channelName)")
```

The format is `[ComponentName] action — details`. This makes it easy to filter console output by component when debugging. Stores log mostly for realtime lifecycle events (subscribe, event received, stop). Services log for every call and result.

`print` works fine for dev builds. If you need structured logging or log levels in production, swap to `os.Logger` — the bracketed prefix convention translates naturally into subsystem/category names.

---

## Supporting types

Enums, structs, and records that belong to one file live at the bottom of that file, after the main class and before the environment key:

```swift
// After the store class closes
enum CommunityLoadState {
    case loading, loaded, refreshing
    case failed(CommunityViewError)
}

enum CommunityRefreshMode {
    case initial, refresh
}

struct CommunityViewError: Error, Identifiable { ... }

// Then the environment key
private struct SocialStoreKey: EnvironmentKey { ... }
```

For services, RPC parameter structs and response records follow the same placement:

```swift
// After the service class
// MARK: - RPC Params
private struct FetchFriendsParams: Encodable, Sendable { ... }
private struct RequestFriendParams: Encodable, Sendable { ... }

// Response/record types
private struct FriendshipStatusRecord: Codable, Sendable { ... }
enum FriendshipStatus: Sendable, Equatable { ... }
```

Types that are private to the file use `private`. Types that other files need (like `FriendshipStatus` or `CommunityLoadState`) are internal. The placement stays the same either way — after the main type, before the environment key.
