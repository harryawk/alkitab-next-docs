# Upstream Markers and Labels: data model and behaviour

Research date: 2026-10-03. Source read: **Upstream** `https://github.com/yukuku/androidbible` at commit `f95ffb89371ddd3b2737f59315b1d61a3df37098` (the same pin as `yes-file-format.md`), cloned and read directly. Findings only; no recommendations.

Convention: matches `docs/research/` (see `yes-file-format.md`). Terms follow `GLOSSARY.md`: **Marker**, **Label**, **ARI**, **Version**, **Progress mark**, **Online features**.

Legend: **Fact** = read from source, cited as `path:Lnn` (repo-root-relative; permalink form is `https://github.com/yukuku/androidbible/blob/f95ffb89371ddd3b2737f59315b1d61a3df37098/<path>#L<n>`). **Derivation** = my reading or inference, not a quote of the code. Where the Upstream's own `docs/modules/markers.md` disagrees with the code, the code wins (disagreements are in section 11).

Not run: no Android build or test was executed; everything here is source reading. The Upstream sync server is not in the repo, so server-side merge behaviour is out of scope except where the client quotes it.

---

## 0. Where the code lives

| Concern | Files |
|---|---|
| Models | `AlkitabModel/src/main/java/yuku/alkitab/model/Marker.java`, `Label.java`, `Marker_Label.java`, `util/Gid.java` |
| Table names and columns | `Alkitab/src/main/java/yuku/alkitab/base/storage/Db.java` |
| DDL, indexes, migrations | `.../storage/InternalDbHelper.java` |
| Raw table access | `.../storage/MarkerDao.kt`, `LabelDao.kt`, `Marker_LabelDao.kt` |
| Orchestration (transactions, highlights, label reorder) | `.../storage/InternalDb.java` |
| Highlight caption codec and colour blending | `.../base/util/Highlights.kt` |
| Label colour codec | `.../base/util/LabelColorUtil.kt` |
| Sync of Markers and Labels | `.../base/sync/Sync_Mabel.java`, `Sync.java`, `SyncApplier.kt`, `SyncAdapter.java`, `SyncUtils.kt` |
| File export/import | `.../datatransfer/process/ExportProcess.kt`, `ImportProcess.kt`, `.../datatransfer/model/JsonFileStructure.kt` |
| UI | `.../base/actionmode/VerseActionModeController.kt`, `dialog/TypeBookmarkDialog.java`, `ac/NoteActivity.java`, `dialog/TypeHighlightDialog.kt`, `ac/MarkersActivity.java`, `ac/MarkerListActivity.kt`, `dialog/LabelEditorDialog.kt`, `widget/AttributeView.java`, `verses/VerseAttributeLoader.kt` |

All paths below are under `Alkitab/src/main/java/yuku/alkitab/` unless they start with `AlkitabModel/` or `docs/`.

---

## 1. Table shapes

Three tables in the app's internal SQLite database. All three have an `_id integer primary key autoincrement` plus a text `gid`.

### 1.1 `Marker`

Fact: DDL at `base/storage/InternalDbHelper.java:184-197`; column-name constants at `base/storage/Db.java:4-13`; model at `AlkitabModel/.../model/Marker.java:7-16`.

| Column | SQLite type | Java type | Meaning |
|---|---|---|---|
| `_id` | integer pk autoincrement | long | local row id only |
| `gid` | text | String | global id for sync (section 3.1) |
| `ari` | integer | int | **ARI** of the first verse covered |
| `kind` | integer | enum `Marker.Kind` | 1 bookmark, 2 note, 3 highlight |
| `caption` | text | String | meaning depends on kind (section 2) |
| `verseCount` | integer | int | number of verses covered, starting at `ari` |
| `createTime` | integer | Date | Unix seconds |
| `modifyTime` | integer | Date | Unix seconds |

Indexes (`InternalDbHelper.java:199-206`): `(ari)`; `(kind, ari)`; `(kind, modifyTime)`; `(kind, createTime)`; `(kind, caption collate NOCASE)`; `(gid)`. None is unique, so `gid` uniqueness on `Marker` is not enforced by the schema.

Kind codes: `AlkitabModel/.../model/Marker.java:18-35` (`bookmark(1)`, `note(2)`, `highlight(3)`; `fromCode` returns null for an unknown code).

