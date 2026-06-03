# AMR (Adaptive Multi-Rate) — Deep Technical Reference
> **Category:** Lossy
> **File Extensions:** `.amr`, `.3gp`, `.awb` (AMR-WB)
> **MIME Types:** `audio/amr`, `audio/3gpp`, `audio/3gpp2`
> **Standardization Body:** 3GPP (3rd Generation Partnership Project)
> **Primary Specification:** 3GPP TS 26.071 (General Description); 3GPP TS 26.090 (Transcoding Functions); RFC 4867 (RTP Payload Format)
> **Patent Status:** Patented — licensed via Via Licensing Corporation
> **License:** Royalty-bearing for commercial use; 3GPP-essential patents
> **Current Version:** 3GPP TS 26.071 V19.0.0 (3GPP Release 19, 2025)
> **Active Development:** Yes — maintained as part of 3GPP specification suite for VoLTE/5G voice

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** ETSI (European Telecommunications Standards Institute), 3GPP
- **Year Created:** 1998–1999 (standardized in GSM Phase 2+ / 3GPP Release 99)
- **Original Purpose:** Replace GSM EFR (Enhanced Full Rate) and provide an adaptive multi-rate codec that could switch between 8 bitrates on a frame-by-frame basis, optimizing voice quality versus radio link conditions. The key innovation was **link adaptation** — the codec bitrate would automatically adjust based on channel quality.
- **Problem with Predecessors:** GSM FR (13 kbps, RPE-LTP) and GSM EFR (12.2 kbps, ACELP) had fixed bitrates. In poor radio conditions, the same number of bits was transmitted regardless of whether they would all be received correctly, wasting bandwidth without improving quality. AMR solved this by allowing the codec to operate at lower bitrates when channel conditions were poor, freeing bits for stronger channel coding (forward error correction), thereby improving robustness.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| 3GPP Release 99 | 1999 | Initial AMR-NB specification, 8 modes |
| 3GPP Release 4 | 2001 | Clarifications, RTP payload format (RFC 3267) |
| 3GPP Release 5 | 2002 | SID (Silence Insertion Descriptor) frame refinements |
| 3GPP Release 6 | 2004 | AMR-WB (wideband) added as separate specification |
| 3GPP Release 7–19 | 2005–2025 | Maintenance updates, 5G voice continuity |

### 1.3 Current Adoption
- **Primary use cases today:** VoLTE (Voice over LTE) in 4G networks, VoNR (Voice over New Radio) in 5G, legacy 3G GSM/UMTS voice, satellite voice terminals, some military communications
- **Platforms with native support:** All 3GPP-compliant mobile devices (iPhone, Android phones), base station infrastructure, VoLTE gateways
- **Major services using this format:** All major mobile carriers worldwide (through 3G/4G/5G), military tactical communications
- **Hardware support:** Integrated into baseband processors and cellular modems on virtually all mobile phones; DSP-accelerated in hardware
- **Status:** Dominant for cellular voice — used by billions of devices globally. AMR-NB remains the fallback for voice in 4G/5G when EVS (Enhanced Voice Services) is not available.

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Parametric / CELP hybrid speech coder
- **Core algorithm:** Algebraic Code-Excited Linear Prediction (ACELP)
- **Loss mechanism:** Perceptual quantization of ACELP parameters; mode switching reduces quality to maintain robustness
- **Frame-based vs sample-based:** Frame-based encoding; fixed 20 ms frames
- **Fixed vs variable frame size:** Fixed frame size; variable bitrate (mode switching)

### 2.2 High-Level Encoding Flow (Block Diagram)
```
Input PCM (8 kHz, 13-bit uniform linear PCM or A-law/μ-law)
      │
      ▼
[Pre-processing: DC removal, A/μ-law to linear conversion]
      │
      ▼
[Analysis: 20 ms frame = 160 samples at 8 kHz]
      │
      ▼
[LPC Analysis: 10th order Linear Prediction, Levinson-Durbin recursion]
      │           (performed once per frame for modes 4.75–7.95 kbps)
      │           (performed twice per frame for mode 12.2 kbps)
      ▼
[Open-loop pitch estimate]
      │
      ▼
[Closed-loop pitch search (per subframe)]
      │           5 subframes of 40 samples each
      ▼
[Algebraic Codebook search]
      │           Per subframe: 4 subframes × 40 samples
      ▼
[Gain quantization]
      │
      ▼
[Parameter bit allocation packing]
      │           Bit count depends on selected mode
      ▼
[Output AMR frame: 95–244 bits depending on mode]
      │
      ▼
[Frame Type Indicator (4 bits) + AMR payload bits]
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | 20 ms frame + ~5 ms lookahead | ~25 ms total |
| Frame size | 160 samples (20 ms at 8 kHz) | |
| Max channels | 1 (mono) | |
| Max bit depth | 13-bit linear PCM input | Internal precision varies |
| Max sample rate | 8 kHz (narrowband) | 300–3400 Hz audio bandwidth |
| Bitrate range | 4.75–12.2 kbps | 8 modes; SID frames at 1.8 kbps |
| Complexity | ~20 MIPS encoder, ~5 MIPS decoder | Hardware-accelerated in baseband |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 AMR File Storage Format (RFC 4867 / 3GPP TS 26.101)

AMR files stored in the 3GPP container (.3gp) or as raw bitstreams use the following structure:

#### AMR Interleaved Format (IF1) — 3GPP TS 26.101
```
Byte Offset   Size    Field Name              Description
-----------  ------  ---------------------  ----------------------------------
0x0000       1       Frame Header           Frame type (0–15) + interleaving flag
0x0001       N       Frame Data            AMR frame payload (95–244 bits, packed)
...          ...     ...                   Next frame
```

#### AMR File Storage Format (IF2) — 3GPP TS 26.101
```
Byte Offset   Size    Field Name              Description
-----------  ------  ---------------------  ----------------------------------
0x0000       1       Frame Header           Frame type byte (Table 1.1 of TS 26.101)
0x0001       N       Frame Data            AMR frame payload bytes
...          ...     ...                   Next frame
```

#### AMR Frame Header Byte (IF2)
```
Bit 7 (MSB)  Reserved = 0
Bits 6–3     Frame Type Index (see table below)
Bit 2        Quality indicator (0=bad frame, 1=good frame)
Bits 1–0     Table of Contents (TOC) for multi-frame:
               00 = no more frames in this block
               01 = CMR (Codec Mode Request) follows
               10 = interleaving index follows
               11 = both CMR and interleaving index follow
