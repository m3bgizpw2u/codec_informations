# RealAudio (COOK, RAAC, RALF) — Deep Technical Reference

> **Category:** Lossy / Lossless (RALF)
> **File Extensions:** `.ra`, `.rm`, `.rmvb`
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

RealNetworks was founded in 1994 with the goal of enabling real-time multimedia streaming over the Internet. RealAudio was the company's first product, released in April 1995, and revolutionized how audio was delivered over the web.

Key motivations for RealAudio:
1. **Streaming requirement** — play audio as it downloads, not after
2. **Low latency** — minimize delay between user action and audio playback
3. **Bandwidth adaptation** — adjust quality based on connection speed
4. **Small files** — maximize audio quality per byte

RealAudio's innovations included:
- **Adaptive stream switching** — seamlessly switch between quality levels
- **Server-push technology** — before standard HTTP streaming
- **Proprietary codecs** — optimized for streaming efficiency
- **Integrated player** — RealPlayer became ubiquitous

### 1.2 Version History
| Version | Year | Codec | Key Changes |
|---------|------|-------|-------------|
| RealAudio 1.0 | 1995 | — | Initial streaming audio, very low bitrate |
| RealAudio G2 | 1998 | Cook | Improved quality, AAC support |
| RealAudio 8 | 1999 | Cook | Better compression, multi-channel |
| RealAudio 9 | 2000 | Cook | Enhanced quality, new features |
| RealAudio 10 | 2001 | Multiple | AAC-based, improved surround |
| RealAudio Lossless | 2000 | RALF | Lossless compression |
| RealMedia HD | 2010s | Various | HD support (later) |

Key milestones:
- 1995: First RealAudio release
- 1998: Cook codec introduction with RealAudio G2
- 2000: RALF lossless codec introduced
- 2001: RealAudio 10 with AAC support
- 2010s: RealMedia HD revival attempt
- 2022: RealNetworks closure

### 1.3 Current Adoption
- **Primary use cases today:** Legacy streaming, archived internet radio, historical recordings
- **Platforms with native support:** None (deprecated)
- **Major services using this format:** None (historical)
- **Hardware support:** No modern hardware support
- **Status:** Deprecated — replaced by MP3, AAC, and streaming services

RealAudio dominated internet audio from 1995-2004:
- **Internet radio** — thousands of stations used RealAudio
- **Music sites** — early online music stores used RealAudio
- **News sites** — NPR, BBC used RealAudio for streaming
- **Live events** — concerts and sports broadcasts

The format was eventually supplanted by:
- MP3 for downloaded audio
- Shoutcast/Icecast for streaming
- Later, HLS and DASH standards

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class

#### COOK Codec
- **Type:** Transform-based
- **Core algorithm:** Modified Discrete Cosine Transform (MDCT)
- **Characteristics:** Low-latency, optimized for streaming

#### RAAC Codec
- **Type:** Transform-based
- **Core algorithm:** Based on AAC (Advanced Audio Coding)
- **Characteristics:** Better quality at higher bitrates

#### RALF Codec
- **Type:** Predictive
- **Core algorithm:** Linear prediction
- **Characteristics:** True lossless compression

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
      │   - Analyze perceptual importance
      │   - Allocate bits to bands
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
  0x0000  4       ".ra4"   RealAudio version 4
  0x0000  4       ".ra5"   RealAudio version 5
  
RealMedia (.rm):
  Offset  Bytes   ASCII     Meaning
  ------  ------  --------  -------------------
  0x0000  4       ".RMF"   RealMedia File
  
RealAudio Lossless (.ra with RALF):
  Offset  Bytes   ASCII     Meaning
  ------  ------  --------  -------------------
  varies  —       —         MIME type: audio/x-ralf-mpeg4
