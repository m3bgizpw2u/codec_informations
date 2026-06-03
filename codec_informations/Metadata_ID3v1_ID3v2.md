# Metadata: ID3v1 and ID3v2 — Deep Technical Reference

> **Category:** Metadata / Tagging System
> **File Extensions:** .mp3, .mp2, .mp1 (primarily)
> **MIME Types:** N/A (metadata system)
> **Standardization Body:** ID3.org (informal standard)
> **Specification Document:** http://id3.org/id3v2.4.0-structure, id3v2.3.0, id3v2.4.0-frames
> **Patent Status:** Patent-free
> **License:** Public domain

---

## 1. HISTORICAL CONTEXT & ORIGIN

The ID3 tagging system was invented in 1996 by Eric Kemp (沿岸), a Swedish developer who created the initial ID3v1 specification to embed metadata into MP3 files. The original ID3v1 tag was placed at the end of an MP3 file, occupying the last 128 bytes, and was designed as a simple, portable solution to the problem of storing song title, artist, album, and other descriptive information alongside compressed audio data. The specification was never standardized by any formal body, existing instead as a de facto standard published at id3.org, maintained by a loose community of volunteers.

ID3v1 quickly became ubiquitous despite its severe limitations: only 30 bytes for title and artist fields, a single byte for genre, and no support for Unicode, embedded images, or custom fields. Recognizing these shortcomings, the ID3v2 standard was developed starting in 1998, with the first major version (ID3v2.2) introducing a frame-based extensible architecture placed at the beginning of files. ID3v2.3 arrived in 2000 with substantially expanded frame types and a more robust header structure. ID3v2.4 followed in 2001, refining the spec with synchsafe integers, multiple text encoding support, and a more consistent frame structure.

The most significant design decision in ID3v2 was placing the tag at the **beginning** of the file rather than the end. This was intentional: older streaming servers and CD-copying software that prepend data to MP3 files would destroy ID3v1 tags placed at the end. By placing metadata at the start, ID3v2 survived most file manipulation operations. The tag is also designed to be skippable: it does not contain MPEG audio sync bytes (`0xFF`), so any player can skip the tag to reach the audio data.

The informal nature of ID3 meant the specification evolved through community consensus rather than formal standardization. This has produced some quirks: there is no formal governing body, versioning policies differ between versions, and some behaviors (like the exact handling of padding or unsynchronisation) vary between implementations. Nevertheless, ID3v2.3 remains the most widely supported version across all MP3 players and software, while ID3v2.4 adoption has grown with modern tools.

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

The ID3 metadata system consists of two completely independent tagging schemes that can coexist in the same file:

**ID3v1 (and v1.1):** A fixed 128-byte footer appended to the end of an MP3 file. It uses a flat, non-extensible record structure with hard-coded field offsets. It is always present in 128 bytes, padded with null bytes or spaces, and identifiable by the ASCII string "TAG" at its start. ID3v1.1 added track number support by repurposing two bytes of the comment field.

**ID3v2 (v2.2, v2.3, v2.4):** A variable-length header at the beginning of an MP3 file, using an extensible frame-based architecture. Each frame has a unique identifier, a size field, optional flags, and arbitrary payload data. New frame types can be added without breaking compatibility. ID3v2 supports multiple text encodings (ISO-8859-1, UTF-16, UTF-16BE, UTF-8), embedded images, custom fields, and binary data.

The coexistence architecture is intentional and safe: ID3v1 sits at the very end of the file (last 128 bytes), while ID3v2 sits at the very beginning (before audio data). Players read both if present, typically preferring ID3v2 for fields that appear in both (title, artist, album, etc.) since v2 has more capacity and richer encoding. The LAME encoder, for example, writes ID3v2 tags using ffmpeg and also writes ID3v1 for maximum compatibility.

Key architectural differences between ID3v2.3 and ID3v2.4:

|| Feature | ID3v2.3 | ID3v2.4 |
|---|---|---|---|
| Frame size encoding | Big-endian uint32 | Synchsafe integer (7 bits/byte) |
| Maximum tag size | ~256 MB | ~256 MB (same limit, different encoding) |
| Text encoding terminator | Null byte ($00) | Null terminator per encoding rules |
| Multiple instances of same frame | Not recommended | Allowed (e.g., multiple TXXX frames) |
| Relative volume adjustment | RVAD frame | RVA2 frame (per-channel) |
| Encrypted frames | Supported via C flag | Supported via C flag |
| Grouping identification | Not native | TAGS frame, G flag |
| Timestamp format | Absolute frames only | Relative + absolute timestamps |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 ID3v1 Tag (at end of file, 128 bytes)

The ID3v1 tag is always exactly 128 bytes, placed at the very end of the file, immediately after the last MPEG audio frame. If an ID3v1 tag is present, it is the final 128 bytes of the file. If a file is exactly N MPEG frames with no tag, its length is a multiple of the frame size. If ID3v1 is appended, the file is N frames plus 128 bytes.

The tag begins with the ASCII string "TAG" (0x54 0x41 0x47). The presence of these three bytes is the only reliable way to detect a v1 tag. The tag is **not null-terminated** — the title field ends exactly at byte 32, and any unused space is filled with null bytes (0x00) or space characters (0x20) depending on the writing software.

**Complete ID3v1 byte layout (128 bytes total):**

```
Offset  Size  Field          Encoding     Description
------  ----  -----          --------     -----------
0x00    3     "TAG"          ASCII        Magic identifier
0x03    30    Title          ASCII/Local  Song title (30 bytes, space or null padded)
0x21    30    Artist         ASCII/Local  Artist/performer name
0x3F    30    Album          ASCII/Local  Album name
0x5D    4     Year           ASCII        4-digit year (e.g., "1999")
0x61    30    Comment        ASCII        Comment field (see v1.1 note)
0x7F    1     Genre          uint8        Genre index (0–79 standard, 80–255 extended)
```

**ID3v1.1 enhancement (track number):** The original ID3v1 specification used all 30 bytes of the comment field. ID3v1.1 repurposes this for track number storage. The convention is: if bytes 97–98 of the comment field are non-null (i.e., the first two bytes of the comment are **not** both 0x00), the field is a standard comment. If byte 97 is 0x00 and byte 98 is non-zero, the field is structured as:

```
Offset  Size  Field          Description
------  ----  -----          -----------
0x61    28    Comment        First 28 bytes of comment text (padded with spaces)
0x7D    1     Null separator Always 0x00 — signals ID3v1.1 format
0x7E    1     Track number   1–255 (0x01 to 0xFF)
0x7F    1     Genre          Genre index (same as v1)
```

The null separator at offset 97 (`0x00`) is the distinguishing feature. Players that support ID3v1.1 check for this: if byte 97 is `0x00` and byte 98 is not `0x00`, the track number is read from byte 98.

**ID3v1 genre list (standard indices 0–79):**

```
0=Blues        16=Jazz          32=Alternative    48=AlternRock
1=Classic Rock 17=Metal         33=Darkwave       49=Heavy Metal
2=Country      18=Alternative   34=Techno-Industrial 50=Black Metal
3=Dance        19=Ska           35=Electronic     51=Celtic
4=Disco        20=Death Metal   36=Pop-Folk       52=Punk
5=Funk         21=Pranks        37=Eurodance      53=Funk
6=Grunge       22=Soundtrack    38=Southern Rock  54=Drum & Bass
7=Hip-Hop      23=CD            39=Comedy         55=Club
8=Jazz         24=Doom Metal    40=Cult           56=Gothic Rock
9=Latin        25=East Coast    41=Gangsta Rap    57=Reggae
10=Metal        26=Gothic        42=Top 40         58=Rock
11=New Age      27=Grunge        43=Christian Rap  59=Ska
12=Oldies       28=Anime         44=Pop/Funk       60=Death Rock
13=Other        29=J-Pop/J-Rock  45=Nu Metal       61=Psychedelic
14=Pop          30=Synthwave     46=Funk Soul      62=Rock & Roll
15=R&B          31=Rap           47=Soul           63=Hardcore
```

Indices 80–147 are defined by WinAmp as extended genres. Indices 148–255 are reserved for user-defined genres.

**Hex diagram of a real ID3v1 tag (annotated):**

```
Last 128 bytes of file:
0x00  54 41 47                                       ; "TAG" magic identifier
0x03  4E 6F 74 68 69 6E 67 20 20 20 20 20 20 20 20  ; "Nothing     " (title, 30B)
0x13  20 20 20 20 20 20 20 20 20 20 20 20 20 20 20  ; padding
0x21  47 61 72 74 68 65 65 6E 20 20 20 20 20 20 20  ; "Garbage     " (artist, 30B)
0x31  20 20 20 20 20 20 20 20 20 20 20 20 20 20 20  ; padding
0x3F  4D 6F 6E 6B 65 79 20 20 20 20 20 20 20 20 20  ; "Monkey      " (album, 30B)
0x4F  20 20 20 20 20 20 20 20 20 20 20 20 20 20 20  ; padding
0x5D  32 30 30 30                                   ; "2000" (year, 4B)
0x61  00 02 11                                      ; v1.1: null sep + track=17
0x64  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00  ; padding (comment)
0x71  00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
0x7E  00                                             ; (padding completes to 127)
0x7F  37                                             ; Genre: 55=Club
```

### 3.2 ID3v2 Tag (at beginning of file)

The ID3v2 tag immediately precedes the first MPEG audio frame. It begins at byte 0 of the file, and the first MPEG frame sync (`0xFF`) cannot appear until after the tag's data. The tag header is exactly 10 bytes, followed by zero or more frames, followed by padding (optional).

**Complete ID3v2 header layout (10 bytes):**

```
Offset  Size  Field       Type         Description
------  ----  -----       ----         -----------
0x00    3     Identifier  "ID3"         Fixed magic bytes: 0x49 0x44 0x33
0x03    1     Version     uint8         0x02=v2.2, 0x03=v2.3, 0x04=v2.4
0x04    1     Revision    uint8         Always 0x00; increments for bug-fix revisions
0x05    1     Flags       bitfield      Flag bits (see below)
0x06    4     Size        syncsafe/v4   Size of tag data excluding header
```

**Flag byte layout (ID3v2.3 and v2.4 share the same flag bits):**

```
Bit  7 (0x80): Unsynchronisation — data has been unsynchronized
Bit  6 (0x40): Extended header — an extended header is present after the main header
Bit  5 (0x20): Experimental indicator — tag was created by experimental software
Bit  4–0 (0x1F): Reserved (must be 0)
```

