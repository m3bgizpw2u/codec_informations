# Nellymoser ASAO Codec — Deep Technical Reference

> **Category:** Lossy
> **File Extensions:** `.flv`, `.f4v`, `.swf` (container), `.nelly` (raw)
> **MIME Types:** `audio/nellymoser`, `video/x-nellymoser`
> **Standardization Body:** None — proprietary codec by Nellymoser Inc.
> **Primary Specification:** Not publicly available; reverse-engineered
> **Patent Status:** Proprietary — Nellymoser Inc. holds all patents
> **License:** Proprietary — licensed to Adobe for Flash integration
> **Current Version:** Unknown — development ceased
> **Active Development:** No — Flash deprecated in 2020

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Nellymoser Inc. (founded by Robert Nellymoser)
- **Year Created:** 2002 (incorporated into Flash Player 6)
- **Original Purpose:** Create a low-bitrate speech codec optimized for real-time encoding of voice audio, suitable for Flash Player microphone recording and streaming
- **Problem with Predecessors:** Existing codecs (Speex wasn't ready, GSM was too low quality) didn't meet Adobe's requirements for voice recording in Flash applications with minimal latency

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| ASAO v1 | 2002 | Initial codec, integrated into Flash MX / Flash Player 6 |
| ASAO v2 | 2005 | Minor improvements [NEEDS VERIFICATION] |
| Discontinued | 2020 | Flash deprecated; replaced by AAC, Opus in WebM |

### 1.3 Current Adoption
- **Primary use cases today:** Legacy Flash video files (FLV), historical recordings, embedded in SWF files
- **Platforms with native support:** Flash Player (legacy), some media players via FFmpeg
- **Major services using this format:** YouTube (early 2005–2008), video messaging services, Flash-based applications
- **Hardware support:** None — software only
- **Status:** Deprecated — Flash officially deprecated December 2020; format superseded by AAC, Opus

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Speech/speech-focused audio coding
- **Core algorithm:** Custom DCT-based transform with psychoacoustic modeling
- **Loss mechanism:** Perceptual coding — optimized for voice
- **Frame-based vs sample-based:** Frame-based — 256 samples per block
- **Fixed vs variable frame size:** Fixed — 256 samples per block

### 2.2 High-Level Encoding Flow (Block Diagram)
```
Input PCM Samples (mono, 8/11/22 kHz)
      │
      ▼
[Pre-processing: DC removal, windowing]
      │
      ▼
[Filterbank: MDCT-like transform]
      │
      ▼
[Psychoacoustic Model: Masking threshold]
      │
      ▼
[Bit Allocation: Distribute bits per band]
      │
      ▼
[Quantization: Non-uniform quantization]
      │
      ▼
[Entropy Coding: Huffman or table-based packing]
      │
      ▼
[Bitstream Packing: Frame assembly]
      │
      ▼
Output Nellymoser Frame
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Sample rates | 8, 11, 16, 22, 44 kHz | 8/16 kHz special cases in FLV |
| Block size | 256 samples | Fixed per block |
| Channels | 1 (mono) | No stereo mode |
| Bitrate | Variable | ~5–32 kbps depending on rate |
| Algorithmic delay | ~5.8 ms | 256 samples at 44.1 kHz |
| Complexity | Low | Optimized for real-time encoding |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
```
Nellymoser has no standalone file signature.
Codec identification is by codec ID in the container format:

In FLV AudioTagHeader:
  SoundFormat = 4: Nellymoser 16 kHz mono
  SoundFormat = 5: Nellymoser 8 kHz mono
  SoundFormat = 6: Nellymoser (variable rate)
```

### 3.2 Frame Header Layout (FLV Context)
```
FLV Audio Tag Header (1 byte):
  Bits 0-3:   SoundFormat (4 bits) — 4, 5, or 6 for Nellymoser
  Bits 4-5:   SoundRate (2 bits)
                0 = 5.5 kHz (for some codecs)
                1 = 11 kHz
                2 = 22 kHz
                3 = 44 kHz
                Note: For Nellymoser 8kHz/16kHz, SoundRate is ignored
  Bit 6:      SoundSize (1 bit) — 0=8-bit, 1=16-bit (Nellymoser always 16-bit)
  Bit 7:      SoundType (1 bit) — 0=mono, 1=stereo (Nellymoser always mono)

Nellymoser Audio Data (variable length):
  Frame starts after Audio Tag Header
  Frame contains: block_size (2 bytes) + compressed_block_data
```

### 3.3 Nellymoser Frame Structure
```
Per-Block Data Layout:
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------
0x0000   2B      Block Size             uint16 LE   Compressed block size in bytes
0x0002   N       Block Data            byte[]      Compressed audio data
```

### 3.4 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Unsigned integer | No | Flash uses 16-bit internally |
| 16-bit | Signed integer | Yes | Flash Player uses 16-bit |
| 24-bit | Signed integer | No | Not supported |
| 32-bit | Signed integer | No | Not supported |
| 32-bit | IEEE float | No | Not supported |

#### Supported Sample Rates
| Rate (Hz) | FLV SoundRate | Supported | Notes |
|-----------|-------------|-----------|-------|
| 8000 | Special (SoundFormat 5) | Yes | Voice quality |
| 11025 | — | No | Not supported |
| 16000 | Special (SoundFormat 4) | Yes | Flash audio recording |
| 22050 | 2 | Yes | Common in FLV |
| 44100 | 3 | Yes | Highest quality |
| 48000 | 3 | Yes | Nearest to 44 kHz |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Pre-Processing Stage
- **DC offset removal:** Implicit in transform
- **Windowing:** Hanning or similar window before transform
- **No pre-emphasis:** Not used

### 4.2 Analysis / Transform Stage

#### Transform Type: MDCT-like Filterbank
```
Nellymoser uses a modified DCT-based transform.
The exact coefficients are proprietary but the transform is similar to MDCT.

Parameters:
  Block size:    256 samples
  Transform:    DCT-like (not standard MDCT)
  Overlap:      50% overlap between blocks
  Window:       Sinusoidal window function
```

### 4.3 Psychoacoustic Model (Proprietary)
```
Nellymoser uses a custom psychoacoustic model optimized for speech.
Key properties:
  - Designed for voice frequencies (300 Hz – 3.4 kHz)
  - Aggressive bit allocation for formants
  - Less efficient for music
  - Simple masking threshold calculation
```

### 4.4 Quantization
```
Type: Non-uniform scalar quantization
- Larger quantization step at high frequencies
- Finer quantization for speech formants
- Bit allocation per frequency band
```

### 4.5 Stereo Encoding Modes
```
Nellymoser does NOT support stereo.
All Nellymoser audio is mono.
For stereo content, two separate mono streams were used.
```

### 4.6 Entropy Coding
```
Method: Table-based packing (proprietary)
- Short Huffman codes for common values
- Fixed-length codes for less common values
- Optimized for speech characteristics
```

### 4.7 Encoder Settings

#### Nellymoser Quality Modes
| Quality Setting | Sample Rate | Bitrate | Intended Use |
|----------------|-------------|---------|--------------|
| Lowest | 8 kHz | ~5–8 kbps | Voice mail |
| Low | 16 kHz | ~10–16 kbps | Flash microphone recording |
| Medium | 22 kHz | ~16–22 kbps | General voice |
| High | 44 kHz | ~24–32 kbps | Highest quality |

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
Nellymoser is stored within FLV container:

1. Read FLV Audio Tag Header
2. Verify SoundFormat = 4, 5, or 6
3. Read block_size (2 bytes)
4. Read block_data of block_size bytes
5. Decode block to 256 PCM samples
6. Repeat for next tag
```

#### Seeking
```
Seeking in FLV requires:
  1. FLV index (keyframes) for video position
  2. Nellymoser audio is continuous — seek to nearest audio tag
  3. Decode from seek point
```

### 5.2 Core Decode Pipeline
```
1. Read FLV Audio Tag Header
     ├── Verify SoundFormat (4, 5, or 6)
     ├── Determine sample rate from format
     └── Skip SoundRate field for 8/16 kHz formats

2. Read block header
     ├── Read block_size (uint16 LE)
     └── Read block_data

3. Decompress block
     ├── Decode entropy-coded data
     ├── Dequantize frequency coefficients
     ├── Apply inverse transform
     └── Apply synthesis window

4. Overlap-add consecutive blocks
     └── 50% overlap with previous block

5. Post-processing
     ├── DC removal
     └── Format as 16-bit PCM

6. Output 256 samples per block
```

### 5.3 Error Concealment
- **Corrupt block detection:** Invalid block size, decode failure
- **Concealment method:** Muting or repeat previous block
- **Maximum consecutive errors:** Undefined — Flash Player behavior varies

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** FLV (Flash Video) or SWF (Shockwave Flash)
- **Overhead:** ~1 byte per tag header + block header
- **Seeking in native container:** By FLV index for video; audio follows
- **Multiple streams in native container:** Yes — audio + video in FLV

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| FLV | Nellymoser | Yes | Limited | Native format |
| SWF | Nellymoser | Limited | No | Audio in SoundChannel |
| F4V | No | — | — | Uses AAC instead |
| Matroska/MKA | Via transcoding | Yes | Full | Not recommended |
| MP4/M4A | No | — | — | Not supported |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** None — Nellymoser has no native metadata
- **Tag block location:** N/A
- **External storage:** FLV metadata onStart/onMeta events

### 7.2 FLV Metadata for Nellymoser Audio
| Field | Description |
|-------|-------------|
| audiocodecs | Audio codec ID (4, 5, or 6 for Nellymoser) |
| audiosamplerate | Sample rate (8000, 11025, 22050, 44100) |
| stereo | false (always mono) |
| duration | Total duration in seconds |

### 7.3 Cover Art Storage
```
Nellymoser has no cover art support.
FLV does not support embedded cover art.
```

### 7.4 Metadata Compatibility Matrix
| Tag System | Nellymoser/FLV | Priority |
|------------|----------------|----------|
| FLV onMeta | ✓ (limited) | Highest |
| ID3v2 | No | N/A |
| APEv2 | No | N/A |

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   nellymoser           # used with -c:a
AV_CODEC_ID:        AV_CODEC_ID_NELLYMOSER  # C constant
Format Name (CLI):  flv                   # for FLV container
Encoder(s):         nellymoser           # Added in FFmpeg 0.5 (2009)
Decoder(s):         nellymoser           # Added in FFmpeg 0.5 (2009)
Muxer(s):          flv                   # FLV muxer
Demuxer(s):        flv                   # FLV demuxer
```

### 8.2 FFmpeg Encoding — Full CLI Reference

```bash
# Encode to Nellymoser (FLV container)
ffmpeg -i input.wav \
  -c:a nellymoser \
  -ar 22050 \
  -ac 1 \
  -f flv \
  output.flv

# Encode 16 kHz Nellymoser (Flash microphone recording)
ffmpeg -i input.wav \
  -c:a nellymoser \
  -ar 16000 \
  -ac 1 \
  -f flv \
  output.flv

# Encode 44.1 kHz Nellymoser (highest quality)
ffmpeg -i input.wav \
  -c:a nellymoser \
  -ar 44100 \
  -ac 1 \
  -f flv \
  output.flv
```

#### Complete FFmpeg Option Table
| Option | Type | Default | Range / Valid Values | Effect on Output |
|--------|------|---------|---------------------|-----------------|
| `-c:a nellymoser` | flag | — | — | Select Nellymoser encoder |
| `-ar` | int | 44100 | 8000, 11025, 16000, 22050, 44100 | Audio sample rate |
| `-ac` | int | 1 | 1 | Channel count (mono only) |
| `-f flv` | flag | — | — | Force FLV container |

### 8.3 FFmpeg Encoding — libavcodec C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/opt.h>

// ─── 1. Find Nellymoser encoder ────────────────────────────────────────────
const AVCodec *codec = avcodec_find_encoder(AV_CODEC_ID_NELLYMOSER);
if (!codec) { fprintf(stderr, "Nellymoser encoder not found\n"); exit(1); }

// ─── 2. Allocate codec context ─────────────────────────────────────────────
AVCodecContext *ctx = avcodec_alloc_context3(codec);
ctx->sample_rate = 22050;
ctx->channels = 1;
av_channel_layout_default(&ctx->ch_layout, 1);
ctx->sample_fmt = AV_SAMPLE_FMT_FLT; // Nellymoser requires float input

// ─── 3. Open codec ─────────────────────────────────────────────────────────
int ret = avcodec_open2(ctx, codec, NULL);
if (ret < 0) { /* handle error */ }

// ─── 4. Allocate frame and packet ──────────────────────────────────────────
AVFrame *frame = av_frame_alloc();
frame->format = ctx->sample_fmt;
frame->sample_rate = ctx->sample_rate;
av_channel_layout_copy(&frame->ch_layout, &ctx->ch_layout);
frame->nb_samples = ctx->frame_size;
av_frame_get_buffer(frame, 0);

AVPacket *pkt = av_packet_alloc();

// ─── 5. Encode loop ───────────────────────────────────────────────────────
void encode_frame(AVCodecContext *ctx, AVFrame *frame, AVPacket *pkt, FILE *outfile) {
    int ret = avcodec_send_frame(ctx, frame);
    if (ret < 0) { /* handle error */ }

    while (ret >= 0) {
        ret = avcodec_receive_packet(ctx, pkt);
        if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF) return;
        if (ret < 0) { /* fatal error */ exit(1); }
        fwrite(pkt->data, 1, pkt->size, outfile);
        av_packet_unref(pkt);
    }
}

// ─── 6. Cleanup ───────────────────────────────────────────────────────────
av_frame_free(&frame);
av_packet_free(&pkt);
avcodec_free_context(&ctx);
```

### 8.4 FFmpeg Decoding — Full CLI Reference

```bash
# Decode FLV/Nellymoser to WAV
ffmpeg -i input.flv \
  -c:a pcm_s16le \
  output.wav

# Decode and resample
ffmpeg -i input.flv \
  -ar 44100 \
  -c:a pcm_s16le \
  output.wav

# Extract Nellymoser audio
ffmpeg -i input.flv \
  -vn \
  -c:a pcm_s16le \
  output.wav

# Convert to MP3
ffmpeg -i input.flv \
  -vn \
  -c:a libmp3lame -q:a 2 \
  output.mp3

# Probe format
ffprobe -v quiet -print_format json -show_streams input.flv
```

### 8.5 FFmpeg Decoding — libavformat C API

```c
// ─── Complete decode program skeleton ────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.flv", NULL, NULL);
avformat_find_stream_info(fmt_ctx, NULL);

// Find audio stream
int audio_idx = av_find_best_stream(fmt_ctx, AVMEDIA_TYPE_AUDIO, -1, -1, NULL, 0);
AVStream *stream = fmt_ctx->streams[audio_idx];

// Find Nellymoser decoder
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
            // frm->data[0] contains PCM samples
            // frm->nb_samples = sample count
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

### 8.6 FFmpeg Metadata Handling

```bash
# Read metadata from FLV
ffprobe -v quiet -print_format json -show_format input.flv

# Note: Nellymoser has no metadata fields
# Extract audio without metadata
ffmpeg -i input.flv -c:a pcm_s16le -map_metadata -1 output.wav
```

### 8.7 Quality / Fidelity Decision Table
| Use Case | Settings | Expected Quality | Notes |
|----------|----------|-----------------|-------|
| Flash microphone recording | 16 kHz | Acceptable voice | Default for Flash mic |
| Legacy FLV video | 22 kHz | Decent voice | Common in YouTube era |
| Highest quality | 44.1 kHz | Best Nellymoser | Rarely used |
| Music / audio | Not recommended | Poor | Use AAC instead |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
```
FLV seeking uses onMetaData cuePoints or keyframe index:
  Location:    FLV header or onMetaData object
  Entry count: Number of keyframes
  Entry format:
    timestamp: Video timestamp in ms
    offset:    Byte offset from file start

Audio seeking: No native support
  Audio is continuous — seek to nearest video keyframe
  Decode audio from seek point
```

### 9.2 Gapless Playback Data
```
Nellymoser does not store gapless playback data.
Encoder delay:   Unknown — likely minimal
Padding:        None
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable | Yes | Designed for real-time |
| Algorithmic delay | ~5.8 ms | 256 samples at 44.1 kHz |
| Live encoding | Yes | Primary use case in Flash |
| HTTP progressive download | Yes | FLV supports progressive |
| RTMP streaming | Yes | Original delivery method |
| WebRTC | No | Not supported |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | Layout Name | Supported | Notes |
|----------|-------------|-----------|-------|
| 1 | Mono | Yes | Only mode |
| 2 | Stereo | No | Not supported |
| >1 | — | No | Not supported |

### 11.2 Downmix
```
Nellymoser has no stereo mode.
Stereo content requires two mono streams or transcoding.
```

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 16-bit | Flash Player limitation |
| Max sample rate | 44,100 Hz | Highest supported |
| Float support | No | Fixed-point |
| DSD support | No | Not applicable |

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

| Platform / API | Encode | Decode | FFmpeg Flag | Notes |
|----------------|--------|--------|-------------|-------|
| Flash Player | Yes | Yes | N/A | Native |
| FFmpeg | Yes | Yes | Built-in | Since 0.5 |
| Hardware acceleration | No | No | N/A | Software only |

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Encoder-Specific Issues
| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| Quality varies | All | Use higher sample rate |
| Mono only | All | Convert to stereo after decode if needed |

### 14.2 Interoperability Issues
- **Flash Player end-of-life:** Nellymoser no longer playable in modern browsers
- **Quality:** Not suitable for music — designed for speech
- **Browser support:** None — Flash disabled everywhere since 2021

### 14.3 Edge Cases to Handle in Converter
- **8 kHz Nellymoser:** Quality very low — warn users
- **16 kHz Nellymoser:** Flash microphone format
- **Missing FLV index:** Scan for audio tags sequentially
- **Corrupt blocks:** Mute and continue

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM Nellymoser/FLV
| Target | FFmpeg Command | Quality Notes |
|--------|---------------|---------------|
| → WAV | `ffmpeg -i in.flv -c:a pcm_s16le -ar 44100 out.wav` | Lossless decode |
| → FLAC | `ffmpeg -i in.flv -c:a flac -compression_level 8 out.flac` | Lossless decode |
| → MP3 | `ffmpeg -i in.flv -c:a libmp3lame -q:a 2 out.mp3` | Re-encode |
| → AAC | `ffmpeg -i in.flv -c:a aac -b:a 128k out.m4a` | Re-encode |
| → Opus | `ffmpeg -i in.flv -c:a libopus -b:a 64k out.opus` | Good for voice |

### 15.2 Converting TO Nellymoser
| Source | FFmpeg Command | Quality Notes |
|--------|---------------|---------------|
| WAV → FLV/Nellymoser | `ffmpeg -i in.wav -c:a nellymoser -ar 22050 -f flv out.flv` | Legacy format |
| WAV → FLV (16kHz) | `ffmpeg -i in.wav -c:a nellymoser -ar 16000 -f flv out.flv` | Flash mic recording |

### 15.3 Lossless Round-Trip Verification
```bash
# Decode Nellymoser to WAV
ffmpeg -i input.flv -c:a pcm_s16le -ar 44100 decoded.wav

# Compare checksums
md5sum decoded.wav

# Note: Nellymoser is lossy — no lossless round-trip
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| Nellymoser Inc. | — | Proprietary | Reference | Reference | N/A (closed) |
| FFmpeg Nellymoser | C | LGPL 2.1+ | Good | Good | ffmpeg.org |
| On2 VP6 | C | Proprietary | — | — | Historical competitor |

### Build Instructions
```bash
# Build FFmpeg with Nellymoser support
./configure --enable-decoder=nellymoser --enable-encoder=nellymoser \
  --enable-demuxer=flv --enable-muxer=flv
make -j$(nproc)
make install
```

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **FLV Specification:** https://www.adobe.com/devnet/flv.html
- **FLV File Format Spec (obsolete):** Archived Adobe documentation

### Technical Resources
- FFmpeg nellymoser decoder: `ffmpeg -decoders | grep nellymoser`
- FFmpeg nellymoser encoder: `ffmpeg -encoders | grep nellymoser`
- Multimedia Wiki: https://wiki.multimedia.cx/index.php?title=Nellymoser

### Historical Context
- Flash Player 6 released 2002 with Nellymoser support
- Flash Player 9 (2007) added AAC support
- Flash Player 10 (2008) added Speex support
- HTML5 Audio replaced Flash audio by 2015
- Flash Player officially deprecated December 2020

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] Identify FFmpeg `./configure` flags: `--enable-decoder=nellymoser --enable-encoder=nellymoser --enable-demuxer=flv --enable-muxer=flv`
- [ ] Verify `ffmpeg -decoders` output confirms Nellymoser decoder is available
- [ ] Verify `ffmpeg -encoders` output confirms Nellymoser encoder is available
- [ ] Note: FFmpeg has full Nellymoser support — no external library needed

### Encoding Pipeline
- [ ] Validate input sample rate is in supported range
- [ ] Set channel count to 1 (mono only)
- [ ] Nellymoser requires float input format (AV_SAMPLE_FMT_FLT)
- [ ] Use libswresample to convert input to float if needed
- [ ] Write to FLV container format

### Decoding Pipeline
- [ ] Read FLV Audio Tag Header to identify Nellymoser format
- [ ] Read block_size and block_data for each audio block
- [ ] Decode Nellymoser blocks to PCM
- [ ] Output as 16-bit signed integer or float

### Metadata
- [ ] Nellymoser has no native metadata
- [ ] Read FLV onMetaData for duration and codec info
- [ ] No tag fields to map

### Quality & Verification
- [ ] Nellymoser is lossy — verify quality is appropriate for speech
- [ ] Test with 8 kHz, 16 kHz, 22 kHz, and 44.1 kHz inputs
- [ ] Test seeking in FLV files with and without keyframe index

### Edge Cases
- [ ] Handle 8 kHz Nellymoser (lowest quality)
- [ ] Handle FLV files with missing keyframe index
- [ ] Handle corrupt Nellymoser blocks gracefully
- [ ] Handle mono input to stereo output (upmix or error)

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
