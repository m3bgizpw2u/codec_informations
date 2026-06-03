# WavPack (.WV) — Deep Technical Reference
> **Category:** Container
> **File Extensions:** `.wv`, `.wvc`
> **MIME Types:** `audio/x-wavpack`, `audio/x-wavpack-correction`
> **Standardization Body:** Public specification (WavPack 5.0 specification)
> **Primary Specification:** WavPack 5 File Format Specification (https://www.wavpack.com/WavPack5FileFormat.pdf)
> **Patent Status:** Patent-free — no known patent claims
> **License:** BSD / Open source
> **Current Version:** 5.0 (since 2010)
> **Active Development:** Yes — last release 2024 (WavPack 5.7.x)

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** David Bryant, Conifer Software
- **Year Created:** 1998 (initial release), 5.0 major revision in 2010
- **Original Purpose:** Create a high-quality, open-source, hybrid lossless audio codec that could compete with proprietary solutions while offering unique hybrid compression (lossy + correction file)
- **Problem with Predecessors:** Other lossless codecs lacked hybrid mode, were too slow for real-time encoding, or had restrictive licensing. WavPack's hybrid mode was a unique innovation enabling efficient lossy previews with bit-perfect correction capability.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| 1.0 | 1998 | Initial release |
| 2.0 | 2000 | Improved compression, faster decoding |
| 3.0 | 2002 | Added hybrid mode, APEv2 tagging |
| 4.0 | 2006 | Complete format redesign, block-based structure |
| 4.4 | 2007 | Multichannel support, improved hybrid |
| 5.0 | 2010 | New block format, MD5 checksums, DSD support, improved float |
| 5.6+ | 2019–2024 | Performance improvements, bug fixes |

### 1.3 Current Adoption
- **Primary use cases today:** Music archival, game audio (Rockbox firmware uses WavPack), streaming (via Matroska), professional audio
- **Platforms with native support:** Rockbox firmware, foobar2000, DBPowerAmp, JRiver Media Center, VLC (via FFmpeg)
- **Major services using this format:** Bandcamp (for audio), archival institutions
- **Hardware support:** Rockbox-based DAPs (iRiver, Sansa Clip, etc.), some studio DAWs
- **Status:** Niche but well-regarded; dedicated user community

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 WavPack Architecture
WavPack stores audio in a **sequence of independent blocks** (since version 4.0). Each block is self-contained with a 32-byte header followed by metadata sub-chunks and audio data. This block-based structure enables:
- Streaming playback (decode each block independently)
- Fast seeking (seek to block, decode forward)
- Hybrid mode (lossy block + correction block)
- Editable APEv2 tags

### 2.2 WavPack File Structure
```
WV File:
  [Block 1]  ← First block: contains audio description
      ├── Block Header (32 bytes) — wvpk magic + metadata
      ├── Metadata Sub-Chunks (ID + size + data)
      └── Audio Data (WV_BITSTREAM or WVC_BITSTREAM)
  [Block 2]  ← Subsequent audio blocks
      ├── Block Header
      ├── Metadata Sub-Chunks
      └── Audio Data
  ...
  [Block N]  ← Final block may contain APEv2 tags
  [APEv2 Tags] (optional, at end of file)
```

### 2.3 Key Design Parameters
|| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | 0 samples | No lookahead required |
| Block size | ~0.5 seconds (configurable) | Typically 1/75–1/30 of a second per block |
| Max channels | 8 | Linear ordering |
| Max bit depth | 32-bit integer, 32-bit float | |
| Max sample rate | 2 MHz [NEEDS VERIFICATION] | |
| Bitrate range | ~300–1400 kbps (lossless) | Content-dependent |
| Compression modes | Lossless, Hybrid, High/Speed modes | |
| Complexity | Encode: high; Decode: low | Asymmetric codec |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       77 76 70 6B      wvpk     WavPack block header magic
```

The `wvpk` magic (0x7776706B) appears at the start of **every block** in a WavPack file. A valid .WV file contains one or more `wvpk` blocks.

### 3.2 WavPack Block Header (32 bytes — always little-endian)
```
Offset  Size   Field Name              Type        Valid Range   Description
------  -----  ---------------------  ----------  ------------  ---------------------------
0x0000  4B     Block ID                char[4]     "wvpk"       Magic number
0x0004  2B     Block version           uint16 LE   0x402–0x50B  Version (0x402=4.02, 0x500=5.0)
0x0006  2B     Track number            uint16 LE   0–255         Track index for multi-file
0x0008  2B     Index flags             uint16 LE   bitfield      Block position info
0x000A  8B     Block samples           uint64 LE   0–N           Samples in this block
0x0012  8B     CRC of original         uint64 LE   any           For verification
0x001A  8B     Data size               uint64 LE   0–N           Total block data size (after header)
```

**Index flags (offset 0x0008) — Bit assignments:**
```
Bit  0:  Initial block — 1=first block of a sequence
Bit  1:  Final block — 1=last block of a sequence
Bits 2–15: Reserved
```

### 3.3 Block Flags (stored in metadata sub-chunks, not header)
The actual compression mode and audio format are stored in the **first metadata sub-chunk** (ID_WV_BITSTREAM):

**Flags field bit assignments (from WavPack 5 specification):**
```
Bits  0–1:  Bytes stored per sample
              00 = 1 byte/sample  (1–8 bits)
              01 = 2 bytes/sample (9–16 bits)
              10 = 3 bytes/sample (15–24 bits)
              11 = 4 bytes/sample (25–32 bits)
Bit   2:     Mono output — 0=stereo, 1=mono
Bit   3:     Hybrid mode — 0=lossless, 1=hybrid lossy
Bit   4:     Joint stereo — 0=true stereo, 1=joint stereo (mid/side)
Bit   5:     Cross-decorrelation — 0=independent, 1=cross-correlation
Bit   6:     Hybrid noise shaping — 0=flat, 1=shaped
Bit   7:     Floating-point data — 0=integer, 1=IEEE 32-bit float
Bit   8:     Extended integer size — 0=≤24-bit, 1=>24-bit or shifted
Bit   9:     Hybrid bitrate mode — 0=noise controlled, 1=bitrate controlled
Bit  10:     Hybrid balance — 0=standard, 1=balance-optimized
Bit  11:     Initial block — 1=first block in sequence
Bit  12:     Final block — 1=last block in sequence
Bits 13–17:  Reserved
Bits 18–22:  Left-shift amount (0–31 places) for decoded output
Bits 23–27:  Maximum magnitude bits (value + 1 = actual bits needed)
Bits 28–30:  Sample rate index (see rate table)
Bit  31:     DSD audio — 1=DSD data (WavPack 5)
```

### 3.4 Metadata Sub-Chunks
Following the 32-byte block header, metadata sub-chunks follow a simple ID/size/data structure:

```
Offset  Size   Field Name              Type        Valid Range   Description
------  -----  ---------------------  ----------  ------------  ---------------------------
0       1B     Metadata ID             uint8       0x00–0xFF     Sub-chunk type
1       3B     Metadata size          uint24 LE   0–2^24−1      Size of metadata data in bytes
4       N      Metadata data          bytes       varies         Sub-chunk content
--- Next metadata sub-chunk follows immediately ---
```

**Metadata Sub-Chunk IDs:**
| ID (hex) | Name | Description |
|----------|------|-------------|
| 0x00 | ID_DUMMY | Padding / no operation |
| 0x01 | ID_RIFF_HEADER | Embedded RIFF WAV header |
| 0x02 | ID_RIFF_TRAILER | Embedded RIFF WAV trailer (for extra metadata) |
| 0x03 | ID_WV_BITSTREAM | Main audio bitstream (block flags + entropy data) |
| 0x04 | ID_WVC_BITSTREAM | Correction bitstream (for hybrid lossless) |
| 0x05 | ID_WVX_BITSTREAM | Extended bitstream (for very high sample rates) |
| 0x06–0x07 | Reserved | Reserved IDs |
| 0x08 | ID_CHANNEL_INFO | Channel ordering info |
| 0x09 | ID_DSD_DATA | DSD audio data (WavPack 5) |
| 0x0B | ID_MD5_CHECKSUM | MD5 signature of unencoded audio |
| 0x0C | ID_SAMPLE_RATE | Custom sample rate (overrides table index) |
| 0x0D | ID_SEEK_TRAVEL_BITS | Bits for seeking table |
| 0x0E | ID_ENCODE_CONFIG | Encoding configuration |
| 0x0F | ID_HYBRID_BITRATE | Hybrid mode bitrate info |
| 0x21 | ID_APEV2_TAG | APEv2 tag embedded in block |
| 0x22+ | Reserved | Future use |

### 3.5 WV_BITSTREAM Sub-Chunk Layout
The main audio sub-chunk begins with block flags followed by entropy-coded audio data:

```
Offset  Size   Field Name              Type        Description
------  -----  ---------------------  ----------  ---------------------------
0       4B     Block flags             uint32 LE   See flags table above
N       M      Entropy data           variable    Compressed audio samples
```

**Entropy Coding:** WavPack uses a modified **Gold-Rabin-Hash (GRH)** or **bitplane coding** scheme for its entropy coding, different from FLAC's Rice coding. The exact details involve adaptive weighting, lossy quantization, and bitplane representation.

### 3.6 Sample Format Support
|| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 1–8-bit | Signed integer | Yes | 1 byte per sample |
| 9–16-bit | Signed integer | Yes | 2 bytes per sample |
| 17–24-bit | Signed integer | Yes | 3 bytes per sample |
| 25–32-bit | Signed integer | Yes | 4 bytes per sample |
| 32-bit | IEEE float | Yes | Since WavPack 3.0 |
| 64-bit | IEEE float | No | Not supported |
| DSD | 1-bit | Yes | Since WavPack 5.0 (DSD64–DSD512) |

### 3.7 Supported Sample Rates
|| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 6000 | — | Yes | |
| 8000 | Telephone | Yes | |
| 11025 | — | Yes | |
| 16000 | Wideband | Yes | |
| 22050 | — | Yes | |
| 32000 | Broadcast | Yes | |
| 44100 | CD audio | Yes | Most common |
| 48000 | Professional | Yes | Standard |
| 64000 | — | Yes | |
| 88200 | 2× CD | Yes | |
| 96000 | High-res | Yes | |
| 176400 | 4× CD | Yes | |
| 192000 | High-res max | Yes | |
| 352800 | DXD | Yes | |
| 384000 | — | Yes | WV5+ |
| 705600 | DSD64 | Yes | DSD mode (WavPack 5) |
| 1411200 | DSD128 | Yes | DSD mode |
| 2822400 | DSD256 | Yes | DSD mode |
| 5644800 | DSD512 | Yes | DSD mode |

**Sample Rate Table Index (from flags bits 28–30):**
| Index | Rate (Hz) |
|-------|-----------|
| 0 | 6010 |
| 1 | 11025 |
| 2 | 16000 |
| 3 | 22050 |
| 4 | 32000 |
| 5 | 44100 |
| 6 | 48000 |
| 7 | 96000 |

Custom rates can override via `ID_SAMPLE_RATE` metadata sub-chunk.

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 WavPack Encoding Pipeline
```
Input PCM Samples
      │
      ▼
[Channel Decorrelation — Optional]
Mid-side stereo transform on frame-by-frame basis
Determined by encoder to minimize entropy
      │
      ▼
[Prediction Stage — Hybrid Mode Only]
Lossy quantization with noise shaping
Lossy coefficients: adaptive, content-dependent
      │
      ▼
[Entropy Coding — Integer Arithmetic]
WavPack uses a unique entropy coding scheme:
  - Weighted prediction errors (not pure LPC)
  - Bitplane representation for high-resolution
  - Adaptive quantization in hybrid mode
      │
      ▼
[Hybrid Mode: Dual Output]
  ├── Lossy WV stream: Quantized coefficients
  └── Correction WV stream: Difference (lossless correction)
      │
      ▼
[Block Assembly]
Assemble: 32-byte header → metadata sub-chunks → audio sub-chunk
      │
      ▼
Output: WavPack Block
```

### 4.2 WavPack Modes

#### Lossless Mode
- Full lossless compression
- Block-based: each block independently decodable
- Higher compression than most lossless codecs for some content
- Decoding is fast (lowest complexity)

#### Hybrid Mode (Unique to WavPack)
- Generates two files: `.wv` (lossy) and `.wvc` (correction)
- Lossy `.wv` alone: playable, smaller, some artifacts
- Combined `.wv + .wvc`: bit-perfect reconstruction
- Correction file is typically <1% of original size
- Enables lossy preview with future lossless restoration

#### High/Speed Modes
- High mode: Better compression, slower encode
- Fast mode: Less compression, faster encode

### 4.3 Encoder Settings / Quality Modes
|| Mode | Compression | Encode Speed | Decode Speed | Notes |
|------|------------|--------------|---------------|-------|
| Fast | Lower | Fastest | Fast | -f flag |
| Normal | Medium | Normal | Fast | Default |
| High | Higher | Slow | Fast | -h flag |
| Very High | Highest | Slowest | Fast | -hh flag |

**Hybrid Mode Bitrate:**
```
Flag -b n:  Set hybrid bitrate in kbps (n = 24–4800)
Flag -b n.n: Set bits per sample (n.n = 4.0–23.9)
Flag -c:    Create correction file (.wvc)
```

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. Scan for "wvpk" magic at any byte offset
2. Read 32-byte block header
3. Verify block version is valid (0x402–0x50B)
4. Parse metadata sub-chunks until audio sub-chunk found
5. Decode audio from WV_BITSTREAM
6. If WVC_BITSTREAM present (hybrid): combine for lossless
7. Verify CRC against stored CRC
8. Continue to next block
```

#### Seeking
WavPack does not have an embedded seek table by default, but supports fast seeking:

```
1. Seek to estimated position (file_size × target_time / duration)
2. Scan forward/backward for "wvpk" magic
3. Decode block headers until target sample is reached
4. Apply MD5 verification if present
```

### 5.2 Core Decode Pipeline
```
1. Open WV file
2. Find first "wvpk" block
3. Parse block header (version, samples, CRC)
4. Parse metadata sub-chunks:
   ├── ID_WV_BITSTREAM: Extract block flags + entropy data
   ├── ID_WVC_BITSTREAM: Store correction data (hybrid)
   ├── ID_MD5_CHECKSUM: Read MD5 for verification
   ├── ID_CHANNEL_INFO: Read channel ordering
   └── ID_SAMPLE_RATE: Override sample rate
5. Decode entropy-coded audio
6. Apply channel decorrelation inverse (if needed)
7. If hybrid + correction available: combine WV + WVC data
8. Verify CRC-64
9. Output PCM samples
10. Continue to next block
```

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** WavPack WV format (self-contained)
- **Overhead:** ~0.1–1% (block headers, metadata)
- **Seeking in native container:** Yes — block-based, fast
- **Multiple streams in native container:** Yes — up to 8 channels per block

### 6.2 Codec-to-Container Compatibility
|| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| WV (native) | Yes | Yes | Full (APEv2) | Preferred |
| Matroska/MKA | Yes (via EBML) | Yes | Full | Via matroska |
| OGG | No | N/A | N/A | Not supported |
| MP4/M4A | No | N/A | N/A | Not supported |
| WAV | No | N/A | N/A | Not applicable |
| AIFF | No | N/A | N/A | Not applicable |
| WebM | No | N/A | N/A | Not supported |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** APEv2 tags (stored in dedicated sub-chunk or at file end)
- **Tag block location:** Embedded in WV block (ID_APEV2_TAG) or at file end
- **Tag block identifier:** `wvpk` block with APEv2 sub-chunk or standalone APEv2 footer

### 7.2 Standard Tag Fields — Complete Reference
|| Tag Field | Internal Key (WV/APEv2) | Max Length | Character Encoding | Multi-value | Notes |
|-----------|---------------------------|------------|-------------------|-------------|---|
| Title | TITLE | unlimited | UTF-8 | Yes | |
| Artist | ARTIST | unlimited | UTF-8 | Yes | |
| Album | ALBUM | unlimited | UTF-8 | Yes | |
| Album Artist | ALBUMARTIST | unlimited | UTF-8 | Yes | |
| Composer | COMPOSER | unlimited | UTF-8 | Yes | |
| Genre | GENRE | unlimited | UTF-8 | Yes | |
| Year | YEAR | 4 bytes | ASCII | No | Four-digit year |
| Date | DATE | unlimited | UTF-8 | No | Full date string |
| Track Number | TRACK | unlimited | ASCII | No | Usually plain number |
| Track Total | TRACKTOTAL | unlimited | ASCII | No | |
| Disc Number | DISCNUMBER | unlimited | ASCII | No | |
| Disc Total | DISCTOTAL | unlimited | ASCII | No | |
| Comment | COMMENT | unlimited | UTF-8 | Yes | |
| ReplayGain Track Gain | REPLAYGAIN_TRACK_GAIN | unlimited | ASCII | No | Format: "-6.20 dB" |
| ReplayGain Track Peak | REPLAYGAIN_TRACK_PEAK | unlimited | ASCII | No | Format: "0.998459" |
| ReplayGain Album Gain | REPLAYGAIN_ALBUM_GAIN | unlimited | ASCII | No | |
| ReplayGain Album Peak | REPLAYGAIN_ALBUM_PEAK | unlimited | ASCII | No | |
| ISRC | ISRC | 12 bytes | ASCII | No | |
| Cover Art | COVER ART (ARTWORK) | up to 16 MB | Binary | No | JPEG/PNG |
| Encoder | ENCODER | unlimited | UTF-8 | No | Software name |
| Encoder Settings | ENCODER_OPTIONS | unlimited | UTF-8 | No | Settings string |

### 7.3 Cover Art Storage
```
Cover art storage format in WavPack:
  Container type:  APEv2 tag item with binary value
  Image formats:   JPEG (recommended), PNG
  Max image size:  No hard limit (practical: 16 MB)
  
  APEv2 item for cover art:
    Key:  "Cover Art (Front)" or "Cover Art (Back)" or "Cover Art"
    Value: [binary image data] — stored directly, no filename prefix
```

### 7.4 Metadata Compatibility Matrix
|| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| APEv2 | ✓ | ✓ | ✓ | Highest (native) |
| ID3v1 | ✓ | ✓ | Partial | Low |
| ID3v2 | ✓ | ✓ | Partial | Low |
| Vorbis Comments | ✗ | ✗ | N/A | Not supported |
| RIFF INFO | ✓ | ✗ | Partial | Low (from embedded header) |

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   wavpack                    # used with -c:a
AV_CODEC_ID:        AV_CODEC_ID_WAVPACK
Format Name (CLI):   wv                         # used with -f
Encoder(s):          wavpack                    # ffmpeg -encoders | grep wavpack
Decoder(s):          wavpack                    # ffmpeg -decoders | grep wavpack
Muxer(s):          wv                          # ffmpeg -muxers | grep wv
Demuxer(s):         wv                          # ffmpeg -demuxers | grep wv
```

### 8.2 FFmpeg Encoding — Full CLI Reference

```bash
# Encoding to WavPack — complete options reference
ffmpeg -i input.wav \
  -c:a wavpack \
  -compression_level 8 \        # Range: 0–8, default: 0 (fast)
  -b:a 128k \                   # Hybrid mode: bitrate in kbps (range: 24–4800)
  -joint_stereo 1 \              # Joint stereo encoding (0=off, 1=on)
  -ar 44100 \                   # Output sample rate
  -ac 2 \                       # Output channel count
  output.wv

# Lossless encoding (default)
ffmpeg -i input.wav \
  -c:a wavpack \
  -compression_level 6 \
  output.wv

# Hybrid encoding with correction file
ffmpeg -i input.wav \
  -c:a wavpack \
  -b:a 256k \                   # Hybrid bitrate
  output.wv

# Note: FFmpeg does not generate .wvc correction files
# Use standalone WavPack encoder for hybrid + correction

# High-resolution WavPack
ffmpeg -i input_24bit_96k.wav \
  -c:a wavpack \
  -compression_level 8 \
  output.wv
```

#### Complete FFmpeg Option Table
|| Option | Type | Default | Range / Valid Values | Effect on Output |
|--------|------|---------|---------------------|-----------------|
| `-compression_level` | int | 0 | 0–8 | Encode speed vs compression |
| `-b:a` | int | — | 24–4800 kbps | Hybrid mode bitrate |
| `-joint_stereo` | bool | 1 | 0/1 | Mid-side stereo encoding |
| `-ar` | int | from input | any | Output sample rate |
| `-ac` | int | from input | 1–8 | Output channel count |

### 8.3 FFmpeg Encoding — libavcodec C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/opt.h>
#include <libswresample/swresample.h>

// ─── 1. Find encoder ─────────────────────────────────────────────────────────
const AVCodec *codec = avcodec_find_encoder(AV_CODEC_ID_WAVPACK);
if (!codec) { fprintf(stderr, "Encoder not found\n"); exit(1); }

// ─── 2. Allocate codec context ───────────────────────────────────────────────
AVCodecContext *ctx = avcodec_alloc_context3(codec);
if (!ctx) { fprintf(stderr, "OOM\n"); exit(1); }

// ─── 3. Set required parameters ──────────────────────────────────────────────
ctx->sample_fmt   = AV_SAMPLE_FMT_S16;    // WAVPACK supports S16, S32, FLTP
ctx->sample_rate  = 44100;
av_channel_layout_copy(&ctx->ch_layout, &(AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO);

// ─── 4. Set encoding options ─────────────────────────────────────────────────
av_opt_set_int(ctx, "compression_level", 6, AV_OPT_SEARCH_CHILDREN);
av_opt_set_int(ctx, "joint_stereo", 1, AV_OPT_SEARCH_CHILDREN);

// ─── 5. Open codec ───────────────────────────────────────────────────────────
int ret = avcodec_open2(ctx, codec, NULL);
if (ret < 0) { /* handle error */ exit(1); }

// ─── 6. Allocate frame and packet ────────────────────────────────────────────
AVFrame  *frame = av_frame_alloc();
frame->format      = ctx->sample_fmt;
frame->sample_rate = ctx->sample_rate;
av_channel_layout_copy(&frame->ch_layout, &ctx->ch_layout);
frame->nb_samples  = ctx->frame_size;   // WavPack uses variable frame sizes
av_frame_get_buffer(frame, 0);

AVPacket *pkt = av_packet_alloc();

// ─── 7. Encode loop ─────────────────────────────────────────────────────────
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

// ─── 8. Flush encoder ────────────────────────────────────────────────────────
encode_frame(ctx, NULL, pkt, outfile);

// ─── 9. Cleanup ──────────────────────────────────────────────────────────────
av_frame_free(&frame);
av_packet_free(&pkt);
avcodec_free_context(&ctx);
```

**Critical Notes:**
- `ctx->frame_size` for WavPack is **variable** — each output packet represents one WavPack block
- WavPack decoder can handle both lossless and hybrid WV files
- FFmpeg does **not** generate hybrid correction files (`.wvc`) — use standalone WavPack encoder
- Always check `codec->sample_fmts` for supported formats

### 8.4 FFmpeg Decoding — Full CLI Reference

```bash
# Decode to raw PCM WAV
ffmpeg -i input.wv \
  -c:a pcm_s16le \
  output.wav

# Decode hybrid WV + WVC for lossless
ffmpeg -i input.wv \
  -c:a pcm_s16le \
  output.wav

# FFmpeg auto-combines .wv + .wvc if both present
ffmpeg -i input.wv \
  -c:a pcm_s16le \
  output.wav

# Extract specific stream
ffmpeg -i input.wv -map 0:a:0 -c:a copy output.wv

# Probe format
ffprobe -v quiet -print_format json -show_streams -show_format input.wv
```

### 8.5 FFmpeg Decoding — libavcodec C API

```c
// ─── Complete decode program skeleton ─────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.wv", NULL, NULL);
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
ffprobe -v quiet -print_format json -show_format input.wv | jq .format.tags

# Write metadata (APEv2 tags)
ffmpeg -i input.wv \
  -c:a copy \
  -metadata title="Song Title" \
  -metadata artist="Artist Name" \
  -metadata album="Album Name" \
  output.wv

# Strip all metadata
ffmpeg -i input.wv -c:a copy -map_metadata -1 output.wv

# Embed cover art
ffmpeg -i input.wv -i cover.jpg \
  -c:a copy \
  -metadata:s:a title="Album cover" \
  -disposition:s:a attached_pic \
  output.wv
```

#### FFmpeg Internal Metadata Key Mapping
|| Standard Field | FFmpeg Key | WV Native Key | Notes |
|----------------|------------|--------------------------|-------|
| Title | title | TITLE | |
| Artist | artist | ARTIST | |
| Album | album | ALBUM | |
| Album Artist | album_artist | ALBUMARTIST | |
| Track Number | track | TRACK | Plain number |
| Disc Number | disc | DISCNUMBER | |
| Genre | genre | GENRE | |
| Date | date | DATE | |
| Comment | comment | COMMENT | |
| Composer | composer | COMPOSER | |
| Year | date | YEAR | Four-digit year |
| Copyright | copyright | COPYRIGHT | |

### 8.7 Quality / Fidelity Decision Table
|| Use Case | Settings | Expected Size | Notes |
|----------|----------|---------------|-------|
| Archival / best quality | `-c:a wavpack -compression_level 8` | ~0.55–0.65× uncompressed | |
| Standard lossless | `-c:a wavpack -compression_level 6` | ~0.58× uncompressed | Default |
| Hybrid lossy preview | `-c:a wavpack -b:a 256k` | ~256 kbps | With .wvc for lossless |
| Fast encode | `-c:a wavpack -compression_level 0` | ~0.65× uncompressed | Fastest |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
WavPack does not embed a traditional seek table. Seeking is block-based:

```
Seeking strategy:
  1. Estimate position from: byte_offset ≈ (target_sample / total_samples) × file_size
  2. Scan for "wvpk" magic at estimated position
  3. If not found: scan backward/forward until found
  4. Read block header, verify block contains target samples
  5. Decode from that block forward to target sample
```

### 9.2 Gapless Playback Data
```
Encoder delay:  0 samples — WavPack block-based has no lookahead
Padding:        0 samples — blocks are independently decodable
Total samples:  Sum of all block samples fields
MD5 signature:  If present, covers all unencoded samples
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

|| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | Block-based independent decoding |
| Algorithmic encoder delay | 0 samples | No lookahead |
| Live encoding feasible | Yes | With streaming mode |
| HTTP progressive download | Yes | Common for music distribution |
| HTTP Live Streaming (HLS) | No | Not natively supported |
| DASH streaming | Yes | Via Matroska/MKA container |
| WebRTC / RTP transport | No | Not natively supported |
| Minimum decode buffer | 1 block | ~0.5 seconds of audio |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
|| Channels | Layout Name | Channel Order | FFmpeg Layout Constant |
|----------|-------------|---------------|----------------------|
| 1 | Mono | C | AV_CHANNEL_LAYOUT_MONO |
| 2 | Stereo | L, R | AV_CHANNEL_LAYOUT_STEREO |
| 3 | 2.1 | L, R, LFE | AV_CHANNEL_LAYOUT_2POINT1 |
| 4 | Quad | FL, FR, RL, RR | AV_CHANNEL_LAYOUT_QUAD |
| 5 | 5.0 | FL, FR, C, SL, SR | AV_CHANNEL_LAYOUT_5POINT0 |
| 6 | 5.1 | FL, FR, C, LFE, SL, SR | AV_CHANNEL_LAYOUT_5POINT1 |
| 7 | 6.1 | FL, FR, C, LFE, BL, BC, BR | |
| 8 | 7.1 | FL, FR, C, LFE, BL, BR, SL, SR | AV_CHANNEL_LAYOUT_7POINT1 |

### 11.2 Downmix Coefficients (5.1 → Stereo)
```
L_out = L + C × 0.7071 + LS × 0.7071
R_out = R + C × 0.7071 + RS × 0.7071
LFE:  discarded

FFmpeg downmix:
ffmpeg -i input_5_1.wv \
  -af "pan=stereo|FL=FC+0.707*FC+0.707*BL|FR=FC+0.707*FC+0.707*BR" \
  output_stereo.wv
```

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

|| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 32-bit integer / 32-bit float | |
| Max sample rate | 2 MHz [NEEDS VERIFICATION] | |
| Float support | Yes | 32-bit IEEE float (since WavPack 3.0) |
| DSD support | Yes | DSD64–DSD512 (WavPack 5.0+) |
| 20-bit support | Yes | High-res standard |
| 24-bit support | Yes | Most common hi-res format |

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

|| Platform / API | Encode | Decode | Notes |
|----------------|--------|--------|-------|
| Rockbox firmware | Yes | Yes | Native on supported DAPs |
| FFmpeg native | Yes | Yes | Built-in |
| foobar2000 | Yes | Yes | Native |
| VLC | Yes | Yes | Via FFmpeg |
| JRiver Media Center | Yes | Yes | Native |

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Encoder-Specific Issues
|| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| Hybrid mode correction files not generated | All | Use standalone WavPack encoder |
| APEv2 tag editing not supported | < 5.0 | Use dedicated tag editors |
| DSD mode limited support | < 6.0 | Decode DSD as PCM |

### 14.2 Interoperability Issues
- **Hybrid .wvc files:** Some players don't understand correction files; play lossy version
- **WavPack 5 files:** Older decoders may not support new block format
- **APEv2 tags at file end:** Some tools don't find them; embedded tags are more portable
- **MD5 verification:** Only works if source was encoded with MD5 enabled

### 14.3 Edge Cases to Handle in Converter
- **Empty file / 0-sample stream:** Valid WV with no audio blocks; skip gracefully
- **File < 1 block:** Valid; decode what exists
- **All-silence audio:** Highly compressible; single sample per block
- **DC offset:** WavPack handles DC efficiently
- **Full-scale sine (0 dB):** No clipping in lossless mode
- **File with corrupt block:** CRC-64 detects; skip block, continue
- **Truncated file:** Stop at last valid block
- **Hybrid file without correction:** Decode lossy version, warn user

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM WavPack

|| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → FLAC | `ffmpeg -i in.wv -c:a flac -compression_level 8 out.flac` | APEv2 → Vorbis | Lossless if source was lossless |
| → ALAC | `ffmpeg -i in.wv -c:a alac out.m4a` | Via -metadata | Lossless if source was lossless |
| → WAV | `ffmpeg -i in.wv -c:a pcm_s16le out.wav` | Limited | Lossless if source was lossless |
| → MP3 | `ffmpeg -i in.wv -c:a libmp3lame -q:a 0 out.mp3` | APEv2 → ID3v2.3 | Generation loss |
| → AAC | `ffmpeg -i in.wv -c:a aac -b:a 256k out.m4a` | Via -metadata | Generation loss |

### 15.2 Converting TO WavPack

|| Source | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| FLAC → | `ffmpeg -i in.flac -c:a wavpack -compression_level 8 out.wv` | Vorbis → APEv2 | Lossless |
| WAV → | `ffmpeg -i in.wav -c:a wavpack -compression_level 8 out.wv` | RIFF INFO → APEv2 | Lossless |
| MP3 → | `ffmpeg -i in.mp3 -c:a wavpack -compression_level 6 out.wv` | ID3v2 → APEv2 | Lossless (re-encode) |
| ALAC → | `ffmpeg -i in.m4a -c:a wavpack -compression_level 8 out.wv` | Via -metadata | Lossless (re-encode) |

### 15.3 Lossless Round-Trip Verification
```bash
# Encode
ffmpeg -i original.wav -c:a wavpack -compression_level 8 output.wv

# Decode back
ffmpeg -i output.wv -c:a pcm_s16le decoded.wav

# Compare checksums
ffmpeg -i original.wav -c:a pcm_s16le original_raw.wav
md5sum original_raw.wav decoded_raw.wav   # Must match for true lossless

# Verify MD5 if present in WV file
ffprobe -v quiet -show_entries stream=tags output.wv | grep MD5
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

|| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| WavPack lib | C | BSD | Reference | Reference | https://www.wavpack.com/ |
| FFmpeg native | C | LGPL 2.1+ | 8/10 | 9/10 | https://ffmpeg.org |
| Rockbox | C | GPL | Native | Native | https://www.rockbox.org |

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **WavPack 5 File Format Specification:** https://www.wavpack.com/WavPack5FileFormat.pdf
- **WavPack 5 Library Documentation:** https://www.wavpack.com/WavPack5LibraryDoc.pdf

### Technical Resources
- FFmpeg encoder options: `ffmpeg -h encoder=wavpack` or https://ffmpeg.org/ffmpeg-codecs.html
- FFmpeg demuxer options: `ffmpeg -h demuxer=wv` or https://ffmpeg.org/ffmpeg-formats.html
- Multimedia Wiki: https://wiki.multimedia.cx/index.php/WavPack
- Hydrogenaudio: https://hydrogenaud.io/index.php/board,55.0.html

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] FFmpeg wavpack encoder/decoder is built into default FFmpeg — no external dependency
- [ ] Verify `ffmpeg -encoders` output confirms wavpack encoder is available
- [ ] Verify `ffmpeg -decoders` output confirms wavpack decoder is available
- [ ] For hybrid correction files: recommend standalone WavPack encoder (not in FFmpeg)