The footer present flag (bit 7) only exists in ID3v2.4. In ID3v2.3, that bit is reserved. The footer is a 10-byte copy of the header placed at the very end of the tag, before the audio frames. It exists to allow backward-compatible appending of data to the end of the tag when editing without rewriting the entire file.

**Synchsafe integer (ID3v2.4 only):**

In ID3v2.4, the tag size and frame sizes use **synchsafe integers** (also called "syncsafe" or "padding" integers). Each byte uses only 7 bits; the MSB of each byte is always 0. This prevents accidental inclusion of MPEG sync bytes (`0xFF`) within the tag data, which would cause players to skip into audio data.

```
Decoding a 4-byte syncsafe integer:
value = (b0 << 21) | (b1 << 14) | (b2 << 7) | b3

Example: bytes 0x00 0x00 0x04 0x9C
value = (0x00 << 21) | (0x00 << 14) | (0x04 << 7) | 0x9C
      = 0 | 0 | 512 | 156
      = 668 bytes
```

Maximum representable syncsafe value: `0x7F 0x7F 0x7F 0x7F` = `((127<<21) | (127<<14) | (127<<7) | 127)` = `268,435,455` bytes (~256 MB). This is the theoretical maximum size of an ID3v2 tag.

**ID3v2.3 size encoding:** In ID3v2.3, the tag size field is a standard big-endian uint32 (all 32 bits used), but the practical limit is also ~256 MB due to player constraints.

**Tag size vs. frame data size:** The size field in the header includes all frames but **excludes** the 10-byte header itself. It does not include any padding. The padding follows the frames and extends to fill the space between the last frame and the start of the audio data.

### 3.3 ID3v2 Extended Header (v2.3)

The extended header in ID3v2.3 is variable length. Its presence is signaled by flag bit 6 in the main header.

**Extended header layout (ID3v2.3):**

```
Size of extended header (uint32, big-endian): minimum 6 bytes
Number of flag bytes (uint8): always 1
Extended flags (1 byte):
  bit 7: Tag is an update (crc and/or text encoding changed)
  bit 6: Tag is a CRC tag
  bit 5: Tag is restricted
  bits 4–0: reserved
CRC data (present if bit 6 set): uint32, big-endian CRC-32 of the tag data (excluding header)
```

The CRC is a standard IEEE CRC-32 (polynomial 0x04C11DB7) computed over the entire tag data (all frames, no padding). The extended header itself is **not** included in the CRC calculation.

The extended header's primary purpose in v2.3 is CRC protection for critical tag data.

### 3.4 ID3v2 Extended Header (v2.4)

ID3v2.4 has a more complex extended header. It begins with a syncsafe size field, followed by flag bytes, then data.

**Extended header layout (ID3v2.4):**

```
Extended header size (synchsafe uint32): minimum 6 bytes, always 6 in practice
Number of flag bytes (uint8): always 1
Extended flags (1 byte):
  bit 7: Tag is an update (changes only — incremental update)
  bit 6: CRC data present
  bit 5: Tag is restricted
  bits 4–0: reserved
Data:
  - If update flag: uint8 flag — 0x00 = no constraints
  - If CRC flag: uint5 padding + 28-bit CRC (synchsafe) — CRC-32 of frame data
  - If restricted flag: uint8 restriction — limits on tag size, frame count, etc.
```

ID3v2.4 also introduces the concept of **tag restriction** in the extended header:
- `b6-5 = 00`: No restriction
- `b4 = 0`: Text encoding not restricted
- `b4 = 1`: UTF-8 encoding required
- `b3-2 = 00`: No limit on number of text fields
- `b3-2 = 01`: Max 64 text frames
- `b3-2 = 10`: Max 64 text frames, each max 1024 bytes
- `b3-2 = 11`: Max 64 text frames, each max 128 bytes
- `b1-0 = 00`: No limit on image encoding dimensions
- `b1-0 = 01`: Max 256×256 pixels
- `b1-0 = 10`: Max 64×64 pixels
- `b1-0 = 11`: Max 64×64 pixels, max 64,000 bytes

### 3.5 ID3v2 Frame Header

Each frame in an ID3v2 tag follows a consistent header structure. The header identifies the frame type, its size, and its flags.

**ID3v2.3 Frame Header (10 bytes):**

```
Offset  Size  Field      Type        Description
------  ----  -----      ----        -----------
0x00    4     Frame ID   ASCII(4)    Alphanumeric: A–Z, 0–9. First char must be letter.
0x04    4     Size       uint32      Big-endian uint32 (0–2^32−1)
0x08    2     Flags      uint16      Flag bits
```

**ID3v2.4 Frame Header (10 bytes):**

```
Offset  Size  Field      Type        Description
------  ----  -----      ----        -----------
0x00    4     Frame ID   ASCII(4)    Alphanumeric: A–Z, 0–9. First char must be letter.
0x04    4     Size       syncsafe    Synchsafe uint32 (7 bits/byte, max 0x0FFFFFFF)
0x08    2     Flags      uint16      Status and format flags
```

**Frame flag bits (ID3v2.4):**

First flag byte (status flags):
```
Bit  7 (0x80): Tag alter preservation — tag software should discard this frame if unknown
Bit  6 (0x40): File alter preservation — file software should preserve if unknown
Bit  5 (0x20): Read-only flag — frame is read-only
Bit  4–0 (0x1F): Reserved
```

Second flag byte (format flags):
```
Bit  7 (0x80): Grouping identity — frame belongs to a group
Bit  6–5 (0x60): Reserved
Bit  4 (0x10): Compressed — frame data is compressed using zlib
Bit  3 (0x08): Encrypted — frame data is encrypted
Bit  2 (0x04): Synchronised — frame data is unsynchronized
Bit  1–0 (0x03): Reserved
```

**Frame flag bits (ID3v2.3):**

First flag byte:
```
Bit  7 (0x80): Tag alter preservation
Bit  6 (0x40): File alter preservation
Bit  5–0: Reserved
```

Second flag byte:
```
Bit  7 (0x80): Read-only
Bit  6–0: Reserved
```

Note: Compression and encryption flags in ID3v2.3 use the same bit positions as format flags but were applied differently. This ambiguity was resolved in ID3v2.4.

If the **grouping identity** flag is set (v2.4), the first byte of the frame data is the group identifier. This allows frames to be logically grouped (e.g., multiple APIC frames for different covers).

If the **compressed** flag is set, the frame data begins with 4 bytes indicating the decompressed size as a big-endian uint32, followed by zlib-compressed payload.

### 3.6 Complete Frame ID Registry

**Text Information Frames (T*):**

|| Frame ID | Field Name | Spec Version | Description |
|---|---|---|---|
| TIT1 | Content Group Description | v2.3+ | Group description (e.g., "Disc 1") |
| TIT2 | Title | v2.3+ | Song title |
| TIT3 | Subtitle/Description | v2.3+ | Subtitle or description |
| TPE1 | Lead Performer(s) | v2.3+ | Primary artist(s) |
| TPE2 | Band/Orchestra/Accompaniment | v2.3+ | Additional performer info |
| TPE3 | Conductor/Performer Refinement | v2.3+ | Conductor or performer credit |
| TPE4 | Interpreted, Remixed, Modified By | v2.3+ | Remixer or editor credit |
| TOPE | Original Artist(s) | v2.3+ | Original artist for cover/remake |
| TCOM | Composer(s) | v2.3+ | Music composer(s) |
| TENC | Encoded By | v2.3+ | Software/hardware used for encoding |
| TEXT | Lyricist/Text Writer | v2.3+ | Lyricist or text author |
| TLAN | Language(s) | v2.3+ | Language(s) of lyrics or dialogue |
| TLEN | Length | v2.3+ | Length in milliseconds (decimal string) |
| TMED | Media Type | v2.3+ | Source media type (e.g., "CD") |
| TMOO | Mood | v2.4+ | Mood descriptor (e.g., "Happy") |
| TALB | Album/Movie/Show Title | v2.3+ | Album name |
| TPOS | Part of Set | v2.3+ | Disc number in a multi-disc set |
| TPUB | Publisher | v2.3+ | Record label/publisher name |
| TRCK | Track Number/Position In Set | v2.3+ | Track number and optional total |
| TRDA | Recording Dates | v2.3 | Date strings (legacy, superseded) |
| TDRC | Recording Time | v2.4+ | Year or full timestamp (YYYY or YYYY-MM-DD) |
| TDRL | Release Time | v2.4+ | Date of first release |
| TDTG | Tagging Time | v2.4+ | Date/time when tag was created |
| TDAT | Date | v2.3 | Date as DDMM (legacy) |
| TIME | Time | v2.3 | Time as HHMM (legacy) |
| TYER | Year | v2.3 | Year as 4-digit string (legacy, superseded by TDRC) |
| TCON | Content Type | v2.3+ | Genre name or numeric index |
| TCMP | Compilation | v2.3+ | Boolean: 1=compilation (Apple extension) |
| TSOA | Album Sort Order | v2.4+ | Album name for alphabetical sorting |
| TSOP | Performer Sort Order | v2.4+ | Artist name for sorting |
| TSOT | Title Sort Order | v2.4+ | Title for alphabetical sorting |
| TSOC | Composer Sort Order | v2.4+ | Composer for sorting |
| TKEY | Initial Key | v2.3+ | Musical key (e.g., "C#m") |
| TBPM | BPM | v2.3+ | Beats per minute (integer) |
| TDTG | Tagging Time | v2.4+ | Timestamp of tagging |
| TDES | Podcast Description | Podcast | Podcast description |
| TIT1 | Podcast Episode | Podcast | Podcast episode name |
| TKWD | Podcast Keywords | Podcast | Podcast keywords |

**User-Defined Text Information Frame:**

|| Frame ID | Description |
|---|---|
| TXXX | User-defined text information. Multiple instances allowed. Format: encoding byte + description (terminated per encoding) + value text. |

**URL Link Frames (W*):**

|| Frame ID | Field Name | Description |
|---|---|---|
| WCOM | Commercial Information | URL to commercial info page |
| WCOP | Copyright/Legal Information | URL to copyright/legal page |
| WOAF | Official Audio Source URL | URL to official audio source |
| WOAR | Official Artist/Performer URL | URL to artist's official page |
| WOAS | Official Audio Source URL | URL to official audio source |
| WORS | Official Radio Station | URL to official radio station |
| WPAY | Payment URL | URL for payment |
| WPUB | Publishers Official Info | URL to publisher's page |
| WXXX | User-Defined URL Link | Encoding byte + description + URL |

