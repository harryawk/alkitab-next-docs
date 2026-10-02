# YES file format (YES1 / YES2) as implemented in yukuku/androidbible

Research date: 2026-10-02. Source read: `https://github.com/yukuku/androidbible` at commit `f95ffb89371ddd3b2737f59315b1d61a3df37098` (master, 2026-09-27, "Fix Compose verse number weight and BSB paragraph indentation (#304)"), shallow-cloned and read directly. Goal: let someone write a correct parser and generator in TypeScript for this Expo app.

Convention note: matches `docs/research/` (see `cross-platform-file-reader-stacks.md`).

Legend: **Fact** = read from source, cited as `path:Lnn`. All paths are relative to the repo root; permalink form is `https://github.com/yukuku/androidbible/blob/f95ffb89371ddd3b2737f59315b1d61a3df37098/<path>#L<n>`. **Derivation** = my inference or my TypeScript, not repo code. The repo's own `docs/binary-formats.md` was used only as a pointer; where it disagrees with the code, the code wins (disagreements are listed in section 9).

Verification status: the TypeScript in section 8 was type-checked with the project's `tsc` and round-trip tested (uncompressed, and compressed with a literal-only Snappy stand-in; BMP + CJK text; 20 KB verses crossing a 32768-byte block boundary; pericopes; xrefs; footnotes; Bintex int edge values). It was NOT validated against bytes produced by the Java code, because the repo ships no `.yes` sample and no JVM build was run. Treat byte-exactness as "faithful to the source reading", not "proven against real files". The project's own `assets/bible/kjv.yes` is the right next check.

---

## 0. Where the code lives

| Concern | Files |
|---|---|
| Version sniffing | `Alkitab/src/main/java/yuku/alkitab/base/storage/YesReaderFactory.kt` |
| YES2 reader/writer | `AlkitabYes2/src/main/java/yuku/alkitab/yes2/Yes2Reader.java`, `Yes2Writer.java` |
| YES2 sections | `.../yes2/section/{VersionInfoSection,BooksInfoSection,PericopesSection,XrefsSection,FootnotesSection,TextSection}.java`, `section/base/SectionContent.java` |
| YES2 models | `.../yes2/model/{SectionIndex,Yes2Book,Yes2PericopeBlock,PericopeData,VerseBytes}.java` |
| Verse decoding | `.../yes2/io/Yes2VerseTextDecoder.java`, `AlkitabIo/src/main/java/yuku/alkitab/io/Utf8Decoder.java` |
| Compression | `.../yes2/compress/{SnappyInputStream,SnappyOutputStream}.java`, `Snappy/src/main/java/yuku/snappy/codec/*` |
| Bintex primitives | `BintexReader/src/main/java/yuku/bintex/{BintexReader,ValueMap}.java`, `BintexWriter/src/main/java/yuku/bintex/BintexWriter.java` |
| Ari / index helpers | `AlkitabModel/src/main/java/yuku/alkitab/util/Ari.java`, `.../model/PericopeIndex.java`, `.../model/Book.java`, `AlkitabYes2/.../util/unsignedBinarySearch.kt` |
| YES1 reader (legacy, read-only) | `Alkitab/src/main/java/yuku/alkitab/yes1/*`, `Alkitab/src/main/java/yuku/alkitab/base/storage/OldVerseTextDecoder.kt` |
| Generators (YES2 only) | `tools/AlkitabConverter/src/main/java/yuku/alkitabconverter/yes_common/Yes2Common.java`, `tools/YetToYes2/src/yuku/alkitabconverter/yet/YetToYes2.java`, `.../yet/YetFileInput.java`, `.../util/{TextDb,XrefDb,FootnoteDb}.java` |
| Text markup renderer | `Alkitab/src/main/java/yuku/alkitab/base/widget/VerseRenderer.kt`, `.../base/util/FormattedVerseText.kt` |
| Tests | `AlkitabYes2/src/test/java/yuku/alkitab/yes2/Yes2RoundTripTest.kt` (uncompressed + Snappy round trip), `.../SnappyStreamRoundTripTest.kt`, `Alkitab/src/test/.../YesReaderFactoryTest.kt` |

There is no YES1 writer anywhere in the repo (grep for the YES1 section names `infoEdisi___` / `teks________` hits only `Yes1Reader.java`). There is no PDB/OSIS/Bibledit-to-YES converter in the YES2 path other than `.yet` -> YES2 (`YetToYes2`); other converters under `tools/AlkitabConverterProcesses/` produce `.yet` or call `Yes2Common.createYesFile`. The `.yet` input is a tab-separated text format described in section 7.

---

## 1. Version detection and header

**Fact.** `YesReaderFactory` reads 8 bytes and compares the first 7 to `98 58 0d 0a 00 5d e0`; byte index 7 selects the version: `0x01` -> `Yes1Reader`, `0x02` -> `Yes2Reader`, else unsupported (`YesReaderFactory.kt:14`, `:28`, `:33-39`).

- YES2 header (8 bytes): `98 58 0d 0a 00 5d e0 02`. Checked in `Yes2Reader.java:110-116`; written from `Yes2Writer.java:44`.
- YES1 header (8 bytes): `98 58 0d 0a 00 5d e0 01` (`Yes1Reader.java:83`).
- No checksum, CRC, or magic footer validation exists in either reader. The YES2 writer appends one footer byte `0x00` (`Yes2Writer.java:189`); the reader never reads it.
- Endianness: all fixed-width integers are **big-endian**, signed 32-bit for `int` (`BintexWriter.java:149-156`, `BintexReader.java:136-140`).

---

## 2. Bintex primitives (byte-exact)

Everything after the 8-byte header is encoded with "Bintex". Terms below: `u8`/`u16` unsigned big-endian; `int` signed 32-bit big-endian; "Latin-1" = each byte is one UTF-16 code unit 0x00-0xFF.

### 2.1 Raw (untyped) primitives

| Name | Encoding | Source |
|---|---|---|
| `int` | 4 bytes BE | `BintexWriter.java:149`, `BintexReader.java:136` |
| `uint8` | 1 byte; writer throws outside 0..255 | `BintexWriter.java:165` |
| `uint16` | 2 bytes BE | `BintexWriter.java:175`, `BintexReader.java:158` |
| `char` | 2 bytes BE (UTF-16 code unit) | `BintexWriter.java:158` |
| `varuint` | variable-length non-negative int, see below | `BintexWriter.java:209-240`, `BintexReader.java:197-228` |
| short string | `u8 len` + `len` x 2-byte chars (always 16-bit; used by YES1 only) | `BintexWriter.java:77`, `BintexReader.java:53` |
| long string | `int len` + `len` x 2-byte chars (YES1) | `BintexWriter.java:94`, `BintexReader.java:73` |
| auto string | kind byte (`01`=8-bit short, `02`=16-bit short, `11`=8-bit long, `12`=16-bit long), then `u8`/`int` length, then chars (YES1 pericopes v3) | `BintexReader.java:102-134` |

**varuint** (used for per-verse byte lengths in the YES2 text section):

| Value range | Bytes | Layout |
|---|---|---|
| 0..0x7F | 1 | `0xxxxxxx` |
| 0x80..0x3FFF | 2 | `10xxxxxx` + 1 byte |
| 0x4000..0x1FFFFF | 3 | `110xxxxx` + 2 bytes |
| 0x200000..0x0FFFFFFF | 4 | `1110xxxx` + 3 bytes |
| larger (to 0x7FFFFFFF) | 5 | `0xF0` + 4 bytes BE |

(Source: writer `BintexWriter.java:209-240`; reader `BintexReader.java:197-228`.) The reader throws for first bytes `0xF1..0xFF`. Note the writer's own comment says "16383-2097151 is 3 bytes" with an off-by-one in the lower bound (`:205`); the code boundary is `a <= 0x3fff` for 2 bytes (`:217`).

### 2.2 Typed "value" encodings (used in maps and in xref/footnote/pericope strings)

Documented in the header comment of `BintexWriter.java:8-55`; reader dispatch in `BintexReader.java:9-27` (type table), `:230-278` (ints), `:280-376` (strings), `:378-472` (arrays), `:474-526` (maps/any).

**Int values** (writer `writeValueInt`, `BintexWriter.java:242-302`):

| Value | Bytes |
|---|---|
| 0 | `0e` |
| 1..7 | `01`..`07` (the value itself) |
| -1 | `0f` |
| positive < 256 | `10 A` |
| positive < 65536 | `20 A[2]` |
| positive < 2^24 | `30 A[3]` |
| positive otherwise | `40 A[4]` |
| negative n | let m = `~n` (bitwise NOT, i.e. `-n-1`); same ladder with type bytes `11`, `21`, `31`, `41` and payload m |

Reader returns `~a` for the `x1` types (`BintexReader.java:250-273`). The reader accepts non-minimal encodings too (it dispatches purely on the type byte).

**String values** (`BintexWriter.java:304-368`, `BintexReader.java:280-376`):

