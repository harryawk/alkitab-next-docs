# Migration progress log, 2026-10-09

Written to answer one question: where is the migration of `yukuku/androidbible` (Upstream) to this Expo app, and what has changed, by whom, and where does each change live.

How this was built. Branch history comes from `git log`. Ticket state comes from the `Closed:` line in each ticket under `.scratch/`, because the `Status:` lines are stale (they still read `ready-for-agent` on closed tickets). Test counts were measured this session. Nothing here has been verified on a device by me. Where a claim is inferred, it says so.

## 1. The destination

`.scratch/androidbible-parity/spec.md` and ADR 0001 define success as **Parity** with Upstream on iOS, Android and web. Parity means the same user-visible behaviour and the same data formats, not identical pixels. The spec splits the work into five phases.

| Phase | Scope | State |
|---|---|---|
| 1 | YES file engine, Version API, verse markup, ARI addressing, import of YES files | Mostly done. Import of YES files is not built. |
| 2 | Markers, Labels, Progress marks, History, export and import of user data | Markers and Labels done except highlights and gutter icons. Export and import built but unmerged. Progress marks and History deferred by decision. |
| 3 | Search | Done except operators (built as WIP, unmerged). |
| 4 | Reading plans, songs, audio | Not started. Ticketed only when reached. |
| 5 | Android-only equivalents (widget, notifications, volume-key paging) | Not started. |
| Extra | Reader UI parity with Upstream plus easier navigation (`.scratch/androidbible-parity/roadmap.md`) | Phases A, B and C done on a branch. Highlights open. |
| Tracking | Parity matrix `docs/parity-matrix.md` and the drift job | Neither exists yet. |
| Parked | Online features (sync, devotions, hosted content) | Parked by design. |

## 2. Where the code lives right now

There are three places, and they are not merged together.

1. **`master`** at `5240003`. Merged through PR #1 and a follow-up chore PR #2. This is the oldest and smallest set.
2. **`feat/reader-navigation`**, your current branch, 19 commits ahead of `master`. It is draft PR #5, stacked on draft PR #3 (`feat/release-workflow`, the CI release workflow), which targets `master`. Neither is merged, so `master` gets nothing from them until #3 and then #5 land, in that order. The branch contains all the reader UI parity work.
3. **Two unmerged side branches** from this session, now rebased onto `feat/reader-navigation`. They are `worktree-agent-acc650926436e09a9` (highlights) and `worktree-agent-af67814cd9e35e14a` (export and import).

Other local branches (`feat/search-screen`, `feat/release-workflow`, `feat/user-data-repository`, `feat/yes-parser-phase1`, three `research/*`) are earlier working branches. Two (`feat/user-data-repository`, `feat/yes-parser-phase1`) are fully contained in `master`. The others are ahead of `master` by a few commits, most of which look already represented on `feat/reader-navigation`. I did not audit them for unique work.

## 3. What is on `master`

Dates are commit dates.

