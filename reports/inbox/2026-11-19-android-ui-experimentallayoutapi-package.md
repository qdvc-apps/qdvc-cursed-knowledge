# `ExperimentalLayoutApi` isn't in the same package as `ExperimentalFoundationApi`

- **Date encountered:** 2026-11-19
- **Reported by:** Claude Opus 5.5, with the project owner
- **Project visibility:** Non-public
- **Project:** Non-public QDVC project
- **Platform:** Kotlin, Jetpack Compose and Android UI
- **Versions:** Android Gradle Plugin 9.3.3; Kotlin 2.4.20; Compose BOM 2026.08.00
- **Log link:** Not available (non-public project)
- **Status:** Confirmed fixed (the errors didn't come back on the next build)
- **Related entry:** The keyboard leaves a huge gap above it, or covers the input

## Symptom

Four errors from one wrong import line:

```
e: <MyScreen>.kt:19:36 Unresolved reference 'ExperimentalLayoutApi'.
e: <MyScreen>.kt:146:8 Unresolved reference 'ExperimentalLayoutApi'.
e: <MyScreen>.kt:146:8 Annotation argument must be a compile-time constant.
e: <MyScreen>.kt:153:28 The API of this layout is experimental and is likely to change in the future.
```

Line 153 was `WindowInsets.isImeVisible`, which the existing entry's fix uses to hide the bottom bar while the keyboard is up.

## Cause

Confirmed by the fix. The import was `androidx.compose.foundation.ExperimentalLayoutApi`, by analogy with `androidx.compose.foundation.ExperimentalFoundationApi`. But `ExperimentalLayoutApi` is declared one package deeper, in `androidx.compose.foundation.layout`, beside the layout APIs it marks. The other three errors all follow from the first: with the annotation class unresolved, the `@OptIn` is invalid, so the experimental-API check fires on its own.

## Fix

`import androidx.compose.foundation.layout.ExperimentalLayoutApi`. Confirmed: the next build got past all four errors.

## Watch for

- Compose's opt-in markers live beside the APIs they mark: `androidx.compose.foundation.ExperimentalFoundationApi`, `androidx.compose.foundation.layout.ExperimentalLayoutApi`, `androidx.compose.material3.ExperimentalMaterial3Api`.
- If `@OptIn` produces "Annotation argument must be a compile-time constant", the marker class itself didn't resolve. Fix the import first and the rest goes away.
- The existing keyboard entry's fix needs this opt-in, which it doesn't mention.

## Minimal example

```kotlin
// Wrong: Unresolved reference 'ExperimentalLayoutApi'
// import androidx.compose.foundation.ExperimentalLayoutApi
import androidx.compose.foundation.layout.ExperimentalLayoutApi
import androidx.compose.foundation.layout.WindowInsets
import androidx.compose.foundation.layout.isImeVisible

@OptIn(ExperimentalLayoutApi::class)
@Composable
fun <MyScreen>() {
    val keyboardUp = WindowInsets.isImeVisible
    // …
}
```
