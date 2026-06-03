# MPEG-1 Audio Layer I — Deep Technical Reference
> **Category:** Lossy
> **File Extensions:** `.mp1`, `.mpa`
> **MIME Types:** `audio/mpeg`, `audio/x-mpeg`
> **Standardization Body:** ISO/IEC (MPEG)
> **Primary Specification:** ISO/IEC 11172-3:1993 (MPEG-1 Part 3)
> **Patent Status:** Patented — expired (most patents expired 2017–2020)
> **License:** Open / Royalty-free for most uses
> **Current Version:** ISO/IEC 11172-3:1993 (unchanged)
> **Active Development:** No — deprecated; successor is Layer II (MP2) and Layer III (MP3)

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** MPEG (Moving Picture Experts Group), ISO/IEC JTC1/SC29/WG11
- **Year Created:** 1993 (standard published as ISO/IEC 11172-3)
- **Original Purpose:** The simplest and lowest-complexity perceptual audio coder in the MPEG-1 family, designed for applications requiring real-time hardware encode and decode with minimal silicon area (consumer electronics, set-top boxes)
- **Problem with Predecessors:** Layer I was the first standardized perceptual audio codec. It was designed as a baseline for comparison — the simplest MPEG audio layer, establishing the QMF filter bank and psychoacoustic modeling framework that all three layers share. It was not intended as the final optimum solution but as a starting point.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| MPEG-1 Audio Layer I (ISO/IEC 11172-3) | 1993 | Original Layer I — 32 subband QMF, 384-sample frame |
| PASC (Digital Compact Cassette) | 1992 | Pre-standardization variant of Layer I locked at 384 kbps; later integrated into ISO/IEC 11172-3 |
| MPEG-2 Audio Layer I (ISO/IEC 13818-3) | 1998 | Extended sampling rates (16/22.05/24 kHz), multichannel support |

### 1.3 Current Adoption
- **Primary use cases today:** Legacy; essentially no active use. Historical use in VHS Hi-Fi, DCC (Digital Compact Cassette), early MPEG-1 audio tracks
- **Platforms with native support:** All MPEG audio decoders support Layer I (decoders are required to decode all three layers)
- **Major services using this format:** None active; DVD-Video uses MP2, not MP1; DCC discontinued 1996
- **Hardware support:** Any MPEG audio decoder chip handles Layer I, but no dedicated hardware plays MP1 files today
- **Status:** Deprecated / legacy — replaced by Layer II (MP2) for broadcast and Layer III (MP3) for general use

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Perceptual audio coder (PAC) / subband coder
- **Core algorithm:** QMF (Quadrature Mirror Filter) polyphase filter bank (identical to Layer II) + simplified psychoacoustic bit allocation
- **Loss mechanism:** Psychoacoustic masking via subband quantization — less sophisticated than Layer II's per-granule allocation; Layer I allocates bits per 12-sample block
- **Frame-based vs sample-based:** Frame-based (fixed-size frames of 384 samples)
- **Fixed vs variable frame size:** Fixed frame size — 384 PCM samples per frame. Layer I's frame is 1/3 the size of Layer II's frame.

