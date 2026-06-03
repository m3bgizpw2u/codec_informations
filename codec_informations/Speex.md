# Speex — Deep Technical Reference
> **Category:** Lossy
> **File Extensions:** `.spx`
> **MIME Types:** `audio/x-speex`, `audio/speex`
> **Standardization Body:** Xiph.org Foundation
> **Primary Specification:** RFC 5574 — RTP Payload Format for the Speex Codec; https://www.speex.org/docs/manual/speex-manual.pdf
> **Patent Status:** Patent-free — no licensing required
> **License:** BSD (Xiph.org variant), GNU General Public License (as part of GNU Project)
> **Current Version:** 1.2 beta 3 (development discontinued)
> **Active Development:** No — discontinued around 2013–2016, succeeded by Opus

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Jean-Marc Valin, Xiph.org Foundation
- **Year Created:** 2002–2003 (first release ~2002, RFC 5574 published 2009)
- **Original Purpose:** Provide a free, open-source, patent-free voice codec as an alternative to expensive proprietary speech codecs such as AMR, AMR-WB, G.729, and G.723.1. Xiph.org sought to lower the barrier of entry for voice applications by offering a royalty-free codec with features comparable to commercial alternatives.
- **Problem with Predecessors:** Proprietary voice codecs required expensive licensing fees for commercial use. GSM-EFR, AMR, G.729, and G.723.1 were all patented and carried licensing costs. Speex filled the gap by providing a CELP-based codec with wideband support, VAD/DTX, and variable bitrate capability — all without licensing restrictions.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| 0.0 | 2002 | Initial release, narrowband only |
| 0.8 | 2003 | Wideband support added |
| 1.0 | 2004 | Stability improvements, Ogg container integration |
| 1.1 | 2005 | Quality improvements, DTX, VAD refinements |
| 1.2 beta 1 | 2007 | Ultra-wideband mode, complexity control, fixed-point port |
| 1.2 beta 2 | 2008 | Bug fixes, stereo support improvements |
| 1.2 beta 3 | 2010 | Final development release; project moved to Opus |

### 1.3 Current Adoption
- **Primary use cases today:** Legacy VoIP applications, archival of older voice recordings, educational reference, legacy game audio engines (e.g., older versions of Mumble, TeamSpeak plugin support)
- **Platforms with native support:** Linux (via libspeex), macOS (via libspeex), iOS (third-party libraries), Android (via NDK / third-party), embedded Linux systems
- **Major services using this format:** Largely historical — Mumble VoIP (early versions), Teamspeak 3 (plugin), some early Asterisk VoIP deployments
- **Hardware support:** Very limited; primarily software decode/encode on general-purpose CPUs and DSPs
- **Status:** Deprecated — replaced by Opus, which incorporated Speex's best features (CELP, VBR, VAD/DTX) and added many improvements. Most modern applications have migrated to Opus. Speex files remain playable but are no longer generated for new projects.

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Hybrid — parametric / waveform hybrid speech coder
- **Core algorithm:** Code-Excited Linear Prediction (CELP) with algebraic codebook
- **Loss mechanism:** Perceptual quantization of CELP excitation and LPC parameters
- **Frame-based vs sample-based:** Frame-based encoding; fixed frame sizes per mode
- **Fixed vs variable frame size:** Fixed frame size per sampling rate mode, but bitrate is variable (VBR/CBR modes)

### 2.2 High-Level Encoding Flow (Block Diagram)
```
Input PCM Samples (8/16/32 kHz)
      │
      ▼
[Pre-processing: DC removal, high-pass filter, pre-emphasis]
      │
      ▼
[Analysis: Frame splitting, windowing]
      │
      ▼
[LPC Analysis: 8th–10th order linear prediction, Levinson-Durbin recursion]
      │
      ▼
[Pitch (LTP) Search: Closed-loop search for best pitch lag and gain]
      │
      ▼
[Algebraic Codebook Search: Find best excitation pulses]
      │
      ▼
[Parameter Quantization: Scalar/vector quantization of LPC, pitch, codebook]
      │
      ▼
[Bitstream Packing: Frame header, mode, parameters, side info]
      │
      ▼
Output Speex Bitstream (OGG container or raw .spx)
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | 30 ms (narrowband), 34 ms (wideband) | Frame size + lookahead |
| Frame size | 240 samples (NB, 8 kHz), 480 samples (WB, 16 kHz), 960 samples (UWB, 32 kHz) | Excludes lookahead |
| Max channels | 1 (mono), 2 (stereo via intensity encoding) | |
| Max bit depth | 16-bit PCM input | Internal: 32-bit float / 32-bit fixed |
| Max sample rate | 32 kHz (UWB); 8 kHz (NB), 16 kHz (WB) | |
| Bitrate range | 2.15–44.2 kbps | Mode and quality dependent |
| Complexity | O(1) with configurable complexity setting 1–10 | Higher = better quality, more CPU |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
Speex files stored in Ogg containers use the standard Ogg page structure. Raw `.spx` files have a simpler header:

```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       53 70 65 65      "Spee"    File signature ("Speex" without 'x')
0x0004  4       (version)        —         Speex version string
...     ...     ...               ...       Rest of header
```

Raw Speex bitstream (no container) structure:

```
Offset  Bytes   Field Name              Description
-------  ------  ---------------------  ----------------------------------
0x0000   8      "Speex   "            8-byte magic: "Speex   " (padded)
0x0008   4      Version ID            Integer version (currently 1)
0x000C   4      User-friendly version  Version string length + string
0x0010   4      Sample rate           Sample rate in Hz
0x0014   4      Mode                  0=NB, 1=WB, 2=UWB
0x0018   4      Mode bitstream version  Codec version for this mode
0x001C   4      Number of channels    1 = mono, 2 = stereo
0x0020   4      Bitrate               Nominal bitrate in bits/second
0x0024   4      Frame size            Frame size in samples
0x0028   4      VBR                   1 = VBR enabled, 0 = CBR
0x002C   4      Frame per packet      Frames per Ogg packet
0x0030   4      Extra headers         Number of extra headers following
0x0034   ...    Comments              Optional comment header
...      ...    Audio Data            Encoded frames
```

### 3.2 Ogg Container Wrapper
When Speex is stored in an Ogg container (`.spx` files, which is the standard), the structure follows the Ogg container specification:

```
Ogg Page Structure (per page):
  [0x00-0x03] page_sync: 4 bytes = 0x4F 67 67 53 ("OggS")
  [0x04]      version: 1 byte = 0x00
  [0x05]      header_type: 1 byte
  [0x06-0x09] granule_position: 8 bytes (little-endian)
  [0x0A-0x0D] bitstream_serial: 4 bytes
  [0x0E-0x11] page_sequence: 4 bytes
  [0x12-0x15] CRC_checksum: 4 bytes
  [0x16]      page_segments: 1 byte (0–255)
  [0x17-...]  segment_table: page_segments bytes
  [...]       page_data: variable, codec-specific
