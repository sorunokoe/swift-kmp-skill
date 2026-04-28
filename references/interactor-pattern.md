# Interactor Pattern — KMP Bridge

Interactors are the primary bridge unit between KMP and Swift feature modules. They are
**stateless**, request/response adapters that call KMP suspend functions and map results
to typed Swift domain values.

---

## Full Template

```swift
import Foundation
@preconcurrency import YourKmpFramework
import os
// import the feature module that defines the output types

final class ProfileInteractor: Sendable {

    // MARK: - Configuration

    struct Configuration: Sendable {
        let component: AppComponent        // injected — never created here
        let mapper: ProfileMapper          // pure-Swift mapper, also Sendable
    }

    // MARK: - Properties

    private let logger = Logger(subsystem: "com.example.app", category: "ProfileInteractor")
    private let configuration: Configuration

    // MARK: - Initialization

    init(configuration: Configuration) {
        self.configuration = configuration
    }

    // MARK: - Bridge Methods

    @concurrent
    nonisolated func fetchProfileData() async -> Result<ProfileData, ProfileError> {
        do {
            let result = try await configuration.component.profileInteractor.fetchProfile()
            switch onEnum(of: result) {
            case let .success(data):
                return .success(configuration.mapper.map(from: data.data))
            case let .failure(error):
                logger.error("\(String(describing: error), privacy: .public)")
                return .failure(.noData)
            }
        } catch {
            logger.critical("\(String(describing: error), privacy: .public)")
            return .failure(.noData)
        }
    }
}
```

---

## `@concurrent nonisolated` — Why It's Required

**Interactor request/response methods** that call KMP suspend functions must be
`@concurrent nonisolated`.

- `@concurrent` — runs on the Swift concurrency thread pool, not the main actor
- `nonisolated` — method has no actor isolation; callers can invoke from any context

> **Official reference**: https://developer.apple.com/documentation/swift/asyncsequence
> Swift 6.2 `@concurrent` is the idiomatic way to leave actor isolation for background work.

```swift
// ✅ Correct — leaves main actor, safe for KMP suspend functions
@concurrent
nonisolated func fetchData() async -> Result<DataModel, AppError> { … }

// ❌ Wrong — blocks main actor during KMP call (dangerous on @MainActor apps)
func fetchData() async -> Result<DataModel, AppError> { … }
```

Because interactors are `Sendable` and methods are `nonisolated`, the method references
are implicitly `@Sendable` and satisfy feature module closure signatures directly.

---

## `onEnum(of:)` — Kotlin Sealed Class Dispatch

SKIE bridges Kotlin `sealed class` / `sealed interface` to Swift using `onEnum(of:)`.

> **SKIE reference**: https://skie.touchlab.co/features
> ⚠️ This is the SKIE-era approach. **Swift Export** will produce native Swift enums.
> See https://kotlinlang.org/docs/native-swift-export.html for the forward-looking path.
> Treat every `onEnum(of:)` usage as transitional and keep it in bridge code only.

```swift
// ✅ Current pattern:
switch onEnum(of: result) {
case let .success(data):
    return .success(mapper.map(from: data.data))
case let .failure(error):
    logger.error("\(String(describing: error), privacy: .public)")
    return .failure(.noData)
}

// ❌ Do not switch on Kotlin sealed types directly:
switch result {  // Kotlin object — pattern matching does not work correctly
case let success as KotlinResult.Success: …
```

---

## Error Handling

**Rule: catch Kotlin errors at the interactor boundary. `KotlinThrowable` must never
reach feature modules.**

```swift
// ✅ Two layers of error handling:
@concurrent
nonisolated func fetchData() async -> Result<DataModel, AppError> {
    do {
        let result = try await configuration.component.dataInteractor.fetch()
        switch onEnum(of: result) {
        case let .success(data):
            return .success(configuration.mapper.map(from: data.data))
        case let .failure(error):
            // KMP business logic failure — map to Swift domain error
            logger.error("\(String(describing: error), privacy: .public)")
            return .failure(.noData)
        }
    } catch {
        // Swift/ObjC exception from KMP runtime — log as critical
        logger.critical("\(String(describing: error), privacy: .public)")
        return .failure(.noData)
    }
}
```

Return `Result<T, E>` where:
- `T` is a pure Swift value type defined in the feature module
- `E` is a Swift `enum` conforming to `Error`, also defined in the feature module

---

## Cancellation Guard

For methods that make multiple sequential KMP calls, guard against cancellation between steps:

```swift
@concurrent
nonisolated func loadNextPage() async {
    guard !Task.isCancelled else { return }

    do {
        try await configuration.component.listInteractor.loadMore()
    } catch {
        logger.error("loadNextPage error: \(String(describing: error), privacy: .public)")
    }
}
```

---

## Logging Conventions

Use `os.Logger` — never `print()` (print is a lint error in Swift projects):

```swift
private let logger = Logger(subsystem: "com.example.app", category: "ProfileInteractor")

logger.debug("\(result, privacy: .public)")           // Non-sensitive debug info
logger.error("\(String(describing: error), privacy: .public)")   // Business logic failures
logger.critical("\(String(describing: error), privacy: .public)") // KMP runtime crashes
logger.critical("\(String(describing: error), privacy: .private)") // Sensitive data (auth, PII)
```

---

## Interactor vs Service

| | Interactor | Service |
|---|---|---|
| **State** | Stateless | Stateful (owns subscriptions/lifecycle) |
| **Lifecycle** | None — called on demand | Long-lived singleton |
| **Output** | `async -> Result<T, E>` | `SkieSwiftFlow<T>` or `AsyncStream<T>` |
| **Examples** | ProfileInteractor, CheckoutInteractor | AuthService, ThemeService |
| **Protocol** | Usually not protocolized | Usually protocolized for testability |

```swift
// Service example — exposes SkieSwiftFlow directly (bridge layer only):
protocol AuthServiceProtocol: Sendable {
    var authState: SkieSwiftFlow<AuthState> { get }
    func login() async
    func logout() async
}

final class AuthService: AuthServiceProtocol {
    private let logger = Logger(subsystem: "com.example.app", category: "AuthService")
    private let component: AppComponent

    var authState: SkieSwiftFlow<AuthState> {
        component.authPresenter.getAuthState()
    }

    func login() async {
        do {
            try await component.authPresenter.login()
        } catch {
            logger.critical("\(String(describing: error), privacy: .private)")
        }
    }
}
```

See `references/flow-bridging.md` for stream consumption patterns.
