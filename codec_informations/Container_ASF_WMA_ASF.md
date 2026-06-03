# Advanced Systems Format (ASF/WMA/WMV) — Deep Technical Reference
> **Category:** Container
> **File Extensions:** `.asf`, `.wma`, `.wmv`
> **MIME Types:** `audio/x-ms-wma`, `video/x-ms-asf`, `video/x-ms-wmv`, `application/vnd.ms-asf`
> **Standardization Body:** Microsoft
> **Primary Specification:** Microsoft Windows Media Format SDK (ASF Specification v1.0.1, December 2004)
> **Patent Status:** Patented — proprietary Microsoft technology
> **License:** Proprietary
> **Current Version:** 01.20.03 (December 2004)
> **Active Development:** No — last release December 2004; maintained in Windows Media Foundation

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Microsoft Corporation
- **Year Created:** 1996 (public specification released February 1998)
- **Original Purpose:** Enable synchronized digital media playback and network streaming for audio and video content over the internet, particularly for the Windows Media Player ecosystem
- **Problem with Predecessors:** Existing media container formats (AVI, WAV) lacked built-in streaming primitives, digital rights management (DRM), and efficient multiplexing of multiple audio/video streams with time-based interleaving. ASF was purpose-built for streamed delivery.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| 1.0 | 1996 | Initial release with basic audio/video streaming support |
| 1.01 | 1997 | Bug fixes, minor spec clarifications |
| 1.02 | 1998 | Added metadata support, improved DRM primitives |
| 1.03 | 1999 | Added Header Extension object, codec-specific enhancements |
| 01.20.03 | 2004 | Final specification; added WMA Pro, WMA Lossless, WMV9 support |

### 1.3 Current Adoption
- **Primary use cases today:** Legacy Windows media distribution, archival of Windows Media–encoded content, MMS/RTSP streaming servers still in operation
- **Platforms with native support:** Windows (Media Player, Media Foundation), Windows Phone (deprecated), some Linux via FFmpeg/libav
- **Major services using this format:** Historically: WindowsMedia.com streaming, some internet radio stations, early podcast distribution
- **Hardware support:** Limited modern hardware support; mostly read via software decoders
- **Status:** Declining / legacy — replaced by MP4/HLS/DASH in most applications

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Container Architecture
ASF is an **object-based container**. Every element in an ASF file is an ASF Object, identified by a 128-bit GUID. This design was chosen for forward compatibility: new object types can be added without breaking parsers that don't recognize them.

The top-level ASF file structure consists of exactly three types of objects:

```
[ASF Header Object]      ← Required, must be first
[ASF Data Object]       ← Required, must follow Header
[ASF Index Object(s)]   ← Optional, for seeking
```

### 2.2 High-Level Object Hierarchy
```
ASF Header Object
├── File Properties Object          (GUID: ASF_File_Properties_Object)
├── Stream Properties Object         (one per stream; GUID: ASF_Stream_Properties_Object)
│   └── Codec-specific initialization data (CodecSpecificData)
├── Header Extension Object         (GUID: ASF_Header_Extension_Object)
│   ├── Extended Stream Properties
│   ├── Codec Comment Object
│   ├── Marker Object
│   └── other extended objects
├── Content Description Object       (title, author, copyright)
├── Script Command Object           (URLs, commands)
├── Language List Object
├── Metadata Object                  (key-value metadata)
└── Padding Object

ASF Data Object
└── Data Packets [N]               (interleaved, time-ordered media data)

ASF Simple Index Object
└── Index Entries [N]              (packet_offset, packet_time, packet_num)
```

### 2.3 Key Design Parameters
|| Parameter | Value | Notes |
|-----------|-------|-------|
| Top-level object types | 3 | Header, Data, Index |
| Max streams | 128 | Per ASF specification |
| Max packet size | 64 KB | Recommended; configurable |
| Packet size default | 3200 bytes | FFmpeg default |
| Timestamp resolution | 100-nanosecond units | Windows FILETIME-compatible |
| Timestamp in packets | Milliseconds (ms) | Relative to first packet via preroll |
| Seeking support | Via Index Object | Video frame-accurate |
| DRM | Yes | Proprietary; PlayReady in later versions |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 ASF Object Base Structure
Every ASF object follows this 24-byte header:

```
Offset   Size   Field Name         Type         Description
-------  -----  -----------------  ----------  ---------------------------
0x0000   16B    Object GUID        uint8[16]   128-bit Globally Unique ID
0x0010   8B     Object Size        uint64 LE    Total size in bytes (header = 24 + data)
```

The Object GUID identifies the object type. All known GUIDs are documented below.

### 3.2 File Signature / Magic Bytes
There is no traditional "magic number" for the entire ASF file. Instead, the file begins with the **ASF Header Object**, whose GUID is:

```
Offset  Bytes   Hex Value                                      ASCII     Meaning
------  ------  --------------------------------------------  --------  --------------------
0x0000  16      30 26 B2 75 8E 66 CF 11 A6 D9 00 AA 00 62 CE 6C  ....f.......b.l  ASF Header Object GUID
```

**Well-Known ASF GUIDs:**

