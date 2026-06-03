# MP3 (MPEG-1/2 Audio Layer III) Stream — Deep Technical Reference

> **Category:** Container
> **File Extensions:** `.mp3`, `.mp2` (MPEG-1 Layer II), `.mp1` (MPEG-1 Layer I)
> **MIME Types:** `audio/mpeg`, `audio/mp3`, `audio/x-mpeg`, `audio/mpeg3`
> **Standardization Body:** ISO/IEC, ITU-T (MPEG-1 Audio)
> **Primary Specification:** ISO/IEC 11172-3 (MPEG-1 Audio Layer III), ISO/IEC 13818-3 (MPEG-2)
> **Patent Status:** Patented — various patent pools (MP3.com, Fraunhofer, others)
> **License:** Royalty-bearing for commercial encoding (LAME is free)
> **Current Version:** MPEG-1 Layer III (1993), MPEG-2 Layer III (1998)
> **Active Development:** No — format finalized

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Moving Picture Experts Group (MPEG), under ISO/IEC
- **Year Created:** 1993 (MPEG-1 Audio Layer III); 1998 (MPEG-2 extension)
- **Original Purpose:** Create a high-quality digital audio compression format suitable for storage and transmission, achieving near-CD quality at ~128 kbps (12:1 compression)
- **Problem with Predecessors:** Earlier formats (MUSICAM, ASPEC) had licensing or quality issues; MP3 became dominant due to its quality, widespread adoption, and free decoder implementations

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| MPEG-1 Audio Layer III | 1993 | Original MP3 standard, 32–320 kbps |
| MPEG-2 Layer III | 1995 | Half sample rate extension, multichannel |
| MPEG-2.5 | 1998 | Unofficial extension for very low sample rates |
| ID3v1 | 1996 | Fixed 128-byte tag at end |
| ID3v2 | 1998 | Variable-length tag at beginning |
| LAME tag | 2000+ | Extended info tag in MP3 |

### 1.3 Current Adoption
- **Primary use cases today:** Music distribution, streaming, podcasts, internet radio
- **Platforms with native support:** Universal — every OS and device
- **Major services using this format:** Spotify, YouTube Music, internet radio, podcasts
- **Hardware support:** Universal
- **Status:** Dominant legacy — still the most widely supported audio format

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Transform coding (hybrid)
- **Core algorithm:** Modified Discrete Cosine Transform (MDCT) with Layer III-specific preprocessing
- **Loss mechanism:** Psychoacoustic masking — bits allocated to spectral components based on perceptual importance
- **Frame-based vs sample-based:** Frame-based — fixed number of samples per frame
- **Fixed vs variable frame size:** Fixed — 1152 samples (MPEG-1), 576 samples (MPEG-2/2.5)

### 2.2 High-Level Encoding Flow (Block Diagram)
```
Input PCM Samples (1152 samples per frame for MPEG-1 Layer III)
      │
      ▼
[Polyphase Filterbank: 32 subbands]
      │
      ▼
[Non-linear Quantization: Power spectral representation]
      │
      ▼
[Psychoacoustic Model: Masking threshold calculation]
      │
      ▼
[Bit Allocation: Distribute bits to subbands]
      │
      ▼
[Huffman Coding: Lossless entropy coding]
      │
      ▼
[Frame Packing: Header + side info + main data + CRC]
      │
      ▼
Output MP3 Frame (4 + side_info + main_data bytes)
```

### 2.3 Key Design Parameters
| Parameter | MPEG-1 | MPEG-2 | MPEG-2.5 | Notes |
|-----------|--------|--------|----------|-------|
| Sample rates | 32, 44.1, 48 kHz | 16, 22.05, 24 kHz | 8, 11.025, 12 kHz | |
| Frame size | 1152 samples | 576 samples | 576 samples | |
| Bitrates | 32–320 kbps | 8–160 kbps | 8–160 kbps | |
| Channels | 1–2 | 1–2 | 1–2 | No multichannel |
| Layers | I, II, III | I, II, III | III only | |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
```
MP3 file has no global magic. Each frame begins with a sync word.
First frame may be an "info frame" (XING/LAME header).

ID3v2 header at beginning (optional):
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  3       49 44 33         'ID3'    ID3v2 magic
0x0003  2       XX XX            —         Version (e.g., 04 00 = ID3v2.4.0)
0x0005  1       XX               —         Flags
0x0006  4       XX XX XX XX      —         Size (synch-safe integer)

First MP3 frame sync:
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       FF FA/FB/FA/F3  —         Frame sync word
0x0000  4       FF E/FF F        —         MPEG-2.5 sync variants
```

