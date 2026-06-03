# G.726 ADPCM — Deep Technical Reference
> **Category:** Lossy
> **File Extensions:** `.g726`, `.adpcm`, `.wav` (when in RIFF/WAV)
> **MIME Types:** `audio/g726`, `audio/x-adpcm`
> **Standardization Body:** ITU-T (International Telecommunication Union)
> **Primary Specification:** ITU-T Recommendation G.726 — 40, 32, 24, 16 kbit/s Adaptive Differential Pulse Code Modulation (ADPCM) (December 1990)
> **Patent Status:** Patent-free — ITU-T standard, freely available to implement
> **License:** ITU copyright but free to implement; no licensing fees
> **Current Version:** G.726 (1990) + Corrigendum 1 (2005), Annex A (1994), Annex B (2003)
> **Active Development:** No — stable, mature standard; no active development

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** CCITT (now ITU-T), Study Group 15
- **Year Created:** 1990 (approved December 14, 1990)
- **Original Purpose:** G.726 unified and replaced two earlier ADPCM standards — G.721 (which covered only 32 kbps ADPCM) and G.723 (which covered 24 and 40 kbps ADPCM). It also introduced a new 16 kbps rate. The goal was to provide a single, unified specification for adaptive differential PCM at multiple bitrates, enabling efficient digital transmission of voice over channels narrower than the standard 64 kbps G.711 PCM channel.
- **Problem with Predecessors:** Before G.726, there were separate recommendations for different ADPCM bitrates (G.721, G.723), creating fragmentation. Also, G.721 required G.711 A-law or μ-law PCM input, not linear PCM. G.726 unified all four bitrates into one standard and added Annex A support for linear PCM I/O, making it a more complete and flexible solution for digital circuit multiplication equipment (DCME), satellite links, and VoIP gateways.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| G.726 (original) | 1990 | Unified G.721 (32 kbps) and G.723 (24/40 kbps), added 16 kbps |
| G.726 Annex A | 1994 | Added support for linear PCM input/output (14-bit) |
| G.726 Appendix III | 1994 | Comparison of ADPCM algorithms (with G.727) |
| G.726 Corrigendum 1 | 2005 | Fixed bug in Annex A decoder (LIMO block) |
| G.726 Annex B | 2003 | Packet format, capability identifier for H.245 signaling |

### 1.3 Current Adoption
- **Primary use cases today:** VoIP gateways (low-bitrate alternative to G.711), digital circuit multiplication equipment (DCME), satellite phone links, digital PBX trunking, fax over IP (FoIP), legacy voice compression in enterprise equipment
- **Platforms with native support:** Linux (via FFmpeg), macOS (via FFmpeg), embedded systems, some VoIP hardware (Cisco, Polycom gateways)
- **Major services using this format:** Primarily enterprise and carrier infrastructure; not common in consumer applications
- **Hardware support:** Some DSP chips in legacy telecom equipment; most modern implementations use software
- **Status:** Declining — G.711 remains dominant for VoIP, and newer codecs (AMR, Opus, EVS) offer better quality at similar or lower bitrates. G.726 is still used in legacy infrastructure and some satellite communications.

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Waveform codec — Adaptive Differential Pulse Code Modulation (ADPCM)
- **Core algorithm:** ADPCM with adaptive quantizer and adaptive predictor
- **Loss mechanism:** Adaptive quantization of the prediction error (difference signal); fewer bits per sample at lower bitrates
- **Frame-based vs sample-based:** Sample-based encoding with adaptive prediction per sample; frame structure is defined by the transport layer
- **Fixed vs variable frame size:** Not a frame-based codec — samples are encoded individually, but packetization is done by the transport (typically 80 samples = 10 ms)

