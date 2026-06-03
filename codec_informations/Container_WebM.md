# WebM Container Format — Deep Technical Reference
> **Category:** Container
> **File Extensions:** `.webm`
> **MIME Types:** `video/webm`, `audio/webm`
> **Standardization Body:** WebM Project (Google), constrained subset of IETF RFC 9559 (Matroska)
> **Primary Specification:** WebM Container Guidelines — https://www.webmproject.org/docs/container/
> **Patent Status:** Patent-free — royalty-free open format
> **License:** BSD / Creative Commons
> **Current Version:** WebM v2 (based on Matroska v3)
> **Active Development:** Yes — maintained by the WebM Project and Alliance for Open Media

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Google, On2 Technologies (acquired 2010), Xiph.Org
- **Year Created:** 2010 (announced May 2010, released December 2010)
- **Original Purpose:** Provide an open, royalty-free multimedia container format optimized for the web, based on Matroska but constrained to ensure efficient streaming, cross-browser compatibility, and simplicity
- **Problem with Predecessors:** The broader Matroska specification contained many features unnecessary for web use, leading to inconsistent implementations. WebM was created as a well-defined subset guaranteed to work across all browsers and players.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| WebM v1 | 2010 | Initial release: VP8 + Vorbis |
| WebM v2 | 2013 | Added VP9 support, Opus codec |
| WebM v3 | 2020 | AV1 codec support, refined guidelines |
| Current | 2024 | Matroska RFC 9559 adoption |

### 1.3 Current Adoption
- **Primary use cases today:** HTML5 video/audio on the web, video conferencing (WebRTC), YouTube (legacy), streaming servers, Chrome/Firefox/Edge native playback
- **Platforms with native support:** All major browsers (Chrome, Firefox, Edge, Safari 14+), WebRTC implementations
- **Major services using this format:** YouTube (legacy uploads), video conferencing (Zoom, Google Meet historically), streaming platforms
- **Hardware support:** Broad — supported in hardware by many modern chips via AV1/VP9 decode
- **Status:** Active / dominant for web video and audio

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 WebM Architecture
WebM is a **constrained subset of Matroska v3** built on the EBML (Extensible Binary Meta Language) binary container format. Every WebM file is technically a valid Matroska file, but not every Matroska file is a valid WebM file.

```
WebM File:
  [EBML Header]                   ← Mandatory, first element
      ├── EBMLVersion (must be 1)
      ├── EBMLReadVersion (must be 1)
      ├── EBMLMaxIDLength (must be 4)
      ├── EBMLMaxSizeLength (must be 8)
      ├── DocType (must be "webm")
      ├── DocTypeVersion (must be 2)
      └── DocTypeReadVersion (must be 2)
  │
  ▼
  [Segment]                        ← Root container for all content
      │
      ├── [SeekHead]                ← Optional: offset lookup table
      ├── [Info]                   ← Mandatory: file-level metadata
      │       ├── MuxingApp
      │       ├── WritingApp
      │       └── Duration
      │
      ├── [Tracks]                 ← Mandatory: track definitions
      │       └── [TrackEntry] × N
      │           ├── Track Number
      │           ├── Track UID
      │           ├── Codec ID (V_VP8, V_VP9, V_AV1, A_VORBIS, A_OPUS)
      │           ├── CodecPrivate (codec initialization data)
      │           └── Audio/Video-specific fields
      │
      ├── [Chapters]               ← Optional
      │
      ├── [Cluster] × N            ← Audio/video data organized by timestamp
      │       ├── Timestamp (ms)
      │       ├── [SimpleBlock] × M — Audio/Video frame
      │       │       ├── track_number
      │       │       ├── timestamp (relative to cluster)
      │       │       ├── keyframe flag
      │       │       └── data (encoded frame)
      │       └── [BlockGroup] × K — Frame with additional metadata
      │
      ├── [Cues]                   ← Optional: seeking index
      │       └── [CuePoint] × N
      │           ├── CueTime
      │           └── CueTrackPositions × M
      │
      └── [Tags]                   ← Optional: metadata
```

### 2.2 WebM vs. Matroska Constraints

WebM is a **restricted profile** of Matroska with the following restrictions:

| Feature | WebM | Matroska | Notes |
|---------|-------|----------|-------|
| Video codec | VP8, VP9, AV1 only | Any | WebM constrains to web codecs |
| Audio codec | Vorbis, Opus only | Any | WebM constrains to open codecs |
| Chapter system | Limited | Full | WebM allows simple chapters |
| Attached files | Not allowed | Allowed | No attachments in WebM |
| Content encoding | Not allowed | Allowed | No compression of tracks |
| Segment sizing | Can use unknown size | Full | WebM allows unknown size for streaming |
| Multiple segments | Not allowed | Allowed | Single Segment only |