### 2.2 High-Level Encoding Flow (Block Diagram)
```
Input PCM Samples (384 per channel)
      │
      ▼
[Pre-processing: PCM input, split into 32 subbands by 512-tap QMF polyphase filter bank]
      │
      ▼
[Frame formation: 12 subband samples per subband (one block per subband)]
      │
      ▼
[Psychoacoustic Model: FFT analysis → masking threshold per subband (simpler than Layer II)]
      │
      ▼
[Bit Allocation: Per-subband allocation for the 12-sample block using MNR optimization]
      │
      ▼
[Scalefactor extraction: Block companding — single scalefactor per subband per block]
      │
      ▼
[Quantization: Uniform mid-tread quantizer per subband (2–15 levels = 1 to 32767 codewords)]
      │
      ▼
[Bitstream Packing: Header + CRC + Bit Allocation (4 bits/subband) + Scalefactor (6 bits/subband) + Samples]
      │
      ▼
Output Encoded MP1 Bitstream
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | 256 samples / ~5.7 ms | Lower delay than Layer II due to shorter frame |
| Frame size | 384 samples | Per channel — 1/3 of Layer II's 1152 |
| Max channels | 2 (stereo) | + multichannel extension in MPEG-2 |
| Max bit depth | 16-bit input | Internal precision varies by encoder |
| Max sample rate | 48 kHz (MPEG-1) | |
| Bitrate range | 32–448 kbps | Highest of all three MPEG audio layers |
| Complexity | O(1) per sample | Lowest complexity of the three layers |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
MP1 files are bare audio streams — same sync word structure as all MPEG audio layers. Files start with the 12-bit sync word `0xFFF`:

```
Offset  Bits    Hex Value        Meaning
------  -----   ---------------  -------------------
0x0000  12      0xFFF            MPEG audio frame sync (all 1s)
0x000C  1       ID=1              MPEG-1
0x000D  2       Layer=01          Layer I (01=Layer I, 10=Layer II, 11=Layer III)
...
```

### 3.2 File-Level Header Layout
Raw MP1 streams have no file-level header. When embedded in RIFF/WAV (rare):

```
Offset   Size    Field Name              Type        Valid Range   Description
-------  ------  ----------------------  ----------  -----------   ---------------------------
0x0000   4B      RIFF Magic             char[4]     "RIFF"       File container magic
0x0004   4B      File Size              uint32 LE   —            File size minus 8 bytes
0x0008   4B      Format Magic          char[4]     "WAVE"       RIFF wave chunk
0x000C   4B      fmt  Magic            char[4]     "fmt "       Format chunk identifier
0x0010   4B      fmt  Chunk Size       uint32 LE   16           Chunk size
0x0014   2B      Audio Format          uint16 LE   0x0060       MP1 = 0x0060 (WAVE_FORMAT_MPEG)
0x0016   2B      Channels              uint16 LE   1–2          1=mono, 2=stereo
0x0018   4B      Sample Rate           uint32 LE   32000,44100,48000  Samples per second
0x001C   4B      Byte Rate             uint32 LE   dependent    Bytes per second (=bitrate/8)
0x0020   2B      Block Align           uint16 LE   dependent    Frame size in bytes
0x0022   2B      Extra Params Size     uint16 LE   22           Size of extra MP1 params
0x0024   2B      Head Bitrate Index    uint16 LE   0x0001–0x000F  See bitrate index table
0x0026   1B      Mode                  uint8       0–3          0=stereo,1=jstereo,2=dual,3=mono
...      ...     ...                   ...         ...           Additional WAV fields
```

### 3.3 Frame / Block Header Layout
Layer I frame header is 32 bits (4 bytes), identical in structure to Layer II except for layer bits:
```
Bit Offset   Bit Width   Field Name              Type      Description
----------   ---------   --------------------    ------    ---------------------------
0            12          Sync Word               uint      Must be 0xFFF (all 1s)
12           1           ID                      uint      1=MPEG-1, 0=MPEG-2
13           2           Layer                   uint      01=Layer I (critical difference)
15           1           Protection              uint      1=no CRC, 0=CRC present
16           4           Bitrate Index           uint      See bitrate table below
20           2           Sample Rate Index       uint      00=44.1,01=48,10=32,11=reserved
22           1           Padding                 bool      1=padding slot present
23           1           Private                 bool      Ancillary data
24           2           Mode                    uint      00=stereo,01=joint stereo,10=dual,11=mono
26           2           Mode Extension          uint      Joint stereo band grouping
28           1           Copyright               bool      1=copyrighted
29           1           Original/Copy           bool      1=original media
30           2           Emphasis                uint      De-emphasis
```

#### Bitrate Index Table
| Index (hex) | Bitrate (kbps) | Notes |
|-------------|---------------|-------|
| 0x0 | Free bitrate | Variable bitrate |
| 0x1 | 32 | Lowest supported bitrate |
| 0x2 | 64 | |
| 0x3 | 96 | |
| 0x4 | 128 | |
| 0x5 | 160 | |
| 0x6 | 192 | |
| 0x7 | 224 | |
| 0x8 | 256 | |
| 0x9 | 288 | [NEEDS VERIFICATION] |
| 0xA | 320 | |
| 0xB | 352 | [NEEDS VERIFICATION] |
| 0xC | 384 | DCC/PASC fixed rate |
| 0xD | 416 | [NEEDS VERIFICATION] |
| 0xE | 448 | Highest Layer I bitrate |
| 0xF | Bad/reserved | Must not be used |

#### Sample Rate Index Table (identical to Layer II)
| Index | MPEG-1 Sample Rate | MPEG-2 Sample Rate |
|-------|-------------------|-------------------|
| 00 | 44,100 Hz | 22,050 Hz |
| 01 | 48,000 Hz | 24,000 Hz |
| 10 | 32,000 Hz | 16,000 Hz |
| 11 | Reserved | Reserved |

### 3.4 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Unsigned integer | No | Not defined in spec |
| 16-bit | Signed integer | Yes | Standard input/output |
| 20-bit | Signed integer | Partial | Internal precision |
| 24-bit | Signed integer | No | Not supported |
| 32-bit | Signed integer | No | Not supported |
| 32-bit | IEEE float | No | Not supported |
| 64-bit | IEEE float | No | Not supported |

#### Supported Sample Rates
| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 8000 | Telephone | No | Not defined |
| 11025 | — | No | Not defined |
| 16000 | Wideband | No (MPEG-2 only) | |
| 22050 | — | Yes (MPEG-2 only) | |
| 32000 | Broadcast | Yes | MPEG-2 |
| 44100 | CD audio | Yes | Primary rate |
| 48000 | Professional | Yes | Primary rate for broadcast |
| 88200 | 2× CD | No | Not defined |
| 96000 | High-res | No | Not defined |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Pre-Processing Stage
- **DC offset removal:** Not explicitly defined — encoder may optionally apply
- **Pre-emphasis filter:** Not applied by codec; emphasis flag in header signals de-emphasis to decoder
- **Windowing function:** QMF polyphase filter bank uses raised-cosine window (512-tap filter with 64 polyphase branches)
- **Level normalization:** Block companding via scalefactors — 12-sample block per subband normalized by a single 6-bit scalefactor
- **Stereo decorrelation pre-step:** For joint stereo: L = (L+R)/√2, R = (L−R)/√2 computed before quantization

### 4.2 Analysis / Transform Stage

#### Transform Type: QMF Polyphase Filter Bank (32 bands)
```
Parameters:
  Number of subbands:  32 (uniformly distributed in frequency)
  Filter length:       512 taps (MPEG-1) per channel
  Downsample factor:   32× (1 sample per subband per input sample)
  Frame input:         384 PCM samples → 12 subband samples per subband
  Block grouping:      Single block of 12 samples per subband (no granule groups)
  Overlap:            Critically sampled (no overlap — perfect reconstruction)
  Window function:    512-tap polyphase filter with cosine-modulated prototype
