# RealAudio (COOK, RAAC, RALF) — Deep Technical Reference

> **Category:** Lossy / Lossless (RALF)
> **File Extensions:** `.ra`, `.rm`, `.rmvb`, `.rmvb`
> **MIME Types:** `audio/x-pn-realaudio`, `audio/x-ralf-mpeg4`, `audio/vnd.rn-realaudio`
> **Standardization Body:** RealNetworks (proprietary, closed)
> **Primary Specification:** No public specification — reverse-engineered
> **Patent Status:** Proprietary — owned by RealNetworks
> **License:** Proprietary — RealNetworks
> **Current Version:** Various (multiple codec generations)
> **Active Development:** No — RealNetworks closed, RealMedia deprecated

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** RealNetworks, Inc. (Seattle, Washington)
- **Year Created:** 1995 (RealAudio 1.0), 1998 (Cook), 2000 (RALF)
- **Original Purpose:** Streaming audio over dial-up modems with low latency
- **Problem with Predecessors:** Early web audio required complete file download. RealAudio pioneered streaming with minimal buffering.

### 1.2 Version History
| Version | Year | Codec | Key Changes |
|---------|------|-------|-------------|
| RealAudio 1.0 | 1995 | — | Initial streaming audio |
| RealAudio G2 | 1998 | Cook | Improved quality, AAC support |
| RealAudio 8 | 1999 | Cook | Better compression |
| RealAudio 9 | 2000 | Cook | Enhanced quality |
| RealAudio 10 | 2001 | Multiple | AAC-based |
| RealAudio Lossless | 2000 | RALF | Lossless compression |
| RealMedia HD | 2010s | Various | HD support (later) |

### 1.3 Current Adoption
- **Primary use cases today:** Legacy streaming, archived internet radio
- **Platforms with native support:** None (deprecated)
- **Major services using this format:** None (historical)
- **Hardware support:** No modern hardware support
- **Status:** Deprecated — replaced by MP3, AAC, and streaming services

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **COOK Codec:** Transform-based using Modified Discrete Cosine Transform (MDCT)
- **RAAC Codec:** AAC-based lossy codec
- **RALF Codec:** Lossless codec using linear prediction

### 2.2 High-Level Encoding Flow (COOK)

```
Input PCM Samples
      │
      ▼
[Pre-processing: Blocking, windowing]
      │
      ▼
[MDCT: Modified Discrete Cosine Transform]
      │
      ▼
[Bit Allocation: Per-band bit assignment]
      │
      ▼
[Quantization: Scalar quantization of MDCT coefficients]
      │
      ▼
[Huffman Coding: Entropy coding]
      │
      ▼
[Bitstream Packing: Frame + side info]
      │
      ▼
Output RealAudio Stream
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | ~100ms | Low-latency streaming |
| Frame size | 1024 samples (COOK) | Fixed |
| Max channels | 6 (5.1 for multichannel COOK) | |
| Max bit depth | 16-bit | Integer PCM |
| Max sample rate | 44.1 kHz (COOK) | |
| Bitrate range | 6-128 kbps | Variable |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signatures
```
RealAudio (.ra):
  Offset  Bytes   ASCII     Meaning
  ------  ------  --------  -------------------
  0x0000  4       ".ra4"    RealAudio version 4
  0x0000  4       ".ra5"    RealAudio version 5
  
RealMedia (.rm):
  Offset  Bytes   ASCII     Meaning
  ------  ------  --------  -------------------
  0x0000  4       ".RMF"    RealMedia File
  
RealAudio Lossless (.ra with RALF):
  Offset  Bytes   ASCII     Meaning
  ------  ------  --------  -------------------
  varies  —       —         MIME type: audio/x-ralf-mpeg4
```

### 3.2 RealAudio File Structure

#### Version 4/5 Header
```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------
0x0000   4B      Magic                  char[4]     ".ra4" or ".ra5"
0x0004   4B      File Size              uint32 LE   Total file size
0x0008   2B      Version                uint16 LE   Codec version
0x000A   2B      Header Size            uint16 LE   Header data size
0x000C   4B      Data Offset            uint32 LE   Offset to audio data
0x0010   4B      Codec FourCC           char[4]     Codec identifier
...      ...     Codec-specific data     ...         Varies by codec
```

### 3.3 RealMedia Container Structure

```
RealMedia (.rm) Structure:
  .RMF header (RealMedia File header)
  PROP properties chunk
  MDPR media properties chunk
  CONT content description chunk
  DATA data chunk (audio/video frames)
  INDX index chunk (optional)
