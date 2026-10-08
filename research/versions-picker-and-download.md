# expo-document-picker and URL download in Expo SDK 57

Sources (fetched 2026-10-09; page text was condensed by a fetch summarizer, so quotes are as returned, not byte-checked):
- [DocPicker] https://docs.expo.dev/versions/v57.0.0/sdk/document-picker/
- [FS] https://docs.expo.dev/versions/v57.0.0/sdk/filesystem/
- [MDN-pick] https://developer.mozilla.org/en-US/docs/Web/API/Window/showOpenFilePicker
- [MDN-CORS] https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS

## expo-document-picker `getDocumentAsync` [DocPicker]

- Shows system UI on Android, iOS and Web.
- `type`: `string | string[]`, "The MIME type(s) of the documents that are available to be picked. It also supports wildcards like `'image/*'`". Default `'*/*'`.
- The docs give no UTI or file-extension filtering, and no per-platform note on how an unregistered extension such as `.yes` maps. Unverified: whether a custom extension has a MIME type on each platform, and whether iOS/Android filter by MIME only. Inference, not documented: use `'*/*'` and validate the name/extension after picking.
- `copyToCacheDirectory` (Android/iOS only): default `true`. "If `true`, the picked file is copied to `FileSystem.CacheDirectory`, which allows other Expo APIs to read the file immediately." Docs suggest `false` for large files when immediate read is not needed. Not applicable on web.
- `multiple`: default `false`. `base64` (Web only): default `true`; `false` returns a file URL.
- Result: success `{ canceled: false, assets: DocumentPickerAsset[], output (web FileList) }`; asset fields `uri`, `name`, `lastModified`, `mimeType?`, `size?`, plus `base64?` and `file?` on web. Cancel: `{ canceled: true, assets: null }` (`output` null on web).
- Web: "The system UI can only be shown after user activation (e.g. a `Button` press)"; must not be called from lifecycle methods like `componentDidMount`. On web the cancel event does not reliably return because of browser inconsistencies [DocPicker], so a cancelled pick may never resolve.
- iOS iCloud: config plugin `expo-document-picker` with `iCloudContainerEnvironment` ("Development"/"Production") and `kvStoreIdentifier`; `ios.usesIcloudStorage` in app config [DocPicker].
- The activation rule is generic browser behavior: `showOpenFilePicker()` needs transient user activation, `<input type="file">` does not per the same page [MDN-pick]. Unverified: which mechanism expo-document-picker uses internally on web.

## expo-file-system v57 download [FS]

- `File.downloadFileAsync(url, destination, options?)` returns `Promise<File>`. Platforms: Android, iOS, tvOS. Web is not listed.
- `DownloadOptions`: `headers` (optional), `idempotent` (default `false`: error if the target exists; `true`: overwrite), `onProgress` (callback with `{ bytesWritten, totalBytes }`), `signal` (`AbortSignal` for cancel). Unverified: two fetches disagreed on whether `onProgress`/`signal` sit on `DownloadOptions` or only on the task; check the typings in `node_modules/expo-file-system` before relying on either.
- `File.createDownloadTask(...)` (Android, iOS, tvOS): creates a task without starting. `downloadAsync()` (once, while idle), `cancel()`, `pauseAsync()`, `resumeAsync()`, `onProgress` with `DownloadProgress { bytesWritten, totalBytes }`.
- Legacy (deprecated) in `expo-file-system/legacy`: `createDownloadResumable()`, `downloadAsync()`.
- `Paths.cache` (temporary) and `Paths.document` (persistent). `File` offers `text()`, `bytes()`, `base64()` (async and Sync variants) and `write()`; implements Blob.
- `expo/fetch` can be used with files as blobs/FormData [FS].
- Web: no support statement for file-system download. Unverified whether `File` classes work on web at all.
- `pickFileAsync` (Android, iOS, tvOS; not web): options `initialUri`, `mimeTypes`, `multipleFiles`. An alternative to document-picker.

## Fetching a URL on web [MDN-CORS]

- Cross-origin `fetch()` from script cannot read the response unless the server sends `Access-Control-Allow-Origin` (origin or `*`). `no-cors` yields an opaque, unreadable response.
- `Content-Length` is only readable by script if the server lists it in `Access-Control-Expose-Headers`; without it, total size (hence percent progress) may be unknown.
- Progress on web (reading `response.body` stream) and cancellation via `AbortController` are standard web APIs, not documented on the Expo pages; unverified here against MDN.
- `<a href download>` navigation avoids CORS reads but gives the app no bytes or progress. Unverified here; general browser behavior.

## Open items to verify on device / in typings
1. `.yes` behaviour of `type` on iOS, Android, web (use of `'*/*'`).
2. Exact location of `onProgress`/`signal` in v57 typings.
3. Whether `File.downloadFileAsync` has any web fallback.