| GUID Name | Hex Bytes | Description |
|-----------|-----------|-------------|
| `ASF_Header_Object` | `30 26 B2 75 8E 66 CF 11 A6 D9 00 AA 00 62 CE 6C` | Container header |
| `ASF_File_Properties_Object` | `A1 DC AB 8C 47 A9 CF 11 8E E3 00 C0 0C 20 53 65` | File-level properties |
| `ASF_Stream_Properties_Object` | `B7 75 9C 41 F0 4F 24 B4 0A C5 4D 60 7D BB B2 8D` | Per-stream properties |
| `ASF_Header_Extension_Object` | `5F BF B4 A0 69 50 48 6B 9F 4D 5B 6A 17 D1 EF CD 9A` | Extended header |
| `ASF_Data_Object` | `36 26 B2 75 8E 66 CF 11 A6 D9 00 AA 00 62 CE 6C` | Media packet data |
| `ASF_Simple_Index_Object` | `90 08 00 33 B1 5A 4F 63 18 48 DA BB 8E 0C 34 56` | Seeking index |
| `ASF_Content_Description_Object` | `40 A4 D5 3C CE 8C E4 11 9F 69 00 07 1D 01 2E 52` | Metadata text fields |
| `ASF_Extended_Content_Description_Object` | `40 C6 D5 3C CE 8C E4 11 9F 69 00 07 1D 01 2E 52` | Extended metadata |
| `ASF_Metadata_Object` | `20 CF 06 53 8D 34 ED 44 9A 59 DA 3A 55 92 D5 6E` | Per-stream metadata |
| `ASF_Marker_Object` | `54 96 DC 89 1F 45 4C 6D 07 C8 1D E3 64 7C 5C D7` | Named positions |
| `ASF_Codec_Comment_Header_Object` | `41 82 82 CD 5A 71 4B 57 96 2C 53 15 A2 DA 63 AB` | Codec descriptions |
| `ASF_Language_List_Object` | `A4 96 94 64 98 5A 47 74 65 52 7C DA 7B BC 7D 45` | Language codes |
| `ASF_Group_Mutual_Exclusion_Object` | `A4 96 94 64 98 5A 47 74 65 52 7C DA 7B BC 7D 46` | Stream grouping |
| `ASF_Extended_Stream_Properties_Object` | `CB A5 75 2F 52 1D 4E E7 8E 7E 11 6D 5E 16 1D 6B` | Advanced stream info |
| `ASF_Stream_Bitrate_Properties_Object` | `7B 83 86 65 8B 75 4F 49 D2 0C CC 31 09 33 34 A2` | Per-stream bitrates |
| `ASF_Content_Encryption_Object` | `FB B3 11 22 7C 52 4C 35 9E 65 3B 39 02 23 9D 13` | DRM headers |
| `ASF_Extended_Content_Encryption_Object` | `14 E6 5A 6C 71 54 4D 3D C5 52 2E 08 E3 8A 57 1F` | Extended DRM |
| `ASF_Digital_Signature_Object` | `FC B1 11 22 7C 52 4C 35 9E 65 3B 39 02 23 9D 14` | Digital signature |
| `ASF_Padding_Object` | `00 AC 1F 40 4E 2E 1A 47 BB 39 0C 24 0E 22 3D 19` | Padding filler |

### 3.3 Header Object Layout
```
Offset   Size   Field Name                        Type        Valid Range   Description
-------  -----  --------------------------------  ----------  ------------  ---------------------------
0x0000   16B    Object GUID                       uint8[16]  fixed         ASF_Header_Object GUID
0x0010   8B     Object Size                       uint64 LE   > 30          Total header size in bytes
0x0018   4B     Number of Header Objects          uint32 LE   1–∞          Count of child objects in header
--- Then each child object follows, one after another ---
```

### 3.4 File Properties Object (ASF_File_Properties_Object)
```
Offset   Size   Field Name                        Type        Valid Range   Description
-------  -----  --------------------------------  ----------  ------------  ---------------------------
0x0000   16B    Object GUID                       uint8[16]  fixed        ASF_File_Properties_Object
0x0010   8B     Object Size                       uint64 LE   104+         Size of this object
0x0018   16B    File ID (GUID)                   uint8[16]  nonzero      Unique per-file identifier
0x0028   8B     File Size                         uint64 LE   any          Total file size in bytes
0x0030   8B     Creation Date                     uint64 LE   any          FILETIME (100ns since 1/1/1601)
0x0038   8B     Data Packets Count                uint64 LE   any          Number of data packets
0x0040   8B     Play Duration                     uint64 LE   any          Playback time in 100ns units
0x0048   8B     Send Duration                     uint64 LE   any          Broadcast time (0 if file)
0x0050   8B     Preroll                           uint64 LE   0–∞         ms to subtract from timestamps
0x0058   4B     Flags                             uint32 LE   bitfield     0x01=broadcast, 0x02=seekable
0x005C   4B     Min Data Packet Size               uint32 LE   ≥ 100        Minimum packet size in bytes
0x0060   4B     Max Data Packet Size               uint32 LE   ≤ 65536      Maximum packet size in bytes
0x0064   4B     Max Bitrate                       uint32 LE   any          Sum of all stream bitrates (bps)
```

**Flags field (offset 0x0058):**
```
Bit 0 (0x01): Broadcast flag — if set, play_time/send_time may be invalid
Bit 1 (0x02): Seekable flag — if set, file supports random access
Bits 2–31: Reserved (must be 0)
```

**Preroll field:** A critical field for WMA streaming. If nonzero, the first `preroll` milliseconds of audio data are "pre-buffered" and timestamps start counting from the preroll point. Decoders must subtract preroll from all timestamps to get actual playback position.

### 3.5 Stream Properties Object (ASF_Stream_Properties_Object)
```
Offset   Size   Field Name                        Type        Valid Range   Description
-------  -----  --------------------------------  ----------  ------------  ---------------------------
0x0000   16B    Object GUID                       uint8[16]  fixed        ASF_Stream_Properties_Object
0x0010   8B     Object Size                       uint64 LE   78+          Size of this object
0x0018   16B    Stream Type GUID                  uint8[16]  fixed        Audio=..., Video=...
0x0028   16B    Error Correction Type GUID         uint8[16]  fixed        No error correction=0
0x0038   8B     Time Offset                       uint64 LE   any          Stream time offset (100ns)
0x0040   4B     Type-Specific Data Length         uint32 LE   varies       Length of codec init data
0x0044   4B     Error Correction Data Length       uint32 LE   0 or 18      ECC data length
0x0048   N      Type-Specific Data                uint8[N]    varies       Codec initialization data
0x0048+N  M      Error Correction Data            uint8[M]    0 or 18      Error correction data
```

**Stream Type GUIDs:**

| Stream Type | GUID Bytes |
|-------------|------------|
| Audio | `40 9E 69 F8 2D B9 45 9A 8D D0 11 8E DF DD DD E1` |
| Video | `C0 EF 19 BC 18 55 48 8C 82 0A C2 88 5C E4 D0 89` |
| Command | `59 3A CF 67 46 A4 E2 11 8E F4 F3 7D 81 3F 07 26` |
| No Error Correction | `00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00` |

