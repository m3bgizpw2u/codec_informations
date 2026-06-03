# G.711 (μ-law / A-law PCM) — Deep Technical Reference
> **Category:** Lossy (companding)
> **File Extensions:** `.wav` (when in RIFF/WAV), `.pcm`, `.raw`, `.alaw`, `.mulaw`, `.ulaw`
> **MIME Types:** `audio/g711-alaw`, `audio/g711-ulaw`, `audio/x-wav`
> **Standardization Body:** ITU-T (International Telecommunication Union)
> **Primary Specification:** ITU-T Recommendation G.711 — Pulse Code Modulation (PCM) of Voice Frequencies (November 1988)
> **Patent Status:** Patent-free — no licensing required (ITU-T standard, freely available)
> **License:** ITU copyright but free to implement; no licensing fees
> **Current Version:** G.711 (1988) + Amendments 1-2 (2009), Annex I (1999), Annex II (2000)
> **Active Development:** No — stable, mature standard maintained in force

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** CCITT (now ITU-T), Working Party XV/4
- **Year Created:** 1972 (first standardized), revised 1980, 1984, 1988 (current version)
- **Original Purpose:** Define a standard method for digitizing analog telephone voice signals at 64 kbps, enabling digital transmission over the PSTN (Public Switched Telephone Network) and PCM transmission systems. G.711 replaced pulse code modulation experiments from the 1950s–1960s with a standardized, internationally interoperable companding algorithm.
- **Problem with Predecessors:** Earlier PCM systems used linear encoding (e.g., 8-bit linear PCM at 64 kbps), which provided good quality but wasted bandwidth for the typical voice signal, where low-amplitude samples are far more frequent than high-amplitude samples. Logarithmic companding (μ-law and A-law) achieved roughly the same perceived quality at 64 kbps that linear PCM would require 84+ kbps to match, saving bandwidth and allowing compatibility with existing transmission systems.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| G.711 (original) | 1972 | Initial μ-law specification for TDM telephony |
| G.711 (Rev 4) | 1984 | Clarified A-law definition |
| G.711 (Rev 5) | 1988 | Current base specification; Annex I added later |
| G.711 Amd. 1 | 2009 | Added clarification on bit-exactness |
| G.711 Amd. 2 | 2009 | Implementation guidance |
| G.711 Annex I | 1999 | G.711.1 wideband extension defined |
| G.711 Annex II | 2000 | G.711.0 lossless extension defined |

### 1.3 Current Adoption
- **Primary use cases today:** PSTN voice transport, VoIP gateways, SIP trunking, ISDN (PRI/BRI), T1/E1 digital circuits, legacy PBX systems, TTY/HCO voice relay, fax over IP (FoIP), music on hold, voice prompts in IVR systems
- **Platforms with native support:** All operating systems — Windows, macOS, Linux, iOS, Android, embedded RTOS; hardware in every PSTN switch and cellular base station
- **Major services using this format:** Every telephone call worldwide that goes through any digital portion of the PSTN; VoIP providers that use "HD Voice" at 64 kbps; ISDN and digital PBX systems
- **Hardware support:** Universal — DSP chips in every cell phone, router, gateway, PBX, and PSTN switch worldwide
- **Status:** Dominant and irreplaceable — the foundational codec of all digital telephony. More voice traffic is transmitted using G.711 than any other codec. Even when VoLTE or 5G voice uses AMR or EVS, the radio interface may transport G.711-equivalent bitrates for legacy interworking.

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Waveform codec — companding (logarithmic compression/expansion)
- **Core algorithm:** Logarithmic companding (μ-law / A-law), NOT linear PCM compression
- **Loss mechanism:** Logarithmic quantization — 14-bit (μ-law) or 13-bit (A-law) input mapped to 8-bit output; inverse at decoder
- **Frame-based vs sample-based:** Sample-by-sample companding; no frame structure in the codec itself
- **Fixed vs variable frame size:** Not applicable — G.711 is a per-sample compression algorithm