### 2.3 Key Design Parameters
|| Parameter | Value | Notes |
|-----------|-------|-------|
| Max streams | Unlimited (audio + video) | Typically 1 video + 1 audio |
| EBML ID length | 4 bytes | Fixed per WebM spec |
| EBML size length | 8 bytes | Fixed per WebM spec |
| Timestamp resolution | ms | Nanoseconds internally |
| Seeking | Via Cues element | Binary search on cluster |
| Streaming | Yes | Unknown size Segment enables streaming |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       1A 45 DF A3     ....     EBML header magic + ID
0x0004  1       9F             .        EBML header size (31 bytes)
```

The EBML header is both a magic number and a valid EBML element. The first 4 bytes `1A 45 DF A3` are both the EBML DocType header ID and the file magic.

### 3.2 EBML Header Layout
```
Offset  Bytes   Field Name              Type        Description
------  -----  ---------------------  ----------  ---------------------------
0x0000  4B     EBML ID                uint32     0x1A45DFA3 — "EBML" element ID
0x0004  1B     EBML Size             uint8      Size of EBML header data (31 bytes for WebM)
0x0005  2B     EBMLVersion            uint32     Must be 1
0x0007  2B     EBMLReadVersion        uint32     Must be 1
0x0009  1B     EBMLMaxIDLength        uint8      Must be 4
0x000A  1B     EBMLMaxSizeLength      uint8      Must be 8
0x000B  1B     DocTypeLength          uint8      Length of DocType string (4 for "webm")
0x000C  4B     DocType               char[4]    Must be "webm"
0x0010  2B     DocTypeVersion        uint16     Must be 2
0x0012  2B     DocTypeReadVersion     uint16     Must be 2
```

### 3.3 EBML Element IDs (WebM/Matroska)
WebM uses 4-byte element IDs. Key IDs:

| Element Name | ID (hex) | Description |
|-------------|----------|-------------|
| EBML | `0x1A45DFA3` | EBML header |
| Segment | `0x18538067` | Root segment |
| SeekHead | `0x114D9B74` | Seek position table |
| Info | `0x1549A966` | Segment metadata |
| Tracks | `0x1654AE6B` | Track definitions |
| TrackEntry | `0xAE` | Single track |
| TrackNumber | `0xD7` | Track number |
| TrackUID | `0x73C5` | Track unique ID |
| CodecID | `0x86` | Codec identifier string |
| CodecPrivate | `0x63A2` | Codec initialization data |
| Timestamp | `0xE7` | Cluster timestamp (ms) |
| SimpleBlock | `0xA3` | Uncoded frame (keyframe flag in flags) |
| Block | `0xA1` | Coded frame with reference |
| BlockGroup | `0xA0` | Frame with metadata |
| Cues | `0x1C53BB6B` | Seeking index |
| CuePoint | `0xBB` | Single index entry |
| Tags | `0x1254C367` | Metadata tags |

### 3.4 Track Entry (TrackEntry) — Video Track
```
Offset  Size   Field Name              Type        Valid Range   Description
------  -----  ---------------------  ----------  ------------  ---------------------------
0       1B     Track Type             uint8       1             1=video, 2=audio
0       1B     Track Number           uint8       1–255        Track identifier
0       4B     Track UID              uint32      nonzero       Unique identifier
0       1B     Codec ID              string      "V_VP8", "V_VP9", "V_AV1"
0       N      CodecPrivate           bytes      varies        VP8: none; VP9: features; AV1: sequence header
0       2B     Video: PixelWidth      uint16      1–∞          Frame width in pixels
0       2B     Video: PixelHeight     uint16      1–∞          Frame height in pixels
0       2B     Video: DisplayWidth     uint16      1–∞          Display width (optional)
0       2B     Video: DisplayHeight   uint16      1–∞          Display height (optional)
```

### 3.5 Track Entry (TrackEntry) — Audio Track
```
Offset  Size   Field Name              Type        Valid Range   Description
------  -----  ---------------------  ----------  ------------  ---------------------------
0       1B     Track Type             uint8       2             2=audio
0       1B     Track Number           uint8       1–255        Track identifier
0       4B     Track UID              uint32      nonzero       Unique identifier
0       1B     Codec ID              string      "A_VORBIS", "A_OPUS"
0       N      CodecPrivate           bytes      varies        Codec initialization data
0       8B     Audio: Sampling Frequency float64   any          Samples per second (e.g., 48000.0)
0       8B     Audio: Channels         float64    1–∞          Number of channels
0       2B     Audio: BitDepth         uint16     8–64         Bits per sample (optional)
```

### 3.6 CodecPrivate for Vorbis (A_VORBIS)
Vorbis in WebM requires all three Vorbis headers concatenated:

```
Total CodecPrivate size: header1_len + header2_len + header3_len + 3 (length prefixes)
  
  [1B]   Number of Vorbis headers = 3 (fixed)
  [2B]   Length of header 1 (identification header)
  [N1]   Vorbis identification header (30 bytes)
  [2B]   Length of header 2 (comment header)
  [N2]   Vorbis comment header (variable)
  [2B]   Length of header 3 (setup header)
  [N3]   Vorbis setup header (variable)
