# MPEG-1 Audio Layer II — Deep Technical Reference
> **Category:** Lossy
> **File Extensions:** `.mp2`, `.mpa`, `.mpg`, `.mpeg`
> **MIME Types:** `audio/mpeg`, `audio/x-mpeg`, `audio/mp2`
> **Standardization Body:** ISO/IEC (MPEG)
> **Primary Specification:** ISO/IEC 11172-3:1993 (MPEG-1 Part 3), ISO/IEC 13818-3:1998 (MPEG-2 Part 3)
> **Patent Status:** Patented — expired (most patents expired 2017–2020)
> **License:** Open / Royalty-free for most uses (patents expired)
> **Current Version:** ISO/IEC 11172-3:1993 (unchanged since 1993)
> **Active Development:** No — deprecated in favor of MP3 and AAC, but widely deployed in legacy infrastructure

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** MPEG (Moving Picture Experts Group), specifically ISO/IEC JTC1/SC29/WG11
- **Year Created:** 1993 (standard published as ISO/IEC 11172-3)
- **Original Purpose:** Efficient storage and transmission of high-quality digital audio at bitrates compatible with CD-ROM data rates (~1.5 Mbit/s total for audio+video)
- **Problem with Predecessors:** Prior solutions (analog录制, PCM) either wasted storage or lacked perceptual coding efficiency. The goal was to achieve CD-quality audio at 192–256 kbps (vs. 1411 kbps for 16-bit/44.1 kHz stereo PCM), enabling real-time audio encoding and decoding at manageable complexity.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| MPEG-1 Audio Layer II (ISO/IEC 11172-3) | 1993 | Original Layer II, 32/44.1/48 kHz, up to 2 channels |
| MPEG-2 Audio Layer II (ISO/IEC 13818-3) | 1998 | Extended bitrates, multichannel (up to 5.1), 16/22.05/24 kHz sample rates |

### 1.3 Current Adoption
- **Primary use cases today:** DAB/DAB+ digital radio, DVB broadcast, DVD-Video PAL audio tracks, Video CD (VCD), SVCD, MPEG-1/MPEG-2 Program Streams, satellite radio, professional broadcast infrastructure
- **Platforms with native support:** All major operating systems (native decoding via system codecs), hardware DAB receivers, car infotainment systems, set-top boxes
- **Major services using this format:** BBC DAB radio, European DAB+ (backwards-compatible with MP2), German DAB, satellite TV audio (DVB-S), DVD-Video PAL region audio
- **Hardware support:** Extremely broad legacy hardware decode support (billions of devices from 1993–present)
- **Status:** Legacy — still deployed in broadcast but largely replaced by AAC/AAC+ in new services

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Perceptual audio coder (PAC) / subband transform coder
- **Core algorithm:** QMF (Quadrature Mirror Filter) polyphase filter bank + psychoacoustic bit allocation
- **Loss mechanism:** Psychoacoustic masking via subband quantization — bits are allocated where noise is most perceptible relative to the masking threshold
- **Frame-based vs sample-based:** Frame-based (fixed-size frames of 1152 samples)
- **Fixed vs variable frame size:** Fixed frame size — 1152 PCM samples per frame (Layer II). Padding frames used for 44.1 kHz at certain bitrates.

