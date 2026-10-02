# Upstream user-data export/import file format

Research date: 2026-10-03. Source read: `https://github.com/yukuku/androidbible` at commit `f95ffb89371ddd3b2737f59315b1d61a3df37098` (same pin as `docs/research/yes-file-format.md`), shallow-fetched and read directly. Question: what file does **Upstream** write when the user exports their data, and how does import treat it?

Legend: **Fact** = read from source, cited as `path:Lnn` (repo-relative to the Upstream repo). **Note** = observation about the source, not a recommendation. Findings only.

## 1. Summary

- **Fact.** Upstream has one user-data export/import path, the "Data Transfer" feature (Settings > data transfer, `Alkitab/src/main/res/xml/settings_data_transfer.xml:5-10`; launched from `Alkitab/src/main/java/yuku/alkitab/base/settings/DataTransferFragment.kt:20-27`). It is a single JSON file. The code is under `Alkitab/src/main/java/yuku/alkitab/datatransfer/`.
- It is **not** a **YES file** and carries no **Version** text.
- The same entity model is the one used by Upstream's online sync (entity `KIND_` constants live in `Sync.java`), so the file is effectively a snapshot of the sync entity sets.

## 2. File type and naming

- **Fact.** JSON, UTF-8 (kotlinx.serialization `encodeToStream`), written with `encodeDefaults = false` and `classDiscriminator = "kind"` (`datatransfer/process/Serializer.kt:5-9`, `datatransfer/process/ExportProcess.kt:41-43`).
- **Fact.** Export filename: `<app_name> data <yyyyMMdd'T'HHmmss'Z' in UTC>.export.json`, written into `cacheDir/data_transfer/` (`datatransfer/ui/DataTransferActivity.kt:130-137`). It is handed to the share sheet with MIME `application/octet-stream` via a FileProvider (`DataTransferActivity.kt:158-163`; `Alkitab/src/main/res/xml/file_provider.xml:4-5`).
- **Fact.** Import opens the system document picker filtered to `application/json` and `application/octet-stream` (`DataTransferActivity.kt:118`). The chosen file is copied to a cache file `data-import-tmp-<uuid>.json` first (`DataTransferActivity.kt:66-73`). No extension or name check.
- **Fact.** Import decoding uses `ignoreUnknownKeys = true` (`Serializer.kt:11-14`).

## 3. Structure

**Fact.** Types in `datatransfer/model/JsonFileStructure.kt`:

```
Root            { success: Boolean, snapshots: Snapshots }            :7-11
Snapshots       { history, mabel, pins, rp }  (all four required)     :13-19
Snapshot<E>     { revno: Int = 0, entities: [E] }                     :21-25
```

Every entity has `gid: String` and `creator_id: String` (`:27-31`). Entities in `history`, `pins`, `rp` also carry an explicit `kind: String` field (`:33-36`). Entities in `mabel` are polymorphic: the `kind` discriminator is the class serial name (`Marker`, `Label`, `Marker_Label`; `:55-97`, constants at `Alkitab/src/main/java/yuku/alkitab/base/sync/Sync.java:104-106`).

| Snapshot key | Entity `kind` | `content` | Lines |
|---|---|---|---|
| `history` | `HistoryEntry` | `{ari: Int, timestamp: Long}` | `JsonFileStructure.kt:38-50` |
| `mabel` | `Marker` | `{kind: Int, ari: Int, verseCount: Int, createTime: Int, modifyTime: Int, caption: String}` | `:58-74` |
| `mabel` | `Label` | `{title: String, ordering: Int, backgroundColor: String? = null}` | `:76-89` |
| `mabel` | `Marker_Label` | `{marker_gid: String, label_gid: String}` | `:91-103` |
| `pins` | `Pins` (one entity, gid `g2:pins`) | `{pins: [{ari: Int, modifyTime: Int, preset_id: Int, caption: String? = null}]}` | `:105-124` |
| `rp` | `RpProgress` | `{startTime: Long, done: [Int]}` | `:126-138` |

