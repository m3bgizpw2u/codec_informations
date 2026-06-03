# Apple Core Audio Format (CAF) — Deep Technical Reference
> **Category:** Container
> **File Extensions:** `.caf`
> **MIME Types:** `audio/x-caf`, `audio/caf`
> **Standardization Body:** Apple Inc.
> **Primary Specification:** Core Audio Format Specification 1.0 (Apple Developer Documentation)
> **Patent Status:** Proprietary Apple technology
> **License:** Proprietary
> **Current Version:** 1.0
> **Active Development:** Limited — Apple still uses CAF internally; public specification has not been updated significantly

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Apple Inc.
- **Year Created:** 2004 (with Mac OS X Tiger / Core Audio framework)
- **Original Purpose:** Provide a professional-grade, extensible audio container for high-resolution audio applications, addressing limitations of the older AIFF/WAV formats — specifically arbitrary sample rates, 64-bit file sizes, and unlimited metadata
- **Problem with Predecessors:** AIFF/WAV formats were limited to 32-bit file sizes (4 GB), had fixed byte ordering constraints, and lacked extensible metadata. CAF was designed as a "super-AIFF" for professional audio workflows.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| 1.0 | 2004 | Initial release with Mac OS X Tiger |
| 1.0 (revised) | 2007–2011 | Minor clarifications in documentation |

### 1.3 Current Adoption
- **Primary use cases today:** Professional audio (Logic Pro, Final Cut Pro, MainStage), high-resolution audio archival, iOS/macOS system sounds, audiobook production
- **Platforms with native support:** macOS (native via AudioToolbox/AVFoundation), iOS, watchOS, tvOS
- **Major services using this format:** Apple professional software ecosystem, some mastering studios
- **Hardware support:** Apple hardware and software; limited third-party support
- **Status:** Niche / professional — widely used in Apple Pro apps but rare elsewhere

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 CAF Architecture
CAF (Core Audio Format) is a **chunk-based container** similar to AIFF/WAV but with 64-bit addressing, arbitrary byte ordering (except audio data), and a richer set of metadata chunk types.

```
CAF File:
  [File Header]              ← 32 bytes: magic, version, flags, chunk count
  [Chunk 1]                 ← Audio Description (kuki) — required, first
      ├── Chunk Header (12 bytes): type + size + data
      └── CAFAudioDescription structure
  [Chunk 2]                 ← Audio Data (data) — required
      ├── Chunk Header
      └── Audio packets (in format specified by description)
  [Chunk 3..N]              ← Optional chunks:
      ├── Packet Table (pak)       — for VBR audio
      ├── Strings (strg)           — localized strings
      ├── Annotation (anns)        — annotations
      ├── Comments (cmt )          — freeform comments
      ├── Info (info)              — standard metadata
      ├── Instrument (inst)        — instrument settings
      ├── Marker (mark)           — named positions
      ├── Region (regn)           — audio regions
      ├── Peak (peak)              — loudness data
      ├── UUID (uuid)             — user-defined data
      └── Label (labl)            — labels
```

