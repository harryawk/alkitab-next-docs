# How Upstream searches a Version

Research date: 2026-10-03. Source read: `https://github.com/yukuku/androidbible` at commit `f95ffb89371ddd3b2737f59315b1d61a3df37098` (the pin used by `docs/research/yes-file-format.md`), fetched at that exact commit and read directly. Terms follow `GLOSSARY.md`: **Upstream**, **Version**, **YES file**, **ARI**, **Verse markup**, **Pericope**.

Convention: matches `docs/research/` (see `yes-file-format.md`). Findings only, no recommendations.

Legend: **Fact** = read from source, cited as `path:Lnn` relative to the Upstream repo root (permalink form `https://github.com/yukuku/androidbible/blob/f95ffb89371ddd3b2737f59315b1d61a3df37098/<path>#L<n>`). **Derivation** = inference from the cited code. Nothing here was executed: no JVM build, no benchmark. Upstream's own prose docs (`docs/modules/search.md`, `docs/text-rendering.md`) were used only as pointers; where they disagree with code, the code wins and the disagreement is listed in section 8.

---

## 0. Where the code lives

| Concern | Files |
|---|---|
| Engine | `Alkitab/src/main/java/yuku/alkitab/base/util/SearchEngine.kt` |
| Query parsing | `Alkitab/src/main/java/yuku/alkitab/base/util/QueryTokenizer.kt`, `SearchEngineQuery.kt` |
| Screen and result rows | `Alkitab/src/main/java/yuku/alkitab/base/ac/SearchActivity.kt` |
| Chapter text supply | `AlkitabModel/.../model/Version.java`, `Alkitab/.../base/model/VersionImpl.java`, `AlkitabYes2/.../yes2/Yes2Reader.java`, `.../yes2/io/Yes2VerseTextDecoder.java`, `AlkitabIo/.../io/Utf8Decoder.java` |
| Markup stripping | `Alkitab/.../base/util/FormattedVerseText.kt`, `Alkitab/.../base/verses/VerseTextCompose.kt` |
| Tests that pin behaviour | `Alkitab/src/test/java/yuku/alkitab/base/util/SearchEngineTest.kt` |

There is a single search implementation, `SearchEngine.searchByGrep` (`SearchEngine.kt:38`). It is the only caller-visible entry point used by the screen (`SearchActivity.kt:582`).

---

## 1. Query parsing (operators)

**Fact.** `QueryTokenizer.tokenize(query)` (`QueryTokenizer.kt:21-60`):

1. Null or empty query returns no tokens (`:22`).
2. The whole query is lowercased with `String.lowercase(Locale.getDefault())` (`:26`).
3. Tokens are cut by the regex `(\+?)((?:["“”].*?["“”]|\S)+)` (`:8`): a run of non-whitespace characters, where a quoted span (ASCII `"` or curly `“` / `”`, in any pairing) may contain whitespace. An optional leading `+` is captured separately.
4. Each raw token is then peeled in a loop (`:36-52`): a leading `+` sets "plussed" and is dropped; a token that both starts and ends with a quote character has both removed and is plussed; a token that only starts with a quote has that one quote removed and is plussed. Empty remainders are discarded (`:54`). The result carries a single leading `+` for plussed tokens (`:55`).

The doc comment gives the intent (`:11-19`): a bare word means "contains"; a quoted word means exact word; several quoted words mean that exact ordered sequence.

Operators that exist, therefore:

| Syntax | Meaning | Source |
|---|---|---|
| `word` | substring match | `SearchEngine.kt:218`, `:272` |
| `+word` | whole-word match | `QueryTokenizer.kt:38-41`; `SearchEngine.kt:214` |
| `"a b c"` (also curly quotes) | ordered whole-word phrase, same verse | `QueryTokenizer.kt:42-45`; `SearchEngine.kt:211` |
| `+a +b` as separate arguments | NOT joined by the code: two separate whole-word tokens, ANDed | `QueryTokenizer.kt:8` splits on whitespace; see section 8 |
| whitespace between tokens | AND, all tokens in the same verse | `SearchEngine.kt:71-82` |
| unbalanced leading quote | one quote dropped, token plussed | `QueryTokenizer.kt:46-49` |

