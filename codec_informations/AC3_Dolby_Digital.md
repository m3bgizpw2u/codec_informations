# Dolby Digital (AC-3) — Deep Technical Reference
> **Category:** Lossy
> **File Extensions:** `.ac3`, `.eac3` (E-AC-3)
> **MIME Types:** `audio/ac3`, `audio/eac3`, `audio/dolbydigital`
> **Standardization Body:** ATSC (Advanced Television Systems Committee), Dolby Laboratories, ETSI
> **Primary Specification:** ATSC A/52:2018, ETSI TS 102 366
> **Patent Status:** Patented — royalty-bearing (Dolby Laboratories licensing)
> **License:** Proprietary — licensed through Dolby Laboratories
> **Current Version:** A/52:2018 (AC-3 and E-AC-3 combined)
> **Active Development:** No — standard is stable/deprecated for new deployments, maintained for backward compatibility

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Dolby Laboratories (Craig Todd, Nicholas G. C. H. F. etc.)
- **Year Created:** 1991 (original AC-3), 2004/2005 (E-AC-3 / Dolby Digital Plus)
- **Original Purpose:** Efficient delivery of multi-channel digital audio over limited-bandwidth channels (digital television, DVD, laserdisc)
- **Problem with Predecessors:** Prior to AC-3, multi-channel audio required separate analog channels or PCM. Dolby Surround (matrix encoding) was lossy and had channel bleed. AC-3 provided discrete multi-channel audio with perceptually optimized lossy compression in a single data stream at broadcast bitrates.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| AC-3 1.0 | 1991 | Initial release; 5.1-channel support, 320 kbps max |
| A/52 | 1994 | ATSC standardization for US HDTV |
| A/52A | 1998 | Added dialog normalization, dynamic range control enhancements |
| A/52B | 2001 | Incorporated into DVD-Video standard |
| A/52:2010 | 2010 | Combined AC-3 and E-AC-3 in one document |
| A/52:2018 | 2018 | Current revision; no fundamental changes to AC-3 core |
| E-AC-3 (Dolby Digital Plus) | 2004 | Annex E of A/52; non-backward-compatible extension with higher bitrates, more channels |

### 1.3 Current Adoption
- **Primary use cases today:** Digital television (ATSC, DVB), streaming (Netflix, Amazon Prime legacy), Blu-ray Disc (secondary audio), DVD-Video legacy
- **Platforms with native support:** Windows, macOS, iOS, Android, Linux (via FFmpeg/libav), game consoles, AV receivers, TVs, set-top boxes
- **Major services using this format:** Legacy streaming services, digital broadcast (ATSC 1.0), cable/satellite TV
- **Hardware support:** Universal in AV receivers, TVs, game consoles (PlayStation, Xbox, Nintendo), Blu-ray players, set-top boxes, car audio
- **Status:** Declining for new content — replaced by E-AC-3 (Dolby Digital Plus), Dolby Atmos (via E-AC-3 + Atmos metadata), and lossless formats (Dolby TrueHD, DTS-HD MA) on Blu-ray/streaming

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Perceptual / transform
- **Core algorithm:** Modified Discrete Cosine Transform (MDCT) via analysis/synthesis filter bank with Time Domain Aliasing Cancellation (TDAC), combined with subband grouping and parametric bit allocation
- **Loss mechanism:** Perceptual masking via psychoacoustic model; quantization of mantissas; coupling of high-frequency coefficients across channels
- **Frame-based vs sample-based:** Frame-based. Each sync frame contains exactly 6 audio blocks.
- **Fixed vs variable frame size:** Fixed frame duration. At 48 kHz: 1536 samples/frame (6 blocks × 256 samples/block) = 32 ms frame duration. At 44.1 kHz: frame size varies with sample rate code.

### 2.2 High-Level Encoding Flow (Block Diagram)
```
Input PCM Samples (32/44.1/48 kHz, up to 5.1 channels)
      │
      ▼
[Pre-processing: High-pass filter (DC removal), input formatting]
      │
      ▼
[Analysis Filter Bank: 512-band QMF polyphase → split into 64 subbands (low frequencies)
                         + remaining bands (high frequencies)]
      │
      ▼
[MDCT Transform: 256-point MDCT per subband group, with window switching
                  for transient detection]
      │
      ▼
[Exponent Coding: Encode spectral envelope as exponents (D15/D25/D45/REUSE strategies)]
      │
      ▼
[Bit Allocation: Hybrid forward/backward adaptive parametric bit allocation
                 computes masking curve and allocates bits to mantissas]
      │
      ▼
[Mantissa Quantization: Uniform symmetric quantizer (3/5/7/11/15 levels) per coefficient]
      │
      ▼
[Coupling: Optionally combine high-frequency coefficients across channels
            into a single coupling channel + angle coordinates]
      │
      ▼
[Stereo Rematrixing: Optionally encode Lt/Rt instead of L/R for stereo input]
      │
      ▼
[Metadata Packing: Dialog level, DRC, compression, channel mode, sample rate, bitrate]
      │
      ▼
[CRC Error Detection: Two 16-bit CRCs per frame]
      │
      ▼
Output AC-3 Bitstream (synchronization frames)
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | 256 samples / 5.33 ms (at 48 kHz) | Per audio block latency |
| Frame size | 1536 samples (6 blocks × 256) | At 48 kHz = 32 ms per frame |
| Max channels | 6 (5.1: L, C, R, SL, SR, LFE) | Plus optional matrixed 6.1 |
| Max bit depth | 24-bit input (internal precision varies) | Output is perceptually coded |
| Max sample rate | 48 kHz (AC-3); 48 kHz base (E-AC-3) | E-AC-3 adds 44.1, 32 kHz |
| Bitrate range | 32–640 kbps | Varies by sample rate; see bitrate table |
| Complexity | O(N log N) per transform | Encoder is significantly heavier than decoder |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  2       0B 77            —         Sync word (0x0B77, big-endian)
0x0002  2       varies           —         CRC-16 (1st half of frame) + frame size code
```

### 3.2 Sync Frame Header Layout (Synchronization Information + BSI)
```
Offset   Bit Range   Field Name              Type      Valid Range              Description
-------  ----------  --------------------    --------  ----------------------   ---------------------------
0        [15:0]     Sync Word               uint16    0x0B77                   Fixed sync word
16       [1:0]      fscod (Sample Rate)      uint2     0–3                     Sample rate code (see table)
18       [5:2]      frmsizecod (Frame Size)  uint4     0–59                    Frame size code (see table)
                                                                                Combined with fscod for frame byte size
         ... BSI (Bit Stream Information) begins after SI ...
SI+0     [1:0]      fscod (SI)              uint2     0–3                     Duplicate of SI field (see fscod table)
SI+2     [5:2]      frmsizecod               uint4     0–59                    Duplicate of SI field
SI+6     [3:0]      bsid (Bit Stream ID)    uint5     0–16                    Codec version; 8=E-AC-3, 0=AC-3
SI+11    [3:0]      bsmod (Bit Stream Mode) uint5     0–12                    Audio service type (main, associated, etc.)
SI+16    [2:0]      acmod (Audio Coding)     uint3     0–7                     Channel configuration (see table)
SI+19    [0]        if acmod==1 (1+1 mode): lfeon  bit     0/1                  LFE channel present flag
SI+19    [3:0]      if acmod!=001:           cplincstr  uint4                   Coupling in-use flag + strategy
         ... more BSI fields follow ...
         Exponent strategy, coupled channels, bit allocation pointers,
         dialog normalization, dynamic range, compression, etc.
```

