# Upstream's reminder behaviour

Research date: 2026-10-18. Source: `https://github.com/yukuku/androidbible`, branch `develop` at commit `583d7fbd82d302fb3993c5c8e67421b1cd271b16` (committed 2026-10-16). Historical code read at the parent of `fc1a15dcd` (2023-04-20). Read through `gh api` contents and commit endpoints.

Terms follow `GLOSSARY.md`: **Upstream**.

Legend: **Fact** = read from source, cited as a path relative to the Upstream repo root. **Derivation** = inference from the cited code.

## 1. Headline

Upstream has no daily reminder on `develop`. The only reminder it ever shipped was a "Devotion reminder", a separate companion module that was merged in 2015. Upstream removed it in 2023 (commit `fc1a15dcd`, message "No more devotion reminders. 4870 out of 2,984,595 users in the last month"). Its CHANGELOG records the removal under `5.0.0-beta.4` (2026-10-14).

Upstream's "push notifications" are not user-visible notifications. They are FCM data messages that trigger a background sync. Parity with the ticket's premise therefore points at the devotion reminder, not at push.

## 2. Facts about the removed devotion reminder (pre-removal state)

Read from parent of `fc1a15dcd`:

- **Time setting.** Stored as a string `reminder_time` in `HHmm` form. Empty means off. `Alkitab/src/main/java/yuku/alkitab/reminder/util/DevotionReminder.java` (`setAlarm`) parses hour and minute from `substring(0,2)` and `substring(2,4)`. If that time has passed today, the first fire is tomorrow.
- **Repeat.** `AlarmManager.setRepeating(RTC_WAKEUP, …, INTERVAL_DAY, pi)`. Not an exact alarm. Same file.
- **Reschedule.** `DevotionReminder.scheduleAlarm()` is called from `App`. `Alkitab/src/main/java/yuku/alkitab/reminder/br/BootCompleteReceiver.kt` is a no-op receiver for `BOOT_COMPLETED`; the comment says the App class reschedules on launch. Permission `RECEIVE_BOOT_COMPLETED` is in the manifest (`Alkitab/src/main/AndroidManifest.xml`, from the removed companion module).
- **Message.** Title `dr_notification_title` = "Daily devotion". Text `dr_notification_text` = "Read today's devotion now." The text can be overridden by the `reminder_text` preference. Source: `values/strings.xml` at the parent commit.
- **Sound and vibration.** Sound defaults to the system notification sound. The user may pick a ringtone (`reminder_sound`; empty means silent). Vibration is a boolean (`reminder_vibrate`) that sets a 500 ms on/off pattern, 9 segments. Source: `reminder/br/DevotionReminderReceiver.kt`.
- **Channel.** Channel id `devotion_reminder`, importance `IMPORTANCE_DEFAULT`, name "Devotion reminders". Upstream deletes this channel on startup (`App.java`, lines 91–95, `deleteNotificationChannel("devotion_reminder")`).
- **Tap behaviour.** `PendingIntent.getActivity(… DevotionActivity.createIntent() …)`. A tap opens the **Devotion** screen, not the Bible reader. Source: `DevotionReminderReceiver.kt`.
- **Settings UI.** A settings screen with a time picker (`ReminderTimePreference.kt`, `HackedTimePickerDialog.java`), with the sound, vibrate, and text preferences enabled only when a time is set. Also reachable from the Devotion menu (`res/menu/activity_devotion.xml`).

## 3. FCM push (not reminders)

- `Alkitab/src/main/java/yuku/alkitab/base/sync/FcmMessagingService.kt` receives FCM messages and forwards them to `FcmIntentService`.
- `Alkitab/src/main/java/yuku/alkitab/base/sync/FcmIntentService.java`, `handle()`: if `kind == "sync"`, it calls `Sync.notifySyncNeeded(syncSetNames)`. No notification is posted. Messages sourced from the device's own installation id are ignored.
- `Sync.notifyNewFcmRegistrationId` runs on token refresh.
- CHANGELOG 5.0.0-beta.4: "Push registration waits until you sign in to sync, so a fresh install no longer asks for notification permission it has no use for."

Derivation: the push channel is sync-only and user-invisible, so it has no equivalent to a daily reminder.

## 4. Current develop

- No file in the `develop` tree matches `reminder` or `Reminder` under `Alkitab/src/main/java` (checked with a recursive tree listing and `search/code`). The `publication/` screenshots and icon still show it.
- `search/code` for `devotion_reminder` finds only `App.java`, which deletes the channel.

## 5. Sources

- `fc1a15dcd` (commit "No more devotion reminders…"): `https://github.com/yukuku/androidbible/commit/fc1a15dcdef9652fb2f7827b16ef063457c31435`
- Parent contents read via `gh api repos/yukuku/androidbible/contents/<path>?ref=fc1a15dcd^`
- `CHANGELOG.md` at `develop`, section `5.0.0-beta.4 — 2026-10-14`
- `Alkitab/src/main/java/yuku/alkitab/base/sync/FcmIntentService.java` and `FcmMessagingService.kt` at `develop`
- Commit history for reminder code: `git log` search gives the 2013–2017 commits (`9f00cbc9a`, `6f58e58c4`, `dd6068a96`, and others); the removal is `fc1a15dcd` (2023-04-20).

Nothing here was executed. No device or emulator was used.