```

### 3.2 AMR Frame Type Index Table
| Frame Type Index | Bitrate (kbps) | Bits per Frame | Frame Size (bytes) | Description |
|-----------------|---------------|---------------|-------------------|-------------|
| 0 | 4.75 | 95 | 13 | AMR 4.75 |
| 1 | 5.15 | 103 | 14 | AMR 5.15 |
| 2 | 5.90 | 118 | 15 | AMR 5.90 |
| 3 | 6.70 | 134 | 17 | AMR 6.70 (PDC EFR) |
| 4 | 7.40 | 148 | 19 | AMR 7.40 (TDMA EFR) |
| 5 | 7.95 | 159 | 20 | AMR 7.95 |
| 6 | 10.20 | 204 | 26 | AMR 10.20 |
| 7 | 12.20 | 244 | 31 | AMR 12.20 (GSM EFR) |
| 8 | 1.80 | 39 | 6 | AMR SID (Comfort Noise) |
| 9 | — | — | — | Reserved for future use |
| 10–14 | — | — | — | Reserved for GSM/IS-95 extensions |
| 15 | — | — | — | No Data (DTX silence) |

### 3.3 Frame Bit Allocation per Mode (20 ms frame)
Per 3GPP TS 26.090, Table 1:

| Parameter | 12.2 kbps | 10.2 kbps | 7.95 kbps | 7.40 kbps | 6.70 kbps | 5.90 kbps | 5.15 kbps | 4.75 kbps |
|-----------|-----------|-----------|-----------|-----------|-----------|-----------|-----------|-----------|
| LPC (LSP) | 38 | 31 | 27 | 26 | 26 | 23 | 23 | 23 |
| Pitch delay (subframe 1) | 9 | 8 | 8 | 8 | 8 | 8 | 8 | 8 |
| Pitch gain (subframe 1) | 4 | 4 | 4 | 4 | 4 | 4 | 4 | 4 |
| Algebraic code (subframe 1) | 35 | 31 | 17 | 17 | 14 | 11 | 9 | 9 |
| Codebook gain (subframe 1) | 5 | 5 | 5 | 5 | 7 | 6 | 6 | 6 |
| Pitch delay (subframe 2) | 6 | 5 | 6 | 5 | 4 | 4 | 4 | 4 |
| Pitch gain (subframe 2) | 4 | 4 | 4 | 4 | 4 | 4 | 4 | 4 |
| Algebraic code (subframe 2) | 35 | 31 | 17 | 17 | 14 | 11 | 9 | 9 |
| Codebook gain (subframe 2) | 5 | 5 | 5 | 5 | 7 | 6 | 6 | 6 |
| Pitch delay (subframe 3) | 9 | 8 | 8 | 8 | 8 | 8 | 8 | 8 |
| Pitch gain (subframe 3) | 4 | 4 | 4 | 4 | 4 | 4 | 4 | 4 |
| Algebraic code (subframe 3) | 35 | 31 | 17 | 17 | 14 | 11 | 9 | 9 |
| Codebook gain (subframe 3) | 5 | 5 | 5 | 5 | 7 | 6 | 6 | 6 |
| Pitch delay (subframe 4) | 6 | 5 | 6 | 5 | 4 | 4 | 4 | 4 |
| Pitch gain (subframe 4) | 4 | 4 | 4 | 4 | 4 | 4 | 4 | 4 |
| Algebraic code (subframe 4) | 35 | 31 | 17 | 17 | 14 | 11 | 9 | 9 |
| Codebook gain (subframe 4) | 5 | 5 | 5 | 5 | 7 | 6 | 6 | 6 |
| **Total bits/frame** | **244** | **204** | **159** | **148** | **134** | **118** | **103** | **95** |
| **Bitrate** | **12.2 kbps** | **10.2 kbps** | **7.95 kbps** | **7.40 kbps** | **6.70 kbps** | **5.90 kbps** | **5.15 kbps** | **4.75 kbps** |

### 3.4 AMR-WB (G.722.2) Frame Type Index Table
AMR-WB operates at 16 kHz sample rate with 9 modes:

| Frame Type Index | Bitrate (kbps) | Bits per Frame | Description |
|-----------------|---------------|---------------|-------------|
| 0 | 6.60 | 132 | AMR-WB 6.60 |
| 1 | 8.85 | 177 | AMR-WB 8.85 |
| 2 | 12.65 | 253 | AMR-WB 12.65 |
| 3 | 14.25 | 285 | AMR-WB 14.25 |
| 4 | 15.85 | 317 | AMR-WB 15.85 |
| 5 | 18.25 | 365 | AMR-WB 18.25 |
| 6 | 19.85 | 397 | AMR-WB 19.85 |
| 7 | 23.05 | 461 | AMR-WB 23.05 |
| 8 | 23.85 | 477 | AMR-WB 23.85 (highest) |
| 9 | 1.75 | 35 | AMR-WB SID |
| 10–13 | — | — | Reserved |
| 14 | SID | 35 | AMR-WB SID |
| 15 | No Data | — | No transmission (DTX) |

### 3.5 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | A-law PCM | Yes | Converted to 13-bit linear internally |
| 8-bit | μ-law PCM | Yes | Converted to 13-bit linear internally |
| 13-bit | Signed linear | Yes | Native format |
| 16-bit | Signed linear | Yes | Accepted, truncated to 13-bit |
| 24-bit | Signed integer | No | Not in specification |
| 32-bit | IEEE float | No | Not in specification |

#### Supported Sample Rates
| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 8000 | Narrowband | Yes | AMR-NB primary mode |
| 16000 | Wideband | Yes | AMR-WB primary mode |
| 44100 | CD audio | No | Not applicable |
| 48000 | Professional | No | Not applicable |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Pre-Processing Stage
- **A-law / μ-law to linear conversion:** Input 8-bit A-law or μ-law PCM (as used in PSTN interconnections) is converted to 13-bit uniform linear PCM using standard G.711 conversion tables before LPC analysis
- **DC offset removal:** Applied implicitly by the high-pass pre-filter with cutoff at approximately 80 Hz
- **Pre-emphasis filter:** None explicitly; the analysis windowing handles this implicitly
- **Windowing function:** Hamming window applied to the 20 ms frame (160 samples at 8 kHz)
- **Level normalization:** Not explicitly applied; AMR operates on the raw linear PCM values

### 4.2 Analysis / Transform Stage

#### LPC Analysis
```
Parameters:
  LPC order:        10 (10th order linear prediction)
  Window:           Hamming window
  Algorithm:        Levinson-Durbin recursion on autocorrelated signal
  Coefficient precision:  Fixed-point (Q12–Q15 format in hardware)
  Update frequency:  Once per frame (20 ms) for modes ≤7.95 kbps
                     Twice per frame (10 ms) for 12.2 kbps mode
  LSP conversion:   LPC coefficients converted to Line Spectral Pairs for quantization
  LSP quantization:  Multi-stage vector quantization (MSVQ)