```

### 3.4 COOK Codec Specific Data
```
Type-specific data (8 bytes for mono, 16 for stereo):
  bytes 0-1    Sample rate index
  bytes 2-3    Bytes per frame
  bytes 4-7    Unknown
  bytes 8-11   Unused
  bytes 12-13  Joint stereo subband start (stereo only)
  bytes 14-15  Joint stereo VLC bits (stereo only)
  +4 bytes     Additional for multichannel
```

### 3.5 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Unsigned integer | No | Not supported |
| 16-bit | Signed integer | Yes | Standard |
| 24-bit | Signed integer | No | Not supported |

#### Supported Sample Rates
| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 8000 | Telephone | Yes | Low-bitrate mode |
| 16000 | Wideband | Yes | |
| 22050 | — | Yes | |
| 32000 | Broadcast | Yes | |
| 44100 | CD audio | Yes | Most common |
| 48000 | Professional | Yes | |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 COOK Algorithm

#### Modified Discrete Cosine Transform (MDCT)
```
MDCT Parameters:
  - Block size: 1024 samples
  - Window: Sine window (MDCT standard)
  - Overlap: 50% (inherent in MDCT)
  - Transform: 1024-point MDCT → 512 frequency bins
  
MDCT Formula:
  X[k] = Σ(n=0 to 2N-1) x[n] · w[n] · cos(π/N · (n + N/2) · (k + 1/2))
```

#### Subband Processing
```
COOK divides the frequency range into subbands:
  - Number of subbands: Variable (depends on bitrate)
  - Bit allocation: Per-subband based on perceptual importance
  - Quantization: Scalar quantization per subband
  
Subband structure:
  - Low frequencies: More subbands, more bits
  - High frequencies: Fewer subbands, fewer bits
```

### 4.2 Stereo Encoding Modes (COOK)
| Mode | Description | Implementation |
|------|-------------|----------------|
| Independent | L and R encoded separately | Default |
| Joint Stereo | Mid-Side coding for some bands | Used for low bitrates |
| Coupled | Some bands coupled | For multichannel |

### 4.3 Quantization
```
COOK Quantization:
  - Scalar quantization per subband
  - Quantizer step size varies by bit allocation
  - Non-uniform quantization (logarithmic)
  - Huffman coding of quantized coefficients
```

### 4.4 Entropy Coding
```
Method: Huffman Coding

Huffman properties:
  - Multiple codebooks
  - Different tables for different data types
  - Run-length encoding for zero coefficients
```

### 4.5 Bitrate Selection

| Bitrate (kbps) | Quality | Typical Use |
|----------------|---------|-------------|
| 6-8 | Very low | Dial-up streaming |
| 14.4 | Low | Early broadband |
| 20 | Medium | Standard streaming |
| 44 | High | Quality streaming |
| 64 | Very high | Near-CD quality |
| 96-128 | High | Premium streaming |

---

## 5. RALF (RealAudio Lossless) DETAIL

### 5.1 RALF Algorithm
```
RALF uses linear prediction for lossless compression:
  - Adaptive linear prediction
  - Multiple filter stages
  - Range coding for residuals
  
RALF Properties:
  - True lossless compression
  - Bitrates: ~400-700 kbps for stereo
  - Based on linear predictive coding
```

### 5.2 RALF Specifications
| Parameter | Value | Notes |
|-----------|-------|-------|
| Bit depth | 16-bit | Integer PCM |
| Sample rate | 44.1 kHz | |
| Channels | 2 (stereo) | |
| Compression | ~60-70% of original | Lossless |

---

## 6. DECODING ALGORITHM — DEEP DETAIL

### 6.1 CODEC Synchronization & Seeking

#### Sync Strategy
```
For .ra files:
  1. Read file header (.ra4/.ra5 magic)
  2. Locate audio data offset
  3. Identify codec FourCC
  4. Initialize codec with type-specific data
  