| Case | Bytes |
|---|---|
| `null` | `0c` |
| empty string | `0d` |
| len 1..15, all chars <= 0xFF | `0x50|len` then len bytes (Latin-1) |
| len 1..15, any char > 0xFF | `0x60|len` then len x 2 bytes BE |
| len 16..255 | `70 len` + bytes (8-bit) or `71 len` + 2-byte chars |
| len >= 256 | `72 int(len)` + bytes, or `73 int(len)` + 2-byte chars |

`len` is the Java `String.length()`, i.e. **UTF-16 code units**, and the 8-bit form is chosen only if every code unit is <= 0xFF. The text is therefore **not UTF-8**; 8-bit strings are Latin-1 and 16-bit strings are UTF-16BE. (In JS, `charCodeAt` matches Java `charAt`, so a straight port is exact, including surrogate pairs being two units.)

**Int arrays** (`BintexWriter.java:370-461`; reader `BintexReader.java:378-472`):

| Type | Layout |
|---|---|
| `c0 A B[A]` | uint8 array, up to 255 elements |
| `c1 A B[A*2]` | uint16 array, up to 255 elements |
| `c4 A B[A*4]` | int32 array, up to 255 elements |
| `c8 A[4] B[A]` | uint8 array, `int` length |
| `c9 A[4] B[A*2]` | uint16 array, `int` length |
| `cc A[4] B[A*4]` | int32 array, `int` length |

`writeValueIntArray` picks the smallest: all values 0..255 -> uint8; all 0..65535 -> uint16; otherwise int32 (negatives force int32). Length < 256 uses the small header, else the `c8/c9/cc` form with a 4-byte length. The reader returns a plain int array regardless of width (`_readValueIntArray`, `:448-472`). **Derivation:** a parser must accept all six array types; a generator may use any of them (the reader does not require minimality), but copying the smallest-width rule gives identical bytes to the Java writer.

**Simple map** (`BintexWriter.java:463-519`, `BintexReader.java:474-507`):

- Empty map: single byte `90`.
- Otherwise: `91`, `u8 entry_count` (max 255), then per entry: `u8 key_len`, `key_len` Latin-1 bytes (key chars must be <= 0xFF), then one **value** (any int / string / int-array / nested simple map). Entry order = insertion order (LinkedHashMap); readers do not depend on order.
- Writer accepts values that are `String`, any `Number` (written via `writeValueInt(intValue())`), `int[]`, or `Map`. A `null` value throws, so producers omit null-valued keys (as `VersionInfoSection`/`Yes2Book` do).
- Reader generic `readValue` dispatch: ints `01-07,0e,0f,10,11,20,21,30,31,40,41`; strings `0c,0d,51-5f,61-6f,70-73`; arrays `c0,c1,c4,c8,c9,cc`; maps `90,91`. Anything else throws (`BintexReader.java:9-27`, `:509-526`).

**Gotcha (Fact):** `ValueMap.getInt(key, def)` returns `0`, **not** `def`, when the key is absent (`ValueMap.java:79-91`, line 82). Consequence: `VersionInfoSection.Reader` documents `textEncoding` default 2 but a file lacking that key actually yields 0 (`VersionInfoSection.java:55`); `Yes2Reader` then logs "not supported" and falls back to ASCII (`Yes2Reader.java:219-227`). All files produced by the writer include `textEncoding`, so this only matters for hand-built files. `getString` stringifies non-strings (`ValueMap.java:93-98`), and `getInt` parses numeric strings.

---

## 3. YES2 container layout

### 3.1 Documented layout (Fact: `Yes2Writer.java:13-39`)

```
uint8[8]  header = 98 58 0d 0a 00 5d e0 02
int       sectionIndex.size
uint8     sectionIndexVersion = 1
int       section_count
section_count x {
  uint8   sectionName.length
  char8[] sectionName            (raw bytes, not a Bintex string)
  int     offset                 (from start of sections area)
  int     attributes_size
  int     content_size
  byte[4] reserved               (written as int 0)
}
section_count x {
  value   section.attributes     (a Bintex simple map, 0x90 if empty)
  byte[]  section.content
}
uint8     footer = 0
```

Absolute byte positions (**Derivation** from the code): header 0..7; `sectionIndex.size` at 8..11; version byte at 12; `section_count` at 13..16; index entries start at 17. `Yes2Reader` does `seek(12)` then `SectionIndex.read` consumes version byte, count, entries (`Yes2Reader.java:118-119`, `SectionIndex.java:24-54`).

### 3.2 Semantics

- `sectionIndex.size` is **the byte length of the entry array only** (excludes the version byte and count): written as `lastPos - savedpos_startOfSectionIndex` where the start was captured after writing version and count (`Yes2Writer.java:109`, `:131-135`). Example: three sections named `versionInfo`, `booksInfo`, `text` give 75 = (1+11+16)+(1+9+16)+(1+4+16). **The reader ignores this field entirely** (it seeks to 12 and parses sequentially), so a parser need not validate it; a generator should still write it identically.
- `offset` is relative to the first byte after the last index entry (`sectionDataStartOffset = input.getFilePointer()` after the entries, `SectionIndex.java:51`). A section's attributes begin at `base + offset`, its content at `base + offset + attributes_size` (`SectionIndex.java:56-83`). `content_size` is stored but **never used by the reader**.
- `reserved` is 4 bytes, skipped (`SectionIndex.java:47`), written as zero (`Yes2Writer.java:126`; the back-patch at `:178-183` only rewrites the first three ints).
- Section name: length byte then raw bytes; names must be <= 255 chars and all chars <= 0xFF (`SectionContent.java:22-37`). Duplicate names: later entry overwrites earlier in the reader's `LinkedHashMap` (`SectionIndex.java:48`).
- Sections are laid out back-to-back in the order of the index, with **no padding or alignment** (`Yes2Writer.java:143-163`). The reader can handle any order and unknown sections (it only looks up by name); it never iterates sections.
- Each section is `attributes` (a simple map value, `0x90` when null/empty, `Yes2Writer.java:153`) then content.
- Unknown `sectionIndexVersion` != 1 -> `SectionIndex.read` returns null (`SectionIndex.java:27-31`), which then NPEs in the reader. A parser should reject with a clear error.

### 3.3 Section names and the order the generator uses

Names (exact, case-sensitive): `versionInfo`, `booksInfo`, `pericopes`, `text`, `xrefs`, `footnotes` (`VersionInfoSection.java:25`, `BooksInfoSection.java:15`, `PericopesSection.java:19`, `TextSection.java:6`, `XrefsSection.java:23`, `FootnotesSection.java:23`).

`Yes2Common.createYesFile` adds sections in this order: `versionInfo`, `booksInfo`, `pericopes` (only if pericope data), `text`, `xrefs` (only if xref entries non-null), `footnotes` (only if footnotes non-null) (`Yes2Common.java:157-173`). The repo's `Yes2RoundTripTest` uses the same order (without xrefs/footnotes) (`Yes2RoundTripTest.kt:70-76`).

Compression: only `pericopes`, `text`, `xrefs`, `footnotes` are ever compressed by the generator; `versionInfo` and `booksInfo` always have empty attributes (`0x90`) and are raw (`Yes2Common.java:157-173`, `SectionContent.java:47-49` returns null attributes -> `Yes2Writer.java:153`).

### 3.4 Section attributes and compression ("compression" is the only attribute the reader uses)

Attributes for a compressed section (`Yes2Common.java:196-209`):

```
91 03
  "compression.name"    = "snappy-blocks"          (string)
  "compression.version" = 1                        (int)
  "compression.info"    = simple map {
      "block_size"             = 32768             (int)
      "compressed_block_sizes" = int[]             (array; per-block compressed byte sizes)
  }
```

Reader: reads attributes with `readValueSimpleMap`; if `compression.name` exists and equals `snappy-blocks`, builds a `SnappyInputStream`; any other name -> exception; `compression.version` > 1 -> exception (default 0 when absent); no encryption support exists (**Fact:** the word appears only in a class comment, `Yes2Reader.java:44`; nothing reads or writes an encryption attribute) (`Yes2Reader.java:60-69`, `:315-341`, `SnappyInputStream.java:130-148`).

Compression algorithm (**Fact**): the section's *logical content* (exactly what would have been written uncompressed) is cut into blocks of `block_size` = 32768 uncompressed bytes (the last block shorter); each block is independently compressed with raw Snappy (not the framing format) and the compressed blocks are concatenated; `compressed_block_sizes[i]` is the compressed length of block i. The section's `content_size` in the index and the bytes stored are the *compressed* total. (`SnappyOutputStream.java:18-80`, `Yes2Common.java:176-222`.)

Snappy format check (**Fact + Derivation**): `Snappy.java:12-19` (`uncompressedLength`) parses a leading base-128 varint length, and the native binding calls `snappy_compress`/`snappy_uncompress` from the bundled C snappy (`Snappy/jni/yuku_snappy_codec_SnappyImplNative.cpp:20,30`), which is the standard raw Snappy block format (varint preamble + literal/copy elements). In JS you need any raw-Snappy implementation (not the `.sz` framed format). The Java fallback is `de.jarnbjo.jsnappy` (`Snappy/src/main/java/de/jarnbjo/jsnappy/`). I did not verify bit-compatibility of the two against each other.

