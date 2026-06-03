# FFmpeg High-Resolution Audio: 24-bit, 32-bit, and DSD — Deep Technical Reference

> **Category:** FFmpeg API
> **File Extensions:** N/A (FFmpeg API reference)
> **MIME Types:** N/A
> **Standardization Body:** FFmpeg Project / Various (ISO/IEC, Sony, Philips)
> **Primary Specification:** FFmpeg source code (`libavutil/samplefmt.h`, `libavcodec/dsddec.c`, `libswresample/`)
> **Patent Status:** Mixed — FFmpeg is LGPL/GPL; DSD is patent-encumbered (Sony/Philips SACD)
> **License:** LGPL 2.1+ (FFmpeg core); proprietary for some DSD-related technology
> **Current Version:** FFmpeg 7.x (ongoing development)
> **Active Development:** Yes — last release ongoing

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation

- **Creator(s):** FFmpeg Project (multilingual open-source team); DSD by Sony and Philips (1999)
- **Year Created:** FFmpeg audio APIs evolved continuously from 2000s; DSD introduced 1999 for SACD
- **Original Purpose:** Handle audio beyond CD-quality (16-bit/44.1kHz) in FFmpeg; DSD is an alternative to PCM using 1-bit sigma-delta modulation
- **Problem with Predecessors:** Standard PCM audio at CD quality cannot capture the full dynamic range and ultrasonic detail available in studio recordings; DSD provides an alternative encoding that is inherently oversampled

### 1.2 Version History

| Version | Year | Key Changes |
|---------|------|-------------|
| FFmpeg 0.6 | 2010 | Basic high-bitdepth PCM support, s32le/s32be introduced |
| FFmpeg 2.0 | 2013 | Expanded sample rate support up to 384kHz |
| FFmpeg 3.0 | 2016 | libswresample redesign, planar formats standardized |
| FFmpeg 4.0 | 2018 | SoX resampler (libsoxr) integration stabilized |
| FFmpeg 6.0 | 2023 | AVFrame `bits_per_raw_sample` field, DSD decoder improvements |
| FFmpeg 7.0 | 2024 | Continued high-res audio improvements, API refinements |

### 1.3 Current Adoption

- **Primary use cases today:** Studio recording archival, high-end audio streaming (Tidal, Qobuz), DSD-to-PCM conversion pipelines, audiophile audio processing
- **Platforms with native support:** All platforms supported by FFmpeg (Linux, Windows, macOS, BSD, mobile)
- **Major services using this format:** Tidal (MQA and hi-res PCM), Qobuz (hi-res FLAC), some SACD rip archives
- **Hardware support:** High-resolution DACs (DSD-capable players, USB DACs), A/V receivers
- **Status:** Growing adoption; hi-res audio market expanding rapidly

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 High-Resolution Audio Definitions

High-resolution (hi-res) audio exceeds Compact Disc quality specifications:

| Parameter | CD Quality | Hi-Res Typical | Notes |
|-----------|-----------|----------------|-------|
| Bit Depth | 16-bit | 24-bit, 32-bit | Each additional bit doubles dynamic range (+6.02 dB) |
| Sample Rate | 44.1 kHz | 88.2, 96, 176.4, 192, 352.8, 384 kHz | Nyquist: rate/2 = max frequency |
| Theoretical Dynamic Range | 96 dB | 144 dB (24-bit), 192 dB (32-bit) | Practical limits ~120-140 dB due to noise floor |
| Frequency Response (96kHz) | 20 Hz – 20 kHz | Up to 48 kHz | Extended high-frequency response |

### 2.2 FFmpeg Sample Format System

FFmpeg internally represents audio using the `AVSampleFormat` enum. The key formats for high-resolution audio are defined in `libavutil/samplefmt.h`:

```c
enum AVSampleFormat {
    AV_SAMPLE_FMT_NONE = -1,
    AV_SAMPLE_FMT_U8,      // unsigned 8-bit integer
    AV_SAMPLE_FMT_S16,     // signed 16-bit integer, packed
    AV_SAMPLE_FMT_S32,     // signed 32-bit integer, packed
    AV_SAMPLE_FMT_FLT,     // 32-bit IEEE float, packed
    AV_SAMPLE_FMT_DBL,     // 64-bit IEEE double, packed
    AV_SAMPLE_FMT_U8P,     // unsigned 8-bit, planar
    AV_SAMPLE_FMT_S16P,    // signed 16-bit, planar
    AV_SAMPLE_FMT_S32P,    // signed 32-bit, planar
    AV_SAMPLE_FMT_FLTP,    // 32-bit float, planar
    AV_SAMPLE_FMT_DBLP,    // 64-bit double, planar
    AV_SAMPLE_FMT_S64,     // signed 64-bit integer, packed
    AV_SAMPLE_FMT_S64P,    // signed 64-bit integer, planar
    AV_SAMPLE_FMT_DSD,     // 1-bit DSD, packed
    AV_SAMPLE_FMT_DSDP,    // 1-bit DSD, planar
    AV_SAMPLE_FMT_NB       // Number of formats (do not use)
};
```

### 2.3 Sample Format Information Table

From `libavutil/samplefmt.c`, each format carries metadata:

| Format Name | Internal Bits | Planar | Alternative Form | Audio Type |
|-------------|--------------|--------|-----------------|------------|
| `u8` | 8 | No | `u8p` | Unsigned integer |
| `s16` | 16 | No | `s16p` | Signed integer |
| `s32` | 32 | No | `s32p` | Signed integer (used for 24-bit) |
| `s32p` | 32 | Yes | `s32` | Signed integer, planar |
| `flt` | 32 | No | `fltp` | IEEE 754 float |
| `fltp` | 32 | Yes | `flt` | IEEE 754 float, planar |
| `dbl` | 64 | No | `dblp` | IEEE 754 double |
| `dblp` | 64 | Yes | `dbl` | IEEE 754 double, planar |
| `s64` | 64 | No | `s64p` | Signed 64-bit integer |
| `dsd` | 8 | No | `dsdp` | 1-bit DSD |
| `dsdp` | 8 | Yes | `dsd` | 1-bit DSD, planar |

### 2.4 High-Level Data Flow for High-Resolution Audio

