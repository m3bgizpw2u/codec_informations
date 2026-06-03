# OptimFROG — Deep Technical Reference

> **Category:** Lossless
> **File Extensions:** `.ofr`, `.ofs`
> **MIME Types:** `audio/x-optimfrog`, `audio/ofr`
> **Standardization Body:** Florin Ghido (proprietary, closed-source)
> **Primary Specification:** No public specification — reverse-engineered format
> **Patent Status:** Unknown — proprietary format
> **License:** Proprietary freeware (closed-source)
> **Current Version:** 5.100 (September 2016)
> **Active Development:** No — discontinued ~2016, stable

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Florin Ghido, Romania
- **Year Created:** 2000 (OptimFROG 1.0), 2001 (OptimFROG 4.0 with stereo decorrelation)
- **Original Purpose:** Maximum-compression lossless audio for archival
- **Problem with Predecessors:** FLAC and other codecs of the era prioritized speed over compression. OptimFROG was designed to achieve the absolute best compression ratios, even at the cost of encoding speed.

Florin Ghido developed OptimFROG as a research project to explore the limits of lossless audio compression. While other codecs like FLAC balanced compression efficiency with encoding/decoding speed, OptimFROG was designed purely for maximum compression, accepting that encoding would be significantly slower.

The development was influenced by:
1. **Early lossless codecs** (Shorten, La) that showed potential for high compression
2. **Arithmetic coding research** from data compression literature
3. **Adaptive filtering techniques** from speech coding
4. **Optimal prediction theory** from signal processing

Key innovations included:
- Generalized stereo decorrelation (GSD)
- Multi-layer adaptive filtering
- Range coder (arithmetic coding) for entropy coding
- Floating-point prediction coefficients

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| 1.0 | 2000 | Initial release, basic lossless compression |
| 2.x | 2001 | Improved compression, early stereo support |
| 3.x | 2001 | Enhanced prediction, APEv2 tagging |
| 4.0 | December 2001 | Generalized stereo decorrelation, optimal predictor |
| 4.5 | 2003 | Better compression, bug fixes, multi-channel support |
| 4.6 | 2004 | Further improvements, stability |
| 5.0 | 2010 | Floating-point WAV support, DualStream hybrid lossy/lossless |
| 5.100 | September 2016 | Final release, bug fixes |

Key milestones:
- 2000: First public release
- 2001: Introduction of GSD with version 4.0
- 2003: Multi-channel support
- 2010: Floating-point audio and DualStream mode
- 2016: Final release

### 1.3 Current Adoption
- **Primary use cases today:** Maximum-efficiency archival, audio quality comparison benchmarks, specialized audio archives
- **Platforms with native support:** Windows, Linux, macOS, FreeBSD
- **Major services using this format:** None (niche archival format)
- **Hardware support:** No hardware support
- **Status:** Legacy — primarily used for archival comparisons and benchmarks

OptimFROG is primarily used today in:
- Audio codec comparison benchmarks (a reference point for maximum compression)
- Specialized archival projects requiring maximum density
- Research into lossless compression limits

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Waveform / Predictive
- **Core algorithm:** Multi-layer adaptive filtering with range coding
- **Loss mechanism:** Lossless — uses sophisticated multi-stage prediction with arithmetic/range coding
- **Frame-based vs sample-based:** Frame-based; variable-length frames with error detection
- **Fixed vs variable frame size:** Variable frame size, dynamically determined

OptimFROG represents the most complex approach to lossless audio compression among common codecs:

1. **Multi-layer adaptive filtering** for accurate signal modeling
2. **Floating-point coefficients** for precise prediction
3. **Range coding** for near-optimal entropy coding
4. **Generalized stereo decorrelation** for inter-channel redundancy reduction