Not present anywhere in the tokenizer or engine: OR, NOT or a minus operator, wildcards, regex, fuzzy matching, proximity, field or book-name syntax in the query. (Fact by absence: the two files above contain no handling for them.) A `+` that is not at the start of a token (`a+b`) is not an operator and stays in the token text. Book scoping is not part of the query string; it is a separate filter (section 5).

Whole-word tokens split into words: `tokenizeMultiwordToken` (`QueryTokenizer.kt:97-106`) runs `[\p{javaLetterOrDigit}'-]+` (`:9`) over the token. Apostrophe and hyphen count as word characters (comment `:90-95`). Fewer than two words returns null, so a single plussed word, even `self-aware`, takes the plain whole-word path (`SearchEngine.kt:207-215`).

Tokens are then sorted longest first, ties alphabetical (`SearchEngine.kt:42-50`), and adjacent duplicates removed (`:53-64`). The sort is by string length including the leading `+` (Derivation from `o1.length`, `:43-44`).

---

## 2. Matching rules

### 2.1 Case

**Fact.** Matching is done on lowercased text with a lowercased query. The haystack is lowercased while decoding, not while searching: the YES2 UTF-8 decoder lowercases ASCII `A-Z` by `| 0x20` and every other code unit with `Character.toLowerCase(int)` (`AlkitabIo/.../Utf8Decoder.java:136-177`, lowercase branches `:142`, `:157-161`, `:171-175`). The query side uses `Locale.getDefault()` (`QueryTokenizer.kt:26`) while the haystack uses locale-independent `Character.toLowerCase`. Upstream records the mismatch as open tech debt, with Turkish as the example (`docs/tech-debt-remediation/REM-39-text-decoding.md:11-13`, "Not started", `:9`). `satisfiesTokens` is explicitly case sensitive and expects both sides pre-lowercased (`SearchEngine.kt:255-258`; test `SearchEngineTest.kt:142`).

### 2.2 Diacritics

**Fact.** No diacritic folding. No `Normalizer`, no accent stripping, no transliteration exists in `Alkitab/src/main`, `AlkitabModel/src`, or `AlkitabIo` (grep for `Normalizer|diacrit` found nothing relevant). Matching is raw UTF-16 code unit equality after lowercasing, so `e` does not match `é`, and a precomposed character does not match its decomposed form (Derivation from `String.indexOf` use, `SearchEngine.kt:218`, `:247`, `:294`). Non-ASCII scripts are found as substrings and whole words by Unicode letter class (tests `SearchEngineTest.kt:237-252`). The CHANGELOG records only that non-ASCII text became searchable (`CHANGELOG.md:476`, 4.4 Beta 2; `:1123`, 3.4.1 "Non-English Latin characters are now searchable").

Limits of the decoder: it handles code points up to U+FFFF only; 4-byte UTF-8 sequences are not decoded (`Utf8Decoder.java:7-8`, 3-byte maximum at `:165-176`).

### 2.3 Substring (no plus)

`String.indexOf(token)` on the lowercased chapter string (`SearchEngine.kt:218`, `:247`). `love` matches `beloved` (`docs/modules/search.md:34` agrees; tests `SearchEngineTest.kt:127`, `:503`).

### 2.4 Whole word (`+word` or a single quoted word)

`indexOfWholeWord` (`SearchEngine.kt:289-316`): find the substring, then require

- left: position 0, or the previous char is not `Character.isLetterOrDigit`, or the previous char is a letter/digit but the char before that is `@` (a Verse markup code letter immediately before the word counts as a boundary) (`:298-305`; tests `SearchEngineTest.kt:206-221`);
- right: end of text, or next char is not letter-or-digit (`:309`).

Apostrophe and hyphen are NOT letters, so `+love` matches in `love-song` and `love's`, even though the multiword tokenizer treats them as word characters (Derivation from `:298`, `:309` vs `QueryTokenizer.kt:9`). Digits count as letters (test `SearchEngineTest.kt:280`).

### 2.5 Phrase (`"a b c"`)

`indexOfWholeMultiword` (`SearchEngine.kt:329-398`): whole-word match of the first word; then for each following word, skip forward over any non-letter-or-digit characters and over Verse markup (a `@` plus one code character, or `@<` up to the next `@>`, `:348-363`), then require the next whole word to start exactly at that point (`:383-389`). In chapter mode the verse separator `\n` aborts the attempt, so a phrase never spans verses (`:366-369`; test `SearchEngineTest.kt:574`). Consequences: punctuation and extra spaces between words are tolerated (test `:298`), word order is enforced (`:322`), another word in between fails (`:328`).

