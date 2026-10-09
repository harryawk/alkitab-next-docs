# Upstream installed-Version registry, Versions screen, in-reader chooser and Compare dialog (yukuku/androidbible)

Research date: 2026-10-09. Source: `https://github.com/yukuku/androidbible`, branch `develop`, commit `3e3f6ad61fe7f0cfccb10042977a89084ad5b2d6` (2026-10-07, "Merge pull request #313 from yukuku/codex/fix-reading-guide-and-drawer"), read from a source tarball of that exact commit.

Convention: **Fact** = read in source, cited as `path:Lnn` relative to the androidbible repo root (all paths below are under `Alkitab/src/main/java/yuku/alkitab/` unless they start with `AlkitabModel/`). **Unverified** = not confirmed by reading code (not executed; nothing here was run). No recommendations are made.

Abbreviations: `base/` = `Alkitab/src/main/java/yuku/alkitab/base/`, `vm/` = `.../yuku/alkitab/versionmanager/`.

---

## 1. Model classes

### 1.1 Hierarchy (Fact)
- `MVersion` (abstract) `base/model/MVersion.java`: public fields `locale`, `shortName`, `longName`, `description`, `int ordering`, transient `group_order` (only populated when the preset list is loaded, not persisted). Abstract: `getVersionId()`, `getVersion()` (nullable `Version` to read from), `getActive()`, `hasDataFile()`. `getPresetName()` defaults to null.
- `MVersionInternal` `base/model/MVersionInternal.java`: the bundled version, "only one". `getVersionId()` returns the constant string `"internal"`; `getActive()` always true; `hasDataFile()` always true; `DEFAULT_ORDERING = 1`. `getPresetName()` returns `BuildConfig.INTERNAL_VERSION_PRESET_NAME` or null if empty. Per flavor that is `"in-tb"` or `"en-kjv"`, or `""` (`Alkitab/build.gradle.kts:228,281,287,291,295`).
- `MVersionDb` `base/model/MVersionDb.java`: a version persisted in the SQLite `Version` table. Fields: `filename` (absolute path of the .yes), `preset_name` (nullable), `cache_active`, `modifyTime` (0 = no update notification; "converted from legacy version of this app, or added from pdb/yes files manually"). `DEFAULT_ORDERING_START = 100`. `hasDataFile()` = `File(filename).exists() && canRead()`.
- `MVersionPreset` `base/model/MVersionPreset.java`: an entry from the downloadable preset list (`version_config.json`), not installed. `getActive()` always false; `getVersion()` throws `RuntimeException`; `getVersionId()` = `"preset/" + preset_name`; `hasDataFile()` checks `AddonManager.getReadableVersionFile(preset_name + ".yes")`.

### 1.2 Id scheme (Fact)
- Internal: `"internal"` (`MVersionInternal.java`, `getVersionInternalId`).
- Installed from a preset: `"preset/" + preset_name` (e.g. `preset/en-kjv`). Side-loaded / URL-downloaded file: `"file/" + filename` where filename is the **absolute path** of the .yes (`MVersionDb.java`, `getVersionId`). The Javadoc notes an id starting `preset/` can be either `MVersionPreset` or `MVersionDb`.
- `MVersionDb.presetNameFromVersionId(id)` strips the `preset/` prefix, else null.
- The id is what is persisted as the last-read version (`Prefkey.lastVersionId`, written in `IsiActivity.kt:1185`) and as key for per-version settings (`PerVersion` table, `versionId` unique index, `base/storage/InternalDbHelper.java:229-246`; font multiplier read via `getPerVersionSettings(versionId)`).

