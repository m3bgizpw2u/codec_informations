# APE Tags — Deep Technical Reference
> **Category:** Metadata
> **File Extensions:** Supported in: `.ape` (Monkey's Audio), `.mp3` (via APEv2 footer), `.flac` (rare), `.wv` (WavPack), `.ofr` (OptimFROG)
> **MIME Types:** `audio/x-ape`, `audio/ape`
> **Standardization Body:** None (de facto standard from Monkey's Audio)
> **Primary Specification:** APEv2 specification (Hydrogenaudio Knowledgebase)
> **Patent Status:** Patent-free
> **License:** Open (used by multiple formats and tools)
> **Current Version:** APEv2 (Version 2000), APEv1 (Version 1000)
> **Active Development:** Yes — maintained by Monkey's Audio and various tag editors

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Matthew T. Ashland (Monkey's Audio)
- **Year Created:** 2000 (APEv1), 2001 (APEv2)
- **Original Purpose:** Tag metadata for the Monkey's Audio lossless audio codec, designed to replace ID3v1/v2 for a format that natively had no metadata
- **Problem with Predecessors:** MP3 files lacked a native metadata system (addressed by ID3). When embedding tags in other lossless formats like FLAC or WAV, developers needed a clean, binary-efficient alternative to ID3

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| APEv1 | 2000 | Initial tag format — footer only, 30-char text fields, at end of file |
| APEv2 | 2001 | Added header, text encoding (UTF-8), binary items, external references, case-insensitive keys |
| APEv2 2000 | Present | Current standard — version field = 2000 |

### 1.3 Current Adoption
- **Primary use cases today:** Monkey's Audio files, WavPack files, OptimFROG files, some FLAC files, legacy MP3 files
- **Platforms with native support:** All major tag editors (Mp3tag, foobar2000, kid3, EasyTAG)
- **Major services using this format:** Niche — primarily used by lossless audio enthusiasts
- **Hardware support:** Limited — most DAPs don't read APE tags natively, rely on embedded ID3
- **Status:** Niche but stable — widely supported in software, declining in new format adoption

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 APE Tag Block Architecture
An APE tag block consists of three components:
```
┌────────────────────┐
│  APE Tag Header     │  32 bytes — optional (v2 only, for tag at beginning of file)
│  (APETAGEX + flags)│
├────────────────────┤
│  Tag Item 1        │  10+ bytes — key-value pairs, sorted by key name
│  Tag Item 2        │
│  ...               │
│  Tag Item N        │
├────────────────────┤
│  APE Tag Footer    │  32 bytes — always present for end-of-file tags
│  (APETAGEX + flags)│
└────────────────────┘
```

**Key rules:**
- Tag at end of file (recommended): MUST have footer, MAY have header
- Tag at beginning of file (strongly discouraged): MUST have header, MAY have footer
- Items MUST be sorted by key name (lexicographic order recommended)
- Each key can only appear once (no duplicate keys)
- Header and footer differ only in bit 29 of the flags field

### 2.2 File Placement
APE tags can be located at:
1. **End of file** (strongly recommended): Placed after all audio data, before ID3v1 (if present)
2. **Beginning of file** (discouraged): Placed before audio data
3. **Embedded in container** (format-dependent): Some formats embed APE within their own metadata structure

For MP3 files with both APE and ID3v1:
```
[MP3 Frames...] [APEv2 Tag Block] [ID3v1 Tag]
                        ↑
                  APE footer here, header optional
```

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 APE Tag Footer Structure (32 bytes)
The footer is identical in structure to the header. Both are 32 bytes.

```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------------
0x0000   8B      Preamble               char[8]     MUST be "APETAGEX" (0x4150455441474558)
0x0008   4B      Version               uint32 LE   1000 = v1.0, 2000 = v2.0
0x000C   4B      Tag Size              uint32 LE   Size of tag excluding header (bytes)
0x0010   4B      Item Count            uint32 LE   Number of items in this tag
0x0014   4B      Flags                 uint32 LE   Tag flags (bitfield)
0x0018   8B      Reserved              uint8[8]    MUST be zero (0x0000000000000000)
```

**APE Tag Footer — Hex Dump Example:**
```
41 50 45 54 41 47 45 58   ; "APETAGEX" preamble
D0 07 00 00               ; Version = 2000 (0x07D0 in little-endian)
2F 01 00 00               ; Tag size = 303 bytes (items + footer)
03 00 00 00               ; Item count = 3
00 00 00 00               ; Flags = 0x00000000 (has footer, no header)
00 00 00 00 00 00 00 00  ; Reserved = 0
```

### 3.2 APE Tag Header Structure (32 bytes, v2 only)
The header has the same structure as the footer, with two differences:
- Bit 31 (Contains Header) is set to 1
- Bit 29 (Is Header) is set to 1

```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------------
0x0000   8B      Preamble               char[8]     MUST be "APETAGEX"
0x0008   4B      Version               uint32 LE   MUST be 2000 (v2.0)
0x000C   4B      Tag Size              uint32 LE   Size of items + footer (excl. header)
0x0010   4B      Item Count            uint32 LE   Number of items
0x0014   4B      Flags                 uint32 LE   Flags with bit 31=1, bit 29=1
0x0018   8B      Reserved              uint8[8]    MUST be zero
```

**APE Tag Header — Hex Dump Example:**
```
41 50 45 54 41 47 45 58   ; "APETAGEX" preamble
D0 07 00 00               ; Version = 2000
2F 01 00 00               ; Tag size = 303
03 00 00 00               ; Item count = 3
A0 00 00 00               ; Flags = 0x000000A0 (has header | is header)
00 00 00 00 00 00 00 00  ; Reserved
```

### 3.3 Flags Field — Bit Layout
```
Bit 31: Contains Header      (1 = tag has a header, 0 = no header)
Bit 30: Contains Footer      (1 = tag has a footer, 0 = no footer)
Bit 29: Is Header            (1 = this is a header, 0 = this is a footer)
Bits 28–3: Reserved          (MUST be zero)
Bit 2–0: Item flags encoding (for header/footer: reserved, must be zero)

For TAG ITEMS (separate bit layout):
Bit 31: Read-only item       (1 = read-only, 0 = editable)
Bits 30–3: Reserved         (MUST be zero)
Bit 2–1: Value Type
           00 = UTF-8 text item
           01 = Binary item
           10 = External reference
           11 = Reserved
Bit 0: Reserved              (MUST be zero)
```

### 3.4 APE Tag Item Structure (minimum 10 bytes)
Each tag item has the following binary layout:
```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------------
0x0000   4B      Value Size             uint32 LE   Length of value in bytes
0x0004   4B      Item Flags            uint32 LE   Item flags (value type, read-only)
0x0008   N       Key Name              char[]      ASCII 0x20–0x7E, 1–255 bytes
                                          + 1 null terminator (0x00)
0xNNNN   M       Value Data            uint8[]     M = Value Size bytes
```

**Minimum item size:** 4 + 4 + 2 + 1 = 11 bytes (size=0, flags=0, key="A" + null, empty value)

**Item Flags — Bit Layout:**
```
Bit 31: Read-only            (1 = read-only, 0 = read-write)
Bits 30–3: Reserved          (MUST be zero)
Bit 2–1: Value Type
           00 (0) = UTF-8 text
           01 (1) = Binary data
           10 (2) = External reference
           11 (3) = Reserved
Bit 0: Reserved              (MUST be zero)
```

### 3.5 APE Tag Item — Complete Hex Dump Example
```
; Item: "ARTIST" = "The Artist Name"

04 00 00 00               ; Value size = 4 bytes ("true" — let's use FALSE)
00 00 00 00               ; Flags = 0x00000000 (UTF-8 text, read-write)
41 52 54 49 53 54 00      ; Key = "ARTIST\0" (7 bytes)
54 68 65 20 41 72 74 69   ; Value = "The Ar"
73 74 20 4E 61 6D 65      ; Value = "ti Name"
```

### 3.6 External Reference Item
External references (value type = 2) store a filename/path to external storage:
```
Value Data: UTF-8 encoded path to external file
```
Use cases: Embedded cover art stored as separate file, multilingual content split across files.

### 3.7 Key Name Restrictions
- **Characters:** ASCII 0x20 (space) through 0x7E (tilde)
- **No control characters, no high-bit characters**
- **Case sensitive** when comparing
- **Illegal to have two keys differing only in case** (e.g., "Artist" and "ARTIST")
- **Maximum length:** 255 bytes (excluding null terminator)
- **Recommended to be ASCII alphanumeric with underscores**
- **Common key names are uppercase:** TITLE, ARTIST, ALBUM, YEAR, GENRE, TRACK, COMMENT

---

## 4. STANDARD TAG FIELDS

### 4.1 Standard APE Tag Keys
| Tag Field | APE Key | Max Length | Type | Multi-value | Notes |
|-----------|---------|-----------|------|-------------|-------|
| Title | TITLE | Unlimited | Text | No | |
| Artist | ARTIST | Unlimited | Text | No | |
| Album | ALBUM | Unlimited | Text | No | |
| Year | YEAR | 4 bytes | Text | No | e.g., "2024" |
| Genre | GENRE | Unlimited | Text | No | Freeform text |
| Track Number | TRACK | 3 bytes | Text | No | e.g., "5" or "5/12" |
| Comment | COMMENT | Unlimited | Text | No | |
| Disk/Disc | DISCNUMBER | 3 bytes | Text | No | e.g., "1/2" |
| Album Artist | ALBUM ARTIST | Unlimited | Text | No | |
| Composer | COMPOSER | Unlimited | Text | No | |
| Encoder | ENCODER | Unlimited | Text | No | Software name |
| Encoder Settings | ENCODER SETTINGS | Unlimited | Text | No | Quality settings |
| ISRC | ISRC | 12 bytes | Text | No | |
| BPM | BPM | 5 bytes | Text | No | e.g., "120" |
| Catalog Number | CATALOG | Unlimited | Text | No | |
| Barcode | BARCODE | Unlimited | Text | No | |
| Publisher | PUBLISHER | Unlimited | Text | No | |
| Copyright | COPYRIGHT | Unlimited | Text | No | |
| ReplayGain Track Gain | REPLAYGAIN_TRACK_GAIN | 12 bytes | Text | No | Format: "-6.20 dB" |
| ReplayGain Track Peak | REPLAYGAIN_TRACK_PEAK | 10 bytes | Text | No | Format: "0.998459" |
| ReplayGain Album Gain | REPLAYGAIN_ALBUM_GAIN | 12 bytes | Text | No | |
| ReplayGain Album Peak | REPLAYGAIN_ALBUM_PEAK | 10 bytes | Text | No | |
| MusicBrainz Track ID | MUSICbrainz_trackid | 36 bytes | Text | No | UUID |
| MusicBrainz Album ID | MUSICbrainz_albumid | 36 bytes | Text | No | |
| MusicBrainz Artist ID | MUSICbrainz_artistid | 36 bytes | Text | No | |
| MusicBrainz Release Group ID | MUSICbrainz_releasegroupid | 36 bytes | Text | No | |
| MusicBrainz Track MBID | MUSICBRAINZ_TRACKID | 36 bytes | Text | No | |
| MusicBrainz Album MBID | MUSICBRAINZ_ALBUMID | 36 bytes | Text | No | |
| MusicBrainz Artist MBID | MUSICBRAINZ_ARTISTID | 36 bytes | Text | No | |
| MusicBrainz Album Artist MBID | MUSICBRAINZ_ALBUMARTISTID | 36 bytes | Text | No | |
| MusicBrainz TRM | MUSICBRAINZ_TRM | 36 bytes | Text | No | (deprecated) |
| Cover Art (Front) | COVER ART (FRONT) | Unlimited | Binary | No | JPEG/PNG |
| Cover Art (Back) | COVER ART (BACK) | Unlimited | Binary | No | JPEG/PNG |
| Cover Art | COVER ART | Unlimited | Binary | No | Generic cover |
| Lyrics | LYRICS | Unlimited | Text | No | No 255-byte limit |
| Comment | COMMENT | Unlimited | Text | No | Multi-value supported |

### 4.2 Cover Art in APE Tags
Cover art is stored as a binary item with key "COVER ART", "COVER ART (FRONT)", or "COVER ART (BACK)".

**Binary item encoding:**
```
Flags = 0x00000001  (binary data type)

Value: Raw image bytes (JPEG or PNG)
```

**Rules:**
- Item flags: value type = binary (bit 1 = 1)
- Value contains raw image bytes — no additional wrapper
- Multiple cover art items allowed (use different keys)
- Maximum image size: implementation-dependent (recommend < 10MB)

### 4.3 Multi-Value Items
APE tags support multiple items with the same key (unlike ID3v2 which uses `;` separator):
```
Item 1: KEY = "Artist"  VALUE = "Lead Vocal"
Item 2: KEY = "Artist"  VALUE = "Background Vocal"
Item 3: KEY = "Artist"  VALUE = "Guitar Solo"
```
Reading tools should concatenate or present all values.

### 4.4 Value Encoding
| Value Type | Encoding | Bit 2-1 | Notes |
|-----------|----------|---------|-------|
| UTF-8 text | UTF-8 | 00 | Default for all text fields |
| Binary data | Raw bytes | 01 | Cover art, binary metadata |
| External reference | UTF-8 path | 10 | Path to external file |
| Reserved | — | 11 | Do not use |

---

## 5. APEv1 COMPATIBILITY

### 5.1 APEv1 vs APEv2 Differences
| Feature | APEv1 | APEv2 |
|---------|-------|-------|
| Header | No | Yes (optional for end-of-file tags) |
| Footer | Yes (32 bytes) | Yes (32 bytes) |
| Version field | 1000 | 2000 |
| Text encoding | Latin-1 (ISO-8859-1) | UTF-8 |
| Binary items | No | Yes |
| External references | No | Yes |
| Key length | 30 bytes | 255 bytes |
| Value length | 32,767 bytes | Unlimited |
| Location | End of file only | Beginning or end of file |
| ID3v1 compatibility | Places before ID3v1 | Places before ID3v1 |

### 5.2 APEv1 Tag Layout
APEv1 has no header — it starts directly with tag items:
```
[Tag Item 1]
[Tag Item 2]
...
[Tag Item N]
[APE Tag Footer (32 bytes)]
```

**APEv1 Footer Hex Dump:**
```
41 50 45 54 41 47 45 58   ; "APETAGEX"
E8 03 00 00               ; Version = 1000
[tag size]
[item count]
[flags]
[reserved]
```

### 5.3 File Positions
For files with both APEv2 and ID3v1:
```
Standard MP3 with APEv2 + ID3v1:
[ID3v1] ← at very end of file
[APEv2 Footer]
[APEv2 Items]
[APEv2 Header] ← optional

For APEv1 + ID3v1:
[ID3v1]
[APEv1 Footer]
[APEv1 Items]
(no header)
```

---

## 6. TAG READING ALGORITHM

### 6.1 Finding APE Tags in a File

**Step 1: Check for APEv2 tag at beginning of file**
```
Read first 32 bytes:
  IF bytes[0:8] == "APETAGEX" AND version == 2000 AND flags.bit29 == 1:
    → Found APEv2 header at beginning
    → Read items until tag size bytes consumed
    → Continue to audio data
```

**Step 2: Check for APE tag at end of file**
```
Seek to file_end - 32:
  IF bytes == "APETAGEX":
    → Found APE footer
    → Read tag size and item count
    → Seek back tag_size bytes from footer
    → Read items
```

**Step 3: Check for ID3v1 tag**
```
Seek to file_end - 128:
  IF bytes[0:3] == "TAG":
    → Found ID3v1 tag
    → Parse 128-byte structure
```

### 6.2 Complete Tag Reading Algorithm
```python
def read_ape_tags(filename):
    with open(filename, 'rb') as f:
        # Check for APEv2 at beginning
        f.seek(0)
        header = f.read(32)
        if header[:8] == b'APETAGEX' and int.from_bytes(header[8:12], 'little') >= 2000:
            if header[20] & 0x20:  # bit 5 (contains header)
                f.seek(0)
                header = parse_ape_header(f)
                items = read_ape_items(f, header['tag_size'])
                # Continue to find footer at end...
        
        # Check for APE at end
        f.seek(-32, 2)  # 2 = from end
        footer = f.read(32)
        if footer[:8] == b'APETAGEX':
            tag_size = int.from_bytes(footer[12:16], 'little')
            item_count = int.from_bytes(footer[16:20], 'little')
            f.seek(-32 - tag_size, 2)
            items = read_ape_items(f, tag_size)
        
        return items

def read_ape_items(f, tag_size):
    end_pos = f.tell() + tag_size
    items = {}
    while f.tell() < end_pos:
        value_size = int.from_bytes(f.read(4), 'little')
        flags = int.from_bytes(f.read(4), 'little')
        key = b''
        while True:
            b = f.read(1)
            if b == b'\x00' or b == b'':
                break
            key += b
        key = key.decode('ascii')
        value = f.read(value_size)
        
        value_type = (flags >> 1) & 0x03
        if value_type == 0:  # UTF-8 text
            value = value.decode('utf-8')
        # binary, external: keep as bytes
        
        if key not in items:
            items[key] = []
        items[key].append(value)
    
    return items
```

### 6.3 Tag Conflict Resolution
When both APEv1 and APEv2 exist in the same file:
- **Read priority:** APEv2 takes precedence over APEv1
- **Write behavior:** When modifying tags, update APEv2 (v2 is newer/more capable)
- **Delete behavior:** Removing APEv2 should not affect APEv1 (independent systems)

When both APE and ID3v1 exist:
- **Read priority:** APEv2 > APEv1 > ID3v1
- **Write behavior:** Most tools prefer writing APE over ID3v1 for new tags

---

## 7. TAG WRITING & STRIPPING

### 7.1 Safe Tag Writing Algorithm
```python
def write_ape_tags(filename, tags, version=2000):
    """Write APEv2 tags to end of file."""
    
    # Build item data
    item_data = b''
    for key, values in tags.items():
        if not isinstance(values, list):
            values = [values]
        for value in values:
            if isinstance(value, str):
                value_bytes = value.encode('utf-8')
                flags = 0x00000000  # UTF-8 text
            else:
                value_bytes = value  # bytes for binary
                flags = 0x00000001  # binary
            
            item_data += int.to_bytes(len(value_bytes), 4, 'little')
            item_data += int.to_bytes(flags, 4, 'little')
            item_data += key.encode('ascii') + b'\x00'
            item_data += value_bytes
    
    # Sort items by key name
    # (Parser requirement: items should be sorted)
    
    # Build footer
    footer = b'APETAGEX'
    footer += int.to_bytes(version, 4, 'little')  # 2000
    footer += int.to_bytes(len(item_data), 4, 'little')  # tag size
    footer += int.to_bytes(len(tags), 4, 'little')  # item count
    footer += int.to_bytes(0x00000000, 4, 'little')  # flags
    footer += b'\x00' * 8  # reserved
    
    # Write to file
    with open(filename, 'r+b') as f:
        f.seek(0, 2)  # end of file
        f.write(item_data)
        f.write(footer)
```

### 7.2 Safe Tag Stripping
To safely remove APE tags:

```python
def strip_ape_tags(filename):
    """Remove all APE tags from file."""
    
    # 1. Read entire file
    with open(filename, 'rb') as f:
        data = f.read()
    
    audio_start = 0
    audio_end = len(data)
    
    # 2. Check for APE at beginning
    if data[:32] == b'APETAGEX':
        # Parse header to find tag size
        tag_size = int.from_bytes(data[12:16], 'little')
        audio_start = 32 + tag_size
    
    # 3. Check for APE at end
    if data[-32:-24] == b'APETAGEX':
        tag_size = int.from_bytes(data[-24:-20], 'little')
        audio_end = len(data) - 32 - tag_size
    
    # 4. Also check for ID3v1
    if audio_end >= 128 and data[audio_end-128:audio_end-128+3] == b'TAG':
        audio_end -= 128
    
    # 5. Write audio data only
    with open(filename, 'wb') as f:
        f.write(data[audio_start:audio_end])
```

### 7.3 Tag Stripping with FFmpeg
```bash
# Remove all APE tags (and other tags)
ffmpeg -i input.ape -c:a copy -map_metadata -1 output.ape

# Remove APE tags only (use taglib/python-eyeD3 if specific removal needed)
```

---

## 8. FFMPEG APE TAG REFERENCE

### 8.1 FFmpeg APE Tag Reading
```bash
# Read all metadata (including APE tags)
ffprobe -v quiet -print_format json -show_format input.ape | jq .format.tags

# Read specific APE tag fields
ffprobe -v quiet -show_entries format_tags=title,artist,album -of default=noprint_wrappers=1 input.ape

# Dump all tags
ffprobe -v error -show_entries format_tags -of ini=input=file input.ape
```

### 8.2 FFmpeg APE Tag Writing
```bash
# Write APE tags to APE file
ffmpeg -i input.ape -c:a copy \
  -metadata title="Song Title" \
  -metadata artist="Artist Name" \
  -metadata album="Album Name" \
  -metadata date="2024" \
  -metadata genre="Electronic" \
  -metadata track="5" \
  -metadata REPLAYGAIN_TRACK_GAIN="-6.20 dB" \
  -metadata REPLAYGAIN_TRACK_PEAK="0.998459" \
  output.ape

# Strip all metadata
ffmpeg -i input.ape -c:a copy -map_metadata -1 output.ape
```

### 8.3 FFmpeg Internal Metadata Key Mapping
| Standard Field | FFmpeg Key | APE Tag Key | Notes |
|----------------|------------|-------------|-------|
| Title | title | TITLE | |
| Artist | artist | ARTIST | |
| Album | album | ALBUM | |
| Album Artist | album_artist | ALBUMARTIST | |
| Track Number | track | TRACK | Format: "N" or "N/T" |
| Disc Number | disc | DISCNUMBER | Format: "N" or "N/T" |
| Date/Year | date | YEAR | |
| Genre | genre | GENRE | |
| Comment | comment | COMMENT | |
| Composer | composer | COMPOSER | |
| Copyright | copyright | COPYRIGHT | |
| Encoder | encoder | ENCODER | |
| BPM | tempo | BPM | |
| ReplayGain Track Gain | REPLAYGAIN_TRACK_GAIN | REPLAYGAIN_TRACK_GAIN | |
| ReplayGain Track Peak | REPLAYGAIN_TRACK_PEAK | REPLAYGAIN_TRACK_PEAK | |
| MusicBrainz Track ID | musicbrainz_trackid | MUSICBRAINZ_TRACKID | |

### 8.4 Cover Art Handling in FFmpeg with APE
```bash
# Extract cover art from APE file
ffmpeg -i input.ape -an -vcodec copy cover.jpg

# Embed cover art into APE file
ffmpeg -i input.ape -i cover.jpg \
  -c:a copy \
  -c:v copy \
  -map 0:a -map 1:v \
  -metadata:s:v title="Cover" \
  output.ape
```

---

## 9. KNOWN TOOLS & IMPLEMENTATIONS

### 9.1 Tag Editors
| Tool | Platform | APE Support | Notes |
|------|----------|-------------|-------|
| Mp3tag | Windows | Full | Best APE support; can edit all fields |
| foobar2000 | Windows | Full | Native; has APE Tag Plus plugin |
| EasyTAG | Linux | Full | GTK-based |
| kid3 | Cross-platform | Full | Qt-based |
| APE Tag Plus | foobar2000 | Extended | Supports custom fields |
| puddletag | Linux | Full | Python-based |
| JAudioTagger | Java | Full | Library used by Bliss |

### 9.2 Command-Line Tools
| Tool | Purpose | Key Commands |
|------|---------|--------------|
| apetag | Read/write APE tags | `apetag -s file.ape` |
| mp3diags | APE + MP3 diagnostics | `mp3diags file.mp3` |
| metaflac | FLAC (supports APE) | `metaflac --list file.flac` |
| Vorbiscomment | OGG/FLAC | Similar interface |

### 9.3 Programming Libraries
| Library | Language | APE Support |
|---------|----------|-------------|
| libapetag | C | Read/write |
| TagLib | C++ | Read/write (via APE tag class) |
| mutagen | Python | Read/write |
| eyeD3 | Python | Read/write (primarily ID3, some APE) |
| jaudiotagger | Java | Read/write |

---

## 10. METADATA COMPATIBILITY

### 10.1 APE Tag Compatibility Matrix
| Format | Read | Write | Notes |
|--------|------|-------|-------|
| Monkey's Audio (.ape) | Native | Native | Built-in |
| WavPack (.wv) | Native | Native | Built-in |
| OptimFROG (.ofr) | Native | Native | Built-in |
| FLAC (.flac) | Via vorbiscomment | Via vorbiscomment | Rarely used |
| MP3 (.mp3) | Via APE footer | Via APE footer | Before ID3v1 |
| AIFF (.aiff) | Rare | Rare | Non-standard |
| WAV (.wav) | Rare | Rare | Non-standard |

### 10.2 APE vs Other Tag Systems
| Feature | APEv2 | ID3v2.4 | Vorbis Comment | iTunes Atoms |
|---------|--------|---------|----------------|--------------|
| Location | Start or end | Start | End | Within container |
| Encoding | UTF-8 | UTF-8, UTF-16, Latin-1 | UTF-8 | UTF-8 |
| Binary data | Yes | Yes (APIC) | Yes (METADATA_BLOCK_PICTURE) | Yes (covr) |
| External refs | Yes | No | No | No |
| Key length | 255 bytes | 64 bytes | 256 bytes | 4 chars + free-form |
| Multi-value | Yes | Via separator | Yes | Yes |
| Standardized | No | Yes (ID3.org) | Yes (Xiph.org) | Yes (Apple) |

---

## 11. EDGE CASES & KNOWN ISSUES

### 11.1 Common Pitfalls
| Issue | Cause | Solution |
|-------|-------|----------|
| Case sensitivity | Keys differ only in case | Normalize to uppercase |
| Duplicate keys | Multiple items with same key | Last value wins (or concatenate) |
| Missing footer | Corrupt/incomplete tag | Reconstruct footer from header |
| Zero item count | Empty tag block | Treat as no tag |
| Invalid UTF-8 | Binary marked as text | Check flags, decode accordingly |
| Oversized values | Value > value_size | Validate size, truncate if needed |

### 11.2 Interoperability Issues
- **foobar2000:** May add custom fields (like `replaygain_track_gain`) that other tools don't understand
- **Mp3tag:** May normalize keys to uppercase, which is technically allowed
- **Kid3:** Preserves case exactly as written
- **FLAC + APE:** Not standard; some players may ignore APE in FLAC

### 11.3 Binary Item Handling
Binary items (value type = 1) require special handling:
```python
# Reading binary item
if (flags & 0x00000006) == 0x00000002:  # binary type
    value = raw_bytes  # Keep as bytes
    # For cover art, detect MIME type from magic bytes:
    if value[:3] == b'\xFF\xD8\xFF':  # JPEG
        mime = 'image/jpeg'
    elif value[:8] == b'\x89PNG\r\n\x1A\n':  # PNG
        mime = 'image/png'
```

---

## 12. CONVERSION GUIDE

### 12.1 APE Tags to Other Formats
| Target | FFmpeg Command | Metadata Preserved |
|--------|---------------|-------------------|
| FLAC | `ffmpeg -i in.ape -c:a flac -compression_level 8 out.flac` | Via -map_metadata |
| MP3 | `ffmpeg -i in.ape -c:a libmp3lame -q:a 0 out.mp3` | ID3v2.3 tags |
| AAC | `ffmpeg -i in.ape -c:a aac -b:a 256k out.m4a` | Via -movflags |
| Opus | `ffmpeg -i in.ape -c:a libopus -b:a 128k out.opus` | Vorbis Comments |

### 12.2 Other Formats to APE Tags
| Source | Method | Notes |
|--------|--------|-------|
| FLAC (Vorbis) | `ffmpeg -i in.flac -c:aape -compression_level 8 out.ape` | Extract + write |
| MP3 (ID3v2) | TagLib/ffmpeg | Convert field names |

---

## 13. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | URL |
|---------|----------|---------|-----|
| TagLib | C++ | LGPL | github.com/taglib/taglib |
| Mutagen | Python | GPL | github.com/quodlibet/mutagen |
| libapetag | C | GPL | (embedded/simple implementation) |
| FFmpeg | C | LGPL 2.1+ | ffmpeg.org |

### 13.1 TagLib APE Tag Example
```cpp
#include <taglib/taglib.h>
#include <taglib/apetag.h>
#include <taglib/fileref.h>

using namespace TagLib;

FileRef f("music.ape");
if (!f.isNull() && f.tag()) {
    APE::Tag *tag = f.tag();
    
    // Read
    String title = tag->itemListMap()["TITLE"].value();
    
    // Write
    tag->setTitle("New Title");
    tag->itemListMap()["ARTIST"] = APE::Item("ARTIST", "Artist Name");
    
    // Binary item (cover art)
    tag->itemListMap()["COVER ART"] = APE::Item("COVER ART", 
        ByteVector(cover_data, cover_size));
    
    f.save();
}
```

### 13.2 Mutagen APE Tag Example (Python)
```python
from mutagen.apev2 import APEv2

# Read
tags = APEv2("music.ape")
title = tags["TITLE"]
artist = tags["ARTIST"]

# Write
tags["TITLE"] = "New Title"
tags["ARTIST"] = "Artist Name"
tags["REPLAYGAIN_TRACK_GAIN"] = "-6.20 dB"
tags["REPLAYGAIN_TRACK_PEAK"] = "0.998459"

# Binary item
with open("cover.jpg", "rb") as f:
    tags["COVER ART (FRONT)"] = f.read()

tags.save()

# Strip all tags
APEv2.delete("music.ape")
```

---

## 14. SPECIFICATIONS & FURTHER READING

### Primary Specification
- **APEv2 Specification:** https://wiki.hydrogenaudio.org/index.php?title=APEv2_specification
- **APE Tag Item:** https://wiki.hydrogenaudio.org/index.php?title=APE_Tag_Item
- **APEv1 Specification:** https://wiki.hydrogenaudio.org/index.php?title=APEv1_specification
- **Mutagen APEv2 Spec:** http://mutagen-specs.readthedocs.io/en/latest/apev2/apev2.html
- **TagLib APE Format:** https://github.com/taglib/taglib/blob/master/taglib/ape/ape-tag-format.txt

### Technical Resources
- Hydrogenaudio Forums: https://hydrogenaudio.org
- Mutagen documentation: https://mutagen.readthedocs.io
- FFmpeg metadata handling: https://ffmpeg.org/ffmpeg-all.html

---

## 15. IMPLEMENTATION CHECKLIST

### Reading Pipeline
- [ ] Scan for APE header at byte 0 (version 2000, has-header flag)
- [ ] Scan for APE footer at file_end - 32 bytes
- [ ] Parse tag size and item count from footer
- [ ] Seek back and read items
- [ ] Handle both binary and text value types (check flags)
- [ ] Sort items by key (parser requirement)
- [ ] Check for ID3v1 at very end of file
- [ ] Resolve conflicts: APEv2 > APEv1 > ID3v1

### Writing Pipeline
- [ ] Validate key names (ASCII 0x20-0x7E, no duplicates differing in case)
- [ ] Validate value sizes (fit in uint32)
- [ ] Encode text values as UTF-8
- [ ] Encode binary values as raw bytes
- [ ] Sort items by key name
- [ ] Calculate tag size (items + footer, excluding header)
- [ ] Write items, then footer
- [ ] Preserve existing audio data
- [ ] Handle read-only flags if present

### Metadata Fields
- [ ] Read/write standard keys: TITLE, ARTIST, ALBUM, YEAR, GENRE, TRACK, COMMENT
- [ ] Read/write extended keys: REPLAYGAIN_*, MUSICBRAINZ_*, COVER ART*
- [ ] Handle multi-value items (list of values per key)
- [ ] Preserve binary items (cover art) as raw bytes
- [ ] Handle UTF-8 encoding for all text fields

### Edge Cases
- [ ] Handle empty tags (0 items)
- [ ] Handle corrupted tags (invalid size, missing footer)
- [ ] Handle truncated files
- [ ] Handle files with both APEv1 and APEv2
- [ ] Handle files with APE + ID3v1
- [ ] Validate UTF-8 encoding for text items
- [ ] Handle binary items with invalid image data

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
