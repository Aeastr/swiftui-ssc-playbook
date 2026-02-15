# The SSC Playbook

## What this is

SSC stands for **Service, Store, Coordinator**. It's a way to organize the code that sits between your SwiftUI views and the outside world — your backend, the OS, device sensors, all of it.

The idea is simple: instead of stuffing everything into a ViewModel or letting views do too much, you split responsibilities into three layers based on what kind of work they do. Not every feature needs all three. Most start with just a Service and a View, and that's fine. The layers exist so you have a clear place to put things when complexity shows up.

This isn't a framework or a mandate. It's a structure that came out of building Rally — a real app with social features, map syncing, friend requests, realtime listeners, and runtime constraints like thermal throttling and low power mode. Rally was complex enough that things needed clear homes, and this is what worked. The examples here are pulled directly from that codebase.

SSC is not a rule that every feature must use all three layers. It's not a replacement for anything. It's just a way of thinking about where code goes that scaled well for one particular app and might be useful for yours.

---

## The three layers

### Service — the boundary

A service is your app's interface to something external. A backend API, the auth system, a database, device location, whatever lives outside your app's own state. Services handle the call, parse the response, and hand back domain types. That's it.

They don't know about screens. They don't track loading states. They don't decide what the UI should show. They're tools that other layers use.

Here's what `AuthService` looks like in practice:

```swift
final class AuthService: Sendable {
    static let shared = AuthService()
    private let client: SupabaseClient

    func signIn(email: String, password: String) async throws {
        let session = try await client.auth.signIn(email: email, password: password)
    }

    func signOut() async throws {
        try await client.auth.signOut()
    }

    var currentUserID: UUID? {
        get async {
            let session = await currentSession
            return session?.user.id
        }
    }
}
```

Notice what's missing: no `isLoading` flag, no error state, no UI concern at all. It's a pure boundary — call in, get a result or an error, done.

Services are typically **long-lived and app-scoped**. You create one instance and share it. They're usually **not** `@Observable` because they don't hold state that views need to watch. The exception is something like a location provider that publishes a live coordinate — if a view needs to react to changing values, observability makes sense.

### Store — the feature brain

A store owns the state and orchestration logic for a specific feature or screen. It coordinates multiple services, manages loading/error/data states, and exposes **intent methods** — functions named after what the user is trying to do, not the implementation detail.

`SocialStore` is a good example. It manages the entire social/community feature: friends list, incoming and outgoing requests, people search, and realtime updates. Here's a simplified look at its shape:

```swift
@MainActor
@Observable
final class SocialStore {
    private let authService: AuthService
    private let friendsService: FriendsService
    private let profileService: ProfileService

    var loadState: CommunityLoadState = .loading
    var friends: [CommunityProfile] = []
    var incomingRequests: [FriendRequestItem] = []
    var outgoingRequests: [FriendRequestItem] = []
    var peopleResults: [CommunityProfile] = []

    // Intent: user pulls to refresh
    func refreshAll(mode: CommunityRefreshMode, query: String, searchScope: CommunitySearchScope) async {
        // set load state, fetch friends + requests + profile concurrently, handle errors
    }

    // Intent: user taps "Add Friend"
    func requestFriend(_ userID: UUID) async { ... }

    // Intent: user accepts/declines/cancels a request
    func respondToFriendRequest(_ userID: UUID, action: FriendRequestResponseAction) async { ... }

    // Intent: user swipes to unfriend
    func unfriend(_ profile: CommunityProfile) async { ... }
}
```

A few things to notice about how stores work:

**They coordinate, they don't fetch.** `refreshAll` calls `friendsService.fetchFriends`, `profileService.fetchProfile`, etc. The store decides *when* and *in what order* to call services, but the actual I/O lives in the services.

**They own load/error state.** The store tracks whether data is loading, loaded, refreshing, or failed — and exposes that to the view. Views never manage this themselves.

**Public methods are intents.** `requestFriend`, `unfriend`, `respondToFriendRequest` — these are named after user actions, not implementation details. The view calls `store.unfriend(profile)` and doesn't need to know about the guard against duplicate unfriends, the service call, or the error mapping happening inside.

**Private methods are plumbing.** Things like `fetchAllFriends()` (which handles pagination) or `fetchPendingRequestsWithProfiles()` (which hydrates profile data onto request records) stay private. The view never calls them.

Stores are usually `@Observable` because their whole purpose is exposing state that views react to.

### Coordinator — the cross-cutting policy layer

A coordinator exists for a specific situation: when multiple features need to make the same decision based on shared signals, and you don't want each feature duplicating that logic.

In Rally, `RuntimePolicyCoordinator` watches device constraints — Low Power Mode, thermal state, Low Data Mode, network type — and boils them down into a single policy that the rest of the app can read:

```swift
@MainActor
@Observable
final class RuntimePolicyCoordinator {
    enum PolicyLevel: String {
        case normal
        case constrained
        case critical
    }

    var isLowPowerModeEnabled: Bool = ProcessInfo.processInfo.isLowPowerModeEnabled
    var thermalState: ProcessInfo.ThermalState = ProcessInfo.processInfo.thermalState
    var isLowDataModeEnabled: Bool = false
    var isExpensiveNetwork: Bool = false
    var isNetworkAvailable: Bool = true

    var policyLevel: PolicyLevel {
        if thermalState == .serious || thermalState == .critical { return .critical }
        if isLowPowerModeEnabled || isLowDataModeEnabled { return .constrained }
        return .normal
    }

    var shouldEnableSocialRealtime: Bool {
        policyLevel == .normal && isNetworkAvailable
    }

    var mapSyncInterval: Duration {
        switch policyLevel {
        case .normal:    return isExpensiveNetwork ? .seconds(25) : .seconds(15)
        case .constrained: return .seconds(45)
        case .critical:    return .seconds(60)
        }
    }
}
```

Without this coordinator, every feature that cares about power state would need its own `if isLowPowerMode && thermalState != .critical ...` logic. The map screen, the social realtime listener, and future features would each reinvent the same policy with slightly different bugs. The coordinator centralizes that decision once.

Coordinators are **not** a place for backend calls. They don't fetch data from APIs. They observe system/runtime signals and compute policy outputs. If you find yourself putting `try await someService.fetch(...)` inside a coordinator, that logic probably belongs in a store or service instead.

Most coordinators are `@Observable` since their outputs drive UI behavior.

---

## How they connect

Dependencies flow one way:

```
View → Store / Coordinator → Service
```

A view reads state from a store or coordinator and calls intent methods. A store calls services to do I/O. A coordinator reads system signals and publishes policy. Services don't know about any of the layers above them.

Stores can depend on services. Coordinators can depend on system APIs. A store can read from a coordinator when it needs to (for example, checking `runtimePolicyCoordinator.shouldEnableSocialRealtime` before starting a listener). But services never depend on stores, and stores never depend on views.

### Environment injection and ownership

This is where `@State` and `@Environment` each play a specific role, and understanding the difference matters.

**`@State` means ownership.** The view that declares `@State private var socialStore = SocialStore()` is the one creating and managing that store's lifecycle. When that view goes away, the store goes with it. You can see exactly where the instance lives.

**`@Environment` means access.** A child view that reads `@Environment(\.socialStore) private var socialStore` is borrowing that store — it didn't create it, doesn't control its lifecycle, and doesn't need to care about that. It just uses it.

Here's what this looks like in practice. Rally's `ContentView` is the owner of the shared stores, and it reads services from the environment:

```swift
struct ContentView: View {
    // Ownership — ContentView creates and manages these stores
    @State private var socialStore = SocialStore()
    @State private var missionsStore = MissionsStore()

    // Access — nobody owns these. The environment key's default IS the singleton.
    @Environment(\.authService) private var authService
    @Environment(\.friendsService) private var friendsService
    @Environment(\.profileService) private var profileService
    @Environment(\.runtimePolicyCoordinator) private var runtimePolicyCoordinator

    var body: some View {
        UnderlayTabView {
            UnderlayTab("World Map", systemImage: "map") { MapHome() }
            UnderlayTab("Missions", systemImage: "list.bullet") { MissionsHome() }
            UnderlayTab("Friends", systemImage: "person.2.fill") { SocialHome() }
            // ...
        }
        // Pass the stores down so child views can access them
        .environment(\.socialStore, socialStore)
        .environment(\.missionsStore, missionsStore)
    }
}
```

Two different things are happening here and the distinction matters.

**Stores are owned.** `ContentView` creates `socialStore` and `missionsStore` with `@State`, which means it controls their lifecycle. Then it explicitly injects them into the environment with `.environment(\.socialStore, socialStore)` so child views can access them. There's a clear owner (ContentView) and clear consumers (everything underneath it).

**Services are unowned by default.** In Rally, nobody creates `authService` — there's no `@State private var authService = AuthService()` sitting in the app root. The environment key itself provides the default:

```swift
private struct AuthServiceKey: EnvironmentKey {
    static let defaultValue = AuthService.shared
}
```

When a view writes `@Environment(\.authService) private var authService`, it's reading the environment key's default value, which is the `.shared` singleton. No parent view ever called `.environment(\.authService, something)`. The key just resolves to `.shared` on its own.