### 1.3 Short/long name (Fact)
- `shortName` / `longName` / `locale` / `description` of an installed version are **copied from the .yes file header at install time** (`reader.getShortName()`, etc.) into the `Version` table (`vm/VersionsActivity.kt:202-210`; `base/br/VersionDownloadCompleteReceiver.java:140-149`), not re-read on each listing. For the internal version they come from `AppConfig` (`internalShortName`, `internalLongName`, `internalLocale`) (`base/S.kt`, `getMVersionInternal`).
- Display rule on the Versions screen row: if `shortName` is non-blank, show shortName (primary) + longName (secondary); otherwise longName is the primary and no secondary (`vm/VersionListFragment.kt:452-460`). The in-reader chooser uses the same rule (`base/util/VersionDialogHelper.kt`, `toRow`).
- `Version.getInitials()` (used by Compare for the row label): returns `shortName` if non-null; else if longName is null `""`; else longName uppercased if length <= 6; else first letter of each alnum word, uppercased (`AlkitabModel/src/main/java/yuku/alkitab/model/Version.java:232-257`). Note a non-null but blank shortName is returned as-is (Unverified whether yes readers ever yield blank).

### 1.4 `active` flag (Fact)
- Persisted column `active` (integer) in table `Version`; in memory `MVersionDb.cache_active`, "so we don't need to keep reading/writing from/to db". `setActive()` updates the cache and writes the DB (`MVersionDb.java`).
- Table `Version` (`base/storage/InternalDbHelper.java:249-262`): `_id integer primary key autoincrement`, `locale, shortName, longName, description, filename, preset_name` text, `modifyTime, active, ordering` integer. Indexes on `ordering`; `(active, longName)`; `preset_name`. No unique constraint on `filename` or `preset_name`.
- `VersionDao.setActive` matches by `preset_name` when non-null, otherwise by `filename`; a source comment notes that if two rows share a non-null preset_name both flip (`base/storage/VersionDao.kt`).
- Installing a version inserts it with `active = true` (`insertOrUpdateVersionWithActive(mvDb, true)` in both install paths).

## 2. Listing, ordering, availability

### 2.1 `S.getAvailableVersions()` (Fact, `base/S.kt:254-270`)
Returns: (1) `getMVersionInternal()`, then (2) every `db.listAllVersions()` row where `hasDataFile() && active`; then `sortWith { lhs.ordering - rhs.ordering }`. So hidden (inactive) versions and rows whose file is gone are excluded; the internal version is always present. `listAllVersions` itself already returns rows `order by ordering asc` (`VersionDao.kt`, `listAll`).
- `getMVersionInternal()` is not a singleton: builds a new `MVersionInternal` each call, `ordering = Preferences.getInt(Prefkey.internal_version_ordering, 1)` (`S.kt`).
- `getVersionFromVersionId(id)`: null for null or `"internal"`, null if no DB row has that id, null if the row has no data file; **it does not check `active`** (an inactive-but-installed version can still be resolved, e.g. as last-read version). Callers treat null as "use internal" (`S.kt`, `ActiveVersionHolder` init).

### 2.2 Ordering (Fact)
- `ordering` is an int; internal's lives in SharedPreferences (`internal_version_ordering`, default 1), DB versions in the column. First DB-installed version gets `DEFAULT_ORDERING_START + 1 = 101` (`maxOrdering` is 0 on an empty table, mapped to 100, then `+1`), later ones `max + 1` (`VersionsActivity.kt:199-208`, `VersionDownloadCompleteReceiver.java:136-137,152`). Unverified: that `getMaxOrdering()` yields 0 for an empty table (`DatabaseUtils.longForQuery` on `select max(...)` with NULL result); the code relies on it.
- Reorder (`InternalDb.reorderVersions(from, to)`, `InternalDb.java:636-682`): shifts the rows between by +/-1 in one transaction, adjusting the internal pref ordering too when it falls in the range, then sets `from`'s ordering to `to`'s old ordering. Only called from the drag-drop on the "Downloaded" tab on drop (`vm/VersionListFragment.kt:552-608`). The internal version can be dragged like any other row (it is in the list, not excluded).
- Ordering is **shared**: the same value drives the Versions screen Downloaded tab, the in-reader chooser, and Compare (Compare gets `getAvailableVersions()`).
- Re-install of a row with an existing filename keeps its existing ordering (`VersionDao.insertOrUpdateWithActive`, see 3.3).

