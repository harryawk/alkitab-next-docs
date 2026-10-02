# Ideation: migrate androidbible to Expo (iOS, Android, web)

Date: 2026-10-02. Focus: "migrate https://github.com/yukuku/androidbible so the users can use the app with exactly the same functionality with iOS and Android and web".

## Subject

Target repo: this one (Expo SDK 57, expo-router, React Native 0.86, zustand, jest). Upstream: `yukuku/androidbible` (default branch `develop`, Apache-2.0 code, active; last push 2026-09-27).

Key grounding, after verification:

- **Local port is tiny and its parser is broken.** It is a KJV book list, chapter picker and chapter reader, under ~5% of upstream. `src/services/YES2Parser.ts` finds sections by `buffer.indexOf("text")` (`:281-282`) and `indexOf("shortName")` (`:311-317`). Verified against the real `assets/bible/kjv.yes`: `indexOf("text")` hits byte 98 inside the section index, so chapter text comes out empty or garbage. Only `getBookNames` works. `npx jest`: 1 failed, 6 passed; the failing test uses a hand-built mock buffer, and nothing tests the real file.
- **The real YES2 layout** (parsed from `kjv.yes`): 8-byte header `98580d0a005de002`, section index with big-endian ints, 4 sections (versionInfo, booksInfo, pericopes, text). The `text` section carries Bintex attributes `compression.name = "snappy-blocks"` with `block_size` and `compressed_block_sizes`, so it is Snappy-compressed per fixed-size block. Upstream `docs/binary-formats.md:288` says the whole section is one block, which this file contradicts.
- **`@`-codes** (upstream `docs/text-rendering.md`): `@6`/`@5` red letter, `@9`/`@7` italic, `@0`-`@4` indent, `@8` line break, `@@`, `@^`, `@<f1@>...@/` footnote, `@<x1@>...@/` xref, `@<r=...@>` ruby. The local `cleanYukukuTags` regex (`YES2Parser.ts:275`) strips only `@` plus one character and leaves tag payload text such as `f1@/`.
- **Effort and scope, from upstream's own iOS-port survey** (`docs/features/ios-port/feasibility-survey.md`): line 5 says a line-by-line 1:1 port "is not feasible" but a faithful functional 1:1 port is; ~1,850 engineer-hours total (line 5, 405); "reader + markers + sync is probably ~1,000 hours" (line 407). It targets native Swift, not Expo. An earlier note of "1,600+ h" in this session was wrong.
- **Android-only upstream pieces:** ContentProvider, AppWidget, BroadcastReceiver, WorkManager, FCM, ExoPlayer/MIDI, SharedPreferences. Storage is SQLite (Marker, Label, Marker_Label, ProgressMark, ReadingPlan and others); the sync protocol and file formats are platform-neutral.
- **Licensing:** upstream code is Apache-2.0 (`LICENSE.txt`, no NOTICE). Bible texts and Firebase config for the published apps come from a private proprietary directory (survey §11), so texts are not covered by the code license. Terms for the maintainer's backend (`api.alkitab.app`) were not found.
- **Upstream JVM tests** exist for YES2 and Snappy (`AlkitabYes2/src/test/java/yuku/alkitab/yes2/Yes2RoundTripTest.kt`, `SnappyStreamRoundTripTest.kt`) but they are synthetic round-trips with no committed golden data from real files. No published JS/TS/Swift YES2 reader was found.

## Ranked ideas

### 1. Oracle-first YES2 core: golden tests, then a real pure-TS reader

- **Summary:** Write jest tests against the real `assets/bible/kjv.yes` first (Gen 1:1, John 3:16, Rev 22:21, verse counts for all 66 books). Optionally generate extra fixtures by running upstream's Kotlin `Yes2Reader` on the JVM. Then build a spec-faithful reader with no React Native or Expo imports: header and section index, Bintex (big-endian ints), per-block Snappy decode, and a typed `Version` with `getVerses(ari)`. First slice: get Gen 1 and John 3 passing, and delete the commented-out parser and test code. Block-wise compression also makes lazy per-block access feasible later.
- **Basis:** `direct:` `src/services/YES2Parser.ts:281-282` (`this.buffer.indexOf(tagName)` for `"text"`) hits the section-index entry at byte 98; reading `readUInt32LE` there gives 11,089,920 instead of the real big-endian offset 14505. `direct:` `kjv.yes` text-section attributes read `compression.name = "snappy-blocks"`. `external:` upstream survey §14 recommends shared test vectors and fixtures generated on Android ("tests to validate round-trip against fixture files generated on Android", line 121).
- **Why it matters:** every other feature (search, markers, split view, versions) reads through this module, and it is the one that must match upstream byte for byte on all three platforms.
- **Size:** M

