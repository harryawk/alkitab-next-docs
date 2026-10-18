# iOS home-screen widget for a chosen verse (issue #35)

Research date: 2026-10-18. Branch `issue-35-ios-home-widget` from master `61ecd26`.

## Decision

**Status: NOT implemented in this branch.** This branch holds research only. It contains no widget code, no `expo-widgets` config, and no native target.

- **iOS equivalent.** A WidgetKit home-screen widget (small or medium family) that shows one verse the user chose in the app. Upstream's Android widget is a seeded daily verse with prev/next (see `docs/research/androidbible-widget.md`). The ticket asks for a verse the user picks, so this is the matching iOS surface.
- **Build path.** Add the `expo-widgets` config plugin to `app.json`, then build a new iOS binary with a development build (or an EAS development build). Expo Go cannot run it.
- **Tap target.** `alkitabnext://reader?ari=<ari>`, following ADR 0002. The app already parses `/reader?ari=` (`app/reader.tsx`). The widget tap wiring is not documented by Expo for home-screen widgets (see "Unverified").

### Why it is not in this branch

1. **Native target.** Expo SDK 57 documents the widget as a config-plugin-generated WidgetKit extension. The repo has no `ios/` folder (managed workflow, Continuous Native Generation). The change needs a prebuild and a new native binary, which this worktree cannot produce or test.
2. **No device.** The iOS simulator and a physical device are not available in this session. Widget rendering and the tap cannot be checked here. The ticket's checkboxes require a device run.
3. **Data path not documented.** Expo's page says a widget gets data through props on `updateSnapshot` or `updateTimeline`, and shares files through `widgetsDirectory`. It does not document how a widget reads the app's SQLite store. Choosing that path (App Group file vs. snapshot write from JS) is a design decision that should be made with the device in hand.
4. **Tap deep link not documented for widgets.** The docs describe a URL for Live Activities only. Whether a widget tap opens `alkitabnext://reader?ari=...` needs a device check.

## Expo SDK 57 facts (read)

Sources read on 2026-10-18:

- https://docs.expo.dev/versions/v57.0.0/sdk/widgets.md (the widgets page, which the `v57.0.0/` index links as "Widgets")
- https://docs.expo.dev/versions/v57.0.0/ (the index, used to find the widgets page)

Facts:

- Package: `expo-widgets`. Install with `npx expo install expo-widgets`.
- Platform: iOS only. The API entries say "Supported platforms: iOS."
- Expo Go: "This library is not available in the Expo Go app — use development builds to try it out."
- Config plugin: `expo-widgets` in the `plugins` array of `app.json`. Properties include `bundleIdentifier`, `groupIdentifier`, `enablePushNotifications`, and `widgets` (each with `name`, `displayName`, `description`, and `ios.supportedFamilies`).
- The plugin is used with Continuous Native Generation (CNG). Its properties "require building a new app binary to take effect."
- `groupIdentifier` defaults to `group.<bundle identifier>`. Prebuild fails if `ios.bundleIdentifier` is unset. This repo sets `com.anonymous.AlkitabNext`.
- Shared files: `widgetsDirectory` contents are "accessible by both the main app and widgets". A widget "can't access files inside your app's sandbox."
- Live Activities only: "When the user taps the activity, the system opens your app with that URL."
- Not on the page: EAS Build, web, Android, the deployment target, a tap URL for home-screen widgets.

Repo facts (read at `61ecd26`):

- `app.json`: `scheme: "alkitabnext"`, `ios.bundleIdentifier: "com.anonymous.AlkitabNext"`, no `ios/` folder.
- `package.json`: `expo ~57.0.18`, `expo-router ~57.0.17`, `react-native 0.86.3`.
- `app/reader.tsx` handles `?ari=`.

## Manual check (a person must run this on a device)

Run on an iPhone with a development build.

1. Install `expo-widgets`, add the plugin to `app.json`, run `npx expo prebuild --clean`, and build a development build for the device (`npx expo run:ios --device` or an EAS development build).
2. Long-press the home screen, add the AlkitabNext widget (small and medium), and confirm it shows the saved verse with its reference.
3. Choose a different verse in the app. Confirm the widget updates within the expected refresh window, or after backgrounding the app.
4. Tap the widget. Confirm the app opens `/reader` at that ARI (for example `John 3:16`). Test once from a cold start and once with the app already open.
5. Confirm the app's `alkitabnext://` scheme works from a Safari link (`alkitabnext://reader?ari=...`) as a control, to separate a tap-routing bug from a scheme bug.
6. Record the result, the device, and the iOS version in the Parity matrix row below.

## Parity matrix row

The file `docs/parity-matrix.md` lives on branch `issue-37-volume-key-paging`, not on master. This branch does not create it. Merge this row into that file later.

| Upstream feature | Upstream file | iOS equivalent | Web | Expo Go | Build path | Status | Manual check |
|---|---|---|---|---|---|---|---|
| Home-screen widget showing a verse (Android: daily verse, `DailyVerseAppWidget*`) | `appwidget/DailyVerseFactory.java`, `br/DailyVerseAppWidgetReceiver.kt` | WidgetKit widget via `expo-widgets` config plugin, showing a user-chosen verse. Tap opens `alkitabnext://reader?ari=...` | No (no home screen) | No (requires development build) | Config plugin, prebuild, development or EAS build (iOS only) | Not implemented (ticket #35). Research only: `docs/research/ios-home-widget.md` | Install on iPhone, pick verse, confirm widget text, tap opens `/reader?ari=...` at that verse (see the manual check list in `docs/research/ios-home-widget.md`) |