## 3. Versions screen (`VersionsActivity` + `VersionListFragment`)

### 3.1 Structure (Fact)
`vm/VersionsActivity.kt`: two tabs via a ViewPager (`SectionsPagerAdapter`, `getCount() = 2`): position 0 "All" (`ed_section_all`), position 1 "Downloaded" (`ed_section_downloaded`) (`:177-189`). Both are `VersionListFragment.newInstance(downloadedOnly = position == 1, query_text)`. Menu: add from local file (`getContentRequest.launch("*/*")`), add from URL; search box broadcasts the query to both fragments (`:151-175`).

### 3.2 What each tab lists (Fact, `VersionListFragment.kt` `VersionAdapter.reload()`, `:345-414`)
- Both tabs: internal version first in the build, plus **all** `listAllVersions()` rows (installed, active or not, file present or not).
- "All" tab only: additionally every preset from `VersionConfig.get().presets` that is not already installed (matched by `preset_name` against DB rows); installed ones get `group_order` copied from the preset. Sort: by `group_order` (0 treated as last), then same-locale by longName case-insensitive, otherwise by display language name; section headers by group/language.
- "Downloaded" tab: sorted by `ordering` ascending; drag handle visible; headers: `prev == null || prev.group_order != mv.group_order || ...` logic (`:468-479`), all group_order are 0 here unless set by an earlier All-tab reload (Unverified interplay).
- Text filter on both tabs: tokens must all match longName, shortName, description or display language.

### 3.3 Row interactions (Fact)
- Right-side checkbox (`panelRight` click -> `itemCheckboxClick`, `:146-152, 449`):
  - `MVersionPreset`: starts a download (no uncheck path; `throw` if checked).
  - `MVersionDb`: if checked -> `mv.active = false` (hide, keep file); else if file exists -> `active = true`; else shows a dialog "file no longer available" with Delete (removes DB row) / No (`:276-291`).
  - `MVersionInternal`: checkbox is **disabled** (`cActive.isEnabled = false`, `:463`) and the click handler has no branch for it. So internal cannot be hidden.
- Name click -> details dialog (`itemNameClick`, `:154-249`): type (Built-in / Downloaded / preset), locale, short name, title, preset filename or "stored in file", description. Buttons: Update (only `MVersionDb` with `hasUpdateAvailable`), Share (`MVersionDb` with data file; FileProvider URI), **Remove** (`buang_dari_daftar`, only `MVersionDb`): `db.deleteVersion(mv)` then emits reload then `File(filename).delete()`, with **no confirmation**; Download for presets; OK if none of the above.
- **Cannot be deleted/removed: the internal version** (no Remove button for `MVersionInternal`; checkbox disabled). Everything installed (`MVersionDb`), including preset-sourced ones, can be removed, and removal deletes the .yes file from disk. A preset-sourced one reappears in the "All" tab as a downloadable preset afterwards.
- `VersionDao.delete`: if `preset_name != null`, deletes `where preset_name=?`; if that deleted >0 rows returns; else deletes `where filename=?` (`VersionDao.kt`).
- Deleting the version that is currently open: Unverified. Not seen handled in these files; on next start `getVersionFromVersionId` returns null -> internal; at runtime `IsiActivity` loads books and falls back to internal if the version has no firstBook (`IsiActivity.kt:675-692`).