```

**Frame structure comparison — Layer I vs Layer II:**
```
Layer I:  384 samples/frame → 12 samples/subband → 1 scalefactor per subband
Layer II: 1152 samples/frame → 36 samples/subband → 3 granules × 12 samples
          → up to 3 scalefactors per subband (if SCFSI indicates reuse)
```

### 4.3 Psychoacoustic Model (Lossy Only)

#### Model Version
- **Model:** MPEG Psychoacoustic Model 1 (ISO 11172-3 Annex D) — simplified version
- **Analysis window:** 512-point FFT (simpler than Layer II's 1024-point)
- **Resolution:** Per-frame analysis — one masking threshold computed per 384-sample frame

#### Masking Thresholds
```
Absolute Threshold of Hearing (ATH):
  ATH(f) = 3.64·(f/1000)^(-0.8) - 6.5·exp(-0.6·(f/1000 - 3.3)^2) + 10^(-3)·(f/1000)^4 dB

Simultaneous Masking:
  Masking slope (upward):   ~5 dB/Bark
  Masking slope (downward): ~15 dB/Bark
  Masking resolution:       Per critical band (32 subbands → ~32 Bark)

Temporal Masking:
  Pre-masking:  ~20 ms (poorly modeled)
  Post-masking: ~100 ms
```

#### Bit Allocation Algorithm
```
Layer I bit allocation uses 4 bits per subband (32 × 4 = 128 bits in header):

1. Compute FFT of input block (512 samples)
2. Compute masking threshold T[k] for each critical band
3. Compute SMR for each subband: SMR[i] = Signal_Level[i] - Threshold[i]
4. Initialize all bit allocations to 0
5. Iterative MNR optimization:
   - MNR[i] = SNR_quantizer[i] - SMR[i]
   - Find subband with minimum MNR (most audible distortion)
   - Increment bit allocation for that subband by 1
   - Update SNR_quantizer[i] from allocation table
   - Recompute MNR[i]
   - Repeat until all bits exhausted

Allocation table (Layer I — 4-bit index → quantization):
  Index 0: 0 bits (no quantization — subband not transmitted)
  Index 1: 2 levels (1 bit per sample)
  Index 2: 3 levels (not used — reserved)
  Index 3: 5 levels (2 bits per sample)
  ...
  Index 15: 32768 levels (15 bits per sample)
```

### 4.4 Quantization
- **Type:** Uniform mid-tread quantizer with block companding
- **Step sizes:** Determined by bit allocation index — 2^1 to 2^15 levels (2 to 32768 levels)
- **Block companding:** 12 samples per subband normalized by a single 6-bit scalefactor index (0–63 dB reduction range)
- **Dequantization:** `sample = (codeword + 0.5) × step_size × scalefactor_multiplier`
- **Noise shaping:** Basic SQNR optimization; no advanced noise shaping or prediction

### 4.5 Stereo Encoding Modes
| Mode | Description | Condition for Selection | Bitrate Range |
|------|-------------|------------------------|---------------|
| Dual Mono | L and R encoded independently | Explicitly requested | any |
| Stereo | L/R encoded independently, shared frame | default | any |
| Joint Stereo / MS | M=(L+R)/2, S=(L−R)/2 encoded separately | When S energy < M energy | any |
| Intensity Stereo | Share high-freq envelope | High compression | < 96 kbps |

### 4.6 Entropy / Lossless Coding Stage
```
Method: None — raw quantized binary samples

Layer I does NOT use Huffman coding or any entropy coding.
All audio data is transmitted as raw binary codes.

