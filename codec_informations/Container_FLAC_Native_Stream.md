# Free Lossless Audio Codec (FLAC) Native Stream Format — Deep Technical Reference
> **Category:** Container
> **File Extensions:** `.flac`
> **MIME Types:** `audio/flac`, `audio/x-flac`
> **Standardization Body:** IETF RFC 9639 (December 2024)
> **Primary Specification:** RFC 9639 — https://www.rfc-editor.org/rfc/rfc9639
> **Patent Status:** Patent-free — no known patent claims
> **License:** BSD (Xiph.Org) / GPL-compatible
> **Current Version:** 1.0 (stable since 2001; RFC 9639 formalizes it)
> **Active Development:** Yes — Xiph.Org Foundation; last libFLAC release 2024

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Josh Coalson (initially), Xiph.Org Foundation (after 2003)
- **Year Created:** 2000 (development began); 1.0 final released July 20, 2001
- **Original Purpose:** Provide a free, open-source, patent-unencumbered lossless audio codec optimized for streaming, seeking, and archival use cases — competing with proprietary solutions like WMA Lossless and Apple Lossless
- **Problem with Predecessors:** Shorten (SHN) was the main open lossless option but lacked metadata, seeking, and streaming support. FLAC was designed from scratch with streaming, seeking, and metadata as first-class features.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| 0.5 | 2000 | Early development releases |
| 1.0 | 2001 | Stable format, public release |
| 1.1 | 2003 | Added SEEKTABLE, CUESHEET metadata; joined Xiph.Org |
| 1.2 | 2007 | 32-bit float encoding, mid-side stereo improvements |
| 1.3 | 2013 | Streaming improvements, larger file support |
| 1.4 | 2022 | Security hardening, large file support |
| RFC 9639 | 2024 | Formal IETF standardization |

### 1.3 Current Adoption
- **Primary use cases today:** Music archival, high-resolution audio distribution, game soundtracks, streaming, podcast archival, studio use
- **Platforms with native support:** Windows (via software), macOS (native QuickTime), iOS (native), Android (native since 4.1), Linux (native via gstreamer/FFmpeg), all major DAWs, game engines
- **Major services using this format:** Bandcamp (for high-res sales), some streaming services, archival institutions, Spotify HiFi (historical)
- **Hardware support:** Excellent — Sonos, Bluesound, Roon, Squeezebox, Network Audio Players, gaming consoles (PS3, PS4, Xbox via streaming)
- **Status:** Dominant open-source lossless format; well-established

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 FLAC as a Container vs. Codec
FLAC is simultaneously both a **codec** (encoding/decoding algorithm) and a **container format** (stream structure). The FLAC stream format defines how encoded audio data, metadata, and error detection are organized. The container is self-describing and self-contained — no external wrapper is required.

The FLAC bitstream structure:
```
fLaC marker (4 bytes)  ← always present at byte offset 0
        │
        ▼
STREAMINFO metadata block (required, always first)  ← global stream parameters
        │
        ▼
Optional metadata blocks (any number, any order):
        ├── APPLICATION       (third-party app data)
        ├── SEEKTABLE         (seek index)
        ├── VORBIS_COMMENT    (metadata tags)
        ├── CUESHEET          (track indices)
        ├── PICTURE           (cover art)
        └── PADDING          (reserved space)
        │
        ▼
Audio frames [1..N]:
        ├── Frame header (sync code, block size, sample rate, channel mode, etc.)
        ├── Subframe [1..channels] (encoded channel data)
        ├── Zero-padding to byte boundary
        └── Frame CRC-16 (footer)
```

### 2.2 High-Level Encoding Flow
```
Input PCM Samples (integer or float)
      │
      ▼
[Channel Decorrelation — optional]
Mid-side stereo transformation on frame-by-frame basis
Determined per-frame by encoder
      │
      ▼
[Prediction — Fixed or LPC]
For each channel, predict samples using:
  Fixed predictors (orders 0–4): predefined coefficients, no stored coefficients
  LPC predictor (orders 1–32): stored quantized coefficients, up to 15-bit precision
      │
      ▼
[Residual Computation]
residual[n] = sample[n] - predicted_sample[n]
      │
      ▼
[Partitioned Rice Coding]
Split residual into partitions
Each partition: encode Rice parameter (4 or 5 bits) + unary quotient + binary remainder
      │
      ▼
[Bit Packing — Frame Assembly]
Assemble: frame header → subframes → padding → CRC-16 footer
      │
      ▼
Output: FLAC Frame
```

### 2.3 Key Design Parameters
|| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | 0 samples | Frame-based, no lookahead required |
| Frame size | 16–65535 samples | Variable block size; fixed block size is common |
| Max channels | 8 | Linear channel ordering |
| Max bit depth | 32-bit integer, 32-bit float | |
| Max sample rate | 1048570 Hz | Limited by 20-bit sample rate field in STREAMINFO |
| Bitrate range | ~600–1700 kbps | Depends on content, block size, compression level |
| Compression levels | 0–8 (9 total) | 0=fastest, 8=smallest |
| Complexity | O(n × channels) | Encode: high; Decode: low |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       66 4C 61 43     fLaC     FLAC stream marker
```

The `fLaC` magic `0x664C6143` must appear at the very start of every FLAC file or stream. This marker indicates that what follows is a FLAC bitstream.

### 3.2 Metadata Block Header (Common to All Blocks)
Every metadata block is preceded by a 4-byte header:

```
Offset  Size   Field Name              Type        Valid Range   Description
------  -----  ---------------------  ----------  ------------  ---------------------------
0x0000  1B     Last-metadata-block flag  uint8     bit 7: 1=last block, 0=more follow
        7B     Metadata block type     uint7       0–6          Block type identifier
