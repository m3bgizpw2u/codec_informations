# DBpoweramp Music Converter — AAC/MP4 iTunes Atom Tag Behavior
*Generated: 2026-06-04 | Format: AAC, M4A, MP4 | Confidence: High*

---

## 1. Executive Summary

MP4/M4A files store metadata in a hierarchical "atom" (also called "box") structure, with iTunes-compatible tags residing in the `moov.udta.meta.ilst` path. DBpoweramp's MP4 tag handling must work within this atom hierarchy: each tag field is a named parent atom containing a `data` child atom with the actual value. Key behaviors include: text fields capped at 255 bytes (UTF-8), `trkn` and `disk` stored as 6-byte binary blobs, `covr` uniquely allowing multiple `data` children for multiple images, and freeform metadata via `----:com.apple.iTunes:*` atoms for ReplayGain and custom fields. The `iTunSMPB` atom stores gapless playback delay/padding. The `©too` atom records the encoder identity.

---

## 2. Format Overview

MP4/M4A is a QuickTime-based container format using a hierarchical atom (box) structure:

```
moov (movie container)
  └─ udta (user data)
      └─ meta (metadata container)
          └─ ilst (iTunes metadata list)
              ├─ ©nam (title)
              ├─ ©ART (artist)
              ├─ ©alb (album)
              ├─ ©day (year/date)
              ├─ ©gen (custom genre)
              ├─ gnre (predefined genre, index+1 vs ID3)
              ├─ trkn (track number, binary)
              ├─ disk (disc number, binary)
              ├─ covr (cover art, binary or multiple data atoms)
              ├─ cpil (compilation flag, 1 byte)
              ├─ tmpo (BPM, 16-bit integer)
              ├─ ©wrt (composer)
              ├─ aART (album artist)
              ├─ pgap (gapless playback flag)
              ├─ ©grp (grouping)
              ├─ ----:com.apple.iTunes:* (freeform)
              └─ iTunSMPB (gapless delay/padding)
```

Each named atom (e.g., `©nam`) contains a `data` child atom that holds the actual value.

