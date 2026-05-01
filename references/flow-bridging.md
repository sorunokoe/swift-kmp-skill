# Kotlin Flow Bridging — KMP Bridge

KMP bridges Kotlin `Flow<T>` to Swift via two patterns. SKIE (Swift Kotlin Interface Enhancer)
generates `SkieSwiftFlow<T>` which conforms to `AsyncSequence`. Choose the pattern based on
whether the stream stays in the bridge layer or needs to cross into a feature module.

> **SKIE reference**: https://skie.touchlab.co/features  
> **Apple AsyncStream docs**: https://developer.apple.com/documentation/swift/asyncstream  
> **Apple AsyncSequence docs**: https://developer.apple.com/documentation/swift/asyncsequence  

### Why `AsyncStream`?

Apple's documentation explains the intent directly:

> *"An asynchronous sequence generated from a closure that calls a continuation to produce
> new elements. `AsyncStream` conforms to `AsyncSequence`, providing a convenient way to
> create an asynchronous sequence without manually implementing an asynchronous iterator.
> In particular, an asynchronous stream is well-suited to adapt callback- or
> delegation-based APIs to participate with `async`-`await`."*
> — [AsyncStream](https://developer.apple.com/documentation/swift/asyncstream)

Kotlin Flows (via SKIE version 0.10.11) are the "callback-based APIs" that `AsyncStream` is designed for.
The `continuation.onTermination` property is the Apple-canonical way to stop the source
when the consumer cancels — **always set it**:

> *"The continuation conforms to `Sendable`, which permits calling it from concurrent
> contexts external to the iteration of the `AsyncStream`."*

---

## Pattern 1: `SkieSwiftFlow` Direct Exposure (Service pattern)

When the service **owns** the stream and exposes it through a protocol, return `SkieSwiftFlow<T>`
directly. `SkieSwiftFlow<T>` conforms to `AsyncSequence`, making it usable with `for await`.

```swift
// Protocol (bridge layer):
protocol AuthServiceProtocol: Sendable {
    var authState: SkieSwiftFlow<AuthState> { get }
    func subscribeToAuthState()
}

// Implementation (bridge layer):
final class AuthService: AuthServiceProtocol {
    private let component: AppComponent

    var authState: SkieSwiftFlow<AuthState> {
        component.authPresenter.getAuthState()
    }

    func subscribeToAuthState() {
        component.authPresenter.activateAuthState()
    }
}
```

**Use when:**
- The service is in the bridge layer
- Callers are also in the bridge layer (e.g. app coordinator, root navigation)
- You don't need to map Kotlin types before exposing the stream

> ❌ **Never pass `SkieSwiftFlow` into feature modules.** Feature modules receive
> `AsyncStream` or plain `@Sendable` closures only.

---

## Pattern 2: `AsyncStream` Wrapping (Interactor pattern)

When the stream needs to cross into a feature module (or requires type mapping),
wrap it in an `AsyncStream`.

```swift
// In a bridge interactor — wraps SKIE flow, maps Kotlin types:
nonisolated func streamItems() -> AsyncStream<ItemListState> {
    AsyncStream { continuation in
        let task = Task {
            let flow = configuration.component.itemInteractor.streamItems()

            do {
                for try await rawState in flow {
                    if Task.isCancelled { break }
                    if let mapped = configuration.mapper.map(rawState) {
                        continuation.yield(mapped)
                    }
                }
            } catch {
                // SKIE flow terminated — log if needed, always call finish
            }
            continuation.finish()
        }

        // Cancel the driving Task when the consumer cancels:
        continuation.onTermination = { @Sendable _ in
            task.cancel()
        }
    }
}
```

**Key rules:**
1. Create a `Task` inside the `AsyncStream` closure to drive the flow
2. Wrap the loop in `do/catch` — SKIE flows can terminate with an error; `finish()` must always run
3. Guard with `if Task.isCancelled { break }` inside the loop
4. **Always** set `continuation.onTermination = { @Sendable _ in task.cancel() }` — omitting this leaks the Task
5. Call `continuation.finish()` after the `do/catch` to signal stream completion

> For guarding Void KMP calls (pagination, fire-and-forget triggers), see the
> **Cancellation Guard** section in `references/interactor-pattern.md`.

---

## Consuming Streams in TCA / MVVM Stores

Feature stores consume streams via `.run` effects. Always pair with cancellation:

```swift
// TCA Reducer:
case .onAppear:
    return .run { [stream = dependencies.itemClient.streamItems] send in
        for await state in stream() {
            await send(.itemsUpdated(state))
        }
    }
    .cancellable(id: CancelID.itemStream, cancelInFlight: true)

case .onDisappear:
    return .cancel(id: CancelID.itemStream)
```

**Rules:**
- Always pair `.run` with `.cancellable(id:, cancelInFlight: true)` for stream effects
- Cancel on the equivalent of `.onDisappear` or screen dismissal
- `cancelInFlight: true` prevents duplicate subscriptions if the view reappears quickly

For non-TCA architectures, use a `Task` stored in your view model:
```swift
private var streamTask: Task<Void, Never>?

func startObserving() {
    streamTask?.cancel()
    streamTask = Task { [weak self] in
        guard let self else { return }
        for await state in interactor.streamItems() {
            self.items = state.items
        }
    }
}

func stopObserving() {
    streamTask?.cancel()
}
```

---

## Choosing Between Patterns

| Scenario | Pattern |
|----------|---------|
| Service in bridge layer, callers also in bridge | `SkieSwiftFlow` direct exposure |
| Needs mapping Kotlin types → Swift types | `AsyncStream` wrapping |
| Will be passed into a feature module | `AsyncStream` wrapping (MUST) |
| TCA dependency client | `AsyncStream` wrapping (concrete type required) |
| Re-expose as a typed protocol property | `SkieSwiftFlow` (bridge protocol only) |

---

## SKIE: Kotlin Flow → SkieSwiftFlow

SKIE (version 0.10.11) automatically bridges `Flow<T>` to `SkieSwiftFlow<T>` which conforms to `AsyncSequence`.
For sealed classes in the flow, use `onEnum(of:)` on the emitted values in the bridge layer:

```swift
for try await rawValue in skieFlow {
    switch onEnum(of: rawValue) {
    case let .loading(l): continuation.yield(.loading)
    case let .success(s): continuation.yield(.loaded(mapper.map(s.data)))
    case let .error(e):   continuation.yield(.failed(e.message ?? "Unknown"))
    }
}
```

> See `references/type-mapping.md` for full `onEnum(of:)` details and the migration path
> to native Swift enums via Swift Export.
