# Vorbis Comment — Deep Technical Reference
> **Category:** Metadata
> **File Extensions:** `.ogg`, `.oga`, `.opus`, `.flac`, `.spx`, `.opus`
> **MIME Types:** `audio/vorbis`, `audio/ogg`, `audio/opus`, `audio/flac`
> **Standardization Body:** Xiph.org Foundation
> **Primary Specification:** Vorbis I Specification (Xiph.org) — https://xiph.org/vorbis/doc/Vorbis_I_spec.html
> **Patent Status:** Patent-free (Vorbis is unencumbered)
> **License:** BSD-3-Clause
> **Current Version:** Vorbis Comment spec 1.0 (informal, evolving through community practice)
> **Active Development:** Community-maintained (Xiph.org, Hydrogenaudio)

---

## 1. HISTORICAL CONTEXT & SPECIFICATION

### 1.1 Origin and Motivation

Vorbis Comments (officially "Vorbis comment header") were created as part of the Vorbis audio codec project by the Xiph.org Foundation, initiated by Chris Montgomery in 1994. The goal was to create a free, patent-unencumbered audio codec with metadata support built in from the start.

Unlike ID3 tags (designed as an afterthought for MP3), Vorbis Comments were designed as an integral part of the Vorbis bitstream. The comment header is one of three mandatory header packets at the start of every Vorbis stream:
1. **Identification header** (type 1) — codec version, sample rate, channels
2. **Comment header** (type 3) — metadata (this document)
3. **Setup header** (type 5) — codec configuration and codebooks

### 1.2 Adoption Beyond Vorbis

Vorbis Comments have become a de facto standard metadata format for open-source audio:

| Format | Container | Vorbis Comments Used |
|--------|-----------|---------------------|
| Ogg Vorbis | OGG | Native metadata |
| FLAC | Native | As `VORBIS_COMMENT` metadata block |
| Opus | OGG | As `VORBIS_COMMENT` metadata block |
| Speex | OGG | As `VORBIS_COMMENT` metadata block |
| Theora | OGG | As `VORBIS_COMMENT` metadata block |
| OGG | OGG | As `VORBIS_COMMENT` metadata block |
| WebM | Matroska | Via Vorbis-in-WebM wrapper |

### 1.3 Specification Documents

The Vorbis Comment specification is distributed across:
1. **Vorbis I specification, Section 5** — bitstream-level encoding/decoding
2. **Xiph.org wiki: VorbisComment** — community-maintained field list and recommendations
3. **IETF draft-ietf-cellar-flac-04** — FLAC-specific Vorbis comment mapping
4. **Hydrogenaudio Knowledgebase: Vorbis Comment** — field recommendations

---

## 2. BINARY FORMAT SPECIFICATION

### 2.1 Comment Header Packet Structure

The Vorbis comment header is a binary packet containing a specific structure. It is **not** a text file with key=value lines — those are the decoded interpretation.

**Binary packet layout:**

```
┌──────────────────────────────────────────────────────┐
│ Vendor String Length (32-bit unsigned, little-endian) │
├──────────────────────────────────────────────────────┤
│ Vendor String (UTF-8, [length] bytes, NOT null-term)  │
├──────────────────────────────────────────────────────┤
│ User Comment List Length (32-bit unsigned, LE)       │
├──────────────────────────────────────────────────────┤
│ For each comment (repeated [list length] times):     │
│  ├ Comment String Length (32-bit unsigned, LE)        │
│  └ Comment String (UTF-8, [length] bytes, key=value)  │
├──────────────────────────────────────────────────────┤
│ Framing Bit (1 bit = 1) — MUST be set                │
└──────────────────────────────────────────────────────┘
```

### 2.2 Complete Binary Layout

```
Offset    Size    Field                      Type           Description
-------   -----   -----                      ----           -----------
0x00      4B      Vendor string length       uint32 LE     Length in bytes
0x04      N       Vendor string              uint8[N]      UTF-8, NOT null-terminated
N+0x04    4B      User comment list length   uint32 LE     Number of key=value pairs
N+0x08    4B      Comment 0 length           uint32 LE     Length of first comment string
N+0x0C    M       Comment 0 string           uint8[M]      UTF-8, "KEY=Value", not null-term
...       ...     (more comments)            ...           ...
          1 bit   Framing bit                bit           Must be 1; end of packet marker
```

