# Compose View Integration — KMP Bridge

Compose Multiplatform produces a `UIViewController`, not a SwiftUI `View`.
The bridge layer wraps this controller for SwiftUI using `UIViewControllerRepresentable`.

> **Official CMP ↔ SwiftUI reference**: https://kotlinlang.org/docs/multiplatform/swiftui-compose-integration.html  
> **Apple UIViewControllerRepresentable**: https://developer.apple.com/documentation/swiftui/uiviewcontrollerrepresentable  
> **Apple makeCoordinator()**: https://developer.apple.com/documentation/swiftui/uiviewcontrollerrepresentable/makecoordinator()-9vwm8  
> **Apple updateUIViewController**: https://developer.apple.com/documentation/swiftui/uiviewcontrollerrepresentable/updateuiviewcontroller(_:context:)  
> **Apple sample — Using SwiftUI with UIKit (WWDC22)**: https://developer.apple.com/documentation/UIKit/using-swiftui-with-uikit  

### Protocol Declaration (from Apple)

```swift
// @MainActor @preconcurrency protocol UIViewControllerRepresentable : View
// where Self.Body == Never
```

`UIViewControllerRepresentable` is a `@MainActor` protocol. All its methods run on the main actor.

---

## The Anti-Pattern: Don't Do This

```swift
// ❌ Recreating the Compose controller on every SwiftUI state change:
ComposeView(buildController: {
    component.sharedViewsProvider.mapViewController(filters: currentFilters)
})
.id(currentFilters)  // Forces SwiftUI to rebuild the entire UIViewController
```

This creates a new Compose lifecycle on every filter change: visual glitches, wasted memory,
broken state management.

---

## Option A: `ComposeView` — Stateless Embedding

Use when the Compose view manages its own state entirely and never needs to react to SwiftUI.

```swift
struct ComposeView: UIViewControllerRepresentable {
    let buildController: () -> UIViewController

    func makeUIViewController(context: Context) -> UIViewController {
        buildController()
    }

    func updateUIViewController(_ uiViewController: UIViewController, context: Context) {}
}
```

**Usage:**
```swift
ComposeView(buildController: {
    // Kotlin function returning UIViewController:
    component.sharedViewsProvider.chartViewController(data: chartData)
})
```

`buildController` is called **once** per representable lifetime. `updateUIViewController` is
intentionally empty — Compose owns its state internally.

---

## Option B: Stateful Bidirectional — The Coordinator Pattern

Use when SwiftUI state needs to flow into the Compose view (e.g. filters, selected index,
user location toggle) **without** recreating the `UIViewController`.

### Why `makeCoordinator()` Is the Right Tool

Apple's documentation states:

> *"Implement this method if changes to your view controller might affect other parts of your app.
> SwiftUI calls this method before calling `makeUIViewController(context:)`. The system provides
> your coordinator either directly or as part of a context structure when calling the other
> methods of your representable instance."*
> — https://developer.apple.com/documentation/swiftui/uiviewcontrollerrepresentable/makecoordinator()-9vwm8

The coordinator is created **once** per representable lifetime — making it the correct place
to hold the Kotlin `UIViewController` reference.

### Generic `ComposeViewPrepresentable`

```swift
protocol ControllerPrepresentable {
    var controller: UIViewController { get }
}

struct ComposeViewPrepresentable<T: ControllerPrepresentable>: UIViewControllerRepresentable {
    let build: () -> T       // called once — creates Kotlin context + UIViewController
    var update: (T) -> Void  // called on every SwiftUI state change

    func makeCoordinator() -> T {
        build()  // ← Kotlin UIViewController created here, exactly once
    }

    func makeUIViewController(context: Context) -> UIViewController {
        context.coordinator.controller
    }

    func updateUIViewController(_ uiViewController: UIViewController, context: Context) {
        update(context.coordinator)
    }
}
```

### Kotlin Side Requirements

The Kotlin `UIViewController` holder must:
1. Expose an `update*` method for each piece of state SwiftUI controls
2. Conform to `ControllerPrepresentable` (via a Swift extension in the bridge layer)