#### Sample Rate Code (fscod)
| fscod | Sample Rate (Hz) | Frame Size (bytes) at 48 kHz | Frame Duration (ms) |
|-------|-------------------|-------------------------------|---------------------|
| 0     | 48 kHz            | See frmsizecod table         | 32 ms               |
| 1     | 44.1 kHz          | See frmsizecod table         | 34.83 ms            |
| 2     | 32 kHz            | See frmsizecod table         | 48 ms               |
| 3     | Reserved          | —                            | —                   |

#### Frame Size Code (frmsizecod) Table for 48 kHz
| frmsizecod | Bitrate (kbps) | Frame Size (bytes) | Notes |
|-------------|----------------|---------------------|-------|
| 0           | 32             | 128                 |       |
| 1           | 40             | 160                 |       |
| 2           | 48             | 192                 |       |
| 3           | 56             | 224                 |       |
| 4           | 64             | 256                 |       |
| 5           | 80             | 320                 |       |
| 6           | 96             | 384                 |       |
| 7           | 112            | 448                 |       |
| 8           | 128            | 512                 |       |
| 9           | 160            | 640                 |       |
| 10          | 192            | 768                 |       |
| 11          | 224            | 896                 |       |
| 12          | 256            | 1024                |       |
| 13          | 320            | 1280                |       |
| 14          | 384            | 1536                | DVD max for 5.1 |
| 15          | 448            | 1792                | DVD-Video max     |
| 16          | 480            | 1920                | [NEEDS VERIFICATION] |
| 17          | 512            | 2048                | [NEEDS VERIFICATION] |
| 18          | 576            | 2304                | [NEEDS VERIFICATION] |
| 19          | 640            | 2560                | Blu-ray max       |
| 20–59       | Reserved       | —                   |                     |

**Frame size formula (AC-3):**
```
frame_size = 2 × (frame_size_code + 1)   [for 48 kHz]
frame_size = 3 × (frame_size_code + 1)   [for 44.1 kHz]
frame_size = 4 × (frame_size_code + 1)   [for 32 kHz]
```

#### Channel Mode (acmod) Table
| acmod | Name | Channels | Description |
|-------|------|----------|-------------|
| 0     | 1+1  | 2 (dual mono) | Two independent mono channels (A/B) |
| 1     | 1/0  | 1 (mono)     | Single full-bandwidth channel |
| 2     | 2/0  | 2 (stereo)   | Left and Right |
| 3     | 3/0  | 3             | L, C, R |
| 4     | 2/1  | 3             | L, R, single surround (S) |
| 5     | 3/1  | 4             | L, C, R, single surround (S) |
| 6     | 2/2  | 4             | L, R, SL, SR |
| 7     | 3/2  | 5             | L, C, R, SL, SR (+ LFE if lfeon=1 → 5.1) |

### 3.3 Audio Block Structure
Each AC-3 sync frame contains 6 audio blocks (AB0–AB5). Each block encodes 256 new samples per channel.

```
Audio Block N structure:
  [2 bits] Block switching flag (per channel)
  [2 bits] Dither flags (per channel)
  [1 bit]  Coupling strategy (coupling in-use)
  [2 bits] Bit allocation strategy (coupled channel pairs)
  [2 bits] Exponent strategy (D15/D25/D45/REUSE)
  [variable] Exponents (D15=1 exp per bin, D25=1 per 2 bins, D45=1 per 4 bins)
  [variable] Bit allocation data (fseg, ftab, bit pool)
  [variable] SNR offset, delta bit allocation
  [variable] Quantized mantissas
  [variable] Auxiliary data
  [1 bit]  CRC check flag
```

### 3.4 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 16-bit | Signed integer (PCM) | Yes | Standard input for consumer formats |
| 20-bit | Signed integer | Yes | Professional / broadcast |
| 24-bit | Signed integer | Yes | High-resolution input |
| 32-bit | IEEE float | Yes | Via FFmpeg float path |
| 8-bit | Unsigned integer | No | Not supported by specification |

#### Supported Sample Rates
| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 32000 | Broadcast | Yes | ATSC/DVB broadcast |
| 44100 | CD audio | Yes | Via 44.1 kHz frame coding |
| 48000 | Professional / DVD/Blu-ray | Yes | Primary rate; default for DVD, Blu-ray, ATSC |
| 8000 | Telephone | No | Not supported in AC-3 |
| 16000 | Wideband | No | Not in AC-3; available in E-AC-3 |
| 96000 | High-res | No | Not in AC-3; use E-AC-3 or Dolby TrueHD |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Pre-Processing Stage
- **DC offset removal:** High-pass filter applied to input with −3 dB cutoff approximately 3 Hz above DC
- **Input formatting:** Input PCM is split into 256-sample blocks (6 blocks per frame)
- **Windowing:** Before MDCT, a sine window (or transient window) is applied. Long blocks use a full sine window; transient blocks use a shorter window shape to minimize pre-echo artifacts.
- **Level normalization:** Not explicitly; managed via exponent coding in the bitstream
- **Stereo decorrelation pre-step:** Optional stereo rematrixing (Lt/Rt encoding) for stereo input

### 4.2 Analysis / Transform Stage

#### Transform Type: Modified DCT (MDCT) with Analysis/Synthesis Filter Bank
```
Parameters:
  Window size:     512 samples (analysis) → 256 samples (synthesis)
  Overlap:        50% (MDCT standard — TDAC provides perfect reconstruction)
  Window function: Sine window (long), transient window (short)
  Subband split:  512-band QMF → low 63 subbands kept separate,
                   remaining subbands grouped
```

**Mathematical definition of the AC-3 Analysis Filter Bank:**

The encoder first applies a 512-band polyphase quadrature mirror filter (QMF) bank to split the full spectrum:
```
Each QMF band k contains a weighted sum of input samples n:
  X_qmf[k][m] = Σ(n=0 to 511) x[m×128 + n] × h_QMF[n][k]
  where h_QMF is the analysis prototype filter
```

The 512 QMF outputs are then transformed via a second-stage MDCT. Low-frequency bands (0–62) are kept at full resolution. Higher bands are grouped to save bits.