**Audio Type-Specific Data ( WAVEFORMATEX structure):**
```
Offset   Size   Field Name                        Type        Valid Range   Description
-------  -----  --------------------------------  ----------  ------------  ---------------------------
0x0000   2B     Format Tag                       uint16 LE   codec-specific WMA: 0x0161 (WMA2), 0x0162 (WMA Pro), 0x0163 (WMA Lossless)
0x0002   2B     Number of Channels                uint16 LE   1–8           Audio channels
0x0004   4B     Samples Per Second               uint32 LE   8000–96000    Sample rate (Hz)
0x0008   4B     Average Bytes Per Second          uint32 LE   any          Bitrate / 8
0x000C   2B     Block Alignment                   uint16 LE   codec-specific Byte boundary for codec frames
0x000E   2B     Codec Data Size                   uint16 LE   0 or ≥18      Extra codec init bytes
0x0010   2B     Bits Per Sample                  uint16 LE   16/8          PCM bits; for WMA varies
--- If Codec Data Size >= 18 ---
0x0012   2B     Codec ID (e.g., WMA version)     uint16 LE   varies        Encoder identifier
0x0014   N      Extra Codec Data                  uint8[N]    varies        Additional codec-specific init data
```

**WMA Codec Format Tags:**
| Codec | Format Tag | Bits Per Sample | Channels | Notes |
|-------|-----------|----------------|----------|-------|
| WMA Standard (v1/v2) | 0x0161 | 16 | 1–2 | CBR/VBR |
| WMA Pro | 0x0162 | 16/24 | up to 8 | High-quality |
| WMA Lossless | 0x0163 | 16/24 | up to 8 | True lossless |

### 3.6 Data Object Layout
```
Offset   Size   Field Name                        Type        Valid Range   Description
-------  -----  --------------------------------  ----------  ------------  ---------------------------
0x0000   16B    Object GUID                       uint8[16]  fixed        ASF_Data_Object GUID
0x0010   8B     Object Size                       uint64 LE   any          Total data object size
0x0018   16B    File ID                          uint8[16]  fixed        Must match File Properties File ID
0x0028   8B     Data Packets Count                uint64 LE   any          Number of packets in this object
0x0030   2B     Reserved                         uint16 LE   0x0101       Must be 0x0101
--- Data Packets follow immediately ---
0x0032   N      Data Packets [N]                 variable    variable     Each packet = packet_size bytes
```

### 3.7 ASF Data Packet Structure
Each data packet is exactly `packet_size` bytes (from File Properties `min_pktsize`/`max_pktsize`). ASF packets are **fixed-length containers** that multiplex multiple streams via payload structures.

#### Packet Header (minimum 12 bytes, may be more)
```
Offset  Bit    Field Name              Bits   Type      Description
------  -----  ---------------------  -----  --------  -------------------------------
0       4      Error Correction Flags   4    uint      0=none, 1=present
        1      Has Padding               1    bool      Padding bytes present
        1      Protocol-specific         1    bool      Reserved
        1      Encrypted Content         1    bool      Content is DRM-encrypted
        1      Has Error Correction      1    bool      ECC data follows header
        1      Has Multiple Payloads     1    bool      More than 1 payload in packet
        1      Has Replicated Data      1    bool      Length-prefixed object present
        1      Has Key Frame            1    bool      Packet contains key frame
0       12B    Error Correction Data    (if present)   Variable-length ECC
--- After error correction ---
Variable 12     Packet Length Type       2     uint      0=variable, 1=prescribed, 2=empty, 3=unknown
        12     Property Flags            4     uint      Parser property flags
        16     Packet Timestamp          32    uint      Relative timestamp (ms)
        48     Duration                 16    uint      Packet duration (ms)
        64     Payload Count Type        2     uint      0=fixed(1), 1=byte, 2=word, 3=none
        66     Replicated Data Length    8     uint      Size of replicated data block
        74     Replicated Data           N     bytes     Codec state / multi-packet info
        74+N   Offset IntoMedia          M     variable  Offset for key-frame targeting
        ...    Payload(s)                ...   variable  One or more encoded frames
        ...    Padding                   ...   bytes     Trailing padding to fill packet_size
```

#### Payload Structure (single-payload case):
```
Offset  Size   Field Name              Type        Description
------  -----  ---------------------  ----------  ------------------------------
0       2B     Stream Number           uint16 LE   Which stream this payload belongs to
2       2B     Media Object Offset      uint16 LE   Offset within media object
4       2B     Replicated Data Length   uint16 LE   Length of replicated data
6       N      Replicated Data          bytes       Codec-specific data (may be 0)
6+N     M      Payload Data             bytes       Actual encoded audio/video data
```

#### Multiple Payloads:
If `Has Multiple Payloads` is set, the packet contains a payload header for each payload, followed by the payload data:
```
[Payload Header 1] [Payload Data 1] [Payload Header 2] [Payload Data 2] ... [Padding]
```

### 3.8 Simple Index Object Layout
```
Offset   Size   Field Name                        Type        Valid Range   Description
-------  -----  --------------------------------  ----------  ------------  ---------------------------
0x0000   16B    Object GUID                       uint8[16]  fixed        ASF_Simple_Index_Object GUID
0x0010   8B     Object Size                       uint64 LE   56+          Size of this object
0x0018   16B    File ID                           uint8[16]  fixed        Must match Data Object File ID
0x0028   8B     Index Entry Time Interval          uint64 LE   > 0          ms between index entries
0x0030   4B     Max Packet Count per Index Entry   uint32 LE   ≥ 1          Packets per index interval
0x0034   4B     Index Entries Count                 uint32 LE   any          Number of index entries
--- Index Entries follow ---
0x0038   N      Index Entries [N]                 variable    N × 6 bytes  See below
```

**Index Entry (6 bytes each):**
```
Offset  Size  Field Name         Type        Description
------  -----  -----------------  ----------  -----------------------------
0       4B     Packet Number       uint32 LE   Data packet number
4       2B     Packet Count        uint16 LE   Packets since last entry
```

### 3.9 Sample Format Support
|| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Unsigned integer | Partial | Only for PCM in ASF |
| 16-bit | Signed integer | Yes | Primary format for WMA Standard |
| 24-bit | Signed integer | Yes | WMA Pro, WMA Lossless |
| 32-bit | Signed integer | Yes | WMA Lossless (32-bit PCM mode) |
| 32-bit | IEEE float | Yes | WMA Pro supports float mode |
| 64-bit | IEEE float | No | Not supported |

