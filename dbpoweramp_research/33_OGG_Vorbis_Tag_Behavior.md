# DBpoweramp Music Converter — OGG Vorbis Tag Behavior
*Generated: 2026-06-04 | Format: OGG (Vorbis) | Confidence: High*

---

## 1. Executive Summary

OGG Vorbis files use the Vorbis Comment format — identical in structure to FLAC's Vorbis Comments — making DBpoweramp's tag handling for OGG and FLAC largely consistent. Key differentiators are the **vendor string** (which reflects the OGG encoder, e.g., "Xiph.Org libVorbis I 20020717"), **OGG page boundary constraints** (tags must fit within OGG page limits and may span pages), and the **LYRICS field** convention. Cover art is stored via `METADATA_BLOCK_PICTURE` as a base64-encoded binary block (older method) rather than as a native OGG picture block. Comment field values have no defined maximum length per the spec (2^32-1 theoretical), but OGG page limits and player implementation constraints create practical ceilings.

---

## 2. Format Overview

An OGG Vorbis file is a bitstream of OGG pages containing Vorbis audio packets:

| Packet | Type | Description |
|--------|------|-------------|
| 1 | Identification header | Codec ID, sample rate, channels, bitrate |
| 2 | Comment header | Vendor string + all Vorbis Comment fields |
| 3+ | Audio packets | Compressed Vorbis audio data |

The Vorbis Comment header is the **second OGG packet** and begins with:
1. Vendor string length (32-bit unsigned LE)
2. Vendor string (UTF-8, not null-terminated)
3. Comment field count (32-bit unsigned LE)
4. Per field: field length (32-bit LE) + field value (UTF-8 string)

The OGG page structure adds a constraint: a Vorbis Comment header that exceeds ~64 KB may span multiple OGG pages, which some older decoders cannot handle.

