# Embedded Cover Art in Audio Containers — Deep Technical Reference
> **Category:** Metadata
> **Topic:** Embedded Cover Art, APIC, METADATA_BLOCK_PICTURE, covr Atom, WM/Picture
> **Related Codecs:** MP3, FLAC, Vorbis, Opus, AAC, WAV, AIFF, WMA, DSF, DFF
> **FFmpeg Options:** -attach, -disposition:v, -metadata:s:v
> **Standards:** ID3v2.3/2.4 (ID3.org), FLAC format (Xiph.org), ISO Base Media (ISO/IEC 14496-12)
> **MIME Types:** image/jpeg, image/png, image/gif (discouraged)

---

## 1. HISTORICAL CONTEXT & MOTIVATION

### 1.1 Why Embed Cover Art?

Embedding cover art directly in audio files provides several advantages over external files:
1. **Portability:** The album art travels with the file — no broken links to sidecar .jpg files
2. **Consistency:** All players display the correct art regardless of folder structure
3. **Metadata completeness:** A complete audio file includes its visual identity
4. **Album identification:** Cover art enables automatic album recognition in music databases

### 1.2 History of Cover Art in Digital Audio

| Year | Development | Format |
|------|------------|--------|
| 1996 | ID3v1.1 extended tags | No cover art support |
| 1998 | ID3v2.0 introduced | APIC frame for embedded images |
| 2001 | FLAC 1.0 released | METADATA_BLOCK_PICTURE for cover art |
| 2002 | Vorbis Comment spec | METADATA_BLOCK_PICTURE via base64 encoding |
| 2003 | iTunes 4.0 | MP4 covr atom |
| 2005 | WMA/ASF | WM/Picture attribute |
| 2011 | Opus in Ogg | Vorbis Comment METADATA_BLOCK_PICTURE |
| 2015 | DSD files (DSF/DFF) | ID3v2 with APIC |

### 1.3 Image Format Support

| Format | Pros | Cons | Recommendation |
|--------|------|------|----------------|
| JPEG (image/jpeg) | Small file size, universal support, ~80-95% quality achievable | Lossy — artifacts on sharp edges | **Recommended** |
| PNG (image/png) | Lossless, transparency support | Larger file size, not universally supported in all players | Use for logos/transparency |
| GIF (image/gif) | Small for simple graphics, animation | 256-color limit, large for photos | Avoid for cover art |
| BMP (image/bmp) | Simple, lossless | Very large files, rarely used | Not recommended |

**Recommended settings for JPEG cover art:**
- Format: JPEG (JFIF)
- Color: RGB or CMYK (RGB preferred for compatibility)
- Quality: 80–90% (typically 500 KB–2 MB for 1200×1200 px)
- Dimensions: 600×600 to 2000×2000 pixels
- Maximum practical size: 10 MB (some players struggle with larger)

---

## 2. APIC FRAME IN ID3v2

### 2.1 APIC Frame Specification

The **APIC** (Attached PICTure) frame is defined in ID3v2.3 and ID3v2.4 specifications. It stores a single image along with its MIME type, picture type, and textual description.

**Frame ID:** `APIC` (0x41 0x50 0x49 0x43)

**Frame header (ID3v2.4 — 10 bytes):**
```
Offset  Size  Field           Type        Description
------  ----  -----           ----        -----------
0x00    4B    Frame ID        char[4]     "APIC"
0x04    4B    Size            uint32 BE   Frame size (excluding 10-byte header)
0x08    2B    Flags           uint16 BE   Flags (0x0000 for APIC)
```

**Frame header (ID3v2.3 — 10 bytes):**
```
Offset  Size  Field           Type        Description
------  ----  -----           ----        -----------
0x00    4B    Frame ID        char[4]     "APIC"
0x04    4B    Size            uint32 BE   Frame size (excluding 10-byte header)
0x08    2B    Flags           uint16 BE   Flags
```

### 2.2 APIC Frame Body — Complete Binary Layout

Following the frame header, the frame body consists of:

```
Offset   Size   Field              Type            Description
-------  -----  -----              ----            -----------
0x00     1B     Text encoding      uint8           0=Latin-1, 1=UTF-16, 2=UTF-16BE, 3=UTF-8
0x01     N+1B   MIME type          string          MIME type + null terminator (e.g., "image/jpeg\x00")
         1B     Picture type       uint8           0x00–0x14 (see picture type table)
         N+1B   Description        string          Description + null terminator (encoding-aware)
         N      Picture data       binary          Raw image bytes
```

