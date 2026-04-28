# KMP Bridge Architecture

> **Official reference**: https://kotlinlang.org/docs/multiplatform/multiplatform-ios-integration-overview.html

## The Bridge Model

KMP for iOS works by compiling shared Kotlin code into an **XCFramework** that is consumed only
by the bridge layer. Feature modules (SPM packages or sub-targets) receive pure Swift types
through `@Sendable` closures — they never touch the KMP framework.

```
┌─────────────────────────────────────────────┐
│  Kotlin (KMP)                                │
│  Shared business logic, domain models        │
│  Coroutines, Flows, suspend functions        │
└─────────────────────┬───────────────────────┘
                      │  .xcframework (via SwiftPM binaryTarget or local path)
                      ▼
┌─────────────────────────────────────────────┐
│  Bridge Layer (main/host target)             │
│  DI wiring                                  │  ← app component singleton
│  Interactors                                │  ← request/response KMP calls
│  Services                                  │  ← long-lived stream/lifecycle bridge
│  Mappers                                   │  ← kill invalid states here
│  Compose helpers                           │  ← UIViewControllerRepresentable wrappers
└─────────────────────┬───────────────────────┘
                      │  @Sendable Swift closures only
                      │  ( fetchData: () async -> Result<T, E> )
                      ▼
┌─────────────────────────────────────────────┐
│  Feature Modules (SPM packages / sub-targets)│
│  Know nothing about KMP                     │
│  Operate on pure Swift types only           │
└─────────────────────────────────────────────┘
```

---

## The Module Boundary Rule

> **Feature modules NEVER import the KMP XCFramework.** They receive `@Sendable` closures
> that return pure Swift types. The word "Kotlin" does not appear in their source code.

Benefits:
- **Fast builds** — feature modules compile only their own code
- **Isolation** — KMP changes don't cascade into feature code
- **Replaceability** — KMP can be swapped without touching features
- **Safety** — invalid states killed in mappers before features see data

### `@preconcurrency import` placement

```swift
// ✅ Bridge layer only — any of these locations:
// Sources/Bridge/DI/ContainerKMP.swift
// Sources/Bridge/Features/Profile/ProfileInteractor.swift
// Sources/Bridge/Services/AuthService.swift
@preconcurrency import YourKmpFramework

// ❌ Never in feature modules:
import YourKmpFramework  // Must not exist in any feature module file
```

---

## XCFramework Build Pipeline

> **Official reference**: https://kotlinlang.org/docs/multiplatform/multiplatform-build-native-binaries.html

```kotlin
// build.gradle.kts (Kotlin side)
val xcf = XCFramework("SharedKit")
listOf(iosArm64(), iosX64(), iosSimulatorArm64()).forEach {
    it.binaries.framework {
        baseName = "SharedKit"
        xcf.add(this)
    }
}
```

```swift
// Package.swift — consume as a SwiftPM binary target
.binaryTarget(
    name: "SharedKit",
    path: "../../shared/build/XCFrameworks/debug/SharedKit.xcframework"
    // or a remote URL with checksum for distribution
)
```

Build variants to consider:
- **Debug/QA**: simulator-only builds are faster (saves CI time)
- **Release**: always build all architectures (`iosArm64` + `iosSimulatorArm64` minimum)

---

## App Component Singleton

The Kotlin application component is created once and shared across all interactors and services.
It should never be instantiated inside individual interactors.

```swift
// In your DI wiring (bridge layer):
// The component is created once and injected into interactors via Configuration.

// Using a simple singleton pattern (DI-framework-agnostic):
final class KmpComponentHolder: Sendable {
    static let shared = KmpComponentHolder()
    let component: AppComponent = {
        let c = AppComponent.companion.create()
        c.initializers.initialize()
        return c
    }()
}

// Or wired through your DI container of choice (FactoryKit, Resolver, manual, etc.)
```

All interactors receive the component through their `Configuration` — they never create it.

---

## Feature Contract: `Feature.initialize(with:)`

The standard pattern for decoupling features from the bridge:

```swift
// Defined in the feature module — uses only the feature's own Swift types:
public struct Configuration {
    public let fetchProfileData: @Sendable () async -> Result<ProfileData, ProfileError>
    public let fetchSettings: @Sendable () async -> Result<[Setting], ProfileError>
    // No KMP types, no AppComponent references
}

// Wired in the bridge layer:
let interactor = ProfileInteractor(configuration: .init(
    component: kmpComponent,
    mapper: ProfileMapper()
))

let feature = ProfileFeature.initialize(with: .init(
    fetchProfileData: interactor.fetchProfileData,
    fetchSettings: interactor.fetchSettings
))
```

Interactor methods marked `@concurrent nonisolated` are implicitly `@Sendable` and match
closure signatures directly — no adapter wrapper needed.

---

## What Lives Where

| Location | Contents |
|----------|----------|
| Bridge layer (main target) | KMP imports, interactors, services, mappers, Compose helpers, DI wiring |
| Feature module | Pure Swift types, TCA/MVVM stores, SwiftUI views, feature-local errors |
| Both | Protocol definitions for testability (feature-owned, bridge implements) |

Never let KMP types "leak" upward from the bridge into feature modules.