### 2.2 High-Level Encoding Flow (Block Diagram)
```
Input PCM Samples (WAV Format)
      │
      ▼
[Pre-processing: WAV header parsing, format validation]
      │
      ▼
[Channel Decorrelation: Generalized stereo decorrelation (GSD)]
      │
      ▼
[Adaptive Filtering: Multi-layer filter cascade]
      │
      │   Stage 1: High-order LMS filter (floating-point)
      │   Stage 2: Medium-order LMS filter
      │   Stage 3: Low-order LMS filter
      │   Stage 4: Optional additional stages
      │
      ▼
[Prediction: Multiple specialized predictors]
      │
      ▼
[Entropy Coding: Range coder (arithmetic coding)]
      │
      │   - Precise probability modeling
      │   - Context-based probability estimation
      │   - Adaptive probability updates
      │
      ▼
[Bitstream Packing: Chunk format + optional correction stream]
      │
      ▼
Output OptimFROG Stream (.ofr/.ofs)
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | Variable | Depends on frame size |
| Frame size | Variable | Dynamically determined |
| Max channels | 8 | Integer PCM modes |
| Max bit depth | 32-bit integer, 32-bit float | Multiple format support |
| Max sample rate | 384 kHz (format limit) | Depends on version |
| Bitrate range | N/A | Lossless — bitrate varies with content |
| Complexity | O(n × complexity) | Slowest encoding among lossless codecs |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       4F 46 52 20     OFR       OptimFROG file signature
       or 4F 46 52 58           OFRX      OptimFROG Safe signature
```

The "OFR " signature (with trailing space) is the standard OptimFROG identifier, while "OFRX" denotes the "Safe" variant with enhanced error detection.

### 3.2 File-Level Header Layout
OptimFROG uses a chunk-based format with specific chunk identifiers:

```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------
0x0000   4B      Magic                  char[4]     "OFR " or "OFRX"
0x0004   4B      Header Size            uint32 LE   12 (4.5alpha), 15, or 17 bytes
0x0008   8B      Number of Samples      uint48 LE   Total samples (all channels)
0x0010   1B      Format ID             uint8       PCM format type
0x0011   1B      Channel Config        uint8       0=mono, 1=stereo
0x0012   4B      Sample Rate           uint32 LE   Sample rate in Hz
0x0016   2B      Version Info          uint16 LE   Packed version data
0x0018   1B      Encoding Parameters   uint8       Speed + method
```

#### Format ID Values
| Value | Format | Description |
|-------|--------|-------------|
| 0 | u8 | Unsigned 8-bit PCM |
| 1 | s8 | Signed 8-bit PCM |
| 2 | u16 | Unsigned 16-bit PCM |
| 3 | s16 | Signed 16-bit PCM |
| 4 | u24 | Unsigned 24-bit PCM |
| 5 | s24 | Signed 24-bit PCM |
| 6 | u32 | Unsigned 32-bit PCM |
| 7 | s32 | Signed 32-bit PCM |
| 8 | f32_1 | 32-bit float (-1.0 to 1.0 range) |
| 9 | f32_16 | 32-bit float (scaled by 32768) |
| 10 | f32_24 | 32-bit float (scaled by 8388608) |

#### Encoding Method Values
| Bits | Method | Speed | Compression |
|------|--------|-------|-------------|
| 0 | fast | Fastest | Lowest |
| 1 | normal | Medium | Medium |
| 2 | high | Slow | High |
| 3 | extra | Very slow | Very high |
| 4 | best | Slowest | Highest |
| 5 | ultra | Ultra slow | Maximum |
| 6 | insane | Slowest | Benchmark |

### 3.3 Chunk Structure
OptimFROG files consist of chunks:

```
Chunk Types:
  "HEAD" - WAV file header (duplicate of original)
  "COMP" - Compressed audio data
  "CORR" - Correction data (DualStream mode)
  "TAIL" - WAV file trailer (original file end)
```

#### Compressed Audio Chunk (COMP)
```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------
0x0000   4B      Chunk ID                char[4]     "COMP"
0x0004   4B      Chunk Size              uint32 LE   Size of compressed data
0x0008   4B      CRC                     uint32 LE   Probably CRC of chunk
0x000C   4B      Samples in Block        uint32 LE   Number of samples in this block
0x0010   1B      Format ID               uint8       Same as header
0x0011   1B      Channel ID              uint8       Same as header
0x0013   2B      Packing Methods         uint16      reader_id << 11 | filter_id << 6 | output_mode_id
0x0015   N B     Compressed Data         bytes       Range-coded audio data
```

#### Filter IDs
| Value | Filter Description |
|-------|-------------------|
| 1 | Basic filter |
| 2 | Improved filter |
| 3 | Advanced filter |
| 4 | Expert filter |