**Complete binary layout (offset from frame body start):**

```
Byte 0:       Text encoding byte (0x03 = UTF-8 most common)
Bytes 1–N+1:  MIME type string + 0x00 terminator
             Example: "image/jpeg\x00" (11 bytes for JPEG)
             Example: "image/png\x00"  (10 bytes for PNG)
Byte N+2:     Picture type byte (0x03 = front cover most common)
Bytes N+3–M:  Description string + 0x00 terminator
             (encoding determined by text encoding byte)
             Example (UTF-8): "Front cover\x00"
Bytes M+1–K:  Binary image data (JPEG or PNG bytes)
```

**Working example — Front Cover JPEG in MP3:**

```
Frame header:
[41 50 49 43]           "APIC" frame ID
[00 00 1F 41]           Frame size = 8001 bytes (big-endian uint32)
[00 00]                 Flags = 0

Frame body:
[03]                    Text encoding = UTF-8
[69 6D 61 67 65 2F      "image/jpeg" (MIME type)
 6A 70 65 67 00]         + null terminator
[03]                    Picture type = 0x03 = Front cover
[46 72 6F 6E 74          "Front cover" (description in UTF-8)
 20 63 6F 76 65          + null terminator
 72 00]
[FF D8 FF E0 ...]       JPEG SOI marker + image data
[... JPEG binary ...]
[FF D9]                 JPEG EOI marker
```

### 2.3 ID3v2 APIC Picture Type Codes

| Hex | Decimal | Name | Description | Uniqueness Rule |
|-----|---------|------|-------------|----------------|
| 0x00 | 0 | Other | Any other image type | Multiple allowed |
| 0x01 | 1 | 32×32 file icon | PNG only, 32×32 pixels | One per file max |
| 0x02 | 2 | Other file icon | Other icon image | One per file max |
| 0x03 | 3 | **Front cover** | Cover of the album/conTAINER | Multiple allowed |
| 0x04 | 4 | Back cover | Cover of the album (back) | Multiple allowed |
| 0x05 | 5 | Leaflet | Inside booklet pages | Multiple allowed |
| 0x06 | 6 | Media | Label side of CD/DVD | Multiple allowed |
| 0x07 | 7 | Lead artist | Lead performer/soloist | Multiple allowed |
| 0x08 | 8 | Artist | Artist/performer | Multiple allowed |
| 0x09 | 9 | Conductor | Conductor | Multiple allowed |
| 0x0A | 10 | Band/Orchestra | Band or orchestra | Multiple allowed |
| 0x0B | 11 | Composer | Composer | Multiple allowed |
| 0x0C | 12 | Lyricist | Lyricist/text writer | Multiple allowed |
| 0x0D | 13 | Recording Location | Recording location | Multiple allowed |
| 0x0E | 14 | During Recording | During recording | Multiple allowed |
| 0x0F | 15 | During Performance | During performance | Multiple allowed |
| 0x10 | 16 | Movie/Video Screen Capture | From movie/video | Multiple allowed |
| 0x11 | 17 | Bright Coloured Fish | Bright colored fish | Multiple allowed |
| 0x12 | 18 | Illustration | Illustration | Multiple allowed |
| 0x13 | 19 | Band/Artist Logotype | Band logotype | Multiple allowed |
| 0x14 | 20 | Publisher/Studio Logotype | Publisher/studio logo | Multiple allowed |

**Best practice:** Always use picture type `0x03` (Front cover) as the primary cover art. Many players and databases only recognize this type.

### 2.4 Text Encoding in APIC

| Encoding Byte | Charset | Description | Null Terminator |
|--------------|---------|-------------|----------------|
| 0x00 | ISO-8859-1 | Latin-1, 8-bit single-byte | 0x00 |
| 0x01 | UTF-16 LE | UTF-16 Little-Endian + BOM | 0x00 0x00 |
| 0x02 | UTF-16 BE | UTF-16 Big-Endian | 0x00 0x00 |
| 0x03 | UTF-8 | UTF-8 (recommended) | 0x00 |

**UTF-8 (0x03) is strongly recommended** for all APIC frames. It provides full Unicode support while remaining compatible with virtually all players.

**Description field rules:**
- Maximum recommended length: 64 characters (per ID3v2.3 spec)
- May be empty (zero-length description) — some players use the MIME type as fallback
- The description provides the "content descriptor" — multiple APIC frames must have unique descriptions