### Encoding Pipeline
- [ ] Convert input sample format to required `ctx->sample_fmt` using libswresample
- [ ] WavPack uses variable frame sizes — no fixed `nb_samples` requirement
- [ ] Implement proper flush/drain at end of stream (send NULL frame)
- [ ] Validate that input sample rate is within supported range
- [ ] Validate that channel layout is supported (1–8 channels)
- [ ] Write APEv2 tags via -metadata flags

### Decoding Pipeline
- [ ] Implement WavPack "wvpk" magic scan for stream recovery
- [ ] Parse 32-byte block header
- [ ] Parse metadata sub-chunks (ID + 3-byte size + data)
- [ ] Handle AVERROR(EAGAIN) in receive_packet loop correctly
- [ ] Flush decoder at EOF (send NULL packet)
- [ ] Handle sample format conversion from decoder output format
- [ ] Verify CRC-64 for block integrity

### Metadata
- [ ] Read APEv2 tags from WV file (embedded or at end)
- [ ] Read ID3v1 tags if present (legacy)
- [ ] Read embedded RIFF headers if present
- [ ] Write all standard tag fields as APEv2 tags
- [ ] Embed cover art as APEv2 binary item
- [ ] Preserve ReplayGain tags through conversion
- [ ] Handle UTF-8 encoding in all tag fields

### Quality & Verification
- [ ] Implement ReplayGain scan via EBU R128 (libebur128 integration)
- [ ] Write ReplayGain tags in APEv2 format
- [ ] Provide bit-exact verification for lossless conversions
- [ ] Verify MD5 checksum if present in WV file
- [ ] Test with: silence, full-scale, clipped, multi-channel, high-resolution files

### Edge Cases
- [ ] Handle files with corrupt or missing block headers
- [ ] Handle hybrid files without .wvc correction file
- [ ] Handle files with only APEv2 tags and no audio
- [ ] Handle very short files (< 1 block)
- [ ] Handle DSD audio files (WavPack 5)

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