```

### 3.7 CodecPrivate for Opus (A_OPUS)
Opus in WebM uses the OpusHead structure:

```
Offset  Size   Field Name              Type        Valid Range   Description
------  -----  ---------------------  ----------  ------------  ---------------------------
0       1B     Channel Mapping        uint8       0–255        0=mono/stereo, 1=surround
1       2B     Mapping Family        uint16      0–255        Opus channel mapping family
3       2B     Stream Count          uint16      1–255        Number of coupled streams
5       1B     Coupled Stream Count  uint8       0–255        Number of coupled audio streams
6       N      Channel Mapping        bytes       varies        Channel mapping table
```

**Opus in WebM also requires OpusTags** — stored in a separate element or concatenated after OpusHead:

```
OpusHead (19 bytes minimum) + OpusTags
```

### 3.8 CodecPrivate for VP9 (V_VP9)
VP9 in WebM stores codec feature metadata:

```
Offset  Size   Field Name              Type        Description
------  -----  ---------------------  ----------  ---------------------------
0       1B     Profile               uint8       VP9 profile (0–3)
1       1B     Level                 uint8       VP9 level
2       1B     BitDepth              uint8       Bit depth (8, 10, 12)
3       1B     ChromaSubsampling     uint8       Chroma subsampling flags
4       1B     ChromaSamplingPosition uint8      Chroma sample position
```

### 3.9 Sample Format Support
|| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Integer | Yes | VP8, Vorbis, Opus |
| 10-bit | Integer | Yes | VP9, AV1, Opus |
| 12-bit | Integer | Yes | VP9, AV1 |
| 16-bit | Integer | Yes | Vorbis, Opus |
| 32-bit | IEEE float | Partial | Opus supports float output |

### 3.10 Supported Sample Rates
|| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 8000 | Telephone | Yes | Opus only |
| 12000 | — | Yes | Opus only |
| 16000 | Wideband | Yes | Opus, Vorbis |
| 24000 | — | Yes | Opus, Vorbis |
| 48000 | Professional | Yes | Opus (mandatory), Vorbis |
| 96000 | — | Yes | Opus only |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 WebM Encoding Pipeline

```
Input: Encoded video frames (VP8/VP9/AV1) + Encoded audio (Vorbis/Opus)
      │
      ▼
[Matroska/EBML Assembly]
1. Write EBML header (DocType = "webm")
2. Write Segment (unknown size for streaming)
3. Write SeekHead (offset table)
4. Write Info (muxing app, duration)
5. Write Tracks (video + audio track entries)
6. For each cluster:
   a. Write Cluster element
   b. Write Timestamp (ms)
   c. Write SimpleBlock for each frame
   d. Update cluster duration
7. Write Cues (seeking index)
8. Write Tags (metadata)
9. Patch Segment size in header
      │
      ▼
Output: WebM File
```

### 4.2 VP9 Encoding
VP9 is a motion-compensated transform codec similar to VP8 but with improved compression:

```
Input Frames
      │
      ▼
[Superframe Organization]
VP9 organizes frames into superframes (1–8 frames)
      │
      ▼
[Frame Types]
- Key frames (I-frames): Intra-predicted, full refresh
- Inter frames (P/B): Motion-compensated prediction
- Golden frames: Long-term reference frames
      │
      ▼
[Transform]
DCT/ADCT on 4×4 to 16×16 blocks
      │
      ▼
[Entropy Coding]
Bool coder with arithmetic coding
      │
      ▼
Output: VP9 compressed frame
```

### 4.3 Opus Encoding
Opus is a parametric codec combining SILK (speech) and CELT (music):

```
Input Audio (48 kHz, 2.5–120 ms frames)
      │
      ▼
[Mode Decision]
Encoder decides: SILK (speech), CELT (music), or hybrid
      │
      ├─ SILK Path (speech-optimized)
│   ├── LP analysis and prediction
│   ├── Redundant coding
│   └── Range coding
│
├─ CELT Path (music-optimized)
│   ├── MDCT transform
│   ├── Band energy coding
│   └── Pulse coding
│
▼
[Bitstream Packing]
Single unified bitstream
      │
      ▼
