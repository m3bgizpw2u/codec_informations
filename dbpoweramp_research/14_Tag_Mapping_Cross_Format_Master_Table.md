# Cross-Format Tag Mapping Master Table

**DBpoweramp Reference Number:** Research Doc 14
**Purpose:** Complete field-to-field mapping for every standard audio metadata tag, across all major tagging systems, for use in DBpoweramp-equivalent conversion pipelines.
**Canonical Sources:** MusicBrainz Picard Tag Mapping v3.0; Mp3tag Tag Field Mappings; Hydrogenaudio ReplayGain 2.0 Specification; ID3.org ID3v2.3.0 / ID3v2.4.0 specifications.
**Version:** 1.0 | 2026-06-04

---

## Section 1: Overview & Purpose

Every audio format stores metadata using a different tagging system. A single logical field — for example, the album artist — may be stored as `TPE2` in an MP3 (ID3v2), `ALBUMARTIST` in a FLAC (Vorbis Comment), `aART` in an M4A (MP4/iTunes atom), and `WM/AlbumArtist` in a WMA (ASF attribute). When converting between formats, these must be mapped correctly or information is silently lost.

This document is the **single authoritative reference** for every standard tag field. It maps each canonical field to its exact storage key in every major tagging system. It also documents DBpoweramp's read/write behavior, multi-value handling, character encoding, length limits, and format-specific quirks.

### Scope

This document covers the nine major tagging systems in active use:

| System | Used In | Standard Body |
|---|---|---|
| **ID3v2.3** | MP3, AIFF | ID3.org (de facto) |
| **ID3v2.4** | MP3, AIFF | ID3.org (de facto) |
| **APEv2** | Musepack, OptimFROG, WavPack, APE | Morphcast / HydrogenAudio (de facto) |
| **Vorbis Comment** | FLAC, Ogg Vorbis, Ogg Opus, Ogg FLAC, Speex | Xiph.org |
| **MP4/iTunes Atom** | AAC, ALAC, MP4 (M4A, M4B, M4P) | ISO/IEC 14496-12 / Apple specification |
| **ASF/WMA Attribute** | WMA, ASF | Microsoft |
| **RIFF INFO** | WAV (PCM) | Microsoft / IBM |
| **AIFF Standard Chunks** | AIFF | Apple / AIFF-1.0 spec |
| **BWF Extension** | Broadcast WAV | EBU Tech 3285 |

### How DBpoweramp Handles Tags During Conversion

DBpoweramp performs a **canonical read → canonical write** cycle:

1. **Read phase:** Extract all tag fields from the source file using the source format's native system.
2. **Canonicalize:** Normalize field names to a canonical internal representation (Picard-style canonical names are used in this document).
3. **Write phase:** Map each canonical field to the target format's native key and write it.

Fields that cannot be represented in a target format are **dropped**, not converted to a generic TXXX or custom field (unless a known mapping exists). This document notes which fields can and cannot be round-tripped.

> **Note for pipeline builders:** Always implement the canonical round-trip model. Do not attempt to store every field in every format — some fields simply have no equivalent. For example, RIFF INFO has no field for MusicBrainz Track ID; that field must be dropped for WAV output.

---

## Section 2: How to Read This Table

### Column Definitions

| Column | Description |
|---|---|
| **Canonical Field** | The normalized field name used as the internal representation. Based on MusicBrainz Picard canonical names. |
| **ID3v2.3 Frame** | The 4-character frame ID for ID3v2.3. `TXXX:description` means a TXXX user-defined text frame with that description string. `UFID:owner` means a UFID frame with that owner identifier. |
| **ID3v2.4 Frame** | Same as above for ID3v2.4. Differs only for date frames (`TYER`/`TORY`/`TRK` in v2.3 → `TDRC`/`TDOR`/`TRCK` in v2.4) and the people list (`IPLS` → `TIPL`/`TMCL`). |
| **APEv2 Key** | The exact key name for APEv2 tags. APEv2 keys are case-sensitive but convention is Title Case. |
| **Vorbis Comment** | The exact field name for Vorbis Comment tags. Vorbis Comment keys are case-insensitive but convention is UPPERCASE. Multiple variants (TRACKTOTAL / TOTALTRACKS) are listed with `/`. |
| **MP4 Atom** | The 4-byte atom identifier inside the `ilst` box. Atoms prefixed with `----` are freeform atoms in the `----:com.apple.iTunes:` namespace. Binary atoms (covr) have no string key. |
| **ASF/WMA Attribute** | The attribute name used in the ASF Extended Content Description object. Case-sensitive. |
| **RIFF INFO** | The chunk ID in the `INFO` LIST chunk. 4 ASCII characters. |
| **AIFF Chunks** | Standard AIFF metadata chunk IDs. Most metadata in AIFF uses the ID3 `ID3 ` chunk (note trailing space) which holds ID3v2 frames. Native AIFF chunks include `NAME`, `AUTH`, `ANNO`, `MIDI`, etc. |

### Cell Abbreviations

| Symbol | Meaning |
|---|---|
| *(empty)* | Field not natively supported in this format — will be dropped during conversion. |
| **n/a** | Format does not support this field type at all. |
| **binary** | Field stores binary data, not text. |
| **multi** | Field supports multiple values. |
| **single** | Field supports only a single value (first occurrence wins). |
| **v2.3 only** | Frame exists only in ID3v2.3; no v2.4 equivalent. |
| **v2.4 only** | Frame exists only in ID3v2.4; v2.3 has no equivalent. |
| **deprecated** | Frame/key exists but is deprecated or non-standard. |

### DBpoweramp R/W Indicators

Throughout the document, `(R)` means DBpoweramp reads this field from source files, `(W)` means DBpoweramp writes it to output files. `(R+W)` means both. Absence of a marker means the behavior has not been independently verified.

### Multi-Value Handling

Some formats (Vorbis Comment, APEv2) natively support multiple values per field. Others (ID3v2, MP4, ASF) store one value per field. When mapping multi-value fields to single-value formats, DBpoweramp joins values with ` / ` (space-slash-space) as the delimiter. When mapping back, it splits on ` / ` to reconstruct multiple values.

### Character Encoding

| Format | Encoding |
|---|---|
| ID3v2.3 | Latin-1 (ISO-8859-1) for text frames; UTF-16 with BOM for international characters |
| ID3v2.4 | UTF-8 for text frames |
| Vorbis Comment | UTF-8 |
| APEv2 | UTF-8 |
| MP4 | UTF-8 (stored as UTF-8 bytes in the atom) |
| ASF | UTF-16-LE |
| RIFF INFO | ASCII |
| AIFF ID3 chunk | Same as ID3v2 (inherits the ID3 frame encoding rules) |

### Max Field Lengths

| Format | Approximate Max |
|---|---|
| ID3v2 frame value | ~2 GB (size field is 4 bytes) but most players truncate at 255–1024 chars |
| Vorbis Comment | No hard limit; players typically handle up to 4096 chars |
| APEv2 item value | 2 GB (size is variable-length integer) |
| MP4 atom value | ~2 GB |
| ASF attribute | 64 KB |
| RIFF INFO | 256 bytes per chunk |

---

## Section 3: Core Identification Fields

### 3.1 Title / Track Title

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO | AIFF |
|---|---|---|---|---|---|---|---|---|
| **TITLE** | `TIT2` | `TIT2` | `Title` | `TITLE` | `©nam` | `Title` | `INAM` | `NAME` or `ID3`→`TIT2` |

- **DBpoweramp (R+W):** Full read/write. Writes as single-value.
- **Notes:** ID3v2 allows only one TIT2 per tag. AIFF's native `NAME` chunk holds the sound name; most tools use the `ID3 ` chunk for full ID3v2 support instead.

### 3.2 Artist

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO | AIFF |
|---|---|---|---|---|---|---|---|---|
| **ARTIST** | `TPE1` | `TPE1` | `Artist` | `ARTIST` | `©ART` | `Author` | `IART` | `AUTH` or `ID3`→`TPE1` |

- **DBpoweramp (R+W):** Full read/write. Multi-artist values joined with ` / ` when converting to ID3v2.
- **Notes:** ASF uses `Author` (from the Content Description object, not the Extended Content Description). AIFF `AUTH` chunk stores the author/artist name.

### 3.3 Album Artist

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO | AIFF |
|---|---|---|---|---|---|---|---|---|
| **ALBUMARTIST** | `TPE2` | `TPE2` | `Album Artist` | `ALBUMARTIST` | `aART` | `WM/AlbumArtist` | — | `ID3`→`TPE2` |

- **DBpoweramp (R+W):** Full read/write. Critical for compilation detection — DBpoweramp uses this field (or the Compilation flag) to set the `ALBUMARTIST = "Various Artists"` fallback.
- **Notes:** When `ALBUMARTIST` is absent and the file is a compilation, DBpoweramp treats the artist as `Various Artists` for grouping purposes. Write this field explicitly when converting compilation albums.

### 3.4 Album

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO | AIFF |
|---|---|---|---|---|---|---|---|---|
| **ALBUM** | `TALB` | `TALB` | `Album` | `ALBUM` | `©alb` | `WM/AlbumTitle` | `IPRD` | `ID3`→`TALB` |

- **DBpoweramp (R+W):** Full read/write.
- **Notes:** RIFF INFO `IPRD` is labeled "product" but widely used for album name.

### 3.5 Composer

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO | AIFF |
|---|---|---|---|---|---|---|---|---|
| **COMPOSER** | `TCOM` | `TCOM` | `Composer` | `COMPOSER` | `©wrt` | `WM/Composer` | `IMUS` | `ID3`→`TCOM` |