**Source:** [Xiph.org — Vorbis Comment Documentation](https://xiph.org/vorbis/doc/v-comment.html)

---

## 3. Tag Reading Behavior

### Vorbis Comment Format (Same as FLAC)

OGG Vorbis follows the exact Xiph.org Vorbis Comment specification:

```
FieldName=Value
ARTIST=Dizzy Gillespie
TITLE=Sonny's Last Stand
ALBUM=Live at the Village Vanguard
TRACKNUMBER=3
DATE=1994
GENRE=Jazz
```

**Field name case insensitivity**: `ARTIST`, `artist`, `Artist` are identical. Most tools normalize to UPPERCASE.

**Multi-value fields**: Multiple values for the same field name use repeated entries:
```
ARTIST=Dizzy Gillespie
ARTIST=Sonny Rollins
ARTIST=Sonny Stitt
```
This is **explicitly encouraged** by the Vorbis spec.

**Field name character restrictions**: ASCII 0x20–0x7D, excluding 0x3D ('='). Values are UTF-8.

**Source:** [Xiph.org — Vorbis Comment Documentation](https://xiph.org/vorbis/doc/v-comment.html), [Wikipedia — Vorbis Comment](https://en.wikipedia.org/wiki/Vorbis_comment)

### Vendor String

The vendor string identifies the encoding software:

| Encoder | Vendor String Example |
|---------|----------------------|
| libvorbis 1.0 | `Xiph.Org libVorbis I 20020717` |
| AoTuV | `AoTuV [20061022]` |
| dBpoweramp | `dBpoweramp Release 15.1` |
| FLAC-in-OGG | Varies by encoder |

DBpoweramp reads and preserves the vendor string on re-tagging, or replaces it with its own identifier.

### Cover Art in OGG: COVERART vs METADATA_BLOCK_PICTURE

OGG Vorbis has two competing cover art conventions:

**Method 1 — `COVERART` field (legacy, older):**
```
COVERART=<base64-encoded JPEG or PNG binary>
```
- Field value is the raw image binary, base64-encoded.
- Simple but limited: no MIME type, no picture type, no dimensions stored.

**Method 2 — `METADATA_BLOCK_PICTURE` field (preferred):**
```
METADATA_BLOCK_PICTURE=<base64-encoded FLAC PICTURE structure>
```
- Stores a complete picture structure with MIME type, picture type, description, dimensions, and image data.
- Same format as FLAC's `METADATA_BLOCK_PICTURE`.
- Allows picture type differentiation (Front cover vs Back cover vs Artist, etc.).

**DBpoweramp behavior**: Writes `METADATA_BLOCK_PICTURE` for cover art in OGG Vorbis files. Older tools may only read `COVERART`.

### LYRICS Field

The `LYRICS` field stores unsynchronized lyrics as plain text:

```
LYRICS= Verse 1...
[blank line]
Verse 2...
```

Synchronized lyrics use the informal `SYNCEDLYRICS` or `USLT` conventions, though this is not part of the official Vorbis Comment spec.

---

## 4. Tag Writing Behavior

### Comment Field Max Length

Per the spec, each Vorbis Comment field value can be up to 2^32-1 bytes (4.29 GB) — purely theoretical. Practical limits arise from:

1. **OGG page limits**: OGG pages have a maximum granule position and byte size. Comment headers spanning many pages may be rejected by strict decoders.
2. **Memory constraints**: Most Vorbis decoders load the entire comment header into memory.
3. **Player implementation**: foobar2000, VLC, and others impose practical limits of 1–16 MB.

**Recommended maximum**: Keep total Vorbis Comment size under 1 MB. Individual field values should be under 64 KB for compatibility.

### OGG Page Boundaries and Tag Chunking

OGG pages are the fundamental unit of the OGG container, with each page containing up to 65,025 bytes of data. The Vorbis Comment header is a single logical packet but may be split across multiple OGG pages.

**Tag chunking considerations**:
- If the comment header exceeds the page size, it is fragmented across pages.
- Some OGG demuxers cannot handle fragmented metadata packets.
- DBpoweramp and libvorbis handle this correctly, but niche tools may fail.
- A "complete" comment header packet is indicated by the `EOS` (End of Stream) flag on the final page of the comment packet.

### Field Normalization

| Field | Normalization | Example |
|-------|--------------|---------|
| `REPLAYGAIN_TRACK_GAIN` | Format: "-6.20 dB" | `REPLAYGAIN_TRACK_GAIN=-6.20 dB` |
| `REPLAYGAIN_TRACK_PEAK` | Decimal, no unit | `REPLAYGAIN_TRACK_PEAK=0.99996948` |
| `REPLAYGAIN_ALBUM_GAIN` | Format: "-4.50 dB" | `REPLAYGAIN_ALBUM_GAIN=-4.50 dB` |
| `REPLAYGAIN_ALBUM_PEAK` | Decimal, no unit | `REPLAYGAIN_ALBUM_PEAK=0.87654321` |
| `TRACKNUMBER` | Often written with total | `TRACKNUMBER=3/12` or `3` |
| `DISCNUMBER` | Same as TRACKNUMBER | `DISCNUMBER=1/2` or `1` |
| `DATE` | ISO 8601 or partial | `2024`, `2024-06-04`, `2024-06` |

---

## 5. Normalization Rules

| Rule | Behavior |
|------|----------|
| **Field name case** | Normalized to UPPERCASE on write (Xiph/TagLib convention) |
| **Value encoding** | UTF-8 exclusively |
| **Multi-value fields** | Preserved as multiple entries with identical field names |
| **Empty fields** | Stripped — no empty field names or values |
| **ReplayGain format** | Values must match "-6.20 dB" format exactly |
| **Cover art** | Written as `METADATA_BLOCK_PICTURE` (preferred) or `COVERART` (legacy) |

---

## 6. Edge Cases

1. **`COVERART` without MIME type**: The `COVERART` field stores raw image bytes base64-encoded. Players must infer the format from magic bytes (JPEG SOI or PNG signature). `METADATA_BLOCK_PICTURE` avoids this ambiguity by storing the MIME type explicitly.

2. **OGG page fragmentation and seek points**: When tags are heavily fragmented across OGG pages, seeking within the file may become unreliable in some players until after the first audio packet is encountered.

3. **Vendor string overwriting**: Re-tagging with a different tool replaces the vendor string. The original encoding software identity is lost unless preserved by the re-tagger.

4. **Duplicate field names with different cases**: `artist=John` + `ARTIST=Paul` in the same file — some readers show both, others deduplicate. Per spec, they are the same field. DBpoweramp/TagLib reads both as separate values (duplicate key within the case-insensitive namespace).

5. **LYRICS exceeding page size**: Long lyrics (or multi-section lyrics) may span multiple OGG pages. Players that read the comment packet before the full page sequence may truncate or fail to display lyrics.

---

## 7. DBpoweramp-Specific Behavior

- **Tag writing**: Uses libvorbis/TagLib for Vorbis Comment I/O. Field names normalized to uppercase.
- **Cover art**: Written as `METADATA_BLOCK_PICTURE` for maximum compatibility.
- **Vendor string**: Updated to "dBpoweramp Release X.X" on re-tag.
- **ReplayGain**: Not written natively. External tools or post-processing required.
- **OGG page handling**: Fragmented comment packets are handled correctly by the OGG container layer.
- **Multi-value fields**: Fully preserved during read/write cycles.

---

## 8. Verification Checklist

- [ ] Field names appear as uppercase (`ARTIST`, not `artist`)
- [ ] Multi-value `ARTIST` fields display all values
- [ ] `METADATA_BLOCK_PICTURE` field present for cover art (base64-encoded binary, not raw bytes)
- [ ] `REPLAYGAIN_TRACK_GAIN` format is "-6.20 dB" exactly
- [ ] Vendor string identifies the encoder
- [ ] No `Invalid UTF-8` errors in kid3 output
- [ ] LYRICS field present if source had lyrics
- [ ] No truncated fields due to OGG page boundary issues

---

## 9. Sources

1. [Xiph.org — Vorbis Comment Documentation](https://xiph.org/vorbis/doc/v-comment.html)
2. [Wikipedia — Vorbis Comment](https://en.wikipedia.org/wiki/Vorbis_comment)
3. [MIT — libvorbis v-comment.html Documentation](https://web.mit.edu/cfox/share/doc/libvorbis-1.0/v-comment.html)
4. [Auphonic Blog — Ogg Vorbis Metadata (Vorbis Comment)](https://auphonic.com/blog/2012/01/22/podcast-comparison-part-3-ogg-vorbis-metadata-vorbis-comment/)
5. [datahacker — Vorbis, Vorbis Comment, and Ogg Vorbis Explained](https://datahacker.blog/home-theater/media-servers/ogg-vorbis-comment-explained)
6. [Hacker Public Radio — Audio Metadata in Ogg, MP3, and others](https://hackerpublicradio.org/eps.php?id=1393)