### 3.4 Duplicate filenames and re-imports (Fact)
- Dedupe key in the DB is `filename` only: `insertOrUpdateWithActive` looks up a row with the same `filename`; if found it **updates** that row (all fields rewritten, `active` set per argument, `ordering` kept from the existing row and written back to `mv.ordering`), else inserts (`VersionDao.kt`). Its own doc comment: dedupes only by filename.
- Local .yes import (`vm/VersionFileImporter.kt`): copies to the cache, validates with `YesReaderFactory.createYesReader`, then copies to `getWritableVersionFile("$baseName.yes")` with `overwrite = false`; if the destination exists -> throws `IOException` with "File %s is already on the list" (`:94-107`). `baseName` is the lowercased display name minus `.gz` and `.yes|.pdb`. Files up to 100 MiB; `.yes`, `.pdb`, and `.gz` of each are accepted. The error is shown in a dialog; nothing is overwritten. Note the check is **file existence in the yes dir, not a DB lookup** (a file on disk without a DB row also triggers it).
- Local .pdb import: file name becomes `pdb-<name with only [0-9a-z_.-]>.yes`; if that yes exists -> "This file is already on the list" error (`:112-130`); otherwise a conversion options dialog, convert to that yes name, then register (`VersionsActivity.kt:302-389`).
- Preset download: file `<preset_name>.yes` in the yes dir, written via `AtomicFile` (overwrites an existing file), validated by `createYesReader` (invalid -> file deleted, error dialog), then inserted/updated by filename; row gets `preset_name`, `modifyTime` from the preset (`VersionDownloadCompleteReceiver.java:51-153`). Re-download = Update: same filename -> same row updated in place, ordering preserved, `active` reset to true.
- URL download: URL must be http/https and last path segment must end in `.yes` (case-insensitive); dest `getWritableVersionFile(<last segment>)`, overwritten if exists (no existence check), `preset_name = null`, `modifyTime = now/1000` (`VersionsActivity.kt:228-278`; receiver `:75-83`). So a URL download whose filename equals an installed one silently replaces the file and updates the row.
- A locally registered file always gets `preset_name = null` and `modifyTime` unset (0): thus id `file/<absolute path>` and no update notifications (`VersionsActivity.kt:194-212`). Two different files with the same internal `shortName`/`longName` are not detected as duplicates; there is no check on header content.
- Yes dir: `getYesDir()` = `<filesDir>/bible/yes` (`base/util/AddonManager.java:10-16`); a legacy external dir `<external>/bible/yes` is also checked by `getReadableVersionFile` only.
- Which tab is shown after an import: Downloaded (`viewPager.currentItem = 1`, `VersionsActivity.kt:225`).
- `MVersionDb.getVersion()` caches `VersionImpl` per filename in a `SoftReference` map; `clearVersionImplCache()` is called after install (`MVersionDb.java`).

## 4. In-reader version chooser (Fact)

- Entry: `IsiActivity.openVersionsDialog()` (`IsiActivity.kt:1686-1699`). If `S.db.listAllVersions()` is empty (no installed versions at all, regardless of active) it skips the chooser and opens `VersionsActivity` directly. Otherwise calls `VersionDialogHelper.openVersionsDialog(activity, versions, activeSplit0.versionId) { mv -> loadVersion(mv); applyPreferences() }`.
- `VersionDialogHelper` (`base/util/VersionDialogHelper.kt`): a Compose bottom sheet. Rows = `versionManager.getAvailableVersions()` mapped 1:1 (so: internal + active installed with files, in `ordering` order). Each row: radio button (selected where `versionId == current`), bold shortName (fallback longName) 18sp and longName 13sp muted if shortName present. The selected row is scrolled into view if not fully visible. Footer: "More versions" (`versi_lainnya`) which launches `VersionsActivity` and dismisses; and for the split-view variant `openVersionsDialogWithNone` a "None" button (shown only when a version is currently selected).
- Tap row -> callback then `dismiss()`. `IsiActivity.loadVersion(mv)` (`:925-945`): gets `mv.version` (throws -> caught, Unverified handling), keeps the current book id (falls back to `version.firstBook`), calls `setActiveVersion(mv)` which sets a volatile triple (mVersion, Version, versionId) (`S.kt:155-160`).
- The same `getAvailableVersions()` also feeds the daily-verse widget config (`base/appwidget/DailyVerseAppWidgetConfigurationActivity.java:191`).

