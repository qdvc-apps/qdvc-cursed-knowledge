> **Outcome (2026-11-20):** New entry "Immich's metadata search returns the hidden video half of each Live Photo" in Other libraries and services (section renamed so it no longer needs renaming for each new library).

# Immich's `POST /search/metadata` returns the hidden video half of each Live Photo as an asset of its own

- **Date encountered:** 2026-11 (month only)
- **Reported by:** Claude Opus 5.5
- **Project visibility:** Non-public
- **Project:** Non-public QDVC project
- **Platform:** Other libraries (Immich server API; suggest adding Immich to that section's title)
- **Versions:** Immich server v3.3 (API read against the v3.3.6 source); Live Photos from iPhones (HEIC still plus MOV motion); clients on iOS 26 and Android 17
- **Log link:** Not available (non-public project)
- **Status:** Confirmed fixed (the following round's report about Live Photos concerned only playback of the merged item, with no stray videos and no repeated download failures)
- **Related entry:** None

## Symptom

A client listed an album's contents with `POST /search/metadata` and a body of `{"albumIds": ["<albumId>"], "withExif": true, "page": 1, "size": …}`. For every iPhone Live Photo in the album it got two items: the HEIC still, and a `.mov` video with no thumbnail. Requests for those videos' thumbnails failed with HTTP 404 on every sync, so a notification kept saying "<N> item(s) couldn't be downloaded", however often it retried. The Immich web app shows the same album with one item per Live Photo.

## Cause

Immich keeps the moving half of a Live Photo, or of an Android motion photo, as a separate `VIDEO` asset. The still points to it through `livePhotoVideoId`, and the video asset has `visibility: "hidden"`. As observed, the metadata search filtered by `albumIds` returns these hidden assets too. Immich's own timelines and album views leave them out. Immich doesn't generate thumbnails for the hidden halves, hence the 404s. The user confirmed which items were failing: all were `.mov` halves of Live Photos whose stills were also listed. The explanation of the 404 comes from reading the Immich source.

Tempting dead end: the first explanation was that Immich hadn't yet made thumbnails for photos edited in Immich. A fallback to the unedited thumbnail was added for that case. It's harmless, but it didn't explain these failures.

## Fix

After fetching a listing, filter it on the client:

1. Drop assets whose `visibility` is `"hidden"` (and `"locked"`).
2. Also drop any asset whose `id` appears as another asset's `livePhotoVideoId`, in case `visibility` is missing from an older server's response.

Fetch the motion half separately, through the still's `livePhotoVideoId`, and treat it as part of the still. Items stored by earlier syncs needed a one-off clean-up of the stray videos and their recorded failures. After that, the download failures stopped.

An alternative is to ask the server for `"visibility": "timeline"` in the search body. That wasn't used: it may also drop archived assets that belong to the album (not verified).

## Watch for

- Any Immich endpoint that lists assets for a client's own use, not just `/search/metadata`. Check whether hidden Live Photo halves come back, and look at `visibility` and `livePhotoVideoId` before counting, downloading or displaying items.
- Album counts and storage totals worked out on the client will be inflated by the hidden halves unless they're filtered first.
- A run of `.mov` items with no thumbnail, or repeated 404s on `/assets/{id}/thumbnail` for videos, is the tell.

## Minimal example

```kotlin
data class Asset(val id: String, val visibility: String?, val livePhotoVideoId: String?)

fun visibleItems(assets: List<Asset>): List<Asset> {
    val motionHalves = assets.mapNotNullTo(HashSet()) { it.livePhotoVideoId }
    return assets.filter {
        it.visibility != "hidden" && it.visibility != "locked" && it.id !in motionHalves
    }
}
```