### 2.6 Multiple tokens

AND within the same verse (section 5 for how). Order of tokens does not matter (tests `SearchEngineTest.kt:159`, `:522`).

---

## 3. How Verse markup is handled (stripped or not)

This differs between matching and display.

### 3.1 Matching: NOT stripped

**Fact.** The haystack is the raw chapter text with all Verse markup intact, only lowercased. `Version.loadChapterTextLowercasedWithoutSplit` "loads a whole chapter as a single string with verses separated by '\n'" (`AlkitabModel/.../model/Version.java:70-74`). The YES2 implementation calls `loadVerseText(book, chapter_1, true, true)` (don't separate, lowercase) (`VersionImpl.java:301-313`) and the decoder only converts bytes to chars, lowercases, and appends `\n` after every verse (`Utf8Decoder.java:113-184`, separator `:180`). No `removeSpecialCodes` call occurs on the search path (`SearchEngine.kt` has no reference to `FormattedVerseText`).

Evident effects, pinned by tests:

- A substring or whole-word token matches text that is inside a markup span, e.g. `ancient` inside `@<ref@>ancient oak@/` (`SearchEngineTest.kt:664-679`).
- A phrase can span plain text into span content (`:691`) and AND tokens can sit inside and outside a span (`:701`).
- Derivation: a substring token also matches the letters of markup itself, such as the attribute text inside `@<...@>` (for example `ref` in `@<ref@>`, or `r=reading` for ruby per `docs/features/ruby/design.md` as pointer only), because nothing removes them before `indexOf` (`SearchEngine.kt:218`). Single-letter code characters after `@` are matchable the same way.
- The only markup awareness in matching is: the `@x` left-boundary allowance (`:299`) and the tag skipping inside phrases (`:348-363`).

### 3.2 Display: stripped, per result, at render time

**Fact.** Result rows reload the unlowercased verse with `searchInVersion.loadVerseText(ari)` (`SearchActivity.kt:760`) and then:

- Legacy View path: `FormattedVerseText.removeSpecialCodes(rawVerseText)` (`SearchActivity.kt:769`). That function (`FormattedVerseText.kt:11-53`) returns the input unchanged if it is null, empty, or does not start with `@` (unless `force`) (`:12-14`); otherwise removes every `@` plus the one following character, and for `@<` also drops everything up to the matching `@>` (the tag's attribute header, content between `@>` and `@/` stays) (`:31-37`), and for codes `0 1 2 3 4 ^ 8` (paragraph, indent, newline family) inserts one space unless the output already ends in whitespace (`:40-44`).
- Compose path: `renderVerseText(ari, rawVerseText)` (`SearchActivity.kt:778`, `VerseTextCompose.kt:231-238`) uses the full verse renderer, so the snippet keeps styling (red letter, italics) rather than plain text.

Highlighting runs on the stripped/rendered string: `SearchEngine.hilite` (`SearchEngine.kt:401-461`) lowercases it char by char (`:409-413`), then re-runs the same token matchers (`indexOfWholeMultiword` called with `isNewlineDelimitedText=false`, `:429`), picks the earliest hit among tokens (`:441-452`) and bolds and colors it (`:455-456`). `withSearchHilite` copies those spans onto the Compose string, relying on equal offsets (`VerseTextCompose.kt:239-257`).

Derivation: because matching used raw text and highlighting uses stripped text, a verse that matched only through markup text appears in the results with no highlighted span. Not tested in the files read.

---

## 4. Result shape and context

**Fact.**

- Engine output: `IntArrayList` of **ARIs**, verse level, no text, no score, no match offsets (`SearchEngine.kt:38`, `:84`). Within one token pass ARIs are appended in book (consecutive books order), chapter, verse order (`:151-159`, `res.add(ariBc + verse0 + 1)` at `:235`), and the intersection step assumes both lists ascending (`:98-110`).
- A verse appears at most once per token however many times it matches (`lastV` guard, `:199`, `:234-237`; test `SearchEngineTest.kt:629`).
- `SearchActivity` stores the list in the adapter (`SearchActivity.kt:842-850`) and shows a snackbar with the count (`:610-611`). No limit, paging or cap on result count appears in either file.
- Each row: a reference line `searchInVersion.reference(ari)` (`:754`) and a snippet that is the whole verse, highlighted (`:760-781`). There is no surrounding context (no neighbouring verses, no truncation, no keyword-in-context window) and no **Pericope** title in the row. Reference formatting is Version-specific (book names come from the **Version**).
- Rows support long-press multi-select and copy using stripped text (`SearchActivity.kt:125-137`), and tapping opens the reader at that **ARI** with the verse selected (`:819`).
- Zero results: a "No verses found for ..." message (`strings.xml:325`); if the query parses as a verse reference (`Jumper`) and that reference exists in the **Version**, the message adds a tap-to-open link to it (`SearchActivity.kt:619-653`); if no books are selected a different message shows (`:656-659`). An empty or blank query does nothing (`:670`).
- Search history: up to 20 queries, deduped, newest first (`SearchActivity.kt:700-716`).
- Tips shown before searching: "`+word` for that exact word", "`"city of david"` for that exact phrase" (`Alkitab/src/main/res/values/strings.xml:324`).

---

## 5. Index or scan

**Fact.** Scan. No index is built, read or shipped in the current code. The engine is named `searchByGrep`, and `docs/modules/search.md:10` calls it "grep-based". The algorithm (`SearchEngine.kt:68-85`, `:141-187`):

1. Tokens are processed one at a time, longest first.
2. First token: for every book of the **Version** that is in the book filter, for every chapter, load the chapter as one lowercased string and scan it (`:150-162`, `:195-253`). Verses are located by counting `\n` positions between hits (`:226-232`); the chapter string is scanned with `indexOf`, not verse by verse.
3. Later tokens: only the book-chapters that contained a hit for the previous intersection result are loaded again (`nextAri`, `:120-139`; loop `:164-183`), and the new hit list is intersected with the previous one (`:77-81`, `:87-113`). The engine does not test all tokens per verse; each token is a separate pass over chapters. (`docs/modules/search.md:65` states the same.)
4. A chapter that fails to load returns null and is skipped (`:196`).

Book filter: `SearchEngineQuery.bookIds` is a `SparseBooleanArray` (`SearchEngineQuery.kt:12-13`); `searchByGrep` falls back to an empty set when null (`SearchEngine.kt:66`), and an empty set searches nothing (`:152`; test `SearchEngineTest.kt:605`). The screen builds it from Old Testament, New Testament and "this book" checkboxes with hard-coded book id ranges `0..38` and `39..65` (`SearchActivity.kt:530-538`), plus a per-book picker (`:542`). All books selected by default (`:325-327`). Books are ids from the **Version**'s `consecutiveBooks` (`SearchEngine.kt:151`).

History of indexing in Upstream (CHANGELOG, as pointer only): first search used SQLite FTS3 with a ~9 MB index taking about 5 minutes to build on a Nexus One, sub-second afterwards (`CHANGELOG.md:1691-1695`, 0.8, 2010-03-26); a scan-based engine with no pre-indexing arrived in 1.0 (`:1667`, 2010-08-04); a preloaded reverse index was later shipped for the first search (`:1269`, 2.9.3, 2013-02-06; `:1340`, 2.8.0); the "search reverse-index" was removed in 4.9.0 (`:159`, 2023-06-08, "search still works and translation downloads are smaller"). No FTS or index code remains in the tree (grep for `FTS|fts3|fts4` found none in `.java/.kt/.sql`).

---

## 6. Cost on large Versions

**Documented.** Upstream documents no latency numbers, no complexity statement, and no limit for current search. `docs/modules/search.md:51` calls longest-first ordering "an optimization so the most selective token is searched first", and `:65` says the first token prunes later passes. The only timing instrumentation is debug logging: per-token milliseconds (`SearchEngine.kt:73-75`), intersection sizes (`:78-80`), per-book progress (`:161`), chapters re-read for later tokens (`:183`), and total and CPU time (`SearchActivity.kt:580-592`), all through `AppLog.d`, no recorded results. The only quantitative statements about search are historical CHANGELOG lines in section 5.

**Evident from the code (Derivation).**

- Work for the first token is proportional to the amount of text in the selected books of the **Version**: every chapter of every selected book is fetched, decoded and scanned. Every later token re-fetches and re-decodes whole chapters that held earlier hits, so a common first token leaves many chapters to reread.
- Per chapter fetch: seek to the chapter offset, and for compressed **YES files** decompress the Snappy blocks that hold it (`Yes2Reader.java:72-93`; block seek and decompress `SnappyInputStream.java:36-57`, which keeps only the most recent block, `:52`). The writer's block size is 32768 bytes (`tools/AlkitabConverter/.../Yes2Common.java:178`). Decode then builds one `String` per chapter through shared thread-local buffers (`Utf8Decoder.java:13-15`, `:113-183`).
- `loadChapterTextLowercasedWithoutSplit` is `synchronized` on the `VersionImpl` (`VersionImpl.java:301`), as are `loadVerseText` and `loadChapterText` (`:150`, `:291`), so the reader, split view and search contend for the same lock (Derivation).
- Search work runs off the UI thread on a single-thread scheduled executor with zero delay (`SearchActivity.kt:577`; `Debouncer.kt` as read: `Executors.newSingleThreadScheduledExecutor()`, `submit` at `:55-73`). A newer submission only causes the older result to be discarded at "before process" (`:60`) or "after process" (`:65`) checks; an already running scan is not interrupted (Derivation).
- The whole result list is held in memory as ints (`IntArrayList`); each visible row then loads its verse by `loadVerseText(ari)`, which decodes and separates the entire chapter to return one verse (`VersionImpl.java:155-170`, `loadVerseText(book, chapter_1, false, false)` at `:156`) (Derivation).
- Phrase and whole-word matching loops call `indexOfWholeWord` repeatedly and restart from `pos + 1` after a failed boundary check (`SearchEngine.kt:302`, `:310`), so a very frequent token with many failed boundaries does repeated `indexOf` calls within one chapter string (Derivation).

---

## 7. Known defects touching search (Upstream's own audit)

- ASCII-encoded **YES files** (`textEncoding == 1`, also the fallback for unknown encodings, `Yes2Reader.java:218-225`): `Yes2VerseTextDecoder.Ascii.separateIntoVerses` lowercases with `verseBuf[i] |= 0x20` where `i` is the verse index rather than the byte index (`Yes2VerseTextDecoder.java:22-28`, bug at `:26`). Upstream's audit says search misses matches and corrupts a byte per verse in the search buffer, display unaffected, "Not started" (`docs/tech-debt-remediation/REM-36-yes2-ascii-decoder.md:8-10`). `makeIntoSingleString` for ASCII goes through this same method (`Yes2VerseTextDecoder.java:40-47`).
- `Utf8Decoder.buf` is a shared static buffer and the UTF-8 single-verse path swallows `ArrayIndexOutOfBoundsException` (`Utf8Decoder.java:11`, `:53-55`, `:106-108`); REM-39 lists this as a race (`REM-39-text-decoding.md`, Steps 1). The search path itself uses the thread-local `toStringFromVersesWithPrependedLengths` for chapters (`Yes2VerseTextDecoder.java:81-83`).
- Locale mismatch between query and haystack lowercasing: section 2.1.
- `Debouncer` leaks one idle executor thread per instance (`docs/tech-debt-remediation.md:54`, REM-46).

---

## 8. Where Upstream's prose docs disagree with the code

1. `docs/modules/search.md:24` says "Multiple consecutive `+word` tokens are joined into a single multi-word token" and gives `+in +the` becomes `["+in the"]` (`:25-26` region). The tokenizer has no joining step: the regex splits on whitespace (`QueryTokenizer.kt:8`) and `tokenize` returns one token per raw token (`:31-59`). `+in +the` yields `["+in", "+the"]`, two whole-word tokens ANDed anywhere in the verse. Only a quoted phrase produces a multiword token. The user-facing tips string mentions only `+word` and quoted phrases (`strings.xml:324`).
2. `docs/text-rendering.md:75-80` says "Search indexing" uses `removeSpecialCodes` and "search and copy must use the stripped text". Search matching uses raw lowercased text with markup (section 3.1); only the results display and copy strip.
3. `docs/modules/search.md:20-22` says quoted text gets a `+` prefix "(triggers whole-word matching)". Matches code (`QueryTokenizer.kt:42-45`).
4. `docs/modules/search.md:36` shows `+"in the beginning"`; the code also accepts `"..."` alone, and the `+` before a quote is absorbed (`QueryTokenizer.kt:36-52`).

---

## 9. Verification status

All claims were read from the pinned commit; nothing was run. Line numbers refer to that commit. Tests cited (`SearchEngineTest.kt`) were read, not executed.