Time encoding: `base/util/Sqlitil.kt:9-27`. `Date` to int is `(date.time / 1000).toInt()`; int to `Date` is `Date(int.toLong() * 1000)`. Seconds, signed 32-bit, no timezone.

### 1.2 `Label`

Fact: DDL at `InternalDbHelper.java:311-320`; model at `AlkitabModel/.../model/Label.java:5-10`.

| Column (SQL name) | Java field | Type | Meaning |
|---|---|---|---|
| `_id` | `_id` | integer pk autoincrement | local id |
| `gid` | `gid` | text | global id |
| `judul` | `title` | text | user-visible name |
| `urutan` | `ordering` | integer | sort position, ascending |
| `warnaLatar` | `backgroundColor` | text, nullable | encoded background colour (section 8) |

The SQL column names are Indonesian (`judul`, `urutan`, `warnaLatar`), while the Java field names are English: `base/storage/Db.java:28-34`. Indexes: `(ordering)` and `(gid)`, `InternalDbHelper.java:322-325`. The `Label` model has no timestamps.

### 1.3 `Marker_Label` (junction)

Fact: DDL at `InternalDbHelper.java:327-335`; model at `AlkitabModel/.../model/Marker_Label.java:5-9`; column names at `Db.java:36-41`.

| Column | Type | Meaning |
|---|---|---|
| `_id` | integer pk autoincrement | local id |
| `gid` | text | global id of the association itself |
| `marker_gid` | text | `Marker.gid` (not `_id`) |
| `label_gid` | text | `Label.gid` (not `_id`) |

Indexes (`InternalDbHelper.java:337-342`): `(marker_gid)`, `(label_gid)`, and a unique index on `(gid)` only. A unique index on `(marker_gid, label_gid)` existed once and is dropped on upgrade for versionCode below 14000172 (`InternalDbHelper.java:159-161`, comment at `:340`), so nothing in the schema stops two rows linking the same Marker to the same Label. No foreign keys are declared anywhere; links are by `gid` string.

Because the association is a row with its own `gid`, it is a separately synced entity (section 3).

### 1.4 Legacy

Fact: tables were introduced at the 4.0.0-beta1 upgrade step (`InternalDbHelper.java:131-140`), converting an older `Bukmak2` table (columns `jenis`, `tulisan`, `waktuTambah`, `waktuUbah`) and `Bukmak2_Label` into `Marker` and `Marker_Label`, giving every converted row a new `gid` and `verseCount` 1 (`InternalDbHelper.java:473-516`, `:549-567`).

---

## 2. Marker kinds and what `caption` holds

| Kind | `caption` | `verseCount` as written by the UI | Source |
|---|---|---|---|
| bookmark (1) | free-text title. Defaults to the reference string (for example "Genesis 1:1–3"); a blank title is replaced by that default on save | number of contiguous selected verses | `base/dialog/TypeBookmarkDialog.java:97, 108-123`; `base/actionmode/VerseActionModeController.kt:334-353` |
| note (2) | the full note text | number of contiguous selected verses | `base/ac/NoteActivity.java:289-316`; `VerseActionModeController.kt:356-367` |
| highlight (3) | an encoded colour (section 7); never user-visible text | always 1 (one Marker per verse) | `base/storage/InternalDb.java:245, 294` |

Fact: the bookmark and note menu items are shown only when the selected verses are contiguous (`VerseActionModeController.kt:95-112`), and creation throws if they are not (`:336-339`, `:358-361`). The new Marker's `ari` is the first selected verse and `verseCount` is `selected.size()`. Highlight has no such restriction (`menuAddHighlight` handler at `:373-394`).

Fact: the reference text for a range is `reference(book, chapter, verse)` plus an en dash and the last verse number; when the verse is 0 or `verseCount == 1` it is just the single reference (`AlkitabModel/.../model/Version.java:100-110`). A range is therefore always inside one chapter.

ARI (Fact, `AlkitabModel/src/main/java/yuku/alkitab/util/Ari.java:10-26, 31-55`): `bits 23..16` book (0 is Genesis), `bits 15..8` chapter (1-based, 0 means whole book), `bits 7..0` verse (1-based, 0 means whole chapter); bits 31..24 are 0. `encodeWithBc(ari_bc, verse)` keeps the book and chapter bits and replaces the verse.

A Marker stores only a start ARI and a count. There is no end ARI field.

### 2.1 Per-chapter attribute derivation (counts and colours)