### 2.2 High-Level Encoding Flow (Block Diagram)
```
Input: A-law or μ-law PCM (64 kbps)  OR  14-bit linear PCM (via Annex A)
      │
      ▼
[Inverse Companding: A-law/μ-law → linear PCM (if applicable)]
      │
      ▼
[Adaptive Quantizer: Convert linear PCM to log-PCM]
      │
      ▼
[Differential Encoding: Compute d(k) = s(k) - ŝ(k)]
      │           where s(k) = input, ŝ(k) = predicted signal
      ▼
[Adaptive Predictor: Update predictor based on quantizer output]
      │           Six zero taps (predictor of differences)
      │           Two pole taps (second-order predictor)
      ▼
[Adaptive Quantizer: Quantize differential signal]
      │           4-bit (32 kbps), 3-bit (24 kbps), etc.
      ▼
[Output: G.726 bitstream (2/3/4/5 bits per sample)]
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | 0.125 ms (1 sample) | No lookahead in basic algorithm |
| Sample rate | 8,000 Hz | Same as G.711 |
| Bit depth | 14-bit linear (Annex A), A/μ-law input (base spec) | |
| Max channels | 1 (mono) | |
| Bandwidth | 300–3,400 Hz (narrowband voice) | Same as G.711 |
| Bitrate range | 16, 24, 32, 40 kbps | 4 modes |
| Complexity | Low (simple multiply-accumulate) | |
| MOS Score | ~3.5 (32 kbps), ~2.9 (16 kbps) | Quality degrades at lower bitrates |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 G.726 Bit Rate and Quantization

#### G.726 Quantizer Specifications
Per ITU-T G.726, the adaptive quantizer operates on the difference signal d(k):

| Bit Rate | Bits per Sample | Quantization Levels | Quantizer Step Size (Δ) | Application |
|----------|----------------|--------------------|-----------------------|-------------|
| 40 kbps | 5 bits | 31 levels | Adaptive | Data modem signals, DCME |
| 32 kbps | 4 bits | 15 levels | Adaptive | Voice (most common) |
| 24 kbps | 3 bits | 7 levels (odd-level) | Adaptive | Voice, satellite |
| 16 kbps | 2 bits | 4 levels (even-level) | Adaptive | Voice, circuit multiplexing |

#### Quantizer Step Size Adaptation
The step size Δ adapts based on the quantizer output:
```
Δ(k+1) = Δ(k) × α^q(k)
where:
  α = 0.0–0.9 depending on bit rate and quantizer input
  q(k) = quantizer output level index
  Fast adaptation: α_fast = 0.9 (for high input variance)
  Slow adaptation: α_slow = 0.99 (for steady signals)
```

#### Predictor Structure (6-tap FIR + 2-tap IIR)
```
Predictor: ŝ(k) = Σ(b_i × d_quot_i(k-i)) + Σ(a_j × s(k-j))
  b_i = zero coefficients (i = 1..6)
  a_j = pole coefficients (j = 1..2)
  d_quot = quantized difference signal
  s(k) = reconstructed signal

Adaptation:
  b_i(k+1) = b_i(k) + β × sign(d(k)) × sign(d_quot(k-i))
  a_j(k+1) = a_j(k) + β × sign(d(k)) × sign(d_quot(k-j))
  where β = adaptation coefficient (0.0 to 1.0)
```

### 3.2 G.726 Frame / Packet Structure
G.726 does not define a native file format. It is typically transported as:

```
RTP Packet for G.726 (RFC 3551):
  [0x00]  NAL header (if using NAL mode) or raw G.726 bytes
  [0x01+] G.726 samples packed into bytes

Packing of G.726 samples into bytes:
  40 kbps (5 bits/sample):  8 samples span 5 bytes
  32 kbps (4 bits/sample): 2 samples per byte
  24 kbps (3 bits/sample): 8 samples span 3 bytes  (complex packing)
  16 kbps (2 bits/sample): 4 samples per byte

Example — 32 kbps packing:
  Sample A: bits [7..4]
  Sample B: bits [3..0]
  Both stored in 1 byte:  [AAAA BBBB]