You *could* own a service with `@State` and inject it explicitly if you wanted to — maybe you need a service instance scoped to a specific flow, or you want to swap in a different configuration for a subtree. The environment supports that. But for most services it's unnecessary. They're stateless boundaries with no lifecycle to manage, so letting the environment key point to the singleton is the right default.

Stores are different because they hold mutable state that belongs to a specific scope. Someone needs to own that state explicitly, and `@State` at the right level is how you do that.

### Why not just use `AuthService.shared` directly?

If the environment key just resolves to `.shared` anyway, why not skip the ceremony and write `let authService = AuthService.shared` directly in your view?

**Visibility.** When a view declares `@Environment(\.authService) private var authService`, every dependency is listed at the top of the struct. You can glance at the property list and immediately see what this view touches. With `AuthService.shared` called inline somewhere in a method body, dependencies become invisible — you have to read the whole implementation to discover them.

**Flexibility in the view hierarchy.** Environment keys can be overridden at any level with `.environment(\.authService, somethingElse)`. You probably won't do this in production, but it's useful for SwiftUI previews — you can drop in a stub without touching any real code.

**Consistency.** Every dependency — services, stores, coordinators — is accessed the same way: `@Environment(\.theThing)`. You don't have to remember which things are singletons you grab directly and which come from the environment. It's all one pattern.

Store testability is a separate concern — that's handled by init injection, not environment keys. A store takes its services as init parameters with `.shared` as the default, so tests can pass in mocks directly. That works regardless of how views access the service.

One more thing about environment declarations: use the full semantic name. `@Environment(\.socialStore) private var socialStore`, not `var store`. `@Environment(\.authService) private var authService`, not `var service`. When a view has five injected dependencies, generic names become meaningless — and you'll always end up renaming later anyway.

### When does a store go in the environment?

Not every store goes in the environment. The default is **local** — a store owned by the screen that needs it, created as `@State` in that view. You promote a store to shared/environment when its state or lifecycle needs to persist across tabs or screens.

In Rally, `socialStore` is shared because the friends list matters across multiple tabs — the social tab shows it, the map uses friend IDs, and realtime listeners need to survive tab switches. `missionsStore` is shared because mission state is core to both the Missions tab and the Map.

A hypothetical settings-edit store would stay local. Only the settings screen needs that draft state, and it should be created fresh each time the user opens settings. No reason to keep it alive in the environment.

The rule is simple: if losing the store's state when the user switches tabs would break the experience, it belongs in the environment. If it wouldn't matter, keep it local.

---

## When should something be `@Observable`?

Use `@Observable` only when SwiftUI needs to react to changing state.

**Stores** are almost always observable — that's their job, holding state the UI watches.

**Coordinators** are observable when they publish policy that views or stores read reactively (like `policyLevel` or `shouldEnableSocialRealtime`).

**Services** are usually not observable. They perform actions and return results. The exception is a service that publishes live system state, like a location provider exposing a changing coordinate.

---

## When do I add a store? When do I add a coordinator?

Don't start with all three layers for every feature. That's over-engineering.

**Start with Service + View.** If a screen just needs to call an API and show the result, that's enough. A view can call a service directly for simple cases.

**Add a Store when orchestration appears.** The moment you're juggling multiple service calls, tracking load/error states, managing realtime listeners, or computing derived data from multiple sources — that's store territory. `SocialStore` exists because the community screen coordinates three services, manages pagination, runs debounced realtime refreshes, and tracks concurrent unfriend operations. That's too much for a view.

**Add a Coordinator when policy logic repeats across features.** If you find two different stores making the same "should I throttle?" decision based on the same system signals, extract that into a coordinator. Don't add one preemptively.

---

## What does it look like when something's in the wrong place?

**UI flags in services.** If your service has an `isLoading` property, that state belongs in the store that's calling it. The service is a boundary — it goes in, comes out, done.

**Pass-through stores.** A store that just forwards every call to a single service with no added logic isn't pulling its weight. If `store.doThing()` just calls `service.doThing()`, skip the store and let the view call the service directly.

**Cross-feature policy in a feature store.** If your `SocialStore` starts checking thermal state and network type to decide whether to enable realtime — stop. That's coordinator territory. The store should read a coordinator's output (`shouldEnableSocialRealtime`), not reimplement the decision.

**Backend calls in coordinators.** Coordinators observe signals and compute policy. They don't make API calls. If a coordinator is importing your Supabase client, something went wrong.

**Long async chains in views.** If a view's `.task` modifier has a multi-step async pipeline with error handling and state management, that's a store waiting to be extracted.

**Generic dependency names.** `@Environment(\.socialStore) private var store` — when you add a second store, `store` becomes ambiguous. Use the real name from the start.

---

## Doc comments for store and coordinator functions

