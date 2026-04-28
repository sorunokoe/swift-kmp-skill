# Changelog

All notable changes to this skill are documented here.

---

## [1.0.0] — 2026-04-28

### Initial production release

**Core skill files**
- `SKILL.md` — hard rules, review checklist, quick pattern reference, reference router
- `references/architecture.md` — 4-layer bridge model with all 5 iOS framework integration methods
- `references/interactor-pattern.md` — full interactor template with `@concurrent nonisolated`, error handling, cancellation guard, Interactor vs Service comparison
- `references/flow-bridging.md` — `SkieSwiftFlow` vs `AsyncStream` decision guide with leak-safe teardown
- `references/type-mapping.md` — Kotlin sealed dispatch, `KotlinThrowable` containment, collection mapping, `Sendable` contracts
- `AGENTS.md` — guidance for AI agents contributing to or applying this skill
- `CONTRIBUTING.md` — contribution principles, PR workflow, code example guidelines

**Architecture**
- Documents all 3 recommended KMP iOS integration methods (direct, SwiftPM local, SwiftPM remote XCFramework) — bridge patterns are identical in all cases
- Clear module boundary rule: KMP imports are confined to the bridge layer regardless of integration method

**Correctness (fact-checked against official documentation)**
- `@concurrent` annotated as Swift 6.2+ with official Swift Blog reference
- `continuation.onTermination` closure correctly annotated `@Sendable` per Apple's canonical `AsyncStream` example
- Swift Export sealed class claim scoped correctly — only `enum class → Swift enum` is documented; sealed class/interface bridging not yet in Swift Export's feature set
- `@preconcurrency import` explanation includes standard Swift 6.x vs default-isolation-MainActor nuance

**AI agent ergonomics**
- Token budget measured at ~1.7k for `SKILL.md` (covers most tasks without loading any reference)
- Reference Router routes to one file per task; "Complete most tasks here" framing prevents unnecessary context loading
- Compose Multiplatform ↔ SwiftUI embedding delegated to companion [`swiftui-compose`](https://github.com/sorunokoe/swiftui-compose-skill) skill to keep this skill focused

**Compatibility badges**
- Works with: Claude, GitHub Copilot, Codex, Cursor, Gemini