```

### 3.3 WAV File Format for G.726
When stored in WAV, G.726 uses the Microsoft ADPCM format:

```
WAV fmt chunk for G.726 ADPCM:
Offset   Size    Field Name              Type        Value         Description
-------  ------  ----------------------  ----------  -----------   ------------------------
0x0000   4B      Subchunk1ID           char[4]     "fmt "       Format chunk identifier
0x0004   4B      Subchunk1Size         uint32 LE   20 or 22      Size of fmt data
0x0008   2B      AudioFormat           uint16 LE   0x0040        wFormatTag = 0x0040 (ADPCM)
0x000A   2B      NumChannels           uint16 LE   1             Mono
0x000C   4B      SampleRate            uint32 LE   8000          8 kHz
0x0010   4B      ByteRate              uint32 LE   3200/4000     bytes/sec (32k/40k bps)
0x0014   2B      BlockAlign            uint16 LE   varies        Samples per block
0x0016   2B      BitsPerSample         uint16 LE   4             ADPCM bits/sample
0x0018   2B      ExtraParamBytes       uint16 LE   2             Extra format bytes
0x001A   2B      SamplesPerBlock       uint16 LE   varies        ADPCM samples per block
0x001C   ...     Extra format bytes    varies       —             Codec-specific
```

**Microsoft ADPCM WAV format tag:** 0x0040 (decimal 64)

### 3.4 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | A-law PCM | Yes | Standard input (converted to linear internally) |
| 8-bit | μ-law PCM | Yes | Standard input (converted to linear internally) |
| 14-bit | Signed linear | Yes | Annex A input/output |
| 16-bit | Signed linear | Yes | Accepted, truncated to 14-bit internally |
| 24-bit | Signed integer | Yes | Accepted, truncated to 14-bit internally |
| 32-bit | IEEE float | No | Not supported |

#### Supported Sample Rates
| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 8000 | Telephone | Yes | Fixed by G.726 standard |
| 16000 | Wideband | No | Not supported |
| 44100 | CD audio | No | Not supported |
| 48000 | Professional | No | Not supported |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 ADPCM Principles

ADPCM achieves compression by exploiting the high correlation between consecutive speech samples. Instead of encoding the absolute sample value, ADPCM encodes the **difference** between the current sample and its predicted value. The predictor and quantizer adapt continuously.

**Core ADPCM theory:**
```
Input: s(k) = current linear PCM sample
Predicted: ŝ(k) = prediction based on past samples and quantized differences
Difference: d(k) = s(k) - ŝ(k)  (the signal to quantize)
Quantized difference: d_q(k) = Q{d(k)}
Reconstructed: sr(k) = ŝ(k) + d_q(k)
Next prediction: ŝ(k+1) = f(sr(k), sr(k-1), ...)

Output bitstream: bits encoding d_q(k)
```

**Why ADPCM works:**
- Speech is highly correlated between samples — adjacent samples differ by less than the full dynamic range
- The prediction error d(k) has a smaller variance than s(k) itself
- Quantizing d(k) requires fewer bits than quantizing s(k) directly
- The predictor adapts to the signal statistics, further reducing prediction error variance

### 4.2 Adaptive Quantizer

For 32 kbps (4-bit ADPCM), the quantizer maps the difference signal d(k) to one of 15 levels:

```
32 kbps (4-bit) quantizer:
  Input range:    ±Δ(k) × [−7.5, −5.5, −4.5, −3.5, −2.5, −1.5, −0.5, +0.5, +1.5, +2.5, +3.5, +4.5, +5.5, +6.5, +7.5]
  Step size Δ:    Adapts from ~0.1 to ~2048 based on signal level
  Adaptation:     Multiplied by 1.1 when quantizer output is ±7, ±8
                  Multiplied by 0.9 when quantizer output is ±1, ±2

16 kbps (2-bit) quantizer — even-level (special case):
  Uses different quantization levels than 3/4-bit quantizers
  The 16 kbps quantizer was specially designed for better performance
  Output: 4 levels with nonlinear spacing
