# Lighthouse baseline for the web build

Research date: 2026-10-18. Ticket: GitHub #14, step 1 (baseline) only. Steps 2 to 7 are not started. No app code changed. Every number below is measured on this machine, on the commit `61ecd26` plus the issue-14 branch (no source edits).

Scope: the numbers come from `http://127.0.0.1:8114/alkitab/reader` on localhost. They do not come from the Tailscale Funnel path, so the tunnel and TLS costs are not in them.

Every number in this note is measured.

---

## 1. Commands

Export, run in the worktree `C:\Users\sonof\Projects\AlkitabNext-worktrees\issue-14`:

```
npx expo export -p web --clear
```

The export ran with `TMP` and `TEMP` set to a scratch directory outside the repo (`C:/Users/sonof/Projects/pw-harness/tmp`). The first export without `--clear` failed. Metro reused a cache entry that pointed at the `issue-22` worktree (`Unable to resolve module @/components/ConfirmDialog from ...\issue-22\app\markers.tsx`). The worktree's `node_modules` is a symlink to `C:\Users\sonof\Projects\AlkitabNext\node_modules`, so the Expo CLI also writes `node_modules/.cache/dev` into that main repo. This note does not touch that path beyond the cache the CLI writes.

Static server, outside the repo: `node C:/Users/sonof/Projects/pw-harness/serve-dist.js <dist> 8114`. It serves `dist` at `/` and at `/alkitab/`, because `app.json` sets `experiments.baseUrl` to `/alkitab` and `index.html` references `/alkitab/_expo/...`. Unknown paths without an extension fall back to `index.html`. The server sends no compression and `cache-control: no-cache`.

Lighthouse, outside the repo: `lighthouse` 13.5.0 installed in `C:\Users\sonof\Projects\pw-harness`. Runner: `node C:/Users/sonof/Projects/pw-harness/lh-run.mjs <outDir> 3`. It launches Chrome with `chrome-launcher` in `--headless=new` mode, one fresh instance per run, with categories performance, accessibility, best-practices, and seo. Mobile uses the Lighthouse default config. Desktop uses `lighthouse/core/config/desktop-config.js`.

Screenshot: `node C:/Users/sonof/Projects/pw-harness/shot-reader.mjs <png>`, using `playwright-core` with `chromium.launch({ channel: 'chrome' })`, viewport 390x844, waiting for network idle plus 3 s.

---

## 2. Bundle sizes

Measured on `dist/` produced by the export above. Gzip and Brotli sizes come from Node `zlib` on the same bytes. Gzip is level 6 (Node default), and Brotli is Node's default quality. The static server does not compress, so Lighthouse transferred the raw bytes.

| File | Raw bytes | gzip -6 | Brotli |
|---|---:|---:|---:|
| `_expo/static/js/web/entry-7dca206f5028938849f390d89eb7e831.js` (entry) | 2,435,331 (2.44 MB, 2.32 MiB) | 569,934 | 444,542 |
| `_expo/static/js/web/worker-a660570b6a4e1bb70b6d871433a76113.js` (worker) | 139,027 | 37,641 | 32,446 |
| `assets/.../expo-sqlite/web/wa-sqlite/wa-sqlite.7ca566fbbc2ec2a172c5aefa63a20f4b.wasm` (WASM) | 621,492 | 304,773 (level 6 only) | not measured |

For context, not part of the entry: `assets/assets/bible/kjv.a0fa34c0d361d32c2da47fab3f456506.yes` is 1,941,000 bytes. The export log lists it as an asset. The ticket quotes a worker of about 133 KB. The measured worker is 139,027 bytes.

---

## 3. Lighthouse results

Three runs per form factor. The median is the middle run. Min and max are the lowest and highest of the three. Run-to-run values are in section 4.

