> **Outcome (2026-11-19):** New entry "Views placed with `.offset` leave their container too narrow to hover or click" in Swift, SwiftUI and AppKit (macOS).

# Views placed with `.offset` leave their container (and its hover and click area) too narrow

- **Date encountered:** 2026-11-15
- **Reported by:** Claude Opus 5.5, with the project owner
- **Project visibility:** Non-public
- **Project:** Non-public QDVC project
- **Platform:** Swift, SwiftUI and AppKit (macOS)
- **Versions:** macOS 14 deployment target, Swift tools 5.10 (SwiftPM); macOS and Xcode versions not recorded
- **Log link:** Not available (non-public project)
- **Status:** Fix not yet confirmed
- **Related entry:** None

## Symptom

A custom horizontal bar made of segments of different widths (sections of one track, laid end to end) stopped responding part-way along. Hovering, clicking and dragging worked over the first stretch of the bar, and did nothing beyond a certain point, though the segments there were drawn correctly. Bars with a single segment worked all the way along. The person first suspected a problem with the data shown in the later segments, which turned out to be fine.

## Cause

The best explanation from reading the code, and it fits the symptom exactly: the dead zone started at the width of the longest segment. Each segment was a fixed-width view in a `ZStack(alignment: .leading)`, moved into place with `.offset(x:)`. `.offset` moves where a view is drawn, but not the frame layout gives it, so the `ZStack` was only as wide as its widest child. The `.onHover`, `.onContinuousHover` and drag gesture were attached to that `ZStack`, so they only covered that width, however far along the offset segments were drawn.

## Fix

Give the container an explicit frame across the full available width (`.frame(width: totalWidth, height: …, alignment: .leading)`) and a `.contentShape(Rectangle())` before the hover and gesture modifiers. Positioning the segments with layout (an `HStack` with computed widths, or leading padding) instead of `.offset` also works. Not yet confirmed on a Mac.

## Watch for

- Any custom slider, timeline, ruler or chart built by offsetting pieces in a `ZStack`: the container's size, and with it anything attached to it (`onHover`, `contentShape`, gestures, `.background`, `.overlay`), follows the pieces' un-offset frames.
- Thumbs and markers placed with `.offset` can be clipped or untouchable for the same reason.
- A symptom that starts "at a point along the bar" that matches the size of one of its parts points to layout, not data.

## Minimal example

```swift
struct SegmentedBar: View {
    let segments: [(start: CGFloat, width: CGFloat)]   // positions in points
    let totalWidth: CGFloat
    @State private var hover: CGFloat?

    var body: some View {
        ZStack(alignment: .leading) {
            ForEach(segments.indices, id: \.self) { i in
                Capsule().frame(width: segments[i].width, height: 4)
                    .offset(x: segments[i].start)        // drawn here, laid out at x = 0
            }
        }
        // Without these two lines the hover area is only as wide as the widest segment.
        .frame(width: totalWidth, height: 18, alignment: .leading)
        .contentShape(Rectangle())
        .onContinuousHover { phase in
            if case .active(let p) = phase { hover = p.x } else { hover = nil }
        }
    }
}
```