#### Supported Sample Rates
|| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 8000 | Telephone | Yes | WMA Voice |
| 11025 | — | Yes | |
| 16000 | Wideband voice | Yes | |
| 22050 | — | Yes | |
| 32000 | Broadcast | Yes | |
| 44100 | CD audio | Yes | Standard |
| 48000 | Professional | Yes | Standard |
| 88200 | 2× CD | Yes | WMA Pro |
| 96000 | High-res | Yes | WMA Pro |
| 176400 | 4× CD | Yes | WMA Lossless |
| 192000 | High-res max | Yes | WMA Lossless |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 WMA Encoding Pipeline
ASF is a container format; the encoding algorithm depends on which audio codec is used within ASF.

#### WMA Standard (WMA v1/v2)
```
Input PCM Samples (16-bit, stereo, 44.1kHz)
      │
      ▼
[MDCT — Modified Discrete Cosine Transform]
Block size: 2048 samples (long), 512 samples (short)
Overlap-add for smooth reconstruction
      │
      ▼
[Perceptual Audio Model — Masking Thresholds]
FFT-based psychoacoustic analysis
Scale factor bands aligned with MDCT coefficients
      │
      ▼
[Bit Allocation — Water-filling]
Allocate bits to bands based on masking curves
VBR mode: target quality → bit budget
CBR mode: fixed bit budget per frame
      │
      ▼
[Quantization — Non-uniform mid-tread]
Scale factors per band (lossless-coded as side information)
Huffman coding of quantized coefficients
      │
      ▼
[Frame Packing — ASF Payload]
ASF packet header + stream-specific payload
Interleave multiple streams in data object
      │
      ▼
Output ASF Data Object with interleaved packets
```

### 4.2 WMA Pro Encoding
WMA Pro extends WMA Standard with:
- **Higher sample rates:** Up to 192 kHz
- **More channels:** Up to 8 channels (7.1 surround)
- **Higher bit depths:** 16-bit, 20-bit, 24-bit
- **SBR (Spectral Band Replication):** Low-bitrate bandwidth extension
- **Parametric Stereo:** Low-bitrate stereo coding
- **MDCT block sizes:** 256, 512, 1024, 2048 (adaptive)

### 4.3 WMA Lossless Encoding
```
Input PCM Samples (up to 24-bit, up to 192kHz)
      │
      ▼
[Adaptive Differential PCM (ADPCM-like)]
Predictive coding with variable filter order
Integer arithmetic for exact reconstruction
      │
      ▼
[Entropy Coding — Arithmetic Coding]
Context-adaptive arithmetic coding
Uses small integer arithmetic (not floating point)
Ensures bit-exact reconstruction
      │
      ▼
[Block Structure]
Variable block sizes (lossless requirement)
Blocks are independently decodable
      │
      ▼
Output: WMA Lossless bitstream in ASF container
```

### 4.4 Encoder Settings / Quality Modes
#### WMA Standard
|| Quality Setting | Bitrate (kbps) | Intended Use Case | Notes |
|---|---|---|---|---|
| Lowest | 48–64 | Dialup streaming | Heavily filtered |
| Low | 64–96 | Low-bandwidth | Reduced stereo coupling |
| Medium | 96–128 | General streaming | Default for most streams |
| High | 128–192 | Broadband streaming | Near-transparency |
| Highest | 192–256 | High-quality | Near CD quality |

#### WMA Pro
|| Quality Setting | Bitrate (kbps) | Channels | Notes |
|---|---|---|---|---|
| Voice | 32–64 | Mono/Stereo | Lowest quality |
| Standard | 96–192 | Up to 5.1 | Default |
| High | 256–384 | Up to 7.1 | HiDPI quality |
| Extreme | 512–768 | Up to 7.1 | Near transparent |

#### WMA Lossless
|| Mode | Compression | Bitrate (approx.) |
|---|---|---|---|
| Fast | Lower | ~800–1000 kbps |
| Normal | Medium | ~600–800 kbps |
| High | Higher | ~500–700 kbps |

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. Read ASF Header Object (GUID = ASF_Header_Object at offset 0)
2. Parse File Properties to get packet_size, play_time, preroll
3. Parse all Stream Properties objects to build stream table
4. Skip to Data Object (sequential after header)
5. For each Data Packet:
   a. Read packet header (12+ bytes)
   b. If Has Multiple Payloads: parse each payload header
   c. Extract stream number from payload header
   d. Decode payload using codec from matching Stream Properties
   e. Apply preroll offset to timestamp: real_time = packet_ts - preroll
6. Continue until Data Packets Count exhausted or play_time reached
```

#### Seeking
Seeking requires the **Simple Index Object** (or the **Index Object** in broadcast scenarios):

```
1. Binary search in index: find entry where entry_time <= target_time
2. Seek to packet at: data_object_start + entry.packet_number × packet_size
3. Read packets until target timestamp is reached
4. Apply preroll correction to timestamps
```

Without an index, seeking is possible only by scanning packets sequentially — extremely slow for large files.

### 5.2 Core Decode Pipeline
```
1. Read ASF Header
   ├── Parse all child objects
   ├── Build stream table (stream_number → codec, sample_rate, channels)
   └── Extract codec initialization data (CodecSpecificData from Stream Properties)

2. For each Data Packet:
   ├── Read packet header (determine payload count, sizes)
   ├── For each payload:
   │   ├── Read payload header (stream_number, offset, replicated_data_length)
   │   ├── Look up codec from stream table
   │   └── If Has Replicated Data: pass to codec init (multi-frame blocks)
   │
   ├── Decode payload with appropriate codec
   ├── Apply timestamp offset (subtract preroll)
   └── Queue decoded samples for output (interleave if needed)

3. Flush codec at end of stream

4. Output:
   └── PCM samples with corrected timestamps
