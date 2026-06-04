# DBpoweramp Music Converter — Tag Reading Pipeline: All Formats

> **Research Area:** DBpoweramp Music Converter — Tag Reading Pipeline
> **Covers:** Multi-tag-system priority, per-format priority matrices, reading sequences, graceful degradation, field-level reading rules, ID3v2 version detection, cover art priority
> **Confidence Level:** High (official docs + TagLib/Mutagen/Mp3tag cross-reference)
> **Cross-references:** File 12 (Tag Writing Pipeline), File 15 (Metadata Preservation), File 16 (Conflict Resolution), File 17 (Custom Tags), File 20 (Cover Art Read Pipeline)

---

## 1. Overview & Purpose

This document exhaustively describes how DBpoweramp Music Converter reads and merges metadata tags from every supported audio format. It covers the priority rules when multiple tag systems coexist in a single file, how DBpoweramp detects and handles different tag versions, what happens when tag data is malformed or ambiguous, and how each format's tag system is resolved into a unified internal representation.

DBpoweramp delegates tag reading to **TagLib** (and to some extent FFmpeg's libavformat), which means its read behavior closely mirrors TagLib's documented semantics, supplemented by DBpoweramp-specific configuration options and the realities of CD-ripping workflows where files arrive with legacy tags from many different encoders and rippers.

### 1.1 DBpoweramp's Tag Reading Philosophy

DBpoweramp's read pipeline is characterized by three principles:

1. **Format-native primary reads.** Each format is read using its canonical tag system first. DBpoweramp does not, for example, look for ID3v2 tags inside FLAC files by default — it reads Vorbis Comments.
2. **Conservative fallback.** When the primary tag system is absent or empty, DBpoweramp falls back to any secondary tag systems the format supports. Legacy systems (ID3v1, RIFF INFO) are treated as compatibility layers, not primary sources.
3. **TagLib as the implementation engine.** DBpoweramp uses TagLib for most tag I/O. TagLib's priority rules therefore define DBpoweramp's behavior. Where TagLib's behavior is underspecified, DBpoweramp's GUI options and documented defaults fill the gaps.

---

## 2. Multi-Tag-System Priority Philosophy

### 2.1 The Fundamental Principle: "First Match Wins, Not Merges"

DBpoweramp's (via TagLib) tag reading strategy is **not a merge**. When multiple tag systems exist in the same file, DBpoweramp selects the highest-priority system that contains a given field and uses that value exclusively. It does not concatenate values from competing tag systems for the same field.

This is confirmed by TagLib's `PropertyMap` semantics: the PropertyMap is a flat dictionary where each key maps to a `StringList` of values, but those values originate from a single tag format — not from a union of all formats. ([TagLib API Documentation](https://taglib.org/api/))

### 2.2 The Three Priority Schools

No universal standard governs which tag system wins when multiple coexist. Every tag reader implements its own priority. The three dominant schools are:

| School | Priority Order | Represented By |
|--------|---------------|----------------|
| **APE-wins** | APEv2 → ID3v2 → ID3v1 | Mp3tag (default), AIMP, foobar2000 |
| **ID3v2-wins** | ID3v2 → APEv2 → ID3v1 | TagLib (used by DBpoweramp), MusicBee, Mutagen |
| **ID3v1-wins** | ID3v1 → ID3v2 → APEv2 | AzuraCast, some legacy players |

DBpoweramp uses **TagLib**, which implements the **ID3v2-wins** school. This is the critical fact that drives most of DBpoweramp's read behavior.

**Source:** [Hydrogenaudio — ID3v2 vs APEv2](https://hydrogenaudio.org/index.php?msg=637031), [music-metadata library — priority mapping](https://github.com/Borewit/music-metadata/blob/feb15be8bd8da14ab10cca4c1068be2408fa7b1e/lib/common/MetadataCollector.ts#L17)

### 2.3 The Mp3tag APE Override Problem

A major source of user confusion arises when Mp3tag writes APEv2 tags to MP3 files. The specific scenario:

1. A file has a populated ID3v2 tag (Title, Artist, Track, Genre).
2. Mp3Gain writes ReplayGain data into an APEv2 tag.
3. If APEv2 is empty for Title, Artist, etc., but the APEv2 tag exists, and a reader uses the APE-wins school, the empty APE fields will **override** the populated ID3v2 fields.

DBpoweramp's ID3v2-wins school prevents this problem: ID3v2 text fields are always read even when an APEv2 tag is present. APEv2 is used only for fields that are APE-native (ReplayGain, some ratings).

**Source:** [Hydrogenaudio — ID3v2 vs APEv2](https://hydrogenaudio.org/index.php?msg=637031)

### 2.4 DBpoweramp's Write-Read Asymmetry

DBpoweramp writes only one active tag system per format (with the MP3 ID3v2+ID3v1 exception). However, files arriving from other sources (internet downloads, rips by other software, CDs with legacy metadata) may arrive with multiple tag systems. DBpoweramp's read pipeline handles these cases robustly via TagLib's priority system.

---

## 3. Per-Format Priority Matrix

### 3.1 Complete Priority Table

| Format | Priority 1 (Primary) | Priority 2 | Priority 3 | Notes |
|--------|---------------------|-----------|-----------|-------|
| **MP3** | ID3v2.4 | ID3v2.3 | APEv2 → ID3v1 | v2.4 and v2.3 coexist; higher version preferred for date frames |
| **FLAC** | Vorbis Comments | ID3v2 (legacy) | ID3v1 (legacy) | ID3v2 is non-standard in FLAC; Vorbis always wins |
| **OGG Vorbis** | Vorbis Comments | — | — | Single system; no fallback |
| **Opus** | Vorbis Comments (OpusTags) | — | — | Single system; no fallback |
| **M4A / AAC / ALAC** | MP4 atoms (ilst) | — | — | Single system; no fallback |
| **WAV** | ID3v2 chunk | RIFF LIST-INFO | BWF bext | ID3v2 preferred for most fields; bext for archival fields |
| **AIFF** | ID3v2 chunk | AIFF native chunks | — | ID3v2 is de facto standard; native chunks macOS-only |
| **WMA / ASF** | ASF Content Description | Extended Content Description | Metadata Objects | Flat collection; no priority |
| **WavPack WV** | APEv2 | ID3v1 (read-only) | — | APEv2 always wins; ID3v1 ignored if APEv2 present |
| **APE (Monkey's Audio)** | APEv2 | APEv1 | ID3v1 | v3.99+ uses APEv2; v1/v1.1 used v1 |
| **TAK** | APEv2 | — | — | Single system; appended at end of file |
| **TTA (True Audio)** | APEv2 | ID3v2 | ID3v1 | Most ambiguous format; application-dependent |
| **MusePack MPC** | APEv2 | ID3v1 (legacy) | — | SV8 format; APEv2 primary |
| **MusePack SV7** | APEv2 | ID3v1 (legacy) | — | Older SV7 format |
| **TTA** | APEv2 (foobar2000/AIMP) | ID3v2 (Picard) | ID3v1 | No universal standard; tool-dependent |

### 3.2 MP3 — The Most Complex Case

MP3 is the only format where DBpoweramp must routinely resolve conflicts between multiple concurrent tag systems. A single MP3 file can legitimately contain all four of these simultaneously:

1. **ID3v2.4** (10-byte header, synchsafe frame sizes)
2. **ID3v2.3** (10-byte header, big-endian frame sizes)
3. **APEv2** (trailing tag at end of file)
4. **ID3v1 / ID3v1.1** (128-byte footer)

#### 3.2.1 DBpoweramp's MP3 Read Priority (via TagLib)

| Priority | Tag System | Used For |
|----------|-----------|---------|
| **1st** | ID3v2.4 | All standard text fields (TIT2, TPE1, TALB, etc.); all binary frames (APIC) |
| **2nd** | ID3v2.3 | Falls back when v2.4 frames are absent |
| **3rd** | APEv2 | ReplayGain (REPLAYGAIN_TRACK_GAIN, REPLAYGAIN_ALBUM_GAIN); ratings (POPM); custom fields |
| **4th** | ID3v1 | Last resort fallback; used when neither ID3v2 nor APEv2 are present |

#### 3.2.2 Critical Detail: ID3v2.4 vs ID3v2.3 Coexistence

A file can have both ID3v2.4 and ID3v2.3 tags simultaneously (written by different tools). TagLib reads **the ID3v2.4 tag** for all standard frames. When v2.4 is absent, v2.3 fills the gaps. The v2.3 tag is never read if v2.4 is present, even for frames where v2.4 lacks a v2.3 counterpart.

#### 3.2.3 Critical Detail: APEv2 in MP3

DBpoweramp uses APEv2 only for APE-native fields. APEv2 in MP3 is typically written by:
- **Mp3Gain** — for ReplayGain tags
- **foobar2000** — for advanced tagging
- **Mp3tag** — when configured to write APE instead of ID3v2

If a file has ID3v2 for text fields and APEv2 for ReplayGain, DBpoweramp reads the text fields from ID3v2 and the ReplayGain from APEv2 — combining both sources for the final metadata set.

#### 3.2.4 Critical Detail: ID3v1 Fallback

ID3v1 is the absolute last resort. TagLib reads ID3v1 only when ID3v2 is entirely absent from the file. The 128-byte ID3v1 tag at the end of the file contains only: Title (30), Artist (30), Album (30), Year (4), Comment (30), Track (1 byte in v1.1), Genre (1 byte). Any field not present in ID3v1 remains empty.

### 3.3 FLAC — Vorbis Comments Only

FLAC files should contain only Vorbis Comments in the `VORBIS_COMMENT` metadata block. However, some non-compliant tools write ID3v2 at the start of FLAC files. DBpoweramp's behavior:

| Scenario | DBpoweramp Reads |
|----------|------------------|
| File has only Vorbis Comments | Vorbis Comments |
| File has Vorbis Comments + ID3v2 (legacy) | **Vorbis Comments only** (ID3v2 is ignored) |
| File has only ID3v2 (no Vorbis Comments) | ID3v2 (for backward compatibility) |

RFC 9639 — the official FLAC format specification — states: "A FLAC file MUST NOT contain more than one Vorbis comment metadata block." It places no official standing on ID3v2 in FLAC. Xiph's tools (`oggenc`, `opusenc`) complain about ID3v2 in FLAC and recommend removal. DBpoweramp writes only Vorbis Comments.

**Source:** [RFC 9639 — FLAC Format Specification](https://www.rfc-editor.org/rfc/rfc9639.txt), [FLAC Format — Xiph.org](https://xiph.org/flac/format.html)

### 3.4 WAV — ID3v2 Over RIFF INFO

WAV files can hold metadata in three concurrent locations. DBpoweramp's read priority:

| Priority | Tag System | Used For |
|----------|-----------|---------|
| **1st** | ID3v2 chunk (`ID3 `) | All standard ID3v2 frames; APIC for cover art |
| **2nd** | RIFF LIST-INFO | IART, INAM, IPRD, IGNR, ICMT, ICOP, ISFT |
| **3rd** | BWF bext | Description, Originator, BWF-specific fields |

DBpoweramp writes to all three locations when converting to WAV, but on read, ID3v2 is the primary source. RIFF INFO fields are mapped to their ID3v2 equivalents internally.

Note: Some applications (Windows Explorer) read only RIFF INFO. DBpoweramp's dual-write ensures compatibility in both directions.

### 3.5 AIFF — ID3v2 Chunk Only

AIFF files store metadata in an `ID3 ` chunk within the FORM structure. DBpoweramp reads and writes this chunk identically to how it handles ID3v2 in MP3. Native AIFF chunks (ANNO, NAME, AUTH, `(c)`) are not read by DBpoweramp — they are macOS-specific and not part of the standard Windows tagging workflow that DBpoweramp targets.

### 3.6 TTA — The Format with No Universal Standard

TTA (True Audio) has the most ambiguous tagging situation of any lossless format. The official TTA specification supports ID3v1, ID3v2, and APEv2 with **no stated priority**:

| Application | Priority Order |
|-------------|----------------|
| **foobar2000** | APEv2 → ID3v2 → ID3v1 |
| **AIMP** | APEv2 → ID3v2 → ID3v1 |
| **Mp3Tag** | APEv2 (even if empty) → ID3v2 |
| **MusicBrainz Picard** | ID3v2 (ignores APEv2) |
| **DeaDBeeF** | APEv2 → ID3v2 → ID3v1 |

DBpoweramp, using TagLib, treats TTA like other APEv2-capable formats: APEv2 is read first, with ID3v2 and ID3v1 as fallbacks. However, this is not confirmed behavior — TTA support in TagLib is limited.

**Source:** [TTA — Hydrogenaudio Knowledgebase](https://wiki.hydrogenaudio.org/index.php?title=TTA), [TTA — Tau Software (official)](https://tausoft.org/en/tta-%D0%BE%D0%BF%D0%B8%D1%81%D0%B0%D0%BD%D0%B8%D0%B5-%D1%84%D0%BE%D1%80%D0%BC%D0%B0%D1%82%D0%B0/)

### 3.7 WavPack, TAK, APE — APEv2 Only

These three formats use APEv2 as their exclusive tag system. ID3v1 may be present for backward compatibility, but DBpoweramp (via TagLib) ignores it when APEv2 is present. There is no ambiguity: if APEv2 exists, it is the source of truth for all metadata.

---

## 4. Reading Sequence and Merge Strategy

### 4.1 The TagLib Read Pipeline

DBpoweramp's tag reading follows this fixed sequence for every file:

```
1. Detect file format (by extension and magic bytes)
2. Locate all possible tag locations for that format
3. Read tags from highest-priority location first
4. For each field, stop at first non-empty value found
5. Fall back to next-priority location for empty fields
6. Normalize field names to canonical names
7. Return unified metadata object
```

This is implemented in TagLib's format-specific file classes (`MPEG::File`, `FLAC::File`, `MP4::File`, etc.). Each class defines its own `tag()` accessor that applies the format's priority rules.

### 4.2 What "First Match Wins" Means in Practice

The first-match-wins rule operates **per field**, not per tag system:

```
ID3v2.4 TALB  = "Greatest Hits"
ID3v2.3 TALB  = "Old Album Name"
APEv2 TITLE   = "Song Title"
ID3v1 TALB    = "Very Old Name"

DBpoweramp reads:
  TALB  = "Greatest Hits"      (from v2.4, first match)
  TIT2  = "Song Title"        (from APEv2, ID3v2 has no TIT2)
  TCOM  = (empty)              (no source found)
```

This means a field can come from one tag system and a different field from a different tag system. The result is a unified metadata object where each field has exactly one source.

### 4.3 DBpoweramp ID Tag Update Processing Order

When the **ID Tag Update** utility is used (either as a standalone utility or as a DSP effect), the processing order is fixed:

```
1. MAP     — copy/rename fields (e.g., DATE → TYER)
2. DELETE  — remove unwanted fields
3. MANIPULATE — apply rules (capitalization, artist splitting)
4. ADD     — add new fields
```

This means deletions happen **after** mapping, so a mapped field can be deleted. Additions happen **last**, so they always win over any previous value. The ID Tag Update utility is a **clean-slate writer**: it removes existing tags and rewrites fresh ones from the current global configuration.

**Source:** [DBpoweramp ID Tag Update Help](https://dbpoweramp.com/Help/dMC/idtagupdate.html)

### 4.4 Cross-Format Field Mapping on Read

When DBpoweramp reads a file, it normalizes format-specific field names into a canonical internal model:

| Canonical Name | ID3v2 Frame | Vorbis Comment | MP4 Atom | RIFF INFO |
|----------------|-------------|----------------|----------|-----------|
| Title | TIT2 | TITLE | ©nam | INAM |
| Artist | TPE1 | ARTIST | ©ART | IART |
| Album | TALB | ALBUM | ©alb | IPRD |
| Year | TYER / TDRC | DATE | ©day | ICRD |
| Genre | TCON | GENRE | ©gen | IGNR |
| Comment | COMM | COMMENT | ©cmt | ICMT |
| Track | TRCK | TRACKNUMBER | trkn | ITRK |
| Disc | TPOS | DISCNUMBER | disk | — |
| Composer | TCOM | COMPOSER | ©wrt | — |
| Album Artist | TPE2 | ALBUMARTIST | aART | — |
| BPM | TBPM | BPM | tmpo | — |
| Encoder | TENR | ENCODER | ©too | ISFT |
| Cover Art | APIC | COVERART / METADATA_BLOCK_PICTURE | covr | — |

**Source:** [Mp3tag Tag Field Mappings](https://help.mp3tag.de/main_tags.html)

---

## 5. Graceful Degradation and Error Handling

### 5.1 Missing UTF-16 BOM in ID3v2 Frames

When an ID3v2 frame declares encoding `$01` (UTF-16 with BOM) but the actual bytes do not start with a valid BOM (`0xFF 0xFE` for LE or `0xFE 0xFF` for BE):

| Scenario | TagLib Behavior | User Impact |
|----------|----------------|-------------|
| BOM present | Correctly decoded | None |
| BOM missing, bytes look like UTF-16LE | Treated as UTF-16LE | May appear correct |
| BOM missing, ambiguous | Assumes Little-Endian | Possible mojibake |
| Completely invalid | Falls back to ISO-8859-1 | Garbled characters |

ID3v2.4 frame definition states that for encoding `$01` (UTF-16), the text should start with a BOM. If BOM is absent, behavior is undefined. TagLib's default for UTF-16 without BOM is little-endian.

**Source:** [ID3v2.4.0 Frame Definitions — ID3.org](https://id3.org/id3v2.4.0-frames)

### 5.2 Invalid or Unknown Encoding Byte

The valid text encoding bytes in ID3v2 are `$00` (ISO-8859-1), `$01` (UTF-16 with BOM), `$02` (UTF-16BE without BOM), and `$03` (UTF-8). Any other value (`$04`–`$FF`) is undefined.

| Encoding Byte | Expected Behavior | TagLib Fallback |
|-------------|-------------------|----------------|
| `$00` | ISO-8859-1 | — |
| `$01` | UTF-16 with BOM | — |
| `$02` | UTF-16BE without BOM | — |
| `$03` | UTF-8 | — |
| `$04`–`$FF` | Undefined | Treat as ISO-8859-1 or return error |

Per ID3.org guidance: "Your application should never crash because the user does something stupid. Likewise, your application should not crash because of erroneous or malformed data in a tag." ([ID3.org](https://id3.org/id3v2.4.0-frames))

### 5.3 Corrupt or Truncated Tag Data

| Corruption Type | DBpoweramp/TagLib Behavior |
|----------------|--------------------------|
| Truncated frame (size exceeds available bytes) | Frame skipped; parsing continues |
| Invalid frame ID (contains non-printable chars) | Frame skipped (forward compatibility) |
| Frame with zero size | Frame skipped |
| Tag header with invalid synchsafe bytes | Tag considered absent |
| ID3v2.4 footer mistaken for header | Footer skipped (10-byte signature check) |
| Corrupted image data in APIC | Image skipped; fallback to next source |

TagLib's frame-parsing code uses the frame size descriptor to skip over unknown or corrupted frames, ensuring that one corrupt frame does not prevent the rest of the file from being read.

### 5.4 Non-ASCII Characters in Filenames

DBpoweramp on Windows handles non-ASCII filenames via the system's Unicode support. On Linux (via Wine), the Wine translation layer handles UTF-8 filenames. Key edge cases:

| Scenario | Behavior |
|----------|----------|
| Filename with valid UTF-8 | Read correctly |
| Filename with invalid byte sequence | Replaced with `?` or similar placeholder |
| Mixed encodings (e.g., Latin-1 in a UTF-8 context) | Appears garbled; no auto-detection |
| Non-ASCII in path with Unicode normalization differences | May resolve to different files on different OS |

### 5.5 Unknown Frame IDs

ID3v2 frames are forward-compatible by design: readers can safely skip any frame they don't recognize by using the 4-byte size descriptor.

| Frame Prefix | Meaning |
|-------------|---------|
| `A`–`Z`, `0`–`9` | Official or reserved frame |
| `X` | Experimental frame (free to use) |
| `Y` | Experimental frame (free to use) |
| `Z` | Experimental frame (free to use) |
| Lowercase letters | Invalid frame ID (skip) |

DBpoweramp (via TagLib): unknown frames are **preserved** during tag editing operations — TagLib copies unknown frames verbatim when rewriting a tag, preventing data loss.

### 5.6 FLAC METADATA_BLOCK_PICTURE vs Vorbis COVERART

FLAC supports two cover art mechanisms:

| Mechanism | Location | DBpoweramp Read Priority |
|-----------|---------|--------------------------|
| METADATA_BLOCK_PICTURE | FLAC metadata block (native) | **Preferred** |
| COVERART in Vorbis Comments | Inside Vorbis Comment block | Fallback |

METADATA_BLOCK_PICTURE is the RFC 9639-compliant mechanism and is what DBpoweramp writes. COVERART is an older Vorbis comment-based mechanism that some legacy encoders still produce. DBpoweramp reads METADATA_BLOCK_PICTURE first; if absent, falls back to COVERART.

---

## 6. Field-Level Reading Rules

### 6.1 Date Fields: TDRC vs TYER vs TDAT

This is the single most complex field-level reading issue in ID3v2.

| Frame | Version | Format | Notes |
|-------|---------|--------|-------|
| TYER | ID3v2.3 only | `YYYY` (4 digits) | Year only; deprecated in v2.4 |
| TDAT | ID3v2.3 only | `DDMM` (4 digits) | Day and month |
| TIME | ID3v2.3 only | `HHMM` (4 digits) | Hour and minute |
| TDRC | ID3v2.4 only | ISO 8601 | `YYYY`, `YYYY-MM`, or `YYYY-MM-DDTHH:MM:SS` |
| TORY | ID3v2.3 only | `YYYY` | Original release year |
| TDOR | ID3v2.4 only | ISO 8601 | Original release time |

#### 6.1.1 DBpoweramp's Date Reading Strategy

DBpoweramp reads date fields in this priority order:

```
1. TDRC (ID3v2.4) — preferred; contains full timestamp
2. TYER (ID3v2.3) — fallback when TDRC absent
3. TDAT + TIME — merged with TYER to reconstruct date if only TDAT/TIME exist
4. TORY/TDOR — original release date (kept separate from recording date)
```

When writing back to ID3v2.3, DBpoweramp extracts the year from TDRC and writes it to TYER, losing any month/day/hour information that was in the full timestamp.

### 6.2 Multiple TCON Genre Frames

ID3v2.3 and ID3v2.4 differ on how multiple genres should be stored:

| Version | Spec-Compliant Storage | Real-World Behavior |
|---------|----------------------|---------------------|
| ID3v2.3 | Single TCON frame with `(1)(2)` parenthetical notation | Many taggers write multiple TCON frames anyway |
| ID3v2.4 | Multiple TCON frames OR null-byte separated in one frame | Multiple frames widely used |

**DBpoweramp's behavior:** Reads the first TCON frame. Multiple genre values within a single frame are parsed by recognizing parenthetical genre codes. Genre ID references (e.g., `(21)`) are resolved to the ID3v1 genre name ("Folk").

The ID3v1 genre list defines genres 0–79 as standard; 80–255 can be custom genres.

### 6.3 APIC Picture Type Priority

When a file contains multiple APIC frames with different picture types:

| Priority | Picture Type | Byte Code | Notes |
|----------|-------------|-----------|-------|
| 1 (highest) | Cover (front) | `0x03` | Preferred by DBpoweramp and most players |
| 2 | Cover (back) | `0x04` | Used for back cover art |
| 3 | Media | `0x06` | CD/vinyl label side |
| 4 | Lead artist | `0x07` | Soloist/performer |
| 5 | Artist/performer | `0x08` | Secondary artist image |
| 6 | Other | `0x00` | Generic; most naive implementations use this |
| 7–20 | Various | `0x09`–`0x14` | Specialized types |

**DBpoweramp reads type 3 (Cover front) first.** If no type 3 exists, it reads type 0 (Other). This is confirmed in the DBPOWERAMP_CRITICAL_BEHAVIORS_QUICKREF: "DBpoweramp writes type 3 (Front Cover). Players that sort by picture type won't show type 0 as album cover."

Multiple APIC frames of the same picture type are allowed. DBpoweramp reads the first frame for each type.

**Source:** [ID3.org id3v2.3.0](https://id3.org/id3v2.3.0), [DBPOWERAMP_CRITICAL_BEHAVIORS_QUICKREF.md](../.research/DBpoweramp-research/DBPOWERAMP_CRITICAL_BEHAVIORS_QUICKREF.md)

### 6.4 USLT vs SYLT Lyrics

| Frame | Type | Support | DBpoweramp Reads |
|-------|------|---------|----------------|
| USLT | Unsynchronized lyrics (plain text) | Universal | **Yes** — primary source |
| SYLT | Synchronized lyrics (timestamped) | Limited (karaoke apps) | Yes — secondary |

DBpoweramp reads USLT as the primary lyrics source. SYLT is read if present but is not used for standard display. Some specialized karaoke applications use SYLT.

### 6.5 Multiple COMM Frames with Different Languages

ID3v2 allows multiple COMM frames with different language codes and descriptions. DBpoweramp's behavior:

1. Reads the first (or English-language) COMM frame as the primary comment
2. Other language COMM frames are preserved during tag editing
3. When writing, DBpoweramp typically writes a single COMM frame
4. Multi-language comments require manual handling via ID Tag Processing DSP

### 6.6 TALB vs TOAL (Original Album)

| Frame | Version | Purpose |
|-------|---------|---------|
| TALB | Both | Album name |
| TOAL | ID3v2.2/2.3 | Original album (for covers/remixes) |
| TOAL | ID3v2.4 | Deprecated (merged into TALB for original album) |

DBpoweramp reads TALB for the current album. TOAL is read separately as the "original album" field and preserved through conversions when the destination format supports it.

### 6.7 ReplayGain Fields

ReplayGain in MP3 files is typically stored in one of these locations:

| Location | Field Names | DBpoweramp Priority |
|----------|-------------|-------------------|
| ID3v2 TXXX | `TXXX:REPLAYGAIN_TRACK_GAIN`, `TXXX:REPLAYGAIN_ALBUM_GAIN` | Secondary |
| APEv2 | `REPLAYGAIN_TRACK_GAIN`, `REPLAYGAIN_ALBUM_GAIN` | **Primary** (written by Mp3Gain) |

DBpoweramp reads ReplayGain from whichever location has it. If both are present, APEv2 takes priority (consistent with TagLib's APE-read path for non-text fields). ReplayGain processing during conversion requires the "Drop ReplayGain" option to be unchecked in dBpoweramp Configuration → Advanced.

---

## 7. ID3v2 Version Handling on Read

### 7.1 Version Detection Mechanism

ID3v2 tags are identified by the `ID3` magic bytes at offset 0–2, followed by the version bytes:

```
Byte 0-2:  $49 $44 $33         (ASCII "ID3")
Byte 3:    Major version (0x03 = v2.3, 0x04 = v2.4)
Byte 4:    Revision number     (always 0x00 in practice)
Byte 5:    Flags byte
Byte 6-9:  Tag size (synchsafe integer, all bytes < 0x80)
```

| Version | Version Bytes | Frame Header Size Encoding |
|---------|-------------|--------------------------|
| ID3v2.2 | `03 00` | 3-byte frame IDs (`TT2`, `TAL`, etc.) — obsolete |
| ID3v2.3 | `03 00` | 4-byte frame IDs, 32-bit big-endian frame sizes |
| ID3v2.4 | `04 00` | 4-byte frame IDs, 28-bit synchsafe frame sizes |

### 7.2 Synchsafe Integer Decoding

The synchsafe integer is the key parsing difference. Each byte's MSB is forced to 0, giving 7 usable bits per byte:

```
Byte:     [xxxx xxxx]
Max:     $7F $7F $7F $7F
Decoded:  (byte0 << 21) | (byte1 << 14) | (byte2 << 7) | byte3
```

This yields a maximum tag size of 268,435,455 bytes (256 MB) and prevents the byte sequence `0xFF 0x00` from appearing in the tag header (which would be mistaken for an MP3 sync word).

### 7.3 Frame Size Encoding Difference (Critical)

| Version | Frame Size Encoding | Max Frame Size |
|---------|-------------------|----------------|
| ID3v2.3 | 32-bit big-endian integer | 4,294,967,295 bytes |
| ID3v2.4 | 28-bit synchsafe integer | 268,435,455 bytes |

This difference is the primary reason DBpoweramp defaults to ID3v2.3 — it avoids the synchsafe constraint and supports larger frames. Some older devices also have incomplete ID3v2.4 support.

### 7.4 ID3v2.2 (Obsolete)

ID3v2.2 uses 3-byte frame IDs (`TT2` for title, `TAL` for album, `TP1` for artist) and a 3-byte frame size field. DBpoweramp (via TagLib) reads ID3v2.2 tags but converts them to ID3v2.3/2.4 frame IDs on write. This is a one-way conversion.

### 7.5 DBpoweramp's Default ID3v2 Version

DBpoweramp defaults to **ID3v2.3** for maximum compatibility. Users can change this to ID3v2.4 via `dBpoweramp Configuration → Codecs → Advanced Options → mp3 ID Tagging → ID3v2 Version`. The reasons for the default are:

1. Broadest device compatibility (many car stereos and portable players only support v2.3)
2. Avoids synchsafe frame size constraints
3. Avoids potential issues with older software that doesn't understand v2.4 frames

When reading, DBpoweramp handles both versions transparently via TagLib's version-agnostic frame parsing.

---

## 8. Cover Art Reading Priority

### 8.1 Complete Cover Art Read Cascade

DBpoweramp follows this cascade when searching for cover art:

```
1. EMBEDDED: Check APIC frames in ID3v2 (MP3/AIFF)
              → METADATA_BLOCK_PICTURE in FLAC
              → covr atom in M4A
              → COVERART in Vorbis Comments
              → APEv2 picture items

   1a. Within embedded images: filter by picture type
       → Type 3 (Cover front) FIRST
       → Type 0 (Other) SECOND (if no type 3)
       → Type 4 (Cover back) THIRD (for back cover)
       → Other types as found

2. EXTERNAL FOLDER: If no embedded art found
   Search in the same directory as the audio file:
   → folder.jpg      (highest priority)
   → folder.png
   → cover.jpg
   → cover.png
   → front.jpg
   → front.png
   → album.jpg
   → album.png
   → art.jpg
   → art.png

3. NO ART: If neither embedded nor folder art exists
   → No cover art associated with this file
```

### 8.2 Embedded Art Validation

Before using embedded art, DBpoweramp validates:

1. **Magic bytes:** JPEG must start with `0xFF 0xD8`, PNG must start with the PNG signature
2. **File size:** Art larger than the configured maximum is rejected (configurable in ID Tag Update codec)
3. **Dimensions:** Art with zero dimensions or excessively large dimensions may be rejected
4. **Format:** Preferred JPEG over PNG for maximum compatibility; PNG accepted but may be converted on write

### 8.3 Multiple Embedded Images

When a file contains multiple embedded images (common in files tagged by MusicBrainz Picard):

1. DBpoweramp reads **all embedded images** into its internal metadata model
2. During conversion, the pipeline writes **one** image (the front cover) to the output file
3. The "ID Tag Update" codec can be configured to export all art or import from `folder.jpg`

MusicBrainz Picard's default behavior embeds **all** images. DBpoweramp's default behavior for conversions is to write only one image. This can result in image count reduction during conversion.

---

## 9. User-Facing Behavior Summary

### 9.1 Would a User Notice Any Difference from DBpoweramp?

**Yes, in these specific scenarios:**

| Scenario | What Happens | User Notice |
|----------|-------------|-------------|
| File with APEv2 ReplayGain + ID3v2 text | DBpoweramp reads text from ID3v2, ReplayGain from APEv2 — both fields work | **No difference** — correct behavior |
| File with empty APEv2 overwriting ID3v2 text | Does NOT happen in DBpoweramp — ID3v2 always wins for text | **No difference** — DBpoweramp immune |
| FLAC with legacy ID3v2 at start | ID3v2 ignored; Vorbis Comments used | **Minor** — if Vorbis Comments are empty and only ID3v2 has data, data appears missing |
| WAV with RIFF INFO + ID3v2 | ID3v2 wins; RIFF INFO fields shown | **No difference** for most users |
| MP3 with ID3v2.3 + ID3v2.4 | ID3v2.4 wins; v2.3 ignored | **No difference** — higher version is newer |
| TTA file with APEv2 + ID3v2 | APEv2 wins (TagLib priority) | **Potential difference** from Picard (which uses ID3v2) |
| WMA with ASF metadata objects | All metadata merged | **No difference** — flat collection |
| AIFF with native chunks + ID3v2 | ID3v2 wins; native chunks ignored | **Potential** for macOS-only metadata to disappear |

### 9.2 Summary Table: DBpoweramp vs Other Tools

| Tool | MP3 Priority | FLAC Priority | APE-in-MP3 Behavior |
|------|-------------|--------------|---------------------|
| **DBpoweramp** | ID3v2 → APEv2 → ID3v1 | Vorbis Comments only | APE for ReplayGain only |
| **Mp3tag** | APEv2 → ID3v2 → ID3v1 | Vorbis Comments only | APE can override ID3v2 text |
| **foobar2000** | ID3v2 → APEv2 → ID3v1 | Vorbis Comments only | APE for extensions |
| **MusicBrainz Picard** | ID3v2 → APEv2 → ID3v1 | Vorbis Comments only | Stripped on write; APE ignored in TTA |
| **Windows Explorer** | ID3v1 (tagbar) | N/A | N/A |
| **iTunes** | ID3v2 only | Vorbis → ID3v2 conversion | N/A |

---

## Sources

1. [ID3.org id3v2.4.0 Structure](https://id3.org/id3v2.4.0-structure) — Official ID3v2.4 tag structure specification
2. [ID3.org id3v2.3.0 Specification](https://id3.org/id3v2.3.0) — Official ID3v2.3 specification
3. [ID3.org id3v2.4.0 Frames](https://id3.org/id3v2.4.0-frames) — Frame definitions for v2.4
4. [ID3.org id3v2.4.0 Changes](https://id3.org/id3v2.4.0-changes) — v2.3 to v2.4 migration guide
5. [TagLib API Documentation](https://taglib.org/api/) — TagLib C++ API reference
6. [RFC 9639 — FLAC Format Specification](https://www.rfc-editor.org/rfc/rfc9639.txt) — Official FLAC specification
7. [Hydrogenaudio — ID3v2 vs APEv2](https://hydrogenaudio.org/index.php?msg=637031) — Tag priority discussion
8. [music-metadata library — MetadataCollector.ts](https://github.com/Borewit/music-metadata/blob/feb15be8bd8da14ab10cca4c1068be2408fa7b1e/lib/common/MetadataCollector.ts#L17) — Tag priority implementation
9. [Mp3tag Tag Field Mappings](https://help.mp3tag.de/main_tags.html) — Comprehensive field mapping across all formats
10. [DBpoweramp ID Tag Update Help](https://dbpoweramp.com/Help/dMC/idtagupdate.html) — DBpoweramp ID Tag Update documentation
11. [DBpoweramp Forum — ID3v1 30-char truncation](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/30115-id-tags-in-mp3-files-truncated-at-30-characters) — ID3v1 truncation behavior
12. [DBpoweramp Forum — WAV ID3 tags](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/327194-id3-tags-in-wav) — WAV tag chunk behavior
13. [TTA — Hydrogenaudio Knowledgebase](https://wiki.hydrogenaudio.org/index.php?title=TTA) — TTA format documentation
14. [Xiph.org FLAC format](https://xiph.org/flac/format.html) — FLAC format overview
15. [Mutagen Specifications — APEv2](https://mutagen-specs.readthedocs.io/en/latest/apev2/apev2.html) — APEv2 tag format
16. [WavPack Documentation](https://www.wavpack.com/wavpack_doc.html) — WavPack tag behavior
17. [MP4 — Mutagen Documentation](https://mutagen.readthedocs.io/en/latest/api/mp4.html) — MP4 tag handling
18. [eyeD3 Compliance Documentation](https://eyed3.readthedocs.io/en/latest/compliance.html) — ID3v2 version compliance
19. [beets fetchart plugin](https://beets.readthedocs.io/en/stable/plugins/fetchart.html) — Folder image search priority
20. [DBPOWERAMP_CRITICAL_BEHAVIORS_QUICKREF.md](../.research/DBpoweramp-research/DBPOWERAMP_CRITICAL_BEHAVIORS_QUICKREF.md) — Critical behavioral facts
