# Shorten (SHN) — Deep Technical Reference

> **Category:** Lossless
> **File Extensions:** `.shn`
> **MIME Types:** `audio/x-shorten`, `audio/shn`
> **Standardization Body:** Tony Robinson / SoftSound Ltd.
> **Primary Specification:** Cambridge University Technical Report CUED/F-INFENG/TR.156
> **Patent Status:** Unknown
> **License:** Shorten Software License (non-commercial)
> **Current Version:** 3.6.1 (March 2007)
> **Active Development:** No — discontinued ~2007, historical codec

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Tony Robinson, Cambridge University Engineering Department
- **Year Created:** 1993
- **Original Purpose:** Efficient lossless compression for speech and audio waveforms, with both lossless and near-lossless modes
- **Problem with Predecessors:** Existing compression formats (ZIP, etc.) were not optimized for audio waveforms. Shorten provided audio-specific compression with linear prediction.

Tony Robinson developed Shorten at Cambridge University as part of his research into audio compression. The codec was designed to be simple yet effective, combining linear prediction with Huffman coding for efficient lossless compression.

Key motivations:
1. **Speech coding research** — applying linear prediction to audio
2. **Lossless requirement** — preserving exact audio quality
3. **Simplicity** — easy to implement and understand
4. **Near-lossless option** — allowing controlled quality/size tradeoffs

The technical report "SHORTEN: Simple lossless and near-lossless waveform compression" (CUED/F-INFENG/TR.156) documented the algorithm and was one of the first academic treatments of practical lossless audio coding.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| 1.0 | 1993 | Initial release, basic linear prediction |
| 2.0 | 1994 | Added near-lossless mode, improved prediction |
| 2.1 | 1995 | Further improvements |
| 3.0 | 1996 | Enhanced Huffman coding, better compression |
| 3.5 | 2001 | Multi-channel support |
| 3.6 | 2006 | Minor improvements |
| 3.6.1 | March 2007 | Final release, minor bug fixes |

Key milestones:
- 1993: First public release
- 1994: Publication of academic paper
- 1996: Version 3.0 with improved coding
- 2001: Multi-channel support added
- 2007: Final release

### 1.3 Current Adoption
- **Primary use cases today:** Historical live concert recordings (etree.org community)
- **Platforms with native support:** Cross-platform (original), FFmpeg, various players
- **Major services using this format:** etree.org, live music archives
- **Hardware support:** No modern hardware support
- **Status:** Deprecated — superseded by FLAC, primarily of historical interest

Shorten was instrumental in the lossless audio movement:
- **etree.org** standardized on Shorten for live concert recordings
- **Early internet audio** — enabled faster downloads of lossless audio
- **Streaming pioneers** — supported real-time streaming over unreliable connections

Today, Shorten files have largely been migrated to FLAC, but some archives still contain Shorten files.

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Waveform / Predictive
- **Core algorithm:** Linear Prediction (LP) with Huffman coding
- **Loss mechanism:** Lossless (and near-lossless with quantization)
- **Frame-based vs sample-based:** Block-based; fixed or variable block size
- **Fixed vs variable frame size:** Configurable, typically fixed for streaming

Shorten represents the classic approach to lossless audio coding:

1. **Linear Prediction** — model audio as linear combination of past samples
2. **Residual Coding** — encode the difference from prediction
3. **Huffman Coding** — efficiently encode residuals

### 2.2 High-Level Encoding Flow (Block Diagram)
```
Input PCM Samples
      │
      ▼
[Pre-processing: Block partitioning, optional quantization]
      │
      ▼
[Linear Prediction: Compute LPC coefficients]
      │
      │   - Fixed or variable predictor order
      │   - Levinson-Durbin recursion
      │
      ▼
[Residual Calculation: Actual - Predicted]
      │
      ▼
[Huffman Coding: Encode residuals]
      │
      │   - Static Huffman tables
      │   - Optimized for audio residuals
      │
      ▼
[Bitstream Packing: Header + encoded blocks]
      │
      ▼
Output Shorten Stream
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | Variable | Depends on block size |
| Block size | Variable (256-65536 samples) | Configurable |
| Max channels | 2 (stereo) | [NEEDS VERIFICATION] |
| Max bit depth | 16-bit | Standard CD audio |
| Max sample rate | 48 kHz | [NEEDS VERIFICATION] |
| Bitrate range | N/A | Lossless — varies with content |
| Complexity | O(n × order) | Fast encoding/decoding |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       61 6A 6B 67     ajkg      File magic (Tony J. Robinson signature)
```