Output: Opus packet
```

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. Read EBML header: verify DocType = "webm"
2. Parse Segment header
3. Parse SeekHead (if present): build offset lookup table
4. Parse Info: get duration, timestamps
5. Parse Tracks: build track table (codec_id, CodecPrivate, parameters)
6. Initialize codec with CodecPrivate:
   - VP9: parse feature metadata
   - Vorbis: feed all three headers to decoder
   - Opus: feed OpusHead + OpusTags to decoder
7. For each Cluster:
   a. Read Cluster timestamp
   b. For each SimpleBlock:
      ├── Read track_number, timestamp delta, keyframe flag
      ├── Look up codec from track table
      ├── Decode frame with appropriate codec
      └── Queue for playback
8. Parse Cues for seeking (optional)
```

#### Seeking
Seeking uses the Cues element for O(log N) access:

```
1. Parse Cues element: build index of CuePoints
2. Binary search: find CuePoint where CueTime ≤ target_time
3. Read CuePoint: get Cluster position from CueTrackPositions
4. Seek to Cluster offset in file
5. Scan Cluster SimpleBlocks until target frame found
6. Decode from that point forward
```

### 5.2 Core Decode Pipeline
```
1. Read EBML header → verify "webm" DocType
2. Parse Info → get duration, timestamp scale
3. Parse Tracks → build codec table:
   ├── Video: VP8/VP9/AV1 codec + CodecPrivate
   └── Audio: Vorbis/Opus codec + CodecPrivate
4. Initialize decoders with CodecPrivate:
   ├── VP8: no private data needed
   ├── VP9: parse profile/level/bitdepth from CodecPrivate
   ├── Vorbis: feed identification + comment + setup headers
   └── Opus: feed OpusHead + OpusTags
5. For each Cluster:
   ├── Read Cluster timestamp
   └── For each SimpleBlock:
       ├── Extract: track_number, timestamp, keyframe flag, data
       ├── Decode frame with track's codec
       └── Output PCM (audio) or display frame (video)
6. Apply timestamp: cluster_ts + block_ts_delta
```

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** WebM (EBML/Matroska subset)
- **Overhead:** ~1–3% (EBML overhead, cluster headers)
- **Seeking in native container:** Yes — via Cues element
- **Multiple streams in native container:** Yes — 1 video + 1 audio (typical)

### 6.2 Codec-to-Container Compatibility
|| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| WebM (native) | VP8, VP9, AV1, Vorbis, Opus | Yes (Cues) | Full (Tags) | Preferred |
| Matroska/MKA | Any codec | Yes | Full | Over-features for web |
| MP4/M4A | H.264, H.265, AAC | Yes | Full | Better for iOS |
| OGG | Vorbis, Opus | Yes | Limited | Alternative |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** Matroska/WebM Tags element
- **Tag block location:** End of Segment (after Clusters)
- **Tag block identifier:** `Tags` element ID (`0x1254C367`)

### 7.2 Standard Tag Fields — Complete Reference
|| Tag Field | Internal Key (WebM) | Max Length | Character Encoding | Multi-value | Notes |
|-----------|---------------------------|------------|-------------------|-------------|---|
| Title | TITLE | unlimited | UTF-8 | No | |
| Artist | ARTIST | unlimited | UTF-8 | Yes | |
| Album | ALBUM | unlimited | UTF-8 | No | |
| Album Artist | ALBUMARTIST | unlimited | UTF-8 | Yes | |
| Composer | COMPOSER | unlimited | UTF-8 | Yes | |
| Genre | GENRE | unlimited | UTF-8 | Yes | |
| Year | DATE | unlimited | UTF-8 | No | Year or full date |
| Track Number | tracknumber | unlimited | UTF-8 | No | Plain number |
| Track Total | totaltracks | unlimited | UTF-8 | No | |
| Disc Number | discnumber | unlimited | UTF-8 | No | |
| Disc Total | totaldiscs | unlimited | UTF-8 | No | |
| Comment | comment | unlimited | UTF-8 | Yes | |
| ReplayGain Track Gain | REPLAYGAIN_TRACK_GAIN | unlimited | ASCII | No | |
| ReplayGain Track Peak | REPLAYGAIN_TRACK_PEAK | unlimited | ASCII | No | |
| Encoder | encoder | unlimited | UTF-8 | No | Software name |
| Cover Art | ATTACHED_PIC | — | — | No | Not supported in WebM |

Note: **Attachments (cover art) are not allowed in WebM** per the specification. Cover art must be stored externally or in a separate file.

### 7.3 Metadata Compatibility Matrix
|| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| WebM Tags element | ✓ | ✓ | ✓ | Highest (native) |
| Vorbis Comments (legacy) | ✓ | ✓ | ✓ | High |
| ID3v1 | ✗ | ✗ | N/A | Not supported |
| ID3v2 | ✗ | ✗ | N/A | Not supported |

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   libvpx (VP8), libvpx-vp9 (VP9), libaom-av1 (AV1)
                     libvorbis (Vorbis), libopus (Opus)