### 2.2 High-Level Encoding Flow (Block Diagram)
```
Input Analog Voice Signal
      │
      ▼
[Anti-aliasing Low-pass Filter: 300-3400 Hz bandwidth]
      │
      ▼
[Sampling: 8,000 samples/second]
      │
      ▼
[PCM Quantization: 14-bit linear (μ-law) or 13-bit linear (A-law)]
      │
      ▼
[Companding: Logarithmic compression to 8-bit]
      │   μ-law: F(x) = sign(x) × ln(1 + μ|x|) / ln(1 + μ)
      │   A-law: F(x) = sign(x) × A|x| / (1 + ln A)  for |x| < 1/A
      │          F(x) = sign(x) × (1 + ln(A|x|)) / (1 + ln A)  for 1/A ≤ |x| ≤ 1
      ▼
[8-bit PCM Sample: transmitted or stored]
      │
      ▼
Output: 64,000 bits/second (8 bits × 8,000 Hz)
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | 125 μs (0.125 ms) per sample | Lowest possible latency |
| Sample rate | 8,000 Hz | Fixed by ITU-T specification |
| Bit depth | 8-bit compressed | Equivalent to 13–14-bit linear |
| Max channels | 1 (mono) | Multi-channel via channel multiplexing |
| Bandwidth | 300–3,400 Hz | Telephone-quality narrowband voice |
| Bitrate | 64,000 bps (8 bits × 8,000 Hz) | Constant for both μ-law and A-law |
| Complexity | O(1) per sample | Table lookup — trivial to implement |
| MOS Score | ~4.1 | Toll quality for voice |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 Companding Tables

#### μ-law (mu-law, u-law, PCMU, G711u)
- **μ parameter:** 255
- **Input range:** 14-bit signed linear PCM (−8192 to +8191), or ±8158 after the 33 (0x21) bias is added
- **Output:** 8-bit codeword (0x00 to 0xFF)
- **Zero-code:** 0x7F (positive) and 0xFF (negative after bit flip) — note: μ-law spec inverts bits before transmission, so quiet μ-law = 0xFF transmitted

**μ-law encoding formula (per ITU-T G.711):**
```
For input sample x (14-bit signed, two's complement):
  1. Add bias 33 (0x21) to x if x < 0, then take absolute value
  2. Clamp to max 8158
  3. Find segment number (0-7) by finding where |x| falls in 16 quantization steps
  4. Find step number within segment (0-15)
  5. Output: sign bit ⊕ segment (3 bits) ⊕ step (4 bits)
  6. Invert all bits before transmission (bit flip)

μ-law quantization step sizes per segment:
  Segment 0: step = 2 (linear quantization, 16 levels)
  Segment 1: step = 4
  Segment 2: step = 8
  Segment 3: step = 16
  Segment 4: step = 32
  Segment 5: step = 64
  Segment 6: step = 128
  Segment 7: step = 256
```

#### A-law (PCMA, G711a)
- **A parameter:** 87.56
- **Input range:** 13-bit signed linear PCM (−4096 to +4095)
- **Output:** 8-bit codeword (0x00 to 0xFF)
- **Zero-code:** 0xD5 (0b11010101) — A-law has a symmetric zero code

**A-law encoding formula (per ITU-T G.711):**
```
For input sample x (13-bit signed):
  1. Take absolute value of x
  2. Normalize to range [0, 1]: x_norm = |x| / 4096
  3. For small signals (x_norm < 1/A ≈ 0.01116):
       output = A × x_norm (linear quantization)
  4. For larger signals (x_norm ≥ 1/A):
       output = (1 + ln(A × x_norm)) / (1 + ln A) (logarithmic quantization)
  5. Apply even inversion: even-numbered bits (0,2,4,6) are inverted for A-law
  6. Sign bit indicates original polarity

A-law quantization:
  8 segments (each with 16 linear steps)
  Segment 0: step = 1 (linear region, 16 levels)
  Segment 1: step = 2
  Segment 2: step = 4
  Segment 3: step = 8
  Segment 4: step = 16
  Segment 5: step = 32
  Segment 6: step = 64
  Segment 7: step = 128
```

### 3.2 G.711 Encoding Table (μ-law, partial excerpt)
```
Input range (14-bit signed) → μ-law output:
  0 to 1           → 0x84
  2 to 3           → 0x83
  4 to 7           → 0x82
  8 to 15          → 0x81
  16 to 31         → 0x80
  32 to 63         → 0xFF (segment 7, step 15) ... down to 0x7F (segment 0, step 0)
  Negative values:  similar mapping, then bitwise NOT of positive code

A-law input range (13-bit signed) → A-law output:
  0                 → 0xAA
  ±1                → 0xD5
  ±2                → 0xD4
  ±4                → 0xD3
  ±8                → 0xD2
  ±16               → 0xD1
  ±32               → 0xD0
  ... (segment encoding similar to μ-law)
```

### 3.3 WAV File Format for G.711
When G.711 is stored in a WAV file, the RIFF/WAVE fmt chunk uses specific format tags:

```
Offset   Size    Field Name              Type        Value      Description
-------  ------  ----------------------  ----------  ---------  ------------------------
0x0000   4B      RIFF magic             char[4]     "RIFF"     File identifier
0x0004   4B      File size − 8         uint32 LE   N          File size
0x0008   4B      WAVE magic            char[4]     "WAVE"     Format identifier

--- fmt sub-chunk ---
0x000C   4B      Subchunk1ID           char[4]     "fmt "     Format chunk identifier
0x0010   4B      Subchunk1Size         uint32 LE   16         Size of fmt data (16 for PCM)
0x0014   2B      AudioFormat           uint16 LE   1 or 6 or 7  1=PCM, 6=A-law, 7=μ-law
0x0016   2B      NumChannels           uint16 LE   1          Number of channels (1 mono)
0x0018   4B      SampleRate            uint32 LE   8000       Samples per second
0x001C   4B      ByteRate              uint32 LE   8000       Bytes per second (8k×1ch)
0x0020   2B      BlockAlign            uint16 LE   1          Bytes per sample (1 byte)
0x0022   2B      BitsPerSample         uint16 LE   8          Bits per sample (8-bit)
0x0024   4B      Subchunk2ID           char[4]     "data"     Data chunk identifier
0x0028   4B      Subchunk2Size         uint32 LE   N          Size of audio data
0x002C   NB      Audio Data            uint8[]     —          G.711 samples
```

**WAV format tags for G.711:**
| Audio Format | Codec | wFormatTag value (hex) |
|-------------|-------|------------------------|
| PCM (linear) | 16-bit signed linear PCM | 0x0001 |
| A-law | ITU-T G.711 A-law | 0x0006 |
| μ-law | ITU-T G.711 μ-law | 0x0007 |

### 3.4 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | μ-law PCM | Yes | Native G.711 μ-law format |
| 8-bit | A-law PCM | Yes | Native G.711 A-law format |
| 8-bit | Unsigned linear | No | Not used with G.711 |
| 13-bit | Signed linear (A-law input) | Yes | A-law internally expands to 13-bit |
| 14-bit | Signed linear (μ-law input) | Yes | μ-law internally expands to 14-bit |
| 16-bit | Signed linear | Yes | Common for WAV files (needs conversion) |
| 24-bit | Signed integer | Yes | Accepted, truncated to 13/14-bit internally |
| 32-bit | IEEE float | No | Not directly supported |

#### Supported Sample Rates
| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 8000 | Telephone | Yes | Fixed by G.711 standard |
| 16000 | Wideband | No | Use G.722 or G.711.1 |
| 44100 | CD audio | No | Not supported |
| 48000 | Professional | No | Not supported |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Companding Principles

#### Why Companding Works for Voice
Human hearing follows a logarithmic sensitivity curve — a doubling of sound intensity does not double perceived loudness. The logarithmic μ-law and A-law companding algorithms exploit this by:
1. **Quantizing small signals finely:** Low-amplitude samples (most voice signals) get more of the 256 available 8-bit codes
2. **Quantizing large signals coarsely:** High-amplitude samples get fewer codes but are perceptually less sensitive to quantization noise
3. **Result:** 8-bit logarithmic PCM at 64 kbps achieves subjective quality comparable to 12–13-bit linear PCM

### 4.2 μ-law Encoding (Detailed)
```
Input: 14-bit signed two's complement integer x ∈ [−8192, +8191]

Step 1 — Bias and absolute value:
  if x < 0:
    y = (−x) + 33   // Add 33 bias for negative values (asymmetric around 0)
  else:
    y = x
  // y now in range [0, 8158] (after clamping)

Step 2 — Segment determination (8 segments of 16 steps each):
  if y < 32:        segment = 0
  elif y < 64:      segment = 1,  step = (y−32)/2
  elif y < 128:     segment = 2,  step = (y−64)/4
  elif y < 256:     segment = 3,  step = (y−128)/8
  elif y < 512:     segment = 4,  step = (y−256)/16
  elif y < 1024:    segment = 5,  step = (y−512)/32
  elif y < 2048:    segment = 6,  step = (y−1024)/64
  else:             segment = 7,  step = (y−2048)/128

Step 3 — Construct 8-bit codeword:
  sign_bit  = 0 if x ≥ 0 else 1
  seg_bits  = segment (3 bits)
  step_bits = floor(step) (4 bits)
  raw_code  = (sign_bit << 7) | (seg_bits << 4) | step_bits

Step 4 — Bit inversion (required by G.711 spec):
  transmitted = ~raw_code  // invert all 8 bits
```

### 4.3 A-law Encoding (Detailed)
```
Input: 13-bit signed two's complement integer x ∈ [−4096, +4095]

Step 1 — Absolute value and normalization:
  y = |x|   // 13-bit range [0, 4095]

Step 2 — Linear vs. logarithmic region:
  if y < 16:           // Within the A-law linear region (segment 0)
    segment = 0
    step = y / 1
  elif y < 32:         segment = 1,  step = (y−16)/1
  elif y < 64:         segment = 2,  step = (y−32)/2
  elif y < 128:        segment = 3,  step = (y−64)/4
  elif y < 256:        segment = 4,  step = (y−128)/8
  elif y < 512:        segment = 5,  step = (y−256)/16
  elif y < 1024:       segment = 6,  step = (y−512)/32
  elif y < 2048:       segment = 7,  step = (y−1024)/64
  else:                segment = 7,  step = 15

Step 3 — Construct raw codeword:
  sign_bit  = 0 if x ≥ 0 else 1
  seg_bits  = segment (3 bits, but segments 0–7 map to bit patterns 1,3,2,6,4,5,7,7)
  step_bits = step (4 bits)
  raw_code  = (sign_bit << 7) | (seg_bits << 4) | step_bits

Step 4 — Even-bit inversion:
  // Invert even-numbered bits (0, 2, 4, 6) per G.711 A-law spec
  transmitted = raw_code XOR 0x55  // 0x55 = 0b01010101
```

### 4.4 G.711 Decoding (Inverse Companding)
```
μ-law decoding:
  1. Invert all 8 received bits
  2. sign = (code >> 7) & 1
  3. segment = (code >> 4) & 0x07
  4. step = code & 0x0F
  5. y = (segment << 4) | step  // De-linearized value
  6. if y == 0:
       x = 0
     else:
       y = (y + 0.5) × (2 ^ segment)  // Find actual magnitude
       x = sign ? −(y − 33) : y

A-law decoding:
  1. Invert even-numbered bits: raw = received XOR 0x55
  2. sign = (raw >> 7) & 1
  3. seg_pat = (raw >> 4) & 0x07  // 3-bit segment pattern (1,3,2,6,4,5,7,7)
  4. Decode seg_pat → actual segment number (0–7)
  5. step = raw & 0x0F
  6. y = (step + 0.5) × (2 ^ (segment − 1))   // De-linearized
  7. x = sign ? −y : y
```

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking
G.711 is a continuous bitstream with no frame headers, sync words, or index structures. Seeking in G.711 data is purely time-based:

```
Seeking in G.711:
  - Byte offset = target_time_ms × 8 bytes/ms
  - E.g., seeking to 5 seconds: byte_offset = 5 × 1000 × 8 = 40000 bytes
  - G.711 has no frame structure — every byte is independently decodable
  - Random access is trivial: seek to byte offset, read bytes, decode each independently

WAV container seeking:
  - Use the 'data' chunk offset + byte offset within data
  - sample_number = byte_offset (since 1 byte per sample at 8 kHz)
  - timestamp_seconds = byte_offset / 8000
```

### 5.2 Core Decode Pipeline
```
G.711 decode is O(1) per sample — pure table lookup:

1. Read 8-bit byte from bitstream (1 byte = 1 sample)
   └── Parallel input: whole buffer can be processed at once

2. For each byte b:
   a. Apply inverse transformation:
      μ-law: b = ~b  (invert all 8 bits)
      A-law: b = b XOR 0x55  (invert even bits)

   b. Extract fields:
      sign       = (b >> 7) & 1
      segment    = (b >> 4) & 0x07  (μ-law)
      segment    = decode_segment_pattern((b >> 4) & 0x07)  // A-law
      quantization = b & 0x0F

   c. Reconstruct linear magnitude:
      μ-law:  mag = (quantization << segment) + (1 << (segment - 1)) - 33
      A-law:  mag = (quantization << (segment - 1)) + (1 << (segment - 2)) - 1

   d. Apply sign:
      if sign == 1:  sample = -mag
      else:          sample = +mag

3. Output 16-bit signed linear PCM
   └── Can output directly as 16-bit PCM in WAV or other container

4. Apply optional post-processing:
   └── Low-pass filter (300 Hz cutoff for TTY/HCO)
   └── High-pass filter (300 Hz cutoff for DC removal)
```

### 5.3 Error Concealment
- **Corrupt byte detection:** G.711 has no error detection — every 8-bit codeword is valid
- **Concealment method:** Packet-level concealment is handled by the transport layer (RTP seq/pts) or container; individual sample errors cannot be detected
- **Corrupted audio:** A single bit error in a G.711 byte produces a single audio sample error (a click or pop), as each sample is independent

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** None — G.711 is a raw bitstream format. The "container" is typically: (a) raw PCM (`.raw`, `.pcm`), (b) WAV (`.wav`), (c) AU (`.au`), or (d) RTP payload
- **Overhead:** 44 bytes WAV header for file; none for raw bitstream
- **Seeking in WAV:** Yes — by byte offset within the data chunk
- **Multiple streams:** G.711 can be multiplexed as multiple channels via time-division multiplexing

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| WAV (.wav) | Yes (fmt=6 or 7) | Yes (byte offset) | RIFF INFO tags | Standard for G.711 files |
| AU (.au) | Yes (encoding=alaw/mulaw) | No | Limited | Sun/NeXT format |
| Raw (.raw, .pcm) | Yes | Byte offset | None | Raw bitstream |
| MP4/M4A | No | — | — | Not stored in MP4 |
| Matroska/MKA | No | — | — | Not natively stored |
| OGG | No | — | — | Not natively stored |
| RTP | Yes | No | No | RFC 3551 payload type 8 (A-law) and 0 (μ-law) |
| WebM | No | — | — | Not natively stored |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** RIFF INFO (in WAV) or no metadata (raw bitstream)
- **Tag block location:** INFO chunk within WAV file, after the data chunk
- **Tag block identifier:** 'LIST' chunk with 'INFO' sub-type

### 7.2 Standard Tag Fields — RIFF INFO
| Tag Field | RIFF INFO Key | Max Length | Character Encoding | Multi-value | Notes |
|-----------|---------------|------------|-------------------|-------------|-------|
| Title | INAM | 256 | ASCII | No | Title |
| Artist | IART | 256 | ASCII | No | Creator/artist |
| Comment | ICMT | 256 | ASCII | No | Comment |
| Copyright | ICOP | 256 | ASCII | No | Copyright |
| Creation Date | ICRD | 256 | ASCII | No | YYYY-MM-DD |
| Genre | IGNR | 256 | ASCII | No | Genre |
| Software | ISFT | 256 | ASCII | No | Encoder name |
| Engineer | IENG | 256 | ASCII | No | Engineer name |

### 7.3 Cover Art Storage
Cover art is not natively supported in WAV G.711 files. To store cover art with G.711 audio:
- Use a different container (e.g., MP4, Matroska)
- Or embed a `LIST` chunk with a `DISC` or `icon` subchunk (non-standard)

### 7.4 Metadata Compatibility Matrix
| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| RIFF INFO | ✓ | ✓ | ✓ | Highest (WAV native) |
| ID3v1 | ✗ | ✗ | ✗ | Not applicable |
| ID3v2 | ✗ | ✗ | ✗ | Not applicable |
| Vorbis Comments | ✗ | ✗ | ✗ | Not applicable |

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   pcm_alaw             # A-law decoder/encoder
                    pcm_mulaw             # μ-law decoder/encoder (alias: pcm_s8)
                    pcm_alaw (for both)  # via FFmpeg native codec
AV_CODEC_ID:        AV_CODEC_ID_PCM_ALAW  # C constant (A-law)
                    AV_CODEC_ID_PCM_MULAW # C constant (μ-law)
Format Name (CLI):  alaw, mulaw            # raw bitstream formats
                    wav                    # WAV container
                    au                     # Sun AU format
Encoder(s):         pcm_alaw, pcm_mulaw   # Native FFmpeg PCM codecs
Decoder(s):         pcm_alaw, pcm_mulaw   # Native FFmpeg PCM codecs
Muxer(s):          wav, alaw, mulaw, au  # Multiple muxers
Demuxer(s):        wav, alaw, mulaw, au   # Multiple demuxers
```

**Note:** G.711 is implemented natively in FFmpeg without any external library dependency. The companding tables are built into FFmpeg's `libavcodec/pcm.c`.

### 8.2 FFmpeg Encoding — Full CLI Reference

```bash
# Encode WAV to μ-law RAW (.ulaw / .raw)
ffmpeg -i input.wav \
  -c:a pcm_mulaw \
  -ar 8000 \
  -ac 1 \
  -f mulaw \
  output.ulaw

# Encode WAV to A-law RAW (.alaw)
ffmpeg -i input.wav \
  -c:a pcm_alaw \
  -ar 8000 \
  -ac 1 \
  -f alaw \
  output.alaw

# Encode WAV to μ-law in WAV container
ffmpeg -i input.wav \
  -c:a pcm_mulaw \
  -ar 8000 \
  -ac 1 \
  output_mulaw.wav

# Encode WAV to A-law in WAV container
ffmpeg -i input.wav \
  -c:a pcm_alaw \
  -ar 8000 \
  -ac 1 \
  output_alaw.wav

# Encode to SUN AU format (A-law default)
ffmpeg -i input.wav \
  -c:a pcm_alaw \
  -ar 8000 \
  -ac 1 \
  output.au
```

#### Complete FFmpeg Option Table
| Option | Type | Default | Range / Valid Values | Effect on Output |
|--------|------|---------|---------------------|-----------------|
| `-ar` | int | 8000 | 8000 | Sample rate (must be 8000 for G.711) |
| `-ac` | int | input | 1 | Channel count (mono only) |
| `-f` | string | auto | alaw, mulaw, wav | Output format/container |

### 8.3 FFmpeg Encoding — libavcodec C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/opt.h>
#include <libswresample/swresample.h>

// ─── 1. Find encoder ─────────────────────────────────────────────────────────
const AVCodec *codec = avcodec_find_encoder(AV_CODEC_ID_PCM_MULAW);
// Alternative: AV_CODEC_ID_PCM_ALAW for A-law
if (!codec) { fprintf(stderr, "Encoder not found\n"); exit(1); }

// ─── 2. Allocate codec context ────────────────────────────────────────────────
AVCodecContext *ctx = avcodec_alloc_context3(codec);
if (!ctx) { fprintf(stderr, "OOM\n"); exit(1); }

// ─── 3. Set required parameters ────────────────────────────────────────────────
ctx->sample_fmt  = AV_SAMPLE_FMT_S16; // Input: 16-bit signed PCM
ctx->sample_rate = 8000;               // G.711 requires 8000 Hz
av_channel_layout_default(&ctx->ch_layout, 1); // Mono

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
frame->nb_samples  = ctx->frame_size; // PCM codecs: frame_size = 1 (sample-by-sample)
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
- G.711 is a per-sample codec — `ctx->frame_size` = 1 (or 0 in some versions)
- G.711 encoder accepts `AV_SAMPLE_FMT_S16` input and produces `AV_SAMPLE_FMT_U8` output (8-bit unsigned sample)
- FFmpeg uses unsigned 8-bit for G.711 samples (0x00–0xFF), NOT signed
- Sample rate MUST be exactly 8000 Hz
- This is one of the simplest codecs in FFmpeg — no external library needed

### 8.4 FFmpeg Decoding — Full CLI Reference

```bash
# Decode μ-law RAW to WAV PCM (16-bit signed linear)
ffmpeg -f mulaw -i input.ulaw \
  -c:a pcm_s16le \
  -ar 8000 \
  -ac 1 \
  output.wav

# Decode A-law RAW to WAV PCM
ffmpeg -f alaw -i input.alaw \
  -c:a pcm_s16le \
  -ar 8000 \
  -ac 1 \
  output.wav

# Decode G.711 WAV file (detect format from WAV header)
ffmpeg -i input_mulaw.wav \
  -c:a pcm_s16le \
  output.wav

# Decode G.711 WAV to raw PCM
ffmpeg -i input.wav \
  -f s16le \
  output.raw

# Probe format without decoding
ffprobe -v quiet -print_format json -show_streams -show_format input.wav

# Extract μ-law from WAV to raw
ffmpeg -i input.wav -c:a copy -f mulaw output.raw
```

### 8.5 FFmpeg Decoding — libavcodec C API

```c
// ─── Complete decode program skeleton ────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.wav", NULL, NULL);
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
            // frm->data[0] = PCM samples
            // For G.711: frm->format = AV_SAMPLE_FMT_U8 (8-bit unsigned)
            // After decoding to PCM: AV_SAMPLE_FMT_S16
            // frm->nb_samples = number of samples
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
# Read all metadata as JSON
ffprobe -v quiet -print_format json -show_format input.wav | jq .format.tags

# Write metadata (RIFF INFO in WAV)
ffmpeg -i input.wav \
  -c:a copy \
  -metadata title="Voice Recording" \
  -metadata artist="Speaker Name" \
  -metadata date="2024" \
  -metadata comment="IVR prompt" \
  output_tagged.wav

# Strip all metadata
ffmpeg -i input.wav -c:a copy -map_metadata -1 output_clean.wav

# Embed cover art (not standard in WAV G.711)
# Note: WAV files don't natively support cover art via RIFF INFO
# Use a different container (MP4, Matroska) for G.711 + cover art
```

#### FFmpeg Internal Metadata Key Mapping
| Standard Field | FFmpeg Key | RIFF INFO Key | Notes |
|----------------|------------|----------------|-------|
| Title | title | INAM | |
| Artist | artist | IART | |
| Comment | comment | ICMT | |
| Copyright | copyright | ICOP | |
| Creation Date | date | ICRD | |
| Genre | genre | IGNR | |
| Software | encoder | ISFT | Auto-set by FFmpeg |

### 8.7 Quality / Fidelity Decision Table
| Use Case | Settings | Expected Size | Notes |
|----------|----------|---------------|-------|
| Standard telephony | `-c:a pcm_mulaw -ar 8000` | 480 KB/min | μ-law (North America/Japan) |
| Standard telephony | `-c:a pcm_alaw -ar 8000` | 480 KB/min | A-law (Europe) |
| High quality | G.711.1 (wideband) | — | Not G.711 |
| TTY/HCO voice relay | `-c:a pcm_mulaw -ar 8000` | 480 KB/min | Must preserve 2245 Hz tone |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
```
G.711 bitstream: No seek table needed.

Random access is trivial:
  - Every byte = 1 sample = 0.125 ms of audio
  - Byte offset for time T: byte_offset = floor(T_seconds × 8000)
  - Time for byte offset N: time_seconds = N / 8000

WAV container seek:
  - RIFF fmt chunk specifies: wFormatTag, nChannels, nSamplesPerSec, nAvgBytesPerSec
  - data chunk offset: byte 44 (standard WAV header)
  - Seek within data: sample_number = byte_offset_in_data
  - Timestamp: timestamp_seconds = sample_number / 8000
```

### 9.2 Gapless Playback Data
```
Encoder delay:   0 samples (sample-by-sample, no lookahead)
Padding:         0 samples (no encoder padding)
Storage:         Not applicable — no delay/padding metadata

G.711 is inherently gapless — no encoder or decoder delay.
The entire "delay" is the 125 μs sampling period.
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | Trivial — decode byte-by-byte |
| Algorithmic encoder delay | 125 μs | One sample period |
| Live encoding feasible | Yes | Trivial — one table lookup per sample |
| HTTP progressive download | Yes | Raw bitstream or WAV container |
| HTTP Live Streaming (HLS) | Yes | G.711 can be used as audio track in MPEG-TS |
| DASH streaming | Yes | G.711 as MPEG-4 audio in DASH manifest |
| WebRTC / RTP transport | Yes | RFC 3551: PT 0 = μ-law, PT 8 = A-law |
| Minimum decode buffer | 1 byte | Trivial decode |

**RTP Payload Type Assignment (RFC 3551):**
```
Payload Type 0:   PCMU (G.711 μ-law, 8 kHz)
Payload Type 8:   PCMA (G.711 A-law, 8 kHz)
Sample Rate:      8000 Hz
Timestamp Rate:   8000 Hz
Bitrate:          64,000 bps
Frame Size:       Typically 160 bytes (20 ms of audio = 160 samples × 1 byte)
```

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | Layout Name | Channel Order | FFmpeg Layout Constant | Notes |
|----------|-------------|---------------|----------------------|-------|
| 1 | Mono | C | AV_CHANNEL_LAYOUT_MONO | Standard |
| 2 | Stereo | L, R | AV_CHANNEL_LAYOUT_STEREO | Not standard but possible via WAV |

**Note:** True multi-channel G.711 is not ITU-T standard. Stereo WAV files with G.711 encoding would simply interleave two mono G.711 streams (left, right, left, right...), which is non-standard. Standard practice is one mono G.711 stream per channel, multiplexed via TDM.

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 13–14 bit effective | After expansion to linear PCM |
| Max sample rate | 8000 Hz | Fixed by standard |
| Float support | No | Fixed-point only |
| DSD support | No | Not applicable |
| 20-bit support | No | Not in specification |
| 24-bit support | No | Not in specification |

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

| Platform / API | Encode | Decode | FFmpeg Flag | Notes |
|----------------|--------|--------|-------------|-------|
| All hardware | Yes | Yes | None | Universal hardware support |
| x86/x64 | Yes | Yes | None | Table lookup in CPU cache |
| ARM | Yes | Yes | None | NEON can accelerate bulk conversion |
| Intel QSV | No | No | — | Not needed — trivial CPU decode |
| Apple AudioToolbox | Yes | Yes | — | System codec on macOS/iOS |
| Android MediaCodec | No | Yes | MediaCodec | Decoding via platform |
| VA-API | No | No | — | Not needed |

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Encoder-Specific Issues
| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| G.711 encoder not found | None | Always available in FFmpeg (built-in) |
| Wrong bit depth output | None | G.711 outputs 8-bit; use `-c:a pcm_s16le` for 16-bit WAV |
| Sample rate mismatch | All | FFmpeg resamples automatically to 8000 Hz |

### 14.2 Interoperability Issues
- **μ-law vs A-law mismatch:** This is the most common G.711 error. If μ-law is decoded as A-law (or vice versa), the audio sounds like loud noise with severe distortion. The receiving system must know which law is in use.
  - μ-law: North America, Japan, most of Asia Pacific
  - A-law: Europe, most of Asia, South America, Africa, Australia
- **Bit-inversion differences:** Some implementations don't apply the μ-law bit-flip before transmission, which can cause interop issues. ITU spec requires the flip.
- **14-bit vs 13-bit input precision:** μ-law expands to 14-bit linear; A-law expands to 13-bit. Mismatched input precision can cause slight quality differences.
- **Non-standard WAV headers:** Some systems write wFormatTag = 1 (PCM) for G.711 data, which is incorrect. FFmpeg's WAV demuxer detects G.711 by header inspection.

### 14.3 Edge Cases to Handle in Converter
- **Empty file / 0-sample stream:** Produce empty WAV file with correct header
- **File < 1 byte:** Encode single byte; WAV header reports 1 sample
- **All-silence audio:** G.711 silence = 0x7F (μ-law) or 0xD5 (A-law)
- **DC offset (non-zero mean):** Encode as-is; DC will appear as near-silence tone
- **Full-scale sine (0 dB):** Encode as full-scale G.711 values; no clipping in log domain
- **Sample rate not 8 kHz:** FFmpeg resamples to 8000 Hz automatically
- **Stereo input:** FFmpeg auto-downmixes to mono

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM G.711

| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → FLAC | `ffmpeg -i in.wav -c:a flac -compression_level 8 out.flac` | RIFF INFO → FLAC | Lossless (after decode) |
| → WAV (16-bit) | `ffmpeg -i in.wav -c:a pcm_s16le out.wav` | RIFF INFO | Lossless decode |
| → MP3 | `ffmpeg -i in.wav -c:a libmp3lame -q:a 2 out.mp3` | ID3v2 | Generation loss |
| → AAC | `ffmpeg -i in.wav -c:a aac -b:a 64k out.m4a` | Partial | Generation loss |
| → Opus | `ffmpeg -i in.wav -c:a libopus -b:a 24k out.opus` | Vorbis Comments | Generation loss |
| → ALAC | `ffmpeg -i in.wav -c:a alac out.m4a` | Partial | Lossless (after decode) |

### 15.2 Converting TO G.711

| Source | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| WAV (16-bit, 8kHz) → μ-law | `ffmpeg -i in.wav -c:a pcm_mulaw -ar 8000 out.ulaw` | None | Lossy |
| WAV (16-bit, any rate) → μ-law | `ffmpeg -i in.wav -c:a pcm_mulaw -ar 8000 -ac 1 out.wav` | RIFF INFO | Lossy |
| WAV (16-bit, 8kHz) → A-law | `ffmpeg -i in.wav -c:a pcm_alaw -ar 8000 out.alaw` | None | Lossy |
| FLAC → μ-law | `ffmpeg -i in.flac -c:a pcm_mulaw -ar 8000 out.wav` | Partial | Transcode lossy |
| MP3 → A-law | `ffmpeg -i in.mp3 -c:a pcm_alaw -ar 8000 out.wav` | Partial | Generation loss |

### 15.3 Lossless Round-Trip Verification
```bash
# G.711 is lossy — true lossless round-trip is NOT possible
# Verify decode pipeline:

# Decode μ-law to 16-bit linear PCM
ffmpeg -f mulaw -i original.ulaw -c:a pcm_s16le decoded.wav

# Decode from source linear WAV
ffmpeg -i source_linear.wav -c:a pcm_s16le source_raw.wav

# Compare (should differ due to G.711 lossy encoding)
md5sum decoded.wav source_raw.wav
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| ITU-T G.191 (reference) | C | ITU copyright | Reference | Reference | https://www.itu.int/rec/T-REC-G.191 |
| FFmpeg native | C | LGPL 2.1+ | Excellent | Excellent | https://ffmpeg.org |
| Sun Microsystems (AU) | C | Public domain | Reference | Reference | Historical reference |
| Speex/G.711 utility | C | BSD | Good | Good | Part of libspeex |

### Build Instructions (for bundling in converter app)
```bash
# G.711 requires NO external library — built into FFmpeg
# FFmpeg must be compiled with default configuration (PCM codecs are always enabled)

# To verify G.711 support:
ffmpeg -encoders | grep -E "pcm_alaw|pcm_mulaw"
ffmpeg -decoders | grep -E "pcm_alaw|pcm_mulaw"

# No external dependencies needed for G.711
# Build FFmpeg with default configuration:
./configure --prefix=/usr/local
make -j$(nproc)
make install
```

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **ITU-T G.711:** Pulse Code Modulation (PCM) of Voice Frequencies — https://www.itu.int/rec/T-REC-G.711
- **ITU-T G.191:** Software Tools Library — G.711 reference C implementation — https://www.itu.int/rec/T-REC-G.191
- **RFC 3551:** RTP Profile for Audio and Video Conferences with Minimal Control — Payload Type assignments
- **RFC 3550:** RTP: A Transport Protocol for Real-Time Applications — RTP transport for G.711

### Technical Resources
- FFmpeg encoder options: `ffmpeg -h encoder=pcm_mulaw` and `ffmpeg -h encoder=pcm_alaw`
- ITU-T G.711 page: https://www.itu.int/rec/T-REC-G.711
- ITU-T G.191 (reference C code): available from ITU
- RFC 3551: https://www.rfc-editor.org/rfc/rfc3551
- Wikipedia G.711: comprehensive article with encoding tables

### Academic Papers
- ITU-T, "Recommendation G.711 — Pulse Code Modulation (PCM) of Voice Frequencies," 1988
- ITU-T G.711 Annex I (G.711.1): Wideband extension to G.711
- ITU-T G.711 Annex II (G.711.0): Lossless compression of G.711 bitstreams
- "A Law and Mu Law Companding: A Tutorial," comprehensive technical explanation

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] G.711 requires NO external library — built into FFmpeg by default
- [ ] Verify `ffmpeg -encoders` output confirms pcm_alaw and pcm_mulaw are available
- [ ] Verify `ffmpeg -decoders` output confirms pcm_alaw and pcm_mulaw are available
- [ ] No external dependencies for G.711 — all implementations are native