### 3.2 Frame Header Layout (32 bits)
```
Bit Layout (MSB first):
Offset   Bits    Field Name              Type      Description
-------  -----   --------------------   ------    ---------------------------
0        11      Frame Sync              uint      0x7FF (all ones)
11       2       MPEG Version           uint      00=MPEG 2.5, 10=MPEG 2, 11=MPEG 1
13       2       Layer                  uint      00=reserved, 01=Layer III, 10=Layer II, 11=Layer I
15       1       Protection Bit         bool      0=CRC present, 1=no CRC
16       4       Bitrate Index          uint      See bitrate table
20       2       Sample Rate Index      uint      See sample rate table
22       1       Padding Bit            bool      0=no padding, 1=padding byte
23       1       Private Bit            bool      Free for use
24       2       Channel Mode           uint      00=Stereo, 01=Joint Stereo, 10=Dual Mono, 11=Mono
26       2       Mode Extension         uint      Joint stereo bands (if applicable)
28       1       Copyright Bit          bool      1=copyrighted
29       1       Original/Copy Bit       bool      1=original
30       2       Emphasis               uint      De-emphasis (00=none, 01=50/15μs, 11=CCITT J.17)

Byte-by-byte layout (big-endian):
Byte 0: 11111111  (sync byte, always 0xFF)
Byte 1: 111XXXXX  (bits 0-4 of rest of header)
Byte 2: XXXXXXXX
Byte 3: XXXXXXXX
```

### 3.3 Bitrate Index Table
| Index | MPEG-1 Layer I | MPEG-1 Layer II | MPEG-1 Layer III | MPEG-2 Layer III |
|-------|----------------|-----------------|------------------|-------------------|
| 0x0 | Free bitrate | Free bitrate | Free bitrate | Free bitrate |
| 0x1 | 32 kbps | 32 kbps | 32 kbps | 8 kbps |
| 0x2 | 64 kbps | 48 kbps | 40 kbps | 16 kbps |
| 0x3 | 96 kbps | 56 kbps | 48 kbps | 24 kbps |
| 0x4 | 128 kbps | 64 kbps | 56 kbps | 32 kbps |
| 0x5 | 160 kbps | 80 kbps | 64 kbps | 40 kbps |
| 0x6 | 192 kbps | 96 kbps | 80 kbps | 48 kbps |
| 0x7 | 224 kbps | 112 kbps | 96 kbps | 56 kbps |
| 0x8 | 256 kbps | 128 kbps | 112 kbps | 64 kbps |
| 0x9 | 288 kbps | 160 kbps | 128 kbps | 80 kbps |
| 0xA | 320 kbps | 192 kbps | 160 kbps | 96 kbps |
| 0xB | 352 kbps | 224 kbps | 192 kbps | 112 kbps |
| 0xC | 384 kbps | 256 kbps | 224 kbps | 128 kbps |
| 0xD | 416 kbps | 320 kbps | 256 kbps | 144 kbps |
| 0xE | 448 kbps | 384 kbps | 320 kbps | 160 kbps |
| 0xF | Bad | Bad | Bad | Bad |

### 3.4 Sample Rate Index Table
| Index | MPEG-1 | MPEG-2 | MPEG-2.5 |
|-------|--------|--------|----------|
| 0x0 | 44100 Hz | 22050 Hz | 11025 Hz |
| 0x1 | 48000 Hz | 24000 Hz | 12000 Hz |
| 0x2 | 32000 Hz | 16000 Hz | 8000 Hz |
| 0x3 | Reserved | Reserved | Reserved |

### 3.5 Frame Size Calculation
```
MPEG-1, Layer III:
  FrameSize = 144 × Bitrate / SampleRate + PaddingBit

MPEG-2/2.5, Layer III:
  FrameSize = 72 × Bitrate / SampleRate + PaddingBit

MPEG-1, Layer I:
  FrameSize = 48 × Bitrate / SampleRate + PaddingBit

MPEG-1, Layer II:
  FrameSize = 144 × Bitrate / SampleRate + PaddingBit

Example: MPEG-1 Layer III, 128 kbps, 44100 Hz, no padding
  FrameSize = 144 × 128000 / 44100 = 417.09 → 417 bytes

Example: MPEG-1 Layer III, 128 kbps, 44100 Hz, with padding
  FrameSize = 144 × 128000 / 44100 + 1 = 418 bytes
```