```

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** ASF (Advanced Systems Format)
- **Overhead:** ~0.5–3% depending on packet_size configuration
- **Seeking in native container:** Yes — requires Simple Index Object
- **Multiple streams in native container:** Yes — up to 128 streams

### 6.2 Codec-to-Container Compatibility
|| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| ASF (native) | Yes (WMA/WMV) | Yes (index) | Full | Native metadata objects |
| MP4/M4A | No | N/A | N/A | Not applicable |
| Matroska/MKA | No | N/A | N/A | Not applicable |
| OGG | No | N/A | N/A | Not applicable |
| WAV | No | N/A | N/A | Not applicable |
| AIFF | No | N/A | N/A | Not applicable |
| WebM | No | N/A | N/A | Not applicable |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** ASF-native Content Description + Extended Content Description objects
- **Tag block location:** Within ASF Header Object
- **Tag block identifier:** GUID-based Content Description Object

### 7.2 Standard Tag Fields — Complete Reference
|| Tag Field | Internal Key (ASF) | Max Length | Character Encoding | Multi-value | Notes |
|-----------|---------------------------|------------|-------------------|-------------|---|
| Title | Title | 512 chars | UTF-16LE | No | Content Description Object |
| Author | Author / WM/Author | 512 chars | UTF-16LE | No | Content Description Object |
| Copyright | Copyright / WM/Copyright | 512 chars | UTF-16LE | No | Content Description Object |
| Description | Description / WM/Description | 2 KB | UTF-16LE | No | Content Description Object |
| Rating | Rating / WM/Rating | 256 chars | UTF-16LE | No | Extended Content Object |
| Album | WM/AlbumTitle | 512 chars | UTF-16LE | No | Extended Content Object |
| Genre | WM/Genre | 256 chars | UTF-16LE | No | Extended Content Object |
| Year | WM/Year | 4 chars | ASCII | No | Extended Content Object |
| Track Number | WM/TrackNumber | 4 chars | ASCII | No | Extended Content Object |
| Encoder | WM/Encoder | 256 chars | UTF-16LE | No | Extended Content Object |
| ISRC | WM/ISRC | 12 bytes | ASCII | No | Extended Content Object |
| ReplayGain Track Gain | WM/ReplayGain_Track_Gain | 16 chars | ASCII | No | Extended Content Object |
| ReplayGain Track Peak | WM/ReplayGain_Track_Peak | 16 chars | ASCII | No | Extended Content Object |
| Cover Art | WM/Picture | Up to 256 MB | Binary JPEG/PNG | No | Extended Content Object (binary) |

### 7.3 Cover Art Storage in ASF
```
Cover art storage format in ASF:
  Container type:  WM/Picture extended content descriptor
  Image formats:   JPEG, PNG, BMP, GIF (JPEG recommended)
  Max image size:  256 MB (implementation limit)
  
  Binary layout (WM/Picture):
    [0x00-0x00]  Picture Type (1 byte): 0=none, 1=other, 2=front, 3=back, etc.
    [0x01-0xNN]  MIME type (null-terminated UTF-16LE string)
    [0xNN-0xMM]  Description (null-terminated UTF-16LE string)
    [0xMM-...]   Binary image data
```

### 7.4 Metadata Compatibility Matrix
|| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| ASF Native | ✓ | ✓ | ✓ | Highest |
| ID3v2 (at file start) | ✗ | ✗ | N/A | N/A |
| APEv2 (at file end) | ✗ | ✗ | N/A | N/A |
| Vorbis Comments | ✗ | ✗ | N/A | N/A |

**Conflict resolution:** ASF is self-contained; no tag system conflicts.

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):    wmav1, wmav2, wmapro, wmalossless   # used with -c:a
AV_CODEC_ID:         AV_CODEC_ID_WMAV1, AV_CODEC_ID_WMAV2, etc.
Format Name (CLI):    asf, asfh                                       # used with -f
Muxer(s):            asf, asfh                                        # ffmpeg -muxers | grep -i asf
Demuxer(s):          asf, asfh                                        # ffmpeg -demuxers | grep -i asf
```

### 8.2 FFmpeg Encoding — Full CLI Reference

```bash
# Encoding to WMA (ASF container) — complete options reference
ffmpeg -i input.wav \
  -c:a wmav2 \
  -b:a 128k \                   # Bitrate — range: 24-768 kbps, default: 128k
  -ar 44100 \                   # Output sample rate (Hz)
  -ac 2 \                       # Output channel count
  -sample_fmt s16 \              # Sample format: s16, s32, fltp
  output.wma

# WMA Pro encoding (up to 8 channels, 24-bit)
ffmpeg -i input.wav \
  -c:a wmapro \
  -b:a 256k \
  -ar 48000 \
  -ac 6 \
  -sample_fmt s32 \
  output.wma

# WMA Lossless encoding
ffmpeg -i input.wav \
  -c:a wmalossless \
  output.wma

# ASF muxer options (for WMA/WMV)
ffmpeg -i input.wav \
  -c:a wmav2 \
  -f asf \
  -packet_size 3200 \            # ASF packet size — range: 100-65536, default: 3200
  output.asf
```

#### Complete FFmpeg Option Table
|| Option | Type | Default | Range / Valid Values | Effect on Output |
|--------|------|---------|---------------------|-----------------|
| `-b:a` | int | 128k | 24k–768k | Target bitrate |
| `-ar` | int | 44100 | 8000–192000 | Output sample rate |
| `-ac` | int | from input | 1–8 | Output channel count |
| `-sample_fmt` | string | s16 | s16, s32, fltp | Sample format |
| `-packet_size` | int | 3200 | 100–65536 | ASF data packet size |
| `-write_id3` | bool | 0 | 0/1 | Write ID3v2 before ASF header |

### 8.3 FFmpeg Encoding — libavcodec C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/opt.h>
#include <libswresample/swresample.h>

// ─── 1. Find encoder ─────────────────────────────────────────────────────────
const AVCodec *codec = avcodec_find_encoder(AV_CODEC_ID_WMAV2);
// Alternative by name: avcodec_find_encoder_by_name("wmav2");
if (!codec) { fprintf(stderr, "Encoder not found\n"); exit(1); }

// ─── 2. Allocate codec context ───────────────────────────────────────────────
AVCodecContext *ctx = avcodec_alloc_context3(codec);
if (!ctx) { fprintf(stderr, "OOM\n"); exit(1); }

