# Where an imported `.yes` Bible file can live (SDK 57)

Scope: imported `.yes` files up to tens of MB on iOS, Android, web. Facts only; no decision.
Legend: **[V]** verified in a primary source named inline; **[U]** unverified or inferred.
Installed: expo-file-system 57.0.7, expo-sqlite 57.0.3 (package.json, node_modules/*/package.json).

## 1. Native (iOS, Android): expo-file-system document directory

- Platforms: docs list only Android, iOS, tvOS (and Expo Go). [V] https://docs.expo.dev/versions/v57.0.0/sdk/filesystem/
- `Paths.document` is "safe from system deletion; intended for user-created content". `Paths.cache` is "subject to deletion when storage is low". [V] same page.
- iOS: Documents is included in iCloud/device backup; Library/Caches and tmp are not backed up and can be purged by the system. [V] https://developer.apple.com/documentation/foundation/optimizing-your-app-s-data-for-icloud-backup . Large imported files in Documents count toward backup unless excluded with `NSURLIsExcludedFromBackupKey`; whether expo-file-system exposes that flag is [U].
- Android: `filesDir` is persistent, `cacheDir` is auto-deleted when space is low; files removed on uninstall; no permission needed. [V] https://developer.android.com/training/data-storage/app-specific . That `Paths.document` maps to `filesDir` is [U] (not stated on the Expo page fetched).
- Size limit: none documented for storing; practical limit is free disk (`Paths.availableDiskSpace` / `totalDiskSpace` exist). [V] Expo page. No Expo-imposed cap found [U: absence of evidence].
- Getting the file in: `File.pickFileAsync()` (single/multiple, MIME filter) exists. Whether the picked file is a copy or a reference is not documented [U]. Android SAF `content://` URIs have mode restrictions (no ReadWrite). [V] Expo page. Copying into `Paths.document` after pick is the likely path, [U] untested.

### Byte-range reads (native)
- `File.open(mode?)` returns `FileHandle` with `offset` (settable), `size`, `readBytes(length)`, `writeBytes`, `close()`. `readBytes` is declared synchronous, returning `Uint8Array<ArrayBuffer>`. [V] node_modules/expo-file-system/src/File.types.ts:90-106 and Expo page.
- Single-read cap: "capped by ArrayBuffer max size (32 bit signed MAX_INT on Android and 64 bit on iOS)". [V] Expo page. Tens of MB is far below that.
- So a `readRange(offset, length)` source can be built as `handle.offset = o; handle.readBytes(n)` without loading the whole file. API shape [V]; performance [U].
- Current code for comparison: `src/services/assetBytes.ts` does `new File(asset.localUri).bytes()`, i.e. loads the entire file into a `Uint8Array`.

## 2. Web

### expo-file-system is not usable
- Not listed as a supported platform [V]; the installed web module is a stub that logs `'expo-file-system is not supported on web'` for File, Directory, `documentDirectory`, `pickFileAsync`, and returns ''/0 for directories and disk space. [V] node_modules/expo-file-system/src/ExpoFileSystem.web.ts
- Matches `src/services/assetBytes.web.ts`, which uses `fetch(asset.uri)` then `arrayBuffer()` (whole file in memory).

### Shared web storage rules (OPFS, IndexedDB and sqlite-on-OPFS)
Source: https://developer.mozilla.org/en-US/docs/Web/API/Storage_API/Storage_quotas_and_eviction_criteria [V]
- Quota: Chrome/Edge up to 60% of disk per origin; Firefox best-effort min(10% disk, 10 GiB group), persistent up to 50% disk; Safari (macOS 14+/iOS 17+) ~60% disk in browser apps, ~15% in embedded WebView, ~60% for Home Screen web apps; older Safari 1 GiB then prompt. Tens of MB is well under all of these.
- Default mode is best-effort: the browser may evict under storage pressure (LRU) without asking. `navigator.storage.persist()` opts into persistent mode (Firefox prompts; Chrome/Safari auto-decide based on user interaction per MDN). Persistent origins are skipped by pressure eviction.
- Eviction is whole-origin: IndexedDB, Cache API and other data of an origin are deleted together.
- Safari proactive eviction: with cross-site tracking prevention, origins with no user interaction for 7 days have script-created data deleted. Whether Home Screen web apps are exempt is [U].
- Data is also lost when the user clears site data (OPFS page). [V]
- Check with `navigator.storage.estimate()`; writes can throw `QuotaExceededError`. [V]

### Option A: OPFS
- Baseline widely available since March 2023, secure context only, available in Workers. [V] https://developer.mozilla.org/en-US/docs/Web/API/File_System_API/Origin_private_file_system
- Main thread: `getFileHandle(name,{create:true})`, `createWritable()` to write, `getFile()` to read. Workers only: `createSyncAccessHandle()` with `read(buffer,{at})`, `write`, `truncate`, `flush`, `close`. [V] same page.
- Byte ranges: sync handle `read(..., {at})` gives true random access in a Worker [V]. On the main thread `getFile()` returns a `File` (a `Blob`); `Blob.slice(start,end)` returns a sub-Blob [V] https://developer.mozilla.org/en-US/docs/Web/API/Blob/slice . That `slice().arrayBuffer()` reads only the range from disk for OPFS-backed files is [U] (MDN does not say).
- Persistence: subject to the shared quota/eviction rules above.

### Option B: IndexedDB
- Same quota/eviction bucket as OPFS. [V]
- Can hold a `Blob`/`ArrayBuffer` value per key (general IndexedDB capability; not re-fetched here). Per-browser blob size caveats are [U].
- Byte ranges: reading a record returns the whole value; a stored `Blob` can be `.slice()`d, but whether the browser materialises the whole blob is [U]. Chunking the file into fixed-size records gives range reads by key; that is a design pattern, not a documented feature [U].

### Option C: expo-sqlite blob
- Web support is **alpha**; needs Metro WASM config and `Cross-Origin-Embedder-Policy` + `Cross-Origin-Opener-Policy` headers for `SharedArrayBuffer`. [V] https://docs.expo.dev/versions/v57.0.0/sdk/sqlite/
- Persistent web DB uses wa-sqlite `AccessHandlePoolVFS` (OPFS sync access handles, in a worker); `:memory:` uses `MemoryVFS`. [V] node_modules/expo-sqlite/web/worker.ts lines 10-11, 48-49, 405-406, 789-790. So a sqlite blob on web ends up in OPFS and shares its quota/eviction.
- SQLite blob size: default `SQLITE_MAX_LENGTH` 1,000,000,000 bytes, hard cap 2^31-3. [V] https://sqlite.org/limits.html . Whether Expo's build overrides this is [U].
- Incremental blob I/O (`sqlite3_blob_open`) is a SQLite C API not covered by sqlite.org/limits; whether expo-sqlite 57 exposes it was **not found** [U]. SQL alternative: `substr(blob, offset, length)` returns a byte range (standard SQLite function, [U] not re-verified; may still read the whole row internally).
- Also usable on native, giving one storage path on all platforms; native DB location and backup behaviour not verified [U].

## 3. Can the engine read ranges? (relation to issue 05)

`.scratch/yes-parser/issues/05-reader-memory-and-utf8.md` asks for lazy per-block Snappy decompression of `text` with a small block cache, and optionally "random-access sources (`readRange`) so big files need not be fully loaded".

| Storage | Range read without full load | Basis |
|---|---|---|
| Native doc dir, `FileHandle` | Yes (`offset` + `readBytes`, sync API) | [V] types + docs |
| Web OPFS, Worker sync handle | Yes (`read(buf,{at})`) | [V] MDN |
| Web OPFS, main-thread `File.slice` | Likely, not documented | [U] |
| Web IndexedDB single Blob/ArrayBuffer | Not guaranteed | [U] |
| Web IndexedDB chunked records | Yes by design | [U] pattern |
| expo-sqlite blob | Unclear (substr / incremental I/O) | [U] |
| Current `loadAssetBytes` (native and web) | No, whole file | repo code |

Note: native `readBytes` is synchronous, while OPFS sync access exists only in Workers and main-thread OPFS/IndexedDB are async; a shared `readRange` interface would need to be async (or worker-hosted) to cover web. Signatures [V]; the conclusion is inference.

## 4. Open items not verified
- Whether `pickFileAsync` copies or references the picked file on iOS/Android.
- expo-sqlite 57 incremental blob API, and web alpha stability for tens-of-MB rows.
- Safari `persist()` grant behaviour and any Home Screen exemption from the 7-day cap.
- Real-world IndexedDB Blob behaviour per browser.
