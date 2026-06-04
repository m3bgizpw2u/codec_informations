# DBpoweramp Music Converter — ID3v2 Tag Behavior Deep Dive
*Generated: 2026-06-04 | Format: MP3 | Confidence: High*

---

## 1. Executive Summary

DBpoweramp's handling of MP3 ID3 tags is governed by two competing standards: ID3v2.3 (1999) and ID3v2.4 (2000). The most consequential difference for users is the date frame migration: v2.3 splits date information across `TYER` (year), `TDAT` (date), and `TIME` (time) frames, while v2.4 consolidates these into `TDRC` using ISO 8601 timestamps. DBpoweramp defaults to v2.3 for broad device compatibility but can write v2.4. Tag read/write priority is ID3v2 > APEv2 > ID3v1. Unsynchronization (0xFF 0xE0 escaping) is handled transparently. Multi-value fields like `TPE1` (artist) can be stored as multiple frames or slash-delimited strings depending on the writer. Custom fields via `TXXX` and `COMM` follow well-defined sub-structures.

---

## 2. Format Overview

MP3 files use the ID3v2 container (prepended to the file) for metadata. A complete ID3v2 tag consists of:

| Component | Description |
|-----------|-------------|
| Header (10 bytes) | Version (2 bytes), flags (1 byte), size (4 bytes synchsafe) |
| Frames | Each frame: frame ID (4 bytes), size (4 bytes), flags (2 bytes) |
| Padding | Optional null bytes to allow tag growth without rewriting file |
| Footer (optional, v2.4 only) | Duplicate of header at end of tag |

**Frame size encoding difference (critical):**
- **ID3v2.3**: Frame size is a plain 32-bit unsigned integer (can hold values up to 2^32-1).
- **ID3v2.4**: Frame size is a "synchsafe" integer (only 28 bits usable, max 268,435,455). Each byte has bit 7 forced to 0, making values like 0xFF 0x00 illegal (which prevents false MP3 frame sync patterns).

---

## 3. Tag Reading Behavior

### ID3v2.3 vs v2.4 Date Frame Mapping

DBpoweramp reads both versions and normalizes internally. When converting from v2.3 to v2.4:

| v2.3 Frames | v2.4 Frame | Format |
|-------------|-----------|--------|
| `TYER` (YYYY) | `TDRC` (Recording time) | `YYYY`, `YYYY-MM`, or `YYYY-MM-DD` |
| `TDAT` (DDMM) | — (merged into TDRC) | Day and month appended to TDRC |
| `TIME` (HHMM) | — (merged into TDRC) | Hour and minute appended to TDRC |
| `TORY` (orig. release year) | `TDOR` | YYYY only |
| `TRDA` | (discarded) | No direct v2.4 equivalent |

The reverse mapping (v2.4 → v2.3) splits `TDRC` back into `TYER`/`TDAT`/`TIME`, losing sub-year precision when the original was a full date.

