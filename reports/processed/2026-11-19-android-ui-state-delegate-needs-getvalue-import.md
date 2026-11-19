> **Outcome (2026-11-19):** New entry "`by rememberUpdatedState(…)` needs `import androidx.compose.runtime.getValue`" in Kotlin, Jetpack Compose and Android UI.

# `by rememberUpdatedState(...)` needs `import androidx.compose.runtime.getValue`

- **Date encountered:** 2026-11-19
- **Reported by:** Claude Opus 5.5, with the project owner
- **Project visibility:** Non-public
- **Project:** Non-public QDVC project
- **Platform:** Kotlin, Jetpack Compose and Android UI
- **Versions:** Android Gradle Plugin 9.3.3; Kotlin 2.4.20; Compose BOM 2026.08.00
- **Log link:** Not available (non-public project)
- **Status:** Confirmed fixed (the errors didn't come back on the next build)
- **Related entry:** None

## Symptom

```
e: <MyScanner>.kt:37:15 Type 'State<(String) -> Unit>' has no method 'getValue(Nothing?, KProperty0<*>)', so it cannot serve as a delegate.
e: <MyScanner>.kt:59:70 Unresolved reference '<found>'.
```

Line 37 was `val found by rememberUpdatedState(onCode)`. The second error is a knock-on: the delegated property never came into existence.

## Cause

Confirmed by the fix. Property delegation calls an operator function named `getValue` (and `setValue` for a `var`). For Compose's `State` and `MutableState`, those operators aren't members: they're extension functions in `androidx.compose.runtime` (and there are separate ones for `IntState`, `FloatState` and the like). Like any extension, they only apply when imported. Android Studio normally adds the import when you type `by`, so this mostly bites code written outside the IDE. The file had imports for `remember` and `rememberUpdatedState` but not `getValue`.

## Fix

`import androidx.compose.runtime.getValue` (plus `import androidx.compose.runtime.setValue` for any `var … by`). Confirmed: the next build got past both errors.

## Watch for

- The message names the missing operator (`getValue(Nothing?, KProperty0<*>)` for a local `val`), which looks like a type problem but is a missing import.
- Before building, check that every file with `by remember`, `by mutableStateOf`, `by produceState`, `by animate…AsState` or `by rememberUpdatedState` imports `getValue`, and `setValue` if it has a `var`. One import of `getValue` covers all the overloads.

## Minimal example

```kotlin
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue          // without this: "cannot serve as a delegate"
import androidx.compose.runtime.rememberUpdatedState

@Composable
fun <MyScanner>(onCode: (String) -> Unit) {
    val found by rememberUpdatedState(onCode)
    // …call found(text) from a long-lived callback…
}
```
