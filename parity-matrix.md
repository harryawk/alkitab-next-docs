# Parity matrix

One row per upstream feature, with what this app does on each platform. The upstream reference is `yukuku/androidbible` on `develop` (ADR 0001). A cell names a replacement when the platform has no equivalent, so a gap is recorded rather than dropped.

| Feature | iOS | Android | Web |
| --- | --- | --- | --- |
| Chapter paging by hardware volume keys (upstream `pref_volumeButtonNavigation`, default off) | Not built. iOS has no volume-key paging, and the app uses the Previous and Next chapter buttons and swipe. | Not built. No Expo SDK 57 API reports hardware volume-button presses (searched the Expo docs and the npm packages on 2026-10-17). Replacement: the Previous and Next chapter buttons and swipe. A native package such as `react-native-volume-manager` would need a development build, and it is deferred because upstream ships this setting off by default. | Not applicable. Replacement: the Previous and Next chapter buttons. |