Bit allocation data: 4 bits per subband (32 × 4 = 128 bits for stereo)
Scalefactor data: 6 bits per scalefactor per subband
Audio sample data: Raw binary codes, N bits per sample based on allocation
```

### 4.7 Encoder Settings / Quality Modes
| Quality Setting | Bitrate Range | Intended Use Case | Transparent? |
|-----------------|---------------|-------------------|--------------|
| Lowest | 32–64 kbps | Low-bitrate speech | No |
| Low | 96–128 kbps | General lossy | No |
| Medium | 160–192 kbps | Near-CD quality | Marginal |
| High | 224–320 kbps | High-quality | Mostly (most content) |
| Highest | 384–448 kbps | Broadcast / archival | Yes (most content) |

**Quality comparison:** MP1 requires significantly higher bitrate than MP2 or MP3 for equivalent quality due to its coarser bit allocation granularity and lack of granule subdivision. MP1 at 320 kbps ≈ MP2 at 192 kbps ≈ MP3 at 128 kbps.

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. Scan bitstream for sync word: 12 consecutive 1-bits (0xFFF)
2. Validate candidate frame:
   a. Verify ID bit (MPEG-1 or MPEG-2)
   b. Verify Layer bits = 01 (Layer I — critical)
   c. Check bitrate index ≠ 0xF (reserved)
   d. Check sample rate index ≠ 0x3 (reserved)
   e. Compute frame_size = (12 × bitrate) / sample_rate [+ padding]
   f. Verify next frame has valid sync at computed offset
3. If validation fails: advance 1 bit and retry
4. Check CRC if protection bit = 0
5. Max failed sync attempts before error: 4096 bytes
```

**Frame size formula for Layer I:**
```
frame_size_bytes = (12 × bitrate) / sample_rate + padding_bit

Examples:
  128 kbps @ 44.1 kHz: (12 × 128000) / 44100 ≈ 417 bytes
  192 kbps @ 48 kHz:   (12 × 192000) / 48000   = 480 bytes
  384 kbps @ 44.1 kHz: (12 × 384000) / 44100 ≈ 1044 bytes (PASC DCC)
```

#### Seeking
- **CBR seeking:** `byte_offset = frame_number × frame_size`
- **VBR seeking:** No seek table — scan required for random access
- **Seek precision:** Frame-accurate
- **In containers:** WAV `idx1` chunk or MPEG PS timestamp-based

### 5.2 Core Decode Pipeline
```
1. Read frame header (32 bits = 4 bytes)
   ├── Verify sync word = 0xFFF
   ├── Verify Layer bits = 01 (Layer I)
   ├── Parse bitrate index → actual bitrate
   ├── Parse sample rate index → sample rate
   ├── Check padding flag
   └── Compute frame_size = (12 × bitrate) / sample_rate [+ padding]

2. Read CRC (16 bits) if protection = 0

3. Read bit allocation (32 × 4 bits = 128 bits for stereo)
   └── Each 4-bit value indexes the quantization table

4. Read scalefactors (32 × 6 bits = 192 bits for stereo)
   └── One 6-bit scalefactor per subband per channel

5. Read quantized subband samples (12 samples per subband per channel)
   └── Raw binary codes, N bits per sample based on allocation

6. Inverse quantization per sample
   └── value = (codeword + 0.5) × step_size × scalefactor_multiplier

7. Apply scalefactor decompanding (multiply by scalefactor value)

8. Stereo processing (if joint stereo):
   ├── MS: L = (M + S) / √2 ; R = (M - S) / √2
   └── Intensity: reconstruct from shared envelope + angle

9. Inverse QMF polyphase synthesis filter bank
   └── 32 subband samples × 12 time slots → 384 output samples per channel

10. Output to PCM buffer (16-bit signed integer)
```

### 5.3 Error Concealment
- **Corrupt frame detection:** CRC-16 check (16 bits per frame, mandatory for most decoders), sync word validation
- **Concealment method:** Muting (replace frame with silence), or last-frame repeat
- **Maximum consecutive errors before silence:** Implementation-defined; typically 1–3 frames before muting

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** Bare bitstream (`.mp1`) — no file-level header
- **Overhead:** ~0% (no container overhead)
- **Seeking in native container:** Limited — frame index required
- **Multiple streams in native container:** No

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| MP4/M4A | No | — | — | MP4 uses AAC |
| Matroska/MKA | Yes | Yes | Full | Rare — Layer I rarely used |
| OGG | No | — | — | |
| WAV | Yes | Yes | Limited | Via WAVE_FORMAT_MPEG (0x0060) |
| AIFF | No | — | — | |
| MPEG PS (.mpg) | Yes | Limited | None | Native |
| MPEG TS (.ts) | Yes | Yes | Via PES | Rare for Layer I |
| VHS Hi-Fi | Yes | No | None | Analog FM multiplexing |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** ID3v1 (at end) or ID3v2.3/ID3v2.4 (at beginning) for `.mp1` files
- **Tag block location:** Beginning (ID3v2) or end (ID3v1) of file
- **Tag block identifier:** `49 44 33` (ID3) at offset 0 for ID3v2; `54 41 47` (TAG) at EOF for ID3v1

