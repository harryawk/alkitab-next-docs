# Decision log

Plain-language record of what was changed, why, and how it was checked.
Newest entries go at the bottom. Each entry names the commit that holds the change.

## 2026-10-10 Overnight run: starting point

**Where things stood.** The reader UI work sat on `feat/reader-navigation`, about 22 commits ahead of `master`. Three other branches held finished work that was not yet in it: highlights, markers export and import, and search with notes, bookmarks and the markers hub. The baseline test run was green: 227 tests passed and the type check was clean.

**Decision 1. Checkpoint the loose work first.** Three files were sitting uncommitted: a Versions prototype screen, its link in the app layout, and a roadmap page. They were committed as-is so nothing gets lost. Agent worktrees and error logs were added to `.gitignore`.
- Commit `80db8fd`, pushed to `origin/feat/reader-navigation`.

**Decision 2. Move `master` up to the feature branch.** The branch had only added commits on top of `master`, so this was a plain fast-forward push, with no force. The main branch now has the same code as the feature branch.
- `origin/master` is now `80db8fd`.

**Decision 3. Merge the highlights branch.** It touched the reader, a highlight sheet and a highlight service, with tests. It merged cleanly.
- Merge commit `ce6e82d`. Ticket 04 (highlights) is done, but its file still needs a `Closed:` line.

**Decision 4. Merge the markers export and import branch.** Export and import came in with a file-transfer service and tests. The tests failed at first because two packages (`expo-document-picker`, `expo-sharing`) were declared in `package.json` but not installed here. I installed them from the merged lockfile. Tests then passed.
- Merge commit `b2c0323`. 278 tests pass, type check is clean.

**Decision 5. Record the search-screen branch as merged without changing any files.** Git reported 14 conflicts. Checking the files showed the search-screen branch's tree is identical to master's squash commit `7248b4b` (PR #1), which already holds the same search, bookmark, note and markers work. Taking either side would give the same result, so I kept the current tree and recorded the merge with `-s ours`.
- Merge commit `92e535c`. No file changed.

**Decision 6. Merge the six research branches.** Each adds one document under `docs/research/`, and the specs point to these files. No conflicts.
- `origin/master` and `origin/feat/reader-navigation` are both at `68e235d`.

**Decision 7. Finish YES ticket 05 (reader memory, UTF-8, bad input).** The text section used to be fully unpacked on the first read, about 4 MB. Now it unpacks one 32 KB block at a time and keeps the last 8 blocks.
- A chapter is read in one go. First chapter went from 26 ms to 3 ms. Opening stays at 2 ms.
- Trade-off: a full pass over the whole Bible is about 5% slower (499 ms against 476 ms), because each pass unpacks the blocks again. Acceptable for a reader that opens instantly.
- Every section must lie inside the file. A file cut short fails at open with a "damaged" message. This was tested at every section boundary.
- A damaged compressed block is found when it is read, not at open. Checking every block at open would mean unpacking the whole text, which is what this change removes.
- 4-byte UTF-8 characters (emoji) now decode correctly. The Android reader gets them wrong, so this is a deliberate departure. The writer still refuses characters outside the BMP, as upstream does.
- Evidence: commit with `SectionBytes.ts`, `Yes2Hardening.test.ts`. 286 tests pass.

**Decision 8. Versions: store imports in the user-data database.** The installed list is a table with one row per Version. The bundled KJV is seeded as `internal` and cannot be removed. Imported files are named `file/<name>`, so the id survives a move of the app's container.
- A file is checked by opening it once before it is saved. Anything that fails leaves nothing behind.
- Duplicates are refused before any copy or download.
- Evidence: commit `1ab88ce`, `src/services/versions/__tests__/importVersion.test.ts` (16 tests).

**Decision 9. Versions UI: prototype variant A.** The three prototype screens were read side by side. I chose A: a Version chip in the reader header opens a sheet to switch, and the drawer gets a Versions screen for import and removal. This was a reading of the variants, not a user test.
- Evidence: commit `c21fd22`, `.scratch/androidbible-versions/decisions/06-versions-ui.md`.

**Decision 10. Switching Versions keeps the place where it can.** When the new Version has the open book, the reader stays there. If not, it goes to the next book in Bible order that the new Version has, at chapter 1. The Back chip is cleared, since its place may not exist in the new Version. Markers stay saved even when their book is missing from the open Version.
- Evidence: `placeInVersion` in `src/store/useBibleStore.ts`, `src/store/__tests__/placeInVersion.test.ts`.

**Decision 11. Compare skips missing files.** It lists the open Version first, then the rest in list order. A Version without the verse shows "Not available in this version." A Version whose file is gone is left out, as the chooser leaves it out.
- Evidence: `src/services/versions/compareVerse.ts`, `src/__tests__/ReaderVersions.test.tsx`.

**Decision 12. Search operators follow Upstream's documented rules.** Upstream's rules were read from `docs/research/upstream-search.md` and followed as written. `+word` and a single quoted word match whole words. A quoted phrase matches consecutive whole words, and punctuation between them is allowed. A bare word still matches anywhere, as before.
- What is not checked: the ticket asks for the same results as Upstream on a recorded sample. Upstream was not run here, so that checkbox is open. The tests cover the documented rules instead.
- Evidence: `src/services/search/search.ts`, `src/services/search/__tests__/searchOperators.test.ts` (21 tests).

**Decision 13. Gutter icons open the existing editors.** A bookmark icon and a note icon sit at the start of the verse where the Marker starts. Tapping one opens that Marker's dialog. The dialogs now take their verses from an explicit target instead of the current selection, so an icon tap does not need a selection first. Saving or deleting a Marker reloads the icons.
- Evidence: `app/reader.tsx`, `src/__tests__/GutterIcons.test.tsx` (4 tests).

**Decision 14. Maestro flows cover what a script can reach.** The Maestro CLI came from its official Homebrew tap (`mobile-dev-inc/tap/maestro`, 2.11.0). The Homebrew `maestro` cask is a different product and was not used. The flows live in `.maestro/`. Picking a file in the system picker is not automated, so the import success path is covered by Jest tests only.

**Note on tests.** Under load, `MarkersScreens`, `SearchScreen` and `VersionsScreen` sometimes fail in the full run, while passing alone. The cause is the iOS build saturating the CPU. `VersionsScreen` now waits up to five seconds. The other two are pre-existing timing-sensitive tests and pass when run on their own.

**Decision 15. The chooser was empty on the device, and a Jest test missed it.** The chooser read the installed list once at render time, so it never updated after loading. The Jest test filled the list by hand before opening it, which hid the bug. The reader now subscribes to the list, so the chooser fills in when it opens. The Maestro chooser flow caught this, and it is the reason to run flows on the real app.
- Evidence: `app/reader.tsx` (`installedVersions`), `.maestro/versions-chooser.yaml`.

**Decision 16. Maestro flows run on the simulator, all pass.** Four flows: the chooser, Compare, removing the KJV (refused), and a refused http link. Two of them needed fixes in the flow itself. Maestro matches an element's accessibility label, not its visible text, and the keyboard swallowed the first tap on a button. Maestro's own screenshot step did not write files, so the screenshots in `docs/progress/screenshots/` were taken with `simctl`.
- Screenshots: `versions-screen.png`, `compare-verse.png`, `refuse-http-link.png`. The chooser flow ends on the Versions screen, so no separate chooser shot exists.
- Full Jest suite on an idle machine: 336 tests pass, type check clean.

**Build note.** The Release build exits with status 1 after it installs and opens the app. The cause is an osascript call that checks for the Simulator window, which fails here. The app is installed and runs, so this does not block testing.