Fact: `base/storage/InternalDb.java:166-215` (`putAttributes`), driven by `base/verses/VerseAttributeLoader.kt:23-30`.

- Query: all Markers whose `ari` is within `[ari_bc & 0x00ffff00, ari_bc | 0xff]`, inclusive on both ends, `order by modifyTime` ascending (`InternalDb.java:177-180`; range comment `:178-179`; chapter counting helper `:158-162`, `MarkerDao.kt:96-105`).
- Bookmark: increments `bookmarkCountMap[verse-1]` by 1 at the start verse only. It does not spread over `verseCount` (`InternalDb.java:196-197`).
- Note: increments `noteCountMap[verse-1]` by 1 at the start verse only (`:198-199`).
- Highlight: for `i in 0 until verseCount`, writes the decoded `Highlights.Info` into `highlightColorMap[verse-1+i]`, stopping at the end of the chapter (`:200-212`). Because the rows are read oldest-first, the latest highlight on a verse wins (`:177`).
- A verse offset beyond the chapter's verse count is logged and skipped (`:191-194`).

Derivation: counts (not booleans) are tracked for bookmark and note, so a verse can carry several of each; `widget/AttributeView.java:201-206` draws the number when the bookmark count is above 1.

Fact: `getHighlightColorRgb(ari_bc, selectedVerses)` (`InternalDb.java:313-352`) returns the single common colour of the selected verses, or `-1` if they differ or none has one; it reads `ari > ariMin` (exclusive) and `ari <= ariMax`, ignores `verseCount`, and reads `info.colorRgb` without a null check at `:335-336` although `Highlights.decode` can return null (`Highlights.kt:79-102`). `anyVerseHasHighlight` (`InternalDb.java:361-382`) is the separate "does at least one verse have one" test.

---

## 3. Sync-ready fields: gid, timestamps, add/modify/delete

### 3.1 `gid`

Fact: `AlkitabModel/.../model/util/Gid.java:5-9`: `"g1:" + UUID.randomUUID().toString()`. Every new Marker, Label and Marker_Label calls `Gid.newGid()` in its factory (`Marker.java:42-44`, `Label.java:22-25`, `Marker_Label.java:20-23`). Entities delivered by the server carry a `creator_id` in the form `i1:<uuid>` (fixture `Alkitab/src/test/resources/fixtures/download_all.json:720-724` area; `Sync.java:64`, `datatransfer/model/JsonFileStructure.kt:28-31`).

### 3.2 Timestamps

Fact: only `Marker` has them: `createTime` and `modifyTime`, both Unix seconds, both sent in sync content (`Sync_Mabel.java:87-88`, `:171-172`).

- Creation sets both to the same `now` (`TypeBookmarkDialog.java:116, 122`; `NoteActivity.java:292, 308`; `InternalDb.java:245, 294`).
- Editing a bookmark sets `modifyTime = now` on every OK, even if only labels changed (`TypeBookmarkDialog.java:116-125`: `modifyTime` is set at `:119` before `updateLabels` at `:125`).
- Editing a note sets `modifyTime` only when the text changed (`NoteActivity.java:294-305`).
- Re-colouring a highlight sets `modifyTime` (`InternalDb.java:234, 274`).
- `Label` and `Marker_Label` have no timestamps; label rename, colour change and reorder leave no time trace (`MarkersActivity.java:203-241`; `InternalDb.java:599-632`).

### 3.3 Add, modify, delete

Fact: there is no `deleted` or `dirty` column and no per-row version. Add, mod and del are not stored; they are computed at sync time by diffing current rows against a stored snapshot.

- Operation enum: `base/sync/Sync.java:37-39` (`add`, `mod`, `del`; names are gson-serialised).
- Operation shape: `{opkind, kind, gid, content, creator_id}` (`Sync.java:59-67`). `kind` is one of `"Marker"`, `"Label"`, `"Marker_Label"` (`Sync.java:103-110`). An entity is identified by `(gid, kind)` (`base/sync/SyncUtils.kt:13-20`).
- The three types are one sync set named `"mabel"` (`base/model/SyncShadow.java:5`; `Sync_Mabel.java:77-119` builds all three entity lists together).
- Snapshot: the `SyncShadow` row for `mabel` holds the last-synced entity list as JSON plus `revno` (`Sync_Mabel.java:57-75`).
- Diff (`Sync_Mabel.java:25-55`): a current entity absent from the snapshot gives `add`; one present but with different content gives `mod` (content equality over all fields, `Sync_Mabel.java:206-226`, `SyncUtils.kt:6-10`); a snapshot entity absent from the current rows gives `del` with null content.
- Content fields by kind (`Sync_Mabel.java:177-203`, `datatransfer/model/JsonFileStructure.kt:66-103`):
  - Marker: `ari, kind, caption, verseCount, createTime, modifyTime`
  - Label: `title, ordering, backgroundColor` (nullable)
  - Marker_Label: `marker_gid, label_gid`