Decompression/offset arithmetic (**Fact**, `SnappyInputStream.java`): a logical offset `n` into the section content maps to block `n / block_size` and in-block skip `n % block_size` (`:36-44`); compressed block i starts at `sectionContentOffset + sum(compressed_block_sizes[0..i-1])` (`:130-147`, `:52`). Reads that cross a block end continue into the next block (`:83-115`). Hence **all offsets stored inside the sections (book offsets, chapter offsets, pericope offsets, xref offsets) are offsets into the uncompressed logical content**, regardless of compression.

---

## 4. Sections in detail

### 4.1 `versionInfo`

Content is one simple map (`VersionInfoSection.java:28-42`):

| Key | Type | Written when | Meaning |
|---|---|---|---|
| `version` | int | always, value 3 | not checked by reader |
| `shortName` | string | non-null | e.g. "KJV" |
| `longName` | string | non-null | |
| `description` | string | non-null | |
| `locale` | string | non-null | e.g. "en" |
| `buildTime` | int | non-zero | unix seconds; reader ignores it (`:44-58` never reads it) |
| `book_count` | int | always | number of books actually present |
| `hasPericopes` | int | always | 0/1; if 0 the reader skips the `pericopes` section entirely (`Yes2Reader.java:246-248`) |
| `textEncoding` | int | always | 1 = ASCII, 2 = UTF-8 (`:21`); generator always writes 2 (`Yes2Common.java:137`) |

Note `buildTime` is `(int)(System.currentTimeMillis()/1000)` (`Yes2Common.java:136`), a 32-bit signed int; fits `writeValueInt`.

### 4.2 `booksInfo`

Content (`BooksInfoSection.java:23-33`, `:36-46`): `int book_count`, then `book_count` simple maps (`Yes2Book.toBytes`, `Yes2Book.java:16-27`; `fromBytes`, `:29-41`):

| Key | Type | Notes |
|---|---|---|
| `version` | int | always 3, not checked |
| `bookId` | int | 0-based canonical id; Genesis 0 ... Revelation 65; "66 and above is defined elsewhere" (`Ari.java:11`). Reader default -1 if missing |
| `shortName` | string | display name |
| `offset` | int | byte offset of this book's text, relative to the start of the text section's (uncompressed) content |
| `chapter_count` | int | |
| `verse_counts` | int[] | length `chapter_count`; `verse_counts[c-1]` = verses in chapter c |
| `chapter_offsets` | int[] | length **`chapter_count + 1`**; offsets relative to the book's `offset`; `[0] = 0`; `[c]` = start of chapter c+1; last entry = total bytes of the book |
| `abbreviation` | string | only written when non-null |

Fact for the `+1` length: `Yes2Common.java:99` allocates `chapter_count + 1`. The YES2 reader itself only indexes `chapter_offsets[chapter-1]` and uses `verse_counts` to know how many verses to read (it does not need the last element), but the format carries it (the YES1 reader does need it, `Yes1Reader.java:232`, `:308`).

Books are stored in ascending `bookId` order (generator uses `TextDb.getBookIds()` which is a `TreeSet`, `TextDb.java:226-236`), and the text section holds books in that same order. A parser must key off `bookId`, not array index, because books can be missing (`Yes2RoundTripTest` writes ids 0, 1, 65).

**Chapter/verse model gotchas (Fact):** the generator derives `chapter_count` as the **max chapter number** and per-chapter verse count as the **max verse number** present (`TextDb.java:243-266`), assuming contiguous numbering from 1. Gaps are only filled with empty verses by `TextDb.normalize()` (`TextDb.java:120-180`), and only if `--ignore-skipped-verses` is passed (`YetToYes2.java:114-116`); otherwise `YetFileInput.parse` rejects out-of-order/skipped verses (`YetFileInput.java:176-196`). `getVerseText` would NPE for a missing verse (`TextDb.java:268-270`). An empty verse is stored as `varuint 0` (one byte `00`).

### 4.3 `text`

Content = concatenation, in book order then chapter then verse order, of one record per verse (`VerseBytes.java:8-31`, `Yes2Common.java:224-254`):

```
varuint  byte_length
byte[]   UTF-8 bytes of the verse (for textEncoding = 2)
```

There are no chapter/book separators, no verse numbers, and no terminators; boundaries come entirely from `booksInfo`. Only the *record per verse* exists, so a missing verse in the middle would shift everything (that is why the generator requires contiguity).

Reading a chapter (**Fact**, `Yes2Reader.java:72-93`): position = `book.offset + book.chapter_offsets[chapter-1]` in the logical content; then read `verse_counts[chapter-1]` records (`varuint` length, then that many bytes). Uncompressed: absolute file position = `sectionContentOffset + position`; compressed: seek in the Snappy stream.