0x0001  3B     Metadata block length  uint24 BE   any          Length of block data in bytes
```

**Metadata Block Types:**
| Type ID | Name | Required | Notes |
|---------|------|----------|-------|
| 0 | STREAMINFO | Yes (first) | Global stream parameters |
| 1 | APPLICATION | No | Third-party application data |
| 2 | SEEKTABLE | No | Seeking index |
| 3 | VORBIS_COMMENT | No | Metadata tags |
| 4 | CUESHEET | No | Track index points |
| 5 | PICTURE | No | Cover art |
| 6 | PADDING | No | Reserved for editing |

### 3.3 STREAMINFO Metadata Block Layout
The STREAMINFO block is always 34 bytes (block data length = 34), and is always the first metadata block.

```
Offset  Size   Field Name                 Bit Range   Type       Valid Range   Description
------  -----  ------------------------  ----------  ---------  ------------  ---------------------------
0       16     Block data length          —           uint16     34            Must be 34
0       16     Minimum block size         0–15        uint16     16–65535     Min frame size in samples
16      16     Maximum block size         16–31       uint16     16–65535     Max frame size in samples
32      24     Minimum frame size         0–23        uint24     0–16777215   Smallest encoded frame in bytes
56      24     Maximum frame size         24–47       uint24     0–16777215   Largest encoded frame in bytes
80      20     Sample rate               0–19        uint20     1–1048570    Samples per second (Hz); 0=unknown
100     3      Channels − 1              20–22       uint3      0–7          channels = value + 1 (1–8)
103     5      Bits per sample − 1       23–27       uint5      0–31         bps = value + 1 (1–32)
108     4      Total samples (high nibble) 28–31      uint4      0–15         Upper 4 bits of 36-bit count
112     32     Total samples (low 32)     0–31       uint32     any          Lower 32 bits of sample count
144     128    MD5 signature              0–127      uint8[16]  nonzero      Unencoded audio data MD5; 0=unknown
```

**STREAMINFO byte layout (34 bytes = 272 bits):**
```
Byte offset  Bit range  Field Name
───────────  ─────────  ───────────────────────────────
0–1          0–15       Minimum block size (uint16 BE)
2–3          0–15       Maximum block size (uint16 BE)
4–6          0–23       Minimum frame size (uint24 BE)
7–9          0–23       Maximum frame size (uint24 BE)
10–11 (bits 0–19)  Sample rate (20 bits)
10–11 (bits 20–22) Channels − 1 (3 bits)
10–11 (bits 23–27) Bits per sample − 1 (5 bits)
11 (bits 28–31) + Total samples upper 4 bits (4 bits)
12–15           Total samples low 32 bits (uint32 BE)
16–31           MD5 signature (16 bytes)
```

### 3.4 Frame Header Layout
Each frame begins with the sync code and header fields:

```
Offset  Bit    Field Name               Bit Width   Type       Valid Range    Description
------  -----  ---------------------  ----------  ---------  -------------  ---------------------------
0       14     Sync code                14          uint       0xFFF8/0xFFF9  Must be 0xFFF8 (fixed) or 0xFFF9 (variable)
14      1      Reserved                 1           uint       0              Must be 0
15      1      Blocking strategy         1           uint       0=fixed, 1=variable block size
16      4      Block size code          4           uint       0–15           See block size table
20      4      Sample rate code         4           uint       0–15           See sample rate table
24      3      Channel assignment       3           uint       0–7            See channel assignment table
27      4      Sample size code         4           uint       0–7            See sample size table
31      1      Reserved                 1           uint       0              Must be 0
32      ?      UTF-8 coded number       variable    uint       —             Frame number (fixed) or sample number (variable)
?       ?      Coding method (if present) variable   —          —             Present if code 6/7 in channel assignment
```

**Block Size Codes:**
| Code | Block Size (samples) |
|------|--------------------|
| 0x0 | Get from STREAMINFO (8-bit: value 192–256) |
| 0x1 | 192 |
| 0x2 | 576 |
| 0x3 | 1152 |
| 0x4 | 2304 |
| 0x5 | 4608 |
| 0x6 | Get from 8-bit header |
| 0x7 | Get from 16-bit header |
| 0x8 | 256 |
| 0x9 | 512 |
| 0xA | 1024 |
| 0xB | 2048 |
| 0xC | 4096 |
| 0xD | 8192 |
| 0xE | 16384 |
| 0xF | 32768 |

**Sample Rate Codes:**
| Code | Sample Rate (Hz) | Notes |
|------|-----------------|-------|
| 0x0 | From STREAMINFO | Inherit from STREAMINFO block |
| 0x1 | 88200 | |
| 0x2 | 176400 | |
| 0x3 | 192000 | |
| 0x4 | 8000 | |
| 0x5 | 16000 | |
| 0x6 | 22050 | |
| 0x7 | 24000 | |
| 0x8 | 32000 | |
| 0x9 | 44100 | CD audio |
| 0xA | 48000 | Professional |
| 0xB | 96000 | |
| 0xC | Get from 8-bit header | |
| 0xD | Get from 16-bit header | |
| 0xE | Get from 8-bit header (÷10) | |
| 0xF | Invalid/reserved | |

**Channel Assignment:**
| Code | Assignment | Channels | Description |
|------|-----------|----------|-------------|
| 0x0 | 1 | 1 | Independent mono |
| 0x1 | 2 | 2 | Independent stereo |
| 0x2 | 3 | 2 | Left+Side (L, R=L−S) |
| 0x3 | 4 | 2 | Right+Side (R, L=R−S) |
| 0x4 | 5 | 2 | Mid+Side (M=L+R, S=L−R) |
| 0x5 | 6 | 2 | Left+Side with independent side |
| 0x6 | 7 | 2 | Right+Side with independent side |
| 0x8–0xF | N | N | N independent channels |

**Sample Size Codes:**
| Code | Bits Per Sample |
|------|----------------|
| 0x0 | From STREAMINFO |
| 0x1 | 8 |
| 0x2 | 12 [NEEDS VERIFICATION] |
| 0x3 | Reserved |
| 0x4 | 16 |
| 0x5 | 20 [NEEDS VERIFICATION] |
| 0x6 | 24 |
| 0x7 | 32 |

### 3.5 Subframe Structure
Each subframe follows the frame header:

```
Bit Offset  Bit Width   Field Name                Description
----------  ---------   ----------------------    ---------------------------
0           1           Subframe type            0=CONSTANT, 1=VERBATIM, etc.
?           ?           Type-specific data       Encoding method data
?           ?           Encoded residual          If predictor was used
--- Zero-padding to byte boundary ---
Last        16          CRC-8 of subframe header  Frame CRC-8
```

**Subframe Types (first 6 bits of type field):**
| Value | Subframe Type | Description |
|-------|---------------|-------------|
| 0 | CONSTANT | Single value for entire block |
| 1 | VERBATIM | Raw samples, no compression |
| 2 | FIXED | Fixed linear predictor, predefined coefficients |
| 3 | LPC | Linear predictor with stored coefficients |

### 3.6 Constant Subframe
```
Bit Offset  Bit Width   Field Name                Description
----------  ---------   ----------------------    ---------------------------
0           6           Type = 0 (CONSTANT)       Must be 0
6           1           Reserved                   Must be 0
7           ?           Unencoded sample          bits_per_sample bits, signed
```

### 3.7 Verbatim Subframe
```
Bit Offset  Bit Width   Field Name                Description
----------  ---------   ----------------------    ---------------------------
0           6           Type = 1 (VERBATIM)       Must be 1
6           1           Reserved                   Must be 0
7           N           Raw samples                block_size × bits_per_sample bits
```

### 3.8 Fixed Predictor Subframe
```
Bit Offset  Bit Width   Field Name                Description
----------  ---------   ----------------------    ---------------------------
0           6           Type = 2 (FIXED)          Must be 2
6           4           Predictor order            0–4 (higher = better for complex signals)
10          1           Reserved                   Must be 0
11          ?           Warm-up samples           order × bits_per_sample bits
?           ?           Encoded residual           Partitioned Rice coding
```

**Fixed Predictor Orders and Coefficients:**
| Order | Coefficients | Best For |
|-------|-------------|----------|
| 0 | None (zero predictor) | Effectively verbatim |
| 1 | [1] | DC signals, ramps |
| 2 | [2, −1] | Linear trends |
| 3 | [3, −3, 1] | Quadratic curves |
| 4 | [4, −6, 4, −1] | Cubic curves |

### 3.9 LPC Subframe
```
Bit Offset  Bit Width   Field Name                Description
----------  ---------   ----------------------    ---------------------------
0           6           Type = 3 (LPC)             Must be 3
6           5           Predictor order            1–32
11          4           Quantized LP precision      0–15 (actual bits = value + 1)
15          5           Quantized LP shift          Signed 2's complement
?           ?           LP coefficients              order × precision bits, signed 2's complement
?           ?           Warm-up samples             order × bits_per_sample bits
?           ?           Encoded residual             Partitioned Rice coding
```

### 3.10 Residual Coding (Partitioned Rice)
```
Bit Offset  Bit Width   Field Name                Description
----------  ---------   ----------------------    ---------------------------
0           2           Residual coding method     0=Rice4, 1=Rice2, 2–3=reserved
2           4           Partition count − 1        0–15 (0=1 partition, 15=16 partitions)
--- For each partition (Rice4: 4-bit params; Rice2: 5-bit params) ---
?           4/5         Rice parameter             k = value (determines coding)
?           unary       Quotient                   Number of 0 bits before 1
?           k bits      Remainder                  Binary remainder
```

### 3.11 Frame Footer
```
Offset  Size   Field Name              Type        Description
------  -----  ---------------------  ----------  ---------------------------
0       2B     CRC-16                  uint16 LE   CRC-16 (polynomial 0x8005) of all bits from frame header start to before this CRC
```

### 3.12 Seekable Metadata Blocks

#### APPLICATION Block (type 1)
```
Offset  Size   Field Name              Type        Description
------  -----  ---------------------  ----------  ---------------------------
0       4B     Application ID         uint32 BE   Registered application identifier
4       N      Application data       bytes       Application-specific data
```

Registered APP IDs:
| ID | Application |
|----|-------------|
| `0x4150` (APTX) | apt-X compression |
| `0x4671` (FLAC) | Xiph.Org internal use |
| `0x5346` (SF01) | SoundFont sample data |

#### SEEKTABLE Block (type 2)
Each seek point is 18 bytes:
```
Offset  Size   Field Name              Type        Description
------  -----  ---------------------  ----------  ---------------------------
0       8B     Sample number           uint64 BE   Sample offset of target frame; 0xFFFFFFFFFFFFFFFF = placeholder
8       8B     Byte offset             uint64 BE   Byte offset of frame from start of stream
16      2B     Frame sample count       uint16 BE   Number of samples in target frame; 0=unknown
```

A seek point with `sample_number = 0xFFFFFFFFFFFFFFFF` is a **placeholder** and must be ignored.

#### VORBIS_COMMENT Block (type 3)
Follows the Vorbis comment specification (same as OGG Vorbis):
```
Offset  Size   Field Name              Type        Description
------  -----  ---------------------  ----------  ---------------------------
0       4B     Vendor string length   uint32 LE   Length of vendor string
4       N      Vendor string          UTF-8       Encoder/vendor name
N+4     4B     Number of comments     uint32 LE   Count of (key=value) pairs
N+8     N      Comment list           [key=val]   Each: 4B length + string
```

#### PICTURE Block (type 5)
```
Offset  Size   Field Name              Type        Description
------  -----  ---------------------  ----------  ---------------------------
0       4B     Picture type           uint32 BE   0=other, 1=32x32 PNG icon, 2=front cover, etc.
4       4B     MIME type length       uint32 BE   Length of MIME type string
8       N      MIME type              string      "image/jpeg" or "image/png"
N+8     4B     Description length      uint32 BE   Length of description string
N+12    M      Description            string      UTF-8 description
N+12+M  4B     Width                  uint32 BE   Image width in pixels
N+16+M  4B     Height                 uint32 BE   Image height in pixels
N+20+M  4B     Color depth             uint32 BE   Bits per pixel
N+24+M  4B     Colors indexed          uint32 BE   0 = not indexed
N+28+M  4B     Picture data length    uint32 BE   Binary data length
N+32+M  P      Picture data           bytes       JPEG/PNG binary image data
```

### 3.13 Sample Format Support
|| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 1-bit | Unsigned integer | No | |
| 8-bit | Signed integer | Yes | Rarely used |
| 16-bit | Signed integer | Yes | Standard |
| 20-bit | Signed integer | Yes | High-res |
| 24-bit | Signed integer | Yes | Common high-res |
| 32-bit | Signed integer | Yes | High-res |
| 32-bit | IEEE float | Yes | Since FLAC 1.2 |
| 64-bit | IEEE float (double) | No | |

### 3.14 Supported Sample Rates
|| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 8000 | Telephone | Yes | |
| 11025 | — | Yes | |
| 16000 | Wideband voice | Yes | |
| 22050 | — | Yes | |
| 32000 | Broadcast | Yes | |
| 44100 | CD audio | Yes | Most common |
| 48000 | Professional | Yes | Standard |
| 88200 | 2× CD | Yes | High-res |
| 96000 | High-res | Yes | Standard hi-res |
| 176400 | 4× CD | Yes | |
| 192000 | High-res max | Yes | |
| 352800 | DXD | Yes | |
| 384000 | Ultra-high-res | Yes | Limited encoder support |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Prediction Algorithm
FLAC uses linear predictive coding (LPC) as its primary predictor. The encoder selects between fixed and LPC predictors on a per-subframe basis.

**Fixed Predictor (orders 0–4):**
```
y[n] = Σ(i=1 to order) c[i] × x[n − i]
where coefficients c[i] are predefined (see table above)
No coefficients are stored in the bitstream — the decoder uses the same predefined values.
```

**LPC Predictor (orders 1–32):**
```
y[n] = Σ(i=1 to order) c[i] × x[n − i]
where c[i] are stored as quantized 4–15 bit signed integers
The quantized coefficients use a shift parameter for scaling.
```

### 4.2 Residual Encoding
After prediction, the residual `e[n] = x[n] − y[n]` is encoded using **partitioned Rice coding**:

**Rice Coding Process:**
```
For each residual value r:
  1. Compute: m = |r|; q = m >> k; remainder = m & ((1 << k) - 1)
  2. Encode quotient: unary code of q (q zeros followed by a one)
  3. Encode remainder: k binary bits
  4. Encode sign: if r < 0, flip the LSB of the remainder