```
Input Audio File (24-bit/96kHz WAV)
        │
        ▼
[Demuxer: Extracts raw audio packets from container]
        │
        ▼
[Decoder: Converts compressed audio to PCM in internal format]
        │
        ▼
[libswresample: Converts sample format, sample rate, channel layout]
        │
   ┌────┴────┐
   │  s32le  │ ←── 24-bit audio stored in 32-bit container
   │  fltp   │ ←── Planar float for internal processing
   │  s32p   │ ←── Planar 32-bit integer
   └─────────┘
        │
        ▼
[Encoder: Encodes to target format (FLAC, ALAC, WAV, etc.)]
        │
        ▼
Output Audio File
```

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 How FFmpeg Stores 24-bit Audio in 32-bit Containers

A critical understanding for hi-res audio in FFmpeg: **24-bit audio does not map directly to any native CPU type**. Most CPU architectures work with 8, 16, 32, or 64-bit types. Therefore, FFmpeg stores 24-bit audio in 32-bit containers:

```
Memory Layout for 24-bit Stereo Sample (interleaved s32):

Offset 0x00: [BYTE 0][BYTE 1][BYTE 2][PAD BYTE]  ← Channel 0 (Left), 24-bit + 8-bit padding
Offset 0x04: [BYTE 0][BYTE 1][BYTE 2][PAD BYTE]  ← Channel 1 (Right), 24-bit + 8-bit padding

Each 24-bit sample occupies 4 bytes (32-bit slot)
Padding byte is typically 0x00 (or sign-extended on some operations)
Effective bit depth: 24 bits per channel
Storage overhead: 33% larger than minimal 24-bit storage
```

### 3.2 Sample Format CLI Naming Conventions

| FFmpeg CLI Name | Description | Bits | Planar |
|-----------------|-------------|------|--------|
| `s8` | Signed 8-bit | 8 | No |
| `s16le` / `s16be` | Signed 16-bit little/big endian | 16 | No |
| `s32le` / `s32be` | Signed 32-bit little/big endian | 32 | No |
| `f32le` / `f32be` | 32-bit float little/big endian | 32 | No |
| `f64le` / `f64be` | 64-bit float little/big endian | 64 | No |
| `s24le` / `s24be` | Signed 24-bit little/big endian | 24 | No |
| `u8` | Unsigned 8-bit | 8 | No |
| `u16le` / `u16be` | Unsigned 16-bit | 16 | No |

### 3.3 DSD (Direct Stream Digital) Binary Format

DSD uses 1-bit sigma-delta modulation at very high sample rates:

| DSD Variant | Sample Rate | Bit Depth | Channels | Effective PCM Equivalent |
|-------------|-------------|-----------|----------|------------------------|
| DSD64 | 2.8224 MHz | 1-bit | Stereo/Mono | 64× CD rate |
| DSD128 | 5.6448 MHz | 1-bit | Stereo/Mono | 128× CD rate |
| DSD256 | 11.2896 MHz | 1-bit | Stereo/Mono | 256× CD rate |
| DSD512 | 22.5792 MHz | 1-bit | Stereo/Mono | 512× CD rate |

DSD File Formats:

| Format | Extension | Description | FFmpeg Support |
|--------|-----------|-------------|----------------|
| DSF | `.dsf` | DSD Stream File (Sony) | Decoder: `dsf` |
| DFF | `.dff` | DSDIFF (Philips) | Decoder: `dff` |
| ISO | `.iso` | SACD disc image | Partial support |

### 3.4 DSF File Structure (Sony Format)

```
Offset     Size    Field Name              Description
---------  ------  ----------------------  ----------------------------
0x0000     4B      Magic                  "DSD " (0x44534420)
0x0004     4B      fmt Offset             Offset to format chunk
0x0008     4B      data Offset            Offset to data chunk
0x000C     4B      fmt Size              Size of format chunk
0x0010     4B      format Version         Format version number
0x0014     4B      format ID              Format ID (1 = DSD)
0x0018     4B      channel Type          1=mono, 2=stereo
0x001C     4B      sample Rate            Sample rate in Hz (e.g., 2822400)
0x0020     4B      bits per Sample        Bits per sample (1 for DSD)
0x0024     8B      sample Count           Total number of samples per channel
0x002C     4B      block Size per Channel Block size in bytes
[Data Chunk starts at data Offset]
           N×B     DSD Data               Raw 1-bit DSD samples
```

---

## 4. FFMPEG SAMPLE FORMAT DETAILS

### 4.1 Packed vs Planar Formats

**Packed formats** (interleaved):
- All channel samples stored consecutively in memory
- Memory layout: `[CH0_S0][CH1_S0][CH0_S1][CH1_S1]...`
- Example: `s16le`, `s32le`, `f32le`
- Best for: Sequential processing, memory efficiency for small channel counts

**Planar formats** (chunked):
- Each channel stored in separate memory region
- Memory layout: `[ALL CH0 SAMPLES][ALL CH1 SAMPLES]...`
- Example: `s32p`, `fltp`, `dblp`
- Best for: SIMD processing, when channels are processed independently
- Required by: Many modern audio codecs (AAC, MP3, Opus, most video codecs)

### 4.2 24-bit Audio Representation in FFmpeg

When FFmpeg reports `s32` for a 24-bit file:

```bash
# Example ffprobe output showing 24-bit in 32-bit container:
# Stream #0:0: Audio: pcm_s24le, 48000 Hz, stereo, s32 (24 bit), 2304 kb/s
```

The `pcm_s24le` demuxer reads 24-bit data, but:
- FFmpeg's internal `sample_fmt` is `AV_SAMPLE_FMT_S32` (32-bit)
- The actual audio precision is 24 bits
- FFprobe may show both: `s32` (container format) and `(24 bit)` (actual precision)

### 4.3 Bit-Exact 24-bit Handling

To maintain bit-exact 24-bit precision through FFmpeg:

```bash
# Read 24-bit file and preserve precision
ffmpeg -i input_24bit.wav -c:a pcm_s24le output.wav

# Verify with ffprobe
ffprobe -v error -show_entries stream=bits_per_raw_sample,sample_fmt \
  -of default=noprint_wrappers=1 input_24bit.wav
# Expected: bits_per_raw_sample=24, sample_fmt=s32

# Convert to 24-bit FLAC (lossless)
ffmpeg -i input_24bit.wav -c:a flac -sample_fmt s32 -compression_level 8 output.flac
```