### Encoding Pipeline
- [ ] Convert input to S16 using libswresample (S16 input required)
- [ ] Resample to exactly 8000 Hz (G.711 requires 8000 Hz)
- [ ] G.711 is per-sample — `ctx->frame_size` = 1 (or auto)
- [ ] Output is U8 (unsigned 8-bit per sample) — NOT S16
- [ ] Use `-f mulaw` or `-f alaw` for raw output, or WAV muxer for container
- [ ] No flush/drain needed (no state between samples)

### Decoding Pipeline
- [ ] Use WAV demuxer for WAV files; raw format for .raw/.ulaw/.alaw
- [ ] FFmpeg's G.711 decoder outputs S16 (16-bit signed PCM) by default
- [ ] Resample output if a different sample rate is needed
- [ ] No AVERROR(EAGAIN) handling needed for raw PCM (no frames)

### Metadata
- [ ] Read RIFF INFO tags from WAV files (INAM, IART, ICMT, ICOP, ICRD, IGNR, ISFT)
- [ ] Write RIFF INFO tags to WAV output
- [ ] WAV files do NOT support cover art natively
- [ ] Raw G.711 bitstreams have no metadata

### Quality & Verification
- [ ] G.711 is lossy — always verify that quality expectations are set correctly
- [ ] Test encode/decode at both μ-law and A-law
- [ ] Verify μ-law ↔ A-law cross-decoding produces audible distortion
- [ ] Test with: voice, DTMF tones, TTY/HCO (2245 Hz tone), silence

### Edge Cases
- [ ] Handle empty files (0 bytes) — create empty WAV with correct header
- [ ] Handle non-8kHz input — resample to 8000 Hz
- [ ] Handle stereo input — downmix to mono
- [ ] Handle non-standard WAV wFormatTag values

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