### 2.2 Key Design Parameters
|| Parameter | Value | Notes |
|-----------|-------|-------|
| File size limit | 2^63 bytes | 64-bit addressing throughout |
| Chunk count | Unlimited | Any number of chunk types |
| Required chunks | 2 | Audio Description + Audio Data |
| Sample rate | Arbitrary | Not limited to standard rates |
| Channel count | Arbitrary | Limited by codec |
| Endianness | Big-endian | All fields except audio data |
| Audio data endianness | Per-format | Configurable in description |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       63 61 66 66     caff     CAF file magic ("caff" in ASCII)
0x0004  2       00 01           ..       File version = 1 (always 1 for v1.0)
0x0006  2       00 00           ..       File flags = 0 (none defined)
0x0008  8       xx xx xx xx     ....      Total chunks count (uint64 BE) — 0 if unknown
```

### 3.2 File Header Layout
```
Offset  Size   Field Name              Type        Valid Range   Description
------  -----  ---------------------  ----------  ------------  ---------------------------
0x0000  4B     File Magic              char[4]     "caff"        CAF magic number
0x0004  2B     File Version            uint16 BE   1             CAF version (always 1)
0x0006  2B     File Flags              uint16 BE   0             Reserved (must be 0)
0x0008  8B     Chunk Count             uint64 BE   0–N          Number of chunks (0=unknown)
```

### 3.3 Chunk Structure
Every chunk follows this format:

```
Offset  Size   Field Name              Type        Description
------  -----  ---------------------  ----------  ---------------------------
0       4B     Chunk Type              char[4]     Four-character code (e.g., "kuki", "data")
8       8B     Chunk Data Size         int64 BE    Size in bytes; -1 = unknown (streaming)
16      N      Chunk Data              bytes       Actual chunk content
--- Next chunk follows immediately ---
```

### 3.4 Required Chunks

#### Audio Description Chunk (kuki)
The `kuki` chunk is required and must be the first chunk after the file header. It contains the `CAFAudioDescription` structure.

```
Offset  Size   Field Name              Type        Valid Range   Description
------  -----  ---------------------  ----------  ------------  ---------------------------
0       8B     Chunk Data Size         int64 BE   32            Must be 32 bytes (sizeof CAFAudioDescription)
0       8B     mSampleRate             Float64 BE any           Samples per second (e.g., 44100.0, 96000.0)
8       4B     mFormatID               uint32 BE  any           Codec format ID (see below)
12      4B     mFormatFlags            uint32 BE  varies        Codec-specific flags
16      4B     mBytesPerPacket         uint32 BE  varies        Bytes per packet (0=VBR)
20      4B     mFramesPerPacket         uint32 BE  varies        Frames per packet (1 for PCM)
24      4B     mChannelsPerFrame        uint32 BE  1–N           Channels per frame
28      4B     mBitsPerChannel         uint32 BE  0–64          Bits per channel (0=variable)
```

**mFormatID — Known Format Codes:**
| Format ID | Name | Description | Bytes/Packet | Notes |
|-----------|------|-------------|-------------|-------|
| `0x6C70636D` (`lpcm`) | Linear PCM | Uncompressed PCM | Variable | Most common |
| `0x696D6134` (`ima4`) | IMA 4:1 ADPCM | Compressed ADPCM | 136 per 80 samples | |
| `0x61616320` (`aac `) | AAC | MPEG-4 AAC | Variable | With space |
| `0x4D414333` (`MAC3`) | MACE 3:1 | Compressed | | Obsolete |
| `0x4D414336` (`MAC6`) | MACE 6:1 | Compressed | | Obsolete |
| `0x756C6177` (`ulaw`) | μ-law | Companded | 1 per sample | |
| `0x616C6177` (`alaw`) | A-law | Companded | 1 per sample | |
| `0x2E6D7031` (`.mp1`) | MPEG-1 Layer I | Compressed | Variable | |
| `0x2E6D7032` (`.mp2`) | MPEG-1 Layer II | Compressed | Variable | |
| `0x2E6D7033` (`.mp3`) | MPEG-1 Layer III | Compressed | Variable | |
| `0x616C6163` (`alac`) | Apple Lossless | Lossless | Variable | |

**lpcm (Linear PCM) Format Flags:**
| Flag Value | Meaning |
|-----------|---------|
| 0x01 | Float (IEEE 754) — else integer |
| 0x02 | Little-endian — else big-endian |
| 0x04 | Packed (all bits used) — else unpacked (aligned to byte boundary) |

### 3.5 Audio Data Chunk (data)
```
Offset  Size   Field Name              Type        Description
------  -----  ---------------------  ----------  ---------------------------
0       8B     Chunk Data Size         int64 BE   Size in bytes of audio data; -1 = unknown
16      N      Audio Data             bytes       PCM/compressed samples
```

If chunk size is -1 (unknown), the data chunk must be the **last chunk in the file**. The audio data extends to the end of the file.

### 3.6 Optional Chunks

#### Packet Table Chunk (pak)
Required for variable bitrate (VBR) audio formats. Contains an array of packet offsets and sizes.

```
Offset  Size   Field Name              Type        Valid Range   Description
------  -----  ---------------------  ----------  ------------  ---------------------------
0       8B     Chunk Data Size         int64 BE   varies        Size of packet table
0       4B     mNumberPackets          uint64 BE  1–N           Total number of packets
8       4B     mNumberFrames           uint64 BE  1–N           Total frames (samples/channels)
16      4B     mSampleRate             Float64 BE any           Sample rate for time calculations
24      N      Packet Table            array       N × 24 bytes  See below
```

**Packet Table Entry (24 bytes each):**
```
Offset  Size   Field Name              Type        Description
------  -----  ---------------------  ----------  ---------------------------
0       8B     mPacketInfo            uint64 BE   Flags (reserved)
8       8B     mFrameOffset           uint64 BE   Byte offset of packet in audio data
16      8B     mPacketSize            uint64 BE   Size of this packet in bytes
```

#### Strings Chunk (strg)
```
Offset  Size   Field Name              Type        Description
------  -----  ---------------------  ----------  ---------------------------
0       4B     Number of strings       uint32 BE   Count of string entries
--- For each string entry ---
0       4B     String ID               uint32 BE   Identifies the string
4       4B     String length          uint32 BE   Length of string in bytes
8       N      String data             UTF-8       The actual string
```

#### Peak Chunk (peak)
Contains loudness/lpeak data for each channel:
```
Offset  Size   Field Name              Type        Description
------  -----  ---------------------  ----------  ---------------------------
0       4B     Version                 uint32 BE   Currently 0
4       4B     Number of channels       uint32 BE   Channel count
8       4B     Number of peaks          uint32 BE   Number of peaks (= channels)
12      N      Peak data                float32[N]  Peak value per channel
```

#### Marker Chunk (mark)
```
Offset  Size   Field Name              Type        Description
------  -----  ---------------------  ----------  ---------------------------
0       4B     Number of markers        uint32 BE   Count of markers
--- For each marker ---
0       4B     Marker ID                uint32 BE   Unique identifier
4       4B     Frame position           uint32 BE   Frame number
8       4B     Marker length            uint32 BE   Length of marker text
12      N      Marker text               UTF-8       Name/label for marker
```

#### Info Chunk (info)
Contains standard metadata as key-value pairs:
```
Offset  Size   Field Name              Type        Description
------  -----  ---------------------  ----------  ---------------------------
--- For each key-value pair ---
0       4B     Key length               uint32 BE   Length of key string
4       N      Key string               UTF-8       Key name (e.g., "©alb", "©ART")
N+4     4B     Value length             uint32 BE   Length of value string
N+8     M      Value string             UTF-8       Value content
```

**Standard Info Keys:**
| Key | Meaning | Type |
|-----|---------|------|
| `©alb` | Album | Text |
| `©ART` | Artist | Text |
| `©day` | Creation date | Text |
| `©nam` | Title | Text |
| `©gen` | Genre | Text |
| `©day` | Date | Text |
| `©cmt` | Comment | Text |

### 3.7 Sample Format Support
|| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Unsigned integer | Yes | |
| 16-bit | Signed integer | Yes | Standard PCM |
| 20-bit | Signed integer | Yes | Unpacked or packed |
| 24-bit | Signed integer | Yes | Common hi-res |
| 32-bit | Signed integer | Yes | |
| 32-bit | IEEE float | Yes | Float32 |
| 64-bit | IEEE float | Yes | Float64 [NEEDS VERIFICATION] |

### 3.8 Supported Sample Rates
|| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| Any integer | — | Yes | CAF allows arbitrary sample rates |
| 8000 | Telephone | Yes | |
| 11025 | — | Yes | |
| 16000 | Wideband | Yes | |
| 22050 | — | Yes | |
| 32000 | Broadcast | Yes | |
| 44100 | CD audio | Yes | Most common |
| 48000 | Professional | Yes | Standard |
| 88200 | 2× CD | Yes | |
| 96000 | High-res | Yes | |
| 176400 | 4× CD | Yes | |
| 192000 | High-res max | Yes | |
| 352800 | DXD | Yes | |
| 384000 | Ultra-high-res | Yes | |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Encoding Pipeline (for lpcm)
```
Input PCM Samples
      │
      ▼