### 3.6 XING / VBRI / LAME Header Layout

#### XING Header Location
```
The XING header is placed in the first MP3 frame, after side information.
Location depends on MPEG version and channel mode:

MPEG-1, Stereo/Joint/Dual:  Offset 36 from frame start (after 4-byte header + 32 bytes side info)
MPEG-1, Mono:               Offset 21 from frame start
MPEG-2, Stereo/Joint/Dual:  Offset 21 from frame start
MPEG-2, Mono:               Offset 13 from frame start
```

#### XING Header Format
```
Offset   Size    Field Name              Type        Description
-------  ------  ---------------------  ----------  ---------------------------
0x0000   4B      XING/LAME ID           char[4]     'Xing' or 'Info' or 'LAME'
0x0004   4B      Flags                  uint32 BE   Bitfield indicating present fields
0x0008   4B      Frames                 uint32 BE   Total frames (if flag & 0x01)
0x000C   4B      Bytes                  uint32 BE   Total bytes (if flag & 0x02)
0x0010   100B    TOC                    uint8[100]  Table of Contents (if flag & 0x04)
0x0074   4B      VBR Scale              uint32 BE   VBR quality (if flag & 0x08)
```

#### LAME Tag Extension (after XING header)
```
Offset   Size    Field Name              Type        Description
-------  ------  ---------------------  ----------  ---------------------------
0x0078   9B      LAME Version           char[9]     "LAME3.99r" etc.
0x0081   1B      LAME Revision          uint8       Revision number
0x0082   1B      VBR Method            uint8       VBR type
0x0083   1B      Lowpass Filter         uint8       Lowpass cutoff × 100 Hz
0x0084   4B      ReplayGain Track       uint32      Radio Replay Gain (see spec)
0x0088   2B      ReplayGain Album       uint16      Album Replay Gain
0x008A   1B      Flags                 uint8       Encoder flags
0x008B   1B      Ath Type              uint8       ATH type
0x008C   1B      Flags                  uint8       More flags
0x008D   1B      Delay Padding High     uint8       Encoder delay high bits
0x008E   1B      Delay Padding Low     uint8       Encoder delay + padding low bits
0x008F   1B      Padding Samples        uint8       Padding sample count
0x0090   1B      Misc                   uint8       Miscellaneous
0x0091   1B      Stereo Mode           uint8       Stereo mode used
0x0092   1B      LAME Tag Revision     uint8       Tag revision
0x0093   2B      Sample Frequency      uint16      Source sample frequency
0x0095   3B      CD Audio Buffer        uint24      For gapless playback
0x0098   2B      LAME Tag CRC          uint16      CRC of LAME tag
```

### 3.7 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Unsigned integer | Yes | Rarely used |
| 16-bit | Signed integer | Yes | Standard input/output |
| 20-bit | Signed integer | No | Not in spec |
| 24-bit | Signed integer | No | Not in spec |
| 32-bit | IEEE float | No | Not in spec |

#### Supported Sample Rates
| Rate (Hz) | MPEG-1 | MPEG-2 | MPEG-2.5 | Notes |
|-----------|--------|--------|----------|-------|
| 8000 | No | No | Yes | Low rate extension |
| 11025 | No | No | Yes | Low rate extension |
| 12000 | No | No | Yes | Low rate extension |
| 16000 | No | Yes | Yes | MPEG-2 half rate |
| 22050 | No | Yes | Yes | MPEG-2 half rate |
| 24000 | No | Yes | Yes | MPEG-2 half rate |
| 32000 | Yes | Yes | Yes | MPEG-1 and 2 |
| 44100 | Yes | No | No | CD audio |
| 48000 | Yes | No | No | Professional |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Pre-Processing Stage
- **DC offset removal:** Optional, encoder-dependent
- **Pre-filtering:** Optional anti-aliasing
- **Windowing:** DCT window before MDCT
- **Level normalization:** Optional peak normalization

### 4.2 Analysis / Transform Stage

#### Polyphase Filterbank + MDCT
```
Layer III uses two filter stages:
1. Polyphase filterbank: 32 equal-width subbands
2. MDCT: 18-point (short) or 36-point (long) within each subband

Parameters:
  Polyphase filter: 512-tap FIR filter
  MDCT window:      36-point (long block) or 18-point (short block)
  Window overlap:    50% (MDCT standard)
  Block types:       Long, Short, Start, Stop
```

