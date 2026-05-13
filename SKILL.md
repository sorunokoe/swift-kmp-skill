---
name: swift-kmp
description: 'KMP bridge standards for iOS. USE FOR: structuring a bridge layer between Kotlin Multiplatform (KMP) and Swift; integrating KMP iOS framework APIs in a Swift app (direct integration or SwiftPM); handling SKIE flows and sealed classes; mapping Kotlin models/errors to Swift domain types; keeping KMP imports confined to the bridge layer. For Compose ↔ SwiftUI embedding, use the swiftui-compose skill.'
argument-hint: 'Describe the Swift-Kotlin bridge integration you need'
applyTo: '**/*.swift'
---

# KMP Swift Bridge Standards

> The goal: **feature modules know nothing about KMP.** They receive `@Sendable` closures
> with pure Swift types. Every KMP import lives in one place — the bridge layer.

---

## Official Documentation

| Topic | Link |
|-------|------|
| Kotlin/Native ↔ Swift/ObjC interop | https://kotlinlang.org/docs/native-objc-interop.html |
| iOS integration methods overview | https://kotlinlang.org/docs/multiplatform/multiplatform-ios-integration-overview.html |
| Swift Export (experimental — future direction) | https://kotlinlang.org/docs/native-swift-export.html |
| SKIE features (flows, sealed classes, suspend) | https://skie.touchlab.co/features |
| Apple `UIViewControllerRepresentable` | https://developer.apple.com/documentation/swiftui/uiviewcontrollerrepresentable |
| Apple `AsyncStream` | https://developer.apple.com/documentation/swift/asyncstream |
| Apple `AsyncSequence` | https://developer.apple.com/documentation/swift/asyncsequence |

---

## Boundary Cheatsheet

| Need | Pattern |
|---|---|
| Call a KMP suspend function from a feature store | Inject as `@Sendable () async throws -> T` closure — never import KMP framework in feature modules |
| Observe a KMP `Flow<T>` | Wrap at the bridge with `AsyncStream(kmpComponent.someFlow)` |
| Map Kotlin sealed class | `onEnum(of:)` in the bridge layer only; return a Swift enum |
| Embed Compose Multiplatform view | `UIViewControllerRepresentable` in the main/bridge target |

---

## Hard Rules

- ❌ **No KMP framework import in feature modules** — only the bridge/main target imports the KMP framework (regardless of integration method)
- ❌ **No `KotlinThrowable` in feature code** — catch at the bridge, re-throw as a typed Swift `Error`
- ❌ **No `SkieSwiftFlow<T>` in feature modules** — wrap to `AsyncStream` at the bridge boundary
- ❌ **Kotlin collections (`KotlinList`, `KotlinArray`) must be mapped to `[T]`** at the bridge, not passed through
- ❌ **No `onEnum(of:)` in feature modules or views** — confine to mapper and bridge layers only

---

## The Core Principle

```
KMP iOS Framework  (direct integration / SwiftPM — see architecture.md)
       ↓  @preconcurrency import — bridge layer only
       │  (Bridge: DI wiring, Interactors, Services, Mappers, Compose helpers)
       ↓  @Sendable closures with pure Swift types only
Feature Modules / SPM Packages
       (know nothing about KMP)
```

Feature modules are fast to build, isolated from KMP churn, and safe — invalid states
are killed in mappers before features ever see data.

---

## Quick Pattern Reference

| Pattern | When to use | Reference |
|---------|-------------|-----------|
| `final class XxxInteractor: Sendable` | Request/response KMP calls | `references/interactor-pattern.md` |
| `@concurrent nonisolated func` | Interactor bridge methods (leaves main actor) | `references/interactor-pattern.md` |
| `onEnum(of: result)` | Kotlin sealed class dispatch (bridge layer only) | `references/type-mapping.md` |
| `SkieSwiftFlow<T>` direct exposure | Service owns a long-lived stream (bridge layer only) | `references/flow-bridging.md` |
| `AsyncStream` wrapping | Re-export stream to feature modules with mapping | `references/flow-bridging.md` |
| Stateless Compose embedding | `UIViewControllerRepresentable` with empty `updateUIViewController` | [`swiftui-compose` skill](https://github.com/sorunokoe/swiftui-compose-skill) |
| Bidirectional Compose embedding | `makeCoordinator()` + `update:` closure | [`swiftui-compose` skill](https://github.com/sorunokoe/swiftui-compose-skill) |

---

## Reference Router

> **Token budget**: This SKILL.md (~1.7k tokens) contains hard rules and a review checklist.
> **Complete most bridge tasks using only this file.** Load a reference only when you need
> the full API contract for that specific topic. Load at most ONE reference per task step.

| Reference | Load when |
|-----------|-----------|
| `references/architecture.md` | Layer ownership is unclear; iOS framework integration setup; `Feature.initialize` pattern |
| `references/interactor-pattern.md` | Creating/reviewing an Interactor or Service class |
| `references/flow-bridging.md` | Bridging Kotlin flows; SKIE vs AsyncStream choice |
| `references/type-mapping.md` | Kotlin sealed types, error mapping, collection mapping, Sendable |

---

## Review Checklist

- [ ] KMP framework imports stay in the bridge layer (main target / bridge module); never in feature modules
- [ ] **If Kotlin API surface changed**: xcframework rebuilt before writing any Swift bridge code (see `references/architecture.md` — SKIE Output as API Contract)
- [ ] Prefer `@preconcurrency import <KmpFramework>` on SKIE-heavy bridge files
- [ ] Feature modules have no KMP framework import anywhere
- [ ] KMP app components are injected via your DI mechanism, never instantiated directly in bridge classes
- [ ] Interactor request/response methods use `@concurrent nonisolated func`
- [ ] Kotlin sealed classes dispatched with `onEnum(of:)` — not raw pattern matching
- [ ] Errors caught at bridge boundary; `KotlinThrowable` never reaches feature code
- [ ] `Result<T, E>` used where `E` is a typed Swift `enum: Error` defined in the feature module
- [ ] `AsyncStream` wrapping includes `continuation.onTermination = { _ in task.cancel() }`
- [ ] `SkieSwiftFlow<T>` is never passed into feature modules
- [ ] Data model types crossing to feature modules are `Sendable` (`struct`/`enum` preferred)
- [ ] Kotlin collections are mapped to `[T]` at the bridge — not passed as `KotlinList<T>`

---

## Transitional Patterns (explicitly temporary)

- `@preconcurrency import <KmpFramework>` is required today for SKIE-generated APIs; treat as a
  bridge-only compatibility shim, not a feature-module pattern.
- `onEnum(of:)` dispatch is SKIE-era interoperability. JetBrains' **Swift Export** currently
  maps Kotlin `enum class` to native Swift `enum`; sealed class/interface bridging as native
  Swift types is not yet in Swift Export's documented feature set
  (see https://kotlinlang.org/docs/native-swift-export.html).
  Plan migrations toward native `switch` as Swift Export stabilizes.
- Keep transition helpers isolated in mapper/interactor layers so future migration edits are localized.

---

## Out of Scope

This skill does NOT cover:
- State management and feature architecture → see your architecture skill
- DI container wiring (FactoryKit/Koin/etc.) → see your DI standards skill
- SPM package setup and concurrency settings → see your SPM module template skill
- Swift concurrency rules (actors, Sendable, MainActor) → see your Swift concurrency skill
- Compose Multiplatform ↔ SwiftUI interop (`UIViewControllerRepresentable` wiring, coordinator pattern, teardown) → see [`swiftui-compose` skill](https://github.com/sorunokoe/swiftui-compose-skill)