### 3.4 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Unsigned integer | Yes | Rarely used |
| 8-bit | Signed integer | Yes | |
| 16-bit | Signed integer | Yes | Standard CD audio |
| 24-bit | Signed integer | Yes | High-resolution standard |
| 32-bit | Signed integer | Yes | Professional audio |
| 32-bit | IEEE float | Yes | Multiple scaling modes |
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
| 384000 | Ultra-high-res | Yes | [NEEDS VERIFICATION] |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Pre-Processing Stage
- **DC offset removal:** Implicit in prediction
- **Pre-emphasis filter:** None
- **Windowing function:** Variable block sizes
- **Level normalization:** None
- **Stereo decorrelation pre-step:** Generalized stereo decorrelation (GSD)

### 4.2 Analysis / Transform Stage

#### Generalized Stereo Decorrelation (GSD)
OptimFROG introduced a sophisticated stereo decorrelation technique:

```
GSD Process:
  1. Compute correlation matrix between L and R channels
  2. Apply optimal rotation matrix R in signal space:
     M = (L + R) / sqrt(2)      [Mid component]
     S = (L - R) / sqrt(2)      [Side component]
     
  3. For more complex stereo, apply channel mixing matrix:
     [C1]   [m11 m12] [L]
     [C2] = [m21 m22] [R]
     
  4. Encode transformed channels (C1, C2) independently
  5. Store mixing matrix coefficients for decoding
  
Advantages over simple M/S:
  - Adapts to actual channel correlation
  - Better compression for non-standard stereo
  - Handles phase differences optimally
```

#### Adaptive Filtering (Multi-Layer)
```
Filter Stages:
  Stage 1: Primary adaptive LMS filter (floating-point coefficients)
    - High-order filter (configurable)
    - Uses floating-point arithmetic for precision
    - Adapts coefficients per-block
    
  Stage 2: Secondary filter processing residuals
    - Medium-order filter
    - Further reduces remaining correlation
    
  Stage 3: Optional additional filter layers
    - Low-order filters
    - Final polishing of residuals
    
  Output: Final residuals for entropy coding
  
Key Innovation:
  Uses floating-point arithmetic internally (unlike most lossless codecs)
  Coefficients adapt per-block
  Provides superior prediction accuracy
```

**LMS (Least Mean Squares) Filter Algorithm:**
```
For each sample n:
  1. Compute prediction:
     ŷ[n] = Σ(k=1 to p) w[k] × x[n-k]
     
  2. Compute error:
     e[n] = x[n] - ŷ[n]
     
  3. Update weights (LMS rule):
     w[k] = w[k] + μ × e[n] × x[n-k]
     
  where μ is the step size (adaptation rate)
```

### 4.3 Psychoacoustic Model (Not Applicable)
OptimFROG is a **lossless** codec. No psychoacoustic modeling is performed.

### 4.4 Quantization
OptimFROG uses **no quantization** — it is a purely lossless codec.

### 4.5 Stereo Encoding Modes
| Mode | Description | Implementation |
|------|-------------|----------------|
| Independent | L and R encoded separately | Default for mono |
| M/S (Mid-Side) | M=(L+R)/2, S=(L-R)/2 | Simple stereo decorrelation |
| Generalized | Optimal linear transformation | GSD — unique to OptimFROG |

### 4.6 Entropy / Lossless Coding Stage
```
Method: Range Coder (Arithmetic Coding)

Range coder is a form of arithmetic coding:

Encoding Process:
  1. Maintain interval [low, high)
  2. For each symbol:
     - Get probability P of symbol
     - Subdivide interval proportionally
     - Update: interval_size = interval_size × P
     - Output bits as interval narrows
     
  3. Output final interval value
  
Decoding Process:
  1. Initialize interval from known bounds
  2. For each symbol:
     - Determine which subdivision current value falls in
     - Output symbol
     - Update interval to selected subdivision
     
Properties:
  - Achieves near-entropy coding efficiency
  - Uses precise probability estimation
  - Adapts to local statistics
  - More efficient than Huffman for correlated symbols
  - Computationally more expensive than Huffman/Rice
```

