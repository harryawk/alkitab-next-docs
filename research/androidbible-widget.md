# Upstream (yukuku/androidbible) home-screen widget

Research date: 2026-10-18. Upstream commit read: `583d7fbd82d302fb3993c5c8e67421b1cd271b16` (`develop` head when fetched). The parity note `docs/research/androidbible-ui-parity.md` cites `3e3f6ad`, an older develop commit; the widget files below were read at `583d7fb`.

Fetched with `gh api repos/yukuku/androidbible/contents/<path>?ref=583d7fb...`. All paths are under `Alkitab/src/main/`.

## What it is

Upstream's widget is a **daily verse** widget. It is not a user-picked verse. Its own name is "Daily verse" (`DailyVerseAppWidget*`, string `dv_activity_title`).

## What it shows

- `java/yuku/alkitab/base/appwidget/DailyVerseFactory.java`: a `RemoteViewsFactory`. Item 0 is a header with a reference label (`version.referenceWithVerseCount(aris[0], aris.length)`), an optional app-logo button, and prev/next buttons. Items 1..n are verse rows. A passage can span several verses; `getVerseViewAt` shows a small superscript verse number when the passage has more than one verse.
- Text size, dark or light text, transparent background (with alpha), and hide-app-icon are per-widget settings (`DailyVerseData.SavedState`).
- Layout: `res/layout/appwidget_daily_verse_container.xml`, `appwidget_daily_verse_item_header.xml`, `appwidget_daily_verse_item_verse.xml`.
- Provider XML: `res/xml/daily_verse_app_widget_provider.xml` (not fetched; referenced from the manifest).

## How the verse is picked

Source: `java/yuku/alkitab/base/appwidget/DailyVerseData.java`.

1. A bundled binary list `R.raw.daily_verses_bt` holds predefined passages. Each entry is an ARI and a verse count (`listAllDailyVerses`).
2. `getIndexFromSeed(appWidgetId, size, click)` seeds `java.util.Random` with the widget id, the day of year (and year), and the `click` offset. The index is `r.nextInt(size)`. Debug builds use a 15-second bucket. Release builds use the day only.
3. So each widget shows the same passage all day. Different widgets differ because the widget id is in the seed.
4. If the chosen passage is missing in the selected version, `getAris` steps `click` by the direction and retries, up to 20 tries.
5. The user changes the passage with the prev/next buttons. `DailyVerseAppWidgetReceiver.ClickReceiver` adds ±1 to `click`, saves it, and rebuilds the widget.
6. A `setRepeating` `AlarmManager` alarm (`DailyVerseAppWidgetReceiver.setAlarm`) rebuilds all widgets daily at local midnight. It reseeds from the day.
7. The version is chosen in `DailyVerseAppWidgetConfigurationActivity`. The configuration activity lists available versions and writes the choice as `app_widget_<id>_version`. It falls back to the internal version if the chosen one is gone (`DailyVerseData.getVersion`).

The user does not pick a verse by reference. The user picks a version, styling, and steps through a seeded list.

## How a tap is handled

Source: `DailyVerseFactory.java`, `br/DailyVerseAppWidgetReceiver.kt` (`ClickReceiver`).

- Tapping the reference header (`tReference`) sets a `PendingIntent.getActivity` with `Launcher.openAppAtBibleLocation(aris[0])`. This opens the reader at the first verse of the passage.
- Tapping a verse row sets a fill-in intent with `app_widget_action = open_verse` and `ari`. `ClickReceiver` then starts `Launcher.getBaseViewIntent()` with an `ari` extra. This is an in-app intent, not a URL.
- The logo sends `open_app`. Prev/next send `update_widget`, which is handled inside the receiver and does not open the app.
- Pending intents use request codes `appWidgetId + 10000` and `appWidgetId * 10 + 2`.

So the Android equivalent of "tap opens the verse" is an Intent with an `ari` extra. There is no URL scheme involved.

## Platform requirements

- `AndroidManifest.xml`: `DailyVerseAppWidgetReceiver` with `APPWIDGET_UPDATE`, `ClickReceiver`, `UpdateService` (JobIntentService), `DailyVerseAppWidgetService` (`BIND_REMOTEVIEWS`), and `DailyVerseAppWidgetConfigurationActivity` (`APPWIDGET_CONFIGURE`).
- Android uses `RemoteViews` and `RemoteViewsService`. These are not available on iOS.

## Relevance to this ticket

- Upstream's widget is a daily verse with a seeded list, prev/next, and a version choice. The ticket says "a verse the user picks". The closest iOS match is a widget that shows one verse the user chose in the app. A "daily verse" mode is a separate, optional follow-up.
- The Android tap contract (open the reader at an ARI) matches this app's `/reader?ari=...` route (ADR 0002).

## Could not verify

- The contents of `res/raw/daily_verses_bt` (binary, not fetched).
- Whether `Launcher.getBaseViewIntent` on upstream carries anything beyond the `ari` extra.
- The provider XML (`daily_verse_app_widget_provider.xml`) sizes and update period.