```

### 4.3 Adaptive Predictor

The G.726 predictor combines a 6-tap zero (FIR) predictor and a 2-pole (IIR) predictor:

```
Zero predictor: Σ(b_i × d_q(k-i)) for i = 1..6
Pole predictor: Σ(a_j × sr(k-j)) for j = 1..2

Adaptation algorithm (normalized gradient):
  μ = adaptation coefficient = 2^-11 (≈ 0.0005)
  b_i(k+1) = b_i(k) + sign(d(k)) × sign(d_q(k-i)) × μ  for i = 1..6
  a_j(k+1) = a_j(k) + sign(d(k)) × sign(sr(k-j)) × μ  for j = 1..2
  Coefficients are clipped to prevent overflow
```

### 4.4 Bitrate Selection Guide
| Bitrate | Quality | Typical Use | MOS Score |
|---------|---------|------------|-----------|
| 40 kbps | Near-toll | DCME data modem, high-quality voice | ~4.0 |
| 32 kbps | Good voice | VoIP gateways, digital trunking | ~3.5 |
| 24 kbps | Fair voice | Satellite links, low-bandwidth links | ~3.2 |
| 16 kbps | Low voice | Circuit multiplexing, emergency links | ~2.9 |

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking
G.726 is a continuous bitstream with no frame headers. Seeking is time-based:

```
Seeking in G.726:
  - Byte offset depends on bitrate:
      40 kbps: 5 bits/sample, byte = 8/5 = 1.6 samples
      32 kbps: 4 bits/sample, byte = 2 samples
      24 kbps: 3 bits/sample, byte = 8/3 ≈ 2.67 samples
      16 kbps: 2 bits/sample, byte = 4 samples
  - Time in ms = (byte_offset × 8 × 1000) / (bitrate)
  - Byte offset = time_ms × (bitrate / 8000)

WAV container:
  - Use sample_number from data chunk
  - sample_number × (bitrate / 8000) bytes per sample
```

### 5.2 Core Decode Pipeline
```
G.726 decode (inverse of encode):

1. Read N bits from bitstream (N = 2, 3, 4, or 5 depending on bitrate)

2. Dequantize bits to difference signal:
   d_q(k) = Δ(k) × quantizer_scale[bits]
   where Δ(k) is the current step size

3. Reconstruct signal:
   sr(k) = ŝ(k) + d_q(k)

4. Update predictor (same as encoder):
   Update zero coefficients b_i based on d_q(k)
   Update pole coefficients a_j based on sr(k)

5. Inverse adaptive quantizer step size update:
   Δ(k+1) = Δ(k) × α^q(k)

6. Apply inverse A-law/μ-law companding (if output requires it):
   Convert linear PCM to A-law or μ-law PCM

