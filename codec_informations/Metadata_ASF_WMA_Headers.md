# Metadata: ASF/WMA Headers — Deep Technical Reference
> **Category:** Metadata
> **File Extensions:** `.wma`, `.wmv`, `.asf`
> **MIME Types:** `audio/x-wma`, `video/x-ms-asf`, `application/vnd.ms-asf`
> **Standardization Body:** Microsoft
> **Primary Specification:** [ASF Specification](https://exse.eyewated.com/fls/54b3ed95bbfb1a92.pdf), Microsoft Windows Media Format SDK
> **Patent Status:** Proprietary — Microsoft holds patents on ASF/WMA
> **License:** Proprietary — requires licensing for encoder implementation; decoders widely supported
> **Current Version:** WMA 10 Pro (profile-based), WMA Lossless v1
> **Active Development:** Limited — Microsoft shifted focus to AAC and newer codecs

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Microsoft Corporation
- **Year Created:** 1999 (Windows Media 7)
- **Original Purpose:** To create a streaming-optimized container and codec format for internet media delivery, competing with RealAudio and early MP3 streaming solutions. ASF (Advanced Systems Format) was designed specifically for network streaming with built-in metadata support.
- **Problem with Predecessors:** Early MP3 files lacked built-in metadata for streaming; RealAudio was platform-locked; AVI was not optimized for streaming. ASF solved these issues with a streaming-first design.

### 1.2 Version History
|| Version | Year | Key Changes |
|---------|------|-------------|
| ASF 1.0 | 1999 | Initial specification, Windows Media 7 |
| ASF 2.0 | 2000 | Windows Media 8, additional codecs |
| ASF 3.0 | 2003 | Windows Media 9 Series, WMA Pro, WMA Lossless |
| WMA 10 | 2005+ | Enhanced profiles, 24-bit audio, 96 kHz |
| Current | 2010s | Maintenance only, AAC competition |

### 1.3 Current Adoption
- **Primary use cases:** Windows Media distribution, legacy streaming servers, some DRM-protected content
- **Platforms with native support:** Windows (native), macOS (via third-party), Linux (via FFmpeg), iOS (no native support), Android (limited)
- **Major services using WMA:** Declining — most services switched to AAC/MP3/HLS/DASH
- **Hardware support:** Older Windows Media Center PCs, some DLNA devices, legacy gaming consoles
- **Status:** Legacy format, declining adoption; AAC and Opus dominate streaming; FLAC for lossless

---

## 2. ASF FILE STRUCTURE

### 2.1 High-Level Object Architecture
ASF is an object-based container. Every element is an object:

```
ASF File Structure:
┌─────────────────────────────────────────────────────┐
│ Header Object (mandatory, first)                    │
│  ├── File Properties Object                        │
│  ├── Stream Properties Object (per stream)         │
│  ├── Content Description Object                    │
│  ├── Extended Content Description Object           │
│  ├── Stream Bitrate Properties Object              │
│  └── Header Extension Object (optional)            │
│       └── Additional metadata objects              │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│ Data Object (mandatory, contains media packets)    │
└─────────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────────┐
│ Index Object (optional, for seeking)               │
└─────────────────────────────────────────────────────┘
```

### 2.2 Object Header Layout
Every ASF object has a 24-byte header:

```
Offset   Size   Field              Type      Description
-------  -----  -----------------  --------  -----------------------
0x0000   16     Object GUID        GUID      Unique identifier for object type
0x0010   8      Object Size        QWORD     Size of entire object in bytes
0x0018   —      Object Payload     varies    Object-specific data
```

### 2.3 GUID Reference Table
| GUID Name | Value | Object Type |
|-----------|-------|-------------|
| ASF_Header_Object | 75B22630-668E-11CF-A6D9-00AA0062CE6C | Header Object |
| ASF_Data_Object | 75B22636-668E-11CF-A6D9-00AA0062CE6C | Data Object |
| ASF_Simple_Index_Object | 33000890-E5B1-11CF-89F4-00A0C90349CB | Index Object |
| ASF_Index_Object | D6E229DF-35DA-11D1-9034-00A0C90349CB | Index Object (alternate) |
| ASF_File_Properties_Object | 8CABDCA1-A947-11CF-8EE4-00C00C205365 | File Properties |
| ASF_Stream_Properties_Object | B7DC0791-A9B7-11CF-8EE6-00C00C205365 | Stream Properties |
| ASF_Header_Extension_Object | 5FBBC03F-8EDB-11CF-A5D6-28DB04C10000 | Header Extension |
| ASF_Content_Description_Object | 75B22633-668E-11CF-A6D9-00AA0062CE6C | Content Description |
| ASF_Extended_Content_Description_Object | D2D0A440-E307-11D2-97F0-00A0C95EA850 | Extended Content |
| ASF_Codec_List_Object | 86D15240-311D-11D0-A3A4-00A0C90348F6 | Codec List |
| ASF_Error_Correction_Object | 75B22635-668E-11CF-A6D9-00AA0062CE6C | Error Correction |

---

## 3. CONTENT DESCRIPTION OBJECT

### 3.1 Overview
The Content Description Object stores bibliographic information. It is part of the Header Object.

### 3.2 Binary Layout
```
Offset   Size   Field                    Type       Description
-------  -----  ----------------------  ---------  -------------------------
0x0000   16     Object GUID              GUID       ASF_Content_Description_Object
0x0010   8      Object Size             QWORD      Total object size
0x0018   2      Title Length            WORD       Length in bytes (UTF-16LE)
0x001A   2      Author Length           WORD       Length in bytes (UTF-16LE)
0x001C   2      Copyright Length        WORD       Length in bytes (UTF-16LE)
0x001E   2      Description Length      WORD       Length in bytes (UTF-16LE)
0x0020   2      Rating Length           WORD       Length in bytes (UTF-16LE)
0x0022   N      Title                  WCHARS     UTF-16LE string
         N      Author                 WCHARS     UTF-16LE string
         N      Copyright              WCHARS     UTF-16LE string
         N      Description            WCHARS     UTF-16LE string
         N      Rating                 WCHARS     UTF-16LE string
```

### 3.3 Standard Fields
|| Field | Max Length (bytes) | Encoding | Description |
|-------|-------------------|----------|-------------|
| Title | 65,535 × 2 | UTF-16LE | Track title |
| Author | 65,535 × 2 | UTF-16LE | Artist/creator name |
| Copyright | 65,535 × 2 | UTF-16LE | Copyright notice |
| Description | 65,535 × 2 | UTF-16LE | Track description |
| Rating | 65,535 × 2 | UTF-16LE | Content rating |

### 3.4 UTF-16LE Encoding
ASF uses UTF-16LE (little-endian) for all string fields:

```c
// Example: Encoding "Hello" in ASF format:
uint16_t title[] = {
    'H', 'e', 'l', 'l', 'o',  // ASCII characters
    0x0000  // Null terminator
};
// Each character is 2 bytes (16 bits)
// For ASCII: character is just 0x00 followed by ASCII byte
// Example: 'H' = 0x0048 = byte[0] = 0x48, byte[1] = 0x00
```

---

## 4. EXTENDED CONTENT DESCRIPTION OBJECT

### 4.1 Overview
The Extended Content Description Object stores arbitrary name-value pairs. This is where the WM/ namespace metadata lives.

### 4.2 Binary Layout
```
Offset   Size   Field                    Type       Description
-------  -----  ----------------------  ---------  -------------------------
0x0000   16     Object GUID              GUID       ASF_Extended_Content...
0x0010   8      Object Size             QWORD      Total object size
0x0018   2      Descriptor Count        WORD       Number of descriptors

Then for each descriptor:
  ├── Descriptor Name Length      WORD
  ├── Descriptor Name            WCHARS (UTF-16LE, null-terminated)
  ├── Descriptor Value Data Type WORD
  ├── Descriptor Value Length    WORD
  └── Descriptor Value           Varies (see data types below)
```

### 4.3 Descriptor Value Data Types
|| Type Value | Type Name | Description | Value Format |
|------------|-----------|-------------|-------------|
| 0x0000 | Unicode string type | Text value | UTF-16LE null-terminated string |
| 0x0001 | BYTE array type | Binary data | Raw bytes (e.g., WM/Picture) |
| 0x0002 | BOOL type | Boolean | 2 bytes: 0x0000 (false) or 0x0001 (true) |
| 0x0003 | DWORD type | 32-bit unsigned | 4 bytes, little-endian |
| 0x0004 | QWORD type | 64-bit unsigned | 8 bytes, little-endian |
| 0x0005 | WORD type | 16-bit unsigned | 2 bytes, little-endian |
| 0x0006 | GUID type | GUID | 16 bytes |

### 4.4 Common WM/ Namespace Tags
|| Tag Name | Data Type | Description | Example Value |
|----------|-----------|------------|---------------|
| WM/Title | String | Track title (mirrors Content Description) | "My Song" |
| WM/Author | String | Artist name | "Artist Name" |
| WM/TrackNumber | String | Track number | "5" or "5/12" |
| WM/AlbumTitle | String | Album name | "My Album" |
| WM/AlbumArtist | String | Album artist | "Album Artist" |
| WM/Genre | String | Genre | "Rock" |
| WM/Year | String | Release year | "2024" |
| WM/Comment | String | Comment | "Recorded in 2024" |
| WM/Composer | String | Composer | "Songwriter" |
| WM/Publisher | String | Label/publisher | "Record Label" |
| WM/CatalogNumber | String | Catalog number | "CAT001" |
| WM/Barcode | String | UPC/EAN barcode | "012345678901" |
| WM/Conductor | String | Conductor | "Maestro" |
| WM/Writer | String | Lyricist/writer | "Writer Name" |
| WM/OriginalAlbumTitle | String | Original album | "Original Name" |
| WM/OriginalArtist | String | Original artist | "Original Artist" |
| WM/SubTitle | String | Subtitle | "Extended Mix" |
| WM/ContentGroupDescription | String | Grouping | "Side A" |
| WM/PartOfSet | String | Disc number | "1/2" |
| WM/ISRC | String | ISRC code | "USRC12345678" |
| WM/Mood | String | Mood tag | "Energetic" |
| WM/Broadcast | String | Broadcast date | "2024-01-01" |
| WM/Language | String | Language code | "en" |
| WM/MediaClassPrimaryID | GUID | Media class | GUID |
| WM/MediaClassSecondaryID | GUID | Media class | GUID |
| WM/ParentalRating | String | Parental rating | "PG-13" |

### 4.5 Numeric IDs in WM/ Namespace
|| Tag Name | Data Type | Description |
|----------|-----------|-------------|
| WM/Track | DWORD | Track number (numeric) |
| WM/TrackRank | DWORD | Track ranking |
| WM/PartOfSet | DWORD | Disc number (numeric) |
| WM/BeatsPerMinute | DWORD | BPM |
| WM/ArtistID | DWORD | Artist ID (MusicBrainz) |
| WM/AlbumID | DWORD | Album ID (MusicBrainz) |
| WM/ContentDistributorID | DWORD | Distributor ID |

### 4.6 ReplayGain Tags in WM/ Namespace
|| Tag Name | Data Type | Description | Example |
|----------|-----------|------------|---------|
| WM/PeakValue | DWORD | Peak amplitude | 2147483647 (= 1.0 in Q14.17) |
| WM/AverageLevel | DWORD | Average RMS level | 1073741824 (= 0.5) |

**Note:** WM/PeakValue uses Q14.17 fixed-point representation:
```
Peak_dBFS = 20 × log10(PeakValue / 2147483648)  [dBFS]
Example: 2147483648 = 0 dBFS
```

---

## 5. WM/PICTURE — ALBUM ART

### 5.1 Binary Layout
WM/Picture is stored as a binary (BYTE array) descriptor in the Extended Content Description Object:

```
Offset   Size   Field              Type     Description
-------  -----  -----------------  -------  -------------------------------
0x0000   1      Picture Type       BYTE     0–20 (see ID3 APIC types)
0x0001   4      Image Data Size    DWORD    Size of image data in bytes
0x0005   N      MIME Type          WCHARS   UTF-16LE null-terminated string
         N      Description         WCHARS   UTF-16LE null-terminated string
         M      Image Data         BYTES    Raw JPEG/PNG data
```

### 5.2 Picture Type Values
|| Value | Name | Description |
|-------|------|-------------|
| 0 | Other | Other image |
| 1 | File icon | 32×32 PNG file icon |
| 2 | Other file icon | Other file icon |
| 3 | Front cover | Front album cover |
| 4 | Back cover | Back album cover |
| 5 | Leaflet | Media leaflet page |
| 6 | Media | Label side of CD |
| 7 | Lead artist | Lead performer/soloist |
| 8 | Artist | Artist/performer |
| 9 | Conductor | Conductor |
| 10 | Orchestra | Band/orchestra |
| 11 | Composer | Composer |
| 12 | Lyricist | Lyricist/writer |
| 13 | Recording location | Recording studio/location |
| 14 | During recording | During recording session |
| 15 | During performance | During performance |
| 16 | Movie/video screen | Screen capture |
| 17 | Bright colored fish | Bright colored fish |
| 18 | Illustration | Illustration |
| 19 | Band logo | Band/artist logo |
| 20 | Publisher logo | Publisher/studio logo |

### 5.3 MIME Types
|| MIME Type | Description |
|-----------|-------------|
| image/jpeg | JPEG image (most common) |
| image/png | PNG image |
| image/bmp | BMP image |
| image/gif | GIF image |

### 5.4 WM/Picture Example
```
// Example WM/Picture binary data (hex representation):
// Picture Type: 0x03 (front cover)
// Image Data Size: 0x0000A123 (41,123 bytes)
// MIME Type: "image/jpeg\0"
// Description: "Front cover\0"
// Image Data: [41,123 bytes of JPEG]

// C structure interpretation:
struct WM_PICTURE {
    uint8_t  picture_type;          // 1 byte: 0-20
    uint32_t image_data_size;        // 4 bytes: little-endian
    // Then UTF-16LE strings...
    wchar_t  mime_type[];            // null-terminated UTF-16LE
    wchar_t  description[];          // null-terminated UTF-16LE
    uint8_t  image_data[];          // binary image
};
```

### 5.5 Multiple Pictures
WM/Picture can appear multiple times with different types. The convention:
- Type 3 = Front cover (primary, most important)
- Type 4 = Back cover
- Others as needed

---

## 6. ASF METADATA READING & WRITING

### 6.1 FFmpeg Metadata Reading
```bash
# Read all metadata from WMA file:
ffprobe -v quiet -print_format json -show_format -show_streams input.wma

# Sample output structure:
# {
#   "format": {
#     "tags": {
#       "title": "Track Title",
#       "artist": "Artist Name",
#       "album": "Album Name",
#       "WM/TrackNumber": "5",
#       "WM/AlbumArtist": "Album Artist",
#       "WM/Genre": "Rock"
#     }
#   }
# }

# Read specific tag:
ffprobe -v quiet -show_entries format_tags=title,artist -of default=noprint_wrappers=1 input.wma
```

### 6.2 FFmpeg Metadata Writing
```bash
# Write metadata to WMA file:
ffmpeg -i input.wav \
  -c:a wmav2 -b:a 192k \
  -metadata title="Song Title" \
  -metadata artist="Artist Name" \
  -metadata album="Album Name" \
  -metadata year="2024" \
  -metadata track="5" \
  -metadata genre="Rock" \
  -metadata WM/AlbumArtist="Album Artist" \
  -metadata WM/TrackNumber="5" \
  -metadata WM/Year="2024" \
  output.wma

# Copy metadata from source:
ffmpeg -i input.wma -i cover.jpg \
  -c copy \
  -metadata:s:v title="Front cover" \
  -metadata:s:v comment="Cover (front)" \
  output_with_cover.wma
```

### 6.3 Embedding Cover Art with FFmpeg
```bash
# Method 1: Using metadata (WM/Picture):
ffmpeg -i input.wma -i cover.jpg \
  -c copy \
  -attach "cover.jpg" -metadata:s:2 title="Front cover" \
  output.wma

# Method 2: Using the WM/Picture tag:
# Note: FFmpeg doesn't directly support WM/Picture embedding
# Use a tool like ffmpeg-normalize or matroska for advanced tags

# Method 3: Use atomicparsley or similar tool:
# atomicparsley input.wma --artwork cover.jpg
```

### 6.4 Safe Metadata Writing
```bash
# FFmpeg metadata behavior:
# - By default, metadata is written directly to the file
# - For WMA, this rewrites the entire file (safe)
# - Some formats require rebuilding (FFV1, some audio)

# Unsafe vs safe writes:
# - Unsafe: Formats that require frame re-encoding
# - Safe: WMA, MP3 (most), FLAC (most)
```

---

## 7. WM/ NAMESPACE COMPLETE REFERENCE

### 7.1 Track/Identification Tags
|| Tag | Data Type | Description | Example |
|-----|-----------|------------|---------|
| WM/Title | String | Track title | "Song Title" |
| WM/TrackNumber | String | Track number | "5" or "5/12" |
| WM/PartOfSet | String | Disc number | "1/2" |
| WM/UniqueFileIdentifier | String | UFID | "http://musicbrainz.org/track/..." |
| WM/MusicBrainzTrackID | String | MB track ID | UUID |
| WM/MusicBrainzAlbumID | String | MB album ID | UUID |
| WM/MusicBrainzArtistID | String | MB artist ID | Comma-separated UUIDs |
| WM/MusicBrainzAlbumArtistID | String | MB album artist ID | UUID |
| WM/MusicBrainzReleaseGroupID | String | MB release group ID | UUID |
| WM/MusicBrainzTrackDuration | DWORD | Duration (ms) | 234000 |
| WM/ISRC | String | ISRC code | "USRC12345678" |
| WM/CatalogNumber | String | Catalog number | "ABC123" |
| WM/Barcode | String | UPC/EAN | "012345678901" |

### 7.2 Artist/Personnel Tags
|| Tag | Data Type | Description |
|-----|-----------|------------|
| WM/Author | String | Primary artist |
| WM/AlbumArtist | String | Album artist |
| WM/Composer | String | Composer |
| WM/Writer | String | Lyricist/writer |
| WM/Conductor | String | Conductor |
| WM/Orchestra | String | Orchestra/band |
| WM/ModifiedBy | String | Remixer |
| WM/OriginalArtist | String | Original artist |
| WM/Producer | String | Producer |
| WM/Engineer | String | Engineer |

### 7.3 Album/Release Tags
|| Tag | Data Type | Description |
|-----|-----------|------------|
| WM/AlbumTitle | String | Album name |
| WM/OriginalAlbumTitle | String | Original album name |
| WM/AlbumDuration | DWORD | Album duration (ms) |
| WM/SetPart | String | Part of compilation |
| WM/ContentGroupDescription | String | Grouping |

### 7.4 Genre/Mood Tags
|| Tag | Data Type | Description |
|-----|-----------|------------|
| WM/Genre | String | Genre (ID3v1 style) |
| WM/GenreID | String | Genre ID (ID3v1 number) |
| WM/Mood | String | Mood descriptor |

### 7.5 Temporal/Date Tags
|| Tag | Data Type | Description | Example |
|-----|-----------|------------|---------|---------|
| WM/Year | String | Year | "2024" |
| WM/OriginalReleaseTime | String | Original release date | "1975" |
| WM/RecordingTime | String | Recording date | "2024-01" |
| WM/ReleaseDate | String | Release date | "2024-06-15" |
| WM/BroadcastDate | String | Broadcast date | "2024-06-15T12:00:00Z" |
| WM/EncodedBy | String | Encoded by | "Encoder Name" |

### 7.6 Technical Tags
|| Tag | Data Type | Description |
|-----|-----------|------------|
| WM/EncodingSettings | String | Encoder settings |
| WM/EncodingTime | QWORD | Encoding timestamp |
| WM/Duration | QWORD | Duration (100-nanosecond units) |
| WM/EndTime | QWORD | End time |
| WM/StartTime | QWORD | Start time |
| WM/InitialKey | String | Musical key |
| WM/BeatsPerMinute | DWORD | BPM |
| WM/MediaType | String | Media type |

### 7.7 Commercial/Legal Tags
|| Tag | Data Type | Description |
|-----|-----------|------------|
| WM/Copyright | String | Copyright notice |
| WM/CommercialInfo | String | Commercial info URL |
| WM/Custom1-10 | String | Custom fields |
| WM/ParentalRating | String | Parental rating |
| WM/PersonalLicense | String | Personal license URL |

### 7.8 ReplayGain/Loudness Tags
|| Tag | Data Type | Description | Format |
|-----|-----------|------------|---------|
| WM/PeakValue | DWORD | Peak amplitude | Q14.17 fixed-point |
| WM/AverageLevel | DWORD | Average RMS | Q14.17 fixed-point |
| WM/Loudness | String | Integrated loudness | dB value |
| WM/LoudnessSpan | String | Loudness range | dB value |
| WM/PeakLevel | String | Peak level | dB value |

**ReplayGain conversion:**
```python
# WM/PeakValue to dBFS:
peak_value = 2147483648  # Example value
peak_linear = peak_value / 2147483648.0
peak_db = 20 * log10(peak_linear)
# Example: 0.0 dBFS

# dBFS to WM/PeakValue:
db = -6.0  # Example: -6 dBFS
linear = 10 ** (db / 20.0)
peak_value = int(linear * 2147483648.0)
```

---

## 8. ASF STREAM PROPERTIES FOR AUDIO

### 8.1 Audio Stream Properties Object
```
Offset   Size   Field                    Type       Description
-------  -----  ----------------------  ---------  -------------------------
0x0000   16     Object GUID              GUID       ASF_Stream_Properties_Object
0x0010   8      Object Size             QWORD      Total object size
0x0018   16     Stream Type GUID         GUID       Audio stream: 4F69... or Video: 4F69...
0x0028   16     Error Correction Type     GUID       Error correction GUID
0x0038   8      Time Offset              QWORD      Time offset (100-ns units)
0x0040   4      Type-Specific Data Len   DWORD      Length of audio/video data
0x0044   4      Error Correction Data Len DWORD      Length of error correction data
0x0048   2      Flags                    WORD       Stream number (lower 10 bits)
0x004A   4      Reserved                 DWORD      Reserved
0x004E   N      Type-Specific Data       BYTES      Audio/video specific data
         M      Error Correction Data     BYTES      Error correction data
```

### 8.2 WAVEFORMATEX for Audio Stream
The audio Type-Specific Data contains a WAVEFORMATEX structure:

```
Offset   Size   Field                    Type       Description
-------  -----  ----------------------  ---------  -------------------------
0x0000   2      wFormatTag               WORD       Codec ID (0x0161 = WMA, 0x0163 = WMA Pro)
0x0002   2      nChannels                WORD       Channel count
0x0004   4      nSamplesPerSec           DWORD      Sample rate
0x0008   4      nAvgBytesPerSec          DWORD      Average bitrate (bytes/s)
0x000C   2      nBlockAlign              WORD       Block alignment
0x000E   2      wBitsPerSample           WORD       Bits per sample
0x0010   2      cbSize                   WORD       Extra data size
0x0012   N      Extra data               BYTES      Codec-specific extra data
```

### 8.3 WMA Codec IDs
| Codec ID | Codec Name | Channels | Sample Rates | Bit Depths |
|----------|-----------|----------|-------------|-----------|
| 0x0161 | WMA 7/8 | 1-2 | 8–48 kHz | 16-bit |
| 0x0162 | WMA Lossless | 1-2 | 44–96 kHz | 16–24-bit |
| 0x0163 | WMA Pro | Up to 8 | 8–96 kHz | 16–24-bit |
| 0x0164 | WMA Voice | 1 | 8–22 kHz | 16-bit |

---

## 9. HEADER EXTENSION OBJECT

### 9.1 Overview
The Header Extension Object contains additional metadata and features beyond the core header.

### 9.2 Binary Layout
```
Offset   Size   Field                    Type       Description
-------  -----  ----------------------  ---------  -------------------------
0x0000   16     Object GUID              GUID       ASF_Header_Extension_Object
0x0010   8      Object Size             QWORD      Total object size
0x0018   16     Reserved Field 1         GUID       Must be ASF_Header_Extension
0x0024   4      Reserved Field 2         DWORD      Must be 0x00000000
0x0028   4      Extension Data Size      DWORD      Size of extension data
0x002C   N      Extension Objects        varies     Child objects
```

### 9.3 Extended Metadata Objects
Inside Header Extension:
| Object | Description |
|--------|-------------|
| Metadata Object | 1-byte language index, stream number |
| Metadata Library Object | Multiple metadata fields in one object |
| Padding Object | Reserved for future use |

---

## 10. FFmpeg IMPLEMENTATION REFERENCE

### 10.1 FFmpeg Codec and Format Identifiers
```
Codecs (libavcodec):
  - wmav1    : Windows Media Audio 1 (WMA 7)
  - wmav2    : Windows Media Audio 2 (WMA 8)
  - wmalossless: Windows Media Audio Lossless
  - wmapro   : Windows Media Audio Pro
  - wmavoice : Windows Media Audio Voice

Format (libavformat):
  - asf      : Advanced Systems Format
  - wma      : Alias for asf

Demuxer:
  ffprobe -v quiet -show_format input.wma
  # Shows: format_name=asf, format_long_name=ASF (Advanced / Advanced Systems Format)

Muxer:
  ffmpeg -i input.wav -c:a wmav2 output.wma
```

### 10.2 FFmpeg Encoding Options
```bash
# WMA v2 (standard quality):
ffmpeg -i input.wav -c:a wmav2 -b:a 192k output.wma

# WMA Lossless:
ffmpeg -i input.wav -c:a wmalossless output.wma

# WMA Pro (high quality, multichannel):
ffmpeg -i input.wav -c:a wmapro -b:a 384k output.wma

# Specific bitrate:
ffmpeg -i input.wav -c:a wmav2 -b:a 128k -ar 44100 -ac 2 output.wma

# Metadata with encoding:
ffmpeg -i input.wav \
  -c:a wmav2 -b:a 256k \
  -metadata title="Track Title" \
  -metadata artist="Artist" \
  -metadata album="Album" \
  output.wma
```

### 10.3 FFmpeg Decoding
```bash
# Decode to WAV:
ffmpeg -i input.wma -c:a pcm_s16le output.wav

# Decode to FLAC (transcode):
ffmpeg -i input.wma -c:a flac -compression_level 8 output.flac

# Extract audio without transcoding (copy):
ffmpeg -i input.wma -c:a copy output.asf

# Specify audio stream:
ffmpeg -i input.asf -map 0:a:0 -c:a copy output.wma
```

### 10.4 FFmpeg Metadata Handling
```bash
# Show all metadata:
ffprobe -v quiet -show_entries format_tags -of json input.wma

# Modify metadata:
ffmpeg -i input.wma \
  -c copy \
  -metadata title="New Title" \
  -metadata author="New Artist" \
  output.wma

# Remove specific tag:
ffmpeg -i input.wma -c copy -metadata title= output.wma

# Copy metadata from one file to another:
ffmpeg -i output.wma -i input.wma -c copy -map_metadata 1 output.wma
```

---

## 11. WMDRM METADATA

### 11.1 WMDRM Overview
Windows Media DRM (WMDRM) was Microsoft's digital rights management system for WMA files. It is deprecated and replaced by PlayReady.

### 11.2 WMDRM Metadata (Proprietary)
```
Note: WMDRM-protected content stores DRM metadata including:
  - Content ID
  - License acquisition URL
  - Expiration date
  - Playback restrictions

This is proprietary and not covered in detail here.
FFmpeg cannot encode/decode WMDRM-protected content.
```

### 11.3 PlayReady (Successor)
Modern Microsoft DRM uses PlayReady. Not covered in this document.

---

## 12. COMPATIBILITY NOTES

### 12.1 Cross-Platform Compatibility
| Platform | Read Support | Write Support | Notes |
|----------|-------------|---------------|-------|
| Windows | Native | Native | Full support |
| macOS | Third-party | Third-party | Use afconvert or XLD |
| Linux | FFmpeg | FFmpeg | Full via FFmpeg |
| iOS | No | No | Convert before transfer |
| Android | FFmpeg | FFmpeg | Limited |

### 12.2 Tag Compatibility Matrix
| Tag System | Read | Write | Notes |
|------------|------|-------|-------|
| ASF Native | ✓ | ✓ | Primary system |
| ID3v1 | ✗ | ✗ | Not supported |
| ID3v2 | ✗ | ✗ | Not supported |
| Vorbis Comments | ✗ | ✗ | Not supported |
| MP4 Atoms | ✗ | ✗ | Not supported |

### 12.3 Known Limitations
- **WM/Picture:** FFmpeg can read but limited write support
- **Custom tags:** Limited to ASF Extended Content
- **Unicode:** Full UTF-16LE support
- **Album art:** JPEG/PNG supported

---

## 13. REFERENCE IMPLEMENTATIONS & TOOLS

### 13.1 Tools Reference
| Tool | Platform | Read WM | Write WM | Notes |
|------|----------|---------|----------|-------|
| FFmpeg | Cross-platform | ✓ | ✓ | Primary tool |
| foobar2000 | Windows | ✓ | ✓ | Full tag editor |
| TagLib | Cross-platform | ✓ | ✓ | Library |
| JAudiotagger | Java | ✓ | ✓ | Library |
| Kid3 | Cross-platform | ✓ | ✓ | Qt-based |

### 13.2 Library Reference
```c
// TagLib ASF support:
// TagLib reads and writes ASF/WMA metadata

// Example with TagLib (C++):
#include <taglib/wma/wmatag.h>
#include <taglib/wma/wmaproperties.h>

TagLib::ASF::File file("input.wma");
file.tag()->setTitle("New Title");
file.tag()->setArtist("New Artist");
file.save();

// Cover art:
TagLib::ASF::Picture *pic = new TagLib::ASF::Picture();
pic->setPicture(TagLib::ByteVector(imageData, size));
pic->setMimeType("image/jpeg");
pic->setType(TagLib::ASF::Picture::FrontCover);
file.tag()->addPicture(pic);
file.save();
```

---

## 14. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specifications
- **ASF Specification:** https://exse.eyewated.com/fls/54b3ed95bbfb1a92.pdf
- **WAVE FORMAT EXTENSIBLE:** Microsoft documentation
- **WM/Picture Structure:** Microsoft Windows Media Format SDK

### Technical Resources
- FFmpeg ASF Demuxer: https://ffmpeg.org/ffmpeg-formats.html#asf
- Hydrogenaudio WMA: https://wiki.hydrogenaud.io/index.php?title=WMA
- TagLib ASF: https://taglib.org/

---

## 15. IMPLEMENTATION CHECKLIST

### Metadata Reading
- [ ] Parse ASF Header Object
- [ ] Read Content Description Object (title, author, copyright, description, rating)
- [ ] Read Extended Content Description Object (WM/ namespace tags)
- [ ] Extract WM/Picture binary data for album art
- [ ] Handle UTF-16LE encoding correctly
- [ ] Handle multiple WM/Picture entries (front cover, back cover, etc.)

### Metadata Writing
- [ ] Write Content Description Object
- [ ] Write Extended Content Description Object
- [ ] Write WM/ namespace tags with correct data types
- [ ] Write WM/Picture with correct binary structure
- [ ] Validate string encoding (UTF-16LE)
- [ ] Handle tag removal (write empty string or omit)

### Format Conversion
- [ ] Preserve WM/ tags when converting ASF → ASF
- [ ] Map tags to target format (FLAC Vorbis Comments, MP4 Atoms, etc.)
- [ ] Handle cover art conversion (WM/Picture → APIC, etc.)
- [ ] Verify lossless round-trip where applicable

### Quality Verification
- [ ] Verify WM/ tags appear in ffprobe output
- [ ] Test WM/Picture extraction with cover art viewers
- [ ] Test special characters (Unicode) in tags
- [ ] Test empty tags handling

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*

---

## 16. ASF STREAM TYPES

### 16.1 Stream Type GUIDs
| GUID Name | Value | Stream Type |
|-----------|-------|-------------|
| ASF_Audio_Media | 56406F64-0000-0010-8000-00AA00389B71 | Audio stream |
| ASF_Video_Media | 52446D73-0000-0010-8000-00AA00389B71 | Video stream |
| ASF_Command_Media | 59FCAC05-0000-0010-8000-00AA00389B71 | Command stream |
| ASF_JFIF_Media | 24DC6D16-0000-0010-8000-00AA00389B71 | JPEG stream |
| ASF_Degradable_JPEG_Media | 3AFBCC2A-0000-0010-8000-00AA00389B71 | Degradable JPEG |
| ASF_No_Error_Concealment | B52F3B00-0000-0010-8000-00AA00389B71 | Audio (no concealment) |
| ASF_Spreadable_Stream_Group | 3CBFBAA0-0000-0010-8000-00AA00389B71 | Stream group |

### 16.2 Audio Stream Properties
The audio-specific portion of the Stream Properties Object:

```python
# WAVEFORMATEX structure for audio:
WAVEFORMATEX = {
    'wFormatTag': 'uint16',     # Codec ID (0x0161=WMA7, 0x0162=WMA Lossless, 0x0163=WMA Pro)
    'nChannels': 'uint16',       # Number of channels
    'nSamplesPerSec': 'uint32',  # Sample rate in Hz
    'nAvgBytesPerSec': 'uint32',# Average bytes per second (bitrate/8)
    'nBlockAlign': 'uint16',     # Block alignment
    'wBitsPerSample': 'uint16',   # Bits per sample
    'cbSize': 'uint16',          # Size of extra data
}
```

### 16.3 WMA Codec Specifications
| Codec | Format Tag | Channels | Sample Rate | Bit Depth | Notes |
|--------|-----------|----------|-------------|-----------|-------|
| WMA v1 | 0x0161 | 1-2 | 8000, 11025, 22050, 44100 | 16-bit | CBR only |
| WMA v2 | 0x0161 | 1-2 | 8000-44100 | 16-bit | Improved |
| WMA Pro | 0x0163 | 1-8 | 8000-96000 | 16-24-bit | Multichannel |
| WMA Lossless | 0x0162 | 1-2 | 44100-96000 | 16-24-bit | Lossless |

---

## 17. ASF DATA PACKET STRUCTURE

### 17.1 Data Packet Layout
```
Offset   Size   Field                    Type       Description
-------  -----  ----------------------  ---------  -------------------------
0x0000   16     Data Object GUID        GUID       ASF_Data_Object
0x0010   8      Data Object Size       QWORD      Total data object size
0x0018   16     File ID                GUID       Links to Header Object
0x0028   8      Total Data Packets     QWORD      Number of packets
0x0030   8      Reserved               QWORD      Must be 0x00000000

Then per packet:
  ├── Packet Header
  ├── Payload Parsing Information
  ├── Payload Data (1 or more payloads)
  └── Padding
```

### 17.2 Packet Header
```
Offset   Size   Field                    Type       Description
-------  -----  ----------------------  ---------  -------------------------
0x0000   1      Flags                  BYTE        [E][R][R][R][P][P][P][P]
                                       E=error correction present
                                       R=error correction data length
                                       P=payload count (1-127)

0x0001   2      Property Flags        WORD        [S][S][L][L][L][L][L][L]
                                       S=multiple payloads
                                       L=padding length

0x0002   2      Packet Length         WORD        Length of packet (if variable)
0x0004   4      Sequence Number       DWORD       Packet sequence number
0x0008   8      Send Time             QWORD       Presentation time (ms)
0x0010   4      Duration              DWORD       Playback duration (ms)
```

### 17.3 Payload Header
```
Offset   Size   Field                    Type       Description
-------  -----  ----------------------  ---------  -------------------------
0x0000   1      Stream Number          BYTE        [0][0][0][S][S][S][S][S]
                                       Stream number (1-127)

0x0001   2      Time Offset            WORD        Offset from packet send time (ms)

Then based on multiple/single payload flag:
  Single payload:
    0x0003   4      Payload Length      DWORD      Length of payload data
  
  Multiple payload:
    0x0003   1      Payload Flags       BYTE       [P][P][P][P][R][R][R][R]
                                       P=payload length type
                                       R=replication length type
```

---

## 18. ASF ERROR CORRECTION

### 18.1 Error Correction Data Types
| Type | Value | Description |
|------|-------|-------------|
| No error correction | 0x00 | No error correction data |
| Audio-only error correction | 0x02 | Simple parity for audio |

### 18.2 Error Correction Object
```
Offset   Size   Field                    Type       Description
-------  -----  ----------------------  ---------  -------------------------
0x0000   16     Object GUID             GUID       ASF_Error_Correction_Object
0x0010   8      Object Size            QWORD      Total object size
0x0018   1      Error Correction Type   BYTE       Type of error correction
0x0019   1      Error Correction Data Length  BYTE  Length of ECC data
0x001A   N      Error Correction Data   BYTES      ECC payload
```

---

## 19. ASF INDEXING

### 19.1 Index Entry Structure
```
Offset   Size   Field                    Type       Description
-------  -----  ----------------------  ---------  -------------------------
0x0000   4      Packet Number           DWORD      Packet index
0x0004   4      Packet Count            DWORD      Number of packets with offset
0x0008   4      Timestamp               FILETIME   Time of first frame
```

### 19.2 Index Types
| Index Type | GUID | Description |
|-----------|------|-------------|
| Simple Index | ASF_Simple_Index_Object | Frame-based index |
| Index | ASF_Index_Object | Time-based index |
| Media Index | — | Custom application index |

### 19.3 Index Interval
The index interval specifies the time between index entries:

```python
# Index interval calculation
INDEX_INTERVAL_CALC = {
    'formula': 'index_interval = packets_per_second × desired_index_spacing',
    'example': {
        'sample_rate': 44100,
        'samples_per_packet': 1152,
        'packets_per_second': 44100 / 1152 = 38.28,
        'desired_spacing_seconds': 1.0,
        'index_interval': 38.28 × 1.0 = 38 (rounded)',
    }
}
```

---

## 20. ASF METADATA LIBRARY OBJECTS

### 20.1 Metadata Object
```
Offset   Size   Field                    Type       Description
-------  -----  ----------------------  ---------  -------------------------
0x0000   2      Language Index          WORD       ISO 639-2 language code
0x0002   2      Stream Number          WORD       Stream index (0=content desc)
0x0004   2      Name Length            WORD       Length of name string
0x0006   N      Name                   WCHARS     UTF-16LE name string
        + 2      Data Type             WORD       0=Unicode, 1=BYTE, 2=BOOL, 3=DWORD
        + 4      Data Length           DWORD      Length of data
        + M      Data                  BYTES      Metadata value
```

### 20.2 Metadata Library Object
Similar to Metadata Object but stores multiple entries more efficiently:

```python
METADATA_LIBRARY = {
    'max_entries': 65535,  # Maximum descriptors
    'name_length': '16-bit unsigned',
    'data_types': {
        0: 'Unicode string',
        1: 'BYTE array',
        2: 'BOOL (2 bytes)',
        3: 'DWORD (4 bytes)',
        4: 'QWORD (8 bytes)',
        5: 'WORD (2 bytes)',
    },
}
```

---

## 21. WMDRM — DIGITAL RIGHTS MANAGEMENT

### 21.1 WMDRM Overview
WMDRM (Windows Media Digital Rights Management) was Microsoft's DRM system for WMA content. It has been deprecated in favor of PlayReady.

### 21.2 WMDRM Metadata
```
Note: WMDRM-protected files contain additional metadata:
  - Content ID
  - License acquisition URL
  - Expiration date
  - Playback restrictions
  
WMDRM is NOT covered in this document as:
  1. It is deprecated
  2. FFmpeg cannot decode WMDRM content
  3. Modern content uses PlayReady or other DRM
```

### 21.3 PlayReady (Successor)
PlayReady is the successor to WMDRM:
- Used in Windows 8+ and Xbox
- Supports WMA, AAC, and other formats
- Not covered in this document

---

## 22. ASF FILE SIZE LIMITS

### 22.1 Size Restrictions
| Component | Size | Notes |
|-----------|------|-------|
| Object size | Up to 2^64 bytes | Most objects |
| Header Object | Up to 2^64 bytes | Contains all metadata |
| Data Object | Up to 2^64 bytes | Contains all packets |
| Index Object | Up to 2^64 bytes | Optional seeking |
| String length | Up to 65,535 WCHARs | UTF-16LE characters |

### 22.2 Maximum Bitrate
```python
ASF_BITRATE_LIMITS = {
    'wma_v1': {
        'max_bitrate': '192 kbps',
        'typical': '64-128 kbps',
    },
    'wma_v2': {
        'max_bitrate': '320 kbps',
        'typical': '64-192 kbps',
    },
    'wma_pro': {
        'max_bitrate': '768 kbps',
        'typical': '128-384 kbps',
    },
    'wma_lossless': {
        'max_bitrate': '~2,500 kbps (24-bit/96kHz stereo)',
        'typical': '800-1500 kbps',
    },
}
```

---

## 23. ASF METADATA TOOL IMPLEMENTATION

### 23.1 Metadata Reader Implementation
```python
import struct

class ASFMetadataReader:
    """Read metadata from ASF/WMA files."""
    
    def __init__(self, filepath):
        self.filepath = filepath
        self.metadata = {}
        self.header_objects = []
    
    def read_header(self):
        """Parse ASF header."""
        with open(self.filepath, 'rb') as f:
            # Read header object
            header_guid = f.read(16)
            header_size = struct.unpack('<Q', f.read(8))[0]
            
            # Read header objects
            header_end = f.tell() + header_size - 24  # Subtract GUID + size
            
            while f.tell() < header_end:
                obj_guid = f.read(16)
                obj_size = struct.unpack('<Q', f.read(8))[0]
                
                # Handle different object types
                if obj_guid == CONTENT_DESCRIPTION_GUID:
                    self._read_content_description(f)
                elif obj_guid == EXTENDED_CONTENT_GUID:
                    self._read_extended_content(f)
                
                # Move to next object
                f.seek(obj_size - 24, 1)  # Relative seek
    
    def _read_content_description(self, f):
        """Read Content Description Object."""
        title_len = struct.unpack('<H', f.read(2))[0]
        author_len = struct.unpack('<H', f.read(2))[0]
        copyright_len = struct.unpack('<H', f.read(2))[0]
        desc_len = struct.unpack('<H', f.read(2))[0]
        rating_len = struct.unpack('<H', f.read(2))[0]
        
        if title_len > 0:
            self.metadata['title'] = f.read(title_len - 2).decode('utf-16-le')
        # ... read other fields
    
    def _read_extended_content(self, f):
        """Read Extended Content Description Object."""
        desc_count = struct.unpack('<H', f.read(2))[0]
        
        for _ in range(desc_count):
            # Read descriptor
            name_len = struct.unpack('<H', f.read(2))[0]
            name = f.read(name_len - 2).decode('utf-16-le')
            
            data_type = struct.unpack('<H', f.read(2))[0]
            data_len = struct.unpack('<I', f.read(4))[0]
            
            if data_type == 0:  # Unicode string
                value = f.read(data_len - 2).decode('utf-16-le')
            elif data_type == 1:  # BYTE array
                value = f.read(data_len)
            elif data_type == 3:  # DWORD
                value = struct.unpack('<I', f.read(4))[0]
            else:
                value = f.read(data_len)
            
            self.metadata[name] = value
```

### 23.2 Cover Art Reader
```python
class WMCPictureReader:
    """Read WM/Picture (cover art) from ASF metadata."""
    
    def parse_wm_picture(self, data):
        """Parse WM/Picture binary data."""
        offset = 0
        
        # Picture type
        picture_type = data[offset]
        offset += 1
        
        # Image data size
        image_size = struct.unpack('<I', data[offset:offset+4])[0]
        offset += 4
        
        # MIME type (UTF-16LE null-terminated)
        mime_end = data.find(b'\x00\x00', offset) + 2
        mime_type = data[offset:mime_end].decode('utf-16-le')
        offset = mime_end
        
        # Description (UTF-16LE null-terminated)
        desc_end = data.find(b'\x00\x00', offset) + 2
        description = data[offset:desc_end].decode('utf-16-le')
        offset = desc_end
        
        # Image data
        image_data = data[offset:offset+image_size]
        
        return {
            'type': picture_type,
            'mime_type': mime_type,
            'description': description,
            'image_data': image_data,
        }
    
    def picture_type_name(self, type_id):
        """Get picture type name."""
        TYPES = {
            0: 'Other',
            1: 'File icon',
            2: 'Other file icon',
            3: 'Front cover',
            4: 'Back cover',
            5: 'Leaflet',
            6: 'Media',
            7: 'Lead artist',
            8: 'Artist',
            9: 'Conductor',
            10: 'Orchestra',
            11: 'Composer',
            12: 'Lyricist',
            13: 'Recording location',
            14: 'During recording',
            15: 'During performance',
            16: 'Movie/video',
            17: 'Bright colored fish',
            18: 'Illustration',
            19: 'Band logo',
            20: 'Publisher logo',
        }
        return TYPES.get(type_id, 'Unknown')
```

---

## 24. ASF TOOLS AND UTILITIES

### 24.1 TagLib Implementation
```cpp
// Reading ASF/WMA metadata with TagLib
#include <taglib/wma/wmatag.h>
#include <taglib/wma/wmaproperties.h>
#include <taglib/fileref.h>

void read_asf_metadata(const char* filename) {
    TagLib::FileRef file(filename);
    
    if (!file.isNull()) {
        TagLib::ASF::File* asfFile = 
            dynamic_cast<TagLib::ASF::File*>(file.file());
        
        if (asfFile) {
            TagLib::ASF::Tag* tag = asfFile->tag();
            
            std::cout << "Title: " << tag->title().to8Bit() << std::endl;
            std::cout << "Artist: " << tag->artist().to8Bit() << std::endl;
            std::cout << "Album: " << tag->album().to8Bit() << std::endl;
            
            // Read WM/ namespace tags
            const TagLib::ASF::AttributeListMap& attrs = tag->attributeListMap();
            for (auto it = attrs.begin(); it != attrs.end(); ++it) {
                std::string name = it->first.to8Bit();
                
                // Get first attribute value
                if (!it->second.isEmpty()) {
                    const TagLib::ASF::Attribute& attr = it->second.front();
                    
                    if (attr.type() == TagLib::ASF::Attribute::UnicodeStringType) {
                        std::cout << name << ": " 
                                  << attr.toString().to8Bit() << std::endl;
                    }
                }
            }
            
            // Read cover art
            const TagLib::ASF::PictureList& pictures = tag->pictureList();
            for (const auto& pic : pictures) {
                std::cout << "Cover: " << pic.mimeType().to8Bit() << std::endl;
                std::cout << "Type: " << pic.type() << std::endl;
                // pic.picture() contains the image data
            }
        }
    }
}

// Writing ASF/WMA metadata with TagLib
void write_asf_metadata(const char* filename) {
    TagLib::FileRef file(filename);
    
    if (!file.isNull()) {
        TagLib::ASF::File* asfFile = 
            dynamic_cast<TagLib::ASF::File*>(file.file());
        
        if (asfFile) {
            TagLib::ASF::Tag* tag = asfFile->tag();
            
            // Set standard tags
            tag->setTitle("New Title");
            tag->setArtist("New Artist");
            tag->setAlbum("New Album");
            tag->setComment("New Comment");
            
            // Set WM/ namespace tags
            tag->setAttribute("WM/AlbumArtist", 
                TagLib::ASF::Attribute("Album Artist Name"));
            tag->setAttribute("WM/Year", 
                TagLib::ASF::Attribute("2024"));
            tag->setAttribute("WM/Genre", 
                TagLib::ASF::Attribute("Rock"));
            
            // Set WM/Picture
            TagLib::ASF::Picture picture;
            picture.setMimeType("image/jpeg");
            picture.setType(TagLib::ASF::Picture::FrontCover);
            picture.setPicture(cover_image_data);
            picture.setDescription("Front cover");
            tag->addPicture(picture);
            
            file.save();
        }
    }
}
```

### 24.2 FFmpeg Metadata Commands
```bash
# Read all metadata
ffprobe -v quiet -show_format -show_streams input.wma | grep -E "^(TAG|metadata)"

# Read specific tag
ffprobe -v error -show_entries format_tags=title,artist,album \
  -of default=noprint_wrappers=1 input.wma

# Write metadata
ffmpeg -i input.wav -c:a wmav2 -b:a 192k \
  -metadata title="Song Title" \
  -metadata artist="Artist" \
  -metadata album="Album" \
  -metadata year="2024" \
  -metadata track="1" \
  -metadata genre="Genre" \
  output.wma

# Copy all metadata from source
ffmpeg -i input.wma -i output.wma \
  -c copy \
  -map_metadata 0 \
  output_metadata.wma

# Remove all metadata
ffmpeg -i input.wma -c copy -map_metadata -1 output.wma

# Extract cover art
ffmpeg -i input.wma -c:v copy -f image2 output.jpg

# Embed cover art
ffmpeg -i input.wma -i cover.jpg -c copy \
  -attach cover.jpg -metadata:s:v comment="Cover" \
  output_with_cover.wma
```

---

## 25. ASF COMPATIBILITY AND LIMITATIONS

### 25.1 Format Limitations
| Limitation | Value | Notes |
|-----------|-------|-------|
| Maximum file size | 2^64 bytes | Theoretical limit |
| Maximum metadata per file | 64 KB | Header size limit |
| Maximum cover art size | Varies | Player-dependent |
| Maximum string length | 65,535 chars | Per field |
| Maximum tags | 65,535 | Per file |
| Stream count | Up to 127 | Limited by stream number |

### 25.2 Player Compatibility
| Player | Read WMA | Write WMA | Cover Art | Notes |
|--------|-----------|-----------|-----------|-------|
| Windows Media Player | ✓ | ✓ | ✓ | Native |
| VLC | ✓ | ✓ | ✓ | Full support |
| Foobar2000 | ✓ | ✓ | ✓ | Best support |
| iTunes | Partial | ✗ | ✗ | Limited |
| Amarok | ✓ | ✗ | ✓ | Linux |
| Rhythmbox | ✓ | ✗ | ✓ | Linux |

### 25.3 Encoding Compatibility
| Encoder | WMA v1/v2 | WMA Pro | WMA Lossless |
|---------|-----------|---------|-------------|
| Windows Media Encoder | ✓ | ✓ | ✓ |
| FFmpeg | ✓ | ✓ | ✓ |
| dbPowerAmp | ✓ | ✓ | ✓ |
| dBpoweramp | ✓ | ✓ | ✓ |
| Winamp | ✓ | ✗ | ✗ |
| iTunes | ✗ | ✗ | ✗ |

---

## 26. ASF REFERENCE IMPLEMENTATIONS

### 26.1 Open-Source Libraries
| Library | Language | License | WMA Support |
|---------|----------|---------|-------------|
| FFmpeg | C | LGPL | Full decode, encode |
| TagLib | C++ | LGPL | Read/Write tags |
| libavformat | C | LGPL | Demuxer/muxer |
| JAudiotagger | Java | LGPL | Read/Write |
| ATL | C++ | Custom | Read/Write |

### 26.2 Specification Documents
```python
ASF_SPECIFICATIONS = {
    'primary': [
        'ASF Specification (Microsoft)',
        'Windows Media Format SDK Documentation',
        'WAVE FORMAT EXTENSIBLE Specification',
    ],
    'related': [
        'ITU-R BS.775 (channel layouts)',
        'ITU-R BS.1770 (loudness)',
        'EBU R128 (broadcast loudness)',
    ],
}
```

---

## 27. TROUBLESHOOTING ASF/WMA

### 27.1 Common Issues
| Issue | Cause | Solution |
|-------|-------|----------|
| Cover art not showing | Player doesn't support WM/Picture | Convert to supported format |
| Metadata lost on re-encode | Metadata not copied | Use `-c copy` instead of re-encode |
| File won't play | Corrupt header | Repair with tool or re-encode |
| Tag not read | Tag format not recognized | Check with multiple tools |
| Cover art corrupt | Malformed WM/Picture | Re-extract or replace |

### 27.2 Validation Commands
```bash
# Check file integrity
ffmpeg -v error -i input.wma -f null - 2>&1

# Check metadata
ffprobe -v quiet -print_format json \
  -show_format -show_streams input.wma

# Check codec info
ffprobe -v quiet -show_streams input.wma | \
  grep -E "codec_name|codec_long_name|bit_rate"

# Check cover art
ffprobe -v quiet -show_entries \
  format_tags=comment,title -of default=noprint_wrappers=1 input.wma

# Extract binary to inspect
ffmpeg -i input.wma -c copy -f data cover.bin
```

### 27.3 Debug Metadata
```bash
# Hex dump of header object
hexdump -C input.wma | head -100

# Use mkvmerge to examine
mkvmerge -i input.wma

# Use mediainfo for detailed info
mediainfo input.wma

# Check with atomicparsley
atomicparsley input.wma -t
```

---

## 28. ASF METADATA MAPPING REFERENCE

### 28.1 ASF to Other Formats
| ASF/WM Tag | ID3v2 | Vorbis | MP4 | Notes |
|-----------|--------|--------|-----|-------|
| Title | TIT2 | TITLE | ©nam | |
| Author | TPER | ARTIST | ©ART | |
| WM/AlbumTitle | TALB | ALBUM | ©alb | |
| WM/AlbumArtist | TPE2 | ALBUMARTIST | aART | |
| WM/TrackNumber | TRCK | TRACKNUMBER | trkn | |
| WM/Year | TDRC | DATE | ©day | |
| WM/Genre | TCON | GENRE | ©gen | |
| WM/Composer | TCOM | COMPOSER | ©wrt | |
| WM/Publisher | TPUB | PUBLISHER | ©pub | |
| WM/Picture | APIC | METADATA_BLOCK_PICTURE | covr | |
| WM/Comment | COMM | COMMENT | ©cmt | |
| WM/ISRC | TSRC | ISRC | — | |
| WM/Lyrics | USLT | LYRICS | — | |
| WM/CatalogNumber | — | CATALOGNUMBER | — | |
| WM/Barcode | — | BARCODE | — | |
| WM/Mood | — | MOOD | — | |
| WM/AlbumArtistSort | TSOA | ALBUMARTISTSORT | soar | |
| WM/ArtistSort | TSOP | ARTISTSORT | soar | |
| WM/ComposerSort | TSOC | COMPOSERSORT | — | |

### 28.2 Conversion Tips
```bash
# WMA → FLAC (preserve tags)
ffmpeg -i input.wma -c:a flac -compression_level 8 output.flac

# WMA → MP3 (tags → ID3v2)
ffmpeg -i input.wma -c:a libmp3lame -q:a 2 output.mp3

# WMA → AAC (tags → MP4 atoms)
ffmpeg -i input.wma -c:a aac -b:a 256k output.m4a

# Preserve WM/ tags during conversion
# Note: FFmpeg converts WM/ tags to target format
# Verify tags after conversion
ffprobe -v quiet -show_format output | grep -E "^TAG"
```

---

*File expanded with: Stream types, data packet structure, error correction, indexing, metadata implementation, tools reference, compatibility tables, and troubleshooting guide*

---

## 29. ASF OBJECT GUID REFERENCE

### 29.1 Complete GUID Table
| Object Name | GUID | Description |
|-------------|------|-------------|
| ASF_Header_Object | 75B22630-668E-11CF-A6D9-00AA0062CE6C | Top-level container |
| ASF_Data_Object | 75B22636-668E-11CF-A6D9-00AA0062CE6C | Audio/video data |
| ASF_Simple_Index_Object | 33000890-E5B1-11CF-89F4-00A0C90349CB | Frame-based index |
| ASF_Index_Object | D6E229DF-35DA-11D1-9034-00A0C90349CB | Time-based index |
| ASF_File_Properties_Object | 8CABDCA1-A947-11CF-8EE4-00C00C205365 | File-level properties |
| ASF_Stream_Properties_Object | B7DC0791-A9B7-11CF-8EE6-00C00C205365 | Stream properties |
| ASF_Header_Extension_Object | 5FBBC03F-8EDB-11CF-A5D6-28DB04C10000 | Extended features |
| ASF_Content_Description_Object | 75B22633-668E-11CF-A6D9-00AA0062CE6C | Title/author/copyright |
| ASF_Extended_Content_Description_Object | D2D0A440-E307-11D2-97F0-00A0C95EA850 | Extended metadata |
| ASF_Codec_List_Object | 86D15240-311D-11D0-A3A4-00A0C90348F6 | Codec information |
| ASF_Script_Command_Object | 1EFB2093-0000-11CF-8EE6-00C00C205365 | Script commands |
| ASF_Marker_Object | F487CD01-A951-11CF-8EE6-00C00C205365 | Chapter markers |
| ASF_Bitrate_Mutual_Exclusion_Object | D6E229E2-35DA-11D1-9034-00A0C90349CB | Stream exclusion |
| ASF_Error_Correction_Object | 75B22635-668E-11CF-A6D9-00AA0062CE6C | Error correction |
| ASF_Content_Branding_Object | 2211B3FA-BD23-11D2-B4B7-00A0C955E0E6 | Branding data |
| ASF_Content_Encryption_Object | 2211B3FB-BD23-11D2-B4B7-00A0C955E0E6 | Encryption |
| ASF_Digital_Signature_Object | 2211B3FC-BD23-11D2-B4B7-00A0C955E0E6 | Digital signature |
| ASF_Padding_Object | 180600D4-18D1-11CF-AEE6-00C00C205365 | Padding/reserved |

### 29.2 Stream Type GUIDs
| Stream Type | GUID | Description |
|------------|------|-------------|
| Audio | 56406F64-0000-0010-8000-00AA00389B71 | Audio stream |
| Video | 52446D73-0000-0010-8000-00AA00389B71 | Video stream |
| Command | 59FCAC05-0000-0010-8000-00AA00389B71 | Script commands |

---

## 30. ASF METADATA SCHEMA

### 30.1 Core Metadata Schema
```xml
ASF Metadata Schema (simplified):
<ASFFile>
  <Header>
    <FileProperties>...</FileProperties>
    <StreamProperties>
      <AudioStream>...</AudioStream>
      <VideoStream>...</VideoStream>  <!-- if present -->
    </StreamProperties>
    <ContentDescription>
      <Title>...</Title>
      <Author>...</Author>
      <Copyright>...</Copyright>
      <Description>...</Description>
      <Rating>...</Rating>
    </ContentDescription>
    <ExtendedContentDescription>
      <!-- WM/ namespace tags -->
      <Tag name="WM/AlbumTitle">...</Tag>
      <Tag name="WM/TrackNumber">...</Tag>
      <Tag name="WM/Picture">...</Tag>  <!-- binary -->
    </ExtendedContentDescription>
  </Header>
  <Data>
    <!-- Audio/video packets -->
  </Data>
</ASFFile>
```

### 30.2 Extended Content Descriptor Types
| Type Value | Type Name | Size | Encoding |
|-----------|-----------|------|----------|
| 0x0000 | Unicode string | Variable | UTF-16LE |
| 0x0001 | BYTE array | Variable | Binary |
| 0x0002 | BOOL | 2 bytes | 0x0000 or 0x0001 |
| 0x0003 | DWORD | 4 bytes | uint32 LE |
| 0x0004 | QWORD | 8 bytes | uint64 LE |
| 0x0005 | WORD | 2 bytes | uint16 LE |
| 0x0006 | GUID | 16 bytes | GUID |

---

## 31. ASF WMA ENCODING EXAMPLES

### 31.1 FFmpeg WMA Encoding Commands
```bash
# WMA Standard (v1/v2):
ffmpeg -i input.wav -c:a wmav2 -b:a 128k output.wma

# WMA Pro (multichannel):
ffmpeg -i input_5_1.wav -c:a wmapro -b:a 384k output.wma

# WMA Lossless:
ffmpeg -i input.wav -c:a wmalossless output.wma

# WMA with metadata:
ffmpeg -i input.wav \
  -c:a wmav2 -b:a 192k \
  -metadata title="Song Title" \
  -metadata artist="Artist" \
  -metadata album="Album" \
  -metadata year="2024" \
  -metadata track="1" \
  -metadata genre="Rock" \
  -metadata WM/AlbumArtist="Album Artist" \
  -metadata WM/Year="2024" \
  output.wma

# Preserve original metadata:
ffmpeg -i input.wav -i source.wma \
  -c:a wmav2 -b:a 192k \
  -map_metadata 1 \
  output.wma

# High-quality settings:
ffmpeg -i input.wav \
  -c:a wmav2 -b:a 320k -ar 44100 -ac 2 \
  -compression_level 0 \
  output.wma
```

### 31.2 Multi-Channel WMA Pro Encoding
```bash
# 5.1 channel WMA Pro:
ffmpeg -i input_5_1.wav \
  -c:a wmapro -b:a 512k \
  -channel_layout 5.1 \
  output_5.1.wma

# Check channel layout:
ffprobe -v quiet -show_streams input_5_1.wav | grep channel_layout

# Downmix to stereo during encoding:
ffmpeg -i input_5_1.wav \
  -c:a wmapro -b:a 256k \
  -ac 2 \
  output_stereo.wma
```

---

## 32. ASF STREAMING

### 32.1 HTTP Streaming with ASF
```bash
# Create streaming-ready ASF:
ffmpeg -i input.wav \
  -c:a wmav2 -b:a 128k \
  -f asf \
  -listen 1 \
  output.asf

# Stream via HTTP:
# Note: Use a streaming server for production
# ASF is not commonly used for web streaming anymore
```

### 32.2 MMS Protocol
```bash
# Windows Media HTTP streaming:
mms://server/path/file.wma

# MMS over HTTP:
mmsh://server/path/file.wma

# MMS over TCP:
mmst://server/path/file.wma

# Note: MMS is deprecated; use HLS or DASH instead
```

---

## 33. ASF METADATA VALIDATION

### 33.1 Metadata Validation Checklist
```
[ ] Title length < 65,535 characters
[ ] Artist length < 65,535 characters
[ ] Album length < 65,535 characters
[ ] All strings use UTF-16LE encoding
[ ] Numeric fields use correct data types (DWORD, QWORD)
[ ] WM/Picture has correct binary structure
[ ] Cover art MIME type is valid (image/jpeg, image/png)
[ ] Cover art picture type is in range 0-20
[ ] Stream properties match actual media format
[ ] Bitrate values are consistent
```

### 33.2 Common Validation Errors
| Error | Cause | Solution |
|-------|-------|----------|
| Title not displayed | Wrong encoding | Use UTF-16LE |
| Cover art missing | Malformed WM/Picture | Rebuild metadata |
| Tags lost on copy | Player doesn't support | Convert to standard format |
| File won't play | Corrupt header object | Re-encode file |
| Wrong duration | Index corruption | Regenerate index |

---

*File expanded with: Complete GUID reference, metadata schema, encoding examples, streaming information, and validation checklist*
