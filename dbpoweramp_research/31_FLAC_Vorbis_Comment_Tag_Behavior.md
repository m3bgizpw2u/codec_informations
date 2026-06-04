# DBpoweramp Music Converter — FLAC Vorbis Comment Tag Behavior
*Generated: 2026-06-04 | Format: FLAC | Confidence: High*

---

## 1. Executive Summary

FLAC files use Vorbis Comments as their primary metadata system — the same field-name=value format as OGG Vorbis and Opus. DBpoweramp's FLAC tag handling is relatively straightforward: field names are case-insensitive (stored internally as uppercase per Xiph.org convention), multi-value fields are natively supported by repeating the field name, ReplayGain uses the standard `REPLAYGAIN_TRACK_GAIN` format ("-6.20 dB"), and cover art is stored as a `METADATA_BLOCK_PICTURE` (base64-encoded binary block, not raw bytes). The critical requirement is that `STREAMINFO` and `SEEKTABLE` are preserved bit-exactly — tag writing must not regenerate or alter these structural metadata blocks.

---

## 2. Format Overview

A FLAC file is structured as:

| Block | Type | Description |
|-------|------|-------------|
| `fLaC` | Marker | File signature |
| `STREAMINFO` | 0 (mandatory, first) | Sample rate, channels, bit depth, frame count, MD5 signature |
| `APPLICATION` | 1 | Third-party data |
| `SEEKTABLE` | 3 | Byte offsets to audio frames (optional but common) |
| `VORBIS_COMMENT` | 4 | All metadata tags |
| `CUESHEET` | 5 | Track/index points (optional) |
| `PICTURE` | 6 | Embedded cover art as native FLAC picture block |
| `FLAC` | Last | End of metadata |

The Vorbis Comment block is stored within FLAC's metadata block system, not as a separate Ogg Vorbis comment header. However, the field format follows the Xiph.org Vorbis Comment specification exactly.

---

## 3. Tag Reading Behavior

### Vorbis Comment Field Case Sensitivity

Per the Xiph.org Vorbis Comment specification:

> "A case-insensitive field name that may consist of ASCII 0x20 through 0x7D, 0x3D ('=') excluded. ASCII 0x41 through 0x5A inclusive (A-Z) is to be considered equivalent to ASCII 0x61 through 0x7A inclusive (a-z)."

This means `ARTIST`, `artist`, and `Artist` are treated as the **same field** during reads.

- **Recommended practice**: Read and write field names in a consistent case (typically UPPERCASE per Xiph convention). DBpoweramp and TagLib follow this convention.
- **Edge case**: A file with both `ARTIST=John` and `artist=Paul` (two different-case entries) will read as two separate values for the same field. Some readers display both, others show only one.