7. Output: 16-bit signed linear PCM or A/μ-law PCM
```

### 5.3 Error Concealment
- **Corrupt bit detection:** G.726 has no internal error detection — bit errors appear as audio distortion
- **Concealment method:** Transport-layer FEC or packet loss concealment (PLC); G.726 itself does not include error correction
- **Corrupted audio:** A single bit error affects the current sample and propagates briefly into future samples through the adaptive predictor (error propagation). Higher bitrates are more robust to single-bit errors.
- **Maximum error propagation:** With adaptive predictors, errors can propagate for several hundred samples (~50–100 ms) before the predictor re-converges.

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** None — G.726 is a raw bitstream format. Common containers: raw bitstream (`.g726`), WAV (`.wav` with ADPCM format tag), RTP
- **Overhead:** None for raw bitstream; WAV header overhead (~44 bytes)
- **Seeking in WAV:** Yes — by byte offset within data chunk
- **Multiple streams:** Not natively — G.726 is mono

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| WAV (.wav) | Yes (fmt=0x0040) | Yes | RIFF INFO tags | Microsoft ADPCM WAV |
| Raw (.g726, .adpcm) | Yes | Byte offset | None | Raw bitstream |
| RTP | Yes | No | No | RFC 3551 payload |
| MP4/M4A | No | — | — | Not natively stored |
| Matroska/MKA | No | — | — | Not natively stored |
| OGG | No | — | — | Not natively stored |
| WebM | No | — | — | Not natively stored |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** RIFF INFO (in WAV) or no metadata (raw bitstream)
- **Tag block location:** INFO chunk within WAV file

### 7.2 Standard Tag Fields — RIFF INFO
| Tag Field | RIFF INFO Key | Max Length | Character Encoding | Multi-value | Notes |
|-----------|---------------|------------|-------------------|-------------|-------|
| Title | INAM | 256 | ASCII | No | Title |
| Artist | IART | 256 | ASCII | No | Creator/artist |
| Comment | ICMT | 256 | ASCII | No | Comment |
| Copyright | ICOP | 256 | ASCII | No | Copyright |

### 7.3 Cover Art Storage
Not supported in G.726 WAV files.

### 7.4 Metadata Compatibility Matrix
| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| RIFF INFO | ✓ | ✓ | ✓ | Highest (WAV native) |
| ID3v1 | ✗ | ✗ | ✗ | Not applicable |
| ID3v2 | ✗ | ✗ | ✗ | Not applicable |

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   adpcm_g726le              # used with -c:a (little-endian)
                    adpcm_g726                # big-endian
                    adpcm_ima_apm             # IMA ADPCM (different algorithm)
                    adpcm_ima_qt              # IMA QT ADPCM
                    adpcm_ima_wav             # IMA WAV ADPCM
                    adpcm_ms                  # Microsoft ADPCM
AV_CODEC_ID:        AV_CODEC_ID_ADPCM_G726    # C constant
Format Name (CLI):  g726, adpcm, wav           # file format
Encoder(s):         adpcm_g726, adpcm_g726le  # G.726 encoder
Decoder(s):         adpcm_g726, adpcm_g726le  # G.726 decoder
Muxer(s):          g726                      # G.726 bitstream muxer
Demuxer(s):        g726                      # G.726 bitstream demuxer
```

### 8.2 FFmpeg Encoding — Full CLI Reference

```bash
# Encode WAV to G.726 (32 kbps, little-endian)
ffmpeg -i input.wav \
  -c:a adpcm_g726le \
  -ar 8000 \
  -ac 1 \
  -b:a 32k \                 # Bitrate: 16k, 24k, 32k, 40k
  -frame_size 80 \           # Samples per packet (80 = 10 ms at 8 kHz)
  output.g726

# Encode to G.726 (24 kbps)
ffmpeg -i input.wav \
  -c:a adpcm_g726le \
  -ar 8000 \
  -ac 1 \
  -b:a 24k \
  output.g726

# Encode to G.726 (16 kbps)
ffmpeg -i input.wav \
  -c:a adpcm_g726le \
  -ar 8000 \
  -ac 1 \
  -b:a 16k \
  output.g726

# Encode to WAV container with Microsoft ADPCM (FFmpeg format)
# Note: Microsoft ADPCM in WAV is a different format from ITU G.726
# Use adpcm_ms for Microsoft ADPCM WAV:
ffmpeg -i input.wav \
  -c:a adpcm_ms \
  -ar 8000 \
  -ac 1 \
  output_msadpcm.wav
```

#### Complete FFmpeg Option Table
| Option | Type | Default | Range / Valid Values | Effect on Output |
|--------|------|---------|---------------------|-----------------|
| `-b:a` | int | 32k | 16k, 24k, 32k, 40k | Target bitrate |
| `-ar` | int | 8000 | 8000 | Sample rate (must be 8000) |
| `-ac` | int | 1 | 1 | Channel count (mono only) |
| `-frame_size` | int | 80 | 40, 80, 160 | Samples per packet |
| `-bit_rate` | int | 32k | 16k–40k | Alias for -b:a |