AV_CODEC_ID:        AV_CODEC_ID_VP8, AV_CODEC_ID_VP9, AV_CODEC_ID_AV1
                     AV_CODEC_ID_VORBIS, AV_CODEC_ID_OPUS
Format Name (CLI):   webm, matroska                     # used with -f
Encoder(s):          libvpx, libvpx-vp9, libaom-av1, libvorbis, libopus
Decoder(s):          libvpx, libvpx-vp9, libaom-av1, libvorbis, libopus
Muxer(s):           webm, matroska                     # ffmpeg -muxers | grep -i webm
Demuxer(s):          webm, matroska                     # ffmpeg -demuxers | grep -i webm
```

### 8.2 FFmpeg Encoding — Full CLI Reference

```bash
# VP9 + Opus WebM (modern, recommended)
ffmpeg -i input.mp4 \
  -c:v libvpx-vp9 \
  -crf 30 \                     # Constant Quality mode (0–63, lower=better)
  -b:v 0 \                      # VBR mode; use -b:v N for CBR
  -c:a libopus \
  -b:a 128k \                  # Audio bitrate
  -f webm \
  output.webm

# VP9 + Opus with two-pass encoding
ffmpeg -i input.mp4 \
  -c:v libvpx-vp9 \
  -crf 30 -b:v 0 \
  -pass 1 \
  -f webm \
  /dev/null
ffmpeg -i input.mp4 \
  -c:v libvpx-vp9 \
  -crf 30 -b:v 0 \
  -pass 2 \
  -c:a libopus \
  -b:a 128k \
  -f webm \
  output.webm

# VP8 + Vorbis WebM (legacy compatibility)
ffmpeg -i input.mp4 \
  -c:v libvpx \
  -crf 30 -b:v 0 \
  -c:a libvorbis \
  -q:a 3 \                    # Vorbis quality (0–10)
  -f webm \
  output.webm

# AV1 + Opus WebM (cutting-edge)
ffmpeg -i input.mp4 \
  -c:v libaom-av1 \
  -crf 35 -b:v 0 \
  -c:a libopus \
  -b:a 128k \
  -f webm \
  output_av1.webm

# WebM muxer options
ffmpeg -i input.mp4 \
  -c:v libvpx-vp9 \
  -c:a libopus \
  -f webm \
  -live 1 \                   # Enable live streaming mode
  output_live.webm
```

#### Complete FFmpeg Option Table
|| Option | Type | Default | Range / Valid Values | Effect on Output |
|--------|------|---------|---------------------|-----------------|
| `-c:v` | string | — | libvpx, libvpx-vp9, libaom-av1 | Video codec |
| `-c:a` | string | — | libvorbis, libopus | Audio codec |
| `-crf` | int | 32 | 0–63 (VP9), 0–63 (AV1) | Quality (lower=better) |
| `-b:v` | int | — | 0=VBR, N=CBR kbps | Video bitrate mode |
| `-b:a` | int | — | N kbps | Audio bitrate |
| `-q:a` | int | 3 | 0–10 (Vorbis) | Vorbis quality |
| `-f webm` | — | — | — | Force WebM container |
| `-live` | bool | 0 | 0/1 | Live streaming mode |

### 8.3 FFmpeg Encoding — libavcodec C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/opt.h>

// ─── WebM muxing setup ─────────────────────────────────────────────────────────
// Use the matroska muxer with WebM constraints:
AVFormatContext *fmt_ctx = NULL;
avformat_alloc_output_context2(&fmt_ctx, NULL, "webm", "output.webm");

// Add video track
AVStream *video_st = avformat_new_stream(fmt_ctx, NULL);
avcodec_parameters_from_context(video_st->codecpar, video_ctx);

// Add audio track
AVStream *audio_st = avformat_new_stream(fmt_ctx, NULL);
avcodec_parameters_from_context(audio_st->codecpar, audio_ctx);

// Open output
avio_open(&fmt_ctx->pb, "output.webm", AVIO_FLAG_WRITE);
avformat_write_header(fmt_ctx, NULL);

// For each packet:
//   av_interleaved_write_frame(fmt_ctx, pkt);

av_write_trailer(fmt_ctx);
```

### 8.4 FFmpeg Decoding — Full CLI Reference

```bash
# Decode WebM to raw audio/video
ffmpeg -i input.webm \
  -c:v libvpx-vp9 \
  -c:a libopus \
  output.mp4

# Extract audio only
ffmpeg -i input.webm \
  -vn \                       # Skip video
  -c:a copy \                # Copy Opus stream directly
  output.opus

ffmpeg -i input.webm \
  -vn \
  -c:a libopus \
  output.wav

# Extract video only
ffmpeg -i input.webm \
  -an \                       # Skip audio
  -c:v copy \                 # Copy VP9 stream directly
  output.vp9

# Remux to MKV
ffmpeg -i input.webm \
  -c copy \
  -f matroska \
  output.mkv

# Probe format
ffprobe -v quiet -print_format json -show_streams -show_format input.webm
```

