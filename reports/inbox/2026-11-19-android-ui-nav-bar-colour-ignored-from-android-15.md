# From Android 15, matching the system navigation bar means turning its scrim off, not setting its colour

- **Date encountered:** 2026-11-19
- **Reported by:** Claude Opus 5.5, with the project owner
- **Project visibility:** Non-public
- **Project:** Non-public QDVC project
- **Platform:** Kotlin, Jetpack Compose and Android UI
- **Versions:** targetSdk and compileSdk 37 (Android 17), minSdk 34; androidx.activity 1.10.1 in the app, and the androidx `main` source as of November 2026 for the reading below; Compose BOM 2026.08.00
- **Log link:** Not available (non-public project)
- **Status:** Fix not yet confirmed (the app hadn't run on a device yet)
- **Related entry:** System bars stay black on Samsung, and the nav bar colour doesn't quite match

## Symptom

Found while carrying an earlier app's approach over to a new app, before anything ran on a device. The earlier app targets SDK 34 and makes its bottom `NavigationBar` and the system's navigation buttons read as one colour by setting `window.navigationBarColor` (and `statusBarColor`) to the theme's `surface`. That is also what the existing entry's fix ("colour-match against the bottom bar's actual container colour") amounts to. For an app targeting SDK 35 or later, that approach no longer applies.

## Cause

From the platform documentation and the androidx source (`activity/activity/src/main/java/androidx/activity/EdgeToEdge.kt`); not yet measured on a device.

- Once an app targets SDK 35 or later, Android 15 and later force it edge to edge. The status bar and the gesture navigation bar are transparent, but the three-button navigation bar is translucent: Android draws its own scrim behind the buttons unless `Window.setNavigationBarContrastEnforced(false)`. `Window.setNavigationBarColor` and `setStatusBarColor` are deprecated, and Play flags them.
- `enableEdgeToEdge()`'s default `navigationBarStyle` is `SystemBarStyle.auto(DefaultLightScrim, DefaultDarkScrim)`. In the source, `isNavigationBarContrastEnforced` is set to `navigationBarStyle.nightMode == MODE_NIGHT_AUTO`, so the default `auto` style leaves the system's scrim on. Only an explicit `SystemBarStyle.light(…)` or `SystemBarStyle.dark(…)` turns it off.
- On API 35 and later, the current androidx source no longer sets the window's bar colours to the scrim (it sets them transparent). It adds its own `ProtectionLayout` view, coloured with the scrim, when the scrim isn't transparent. So a non-transparent scrim colour is an extra view drawn at the screen's edges, not the old window colour.

## Fix

Not yet confirmed on a device:

- Call `enableEdgeToEdge` with explicit, fully transparent styles chosen from the app's resolved theme: `SystemBarStyle.dark(Color.TRANSPARENT)`, or `SystemBarStyle.light(Color.TRANSPARENT, Color.TRANSPARENT)`. That turns contrast enforcement off and keeps androidx from adding protection views. Set `window.isNavigationBarContrastEnforced = false` explicitly as well, for clarity.
- Let the app's own bottom `NavigationBar` supply the colour: `containerColor = MaterialTheme.colorScheme.surface`, `tonalElevation = 0.dp` (the existing entry's tint point), and its default `windowInsets`, so it reaches down behind the buttons and pads its icons above them.
- Call it again whenever the theme changes, and after any full-screen view that hides the bars, so the icons' light or dark appearance follows.

## Watch for

- With gesture navigation there's no scrim either way, so testing only in gesture mode hides the problem. Test with three-button navigation.
- Screens without the bottom bar (settings, full-screen views) need their own background behind the transparent navigation bar.
- `enableEdgeToEdge()` with no arguments is not the same as "transparent": its navigation bar style is `auto`, which keeps the scrim.

## Minimal example

```kotlin
fun blendSystemBars(activity: ComponentActivity, dark: Boolean) {
    // Explicit light/dark (not auto) is what turns contrast enforcement off.
    val style = if (dark) SystemBarStyle.dark(Color.TRANSPARENT)
                else SystemBarStyle.light(Color.TRANSPARENT, Color.TRANSPARENT)
    activity.enableEdgeToEdge(statusBarStyle = style, navigationBarStyle = style)
    activity.window.isNavigationBarContrastEnforced = false
}

@Composable
fun <BottomBar>() {
    NavigationBar(containerColor = MaterialTheme.colorScheme.surface, tonalElevation = 0.dp) {
        // items…
    }
}
```