---

## 3. FLAC METADATA_BLOCK_PICTURE

### 3.1 FLAC Picture Block Specification

The FLAC format (per Xiph.org FLAC format specification) supports picture data in a `METADATA_BLOCK_PICTURE` metadata block. This format is also used by Vorbis Comment containers (Ogg Vorbis, Opus) via base64 encoding.

**Block type:** `METADATA_BLOCK_PICTURE` (type 6)

### 3.2 Complete Binary Layout

```
Offset    Size    Field                  Type            Description
-------   -----   -----                  ----            -----------
0x00      4B      Picture type          uint32 BE       0–20 (same as ID3v2 APIC)
0x04      4B      MIME length            uint32 BE       Length of MIME string
0x08      N       MIME type              string          Printable ASCII (0x20–0x7E), e.g., "image/jpeg"
0x08+N    4B      Description length    uint32 BE       Length of description string
0x0C+N    N       Description           string          UTF-8 encoded description
0x10+N    4B      Width                 uint32 BE       Image width in pixels
0x14+N    4B      Height                uint32 BE       Image height in pixels
0x18+N    4B      Color depth           uint32 BE       Bits per pixel (24 for JPEG RGB)
0x1C+N    4B      Colors used           uint32 BE       0 for non-indexed images, N for palette
0x20+N    4B      Picture data length   uint32 BE       Length of binary image data
0x24+N    M       Picture data          binary          Raw image bytes
```

**Total block size:** Sum of all fields above.

**Working example — FLAC front cover JPEG:**

```
METADATA_BLOCK_PICTURE:
[00 00 00 03]           Picture type = 3 (front cover)
[00 00 00 0A]           MIME length = 10
[69 6D 61 67 65 2F       "image/jpeg" (printable ASCII)
 6A 70 65 67]            (no null terminator — length is explicit)
[00 00 00 0C]           Description length = 12
[46 72 6F 6E 74          "Front cover" (UTF-8)
 20 63 6F 76 65 72]      (no null terminator)
[00 00 04 B0]           Width = 1200 pixels
[00 00 04 B0]           Height = 1200 pixels
[00 00 00 18]           Color depth = 24 bits per pixel
[00 00 00 00]           Colors used = 0 (RGB, not indexed)
[00 1D 4C 94]           Picture data length = 1923988 bytes (~1.9 MB)
[FF D8 FF E0 ...]       JPEG SOI + image data
[... binary JPEG ...]
[FF D9]                 JPEG EOI
```

### 3.3 FLAC Picture Block vs ID3v2 APIC — Differences

| Property | FLAC METADATA_BLOCK_PICTURE | ID3v2 APIC |
|----------|---------------------------|------------|
| Location | Metadata block (anywhere in stream) | ID3v2 tag (beginning of file) |
| MIME type | Mandatory, ASCII only | Mandatory, null-terminated string |
| Description | Mandatory, UTF-8, length-encoded | Optional, encoding-aware |
| Width/Height | Mandatory, in pixels | Not stored |
| Color depth | Mandatory | Not stored |
| Colors (palette) | Mandatory | Not stored |
| Encoding | Binary block, not base64 in native FLAC | Binary in frame body |
| URL linking | Supported (MIME="-->") | Supported (MIME="-->") |
| Uniqueness | Type + description must be unique | Type + description must be unique |

### 3.4 FLAC Picture Block Advantages

The FLAC picture block stores additional metadata (dimensions, color depth) that ID3v2 APIC does not. This enables:
- Players to display a thumbnail without fully decoding the image
- Searching for appropriate-sized images without scanning all
- Color depth information for display optimization

---

## 4. VORBIS COMMENT METADATA_BLOCK_PICTURE

### 4.1 Vorbis Comment Picture Storage