**Source:** [AtomicParsley — MPEG-4 Files Documentation](https://atomicparsley.sourceforge.net/mpeg-4files.html), [Apple Developer — Atoms](https://developer.apple.com/documentation/quicktime-file-format/atoms)

---

## 3. Tag Reading Behavior

### Core iTunes Atoms

| Atom | Data Type | Description | Value Format |
|------|-----------|-------------|-------------|
| `©nam` | UTF-8 string | Title | Text, max 255 bytes |
| `©ART` | UTF-8 string | Artist | Text, max 255 bytes |
| `©alb` | UTF-8 string | Album | Text, max 255 bytes |
| `©day` | UTF-8 string | Release date | `YYYY-MM-DD`, may be partial |
| `©gen` | UTF-8 string | Custom genre | Text, overrides `gnre` |
| `gnre` | 16-bit integer | Predefined genre | ID3v1 genre index + 1 |
| `©wrt` | UTF-8 string | Composer | Text, max 255 bytes |
| `aART` | UTF-8 string | Album artist | Text, max 255 bytes |
| `©grp` | UTF-8 string | Grouping | Text, max 255 bytes |
| `©cmt` | UTF-8 string | Comment | Text |
| `cprt` | UTF-8 string | Copyright | Text |
| `©lyr` | UTF-8 string | Lyrics | Text (no 255-byte limit unlike other text) |
| `©too` | UTF-8 string | Encoder | e.g., "dBpoweramp 15.1" |

**Numeric atoms:**

| Atom | Data Type | Description | Value Format |
|------|-----------|-------------|-------------|
| `trkn` | 6-byte binary | Track number | `[reserved=0, track_hi, track_lo, total_hi, total_lo, reserved=0]` |
| `disk` | 6-byte binary | Disc number | Same structure as `trkn` |
| `tmpo` | 16-bit integer | BPM | 0–65535 |
| `cpil` | 8-bit integer | Compilation | 0 or 1 |
| `pgap` | 8-bit integer | Part of gapless album | 0 or 1 |

**Binary atoms:**

| Atom | Data Type | Description |
|------|-----------|-------------|
| `covr` | JPEG/PNG/BMP | Cover art. Unique: may contain multiple `data` children for multiple images |

**Source:** [libmp4v2 Wiki — iTunesMetadata](https://github.com/sergiomb2/libmp4v2/wiki/iTunesMetadata), [Android mp4v2 — mp4meta.cpp](https://android.googlesource.com/platform/external/mp4v2/+/master/src/mp4meta.cpp)

### trkn Atom Format (6 bytes)

```
Byte 0:  Reserved (0x00)
Byte 1:  Track number high byte
Byte 2:  Track number low byte
Byte 3:  Total tracks high byte
Byte 4:  Total tracks low byte
Byte 5:  Reserved (0x00)
```

For track 3 of 12: `[0x00, 0x00, 0x03, 0x00, 0x0C, 0x00]`

The mp4v2 library confirms this layout: track and total tracks are stored as 16-bit big-endian values.

### covr Atom (Multiple Images)

`covr` is the **only** MP4 atom that permits multiple `data` child atoms. This allows:
- Multiple cover art images (front, back, booklet)
- Multiple formats (JPEG and PNG simultaneously)

DBpoweramp can read all covr data children. On write, it may replace all existing covr entries or append new ones depending on configuration.

### gnre vs ©gen

- `gnre`: Predefined genre stored as ID3v1 genre index + 1 (e.g., 18 for "Rock"). This is because iTunes uses 1-based indexing while ID3v1 uses 0-based.
- `©gen`: Custom/freeform genre text. If present, it overrides `gnre` for display purposes.

### Freeform Atoms (----:com.apple.iTunes:*)

These atoms enable ReplayGain and custom metadata:

| Atom Name | Purpose | Example Value |
|-----------|---------|---------------|
| `----:com.apple.iTunes:REPLAYGAIN_TRACK_GAIN` | Track gain | `-6.20 dB` |
| `----:com.apple.iTunes:REPLAYGAIN_TRACK_PEAK` | Track peak | `0.99996948` |
| `----:com.apple.iTunes:REPLAYGAIN_ALBUM_GAIN` | Album gain | `-4.50 dB` |
| `----:com.apple.iTunes:REPLAYGAIN_ALBUM_PEAK` | Album peak | `0.87654321` |
| `----:com.apple.iTunes:iTunSMPB` | Gapless delay/padding | `0000 1A20 0000 1B78 0000 0000 0000 0000` |
| `----:com.apple.iTunes:ENCODER` | Encoder identity | `dBpoweramp 15.1` |

The `----` prefix signals a freeform atom. The `com.apple.iTunes:` namespace prevents collisions with standard atoms.

---

## 4. Tag Writing Behavior

### Text Field Length Limit

Standard text atoms in MP4 have a **255-byte limit** for the `data` value (after the atom type and flags). This includes:
- All `©*` text atoms (title, artist, album, composer, etc.)
- The exception is `©lyr` (lyrics), which has no documented 255-byte limit.

When a field exceeds 255 bytes, some taggers truncate silently; others refuse to write. DBpoweramp may truncate or preserve depending on configuration.

### iTunSMPB for Gapless Playback

The `iTunSMPB` atom stores delay and padding samples for gapless playback:

```
Format: "DDDD PPPP [optional]"
- DDDD: Number of silence samples before audio begins (delay)
- PPPP: Number of silence samples at end (padding)
```

Example: `iTunSMPB=0000 1A20 0000 1B78` means:
- Delay: 0x1A20 = 6688 samples
- Padding: 0x1B78 = 7032 samples

This is iTunes-specific and not understood by all players. DBpoweramp may preserve or strip it depending on the conversion path.

### Multiple covr Images

DBpoweramp can write multiple `covr` entries:
- The `covr` parent atom contains multiple `data` children.
- Each `data` child has a `mdia.minf.hdlr` class type indicating format (JPEG/PNG).
- Reading tools may display only the first image or all images.

### ©too Encoder Tag

`©too` (encoder) records the software used to create the file. DBpoweramp writes this atom identifying itself. Example:

```
©too = "dBpoweramp Release 15.1"
```

---

## 5. Normalization Rules

| Rule | Behavior |
|------|----------|
| **Genre mapping** | Numeric genre → `gnre` atom (ID3 index + 1). Text genre → `©gen` atom. Both may coexist. |
| **Track/disc number** | Written as binary 6-byte `trkn`/`disk` atoms. Display often shows "3/12". |
| **Cover art format** | JPEG preferred by iTunes. PNG supported. BMP less common. |
| **Text encoding** | UTF-8 without BOM, not null-terminated |
| **Sort atoms** | `soal` (album sort), `soar` (artist sort), `sosn` (show sort), `sonm` (title sort) are standard MP4 atoms |
| **Compilation flag** | `cpil=1` sets the compilation flag in iTunes |

---

## 6. Edge Cases

1. **trkn total = 0**: Some encoders write `trkn` with total=0 meaning "track N of unknown total". iTunes displays this as just "N" without the slash.

2. **©day partial dates**: A file with `©day=2024` (year only) is valid. The ISO 8601 partial date format is supported. When converting from FLAC's `DATE=2024-06-04`, DBpoweramp may store only the year or the full date.

3. **Multiple gnre atoms**: Some files have both `gnre` (numeric) and `©gen` (text) for the same genre. iTunes prioritizes `©gen` for display. DBpoweramp may write both or normalize to one.

4. **covr data atom without mime type**: Older files may have `covr` children without proper `mdia.minf.hdlr` class indicators. DBpoweramp infers format from magic bytes (JPEG SOI 0xFF 0xD8, PNG signature).

5. **iTunSMPB with incorrect values**: If delay+padding exceeds the actual audio length (for very short tracks), playback may have audible glitches. DBpoweramp should preserve original values during lossless transcoding.

---

## 7. DBpoweramp-Specific Behavior

- **Tag writing**: DBpoweramp uses the `mp4v2` or equivalent library to write standard iTunes-compatible atoms.
- **Cover art**: Written as `covr` with proper JPEG/PNG binary data.
- **trkn/disk**: Written as properly formatted 6-byte binary atoms.
- **ReplayGain**: May write to `----:com.apple.iTunes:REPLAYGAIN_TRACK_GAIN` freeform atoms if configured.
- **iTunSMPB**: Preserved during lossless transcoding (e.g., ALAC → AAC-LC). Stripped during lossy conversion.
- **Sort atoms**: Preserved if source had them; may be stripped during re-encode.
- **©too encoder tag**: Always written, identifying DBpoweramp.

---

## 8. Verification Checklist

- [ ] `©nam`, `©ART`, `©alb`, `©day` display correctly (UTF-8, no garbled characters)
- [ ] `trkn` shows as "3/12" format (not binary hex)
- [ ] `covr` shows as `[JPEG, N bytes]` or `[PNG, N bytes]` in kid3
- [ ] `pgap` present if source had gapless flag
- [ ] `©too` shows "dBpoweramp" (not stripped)
- [ ] `----:com.apple.iTunes:*` freeform atoms preserved if source had ReplayGain
- [ ] No `TagLib: MP4: Ignoring duplicate atom` warnings
- [ ] Numeric IDs (`Artist ID`, `Album ID`) preserved if present in source

---

## 9. Sources

1. [AtomicParsley — MPEG-4 Files Documentation](https://atomicparsley.sourceforge.net/mpeg-4files.html)
2. [libmp4v2 Wiki — iTunesMetadata](https://github.com/sergiomb2/libmp4v2/wiki/iTunesMetadata)
3. [Android mp4v2 — src/mp4meta.cpp](https://android.googlesource.com/platform/external/mp4v2/+/master/src/mp4meta.cpp)
4. [Apple Developer Documentation — Atoms](https://developer.apple.com/documentation/quicktime-file-format/atoms)
5. [KaijuConverter — M4A Format Guide](https://kaijuconverter.com/guides/m4a-mpeg4-audio-format)
6. [SteveMarshall/mp4-quicktime — atom.py](https://github.com/SteveMarshall/mp4-quicktime/blob/master/atom.py)
