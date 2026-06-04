# DBpoweramp Music Converter — WAV/AIFF RIFF/BWF Tag Behavior
*Generated: 2026-06-04 | Format: WAV, AIFF, BWF | Confidence: High*

---

## 1. Executive Summary

WAV files use the RIFF container format with two distinct metadata systems: the legacy **RIFF INFO** chunk list (using four-character codes like `INAM`, `IART`, `ICRD`) and the professional **BWF (Broadcast Wave Format)** extension with its `bext` (broadcast extension) chunk containing production metadata. AIFF uses a similar chunk-based structure with `ANNO` and `COMM`-adjacent ID3v2 storage. DBpoweramp's WAV tag handling must navigate these overlapping systems carefully: RIFF INFO is the most portable, BWF bext is for archival/professional use, and ID3v2 in WAV is a non-standard but widespread convention that may conflict with RIFF INFO.

---

## 2. Format Overview

### WAV / RIFF Structure

```
RIFF (container)
  └─ WAVE (format type)
      ├─ fmt  (format chunk — audio encoding parameters)
      ├─ data (audio samples)
      ├─ LIST 'INFO' (RIFF INFO metadata)
      │    ├─ INAM (Title)
      │    ├─ IART (Artist)
      │    ├─ ICMT (Comment)
      │    ├─ ICRD (Creation Date)
      │    ├─ IGNR (Genre)
      │    ├─ ICOP (Copyright)
      │    ├─ ISFT (Software/Encoder)
      │    ├─ IPRD (Product)
      │    └─ ... (other INFO fields)
      └─ bext (BWF broadcast extension — optional)
           ├─ Description (256 chars)
           ├─ Originator (32 chars)
           ├─ OriginatorReference (32 chars)
           ├─ OriginationDate (YYYY-MM-DD)
           ├─ OriginationTime (HH-MM-SS)
           ├─ TimeReference (48-bit sample offset)
           └─ CodingHistory (unlimited)
```

### AIFF Structure

```
FORM (container)
  └─ AIFF (format type)
      ├─ COMM (common chunk — audio parameters)
      ├─ SSND (sound data chunk)
      ├─ ANNO (annotation — text comments)
      └─ ID3  (ID3v2 tag — non-standard but common)
```

**Critical ordering rule**: In both RIFF/WAV and AIFF, the `fmt`/`COMM` chunk **must** come before the `data` chunk. Metadata chunks typically follow the format/data but can appear in various positions depending on the writer.

---

## 3. Tag Reading Behavior

### RIFF INFO Fields

Standard RIFF INFO four-character codes (per the Microsoft/IBM RIFF specification):

| Code | Field Name | Description | Max Length |
|------|-----------|-------------|-----------|
| `INAM` | Name/Title | Track title | 256 chars |
| `IART` | Artist | Artist/creator | 256 chars |
| `ICRD` | Creation Date | Date of creation | 256 chars |
| `IGNR` | Genre | Genre description | 256 chars |
| `ICMT` | Comment | General comments | 256 chars |
| `ICOP` | Copyright | Copyright notice | 256 chars |
| `ISFT` | Software | Encoder/software | 256 chars |
| `IPRD` | Product/Album | Product name or album | 256 chars |
| `IENG` | Engineer | Recording engineer | 256 chars |
| `ISBJ` | Subject | Subject description | 256 chars |
| `IMED` | Medium | Original medium | 256 chars |
| `IKEY` | Keywords | Keywords | 256 chars |
| `IASD` | Artistic Credit | Artistic credit | 256 chars |
| `IRTD` | Rating | Content rating | 256 chars |
| `ISHP` | Sharpness | Post-processing | 256 chars |
| `IWAX` | Wax | Cataloging info | 256 chars |
| `IMCU` | MCU | MCU info | 256 chars |

