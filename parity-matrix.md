# Parity matrix

One row per Upstream feature, with a column per platform. Per ADR 0001, each row records the Upstream commit it was checked against and whether a test verifies it. The full matrix and its drift job are a separate ticket.

Upstream checked against: `yukuku/androidbible` `develop` at `583d7fbd82d302fb3993c5c8e67421b1cd271b16`, except where a row says otherwise.

| Feature | iOS | Android | Web | Upstream commit | Verified by test |
|---|---|---|---|---|---|
| Chapter paging by hardware volume keys (upstream `pref_volumeButtonNavigation`, default off) | Not built. iOS has no volume-key paging, and the app uses the Previous and Next chapter buttons and swipe. | Not built. No Expo SDK 57 API reports hardware volume-button presses (searched the Expo docs and the npm packages on 2026-10-17). Replacement: the Previous and Next chapter buttons and swipe. A native package such as `react-native-volume-manager` would need a development build, and it is deferred because upstream ships this setting off by default. | Not applicable. Replacement: the Previous and Next chapter buttons. | Not recorded | No |
| Chapter audio, play, pause, resume | Not built. Blocked on a device check (`docs/research/upstream-audio.md` section 7). Chosen package: `expo-audio` (ADR 0005). | Not built. Blocked on a device check. Chosen package: `expo-audio` (ADR 0005). | Not built. `expo-audio` lists Web, unverified. | 583d7fb | No |
| Audio lock-screen and background playback | Not built. Needs `enableBackgroundPlayback` and `shouldPlayInBackground`. | Not built. Needs `setActiveForLockScreen` and the notification permission. | Not applicable. The web platform has no lock-screen media session in this plan. | 583d7fb | No |
| Audio source terms (`api.alkitab.app`) | Parked under ADR 0001 until the maintainer confirms terms. | Parked under ADR 0001. | Parked under ADR 0001. | 583d7fb | No |