```

### 3.3 Frame / Block Header Layout (Speex Frame Bits)
A Speex frame is organized as follows (narrowband example at 8 kHz, mode 3):

```
Bit Offset   Bit Width   Field Name              Description
----------   ---------   --------------------    ---------------------------
0            1           Frame header / sync     Always 1 (intra-frame marker)
1            4           Mode bits               0=NB, 1=WB, 2=UWB, 3=SWB
5            1           Quality bits present   1 if quality is in-band
6            3           Mode request (in-band) For embedded/wideband
9            1           VBR flag               1 = variable bitrate
10           1           VAD/DTX flag           1 = voice activity detection
11           1           Frame size MSB         Part of frame size encoding
12           1           Reserved               Must be 0
13           8           Frame size LSB         Lower bits of frame size
...          ...         LPC parameters          Quantized LSP coefficients
...          ...         Pitch parameters        Lag and gain
...          ...         Excitation             Algebraic codebook indices
```

### 3.4 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Unsigned integer | No | Not supported |
| 16-bit | Signed integer | Yes | Primary input format |
| 20-bit | Signed integer | Yes | Internally promoted to 32-bit |
| 24-bit | Signed integer | Yes | Internally promoted to 32-bit |
| 32-bit | Signed integer | Yes | Accepted, truncated internally |
| 32-bit | IEEE float | Yes | Used in floating-point builds |
| 64-bit | IEEE float (double) | No | Not supported |

#### Supported Sample Rates
| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 8000 | Narrowband voice | Yes | Primary mode for telephony |
| 16000 | Wideband voice | Yes | Primary wideband mode |
| 32000 | Ultra-wideband voice | Yes | Highest Speex sample rate |
| 48000 | — | No | Not supported (use resampler) |
| 44100 | — | No | Not supported |
| 22050 | — | No | Not supported |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Pre-Processing Stage
- **DC offset removal:** High-pass filter with cutoff at approximately 80 Hz applied to remove DC component and low-frequency rumble
- **Pre-emphasis filter:** First-order high-pass filter H(z) = 1 − 0.7·z⁻¹ applied to emphasize high-frequency content before analysis
- **Windowing function:** Hann window applied to each frame of samples before LPC analysis
- **Level normalization:** Automatic gain control (AGC) in the Speex preprocessor, configurable from −24 dB to +24 dB
- **Stereo decorrelation pre-step:** For stereo input, the left and right channels are downmixed to mono for CELP encoding, with intensity stereo encoding applied to high-frequency bands

### 4.2 Analysis / Transform Stage

#### Transform Type: LPC (Linear Predictive Coding)
```
Parameters:
  LPC order:        8 (narrowband), 10 (wideband/ultra-wideband)
  Window:           Hann (raised cosine)
  Algorithm:        Levinson-Durbin recursion
  Coefficient precision:  32-bit float (floating-point build)
                        32-bit fixed (fixed-point build, Q15 format)
```

**Mathematical definition — LPC analysis:**
```
For frame n of N samples x[n], compute:
  Autocorrelation: r[k] = Σ(n=0 to N−k) x[n] · x[n+k], k = 0..p
  Levinson-Durbin: Solve R · a = −r for prediction coefficients a[1..p]
  where R is the Toeplitz matrix [r[i+j]] for i,j = 1..p
  Output: LP coefficients a[1], a[2], ..., a[p]
  Convert to Line Spectral Pairs (LSP) for quantization