### 4.3 Psychoacoustic Model (Perceptual Coding)

#### Masking Thresholds
```
Absolute Threshold of Hearing (ATH):
  ATH(f) = 3.64·(f/1000)^(-0.8) - 6.5·exp(-0.6·(f/1000 - 3.3)^2) + 10^(-3)·(f/1000)^4 dB

Simultaneous Masking:
  - Upward masking: ~10 dB/Bark (less critical)
  - Downward masking: ~40 dB/Bark (more critical)

Temporal Masking:
  - Pre-masking: ~20 ms (before masker)
  - Post-masking: ~100–200 ms (after masker)
```

### 4.4 Bit Allocation Algorithm
```
1. Compute FFT of input signal
2. Apply psychoacoustic model to compute masking threshold per subband
3. Determine allowed distortion per band
4. Outer iteration loop (SHORT/SHORT):
     a. Inner iteration loop:
          - Quantize with current step
          - Count bits needed for Huffman coding
          - If bits > available, increase step size
     b. If bits < available, try smaller quantizer
     c. Check noise-to-mask ratio
     d. If not satisfied, repeat inner loop
5. Repeat outer loop for long blocks
```

### 4.5 Stereo Encoding Modes
| Mode | Description | Bitrate Overhead | Quality |
|------|-------------|------------------|---------|
| Stereo | L and R encoded independently | Full stereo | Best |
| Joint Stereo (MS) | Mid = (L+R)/2, Side = (L-R)/2 | Same as stereo | Slight loss |
| Intensity Stereo | High freq as magnitude + position | Significant saving | Degraded |
| Dual Mono | L and R completely separate | Full stereo | Same as stereo |

### 4.6 Entropy Coding
```
Method: Huffman coding with multiple codebooks

Huffman codebook selection:
  - 34 codebooks total (MPEG-1 Layer III)
  - Selection based on max absolute value in region
  - Escape codes for out-of-range values
  - "Part2_3_length" determines which codebook per region
```

### 4.7 Encoder Settings / Quality Modes

#### For Lossy MP3
| Quality Setting | Bitrate Range | Intended Use Case | Transparent? |
|----------------|---------------|-------------------|-------------|
| Lowest | 32–64 kbps | Voice, low bandwidth | No |
| Low | 96 kbps | Casual listening | No |
| Medium | 128 kbps | General use | Mostly |
| High | 192 kbps | Good quality | Yes (most content) |
| Highest | 256–320 kbps | Audiophile | Yes |

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. Scan for 0xFF byte in bitstream
2. Validate candidate header:
     a. Check sync word (11 bits = 1)
     b. Check MPEG version (not 01)
     c. Check layer (not 00)
     d. Check bitrate (not 1111)
     e. Check sample rate (not 11)
3. If validation fails, advance 1 byte and retry
4. Validate next frame at computed offset
5. Max failed sync attempts: recommend 4096 bytes
```

#### Seeking
```
CBR seeking:
  byte_offset = (target_time_sec × bitrate × 1000) / 8

VBR seeking:
  Use TOC from XING/LAME header
  TOC[i] = byte position for seek point i (0–99)
  Seeking: binary search TOC, then scan to exact position

FFmpeg seeking:
  av_seek_frame() handles both CBR and VBR
```

### 5.2 Core Decode Pipeline
```
1. Read frame header (4 bytes)
     ├── Verify sync word (0xFF in first 11 bits)
     ├── Parse: version, layer, bitrate, sample rate, channel mode
     └── Compute frame_size = 144 × bitrate / sample_rate + padding

2. Read side information (32 bytes MPEG-1 stereo)
     ├── Part2_3_length for each granule
     ├── Big_values, count1, global_gain
     ├── Scalefac_compress, blocksplit_flag
     ├── Table select, region0/1_address
     └── Pre-flag, scalefac_scale, count1table_select

3. Read main data (variable)
     ├── Read scale factors (if not block_type == 2 && mixed == 0)
     ├── Read Huffman-coded spectral data
     └── Reorder short blocks

4. Requantize
     Requantized[i] = (|Huffman[i]| / 2^(4 × (global_gain - 210)))^(4/3)

5. Stereo processing
     ├── MS: L = (M + S) / √2; R = (M - S) / √2
     └── Intensity: combine high-freq bands