**Comment Frame (COMM):**

|| Field | Size | Description |
|---|---|---|
| Text encoding | 1 byte | $00=ISO-8859-1, $01=UTF-16, $02=UTF-16BE, $03=UTF-8 |
| Language | 3 bytes | ISO 639-2 language code (e.g., "eng") |
| Description | variable | Short description string, terminated per encoding |
| Null terminator | 1 byte | Terminated per encoding (e.g., $00 for ISO-8859-1) |
| Actual comment | variable | Full comment text, terminated per encoding |

Multiple COMM frames are allowed. The description field is not optional — even if empty, it must be terminated. The combination of description and language should uniquely identify each comment frame.

**Attached Picture Frame (APIC):**

```
Offset  Size  Field              Description
------  ----  -----              -----------
0x00    1     Text encoding      $00=ISO-8859-1, $01=UTF-16, $02=UTF-16BE, $03=UTF-8
0x01    N     MIME type          Image MIME type string (e.g., "image/jpeg"), null-terminated
                                        If first byte is "<", the type is a PNG signature
                                        ("<PNG>" = image/png, "<JPG>" = image/jpeg)
0x??    1     Picture type       uint8: 0–20 (see picture type table)
0x??    N     Description        Image description string, null-terminated per encoding
0x??    N     Image data         Raw binary image data (JPEG, PNG, etc.)
```

**APIC Picture Types:**

```
00 = Other                         10 = Composer
01 = 32x32 file icon (PNG)         11 = Lyricist
02 = Other file icon               12 = Recording Location
03 = Front cover                    13 = During recording
04 = Back cover                     14 = During performance
05 = Leaflet page                  15 = Movie/video screenshot
06 = Media (label side of CD)      16 = Bright colored fish (intentionally humorous)
07 = Lead artist/performer          17 = Illustration
08 = Artist/performer              18 = Band/Orchestra logo
09 = Conductor                      19 = Publisher/Studio logo
```

Picture types 0x01 and 0x02 are specifically defined for small embedded icons. For cover art, the most common types are `0x03` (front cover) and `0x04` (back cover).

**Unsynchronised Lyrics/Text Frame (USLT):**

```
Offset  Size  Field              Description
------  ----  -----              -----------
0x00    1     Text encoding      $00=ISO-8859-1, $01=UTF-16, $02=UTF-16BE, $03=UTF-8
0x01    3     Language           ISO 639-2 language code
0x04    N     Content descriptor Null-terminated string per encoding (can be empty)
0x??    N     Lyrics/text        Full lyrics text, null-terminated per encoding
```

**Synchronised Lyrics/Text Frame (SYLT):**

```
Offset  Size  Field              Description
------  ----  -----              -----------
0x00    1     Text encoding      $00=ISO-8859-1, $01=UTF-16, $02=UTF-16BE, $03=UTF-8
0x01    3     Language           ISO 639-2 language code
0x04    1     Time stamp format  1=$01 (MPEG frames), 2=$02 (milliseconds)
0x05    1     Content type       0=Other, 1=Lyrics, 2=Text transcription, 3=Movement, 4=Events, 5=Chords, 6=Trivia
0x06    N     Content descriptor Null-terminated string
0x??    N     Synchronised text  Series of: timestamp (4 bytes, big-endian) + text (null-terminated)
```

**Relative Volume Adjustment (RVA2):**

```
Offset  Size  Field              Description
------  ----  -----              -----------
0x00    N     Identification     Null-terminated string identifying the application
0x??    1     Type of channel    0=Other, 1=Master volume, 2=Front right, 3=Front left,
                                       4=Back right, 5=Back left, 6=Front center, 7=Back center,
                                       8=Subwoofer
0x??    2     Volume adjustment  int16: signed, -1.0 dB = 0xF800, +1.0 dB = 0x0800
0x??    1     Bits representing peak volume: 0=no peak, 1–255=number of bits
0x??    N     Peak volume        Big-endian: number of bits from previous byte (rounded up to byte boundary)
```

**Unique File Identifier (UFID):**

```
Offset  Size  Field              Description
------  ----  -----              -----------
0x00    N     Owner identifier   Null-terminated string identifying the owner (e.g., "http://musicbrainz.org")
0x??    N     Identifier         Binary data (max 64 bytes) — unique within the given owner
```

**Music CD Identifier (MCDI):**

Binary data containing the Table of Contents (TOC) from the audio CD. Used for MusicBrainz disc lookup.

**Private Frame (PRIV):**

```
Offset  Size  Field              Description
------  ----  -----              -----------
0x00    N     Owner identifier   Null-terminated string identifying the owner
0x??    N     Private data       Binary data specific to the owner
```

The PRIV frame is the standard extension mechanism for proprietary data. Apple, LAME, ReplayGain, and many other tools store their data in PRIV frames.

**Recommended Frame IDs (deprecated, v2.3 only):**

```
TORY: Original release year (v2.3, deprecated — use TORY in v2.3, TDOR in v2.4)
IPLS: Involved People List (v2.3, deprecated — use TMCL in v2.4)
```

**Timestamp Frames (v2.4):**

```
TDRC: Recording time (YYYY, YYYY-MM, YYYY-MM-DD, YYYY-MM-DDTHH, YYYY-MM-DDTHH:MM, YYYY-MM-DDTHH:MM:SS)
TDOR: Original release time
TDRL: Release time
TDTG: Tagging time
TIPL: Involved People List (v2.4, replaced IPLS from v2.3)
TMCL: Musician Credits List (v2.4): "instrument1=artist1\0instrument2=artist2\0..."
```

### 3.7 Text Information Frames (T* frames)

All text frames share a common structure. The first byte is the **text encoding byte**, which determines how all strings in the frame are encoded. For text-only frames, there is exactly one string after the encoding byte.

**Text frame structure (single value):**

```
Offset  Size  Field              Description
------  ----  -----              -----------
0x00    1     Text encoding      $00, $01, $02, or $03
0x01    N     Text               The actual text string, terminated per encoding rules
```

**Text encoding byte values:**

```
$00 (0): ISO-8859-1 (Latin-1) — single-byte encoding covering most Western languages
$01 (1): UTF-16 with BOM — Unicode with byte-order mark (FF FE = little-endian, FE FF = big-endian)
$02 (2): UTF-16BE — big-endian Unicode without BOM
$03 (3): UTF-8 — variable-width Unicode encoding
```

**Termination rules per encoding:**

| Encoding | Terminator | Notes |
|---|---|---|
| ISO-8859-1 ($00) | Null byte ($00) | Single-byte, simple termination |
| UTF-16 with BOM ($01) | Double null ($00 $00) for multiple strings | A single null in UTF-16 is 2 bytes ($00 $00) |
| UTF-16BE ($02) | Double null ($00 $00) | No BOM, so byte order is always big-endian |
| UTF-8 ($03) | Null byte ($00) | Standard null termination |

**Multi-value fields:** Some text frames (TPE1, TCOM, TCON, etc.) support multiple values separated by a forward slash (`/`) per convention, or they can appear as multiple frames with the same Frame ID. ID3v2.4 explicitly allows multiple instances of the same frame, while ID3v2.3 recommended against it but most implementations handle it.

Example multi-value: `TPE1` = "Artist One / Artist Two / Artist Three"

**Genre handling in TCON:** The TCON frame can contain either a numeric genre index in parentheses (e.g., "(17)" = Jazz) or a freeform genre string (e.g., "Electronic Rock"). The parsing logic is:
1. If the string begins with `(`, extract the number inside parentheses and use it as the genre index.
2. Otherwise, treat the entire string as a freeform genre.

The ID3v1 genre list maps numeric indices 0–79 to standard genre names. Extended indices (80+) were originally WinAmp extensions but have become quasi-standard. Modern software uses freeform genres far more than numeric indices.

### 3.8 URL Link Frames (W* frames)

URL link frames store internet addresses. Unlike text frames, they do **not** have a text encoding byte — URLs are always encoded as ISO-8859-1.

**WXXX (User-Defined URL Link Frame):**

```
Offset  Size  Field              Description
------  ----  -----              -----------
0x00    1     Text encoding      $00=ISO-8859-1, $01=UTF-16, $02=UTF-16BE, $03=UTF-8
0x01    N     Description        Null-terminated string (can be empty if only one link)
0x??    N     URL               Null-terminated ISO-8859-1 string
```

The WXXX frame encodes the description field with the specified encoding, but the URL itself is always ISO-8859-1 (null-terminated). This is a known inconsistency in the spec.

### 3.9 Comment Frame (COMM)

The COMM frame stores a comment with a language identifier and a short description. Multiple COMM frames can exist with different language/description combinations.

```
Offset  Size  Field              Description
------  ----  -----              -----------
0x00    1     Text encoding      Encoding byte for description and comment text
0x01    3     Language           ISO 639-2 three-letter language code (e.g., "eng")
0x04    N     Short description  Null-terminated string (per encoding byte)
0x??    1     Null terminator    Per encoding: $00 for ISO-8859-1/UTF-8, $00 $00 for UTF-16
0x??    N     Actual comment    Full comment text, terminated per encoding
```

The description should identify the nature of the comment. A commonly used description is `"comment"` or `"LYRICS"`. The iTunes/Apple comment frame uses description `"iTunSMPB"` for gapless metadata and `"iTunNORM"` for sound check normalization data.

**Common COMM frame uses:**

| Description | Content | Owner/Application |
|---|---|---|
| `iTunSMPB` | Hex-encoded gapless parameters | Apple iTunes |
| `iTunNORM` | Normalization settings | Apple iTunes Sound Check |
| `GRacenote` | Gracenote CDDB data | Various players |
| `DESCRIPTION` | ReplayGain analysis info | ReplayGain 1.0 |

### 3.10 Picture Frame (APIC)

The APIC frame embeds an image file. There can be multiple APIC frames in a single tag, distinguished by picture type and/or description. Front cover (`0x03`) is the most common and most widely supported.

```
Offset  Size  Field              Description
------  ----  -----              -----------
0x00    1     Text encoding      $00=ISO-8859-1, $01=UTF-16, $02=UTF-16BE, $03=UTF-8
0x01    N     MIME type          Image MIME type (e.g., "image/jpeg", "image/png"), null-terminated
                                        OR a special code: "<PNG>" for PNG, "<JPG>" for JPEG
0x??    1     Picture type       uint8: 0–20 (see picture type table above)
0x??    N     Description        Null-terminated string (per encoding), describes the image
0x??    N     Image data         Raw binary image bytes (JPEG, PNG, GIF, BMP, etc.)
```