// ─── 3. Set required parameters ──────────────────────────────────────────────
ctx->bit_rate    = 128000;                  // bits/sec
ctx->sample_fmt  = AV_SAMPLE_FMT_S16;       // WMAv2 supports S16, S32
ctx->sample_rate = 44100;                   // Hz
av_channel_layout_copy(&ctx->ch_layout, &(AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO);

// ─── 4. Open codec ───────────────────────────────────────────────────────────
AVDictionary *opts = NULL;
av_dict_set(&opts, "key", "value", 0);
int ret = avcodec_open2(ctx, codec, &opts);
if (ret < 0) {
    char errbuf[128]; av_strerror(ret, errbuf, sizeof(errbuf));
    fprintf(stderr, "avcodec_open2 failed: %s\n", errbuf);
    exit(1);
}

// ─── 5. Allocate frame and packet ────────────────────────────────────────────
AVFrame  *frame = av_frame_alloc();
frame->format      = ctx->sample_fmt;
frame->sample_rate = ctx->sample_rate;
av_channel_layout_copy(&frame->ch_layout, &ctx->ch_layout);
frame->nb_samples  = ctx->frame_size;   // WMA: typically 1152 or 2048 samples
av_frame_get_buffer(frame, 0);

AVPacket *pkt = av_packet_alloc();

// ─── 6. Encode loop ──────────────────────────────────────────────────────────
void encode_frame(AVCodecContext *ctx, AVFrame *frame, AVPacket *pkt, FILE *outfile) {
    int ret = avcodec_send_frame(ctx, frame);
    if (ret < 0) { /* handle AVERROR(EAGAIN), AVERROR(EINVAL) */ }

    while (ret >= 0) {
        ret = avcodec_receive_packet(ctx, pkt);
        if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF) return;
        if (ret < 0) { /* fatal error */ exit(1); }
        fwrite(pkt->data, 1, pkt->size, outfile);
        av_packet_unref(pkt);
    }
}

// ─── 7. Flush encoder ────────────────────────────────────────────────────────
encode_frame(ctx, NULL, pkt, outfile);

// ─── 8. Cleanup ──────────────────────────────────────────────────────────────
av_frame_free(&frame);
av_packet_free(&pkt);
avcodec_free_context(&ctx);
```

**Critical Notes:**
- `ctx->frame_size` for WMA codecs is typically **1152 samples** per encoded frame
- WMA codecs use **MDCT-based encoding**; frame sizes are fixed
- WMA Lossless requires `AV_SAMPLE_FMT_S32` output format
- Always check `codec->sample_fmts` before setting sample_fmt

### 8.4 FFmpeg Decoding — Full CLI Reference

```bash
# Decode to raw PCM WAV
ffmpeg -i input.wma \
  -c:a pcm_s16le \
  -ar 44100 \
  output.wav

# Extract specific stream (multi-stream ASF)
ffmpeg -i input.asf -map 0:a:0 -c:a copy output.wma

# Probe format without decoding
ffprobe -v quiet -print_format json -show_streams -show_format input.wma
```

### 8.5 FFmpeg Decoding — libavcodec C API

```c
// ─── Complete decode program skeleton ─────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.wma", NULL, NULL);
avformat_find_stream_info(fmt_ctx, NULL);

// Find audio stream
int audio_idx = av_find_best_stream(fmt_ctx, AVMEDIA_TYPE_AUDIO, -1, -1, NULL, 0);
AVStream *stream = fmt_ctx->streams[audio_idx];

// Find and open decoder
const AVCodec *dec = avcodec_find_decoder(stream->codecpar->codec_id);
AVCodecContext *dec_ctx = avcodec_alloc_context3(dec);
avcodec_parameters_to_context(dec_ctx, stream->codecpar);
avcodec_open2(dec_ctx, dec, NULL);

// Decode loop
AVPacket *pkt = av_packet_alloc();
AVFrame  *frm = av_frame_alloc();

while (av_read_frame(fmt_ctx, pkt) >= 0) {
    if (pkt->stream_index == audio_idx) {
        avcodec_send_packet(dec_ctx, pkt);
        while (avcodec_receive_frame(dec_ctx, frm) == 0) {
            // frm->data[0..channels-1] contain audio samples
            // frm->nb_samples = sample count per channel
            // frm->format = AVSampleFormat
            // frm->sample_rate = actual rate
            // frm->pts = presentation timestamp (apply preroll correction)
            process_audio_frame(frm);
            av_frame_unref(frm);
        }
    }
    av_packet_unref(pkt);
}

// Flush decoder
avcodec_send_packet(dec_ctx, NULL);
while (avcodec_receive_frame(dec_ctx, frm) == 0) {
    process_audio_frame(frm);
    av_frame_unref(frm);
}

// Cleanup
av_frame_free(&frm);
av_packet_free(&pkt);
avcodec_free_context(&dec_ctx);
avformat_close_input(&fmt_ctx);
```

### 8.6 FFmpeg Metadata Handling

```bash
# Read all metadata as JSON
ffprobe -v quiet -print_format json -show_format input.wma | jq .format.tags

# Write metadata (copy audio, update tags)
ffmpeg -i input.wma \
  -c copy \
  -metadata title="Song Title" \
  -metadata author="Artist Name" \
  -metadata copyright="(c) 2024" \
  -metadata title="Album Name" \
  output.wma

# Strip all metadata
ffmpeg -i input.wma -c copy -map_metadata -1 output.wma

# Write ID3v2-compatible tags (FFmpeg ID3v2 writing for ASF)
ffmpeg -i input.wma \
  -c copy \
  -write_id3 1 \
  -id3v2_version 3 \
  -metadata title="Title" \
  output.asf
```

#### FFmpeg Internal Metadata Key Mapping
|| Standard Field | FFmpeg Key | ASF Native Key | Notes |
|----------------|------------|--------------------------|-------|
| Title | title | title | Title | |
| Author | artist | author | Author / WM/Author | |
| Copyright | copyright | copyright | Copyright | |
| Description | description | description | WM/Description | |
| Album | album | album | WM/AlbumTitle | |
| Genre | genre | genre | WM/Genre | |
| Year | date | date | WM/Year | |
| Track Number | track | track | WM/TrackNumber | |

### 8.7 Quality / Fidelity Decision Table
|| Use Case | Settings | Expected Size (min) | Notes |
|----------|----------|---------------------|-------|
| Archival / lossless | `-c:a wmalossless` | ~0.55× uncompressed | WMA Lossless |
| High-quality streaming | `-c:a wmapro -b:a 256k` | ~256 kbps | 7.1 surround |
| Standard streaming | `-c:a wmav2 -b:a 128k` | ~128 kbps | Default choice |
| Low bandwidth | `-c:a wmav1 -b:a 64k` | ~64 kbps | Mono/low-bitrate |
| Podcast / voice | `-c:a wmav2 -b:a 48k -ac 1` | ~48 kbps | Mono |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
ASF uses the **Simple Index Object** for seeking, indexed by video stream (time-ordered).

```
Simple Index Object:
  Location:     End of file (after Data Object)
  Magic:       GUID = ASF_Simple_Index_Object
  Entry size:  6 bytes per entry
  Entry format:
    [0x00–0x03]  Packet Number  (uint32 LE)
    [0x04–0x05]  Packet Count   (uint16 LE) — packets in this interval
  Index interval: stored in Index Entry Time Interval field (ms)
  Max entries:   (Object Size - 56) / 6