6. Inverse MDCT
     └── 36-point (long) or 18-point (short) inverse MDCT

7. Overlap-add
     └── Add previous block's second half to current output

8. Apply synthesis filterbank
     └── 32-subband polyphase synthesis filter

9. Reorder short blocks
     └── Frequency ordering within each granule

10. Output 1152 samples per frame (MPEG-1), 576 (MPEG-2)
```

### 5.3 Error Concealment
- **Corrupt frame detection:** CRC check, sync word failure
- **Concealment method:** Repeat last good frame or mute
- **Maximum consecutive errors before silence:** Decoder-dependent, typically 3–5 frames

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** MP3 is a raw bitstream format
- **Overhead:** 4 bytes per frame (header) + optional CRC (2 bytes)
- **Seeking in native container:** Requires TOC or frame scanning
- **Multiple streams in native container:** No

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| MP3 (.mp3) | MP3 | Yes (TOC) | ID3v1, ID3v2 | Native |
| WAV | MP3 | No | RIFF INFO | data chunk |
| AVI | MP3 | No | AVI metadata | audio stream |
| Matroska/MKA | MP3 | Yes | Vorbis Comments | via webm |
| OGG | MP3 | Yes | Vorbis Comments | Not recommended |
| MP4/M4A | MP3 | Yes | iTunes atoms | Supported |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** ID3v2 (at beginning) + ID3v1 (at end)
- **Tag block location:** Beginning (ID3v2) and end (ID3v1)
- **VBR info location:** First frame, after side information

### 7.2 Standard Tag Fields — ID3v2
| Tag Field | Internal Key | Max Length | Character Encoding | Multi-value | Notes |
|-----------|-------------|------------|-------------------|-------------|-------|
| Title | TIT2 | variable | UTF-16, UTF-8 | No | |
| Artist | TPE1 | variable | UTF-16, UTF-8 | Yes | |
| Album | TALB | variable | UTF-16, UTF-8 | No | |
| Album Artist | TPE2 | variable | UTF-16, UTF-8 | Yes | |
| Composer | TCOM | variable | UTF-16, UTF-8 | Yes | |
| Genre | TCON | variable | UTF-16, UTF-8 | No | ID3v1 genre number also used |
| Year / Date | TDRC | 4 bytes | ASCII | No | YYYY format |
| Track Number | TRCK | variable | ASCII | No | Format "N" or "N/Total" |
| Disc Number | TPOS | variable | ASCII | No | Format "N" or "N/Total" |
| Comment | COMM | variable | UTF-16, UTF-8 | No | With language + description |
| Lyrics | USLT | variable | UTF-16, UTF-8 | No | Synchronized or unsynced |
| Cover Art | APIC | up to 10 MB | Binary | No | JPEG/PNG |
| BPM | TBPM | variable | ASCII | No | |
| Compilation | TCMP | 1 byte | Binary | No | Boolean |
| Copyright | TCOP | variable | UTF-16, UTF-8 | No | |
| Encoder | TENC | variable | UTF-16, UTF-8 | No | Software name |
| ISRC | TSRC | 12 bytes | ASCII | No | |
| ReplayGain Track Gain | TXXX:REPLAYGAIN_TRACK_GAIN | variable | ASCII | No | Format "-6.20 dB" |
| ReplayGain Track Peak | TXXX:REPLAYGAIN_TRACK_PEAK | variable | ASCII | No | Format "0.998459" |

### 7.3 Cover Art Storage in MP3 (ID3v2)
```
ID3v2 APIC frame structure:
Offset   Size    Field Name              Type        Description
-------  ------  ---------------------  ----------  ---------------------------
0x0000   4B      Frame ID               char[4]     'APIC'
0x0004   4B      Frame Size            uint32      Size excluding this header
0x0008   2B      Frame Flags           uint16      Flags
0x000A   1B      Text Encoding         uint8       0=Latin-1, 1=UTF-16, 2=UTF-16BE, 3=UTF-8
0x000B   N       MIME Type            char[N]     'image/jpeg' or 'image/png', null-terminated
0x000C   1B      Picture Type         uint8       0–20, 3=Front cover
0x000D   N       Description          char[N]    Null-terminated
0x000E   M       Image Data           byte[M]     Binary JPEG/PNG data
```

### 7.4 Metadata Compatibility Matrix
| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| ID3v2.4 | ✓ | ✓ | ✓ | Highest |
| ID3v2.3 | ✓ | ✓ | ✓ | High |
| ID3v1 | ✓ | ✓ | Limited | Low |
| APEv2 | ✓ | ✓ | ✓ | Medium |

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   libmp3lame (LAME MP3 encoder)
                    mp3 (native decoder)
AV_CODEC_ID:       AV_CODEC_ID_MP3
Format Name (CLI):  mp3 (native format)
Encoder(s):         libmp3lame, mp3fixed
Decoder(s):         mp3 (native)
Muxer(s):          mp3, ipod (MP4)
Demuxer(s):        mp3, aiff (if MP3 in AIFF), avi, matroska
```