### 8.3 FFmpeg Encoding — libavcodec C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/opt.h>
#include <libswresample/swresample.h>

// ─── 1. Find encoder ─────────────────────────────────────────────────────────
const AVCodec *codec = avcodec_find_encoder(AV_CODEC_ID_ADPCM_G726);
if (!codec) { fprintf(stderr, "G.726 encoder not found\n"); exit(1); }

// ─── 2. Allocate codec context ────────────────────────────────────────────────
AVCodecContext *ctx = avcodec_alloc_context3(codec);
if (!ctx) { fprintf(stderr, "OOM\n"); exit(1); }

// ─── 3. Set required parameters ────────────────────────────────────────────────
ctx->bit_rate    = 32000;              // bits/sec (32 kbps default)
ctx->sample_fmt  = AV_SAMPLE_FMT_S16; // Input: signed 16-bit PCM
ctx->sample_rate = 8000;               // Hz — MUST be 8000 for G.726
av_channel_layout_default(&ctx->ch_layout, 1); // Mono

// G.726-specific options:
av_opt_set_int(ctx, "bit_rate", 32000, AV_OPT_SEARCH_CHILDREN);
av_opt_set_int(ctx, "frame_size", 80, AV_OPT_SEARCH_CHILDREN); // 10 ms = 80 samples

// ─── 4. Open codec ─────────────────────────────────────────────────────────
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
frame->nb_samples  = ctx->frame_size; // 80 samples (10 ms at 8 kHz)
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
- G.726 requires exactly 8000 Hz input; resample to 8000 Hz before encoding
- Input format: `AV_SAMPLE_FMT_S16` (signed 16-bit PCM)
- Output format: packed G.726 bitstream (no WAV header by default)
- To output WAV with Microsoft ADPCM, use `adpcm_ms` codec instead
- `ctx->frame_size` determines how many samples per encoded packet

### 8.4 FFmpeg Decoding — Full CLI Reference

```bash
# Decode G.726 to WAV PCM
ffmpeg -f g726 -i input.g726 \
  -c:a pcm_s16le \
  -ar 8000 \
  -ac 1 \
  output.wav

# Decode G.726 little-endian to WAV PCM
ffmpeg -f g726 -i input.g726 \
  -c:a pcm_s16le \
  output.wav

# Decode WAV with Microsoft ADPCM
ffmpeg -i input_msadpcm.wav \
  -c:a pcm_s16le \
  output.wav

# Probe format
ffprobe -v quiet -print_format json -show_streams -show_format input.g726
```

### 8.5 FFmpeg Decoding — libavcodec C API

```c
// ─── Complete decode program skeleton ────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.g726", NULL, NULL);
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
            // frm->nb_samples = samples per frame
            // frm->sample_rate = 8000
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
# Read metadata
ffprobe -v quiet -print_format json -show_format input.wav | jq .format.tags

# Write metadata
ffmpeg -i input.wav \
  -c:a copy \
  -metadata title="Voice Recording" \
  -metadata artist="Name" \
  output_tagged.wav

# Strip metadata
ffmpeg -i input.wav -c:a copy -map_metadata -1 output_clean.wav
```

### 8.7 Quality / Fidelity Decision Table
| Use Case | Settings | Expected Size | Notes |
|----------|----------|---------------|-------|
| VoIP (standard) | `-c:a adpcm_g726le -b:a 32k` | ~24 KB/min | Good voice quality |
| VoIP (low bandwidth) | `-c:a adpcm_g726le -b:a 24k` | ~18 KB/min | Fair voice quality |
| Legacy PBX | `-c:a adpcm_g726le -b:a 16k` | ~12 KB/min | Low quality |
| Data + voice (DCME) | `-c:a adpcm_g726le -b:a 40k` | ~30 KB/min | High quality |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
```
G.726 bitstream: No seek table.

Random access:
  - G.726 is a continuous bitstream — no frame headers
  - Byte offset = time_ms × (bitrate / 8000)
  - At 32 kbps: byte = time_ms × 4 bytes/ms
  - At 16 kbps: byte = time_ms × 2 bytes/ms

Seeking precision: byte-level (not sample-level)
  - Approximate precision: ±0.25 ms at 32 kbps
```

