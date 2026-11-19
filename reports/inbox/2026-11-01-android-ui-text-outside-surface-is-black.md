# Compose `Text` outside a `Surface` or `Scaffold` is black, even in dark mode

- **Date encountered:** 2026-11 (month only)
- **Reported by:** Claude Opus 5.5
- **Project visibility:** Non-public
- **Project:** Non-public QDVC project
- **Platform:** Kotlin, Jetpack Compose and Android UI
- **Versions:** Compose BOM 2026.06.00 (Material 3), Kotlin 2.4.0; Android 17 (API 37) on a Samsung Galaxy phone, system dark mode on
- **Log link:** Not available (non-public project)
- **Status:** Fix not yet confirmed (not reported again in later rounds)
- **Related entry:** None

## Symptom

In dark mode, some lines of a details panel were drawn in black on the dark background and were almost invisible. Others in the same panel looked right. The panel belonged to a full-screen overlay. The black lines were plain `Text(...)` calls with no colour; the readable ones had an explicit colour or a style with one. The same screens looked correct in light mode, where black text on a light background is what you'd expect anyway.

## Cause

Material 3's `Text` takes its colour from the `color` argument, then from the style's colour, then from `LocalContentColor`. Outside any provider, `LocalContentColor` defaults to `Color.Black`. `Surface` provides it (as `contentColorFor` its colour), and `Scaffold`, `TopAppBar`, `NavigationBar` and cards all draw a `Surface`, so text inside them follows the theme. `MaterialTheme` itself doesn't set `LocalContentColor`.

The overlay was a plain `Box` drawn above the app's `Scaffold`, so it could cover the bottom bar. It was therefore outside every `Surface` and got the black default. This is from the Material 3 API and fits the symptom exactly; it wasn't measured.

## Fix

The panel now wraps its content in `CompositionLocalProvider(LocalContentColor provides MaterialTheme.colorScheme.onSurface)`. The same provider was also added once at the root of the app, around everything, so any other layer drawn outside a `Surface` inherits a theme colour instead of black. The user didn't report the problem again, but didn't explicitly confirm the fix either.

## Watch for

- Full-screen viewers, overlays and layers in `AnimatedVisibility` placed over or beside a `Scaffold`, and content built from plain `Box`es and `Column`s with no `Surface` behind them.
- `Icon` uses `LocalContentColor` for its tint too, so untinted icons in the same places are black as well.
- Light mode hides the bug completely, so look at every screen in dark mode.
- A provider near the root is a cheap safety net. Inside a `Surface`, the `Surface`'s own value still takes over.

## Minimal example

```kotlin
@Composable
fun App() {
    MaterialTheme(colorScheme = darkColorScheme()) {
        Box(Modifier.fillMaxSize().background(MaterialTheme.colorScheme.surface)) {
            Text("Black on dark grey")           // LocalContentColor's default: Color.Black
        }
        Surface {
            Text("Follows the theme")            // Surface provides onSurface
        }
    }
}

// Fix: provide a theme colour around content that isn't inside a Surface.
CompositionLocalProvider(LocalContentColor provides MaterialTheme.colorScheme.onSurface) {
    Text("Follows the theme")
}
```