```

### 3.2 RealAudio File Structure (Version 4/5)

#### Header Structure
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
       │
       ├── Version (4 bytes)
       ├── File ID (4 bytes)
       ├── Timestamp base (4 bytes)
       └── Reserved (4 bytes)
       │
  PROP properties chunk
       │
       ├── Maximum bitrate
       ├── Average bitrate
       ├── Timestamp scale
       └── Duration
       │
  MDPR media properties chunk
       │
       ├── Stream number
       ├── Stream name
       ├── MIME type
       └── Codec-specific data
       │
  CONT content description chunk
       │
       ├── Title
       ├── Author
       ├── Copyright
       └── Comment
       │
  DATA data chunk (audio/video frames)
       │
       └── Multiple packets with timestamps
       │
  INDX index chunk (optional)
       │
       └── Seek points
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
  
Inverse MDCT:
  x[n] = (1/N) Σ(k=0 to N-1) X[k] · cos(π/N · (k + 1/2) · (n + N/2))
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
  
Typical subband count:
  - Low bitrate: 30-40 subbands
  - High bitrate: 50+ subbands
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
  - Similar approach to Shorten/LA
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
| Max sample rate | 44.1 kHz (COOK) | [NEEDS VERIFICATION] |
| Float support | No | Not supported |
| High-res audio | No | Not supported |

---

## 13. KNOWN ISSUES, BUGS & EDGE CASES

### 13.1 FFmpeg Decoder Issues
| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| No RealAudio encoder | All versions | Use RealNetworks encoder |
| Limited codec support | All versions | Some variants may not decode |
| Container issues | Some versions | Use demuxer with most support |

### 13.2 Interoperability Issues
- **Format variations:** Many RealAudio variants exist
- **Codec parameters:** Extradata format varies
- **Container mixing:** Some files use non-standard containers

### 13.3 Edge Cases
- **Corrupt frames:** Replace with silence
- **Missing extradata:** Decode may fail
- **Version mismatches:** Some old files may not decode

---

## 14. CONVERSION GUIDE (DBpoweramp Context)

### 14.1 Converting FROM RealAudio

| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → FLAC | `ffmpeg -i in.rm -c:a flac -compression_level 8 out.flac` | None (streaming) | Lossless if RALF |
| → WAV | `ffmpeg -i in.rm -c:a pcm_s16le out.wav` | None | Lossless if RALF |
| → MP3 | `ffmpeg -i in.rm -c:a libmp3lame -q:a 0 out.mp3` | None | Generation loss |
| → AAC | `ffmpeg -i in.rm -c:a aac -b:a 256k out.m4a` | None | Generation loss |
| → Opus | `ffmpeg -i in.rm -c:a libopus -b:a 128k out.opus` | None | Generation loss |

### 14.2 Converting TO RealAudio
FFmpeg does **NOT** support encoding RealAudio. RealAudio encoding requires RealNetworks' proprietary encoders which are no longer available.

### 14.3 Lossless Round-Trip Verification (RALF)
```bash
# Decode RALF
ffmpeg -i input.rm -c:a pcm_s16le decoded.wav

# Compare checksums (if source was lossless)
md5sum original.wav decoded.wav
```

---

## 15. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Notes |
|---------|----------|---------|-------|
| FFmpeg libavcodec | C | LGPL 2.1+ | COOK decoder | https://ffmpeg.org |
| RealNetworks SDK | — | Proprietary | Historical only |
| Rockbox | C | Various | COOK decoder |

---

## 16. HISTORICAL SIGNIFICANCE

RealAudio was revolutionary for its time:

1. **First streaming audio** — enabled real-time audio on the Internet
2. **Adaptive streaming** — pioneered quality adaptation
3. **Internet radio** — thousands of stations used RealAudio
4. **Online audio** — early online music delivery
5. **Codec innovation** — COOK was innovative for its time

The format's decline came from:
- MP3's dominance for downloaded audio
- Shoutcast/Icecast for streaming (using MP3)
- Proprietary lock-in concerns
- Competition from standards-based approaches

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

## 18. CODEC COMPARISON

| Codec | Type | Year | Max Bitrate | Notes |
|-------|------|------|-------------|-------|
| RealAudio 1.0 | Various | 1995 | 14.4 kbps | Very low quality |
| COOK | MDCT | 1998 | 128 kbps | Low-latency streaming |
| RALF | Lossless | 2000 | 700 kbps | Lossless alternative |
| RAAC | AAC-based | 2001 | 256 kbps | Better quality |
| MP3 | MDCT | 1987 | Unlimited | Universal standard |
| AAC | MDCT | 1997 | Unlimited | Modern standard |

---

## 19. TECHNICAL DEEP DIVE

### 19.1 COOK Bit Allocation
COOK uses sophisticated bit allocation:
```
1. Compute perceptual masking thresholds
2. Allocate bits to subbands based on:
   - Masking threshold
   - Available bitrate
   - Perceptual importance