**Block switching:**
- Long block: 512-sample window → 256 output MDCT coefficients per band
- Short block: Triggered by transient detection (sudden energy attacks)
  - Switch sequence: LONG → SHORT(×8) → LONG (or START → SHORT × 6 → STOP → LONG)
  - Short block uses 256-sample window with appropriate window shape
  - Transition windows (START, STOP) used to maintain perfect reconstruction during mode switch

### 4.3 Psychoacoustic Model (Lossy Only)

#### Model Version
- **Model:** Proprietary parametric psychoacoustic model (not published in full)
- **Analysis window:** 256 samples (one audio block) — corresponds to ~5.33 ms at 48 kHz
- **Key metrics computed:** Excitation pattern (masking curve), signal-to-mask ratio per frequency bin

#### Masking Thresholds
```
Absolute Threshold of Hearing (ATH):
  ATH(f) ≈ 3.64·(f/1000)^(-0.8) - 6.5·exp(-0.6·(f/1000 - 3.3)^2) + 10^(-3)·(f/1000)^4 dB

Simultaneous Masking:
  Masking slope (upward):   ~15 dB/Bark
  Masking slope (downward): ~25 dB/Bark

Temporal Masking:
  Pre-masking:  ~50–100 ms (limited by encoder window size)
  Post-masking: ~100–200 ms
```

#### Bit Allocation Algorithm (Parametric Hybrid Forward/Backward Adaptive)
The AC-3 bit allocation is the defining algorithmic feature of the codec:

```
AC-3 Bit Allocation Process:

1. Compute power spectral density (PSD) from exponents
   PSD[b] = 2^(3 × exponent[b])  [per critical band b]

2. Compute excitation function via excitation table lookups
   excitation[b] = Σ(k) table[fcx][k] × PSD[band[k]]

3. Compute masking curve (hearing threshold + masking)
   mask[b] = min(ATH[b], excitation[b])

4. Compute bit allocation from masking curve
   bitalloc[b] = f(PSD[b], mask[b], snroffset)
   where snroffset is an encoder-controlled quality parameter

5. Apply delta bit allocation (deltba) if signaled
   - dba_mode: REUSE, NEW, DELTA
   - Allows encoder to refine allocation frame-to-frame

6. Allocate bits to mantissas based on bitalloc values
   Higher bitalloc → more quantization levels (finer quantization)
```

**snroffset (Signal-to-Noise Ratio Offset):**
- Range: 0–1023 (stored as 6-bit fgaincod + bit allocation master gain)
- Higher values → better quality (more bits allocated to mantissas)
- Controls the margin above the masking threshold

### 4.4 Quantization
- **Type:** Uniform symmetric quantization (mid-tread) for mantissas
- **Quantization levels:** 3, 5, 7, 11, or 15 levels per coefficient (depending on bit allocation)
- **Exponent encoding:** Differential coding across frequency bins
  - **D15:** One exponent per spectral line (full resolution)
  - **D25:** One exponent per 2 spectral lines
  - **D45:** One exponent per 4 spectral lines
  - **REUSE:** Reuse exponents from previous block (for stationary signals)
- **Dequantization formula:**
  ```
  quantized_mantissa[i] = sign(code) × |code| × 2^(-(gain - 210)/4)
  ```
- **Noise shaping:** Not explicitly; achieved through bit allocation targeting the masking curve

### 4.5 Stereo Encoding Modes
| Mode | acmod | Description | Condition for Selection |
|------|-------|-------------|------------------------|
| Stereo (independent) | 2 | L and R encoded independently | Default for stereo content |
| Joint Stereo (MS) | 2 | Mid=(L+R)/2, Side=(L−R)/2 | Optional; may be applied at encoder's discretion |
| Lt/Rt (matrixed) | 2 | ProLogic-compatible matrix encoding | Encoder option for surround compatibility |
| Intensity Stereo | — | Couple high frequencies, send per-channel angle | E-AC-3 only; at very low bitrates |
| Dual Mono | 0 | Two independent mono channels (A/B) | Forced for bilingual content |

### 4.6 Coupling
Coupling is an efficiency technique used when the encoder runs low on bits:

```
Coupling process:
1. Above a certain frequency (coupling frequency, controlled by cplbegf in bitstream)
2. Two or more channels share a single "coupling channel" containing the sum of their spectra
3. Each coupled channel sends "angle coordinates" (m + Δ angle) to reconstruct individual channels
4. Reconstructed = coupling_channel × cos(angle) + (per-channel delta)
5. cplstartf and cplendf specify the frequency range of coupling
```

Coupling can be applied per block and per channel pair. The coupling start band is configurable (default: band 4, approximately 2.8 kHz at 48 kHz).

### 4.7 Encoder Settings / Quality Modes

#### For Lossy AC-3
| Quality Setting | Bitrate (Stereo) | Bitrate (5.1) | Intended Use Case | Transparent? |
|----------------|------------------|---------------|-------------------|--------------|
| Lowest | 32–64 kbps | 64–96 kbps | Voice, secondary audio | No |
| Low | 96–128 kbps | 192–224 kbps | Low bandwidth streaming | No |
| Medium | 128–192 kbps | 256–320 kbps | Standard streaming | Marginal |
| High | 192–256 kbps | 384–448 kbps | DVD-Video, Blu-ray | Near-transparent |
| Highest | 320 kbps | 640 kbps | Blu-ray, archival lossy | Transparent (most content) |

**Notes on transparency:**
- Dolby stated 64 kbps per channel for transparency (confirmed via Dolby Labs correspondence)
- For stereo: ~192 kbps is generally considered transparent for music
- For 5.1: 384–448 kbps is the DVD-Video standard range; 640 kbps is Blu-ray maximum
- Subjective quality depends heavily on source material complexity

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. Scan bitstream for sync word: 0x0B77 (16 bits, big-endian)
2. Validate candidate frame:
   a. Parse fscod and frmsizecod from SI
   b. Compute frame_size from frmsizecod formula
   c. Verify CRC-16 of first 5 bytes + SI + BSI
   d. Check next frame at computed offset also has valid sync
3. If validation fails: advance 1 byte and retry
4. Cache up to 5 CRCs for error concealment decisions
```

#### Seeking
- **CBR seeking:** `byte_offset = floor(target_time_sec × bitrate_bps / 8)`
- **VBR seeking:** Requires frame scan or seek table (not natively supported by AC-3)
- **Seek table format:** Not defined in AC-3; seek tables are container-specific (e.g., MPEG-TS PSI, program map)
- **Seek precision:** Frame-accurate (32 ms at 48 kHz); sample-accurate requires interpolation

### 5.2 Core Decode Pipeline
```
1. Read SI (Synchronization Information) — 2 bytes
   ├── Verify sync word 0x0B77
   ├── Parse fscod → determine sample rate
   └── Parse frmsizecod → compute frame_size