### 8.2 FFmpeg Encoding — Full CLI Reference

```bash
# Encode to MP3 using LAME
ffmpeg -i input.wav \
  -c:a libmp3lame \
  -q:a 2 \
  output.mp3

# Encode CBR MP3
ffmpeg -i input.wav \
  -c:a libmp3lame \
  -b:a 192k \
  -compression_level 0 \
  output.mp3

# Encode VBR MP3 (high quality)
ffmpeg -i input.wav \
  -c:a libmp3lame \
  -q:a 0 \
  -id3v2_version 3 \
  output.mp3

# Encode with LAME tag
ffmpeg -i input.wav \
  -c:a libmp3lame \
  -q:a 2 \
  -write_xing 1 \
  -metadata title="Song Title" \
  -metadata artist="Artist" \
  output.mp3
```

#### Complete FFmpeg Option Table
| Option | Type | Default | Range / Valid Values | Effect on Output |
|--------|------|---------|---------------------|-----------------|
| `-q:a` | float | 2.0 | 0–9.0 (lower=better) | VBR quality setting |
| `-b:a` | int | 128k | 32k–320k | CBR bitrate |
| `-compression_level` | int | 0 | 0–9 | Encoding effort (not LAME) |
| `-id3v2_version` | int | 0 | 0, 2, 3, 4 | ID3v2 version |
| `-write_xing` | int | 1 | 0, 1 | Write XING header |
| `-write_id3v1` | int | 1 | 0, 1 | Write ID3v1 tag |

### 8.3 FFmpeg Encoding — libavcodec C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/opt.h>

// ─── 1. Find LAME encoder ───────────────────────────────────────────────────
const AVCodec *codec = avcodec_find_encoder_by_name("libmp3lame");
if (!codec) { fprintf(stderr, "LAME encoder not found\n"); exit(1); }