- **Fact.** Note the two different `kind` meanings inside a `Marker` entity: the entity-level discriminator `"Marker"`, and `content.kind`, an integer **Marker** kind: `bookmark = 1`, `note = 2`, `highlight = 3` (`AlkitabModel/src/main/java/yuku/alkitab/model/Marker.java:18-35`).
- **Fact.** `createTime`, `modifyTime` (**Marker**) and `modifyTime` (pins) are Unix seconds as Int (`Alkitab/src/main/java/yuku/alkitab/base/util/Sqlitil.kt:17-27`, used at `ExportProcess.kt:69-70,92`). History `timestamp` and reading-plan `startTime` are epoch milliseconds Long (`History.kt:70` uses `System.currentTimeMillis()`; `JsonFileStructure.kt:49,136`).
- **Fact.** The `revno` field is a sync revision number. Export writes it as the default `0`, and with `encodeDefaults = false` it is omitted (`ExportProcess.kt:59,86,97,107`; `Serializer.kt:8`). The test fixture, a sync download, has real values such as `"revno": 10` (`Alkitab/src/test/resources/fixtures/download_all.json`, pins snapshot).
- **Fact.** The top-level `success: true` is written by export (`ExportProcess.kt:39`); the field mirrors the sync server's response shape (the fixture `download_all.json` begins `{"snapshots": ...}`, parsed by the same `Root` in `Alkitab/src/test/java/yuku/alkitab/datatransfer/model/JsonFileExportTest.kt:9-16`).
- **Note.** `docs/modules/data-transfer.md:16-34` (Upstream's own doc) shows `mabel.entities` as an object with `markers`/`labels`/`marker_labels` arrays. The code does not match: `mabel.entities` is a flat list discriminated by `kind` (`JsonFileStructure.kt:14-16,55-97`; `ExportProcess.kt:62-87`). The code is authoritative here.

## 4. Versioning

- **Fact.** There is no format version, schema version, magic number or app-version field in the file. The only top-level fields are `success` and `snapshots` (`JsonFileStructure.kt:7-19`); no `version`/`Version` identifier appears anywhere under `datatransfer/` (grep).
- **Fact.** Compatibility relies on `ignoreUnknownKeys = true` on read (`Serializer.kt:11-14`) and on `kind` strings. An unknown `mabel` `kind` has no registered subclass, which kotlinx.serialization rejects by default; the import `when` has no else branch (`ImportProcess.kt:126-168`). REM-33 in Upstream's remediation notes lists "invalid kind enum row" as an untested case (`docs/tech-debt-remediation/REM-33-import-safety.md:15-16`).
- **Fact.** Provenance is per entity via `creator_id`, the exporting installation's id `i1:<uuid>` (`ExportProcess.kt:29`, `DataTransferActivity.kt:125`, `Alkitab/src/main/java/yuku/alkitab/base/util/InstallationUtil.kt:28`).

## 5. Entities carried

| Entity | Carried? | Source |
|---|---|---|
| **Markers** (bookmark, note, highlight) | Yes, `mabel` / `Marker` | `ExportProcess.kt:64-75` |
| **Labels** | Yes, `mabel` / `Label` | `ExportProcess.kt:76-80` |
| **Marker**-**Label** assignments | Yes, `mabel` / `Marker_Label`, linked by `marker_gid`/`label_gid` | `ExportProcess.kt:81-85` |
| **Progress marks** | Yes, as `pins`: one entity holding all pins, keyed by `preset_id` | `ExportProcess.kt:89-98` |
| History (reading history, last up to 20 entries) | Yes, `history` | `ExportProcess.kt:52-60`; cap `History.kt:12,73` |
| Reading plan progress | Yes, `rp`, one entity per plan gid `g2:rp_progress:<plan name>`: `done` reading codes plus `startTime` | `ExportProcess.kt:100-108`; `ReadonlyStorageImpl.kt:34-68`; `Alkitab/src/main/java/yuku/alkitab/base/model/ReadingPlan.java:4,21-23` |
| **Version** text, **YES files**, downloaded Versions | No | not referenced under `datatransfer/` |
| Reading plan definitions (plan data), devotions, songs | No | not referenced; only progress is read (`ReadonlyStorageImpl.kt:38-67`) |
| App settings and preferences | No | not referenced |
| Progress mark change history (`ProgressMarkHistory` table) | No | `ReadonlyStorageImpl.kt:30-32` reads only `listAllProgressMarks()` |

- **Fact.** Reading plan progress for a plan that no longer exists in the `ReadingPlan` table is dropped on export, because `startTimes[gid] ?: continue` skips it (`ReadonlyStorageImpl.kt:45-48`; REM-33 step 4 describes the same).
- **Note.** "Reading plan progress" is not a term in `GLOSSARY.md`; it is named here as Upstream names it (`rp`, `RpProgress`). Upstream's `pins` is the **Progress mark** of `GLOSSARY.md` (`ProgressMark` model: `AlkitabModel/src/main/java/yuku/alkitab/model/ProgressMark.java`).
- **Fact.** Each import choice is a boolean (`importHistory`, `importMabel`, `importPins`, `importRpp`), but the only caller sets all four true; there is no UI to choose a subset (`ImportProcess.kt:28-34`, `DataTransferActivity.kt:175-181`).

## 6. Import: merge or replace

Import first runs a simulation (`actualRun = false`) which logs and then waits for the user to press Start; the second run is the actual one (`DataTransferActivity.kt:171-208`). Both runs execute inside one DB transaction that commits only when `actualRun` (`ImportProcess.kt:50-71`; `Alkitab/src/main/java/yuku/alkitab/base/storage/InternalDbTxWrapper.kt:8-17`).

| Snapshot | Behaviour | Source |
|---|---|---|
| `history` | **Replace** all: local entries are discarded and replaced by the incoming list; no merging. | `ImportProcess.kt:76-89`; `History.kt:78-82` |
| `mabel` | **Merge by `gid`**: for each incoming entity, a local row with the same `gid` is overwritten in place (keeping its local `_id`), otherwise inserted. Nothing local is deleted. Same rule for **Marker**, **Label**, and Marker_Label rows. | `ImportProcess.kt:91-174`; DAO upserts update by `_id`, insert if 0 (`Alkitab/src/main/java/yuku/alkitab/base/storage/MarkerDao.kt:67-74`, `LabelDao.kt:69-76`, `Marker_LabelDao.kt:50-57`) |
| `pins` | **Merge by `preset_id`**: each incoming pin overwrites the local pin with the same `preset_id`, else inserts; local pins with other `preset_id`s remain. Each write also appends a row to the local progress mark history table. Only `pins.entities.firstOrNull()` is read. | `ImportProcess.kt:176-201`; `Alkitab/src/main/java/yuku/alkitab/base/storage/ProgressMarkDao.kt:52-81` |
| `rp` | **Replace per plan gid**: all local progress rows for that gid are deleted and the incoming `done` codes inserted with `checkTime = now`. Plans not in the file are untouched. `startTime` from the file is parsed but not written by `replaceRpp`. | `ImportProcess.kt:203-220`; `datatransfer/process/ReadWriteStorageImpl.kt:54-59`; `ReadingPlanDao.kt:110-130` |

- **Fact.** Imported **Marker**/**Label** values are copied field by field; `Marker.Kind.fromCode(content.kind)` returns `null` for an unknown code (`ImportProcess.kt:134`; `Marker.java:29-34`).
- **Fact.** `Marker_Label` rows are not validated against existing marker/label gids (comment "check validity" at `ImportProcess.kt:157`, no check follows).
- **Fact.** Imported **Markers** do not remove local **Markers** missing from the file, and local **Label** assignments absent from the file stay.
- **Fact (side effects).** History: `replaceAllEntries` only mutates the in-memory list; it does not call `History.save()` (`History.kt:46-56,78-82`; `ReadWriteStorageImpl.kt:26-28`). This also happens during the simulation run (`ImportProcess.kt:51-53` is not gated by `actualRun`). REM-33 documents both (`docs/tech-debt-remediation/REM-33-import-safety.md:14-16,23`).
- **Fact.** Each DB write calls `Sync.notifySyncNeeded(...)` (`InternalDb.java:93-96,560-567,699-702,725-728`).

## 7. ARI encoding

- **Fact.** Every address is the bare integer **ARI**, serialised as a JSON number: `HistoryContent.ari`, `MarkerContent.ari`, `Pin.ari` are `Int` (`JsonFileStructure.kt:47-48,67-68,119-120`). No string form, no hex, no book name.
- **Fact.** Layout, from `AlkitabModel/src/main/java/yuku/alkitab/util/Ari.java:7-16`: bits 31..24 zero, bits 23..16 book id (0 = Genesis, 65 = Revelation, 66+ defined elsewhere), bits 15..8 chapter (1-based, 0 = whole book), bits 7..0 verse (1-based, 0 = whole chapter). `encode(book, chapter, verse) = (book & 0xff) << 16 | (chapter & 0xff) << 8 | (verse & 0xff)`.
- **Fact (example).** Fixture pin `"ari": 592386` = `0x090102`, book 9, chapter 1, verse 2 (`Alkitab/src/test/resources/fixtures/download_all.json`, pins snapshot; arithmetic mine).
- **Fact.** `verseCount` on a **Marker** is the number of consecutive verses starting at `ari` (field at `Marker.java:14`, exported at `ExportProcess.kt:68`).
- **Fact.** Reading plan `done` values are "reading codes" (`IntArrayList`), not ARIs (`ReadWriteStorageImpl.kt:55-58`).

## 8. Version ids

- **Fact.** The file contains no **Version** id, no **YES file** reference, and no book-name table. `MarkerContent`, `HistoryContent` and `Pin` have no version field (`JsonFileStructure.kt:47-50,66-74,118-124`), and the `Marker` model has none either (`Marker.java:9-16`). Upstream's markers, history and pins therefore address verses by **ARI** alone, independent of which **Version** is open.
- **Fact (the only ids that exist in the file).** All are opaque strings:
  - entity `gid`s generated as `"g1:" + UUID` for **Markers**, **Labels**, Marker_Label rows and history entries (`AlkitabModel/src/main/java/yuku/alkitab/model/util/Gid.java:6-8`; `Marker.java:44`, `Label.java:25`, `Marker_Label.java:23`, `History.kt:70`);
  - fixed `g2:pins` for the pins entity (`Alkitab/src/main/java/yuku/alkitab/base/sync/Sync_Pins.java:29`);
  - `g2:rp_progress:<reading plan name>` for reading plan progress (`ReadingPlan.java:4,21-23`);
  - `creator_id` = `i1:<uuid>` installation id (`InstallationUtil.kt:28`).
- **Fact.** Pin `preset_id` is a 0-based slot index (the UI logs it as "Pin N+1", `ImportProcess.kt:184,189`); the slot count is `AttributeView.PROGRESS_MARK_TOTAL_COUNT` (loop at `Alkitab/src/main/java/yuku/alkitab/base/verses/VerseItem.kt:299`).

## 9. Not found

- No other user-data export/import path exists: a grep for `backup`/`export` across the main source tree finds the Data Transfer code, Android auto-backup preference plumbing (`NoBackupSharedPreferences.java`, `Prefkey.kt`, `InstallationUtil.kt`), and the sync classes (`Fcm.java`), and nothing that writes another file format.
- No unit test exercises `ExportProcess`/`ImportProcess` directly; the only test under `datatransfer/` parses the sync fixture (`JsonFileExportTest.kt:9-16`).
- Not read: the online sync server protocol (`docs/backend-communication.md`), which is **Online features** and out of scope here. The export and the sync downloads share the `Root`/`Snapshots` types (`JsonFileExportTest.kt:15`).
