# KMP Bridge Architecture

> **Official reference**: https://kotlinlang.org/docs/multiplatform/multiplatform-ios-integration-overview.html

## The Bridge Model

KMP for iOS compiles shared Kotlin code into a native iOS framework that is consumed only
by the bridge layer. **The integration method does not affect the bridge patterns** — the
same interactor, service, and mapper code works with any of the three integration approaches.
Feature modules receive pure Swift types through `@Sendable` closures and never touch the KMP framework.

```
┌─────────────────────────────────────────────┐
│  Kotlin (KMP)                                │
│  Shared business logic, domain models        │
│  Coroutines, Flows, suspend functions        │
└─────────────────────┬───────────────────────┘
                      │  iOS framework (direct / SwiftPM / CocoaPods)
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

> **Feature modules NEVER import the KMP framework.** They receive `@Sendable` closures
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

## iOS Framework Integration Methods

> **Official reference**: https://kotlinlang.org/docs/multiplatform/multiplatform-ios-integration-overview.html

Kotlin produces a standard Apple framework (`.framework` or `.xcframework`). Choose the
integration method that fits your project structure — the bridge patterns are identical in all cases.

| Method | How it works | Best for |
|--------|-------------|----------|
| **Direct integration** | Xcode run script calls `embedAndSignAppleFrameworkForXcode` Gradle task during every build | Monorepo, simultaneous iOS+KMP development; default in KMP IDE plugin |
| **SwiftPM local package** | KMP framework declared as a local `.binaryTarget` in `Package.swift` | Monorepo with SPM-based iOS project |
| **CocoaPods local podspec** | KMP framework connected via local `.podspec` | Projects with CocoaPods dependencies |
| **SwiftPM remote (XCFramework)** | XCFramework published as a GitHub release; consumed as `.binaryTarget` with URL + checksum | Separated codebases; third-party SDK distribution |
| **CocoaPods remote (XCFramework)** | XCFramework distributed through CocoaPods | Projects using CocoaPods with a separated codebase |

### Option A: Direct integration (most common for monorepos)

```kotlin
// build.gradle.kts — no extra XCFramework task needed
kotlin {
    iosArm64()
    iosSimulatorArm64()
    // The embedAndSignAppleFrameworkForXcode task is registered automatically
}
```

```bash
# Xcode run script phase (before "Compile Sources"):
cd "$SRCROOT/.."
./gradlew :shared:embedAndSignAppleFrameworkForXcode
```

The framework is built and embedded by Gradle during every Xcode build. The Swift import is identical to other methods:
```swift
@preconcurrency import SharedKit  // same import regardless of integration method
```

### Option B: Remote XCFramework via SwiftPM

```kotlin
// build.gradle.kts
import org.jetbrains.kotlin.gradle.plugin.mpp.apple.XCFramework

val xcf = XCFramework("SharedKit")
listOf(iosArm64(), iosSimulatorArm64()).forEach {
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
    url: "https://example.com/releases/SharedKit.xcframework.zip",
    checksum: "<sha256 checksum>"
    // or use a local path for development:
    // path: "../../shared/build/XCFrameworks/debug/SharedKit.xcframework"
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