For .rm files:
  1. Read .RMF header
  2. Find MDPR chunk with codec info
  3. Find audio stream properties
  4. Process DATA chunk frames
```

### 6.2 Core Decode Pipeline (COOK)
```
1. Read frame header
   ├── Frame size
   └── Codec-specific parameters

2. Decode entropy (Huffman)
   ├── Select codebook
   └── Decode quantized coefficients

3. Dequantization
   └── Apply inverse quantizer step sizes

4. Inverse MDCT
   ├── Apply inverse window
   └── Overlap-add for continuous output

5. Post-processing
   ├── Stereo reconstruction (if joint stereo)
   └── Output samples
```

### 6.3 Error Concealment
- **Corrupt frame detection:** Invalid Huffman codes, out-of-range values
- **Concealment method:** Repeat previous frame or mute
- **Error resilience:** Limited — streaming optimized

---

## 7. CONTAINER / WRAPPER INTEGRATION

### 7.1 Native Containerization
- **Native container:** RealMedia (.rm) for video, RealAudio (.ra) for audio-only
- **Overhead:** Variable (container overhead)
- **Seeking in native container:** Via index chunk (INDX)
- **Multiple streams in native container:** Yes (audio + video)

### 7.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| RealAudio (.ra) | COOK, RALF, RAAC | Limited | Limited | Audio-only |
| RealMedia (.rm) | COOK, RALF, RAAC | Yes | Yes | Audio + video |
| Matroska/MKA | COOK, RALF | Yes | Full | Via MKV support |
| Other containers | Rarely | — | — | Not standard |

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   cook             # used with -c:a for COOK
AV_CODEC_ID:        AV_CODEC_ID_COOK  # C constant in libavcodec/codec_id.h
Format Name (CLI):  rm               # for RealMedia containers
Encoder(s):         NONE — FFmpeg does NOT encode RealAudio
Decoder(s):         cook (libavcodec) # ffmpeg -decoders | grep -i cook
Muxer(s):           rm, rtsp         # ffmpeg -muxers | grep -i rm
Demuxer(s):         rm, rtsp         # ffmpeg -demuxers | grep -i rm
```

### 8.2 FFmpeg Encoding — NOT SUPPORTED
FFmpeg does **NOT** support encoding RealAudio formats. RealAudio encoding requires RealNetworks' proprietary encoders.

### 8.3 FFmpeg Decoding — Full CLI Reference

```bash
# Decode RealAudio to WAV
ffmpeg -i input.rm \
  -c:a pcm_s16le \
  output.wav

# Extract audio from RealMedia
ffmpeg -i input.rm \
  -c:a copy \
  -map 0:a \
  output.ra

# Probe RealMedia file information
ffprobe -v quiet -print_format json -show_streams -show_format input.rm

# Decode specific audio stream
ffmpeg -i input.rm \
  -map 0:a:0 \
  -c:a pcm_s16le \
  output.wav
```

### 8.4 FFmpeg Decoding — libavcodec C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>

// ─── Complete decode program skeleton ────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.rm", NULL, NULL);
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
RealMedia Index (INDX chunk):
  Location:     End of file (after DATA chunk)
  Entry size:   Variable (depends on index type)
  Entry format:
    [0x00–0x03]  timestamp (uint32)    — Presentation time
    [0x04–0x07]  offset (uint32)       — Offset within DATA chunk
  Purpose:      Random access within file
```

### 9.2 Gapless Playback Data
```
Encoder delay:   ~100ms (streaming optimized)
Padding:         ~100ms
Storage location: Not stored — real-time format
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable | Yes | Designed for streaming |
| Algorithmic delay | ~100ms | Low-latency |
| Live encoding | Yes | Real-time capable |
| HTTP progressive download | Yes | Supported |
| RTSP/RTP | Yes | Native streaming protocol |
| HLS / DASH | No | Not supported |
| WebRTC | No | Not applicable |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts (COOK Multichannel)
| Channels | Layout Name | Notes |
|----------|-------------|-------|
| 1 | Mono | Supported |
| 2 | Stereo | Primary mode |
| 5 | 5.0 | COOK multichannel |
| 6 | 5.1 | COOK multichannel |