### 4.4 Floating-Point Audio in FFmpeg

| Format | Type | Range (normalized) | Use Case |
|--------|------|-------------------|----------|
| `flt` / `fltp` | 32-bit IEEE 754 float | [-1.0, +1.0] | Internal processing, most modern codecs |
| `dbl` / `dblp` | 64-bit IEEE 754 double | [-1.0, +1.0] | High-precision intermediate calculations |

**Critical note:** Float formats in FFmpeg use normalized range [-1.0, +1.0], not the full range of the integer type. Values outside this range will clip.

```c
// Float to int16 conversion example:
float sample_f = 0.5f;  // Normalized float
int16_t sample_i16 = (int16_t)(sample_f * 32767.0f);
```

---

## 5. DSD (DIRECT STREAM DIGITAL) SUPPORT IN FFMPEG

### 5.1 DSD Decoders in FFmpeg

FFmpeg provides DSD-to-PCM decoders for DSF and DFF formats:

| Codec Name | AV_CODEC_ID | Description | Bit Order |
|-----------|-------------|-------------|-----------|
| `dsd_lsbf` | `AV_CODEC_ID_DSD_LSBF` | DSD, least significant bit first | LSB first |
| `dsd_msbf` | `AV_CODEC_ID_DSD_MSBF` | DSD, most significant bit first | MSB first |
| `dsd_lsbf_planar` | `AV_CODEC_ID_DSD_LSBF_PLANAR` | DSD, LSB first, planar | LSB first |
| `dsd_msbf_planar` | `AV_CODEC_ID_DSD_MSBF_PLANAR` | DSD, MSB first, planar | MSB first |

### 5.2 DSD to PCM Conversion Process

```
DSD Bitstream (1-bit @ 2.8224 MHz)
        │
        ▼
[DSD Decoder: Unpacks 1-bit samples]
        │
        ▼
[dsd2pcm Algorithm: Sigma-delta to PCM conversion]
        │
   Output: Float PCM @ 352.8 kHz (DSD64) or 705.6 kHz (DSD128)
```

The `dsd2pcm` algorithm (originally from Sebastian Gesemann, BSD licensed) performs noise-shaped decimation:

```c
// From libavcodec/dsddec.c
void ff_dsd2pcm_translate(DSDContext *s, size_t samples,
                          int lsbf, const uint8_t *src, int src_stride,
                          float *dst, int dst_stride);
```

### 5.3 DSD Decoding Commands

```bash
# Basic DSD to PCM conversion (outputs to stdout)
ffmpeg -i input.dsf -c:a dsd_msbf output.wav

# DSD to FLAC (24-bit/96kHz PCM)
ffmpeg -i input.dsf -c:a dsd_msbf -ar 96000 -sample_fmt s32 output.flac

# DSD to WAV with lowpass filter (recommended to remove ultrasonic noise)
ffmpeg -i input.dsf -af "lowpass=24000,volume=6dB" -ar 48000 output.wav

# DSD to high-resolution FLAC
ffmpeg -i input.dsf -c:a dsd_msbf -ar 352800 -sample_fmt s32 -c:a flac \
  -compression_level 8 output.flac
```

### 5.4 DSD Sample Rate Relationship to PCM

| DSD Rate | PCM Output Rate | Notes |
|----------|----------------|-------|
| DSD64 (2.8224 MHz) | 352.8 kHz | 2.8224MHz / 8 = 352.8kHz |
| DSD128 (5.6448 MHz) | 705.6 kHz | 2× DSD64 |
| DSD256 (11.2896 MHz) | 1411.2 kHz | 4× DSD64 |
| DSD512 (22.5792 MHz) | 2822.4 kHz | 8× DSD64 |

### 5.5 DSD-over-PCM (DoP) Encoding

DoP is a method to transport DSD through standard PCM infrastructure:

```bash
# DSD-over-PCM encoding (DSD in 24-bit PCM container)
# Each 24-bit PCM word contains 3 bytes of DSD data
# Markers in bits 21-23 indicate DSD byte boundaries
ffmpeg -i input.dsf -c:a dop output.dff

# DoP uses 176.4 kHz PCM to carry DSD64
# Formula: DSD_rate / 16 = DoP_PCM_rate
```

---

## 6. FFMPEG SAMPLE RATE SUPPORT

### 6.1 Supported Sample Rates for High-Resolution

FFmpeg supports arbitrary sample rates via libswresample. Standard hi-res rates:

| Sample Rate | Common Name | CD Multiplier | Notes |
|-------------|-------------|---------------|-------|
| 88200 Hz | 2× CD | 2× | Hi-res standard |
| 96000 Hz | HD Audio | 2.18× | Common in studios |
| 176400 Hz | 4× CD | 4× | DXD format |
| 192000 Hz | HD Audio max | 4.35× | Blu-ray audio standard |
| 352800 Hz | DXD | 8× | DSD64 equivalent PCM |
| 384000 Hz | Ultra HD | 8.7× | Maximum commonly supported |
| 705600 Hz | — | 16× | DSD128 equivalent PCM |
| 768000 Hz | — | 17.4× | Professional audio |

### 6.2 Sample Rate Conversion

```bash
# Upsample to 192kHz using default SWR resampler
ffmpeg -i input.wav -ar 192000 output.wav

# High-quality upsampling using SoX resampler (requires libsoxr)
ffmpeg -i input.wav -af "aresample=resampler=soxr:precision=28" -ar 192000 output.wav

# Verify output sample rate
ffprobe -v error -select_streams a:0 -show_entries stream=sample_rate \
  -of default=noprint_wrappers=1:nokey=1 output.wav
```

### 6.3 Arbitrary Sample Rates

FFmpeg accepts any integer sample rate within the supported range:

```bash
# Common non-standard rates
ffmpeg -i input.wav -ar 88200 output.wav      # 2× CD rate
ffmpeg -i input.wav -ar 176400 output.wav     # 4× CD rate

# Verify rate is maintained
ffprobe -v error -select_streams a -show_entries stream=sample_rate \
  -of csv=p=0 input.wav output.wav
```

---

