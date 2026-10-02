# YES2 `xrefs` / `footnotes` sections and verse link markers (yukuku/androidbible)

Research date: 2026-10-02. Source: `https://github.com/yukuku/androidbible` at commit `f95ffb89371ddd3b2737f59315b1d61a3df37098`, read from a local clone. Follow-up to `docs/research/yes-file-format.md` (sections 4.5 and 6); where this document is more precise it supersedes them (see section 9).

Convention (same as the other `docs/research/` files): **Fact** = read from source or observed by running upstream code, cited as `path:Lnn` relative to the androidbible repo root (permalink `https://github.com/yukuku/androidbible/blob/f95ffb89371ddd3b2737f59315b1d61a3df37098/<path>#L<n>`). **Derivation** = my inference, or my own test/harness code, not upstream code.

Fixtures produced by running the upstream Java writer are under `src/services/yes2/__tests__/fixtures/` (section 4). They are the reference samples for xrefs/footnotes; `assets/bible/kjv.yes` has none.

---

## 0. Key findings

1. **Fact.** `xrefs` and `footnotes` are byte-for-byte the same structure: `u8 version=1`, `int count`, `int arifs[count]`, `int offsets[count]`, then `count` Bintex string values. Offsets are relative to the first byte after the offsets array. The two readers differ only by class names and the entry class (`XrefsSection.java`, `FootnotesSection.java`).
2. **Fact.** An entry's content is *formatted text*, not a bare reference list. Grammar: `@9..@7` italic, `@8` newline, `@<TAG@>shown text@/` tags, no `@@` header required. The only tag the app acts on is `t...` (target), with TargetDecoder syntax `a:`/`ari:`/`o:`/`lid:`. Real examples: `@<ta:3736580,3736586-3736587@>Book. 4:4, 10-11@/`, `@<to:Gen.2.1@>Dest 1@/`.
3. **Fact.** A verse marker `@<f1@>@/` / `@<x1@>@/` maps to `arif = (ari_of_the_containing_verse << 8) | field`, field 1..255, with footnote and xref field numbers independent of each other. The renderer shows a superscript of the *field number* (footnote) or `※` U+203B (xref).
4. **Fact (observed).** Real upstream-generated data exists: `Alkitab/src/plain/assets/internal/ddd_xrefs_bt.bt` and `ddd_footnotes_bt.bt` are raw section bodies, and `tools/in-ddd/in-ddd.yet` holds the `xref`/`footnote` lines they were built from. Running upstream's own writer on that `.yet` reproduces both `.bt` files **byte-for-byte**. I also generated a full `.yes` from a hand-written `.yet` using upstream `Yes2Common.createYesFile` compiled with plain `javac`; upstream's `Yes2Reader` reads it (section 4).
5. **Fact (observed).** The writer does NOT sort. `YetToYes2` passes `.yet` line order straight through, and `XrefDb`/`FootnoteDb` sort with a *signed* `TreeMap<Integer>`, which puts books >= 128 first and breaks the reader's *unsigned* search for the other books. Our reader must binary-search unsigned; our writer must sort unsigned (section 5).

---

## 1. Byte layout and algorithms

### 1.1 Layout (Fact)

Header comment, identical in both classes: `XrefsSection.java:11-19`, `FootnotesSection.java:11-19`.

```
uint8  data_format_version = 1          reader throws RuntimeException if != 1   (XrefsSection.java:37-40)
int    entry_count                      4 bytes BE, signed                        (:42)
int    arifs[entry_count]               4 bytes BE each, must be sorted ascending as UNSIGNED ints   (:44-47)
int    offsets[entry_count]             4 bytes BE each                           (:48-51)
value<string> contents[entry_count]     Bintex string values, back to back
```

- All ints are big-endian signed 32-bit (`BintexReader.readInt`, `BintexReader.java:136-140`; `BintexWriter.writeInt`, `BintexWriter.java:149-156`).
- Section position in the container: sections `xrefs`, `footnotes` come after `text` when present (`Yes2Common.java:165-171`); each is optionally `snappy-blocks` compressed (`Yes2Common.java:286-330`). The reader's `prepareLoadSection` returns a stream over the *logical (decompressed)* content, already positioned at the section start (`Yes2Reader.java:315-341`). So every offset below is a logical offset.
- **Offsets are relative to `content_start_offset`**, which is the stream position right after the offsets array: `this.content_start_offset = (int) input.getFilePointer()` (`XrefsSection.java:53`). It equals `1 + 4 + 8*count` bytes from the section start (**Derivation**). Lookup does `abs_offset = content_start_offset + index_offset[pos]` then `seek(abs_offset)` (`XrefsSection.java:63-67`).
- Each content is one Bintex *value* string (`BintexReader.readValueString`, `BintexReader.java:280-376`): `0c` = null, `0d` = empty, `51..5f` = 8-bit string of length 1..15, `61..6f` = 16-bit string of length 1..15, `70 len8` / `71 len8` = 8-/16-bit string with 1-byte length, `72 len32` / `73 len32` = long 8-/16-bit. 8-bit strings are Latin-1 (`new String(buf, 0x00, 0, len)`, `BintexReader.java:360`: high byte 0 => each byte is a UTF-16 unit 0x00..0xFF, **not UTF-8**); 16-bit strings are UTF-16BE code units, read one char at a time (`:371-375`), so supplementary characters are surrogate pairs. The writer picks 8-bit only if every char is <= 0xFF (`BintexWriter.java:304-345`). Lengths count UTF-16 code units, not bytes or code points.