Rice parameter k: chosen to minimize coded size
  = round(log2(mean(|residual|))) for each partition
  Parameter stored as 4 bits (Rice4) or 5 bits (Rice2)
```

**Partition Structure:**
- Frame residual is divided into `partition_count` equal-sized partitions
- Each partition has its own Rice parameter
- Partition sizes are powers of 2 (enforced by encoder)

### 4.3 Channel Decorrelation
FLAC applies channel transformations on a per-frame basis. The encoder tries multiple transforms and picks the best:

| Transform | Code | Formula | Best For |
|-----------|------|---------|----------|
| Independent | 0, 1 | L, R as-is | Uncorrelated channels |
| Left+Side | 2 | L, S where S = R−L | When right channel has lower entropy |
| Right+Side | 3 | R, S where S = L−R | When left channel has lower entropy |
| Mid+Side | 4 | M = (L+R)/2, S = L−R | Stereo music; most common |

### 4.4 Encoder Settings / Quality Modes
|| Compression Level | Encoding Speed | Decode Speed | File Size (vs Level 0) | Notes |
|---|---|---|---|---|
| 0 | ~300×RT | ~100×RT | +0% | Fastest, least compressed |
| 1 | ~100×RT | ~90×RT | -2% | |
| 2 | ~50×RT | ~80×RT | -5% | |
| 3 | ~25×RT | ~70×RT | -8% | |
| 4 | ~15×RT | ~60×RT | -10% | Default |
| 5 | ~8×RT | ~50×RT | -13% | |
| 6 | ~4×RT | ~40×RT | -16% | |
| 7 | ~2×RT | ~30×RT | -19% | |
| 8 | ~1×RT | ~20×RT | -22% | Slowest, smallest |

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. Read fLaC marker (4 bytes) at offset 0
2. Parse STREAMINFO metadata block:
   ├── Verify block data length = 34
   ├── Extract: sample_rate, channels, bits_per_sample, block_size range
   ├── Extract total_samples (36-bit value) and MD5 signature
3. Parse remaining metadata blocks (in order):
   ├── Last-metadata-block flag tells us when audio frames begin
   └── Build seek table from SEEKTABLE block
4. For each frame:
   ├── Find sync code (0xFFF8 or 0xFFF9) at frame start
   ├── Validate: blocking strategy, block size, sample rate, channel assignment
   ├── Verify frame CRC-16 footer
   └── Decode all subframes
```

