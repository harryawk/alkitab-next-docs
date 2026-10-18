# Daily reminders use `expo-notifications` on iOS and Android, and the Notification API on web

Daily reminders use `expo-notifications` local scheduling on iOS and Android, and the browser Notification API on web while the app is open. Upstream has no daily reminder to match: its devotion reminder was removed in 2023 (see `docs/research/upstream-reminders.md`), and its "push notifications" are a silent sync trigger. So the reminder is a new feature defined by the ticket, not a port.

## Package decision

`expo-notifications` in SDK 57, because it is the Expo-managed package that the ticket's native targets fit. Its docs list scheduling as supported on Android and iOS, with an Android channel requirement and `SCHEDULE_EXACT_ALARM` for exact-time triggers on Android 12+. Its tap listener returns the notification `data` payload, so a reminder can carry a route. Expo's docs state that local notifications remain available in Expo Go, and that a development build is needed for remote push. Local scheduling in Expo Go is therefore not ruled out, but this ADR does not depend on it: the native check below runs on a development build.

Considered and rejected:

- `@notifee/react-native`: a bare native library with its own channel and trigger API. It would add a second native stack next to Expo's, with no capability the ticket needs that `expo-notifications` lacks.
- `react-native-push-notification`: unmaintained against current Android and iOS requirements.
- Server push (FCM, APNs, or Web Push): needs a server that sends at each user's chosen time, and `api.alkitab.app` is parked until the maintainer confirms terms (ADR 0001). Upstream's FCM is sync-only and does not carry reminders.

## Web fallback

The Notification API is the web fallback, but it is weaker than native. The MDN page read for this ADR says Notification is "available only in secure contexts (HTTPS)", that a permission is `default`, `granted` or `denied`, and that browsers can refuse permission requests not triggered by a user gesture. It does not say a notification can be scheduled to show later. So on web the reminder is shown only when all of these hold:

- the page is open or a service worker is running at the reminder time,
- the user has granted permission from a button tap, and
- the site is served over HTTPS.

Behaviour in the other cases:

- Permission `denied`: no reminder is shown. Settings shows "Reminders are blocked in this browser. Allow notifications for this site to get them." and does not ask again automatically.
- Permission `default`: the reminder is set only after the user taps "Allow reminders"; the app does not prompt on load.
- Unsupported (no `Notification` in `window`, or iframe): the reminder row is hidden.

The web fallback is recorded in the Parity matrix as the ticket asks, once the matrix row exists; this branch does not edit the matrix. Scheduled web notifications through a service worker or the Notification Triggers API are not decided here; neither was verified in this research.

## Tap to reader

On native, the reminder payload carries `{ route: "/reader" }`. The app's `addNotificationResponseReceivedListener` calls `router.push` with it, and `useLastNotificationResponse` (or `getLastNotificationResponse`) handles a tap that launches a cold app. On web, the notification's `onclick` focuses the window and pushes `/reader`.

Upstream's tap opens the Devotion screen, not the reader. This ADR chooses the reader because the ticket asks for it, and the difference is recorded here.

## Consequences

- No code is written for this feature yet. Native scheduling cannot be verified here: there is no device or development build, and the Expo `expo-notifications` install would change `package.json`, so it waits for the device check.
- Exact-time delivery on Android 12+ depends on the user's exact-alarm permission. Without it, the reminder can fire late.

## Native check (blocks the implementation)

A person must run these on a development build (`npx expo run:android` and `npx expo run:ios`, or an EAS dev build) before the feature is merged:

1. Install `expo-notifications` at the version pinned for SDK 57 and rebuild the dev client.
2. Android 13+ and iOS: grant notification permission; schedule a daily reminder for two minutes ahead; background the app; confirm it appears at the time.
3. Android 12+: check whether `SCHEDULE_EXACT_ALARM` is granted and note the delay with it denied.
4. Tap the notification with the app killed; confirm it opens the reader at the verse or chapter the reminder carries.
5. Tap with the app backgrounded; confirm the same.
6. Web: in Chrome and Safari (HTTPS), click "Allow reminders", set a reminder two minutes ahead with the tab open, and confirm the notification shows. Then deny permission and confirm the settings message appears.

The Parity matrix row for web reminders waits for steps 6 and the denied path.