### 8.5 FFmpeg Decoding — libavcodec C API

```c
// ─── Complete decode program skeleton ─────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.webm", NULL, NULL);
avformat_find_stream_info(fmt_ctx, NULL);

int audio_idx = av_find_best_stream(fmt_ctx, AVMEDIA_TYPE_AUDIO, -1, -1, NULL, 0);
int video_idx = av_find_best_stream(fmt_ctx, AVMEDIA_TYPE_VIDEO, -1, -1, NULL, 0);

AVStream *audio_stream = fmt_ctx->streams[audio_idx];
AVStream *video_stream = fmt_ctx->streams[video_idx];

// Initialize audio decoder
const AVCodec *audio_dec = avcodec_find_decoder(audio_stream->codecpar->codec_id);
AVCodecContext *audio_ctx = avcodec_alloc_context3(audio_dec);
avcodec_parameters_to_context(audio_ctx, audio_stream->codecpar);
avcodec_open2(audio_ctx, audio_dec, NULL);

// Initialize video decoder
const AVCodec *video_dec = avcodec_find_decoder(video_stream->codecpar->codec_id);
AVCodecContext *video_ctx = avcodec_alloc_context3(video_dec);
avcodec_parameters_to_context(video_ctx, video_stream->codecpar);
avcodec_open2(video_ctx, video_dec, NULL);

AVPacket *pkt = av_packet_alloc();
AVFrame  *frm = av_frame_alloc();

while (av_read_frame(fmt_ctx, pkt) >= 0) {
    if (pkt->stream_index == audio_idx) {
        avcodec_send_packet(audio_ctx, pkt);
        while (avcodec_receive_frame(audio_ctx, frm) == 0) {
            // frm->data[0] = PCM samples
            // frm->nb_samples = samples per frame
            // frm->sample_rate = 48000 (Opus) or 44100/48000 (Vorbis)
            process_audio_frame(frm);
            av_frame_unref(frm);
        }
    } else if (pkt->stream_index == video_idx) {
        avcodec_send_packet(video_ctx, pkt);
        while (avcodec_receive_frame(video_ctx, frm) == 0) {
            // frm->data[0] = Y plane, frm->data[1] = U, frm->data[2] = V
            // frm->width, frm->height = frame dimensions
            process_video_frame(frm);
            av_frame_unref(frm);
        }
    }
    av_packet_unref(pkt);
}

// Cleanup
av_frame_free(&frm);
av_packet_free(&pkt);
avcodec_free_context(&audio_ctx);
avcodec_free_context(&video_ctx);
avformat_close_input(&fmt_ctx);
```

### 8.6 FFmpeg Metadata Handling

```bash
# Read all metadata as JSON
ffprobe -v quiet -print_format json -show_format input.webm | jq .format.tags

# Write metadata
ffmpeg -i input.webm \
  -c copy \
  -metadata title="Song Title" \
  -metadata artist="Artist Name" \
  -metadata album="Album Name" \
  output.webm

# Note: Cover art (attachments) are NOT allowed in WebM
# FFmpeg will refuse to write cover art to WebM files

# Strip all metadata
ffmpeg -i input.webm -c copy -map_metadata -1 output.webm
```

#### FFmpeg Internal Metadata Key Mapping
|| Standard Field | FFmpeg Key | WebM Native Key | Notes |
|----------------|------------|--------------------------|-------|
| Title | title | TITLE | |
| Artist | artist | ARTIST | |
| Album | album | ALBUM | |
| Album Artist | album_artist | ALBUMARTIST | |
| Track Number | track | tracknumber | |
| Disc Number | disc | discnumber | |
| Genre | genre | GENRE | |
| Date | date | DATE | |
| Comment | comment | comment | |
| Encoder | encoder | encoder | |

### 8.7 Quality / Fidelity Decision Table
|| Use Case | Settings | Expected Size | Notes |
|----------|----------|---------------|-------|
| Web video (streaming) | VP9, CRF 30–35 | ~1–2 Mbps | YouTube quality |
| High-quality archival | VP9, CRF 20–25 | ~3–5 Mbps | Near-lossless |
| Low bandwidth | VP9, CRF 40–50 + AV1 | ~500 kbps | AV1 more efficient |
| Audio (streaming) | Opus, 64–128 kbps | ~64–128 kbps | Opus preferred |
| Audio (high quality) | Opus, 192–256 kbps | ~192–256 kbps | Transparent |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
```
WebM Cues element:
  Location:     Within Segment (typically near end)
  Element ID:  0x1C53BB6B
  Entry size:  Variable (CuePoint × N)
  
CuePoint structure:
  Offset  Size   Field Name              Type        Description
  ------  -----  ---------------------  ----------  ---------------------------
  0       1B     CuePoint ID            uint8       0xBB
  ?       ?      CueTime               uint        Timestamp (ms)
  ?       1B     CueTrackPositions ID   uint8       0xB7
  ?       ?      CueTrack              uint        Track number
  ?       ?      CueClusterPosition     uint        Byte offset from Segment start
```

