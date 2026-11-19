# An SDK constant that's an `Int` turns an `if` into `Number & Comparable<*>`

- **Date encountered:** 2026-11-19
- **Reported by:** Claude Opus 5.5, with the project owner
- **Project visibility:** Non-public
- **Project:** Non-public QDVC project
- **Platform:** Kotlin, Jetpack Compose and Android UI
- **Versions:** Android Studio (version not recorded); Android Gradle Plugin 9.3.3 with its built-in Kotlin; Kotlin 2.4.20; compileSdk 37
- **Log link:** Not available (non-public project)
- **Status:** Confirmed fixed (the error didn't come back on the next build)
- **Related entry:** None

## Symptom

```
e: <MyJobService>.kt:78:43 Argument type mismatch: actual type is 'Number & Comparable<CapturedType(*)>', but 'Long' was expected.
```

The line was a call to `JobInfo.Builder.setEstimatedNetworkBytes`, which takes two `long`s.

## Cause

Confirmed by the error and the fix. `JobInfo.NETWORK_BYTES_UNKNOWN` is declared as a Java `int`, even though every method that accepts it takes a `long`. Kotlin never widens `Int` to `Long` implicitly, and the two branches of `if (bytes > 0) bytes else JobInfo.NETWORK_BYTES_UNKNOWN` (a `Long` and an `Int`) unify to their common supertype, `Number & Comparable<*>`. That's the strange type in the message. The literal `0` in the second argument was fine, because integer literals take on the expected type; a named constant doesn't.

## Fix

`JobInfo.NETWORK_BYTES_UNKNOWN.toLong()` (and `0L` for clarity). Confirmed: the next build got past it.

## Watch for

- Any Java or Android constant declared `int` but meant for a `long` parameter. It fails on its own too, not just inside an `if`.
- Other Java libraries do the same with their "unset" or "end" sentinels: check a constant's declared type before mixing it with `Long` values.
- If you see `Number & Comparable<…>` in an error, look for an `if`, `when` or `?:` whose branches mix `Int` and `Long`.

## Minimal example

```kotlin
val bytes: Long = estimate()
JobInfo.Builder(1, component)
    // Error: actual type is 'Number & Comparable<CapturedType(*)>', but 'Long' was expected.
    .setEstimatedNetworkBytes(if (bytes > 0) bytes else JobInfo.NETWORK_BYTES_UNKNOWN, 0)

JobInfo.Builder(1, component)
    // Compiles.
    .setEstimatedNetworkBytes(if (bytes > 0) bytes else JobInfo.NETWORK_BYTES_UNKNOWN.toLong(), 0L)
```