The image data is raw binary — no additional size field is needed because the frame's own size tells the parser where the image data ends. The MIME type or special code tells the decoder how to interpret the binary data.

**Common MIME type patterns:**
- `image/jpeg` / `<JPG>` — JPEG image (most common, smallest for photos)
- `image/png` / `<PNG>` — PNG image (lossless, supports transparency)
- `image/gif` — GIF image (rare for covers, supports animation)
- `image/bmp` — BMP image (uncompressed, rare)

The APIC frame must use the correct encoding byte for the description field. The image data itself is binary and is not encoded — only the description string is subject to the text encoding byte.

**Cover art storage recommendations:**
- Preferred format: JPEG with `image/jpeg` MIME type, picture type `0x03` (front cover)
- Preferred resolution: 500×500 to 1000×1000 pixels (enough for display, not excessive)
- Maximum recommended size: ~500 KB per image
- The APIC frame should be placed near the end of the tag (or at least after essential text frames) to make it easy to strip without losing other data

### 3.11 Unsynchronisation Scheme

Unsynchronisation is a data transformation applied to prevent false MPEG sync byte detection within the tag data. An MPEG audio frame begins with the byte `0xFF`, followed by bytes whose top 3 bits are not all zero. If an ID3v2 tag contains the sequence `0xFF 0x00`, some players or hardware could misinterpret the tag data as an audio frame, causing playback to skip part of the tag.

**ID3v2.3 unsynchronisation:**

When the unsynchronisation flag (bit 5 in header flags) is set, any occurrence of the pattern `0xFF 0x00` in the tag data is replaced with `0xFF 0x00 0x00`. This is called byte-stuffing: a null byte is inserted after every `0xFF 0x00` pattern. On reading, the extra `0x00` is removed (de-stuffed) to recover the original data.

Note: In ID3v2.3, the pattern `0xFF 0x00` within the tag must be detected **after** all frame data is assembled and **before** writing. The unsynchronisation is applied to the entire tag payload (frames + padding) as a single stream.

**ID3v2.4 unsynchronisation:**

ID3v2.4 uses a more refined approach. Instead of byte-stuffing the entire tag, it only applies unsynchronisation to individual frames where the **synchronised** flag is set in the frame's format flags. This is frame-level unsynchronisation.

Additionally, ID3v2.4 mandates that frame sizes (stored as synchsafe integers) are computed from the **unsynchronised** data length. This means: if a frame is unsynchronised, its synchsafe size reflects the length **after** unsynchronisation (with the extra bytes inserted), not the original data length.

The decoder must:
1. Read the synchsafe frame size
2. Read that many bytes of frame data
3. If unsynchronised, replace every `0xFF 0x00 0x00` sequence with `0xFF 0x00` (de-unsynchronise)

In practice, most modern encoders and decoders handle unsynchronisation correctly, but compatibility issues remain with very old or poorly implemented players.

**The `0xFF 0x00` pattern specifically:**

Not every `0xFF` byte needs to be escaped — only the two-byte sequence `0xFF 0x00`. The byte `0xFF` followed by any byte with value 0x01–0xEF is safe because those byte values cannot appear at the start of an MPEG sync pattern. However, `0xFF 0x00` is dangerous because the next byte `0x00` could be interpreted as part of an audio frame header (in some MPEG modes, `0xFF` followed by `0x00` through `0x0F` could trigger sync detection depending on the decoder).

### 3.12 Complete Field-by-Field Bitstream Map

**ID3v2 Header (10 bytes, big-endian throughout except synchsafe fields):**

```
Byte 0      [0x49]              ; 'I' — first byte of "ID3" magic
Byte 1      [0x44]              ; 'D' — second byte
Byte 2      [0x33]              ; '3' — third byte (ASCII "3" = 0x33)
Byte 3      [version]           ; 0x02, 0x03, or 0x04
Byte 4      [revision]          ; 0x00 (always)
Byte 5      [flags]             ; bit 7=unsync, 6=ext, 5=exp, 4–0=reserved
Byte 6-9    [size]              ; v2.3: uint32 BE; v2.4: syncsafe uint32
```

**ID3v2.3 Frame (10-byte header + payload):**

```
Bytes 0-3   [Frame ID]         ; 4 ASCII chars (A-Z, 0-9)
Bytes 4-7   [Size]              ; uint32 BE (0 to 2^32-1)
Bytes 8-9   [Flags]             ; uint16: b15=tag alt, b14=file alt, b7=read-only
Bytes 10-N  [Frame data]        ; Payload per frame type
```

**ID3v2.4 Frame (10-byte header + payload):**

```
Bytes 0-3   [Frame ID]          ; 4 ASCII chars
Bytes 4-7   [Size]              ; syncsafe uint32 (max 0x0FFFFFFF)
Bytes 8-9   [Flags]             ; uint16: status flags (8-15), format flags (0-7)
Bytes 10-N  [Frame data]        ; Payload per frame type
```

**Text Frame (T*) — complete bitstream:**

```
Byte 0      [encoding]          ; $00=ISO-8859-1, $01=UTF-16, $02=UTF-16BE, $03=UTF-8
Bytes 1-N   [text data]         ; Variable-length string, terminated per encoding
```

**APIC Frame — complete bitstream:**

```
Byte 0      [encoding]          ; Text encoding for description
Bytes 1-N   [MIME type]         ; ISO-8859-1 null-terminated string
Bytes 1 or 3 [special marker]   ; If "<PNG>" or "<JPG>" (5 bytes incl. null)
Byte M      [picture type]       ; uint8: 0–20
Bytes M+1-N [description]       ; Null-terminated per encoding
Bytes N+1-  [image data]         ; Binary JPEG/PNG/GIF data to end of frame
```

**COMM Frame — complete bitstream:**

```
Byte 0      [encoding]          ; Text encoding for description and comment
Bytes 1-3   [language]          ; 3 ASCII chars (ISO 639-2)
Bytes 4-N   [description]       ; Null-terminated per encoding
Byte X      [null]              ; Extra null if encoding is UTF-16 ($01 or $02)
Bytes X+1-N [comment]           ; Full comment text, null-terminated per encoding
```

---

## 4. ENCODING / WRITING ALGORITHM

Writing an ID3v2 tag involves constructing the tag header, encoding each frame with its appropriate payload, applying unsynchronisation if needed, and appending padding to fill the space before audio data.

**Step 1 — Gather and encode frame data**

For each metadata field to be written:

1. Select the appropriate frame ID (e.g., `TIT2` for title, `APIC` for cover art).
2. Determine the text encoding byte: prefer `$03` (UTF-8) for maximum compatibility with modern software, or `$00` (ISO-8859-1) for ASCII-only content. Use `$01` (UTF-16) when Unicode characters outside BMP are needed.
3. Encode the field value(s) according to the selected encoding.
4. For multi-value fields, concatenate values with ` / ` as separator.
5. For binary frames (APIC, UFID, PRIV, MCDI), copy data directly without encoding.

**Step 2 — Build frame headers**

For each frame:

1. Write the 4-byte Frame ID (ASCII).
2. Encode the frame data size as:
   - ID3v2.3: big-endian uint32
   - ID3v2.4: synchsafe uint32 (7 bits per byte, max 0x0FFFFFFF)
3. Write frame flags (usually 0x0000 if not using compression/encryption/grouping).
4. Append the encoded frame data.

**Step 3 — Compute unsynchronisation**

1. Concatenate all frame data into a byte stream.
2. Scan for `0xFF 0x00` patterns.
3. If found, set the unsynchronisation flag in the header.
4. Replace each `0xFF 0x00` with `0xFF 0x00 0x00` in the data stream (ID3v2.3), or mark affected frames with the synchronised flag (ID3v2.4).

**Step 4 — Assemble the tag**

```
[ID3v2 Header: 10 bytes]
  - "ID3" magic bytes
  - version byte (0x03 or 0x04)
  - revision byte (0x00)
  - flags byte
  - syncsafe size (v2.4) or uint32 size (v2.3)

[Frame 1 Header + Data]
[Frame 2 Header + Data]
...
[Frame N Header + Data]

[Padding: 0x00 bytes to fill space before audio data]
```

**Step 5 — Calculate padding**

The total tag size (excluding the 10-byte header) is stored in the header's size field. After computing the total size of all frame headers + frame data, append null bytes (`0x00`) to reach the desired tag size. Padding is recommended to be at least 128 bytes to allow future tag updates without rewriting the audio data. A common practice is to pad to a 2 KB or 4 KB boundary.

**ID3v1 writing algorithm:**

1. Open file and seek to end.
2. Check if last 3 bytes are "TAG". If yes, seek back 128 bytes and overwrite. If no, seek to EOF position.
3. Write the 128-byte ID3v1 tag:
   - Bytes 0–2: "TAG"
   - Bytes 3–32: title (30 bytes, space-padded)
   - Bytes 33–62: artist (30 bytes, space-padded)
   - Bytes 63–92: album (30 bytes, space-padded)
   - Bytes 93–96: year (4 ASCII digits or spaces)
   - Bytes 97–126: comment (30 bytes) OR v1.1 structured comment
   - Byte 127: genre index
4. For ID3v1.1: write byte 97 as `0x00`, byte 98 as track number.

**Special considerations for writing:**

- Writing ID3v2 must preserve the audio data unchanged. The tag is prepended to the file by reading audio data starting at the old file's byte 0, then writing the new tag, then writing the audio data back. This is why padding matters: it gives room to grow the tag in future edits without moving the audio data.
- When updating an existing tag, the safest approach is to parse the existing tag, modify the changed frames, and rewrite the entire tag with new padding. Overwriting in-place can work if the new tag is exactly the same size as the old tag, but risks corrupting the file if sizes differ.
- For ffmpeg-based writing, use `-id3v2_version 3` for maximum compatibility (ID3v2.3) or `-id3v2_version 4` for the newer format. FFmpeg's default is v3.
- Metadata must be written **after** audio encoding completes, not before. If audio encoding changes the number of audio frames (e.g., through padding/encoder delay), field values like `TLEN` must be updated to reflect the actual encoded audio length.

---

## 5. DECODING / READING ALGORITHM

### 5.1 ID3v2 Detection and Parsing

The detection and parsing algorithm must handle both ID3v2.3 and ID3v2.4, as well as the older ID3v2.2 (which uses 3-byte frame IDs and a different header format). The algorithm must also gracefully handle malformed tags, padding variations, and unsynchronisation.