## 7. FFMPEG RESAMPLING AND QUALITY

### 7.1 Resampler Options Comparison

| Resampler | Option | Quality | Speed | Precision Range |
|-----------|--------|---------|-------|-----------------|
| SWR (built-in) | `resampler=swr` | Medium | Fast | Fixed |
| SoX | `resampler=soxr` | High | Medium | 15–33 bits |

### 7.2 SoX Resampler (libsoxr) Options

```bash
# Very High Quality SoX resampling
ffmpeg -i input.wav \
  -af "aresample=resampler=soxr:precision=28" \
  -ar 384000 output.wav

# SoX with custom cutoff (preserve more high frequencies)
ffmpeg -i input.wav \
  -af "aresample=resampler=soxr:precision=28:cutoff=0.95" \
  -ar 192000 output.wav

# SoX with Chebyshev passband (for irrational sample rate ratios)
ffmpeg -i input.wav \
  -af "aresample=resampler=soxr:precision=28:cheby=1" \
  -ar 44100 output.wav
```

### 7.3 SoX Resampler Precision Levels

| Precision Value | Quality Level | Notes |
|-----------------|---------------|-------|
| 15 | Minimum | Fast, lower quality |
| 20 | High (default) | SoX default "High Quality" |
| 28 | Very High | Recommended for archival |
| 33 | Maximum | Overkill for most cases |

---

## 8. DITHERING AND TRUNCATION

### 8.1 The Truncation Problem

When reducing bit depth (e.g., 24-bit to 16-bit), simple truncation introduces quantization distortion:

```python
# Truncation example: 24-bit to 16-bit
# Original 24-bit sample: 0x123456 (big value)
# Truncated to 16-bit:   0x3456 (loses 8 bits of precision)

# This creates systematic error at each quantization step
# Result: audible distortion, especially for low-level signals
```

### 8.2 Dithering Methods in FFmpeg

FFmpeg provides multiple dithering algorithms via the SWR resampler:

| Method | Type | Notes |
|--------|------|-------|
| `rectangular` | Basic | Flat noise floor |
| `triangular` | Basic | 6 dB lower noise floor than rectangular |
| `triangular_hp` | Basic | Triangular with high-pass filter |
| `shibata` | Noise shaping | Optimized for audibility |
| `low_shibata` | Noise shaping | Lower computational cost |
| `high_shibata` | Noise shaping | Maximum noise shaping |
| `lipshitz` | Noise shaping | Low in-band noise |
| `f_weighted` | Noise shaping | Frequency-weighted |
| `modified_e_weighted` | Noise shaping | Modified E-weighted |
| `improved_e_weighted` | Noise shaping | Improved E-weighted |

### 8.3 Applying Dithering

```bash
# Dither when reducing bit depth (24-bit to 16-bit)
ffmpeg -i input_24bit.wav \
  -af "aresample=osf=s16:dither_method=shibata" \
  -c:a pcm_s16le output_16bit.wav

# Dither with custom dither scale
ffmpeg -i input_24bit.wav \
  -af "aresample=osf=s16:dither_method=shibata:dither_scale=1.0" \
  output_16bit.wav

# High-quality downsample with dithering
ffmpeg -i input_24bit_96k.wav \
  -af "aresample=44100:resampler=soxr:precision=28:osf=s16:dither_method=shibata" \
  -c:a alac output.m4a
```

### 8.4 When Dithering Is Important

| Scenario | Dithering Recommended? |
|----------|----------------------|
| 24-bit → 16-bit (music) | Yes — always |
| 32-bit float → 24-bit | Yes — for archival |
| 24-bit → 24-bit (same rate) | No — no bit reduction |
| 32-bit float → 32-bit float | No — no bit reduction |
| DSD → 24-bit/96kHz PCM | Yes — after lowpass |

---

## 9. HIGH-RESOLUTION ENCODING

### 9.1 FLAC at 24-bit

```bash
# 24-bit FLAC encoding
ffmpeg -i input.wav -c:a flac -sample_fmt s32 -compression_level 8 output.flac

# Verify 24-bit encoding
ffprobe -v error -select_streams a:0 \
  -show_entries stream=bits_per_raw_sample,sample_fmt,sample_rate \
  -of default=noprint_wrappers=1 output.flac
# Expected: bits_per_raw_sample=24, sample_fmt=s32, sample_rate=96000
```

### 9.2 ALAC at 24-bit

```bash
# 24-bit ALAC (Apple Lossless)
ffmpeg -i input.wav -c:a alac -sample_fmt s32 output.m4a

# Verify with MediaInfo equivalent ffprobe output
ffprobe -v error -select_streams a:0 \
  -show_entries stream=codec_name,bits_per_raw_sample,sample_rate \
  -of default=noprint_wrappers=1 output.m4a
```

### 9.3 WAV at 24-bit

```bash
# 24-bit WAV (PCM)
ffmpeg -i input.wav -c:a pcm_s24le output.wav

# 32-bit WAV (for processing)
ffmpeg -i input.wav -c:a pcm_s32le output.wav
```

### 9.4 FFmpeg Build for SoX Resampler

```bash
# Check if libsoxr is available
ffmpeg -buildconf 2>&1 | grep -i soxr
# If present: --enable-libsoxr

# Build FFmpeg with libsoxr support
./configure --enable-libsoxr --extra-cflags="-I/usr/local/include" \
  --extra-ldflags="-L/usr/local/lib"
make -j$(nproc)
sudo make install

# Verify SoX resampler is available
ffmpeg -h filter=aresample 2>&1 | grep -i soxr
```

---

## 10. FFMPEG CLI REFERENCE — HIGH-RESOLUTION AUDIO

### 10.1 Preserving 24-bit Audio Through Conversion

```bash
# Copy audio without re-encoding (bit-exact)
ffmpeg -i input.flac -c:a copy output.wav

# Convert to 24-bit FLAC, preserve all metadata
ffmpeg -i input.wav -c:a flac -sample_fmt s32 \
  -compression_level 8 \
  -map_metadata 0 \
  output.flac

# Convert to 24-bit ALAC with metadata
ffmpeg -i input.wav -c:a alac -sample_fmt s32 \
  -map_metadata 0 \
  output.m4a

# Convert DSD to 24-bit/352.8kHz PCM FLAC
ffmpeg -i input.dsf -c:a dsd_msbf -ar 352800 -sample_fmt s32 \
  -c:a flac -compression_level 8 output.flac
```

