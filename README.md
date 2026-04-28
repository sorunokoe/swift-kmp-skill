<div align="center">

<pre align="center">
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃                                                    ┃
┃   ◆  swift · kmp                  AI Coding Skill  ┃
┃   ────────────────────────────────────────────     ┃
┃                                                    ┃
┃   KMP  ──────────────────────────────────► Swift   ┃
┃                                                    ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛
</pre>

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Kotlin](https://img.shields.io/badge/Kotlin-2.x-7F52FF?logo=kotlin&logoColor=white)](https://kotlinlang.org)
[![Swift](https://img.shields.io/badge/Swift-6.x-FA7343?logo=swift&logoColor=white)](https://swift.org)
[![SKIE](https://img.shields.io/badge/SKIE-0.8+-blue)](https://skie.touchlab.co)
[![Works with Claude](https://img.shields.io/badge/Works%20with-Claude-9370DB)](https://claude.ai)
[![Works with Codex](https://img.shields.io/badge/Works%20with-Codex-412991?logo=openai&logoColor=white)](https://openai.com/codex)
[![Works with GitHub Copilot](https://img.shields.io/badge/Works%20with-GitHub%20Copilot-2ea44f?logo=github)](https://github.com/features/copilot)
[![Works with Cursor](https://img.shields.io/badge/Works%20with-Cursor-000000)](https://cursor.com)
[![Works with Gemini](https://img.shields.io/badge/Works%20with-Gemini-4285F4?logo=google&logoColor=white)](https://gemini.google.com)

**An AI coding skill for structuring the bridge between  
Kotlin Multiplatform (KMP) and Swift feature modules.**

[What it covers](#what-this-skill-covers) · [Files](#files) · [Core principle](#core-principle) · [Quick start](#quick-start)

</div>

---

## What This Skill Covers

- ✅ **Bridge architecture** — 4-layer model: Kotlin → Bridge → Swift types → Feature modules
- ✅ **Interactor pattern** — request/response KMP calls with `@concurrent nonisolated` bridge methods
- ✅ **Flow bridging** — `SkieSwiftFlow` direct exposure vs `AsyncStream` wrapping with leak-safe teardown
- ✅ **Type mapping** — Kotlin sealed dispatch with `onEnum(of:)`, `KotlinThrowable` containment, collections
- ✅ **Swift Export migration** — transition path from SKIE-era `onEnum(of:)` to native Swift enums
- ✅ **Sendable safety** — what types must be `Sendable` at bridge boundaries and why
- ✅ **Apple doc citations** — `AsyncStream`, `AsyncSequence` official quotes
- 🔗 **Compose embedding** → covered by the companion [`swiftui-compose`](https://github.com/sorunokoe/swiftui-compose-skill) skill

---

## Core Principle

> **Feature modules know nothing about KMP.**  
> They receive `@Sendable` closures with pure Swift types.  
> Every KMP import lives in one place — the bridge layer.

```
KMP iOS Framework  (direct integration / SwiftPM)
       │  @preconcurrency import — bridge layer only
       ▼
  Bridge Layer
  ┌─ Interactors (request/response KMP calls)
  ├─ Services (long-lived stream/lifecycle bridge)
  └─ Mappers (kill invalid state here)
       │  @Sendable closures with pure Swift types only
       ▼
  Feature Modules (SPM packages)
  (no KMP imports — no Kotlin types — no KotlinThrowable)
```

**Benefits**: fast builds, KMP changes don't cascade into feature code, KMP is replaceable,
invalid states are killed in mappers before features ever see data.

---

## Files

| File | Load when |
|------|-----------|
| [`SKILL.md`](SKILL.md) | **Always first** — hard rules, quick cheatsheet, review checklist |
| [`references/architecture.md`](references/architecture.md) | Module boundary unclear; iOS framework integration setup; `Feature.initialize` pattern |
| [`references/interactor-pattern.md`](references/interactor-pattern.md) | Creating/reviewing an Interactor or Service |
| [`references/flow-bridging.md`](references/flow-bridging.md) | Bridging Kotlin Flows; `SkieSwiftFlow` vs `AsyncStream` |
| [`references/type-mapping.md`](references/type-mapping.md) | Kotlin sealed types, error mapping, collections, `Sendable` |

> **Token budget**: `SKILL.md` (~1.7k tokens) covers most tasks. Load one reference only when
> you need the detailed API contract for that topic.

> 🔗 **Embedding Compose views in SwiftUI?** See the companion [**swiftui-compose**](https://github.com/sorunokoe/swiftui-compose-skill) skill — it covers `UIViewControllerRepresentable` wiring, the coordinator pattern, bidirectional state, and teardown.

---

## Quick Start

### GitHub Copilot

```bash
# Copy to your repo:
cp -r swift-kmp /path/to/your-project/.github/skills/
```

Then in Copilot Chat:
```
@swift-kmp Implement a ProfileInteractor that fetches user data from KMP
```

### Claude / Any AI agent

```
Load swift-kmp/SKILL.md, then swift-kmp/references/interactor-pattern.md.

Implement a Swift interactor that:
- Calls component.profileInteractor.fetchProfile() (Kotlin suspend function)
- Maps the KMP result to a Swift ProfileData struct
- Catches KotlinThrowable and returns Result<ProfileData, ProfileError>
```

---

## Key Rules (summary)

- ❌ **No KMP framework import in feature modules** — only the bridge/main target
- ❌ **No `KotlinThrowable` in feature code** — catch and re-throw as typed Swift `Error`
- ❌ **No `SkieSwiftFlow<T>` in feature modules** — wrap to `AsyncStream` at the boundary
- ❌ **No raw Kotlin collections (`KotlinList<T>`)** in feature modules — map to `[T]`
- ✅ **`@concurrent nonisolated` on interactor bridge methods**
- ✅ **`continuation.onTermination` always set** when wrapping Kotlin flows

Full rules and review checklist in [`SKILL.md`](SKILL.md).

---

## Official Documentation

| Topic | Link |
|-------|------|
| Kotlin/Native ↔ Swift/ObjC interop | https://kotlinlang.org/docs/native-objc-interop.html |
| KMP iOS integration overview | https://kotlinlang.org/docs/multiplatform/multiplatform-ios-integration-overview.html |
| SKIE features | https://skie.touchlab.co/features |
| Swift Export (experimental) | https://kotlinlang.org/docs/native-swift-export.html |
| Apple `AsyncStream` | https://developer.apple.com/documentation/swift/asyncstream |
| Apple `UIViewControllerRepresentable` | https://developer.apple.com/documentation/swiftui/uiviewcontrollerrepresentable |

---

## Requirements

| | Version |
|---|---|
| Kotlin | 1.9+ |
| SKIE | 0.8+ |
| Swift | 6.2+ (for `@concurrent`; 5.9+ minimum, 6.x for full concurrency) |
| iOS | 15.0+ |

---

## Related Skills

> 🔗 **Embedding Compose Multiplatform views inside your SwiftUI app?**  
> Check out [**swiftui-compose**](https://github.com/sorunokoe/swiftui-compose-skill) — the companion skill covering `UIViewControllerRepresentable` wiring, the coordinator pattern, bidirectional state sharing, and `dismantleUIViewController` teardown.

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for how to improve patterns or add new reference files.

## License

[MIT](LICENSE)