```

**Mathematical definition — Levinson-Durbin recursion:**
```
Given autocorrelation r[0..p], solve Toeplitz system:
  [r[0] r[1] ... r[p]]   [a[1]]   [−r[1]]
  [r[1] r[0] ... r[p-1]] [a[2] = [−r[2]]
  [ ...     ...     ...  ][ ... ]  [ ... ]
  [r[p] r[p-1] ... r[0] ]][a[p]]  [−r[p]]

Output: LP coefficients a[1..p], reflection coefficients rc[1..p]
Convert rc → LAR (Log Area Ratios) → quantize
Convert rc → LSP → quantize (AMR uses LSP for higher modes)
```

#### Pitch (Long-Term Prediction) Analysis
```
  Search range:     lag = 18 to 143 samples (2.25 ms to ~17.875 ms at 8 kHz)
  Subframe size:     40 samples (5 ms)
  Subframes per frame: 4
  Method:           Closed-loop (analysis-by-synthesis) search
  Resolution:       Integer for all modes
  Gain quantization:  4-bit scalar quantization (logarithmic)
```

#### Algebraic Codebook Structure
The AMR ACELP codebook uses a **fixed algebraic structure** — no stored codebook. This is a key differentiator from earlier CELP codecs:

```
AMR 12.2 kbps algebraic codebook structure:
  Per subframe: 2 non-zero pulses
  Pulse positions: 10-bit encoded position for each pulse
  Pulse amplitudes: ±1 (ternary: −1, 0, +1)
  Grid spacing: 1 (every sample position)
  
  Encoding: 
    Pulse 1: position p1 (0–39), amplitude ±1 (1 bit)
    Pulse 2: position p2 (0–39, p2 ≠ p1), amplitude ±1 (1 bit)
    Total: 10 + 10 + 1 + 1 = 22 bits per subframe

AMR 7.95 kbps algebraic codebook structure:
  Per subframe: 2 non-zero pulses
  Pulse positions: 8-bit (position + sign combined)
  Total: 8 + 8 = 16 bits per subframe
```

### 4.3 AMR Mode Switching and Link Adaptation
The codec supports 8 bitrates with seamless frame-by-frame switching. The **Codec Mode Request (CMR)** field in the RTP payload or file header indicates the desired mode:

```
CMR field (4 bits, in-band signaling):
  0000 = Request 4.75 kbps
  0001 = Request 5.15 kbps
  0010 = Request 5.90 kbps
  0011 = Request 6.70 kbps
  0100 = Request 7.40 kbps
  0101 = Request 7.95 kbps
  0110 = Request 10.20 kbps
  0111 = Request 12.20 kbps
  1000 = Request SID frame
  1001–1111 = Reserved