**Note:** Multichannel COOK requires 'cook' FourCC with multichannel extension.

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 16-bit | Integer only |
| Max sample rate | 44.1 kHz (COOK) | [NEEDS VER] |
| Float support | No | Not supported |
| High-res audio | No | Not supported |

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

| Platform / API | Encode | Decode | Notes |
|----------------|--------|--------|-------|
| x86 SIMD | No | Limited | Not optimized |
| ARM NEON | No | No | Not supported |
| Hardware decode | No | No | Deprecated |

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Decoder Issues
| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| No RealAudio encoder | All versions | Use RealNetworks encoder |
| Limited codec support | All versions | Some variants may not decode |
| Container issues | Some versions | Use demuxer with most support |

### 14.2 Interoperability Issues
- **Format variations:** Many RealAudio variants exist
- **Codec parameters:** Extradata format varies
- **Container mixing:** Some files use non-standard containers

### 14.3 Edge Cases
- **Corrupt frames:** Replace with silence
- **Missing extradata:** Decode may fail
- **Version mismatches:** Some old files may not decode

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM RealAudio

| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → FLAC | `ffmpeg -i in.rm -c:a flac -compression_level 8 out.flac` | None (streaming) | Lossless if RALF |
| → WAV | `ffmpeg -i in.rm -c:a pcm_s16le out.wav` | None | Lossless if RALF |
| → MP3 | `ffmpeg -i in.rm -c:a libmp3lame -q:a 0 out.mp3` | None | Generation loss |
| → AAC | `ffmpeg -i in.rm -c:a aac -b:a 256k out.m4a` | None | Generation loss |
| → Opus | `ffmpeg -i in.rm -c:a libopus -b:a 128k out.opus` | None | Generation loss |

### 15.2 Converting TO RealAudio
FFmpeg does **NOT** support encoding RealAudio. RealAudio encoding requires RealNetworks' proprietary encoders which are no longer available.

### 15.3 Lossless Round-Trip Verification (RALF)
```bash
# Decode RALF
ffmpeg -i input.rm -c:a pcm_s16le decoded.wav

# Compare checksums (if source was lossless)
md5sum original.wav decoded.wav
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Notes |
|---------|----------|---------|-------|
| FFmpeg libavcodec | C | LGPL 2.1+ | COOK decoder | https://ffmpeg.org |
| RealNetworks SDK | — | Proprietary | Historical only |
| Rockbox | C | Various | COOK decoder |

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **No public specification available**
- RealNetworks never published codec details
- All information from reverse engineering

### Technical Resources
- FFmpeg codec support: `ffmpeg -decoders | grep cook`
- Multimedia Wiki COOK: https://wiki.multimedia.cx/index.php/RealAudio_cook
- Multimedia Wiki RealMedia: https://wiki.multimedia.cx/index.php/RealMedia
- Wikipedia RealAudio: https://en.wikipedia.org/wiki/RealAudio

### Historical Context
- RealAudio was the dominant streaming format 1998-2004
- Replaced by MP3 streaming and later HLS/DASH
- RealNetworks closed in 2022

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] Identify FFmpeg built with COOK decoder (verify `ffmpeg -decoders | grep cook`)
- [ ] Note: FFmpeg does NOT encode RealAudio — RealNetworks encoders deprecated
- [ ] Handle missing extradata cases

### Decoding Pipeline
- [ ] Implement sync for .ra and .rm file formats
- [ ] Parse RealMedia container (RMF, PROP, MDPR, DATA, INDX)
- [ ] Extract codec-specific extradata
- [ ] Initialize COOK decoder with extradata
- [ ] Implement Huffman decoding
- [ ] Apply inverse quantization
- [ ] Apply inverse MDCT with windowing
- [ ] Handle joint stereo reconstruction
- [ ] Flush decoder at EOF

### Error Handling
- [ ] Handle missing extradata gracefully
- [ ] Implement frame error concealment
- [ ] Test with various RealAudio variants

### Metadata
- [ ] Extract metadata from CONT chunk
- [ ] No native tag preservation needed
- [ ] Streaming format — limited metadata

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
