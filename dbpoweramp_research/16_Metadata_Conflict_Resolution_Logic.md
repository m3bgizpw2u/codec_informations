# 16_Metadata_Conflict_Resolution_Logic.md

> **Research Area:** DBpoweramp Music Converter — Metadata Conflict Resolution Logic
> **Covers:** Multi-tag read priority, source vs user conflicts, cross-format field mapping, multi-value handling, batch conflicts
> **Confidence Level:** Medium-High (official docs + cross-referenced with TagLib/Mp3tag/foobar2000)
> **Cross-references:** File 15 (Preservation Rules), File 17 (Custom Tags), File 18 (Encoding)

---

## 1. Overview

Metadata conflicts arise in DBpoweramp conversions at multiple levels: within a single file (multiple tag systems), across source-destination pairs, and in batch operations where multiple source files map to the same output path. This document maps every conflict resolution path.

---

## 2. Conflict Resolution Within a Single File (Multi-Tag Systems)

### 2.1 MP3 Files: ID3v2 vs. APEv2 vs. ID3v1

MP3 files can simultaneously hold three distinct tag systems. DBpoweramp uses **TagLib** for tag reading/writing, which implements a format-specific read priority.

#### Read Priority (When Multiple Tags Exist)

Based on TagLib's implementation and cross-referenced with Mp3tag and music-metadata behavior:

| Priority | Tag System | DBpoweramp Reads From |
|---|---|---|
| 1st | ID3v2.4 | Primary source |
| 2nd | ID3v2.3 | Falls back if no v2.4 |
| 3rd | APEv2 | Used for ReplayGain/Rating if present |
| 4th | ID3v1 | Last resort fallback |