```

In cellular networks, the Radio Interface Protocol (RIPE) selects the AMR mode based on:
1. Channel Quality Indicator (CQI) from the mobile station
2. System load and interference level
3. Codec mode control commands from the base station

### 4.4 DTX (Discontinuous Transmission) and CNG
```
DTX Operation:
  1. Voice Activity Detection (VAD): Detects speech vs. silence
     - Uses signal energy, zero-crossing rate, LPC residual energy
     - VAD flag set every 20 ms frame
  2. When VAD=0 (silence): No transmission (TX=off)
  3. SID (Silence Insertion Descriptor) frames:
     - Transmitted every 8th frame during silence (~160 ms interval)
     - Contains comfort noise parameters (LSP, gain)
     - AMR SID frame: 39 bits at 1.8 kbps
  4. Receiver generates comfort noise between SID frames
     - Comfort Noise Generation (CNG): Interpolate SID parameters
     - Fade-in/fade-out to avoid clicks at speech/silence transitions

SID Frame Content (39 bits):
  - LSP quantization index (first set): 26 bits
  - Frame energy index: 6 bits
  - LSP quantization index (second set): interpolated
  - Reserved: 7 bits
```

### 4.5 Encoder Settings / Quality Modes
| Mode | Bitrate | Quality | Typical Use Case |
|------|---------|---------|----------------|
| 12.2 kbps | 12.2 kbps | Highest (toll quality) | VoLTE fallback, GSM EFR compatibility |
| 10.2 kbps | 10.2 kbps | Very High | Standard VoLTE |
| 7.95 kbps | 7.95 kbps | High | VoLTE (typical default) |
| 7.40 kbps | 7.40 kbps | Good | TDMA EFR compatibility |
| 6.70 kbps | 6.70 kbps | Good | PDC EFR compatibility |
| 5.90 kbps | 5.90 kbps | Medium | Good channel conditions |
| 5.15 kbps | 5.15 kbps | Fair | Degraded channel |
| 4.75 kbps | 4.75 kbps | Minimum | Very poor channel / interference |

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. For AMR file (.3gp, .amr):
   a. Read frame header byte (1 byte)
   b. Extract frame type index (bits 6–3)
   c. Look up frame size from Table 1.1
   d. Read frame data of the computed size
   e. Validate: check frame type is in range 0–15
   f. Check quality indicator bit (bit 2) — marks bad frames

2. For AMR over RTP:
   a. Read payload header (1–2 bytes)
   b. Extract CMR field
   c. Read AMR frames from payload
   d. TOC byte format: F=1 marks last frame in payload
   e. CMR byte (if indicated): decode mode request
```

### 5.2 Core Decode Pipeline
```
1. Parse frame header
   ├── Frame type index → determine bitrate and frame size
   ├── Quality indicator → mark frame as good/bad
   └── TOC/CMR handling for multi-frame and mode request

2. Decode LSP (Line Spectral Pairs) coefficients
   ├── Dequantize LSP indices from frame bits
   ├── Interpolate between previous and current frame LSP
   └── Convert LSP → LP coefficients via Levinson-Durbin inverse

3. For each subframe (4 subframes per 20 ms frame):
   a. Decode pitch delay index → pitch lag
   b. Decode pitch gain
   c. Decode algebraic codebook index → excitation pulses
   d. Decode codebook gain
   e. Compute pitch contribution: pitch_gain × past_excitation[pitch_lag]
   f. Compute codebook contribution: algebraic_pulses × codebook_gain
   g. Total excitation = pitch_contrib + codebook_contrib
   h. Apply LP synthesis filter: s[n] = excitation[n] + Σ(i=1..10) a[i]·s[n-i]

4. Post-processing
   ├── Adaptive pitch enhancement filter
   ├── High-pass post-filter (0.5–1.5 kHz enhancement)
   └── Comfort noise generation (during SID/silence frames)

5. Error concealment (for bad frames):
   ├── If current frame bad: repeat parameters from last good frame
   ├── Pitch period: gradually move toward median pitch
   └── Energy: attenuate over time if multiple consecutive bad frames

6. Output
   └── 160 samples × 16-bit linear PCM (8 kHz)
```

### 5.3 Error Concealment
- **Corrupt frame detection:** Quality indicator bit in frame header; CRC check at transport layer (LTE PDCP, RLC); AMR frame type out of range
- **Concealment method:** Frame repetition with attenuation; bad frame indicator (BFI) passed from channel decoder; gradual attenuation for multiple bad frames
- **Maximum consecutive errors before silence:** Implementation-dependent; typically 3–5 frames before muting to silence

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** 3GPP (ISO 14496-12 based, .3gp) for file storage; no container for raw bitstreams
- **Overhead:** ~1 byte per frame (header byte)
- **Seeking in native container:** By frame index (no seek table)
- **Multiple streams in native container:** Yes — video + audio + text in .3gp

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| 3GP (.3gp) | Yes | No | 3GPP metadata | Standard mobile format |
| AMR (.amr) | Yes (raw) | No | No | Raw bitstream, no container |
| MP4/M4A | Yes (in some profiles) | Yes | Limited | AMR in MP4 for some profiles |
| Matroska/MKA | No | — | — | Not natively supported |
| OGG | No | — | — | Not natively supported |
| WAV | No | — | — | No native AMR WAV format |
| RTP | Yes | No | No | RFC 4867 payload format |
| WebM | No | — | — | Not natively supported |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** 3GPP metadata atoms (when in .3gp container)
- **Tag block location:** Within the 3GP/MPEG-4 container atom structure
- **Tag block identifier:** `@typ` atom (handler box), `meta` atom (metadata box)