### 2.2 High-Level Encoding Flow (Block Diagram)
```
Input PCM Samples (1152 per channel)
      │
      ▼
[Pre-processing: PCM input, split into 32 subbands by polyphase filter bank]
      │
      ▼
[Frame formation: 384 subband samples per subband (1152/3 granules)]
      │
      ▼
[Psychoacoustic Model: FFT analysis → masking threshold per subband]
      │
      ▼
[Bit Allocation: Iterative MNR optimization, allocating bits to subbands]
      │
      ▼
[Scalefactor extraction: Block companding — divide by scalefactor per subband]
      │
      ▼
[Quantization: Uniform mid-tread quantizer per subband (2–15 levels)]
      │
      ▼
[Bitstream Packing: Header + CRC + Bit Allocation + SCFSI + Scalefactors + Samples]
      │
      ▼
Output Encoded MP2 Bitstream
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | 384 samples / ~8.5 ms | Layer II encoding delay |
| Frame size | 1152 samples | Per channel |
| Max channels | 2 (stereo) + multichannel extension | Up to 5.1 in MPEG-2 |
| Max bit depth | 16-bit input | Internal 20-bit precision in encoder |
| Max sample rate | 48 kHz (MPEG-1), 32 kHz (MPEG-2 multichannel) | |
| Bitrate range | 32–384 kbps | MPEG-1; up to ~1066 kbps with MPEG-2 extension |
| Complexity | O(1) per sample | Fixed 32-subband structure |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
MP2 files are bare audio streams — there is no native container/file magic. Files typically open with the first 12 bits set to `0xFFF`, the MPEG audio sync word:

```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  12 bits 111111111111     (sync)    MPEG audio frame sync
0x0001  4 bits  [ID bits]      —         1=MPEG-1, 0=MPEG-2
0x0001  2 bits  [Layer bits]    —         01=Layer I, 10=Layer II, 11=Layer III
...
```

In RIFF/WAV containers, MP2 audio uses the standard RIFF/WAVE header. In MPEG-1/MPEG-2 Program Streams (`.mpg`), the sync word is at file start.

### 3.2 File-Level Header Layout
MP2 as a raw stream has no file header — only frame headers. When wrapped in a container:

**RIFF/WAV (MP2-in-WAV):**
```
Offset   Size    Field Name              Type        Valid Range   Description
-------  ------  ----------------------  ----------  -----------   ---------------------------
0x0000   4B      RIFF Magic             char[4]     "RIFF"       File container magic
0x0004   4B      File Size              uint32 LE   —            File size minus 8 bytes
0x0008   4B      Format Magic          char[4]     "WAVE"       RIFF wave chunk
0x000C   4B      fmt  Magic            char[4]     "fmt "       Format chunk identifier
0x0010   4B      fmt  Chunk Size       uint32 LE   16 or 18     Chunk size (16=PCM, 18=extra)
0x0014   2B      Audio Format          uint16 LE   0x0050       MP2 = 0x0050 (WAVE_FORMAT_MPEG)
0x0016   2B      Channels              uint16 LE   1–2          1=mono, 2=stereo
0x0018   4B      Sample Rate           uint32 LE   32000,44100,48000  Samples per second
0x001C   4B      Byte Rate             uint32 LE   dependent    Bytes per second (=bitrate/8)
0x0020   2B      Block Align           uint16 LE   dependent    Frame size in bytes
0x0022   2B      Extra Params Size     uint16 LE   22           Size of extra MP2 params
0x0024   2B      Head Bitrate Index    uint16 LE   0x0001–0x000E  See bitrate index table
0x0026   1B      Mode                  uint8       0–3          0=stereo,1=jstereo,2=dual,3=mono
...      ...     ...                   ...         ...           Additional WAV fields
```

### 3.3 Frame / Block Header Layout
```
Bit Offset   Bit Width   Field Name              Type      Description
----------   ---------   --------------------    ------    ---------------------------
0            12          Sync Word               uint      Must be 0xFFF (all 1s)
12           1           ID                     uint      1=MPEG-1, 0=MPEG-2
13           2           Layer                  uint      01=Layer I, 10=Layer II, 11=Layer III
15           1           Protection             uint      1=no CRC, 0=CRC present
16           4           Bitrate Index          uint      See bitrate table below
20           2           Sample Rate Index      uint      00=44.1,01=48,10=32,11=reserved
22           1           Padding                bool       1=padding slot present
23           1           Private                bool       Can be used for ancillary data
24           2           Mode                   uint      00=stereo,01=joint stereo,10=dual,11=mono
26           2           Mode Extension         uint      Joint stereo band grouping
28           1           Copyright              bool       1=copyrighted
29           1           Original/Copy          bool       1=original media
30           2           Emphasis              uint       De-emphasis: 00=none,01=50/15μs,11=CCITT J.17
```

#### Bitrate Index Table
| Index (hex) | Bitrate (kbps) | Notes |
|-------------|---------------|-------|
| 0x0 | Free bitrate | Variable bitrate (not common in Layer II) |
| 0x1 | 32 | Lowest supported bitrate |
| 0x2 | 48 | |
| 0x3 | 56 | |
| 0x4 | 64 | |
| 0x5 | 80 | |
| 0x6 | 96 | |
| 0x7 | 112 | |
| 0x8 | 128 | Common for general use |
| 0x9 | 160 | |
| 0xA | 192 | Broadcast standard |
| 0xB | 224 | |
| 0xC | 256 | |
| 0xD | 320 | Highest Layer II in MPEG-1 |
| 0xE | 384 | Professional applications |
| 0xF | Bad/reserved | Must not be used |

#### Sample Rate Index Table
| Index | MPEG-1 Sample Rate | MPEG-2 Sample Rate |
|-------|-------------------|-------------------|
| 00 | 44,100 Hz | 22,050 Hz |
| 01 | 48,000 Hz | 24,000 Hz |
| 10 | 32,000 Hz | 16,000 Hz |
| 11 | Reserved | Reserved |

### 3.4 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Unsigned integer | No | Not defined in MPEG audio spec |
| 16-bit | Signed integer | Yes | Standard input format |
| 20-bit | Signed integer | Partial | Internal precision; 20-bit input requires dithering to 16-bit |
| 24-bit | Signed integer | No | Not supported by standard encoder |
| 32-bit | Signed integer | No | Not supported |
| 32-bit | IEEE float | No | Not supported |
| 64-bit | IEEE float (double) | No | Not supported |

#### Supported Sample Rates
| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 8000 | Telephone | No | Not defined |
| 11025 | — | No | Not defined |
| 16000 | Wideband voice | No | Not defined in MPEG-1 |
| 22050 | — | Yes (MPEG-2 only) | |
| 32000 | Broadcast | Yes (MPEG-2) | DVD-Video PAL for 32 kHz tracks |
| 44100 | CD audio | Yes | Primary rate |
| 48000 | Professional | Yes | Primary rate for broadcast |
| 88200 | 2× CD | No | Not defined |
| 96000 | High-res | No | Not defined |
| 176400 | 4× CD | No | Not defined |
| 192000 | High-res max | No | Not defined |
| 352800 | DXD | No | Not defined |
| 384000 | Ultra-high-res | No | Not defined |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Pre-Processing Stage
- **DC offset removal:** Not explicitly defined in spec — encoder may optionally apply
- **Pre-emphasis filter:** Not applied in MPEG audio encoding (handled at system level via emphasis flag in header)
- **Windowing function:** Applied by the QMF polyphase filter bank internally (raised-cosine window within filter)
- **Level normalization:** Block companding via scalefactors — each subband's 12-sample block is normalized by a scalefactor
- **Stereo decorrelation pre-step:** For joint stereo (MS): L = (L+R)/√2, R = (L−R)/√2 computed before quantization

### 4.2 Analysis / Transform Stage

#### Transform Type: QMF Polyphase Filter Bank (32 bands)
```
Parameters:
  Number of subbands:  32 (uniformly distributed in frequency)
  Filter length:       512 taps (MPEG-1) per channel
  Downsample factor:   32× (1 sample per subband per input sample)
  Frame input:         1152 PCM samples → 36 subband samples per subband
  Granule grouping:    3 groups of 12 samples each per subband
  Overlap:            Critically sampled (no overlap — perfect reconstruction)
  Window function:    Polyphase filter with 64× downsampling per 32-band output
