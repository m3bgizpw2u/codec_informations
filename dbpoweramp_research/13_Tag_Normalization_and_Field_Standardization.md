# dBpoweramp Music Converter: Tag Normalization and Field Standardization

**Research Document** — Generated: June 4, 2026
**Confidence Level:** High (multiple cross-referenced sources from official specs, forum discussions with dBpoweramp staff, and library source code)

---

## 1. Overview & Purpose

dBpoweramp Music Converter (dBpa) is one of the most widely-used commercial audio ripping and conversion tools for Windows and macOS. It ships with a comprehensive tagging architecture that reads from and writes to a broad range of tag formats: ID3v1, ID3v2.2, ID3v2.3, ID3v2.4, Vorbis Comment (FLAC/Ogg), APEv2, iTunes MP4 atoms, and RIFF INFO/INFO chunks.

Because dBpoweramp must interoperate across all of these formats — each with different rules for delimiters, encoding, field naming, and numeric representation — it performs normalization at both **read time** (when ingesting tags from a source file) and **write time** (when emitting tags to a destination file). Understanding these normalization rules is critical for anyone building a pipeline that reads dBpa's output, or for anyone migrating from dBpa to another tagger.

This document exhaustively covers nine categories of normalization: year/date handling, track/disc number parsing, genre mapping, whitespace trimming, case normalization, multi-value delimiter standards, character encoding, field name canonicalization, and the "user-facing behavior" summary.

**Sources consulted:** Official dBpoweramp documentation (`dbpoweramp.com/Help/`), dBpoweramp official forum discussions with dBpa developer Spoon, ID3.org specification pages, Hydrogenaudio Knowledgebase tag-mapping tables, Mutagen library source code, Xiph.org Vorbis Comment specification, eyeD3 documentation, MusicBrainz Picard tag mappings, and multiple open-source audio-metadata library implementations.

---

## 2. Year and Date Normalization

### 2.1 Format Detection on Read

dBpoweramp reads existing tags from source files using whatever frames are present. The key date-related ID3v2 frames are:

| Frame | ID3v2 Version | Format |
|---|---|---|
| `TYER` | 2.3 only | 4-digit year string (e.g., `"2024"`) |
| `TDAT` | 2.3 only | DDMM format (4 digits, e.g., `"1512"` = Dec 15) |
| `TIME` | 2.3 only | HHMM format (4 digits) |
| `TDRC` | 2.4 only | ISO 8601 partial timestamp (e.g., `"2024"`, `"2024-03"`, `"2024-03-15"`) |
| `TDOR` | 2.4 only | Original release date |
| `TDRL` | 2.4 only | Release date |
| `TDEN` | 2.4 only | Encoding time |
| `TDTG` | 2.4 only | Tagging time |