### 7.2 Standard Tag Fields — Complete Reference
| Tag Field | Internal Key (3GPP Atom) | Max Length | Character Encoding | Multi-value | Notes |
|-----------|---------------------------|------------|-------------------|-------------|-------|
| Title | `©tit` (©title) | — | UTF-8 | No | Track title |
| Artist | `©ART` (©artist) | — | UTF-8 | No | Performer |
| Album | `©alb` (©album) | — | UTF-8 | No | Album name |
| Genre | `©gen` (©genre) | — | UTF-8 | No | Genre |
| Year / Date | `©day` (©date) | — | UTF-8 | No | Year |
| Track Number | `©trk` (©track) | — | UTF-8 | No | Track number |
| Comment | `©cmt` (©comment) | — | UTF-8 | No | Comment |
| Copyright | `cprt` | — | UTF-8 | No | Copyright |
| Encoder | `©xyz` (©encoder) | — | UTF-8 | No | Encoder name |
| Album Artist | `aART` | — | UTF-8 | No | Album artist |
| Disc Number | `©dis` | — | UTF-8 | No | Disc number |

### 7.3 Cover Art Storage
Cover art in 3GP files is stored as an `covr` atom (MPEG-4/iTunes-style):

```
'covr' atom structure:
  [0x0000-0x0003]  Atom size (uint32)
  [0x0004-0x0007]  'covr' (4 bytes)
  [0x0008-...]     Cover image data
                   Multiple images may be concatenated
                   Format: JPEG or PNG
```

### 7.4 Metadata Compatibility Matrix
| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| 3GPP Atoms | ✓ | ✓ | ✓ | Highest (native) |
| ID3v2 | ✗ | ✗ | ✗ | N/A |
| Vorbis Comments | ✗ | ✗ | ✗ | N/A |
| APEv2 | ✗ | ✗ | ✗ | N/A |

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   libopencore_amrnb         # used with -c:a
AV_CODEC_ID:        AV_CODEC_ID_AMR_NB        # C constant
Format Name (CLI):  amr, 3gp                  # file format
Encoder(s):         libopencore_amrnb          # via opencore-amr library
Decoder(s):         libopencore_amrnb          # via opencore-amr library
Muxer(s):          3gp, amr                   # 3GPP muxer
Demuxer(s):        amr, 3gp                   # AMR/3GPP demuxer

AMR-WB:
Codec Name (CLI):   libopencore_amrwb         # used with -c:a
AV_CODEC_ID:        AV_CODEC_ID_AMR_WB       # C constant
Encoder(s):         libopencore_amrwb
Decoder(s):         libopencore_amrwb
```

### 8.2 FFmpeg Encoding — Full CLI Reference

```bash
# Encode WAV to AMR-NB (narrowband, 8 kHz)
ffmpeg -i input.wav \
  -c:a libopencore_amrnb \
  -ar 8000 \
  -ac 1 \
  -b:a 4.75k \                   # Bitrate: 4.75, 5.15, 5.90, 6.70, 7.40, 7.95, 10.20, 12.20 kbps
  -frame_size 160 \              # 160 samples (20 ms at 8 kHz)
  output.amr

# Encode to AMR-NB with VBR (variable bitrate)
ffmpeg -i input.wav \
  -c:a libopencore_amrnb \
  -ar 8000 \
  -ac 1 \
  -b:a 7.95k \                   # Default mode (VBR not fully supported)
  output.amr

# Encode to AMR-WB (wideband, 16 kHz)
ffmpeg -i input_wb.wav \
  -c:a libopencore_amrwb \
  -ar 16000 \
  -ac 1 \
  -b:a 23.85k \                  # Bitrate: 6.60, 8.85, 12.65, 14.25, 15.85, 18.25, 19.85, 23.05, 23.85
  output.awb