```

**QMF Filter Bank Formula:**

The analysis filter bank computes 32 subband signals s[i][n] from input x[n]:
```
s[i][n] = Σ(k=0 to 511) x[n·32 + k] · C_i[k mod 32][floor(k/32)]
```
where C_i are the 32 polyphase filter coefficients (512 total = 32×16 taps per phase).

The synthesis filter bank reconstructs the audio from subband samples using the same filter structure mirrored.

### 4.3 Psychoacoustic Model (Lossy Only)

#### Model Version
- **Model:** MPEG Psychoacoustic Model 1 (ISO 11172-3 Annex D) or Model 2 (more complex)
- **Analysis window:** 1024-point FFT (Model 1) or 2048-point FFT (Model 2) at input sample rate

#### Masking Thresholds
```
Absolute Threshold of Hearing (ATH):
  ATH(f) = 3.64·(f/1000)^(-0.8) - 6.5·exp(-0.6·(f/1000 - 3.3)^2) + 10^(-3)·(f/1000)^4 dB

Simultaneous Masking:
  Masking slope (upward):   ~5 dB/Bark (limited range)
  Masking slope (downward): ~15 dB/Bark (steeper for spread masking)
  Masking onset:            ~2 ms before masker

Temporal Masking:
  Pre-masking:  ~20 ms (poorly modeled in Layer II)
  Post-masking: ~100–200 ms
```

#### Bit Allocation Algorithm
```
1. Compute FFT of input block (1024 or 2048 samples)
2. Compute masking threshold T[k] for each critical band k
3. Map subband energies to masking thresholds via band grouping
4. Compute Signal-to-Mask Ratio (SMR) for each subband: SMR[i] = Signal_Level[i] / T[i]
5. Initialize all bit allocations to 0
6. Iterate:
   - Find subband with minimum MNR = (SNR[i] - SMR[i]) where SNR[i] is quantizer SNR
   - Increment bit allocation for that subband
   - Recompute SNR[i] based on new quantization step size
   - Repeat until all bits exhausted or no subband benefits
7. Transmit bit allocation indices for all 32 subbands
```

### 4.4 Quantization
- **Type:** Uniform mid-tread quantizer with block companding
- **Step sizes:** Determined by bit allocation index (0–15 levels = 2^1 to 2^15 = 2 to 32768 levels)
- **Block companding:** Each 12-sample group in a subband is divided by a scalefactor (6-bit index → 0–63 dB reduction)
- **Scalefactor bands:** Subbands are grouped; one scalefactor per group per 12-sample block
- **Dequantization:** `sample = (codeword + 0.5) × step_size` × scalefactor_multiplier
- **Noise shaping:** Basic signal-to-quantization-noise ratio (SQNR) optimization; no advanced noise shaping

### 4.5 Stereo Encoding Modes
| Mode | Description | Condition for Selection | Bitrate Range |
|------|-------------|------------------------|---------------|
| Dual Mono | L and R encoded independently | Explicitly requested | any |
| Stereo | L/R encoded independently, shared frame | default | any |
| Joint Stereo / MS | M=(L+R)/2, S=(L−R)/2 encoded separately | When S energy < M energy | < any bitrate |
| Intensity Stereo | Share high-freq envelope, M/S at low freq | High compression | < 96 kbps typical |

### 4.6 Entropy / Lossless Coding Stage
```
Method: None for Layer II audio data (raw quantized samples)

Layer II does NOT use Huffman coding for audio data — unlike Layer III.
Audio samples are transmitted as raw binary codes after quantization.

Bit allocation + scalefactor data: encoded with fixed-width fields per subband
  - 4 bits per subband for bit allocation index
  - 6 bits per scalefactor
  - Raw PCM bits per sample (1–15 bits based on allocation)