Decoding (`Yes2VerseTextDecoder.java`, `Yes2Reader.java:219-227`):
- `textEncoding == 2`: `Utf8Decoder`, which is "intentionally incomplete" and handles only 1-, 2-, and 3-byte sequences (up to U+FFFF) (`Utf8Decoder.java:7-58`). It does no validation. A **4-byte UTF-8 sequence** (emoji, supplementary CJK) is mis-decoded: first byte `>= 0xE0` is treated as a 3-byte lead, consuming only 3 bytes and leaving the 4th as garbage. The writer uses `String.getBytes("utf-8")` (`VerseBytes.java:24`) which would emit proper 4-byte sequences for surrogate pairs, so such text round-trips incorrectly in the Android reader. **Derivation:** a TS parser can use a real UTF-8 decoder (`TextDecoder`) and be strictly more correct; avoid supplementary-plane characters if bit-for-bit app compatibility with the Android reader matters (the repo's own test sticks to the BMP: `Yes2RoundTripTest.kt:114-118`).
- `textEncoding == 1`: bytes < 0x80 only, one char per byte (`Yes2VerseTextDecoder.java:10-48`; the lowercase path has a known index bug `verseBuf[i]` vs `[j]` at `:26`, documented in `docs/tech-debt-remediation/REM-36-yes2-ascii-decoder.md`; display is unaffected). Any other value falls back to ASCII with an error log.
- An `Ascii` verse longer than 4000 bytes overflows the thread-local buffer (`:11`, `:20`): fixed 4000-byte buffer with no resize.

### 4.4 `pericopes` (section headings)

Only present if `hasPericopes == 1`. Content layout (writer `PericopesSection.java:45-133`, reader `:136-192`):

```
uint8  data_format_version = 3          (reader also accepts 2)
int    index_size                        (writer stores the whole section size; reader ignores it)
int    entry_count
entry_count x entry:                     // version 3
    ari part:   u16 d
                if (d & 0x8000) == 0 : absolute: ari = (d << 16) | next u16     (4 bytes total)
                else                 : relative: ari = last_ari + (d & 0x7FFF)   (2 bytes total)
    u16 offset_delta                      // block offset = previous offset + delta (first one starts from 0)
blocks (concatenated, starting right after the last entry):
    uint8 data_format_version = 4
    value title                            // Bintex string value
    uint8 parallel_count
    value[parallel_count] parallels        // Bintex string values
```

Writer rules (**Fact**, `PericopesSection.java:67-86`): the first entry (when `last_entry_ari == 0`) and any entry whose ari delta is negative or > 0x7FFF is written absolutely as `int(ari & 0x00FFFFFF)` — which is two uint16s with the first being the book number (0..255, so its high bit is never set); otherwise `u16(0x8000 | delta)`. Offsets: each entry's block offset (relative to the first block byte) is written as the uint16 difference from the previous block's offset; throw if a delta > 0xFFFF (`:121-129`), i.e. a single block must be < 64 KB.

Version 2 index (reader only, `PericopesSection.java:158-162`): entries are `int ari, int offset` pairs. In YES1 files the pericope index is also `int ari, int offset` pairs but with absolute offsets into a separate block section (`Yes1PericopeIndex.java:11-25`).

`ari` here is the ari of the verse the pericope precedes, with `verse` possibly 1 or any verse (`YetFileInput.java:149`). **Entries must be sorted ascending by ari**; the reader does a binary search (`PericopeIndex.java:15-48`, `Arrays.binarySearch` on the `aris` array) to find the first entry in `[ari(book,chapter,0), +0xFF]`, then walks forward while book and chapter match (`PericopesSection.java:199-228`). The generator does not sort, so ordering depends on the `.yet` input order (`PericopeData.addEntry` just appends).

The `pericopes` section is read through the same compressed-section mechanism, with the logical content starting at position 0 (`Yes2Reader.java:315-341`; the Snappy stream starts at logical offset 0 when no seek is issued). `data_offset_` is the logical offset of the first block (`PericopesSection.java:188`).

### 4.5 `xrefs` and `footnotes`

Identical structure (`XrefsSection.java:12-19,32-55`, `FootnotesSection.java:11-19,32-55`; writers in `XrefDb.writeXrefEntriesTo` `XrefDb.java:131-155` and `FootnoteDb.writeFootnoteEntriesTo` `FootnoteDb.java:102-126`):

```
uint8  data_format_version = 1
int    entry_count
int[entry_count]  arifs       // sorted ascending as UNSIGNED ints
int[entry_count]  offsets     // offset of each content value, relative to the start of the contents area
value<string> contents[entry_count]   // each is a Bintex string value (0c null, 0d empty, 5x/6x/70-73 ...)
```

- `arif` = `(ari << 8) | field_index` where `field_index` is 1..255 (the 1-based sequence number of the marker inside that verse) (`XrefsSection.java:16`, `YetFileInput.java:220,232`, `VerseRenderer.kt:288-304`: `ari shl 8 or field`).
- Lookup is an **unsigned** binary search (`unsignedBinarySearch.kt:8-30`). Because `ari` uses bits 23..0, `ari << 8` can set bit 31 when book >= 128, which is why unsigned comparison matters (books above 127 are non-canonical, but the code supports it).
- **Gotcha (Fact):** the writers emit entries in the iteration order of the supplied `LinkedHashMap` and do NOT sort (`XrefDb.java:139-141`, `FootnoteDb.java:110`; `YetFileInput.addXrefEntry` uses insertion order). The reader requires sorted-by-unsigned order. The `.yet` files must therefore list xref/footnote lines in ascending order; my TypeScript generator sorts defensively (**Derivation**).
- Offsets are computed with the contents `BintexWriter.getPos()` before each value is written (`XrefDb.java:148-151`), so entry i's offset = total bytes of entries 0..i-1; contents area begins immediately after the offsets array. Reader: `content_start_offset` = file pointer after reading both arrays (`XrefsSection.java:53`).
- Content strings are the raw marker content, e.g. `"+ Joh 3:16; Joh 16:29"` for xrefs; `XrefDb.addComplete` strips a leading `[a-zA-Z+-]<space>` marker in some converters (`XrefDb.java:86-109`). Xref targets are parsed from the string at display time by app code, not stored structurally.
- YES1 files cannot contain xrefs or footnotes (`Yes1Reader.java:431-441`).

---

## 5. Ari and arif (Fact: `Ari.java:3-54`)

`ari` is a 32-bit int: bits 31..24 unused (0), bits 23..16 = book id (0..255; 0 = Genesis, 65 = Revelation, 66+ non-canonical), bits 15..8 = chapter (1-based; 0 = undefined/whole book), bits 7..0 = verse (1-based; 0 = whole chapter).

`encode(book, chapter, verse) = (book & 0xff) << 16 | (chapter & 0xff) << 8 | (verse & 0xff)`. Chapters and verses are therefore capped at 255 in anything keyed by ari (pericopes, xref/footnote arifs, and the `1..255` field check `YetFileInput.java:258-268`); `booksInfo` itself stores plain ints so the *text* could hold more verses, but `.yet` ingestion goes through `Ari.encode` and would silently truncate (**Derivation**).

`arif = (ari << 8) | field` (field 1..255).

Locating a verse (**Derivation** summarizing `Yes2Reader.loadVerseText`): find the `booksInfo` entry with `bookId == ari.book`; verse index = `ari.verse - 1` within the array returned for chapter `ari.chapter`; the library never builds a per-verse offset table, so reading verse k means parsing the preceding k-1 length prefixes of that chapter (cheap, since a chapter is a few KB).

---

## 6. Verse text markup (inline codes inside verse strings)

Verse strings are plain text unless they start with `@@`. **Fact** (`VerseRenderer.kt:94-97`): a verse is "formatted" only if its first two characters are `@@`; otherwise it is rendered verbatim (a lone `@` elsewhere is literal text in that case). The generator enforces this: `TextDb.append` prepends `@@` if the text contains `@` and does not already start with `@@` (`TextDb.java:99-101`) — **Derivation:** a producer must do the same, or any literal `@` in an unformatted verse will still be treated as plain text but a formatted verse missing `@@` would show raw codes.

After the `@@` prefix, `@` is an escape character followed by one marker char (`VerseRenderer.kt:67-75` comment, parser `:155-270`, switch at `:198`):

| Code | Meaning |
|---|---|
| `@@` | marks the verse as formatted (must be the first two chars) |
| `@0` .. `@4` | start paragraph with indent level 0..4 |
| `@^` | start paragraph (first-line indent + hanging indent); `TextDb` uses code `-2` for it (`TextDb.java:73-74`) |
| `@6` / `@5` | start / end of red text (words of Jesus) |
| `@9` / `@7` | start / end of italic |
| `@8` | line break / blank line to the next verse |
| `@<` ... `@>` | open/close the "tag" part of a special element; text between is the tag |
| `@/` | end of the special element (all special tags must be closed, since 2013-10-04 per the comment) |

Special elements (**Fact**, `VerseRenderer.kt:284-309`): the tag text between `@<` and `@>` starts with a letter then a number. `f<N>` = footnote with field N, `x<N>` = cross-reference with field N, both `1 <= N <= 255`, written in text as `@<f1@>@/` and `@<x2@>@/`; the lookup key is `arif = (ari << 8) | N`. Unknown tag letters are ignored by the View renderer (the Compose renderer additionally handles `r`: `@<r=reading@>base@/` ruby/furigana, `VerseRendererCompose.kt:308`, data format in `docs/features/ruby/design.md`). Between `@<` and `@>` the text is treated as the tag name (not output); content between `@>` and `@/` is the displayed text.

`YetFileInput` validates each `@<f(\d+)@>` / `@<x(\d+)@>` in verse text has a matching `footnote` / `xref` line (`YetFileInput.java:104-105`, `:246-296`).

Plain-text stripping (**Fact**, `FormattedVerseText.kt:11-53`): if the text starts with `@`, remove every `@` plus the next char; for `@<` also delete up to the matching `@>`; for `@0..@4`, `@^`, `@8` insert one space unless the output already ends in whitespace. Note: it strips tag *names* but keeps the displayed content between `@>` and `@/`.

An empty formatted placeholder is normalized: `"@@@0"` -> `""` by `TextDb.removeEmptyVerses` (`TextDb.java:173-180`).

---

## 7. Generator logic (YES2) end to end

### 7.1 Source: the `.yet` text format (Fact: `YetFileInput.java:111-244`)

UTF-8 text, tab-separated fields, one record per line (`line.split("\t", -1)`):

```
info	<key>	<value>                       # locale, shortName, longName (required), description
book_name	<book_1>	<name>	[<abbrev>]
verse	<book_1>	<chapter_1>	<verse_1>	<text with @ codes>
pericope	<book_1>	<chapter_1>	<verse_1>	<title>
parallel	<text>                             # attaches to the last pericope
xref	<book_1>	<chapter_1>	<verse_1>	<field_1>	<content>
footnote	<book_1>	<chapter_1>	<verse_1>	<field_1>	<content>
# comment / blank lines ignored
```

`book_1` is 1-based; `bookId = book_1 - 1`. Required: `longName` info, at least one verse, book names for every book used (`YetToYes2.java:69-96`). Default ordering check: each verse must be the next verse, or verse 1 of the next chapter, or 1:1 of a book (`YetFileInput.java:176-196`).

### 7.2 Pipeline (`Yes2Common.createYesFile`, `Yes2Common.java:141-174`)

1. Build a `TextDb`: sorted map ari -> verse text (`TextDb.java:20`).
2. `versionInfo` section: `book_count` = number of distinct books, `hasPericopes` = pericope data non-null, `textEncoding` = 2, `buildTime` = now (`Yes2Common.java:128-139`).
3. `booksInfo`: iterate books in id order; for each chapter 1..max, `verse_counts[c-1]` = max verse; accumulate `offsetPassed += len(bytesForAVerse(text))`; `chapter_offsets[0]=0`, `chapter_offsets[c] = offsetPassed`; `book.offset = offsetTotal` (running total), then `offsetTotal += offsetPassed`; shortName from the `book_name` table, abbreviation nullable (`Yes2Common.java:86-126`).
4. Compress flag (`!--no-compress`, default compressed, `YetToYes2.java:118`): for each compressible section, content bytes are written to an in-memory buffer through a `SnappyOutputStream(blockSize 32768)` and `flush()`ed to get `compressed_block_sizes` (`Yes2Common.java:176-222`). The text section is built eagerly in the constructor of `CompressibleLazyText` by emitting each verse record in `TextDb` iteration (ari-sorted) order (`:224-245`).
5. `Yes2Writer.writeToFile` (`Yes2Writer.java:78-193`): write header; reserve section-index placeholders (size int = -1; per section: name, then `-1,-1,-1,0`); back-patch the index size; write each section's attributes then content, recording relative offset, attributes size, content size; seek back and patch the per-section ints; write footer `00`. (The use of seek-back means a streaming generator must precompute sizes — in TypeScript simply build all section buffers first, as my sketch does.)

Padding/alignment: none. Reserved field: always 0.

### 7.3 Verse record order consistency

The text section's byte order is the `TextDb` ari order (book, chapter, verse). `booksInfo` offsets are computed from the same iteration. **The only invariant** the reader relies on is: for every book, text at `book.offset + chapter_offsets[c-1]` begins `verse_counts[c-1]` consecutive verse records. Empty or missing verses must still occupy a record.

---

## 8. TypeScript sketches (Derivation, not repo code)

Tested: compile with `--strict`; round trip of uncompressed and compressed (literal-only Snappy stand-in) files including cross-block verses, pericopes (relative + absolute ari paths), xrefs/footnotes (including a book-65 arif with bit 30 set), and Bintex int boundaries. Not tested against Java-produced files. Production code needs: a real raw-Snappy implementation passed as the `snappy` argument, and `Uint8Array`/`DataView` use (no Node `Buffer`). Practical notes: in React Native read the asset as an `ArrayBuffer`/`Uint8Array`; for random access without loading the whole file, replace `file: Uint8Array` by a reader with `readRange(offset, len)`; keep `loadChapter` as is.

```ts
// Derivation of the androidbible YES2 reader/writer logic in TypeScript.
// Types: ints are JS numbers (treated as signed 32-bit where the repo uses Java int).

export type BintexValue = number | string | null | number[] | { [k: string]: BintexValue };

// ---------------------------------------------------------------- Bintex writer
export class BintexWriter {
  private chunks: number[] = [];
  get pos(): number { return this.chunks.length; }
  bytes(): Uint8Array { return Uint8Array.from(this.chunks); }
  raw(b: ArrayLike<number>): void { for (let i = 0; i < b.length; i++) this.chunks.push(b[i] & 0xff); }
  u8(a: number): void { if (a < 0 || a > 255) throw new Error('uint8'); this.chunks.push(a); }
  u16(a: number): void { if (a < 0 || a > 0xffff) throw new Error('uint16'); this.chunks.push((a >> 8) & 0xff, a & 0xff); }
  int(a: number): void { this.chunks.push((a >>> 24) & 0xff, (a >>> 16) & 0xff, (a >>> 8) & 0xff, a & 0xff); }
  varUint(a: number): void {
    if (a < 0) throw new Error('uint must be non-negative');
    if (a <= 0x7f) this.chunks.push(a);
    else if (a <= 0x3fff) this.chunks.push(((a >> 8) & 0xff) | 0x80, a & 0xff);
    else if (a <= 0x1fffff) this.chunks.push(((a >> 16) & 0xff) | 0xc0, (a >> 8) & 0xff, a & 0xff);
    else if (a <= 0x0fffffff) this.chunks.push(((a >>> 24) & 0xff) | 0xe0, (a >> 16) & 0xff, (a >> 8) & 0xff, a & 0xff);
    else this.chunks.push(0xf0, (a >>> 24) & 0xff, (a >> 16) & 0xff, (a >> 8) & 0xff, a & 0xff);
  }
  valueInt(a: number): void {
    if (a === 0) this.chunks.push(0x0e);
    else if (a >= 1 && a <= 7) this.chunks.push(a);
    else if (a === -1) this.chunks.push(0x0f);
    else {
      const neg = a < 0;
      const m = neg ? ~a : a; // Java: a = ~a
      const base = neg ? 0x01 : 0x00;
      if (m < 256) this.chunks.push(0x10 | base, m);
      else if (m < 65536) this.chunks.push(0x20 | base, (m >> 8) & 0xff, m & 0xff);
      else if (m < 16777216) this.chunks.push(0x30 | base, (m >> 16) & 0xff, (m >> 8) & 0xff, m & 0xff);
      else this.chunks.push(0x40 | base, (m >>> 24) & 0xff, (m >> 16) & 0xff, (m >> 8) & 0xff, m & 0xff);
    }
  }
  valueString(s: string | null): void {
    if (s === null) { this.chunks.push(0x0c); return; }
    if (s.length === 0) { this.chunks.push(0x0d); return; }
    let all8 = true;
    for (let i = 0; i < s.length; i++) if (s.charCodeAt(i) > 0xff) { all8 = false; break; }
    const len = s.length; // UTF-16 code units, like Java String.length()
    if (len < 16) this.chunks.push((all8 ? 0x50 : 0x60) | len);
    else if (len < 256) this.chunks.push(all8 ? 0x70 : 0x71, len);
    else { this.chunks.push(all8 ? 0x72 : 0x73); this.int(len); }
    for (let i = 0; i < len; i++) {
      const c = s.charCodeAt(i);
      if (all8) this.chunks.push(c); else this.chunks.push((c >> 8) & 0xff, c & 0xff);
    }
  }
  valueIntArray(a: number[]): void {
    let all8 = true, all16 = true;
    for (const e of a) if (e < 0 || e > 0xffff) { all8 = all16 = false; break; }
    if (all16) for (const e of a) if (e > 255) { all8 = false; break; }
    const len = a.length;
    const hdr = (small: number, big: number) => { if (len < 256) { this.u8(small); this.u8(len); } else { this.u8(big); this.int(len); } };
    if (all8) { hdr(0xc0, 0xc8); for (const e of a) this.chunks.push(e); }
    else if (all16) { hdr(0xc1, 0xc9); for (const e of a) this.chunks.push((e >> 8) & 0xff, e & 0xff); }
    else { hdr(0xc4, 0xcc); for (const e of a) this.int(e); }
  }
  simpleMap(map: { [k: string]: BintexValue }): void {
    const keys = Object.keys(map);
    if (keys.length === 0) { this.chunks.push(0x90); return; }
    if (keys.length > 255) throw new Error('entries of map max 255');
    this.chunks.push(0x91, keys.length);
    for (const k of keys) {
      this.chunks.push(k.length); // key chars must be <= 0xff, 8-bit
      for (let i = 0; i < k.length; i++) this.chunks.push(k.charCodeAt(i));
      const v = map[k];
      if (typeof v === 'string') this.valueString(v);
      else if (typeof v === 'number') this.valueInt(v);
      else if (Array.isArray(v)) this.valueIntArray(v);
      else if (v && typeof v === 'object') this.simpleMap(v);
      else throw new Error('map entry values must be string, int, int array or simple map');
    }
  }
}

// ---------------------------------------------------------------- Bintex reader
export class BintexReader {
  constructor(public buf: Uint8Array, public pos = 0) {}
  u8(): number { return this.buf[this.pos++]; }
  u16(): number { const r = (this.buf[this.pos] << 8) | this.buf[this.pos + 1]; this.pos += 2; return r; }
  int(): number { const b = this.buf; const p = this.pos; this.pos += 4; return (b[p] << 24) | (b[p + 1] << 16) | (b[p + 2] << 8) | b[p + 3]; } // signed
  varUint(): number {
    const f = this.u8();
    if ((f & 0x80) === 0) return f;
    if ((f & 0xc0) === 0x80) return ((f & 0x3f) << 8) | this.u8();
    if ((f & 0xe0) === 0xc0) { const n1 = this.u8(), n0 = this.u8(); return ((f & 0x1f) << 16) | (n1 << 8) | n0; }
    if ((f & 0xf0) === 0xe0) { const n2 = this.u8(), n1 = this.u8(), n0 = this.u8(); return ((f & 0x0f) << 24) | (n2 << 16) | (n1 << 8) | n0; }
    if (f === 0xf0) return this.int();
    throw new Error('unknown first byte in varuint: ' + f);
  }
  private valueIntT(t: number): number {
    if (t === 0x0e) return 0;
    if (t >= 1 && t <= 7) return t;
    if (t === 0x0f) return -1;
    let a: number;
    if (t === 0x10 || t === 0x11) a = this.u8();
    else if (t === 0x20 || t === 0x21) a = this.u16();
    else if (t === 0x30 || t === 0x31) a = (this.u8() << 16) | (this.u8() << 8) | this.u8();
    else if (t === 0x40 || t === 0x41) a = this.int();
    else throw new Error('value is not int: ' + t.toString(16));
    return (t & 1) === 1 && t >= 0x10 ? ~a : a;
  }
  private str8(len: number): string { let s = ''; for (let i = 0; i < len; i++) s += String.fromCharCode(this.u8()); return s; } // Latin-1
  private str16(len: number): string { let s = ''; for (let i = 0; i < len; i++) s += String.fromCharCode(this.u16()); return s; }
  private valueStringT(t: number): string | null {
    if (t === 0x0c) return null;
    if (t === 0x0d) return '';
    if (t >= 0x51 && t <= 0x5f) return this.str8(t & 0x0f);
    if (t >= 0x61 && t <= 0x6f) return this.str16(t & 0x0f);
    if (t === 0x70) return this.str8(this.u8());
    if (t === 0x71) return this.str16(this.u8());
    if (t === 0x72) return this.str8(this.int());
    if (t === 0x73) return this.str16(this.int());
    throw new Error('value is not string: ' + t.toString(16));
  }
  private intArrayT(t: number): number[] {
    const small = t === 0xc0 || t === 0xc1 || t === 0xc4;
    if (![0xc0, 0xc8, 0xc1, 0xc9, 0xc4, 0xcc].includes(t)) throw new Error('value is not int array: ' + t.toString(16));
    const len = small ? this.u8() : this.int();
    const out: number[] = new Array(len);
    for (let i = 0; i < len; i++) out[i] = (t === 0xc0 || t === 0xc8) ? this.u8() : (t === 0xc1 || t === 0xc9) ? this.u16() : this.int();
    return out;
  }
  valueString(): string | null { return this.valueStringT(this.u8()); }
  simpleMap(): { [k: string]: BintexValue } { return this.simpleMapT(this.u8()); }
  private simpleMapT(t: number): { [k: string]: BintexValue } {
    if (t === 0x90) return {};
    if (t !== 0x91) throw new Error('value is not simple map: ' + t.toString(16));
    const size = this.u8();
    const res: { [k: string]: BintexValue } = {};
    for (let i = 0; i < size; i++) { const k = this.str8(this.u8()); res[k] = this.value(); }
    return res;
  }
  value(): BintexValue {
    const t = this.u8();
    if ((t >= 1 && t <= 7) || t === 0x0e || t === 0x0f || [0x10, 0x11, 0x20, 0x21, 0x30, 0x31, 0x40, 0x41].includes(t)) return this.valueIntT(t);
    if (t === 0x0c || t === 0x0d || (t >= 0x51 && t <= 0x5f) || (t >= 0x61 && t <= 0x6f) || (t >= 0x70 && t <= 0x73)) return this.valueStringT(t);
    if ([0xc0, 0xc1, 0xc4, 0xc8, 0xc9, 0xcc].includes(t)) return this.intArrayT(t);
    if (t === 0x90 || t === 0x91) return this.simpleMapT(t);
    throw new Error('value has unknown type: ' + t.toString(16));
  }
}

// ---------------------------------------------------------------- Ari
export const ari = (book: number, ch: number, v: number) => ((book & 0xff) << 16) | ((ch & 0xff) << 8) | (v & 0xff);
export const ariBook = (a: number) => (a & 0x00ff0000) >> 16;
export const ariChapter = (a: number) => (a & 0x0000ff00) >> 8;
export const ariVerse = (a: number) => a & 0xff;

// ---------------------------------------------------------------- UTF-8 (repo-compatible: BMP only on decode)
const te = new TextEncoder();
export function utf8Bytes(s: string): Uint8Array { return te.encode(s); } // NOTE: Java getBytes("utf-8") encodes lone surrogates as '?'
export function utf8DecodeBmp(b: Uint8Array, start: number, len: number): string {
  let s = '';
  for (let i = start; i < start + len; i++) {
    const c0 = b[i];
    if (c0 < 0x80) { s += String.fromCharCode(c0); continue; }
    const c1 = b[++i];
    if (c0 < 0xe0) { s += String.fromCharCode(((c0 & 0x1f) << 6) | (c1 & 0x3f)); continue; }
    const c2 = b[++i];
    s += String.fromCharCode(((c0 & 0x0f) << 12) | ((c1 & 0x3f) << 6) | (c2 & 0x3f));
  }
  return s;
}

// ---------------------------------------------------------------- Model
export interface Yes2BookInfo {
  bookId: number; shortName: string | null; abbreviation?: string | null;
  offset: number; chapter_count: number; verse_counts: number[]; chapter_offsets: number[]; // length chapter_count+1
}
export interface VersionInfo { shortName?: string; longName?: string; description?: string; locale?: string; book_count: number; hasPericopes: number; textEncoding: number; buildTime?: number }
export interface PericopeEntry { ari: number; title: string | null; parallels?: string[] }
export interface Yes2File {
  version: VersionInfo; books: Yes2BookInfo[]; pericopes: PericopeEntry[];
  text: Uint8Array; // DEcompressed 'text' section content
  xrefs?: Map<number, string | null>; footnotes?: Map<number, string | null>;
}
export interface Snappy { compress(src: Uint8Array): Uint8Array; decompress(src: Uint8Array): Uint8Array } // raw snappy format
const MAGIC = [0x98, 0x58, 0x0d, 0x0a, 0x00, 0x5d, 0xe0, 0x02];

// ---------------------------------------------------------------- Section helpers
function writeIndexedStrings(map: Map<number, string | null>): Uint8Array { // xrefs / footnotes (YetFileOutput side: XrefDb.writeXrefEntriesTo)
  const keys = [...map.keys()].sort((a, b) => (a >>> 0) - (b >>> 0)); // reader needs UNSIGNED ascending order
  const w = new BintexWriter();
  w.u8(1); w.int(keys.length);
  for (const k of keys) w.int(k);
  const contents = new BintexWriter();
  for (const k of keys) { w.int(contents.pos); contents.valueString(map.get(k)!); }
  w.raw(contents.bytes());
  return w.bytes();
}
function readIndexedStrings(b: Uint8Array): { arifs: number[]; get(arif: number): string | null | undefined } {
  const r = new BintexReader(b);
  if (r.u8() !== 1) throw new Error('version');
  const n = r.int();
  const arifs: number[] = [], offs: number[] = [];
  for (let i = 0; i < n; i++) arifs.push(r.int());
  for (let i = 0; i < n; i++) offs.push(r.int());
  const base = r.pos;
  return {
    arifs,
    get(arif: number) {
      let lo = 0, hi = n - 1; // unsigned binary search
      while (lo <= hi) {
        const mid = (lo + hi) >>> 1; const c = (arifs[mid] >>> 0) - (arif >>> 0);
        if (c < 0) lo = mid + 1; else if (c > 0) hi = mid - 1; else return new BintexReader(b, base + offs[mid]).valueString();
      }
      return undefined;
    },
  };
}
function writePericopes(entries: PericopeEntry[]): Uint8Array {
  const w = new BintexWriter();
  w.u8(3);
  const sizePos = w.pos; w.int(-1);
  w.int(entries.length);
  const offPos: number[] = [];
  let last = 0;
  for (const e of entries) {
    const diff = e.ari - last;
    if (last === 0 || diff < 0 || diff > 0x7fff) w.int(e.ari & 0x00ffffff); else w.u16(0x8000 | diff);
    last = e.ari;
    offPos.push(w.pos); w.u16(0xffff);
  }
  const dataBegin = w.pos;
  const offs: number[] = [];
  const blocks = new BintexWriter();
  for (const e of entries) {
    offs.push(blocks.pos);
    blocks.u8(4); blocks.valueString(e.title); blocks.u8(e.parallels ? e.parallels.length : 0);
    for (const p of e.parallels ?? []) blocks.valueString(p);
  }
  const head = w.bytes(); const out = new Uint8Array(head.length + blocks.bytes().length);
  out.set(head); out.set(blocks.bytes(), head.length);
  const dv = new DataView(out.buffer);
  dv.setInt32(sizePos, out.length); // whole section size
  let lastOff = 0;
  offs.forEach((o, i) => { const d = o - lastOff; if (d > 0xffff) throw new Error('pericope entry > 65535 bytes'); dv.setUint16(offPos[i], d); lastOff = o; });
  void dataBegin;
  return out;
}
function readPericopes(b: Uint8Array): PericopeEntry[] {
  const r = new BintexReader(b);
  const v = r.u8(); if (v !== 2 && v !== 3) throw new Error('pericope version');
  r.int(); const n = r.int();
  const aris: number[] = [], offs: number[] = [];
  if (v === 2) for (let i = 0; i < n; i++) { aris.push(r.int()); offs.push(r.int()); }
  else { let la = 0, lo = 0; for (let i = 0; i < n; i++) { const d = r.u16(); const a = (d & 0x8000) === 0 ? (d << 16) | r.u16() : la + (d & 0x7fff); aris.push(la = a); lo += r.u16(); offs.push(lo); } }
  const base = r.pos;
  return aris.map((a, i) => {
    const br = new BintexReader(b, base + offs[i]);
    if (br.u8() !== 4) throw new Error('pericope block version');
    const title = br.valueString(); const pc = br.u8(); const parallels: string[] = [];
    for (let j = 0; j < pc; j++) parallels.push(br.valueString() as string);
    return { ari: a, title, parallels };
  });
}

// ---------------------------------------------------------------- Generator
export const COMPRESS_BLOCK_SIZE = 32768;
function sectionBody(content: Uint8Array, snappy?: Snappy): { attrs: { [k: string]: BintexValue }; body: Uint8Array } {
  if (!snappy) return { attrs: {}, body: content };
  const sizes: number[] = []; const parts: Uint8Array[] = [];
  for (let o = 0; o < content.length; o += COMPRESS_BLOCK_SIZE) { const c = snappy.compress(content.subarray(o, Math.min(o + COMPRESS_BLOCK_SIZE, content.length))); parts.push(c); sizes.push(c.length); }
  const body = new Uint8Array(sizes.reduce((a, b) => a + b, 0)); let p = 0; for (const c of parts) { body.set(c, p); p += c.length; }
  return { attrs: { 'compression.name': 'snappy-blocks', 'compression.version': 1, 'compression.info': { block_size: COMPRESS_BLOCK_SIZE, compressed_block_sizes: sizes } }, body };
}

/** chapters[bookIndex][chapterIndex] = verse strings. Books must be in ascending bookId order. */
export function writeYes2(opts: {
  info: { shortName?: string; longName?: string; description?: string; locale?: string; buildTime?: number };
  books: { bookId: number; shortName: string; abbreviation?: string; chapters: string[][] }[];
  pericopes?: PericopeEntry[]; xrefs?: Map<number, string>; footnotes?: Map<number, string>; snappy?: Snappy;
}): Uint8Array {
  // text + booksInfo
  const text = new BintexWriter();
  const booksInfo: Yes2BookInfo[] = [];
  let offsetTotal = 0;
  for (const b of opts.books) {
    const chapter_offsets = [0]; const verse_counts: number[] = []; let passed = 0;
    for (const ch of b.chapters) {
      verse_counts.push(ch.length);
      for (const v of ch) { const vb = utf8Bytes(v); const vw = new BintexWriter(); vw.varUint(vb.length); vw.raw(vb); text.raw(vw.bytes()); passed += vw.bytes().length; }
      chapter_offsets.push(passed);
    }
    booksInfo.push({ bookId: b.bookId, shortName: b.shortName, abbreviation: b.abbreviation, offset: offsetTotal, chapter_count: b.chapters.length, verse_counts, chapter_offsets });
    offsetTotal += passed;
  }
  const vi = new BintexWriter();
  const viMap: { [k: string]: BintexValue } = { version: 3 };
  if (opts.info.shortName != null) viMap.shortName = opts.info.shortName;
  if (opts.info.longName != null) viMap.longName = opts.info.longName;
  if (opts.info.description != null) viMap.description = opts.info.description;
  if (opts.info.locale != null) viMap.locale = opts.info.locale;
  if (opts.info.buildTime) viMap.buildTime = opts.info.buildTime;
  viMap.book_count = opts.books.length; viMap.hasPericopes = opts.pericopes ? 1 : 0; viMap.textEncoding = 2;
  vi.simpleMap(viMap);
  const bi = new BintexWriter(); bi.int(booksInfo.length);
  for (const b of booksInfo) {
    const m: { [k: string]: BintexValue } = { version: 3, bookId: b.bookId, shortName: b.shortName, offset: b.offset, chapter_count: b.chapter_count, verse_counts: b.verse_counts, chapter_offsets: b.chapter_offsets };
    if (b.abbreviation != null) m.abbreviation = b.abbreviation;
    bi.simpleMap(m);
  }
  type Sec = { name: string; attrs: Uint8Array; content: Uint8Array };
  const secs: Sec[] = [];
  const plain = (name: string, content: Uint8Array): Sec => ({ name, attrs: Uint8Array.of(0x90), content });
  const comp = (name: string, content: Uint8Array): Sec => { const { attrs, body } = sectionBody(content, opts.snappy); const aw = new BintexWriter(); aw.simpleMap(attrs); return { name, attrs: aw.bytes(), content: body }; };
  secs.push(plain('versionInfo', vi.bytes()), plain('booksInfo', bi.bytes()));
  if (opts.pericopes) secs.push(comp('pericopes', writePericopes(opts.pericopes)));
  secs.push(comp('text', text.bytes()));
  if (opts.xrefs) secs.push(comp('xrefs', writeIndexedStrings(opts.xrefs)));
  if (opts.footnotes) secs.push(comp('footnotes', writeIndexedStrings(opts.footnotes)));

  const out = new BintexWriter();
  out.raw(MAGIC);
  let indexBytes = 0; for (const s of secs) indexBytes += 1 + s.name.length + 16;
  out.int(indexBytes); // sectionIndex.size = bytes of the entries only
  out.u8(1); out.int(secs.length);
  let off = 0;
  for (const s of secs) { out.u8(s.name.length); for (let i = 0; i < s.name.length; i++) out.u8(s.name.charCodeAt(i)); out.int(off); out.int(s.attrs.length); out.int(s.content.length); out.int(0); off += s.attrs.length + s.content.length; }
  for (const s of secs) { out.raw(s.attrs); out.raw(s.content); }
  out.u8(0); // footer
  return out.bytes();
}

// ---------------------------------------------------------------- Parser
export function parseYes2(file: Uint8Array, snappy?: Snappy): Yes2File {
  for (let i = 0; i < 8; i++) if (file[i] !== MAGIC[i]) throw new Error('YES2: Header is incorrect');
  const r = new BintexReader(file, 12); // skip header(8) + sectionIndex.size(4)
  if (r.u8() !== 1) throw new Error('unsupported section index version');
  const count = r.int();
  const idx = new Map<string, { offset: number; attrSize: number; contentSize: number }>();
  for (let i = 0; i < count; i++) {
    const name = String.fromCharCode(...file.subarray(r.pos + 1, r.pos + 1 + file[r.pos])); r.pos += 1 + name.length;
    const offset = r.int(), attrSize = r.int(), contentSize = r.int(); r.pos += 4;
    idx.set(name, { offset, attrSize, contentSize });
  }
  const base = r.pos;
  const section = (name: string): Uint8Array | null => {
    const e = idx.get(name); if (!e) return null;
    const attrs = new BintexReader(file, base + e.offset).simpleMap();
    const raw = file.subarray(base + e.offset + e.attrSize, base + e.offset + e.attrSize + e.contentSize);
    const cname = attrs['compression.name'];
    if (cname == null) return raw;
    if (cname !== 'snappy-blocks') throw new Error('Compression ' + cname + ' is not supported');
    if (!snappy) throw new Error('snappy required');
    if (((attrs['compression.version'] as number) ?? 0) > 1) throw new Error('compression version');
    const info = attrs['compression.info'] as { block_size: number; compressed_block_sizes: number[] };
    const parts: Uint8Array[] = []; let p = 0;
    for (const sz of info.compressed_block_sizes) { parts.push(snappy.decompress(raw.subarray(p, p + sz))); p += sz; }
    const total = parts.reduce((a, b) => a + b.length, 0); const out = new Uint8Array(total); let q = 0; for (const x of parts) { out.set(x, q); q += x.length; }
    return out;
  };
  const viRaw = section('versionInfo'); if (!viRaw) throw new Error('no versionInfo');
  const vm = new BintexReader(viRaw).simpleMap();
  const num = (v: BintexValue | undefined, d: number) => (typeof v === 'number' ? v : d); // NB: Java getInt returns 0 (not def) when key missing
  const version: VersionInfo = { shortName: vm.shortName as string, longName: vm.longName as string, description: vm.description as string, locale: vm.locale as string, book_count: num(vm.book_count, 0), hasPericopes: num(vm.hasPericopes, 0), textEncoding: num(vm.textEncoding, 0), buildTime: num(vm.buildTime, 0) };
  const biRaw = section('booksInfo'); if (!biRaw) throw new Error('no booksInfo');
  const br = new BintexReader(biRaw); const bc = br.int(); const books: Yes2BookInfo[] = [];
  for (let i = 0; i < bc; i++) { const m = br.simpleMap(); books.push({ bookId: num(m.bookId, -1), shortName: (m.shortName as string) ?? null, abbreviation: (m.abbreviation as string) ?? null, offset: num(m.offset, 0), chapter_count: num(m.chapter_count, 0), verse_counts: m.verse_counts as number[], chapter_offsets: m.chapter_offsets as number[] }); }
  const per = version.hasPericopes ? section('pericopes') : null;
  const xr = section('xrefs'), fn = section('footnotes');
  const toMap = (b: Uint8Array | null) => { if (!b) return undefined; const x = readIndexedStrings(b); const m = new Map<number, string | null>(); for (const a of x.arifs) m.set(a, x.get(a)!); return m; };
  return { version, books, pericopes: per ? readPericopes(per) : [], text: section('text')!, xrefs: toMap(xr), footnotes: toMap(fn) };
}

export function loadChapter(f: Yes2File, book: Yes2BookInfo, chapter1: number): string[] {
  if (chapter1 <= 0 || chapter1 > book.chapter_count) return [];
  const r = new BintexReader(f.text, book.offset + book.chapter_offsets[chapter1 - 1]);
  const out: string[] = [];
  for (let i = 0; i < book.verse_counts[chapter1 - 1]; i++) {
    const len = r.varUint();
    out.push(f.version.textEncoding === 1 ? String.fromCharCode(...f.text.subarray(r.pos, r.pos + len)) : utf8DecodeBmp(f.text, r.pos, len));
    r.pos += len;
  }
  return out;
}
```

Parser flow in words (Derivation, mirrors `Yes2Reader`): validate 8-byte header; read at offset 12: `u8 version == 1`, `int section_count`, entries (`u8 nameLen`, name bytes, `int offset`, `int attrSize`, `int contentSize`, skip 4); `base` = position after the last entry; for each wanted section read attributes map at `base + offset`, content at `base + offset + attrSize`; if `compression.name == "snappy-blocks"` decompress block-by-block (or lazily by block index = floor(logicalOffset / block_size)); parse `versionInfo`, `booksInfo`; read verses lazily by `book.offset + chapter_offsets[chapter-1]`.

---

## 9. Edge cases, gotchas, and discrepancies

1. **`ValueMap.getInt` ignores the default for missing keys** (returns 0) (`ValueMap.java:82`) — see section 2.
2. **Section index `size` is the entries length only**, and the reader never reads it (section 3.2). Same for `content_size`, `index_size` in pericopes, the footer byte, `buildTime`, and `version` fields in maps.
3. **Offsets are always in decompressed space** (section 3.4).
4. **Xref/footnote arifs must be sorted unsigned**, but the writers do not sort (section 4.5).
5. **Pericope entries must be sorted by ari** and must each be < 64 KB (section 4.4). Relative ari delta of exactly 0 is encoded relatively (`diff_ari == 0` passes the `0 <= diff <= 0x7fff` test) except for the very first entry, because the check is `last_entry_ari == 0 ||` (`PericopesSection.java:75`); an entry whose ari is 0 would be ambiguous but is not a valid verse anyway.
6. **UTF-8 decoder is BMP-only** (section 4.3). The writer encodes with the JVM's UTF-8 which writes 4-byte sequences for supplementary characters; the Android reader then garbles them.
7. **String lengths are UTF-16 code units** in Bintex string values; a TS port gets this for free from `String.length`.
8. **Bintex 8-bit strings are Latin-1**, so xref/footnote/pericope strings containing any char above U+00FF are stored as UTF-16BE (type `6x`/`71`/`73`), not UTF-8. The `text` section is the only place real UTF-8 appears.
9. **Book table is sparse**: `booksInfo` lists only books present; array position != bookId. YES1's loader likewise places books into a 256-slot array by `bookId` (`Yes1Reader.java:189`, `:256-259`, `:263-270`).
10. **`TextSectionReader`/`loadVerseText` depend on `versionInfo_` having been loaded** (via `loadBooks`/getters) before the first `loadVerseText`; otherwise `versionInfo_` is null (`Yes2Reader.java:219`). A TS parser should simply load it eagerly.
11. **`SnappyInputStream` EOF handling:** after the last block it increments `current_block_index` to `compressed_block_sizes.length` and calls `prepareBuffer`, which indexes `compressed_block_sizes[length]` out of bounds (`SnappyInputStream.java:94-101`, `:52-54`). Only reachable when reading past the end of the section; ignore in a TS port.
12. **Section name quirk:** `docs/binary-formats.md` calls the pericope section `pericopies`; the code name is `pericopes`. It also refers to a `Yes2PericopeIndex` class that does not exist (pericope index code is in `PericopesSection` + `model/PericopeIndex`), describes the section index name as "length-prefixed string" (it is `u8` length + raw bytes) and lists `booksInfo` as having "names, chapter counts, verse counts" without the offset arrays. Trust the code.
13. **Max sizes:** simple map max 255 keys (`BintexWriter.java:472-474`); section-name max 255; uint8 arrays only if all values <= 255; `parallel_count` and `data_format_version` are `uint8`.
14. **Verse/chapter limits via ari:** 255 chapters, 255 verses for ari-keyed data (pericopes, xrefs, footnotes); book id 0..255.
15. **Internal (APK-bundled) format** and **YES1** are separate formats (YES1 detailed below); the Android app's downloadable versions are YES2 only (`docs/binary-formats.md` "Differences from YES2").

### 9.1 YES1 (legacy, read-only; only for completeness)

Everything below is **Fact** from `Yes1Reader.java` unless noted; no writer exists in the repo, so details of production are unknown.

- Header: 8 bytes `98 58 0d 0a 00 5d e0 01` (`:83`). Then a sequence of sections, each: **12-byte ASCII name** (padded with `_`, e.g. `infoEdisi___`, `infoKitab___`, `teks________`, `perikopIndex`, `perikopBlok_`), `int size` (big-endian, `RandomAccessFile.readInt`), then `size` bytes of data (`readSectionName` `:325-329`, `readSectionSize` `:331-333`, `skipUntilSection` `:47-68`). The sequence ends at EOF or a section named `____________`.
- `infoEdisi___` (version info): a stream of `short string key` + value, terminated by key `end`. Keys: `versi` (int, must be <= 2), `format` (int, deprecated), `nama` (short string), `shortName`/`shortTitle` (short string), `judul` (short string = longName), `keterangan` (long string = description), `nkitab` (int = book_count), `perikopAda` (int = has pericopes), `encoding` (int, 1 = ascii, 2 = utf-8; default 1), `locale` (short string); unknown key throws (`:122-180`). Here "short string" is the 16-bit `u8 len + chars` form (section 2.1).
- `infoKitab___`: for each of `book_count` books, a key/value stream terminated by `end`: `versi`, `pos` (int = bookId), `nama`/`judul` (short string), `npasal` (int = chapter_count), `nayat` (`uint8` per chapter), `ayatLoncat`/`pdbBookNumber`/`encoding` (int, ignored), `pasal_offset` (`chapter_count + 1` ints), `offset` (int); a book whose first key is `end` is an empty slot (`:183-277`).
- `teks________`: raw text; chapter bytes at `text_baseOffset + book.offset + chapter_offsets[c-1]`, length `chapter_offsets[c] - chapter_offsets[c-1]` (`:280-322`). Verses are **separated by `\n` (0x0A)**, no length prefixes (`OldVerseTextDecoder.kt:6-68`: ASCII or UTF-8 split on newlines; note the `Ascii` decoder only supports bytes < 0x80 and has the 4000-char buffer issue per `REM-48`). The decoder drops a trailing verse that is not newline-terminated (the loop only appends on `\n`).
- `perikopIndex`: `int entry_count` then `(int ari, int offset)` pairs (`Yes1PericopeIndex.java:11-25`). `perikopBlok_`: blocks read sequentially; offsets are absolute positions in the stream the Bintex reader counts from the start of the block section (`Yes1PericopeIndex.getBlock` `:27-44` skips `offset - currentPos`). Block versions 1/2/3: `u8 version`, title (short string / long string / auto string), `u8 count`, parallels (short strings, or auto strings in v3) (`Yes1PericopeBlock.java:30-59`).
- No xref/footnote sections.

---

## 10. Open questions (not resolved from source)

1. **No real `.yes` sample in the repo** to confirm byte-level interoperability of my TS writer/reader against Java output (only `tools/in-ddd/in-ddd.yet` as input). Next step: parse the app's `assets/bible/kjv.yes` and diff `sectionIndex.size`, book offsets, and re-generate it.
2. **Snappy compatibility between the two backends** (native C `snappy_compress` vs the pure-Java `de.jarnbjo.jsnappy` fallback chosen in `Snappy.Factory.newInstance`, `Snappy.java:36-60`): both must produce valid raw Snappy for the reader to work, but I did not read the Java compressor/decompressor to confirm. A JS Snappy decoder is the only hard requirement for reading.
3. **Whether `reserved` bytes or `content_size` are ever non-zero / meaningful in files in the wild**: no code reads them, no code writes anything but 0 / the true size.
4. **Larger books/chapters (>255 verses/chapters) or book ids >= 66** (deuterocanon): the text/books sections allow them, but ari-keyed data cannot represent chapters/verses over 255; I did not find a documented policy.
5. **Why `index_size` in the pericopes header stores the whole section size** (`PericopesSection.java:114-118`): inferred as a historical quirk; the reader ignores it.
6. **Ruby (`@<r=...@>`) and other tags** beyond `f`/`x`: only the Compose renderer was cited; I did not read its tag grammar beyond the doc pointer (`docs/features/ruby/design.md`).
7. **Other converters** (PDB/OSIS/USFM -> `.yet`) under `tools/AlkitabConverterProcesses/` and `BiblePlus` were located by grep but not read; they were not needed to define the container format. `docs/binary-formats.md:294` states the BiblePlus module converts PDB to YES2 on import, unverified here.

## 11. Round-trip validation against a real file (added 2026-10-02)

Fact (observed, not from the repo): `assets/bible/kjv.yes` (1,941,000 bytes, KJV, 66 books, 31,102 verses, 138 pericopes) was parsed with the section 8 `parseYes2` and regenerated with `writeYes2`.

- Section order is `versionInfo, booksInfo, pericopes, text`; no `xrefs`/`footnotes`. `text` and `pericopes` are `snappy-blocks` compressed; `versionInfo`/`booksInfo` are not. This confirms the section 8 layout against a Java-produced file.
- Regenerated `versionInfo` and `booksInfo` are **byte-identical** to the originals (196 and 12,027 bytes including the one-byte empty attribute map).
- Decompressed `pericopes` (6,490 bytes) and `text` (4,288,559 bytes) are **byte-identical** to the originals, and the parsed version info, books and pericopes compare equal, both for uncompressed output and for output compressed with a literal-only Snappy stand-in.
- Not identical: the full file bytes when compressed. The original's compressed blocks come from a real Snappy compressor; a byte-exact match requires the same compressor. Any valid raw Snappy stream decodes the same, so this does not affect compatibility.
- A pure raw-Snappy decoder (about 15 lines) was enough to read the real file; no framing is involved.
- Still open: xrefs/footnotes against a real file, and `reserved` bytes (all zero in this file).