```

#### Complete FFmpeg Option Table (AMR-NB)
| Option | Type | Default | Range / Valid Values | Effect on Output |
|--------|------|---------|---------------------|-----------------|
| `-b:a` | float | 4.75k | 4.75, 5.15, 5.90, 6.70, 7.40, 7.95, 10.20, 12.20 kbps | Target bitrate mode |
| `-ar` | int | 8000 | 8000 | Sample rate (fixed) |
| `-ac` | int | 1 | 1 | Channel count (mono only) |
| `-frame_size` | int | 160 | 160 | Samples per frame (fixed) |

#### Complete FFmpeg Option Table (AMR-WB)
| Option | Type | Default | Range / Valid Values | Effect on Output |
|--------|------|---------|---------------------|-----------------|
| `-b:a` | float | 6.60k | 6.60–23.85 kbps | Target bitrate mode |
| `-ar` | int | 16000 | 16000 | Sample rate (fixed) |
| `-ac` | int | 1 | 1 | Channel count (mono only) |

### 8.3 FFmpeg Encoding — libavcodec C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/opt.h>
#include <libswresample/swresample.h>

// ─── 1. Find encoder ─────────────────────────────────────────────────────────
const AVCodec *codec = avcodec_find_encoder(AV_CODEC_ID_AMR_NB);
if (!codec) { fprintf(stderr, "AMR-NB encoder not found\n"); exit(1); }

// ─── 2. Allocate codec context ────────────────────────────────────────────────
AVCodecContext *ctx = avcodec_alloc_context3(codec);
if (!ctx) { fprintf(stderr, "OOM\n"); exit(1); }

// ─── 3. Set required parameters ────────────────────────────────────────────────
ctx->bit_rate    = 7950;           // bits/sec (7.95 kbps is common default)
ctx->sample_fmt  = AV_SAMPLE_FMT_S16; // AMR requires signed 16-bit PCM
ctx->sample_rate = 8000;           // Hz — MUST be 8000 for AMR-NB
av_channel_layout_default(&ctx->ch_layout, 1); // Mono only

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
frame->nb_samples  = ctx->frame_size; // Fixed at 160 for AMR-NB (20 ms at 8 kHz)
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
- AMR-NB requires exactly 8000 Hz input; resample to 8000 Hz before encoding
- AMR-WB requires exactly 16000 Hz input; resample to 16000 Hz before encoding
- `ctx->frame_size` is 160 samples (AMR-NB) and 320 samples (AMR-WB)
- AMR encoder accepts only `AV_SAMPLE_FMT_S16`; use libswresample to convert from other formats
- AMR output is raw bitstream or wrapped in 3GP container

### 8.4 FFmpeg Decoding — Full CLI Reference

```bash
# Decode AMR-NB to WAV PCM
ffmpeg -i input.amr \
  -c:a pcm_s16le \
  -ar 8000 \
  -ac 1 \
  output.wav

# Decode AMR-WB to WAV PCM
ffmpeg -i input.awb \
  -c:a pcm_s16le \
  -ar 16000 \
  -ac 1 \
  output_wb.wav

# Decode 3GP to WAV
ffmpeg -i input.3gp \
  -c:a pcm_s16le \
  output.wav

# Probe format without decoding
ffprobe -v quiet -print_format json -show_streams -show_format input.amr
```

### 8.5 FFmpeg Decoding — libavcodec C API

```c
// ─── Complete decode program skeleton ────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.amr", NULL, NULL);
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
            // frm->nb_samples = 160 (AMR-NB) or 320 (AMR-WB)
            // frm->sample_rate = 8000 or 16000
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
ffprobe -v quiet -print_format json -show_format input.3gp | jq .format.tags

# Write metadata (3GP container)
ffmpeg -i input.3gp \
  -c:a copy \
  -metadata title="Recording Title" \
  -metadata artist="Speaker Name" \
  -metadata date="2024" \
  output_tagged.3gp

# Strip all metadata
ffmpeg -i input.3gp -c:a copy -map_metadata -1 output_clean.3gp
```

#### FFmpeg Internal Metadata Key Mapping
| Standard Field | FFmpeg Key | 3GPP Atom Key | Notes |
|----------------|------------|----------------|-------|
| Title | title | ©tit | |
| Artist | artist | ©ART | |
| Album | album | ©alb | |
| Album Artist | album_artist | aART | |
| Track Number | track | ©trk | |
| Disc Number | disc | ©dis | |
| Genre | genre | ©gen | |
| Date/Year | date | ©day | |
| Comment | comment | ©cmt | |
| Copyright | copyright | cprt | |
| Encoder | encoder | ©xyz | Auto-set by encoder |

### 8.7 Quality / Fidelity Decision Table
| Use Case | Settings | Expected Size | Notes |
|----------|----------|---------------|-------|
| VoLTE (standard) | `-c:a libopencore_amrnb -b:a 7.95k` | ~24 KB/min | Default VoLTE |
| VoLTE (high quality) | `-c:a libopencore_amrnb -b:a 12.2k` | ~37 KB/min | AMR EFR compatible |
| Mobile (low bandwidth) | `-c:a libopencore_amrnb -b:a 4.75k` | ~14 KB/min | Weak signal |
| AMR-WB (HD voice) | `-c:a libopencore_amrwb -b:a 23.85k` | ~72 KB/min | Wideband |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
```
AMR files do not contain a seek table.

Seeking in AMR:
  - Byte offset = frame_index × (frame_size + 1 byte header)
  - Frame index = floor(target_time_ms / 20)
  - Precision: ±20 ms (frame-accurate)
  - No random access in raw AMR; seek by scanning from beginning

In 3GP containers:
  - Seeking via media timeline in the movie atom ('moov')
  - AMR track 'tkhd' atom provides duration
  - Seeking to nearest sync sample (not applicable — all AMR frames are keyframes)
```

### 9.2 Gapless Playback Data
```
Encoder delay:   ~5 ms (lookahead in pitch analysis)
Padding:         ~0 ms (no encoder padding added)
Storage location: Not stored in AMR bitstream
                  Total samples = frames × 160
                  Actual audio = frames × 160 (no delay/padding recorded)