### 1.2 Reader algorithm (Fact)

Constructor (`XrefsSection.java:32-55`): read version, count, read `count` ints into `index_arifs`, `count` ints into `index_offset`, remember `content_start_offset` and the stream. Contents are **not** read up front.

Lookup `getXrefEntry(arif)` (`XrefsSection.java:57-76`):

1. `pos = unsignedIntBinarySearch(index_arifs, arif)`; `pos < 0` => return `null`.
2. `seek(content_start_offset + index_offset[pos])`, `readValueString()`, wrap in `XrefEntry{content}`. On `IOException` log and return `null`.

`unsignedIntBinarySearch` (`AlkitabYes2/src/main/java/yuku/alkitab/yes2/util/unsignedBinarySearch.kt:8-30`): classic binary search with `mid = (low + high) ushr 1`, comparing `(a xor Int.MIN_VALUE) compareTo (b xor Int.MIN_VALUE)` (`:30`), i.e. unsigned order; returns `-(low+1)` when missing. Its test `UnsignedBinarySearchKtTest.kt:10-40` covers arrays that mix positive and "negative" (>= 0x80000000) values.

Caller chain (`Yes2Reader.java:272-308`): `getXrefEntry(arif)` lazily builds the section once: `prepareLoadSection("xrefs")`; `null` (section absent) => return `null` (and since the field stays null, **every** later call retries the section-index lookup); otherwise `new XrefsSection.Reader().read(sectionInput)`; any exception is logged and returns `null`. `getFootnoteEntry` is identical with `"footnotes"`. Interface: `BibleReader.java:28-37` ("arif: 24bit ari at the MSB + which xref field at the 8bit LSB (starts from 1)"). Result classes are just `public String content` (`XrefEntry.java:3-5`, `FootnoteEntry.java:3-5`).

Non-YES2 storage: `InternalReader.java:279-328` reads the *same* section bodies as stand-alone asset files `internal/<prefix>_xrefs_bt.bt` / `_footnotes_bt.bt` (raw body, no container, no compression); YES1 has no support (`Yes1Reader.java:431-441`).

### 1.3 Writer algorithm (Fact)

`XrefDb.writeXrefEntriesTo(LinkedHashMap<Integer arif, XrefEntry>, BintexWriter)` at `tools/AlkitabConverter/src/main/java/yuku/alkitabconverter/util/XrefDb.java:131-155`; `FootnoteDb.writeFootnoteEntriesTo` at `FootnoteDb.java:102-126` (identical modulo type names; I diffed them with names normalized):

1. `writeUint8(1)`; `writeInt(map.size())`.
2. For each entry **in map iteration order**: `writeInt(arif)`.
3. For each entry in the same order: `writeInt(contentsBw.getPos())` and `contentsBw.writeValueString(content)` into a temporary buffer, so offset *i* = total bytes of entries 0..i-1.
4. `writeRaw(buffer)`.

It is wrapped in `CompressibleXrefsSection` / `CompressibleFootnotesSection` (`Yes2Common.java:286-330`), added only when the map is non-null (`Yes2Common.java:165-171`). An *empty* non-null map still produces a section (`01 00 00 00 00`, observed, section 5).

### 1.4 Differences between the two sections

**Fact.** None in the file format. Differences are only in code around it:

| Aspect | `XrefsSection` | `FootnotesSection` |
|---|---|---|
| Constructor `throws` | `IOException` (`XrefsSection.java:32`) | `Exception` (`FootnotesSection.java:32`) |
| Reader `read` throws | `IOException` | `Exception` |
| Entry class / log tag | `XrefEntry`, "load xref failed" | `FootnoteEntry`, "load footnote failed" |
| `InternalReader` failure handling | rethrows `RuntimeException` (`InternalReader.java:296-300`) | logs and returns null (`:322-325`) |
| Content builders in `*Db` | `addComplete` strips leading `[a-zA-Z+-]<space>` (`XrefDb.java:86-109`); `DefaultXrefProcessor` rewrites reference lists to `@<ta:...@>` tags (`XrefDb.java:157-270`) | none; `appendText` has inverted null check but same result (`FootnoteDb.java:70-82`) |
| App rendering | `XrefDialog` (a verse list below the text) | `AlertDialog` in `IsiActivity` (text only, URLs auto-linked) |

---

## 2. Content format of entries

### 2.1 Parser: `FormattedTextRenderer.render` (Fact)

Both dialogs render the stored string with the same function, `Alkitab/src/main/java/yuku/alkitab/base/widget/FormattedTextRenderer.kt:34-115`:

- Signature default `mustHaveFormattedHeader = false` (`:36`), and both call sites rely on it (`XrefDialog.kt:135`, `IsiActivity.kt:2144-2148` passes `mustHaveFormattedHeader = false` explicitly). So **the content is treated as formatted text without needing a leading `@@`**. If it does start with `@@`, those two chars are absorbed (`:48-50`; observed in my fixture where a footnote starts `@@Cp. ...`).
- Grammar after the optional `@@`: text runs, and `@` followed by one marker character (`:75-111`):

