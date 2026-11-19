# A `var` with a `private set` still clashes with your own `setX()` function

- **Date encountered:** 2026-11-19
- **Reported by:** Claude Opus 5.5, with the project owner
- **Project visibility:** Non-public
- **Project:** Non-public QDVC project
- **Platform:** Kotlin, Jetpack Compose and Android UI
- **Versions:** Android Studio (version not recorded); Android Gradle Plugin 9.3.3 with its built-in Kotlin; Kotlin 2.4.20 (the Compose compiler plugin's version); Gradle 9.5.0; compileSdk 37; Compose BOM 2026.08.00
- **Log link:** Not available (non-public project)
- **Status:** Fix not yet confirmed
- **Related entry:** None

## Symptom

`compileDebugKotlin` failed with three pairs of errors like this, one pair per property, and Gradle summarised them as `Kotlin compiler: CONFLICTING_JVM_DECLARATIONS`:

```
e: <AppModel>.kt:78:9 Platform declaration clash: The following declarations have the same JVM signature (setTheme(Ljava/lang/String;)V):
    fun `<set-theme>`(`<set-?>`: String): Unit defined in <package>.<AppModel>
    fun setTheme(value: String): Unit defined in <package>.<AppModel>
```

The other two were for two `Float` properties in another class, each with its own `setX(value: Float)` function.

## Cause

Confirmed by the compiler's own message. Every Kotlin `var` compiles to a JVM getter and setter, named `getX`/`setX`. Making the setter `private` only changes its visibility: the JVM method `setX(T)` still exists, and the JVM doesn't let two methods in one class share a name and parameter types, whatever their visibility. So a public `fun setX(value: T)` beside `var x: T private set` can't compile. It doesn't matter whether the property is delegated (`by mutableStateOf(...)`) or not.

It was easy to write because Swift allows it. `private(set) var scale` alongside `func setScale(_:)` is a common Swift pattern, and the code was ported from a Swift app in that style.

## Fix

The three functions were renamed (`setScale` became `changeScale`, and so on). The class still exposes the property read-only and changes it, with side effects, through a function. The renaming fixed the declarations the compiler named; whether the build then goes further hadn't been confirmed when this was written.

Alternatives: drop the `private set` and do the side effects elsewhere, or rename the setter's JVM name with `@set:JvmName("…")`.

## Watch for

- The same clash happens between `val x` and your own `fun getX()`, and between `val isX` and `fun isX()`.
- It's most likely when porting from Swift, or when a class exposes state read-only and changes it through `setX` functions.
- Before committing, search for `fun set[A-Z]` and `fun get[A-Z]` and check each against the class's properties. The compiler only reports it at the JVM-signature stage.

## Minimal example

```kotlin
class <Model> {
    var scale by mutableFloatStateOf(1f)
        private set

    // Platform declaration clash: same JVM signature as the property's (private) setter.
    fun setScale(value: Float) {
        scale = value
        // …apply it somewhere…
    }
}

// Compiles:
class <ModelFixed> {
    var scale by mutableFloatStateOf(1f)
        private set

    fun changeScale(value: Float) {
        scale = value
    }
}
```