AMR is a cellular codec with very low inherent latency.
The channel coding (GSM 05.03 / LTE PDCP) adds additional framing overhead.
Gapless playback is generally not a concern for voice codecs.
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | Frame-by-frame decode |
| Algorithmic encoder delay | 20 ms frame + ~5 ms lookahead | ~25 ms total |
| Live encoding feasible | Yes | Designed for real-time cellular voice |
| HTTP progressive download | Yes | Supported for .3gp and .amr files |
| HTTP Live Streaming (HLS) | No | Not commonly used for AMR |
| DASH streaming | Yes | 3GPP DASH profile for video+voice |
| WebRTC / RTP transport | Yes | RFC 4867 defines AMR over RTP/AVP |
| Minimum decode buffer | 1 frame (20 ms) | Very low latency |

**RTP Payload Format (RFC 4867):**
```
AMR RTP Packet Structure:
  [0x00]  Payload Header (1 byte)
           Bits 7–4: Frame type index (0–8, 15)
           Bit 3:   Start of talkspurt (1 = first frame after silence)
           Bit 2:   Reserved (must be 0)
           Bits 1–0: Table of Contents (TOC) flags
  [0x01+] AMR frame(s) — one or more frames per RTP packet
           Each frame: frame header + payload
           TOC byte after first frame if CMR/interleaving needed
```

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | Layout Name | Channel Order | FFmpeg Layout Constant | Notes |
|----------|-------------|---------------|----------------------|-------|
| 1 | Mono | C | AV_CHANNEL_LAYOUT_MONO | Only supported mode |

**Note:** AMR-NB and AMR-WB are strictly mono codecs. Multi-channel voice is handled by multiplexing multiple AMR sessions or using a different codec. There is no stereo AMR mode.

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 13-bit linear PCM input | Internal processing at higher precision |
| Max sample rate | 16 kHz (AMR-WB) | AMR-NB is 8 kHz only |
| Float support | No | Fixed-point only |
| DSD support | No | Not applicable |
| 20-bit support | No | Not in specification |
| 24-bit support | No | Not in specification |

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

| Platform / API | Encode | Decode | FFmpeg Flag | Notes |
|----------------|--------|--------|-------------|-------|
| Cellular baseband (Qualcomm, MediaTek, etc.) | Yes | Yes | Native | Hardware-accelerated in baseband DSP |
| Apple Core Audio | Yes | Yes | — | iOS/macOS system codecs |
| Android MediaCodec | Yes | Yes | MediaCodec | AMR-NB/WB via platform decoder |
| x86/x64 (generic) | Yes | Yes | libopencore_amrnb/wb | Software via opencore-amr |
| ARM (32/64-bit) | Yes | Yes | libopencore_amrnb/wb | NEON-optimized software |
| Intel QSV | No | No | — | Not supported |
| VA-API | No | No | — | Not applicable |

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Encoder-Specific Issues
| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| libopencore_amrnb not compiled in FFmpeg | Many | Build FFmpeg with `--enable-libopencore_amrnb` |
| AMR encoder bitrate selection | All | Use exact bitrate values; mode is selected by bitrate |
| AMR-WB encoder mode not supported | Some | Check `ffmpeg -encoders` for AMR-WB availability |
| DTX encoding not controllable | All | DTX is implementation-dependent |

### 14.2 Interoperability Issues
- **AMR frame type mismatch:** Some implementations may not support all 8 modes. Minimum compliance requires modes 4.75 kbps and 12.2 kbps.
- **3GP container vs raw AMR:** Some players only accept raw `.amr` files, others only `.3gp`. FFmpeg can convert between formats.
- **A-law vs μ-law input:** AMR decoder must know whether input was A-law or μ-law. This is signaled in the 3GP container or RTP payload.
- **SID frames:** When DTX is active, SID frames (type 8) contain comfort noise parameters. Some decoders may not properly generate comfort noise.

### 14.3 Edge Cases to Handle in Converter
- **Empty file / 0-sample stream:** Produce empty output file with proper headers
- **File < 1 frame of audio:** Encode as single frame with zero padding
- **All-silence audio:** AMR encoder generates SID frames and no-data frames (DTX)
- **Corrupt frame type:** Skip frame; treat as bad frame (BFI)
- **Sample rate not 8 kHz (AMR-NB):** FFmpeg automatically resamples to 8 kHz
- **Channel count > 1:** FFmpeg auto-downmixes stereo to mono

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM AMR

| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → FLAC | `ffmpeg -i in.amr -c:a flac -compression_level 8 out.flac` | 3GP metadata | Lossless decode |
| → WAV | `ffmpeg -i in.amr -c:a pcm_s16le -ar 8000 out.wav` | RIFF INFO (partial) | Lossless decode |
| → MP3 | `ffmpeg -i in.amr -c:a libmp3lame -q:a 2 out.mp3` | ID3v2 (title/artist) | Generation loss |
| → AAC | `ffmpeg -i in.amr -c:a aac -b:a 64k out.m4a` | Limited | Generation loss |
| → Opus | `ffmpeg -i in.amr -c:a libopus -b:a 24k out.opus` | No | Recommended upgrade |
| → EVS | Not available in FFmpeg | — | EVS requires carrier infrastructure |

### 15.2 Converting TO AMR

| Source | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| WAV (8kHz) → | `ffmpeg -i in.wav -c:a libopencore_amrnb -b:a 7.95k -ar 8000 out.amr` | None | Lossy |
| WAV (16kHz) → | `ffmpeg -i in.wav -c:a libopencore_amrwb -b:a 23.85k -ar 16000 out.awb` | None | Lossy |
| FLAC → | `ffmpeg -i in.flac -c:a libopencore_amrnb -ar 8000 out.amr` | None | Transcode lossy |

