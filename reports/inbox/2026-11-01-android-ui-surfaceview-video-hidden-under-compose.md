# A Media3 `PlayerView` in Compose plays sound but no picture when other content is drawn over it

- **Date encountered:** 2026-11 (month only)
- **Reported by:** Claude Opus 5.5
- **Project visibility:** Non-public
- **Project:** Non-public QDVC project
- **Platform:** Kotlin, Jetpack Compose and Android UI
- **Versions:** Media3 (ExoPlayer and UI) 1.6.1, Compose BOM 2026.06.00, Kotlin 2.4.0; Android 17 (API 37) on a Samsung Galaxy phone
- **Log link:** Not available (non-public project)
- **Status:** Fix not yet confirmed
- **Related entry:** None

## Symptom

A short video was meant to play in place of a still image (its poster frame) while the user held a finger on it. Holding played the video's sound, but the picture never appeared, and the still image stayed on screen. The video was a Media3 `PlayerView` hosted in Compose through `AndroidView`. It sat in the same `Box` as the still image, inside a zoomable area that uses `graphicsLayer` for pinch and pan.

## Cause

Best explanation; not proven, because the fix changed two things at once.

`PlayerView` renders into a `SurfaceView` by default. A `SurfaceView`'s surface is a separate layer, composited behind the app's window, and the view punches a transparent hole in the window so it shows through. Anything the window draws over that area, here the still image, hides the video. The `SurfaceView` also doesn't follow Compose's draw order, alpha, clipping or `graphicsLayer` scale and translation the way ordinary content does. Audio without a picture is what you'd expect: the player was running normally, but its frames were going to a surface nobody could see.

The fix both switched to a `TextureView` and moved the video in front of the still in the composition. Draw order alone might have been enough, so which change mattered isn't certain.

## Fix

The video is now rendered into a `TextureView`, which draws as part of the window like any other content. It's placed after (in front of) the still image, and sized to the video:

```kotlin
var aspect by remember { mutableStateOf<Float?>(null) }
// Player.Listener.onVideoSizeChanged sets aspect = width * pixelWidthHeightRatio / height
AndroidView(
    modifier = Modifier.fillMaxSize().wrapContentSize().aspectRatio(aspect ?: 1f),
    factory = { ctx -> TextureView(ctx).also { player.setVideoTextureView(it) } },
    update = { player.setVideoTextureView(it) },
)
// On dispose: player.clearVideoSurface(); player.release()
```

A second full-screen player in the same app, shown over a blurred backdrop, was changed the same way. The fix isn't confirmed yet: the session ended before it was tried on the phone.

Other ways to get a `TextureView`: `PlayerView` accepts `app:surface_type="texture_view"`, but only as an XML attribute, so the view has to be inflated from a layout. Newer Media3 releases also have a `PlayerSurface` composable (in `media3-ui-compose`) with a texture-view option; check that your version has it.

## Watch for

- Any `SurfaceView` hosted in Compose with other content over it, or inside a pager, a zoom container or an animated transition: video players, camera previews (CameraX `PreviewView`'s default performance mode uses a `SurfaceView`), maps.
- Sound with no picture, or a picture that stays put while the rest of the screen animates, points to a `SurfaceView` layering problem rather than a decoding problem.
- `TextureView` costs a little more power and latency. For a plain full-screen video with nothing drawn over it, a `SurfaceView` is still the better choice.

## Minimal example

```kotlin
@Composable
fun PosterThenVideo(player: ExoPlayer, poster: ImageBitmap, playing: Boolean) {
    Box(Modifier.fillMaxSize()) {
        if (playing) {
            // SurfaceView (PlayerView's default): sound plays, but the picture can't be seen
            AndroidView(factory = { PlayerView(it).apply { this.player = player; useController = false } })
        }
        Image(poster, contentDescription = null)   // drawn over the hole the SurfaceView punched
    }
}
```
