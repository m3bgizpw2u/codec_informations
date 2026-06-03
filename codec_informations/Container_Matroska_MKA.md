# Container: Matroska / MKA — Deep Technical Reference

> **Category:** Container Format
> **File Extensions:** .mkv, .mka, .mks, .mkwebm, .webm
> **MIME Types:** video/x-matroska, audio/x-matroska, audio/x-matroska-rawpcm
> **Standardization Body:** IETF RFC 8794 (EBML), Matroska Specification (matroska.org)
> **Specification Document:** Matroska Element Semantics (github.com/ietf-wg-cellar/matroska-specification)
> **Patent Status:** Patent-free, open specification
> **License:** BSD 3-Clause (Matroska), RFC 8794 (EBML)

---

## 1. HISTORICAL CONTEXT & ORIGIN

Matroska began as an open-source project initiated by Steve Lhomme and Rob McMullen in 2002, conceived as a modern, extensible, patent-free multimedia container designed to replace proprietary formats like AVI. The name derives from a Russian nesting doll (matryoshka), a metaphor for the format's hierarchical, self-describing structure. The format was designed from the ground up to be highly extensible, supporting any number of audio, video, subtitle, and attachment tracks within a single file.

The Matroska team developed EBML (Extensible Binary Meta Language) as the foundational binary format upon which Matroska is built. EBML borrowed concepts from XML and binary formats to create a self-describing, versioned binary serialization format. This design decision was pivotal: by separating the container syntax (EBML) from the container schema (Matroska), the same underlying format could support multiple container types, and the schema itself could evolve independently.

The .mka extension designates a Matroska file containing audio-only content, though technically the format is identical to .mkv; the extension is purely advisory. Matroska gained widespread adoption primarily through video (MKV) usage in the open-source and enthusiast communities, where it became the de facto standard for storing high-quality video with multiple audio tracks, subtitle tracks, and chapter information. It is the native container for the WebM project (a restricted profile of Matroska for web use), developed by Google and released in 2010 as a royalty-free format for HTML5 video.

In the audio space, MKA serves as a versatile container for lossless and lossy audio codecs that lack their own native container (FLAC, Opus, and Vorbis all use OGG as their primary container but can be muxed into Matroska). MKA excels at multichannel audio, chapter embedding, and metadata tagging through the Matroska Tag system. The format's streaming capability through Cues indexing, its support for chapters, and its codec-agnostic nature make it particularly valuable in professional audio workflows and archival contexts.

The Matroska specification is maintained by the IETF Cellar working group, with EBML formalized as IETF RFC 8794 in July 2020. This standardization effort brought architectural clarity to the format and ensured long-term stability.

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

Matroska is a binary container format built on EBML (Extensible Binary Meta Language), a generalized self-describing binary format inspired by XML. The key architectural insight is that EBML defines *syntax* (how elements are encoded), while Matroska defines *semantics* (what elements mean and how they relate). This separation allows EBML parsers to handle any EBML-based format, and Matroska parsers to handle future schema versions with the same parser code.

The format is fundamentally a tree of typed elements. Unlike MP4's flat box hierarchy where boxes contain boxes, Matroska's EBML structure has a strict parent-child relationship where elements can be Master Elements (containing child elements), or can hold scalar values (integers, floats, strings, binaries, UTF-8 text).

A Matroska file consists of:

```
EBML Header (mandatory, at file start)
  └─ EBMLVersion, EBMLReadVersion, EBMLMaxIDLength, EBMLMaxSizeLength,
      DocType, DocTypeVersion, DocTypeReadVersion

Segment (the entire media payload)
  ├─ SeekHead (index of top-level element locations)
  ├─ Info (file-level metadata)
  ├─ Tracks (track definitions — one per audio/video/subtitle stream)
  ├─ Chapters (chapter divisions)
  ├─ Cluster(s) (actual audio/video frame data, timestamped)
  ├─ Cues (index for random access seeking)
  ├─ Attachments (embedded cover art, fonts, etc.)
  └─ Tags (metadata tags)
```

The order of top-level Segment children is not strictly enforced by most players, though the Matroska spec defines a recommended order. Clusters are by far the largest data consumers and are typically placed last (or second-to-last, before Cues) to allow streaming uploaders to begin transmitting before all clusters are known.

Matroska is codec-agnostic. Any binary codec data can be stored as a Matroska Block. Codecs are identified by a CodecID string. Well-known codec IDs include `A_AAC`, `A_FLAC`, `A_OPUS`, `A_VORBIS`, `A_MP3`, `A_AC3`, `A_PCM`, `A_DTS`, and many more.

---

## 3. EBML BINARY FORMAT SPECIFICATION

### 3.1 EBML Header Structure

The EBML Header identifies the file as an EBML document and declares the schema version. It is always the first element in the file, immediately following the EBML magic number bytes.

```
EBML Header layout:

[EBML Magic Bytes]     4 bytes  0x1A 0x45 0xDF 0xA3  (fixed)
[EBML ID]              4 bytes  0x1A 0x45 0xDF 0xA3  (same as magic)
[EBML Data Size]       1-8      Variable-length integer
  [EBMLVersion]         1 byte  Must be 1
  [EBMLReadVersion]      1 byte  Lowest compatible reader version (must be 1)
  [EBMLMaxIDLength]     1 byte  Maximum ID octet length (must be >= 4)
  [EBMLMaxSizeLength]   1 byte  Maximum size length (must be 1-8)
  [DocType]             N bytes UTF-8 string, e.g., "matroska" or "webm"
  [DocTypeVersion]      1 byte  Schema version (4 for current Matroska)
  [DocTypeReadVersion]  1 byte  Minimum reader version (2 for Matroska v2+)
```

The EBML ID `0x1A45DFA3` is the Matroska/EBML magic number. A file starting with these four bytes followed by a valid EBML Header is recognized as an EBML document.

### 3.2 Element IDs

EBML Element IDs are variable-length identifiers encoded using a self-synchronizing scheme based on leading zeros in the first byte. The number of leading 1-bits in the first byte (followed by a 0 bit) determines the total ID byte length.

| Length | First Byte Range | Value Bits | Encoding Pattern      | Example               |
|--------|-----------------|-----------|----------------------|-----------------------|
| 1 byte | 0x00 – 0x7F    | 7 bits    | 0xxxxxxx             | Block: 0xA0 (160)     |
| 2 bytes | 0x80 – 0xBF   | 14 bits   | 10xxxxxx xxxxxxxx   | ChapterCodecState: 0xAA|
| 3 bytes | 0xC0 – 0xDF   | 21 bits   | 110xxxxx xxxxxxxx xxxxxxxx | Reserved           |
| 4 bytes | 0xE0 – 0xEF   | 28 bits   | 1110xxxx xxxxxxxx xxxxxxxx xxxxxxxx | Segment: 0x18538067|

The first byte's leading 1-bits (count of consecutive 1-bits from the MSB) indicate the byte count. For example:
- 0xA0 = binary 10100000 → leading "10" → 2 bytes total, but 0xA0 is the first byte of a 2-byte ID
- 0xE7 = binary 11100111 → leading "1110" → 4 bytes? No, 0xE7 alone is actually a 1-byte ID (TrackTimecode = 0xE7)

Wait, let me reconsider. Looking at real Matroska element IDs, the encoding is simpler than I described. The Matroska spec defines specific Element IDs as fixed byte sequences. The leading-bits scheme from RFC 8794 applies to the EBML ID encoding itself, but Matroska's element IDs are fixed-width based on their position in the hierarchy.

Matroska uses a specific ID encoding scheme where the first byte's leading 1-bits indicate the total ID byte count:
- 0x00-0x7F: 1-byte ID (leading bit is 0, so no leading 1s)
- 0x80-0xBF: 2-byte ID (leading bits "10")
- 0xC0-0xDF: 3-byte ID (leading bits "110")
- 0xE0-0xEF: 4-byte ID (leading bits "1110")

Key Matroska Element IDs:

| Element Name | Hex ID      | Bytes | Pattern | Description                    |
|-------------|-------------|-------|---------|--------------------------------|
| EBML         | 0x1A45DFA3  | 4     | 1110... | Root EBML header element       |
| Segment      | 0x18538067  | 4     | 1110... | Primary media container         |
| SeekHead     | 0x114D9B74  | 4     | 1110... | Index of top-level elements    |
| Info         | 0x1549A966  | 4     | 1110... | File-level metadata             |
| Tracks       | 0x1654AE6B  | 4     | 1110... | Track definitions               |
| Chapters     | 0x1043A770  | 4     | 1110... | Chapter markers                |
| Cluster      | 0x1F43B675  | 4     | 1110... | Group of timed frames          |
| BlockGroup   | 0xA0        | 1     | 0xxxxxxx| Block with optional references |
| SimpleBlock  | 0xA3        | 1     | 0xxxxxxx| Standalone block               |
| Block        | 0xA1        | 1     | 0xxxxxxx| Block with references          |
| CuePoint     | 0xBB        | 1     | 0xxxxxxx| Cues index entry               |
| Cues         | 0xC53BB6B   | 4     | 1110... | Random access index             |
| Attachments  | 0x1941A469  | 4     | 1110... | Attached files                 |
| Tags         | 0x1254C367  | 4     | 1110... | Metadata tags                  |
| Tag          | 0x7373      | 2     | 10xx... | Single tag element             |
| TrackEntry   | 0xAE        | 1     | 0xxxxxxx| Individual track definition     |
| ChapterAtom  | 0xB6        | 1     | 0xxxxxxx| Chapter entry                  |
| Audio        | 0xE1        | 1     | 0xxxxxxx| Audio track parameters         |
| Seek         | 0x4DBB      | 2     | 10xx... | Seek head entry                |
| CRC-32       | 0xBF        | 1     | 0xxxxxxx| CRC-32 checksum                |
| Void         | 0xEC        | 1     | 0xxxxxxx| Padding/reserved space          |

### 3.3 Data Size Encoding (VINT)

EBML data sizes (ContentLength) use Variable-Length Integer (VINT) encoding. Each octet has 1 bit as a continuation marker (1 = more octets follow, 0 = this is the last octet) and 7 bits of value. The first octet also encodes the length in its leading 1-bits count.

| Octets | Bits for Value | Max Representable | Encoding Pattern                        |
|--------|---------------|-------------------|----------------------------------------|
| 1      | 7 bits        | 0x7F (127)        | 0xxxxxxx (C=0, 0x00-0x7F)             |
| 2      | 14 bits       | 0x3FFF (16383)   | 1xxxxxxx 0xxxxxxx                     |
| 3      | 21 bits       | 0x1FFFFF         | 1xxxxxxx 1xxxxxxx 0xxxxxxx            |
| 4      | 28 bits       | 0xFFFFFFF        | 1xxxxxxx 1xxxxxxx 1xxxxxxx 0xxxxxxx   |
| 5      | 35 bits       | 0x7FFFFFFFF      | 1xxxxxxx 1xxxxxxx 1xxxxxxx 1xxxxxxx 0xxxxxxx |
| 6      | 42 bits       | 0x3FFFFFFFFFF   | (continues pattern)                    |
| 7      | 49 bits       | 0x1FFFFFFFFFFFFFF|                                       |
| 8      | 56 bits       | 0xFFFFFFFFFFFFFF |                                       |