```

**LSP (Line Spectral Pairs) conversion:**
- LPC coefficients are converted to LSP coefficients for better quantization properties
- LSP coefficients are ordered by frequency (ω₀ < ω₁ < ... < ωₚ₋₁)
- Stability is guaranteed as long as LSP coefficients remain interlaced
- Quantized using multi-stage vector quantization (MSVQ) or scalar quantization per mode

#### Pitch Analysis (Long-Term Prediction)
```
  Search range:  lag = 16 to 256 samples (NB, 8 kHz)
                 lag = 16 to 512 samples (WB, 16 kHz)
  Method:        Closed-loop search (analysis-by-synthesis)
  Resolution:    Integer or fractional (depending on mode)
  Open-loop pitch: Estimated first for search initialization
  Pitch gain:   Quantized to 4 levels: 0.0, 0.2, 0.5, 0.9 (approximate)
```

#### Algebraic Codebook Search
```
  Codebook type:  Algebraic (no stored table — computed on-the-fly)
  Grid:           2-pulse or 4-pulse per subframe depending on mode
  Search:         Staged search: pitch first, then codebook
  Complexity:     Controlled by encoder complexity setting (1–10)
  Pulse positions: Optimal positions selected via signal-to-noise ratio criterion
```

### 4.3 Bitrate Modes and Speex Modes

#### Narrowband (8 kHz)
| Mode | Bitrate (approx.) | Description |
|------|------------------|-------------|
| 0 | 2.15 kbps | Lowest quality, lowest bitrate |
| 1 | 3.95 kbps | Very low bitrate |
| 2 | 5.95 kbps | Low bitrate |
| 3 | 8.00 kbps | Default narrowband |
| 4 | 11.0 kbps | Medium quality |
| 5 | 15.0 kbps | Recommended narrowband |
| 6 | 18.2 kbps | High quality narrowband |
| 7 | 24.6 kbps | Highest narrowband bitrate |

#### Wideband / Ultra-wideband (16/32 kHz)
| Mode | Bitrate (approx.) | Description |
|------|------------------|-------------|
| 0 | 4.0 kbps | Very low |
| 1 | 5.8 kbps | Low |
| 2 | 7.0 kbps | — |
| 3 | 8.0 kbps | — |
| 4 | 9.2 kbps | — |
| 5 | 11.0 kbps | Default wideband |
| 6 | 13.3 kbps | — |
| 7 | 15.5 kbps | — |
| 8 | 18.8 kbps | — |
| 9 | 22.4 kbps | — |
| 10 | 27.8 kbps | Recommended wideband |
| 11 | 36.5 kbps | — |
| 12 | 44.2 kbps | Highest ultra-wideband |

### 4.4 Encoder Settings / Quality Modes

#### Speex Quality vs. Bitrate
| Quality Setting | Mode Used | Approx. Bitrate | Intended Use |
|----------------|-----------|----------------|--------------|
| 0 (lowest) | NB mode 0 | ~2.15 kbps | Extreme low bandwidth |
| 2 | NB mode 2 | ~5.95 kbps | Low bandwidth |
| 4 | NB mode 3 | ~8 kbps | Default (telephone quality) |
| 6 | NB mode 5 | ~15 kbps | Good VoIP quality |
| 8 (default) | WB mode 8 | ~18.8 kbps | Wideband quality |
| 10 (highest) | WB/UWB mode 10 | ~27.8–44.2 kbps | HD voice |

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. Scan for Ogg page sync word: 0x4F 67 67 53 ("OggS")
2. Verify page CRC-32 checksum
3. Identify Speex codec identification header (first header packet):
   - Magic bytes "Speex   "
   - Version string and sample rate
4. Parse comment header (second header packet)
5. Begin decoding audio pages, each containing one or more Speex frames
6. Track granule position for timestamp-to-sample mapping
```

#### Seeking in Ogg/Speex
- **Seeking:** By Ogg page index (indexed by granule position)
- **Seek table format:** Ogg provides a seek table via the `position` field in page headers
- **Seek precision:** Granule-position-based seeking; accuracy depends on frame size

### 5.2 Core Decode Pipeline
```
1. Read Ogg page header (27 + N bytes)
   ├── Verify sync word (0x4F676753)
   ├── Verify CRC-32 checksum
   └── Extract granule_position → compute presentation timestamp

2. Read Speex frame data from page payload
   ├── Parse frame header (mode bits, quality, VBR flag)
   └── Extract quantized parameter bitfields

3. Decode LSP coefficients
   └── Multi-stage vector dequantization → Levinson-Durbin → LPC filter

4. Decode pitch parameters
   └── Depacketize lag index, gain index → reconstruct pitch gain

5. Decode excitation
   └── Algebraic codebook: reconstruct pulse positions and amplitudes
   └── Add pitch contribution: excitation = pitch_excitation + codebook_excitation

6. CELP synthesis
   └── For each subframe:
       s[n] = excitation[n] + Σ(i=1 to p) a[i] · s[n−i]
       (LPC synthesis filter applied to total excitation)

7. Post-processing
   ├── High-pass post-filter (speex_preprocess, cutoff ~80 Hz)
   ├── Apply denoising (if preprocessing was enabled)
   └── AGC post-processing (if enabled)

8. Format output
   └── 16-bit PCM stereo or mono → write to output buffer
```

