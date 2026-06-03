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

## 14. ADVANCED COVER ART OPERATIONS

### 14.1 Converting Between Cover Art Formats

**FLAC PICTURE → MP4 covr conversion:**

```python
import struct

def flac_picture_to_mp4_covr(flac_picture_data):
    """Convert FLAC METADATA_BLOCK_PICTURE to MP4 covr data atom."""
    offset = 0

    # Parse FLAC picture block
    picture_type = struct.unpack('>I', flac_picture_data[offset:offset+4])[0]; offset += 4
    mime_len = struct.unpack('>I', flac_picture_data[offset:offset+4])[0]; offset += 4
    mime = flac_picture_data[offset:offset+mime_len].decode('ascii'); offset += mime_len
    desc_len = struct.unpack('>I', flac_picture_data[offset:offset+4])[0]; offset += 4
    desc = flac_picture_data[offset:offset+desc_len].decode('utf-8', errors='replace'); offset += desc_len
    width = struct.unpack('>I', flac_picture_data[offset:offset+4])[0]; offset += 4
    height = struct.unpack('>I', flac_picture_data[offset:offset+4])[0]; offset += 4
    depth = struct.unpack('>I', flac_picture_data[offset:offset+4])[0]; offset += 4
    colors = struct.unpack('>I', flac_picture_data[offset:offset+4])[0]; offset += 4
    image_len = struct.unpack('>I', flac_picture_data[offset:offset+4])[0]; offset += 4
    image_data = flac_picture_data[offset:offset+image_len]

    # Build MP4 data atom
    if mime == 'image/jpeg':
        data_type = 0x0D  # JPEG
    elif mime == 'image/png':
        data_type = 0x0E  # PNG
    else:
        raise ValueError(f"Unsupported MIME type: {mime}")

    atom = bytearray()
    atom += struct.pack('>I', 8 + len(image_data))  # Atom size
    atom += b'data'  # Atom type
    atom += struct.pack('>I', 0x00000000)  # Reserved
    atom += struct.pack('>H', data_type)  # Data type
    atom += struct.pack('>H', 0x0000)  # Reserved
    atom += image_data

    return bytes(atom)
```

**ID3v2 APIC → FLAC PICTURE conversion:**

```python
def apic_to_flac_picture(apic_frame_body, mime_type, description,
                          picture_type, width, height, depth, colors):
    """Convert ID3v2 APIC frame body to FLAC METADATA_BLOCK_PICTURE."""
    picture_block = bytearray()
    picture_block += struct.pack('>I', picture_type)  # Picture type
    picture_block += struct.pack('>I', len(mime_type))  # MIME length
    picture_block += mime_type.encode('ascii')  # MIME type
    picture_block += struct.pack('>I', len(description))  # Description length
    picture_block += description.encode('utf-8')  # Description
    picture_block += struct.pack('>I', width)  # Width
    picture_block += struct.pack('>I', height)  # Height
    picture_block += struct.pack('>I', depth)  # Color depth
    picture_block += struct.pack('>I', colors)  # Colors used
    picture_block += struct.pack('>I', len(apic_frame_body))  # Image data length
    picture_block += apic_frame_body  # Image data
    return bytes(picture_block)
```

### 14.2 Batch Cover Art Operations

```bash
#!/bin/bash
# Extract cover art from all FLAC files and save as folder.jpg

for f in *.flac; do
    ffmpeg -y -i "$f" -an -codec:v copy "cover_$(basename "$f" .flac).jpg" 2>/dev/null
    # Use first found cover as folder.jpg
    if [ -f "cover_$(basename "$f" .flac).jpg" ] && [ ! -f "folder.jpg" ]; then
        cp "cover_$(basename "$f" .flac).jpg" "folder.jpg"
    fi
done

# Clean up individual covers
rm -f cover_*.jpg
```

```python
# Python batch processing
from pathlib import Path
import subprocess

def embed_cover_to_all(folder_path, cover_image):
    """Embed cover art into all audio files in a folder."""
    folder = Path(folder_path)
    cover = Path(cover_image)

    for audio_file in folder.glob('*.flac'):
        subprocess.run([
            'ffmpeg', '-y', '-i', str(audio_file),
            '-i', str(cover),
            '-c', 'copy',
            '-map', '0:a', '-map', '1:v',
            '-metadata:s:v', 'title=Front cover',
            '-metadata:s:v', 'comment=Cover (front)',
            '-disposition:v', 'attached_pic',
            str(audio_file.with_suffix('.tmp')),
        ], check=True)
        audio_file.with_suffix('.tmp').replace(audio_file)
```