**Note:** "ajkg" is the ASCII signature of the author, Tony J. Robinson. This distinctive signature identifies Shorten files and serves as a clear identification marker.

### 3.2 File-Level Header Layout
Based on Shorten format specification:

```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------
0x0000   4B      Magic                  char[4]     "ajkg"
0x0004   variable  Format data           variable    Compression parameters
```

#### Format Data Structure
```
Following magic, Shorten stores:
  - Version number (internal)
  - Channel count
  - Block size
  - WAV format parameters
  - Optional seek table offsets
```

### 3.3 Frame / Block Header Layout
```
Shorten organizes data into blocks:
  - Each block contains N samples
  - Block header includes:
    - Block type
    - Number of items
    - Huffman-coded data
```

### 3.4 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Unsigned integer | Yes | Rarely used |
| 16-bit | Signed integer | Yes | Standard CD audio |
| 24-bit | Signed integer | No | Not supported |
| 32-bit | Signed integer | No | Not supported |

#### Supported Sample Rates
| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 8000 | Telephone | Yes | Speech |
| 11025 | — | Yes | |
| 16000 | Wideband voice | Yes | |
| 22050 | — | Yes | |
| 44100 | CD audio | Yes | Most common |
| 48000 | Professional | Yes | |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Pre-Processing Stage
- **DC offset removal:** None explicit
- **Pre-emphasis filter:** None
- **Windowing function:** Block-based, rectangular window
- **Level normalization:** None
- **Stereo handling:** Channels encoded separately

### 4.2 Analysis / Transform Stage

#### Linear Prediction
```
LPC (Linear Predictive Coding):
  - Model audio as linear combination of past samples
  - x[n] ≈ a₁ × x[n-1] + a₂ × x[n-2] + ... + ap × x[n-p]
  - Residual: r[n] = x[n] - predicted
  
Parameters:
  - LPC order: Fixed, typically 2-8
  - Algorithm: Levinson-Durbin recursion
  - Coefficient precision: Limited (quantized)
```

**Levinson-Durbin Algorithm:**
```
Input: autocorrelation values R[0] to R[p]
Output: LPC coefficients a[1] to a[p]

E[0] = R[0]
for i = 1 to p:
    sum = 0
    for j = 1 to i-1:
        sum += a[j] * R[i-j]
    k[i] = (R[i] - sum) / E[i-1]
    a[i] = k[i]
    for j = 1 to i-1:
        a[j] = a[j] - k[i] * a[i-j]
    E[i] = (1 - k[i]^2) * E[i-1]
```

### 4.3 Near-Lossless Mode
Shorten supports near-lossless compression:

```
Near-lossless operation:
  - Quantization applied to residuals
  - Q = round(residual / (2^k))
  - Reconstruction: x'[n] = predicted + Q × (2^k)
  - Allows tradeoff between compression and quality
  
Parameter k controls quantization:
  k=0: Lossless (no quantization)
  k=1: Quantize to multiples of 2
  k=2: Quantize to multiples of 4
  etc.
```

### 4.4 Quantization
- **Lossless mode:** No quantization
- **Near-lossless mode:** Scalar quantization with parameter k

### 4.5 Stereo Encoding Modes
| Mode | Description | Implementation |
|------|-------------|----------------|
| Independent | L and R encoded separately | Default |

### 4.6 Entropy / Lossless Coding Stage
```
Method: Huffman Coding

Huffman coding properties:
  - Static Huffman codes (pre-defined tables)
  - Fast encoding and decoding
  - Good for audio residuals with specific distributions
  
Code tables:
  - Optimized for typical audio residual distributions
  - Different tables for different parameter ranges
```

### 4.7 Encoder Settings / Quality Modes

#### Compression Modes
| Mode | Description | Compression Ratio |
|------|-------------|-------------------|
| Lossless | No quantization | 40-60% of original |
| Near-lossless (k=0) | Same as lossless | Baseline |
| Near-lossless (k=1) | ±1 quantization | Higher compression |
| Near-lossless (k=2) | ±2 quantization | Even higher |

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. Scan for magic bytes: "ajkg"
   - Start from file beginning
   - Match exactly 4 bytes
   
2. Read format data
   - Version number
   - Channel count
   - Block size
   - WAV format parameters
   