| Metric | Mobile median (min to max) | Desktop median (min to max) |
|---|---|---|
| Performance score | 60 (57 to 63) | 78 (78 to 78) |
| LCP | 23,103 ms (23,102 to 23,103 ms) | 3,803 ms (3,763 to 3,803 ms) |
| TBT | 384 ms (309 to 503 ms) | 23 ms (16 to 24 ms) |
| CLS | 0.039 (0.039 to 0.039) | 0.064 (0.064 to 0.064) |
| FCP | 1,202 ms (1,202 to 1,202 ms) | 282 ms (282 to 282 ms) |
| Speed Index | 6,059 ms (5,983 to 6,146 ms) | 1,359 ms (1,339 to 1,380 ms) |

Other category scores, same runs: Accessibility 93 on both form factors (all three runs), Best Practices 100 on both, SEO 90 on both.

Saved reports, run 1 of each form factor: `docs/proof/issue-14/lighthouse-mobile-run1.json`, `docs/proof/issue-14/lighthouse-mobile-run1.html`, `docs/proof/issue-14/lighthouse-desktop-run1.json`, `docs/proof/issue-14/lighthouse-desktop-run1.html`.

---

## 4. Environment

- Host: Windows 11 Home Single Language, build 10.0.26300.
- Browser: Google Chrome 155.0.8059.82 (`C:\Program Files\Google\Chrome\Application\chrome.exe`), run as HeadlessChrome/155.0.0.0.
- Lighthouse 13.5.0 through `chrome-launcher`. Node v26.11.1. Expo CLI 57.0.20. Expo SDK 57.
- Mobile emulation: 412x823, deviceScaleFactor 1.75, mobile true. Throttling method `simulate`: RTT 150 ms, throughput 1638.4 Kbps, CPU slowdown 4x.
- Desktop emulation: 1350x940, deviceScaleFactor 1, mobile false. Throttling method `simulate`: RTT 40 ms, throughput 10240 Kbps, CPU slowdown 1x.
- Cache: Lighthouse's default cold load, one fresh Chrome per run. The small spread in FCP, LCP, and CLS is consistent with simulated throttling, which derives throttled timings from one observed trace. That reason is an inference, not measured here.
- Run order: all three mobile runs, then all three desktop runs. Each run started about eight seconds after the last.
- Page errors during the screenshot load: 0.

---

## 5. Top Lighthouse opportunities and diagnostics

Lighthouse lists one audit as an opportunity with savings, and it appears on both form factors:

- `unused-javascript` on the entry bundle. Estimated savings 1,069 KiB. Measured on mobile run 1: 1,095,113 wasted bytes out of 2,435,330, about 45 percent of the entry.

Failing diagnostics on mobile run 1: LCP (score 0, 23.1 s), TBT (0.58, 500 ms in that run), Speed Index (0.44, 6.1 s), Time to Interactive (0.01, 23.1 s), network dependency tree (0), and total byte weight (0.5, 4,989 KiB across 7 network requests). Mobile had 4 long tasks of 285, 241, 95, and 82 ms.

Failing diagnostics on desktop run 1: LCP (0.19, 3.8 s), Speed Index (0.88, 1.4 s), Time to Interactive (0.64, 3.8 s), network dependency tree (0), `unused-javascript`, and total byte weight (4,989 KiB).

LCP element, mobile run 1: a verse text span (`And God called the light Day, and the darkness he called Night...`, verse 5). The LCP breakdown reports time to first byte of 2.6 ms and element render delay of 408.7 ms. Those two subparts do not add up to 23.1 s. This note does not attribute the rest. Desktop run 1 has the same element type and a render delay of 417.5 ms.

Accessibility and SEO failures, present in every run, were not part of step 1. They are listed for step 5 and step 6 to pick up. Accessibility: `aria-allowed-attr` (score 0) and `label-content-name-mismatch` (score 0). SEO: `meta-description` (score 0). Best Practices: `valid-source-maps` (score 0).

---

## 6. Unknowns

- The mobile LCP of 23 s is much larger than the FCP of 1.2 s, and the breakdown does not explain it. The next step is a trace of the mobile run, which is not part of step 1.
- These numbers come from localhost. The Funnel path, with TLS and the tunnel, is not measured.
- The Lighthouse transfer sizes are uncompressed. A compressed server would change `total-byte-weight` and the timings on the wire.