In Vorbis comments (used by Ogg Vorbis, Opus, and FLAC's Vorbis comment block), the picture data is stored using the `METADATA_BLOCK_PICTURE` field name. The picture block is **base64-encoded** before storage.

**Field name:** `METADATA_BLOCK_PICTURE` (case-insensitive per Vorbis spec)

**Field value:** Base64-encoded binary data of the FLAC METADATA_BLOCK_PICTURE structure

### 4.2 Base64 Encoding for Vorbis Comments

```python
import base64

# FLAC picture block (binary)
flac_picture = b'\x00\x00\x00\x03'  # picture type
flac_picture += b'\x00\x00\x00\x0A'  # MIME length
flac_picture += b'image/jpeg'        # MIME type
# ... rest of picture block ...

# Base64 encode for Vorbis comment
vorbis_field_value = base64.b64encode(flac_picture).decode('ascii')

# Store as: METADATA_BLOCK_PICTURE=<base64_string>
```

### 4.3 METADATA_BLOCK_PICTURE in FFmpeg

```bash
# Embed cover art in FLAC using FFmpeg
ffmpeg -i input.flac -i cover.jpg \
  -c copy \
  -map 0:a -map 1:v \
  -metadata:s:v title="Front cover" \
  -metadata:s:v comment="Cover (front)" \
  output.flac

# Extract cover art from FLAC
ffmpeg -i input.flac -an -codec:v copy cover_art.jpg

# Embed cover art in Vorbis/Ogg
ffmpeg -i input.ogg -i cover.jpg \
  -c copy \
  -map 0:a -map 1:v \
  -metadata:s:v title="Front cover" \
  output_with_cover.ogg

# Embed cover art in Opus
ffmpeg -i input.opus -i cover.jpg \
  -c copy \
  -map 0:a -map 1:v \
  -metadata:s:v title="Front cover" \
  output_with_cover.opus
```

### 4.4 Multiple Pictures in Vorbis Comments

Vorbis comments support multiple `METADATA_BLOCK_PICTURE` fields, one per picture. Each is a separate field with a unique base64-encoded picture block. Picture types follow the same numbering as ID3v2 APIC and FLAC.

---

## 5. MP4/M4A COVER ART — COVR ATOM

### 5.1 MP4 covr Atom Structure

MP4/M4A files store cover art in a `covr` atom located within the metadata hierarchy:

```
moov
 └── udta
      └── meta
           └── ilst
               └── covr
                    ├── data[0]   (first cover image)
                    ├── data[1]   (second cover image)
                    └── data[N]   (Nth cover image)
```

### 5.2 MP4 Atom Binary Format

The `covr` atom is a container atom. Each child `data` atom contains a single cover image.

**`data` atom layout (inside covr):**

```
Offset   Size   Field                Type            Description
------   ----   -----                ----            -----------
0x00     4B     Atom size            uint32 BE       Total atom size
0x04     4B     Atom type            char[4]         "data"
0x08     4B     Reserved             uint32 BE       0x00000000
0x0C     2B     Data type indicator  uint16 BE       0x0D = JPEG, 0x0E = PNG
0x0E     2B     Reserved             uint16 BE       0x0000
0x10     N      Image data          binary          Raw JPEG or PNG bytes
```

**Data type indicators for covr:**
| Value | Meaning |
|-------|---------|
| 0x0D | `mdta` — JPEG image data |
| 0x0E | `mdta` — PNG image data |

### 5.3 Complete MP4 covr Atom Example

```
covr atom:
  Size = 0x001D_4E98 (1923992 bytes including header)
  Type = "covr"

  data[0] atom:
    [00 1D 4E 94]           Atom size = 1923988 bytes
    [64 61 74 61]           "data"
    [00 00 00 00]           Reserved
    [00 0D]                 Data type = JPEG (0x0D)
    [00 00]                 Reserved
    [FF D8 FF E0 ...]       JPEG SOI + image data
    [... binary JPEG ...]
    [FF D9]                 JPEG EOI
```

### 5.4 Multiple Covers in MP4

MP4 supports multiple cover images in the `covr` atom. Each is a separate `data` child atom. The order in the file may indicate priority, but most players prefer picture type 3 (Front cover).

```bash
# Embed multiple cover images in MP4
ffmpeg -i input.m4a -i front_cover.jpg -i back_cover.jpg \
  -c copy \
  -map 0:a -map 1:v -map 2:v \
  -metadata:s:v:0 title="Front cover" \
  -metadata:s:v:0 comment="Cover (front)" \
  -metadata:s:v:1 title="Back cover" \
  -metadata:s:v:1 comment="Cover (back)" \
  output.m4a
```

---

## 6. OTHER CONTAINER FORMATS

### 6.1 AIFF Cover Art — ID3 Chunk

AIFF files can store ID3v2 tags (including APIC frames) in an `ID3` chunk at the beginning of the file, after the FORM header and before the Audio Data chunk:

```
FORM chunk
 ├── ID3 chunk (APIC frame here)
 ├── COMM chunk (audio metadata)
 ├── SSND chunk (audio data)
 └── ANNO chunk / NAME chunk / other chunks
```

```bash
# Embed cover art in AIFF
ffmpeg -i input.aiff -i cover.jpg \
  -c copy \
  -map 0:a -map 1:v \
  -metadata:s:v title="Front cover" \
  output_with_art.aiff
```

### 6.2 WAV Cover Art — RIFF INFO JPEG

Standard WAV files (RIFF/WAVE) do not natively support embedded cover art. Options include:

**Option 1:** Store JPEG in a separate LIST INFO chunk:
```
RIFF header
 └── LIST "INFO"
      └── "IJPG" chunk (rare, non-standard)
```

**Option 2:** Use a `disposition:attached_pic` in FFmpeg (stored in output container, not WAV):
```bash
# FFmpeg embeds cover art in the output container, not as WAV RIFF data
# For MP3 output from WAV + cover:
ffmpeg -i input.wav -i cover.jpg \
  -c:a copy -c:v copy \
  output.mp3  # Cover art stored in ID3v2 APIC
```

**Option 3:** WAV supports `LIST INFO` with a custom `ICRD` or `ICMT` text field, but not binary image data.

### 6.3 WMA/ASF Cover Art — WM/Picture

WMA/ASF files store cover art as a binary attribute in the header:

**WM/Picture attribute structure:**

```
Offset   Size   Field              Type              Description
------   ----   -----              ----              -----------
0x00     1B     Picture type       uint8             0=other, 3=front cover, etc.
N+1      4B     MIME type length   uint32 LE         Length of MIME string
N+5      N      MIME type          string            e.g., "image/jpeg"
N+5+N    4B     Description length uint32 LE         Length of description
N+9+N    N      Description       string            UTF-16 LE
N+9+2N   4B     Picture data len   uint32 LE         Length of image binary
N+13+2N  M      Picture data       binary            JPEG/PNG bytes
```

### 6.4 DSF (DSD Stream File) Cover Art

DSF files use an ID3v2 tag at the beginning of the file, identical to MP3:
```bash
# Embed cover art in DSF
ffmpeg -i input.dsf -i cover.jpg \
  -c copy -map 0:a -map 1:v \
  -metadata:s:v title="Front cover" \
  output.dsf
```

---

## 7. COVER ART SIZE LIMITS & PRACTICAL CONSTRAINTS

### 7.1 Size Limits by Format

| Format | Theoretical Max | Practical Limit | Notes |
|--------|-----------------|------------------|-------|
| ID3v2.3/2.4 (MP3) | ~256 MB per frame | 1–5 MB recommended | Players may truncate larger |
| FLAC | Unlimited (limited by file) | 5–10 MB recommended | FLAC is efficient with large blocks |
| Vorbis (OGG) | ~4 GB per comment field | 1–5 MB recommended | Base64 adds 33% overhead |
| MP4/M4A | No explicit limit | 5–10 MB | iTunes typically handles up to 10 MB |
| AIFF | Limited by ID3 chunk | 1–5 MB recommended | ID3v2 chunk before audio data |
| WAV | Not natively supported | N/A | Use sidecar file or container wrapper |

### 7.2 Large Image Problems

**Players that may fail or misbehave with large cover art:**
- Older versions of Windows Media Player: struggles with >500 KB
- Some car audio systems: limited parsing of large metadata
- Streaming services: may reject files >10 MB
- Mobile apps: memory limits on low-end devices

**Best practice:** Keep cover art under 2 MB for maximum compatibility.

### 7.3 Image Dimension Recommendations

| Use Case | Recommended Dimensions | Typical File Size |
|----------|----------------------|-------------------|
| Thumbnail/preview | 160×160 to 300×300 px | 20–50 KB |
| Standard display | 600×600 px | 100–300 KB |
| High-quality (desktop) | 1200×1200 px | 300 KB – 1 MB |
| Print quality | 2000×2000 to 4000×4000 px | 1–5 MB |
| Maximum quality | 5000×5000 px | 3–10 MB |

### 7.4 Image Optimization

```bash
# Optimal JPEG compression for cover art
# Using ImageMagick to resize and compress:
convert input.png -resize 1200x1200\> \
  -quality 85 \
  -sampling-factor 2x2 \
  -strip \
  -interlace Plane \
  cover.jpg

# FFmpeg can extract, but not resize — use ImageMagick first:
ffmpeg -i cover.jpg -frames:v 1 cover_small.jpg

# Verify JPEG quality
ffprobe -v quiet -show_entries stream=width,height,bit_rate cover.jpg
```

---

## 8. FFMPEG COVER ART OPERATIONS

### 8.1 Embedding Cover Art with FFmpeg

```bash
# Basic embedding in any container that supports it
ffmpeg -i audio_source -i cover_image.jpg \
  -c copy \
  -map 0:a -map 1:v \
  output_with_cover.ext

# Specify picture type and description
ffmpeg -i audio_source -i cover_image.jpg \
  -c copy \
  -map 0:a -map 1:v \
  -metadata:s:v title="Album cover" \
  -metadata:s:v comment="Cover (front)" \
  output_with_cover.ext

# Set disposition to attached_pic (ensures it's the primary cover)
ffmpeg -i audio_source -i cover_image.jpg \
  -c copy \
  -map 0:a -map 1:v \
  -disposition:v:0 attached_pic \
  output_with_cover.ext

# Embed with specific picture type (type 3 = front cover)
ffmpeg -i audio_source -i cover_image.jpg \
  -c copy \
  -map 0:a -map 1:v \
  -disposition:v:0 attached_pic \
  output_with_cover.ext
```

### 8.2 Cover Art Format Support in FFmpeg Muxers

| Container | Cover Art Support | Format | FFmpeg Flag |
|-----------|------------------|--------|-------------|
| MP3 (ID3v2) | Yes | JPEG, PNG, GIF | -map 1:v |
| FLAC | Yes | JPEG, PNG | -map 1:v |
| OGG/Vorbis | Yes (METADATA_BLOCK_PICTURE) | JPEG, PNG | -map 1:v |
| Opus in OGG | Yes (METADATA_BLOCK_PICTURE) | JPEG, PNG | -map 1:v |
| MP4/M4A | Yes (covr atom) | JPEG, PNG | -map 1:v |
| Matroska/MKA | Yes (attached picture) | Any | -map 1:v |
| WebM | Yes | JPEG, PNG, WebP | -map 1:v |
| WAV | No native support | N/A | Not directly supported |
| AIFF | Yes (ID3 in ID3 chunk) | JPEG, PNG | -map 1:v |
| DSF | Yes (ID3v2) | JPEG, PNG | -map 1:v |

### 8.3 FFmpeg -disposition Option

The `-disposition` option controls which stream is the "primary" or "default" stream of its type:

```bash
# Primary (attached pic) cover — always shown
-disposition:v:0 attached_pic

# No disposition — secondary/backup cover
-disposition:v:0 0

# Default stream — may be shown in some players
-disposition:v:0 default

# Multiple covers — one attached_pic, others with no disposition
ffmpeg -i input.flac -i front.jpg -i back.jpg -i inside.jpg \
  -c copy \
  -map 0:a -map 1:v -map 2:v -map 3:v \
  -disposition:v:0 attached_pic \
  -disposition:v:1 0 \
  -disposition:v:2 0 \
  output.flac
```

### 8.4 Extracting Cover Art with FFmpeg

```bash
# Extract all attached pictures (best quality)
ffmpeg -i input.mp3 -an -codec:v copy cover_%03d.jpg

# Extract first (primary) cover as single file
ffmpeg -i input.flac -an -codec:v copy cover.jpg

# Extract using ffprobe (find picture stream)
ffprobe -v quiet -show_entries stream=codec_type,codec_name,width,height \
  -show_entries format=filename input.mp3

# Extract with kid3-cli
kid3-cli -c "get picture:0" "input.mp3"
kid3-cli -c "export picture:0 cover.jpg" "input.mp3"
```

### 8.5 Cover Art with Metadata Stripping

```bash
# Embed cover art and copy all other tags
ffmpeg -i input.mp3 -i cover.jpg \
  -c copy \
  -map 0:a -map 1:v \
  -map_metadata 0 \
  output_with_cover.mp3

# Embed cover art and strip all other metadata
ffmpeg -i input.mp3 -i cover.jpg \
  -c copy \
  -map 0:a -map 1:v \
  -map_metadata -1 \
  -metadata title="New Title" \
  -metadata artist="New Artist" \
  output.mp3
```

---

## 9. COVER ART EXTRACTION & VERIFICATION

### 9.1 Using ffprobe

```bash
# List all streams including picture streams
ffprobe -v quiet -print_format json \
  -show_streams -show_format input.mp3 | jq '.streams[] | select(.codec_type=="video")'

# Output example:
# {
#   "index": 3,
#   "codec_name": "mjpeg",
#   "codec_long_name": "Motion JPEG",
#   "width": 1200,
#   "height": 1200,
#   "coded_width": 1200,
#   "coded_height": 1200,
#   "has_b_frames": 0,
#   "pix_fmt": "yuvj420p",
#   "nb_frames": 1
# }
```

### 9.2 Using kid3-cli

```bash
# Read cover art info from MP3
kid3-cli -c "get" "input.mp3"
# Output includes:
#   [APIC] description: Front cover, MIME: image/jpeg, size: 1923988

# Export cover art
kid3-cli -c "export picture:0 cover.jpg" "input.mp3"

# Read cover art info from FLAC
kid3-cli -c "get" "input.flac"
# Output includes:
#   [PICTURE] type: Front cover, MIME: image/jpeg, width: 1200, height: 1200, size: 1923988

# Read cover art info from MP4
kid3-cli -c "get" "input.m4a"
# Output includes:
#   [covr] format: JPEG, size: 1923988
```

### 9.3 Using ffmetadata

```bash
# Extract all metadata as text file
ffmpeg -i input.mp3 -f ffmetadata metadata.txt
cat metadata.txt

# Import metadata with cover art
ffmpeg -i input.mp3 -i cover.jpg \
  -i metadata.txt \
  -map_metadata 2 \
  -map 0:a -map 1:v \
  -c copy \
  output_with_cover.mp3
```

---

## 10. MULTIPLE PICTURES — HANDLING PRIORITIES

### 10.1 Player Behavior with Multiple Covers

| Player | Priority Order | Notes |
|--------|---------------|-------|
| iTunes/Apple Music | Front cover (type 3) first | Falls back to any picture |
| Windows Media Player | First picture in file | May not respect type |
|foobar2000|Front cover (type 3) if present | Falls back to any picture |
| Spotify | Front cover (type 3) | Uses embedded or folder.jpg |
| Android (built-in) | Front cover (type 3) | Varies by app |
| Car audio | Varies widely | Many only support one cover |
| Streaming services | Front cover (type 3) | Reject or ignore others |

### 10.2 Recommended Priority Scheme

For files with multiple embedded pictures:

1. **Front cover (type 3)** — Primary, must always be present
2. **Back cover (type 4)** — Optional, secondary
3. **Leaflet/media (type 5/6)** — Optional, tertiary
4. **Artist/conductor (type 7–9)** — Optional, may not be displayed

**Best practice for converters:**
1. Preserve the primary front cover
2. If converting between formats, map FLAC PICTURE → MP4 covr → Vorbis METADATA_BLOCK_PICTURE
3. Never embed the same image multiple times as different types
4. Remove duplicate covers of the same image
5. Resize very large images (>5 MB) before embedding

### 10.3 Cover Art in Folder.jpg Conventions

Many players and media servers look for `folder.jpg`, `cover.jpg`, or `album.jpg` in the same directory as audio files, even when no embedded art is present. This is separate from embedded art — it's a sidecar file convention:

| Filename | Priority | Used By |
|----------|----------|---------|
| folder.jpg | Highest | iTunes, Plex, UPnP/DLNA servers |
| cover.jpg | High | Various players |
| album.jpg | Medium | Some Linux players |
| front.jpg | Medium | Foobar2000 (configurable) |
| 000.jpg | Low | Scan for any .jpg in folder |

FFmpeg does not automatically generate sidecar files. Use ImageMagick to create them:
```bash
# Extract and create sidecar folder.jpg
ffmpeg -i input.mp3 -an -codec:v copy cover_extracted.jpg
cp cover_extracted.jpg folder.jpg
```

---

## 11. MIME TYPES IN COVER ART

### 11.1 Valid MIME Types

| MIME Type | Format | FLAC/OGG | MP4 | ID3v2 APIC | Notes |
|-----------|--------|---------|-----|------------|-------|
| image/jpeg | JPEG/JFIF | Yes | Yes | Yes | **Recommended** |
| image/png | PNG | Yes | Yes | Yes | Use for transparency |
| image/gif | GIF | Rare | Rare | Rare | 256 colors, avoid |
| image/bmp | BMP | Rare | Rare | Rare | Very large, avoid |
| --> | URL link | Yes | No | Yes | External link (discouraged) |

### 11.2 MIME Type Edge Cases

**ID3v2:** If MIME type is omitted, `image/` is implied. Only `image/png` and `image/jpeg` are standard-compliant.

**FLAC:** MIME type is mandatory and must be printable ASCII (0x20–0x7E). The value `-->` denotes a URL to the image file (external linking).

**MP4:** The covr data atom uses type indicators 0x0D (JPEG) and 0x0E (PNG). PNG is stored as raw PNG data; JPEG is stored as raw JPEG data.

---

## 12. BINARY LAYOUT — COMPLETE REFERENCE

### 12.1 ID3v2.4 APIC Frame (Complete Hex Dump Example)

```
Filename: test_with_cover.mp3

APIC Frame:
  [41 50 49 43]           Frame ID "APIC"
  [00 1D 4E 94]           Frame size = 1923988 bytes (big-endian)
  [00 00]                 Flags = 0

  Frame body:
  [03]                    Text encoding = UTF-8
  [69 6D 61 67 65 2F      "image/jpeg"
   6A 70 65 67 00]         + null terminator
  [03]                    Picture type = Front cover (0x03)
  [46 72 6F 6E 74 20      "Front cover"
   63 6F 76 65 72 00]       + null terminator
  [FF D8 FF E0 ...]        JPEG SOI (Start Of Image)
  [... 1923901 bytes ...]
  [... JPEG image data ...]
  [FF D9]                 JPEG EOI (End Of Image)
```

### 12.2 FLAC METADATA_BLOCK_PICTURE (Complete Hex Dump Example)

```
FLAC METADATA_BLOCK_PICTURE (type 6):
  [00 00 00 06]           Block type = 6 (PICTURE)
  [00 1D 4E 88]           Block length = 1923976 bytes

  Block data:
  [00 00 00 03]           Picture type = 3 (front cover)
  [00 00 00 0A]           MIME length = 10
  [69 6D 61 67 65 2F      "image/jpeg"
   6A 70 65 67]            (ASCII, no null terminator)
  [00 00 00 0C]           Description length = 12
  [46 72 6F 6E 74 20      "Front cover"
   63 6F 76 65 72]         (UTF-8, no null terminator)
  [00 00 04 B0]           Width = 1200 pixels
  [00 00 04 B0]           Height = 1200 pixels
  [00 00 00 18]           Color depth = 24 bits per pixel
  [00 00 00 00]           Colors used = 0 (RGB)
  [00 1D 4E 5C]           Picture data length = 1923932 bytes
  [FF D8 FF E0 ...]       JPEG SOI
  [... JPEG binary data ...]
  [FF D9]                 JPEG EOI
```

---

## 13. COVER ART IMPLEMENTATION CHECKLIST

### 13.1 Embedding Cover Art

- [ ] Image is JPEG or PNG (JPEG preferred for photos, PNG for logos/transparency)
- [ ] Image is resized to appropriate dimensions (600×600 to 2000×2000)
- [ ] Image file size is under 5 MB (preferably under 2 MB)
- [ ] Picture type is set to 0x03 (front cover) for primary image
- [ ] MIME type is correctly set (`image/jpeg` or `image/png`)
- [ ] Text encoding is UTF-8 (0x03 for ID3v2 APIC)
- [ ] Description field is present and non-empty for primary cover
- [ ] Multiple covers have unique picture types and descriptions
- [ ] `disposition:v attached_pic` is set for primary cover

### 13.2 Extracting Cover Art

- [ ] Use `-an -codec:v copy` to extract without transcoding
- [ ] Verify extracted image is valid JPEG/PNG by opening it
- [ ] Check dimensions match embedded metadata
- [ ] Use `kid3-cli` for detailed inspection if ffprobe is insufficient

### 13.3 Format Conversion

- [ ] FLAC PICTURE block → MP3 APIC frame: map all fields correctly
- [ ] FLAC PICTURE block → MP4 covr: convert to JPEG/PNG, set type indicator
- [ ] MP4 covr → Vorbis METADATA_BLOCK_PICTURE: base64 encode
- [ ] ID3v2 APIC → FLAC PICTURE: set all required fields, no base64 encoding
- [ ] Preserve front cover as primary during all conversions
- [ ] Remove duplicate covers from different sources

---

*File generated for: Audio Engineering Knowledge Base*
*Topic: Embedded Cover Art, APIC, METADATA_BLOCK_PICTURE, covr Atom*
*[NEEDS VERIFICATION] markers indicate specific numerical values that require additional source confirmation*
*[NEEDS VERIFICATION]: The WMA/ASF WM/Picture binary format — the structure described is based on typical ASF_PICTURE object layouts but should be verified against the ASF specification document*