- **DBpoweramp (R+W):** Full read/write.
- **Notes:** MP4 uses `©wrt` (writer/composer). RIFF INFO uses `IMUS` (musician). Multiple composers in Vorbis/ID3: separate `COMPOSER` fields or ` / ` join.

### 3.6 Conductor / Performer

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO | AIFF |
|---|---|---|---|---|---|---|---|---|
| **CONDUCTOR** | `TPE3` | `TPE3` | `Conductor` | `CONDUCTOR` | `©con` | `WM/Conductor` | — | `ID3`→`TPE3` |
| **PERFORMER**¹ | `IPLS:conductor` (v2.3) / `TMCL:conductor` (v2.4) | `TIPL:conductor` | `Performer=Artist` | `PERFORMER=Artist` | — | — | — | `ID3`→`TIPL/TMCL` |

1. Performer uses a role-qualified format. The performer instrument/person mapping is stored as `PERFORMER=Piano:Claude Debussy` in Vorbis, `TMCL:instrument=Person` in ID3v2.4. See Section 9 for the full TMCL/TIPL mapping.

- **DBpoweramp (R+W):** `TPE3` / `CONDUCTOR` read/write. Role-qualified performers may be dropped or stored in custom fields.
- **Notes:** `TPE3` in ID3v2 is specifically "Interpreted, remixed, or otherwise modified by." Many taggers use it for conductor. For classical music with specific instrument performers, use TMCL/TIPL.

### 3.7 Arranger

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO | AIFF |
|---|---|---|---|---|---|---|---|---|
| **ARRANGER** | `IPLS:arranger` | `TIPL:arranger` | `Arranger` | `ARRANGER` | — | — | — | `ID3`→`TIPL` |

- **DBpoweramp:** Limited. Not stored in MP4/ASF/RIFF.
- **Notes:** ID3v2.3 uses `IPLS` (involved people list) with `arranger` as role. ID3v2.4 splits this into `TIPL` (maintained people) and `TMCL` (musician credits). In practice, many tools flatten these to simple text frames.

### 3.8 Work (Musical Work Title)

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO | AIFF |
|---|---|---|---|---|---|---|---|---|
| **WORK** | `TXXX:WORK` + `TIT1`¹ | `TXXX:WORK` + `TIT1`¹ | `WORK` | `WORK` | `©wrk` | `WM/Work` | — | `ID3`→`TXXX:WORK` |

1. With "Save iTunes compatible grouping and work" enabled in Picard ≥2.1, both `TXXX:WORK` and `TIT1` are written to ID3v2. The `TIT1` also serves as the Content Group Description.

- **DBpoweramp (R):** Reads `TXXX:WORK` or `TIT1`. May not write.
- **Notes:** This field identifies the larger musical work a track belongs to (e.g., a symphony). Distinct from `TITLE` (the track/section name) and `MOVEMENTNAME` (the movement name).

### 3.9 Movement Name

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO | AIFF |
|---|---|---|---|---|---|---|---|---|
| **MOVEMENTNAME** | `MVNM` | `MVNM` | `MOVEMENTNAME` | `MOVEMENTNAME` | `©mvn` | — | — | `ID3`→`MVNM` |

- **DBpoweramp (R):** Read but not natively written to MP3 in older versions.
- **Notes:** ID3v2.4 defines `MVNM` (Movement/Work name). Picard maps this to `©mvn` in MP4 (iTunes 8+). No ASF or RIFF equivalent.

### 3.10 Subtitle / Disc Subtitle

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO | AIFF |
|---|---|---|---|---|---|---|---|---|
| **SUBTITLE** | `TIT3` | `TIT3` | `Subtitle` | `SUBTITLE` | — | `WM/SubTitle` | — | `ID3`→`TIT3` |
| **SETSUBTITLE** (Disc Subtitle) | — | `TSST` (v2.4 only) | `DiscSubtitle` | `DISCSUBTITLE` | — | `WM/SetSubTitle` | — | `ID3`→`TSST` |

- **DBpoweramp:** `TIT3` is full read/write. `TSST` only for ID3v2.4 output; DBpoweramp will drop disc subtitle when converting to ID3v2.3 MP3.
- **Notes:** `SETSUBTITLE` identifies the name of a disc within a multi-disc set (e.g., "Live in Tokyo — Night 1"). Not supported in ID3v2.3.

---

## Section 4: Numeric Fields

### 4.1 Track Number & Total Tracks

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO | AIFF |
|---|---|---|---|---|---|---|---|---|
| **TRACKNUMBER** | `TRCK` (format: `N` or `N/O`) | `TRCK` | `Track` | `TRACKNUMBER` | `trkn` (packed uint16 pair) | `WM/TrackNumber` | `ITRK` | `ID3`→`TRCK` |
| **TOTALTRACKS** | `TRCK` (format: `N/O`) | `TRCK` | `Track` | `TRACKTOTAL` **or** `TOTALTRACKS` | `trkn` (total in high word) | — | — | `ID3`→`TRCK` |

- **DBpoweramp (R+W):** Full read/write. Critical parsing behavior:
  - **ID3 → Vorbis:** `TRCK="5/12"` → `TRACKNUMBER=5` + `TOTALTRACKS=12`. DBpoweramp splits on `/`.
  - **Vorbis → ID3:** `TRACKNUMBER=5` + `TOTALTRACKS=12` → `TRCK="5/12"`. DBpoweramp joins with `/`.
  - **MP4 → anything:** `trkn` atom stores track and total as a packed big-endian uint16 pair: `((track << 16) | total)`. Read both parts.
  - **APE → anything:** `Track="5/12"` → split on `/`.
  - **Never write** `"5/12"` as a single Vorbis `TRACKNUMBER` value — only ID3 uses the slash format.
- **Notes:** MP4 `trkn` has no text equivalent — always packed binary. ASF `WM/TrackNumber` is a text string, not numeric. `TOTALTRACKS` is not stored in ASF or RIFF — it must be dropped.

### 4.2 Disc Number & Total Discs

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO | AIFF |
|---|---|---|---|---|---|---|---|---|
| **DISCNUMBER** | `TPOS` (format: `N` or `N/O`) | `TPOS` | `Disc` | `DISCNUMBER` | `disk` (packed uint16 pair) | `WM/PartOfSet` | — | `ID3`→`TPOS` |
| **TOTALDISCS** | `TPOS` (format: `N/O`) | `TPOS` | `Disc` | `DISCTOTAL` **or** `TOTALDISCS` | `disk` (total in high word) | `WM/PartOfSet` | — | `ID3`→`TPOS` |

- **DBpoweramp (R+W):** Full read/write. Same slash-join/split logic as track numbers.
- **Notes:** ASF `WM/PartOfSet` uses the format `N` or `N/O` (e.g., `"2"` or `"2/3"`). This is the canonical ASF field for disc information. RIFF has no disc field.

### 4.3 Year / Date Fields

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO | AIFF |
|---|---|---|---|---|---|---|---|---|
| **YEAR** | `TYER` (4-digit year) | `TDRC` (full timestamp) | `Year` | `DATE` | `©day` | `WM/Year` | `ICRD` | `ID3`→`TYER`/`TDRC` |
| **DATE** (Release Date) | `TYER`+`TDAT` | `TDRC` (YYYY-MM-DD) | `Year` or `Date` | `DATE` | `©day` | `WM/Year` | `ICRD` | `ID3`→`TDRC` |
| **ORIGINALYEAR** | `TORY` (4-digit) | `TDOR` (full timestamp) | `ORIGINALYEAR` | `ORIGINALYEAR` | — | `WM/OriginalReleaseYear` | — | `ID3`→`TDOR` |
| **RELEASETIME** | — | `TDRL` | — | — | — | `WM/CommercialReleaseDate` | — | `ID3`→`TDRL` |
| **TAGGINGTIME** | — | `TDTG` | — | — | — | — | — | `ID3`→`TDTG` |
| **ENCODINGTIME** | — | `TDEN` | — | — | — | `WM/EncodingTime` | — | `ID3`→`TDEN` |

- **DBpoweramp (R+W):** Year is fully supported. Other date fields read/write depends on version.
- **Critical parsing notes:**
  - **ID3v2.3 date storage:** `TYER` (year, 4 digits), `TDAT` (date as DDMM), `TIME` (time as HHMM). These are separate frames. `TYER` alone is the most compatible.
  - **ID3v2.4 date storage:** `TDRC` uses a timestamp format: `YYYY`, `YYYY-MM`, or `YYYY-MM-DD`. Month and day are optional.
  - **DBpoweramp default for MP3 output:** ID3v2.3, writing `TYER` with the year (or full date from TDRC). It does not write `TDRC` in v2.3 mode.
  - **Best practice:** Write both `TYER` and `TDRC` when converting to MP3 — this maximizes compatibility (old players read TYER, new players read TDRC).
  - **MP4 `©day`:** Accepts `YYYY`, `YYYY-MM`, or `YYYY-MM-DD` as text. iTunes writes ISO 8601 with time (`2024-03-15T00:00:00Z`).
  - **Vorbis `DATE`:** Should be `YYYY-MM-DD` per convention but many tools write just `YYYY`.

### 4.4 BPM (Beats Per Minute)

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO | AIFF |
|---|---|---|---|---|---|---|---|---|
| **BPM** | `TBPM` | `TBPM` | `BPM` | `BPM` | `tmpo` (uint16 integer) | `WM/BeatsPerMinute` | — | `ID3`→`TBPM` |

- **DBpoweramp (R+W):** Full read/write.
- **Notes:** ID3 `TBPM` can store either a numeric BPM or a text value (e.g., `"120"`). MP4 `tmpo` stores an integer BPM value (0–65535). Converting from `tmpo=120` to Vorbis: write `BPM=120`. Converting from `BPM="120"` to MP4: parse integer and write to `tmpo`.