```

### 4.7 Encoder Settings / Quality Modes
| Quality Setting | Bitrate Range | Intended Use Case | Transparent? |
|-----------------|---------------|-------------------|--------------|
| Lowest | 32–48 kbps | Voice, extremely low bandwidth | No |
| Low | 56–80 kbps | Low-bitrate speech | No |
| Medium | 96–128 kbps | General lossy | Marginal |
| High | 160–192 kbps | Near-CD quality | Mostly (most content) |
| Highest | 224–384 kbps | Broadcast / archival | Yes (most content) |

**Quality comparison:** MP2 at 192 kbps ≈ MP3 at ~128 kbps in perceived quality. MP2 requires ~30–50% higher bitrate than MP3 for equivalent quality due to its simpler subband-only approach (no hybrid MDCT).

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. Scan bitstream for sync word: 12 consecutive 1-bits (0xFFF)
2. Validate candidate frame:
   a. Verify ID bit (MPEG-1 or MPEG-2)
   b. Verify Layer bits = 10 (Layer II)
   c. Check bitrate index ≠ 0xF (reserved)
   d. Check sample rate index ≠ 0x3 (reserved)
   e. Compute frame_size = (144 × bitrate) / sample_rate [+ padding]
   f. Verify next frame has valid sync at computed offset
3. If validation fails: advance 1 bit and retry
4. Check CRC if protection bit = 0
5. Max failed sync attempts before error: 4096 bytes (implementation-defined)
```

#### Seeking
- **CBR seeking:** `byte_offset = frame_number × frame_size`
- **VBR seeking:** No built-in seek table in raw MP2. Use frame index or full scan.
- **Seek precision:** Frame-accurate (not sample-accurate without index)
- **In containers:** WAV seek table (`data` chunk offset + index), MPEG PS timestamp-based seeking

### 5.2 Core Decode Pipeline
```
1. Read frame header (32 bits = 4 bytes)
   ├── Verify sync word = 0xFFF
   ├── Parse ID, Layer, Protection bits
   ├── Parse bitrate index → actual bitrate
   ├── Parse sample rate index → sample rate
   ├── Check padding flag
   └── Compute frame_size = (144 × bitrate) / sample_rate [+ padding]

2. Read CRC (16 bits) if protection = 0

3. Read bit allocation (32 × 4 bits = 128 bits for stereo)

4. Read scalefactor selection info (SCFSI) per subband (2 bits per subband)

5. Read scalefactors per subband per granule
   └── 6-bit scalefactor index per scalefactor band per channel

6. Read quantized subband samples per subband per granule
   └── Raw binary codes, 1–15 bits per sample based on allocation

7. Inverse quantization per sample
   └── value = (codeword + 0.5) × step_size × scalefactor_multiplier

8. Apply scalefactor decompanding (multiply by scalefactor value)

9. Stereo processing (if joint stereo):
   ├── MS: L = (M + S) / √2 ; R = (M - S) / √2
   └── Intensity: reconstruct from shared envelope + angle

10. Inverse QMF polyphase synthesis filter bank
    └── 32 subband samples → 32×12 = 384 output samples per block
    └── 3 blocks × 384 = 1152 output samples per frame

11. Output to PCM buffer (16-bit signed integer)
```

### 5.3 Error Concealment
- **Corrupt frame detection:** CRC-16 check (if enabled), sync word validation, out-of-range bit allocation values
- **Concealment method:** Muting (set frame to silence) or interpolate from neighboring frames
- **Maximum consecutive errors before silence:** Implementation-defined; typically 3–5 frames before muting

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** Bare bitstream (`.mp2`) — no file-level header
- **Overhead:** ~0% (no container overhead for raw stream)
- **Seeking in native container:** Limited — frame index required for efficient seeking
- **Multiple streams in native container:** No — raw bitstream is single-stream

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| MP4/M4A | No | — | — | MP4 uses AAC, not MP2 |
| Matroska/MKA | Yes | Yes | Full | Via Matroska Audio codec ID |
| OGG | No | — | — | OGG uses Vorbis/Opus |
| WAV | Yes | Yes | Limited (INFO) | RIFF format with MPEG tag |
| AIFF | No | — | — | AIFF is PCM-only |
| MPEG PS (.mpg) | Yes | Limited | None | Program Stream native |
| MPEG TS (.ts) | Yes | Yes | Via PES | Transport Stream audio |
| DAB | Yes | No | DAB ensemble | DAB broadcast native |
| DVB | Yes | Yes | Via PES | DVB broadcast native |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** ID3v1 (at end of file) or ID3v2.3/ID3v2.4 (at beginning of file) for `.mp2` files
- **Tag block location:** Beginning (ID3v2) or end (ID3v1) of file
- **Tag block identifier:** `49 44 33` (ID3) at offset 0 for ID3v2; `54 41 47` (TAG) at EOF for ID3v1