### 14.3 Resizing Large Cover Art

```bash
# Using ImageMagick to resize cover art for optimal quality/size
# Create multiple sizes for different purposes

# Full quality (for archival)
convert input.png \
  -resize 2000x2000\> \
  -quality 90 \
  -sampling-factor 2x2 \
  cover_full.jpg

# Medium quality (for general use)
convert input.png \
  -resize 600x600\> \
  -quality 85 \
  -sampling-factor 2x2 \
  cover_medium.jpg

# Thumbnail (for quick browsing)
convert input.png \
  -resize 160x160\> \
  -quality 80 \
  cover_thumb.jpg

# Optimal FFmpeg-compatible JPEG settings:
# -colorspace sRGB
# -sampling-factor 2x2 (4:2:0 chroma subsampling for smaller file)
# -quality 85 (balance quality/size)
# -interlace Plane (progressive JPEG)
```

### 14.4 Verifying Cover Art Integrity

```bash
# Verify JPEG is valid
ffmpeg -v error -i cover.jpg -f null - 2>&1 | grep -i error

# Verify PNG is valid
ffmpeg -v error -i cover.png -f null - 2>&1 | grep -i error

# Check image dimensions and format
ffprobe -v quiet -show_entries stream=width,height,codec_name \
  -of default=noprint_wrappers=1 cover.jpg

# Check file size
ls -la cover.jpg

# Verify cover art with kid3
kid3-cli -c "get" input.flac | grep -i picture
```

---

## 15. COVER ART IN STREAMING SERVICES

### 15.1 Streaming Service Requirements

| Service | Format | Max Size | Max Dimensions | Notes |
|---------|--------|----------|---------------|-------|
| Spotify | JPEG | 5 MB | 4096×4096 | Must be embedded or via API |
| Apple Music | JPEG/PNG | 10 MB | 6000×6000 | Embed in AAC or ALAC |
| YouTube Music | JPEG | 5 MB | 4096×4096 | Via API upload |
| Amazon Music | JPEG | 5 MB | 4000×4000 | Embed or via API |
| Tidal | JPEG/PNG | 5 MB | 4096×4096 | Embed in source file |
| Deezer | JPEG | 5 MB | 4000×4000 | Via API or embed |

### 15.2 Streaming Cover Art Best Practices