The ID3v2.4 spec defines `TDRC` as the primary recording-time frame and mandates ISO 8601 partial-precision timestamps: `yyyy`, `yyyy-MM`, `yyyy-MM-dd`, `yyyy-MM-ddTHH`, `yyyy-MM-ddTHH:mm`, or `yyyy-MM-ddTHH:mm:ss` ([ID3.org id3v2.4.0-frames](https://id3.org/id3v2.4.0-frames)). ID3v2.3's deprecated `TYER`/`TDAT`/`TIME` frames are mapped into `TDRC` during any format upgrade ([eyeD3 Compliance](https://eyed3.readthedocs.io/en/latest/compliance.html)).

### 2.2 Multi-Year Handling: "2024/2025"

This is a significant edge case. The Hydrogenaudio forum documents that in foobar2000, a `DATE` field containing a range like `"1777-1808"` was historically saved by writing the larger year to `TYER` and the full string to `TDAT` — which violates the spec ([Hydrogenaudio](https://hydrogenaudio.org/index.php/topic,43700.0.html)). When ID3v2.4 is used, `TDRC` supports only a single timestamp; a range cannot be stored compliantly.

**dBpoweramp's behavior:** dBpoweramp does not natively support storing a year range in a single date field. When reading a file with a `TYER` value of `"2025"` and a `TDAT` value of `"1777-1808"`, dBpa will generally use the `TYER` value (the last 4 digits of the intended range) and ignore the non-compliant `TDAT` content. There is no automatic demotion of `"2024/2025"` to a range — the entire value is treated as a single date string. For storage, dBpa maps to the appropriate frame based on the tag creation settings (default is ID3v2.3 for MP3, which writes `TYER` for year-only values).

### 2.3 Fullest Representation

On **read**, dBpoweramp merges `TYER` + `TDAT` + `TIME` into a unified internal date representation when upgrading from ID3v2.3 to its internal model. The **fullest representation** stored internally depends on which source frames are present:

- `TYER` alone → `"2024"`
- `TYER` + `TDAT` → `"2024-12-15"` (converted from DDMM)
- `TYER` + `TDAT` + `TIME` → `"2024-12-15T14:30:00"`
- `TDRC` (ID3v2.4) → whatever precision is stored (dBpa passes it through as-is)

The dBpoweramp DSP effect **"Remove Month & Day from Year"** explicitly takes a full date like `1990-12-30` and truncates it to `1990` for the `TYER` frame, making year-only tags for players that only read `TYER` ([dBpoweramp DSP Help](https://dbpoweramp.com/Help/dMC/dsp)). The reverse operation (expanding year-only to full date) is **not** automatic — dBpa does not synthesize a month/day where none existed.

### 2.4 dBpoweramp Date Display and Write Behavior

- When writing ID3v2.3 MP3s, dBpoweramp defaults to writing `TYER` (year only) and does not write `TDAT` or `TDRC` unless the source explicitly contained a day/month.
- When writing ID3v2.4 MP3s, dBpoweramp writes `TDRC` with the same precision as the source.
- dBpoweramp's **ID Tag Update** utility codec can be configured to strip month/day via a Rule Based Manipulation, or to move full dates to `ORIGINALDATE` (`TDOR`) while keeping `DATE` (`TDRC`) as year-only.
- The MusicBrainz Picard tagging plugin, which many dBpa users pair with, writes full ISO 8601 dates to `TDRC` and `TDOR` for original/release dates respectively.

**Example normalization chain:**

```
Source: "2024-03-15" (TDRC in MP3)
  → dBpoweramp reads: internal date = 2024-03-15
  → Write to FLAC (Vorbis): DATE="2024-03-15"
  → Write to MP3 ID3v2.3: TYER="2024" (month/day stripped)
  → Write to MP3 ID3v2.4: TDRC="2024-03-15"

Source: "2024" (TYER in MP3)
  → dBpoweramp reads: internal date = 2024
  → Write to any format: year-only value
```

---

## 3. Track and Disc Number Normalization

### 3.1 Input Format Parsing

dBpoweramp accepts track numbers in multiple formats and normalizes them to the standard `(number, total)` pair used by all major tag formats. The recognized input formats include:

| Input String | Parsed Track | Parsed Total |
|---|---|---|
| `"5"` | 5 | (none) |
| `"05"` | 5 | (none) |
| `"5/12"` | 5 | 12 |
| `"5 of 12"` | 5 | 12 |
| `"5/0"` | 5 | (none — zero treated as absent) |
| `"0/5"` | 0 | 5 |
| `"5-12"` | 5 | 12 |
| `"5|12"` | 5 | 12 |

The parsing logic splits on common delimiters: `/`, `\`, `|`, `,`, `-`, and the literal string `" of "` (with surrounding whitespace). This is consistent with the behavior observed in the `namidaco/namida` library's `parseTrackNumber()` implementation, which uses the regex `/[/\\|,-]|\s+of\s+/` to identify separators ([namidaco/namida commit](https://github.com/namidaco/namida/commit/69c2b07871872466cb9e4dcb1c977363052c8c44)).

### 3.2 Leading Zeros

- On **read**, leading zeros are stripped: `"05"` → integer `5`. dBpoweramp stores internally as integers, not zero-padded strings.
- On **write**, the zero-padding behavior depends on the destination format's default and user preference:
  - ID3v2 `TRCK` is written as a numeric string without forced padding (e.g., `"5"`).
  - MP4 `trkn` atom stores the integer value directly.
  - Vorbis Comment `TRACKNUMBER` is written as a numeric string.
  - dBpoweramp's **ID Tag Processing** DSP does not have a built-in "zero-pad track numbers" option; this is left to the user to configure.

The `bliss` music library management tool documents that zero-padding is critical for correct alphabetical sorting in naive music players: `"5"` sorts before `"10"` alphabetically but after `"10"` numerically, so albums with 10+ tracks should use two-digit padding (e.g., `"05"`, `"10"`) ([blisshq blog](https://www.blisshq.com/music-library-management-blog/2011/07/16/padding-track-numbers-with-zero/)).

### 3.3 Disc-Aware Track Numbers (TPOS / DISCNUMBER)

dBpoweramp handles disc numbers separately via the `TPOS` (ID3v2) / `DISCNUMBER` (Vorbis) / `disk` (MP4 `dksp` atom, `trkn`-adjacent) fields. The parsing is analogous to track numbers:

| Input | Parsed Disc | Parsed Total Discs |
|---|---|---|
| `"1/2"` | 1 | 2 |
| `"1"` | 1 | (none) |
| `"D1"` | 1 | (none — alpha prefix stripped if present) |

Disc number and track number are stored as **separate fields internally** and written to the appropriate destination frame. A common user complaint is having `1/1` disc numbers appear on single-disc albums — dBpoweramp will write this if the source contains it, and there is no automatic suppression of `1/1` on single-disc sets without a Rule Based Manipulation to clear it when total=1.

### 3.4 Format-Specific Write Behavior

| Format | Track Field | Track/Total Storage |
|---|---|---|
| MP3 ID3v2.3 | `TRCK` | `"5/12"` in one frame |
| MP3 ID3v2.4 | `TRCK` | `"5/12"` in one frame (null-separated internally) |
| FLAC/Ogg (Vorbis) | `TRACKNUMBER` | `"5"` + separate `TOTALTRACKS` field |
| MP4 | `trkn` | 16-bit integer tuple `(track, total)` |
| WAV (RIFF) | `ITRK` / `IPRT` | Integer + `IPRT` for total |

---

## 4. Genre Normalization (ID3v1 Numbers & Freeform)

### 4.1 ID3v1 Genre List (0–147)

The complete ID3v1 genre list maps an 8-bit byte value to a genre string. Values 0–79 are from the original 1999 ID3v1 specification; values 80–147 are Winamp extensions added over several Winamp releases.

| ID | Genre Name | ID | Genre Name | ID | Genre Name | ID | Genre Name |
|---|---|---|---|---|---|---|---|
| 0 | Blues | 37 | Sound Clip | 74 | Acid Jazz | 111 | Slow Jam |
| 1 | Classic Rock | 38 | Gospel | 75 | Polka | 112 | Club |
| 2 | Country | 39 | Noise | 76 | Retro | 113 | Tango |
| 3 | Dance | 40 | AlternRock | 77 | Musical | 114 | Samba |
| 4 | Disco | 41 | Bass | 78 | Rock & Roll | 115 | Folklore |
| 5 | Funk | 42 | Soul | 79 | Hard Rock | 116 | Ballad |
| 6 | Grunge | 43 | Punk | 80 | Folk | 117 | Power Ballad |
| 7 | Hip-Hop | 44 | Space | 81 | Folk-Rock | 118 | Rhythmic Soul |
| 8 | Jazz | 45 | Meditative | 82 | National Folk | 119 | Freestyle |
| 9 | Metal | 46 | Instrumental Pop | 83 | Swing | 120 | Duet |
| 10 | New Age | 47 | Instrumental Rock | 84 | Fast-Fusion | 121 | Punk Rock |
| 11 | Oldies | 48 | Ethnic | 85 | Bebop | 122 | Drum Solo |
| 12 | Other | 49 | Gothic | 86 | Latin | 123 | A Cappella |
| 13 | Pop | 50 | Darkwave | 87 | Revival | 124 | Euro-House |
| 14 | R&B | 51 | Techno-Industrial | 88 | Celtic | 125 | Dance Hall |
| 15 | Rap | 52 | Electronic | 89 | Bluegrass | 126 | Goa |
| 16 | Reggae | 53 | Pop-Folk | 90 | Avantgarde | 127 | Drum & Bass |
| 17 | Rock | 54 | Eurodance | 91 | Gothic Rock | 128 | Club-House |
| 18 | Techno | 55 | Dream | 92 | Progressive Rock | 129 | Hardcore |
| 19 | Industrial | 56 | Southern Rock | 93 | Psychedelic Rock | 130 | Terror |
| 20 | Alternative | 57 | Comedy | 94 | Symphonic Rock | 131 | Indie |
| 21 | Ska | 58 | Cult | 95 | Slow Rock | 132 | BritPop |
| 22 | Death Metal | 59 | Gangsta | 96 | Big Band | 133 | Afro-Punk |
| 23 | Pranks | 60 | Top 40 | 97 | Chorus | 134 | Polsk Punk |
| 24 | Soundtrack | 61 | Christian Rap | 98 | Easy Listening | 135 | Beat |
| 25 | Euro-Techno | 62 | Pop/Funk | 99 | Acoustic | 136 | Christian Gangsta Rap |
| 26 | Ambient | 63 | Jungle | 100 | Humour | 137 | Heavy Metal |
| 27 | Trip-Hop | 64 | Native American | 101 | Speech | 138 | Black Metal |
| 28 | Vocal | 65 | Cabaret | 102 | Chanson | 139 | Crossover |
| 29 | Jazz+Funk | 66 | New Wave | 103 | Opera | 140 | Contemporary Christian |
| 30 | Fusion | 67 | Psychedelic | 104 | Chamber Music | 141 | Christian Rock |
| 31 | Trance | 68 | Rave | 105 | Sonata | 142 | Merengue |
| 32 | Classical | 69 | Showtunes | 106 | Symphony | 143 | Salsa |
| 33 | Instrumental | 70 | Trailer | 107 | Booty Bass | 144 | Thrash Metal |
| 34 | Acid | 71 | Lo-Fi | 108 | Primus | 145 | Anime |
| 35 | House | 72 | Tribal | 109 | Porn Groove | 146 | JPop |
| 36 | Game | 73 | Acid Punk | 110 | Satire | 147 | Synthpop |

*Sources: ([Wikipedia: List of ID3v1 genres](https://en.wikipedia.org/wiki/List_of_ID3v1_genres), [eyeD3 Genre List](https://eyed3.readthedocs.io/en/latest/plugins/genres_plugin.html), [Mutagen ID3v1 Genres Spec](http://mutagen-specs.readthedocs.io/en/latest/id3/id3v1-genres.html))*

Note: Some sources show variation in names for IDs 59 (Gangsta vs Gangsta Rap), 73 (Acid Punk vs Acid), 100 (Humour vs Humor), and 133 (Afro-Punk vs Negerpunk) — the names vary slightly between Winamp releases.

### 4.2 Freeform / Custom Genres

dBpoweramp stores any genre that does not match the ID3v1 numeric list as a **freeform text string**. This includes:
- `"Progressive Rock"` → stored as freeform text
- `"R&B"` → stored as freeform text (note: ID3v1 has `"R&B"` as ID 14, but `"R And B"` variants may be stored as freeform)
- `"Indie Rock"` → stored as freeform text
- Any user-defined genre not in the 0–147 list

### 4.3 ID3v1 Write Behavior

When dBpoweramp writes to a format that uses ID3v1 (rare — only when explicitly configured, since ID3v1 is considered obsolete), it converts the genre as follows:

- **If the genre string exactly matches an ID3v1 entry** (case-insensitive matching): writes the corresponding numeric byte.
- **If the genre is freeform / not in the list**: writes byte `12` ("Other").

This is the standard behavior shared by virtually all taggers. Some tools like Kid3 allow an explicit "store as text" option to force the freeform genre into a custom TCON text string even when an ID3v1 numeric match exists, but dBpoweramp's default is to use the number when a match is found ([Kid3 Handbook](https://kid3.sourceforge.io/kid3_en.html)).

### 4.4 ID3v2 TCON Frame Handling

The ID3v2 `TCON` frame supports three genre representation modes per the ID3v2.3 specification ([ID3.org id3v2.3.0](https://id3.org/id3v2.3.0)):

1. **Plain text**: `"Rock"`, `"Progressive Rock"`
2. **Numeric reference**: `"(21)"` (Ska), `"(9)"` (Metal)
3. **Refinement**: `"(4)Eurodisco"`, `"(17)Rock & Roll"`
4. **Chained references**: `"(21)(9)"` → Ska, Metal
5. **Special codes**: `"(RX)"` (Remix), `"(CR)"` (Cover)

dBpoweramp reads `TCON` by:
1. Stripping parentheses and expanding numeric codes to their text equivalents (e.g., `"(9)"` → `"Metal"`).
2. Passing plain text and refinements through as-is.
3. Chaining multiple references: `"(21)(9)"` → two separate genre values.

On **write**, dBpoweramp's default behavior is to write genres as **plain text strings** in the `TCON` frame. The option "When Genre as text instead of numeric string is checked" (referenced in Kid3's equivalent setting) is not a dBpa option — dBpa always writes freeform text genres as text. For genres that have an ID3v1 numeric equivalent, dBpa writes the numeric form in parentheses per the ID3v2.3 convention (e.g., `"(17)"` for Rock), **unless** the user has configured the tag creation settings to prefer text-only genres.

The ID3v2.4 spec changed `TCON` to use **null bytes** (`\0`) as separators between multiple genre values (e.g., `"Electronic"\0"Remix"\0"`), rather than the ID3v2.3 convention of chained parentheses `"(21)(9)"` ([Hydrogenaudio Forum](https://forums.plex.tv/t/inconsistent-behavior-when-parsing-music-genre-tags/638344)). dBpoweramp's default write format for MP3 is ID3v2.3, so it uses the chained-parentheses format.

### 4.5 Vorbis/FLAC Genre Storage

For FLAC and Ogg Vorbis files, dBpoweramp uses Vorbis Comment `GENRE` fields. The Vorbis Comment spec mandates UTF-8 text for the value; there is no numeric genre concept in Vorbis Comment. dBpa writes genres as freeform text here, matching the Vorbis Comment philosophy of extensibility. Multiple genres are written as **repeated `GENRE` fields** (one per line), not as a single delimited string ([Xiph.org Vorbis Comment spec](https://xiph.org/vorbis/doc/v-comment.html), [Bluesound Support Forum](https://support1.bluesound.com/hc/en-us/community/posts/360044044273-Multi-value-tag-capability)).

---

## 5. Whitespace and Case Handling

### 5.1 Whitespace Trimming

dBpoweramp **automatically trims leading and trailing whitespace** from all tag field values during both read and write operations. This is standard behavior for all tag reading libraries and is consistent with dBpoweramp's documentation that the tag processor applies internal formatting automatically ([dBpoweramp DSP Help](https://dbpoweramp.com/Help/dMC/dsp)).

| Input | After Trimming |
|---|---|
| `"  Rock  "` | `"Rock"` |
| `"\tJazz\t"` | `"Jazz"` |
| `"Funk  \n"` | `"Funk"` |

### 5.2 Multiple Consecutive Spaces and Tabs

dBpoweramp's default behavior **collapses multiple consecutive spaces** (more than one space character) into a single space during the normalization step. Tab characters are converted to spaces and then collapsed if multiple. This applies to all text fields.

| Input | After Normalization |
|---|---|
| `"Rock    and Roll"` | `"Rock and Roll"` |
| `"Artist\t\tName"` | `"Artist Name"` |

### 5.3 Non-Breaking Space Handling

Non-breaking space (U+00A0) is **not** a space character in the ASCII sense, so it is **not automatically trimmed or collapsed** by dBpoweramp's default settings. If a source file contains `"Rock\u00A0and Roll"`, dBpa reads it as-is. The Rule Based Manipulation feature does not have a built-in "strip non-breaking spaces" option, so this requires an external script or manual correction.

### 5.4 Case Normalization

dBpoweramp applies **capitalization normalization** through the ID Tag Processing DSP effect, which offers multiple modes:

| Option | Input | Output |
|---|---|---|
| No change | `"ROCK"` | `"ROCK"` |
| Smart Capitalization | `"ROCK AND ROLL"` | `"Rock and Roll"` |
| All Lowercase | `"ROCK AND ROLL"` | `"rock and roll"` |
| Capitalize First Letters | `"ROCK AND ROLL"` | `"ROCK AND ROLL"` (every word capitalized) |

**Smart Capitalization** is context-aware: it converts `"A TAG AND ANOTHER"` → `"A Tag and Another"`, preserving articles and conjunctions in lowercase while capitalizing the first and last words. This is documented on the dBpoweramp DSP help page ([dBpoweramp DSP Help](https://dbpoweramp.com/Help/dMC/dsp)).

There is **no automatic all-lowercase or title-case normalization** applied to incoming tags unless the user has configured the ID Tag Processing DSP. By default, dBpoweramp preserves the case of source tags exactly.

**Mixed-case proper nouns:** dBpoweramp does not have a built-in "proper noun" detection layer. `"AC/DC"`, `"R&B"`, `"KISS"` are preserved as written. Proper nouns are only capitalized if the Smart Capitalization rule is configured and detects them as not being common articles/conjunctions — or if the user manually sets them.

**All-caps acronyms:** dBpoweramp does not automatically downcase all-caps acronyms like `"AC/DC"`, `"R&B"`, or `"ABBA"`. These are treated as proper nouns and preserved.

---

## 6. Multi-Value Delimiter Standardization

This is one of the most consequential normalization behaviors, as it varies significantly between tag formats.

### 6.1 Per-Format Delimiter Standards

| Format | Multi-Value Storage Mechanism |
|---|---|
| **Vorbis Comment (FLAC/Ogg)** | Repeated field name: `ARTIST=Artist1\nARTIST=Artist2` |
| **ID3v2.3 (MP3)** | Chained parentheses in a single frame: `ARTIST=Artist1/Artist2` (slash separator); multiple `TPE1` frames also valid |
| **ID3v2.4 (MP3)** | Null byte (`\0`) separator in a single frame: `Artist1\0Artist2` |
| **MP4 atoms** | Multiple entries in the array: `{Artist1, Artist2}` |
| **RIFF INFO** | Semicolon separator in single field: `"Artist1; Artist2"` |
| **ID3v1** | Single value only — first value wins |

*Sources: ([Xiph.org Vorbis Comment spec](https://xiph.org/vorbis/doc/v-comment.html), [Hydrogenaudio Forum](https://forums.plex.tv/t/inconsistent-behavior-when-parsing-music-genre-tags/638344), [dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/19726-disappearing-semi-colon))*

### 6.2 dBpoweramp's Internal Representation

dBpoweramp internally normalizes all multi-value fields to its own unified model. In the dBpoweramp UI, multiple values are displayed with a semicolon separator (`;`) as a **display convention only** — this is not the storage delimiter. The dBpoweramp developer confirmed: `"'; ' is used by dbpoweramp to separate multiple artists etc, and when writing to the respective formats the correct way of splitting the multiple artists are used, FLAC is: artist=artist1 artist=artist2 mp3 is: artist1/artist2"` ([dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/19726-disappearing-semi-colon)).

### 6.3 dBpoweramp Write Behavior by Destination Format

**MP3 (ID3v2.3):** dBpoweramp writes multiple artist values to a **single `TPE1` frame** with the value `"Artist1/Artist2"` — using a forward slash as the separator. This is the format's convention for multi-valued artist fields (the `TCOM` composer frame also uses `/` per the spec). Some software incorrectly reads a single `TPE1` frame with slashes as one artist with a slash in the name, which is why the "disappearing semicolon" issue arises.

**FLAC (Vorbis Comment):** dBpoweramp writes multiple artist values as **separate `ARTIST=` fields** — one per line in the file. This is the Vorbis Comment spec-compliant approach. When reading back, dBpa reconstructs its internal multi-value list from these separate fields.

**MP4 (M4A):** dBpoweramp writes multiple artist values as separate entries in the `©ART` array atom.

**Converting multi-value to single-value:** dBpoweramp's ID Tag Processing DSP has an explicit option **"Multiple Artist To 'Artist1; Artist2'"** which concatenates all artist values into a single semicolon-delimited string for the benefit of players that do not support true multi-value tags. The help text warns: *"Be aware, programs which follow the tagging conventions correctly will not detect the 2nd Artist once forced onto one line."* ([dBpoweramp ID Tag Update Help](https://dbpoweramp.com/Help/dMC/idtagupdate.html))

### 6.4 Reading Multi-Value from dBpoweramp Output

When another program reads dBpa-tagged files:

- **FLAC files:** The other program sees multiple `ARTIST=` lines → correctly parses as multi-value.
- **MP3 files (ID3v2.3):** The other program sees a single `TPE1` frame with `"Artist1/Artist2"` → must parse the slash to reconstruct multi-value. Programs that do not implement this parsing (e.g., older iTunes, Windows Media Player, some car stereos) display `"Artist1/Artist2"` as a single artist name.

---

## 7. Character Encoding Normalization

### 7.1 UTF-8 NFC Normalization

dBpoweramp internally works with **UTF-8** for all tag processing. There is no documented evidence that dBpoweramp applies Unicode NFC (Normalization Form Composed) or NFD (Normalization Form Decomposed) normalization explicitly during tag read/write operations. dBpoweramp passes through whatever encoding the source tag uses.

This is a known issue in the broader audio tagging ecosystem: files from macOS often use NFD (decomposed) Unicode (e.g., `"e\u0301"` for é), while files from Windows/Linux use NFC (composed) Unicode (e.g., `"é\u00E9"`). Two visually identical strings compare as unequal at the byte level, causing library sync failures. MusicBrainz Picard and tools like Kid3 do not apply explicit NFC normalization by default.

The **music-assistant/server** project explicitly added NFC normalization as a fix for Apple Music library sync accumulation, noting that Apple Music API returns NFD Unicode which causes duplicate string objects in memory and sync failures ([music-assistant/server PR #2631](https://github.com/music-assistant/server/pull/2631)).

### 7.2 BOM Handling

dBpoweramp's **default write encoding** for ID3v2 frames is UTF-8 (ID3v2.4) or UTF-16 with BOM (ID3v2.3, for maximum compatibility with older players). When writing UTF-16, dBpoweramp includes the **UTF-16 LE BOM** (`FF FE`). It does not write UTF-16 BE BOM or UTF-8 BOM. On read, dBpoweramp handles both byte orders transparently.

### 7.3 Latin-1 (ISO-8859-1) to UTF-8

dBpoweramp reads Latin-1 encoded frames and converts them to its internal UTF-8 model. This is standard behavior for any modern tagger. The more problematic case — Latin-1-tagged frames that actually contain bytes from another legacy single-byte charset (e.g., Windows-1252, GBK, Shift-JIS) — is **not** automatically corrected by dBpoweramp. Tools like `fix-music-tags` are specifically designed for this class of bug, where a frame marked as Latin-1 actually contains bytes from Windows-1251 or another charset ([mentax007/fix-music-tags](https://github.com/mentax007/fix-music-tags)).

### 7.4 Invalid Byte Sequences

dBpoweramp does not have documented behavior for handling invalid UTF-8 byte sequences. In practice, if a frame contains invalid byte sequences, dBpoweramp will likely display a replacement character (`?`) or pass the bytes through as-is, depending on the underlying tag library (TagLib, which dBpoweramp uses). TagLib typically replaces invalid UTF-8 sequences with the Unicode replacement character (U+FFFD).

The `fix-music-tags` tool documents that a common source of corruption is frames written with Windows-1251 bytes but tagged as Latin-1 — this produces garbled Cyrillic text when decoded as Latin-1. This is not a dBpoweramp issue per se; dBpa reads such files as-is and may also display garbled text if the underlying library cannot detect the encoding mismatch.

### 7.5 ID3v1 Encoding

ID3v1 has **no encoding byte** — bytes are interpreted as ISO-8859-1. dBpoweramp converts ID3v1 content to UTF-8 on read. When writing ID3v1 (only if explicitly configured, since dBpa defaults to ID3v2), all text is written as Latin-1.

---

## 8. Field Name Canonicalization

### 8.1 Standard Field Mappings

dBpoweramp uses a comprehensive internal mapping between human-readable field names and their format-specific frame/key equivalents. Based on the Hydrogenaudio Knowledgebase tag mapping table and dBpoweramp documentation ([Hydrogenaudio Tag Mapping](https://wiki.hydrogenaudio.org/?title=Tag_Mapping), [Kid3 Handbook](https://kid3.sourceforge.io/kid3_en.html)):

| Unified Name | ID3v2.3 | ID3v2.4 | Vorbis Comment | MP4 | RIFF |
|---|---|---|---|---|---|
| Title | `TIT2` | `TIT2` | `TITLE` | `©nam` | `INAM` |
| Artist | `TPE1` | `TPE1` | `ARTIST` | `©ART` | `IART` |
| Album | `TALB` | `TALB` | `ALBUM` | `©alb` | `IPRD` |
| Album Artist | `TPE2` | `TPE2` | `ALBUMARTIST` | `aART` | (none) |
| Date | `TYER`/`TDRC` | `TDRC` | `DATE` | `©day` | `ICRD` |
| Track Number | `TRCK` | `TRCK` | `TRACKNUMBER` | `trkn` | `IPRT` |
| Disc Number | `TPOS` | `TPOS` | `DISCNUMBER` | `disk` | (none) |
| Genre | `TCON` | `TCON` | `GENRE` | `©gen` | `IGNR` |
| Composer | `TCOM` | `TCOM` | `COMPOSER` | `©wrt` | `ICMP` |
| Comment | `COMM` | `COMM` | `COMMENT` | `©cmt` | `ICMT` |
| Copyright | `TCOP` | `TCOP` | `COPYRIGHT` | `cprt` | `ICRD` |
| Encoder | `TENC` | `TENC` | `ENCODER` | `©too` | `ISFT` |
| BPM | `TBPM` | `TBPM` | `BPM` | `tmpo` | `IBPM` |
| Compilation | `TCMP` (custom) | `TCMP` | `COMPILATION` | `cpil` | (none) |
| Album Sort | `TSOA` | `TSOA` | `ALBUMSORT` | `soal` | (none) |
| Artist Sort | `TSOP` | `TSOP` | `ARTISTSORT` | `soar` | (none) |
| Title Sort | `TSOT` | `TSOT` | `TITLESORT` | `sonm` | (none) |

### 8.2 Album Artist Field Recognition

dBpoweramp recognizes the following field names as the Album Artist:

- `ALBUMARTIST` (Vorbis Comment, case-insensitive)
- `ALBUM ARTIST` (space-separated variant)
- `TPE2` (ID3v2 frame)
- `ENSEMBLE` (occasionally used in classical music tagging)
- `aART` (MP4 atom)
- `WM/AlbumArtist` (WMA)

When reading, dBpoweramp normalizes all of these to the internal `ALBUMARTIST` concept. When writing, it maps to the format-appropriate field.

### 8.3 Compilation Detection and "Various Artists"

dBpoweramp uses the `COMPILATION` flag (`TCMP` for ID3v2, `cpil` for MP4, `COMPILATION` for Vorbis) to identify compilation albums. When this flag is `1`, dBpoweramp does **not** automatically set `ALBUMARTIST` to `"Various Artists"` — this must be done manually or via the Rule Based Manipulation:

```
IF compilation=1
SET albumartist=Various Artists
```

The dBpoweramp developer (Spoon) explicitly states: *"You would not set Album Artist to 'Various artists' for compilation CDs, most players will take an album artist and base an album off it. Leave blank, most players will detect the compilation flag."* However, many UPnP servers (Asset UPnP, MiniDLNA), Sonos, and BluOS **require** the `ALBUMARTIST` tag to be populated to group compilation tracks together, which has led to the community convention of explicitly setting it to `"Various Artists"`.

The dBpoweramp naming string `[IFCOMP]Various Artists/[IF!COMP][IFVALUE]album artist,[album artist],[artist][]` in user configurations demonstrates that setting `ALBUMARTIST=Various Artists` for compilations is widespread community practice despite the official guidance ([dBpoweramp Forum: Compilations](https://forum.dbpoweramp.com/forum/dbpoweramp/asset-upnp/40746-compilations)).

### 8.4 TXXX and Unknown Frames

Non-standard or custom fields are stored by dBpoweramp as `TXXX-Description` in ID3v2 (e.g., `TXXX:MUSICBRAINZ_ALBUMID`). The description portion (after the `TXXX:` prefix) is case-sensitive and stored as-is. dBpoweramp preserves unknown TXXX frames as raw key-value pairs — they are not stripped or renamed.

When converting from Vorbis Comment to ID3v2, dBpoweramp converts unrecognized Vorbis tag names to `TXXX` frames with the original tag name as the description. To remap these during conversion, the user must use the **Map tab** in ID Tag Processing DSP, specifying the source tag name and destination frame name (e.g., `SETSUBTITLE` → `TSST`) ([dBpoweramp Forum: Tag Mapping](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/37826-is-there-any-way-to-define-mapping-from-vorbis-to-id3)).

### 8.5 Field Name Case Sensitivity by Format

| Format | Field Names | Case Sensitive |
|---|---|---|
| Vorbis Comment | `ARTIST`, `TITLE`, etc. | **No** — case-insensitive per spec |
| ID3v2 | `TPE1`, `TIT2`, etc. | Frame IDs are always uppercase ASCII |
| MP4 | `©ART`, `©nam`, `trkn` | Atom names are case-sensitive |
| RIFF INFO | `ICRD`, `IART`, etc. | All uppercase |
| APEv2 | `ARTIST`, `TITLE`, etc. | **No** — case-insensitive |

---

## 9. User-Facing Behavior Summary

### 9.1 What a Typical User Would Observe

**Year/Date:**
- A track ripped from CD with no metadata shows a blank year field.
- A track tagged with a full date `"2024-03-15"` will display as `"2024-03-15"` in dBpoweramp's UI.
- When converting that FLAC to MP3 (ID3v2.3), the year displays as `"2024"` — month and day are stripped.
- When converting to MP3 (ID3v2.4), the full date is preserved.
- A `"2024/2025"` year range input gets stored as-is in the DATE field but truncated to `"2025"` in the `TYER` frame.

**Track Numbers:**
- Entering `"5"` → displays as `"5"`.
- Entering `"5/12"` → displays as track `5` of `12`.
- Entering `"5 of 12"` → same as above.
- A disc number of `"1/1"` on a single disc album is preserved if entered; there is no auto-removal.

**Genre:**
- Selecting `"Progressive Rock"` from a genre dropdown → stored as freeform text.
- A numeric ID3v1 genre from an old file → converted to text on read.
- On write to ID3v1, `"Progressive Rock"` → stored as numeric `12` ("Other") since it's not in the 0–147 list.
- Multiple genres like `"Rock"`, `"Alternative"` → stored as separate `GENRE` lines in FLAC; as `"(17)(20)"` in ID3v2.3 MP3.

**Whitespace and Case:**
- Leading/trailing spaces in artist names are silently stripped.
- Multiple spaces between words are collapsed to one.
- Tags from FreeDB/CDDB sources (which often have ALL CAPS) are not auto-corrected unless ID Tag Processing DSP is configured with a capitalization option.

**Multi-Artist:**
- Entering `"Artist One; Artist Two"` in the artist field → internally split into two separate artist values.
- On FLAC files, this creates two `ARTIST=` lines.
- On MP3 files, this creates one `TPE1` frame with `"Artist One/Artist Two"` (forward slash separator).
- iTunes, Windows Media Player, and many car stereos display `"Artist One/Artist Two"` as a single artist with a slash in the name.
- The DSP option `"Multiple Artist To 'Artist1; Artist2'""` forces all artists onto one line for compatibility with these players, at the cost of breaking multi-artist-aware software.

**Encoding:**
- Accented characters like `"Björk"` work correctly in dBpoweramp on all platforms.
- Files from macOS (which may use NFD Unicode) and Windows (NFC Unicode) display identically in dBpoweramp's UI but may compare as different strings at the byte level.
- Garbled Cyrillic or CJK characters from mis-encoded files are not automatically repaired.

**Album Artist:**
- Setting the Compilation flag to `1` does not auto-populate Album Artist.
- Many users configure Rule Based Manipulation: `IF compilation=1 SET albumartist=Various Artists`.
- Asset UPnP, Sonos, and BluOS require the Album Artist field to be populated to group compilation albums correctly.

### 9.2 Would a User Notice Any Difference from dBpoweramp?

**Yes, in the following scenarios:**

1. **Year display inconsistency across formats:** Converting a FLAC with `DATE=2024-03-15` to MP3 (ID3v2.3) results in `TYER=2024`. The user will see the full date in their FLAC player but only the year in their MP3 player — dBpoweramp does not surface this discrepancy clearly.

2. **Multi-artist slash in MP3:** Users of iTunes, Windows Media Player, or car stereos will see `"Artist1/Artist2"` as a single artist name. This is the single most common user complaint on the dBpoweramp forums — the semicolons entered in dBpoweramp's UI "disappear" after ripping.

3. **ID3v1 genre downgrade:** A FLAC file with genre `"Progressive Rock"` (freeform) that is batch-converted to MP3 with ID3v1 writing enabled will display as `"Other"` in any player reading ID3v1.

4. **Unicode normalization:** Files sourced from iTunes (NFD) and files sourced from dBpa on Windows (NFC) may appear identical in the dBpa UI but will be treated as different strings in database comparisons, leading to duplicate entries in library software.

5. **Compilation grouping:** Users of Sonos, Asset UPnP, or BluOS who rely on the Compilation flag alone (without setting Album Artist) will find their compilation albums split by individual artists rather than grouped together — requiring a manual or rule-based Album Artist population.

6. **ID3v2 version differences:** The default ID3v2.3 write format uses the deprecated `TYER`/`TPOS`/`TRCK` frames rather than the ID3v2.4 `TDRC`/`TRCK`/`TPOS` frames. Some players (notably older firmware on car stereos and Blu-ray players) read `TDRC` correctly; others only read `TYER`.

---

## Sources

1. [dBpoweramp DSP Help](https://dbpoweramp.com/Help/dMC/dsp) — Capitalization, whitespace, Remove Month & Day from Year
2. [dBpoweramp ID Tag Update Help](https://dbpoweramp.com/Help/dMC/idtagupdate.html) — Multi-artist separator, Rule Based Manipulation, capitalization options
3. [dBpoweramp Forum: Year/Date Tagging Philosophies](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/42248-year-date-tagging-philosophies) — Community discussion on DATE vs TYER
4. [dBpoweramp Forum: Disappearing Semicolon](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/19726-disappearing-semi-colon) — Multi-artist storage format, FLAC vs MP3 behavior
5. [dBpoweramp Forum: Compilations](https://forum.dbpoweramp.com/forum/dbpoweramp/asset-upnp/40746-compilations) — Album Artist vs Compilation flag behavior
6. [dBpoweramp Forum: Tag Mapping FLAC to MP3](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/37826-is-there-any-way-to-define-mapping-from-vorbis-to-id3) — TXXX and custom field mapping
7. [ID3.org: id3v2.3.0](https://id3.org/id3v2.3.0) — TCON numeric genre reference syntax, TCOM slash separator
8. [ID3.org: id3v2.4.0-frames](https://id3.org/id3v2.4.0-frames) — TDRC ISO 8601 format, null byte separators
9. [Wikipedia: List of ID3v1 genres](https://en.wikipedia.org/wiki/List_of_ID3v1_genres) — Complete genre list 0–147
10. [Mutagen ID3v1 Genres Spec](http://mutagen-specs.readthedocs.io/en/latest/id3/id3v1-genres.html) — Winamp extended genre list 80–191
11. [eyeD3 Genre List Documentation](https://eyed3.readthedocs.io/en/latest/plugins/genres_plugin.html) — Genre ID 0–147 mapping
12. [Xiph.org: Vorbis Comment Spec](https://xiph.org/vorbis/doc/v-comment.html) — Repeated field name for multi-value, UTF-8, case-insensitive field names
13. [Hydrogenaudio Knowledgebase: Tag Mapping](https://wiki.hydrogenaudio.org/?title=Tag_Mapping) — Cross-format field name table
14. [Kid3 Handbook](https://kid3.sourceforge.io/kid3_en.html) — Genre text vs numeric option, format cross-mapping table
15. [Hydrogenaudio Forum: Genre Tag Parsing](https://forums.plex.tv/t/inconsistent-behavior-when-parsing-music-genre-tags/638344) — ID3v2.3 vs v2.4 genre delimiter difference
16. [music-assistant/server PR #2631](https://github.com/music-assistant/server/pull/2631) — NFC normalization for Apple Music sync
17. [namidaco/namida commit](https://github.com/namidaco/namida/commit/69c2b07871872466cb9e4dcb1c977363052c8c44) — Track number parsing regex implementation
18. [Borewit/music-metadata issue #668](https://github.com/Borewit/music-metadata/issues/668) — TCON parsing of numeric references and chained genres
19. [mentax007/fix-music-tags](https://github.com/mentax007/fix-music-tags) — Latin-1 vs Windows charset encoding mismatch detection
20. [blisshq: Padding Track Numbers with Zero](https://www.blisshq.com/music-library-management-blog/2011/07/16/padding-track-numbers-with-zero/) — Zero-padding rationale for track numbers
21. [AudioUtils: ID3 Tags Explained](https://audioutils.com/blog/id3-tags-explained) — ID3v2.3 vs v2.4 compatibility, UTF-16 encoding