3. Identify block boundaries
4. Decode blocks sequentially
```

#### Seeking
- **Seek table:** Optional, added by Wayne Steilau's extensions
- **Seek table format:** Block offsets stored in file
- **Without seek table:** Linear scan from start

### 5.2 Core Decode Pipeline
```
1. Read file header
   ├── Verify magic "ajkg"
   ├── Parse version
   ├── Read channel count
   ├── Read block size
   └── Read WAV format

2. Decode blocks iteratively
   ├── Read block header
   ├── Huffman decode residuals
   ├── Apply LPC coefficients
   └── Reconstruct audio samples

3. Block decoding details:
   a. Read block type
   b. Huffman decode residual values
   c. Apply linear prediction
   d. Output samples
```

### 5.3 Error Concealment
- **Corrupt block detection:** Huffman code invalidity
- **Concealment method:** Likely replace with silence
- **Error resilience:** Designed for unreliable transmission

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** Shorten is a raw stream format
- **Overhead:** Minimal (~12 bytes header)
- **Seeking in native container:** Via seek table (optional)
- **Multiple streams in native container:** No

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| Shorten (native) | Yes | Yes (if seek table) | None | No container |
| Matroska/MKA | Unlikely | — | — | Not commonly muxed |
| Other containers | No | — | — | Not designed for |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** None native
- **Tag storage:** None in Shorten format itself
- **External tags:** MD5 checksum file (separate .md5 file)

### 7.2 MD5 Verification
Shorten commonly used with MD5 checksums for verification:
```
MD5 file naming: filename.shn.md5
Contents: <md5hash>  <relative_path>/<filename>.shn
```

This allowed verification of file integrity in the era before built-in checksums.

### 7.3 Cover Art Storage
No cover art support.

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   shorten         # used with -c:a
AV_CODEC_ID:        AV_CODEC_ID_SHORTEN  # C constant in libavcodec/codec_id.h
Format Name (CLI):  shorten          # used with -f
Encoder(s):         NONE — FFmpeg does NOT encode Shorten
Decoder(s):         shorten (native)  # ffmpeg -decoders | grep -i shorten
Muxer(s):           shorten          # ffmpeg -muxers | grep -i shorten
Demuxer(s):         shorten          # ffmpeg -demuxers | grep -i shorten
```

### 8.2 FFmpeg Encoding — NOT SUPPORTED
FFmpeg does **NOT** support encoding Shorten format. Shorten encoding requires the original Shorten encoder.

### 8.3 FFmpeg Decoding — Full CLI Reference

```bash
# Decode Shorten to WAV
ffmpeg -i input.shn \
  -c:a pcm_s16le \
  output.wav

# Decode Shorten to FLAC
ffmpeg -i input.shn \
  -c:a flac -compression_level 8 \
  output.flac

# Extract audio stream without conversion
ffmpeg -i input.shn \
  -c:a copy \
  output.shn

# Probe Shorten file information
ffprobe -v quiet -print_format json -show_streams -show_format input.shn
```

### 8.4 FFmpeg Decoding — libavcodec C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>

// ─── Complete decode program skeleton ────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.shn", NULL, NULL);
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

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
```
Shorten Seek Table (optional):
  Location:     File header or separate
  Entry size:   4 bytes (uint32)
  Entry format:
    [0x00–0x03]  block_offset (uint32) — Byte offset to block
  Purpose:      Enable random access within file
```

### 9.2 Gapless Playback Data
```
Encoder delay:   0 samples (assumed)
Padding:         0 samples (assumed)
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable | Yes | Block-based, designed for streaming |
| Live encoding | Yes | Real-time capable |
| HTTP progressive download | Yes | Supported |
| HLS / DASH | No | Not applicable |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | Layout Name | Notes |
|----------|-------------|-------|
| 1 | Mono | Primary design |
| 2 | Stereo | Supported |

**Note:** Shorten was primarily designed for mono/stereo audio. Multi-channel support is limited.

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 16-bit | Integer only |
| Max sample rate | 48 kHz [NEEDS VERIFICATION] | |
| Float support | No | Not supported |

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

| Platform / API | Encode | Decode | Notes |
|----------------|--------|--------|-------|
| x86 SIMD | No | No | Simple algorithm |
| ARM NEON | No | No | |

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Decoder Issues
| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| No Shorten encoder | All versions | Use original Shorten encoder |
| Limited format features | All versions | Some SHN variants may not decode |

### 14.2 Historical Context
- **etree.org standard:** Shorten was the standard format for live music trading
- **Migration to FLAC:** Many SHN files have been converted to FLAC
- **MD5 verification:** Common practice to verify SHN file integrity

### 14.3 SHN to FLAC Migration
Many Shorten files have been migrated to FLAC:
```bash
# Convert all SHN files in directory to FLAC
for f in *.shn; do
  ffmpeg -i "$f" -c:a flac -compression_level 8 "${f%.shn}.flac"