- Every local write to these tables calls `Sync.notifySyncNeeded(SyncShadow.SYNC_SET_MABEL)` (all `InternalDb` methods in `InternalDb.java:72-632` and `:809-840`). It does nothing unless a sync account token is set (`Sync.java:208-212`).
- Applying a server delta (`base/sync/SyncApplier.kt:30-97`): `del` deletes by `gid` for each kind (`:57-62`); `add` and `mod` are handled identically: look up by `gid`, overwrite every field from the content, or create (`:63-80`; the `update*WithEntityContent` helpers say "will not merge content, will only overwrite", `Sync_Mabel.java:121-175`). The whole apply runs in one transaction and is rejected with `dirty_entities` if the current rows changed since the request began (`SyncApplier.kt:42-46`) or `dirty_sync_account` if the account changed (`:48-52`). The shadow is then patched with both the client delta and the append delta (`:84-89`; `SyncAdapter.java:65-104`).
- Large client deltas are chopped to 100 operations per request (`SyncAdapter.java:45, 196-240`).
- Server-side conflict handling is not in the repo. The client's `patchNoConflict` quotes the server's version of the same rule: `del` removes, `add` and `mod` overwrite (`SyncAdapter.java:66-84`). Upstream's `docs/modules/sync.md:43-47` says "server wins" for local changes; the client code read above does not itself resolve conflicts.
- `ImportProcess` (file import from the data-transfer screen) follows the same shape: it matches by `gid`, replaces if found, inserts otherwise, and does not delete (`datatransfer/process/ImportProcess.kt:92-173`; storage calls `ReadWriteStorageImpl.kt:22-42` go through `insertOrUpdate*`, which also fires `notifySyncNeeded`). Export writes all three entity kinds with the same content fields (`datatransfer/process/ExportProcess.kt:62-86`).

Online features (sync) are parked per `GLOSSARY.md`; the above is recorded because the field set and delete behaviour come from this machinery.

---

## 4. Label ordering

Fact:

- `ordering` is an `int`, listed ascending (`base/storage/LabelDao.kt:21-30`). `Label.compareTo` is `this.ordering - another.ordering` (`AlkitabModel/.../model/Label.java:14-17`). `Label` has no `equals` or `hashCode`.
- New label: `ordering = getMaxOrdering() + 1` (`LabelDao.kt:32-37, 44-48`). Derivation: the max over an empty table is SQL NULL, and the code reads it as `simpleQueryForLong().toInt()`; I did not verify what Android returns for NULL there, so the first label's number is unconfirmed. The numbers need not be contiguous.
- Drag reorder: moves one label to the position of another and shifts the labels between by one. Up: `ordering += 1` for `to <= ordering < from`, then set the moved label to `to`. Down: `ordering -= 1` for `from < ordering <= to`, then set it to `to` (`InternalDb.java:599-632`, explanatory comment `:600-613`). One transaction.
- Alphabetical sort: sorts by `title.compareToIgnoreCase` (a null title compares as equal) then rewrites `ordering = 1..n` and upserts each (`InternalDb.java:574-593`).
- Labels attached to a Marker are returned in `ordering` ascending (`LabelDao.kt:90-103`).
- Derivation: since `ordering` is part of the synced Label content, a reorder or alphabetical sort produces a `mod` for every Label whose number changed.
- Derivation: the bookmark dialog keeps its labels in a `TreeSet<Label>` (`TypeBookmarkDialog.java:52`); that set compares by `ordering`, so two labels with equal `ordering` (which sync can produce) would collapse into one entry in that dialog.
- Marker_Label rows have no ordering.

Title rules, from the editor dialog (`base/dialog/LabelEditorDialog.kt:16, 43-47`): trimmed, non-empty, at most 48 characters, and not equal (case-sensitive `==` on trimmed text) to an existing label title. The database itself has no uniqueness or length constraint on `judul`.

---

## 5. Label colours