### 15.3 Lossless Round-Trip Verification
```bash
# AMR is lossy — true lossless round-trip is NOT possible
# Verify decode pipeline:

# Decode AMR to PCM
ffmpeg -i original.amr -c:a pcm_s16le -ar 8000 decoded.wav

# Decode from source WAV (for reference)
ffmpeg -i source.wav -c:a pcm_s16le -ar 8000 source_raw.wav

# Compare (should differ due to lossy AMR encoding)
md5sum decoded.wav source_raw.wav
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| opencore-amr (3GPP reference) | C/C++ | Apache 2.0 | Reference | Reference | https://sourceforge.net/projects/opencore-amr/ |
| FFmpeg native wrapper | C | LGPL 2.1+ | Good | Good | https://ffmpeg.org |
| Android platform AMR | C | Proprietary | Good | Good | Platform-provided |
| Apple CoreAudio AMR | C | Proprietary | Good | Good | macOS/iOS system |

### Build Instructions (for bundling in converter app)
```bash
# Build opencore-amr from source
git clone https://git.code.sf.net/p/opencore-amr/code opencore-amr
cd opencore-amr
./bootstrap
./configure --prefix=/usr/local --disable-shared --enable-static
make -j$(nproc)
make install

# Build FFmpeg with AMR support:
./configure --prefix=/usr/local --enable-libopencore_amrnb \
  --enable-libopencore_amrwb --disable-shared --enable-static
make -j$(nproc)
make install

# Link:
# LDFLAGS += -L/usr/local/lib -lopencore-amrnb -lopencore-amrwb
# CFLAGS  += -I/usr/local/include
```

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **3GPP TS 26.071:** AMR Speech Codec — General Description — https://www.3gpp.org/specifications/
- **3GPP TS 26.090:** AMR Speech Codec — Transcoding Functions
- **3GPP TS 26.101:** AMR Speech Codec — Frame Structure
- **RFC 4867:** RTP Payload Format and File Storage Format for AMR and AMR-WB — https://datatracker.ietf.org/doc/html/rfc4867
- **RFC 4281:** The Codecs Parameter for "Bucket" Media Types

### Technical Resources
- FFmpeg encoder options: `ffmpeg -h encoder=libopencore_amrnb`
- Opencore AMR source: https://sourceforge.net/projects/opencore-amr/
- 3GPP specification portal: https://www.3gpp.org/specifications/
- Multimedia Wiki: https://wiki.multimedia.cx/index.php/AMR
- Via Licensing (AMR patents): https://www.vialicensing.com/

### Academic Papers
- 3GPP, "AMR Speech Codec: General Description," 3GPP TS 26.071 series
- "A Comparison of AMR and EVS Codecs for VoLTE," comparative analysis literature
- ETSI, "Mandatory Speech Codec Speech Processing Functions; Adaptive Multi-Rate (AMR) Speech Codec Transcoding Functions," 3GPP TS 26.090

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] Identify FFmpeg `./configure` flags: `--enable-libopencore_amrnb --enable-libopencore_amrwb`
- [ ] Verify `ffmpeg -encoders` output confirms AMR-NB encoder is available
- [ ] Verify `ffmpeg -decoders` output confirms AMR-NB/WB decoder is available
- [ ] Note external library dependency: `libopencore-amrnb`, `libopencore-amrwb`
- [ ] AMR is widely supported on mobile; hardware decoders may be available

### Encoding Pipeline
- [ ] Convert input sample format to S16 using libswresample
- [ ] Resample to exactly 8000 Hz (AMR-NB) or 16000 Hz (AMR-WB)
- [ ] Validate `ctx->frame_size` = 160 (AMR-NB) or 320 (AMR-WB)
- [ ] Use 3GP muxer for file output (`-f 3gp`) or AMR muxer (`-f amr`)
- [ ] Implement proper flush/drain at end of stream (send NULL frame)
- [ ] Handle bitrate-to-mode mapping: FFmpeg selects mode from bitrate value

### Decoding Pipeline
- [ ] Use AMR/3GP demuxer to read .amr/.3gp files
- [ ] Handle AVERROR(EAGAIN) in receive_packet loop correctly
- [ ] Flush decoder at EOF (send NULL packet)
- [ ] Handle S16 output format (16-bit signed PCM)
- [ ] Resample to output sample rate if different from source

### Metadata
- [ ] Read 3GP metadata atoms (©tit, ©ART, ©alb, etc.)
- [ ] Write 3GP metadata atoms to output container
- [ ] Read cover art from `covr` atom
- [ ] Write cover art to `covr` atom
- [ ] Handle character encoding: UTF-8 in 3GP containers

### Quality & Verification
- [ ] Test encode/decode at all 8 AMR-NB bitrates
- [ ] Test encode/decode at all 9 AMR-WB bitrates
- [ ] Verify SID frame generation during silence
- [ ] Test with: speech, DTMF tones, music on hold, silence

### Edge Cases
- [ ] Handle files with corrupt frame headers (invalid frame type)
- [ ] Handle files with 0 samples
- [ ] Handle sample rate mismatch (resample to 8k/16k)
- [ ] Handle stereo input (auto-downmix to mono)
- [ ] Handle very short files (< 1 frame)

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