**Detection step:**

1. Open the file in binary mode and read the first 10 bytes.
2. If bytes 0–2 equal `0x49 0x44 0x33` ("ID3"), an ID3v2 tag is present.
3. Parse the version byte (byte 3): `0x02` = ID3v2.2, `0x03` = ID3v2.3, `0x04` = ID3v2.4.
4. Read the flags byte (byte 5).
5. Read the size:
   - ID3v2.2: big-endian uint32 (bytes 4–7, but only 3 bytes actually used for size)
   - ID3v2.3: big-endian uint32 (bytes 6–9)
   - ID3v2.4: syncsafe uint32 (bytes 6–9)

**Tag size interpretation:**

The size field in the header tells how many bytes follow the header (frames + padding, excluding the header itself). The total tag occupies `10 + size` bytes.

```
ID3v2.3: total_tag_size = 10 + (b6 << 24 | b7 << 16 | b8 << 8 | b9)
ID3v2.4: total_tag_size = 10 + ((b6 & 0x7F) << 21 | (b7 & 0x7F) << 14 | (b8 & 0x7F) << 7 | (b9 & 0x7F))
```

After reading the header, seek to byte offset `10 + size` to find the start of the audio data. The audio data begins at the first `0xFF` byte where the next byte's top 3 bits are set (MPEG sync).

**ID3v2.2 compatibility (legacy):**

ID3v2.2 uses 3-byte frame IDs (e.g., "TT2" instead of "TIT2", "PIC" instead of "APIC"). The header is 6 bytes: 3-byte magic "ID3" + version byte + flags byte + size (3 bytes, each using only 7 bits like syncsafe). If ID3v2.2 is detected, translate frame IDs using the official mapping table:

```
TT2 → TIT2   TP1 → TPE1   TAL → TALB   TRK → TRCK
TYE → TYER   TYE → TYER   TCO → TCON   TEN → TENC
TP2 → TPE2   TP3 → TPE3   TP4 → TPE4   TCM → TCOM
TXT → TEXT   TLA → TLAN   TLE → TLEN   TMT → TMED
MCI → MCDI   ETC → ETC    IPL → IPLS   TXX → TXXX
WXX → WXXX   PIC → APIC   COM → COMM
```

### 5.2 Frame Parsing

Once the tag is located, iterate through frames:

1. Read the frame header (10 bytes for v2.3/v2.4, 6 bytes for v2.2).
2. Read the frame ID (4 ASCII characters).
3. Read the frame size:
   - ID3v2.3: big-endian uint32
   - ID3v2.4: syncsafe uint32
   - ID3v2.2: 3-byte syncsafe (effectively 21 bits of data)
4. Read the frame flags (2 bytes).
5. If frame ID is all zeros (padding between frames), skip to the next 10-byte boundary.
6. Read `size` bytes of frame data.
7. If the frame is unsynchronised (frame flag bit 2 set), de-unsynchronise by replacing `0xFF 0x00 0x00` with `0xFF 0x00`.
8. If the frame is compressed (frame flag bit 4 set in v2.4), decompress using zlib before further processing.
9. If the frame is encrypted (frame flag bit 3 set), decrypt using the specified method (no standard encryption algorithm is defined in the spec; this is implementation-dependent).
10. If the frame has grouping identity (frame flag bit 7 set in v2.4), remove the first byte as the group identifier.
11. Parse the frame data according to the frame type (text, URL, comment, picture, etc.).
12. Repeat until the accumulated offset reaches the tag size (header size field value).

**Parsing complete — loop:**

```
offset = 0
while offset < tag_size:
    header = read(10 bytes)
    frame_id = header[0:4]
    if frame_id == "\x00\x00\x00\x00": break  # padding
    frame_size = decode_size(header[4:8])
    frame_flags = header[8:10]
    frame_data = read(frame_size bytes)
    # de-unsynchronise, decompress, decrypt, de-group as needed
    parse_frame(frame_id, frame_data)
    offset += 10 + frame_size
```

**Frame size decoding — v2.3 vs v2.4:**

```python
def decode_frame_size_v23(raw):
    return (raw[0] << 24) | (raw[1] << 16) | (raw[2] << 8) | raw[3]

def decode_frame_size_v24(raw):
    return ((raw[0] & 0x7F) << 21) | ((raw[1] & 0x7F) << 14) | \
           ((raw[2] & 0x7F) << 7) | (raw[3] & 0x7F)
```

### 5.3 Encoding Detection

When parsing text frames, the first byte is the **text encoding byte** and determines how to decode all strings in that frame. The encoding byte applies to:
- All text in T* frames
- Description and comment text in COMM frames
- Description in USLT/SYLT frames
- Description in APIC/WXXX/TXXX frames

**UTF-16 detection:** When the encoding byte is `$01` (UTF-16 with BOM), the first 2 bytes of the string are the Byte Order Mark:
- `0xFF 0xFE` → little-endian UTF-16 (LSB first, Intel/ARM byte order)
- `0xFE 0xFF` → big-endian UTF-16 (MSB first, network byte order)

UTF-16 strings are null-terminated by a 2-byte null (`0x00 0x00`). Single characters take 2 bytes (or 4 bytes for characters outside the Basic Multilingual Plane, represented as surrogate pairs).

**Reading text frames:**

```python
def read_text_frame(data):
    encoding = data[0]
    text = data[1:]

    if encoding == 0x00:  # ISO-8859-1
        return text.split(b'\x00')[0].decode('latin-1')
    elif encoding == 0x01:  # UTF-16 with BOM
        if text.startswith(b'\xff\xfe'):
            return text[2:].decode('utf-16-le').split('\x00')[0]
        elif text.startswith(b'\xfe\xff'):
            return text[2:].decode('utf-16-be').split('\x00')[0]
        else:
            return text.decode('utf-16-be').split('\x00')[0]
    elif encoding == 0x02:  # UTF-16BE
        return text.decode('utf-16-be').split('\x00')[0]
    elif encoding == 0x03:  # UTF-8
        return text.split(b'\x00')[0].decode('utf-8')

    return text.split(b'\x00')[0].decode('utf-8', errors='replace')
```

**Genre string parsing from TCON:**

```python
def parse_genre(genre_str):
    import re
    # Match genre index in parentheses: "(17)" or "(17)Electronic"
    match = re.match(r'^\((\d+)\)(.*)$', genre_str)
    if match:
        index = int(match.group(1))
        refinements = match.group(2).strip()
        return {'index': index, 'genre': refinements or ID3V1_GENRES.get(index, 'Unknown')}
    return {'index': None, 'genre': genre_str}
```

### 5.4 Padding and Size Computation

Padding in ID3v2 is a region of `0x00` bytes between the last frame and the start of audio data. Its purpose is to allow tag editing: when adding or modifying frames, the new tag may be slightly larger than the old tag. With sufficient padding, the audio data does not need to be moved.

**Reading padding:** When iterating through frames, if a 10-byte region consists entirely of `0x00`, it is padding and the parsing loop should terminate (or continue past the padding to the end of the tag size). Some parsers treat any frame with ID `\x00\x00\x00\x00` as padding and break the loop.

**Computing tag data size:** The size field in the header represents the number of bytes from the first frame to the last byte of padding (inclusive). If padding is present, `header_size + all_frame_sizes + padding_size = tag_data_size`.

**Safe padding amounts:**
- Minimum recommended padding: 128 bytes
- Standard padding: 1 KB to 4 KB
- Maximum reasonable padding: ~128 KB (prevents excessively large files)

Very old implementations sometimes interpreted padding bytes as part of a frame (reading a zero-size frame as an infinitely large frame, reading past the end of the tag). Modern implementations stop at the tag size boundary.

### 5.5 Handling Unsynchronisation

**Reading (de-unsynchronisation):**

If the header flag unsynchronisation bit is set, the entire tag data (after the header, up to `size` bytes) has been byte-stuffed. Replace each `0xFF 0x00 0x00` sequence with `0xFF 0x00`:

```python
def de_unsynchronise(data):
    result = bytearray()
    i = 0
    while i < len(data):
        if i < len(data) - 2 and data[i] == 0xFF and data[i+1] == 0x00 and data[i+2] == 0x00:
            result.append(0xFF)
            result.append(0x00)
            i += 3
        else:
            result.append(data[i])
            i += 1
    return bytes(result)
```

Note that this must be applied to the entire tag payload **before** iterating through frames. After de-unsynchronisation, the `size` field in the header is no longer accurate — it describes the synchronised data length, not the de-unsynchronised length. The de-unsynchronised data is always smaller than the synchronised data.

**Frame-level unsynchronisation (ID3v2.4):**

In ID3v2.4, if a specific frame has the synchronised flag set, only that frame's data needs de-unsynchronisation. The synchsafe size in the frame header already accounts for the synchronised length. Read the frame size (synchsafe), read that many bytes, then de-unsynchronise within those bytes.

---

## 6. METADATA ARCHITECTURE

### 6.1 Complete Supported Tag Fields