### 7.2 Standard Tag Fields — Complete Reference
| Tag Field | Internal Key (this format) | Max Length | Character Encoding | Multi-value | Notes |
|-----------|---------------------------|------------|-------------------|-------------|-------|
| Title | TIT2 | 30 chars (ID3v1) / unlimited (ID3v2) | ISO-8859-1 / UTF-16 | No | |
| Artist | TPE1 | 30 chars / unlimited | | No | |
| Album | TALB | 30 chars / unlimited | | No | |
| Album Artist | TPE2 | 30 chars / unlimited | | No | ID3v2.4 only |
| Composer | TCOM | 30 chars / unlimited | | Yes | |
| Genre | TCON | 30 chars / unlimited | | No | ID3v1 genre numbers vs freeform |
| Year / Date | TYER / TDRC | 4 chars / unlimited | | No | |
| Track Number | TRCK | 30 chars / unlimited | | No | Format: "N" or "N/Total" |
| Disc Number | TPOS | 30 chars / unlimited | | No | ID3v2 only |
| Comment | COMM | 30 chars / unlimited | | Yes | |
| Lyrics | USLT | unlimited | | No | ID3v2 only |
| BPM | TBPM | 30 chars | | No | |
| Compilation | TCMP | 1 char | | No | ID3v2 only (boolean) |
| Copyright | TCOP | 30 chars / unlimited | | No | |
| Publisher/Label | TPUB | 30 chars / unlimited | | No | |
| Encoder | TENC | 30 chars / unlimited | | No | Software name |
| Encoder Settings | TSSE | 30 chars / unlimited | | No | Bitrate/quality params |
| ISRC | TSRC | 12 bytes | ASCII | No | International Standard Recording Code |
| MusicBrainz Track ID | — | 36 bytes | ASCII | No | Not standard in ID3v1 |
| MusicBrainz Artist ID | — | 36 bytes | ASCII | No | |
| MusicBrainz Album ID | — | 36 bytes | ASCII | No | |
| ReplayGain Track Gain | — | — | — | No | Not natively in ID3 — use Vorbis Comment in wrapper |
| ReplayGain Track Peak | — | — | — | No | |
| ReplayGain Album Gain | — | — | — | No | |
| ReplayGain Album Peak | — | — | — | No | |
| Cover Art (front) | APIC | up to several MB | Binary (JPEG/PNG) | No | ID3v2 only |
| Arbitrary/Custom | TXXX | unlimited | UTF-8 | Yes | User-defined text frame |

### 7.3 Cover Art Storage
```
Cover art storage format in MP2 (via ID3v2):
  Container type:  APIC frame in ID3v2 tag
  Image formats:   JPEG (recommended), PNG, BMP (if supported)
  Max image size:  Implementation limit (no spec limit)
  Max dimensions:  No spec limit
  Picture types:   APIC type 3 = front cover (primary)
  
  Binary layout (APIC frame):
    [0x00-0x03]  Frame ID: "APIC" (4 bytes)
    [0x04-0x07]  Frame size (4 bytes, syncsafe integer in v2.4, big-endian in v2.3)
    [0x08-0x09]  Frame flags (2 bytes)
    [0x0A]       Text encoding (1 byte: 0=Latin-1, 1=UTF-16, 2=UTF-16BE, 3=UTF-8)
    [0x0B-...]   MIME type (null-terminated string, e.g. "image/jpeg")
    [...]         Picture type (1 byte: 0=Other, 1=32x32, 2=Other, 3=Front cover)
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

**Conflict resolution:** When multiple tag systems are present, read priority: ID3v2.4 > ID3v2.3 > ID3v1.

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   mp2                          # used with -c:a
AV_CODEC_ID:        AV_CODEC_ID_MP2               # C constant in libavcodec/codec_id.h
Format Name (CLI):  mp2, mp3, mpeg audio         # raw bitstream
Encoder(s):         mp2 (native, libavcodec)       # also TwoLAME via external lib
Decoder(s):         mp2, mp3 (auto-detect)        # mp2 and mp3 both decoded by mp2 decoder
Muxer(s):           mp2, mpeg, mpegts, mpegvideo  # raw and container muxers
Demuxer(s):         mp2, mp3, mpeg, mpegts       # auto-detects MP2 bitstream
```

### 8.2 FFmpeg Encoding — Full CLI Reference

```bash
# Encoding to MP2 — complete options reference
ffmpeg -i input.wav \
  -c:a mp2 \
  -b:a 192k \           # CBR bitrate — range: 32k–384k, default: 128k
  -ar 44100 \           # Output sample rate (Hz) — 32000, 44100, 48000
  -ac 2 \               # Output channel count — 1 (mono), 2 (stereo)
  -sample_fmt s16 \     # Sample format: s16 (signed 16-bit)
  output.mp2

# Encoding with TwoLAME encoder (if available, provides higher quality)
ffmpeg -i input.wav \
  -c:a libtwolame \
  -b:a 192k \
  -ar 48000 \
  output.mp2
```

#### Complete FFmpeg Option Table
| Option | Type | Default | Range / Valid Values | Effect on Output |
|--------|------|---------|---------------------|-----------------|
| `-b:a` | int | 128k | 32k–384k | Target bitrate for CBR |
| `-ar` | int | input rate | 32000, 44100, 48000 | Output sample rate |
| `-ac` | int | input channels | 1–2 | Output channel count |
| `-pad_idx` | int | 1 | 0–2 | MPEG padding algorithm variant |
| `-joint_stereo` | bool | 1 | 0=force L/R, 1=MS when beneficial | Enable joint stereo |

### 8.3 FFmpeg Encoding — libavcodec C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/opt.h>
#include <libswresample/swresample.h>

// ─── 1. Find encoder ─────────────────────────────────────────────────────────
const AVCodec *codec = avcodec_find_encoder(AV_CODEC_ID_MP2);
// Alternative by name: avcodec_find_encoder_by_name("mp2");
if (!codec) { fprintf(stderr, "Encoder not found\n"); exit(1); }

// ─── 2. Allocate codec context ───────────────────────────────────────────────
AVCodecContext *ctx = avcodec_alloc_context3(codec);
if (!ctx) { fprintf(stderr, "OOM\n"); exit(1); }

