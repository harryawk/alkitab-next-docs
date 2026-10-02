# Cross-platform (iOS + Android + Web) reader stacks: what open-source readers use

Research date: 2026-10-02. Repo context: Expo SDK 57 / RN 0.86 / Expo Router / Jest / bundled `assets/bible/kjv.yes` (YES2 binary), custom `YES2Parser`, zustand.

Convention note: `docs/` only held `docs/agents/` (no notes convention existed), so this lives in a new `docs/research/`.

Legend: **Fact** = verified against a primary source (URL inline). **Rec** = my judgement (section 6 only). Registry/GitHub numbers were pulled live from `registry.npmjs.org`, `api.github.com`, `pub.dev/api` on 2026-10-02. GitHub's REST `open_issues_count` includes PRs.

---

## 1. Survey of real open-source readers (verified stacks)

| Project | Platforms | Stack (verified) | License | Health (2026-10-02) |
|---|---|---|---|---|
| Readest | macOS/Win/Linux/Android/iOS/Web | Next.js 16 + Tauri v2, "modern rewrite of Foliate" ([README](https://github.com/readest/readest/blob/main/README.md)). Manifest: next 16.3.3, react 19.2.8, zustand 5.0.10, `@tauri-apps/*` v2 plugins, `foliate-js` workspace dep, Radix UI, `@opennextjs/cloudflare` for web, vitest/playwright ([package.json](https://github.com/readest/readest/blob/main/apps/readest-app/package.json)); `src-tauri/Info-ios.plist` present ([tree](https://api.github.com/repos/readest/readest/contents/apps/readest-app/src-tauri)) | AGPL-3.0 | latest release v0.12.10 (2026-09-21), pushed 2026-10-01 ([API](https://api.github.com/repos/readest/readest)) |
| foliate-js | any browser / webview | Pure ES modules, no build step, EPUB/MOBI/KF8/FB2/CBZ, PDF experimental via pdf.js. README says the library "is not stable... API to change at any time" and suggests a git submodule ([README](https://github.com/johnfactotum/foliate-js)) | MIT | no releases; last push 2026-05-01 ([API](https://api.github.com/repos/johnfactotum/foliate-js)) |
| Readium ts-toolkit | Web / embedded browser frames | TypeScript; npm `@readium/shared`, `@readium/navigator`, `@readium/navigator-html-injectables`; EPUB, audiobooks, comics ([repo](https://github.com/readium/ts-toolkit)) | BSD-3-Clause | `@readium/navigator` 2.11.1 published 2026-09-30 ([npm](https://registry.npmjs.org/@readium/navigator)) |
| Readium Kotlin / Swift toolkits | Android / iOS native | Native toolkits; Kotlin 3.4.0 (2026-09-11), Swift 3.11.0 (2026-07-17) ([kotlin](https://api.github.com/repos/readium/kotlin-toolkit/releases/latest), [swift](https://api.github.com/repos/readium/swift-toolkit/releases/latest)) | BSD-3-Clause | active |
| react-native-readium (5-stones) | iOS 15.1+, Android, Web | RN wrapper over Readium Swift/Kotlin toolkits, built on Nitro Modules, new + old arch; EPUB 2/3 and PDF; works in Expo via prebuild/EAS plus a config plugin; no DRM; needs Swift 6 / Xcode 16.4+, Kotlin >= 2.3.20 ([README](https://github.com/5-stones/react-native-readium)) | MIT | pushed 2026-09-28, 173 stars ([API](https://api.github.com/repos/5-stones/react-native-readium)) |
| epubjs-react-native | iOS, Android (web not documented) | epub.js inside `react-native-webview`; Expo variant `@epubjs-react-native/expo-file-system` ([README](https://github.com/victorsoares96/epubjs-react-native)) | MIT | core 1.4.8 published 2026-09-08 ([npm](https://registry.npmjs.org/@epubjs-react-native/core)) |
| Mozilla pdf.js | Browsers (+ Firefox) | JS PDF renderer; `pdfjs-dist` 6.3.289 | Apache-2.0 | release v6.3.289 (2026-08-29), pushed 2026-10-01 ([API](https://api.github.com/repos/mozilla/pdf.js)) |
| epub.js | Browsers | Classic JS EPUB renderer. GitHub release tag v0.3.88 is from 2020; npm `epubjs` 0.3.93 published 2022-02-16 ([npm](https://registry.npmjs.org/epubjs)); ~517 open issues+PRs ([API](https://api.github.com/repos/futurepress/epub.js)). GitHub's license detector returns NOASSERTION; npm says BSD-2-Clause | BSD-2 (npm) | little release activity since 2022 (my reading of the dates) |
| Alkitab / Quick Bible (yukuku/androidbible) | Android only | Kotlin/Java, Gradle, SQLite, ExoPlayer; native `.yes` (YES2) Bible format ([README](https://github.com/yukuku/androidbible)); README mentions no iOS/web app | Apache-2.0 | pushed 2026-09-27 ([API](https://api.github.com/repos/yukuku/androidbible)) |
| AndBible | Android only | Kotlin/Java, Vue 3 UI rendered in a WebView, JSword (SWORD-based) as git submodule ([README](https://github.com/AndBible/and-bible)) | GPL-3.0 | pushed 2026-10-01 ([API](https://api.github.com/repos/AndBible/and-bible)) |

Pattern worth noticing (**Fact-derived**): the two readers that genuinely reach iOS+Android+Web from one codebase (Readest, react-native-readium) both reuse web rendering code (foliate-js / Readium ts-toolkit) inside a webview or browser, rather than re-implementing EPUB layout natively. Readest runs it in Tauri's system webview ([Tauri docs](https://v2.tauri.app/start/): "Tauri apps take advantage of the web view already available on every user's system").

Could NOT verify (no usable primary source fetched): YouVersion Platform SDK/licensing (developers.youversion.com returned empty content), CrossWire/SWORD mobile frontends list (wiki URL 404), Digital Bible Platform/API.Bible tooling, Bible.org open code, KOReader web. Nothing is claimed about them. Also no Flutter-based end-user reader app was verified; only Flutter packages (below).

Bible-format specs verified:
- USFM/USX/USJ docs: spec site shows release 3.1 with 3.1.1, 3.1.2, 3.2 available; docs source under MPL-2.0 ([docs.usfm.bible](https://docs.usfm.bible/usfm/3.1/index.html)). Maintainer not stated on that page.
- Scripture Burrito 1.0.0: packaging/interchange spec built on DBL metadata and USFM/USX, maintained by a committee incl. ABS, SIL, unfoldingWord, UBS ([docs.burrito.bible](https://docs.burrito.bible/)).
- YES2: format documented in `docs/binary-formats.md` of yukuku/androidbible ([README](https://github.com/yukuku/androidbible)). Registry-wise it is a niche format; I found no npm parser (not exhaustively searched), so the repo's own `YES2Parser.ts` is likely the only JS implementation. Treat as unverified.
- unfoldingWord/usfm-js exists as a JS USFM parser, pushed 2026-05-23, no license reported by GitHub API ([API](https://api.github.com/repos/unfoldingWord/usfm-js)); license not verified.

---

## 2. Stack families for one codebase on iOS + Android + Web

| Family | Text/EPUB/PDF/markdown rendering | Web maturity | Release cadence / upgrade burden | License |
|---|---|---|---|---|
| **Expo / React Native + react-native-web + Expo Router** | Native `Text` for plain/JSON/Bible text. EPUB: webview + JS lib (epubjs-react-native) or Readium native toolkits (react-native-readium). PDF: native modules (react-native-pdf: PDFKit/pdfium) on native, pdf.js on web. Expo "DOM components" (`'use dom'`) wrap React-DOM code in a WebView on native and render plainly on web ([Expo docs](https://docs.expo.dev/guides/dom-components/)) | RNW is "a set of component libraries such as `<View>`, and `<Text>`, that wrap `react-dom` primitives"; Expo can output static or client-rendered web ([Expo web](https://docs.expo.dev/workflow/web/)). Native-only libs (react-native-pdf, react-native-webview) have no web target ([RN-pdf](https://github.com/wonday/react-native-pdf), [Expo webview](https://docs.expo.dev/versions/v57.0.0/sdk/webview/)) | SDK 56: 2026-05-21, SDK 57: 2026-06-30, SDK 58 beta: 2026-09-15 ([changelog](https://expo.dev/changelog)). SDK 57 pins RN 0.86, React 19.2.3, RNW 0.21.0, Node >= 22.13 ([SDK 57](https://docs.expo.dev/versions/v57.0.0/)). npm latest: expo 57.0.26, react-native 0.87.1 (ahead of Expo's pin) ([expo](https://registry.npmjs.org/expo), [rn](https://registry.npmjs.org/react-native)) | MIT (expo, RN, RNW) |
| **Flutter** | Own renderer (Skia/Impeller), no native text widgets. Packages: pdfrx (PDFium, incl. Web via WASM) ([pub](https://pub.dev/packages/pdfrx)); `epub_view` 3.3.0 (2026-08-20), `epubx` 4.0.0 (2023-06-30) ([pub API](https://pub.dev/api/packages/epub_view)); `flutter_widget_from_html` 0.17.4 | Flutter docs say it is for app-like SPAs and that "Text-rich, flow-based, static content... benefit from the document-centric model that the web is built around" ([Flutter web](https://docs.flutter.dev/platform-integration/web)) | stable 3.47.5 (2026-09-18, Dart 3.13.4) ([release manifest](https://storage.googleapis.com/flutter_infra_release/releases/releases_linux.json)) | BSD-3-Clause; pdfrx MIT; Syncfusion PDF viewer is a commercial-licence package (not verified here, check before use) |
| **Capacitor / Ionic + web tech** | Browser rendering (HTML/CSS) in a native webview, so foliate-js, pdf.js, epub.js, Readium ts-toolkit all run unchanged | "supports iOS, Android, and web/PWA"; "If it works in the browser, it probably works in a mobile app" ([Capacitor](https://capacitorjs.com/docs)) | Capacitor 8.5.2 (2026-09-11) ([API](https://api.github.com/repos/ionic-team/capacitor/releases/latest)) | MIT |
| **Tauri v2 (mobile)** | System webview; same web libs (Readest proves the pattern) | Web is just your SPA | Docs cover Google Play and App Store distribution ([distribute](https://v2.tauri.app/distribute/)); v3.0.0-alpha.4 tagged 2026-10-01 ([API](https://api.github.com/repos/tauri-apps/tauri/releases/latest)). Mobile maturity: could not verify from docs; Rust toolchain required | Apache-2.0 (MIT dual not verified) |
| **Kotlin Multiplatform + Compose Multiplatform** | Compose canvas UI; Readium Kotlin toolkit is Android-only; no verified cross-platform EPUB/PDF component | Compose UI: Android/iOS/Desktop **Stable**, Web (Wasm) **Beta** ([stability](https://kotlinlang.org/docs/multiplatform/supported-platforms.html)); Web needs WasmGC browsers ([compat](https://kotlinlang.org/docs/multiplatform/compose-compatibility-and-versioning.html)) | Compose Multiplatform 1.12.1 (2026-09-22) ([API](https://api.github.com/repos/JetBrains/compose-multiplatform/releases/latest)) | Apache-2.0 |

---

## 3. Libraries per format (Expo/RN + web)

| Format | Option | Platforms | License | Latest / health |
|---|---|---|---|---|
| EPUB | `react-native-readium` (Readium native toolkits; web via ts-toolkit) | iOS, Android, Web | MIT (wrapper), BSD-3 (Readium) | active, pushed 2026-09-28 |
| EPUB | `@epubjs-react-native/core` (+ `expo-file-system` 2.0.0) | iOS, Android | MIT | 1.4.8, 2026-09-08; needs react-native-webview |
| EPUB | `epubjs` (web) | Web | BSD-2 | 0.3.93, 2022-02-16 |
| EPUB / MOBI / FB2 / CBZ | foliate-js via DOM component or webview | Web + webview | MIT | not stable per its README |
| EPUB | `@readium/navigator` 2.11.1 | Web | BSD-3 | published 2026-09-30 |
| PDF | `react-native-pdf` 7.0.5 (PDFKit / pdfium), needs `react-native-blob-util` and a dev client, no web | iOS, Android, Windows | MIT | 2026-08-13 ([npm](https://registry.npmjs.org/react-native-pdf)) |
| PDF | `pdfjs-dist` 6.3.289 | Web (and webview) | Apache-2.0 | 2026-08-29 |
| PDF | `react-native-pdf-light` 3.2.1 | native | MIT | 2026-05-14 (platform scope not verified) |
| Markdown | `react-native-enriched-markdown` 1.1.0 (Software Mansion) | RN (web not verified) | MIT | 2026-10-01 ([npm](https://registry.npmjs.org/react-native-enriched-markdown)) |
| Markdown | `react-native-markdown-display` 7.0.2 | RN | MIT | 2023-12-11 (stale) ([npm](https://registry.npmjs.org/react-native-markdown-display)) |
| HTML | `react-native-render-html` 6.3.4 | RN | BSD-2 | 2022-01-24 (stale) ([npm](https://registry.npmjs.org/react-native-render-html)) |
| Plain text / JSON / Bible text | Native `Text` + a virtualised list (`@shopify/flash-list` 2.3.3, MIT, 2026-10-01 ([npm](https://registry.npmjs.org/@shopify/flash-list))) | all | n/a | n/a |
| Bible data | Bundled SQLite via `expo-sqlite` 57.0.3: `SQLiteProvider assetSource={{ assetId: require(...) }}` loads a pre-built DB; Expo says web support is **alpha** and needs Metro WASM/SharedArrayBuffer header config ([docs](https://docs.expo.dev/versions/v57.0.0/sdk/sqlite/)) | iOS, Android, Web (alpha) | MIT | 2026-09-11 |
| USFM | `unfoldingWord/usfm-js` (JS) | any JS | not verified | pushed 2026-05-23 |
| OSIS / SWORD | No verified RN/web library found | | | not verified |

Not verified: open-issue activity inside each library's own tracker beyond the GitHub counts above; I did not triage issues.

---

## 4. Verified facts about the current repo (for context)

- `package.json`: expo ~57.0.18, react-native 0.86.3, react-native-web ~0.21.0, expo-router ~57.0.17, jest-expo ~57.0.5, zustand ^5.0.15; `buffer` ^6.0.3 used by `src/services/YES2Parser.ts` (reads a `Buffer`).
- `metro.config.js` adds `yes` to `assetExts`; the bundled file is `assets/bible/kjv.yes` (~1.9 MB).
- Installed Expo is 57.0.18 while registry latest is 57.0.26 (patch drift) and SDK 58 is in beta per the changelog.

---

## 5. What the sources do and do not tell us

- A KJV-sized text corpus needs no EPUB/PDF engine; every reader engine above exists to solve reflowable-ebook or page-fixed layout, which Bible-by-verse text does not need.
- The sources show web-tech reuse (foliate-js / Readium TS) as the dominant way real readers get EPUB on all three platforms.
- Flutter's own docs warn against text-heavy document content on web.
- Nothing verified shows a Bible-specific open-source app on iOS+Android+Web from one codebase.

---

## 6. Recommendation for THIS repo (opinion, not sourced fact)

1. **Stay on Expo SDK 57 + Expo Router + RNW.** The existing app is plain text in native `Text`, which already works on all three targets with zero rendering dependencies, and Jest/jest-expo is the officially paired test setup. Switching families would discard working code for no gain I can source.
2. **Keep the YES2 parser pure TypeScript and platform-neutral** (inputs `Uint8Array`/`Buffer`, outputs plain objects), tested in Jest. Consider a build-time script converting `.yes` to JSON/SQLite so runtime has no custom binary parsing; that decouples the app from the niche format. Trade-off: more build steps vs. less runtime code and easier web support.
3. **Do not add EPUB/PDF libraries until a user story requires them.** If EPUB later: `react-native-readium` is the only option I found covering iOS+Android+Web with native-quality rendering, but it requires a dev client / prebuild, Swift 6 / Xcode 16.4+, and Kotlin >= 2.3.20. Lighter fallback: foliate-js or `@readium/navigator` inside an Expo DOM component (web-reuse pattern Readest proves), accepting WebView performance limits that Expo itself documents.
4. **Storage:** for text-by-verse lookups prefer `expo-sqlite` with a bundled DB, but gate on its web alpha status; keep an in-memory/JSON fallback behind an interface so web does not depend on it.
5. **Avoid:** react-native-pdf (no web, dev-client only), unmaintained `react-native-markdown-display` / `react-native-render-html` (last publish 2022-2023), `epubjs` as a new dependency (last npm publish 2022), foliate-js pinned without a submodule/lock (self-declared unstable), and `react-native-reanimated`/Worklets upgrades outside Expo's pinned versions (use `npx expo install --check`).
6. **Licensing watch:** Readest (AGPL-3.0) and AndBible (GPL-3.0) are reference-only; copying code would impose copyleft. Bible *text* licensing (KJV public domain vs. modern translations) is a separate question not researched here.

---

## 7. Open questions

- Is the app expected to offer user-imported EPUB/PDF, or only bundled Bible text? This decides section 6 item 3.
- Should additional Bible formats (USFM, OSIS, SWORD modules, other `.yes` translations) be supported at runtime, or converted at build time?
- Expo web output mode (static vs. single) and SEO needs for verse pages are undecided; docs list both modes ([Expo web](https://docs.expo.dev/workflow/web/)).
- Upgrade timing: SDK 58 beta exists (2026-09-15); AGENTS.md pins v57 docs. Decide when to move.
- Tauri 2 mobile production-readiness, YouVersion/SWORD/DBP tooling, and USFM parser licenses were not verified.
- Which translations and licences does the project plan to ship (affects distribution on App Stores and web)?