2. Read BSI (Bit Stream Information) — variable length
   ├── Parse bsid (must be 0 for AC-3; 8 for E-AC-3)
   ├── Parse bsmod → audio service type
   ├── Parse acmod → channel configuration
   ├── Parse lfeon → LFE present flag
   ├── Parse dialog normalization level (dialnorm)
   ├── Parse compression (compr) and dynrnge
   └── Parse coupling flags, rematrix flags

3. Read Audio Block 0–5 (1536 samples per channel per frame)
   For each block:
   ├── Parse block switch flags (per channel)
   ├── Parse dither flags (per channel)
   ├── Parse coupling strategy (cplstre)
   ├── Parse exponent strategy → determine D15/D25/D45/REUSE
   ├── Decode exponents (differential, per band)
   ├── Decode bit allocation data (fgaincod, sdecay, fdecay, etc.)
   ├── Compute masking curve using bit allocation parameters
   ├── Apply delta bit allocation if present (deltba)
   ├── Decode SNR offset (snroffst) → final bit allocation
   ├── Decode quantized mantissas per frequency bin
   ├── Apply coupling channel + angle coordinates if coupled
   └── Decode LFE (separate downsampled path, 128 samples/block)

4. Decode exponents and requantize mantissas
   For each frequency bin:
     requantized = sign(mantissa) × |mantissa| × 2^(-(bitalloc[bin] + 210)/4)

5. Inverse transform:
   a. Undo coupling: reconstruct individual channel spectra from coupling channel
   b. Grouped bands: expand coefficients back to full resolution
   c. Apply inverse MDCT (IMDCT) per subband
   d. Apply synthesis window
   e. Overlap-add (50% overlap with previous block)

6. Synthesis filter bank: 64-band QMF synthesis
   For each output sample:
     y[n] = Σ(k=0 to 63) IMDCT_out[k][n mod 256] × h_synthesis[k][n]

7. Apply metadata:
   ├── Dialog normalization: apply gain = dialnorm (typically −31 dB)
   ├── Dynamic range compression: apply compr/dynrnge scaling
   └── Downmix if output has fewer channels than encoded

8. Format output:
   ├── Apply clipping protection (sample rate × gain)
   └── Output as interleaved PCM or planar PCM
```

### 5.3 Error Concealment
- **Corrupt frame detection:** CRC-16 mismatch on frame data; sync word loss
- **Concealment method:** Mute entire frame on CRC error (or repeat previous frame with gain reduction)
- **Maximum consecutive errors before silence:** Implementation-defined; typically 3–5 frames
- **DSI (Dolby Surround) flags:** Bitstream can signal Lt/Rt vs Lo/Ro for proper matrix decoding

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** None — AC-3 is a bare bitstream (elementary stream)
- **Overhead:** ~2–4 bytes per frame (sync word + CRC); negligible
- **Seeking in native container:** Not supported; requires container-level index
- **Multiple streams in native container:** Not in AC-3 itself; containers multiplex multiple programs

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| MPEG-2 TS | Yes (PID type 0x81) | Yes (by PCR/PTS) | Via PMT/PES headers | ATSC/DVB broadcast |
| MPEG-2 PS | Yes (stream type 0x80/0x81) | Limited | Via DVD IFO | DVD-Video |
| Matroska/MKV | Yes (codec ID: A_AC3) | Yes (via cues) | Full Vorbis-style tags | |
| MP4/M4A | No (use E-AC-3) | — | — | AC-3 not in MP4 spec; use E-AC-3 |
| OGG | No | — | — | OGG supports Vorbis, FLAC, Opus only |
| WAV | No | — | — | PCM only |
| AIFF | No | — | — | PCM only |
| WebM | No | — | — | Uses Opus, Vorbis |
| Blu-ray (m2ts) | Yes (PCR stream) | Yes (via timestamps) | Limited | Primary audio on many Blu-rays |
| DVD-VOB | Yes | Yes (via IFO) | Limited | Maximum 448 kbps |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** AC-3 has no native tagging system; metadata is encoded in the bitstream itself
- **Metadata block location:** Within each sync frame's BSI (Bit Stream Information) and per-block fields
- **User-accessible metadata:** Limited to bitstream metadata (not arbitrary tags)

### 7.2 Standard Tag Fields — Bitstream Metadata Reference
| Field | BSI Location | Encoding | Range | Description |
|-------|-------------|----------|-------|-------------|
| Dialog Normalization (dialnorm) | Frame BSI | uint4 | 0–31 | Dialogue level in −dB (31 = −31 dBFS) |
| Compression (compr) | Frame BSI | uint8 | 0–255 | Dynamic range compression strength |
| Heavy Compression | Per block | uint8 | 0–255 | Frame-level heavy compression override |
| Language Code | Optional | uint8[3] | ISO 639-2 | Audio service language |
| Audio Production Info | Optional | bit field | — | Production type (mixed/separated) |
| Room Type | Optional | uint2 | 0–3 | Studio characteristics |
| Copyright Bit | Optional | bit | 0/1 | Copyright status |
| Original Bit | Optional | bit | 0/1 | Original recording flag |

### 7.3 Dialog Normalization & Dynamic Range Control
```
Dialog Normalization (DIALNORM):
  - Encodes the dialogue level of the program to normalize playback volume
  - Range: 0–31 representing −1 dBFS to −31 dBFS
  - Decoder applies gain correction: gain = dialnorm (typical −31 dB default)
  - User can disable or adjust DRC in decoder

Dynamic Range Compression (DYNRNG/COMPR):
  - Two compression profiles:
    Light Compression: DYNRNG applied at block level, gentle peak limiting
    Heavy Compression: COMPR applied frame-wide, dramatic dynamic range reduction
  - For TV viewing at low volumes, heavy compression preserves dialogue clarity
  - For home theater at reference volume, full dynamic range is desired
  - Flags transmitted in BSI: cplis, cplle, peleov, edhinf
```

### 7.4 Metadata Compatibility Matrix
AC-3 does not support arbitrary metadata tags. Metadata is exclusively in the bitstream.
| Tag System | Read | Write | Notes |
|------------|------|-------|-------|
| ID3v1 | N/A | N/A | Not applicable — AC-3 is a transport codec |
| ID3v2 | N/A | N/A | Not applicable |
| Vorbis Comments | N/A | N/A | Not applicable |
| AC-3 Bitstream | N/A | N/A | Proprietary metadata only |

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   ac3                # AC-3
                    eac3               # E-AC-3 (Dolby Digital Plus)
AV_CODEC_ID:        AV_CODEC_ID_AC3    # (in libavcodec/codec_id.h)
                    AV_CODEC_ID_EAC3
Format Name (CLI):  ac3 (raw bitstream) # used with -f ac3
Demuxer(s):         ac3, eac3          # auto-detects based on sync word
Muxer(s):           ac3                # raw AC-3 elementary stream
Encoder(s):         ac3, ac3_fixed     # FFmpeg's native AC-3 encoder
Decoder(s):         ac3, eac3          # FFmpeg's native AC-3/E-AC-3 decoder
```