### 4.7 Encoder Settings / Quality Modes

#### Compression Methods and Speeds
| Method | Speed | Compression | Description |
|--------|-------|-------------|-------------|
| fast / 0 | Fastest | Lowest | Quick encoding |
| normal / 1 | Medium | Medium | Balanced |
| high / 2 | Slow | High | Better compression |
| extra / 3 | Very slow | Very high | Excellent compression |
| best / 4 | Slowest | Highest | Maximum compression |
| ultra | Ultra slow | Maximum | Extreme compression |
| insane | Slowest | Maximum | Benchmark mode |

#### Encoding Speed Multipliers (Intel Core i7 6700HQ @ 2.6GHz)
| Preset | Encode Speed | Decode Speed |
|--------|-------------------------|--------------|
| 0 (fast) | 140x real-time | 138x real-time |
| 1 (normal) | 66.9x real-time | 96.9x real-time |
| 2 (high) | ~20x real-time | ~80x real-time |
| 3 (extra) | ~10x real-time | ~60x real-time |
| 4 (best) | ~5x real-time | ~50x real-time |

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. Scan for "OFR " or "OFRX" magic bytes
   - Start from file beginning
   - Match exactly 4 bytes
   
2. Parse and validate header chunk
   - Read all header fields
   - Validate format parameters
   
3. Iterate through COMP chunks
   - Each chunk contains CRC for validation
   - Read chunk size and data
   
4. Handle TAIL chunk
   - Marks end of audio data