### 10.2 High-Resolution Sample Rate Conversion

```bash
# Upsample to 192kHz with SoX resampler
ffmpeg -i input.wav \
  -af "aresample=resampler=soxr:precision=28" \
  -ar 192000 \
  output_192k.wav

# Downsample to 48kHz with dithering
ffmpeg -i input_96k.wav \
  -af "aresample=resampler=soxr:precision=28:osf=s16:dither_method=shibata" \
  -ar 48000 \
  -c:a alac \
  output_48k.m4a

# Arbitrary rate conversion (44.1kHz to 96kHz)
ffmpeg -i input_44.1k.wav \
  -af "aresample=resampler=soxr:precision=28" \
  -ar 96000 \
  output_96k.wav
```

### 10.3 Batch Processing High-Resolution Files

```bash
# Convert all FLAC files to 24-bit/96kHz ALAC
for f in *.flac; do
  ffmpeg -i "$f" -c:a alac -sample_fmt s32 -ar 96000 \
    -map_metadata 0 "${f%.flac}.m4a"
done

# Verify all converted files
for f in *.m4a; do
  echo "=== $f ==="
  ffprobe -v error -select_streams a:0 \
    -show_entries stream=bits_per_raw_sample,sample_rate \
    -of default=noprint_wrappers=1 "$f"
done
```

---

## 11. FFMPEG LIBavfilter — aformat FILTER

### 11.1 aformat Filter Syntax

The `aformat` filter constrains output to specific formats:

```bash
# Set output to 24-bit/96kHz stereo
ffmpeg -i input.mkv \
  -af "aformat=sample_fmts=s32:sample_rates=96000:channel_layouts=stereo" \
  -c:a flac output.flac

# Multiple acceptable formats
ffmpeg -i input.mkv \
  -af "aformat=sample_fmts=s32|s32p:sample_rates=96000|192000:channel_layouts=stereo" \
  output.wav
```

### 11.2 Combining aformat with aresample

```bash
# Force 24-bit output with resampling
ffmpeg -i input.wav \
  -af "aresample=osf=s32, aformat=sample_fmts=s32:sample_rates=96000" \
  -c:a pcm_s32le output.wav

# Chain: resample → format → encode
ffmpeg -i input_44.1k.wav \
  -af "aresample=resampler=soxr:precision=28, aformat=sample_fmts=s32:sample_rates=96000" \
  -c:a flac output.flac
```

---

## 12. FFMPEG C API — HIGH-RESOLUTION AUDIO

### 12.1 Sample Format Detection

```c
#include <libavutil/samplefmt.h>
#include <libavutil/opt.h>
#include <stdio.h>

void print_sample_format_info(enum AVSampleFormat fmt) {
    const char *name = av_get_sample_fmt_name(fmt);
    int bits = av_get_bytes_per_sample(fmt) * 8;
    int planar = av_sample_fmt_is_planar(fmt);
    
    printf("Format: %s\n", name);
    printf("Bytes per sample: %d\n", av_get_bytes_per_sample(fmt));
    printf("Planar: %s\n", planar ? "Yes" : "No");
    printf("Is float: %s\n", 
           av_sample_fmt_is_planar(fmt) ? "N/A" : 
           (fmt == AV_SAMPLE_FMT_FLT || fmt == AV_SAMPLE_FMT_DBL) ? "Yes" : "No");
}

void analyze_audio_stream(AVStream *stream) {
    AVCodecParameters *par = stream->codecpar;
    
    printf("Sample Rate: %d Hz\n", par->sample_rate);
    printf("Channels: %d\n", par->ch_layout.nb_channels);
    printf("Sample Format: %s\n", 
           av_get_sample_fmt_name(par->format));
    printf("Bits per raw sample: %d\n", par->bits_per_raw_sample);
    
    // Check if actual bit depth is different from container
    int container_bits = av_get_bytes_per_sample(par->format) * 8;
    if (par->bits_per_raw_sample > 0 && 
        par->bits_per_raw_sample != container_bits) {
        printf("Note: %d-bit audio stored in %d-bit container\n",
               par->bits_per_raw_sample, container_bits);
    }
}
```

### 12.2 High-Resolution Encoding with libavcodec

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/opt.h>
#include <libswresample/swresample.h>