### 5.3 Error Concealment
- **Corrupt frame detection:** Ogg page CRC-32 check; invalid mode bits; out-of-range parameter values
- **Concealment method:** Packet loss concealment (PLC) using pitch-adaptive noise injection and previous frame repetition
- **Maximum consecutive errors before silence:** Codec-dependent; Speex PLC can sustain several frames before muting

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** Ogg (standard), or raw bitstream (.spx without container)
- **Overhead:** Minimal — Ogg page header overhead is ~27 bytes per page
- **Seeking in native container:** Yes — Ogg provides indexed seeking via granule position
- **Multiple streams in native container:** Yes — Ogg can multiplex audio, video, and text streams

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| OGG | Yes | Yes | Vorbis Comments | Standard .spx format |
| MP4/M4A | No | — | — | Speex not supported in MP4 |
| Matroska/MKA | Partial | Yes | Vorbis Comments | Via libmatroska custom element |
| WAV | No | — | — | No native Speex support |
| WebM | No | — | — | VP8/VP9 video, Opus audio |
| RTP | Yes | No | No | RFC 5574 payload format |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** Vorbis Comments (when in Ogg container)
- **Tag block location:** Second header packet after the identification header
- **Tag block identifier:** Ogg packet type = 3 (comment packet)

### 7.2 Standard Tag Fields — Complete Reference
| Tag Field | Internal Key (Vorbis Comments) | Max Length | Character Encoding | Multi-value | Notes |
|-----------|---------------------------|------------|-------------------|-------------|-------|
| Title | TITLE | Unlimited | UTF-8 | No | Track title |
| Artist | ARTIST | Unlimited | UTF-8 | No | Performer |
| Album | ALBUM | Unlimited | UTF-8 | No | Album name |
| Album Artist | ALBUMARTIST | Unlimited | UTF-8 | No | Album-level artist |
| Genre | GENRE | Unlimited | UTF-8 | No | Freeform genre |
| Year / Date | DATE | Unlimited | UTF-8 | No | Year or full date |
| Track Number | TRACKNUMBER | Unlimited | UTF-8 | No | Format: "N" |
| Disc Number | DISCNUMBER | Unlimited | UTF-8 | No | Format: "N" |
| Comment | COMMENT | Unlimited | UTF-8 | No | Freeform comment |
| Encoder | ENCODER | Unlimited | UTF-8 | No | Software encoder name |
| Vendor | VENDOR | Unlimited | UTF-8 | No | Library vendor string |

### 7.3 Cover Art Storage
- Cover art in Speex/Ogg files is stored as an optional Ogg page containing a METADATA_BLOCK_PICTURE Vorbis comment field. The field contains a base64-encoded Picture block:

```
METADATA_BLOCK_PICTURE format:
  [0-3]  picture_type (uint32): 0–20 (3 = front cover recommended)
  [4-7]  mime_length (uint32)
  [8..]  mime_type (UTF-8 string)
  [...]  description_length (uint32)
  [...]  description (UTF-8 string)
  [...]  width (uint32), height (uint32), color_depth (uint32), colors_used (uint32)
  [...]  picture_data_length (uint32)
  [...]  picture_data (binary)
```

### 7.4 Metadata Compatibility Matrix
| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| Vorbis Comments | ✓ | ✓ | ✓ | Highest (native) |
| ID3v1 | ✗ | ✗ | ✗ | N/A |
| ID3v2 | ✗ | ✗ | ✗ | N/A |
| APEv2 | ✗ | ✗ | ✗ | N/A |

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   libspeex           # used with -c:a
AV_CODEC_ID:        AV_CODEC_ID_SPEEX  # C constant in libavcodec/codec_id.h
Format Name (CLI):  ogg                # Ogg container for .spx files
Encoder(s):         libspeex           # via libspeex library
Decoder(s):         libspeex           # via libspeex library (FFmpeg >= 1.0)
Muxer(s):          ogg                # Ogg muxer
Demuxer(s):        ogg                # Ogg demuxer
```

**Note on FFmpeg >= 4.x:** FFmpeg's Speex decoder and encoder are implemented as a thin wrapper around the external `libspeex` library (`libavcodec/libspeexdec.c` for decoding, `libavcodec/libspeexenc.c` for encoding). This is independent of Vorbis decoding. In older FFmpeg versions, Vorbis decoding was done via external `libvorbis`; FFmpeg 4.0+ ships a native Vorbis decoder. Speex has always used `libspeex` via the wrapper.

### 8.2 FFmpeg Encoding — Full CLI Reference

```bash
# Encode WAV to Speex OGG (narrowband, 8 kHz)
ffmpeg -i input.wav \
  -c:a libspeex \
  -ar 8000 \
  -ac 1 \
  -b:a 8k \                  # Target bitrate (CBR mode)
  -libspeex_bitrate 8000 \   # Codec-specific bitrate (overrides -b:a)
  -libspeex_quality 8 \     # Quality 0-10 (higher = better quality)
  -libspeex_framesize 20 \  # Frame size in ms (default: 20)
  -libspeex_dtx 0 \          # DTX: 0=off, 1=on (discontinuous transmission)
  output.spx