Fact (`base/util/LabelColorUtil.kt:10-34`):

- Stored string: `"b"` + 6 lowercase hex digits `rrggbb`, zero-padded, no alpha (`encodeBackground`). Example in the sync fixture: `"b4826d6"` (`download_all.json:2454`).
- Decode returns `-1` for null, empty, a string shorter than 7 characters, or a first character other than `b` (`decodeBackground`). The fixture contains one empty-string value and DDL allows null, and the label chooser creates labels with a null colour (`TypeBookmarkDialog.java:251`).
- Default when decode fails: `0x212121` (grey 900) (`LabelColorUtil.kt:52-55`).
- Foreground text colour is not stored. It is computed from the background in HSL: lightness minus 0.44 if above 0.5, otherwise plus 0.44 (`LabelColorUtil.kt:36-48`).
- Colour is set from "Change label colour" on the label's long-press menu, through a free-form colour picker (`base/ac/MarkersActivity.java:234-241`); the stored value is masked to `0x00ffffff` before encoding (`:237`). There is no fixed Label palette. When the Label has no valid colour, the picker opens on `0xff000000 | -1`, which is `0xffffffff` (`:235-236`, derivation from the arithmetic).
- Labels are drawn as a coloured chip (`LabelColorUtil.apply`, `LabelColorUtil.kt:50-75`) in the label list, the bookmark dialog, the bookmark list and the multi-select popup (`MarkersActivity.java:456`, `TypeBookmarkDialog.java:197`, `ac/MarkerListActivity.kt:755-761`, `base/IsiActivity.kt:2033-2038`).

---

## 6. Label to Marker relation

Fact:

- Many-to-many through `Marker_Label`, joined by `gid` strings (`LabelDao.kt:90-103`, `Marker_LabelDao.kt:27-37`).
- Setting a Marker's labels diffs the old and new sets and inserts or deletes whole `Marker_Label` rows, never updating them; it also deletes surplus duplicate rows for the same Label (`InternalDb.java:469-527`, duplicate handling `:501-506`).
- The data model, sync and import accept a `Marker_Label` for a Marker of any kind (`SyncApplier.kt:74-78`, `ImportProcess.kt:156-166`). The UI only offers label assignment for bookmarks (section 10).
- "Unlabelled bookmarks" query: bookmarks whose `gid` is not in `select distinct marker_gid from Marker_Label` (`InternalDb.java:133-134`). Derivation: the subquery does not check that the Label still exists, so a Marker whose only link points to a deleted Label would not appear under "Unlabelled bookmarks" and would show no label chip.
- Label usage count: `count(*)` of `Marker_Label` rows for the label's `gid` (`Marker_LabelDao.kt:79-83`).

---

## 7. Highlights: colour and range

### 7.1 Caption format

Fact (`base/util/Highlights.kt:26-102`):

- Current encoding is JSON via a plain `Gson`: `{"colorRgb":<int>}` for a whole verse, or `{"colorRgb":<int>,"partial":{"hashCode":<int>,"startOffset":<int>,"endOffset":<int>}}` for a part of a verse (`encode` at `:47-72`; class `Info` fields at `:27-38`). `colorRgb` is a decimal int of `0xRRGGBB` with no alpha (comment at `:28`). Gson omits nulls, so a whole-verse caption has no `partial` key (`App.java:38-42` is a bare `new Gson()`; fixture example `download_all.json:728`: `"{\"colorRgb\":16744448}"`).
- Legacy decoding: a caption of at least 7 characters starting with `c` is read as `c` + 6 hex digits (`decode`, `:79-91`). The sync fixture contains both forms (for example `"c0080ff"` at `download_all.json:715` next to the JSON form).
- A caption that is neither returns null from `decode` (`:99-100`) or on a JSON syntax error (`:92-98`).
- `partial.hashCode` is Java's `String.hashCode` of the verse's plain text (`Highlights.kt:195-198`), and a partial highlight is honoured at render time only if the hash still equals the current text's hash and both offsets are within the text length (`shouldRenderAsPartialForVerseText`, `:40-43`). Otherwise the whole verse body is highlighted (`base/widget/VerseRenderer.kt:246-257`; Compose path `VerseRendererCompose.kt:272-283`).
- Offsets are character offsets into the plain verse text (after Verse markup is removed), start and end normalised by min and max at render and save (`VerseRenderer.kt:250-256`, `TypeHighlightDialog.kt:167-172`).