### 9.2 Gapless Playback Data
```
WebM/Opus gapless requires:
  - Pre-roll: Opus requires 312 samples of pre-roll at start
  - Codec delay: 5 ms (Opus lookahead)
  - Discard: First 384 samples of decoded output should be discarded
  
OpusHead structure in CodecPrivate:
  - Pre-skip: Number of samples to discard at start (set by encoder)
  - Typically 384 samples at 48 kHz = 8 ms
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

|| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | Chunked encoding supported |
| Algorithmic encoder delay | VP9: ~2 frames; Opus: ~5 ms | Codec-dependent |
| Live encoding feasible | Yes | With `-live` flag |
| HTTP progressive download | Yes | Standard WebM |
| HTTP Live Streaming (HLS) | No | Use WebM in MSE or DASH-IF |
| DASH streaming | Yes | ISOBMFF segment wrapping or WebM DASH |
| WebRTC / RTP transport | Yes | VP8/VP9/Opus are standard WebRTC codecs |
| Media Source Extensions (MSE) | Yes | Direct playback in browser |

### 10.1 WebM Streaming Architecture
```
┌─────────────┐    WebM    ┌──────────────┐    HTML5     ┌────────────┐
│  Encoder    │ ─────────► │   Server     │ ───────────► │  Browser   │
│  (FFmpeg)   │  streaming │  (HTTP/CDN) │  progressive │  <video>   │
└─────────────┘            └──────────────┘              └────────────┘
                                                            │
                                                    ┌──────────────┐
                                                    │ Media Source  │
                                                    │ Extensions    │
                                                    │ (MSE)        │
                                                    └──────────────┘
```

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
|| Channels | Layout Name | Channel Order | Notes |
|----------|-------------|---------------|-------|
| 1 | Mono | C | Opus, Vorbis |
| 2 | Stereo | L, R | Opus, Vorbis |
| 2 | Stereo | L, R | Vorbis (mid-side stereo possible) |
| N | Multichannel | Opus mapping families 0–255 | Opus supports up to 255 channels |

### 11.2 Opus Channel Mappings
| Mapping Family | Channels | Coupled | Example |
|---------------|----------|---------|---------|
| 0 | 1–2 | 0–1 | Mono/Stereo |
| 1 | 1–8 | 0–2 | Surround (5.1, 7.1) |

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

|| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 24-bit (internal) | |
| Max sample rate | 48000 Hz (mandatory), 96000 Hz (Opus) | |
| Float support | Yes (32-bit) | Opus supports float output |
| DSD support | No | Not applicable |

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

|| Platform / API | Encode | Decode | Notes |
|----------------|--------|--------|-------|
| VP9 HW decode | Yes | Yes | Chrome, Edge, Android |
| AV1 HW decode | Yes | Yes | Modern GPUs, Chrome |
| Opus (CPU) | Yes | Yes | Low complexity |
| Chrome (browser) | — | Yes | Native |
| Firefox (browser) | — | Yes | Native |
| Safari (browser) | — | Yes | Since Safari 14 |
| FFmpeg libvpx-vp9 | Yes | Yes | Software |
| FFmpeg libopus | Yes | Yes | CPU-based |

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Encoder-Specific Issues
|| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| WebM muxer does not support cover art | All | Use MKV for files with cover art |
| Opus pre-roll not auto-handled | < 5.0 | Use `-audio_service_type music` |
| VP9 CRF mode quality varies | All | Use two-pass for consistent quality |

### 14.2 Interoperability Issues
- **Safari < 14:** No WebM support before Safari 14
- **Attachments in WebM:** FFmpeg refuses to write attachments to WebM — use MKV instead
- **VP8 in modern browsers:** VP8 support deprecated; use VP9 or AV1
- **Vorbis vs Opus:** Opus is preferred for new content (better quality/efficiency)

### 14.3 Edge Cases to Handle in Converter
- **Opus pre-roll samples:** Discard first 384 samples at 48 kHz
- **VP9 keyframe spacing:** Ensure keyframes at regular intervals for seeking
- **Unknown-size Segment:** WebM can use unknown size for streaming — handle gracefully
- **Missing Cues:** Seeking still works via cluster scan, just slower

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM WebM

|| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → MP4 | `ffmpeg -i in.webm -c copy out.mp4` | Full | Remux only |
| → FLAC | `ffmpeg -i in.webm -vn -c:a flac -compression_level 8 out.flac` | Vorbis/Opus tags | Lossless decode + re-encode |
| → AAC | `ffmpeg -i in.webm -vn -c:a aac -b:a 256k out.m4a` | Partial | Transcode |
| → MP3 | `ffmpeg -i in.webm -vn -c:a libmp3lame -q:a 0 out.mp3` | Partial | Transcode |
| → WAV | `ffmpeg -i in.webm -vn -c:a pcm_s16le out.wav` | None | Lossless decode |

### 15.2 Converting TO WebM

|| Source | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| MP4 → | `ffmpeg -i in.mp4 -c:v libvpx-vp9 -c:a libopus -f webm out.webm` | Full | Transcode |
| FLAC → | `ffmpeg -i in.flac -c:a libopus -f webm out.webm` | Vorbis/Opus tags | Audio only |
| AAC → | `ffmpeg -i in.m4a -c:a libopus -f webm out.webm` | Partial | Audio only |
| MP3 → | `ffmpeg -i in.mp3 -c:a libopus -f webm out.webm` | Partial | Audio only |

### 15.3 Lossless Round-Trip Verification
WebM is a **lossy container** — no lossless round-trip is possible for video/audio. For pure audio:

```bash
# Extract audio to Opus
ffmpeg -i input.flac -c:a libopus -b:a 256k output.webm

