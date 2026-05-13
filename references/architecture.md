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
                      │  iOS framework (direct integration / SwiftPM)
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
| **SwiftPM remote (XCFramework)** | XCFramework published as a GitHub release; consumed as `.binaryTarget` with URL + checksum | Separated codebases; third-party SDK distribution |

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

## SKIE Output as API Contract

The xcframework built from your KMP shared module contains SKIE-generated `.swiftinterface`
files. These files are the **authoritative source of truth** for the Swift-visible API surface:

- `onEnum(of:)` case names (derived from Kotlin subtype names and `@ObjCName` annotations)
- Type names visible to Swift (controlled by `@ObjCName`)
- Property nullability (Kotlin optionality mapped to Swift optionality)
- Function signatures visible across the bridge

```
YourKmpFramework.xcframework/
└── ios-arm64_x86_64-simulator/
    └── YourKmpFramework.framework/
        └── Modules/
            └── YourKmpFramework.swiftmodule/
                └── arm64-apple-ios-simulator.swiftinterface  ← read this
```

### When to rebuild before writing Swift

Kotlin changes that modify the Swift-visible API surface require **rebuilding the xcframework
before writing any Swift bridge code**. Writing Swift against stale SKIE output produces code
that won't compile.

The Swift-visible API surface is defined by types annotated with `@ObjCName` (or subtypes
of those types).

**Rebuild required when a `@ObjCName`-annotated type (or its subtypes) has:**

| Change | Why it matters |
|--------|---------------|
| Property added, removed, or renamed | Changes the generated Swift property name |
| Sealed subtype added, removed, or renamed | Changes available `onEnum` case names |
| Supertype or interface delegation changed | May add/remove inherited Swift-visible members |
| `@ObjCName` annotation added or removed | Changes whether the type is Swift-visible at all |

**Rebuild NOT required when the Kotlin change is:**

| Change | Reason |
|--------|--------|
| In test source sets (`commonTest`, `*Test.kt`) | Test code is not included in the iOS framework |
| In Android-only code (`androidMain`) | Not compiled into the iOS framework |
| Only to `private` members | Private API is never emitted to `.swiftinterface` |
| Only to a function body with no signature change | The ABI surface is unchanged |
| Only to Gradle or build files | No Kotlin source is modified |

### Best practice: read `.swiftinterface` first

After rebuilding the xcframework following a Kotlin API change, read the relevant
`.swiftinterface` file **before** writing bridge code. The Swift type names, case names,
and nullability in that file are what the compiler will enforce — not the Kotlin source.

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

All bridge classes receive the KMP component through constructor injection — they never create it.

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
| Feature module | Pure Swift types, stores/view models/coordinators, SwiftUI views, feature-local errors |
| Both | Protocol definitions for testability (feature-owned, bridge implements) |

Never let KMP types "leak" upward from the bridge into feature modules.