```swift
// Bridge layer — conformance declared here, not in feature modules:
extension MapViewControllerHolder: ControllerPrepresentable {}
```

On the Kotlin side, `MapViewControllerHolder` updates Compose state via `mutableStateOf`:
```kotlin
// Kotlin (shared code):
class MapViewControllerHolder {
    private val filtersState = mutableStateOf<List<MapFilter>>(emptyList())

    fun updateFilters(newFilters: List<MapFilter>) {
        filtersState.value = newFilters
    }

    // Compose reads filtersState automatically
}
```

---

## Teardown: `dismantleUIViewController`

> *"Use this method to perform additional clean-up work related to your custom view
> controller. For example, you might use this method to remove observers."*
> — [dismantleUIViewController(_:coordinator:)](https://developer.apple.com/documentation/swiftui/uiviewcontrollerrepresentable/dismantleuiviewcontroller(_:coordinator:))

Add a static `dismantleUIViewController` to clean up resources when the Compose view
leaves the SwiftUI hierarchy:

```swift
extension ComposeViewPrepresentable {
    static func dismantleUIViewController(
        _ uiViewController: UIViewController,
        coordinator: Coordinator
    ) {
        // Cancel any Task or subscriptions held by the coordinator
        coordinator.teardown()
    }
}

final class Coordinator {
    var context: Context?
    private var tasks: [Task<Void, Never>] = []

    func addTask(_ task: Task<Void, Never>) { tasks.append(task) }
    func teardown() { tasks.forEach { $0.cancel() } }
}
```

---

## Full Wiring Example (Bidirectional Map)

```swift
// Bridge layer DI wiring:
func makeMapView(
    filters: [MapFilter],
    onMarkerClick: @escaping (String) -> Void
) -> ComposeViewPrepresentable<MapViewControllerHolder> {
    let sharedViewsProvider = kmpComponent.sharedViewsProvider
    let filterMapper = MapFilterMapper()

    return ComposeViewPrepresentable {
        // build — called once: creates the Kotlin map controller
        sharedViewsProvider.mapViewController(
            filters: filterMapper.map(filters),
            onMarkerClick: onMarkerClick
        )
    } update: { [weak filterMapper] context in
        // update — called on every SwiftUI @State change
        guard let filterMapper else { return }
        context.updateFilters(newFilters: filterMapper.map(filters))
    }
}
```

**Bidirectional data flow:**
- **SwiftUI → Compose**: `update` closure calls `context.updateFilters(…)` — Compose reacts via `mutableStateOf`
- **Compose → SwiftUI**: `onMarkerClick` callback is provided at build time, called by Compose


---

## Feature Module Pattern (Generic Type Parameter)

Feature modules should never depend on `KotlinViewControllerHolder` types. Use a generic
type parameter constrained to `View` — the concrete type is resolved by the bridge layer:

```swift
// In the feature module — no KMP dependency:
public struct Configuration<MapView: View> {
    public var mapView: @MainActor ([MapFilter]) -> MapView
}

public struct MyFeature<MapView: View> {
    public static func initialize(with config: Configuration<MapView>) -> Self { … }
}

// In the bridge layer — concrete type resolved here:
let feature = MyFeature<ComposeViewPrepresentable<MapViewControllerHolder>>.initialize(
    with: .init(
        mapView: { filters in
            makeMapView(filters: filters, onMarkerClick: { … })
        }
    )
)
```

The `MapViewControllerHolder` KMP type is never seen by the feature module.
Swift's generic specialization happens at the call site (bridge layer).

---

## Summary: Lifecycle Callbacks

| Callback | Called | Purpose |
|----------|--------|---------|
| `makeCoordinator()` | Once | Creates Kotlin context (UIViewController + state holder) |
| `makeUIViewController(context:)` | Once | Returns the controller from the coordinator |
| `updateUIViewController(_, context:)` | Every SwiftUI state change | Pushes new state into Compose via context methods |

See also: `swiftui-compose` skill for the general Compose Multiplatform ↔ SwiftUI integration
guide (including `UIKitViewController` for the reverse direction).