[Byte Order Conversion — if needed]
Convert to desired endianness (big-endian by default for CAF)
      │
      ▼
[Sample Format Packing — for non-byte-aligned bit depths]
Pack 20-bit or 24-bit samples into minimal bytes
      │
      ▼
[Channel Interleaving — for multi-channel]
Interleave samples from all channels into frames
      │
      ▼
[Write Audio Data Chunk]
Write packets to data chunk, maintaining packet boundaries
      │
      ▼
[Write Packet Table — for VBR]
Track offset and size of each packet
      │
      ▼
Output: CAF File
```

### 4.2 Sample Packing (Unpacked vs. Packed PCM)
**Unpacked (default):** Each sample occupies at least `mBitsPerChannel` bits, padded to the next byte boundary:

```
Unpacked 20-bit example:
  [SSSS SSSS SSSS SSSS SSSS ----] [---- SSSS SSSS SSSS SSSS SSSS] ...
  Each sample uses 24 bits (3 bytes), even though only 20 bits are used.
```

**Packed (flag 0x04):** Samples are packed with no wasted bits:

```
Packed 20-bit example (5 samples × 20 bits = 100 bits = 13 bytes):
  [SSSS SSSS SSSS SSSS SSSS] [SSSS SSSS SSSS SSSS SSSS] [SSSS SSSS --]
  Total: 13 bytes for 5 samples (no byte padding within channel)