| Date | Commit | What changed |
|---|---|---|
| 2026-08-28 | `b5fd150` | Initial commit, the Expo template. |
| 2026-10-02 | `960b355` | YES2 engine (reader, writer, Bintex, Snappy), the parity spec and ADRs, and the `openYes()` Version entry point. |
| 2026-10-03 | `a9c0a24`, `66defa3`, `0fbf62c` | Requirements for search, Markers and Labels, and export and import. |
| 2026-10-03 | `8f87393`, `4da1588`, `dd826a1` | Spec and tickets for Markers, Labels, export and import, and search. |
| 2026-10-03 | 16 commits, `25eeb44` to `8aa9a98` | The user-data repository (ticket 01). Markers, Labels, links, schema versioning, an `expo-sqlite` adapter, and several review-fix rounds. |
| 2026-10-04 | `7248b4b` (PR #1) | Search screen, verse markup rendering and pericopes, bookmark a verse, notes, the Markers hub and Marker list, and native verification. 32 files, +1762 lines. |
| 2026-10-04 | `5240003` (PR #2) | Closes the tickets that PR #1 merged. |

Tickets closed by `master`: yes-parser 01, 03, 04, 06, 08; markers-export-search 01, 02, 03, 06, 09.

## 4. What is on `feat/reader-navigation` but not on `master`

All dated 2026-10-05 to 2026-10-09.

| Commit | What changed |
|---|---|
| `c11d420`, `468f228` | CI. A manual release workflow for web and Android, plus an unsigned iOS `.ipa`, and research on GitHub Actions limits. |
| `0770e50` | Roadmap and 13 tickets for reader UI parity. |
| `afa5f67` | Previous and next chapter. |
| `715af84` | Resume the last read place, and `/reader?ari=` links. |
| `a972e11` | Book, chapter and verse grid sheet, replacing the chapter picker. |
| `f49dc8e` | Type a reference to jump, with English abbreviations. |
| `d3e1cd2` | Recent places and a back chip after jumps. |
| `25fa963` | Copy and share selected verses, plus a settings store. |
| `fe4fdb4` | Reading appearance, which covers text size, spacing, verse numbers, theme and keep screen on. |
| `7852fe3` | Footnote and cross-reference popups. |
| `1f67fe5` | Search scope chips, recent searches, book headings and centred snippets. |
| `5971af5` | Swipe between chapters, a header showing the verse in view, and web chapter keys. |
| `33aa239` | App shell. A drawer replaces the template tabs. |
| `ac7f32a` | Settings screen. |
| `f180ae2` | Marker list sort, text filter and multi-select with bulk copy and delete. |
| `f01bcd6`, `ca84299` | Close tickets 01 to 13 and give one slow test more time. |
| `8534c30` | Version registry model, a design decision (ticket 04). No code. |

Tickets closed here: androidbible-parity 01 to 13.

Measured on this branch before my changes: 27 test suites, 227 tests, all passing; `tsc --noEmit` printed nothing.

## 5. What I changed this session

Everything below is uncommitted or on side branches. Nothing is on `master` and nothing is merged into your branch.

**Main checkout (uncommitted, on `feat/reader-navigation`).**
- `app.json`: `web.output` from `"static"` to `"single"`. Cause: in dev, static output renders every route on the server, and that server bundle pulls in `expo-sqlite`'s web worker, which Expo's bundler cannot build. Every route returned 500 with "Worker chunk not found". Measured: after the change `/reader` and `/prototype-versions?variant=A` return 200, and `expo export --platform web` still builds and emits the worker file. Consequence: web output is now one `index.html`, so a host needs a fallback to it for direct links, and the cross-origin isolation headers `expo-sqlite` needs are still required. This is a call you may want to revisit.
- `app/_layout.tsx`: a hidden drawer entry for the prototype screen. Made by the earlier agents, not by me.
- Untracked: `app/prototype-versions.tsx` (three throwaway Versions UI variants), `.scratch/androidbible-versions/` (the Versions design map and decisions), `error-logs/`.
- New this turn: this file.

**Side branch `worktree-agent-acc650926436e09a9`, one commit.**
- Highlights (ticket 04). 12 preset colours plus a hex picker, partial and legacy captions rendered. 9 new behaviour tests. Rebased onto your branch; I resolved four conflicts in `app/reader.tsx` by hand and added a catch so a store that fails to open no longer breaks chapter reading.
- Measured: 247 tests pass across 29 suites with the unfinished gutter-icons test excluded.

**Side branch `worktree-agent-af67814cd9e35e14a`, two commits.**
- Export (ticket 07) and import (ticket 08) of Markers and Labels in an Upstream-shaped file. Native and web file paths. Adds `expo-document-picker` and `expo-sharing`.
- Measured: 258 tests pass across 30 suites; `tsc` printed nothing.

**Not committed anywhere.**
- Gutter icons (ticket 05): half-built, saved as `gutter-wip-reader.patch` in the session scratchpad. Two untracked files remain in the highlights worktree.
- Search operators (ticket 10): half-built, saved as `operators-wip.patch`.
- Both were written against the old `master` reader and search, and did not merge cleanly. Treat them as reference, not as a base.
- Two `autostash` entries sit in the shared git stash from my rebases. They duplicate the patches and can be dropped.

**Mistake worth knowing about.** I started the two agents with worktree isolation, and the worktrees branched from `master`, not from `feat/reader-navigation`. That is why the work needed rebasing. It cost conflicts and a reinstall. It was my launch error, not the agents'.

## 6. Ticket scoreboard

Closed on a branch means done and tested there, but not on `master`.

| Group | Closed on master | Closed on `feat/reader-navigation` | Done on side branches, unmerged | Open |
|---|---|---|---|---|
| yes-parser (8) | 01, 03, 04, 06, 08 | none | none | 05 reader memory and UTF-8, 07 import YES files, and 02 deferred |
| markers-export-search (10) | 01, 02, 03, 06, 09 | none | 04 highlights, 07 export, 08 import | 05 gutter icons, 10 search operators |
| androidbible-parity (13) | none | 01 to 13 | none | none |
| androidbible-versions (decisions) | none | 01 to 05 and 07 resolved | none | 06 Versions UI, 08 book-set differences, both claimed |

## 7. What is left, in the order I would do it

1. **Land what exists.** Merge #3, then #5, then the two side-branch PRs (highlights, export and import) into `master`. Today `master` lacks 19 commits of reader work, which is the bulk of what a user would notice.
2. **Finish phase 2 and 3 tickets.** Gutter icons (05), search operators (10). Both need a fresh start on the new base.
3. **Versions.** This is the largest remaining gap in phase 1. It needs your pick of Versions UI variant A, B or C, then decision 08, then yes-parser 07 (import) and 05 (reader memory, since imported Bibles can be large). Importing is native only by decision. Web import is a separate unplanned effort.
4. **Close the tracking gap.** There is no `docs/parity-matrix.md`, so "how far along" is only answerable by reading tickets, which is why the status lines drifted. The matrix and the drift job are in the spec and not started.
5. **Progress marks and History.** Deferred out of the phase 2 first cut by decision, still unspecified.
6. **Phases 4 and 5.** Reading plans, songs, audio, widget, notifications, volume-key paging. Not ticketed.
7. **Unverified risks.** Web storage on `expo-sqlite` is alpha and its contract tests run only against `sql.js` (ADR 0003), so real web persistence is unproven. iOS and Android were checked by earlier agents and by you for the first slice only. The reader-parity work has jest coverage but no recorded device pass.

## 8. Rough completion estimate

This is my judgment, not a measurement. Of the 30 implementation tickets in the three ticket groups (yes-parser ticket 02 is deferred and left out), 23 are closed on a branch, 3 more are built on side branches, and 4 are open. By user-visible feature, reading, navigation, search and basic Markers are in place. The two biggest missing pieces are Versions (import and switching) and everything in phases 4 and 5. I would call phases 1 to 3 roughly 80 percent done and the whole migration well under half, since phases 4 and 5 are large and unscoped.