### 7.2 Standard Tag Fields — Complete Reference
| Tag Field | Internal Key (this format) | Max Length | Character Encoding | Multi-value | Notes |
|-----------|---------------------------|------------|-------------------|-------------|-------|
| Title | TIT2 | 30 chars / unlimited | ISO-8859-1 / UTF-16 | No | |
| Artist | TPE1 | 30 chars / unlimited | | No | |
| Album | TALB | 30 chars / unlimited | | No | |
| Album Artist | TPE2 | 30 chars / unlimited | | No | |
| Composer | TCOM | 30 chars / unlimited | | Yes | |
| Genre | TCON | 30 chars / unlimited | | No | ID3v1 genre numbers |
| Year / Date | TYER / TDRC | 4 chars / unlimited | | No | |
| Track Number | TRCK | 30 chars / unlimited | | No | Format: "N" or "N/Total" |
| Disc Number | TPOS | 30 chars / unlimited | | No | |
| Comment | COMM | 30 chars / unlimited | | Yes | |
| Lyrics | USLT | unlimited | | No | |
| BPM | TBPM | 30 chars | | No | |
| Compilation | TCMP | 1 char | | No | |
| Copyright | TCOP | 30 chars / unlimited | | No | |
| Publisher/Label | TPUB | 30 chars / unlimited | | No | |
| Encoder | TENC | 30 chars / unlimited | | No | |
| Encoder Settings | TSSE | 30 chars / unlimited | | No | |
| ISRC | TSRC | 12 bytes | ASCII | No | |
| Cover Art (front) | APIC | up to several MB | Binary (JPEG/PNG) | No | ID3v2 only |
| Arbitrary/Custom | TXXX | unlimited | UTF-8 | Yes | |

### 7.3 Cover Art Storage
```
Cover art storage format in MP1 (via ID3v2):
  Container type:  APIC frame in ID3v2 tag
  Image formats:   JPEG (recommended), PNG
  Picture types:   APIC type 3 = front cover (primary)
  
  Binary layout (same as MP2):
    [0x00-0x03]  Frame ID: "APIC" (4 bytes)
    [0x04-0x07]  Frame size (syncsafe integer v2.4, big-endian v2.3)
    [0x08-0x09]  Frame flags (2 bytes)
    [0x0A]       Text encoding (1 byte)
    [0x0B-...]   MIME type (null-terminated string)
    [...]         Picture type (1 byte)
    [...]         Description (null-terminated string)
    [...]         Binary image data
```

### 7.4 Metadata Compatibility Matrix
| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| ID3v1 | ✓ | ✓ | ✗ | Lowest |
| ID3v2.3 | ✓ | ✓ | ✓ (if same encoding) | Medium |
| ID3v2.4 | ✓ | ✓ | ✓ (if same encoding) | High |
| APEv2 | ✗ | ✗ | ✗ | N/A |
| Vorbis Comments | ✗ | ✗ | ✗ | N/A |
| MP4 Atoms | ✗ | ✗ | ✗ | N/A |

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   mp1                        # used with -c:a
AV_CODEC_ID:        AV_CODEC_ID_MP1            # C constant in libavcodec/codec_id.h
Format Name (CLI):  mp1, mpeg audio             # raw bitstream
Encoder(s):         mp1 (native, libavcodec)      # Decode-only in practice — no production encoder
Decoder(s):         mp1, mp2, mp3 (auto-detect)  # All MPEG audio layers decoded by same decoder
Muxer(s):           mp2, mpeg, mpegts, mpegvideo
Demuxer(s):         mp1, mp2, mp3, mpeg, mpegts
```

**Important:** FFmpeg does not have a production-quality MP1 encoder. The mp1 encoder exists but is rarely used. MP1 decoding is supported via the same `mp2` decoder.

### 8.2 FFmpeg Encoding — Full CLI Reference

```bash
# FFmpeg has an MP1 encoder but it is rarely used in practice
# MP2 encoder is preferred for all production use cases
ffmpeg -i input.wav \
  -c:a mp1 \
  -b:a 192k \           # CBR bitrate — range: 32k–448k, default: 128k
  -ar 44100 \           # Output sample rate (Hz)
  -ac 2 \               # Output channel count
  -sample_fmt s16 \     # Sample format: s16
  output.mp1