### 4.5 Compilation Flag

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO | AIFF |
|---|---|---|---|---|---|---|---|---|
| **COMPILATION** | `TCMP` (custom) | `TCMP` | `Compilation` | `COMPILATION` | `cpil` (uint8: 0 or 1) | `WM/IsCompilation` | — | `ID3`→`TCMP` |

- **DBpoweramp (R+W):** Full read/write.
- **Notes:** `TCMP` is a non-standard but widely-supported custom ID3v2 frame used by iTunes. It uses values `0` (not a compilation) or `1` (compilation). MP4 `cpil` is a native boolean atom. ASF `WM/IsCompilation` is a text boolean. DBpoweramp uses this flag to infer `ALBUMARTIST = "Various Artists"` when the album artist is absent.

---

## Section 5: Classification Fields

### 5.1 Genre

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO | AIFF |
|---|---|---|---|---|---|---|---|---|
| **GENRE** | `TCON` | `TCON` | `Genre` | `GENRE` | `©gen` or `gnre`¹ | `WM/Genre` | `IGNR` | `ID3`→`TCON` |

1. MP4 `gnre` is a numeric genre (ID3v1 genre number); `©gen` is freeform text. Prefer `©gen`.

- **DBpoweramp (R+W):** Full read/write.
- **Critical parsing rule:** `TCON` in ID3v2 can contain:
  - A genre number in parentheses: `(17)` → "Rock" (from ID3v1 genre table)
  - A freeform string: `"Ambient"`
  - Both: `(17)Ambient` (legacy/redundant)
  - Special codes: `(RX)` = Remix/Cover, `(CR)` = Remix
- **Read strategy:** Strip the `(N)` wrapper. If a numeric ID remains, look it up in the ID3v1 genre table. Store the resolved string.
- **Write strategy:** Write freeform strings only. Do not write `(N)` prefixes. The `(N)` format is a legacy of ID3v1 and should not be generated for new files.
- **ID3v1 Genre Table (first 20 categories):**
  ```
  0=Blues, 1=Classic Rock, 2=Country, 3=Dance, 4=Disco,
  5=Funk, 6=Grunge, 7=Hip-Hop, 8=Jazz, 9=Metal,
  10=New Age, 11=Oldies, 12=Other, 13=Pop, 14=R&B,
  15=Rap, 16=Reggae, 17=Rock, 18=Techno, 19=Industrial,
  20=Alternative, 21=Ska, 22=Death Metal, 23=Pranks, 24=Soundtrack,
  25=Euro-Techno, 26=Ambient, 27=Trip-Hop, 28=Vocal, 29=Jazz+Funk,
  30=Fusion, 31=Trance, 32=Classical, 33=Instrumental, 34=Acid,
  35=House, 36=Game, 37=Sound Clip, 38=Gospel, 39=Noise,
  40=Alternative Rock, 41=Bass, 42=Soul, 43=Punk, 44=Space,
  45=Meditative, 46=Instrumental Pop, 47=Instrumental Rock, 48=Ethnic,
  49=Gothic, 50=Darkwave, 51=Techno-Industrial, 52=Electronic,
  53=Pop-Folk, 54=Eurodance, 55=Dream, 56=Southern Rock, 57=Comedy,
  58=Cult, 59=Gangsta, 60=Top 40, 61=Christian Rap, 62=Pop/Funk,
  63=Jungle, 64=Native American, 65=Cabaret, 66=New Wave, 67=Psychedelic,
  68=Rave, 69=Showtunes, 70=Trailer, 71=Lo-Fi, 72=Tribal, 73=Acid Punk,
  74=Acid Jazz, 75=Polka, 76=Retro, 77=Musical, 78=Rock & Roll,
  79=Hard Rock
  ```
  (Full table has 80 more categories up to index 191; 192 = "Reflex" and is the last defined.)

### 5.2 Mood

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO | AIFF |
|---|---|---|---|---|---|---|---|---|
| **MOOD** | — | `TMOO` | `Mood` | `MOOD` | — | `WM/Mood` | — | `ID3`→`TMOO` |

- **DBpoweramp (R):** Read. ID3v2.3 has no native mood field — DBpoweramp cannot write to ID3v2.3 MP3 without losing this field.
- **Notes:** `TMOO` was introduced in ID3v2.4 only. When writing to ID3v2.3, this field must be dropped. Picard can store it in a TXXX frame (`TXXX:MOOD`) for ID3v2.3 compatibility, but this is non-standard. MP4 has no native mood atom — use `----:com.apple.iTunes:MOOD` freeform.

### 5.3 Initial Key (Musical Key)

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO | AIFF |
|---|---|---|---|---|---|---|---|---|
| **INITIALKEY** | `TKEY` | `TKEY` | `KEY` | `KEY` | — | `WM/InitialKey` | — | `ID3`→`TKEY` |

- **DBpoweramp (R+W):** Full read/write for ID3. No MP4/RIFF equivalent.
- **Notes:** Standard values use the format `"C Major"`, `"A minor"`, `"Db"`, etc. Also accepts shorthand like `"C"`, `"Am"`. Value is freeform text; no fixed enumeration.

### 5.4 Media Type

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO | AIFF |
|---|---|---|---|---|---|---|---|---|
| **MEDIA** | `TMED` | `TMED` | `Media` | `MEDIA` | — | `WM/Media` | `IMED` | `ID3`→`TMED` |

- **DBpoweramp (R+W):** Full read/write.
- **Notes:** Freeform text field. Common values: `"CD"`, `"Vinyl"`, `"Digital Audio"`, `"cassette"`, `"digital"`.

### 5.5 Grouping

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO | AIFF |
|---|---|---|---|---|---|---|---|---|
| **GROUPING** | `TIT1` (content group) | `TIT1` | `Grouping` | `GROUPING` | `©grp` | `WM/ContentGroupDescription` | — | `ID3`→`TIT1` |

- **DBpoweramp (R+W):** Full read/write.
- **Notes:** `TIT1` (Content Group Description) and `GROUPING` serve similar purposes but have different semantics. `GROUPING` typically refers to a musical grouping (e.g., "String Quartet No. 1"), while `TIT1` refers to a content grouping (e.g., "Soundtrack"). Some tools conflate them.

### 5.6 Language

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO | AIFF |
|---|---|---|---|---|---|---|---|---|
| **LANGUAGE** | `TLAN` | `TLAN` | `Language` | `LANGUAGE` | — | `WM/Language` | `ILNG` | `ID3`→`TLAN` |

- **DBpoweramp (R+W):** Full read/write.
- **Notes:** Uses ISO 639-2 language codes (3 letters): `"ENG"`, `"SPA"`, `"JPN"`, `"UND"` (undefined).

---

## Section 6: Identifier Fields

### 6.1 ISRC (International Standard Recording Code)

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO | AIFF |
|---|---|---|---|---|---|---|---|---|
| **ISRC** | `TSRC` | `TSRC` | `ISRC` | `ISRC` | `tsrc` | `WM/ISRC` | — | `ID3`→`TSRC` |

- **DBpoweramp (R+W):** Full read/write.
- **Notes:** Format: `CCXXXYYNNNNN`. Example: `"USRC12345678"`. MP4 uses `tsrc` (freeform text atom). No RIFF equivalent.

### 6.2 Catalog Number

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO | AIFF |
|---|---|---|---|---|---|---|---|---|
| **CATALOGNUMBER** | `TXXX:CATALOGNUMBER` | `TXXX:CATALOGNUMBER` | `CatalogNumber` | `CATALOGNUMBER` | `----:com.apple.iTunes:CATALOGNUMBER` | `WM/CatalogNo` | — | `ID3`→`TXXX:CATALOGNUMBER` |

- **DBpoweramp (R+W):** Full read/write for TXXX. Not stored natively in MP4/ASF without freeform atoms.
- **Notes:** Freeform text. Multiple catalog numbers from different labels should be stored as separate `CATALOGNUMBER` values (Vorbis/APE multi-value) or `TXXX:CATALOGNUMBER` frames.

### 6.3 Barcode / UPC / EAN

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO | AIFF |
|---|---|---|---|---|---|---|---|---|
| **BARCODE** | `TXXX:BARCODE` | `TXXX:BARCODE` | `Barcode` | `BARCODE` | `----:com.apple.iTunes:BARCODE` | `WM/Barcode` | — | `ID3`→`TXXX:BARCODE` |
| **UPC** (EAN/UPC) | `TXXX:UPC` | `TXXX:UPC` | `UPC` | `UPC` | `----:com.apple.iTunes:UPC` | `WM/Barcode` | — | `ID3`→`TXXX:UPC` |

- **DBpoweramp (R+W):** Reads from TXXX/UPC. Writes as TXXX in MP3.
- **Notes:** `BARCODE` is the canonical tag name for any product identifier (UPC, EAN). Some tools write `UPC` specifically. DBpoweramp reads both. ASF uses a single `WM/Barcode` field for both.

### 6.4 Label Code (LABEL)

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO | AIFF |
|---|---|---|---|---|---|---|---|---|
| **LABEL** | `TPUB` | `TPUB` | `Label` | `LABEL` | `----:com.apple.iTunes:LABEL` | `WM/Publisher` | — | `ID3`→`TPUB` |

- **DBpoweramp (R+W):** Full read/write. Note that `TPUB` (Publisher) and `LABEL` share a field in some systems.
- **Notes:** LABEL specifically refers to the record label code (e.g., "LC-00123"). TPUB is the publisher name, which may or may not be the same as the label.