```

### 9.2 Gapless Playback Data
```
ASF does not store explicit gapless metadata.
Encoder delay:  Varies by codec (WMA: ~4–6 ms per frame)
Padding:       Last frame padded to frame boundary
Preroll field: Used for buffering, not gapless; subtract from all timestamps
Storage location: File Properties Object → preroll field
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

|| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | ASF was designed for streaming |
| Algorithmic encoder delay | ~20–50 ms | MDCT lookahead + frame size |
| Live encoding feasible | Yes | With broadcast flag set |
| HTTP progressive download | Yes | .wma/.wmv files |
| HTTP Live Streaming (HLS) | No | Not supported |
| DASH streaming | No | Not natively supported |
| WebRTC / RTP transport | Partial | Via MMS/MMSH |
| RTP/RTSP streaming | Yes | MMS/RTSP protocol |
| Minimum decode buffer | ~1152–2048 samples | One frame |

### 10.1 Streaming Protocols
| Protocol | Description | FFmpeg Support |
|----------|-------------|---------------|
| MMS (Microsoft Media Server) | Old Windows Media streaming | Yes (mms://) |
| MMSH (MMS over HTTP) | HTTP tunneling for MMS | Yes (mmsh://) |
| MMST (MMS over TCP) | Direct TCP MMS | Yes (mmst://) |
| RTSP (Real Time Streaming) | Standard streaming protocol | Partial |
| HTTP | Progressive download | Yes |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
|| Channels | Layout Name | Channel Order | FFmpeg Layout Constant |
|----------|-------------|---------------|----------------------|
| 1 | Mono | C | AV_CHANNEL_LAYOUT_MONO |
| 2 | Stereo | L, R | AV_CHANNEL_LAYOUT_STEREO |
| 2 | Joint Stereo | M, S | Mid-side stereo |
| 3 | 2.1 | L, R, LFE | AV_CHANNEL_LAYOUT_2POINT1 |
| 4 | Quad | FL, FR, RL, RR | AV_CHANNEL_LAYOUT_QUAD |
| 5 | 5.0 | FL, FR, C, SL, SR | AV_CHANNEL_LAYOUT_5POINT0 |
| 6 | 5.1 | FL, FR, C, LFE, SL, SR | AV_CHANNEL_LAYOUT_5POINT1 |
| 7 | 6.1 | FL, FR, C, LFE, BL, BC, BR | WMA Pro extended |
| 8 | 7.1 | FL, FR, C, LFE, BL, BR, SL, SR | AV_CHANNEL_LAYOUT_7POINT1 |

### 11.2 Downmix Coefficients (5.1 → Stereo)
```
L_out = L + C × 0.7071 + LS × 0.7071
R_out = R + C × 0.7071 + RS × 0.7071
LFE:  discarded

FFmpeg downmix for WMA:
ffmpeg -i input_5_1.wma \
  -af "pan=stereo|FL=FC+0.707*FC+0.707*BL|FR=FC+0.707*FC+0.707*BR" \
  output_stereo.wma
```

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

|| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 24-bit (WMA Pro, WMA Lossless) | Integer |
| Max sample rate | 192 kHz (WMA Pro, WMA Lossless) | |
| Float support | Yes (32-bit float in WMA Pro) | |
| DSD support | No | Not applicable |
| 20-bit support | Yes | WMA Pro |

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

|| Platform / API | Encode | Decode | FFmpeg Flag | Notes |
|----------------|--------|--------|-------------|-------|
| Windows Media Foundation | Yes | Yes | Native | Built-in Windows codec |
| FFmpeg native | Yes | Yes | `-c:a wmav2` | LGPL |
| Intel QSV | No | No | — | Not supported |
| Apple VideoToolbox | No | No | — | WMA not supported |
| Android MediaCodec | No | Partial | — | Some devices via software |

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Encoder-Specific Issues
|| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| WMA Lossless decode issues with certain sample rates | Older versions (< 4.2) | Update FFmpeg |
| Seeking without index very slow | All | Rebuild file with index |
| Preroll not correctly applied in some players | All | Use `-ss` for seeking instead |
| WMA Pro 8-channel support incomplete | Older versions | Downmix to fewer channels |

### 14.2 Interoperability Issues
- **FFmpeg → Windows Media Player:** Files encoded with FFmpeg may have DRM header issues in older WMP versions
- **Files with very large preroll values:** Some decoders misinterpret preroll as buffer time rather than timestamp offset
- **WMA Lossless → non-WMA players:** Requires transcoding; WMA Lossless is proprietary
- **Variable packet sizes in non-streaming mode:** Not well supported; use fixed packet_size

### 14.3 Edge Cases to Handle in Converter
- **Empty file / 0-sample stream:** ASF header parses but no audio data; skip gracefully
- **File < 1 packet:** Valid; decode what exists
- **All-silence audio:** ASF handles efficiently; constant subframes
- **DC offset (non-zero mean):** WMA encoders may strip DC; use PCM source
- **Full-scale sine (0 dB):** No clipping in WMA Lossless; possible clipping in WMA lossy
- **File with corrupt header:** Attempt to find Data Object by scanning for ASF_Data_Object GUID
- **Truncated file:** Stop at last valid packet; don't report error for incomplete last packet
- **Sample rate not supported by codec:** Error out with clear message
- **Channel count > 8:** Not supported by any WMA codec

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM ASF/WMA

|| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → FLAC | `ffmpeg -i in.wma -c:a flac -compression_level 8 out.flac` | Via ASF metadata objects | Lossless if source was WMA Lossless |
| → ALAC | `ffmpeg -i in.wma -c:a alac out.m4a` | Partial (tag mapping) | Lossless if source was WMA Lossless |
| → MP3 | `ffmpeg -i in.wma -c:a libmp3lame -q:a 0 out.mp3` | ID3v2.3 via -metadata | Generation loss if re-encoding lossy |
| → AAC | `ffmpeg -i in.wma -c:a aac -b:a 256k out.m4a` | Partial | Generation loss if re-encoding lossy |
| → Opus | `ffmpeg -i in.wma -c:a libopus -b:a 128k out.opus` | Limited | Generation loss if re-encoding lossy |
| → WAV | `ffmpeg -i in.wma -c:a pcm_s16le out.wav` | Limited | Lossless if source was WMA Lossless |
| → OGG Vorbis | `ffmpeg -i in.wma -c:a libvorbis -q:a 6 out.ogg` | Limited | Generation loss |

### 15.2 Converting TO ASF/WMA

|| Source | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| FLAC → WMA Lossless | `ffmpeg -i in.flac -c:a wmalossless out.wma` | ASF native | Lossless |
| WAV → WMA | `ffmpeg -i in.wav -c:a wmav2 -b:a 128k out.wma` | ASF native | Generation loss |
| MP3 → WMA | `ffmpeg -i in.mp3 -c:a wmav2 -b:a 128k out.wma` | ID3v2 → ASF | Double generation loss |
| AAC → WMA | `ffmpeg -i in.m4a -c:a wmav2 -b:a 128k out.wma` | Via -metadata | Generation loss |
| Vorbis → WMA | `ffmpeg -i in.ogg -c:a wmav2 -b:a 128k out.wma` | Limited | Generation loss |

### 15.3 Lossless Round-Trip Verification
```bash
# Encode
ffmpeg -i original.wav -c:a wmalossless output.wma

# Decode back
ffmpeg -i output.wma -c:a pcm_s16le decoded.wav

# Compare checksums
ffmpeg -i original.wav -c:a pcm_s16le original_raw.wav
md5sum original_raw.wav decoded_raw.wav   # Must match for true lossless

# Or use FFmpeg's built-in checksumming:
ffmpeg -i original.wav -map 0:a -f framemd5 original.md5
ffmpeg -i output.wma -map 0:a -f framemd5 output.md5
diff original.md5 output.md5   # Empty diff = bit-perfect
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

|| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| Windows Media Format SDK | C/C++ | Proprietary | Reference | Reference | Microsoft (archived) |
| FFmpeg native (libavcodec) | C | LGPL 2.1+ | 8/10 | 9/10 | https://ffmpeg.org |
| libvpx (for WMV9/VC-1) | C | BSD | N/A | 10/10 | https://www.webmproject.org |
| MEncoder | C | GPL | 7/10 | — | Part of FFmpeg |

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **ASF Specification v1.01:** Microsoft — https://download.microsoft.com (archived)
- **Windows Media Format SDK Documentation:** Microsoft Learn (archived)

### Technical Resources
- FFmpeg encoder options: `ffmpeg -h encoder=wmav2` or https://ffmpeg.org/ffmpeg-codecs.html
- FFmpeg muxer options: `ffmpeg -h muxer=asf` or https://ffmpeg.org/ffmpeg-formats.html
- Multimedia Wiki: https://wiki.multimedia.cx/index.php/ASF
- Hydrogenaudio: https://hydrogenaud.io/index.php/board,83.0.html

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] Identify FFmpeg `./configure` flags needed to enable WMA codecs
- [ ] Verify `ffmpeg -encoders` output confirms wmav1, wmav2, wmapro, wmalossless are available
- [ ] Verify `ffmpeg -decoders` output confirms ASF demuxer is available
- [ ] Note: WMA codecs are built into FFmpeg (no external dependency)
- [ ] Handle platform where WMA may not be available (some minimal FFmpeg builds)

### Encoding Pipeline
- [ ] Convert input sample format to required `ctx->sample_fmt` using libswresample
- [ ] Handle fixed-frame-size encoders (`ctx->frame_size` = 1152 for WMA)
- [ ] Implement proper flush/drain at end of stream (send NULL frame)
- [ ] Store encoder delay and padding for gapless playback
- [ ] Validate that input sample rate is supported (8000–192000 Hz)
- [ ] Validate that channel layout is supported, auto-downmix if > 8 channels
- [ ] Set preroll correctly for streaming applications

### Decoding Pipeline
- [ ] Implement ASF Header Object parsing (sequential GUID-based objects)
- [ ] Handle AVERROR(EAGAIN) in receive_packet loop correctly
- [ ] Flush decoder at EOF (send NULL packet)
- [ ] Apply preroll correction to all presentation timestamps
- [ ] Handle sample format conversion from decoder output format
- [ ] Build stream table for multi-stream files

### Metadata
- [ ] Read all standard ASF metadata fields (Title, Author, Copyright, Description)
- [ ] Read Extended Content Descriptors (WM/AlbumTitle, WM/Genre, etc.)
- [ ] Read WM/Picture cover art
- [ ] Write all standard ASF metadata fields to output
- [ ] Embed cover art in WM/Picture extended content
- [ ] Handle UTF-16LE encoding in ASF metadata strings
- [ ] Preserve ReplayGain tags through conversion
- [ ] Handle DRM Content Encryption objects (read-only, pass through)

### Quality & Verification
- [ ] Implement ReplayGain scan via EBU R128 (libebur128 integration)
- [ ] Write ReplayGain tags in ASF Extended Content format
- [ ] Provide bit-exact verification for lossless conversions (WMA Lossless)
- [ ] Implement progress reporting from FFmpeg stats output
- [ ] Implement error detection for corrupt ASF headers
- [ ] Test with: silence, full-scale, clipped, multi-channel, high-resolution files

### Edge Cases
- [ ] Handle files with corrupt or missing Index Object (slow seeking)
- [ ] Handle files with preroll > 0 (timestamp correction)
- [ ] Handle sample rate mismatch (trigger libswresample)
- [ ] Handle channel count mismatch (trigger downmix or upmix)
- [ ] Handle bit depth mismatch (trigger conversion)
- [ ] Handle very short files (< 1 packet)
- [ ] Handle files with DRM Content Encryption Object (cannot decode without DRM)

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
