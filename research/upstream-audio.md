# Upstream audio: where it is offered, how it is fetched, and its terms

Research date: 2026-10-18. Source: `https://github.com/yukuku/androidbible` at `develop` commit `583d7fbd82d302fb3993c5c8e67421b1cd271b16` (committed 2026-10-16). Read with `gh api` (file contents at that ref). Nothing was built or run.

Legend: **Read** means read directly from the cited file at that commit. **Inferred** means derived from the cited code. **Unconfirmed** means no source read here settles it.

Terms used here follow `GLOSSARY.md`: **Upstream**, **Version**, **Parity matrix**.

---

## 1. Where audio is offered

**Read.** Upstream offers Bible chapter audio in the reader. The audio bar, the recording chooser, and the verse highlight live in `Alkitab/src/main/java/yuku/alkitab/base/audio/` (`AudioBarController.kt`, `BibleAudioService.kt`, `HighlightTracker.kt`, and `ui/AudioBar.kt`). The reader menu has an Audio item (`res/menu/activity_isi.xml`, `menuAudio`, as recorded in `docs/research/androidbible-ui-parity.md`). The verse action mode has "Play audio from verse" (`docs/research/androidbible-ui-parity.md`, verse action bar row).

**Read.** Upstream also has song (hymn) audio in `Alkitab/src/main/java/yuku/alkitab/songs/`. That is out of scope for this ticket.

**Read.** `docs/modules/audio-playback.md` says playback runs in a foreground `MediaSessionService` on AndroidX media3 ExoPlayer, which gives background playback and a lock-screen notification.

## 2. Bundled or fetched

**Read.** Audio is fetched, not bundled. `docs/features/audio-bible/prd.md` §6 says audio and timing data "are served by `api.alkitab.app`, which may 302 to sabda.org or a CDN."

**Read.** `Alkitab/build.gradle.kts` line 187 sets `val serverHost = "https://api.alkitab.app"`, and line 217 exposes it as `BuildConfig.SERVER_HOST`.

**Read.** `docs/features/audio-bible/backend-contract.md` says the app "ships **no bundled fallback**." It also says the app never builds a media path. It calls `GET /audio/sets/<preset>` and expands the returned `mp3UrlTemplate` and `timingUrlTemplate` with `{book_1}` and `{chapter_1}`.

**Read.** `BibleAudioRepository.kt` builds URLs with `BuildConfig.SERVER_HOST` as the base, and `AudioSetsRepository.kt` uses `SETS_PATH = "/audio/sets/"`.

**Read, with a conflict.** `docs/modules/audio-playback.md` names `audio/AudioCatalogRepository.kt` and says it has "a bundled fallback." That file is absent from the tree at `583d7fb`, and `multi-audio-sets-design.md` says the bundled `assets/audio_catalog.json` was replaced. Treat the module doc as stale. The backend contract and the design doc agree that nothing is bundled.

**Inferred.** Because the app takes URL templates from the backend, the actual MP3 host is whatever the backend redirects to. The PRD says that may be sabda.org or a CDN. Those hosts were not read.

## 3. Endpoints

**Read** from `backend-contract.md`:

| Method | Path | Notes |
|---|---|---|
| GET | `/audio/sets/<preset>` | `preset` is the Version's `preset_name` (`in-tb`, `en-kjv`, `ms-avb`). Returns `sets` with `audioId`, `hasTiming`, `books_1`, and URL templates. |
| GET | `/audio/file/<preset>/<audioId>/<book_1>/<chapter_1>.mp3` | Chapter audio, served directly (not a redirect). `Accept-Ranges: bytes`. |
| GET | `/audio/timing/<preset>/<audioId>/<book_1>/<chapter_1>.json` | Verse timing in integer milliseconds. `verses: []` means audio without timing. |

**Read.** Caching: sets use `ETag` and `max-age=86400`, MP3 uses `max-age=2592000`, timing uses `max-age=604800` (`backend-contract.md`).

## 4. File format

**Read.** Chapter files are MP3: `Content-Type: audio/mpeg` (`backend-contract.md`). The URL scheme ends in `.mp3`.

**Read.** Upstream plays MP3 through ExoPlayer over OkHttp, with an MP3-only extractor (`docs/features/audio-bible/prd.md` §5, "We borrow ... MP3-only extractor factory").

**Read.** Verse timing is JSON with integer `startMs` and `endMs` (`backend-contract.md`, `model/ChapterTiming.kt`).

## 5. Terms and licence

**Read.** The code is under the Apache License 2.0 (`LICENSE.txt` at the repo root, first lines: "Apache License Version 2.0, January 2004"). That covers Upstream's code, not the audio.

**Read.** `prd.md` §6: "The backend carries the licensing responsibility — there is no per-version attribution surface in the client UI for v1."

**Read.** `prd.md` §1 calls SABDA's service "a free Bible-audio service."

**Read.** `multi-audio-sets-design.md` lists per-set attribution as an open idea ("The backend can serve narrator/publisher text, but the bar has no room for it").

**Unconfirmed.** No source read here states a licence, terms of use, or permission for the audio recordings. The "free" wording in the PRD is a description, not a licence. The terms of `api.alkitab.app` and of sabda.org were not read. ADR 0001 already parks online features until the maintainer confirms terms for `api.alkitab.app`, so this stays open.

## 6. Relevance to AlkitabNext

**Derivation.** The chapter audio is an online feature under ADR 0001, because it comes from `api.alkitab.app`. Playback waits on the parked ticket for those terms. The Parity matrix row for audio should say so.

**Derivation.** The app needs the `preset_name` of a Version. This repo does not yet store one. That is a separate issue for the audio ticket's implementation.

## 7. Native check (blocked)

The build of chapter playback is blocked on a native check. No iOS or Android device or dev build exists in this environment. The package `expo-audio` is not installed, and per the task it must not be installed here. Steps for a person with the hardware:

1. On a branch, run `npx expo install expo-audio` and commit the lockfile change.
2. Add the config plugin in `app.json`: `"enableBackgroundPlayback": true`. Rebuild the dev client with `npx expo prebuild` or `eas build --profile development`.
3. Confirm `GET https://api.alkitab.app/audio/sets/in-tb` returns JSON with `sets` (a `curl -i` check, recording the status and headers).
4. Pick a Version with audio, open a chapter it covers, and play the chapter MP3 from the URL template with `useAudioPlayer`.
5. On iOS, check that playback continues after the app backgrounds and that the lock screen shows controls, with `shouldPlayInBackground: true` and `interruptionMode: "doNotMix"`.
6. On Android, check the media notification and the lock-screen controls with `setActiveForLockScreen(true, ...)` and `requestNotificationPermissionsAsync()`. Let the app sit in the background for more than 3 minutes and confirm it keeps playing.
7. Pause, resume, and seek on each platform. Record the result in the Parity matrix row for audio, per platform.
8. Run the same chapter in a browser on the web build and record whether it plays, with its CORS result.

Until steps 2 to 7 pass on a device, the ticket's "plays, pauses and resumes on iOS and Android" checkbox stays open.