// ─── 3. Set required parameters ──────────────────────────────────────────────
ctx->bit_rate    = 192000;              // bits/sec for CBR
ctx->sample_fmt  = AV_SAMPLE_FMT_S16;  // MP2 encoder accepts S16
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
frame->nb_samples  = ctx->frame_size;  // MP2: ctx->frame_size = 1152
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
- `ctx->frame_size` for MP2 is 1152 samples — use this for `nb_samples`
- MP2 encoder accepts `AV_SAMPLE_FMT_S16` (signed 16-bit) — not float
- For float input (WAV files), use `libswresample` to convert to S16 first
- `avcodec_open2` will fail with `AVERROR(EINVAL)` if sample_fmt or sample_rate are unsupported

### 8.4 FFmpeg Decoding — Full CLI Reference

```bash
# Decode to raw PCM WAV
ffmpeg -i input.mp2 \
  -c:a pcm_s16le \
  -ar 44100 \          # Resample if needed
  -ac 2 \              # Downmix if needed
  output.wav

# Extract specific stream (multi-stream container)
ffmpeg -i input.mpg -map 0:a:0 -c:a copy output.mp2

# Probe format without decoding
ffprobe -v quiet -print_format json -show_streams -show_format input.mp2
```

### 8.5 FFmpeg Decoding — libavcodec C API

```c
// ─── Complete decode program skeleton ────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.mp2", NULL, NULL);
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
ffprobe -v quiet -print_format json -show_format input.mp2 | jq .format.tags

# Write metadata (copy audio, update tags)
ffmpeg -i input.mp2 \
  -c copy \
  -metadata title="Song Title" \
  -metadata artist="Artist Name" \
  -metadata album="Album Name" \
  -metadata track="5/12" \
  -metadata disc="1/2" \
  -metadata date="2024" \
  -metadata genre="Electronic" \
  -metadata comment="Any comment" \
  output.mp2

# Strip all metadata
ffmpeg -i input.mp2 -c copy -map_metadata -1 output.mp2

# Embed cover art
ffmpeg -i input.mp2 -i cover.jpg \
  -c copy \
  -map 0:a -map 1:v \
  -metadata:s:v title="Album cover" \
  -disposition:v attached_pic \
  output.mp2
```

#### FFmpeg Internal Metadata Key Mapping
| Standard Field | FFmpeg Key | MP2 Native Key | Notes |
|----------------|------------|--------------------------|-------|
| Title | title | TIT2 | |
| Artist | artist | TPE1 | |
| Album | album | TALB | |
| Album Artist | album_artist | TPE2 | ID3v2 only |
| Track Number | track | TRCK | FFmpeg uses "N/Total" format |
| Disc Number | disc | TPOS | FFmpeg uses "N/Total" format |
| Genre | genre | TCON | |
| Date/Year | date | TYER / TDRC | |
| Comment | comment | COMM | |
| Composer | composer | TCOM | |
| Copyright | copyright | TCOP | |
| Encoder | encoder | TENC | Auto-set by FFmpeg |

### 8.7 Quality / Fidelity Decision Table
| Use Case | Settings | Expected Size (min) | Notes |
|----------|----------|---------------------|-------|
| Archival / professional | `-c:a mp2 -b:a 384k` | ~26 MB/min | Broadcast master |
| Audiophile lossy | `-c:a mp2 -b:a 256k` | ~17 MB/min | Near-transparent |
| Streaming (high) | `-c:a mp2 -b:a 192k` | ~13 MB/min | DAB broadcast standard |
| Streaming (standard) | `-c:a mp2 -b:a 128k` | ~8.5 MB/min | VCD quality |
| Podcast / voice | `-c:a mp2 -b:a 64k -ac 1` | ~4 MB/min | Mono |
| Mobile / low bandwidth | `-c:a mp2 -b:a 48k -ac 1` | ~3 MB/min | DAB voice mode |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
MP2 raw streams have no native seek table. In WAV containers, seeking relies on the WAV index (`idx1` chunk) which maps sample offsets to data positions.

### 9.2 Gapless Playback Data
```
Encoder delay:   0 samples (Layer II adds no systematic delay beyond filter bank)
Padding:         0 samples (or 1 padding frame at 44.1 kHz when padding=1)
Storage location: N/A (no gapless metadata in MP2)
Example value:    Not applicable — MP2 gapless handled at player level
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | Frame-based; start playback after 1 frame |
| Algorithmic encoder delay | 384 samples / ~8.5 ms | Encoder lookahead + filter bank |
| Live encoding feasible | Yes | Low delay, fixed frame size |
| HTTP progressive download | Yes | Frame-based, no index required |
| HTTP Live Streaming (HLS) | Partial | Can be muxed into MPEG-TS for HLS |
| DASH streaming | Yes | MPEG-TS or MP4 mux |
| WebRTC / RTP transport | Yes | Via RTP payload type for MPEG audio |
| Minimum decode buffer | 1 frame / ~26 ms | |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | Layout Name | Channel Order | FFmpeg Layout Constant |
|----------|-------------|---------------|----------------------|
| 1 | Mono | C | AV_CHANNEL_LAYOUT_MONO |
| 2 | Stereo | L, R | AV_CHANNEL_LAYOUT_STEREO |
| 3 | — | L, R, C | AV_CHANNEL_LAYOUT_2_1 |
| 4 | — | L, R, SL, SR | AV_CHANNEL_LAYOUT_2_2 |
| 5 | — | L, R, C, BL, BR | AV_CHANNEL_LAYOUT_5_0 |
| 6 | 5.1 | FL, FR, C, LFE, SL, SR | AV_CHANNEL_LAYOUT_5_1 |

**Note:** 5.1 channel support requires MPEG-2 Audio Layer II (ISO/IEC 13818-3), not the original MPEG-1.

### 11.2 Downmix Coefficients (5.1 → Stereo)
```
L_out = L_in + (C_in × 0.7071) + (SL_in × 0.7071)
R_out = R_in + (C_in × 0.7071) + (SR_in × 0.7071)
LFE:  discarded (or mixed with coefficient 0.0–1.0 based on implementation)