|| Frame ID | Field Name | Max Length | Encoding | Multi-value | Notes |
|---|---|---|---|---|---|---|
| TIT2 | Title | Unlimited (per tag size) | All | No | Primary song title |
| TPE1 | Artist | Unlimited | All | Yes (separator `/`) | Lead performer(s) |
| TALB | Album | Unlimited | All | No | Album name |
| TDRC | Recording Year | 19 chars | All | No | v2.4: full timestamp; v2.3: use TYER |
| TYER | Year (v2.3) | 4 chars | All | No | v2.3 only; superseded by TDRC |
| TRCK | Track Number | 4+2 chars | All | No | Format: `N` or `N/M` (total) |
| TPOS | Part of Set | 4+2 chars | All | No | Disc number; format: `N` or `N/M` |
| TCON | Genre | Unlimited | All | Yes (refinements) | Numeric index or freeform text |
| TCOM | Composer | Unlimited | All | Yes | Music composer(s) |
| TENC | Encoded By | Unlimited | All | No | Software/hardware name |
| TPUB | Publisher | Unlimited | All | No | Record label or publisher |
| TLEN | Length | 11 chars | All | No | Milliseconds, decimal string |
| TKEY | Initial Key | 6 chars | All | No | Musical key (e.g., "C#m") |
| TBPM | BPM | 5 chars | All | No | Beats per minute, integer |
| TENC | Encoded By | Unlimited | All | No | Encoder name and version |
| TSOA | Album Sort Order | Unlimited | All | No | For alphabetical sorting |
| TSOP | Artist Sort Order | Unlimited | All | Yes | For alphabetical artist sorting |
| TSOT | Title Sort Order | Unlimited | All | No | For alphabetical title sorting |
| APIC | Cover Art | Unlimited (per tag size) | Description only | Yes (multiple) | MIME type + binary image data |
| COMM | Comment | Unlimited | All | Yes (different languages/descs) | Language + description + text |
| USLT | Unsynchronized Lyrics | Unlimited | All | Yes | Language + lyrics text |
| TXXX | User-Defined Text | Unlimited | All | Yes | Description + value |
| WXXX | User-Defined URL | Unlimited | URL is ISO-8859-1 | Yes | Description + URL |
| UFID | Unique File Identifier | 64 bytes | Binary | Yes | Owner ID + binary identifier |
| PRIV | Private Frame | Unlimited | Binary | Yes | Owner ID + private data |
| RVA2 | Relative Volume | Unlimited | Binary | Yes (channels) | Channel identification + adjustment |
| MCDI | Music CD Identifier | Unlimited | Binary | No | CD TOC binary data |
| TOLY | Original Lyricist | Unlimited | All | Yes | Original lyricist |
| TOPE | Original Artist | Unlimited | All | Yes | Original artist for cover songs |
| TMED | Media Type | Unlimited | All | No | e.g., "CD", "Vinyl", "Digital Audio" |
| TCMP | Compilation | 1 char | All | No | Apple extension: "1" = yes |
| TDES | Podcast Description | Unlimited | All | No | Podcast-specific |
| TIT1 | Content Group | Unlimited | All | No | Group description (not episode title) |

### 6.2 Genre List (ID3v1 genres and v2 genre mapping)

The ID3v1 genre system uses a single byte (0–255) as an index into a predefined genre list. ID3v2.3 and later support both numeric indices (for compatibility with ID3v1) and freeform genre text.

**Standard ID3v1 genres (indices 0–79):**

```
0=Blues         1=Classic Rock   2=Country        3=Dance
4=Disco         5=Funk          6=Grunge         7=Hip-Hop
8=Jazz          9=Latin         10=Metal          11=New Age
12=Oldies       13=Other         14=Pop            15=R&B
16=Reggae       17=Rock          18=Techno         19=Industrial
20=Alternative  21=Ska          22=Death Metal    23=Pranks
24=Soundtrack   25=CD            26=Doom Metal     27=East Coast
28=Gothic       29=Grunge       30=Anime          31=J-Pop
32=Synthwave    33=Rap          34=Eurodance      35=Eurodance (alt)
36=Broadway     37=Irish         38=J-Rap          39=Showtunes
40=Trailer      41=Lo-Fi         42=Tribal         43=Acid Punk
44=Acid         45=Contemporary Christian 46=Christian Rock 47=Merengue
48=Salsa        49=Thrash Metal  50=Anime          51=J-Pop (alt)
52=Dancehall    53=Rock & Roll   54=Funk Soul      55=Dance Hall
56=Gothic Rock  57=Drum & Bass  58=Club-House     59=Club-House (alt)
60=Hardcore     61=Terror        62=Indie         63=Gangsta Rap
64=Cover        65=Euro House    66=Dance Pop      67=Soul
68=Brazilian    69=Bitmetal      70=Afrobeat       71=French
72=Indie Rock   73=Gothic Metal 74=Funk Metal     75=Drum
76=New Wave      77=Soft Rock    78=Cabaret        79=Swing
```

**WinAmp extended genres (indices 80–147):**

```
80=American     81=Contemporary   82=Dance Hall    83=Dirty South
84=Dirty South  85=Freestyle     86=Groove        87=Groove (alt)
88=Hip-Hop     89=Hip-Hop (alt) 90=Hip Hop       91=Indie
92=Industrial   93=Jam Band      94=Krautrock     95=Latin
96=Math         96=Metalcore     97=Post-Rock     98=Metalcore
99=Moot         100=Post-Punk    101=Post-Rock    102=Power Pop
103=Psytrance   104=Punk         105=Punk (alt)  106=Rap
107=Rap (alt)   108=Rock         109=Rock (alt)  110=Screamo
111=Seattle      112=Shoegaze    113=Shoegaze (alt) 114=Stoner Rock
115=Surf        116=Symphonic    117=Symphonic Metal 118=Symphonic Metal (alt)
119=Symphony    120=Synthpop     121=Synthpop (alt) 122=Tech House
123=Techno      124=Techno (alt) 125=Trance        126=Trance (alt)
127=Trip Hop    128=Vaporwave    129=Vocal trance  130=Vocal trance (alt)
131=West Coast  132=Witch House  133=World         134=World (alt)
135=Neue Deutsche Welle  136=Neue Deutsche Härte  137=German Folk  138=German Pop
139=German Punk  140=Krautrock (alt) 141=Lieder  142=Deutsch Rock
143=Deutsch Pop  144=German Hardcore 145=Gothic Industrial 146=Gothic Industrial (alt)
147=Chanson     148=Chanson (alt)
```

**Genre refinements:** In ID3v2, genres like "(17)Rock" combine the numeric base genre with a refinement. The parentheses-enclosed number refers to the ID3v1 genre index, while the text after it is additional information.

### 6.3 Padding Usage

Padding in ID3v2 is allocated space filled with `0x00` bytes, positioned between the last frame and the start of the audio data. It serves three purposes:

1. **Edit accommodation:** The primary purpose of padding is to allow tag editors to add, modify, or remove frames without rewriting the audio portion of the file. If a new frame is slightly larger than the old one, the padding absorbs the difference.

2. **Alignment:** Some implementations use padding to align the start of the audio data to a convenient boundary (e.g., a 4 KB boundary for filesystem efficiency).

3. **Placeholder for future data:** Padding can be explicitly reserved by tag editors for known future extensions, such as embedded album art that may be added later.

**Padding in reading:** A robust parser must handle padding correctly:
- Any sequence of 10 or more `0x00` bytes within the tag data should be treated as padding.
- Parsing should stop at the tag boundary (header size + declared size), not assume the last frame ends exactly where padding begins.
- Padding size can be computed as: `padding = tag_data_size - sum(frame_header_size + frame_data_size for all frames)`.

**Padding in writing:** When writing a new tag:
- Always include padding. A minimum of 128 bytes is recommended.
- Round padding up to a convenient size (e.g., 1024 bytes, 2048 bytes).
- The padding size should be documented in the file format for future editors.

---

## 7. FFMPEG IMPLEMENTATION REFERENCE

### 7.1 Reading Tags with ffprobe

FFmpeg and ffprobe read ID3v2 tags from MP3 files automatically as part of standard format probing. All metadata is exposed through ffprobe's format and stream output.

```bash
# Read all metadata from an MP3 file
ffprobe -v quiet -show_format input.mp3

# Read in JSON format for programmatic parsing
ffprobe -v quiet -print_format json -show_format -show_streams input.mp3

# Show only tags (formatted)
ffprobe -v quiet -show_entries format_tags -show_entries stream_tags input.mp3

# Extract specific tag fields
ffprobe -v quiet -show_entries format_tag=title,artist,album,track -of default=noprint_wrappers=1 input.mp3

# Read ID3v2.4 tags specifically
ffprobe -v quiet -show_format input.mp3 | grep -E "^TAG:"

# Check for APIC (cover art)
ffprobe -v quiet -show_format input.mp3 | grep -i "comment\|picture\|image"
```

FFmpeg reads both ID3v2 and ID3v1 tags. When both are present, ID3v2 fields take precedence for display. The `ffprobe` output includes:
- `TAG` fields for ID3v1
- `ID3` fields for ID3v2
- Additional fields like `ID3V1_TAG_*` for legacy tags

### 7.2 Writing Tags with ffmpeg

FFmpeg's `-metadata` option writes ID3v2 tags during encoding or transcoding. The specific ID3v2 version used depends on the `-id3v2_version` option (default: 3 for maximum compatibility).

```bash
# Write ID3v2.3 tags (most compatible)
ffmpeg -i input.wav -c:a libmp3lame -q:a 2 \
  -metadata title="Song Title" \
  -metadata artist="Artist Name" \
  -metadata album="Album Name" \
  -metadata year=2024 \
  -metadata track=4 \
  -metadata genre="Rock" \
  -id3v2_version 3 \
  output.mp3

# Write ID3v2.4 tags (modern format)
ffmpeg -i input.wav -c:a libmp3lame -q:a 2 \
  -metadata title="Song Title" \
  -metadata artist="Artist Name" \
  -metadata album="Album Name" \
  -metadata date=2024-03-15 \
  -metadata track=4 \
  -metadata genre="Rock" \
  -id3v2_version 4 \
  output.mp3

# Write cover art using a filter
ffmpeg -i input.wav -i cover.jpg -c:a libmp3lame -q:a 2 \
  -metadata title="Song Title" \
  -metadata artist="Artist" \
  -map 0:a -map 1:v \
  -id3v2_version 3 \
  -disposition:1 attached_pic \
  output.mp3

# Write cover art using -metadata flag with a special key
ffmpeg -i input.wav -c:a libmp3lame -q:a 2 \
  -metadata title="Song Title" \
  -metadata artist="Artist" \
  -metadata:s:a comment="Cover (front)" \
  -id3v2_version 3 \
  output.mp3
```

Note: FFmpeg's handling of APIC through `-metadata:s:a comment=` is a workaround. For reliable cover art embedding, use `ffmpeg -i input.mp3 -i cover.jpg -c:a copy -c:v copy -map 0:a -map 1:v -metadata:s:v comment="Cover" output.mp3` to copy the image stream as an attached picture.

### 7.3 Preserving Tags During Transcode

When transcoding MP3 to another format, metadata must be explicitly handled because some formats (WAV, PCM) do not support in-stream tags.

```bash
# Preserve all metadata when transcoding MP3 → FLAC
ffmpeg -i input.mp3 -c:a flac -compression_level 8 -c:a copy \
  -map_metadata 0 output.flac

# Strip metadata and re-add specific fields
ffmpeg -i input.mp3 -c:a libmp3lame -q:a 2 \
  -map_metadata -1 \
  -metadata title="New Title" \
  -metadata artist="New Artist" \
  output.mp3

# Copy metadata from source to destination (all streams)
ffmpeg -i input.mp3 -c:a libmp3lame -q:a 2 \
  -map_metadata 0 \
  output.mp3

# Copy metadata from a specific input file
ffmpeg -i source.mp3 -i audio.wav -map_metadata 0 \
  -c:a libmp3lame -q:a 2 \
  output.mp3
```