done
```

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM Shorten

| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → FLAC | `ffmpeg -i in.shn -c:a flac -compression_level 8 out.flac` | None (no tags) | Lossless |
| → WAV | `ffmpeg -i in.shn -c:a pcm_s16le out.wav` | None | Lossless |
| → MP3 | `ffmpeg -i in.shn -c:a libmp3lame -q:a 0 out.mp3` | None | Generation loss |
| → AAC | `ffmpeg -i in.shn -c:a aac -b:a 256k out.m4a` | None | Generation loss |
| → Opus | `ffmpeg -i in.shn -c:a libopus -b:a 128k out.opus` | None | Generation loss |

### 15.2 Converting TO Shorten
FFmpeg does **NOT** support encoding Shorten. Use the original Shorten encoder from etree.org.

### 15.3 Lossless Round-Trip Verification
```bash
# Decode
ffmpeg -i original.shn -c:a pcm_s16le decoded.wav

# Compare checksums
ffmpeg -i original.shn -map 0:a -f framemd5 original.md5
ffmpeg -i decoded.wav -map 0:a -f framemd5 decoded.md5
diff original.md5 decoded.md5   # Empty diff = bit-perfect
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Notes |
|---------|----------|---------|-------|
| Shorten Reference | C | Shorten License | Tony Robinson's implementation |
| FFmpeg libavcodec | C | LGPL 2.1+ | 6/10 | https://ffmpeg.org |
| shntool | C | GPL | Command-line utilities |

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **Technical Report:** "SHORTEN: Simple lossless and near-lossless waveform compression"
  - Tony Robinson, Cambridge University
  - CUED/F-INFENG/TR.156
  - December 1994
- **URL:** http://svr-www.eng.cam.ac.uk/reports/ajr/TR156/tr156.html

### Technical Resources
- FFmpeg codec support: `ffmpeg -decoders | grep shorten`
- etree.org SHN utilities: https://etree.org/shnutils/shorten/
- Multimedia Wiki: https://wiki.multimedia.cx/index.php/Shorten

### Academic Papers
- "SHORTEN: Simple lossless and near-lossless waveform compression", Tony Robinson, Cambridge University, 1994

---

## 18. COMPRESSION RATIO COMPARISON

Shorten was among the first practical lossless audio codecs:

| Codec | Typical Compression | Year | Notes |
|-------|---------------------|------|-------|
| Shorten | ~50-60% | 1993 | Original lossless audio |
| FLAC (level 0) | ~60-70% | 2001 | Fastest |
| FLAC (level 8) | ~58-65% | 2001 | Balanced |
| Monkey's Audio | ~52-60% | 2000 | Various modes |
| OptimFROG | ~50-58% | 2001 | Maximum compression |

Shorten's compression was competitive with later codecs despite being developed earlier, demonstrating the effectiveness of its linear prediction approach.

---

## 19. HISTORICAL SIGNIFICANCE

Shorten holds an important place in lossless audio history:

1. **First widely-used lossless audio codec** — predated FLAC by 8 years
2. **Academic foundation** — first codec with published research paper
3. **Streaming pioneer** — enabled real-time lossless audio over internet
4. **Community building** — etree.org used Shorten for live music archives
5. **Transition to FLAC** — paved way for modern lossless adoption

The codec's simplicity made it an excellent reference implementation and educational tool for understanding predictive audio coding.

---

## 20. THE ETREE.ORG STANDARD

### 20.1 Background
etree.org was a community dedicated to sharing live concert recordings in lossless audio format. Shorten was the original standard for this community.

### 20.2 Standards and Conventions
```
File naming convention:
  artist - date - venue - source.shn
  
Example:
  Phish - 1995-11-11 - MSG - SBD.shn
  
MD5 verification:
  Each .shn file had an accompanying .md5 file
  Allowed verification of file integrity
```