## 5. Compare dialog (`VersesDialog.newCompareInstance`)

### 5.1 Entry point (Fact)
`VerseActionModeController.kt` menu `menuCompare`: `ari = Ari.encode(activeSplit0Book.bookId, chapter_1, selected.get(0))` (first selected verse only); `VersesDialog.newCompareInstance(ari)` (`base/dialog/VersesDialog.kt:187-193`); listener `onComparedVerseSelected(ari, mversion)` -> `actions.loadVersion(mversion)` then `dialog.dismiss()`; added with `commitAllowingStateLoss` (`base/actionmode/VerseActionModeController.kt:316-331`).

### 5.2 Which Versions and order (Fact, `VersesDialog.kt:137-149`)
- `mversions = getAvailableVersions().toMutableList()` (internal + active, file-present installed; `ordering` order), then `mversions.sortBy { if (it.versionId == sourceVersionId) -1 else 0 }`: the **currently active version is moved to the first row**, others keep ordering relative to each other (Kotlin `sortBy` on a MutableList is a stable sort; Unverified by test, relies on that standard-library guarantee). Comment: "First version must be the sourceVersion."
- Header reference label uses the active version's `reference(ari)` (`:115`).
- One list row per Version; the list is a verses list in `singleClick` mode with verse numbers shown (`:80-99`).

### 5.3 Row content (Fact, `base/dialog/VersesDialogCompareVerses.kt`)
- Row "verse number" text = `version.initials` (see 1.3); text = `version.loadVerseText(ari)`; per-row text-size multiplier = `getPerVersionSettings(mversion.versionId).fontSizeMultiplier`. `Version` objects are loaded lazily per row and memoized in the `displayedVersion` array for the dialog.
- Verse missing in that Version (`loadVerseText` returns null: book absent, chapter absent, or `verse_0 >= verseCount`; `VersionImpl.java:150-171`): the row text is the string `generic_verse_not_available_in_this_version` = "Not available in this version." (`res/values/strings.xml:157`). The row is still shown, in the same position.
- Version fails to open (`mversion.version == null`): text `version_error_opening` "Error opening %s." with the versionId; number text "ERROR" (`getVerseNumberText`).

### 5.4 Tapping a row (Fact, `VersesDialog.kt:159-177`)
Tapping row N (single click): `customCallbackDatas[position]` is `WithMVersion`; **only if** `mversion.version?.loadVerseText(ari) != null` does it call `listener.onComparedVerseSelected(ari, mversion)`. A row for a version missing the verse, or that failed to load, does nothing (the dialog stays open). In the reader, the callback switches the active Version via `loadVersion(mversion)` (keeps current book id, falling back to the version's first book) and dismisses; no `applyPreferences()` call is visible in that path (compare `openVersionsDialog` which does call it) (Unverified whether `loadVersion` applies per-version settings by another route, e.g. via the `activeVersionChanged` event observed at `IsiActivity.kt:727,741`). Tapping the first row (the already-active version) behaves the same as any other row (reloads that same Version).

## 6. Quick answers keyed to the questions

| Question | Answer (all Fact unless marked) |
|---|---|
| id scheme | `internal`; `preset/<preset_name>`; `file/<absolute path>` |
| ordering | int; internal in prefs (default 1), DB rows from 101 up (`max+1`); drag-reorder on Downloaded tab; shared by chooser and Compare |
| active | per-row column; internal always true; chooser/Compare list only active+file-present |
| cannot be deleted | internal only (no Remove, checkbox disabled); all installed (MVersionDb) can, deleting the file too, no confirm |
| duplicates | DB dedupe by `filename`; local yes import refuses if the file name exists; URL/preset downloads overwrite and update the row in place keeping ordering |
| compare picks | `getAvailableVersions()`, active first, rest in ordering |
| compare missing verse | row shown with "Not available in this version.", tap is a no-op |
| compare tap | switches reader to that Version via `loadVersion`, closes dialog |