The `-map_metadata` option controls which input's metadata is carried over:
- `-map_metadata 0` — copy metadata from first input
- `-map_metadata -1` — disable metadata mapping (no metadata in output)
- `-map_metadata 0:g` — copy global metadata from first input (not per-stream)

### 7.4 Metadata Key Mapping

FFmpeg uses its own internal metadata key names, which map to ID3v2 frame IDs. When reading, FFmpeg exposes these as standard field names. When writing, these standard names are mapped to appropriate ID3v2 frames.

**FFmpeg → ID3v2.3 mapping:**

|| FFmpeg Key | ID3v2.3 Frame | Notes |
|---|---|---|---|
| `title` | TIT2 | Song title |
| `artist` | TPE1 | Lead performer |
| `album` | TALB | Album name |
| `date` / `year` | TYER | Year (v2.3 uses TYER, not TDRC) |
| `track` | TRCK | Track number |
| `disc` | TPOS | Disc number |
| `genre` | TCON | Genre (numeric or text) |
| `comment` | COMM | Comment frame |
| `composer` | TCOM | Composer |
| `encodedby` | TENC | Encoder name |
| `copyright` | TCOP | Copyright |
| `publisher` | TPUB | Publisher |
| `BPM` | TBPM | Beats per minute |
| `language` | TLAN | Language |
| `lyrics` | USLT | Unsynchronized lyrics |
| `album_artist` | TPE2 | Album performer/band |
| `performer` | TPE1 | Performer (synonym) |
| `conductor` | TPE3 | Conductor |
| `remixer` | TPE4 | Remixer |
| `key` | TKEY | Musical key |
| `length` | TLEN | Length in milliseconds |

**Critical differences between v2.3 and v2.4 mapping:**

The `date` metadata key maps to:
- TYER (ID3v2.3) — 4-digit year string
- TDRC (ID3v2.4) — full or partial timestamp (YYYY or YYYY-MM-DD)

When writing with `-id3v2_version 3`, FFmpeg truncates date to a 4-digit year. When writing with `-id3v2_version 4`, FFmpeg writes the full date string.

**ReplayGain metadata keys:**

|| Key | Frame | Description |
|---|---|---|---|
| `REPLAYGAIN_TRACK_GAIN` | TXXX (description="REPLAYGAIN_TRACK_GAIN") | Track gain adjustment in dB |
| `REPLAYGAIN_TRACK_PEAK` | TXXX (description="REPLAYGAIN_TRACK_PEAK") | Track peak amplitude |
| `REPLAYGAIN_ALBUM_GAIN` | TXXX (description="REPLAYGAIN_ALBUM_GAIN") | Album gain adjustment in dB |
| `REPLAYGAIN_ALBUM_PEAK` | TXXX (description="REPLAYGAIN_ALBUM_PEAK") | Album peak amplitude |

ReplayGain data is stored in TXXX user-defined text frames. The description field is `REPLAYGAIN_TRACK_GAIN`, `REPLAYGAIN_TRACK_PEAK`, etc., and the value field contains the numeric value.

---

## 8. TAG COLLISION HANDLING

Tag collision occurs when both ID3v1 and ID3v2 tags are present in the same file, or when multiple instances of the same frame type exist within a single ID3v2 tag.

**ID3v1 vs. ID3v2 precedence:**

Standard practice is to prefer ID3v2 data over ID3v1 data when both exist. ID3v2 has richer encoding, more fields, and larger capacity. ID3v1 exists as a compatibility layer for very old software that cannot read v2 tags.

**Field-level precedence (when both tags exist):**

| Field | Recommended Source | Notes |
|---|---|---|
| Title | ID3v2 (TIT2) | v2 is authoritative |
| Artist | ID3v2 (TPE1) | v2 is authoritative |
| Album | ID3v2 (TALB) | v2 is authoritative |
| Year | ID3v2 (TYER/TDRC) | v2 is authoritative |
| Track | ID3v2 (TRCK) | v2 is authoritative |
| Genre | ID3v2 (TCON) | Prefer v2 text over v1 index |
| Comment | ID3v2 (COMM) | v2 can hold multiple comments |
| Cover art | ID3v2 (APIC) | v1 has no cover art support |

**Multiple frames of the same type (ID3v2.4):**

ID3v2.4 explicitly allows multiple frames with the same Frame ID. This is commonly used for:
- Multiple TXXX frames with different descriptions (e.g., separate ReplayGain fields)
- Multiple APIC frames for multiple images
- Multiple COMM frames in different languages
- Multiple TPE1 frames for multiple artists

**Reading multiple instances:**
```python
def read_all_frames(tag_data, frame_id):
    frames = []
    for frame in iterate_frames(tag_data):
        if frame.frame_id == frame_id:
            frames.append(frame)
    return frames
```

**Writing with multiple instances:** When writing, aggregate all values for a given field into multiple frames rather than overwriting. For example, if writing three artists, either write `TPE1 = "Artist1 / Artist2 / Artist3"` or write three `TPE1` frames.

**ID3v2.3 multiple-frame handling:** While ID3v2.3 technically allows multiple frames, many implementations overwrite rather than append. A robust reader should handle both cases: if multiple `TPE1` frames exist, concatenate them with the separator.

**Conflict between multiple frames of the same type:**
- Text fields: concatenate with ` / ` separator
- APIC: present all images; front cover (type 0x03) takes precedence for display
- COMM: present all comments; present a UI for switching between them
- TXXX: match by description field; present all distinct user-defined fields

---

## 9. TAG STRIPPING

Tag stripping means removing metadata tags from a file, leaving only the raw audio data. This is sometimes needed for privacy (removing personal tags before sharing) or for creating minimal files.

**Stripping ID3v2:**

Remove the first N bytes of the file, where N = `10 + tag_data_size`. The audio data begins at that offset. After stripping, the file begins with an MPEG sync byte (`0xFF`).

```python
def strip_id3v2(file_path):
    with open(file_path, 'rb') as f:
        header = f.read(10)
        if header[:3] != b'ID3':
            return  # No ID3v2 tag
        version = header[3]
        if version == 0x04:
            size = ((header[6] & 0x7F) << 21) | ((header[7] & 0x7F) << 14) | \
                   ((header[8] & 0x7F) << 7) | (header[9] & 0x7F)
        else:  # v2.3
            size = (header[6] << 24) | (header[7] << 16) | \
                   (header[8] << 8) | header[9]
        # Read audio data from offset 10 + size
        f.seek(10 + size)
        audio_data = f.read()
    # Write audio data back to file
    with open(file_path, 'wb') as f:
        f.write(audio_data)
```

**Stripping ID3v1:**

Remove the last 128 bytes of the file, if the last 3 bytes are "TAG". Check for the "TAG" identifier before stripping.

```python
def strip_id3v1(file_path):
    with open(file_path, 'rb') as f:
        f.seek(-128, 2)  # Seek to 128 bytes before EOF
        tag_marker = f.read(3)
        if tag_marker == b'TAG':
            audio_size = os.path.getsize(file_path) - 128
            with open(file_path, 'rb') as f:
                audio_data = f.read(audio_size)
            with open(file_path, 'wb') as f:
                f.write(audio_data)
```

**FFmpeg tag stripping:**

```bash
# Strip all metadata
ffmpeg -i input.mp3 -c:a copy -map_metadata -1 output.mp3

# Strip only ID3v2 (keep ID3v1)
ffmpeg -i input.mp3 -c:a copy -id3v2_version 0 output.mp3

# Strip ID3v1 and write specific metadata
ffmpeg -i input.mp3 -c:a libmp3lame -q:a 2 -metadata TITLE="" -metadata ARTIST="" \
  -map_metadata -1 output.mp3
```

**Stripping specific frames:**

To remove individual frames while keeping others:
1. Parse the existing tag
2. Filter out the unwanted frame IDs
3. Rewrite the tag with remaining frames + padding

This is useful for removing embedded cover art (APIC frames) without losing other metadata, or for stripping only ReplayGain data.

---

## 10. KNOWN ISSUES & EDGE CASES

**ID3v1 truncation:** ID3v1 fields are exactly 30 bytes. Software that writes text longer than 30 bytes silently truncates it. Reading back, the truncated text appears as expected, but the original longer text is lost. Always prefer ID3v2 for fields that may exceed 30 characters.

**ID3v1 year as non-numeric:** The year field is defined as 4 bytes of ASCII text. Some software writes non-numeric values like "N/A" or leaves it blank (spaces). Parsers must handle non-numeric year values gracefully.

**ID3v1.1 track number ambiguity:** The track number detection (byte 97 = 0x00) can produce false positives if a comment actually begins with a null byte. This is extremely rare in practice but theoretically possible. Some parsers additionally require that byte 98 is non-zero and byte 99 is a space or null.

**ID3v2.4 syncsafe integer off-by-one:** The maximum syncsafe value is `0x7F 0x7F 0x7F 0x7F` = 268,435,455 bytes. Some buggy encoders write `0xFF` bytes in the size field, which are invalid (MSB must be 0). Parsers should mask each byte with `0x7F` to handle this gracefully.

**UTF-16 null terminator confusion:** UTF-16 uses 2-byte characters. A null terminator in UTF-16 is `0x00 0x00` (2 bytes). Some implementations incorrectly use a single `0x00` byte, breaking string parsing. When reading UTF-16, always treat `0x00 0x00` as the null terminator, not just `0x00`.

**UTF-16 BOM inconsistency:** Some encoders write UTF-16 without a BOM, expecting the reader to assume big-endian (no BOM is the UTF-16BE convention). Others write little-endian without a BOM. The spec says UTF-16 implies BOM for byte-order detection. A robust reader should handle the four cases: BE BOM present, LE BOM present, no BOM (assume BE), no BOM (assume LE based on stream context).

**Padding interpreted as frame:** Very old or buggy implementations sometimes treat padding `0x00` bytes as the frame ID portion of a frame header, reading garbage size values and potentially reading past the end of the file. Modern parsers check the tag size boundary explicitly and stop parsing at `10 + tag_size`.

**ID3v2 footer misinterpretation:** ID3v2.4 can have a 10-byte footer identical to the header placed after all frames and padding. Some parsers that do not check the tag size field may interpret this footer as an additional tag header, creating an infinite loop or misreading frames. The footer is rare in practice.