### 6.5 MusicBrainz Identifier Fields

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO |
|---|---|---|---|---|---|---|---|
| `MUSICBRAINZ_TRACKID` | `UFID:http://musicbrainz.org` | `UFID:http://musicbrainz.org` | `MUSICBRAINZ_TRACKID` | `MUSICBRAINZ_TRACKID` | `----:com.apple.iTunes:MusicBrainz Track Id` | `MusicBrainz/Track Id` | — |
| `MUSICBRAINZ_ALBUMID` | `TXXX:MusicBrainz Album Id` | `TXXX:MusicBrainz Album Id` | `MUSICBRAINZ_ALBUMID` | `MUSICBRAINZ_ALBUMID` | `----:com.apple.iTunes:MusicBrainz Album Id` | `MusicBrainz/Album Id` | — |
| `MUSICBRAINZ_ARTISTID` | `TXXX:MusicBrainz Artist Id` | `TXXX:MusicBrainz Artist Id` | `MUSICBRAINZ_ARTISTID` | `MUSICBRAINZ_ARTISTID` | `----:com.apple.iTunes:MusicBrainz Artist Id` | `MusicBrainz/Artist Id` | — |
| `MUSICBRAINZ_ALBUMARTISTID` | `TXXX:MusicBrainz Album Artist Id` | `TXXX:MusicBrainz Album Artist Id` | `MUSICBRAINZ_ALBUMARTISTID` | `MUSICBRAINZ_ALBUMARTISTID` | `----:com.apple.iTunes:MusicBrainz Album Artist Id` | `MusicBrainz/Album Artist Id` | — |
| `MUSICBRAINZ_RELEASEGROUPID` | `TXXX:MusicBrainz Release Group Id` | `TXXX:MusicBrainz Release Group Id` | `MUSICBRAINZ_RELEASEGROUPID` | `MUSICBRAINZ_RELEASEGROUPID` | `----:com.apple.iTunes:MusicBrainz Release Group Id` | `MusicBrainz/Release Group Id` | — |
| `MUSICBRAINZ_RELEASETRACKID` | `TXXX:MusicBrainz Release Track Id` | `TXXX:MusicBrainz Release Track Id` | `MUSICBRAINZ_RELEASETRACKID` | `MUSICBRAINZ_RELEASETRACKID` | `----:com.apple.iTunes:MusicBrainz Release Track Id` | `MusicBrainz/Release Track Id` | — |
| `MUSICBRAINZ_DISCID` | `TXXX:MusicBrainz Disc Id` | `TXXX:MusicBrainz Disc Id` | `MUSICBRAINZ_DISCID` | `MUSICBRAINZ_DISCID` | `----:com.apple.iTunes:MusicBrainz Disc Id` | `MusicBrainz/Disc Id` | — |
| `MUSICBRAINZ_WORKID` | `TXXX:MusicBrainz Work Id` | `TXXX:MusicBrainz Work Id` | `MUSICBRAINZ_WORKID` | `MUSICBRAINZ_WORKID` | `----:com.apple.iTunes:MusicBrainz Work Id` | `MusicBrainz/Work Id` | — |
| `MUSICBRAINZ_COMPOSERID` | `TXXX:MusicBrainz Composer Id` | `TXXX:MusicBrainz Composer Id` | `MUSICBRAINZ_COMPOSERID` | `MUSICBRAINZ_COMPOSERID` | `----:com.apple.iTunes:MusicBrainz Composer Id` | `MusicBrainz/Composer Id` | — |
| `MUSICBRAINZ_ORIGINALARTISTID` | `TXXX:MusicBrainz Original Artist Id` | `TXXX:MusicBrainz Original Artist Id` | — | `MUSICBRAINZ_ORIGINALARTISTID` | `----:com.apple.iTunes:MusicBrainz Original Artist Id` | `MusicBrainz/Original Artist Id` | — |
| `MUSICBRAINZ_ORIGINALALBUMID` | `TXXX:MusicBrainz Original Album Id` | `TXXX:MusicBrainz Original Album Id` | — | `MUSICBRAINZ_ORIGINALALBUMID` | `----:com.apple.iTunes:MusicBrainz Original Album Id` | `MusicBrainz/Original Album Id` | — |
| `MUSICBRAINZ_TRMID` (deprecated) | `TXXX:MusicBrainz TRM Id` | `TXXX:MusicBrainz TRM Id` | `MUSICBRAINZ_TRMID` | `MUSICBRAINZ_TRMID` | `----:com.apple.iTunes:MusicBrainz TRM Id` | `MusicBrainz/TRM Id` | — |
| `MUSICBRAINZ_ALBUMRELEASECOUNTRY` | `TXXX:MusicBrainz Album Release Country` | `TXXX:MusicBrainz Album Release Country` | `MUSICBRAINZ_ALBUMRELEASECOUNTRY` | `RELEASECOUNTRY` | `----:com.apple.iTunes:MusicBrainz Album Release Country` | `MusicBrainz/Album Release Country` | — |
| `MUSICBRAINZ_ALBUMSTATUS` | `TXXX:MusicBrainz Album Status` | `TXXX:MusicBrainz Album Status` | `MUSICBRAINZ_ALBUMSTATUS` | `RELEASESTATUS` | `----:com.apple.iTunes:MusicBrainz Album Status` | `MusicBrainz/Album Status` | — |
| `MUSICBRAINZ_ALBUMTYPE` | `TXXX:MusicBrainz Album Type` | `TXXX:MusicBrainz Album Type` | `MUSICBRAINZ_ALBUMTYPE` | `RELEASETYPE` | `----:com.apple.iTunes:MusicBrainz Album Type` | `MusicBrainz/Album Type` | — |
| `MUSICBRAIN_ALBUMRELEASECOUNTRY` (alternate) | `TXXX:MusicBrainz Album Release Country` | `TXXX:MusicBrainz Album Release Country` | `RELEASECOUNTRY` | `RELEASECOUNTRY` | — | — | — |

- **DBpoweramp (R):** Reads all MusicBrainz identifiers from TXXX frames and freeform atoms. Write behavior depends on whether the target format supports freeform text fields.
- **Critical notes:**
  - **`UFID` vs `TXXX`:** `MUSICBRAINZ_TRACKID` uses a unique file identifier frame (`UFID`) with owner identifier `http://musicbrainz.org`. All other MusicBrainz IDs use `TXXX` with the descriptive name.
  - **Vorbis:** `RELEASECOUNTRY`, `RELEASESTATUS`, `RELEASETYPE` are the canonical Vorbis key names for these fields (not prefixed with `MUSICBRAINZ_`).
  - **RIFF:** No MusicBrainz identifiers can be stored in RIFF INFO — they must be dropped for WAV output.
  - **MP4:** All MusicBrainz identifiers use the `----:com.apple.iTunes:` freeform namespace.
  - **ID3v2.3 vs v2.4:** No difference for TXXX-based IDs. The `UFID` frame is identical in both versions.

### 6.6 AcoustID

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO |
|---|---|---|---|---|---|---|---|
| `ACOUSTID_ID` | `TXXX:Acoustid Id` | `TXXX:Acoustid Id` | `ACOUSTID_ID` | `ACOUSTID_ID` | `----:com.apple.iTunes:Acoustid Id` | `Acoustid/Id` | — |
| `ACOUSTID_FINGERPRINT` | `TXXX:Acoustid Fingerprint` | `TXXX:Acoustid Fingerprint` | `ACOUSTID_FINGERPRINT` | `ACOUSTID_FINGERPRINT` | `----:com.apple.iTunes:Acoustid Fingerprint` | `Acoustid/Fingerprint` | — |

- **DBpoweramp (R):** Reads from TXXX. Not written to most output formats.
- **Notes:** AcoustID is a Chromaprint audio fingerprint stored alongside its ID. Both should be kept together.

### 6.7 ASIN (Amazon Standard Identification Number)

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO |
|---|---|---|---|---|---|---|---|
| `ASIN` | `TXXX:ASIN` | `TXXX:ASIN` | `ASIN` | `ASIN` | `----:com.apple.iTunes:ASIN` | `ASIN` | — |

- **DBpoweramp (R):** Read from TXXX.

---

## Section 7: ReplayGain Fields

### 7.1 Complete ReplayGain Mapping Table

| Canonical Field | ID3v2 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO |
|---|---|---|---|---|---|---|
| `REPLAYGAIN_TRACK_GAIN` | `TXXX:REPLAYGAIN_TRACK_GAIN` | `REPLAYGAIN_TRACK_GAIN` | `REPLAYGAIN_TRACK_GAIN` | `----:com.apple.iTunes:REPLAYGAIN_TRACK_GAIN` | `REPLAYGAIN_TRACK_GAIN` | — |
| `REPLAYGAIN_TRACK_PEAK` | `TXXX:REPLAYGAIN_TRACK_PEAK` | `REPLAYGAIN_TRACK_PEAK` | `REPLAYGAIN_TRACK_PEAK` | `----:com.apple.iTunes:REPLAYGAIN_TRACK_PEAK` | `REPLAYGAIN_TRACK_PEAK` | — |
| `REPLAYGAIN_ALBUM_GAIN` | `TXXX:REPLAYGAIN_ALBUM_GAIN` | `REPLAYGAIN_ALBUM_GAIN` | `REPLAYGAIN_ALBUM_GAIN` | `----:com.apple.iTunes:REPLAYGAIN_ALBUM_GAIN` | `REPLAYGAIN_ALBUM_GAIN` | — |
| `REPLAYGAIN_ALBUM_PEAK` | `TXXX:REPLAYGAIN_ALBUM_PEAK` | `REPLAYGAIN_ALBUM_PEAK` | `REPLAYGAIN_ALBUM_PEAK` | `----:com.apple.iTunes:REPLAYGAIN_ALBUM_PEAK` | `REPLAYGAIN_ALBUM_PEAK` | — |
| `REPLAYGAIN_REFERENCE_LOUDNESS` | `TXXX:REPLAYGAIN_REFERENCE_LOUDNESS` | `REPLAYGAIN_REFERENCE_LOUDNESS` | `REPLAYGAIN_REFERENCE_LOUDNESS` | `----:com.apple.iTunes:REPLAYGAIN_REFERENCE_LOUDNESS` | `REPLAYGAIN_REFERENCE_LOUDNESS` | — |
| `REPLAYGAIN_TRACK_RANGE` | `TXXX:REPLAYGAIN_TRACK_RANGE` | `REPLAYGAIN_TRACK_RANGE` | `REPLAYGAIN_TRACK_RANGE` | `----:com.apple.iTunes:REPLAYGAIN_TRACK_RANGE` | `REPLAYGAIN_TRACK_RANGE` | — |
| `REPLAYGAIN_ALBUM_RANGE` | `TXXX:REPLAYGAIN_ALBUM_RANGE` | `REPLAYGAIN_ALBUM_RANGE` | `REPLAYGAIN_ALBUM_RANGE` | `----:com.apple.iTunes:REPLAYGAIN_ALBUM_RANGE` | `REPLAYGAIN_ALBUM_RANGE` | — |

