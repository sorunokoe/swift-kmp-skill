# Contributing to swift-kmp

Thank you for contributing! This skill is a community resource — your real-world patterns,
corrections, and additions make it better for everyone building KMP + Swift apps.

---

## Ways to Contribute

- **Fix a bug** — a pattern that's incorrect or produces broken code
- **Improve a pattern** — add a missing edge case, better example, or official doc quote
- **Add a reference file** — a new deep-dive covering a pattern not yet documented
- **Update for Swift Export** — as Swift Export stabilizes, patterns can be migrated from SKIE-era

---

## Principles

1. **Grounded in official docs** — cite [kotlinlang.org](https://kotlinlang.org/docs/native-objc-interop.html), [skie.touchlab.co](https://skie.touchlab.co/features), or Apple docs. If undocumented, label it clearly.
2. **Battle-tested** — include only patterns that work in production KMP projects.
3. **Explicit about tradeoffs** — show both ✅ correct pattern and ❌ anti-pattern with a *why*.
4. **AI-agent-friendly** — skills are read under token budget. Every sentence earns its place. No repetition.
5. **Version-aware** — label SKIE-era vs Swift Export-era patterns. Don't erase migration notes.

---

## How to Improve a Pattern

1. Fork and clone the repo
2. Edit the relevant `.md` file in `references/`
3. Verify code examples compile (paste into Xcode or a KMP project)
4. Open a PR titled: `[topic] Brief description` (e.g., `[flow-bridging] Add ThrowingStream example`)
5. Include the official doc link you're citing in the PR description

---

## How to Add a New Reference File

Add a reference file when a topic needs 100–300 lines of focused deep-dive content.

**Template:**
```markdown
# <Topic Name> — swift-kmp

> **Official reference**: <URL>

---

## Pattern: <Name>

```swift
// ✅ Correct:
<example>

// ❌ Anti-pattern:
<wrong example>
```

<explanation>
```

After adding the file, update the **Reference Router** table in `SKILL.md`.

---

## Code Example Guidelines

Use generic placeholder names — never project-specific class names:

| Concept | Use this |
|---------|----------|
| KMP iOS framework | `YourKmpFramework` |
| App DI component | `AppComponent` |
| Feature module | `<Feature>Feature` |
| Bundle ID | `com.example.app` |

---

## Companion Skill

This skill pairs with [**swiftui-compose**](https://github.com/sorunokoe/swiftui-compose-skill).
If your contribution touches `UIViewControllerRepresentable` or Compose embedding, consider
whether it belongs there instead (or in both).