**Source:** [Xiph.org — Vorbis Comment Documentation](https://xiph.org/vorbis/doc/v-comment.html), [Wikipedia — Vorbis Comment](https://en.wikipedia.org/wiki/Vorbis_comment)

### Vendor String

The first field in every Vorbis Comment block is the **vendor string**, stored as a length-prefixed field separate from the comment list. Example:

```
vendor_string = "reference libFLAC 1.3.4"
```

DBpoweramp preserves the vendor string during tag updates. When re-tagging, the new vendor string reflects the writing application (e.g., "dBpoweramp 15.1").

### Multi-Value Fields

Vorbis Comments natively support multiple values for the same field name by repeating the field:

```
ARTIST=Dizzy Gillespie
ARTIST=Sonny Rollins
ARTIST=Sonny Stitt
```

This is explicitly encouraged by the spec. DBpoweramp reads all values and passes them through on write.

### FIELD=NORMALIZE Behavior

There is no formal `FIELD=NORMALIZE` convention in Vorbis Comments. However, DBpoweramp and other tools often apply informal normalization:

| Field | Normalization |
|-------|--------------|
| `REPLAYGAIN_TRACK_GAIN` | Format: "-6.20 dB" (signed float, 2 decimal places) |
| `REPLAYGAIN_ALBUM_GAIN` | Same format |
| `REPLAYGAIN_TRACK_PEAK` | Format: "0.99996948" (decimal, no unit) |
| `REPLAYGAIN_ALBUM_PEAK` | Same format |
| `TRACKNUMBER` | May be written as "3" or "03" or "3/12" |
| `DISCNUMBER` | Same as track number |

### METADATA_BLOCK_PICTURE (Cover Art)

FLAC supports cover art in two ways:

1. **Native FLAC PICTURE block** (type 6 in metadata block system) — preferred
2. **Vorbis Comment `METADATA_BLOCK_PICTURE`** — legacy, stores the entire PICTURE block as a base64-encoded string

The `METADATA_BLOCK_PICTURE` Vorbis field value must be:
- The complete binary PICTURE block (same structure as a native FLAC PICTURE block)
- **base64-encoded** (not raw binary bytes)
- Prefixed with: MIME type, picture type (1=Front cover), description, dimensions

**Critical**: Writing raw image bytes (e.g., `METADATA_BLOCK_PICTURE=@cover.jpg`) is **incorrect** and will not display as cover art in most players. The field value must be the base64-encoded binary representation of the FLAC PICTURE structure.

**Source:** [Xiph.org — FLAC METADATA_BLOCK_PICTURE](https://xiph.org/flac/format.html#metadata_block_picture)

---

## 4. Tag Writing Behavior

### Field Length Limits

| Limit Type | Value |
|-----------|-------|
| Max comment list length | 4,294,967,295 fields (32-bit unsigned) |
| Max individual field value length | 4,294,967,295 bytes (theoretical) |
| FLAC practical limit | Varies by player; 1–16 MB is safe for most |
| `METADATA_BLOCK_PICTURE` value | Up to ~16 MB recommended; some players truncate |

FLAC's metadata block system places a practical limit because the entire `VORBIS_COMMENT` block is loaded into memory by most decoders.

### ReplayGain Format

The ReplayGain specification for Vorbis/FLAC uses:

```
REPLAYGAIN_TRACK_GAIN=-6.20 dB    # Gain in dB, signed float
REPLAYGAIN_TRACK_PEAK=0.99996948   # Peak amplitude, float (1.0 = max)
REPLAYGAIN_ALBUM_GAIN=-4.50 dB
REPLAYGAIN_ALBUM_PEAK=0.87654321
```

**Important**: DBpoweramp does NOT write ReplayGain tags natively. Use `metaflac --add-replay-gain` or a dedicated ReplayGain scanner.

### CUE Sheet Preservation

FLAC files may contain a `CUESHEET` metadata block (type 5) storing track/index cue points. This block is **separate from** any Vorbis Comment `CUESHEET` field (which stores an external CUE file as text). DBpoweramp:
- Preserves the native CUESHEET block during tag updates.
- Does NOT parse or modify embedded cue data.
- Does NOT write `CUESHEET` text fields by default.

### STREAMINFO and SEEKTABLE Preservation

`STREAMINFO` is the first mandatory metadata block and contains:
- `sample_rate`, `channels`, `bits_per_sample`, `total_samples`
- `minimum_block_size`, `maximum_block_size`
- `frame_count`
- `audio_md5` signature

`SEEKTABLE` stores byte offsets for random-access seeking.

**Critical rule**: Tag writing operations must **never regenerate** these blocks. A proper FLAC tagger:
1. Reads all existing metadata blocks.
2. Modifies only the `VORBIS_COMMENT` block.
3. Writes all blocks back in the same order.

DBpoweramp respects this rule during conversion operations.

---

## 5. Normalization Rules

| Rule | Behavior |
|------|----------|
| **Field name case** | Normalized to UPPERCASE on write (DBpoweramp/TagLib convention) |
| **Value encoding** | Always UTF-8 (no BOM, no escape sequences) |
| **Multi-value order** | Preserved as written (first occurrence = primary for display) |
| **Empty fields** | Stripped — empty field names or values are not written |
| **ReplayGain format** | Values checked for correct format; malformed values may be discarded |

---

## 6. Edge Cases

1. **Mixed-case duplicate fields**: A file with `artist=John` and `ARTIST=Paul` (same field, different case) will read as two artist values. Some tools deduplicate on write, some preserve both.

2. **`METADATA_BLOCK_PICTURE` base64 vs raw**: If a tool writes raw JPEG/PNG bytes as the field value (not base64), most players will fail to display the image. The field content must be the base64 encoding of the full FLAC PICTURE structure (which includes MIME type, dimensions, picture type header).

3. **SEEKTABLE and tag size growth**: Adding many tags can push the `VORBIS_COMMENT` block size past certain thresholds. Most FLAC encoders handle this, but very large comment blocks (>8 MB) may cause seek table recalculation by some tools.

4. **Vorbis comment vendor string overwriting**: The vendor string is part of the Vorbis Comment block header, not a regular field. Re-tagging with some tools overwrites it with the new encoder's string, which is technically correct but may lose the original encoding software identity.

5. **`REPLAYGAIN_*_GAIN` with trailing space or extra unit**: `REPLAYGAIN_TRACK_GAIN=-6.20 dB ` (trailing space) or `REPLAYGAIN_TRACK_GAIN=-6.20db` (lowercase, no space) may be read incorrectly by some players. Format must be exactly "-6.20 dB" with the unit.

---

## 7. DBpoweramp-Specific Behavior

- **Tag writing**: DBpoweramp writes standard Vorbis Comments to FLAC output. Field names are normalized to uppercase.
- **Cover art**: Written as native FLAC PICTURE block (metadata type 6), not as a `METADATA_BLOCK_PICTURE` Vorbis field. This is the correct, preferred method.
- **ReplayGain**: Not written by DBpoweramp. External tools required.
- **STREAMINFO/SEEKTABLE**: Always preserved during conversion operations.
- **Multi-value fields**: Fully supported and preserved.
- **TagLib backend**: DBpoweramp uses TagLib for FLAC tag I/O, ensuring consistent field name casing and encoding.

---

## 8. Verification Checklist

- [ ] Field names appear as uppercase in kid3 output (`ARTIST`, not `artist`)
- [ ] Multi-value fields (e.g., multiple `ARTIST`) display all values
- [ ] `REPLAYGAIN_TRACK_GAIN` format is exactly "-6.20 dB" (not "-6.2 dB" or "-6.20")
- [ ] Cover art shows as `[JPEG, N bytes]` or `[PNG, N bytes]` in kid3
- [ ] No `Invalid UTF-8` errors
- [ ] No `Ignoring duplicate atom` warnings (N/A for FLAC, but check for malformed fields)
- [ ] STREAMINFO is preserved (compare `metaflac --show-stats` before and after)
- [ ] SEEKTABLE preserved if originally present

---

## 9. Sources

1. [Xiph.org — Vorbis Comment Documentation](https://xiph.org/vorbis/doc/v-comment.html)
2. [Xiph.org — FLAC Format Specification (METADATA_BLOCK_PICTURE)](https://xiph.org/flac/format.html#metadata_block_picture)
3. [Wikipedia — Vorbis Comment](https://en.wikipedia.org/wiki/Vorbis_comment)
4. [MIT — libvorbis v-comment.html Documentation](https://web.mit.edu/cfox/share/doc/libvorbis-1.0/v-comment.html)
5. [Auphonic Blog — Ogg Vorbis Metadata (Vorbis Comment)](https://auphonic.com/blog/2012/01/22/podcast-comparison-part-3-ogg-vorbis-metadata-vorbis-comment/)
6. [Hydrogenaudio Knowledgebase — ReplayGain Tag Format](https://wiki.hydrogenaudio.org/)