```

#### Complete FFmpeg Option Table
| Option | Type | Default | Range / Valid Values | Effect on Output |
|--------|------|---------|---------------------|-----------------|
| `-b:a` | int | 128k | 32k–448k | Target bitrate for CBR |
| `-ar` | int | input rate | 32000, 44100, 48000 | Output sample rate |
| `-ac` | int | input channels | 1–2 | Output channel count |
| `-joint_stereo` | bool | 1 | 0=force L/R, 1=MS | Enable joint stereo |

**Note:** MP1 encoding in FFmpeg is considered experimental and rarely used. For production audio encoding, use MP2 (`-c:a mp2`) or MP3 (`-c:a libmp3lame`) instead.

### 8.3 FFmpeg Encoding — libavcodec C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/opt.h>
#include <libswresample/swresample.h>

// ─── 1. Find encoder ─────────────────────────────────────────────────────────
const AVCodec *codec = avcodec_find_encoder(AV_CODEC_ID_MP1);
// Alternative by name: avcodec_find_encoder_by_name("mp1");
if (!codec) { fprintf(stderr, "MP1 encoder not found\n"); exit(1); }

// ─── 2. Allocate codec context ───────────────────────────────────────────────
AVCodecContext *ctx = avcodec_alloc_context3(codec);
if (!ctx) { fprintf(stderr, "OOM\n"); exit(1); }

// ─── 3. Set required parameters ──────────────────────────────────────────────
ctx->bit_rate    = 192000;              // bits/sec for CBR
ctx->sample_fmt  = AV_SAMPLE_FMT_S16;  // MP1 encoder accepts S16
ctx->sample_rate = 44100;              // Hz — 32000, 44100, or 48000
ctx->channels    = 2;                  // 1=mono, 2=stereo
ctx->channel_layout = AV_CH_LAYOUT_STEREO;

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
frame->nb_samples  = ctx->frame_size;  // MP1: ctx->frame_size = 384
av_channel_layout_copy(&frame->ch_layout, &ctx->ch_layout);
av_frame_get_buffer(frame, 0);

AVPacket *pkt = av_packet_alloc();

// ─── 6. Encode loop ─────────────────────────────────────────────────────────
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

// ─── 7. Flush encoder ────────────────────────────────────────────────────────
encode_frame(ctx, NULL, pkt, outfile); // NULL frame triggers flush/drain

// ─── 8. Cleanup ──────────────────────────────────────────────────────────────
av_frame_free(&frame);
av_packet_free(&pkt);
avcodec_free_context(&ctx);
```

**Critical Notes:**
- `ctx->frame_size` for MP1 is 384 samples — use this for `nb_samples`
- MP1 encoder accepts `AV_SAMPLE_FMT_S16` (signed 16-bit)
- For float input, use `libswresample` to convert to S16 first
- MP1 encoding is rarely used — prefer MP2 for production work

### 8.4 FFmpeg Decoding — Full CLI Reference

```bash
# Decode to raw PCM WAV
ffmpeg -i input.mp1 \
  -c:a pcm_s16le \
  -ar 44100 \          # Resample if needed
  -ac 2 \              # Downmix if needed
  output.wav

# Extract specific stream
ffmpeg -i input.mpg -map 0:a:0 -c:a copy output.mp1

# Probe format without decoding
ffprobe -v quiet -print_format json -show_streams -show_format input.mp1
```

### 8.5 FFmpeg Decoding — libavcodec C API

```c
// ─── Complete decode program skeleton ────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.mp1", NULL, NULL);
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

### 8.6 FFmpeg Metadata Handling

```bash
# Read all metadata as JSON
ffprobe -v quiet -print_format json -show_format input.mp1 | jq .format.tags

# Write metadata (copy audio, update tags)
ffmpeg -i input.mp1 \
  -c copy \
  -metadata title="Song Title" \
  -metadata artist="Artist Name" \
  -metadata album="Album Name" \
  -metadata track="5/12" \
  -metadata date="2024" \
  -metadata genre="Electronic" \
  output.mp1

# Strip all metadata
ffmpeg -i input.mp1 -c copy -map_metadata -1 output.mp1

# Embed cover art
ffmpeg -i input.mp1 -i cover.jpg \
  -c copy \
  -map 0:a -map 1:v \
  -metadata:s:v title="Album cover" \
  -disposition:v attached_pic \
  output.mp1