3. Quantize coefficients
4. Code quantized values
```

### 19.2 Subband Processing Details
```
COOK subband structure:
  Low frequencies:
    - More subbands (better frequency resolution)
    - More bits (better quality)
    
  High frequencies:
    - Fewer subbands
    - Fewer bits
    - Apply perceptual weighting
```

### 19.3 Frame Structure
```
Each COOK frame:
  - 1024 input samples
  - 512 MDCT coefficients
  - Variable bits per subband
  - Per-frame side information
```

---

## 20. ERROR HANDLING

### 20.1 Error Detection
```
Frame corruption detection:
  - Huffman decode errors
  - Out-of-range values
  - Checksum failures
```

### 20.2 Error Concealment
```
Concealment strategies:
  - Muting (replace with silence)
  - Frame repetition
  - Interpolation
```

---

## 21. STREAMING PROTOCOLS

### 21.1 RTSP/RTP
RealAudio used RTSP for control:
```
RTSP: Real Time Streaming Protocol
  - Session setup
  - Play/Pause control
  - Teardown
  
RTP: Real-time Transport Protocol
  - Audio packet delivery
  - Timestamp synchronization
```

### 21.2 Adaptive Streaming
RealAudio pioneered adaptive streaming:
```
Bandwidth detection:
  - Measure download speed
  - Adjust quality level
  
Quality switching:
  - Seamless transition between bitrates
  - Buffer management
```

---

## 22. LEGACY CONTENT

### 22.1 Archival Considerations
Many RealAudio files exist in archives:
1. **Internet radio recordings** from 1998-2004
2. **News broadcasts** (NPR, BBC archives)
3. **Music services** (early online music stores)
4. **Live concert recordings** from that era

### 22.2 Migration Needs
```
Migration to modern formats:
  1. Identify RealAudio files (.ra, .rm)
  2. Extract audio with FFmpeg
  3. Convert to FLAC for lossless
  4. Convert to AAC/MP3 for lossy
  5. Preserve metadata where possible
```

### 22.3 Metadata Preservation
RealAudio metadata is limited:
```
Container metadata:
  - Title
  - Author
  - Copyright
  - Comment
  
Limitations:
  - No standard tagging format
  - Limited field support
  - Often lost in conversion
```

---

## 23. ROCKBOX SUPPORT

### 23.1 Rockbox Implementation
Rockbox includes a COOK decoder:
```
Supported formats:
  - COOK (mono/stereo)
  - Various bitrates
  
Features:
  - Gapless playback
  - ReplayGain support
```

### 23.2 Limitations
```
Not supported:
  - Multichannel COOK
  - Some codec variants
  - DRM-protected content
```

---

## 24. FFmpeg COOK DECODER

### 24.1 Implementation Details
FFmpeg's COOK decoder:
```
Based on:    Reverse engineering
Features:    Basic COOK support
Limitations: Some codec variants unsupported
```

### 24.2 Extradata Handling
```
COOK requires extradata from container:
  - 8 bytes for mono
  - 16 bytes for stereo
  - Contains codec initialization data
```

### 24.3 Known Limitations
```
FFmpeg COOK decoder issues:
  - Some bitrates unsupported
  - Joint stereo handling varies
  - Multichannel limited
```

---

## 25. PRESERVATION RECOMMENDATIONS

### 25.1 Long-term Preservation
For archival purposes:
```
1. Convert to modern format (FLAC/AAC)
2. Preserve original if possible
3. Document source quality
4. Store metadata separately
```

### 25.2 Quality Considerations
```
For lossy sources:
  - Don't re-encode multiple times
  - Convert directly to target format
  - Document source characteristics
  
For RALF sources:
  - Extract to WAV first
  - Verify bit-perfect
  - Then compress to FLAC
```

---

## 26. COMPARISON WITH MODERN STREAMING

### 26.1 RealAudio vs HLS/DASH
| Feature | RealAudio | HLS/DASH |
|---------|-----------|----------|
| Adaptive streaming | Yes | Yes |
| Protocol | RTSP/RTP | HTTP |
| Codec | Proprietary | Standard (H.264/AAC) |
| Browser support | No (requires plugin) | Native |
| Mobile support | Limited | Universal |

### 26.2 Why RealAudio Was Superseded
1. **Proprietary codecs** — not supported in browsers
2. **Plugin requirement** — RealPlayer needed
3. **HTTP-based streaming** — simpler infrastructure
4. **Standards adoption** — HLS/DASH won

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