**Source:** [TheAudioArchive — BWF Metadata](http://www.theaudioarchive.com/TAA_Resources_Metadata.htm)

### BWF bext Chunk

The BWF extension (EBU Tech 3285) adds a `bext` chunk with archival-grade metadata:

| Field | Length | Format |
|-------|--------|--------|
| `Description` | 256 bytes | ASCII, null-padded |
| `Originator` | 32 bytes | ASCII, null-padded |
| `OriginatorReference` | 32 bytes | ASCII, null-padded |
| `OriginationDate` | 10 bytes | `YYYY-MM-DD` format |
| `OriginationTime` | 8 bytes | `HH-MM-SS` format |
| `TimeReference` | 48 bits (8 bytes) | Sample offset to start of audio |
| `Version` | 2 bytes | BWF version |
| `UMID` | 64 bytes | Unique Material Identifier (optional) |
| `LoudnessValue` | 4 bytes | Per EBU R128 (BWF v2) |
| `LoudnessRange` | 4 bytes | Per EBU R128 (BWF v2) |
| `MaxTruePeakLevel` | 2 bytes | Per EBU R128 (BWF v2) |
| `MaxMomentaryLoudness` | 2 bytes | Per EBU R128 (BWF v2) |
| `MaxShortTermLoudness` | 2 bytes | Per EBU R128 (BWF v2) |
| `CodingHistory` | Variable | ASCII, CR/LF separated |

**BWF v2 additions** (EBU Tech 3285 v2, 2011): Loudness metadata fields per EBU R 128 standard.

**Source:** [MediaArea — BWF MetaEdit BEXT Information](https://mediaarea.net/BWFMetaEdit/bext), [FADGI — Embedding Metadata in BWF](https://www.digitizationguidelines.gov/guidelines/digitize-embedding.html)

### ID3v2 in WAV (Non-Standard)

Some tools embed an ID3v2 tag in WAV files, typically:
- At the very start of the file (before the `RIFF` header)
- In an `ID3 ` chunk within the RIFF structure
- In the `data` chunk as a prelude

**This is non-standard** but widely used by tag editors and rippers. DBpoweramp:
- Can read ID3v2 from WAV if present.
- May strip or preserve ID3v2 depending on conversion settings.
- Does not write ID3v2 to WAV by default.

### AIFF ID3v2 Storage

AIFF files commonly store ID3v2 metadata in:
- An `ANNO` chunk (annotation text, multiple chunks allowed)
- A `ID3 ` chunk (official but non-standard extension)

ID3v2 in AIFF supports the full frame set, making AIFF+ID3v2 functionally equivalent to MP3+ID3v2 for tag purposes.

---

## 4. Tag Writing Behavior

### Multi-Tag WAV Files

WAV files can simultaneously contain RIFF INFO, BWF bext, and ID3v2 metadata. DBpoweramp's tag writing strategy:

1. **Primary**: RIFF INFO (`LIST 'INFO'`) — most portable and compatible
2. **Secondary**: BWF bext if configured for professional output
3. **Stripped**: ID3v2 (unless explicitly configured to preserve)

**Conflict resolution**: When multiple tag systems exist, DBpoweramp reads RIFF INFO as primary and may ignore or merge bext/ID3v2 fields.

### RIFF INFO Field Format

- **Encoding**: ASCII for all standard fields. Non-ASCII characters may not display correctly in all players.
- **Length limits**: 256 characters per field (soft limit enforced by some writers).
- **Null padding**: Fields shorter than max length are null-padded, not space-padded.
- **Multiple values**: Not natively supported. Multiple artists require delimiter use (e.g., "/").

### AIFF Chunk Ordering

AIFF requires `COMM` before `SSND` (sound data). Metadata chunks (`ANNO`, ID3) can appear before, after, or interleaved. DBpoweramp writes them after `SSND` to maintain structural validity.

### fmt Chunk Ordering Requirement

In WAV, the `fmt ` chunk should come before `data`. DBpoweramp maintains this order during all write operations.

---

## 5. Normalization Rules

| Rule | Behavior |
|------|----------|
| **Field mapping** | `TITLE` → `INAM`, `ARTIST` → `IART`, `DATE` → `ICRD`, etc. |
| **Encoding** | ASCII only for RIFF INFO; BWF bext also ASCII; ID3v2 in WAV uses ID3 encoding |
| **Case** | RIFF INFO codes are uppercase; values are case-preserved |
| **Date format** | `ICRD` / `OriginationDate` uses `YYYY-MM-DD` or freeform text |
| **BWF coding history** | Each entry on new line: `PCM, 48000Hz, 24-bit, Stereo` |

---

## 6. Edge Cases

1. **BWF `OriginationDate` separator**: The spec allows hyphen, underscore, colon, space, or period as date separator. DBpoweramp may write ISO 8601 (`2024-06-04`) while some professional tools write `YYYYMMDD` without separators.

2. **`TimeReference` and sample rate**: The 48-bit `TimeReference` field stores sample offset. To convert to time, multiply by `(1 / sample_rate)`. If the WAV's sample rate differs from the bext's reference rate, calculations become incorrect.

3. **BWF v2 loudness fields**: Files with BWF v2 loudness metadata (LoudnessValue, LoudnessRange, etc.) may not be preserved by tools that only understand BWF v1. DBpoweramp may strip v2-specific fields.

4. **ID3v2 + RIFF INFO in same file**: Some rippers write both. Reading priority is ambiguous. DBpoweramp typically reads RIFF INFO, ignoring the embedded ID3v2 unless configured otherwise.

5. **AIFF `ANNO` chunk conflicts**: Multiple `ANNO` chunks are allowed. If field values exceed 256 chars, some tools write multiple `ANNO` chunks. DBpoweramp may consolidate these or truncate.

---

## 7. DBpoweramp-Specific Behavior

- **Default output**: Writes RIFF INFO (`LIST 'INFO'`) for standard WAV output.
- **BWF output**: Available as a separate output profile. Writes BWF bext with appropriate fields.
- **Cover art**: WAV has no standard cover art mechanism. DBpoweramp may skip cover art or embed it in a non-standard chunk.
- **ID3v2**: Not written to WAV. Stripped if present in source (unless preservation is configured).
- **BWF v2 loudness**: Not written natively. Professional BWF tools required for loudness metadata.
- **AIFF**: Not a primary output format. If produced, uses `ANNO` chunks for metadata.

---

## 8. Verification Checklist

- [ ] `INAM` field shows title (not blank or garbled ASCII)
- [ ] `IART` field shows artist
- [ ] `ICRD` date format is consistent (YYYY-MM-DD or source format)
- [ ] BWF `bext` fields present if configured for BWF output
- [ ] `CodingHistory` shows encoder info (e.g., "dBpoweramp")
- [ ] `fmt ` chunk precedes `data` chunk (structural validity)
- [ ] No ID3v2 remnants if configured to strip (check start of file)
- [ ] No duplicate RIFF INFO fields (multiple IART chunks may exist)

---

## 9. Sources

1. [TheAudioArchive — BWF Metadata Resources](http://www.theaudioarchive.com/TAA_Resources_Metadata.htm)
2. [Federal Agencies Digital Guidelines Initiative — Embedding Metadata in BWF](https://www.digitizationguidelines.gov/guidelines/digitize-embedding.html)
3. [Library of Congress — Broadcast WAVE Audio File Format, Version 1](https://www.loc.gov/preservation/digital/formats/fdd/fdd000356.shtml)
4. [MediaArea — BWF MetaEdit BEXT Audio Metadata Information](https://mediaarea.net/BWFMetaEdit/bext)
5. [Fast.io — How to Extract Metadata from FLAC, WAV, Lossless Audio](https://fast.io/resources/metadata-extraction-from-flac-wav-lossless-audio/)