**Take it further:**

```
/supermatt:grill-with-docs Take this idea into shape: "Oracle-first YES2 core". Context: we are migrating yukuku/androidbible (https://github.com/yukuku/androidbible, branch develop) to this Expo SDK 57 repo so iOS, Android and web behave identically. The current parser src/services/YES2Parser.ts is broken on the real assets/bible/kjv.yes (it uses buffer.indexOf("text") at :281-282, which hits byte 98 in the section index; only getBookNames works). Real layout: header 98580d0a005de002, section index with big-endian ints, 4 sections (versionInfo, booksInfo, pericopes, text), text section attributes say compression "snappy-blocks" with block_size and compressed_block_sizes (upstream docs/binary-formats.md:288 wrongly says one block). Proposal: write golden jest tests against the real kjv.yes first (and maybe JVM-generated fixtures from upstream's Yes2Reader; upstream tests are only synthetic round-trips in AlkitabYes2/src/test/.../Yes2RoundTripTest.kt), then a pure-TS reader (section index, Bintex, Snappy blocks, typed Version with getVerses(ari)) with no React Native imports. Decide: package boundary, Snappy implementation (hand-written vs a vetted npm package), build-time conversion vs runtime parsing, and how the oracle is produced.
```

### 2. `parseVerse()`: one `@`-code grammar to a typed span tree

- **Summary:** Replace `cleanYukukuTags` and the `string[]` verse type with a tokenizer that returns typed spans: red letter, italic, indent, line break, footnote and xref anchors, ruby and Strong's, plus pericope headings. Each platform maps spans to `<Text>` styles. The same tree feeds search (strip markup), copy and share, and later selection and highlights. Pin it with string-fixture tests that follow upstream's `docs/text-rendering.md`.
- **Basis:** `direct:` `YES2Parser.ts:275` is `text.replace(/@[<|>|0-9|7|8|9]/g, "").trim()`; verified it leaves `@<f1@>@/` as `f1@/` and ignores `@@`, `@^`, `@/`. `direct:` upstream `docs/text-rendering.md` defines `@6/@5` red letter, `@9/@7` italic, `@0-@4` indent, `@<f1@>`/`@<x1@>`/`@<r=...@>` special elements.
- **Why it matters:** this is the "reads the same on three platforms" surface; a plain-string verse type blocks selection, highlights and footnotes later.
- **Size:** M

**Take it further:**

```
/supermatt:grill-with-docs Take this idea into shape: "parseVerse() span AST for the Alkitab @-code markup". Context: migrating yukuku/androidbible to this Expo repo (iOS, Android, web, identical behaviour). src/services/YES2Parser.ts:275 cleanYukukuTags is /@[<|>|0-9|7|8|9]/g, which strips only '@'+1 char and leaves footnote payload text ('@<f1@>@/' becomes 'f1@/'); app/reader.tsx:37-47 renders verses as plain strings with verse number = array index+1. Upstream grammar (https://raw.githubusercontent.com/yukuku/androidbible/develop/docs/text-rendering.md): @6/@5 red letter, @9/@7 italic, @0-@4 indent, @8 line break, @@ and @^ verse-level, @<f1@>..@/ footnote, @<x1@>..@/ xref, @<r=reading@>base@/ ruby. Proposal: a pure function raw text -> typed spans, rendered by one RN <Text> tree, also used for search stripping, copy and share. Decide: span type model, how footnotes/xrefs reference the xrefs/footnotes sections, where pericope headings live, and the fixture set.
```

### 3. Define "exactly the same" as a pinned, testable parity contract

- **Summary:** Pin one upstream commit SHA and one flavor (`plain`, `yuku_alkitab`, `yuku_quick_bible` or `sabda_alkitab`). Define parity as the data contract (YES2 bytes, ARI, sync protocol) plus behavior, not pixel-identical screens. Keep `docs/parity-matrix.md` with one row per upstream feature, columns iOS, Android and Web, each cell "verified by test" or "unverified", and explicit deferred / cannot-be-identical rows (AppWidget, ContentProvider, FCM, volume-key paging). Phase it: reader and versions, then markers, search and JSON export/import, then sync, plans, devotions and songs. The first ADR records the parity definition.
- **Basis:** `external:` upstream `docs/features/ios-port/feasibility-survey.md:5` ("A true line-by-line 1:1 port is not feasible ... a faithful functional 1:1 port ... is feasible"), `:405-407` (~1,850 h total; MVP "reader + markers + sync is probably ~1,000 hours"). `reasoned:` upstream is a moving target and has four flavors, so "exactly the same" is not falsifiable without a pinned target; features with no iOS or web equivalent make literal identity impossible, so the exclusions must be written down.
- **Why it matters:** it turns an open-ended migration into a checklist you can stop at, and it must exist before tickets are written.
- **Size:** S

