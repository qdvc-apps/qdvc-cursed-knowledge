> **Outcome (2026-11-20):** New entry "A `Unit` suspend call used as a value crashes with `Float cannot be cast to kotlin.Unit`" in Kotlin, Jetpack Compose and Android UI.

# `scrollState.animateScrollTo` inside a `when` expression crashes with `Float cannot be cast to kotlin.Unit`

- **Date encountered:** 2026-11 (month only)
- **Reported by:** Claude Opus 5.5
- **Project visibility:** Non-public
- **Project:** Non-public QDVC project
- **Platform:** Kotlin, Jetpack Compose and Android UI
- **Versions:** Kotlin 2.4.0, Compose BOM 2026.06.00, kotlinx-coroutines 1.10.2, AGP 9.3.3; Android 17 (API 37) on a Samsung Galaxy phone
- **Log link:** Not available (non-public project)
- **Status:** Fix not yet confirmed
- **Related entry:** None

## Symptom

Tapping the bottom-navigation tab you were already on should scroll that screen back to the top. On one tab this crashed the app at once. The other tabs worked. The crashing tab was a `Column` with `Modifier.verticalScroll(scrollState)`, and the working tabs were lazy grids. The code was shaped like this:

```kotlin
scope.launch {
    when (tab) {
        Tab.A -> gridStateA.animateScrollToItem(0)
        Tab.B -> gridStateB.animateScrollToItem(0)
        Tab.C -> scrollState.animateScrollTo(0)   // only this branch crashed
    }
}
```

```text
FATAL EXCEPTION: main
java.lang.ClassCastException: java.lang.Float cannot be cast to kotlin.Unit
	at <app frame> (the launch lambda's invokeSuspend)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:34)
	at kotlinx.coroutines.internal.ScopeCoroutine.afterResume(Scopes.kt:35)
	at kotlinx.coroutines.AbstractCoroutine.resumeWith(AbstractCoroutine.kt:101)
	at kotlin.coroutines.jvm.internal.BaseContinuationImpl.resumeWith(ContinuationImpl.kt:47)
	at kotlinx.coroutines.DispatchedTask.run(DispatchedTask.kt:100)
	at androidx.compose.ui.platform.AndroidUiDispatcher.performTrampolineDispatch(AndroidUiDispatcher.android.kt:79)
	at androidx.compose.ui.platform.AndroidUiDispatcher$dispatchCallback$1.run(AndroidUiDispatcher.android.kt:57)
	at android.os.Handler.handleCallback(Handler.java:1095)
	at android.os.Looper.loop(Looper.java:422)
	at android.app.ActivityThread.main(ActivityThread.java:10616)
	Suppressed: kotlinx.coroutines.internal.DiagnosticCoroutineContextException: [androidx.compose.ui.platform.MotionDurationScaleImpl@…, androidx.compose.runtime.BroadcastFrameClock@…, StandaloneCoroutine{Cancelling}@…, AndroidUiDispatcher@…]
```

## Cause

Best explanation, from the stack trace, the API reference and how Kotlin compiles suspend calls; not reproduced in isolation.

`ScrollState.animateScrollTo` is declared `suspend fun animateScrollTo(value: Int, animationSpec: AnimationSpec<Float> = SpringSpec()): Unit`. Its body, however, is understood to end with a call to `ScrollableState.animateScrollBy`, which returns a `Float` (the amount scrolled). The androidx source wasn't checked for this report. Kotlin compiles a suspend call in tail position by passing the caller's continuation straight on. So when a `Unit` function's last call is a non-`Unit` suspend function and it really suspends (an animation always does), its caller is resumed with that inner `Float`, not with `Unit`. kotlin-reflect's `callSuspend` has a source comment about exactly this: a `Unit` suspend function in tail-call position can appear to return something else.

Most callers never notice, because they ignore the result. Here every branch of the `when` was declared `Unit`, so the `when` was a `Unit` expression. As the last expression of the `launch` lambda, it was the lambda's result. The compiler therefore used the resumed value and cast it to `Unit`, and that cast failed for the `ScrollState` branch only. The lazy-grid branches resumed with a genuine `Unit`. The plain one-call form used in the official samples, `scope.launch { scrollState.animateScrollTo(0) }`, is not known to crash. Wrapping the call in a `when` or `if` expression seems to be what exposes the leaked value.

The first explanation in the session was that `animateScrollTo` returns a `Float` and the branches' types were combined. That's wrong in detail: the declared type is `Unit`. Had a branch really been typed `Float`, the `when` would have been `Any`, and Kotlin would have discarded the value instead of casting it.

## Fix

The scroll was moved into its own `suspend fun` that calls each scroll function as a statement, never as a value, and returns `Unit` explicitly:

```kotlin
suspend fun scrollToTop(tab: Tab) {
    if (tab == Tab.C) {
        scrollState.animateScrollTo(0)
        return
    }
    val grid = when (tab) { Tab.A -> gridStateA; else -> gridStateB }
    grid.animateScrollToItem(0)
}

scope.launch { scrollToTop(tab) }
```

The fix isn't confirmed yet: the session ended before the crash was tested again. A smaller fix that should also work is to end the lambda with an explicit `Unit` after the `when`, so the `when`'s value is discarded.

## Watch for

- Any `Unit` suspend call whose result is used as a value: as the last expression of a lambda inside an `if` or `when`, in `val x = …`, after `?:`, or through a generic `T` (`async { … }.await()`, `runCatching { … }`, `withContext(…) { … }` returning its value).
- It only crashes when the call actually suspends, so instant paths and tests with an immediate dispatcher can pass.
- Other library functions with the same shape (a `Unit` declaration whose body ends in a value-returning suspend call) can do the same. If you see `X cannot be cast to kotlin.Unit` in `invokeSuspend`, look for a `Unit` suspend call used as an expression, and turn it into a statement.

## Minimal example

```kotlin
@Composable
fun BackToTop(useList: Boolean) {
    val scope = rememberCoroutineScope()
    val scrollState = rememberScrollState()
    val listState = rememberLazyListState()
    Button(onClick = {
        scope.launch {
            when (useList) {
                true -> listState.animateScrollToItem(0)
                false -> scrollState.animateScrollTo(0)   // ClassCastException: Float cannot be cast to kotlin.Unit
            }
        }
    }) { Text("Top") }
}
```