```

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. Read CAF file header (32 bytes)
2. Verify magic = "caff" and version = 1
3. Read chunk count or scan until EOF
4. Find Audio Description chunk (kuki) — must be first after header
5. Parse CAFAudioDescription for format parameters
6. Find Audio Data chunk (data)
7. If mBytesPerPacket = 0: Read Packet Table chunk (pak) for VBR offsets
8. Seek to audio data offset
9. Read and decode audio packets
```

#### Seeking (for VBR audio)
With a Packet Table chunk, seeking is O(1) using binary search on packet offsets:

```
1. Binary search packet table: find entry where mFrameOffset ≤ target_frame
2. Seek to mFrameOffset in audio data
3. Decode from that packet forward to target frame
```

Without a packet table, seeking requires linear scan.

### 5.2 Core Decode Pipeline
```
1. Read and validate CAF header
2. Parse Audio Description chunk (kuki):
   ├── Extract mSampleRate
   ├── Extract mFormatID → determine codec
   ├── Extract mFormatFlags → determine endianness, float/int
   ├── Determine bytes_per_sample
   └── If VBR: parse Packet Table chunk
3. Locate Audio Data chunk
4. For each packet:
   ├── Read packet_size bytes from audio data
   ├── Decode based on mFormatID:
   │   ├── lpcm: no decoding (raw PCM)
   │   ├── alac: Apple Lossless decode
   │   ├── aac: AAC decode
   │   └── etc.
   └── If VBR: advance by packet_size from table
5. Apply endianness conversion if needed
6. Apply float/int conversion if needed
7. Output PCM samples
```

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** CAF (self-contained)
- **Overhead:** ~0.01–0.1% (chunk headers)
- **Seeking in native container:** Yes — via Packet Table for VBR
- **Multiple streams in native container:** No — single audio stream

### 6.2 Codec-to-Container Compatibility
|| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| CAF (native) | Yes | Yes (packet table) | Full (info, strg) | Preferred |
| AIFF | Yes (audio only) | Limited | Partial | Via AudioToolbox |
| WAV | Yes (audio only) | Limited | Partial | Via AudioToolbox |
| MP4/M4A | Partial | Yes | Full | CAF preferred for pro audio |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** CAF Info chunk (keyed text metadata) + Strings chunk + Annotation chunk
- **Tag block location:** Info chunks within CAF file
- **Tag block identifier:** "info" chunk type code