### 7.2 Palette and picking

Fact (`base/dialog/TypeHighlightDialog.kt:192-197`): 12 preset RGB colours in swatch order: `0xff0000, 0xff8000, 0xffff00, 0x80ff00, 0x00ff00, 0x00ff80, 0x00ffff, 0x0080ff, 0x0000ff, 0x8000ff, 0xff00ff, 0xff0080`. A free-form colour picker is also reachable from the sheet (`:104-113`, via `ColorPickerDialog`); the result is masked to 24 bits.

The stored colour is not the displayed colour. At draw time `Highlights.blendOver(colorRgb, backgroundColor)` computes an alpha so the band sits about 0.22 Oklab distance from the current background, floored at 0.30 sRGB distance (`Highlights.kt:14-18, 104-145`). It is therefore recalculated per theme and the alpha is never stored.

### 7.3 Writing a highlight (one Marker per verse)

Fact (`base/storage/InternalDb.java:219-304`):

- Selecting several verses and picking a colour runs `updateOrInsertHighlights` over each selected verse separately: find the highlight Markers at that exact `ari` ordered `modifyTime desc`; if one exists, update its caption and `modifyTime` (or delete it when the colour is -1); delete any older extras at the same `ari` ("caused by sync", `:241-242, 284-288`); if none exists and the colour is not -1, insert a Marker with `verseCount = 1` and `createTime = modifyTime = now`.
- A partial highlight (single verse only, with the range not covering the whole text and not empty) goes through `updateOrInsertPartialHighlight`, same find-update-delete-extras-or-insert pattern with the partial caption (`:221-255`; chosen at `TypeHighlightDialog.kt:155-182`).
- Each is wrapped in one transaction and ends with a sync notification.
- A verse therefore carries at most one highlight Marker through the UI; duplicates can only arise from sync or import and are cleaned on the next local write to that verse.

---

## 8. Delete semantics

Fact: all deletion is a hard `DELETE` of the row. There is no soft-delete flag, tombstone table or trash.

| Action | What is removed | Source |
|---|---|---|
| Bookmark dialog "Delete" (asks for confirmation) | the Marker's `Marker_Label` rows, then the Marker, in one transaction | `TypeBookmarkDialog.java:158-175`; `InternalDb.java:104-118` (`deleteMarkerById`) |
| Marker list, edit-mode "Delete" (any kind; one item selected; no confirmation) | same as above | `ac/MarkerListActivity.kt:510-519` |
| Note editor "Delete" (confirms when the Marker exists or the buffer has text) | the Marker row only (`deleteNonBookmarkMarkerById`); its `Marker_Label` rows, if any, are not touched | `NoteActivity.java:257-280`; `InternalDb.java:120-123` |
| Note saved with an empty text on an existing note | same as above, instead of an update | `NoteActivity.java:298-299` |
| Note saved with an empty text when new | nothing is inserted | `NoteActivity.java:306-310` |
| Highlight "remove" (colour -1) | each selected verse's highlight Marker, plus any older duplicates at that `ari` | `InternalDb.java:279-281, 284-288`; `TypeHighlightDialog.kt:121-125` |
| Remove a Label chip from a bookmark in the dialog (confirms) | on OK, the `Marker_Label` row (a new `Marker_Label` row is created if re-added later) | `TypeBookmarkDialog.java:138-156`; `InternalDb.java:514-516` |
| Delete a Label | its `Marker_Label` rows, then the Label, in one transaction; Markers survive; confirmation names the Marker count, skipped when the count is 0 | `MarkersActivity.java:212-232`; `InternalDb.java:546-558` |
| Server delta `del` | exactly the row named by `gid`; no cascade, so related rows arrive as their own `del` operations | `SyncApplier.kt:57-62`; `InternalDb.java:809-840` |

Fact: no code path read deletes a Marker because its verse is missing from the current **Version**; the copy-to-clipboard dump substitutes a "verse not available in this version" string (`MarkerListActivity.kt:540-546`) and the highlight editor passes an empty verse text (`:484-497`).

---

## 9. Reading and list behaviour (queries)

Fact:

- `listMarkersForAriKind(ari, kind)`: exact `ari`, one kind, `modifyTime desc` (`MarkerDao.kt:43-54`). Used when a verse's bookmark or note icon is tapped.
- `listMarkers(kind, label_id, sortColumn, ascending)`: `label_id == 0` all of the kind; `LABELID_noLabel (-1)` those without any `Marker_Label`; otherwise those linked to that Label; caption sorts use `collate NOCASE` (`InternalDb.java:125-148`; constant `MarkerListActivity.kt:745`).