#### Seeking
FLAC supports **sample-accurate seeking** using the SEEKTABLE:

```
1. Binary search in seek table: find nearest seek point with sample_number ≤ target
2. Seek to byte_offset from seek point
3. Decode forward from seek point until target sample is reached
4. For files without seek table: linear scan from beginning (slow)
```

### 5.2 Core Decode Pipeline
```
1. Read STREAMINFO (mandatory)
2. Read metadata blocks until last-metadata-block flag is set
3. Build seek table from SEEKTABLE block
4. For each frame:
   a. Read frame header (block size, sample rate, channel assignment)
   b. Decode blocking strategy: fixed block size or variable
   c. For each channel subframe:
      - Read subframe type
      - If CONSTANT: read single value, fill block
      - If VERBATIM: read raw samples
      - If FIXED: read warm-up samples, decode residual via Rice decoding, add prediction
      - If LPC: read warm-up samples, read coefficients, decode residual, add prediction
   d. Apply channel decorrelation inverse transform
   e. Verify frame CRC-16
5. Apply 0-padding to byte boundary
6. Return PCM samples
```

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** FLAC native stream format
- **Overhead:** ~0.1–2% (metadata blocks)
- **Seeking in native container:** Yes — via SEEKTABLE, sample-accurate
- **Multiple streams in native container:** No — single audio stream only

### 6.2 Codec-to-Container Compatibility
|| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| FLAC (native) | Yes | Yes (SEEKTABLE) | Full (Vorbis Comments) | Preferred |
| OGG | Yes (OggFLAC) | Yes | Full | Via Ogg container |
| MP4/M4A | Yes (fLaC box) | Yes | Full | Since iTunes 11 |
| Matroska/MKA | Yes | Yes | Full | Via EBML |
| WAV | No | N/A | N/A | Not applicable |
| AIFF | No | N/A | N/A | Not applicable |
| WebM | No | N/A | N/A | Not applicable |

### 6.3 OGG FLAC Mapping
OggFLAC wraps FLAC frames in Ogg pages. The mapping uses:
- Ogg page granule position = sample number (for seeking)
- FLAC frames may be split across Ogg pages or combined
- Codec identification: `Xiph` comment `FRAMING` byte at end of each page

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** Vorbis Comment (as in OGG Vorbis)
- **Tag block location:** Within FLAC stream as metadata block type 4
- **Tag block identifier:** Metadata block header with type = 4