1. **Resolution:** 3000×3000 pixels is the sweet spot for most services
2. **Format:** JPEG at 85–90% quality provides best balance
3. **Color space:** sRGB (most players don't support wide-gamut cover art)
4. **Embedded:** Always embed cover art in the source file
5. **Separate upload:** Also upload to the streaming platform's API/portal
6. **Front cover:** Always use picture type 3 (Front cover) as primary

### 15.3 Cover Art Metadata for Streaming

When delivering to streaming services, include:
- **Title/Description:** "Front cover" or album name
- **Copyright:** Include if applicable
- **Dimensions:** Ensure metadata matches actual image dimensions
- **Color depth:** 24-bit (JPEG RGB) recommended

### 15.4 Common Streaming Artwork Errors

| Error | Cause | Solution |
|-------|-------|----------|
| Artwork not appearing | Missing or incorrect MIME type | Set correct MIME type |
| Artwork rejected | File too large | Resize to < 5 MB |
| Artwork rejected | Dimensions too large | Resize to < 4096×4096 |
| Artwork rejected | Invalid JPEG (truncated) | Re-encode from source |
| Artwork blurry | Resolution too low | Use 1000×1000 minimum |
| Artwork cropped | Aspect ratio mismatch | Use square images |

---

## 16. COVER ART ACCESSIBILITY

### 16.1 Alt Text for Cover Art

Currently, no audio format supports alt text descriptions for cover art (for visually impaired users). However:
- The description field in APIC/PICTURE can serve as a basic description
- MusicBrainz Picard uses the description field for the cover art comment
- Some metadata editors allow storing descriptive text

### 16.2 Accessible Audio Description

For music services, accessibility is typically handled at the application level:
- Screen readers can read song titles and artist names (metadata)
- Cover art descriptions can be provided via separate APIs
- This is a gap in current audio metadata standards

---

## 17. COVER ART FORMATS DEEP DIVE

### 17.1 JPEG/JFIF Technical Details

**JPEG marker sequence:**
```
FF D8  (SOI - Start Of Image)
FF E0  (APP0 - JFIF marker)
  ... JFIF data ...
FF E1  (APP1 - EXIF/XMP metadata, optional)
  ... EXIF/XMP data ...
FF DA  (SOS - Start Of Scan)
  ... compressed image data ...
FF D9  (EOI - End Of Image)
```

**JFIF (JPEG File Interchange Format) APP0 marker:**
```
Bytes 0-1: APP0 marker (FF E0)
Bytes 2-3: Length (total APP0 length)
Bytes 4-7: Identifier "JFIF\0"
Bytes 8-9: Version (e.g., 01 02 = 1.2)
Byte 10:   Units (0=no units, 1=inch, 2=cm)
Bytes 11-12: X density
Bytes 13-14: Y density
Byte 15:   Thumbnail width (0 if no thumbnail)
Byte 16:   Thumbnail height
Bytes 17+: Thumbnail data (optional)
```

**Color space:** JPEG/JFIF uses YCbCr (Y' = luma, Cb/Cr = chroma). Cover art should use sRGB color space, which JFIF specifies via the marker structure.

### 17.2 PNG Technical Details

**PNG chunks:**
```
89 50 4E 47 0D 0A 1A 0A  (PNG signature)
IHDR chunk (image header)
PLTE chunk (palette, optional)
IDAT chunks (image data, one or more)
IEND chunk (end of image)
```

**IHDR chunk:**
```
Length: 13 bytes
Type: "IHDR"
Width: 4 bytes (unsigned int)
Height: 4 bytes
Bit depth: 1 byte (1, 2, 4, 8, or 16)
Color type: 1 byte
  0 = grayscale
  2 = RGB
  3 = palette
  4 = grayscale + alpha
  6 = RGBA
Compression: 1 byte (always 0 = deflate)
Filter: 1 byte (always 0 = adaptive)
Interlace: 1 byte (0=no interlace, 1=Adam7)
```

**PNG for cover art:** Most cover art is RGB (color type 2) or RGBA (color type 6, for transparency). Bit depth 8 is standard.

### 17.3 WebP Support

Some newer containers (WebM, newer MP4 muxers) support WebP as cover art:
```bash
# FFmpeg can extract WebP as cover art
ffmpeg -i input.webm -an -codec:v copy cover.webp
```

WebP offers:
- Better compression than JPEG at equivalent quality
- Lossless mode available
- Alpha channel support (unlike JPEG)
- Not universally supported in all audio players

### 17.4 HEIF/HEIC Support

Apple devices may produce HEIF/HEIC cover art. Most audio players do not support HEIF cover art:
- Convert to JPEG before embedding
- Use ImageMagick: `convert input.heic cover.jpg`

---

## 18. COVER ART IN SPECIFIC AUDIO FORMATS

### 18.1 WavPack Cover Art

WavPack (.wv) supports embedded cover art via a special metadata chunk:
```bash
# Embed cover art in WavPack
wvtag -i cover.jpg album.wv
# Extract cover art
wvtag -r album.wv
```

### 18.2 TAK Cover Art

TAK (.tak) supports embedded pictures via a dedicated metadata field:
```bash
# TAK Tools for tag manipulation
takcreator -tags "TITLE=Track" input.wav output.tak
takcreator -addcover cover.jpg output.tak
```

### 18.3 OptimFROG Cover Art

OptimFROG (.ofr, .ofs) supports ID3v2 tags for cover art when saved as AFOB format.

### 18.4 Musepack Cover Art

Musepack (.mpc) uses SV8 metadata with APIC-equivalent frames.

### 18.5 AMR Cover Art

AMR files (.amr) do not support embedded cover art. Cover art must be delivered as a separate sidecar file.

### 18.6 DTS Coherent Acoustics Cover Art

DTS-CD and DTS audio files typically store cover art in a separate ID3v2 header before the DTS data.

---

## 19. COVER ART AND MUSIC RECOGNITION

### 19.1 Cover Art Fingerprinting

Music recognition services (Shazam, SoundHound) use audio fingerprinting, not cover art. However:
- Cover art helps users confirm the correct match visually
- Inconsistent cover art across releases can confuse users
- Some Gracenote/CDDB databases use cover art as a matching signal

### 19.2 MusicBrainz Cover Art Archive

MusicBrainz maintains a Cover Art Archive (coverartarchive.org):
- Hosts cover art for releases in their database
- Provides API for fetching cover art
- Artist can upload high-resolution images
- CDN-delivered for fast access

### 19.3 Cover Art Licensing

Cover art has complex copyright issues:
- Album covers are typically copyrighted by the label/publisher
- Embedding in digital files may require license
- Some artists/designers retain rights
- Stock photo licenses may cover some covers

---

## 20. COVER ART QUALITY ASSESSMENT

### 20.1 Quality Scoring

| Factor | Weight | Assessment Criteria |
|--------|--------|-------------------|
| Resolution | 30% | ≥ 1000×1000 = excellent, 500–1000 = good, < 500 = poor |
| Compression quality | 25% | JPEG quality ≥ 85 = excellent, 70–85 = good, < 70 = poor |
| Color accuracy | 20% | Matches original artwork |
| Crop correctness | 15% | Full cover visible, no truncation |
| File format | 10% | JPEG or PNG preferred |

### 20.2 Common Cover Art Quality Problems

| Problem | Cause | Impact |
|---------|-------|--------|
| Pixelation | Low resolution, upscaled | Blurry appearance |
| Compression artifacts | Low JPEG quality | Blocking, color banding |
| Wrong colors | CMYK source saved as sRGB | Colors appear off |
| Cropped edges | Scanned at angle | Missing content |
| Text unreadable | Resolution too low | Quality perception |
| Visible barcode | Poor source image | Unprofessional appearance |

### 20.3 Recommended Quality Workflow

```
1. Acquire highest resolution source available
   ├── Master artwork file from label
   ├── High-res scan of physical CD booklet
   └── Professional photography

2. Pre-process
   ├── Convert to RGB color space
   ├── Correct aspect ratio to square
   ├── Remove extraneous content
   └── Crop to square if needed

3. Export at target quality
   ├── Format: JPEG (photos) or PNG (graphics)
   ├── Size: 3000×3000 pixels maximum
   ├── Quality: 85–90 for JPEG
   └── File size: under 5 MB

4. Verify
   ├── Open in image viewer at actual size
   ├── Check for compression artifacts
   └── Verify colors match original
```

---

## 21. APPENDIX: COVER ART QUICK REFERENCE

### 21.1 Format Support Matrix

| Container | Supports Embedded | Format | Multiple | FFmpeg |
|-----------|----------------|--------|---------|--------|
| MP3 | Yes | JPEG, PNG | Yes | ✓ |
| FLAC | Yes | JPEG, PNG | Yes | ✓ |
| OGG Vorbis | Yes | JPEG, PNG | Yes | ✓ |
| Opus | Yes | JPEG, PNG | Yes | ✓ |
| AAC (MP4) | Yes | JPEG, PNG | Yes | ✓ |
| ALAC (MP4) | Yes | JPEG, PNG | Yes | ✓ |
| WAV | No | — | — | ✗ |
| AIFF | Yes | JPEG, PNG | Yes | ✓ |
| WMA | Yes | JPEG, PNG | Yes | ✓ |
| DSF | Yes | JPEG, PNG | Yes | ✓ |
| Matroska | Yes | Any image | Yes | ✓ |
| WebM | Yes | JPEG, PNG | Yes | ✓ |

### 21.2 Recommended Cover Art Settings

| Use Case | Resolution | Format | Quality | Size |
|----------|------------|--------|---------|------|
| Streaming | 3000×3000 | JPEG | 90% | 1–3 MB |
| CD distribution | 1200×1200 | JPEG | 85% | 200–500 KB |
| Archive master | 3000×3000 | PNG | lossless | 5–20 MB |
| Portable player | 600×600 | JPEG | 80% | 50–100 KB |
| Car audio | 500×500 | JPEG | 80% | 30–80 KB |

### 21.3 Troubleshooting Cover Art

| Problem | Solution |
|---------|---------|
| Cover not showing in player | Verify picture type 3 (front cover) |
| Cover shows in some players but not others | Check format compatibility |
| Multiple covers showing wrong one | Set disposition:attached_pic |
| Cover appears blank in iTunes | Check MIME type is correct |
| Cover appears in browser but not app | Some apps require exact picture type |
| Large file size from cover | Resize and re-compress |

---

## 22. COVER ART IN SPECIFIC ECOSYSTEMS

### 22.1 iTunes/Apple Music Cover Art

iTunes and Apple Music display cover art embedded in:
- MP3 files (ID3v2 APIC)
- AAC/M4A files (MP4 covr atom)
- ALAC files (MP4 covr atom)
- Audio books (MP4/M4B)

**iTunes-specific behavior:**
- Prefers picture type 0x03 (front cover)
- Will display the first APIC if no type 3 is present
- JPEG preferred over PNG for compatibility
- Minimum recommended size: 600×600 pixels
- Maximum accepted size: 6000×6000 pixels
- Cover art is cached at multiple resolutions

**iTunes artwork extraction:**
```bash
# Extract artwork from iTunes-purchased files
ffmpeg -i "iTunes file.m4a" -an -codec:v copy cover.jpg

# Note: Some iTunes Plus files have DRM that prevents extraction
```

### 22.2 Spotify Cover Art

Spotify displays cover art via:
1. Embedded cover art in the audio file
2. Cover art from the Spotify CDN (uploaded during distribution)
3. Cover art from MusicBrainz Cover Art Archive

**Spotify behavior:**
- Embedded cover art in uploaded files is used if no CDN art is available
- CDN art takes priority over embedded art
- Supports JPEG and PNG
- Preferred size: 640×640 to 2048×2048 pixels
- Album art is used for:
  - Spotify app display
  - Voice assistant queries ("play [album name] by [artist]")
  - Spotify Wrapped

### 22.3 Google Play Music / YouTube Music

Google/YouTube music services:
- Display embedded cover art from uploaded files
- Use CDN-hosted art for distributed tracks
- Support JPEG and PNG
- Preferred size: 512×512 minimum, 1600×1600 recommended
- Some services require square images (1:1 aspect ratio)

### 22.4 Amazon Music

Amazon Music:
- Supports embedded cover art in MP3, FLAC, AAC
- Uses embedded art for locally stored files
- CDN art for streaming catalog
- JPEG preferred, PNG supported
- Size requirement: 500×500 minimum

### 22.5 Tidal

Tidal:
- Primarily uses CDN-hosted cover art
- Embedded art supported for uploaded files
- JPEG and PNG supported
- Preferred size: 1280×1280 minimum

### 22.6 Deezer

Deezer:
- Uses CDN-hosted cover art
- Embedded art may be displayed for local files
- JPEG preferred
- Size requirement: 500×500 minimum, 1000×1000 recommended

### 22.7 Plex Media Server

Plex:
- Displays embedded cover art from all supported formats
- Also scans for sidecar files: folder.jpg, cover.jpg, album.jpg
- Priority: Embedded > folder.jpg > Cover.jpg > album.jpg
- Supports multiple covers per album
- Plexamp (audio-focused client) displays all embedded covers

### 22.8 Roon

Roon:
- Full support for embedded cover art
- Multiple covers with type selection
- Album matching via Gracenote and MusicBrainz
- Embedded art preserved during transcoding
- Display size: up to 2000×2000 for Retina displays

### 22.9 Logitech Media Server (LMS)

Logitech Media Server (Squeezebox ecosystem):
- Supports embedded cover art in MP3, FLAC, OGG, AAC
- Sidecar files: folder.jpg, album.jpg, cover.jpg, front.jpg
- Multiple covers supported (front, back, etc.)
- Cover art cached on server for streaming

### 22.10 Sonos

Sonos:
- Supports embedded cover art
- JPEG preferred
- PNG supported (may be transcoded)
- Sidecar file: folder.jpg
- Size recommendation: 500×500 minimum

---

## 23. COVER ART GENERATION AND SOURCES

### 23.1 Acquiring Cover Art

| Source | Quality | Copyright | Notes |
|--------|---------|---------|-------|
| Label-provided master | Highest | Licensed | Request from label |
| MusicBrainz Cover Art Archive | High | Various | archive.org/coverart |
| Discogs | Medium-High | Various | Check image license |
| Wikipedia Commons | Variable | CC licenses | Check specific license |
| Google Images | Variable | Various | Verify rights |
| Amazon product images | Medium | Varies | Check license |
| Manual photography | Highest | Owned | Scan physical cover |
| Artist-provided | Variable | Licensed | Request from artist |

### 23.2 Cover Art from MusicBrainz

```bash
# Using musicbrainz-utils or coverart-api
musicbrainz-cover-art --release MBID --directory ./covers

# Or using curl directly
RELEASE_ID="12345678-1234-1234-1234-123456789012"
curl -s "https://coverartarchive.org/release/$RELEASE_ID/front-250" -o cover.jpg
```

### 23.3 Cover Art from Discogs

Discogs provides cover art via their API:
```bash
# Get release artwork
curl -s "https://api.discogs.com/releases/{release_id}" \
  -H "Authorization: Discogs token=YOUR_TOKEN" \
  | jq '.images[0].uri'
```

### 23.4 Creating Placeholder Cover Art

For tracks without cover art:
- Generate a simple placeholder with track name and artist
- Use a consistent template for a collection
- Do not leave files without cover art (many players display broken icons)

```python
# Generate placeholder cover art
from PIL import Image, ImageDraw, ImageFont

def create_placeholder(title, artist, size=(1200, 1200), color=(64, 64, 64)):
    img = Image.new('RGB', size, color)
    draw = ImageDraw.Draw(img)

    # Add text
    try:
        font_large = ImageFont.truetype("/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf", 60)
        font_small = ImageFont.truetype("/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf", 40)
    except:
        font_large = ImageFont.load_default()
        font_small = ImageFont.load_default()

    # Title (centered)
    bbox = draw.textbbox((0, 0), title, font=font_large)
    text_width = bbox[2] - bbox[0]
    x = (size[0] - text_width) // 2
    draw.text((x, size[1]//2 - 80), title, fill=(255, 255, 255), font=font_large)

    # Artist
    bbox = draw.textbbox((0, 0), artist, font=font_small)
    text_width = bbox[2] - bbox[0]
    x = (size[0] - text_width) // 2
    draw.text((x, size[1]//2 + 20), artist, fill=(200, 200, 200), font=font_small)

    img.save('placeholder.jpg', 'JPEG', quality=85)
```

---

## 24. COVER ART STANDARDS COMPLIANCE

### 24.1 ID3.org Compliance

ID3v2.4 APIC frame compliance:
- Frame ID must be "APIC" (0x41 0x50 0x49 0x43)
- MIME type string must be null-terminated
- Text encoding byte determines string encoding
- Description string must be null-terminated
- Picture data may be any length
- Multiple APIC frames allowed with unique descriptions

### 24.2 Xiph.org FLAC Compliance

FLAC METADATA_BLOCK_PICTURE compliance:
- All numeric fields are big-endian uint32
- MIME type is ASCII printable (0x20–0x7E)
- Width, height, depth, colors are mandatory
- Picture type follows ID3v2 numbering
- Multiple picture blocks allowed
- No null terminators on any string (length is explicit)

### 24.3 MP4/ISO Base Media Compliance

MP4 covr atom compliance:
- covr atom must be inside moov.udta.meta.ilst
- Each child data atom has type 0x0D (JPEG) or 0x0E (PNG)
- Image data is raw binary (no encoding)
- Multiple covers stored as multiple data atoms

### 24.4 Vorbis Comment Compliance

METADATA_BLOCK_PICTURE compliance:
- Field name is case-insensitive
- Value is base64-encoded FLAC picture block
- Multiple fields allowed (one per picture)
- Base64 must be standard alphabet (A-Z, a-z, 0-9, +, /)
- Padding with = characters

---

## 25. APPENDIX: COMPLETE BINARY REFERENCE

### 25.1 APIC Frame — Byte-by-Byte Analysis

Complete APIC frame from a real MP3 file:

```
Frame Header (10 bytes):
41 50 49 43          Frame ID "APIC"
00 1D 4E 94          Frame size = 1,923,988 bytes (big-endian uint32)
00 00                Flags = 0x0000

Frame Body (variable, 1,923,988 bytes):
[03]                 Text encoding = UTF-8
[69 6D 61 67 65 2F  "image/jpeg"
 6A 70 65 67 00]    MIME type "image/jpeg" + null terminator
[03]                 Picture type = 0x03 (Front cover)
[46 72 6F 6E 74 20  "Front cover"
 63 6F 76 65 72 00] Description "Front cover" + null terminator (UTF-8)
[FF D8 FF E0 00 10  JPEG SOI and APP0 marker
 4A 46 49 46 00 01
 01 00 00 01 00 01
 00 00 ...]         JPEG image data continues...
[... continues for 1,923,900 more bytes ...]
[FF D9]             JPEG EOI marker
```

### 25.2 FLAC PICTURE Block — Complete Structure

```
METADATA_BLOCK_PICTURE (32 + variable bytes):

Bytes 0-3:    Picture type = 0x00000003 (front cover)
Bytes 4-7:    MIME length = 0x0000000A (10 bytes)
Bytes 8-17:   MIME type = "image/jpeg" (10 ASCII bytes, no null)
Bytes 18-21:  Description length = 0x0000000C (12 bytes)
Bytes 22-33:  Description = "Front cover" (12 UTF-8 bytes)
Bytes 34-37:  Width = 0x000004B0 (1200 pixels)
Bytes 38-41:  Height = 0x000004B0 (1200 pixels)
Bytes 42-45:  Color depth = 0x00000018 (24 bits per pixel)
Bytes 46-49:  Colors used = 0x00000000 (0 = not indexed)
Bytes 50-53:  Picture data length = 0x001D4E5C (1,923,956 bytes)
Bytes 54+:    JPEG binary data (1,923,956 bytes)

Total block size: 54 + 1,923,956 = 1,924,010 bytes
Block header (4 bytes): 00 00 1D 4E (little-endian = 1,923,982)
Wait — FLAC metadata blocks use big-endian for type/size...
```

### 25.3 MP4 covr Atom Structure

```
moov.udta.meta.ilst.covr.data (in file):

Box Header (8 bytes):
00 1D 4E 98          Box size = 1,924,008 bytes (big-endian uint32)
63 6F 76 72          "covr" atom type

data[0] Box (8 bytes header + image):
00 1D 4E 90          Box size = 1,923,952 bytes
64 61 74 61          "data" atom type
00 00 00 00          Reserved (4 bytes)
00 0D                 Data type = JPEG (0x0D = mdta/jpeg)
00 00                Reserved (2 bytes)
[FF D8 FF E0 ...]    JPEG SOI and image data
[... JPEG continues ...]
[FF D9]              JPEG EOI marker
```

---

## 26. COVER ART IMPLEMENTATION CHECKLIST

### 26.1 For Audio Converters

- [ ] Extract cover art from source file (APIC, METADATA_BLOCK_PICTURE, covr)
- [ ] Convert image format if needed (PNG → JPEG for MP3)
- [ ] Resize image if exceeding maximum dimensions
- [ ] Compress image if exceeding maximum file size
- [ ] Preserve picture type (0x03 = front cover)
- [ ] Set correct MIME type in destination format
- [ ] Set correct text encoding (UTF-8 recommended)
- [ ] Embed cover art with appropriate disposition
- [ ] Verify embedded art matches source (compare checksums)
- [ ] Test output file in multiple players

### 26.2 For Tag Editors

- [ ] Display all embedded pictures with their types
- [ ] Allow adding new pictures with type selection
- [ ] Allow removing individual pictures
- [ ] Support reordering multiple pictures
- [ ] Validate image format before embedding
- [ ] Warn about oversized images
- [ ] Support batch embedding from folder.jpg
- [ ] Extract cover art to external file
- [ ] Preserve all existing tags during cover art changes

### 26.3 For Media Servers

- [ ] Scan and index all embedded cover art
- [ ] Support sidecar files (folder.jpg, etc.)
- [ ] Generate thumbnails at multiple sizes
- [ ] Cache cover art for fast retrieval
- [ ] Handle files without cover art gracefully
- [ ] Support multiple covers per album
- [ ] Display cover art in player UI
- [ ] Enable cover art download/export

### 26.4 For Streaming Uploads

- [ ] Resize to service's recommended dimensions
- [ ] Compress to service's recommended file size
- [ ] Convert to JPEG if needed
- [ ] Verify square aspect ratio
- [ ] Upload to CDN via API
- [ ] Embed in source file as backup
- [ ] Test display in target service's app
- [ ] Handle multiple release variants

---

*File generated for: Audio Engineering Knowledge Base*
*Topic: Embedded Cover Art, APIC, METADATA_BLOCK_PICTURE, covr Atom*
*[NEEDS VERIFICATION] markers indicate specific numerical values that require additional source confirmation*
*[NEEDS VERIFICATION]: The WMA/ASF WM/Picture binary format — the structure described is based on typical ASF_PICTURE object layouts but should be verified against the ASF specification document*
*[NEEDS VERIFICATION]: The exact max sizes for streaming services in Section 15.1 — these change frequently and should be verified against current platform documentation*
*[NEEDS VERIFICATION]: Specific size limits for iTunes (6000×6000) and other platform-specific maximum dimensions*