# Encode to Speex with VBR (Variable Bitrate)
ffmpeg -i input.wav \
  -c:a libspeex \
  -ar 16000 \
  -ac 1 \
  -vbr 1 \                  # VBR mode: 0=off(CBR), 1=on(VBR)
  -q:a 8 \                 # Quality for VBR mode
  output_wb.spx

# Wideband Speex (16 kHz)
ffmpeg -i input_wb.wav \
  -c:a libspeex \
  -ar 16000 \
  -ac 1 \
  -libspeex_quality 10 \
  -libspeex_bitrate 27000 \
  output_wb.spx
```

#### Complete FFmpeg Option Table
| Option | Type | Default | Range / Valid Values | Effect on Output |
|--------|------|---------|---------------------|-----------------|
| `-b:a` | int | auto | 2150–44200 | Target bitrate in bps |
| `-libspeex_bitrate` | int | 0 | 0=auto, 2150–44200 | Codec-specific bitrate |
| `-libspeex_quality` | int | 8 | 0–10 | Encoding quality |
| `-libspeex_framesize` | int | 20 | 20, 40, 60 | Frame size in ms |
| `-vbr` | int | 0 | 0=off, 1=on | Variable bitrate mode |
| `-ar` | int | 8000 | 8000, 16000, 32000 | Sample rate |
| `-ac` | int | 1 | 1, 2 | Channel count |

### 8.3 FFmpeg Encoding — libavcodec C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/opt.h>
#include <libswresample/swresample.h>

// ─── 1. Find encoder ─────────────────────────────────────────────────────────
const AVCodec *codec = avcodec_find_encoder(AV_CODEC_ID_SPEEX);
// Alternative by name: avcodec_find_encoder_by_name("libspeex");
if (!codec) { fprintf(stderr, "Encoder not found\n"); exit(1); }

// ─── 2. Allocate codec context ────────────────────────────────────────────────
AVCodecContext *ctx = avcodec_alloc_context3(codec);
if (!ctx) { fprintf(stderr, "OOM\n"); exit(1); }

// ─── 3. Set required parameters ────────────────────────────────────────────────
ctx->bit_rate    = 8000;            // bits/sec for narrowband Speex
ctx->sample_fmt  = AV_SAMPLE_FMT_S16; // REQUIRED: check codec->sample_fmts[]
ctx->sample_rate = 8000;            // Hz — Speex supports 8000, 16000, 32000
av_channel_layout_default(&ctx->ch_layout, 1); // mono

// Speex-specific options via AVDictionary:
av_opt_set_int(ctx,    "libspeex_bitrate",  8000,   AV_OPT_SEARCH_CHILDREN);
av_opt_set_int(ctx,    "libspeex_quality",  8,      AV_OPT_SEARCH_CHILDREN);
av_opt_set_int(ctx,    "libspeex_framesize", 20,    AV_OPT_SEARCH_CHILDREN);
av_opt_set_int(ctx,    "vbr",               0,      AV_OPT_SEARCH_CHILDREN);

// ─── 4. Open codec ───────────────────────────────────────────────────────────
int ret = avcodec_open2(ctx, codec, NULL);
if (ret < 0) {
    char errbuf[128]; av_strerror(ret, errbuf, sizeof(errbuf));
    fprintf(stderr, "avcodec_open2 failed: %s\n", errbuf);
    exit(1);
}

// ─── 5. Allocate frame and packet ────────────────────────────────────────────
AVFrame  *frame = av_frame_alloc();
frame->format      = ctx->sample_fmt;
frame->sample_rate = ctx->sample_rate;
av_channel_layout_copy(&frame->ch_layout, &ctx->ch_layout);
frame->nb_samples  = ctx->frame_size; // Fixed at 160 samples for 8kHz, 20ms
av_frame_get_buffer(frame, 0);

AVPacket *pkt = av_packet_alloc();

// ─── 6. Encode loop ───────────────────────────────────────────────────────────
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

// ─── 7. Flush encoder ─────────────────────────────────────────────────────────
encode_frame(ctx, NULL, pkt, outfile); // NULL frame triggers flush/drain

// ─── 8. Cleanup ───────────────────────────────────────────────────────────────
av_frame_free(&frame);
av_packet_free(&pkt);
avcodec_free_context(&ctx);
```

**Critical Notes:**
- Speex's `ctx->frame_size` is 160 samples at 8 kHz (20 ms), 320 at 16 kHz, 640 at 32 kHz
- Speex requires `AV_SAMPLE_FMT_S16` (planar signed 16-bit PCM) — NOT float
- Always resample input to exactly 8000, 16000, or 32000 Hz before encoding
- For stereo, use two mono streams or intensity stereo encoding

### 8.4 FFmpeg Decoding — Full CLI Reference

```bash
# Decode Speex OGG to WAV PCM
ffmpeg -i input.spx \
  -c:a pcm_s16le \
  -ar 16000 \           # Output at 16 kHz (or resample if needed)
  -ac 1 \               # Mono output
  output.wav

# Decode to raw PCM
ffmpeg -i input.spx \
  -c:a pcm_s16le \
  -f s16le \
  output.raw

# Probe format without decoding
ffprobe -v quiet -print_format json -show_streams -show_format input.spx

# Extract audio from Ogg container, copy to new container
ffmpeg -i input.spx -c:a copy -map 0:a output_copy.ogg
```