### 9.2 Gapless Playback Data
```
Encoder delay:   0 samples (no lookahead)
Padding:         0 samples
Storage:         Not applicable
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | Trivial — decode sample by sample |
| Algorithmic encoder delay | 0 ms | No lookahead |
| Live encoding feasible | Yes | Real-time ADPCM is trivial on any CPU |
| HTTP progressive download | Yes | Raw bitstream or WAV container |
| HTTP Live Streaming (HLS) | No | Not commonly used |
| DASH streaming | No | Not commonly used |
| WebRTC / RTP transport | Yes | RFC 3551 payload type dynamic |
| Minimum decode buffer | 1 packet | Typically 10 ms (80 samples) |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | Layout Name | Channel Order | FFmpeg Layout Constant | Notes |
|----------|-------------|---------------|----------------------|-------|
| 1 | Mono | C | AV_CHANNEL_LAYOUT_MONO | Standard mode |

**Note:** G.726 is strictly mono. Multi-channel requires separate G.726 streams or a different codec.

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 14-bit linear (Annex A) | |
| Max sample rate | 8000 Hz | Fixed by standard |
| Float support | No | Fixed-point only |
| DSD support | No | Not applicable |

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

| Platform / API | Encode | Decode | FFmpeg Flag | Notes |
|----------------|--------|--------|-------------|-------|
| x86/x64 (generic) | Yes | Yes | None | Table lookup / DSP |
| ARM | Yes | Yes | None | Simple operations |
| DSP chips (legacy) | Yes | Yes | None | Some telecom DSPs |
| Intel QSV | No | No | — | Not applicable |
| Apple AudioToolbox | Yes | Yes | — | Some macOS/iOS versions |
| VA-API | No | No | — | Not applicable |

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Encoder-Specific Issues
| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| G.726 encoder not available | Some | Build FFmpeg with ADPCM support |
| Wrong bitrate flag | All | Use `-b:a 32k` explicitly |
| Little-endian vs big-endian | All | Use `adpcm_g726le` for LE, `adpcm_g726` for BE |

### 14.2 Interoperability Issues
- **Bit ordering within bytes:** G.726 bit ordering within bytes varies between implementations. Some pack MSB-first, others LSB-first. FFmpeg uses `adpcm_g726le` (little-endian = LSB first) and `adpcm_g726` (big-endian = MSB first).
- **WAV Microsoft ADPCM vs ITU G.726:** These are different algorithms despite similar names. Microsoft ADPCM uses IMA-style 4-bit samples; ITU G.726 uses the G.726 adaptive quantizer. FFmpeg's `adpcm_ms` codec produces WAV files that sound different from ITU G.726.
- **A-law / μ-law input:** G.726 base specification requires A-law or μ-law PCM input (not linear). Annex A defines linear PCM I/O. FFmpeg's G.726 codec handles both internally.

### 14.3 Edge Cases to Handle in Converter
- **Empty file / 0-sample stream:** Produce empty output file
- **File < 1 byte:** Encode as partial packet
- **All-silence audio:** ADPCM of silence produces repeating patterns; decode correctly
- **Corrupt bitstream:** Decoder may produce garbage audio or crash
- **Sample rate not 8 kHz:** FFmpeg resamples to 8000 Hz

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM G.726

| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → FLAC | `ffmpeg -f g726 -i in.g726 -c:a flac -compression_level 8 out.flac` | Limited | Lossless decode |
| → WAV | `ffmpeg -f g726 -i in.g726 -c:a pcm_s16le -ar 8000 out.wav` | RIFF INFO | Lossless decode |
| → MP3 | `ffmpeg -f g726 -i in.g726 -c:a libmp3lame -q:a 2 out.mp3` | ID3 | Generation loss |
| → Opus | `ffmpeg -f g726 -i in.g726 -c:a libopus -b:a 24k out.opus` | None | Generation loss |

### 15.2 Converting TO G.726

| Source | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| WAV → | `ffmpeg -i in.wav -c:a adpcm_g726le -b:a 32k -ar 8000 out.g726` | None | Lossy |
| FLAC → | `ffmpeg -i in.flac -c:a adpcm_g726le -b:a 32k -ar 8000 out.g726` | None | Transcode |

### 15.3 Lossless Round-Trip Verification
```bash
# G.726 is lossy — true lossless round-trip is NOT possible
# Verify decode pipeline:
ffmpeg -f g726 -i original.g726 -c:a pcm_s16le decoded.wav
md5sum decoded.wav reference_raw.wav
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| ITU-T G.191 (reference) | C | ITU copyright | Reference | Reference | https://www.itu.int/rec/T-REC-G.191 |
| FFmpeg native | C | LGPL 2.1+ | Good | Good | https://ffmpeg.org |
| ITU G.726 Annex A | C | ITU copyright | Reference | Reference | Included in G.191 |