int encode_24bit_audio(const char *input_file, const char *output_file) {
    AVFormatContext *ifmt_ctx = NULL, *ofmt_ctx = NULL;
    AVCodecContext *enc_ctx = NULL;
    const AVCodec *encoder = NULL;
    int ret;
    AVPacket *pkt = av_packet_alloc();
    AVFrame *frame = av_frame_alloc();
    
    // Open input
    ret = avformat_open_input(&ifmt_ctx, input_file, NULL, NULL);
    if (ret < 0) goto end;
    
    ret = avformat_find_stream_info(ifmt_ctx, NULL);
    if (ret < 0) goto end;
    
    // Find audio stream
    int audio_stream_idx = av_find_best_stream(ifmt_ctx, AVMEDIA_TYPE_AUDIO, -1, -1, NULL, 0);
    if (audio_stream_idx < 0) { ret = AVERROR_STREAM_NOT_FOUND; goto end; }
    AVStream *in_stream = ifmt_ctx->streams[audio_stream_idx];
    
    // Find encoder (FLAC for lossless hi-res)
    encoder = avcodec_find_encoder(AV_CODEC_ID_FLAC);
    if (!encoder) { ret = AVERROR_ENCODER_NOT_FOUND; goto end; }
    
    // Create output context
    avformat_alloc_output_context2(&ofmt_ctx, NULL, NULL, output_file);
    if (!ofmt_ctx) { ret = AVERROR(ENOMEM); goto end; }
    
    // Create output stream
    AVStream *out_stream = avformat_new_stream(ofmt_ctx, NULL);
    if (!out_stream) { ret = AVERROR(ENOMEM); goto end; }
    
    // Allocate encoder context
    enc_ctx = avcodec_alloc_context3(encoder);
    if (!enc_ctx) { ret = AVERROR(ENOMEM); goto end; }
    
    // Set encoder parameters for 24-bit hi-res
    enc_ctx->sample_fmt = AV_SAMPLE_FMT_S32P;  // FLAC accepts planar 32-bit
    enc_ctx->sample_rate = 96000;               // Hi-res sample rate
    enc_ctx->bit_rate = 0;                      // Lossless, no bitrate target
    enc_ctx->compression_level = 8;              // Maximum compression
    
    // Copy channel layout
    av_channel_layout_copy(&enc_ctx->ch_layout, &in_stream->codecpar->ch_layout);
    
    // Open encoder
    ret = avcodec_open2(enc_ctx, encoder, NULL);
    if (ret < 0) goto end;
    
    // Copy codec parameters to output stream
    ret = avcodec_parameters_from_context(out_stream->codecpar, enc_ctx);
    if (ret < 0) goto end;
    
    // Open output file
    if (!(ofmt_ctx->oformat->flags & AVFMT_NOFILE)) {
        ret = avio_open(&ofmt_ctx->pb, output_file, AVIO_FLAG_WRITE);
        if (ret < 0) goto end;
    }
    
    // Write header
    ret = avformat_write_header(ofmt_ctx, NULL);
    if (ret < 0) goto end;
    
    // Decode and encode loop
    SwrContext *swr_ctx = NULL;
    AVFrame *resampled_frame = NULL;
    
    while (1) {
        AVPacket *pkt = av_packet_alloc();
        ret = av_read_frame(ifmt_ctx, pkt);
        if (ret < 0) {
            av_packet_free(&pkt);
            break;
        }
        
        if (pkt->stream_index == audio_stream_idx) {
            // Decode frame
            ret = avcodec_send_packet(enc_ctx, pkt);
            av_packet_free(&pkt);
            
            while (ret >= 0) {
                AVFrame *dec_frame = av_frame_alloc();
                ret = avcodec_receive_frame(enc_ctx, dec_frame);
                if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF) {
                    av_frame_free(&dec_frame);
                    break;
                }
                if (ret < 0) {
                    av_frame_free(&dec_frame);
                    goto end;
                }
                
                // Resample if needed
                if (swr_ctx) {
                    swr_convert(swr_ctx, NULL, 0, (const uint8_t**)&dec_frame->data, dec_frame->nb_samples);
                }
                
                av_frame_free(&dec_frame);
            }
        } else {
            av_packet_free(&pkt);
        }
    }
    
    // Flush encoder
    avcodec_send_frame(enc_ctx, NULL);
    while (1) {
        ret = avcodec_receive_packet(enc_ctx, pkt);
        if (ret == AVERROR_EOF) break;
        if (ret < 0) goto end;
        av_interleaved_write_frame(ofmt_ctx, pkt);
    }
    
    av_write_trailer(ofmt_ctx);
    
end:
    av_packet_free(&pkt);
    av_frame_free(&frame);
    avcodec_free_context(&enc_ctx);
    avformat_close_input(&ifmt_ctx);
    if (ofmt_ctx && !(ofmt_ctx->oformat->flags & AVFMT_NOFILE))
        avio_closep(&ofmt_ctx->pb);
    avformat_free_context(ofmt_ctx);
    if (swr_ctx) swr_free(&swr_ctx);
    
    return ret < 0 ? ret : 0;
}
```

### 12.3 DSD Decoding with libavcodec

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>

int decode_dsd_to_pcm(const char *dsd_file, const char *pcm_file,
                      int output_sample_rate, enum AVSampleFormat output_fmt) {
    AVFormatContext *fmt_ctx = NULL;
    int ret;
    
    // Open DSD file
    ret = avformat_open_input(&fmt_ctx, dsd_file, NULL, NULL);
    if (ret < 0) {
        char errbuf[128];
        av_strerror(ret, errbuf, sizeof(errbuf));
        fprintf(stderr, "Cannot open input: %s\n", errbuf);
        return ret;
    }
    
    ret = avformat_find_stream_info(fmt_ctx, NULL);
    if (ret < 0) {
        fprintf(stderr, "Cannot find stream info\n");
        goto end;
    }
    
    // Find audio stream
    int audio_stream = av_find_best_stream(fmt_ctx, AVMEDIA_TYPE_AUDIO, -1, -1, NULL, 0);
    if (audio_stream < 0) {
        fprintf(stderr, "Cannot find audio stream\n");
        ret = AVERROR_STREAM_NOT_FOUND;
        goto end;
    }
    
    AVStream *stream = fmt_ctx->streams[audio_stream];
    const AVCodec *decoder = avcodec_find_decoder(stream->codecpar->codec_id);
    if (!decoder) {
        fprintf(stderr, "Codec not found\n");
        ret = AVERROR_DECODER_NOT_FOUND;
        goto end;
    }
    
    // Check if this is a DSD decoder
    if (stream->codecpar->codec_id == AV_CODEC_ID_DSD_LSBF ||
        stream->codecpar->codec_id == AV_CODEC_ID_DSD_MSBF ||
        stream->codecpar->codec_id == AV_CODEC_ID_DSD_LSBF_PLANAR ||
        stream->codecpar->codec_id == AV_CODEC_ID_DSD_MSBF_PLANAR) {
        printf("DSD stream detected: %s\n", decoder->name);
    }
    
    // Open decoder
    AVCodecContext *dec_ctx = avcodec_alloc_context3(decoder);
    if (!dec_ctx) {
        ret = AVERROR(ENOMEM);
        goto end;
    }
    
    ret = avcodec_parameters_to_context(dec_ctx, stream->codecpar);
    if (ret < 0) goto end;
    
    ret = avcodec_open2(dec_ctx, decoder, NULL);
    if (ret < 0) goto end;
    
    printf("Decoding %s @ %d Hz to %s @ %d Hz\n",
           dsd_file, dec_ctx->sample_rate, pcm_file, output_sample_rate);
    
    // Decode loop
    AVPacket *pkt = av_packet_alloc();
    AVFrame *frame = av_frame_alloc();
    
    while (av_read_frame(fmt_ctx, pkt) >= 0) {
        if (pkt->stream_index == audio_stream) {
            ret = avcodec_send_packet(dec_ctx, pkt);
            if (ret < 0) {
                fprintf(stderr, "Send packet error\n");
                break;
            }
            
            while (ret >= 0) {
                ret = avcodec_receive_frame(dec_ctx, frame);
                if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF) {
                    break;
                }
                if (ret < 0) {
                    fprintf(stderr, "Receive frame error\n");
                    goto decode_end;
                }
                
                // frame now contains PCM float samples
                // Output format: float planar, sample_rate may be changed
                printf("Frame: %d samples @ %d Hz, format: %s\n",
                       frame->nb_samples, frame->sample_rate,
                       av_get_sample_fmt_name(frame->format));
                
                av_frame_unref(frame);
            }
        }
        av_packet_unref(pkt);
    }
    
decode_end:
    // Flush decoder
    avcodec_send_packet(dec_ctx, NULL);
    while (avcodec_receive_frame(dec_ctx, frame) == 0) {
        printf("Flush frame: %d samples\n", frame->nb_samples);
        av_frame_unref(frame);
    }
    
    av_frame_free(&frame);
    av_packet_free(&pkt);
    
end:
    avcodec_free_context(&dec_ctx);
    avformat_close_input(&fmt_ctx);
    
    return ret;
}
```

