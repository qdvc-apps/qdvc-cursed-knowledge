# Targeting Android 17, connections to the local network just time out without `ACCESS_LOCAL_NETWORK`

- **Date encountered:** 2026-11-19
- **Reported by:** Claude Opus 5.5, with the project owner
- **Project visibility:** Non-public
- **Project:** Non-public QDVC project
- **Platform:** New section suggested: "Android platform: permissions, networking and background work" (none of the existing Android sections fits)
- **Versions:** Android 17 (API 37), for apps whose targetSdk is 37 or higher
- **Log link:** Not available (non-public project)
- **Status:** Fix not yet confirmed (not yet run on an Android 17 device)
- **Related entry:** None

## Symptom

Found while raising an app's targetSdk to 37, before it ran on a device. The app talks to a server on the home network by IP address or `.local` name. According to the documentation, without the new permission its TCP connections to that server would simply time out, which the app would take to mean "the server isn't reachable", with no error naming a permission.

## Cause

From the Android documentation ("Local network permission", developer.android.com/privacy-and-security/local-network-permission), not yet measured:

- For apps targeting Android 17 or higher, all local network traffic is blocked unless the user grants `ACCESS_LOCAL_NETWORK`. Apps targeting 36 or lower get it implicitly with `INTERNET`, but only until they raise targetSdk to 37.
- It's a runtime permission in the `NEARBY_DEVICES` group. If the user already granted another permission in that group (Bluetooth, say), they aren't asked again.
- When it's blocked, TCP typically gives a timeout and UDP an `EPERM`. It applies to every networking API (sockets, `HttpURLConnection`, OkHttp, Cronet, `NsdManager`), and resolving `.local` names needs it too. Internet and mobile-network traffic aren't affected, and neither is DNS to a local DNS server on port 53.
- On Android 16 the same restriction could be tried with `adb shell am compat enable RESTRICT_LOCAL_NETWORK <package>`, and was then lifted with `NEARBY_WIFI_DEVICES`.

## Fix

Not yet confirmed on a device:

- Declare `<uses-permission android:name="android.permission.ACCESS_LOCAL_NETWORK" />`.
- Ask for it (`ActivityResultContracts.RequestPermission()`) before the first connection, with a sentence saying why.
- Before trying to connect, check `checkSelfPermission(…) == GRANTED`, only when `Build.VERSION.SDK_INT >= 37` (on earlier versions the permission doesn't exist, so the check would always report it denied). Show a "not allowed" state rather than treating the timeout as "unreachable".
- If it was turned down and the system won't ask again, offer the app's settings page (`Settings.ACTION_APPLICATION_DETAILS_SETTINGS`). Check again in `onResume`, since it can be changed there.

## Watch for

- Background work (jobs, foreground services) needs it too: the grant belongs to the app, not to an activity.
- Any app that talks to a LAN server, a NAS, a printer, a speaker or a local web UI is affected the day it targets 37, even if nothing in its code changed.
- A timeout is the only visible symptom, so the app's "away / offline" logic will happily misreport it.

## Minimal example

```kotlin
object LocalNetwork {
    const val PERMISSION = "android.permission.ACCESS_LOCAL_NETWORK"
    val required get() = Build.VERSION.SDK_INT >= 37
    fun granted(context: Context) =
        !required || context.checkSelfPermission(PERMISSION) == PackageManager.PERMISSION_GRANTED
}

// In a composable:
val ask = rememberLauncherForActivityResult(ActivityResultContracts.RequestPermission()) { /* re-check */ }
if (!LocalNetwork.granted(context)) Button(onClick = { ask.launch(LocalNetwork.PERMISSION) }) { Text("Allow access to the local network") }
```