### 8.5 FFmpeg Decoding — libavcodec C API

```c
// ─── Complete decode program skeleton ────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.spx", NULL, NULL);
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
        ret = avcodec_send_packet(dec_ctx, pkt);
        if (ret < 0) { /* handle error */ }
        while (avcodec_receive_frame(dec_ctx, frm) == 0) {
            // frm->data[0] = PCM samples (S16 format)
            // frm->nb_samples = samples per channel
            // frm->sample_rate = 8000/16000/32000
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

### 8.6 FFmpeg Metadata Handling

```bash
# Read all metadata as JSON
ffprobe -v quiet -print_format json -show_format input.spx | jq .format.tags

# Write metadata to Speex/Ogg
ffmpeg -i input.spx \
  -c:a copy \
  -metadata title="Recording Title" \
  -metadata artist="Speaker Name" \
  -metadata date="2024" \
  -metadata genre="Speech" \
  output_tagged.spx

# Strip all metadata
ffmpeg -i input.spx -c:a copy -map_metadata -1 output_clean.spx

# Embed cover art
ffmpeg -i input.spx -i cover.jpg \
  -c:a copy \
  -c:v copy \
  -map 0:a -map 1:v \
  -metadata:s:v title="Album cover" \
  -disposition:v attached_pic \
  output_with_cover.spx
```

#### FFmpeg Internal Metadata Key Mapping
| Standard Field | FFmpeg Key | Vorbis Native Key | Notes |
|----------------|------------|--------------------------|-------|
| Title | title | TITLE | |
| Artist | artist | ARTIST | |
| Album | album | ALBUM | |
| Album Artist | album_artist | ALBUMARTIST | |
| Track Number | track | TRACKNUMBER | |
| Disc Number | disc | DISCNUMBER | |
| Genre | genre | GENRE | |
| Date/Year | date | DATE | |
| Comment | comment | COMMENT | |
| Encoder | encoder | ENCODER | Auto-set by encoder |
| Vendor | — | VENDOR | Set by library |

### 8.7 Quality / Fidelity Decision Table
| Use Case | Settings | Expected Size | Notes |
|----------|----------|---------------|-------|
| Legacy VoIP archival | `-c:a libspeex -b:a 8k -ar 8000` | ~2.5 KB/min | Narrowband |
| Standard VoIP | `-c:a libspeex -b:a 15k -ar 16000` | ~56 KB/min | Wideband |
| HD voice archival | `-c:a libspeex -vbr 1 -q:a 8 -ar 16000` | ~80 KB/min | VBR |
| Low bandwidth emergency | `-c:a libspeex -b:a 2.15k -ar 8000` | ~0.8 KB/min | Mode 0 |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
```
Ogg Bitstream Seek Index:
  Location:     Within Ogg page headers (granule_position field)
  Entry size:  8 bytes per index entry
  Entry format:
    [0x00–0x07]  granule_position (uint64) — absolute sample number
    [0x08–0x0B]  page_offset (uint32) — byte offset in file
  Max entries: Unlimited
```

### 9.2 Gapless Playback Data
```
Encoder delay:   ~2.5 ms (lookahead for pitch search in CELP)
Padding:         ~2.5 ms (encoder lookahead, decoder needs this to reconstruct)
Storage location: Not stored in Speex — embedded in Ogg granule_position
                  granule_position on first audio page = 0
                  granule_position on last page = total samples + encoder_delay

FFmpeg gapless flags:
  Input:  Ogg demuxer uses granule_position for seeking
  Output: -movflags faststart (not applicable to Ogg)
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | Ogg container allows progressive download |
| Algorithmic encoder delay | 30 ms (NB), 34 ms (WB) | Frame + lookahead |
| Live encoding feasible | Yes | Speex was designed for real-time VoIP |
| HTTP progressive download | Yes | Ogg/Speex supports range requests |
| HTTP Live Streaming (HLS) | No | No native HLS muxer for Speex |
| DASH streaming | No | Not commonly used |
| WebRTC / RTP transport | Yes | RFC 5574 defines RTP payload for Speex |
| Minimum decode buffer | 1 frame (20 ms) | Very low latency codec |

**RTP Payload Format (RFC 5574):**
- Speex frames are mapped to RTP packets with optional Speex-specific header
- RTP timestamp increments by 160 (NB), 320 (WB), 640 (UWB) per frame
- Payload type: dynamic (not a fixed RFC 3551 type assignment)
- The Speex RTP payload includes an optional TOC (Table of Contents) byte for multiple frames per packet

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | Layout Name | Channel Order | FFmpeg Layout Constant | Notes |
|----------|-------------|---------------|----------------------|-------|
| 1 | Mono | C | AV_CHANNEL_LAYOUT_MONO | Primary mode |
| 2 | Stereo (intensity) | L, R | AV_CHANNEL_LAYOUT_STEREO | Intensity stereo only |

**Note:** Speex does NOT support true stereo encoding (L/R independently encoded). Stereo in Speex uses intensity stereo encoding where only the overall energy envelope is transmitted for the high-frequency bands, and only mono is encoded for the baseband. This is similar to MP3's intensity stereo but more limited. True stereo requires dual mono encoding (2× bitrate).