### 7.2 Standard Tag Fields — Complete Reference
|| Tag Field | Internal Key (FLAC) | Max Length | Character Encoding | Multi-value | Notes |
|-----------|---------------------------|------------|-------------------|-------------|---|
| Title | TITLE | unlimited | UTF-8 | Yes | |
| Artist | ARTIST | unlimited | UTF-8 | Yes | |
| Album | ALBUM | unlimited | UTF-8 | Yes | |
| Album Artist | ALBUMARTIST | unlimited | UTF-8 | Yes | |
| Composer | COMPOSER | unlimited | UTF-8 | Yes | |
| Genre | GENRE | unlimited | UTF-8 | Yes | Freeform or standard genres |
| Date | DATE | unlimited | UTF-8 | No | Year or full date |
| Track Number | TRACKNUMBER | unlimited | UTF-8 | No | Often "N" or "N/Total" |
| Track Total | TRACKTOTAL | unlimited | UTF-8 | No | |
| Disc Number | DISCNUMBER | unlimited | UTF-8 | No | |
| Disc Total | DISCTOTAL | unlimited | UTF-8 | No | |
| Comment | COMMENT | unlimited | UTF-8 | Yes | |
| Lyrics | LYRICS | unlimited | UTF-8 | Yes | |
| BPM | BPM | unlimited | UTF-8 | No | |
| Catalog Number | CATALOG | unlimited | UTF-8 | No | |
| Copyright | COPYRIGHT | unlimited | UTF-8 | No | |
| ISRC | ISRC | 12 bytes | ASCII | No | |
| MusicBrainz Track ID | MUSICBRAINZ_TRACKID | 36 bytes | ASCII | No | UUID |
| MusicBrainz Artist ID | MUSICBRAINZ_ARTISTID | 36 bytes | ASCII | No | UUID |
| MusicBrainz Album ID | MUSICBRAINZ_ALBUMID | 36 bytes | ASCII | No | UUID |
| MusicBrainz Album Artist ID | MUSICBRAINZ_ALBUMARTISTID | 36 bytes | ASCII | No | UUID |
| MusicBrainz Release Group ID | MUSICBRAINZ_RELEASEGROUPID | 36 bytes | ASCII | No | UUID |
| MusicBrainz Track Disc ID | MUSICBRAINZ_DISCID | 36 bytes | ASCII | No | |
| MusicBrainz TRM | MUSICBRAINZ_RELEASETRACKID | 36 bytes | ASCII | No | |
| ReplayGain Track Gain | REPLAYGAIN_TRACK_GAIN | unlimited | ASCII | No | Format: "-6.20 dB" |
| ReplayGain Track Peak | REPLAYGAIN_TRACK_PEAK | unlimited | ASCII | No | Format: "0.998459" |
| ReplayGain Album Gain | REPLAYGAIN_ALBUM_GAIN | unlimited | ASCII | No | |
| ReplayGain Album Peak | REPLAYGAIN_ALBUM_PEAK | unlimited | ASCII | No | |
| Cover Art | METADATA_BLOCK_PICTURE | up to 16 MB | Binary | No | PICTURE block |
| Encoder | ENCODER | unlimited | UTF-8 | No | Software name |
| Encoder Settings | ENCODER_OPTIONS | unlimited | UTF-8 | No | Encoding parameters |

### 7.3 Cover Art Storage
```
Cover art storage format in FLAC:
  Container type:  PICTURE metadata block (type 5)
  Image formats:   JPEG (recommended), PNG, BMP, GIF
  Max image size:  16 MB (implementation limit)
  
  FLAC PICTURE block structure:
    [0x00–0x03]  Picture type (4 bytes BE): 0=none, 1=32x32 PNG, 2=front cover (primary)
    [0x04–0x07]  MIME type length (4 bytes BE)
    [0x08–...]   MIME type string (e.g., "image/jpeg", null-terminated)
    [...]         Description length (4 bytes BE)
    [...]         Description string (UTF-8)
    [...]         Width (4 bytes BE), Height (4 bytes BE)
    [...]         Color depth (4 bytes BE), Colors (4 bytes BE = 0 for RGB)
    [...]         Binary image data length (4 bytes BE)
    [...]         Binary image data
```

### 7.4 Metadata Compatibility Matrix
|| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| Vorbis Comments | ✓ | ✓ | ✓ | Highest (native) |
| ID3v1 (at file end) | ✓ | ✓ | Partial | Low |
| ID3v2 (at file start) | ✓ | ✓ | Partial | Low |
| APEv2 | ✗ | ✗ | N/A | Not supported |

**Conflict resolution:** FLAC files use only Vorbis Comments. Some tools may add ID3v1 at the end — read both, prefer Vorbis Comments.

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   flac                      # used with -c:a
AV_CODEC_ID:        AV_CODEC_ID_FLAC
Format Name (CLI):   flac                     # used with -f
Encoder(s):          flac                     # ffmpeg -encoders | grep flac
Decoder(s):          flac                     # ffmpeg -decoders | grep flac
Muxer(s):           flac                     # ffmpeg -muxers | grep flac
Demuxer(s):          flac                     # ffmpeg -demuxers | grep flac
```

### 8.2 FFmpeg Encoding — Full CLI Reference

```bash
# Encoding to FLAC — complete options reference
ffmpeg -i input.wav \
  -c:a flac \
  -compression_level 8 \       # Range: 0–12, default: 5
  -ar 44100 \                  # Output sample rate (Hz)
  -ac 2 \                      # Output channel count
  -sample_fmt s16 \             # Sample format: s16, s32, fltp
  -frame_size 4096 \           # Set encoder frame size (optional)
  output.flac

# High-resolution FLAC
ffmpeg -i input_24bit_96k.wav \
  -c:a flac \
  -compression_level 8 \
  -ar 96000 \
  -ac 2 \
  -sample_fmt s32 \
  output.flac

# Fast encoding (level 0)
ffmpeg -i input.wav \
  -c:a flac \
  -compression_level 0 \
  output.flac