### 7.2 Standard Tag Fields — Complete Reference
|| Tag Field | Internal Key (CAF) | Max Length | Character Encoding | Multi-value | Notes |
|-----------|---------------------------|------------|-------------------|-------------|---|
| Title | `©nam` | unlimited | UTF-8 | No | Info chunk |
| Artist | `©ART` | unlimited | UTF-8 | Yes | Info chunk |
| Album | `©alb` | unlimited | UTF-8 | No | Info chunk |
| Composer | `©wrt` | unlimited | UTF-8 | Yes | Info chunk |
| Genre | `©gen` | unlimited | UTF-8 | No | Info chunk |
| Date | `©day` | unlimited | UTF-8 | No | Info chunk |
| Comment | `©cmt` | unlimited | UTF-8 | Yes | Info chunk |
| Copyright | `©cpy` | unlimited | UTF-8 | No | Info chunk |
| Album Artist | `©aART` | unlimited | UTF-8 | No | Info chunk |
| Track Number | `©trk` | unlimited | UTF-8 | No | Info chunk |
| Disc Number | `©dsk` | unlimited | UTF-8 | No | Info chunk |
| Encoder | `©too` | unlimited | UTF-8 | No | Info chunk |
| Cover Art | (not native) | — | — | — | Use external file |

### 7.3 Cover Art Storage
CAF does **not** have a native cover art mechanism. Cover art is typically stored as:
- Separate image file alongside the CAF
- Embedded in a custom UUID chunk
- Converted to a format that supports embedded art

### 7.4 Metadata Compatibility Matrix
|| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| CAF Info chunk | ✓ | ✓ | ✓ | Highest (native) |
| CAF Strings chunk | ✓ | ✓ | ✓ | High |
| CAF Annotation | ✓ | ✓ | ✓ | Medium |
| ID3v2 | ✗ | ✗ | N/A | Not supported |
| APEv2 | ✗ | ✗ | N/A | Not supported |

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   alac (Apple Lossless)       # used with -c:a
AV_CODEC_ID:        AV_CODEC_ID_ALAC
Format Name (CLI):   caf                         # used with -f
Encoder(s):          alac (via caf muxer)        # ffmpeg -encoders | grep -i caf
Decoder(s):          alac (via caf demuxer)      # ffmpeg -decoders | grep -i caf
Muxer(s):           caf                         # ffmpeg -muxers | grep -i caf
Demuxer(s):          caf                         # ffmpeg -demuxers | grep -i caf
```

### 8.2 FFmpeg Encoding — Full CLI Reference

```bash
# Encoding to CAF with Apple Lossless
ffmpeg -i input.wav \
  -c:a alac \
  -f caf \
  output.caf

# Encoding to CAF with linear PCM (24-bit/96kHz)
ffmpeg -i input_24bit_96k.wav \
  -c:a pcm_s24be \                    # Signed 24-bit big-endian
  -f caf \
  output.caf

# Encoding with specific metadata
ffmpeg -i input.wav \
  -c:a alac \
  -f caf \
  -metadata title="Song Title" \
  -metadata artist="Artist Name" \
  -metadata album="Album Name" \
  output.caf

# CAF muxer options
ffmpeg -i input.wav \
  -c:a alac \
  -f caf \
  -caf_options ... \                   # Note: limited CAF-specific options in FFmpeg
  output.caf
```

#### Complete FFmpeg Option Table
|| Option | Type | Default | Range / Valid Values | Effect on Output |
|--------|------|---------|---------------------|-----------------|
| `-c:a` | string | — | alac, pcm_s16be, pcm_s24be, pcm_s32be | Audio codec |
| `-ar` | int | from input | any | Output sample rate |
| `-ac` | int | from input | 1–N | Output channel count |
| `-sample_fmt` | string | s16 | s16, s32, fltp | Sample format |

### 8.3 FFmpeg Encoding — libavcodec C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/opt.h>
#include <libswresample/swresample.h>

// ─── 1. Find encoder ─────────────────────────────────────────────────────────
const AVCodec *codec = avcodec_find_encoder(AV_CODEC_ID_ALAC);
if (!codec) { fprintf(stderr, "Encoder not found\n"); exit(1); }

// ─── 2. Allocate codec context ───────────────────────────────────────────────
AVCodecContext *ctx = avcodec_alloc_context3(codec);
if (!ctx) { fprintf(stderr, "OOM\n"); exit(1); }

// ─── 3. Set required parameters ──────────────────────────────────────────────
ctx->sample_fmt   = AV_SAMPLE_FMT_S16;    // ALAC supports S16, S32
ctx->sample_rate  = 44100;
av_channel_layout_copy(&ctx->ch_layout, &(AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO);

// ─── 4. Open codec ───────────────────────────────────────────────────────────
int ret = avcodec_open2(ctx, codec, NULL);
if (ret < 0) { /* handle error */ exit(1); }

// ─── 5. Allocate frame and packet ────────────────────────────────────────────
AVFrame  *frame = av_frame_alloc();
frame->format      = ctx->sample_fmt;
frame->sample_rate = ctx->sample_rate;
av_channel_layout_copy(&frame->ch_layout, &ctx->ch_layout);
frame->nb_samples  = ctx->frame_size;   // ALAC: typically 4096
av_frame_get_buffer(frame, 0);

AVPacket *pkt = av_packet_alloc();

// ─── 6. Encode loop ─────────────────────────────────────────────────────────
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

// ─── 8. Cleanup ─────────────────────────────────────────────────────────────
av_frame_free(&frame);
av_packet_free(&pkt);
avcodec_free_context(&ctx);
```