// ─── 2. Allocate codec context ───────────────────────────────────────────────
AVCodecContext *ctx = avcodec_alloc_context3(codec);
ctx->bit_rate = 192000;
ctx->sample_fmt = AV_SAMPLE_FMT_S16P;
ctx->sample_rate = 44100;
av_channel_layout_copy(&ctx->ch_layout, &(AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO);

// ─── 3. Open codec ───────────────────────────────────────────────────────────
avcodec_open2(ctx, codec, NULL);

// ─── 4. Allocate frame and packet ──────────────────────────────────────────
AVFrame *frame = av_frame_alloc();
frame->format = ctx->sample_fmt;
frame->sample_rate = ctx->sample_rate;
av_channel_layout_copy(&frame->ch_layout, &ctx->ch_layout);
frame->nb_samples = ctx->frame_size;
av_frame_get_buffer(frame, 0);

AVPacket *pkt = av_packet_alloc();

// ─── 5. Encode loop ─────────────────────────────────────────────────────────
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

// ─── 6. Cleanup ─────────────────────────────────────────────────────────────
av_frame_free(&frame);
av_packet_free(&pkt);
avcodec_free_context(&ctx);
```

### 8.4 FFmpeg Decoding — Full CLI Reference

```bash
# Decode MP3 to WAV
ffmpeg -i input.mp3 \
  -c:a pcm_s16le \
  output.wav

# Decode and resample
ffmpeg -i input.mp3 \
  -ar 48000 \
  -c:a pcm_s16le \
  output.wav

# Decode with downmix
ffmpeg -i input.mp3 \
  -ac 2 \
  -c:a pcm_s16le \
  output.wav

# Extract specific stream
ffmpeg -i input.mp3 -map 0:a:0 -c:a copy output.mp3

# Probe format
ffprobe -v quiet -print_format json -show_streams -show_format input.mp3
```

### 8.5 FFmpeg Decoding — libavformat C API

```c
// ─── Complete decode program skeleton ────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.mp3", NULL, NULL);
avformat_find_stream_info(fmt_ctx, NULL);

int audio_idx = av_find_best_stream(fmt_ctx, AVMEDIA_TYPE_AUDIO, -1, -1, NULL, 0);
AVStream *stream = fmt_ctx->streams[audio_idx];

const AVCodec *dec = avcodec_find_decoder(stream->codecpar->codec_id);
AVCodecContext *dec_ctx = avcodec_alloc_context3(dec);
avcodec_parameters_to_context(dec_ctx, stream->codecpar);
avcodec_open2(dec_ctx, dec, NULL);

AVPacket *pkt = av_packet_alloc();
AVFrame  *frm = av_frame_alloc();

while (av_read_frame(fmt_ctx, pkt) >= 0) {
    if (pkt->stream_index == audio_idx) {
        avcodec_send_packet(dec_ctx, pkt);
        while (avcodec_receive_frame(dec_ctx, frm) == 0) {
            // frm->data[0] contains PCM samples
            // frm->nb_samples = sample count per channel
            // frm->sample_rate = actual rate
            process_audio_frame(frm);
            av_frame_unref(frm);
        }
    }
    av_packet_unref(pkt);
}

avcodec_send_packet(dec_ctx, NULL);
while (avcodec_receive_frame(dec_ctx, frm) == 0) {
    process_audio_frame(frm);
    av_frame_unref(frm);
}

av_frame_free(&frm);
av_packet_free(&pkt);
avcodec_free_context(&dec_ctx);
avformat_close_input(&fmt_ctx);
```

### 8.6 FFmpeg Metadata Handling

```bash
# Read all metadata as JSON
ffprobe -v quiet -print_format json -show_format input.mp3 | jq .format.tags

# Write metadata
ffmpeg -i input.mp3 \
  -c copy \
  -metadata title="Song Title" \
  -metadata artist="Artist Name" \
  -metadata album="Album Name" \
  -metadata date="2024" \
  -metadata track="5/12" \
  output.mp3

# Embed cover art
ffmpeg -i input.mp3 -i cover.jpg \
  -c copy \
  -map 0:a -map 1:v \
  -metadata:s:v title="Album cover" \
  output.mp3

# Strip all metadata
ffmpeg -i input.mp3 -c copy -map_metadata -1 -id3v2_version 0 output.mp3
```

### 8.7 Quality / Fidelity Decision Table
| Use Case | Settings | Expected Size | Notes |
|----------|----------|----------------|-------|
| Archival / lossless | Not applicable | N/A | Use FLAC |
| Audiophile streaming | `-q:a 0` (VBR) | ~210–260 kbps | Transparent |
| High quality | `-q:a 2` (VBR) | ~180–210 kbps | Excellent |
| Standard streaming | `-b:a 192k` | 192 kbps | Good |
| Low bandwidth | `-b:a 96k` | 96 kbps | Acceptable |
| Voice / podcast | `-b:a 64k` | 64 kbps | Voice optimized |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
```
XING TOC (Table of Contents):
  Location:    First frame, after XING header
  Entry count: 100 entries
  Entry size:  1 byte each
  Entry format:
    TOC[i] = byte position for seek point i/100 of file
  Use: Seek by binary searching TOC, then scan to exact position

VBRI Header (Fraunhofer):
  Location:    First frame, offset 32 from header
  Entry size:  Variable
  Format:     Frame-based index table
```

### 9.2 Gapless Playback Data
```
Encoder delay:   576 samples (padding) — stored in LAME tag as iTunSMPB
Padding:         576 samples (padding) — stored in LAME tag as iTunSMPB
Storage:        LAME tag field iTunSMPB
Example value:   " 00000000 00000B40 000001E0 00000000 000001E0 00000480"

FFmpeg gapless: Input file with LAME tag → FFmpeg preserves delay/padding
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable | Yes | Frame-by-frame decode |
| Algorithmic delay | ~26 ms | 1152 samples at 44.1 kHz |
| Live encoding | Yes | LAME supports live encoding |
| HTTP progressive download | Yes | With XING TOC |
| HTTP Live Streaming (HLS) | Yes | Via FFmpeg HLS muxer |
| WebRTC | Yes | Opus now preferred |
| RTP payload | Yes | RFC 2250 |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | Layout Name | MPEG-1 | MPEG-2 | Notes |
|----------|-------------|--------|--------|-------|
| 1 | Mono | ✓ | ✓ | Standard |
| 2 | Stereo | ✓ | ✓ | Standard |
| 2 | Joint Stereo (MS) | ✓ | ✓ | Bit-saving |
| 2 | Dual Mono | ✓ | ✓ | Independent mono |

### 11.2 Downmix Coefficients (Pro Logic compatible)
```
L_out = 1.0 × L + 0.7071 × C
R_out = 1.0 × R + 0.7071 × C
LFE: discarded
```

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 16-bit input | Output is 16-bit PCM |
| Max sample rate | 48,000 Hz | MPEG-1 max |
| Float support | No | Fixed-point decode |
| DSD support | No | Not applicable |

---

## 13. KNOWN ISSUES, BUGS & EDGE CASES

### 13.1 FFmpeg Issues
| Issue | Versions Affected | Workaround |
|-------|------------------|------------|
| ID3v2.4 unsync | < 4.1 | Use `-id3v2_version 3` |
| Gapless with LAME | All | Use LAME tag, verify delay |
| Free bitrate | All | Specify explicit bitrate |

### 13.2 Edge Cases
- **All-silence:** Valid MP3, many frames with zero main_data
- **ID3v1 at both ends:** Some players read only one
- **Truncated file:** Decode available frames, report incomplete
- **Invalid frame:** Skip and continue with next sync

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM MP3
| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → FLAC | `ffmpeg -i in.mp3 -c:a flac -compression_level 8 out.flac` | Via tag mapping | Lossless decode |
| → WAV | `ffmpeg -i in.mp3 -c:a pcm_s16le out.wav` | Limited | Lossless decode |
| → AAC | `ffmpeg -i in.mp3 -c:a aac -b:a 256k out.m4a` | Tags via metadata | Re-encode |
| → Opus | `ffmpeg -i in.mp3 -c:a libopus -b:a 128k out.opus` | Via comment | Re-encode |

### 15.2 Converting TO MP3
| Source | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| FLAC → | `ffmpeg -i in.flac -c:a libmp3lame -q:a 2 out.mp3` | Via ID3v2 | Generation loss |
| WAV → | `ffmpeg -i in.wav -c:a libmp3lame -q:a 0 out.mp3` | Via ID3v2 | Lossy |
| AAC → | `ffmpeg -i in.m4a -c:a libmp3lame -q:a 2 out.mp3` | Via ID3v2 | Generation loss |

### 15.3 Lossless Round-Trip Verification
```bash
# Decode MP3 to PCM
ffmpeg -i input.mp3 -c:a pcm_s16le -ar 44100 decoded.wav

# Compare checksums
ffmpeg -i original.wav -c:a pcm_s16le -f md5 -
ffmpeg -i decoded.wav -c:a pcm_s16le -f md5 -
# Will NOT match due to lossy re-encode
```

---

## 16. REFERENCE IMPLEMENTATIONS

| Library | Language | License | Quality | URL |
|---------|----------|---------|---------|-----|
| LAME | C | LGPL | Reference | lame.sourceforge.io |
| FFmpeg MP3 | C | LGPL 2.1+ | Good | ffmpeg.org |
| mpg123 | C | LGPL 2.1+ | Good | mpg123.de |
| libmad | C | GPL | Good | Underbit MAD |

---

## 17. RELEVANT SPECIFICATIONS

- **ISO/IEC 11172-3:** MPEG-1 Audio Layer I, II, III
- **ISO/IEC 13818-3:** MPEG-2 Audio
- **RFC 2250:** RTP Payload Format for MPEG-1/MPEG-2 Audio
- **ID3v2 Spec:** https://id3.org/Developer%20Information

---

## 18. IMPLEMENTATION CHECKLIST

### Build & Environment
- [ ] Verify LAME encoder: `ffmpeg -encoders | grep mp3`
- [ ] Verify MP3 decoder: `ffmpeg -decoders | grep mp3`

### Encoding Pipeline
- [ ] Select bitrate or quality setting
- [ ] Configure ID3v2 version
- [ ] Enable XING header for VBR seeking

### Decoding Pipeline
- [ ] Handle sync word search
- [ ] Parse frame header correctly
- [ ] Handle CRC checking
- [ ] Decode Huffman-coded data

### Metadata
- [ ] Read ID3v2.4 (preferred) or ID3v2.3
- [ ] Write ID3v2.4 with UTF-8 encoding
- [ ] Preserve cover art as APIC frame

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