Every public function on a store or coordinator should have a doc comment that covers **what** it does, **how** it works, and **why** it exists. If it calls services, list them. This matters because stores are where orchestration logic lives — the "what happens when the user taps this" logic — and future-you needs to understand the intent, not just the implementation.

```swift
/// Refreshes all core community data for initial load and pull-to-refresh.
/// - What: Reloads friends, requests, profile, and dependent people-search state.
/// - How: Chooses load state by mode, performs concurrent fetches, then commits results.
/// - Why: Keeps social state coherent after app/tab entry and manual refresh.
/// - Service Calls: FriendsService.fetchFriends, ProfileService.fetchProfile
/// - Parameters:
///   - mode: Initial or user-triggered refresh mode.
///   - query: Current search query.
func refreshAll(mode: CommunityRefreshMode, query: String, ...) async { ... }
```

---

## How does this compare to MVVM?

SSC isn't trying to replace MVVM. It's not a competing architecture. It's just a structure that emerged from building an app that was complex enough to need clear boundaries — social features, realtime listeners, map syncing, runtime policy — and a single ViewModel per screen wasn't giving things a clear enough home.

If you're using MVVM and it works for your app, there's nothing wrong with that. Where it tends to get uncomfortable is when ViewModels start doing too many jobs at once — API calls, loading state, derived data, cross-feature policy — and when two ViewModels need to share state or coordinate with each other.

SSC is what happened when those jobs got pulled apart into layers based on what they actually do: boundary I/O (Service), feature state and orchestration (Store), cross-feature policy (Coordinator). It does ask for more structure up front — more thinking time about what something should be, more deliberate decisions about where code lives. For me, that structure is what kept me sane. When Rally had realtime listeners, concurrent data flows, and runtime policy all in play, knowing exactly where to look for something was worth the upfront cost.

If you're in a codebase that's feeling the pressure, the natural migration path is to extract a Store first — pull the orchestration and state management out of the ViewModel — and only add a Coordinator later if you find policy logic repeating across features.

---

## Testing

Each layer has a natural testing boundary:

**Services** test request/response mapping and error handling. Given these inputs, does the service make the right call and return the right type? Does it surface errors correctly?

```swift
let friends = try await friendsService.fetchFriends(limit: 25, offset: 0)
#expect(friends.count == 3)
```

**Stores** test intent behavior and state transitions. When `refreshAll` is called, does `loadState` go from `.loading` to `.loaded`? When a service call fails, does `presentedError` get set? When `unfriend` is called twice for the same profile, does it guard against the duplicate?

```swift
await socialStore.refreshAll(mode: .initial, query: "", searchScope: .all)
#expect(socialStore.loadState == .loaded)
```

**Coordinators** test signal combinations and policy outputs. When thermal state is `.serious`, is `policyLevel` `.critical`? When Low Power Mode is on but thermal is `.nominal`, is it `.constrained`? Is `mapSyncInterval` correct for each combination?

```swift
coordinator.thermalState = .serious
#expect(coordinator.policyLevel == .critical)
```

**Integration tests** cover critical flows across all three layers — a user action in a view triggering a store intent that calls a service and produces the expected state change.

---

## Boundary snapshot

A quick reference for where things belong. Each line links back to the section that explains why.

| Question | Answer | Section |
|---|---|---|
| Where does API/backend I/O live? | Service | [Service — the boundary](#service--the-boundary) |
| Where does load/error/data state live? | Store | [Store — the feature brain](#store--the-feature-brain) |
| Where does cross-feature policy live? | Coordinator | [Coordinator — the cross-cutting policy layer](#coordinator--the-cross-cutting-policy-layer) |
| Which direction do dependencies flow? | View → Store/Coordinator → Service | [How they connect](#how-they-connect) |
| Who owns a shared store's lifecycle? | The view that declares it as `@State` | [Environment injection and ownership](#environment-injection-and-ownership) |
| Who owns a service's lifecycle? | Nobody — the environment key defaults to `.shared` | [Why not just use `.shared` directly?](#why-not-just-use-authserviceshared-directly) |
| When does a store go in the environment? | When its state must survive tab/screen switches | [When does a store go in the environment?](#when-does-a-store-go-in-the-environment) |
| When do I make something `@Observable`? | When a view needs to react to its changing state | [When should something be `@Observable`?](#when-should-something-be-observable) |
| `isLoading` in a service — okay? | No, that's store state | [What does it look like when something's in the wrong place?](#what-does-it-look-like-when-somethings-in-the-wrong-place) |
| Backend calls in a coordinator — okay? | No, that belongs in a service | [What does it look like when something's in the wrong place?](#what-does-it-look-like-when-somethings-in-the-wrong-place) |
