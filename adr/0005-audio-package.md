# expo-audio for chapter audio, streamed from the backend

Chapter audio uses `expo-audio` on iOS and Android. The app streams each chapter MP3 from the backend URL that `/audio/sets/<preset>` returns, as Upstream does. Nothing is bundled into the app. `expo-audio` is the Expo SDK 57 package for audio playback, and its web support is listed by the docs but not verified here.

The source is Upstream's audio service on `https://api.alkitab.app`, documented in `docs/research/upstream-audio.md`. ADR 0001 parks online features until the maintainer confirms the terms for `api.alkitab.app`. Playback from that host is therefore an online feature under ADR 0001, and it ships only once that ticket clears. The audio recordings themselves are published by SABDA, and the terms for them are not confirmed in any source read here.

Considered and rejected: `expo-av`. The SDK 57 docs URL for `expo-av` returned 404 when fetched for this ADR, which suggests the package is no longer documented for SDK 57. That is an observation, not a changelog reading, so it needs confirming before a final choice. Upstream's own player is ExoPlayer through media3, which has no Expo binding. Bundling MP3 files into the app was rejected because Upstream does not bundle them, and the backend's contract says the app ships no fallback audio.

## Decision

Use `expo-audio` (`npx expo install expo-audio`, which is not yet in `package.json`). The reader calls `useAudioPlayer` or `createAudioPlayer` with a remote HTTPS source built from the URL template.

Background playback and lock-screen controls are needed for the ticket's "plays, pauses and resumes" intent on mobile. The SDK 57 docs describe the setup:

- `app.json` config plugin: `enableBackgroundPlayback` (default `true`). On Android it adds a media playback foreground service. On iOS it adds the `audio` background mode.
- Runtime: `setAudioModeAsync({ shouldPlayInBackground: true, interruptionMode: "doNotMix" })` on iOS.
- Android: `player.setActiveForLockScreen(true, metadata, options)`, and `requestNotificationPermissionsAsync()` for the notification shade controls. The docs warn that Android playback stops after about 3 minutes in the background without the lock-screen setup.

## Web support

The SDK 57 docs list Web as a supported platform for `expo-audio`. They also say that browsers serve audio from secure origins only for microphone access, which does not affect playback. The docs say nothing that rules web playback out. Web playback of the remote MP3 is not tested. The Parity matrix records web as unverified until a browser check runs.

Web-specific options (`crossOrigin`, and `downloadFirst`, which needs `Access-Control-Allow-Origin` from the server) matter if web playback is wired up later. The backend's MP3 responses would need CORS headers for that path. That is noted here and not checked.

## Blocked

No native build was run. There is no iOS or Android device or dev build in this environment, and the package is not installed. The ticket's "plays, pauses and resumes on iOS and Android" checkbox stays open until the steps in section 7 of `docs/research/upstream-audio.md` run on a device. This ADR records the choice, not a working player.