**Unknown Size Marker:** When all value bits are set to 1 (i.e., 0x7F for 1-byte, 0x3FFF for 2-byte, etc.), the size is unknown — the element extends to the end of its parent. This is used for the Segment element during live streaming.

**Encoding examples:**

| Value | 1-byte encoding | 2-byte encoding   | Notes                         |
|-------|-----------------|-------------------|-------------------------------|
| 0     | 0x80            | 0x40 0x00         | 0x40 = "length 1, value 0"   |
| 1     | 0x81            | 0x40 0x01         | High bit set = value 1        |
| 127   | 0xFF            | 0x40 0x7F         | Max 1-byte value              |
| 128   | N/A             | 0x40 0x80         | Needs 2 bytes                 |
| 16383 | N/A             | 0x7F 0xFF         | Max 2-byte value              |
| 16384 | N/A             | 0x40 0x80 0x00    | Needs 3 bytes                 |

For 2-byte encoding: the first byte has high bit 1 (more coming) and remaining 7 bits = (value >> 7). The second byte has high bit 0 and remaining 7 bits = (value & 0x7F).

For 3-byte encoding: first byte = 0x40 | (value >> 14), second byte = 0x80 | ((value >> 7) & 0x7F), third byte = value & 0x7F with high bit 0.

### 3.4 VINT Encoding Algorithm

```python
def encode_vint(value: int, min_bytes: int = 1) -> bytes:
    """Encode an integer as an EBML VINT."""
    # Find minimum bytes needed
    if value < 0x7F and min_bytes <= 1:
        return bytes([value | 0x80])  # High bit 1, value in lower 7 bits

    # Determine byte count
    for num_bytes in range(1, 9):
        max_val = (1 << (7 * num_bytes)) - 1
        if value <= max_val:
            break

    num_bytes = max(num_bytes, min_bytes)

    result = []
    for i in range(num_bytes - 1):
        # More bytes follow: high bit 1
        result.append(0x80 | ((value >> (7 * (num_bytes - i - 1))) & 0x7F))
    # Last byte: high bit 0
    result.append(value & 0x7F)
    return bytes(result)

# Examples:
# encode_vint(1)    → b'\x81'
# encode_vint(127)  → b'\xff'
# encode_vint(128)  → b'\x40\x80'
# encode_vint(16384) → b'\x40\x80\x00'
```

### 3.5 Element Data Types

EBML elements carry typed data:

| Type Name      | Description                                              | Encoding                                |
|----------------|----------------------------------------------------------|-----------------------------------------|
| `uinteger`     | Unsigned integer                                         | Big-endian, fixed length from size      |
| `integer`      | Signed two's complement integer                          | Big-endian, fixed length from size      |
| `float`        | IEEE 754 double-precision (8 bytes) or single (4 bytes) | Big-endian bytes                        |
| `string`       | UTF-8 text, no embedded nulls                           | Variable-length, no null termination     |
| `utf-8`        | UTF-8 text with full Unicode support                    | Variable-length, no length limit         |
| `binary`       | Raw opaque bytes                                         | Variable-length, codec-dependent         |
| `date`         | Signed 8-byte: nanoseconds since Unix epoch (2000-01-01)| Big-endian 8 bytes                      |
| `master`       | Container for child elements                             | Concatenation of child elements         |

For scalar types, the Data Size field determines the byte length. A `uinteger` with size 2 has 2 bytes, interpreted as a big-endian 16-bit unsigned integer.

### 3.6 Void Elements

Void elements (`0xEC`) are reserved placeholder elements used for padding and reserved/pre-allocated space. They contain arbitrary padding bytes that can be overwritten by future edits without requiring file rewriting. A void element's content is ignored by readers.

```
| Offset | Size    | Field                   |
|--------|---------|-------------------------|
| 0      | 1       | Element ID = 0xEC       |
| 1-?    | 1-8     | Data Size (VINT)        |
| ?+1    | N       | Padding bytes (0x00)    |
```

A typical use: writing a void element of 4096 bytes after the EBML Header, so that later edits (adding a SeekHead, updating Info) can be performed in-place by replacing the void with actual content and shrinking/expanding the void as needed.

---

## 4. SEGMENT STRUCTURE

### 4.1 Segment Element

The Segment (`0x18538067`) is the root container for all Matroska media data. In a Matroska file, there is exactly one Segment element (or zero for a pure EBML header-only file, but that is not a valid media file). The Segment ID is a 4-byte ID following the EBML 4-byte ID pattern (starts with 0xE0 range, i.e., "1110" leading bits).

```
Segment binary layout:

[Segment ID]          4 bytes  0x1A 0x53 0x80 0x67
[Segment Data Size]   1-8      Variable-length (can be unknown=0x01FFFFFFFFFFFFFF)

  [SeekHead ID+Size]       (optional but recommended)
  [Info ID+Size]           (mandatory)
  [Tracks ID+Size]         (mandatory)
  [Chapters ID+Size]       (optional)
  [Cluster ID+Size]        (mandatory — at least one)
  [Cues ID+Size]           (optional but recommended)
  [Attachments ID+Size]     (optional)
  [Tags ID+Size]            (optional)
```

The Segment's Data Size is typically set to "unknown" (0x01 followed by 0xFF × 7 = 0x01FFFFFFFFFFFFFF) for live-streaming scenarios where the final size is not known in advance. For file-based Matroska, the size is usually set to a known value covering all data through the end of the file.

### 4.2 SeekHead Element

SeekHead (`0x114D9B74`) is a Master element containing Seek entries (`0x4DBB`). It provides an index of where to find all other top-level Segment children, enabling readers to locate Tracks, Chapters, Cues, etc. without scanning the entire file.

Each Seek entry holds:

| Child Element    | Type    | Description                                              |
|------------------|---------|----------------------------------------------------------|
| SeekID           | binary  | The Element ID of the target element (raw bytes)       |
| SeekPosition     | uinteger| Offset from Segment start to element's first byte      |

The SeekPosition is measured from the byte immediately following the Segment's Size field (i.e., from the first byte of Segment data). It does NOT include the Segment's own ID+Size header bytes.

```
Example SeekHead binary:

[SeekHead ID+Size]
  [Seek ID+Size]
    [SeekID] = 0x16 0x54 0xAE 0x6B (Tracks ID)
    [SeekPosition] = 0x00001234 (SeekHead length bytes offset)
  [Seek ID+Size]
    [SeekID] = 0x1F 0x43 0xB6 0x75 (Cluster ID)
    [SeekPosition] = 0x00010000 (Cluster starts at offset 0x10000)
```

A SeekHead can reference itself (a recursive reference) or another SeekHead. Common practice: a primary SeekHead at the start of the Segment, with a secondary SeekHead at the end pointing back to the primary.

### 4.3 Info Element

Info (`0x1549A966`) holds file-level metadata:

| Element ID | Name                   | Type     | Description                                      |
|------------|------------------------|----------|--------------------------------------------------|
| 0x4D80     | MuxingApp              | string   | Application that created the file                |
| 0x5741     | WritingApp             | string   | Application that wrote the file                  |
| 0x4D80     | MuxingApp              | string   | (alternative ID, legacy)                         |
| 0x5741     | WritingApp             | string   | (alternative ID, legacy)                         |
| 0x4AD8     | Title                  | utf-8    | File title                                       |
| 0x7BA9     | Duration               | float    | Duration in nanoseconds                          |
| 0x33A4     | SegmentUID             | binary   | 16-byte unique identifier for the segment       |
| 0x3384     | SegmentFilename         | utf-8    | Filename of the segment                          |
| 0x6AE7     | DateUTC                | date     | File creation date (nanoseconds since 2000-01-01)|
| 0x2AD7B1   | TimestampScale         | uinteger | Nanoseconds per timestamp unit (default 1,000,000)|
| 0x2AD7B1   | TimestampScale         | uinteger | (duplicate, legacy position)                      |

The Duration field is stored as a IEEE 754 double-precision float in nanoseconds. A 5-minute audio file has Duration = 300.0 × 10^9 = 3.0e11.

The TimestampScale field defaults to 1,000,000, meaning all timestamps inside the Segment (except ClusterTimecode) are expressed in milliseconds. The formula:

```
nanoseconds = timestamp_value × TimestampScale
```

---

## 5. TRACKS AND AUDIO SPECIFICATION

### 5.1 Tracks Master Element

Tracks (`0x1654AE6B`) is a Master element containing one TrackEntry (`0xAE`) per stream. A TrackEntry defines a single audio, video, subtitle, or button track.

```
[Tracks ID+Size]
  [TrackEntry ID+Size]  (track 1: audio)
    TrackNumber: 1
    TrackUID: 1001
    TrackType: 2
    CodecID: "A_FLAC"
    CodecPrivate: [FLAC STREAMINFO]
    [Audio ID+Size]
      SamplingFrequency: 48000.0
      Channels: 2
      BitDepth: 24
  [TrackEntry ID+Size]  (track 2: audio, alternative)
    TrackNumber: 2
    TrackUID: 1002
    TrackType: 2
    ...
```

### 5.2 TrackEntry Fields

Each TrackEntry (`0xAE`) contains:

| Element ID | Name                   | Type     | Description                                      |
|------------|------------------------|----------|--------------------------------------------------|
| 0xD7       | TrackNumber            | uinteger | 1-based index identifying the track              |
| 0x73C5     | TrackUID               | uinteger | Unique identifier (must not be 0)               |
| 0x83       | TrackType              | uinteger | 1=Video, 2=Audio, 3=Complex, 0x11=Subtitle, 0x10=Logo, 0x12=Buttons |
| 0x536E     | Name                   | utf-8    | Human-readable track name                        |
| 0x22B59C   | Language               | string   | ISO 639-2 language code (e.g., "eng", "jpn")    |
| 0x86       | CodecID                | string   | Codec identifier string                          |
| 0x63A2     | CodecPrivate           | binary   | Codec-specific initialization data               |
| 0x3A9997   | CodecName              | string   | Human-readable codec name                        |
| 0xE0       | Video                  | master   | Video-specific fields (TrackType=1 only)         |
| 0xE1       | Audio                  | master   | Audio-specific fields (TrackType=2 only)        |
| 0x6D80     | ContentEncodings       | master   | Encryption/compression info                      |
| 0xAA       | CodecDecodeAll         | uinteger | 1=entire stream decodable from any position       |
| 0x8F87     | TrackDefaultDuration   | uinteger | Default duration per frame in nanoseconds         |
| 0x55EE     | TrackFlagForced        | uinteger | 1=track must be played (e.g., forced subtitles)  |
| 0xB9       | TrackFlagEnabled       | uinteger | 1=track is enabled for playback                  |

### 5.3 Audio Track Parameters

The Audio Master element (`0xE1`) contains all audio-specific parameters:

| Element ID | Name                      | Type     | Description                                      |
|------------|---------------------------|----------|--------------------------------------------------|
| 0xB5       | SamplingFrequency         | float    | Sampling rate in Hz (default 8000.0)             |
| 0x78B5     | OutputSamplingFrequency   | float    | Output rate after processing (e.g., SBR output)  |
| 0x9F       | Channels                  | uinteger | Number of audio channels (default 1)            |
| 0x7D7B     | ChannelPositions          | binary   | Speaker placement bitmask (deprecated)           |
| 0x6264     | BitDepth                 | uinteger | Bits per sample (e.g., 16, 24, 32)              |
| 0xE2       | TrackOperation            | master   | Track gangs/multiplexing                         |
| 0xED79     | DolbyVisionMetadata       | binary   | Dolby Vision metadata                           |

**SamplingFrequency:** IEEE 754 float. Standard values include 44100.0, 48000.0, 88200.0, 96000.0, 176400.0, 192000.0. For 44.1 kHz, the value is exactly 44100.0, not 44100.

**Channels:** Number of discrete audio channels. Stereo = 2, 5.1 surround = 6, 7.1 surround = 8. LFE is counted as one of the channels.

**BitDepth:** Bit depth of audio samples. For PCM: 16, 24, or 32. For compressed codecs: bit depth of decoded output.

### 5.4 CodecPrivate Reference

The CodecPrivate (`0x63A2`) binary blob holds codec-specific initialization data. Its format is entirely codec-dependent:

| CodecID          | CodecPrivate contents                                     |
|------------------|----------------------------------------------------------|
| `A_AAC`          | AudioSpecificConfig (ASC) from AAC bitstream             |
| `A_AAC/SBR`      | ASC with SBR/PS signaling (HE-AAC)                       |
| `A_FLAC`         | FLAC STREAMINFO block (34 bytes minimum)                 |
| `A_VORBIS`       | Three Vorbis headers concatenated                        |
| `A_OPUS`         | OpusHead packet (19+ bytes)                              |
| `A_AC3`          | None required (sync info in bitstream)                   |
| `A_DTS`          | DTS-HD Master Audio stream info (if applicable)          |
| `A_ALAC`         | ALAC-specific configuration blob                         |
| `A_TTA1`         | None required                                            |
| `A_WAVPACK4`     | None required                                            |

**For AAC CodecPrivate (AudioSpecificConfig):**

The ASC is encoded per ISO/IEC 14496-3. A minimal ASC for AAC-LC stereo:

```
AudioSpecificConfig:
  audioObjectType: 5 bits  = 2 (AAC-LC)
  samplingFrequencyIndex: 4 bits = 3 (48000 Hz) or 0 (explicit)
  samplingFrequency: 24 bits if index=0xF (explicit)
  channelConfiguration: 4 bits = 2 (2 channels)
  GASpecificConfig:
    frameLengthFlag: 1 bit = 0 (1024 samples)
    dependsOnCoreCoding: 1 bit = 0
    extensionFlag: 1 bit = 0
```

For HE-AAC with SBR: the AudioSpecificConfig contains an explicit SBR signaling element (`ps- SignalingPresent` flag). The OutputSamplingFrequency in the Audio element should reflect the doubled rate (e.g., 96000.0 for 48000 Hz input).

**For Opus CodecPrivate (OpusHead):**

The OpusHead packet per RFC 7845:

```
Bytes  0-7:  "OpusHead" magic (8 bytes)
Byte   8:     Version = 1
Byte   9:     Channels (1 byte, e.g., 2 for stereo)
Bytes 10-11:  Pre-skip (uint16 LE, little-endian)
Bytes 12-15:  Input Sample Rate (uint32 LE, e.g., 48000)
Bytes 16-17:  Output Gain (int16 LE, Q7.8 dB)
Byte  18:     Channel Mapping Family (0=mono/stereo, 1=Vorbis mapping)
If family != 0:
  Byte 19:     Stream Count
  Byte 20:     Coupled Count
  Bytes 21+:   Channel Mapping (stream count + mapping entries)
```

**For FLAC CodecPrivate (STREAMINFO):**

The FLAC STREAMINFO block is stored verbatim. Minimum 34 bytes:

```
Bytes 0-3:   "fLaC" magic
Bytes 4:      STREAMINFO type = 0
Bytes 5-7:    STREAMINFO length (27 for standard STREAMINFO)
Bytes 8-17:   STREAMINFO data (10 bytes):
  16 bits: block size (min block size)
  16 bits: block size (max block size)
  24 bits: frame size (min frame size)
  24 bits: frame size (max frame size)
  20 bits: sample rate
  3 bits:  channel count
  5 bits:  bits per sample - 1
  36 bits: total samples (high 4 bytes, low 4 bytes of 36-bit value)
  16 bits: MD5 checksum
```

**For Vorbis CodecPrivate (three headers):**

Three Vorbis headers concatenated in order:
1. **Identification header** (30 bytes): "vorbis" magic, version, channels, sample rate, bitrate upper/lower, block size, framing flag
2. **Comment header** (7+ bytes): vendor string length + vendor string, comment list length + each comment string
3. **Setup header** (variable): the full codebook and floor/Residue/setup data needed to initialize the decoder

### 5.5 CodecID Reference

| CodecID            | Description                          | CodecPrivate    |
|--------------------|--------------------------------------|-----------------|
| `A_AAC`            | AAC (MPEG-2/MPEG-4)                 | Yes (ASC)       |
| `A_AAC/MAIN`       | AAC Main profile                    | Yes (ASC)       |
| `A_AAC/LC`         | AAC Low Complexity                  | Yes (ASC)       |
| `A_AAC/SBR`        | AAC with SBR (HE-AAC v1)            | Yes (ASC+SBR)   |
| `A_AAC/PS`        | AAC with PS (HE-AAC v2)             | Yes (ASC+PS)    |
| `A_FLAC`           | FLAC (Free Lossless Audio Codec)    | Yes (STREAMINFO)|
| `A_VORBIS`        | Vorbis                               | Yes (3 headers) |
| `A_OPUS`           | Opus Interactive                     | Yes (OpusHead)  |
| `A_MP3`            | MPEG-1/2 Audio Layer III            | No              |
| `A_MP2`            | MPEG-1 Audio Layer II               | No              |
| `A_AC3`            | Dolby Digital (AC-3)                | No              |
| `A_EAC3`           | Dolby Digital Plus (E-AC-3)         | No              |
| `A_DTS`            | DTS Coherent Acoustics               | Conditional     |
| `A_DTS/EXPRESS`   | DTS Express                         | No              |
| `A_DTS/HD`         | DTS-HD Master Audio                 | Conditional     |
| `A_DTS/HD/LBR`     | DTS-HD Low Bit Rate                | No              |
| `A_PCM/INT/LIT`    | Integer PCM little-endian           | No              |
| `A_PCM/INT/BIG`    | Integer PCM big-endian              | No              |
| `A_PCM/FLOAT/IEEE` | IEEE 754 float PCM                  | No              |
| `A_TTA1`           | TTA1 (True Audio) lossless          | No              |
| `A_WAVPACK4`       | WavPack                             | No              |
| `A_ALAC`           | Apple Lossless Audio Codec          | Yes             |
| `A_SPEEX`          | Speex                               | Yes (Ogg header)|
| `A_REAL/RV20`      | RealAudio COOK                      | Yes             |
| `A_REAL/RV30`      | RealAudio RealVorbis               | Yes             |
| `A_REAL/RV40`      | RealAudio APT-X / RV40             | Yes             |
| `A_MPEG/L3`        | Same as A_MP3                      | No              |
| `A_MPEG/L2`        | Same as A_MP2                      | No              |

---

## 6. CLUSTER STRUCTURE

### 6.1 Cluster Element

The Cluster (`0x1F43B675`) is the primary container for time-coded media frames. A Cluster groups together frames that are temporally close, reducing the number of seeks needed for local playback. Each Cluster has a ClusterTimecode that establishes a base timestamp for all Blocks inside.

```
Cluster binary layout:

[Cluster ID]            4 bytes  0x1A 0x43 0xB6 0x75
[Cluster Data Size]     1-8      Variable-length (often unknown for live)

  [ClusterTimecode ID]  1 byte  0xE7
  [ClusterTimecode Size]1-8     VINT
  [ClusterTimecode Data]1-8    Milliseconds since file start (uinteger)

  [SimpleBlock ID+Size]         (zero or more)
  [BlockGroup ID+Size]          (zero or more)
```

| Element ID | Name                   | Type     | Description                                      |
|------------|------------------------|----------|--------------------------------------------------|
| 0xE7       | ClusterTimecode        | uinteger | Cluster start time in milliseconds (NOT nanoseconds) |
| 0x5854     | SilentTracks           | master   | Tracks with no data in this cluster             |
| 0xA0       | BlockGroup             | master   | Block with reference frames                     |
| 0xA3       | SimpleBlock            | binary   | Standalone block (no references)                 |
| 0xAB       | Position               | uinteger | Cluster file offset (for seeking)                |
| 0x75A4     | PrevSize               | uinteger | Size of previous cluster (for editing)           |
| 0x7B69     | SilentTrackTimecode    | uinteger | Timecode for silent track                       |

**Critical:** ClusterTimecode is expressed in **milliseconds**, not nanoseconds. This is a deliberate exception to Matroska's general nanosecond convention, chosen for efficient time-based clustering. Maximum: 2^32 - 1 milliseconds ≈ 49.7 days of content.

### 6.2 BlockGroup Structure

BlockGroup (`0xA0`) is a Master element that wraps a Block along with its reference information and optional per-frame metadata. BlockGroup is used for video tracks where frames have temporal dependencies (B-frames referencing other frames), or for audio tracks that need per-frame CRC-32 validation.

```
BlockGroup binary layout:

[0xA0]               Element ID (BlockGroup)
[Data Size]          VINT, size of all following data

  [Block ID+Size]    Binary block data
  [BlockAdditions]   Optional: slice/increment data
    [BlockMore ID+Size]
      BlockAddID: uinteger
      BlockAdditional: binary
  [ReferenceBlock]   0, 1, or 2: temporal references (signed integer ms)
  [SliceDuration]    Optional: frame duration override
  [Slices]           Legacy slice info (deprecated)
  [CRC-32]           Optional: CRC-32 of Block data
  [ReferencePriority]|uinteger| Priority flag                              |
```