### 11.2 Downmix Coefficients (2.0 → 1.0)
```
L_out = (L_in + R_in) / 2
R_out = (discarded for mono output)

FFmpeg downmix command for Speex stereo to mono:
ffmpeg -i input_stereo.spx \
  -af "apan=mono=c=FC" \
  output_mono.spx
```

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 16-bit PCM input | Internally processed at higher precision |
| Max sample rate | 32 kHz | Ultra-wideband mode |
| Float support | 32-bit float only (floating-point build) | Not in fixed-point builds |
| DSD support | No | Not applicable |
| 20-bit support | No | Promoted to 32-bit internally |
| 24-bit support | No | Promoted to 32-bit internally |

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

| Platform / API | Encode | Decode | FFmpeg Flag | Notes |
|----------------|--------|--------|-------------|-------|
| x86/x64 (generic) | Yes | Yes | None | libspeex runs on CPU |
| ARM (32/64-bit) | Yes | Yes | None | libspeex has NEON optimizations |
| MIPS | Yes | Yes | None | libspeex has MIPS optimizations |
| Intel QSV | No | No | — | No QSV plugin for Speex |
| Apple AudioToolbox | No | No | — | No native Speex in iOS/macOS |
| Android MediaCodec | No | No | — | No native MediaCodec for Speex |
| VA-API (Linux) | No | No | — | Not applicable |

**Note:** Speex hardware acceleration is virtually nonexistent because:
1. The codec is deprecated and replaced by Opus
2. Modern devices support Opus natively in hardware (Intel, ARM, Qualcomm DSPs)
3. The market incentive for Speex hardware support was never strong due to its free licensing model

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Encoder-Specific Issues
| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| Speex encoder not available without libspeex | All | Build FFmpeg with `--enable-libspeex` |
| libspeex library not found | All | Install libspeex-dev or compile libspeex |
| Fixed-point Speex vs floating-point | All | Use matching FFmpeg build |
| Ogg container CRC errors | Rare | Regenerate from source |

### 14.2 Interoperability Issues
- **libspeex → other decoders:** Speex decoders from different implementations (Xiph libspeex, FFmpeg wrapper, open-source ports) may produce slightly different output due to floating-point precision differences. For bit-exact reproduction, use the same library version.
- **Wideband Ogg streams:** Some legacy players may only support narrowband Speex and reject wideband streams.
- **VBR + streaming:** VBR (variable bitrate) Speex causes variable packet sizes, which can stress some network jitter buffers.
- **Files with corrupt comment header:** FFmpeg may skip metadata or fail to read the file if the comment header is malformed.

### 14.3 Edge Cases to Handle in Converter
- **Empty file / 0-sample stream:** Produce empty output; do not write headers with no audio data
- **File < 1 frame of audio:** Encode as partial frame or pad with silence
- **All-silence audio:** Speex encoder may output DTX frames (comfort noise) — handle as valid frames
- **DC offset (non-zero mean):** Pre-filter removes DC before encoding; output may have small DC
- **Full-scale sine (0 dB):** No clipping expected; encoder handles peak levels
- **Corrupt frame:** Ogg page CRC detects corruption; FFmpeg decoder may mute or repeat last frame
- **Truncated file:** Decoder mutes at end; no crash expected
- **Sample rate not supported by codec:** FFmpeg resamples to nearest supported rate (8k/16k/32k)
- **Channel count not supported:** Mono is standard; stereo downmixes to mono or encodes as dual mono

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM Speex

| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → FLAC | `ffmpeg -i in.spx -c:a flac -compression_level 8 out.flac` | All (Vorbis→Vorbis Comments) | Lossless if source decoded first |
| → WAV | `ffmpeg -i in.spx -c:a pcm_s16le out.wav` | RIFF INFO (partial) | Lossless decode |
| → MP3 | `ffmpeg -i in.spx -c:a libmp3lame -q:a 0 out.mp3` | ID3v2.3 (title/artist/album) | Generation loss |
| → AAC | `ffmpeg -i in.spx -c:a aac -b:a 128k out.m4a` | Partial | Generation loss |
| → Opus | `ffmpeg -i in.spx -c:a libopus -b:a 64k out.opus` | Vorbis Comments | Generation loss |
| → OGG Vorbis | `ffmpeg -i in.spx -c:a libvorbis -q:a 6 out.ogg` | Vorbis Comments | Generation loss |

### 15.2 Converting TO Speex

| Source | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| WAV (8kHz) → | `ffmpeg -i in.wav -c:a libspeex -ar 8000 -b:a 8k out.spx` | Vorbis Comments | Lossy |
| WAV (16kHz) → | `ffmpeg -i in.wav -c:a libspeex -ar 16000 -b:a 18k out.spx` | Vorbis Comments | Lossy |
| FLAC → | `ffmpeg -i in.flac -c:a libspeex -ar 8000 out.spx` | Partial (Vorbis→Ogg) | Transcode lossy |
| MP3 → | `ffmpeg -i in.mp3 -c:a libspeex -ar 8000 out.spx` | ID3→Vorbis (partial) | Double lossy |
| Opus → | `ffmpeg -i in.opus -c:a libspeex -ar 8000 out.spx` | Partial | Transcode lossy |