### 7.2 ReplayGain Value Format Specification

ReplayGain values follow a strict format specification from the Hydrogenaudio ReplayGain 1.0 and 2.0 specifications:

| Value Type | Format | Example |
|---|---|---|
| **Gain (dB)** | `[-]a.bb dB` (space + "dB" suffix) | `"-6.20 dB"`, `"+3.50 dB"` |
| **Peak (amplitude)** | `c.dddddd` (no suffix) | `"0.987654"`, `"1.000000"` |
| **Reference Loudness** | `[-]a.bb LUFS` | `"-18.00 LUFS"` |
| **Range (dB)** | `[-]a.bb dB` | `"-12.50 dB"` |

**Key formatting rules:**
- Gain values **must** include a space followed by `"dB"` or `"LUFS"`. Players that don't find the suffix may reject the value.
- Peak values **must not** have a suffix. Value 1.000000 represents digital full scale.
- Positive gains have no `+` prefix (e.g., `"3.50 dB"`, not `"+3.50 dB"`).
- Decimal portion is exactly 2 digits for gain, 6 digits for peak.
- All fields are **single-value** — a file cannot have multiple `REPLAYGAIN_TRACK_GAIN` values with different meanings.

### 7.3 Alternative (Legacy/Non-Standard) ReplayGain Key Names

Tools vary in which key names they write. A robust converter must read all of these:

| Format | Alternative Names to Read |
|---|---|
| **Vorbis / FLAC** | `RG_RADIO_GAIN`, `RG_AUDIOFILE_GAIN`, `ALBUM_GAIN`, `TRACK_GAIN`, `replaygain_track_gain` (lowercase), `Replaygain_Track_Gain` (mixed) |
| **ID3v2** | `RVA2` (ID3v2.4 relative volume adjustment — different format, not just key name), `RVAD` (ID3v2.3 relative volume adjustment — deprecated), `RGAD` (legacy ReplayGain frame) |
| **MP4** | Lowercase variants: `----:com.apple.iTunes:replaygain_track_gain`, etc. |
| **ASF/WMA** | `WM/ReplayGain_Track_Gain` (forward slash variant), `WMG/PlaybackGain/TrackGain` (older) |

### 7.4 Opus R128 Tags — CRITICAL DIFFERENCE

Opus files do **not** use ReplayGain tags. They use **R128_TRACK_GAIN** and **R128_ALBUM_GAIN** with a completely different value format:

| Tag | Value Format | Reference |
|---|---|---|
| `R128_TRACK_GAIN` | Integer in Q7.8 fixed-point (units of 1/256 LU) | -23 LUFS reference |
| `R128_ALBUM_GAIN` | Integer in Q7.8 fixed-point (units of 1/256 LU) | -23 LUFS reference |
| `R128_TRACK_PEAK` | Not used (Opus has internal peak limiting) | — |

**Conversion formula from ReplayGain to R128:**
```
R128_value = round((ReplayGain_dB + 5.0) * 256)
```
The `+5.0` accounts for the difference between ReplayGain's -18 LUFS reference and Opus's -23 LUFS reference.

**Example:** `REPLAYGAIN_TRACK_GAIN = "-6.20 dB"` → R128 = round((-6.20 + 5.0) × 256) = round(-1.20 × 256) = -307

### 7.5 iTunes SoundCheck Tags

iTunes uses a proprietary normalization system, completely separate from ReplayGain. These tags store peak values for iTunes' volume normalization:

| Tag | Format | Notes |
|---|---|---|
| `iTunSMPB` | Hex: peak (10 bytes hex) + padding (6 bytes `00 00 00 00 00 00`) | Peak value in hex, followed by 6 zero bytes |
| `iTunNORM` | Hex: peak left (5 bytes hex) + peak right (5 bytes hex) | Legacy normalization values |

**Critical:** Do not confuse SoundCheck tags with ReplayGain tags. They are not interchangeable and use completely different measurement algorithms.

---

## Section 8: Cover Art / Picture Fields

### 8.1 Cover Art Across All Formats

| Format | Storage Mechanism | Key | Binary? | Multi? | Picture Type |
|---|---|---|---|---|---|
| **MP3 (ID3v2)** | `APIC` frame (multiple allowed) | `APIC` | Yes | Yes | 1-byte picture type (0–20) |
| **FLAC/Ogg (Vorbis Comment)** | `METADATA_BLOCK_PICTURE` field (base64-encoded binary) | `METADATA_BLOCK_PICTURE` | Yes (encoded) | Yes | Embedded in picture block |
| **MP4/M4A** | `covr` atom (binary; multiple images in single atom or separate atoms) | `covr` | Yes | Yes | MIME type in data |
| **ASF/WMA** | `WM/Picture` attribute (binary blob per image) | `WM/Picture` | Yes | Yes | Type byte in blob |
| **WAV (RIFF)** | No standard. Options: `LIST RESOURCE` chunk, `ID3 ` chunk with APIC, or `covr` (non-standard) | N/A | — | — | — |
| **AIFF** | `ID3 ` chunk with APIC frames, or `APPL`/`PICT` chunks (non-standard) | `ID3`→`APIC` | Yes | Yes | Same as ID3 |
| **BWF (Broadcast WAV)** | `ID3 ` chunk with APIC frames (same as MP3/AIFF) | `ID3`→`APIC` | Yes | Yes | Same as ID3 |
| **Musepack (APEv2)** | `Cover Art (binary)` item + `Cover Art MIME` item | `Cover Art` | Yes | No | MIME type in separate item |
| **WavPack** | `Cover Art (binary)` item | `Cover Art` | Yes | No | Detected from magic bytes |
| **OptimFROG** | `Cover Art (binary)` item | `Cover Art` | Yes | No | Detected from magic bytes |

### 8.2 ID3v2 APIC Frame Detail

The `APIC` frame is the standard cover art container for MP3 and AIFF files. It has the following internal structure:

```
APIC:
  Text encoding     (1 byte: $00=Latin-1, $01=UTF-16, $02=UTF-16BE, $03=UTF-8)
  MIME type        (string, null-terminated, e.g. "image/jpeg", "image/png")
  Picture type     (1 byte: see table below)
  Description      (string, null-terminated, encoded per text encoding byte)
  Picture data     (binary)
```

**ID3v2 Picture Type Codes:**

| Code | Name | Description |
|---|---|---|
| `0x00` | Other | Other image |
| `0x01` | 32x32 file icon | 32x32 PNG icon (only for file icons) |
| `0x02` | Other file icon | Other file icon |
| `0x03` | **Front cover** | **Front cover image of album — USE THIS for primary cover art** |
| `0x04` | Back cover | Back cover of album |
| `0x05` | Leaflet | Leaflet page in CD case |
| `0x06` | Media | Image on the media (CD, vinyl, etc.) |
| `0x07` | Lead artist | Lead artist/performer |
| `0x08` | Artist/performer | Artist/performer |
| `0x09` | Conductor | Conductor |
| `0x0A` | Band/Orchestra | Band/orchestra |
| `0x0B` | Composer | Composer |
| `0x0C` | Lyricist | Lyricist/text writer |
| `0x0D` | Recording Location | Recording location |
| `0x0E` | During recording | During recording |
| `0x0F` | During performance | During performance |
| `0x10` | Movie/video capture | Movie/video screen capture |
| `0x11` | Bright colored fish | A bright colored fish |
| `0x12` | Illustration | Illustration |
| `0x13` | Band/artist logotype | Band/artist logotype |
| `0x14` | Publisher/Studio logotype | Publisher/Studio logotype |

> **Critical:** DBpoweramp writes type `0x03` (Front Cover) for the primary album cover. Many naive converters write type `0x00` (Other), which causes players to fail to detect it as the album cover. **Always use type `0x03` for the primary cover art.**

### 8.3 FLAC METADATA_BLOCK_PICTURE Format

FLAC and Ogg FLAC store cover art using `METADATA_BLOCK_PICTURE`, a Vorbis Comment field containing a base64-encoded binary structure defined in the FLAC format specification:

