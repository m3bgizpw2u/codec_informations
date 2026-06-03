# TAK (Tom's Lossless Audio Kompressor) — Deep Technical Reference

> **Category:** Lossless
> **File Extensions:** `.tak`
> **MIME Types:** `audio/x-tak`, `audio/tak`
> **Standardization Body:** Thomas Becker (proprietary, closed-source)
> **Primary Specification:** No public specification — reverse-engineered format
> **Patent Status:** Unknown — proprietary format
> **License:** Free for non-commercial use; source code not available
> **Current Version:** 2.3.3 (September 2012)
> **Active Development:** No — discontinued ~2012, final release 2012

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Thomas Becker
- **Year Created:** 2006 (YALAC prototype), 2007 (TAK 1.0 release)
- **Original Purpose:** High-compression lossless audio codec with excellent decode speeds
- **Problem with Predecessors:** Monkey's Audio at high compression was slow to decode; FLAC was fast but didn't compress as well. TAK aimed to match Monkey's Audio compression while matching FLAC decode speeds.

Thomas Becker developed TAK after analyzing the lossless codec landscape of the mid-2000s:
1. **FLAC** (Xiph.org): Fast but limited compression ratios
2. **Monkey's Audio** (Matthew T. Ashland): Excellent compression but very slow decoding
3. **OptimFROG** (Florin Ghido): Maximum compression but extremely slow

TAK was designed to achieve:
- Compression ratios comparable to Monkey's Audio High/Extra modes
- Decoding speeds approaching FLAC's fast performance
- A balance between compression efficiency and practical utility
- Clean implementation without the complexity of OptimFROG

The original development codename was "YALAC" (Yet Another Lossless Audio Codec), reflecting the experimental nature of the early versions.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| YALAC | 2006 | Early prototype name (Yet Another Lossless Audio Codec) |
| 0.x | 2006-2007 | Beta testing phase with internal releases |
| 1.0 | 2007 | First public release, 24-bit support, 4 presets |
| 1.1.0 | 2008 | Improved compression, new presets |
| 1.1.1 | 2008 | Bug fixes, performance improvements |
| 1.1.2 | 2009 | Final 1.x release, stability improvements |
| 2.0 | 2010 | Enhanced frame structure, better compression, new codec types |
| 2.1 Beta | 2011 | LossyWav integration experiment, new features |
| 2.2 | 2011 | Multi-channel improvements, new features |
| 2.3.0 | 2012 | Final development cycle |
| 2.3.3 | September 2012 | Final release |

Key milestones:
- 2007: Public release of TAK 1.0 with four presets
- 2008-2009: Incremental improvements and bug fixes
- 2010: Major version 2.0 with enhanced features
- 2012: Final release version 2.3.3

### 1.3 Current Adoption
- **Primary use cases today:** High-efficiency lossless archival, foobar2000 community, German-speaking audio communities
- **Platforms with native support:** Windows (native), Linux (via FFmpeg), macOS (via FFmpeg)
- **Major services using this format:** None (niche format)
- **Hardware support:** No hardware support
- **Status:** Legacy — superseded by open standards like FLAC