```

#### Seeking
- **Seek table:** Not natively stored
- **Method:** Scan through chunks to find target position
- **Precision:** Chunk-level (block-level) seeking
- **OFS variant:** "OptimFROG Safe" has enhanced error detection

### 5.2 Core Decode Pipeline
```
1. Read file header
   ├── Verify magic ("OFR "/OFRX")
   ├── Parse WAV header info
   ├── Read format parameters
   └── Initialize decoder

2. Process chunks iteratively
   ├── Read chunk ID and size
   ├── For COMP chunks:
   │   ├── Read block parameters
   │   ├── Read compressed data
   │   ├── Validate CRC (if present)
   │   └── Range decode audio
   ├── Handle CORR chunks (DualStream)
   └── Continue until TAIL chunk

3. Range decode residuals
   ├── Probability estimation
   └── Symbol reconstruction

4. Apply inverse filters
   └── Cascade in reverse order of encoding

5. Apply inverse stereo decorrelation
   └── Reverse GSD transformation
   
6. Output samples
   └── Format conversion if needed
```

### 5.3 Error Concealment
- **Corrupt chunk detection:** CRC per chunk (OFS has enhanced detection)
- **Concealment method:** Replace corrupt block with silence
- **OFS (Safe):** Enhanced error detection for archival
- **Error resilience:** Designed for streaming over unreliable networks

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** OptimFROG chunk format
- **Overhead:** Variable (depends on compression level)
- **Seeking in native container:** Limited — chunk-based
- **Multiple streams in native container:** No

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| OptimFROG (native) | Yes | Limited | APEv2 | Chunk-based format |
| Matroska/MKA | Yes | Yes | Full | Via Matroska codec |
| FLAC | No | N/A | N/A | Different format |
| MP4/M4A | No | N/A | N/A | Not designed for MP4 |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** APEv2 (stored at end of file)
- **Tag block location:** End of file (after audio chunks)
- **Tag block identifier:** "APETAGEX"
- **ID3v1:** Also supported for compatibility

### 7.2 Standard Tag Fields — Complete Reference
| Tag Field | Internal Key (APEv2) | Max Length | Character Encoding | Multi-value | Notes |
|-----------|---------------------|------------|-------------------|-------------|-------|
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
| Encoder | ENCODER | 64 bytes | UTF-8 | No | Software and version |
| ReplayGain Track Gain | REPLAYGAIN_TRACK_GAIN | 20 bytes | ASCII | No | Format: "-6.20 dB" |
| ReplayGain Track Peak | REPLAYGAIN_TRACK_PEAK | 20 bytes | ASCII | No | Format: "0.998459" |

### 7.3 Cover Art Storage
OptimFROG supports embedded cover art via APEv2's "COVER ART" field.

### 7.4 Metadata Compatibility Matrix
| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| ID3v1 | ✓ | ✓ | ✓ | Lowest |
| APEv2 | ✓ | ✓ | ✓ | Highest |

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   optimfrog         # used with -c:a
AV_CODEC_ID:        AV_CODEC_ID_OPTIMFOOG  # C constant in libavcodec/codec_id.h
Format Name (CLI):  optimfrog          # used with -f
Encoder(s):         NONE — FFmpeg does NOT encode OptimFROG
Decoder(s):         optimfrog (native)  # ffmpeg -decoders | grep -i optimfrog
Muxer(s):           optimfrog          # Limited
Demuxer(s):         optimfrog          # ffmpeg -demuxers | grep -i optimfrog
```

### 8.2 FFmpeg Encoding — NOT SUPPORTED
FFmpeg does **NOT** support encoding OptimFROG format. OptimFROG encoding requires the official OptimFROG encoder.

### 8.3 FFmpeg Decoding — Full CLI Reference

```bash
# Decode OptimFROG to WAV
ffmpeg -i input.ofr \
  -c:a pcm_s16le \
  output.wav

# Decode OptimFROG to FLAC
ffmpeg -i input.ofr \
  -c:a flac -compression_level 8 \
  output.flac

# Extract audio stream without conversion
ffmpeg -i input.ofr \
  -c:a copy \
  output.ofr

# Probe OptimFROG file information
ffprobe -v quiet -print_format json -show_streams -show_format input.ofr

# Extract metadata
ffprobe -v quiet -print_format json -show_format input.ofr | jq .format.tags
```

### 8.4 FFmpeg Decoding — libavcodec C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>

// ─── Complete decode program skeleton ────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.ofr", NULL, NULL);
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
ffprobe -v quiet -print_format json -show_format input.ofr | jq .format.tags

# Strip all metadata
ffmpeg -i input.ofr -c:a copy -map_metadata -1 output.ofr

# Note: FFmpeg cannot write APEv2 tags to OptimFROG files natively
# Use external tools like: apetag, foobar2000, tag
```

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
```
OptimFROG does not store a seek table.
Seeking is performed by:
  1. Scanning through chunks from file start
  2. Tracking sample count as chunks are processed
  3. Stopping when target sample is reached
```

### 9.2 Gapless Playback Data
```
Encoder delay:   0 samples (OptimFROG adds no padding)
Padding:         0 samples
Storage location: TAIL chunk contains original file end markers
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | Chunk-based, error tolerant |
| Algorithmic encoder delay | Variable | Depends on block size |
| Live encoding feasible | No | Too slow for real-time |
| HTTP progressive download | Yes | Supported |
| HTTP Live Streaming (HLS) | No | Not a standard HLS segment format |
| DASH streaming | No | Not commonly used |
| WebRTC / RTP transport | No | Not designed for real-time transport |
| Minimum decode buffer | 1 chunk | Variable size |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | Layout Name | Channel Order | Notes |
|----------|-------------|---------------|-------|
| 1 | Mono | C | |
| 2 | Stereo | L, R | Best compression |
| 3 | 2.1 | L, R, LFE | |
| 4 | Quad | FL, FR, RL, RR | |
| 5 | 5.0 | FL, FR, C, SL, SR | |
| 6 | 5.1 | FL, FR, C, LFE, SL, SR | |
| 8 | 7.1 | FL, FR, C, LFE, BL, BR, SL, SR | [NEEDS VERIFICATION] |

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
| Max bit depth | 32-bit integer, 32-bit float | Multiple modes |
| Max sample rate | 384 kHz (format) | [NEEDS VERIFICATION] |
| Float support | 32-bit float | Three scaling modes |
| DSD support | No | Not supported |
| 20-bit support | Yes | Common in professional audio |
| 24-bit support | Yes | High-res standard |

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

| Platform / API | Encode | Decode | Notes |
|----------------|--------|--------|-------|
| x86 SIMD | No | Yes | SSE optimization in reference |
| ARM NEON | No | Limited | Not officially optimized |
| CUDA/NVENC | No | No | Not applicable |

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Decoder Issues
| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| No OptimFROG encoder | All versions | Use official encoder |
| Limited format support | All versions | Some variants may not decode |
| Floating-point modes | Limited | May not be fully supported |

### 14.2 Interoperability Issues
- **FFmpeg → Official decoder:** Generally compatible
- **Official → FFmpeg decode:** Generally compatible
- **Version compatibility:** Older versions may have issues with newer formats

### 14.3 Edge Cases to Handle in Converter
- **Empty file:** Invalid OptimFROG file
- **All-silence audio:** Encodes very efficiently
- **Corrupt chunk:** CRC failure, block replaced with silence
- **Truncated file:** Decoder stops at last valid chunk
- **Floating-point audio:** Special handling required

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM OptimFROG

| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → FLAC | `ffmpeg -i in.ofr -c:a flac -compression_level 8 out.flac` | All tags | Lossless |
| → ALAC | `ffmpeg -i in.ofr -c:a alac out.m4a` | Via MP4 atoms | Lossless |
| → WAV | `ffmpeg -i in.ofr -c:a pcm_s16le out.wav` | RIFF INFO | Lossless |
| → MP3 | `ffmpeg -i in.ofr -c:a libmp3lame -q:a 0 out.mp3` | Via ID3v2 | Generation loss |
| → AAC | `ffmpeg -i in.ofr -c:a aac -b:a 256k out.m4a` | Via MP4 | Generation loss |
| → Opus | `ffmpeg -i in.ofr -c:a libopus -b:a 128k out.opus` | Via Ogg | Generation loss |

### 15.2 Converting TO OptimFROG
FFmpeg does **NOT** support encoding OptimFROG. Use the official OptimFROG encoder from losslessaudio.org.

### 15.3 Lossless Round-Trip Verification
```bash
# Decode
ffmpeg -i original.ofr -c:a pcm_s24le decoded.wav

# Compare checksums
ffmpeg -i original.ofr -map 0:a -f framemd5 original.md5
ffmpeg -i decoded.wav -map 0:a -f framemd5 decoded.md5
diff original.md5 decoded.md5   # Empty diff = bit-perfect
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| OptimFROG Reference | C/C++ | Proprietary | Reference | Reference | losslessaudio.org |
| FFmpeg libavcodec | C | LGPL 2.1+ | N/A | 7/10 | https://ffmpeg.org |
| foobar2000 plugin | C++ | Proprietary | Reference | Reference | foobar2000 |
| SDK | C/C++ | Proprietary | Reference | Reference | losslessaudio.org/SDK.php |

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **Official Site:** http://losslessaudio.org/
- **SDK Documentation:** http://losslessaudio.org/SDK.php
- **Multimedia Wiki:** https://wiki.multimedia.cx/index.php/OptimFROG

### Technical Resources
- FFmpeg codec support: `ffmpeg -decoders | grep optimfrog`
- Hydrogenaudio OptimFROG page: https://wiki.hydrogenaudio.org/index.php?title=OptimFROG
- Wikipedia: https://en.wikipedia.org/wiki/OptimFROG

### Academic Papers
- None published (closed development)

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] Identify FFmpeg built with OptimFROG support (verify `ffmpeg -decoders | grep optimfrog`)
- [ ] Note: FFmpeg does NOT encode OptimFROG — must use official encoder
- [ ] Handle platform-specific OptimFROG encoder availability

### Decoding Pipeline
- [ ] Implement sync word search for "OFR " or "OFRX" magic
- [ ] Parse header chunk (HEAD)
- [ ] Validate WAV format information
- [ ] Process COMP chunks iteratively
- [ ] Validate chunk CRC (if present)
- [ ] Implement range decoding
- [ ] Apply inverse filters
- [ ] Apply inverse stereo decorrelation
- [ ] Handle CORR chunks (DualStream mode)
- [ ] Flush decoder at TAIL chunk

### Metadata
- [ ] Read APEv2 tags from end of file
- [ ] Read ID3v1 tags from end of file
- [ ] Preserve tag priority (APEv2 > ID3v1)
- [ ] Cover art extraction from tags

### Quality & Verification
- [ ] Implement chunk-level CRC verification
- [ ] Track corrupted chunks for error reporting
- [ ] Test with: silence, full-scale, multi-channel, high-resolution, floating-point files

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