### 8.4 FFmpeg Decoding — Full CLI Reference

```bash
# Decode CAF to WAV
ffmpeg -i input.caf \
  -c:a pcm_s16le \
  output.wav

# Decode with specific format
ffmpeg -i input.caf \
  -c:a pcm_s24be \
  output_24bit.wav

# Extract audio without conversion
ffmpeg -i input.caf -c:a copy output.caf

# Probe format
ffprobe -v quiet -print_format json -show_streams -show_format input.caf
```

### 8.5 FFmpeg Decoding — libavcodec C API

```c
// ─── Complete decode program skeleton ─────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.caf", NULL, NULL);
avformat_find_stream_info(fmt_ctx, NULL);

int audio_idx = av_find_best_stream(fmt_ctx, AVMEDIA_TYPE_AUDIO, -1, -1, NULL, 0);
AVStream *stream = fmt_ctx->streams[audio_idx];

const AVCodec *dec = avcodec_find_decoder(stream->codecpar->codec_id);
AVCodecContext *dec_ctx = avcodec_alloc_context3(dec);
avcodec_parameters_to_context(dec_ctx, stream->codecpar);
avcodec_open2(dec_ctx, dec, NULL);

AVPacket *pkt = av_packet_alloc();
AVFrame  *frm = av_frame_alloc();

while (av_read_frame(fmt_ctx, pkt) >= 0) {
    if (pkt->stream_index == audio_idx) {
        avcodec_send_packet(dec_ctx, pkt);
        while (avcodec_receive_frame(dec_ctx, frm) == 0) {
            process_audio_frame(frm);
            av_frame_unref(frm);
        }
    }
    av_packet_unref(pkt);
}

avcodec_send_packet(dec_ctx, NULL);
while (avcodec_receive_frame(dec_ctx, frm) == 0) {
    process_audio_frame(frm);
    av_frame_unref(frm);
}

av_frame_free(&frm);
av_packet_free(&pkt);
avcodec_free_context(&dec_ctx);
avformat_close_input(&fmt_ctx);
```

### 8.6 FFmpeg Metadata Handling

```bash
# Read all metadata as JSON
ffprobe -v quiet -print_format json -show_format input.caf | jq .format.tags

# Write metadata (CAF info chunks)
ffmpeg -i input.caf \
  -c:a copy \
  -metadata title="Song Title" \
  -metadata artist="Artist Name" \
  -metadata album="Album Name" \
  output.caf

# Strip all metadata
ffmpeg -i input.caf -c:a copy -map_metadata -1 output.caf
```

#### FFmpeg Internal Metadata Key Mapping
|| Standard Field | FFmpeg Key | CAF Native Key | Notes |
|----------------|------------|--------------------------|-------|
| Title | title | `©nam` | |
| Artist | artist | `©ART` | |
| Album | album | `©alb` | |
| Composer | composer | `©wrt` | |
| Genre | genre | `©gen` | |
| Date | date | `©day` | |
| Comment | comment | `©cmt` | |
| Copyright | copyright | `©cpy` | |
| Encoder | encoder | `©too` | |

