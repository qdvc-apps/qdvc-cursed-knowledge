# SwiftUI: "unable to type-check this expression in reasonable time" from literal arithmetic inside a modifier chain

- **Date encountered:** 2026-11 (month only)
- **Reported by:** Claude Opus 5.5
- **Project visibility:** Non-public
- **Project:** Non-public QDVC project
- **Platform:** Swift, SwiftUI and UIKit (iOS and iPadOS)
- **Versions:** iOS 26 deployment target (iPhone and iPad); Xcode version not recorded
- **Log link:** Not available (non-public project)
- **Status:** Confirmed fixed (the next build reported no further errors in that file)
- **Related entry:** "`private(set)` doesn't reach an extension in another file" (its Cause mentions a type-checker timeout from a long closure chain, but not this error text or this shape of expression)

## Symptom

Building a SwiftUI view failed on a short chain of modifiers in a view body:

```swift
if let current {
    overlay(for: current)
        .opacity(Double(max(0, (1 - progress * 2) * (1 - fraction * 4))))
        .allowsHitTesting(isVisible && progress == 0 && offset == 0)
        .opacity(isVisible ? 1 : 0)
```

with:

`The compiler is unable to type-check this expression in reasonable time; try breaking up the expression into distinct sub-expressions`

The error is reported against the whole statement, not the part that causes it. `progress`, `fraction` and `offset` were `CGFloat` state values driven by a drag gesture.

## Cause

Best explanation, not measured. A chain of modifiers is a single expression, so the type checker has to solve all of it at once. Inside it, every untyped literal (`0`, `1`, `2`, `4`) could be an `Int`, `Double`, `CGFloat` or another numeric type. `max` is generic, `Double(...)` has many overloaded initialisers, and Swift converts implicitly between `CGFloat` and `Double`. Together these multiply the combinations the solver tries. The two `== 0` comparisons and the `isVisible ? 1 : 0` add more literal overloads to the same expression.

## Fix

Each value was worked out first, as a separate constant with an explicit type, and the view then used the constants. The two `.opacity` modifiers were merged into one at the same time:

```swift
let fade: CGFloat = max(0, (1 - progress * 2) * (1 - fraction * 4))
let overlayOpacity: Double = isVisible ? Double(fade) : 0
let interactive: Bool = isVisible && progress == 0 && offset == 0

overlay(for: current)
    .opacity(overlayOpacity)
    .allowsHitTesting(interactive)
```

(Plain `let` declarations are allowed inside a `ViewBuilder`, including inside a `GeometryReader` closure.) The view looks and behaves exactly as before. The next build got past the error. In the same pass, three other expressions of the same shape in other files were split up before they could fail: a drag-progress calculation, an aspect-ratio comparison and a string built from a count.

## Watch for

- Gesture-driven views are the usual place for this: arithmetic with literals on `CGFloat` state, written directly inside `.opacity`, `.offset`, `.scaleEffect`, `.frame` or `.rotationEffect`.
- Wrapping a mixed expression in `Double(...)` or `CGFloat(...)` makes it worse. Give the intermediate value its type instead.
- When the error appears once, search the other view files for the same shape. Code written without a compiler tends to repeat it, and the compiler stops at the first file that times out.

## Minimal example

```swift
struct FadingOverlay: View {
    @State private var progress: CGFloat = 0
    @State private var fraction: CGFloat = 0
    @State private var offset: CGFloat = 0
    var isVisible = true

    var body: some View {
        Color.red
            // May fail with "unable to type-check this expression in reasonable time":
            .opacity(Double(max(0, (1 - progress * 2) * (1 - fraction * 4))))
            .allowsHitTesting(isVisible && progress == 0 && offset == 0)
            .opacity(isVisible ? 1 : 0)
    }
}
```