| Marker | Effect |
|---|---|
| `@9` ... `@7` | italic span (`:76-84`); `@9` stores a start index, `@7` closes it, a second `@9` before `@7` overwrites the start |
| `@8` | newline `\n` (`:85-87`) |
| `@0 @1 @2 @3 @4 @5 @6 @^ @>` | consumed and ignored (`:88-89`) |
| `@<TAG@>` | start of tag: finds the next literal `@>` and takes everything between as the tag name; pushes `TagMark(start=sb.length, tag)` (`:90-99`). No closing `@>` => the `@<` is silently dropped and parsing continues at `pos+1` |
| `@/` | pops the most recent tag mark and calls `tagListener.onTag(tag, buffer, start, sb.length)` for the text between `@>` and `@/` (`:100-106`); a stray `@/` with an empty stack is ignored. Tags nest (stack) |
| any other `@x` | emitted literally as `@x` (`:107-110`); a trailing lone `@` is dropped (`:72-74`) |

The text between `@>` and `@/` is **appended to the output** (it is the visible label); the tag only decorates it with a span. Plain text may contain anything else; since `.yet` lines cannot contain newlines (section 4.3), `@8` is the way to break lines.

### 2.2 Tags the app understands (Fact)

- Xref dialog: `tag.startsWith("t")` is "the only supported tag at the moment" (`XrefDialog.kt:135-153`); `encodedTarget = tag.substring(1)`. The first link (`displayedLinkPos == -1`, link 0) is shown bold and its verses are loaded immediately; the others become `ClickableSpan`s. `linkPos` increments for *every* tag, including unsupported ones (`:137`).
- Footnote dialog: `t...` => `ClickableSpan` opening `VersesDialog` for the decoded ranges (`IsiActivity.kt:2149-2166`); any other tag shows an error dialog `"footnote ... contains unsupported tag"` (`:2168-2174`). After rendering, web URLs (`PatternsCompat.WEB_URL`) in the rendered text are made clickable (`:2178-2196`). So a footnote may contain bare URLs; they need no markup (real example in `in-ddd.yet:31815`).
- Search/plain-text stripping of *verse* text (`FormattedVerseText.removeSpecialCodes`, `FormattedVerseText.kt:11-53`) is for verses, not for entry content.

### 2.3 Target grammar: `TargetDecoder.decode` (Fact, `Alkitab/src/main/java/yuku/alkitab/base/util/TargetDecoder.kt:23-56`)

The full tag is `t` + encoded target, so on disk it is e.g. `@<ta:...@>`, `@<tari:...@>`, `@<to:...@>`, `@<tlid:...@>`:

```
tag      := "t" encoded
encoded  := "a:" ranges | "ari:" ranges | "o:" ranges | "lid:" ranges
ranges   := range ("," range)*
range    := single | single "-" single        (split on first "-" only: split(range, 2))
single   := type-specific token
```

- `a:` / `ari:` token: `Ari.parseInt(s, 0)`: decimal int or `0x`-prefixed hex, trimmed of a leading/trailing space, 0 on failure (`Ari.java:56-80`). So `ta:3736580` (decimal) and `ta:0x38e804` are equivalent. A single `x` becomes the range `[x, x]`.
- `o:` token: OSIS `Book.chapter[.verse]`, e.g. `Gen.2.1`, `Gen.2`, `2Co.4.6`; chapter only => verse 0 (`OsisBookNames.kt:140-151`; test `TargetDecoderTest.java:28-35`).
- `lid:` token: legacy KJV "lid" 1..31102 mapped by a table (`LidToAri.kt:28-30`; KJV-only).
- A range with an undecodable end (value 0) is dropped; unknown prefix => empty list and a logged error (`TargetDecoder.kt:29-32`).
- Resolution in a version: verse 0 in a start means verse 1, verse 0 in an end means "to end of chapter" (`ari |= 0xff`), spans across chapters are expanded chapter by chapter (`VersionImpl.java:180-245`). Non-existent verses are skipped.

The generator emits `ta:` targets: `XrefDb.DefaultXrefProcessor` finds reference substrings with `DesktopVerseFinder`, parses them with `DesktopVerseParser`, and wraps each as `@<ta:START-END,...@>original text@/`, choosing hex or decimal by whichever string is shorter (`XrefDb.java:168-269`, `ariToString` `:264-269`). Source texts of the form `"Joh 3:16; Joh 16:29"` therefore exist only *before* conversion; in the file, the semicolons and spaces between tags are plain text between `@/` and the next `@<` (see the real example below).

### 2.4 Real examples (Fact; byte-verified, section 4)

From `tools/in-ddd/in-ddd.yet:31812-31823` (tab-separated `kind book chapter verse field content`):

```
xref  1 1 2 1  Tap @9here@7!!! -> @<to:Gen.2.1@>Dest 1@/ and @<to:Gen.2.11@>Dest 2@/
xref  1 2 2 1  2:2-3: @<ta:70667@>Book. 20:11@/; @<ta:3736580,3736586@>Book. 4:4, 10@/
xref  1 2 24 1 2:24: @<ta:2560773@>Book. 19:5@/; @<ta:2624007-2624008@>Book. 10:7-8@/; @<ta:2950672@>Book. 6:16@/; @<ta:3147039@>Book. 5:31@/
footnote 1 4 1 1  The first footnote on verse 1
footnote 1 4 2 1  Formatting and links test on footnote: Tap @9here@7!!! -> @<to:Gen.2.1@>Going to Gen 2:1@/ and @<ta:3736580,3736586-3736587@>Book. 4:4, 10-11@/
footnote 1 4 2 2  https://www.alkitab.app and http://www.example.com, also www.example.com and www.alkitab.app
```