FFmpeg downmix command:
ffmpeg -i input_5_1.mp2 \
  -af "pan=stereo|FL=FC+0.707*FC+0.707*BL|FR=FC+0.707*FC+0.707*BR" \
  output_stereo.mp2
```

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 16-bit | Fixed at 16-bit PCM input/output |
| Max sample rate | 48,000 Hz | MPEG-1; 32 kHz in MPEG-2 multichannel |
| Float support | None | Only integer PCM |
| DSD support | No | |
| 20-bit support | No | Internal encoder precision may be higher |
| 24-bit support | No | Not defined in MPEG audio spec |

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

| Platform / API | Encode | Decode | FFmpeg Flag | Notes |
|----------------|--------|--------|-------------|-------|
| NVIDIA NVENC | No | — | — | No hardware MP2 encoder |
| NVIDIA NVDEC | — | No | — | No hardware MP2 decoder |
| Intel QSV | No | No | — | Not supported |
| Apple VideoToolbox / AudioToolbox | No | No | — | No hardware MP2 support |
| Android MediaCodec | No | Partial | — | System decoders may support |
| VA-API (Linux) | No | No | — | Not supported |

**Note:** MP2 is universally decoded in software. Hardware support exists in dedicated broadcast ASICs and legacy DVD/DAB decoder chips, but not in modern GPU APIs.

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Encoder-Specific Issues
| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| Default bitrate too low (128k) | All | Always specify `-b:a` explicitly |
| 44.1 kHz padding frame handling | All | Spec-compliant; no workaround needed |
| No VBR support in native encoder | All | Use TwoLAME for VBR |
| Mono/stereo mode auto-detection | All | Specify `-ac` explicitly |

### 14.2 Interoperability Issues
- **Broadcast encoder → consumer decoder:** Some professional encoders use non-standard emphasis settings; consumer players may not handle correctly
- **Files with ID3v1 only:** Many professional MP2 decoders ignore ID3v1; use ID3v2 for compatibility
- **Files with non-standard sample rates (22.05 kHz MPEG-2):** Limited decoder support outside broadcast equipment

### 14.3 Edge Cases to Handle in Converter
- **Empty file / 0-sample stream:** Return empty output; do not crash
- **File < 1 frame of audio:** Encode single partial frame if possible; otherwise pad with silence
- **All-silence audio:** Encodes efficiently; all bit allocations = 0
- **DC offset (non-zero mean):** Encoder handles; DC component in subband 0
- **Full-scale sine (0 dB):** May cause clipping in quantized subbands — encoder applies clipping prevention
- **File with corrupt header:** Skip to next sync word; report corruption
- **Truncated file:** Decode up to last complete frame; truncate gracefully
- **Sample rate not supported by codec:** Error out or auto-resample to nearest supported rate
- **Channel count not supported:** Auto-downmix mono to stereo or vice versa

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM MP2

| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|--------------|
| → FLAC | `ffmpeg -i in.mp2 -c:a flac -compression_level 8 out.flac` | All tags (via ID3→Vorbis rewrite) | Lossless decode |
| → ALAC | `ffmpeg -i in.mp2 -c:a alac out.m4a` | Partial (tag mapping) | Lossless decode |
| → MP3 | `ffmpeg -i in.mp2 -c:a libmp3lame -q:a 0 out.mp3` | Tags via ID3v2.3 | Generation loss |
| → AAC | `ffmpeg -i in.mp2 -c:a aac -b:a 256k out.m4a` | Partial | Generation loss |
| → Opus | `ffmpeg -i in.mp2 -c:a libopus -b:a 128k out.opus` | Vorbis Comments | Generation loss |
| → WAV | `ffmpeg -i in.mp2 -c:a pcm_s16le out.wav` | RIFF INFO tags | Lossless decode |
| → OGG Vorbis | `ffmpeg -i in.mp2 -c:a libvorbis -q:a 6 out.ogg` | Vorbis Comments | Generation loss |

### 15.2 Converting TO MP2

| Source | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|--------------|
| FLAC → | `ffmpeg -i in.flac -c:a mp2 -b:a 192k out.mp2` | Vorbis → ID3v2 mapping | Lossless decode, lossy encode |
| WAV → | `ffmpeg -i in.wav -c:a mp2 -b:a 192k out.mp2` | Limited (RIFF INFO → ID3v2) | Lossless decode |
| MP3 → | `ffmpeg -i in.mp3 -c:a mp2 -b:a 192k out.mp2` | ID3v2.3 → ID3v2 | Generation loss |
| AAC → | `ffmpeg -i in.m4a -c:a mp2 -b:a 192k out.mp2` | Partial | Generation loss |
| Vorbis → | `ffmpeg -i in.ogg -c:a mp2 -b:a 192k out.mp2` | Vorbis → ID3v2 | Generation loss |

### 15.3 Lossless Round-Trip Verification
```bash
# Encode
ffmpeg -i original.wav -c:a mp2 -b:a 192k output.mp2