**Source:** [ID3v2.4.0 Changes — ID3.org](https://id3.org/id3v2.4.0-changes), [eyeD3 Compliance Documentation](https://eyed3.readthedocs.io/en/latest/compliance.html)

### ID3v1 Coexistence

Most MP3 files contain both an ID3v2 tag (at the start) and an ID3v1 tag (at the end, 128 bytes). DBpoweramp:
- **Reads**: ID3v2 takes priority. If ID3v2 is absent or incomplete, falls back to ID3v1.
- **Writes**: Updates both ID3v2 and ID3v1 by default (configurable).
- **Conflict resolution**: ID3v2 fields override ID3v1 fields for the same property.

### APEv2 in MP3 (Read Priority)

APEv2 tags can be embedded in MP3 files (after the audio data, before or after ID3v1). DBpoweramp's read priority order is:

1. **ID3v2** (highest priority)
2. **APEv2** (if no ID3v2, or if configured to merge)
3. **ID3v1** (fallback)

Some older DBpoweramp versions or configurations treat APEv2 as a passthrough container; verify with `kid3-cli -c "get" <file>`.

### Unsynchronization (0xFF 0xE0 Escaping)

When a frame contains bytes that look like MP3 audio sync patterns (0xFF followed by 0xE0–0xFF), the tag writer applies "unsynchronization" — inserting a 0x00 byte after any 0xFF that would otherwise be misinterpreted as an audio frame sync.

- **Reading**: DBpoweramp automatically reverses unsynchronization when the unsync flag (`%10000000`) is set in the tag header.
- **Writing**: DBpoweramp can write unsynchronized tags to maximize compatibility with players that have sync-detection bugs.
- **Edge case**: The 0xFF 0x00 pattern inserted by unsync is NOT the same as a real 0xFF 0x00 audio byte sequence. Mismatched handling causes audio glitches or tag truncation.

**Source:** [ID3.org — Unsynchronisation](https://id3.org/Unsynchronisation)

### COMM (Comment) Frame Structure

The `COMM` frame stores 4 bytes of language code (ISO 639-2), followed by a null-terminated content descriptor, then the actual comment text.

```
COMM, size (4 bytes), flags (2 bytes)
  [encoding byte 0x00–0x03]
  [language: 3 ASCII bytes]
  [descriptor: null-terminated string]
  [comment: text in specified encoding]
```

- `TPE1` (Lead Artist/Lead Performers): Multiple artists are commonly stored as multiple `TPE1` frames, or as a single frame with "/" or "feat." separators.
- **Edge case**: The language field is often left blank ("   " or "XXX"). Some readers treat empty descriptors differently than absent descriptors.

---

## 4. Tag Writing Behavior

### ID3v2.4 Text Encoding

ID3v2.4 adds two encodings not available in v2.3:

| Encoding Byte | Charset |
|--------------|---------|
| `0x00` | ISO-8859-1 (Latin-1) |
| `0x01` | UTF-16 with BOM (byte order mark) |
| `0x02` | UTF-16BE (big-endian, no BOM) |
| `0x03` | **UTF-8** (v2.4 only) |

- **ID3v2.3**: No native UTF-8 support. Writers must use UTF-16 or ISO-8859-1.
- **DBpoweramp behavior**: Configurable encoding selection. UTF-16 is the safest v2.3-compatible option for non-ASCII characters.
- **Device compatibility warning**: Many car stereos, budget Bluetooth speakers, and hardware MP3 players manufactured before 2015 do not read v2.4 correctly and may display blank fields or fall back to ID3v1.

**Source:** [AudioUtils — ID3 Tags Explained](https://audioutils.com/blog/id3-tags-explained)

### Multi-Value TPE1 Handling

There are three common approaches for multiple artists:

1. **Multiple TPE1 frames**: `TPE1=Dizzy Gillespie` + `TPE1=Sonny Rollins` (permissive, recommended by spec)
2. **Slash-delimited**: `TPE1=Dizzy Gillespie / Sonny Rollins`
3. **Featuring format**: `TPE1=Dizzy Gillespie` + `TPE1=Sonny Rollins` or combined in one frame

- **DBpoweramp**: Typically writes single-value fields or uses slash delimiters for multi-artist. Verify with `kid3-cli`.
- **Risk**: Some players display "Dizzy Gillespie / Sonny Rollins" as a single artist name rather than two artists.

### TXXX Custom Frames

`TXXX` stores user-defined text frames with a description and value:

```
TXXX, size, flags
  [encoding byte]
  [description: null-terminated string]
  [value: text]
```

ReplayGain is commonly stored as:
- `TXXX:replaygain_track_gain=-6.20 dB`
- `TXXX:replaygain_track_peak=0.99996948`

DBpoweramp may or may not preserve `TXXX` frames during transcoding depending on whether it treats them as "known" or "custom" fields. Always verify with `kid3-cli`.

### Padding and Tag Growth

ID3v2 tags use padding (null bytes) after the last frame to allow adding new frames without rewriting the entire file. DBpoweramp:
- May strip padding during conversion to minimize file size.
- May add padding when writing if configured to preserve exact tag layout.

**Edge case**: A file with a heavily padded v2.3 tag converted to v2.4 will have a different byte layout. The sync-safe size encoding in v2.4 may also change the file's byte-level structure.

---

## 5. Normalization Rules

| Rule | Behavior |
|------|----------|
| **Date normalization** | TDRC values are parsed as ISO 8601. Sub-year precision is preserved but may be lost on v2.3 round-trip. |
| **Genre normalization** | Numeric genres `(17)` are converted to text "Pop" and vice versa. Custom genres remain as freeform strings. |
| **Track number** | Written as `TRCK` in "current/total" format (e.g., "3/12"). Some sources use plain "3". DBpoweramp handles both. |
| **Case normalization** | Frame IDs are always uppercase (TIT2, TPE1, etc.). Field values are preserved as-is. |
| **Encoding normalization** | On v2.3 write, UTF-8 values are re-encoded to UTF-16 or ISO-8859-1. |
| **Sort fields** | TSOA (album sort), TSOP (performer sort), TSOT (title sort), TSSC (subtitle sort) are v2.4-only. DBpoweramp strips or maps these on v2.3 write. |

---

## 6. Edge Cases

1. **TDRC with only year vs. full ISO 8601**: A `TDRC=2024` value is valid (year only). When mapped to v2.3 `TYER=2024`, the month/day/time are discarded. A full `TDRC=2024-06-04T14:30:00` maps to `TYER=2024` + `TDAT=0406` + `TIME=1430`.

2. **Xing/Info header and gapless playback**: MP3 files may contain a Xing or Info header after the first frame's audio sync word. This header stores frame count and audio properties but NOT delay/padding for gapless playback. Gapless info for MP3 is stored in a `LAME` tag (in a `TXXX` or `TLAN` frame) or in the ` Xing` header's `VBRI`/`LAME` vendor field.

3. **COMM frame with multiple languages**: A single file may have multiple `COMM` frames with different language codes. DBpoweramp reads the first matching one, or the one matching the system's language preference.

4. **APEv2 in MP3 with ID3v1 also present**: Priority is ID3v2 > APEv2 > ID3v1. An APEv2 field with the same name as an ID3v2 field will be hidden unless the ID3v2 field is stripped.

5. **Unsynchronized frame with embedded 0xFF 0xE0**: If the unsync byte (0x00) is missing due to a bug, the parser will read past the frame boundary into the next frame or into audio data, producing garbage field values or truncating subsequent frames.

---

## 7. DBpoweramp-Specific Behavior

- **Default tag version**: ID3v2.3 with UTF-16 encoding for non-ASCII text.
- **Gapless info**: DBpoweramp does not embed LAME tag delay/padding info unless explicitly configured. Verify with a hex editor looking for `LAME` strings or `XVBR`/`LAME` frames.
- **ReplayGain**: DBpoweramp does not natively write ReplayGain tags to MP3. Use a separate tool (e.g., `mp3gain`, `aacgain`) or post-process with `kid3-cli`.
- **ID3v2.4 footer**: Not written by default. Can be enabled in advanced settings.
- **APEv2 passthrough**: When converting to MP3, DBpoweramp does NOT write APEv2 tags. ID3v2 is the only output tag format.
- **Copy protection flag**: `TCOP` (Copyright) is preserved if present in source.

---

## 8. Verification Checklist

- [ ] `TYER`/`TDAT`/`TIME` preserved as separate frames in v2.3 output (or correctly merged into `TDRC` in v2.4)
- [ ] No `Invalid UTF-8` errors in kid3 output
- [ ] Multi-artist fields display correctly (not as combined strings)
- [ ] COMM frame has correct language descriptor
- [ ] TXXX fields (ReplayGain, custom) present if source had them
- [ ] No duplicate frames introduced during version conversion
- [ ] Xing/Info header preserved for VBR MP3
- [ ] Padding does not cause false sync reads in audio players

---

## 9. Sources

1. [ID3.org — ID3v2.4.0 Changes](https://id3.org/id3v2.4.0-changes)
2. [ID3.org — Unsynchronisation](https://id3.org/Unsynchronisation)
3. [eyeD3 0.9.8 — Compliance Documentation](https://eyed3.readthedocs.io/en/latest/compliance.html)
4. [AudioUtils — ID3 Tags Explained: MP3 Metadata Standard](https://audioutils.com/blog/id3-tags-explained)
5. [AudioUtils — ID3 Tags Explained (Full Page)](https://audioutils.com/blog/id3-tags-explained)
6. [Hydrogenaudio Knowledgebase — ID3 Tag Mapping](https://wiki.hydrogenaudio.org/)
7. [zmusic-ng — ID3v2.3/2.4 Conversion Code](https://git.zx2c4.com/zmusic-ng/commit/?id=f3194b59be16672dcb4b8856ef3a2f6e9162b108)