**Duplicate frame IDs in v2.3:** While v2.4 explicitly supports multiple frames of the same type, v2.3 does not. Some software (including older versions of FFmpeg) writes multiple TXXX frames with different descriptions, while other software overwrites the first TXXX frame when writing a new one. Readers must handle both patterns.

**Cover art MIME type inconsistencies:** Some encoders write the MIME type as `image/jpeg`, others as `image/jpg`, and older encoders use the special codes `<JPG>` and `<PNG>`. Readers should accept all variants. The image data itself (JPEG signature `0xFF 0xD8` or PNG signature `0x89 0x50 0x4E 0x47`) is a more reliable indicator of the actual format than the MIME type string.

**Unsynchronisation of frame boundaries:** When unsynchronisation is applied at the frame level (ID3v2.4), the frame boundaries shift. The synchsafe size in the frame header is the size of the unsynchronised data, but the de-unsynchronisation replacement (`0xFF 0x00 0x00` → `0xFF 0x00`) restores the original boundary. The total tag size in the header, however, reflects the unsynchronised length. This creates a consistency requirement: `sum of frame synchsafe sizes + padding = tag_data_size`.

**Frame flag compression on v2.3:** In ID3v2.3, compression was indicated by setting bit 0 of the second flag byte. In v2.4, compression is bit 4 of the first format flag byte. Some implementations confuse these, causing frames to be misread. The safest approach is to ignore compression flags for unknown frame types (treating them as data).

**LAME tag in INFO frame:** LAME stores encoder info in an INFO tag within the first MPEG frame's main data area, not in an ID3v2 frame. This is separate from ID3v2 tags. The INFO tag begins with "LAME" followed by version info, delay/padding values, and ReplayGain data. It is not part of the ID3 standard but is widely recognized.

**Zero-byte files:** A file consisting only of an ID3v2 tag (no audio frames) is technically valid but not playable as audio. Some tag editors produce such files accidentally when writing tags before encoding.

**Tag size greater than file size:** Malformed files may have an ID3v2 header claiming a size larger than the actual file. Readers must verify that `10 + size <= file_size` before attempting to read.

---

## 11. REFERENCE IMPLEMENTATIONS

**TagLib (C++/Qt):** The most complete open-source library for reading and writing ID3v1 and ID3v2 tags. Used by Amara, Clementine, and many other applications. TagLib 2.0+ has full ID3v2.4 support including UTF-8 encoding.

**Mutagen (Python):** A mature Python library supporting all ID3v1 and ID3v2 versions. Provides both high-level tag dictionaries and low-level frame access. Handles unsynchronisation, multiple text encodings, and frame iteration correctly.

```python
from mutagen.mp3 import MP3
audio = MP3("input.mp3")
print(audio.tags["TIT2"])      # Title
print(audio.tags["APIC:"].data) # Cover art binary
```

**ID3Python (Python):** Lower-level ID3v2 parser. Less actively maintained than Mutagen but provides frame-level control.

**jAudioTagger (Java):** Full ID3v1/v2 support in Java. Used by Jaikoz and other Java-based music managers.

**MediaInfo (C++):** Library and tool for reading MP3 metadata. Does not write tags.

**kid3 (Qt/C++):** Full-featured tag editor with command-line interface (kid3-cli). Supports reading and writing all ID3v1 and ID3v2 frames, batch editing, and genre management.

**FFmpeg / libavformat (C):** Reads and writes ID3v2 tags through the format layer. Uses `-metadata` for tag access. Does not expose all frame types (some are converted to standard keys).

**lame (C):** The LAME encoder writes ID3v2 tags via its own built-in tag-writing functions. It also writes the LAME INFO frame (stored as MPEG frame data, not ID3v2).

---

## 12. RELEVANT SPECIFICATIONS & FURTHER READING

**Primary Specifications:**
- ID3v2.4 spec: `https://id3.org/id3v2.4.0-structure` — Official ID3v2.4 structure specification
- ID3v2.3 spec: `https://id3.org/id3v2.3.0` — Official ID3v2.3 specification
- ID3v2.2 spec: `https://id3.org/id3v2.2.0` — Official ID3v2.2 specification (obsolete but still encountered)
- ID3v2.4 frame IDs: `https://id3.org/id3v2.4.0-frames` — Complete frame ID listing for v2.4
- ID3v2.3 frame IDs: `https://id3.org/id3v2.3.0-frames` — Complete frame ID listing for v2.3

**Unofficial Extensions:**
- ID3v1.1 specification: widely documented on id3.org and community wikis
- WinAmp extended genres: documented on various community resources
- iTunes comment tags (iTunSMPB, iTunNORM): reverse-engineered from iTunes behavior
- LAME INFO tag format: documented in LAME source code and community wiki

**Reference Implementations for Study:**
- TagLib source: `https://github.com/taglib/taglib` — Read `taglib/mpeg/id3v2/` directory
- Mutagen source: `https://github.com/quodlibet/mutagen` — Read `mutagen/mp3.py` and `mutagen/_id3util.py`
- LAME source: `https://lame.sourceforge.io/` — `TAGS` in `lame/src/`
- FFmpeg source: `https://github.com/FFmpeg/FFmpeg` — `libavformat/id3v2.c`

**Community Resources:**
- Hydrogenaudio Knowledgebase: `https://wiki.hydrogenaudio.org/` — Tagging section covers edge cases and best practices
- MusicBrainz style guidelines: `https://musicbrainz.org/doc/style/` — Detailed conventions for tag values
- WinAmp genre list: Various community-maintained lists of extended genre indices
- What is Podcast tagging? Specification for podcast-specific frames on id3.org

**RFC and Standards References:**
- ISO 639-2: Codes for the representation of names of languages — used for language codes in COMM/USLT frames
- Unicode Standard Annex #14: Line Breaking Properties — relevant for UTF-16 string handling
- IEEE CRC-32: Polynomial 0x04C11DB7 — used in ID3v2.3 extended header CRC

---

## 13. IMPLEMENTATION CHECKLIST (for the Converter Developer)

For developers building MP3 conversion tools that handle ID3 metadata:

**ID3v2.3 and v2.4 Tag Reading:**
- [ ] Detect ID3v2 tag by checking first 3 bytes for "ID3" magic
- [ ] Parse version byte (0x03 = v2.3, 0x04 = v2.4, 0x02 = v2.2)
- [ ] Decode size field: big-endian uint32 (v2.3) or syncsafe uint32 (v2.4)
- [ ] Check unsynchronisation flag and de-unsynchronise entire tag payload if set (v2.3)
- [ ] Check for extended header (flag bit 6) and skip if present
- [ ] Iterate through frames: read 10-byte header, then size bytes of data
- [ ] Stop at padding (`\x00\x00\x00\x00` frame ID) or when accumulated offset reaches tag size
- [ ] Handle frame-level unsynchronisation (v2.4 format flag bit 2)
- [ ] Handle frame compression (decompress zlib if flagged)
- [ ] Handle frame grouping (remove leading group byte if flagged)
- [ ] Parse frame data based on frame ID (T*, W*, APIC, COMM, etc.)
- [ ] Read text encoding byte and decode strings accordingly
- [ ] Handle UTF-16 BOM (FF FE = LE, FE FF = BE)
- [ ] Read APIC MIME type and binary image data correctly
- [ ] Read COMM language code and description/text correctly
- [ ] Read TXXX user-defined frames by description field
- [ ] Store all parsed frames in an ordered list

**ID3v2 Writing:**
- [ ] Determine target version (prefer v2.3 for compatibility, v2.4 for UTF-8)
- [ ] Encode each field with appropriate text encoding byte ($03 UTF-8 preferred)
- [ ] Pack frame headers: Frame ID + size (big-endian v2.3, syncsafe v2.4) + flags
- [ ] Check for $FF $00 patterns in frame data and apply unsynchronisation if needed
- [ ] Set header unsynchronisation flag if any data was unsynchronised
- [ ] Calculate total frame data size and append padding (minimum 128 bytes)
- [ ] Write header (10 bytes): "ID3" + version + revision + flags + size
- [ ] Write all frames in order
- [ ] Write padding bytes
- [ ] Prepend entire tag to audio data (not overwrite — preserves audio)

**ID3v1 Reading:**
- [ ] Seek to `file_size - 128` bytes from EOF
- [ ] Verify bytes at that position are "TAG" (0x54 0x41 0x47)
- [ ] Read title (30 bytes, strip trailing nulls/spaces)
- [ ] Read artist (30 bytes)
- [ ] Read album (30 bytes)
- [ ] Read year (4 bytes)
- [ ] Read comment (30 bytes): check byte 97 for 0x00 to detect v1.1 format
- [ ] If v1.1: byte 98 = track number
- [ ] Read genre byte as index into genre list
- [ ] Treat empty title/artist/album as indication of cleared/missing tag

**ID3v1 Writing:**
- [ ] Seek to EOF minus 128 bytes
- [ ] Check for existing "TAG" — overwrite if present, otherwise append
- [ ] Write "TAG" identifier
- [ ] Write title (30 bytes, space-padded or null-padded)
- [ ] Write artist (30 bytes)
- [ ] Write album (30 bytes)
- [ ] Write year (4 ASCII digits)
- [ ] Write comment (30 bytes): for v1.1, write null + track byte at bytes 97-98
- [ ] Write genre index byte
- [ ] Write exactly 128 bytes total

**Metadata Preservation (Round-trip):**
- [ ] Copy all ID3v2 frames from source to destination unchanged
- [ ] Copy ID3v1 fields if source had them and destination format supports them
- [ ] Preserve APIC binary image data exactly (do not re-encode)
- [ ] Preserve COMM frames with exact language, description, and text
- [ ] Preserve TXXX frames by description field
- [ ] Preserve PRIV and UFID frames
- [ ] Update TLEN if audio length changes after re-encoding
- [ ] Update TDRC/TYER if date changes
- [ ] Strip iTunes-specific frames when converting to non-iTunes formats
- [ ] Convert ReplayGain TXXX frames to the equivalent format for the destination codec

**Verification:**
- [ ] After writing, read back the tag with a different tool (e.g., ffprobe + kid3-cli)
- [ ] Verify all frame IDs, sizes, and values match the source
- [ ] Verify no Invalid UTF-8 errors in kid3 output
- [ ] Verify no duplicate atom warnings
- [ ] Verify cover art binary matches source (byte-for-byte)
- [ ] Verify sort fields appear as their native atoms, not freeform text
- [ ] Verify numeric IDs (Artist ID, Album ID) preserved if present in source