# Extract back to WAV
ffmpeg -i output.webm -vn -c:a pcm_s16le decoded.wav

# Compare (Opus is lossy — expect differences)
# Use lossy verification tools like audiotools
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

|| Library | Language | License | Encoder | Decoder | URL |
|---------|----------|---------|---------|---------|-----|
| libvpx | C | BSD | Yes (VP8/VP9) | Yes | https://www.webmproject.org |
| libaom | C | BSD | Yes (AV1) | Yes | https://aomediacodec.github.io |
| libopus | C | BSD | Yes | Yes | https://opus-codec.org |
| libvorbis | C | BSD | Yes | Yes | https://xiph.org/vorbis/ |
| FFmpeg | C | LGPL 2.1+ | Yes | Yes | https://ffmpeg.org |
| libwebm | C++ | BSD | — | Yes | https://github.com/webmproject/libwebm |

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **WebM Container Guidelines:** https://www.webmproject.org/docs/container/
- **RFC 9559 (Matroska):** https://www.ietf.org/rfc/rfc9559.html
- **RFC 8794 (EBML):** https://datatracker.ietf.org/doc/html/rfc8794

### Technical Resources
- FFmpeg muxer options: `ffmpeg -h muxer=webm` or https://ffmpeg.org/ffmpeg-formats.html
- WebM Project: https://www.webmproject.org/
- Multimedia Wiki: https://wiki.multimedia.cx/index.php/WebM
- W3C MSE WebM: https://www.w3.org/TR/2024/NOTE-mse-byte-stream-format-webm-20240718/

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] Verify FFmpeg webm muxer is available (`ffmpeg -muxers | grep webm`)
- [ ] Verify libvpx-vp9, libopus encoders are available
- [ ] Verify webm demuxer is available
- [ ] Note: Cover art (attachments) are NOT supported in WebM

### Encoding Pipeline
- [ ] Use `-f webm` to force WebM container format
- [ ] Set EBML DocType to "webm" automatically
- [ ] For VP9: use `-crf` for VBR quality mode, `-b:v 0`
- [ ] For Opus: use `-b:a` for bitrate or `-vbr on` for VBR
- [ ] For Vorbis: use `-q:a` for quality (0–10)
- [ ] Do NOT attempt to write attachments (cover art) to WebM
- [ ] Set duration correctly in Info element for seeking

### Decoding Pipeline
- [ ] Implement EBML header parsing: verify DocType = "webm"
- [ ] Parse Tracks: build codec table with CodecPrivate
- [ ] Initialize Opus decoder with OpusHead + OpusTags
- [ ] Initialize Vorbis decoder with all three headers
- [ ] Parse Cues for O(log N) seeking
- [ ] Handle Opus pre-roll: discard first 384 samples
- [ ] Handle unknown-size Segment for streaming

### Metadata
- [ ] Read Tags element for standard metadata
- [ ] Write Tags element for output
- [ ] Note: Cannot write cover art to WebM — inform user
- [ ] Handle UTF-8 encoding throughout

### Quality & Verification
- [ ] Use CRF mode for consistent quality
- [ ] Two-pass encoding for better quality/size trade-off
- [ ] Test with various VP9 profiles (0–3)
- [ ] Test Opus at various bitrates

### Edge Cases
- [ ] Handle WebM with unknown-size Segment (streaming mode)
- [ ] Handle WebM without Cues element (seek by cluster scan)
- [ ] Handle VP9 keyframes correctly for seeking
- [ ] Handle Opus pre-roll samples correctly
- [ ] Handle files with multiple audio tracks (use first audio)

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
