# Type Mapping — KMP Bridge

> The goal: *"If it compiles, it will work as intended — no invalid state."*
> Invalid states are chased into mappers and killed with tests. Feature code beyond mappers
> lives in a "safe space" where types express exactly what is valid.

---

## Official Documentation

| Topic | Link |
|-------|------|
| Kotlin/Native ↔ Swift/ObjC interop | https://kotlinlang.org/docs/native-objc-interop.html |
| SKIE features (sealed, flows, defaults) | https://skie.touchlab.co/features |
| Swift Export (experimental) | https://kotlinlang.org/docs/native-swift-export.html |

---

## The Two-Mapper Chain

Every piece of data flows through two mappers before reaching SwiftUI:

```
Kotlin (raw API / network response)
    ↓
KMP Domain Mapper  (Kotlin side — kills invalid server data, produces domain model)
    ↓
Kotlin Domain Model  (clean, structured — exposed via AppComponent)
    ↓
Swift Presentation Mapper  (bridge layer — kills invalid domain data, produces SwiftUI-ready types)
    ↓
Swift Presentation Model  (SwiftUI-ready: no loose optionals, no unhandled enum cases)
    ↓
Feature Module  ("safe space" — no invalid state possible)
```

### Control-flow vs Data-flow

- **Mappers**: contain `if/else`, `guard`, `switch` — intentional. Bugs live here; tests target here.
- **Feature code ("safe space")**: operates on data-flow — `map`, `flatMap`, transformations.
  No unexpected `nil`, no unhandled cases.

---

## Kotlin Sealed Classes: `onEnum(of:)`

SKIE bridges Kotlin `sealed class` / `sealed interface` to Swift using `onEnum(of:)`.

> ⚠️ **Transitional pattern.** JetBrains' **Swift Export** will produce native Swift enums
> directly. Keep `onEnum(of:)` confined to bridge layer mappers and interactors — this makes
> the future migration to `switch` on Swift enums a localized refactor.
> Reference: https://kotlinlang.org/docs/native-swift-export.html

```swift
// ✅ Current pattern:
switch onEnum(of: result) {
case let .success(data):
    return .success(mapper.map(from: data.data))
case let .failure(error):
    logger.error("\(String(describing: error), privacy: .public)")
    return .failure(.noData)
}

// ❌ Do not switch on KMP sealed types directly:
switch result {  // result is a Kotlin object — pattern matching does not work correctly
case let s as KotlinResultSuccess: …
```

### Nested sealed types

For nested sealed hierarchies (e.g. `KmpResult<T>` wrapping inner domain results):

```swift
switch onEnum(of: outerResult) {
case let .success(wrapper):
    // wrapper.data may itself be a sealed type — use onEnum again if needed
    return .success(mapper.map(from: wrapper.data))
case let .failure(wrapper):
    return .failure(.networkError)
}
```

---

## Kotlin Errors: `KotlinThrowable` → Swift Domain Error

Kotlin `suspend` functions throw `KotlinThrowable` on runtime failure.
**Always catch at the bridge. Never let `KotlinThrowable` reach feature code.**

```swift
// ❌ Leaking to feature code:
func fetchData() async throws -> SomeData {
    try await component.interactor.fetchData()
    // throws KotlinThrowable — feature code cannot meaningfully handle this
}

// ✅ Map to typed Swift domain error at the bridge:
func fetchData() async -> Result<SomeData, AppError> {
    do {
        let result = try await component.interactor.fetchData()
        switch onEnum(of: result) {
        case let .success(data): return .success(mapper.map(from: data.data))
        case let .failure(error):
            logger.error("\(String(describing: error), privacy: .public)")
            return .failure(.noData)
        }
    } catch {
        // KotlinThrowable caught here — never reaches features
        logger.critical("\(String(describing: error), privacy: .public)")
        return .failure(.noData)
    }
}
```

Feature module `AppError` is a plain Swift `enum: Error` with no KMP types.

---

## Migration Path: SKIE → Swift Export

When Swift Export is available for a bridged type:
1. Add a focused mapper test first (before refactor) to pin existing behavior.
2. Replace `onEnum(of:)` in that mapper with native `switch` on the exported Swift enum.
3. Drop `@preconcurrency import` only when the specific import path no longer needs it.
4. Keep output types to feature modules unchanged — the migration is invisible to feature code.

---

## Kotlin Collections → Swift Collections

Map Kotlin collections at the bridge boundary. Never pass `KotlinList<T>` or `KotlinArray<T>`
into feature modules.

```swift
// ❌ Passing raw Kotlin collection:
func getItems() -> KotlinList<KotlinItem>  // feature modules can't idiomatically use this

// ✅ Map to Swift array at the bridge:
func getItems() -> [Item] {
    configuration.component.interactor.getItems().map { Item(from: $0) }
}

// For optional SKIE bridging with possible nil:
let items = (kotlinResult.data as? [KotlinDomainItem])?.map(Item.init) ?? []
```

---

## `Sendable` Contract for Feature-Crossing Types

**All types that cross from the bridge into feature modules must be `Sendable`.**

```swift
// ✅ Data model — value type (Sendable by default):
public struct ProfileData: Equatable {
    public let displayName: String
    public let avatarURL: URL?
}

// ✅ Dependency object — final class with explicit Sendable conformance:
public final class RefreshTrigger: Sendable {
    private let _trigger = OSAllocatedUnfairLock(initialState: 0)
    public func trigger() { _trigger.withLockUnchecked { $0 += 1 } }
}

// ❌ Not acceptable — mutable reference type, data race risk:
class DataSource: ObservableObject {
    @Published var items: [Item] = []
}
```

---

## What NEVER Crosses Into Feature Modules

| Type | Why |
|------|-----|
| `AppComponent` (or your KMP component class) | KMP runtime — features must not call KMP |
| Any KMP framework class | Defeats the module boundary |
| `KotlinThrowable` | Untyped; features cannot meaningfully handle it |
| `KotlinList<T>`, `KotlinArray<T>` | Not Swift-native |
| `SkieSwiftFlow<T>` | Requires KMP import; wrap in `AsyncStream` |
| Mutable reference types without `Sendable` | Data race risk |

---

## Mapper Pattern

```swift
// Pure-Swift mapper — no KMP types visible to callers:
struct ProfileMapper: Sendable {
    func map(from raw: KmpProfileDomain) -> ProfileData {
        ProfileData(
            displayName: raw.displayName ?? "Unknown",
            avatarURL: raw.avatarUrl.flatMap(URL.init)
        )
    }
}
```

Mappers live in the bridge layer and are injected into interactors via `Configuration`.
They should be stateless `struct`s or `final class: Sendable` for thread safety.
