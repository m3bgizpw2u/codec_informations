# FFmpeg Codec Encoder Options — Per-Codec Reference — Deep Technical Reference
> **Category:** FFmpeg API
> **File Extensions:** N/A (FFmpeg library)
> **MIME Types:** N/A (FFmpeg library)
> **Standardization Body:** FFmpeg Project
> **Primary Specification:** https://ffmpeg.org/ffmpeg-codecs.html
> **Patent Status:** N/A (FFmpeg library, codec implementations vary)
> **License:** LGPL 2.1+ (FFmpeg native) / Various (external libraries)
> **Current Version:** FFmpeg n8.1.1 (as of 2026)
> **Active Development:** Yes — active maintenance and releases

---

## 1. INTRODUCTION

### 1.1 Purpose of This Document

This document provides a comprehensive reference for **codec-specific encoder options** in FFmpeg. While FFmpeg provides global encoding options through `AVCodecContext` (bitrate, sample rate, channel layout, etc.), each encoder also supports **private options** specific to that codec. These options control codec-specific behavior like VBR quality, stereo modes, prediction algorithms, and psychoacoustic tuning.

This document covers:
- All native FFmpeg audio encoders (AAC, AC3, FLAC, ALAC, MP2, etc.)
- External library encoders (libmp3lame, libopus, libvorbis, libfdk_aac)
- Per-codec option tables with types, ranges, and defaults
- FFmpeg CLI syntax and C API usage examples
- Error handling and version compatibility notes

### 1.2 How to Use This Document

Each codec section follows the same structure:
```
Codec Name
├── CLI Usage
├── Option Table (sorted alphabetically)
├── FFmpeg CLI Examples
└── C API Examples
```

### 1.3 Finding Available Options

```bash
# Get encoder options for any codec
ffmpeg -h encoder=<encoder_name>

# Examples:
ffmpeg -h encoder=aac
ffmpeg -h encoder=libopus
ffmpeg -h encoder=libmp3lame

# List all audio encoders
ffmpeg -encoders 2>&1 | grep "^ A.E"

# Get help for a specific option
ffmpeg -h encoder=aac -topic encoder=aac-coder
```

---

## 2. AAC ENCODER OPTIONS

### 2.1 Native FFmpeg AAC Encoder

**Encoder Name:** `aac`  
**Codec ID:** `AV_CODEC_ID_AAC`  
**FFmpeg CLI Name:** `-c:a aac`  
**Library:** Native FFmpeg (no external dependency)  
**Sample Formats:** `fltp` (float planar)  
**Supported Sample Rates:** 96000, 88200, 64000, 48000, 44100, 32000, 24000, 22050, 16000, 12000, 11025, 8000, 7350  

#### Option Table

| Option | Type | Default | Range | Description |
|--------|------|---------|-------|-------------|
| `-aac_coder` | int | `twoloop` | `twoloop` (0), `fast` (1) | Coding algorithm. `twoloop` uses two-loop search (better quality, slower). `fast` uses faster single-pass search. |
| `-aac_ms` | boolean | `auto` | `auto`, `true`, `false` | Force M/S (mid/side) stereo coding. `auto` lets the encoder decide per-frame. |
| `-aac_is` | boolean | `true` | `true`, `false` | Enable intensity stereo coding (parametric stereo for high frequencies). |
| `-aac_pns` | boolean | `true` | `true`, `false` | Enable Perceptual Noise Substitution — replaces random spectral content with shaped noise. |
| `-aac_tns` | boolean | `true` | `true`, `false` | Enable Temporal Noise Shaping — shapes quantization noise across time. |
| `-aac_pce` | boolean | `false` | `true`, `false` | Force the use of Program Configuration Elements (PCE) for channel configurations. |

#### FFmpeg CLI Examples

```bash
# High quality AAC (default twoloop coder)
ffmpeg -i input.wav -c:a aac -b:a 256k output.m4a

# Fast encoding (uses fast coder)
ffmpeg -i input.wav -c:a aac -aac_coder fast -b:a 192k output.m4a

# Disable PNS and TNS for specific quality profile
ffmpeg -i input.wav -c:a aac -aac_pns 0 -aac_tns 0 -b:a 320k output.m4a

# Force M/S stereo
ffmpeg -i input.wav -c:a aac -aac_ms 1 -b:a 192k output.m4a
```

#### C API Examples

```c
#include <libavcodec/avcodec.h>
#include <libavutil/opt.h>

const AVCodec *codec = avcodec_find_encoder(AV_CODEC_ID_AAC);
AVCodecContext *ctx = avcodec_alloc_context3(codec);

ctx->bit_rate = 256000;
ctx->sample_fmt = AV_SAMPLE_FMT_FLTP;
ctx->sample_rate = 48000;
av_channel_layout_default(&ctx->ch_layout, 2);

// Set AAC-specific options
av_opt_set_int(ctx, "aac_coder", 0, AV_OPT_SEARCH_CHILDREN);  // twoloop (default)
av_opt_set_int(ctx, "aac_ms", 0, AV_OPT_SEARCH_CHILDREN);      // auto
av_opt_set_int(ctx, "aac_is", 1, AV_OPT_SEARCH_CHILDREN);      // enabled (default)
av_opt_set_int(ctx, "aac_pns", 1, AV_OPT_SEARCH_CHILDREN);    // enabled (default)
av_opt_set_int(ctx, "aac_tns", 1, AV_OPT_SEARCH_CHILDREN);    // enabled (default)
av_opt_set_int(ctx, "aac_pce", 0, AV_OPT_SEARCH_CHILDREN);    // disabled (default)

avcodec_open2(ctx, codec, NULL);
```

---

## 3. LIBOPUS ENCODER OPTIONS

### 3.1 libopus (Opus Interactive Audio Codec)

