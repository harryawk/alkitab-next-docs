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
