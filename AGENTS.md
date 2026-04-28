# AGENTS.md — swift-kmp

This file is for AI coding agents reading or contributing to this repository.

---

## What This Repository Is

A structured AI coding skill for developers building
[Kotlin Multiplatform (KMP)](https://kotlinlang.org/docs/multiplatform-intro.html) apps
with Swift/iOS native UI.

**The core rule**: Feature modules must never import the KMP framework (regardless of integration method).
They receive `@Sendable` closures with pure Swift types. Every KMP import lives
in one place — the bridge layer.

---

## Repository Structure

```
SKILL.md                   ← ALWAYS load first — hard rules, cheatsheet, review checklist
references/
├── architecture.md        ← 4-layer bridge model, iOS integration methods, module boundary rule
├── interactor-pattern.md  ← Interactor template, @concurrent nonisolated, error handling
├── flow-bridging.md       ← SkieSwiftFlow vs AsyncStream, leak-safe onTermination
└── type-mapping.md        ← Kotlin sealed dispatch, KotlinThrowable, collections, Sendable
```

> 🔗 **Compose Multiplatform ↔ SwiftUI** (`UIViewControllerRepresentable`, coordinator pattern, teardown)
> is covered by the companion [**swiftui-compose**](https://github.com/sorunokoe/swiftui-compose-skill) skill.

---

## How to Apply This Skill

1. Load `SKILL.md` — this contains the hard rules and a review checklist
2. Check the **Reference Router** table in `SKILL.md` to identify which reference is needed
3. Load that one reference file only (loading all wastes tokens unnecessarily)
4. Apply the patterns, then run the **Review Checklist**

### Skill loading order

```
SKILL.md  →  references/architecture.md (if module boundary ownership is unclear)
          →  references/<specific-topic>.md (for this task's API contract)
```

---

## Key Invariants

When modifying any file in this repository:

1. **No project-specific names** — use `YourKmpFramework`, `AppComponent`, generic feature names
2. **All code examples must be compilable** — no pseudo-code without clear labeling
3. **Official doc links must be live** — verify any URL you add
4. **Hard rules stay hard** — never soften a ❌ rule without strong justification
5. **Transitional patterns stay labeled** — `onEnum(of:)` and `@preconcurrency import` are explicitly SKIE-era; keep migration notes

---

## Companion Skill

This skill is paired with [**swiftui-compose**](https://github.com/sorunokoe/swiftui-compose-skill),
which covers `UIViewControllerRepresentable` wiring and bidirectional state patterns.

When a task involves embedding Compose views in SwiftUI, refer the user to that skill for
the SwiftUI lifecycle patterns, and use this skill for the KMP bridge layer patterns.

---

## Out of Scope

- General Swift concurrency (actors, `Sendable`, `MainActor`)
- TCA / SwiftUI architecture patterns
- Android-specific Compose patterns
- KMP Gradle build configuration
- SPM package structure