---

## 10. UI surfaces Upstream offers

### 10.1 Creating (reader screen)

Fact: long-press verse selection opens a contextual menu with one "Add" group of three items: Add bookmark, Add note, Highlight (`res/menu/context_isi.xml:65-85`). Handlers in `base/actionmode/VerseActionModeController.kt`:

- Add bookmark `:334-353` opens `TypeBookmarkDialog.NewBookmark` (always creates a new Marker, never edits an existing one).
- Add note `:356-368` opens `NoteActivity` with a reference string, `ari` and `verseCount` for a new note.
- Highlight `:373-394` opens `TypeHighlightDialog`: for one verse it passes the current verse text and existing partial info; for several verses it passes only the shared colour (or -1).
- After bookmark or highlight, the selection is cleared and the attribute maps reload (`:344-349`, `:378-381`).

### 10.2 Indicators and tapping them

Fact: per-verse icons in the gutter (`base/widget/AttributeView.java`): bookmark (with a count when above 1, `:201-206`), note, Progress mark, map. Highlights are not an icon but a coloured background on the verse text (section 7). Tap handlers exist for bookmark, note, Progress mark and map (`base/verses/VersesController.kt:24-30`; `AttributeView.java:247-275`; Compose `VerseItemCompose.kt:886-933`); there is none for highlight. In `base/IsiActivity.kt:1948-2067`: one matching Marker opens its editor directly (bookmark dialog or note editor); several open a chooser list showing each Marker's dates, caption or note text, and for bookmarks their label chips.

### 10.3 Bookmark dialog (`base/dialog/TypeBookmarkDialog.java`)

Title is the reference; a single caption field; label chips with tap-to-remove (confirm) and an "Add label" button that opens a chooser of existing labels in `ordering` order plus a "Create label…" entry (`:215-270`). New labels from here are inserted with a null colour (`:251`). Buttons: OK, Delete (neutral). OK saves caption, `modifyTime`, then the label set (`:108-128`).

### 10.4 Note editor (`base/ac/NoteActivity.java`)

A full-screen activity. Existing notes open read-only and switch to editing by menu or by tapping the text, unless the preference "Tap to edit note" (`pref_tapToEditNote`, default true, `res/values/pref_defaults.xml:17`) is off (`:112-121, 214-224`). In read mode, verse references in the text are detected with `DesktopVerseFinder` and become tappable links that open a verse list dialog and then jump to the verse (`:147-174, 187-200`). Menu: Edit, Delete, OK (`res/menu/activity_note.xml`). Saving happens on OK and also on leaving (`finish()` calls `ok_click()`, `:318-323`). The note editor has no Label UI (no Label references in `NoteActivity.java`).

### 10.5 Highlight sheet (`base/dialog/TypeHighlightDialog.kt`)

A Compose bottom sheet with the reference as title, the 12 swatches, a custom colour picker entry, a delete action (the sheet is told whether any selected verse already has a highlight, `:75-76`), and, for a single verse, a partial-range edit whose OK commits only when the range changed (`:75-135`, `:155-182`). The sheet's composable body (`HighlightSheetContent`, from `:200` on) was not read in full, so its exact layout is not described here.

### 10.6 Markers hub (`base/ac/MarkersActivity.java`)

Opened from the left drawer's markers button (`base/widget/LeftDrawer.java:349-351`; `IsiActivity.kt:2295-2296`). A list with four fixed rows then the user's Labels (`:391-460`):

1. All bookmarks
2. Notes
3. Highlights
4. Unlabelled bookmarks (shown only when at least one Label exists: the row count is `3 + (hasLabels ? 1 + labels.size : 0)`, `:421-424`)
5. Then each Label as a coloured chip with a drag handle

Taps open `MarkerListActivity` filtered by kind (rows 1 to 4) or by bookmarks with that Label (`:154-172`). Label rows: drag handle reorder (rows below the fixed four only, `:258-336`), long-press popup with Change colour, Rename, Delete (`:174-246`, `res/menu/context_markers.xml`), and an overflow item "Sort labels alphabetically", shown only when more than one Label exists (`:132-152`, `res/menu/activity_markers.xml`). There is no "create Label" control in this screen; Labels are created from the bookmark dialog (section 10.3) (derivation from `activity_markers.xml` and the activity code read in full). A footer panel offering to set up sync is shown when no sync account is configured (`:86-124`).