### 8.2 FFmpeg Encoding — Full CLI Reference

```bash
# Basic AC-3 encoding — stereo at 192 kbps
ffmpeg -i input.wav \
  -c:a ac3 \
  -b:a 192k \
  -ac 2 \
  output.ac3

# AC-3 encoding — 5.1 surround at 448 kbps (DVD max)
ffmpeg -i input.wav \
  -c:a ac3 \
  -b:a 448k \
  -ac 6 \
  -channel_layout 5.1 \
  output.ac3

# AC-3 encoding — 5.1 at 640 kbps (Blu-ray max)
ffmpeg -i input.wav \
  -c:a ac3 \
  -b:a 640k \
  -ac 6 \
  -channel_layout 5.1 \
  output.ac3

# E-AC-3 (Dolby Digital Plus) encoding
ffmpeg -i input.wav \
  -c:a eac3 \
  -b:a 256k \
  -ac 6 \
  output.eac3

# Fixed-point AC-3 encoder (for embedded/telephony hardware)
ffmpeg -i input.wav \
  -c:a ac3_fixed \
  -b:a 128k \
  -ac 2 \
  output.ac3

# AC-3 with stereo rematrixing disabled (for testing)
ffmpeg -i input.wav \
  -c:a ac3 \
  -b:a 256k \
  -stereo_rematrixing 0 \
  output.ac3

# AC-3 with custom coupling start band
ffmpeg -i input.wav \
  -c:a ac3 \
  -b:a 384k \
  -channel_coupling 1 \
  -cpl_start_band 8 \
  output.ac3
```

#### Complete FFmpeg AC-3 Encoder Option Table
| Option | Type | Default | Range / Valid Values | Effect on Output |
|--------|------|---------|---------------------|-----------------|
| `-b:a` | int | 128k | 32k–640k | Target bitrate (CBR only) |
| `-ac` | int | auto | 1–6 | Output channel count |
| `-ar` | int | auto | 32000, 44100, 48000 | Output sample rate |
| `-stereo_rematrixing` | bool | 1 | 0, 1 | Enable Lt/Rt encoding for stereo |
| `-channel_coupling` | bool | auto | -1 (auto), 0, 1 | Enable coupling |
| `-cpl_start_band` | int | auto | 1–15 | Coupling start frequency band |
| `-bitstream_size` | — | — | — | [NEEDS VERIFICATION] |

**Note:** FFmpeg's AC-3 encoder does NOT support VBR. The `-vbr` and `-qscale:a` options are ignored. Bitrate is the only quality control mechanism.

### 8.3 FFmpeg Encoding — libavcodec C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/opt.h>
#include <libswresample/swresample.h>

// ─── 1. Find AC-3 encoder ─────────────────────────────────────────────────────
const AVCodec *codec = avcodec_find_encoder(AV_CODEC_ID_AC3);
// Alternative: avcodec_find_encoder_by_name("ac3") or ("ac3_fixed")
if (!codec) { fprintf(stderr, "AC-3 encoder not found\n"); exit(1); }

// ─── 2. Allocate codec context ───────────────────────────────────────────────
AVCodecContext *ctx = avcodec_alloc_context3(codec);
if (!ctx) { fprintf(stderr, "OOM\n"); exit(1); }

// ─── 3. Set required parameters ───────────────────────────────────────────────
ctx->bit_rate     = 448000;                        // 448 kbps for DVD-quality 5.1
ctx->sample_fmt   = AV_SAMPLE_FMT_FLTP;            // REQUIRED: planar float
ctx->sample_rate  = 48000;                         // Standard rate
av_channel_layout_default(&ctx->ch_layout, 6);    // 5.1 channels
// ctx->ch_layout = (AVChannelLayout)AV_CHANNEL_LAYOUT_5POINT1;

// ─── 4. Open codec ───────────────────────────────────────────────────────────
int ret = avcodec_open2(ctx, codec, NULL);
if (ret < 0) {
    char errbuf[128]; av_strerror(ret, errbuf, sizeof(errbuf));
    fprintf(stderr, "avcodec_open2 failed: %s\n", errbuf);
    exit(1);
}

// ─── 5. Allocate frame and packet ───────────────────────────────────────────
AVFrame *frame = av_frame_alloc();
frame->format      = ctx->sample_fmt;
frame->sample_rate = ctx->sample_rate;
av_channel_layout_copy(&frame->ch_layout, &ctx->ch_layout);
frame->nb_samples  = ctx->frame_size;  // AC-3: ctx->frame_size = 1536
av_frame_get_buffer(frame, 0);

AVPacket *pkt = av_packet_alloc();

// ─── 6. Encode loop ──────────────────────────────────────────────────────────
void encode_frame(AVCodecContext *ctx, AVFrame *frame, AVPacket *pkt, FILE *out) {
    int ret = avcodec_send_frame(ctx, frame);
    if (ret < 0) { fprintf(stderr, "send error\n"); exit(1); }

    while (ret >= 0) {
        ret = avcodec_receive_packet(ctx, pkt);
        if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF) return;
        if (ret < 0) { fprintf(stderr, "encode error\n"); exit(1); }
        fwrite(pkt->data, 1, pkt->size, out);
        av_packet_unref(pkt);
    }
}

// ─── 7. Flush encoder ────────────────────────────────────────────────────────
encode_frame(ctx, NULL, pkt, outfile);  // NULL frame triggers drain

// ─── 8. Cleanup ───────────────────────────────────────────────────────────────
av_frame_free(&frame);
av_packet_free(&pkt);
avcodec_free_context(&ctx);
```

**Critical Notes:**
- `ctx->frame_size`: Always 1536 for AC-3 (6 blocks × 256 samples). Set `frame->nb_samples` accordingly.
- `AV_SAMPLE_FMT_FLTP` is REQUIRED. AC-3 encoder only accepts planar 32-bit float. Use `libswresample` to convert from S16/S32 input.
- AC-3 encoder is CBR only. No VBR quality mode is available.

### 8.4 FFmpeg Decoding — Full CLI Reference

```bash
# Decode AC-3 to raw PCM WAV
ffmpeg -i input.ac3 \
  -c:a pcm_s16le \
  output.wav

# Decode and resample to 44.1 kHz
ffmpeg -i input.ac3 \
  -c:a pcm_s16le \
  -ar 44100 \
  output.wav

# Decode and downmix to stereo
ffmpeg -i input_5_1.ac3 \
  -c:a pcm_s16le \
  -ac 2 \
  output_stereo.wav

# Decode E-AC-3 to WAV
ffmpeg -i input.eac3 \
  -c:a pcm_s16le \
  output.wav

# Probe AC-3 file info
ffprobe -v quiet -print_format json -show_streams -show_format input.ac3 | jq .streams[0]