**Encoder Name:** `libopus`  
**Codec ID:** `AV_CODEC_ID_OPUS`  
**FFmpeg CLI Name:** `-c:a libopus`  
**Library:** libopus (https://opus-codec.org)  
**Sample Formats:** `s16`, `flt` (signed 16-bit or float)  
**Supported Sample Rates:** 48000, 24000, 16000, 12000, 8000  
**Channels:** 1–255 (mono to multichannel)  

#### Option Table

| Option | Type | Default | Range | Description |
|--------|------|---------|-------|-------------|
| `-application` | int | `audio` (2049) | `voip` (2048), `audio` (2049), `lowdelay` (2051) | Intended application type. `voip` optimizes for speech intelligibility. `audio` optimizes for general audio fidelity. `lowdelay` minimizes algorithmic delay for real-time apps. |
| `-b:a` | int | 0 (auto) | 0, 6000–510000 | Target bitrate in bits/second. 0 = automatic VBR based on quality. Common values: 64000, 96000, 128000, 192000, 256000. |
| `-vbr` | int | `on` (1) | `off` (0), `on` (1), `constrained` (2) | VBR mode. `off` = constant bitrate (CBR). `on` = variable bitrate. `constrained` = constrained VBR (bitrate stays within ±10% of target). |
| `-compression_level` | int | N/A | N/A | **Note:** libopus uses application type, not compression level. Not applicable. |
| `-frame_duration` | float | `20.0` | 2.5–120.0 | Duration of each frame in milliseconds. Default 20ms. Can be 2.5, 5, 10, 20, 40, 60, 80, 100, or 120. Smaller = lower latency. |
| `-packet_loss` | int | `0` | 0–100 | Expected packet loss percentage (0–100). Enables forward error correction (FEC) if non-zero. |
| `-fec` | boolean | `false` | `true`, `false` | Enable inband Forward Error Correction. Requires `packet_loss` to be non-zero to have effect. |
| `-mapping_family` | int | `-1` (auto) | -1–255 | Channel mapping family. `-1` = automatic. `0` = mono/stereo. `1` = surround sound (up to 8 channels). `255` = custom. |
| `-apply_phase_inv` | boolean | `true` | `true`, `false` | Apply intensity stereo phase inversion. `false` may improve quality for stereo at low bitrates but is not RFC 6716 compliant. |

#### FFmpeg CLI Examples

```bash
# Default Opus encoding (VBR, audio mode, 128kbps target)
ffmpeg -i input.wav -c:a libopus -b:a 128k output.opus

# Voice-optimized (VoIP application)
ffmpeg -i input.wav -c:a libopus -application voip -b:a 64k output.opus

# Low-latency (for real-time communication)
ffmpeg -i input.wav -c:a libopus -application lowdelay -frame_duration 10 -b:a 96k output.opus

# Music-optimized (general audio)
ffmpeg -i input.wav -c:a libopus -application audio -b:a 192k output.opus

# Stereo with FEC for lossy networks
ffmpeg -i input.wav -c:a libopus -packet_loss 10 -fec 1 -b:a 96k output.opus

# Constrained VBR (bitrate stays near target)
ffmpeg -i input.wav -c:a libopus -vbr constrained -b:a 128k output.opus
```

#### C API Examples

```c
#include <libavcodec/avcodec.h>
#include <libavutil/opt.h>

const AVCodec *codec = avcodec_find_encoder_by_name("libopus");
AVCodecContext *ctx = avcodec_alloc_context3(codec);

ctx->bit_rate = 128000;  // 128 kbps
ctx->sample_fmt = AV_SAMPLE_FMT_FLTP;
ctx->sample_rate = 48000;
av_channel_layout_default(&ctx->ch_layout, 2);

// Set Opus-specific options
av_opt_set_int(ctx, "application", 2049, AV_OPT_SEARCH_CHILDREN);  // audio
av_opt_set_int(ctx, "vbr", 1, AV_OPT_SEARCH_CHILDREN);           // VBR on
av_opt_set_double(ctx, "frame_duration", 20.0, AV_OPT_SEARCH_CHILDREN);
av_opt_set_int(ctx, "packet_loss", 0, AV_OPT_SEARCH_CHILDREN);
av_opt_set_int(ctx, "fec", 0, AV_OPT_SEARCH_CHILDREN);
av_opt_set_int(ctx, "mapping_family", -1, AV_OPT_SEARCH_CHILDREN);
av_opt_set_int(ctx, "apply_phase_inv", 1, AV_OPT_SEARCH_CHILDREN);

avcodec_open2(ctx, codec, NULL);
```

---

## 4. LIBVORBIS ENCODER OPTIONS

### 4.1 libvorbis (Vorbis Audio Codec)

**Encoder Name:** `libvorbis`  
**Codec ID:** `AV_CODEC_ID_VORBIS`  
**FFmpeg CLI Name:** `-c:a libvorbis`  
**Library:** libvorbis (Xiph.org)  
**Sample Formats:** `fltp` (float planar)  
**Supported Sample Rates:** Any ( Vorbis is sample-rate independent)  

#### Option Table

| Option | Type | Default | Range | Description |
|--------|------|---------|-------|-------------|
| `-q:a` | float | `4.0` | 0.0–10.0 | VBR quality level. Higher = better quality/larger file. 0.0 = worst, 10.0 = best. `-q:a 6` is considered transparent for most content. |
| `-b:a` | int | N/A | N/A | Vorbis does not support CBR. Use `-q:a` for VBR quality control. |
| `-minrate` | float | `0.0` | 0.0–inf | Minimum bitrate (kbps). 0 = no minimum. Rarely used. |
| `-maxrate` | float | `0.0` | 0.0–inf | Maximum bitrate (kbps). 0 = no maximum. Important for streaming with bandwidth constraints. |
| `-iblock` | float | `0.0` | -15.0–0.0 | Impulse block bias. Negative values favor impulse blocks (transients). 0 = no bias. Lower (more negative) = better for percussive music, worse for sustained tones. |
| `-aotuv` | boolean | `false` | `true`, `false` | Use AoTuV psychoacoustic improvements (from Xiph.Org's Vorbis tuning fork). Generally provides better quality, especially at lower bitrates. |

#### FFmpeg CLI Examples

```bash
# High quality Vorbis (quality 6)
ffmpeg -i input.wav -c:a libvorbis -q:a 6 output.ogg

# Medium quality (quality 4, default)
ffmpeg -i input.wav -c:a libvorbis -q:a 4 output.ogg

# Low bitrate streaming (max 128kbps)
ffmpeg -i input.wav -c:a libvorbis -q:a 3 -maxrate 128k output.ogg

# Use AoTuV psychoacoustics (recommended for better quality)
ffmpeg -i input.wav -c:a libvorbis -q:a 5 -aotuv 1 output.ogg

# Impulse block bias for percussion-heavy music
ffmpeg -i drums.wav -c:a libvorbis -q:a 5 -iblock -10 output.ogg
```

#### C API Examples

```c
#include <libavcodec/avcodec.h>
#include <libavutil/opt.h>

const AVCodec *codec = avcodec_find_encoder_by_name("libvorbis");
AVCodecContext *ctx = avcodec_alloc_context3(codec);

ctx->sample_fmt = AV_SAMPLE_FMT_FLTP;
ctx->sample_rate = 48000;
av_channel_layout_default(&ctx->ch_layout, 2);

// libvorbis uses global_quality for VBR quality
// FFmpeg maps -q:a to ctx->global_quality
av_opt_set_double(ctx, "q", 6.0, AV_OPT_SEARCH_CHILDREN);  // Quality 6

// Set other options
av_opt_set_double(ctx, "iblock", 0.0, AV_OPT_SEARCH_CHILDREN);
// Note: aotuv is a special flag, may need build-specific handling

avcodec_open2(ctx, codec, NULL);
```

---

## 5. LIBMP3LAME ENCODER OPTIONS

### 5.1 libmp3lame (LAME MP3 Encoder)

**Encoder Name:** `libmp3lame`  
**Codec ID:** `AV_CODEC_ID_MP3`  
**FFmpeg CLI Name:** `-c:a libmp3lame`  
**Library:** libmp3lame (https://lame.sourceforge.io)  
**Sample Formats:** `s32p`, `fltp`, `s16p`  
**Supported Sample Rates:** 44100, 48000, 32000, 22050, 24000, 16000, 11025, 12000, 8000  
**Supported Channel Layouts:** Mono, Stereo  

#### Option Table

| Option | Type | Default | Range | Description |
|--------|------|---------|-------|-------------|
| `-b:a` | int | 128k | 8k–320k | Bitrate for CBR/ABR modes. |
| `-q:a` | int | 2 | 0–9 | VBR quality (0=best, 9=worst). LAME's VBR quality scale. Lower = better quality. |
| `-abr` | boolean | `false` | `true`, `false` | Enable Average Bitrate mode. Combines VBR quality with bitrate targeting. |
| `-reservoir` | boolean | `true` | `true`, `false` | Enable bit reservoir. Allows temporary bitrate bursts above frame average. Required for proper LAME quality. |
| `-joint_stereo` | boolean | `true` | `true`, `false` | Enable joint stereo (M/S encoding). Better quality at low bitrates, transparent at high. |
| `-copyright` | boolean | `false` | `true`, `false` | Set copyright bit in the MP3 frame header. |
| `-original` | boolean | `true` | `true`, `false` | Set original bit. Set to 0 for transcodes, 1 for original source. |

#### FFmpeg CLI Examples

```bash
# CBR 192 kbps (high quality)
ffmpeg -i input.wav -c:a libmp3lame -b:a 192k output.mp3

# VBR quality 2 (very high quality)
ffmpeg -i input.wav -c:a libmp3lame -q:a 2 output.mp3

# ABR 192 kbps (average bitrate, targets 192k but varies)
ffmpeg -i input.wav -c:a libmp3lame -abr 1 -b:a 192k output.mp3

# Disable joint stereo (rarely needed)
ffmpeg -i input.wav -c:a libmp3lame -joint_stereo 0 -b:a 320k output.mp3

# Disable bit reservoir (for strict bitrate compliance)
ffmpeg -i input.wav -c:a libmp3lame -reservoir 0 -b:a 128k output.mp3
```

#### C API Examples

```c
#include <libavcodec/avcodec.h>
#include <libavutil/opt.h>

const AVCodec *codec = avcodec_find_encoder_by_name("libmp3lame");
AVCodecContext *ctx = avcodec_alloc_context3(codec);

ctx->bit_rate = 192000;  // 192 kbps
ctx->sample_fmt = AV_SAMPLE_FMT_FLTP;
ctx->sample_rate = 44100;
av_channel_layout_default(&ctx->ch_layout, 2);

// Set libmp3lame-specific options
av_opt_set_int(ctx, "reservoir", 1, AV_OPT_SEARCH_CHILDREN);  // enabled (default)
av_opt_set_int(ctx, "joint_stereo", 1, AV_OPT_SEARCH_CHILDREN);  // enabled (default)
av_opt_set_int(ctx, "abr", 0, AV_OPT_SEARCH_CHILDREN);           // disabled (CBR/VBR)
av_opt_set_int(ctx, "copyright", 0, AV_OPT_SEARCH_CHILDREN);
av_opt_set_int(ctx, "original", 1, AV_OPT_SEARCH_CHILDREN);

avcodec_open2(ctx, codec, NULL);
```

---

## 6. AC3 ENCODER OPTIONS

### 6.1 ATSC A/52 AC-3 Encoder

**Encoder Name:** `ac3`  
**Codec ID:** `AV_CODEC_ID_AC3`  
**FFmpeg CLI Name:** `-c:a ac3`  
**Library:** Native FFmpeg  
**Sample Formats:** `fltp` (float planar)  
**Supported Sample Rates:** 48000, 44100, 32000  
**Supported Channel Layouts:** Mono, Stereo, 3.0, 4.0, 5.0, 5.1, etc.  

#### Option Table

| Option | Type | Default | Range | Description |
|--------|------|---------|-------|-------------|
| `-b:a` | int | 128k | 32k–640k | Bitrate in bits/second. Standard rates: 64000, 96000, 128000, 192000, 224000, 256000, 320000, 384000, 448000, 512000, 576000, 640000. |
| `-ac` | int | (from input) | 1–6 | Number of output channels. |
| `-center_mixlev` | float | `0.5946` | 0.0–1.0 | Center mix level for stereo downmix. The level by which the center channel is attenuated in a Lt/Rt downmix. |
| `-surround_mixlev` | float | `0.5` | 0.0–1.0 | Surround mix level for stereo downmix. |
| `-mixing_level` | int | `-1` | -1–111 | Mixing level in dB for surround channels (default: -1 = not indicated). |
| `-room_type` | int | `-1` | -1–2 | Room type: -1=not indicated, 0=not small, 1=large, 2=small. |
| `-per_frame_metadata` | boolean | `false` | `true`, `false` | Allow changing metadata per frame. |
| `-copyright` | int | `-1` | -1–1 | Copyright bit: -1=not indicated, 0=no copyright, 1=copyright. |
| `-dialnorm` | int | `-31` | -31–-1 | Dialogue normalization in dB. Target dialogue level (typically -31 to -1 dB). |
| `-dsur_mode` | int | `-1` | -1–2 | Dolby Surround mode: -1=not indicated, 0=not Dolby Surround, 1=not Dolby Surround, 2=Dolby Surround encoded. |
| `-original` | int | `-1` | -1–1 | Original bit stream: -1=not indicated, 0=not original, 1=original. |
| `-dmix_mode` | int | `-1` | -1–3 | Preferred stereo downmix mode: -1=not indicated, 0=not indicated, 1=Lt/Rt, 2=Lo/Ro, 3=Dolby Pro Logic II. |
| `-ltrt_cmixlev` | float | `-1` | -1.0–2.0 | Lt/Rt center mix level for preferred downmix. |
| `-ltrt_surmixlev` | float | `-1` | -1.0–2.0 | Lt/Rt surround mix level for preferred downmix. |
| `-loro_cmixlev` | float | `-1` | -1.0–2.0 | Lo/Ro center mix level for preferred downmix. |
| `-loro_surmixlev` | float | `-1` | -1.0–2.0 | Lo/Ro surround mix level for preferred downmix. |
| `-dsurex_mode` | int | `-1` | -1–3 | Dolby Surround EX mode: -1=not indicated, 0=not indicated, 1=not Dolby Surround EX, 2=Dolby Surround EX encoded, 3=Dolby Pro Logic IIz-encoded. |
| `-dheadphone_mode` | int | `-1` | -1–2 | Dolby Headphone mode. |
| `-ad_conv_type` | int | `-1` | -1–1 | A/D converter type: -1=not indicated, 0=standard, 1=HDCD. |
| `-stereo_rematrixing` | boolean | `true` | `true`, `false` | Enable stereo rematrixing ( Lt/Rt ↔ Lo/Ro conversion). |
| `-channel_coupling` | int | `-1` | -1–1 | Channel coupling: -1=auto, 0=off, 1=on. |
| `-cpl_start_band` | int | `-1` | -1–15 | Coupling start band: -1=auto, 0–15=manual. |

#### FFmpeg CLI Examples

```bash
# Standard 5.1 AC-3 at 384 kbps
ffmpeg -i input.wav -c:a ac3 -b:a 384k -ac 6 output.ac3

# Stereo AC-3 at 192 kbps
ffmpeg -i input.wav -c:a ac3 -b:a 192k -ac 2 output.ac3

# With dialogue normalization (-27 dB)
ffmpeg -i input.wav -c:a ac3 -b:a 256k -dialnorm -27 output.ac3

# Dolby Pro Logic II preferred downmix
ffmpeg -i input.wav -c:a ac3 -b:a 384k -dmix_mode 3 output.ac3

# HDCD source flag
ffmpeg -i input.wav -c:a ac3 -b:a 256k -ad_conv_type 1 output.ac3
```

#### C API Examples

```c
#include <libavcodec/avcodec.h>
#include <libavutil/opt.h>

const AVCodec *codec = avcodec_find_encoder(AV_CODEC_ID_AC3);
AVCodecContext *ctx = avcodec_alloc_context3(codec);

ctx->bit_rate = 384000;
ctx->sample_fmt = AV_SAMPLE_FMT_FLTP;
ctx->sample_rate = 48000;
av_channel_layout_default(&ctx->ch_layout, 6);  // 5.1

// Set AC3-specific options
av_opt_set_double(ctx, "center_mixlev", 0.5946, AV_OPT_SEARCH_CHILDREN);
av_opt_set_double(ctx, "surround_mixlev", 0.5, AV_OPT_SEARCH_CHILDREN);
av_opt_set_int(ctx, "dialnorm", -31, AV_OPT_SEARCH_CHILDREN);
av_opt_set_int(ctx, "stereo_rematrixing", 1, AV_OPT_SEARCH_CHILDREN);

avcodec_open2(ctx, codec, NULL);
```

---

## 7. FLAC ENCODER OPTIONS

### 7.1 Free Lossless Audio Codec Encoder

**Encoder Name:** `flac`  
**Codec ID:** `AV_CODEC_ID_FLAC`  
**FFmpeg CLI Name:** `-c:a flac`  
**Library:** Native FFmpeg  
**Sample Formats:** `s16`, `s32` (signed 16/32-bit)  
**Supported Sample Rates:** All standard rates  

#### Option Table

| Option | Type | Default | Range | Description |
|--------|------|---------|-------|-------------|
| `-compression_level` | int | 5 | 0–12 | Compression level (0=fastest, 12=smallest). Higher = slower encode, better compression. Default 5. |
| `-lpc_coeff_precision` | int | `15` | 0–15 | LPC coefficient precision in bits. Higher = better prediction, slower. 0=auto. |
| `-lpc_type` | int | `-1` | -1–3 | LPC algorithm: -1=auto, 0=none (fixed), 1=fixed, 2=Levinson, 3=Cholesky. |
| `-lpc_passes` | int | `2` | 1–INT_MAX | Number of passes for Cholesky factorization (when lpc_type=3). |
| `-min_partition_order` | int | `-1` | -1–8 | Minimum Rice partition order. -1=auto. |
| `-max_partition_order` | int | `-1` | -1–8 | Maximum Rice partition order. -1=auto. |
| `-prediction_order_method` | int | `-1` | -1–5 | Search method: -1=auto, 0=estimation, 1=2level, 2=4level, 3=8level, 4=search, 5=log. |
| `-ch_mode` | int | `-1` | -1–3 | Stereo decorrelation mode: -1=auto, 0=indep, 1=left_side, 2=right_side, 3=mid_side. |
| `-exact_rice_parameters` | boolean | `false` | `true`, `false` | Calculate exact Rice parameters instead of estimating. Slower but better compression. |
| `-multi_dim_quant` | boolean | `false` | `true`, `false` | Multi-dimensional quantization. |
| `-min_prediction_order` | int | `-1` | -1–32 | Minimum LPC prediction order. -1=auto. |
| `-max_prediction_order` | int | `-1` | -1–32 | Maximum LPC prediction order. -1=auto. |

#### FFmpeg CLI Examples

```bash
# Default FLAC (compression level 5)
ffmpeg -i input.wav -c:a flac output.flac

# Maximum compression (level 12)
ffmpeg -i input.wav -c:a flac -compression_level 12 output.flac

# Fast encoding (level 0)
ffmpeg -i input.wav -c:a flac -compression_level 0 output.flac

# Exact Rice parameters (better compression, slower)
ffmpeg -i input.wav -c:a flac -exact_rice_parameters 1 output.flac

# Mid-side stereo encoding
ffmpeg -i input.wav -c:a flac -ch_mode 3 output.flac

# Force left-side stereo
ffmpeg -i input.wav -c:a flac -ch_mode 1 output.flac
```

#### C API Examples

```c
#include <libavcodec/avcodec.h>
#include <libavutil/opt.h>

const AVCodec *codec = avcodec_find_encoder(AV_CODEC_ID_FLAC);
AVCodecContext *ctx = avcodec_alloc_context3(codec);

ctx->sample_fmt = AV_SAMPLE_FMT_S32;
ctx->sample_rate = 48000;
av_channel_layout_default(&ctx->ch_layout, 2);

// Set FLAC-specific options
av_opt_set_int(ctx, "compression_level", 8, AV_OPT_SEARCH_CHILDREN);
av_opt_set_int(ctx, "lpc_coeff_precision", 15, AV_OPT_SEARCH_CHILDREN);
av_opt_set_int(ctx, "lpc_type", -1, AV_OPT_SEARCH_CHILDREN);  // auto
av_opt_set_int(ctx, "ch_mode", -1, AV_OPT_SEARCH_CHILDREN);   // auto
av_opt_set_int(ctx, "exact_rice_parameters", 0, AV_OPT_SEARCH_CHILDREN);

avcodec_open2(ctx, codec, NULL);
```

---

## 8. ALAC ENCODER OPTIONS

### 8.1 Apple Lossless Audio Codec Encoder

**Encoder Name:** `alac`  
**Codec ID:** `AV_CODEC_ID_ALAC`  
**FFmpeg CLI Name:** `-c:a alac`  
**Library:** Native FFmpeg (reverse-engineered implementation)  
**Sample Formats:** `s32p`, `s16p`  
**Supported Channel Layouts:** Mono, Stereo, 3.0, 4.0, 5.0, 5.1, 6.1, 7.1  

#### Option Table

| Option | Type | Default | Range | Description |
|--------|------|---------|-------|-------------|
| `-min_prediction_order` | int | `4` | 1–30 | Minimum LPC prediction order. Higher = better compression, slower. |
| `-max_prediction_order` | int | `6` | 1–30 | Maximum LPC prediction order. Higher = better compression, slower. |

#### FFmpeg CLI Examples

```bash
# Default ALAC encoding
ffmpeg -i input.wav -c:a alac output.m4a

# Maximum prediction order (better compression)
ffmpeg -i input.wav -c:a alac -max_prediction_order 30 output.m4a

# Fast encoding (minimal prediction)
ffmpeg -i input.wav -c:a alac -min_prediction_order 1 -max_prediction_order 4 output.m4a
```

#### C API Examples

```c
#include <libavcodec/avcodec.h>
#include <libavutil/opt.h>

const AVCodec *codec = avcodec_find_encoder(AV_CODEC_ID_ALAC);
AVCodecContext *ctx = avcodec_alloc_context3(codec);

ctx->sample_fmt = AV_SAMPLE_FMT_S32P;
ctx->sample_rate = 48000;
av_channel_layout_default(&ctx->ch_layout, 2);

// Set ALAC-specific options
av_opt_set_int(ctx, "min_prediction_order", 4, AV_OPT_SEARCH_CHILDREN);
av_opt_set_int(ctx, "max_prediction_order", 6, AV_OPT_SEARCH_CHILDREN);

avcodec_open2(ctx, codec, NULL);
```

---

## 9. WAVPACK ENCODER OPTIONS

### 9.1 WavPack Encoder

**Encoder Name:** `wavpack`  
**Codec ID:** `AV_CODEC_ID_WAVPACK`  
**FFmpeg CLI Name:** `-c:a wavpack`  
**Library:** Native FFmpeg (libwavpack integration)  
**Sample Formats:** `u8p`, `s16p`, `s32p`, `fltp`  

#### Option Table

| Option | Type | Default | Range | Description |
|--------|------|---------|-------|-------------|
| `-joint_stereo` | boolean | `auto` | `auto`, `true`, `false` | Joint stereo mode. `auto` lets encoder decide. |
| `-optimize_mono` | boolean | `false` | `true`, `false` | Optimize for mono input. |

#### FFmpeg CLI Examples

```bash
# Default WavPack encoding
ffmpeg -i input.wav -c:a wavpack output.wv

# Joint stereo enabled
ffmpeg -i input.wav -c:a wavpack -joint_stereo 1 output.wv

# Mono optimization
ffmpeg -i input.wav -c:a wavpack -optimize_mono 1 output.wv
```

#### C API Examples

```c
#include <libavcodec/avcodec.h>
#include <libavutil/opt.h>

const AVCodec *codec = avcodec_find_encoder(AV_CODEC_ID_WAVPACK);
AVCodecContext *ctx = avcodec_alloc_context3(codec);

ctx->sample_fmt = AV_SAMPLE_FMT_FLTP;
ctx->sample_rate = 48000;
av_channel_layout_default(&ctx->ch_layout, 2);

av_opt_set_int(ctx, "joint_stereo", -1, AV_OPT_SEARCH_CHILDREN);  // auto
av_opt_set_int(ctx, "optimize_mono", 0, AV_OPT_SEARCH_CHILDREN);

avcodec_open2(ctx, codec, NULL);
```

---

## 10. LIBFDK_AAC ENCODER OPTIONS

### 10.1 Fraunhofer FDK AAC Encoder

**Encoder Name:** `libfdk_aac`  
**Codec ID:** `AV_CODEC_ID_AAC`  
**FFmpeg CLI Name:** `-c:a libfdk_aac`  
**Library:** Fraunhofer FDK AAC (libfdk-aac)  
**Sample Formats:** `s16` (signed 16-bit interleaved)  
**Note:** This encoder requires FFmpeg to be built with `--enable-libfdk-aac` and `--enable-nonfree`.  

#### Option Table

| Option | Type | Default | Range | Description |
|--------|------|---------|-------|-------------|
| `-profile:a` | int | `aac_low` | `aac_low`, `aac_he`, `aac_he_v2`, `aac_ld`, `aac_eld` | AAC profile. See profiles table below. |
| `-b:a` | int | 0 | 0–512000 | Target bitrate in bits/second. 0 = VBR mode. |
| `-vbr` | int | `0` | 0–5 | VBR quality (0=off/CBR, 1=lowest, 5=highest). Only effective when `-b:a 0`. |
| `-afterburner` | int | `1` | 0–1 | Afterburner improves quality by ~10% at the cost of ~30% slower encoding. |
| `-eld_sbr` | int | `0` | 0–1 | Enable SBR (Spectral Band Replication) for ELD (Enhanced Low Delay) profile. |
| `-eld_v2` | int | `0` | 0–1 | Enable ELDv2 (LD-MPS extension for ELD stereo signals). |
| `-signaling` | int | `-1` | -1–2 | SBR/PS signaling style: -1=auto, 0=implicit, 1=explicit SBR, 2=explicit hierarchical. |
| `-latm` | int | `0` | 0–1 | Output LATM/LOAS encapsulated data instead of ADTS. |
| `-header_period` | int | `0` | 0–65535 | StreamMuxConfig and PCE repetition period in frames. |
| `-drc_profile` | int | `0` | 0–256 | DRC compression profile for AAC DRC. |
| `-drc_target_ref` | float | `0` | -31.75–0 | Expected target reference level at decoder in dB. |
| `-comp_profile` | int | `0` | 0–256 | Compression profile for AAC DRC. |
| `-comp_target_ref` | float | `0` | -31.75–0 | Compression target reference level in dB. |
| `-prog_ref` | float | `0` | -31.75–0 | Program reference level or dialog level in dB. |
| `-frame_length` | int | `-1` | -1–1024 | Desired frame length: -1=auto, 1024=long, 480=short (ELD). |

#### AAC Profiles

| Profile Name | Description | Typical Bitrate | Notes |
|--------------|-------------|-----------------|-------|
| `aac_low` | AAC-LC (Low Complexity) | 128–320 kbps | Most common, good quality |
| `aac_he` | HE-AAC (SBR) | 48–128 kbps | Bandwidth extension, good at low rates |
| `aac_he_v2` | HE-AAC v2 (PS+SBR) | 32–64 kbps | Parametric stereo, very low bitrates |
| `aac_ld` | AAC-LD (Low Delay) | 128–256 kbps | Reduced delay for real-time |
| `aac_eld` | ELD (Enhanced Low Delay) | 64–128 kbps | Lowest delay AAC profile |

#### FFmpeg CLI Examples

```bash
# AAC-LC at 256 kbps
ffmpeg -i input.wav -c:a libfdk_aac -profile:a aac_low -b:a 256k output.m4a

# HE-AAC at 64 kbps (bandwidth extension)
ffmpeg -i input.wav -c:a libfdk_aac -profile:a aac_he -b:a 64k output.m4a

# HE-AAC v2 at 32 kbps (stereo at very low bitrate)
ffmpeg -i input.wav -c:a libfdk_aac -profile:a aac_he_v2 -b:a 32k output.m4a

# VBR mode (quality-based)
ffmpeg -i input.wav -c:a libfdk_aac -vbr 4 -b:a 0 output.m4a

# Without afterburner (faster encode)
ffmpeg -i input.wav -c:a libfdk_aac -afterburner 0 -b:a 192k output.m4a

# ELD low-delay for real-time
ffmpeg -i input.wav -c:a libfdk_aac -profile:a aac_eld -b:a 96k output.m4a
```

#### C API Examples

```c
#include <libavcodec/avcodec.h>
#include <libavutil/opt.h>

const AVCodec *codec = avcodec_find_encoder_by_name("libfdk_aac");
AVCodecContext *ctx = avcodec_alloc_context3(codec);

ctx->bit_rate = 256000;
ctx->sample_fmt = AV_SAMPLE_FMT_S16;
ctx->sample_rate = 48000;
av_channel_layout_default(&ctx->ch_layout, 2);

// Set libfdk_aac-specific options
av_opt_set_int(ctx, "profile", 0, AV_OPT_SEARCH_CHILDREN);  // aac_low
av_opt_set_int(ctx, "afterburner", 1, AV_OPT_SEARCH_CHILDREN);
av_opt_set_int(ctx, "signaling", -1, AV_OPT_SEARCH_CHILDREN);  // auto
av_opt_set_int(ctx, "vbr", 0, AV_OPT_SEARCH_CHILDREN);
av_opt_set_int(ctx, "latm", 0, AV_OPT_SEARCH_CHILDREN);
av_opt_set_double(ctx, "drc_target_ref", 0, AV_OPT_SEARCH_CHILDREN);

avcodec_open2(ctx, codec, NULL);
```

---

## 11. G.722 ENCODER OPTIONS

### 11.1 G.722 ADPCM Encoder

**Encoder Name:** `g722`  
**Codec ID:** `AV_CODEC_ID_ADPCM_G722`  
**FFmpeg CLI Name:** `-c:a g722`  
**Library:** Native FFmpeg  
**Sample Format:** `s16` (signed 16-bit)  
**Channels:** Mono only  

#### Option Table

| Option | Type | Default | Range | Description |
|--------|------|---------|-------|-------------|
| `-bits_per_codeword` | int | N/A | N/A | **Note:** G.722 uses a fixed rate. This option is not applicable. |

#### FFmpeg CLI Examples

```bash
# Basic G.722 encoding (64 kbps wideband)
ffmpeg -i input.wav -c:a g722 output.g722

# G.722 with custom sample rate
ffmpeg -i input.wav -ar 16000 -c:a g722 output.g722
```

---

## 12. ADPCM ENCODER OPTIONS

### 12.1 ADPCM Variants

FFmpeg supports several ADPCM encoders. These are legacy codecs primarily used for compatibility.

| Encoder Name | Codec ID | Description |
|--------------|-----------|-------------|
| `adpcm_ima_qt` | AV_CODEC_ID_ADPCM_IMA_QT | Apple QuickTime ADPCM |
| `adpcm_ima_wav` | AV_CODEC_ID_ADPCM_IMA_WAV | IMA ADPCM (WAV format) |
| `adpcm_ima_ws` | AV_CODEC_ID_ADPCM_IMA_WS | Westwood ADPCM |
| `adpcm_ima_ssi` | AV_CODEC_ID_ADPCM_IMA_SSI | ADPCM (Sims) |
| `adpcm_ms` | AV_CODEC_ID_ADPCM_MS | Microsoft ADPCM |
| `adpcm_swf` | AV_CODEC_ID_ADPCM_SWF | Flash ADPCM |
| `adpcm_yamaha` | AV_CODEC_ID_ADPCM_YAMAHA | Yamaha ADPCM |

#### Option Table

| Option | Type | Default | Range | Description |
|--------|------|---------|-------|-------------|
| `-audio_frames_per_packet` | int | 0 | 0–1 | Number of audio frames per packet. 0 = 1 frame per packet. |

#### FFmpeg CLI Examples

```bash
# IMA ADPCM (WAV format)
ffmpeg -i input.wav -c:a adpcm_ima_wav output.wav

# Microsoft ADPCM
ffmpeg -i input.wav -c:a adpcm_ms output.wav
```

---

## 13. PCM ENCODER OPTIONS

### 13.1 PCM Encoders

PCM encoders write raw uncompressed audio. FFmpeg supports many PCM variants:

| Encoder Name | Format | Bits | Channels | Planar |
|--------------|--------|------|----------|--------|
| `pcm_s16le` | Signed 16-bit LE | 16 | Any | No |
| `pcm_s16be` | Signed 16-bit BE | 16 | Any | No |
| `pcm_s32le` | Signed 32-bit LE | 32 | Any | No |
| `pcm_s32be` | Signed 32-bit BE | 32 | Any | No |
| `pcm_f32le` | Float 32-bit LE | 32 | Any | No |
| `pcm_f32be` | Float 32-bit BE | 32 | Any | No |
| `pcm_f64le` | Double 64-bit LE | 64 | Any | No |
| `pcm_f64be` | Double 64-bit BE | 64 | Any | No |
| `pcm_s16pl` | Signed 16-bit Planar | 16 | Any | Yes |

#### Option Table

PCM encoders have no codec-specific options.

#### FFmpeg CLI Examples

```bash
# Signed 16-bit little-endian PCM
ffmpeg -i input.wav -c:a pcm_s16le output.pcm

# Signed 32-bit little-endian PCM
ffmpeg -i input.wav -c:a pcm_s32le output.pcm

# Float 32-bit little-endian PCM
ffmpeg -i input.wav -c:a pcm_f32le output.pcm
```

---

## 14. GLOBAL CODEC OPTIONS (AVCODECCONTEXT)

### 14.1 Options Available for All Encoders

These options are set on `AVCodecContext` and apply to all encoders:

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `bit_rate` | int64 | 0 | Target bitrate in bits/second (CBR/ABR modes). |
| `bit_rate_tolerance` | int | 0 | Allowed bitrate variance. |
| `global_quality` | int | 0 | Quality for VBR encoders (mapped from `-q:a`). Higher = better quality. |
| `compression_level` | int | -1 | Encoding speed vs. compression ratio. Higher = slower, better. |
| `sample_fmt` | enum | first | Output sample format. Must be in `codec->sample_fmts`. |
| `sample_rate` | int | 0 | Output sample rate in Hz. Must be in `codec->supported_samplerates`. |
| `ch_layout` | struct | default | Output channel layout. Must be in `codec->ch_layouts`. |
| `frame_size` | int | 0 | Frame size in samples (read-only, set by encoder). |
| `max_bit_rate` | int64 | 0 | Maximum bitrate (for VBR). |
| `rc_buffer_size` | int | 0 | Rate control buffer size. |
| `rc_initial_buffer_occupancy` | int | 0 | Initial buffer fullness. |
| `strict_std_compliance` | int | 0 | Strictness level (FF_COMPLIANCE_*). |

### 14.2 Setting Global Options

```c
#include <libavcodec/avcodec.h>
#include <libavutil/opt.h>

AVCodecContext *ctx = avcodec_alloc_context3(codec);

// Method 1: Direct assignment
ctx->bit_rate = 192000;
ctx->sample_fmt = AV_SAMPLE_FMT_FLTP;
ctx->sample_rate = 48000;

// Method 2: AVOptions API
av_opt_set_int(ctx, "b", 192000, AV_OPT_SEARCH_CHILDREN);
av_opt_set_int(ctx, "ar", 48000, AV_OPT_SEARCH_CHILDREN);
av_opt_set_int(ctx, "compression_level", 5, AV_OPT_SEARCH_CHILDREN);
```

---

## 15. OPTION INSPECTION AND DISCOVERY

### 15.1 Querying Available Options at Runtime

```c
#include <libavutil/opt.h>

// Get first option from codec context
const AVOption *opt = NULL;
while ((opt = av_opt_next(ctx, opt))) {
    printf("Option: %s (%s), type=%d, default=%lld\n",
           opt->name, opt->help, opt->type, opt->default_val.i64);
}

// Get option by name
const AVOption *found = av_opt_find(ctx, "aac_coder", NULL, 0, AV_OPT_SEARCH_CHILDREN);

// Query option ranges
int ret;
AVOptionRange **ranges;
ret = av_opt_query_ranges(&ranges, ctx, "bit_rate", AV_OPT_SEARCH_CHILDREN);
if (ret >= 0) {
    printf("Bit rate range: %f to %f\n",
           ranges[0]->range_min, ranges[0]->range_max);
}
```

### 15.2 FFmpeg CLI for Option Discovery

```bash
# Get all options for an encoder
ffmpeg -h encoder=aac full

# Get specific option help
ffmpeg -h encoder=aac -topic encoder=aac-coder

# List all encoders
ffmpeg -encoders 2>&1 | grep "^ A.E"

# List all audio encoders
ffmpeg -encoders 2>&1 | grep "^ A.E" | grep "A"

# Get encoder capabilities
ffmpeg -encoders 2>&1 | grep -A 5 "aac "
```

---

## 16. VERSION COMPATIBILITY

### 16.1 Option Changes by FFmpeg Version

| Version | Changes |
|---------|---------|
| FFmpeg 4.0 | Removed libavresample, consolidated into libswresample |
| FFmpeg 4.2 | Added `aac_pce` option for AAC encoder |
| FFmpeg 4.3 | Improved AAC encoder quality |
| FFmpeg 5.0 | New channel layout API (AVChannelLayout) |
| FFmpeg 5.1 | Updated libopus integration |
| FFmpeg 6.0 | Filter graph segment API (avfilter_graph_segment_*) |
| FFmpeg 7.0 | libswresample improvements |

### 16.2 Checking FFmpeg Version at Compile Time

```c
#include <libavutil/avutil.h>

printf("FFmpeg version: %s\n", av_version_info());
printf("avutil version: %d\n", LIBAVUTIL_VERSION_INT);

// Check for specific capabilities
#if LIBAVCODEC_VERSION_INT >= AV_VERSION_INT(60, 0, 0)
    // New codec features available
#endif
```

---

## 17. ERROR HANDLING

### 17.1 Common Error Codes

```c
#include <libavutil/error.h>

// Open encoder failed
int ret = avcodec_open2(ctx, codec, &opts);
if (ret < 0) {
    char errbuf[128];
    av_strerror(ret, errbuf, sizeof(errbuf));
    
    if (ret == AVERROR(EINVAL)) {
        fprintf(stderr, "Invalid codec context parameters: %s\n", errbuf);
        // Check sample_fmt, sample_rate, ch_layout
    } else if (ret == AVERROR_EOF) {
        fprintf(stderr, "End of file during codec open\n");
    } else {
        fprintf(stderr, "Failed to open encoder: %s\n", errbuf);
    }
}

// Encoding error
ret = avcodec_send_frame(ctx, frame);
if (ret < 0) {
    if (ret == AVERROR(EINVAL)) {
        fprintf(stderr, "Encoder not opened or invalid frame\n");
    } else if (ret == AVERROR(ENOMEM)) {
        fprintf(stderr, "Out of memory for encoder buffers\n");
    }
}
```

### 17.2 Debugging Option Problems

```bash
# Verbose encoder output
ffmpeg -loglevel debug -i input.wav -c:a aac -b:a 256k output.m4a 2>&1 | grep -E "aac|encoder|options"

# Check what options are actually set
ffmpeg -i input.wav -c:a aac -b:a 256k -aac_coder fast -f md5 - 2>&1

# Test with minimal options
ffmpeg -i input.wav -c:a aac output.m4a
```

---

## 18. QUICK REFERENCE TABLES

### 18.1 Codec to Format Mapping

| Codec | CLI Name | Container | Sample Format | Notes |
|-------|----------|-----------|---------------|-------|
| AAC | `aac` | M4A, ADTS | `fltp` | Native FFmpeg |
| AAC (FDK) | `libfdk_aac` | M4A | `s16` | Proprietary |
| Opus | `libopus` | OPUS, OGG | `s16`, `flt` | RFC 6716 |
| Vorbis | `libvorbis` | OGG | `fltp` | OGG container |
| MP3 | `libmp3lame` | MP3 | `s32p`, `fltp`, `s16p` | LAME engine |
| AC3 | `ac3` | AC3, EAC3 | `fltp` | Dolby |
| EAC3 | `eac3` | EAC3, MATROSKA | `fltp` | Dolby Digital+ |
| FLAC | `flac` | FLAC, OGG | `s16`, `s32` | Lossless |
| ALAC | `alac` | M4A | `s32p`, `s16p` | Apple Lossless |
| WavPack | `wavpack` | WV | `u8p`, `s16p`, `s32p`, `fltp` | Hybrid lossless |
| G.722 | `g722` | G722 | `s16` | Wideband ADPCM |
| PCM | `pcm_*` | WAV, AU | Various | Uncompressed |

### 18.2 Recommended Settings by Use Case

| Use Case | Recommended Codec | Settings |
|----------|------------------|----------|
| Maximum compatibility | `libmp3lame` | `-b:a 192k` |
| High quality music | `libopus` | `-c:a libopus -b:a 192k -application audio` |
| Streaming (mid-bitrate) | `aac` | `-c:a aac -b:a 128k -aac_coder fast` |
| Lossless archival | `flac` | `-c:a flac -compression_level 8` |
| Apple ecosystem | `libfdk_aac` | `-c:a libfdk_aac -profile:a aac_low -b:a 256k` |
| Very low bitrate speech | `libopus` | `-c:a libopus -b:a 32k -application voip` |
| Broadcast | `ac3` | `-c:a ac3 -b:a 384k -ac 6` |

### 18.3 Sample Format Requirements by Encoder

| Encoder | Required Format | Notes |
|---------|-----------------|-------|
| `aac` | `fltp` | Planar float only |
| `libopus` | `s16` or `flt` | Interleaved only |
| `libvorbis` | `fltp` | Planar float only |
| `libmp3lame` | `s32p`, `fltp`, `s16p` | Planar formats |
| `ac3` | `fltp` | Planar float only |
| `flac` | `s16`, `s32` | Integer formats |
| `alac` | `s32p`, `s16p` | Planar integer |
| `libfdk_aac` | `s16` | Signed 16-bit interleaved |

### 18.4 Bitrate Reference Table

Common bitrate recommendations for various audio encoders, based on typical use cases:

| Encoder | Low Quality | Standard | High Quality | Maximum |
|---------|-------------|----------|--------------|---------|
| `aac` | 64 kbps | 128 kbps | 256 kbps | 320 kbps |
| `libopus` | 32 kbps | 96 kbps | 192 kbps | 510 kbps |
| `libvorbis` | 96 kbps | 160 kbps | 256 kbps | 500 kbps |
| `libmp3lame` | 128 kbps | 192 kbps | 320 kbps | 320 kbps |
| `ac3` | 192 kbps | 384 kbps | 640 kbps | 640 kbps |
| `flac` | N/A (lossless) | N/A | N/A | N/A |
| `alac` | N/A (lossless) | N/A | N/A | N/A |

### 18.5 VBR Quality Scales by Encoder

Different encoders use different scales for VBR quality settings:

| Encoder | Quality Option | Range | Direction | Notes |
|---------|---------------|-------|-----------|-------|
| `aac` | `-q:a` (via global_quality) | 10–500 | Higher = better | Maps to bitrate internally |
| `libopus` | `-b:a` with VBR | 6000–510000 | Higher = better | Bitrate in bps |
| `libvorbis` | `-q:a` | 0–10 | Higher = better | Logarithmic scale |
| `libmp3lame` | `-q:a` | 0–9 | Lower = better | 0 = best quality |

---

## 19. ADVANCED TOPICS

### 19.1 Encoding Quality Optimization Strategies

Optimizing encoder quality requires understanding how each codec makes tradeoffs:

```
Quality Optimization Strategies:
─────────────────────────────────────────────────────────────────────────

1. AAC (Native) Quality Optimization:
   • Use twoloop coder (-aac_coder twoloop) for best quality
   • Enable all psychoacoustic tools: TNS, PNS, IS
   • Use higher bitrates for transient content
   • Avoid very low bitrates (below 64 kbps) for music

2. Opus Quality Optimization:
   • Use -application audio for music content
   • Use -application voip for speech content
   • Consider frame_duration based on latency requirements:
     - 20ms: Standard quality
     - 10ms: Lower latency, slightly lower quality
     - 5ms: Very low latency (real-time comms)
   • Constrained VBR (-vbr constrained) for streaming

3. Vorbis Quality Optimization:
   • Use -aotuv 1 for improved psychoacoustics
   • Quality 5-6 is generally transparent
   • Avoid maxrate limiting when possible (causes quality degradation)

4. MP3 Quality Optimization:
   • Use VBR (-q:a) instead of CBR for better quality/size ratio
   • Quality 2 (VBR) ≈ 190-250 kbps, transparent for most content
   • Keep bit reservoir enabled (default)
   • joint_stereo typically improves quality at all bitrates

5. AC3 Quality Optimization:
   • Higher bitrate = more bit budget for complex audio
   • channel_coupling typically improves efficiency
   • stereo_rematrixing helps stereo encoding
   • per_frame_metadata allows dynamic range control
```

### 19.2 Real-Time Streaming Considerations

For live streaming applications, specific encoder settings optimize for low latency and consistent quality:

```
Real-Time Streaming Optimization:
─────────────────────────────────────────────────────────────────────────

1. Latency Considerations:
   • AAC: Default frame size ~1024 samples (21ms at 48kHz)
     - Use shorter frames if lower latency needed
   • Opus: Frame sizes 2.5ms to 60ms
     - 20ms is default, good balance of quality/latency
     - 2.5-5ms for very low latency applications
   • Vorbis: No fixed frame concept, VBR optimization
   • MP3: 1152 samples per frame (~26ms at 44.1kHz)

2. Bitrate Control for Streaming:
   • CBR (-b:a) provides consistent bandwidth
   • VBR provides better quality but variable bandwidth
   • Constrained VBR balances both (Opus -vbr constrained)

3. Network-Friendly Settings:
   • Opus: Enable FEC for lossy networks (-fec 1 -packet_loss N)
   • AAC: Use ADTS header for compatibility
   • MP3: Larger frames reduce header overhead

4. Recommended Streaming Settings:
   Opus Voice:
     ffmpeg -i input -c:a libopus -b:a 64k -vbr on -application voip -frame_duration 20
   
   Opus Music:
     ffmpeg -i input -c:a libopus -b:a 128k -vbr on -application audio
   
   AAC Standard:
     ffmpeg -i input -c:a aac -b:a 128k -aac_coder fast
   
   MP3:
     ffmpeg -i input -c:a libmp3lame -b:a 192k -q:a 2
```

### 19.3 Hardware Acceleration Support

FFmpeg can leverage hardware encoders where available:

```
Hardware Acceleration Support:
─────────────────────────────────────────────────────────────────────────

1. Platform-Specific Hardware Encoders:
   • NVIDIA NVENC: Not available for audio codecs (NVENC is video-only)
   • Intel QSV: Limited audio encoding support
   • AMD VCE: Not available for audio codecs
   • Apple VideoToolbox: Native AAC encoding on macOS/iOS
   • Android MediaCodec: Native AAC, Opus encoding

2. FFmpeg Hardware Audio Encoding:
   • Hardware encoders typically use platform-specific APIs
   • Quality may differ from software encoders
   • Hardware is usually faster but may have limitations:
     - Fewer codec options
     - Fixed quality profiles
     - Platform-specific implementations

3. Checking Hardware Encoder Availability:
   ffmpeg -encoders 2>&1 | grep -E "hw|vaapi|qsv|cuda|videotoolbox"

4. Using Hardware Encoders:
   Apple VideoToolbox:
     ffmpeg -i input -c:a aac_at -b:a 256k output.m4a
   
   VA-API (Linux):
     ffmpeg -i input -c:a aac -hwaccel vaapi -b:a 256k output.m4a
   
   QSV (Intel):
     ffmpeg -i input -c:a aac -qsv -b:a 256k output.m4a
```

### 19.4 Encoding for Specific Output Formats

Different output formats require specific encoding considerations:

```
Output Format Encoding Requirements:
─────────────────────────────────────────────────────────────────────────

1. MP4/M4A (AAC):
   • Use `-c:a aac` or `-c:a libfdk_aac`
   • Container auto-selected based on extension (.m4a)
   • Can force container: -f mp4
   • Metadata preserved through tag writing
   • Gapless playback supported with correct container flags

2. OGG (Vorbis/Opus):
   • Use `-c:a libvorbis` or `-c:a libopus`
   • Extension: .ogg for Vorbis, .opus or .ogg for Opus
   • Vorbis Comments used for metadata
   • Streaming-friendly with seeking support

3. MKV/Matroska (Any Codec):
   • Use `-c:a aac`, `-c:a libopus`, etc.
   • Extension: .mkv or .mka for audio-only
   • Most flexible container (supports virtually all codecs)
   • Metadata preserved through Matroska tags

4. WebM (Opus/VP8/VP9):
   • Only VP8/VP9 video and Opus audio are web-compatible
   • Use `-c:a libopus` (not Vorbis)
   • Extension: .webm
   • Streaming-optimized container

5. 3GP (AMR/AAC):
   • Use `-c:a aac` or `-c:a amr_nb`
   • Designed for mobile streaming
   • Limited metadata support
   • Low overhead

6. WAV (PCM):
   • No encoding, just format conversion
   • Use `-c:a pcm_s16le` or similar
   • No compression
   • Large file sizes
```

---

## 20. SPECIFICATIONS AND FURTHER READING

### Primary Documentation
- FFmpeg Codecs: https://ffmpeg.org/ffmpeg-codecs.html
- AAC Encoder: https://ffmpeg.org/ffmpeg-codecs.html#aac
- libopus: https://ffmpeg.org/ffmpeg-codecs.html#libopus
- libvorbis: https://ffmpeg.org/ffmpeg-codecs.html#libvorbis
- libmp3lame: https://ffmpeg.org/ffmpeg-codecs.html#libmp3lame

### Opus Specification
- RFC 6716: Opus Interactive Audio Codec

### AAC Specifications
- ISO/IEC 13818-7: AAC encoding
- 3GPP TS 26.403: AAC encoder

### LAME / MP3
- LAME Project: https://lame.sourceforge.io

### Vorbis
- Xiph.Org Vorbis: https://xiph.org/vorbis/

---

## 20. IMPLEMENTATION CHECKLIST

### Encoder Selection
- [ ] Determine audio content type (speech, music, mixed)
- [ ] Determine target bitrate or quality level
- [ ] Check container compatibility (M4A, OGG, MKV, etc.)
- [ ] Verify FFmpeg build has required encoder (`ffmpeg -encoders`)
- [ ] Consider patent/licensing requirements (libfdk_aac requires nonfree)

### Context Setup
- [ ] Find encoder with `avcodec_find_encoder()` or `avcodec_find_encoder_by_name()`
- [ ] Verify encoder is not NULL (check availability)
- [ ] Set required parameters: `bit_rate`, `sample_fmt`, `sample_rate`, `ch_layout`
- [ ] Set codec-specific private options
- [ ] Call `avcodec_open2()` and check for errors

### Sample Format Conversion
- [ ] Use `libswresample` to convert input to encoder's required format
- [ ] Check `codec->sample_fmts[]` for supported formats
- [ ] Convert interleaved to planar if needed (or vice versa)

### Encoding Loop
- [ ] Prepare frames with `nb_samples = ctx->frame_size` (or appropriate size)
- [ ] Call `avcodec_send_frame()` in a loop
- [ ] Handle `AVERROR(EAGAIN)` (output buffer full)
- [ ] Call `avcodec_receive_packet()` in a loop
- [ ] Handle `AVERROR_EOF` at end of stream
- [ ] Flush encoder with `avcodec_send_frame(ctx, NULL)`

### Error Handling
- [ ] Check `avcodec_open2()` return value
- [ ] Check `avcodec_send_frame()` return values
- [ ] Check `avcodec_receive_packet()` return values
- [ ] Handle `AVERROR(EINVAL)` for invalid configurations
- [ ] Handle `AVERROR(EAGAIN)` correctly (not an error)

### Cleanup
- [ ] Call `avcodec_free_context()` to free codec context
- [ ] Unref frames with `av_frame_unref()`
- [ ] Free packets with `av_packet_unref()`
- [ ] Free `SwrContext` if using resampling

---

## 21. ADVANCED TOPICS

### 21.1 Batch Processing and Pipeline Design

For high-throughput audio processing applications, efficient pipeline design is essential:

```c
// ─── Batch Processing Architecture ────────────────────────────────────────────

typedef struct {
    AVFormatContext *fmt_ctx;
    AVCodecContext *dec_ctx;
    AVCodecContext *enc_ctx;
    SwrContext *swr;
    AVPacket *pkt;
    AVFrame *dec_frame;
    AVFrame *enc_frame;
    int stream_idx;
} AudioPipeline;

typedef struct {
    AudioPipeline *pipelines;
    int num_pipelines;
    int current_idx;
} BatchProcessor;

// Initialize batch processor with multiple pipelines
int batch_processor_init(BatchProcessor *bp, int num_threads) {
    bp->pipelines = av_mallocz_array(num_threads, sizeof(AudioPipeline));
    bp->num_pipelines = num_threads;
    
    for (int i = 0; i < num_threads; i++) {
        // Initialize each pipeline independently
        init_pipeline(&bp->pipelines[i], i);
    }
    
    return 0;
}

// Process multiple files in parallel
int batch_process_files(BatchProcessor *bp, const char **files, int count) {
    #pragma omp parallel for
    for (int i = 0; i < count; i++) {
        int thread_id = omp_get_thread_num();
        AudioPipeline *p = &bp->pipelines[thread_id % bp->num_pipelines];
        
        process_file(p, files[i]);
    }
    
    return 0;
}
```

### 21.2 Dynamic Bitrate Adaptation

For adaptive streaming, dynamic bitrate adaptation based on network conditions:

```c
// ─── Adaptive Bitrate Controller ───────────────────────────────────────────────

typedef struct {
    int current_bitrate;
    int target_bitrate;
    int min_bitrate;
    int max_bitrate;
    double congestion_level;
    int frames_since_change;
    int frames_per_change;
} BitrateController;

void bitrate_controller_init(BitrateController *bc,
                            int initial_bitrate,
                            int min_bitrate,
                            int max_bitrate) {
    bc->current_bitrate = initial_bitrate;
    bc->target_bitrate = initial_bitrate;
    bc->min_bitrate = min_bitrate;
    bc->max_bitrate = max_bitrate;
    bc->congestion_level = 0.0;
    bc->frames_since_change = 0;
    bc->frames_per_change = 50;  // Change bitrate every 50 frames
}

void bitrate_controller_update(BitrateController *bc, double congestion_factor) {
    bc->congestion_level = congestion_factor;
    bc->frames_since_change++;
    
    if (bc->frames_since_change >= bc->frames_per_change) {
        bc->frames_since_change = 0;
        
        // Adjust bitrate based on congestion
        if (congestion_factor > 0.7) {
            // High congestion, reduce bitrate
            bc->target_bitrate = bc->current_bitrate * 0.8;
        } else if (congestion_factor < 0.3) {
            // Low congestion, can increase bitrate
            bc->target_bitrate = bc->current_bitrate * 1.2;
        }
        
        // Clamp to valid range
        bc->target_bitrate = av_clip(bc->target_bitrate,
                                    bc->min_bitrate,
                                    bc->max_bitrate);
        
        // Smooth transition
        bc->current_bitrate = (bc->current_bitrate + bc->target_bitrate) / 2;
    }
}

void bitrate_controller_apply(BitrateController *bc, AVCodecContext *ctx) {
    if (ctx->bit_rate != bc->current_bitrate) {
        ctx->bit_rate = bc->current_bitrate;
    }
}
```

### 21.3 Error Resilience and Recovery

Robust error handling for streaming applications:

```c
// ─── Error Resilience Strategies ───────────────────────────────────────────────

typedef struct {
    int corrupt_frames;
    int total_frames;
    int concealment_frames;
    int resync_attempts;
} EncoderStats;

void encoder_stats_init(EncoderStats *stats) {
    memset(stats, 0, sizeof(*stats));
}

void encoder_stats_update(EncoderStats *stats, int frame_corrupt, int concealment) {
    stats->total_frames++;
    if (frame_corrupt) {
        stats->corrupt_frames++;
        stats->resync_attempts++;
    }
    if (concealment) {
        stats->concealment_frames++;
    }
}

// Recovery strategies
typedef enum {
    RECOVERY_NONE,
    RECOVERY_REPEAT_FRAME,
    RECOVERY_INTERPOLATE,
    RECOVERY_SILENCE,
    RECOVERY_RESYNC
} RecoveryStrategy;

RecoveryStrategy select_recovery_strategy(EncoderStats *stats) {
    double corrupt_rate = (double)stats->corrupt_frames / stats->total_frames;
    
    if (corrupt_rate < 0.01) {
        return RECOVERY_REPEAT_FRAME;
    } else if (corrupt_rate < 0.05) {
        return RECOVERY_INTERPOLATE;
    } else if (corrupt_rate < 0.1) {
        return RECOVERY_SILENCE;
    } else {
        return RECOVERY_RESYNC;
    }
}
```

### 21.4 Profiling and Performance Tuning

```c
// ─── Performance Profiling ───────────────────────────────────────────────────

#include <time.h>
#include <sys/time.h>

typedef struct {
    const char *name;
    double total_time;
    int count;
    double min_time;
    double max_time;
} ProfileEntry;

typedef struct {
    ProfileEntry *entries;
    int num_entries;
    int capacity;
} ProfileContext;

void profile_init(ProfileContext *ctx, int capacity) {
    ctx->entries = av_mallocz_array(capacity, sizeof(ProfileEntry));
    ctx->num_entries = 0;
    ctx->capacity = capacity;
}

void profile_start(ProfileContext *ctx, const char *name, double *start_time) {
    struct timeval tv;
    gettimeofday(&tv, NULL);
    *start_time = tv.tv_sec + tv.tv_usec / 1000000.0;
}

void profile_end(ProfileContext *ctx, const char *name, double start_time) {
    struct timeval tv;
    gettimeofday(&tv, NULL);
    double end_time = tv.tv_sec + tv.tv_usec / 1000000.0;
    double elapsed = end_time - start_time;
    
    // Find or create entry
    ProfileEntry *entry = NULL;
    for (int i = 0; i < ctx->num_entries; i++) {
        if (strcmp(ctx->entries[i].name, name) == 0) {
            entry = &ctx->entries[i];
            break;
        }
    }
    
    if (!entry && ctx->num_entries < ctx->capacity) {
        entry = &ctx->entries[ctx->num_entries++];
        entry->name = name;
        entry->total_time = 0;
        entry->count = 0;
        entry->min_time = DBL_MAX;
        entry->max_time = 0;
    }
    
    if (entry) {
        entry->total_time += elapsed;
        entry->count++;
        entry->min_time = FMIN(entry->min_time, elapsed);
        entry->max_time = FMAX(entry->max_time, elapsed);
    }
}

void profile_print(ProfileContext *ctx) {
    printf("\nPerformance Profile:\n");
    printf("%-30s %12s %10s %12s %12s\n", "Operation", "Total (ms)", "Count", "Avg (ms)", "Min-Max (ms)");
    printf("%-30s %12s %10s %12s %12s\n", "---------", "-----------", "------", "--------", "----------");
    
    for (int i = 0; i < ctx->num_entries; i++) {
        ProfileEntry *e = &ctx->entries[i];
        double avg = e->total_time / e->count * 1000.0;
        printf("%-30s %12.2f %10d %12.2f %12.2f\n",
               e->name,
               e->total_time * 1000.0,
               e->count,
               avg,
               e->min_time * 1000.0,
               e->max_time * 1000.0);
    }
}
```

---

## 22. TROUBLESHOOTING GUIDE

### 22.1 Common Issues and Solutions

| Issue | Symptom | Cause | Solution |
|-------|---------|-------|----------|
| Encoder not found | `Encoder not found` error | FFmpeg built without encoder | Rebuild with `--enable-libencoder` or use native encoder |
| Invalid sample format | `Sample format X not supported` | Wrong input format | Add aformat filter to convert |
| Invalid sample rate | `Sample rate Y not supported` | Rate not supported by codec | Use aresample to resample |
| Out of memory | `Cannot allocate buffer` | Frame buffers too large | Reduce frame size |
| Encoding too slow | Real-time factor < 1.0 | CPU too slow | Use faster preset or hardware encode |
| Output file too large | Unexpected size | Bitrate too high | Reduce `-b:a` or `-q:a` |
| Audio quality poor | Artifacts, distortion | Bitrate too low | Increase `-b:a` or `-q:a` |
| Metadata lost | Tags missing in output | Container doesn't support | Use different container |
| Seeking broken | Can't seek in output | No index written | Use `-movflags +faststart` for MP4 |

### 22.2 Debugging Techniques

```bash
# Enable debug logging
ffmpeg -loglevel debug -i input.wav -c:a aac output.m4a 2>&1 | tee debug.log

# Check encoder capabilities
ffmpeg -encoders 2>&1 | grep -i "aac\|opus\|vorbis"

# Verify input format
ffprobe -v debug -show_streams input.wav

# Check bitrate allocation
ffmpeg -i input.wav -c:a libmp3lame -b:a 192k -stats output.mp3

# Profile encoder performance
time ffmpeg -i input.wav -c:a aac -b:a 256k output.m4a

# Compare two encoders
ffmpeg -i input.wav -c:a libmp3lame -q:a 2 out1.mp3
ffmpeg -i input.wav -c:a aac -b:a 192k out2.m4a
ffprobe -v quiet -show_format out1.mp3 | grep bit_rate
ffprobe -v quiet -show_format out2.m4a | grep bit_rate
```

### 22.3 Version Compatibility Matrix

| Feature | FFmpeg 4.x | FFmpeg 5.x | FFmpeg 6.x | FFmpeg 7.x |
|---------|------------|-------------|-------------|-------------|
| Native AAC encoder | ✓ | ✓ | ✓ | ✓ |
| libopus support | ✓ | ✓ | ✓ | ✓ |
| libvorbis support | ✓ | ✓ | ✓ | ✓ |
| AVChannelLayout | ✓ | ✓ | ✓ | ✓ |
| swr_alloc_set_opts2 | ✓ | ✓ | ✓ | ✓ |
| Filter segment API | New | ✓ | ✓ | ✓ |
| VAAPI audio | Limited | ✓ | ✓ | ✓ |

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