### 8.7 Quality / Fidelity Decision Table
|| Use Case | Settings | Expected Size | Notes |
|----------|----------|---------------|-------|
| Archival / best quality | `-c:a alac -f caf` | ~0.55–0.65× uncompressed | Apple Lossless |
| Linear PCM archival | `-c:a pcm_s24be -f caf` | 1.0× uncompressed | No compression |
| Standard use | `-c:a alac -f caf` | ~0.60× uncompressed | Default |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
```
CAF Packet Table (pak chunk):
  Location:     After Audio Data chunk (for VBR audio)
  Magic:       "pak" chunk type code
  Entry size:  24 bytes per packet
  Entry format:
    [0x00–0x07]  mPacketInfo    (uint64 BE, flags)
    [0x08–0x0F]  mFrameOffset   (uint64 BE) — byte offset in audio data
    [0x10–0x17]  mPacketSize    (uint64 BE) — packet size in bytes
  Max entries:  Limited by chunk size
```

### 9.2 Gapless Playback Data
```
CAF does not store explicit gapless metadata.
Encoder delay:  Varies by codec (ALAC: ~0 samples)
Padding:        Varies by implementation
Storage:        Not standardized in CAF
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

|| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | If audio data chunk size is known |
| Algorithmic encoder delay | 0 samples | For lpcm; varies for compressed codecs |
| Live encoding feasible | Yes | With unknown chunk size (size = -1) |
| HTTP progressive download | Yes | |
| HTTP Live Streaming (HLS) | No | Not natively supported |
| DASH streaming | No | Not natively supported |
| WebRTC / RTP transport | No | Not natively supported |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
|| Channels | Layout Name | Channel Order | Notes |
|----------|-------------|---------------|-------|
| 1 | Mono | C | |
| 2 | Stereo | L, R | |
| 3 | 2.1 | L, R, LFE | |
| 4 | Quad | FL, FR, RL, RR | |
| 5 | 5.0 | FL, FR, C, SL, SR | |
| 6 | 5.1 | FL, FR, C, LFE, SL, SR | |
| 8 | 7.1 | FL, FR, C, LFE, BL, BR, SL, SR | |

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

|| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 32-bit (lpcm) | Can be any value up to 32 |
| Max sample rate | Arbitrary | No fixed limit; 2^53 Hz theoretical |
| Float support | Yes | 32-bit and 64-bit IEEE float |
| DSD support | No | Not natively supported |
| 20-bit support | Yes | High-res standard |
| 24-bit support | Yes | Common hi-res format |

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

|| Platform / API | Encode | Decode | Notes |
|----------------|--------|--------|-------|
| Apple AudioToolbox | Yes | Yes | Native macOS/iOS |
| Apple AVFoundation | Yes | Yes | Native macOS/iOS |
| FFmpeg native | Yes | Yes | Via libavformat/cafdec.c |
| Core Audio (iOS) | Yes | Yes | Native iOS |
| Android MediaCodec | No | No | CAF not supported |

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Encoder-Specific Issues
|| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| Limited CAF muxer options | All | Use AudioToolbox for full control |
| Packet table not auto-generated | All | Manually create if needed |
| Cover art not supported | All | Use separate image file |

### 14.2 Interoperability Issues
- **Non-Apple platforms:** Most non-Apple software cannot read CAF files
- **lpcm byte ordering:** Must specify correct endianness; wrong endianness produces noise
- **Packed vs. unpacked PCM:** Some software assumes unpacked; data sounds corrupted if packed
- **Unknown chunk sizes (size = -1):** Requires streaming to end-of-file

### 14.3 Edge Cases to Handle in Converter
- **Empty file / 0-sample stream:** Valid CAF with no audio data; skip gracefully
- **CAF with unknown chunk count:** Scan until EOF
- **Audio data chunk with size = -1:** Data extends to end of file
- **All-silence audio:** Valid; very small file
- **DC offset:** CAF lpcm preserves DC exactly
- **Full-scale sine (0 dB):** No clipping for PCM; codec-dependent for compressed
- **File with corrupt chunk:** Skip unknown chunks; use known chunks
- **Truncated file:** Stop at last complete packet

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM CAF

|| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → FLAC | `ffmpeg -i in.caf -c:a flac -compression_level 8 out.flac` | CAF info → Vorbis | Lossless if source was lossless |
| → WAV | `ffmpeg -i in.caf -c:a pcm_s16le out.wav` | Limited | Lossless if source was PCM |
| → ALAC | `ffmpeg -i in.caf -c:a alac out.m4a` | Via -metadata | Lossless if source was ALAC |
| → MP3 | `ffmpeg -i in.caf -c:a libmp3lame -q:a 0 out.mp3` | CAF info → ID3v2.3 | Generation loss |
| → AAC | `ffmpeg -i in.caf -c:a aac -b:a 256k out.m4a` | Via -metadata | Generation loss |

### 15.2 Converting TO CAF

|| Source | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| FLAC → | `ffmpeg -i in.flac -c:a alac -f caf out.caf` | Vorbis → CAF info | Lossless |
| WAV → | `ffmpeg -i in.wav -c:a alac -f caf out.caf` | RIFF INFO → CAF info | Lossless |
| MP3 → | `ffmpeg -i in.mp3 -c:a alac -f caf out.caf` | ID3v2 → CAF info | Lossless (re-encode) |
| ALAC → | `ffmpeg -i in.m4a -c:a alac -f caf out.caf` | Via -metadata | Lossless (re-encode) |

### 15.3 Lossless Round-Trip Verification
```bash
# Encode
ffmpeg -i original.wav -c:a alac -f caf output.caf