# Extract AC-3 elementary stream from container
ffmpeg -i input.mkv -c:a copy -map 0:a:0 output.ac3

# Apply heavy dynamic range compression during decode
ffmpeg -i input.ac3 -c:a pcm_s16le -drc_scale 1.0 output.wav

# Disable dynamic range compression (preserve full dynamic range)
ffmpeg -i input.ac3 -c:a pcm_s16le -drc_scale 0.0 output.wav
```

### 8.5 FFmpeg Decoding — libavcodec C API

```c
// ─── Complete decode program skeleton ────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.ac3", NULL, NULL);
avformat_find_stream_info(fmt_ctx, NULL);

// Find audio stream
int audio_idx = av_find_best_stream(fmt_ctx, AVMEDIA_TYPE_AUDIO, -1, -1, NULL, 0);
if (audio_idx < 0) { fprintf(stderr, "No audio stream found\n"); exit(1); }
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
        if (ret < 0) { av_packet_unref(pkt); continue; }

        while (avcodec_receive_frame(dec_ctx, frm) == 0) {
            // frm->data[0..channels-1] contain audio samples (planar float)
            // frm->nb_samples = sample count per channel (256 per block, up to 1536 per frame)
            // frm->format = AVSampleFormat (FLTP for AC-3)
            // frm->sample_rate = 48000
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

AC-3 bitstream metadata is read/written through private options:

```bash
# Read AC-3 bitstream metadata
ffprobe -v quiet -print_format json -show_format input.ac3 | jq .format.tags

# AC-3 bitstream metadata fields (accessed via FFmpeg private options):
# - dialnorm: Dialog normalization (0–31, typically 31)
# - drc_scale: Dynamic range control scale (0.0–1.0, default 1.0)

# Encode with custom dialog normalization
ffmpeg -i input.wav -c:a ac3 -b:a 384k \
  -dialnorm 27 \
  output.ac3

# Strip all metadata
ffmpeg -i input.ac3 -c:a ac3 -b:a 384k -map_metadata -1 output.ac3

# Embed cover art (not native to AC-3; use container format)
ffmpeg -i input.ac3 -i cover.jpg -c copy -map 0:a -map 1:v output.mkv
```

### 8.7 Quality / Fidelity Decision Table
| Use Case | Settings | Expected Size (min) | Notes |
|----------|----------|---------------------|-------|
| Archival / lossy | `-c:a ac3 -b:a 640k -ar 48000 -ac 6` | ~2.5 MB/min (5.1) | Transparent for most content |
| Audiophile lossy | `-c:a ac3 -b:a 384k -ar 48000 -ac 6` | ~1.5 MB/min (5.1) | Near-transparent; DVD standard |
| Streaming (high) | `-c:a ac3 -b:a 448k -ar 48000 -ac 6` | ~1.75 MB/min | DVD-Video standard |
| Stereo music | `-c:a ac3 -b:a 256k -ar 48000 -ac 2` | ~1 MB/min | Good quality stereo |
| Podcast / voice | `-c:a ac3 -b:a 64k -ar 32000 -ac 1` | ~0.25 MB/min | Mono, low bandwidth |
| Mobile / low bandwidth | `-c:a ac3 -b:a 96k -ar 32000 -ac 1` | ~0.4 MB/min | Voice-optimized |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
```
AC-3 has no native seek table. Seeking is handled by:
  - Container-level index (MPEG-TS PSI, Matroska cues, MKV seek points)
  - CBR byte offset calculation for raw AC-3 elementary streams

CBR seek formula:
  byte_offset = floor(target_time_seconds × bitrate_bps / 8)
  frame_number = floor(target_time_seconds / frame_duration)
  byte_offset = frame_number × frame_size_bytes
```

### 9.2 Gapless Playback Data
```
Encoder delay:   256 samples (48 kHz) = 5.33 ms (added at start of first frame)
Padding:        256 samples (48 kHz) = 5.33 ms (added at end of last frame)
Total delay:    512 samples = 10.67 ms

Storage location: Not stored in AC-3 bitstream.
                   Managed by container (e.g., MP4 edit list, Matroska cues)

FFmpeg handling:
  - Decoder automatically skips encoder delay (first 256 samples)
  - User must manually trim padding samples based on frame count

Gapless command:
ffmpeg -i input.wav -c:a ac3 -b:a 384k output.ac3
# The output.ac3 contains 256-sample delay at start and 256-sample padding at end
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | Frame-based; can decode after first frame (32 ms) |
| Algorithmic encoder delay | 256 samples / 5.33 ms | Per block; frame = 6 × 5.33 = 32 ms |
| Live encoding feasible | Yes | Low-latency mode possible with reduced frame size |
| HTTP progressive download | Yes | Via containers (MKV, M2TS) or raw AC-3 |
| HTTP Live Streaming (HLS) | Yes | Via MPEG-TS segments (ATSC standard) |
| DASH streaming | Yes | Via MPEG-TS or MP4 (E-AC-3) segments |
| WebRTC / RTP transport | Yes | RFC 4184 defines RTP payload for AC-3 |
| Minimum decode buffer | 1 frame / 32 ms | At 48 kHz |

**RTP Payload (RFC 4184):**
```
AC-3 RTP payload format:
  - Each RTP packet starts with 2-byte payload header
  - Payload header: 2 bytes containing frame fragment info
  - AC-3 frames may be fragmented across multiple RTP packets
  - MTU recommendation: 576 bytes (to avoid fragmentation)
```

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | Layout Name | Channel Order | FFmpeg Layout Constant | acmod |
|----------|-------------|---------------|-----------------------|-------|
| 1 | Mono | C | AV_CHANNEL_LAYOUT_MONO | 1 |
| 2 | Stereo | L, R | AV_CHANNEL_LAYOUT_STEREO | 2 |
| 2 | Dual Mono | A, B | — | 0 |
| 2 | Lt/Rt (ProLogic) | Lt, Rt | — | 2 (with rematrixing) |
| 3 | 3.0 | L, C, R | — | 3 |
| 4 | 4.0 | L, R, SL, SR | AV_CHANNEL_LAYOUT_QUAD | 6 |
| 5 | 5.0 | FL, FR, C, SL, SR | AV_CHANNEL_LAYOUT_5POINT0 | 7 (lfeon=0) |
| 6 | 5.1 | FL, FR, C, LFE, SL, SR | AV_CHANNEL_LAYOUT_5POINT1 | 7 (lfeon=1) |
| 6 | 4.1 | FL, FR, SL, SR, LFE | — | 6 (lfeon=1) |

**Notes on acmod:**
- acmod=0: 1+1 mode — two independent mono channels (language A and B)
- acmod=1: 1/0 mode — single mono channel
- acmod=2: 2/0 mode — stereo (L, R)
- acmod=3: 3/0 mode — L, C, R
- acmod=4: 2/1 mode — L, R, S (single surround)
- acmod=5: 3/1 mode — L, C, R, S
- acmod=6: 2/2 mode — L, R, SL, SR
- acmod=7: 3/2 mode — L, C, R, SL, SR (+ LFE optionally)

### 11.2 Downmix Coefficients (5.1 → Stereo)
```
Standard AC-3 downmix (Lt/Rt compatibility):
  L_out = L_in + 0.7071 × C_in + 0.7071 × SL_in
  R_out = R_in + 0.7071 × C_in + 0.7071 × SR_in
  LFE: discarded

FFmpeg downmix command (using pan filter for custom coefficients):
ffmpeg -i input_5.1.ac3 -c:a pcm_s16le \
  -af "pan=stereo|FL=FC+0.707*FC+0.707*BL|FR=FR+0.707*FC+0.707*BR" \
  output_stereo.wav

FFmpeg auto-downmix (default coefficients):
ffmpeg -i input_5.1.ac3 -c:a pcm_s16le -ac 2 output_stereo.wav
  # Uses standard Lt/Rt downmix coefficients
```

### 11.3 Matrix Encoding (Dolby Surround / Lt/Rt)
```
Dolby Surround matrix encoding:
  Lt = L + 0.707 × C + 0.707 × SL
  Rt = R + 0.707 × C + 0.707 × SR

AC-3 bitstream signals:
  - Rematrix flags per frequency band for stereo input
  - Decoder uses Lt/Rt → Lo/Ro matrix for ProLogic decoding
  - bsmod = 2 indicates decoded Lt/Rt is surround-compatible
```

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 24-bit input | Internal precision varies by encoder |
| Max sample rate | 48 kHz | AC-3; E-AC-3 supports 96 kHz (via downsampling) |
| Float support | 32-bit IEEE float | FFmpeg float path |
| DSD support | No | DSD → PCM required before encoding |
| 20-bit support | Yes | Broadcast / professional formats |
| 24-bit support | Yes | High-resolution input, not broadcast |

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

| Platform / API | Encode | Decode | FFmpeg Flag | Notes |
|----------------|--------|--------|-------------|-------|
| NVIDIA NVENC | No | — | — | NVENC does not support AC-3 |
| NVIDIA NVDEC | — | Yes | `-hwaccel cuda` | GPU decode for video pipelines |
| Intel QSV | No | Yes | `-hwaccel qsv` | Via Quick Sync Video |
| Apple AudioToolbox | Yes | Yes | `-c:a ac3_at` | Hardware AC-3 encode/decode on macOS/iOS |
| Android MediaCodec | Partial | Yes | `-c:a ac3_mediacodec` | Encoder optional, decoder required |
| VA-API (Linux) | No | Yes | `-hwaccel vaapi` | Decode via VA-API |
| ARM NEON | No native | Via libav | — | Software decode optimized |
| DSP/Embedded | No | Yes | — | Many SoCs have AC-3 decoder hardware |

```bash
# macOS AudioToolbox AC-3 encoding example
ffmpeg -i input.wav -c:a ac3_at -b:a 448k output.ac3

# Build FFmpeg with AudioToolbox support:
./configure --enable-audiotoolbox --enable-encoder=ac3

# Android MediaCodec decode
ffmpeg -i input.ac3 -c:a ac3_mediacodec output.wav
```

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Encoder-Specific Issues
| Issue | FFmpeg Versions Affected | Workaround |
|-------|------------------------|------------|
| AC-3 encoder quality lower than reference encoders (Dolby, Surcode) | All versions | Use reference encoder for production; FFmpeg encoder acceptable for testing |
| AC-3 encoder produces slight high-frequency artifacts at very low bitrates (<64 kbps) | All versions | Use higher bitrate or E-AC-3 at low bitrates |
| `ac3_fixed` encoder produces different output than `ac3` | All versions | Use `ac3` (floating point) for best quality; `ac3_fixed` for embedded/telephony |
| Encoder ignores `-qscale:a` and `-vbr` | All versions | Only `-b:a` bitrate is supported |
| Bit-exact output not guaranteed across FFmpeg versions | All versions | Test encoder output if bit-exactness is critical |
| E-AC-3 encoder does not support all E-AC-3 features | All versions | Full E-AC-3 encoding requires Dolby's reference encoder |

### 14.2 Interoperability Issues
- **FFmpeg encoder → consumer AV receivers:** FFmpeg-encoded AC-3 is widely compatible but some receivers may have stricter compliance checks
- **Files with non-standard bitrates (e.g., 576 kbps):** Not all players handle non-standard bitrates
- **Dual mono (acmod=0):** Some AV receivers incorrectly play only channel A
- **E-AC-3 frames in AC-3 container:** Invalid; E-AC-3 (bsid=8) cannot be decoded as AC-3
- **AC-3 at 44.1 kHz:** Some older receivers do not support 44.1 kHz AC-3

### 14.3 Edge Cases to Handle in Converter
- **Empty file / 0-sample stream:** Return empty output; no error
- **File < 1 frame:** Encode minimum 1 frame (1536 samples); pad with silence if needed
- **All-silence audio:** AC-3 encoder handles gracefully; may use dither flags
- **DC offset (non-zero mean):** Encoder applies high-pass filter; no special handling needed
- **Full-scale sine (0 dB):** Output samples may clip; encoder does not apply makeup gain
- **File with corrupt header:** Return error; do not produce partial output
- **Truncated file:** Decode up to last complete frame; report frame count mismatch
- **Sample rate not supported by codec:** Resample to 48 kHz before encoding
- **Channel count not supported:** Downmix to stereo or mono as appropriate

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM AC-3

| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → FLAC | `ffmpeg -i in.ac3 -c:a flac -compression_level 8 out.flac` | N/A (no AC-3 tags) | Lossless decode; ReplayGain scan needed |
| → ALAC | `ffmpeg -i in.ac3 -c:a alac out.m4a` | N/A | Lossless decode |
| → MP3 | `ffmpeg -i in.ac3 -c:a libmp3lame -q:a 0 out.mp3` | No | Generation loss (lossy→lossy) |
| → AAC | `ffmpeg -i in.ac3 -c:a aac -b:a 256k out.m4a` | No | Generation loss |
| → Opus | `ffmpeg -i in.ac3 -c:a libopus -b:a 128k out.opus` | No | Generation loss |
| → WAV | `ffmpeg -i in.ac3 -c:a pcm_s16le out.wav` | No | Lossless decode |
| → OGG Vorbis | `ffmpeg -i in.ac3 -c:a libvorbis -q:a 6 out.ogg` | No | Generation loss |

### 15.2 Converting TO AC-3

| Source | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| FLAC → | `ffmpeg -i in.flac -c:a ac3 -b:a 448k -ar 48000 out.ac3` | No (AC-3 has no tags) | Generation loss |
| WAV → | `ffmpeg -i in.wav -c:a ac3 -b:a 448k out.ac3` | No | Encode from PCM |
| MP3 → | `ffmpeg -i in.mp3 -c:a ac3 -b:a 256k out.ac3` | ID3 → lost | Double lossy encoding |
| AAC → | `ffmpeg -i in.m4a -c:a ac3 -b:a 384k out.ac3` | No | Generation loss |
| Vorbis → | `ffmpeg -i in.ogg -c:a ac3 -b:a 384k out.ac3` | No | Generation loss |

### 15.3 Lossless Round-Trip Verification
AC-3 is lossy; no lossless round-trip is possible.
```bash
# Verify decoded quality vs original
ffmpeg -i original.wav -c:a pcm_s16le original_raw.wav
ffmpeg -i original.wav -c:a ac3 -b:a 640k -ar 48000 out.ac3
ffmpeg -i out.ac3 -c:a pcm_s16le decoded.wav

# Compare (there will be differences — AC-3 is lossy)
md5sum original_raw.wav decoded.wav

# Use EBU R128 for loudness comparison
ffmpeg -i original.wav -af loudnorm=print_format=json -f null - 2>&1 | jq .input_i
ffmpeg -i out.ac3 -af loudnorm=print_format=json -f null - 2>&1 | jq .input_i
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| Dolby Reference Encoder | C/C++ | Proprietary | 10/10 (reference) | — | dolby.com (licensed) |
| FFmpeg (libavcodec) | C | LGPL 2.1+ | 7/10 | 9/10 | https://ffmpeg.org |
| liba52 | C | GPL | 6/10 | 8/10 | Historical; largely replaced by FFmpeg |
| A52DEC | C | GPL | — | 8/10 | Historical decoder |
| Surcode DVD (Dolby) | Proprietary | Commercial | 9/10 | — | Licensed software |
| Sonic (DTS/Audio) | C++ | Proprietary | — | Various | Historical |

### Build Instructions
```bash
# Build FFmpeg with AC-3 support (AC-3 encoder is built-in, no external lib needed)
git clone https://github.com/FFmpeg/FFmpeg.git
cd FFmpeg
./configure --enable Encoder=ac3,ac3_fixed --enable-decoder=ac3,eac3
make -j$(nproc)
sudo make install

# Verify encoder is available
./ffmpeg -encoders | grep -E "ac3|eac3"
# Should show:  A..D ac3             ATSC A/52
#              A..D ac3_fixed       ATSC A/52 (fixed-point)
#              A..D eac3            ATSC A/52 (E-AC-3)
```

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **ATSC A/52:2018:** Digital Audio Compression (AC-3) (E-AC-3) Standard — https://www.atsc.org/wp-content/uploads/2021/04/A52-2018.pdf
- **ETSI TS 102 366:** Digital Audio Compression (AC-3, E-AC-3); Transcoding — https://www.etsi.org/deliver/etsi_ts/102300_102399/102366/
- **RFC 4184:** RTP Payload Format for AC-3 Audio — https://www.rfc-editor.org/rfc/rfc4184
- **RFC 4598:** RTP Payload Format for E-AC-3 Audio — https://datatracker.ietf.org/doc/rfc4598/

### Technical Resources
- FFmpeg AC-3 encoder options: `ffmpeg -h encoder=ac3`
- FFmpeg E-AC-3 encoder options: `ffmpeg -h encoder=eac3`
- Multimedia Wiki AC-3: https://wiki.multimedia.cx/index.php/AC-3
- Multimedia Wiki E-AC-3: https://wiki.multimedia.cx/index.php/E-AC-3
- Hydrogenaudio AC-3: https://hydrogenaudio.org/wiki/AC-3
- Dolby Digital Plus whitepaper: https://professional.dolby.com/globalassets/dolby-digital-plus/aes-convention-paper-intro-to-dolby-digital-plus.pdf

### Academic Papers
- Craig B. F. Todd et al., "AC-3: Flexible Perceptual Coding for Audio Transmission and Storage," AES 96th Convention, 1994 — https://www.mp3-tech.org/programmer/docs/ac3-flex.pdf
- Stanley P. Lipshitz and John Vanderkooy, "The ACOUSTICS of AC-3," AES 101st Convention, 1996
- Grant Davidson et al., "Vector Quantization of Audio Spectral Coefficients," AES 88th Convention, 1990

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] Identify FFmpeg `./configure` flags needed (AC-3 is built-in; no external library needed)
- [ ] Verify `ffmpeg -encoders` output confirms ac3 and eac3 encoders are available
- [ ] Verify `ffmpeg -decoders` output confirms ac3 and eac3 decoders are available
- [ ] No external library dependency for AC-3; E-AC-3 is also built-in
- [ ] Handle platform licensing (Dolby patents — commercial licensing may be needed for distribution)