**Integer encoding:** All multi-byte integers in Vorbis comments are encoded in **little-endian** byte order (unlike FLAC's big-endian metadata blocks).

**String encoding:** All strings are UTF-8 encoded. No null terminators are used — lengths are explicit.

**Framing bit:** A single bit at the end of the packet, set to 1. If this bit is 0 or absent, the stream is invalid.

### 2.3 Working Binary Example

```
Hex dump of Vorbis comment packet:
[54 00 00 00]              Vendor length = 84 bytes
[58 69 70 68 2E 4F 72 67   "Xiph.Org"
 "6C 69 62 56 6F 72 62 69  "libVorbis"
73 20 49 20 32 30 30 32    "I 2002"
30 37 31 37 20 28 4C 69    "0717 (Li"
62 56 6F 72 62 69 73 29]   "bVorbis)"
                           Vendor: "Xiph.Org libVorbis I 20020717"
[0E 00 00 00]              Comment list length = 14 comments
[07 00 00 00]              Comment 0 length = 7 bytes
[54 49 54 4C 45 3D 54      "TITLE=T"
"65 73 74]                  "est"
[07 00 00 00]              Comment 1 length = 7 bytes
[41 52 54 49 53 54 3D      "ARTIST=A"
"Test]                      "Test"
[06 00 00 00]              Comment 2 length = 6 bytes
[41 4C 42 55 4D 3D         "ALBUM="
[04 00 00 00]              Comment 3 length = 4 bytes
[54 65 73 74]              "Test"
... (more comments) ...
[08 00 00 00]              Comment 13 length = 8 bytes
[43 4F 4D 4D 45 4E 54 3D]  "COMMENT="
[01]                        Framing bit = 1 (padded to byte)
```

### 2.4 Comment String Format

Each user comment string is a **key=value** pair encoded in UTF-8:

```
KEY=VALUE
```

**Rules:**
- The key is ASCII (0x20–0x7D, excluding 0x7E `~`, and MUST NOT contain `=`)
- The key is **case-insensitive** (ARTIST, artist, Artist are the same field)
- The `=` separator is **mandatory** (there must be exactly one `=`)
- The value is **case-sensitive** (UTF-8 encoded, any Unicode content allowed)
- Leading and trailing whitespace in keys and values is **preserved** (not trimmed)
- Leading and trailing whitespace in values is typically trimmed by players
- Null bytes (0x00) in strings are **undefined behavior** — avoid them
- Multiple `=` signs in the value are allowed (first `=` is the separator)

**Valid examples:**
```
TITLE=The Long and Winding Road
ARTIST=The Beatles
ALBUMARTIST=The Beatles
REPLAYGAIN_TRACK_GAIN=-6.20 dB
REPLAYGAIN_TRACK_PEAK=0.998459
COMMENT=Recorded at Abbey Road Studios
METADATA_BLOCK_PICTURE=AAAAIG锌... (base64-encoded binary)
```

**Invalid examples:**
```
=NoKey                      # No key before =
TITLE=                      # Valid (empty value allowed)
KEY=VALUE=EXTRA             # Valid (extra = is part of value)
ARTIST                      # Invalid (no = separator)
```

---

## 3. FIELD NAMES — COMPLETE REFERENCE

### 3.1 Standard Field Names (Vorbis Comment Specification)

These field names are defined in the official Vorbis I specification:

| Field Name | Description | Multi-value | Character Encoding |
|------------|-------------|-------------|-------------------|
| TITLE | Track title | No | UTF-8 |
| VERSION | Version of track (e.g., "Radio Edit") | No | UTF-8 |
| ALBUM | Album title | No | UTF-8 |
| ARTIST | Primary artist/creator name | Yes | UTF-8 |
| PERFORMER | Performing artist (if different from composer) | Yes | UTF-8 |
| COPYRIGHT | Copyright message | Yes | UTF-8 |
| LICENSE | License (e.g., Creative Commons) | Yes | UTF-8 |
| ORGANIZATION | Recording label/organization | No | UTF-8 |
| DESCRIPTION | Description of track | No | UTF-8 |
| GENRE | Genre (freeform or numeric ID3 genre) | Yes | UTF-8 |
| DATE | Date of recording (YYYY-MM-DD or just YYYY) | No | UTF-8 |
| LOCATION | Recording location | No | UTF-8 |
| CONTACT | Contact information (URL, email, phone) | No | UTF-8 |
| ISRC | International Standard Recording Code | Yes | ASCII |

### 3.2 Extended Field Names (Community Standard)

These fields are not in the original spec but are widely supported by players, taggers, and databases:

| Field Name | Description | Multi-value | Notes |
|------------|-------------|-------------|-------|
| ALBUMARTIST | Album-level artist (for compilations) | No | Critical for album grouping |
| COMPOSER | Composer/arranger | Yes | |
| GENRE | Genre (freeform text) | Yes | e.g., "Electronic" or "Electronic; Ambient" |
| TRACKNUMBER | Track number within album | No | Format: "1", "01", "1/12" |
| TRACKTOTAL | Total number of tracks | No | e.g., "12" |
| DISCNUMBER | Disc number within multi-disc set | No | Format: "1", "1/3" |
| DISCTOTAL | Total number of discs | No | e.g., "3" |
| COMMENT | General comment | Yes | Same as DESCRIPTION |
| BPM | Beats per minute (tempo) | No | Numeric, e.g., "120" |
| ENCODER | Encoder software | No | Often auto-set |
| ENCODEDBY | Encoding service/person | No | |
| RELEASETIME | Release date (extended) | No | ISO 8601: YYYY-MM-DD or YYYY-MM-DDTHH:MM:SS |
| SOURCEMEDIA | Media type | No | e.g., "CD", "Digital Media", "Vinyl" |
| BARCODE | Product barcode | No | EAN/UPC |
| CATALOGNUMBER | Label catalog number | No | |
| UPC | Universal Product Code | No | EAN/UPC |
| EAN | European Article Number | No | Often same as UPC |
| LABEL | Record label | No | |
| LICENSE | License URI | No | |
| ARTISTSORT | Sort order for artist name | No | e.g., "Beatles, The" |
| ALBUMARTISTSORT | Sort order for album artist | No | |
| TITLESORT | Sort order for title | No | |
| COMPOSERSORT | Sort order for composer | No | |
| ORIGINALDATE | Original recording date | No | For reissues |
| ORIGINALYEAR | Original recording year | No | |
| RELEASETYPE | Release type | No | e.g., "album", "single", "EP" |
| RELEASECOUNTRY | Release country code | No | ISO 3166-1 alpha-2 |
| SCRIPT | Character encoding of source | No | e.g., "Latn" (Latin) |
| MEDIA | Media type (alias for SOURCEMEDIA) | No | |
| TOTALTRACKS | Total tracks (alias for TRACKTOTAL) | No | |
| TOTALDISCS | Total discs (alias for DISCTOTAL) | No | |

### 3.3 ReplayGain Field Names

ReplayGain metadata is stored as Vorbis comments using these field names:

| Field Name | Format | Example |
|------------|--------|---------|
| REPLAYGAIN_TRACK_GAIN | `[-]X.XX dB` | `-6.20 dB` |
| REPLAYGAIN_TRACK_PEAK | `X.DDDDDD` | `0.998459` |
| REPLAYGAIN_ALBUM_GAIN | `[-]X.XX dB` | `-5.80 dB` |
| REPLAYGAIN_ALBUM_PEAK | `X.DDDDDD` | `0.998459` |

**ReplayGain format details:**
- **Gain values:** The gain is in decibels (dB), prefixed with `-` for attenuation. Positive gains (amplification) have no prefix.
  - Format: `[-]a.bb dB` where `a` = 1 or 2 digits, `bb` = 2 decimal places
  - Example: `-6.20 dB`, `+2.50 dB`, `0.00 dB`
- **Peak values:** Dimensionless ratio where 1.000000 = digital full scale. May exceed 1.0 for heavily compressed audio.
  - Format: `c.dddddd` where `c` is typically 0 or 1, and 6 decimal places
  - Example: `0.998459`, `1.218222`, `0.750000`

### 3.4 MusicBrainz Field Names

| Field Name | Description | Format |
|------------|-------------|--------|
| MUSICBRAINZ_TRACKID | Recording ID | UUID |
| MUSICBRAINZ_ALBUMID | Release ID | UUID |
| MUSICBRAINZ_ALBUMARTISTID | Release artist ID | UUID |
| MUSICBRAINZ_ARTISTID | Artist ID | UUID |
| MUSICBRAINZ_RELEASEGROUPID | Release group ID | UUID |
| MUSICBRAINZ_TRACKID | Recording ID | UUID |
| MUSICBRAINZ_RELEASETRACKID | Release-specific track ID | UUID |
| MUSICBRAINZ_WORKID | Work/composition ID | UUID |
| MUSICBRAINZ_INSTRUMENTID | Instrument ID | UUID (multi-value) |
| MUSICBRAINZ_TYPE | Release type | Freeform |
| MUSICBRAINZ_STATUS | Release status | Freeform |
| MUSICBRAINZ_URL | Related URL | URI |

### 3.5 Cover Art Field

| Field Name | Description | Encoding |
|------------|-------------|----------|
| METADATA_BLOCK_PICTURE | Cover art image | Base64-encoded FLAC METADATA_BLOCK_PICTURE binary |

---

## 4. FIELD NAME CHARACTERISTICS

### 4.1 Case Sensitivity

**Keys:** Case-insensitive. Per the Vorbis spec, field names are compared case-insensitively. In practice:
- `ARTIST=Beatles` and `artist=Beatles` are the same field
- Some taggers normalize to uppercase (e.g., `ARTIST`)
- Some taggers preserve original casing

**Values:** Case-sensitive. The value `Beatles` is different from `beatles`.

### 4.2 Multi-Value Fields

Fields that can appear multiple times with different values:

| Field | Multiple Allowed | Example |
|-------|-----------------|---------|
| ARTIST | Yes | ARTIST=John; ARTIST=Paul |
| PERFORMER | Yes | PERFORMER=Vocals; PERFORMER=Guitar |
| GENRE | Yes | GENRE=Rock; GENRE=British |
| COMMENT | Yes | Multiple language comments |
| COPYRIGHT | Yes | Multiple copyright claims |
| LICENSE | Yes | Multiple licenses |
| ISRC | Yes | Multiple ISRC codes (for multi-track files) |
| MUSICBRAINZ_ARTISTID | Yes | Multiple artist IDs |

### 4.3 Field Ordering Recommendation

The Vorbis spec recommends ordering fields alphabetically within groups. The canonical ordering:

```
1. Vendor string (first, fixed by encoder)
2. ARTIST (multiple values, alphabetically sorted)
3. ALBUMARTIST
4. ALBUM
5. TITLE
6. TRACKNUMBER / TRACKTOTAL (TRACKNUMBER before TOTALTRACKS)
7. DISCNUMBER / DISCTOTAL
8. DATE / ORIGINALDATE / RELEASETYPE
9. GENRE
10. COMMENT / DESCRIPTION
11. COMPOSER / PERFORMER / ARRANGER
12. CONDUCTOR / ENSEMBLE
13. LYRICIST
14. BPM / ISRC
15. COPYRIGHT / LICENSE
16. ENCODER / ENCODEDBY
17. REPLAYGAIN_* (track before album)
18. MUSICBRAINZ_* (in ID order)
19. METADATA_BLOCK_PICTURE
20. Other custom fields
```

This ordering is a recommendation only — players must handle fields in any order.

### 4.4 Leading/Trailing Whitespace

Per the spec, leading and trailing whitespace in keys and values is **allowed and preserved**. However:
- Most taggers trim whitespace from values for display
- Metadata editors typically strip leading/trailing whitespace
- Null bytes (0x00) in strings are **forbidden** — they terminate the string in C and cause undefined behavior in most parsers

---

## 5. VENDOR STRING

### 5.1 Purpose

The vendor string identifies the software or hardware that encoded the Vorbis/Opus/FLAC file. It is stored as the first field in every Vorbis comment block and is **read-only** — it cannot be modified without re-encoding the stream.

### 5.2 Common Vendor Strings

| Encoder | Vendor String |
|---------|---------------|
| libVorbis 1.0 | `Xiph.Org libVorbis I 20020717` |
| libVorbis 1.3 | `Xiph.Org libVorbis I` |
| aoTuV | `aoTuV [based on libVorbis]` |
| FFmpeg (native) | `Lavf58.XX.X` (FFmpeg version) |
| libopusenc | `libopusenc 0.1` |
| opusenc | `libopusenc 0.1` |
| flac 1.2.1 | `reference libFLAC 1.2.1 20070707` |
| flac 1.4.2 | `reference libFLAC 1.4.2 20220720` |
| ReplayGain | `ReplayGain 1.0` |
| rgain | `rgain 1.0` |

### 5.3 Vendor String Extraction

```bash
# Extract vendor string from FLAC
ffprobe -v quiet -show_entries format_tags=vendor -of default=noprint_wrappers=1 input.flac

# Extract vendor string from Ogg Vorbis
ffprobe -v quiet -show_entries format_tags=vendor -of default=noprint_wrappers=1 input.ogg

# Extract from Opus
ffprobe -v quiet -show_entries format_tags=vendor -of default=noprint_wrappers=1 input.opus
```

---

## 6. METADATA_BLOCK_PICTURE IN VORBIS COMMENTS

### 6.1 Storing Cover Art

In Vorbis Comment containers (Ogg Vorbis, Opus), cover art is stored in the `METADATA_BLOCK_PICTURE` field. The value is a **base64-encoded** FLAC METADATA_BLOCK_PICTURE binary structure.

### 6.2 Base64 Encoding Process

```python
import base64

# 1. Build FLAC METADATA_BLOCK_PICTURE binary structure
picture_block = bytearray()
picture_block += struct.pack('>I', 3)          # Picture type: front cover
picture_block += struct.pack('>I', len(b'image/jpeg'))  # MIME length
picture_block += b'image/jpeg'                # MIME type (ASCII, no null)
picture_block += struct.pack('>I', len('Front cover')) # Description length
picture_block += b'Front cover'               # Description (UTF-8)
picture_block += struct.pack('>I', 1200)      # Width
picture_block += struct.pack('>I', 1200)      # Height
picture_block += struct.pack('>I', 24)        # Color depth (bits per pixel)
picture_block += struct.pack('>I', 0)         # Colors (0 = non-indexed)
picture_block += struct.pack('>I', len(image_data))  # Image data length
picture_block += image_data                   # Raw JPEG/PNG bytes

# 2. Base64 encode
base64_value = base64.b64encode(picture_block).decode('ascii')

# 3. Store as Vorbis comment field
# METADATA_BLOCK_PICTURE=<base64_value>
```

### 6.3 Decoding METADATA_BLOCK_PICTURE

```python
import base64

# Read METADATA_BLOCK_PICTURE field value from Vorbis comment
base64_string = "AAAAGG..."  # from vorbiscomment or ffprobe

# Decode base64
binary_data = base64.b64decode(base64_string)

# Parse FLAC picture block structure
offset = 0
picture_type = struct.unpack('>I', binary_data[offset:offset+4])[0]; offset += 4
mime_len = struct.unpack('>I', binary_data[offset:offset+4])[0]; offset += 4
mime = binary_data[offset:offset+mime_len].decode('ascii'); offset += mime_len
desc_len = struct.unpack('>I', binary_data[offset:offset+4])[0]; offset += 4
desc = binary_data[offset:offset+desc_len].decode('utf-8'); offset += desc_len
width = struct.unpack('>I', binary_data[offset:offset+4])[0]; offset += 4
height = struct.unpack('>I', binary_data[offset:offset+4])[0]; offset += 4
depth = struct.unpack('>I', binary_data[offset:offset+4])[0]; offset += 4
colors = struct.unpack('>I', binary_data[offset:offset+4])[0]; offset += 4
data_len = struct.unpack('>I', binary_data[offset:offset+4])[0]; offset += 4
image_data = binary_data[offset:offset+data_len]

# Save image
with open('cover.jpg', 'wb') as f:
    f.write(image_data)
```

### 6.4 Multiple METADATA_BLOCK_PICTURE Fields

Each `METADATA_BLOCK_PICTURE` field in Vorbis comments contains one picture. Multiple pictures are stored as multiple `METADATA_BLOCK_PICTURE` fields. Each has its own base64-encoded picture block with its own picture type.

---

## 7. REPLAYGAIN IN VORBIS COMMENTS

### 7.1 Complete ReplayGain Storage

The ReplayGain 1.0 specification defines four values for each file:

```
REPLAYGAIN_TRACK_GAIN=-6.20 dB
REPLAYGAIN_TRACK_PEAK=0.998459
REPLAYGAIN_ALBUM_GAIN=-5.80 dB
REPLAYGAIN_ALBUM_PEAK=0.998459
```

### 7.2 ReplayGain Format Specification

**Gain field format (REPLAYGAIN_TRACK_GAIN, REPLAYGAIN_ALBUM_GAIN):**
- Pattern: `[-]a.bb dB`
- Optional minus sign for negative gain (attenuation)
- Optional plus sign for positive gain (amplification)
- `a` = integer part (1 or 2 digits, or 0)
- `bb` = two decimal digits
- Space before `dB` (mandatory)
- `dB` suffix (lowercase, mandatory)

**Peak field format (REPLAYGAIN_TRACK_PEAK, REPLAYGAIN_ALBUM_PEAK):**
- Pattern: `c.dddddd`
- `c` = integer part (typically 0 or 1; can be > 1 for highly compressed audio)
- `dddddd` = six decimal digits
- No suffix
- Represents amplitude ratio relative to full scale (1.0 = digital full scale)

### 7.3 ReplayGain Implementation

```python
def parse_replaygain_gain(value: str) -> float:
    """Parse REPLAYGAIN_TRACK_GAIN value to dB float."""
    # Format: "-6.20 dB" or "+2.50 dB" or "0.00 dB"
    if not value.endswith(' dB'):
        raise ValueError(f"Invalid gain format: {value}")
    db_str = value[:-3].strip()  # Remove " dB"
    return float(db_str)

def parse_replaygain_peak(value: str) -> float:
    """Parse REPLAYGAIN_TRACK_PEAK value to amplitude ratio."""
    # Format: "0.998459" or "1.218222"
    return float(value)

def format_replaygain_gain(db: float) -> str:
    """Format dB float to REPLAYGAIN gain string."""
    # Round to 2 decimal places
    return f"{db:.2f} dB"

def format_replaygain_peak(peak: float) -> str:
    """Format amplitude ratio to REPLAYGAIN peak string."""
    # Format with 6 decimal places
    return f"{peak:.6f}"
```

---

## 8. FFMPEG VORBIS TAG HANDLING

### 8.1 FFmpeg Metadata CLI Options

```bash
# Write individual tags
ffmpeg -i input.wav -c:a libvorbis -q:a 6 \
  -metadata title="Song Title" \
  -metadata artist="Artist Name" \
  -metadata album="Album Name" \
  -metadata date="2024" \
  -metadata genre="Electronic" \
  -metadata track="1/12" \
  output.ogg

# Write ReplayGain tags
ffmpeg -i input.ogg \
  -metadata REPLAYGAIN_TRACK_GAIN="-6.20 dB" \
  -metadata REPLAYGAIN_TRACK_PEAK="0.998459" \
  -c copy output_with_rg.ogg

# Write METADATA_BLOCK_PICTURE (requires pre-encoded base64)
ffmpeg -i input.ogg \
  -metadata:s:v title="Front cover" \
  -i cover.jpg \
  -c copy \
  -map 0:a -map 1:v \
  output_with_cover.ogg
```

### 8.2 FFmpeg Metadata Key Mapping

FFmpeg maps its generic metadata keys to container-specific field names:

| FFmpeg Key | Vorbis Comment Field | Notes |
|------------|---------------------|-------|
| title | TITLE | |
| artist | ARTIST | |
| album | ALBUM | |
| album_artist | ALBUMARTIST | |
| track | TRACKNUMBER | FFmpeg writes as single number; use TRACKTOTAL separately |
| tracktotal | TRACKTOTAL | |
| disc | DISCNUMBER | |
| disctotal | DISCTOTAL | |
| genre | GENRE | |
| date | DATE | |
| comment | COMMENT | |
| copyright | COPYRIGHT | |
| encoder | ENCODER | Auto-set by FFmpeg |
| composer | COMPOSER | |
| replaygain_track_gain | REPLAYGAIN_TRACK_GAIN | |
| replaygain_track_peak | REPLAYGAIN_TRACK_PEAK | |
| replaygain_album_gain | REPLAYGAIN_ALBUM_GAIN | |
| replaygain_album_peak | REPLAYGAIN_ALBUM_PEAK | |

### 8.3 Copying Metadata with FFmpeg

```bash
# Copy all tags from input to output
ffmpeg -i input.mp3 -c:a libvorbis -q:a 6 \
  -map_metadata 0 \
  output.ogg

# Copy tags but update specific fields
ffmpeg -i input.flac -c:a libvorbis -q:a 6 \
  -map_metadata 0 \
  -metadata title="New Title" \
  -metadata genre="Ambient" \
  output.ogg

# Strip all metadata
ffmpeg -i input.flac -c:a libvorbis -q:a 6 \
  -map_metadata -1 \
  output.ogg

# Copy all metadata AND cover art
ffmpeg -i input.flac -i cover.jpg \
  -c:a libvorbis -q:a 6 \
  -map_metadata 0 \
  -map 0:a -map 1:v \
  -metadata:s:v title="Front cover" \
  output.ogg
```

### 8.4 Reading Vorbis Comments with FFmpeg

```bash
# Read all tags as JSON
ffprobe -v quiet -print_format json \
  -show_format input.ogg | jq '.format.tags'

# Read specific tags
ffprobe -v quiet -show_entries format_tags=title,artist,album \
  -of default=noprint_wrappers=1 input.ogg

# Read ReplayGain tags
ffprobe -v quiet -show_entries format_tags=REPLAYGAIN_TRACK_GAIN,REPLAYGAIN_TRACK_PEAK \
  -of default=noprint_wrappers=1 input.ogg

# Read cover art metadata
ffprobe -v quiet -print_format json \
  -show_streams input.ogg | jq '.streams[] | select(.codec_type=="video")'
```

---

## 9. VORBISCOMMENT TOOL (vorbis-tools)

### 9.1 vorbiscomment Usage

The `vorbiscomment` tool from vorbis-tools allows direct manipulation of Vorbis comment fields:

```bash
# Read all tags
vorbiscomment -l input.ogg

# Add/update a tag
vorbiscomment -a -t "ARTIST=New Artist" input.ogg -o output.ogg

# Write multiple tags from file
vorbiscomment -w -f commentfile.txt input.ogg -o output.ogg

# Remove a tag
vorbiscomment -e -t "COMMENT" input.ogg -o output.ogg

# Add ReplayGain tags
vorbiscomment -a \
  -t "REPLAYGAIN_TRACK_GAIN=-6.20 dB" \
  -t "REPLAYGAIN_TRACK_PEAK=0.998459" \
  input.ogg -o output.ogg

# Add cover art
vorbiscomment -a -c cover.jpg input.ogg -o output.ogg
```

### 9.2 Comment File Format

`vorbiscomment -l` outputs one `KEY=Value` per line, which can be redirected to a file and used with `-f`:

```bash
# Export tags to file
vorbiscomment -l input.ogg > comments.txt

# Edit comments.txt (text editor)
# ...
vorbiscomment -w -f comments.txt input.ogg -o output.ogg
```

### 9.3 Scripting with vorbiscomment

```bash
#!/bin/bash
# Batch add ReplayGain to all .ogg files in current directory

for file in *.ogg; do
    # Run replaygain scan (requires rgain or aegisub)
    GAIN=$(aegisub-replaygain --track -f "$file" 2>/dev/null | grep 'Track gain' | awk '{print $3}')
    PEAK=$(aegisub-replaygain --track -f "$file" 2>/dev/null | grep 'Track peak' | awk '{print $3}')

    if [ -n "$GAIN" ]; then
        echo "Tagging $file: gain=$GAIN, peak=$PEAK"
        vorbiscomment -a \
          -t "REPLAYGAIN_TRACK_GAIN=$GAIN dB" \
          -t "REPLAYGAIN_TRACK_PEAK=$PEAK" \
          "$file" -o "${file%.ogg}_tagged.ogg"
        mv "${file%.ogg}_tagged.ogg" "$file"
    fi
done
```

---

## 10. INVALID TAGS AND LENIENT PARSING

### 10.1 Non-Standard Field Names

Vorbis Comments are highly permissive. Any field name using printable ASCII (0x20–0x7D, excluding `=` and `~`) is valid. Common non-standard fields include:

| Field Name | Use Case | Support |
|------------|----------|---------|
| SOURCE | Original source medium | Some players |
| PRODUCER | Record producer | Some players |
| ARRANGER | Arranger credit | Some players |
| LYRICIST | Lyricist credit | Some players |
| CONDUCTOR | Conductor credit | Some players |
| ENGINEER | Recording engineer | Some players |
| MIXER | Mixing engineer | Some players |
| REMIXER | Remixer credit | Some players |
| PUBLISHER | Label/publisher | Some players |
| LABEL | Record label | Some players |
| CATALOG | Catalog number | Some players |
| BARCODE | Product barcode | Some players |
| TOTALDISCS | Total discs | Many players |
| TOTALTRACKS | Total tracks | Many players |

### 10.2 Lenient vs Strict Parsing

| Parser | Behavior |
|--------|----------|
| **FFmpeg** | Lenient — reads any valid UTF-8 key=value |
| **Vorbiscomment** | Lenient — writes and reads any field name |
| **Mutagen (Python)** | Lenient — allows any field |
| **iTunes** | Lenient (for Vorbis in FLAC) — some custom fields ignored |
| **Roku** | Lenient — some custom fields ignored |
| **Spotify** | Strict-ish — only recognized fields are indexed |
| **MusicBrainz Picard** | Lenient — reads all fields, uses mapping |

### 10.3 Invalid Scenarios

**Field names with invalid characters:**
```
TITLE=Valid
TITLE[EXTRA]=Invalid  # [] are not in the valid range
TIT LE=Invalid         # Space in key
TIT=E=Invalid          # = in key
```

**Empty keys:**
```
=Value                  # Invalid: no key before =
```

**Empty values (allowed):**
```
TITLE=                 # Valid: empty value
ARTIST=               # Valid: empty value
```

**UTF-8 encoding errors:**
- Invalid UTF-8 byte sequences are **undefined behavior**
- Most parsers either skip the field or replace invalid bytes with replacement character (U+FFFD)

---

## 11. LACING AND SERIALIZATION

### 11.1 Vorbis Comment Lacing

The Vorbis comment list is serialized using a "lacing" scheme where each string is preceded by its length:

```
┌─────────────────┐
│ Vendor length N │  4 bytes (uint32 LE)
├─────────────────┤
│ Vendor string   │  N bytes
├─────────────────┤
│ List length M   │  4 bytes (uint32 LE)
├─────────────────┤
│ Comment 0 len L0│  4 bytes (uint32 LE)
├─────────────────┤
│ Comment 0 string│  L0 bytes (key=value)
├─────────────────┤
│ Comment 1 len L1│  4 bytes (uint32 LE)
├─────────────────┤
│ Comment 1 string│  L1 bytes
├─────────────────┤
│ ...             │
├─────────────────┤
│ Framing bit     │  1 bit = 1
└─────────────────┘
```

### 11.2 Framing Bit

The framing bit is a critical validation element:
- It is a single bit set to `1` at the very end of the comment packet
- If it is `0` or missing, the stream is invalid
- In file storage, the framing bit occupies 1 byte (0x01) padded to byte boundary

**Why the framing bit matters:**
1. Prevents truncation attacks — a truncated file will have the framing bit removed
2. Validates complete parsing — all comment data must be consumed before the framing bit
3. Distinguishes between end-of-packet and mid-packet errors

---

## 12. FLAC VORBIS COMMENTS

### 12.1 FLAC Metadata Block Structure

FLAC stores Vorbis comments in a `VORBIS_COMMENT` metadata block (type 4):

```
FLAC STREAMINFO metadata block (always first)
FLAC VORBIS_COMMENT metadata block (type 4)
  ├─ Vendor string length (uint32 LE)
  ├─ Vendor string
  ├─ Comment count (uint32 LE)
  └─ Comment strings (each: uint32 LE length + string)
FLAC PICTURE metadata block (type 6) — optional
FLAC application metadata blocks — optional
FLAC audio frames
```

### 12.2 FLAC vs Native Vorbis in OGG

| Property | FLAC VORBIS_COMMENT | OGG/Vorbis Comment |
|----------|---------------------|-------------------|
| Byte order | Little-endian (like Vorbis) | Little-endian |
| Block type | 4 (VORBIS_COMMENT) | Packet type 3 |
| Location | After STREAMINFO | Second packet in stream |
| Framing bit | Not used in block (in packet) | Required |
| Picture storage | METADATA_BLOCK_PICTURE field | Same |
| Vendor string | Same format | Same format |

### 12.3 Reading FLAC Vorbis Comments

```bash
# With ffprobe
ffprobe -v quiet -print_format json \
  -show_format input.flac | jq '.format.tags'

# With vorbiscomment (FLAC is compatible)
vorbiscomment -l input.flac

# With kid3-cli
kid3-cli -c "get" "input.flac"
```

---

## 13. OPUS IN OGG VORBIS COMMENTS

### 13.1 Opus Metadata

Opus files in OGG containers use Vorbis comments for metadata. The format is identical to Ogg Vorbis:

```bash
# Read Opus metadata
ffprobe -v quiet -print_format json \
  -show_format input.opus | jq '.format.tags'

# Write Opus metadata
ffmpeg -i input.opus \
  -metadata title="Opus Title" \
  -metadata artist="Artist" \
  -c copy output_tagged.opus
```

### 13.2 Opus-Specific Tags

| Tag | Description | Notes |
|-----|-------------|-------|
| OPUSFIELD | Opus-specific fields | Opus does not add its own fields |
| ENCODER | Encoder | Set by libopusenc: `libopusenc 0.1` |
| ENCODER_OPTIONS | Encoding parameters | |
| R128_TRACK_GAIN | Opus-specific: R128 gain | Alternative to ReplayGain |
| R128_ALBUM_GAIN | Opus-specific: R128 album gain | |

---

## 14. PRACTICAL REFERENCE TABLES

### 14.1 Complete Tag Field Reference

| Tag | Key | Multi-value | Example | FFmpeg Key |
|-----|-----|-------------|---------|-----------|
| Title | TITLE | No | `TITLE=Hello World` | title |
| Artist | ARTIST | Yes | `ARTIST=John; ARTIST=Paul` | artist |
| Album | ALBUM | No | `ALBUM=Greatest Hits` | album |
| Album Artist | ALBUMARTIST | No | `ALBUMARTIST=The Beatles` | album_artist |
| Track Number | TRACKNUMBER | No | `TRACKNUMBER=5` or `5/12` | track |
| Total Tracks | TRACKTOTAL | No | `TRACKTOTAL=12` | tracktotal |
| Disc Number | DISCNUMBER | No | `DISCNUMBER=1` or `1/3` | disc |
| Total Discs | DISCTOTAL | No | `DISCTOTAL=3` | disctotal |
| Year | DATE | No | `DATE=2024` | date |
| Genre | GENRE | Yes | `GENRE=Rock` | genre |
| Comment | COMMENT | Yes | `COMMENT=Recorded live` | comment |
| Composer | COMPOSER | Yes | `COMPOSER=Bach` | composer |
| Performer | PERFORMER | Yes | `PERFORMER=Vocals` | |
| Conductor | CONDUCTOR | No | `CONDUCTOR=Karajan` | |
| Lyricist | LYRICIST | Yes | `LYRICIST=Poet` | |
| Arranger | ARRANGER | Yes | `ARRANGER=Orchestrator` | |
| BPM | BPM | No | `BPM=120` | |
| Encoder | ENCODER | No | `ENCODER=LAME 3.100` | encoder |
| ISRC | ISRC | Yes | `ISRC=USRCG0001234` | |
| Copyright | COPYRIGHT | Yes | `COPYRIGHT=(c)2024` | copyright |
| License | LICENSE | Yes | `LICENSE=CC BY-SA 4.0` | |
| Label | LABEL | No | `LABEL=EMI` | |
| Catalog Number | CATALOGNUMBER | No | `CATALOGNUMBER=CMD-123` | |
| Barcode | BARCODE | No | `BARCODE=012345678901` | |
| UPC | UPC | No | `UPC=0012345678905` | |
| ReplayGain Track Gain | REPLAYGAIN_TRACK_GAIN | No | `REPLAYGAIN_TRACK_GAIN=-6.20 dB` | |
| ReplayGain Track Peak | REPLAYGAIN_TRACK_PEAK | No | `REPLAYGAIN_TRACK_PEAK=0.998459` | |
| ReplayGain Album Gain | REPLAYGAIN_ALBUM_GAIN | No | `REPLAYGAIN_ALBUM_GAIN=-5.80 dB` | |
| ReplayGain Album Peak | REPLAYGAIN_ALBUM_PEAK | No | `REPLAYGAIN_ALBUM_PEAK=0.998459` | |
| MusicBrainz Track ID | MUSICBRAINZ_TRACKID | No | `MUSICBRAINZ_TRACKID=...` | |
| MusicBrainz Album ID | MUSICBRAINZ_ALBUMID | No | `MUSICBRAINZ_ALBUMID=...` | |
| MusicBrainz Artist ID | MUSICBRAINZ_ARTISTID | Yes | `MUSICBRAINZ_ARTISTID=...` | |
| MusicBrainz Release Group ID | MUSICBRAINZ_RELEASEGROUPID | No | | |
| Cover Art | METADATA_BLOCK_PICTURE | Yes | `METADATA_BLOCK_PICTURE=<base64>` | (via -i image) |
| Artist Sort | ARTISTSORT | No | `ARTISTSORT=Beatles, The` | |
| Album Artist Sort | ALBUMARTISTSORT | No | | |
| Composer Sort | COMPOSERSORT | No | | |
| Release Type | RELEASETYPE | No | `RELEASETYPE=album` | |
| Original Date | ORIGINALDATE | No | `ORIGINALDATE=1969` | |
| Release Country | RELEASECOUNTRY | No | `RELEASECOUNTRY=GB` | |
| Media | SOURCEMEDIA | No | `SOURCEMEDIA=CD` | |
| Total Discs | TOTALDISCS | No | `TOTALDISCS=3` | |
| Total Tracks | TOTALTRACKS | No | `TOTALTRACKS=22` | |

### 14.2 Recommended Minimum Tag Set

For maximum compatibility, always include these tags:

```
TITLE=<title>
ARTIST=<artist>
ALBUM=<album>
DATE=<year or YYYY-MM-DD>
TRACKNUMBER=<n>
TRACKTOTAL=<total>
GENRE=<genre>
```

### 14.3 Tag Writing Order in FFmpeg

```bash
# Optimal tag writing order in FFmpeg for Ogg Vorbis
ffmpeg -i input.wav \
  -c:a libvorbis -q:a 6 \
  -metadata title="Song Title" \
  -metadata artist="Artist" \
  -metadata album="Album" \
  -metadata date="2024" \
  -metadata genre="Electronic" \
  -metadata track="5" \
  -metadata tracktotal="12" \
  -metadata disc="1" \
  -metadata disctotal="1" \
  -metadata REPLAYGAIN_TRACK_GAIN="-6.20 dB" \
  -metadata REPLAYGAIN_TRACK_PEAK="0.998459" \
  -i cover.jpg \
  -map 0:a -map 1:v \
  -metadata:s:v title="Front cover" \
  output.ogg
```

---

## 15. ADVANCED VORBIS COMMENT OPERATIONS

### 15.1 Complete Vorbis Comment Parser

```python
import struct

def parse_vorbis_comment_packet(packet_bytes):
    """Parse a raw Vorbis comment packet into a structured format."""
    offset = 0

    # Vendor string
    vendor_len = struct.unpack('<I', packet_bytes[offset:offset+4])[0]
    offset += 4
    vendor_string = packet_bytes[offset:offset+vendor_len].decode('utf-8', errors='replace')
    offset += vendor_len

    # User comment list
    comment_count = struct.unpack('<I', packet_bytes[offset:offset+4])[0]
    offset += 4

    comments = []
    for _ in range(comment_count):
        comment_len = struct.unpack('<I', packet_bytes[offset:offset+4])[0]
        offset += 4
        comment_data = packet_bytes[offset:offset+comment_len].decode('utf-8', errors='replace')
        offset += comment_len

        # Split into key=value
        if '=' in comment_data:
            eq_pos = comment_data.index('=')
            key = comment_data[:eq_pos]
            value = comment_data[eq_pos+1:]
            comments.append({'key': key, 'value': value})
        else:
            comments.append({'key': '', 'value': comment_data})

    # Framing bit (1 bit after all comments, padded to byte boundary)
    # In practice, this is the next byte if present

    return {
        'vendor_string': vendor_string,
        'comments': comments
    }

def build_vorbis_comment_packet(vendor_string, comments):
    """Build a raw Vorbis comment packet from structured data."""
    packet = bytearray()

    # Vendor string
    vendor_bytes = vendor_string.encode('utf-8')
    packet += struct.pack('<I', len(vendor_bytes))
    packet += vendor_bytes

    # User comments
    packet += struct.pack('<I', len(comments))
    for comment in comments:
        key_value = f"{comment['key']}={comment['value']}"
        key_value_bytes = key_value.encode('utf-8')
        packet += struct.pack('<I', len(key_value_bytes))
        packet += key_value_bytes

    # Framing bit (as a byte 0x01)
    packet += b'\x01'

    return bytes(packet)
```

### 15.2 Batch Tag Editing

```bash
#!/bin/bash
# Batch add ReplayGain tags to all FLAC files in a directory

ALBUM_GAIN=""
ALBUM_PEAK=""

for f in *.flac; do
    # Run ReplayGain scan
    RESULT=$(aegisub-replaygain --track "$f" 2>/dev/null)
    GAIN=$(echo "$RESULT" | grep "Track gain" | awk '{print $3}')
    PEAK=$(echo "$RESULT" | grep "Track peak" | awk '{print $3}')

    if [ -n "$GAIN" ]; then
        # Add tags with FFmpeg
        ffmpeg -y -i "$f" \
          -metadata REPLAYGAIN_TRACK_GAIN="$GAIN dB" \
          -metadata REPLAYGAIN_TRACK_PEAK="$PEAK" \
          -c copy \
          "${f%.flac}_tagged.flac"

        # Replace original
        mv "${f%.flac}_tagged.flac" "$f"
    fi
done

# Album gain (after all tracks are scanned)
echo "Album gain: $ALBUM_GAIN"
echo "Album peak: $ALBUM_PEAK"
```

```python
# Python batch tagging with mutagen
from pathlib import Path
from mutagen.flac import FLAC
from mutagen.oggvorbis import OggVorbis
import shutil

def batch_tag_folder(folder, metadata):
    """Batch apply metadata to all audio files in a folder."""
    folder = Path(folder)

    for f in folder.glob('*.flac'):
        audio = FLAC(str(f))
        for key, value in metadata.items():
            audio[key] = value
        audio.save()

    for f in folder.glob('*.ogg'):
        audio = OggVorbis(str(f))
        for key, value in metadata.items():
            audio[key] = value
        audio.save()
```

### 15.3 Tag Migration Between Formats

```python
# Migrate tags from FLAC to Opus while preserving everything
from mutagen.flac import FLAC
from mutagen.oggopus import OggOpus
from mutagen.mp4 import MP4
import subprocess

def migrate_tags(source_file, target_file):
    """Copy all applicable tags from source to target format."""

    # Read source tags
    if source_file.endswith('.flac'):
        source = FLAC(source_file)
    elif source_file.endswith('.mp3'):
        source = MP3(source_file)
    elif source_file.endswith('.m4a'):
        source = MP4(source_file)
    else:
        raise ValueError(f"Unsupported source format: {source_file}")

    # Get all tags
    tags = dict(source.tags)

    # Write to target
    if target_file.endswith('.flac'):
        target = FLAC(target_file)
    elif target_file.endswith('.ogg'):
        target = OggVorbis(target_file)
    else:
        raise ValueError(f"Unsupported target format: {target_file}")

    # Write each tag
    for key, value in tags.items():
        # Map key names if needed
        mapped_key = map_tag_key(key)
        if mapped_key:
            target[mapped_key] = value

    target.save()
```

---

## 16. REPLAYGAIN — COMPREHENSIVE GUIDE

### 16.1 ReplayGain 1.0 vs 2.0

**ReplayGain 1.0 (Original):**
- Scan analyzes RMS levels using a simple filter
- Track gain applied to normalize to -18 dB relative to 83 dB SPL playback
- Album gain applied to normalize to album's average level
- Peak protection via peak amplitude values

**ReplayGain 2.0 (Revised):**
- Compatible with EBU R128 loudness standard
- Uses ITU-R BS.1770-4 measurement algorithm
- Filters: K-weighting + pre-filter + RLB weighting
- Track gain normalizes to -18 LUFS
- Album gain normalizes to -18 LUFS for album
- Peak values are still included for protection

### 16.2 ReplayGain Measurement Process

```
1. Input: PCM samples at 44.1/48 kHz (or any rate)
         ↓
2. Apply replay gain filter (pre-filter + K-weighting)
   - Pre-filter: High-shelf boost above ~1500 Hz
   - K-weighting: High-pass filter
         ↓
3. RMS measurement over the entire track
         ↓
4. Calculate gain: G = -18 - measured_LUFS dB
         ↓
5. Peak measurement: Find maximum absolute sample value
         ↓
6. Output: REPLAYGAIN_TRACK_GAIN and REPLAYGAIN_TRACK_PEAK
```

### 16.3 ReplayGain Filter Coefficients

The ReplayGain pre-filter and K-weighting filters have specific coefficients:

**Pre-filter (high-shelf boost above ~1500 Hz):**
```python
# Pre-filter coefficients (approximate):
# High-shelf: +4 dB boost at 3 kHz
# Corner frequency: ~1500 Hz
# Q: ~0.707
```

**K-weighting filter:**
```python
# K-weighting (from ITU-R BS.1770-4):
# High-pass at ~38 Hz
# High-shelf from 1500 Hz to infinity
# Band-pass from 1500 Hz to ~7800 Hz
```

### 16.4 ReplayGain and Streaming Services

Most streaming services now use EBU R128 (loudness normalization) instead of ReplayGain:
| Service | Standard | Target Level |
|---------|----------|-------------|
| Spotify | Loudness normalization | -14 LUFS |
| Apple Music | Sound Check (proprietary) | ~-16 LUFS |
| YouTube | R128 | -14 LUFS |
| Amazon Music | R128 | -14 LUFS |
| Tidal | R128 | -14 LUFS |
| Deezer | R128 | -14 LUFS |

### 16.5 ReplayGain Tools Comparison

| Tool | RG 1.0 | RG 2.0 | R128 | Notes |
|------|---------|---------|------|-------|
| loudgain | Yes | Yes | Yes | Modern, recommended |
| aegisub-replaygain | Yes | No | No | Deprecated |
| ffmpeg (loudnorm) | No | No | Yes | Built into FFmpeg |
| rgain | Yes | No | No | Python-based |
| vorbisgain | Yes | No | No | Ogg-specific |

---

## 17. VORBIS COMMENT VALIDATION

### 17.1 Validation Checklist

When processing Vorbis comments, validate:

```python
def validate_vorbis_comment(key, value):
    """Validate a Vorbis comment key=value pair."""

    # Key validation
    if not key:
        return False, "Empty key"

    # Key characters: 0x20 to 0x7D, excluding 0x3D (=) and 0x7E (~)
    for char in key:
        code = ord(char)
        if code < 0x20 or code > 0x7D:
            return False, f"Invalid character in key: 0x{code:02X}"
        if code == 0x3D:  # =
            return False, "Key contains '=' character"
        if code == 0x7E:  # ~
            return False, "Key contains '~' character"

    # Value validation: UTF-8 encoding
    try:
        value.encode('utf-8')
    except UnicodeEncodeError:
        return False, "Value is not valid UTF-8"

    # Check for null bytes
    if '\x00' in value:
        return False, "Value contains null byte"

    return True, "Valid"
```

### 17.2 Common Validation Errors

| Error | Cause | Solution |
|-------|-------|----------|
| Key contains `=` | Invalid key character | Split at first `=` |
| Empty value after `=` | Valid, but may be unexpected | Accept or reject per spec |
| Non-UTF-8 in value | Corrupted data | Attempt replacement char decode |
| Null byte in string | Corrupted data | Truncate at null |
| Key < 0x20 | Invalid character | Remove or reject |
| Framing bit missing | Truncated packet | Reject file |

### 17.3 Tag Normalization

When reading Vorbis comments, normalize:
- Keys to uppercase (for comparison)
- Whitespace in values (trim leading/trailing)
- Track numbers to consistent format
- Dates to YYYY-MM-DD format

```python
def normalize_vorbis_comment(key, value):
    """Normalize a Vorbis comment for comparison."""

    # Normalize key (uppercase for comparison, preserve for display)
    normalized_key = key.upper()

    # Normalize specific field values
    if normalized_key == 'TRACKNUMBER':
        # Extract just the track number
        value = re.sub(r'^(\d+).*$', r'\1', value.strip())

    elif normalized_key == 'DATE':
        # Extract year
        value = re.sub(r'^(\d{4}).*$', r'\1', value.strip())

    elif normalized_key == 'REPLAYGAIN_TRACK_GAIN':
        # Ensure format: "-6.20 dB" or "+2.50 dB"
        value = value.strip()

    elif normalized_key == 'REPLAYGAIN_TRACK_PEAK':
        # Ensure format: "0.998459"
        value = f"{float(value):.6f}"

    return normalized_key, value
```

---

## 18. VORBIS COMMENT IN SPECIFIC CONTEXTS

### 18.1 Opus Cell-Based Broadcasting

In CELT/Opus cell-based broadcasting systems:
- Metadata is transmitted in a separate channel
- Vorbis comments are used for metadata
- Low-latency requirements limit metadata update frequency

### 18.2 Podcast Metadata in Ogg Opus

Podcast feeds use specific Vorbis comment fields:
```
TITLE=Episode title
ARTIST=Podcast name
ALBUM=Series name
DATE=2024-01-15
DESCRIPTION=Episode description
LICENSE=CC BY 4.0
```
Podcast RSS feeds typically include metadata in the RSS itself, not just the audio file.

### 18.3 Game Audio Vorbis Comments

Game engines embed specific metadata:
```
TITLE=Level1_BGM
CATEGORY=Music
SUBCATEGORY=Battle
LOOP=YES
CROSSFADE=2.0
```
These are non-standard but widely used in game development.

### 18.4 Field Recording Metadata

Field recordists use:
```
LOCATION=Central Park, New York
DATE=2024-06-15T14:30:00
RECORDER=Sony PCM-D100
MICROPHONE=Sennheiser MKH416
WEATHER=Sunny, 25°C
NOTES=Street performers near the fountain
```

### 18.5 Broadcast Wave Format (BWF) Tags

BWF (Broadcast Wave Format) uses RIFF chunks, but Ogg-encapsulated BWF may use Vorbis comments:
```
ORIGINAL_DATE=2024-06-15
BROADCAST_DATE=2024-06-15T18:00:00
TAPE_ID=TAPE001
```
BWF has its own standards (EBU R98) for timecode and production metadata.

---

## 19. VORBIS COMMENT API REFERENCE

### 19.1 libvorbis Comment API

```c
#include <vorbis/codec.h>

// Get the vendor string from a vorbis_info
const char *vorbis_comment_query(vorbis_comment *vc, const char *tag, int count);
char **vorbis_comment_query_list(vorbis_comment *vc, const char *tag, int *count);
void vorbis_comment_add(vorbis_comment *vc, const char *comment);
void vorbis_comment_add_tag(vorbis_comment *vc, const char *tag, const char *contents);
void vorbis_comment_clear(vorbis_comment *vc);
vorbis_comment *vorbis_comment_new(void);
```

### 19.2 libopus Comment API

```c
#include <opus.h>
#include <opuscomment.h>

// opuscomment API (from libopusenc)
opuscomment *oc_new(void);
void oc_add_comment(opuscomment *oc, const char *tag, const char *val);
char **oc_get_comments(opuscomment *oc, int *count);
const char *oc_get_vendor(opuscomment *oc);
void oc_set_vendor(opuscomment *oc, const char *vendor);
void oc_clear(opuscomment *oc);
```

### 19.3 Mutagen (Python) API

```python
from mutagen.flac import FLAC
from mutagen.oggvorbis import OggVorbis
from mutagen.oggopus import OggOpus

# FLAC
audio = FLAC('file.flac')
audio['TITLE'] = 'Song Title'
audio['ARTIST'] = 'Artist'
audio.save()

# Vorbis (OGG)
audio = OggVorbis('file.ogg')
audio['TITLE'] = 'Song Title'
audio.save()

# Opus
audio = OggOpus('file.opus')
audio['TITLE'] = 'Song Title'
audio.save()

# Reading tags
title = audio.get('TITLE', [''])[0]
artists = audio.get('ARTIST', [])  # All artists
```

---

## 20. VORBIS COMMENT SECURITY CONSIDERATIONS

### 20.1 Buffer Overflow Prevention

When parsing Vorbis comments:
- Validate all length fields before allocating
- Set maximum comment length (e.g., 8192 bytes per comment)
- Set maximum total comments (e.g., 65536)
- Use bounds-checked string operations

```c
#define MAX_COMMENT_LENGTH 8192
#define MAX_COMMENTS 65536
#define MAX_VENDOR_LENGTH 8192

if (comment_length > MAX_COMMENT_LENGTH) {
    return ERROR_COMMENT_TOO_LONG;
}
```

### 20.2 UTF-8 Security

Malformed UTF-8 in Vorbis comments can cause issues:
- Overlong encodings
- Invalid byte sequences
- Null bytes in strings

Proper handling:
```c
// Check for valid UTF-8 before storing
if (!is_valid_utf8(comment_data, comment_length)) {
    // Option 1: Reject
    return ERROR_INVALID_UTF8;

    // Option 2: Replace invalid bytes
    char *sanitized = replace_invalid_utf8(comment_data, comment_length);
}
```

### 20.3 Path Traversal in Values

Some tag values (e.g., URLs) could contain malicious content:
- Sanitize URLs in LICENSE and CONTACT fields
- Strip control characters
- Limit field lengths

---

## 21. VORBIS COMMENT FIELD COMPATIBILITY

### 21.1 iTunes/Apple Music Tag Support

| Field | Apple Music Support | Notes |
|-------|-------------------|-------|
| TITLE | ✓ | |
| ARTIST | ✓ | |
| ALBUM | ✓ | |
| ALBUMARTIST | ✓ | |
| TRACKNUMBER | ✓ | |
| DISCNUMBER | ✓ | |
| DATE | ✓ | |
| GENRE | ✓ | iTunes genre mapping |
| COMMENT | ✓ | |
| COMPOSER | ✓ | |
| COVER ART | ✓ | MP4/M4A covr atom |
| REPLAYGAIN | ✗ | Not supported |
| MUSICBRAINZ_* | ✗ | Ignored |

### 21.2 Android Tag Support

| Field | Android Support | Notes |
|-------|----------------|-------|
| TITLE | ✓ | |
| ARTIST | ✓ | |
| ALBUM | ✓ | |
| COVER ART | ✓ | |
| Most standard fields | ✓ | |
| REPLAYGAIN | ✗ | Not supported |
| MUSICBRAINZ_* | ✗ | Ignored |

### 21.3 Windows Media Player Support

| Field | WMP Support | Notes |
|-------|-------------|-------|
| TITLE | ✓ | |
| ARTIST | ✓ | |
| ALBUM | ✓ | |
| COVER ART | ✓ | |
| Extended fields | Partial | Some custom fields ignored |
| REPLAYGAIN | ✗ | Not supported |

### 21.4 Linux/Unix Tag Support

| Field | Support | Notes |
|-------|---------|-------|
| All standard fields | ✓ | |
| REPLAYGAIN | ✓ | Via specialized tools |
| MUSICBRAINZ_* | ✓ | Via Picard |
| METADATA_BLOCK_PICTURE | ✓ | Via vorbiscomment |

---

## 22. APPENDIX: COMPLETE FIELD REFERENCE

### 22.1 Standard + Extended Field List

| # | Field Name | Category | Multi-value | Example |
|---|-----------|---------|-------------|---------|
| 1 | TITLE | Basic | No | `TITLE=Hello World` |
| 2 | VERSION | Basic | No | `VERSION=Album Version` |
| 3 | ALBUM | Basic | No | `ALBUM=Greatest Hits` |
| 4 | ALBUMARTIST | Basic | No | `ALBUMARTIST=The Beatles` |
| 5 | ARTIST | Basic | Yes | `ARTIST=John; ARTIST=Paul` |
| 6 | PERFORMER | Attribution | Yes | `PERFORMER=Vocals` |
| 7 | COMPOSER | Attribution | Yes | `COMPOSER=Bach` |
| 8 | ARRANGER | Attribution | Yes | `ARRANGER=Orchestrator` |
| 9 | LYRICIST | Attribution | Yes | `LYRICIST=Poet` |
| 10 | CONDUCTOR | Attribution | No | `CONDUCTOR=Karajan` |
| 11 | ENSEMBLE | Attribution | No | `ENSEMBLE=Vienna Philharmonic` |
| 12 | TRACKNUMBER | Identification | No | `TRACKNUMBER=5` |
| 13 | TRACKTOTAL | Identification | No | `TRACKTOTAL=12` |
| 14 | DISCNUMBER | Identification | No | `DISCNUMBER=1` |
| 15 | DISCTOTAL | Identification | No | `DISCTOTAL=3` |
| 16 | DATE | Identification | No | `DATE=2024-06-15` |
| 17 | ORIGINALDATE | Identification | No | `ORIGINALDATE=1969` |
| 18 | GENRE | Classification | Yes | `GENRE=Rock` |
| 19 | MOOD | Classification | Yes | `MOOD=Energetic` |
| 20 | STYLE | Classification | Yes | `STYLE=Progressive Rock` |
| 21 | DESCRIPTION | Text | No | `DESCRIPTION=Live recording` |
| 22 | COMMENT | Text | Yes | `COMMENT=Recorded in London` |
| 23 | COMMENT | Text | Yes | `COMMENT[lang=eng]=English` |
| 24 | LOCATION | Text | No | `LOCATION=Abbey Road Studios` |
| 25 | CONTACT | Text | No | `CONTACT=info@example.com` |
| 26 | LICENSE | Legal | Yes | `LICENSE=CC BY 4.0` |
| 27 | COPYRIGHT | Legal | Yes | `COPYRIGHT=(c)2024` |
| 28 | ISRC | Identification | Yes | `ISRC=USRCG0001234` |
| 29 | BPM | Identification | No | `BPM=120` |
| 30 | ENCODER | Technical | No | `ENCODER=LAME 3.100` |
| 31 | ENCODEDBY | Technical | No | `ENCODEDBY=John Doe` |
| 32 | ORGANIZATION | Attribution | No | `ORGANIZATION=EMI` |
| 33 | LABEL | Attribution | No | `LABEL=EMI Records` |
| 34 | CATALOGNUMBER | Attribution | No | `CATALOGNUMBER=EMI-12345` |
| 35 | BARCODE | Attribution | No | `BARCODE=012345678901` |
| 36 | UPC | Attribution | No | `UPC=012345678905` |
| 37 | SOURCEMEDIA | Classification | No | `SOURCEMEDIA=CD` |
| 38 | MEDIA | Classification | No | `MEDIA=Vinyl` |
| 39 | REPLAYGAIN_TRACK_GAIN | ReplayGain | No | `REPLAYGAIN_TRACK_GAIN=-6.20 dB` |
| 40 | REPLAYGAIN_TRACK_PEAK | ReplayGain | No | `REPLAYGAIN_TRACK_PEAK=0.998459` |
| 41 | REPLAYGAIN_ALBUM_GAIN | ReplayGain | No | `REPLAYGAIN_ALBUM_GAIN=-5.80 dB` |
| 42 | REPLAYGAIN_ALBUM_PEAK | ReplayGain | No | `REPLAYGAIN_ALBUM_PEAK=0.998459` |
| 43 | ARTISTSORT | Sorting | No | `ARTISTSORT=Beatles, The` |
| 44 | ALBUMARTISTSORT | Sorting | No | `ALBUMARTISTSORT=Beatles, The` |
| 45 | TITLESORT | Sorting | No | `TITLESORT=Hello World` |
| 46 | COMPOSERSORT | Sorting | No | `COMPOSERSORT=Bach, Johann Sebastian` |
| 47 | MUSICbrainz_TRACKID | MusicBrainz | No | `MUSICBRAINZ_TRACKID=...` |
| 48 | MUSICBRAINZ_ALBUMID | MusicBrainz | No | `MUSICBRAINZ_ALBUMID=...` |
| 49 | MUSICBRAINZ_ARTISTID | MusicBrainz | Yes | `MUSICBRAINZ_ARTISTID=...` |
| 50 | MUSICBRAINZ_RELEASEGROUPID | MusicBrainz | No | `MUSICBRAINZ_RELEASEGROUPID=...` |
| 51 | METADATA_BLOCK_PICTURE | Cover Art | Yes | `METADATA_BLOCK_PICTURE=<base64>` |

---

*File generated for: Audio Engineering Knowledge Base*
*Topic: Vorbis Comment, Xiph.org Metadata, FLAC Tags, OGG Metadata*
*[NEEDS VERIFICATION] markers indicate specific numerical values that require additional source confirmation*
*[NEEDS VERIFICATION]: The exact minimum/maximum character length limits for Vorbis comment fields — the specification does not impose explicit limits, but the 32-bit length fields allow for very large strings. Some implementations may have practical limits.*
*[NEEDS VERIFICATION]: The full list of extended field names accepted by major players (Spotify, Apple Music, YouTube Music) for Vorbis/FLAC/Opus metadata — this information is proprietary and not publicly documented.*
*[NEEDS VERIFICATION]: ReplayGain 1.0 filter coefficients — the exact filter design should be verified against the original ReplayGain specification document*
*[NEEDS VERIFICATION]: The ITU-R BS.1770-4 K-weighting filter coefficients for ReplayGain 2.0 measurement*