### Build Instructions
```bash
# G.726 is built into FFmpeg by default (ADPCM support)
# Verify support:
ffmpeg -encoders | grep adpcm
ffmpeg -decoders | grep adpcm

# No external dependencies needed
# FFmpeg default configuration includes ADPCM codecs
```

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **ITU-T G.726:** 40, 32, 24, 16 kbit/s Adaptive Differential Pulse Code Modulation (ADPCM) — https://www.itu.int/rec/T-REC-G.726
- **ITU-T G.191:** Software Tools Library — G.726 reference C implementation
- **RFC 3551:** RTP Profile for Audio and Video Conferences — G.726 payload type
- **ITU-T G.727:** Embedded ADPCM (related standard with variable rate)

### Technical Resources
- FFmpeg encoder options: `ffmpeg -h encoder=adpcm_g726`
- FFmpeg decoder options: `ffmpeg -h decoder=adpcm_g726`
- ITU-T G.726 page: https://www.itu.int/rec/T-REC-G.726
- Wikipedia G.726: comprehensive overview

### Academic Papers
- ITU-T, "Recommendation G.726 — ADPCM at 40, 32, 24 and 16 kbit/s," 1990
- CCITT, "Recommendation G.721 — 32 kbit/s Adaptive Differential Pulse Code Modulation," (superseded by G.726)

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] G.726 requires NO external library — built into FFmpeg by default
- [ ] Verify `ffmpeg -encoders` output confirms adpcm_g726 is available
- [ ] Verify `ffmpeg -decoders` output confirms adpcm_g726 is available
- [ ] Note: Microsoft ADPCM WAV (`adpcm_ms`) is a different codec

### Encoding Pipeline
- [ ] Convert input to S16 using libswresample
- [ ] Resample to exactly 8000 Hz
- [ ] Use `adpcm_g726le` (little-endian) or `adpcm_g726` (big-endian) as appropriate
- [ ] Set bitrate explicitly: 16k, 24k, 32k, or 40k
- [ ] Use `-f g726` format for raw output, or WAV muxer for container

### Decoding Pipeline
- [ ] Use `-f g726` format for raw bitstream input
- [ ] Handle S16 output from decoder
- [ ] Resample output if needed

### Metadata
- [ ] Read RIFF INFO tags from WAV files
- [ ] Write RIFF INFO tags to WAV output
- [ ] Raw G.726 bitstreams have no metadata

### Edge Cases
- [ ] Handle empty files
- [ ] Handle non-8kHz input (resample)
- [ ] Handle bit ordering (LE vs BE) based on file format

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
