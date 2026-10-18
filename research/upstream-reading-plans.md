# Upstream reading plan data (yukuku/androidbible)

Research date: 2026-10-18. Issue: #32 (reading plans). Upstream: `yukuku/androidbible`, branch `develop`, head `583d7fbd82d302fb3993c5c8e67421b1cd271b16` (committed 2026-10-16). Read through `gh api` contents calls with `ref=develop`. The `develop` head moved since the parity research (`3e3f6ad`, 2026-10-08), so these paths were read at the newer head.

## Decision

**Stopped. Plan data is not bundled in this build.** No UI was built.

Blocking reasons:

1. **No license for the plan content.** Upstream's `LICENSE.txt` is Apache License 2.0 and covers the code. The repo has no separate license for the plan files. The plan sources credit third parties: the ESV plans cite `www.esv.org/biblereadingplans`, and the Blue Letter plans are Blue Letter Bible's. Neither their terms nor a permission to redistribute are in the repo. The rule for bundling requires the license to permit it, and I could not establish that.
2. **Upstream does not ship the plan files in the app tree.** `docs/tools.md` calls the `tools/AlkitabConverter/file/` sources "the bundled plans", but no `.rpb` or `.rpa` file exists under `Alkitab/src/main/assets` or `res/raw`. `ReadingPlanActivity.downloadReadingPlanFromServer` fetches plans from a server (`alkitab://addon/download?kind=readingplan&type=rpb&name=...`). Whether a build step copies the converter sources into the APK is unverified. The `CopyProprietaryAssetsTask` in `Alkitab/build.gradle.kts` copies Bible assets from a proprietary overlay, and I did not check whether that overlay includes plans.
3. **The plan list comes from the server.** `ADR 0001` parks online features until the maintainer confirms the terms for `api.alkitab.app`. The plan list and download sit behind that.

The ticket still needs a plan list. The next step is for the maintainer to confirm the license of the plan files, or to name a plan that is public domain. Then this ticket can bundle that plan.

## Source paths (upstream, develop)

| Purpose | Path |
|---|---|
| Format doc, RPA authoring | `docs/developer/reading-plans.md` |
| Format doc, RPB app side | `docs/modules/reading-plans.md` |
| Converter docs | `docs/tools.md` (section "Reading plan converters") |
| RPA to RPB converter | `tools/RpaToRpb/src/yuku/readingplanconverter/RpaToRpb.java`, `RpaInput.java` |
| RPA source converter | `tools/AlkitabConverter/src/main/java/yuku/alkitabconverter/reading_plan/RpaConverter.java` |
| Reader of RPB | `Alkitab/src/main/java/yuku/alkitab/base/util/ReadingPlanManager.java` |
| Plan UI | `Alkitab/src/main/java/yuku/alkitab/base/ac/ReadingPlanActivity.java` |
| Plan sources | `tools/RpaToRpb/file/*.rpa`, `tools/AlkitabConverter/file/*.txt` and `*.rpa` |
| Binary samples | `tools/RpaToRpb/file/*.rpb`, `tools/RpbTester/file/*.rpb` |

Plan names in `tools/RpaToRpb/file/` (12 `.rpa` files): `backtothebible_chronological`, `bibleplan_gospels`, `bibleplan_proverbs`, `bibleplan_psalms`, `blueletter_one_year_canonical`, `blueletter_one_year_historical`, `blueletter_ot_and_nt_together`, `esv_daily_reading_bible`, `esv_everyday_in_the_word`, `esv_through_the_bible_in_a_year`, `mcheyne`, `wsts_twenty_minutes_a_day`. I read the header of four of them (below); the other titles are not verified.

## Format

Two formats exist.

**RPA (text, authoring).** UTF-8, LF line endings, tab-separated. Upstream's guide gives two forms.

- The current uploader form, from `docs/developer/reading-plans.md`:
  ```
  info	version	1
  info	title	<title>
  info	description	<one line>
  info	duration	<days>
  day	<n>	<range>	<range>...
  ```
  Each range is `Gen.1.1-Gen.1.10`, `Matt.1`, or a decimal or hex ARI. A day holds at most 127 ranges.
- The older desktop converter form, `plan\t<day>\t<startARI>\t<endARI>`, in `tools/RpaToRpb/file/bibleplan_psalms.rpa`.

**RPB (binary, runtime).** Read by `ReadingPlanManager.readVersion1`.

- A 7-byte header `52 8a 61 34 00 e0 ea`, then a version byte (1).
- A Bintex simple map with `title`, `description`, `duration`.
- For each of `duration` days: a `uint8` count, then `count` Bintex ints, which are ARI start/end pairs.
- A footer byte `00`.

The plan name is not in the file. The app stores it separately, and progress is keyed by `(dayNumber << 8) | sequenceIndex`.

Upstream stores progress in a `ReadingPlanProgress` table (`readingPlanId`, `readingCode`, `checkTime`). It syncs as `Sync_Rp`. This matches the DB shape in `docs/modules/reading-plans.md`.

## License

- Code: Apache License 2.0 (`LICENSE.txt` at the repo root).
- Plan content: no license statement found. A search for "reading plan licen" in the repo returned nothing. Attribution for the ESV plans points to `www.esv.org/biblereadingplans`. The Blue Letter plans name Blue Letter Bible.

Treat the plan content as unlicensed until the maintainer says otherwise.

## Size

- `tools/AlkitabConverter/file/esv_daily_reading_bible.txt`: 11,457 bytes, 365 days.
- `tools/RpaToRpb/file/esv_daily_reading_bible.rpb`: 9,350 bytes, 365 days.
- `tools/RpaToRpb/file/bibleplan_psalms.rpa`: 849 bytes, 31 days.
- All 12 `tools/RpaToRpb/file/*.rpa`: about 148 KB of source text in total (sum of sizes from the git tree).

Each plan is small and static. Size is not the blocker; the license is.

## Sample: two days

From `tools/AlkitabConverter/file/esv_daily_reading_bible.txt`, the plan named "ESV Daily Reading Bible" (365 days). Each day has three readings separated by `;`.

- Day 1: `Gen 1-2; Ps 1; Matt 1-2`
- Day 2: `Gen 3-4; Ps 2; Matt 3-4`

From `tools/RpaToRpb/file/bibleplan_psalms.rpa` (31 days). The ARI values decode as `book*65536 + chapter*256 + verse`, with book 18 = Ps.

- Day 1: `plan 1 1179904 1180928` = Ps 1:0 to Ps 5:0, i.e. Psalms 1-5.
- Day 2: `plan 1 1181184 1182208` = Ps 6:0 to Ps 10:0, i.e. Psalms 6-10.

ARI 0 verse means the whole chapter.

## What this means for the app

If the license is settled, the minimum bundled slice is one plan as a static `.rpa` or `.rpb` file, read at build time. Each day then becomes a list of passages, each one an OSIS reference that links to the reader.

## Gaps

- The plan list and downloads from `api.alkitab.app` are not researched. ADR 0001 defers them.
- Export and import of plan progress are not built. Upstream syncs progress as `Sync_Rp`. The app's export format is not yet designed for it. This is a gap for the Parity matrix.
- The "bundled" claim in `docs/tools.md` is not confirmed against a build step.