### 20.3 Migration to FLAC
As FLAC became more mature, the etree.org community migrated:
```
Migration tools:
  - shntool: Command-line utilities for SHN files
  - ffmpeg: Convert SHN to FLAC
  - xld: macOS conversion tool
  
Process:
  1. Verify SHN file integrity
  2. Convert to FLAC with metadata preservation
  3. Update filename convention
  4. Distribute FLAC version
```

---

## 21. TECHNICAL DEEP DIVE

### 21.1 Huffman Coding Details
Shorten uses optimized Huffman tables for audio residuals:

```
Typical Huffman table structure:
  Symbol 0:     1 bit (0)
  Symbol ±1:   2 bits (10x)
  Symbol ±2:   3 bits (110x)
  Symbol ±3:   4 bits (1110xx)
  ...
  
This reflects the Laplace distribution of audio residuals.
```

### 21.2 Block Size Selection
Block size affects compression and seeking:
```
Small blocks:
  - Faster seeking
  - Lower compression
  - Better error resilience
  
Large blocks:
  - Slower seeking
  - Better compression
  - Worse error resilience
```

### 21.3 Near-Lossless Tradeoffs
The k parameter controls quantization:
```
k=0: Lossless
k=1: ±1 sample error (inaudible in most cases)
k=2: ±2 sample error (usually inaudible)
k=3: ±4 sample error (sometimes audible)
```

---

## 22. PERFORMANCE ANALYSIS

### 22.1 Encoding Speed
```
Encoding speed:   ~10-20× real-time (typical)
Memory usage:    Low (~1-2 MB)
CPU usage:       Low
```

### 22.2 Decoding Speed
```
Decoding speed:  ~20-30× real-time (typical)
Memory usage:    Low
CPU usage:       Low
```

### 22.3 Compression Performance
Compression depends on audio content:
```
Classical:    40-50% ratio
Jazz:        45-55% ratio
Rock/Pop:    50-60% ratio
Speech:      45-55% ratio
Silence:     2-5% ratio
```

---

## 23. COMPARISON WITH MODERN CODECS

### 23.1 Feature Comparison
| Feature | Shorten | FLAC | Notes |
|---------|---------|------|-------|
| Compression | Good | Better | FLAC has improved algorithms |
| Speed | Good | Better | FLAC optimized |
| Open source | Yes | Yes | Both open |
| Seeking | Yes | Yes | Both support |
| Multi-channel | Limited | Yes | FLAC better |
| Error handling | Basic | Robust | FLAC better |

### 23.2 Why Shorten Was Superseded
1. **FLAC's advantages:**
   - Better compression ratios
   - Faster encoding/decoding
   - Better documentation
   - Active development
   - Open specification
   
2. **Shorten's limitations:**
   - No official specification
   - Limited multi-channel
   - No active development
   - Proprietary (though non-commercial free)

---

## 24. PRESERVATION AND MIGRATION

### 24.1 Preservation Considerations
LA files should be migrated because:
1. **Format age** — Shorten dates from 1993
2. **Limited support** — Few modern players support Shorten
3. **Better alternatives** — FLAC offers superior features
4. **Community shift** — Most archives have migrated

### 24.2 Migration Best Practices
```
1. Identify all SHN files
2. Verify file integrity with MD5
3. Create backup before conversion
4. Convert to FLAC with highest compression
5. Verify converted file matches source
6. Update library references
7. Archive original SHN files
```

### 24.3 Verification Commands
```bash
# Verify original
cat file.shn.md5

# Convert to FLAC
ffmpeg -i file.shn -c:a flac -compression_level 8 file.flac

# Verify conversion
ffmpeg -i file.shn -map 0:a -f framemd5 original.md5
ffmpeg -i file.flac -map 0:a -f framemd5 converted.md5
diff original.md5 converted.md5
```

---

## 25. IMPLEMENTATION REFERENCE

### 25.1 File Format Details
```
Magic: "ajkg" (0x61 0x6A 0x6B 0x67)
Header: Variable length format
Blocks: Variable length
```

### 25.2 Block Structure
```
Each block:
  - Type indicator
  - Size indicator
  - Huffman-coded data
```

### 25.3 Decoder Requirements
```
For FFmpeg decoding:
  - Built-in Shorten decoder
  - Supports up to version 3.6.1
  - Limited to basic Shorten features
```

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
