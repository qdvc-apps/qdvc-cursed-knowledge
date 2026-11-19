# A local constant hides a method with the same base name: "cannot call value of non-function type"

- **Date encountered:** 2026-11-19
- **Reported by:** Claude Opus 5.5
- **Project visibility:** Non-public
- **Project:** Non-public QDVC project
- **Platform:** Swift, SwiftUI and AppKit (macOS)
- **Versions:** macOS on Apple Silicon (arm64), Xcode 26 or later (exact version unknown), Swift package with `swift-tools-version:5.9`
- **Log link:** Not available (non-public project)
- **Status:** Fix not yet confirmed (the error didn't come back in the next build output)
- **Related entry:** "Code it couldn't compile, run or see" (Mistakes the AI kept repeating)

## Symptom

```
error: cannot call value of non-function type 'Int'
        try archive([path])
            `- error: cannot call value of non-function type 'Int'
```

(Method and variable names replaced with placeholders.)

## Cause

Earlier in the same method, a local constant had the same name as an instance method: `let archive = flag ? 2 : 3`, and later `try archive([path])` was meant to call the method. Swift's name lookup finds the innermost declaration first, the local `Int`, and doesn't fall back to the method, even though the call syntax only makes sense for the method. Languages with separate namespaces for methods and variables (and code written by habit from them) don't have this problem. Confirmed by the compiler message and by renaming the constant.

The AI wrote this while porting code from another language, where the same shape was harmless.

## Fix

Rename the local (`let archiveIndex = …`), or call the method as `self.archive(…)`. Not yet confirmed by a complete build.

The same session also renamed two `guard let item = item(for: id)` bindings preemptively, where the bound name matched a method called in its own initialiser. It's not known whether those would have failed too.

## Watch for

- Local names that match a method of the enclosing type, especially short verbs and nouns (`save`, `load`, `count`, `items`).
- `let x = x(…)` and `guard let x = x(…)` patterns, which can also give "use of local variable before its declaration".
- A quick grep for locals that share a name with a method called later in the same function catches these before a build.

## Minimal example

```swift
final class Store {
    func archive(_ items: [String]) {}

    func finish(flag: Bool) {
        let archive = flag ? 2 : 3
        print(archive)
        archive(["a"])   // error: cannot call value of non-function type 'Int'
    }
}
```