```

#### FFmpeg Internal Metadata Key Mapping
| Standard Field | FFmpeg Key | MP1 Native Key | Notes |
|----------------|------------|--------------------------|-------|
| Title | title | TIT2 | |
| Artist | artist | TPE1 | |
| Album | album | TALB | |
| Album Artist | album_artist | TPE2 | |
| Track Number | track | TRCK | |
| Disc Number | disc | TPOS | |
| Genre | genre | TCON | |
| Date/Year | date | TYER / TDRC | |
| Comment | comment | COMM | |
| Encoder | encoder | TENC | Auto-set by FFmpeg |

### 8.7 Quality / Fidelity Decision Table
| Use Case | Settings | Expected Size (min) | Notes |
|----------|----------|---------------------|-------|
| Archival / professional | `-c:a mp1 -b:a 384k` | ~26 MB/min | Broadcast master |
| Audiophile lossy | `-c:a mp1 -b:a 256k` | ~17 MB/min | Near-transparent (theoretical) |
| Streaming (high) | `-c:a mp1 -b:a 192k` | ~13 MB/min | MP2 is better at same bitrate |
| Podcast / voice | `-c:a mp1 -b:a 64k -ac 1` | ~4 MB/min | Mono |
| Mobile / low bandwidth | `-c:a mp1 -b:a 32k -ac 1` | ~2 MB/min | Not recommended — use MP2 or MP3 |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
MP1 raw streams have no native seek table. Seeking relies on frame-based calculation for CBR or full scan for VBR.

### 9.2 Gapless Playback Data
```
Encoder delay:   0 samples (no systematic delay beyond filter bank)
Padding:         0 samples (or 1 padding slot at 44.1 kHz when padding=1)
Storage location: N/A — no gapless metadata in MP1
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | Frame-based; 384 samples/frame |
| Algorithmic encoder delay | 256 samples / ~5.7 ms | Lower than Layer II |
| Live encoding feasible | Yes | Low delay, fixed frame size |
| HTTP progressive download | Yes | Frame-based |
| HTTP Live Streaming (HLS) | No | Not used for HLS |
| DASH streaming | No | Not used for DASH |
| WebRTC / RTP transport | Yes | Via RTP payload type for MPEG audio |
| Minimum decode buffer | 1 frame / ~8.7 ms | |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | Layout Name | Channel Order | FFmpeg Layout Constant |
|----------|-------------|---------------|----------------------|
| 1 | Mono | C | AV_CHANNEL_LAYOUT_MONO |
| 2 | Stereo | L, R | AV_CHANNEL_LAYOUT_STEREO |
| 3 | — | L, R, C | AV_CHANNEL_LAYOUT_2_1 |
| 4 | — | L, R, SL, SR | AV_CHANNEL_LAYOUT_2_2 |
| 5 | — | L, R, C, BL, BR | AV_CHANNEL_LAYOUT_5_0 (MPEG-2 only) |
| 6 | 5.1 | FL, FR, C, LFE, SL, SR | AV_CHANNEL_LAYOUT_5_1 (MPEG-2 only) |

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 16-bit | Fixed |
| Max sample rate | 48,000 Hz | MPEG-1 |
| Float support | None | Integer PCM only |
| DSD support | No | |
| 24-bit support | No | Not defined |

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

| Platform / API | Encode | Decode | FFmpeg Flag | Notes |
|----------------|--------|--------|-------------|-------|
| NVIDIA NVENC | No | — | — | No hardware MP1 encoder |
| NVIDIA NVDEC | — | No | — | No hardware MP1 decoder |
| Intel QSV | No | No | — | Not supported |
| Apple VideoToolbox | No | No | — | No hardware MP1 support |
| Android MediaCodec | No | Partial | — | System decoders may support |

**Note:** MP1 is universally decoded in software. Any MPEG audio hardware decoder handles MP1 but modern implementations favor MP3 and AAC.

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Encoder-Specific Issues
| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| MP1 encoder is experimental/rarely maintained | All | Use MP2 encoder instead |
| Default bitrate (128k) too low | All | Always specify `-b:a` explicitly |
| Limited encoder quality vs reference | All | Use reference encoder if available |

### 14.2 Interoperability Issues
- **Files with ID3v1 only:** Most MP1-capable decoders may ignore ID3v1; use ID3v2
- **DCC PASC files:** Non-standard padding handling at 44.1 kHz — set padding flag appropriately

### 14.3 Edge Cases to Handle in Converter
- **Empty file / 0-sample stream:** Return empty output
- **File < 1 frame:** Encode partial frame or pad with silence
- **All-silence audio:** Encodes efficiently
- **DC offset:** Encoder handles; DC component concentrated in subband 0
- **Full-scale sine:** May cause clipping in quantized subbands
- **File with corrupt header:** Skip to next sync word
- **Truncated file:** Decode up to last complete frame
- **Sample rate not supported:** Error out or resample

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM MP1

| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|--------------|
| → FLAC | `ffmpeg -i in.mp1 -c:a flac -compression_level 8 out.flac` | All tags (via ID3→Vorbis rewrite) | Lossless decode |
| → ALAC | `ffmpeg -i in.mp1 -c:a alac out.m4a` | Partial | Lossless decode |
| → MP3 | `ffmpeg -i in.mp1 -c:a libmp3lame -q:a 0 out.mp3` | Tags via ID3v2.3 | Generation loss |
| → AAC | `ffmpeg -i in.mp1 -c:a aac -b:a 256k out.m4a` | Partial | Generation loss |
| → Opus | `ffmpeg -i in.mp1 -c:a libopus -b:a 128k out.opus` | Vorbis Comments | Generation loss |
| → WAV | `ffmpeg -i in.mp1 -c:a pcm_s16le out.wav` | RIFF INFO | Lossless decode |
| → MP2 | `ffmpeg -i in.mp1 -c:a mp2 -b:a 192k out.mp2` | ID3v2 | Generation loss (re-encode) |

### 15.2 Converting TO MP1

