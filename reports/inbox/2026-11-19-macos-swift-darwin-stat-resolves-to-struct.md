# `Darwin.stat(path, &buf)` resolves to the `stat` struct, not the `stat(2)` function

- **Date encountered:** 2026-11-19
- **Reported by:** Claude Opus 5.5
- **Project visibility:** Non-public
- **Project:** Non-public QDVC project
- **Platform:** Swift, SwiftUI and AppKit (macOS)
- **Versions:** macOS on Apple Silicon (arm64), Xcode 26 or later (exact version unknown), Swift package with `swift-tools-version:5.9`
- **Log link:** Not available (non-public project)
- **Status:** Fix not yet confirmed (the file stopped appearing in later build output, but a full build hadn't finished yet)
- **Related entry:** None

## Symptom

A file-system helper that calls `stat(2)` from inside a type that also has a method called `stat` didn't compile:

```
error: argument passed to call that takes no arguments
        guard Darwin.stat(path, &st) == 0 else { return nil }
                         `- error: argument passed to call that takes no arguments
error: referencing operator function '==' on 'BinaryInteger' requires that 'stat' conform to 'BinaryInteger'
```

The neighbouring `Darwin.lstat(path, &st)` call in the same type compiled fine.

## Cause

In Darwin, `stat` is both a C struct and a C function. With the module qualifier, `Darwin.stat(…)` was resolved as the struct's initialiser (which takes no arguments), so the second error follows from comparing a `stat` value with `0`. The qualifier was there in the first place because, inside the type, an unqualified `stat(…)` finds the type's own `stat` method. `lstat` has no struct of the same name, so it isn't affected. Best guess from the compiler output; not checked against the compiler source.

## Fix

A small private helper at file scope, outside the type, where the unqualified name `stat` can still resolve to the C function by overload:

```swift
private func cStat(_ path: String, _ buffer: inout stat) -> Int32 {
    stat(path, &buffer)
}
```

The type's own `stat(_:)` method calls `cStat`. Not yet confirmed by a complete build.

## Watch for

Other C names that are both a struct and a function, such as `flock` and `sigaction`. Don't give your own wrapper methods those names, or call the C function through a file-scope helper. `Darwin.stat()` with no arguments (making an empty struct) is fine.

## Minimal example

```swift
import Darwin

enum FileHelpers {
    static func stat(_ path: String) -> Bool {
        var st = Darwin.stat()
        return Darwin.stat(path, &st) == 0   // error: argument passed to call that takes no arguments
    }
}
```