**Take it further:**

```
/supermatt:grill-with-docs Take this idea into shape: "parity contract for the androidbible migration". Context: we are porting yukuku/androidbible (https://github.com/yukuku/androidbible, default branch develop, four flavors: plain, yuku_alkitab, yuku_quick_bible, sabda_alkitab) to this Expo SDK 57 repo (iOS, Android, web). Upstream's own survey docs/features/ios-port/feasibility-survey.md says a line-by-line port is not feasible (line 5) but a functional 1:1 port is, ~1,850 engineer-hours (line 405) with an MVP of reader+markers+sync at ~1,000 h (line 407); it targets native Swift, not Expo. The local app is under 5% of upstream. Proposal: pin a commit SHA and flavor, define parity as data contract plus behaviour, keep docs/parity-matrix.md (feature x iOS/Android/Web, verified-by-test), list deferred and cannot-be-identical features (widgets, ContentProvider, FCM, volume keys), and phase the work. Decide: which flavor and SHA, what 'same' means per feature, which features are in v1, and write the GLOSSARY.md terms and first ADR.
```

### 4. ARI as the only address, and a serialisable session snapshot

- **Summary:** Replace `(bookIndex, chapterIndex)` with upstream's 24-bit ARI (`bookId<<16 | chapter<<8 | verse`) in the store, routes (`/reader?ari=...`, which also gives web deep links) and any future DB column. Persist only `{versionId, ari, scroll, settings}`, and hold the parser as a service looked up by version id instead of in zustand state. Version ids follow upstream's `internal`, `preset/<name>`, `file/<path>` scheme.
- **Basis:** `direct:` `src/store/useBibleStore.ts:13-20` holds `parser, bookIndex, chapterIndex` with no persistence, no version id and a non-serialisable parser instance. `direct:` `app/reader.tsx:37-47` derives the verse number from array index + 1. `external:` upstream survey §6 recommends keeping ARI for sync and URL compatibility; markers and sync key on `ari` (`docs/modules/markers.md`, per grounding).
- **Why it matters:** markers, history, pins, plan progress and sync all key on ARI, so a wrong address model forces a data migration later; this is cheap now.
- **Size:** S

**Take it further:**

```
/supermatt:grill-with-docs Take this idea into shape: "ARI addressing and serialisable session state". Context: migrating yukuku/androidbible to this Expo repo (iOS, Android, web). src/store/useBibleStore.ts:13-20 keeps a YES2Parser instance plus bookIndex/chapterIndex in zustand with no persistence and no version concept; app/reader.tsx:37-47 uses array index+1 as verse number. Upstream addresses verses with ARI = bookId<<16 | chapter<<8 | verse (docs/features/ios-port/feasibility-survey.md §6) and identifies versions as 'internal', 'preset/<name>', 'file/<path>' (docs/modules/versions.md). Proposal: ARI everywhere (store, routes like /reader?ari=, future DB), persist {versionId, ari, scroll, settings}, parser as a service keyed by versionId. Decide: book id mapping vs file book order, route shape for web deep links, persistence layer (AsyncStorage vs expo-sqlite), and the migration of the existing store tests.
```

### 5. Storage port with contract tests, SQL schema kept faithful

- **Summary:** Define one repository interface for markers (bookmark/note/highlight), labels, progress-mark pins and history. Back it with the upstream table shapes, `expo-sqlite` on native and sqlite-wasm or IndexedDB on web, and run one contract-test suite against every adapter. Keep sync-shaped fields (`gid`, timestamps, add/mod/del) from day one so sync can be added without a migration. Rejected variant: a single JSON document keyed by ARI, because it conflicts with a faithful data and sync contract.
- **Basis:** `external:` upstream `docs/storage.md` and `docs/modules/markers.md` (tables Marker, Label, Marker_Label, ProgressMark, History); survey line ~200 h for DB and "reuse the schema verbatim" (per grounding). `direct:` `package.json` has no SQLite or storage module; `app/index.tsx:25` already branches on `Platform.OS === "web"` inside a screen. `reasoned:` the storage split between native and web is where cross-platform parity is most likely to drift, and a contract test catches a divergence in one file.
- **Why it matters:** markers, labels, pins, history and plan progress all sit on this layer; deciding it late means doing the work twice.
- **Size:** L

**Take it further:**