```

#### Complete FFmpeg Option Table
|| Option | Type | Default | Range / Valid Values | Effect on Output |
|--------|------|---------|---------------------|-----------------|
| `-compression_level` | int | 5 | 0–12 | Encode speed vs compression ratio |
| `-ar` | int | from input | 8000–384000 | Output sample rate |
| `-ac` | int | from input | 1–8 | Output channel count |
| `-sample_fmt` | string | s16 | s16, s32 | Sample format |
| `-frame_size` | int | auto | 16–65535 | Encoder block size |
| `-lpc_type` | int | auto | 0=none, 1=fixed, 2=Levinson, 3=cholesky | LPC algorithm |
| `-lpc_passes` | int | 2 | 1–15 | LPC analysis passes |
| `-exact_rice_parameters` | bool | 0 | 0/1 | Exhaustive Rice parameter search |
| `-prediction_order_method` | int | auto | 0–6 | Fixed/LPC order selection |
| `-ch_mode` | int | auto | -1=auto, 0=indep, 1=left/side, 2=right/side, 3=mid/side | Channel decorrelation |
| `-write_to_stdin` | bool | 0 | 0/1 | Read input from stdin |

### 8.3 FFmpeg Encoding — libavcodec C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/opt.h>
#include <libswresample/swresample.h>

// ─── 1. Find encoder ─────────────────────────────────────────────────────────
const AVCodec *codec = avcodec_find_encoder(AV_CODEC_ID_FLAC);
if (!codec) { fprintf(stderr, "Encoder not found\n"); exit(1); }

// ─── 2. Allocate codec context ───────────────────────────────────────────────
AVCodecContext *ctx = avcodec_alloc_context3(codec);
if (!ctx) { fprintf(stderr, "OOM\n"); exit(1); }

// ─── 3. Set required parameters ──────────────────────────────────────────────
ctx->sample_fmt   = AV_SAMPLE_FMT_S16;    // FLAC supports S16, S32
ctx->sample_rate  = 44100;                // Hz — any valid rate
av_channel_layout_copy(&ctx->ch_layout, &(AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO);

// ─── 4. Set encoding options ─────────────────────────────────────────────────
av_opt_set_int(ctx, "compression_level", 8, AV_OPT_SEARCH_CHILDREN);
av_opt_set_int(ctx, "frame_size", 4096, AV_OPT_SEARCH_CHILDREN);

// ─── 5. Open codec ───────────────────────────────────────────────────────────
AVDictionary *opts = NULL;
int ret = avcodec_open2(ctx, codec, &opts);
if (ret < 0) {
    char errbuf[128]; av_strerror(ret, errbuf, sizeof(errbuf));
    fprintf(stderr, "avcodec_open2 failed: %s\n", errbuf);
    exit(1);
}

// ─── 6. Allocate frame and packet ────────────────────────────────────────────
AVFrame  *frame = av_frame_alloc();
frame->format      = ctx->sample_fmt;
frame->sample_rate = ctx->sample_rate;
av_channel_layout_copy(&frame->ch_layout, &ctx->ch_layout);
frame->nb_samples  = ctx->frame_size;   // Fixed at 4096 (or encoder-determined)
av_frame_get_buffer(frame, 0);

AVPacket *pkt = av_packet_alloc();

// ─── 7. Encode loop ─────────────────────────────────────────────────────────
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

// ─── 8. Flush encoder ────────────────────────────────────────────────────────
encode_frame(ctx, NULL, pkt, outfile);

// ─── 9. Cleanup ──────────────────────────────────────────────────────────────
av_frame_free(&frame);
av_packet_free(&pkt);
avcodec_free_context(&ctx);
```

**Critical Notes:**
- `ctx->frame_size` for FLAC is the **encoder block size** — typically 4096 samples
- FLAC is a **frame-based codec**; each output packet is one FLAC frame
- FLAC supports **both fixed and variable block sizes** — if variable, use 0 as `nb_samples` to indicate variable size
- Always check `codec->sample_fmts` for supported formats

### 8.4 FFmpeg Decoding — Full CLI Reference

```bash
# Decode to raw PCM WAV
ffmpeg -i input.flac \
  -c:a pcm_s16le \
  output.wav

# Decode with resampling
ffmpeg -i input.flac \
  -c:a pcm_s16le \
  -ar 48000 \
  output.wav

# Extract with metadata copy
ffmpeg -i input.flac -c:a copy output.flac

# Probe format without decoding
ffprobe -v quiet -print_format json -show_streams -show_format input.flac
```

### 8.5 FFmpeg Decoding — libavcodec C API

```c
// ─── Complete decode program skeleton ─────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.flac", NULL, NULL);
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
            // frm->format = AVSampleFormat (S16 or S32)
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

### 8.6 FFmpeg Metadata Handling

```bash
# Read all metadata as JSON
ffprobe -v quiet -print_format json -show_format input.flac | jq .format.tags

# Write metadata (Vorbis Comments)
ffmpeg -i input.flac \
  -c:a copy \
  -metadata title="Song Title" \
  -metadata artist="Artist Name" \
  -metadata album="Album Name" \
  -metadata album_artist="Album Artist" \
  -metadata track="5/12" \
  -metadata disc="1/2" \
  -metadata date="2024" \
  -metadata genre="Electronic" \
  -metadata REPLAYGAIN_TRACK_GAIN="-6.20 dB" \
  -metadata REPLAYGAIN_TRACK_PEAK="0.998459" \
  output.flac

# Strip all metadata
ffmpeg -i input.flac -c:a copy -map_metadata -1 output.flac

# Embed cover art
ffmpeg -i input.flac -i cover.jpg \
  -c:a copy \
  -metadata:s:v title="Album cover" \
  -metadata:s:v comment="Cover (front)" \
  -disposition:v attached_pic \
  output.flac