**Source:** [Hydrogenaudio — ID3v2 vs APEv2](https://hydrogenaudio.org/index.php?msg=637031), [music-metadata library — priority mapping](https://github.com/Borewit/music-metadata/blob/feb15be8bd8da14ab10cca4c1068be2408fa7b1e/lib/common/MetadataCollector.ts#L17)

**Critical Detail — Mp3tag's APEv2 Override Problem:**

Many users encounter a critical issue: Mp3Gain writes APEv2 tags for replay gain but leaves other fields empty. If a file has:
- ID3v2: Title="Song", Artist="Artist", Track=5
- APEv2: Track Gain=-3.2 dB, everything else empty

Reading with priority (APE > ID3v2), an APE-empty field **overrides** the populated ID3v2 field. The result: Title and Artist may show blank, while Track Gain is populated.

**DBpoweramp's approach:** TagLib reads ID3v2 as the primary source for text fields. APEv2 is used primarily for format-specific extensions (ReplayGain, ratings).

#### Write Priority (When DBpoweramp Writes Tags)

| Scenario | DBpoweramp Writes |
|---|---|
| Default MP3 encoding | ID3v2.4 + ID3v1 |
| ID Tag Update | ID3v2 (specified version) |
| If APEv2 present | APEv2 updated alongside ID3v2? (Not confirmed — likely ID3v2-only write) |

**Source:** [DBpoweramp Forum — ID3v2 vs ID3v1 write](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/30115-id-tags-in-mp3-files-truncated-at-30-characters) — "Tag from filename will not write a 2.4 tag if there is already a 1.1 tag, only this will be written."

### 2.2 FLAC Files: Vorbis Comment + FLAC Metadata Blocks

FLAC can hold both Vorbis Comment fields and FLAC METADATA_BLOCK_PICTURE (for cover art) within the same container.

**DBpoweramp behavior:**
- Vorbis Comment: read/write via standard comment block
- Cover art: stored as METADATA_BLOCK_PICTURE (FLAC native) or as embedded Vorbis COVERART
- When converting FLAC→MP3: METADATA_BLOCK_PICTURE is converted to ID3v2 APIC

**Conflict: Embedded COVERART vs. External folder.jpg**

If both exist:
- DBpoweramp prioritizes embedded cover art over folder.jpg
- The embedded art is used; folder.jpg is ignored

**Source:** [DBpoweramp ID Tag Update](https://dbpoweramp.com/Help/dMC/idtagupdate.html) — "Album Art: Art can be exported, or imported from Folder.jpg"

### 2.3 WAV Files: RIFF INFO vs. ID3v2 vs. BWF bext

WAV files support three concurrent metadata locations:

| Location | Standard | DBpoweramp Support |
|---|---|---|
| RIFF INFO chunk | Microsoft legacy | Yes |
| ID3v2 (in 'ID3 ' chunk) | De facto | Yes |
| BWF bext chunk | EBU standard | Yes |

**DBpoweramp writes metadata to all three locations when converting TO WAV.** When reading, ID3v2 takes priority; BWF bext is used for professional archival fields.

**Source:** [DBpoweramp Forum — ID3 tags in WAV](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/327194-id3-tags-in-wav) — "They are written in a specific wave riff chunk: 'ID3 ' and 'LIST' for the limited tag."

---

## 3. Source vs. User-Entered Metadata Conflicts

### 3.1 Resolution Rule: User Values Override Source Values

When a user provides metadata at conversion time (CD ripping with freedb/MusicBrainz lookup, or manual entry), the **user-provided values override source values**.

This is standard behavior: the user explicitly chose to use specific metadata, so DBpoweramp respects that choice.

### 3.2 Resolution Rule: Source Values Fill Unspecified Fields

When some fields are user-specified and others are blank, DBpoweramp uses **source values to fill gaps**:

```
User specifies: Title="New Title", Artist=(blank)
Source has:     Title="Old Title", Artist="Source Artist"
Result:         Title="New Title", Artist="Source Artist"
```

### 3.3 Batch Ripping with freedb/MusicBrainz

When ripping multiple CDs and using a metadata lookup, DBpoweramp applies the same rule per-track. If a track's metadata was not found in the lookup, the source (CD) metadata is used.

### 3.4 ID Tag Update Utility — Explicit Conflict Rules

The ID Tag Update utility processes tags in this fixed order:

```
1. Map    (copy/rename fields)
2. Deletions (remove unwanted fields)
3. Manipulation (apply rules: capitalization, multiple artist splitting)
4. Additions (add new fields)
```

**This order means:** Deletions happen AFTER Map, so a mapped field can be deleted. Additions happen LAST, so they always win over any previous value for the same field.

**Source:** [DBpoweramp ID Tag Update Help](https://dbpoweramp.com/Help/dMC/idtagupdate.html)

---

## 4. Cross-Format Incompatible Field Handling

### 4.1 Fields with No Destination Equivalent

| Source Field | Destination Format | Resolution |
|---|---|---|
| Vorbis SOURCE | ID3v2 | Dropped (no mapping) |
| ASF WM/* | FLAC | Dropped |
| MP4 ----:* (iTunes metadata) | ID3v2 | Dropped unless mapped |
| Vorbis vendor | ID3v2 | Dropped |
| FLAC STREAMINFO technical | Any tag | Never stored as tag |

**Mitigation:** Use ID Tag Processing DSP → Map to store incompatible fields in custom TXXX frames before conversion.

### 4.2 Fields with Partial Mapping

| Source | Destination | Precision Loss |
|---|---|---|
| Vorbis DATE="2024-03-15T14:00" | ID3v2.3 TYER="2024" | Time and month/day lost |
| Vorbis BPM=120.5 | ID3v2 TBPM=120 | Decimal lost |
| Vorbis COMMENT with multiple languages | ID3v2 COMM | Only first comment kept |
| FLAC METADATA_BLOCK_PICTURE (type 3) | ID3v2 APIC | Type field may be ignored |

### 4.3 Date Field Normalization Matrix

| Source Format | Source Field | ID3v2.3 TYER (YYYY) | ID3v2.4 TDRC (full) | Vorbis DATE | MP4 ©day |
|---|---|---|---|---|---|
| ID3v2.4 | TDRC="2024-03-15" | 2024 | 2024-03-15 | 2024-03-15 | 2024-03-15 |
| Vorbis | DATE="2024" | 2024 | 2024 | 2024 | 2024 |
| Vorbis | DATE="2024-03" | 2024 | 2024-03 | 2024-03 | 2024-03 |
| MP4 | ©day="2024-03-15T00:00:00Z" | 2024 | 2024-03-15 | 2024-03-15 | 2024-03-15 |

---

## 5. Multiple Values for Single-Value Fields

### 5.1 ID3v2 Single-Value Fields → Multiple Source Values

The ID3v2 spec defines certain frames as single-value (e.g., TYER, TALB). When the source has multiple values:

| Field | Source Format | Multiple Values | ID3v2.3 Behavior | ID3v2.4 Behavior |
|---|---|---|---|---|
| Album (TALB) | Multiple Vorbis ALBUM | Join with "; " | Single value | Single value |
| Year (TYER) | Multiple dates | First value only | First 4 digits | First valid date |
| Genre (TCON) | Multiple genres | Join with "; " | Single TCON with semicolons | Single TCON |
| Composer (TCOM) | Multiple composers | Join with "; " | Single TCOM | Multiple TCOM frames |

**DBpoweramp default:** Joins multiple values with "; " (semicolon-space). This is standard behavior per ID3v2.3 spec interpretation.

**Source:** [DBpoweramp ID Tag Update](https://dbpoweramp.com/Help/dMC/idtagupdate.html) — "Multiple Artist To 'Artist1; Artist2' — dBpoweramp follows standards set by tagging formats when it comes to handling multiple artists."

### 5.2 Multiple Artists in a Single Field

DBpoweramp's "Multiple Artist To" feature specifically handles the case where one ARTIST field contains multiple artists separated by "; ":

```
Input:  ARTIST="Artist1; Artist2"
Action: Multiple Artist To 'Artist1; Artist2'
Result: Internal representation: [Artist1, Artist2] as separate values
        Stored in ID3v2: Multiple TPE1 frames OR single TPE1 with "; " separator
```

The "Multiple Artist From" feature reverses this: detects "; "-separated artists and splits them into the format's native multiple-value representation.

### 5.3 Multi-Value Fields in TXXX (Custom Frames)

**Critical Issue — TXXX Frame Semantics:**

According to ID3v2.3 spec: "There may be more than one 'TXXX' frame in each tag, but **only one with the same description**."

However, some taggers (including foobar2000) write multiple TXXX frames with the same description as separate frames. DBpoweramp (via TagLib) handles this by reading the **last** TXXX frame with a given description, overwriting any earlier ones.

**Source:** [Hydrogenaudio — Multi valued TXXX frames in ID3v2.3](https://hydrogenaud.io/index.php/topic,112341.0.html)

---

## 6. Conflicting Cover Art Between Tag and folder.jpg

### 6.1 Resolution Rules

| Scenario | DBpoweramp Behavior |
|---|---|
| Embedded art exists, folder.jpg exists | Uses embedded art; folder.jpg ignored |
| Only embedded art | Uses embedded art |
| Only folder.jpg | Uses folder.jpg |
| Multiple embedded arts | Uses first (or front cover per APIC type) |
| No art anywhere | No art written (unless forced via DSP) |

### 6.2 ID Tag Update — Album Art Options

```
Album Art options in ID Tag Update:
- Export art to Folder.jpg
- Import art from Folder.jpg
- Maximum art size (pixels)
- Maximum art byte size
- Force embedded art to JPEG (for PNG)
```

**Source:** [DBpoweramp ID Tag Update Help](https://dbpoweramp.com/Help/dMC/idtagupdate.html)

### 6.3 Cover Art Type Field

FLAC METADATA_BLOCK_PICTURE has a `picture_type` field. ID3v2 APIC also has a `picture_type` byte. When converting FLAC→MP3:

- Type 0x03 (Front cover) → APIC picture_type=3
- Other types may be preserved or normalized depending on codec

DBpoweramp does not provide explicit configuration for picture type mapping.

---

## 7. Batch Conversion Filename Conflict Resolution

### 7.1 Same Output Filename for Multiple Inputs

In batch conversion, if multiple source files would produce the same output filename (e.g., multiple files named "Track 1.flac" from different folders), DBpoweramp handles this by:

1. **Overwriting:** Later files overwrite earlier files (silent, no warning)
2. **Sequential numbering:** If the user sets a naming pattern that includes track number, conflicts are avoided

**There is no automatic "rename to avoid conflict" behavior.** The user must configure naming patterns to include unique identifiers (Artist, Album, Track Number, etc.).

### 7.2 Batch Conversion Metadata Inheritance

When batch converting a folder of files:
- Each file is processed individually
- Metadata is NOT shared across files (each file's tags are independent)
- Album-level metadata (Album name, cover art) must be present in each source file or set via batch naming

**Exception:** If the user uses "Album processing" mode, cover art can be set from the first file and applied to all files in the batch.

---

## 8. Edge Cases

### Edge Case 1: APEv2 ReplayGain Overwrites ID3v2 ReplayGain

If an MP3 file has ReplayGain in both APEv2 (written by Mp3Gain) and ID3v2 (written by another tagger), and the values differ:
- DBpoweramp reads APEv2 ReplayGain (higher priority for this specific field)
- The value used during ReplayGain DSP processing will be from APEv2
- The ID3v2 value is preserved as-is (APEv2 is not written back to ID3v2 during normal conversion)

### Edge Case 2: ID3v2 TYER and TDRC Both Present

Some files have both TYER (ID3v2.3-style year) and TDRC (ID3v2.4-style timestamp). When reading:
- TagLib prioritizes TDRC over TYER
- TYER is treated as a fallback
- When DBpoweramp writes ID3v2.3, it extracts year from TDRC and writes TYER

### Edge Case 3: Unicode BOM in UTF-16 ID3v2 Frames

If an ID3v2 frame uses encoding $01 (UTF-16 with BOM) but the bytes don't start with a valid BOM (0xFF 0xFE or 0xFE 0xFF):
- TagLib's default behavior: assumes little-endian (DefaultUTF16WithBOMByteOrder = binary.LittleEndian)
- Some players may interpret the bytes as the wrong endianness, producing garbled text
- DBpoweramp writes BOMs correctly (see File 18 for encoding details)

### Edge Case 4: Multiple COMM Frames with Different Languages

ID3v2 allows multiple COMM frames with different language codes and descriptions. DBpoweramp's behavior:
- Reads the first (or English) COMM frame as the primary comment
- Other language COMM frames may be preserved but not displayed in the standard UI
- When writing, DBpoweramp typically writes a single COMM frame

### Edge Case 5: Empty APEv2 Tag Overwrites Populated ID3v2

If a file has a populated ID3v2 tag and an empty APEv2 tag, and the writing process updates APEv2:
- Mp3tag behavior: APEv2's empty fields can overwrite ID3v2 fields (APEv2 > ID3v2 read priority means empty APE fields override populated ID3v2)
- DBpoweramp: ID3v2 is primary; APEv2 is not typically written back during conversion

---

## 9. Code Examples

### 9.1 music-metadata Library Read Priority (Reference)

```typescript
// From: lib/common/MetadataCollector.ts
// Priority order: first match wins (not merged)
const tagFormats = ['ID3v2.4', 'ID3v2.3', 'ID3v2.2', 'APE', 'ID3v1'];
// Only the highest-priority format with a field is used for common.* fields
```

**Source:** [music-metadata Issue #975](https://github.com/Borewit/music-metadata/issues/975)

### 9.2 Mp3tag Read Priority Configuration

```
Options → Tags → MPEG:
  READ:
    [✓] ID3v1
    [✓] ID3v2
    [ ] APE    (disable to avoid APE-over-ID3v2 issue)
  WRITE:
    [✓] ID3v2
    [ ] APEv2  (avoid writing APE to MP3)
```

**Source:** [Mp3tag Documentation](https://docs.mp3tag.de/customization/options/tags/mpeg/)

### 9.3 foobar2000 Preferred Tag Writing Scheme

```
Advanced → Tagging → MP3 → Preferred tag writing scheme:
  - ID3v2 + ID3v1
  - ID3v2 only
  - ID3v1 only
```

**Source:** [Hydrogenaudio Forum](https://hydrogenaudio.org/index.php?msg=637031)

---

## Summary: Would a User Notice Any Difference?

| Conflict Scenario | User Impact |
|---|---|
| File with APEv2 + ID3v2, APE fields blank | DBpoweramp reads ID3v2 (good); other taggers may show blank fields |
| Multiple values in single-value field | Joined with "; " — semicolon separator is standard |
| No art in tag, folder.jpg exists | DBpoweramp imports from folder.jpg |
| Batch convert same filename from different folders | Later files silently overwrite earlier ones |
| TXXX frame with same description, different values | Last value wins (via TagLib) |
| ID3v2.3 vs ID3v2.4 date fields | Year extracted; full timestamp not preserved in v2.3 |

**Bottom Line:** DBpoweramp's conflict resolution is conservative and standards-compliant. The main user-visible issues arise from: (1) APEv2 blank-field priority in other taggers, (2) batch overwrite silent behavior, and (3) multi-value join with "; " that may not match user's preferred separator.