```
METADATA_BLOCK_PICTURE (base64-encoded binary block):
  Picture type      (4 bytes, big-endian uint32)  — same type codes as APIC
  MIME length      (4 bytes, big-endian uint32)
  MIME type        (variable, UTF-8, e.g. "image/jpeg")
  Description len  (4 bytes, big-endian uint32)
  Description      (variable, UTF-8)
  Width             (4 bytes, big-endian uint32)
  Height            (4 bytes, big-endian uint32)
  Color depth       (4 bytes, big-endian uint32)
  Colors used       (4 bytes, big-endian uint32) — 0 for indexed
  Picture data len  (4 bytes, big-endian uint32)
  Picture data      (binary JPEG/PNG/GIF/WebP)
```

**Critical:** A naive implementation that writes raw JPEG bytes as a Vorbis Comment field (e.g., `COVERART=<raw_jpeg_bytes>`) is incorrect and will not work with most players. The JPEG bytes must be wrapped in the `METADATA_BLOCK_PICTURE` structure and base64-encoded.

Some older tools use the legacy `COVERART` and `COVERARTMIME` fields instead of `METADATA_BLOCK_PICTURE`. These are non-standard but widely supported:
- `COVERART` = base64-encoded image data
- `COVERARTMIME` = MIME type (e.g., `image/jpeg`)

**DBpoweramp:** Writes `METADATA_BLOCK_PICTURE` in FLAC. Reads both `METADATA_BLOCK_PICTURE` and legacy `COVERART`.

### 8.4 MP4 covr Atom

The `covr` atom in MP4 stores cover art as binary data. Multiple images are stored as separate `covr` atoms (each containing one image) or as a single `covr` atom with multiple images concatenated. The image format (JPEG, PNG, GIF, etc.) is detected from the binary data's magic bytes.

| Magic Bytes | Format |
|---|---|
| `FF D8 FF` | JPEG |
| `89 50 4E 47` | PNG |
| `47 49 46 38` | GIF |
| `52 49 46 46 ... 57 45 42 50` | WebP (RIFF container) |

**DBpoweramp:** Writes all cover art types to `covr`. Reads `covr` and converts to target format's native storage.

### 8.5 ASF WM/Picture Attribute

ASF stores pictures in a `WM/Picture` attribute as a binary blob:

```
WM/Picture (binary):
  Picture type     (1 byte)
  MIME length     (4 bytes, uint32)
  MIME type       (variable, UTF-16-LE)
  Description len (4 bytes, uint32)
  Description     (variable, UTF-16-LE)
  Picture data len (4 bytes, uint32)
  Picture data    (binary)
```

Same type codes as ID3v2 APIC. Use type `0x03` for front cover.

### 8.6 Multiple Cover Art Images

When a file has multiple cover art images, the recommended priority order is:

1. **Front Cover** (`0x03`) — primary album art. This is the one to extract and display.
2. **Back Cover** (`0x04`) — back of album packaging.
3. **Media** (`0x06`) — image of the CD/vinyl.
4. **Other** (`0x00`) — any other relevant image.

For conversion, always preserve at least the Front Cover. Drop others if the target format does not support multiple images (e.g., if forced to write only one).

---

## Section 9: Custom and Extended Fields

### 9.1 TXXX User-Defined Frame Keys

ID3v2 `TXXX` frames store user-defined key-value pairs. The frame structure is:

```
TXXX:
  Text encoding   (1 byte)
  Description     (string, null-terminated, encoding per above)
  Value           (string)
```

The description field is the "key" — e.g., `TXXX: CATALOGNUMBER` stores the catalog number. The following TXXX keys are standardized by various communities:

| TXXX Description | Canonical Field | Notes |
|---|---|---|
| `CATALOGNUMBER` | `CATALOGNUMBER` | Record label catalog number |
| `BARCODE` | `BARCODE` | Product barcode |
| `UPC` | `UPC` | UPC/EAN code |
| `MUSICBRAINZ ALBUM ID` | `MUSICBRAINZ_ALBUMID` | Album MBID |
| `MUSICBRAINZ ARTIST ID` | `MUSICBRAINZ_ARTISTID` | Artist MBID |
| `MUSICBRAINZ TRACK ID` | (use `UFID:http://musicbrainz.org`) | Use UFID for track ID |
| `MUSICBRAINZ RELEASE TRACK ID` | `MUSICBRAINZ_RELEASETRACKID` | Release track MBID |
| `ACOUSTID ID` | `ACOUSTID_ID` | AcoustID |
| `ACOUSTID FINGERPRINT` | `ACOUSTID_FINGERPRINT` | Chromaprint fingerprint |
| `SCRIPT` | `SCRIPT` | Script type (e.g., "Latn") |
| `DIRECTOR` | `DIRECTOR` | Film/video director |
| `MOOD` | `MOOD` | Mood (non-standard for ID3v2.3) |
| `WORK` | `WORK` | Musical work name |
| `WRITER` | `WRITER` | Writer (songwriter) |
| `SHOWMOVEMENT` | `SHOWMOVEMENT` | Show movement flag |
| `ASIN` | `ASIN` | Amazon Standard Identification Number |
| `iTunes_CDDB_1` | `itunes_cddb_1` | iTunes CD disc ID |
| `iTunes_CDDB_2` | `itunes_cddb_2` | Additional CD disc ID |
| `MusicMagic Fingerprint` | `musicip_fingerprint` | MusicIP PUID generator fingerprint |
| `MusicIP PUID` | `musicip_puid` | MusicIP PUID |

> **Note:** The TXXX description is case-sensitive in ID3v2, but most tools use mixed case. Picard writes `MusicBrainz Album Id` with specific capitalization. DBpoweramp preserves the description string exactly as read.

### 9.2 ID3v2 TMCL (Musician Credits List) — ID3v2.4 Only

`TMCL` stores instrument-to-performer mappings. Format: `instrument=performer` pairs, separated by the null character (`\0`):

```
TMCL:
  Text encoding:  $03 (UTF-8)
  People list:   "Guitar=John Smith\0Drums=Mike Jones\0Bass=Sarah Lee\0"
```

Common instrument values used in TMCL:

| Instrument | Role in TMCL | Notes |
|---|---|---|
| Piano / acoustic-piano / grand-piano | Piano | — |
| Violin / fiddle | Violin | — |
| Cello | Cello | — |
| Viola | Viola | — |
| Double bass / bass / acoustic-bass | Bass | — |
| Guitar / electric-guitar / acoustic-guitar | Guitar | — |
| Drums / drum-set / percussion | Drums | — |
| Flute | Flute | — |
| Clarinet | Clarinet | — |
| Oboe | Oboe | — |
| Trumpet | Trumpet | — |
| Saxophone / tenor-saxophone / alto-saxophone | Saxophone | — |
| Trombone | Trombone | — |
| French horn | Horn | — |
| Harp | Harp | — |
| Organ / Hammond-organ | Organ | — |
| Synthesizer / synthesizer / keyboards | Synthesizer | — |
| Voice / vocals / vocal | Vocals | — |
| Choir / chorus / vocal | Choir | — |

### 9.3 ID3v2 TIPL (Involved People List) — ID3v2.4 Only

`TIPL` stores general roles and the people who performed them (as opposed to `TMCL` which maps instruments to performers). Format is the same as TMCL:

```
TIPL:
  Text encoding: $03 (UTF-8)
  People list:  "producer=John Doe\0engineer=Jane Smith\0mixer=Bob Wilson\0"
```

Standard roles for TIPL:

| Role | ID3v2.4 | ID3v2.3 equivalent |
|---|---|---|
| Producer | `TIPL:producer` | `IPLS:producer` |
| Engineer | `TIPL:engineer` | `IPLS:engineer` |
| Mixer | `TIPL:mix` | `IPLS:mix` |
| DJ-Mix / DJ Mixer | `TIPL:DJ-mix` | `IPLS:DJ-mix` |
| Arranger | `TIPL:arranger` | `IPLS:arranger` |
| Lyricist | `TIPL:lyricist` | `IPLS:lyricist` |
| Original artist | `TIPL:originalartist` | `IPLS:originalartist` |
| Composer | `TIPL:composer` | `IPLS:composer` |
| Lyricist | `TIPL:lyricist` | `IPLS:lyricist` |
| Publisher | `TIPL:publisher` | `IPLS:publisher` |
| Record label | `TIPL:label` | `IPLS:label` |

### 9.4 MP4 Freeform Atoms (----:com.apple.iTunes:)

MP4 stores custom fields in the `----` freeform namespace. The full atom path is:

```
moov/udta/meta/ilst/----/mean (contains "com.apple.iTunes")/----/name (contains the key)/----/data (contains the value)
```

Simplified representation: `----:com.apple.iTunes:KEYNAME`

These freeform atoms are widely supported by third-party tools and are the standard way to store custom metadata in M4A files. Picard and many taggers use them for MusicBrainz IDs, AcoustID, ReplayGain, and many other fields.

### 9.5 ASF Custom Attributes

ASF supports arbitrary attribute names in the Extended Content Description object. Picard and other tools use namespaced prefixes:

| Namespace | Example Attribute | Notes |
|---|---|---|
| `MusicBrainz/` | `MusicBrainz/Track Id` | MBIDs |
| `Acoustid/` | `Acoustid/Id` | AcoustID |
| `beets/` | `beets/Album Artist Credit` | beets custom fields |
| `WM/` | `WM/AlbumArtist` | Standard WMA attributes |
| `REPLAYGAIN_` | `REPLAYGAIN_TRACK_GAIN` | ReplayGain attributes |

### 9.6 Vorbis Comment Role-Qualified Fields

Vorbis Comment uses a `ROLE=VALUE` syntax for role-qualified fields. The role appears after an equals sign:

| Field | Format | Example |
|---|---|---|
| **Performer** | `PERFORMER=Artist` | `PERFORMER=Piano:Clara Haskil` |
| **Composer** | `COMPOSER=Artist` | `COMPOSER=Orchestra:Berlin Philharmonic` |
| **Arranger** | `ARRANGER=Artist` | `ARRANGER=Orchestration:Debussy` |
| **Engineer** | `ENGINEER=Name` | `ENGINEER=Studio:Abbey Road` |
| **Producer** | `PRODUCER=Name` | `PRODUCER=Album:Rick Rubin` |
| **Label** | `LABEL=Name` | `LABEL=Catalog:ECM 1234` |

This is the Vorbis equivalent of ID3v2's TMCL/TIPL system. Not all players parse these correctly.

### 9.7 Podcast-Specific Fields

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO |
|---|---|---|---|---|---|---|---|
| `PODCAST` | `PCST` | `PCST` | — | — | `pcst` (bool) | — | — |
| `PODCASTURL` | `WFED` | `WFED` | — | — | `purl` | — | — |
| `PODCASTDESC` | `TDES` | `TDES` | — | — | `ldes` | — | — |
| `PODCASTID` | `TGID` | `TGID` | — | — | `egid` | — | — |
| `PODCASTCATEGORY` | `TCAT` | `TCAT` | — | — | `catg` | — | — |
| `PODCASTKEYWORDS` | `TKWD` | `TKWD` | — | — | `keyw` | — | — |

### 9.8 Rating Fields

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO |
|---|---|---|---|---|---|---|---|
| `POPULARIMETER` (rating+counter) | `POPM:email@server` | `POPM:email@server` | — | `RATING:user@email` | — | `WM/SharedUserRating` | — |
| `PLAYCOUNTER` | `PCNT` | `PCNT` | — | — | — | — | — |

**POPM frame structure (ID3v2):**
```
POPM:
  Email to user     (string, null-terminated)
  Rating            (1 byte: 0=worst, 255=best; commonly 1-5 stars mapped to 0-255)
  Counter           (4 bytes, big-endian uint32, optional play count)
```

Common rating scales:
- **Windows Media Player:** 1–5 stars → POPM value 1–255 (WMP maps stars to specific values)
- **MediaMonkey:** 1–5 stars → POPM value 1–5 (MediaMonkey uses direct 1-5 scale)
- **iTunes:** Uses the `rate` atom (0–100 scale), not POPM
- **MP4 `rate`:** Integer 0–100

### 9.9 Encoder / Production Fields

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO |
|---|---|---|---|---|---|---|---|
| `ENCODEDBY` | `TENC` | `TENC` | `EncodedBy` | `ENCODEDBY` | `©too` | `WM/EncodedBy` | `ISFT` |
| `ENCODERSETTINGS` | `TSSE` | `TSSE` | `EncoderSettings` | `ENCODERSETTINGS` | — | `WM/EncodingSettings` | — |
| `COPYRIGHT` | `TCOP` | `TCOP` | `Copyright` | `COPYRIGHT` | `cprt` | `Copyright` | `ICOP` |
| `SOURCE` (Original Filename) | `TOFN` | `TOFN` | `OriginalFilename` | `ORIGINALFILENAME` | — | `WM/OriginalFilename` | — |
| `SOURCE` (Source Media) | — | — | `Source` | `SOURCEMEDIA` | — | `WM/Media` | `IMED` |

- **DBpoweramp:** `ENCODEDBY` is read/write. When converting, DBpoweramp updates this to reflect the new encoder rather than preserving the old one. `ISFT` in RIFF INFO records the software used to create the file.
- **ENCODERSETTINGS** (`TSSE`) in ID3v2 records the encoder settings (e.g., `"LAME 3.100 -V2"`). This is not supported in MP4, RIFF, or AIFF natively.

### 9.10 Lyrics and Comment Fields

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO |
|---|---|---|---|---|---|---|---|
| `UNSYNCEDLYRICS` | `USLT:description` | `USLT:description` | `Lyrics` | `LYRICS` | `©lyr` | `WM/Lyrics` | — |
| `SYNCEDLYRICS` | `SYLT:description` | `SYLT:description` | — | — | — | `WM/Lyrics_Synchronised` | — |
| `COMMENT` | `COMM:description` | `COMM:description` | `Comment` | `COMMENT` | `©cmt` | `Description` | `ICMT` |
| `DESCRIPTION` | `TXXX:DESCRIPTION` | `TXXX:DESCRIPTION` | `Description` | `DESCRIPTION` | `desc` | `Description` | — |

**COMM/USLT frame structure (ID3v2):**
```
COMM/USLT:
  Text encoding     (1 byte)
  Language          (3 bytes: ISO 639-2, e.g. "eng", "XXX" for undefined)
  Content descriptor (string, null-terminated) — empty for generic comment
  Text              (the actual comment/lyrics)
```

**DBpoweramp behavior for comments:**
- **Read:** Reads the first `COMM` frame regardless of language, or `COMM` with language `XXX` or `eng`.
- **Write:** Writes `COMM` with language `\0\0\0` (null/disabled) or `"XXX"` and an empty descriptor for maximum compatibility.
- **Multiple comments:** If source has multiple `COMM` frames with different descriptors, DBpoweramp may concatenate them with newlines or keep only the first. When converting to formats that don't support multiple comments (RIFF INFO has only one `ICMT`), only the first is written.

### 9.11 Sort Order Fields

| Canonical Field | ID3v2.3 | ID3v2.4 | APEv2 | Vorbis Comment | MP4 Atom | ASF/WMA | RIFF INFO |
|---|---|---|---|---|---|---|---|
| `ALBUMSORT` | `TSOA` | `TSOA` | `ALBUMSORT` | `ALBUMSORT` | `soal` | `WM/AlbumSortOrder` | — |
| `ARTISTSORT` | `TSOP` | `TSOP` | `ARTISTSORT` | `ARTISTSORT` | `soar` | `WM/ArtistSortOrder` | — |
| `ALBUMARTISTSORT` | `TSO2` / `TXXX:ALBUMARTISTSORT` | `TSO2` | `ALBUMARTISTSORT` | `ALBUMARTISTSORT` | `soaa` | `WM/AlbumArtistSortOrder` | — |
| `TITLESORT` | `TSOT` | `TSOT` | `TITLESORT` | `TITLESORT` | `sonm` | `WM/TitleSortOrder` | — |
| `COMPOSERSORT` | `TSOC` / `TXXX:COMPOSERSORT` | `TSOC` | `COMPOSERSORT` | `COMPOSERSORT` | `soco` | `WM/ComposerSortOrder` | — |
| `SHOWSORT` | — | — | — | — | `sosn` | — | — |

- **ID3v2.3 note:** `TSO2` (Album Artist Sort) was introduced in ID3v2.4. In ID3v2.3, Picard uses `TXXX:ALBUMARTISTSORT`. DBpoweramp may use either. When converting to ID3v2.3, prefer `TXXX:ALBUMARTISTSORT` for compatibility.
- **Picard version history:** `TSOC` (Composer Sort) was added in Picard 1.3. Earlier versions used `TXXX:COMPOSERSORT`.
- **MP4 sort atoms:** `soal`=`TSOA`, `soar`=`TSOP`, `soaa`=`TSO2`, `sonm`=`TSOT`, `soco`=`TSOC`. These are iTunes-defined atoms starting with `so`.

---

## Appendix A: AIFF Standard Chunks

AIFF (Audio Interchange File Format) uses a chunk-based IFF container. Native AIFF metadata chunks include:

| Chunk ID | Contents | Notes |
|---|---|---|
| `NAME` | Sound name (title) | ASCII, up to 255 chars |
| `AUTH` | Author/artist name | ASCII, up to 255 chars |
| `ANNO` | Annotation/comment | ASCII, freeform |
| `MIDI` | MIDI grab chunk | Rarely used |
| `COMT` | Comment chunk (multiple) | Structured: timestamp + text |
| `(c) ` | Copyright notice | ASCII |
| `ID3 ` | ID3v2 tag chunk (non-standard) | Contains full ID3v2 frames |

**`COMT` chunk structure (AIFF standard):**
```
COMT:
  Timestamp       (4 bytes, milliseconds since 1904)
  Marker count    (2 bytes, uint16)
  Comments:       [timestamp, marker_id, count (2 bytes), text]* repeated
```

**`ID3 ` chunk (non-standard but universal):**
The `ID3 ` chunk (note the trailing space) is universally used by audio tools to embed ID3v2 metadata in AIFF files. It contains a complete ID3v2 tag (header + frames) and follows the same rules as ID3v2 in MP3. This is the mechanism by which full ID3v2 field support is achieved in AIFF files.

> **DBpoweramp behavior:** When converting to AIFF, DBpoweramp writes an `ID3 ` chunk with ID3v2.4 frames (or ID3v2.3 if configured). Native AIFF chunks are written for basic fields (NAME, AUTH, ANNO). The `ID3 ` chunk carries all extended fields (MusicBrainz IDs, ReplayGain, etc.).

---

## Appendix B: RIFF INFO Reference

RIFF INFO is the standard metadata system for WAV files. It uses a `LIST INFO` container with 4-character chunk IDs:

| Chunk ID | Contents | Max Length | Notes |
|---|---|---|---|
| `INAM` | Title/song name | 256 | Maps to `TITLE` |
| `IART` | Artist | 256 | Maps to `ARTIST` |
| `IPRD` | Product/album | 256 | Maps to `ALBUM` |
| `ICRD` | Creation date | 256 | Maps to `YEAR`/`DATE` (write as `YYYY` or `YYYY-MM-DD`) |
| `ICMT` | Comment | 256 | Maps to `COMMENT` |
| `ICOP` | Copyright | 256 | Maps to `COPYRIGHT` |
| `IGNR` | Genre | 256 | Maps to `GENRE` |
| `ISFT` | Software/encoder | 256 | Maps to `ENCODEDBY` / encoder name |
| `ISRC` | ISRC code | 256 | Maps to `ISRC` |
| `IWRI` | Writer/lyricist | 256 | Maps to `WRITER` |
| `IENG` | Engineer | 256 | Maps to `ENGINEER` |
| `IMED` | Medium/source | 256 | Maps to `MEDIA` |
| `IPRO` | Producer | 256 | Maps to `PRODUCER` |
| `ILNG` | Language | 256 | Maps to `LANGUAGE` |
| `ITRK` | Track number | 256 | Non-standard but used by some tools |
| `ICNT` | Country | 256 | Maps to `RELEASECOUNTRY` |
| `ICRP` | Crop | 256 | Non-standard |
| `IDIM` | Dimensions | 256 | Non-standard |
| `IDPI` | Dots per inch | 256 | Non-standard |
| `ICLS` | Class | 256 | Non-standard |
| `ISSN` | Serial number | 256 | Non-standard |
| `ISGN` | Signed/unsigned | 256 | Non-standard |
| `ITCK` | Ticket | 256 | Non-standard |

**DBpoweramp behavior for WAV output:** DBpoweramp writes RIFF INFO chunks. Fields that have no RIFF equivalent (MusicBrainz IDs, ReplayGain, cover art) are written to a `LIST INFO` chunk using `TXXX`-style keys if possible, or dropped if not. BWF (Broadcast WAV) additionally supports the `LIST` chunk with `XMP` (Extensible Metadata Platform) for full XMP support.

**For BWF (Broadcast WAV):** The `BEXT` chunk provides standardized broadcast metadata (description, originator, origination date/time, etc.) and is distinct from RIFF INFO. See the BWF specification (EBU Tech 3285) for the full BEXT field list.

---

## Appendix C: Max Field Value Lengths by Format

| Format | Per-Field Limit | Notes |
|---|---|---|
| **ID3v2.3** | ~2 GB per frame (4-byte size field) | Players typically truncate at 255–1024 chars |
| **ID3v2.4** | ~2 GB per frame (4-byte size field) | Same as v2.3 |
| **Vorbis Comment** | No hard limit; recommended ≤ 4096 chars | libvorbis default vendor string length limits apply to the vendor tag |
| **APEv2** | 2 GB per item (variable-length size) | Practical limit: whatever the tool supports |
| **MP4 atom** | No specific limit; 4-byte size field implies ~2 GB | iTunes typically handles up to several MB per atom |
| **ASF attribute** | 64 KB per attribute | Hard limit defined in ASF spec |
| **RIFF INFO** | 256 bytes per chunk | Hard limit (chunk size is 4 bytes, used for data) |
| **AIFF chunk** | No specific limit for metadata chunks | Limited by file size; NAME/AUTH/ANNO typically ≤ 255 |

---

## Appendix D: Character Encoding by Format

| Format | Default Encoding | International Characters |
|---|---|---|
| **ID3v2.3** | Latin-1 (ISO-8859-1) for frames; UTF-16 with BOM for international | Use `$01` (UTF-16) text encoding flag for non-Latin scripts |
| **ID3v2.4** | UTF-8 | All text frames use UTF-8 |
| **Vorbis Comment** | UTF-8 | UTF-8 per spec; tools may fall back to Latin-1 |
| **APEv2** | UTF-8 | UTF-8 item flag |
| **MP4** | UTF-8 (stored as UTF-8 bytes) | Full Unicode support |
| **ASF** | UTF-16-LE | Full Unicode support |
| **RIFF INFO** | ASCII (ISO-646) | No native Unicode; use Latin-1 for Western European |
| **AIFF ID3 chunk** | Same as ID3v2 (inherits ID3 encoding rules) | — |

---

## Appendix E: Format Support Summary — Quick Lookup

Use this table to quickly determine which formats support which field categories:

| Field Category | MP3 | FLAC | MP4/M4A | WMA | WAV | AIFF | Musepack | WavPack |
|---|---|---|---|---|---|---|---|---|
| Basic fields (Title, Artist, Album) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Track/Disc numbers | ✅ | ✅ | ✅ | ✅ | Limited | ✅ | ✅ | ✅ |
| Date fields (full timestamps) | ✅ v2.4 | ✅ | ✅ | Limited | ICRD only | ✅ v2.4 | ✅ | ✅ |
| MusicBrainz IDs | ✅ TXXX/UFID | ✅ | ✅ freeform | ✅ | ❌ | ✅ | ✅ | ✅ |
| ReplayGain | ✅ TXXX | ✅ Vorbis | ✅ freeform | ✅ | ❌ / BEXT | ✅ APEv2 | ✅ APEv2 |
| Cover art | ✅ APIC | ✅ METADATA_BLOCK_PICTURE | ✅ covr | ✅ WM/Picture | ❌ / BEXT | ✅ ID3 | ✅ APEv2 | ✅ APEv2 |
| Custom fields | ✅ TXXX | ✅ freeform | ✅ freeform | ✅ freeform | ❌ | ✅ TXXX | ✅ freeform | ✅ freeform |
| Mood | ✅ v2.4 | ✅ | ❌ | ✅ | ❌ | ✅ v2.4 | ✅ | ✅ |
| Movement/Mwork | ✅ v2.4 | ✅ | ✅ | ❌ | ❌ | ✅ v2.4 | ✅ | ✅ |
| Sort order | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ | ✅ | ✅ |
| Multi-value per field | ❌ (use TXXX) | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ✅ |

---

## Sources

1. **MusicBrainz Picard Tag Mapping v3.0** — https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html
2. **Mp3tag Tag Field Mappings** — https://help.mp3tag.de/main_tags.html
3. **Hydrogenaudio ReplayGain 2.0 Specification** — https://wiki.hydrogenaudio.org/index.php?title=ReplayGain_2.0_specification
4. **Hydrogenaudio Original ReplayGain Specification** — https://wiki.hydrogenaudio.org/index.php?title=Original_ReplayGain_specification
5. **ID3.org ID3v2.3.0 Specification** — https://id3.org/id3v2.3.0
6. **TagLib API Documentation** — https://taglib.org/api/
7. **Xiph.org Vorbis Comment Specification** — https://xiph.org/vorbis/doc/v-comment.html
8. **beetbox/mediafile** (beets cross-format tagging library) — https://github.com/beetbox/mediafile
9. **FLAC Format Specification** — https://xiph.org/flac/documentation_format_overview.html
10. **AIFF Format (Just Solve the File Format Problem)** — http://justsolve.archiveteam.org/wiki/AIFF
11. **Broadcast Bridge: ID3 Metadata Tagging** — https://www.thebroadcastbridge.com/content/entry/21824/standards-id3-metadata-tagging
12. **Apple iTunes Metadata Format Specification** — (referenced via Mp3tag and Picard documentation)
13. **EBU Tech 3285 — BWF Broadcast WAV Format** — (standard reference for BEXT chunk fields)

---

## Would a User Notice Any Difference from DBpoweramp?

Yes — and for several specific field categories:

1. **Mood (`TMOO`)** — DBpoweramp can write this to MP3 only in ID3v2.4 mode. Converting to ID3v2.3 or WAV/AIFF will silently drop the Mood field. Users who tag with Mood (from MusicBrainz Picard or specialized taggers) will find it missing after conversion unless the pipeline preserves it via a custom TXXX frame.

2. **Disc Subtitle (`TSST`)** — ID3v2.4 only. When DBpoweramp converts to ID3v2.3 MP3, disc subtitle names are lost. This is a known limitation of v2.3 compatibility.

3. **Sort Order fields (`TSOA`, `TSOP`, `TSOT`, `TSOC`, `TSO2`)** — All are in ID3v2.3 and later, but MP4 sort atoms (`soal`, `soar`, `sonm`, `soco`, `soaa`) are iTunes-specific. When converting from MP4 to WAV, sort order fields have no RIFF INFO equivalent and will be dropped.

4. **MusicBrainz identifiers in WAV** — No standard WAV field for MBIDs. Any pipeline that converts FLAC/M4A/MP3 to WAV will lose all MusicBrainz track/album/artist IDs unless it uses a non-standard extension. Users who rely on MBIDs for music identification will notice complete data loss.

5. **ReplayGain in WAV** — Standard WAV/PCM has no metadata system. Only BWF (Broadcast WAV) can carry ReplayGain via an ID3 `ID3 ` chunk. A naive WAV conversion loses all ReplayGain data.

6. **Cover art in WAV** — No standard. DBpoweramp embeds cover art in WAV only if the output format supports it (some tools write to a `LIST RESOURCE` chunk). Standard WAV players will not display embedded cover art.

7. **Date field precision** — When converting from MP4 (`©day = "2024-03-15T00:00:00Z"`) to ID3v2.3 MP3, the timestamp is reduced to a 4-digit year (`TYER = "2024"`). The month and day are lost. Users who rely on precise release dates will notice truncation.

8. **Compilation flag in non-iTunes formats** — `TCMP` in ID3v2 is non-standard (it's an iTunes frame, not in the ID3 spec). While widely supported, some players do not recognize it. DBpoweramp's treatment of this flag for Various Artists grouping may not carry over to all playback systems.

9. **Multiple values in single-value fields** — When a FLAC file has multiple `COMPOSER` values (e.g., two co-composers), DBpoweramp joins them with ` / ` when converting to MP3. Some downstream tools split on this delimiter incorrectly, creating phantom duplicate entries.

10. **Initial Key (`TKEY`)** — Not supported in MP4 at all. Not supported in RIFF INFO. Any key signature data is lost when converting to these formats.