(Decoding: `ta:70667` = 0x01140B = book id 1, chapter 20, verse 11, matching its label "Book. 20:11" where "Book" is the dummy name of book 2 in this data; `ta:3736580` = book 57, 4:4. So labels do match the targets.)

**Derivation.** Our reader should expose `content` as a raw string and provide a separate pure parser that turns it into `Segment[] = text | italic | newline | link{target: Range[], label}`. Treat unknown tags like the app does in the xref dialog: keep the label text, ignore the link. For the verse list shown in the xref dialog, a link's `ranges` is the `[start,end,start,end,...]` flat list from `TargetDecoder` above.

---

## 3. Verse markers `@<f1@>@/` and `@<x1@>@/`

### 3.1 Mapping to arif (Fact)

Verse renderer, `Alkitab/src/main/java/yuku/alkitab/base/widget/VerseRenderer.kt`:

- A verse is formatted only if it starts with `@@` (`:93-97`); markers are looked at only then.
- Scanner: `@<` sets `inSpecialTag` (`:222`); at the following `@>` the characters since the last `@` are copied into `tag` (`:181-186`, `:223`); text between `@>` and `@/` is appended to the verse as ordinary text; `@/` calls `processSpecialTag(sb, tag, factory, ari)` (`:224`).
- `processSpecialTag` (`:284-309`): requires `tag.length >= 2`; `tag[0] == 'f'`: `field = tag.substring(1).toInt()`, valid only if `1 <= field <= 255`, then **appends the superscript digits of `field`** (U+2070, U+00B9, U+00B2, U+00B3, U+2074..U+2079; two digits for >= 10, `:26-28`, `:322-331`) and attaches a click span for `arif = ari shl 8 or field` covering just those appended characters. `tag[0] == 'x'`: same validation, appends `※` (`XREF_MARK = '※'`, `:31`), arif `ari shl 8 or field`. Invalid field => a Toast, no link. Other first letters are ignored (Compose renderer additionally handles `r` ruby, `VerseRendererCompose.kt:308-315`; same `f`/`x` logic `:316-330`).
- `ari` is the argument of `VerseRenderer.render(..., ari, text, ...)` (`VerseRenderer.kt:77-87`), i.e. **the ari of the verse whose text contains the marker**. `VerseRendererCompose` doc says the same: "encoded verse reference; used as the base for inline-link arif" (`VerseRendererCompose.kt:89`).
- Click handling: `IsiActivity.VerseInlineLinkSpanFactory` (`IsiActivity.kt:2106-2205`) uses `arif` for `XrefDialog.newInstance(arif)` and `version.getFootnoteEntry(arif)`; the footnote dialog prints the superscript of `arif and 0xff` followed by the content (`IsiActivity.kt:2138-2141`); the xref dialog uses `arif ushr 8` to recover the source ari (`XrefDialog.kt:107`, `IsiActivity.kt:2118`).

**What is displayed.** For `@<f1@>@/` the reader sees `¹`; for `@<x1@>@/` it sees `※`. If the marker has non-empty label text (`@<f1@>label@/`), the label is shown first and the mark appended after it (the click span covers only the mark) (**Derivation** from the scanner order; upstream data always uses an empty label, e.g. `in-ddd.yet:72`, `:151`).

### 3.2 Several markers, field numbering (Fact + Derivation)

- A verse may have many markers; the field numbers are *chosen by the producer* and nothing in the reader requires them to be contiguous or ordered. The yet converter's checker only requires each marker's `(ari, field)` to have an entry and `1 <= field <= 255` (`YetFileInput.java:246-296`). Real data: verse 1:1:2 has `@<x1@>@/ ... @<x2@>@/` (`in-ddd.yet:72`), verse 1:4:1 has `@<f1@>@/ ... @<f2@>@/` (`:151`).
- Footnote and xref fields live in **separate namespaces** (two sections, `YetFileInput.java:104-105` have separate matchers, `footnoteArifs`/`xrefArifs`). `x1` and `f1` of the same verse have the same arif number but are looked up in different sections (observed in my fixture, verse 1:1:1 has both).
- The same field may appear twice in a verse text; both markers resolve to the same entry.
- `XrefDb.toEntries()` renumbers entries per ari as 1..n in sorted order (`XrefDb.java:39-48`); that is a converter detail, not a file rule.
- The marker's verse is the verse whose text holds it. Entries are keyed by that verse's ari; there is no notion of "a marker in verse A whose entry lives under chapter B". A mis-keyed marker simply yields `null` from the lookup. `YetFileInput` verification catches this at build time (`:274-288`).

### 3.3 How to parse the markers in our reader (Derivation)

For a formatted verse (starts with `@@`), scan for `@<` ... `@>` ... `@/` in text order. When the tag is `f<digits>` or `x<digits>` with value 1..255, compute `arif = ((ari << 8) | field) >>> 0` (as an unsigned 32-bit; for books >= 128 `ari << 8` overflows into the sign bit). Then call `getFootnote(arif)` / `getXref(arif)`.