### 12.4 libswresample for High-Resolution Conversion

```c
#include <libswresample/swresample.h>
#include <libavutil/opt.h>

SwrContext* create_hi_res_resampler(
    int in_sample_rate, enum AVSampleFormat in_fmt,
    const AVChannelLayout *in_layout,
    int out_sample_rate, enum AVSampleFormat out_fmt,
    const AVChannelLayout *out_layout,
    int use_soxr) {
    
    SwrContext *swr = swr_alloc();
    if (!swr) return NULL;
    
    // Set conversion parameters
    av_opt_set_int(swr, "in_sample_rate", in_sample_rate, 0);
    av_opt_set_int(swr, "out_sample_rate", out_sample_rate, 0);
    av_opt_set_sample_fmt(swr, "in_sample_fmt", in_fmt, 0);
    av_opt_set_sample_fmt(swr, "out_sample_fmt", out_fmt, 0);
    av_opt_set_chlayout(swr, "in_ch_layout", in_layout, 0);
    av_opt_set_chlayout(swr, "out_ch_layout", out_layout, 0);
    
    // Configure SoX resampler if available and requested
    if (use_soxr) {
        av_opt_set(swr, "resampler", "soxr", 0);
        av_opt_set_double(swr, "precision", 28.0, 0);  // Very high quality
        av_opt_set_double(swr, "cutoff", 0.95, 0);    // 95% bandwidth
    }
    
    // Configure dithering for bit depth reduction
    if (out_fmt == AV_SAMPLE_FMT_S16 || out_fmt == AV_SAMPLE_FMT_S16P) {
        av_opt_set(swr, "dither_method", "shibata", 0);
        av_opt_set_double(swr, "dither_scale", 1.0, 0);
    }
    
    int ret = swr_init(swr);
    if (ret < 0) {
        swr_free(&swr);
        return NULL;
    }
    
    return swr;
}

int check_soxr_available(void) {
    const char *config = swresample_configuration();
    if (config && strstr(config, "libsoxr")) {
        printf("SoX resampler (libsoxr) is available\n");
        return 1;
    }
    printf("SoX resampler NOT available (using SWR)\n");
    return 0;
}

int perform_resample(SwrContext *swr, AVFrame *in_frame, AVFrame *out_frame) {
    int ret;
    
    // Calculate output buffer size
    int max_out_samples = av_rescale_rnd(
        swr_get_delay(swr, in_frame->sample_rate) + in_frame->nb_samples,
        out_frame->sample_rate, in_frame->sample_rate, AV_ROUND_UP);
    
    // Ensure output buffer is large enough
    if (max_out_samples > out_frame->nb_samples) {
        av_frame_unref(out_frame);
        out_frame->nb_samples = max_out_samples;
        ret = av_frame_get_buffer(out_frame, 0);
        if (ret < 0) return ret;
    }
    
    // Perform resampling
    ret = swr_convert(swr,
                      out_frame->extended_data, out_frame->nb_samples,
                      (const uint8_t **)in_frame->extended_data, in_frame->nb_samples);
    if (ret < 0) return ret;
    
    out_frame->nb_samples = ret;  // Actual number of output samples
    
    return 0;
}
```

---

## 13. BIT-EXACT VERIFICATION

### 13.1 Verifying Lossless Conversion

```bash
# Method 1: Compare checksums with framemd5
ffmpeg -i original.wav -map 0:a -f framemd5 original.md5
ffmpeg -i converted.flac -map 0:a -f framemd5 converted.md5
diff original.md5 converted.md5

# Method 2: Use bitstream comparison
ffmpeg -i original.wav -c:a pcm_s24le -f md5 -
ffmpeg -i converted.flac -c:a pcm_s24le -f md5 -

# Method 3: Hexdump comparison for short clips
ffmpeg -i original.wav -c:a pcm_s24le -t 5 -f s24le - | md5sum
ffmpeg -i converted.flac -c:a pcm_s24le -t 5 -f s24le - | md5sum
```

### 13.2 FFprobe Bit-Depth Verification

```bash
# Get detailed stream information
ffprobe -v error -show_streams -show_format input.flac

# Quick bit depth check
ffprobe -v error -select_streams a:0 \
  -show_entries stream=bits_per_raw_sample,codec_name,sample_rate \
  -of default=noprint_wrappers=1 input.flac

# Expected output for 24-bit/96kHz:
# bits_per_raw_sample=24
# codec_name=flac
# sample_rate=96000
```

---

## 14. METADATA AND HIGH-RESOLUTION FLAGS

### 14.1 Signaling High-Resolution in Containers

Different containers signal hi-res differently:

| Container | Method | Example |
|-----------|--------|---------|
| FLAC | `STREAMINFO` block bits-per-sample | `bits_per_sample: 24` |
| WAV | `fmt` chunk wBitsPerSample | `wBitsPerSample: 24` |
| AIFF | `COMM` chunk sampleSize | `sampleSize: 24` |
| MP4/M4A | `escal` atom | Bit depth in ES descriptor |
| Matroska/MKA | `Audio` element | `bitDepth: 24` |

### 14.2 Preserving Hi-Res Metadata