### 15.3 Lossless Round-Trip Verification
```bash
# NOTE: Speex is lossy — true lossless round-trip is NOT possible
# Lossless verification for decode pipeline only:

# Decode Speex to PCM
ffmpeg -i original.spx -c:a pcm_s16le original_decoded.wav

# Decode another Speex file
ffmpeg -i re-encoded.spx -c:a pcm_s16le re_decoded.wav

# Compare decoded outputs (should match for same source)
md5sum original_decoded.wav re_decoded.wav
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| libspeex (official) | C | BSD/Xiph | Reference | Reference | https://www.speex.org/ |
| FFmpeg native wrapper | C | LGPL 2.1+ | Good | Good | https://ffmpeg.org |
| SpeexDSP | C | BSD | — (decoder only) | Reference | https://github.com/xiph/speexdsp |

### Build Instructions (for bundling in converter app)
```bash
# Build libspeex from source
git clone https://github.com/xiph/speexdsp.git
cd speexdsp
./bootstrap
./configure --prefix=/usr/local --disable-shared --enable-static
make -j$(nproc)
make install

# Build FFmpeg with Speex support:
./configure --prefix=/usr/local --enable-libspeex --enable-nonfree \
  --disable-shared --enable-static
make -j$(nproc)
make install

# Link:
# LDFLAGS += -L/usr/local/lib -lspeex -lspeexdsp
# CFLAGS  += -I/usr/local/include/speex
```

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **RFC 5574:** RTP Payload Format for the Speex Codec — https://www.rfc-editor.org/rfc/rfc5574
- **Speex Codec Manual:** https://www.speex.org/docs/manual/speex-manual.pdf
- **Xiph.org Speex Home:** https://www.speex.org/
- **3GPP TS 26.071:** AMR Codec General Description (related CELP codec for comparison)
- **IETF RFC 5574:** Speex in RTP (covers narrowband/wideband/UWB, VBR, TOC)

### Technical Resources
- FFmpeg encoder options: `ffmpeg -h encoder=libspeex` or https://ffmpeg.org/ffmpeg-codecs.html
- Speex DSP library: https://github.com/xiph/speexdsp
- Speex source code: https://github.com/xiph/speex
- Multimedia Wiki: https://wiki.multimedia.cx/index.php/Speex
- Hydrogenaudio Speex forum: https://hydrogenaud.io/index.php/board,54.0.html

### Academic Papers
- Jean-Marc Valin, "Speex: A Free Codec for Free Speech", Xiph.org, 2006 — https://speex.org/docs/
- Jean-Marc Valin et al., "Speex: A Open-Source/Free Speech Codec", Proc. of the 11th ACM Workshop on Multimedia, 2009
- Acepnt (RFC 5574 authors): Jean-Marc Valin — specification is the primary reference
- "On the Use of G.711, G.729, Speex and AMR for VoIP", comparative study

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] Identify FFmpeg `./configure` flags needed: `--enable-libspeex`
- [ ] Verify `ffmpeg -encoders` output confirms encoder is available
- [ ] Verify `ffmpeg -decoders` output confirms decoder is available
- [ ] Note external library dependency: `libspeex` (or `speexdsp` for decode only)
- [ ] Handle platform where libspeex may not be available (Windows binaries may be limited)

### Encoding Pipeline
- [ ] Convert input sample format to S16 using libswresample
- [ ] Validate input sample rate is 8000, 16000, or 32000 Hz (resample if not)
- [ ] Set `ctx->frame_size` correctly: 160 (NB), 320 (WB), 640 (UWB)
- [ ] Implement proper flush/drain at end of stream (send NULL frame)
- [ ] Use Ogg muxer with `-f ogg` or default `.spx` extension auto-selects Ogg
- [ ] Store Vorbis Comments for metadata

### Decoding Pipeline
- [ ] Use Ogg demuxer to read `.spx` files
- [ ] Handle AVERROR(EAGAIN) in receive_packet loop correctly
- [ ] Flush decoder at EOF (send NULL packet)
- [ ] Handle S16 output format from Speex decoder (not float)
- [ ] Resample if output needs a different sample rate than source

### Metadata
- [ ] Read all Vorbis Comment fields from Ogg comment header
- [ ] Map standard keys: TITLE, ARTIST, ALBUM, DATE, GENRE, TRACKNUMBER
- [ ] Handle vendor string (ENCODER field)
- [ ] Read METADATA_BLOCK_PICTURE for cover art
- [ ] Write Vorbis Comments to Ogg output container
- [ ] Preserve ReplayGain tags (not native to Speex; may be embedded as comments)

### Quality & Verification
- [ ] Implement VBR vs CBR selection for encoding
- [ ] Test encode at all 3 sample rates (NB/WB/UWB)
- [ ] Verify mono encoding and stereo intensity encoding
- [ ] Test with: silence, speech, music-on-hold, DTMF tones

### Edge Cases
- [ ] Handle files with corrupt Ogg page CRC (skip or mute frame)
- [ ] Handle files with 0 samples
- [ ] Handle sample rate mismatch (resample to 8k/16k/32k)
- [ ] Handle stereo input with mono encoder (auto-downmix)
- [ ] Handle very short files (< 1 frame)

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