### Encoding Pipeline
- [ ] Convert input sample format to `AV_SAMPLE_FMT_FLTP` using libswresample
- [ ] AC-3 has fixed frame size of 1536 samples; set `frame->nb_samples = 1536`
- [ ] Implement proper flush/drain at end of stream (send NULL frame)
- [ ] Validate that input sample rate is in {32000, 44100, 48000} — resample if not
- [ ] Validate that channel layout is supported; auto-downmix if needed
- [ ] Note: No seek table support in raw AC-3; containers (MKV, M2TS) handle seeking

### Decoding Pipeline
- [ ] Implement sync word search (0x0B77) for stream recovery
- [ ] Handle AVERROR(EAGAIN) in receive_frame loop correctly
- [ ] Flush decoder at EOF (send NULL packet)
- [ ] AC-3 decoder delay is 256 samples; skip at stream start (managed by FFmpeg)
- [ ] Handle sample format conversion from FLTP to target format (S16/S32)
- [ ] Respect DRC metadata from bitstream (dialnorm, dynrnge, compr)

### Metadata
- [ ] AC-3 has no user-metadata tag system; metadata is in bitstream only
- [ ] Read dialog normalization level from bitstream
- [ ] Read DRC/compression settings from bitstream
- [ ] Apply or bypass DRC based on user preference
- [ ] Container-level tags are not preserved through AC-3 encode/decode

### Quality & Verification
- [ ] Verify decoded audio quality meets expectations for target bitrate
- [ ] Test with silence, full-scale sine, transient-rich, dialogue-heavy content
- [ ] Verify frame size is constant (CBR) within ±1 byte tolerance
- [ ] Verify sample rate is correctly decoded from fscod field
- [ ] Verify channel count and LFE flag are correctly decoded

### Edge Cases
- [ ] Handle files with corrupt or missing sync words (search forward for next 0x0B77)
- [ ] Handle files with 0 samples (produce empty output)
- [ ] Handle sample rate mismatch (resample input to 32/44.1/48 kHz)
- [ ] Handle channel count mismatch (downmix via pan filter or auto-downmix)
- [ ] Handle very short files (< 1 frame) — pad with silence to 1536 samples

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