# Decode back
ffmpeg -i output.caf -c:a pcm_s16le decoded.wav

# Compare checksums
ffmpeg -i original.wav -c:a pcm_s16le original_raw.wav
md5sum original_raw.wav decoded_raw.wav   # Must match for true lossless
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

|| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| Core Audio (Apple) | C/C++ | Proprietary | Reference | Reference | Apple Developer |
| AudioToolbox | C/C++ | Proprietary | Reference | Reference | Apple Developer |
| FFmpeg native | C | LGPL 2.1+ | 8/10 | 9/10 | https://ffmpeg.org |

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **Core Audio Format Specification 1.0:** https://developer.apple.com/library/archive/documentation/MusicAudio/Reference/CAFSpec/CAF_spec/CAF_spec.html

### Technical Resources
- FFmpeg muxer options: `ffmpeg -h muxer=caf` or https://ffmpeg.org/ffmpeg-formats.html
- Apple Developer: Core Audio File Format — https://developer.apple.com/documentation/audiotoolbox/core_audio_file_format
- Multimedia Wiki: https://wiki.multimedia.cx/index.php/CAF

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] Verify FFmpeg CAF muxer/demuxer is available
- [ ] Verify ALAC encoder/decoder is available in FFmpeg
- [ ] Test with various bit depths (16, 20, 24, 32-bit)

### Encoding Pipeline
- [ ] Convert input sample format using libswresample
- [ ] Set CAF chunk headers correctly (big-endian throughout)
- [ ] Write Audio Description chunk (kuki) first after header
- [ ] Write Audio Data chunk with correct byte ordering
- [ ] Write Packet Table chunk for VBR audio
- [ ] Write Info chunks for metadata

### Decoding Pipeline
- [ ] Implement CAF magic verification ("caff")
- [ ] Parse Audio Description chunk to determine format
- [ ] Handle both big-endian and little-endian audio data
- [ ] Handle packed vs. unpacked PCM formats
- [ ] Parse Packet Table chunk for VBR seeking
- [ ] Handle unknown chunk data sizes (size = -1)

### Metadata
- [ ] Read CAF Info chunks (keyed UTF-8 metadata)
- [ ] Read Strings chunk for localized metadata
- [ ] Read Annotation chunk
- [ ] Write Info chunks with standard keys
- [ ] Handle UTF-8 encoding throughout
- [ ] Note: Cover art not natively supported in CAF

### Edge Cases
- [ ] Handle CAF files with unknown chunk count
- [ ] Handle audio data with size = -1 (streaming to EOF)
- [ ] Handle various sample formats (integer/float, packed/unpacked)
- [ ] Handle various byte orders (big/little endian)

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