# Decode back
ffmpeg -i output.mp2 -c:a pcm_s16le decoded.wav

# Compare checksums (MP2 is lossy — checksums will NOT match)
# Instead verify audio quality / listening test
# For archival: always keep the original WAV or lossless source
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| FFmpeg native (libavcodec) | C | LGPL 2.1+ | 7/10 | 10/10 | https://ffmpeg.org |
| TwoLAME | C | LGPL | 8/10 | 10/10 | https://www.twolame.org |
| ISO reference (ISO/IEC 11172-3) | C | Royalty-bearing | Reference | Reference | Via ISO/IEC |
| Rockbox | C | Various | — | 10/10 | https://www.rockbox.org |
| libmad | C | GPL | 7/10 | 10/10 | https://www.underbit.com/products/mad/ |

### Build Instructions (for bundling in converter app)
```bash
# Build TwoLAME from source (better quality than FFmpeg native MP2)
git clone https://github.com/njh/twolame.git
cd twolame
./configure --prefix=/usr/local --disable-shared --enable-static
make -j$(nproc)
make install

# Link into your project:
# LDFLAGS += -L/usr/local/lib -ltwolame
# CFLAGS  += -I/usr/local/include
```

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **ISO/IEC 11172-3:1993:** Coding of moving pictures and associated audio for digital storage media at up to about 1.5 Mbit/s — Part 3: Audio (MPEG-1 Audio)
- **ISO/IEC 13818-3:1998:** Generic coding of moving pictures and associated audio information — Part 3: Audio (MPEG-2 Audio, backward compatible extension)
- **RFC 2250:** RTP Payload Format for MPEG1/MPEG2 Audio (defines RTP packaging)

### Technical Resources
- FFmpeg codec documentation: `ffmpeg -h encoder=mp2` or https://ffmpeg.org/ffmpeg-codecs.html
- FFmpeg muxer documentation: https://ffmpeg.org/ffmpeg-formats.html
- Multimedia Wiki MP2: https://wiki.multimedia.cx/index.php/MPEG-1_Audio
- Hydrogenaudio MP2 discussion: https://hydrogenaud.io/
- Library of Congress format entry: https://www.loc.gov/preservation/digital/formats/fdd/fdd000338.shtml

### Academic Papers
- *Coding of Moving Pictures and Associated Audio for Digital Storage Media at up to 1.5 Mbit/s*, ISO/IEC 11172-3, 1993
- Brandenburg, K. et al., *ASPEC and MRS — The ISO/MPEG Audio Coding Proposals*, AES 91st Convention, 1991
- Stoll, G. & Dehery, Y.F., *MPEG/Audio — A standardized scheme for high quality audio coding at 128 kbit/s per channel*, AES 10th International Conference, 1991

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] Identify FFmpeg `./configure` flags needed to enable MP2 (enabled by default in native builds)
- [ ] Verify `ffmpeg -encoders` output confirms mp2 encoder is available
- [ ] Verify `ffmpeg -decoders` output confirms mp2 decoder is available
- [ ] No external library dependency needed (native libavcodec)
- [ ] Handle platform compatibility (MP2 encoding works on all FFmpeg platforms)

### Encoding Pipeline
- [ ] Convert input sample format to required `AV_SAMPLE_FMT_S16` using libswresample
- [ ] Handle fixed-frame-size encoders (`ctx->frame_size` = 1152)
- [ ] Implement proper flush/drain at end of stream (send NULL frame)
- [ ] Validate that input sample rate is in {32000, 44100, 48000}
- [ ] Validate that channel layout is supported (mono/stereo only for MPEG-1)
- [ ] Always specify `-b:a` explicitly — default 128k may be too low for quality use cases

### Decoding Pipeline
- [ ] Implement sync word search (0xFFF pattern) for stream recovery
- [ ] Handle AVERROR(EAGAIN) in receive_packet loop correctly
- [ ] Flush decoder at EOF (send NULL packet)
- [ ] Handle sample format conversion from decoder output format (S16)
- [ ] Handle mono/stereo output layout correctly

### Metadata
- [ ] Read all standard ID3v1 and ID3v2 tag fields (see Section 7.2 table)
- [ ] Map ID3v2 tag fields through standard key mapping (Section 8.6 table)
- [ ] Read cover art (APIC frame) and preserve as JPEG/PNG binary
- [ ] Write all standard tag fields as ID3v2.3 (most compatible)
- [ ] Embed cover art as APIC frame
- [ ] Handle tag encoding (ISO-8859-1 for Latin-1, UTF-16 for Unicode)
- [ ] Handle conflicting ID3v1 + ID3v2 (prioritize ID3v2)

### Quality & Verification
- [ ] Provide bit-exact verification for lossless conversions (source → decode)
- [ ] Implement progress reporting from FFmpeg stats output
- [ ] Implement error detection and partial-file recovery
- [ ] Test with: silence, full-scale, clipped, mono, stereo files

### Edge Cases
- [ ] Handle files with corrupt or missing headers (sync search)
- [ ] Handle files with 0 samples (empty output)
- [ ] Handle sample rate mismatch (resample via libswresample)
- [ ] Handle channel count mismatch (auto-downmix or upmix)
- [ ] Handle very short files (< 1 frame)
- [ ] Handle non-standard emphasis settings in header

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