---

## 4. Real sample data in the repo, and a real upstream-generated file

### 4.1 Where real data exists (Fact, paths relative to the androidbible repo root)

Search over all 1,699 tracked files (`git ls-files`), no `.yes` file exists, no `.yet` besides one:

| Path | What |
|---|---|
| `tools/in-ddd/in-ddd.yet` (713,107 bytes) | The only `.yet` file. 31,317 `verse`, 223 `pericope`, 201 `parallel`, 66 `book_name`, 4 `info`, **4 `footnote` (lines 31812-31815)** and **8 `xref` (lines 31816-31823)**. Verse text with markers at lines 72 (`@<x1@>`, `@<x2@>`), 151-152 (`@<f1@>`, `@<f2@>`). |
| `Alkitab/src/plain/assets/internal/ddd_xrefs_bt.bt` (602 bytes) | Raw `xrefs` section body for DDD (version 1, 8 entries). |
| `Alkitab/src/plain/assets/internal/ddd_footnotes_bt.bt` (339 bytes) | Raw `footnotes` section body (version 1, 4 entries). |
| `Alkitab/src/plain/assets/internal/ddd_k*.txt`, `ddd_index_bt.bt`, `ddd_pericope_*_bt.bt` | The rest of the DDD "internal" module (text/pericopes, Bintex). |
| `tools/AlkitabConverter/src/main/java/yuku/alkitabconverter/yet/YetFileInput.java` | `.yet` parser (`xref`/`footnote` lines `:210-233`, verification `:246-296`). |
| `tools/AlkitabConverter/src/main/java/yuku/alkitabconverter/util/{XrefDb,FootnoteDb}.java` | Section writers, sorting wrappers. |
| `tools/AlkitabConverter/src/main/java/yuku/alkitabconverter/yes_common/Yes2Common.java` | `createYesFile(...)` (`:145-174`) + the two Compressible sections (`:286-330`). |
| `tools/YetToYes2/src/yuku/alkitabconverter/yet/YetToYes2.java` | CLI entry: `.yet` -> `.yes` (`:46-122`). Uses JCommander only for flag parsing. |
| `tools/AlkitabConverterProcesses/src/yuku/alkitabconverter/in_tb_usfm/Proses3.java:32-34`, `in_ddd/Proses3.java`, `en_kjv_yet/ProcessToInternal.java` | Other users of `XrefDb`/`FootnoteDb` (they call `createInternalFiles`, the `.bt` writer). |
| `tools/prog/wordsend/toomanyxref.xml`, `publication/prog/en-leb-processor/extractfootnote.py` | Unrelated to the `xrefs`/`footnotes` sections (an XML sample and a Python helper); not used here. |
| `AlkitabYes2/src/test/java/yuku/alkitab/yes2/util/UnsignedBinarySearchKtTest.kt` | Tests the unsigned search incl. mixed-sign arrays. |
| `AlkitabYes2/src/test/java/yuku/alkitab/yes2/Yes2RoundTripTest.kt:498-509` | Only asserts `getXrefEntry`/`getFootnoteEntry` return `null` when the sections are absent. |
| `Alkitab/src/test/java/yuku/alkitab/base/util/TargetDecoderTest.java`, `.../widget/FormattedTextRendererTest.java` | Unit tests for target decoding and the content renderer. |

**Fact (searched, absent):** no unit tests exist for `XrefsSection`, `FootnotesSection`, `YetFileInput`, `XrefDb` or `FootnoteDb` (`git ls-files | grep -i test | grep -i 'yet\|xref\|footnote\|yes2'` finds only the three files in the table), and no androidTest assets contain xref/footnote data.

### 4.2 Verified decoding of the checked-in `.bt` files (Observed)

I opened `ddd_xrefs_bt.bt` / `ddd_footnotes_bt.bt` with upstream's own `XrefsSection.Reader` / `FootnotesSection.Reader` (compiled with plain javac; harness `Bt.java`). Output saved as `fixtures/upstream-ddd_bt.decoded.txt`. Header bytes of the xrefs file:

```
01                      version
00 00 00 08             count = 8
00 01 02 01  00 01 02 02  00 02 02 01  00 02 07 01  00 02 09 01  00 02 18 01  00 03 01 01  00 03 01 02     arifs (book 0 chapter 1 verse 2 field 1, ...)
00 00 00 00  00 00 00 47  00 00 00 72  00 00 00 bb  00 00 00 dd  00 00 01 17  00 00 01 97  00 00 01 e7     offsets
70 45 54 61 70 20 40 39 ...                                                                                  first content: 8-bit string, len 0x45
```

and the entry decoded for arif `0x00010201` is `Tap @9here@7!!! -> @<to:Gen.2.1@>Dest 1@/ and @<to:Gen.2.11@>Dest 2@/`.