### 10.7 Marker list (`base/ac/MarkerListActivity.kt`)

- One list per kind (and per Label for bookmarks); titles per case at `:177-208`.
- Search box with a 200 ms debounce; text is tokenised and matched against the caption (not for highlights, "caption in highlights only stores color information") and against the verse text in the current **Version**, case-insensitively (`:232-266`, `:769-795`).
- Sort options: Bible order (`ari`, ascending), caption (ascending; labelled "color" for highlights), date created (descending), date modified (descending); the choice and direction persist in preferences (`:122-146`, `:299-363`, `Prefkey.marker_list_sort_column`). Default is `createTime` descending (`:87-91, 123-124`). Derivation: sorting highlights "by colour" is a string sort of the caption JSON text, not of hue.
- Row: reference with date, plus snippet by kind (bookmark: caption and verse text, then Label chips; note: note text; highlight: the verse text painted with its colour band and partial range) (`:595-725`).
- Tap opens the reader at the verse range (`:367-377`). Long-press starts a multi-select mode with Select all, Copy, Edit (one item), Delete (one item) (`res/menu/context_marker_list.xml`, `:400-535`). Edit opens the kind's own editor (bookmark dialog, note editor or highlight sheet). Copy writes a text dump: bookmark = caption, reference if different, verse text, "..." if more than one verse, then comma-joined Label titles; note = reference and note text; highlight = reference and verse text (`:540-580`).
- Reloads on `AppEvents.markerListReload` (`:156-158`; `base/events/AppEvents.kt:64-68, 95-96`).

### 10.8 Backup and transfer

Fact: the data-transfer screen (`datatransfer/ui/DataTransferActivity.kt`) exports and imports the three entity kinds in the JSON structure of section 3.3 (`datatransfer/process/ExportProcess.kt:62-86`, `ImportProcess.kt:92-173`). Import logs counts of new versus replaced Markers, Labels and associations. Entity wrapper adds `creator_id` per entity (`JsonFileStructure.kt:28-31, 58-103`).

---

## 11. Disagreements between Upstream docs and code

Fact:

- `docs/modules/markers.md:25` says `kind: int — 0=bookmark, 1=note, 2=highlight`. The code is 1, 2, 3 (`AlkitabModel/.../model/Marker.java:19-21`); the fixture agrees with the code (`"kind": 3` on highlights).
- `docs/modules/markers.md:46` describes `Highlights.alphaMix()` with a fixed `0xA0` alpha. No `alphaMix` exists at this commit; the function is `blendOver` (`Highlights.kt:117`; `git grep alphaMix` finds only docs).
- `docs/modules/markers.md:37` says Labels organise "bookmarks"; the schema allows any Marker kind, the UI offers it for bookmarks only.
- `docs/modules/markers.md:11` says `MarkersActivity` handles "create" of Labels; the code has no create control there (section 10.6).
- `docs/modules/sync.md:43-47` says "server wins" on local change; the client code only overwrites with the server's append delta after confirming nothing local changed during the request (section 3.3).

---

## 12. Summary

Condensed:

| Topic | Upstream behaviour |
|---|---|
| Tables | `Marker`, `Label` (columns `judul`, `urutan`, `warnaLatar`), `Marker_Label`; all with `gid` text `g1:<uuid>` |
| Marker fields | `ari`, `kind` (1/2/3), `caption`, `verseCount`, `createTime`, `modifyTime` (Unix seconds) |
| Range | start ARI plus `verseCount`; same chapter; no end ARI |
| Highlight | one Marker per verse, `verseCount` 1; caption JSON `{"colorRgb":N[,"partial":{...}]}` or legacy `cRRGGBB` |
| Highlight colours | 12 presets plus free picker; stored as RGB; alpha computed at draw time |
| Label | `title` (max 48 in UI), `ordering` int ascending, `backgroundColor` `bRRGGBB` or null |
| Sync | no flags; add/mod/del computed by diff vs. a stored snapshot; one sync set `mabel` |
| Delete | hard delete; Marker delete removes its links; Label delete removes its links but keeps Markers; note-editor delete leaves links |
| UI | verse-selection add menu, bookmark dialog, note editor, highlight sheet, Markers hub, Marker list, data-transfer screen |