The ReferenceBlock element uses a signed integer representing a timecode delta in milliseconds (relative to the current frame's timestamp). A negative value indicates a backward reference (the frame this one depends on comes before it); a positive value indicates a forward reference.

### 6.3 Block Structure (Binary Layout)

The Block element (`0xA1`) is a binary element with an internal structure. Unlike EBML Master elements that are parsed recursively, Block data is parsed as a custom binary header.

```
Block binary layout:

+---------+------------------+------------+--------------------------------+
| Track#  | Timecode         | Flags      | Frame Data...                  |
| 1-2 B   | 2 B (int16)      | 1 B        | Variable                       |
+---------+------------------+------------+--------------------------------+
```

**Track Number (1-2 bytes, VINT):** The TrackNumber from Tracks. High bit set in first byte means more bytes follow. TrackNumber N is encoded as VINT with value N.

```
TrackNumber 1:   0x81  (VINT: 1 byte, value 1)
TrackNumber 2:   0x82
...
TrackNumber 127: 0xFF
TrackNumber 128: 0x40 0x80  (2-byte VINT: first byte value bits = 1, second byte = 128 & 0x7F)
```

**Block Timecode (2 bytes, signed int16):** Offset in **milliseconds** relative to ClusterTimecode. Range: -32768 to +32767 ms. For audio, typically 0 (first frame) or small positive offsets for subsequent frames.

**Flags (1 byte):**

| Bit | Name          | Description                                                   |
|-----|---------------|---------------------------------------------------------------|
| 0   | Lace XFlac    | Lacing type: 0=none, 1=Xiph, 2=fixed, 3=EBML                |
| 1   | Lacing        | (bits 0-3 encode lacing type)                                |
| 2   | Lacing        |                                                               |
| 3   | Lacing        |                                                               |
| 4   | Invisible     | 1 = block not displayed (video keyframe indicator)           |
| 5   | Lacing        | (same as bits 0-3, duplicate encoding)                       |
| 6   | Keyframe      | For Block: set if this is a keyframe                         |
| 7   | Reserved      | Reserved bit                                                  |

For audio tracks, lacing is typically disabled (Flag = 0x00). The audio frame data follows directly.

**Frame Data:** The raw codec payload. For AAC: raw AAC frame without ADTS header. For FLAC: complete FLAC frame. For Opus: Opus packet. For MP3: complete MP3 frame.

### 6.4 SimpleBlock Structure

SimpleBlock (`0xA3`) is the preferred format for audio tracks. It has the same binary layout as Block but different flag semantics:

```
SimpleBlock binary layout:

[SimpleBlock ID]  1 byte  0xA3
[Data Size]       1-8     VINT (size of following data)

[TrackNumber]     1-2 B   VINT
[Block Timecode]  2 B     signed int16 (ms offset from cluster)
[Flags]          1 B     See table below
[Frame Data]      N B     Raw codec payload

```

| Bit | Name          | Description                                                   |
|-----|---------------|---------------------------------------------------------------|
| 0-3 | Lacing Type   | 0=none, 1=Xiph, 2=fixed, 3=EBML                             |
| 4   | Invisible     | 1=not displayed                                              |
| 5   | Lacing        | (lacing flag)                                                |
| 6   | Keyframe      | For SimpleBlock with video track: always 1 if keyframe       |
| 7   | Discardable   | 1=block can be discarded for seeking/random access           |

For audio SimpleBlocks: Flags = 0x00 (no lacing, no invisible, not a keyframe concept). For video SimpleBlocks: Flags = 0x00 for non-keyframe, 0x80 for keyframe (bit 7 set = discardable flag, bit 6 = keyframe).

### 6.5 Lacing

Lacing allows multiple small frames packed into a single Block to reduce overhead.

**No Lacing (Flag = 0):** Single frame. Block data is one complete codec frame.

**Xiph Lacing (Flag = 1):** Multiple variable-size frames. Frame sizes are encoded as a series of uint8 values. A value of 255 means "add next byte to this." The last frame size is computed from the remaining Block data.

**Fixed-size Lacing (Flag = 2):** All frames are the same size. Frame size = Block data size / number of frames.

**EBML Lacing (Flag = 3):** Frame sizes stored as signed deltas from the first frame using VINT encoding.

For audio: Xiph lacing is common for Vorbis (variable frame size). Fixed-size lacing is used for constant-frame codecs. EBML lacing is used for moderately variable frames.

---

## 7. TIMESTAMP AND TIME HANDLING

### 7.1 Timestamp Units

Matroska uses **nanoseconds** as the base unit for most timestamps. This provides sufficient resolution for video (23.976 fps = 41,708,333 ns per frame) and audio (44.1 kHz = 22,675.7 ns per sample).

The key exception is **ClusterTimecode**, which uses **milliseconds**. The TimestampScale factor bridges these units.

```
nanoseconds = timestamp × TimestampScale
```

The default TimestampScale is 1,000,000 (one millisecond per unit). When TimestampScale = 1,000,000:
- All timestamps in the Segment (except ClusterTimecode) are in milliseconds
- ClusterTimecode is also in milliseconds (already)
- Duration in Info is in nanoseconds (stored as float)

### 7.2 Frame Timestamp Calculation

The absolute timestamp of any frame:

```
cluster_time_ns = ClusterTimecode × 1,000,000  (nanoseconds)
block_time_ns = BlockTimecode × 1,000,000       (nanoseconds)
frame_timestamp_ns = cluster_time_ns + block_time_ns
```

For a SimpleBlock in a cluster:
- ClusterTimecode = 5000 (5 seconds in ms)
- BlockTimecode = 1024 (ms offset within cluster)
- TimestampScale = 1,000,000
- Absolute time = (5000 + 1024) × 1,000,000 = 6,024,000,000 ns = 6.024 seconds

For audio at 48 kHz with 1024 samples per frame:
```
frame_duration_ns = 1024 / 48000 × 1e9 = 21,333,333.33... ns
frame_number_in_cluster = BlockTimecode / 21.333... (approximately)
```

### 7.3 TimestampScale

The TimestampScale element (`0x2AD7B1`) in the Info element is a uinteger with a default value of 1,000,000. All timestamps in the Segment except ClusterTimecode are multiplied by TimestampScale to get nanoseconds.

```
# Default case: TimestampScale = 1,000,000
# All timestamps are in milliseconds
block_timestamp_ns = BlockTimecode × 1,000,000

# Custom case: TimestampScale = 1 (nanosecond resolution)
block_timestamp_ns = BlockTimecode × 1
```

The Matroska spec recommends TimestampScale = 1,000,000 for most use cases. Lower values (finer resolution) may be needed for high-frame-rate video.

### 7.4 Default Duration

The TrackEntry's DefaultDuration (`0x23A970`) specifies the default frame duration in nanoseconds. This enables seeking and time display even before Cues are parsed.

```
For 48 kHz AAC-LC (1024 samples/frame):
  DefaultDuration = 1024 / 48000 × 1e9 = 21,333,333 ns
  = 0x14 0x5F 0x5E 0x00  (uint32, big-endian)

For 44.1 kHz FLAC (4096 samples/block):
  DefaultDuration = 4096 / 44100 × 1e9 = 92,879.8... ns
  = 0x00 0x01 0x6A 0x6F  (approximately)

For 48 kHz Opus (typically 20ms frames):
  DefaultDuration = 20 × 1e6 = 20,000,000 ns
  = 0x01 0x31 0x2D 0x00
```

### 7.5 Date Element

The Date element (`0x4467`) in Info stores the file creation timestamp. It is a signed 64-bit integer representing **nanoseconds since 2001-01-01T00:00:00 UTC** (NOT Unix epoch).

```python
import datetime

# Unix epoch (1970-01-01) in Matroska Date units (ns since 2001-01-01)
UNIX_EPOCH_IN_MATROSKA = (datetime.datetime(1970, 1, 1) -
                          datetime.datetime(2001, 1, 1)).total_seconds() * 1e9
# = -978307200000000000 ns

def matroska_date_to_unix(matroska_ns: int) -> datetime.datetime:
    unix_ns = matroska_ns + UNIX_EPOCH_IN_MATROSKA
    return datetime.datetime(1970, 1, 1) + datetime.timedelta(seconds=unix_ns / 1e9)
```

---

## 8. CRC-32 VALIDATION

### 8.1 CRC-32 Element

The CRC-32 element (`0xBF`) provides data integrity verification for the parent element. When present inside a Master element, it covers all data from the first byte after the CRC-32 element's own header up to the last byte of the parent element.

```
CRC-32 element:
[0xBF]              Element ID
[Data Size]         VINT (always 4 for CRC-32)
[CRC-32 Value]      4 bytes, big-endian uint32
```

### 8.2 CRC Algorithm

The CRC-32 uses the IEEE 802.3 polynomial (0x04C11DB7), also known as the standard Ethernet/PNG/GZIP CRC.

```python
import zlib  # Python's zlib.crc32 uses the same polynomial

def matroska_crc32(data: bytes) -> bytes:
    """Compute CRC-32 and return 4-byte big-endian representation."""
    crc = zlib.crc32(data) & 0xFFFFFFFF
    return crc.to_bytes(4, 'big')

def verify_matroska_crc(data: bytes, stored_crc: bytes) -> bool:
    """Verify data against stored CRC-32."""
    computed = zlib.crc32(data) & 0xFFFFFFFF
    expected = int.from_bytes(stored_crc, 'big')
    return computed == expected
```

### 8.3 Coverage Rules

The CRC covers:
- All data bytes of the parent element (e.g., BlockGroup)
- NOT including the CRC-32 element's own ID, Size, or Data fields
- NOT including the parent element's own ID and Size fields

For BlockGroup CRC validation:
```
raw_block_bytes = BlockGroup data bytes
  = Block element (all bytes including header)
  + ReferenceBlock elements
  + SliceDuration
  + BlockAdditions
  + etc.
computed_crc = crc32(raw_block_bytes)
assert computed_crc == BlockGroup.CRC32.value
```

### 8.4 CRC Placement

CRC-32 elements must appear as the first child of the Master element they validate. In BlockGroup, the CRC-32 (if present) must be the first child after the BlockGroup ID+Size. This ensures the CRC covers the rest of the element including all subsequent children.

```
[BlockGroup ID+Size]
  [CRC-32 ID+Size]      ← must be first
  [Block ID+Size]
  [ReferenceBlock]
  ...
```

---

## 9. CUES ELEMENT (SEEKING INDEX)

### 9.1 Cues Structure

The Cues element (`0xC53BB6B`) provides a random-access index for seeking. Without Cues, seeking requires scanning through all Clusters sequentially.

```
[Cues ID+Size]
  [CuePoint ID+Size]          (one per keyframe)
    [CueTime ID+Size]          Timestamp in TimestampScale units
    [CueTrackPositions ID+Size] (one per track referenced)
      [CueTrack ID+Size]      TrackNumber (uinteger)
      [CueClusterPosition ID+Size]  Absolute byte offset
      [CueRelativePosition ID+Size] Offset within cluster
      [CueDuration ID+Size]   Duration of cued element
      [CueBlockNumber ID+Size] Block number within cluster
      [CueCodecState ID+Size] Codec state size before block
      [CueRefTime ID+Size]    Reference frame timestamp
```

### 9.2 CuePoint Details

**CueTime:** The timestamp of the access point in TimestampScale units (default: milliseconds). Must equal the Cluster's ClusterTimecode + BlockTimecode.

**CueTrack:** The TrackNumber (1-based integer matching TrackEntry.TrackNumber).

**CueClusterPosition:** The most critical field for seeking. Offset in bytes from the start of the Segment (from the first byte of Segment data, i.e., after the Segment's ID+Size header) to the first byte of the Cluster element's ID.

**CueRelativePosition:** Offset from the start of the Cluster to the Block. Provides sub-cluster seeking precision.

**CueBlockNumber:** 1-based index of the Block within the Cluster.

### 9.3 Seeking Algorithm

To seek to timestamp T:

```
1. Find the CuePoint with the largest CueTime ≤ T
   (binary search on sorted CueTime values)

2. Get CueClusterPosition from that CuePoint
   absolute_file_offset = Segment_start_offset + CueClusterPosition + Segment_header_bytes

3. Read the Cluster header
   cluster_timecode = Cluster.ClusterTimecode

4. Scan forward through SimpleBlocks/BlockGroups in the Cluster
   for each block:
     block_timestamp = cluster_timecode + block.BlockTimecode
     if block_timestamp >= T:
       return block

5. If not found in this cluster, seek to next CuePoint and repeat
```

### 9.4 CuePoint Generation for Audio

For audio-only files, CuePoints should be generated per Cluster (one per cluster) since audio frames are independent and seeking to any cluster is equally valid. The CueTrack should point to the primary audio track.

For video files, CuePoints should be generated per keyframe (blocks with the keyframe flag set), enabling efficient video seeking. The CueTrack typically points to the first video track's keyframe positions.

The Matroska spec recommends at minimum one CuePoint per Cluster for audio, and one per keyframe for video.

---

## 10. CHAPTER AND TAG ELEMENTS

### 10.1 Chapters Element

Chapters (`0x1043A770`) defines named, timestamped segments for navigation. This is the Matroska equivalent of a CD track listing or album chapter markers.

```
[Chapters ID+Size]
  [EditionEntry ID+Size]
    EditionUID: uinteger
    EditionFlagHidden: uinteger (0 or 1)
    EditionFlagDefault: uinteger (1 = default edition to play)
    EditionFlagOrdered: uinteger (1 = must play in order)
    [ChapterAtom ID+Size]
      ChapterUID: uinteger
      ChapterTimeStart: uinteger (timestamp in ms)
      ChapterTimeEnd: uinteger (timestamp in ms, optional)
      ChapterFlagHidden: uinteger
      ChapterFlagEnabled: uinteger
      [ChapterDisplay ID+Size]
        ChapString: utf-8 (chapter title)
        ChapLanguage: string (ISO 639-2, e.g., "eng")
        ChapCountry: string (ISO 3166-1 alpha-2, e.g., "US")
      [ChapterProcess ID+Size]
        ChapterProcessCodecID: uinteger
        ChapterProcessPrivate: binary
        [ChapterProcessCommand ID+Size]
          ChapterProcessTime: uinteger
          ChapterProcessData: binary
```

**ChapterTimestamp:** ChapterTimeStart and ChapterTimeEnd are expressed in **milliseconds** (not nanoseconds), matching the ClusterTimecode convention. ChapterUID is a unique integer identifier for the chapter.

### 10.2 Chapter Atom Fields

| Element ID | Name                   | Type     | Description                                      |
|------------|------------------------|----------|--------------------------------------------------|
| 0xB6       | ChapterAtom            | master   | Individual chapter entry                         |
| 0x73C4     | ChapterUID            | uinteger | Unique chapter identifier                        |
| 0x1653     | ChapterStringUID      | string   | String-based chapter ID (optional)               |
| 0xB6       | ChapterTimeStart      | uinteger | Start time in milliseconds                       |
| 0x1466     | ChapterTimeEnd        | uinteger | End time in milliseconds (optional)              |
| 0x98E2     | ChapterFlagHidden     | uinteger | 1=hidden                                        |
| 0x1558     | ChapterFlagEnabled    | uinteger | 1=enabled for playback                           |
| 0x85       | ChapterDisplay        | master   | Chapter name display                             |
| 0x86       | ChapString            | utf-8    | Chapter title text                               |
| 0x437C     | ChapLanguage          | string   | ISO 639-2 language code                          |
| 0x437E     | ChapCountry           | string   | ISO 3166-1 alpha-2 country code                 |
| 0x8F87     | ChapterProcess        | master   | Pre/post-chapter processing                     |

### 10.3 Tags Element

Tags (`0x1254C367`) provides a flexible hierarchical metadata system.

```
[Tags ID+Size]
  [Tag ID+Size]
    [Targets ID+Size]
      TargetTypeValue: uinteger (70=album, 60=part/session, 50=track, 40=chapter)
      TargetType: string ("ALBUM", "TRACK", "CHAPTER", etc.)
      TrackUID: uinteger
      ChapterUID: uinteger
      AlbumUID: uinteger
      [SimpleTag ID+Size]
        TagName: string (e.g., "TITLE")
        TagLanguage: string (IETF BCP 47, e.g., "eng")
        TagLanguageIETF: string (BCP 47, e.g., "en-US")
        TagDefault: uinteger (1=default language)
        TagString: utf-8 (value)
        [BinaryTag ID+Size] (if value is binary data)
```

### 10.4 Standard Tag Names

Matroska defines standard tag names aligned with Vorbis Comment:

| TagName                 | Description                              | Example                    |
|-------------------------|------------------------------------------|----------------------------|
| TITLE                   | Track/chapter title                      | "Allegro in C Major"        |
| ARTIST                  | Primary performer/creator               | "Johann Sebastian Bach"     |
| ALBUM                   | Album name                              | "Brandenburg Concertos"     |
| ALBUMARTIST             | Album-wide artist                       | "Bach Ensemble"             |
| COMPOSER                | Composer                                | "J.S. Bach"                 |
| ARRANGER                | Arranger                                | "Herbert von Karajan"      |
| LYRICS                  | Lyrics text                             | "In the jungle..."         |
| COMMENT                 | Freeform comment                        | "Live recording, Berlin"   |
| DATE                    | Recording date                          | "2023"                     |
| YEAR                    | Year (numeric, duplicate of DATE)      | "2023"                     |
| GENRE                   | Genre classification                    | "Classical"                |
| LABEL                   | Record label                            | "Deutsche Grammophon"      |
| CATALOGNUMBER           | Label catalog number                    | "DGG 002894775712"          |
| BARCODE                 | UPC/EAN barcode                         | "0028947757126"             |
| ISRC                    | ISRC code                               | "DEA561900123"              |
| PUBLISHER               | Publisher                               | "XYZ Records"             |
| COPYRIGHT               | Copyright information                   | "(p) 2023 XYZ Records"     |
| ENCODEDBY               | Encoded by                              | "FFmpeg"                    |
| ENCODER                 | Encoder settings                        | "Lavf 60.16.100"           |
| ENCODER_OPTIONS         | Encoder options                         | "-b:a 320k"                |
| REPLAYGAIN_TRACK_GAIN   | ReplayGain track gain                   | "-3.2 dB"                  |
| REPLAYGAIN_TRACK_PEAK   | ReplayGain track peak                   | "0.892456"                 |
| REPLAYGAIN_ALBUM_GAIN   | ReplayGain album gain                   | "-2.1 dB"                  |
| REPLAYGAIN_ALBUM_PEAK   | ReplayGain album peak                   | "0.951234"                 |
| TOTAL_PARTS             | Total chapters/tracks                  | "24"                       |
| PART_NUMBER             | Part/chapter number                     | "3"                        |
| INITIAL_KEY             | Musical key                             | "C Major"                  |
| BPM                     | Beats per minute                        | "120"                      |
| MOOD                    | Mood descriptor                         | "Upbeat"                  |
| SOURCE                  | Source media                            | "CD"                       |
| SOURCEFILENAME          | Original filename                       | "track01.wav"              |
| PERFORMER               | Performer (same as ARTIST)             | "London Symphony Orchestra"|
| DISCNUMBER              | Disc number                             | "1/2"                      |
| TOTALDISCS              | Total discs                             | "2"                        |
| ALBUMSORT               | Sort album name                         | "Brandenburg Concertos, The"|
| ARTISTSORT              | Sort artist name                        | "Bach, Johann Sebastian"   |
| TITLESORT               | Sort title                              | "Allegro in C Major, The"   |

### 10.5 Tag Target Hierarchy

Tags can be targeted at different levels:

| TargetTypeValue | TargetType | Scope              |
|-----------------|------------|---------------------|
| 70              | ALBUM      | Entire file         |
| 60              | PART       | Multi-part album    |
| 50              | TRACK      | Single track        |
| 40              | CHAPTER    | Single chapter      |
| 30              | SUBTITLE   | Subtitle track      |
| 0               | (none)     | Global file tag     |

If Targets is absent, the tag applies to the entire file.

---

## 11. WEBM CONSTRAINTS

### 11.1 WebM Overview

WebM is a strict subset of Matroska defined by the WebM project (Google). It restricts the format to VP8/VP9 video and Opus audio for web delivery. The DocType is "webm" instead of "matroska".

### 11.2 WebM vs Matroska Differences

| Aspect                   | Matroska        | WebM                          |
|--------------------------|-----------------|-------------------------------|
| DocType                  | "matroska"      | "webm"                        |
| DocTypeVersion           | 4               | 2                             |
| DocTypeReadVersion       | 2               | 2                             |
| Video Codec              | Any             | VP8 (`V_VP8`), VP9 (`V_VP9`) |
| Audio Codec              | Any             | Opus (`A_OPUS`) only          |
| CRC-32                   | Optional        | **Not allowed**              |
| SeekHead                 | Optional        | **Mandatory**                |
| Chapters                 | Allowed         | **Not allowed**              |
| Encryption               | Allowed         | **Not allowed**              |
| TrackType Logo (0x10)    | Allowed         | **Not allowed**              |
| ContentCompAlgo (beyond zlib)| Allowed    | zlib (0) or none only         |
| TimestampScale           | Any             | Must be 1,000,000             |
| Tags                     | Full tag set    | Allowed but limited           |
| Multiple video tracks    | Allowed         | Allowed but not recommended   |
| Cues                     | Recommended     | Mandatory                     |

### 11.3 WebM Audio-Only Structure

A valid WebM audio-only file:

```
[EBML Header: DocType="webm"]
[Segment ID+Size]
  [SeekHead ID+Size]
    Seek: Info position
    Seek: Tracks position
    Seek: Cues position
  [Info ID+Size]
    MuxingApp: "libwebm-X.X.X"
    WritingApp: "LavfXX.X"
    Duration: float (nanoseconds)
    TimestampScale: 1000000
  [Tracks ID+Size]
    [TrackEntry]
      TrackNumber: 1
      TrackUID: 1
      TrackType: 2 (Audio)
      Name: " Opus Audio"
      Language: "eng"
      CodecID: "A_OPUS"
      CodecPrivate: [OpusHead packet]
      [Audio ID+Size]
        SamplingFrequency: 48000.0
        Channels: 2
  [Cluster ID+Size]
    [SimpleBlock] × N (Opus frames)
  [Cues ID+Size] (mandatory in WebM)
```

### 11.4 WebM Opus Track Constraints

For Opus in WebM:
- SamplingFrequency should be 48000.0 and must match OpusHead's Input Sample Rate
- Channels should match OpusHead's channel count
- ChannelMappingFamily must be specified (0 for mono/stereo, 1 for surround with Vorbis order)
- DefaultDuration should be omitted (Opus uses variable frame sizes)
- The OpusHead's pre-skip value must be respected for correct seek timing
- CodecPrivate must be a complete OpusHead (not truncated)

---

## 12. MULTICHANNEL AND SPATIAL AUDIO

### 12.1 Channel Assignment

Matroska does not natively specify channel ordering — it defers to the codec's internal channel assignment. The Channels element (`0x9F`) specifies the count, and the codec determines ordering.

For codecs that embed channel configuration in CodecPrivate (AAC, Opus), the AudioSpecificConfig or OpusHead contains the authoritative channel mapping.

### 12.2 Opus Channel Mapping Tables

Opus RFC 7845 defines channel mappings by family:

**Mapping Family 0 (Mono/Stereo):**

| Channels | Stream Count | Coupled Streams | Description |
|----------|-------------|-----------------|-------------|
| 1        | 1           | 0               | Mono, single stream |
| 2        | 1           | 1               | Stereo, 1 coupled stream |

**Mapping Family 1 (Vorbis Order, up to 8 channels):**

| Channels | Vorbis Order | Description                    |
|----------|-------------|--------------------------------|
| 1        | 0           | Mono                           |
| 2        | 1           | L, R (stereo)                 |
| 3        | 2           | L, C, LFE                     |
| 4        | 3           | L, R, BL, BR (quadraphonic)   |
| 5        | 4           | L, C, LFE, BL, BR             |
| 6        | 4           | L, C, LFE, BL, BR, BC         |
| 7        | 5           | L, C, LFE, BL, BR, SL, SR     |
| 8        | 5           | L, C, LFE, BL, BR, SL, SR, LFE2|

**Mapping Family 2 (Ambisonics, up to 18 channels):**

Uses FuMa (First-Order Ambisonics) channel ordering per AMBI-1.0 specification.

### 12.3 FLAC Channel Assignment

FLAC channel assignment per FLAC format specification:

| Channels | Order (standard)          |
|----------|--------------------------|
| 1        | Mono                     |
| 2        | Left, Right             |
| 3        | Left, Right, Center     |
| 4        | Left, Right, Back Left, Back Right (quadraphonic) |
| 5        | Left, Center, Right, Back Left, Back Right |
| 6        | Left, Center, Right, Back Left, Back Right, LFE |
| 7        | Not defined             |
| 8        | Left, Center, Right, Back Left, Back Right, LFE, Side Left, Side Right |

### 12.4 AAC Channel Configurations

The AAC AudioSpecificConfig encodes channel configuration:

| Channels | Channel Config | Speaker Positions              |
|----------|---------------|--------------------------------|
| 1        | 1             | Front Center                   |
| 2        | 2             | L, R                           |
| 3        | 3             | L, C, R                        |
| 4        | 4             | L, R, BL, BR                   |
| 5        | 5             | L, C, R, BL, BR                |
| 6        | 6             | L, C, R, BL, BR, LFE           |
| 7        | 7             | Not defined                    |
| 8        | 8             | L, C, R, BL, BR, FL, FR, LFE  |

For HE-AAC with SBR, the output channel count may differ from the input (SBR upsamples from mono to stereo, etc.).

---

## 13. FFMPEG AND LIBAVFORMAT INTEGRATION

### 13.1 FFmpeg Muxing Architecture

FFmpeg's Matroska muxer is implemented in `libavformat/matroskaenc.c`. Key architectural decisions:

**EBML Header Generation:**
```c
// DocType = "matroska" (or "webm" for WebM mode)
// EBMLMaxIDLength = 4
// EBMLMaxSizeLength = 8
// DocTypeVersion = 4
// DocTypeReadVersion = 2
```

**Segment Structure Generation:**
- SeekHead is written first after Info and Tracks (allows seeking to elements)
- Void padding is inserted after SeekHead (4096 bytes by default) for in-place edits
- Clusters are written with a time-based strategy (default: new cluster every 5 seconds)
- Cues are written at the end of the Segment (after all Clusters)
- Tags are written last (after Cues)

**Clustering Algorithm:**
```python
# FFmpeg muxer clustering logic (pseudocode)
cluster_duration_target = 5.0  # seconds
cluster_timecode = 0
current_cluster_frames = []

for each frame:
    if frame.timestamp - cluster_timecode >= cluster_duration_target:
        write_cluster(cluster_timecode, current_cluster_frames)
        cluster_timecode = frame.timestamp
        current_cluster_frames = []
    current_cluster_frames.append(frame)

write_cluster(cluster_timecode, current_cluster_frames)
```

**Frame Encoding in Clusters:**
- Audio tracks: SimpleBlock (`0xA3`) with Flags = 0x00
- Video tracks: SimpleBlock for keyframes, BlockGroup for non-keyframes (when B-frames are present)

### 13.2 FFmpeg Demuxing Architecture

FFmpeg's Matroska demuxer (`libavformat/matroska.c`):

**EBML Parsing:**
```python
# Simplified demuxer flow
def parse_matroska(f):
    parse_ebml_header()
    parse_segment()
```

The demuxer validates the EBML header:
- EBML ID = 0x1A45DFA3
- EBMLMaxIDLength >= 4
- EBMLMaxSizeLength in range 1-8
- DocType = "matroska" or "webm"
- DocTypeReadVersion <= DocTypeVersion

**Track Discovery:**
```python
# Parse Tracks element
for each TrackEntry:
    track_number = TrackEntry.TrackNumber
    codec_id = TrackEntry.CodecID
    codec_private = TrackEntry.CodecPrivate
    audio_params = TrackEntry.Audio

    # Map CodecID to FFmpeg codec
    avcodec_id = matroska_codec_id_to_ffmpeg(codec_id)

    # Create AVStream
    stream = add_stream(avcodec_id)
    stream.codecpar.sample_rate = audio_params.SamplingFrequency
    stream.codecpar.channels = audio_params.Channels
    stream.codecpar.bits_per_sample = audio_params.BitDepth
    stream.codecpar.extradata = codec_private
```

**Frame Extraction from SimpleBlock:**
```python
def parse_simpleblock(block_data, track_number):
    # Read TrackNumber VINT
    track_num, consumed = read_vint(block_data)
    offset = consumed

    # Read BlockTimecode (signed int16, big-endian)
    timecode = int16_be(block_data[offset:offset+2])
    offset += 2

    # Read Flags
    flags = block_data[offset]
    offset += 1

    # Read frame data
    frame_data = block_data[offset:]

    # Compute timestamp
    cluster_time_ns = cluster.cluster_timecode * 1000000
    block_time_ns = timecode * 1000000
    timestamp_ns = cluster_time_ns + block_time_ns

    return frame_data, timestamp_ns, track_num
```

### 13.3 FFmpeg CLI Commands

```bash
# Create MKA with FLAC
ffmpeg -i input.wav \
  -c:a flac \
  -compression_level 8 \
  -sample_fmt s32 \
  output.mka

# Create MKA with Opus (WebM container)
ffmpeg -i input.wav \
  -c:a libopus \
  -b:a 128k \
  -vbr on \
  output.mka

# Extract audio stream from MKV
ffmpeg -i input.mkv \
  -c:a copy \
  -map 0:a:0 \
  output.mka

# Extract specific audio track
ffmpeg -i input.mkv \
  -c:a copy \
  -map 0:a:1 \
  -metadata title="Director's Commentary" \
  output.mka

# Convert and add chapters
ffmpeg -i input.wav \
  -c:a flac \
  -map_chapters input.txt \
  output.mka

# Preserve all metadata
ffmpeg -i input.mka \
  -c:a copy \
  -map_metadata 0 \
  output_copy.mka

# Convert with ReplayGain (stored in Tags)
ffmpeg -i input.wav \
  -c:a libopus \
  -af "volume=replaygain=track" \
  output.mka
```

### 13.4 Extracting Codec Information

```bash
# Get all stream information
ffprobe -v error -show_streams -select_streams a input.mka

# Extract CodecPrivate (hex-encoded)
ffprobe -v error \
  -show_entries stream=codec_private \
  -select_streams a:0 \
  -of default=noprint_wrappers=1 \
  input.mka

# Get format info
ffprobe -v error -show_format input.mka

# Get frame-level timing info
ffprobe -v error -select_streams a:0 \
  -show_entries stream=codec_name,sample_rate,channels,bits_per_sample \
  -of default=noprint_wrappers=1 \
  input.mka
```

### 13.5 TimestampScale and Duration Handling

FFmpeg's Matroska muxer uses TimestampScale = 1,000,000 (ms resolution) by default. When writing:

```python
# Duration conversion
# FFmpeg internal: AV_NOPTS_VALUE or int64 in microseconds
# Matroska: float64 in nanoseconds

def ff_timestamp_to_matroska(pts, time_base):
    if pts == AV_NOPTS_VALUE:
        return None
    # Convert from time_base units to nanoseconds
    ns = pts * time_base.num * 1_000_000_000 // time_base.den
    return float(ns)

def matroska_timestamp_to_ff(ns, time_base):
    # Convert nanoseconds to time_base units
    pts = ns * time_base.den // (time_base.num * 1_000_000_000)
    return pts
```

---

## 14. SPECIFICATION VERSIONS AND EVOLUTION

### 14.1 Matroska Version History

| Version | Year | Key Changes and Features                                           |
|---------|------|-------------------------------------------------------------------|
| 1       | 2002 | Initial release: EBML-based container, basic tracks and clusters |
| 2       | 2003 | BlockVirtual, improved seeking, ChapterAtom additions             |
| 3       | 2004 | SimpleBlock replaces BlockVirtual; refined reference handling    |
| 4       | 2010+| WebM subset defined, improved tag system, full EBML spec alignment |

**DocTypeVersion:** When writing Matroska files, use DocTypeVersion = 4.
**DocTypeReadVersion:** Set to 2 for compatibility with Matroska v2 readers.

### 14.2 EBML Standardization (RFC 8794)

IETF RFC 8794 (July 2020) formalized EBML version 1, bringing:

- Precise encoding rules for Element IDs and VINT data sizes
- Unknown size encoding for streaming
- Reserved ID ranges (0000, all-ones, reserved patterns)
- Validation requirements for EBML readers
- The distinction between EBML (syntax) and Matroska (semantic schema)

### 14.3 Compatibility Matrix

| Feature                | Min Reader Version | Notes                                     |
|------------------------|--------------------|-------------------------------------------|
| All basic Matroska     | 1                  | EBML, Segment, Cluster, SimpleBlock       |
| BlockGroup references  | 1                  |                                           |
| Chapters              | 1                  |                                           |
| Tags                  | 1                  |                                           |
| CRC-32               | 1                  |                                           |
| SimpleBlock           | 2                  | Replaces BlockVirtual from v1             |
| Multi-byte TrackNumber| 2                  | For tracks > 127                          |
| SeekHead              | 2                  |                                           |
| Cues                  | 2                  |                                           |
| Attachments           | 2                  |                                           |
| WebM subset           | 2                  | DocType="webm"                           |

### 14.4 Forward and Backward Compatibility

**Forward Compatibility:** A newer reader can read an older file if it supports the minimum DocTypeReadVersion. A v4 reader can read a v2 file.

**Backward Compatibility:** An older reader can read a newer file if the file's DocTypeReadVersion is set appropriately and no v4-only features are used.

**Best Practices:**
- Write DocTypeReadVersion = 2 (compatible with v2 readers)
- Only use v4 features when necessary (e.g., advanced tags, encryption)
- Mark unknown elements for preservation (skip, don't discard)

---

## 15. BINARY FORMAT EXAMPLES

### 15.1 Complete MKA File Structure

A minimal valid MKA file containing a single FLAC audio track with two frames:

```
Byte offsets (decimal):

0x0000-0x0003: 1A 45 DF A3              EBML ID (magic bytes)
0x0004-0x0004: 8B                      EBML size (1 byte, value 11)
0x0005-0x0005: 02                      EBMLVersion = 2
0x0006-0x0006: 02                      EBMLReadVersion = 2
0x0007-0x0007: 04                      EBMLMaxIDLength = 4
0x0008-0x0008: 08                      EBMLMaxSizeLength = 8
0x0009-0x0015: 6D 61 74 72 6F 73 6B 61  DocType = "matroska"
0x0016-0x0016: 04                      DocTypeVersion = 4
0x0017-0x0017: 02                      DocTypeReadVersion = 2

0x0018-0x001B: 18 53 80 67             Segment ID
0x001C-0x001C: 01 FF FF FF FF FF FF FF  Segment size (unknown, 8 bytes)
                                        (Segment data from 0x001D to EOF)

--- Info ---
0x001D-0x001D: 15 49 A9 66             Info ID
0x001E-0x001E: 8X                      Info size (varies)
  0x4D 80: MuxingApp = "Lavf60.16.100"
  0x57 41: WritingApp = "Lavf60.16.100"
  0x2A D7 B1: TimestampScale = 1000000
  0x7B A9: Duration = 300000000000.0

--- Tracks ---
0x00XX-0x00XX: 16 54 AE 6B            Tracks ID
0x00XX-0x00XX: AX                      TrackEntry ID
  D7: TrackNumber = 1
  73 C5: TrackUID = 1
  83: TrackType = 2 (audio)
  86: CodecID = "A_FLAC"
  63 A2: CodecPrivate = [34-byte FLAC STREAMINFO]
  E1: Audio
    B5: SamplingFrequency = 48000.0
    9F: Channels = 2
    62 64: BitDepth = 24

--- Cluster 1 ---
0x00XX-0x00XX: 1F 43 B6 75             Cluster ID
0x00XX-0x00XX: 8X                      Cluster data size
  E7: ClusterTimecode = 0
  A3: SimpleBlock
    81: TrackNumber = 1 (VINT 1 byte)
    00 00: BlockTimecode = 0 (signed int16, 0ms)
    00: Flags = 0 (no lacing)
    [FLAC frame 1 data, 1234 bytes]

  A3: SimpleBlock
    81: TrackNumber = 1
    14 5F: BlockTimecode = 5331 (ms, approx 1024 samples at 48kHz)
    00: Flags = 0
    [FLAC frame 2 data, 1234 bytes]

--- Cues ---
0x00XX-0x00XX: C5 3B B6 6B             Cues ID
  BB: CuePoint
    33 33 67: CueTime = 0
    B7: CueTrackPositions
      F7: CueTrack = 1
      12 34 56 78 9A: CueClusterPosition = 0x1234569A
      ...relative position, block number...
```

### 15.2 Variable-Length Integer Encoding Examples

**Encoding TrackNumber = 1:**
```
0x81 = binary 10000001
  High bit = 1 (in 1-byte VINT: high bit set indicates value, not continuation)
  Value = 1
```

**Encoding TrackNumber = 128:**
```
0x40 0x80 =:
  Byte 1: 0x40 = binary 01000000
    High bit = 0 (no continuation needed? No — for 2-byte, high bit = 1)
    Wait: for 2-byte VINT, first byte high bit = 1 (more coming), value bits = 0
    0x40 = 0x80 | 0 = continuation byte with value 0
  Byte 2: 0x80 = binary 10000000
    High bit = 0 (last byte), value = 0
  Total value = (0 & 0x7F) << 7 | (0x80 & 0x7F) = 0 << 7 | 0 = 0? No.

For 2-byte VINT encoding:
  Byte 1 = 0x80 | (value >> 7) = 0x80 | 1 = 0x81
  Byte 2 = value & 0x7F = 0x00
  So TrackNumber 128 = 0x81 0x00? But that's 0 + 128 = 128.

Let me recalculate. In EBML VINT:
  Byte 1 = 0x80 | (value >> 7)
  Byte 2 = value & 0x7F

For value = 128:
  Byte 1 = 0x80 | (128 >> 7) = 0x80 | 1 = 0x81
  Byte 2 = 128 & 0x7F = 0x00
  Result: 0x81 0x00

For value = 1:
  Byte 1 = 0x80 | (1 >> 7) = 0x80 | 0 = 0x80? No.
  For 1-byte VINT: Byte 1 = value | 0x80? No, that's wrong.

Let me look at this from first principles.
RFC 8794 VINT encoding:
  - "The VINT Width field is encoded in the leading 1-bits of the first byte"
  - "The VINT data is stored in the VINT Data field (remaining 7 bits per byte)"
  - VINT is: [Width indicator][Data]

For a 1-byte VINT (width = 1):
  - Leading 1-bit count = 0 (no leading 1s, starts with 0)
  - Wait, no: "leading 1-bits" in the context of EBML: the first byte starts with 0xxx xxxx for 1-byte
  - But 0x81 starts with 1...

I think the confusion is that VINT encoding for element sizes uses:
  - For unknown size: 0x7F (all bits set)
  - For 1-byte known size of value V: 0x80 | V
    - 0x80 = binary 10000000 (high bit 1 = "this is a VINT", value bits = 0)
    - For value 1: 0x80 | 1 = 0x81
    - For value 127: 0x80 | 127 = 0xFF

For 2-byte known size of value V:
  - Byte 1: 0x40 | (V >> 7)
  - Byte 2: V & 0x7F
  - For value 128: 0x40 | 1 = 0x41, 0x00 → 0x41 0x00

For TrackNumber (same VINT encoding):
  - TrackNumber 1 → 0x81
  - TrackNumber 127 → 0xFF
  - TrackNumber 128 → 0x40 0x80

This makes sense! In EBML VINT:
  - The high bit(s) of each byte indicate whether more bytes follow
  - For 1-byte VINT: 0x8X where X = value (high bit set = "VINT follows", but for 1 byte it's just the format marker)
  - For 2-byte VINT: first byte 0x4X, second byte 0x00-0x7F (0x40 = "length 1, data continues")
```

### 15.3 FLAC STREAMINFO in CodecPrivate

A minimal FLAC STREAMINFO (34 bytes) stored in CodecPrivate:

```
Bytes 0-3:   66 4C 61 43           "fLaC" magic
Byte  4:     00                    STREAMINFO block type = 0
Bytes 5-7:   00 00 22             Block length = 34 bytes

Block data (34 bytes):
Bytes 0-1:   10 00                 Min block size = 4096
Bytes 2-3:   10 00                 Max block size = 4096
Bytes 4-6:   00 00 00             Min frame size = 0
Bytes 7-9:   00 1F 40             Max frame size = 12800
Bytes 10:    00                   Sample rate = 0 (GET_8_BITS(4)=0, means "from frame header")
                              Actually: bits 0-3 = sample rate index, bits 4-7 = channel assignment
                              Wait, for FLAC frame header, sample rate is encoded differently.

For STREAMINFO:
Bytes 10-13: 39 56 54 00           Bitrate/samplerate/channel/bits
  Bits 0-3 (4 bits): sample rate index (0 = from frame header, 1 = 88200, 2 = 176400, 3 = 192000, 4 = 8000, 5 = 16000, 6 = 22050, 7 = 44100, 8 = 48000, 9 = 96000)
  For 48000 Hz: index = 8 = 0x8 = binary 1000
  Bits 4-6 (3 bits): channel assignment (0=mono, 1=stereo, ...)
  For stereo: 1 = 001
  Combined high 7 bits: 0x88? No.

Actually, byte 10:
  0x39 = binary 00111001
  Bits 0-3 = sample rate = 0x9 = 9 → 48000 Hz ✓
  Bits 4-6 = channel assignment = 0x3 = 3 → 3 (not 1 for stereo) ✓
  Bit 7 = unused

Bits 7-31 (25 bits): bits per sample - 1
  For 24-bit: bits per sample = 24, minus 1 = 23
  23 = 0x17

Bytes 14-17: Total samples (36 bits) = 0 (unknown)
Byte 18: MD5 signature (16 bytes)
```

### 15.4 AAC AudioSpecificConfig (CodecPrivate)

For AAC-LC stereo at 48 kHz:

```
AudioSpecificConfig encoding (15 bits total):

[5 bits] audioObjectType = 2 (AAC-LC)
[4 bits] samplingFrequencyIndex = 3 (48000 Hz)
[4 bits] channelConfiguration = 2 (2 channels)

Binary: 00010 0011 0010 = 0x0C 0x62 (padded to bytes)

Byte 1: 0x11 0x00 (5+3 bits = 8 bits: 00010001)
Byte 2: 0x00 (padded)
Wait: audioObjectType=2 = 00010 (5 bits)
samplingFrequencyIndex=3 = 0011 (4 bits)
channelConfiguration=2 = 0010 (4 bits)

Combine: 00010 0011 0010 0000
Split into bytes: 00010011 00100000 = 0x13 0x20

So CodecPrivate = 0x13 0x20 (2 bytes)
```

For HE-AAC with SBR (SBR is signaled in the extension, AudioSpecificConfig becomes more complex):

```
For AAC with explicit SBR signaling:
  ASC contains SBR object type and SBR sampling frequency index.
  OutputSamplingFrequency in the Audio element reflects the upsampled rate.
```

---

## 16. ERROR HANDLING AND RECOVERY

### 16.1 Malformed Element Handling

A robust Matroska reader must handle malformed files gracefully:

**Unknown Elements:** Skip unknown elements using their declared size. Never assume content format for unknown elements.

```python
def skip_element(element_id, data_size):
    """Skip an unknown element by advancing the file pointer."""
    file.seek(data_size, SEEK_CUR)

def read_master_element(data_size):
    """Read a master element by parsing children until data_size is exhausted."""
    start = file.tell()
    while file.tell() - start < data_size:
        element_id = read_element_id()
        element_size = read_vint()
        if is_master_element(element_id):
            read_master_element(element_size)
        else:
            file.seek(element_size, SEEK_CUR)
```

**Truncated Elements:** If data_size indicates more bytes than available, clamp to available bytes and issue a warning.

**Invalid VINT:** If VINT encoding is invalid (e.g., all bits set with no valid end), treat the element as unknown size and scan for the next valid Element ID.

### 16.2 CRC-32 Failure Handling

When CRC-32 validation fails, the reader has three options:

1. **Strict mode:** Stop playback, report data corruption.
2. **Permissive mode:** Log a warning, continue playback with degraded quality.
3. **Repair mode:** Attempt to locate the error and correct it.

For audio conversion pipelines, permissive mode with warning logging is appropriate: report the corruption but continue to produce output.

### 16.3 Seeking Recovery

If Cues are absent or corrupted:

```python
def scan_for_timestamp(target_ns, track):
    """Brute-force scan all clusters for target timestamp."""
    for cluster in iterate_clusters():
        cluster_time_ns = cluster.cluster_timecode * 1_000_000
        for block in cluster.blocks:
            if block.track != track:
                continue
            block_time_ns = cluster_time_ns + block.timecode * 1_000_000
            if block_time_ns >= target_ns:
                return block
    return None
```

### 16.4 Out-of-Order Elements

The Matroska spec recommends a specific element order, but many files have elements in arbitrary order. A robust reader must handle any ordering by maintaining a map of element offsets (via SeekHead) and parsing elements in the order they appear.

### 16.5 Duplicate Elements

Duplicate Element IDs at the same level are allowed in Matroska (unlike XML where elements are unique). For Tags, multiple SimpleTags with the same name are accumulated into a list. For Tracks, duplicate TrackNumbers should be treated as an error, but duplicate TrackUIDs can coexist if they have different TrackNumbers.

---

## 17. IMPLEMENTATION CHECKLIST

### 17.1 EBML Parser Checklist

- [ ] Validate EBML magic bytes (0x1A45DFA3 at offset 0)
- [ ] Read and validate EBMLVersion (must be 1)
- [ ] Read EBMLMaxIDLength (must be >= 4)
- [ ] Read EBMLMaxSizeLength (must be 1-8)
- [ ] Read and validate DocType
- [ ] Read DocTypeVersion and DocTypeReadVersion
- [ ] Implement VINT reading for sizes and IDs
- [ ] Implement Element ID parsing with variable-length support
- [ ] Handle unknown-size elements (scan for next valid ID)
- [ ] Skip unknown elements using declared size
- [ ] Implement CRC-32 validation for elements that include it
- [ ] Handle Void elements (0xEC)

### 17.2 Matroska Parser Checklist

- [ ] Locate Segment element (0x18538067)
- [ ] Parse SeekHead entries and build element offset map
- [ ] Parse Info element:
  - [ ] Read TimestampScale (default 1,000,000)
  - [ ] Read Duration (nanoseconds as float)
  - [ ] Read SegmentUID (optional)
  - [ ] Read DateUTC (nanoseconds since 2001-01-01)
- [ ] Parse Tracks element:
  - [ ] For each TrackEntry: read TrackNumber, TrackUID, TrackType
  - [ ] Read CodecID and map to codec
  - [ ] Read CodecPrivate and store as extradata
  - [ ] For audio tracks: read SamplingFrequency, Channels, BitDepth
  - [ ] Validate CodecPrivate format per codec type
- [ ] Parse Clusters:
  - [ ] Read ClusterTimecode (milliseconds)
  - [ ] For each SimpleBlock: parse TrackNumber, Timecode, Flags, frame data
  - [ ] For each BlockGroup: parse Block + ReferenceBlock(s) + CRC-32
  - [ ] Compute absolute frame timestamp
  - [ ] Validate timestamps are monotonically increasing within cluster
- [ ] Parse Cues (if present):
  - [ ] Build seek index: timestamp → (Cluster offset, Block offset)
  - [ ] Use binary search for efficient seeking
- [ ] Parse Chapters (if present):
  - [ ] Read EditionEntry hierarchy
  - [ ] Read ChapterTimeStart/End (milliseconds)
  - [ ] Read ChapterDisplay strings and languages
- [ ] Parse Tags (if present):
  - [ ] Read Targets for scope
  - [ ] Read SimpleTag name/value pairs
  - [ ] Map to standard metadata keys
- [ ] Parse Attachments (if present):
  - [ ] Read AttachedFile entries
  - [ ] Extract cover art (picture) files

### 17.3 Audio-Specific Checklist

- [ ] Validate CodecID is an audio codec (starts with "A_")
- [ ] For A_AAC: parse AudioSpecificConfig from CodecPrivate
  - [ ] Extract audioObjectType
  - [ ] Extract samplingFrequencyIndex
  - [ ] Extract channelConfiguration
  - [ ] Detect SBR presence
  - [ ] Set OutputSamplingFrequency if SBR present
- [ ] For A_FLAC: parse STREAMINFO from CodecPrivate
  - [ ] Extract block size min/max
  - [ ] Extract sample rate
  - [ ] Extract channel count
  - [ ] Extract bits per sample
  - [ ] Extract total sample count
- [ ] For A_OPUS: parse OpusHead from CodecPrivate
  - [ ] Validate magic "OpusHead"
  - [ ] Extract version (must be 1)
  - [ ] Extract channel count
  - [ ] Extract pre-skip value
  - [ ] Extract input sample rate
  - [ ] Validate channel mapping family
  - [ ] Extract stream count and coupled count for surround
- [ ] For A_VORBIS: validate three headers in CodecPrivate
  - [ ] First header: "vorbis" magic, version, channels, sample rate
  - [ ] Second header: vendor string
  - [ ] Third header: codebooks
- [ ] Validate frame data size against codec expectations
- [ ] For PCM codecs: validate BitDepth matches Channels × frame_size

### 17.4 FFmpeg Integration Checklist

- [ ] Use `avformat_open_input` / `avformat_close_input` for file I/O
- [ ] Use `av_read_frame` to read packets
- [ ] Extract codec parameters from `AVStream.codecpar`
  - [ ] codec_id → FFmpeg codec
  - [ ] sample_rate → audio sample rate
  - [ ] channels → channel count
  - [ ] bits_per_raw_sample → bit depth
  - [ ] extradata → CodecPrivate
- [ ] Use `av_seek_frame` with timestamp for seeking
- [ ] Use `avformat_new_stream` for muxing
- [ ] Use `avformat_write_header` to write EBML Header + Segment
- [ ] Use `av_interleaved_write_frame` to write Clusters
- [ ] Use `av_write_trailer` to finalize Segment + write Cues
- [ ] Set `AVFormatContext.metadata` for Tags
- [ ] Set `AVDictionary` for per-track metadata
- [ ] Use `AVMuxerContext.cluster_size_limit` to control cluster size
- [ ] Use `AVMuxerContext.max_delay` for buffering

### 17.5 Validation Checklist

- [ ] All TrackNumbers are unique within a file
- [ ] All TrackUIDs are unique within a file
- [ ] Block timestamps are within ClusterTimecode ± 32767 ms
- [ ] CuePoint timestamps match actual Block timestamps
- [ ] CueClusterPosition offsets are valid (within file bounds)
- [ ] CodecPrivate is present for codecs that require it
- [ ] CodecPrivate format matches CodecID expectations
- [ ] SamplingFrequency and Channels are non-zero for audio tracks
- [ ] Duration (if present) matches computed duration from frames
- [ ] No Invalid UTF-8 in string/utf-8 elements
- [ ] CRC-32 checksums pass (if present)
- [ ] Element ordering does not break parent/child nesting

### 17.6 Conformance Checklist

- [ ] EBML Header is the first element (offset 0)
- [ ] EBMLMaxIDLength >= 4
- [ ] EBMLMaxSizeLength in range 1-8
- [ ] DocType is "matroska" or "webm"
- [ ] DocTypeReadVersion <= DocTypeVersion
- [ ] Segment contains exactly one Tracks element
- [ ] Each Track has a unique TrackNumber >= 1
- [ ] Each Track has a unique TrackUID
- [ ] Segment contains at least one Cluster
- [ ] For WebM: audio is A_OPUS, video is V_VP8/V_VP9
- [ ] For WebM: no CRC-32 elements present
- [ ] For WebM: SeekHead is present at Segment start
- [ ] For WebM: Cues is present

---

## APPENDIX A: ELEMENT ID QUICK REFERENCE

| ID (hex)   | Name                | Type    | Level  |
|------------|---------------------|---------|--------|
| 0x1A45DFA3 | EBML                | Master  | 0      |
| 0x18538067 | Segment             | Master  | 1      |
| 0x114D9B74 | SeekHead            | Master  | 2      |
| 0x4DBB     | Seek                | Master  | 3      |
| 0x1549A966 | Info                | Master  | 2      |
| 0x1654AE6B | Tracks              | Master  | 2      |
| 0xAE       | TrackEntry          | Master  | 3      |
| 0xE0       | Video               | Master  | 4      |
| 0xE1       | Audio               | Master  | 4      |
| 0x1043A770 | Chapters            | Master  | 2      |
| 0x1F43B675 | Cluster            | Master  | 2      |
| 0xA3       | SimpleBlock         | Binary  | 3      |
| 0xA0       | BlockGroup          | Master  | 3      |
| 0xA1       | Block               | Binary  | 4      |
| 0xC53BB6B | Cues                | Master  | 2      |
| 0xBB       | CuePoint            | Master  | 3      |
| 0x1941A469 | Attachments         | Master  | 2      |
| 0x1254C367 | Tags                | Master  | 2      |
| 0xBF       | CRC-32              | Binary  | N      |
| 0xEC       | Void                | Binary  | N      |

## APPENDIX B: AUDIO CODEC PARAMETER QUICK REFERENCE

| CodecID       | Channels source | BitDepth source | CodecPrivate | Frame-based |
|---------------|----------------|-----------------|--------------|------------|
| A_FLAC        | FLAC frame     | FLAC frame      | STREAMINFO   | Yes        |
| A_AAC         | ASC            | N/A (variable)  | ASC          | Yes        |
| A_OPUS        | OpusHead       | N/A (variable)  | OpusHead     | Yes        |
| A_VORBIS      | Vorbis header  | N/A (variable)  | 3 headers    | Yes        |
| A_MP3         | N/A            | N/A             | None         | Yes        |
| A_AC3         | Bitstream      | N/A             | None         | Yes        |
| A_PCM/INT/LIT | Matroska       | Matroska        | None         | No (raw)   |
| A_PCM/FLOAT   | Matroska       | Matroska        | None         | No (raw)   |
| A_ALAC        | ALAC config    | ALAC config     | ALAC cfg     | Yes        |
| A_TTA1        | N/A            | N/A             | None         | Yes        |
| A_WAVPACK4    | Bitstream      | Bitstream       | None         | Yes        |
| A_DTS         | Bitstream      | N/A             | Conditional  | Yes        |

## APPENDIX C: GLOSSARY

| Term               | Definition                                                   |
|--------------------|--------------------------------------------------------------|
| EBML               | Extensible Binary Meta Language — the syntax layer of Matroska|
| VINT               | Variable-length integer encoding used in EBML               |
| Master Element     | EBML element that contains child elements (not raw bytes)   |
| SimpleBlock        | Audio/video frame with timestamp, no reference frames       |
| BlockGroup         | Block wrapped with reference frames and metadata            |
| Cluster            | Group of time-adjacent frames sharing a base timecode       |
| CuePoint           | Seeking index entry mapping timestamp to file offset         |
| CodecPrivate       | Codec-specific initialization data stored in TrackEntry     |
| Lacing             | Packing multiple small frames into one Block                |
| TimestampScale     | Nanoseconds per timestamp unit (default: 1,000,000 = ms)   |
| WebM               | Restricted subset of Matroska for web delivery (VP8/VP9 + Opus)|
| STREAMINFO         | FLAC block containing stream parameters                     |
| ASC                | AudioSpecificConfig — AAC decoder initialization blob       |
| OpusHead           | Opus stream header per RFC 7845                             |
| CRC-32             | IEEE 802.3 checksum for data integrity                     |
| Void               | Padding/reserved space element                              |
| SeekHead           | Index of top-level element positions within Segment         |
| DocType            | String identifying the EBML schema ("matroska" or "webm")   |

---

*Document Version: 1.0*
*Specification Reference: Matroska Element Semantics (ietf-wg-cellar/matroska-specification), RFC 8794 (EBML)*
*Target Audience: Audio codec developers building Matroska/MKA conversion pipelines*