```bash
# Copy all metadata
ffmpeg -i input.wav -c:a flac -sample_fmt s32 \
  -map_metadata 0 \
  -map_flags +copy_metadata \
  output.flac

# Copy specific tags
ffmpeg -i input.wav -c:a alac -sample_fmt s32 \
  -metadata title="Album Title" \
  -metadata artist="Artist Name" \
  -metadata album="Album" \
  -metadata date="2024" \
  -metadata track="1" \
  -metadata genre="Classical" \
  output.m4a
```

---

## 15. KNOWN ISSUES AND EDGE CASES

### 15.1 Common FFmpeg Hi-Res Audio Issues

| Issue | Symptom | Cause | Solution |
|-------|---------|-------|----------|
| 24-bit shown as 32-bit | `sample_fmt=s32` for 24-bit file | FFmpeg uses 32-bit container | Use `bits_per_raw_sample` to verify actual depth |
| Upsampling noise | Artifacts after upsampling | Low-quality resampling | Use SoX resampler with precision=28 |
| DSD decode failure | "Decoder not found" | FFmpeg built without DSD support | Rebuild with full DSD decoder support |
| Metadata loss | Tags missing after conversion | Incompatible tag systems | Use `-map_metadata 0` or tag-specific options |
| Clipping in float | Distorted output | Float values outside [-1, +1] | Apply normalization filter |

### 15.2 Precision Loss Scenarios

| Conversion | Precision Risk | Mitigation |
|------------|---------------|------------|
| 24-bit → 16-bit | High (8 bits lost) | Always dither |
| 32-bit float → 24-bit | Medium | Apply dithering |
| 32-bit float → 32-bit float | None | No action needed |
| 24-bit → 24-bit (same) | None | No action needed |
| DSD64 → PCM@352.8k | Low | Lowpass filter for ultrasonic noise |
| 384kHz → 96kHz | High | Use high-quality resampling with dithering |

### 15.3 [NEEDS VERIFICATION] Platform-Specific Behaviors

- **Apple Silicon:** Native DSD decoding may require Rosetta 2 or native FFmpeg build
- **Windows:** Some DSD formats may require additional CODECs installed
- **Linux:** DSD-over-USB requires ALSA with DSD support or DoP mode

---

## 16. REFERENCE COMMANDS QUICK REFERENCE

### 16.1 Common High-Resolution Tasks

```bash
# 1. Convert any audio to 24-bit/96kHz FLAC
ffmpeg -i input -c:a flac -sample_fmt s32 -ar 96000 -compression_level 8 output.flac

# 2. Convert DSD to hi-res PCM FLAC
ffmpeg -i input.dsf -c:a dsd_msbf -ar 352800 -sample_fmt s32 -c:a flac -compression_level 8 output.flac

# 3. High-quality downsampling with dithering
ffmpeg -i input_192k.wav -af "aresample=resampler=soxr:precision=28:osf=s16:dither_method=shibata" -ar 44100 output_44k.wav

# 4. Preserve original hi-res format
ffmpeg -i input.flac -c:a copy output.wav

# 5. Verify hi-res encoding
ffprobe -v error -select_streams a:0 -show_entries stream=bits_per_raw_sample,sample_rate -of default=noprint_wrappers=1 output.flac
```

### 16.2 Format Detection

```bash
# Detect actual bit depth (not container size)
ffprobe -v error -select_streams a:0 \
  -show_entries stream=bits_per_raw_sample,bits_per_code_sample,sample_fmt \
  -of default=noprint_wrappers=1 input.flac

# For 24-bit FLAC: bits_per_raw_sample=24
# For 16-bit WAV: bits_per_raw_sample=16
# For 24-bit WAV: bits_per_raw_sample=24 (even though container is 32-bit)
```

---

## 17. BUILD CONFIGURATION REFERENCE

### 17.1 Configure Flags for High-Resolution Audio

```bash
# Essential flags for hi-res audio support
./configure \
  --enable-gpl \
  --enable-version3 \
  --enable-nonfree \
  --enable-libsoxr \
  --enable-libflac \
  --enable-libvorbis \
  --enable-libopus \
  --enable-libfdk-aac \
  --enable-libmp3lame

# Check build configuration
ffmpeg -buildconf 2>&1 | grep -E "(soxr|flac|vorbis|opus)"
```

### 17.2 Verifying FFmpeg Capabilities

```bash
# Check supported encoders
ffmpeg -encoders 2>/dev/null | grep -E "(FLAC|ALAC|pcm_)" 

# Check supported decoders
ffmpeg -decoders 2>/dev/null | grep -E "(dsd|dsf|dff|flac)"

# Check filters
ffmpeg -filters 2>/dev/null | grep -E "(aresample|aformat|pan|volume)"

# Check sample formats
ffmpeg -formats 2>/dev/null | grep -i "s32\|fltp"
```

---

## 18. IMPLEMENTATION CHECKLIST

### Build & Environment
- [ ] Verify FFmpeg supports desired sample formats (`ffmpeg -formats`)
- [ ] Check SoX resampler availability (`ffmpeg -buildconf | grep soxr`)
- [ ] Test DSD decoder if needed (`ffmpeg -decoders | grep dsd`)

### High-Resolution Encoding
- [ ] Set correct `sample_fmt` (s32/s32p for 24-bit, fltp for float)
- [ ] Verify sample rate is accepted by encoder
- [ ] Use maximum compression level for lossless codecs
- [ ] Apply dithering when reducing bit depth

### DSD Processing
- [ ] Identify DSD format (DSF vs DFF)
- [ ] Apply lowpass filter if converting to PCM (typically 20-50kHz cutoff)
- [ ] Handle volume adjustment for DSD→PCM (typically +6dB)
- [ ] Verify output sample rate matches DSD-to-PCM ratio

### Quality Verification
- [ ] Use `bits_per_raw_sample` to verify actual bit depth
- [ ] Compare framemd5 checksums for lossless verification
- [ ] Check output file size is reasonable (FLAC 24-bit ≈ 1.4× 16-bit)

### Edge Cases
- [ ] Handle files with 0 samples
- [ ] Handle corrupted DSD headers
- [ ] Handle unsupported sample rates (use resampling)
- [ ] Handle floating-point overflow (clipping prevention)

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
