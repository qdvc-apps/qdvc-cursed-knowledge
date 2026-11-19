# `Timer.publish(…).autoconnect()` in a SwiftUI view warns "'Combine' was not imported by this file"

- **Date encountered:** 2026-11-19
- **Reported by:** Claude Opus 5.5
- **Project visibility:** Non-public
- **Project:** Non-public QDVC project
- **Platform:** Swift, SwiftUI and AppKit (macOS)
- **Versions:** macOS on Apple Silicon (arm64), Xcode 26 or later (exact version unknown; the SDK's SwiftUI interface uses `~Sendable`), Swift package with `swift-tools-version:5.9`
- **Log link:** Not available (non-public project)
- **Status:** Fix not yet confirmed
- **Related entry:** None

## Symptom

A SwiftUI view that imported only `SwiftUI` and `AppKit` and stored a timer publisher as a property:

```
warning: cannot use generic class 'Autoconnect' in a property declaration member of a type not marked '@_implementationOnly'; 'Combine' was not imported by this file
    let ticker = Timer.publish(every: 1, on: .main, in: .common).autoconnect()
warning: cannot use enum 'Publishers' in a property declaration member of a type not marked '@_implementationOnly'; 'Combine' was not imported by this file
```

The same code had compiled without warnings in older SwiftUI examples, which is why it was written without `import Combine`.

## Cause

The property's inferred type, `Publishers.Autoconnect<Timer.TimerPublisher>`, comes from Combine. The file could see Combine's types only indirectly, through another import, and the compiler warns when such a type appears in a declaration's signature. Best guess: newer SDKs no longer make Combine's types fully visible through `import SwiftUI`. Not checked against the SDK release notes.

## Fix

Add `import Combine` to every file that uses Combine types in declarations. That includes `ObservableObject` and `@Published` in model classes that import only `Foundation`, which hadn't warned yet in this build. Not yet confirmed.

## Watch for

Any declaration whose type comes from a framework the file doesn't import directly: `ObservableObject` models, `AnyCancellable` properties, `onReceive` publishers stored as properties. If you lint imports against an allowlist, add `Combine` to it.

## Minimal example

```swift
import SwiftUI

struct ClockView: View {
    let ticker = Timer.publish(every: 1, on: .main, in: .common).autoconnect()   // warning: 'Combine' was not imported by this file
    @State private var ticks = 0

    var body: some View {
        Text("\(ticks)").onReceive(ticker) { _ in ticks += 1 }
    }
}
```