```
/supermatt:grill-with-docs Take this idea into shape: "cross-platform storage port with contract tests". Context: migrating yukuku/androidbible to this Expo SDK 57 repo (iOS, Android, web). Upstream stores user data in SQLite tables Marker, Label, Marker_Label, ProgressMark(+History), ReadingPlan(+Progress), Devotion, SyncShadow, SyncLog (docs/storage.md, docs/modules/markers.md at https://raw.githubusercontent.com/yukuku/androidbible/develop/docs/). Locally package.json has no SQLite/storage module and app/index.tsx:25 branches on Platform.OS inside a screen. Proposal: one repository interface for markers/labels/pins/history, adapters for expo-sqlite (native) and sqlite-wasm or IndexedDB (web), a shared contract-test suite, sync-shaped fields (gid, timestamps, add/mod/del) from day one. Rejected: single JSON document keyed by ARI (conflicts with a faithful schema and sync contract). Decide: web adapter choice, schema versioning/migrations, and what the first slice persists.
```

### 6. Licensing gate: ship engine + public-domain KJV + "import your own .yes" first

- **Summary:** Make importing a `.yes` file (file picker, or URL) a first-class early feature, and ship only the engine plus the public-domain KJV. Defer proprietary Bible texts and the maintainer's backend until terms are confirmed. Keep attribution for Apache-2.0 ported code (no NOTICE upstream).
- **Basis:** `external:` upstream survey §11 (published apps pull proprietary `tb_*`, `kjv_*` texts and Firebase config from a private `$ALKITAB_PROPRIETARY_DIR`), and the open-source `plain` build ships only placeholder data (`docs/binary-formats.md`); `LICENSE.txt` is Apache-2.0 and there is no NOTICE. Not verified: per-version copyright in `version_config.json`, and terms for `api.alkitab.app`. `direct:` `assets/bible/kjv.yes` is the only Bible locally and its origin is unverified.
- **Why it matters:** it removes a hidden legal blocker from the "same functionality" goal before feature work, and it makes version import the early feature that lets any user bring their own texts.
- **Size:** S

**Take it further:**

```
/supermatt:grill-with-docs Take this idea into shape: "licensing gate and import-your-own-.yes for the androidbible migration". Context: migrating yukuku/androidbible to this Expo repo. Upstream code is Apache-2.0 (LICENSE.txt, no NOTICE) but Bible texts and Firebase config for the published apps come from a private $ALKITAB_PROPRIETARY_DIR (docs/features/ios-port/feasibility-survey.md §11); the open-source 'plain' flavor ships only placeholder data. Terms for the maintainer's backend (api.alkitab.app) and per-version copyright in version_config.json were NOT verified. Locally assets/bible/kjv.yes (1,941,000 bytes) is the only Bible and its provenance is unverified. Proposal: ship engine + public-domain KJV + first-class .yes import (file picker/URL), defer proprietary texts and the hosted backend until terms are confirmed, keep Apache-2.0 attribution. Decide: where kjv.yes came from and whether it can be bundled, how version import fits the version model, and what to ask the maintainer.
```

## Cut

| Idea | Reason |
| --- | --- |
| Build-time converter that emits platform-neutral JSON and deletes runtime YES2 parsing | Alternative to idea 1 for the bundled asset only; loses "import your own .yes" (idea 6). Fold in as a later optimization once the reader exists. |
| Keep user data as one JSON document keyed by ARI | Conflicts with a faithful schema and sync contract; weakest idea for fidelity (verifier). Superseded by idea 5. |
| Foundation PR (delete template residue, fix `initialRouteName`, declare `expo-asset`/`expo-file-system`, remove `console.log`, fix the failing test) | Real but outside the parity focus; it is hygiene. Do it as the first slice of idea 1. |
| One-day spike: render John 3 with red letters on all three platforms | Answer already known: the current parser produces empty or garbage text on `kjv.yes`, and a spike would need idea 1 first. Duplicate of ideas 1 and 2. |
| Offline-first v1 with no backend (drop sync and devotions) | Not "exactly the same" functionality; use as a phase inside idea 3, not a separate idea. |
| Lazy section access vs decompress-once cache decision | Premise settled: `kjv.yes` is Snappy-compressed per block with a block-size table, so lazy per-block access is feasible. Falls out of idea 1. |
| Platform-split rule (no `Platform.OS` in `app/`, `.native.ts`/`.web.ts` behind interfaces) | Process rule, not a build candidate; fold into ideas 3 and 5. |
| Combination "J+D+C thin vertical slice" | Redundant with ideas 1 and 2 once the spike is cut. |
| Combination "E+F address and persistence foundation" | Covered by ideas 4 and 5 as separate, smaller moves. |