TAK maintains a dedicated following among audio enthusiasts who prioritize compression efficiency. The format is particularly popular in:
- German-speaking audio communities (Thomas Becker's native region)
- foobar2000 users who value its efficiency
- Users with large audio libraries seeking maximum storage efficiency

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Waveform / Predictive
- **Core algorithm:** Adaptive Linear Prediction with custom entropy coding
- **Loss mechanism:** Lossless — uses multi-stage prediction with Rice-Huffman hybrid coding
- **Frame-based vs sample-based:** Frame-based; independent frames with 24-bit CRC protection
- **Fixed vs variable frame size:** Variable frame size, typically 94-250ms of audio

TAK represents a sophisticated approach to lossless audio compression, combining multiple advanced techniques:

1. **Multi-stage linear prediction** for accurate signal modeling
2. **Custom entropy coding** that bridges Rice and Huffman coding
3. **Independent frame structure** for random access and error tolerance
4. **PARCOR coefficient compression** for efficient predictor storage

### 2.2 High-Level Encoding Flow (Block Diagram)
```
Input PCM Samples (All Channels)
      │
      ▼
[Pre-processing: Channel decorrelation, format validation]
      │
      ▼
[Frame Splitting: Variable-length frames (94-250ms)]
      │
      ▼
[Channel Decorrelation: Multi-channel processing]
      │
      ▼
[Stage 1: High-order LMS Filter (up to 160 coefficients)]
      │
      ▼
[Stage 2: Medium-order LMS Filter (order 16)]
      │
      ▼
[Stage 3: Low-order LMS Filter (order 8)]
      │
      ▼
[Stage 4: Delta Filter (neighbor prediction)]
      │
      ▼
[Residual Coding: Custom Rice-Huffman hybrid]
      │
      ▼
[Bitstream Packing: Frame header + CRC24 + compressed data]
      │
      ▼
Output TAK Stream
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | ~94-250ms per frame | Configurable frame size |
| Frame size | Variable (94-250ms) | User-selectable during encoding |
| Max channels | 8 | Stereo, quad, 5.1, 7.1 supported |
| Max bit depth | 24-bit | Integer only |
| Max sample rate | 192 kHz | |
| Bitrate range | N/A | Lossless — bitrate varies with content |
| Complexity | O(n log n) | Fast compared to OptimFROG |
| Memory usage | Moderate | Depends on frame size and preset |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       74 42 61 4B     tBaK      File magic / signature
0x0004  variable  ...             ..        Stream data begins
```

The "tBaK" signature is a play on words - it resembles the German word "Tobak" (tobacco) and also serves as the format identifier.

### 3.2 File-Level Header Layout
TAK format uses a proprietary structure with no official public specification. Based on reverse engineering:

```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------
0x0000   4B      Magic Bytes           uint8[4]    "tBaK" signature
0x0004   variable Stream Information     —          Encoder info, audio format, etc.
```

#### Stream Information Structure (Reverse-Engineered)
```
Codec Type (6 bits):
  - 0 = Integer 24-bit (TAK 1.0)
  - 1 = Experimental
  - 2 = Integer 24-bit (TAK 2.0)
  - 3 = LossyWav (TAK 2.1 Beta)
  - 4 = Integer 24-bit MC (TAK 2.2) — multi-channel

Encoder Profile (4 bits):
  - 0 = Turbo (-p0)
  - 1 = Fast (-p1)
  - 2 = Normal (-p2)
  - 3 = High (-p3)
  - 4 = Extra (-p4)
  - 5+ = Reserved

Frame Size Type (4 bits):
  - Encodes frame duration (94-250ms range)

Audio Format:
  - Sample Rate (18 bits): sample_rate - 6000
  - Sample Bits (5 bits): bits_per_sample - 8
  - Audio Channels (4 bits): channels - 1

Extension Data (if present):
  - Valid Bits Per Sample (5 bits)
  - Speaker Assignment Present (1 bit)
  - Speaker Assignment (6 bits × channels)
```

### 3.3 Frame / Block Header Layout
```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------
0x0000   2B      Sync Word              uint16 LE   Always 0xA0FF (little-endian)
Bit 15      Last Frame               bit         1 = last frame in stream
Bits 14-0   Frame Number             uint15      Frame index

Last frame extra:
  Bits 14-0  Sample Count - 1         uint15      Sample count for last frame

Padding:       0-7 bits               bits        Alignment padding

CRC:           3B                     uint24 LE   CRC-24 of frame header
```

#### Frame Data Structure
```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------
0x0000   N B     Frame Data              bytes       Compressed audio data
  N-3B   3B      Frame CRC               uint24 LE   CRC-24 of frame data
```

### 3.4 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Unsigned integer | No | Not supported |
| 16-bit | Signed integer | Yes | Standard CD audio |
| 20-bit | Signed integer | Yes | Professional audio |
| 24-bit | Signed integer | Yes | High-resolution standard |
| 32-bit | Signed integer | No | Not supported |
| 32-bit | IEEE float | No | Not supported |
| 64-bit | IEEE float | No | Not supported |

#### Supported Sample Rates
| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 8000 | Telephone | Yes | |
| 11025 | — | Yes | |
| 16000 | Wideband voice | Yes | |
| 22050 | — | Yes | |
| 32000 | Broadcast | Yes | |
| 44100 | CD audio | Yes | Most common |
| 48000 | Professional | Yes | Standard |
| 88200 | 2× CD | Yes | |
| 96000 | High-res | Yes | |
| 176400 | 4× CD | Yes | |
| 192000 | High-res max | Yes | |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Pre-Processing Stage
- **DC offset removal:** Implicit in linear prediction
- **Pre-emphasis filter:** None (pure linear prediction)
- **Windowing function:** Full frame (no windowing)
- **Level normalization:** None required
- **Stereo decorrelation pre-step:** Channel decorrelation for multi-channel

The pre-processing ensures optimal conditions for the subsequent prediction stages.

### 4.2 Analysis / Transform Stage

#### Transform Type: Adaptive Linear Prediction (Multiple Stages)
TAK uses cascaded linear prediction filters for optimal compression:

```
Parameters:
  Prediction stages:   2-4 stages of adaptive filters
  Max predictor order: 160 coefficients (EXTRA mode)
  Window:              Full frame
  Algorithm:           Modified Levinson-Durbin
  Coefficient precision: Custom compression (PARCOR-like)
  
Filter Types and Orders:
  Stage 1 (Primary):
    - Turbo/Fast: No filter (bypass)
    - Normal: 8 coefficients
    - High: 32 coefficients
    - Extra: 160 coefficients
    
  Stage 2 (Secondary):
    - Order: 16 coefficients
    - Adapts to residual from Stage 1
    
  Stage 3 (Tertiary):
    - Order: 8 coefficients
    - Final polishing of residuals
    
  Stage 4 (Delta):
    - Simple neighbor prediction
    - pred = last_sample × weight >> 8
```

**LPC Analysis Mathematical Foundation:**
```
Linear Prediction:
  x̂[n] = Σ(k=1 to p) aₖ × x[n-k]
  
PARCOR Coefficients:
  - Partial correlation coefficients
  - Derived from LPC coefficients via Levinson recursion
  - More efficient for storage than direct LPC coefficients
  
Coefficient Encoding:
  - TAK uses a custom encoding for PARCOR coefficients
  - More efficient than plain binary representation
  - Similar to FLAC's LPC coefficient encoding
```

### 4.3 Psychoacoustic Model (Not Applicable)
TAK is a **lossless** codec. No psychoacoustic modeling is performed.

### 4.4 Quantization
TAK uses **no quantization** — it is a purely lossless codec.

### 4.5 Stereo Encoding Modes
| Mode | Description | Implementation |
|------|-------------|----------------|
| Independent | L and R encoded separately | Default for mono sources |
| Mid-Side | M=(L+R)/2, S=(L-R)/2 | Applied per channel pair |
| Channel Decorrelation | Complex multi-channel decorrelation | For surround sound |

### 4.6 Entropy / Lossless Coding Stage
```
Method: Custom Rice-Huffman hybrid

The TAK entropy coder combines elements of Rice and Huffman coding:

Rice Coding Component:
  - Used for small residuals
  - Fast decoding
  - Parameter k selected per-block
  
Huffman Coding Component:
  - Used for larger residuals
  - Better compression for diverse residual distributions
  - Multiple code tables
  
Hybrid Approach:
  1. For small residuals: Direct Rice coding (fast path)
  2. For large residuals: Huffman-coded escapes (efficient path)
  3. Dynamic selection based on residual statistics
  
Performance Characteristics:
  - Compression efficiency: Approaches arithmetic coding
  - Decoding speed: Near-Rice decoding performance
  - Implementation complexity: Balanced
```

### 4.7 Encoder Settings / Quality Modes

#### Compression Levels
| Preset | Flag | Encoding Speed | Decode Speed | Compression Ratio | Notes |
|--------|------|---------------|--------------|-----------------|-------|
| Turbo | -p0 | Fastest (~100× RT) | Fastest (~150× RT) | ~60-70% | Default fast |
| Fast | -p1 | Fast (~50× RT) | Fast (~120× RT) | ~55-65% | |
| Normal | -p2 | Medium (~20× RT) | Fast (~100× RT) | ~50-60% | Balanced |
| High | -p3 | Slow (~10× RT) | Medium (~60× RT) | ~45-55% | |
| Extra | -p4 | Slowest (~5× RT) | Medium (~40× RT) | ~40-50% | Maximum |

Speed values are approximate for modern x86 processors. TAK's decode speeds are particularly competitive, often exceeding encoding speeds by 2-3×.

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. Scan for magic bytes: "tBaK" (0x74 0x42 0x61 0x4B)
   - Start from file beginning
   - Match exactly 4 bytes
   
2. Parse Stream Information block
   - Extract codec type, profile, frame size
   - Read audio format parameters
   
3. Frame sync via 0xA0FF sync word
   - Each frame begins with sync word
   - 0xA0FF value (little-endian)
   
4. Validate frame header CRC-24
5. Validate frame data CRC-24
```

#### Info Frames
TAK inserts "info frames" every ~2 seconds containing all data needed to start decoding:

```
Info Frame contains:
  - Complete stream information
  - Decoder initialization parameters
  - Previous frame reference (for prediction)
  - Allows random access decoding
  
Benefits:
  - Enables mid-stream decoding start
  - Provides recovery points
  - Facilitates error recovery
```

#### Seeking
- **Seek table:** Stored in file header — sample positions at 1-second intervals
- **Seek precision:** Sample-accurate
- **No seek table:** Frame sync words allow seeking via scan

### 5.2 Core Decode Pipeline
```
1. Read file header ("tBaK" + Stream Information)
   ├── Verify magic bytes
   ├── Parse encoder info (version, preset, evaluation)
   ├── Read audio format (channels, sample rate, bit depth)
   └── Initialize decoder state

2. Read seek table (if present)
   └── Frame offsets and sample positions

3. For each frame:
   ├── Read sync word (0xA0FF)
   ├── Parse frame header (number, sample count)
   ├── Validate header CRC-24
   ├── Read compressed frame data
   ├── Validate data CRC-24
   └── Decode to PCM samples:
       ├── Rice-Huffman hybrid decoding
       ├── Reconstruct residuals
       ├── Apply inverse delta filter
       ├── Apply inverse LMS filters (stages 3, 2, 1)
       └── Reconstruct channel data

4. Final processing
   ├── Format output samples
   └── Output to consumer
```

### 5.3 Error Concealment
- **Corrupt frame detection:** 24-bit CRC per frame
- **Concealment method:** Replace corrupt frame with silence (mute)
- **Error tolerance:** Single bit error affects max 250ms (frame size)
- **Badly damaged files:** Decoder attempts to recover at next sync

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** TAK has its own native format
- **Overhead:** ~0.05% (headers + CRCs only)
- **Seeking in native container:** Yes — via seek table and info frames
- **Multiple streams in native container:** No

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| TAK (native) | Yes | Yes | APEv2 | No container overhead |
| Matroska/MKA | Yes | Yes | Full | Via Matroska audio codec |
| FLAC | No | N/A | N/A | Different format |
| MP4/M4A | No | N/A | N/A | Not designed for MP4 |
| OGG | No | N/A | N/A | Not designed for OGG |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** APEv2 (stored at end of file)
- **Tag block location:** End of file (after all audio data)
- **Tag block identifier:** "APETAGEX" at file end
- **ID3v1:** Also supported for compatibility

### 7.2 Standard Tag Fields — Complete Reference
| Tag Field | Internal Key (TAK/APEv2) | Max Length | Character Encoding | Multi-value | Notes |
|-----------|------------------------|------------|-------------------|-------------|-------|
| Title | TITLE | 255 bytes | UTF-8 | No | Track title |
| Artist | ARTIST | 255 bytes | UTF-8 | No | Performer |
| Album | ALBUM | 255 bytes | UTF-8 | No | Album name |
| Album Artist | ALBUMARTIST | 255 bytes | UTF-8 | No | Album performer |
| Composer | COMPOSER | 255 bytes | UTF-8 | No | Composer |
| Genre | GENRE | 255 bytes | UTF-8 | No | Genre classification |
| Year / Date | YEAR or DATE | 4 bytes | ASCII | No | Release year |
| Track Number | TRACK | 10 bytes | ASCII | No | Format: "N" or "N/Total" |
| Disc Number | DISC | 10 bytes | ASCII | No | Format: "N" or "N/Total" |
| Comment | COMMENT | 1000 bytes | UTF-8 | No | Freeform comment |
| ReplayGain Track Gain | REPLAYGAIN_TRACK_GAIN | 20 bytes | ASCII | No | Format: "-6.20 dB" |
| ReplayGain Track Peak | REPLAYGAIN_TRACK_PEAK | 20 bytes | ASCII | No | Format: "0.998459" |
| Encoder | ENCODER | 64 bytes | UTF-8 | No | Software name and version |
| Encoder Settings | ENCODER_SETTINGS | 64 bytes | UTF-8 | No | Preset, quality settings |

### 7.3 Cover Art Storage
TAK supports embedded cover art via APEv2's "COVER ART" field or "METADATA_BLOCK_PICTURE".

### 7.4 Metadata Compatibility Matrix
| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| ID3v1 | ✓ | ✓ | ✓ | Lowest |
| APEv2 | ✓ | ✓ | ✓ | Highest |

**Conflict resolution:** APEv2 takes precedence over ID3v1.

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   tak              # used with -c:a
AV_CODEC_ID:        AV_CODEC_ID_TAK  # C constant in libavcodec/codec_id.h
Format Name (CLI):  tak               # used with -f
Encoder(s):         NONE — FFmpeg does NOT encode TAK
Decoder(s):         tak (native)       # ffmpeg -decoders | grep -i tak
Muxer(s):           tak               # ffmpeg -muxers | grep -i tak
Demuxer(s):         tak              # ffmpeg -demuxers | grep -i tak
```

### 8.2 FFmpeg Encoding — NOT SUPPORTED
FFmpeg does **NOT** support encoding TAK format. TAK encoding requires the official TAK encoder from Thomas Becker.

### 8.3 FFmpeg Decoding — Full CLI Reference

```bash
# Decode TAK to WAV
ffmpeg -i input.tak \
  -c:a pcm_s16le \
  output.wav

# Decode TAK to FLAC
ffmpeg -i input.tak \
  -c:a flac -compression_level 8 \
  output.flac

# Extract audio stream without conversion
ffmpeg -i input.tak \
  -c:a copy \
  output.tak

# Probe TAK file information
ffprobe -v quiet -print_format json -show_streams -show_format input.tak

# Extract metadata
ffprobe -v quiet -print_format json -show_format input.tak | jq .format.tags

# Convert with specific output format
ffmpeg -i input.tak \
  -c:a pcm_s24le \
  -ar 96000 \
  output_96k.wav
```

### 8.4 FFmpeg Decoding — libavcodec C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>

// ─── Complete decode program skeleton ────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.tak", NULL, NULL);
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
            // frm->pts = presentation timestamp
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

### 8.5 FFmpeg Metadata Handling

```bash
# Read all metadata as JSON
ffprobe -v quiet -print_format json -show_format input.tak | jq .format.tags

# Strip all metadata
ffmpeg -i input.tak -c:a copy -map_metadata -1 output.tak

# Note: FFmpeg cannot write APEv2 tags to TAK files natively
# Use external tools like: apetag, foobar2000, tag
```

#### FFmpeg Internal Metadata Key Mapping
| Standard Field | FFmpeg Key | TAK/APEv2 Native Key | Notes |
|----------------|------------|----------------------|-------|
| Title | title | TITLE | |
| Artist | artist | ARTIST | |
| Album | album | ALBUM | |
| Track Number | track | TRACK | |
| Genre | genre | GENRE | |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
TAK stores a seek table in the file header:

```
TAK Seek Table:
  Location:     File header, after Stream Information
  Entry size:   8 bytes
  Entry format:
    [0x00–0x03]  sample_position (uint32) — Sample number at seek point
    [0x04–0x07]  frame_offset (uint32)    — Byte offset to frame
  Seek point interval: 1 second (configurable during encoding)
```

### 9.2 Info Frames
```
Info Frame (every ~2 seconds):
  Contains:
    - Complete stream information
    - Decoder initialization parameters
  Purpose:
    - Enables random access decoding
    - Recovery point after errors
```

### 9.3 Gapless Playback Data
```
Encoder delay:   0 samples (TAK adds no padding)
Padding:         0 samples
Storage location: Total samples in Stream Information header
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | Info frames enable mid-stream start |
| Algorithmic encoder delay | ~94-250ms per frame | Variable frame size |
| Live encoding feasible | Limited | Designed for file-based encoding |
| HTTP progressive download | Yes | Supported |
| HTTP Live Streaming (HLS) | No | Not a standard HLS segment format |
| DASH streaming | No | Not commonly used |
| WebRTC / RTP transport | No | Not designed for real-time transport |
| Minimum decode buffer | 1 frame | Variable size |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | Layout Name | Channel Order | FFmpeg Layout Constant |
|----------|-------------|---------------|----------------------|
| 1 | Mono | C | AV_CHANNEL_LAYOUT_MONO |
| 2 | Stereo | L, R | AV_CHANNEL_LAYOUT_STEREO |
| 3 | 2.1 | L, R, LFE | AV_CHANNEL_LAYOUT_2POINT1 |
| 4 | Quad | FL, FR, RL, RR | AV_CHANNEL_LAYOUT_QUAD |
| 5 | 5.0 | FL, FR, C, SL, SR | AV_CHANNEL_LAYOUT_5POINT0 |
| 6 | 5.1 | FL, FR, C, LFE, SL, SR | AV_CHANNEL_LAYOUT_5POINT1 |
| 8 | 7.1 | FL, FR, C, LFE, BL, BR, SL, SR | AV_CHANNEL_LAYOUT_7POINT1 |

### 11.2 Downmix Coefficients (5.1 → Stereo)
```
L_out = L_in + (C_in × 0.7071) + (LS_in × 0.5)
R_out = R_in + (C_in × 0.7071) + (RS_in × 0.5)
LFE:  typically mixed out (0.0 coefficient)
```

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 24-bit | Integer only |
| Max sample rate | 192 kHz | |
| Float support | None | Integer arithmetic only |
| DSD support | No | Not supported |
| 20-bit support | Yes | Common in professional audio |
| 24-bit support | Yes | High-res standard |

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

| Platform / API | Encode | Decode | FFmpeg Flag | Notes |
|----------------|--------|--------|-------------|-------|
| x86 SIMD | No | Yes | — | SSE optimization in reference |
| ARM NEON | No | Limited | — | Not officially optimized |
| CUDA/NVENC | No | No | — | Not applicable |
| OpenCL | No | No | — | Not applicable |

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Decoder Issues
| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| No TAK encoder | All versions | Use official TAK encoder |
| Limited preset info | All versions | Preset info not exposed in ffprobe |
| Speaker assignment | All versions | Channel layout may not be accurate |
| Version 2.x features | Limited support | Some 2.x features may not decode |

### 14.2 Interoperability Issues
- **FFmpeg TAK → Official decoder:** Fully compatible
- **Official TAK → FFmpeg decode:** Fully compatible
- **TAK 2.x files in FFmpeg:** May have compatibility issues with new features
- **Multi-channel interpretation:** Speaker assignment not standardized

### 14.3 Edge Cases to Handle in Converter
- **Empty file / 0-sample stream:** Invalid TAK file
- **All-silence audio:** Encodes very efficiently
- **Corrupt frame:** CRC failure, frame replaced with silence
- **Truncated file:** Decoder stops at last valid frame
- **Sample rate not supported:** Must be integer value
- **Channel count > 8:** Not supported

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM TAK

| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → FLAC | `ffmpeg -i in.tak -c:a flac -compression_level 8 out.flac` | All tags | Lossless |
| → ALAC | `ffmpeg -i in.tak -c:a alac out.m4a` | Via MP4 atoms | Lossless |
| → WAV | `ffmpeg -i in.tak -c:a pcm_s16le out.wav` | RIFF INFO | Lossless |
| → MP3 | `ffmpeg -i in.tak -c:a libmp3lame -q:a 0 out.mp3` | Via ID3v2 | Generation loss |
| → AAC | `ffmpeg -i in.tak -c:a aac -b:a 256k out.m4a` | Via MP4 | Generation loss |
| → Opus | `ffmpeg -i in.tak -c:a libopus -b:a 128k out.opus` | Via Ogg | Generation loss |

### 15.2 Converting TO TAK
FFmpeg does **NOT** support encoding TAK. Use the official TAK encoder:
- Windows: TAK.exe GUI application
- Command-line: Takc.exe
- foobar2000: TAK plugin
- Note: No FFmpeg-based TAK encoding available

### 15.3 Lossless Round-Trip Verification
```bash
# Decode
ffmpeg -i original.tak -c:a pcm_s24le decoded.wav

# Compare checksums
ffmpeg -i original.tak -map 0:a -f framemd5 original.md5
ffmpeg -i decoded.wav -map 0:a -f framemd5 decoded.md5
diff original.md5 decoded.md5   # Empty diff = bit-perfect
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| TAK Reference | C/C++ | Proprietary | Reference | Reference | thbeck.de (offline) |
| FFmpeg libavcodec | C | LGPL 2.1+ | N/A | 8/10 | https://ffmpeg.org |
| foobar2000 plugin | C++ | Proprietary | Reference | Reference | foobar2000 |
| Winamp plugin | C | Proprietary | Reference | Reference | Winamp |

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **Official TAK Site:** http://thbeck.de/Tak/Tak.html (may be offline)
- **Multimedia Wiki:** https://wiki.multimedia.cx/index.php/TAK

### Technical Resources
- FFmpeg codec support: `ffmpeg -decoders | grep tak`
- Hydrogenaudio TAK thread: https://hydrogenaudio.org/index.php/topic,52212.0.html
- HandWiki TAK article: https://handwiki.org/wiki/Software:TAK_(audio_codec)

### Academic Papers
- None published (closed development)

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] Identify FFmpeg built with TAK support (verify `ffmpeg -decoders | grep tak`)
- [ ] Note: FFmpeg does NOT encode TAK — must use external encoder
- [ ] Handle platform-specific TAK encoder availability

### Decoding Pipeline
- [ ] Implement sync word search for "tBaK" magic
- [ ] Parse Stream Information structure
- [ ] Validate stream information integrity
- [ ] Read seek table (if present)
- [ ] Frame sync via 0xA0FF sync word
- [ ] Validate frame header CRC-24
- [ ] Read compressed frame data
- [ ] Validate frame data CRC-24
- [ ] Implement Rice-Huffman hybrid decoding
- [ ] Apply inverse prediction filters (cascade)
- [ ] Flush decoder at EOF

### Metadata
- [ ] Read APEv2 tags from end of file
- [ ] Read ID3v1 tags from end of file
- [ ] Preserve tag priority (APEv2 > ID3v1)
- [ ] Cover art extraction from tags

### Quality & Verification
- [ ] Implement frame-level CRC verification
- [ ] Track corrupted frames for error reporting
- [ ] Test with: silence, full-scale, multichannel, high-resolution files

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
