# TTA (True Audio) — Deep Technical Reference

> **Category:** Lossless
> **File Extensions:** `.tta`
> **MIME Types:** `audio/x-tta`, `audio/tta`
> **Standardization Body:** Tau Software / Open-source community
> **Primary Specification:** https://tausoft.org/en/true-audio-codec-tta/
> **Patent Status:** Patent-free
> **License:** GPL / LGPL v3
> **Current Version:** 3.4 (encoder), TTA1 format
> **Active Development:** No — discontinued ~2012, last release 2007

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Alexander Leschinsky, Tau Software
- **Year Created:** 2007
- **Original Purpose:** Realtime lossless audio compression with emphasis on fast encoding and decoding speeds
- **Problem with Predecessors:** Existing lossless codecs at the time (FLAC 0.x, Monkey's Audio, etc.) were either too slow or had proprietary licensing. TTA was designed as a free, open alternative that could achieve transparent compression in real-time.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| 1.0 | 2000 | Initial release with basic lossless compression |
| 2.0 | 2003 | Improved predictor algorithms, better compression |
| 3.0 | 2005 | Added multichannel support, higher bit depths |
| 3.4 | 2007 | Final release with encryption support, bug fixes |

### 1.3 Current Adoption
- **Primary use cases today:** Archival of live concert recordings, niche audio archiving communities
- **Platforms with native support:** Windows (via DirectShow filters), Linux (via FFmpeg/libavcodec), macOS (via plugins)
- **Major services using this format:** None (niche/deprecated format)
- **Hardware support:** Some older DVD players and set-top boxes with TTA support
- **Status:** Legacy — superseded by FLAC and other open standards

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Waveform / Predictive
- **Core algorithm:** Adaptive Linear Prediction (ALP) with integer-only arithmetic
- **Loss mechanism:** Lossless — uses predictive coding with entropy coding of residuals
- **Frame-based vs sample-based:** Frame-based; each frame is independently decodable
- **Fixed vs variable frame size:** Fixed frame size (64KB per frame) for simplicity and seekability

### 2.2 High-Level Encoding Flow (Block Diagram)
```
Input PCM Samples
      │
      ▼
[Pre-processing: DC removal, endianness handling]
      │
      ▼
[Frame Splitting: Fixed 64KB frames]
      │
      ▼
[Adaptive Linear Prediction: FAST/NORMAL/HIGH/EXTRA modes]
      │
      ▼
[Residual Calculation: Actual - Predicted]
      │
      ▼
[Entropy Coding: Rice-Golomb coding of residuals]
      │
      ▼
[Bitstream Packing: Frame header + CRC32 + compressed data]
      │
      ▼
Output TTA Stream
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | ~64KB frame | ~1.4 seconds at 44.1kHz stereo |
| Frame size | 64KB (fixed) | 65,536 bytes per frame |
| Max channels | 16 (FFmpeg), 6 (reference encoder) | Format allows up to 65536 |
| Max bit depth | 24-bit | 8, 16, and 24-bit supported |
| Max sample rate | 4 GHz (format), 2 GHz (encoder) | Integer rates only |
| Bitrate range | N/A | Lossless — bitrate varies with content |
| Complexity | O(n) linear | Very fast encoding/decoding |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       54 54 41 31     TTA1      File magic / signature
0x0004  2       XX XX            ..        Reserved (header length indicator)
```

### 3.2 File-Level Header Layout
Full byte-map of the TTA1 file header:
```
Offset   Size    Field Name              Type        Valid Range   Description
-------  ------  ----------------------  ----------  -----------   ---------------------------
0x0000   4B      Magic Bytes           uint8[4]    "TTA1"       File signature
0x0004   4B      Audio Format          uint32 LE   1            1 = PCM format
0x0008   4B      Number of Channels    uint32 LE   1–65536     Channel count
0x000C   4B      Bits Per Sample       uint32 LE   8,16,24      Bit depth
0x0010   4B      Sample Rate          uint32 LE   any int      Sample rate in Hz
0x0014   8B      Data Length          uint64 LE   0–2^64       Total audio samples (all channels)
0x001C   4B      Header CRC32          uint32 LE   valid CRC    CRC32 of header
```

### 3.3 Frame / Block Header Layout
Each TTA frame has the following structure:
```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------
0x0000   4B      Frame CRC32            uint32 LE   CRC32 checksum of frame data
0x0004   N B     Frame Data             uint8[]     Compressed audio data
```

#### Frame Data Internal Structure
```
Bit Offset   Bit Width   Field Name              Type      Description
----------   ---------   --------------------    ------    ---------------------------
0            16         Sync/Length            uint16    Frame data length
16           variable   Compressed PCM          bits      Rice-coded prediction residuals
```

### 3.4 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Unsigned integer | Yes | Limited use — typically for voice |
| 16-bit | Signed integer | Yes | Standard CD audio |
| 20-bit | Signed integer | Yes | Professional audio |
| 24-bit | Signed integer | Yes | High-resolution standard |
| 32-bit | Signed integer | No | Not supported |
| 32-bit | IEEE float | No | Not supported |
| 64-bit | IEEE float (double) | No | Not supported |

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
- **DC offset removal:** Subtraction of mean value from samples
- **Pre-emphasis filter:** Optional first-order high-shelf filter
- **Windowing function:** None for linear prediction (uses entire frame)
- **Level normalization:** None required — uses full integer range
- **Stereo decorrelation pre-step:** Optional mid-side (M/S) transformation for stereo

### 4.2 Analysis / Transform Stage

#### Transform Type: None (Pure Linear Prediction)
TTA uses adaptive linear prediction exclusively — no MDCT, FFT, or frequency-domain transforms.

```
Parameters:
  Prediction modes:  FAST (1), NORMAL (2), HIGH (3), EXTRA (4)
  Predictor order:   Adaptive, up to 32 coefficients
  Window:            Full frame (65,536 samples)
  Algorithm:         Levinson-Durbin recursion for LPC coefficients
  Coefficient precision: 32-bit integer arithmetic
```

**LPC Analysis:**
```
LPC order:    8–32 (adaptive per frame)
Window:       Rectangular (entire frame)
Algorithm:    Modified Levinson-Durbin
Coefficient precision: 32-bit fixed point
```

### 4.3 Psychoacoustic Model (Not Applicable)
TTA is a **lossless** codec. No psychoacoustic modeling is performed — all samples are preserved bit-exactly.

### 4.4 Quantization
TTA uses **no quantization** — it is a purely lossless codec. Prediction residuals are entropy-coded without any lossy approximation.

### 4.5 Stereo Encoding Modes
| Mode | Description | Condition for Selection | Notes |
|------|-------------|------------------------|-------|
| Independent | L and R encoded separately | Default | Used in reference encoder |
| M/S (Mid-Side) | M=(L+R)/2, S=(L-R)/2 | Optional per frame | Can improve compression for certain content |

### 4.6 Entropy / Lossless Coding Stage
```
Method: Rice-Golomb coding (simplified arithmetic coding)

Rice parameter selection:
  - Parameter k: Computed from variance of residuals in frame
  - Quotient: Unary coded
  - Remainder: Binary coded with k bits
  
Escape codes: Not needed — Rice coding handles all residual values
```

### 4.7 Encoder Settings / Quality Modes

#### For Lossless Codecs
| Compression Level | Encoding Speed | Decode Speed | Compression Ratio | Notes |
|---|---|---|---|---|
| FAST (1) | Fastest | Fastest | ~40-60% | Minimal prediction |
| NORMAL (2) | Fast | Fast | ~35-55% | Balanced (default) |
| HIGH (3) | Medium | Medium | ~30-50% | Better prediction |
| EXTRA (4) | Slow | Medium | ~25-45% | Best compression |

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. Scan bitstream for magic bytes: "TTA1" (0x54 0x54 0x41 0x31)
2. Read and validate header fields
3. Verify header CRC32
4. Frame synchronization via CRC32 checks at end of each frame
5. If CRC fails: skip to next frame (frame boundaries known)
```

#### Seeking
- **Seek table:** Stored at end of file — list of frame offsets and sample positions
- **Seek precision:** Sample-accurate via seek table
- **Random access:** Each frame independently decodable

### 5.2 Core Decode Pipeline
```
1. Read file header
   ├── Verify magic bytes "TTA1"
   ├── Parse channel count, bit depth, sample rate
   └── Validate header CRC32

2. Read frame header (4 bytes before compressed data)
   ├── Read frame CRC32
   └── Read compressed frame data

3. Entropy decode residuals
   └── Rice-Golomb decoding with per-frame parameter k

4. Inverse prediction
   └── Apply LPC coefficients to reconstruct samples

5. Post-process
   ├── Stereo decorrelation reversal (if M/S was used)
   └── Output PCM samples
```

### 5.3 Error Concealment
- **Corrupt frame detection:** CRC32 checksum per frame
- **Concealment method:** Replace corrupt frame with silence (mute)
- **Maximum consecutive errors:** Decoder stops after too many corrupt frames

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** None — TTA is a raw stream format
- **Overhead:** ~0.1% (header + frame CRCs only)
- **Seeking in native container:** Yes — via seek table at file end
- **Multiple streams in native container:** No

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| TTA (native) | Yes | Yes | ID3v1/v2, APEv2 | No container overhead |
| Matroska/MKA | Yes | Yes | Full | Via Matroska audio codec |
| RIFF WAV | No | N/A | N/A | Not designed for WAV |
| AIFF | No | N/A | N/A | Not designed for AIFF |
| OGG | No | N/A | N/A | Not designed for OGG |
| MP4/M4A | No | N/A | N/A | Not designed for MP4 |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** ID3v1, ID3v2, and APEv2 (stored after audio data)
- **Tag block location:** End of file (after all audio frames)
- **Tag block identifier:** "TAG" (ID3v1) or "APETAGEX" (APEv2)

### 7.2 Standard Tag Fields — Complete Reference
| Tag Field | Internal Key (TTA) | Max Length | Character Encoding | Multi-value | Notes |
|-----------|---------------------------|------------|-------------------|-------------|-------|
| Title | TITLE | 255 bytes | UTF-8/UTF-16 | No | |
| Artist | ARTIST | 255 bytes | UTF-8/UTF-16 | No | |
| Album | ALBUM | 255 bytes | UTF-8/UTF-16 | No | |
| Album Artist | ALBUMARTIST | 255 bytes | UTF-8/UTF-16 | No | |
| Genre | GENRE | 255 bytes | UTF-8/UTF-16 | No | ID3v1 genre numbers also accepted |
| Year / Date | YEAR or TDRC | 4 bytes | ASCII | No | |
| Track Number | TRACK or TRACKNUMBER | 10 bytes | ASCII | No | Format: "N" or "N/Total" |
| Disc Number | DISCNUMBER | 10 bytes | ASCII | No | Format: "N" or "N/Total" |
| Comment | COMMENT | 1000 bytes | UTF-8/UTF-16 | No | |
| ReplayGain Track Gain | REPLAYGAIN_TRACK_GAIN | 20 bytes | ASCII | No | Format: "-6.20 dB" |
| ReplayGain Track Peak | REPLAYGAIN_TRACK_PEAK | 20 bytes | ASCII | No | Format: "0.998459" |

### 7.3 Cover Art Storage
TTA does not natively support embedded cover art. Cover art must be stored in the tag metadata using APEv2's "COVER ART" field or ID3v2's APIC frame.

### 7.4 Metadata Compatibility Matrix
| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| ID3v1 | ✓ | ✓ | ✓ | Lowest |
| ID3v2.3 | ✓ | ✓ | ✓ | |
| ID3v2.4 | ✓ | ✓ | ✓ | |
| APEv2 | ✓ | ✓ | ✓ | Highest |

**Conflict resolution:** APEv2 takes precedence over ID3v2, which takes precedence over ID3v1.

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   tta              # used with -c:a
AV_CODEC_ID:        AV_CODEC_ID_TTA  # C constant in libavcodec/codec_id.h
Format Name (CLI):  tta              # used with -f
Encoder(s):         NONE — FFmpeg does NOT encode TTA
Decoder(s):         tta (native)      # ffmpeg -decoders | grep -i tta
Muxer(s):           tta              # ffmpeg -muxers | grep -i tta
Demuxer(s):         tta              # ffmpeg -demuxers | grep -i tta
```

### 8.2 FFmpeg Encoding — NOT SUPPORTED
FFmpeg does **NOT** support encoding TTA format. TTA encoding requires the official TTA encoder from tausoftware.org or third-party encoders.

### 8.3 FFmpeg Decoding — Full CLI Reference

```bash
# Decode TTA to WAV
ffmpeg -i input.tta \
  -c:a pcm_s16le \
  output.wav

# Decode TTA to FLAC
ffmpeg -i input.tta \
  -c:a flac -compression_level 8 \
  output.flac

# Extract audio stream without conversion
ffmpeg -i input.tta \
  -c:a copy \
  output.tta

# Probe TTA file information
ffprobe -v quiet -print_format json -show_streams -show_format input.tta

# Extract metadata
ffprobe -v quiet -print_format json -show_format input.tta | jq .format.tags
```

### 8.4 FFmpeg Decoding — libavcodec C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>

// ─── Complete decode program skeleton ────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.tta", NULL, NULL);
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
ffprobe -v quiet -print_format json -show_format input.tta | jq .format.tags

# Strip all metadata
ffmpeg -i input.tta -c:a copy -map_metadata -1 output.tta

# Note: FFmpeg cannot write ID3v2/APEv2 tags to TTA files natively
# Use external tools like: mid3v2, id3v2, apetag
```

#### FFmpeg Internal Metadata Key Mapping
| Standard Field | FFmpeg Key | TTA Native Key | Notes |
|----------------|------------|--------------------------|-------|
| Title | title | TIT2 (ID3v2) | |
| Artist | artist | TPE1 (ID3v2) | |
| Album | album | TALB (ID3v2) | |
| Track Number | track | TRCK (ID3v2) | |
| Genre | genre | TCON (ID3v2) | |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
TTA stores a seek table at the end of the file before any metadata:

```
TTA Seek Table:
  Location:     End of file, before metadata
  Magic:        None (identified by position)
  Entry size:   12 bytes
  Entry format:
    [0x00–0x07]  frame_offset (uint64)      — Byte offset from file start
    [0x08–0x0B]  frame_samples (uint32)     — Sample number at frame start
  Max entries:  Limited by file size
```

### 9.2 Gapless Playback Data
```
Encoder delay:   0 samples (TTA adds no padding)
Padding:         0 samples
Storage location: Not stored — total samples in header
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | Frame-based, each frame independent |
| Algorithmic encoder delay | ~64KB frame | ~1.4 seconds at 44.1kHz stereo |
| Live encoding feasible | Yes | Designed for real-time operation |
| HTTP progressive download | Yes | Supported |
| HTTP Live Streaming (HLS) | No | Not a standard HLS segment format |
| DASH streaming | No | Not commonly used |
| WebRTC / RTP transport | No | Not designed for real-time transport |
| Minimum decode buffer | 1 frame | 64KB |

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
| 16 | — | Up to 16 channels | FFmpeg limit |

**Note:** TTA format has no native channel assignment metadata. Channel meaning must be inferred from external context or tags.

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
| Max sample rate | 2 GHz | Encoder limit, format supports higher |
| Float support | None | Integer arithmetic only |
| DSD support | No | Not supported |
| 20-bit support | Yes | Common in professional audio |
| 24-bit support | Yes | High-res standard |

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

| Platform / API | Encode | Decode | FFmpeg Flag | Notes |
|----------------|--------|--------|-------------|-------|
| x86 SIMD | No | Yes | — | SSE/SSE2 optimization in reference |
| ARM NEON | No | Limited | — | Not officially optimized |
| CUDA/NVENC | No | No | — | Not applicable |
| OpenCL | No | No | — | Not applicable |

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Encoder-Specific Issues
| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| No TTA encoder | All versions | Use official TTA encoder |
| Limited multichannel support | All versions | FFmpeg limits to 16 channels |

### 14.2 Interoperability Issues
- **FFmpeg TTA → Official decoder:** Fully compatible
- **Official TTA → FFmpeg decode:** Fully compatible
- **Multichannel interpretation:** No standard channel assignment in format
- **Files with mixed tag systems:** Last valid tag system wins

### 14.3 Edge Cases to Handle in Converter
- **Empty file / 0-sample stream:** Invalid TTA file
- **File < 1 frame of audio:** Rare but valid
- **All-silence audio:** Encodes very efficiently (~5% size)
- **DC offset (non-zero mean):** Encoder handles, no special processing
- **Full-scale sine (0 dB):** No clipping risk in lossless
- **Corrupt frame:** CRC failure, frame replaced with silence
- **Truncated file:** Decoder stops at last valid frame
- **Sample rate not supported:** Must be integer value
- **Channel count > 16:** FFmpeg rejects, reference encoder max 6

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM TTA

| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → FLAC | `ffmpeg -i in.tta -c:a flac -compression_level 8 out.flac` | All tags | Lossless |
| → ALAC | `ffmpeg -i in.tta -c:a alac out.m4a` | Via MP4 atoms | Lossless |
| → WAV | `ffmpeg -i in.tta -c:a pcm_s16le out.wav` | RIFF INFO | Lossless |
| → MP3 | `ffmpeg -i in.tta -c:a libmp3lame -q:a 0 out.mp3` | Via ID3v2 | Generation loss |
| → AAC | `ffmpeg -i in.tta -c:a aac -b:a 256k out.m4a` | Via MP4 | Generation loss |
| → Opus | `ffmpeg -i in.tta -c:a libopus -b:a 128k out.opus` | Via Ogg | Generation loss |

### 15.2 Converting TO TTA
FFmpeg does **NOT** support encoding TTA. Use the official TTA encoder:
- Windows: Tau Producer GUI
- Cross-platform: ttaenclib-based tools
- Note: No FFmpeg-based TTA encoding available

### 15.3 Lossless Round-Trip Verification
```bash
# Decode
ffmpeg -i original.tta -c:a pcm_s16le decoded.wav

# Compare checksums
ffmpeg -i original.tta -map 0:a -f framemd5 original.md5
ffmpeg -i decoded.wav -map 0:a -f framemd5 decoded.md5
diff original.md5 decoded.md5   # Empty diff = bit-perfect
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| TTA Reference | C/C++ | GPL/LGPL | Reference | Reference | https://tausoft.org |
| TTA-lib | C | LGPL | Reference | Reference | SourceForge |
| FFmpeg libavcodec | C | LGPL 2.1+ | N/A | 8/10 | https://ffmpeg.org |
| foobar2000 TTA plugin | C++ | Proprietary | Reference | Reference | foobar2000 |

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **TTA Format Spec:** https://tausoft.org/en/true-audio-codec-tta/
- **Source Code:** https://sourceforge.net/projects/tta/

### Technical Resources
- FFmpeg codec support: `ffmpeg -decoders | grep tta`
- Hydrogenaudio TTA page: https://wiki.hydrogenaudio.org/index.php?title=TTA
- Multimedia Wiki: https://wiki.multimedia.cx/index.php/TTA

### Academic Papers
- None published (closed development)

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] Identify FFmpeg built with TTA support (verify `ffmpeg -decoders | grep tta`)
- [ ] Note: FFmpeg does NOT encode TTA — must use external encoder
- [ ] Handle platform-specific TTA encoder availability

### Decoding Pipeline
- [ ] Implement sync word search for "TTA1" magic
- [ ] Parse file header (channels, bit depth, sample rate, total samples)
- [ ] Validate header CRC32
- [ ] Read frame CRC32 at start of each frame
- [ ] Implement Rice-Golomb entropy decoding
- [ ] Apply inverse linear prediction
- [ ] Handle stereo M/S decorrelation reversal
- [ ] Flush decoder at EOF

### Metadata
- [ ] Read ID3v1 tags from end of file
- [ ] Read ID3v2 tags from end of file (before ID3v1)
- [ ] Read APEv2 tags from end of file
- [ ] Preserve tag priority (APEv2 > ID3v2 > ID3v1)
- [ ] Cover art extraction from tags

### Quality & Verification
- [ ] Implement frame-level CRC verification
- [ ] Track corrupted frames for error reporting
- [ ] Test with: silence, full-scale, multichannel, high-resolution files

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