**Strong check (Observed):** I ran upstream `YetFileInput` + `Yes2Common.createYesFile(..., compressed=false, ...)` on the *whole* `tools/in-ddd/in-ddd.yet` (31,317 verses), extracted the `xrefs` and `footnotes` section bodies from the resulting `.yes`, and compared them to the two checked-in `.bt` files: both are **identical byte for byte** (602/602 and 339/339 bytes; each section's attribute map is the single byte `0x90` when uncompressed). This proves our reading of the layout against bytes the Android app itself ships and reads.

### 4.3 `.yet` line syntax (Fact: `YetFileInput.java`)

UTF-8 text, read with `Scanner.nextLine()` (`:126-128`); each line `split("\t", -1)` (`:133`), first field is the command:

| Command | Fields | Source |
|---|---|---|
| `info` | `info <TAB> key <TAB> value` (`locale`, `shortName`, `longName` (required by `YetToYes2:75`), `description`) | `:136-139` |
| `book_name` | `book_name <TAB> book_1 <TAB> name [<TAB> abbreviation]` | `:158-169` |
| `verse` | `verse <TAB> book_1 <TAB> chapter_1 <TAB> verse_1 <TAB> text` (ordering enforced `:176-196`: next verse, or verse 1 of next chapter, or 1:1 of a new book) | `:170-209` |
| `pericope` | `pericope <TAB> book_1 <TAB> chapter_1 <TAB> verse_1 <TAB> title` | `:140-151` |
| `parallel` | `parallel <TAB> text` (attaches to the last `pericope`) | `:152-157` |
| `xref` | `xref <TAB> book_1 <TAB> chapter_1 <TAB> verse_1 <TAB> field_1 <TAB> content` | `:210-220` |
| `footnote` | `footnote <TAB> book_1 <TAB> chapter_1 <TAB> verse_1 <TAB> field_1 <TAB> content` | `:222-232` |
| `#...` / blank | ignored | `:234-235` |

- `arif = (Ari.encode(book_1 - 1, chapter_1, verse_1) << 8) | field_1` (`:220`, `:232`). `book_1` is **1-based** in `.yet`, `ari` is 0-based.
- `Ari.encode` masks each part to 8 bits (`Ari.java:16-18`), so chapter/verse/book > 255 silently wrap; `field_1` is **not** masked or range-checked on the xref/footnote lines themselves (a value > 255 would corrupt the ari bits). Only the markers in verse text are range-checked (`:258-268`).
- Content is `splits[5]` only: a TAB inside content truncates it; newlines cannot occur (line-based), so use `@8`. A missing 6th field throws `ArrayIndexOutOfBounds`; an empty one gives `""` (written as `0d`). Content is stored verbatim; no escaping.
- Duplicate `(ari, field)`: stored in a `LinkedHashMap`, so the later line replaces the value but keeps the first line's position (`:62-74`; observed in `edge-unsorted-null.yes`).
- **Line order is not sorted** (`YetFileInput` keeps insertion order; `YetToYes2.java:119` passes the maps straight to `createYesFile`). `in-ddd.yet` happens to be in ascending order.
- Validation (`:246-296`): every `@<f(\d+)@>` / `@<x(\d+)@>` in verse text must have a matching `footnote` / `xref` line, else the run fails with "there are footnotes and/or xrefs not resolved". The reverse (entry without marker) is allowed.

### 4.4 Producing a real `.yes` with the upstream Java writer: DONE (Observed)

**It works with plain `javac`; no Kotlin or Android SDK is needed.** The existing harness already provides stubs for `android.util.Log`, `android.os.Parcel(able)`, `androidx.annotation.*` and Java transliterations of the three Kotlin files used by the *reader* (`tools/upstream-check/README.md`). For the *writer* path nothing else is required: `Yes2Writer`, `Yes2Common`, `YetFileInput`, `XrefDb`, `FootnoteDb`, `TextDb`, `KjvUtils`, `DesktopVerse*` are plain Java and compile through `-sourcepath tools/AlkitabConverter/src/main/java`. The only upstream class replaced is `YetToYes2`'s `main`: it needs `com.beust.jcommander` for flags only, so my 20-line `Gen.java` repeats its `main()` logic (`YetToYes2.java:63-119`) without the CLI (**Derivation**; everything else is upstream code).

Commands (reproducible; `fixtures/upstream-gen/build.sh <androidbible clone> <outdir>`):

```
javac -d OUT -sourcepath tools/upstream-check/stubs:<repo>/tools/AlkitabConverter/src/main/java \
      <stubs> Gen.java @<AlkitabYes2, AlkitabIo, AlkitabModel, BintexReader, BintexWriter, Snappy java files>
java -cp OUT Gen upstream-xrefs.yet upstream-xrefs.yes [--no-compress]
tools/upstream-check/run.sh src/services/yes2/__tests__/fixtures/upstream-xrefs.yes
```

Regenerating gives the same file except the 2 low bytes of `versionInfo.buildTime` (checked with `cmp -l`: 2 differing bytes).

Files produced, all under `src/services/yes2/__tests__/fixtures/`:

| File | What |
|---|---|
| `upstream-xrefs.yet` | Hand-written input (7 verses in 3 books, 2 pericopes with parallels, 8 xrefs, 5 footnotes) |
| `upstream-xrefs.yes` | **Default output of upstream's writer** (all four big sections Snappy-compressed). 2,084 bytes |
| `upstream-xrefs-uncompressed.yes` | Same with `--no-compress` (1,837 bytes); easier to inspect, sections `xrefs` at logical content size 229, `footnotes` 326 |
| `upstream-xrefs.dump.txt` | `tools/upstream-check/run.sh` output (upstream `Yes2Reader`); identical for compressed and uncompressed |
| `upstream-ddd_xrefs_bt.bt`, `upstream-ddd_footnotes_bt.bt`, `upstream-ddd_bt.decoded.txt` | Copies of the repo's raw section bodies and what upstream's section readers return |
| `upstream-edge-*.yes`, `upstream-edge-lookups.txt` | Edge-case files and upstream lookups (section 5) |
| `upstream-gen/` | `Gen.java`, `Edge.java`, `Lookup.java`, `Bt.java`, `build.sh` (harness code, **Derivation**, not upstream) |

Upstream-read dump (`upstream-xrefs.dump.txt`), selected lines. `F`/`X` lines show `arif` in decimal and the content returned by `getFootnoteEntry`/`getXrefEntry` for every marker found in verse text; no `<null>`:

```
V 0 1 1 @@In the beginning God created the heaven and the earth.@<f1@>@/@<x1@>@/
F 65793 Or, in a beginning; Hebrew: bérēʼsîṯ
X 65793 @<ta:0x2@>Gen. 2:1@/; @<to:Joh.1.1-Joh.1.3@>Joh 1:1-3@/
F 66049 @@Cp. @9Isa 45:18@7; @<ta:0x10101@>Gen 1:1@/ — 中文 😀
X 1090584833 @<ta:0x20301@>Gen 1:1@/          (book 65, ari 0x410101, field 1 => arif 0x41010101)
F 1090584833 Unknown tag test @<zz@>link@/ end
P 257 The Creation [Joh 1:1-3, Heb 11:3]
```

Section bytes of `upstream-xrefs-uncompressed.yes` (xrefs, 229 bytes; content area starts at byte 45):

```
01                                             version
00 00 00 05                                    count
00 01 01 01 | 00 01 02 01 | 00 01 03 01 | 01 01 01 01 | 41 01 01 01          arifs  (Gen 1:1, 1:2, 1:3, Exo 1:1, Rev 1:1; field 1)
00 00 00 00 | 00 00 00 39 | 00 00 00 53 | 00 00 00 87 | 00 00 00 9f          offsets (relative to byte 45)
70 37 40 3c 74 61 3a 30 78 32 40 3e ...                                      first content: 8-bit string, 0x37 chars
```

and in `footnotes`, entry 0 is a **16-bit string** (`71 24` = 36 UTF-16 units, each char `00 xx`, `00 e9` for "é", `01 13` for "ē"), entry 1 contains the surrogate pair `d8 3d de 00` for U+1F600 and the CJK `4e 2d 65 87`; the last entry is an **8-bit string containing `a0`** (U+00A0, shows the 8-bit form is Latin-1, not UTF-8).

---

## 5. Edge cases (Observed unless marked)

Experiments used upstream's writer `Yes2Common.createYesFile` driven programmatically (`upstream-gen/Edge.java`) and upstream's `Yes2Reader` for lookups (`upstream-gen/Lookup.java`). Results: `upstream-edge-lookups.txt`.

| Case | Result |
|---|---|
| **Books >= 128 (arif sign bit)** | `arif = ari << 8 | field` is a negative Java int when book >= 128 (e.g. book 128 => `0x80010101` = -2147417855; book 200 => `0xc8010101`). The reader compares unsigned, so the sorted order must be `0x00..`, `0x7f..`, `0x80..`, `0xc8..`. `Ari.encode` masks the book with 0xff, so max book id is 255 (`Ari.java:16-18`). In TypeScript, **do not use `<<`/`|` results directly**: use `(((ari << 8) | field) >>> 0)` and compare as unsigned. |
| **Writer sort order trap** | `new XrefDb(map).toEntries()` iterates a `TreeMap<Integer,...>` (`XrefDb.java:20,26`), which is **signed** order, so books >= 128 come *first*. File written in that order (books 200,128,0,127): upstream lookups found 128 and 200 but returned **no entry for books 0 and 127** (`edge-highbooks-signed.yes`). Written in unsigned order (0,127,128,200) all four are found (`edge-highbooks-unsigned.yes`). Also `XrefDb.java:28` uses `arif >> 8` (arithmetic shift), harmless for round trip but sign-extends the ari key. For canonical Bibles (books 0..65) there is no difference. |
| **Unsorted input** | A map inserted as arifs 3,1,2 is written in that order; upstream then finds 1 and 2 but **not 3** (`edge-unsorted-null.yes`, `0x00010103` => no entry). The failure is silent (`null`). The `.yet` path does not sort (section 4.3). Our reader should not assume sorting works blindly: **Derivation:** after reading the arif array, verify strict unsigned ascending order and fall back to building a `Map` (linear scan) if not, so files produced by upstream tools with unsorted `.yet` input still work. |
| **Null content** | `XrefEntry.content == null` => `writeValueString(null)` = byte `0c` (`BintexWriter.java:306-308`); reader returns an entry whose `content` is null, **not** a null entry (`F=[null]` in `upstream-edge-lookups.txt`). The `.yet` path cannot produce it (a field is at least `""`). The app would hit a Kotlin non-null parameter in `FormattedTextRenderer.render(text: String ...)` (`FormattedTextRenderer.kt:35`) for such a file, so treat null as "no entry" (**Derivation**). |
| **Empty content** | `""` => `0d` (`BintexWriter.java:309-311`); reader returns `""` (`F=[]`). |
| **Empty table** | Non-null empty map => section body `01 00 00 00 00`; reader builds zero-length arrays, binary search returns -1, lookup `null`. If the section is absent entirely, `prepareLoadSection` returns null and the lookup is `null` (`Yes2Reader.java:274-278`, `:315-341`). With `count == 0` no content area exists; `content_start_offset` = 5. |
| **Duplicate arifs** | Cannot be written through `LinkedHashMap`-based writers (same key overwrites, observed: `first` replaced by `first-overwritten` keeping position). If a foreign file had duplicates, the unsigned binary search returns whichever it hits first (**Derivation**). |
| **Field limits** | Marker field must be 1..255 for the renderer (`VerseRenderer.kt:290-291`, `:300`) and the yet verifier (`YetFileInput.java:258-268`); field 0 is invalid. A `.yet` `xref`/`footnote` line with field > 255 is not checked (section 4.3) and would corrupt the ari. Superscript for fields >= 10 uses one char per decimal digit (`VerseRenderer.kt:322-331`). |
| **UTF-16 content** | Supported natively: `71 len8` / `73 len32` (16-bit strings); BMP + surrogate pairs round-trip (`upstream-xrefs.yes`: "bérēʼsîṯ", CJK, U+1F600 read back identically by upstream). Length byte counts UTF-16 units, so a string >= 256 units uses `72`/`73` with a 4-byte length. |
| **Large contents** | With compression, strings can straddle 32768-byte block boundaries; upstream's `_read8BitString` does a single `is_.read(buf, 0, len)` (`BintexReader.java:358`), which relies on `SnappyInputStream.read` continuing across blocks (`SnappyInputStream.java:83-115`). Not exercised by my small files. |
| **`count` / offsets validation** | None: a negative count would throw `NegativeArraySizeException`, offsets are not bounds-checked, only `version != 1` is rejected. |

---

## 6. Recommended implementation for the TypeScript reader (Derivation)

1. `readXrefs(section: Uint8Array)` / `readFootnotes(section)` (same function): check `version === 1`; read `count`; read `arifs` as `Uint32` (use `getUint32`, not `getInt32`); read `offsets`; `contentStart = 5 + 8*count`. Keep lazy: decode a string only when looked up.
2. Lookup by unsigned binary search on the `Uint32Array`; if the array is found to be non-ascending (check once at load), build a `Map<number, number>` instead.
3. Decode a value string per section 1.1: `0c` -> null, `0d` -> `""`, `5x`/`6x` short, `70/71` 1-byte len, `72/73` 4-byte len; 8-bit = Latin-1, 16-bit = UTF-16BE. Reuse the project's existing Bintex string reader (it already reads the same value type in `versionInfo` etc.).
4. Offsets are logical offsets in the decompressed section content, so read from the already-decompressed `Uint8Array` of the section.
5. Verse markers: scan formatted verses for `@<f<N>@>` / `@<x<N>@>` (N 1..255) -> `arif = (((ari << 8) | N) >>> 0)`; display `¹..` superscript digits for `f`, `※` for `x`.
6. Entry content parser as in section 2 (`@9 @7 @8 @<tag@> @/`, `@@` prefix optional, targets `a:`/`ari:`/`o:`/`lid:`). OSIS names need a book table (`OsisBookNames.kt`, not read in full here).
7. Tests: use `fixtures/upstream-xrefs.yes` (compressed), `upstream-xrefs-uncompressed.yes`, `upstream-ddd_*_bt.bt` (raw bodies; expect the decoded entries in `upstream-ddd_bt.decoded.txt`), and the `upstream-edge-*.yes` files.

---

## 7. Corrections / additions to `yes-file-format.md` (Derivation from the above)

- 4.5 said xref content examples like `"+ Joh 3:16; Joh 16:29"` are stored raw. They are only the *input* of `XrefDb.addComplete`; stored content is formatted text with `@<ta:...@>label@/` tags (section 2). Also `@<to:...@>` OSIS targets are used in the repo's own data.
- 4.5 said "writers do NOT sort" and that the reader needs sorted order: confirmed by experiment, and the specific `XrefDb` signed-`TreeMap` trap for books >= 128 is new (section 5).
- 6 said text between `@>` and `@/` is the displayed text: true; for `f`/`x` the mark (superscript field number / `※`) is appended *after* any such label.

## 8. Open questions

1. Whether upstream validates that a `ta:` target matches the verse named in its label: no such check was found in the code read (the DDD data happens to be consistent).
2. Whether any real-world `.yes` files in the wild (e.g. Alkitab's published bibles) contain unsorted xref tables; this would silently break upstream's own app, so likely not, but I cannot check without such files.
3. `OsisBookNames.kt` book-name table (for the `o:` targets) was not read in full; needed if we want to resolve `to:` targets (only referenced at `TargetDecoder.kt:25`, `OsisBookNames.kt:114-151`).
4. Behaviour with strings crossing a Snappy block boundary inside these sections was not tested with a real compressed file (our fixtures are < 32 KB per section).
5. The `.yet` field_1 range check gap (> 255) is only an observation from reading; not run.