| Source | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|--------------|
| FLAC → | `ffmpeg -i in.flac -c:a mp1 -b:a 256k out.mp1` | Vorbis → ID3v2 | Not recommended — use MP2 instead |
| WAV → | `ffmpeg -i in.wav -c:a mp1 -b:a 256k out.mp1` | Limited | Not recommended — use MP2 instead |
| MP3 → | `ffmpeg -i in.mp3 -c:a mp1 -b:a 256k out.mp1` | ID3v2 | Not recommended — use MP2 instead |

### 15.3 Lossless Round-Trip Verification
```bash
# Decode only (MP1 is lossy — no lossless round-trip possible)
ffmpeg -i input.mp1 -c:a pcm_s16le decoded.wav

# Verify decoder output matches reference
ffmpeg -i input.mp1 -map 0:a -f framemd5 input.md5
ffmpeg -reference.wav -map 0:a -f framemd5 reference.md5
diff input.md5 reference.md5   # Will differ due to lossy encoding
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| FFmpeg native (libavcodec) | C | LGPL 2.1+ | 6/10 | 10/10 | https://ffmpeg.org |
| ISO reference | C | Royalty-bearing | Reference | Reference | Via ISO/IEC |
| Rockbox | C | Various | — | 10/10 | https://www.rockbox.org |
| libmad | C | GPL | 7/10 | 10/10 | https://www.underbit.com/products/mad/ |

### Build Instructions
```bash
# FFmpeg includes MP1 encoder and decoder by default
# No additional configure flags needed
./configure --enable-gpl --enable-nonfree  # Optional
make -j$(nproc)
```

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **ISO/IEC 11172-3:1993:** Coding of moving pictures and associated audio for digital storage media at up to about 1.5 Mbit/s — Part 3: Audio (MPEG-1 Audio, all layers)
- **ISO/IEC 13818-3:1998:** Generic coding of moving pictures and associated audio information — Part 3: Audio (MPEG-2 Audio extension)

### Technical Resources
- FFmpeg codec documentation: `ffmpeg -h encoder=mp1` or https://ffmpeg.org/ffmpeg-codecs.html
- Multimedia Wiki MPEG Audio: https://wiki.multimedia.cx/index.php/MPEG-1_Audio
- Hydrogenaudio knowledge base: https://hydrogenaud.io/
- Digital Compact Cassette (DCC/PASC): https://en.wikipedia.org/wiki/Digital_Compact_Cassette

### Academic Papers
- *Coding of Moving Pictures and Associated Audio for Digital Storage Media at up to 1.5 Mbit/s*, ISO/IEC 11172-3, 1993
- Brandenburg, K. et al., *ASPEC and MRS — The ISO/MPEG Audio Coding Proposals*, AES 91st Convention, 1991
- Stoll, G., *MPEG/Audio — A standardized scheme for high quality audio coding at 128 kbit/s per channel*, AES 10th International Conference, 1991

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] Identify FFmpeg `./configure` flags (enabled by default)
- [ ] Verify `ffmpeg -encoders` confirms mp1 encoder is available (experimental)
- [ ] Verify `ffmpeg -decoders` confirms mp1 decoder is available
- [ ] Note: MP1 encoder quality is not production-grade — recommend MP2 for all production work
- [ ] Handle platform compatibility (encoding works on all FFmpeg builds)

### Encoding Pipeline
- [ ] Convert input sample format to `AV_SAMPLE_FMT_S16` using libswresample
- [ ] Handle fixed-frame-size (`ctx->frame_size` = 384)
- [ ] Implement proper flush/drain at end of stream
- [ ] Validate sample rate in {32000, 44100, 48000}
- [ ] Validate channel layout (mono/stereo only for MPEG-1)
- [ ] Warn user if MP1 is selected — recommend MP2 instead

### Decoding Pipeline
- [ ] Implement sync word search (0xFFF pattern) for stream recovery
- [ ] Handle AVERROR(EAGAIN) in receive_packet loop correctly
- [ ] Flush decoder at EOF (send NULL packet)
- [ ] Handle mono/stereo output layout correctly

### Metadata
- [ ] Read all standard ID3v1 and ID3v2 tag fields
- [ ] Map ID3v2 tag fields through standard key mapping
- [ ] Read cover art (APIC frame) and preserve as JPEG/PNG binary
- [ ] Write all standard tag fields as ID3v2.3
- [ ] Embed cover art as APIC frame
- [ ] Handle tag encoding (ISO-8859-1 / UTF-16 / UTF-8)

### Quality & Verification
- [ ] Note: Lossless verification not applicable (MP1 is lossy)
- [ ] Implement progress reporting from FFmpeg stats output
- [ ] Implement error detection and partial-file recovery
- [ ] Test with: silence, full-scale, mono, stereo files

### Edge Cases
- [ ] Handle files with corrupt or missing headers
- [ ] Handle files with 0 samples
- [ ] Handle sample rate mismatch (resample)
- [ ] Handle channel count mismatch (auto-downmix)
- [ ] Handle very short files (< 1 frame)

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