```

#### FFmpeg Internal Metadata Key Mapping
|| Standard Field | FFmpeg Key | FLAC Native Key | Notes |
|----------------|------------|--------------------------|-------|
| Title | title | TITLE | |
| Artist | artist | ARTIST | |
| Album | album | ALBUM | |
| Album Artist | album_artist | ALBUMARTIST | |
| Track Number | track | TRACKNUMBER | FFmpeg uses "N/Total" format |
| Disc Number | disc | DISCNUMBER | FFmpeg uses "N/Total" format |
| Genre | genre | GENRE | |
| Date | date | DATE | |
| Comment | comment | COMMENT | |
| Composer | composer | COMPOSER | |
| Copyright | copyright | COPYRIGHT | |
| Encoder | encoder | ENCODER | Auto-set by FFmpeg |
| BPM | handle | BPM | |

### 8.7 Quality / Fidelity Decision Table
|| Use Case | Settings | Expected Size | Notes |
|----------|----------|---------------|-------|
| Archival / best quality | `-c:a flac -compression_level 8` | ~0.55–0.65× uncompressed | |
| High-quality streaming | `-c:a flac -compression_level 5` | ~0.58× uncompressed | Default |
| Fast encode | `-c:a flac -compression_level 0` | ~0.65× uncompressed | Real-time encode possible |
| High-resolution archival | `-c:a flac -compression_level 8 -ar 192000` | Variable | 24-bit/192kHz archival |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
```
FLAC Seektable:
  Location:     Within FLAC stream as metadata block type 2
  Magic:       No magic; identified by metadata block type = 2
  Entry size:  18 bytes per seek point
  Entry format:
    [0x00–0x07]  Sample number  (uint64 BE) — 0xFFFFFFFFFFFFFFFF = placeholder
    [0x08–0x0F]  Byte offset    (uint64 BE) — from stream start
    [0x10–0x11]  Frame sample count (uint16 BE) — samples in target frame
  Max entries:  (block_size − 4) / 18
```

### 9.2 Gapless Playback Data
```
FLAC does not store explicit gapless metadata in the stream.
Encoder delay:  0 samples — FLAC frame-based encoding has zero algorithmic delay
Padding:        0 samples — padding is implicit in frame boundaries
Total samples field in STREAMINFO includes all valid samples.
Seek to sample 0 for true gapless start.
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

|| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | FLAC frames are independently decodable |
| Algorithmic encoder delay | 0 samples | No lookahead required |
| Live encoding feasible | Yes | Streaming mode with fixed block size |
| HTTP progressive download | Yes | Common for music distribution |
| HTTP Live Streaming (HLS) | Yes | FLAC segments in HLS (since HLS r6) |
| DASH streaming | Yes | FLAC as audio codec in MP4/WebM segments |
| WebRTC / RTP transport | Yes | Via Ogg or custom FLAC framing |
| Minimum decode buffer | 1 frame | ~1152–8192 samples depending on block size |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
|| Channels | Layout Name | Channel Order | FFmpeg Layout Constant |
|----------|-------------|---------------|----------------------|
| 1 | Mono | C | AV_CHANNEL_LAYOUT_MONO |
| 2 | Stereo | L, R | AV_CHANNEL_LAYOUT_STEREO |
| 3 | 2.1 | L, R, LFE | AV_CHANNEL_LAYOUT_2POINT1 |
| 4 | Quad | FL, FR, RL, RR | AV_CHANNEL_LAYOUT_QUAD |
| 5 | 5.0 | FL, FR, C, SL, SR | AV_CHANNEL_LAYOUT_5POINT0 |
| 6 | 5.1 | FL, FR, C, LFE, SL, SR | AV_CHANNEL_LAYOUT_5POINT1 |
| 7 | 6.1 | FL, FR, C, LFE, BL, BC, BR | AV_CHANNEL_LAYOUT_6POINT1 |
| 8 | 7.1 | FL, FR, C, LFE, BL, BR, SL, SR | AV_CHANNEL_LAYOUT_7POINT1 |

### 11.2 Downmix Coefficients (5.1 → Stereo)
```
L_out = L + C × 0.7071 + LS × 0.7071
R_out = R + C × 0.7071 + RS × 0.7071
LFE:  discarded

FFmpeg downmix:
ffmpeg -i input_5_1.flac \
  -af "pan=stereo|FL=FC+0.707*FC+0.707*BL|FR=FC+0.707*FC+0.707*BR" \
  output_stereo.flac
```

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

|| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 32-bit integer / 32-bit float | |
| Max sample rate | 1,048,570 Hz | Limited by 20-bit field |
| Float support | Yes | 32-bit IEEE float (since FLAC 1.2) |
| DSD support | No | Not applicable |
| 20-bit support | Yes | High-res standard |
| 24-bit support | Yes | Most common hi-res format |

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

|| Platform / API | Encode | Decode | Notes |
|----------------|--------|--------|-------|
| Native (libFLAC) | Yes | Yes | Reference implementation |
| FFmpeg native | Yes | Yes | Built-in |
| ARM NEON | — | Yes | Hardware acceleration in some builds |
| Apple AudioToolbox | No | Yes | Via macOS/iOS system decoder |
| Android MediaCodec | No | Yes | Native Android supports FLAC decode |
| VA-API | No | No | Not applicable (CPU decode is fast) |

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Encoder-Specific Issues
|| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| LPC precision bug at high compression | < 6.0 | Use level 7 instead of 8 |
| MD5 signature mismatch with float audio | < 4.4 | Ignore MD5 for float sources |
| Corruption on non-seekable output | < 5.0 | Use `-write_to_stdin 0` |

### 14.2 Interoperability Issues
- **FLAC level 0–2 files:** Some hardware players only support levels 0–5
- **Variable block size:** Not supported by all hardware players (e.g., older Sonos firmware)
- **32-bit float:** Not universally supported in hardware
- **Embedded cue sheets:** Not all players recognize CUESHEET metadata block

### 14.3 Edge Cases to Handle in Converter
- **Empty file / 0-sample stream:** Valid FLAC with STREAMINFO, no frames; decode produces no output
- **File < 1 frame:** Valid; decode what exists
- **All-silence audio:** CONSTANT subframes encode efficiently; single value per block
- **DC offset (non-zero mean):** Fixed order 1 predictor handles DC efficiently
- **Full-scale sine (0 dB):** No clipping; FLAC is lossless
- **File with corrupt frame:** CRC-16 detects; seek past corrupted frame
- **Truncated file:** Stop at last valid frame; report truncated
- **Sample rate 0 (unknown):** Some decoders may fail; inform user

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM FLAC

|| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → WAV | `ffmpeg -i in.flac -c:a pcm_s16le out.wav` | RIFF INFO tags | Lossless |
| → ALAC | `ffmpeg -i in.flac -c:a alac out.m4a` | Via -metadata | Lossless |
| → MP3 | `ffmpeg -i in.flac -c:a libmp3lame -q:a 0 out.mp3` | ID3v2.3 | Generation loss |
| → AAC | `ffmpeg -i in.flac -c:a aac -b:a 256k out.m4a` | Via -metadata | Generation loss |
| → Opus | `ffmpeg -i in.flac -c:a libopus -b:a 128k out.opus` | Vorbis Comments | Generation loss |
| → OGG | `ffmpeg -i in.flac -c:a libvorbis -q:a 6 out.ogg` | Vorbis Comments | Generation loss |
| → WMA Lossless | `ffmpeg -i in.flac -c:a wmalossless out.wma` | Via -metadata | Lossless |

### 15.2 Converting TO FLAC

|| Source | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| WAV → | `ffmpeg -i in.wav -c:a flac -compression_level 8 out.flac` | RIFF INFO → Vorbis | Lossless |
| ALAC → | `ffmpeg -i in.m4a -c:a flac -compression_level 8 out.flac` | Via -metadata | Lossless |
| MP3 → | `ffmpeg -i in.mp3 -c:a flac -compression_level 8 out.flac` | ID3v2 → Vorbis | Lossless (re-encode) |
| AAC → | `ffmpeg -i in.m4a -c:a flac -compression_level 8 out.flac` | Via -metadata | Lossless (re-encode) |
| Vorbis → | `ffmpeg -i in.ogg -c:a flac -compression_level 8 out.flac` | Vorbis Comments | Lossless (re-encode) |

### 15.3 Lossless Round-Trip Verification
```bash
# Encode
ffmpeg -i original.wav -c:a flac -compression_level 8 output.flac

# Decode back
ffmpeg -i output.flac -c:a pcm_s16le decoded.wav

# Compare checksums
ffmpeg -i original.wav -c:a pcm_s16le original_raw.wav
md5sum original_raw.wav decoded_raw.wav   # Must match for true lossless

# Or use FFmpeg's built-in checksumming:
ffmpeg -i original.wav -map 0:a -f framemd5 original.md5
ffmpeg -i output.flac -map 0:a -f framemd5 output.md5
diff original.md5 output.md5   # Empty diff = bit-perfect

# Verify MD5 in STREAMINFO
ffprobe -v quiet -show_entries stream=md5 output.flac
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

|| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| libFLAC | C | BSD | Reference | Reference | https://xiph.org/flac/ |
| libFLAC++ | C++ | BSD | Reference | Reference | Part of libFLAC |
| FFmpeg native | C | LGPL 2.1+ | 9/10 | 10/10 | https://ffmpeg.org |
| flake | C | BSD | 10/10 | — | Part of libFLAC |

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **RFC 9639:** Free Lossless Audio Codec (FLAC) — https://www.rfc-editor.org/rfc/rfc9639
- **FLAC format specification:** https://xiph.org/flac/format.html
- **Xiph.Org FLAC:** https://xiph.org/flac/

### Technical Resources
- FFmpeg encoder options: `ffmpeg -h encoder=flac` or https://ffmpeg.org/ffmpeg-codecs.html
- FFmpeg muxer options: `ffmpeg -h muxer=flac` or https://ffmpeg.org/ffmpeg-formats.html
- Multimedia Wiki: https://wiki.multimedia.cx/index.php/FLAC
- Hydrogenaudio: https://hydrogenaud.io/index.php/board,51.0.html
- Vorbis Comment spec: https://xiph.org/vorbis/doc/v-comment.html

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] FFmpeg libFLAC encoder/decoder is built into default FFmpeg — no external dependency
- [ ] Verify `ffmpeg -encoders` output confirms flac encoder is available
- [ ] Verify `ffmpeg -decoders` output confirms flac decoder is available
- [ ] Verify `ffmpeg -muxers` output confirms flac muxer is available

### Encoding Pipeline
- [ ] Convert input sample format to required `ctx->sample_fmt` using libswresample
- [ ] Handle fixed-frame-size encoders (`ctx->frame_size` = encoder block size)
- [ ] Implement proper flush/drain at end of stream (send NULL frame)
- [ ] Write SEEKTABLE metadata block if seeking is desired
- [ ] Validate that input sample rate is within 1–1048570 Hz
- [ ] Validate that channel layout is supported (1–8 channels)
- [ ] Write VORBIS_COMMENT metadata block with all tags
- [ ] Write PICTURE metadata block for cover art

### Decoding Pipeline
- [ ] Implement FLAC sync word search (0x664C6143 → metadata → frames)
- [ ] Parse STREAMINFO for global stream parameters
- [ ] Parse metadata blocks until last-metadata-block flag
- [ ] Build seek table from SEEKTABLE if present
- [ ] Handle AVERROR(EAGAIN) in receive_packet loop correctly
- [ ] Flush decoder at EOF (send NULL packet)
- [ ] Handle sample format conversion from decoder output format (S16/S32)
- [ ] Verify frame CRC-16 for error detection

### Metadata
- [ ] Read Vorbis Comments from FLAC stream (metadata block type 4)
- [ ] Read PICTURE metadata block for cover art (type 5)
- [ ] Read CUESHEET metadata block for track indices (type 4)
- [ ] Read APPLICATION metadata block (type 1)
- [ ] Write all standard tag fields as Vorbis Comments
- [ ] Embed cover art as PICTURE metadata block
- [ ] Preserve ReplayGain tags through conversion
- [ ] Handle UTF-8 encoding in all tag fields
- [ ] Copy MD5 signature from STREAMINFO

### Quality & Verification
- [ ] Implement ReplayGain scan via EBU R128 (libebur128 integration)
- [ ] Write ReplayGain tags in FLAC Vorbis Comment format
- [ ] Verify bit-exact output for lossless conversions
- [ ] Use STREAMINFO MD5 signature for integrity checking
- [ ] Implement progress reporting from FFmpeg stats output
- [ ] Test with: silence, full-scale, clipped, multi-channel, high-resolution files

### Edge Cases
- [ ] Handle files with corrupt or missing STREAMINFO (required)
- [ ] Handle variable block size FLAC (not all players support it)
- [ ] Handle 32-bit float FLAC (verify output format)
- [ ] Handle files without SEEKTABLE (slow seeking)
- [ ] Handle files with only PADDING blocks
- [ ] Handle very short files (< 1 frame)
- [ ] Handle sample rate = 0 (unknown) in STREAMINFO

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*RFC 9639 formalizes FLAC as IETF standard (December 2024)*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
