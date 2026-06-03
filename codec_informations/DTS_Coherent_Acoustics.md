# DTS Coherent Acoustics — Deep Technical Reference
> **Category:** Lossy (core), Lossless (DTS-HD MA extension)
> **File Extensions:** `.dts`, `.dtshd`, `.dtsma`, `.dtshr`
> **MIME Types:** `audio/vnd.dts`, `audio/x-dts`
> **Standardization Body:** ETSI (European Telecommunications Standards Institute)
> **Primary Specification:** ETSI TS 102 114 (all versions of DTS Coherent Acoustics including DTS, DTS-ES, DTS 96/24, DTS-HD)
> **Patent Status:** Patented — royalty-bearing (DTS, Inc. licensing)
> **License:** Proprietary — licensed through DTS, Inc.
> **Current Version:** ETSI TS 102 114 V1.6.1 (2019-08)
> **Active Development:** Limited — DTS-HD MA is stable; focus shifted to DTS:X (object-based)

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** DTS (Digital Theater Systems), Inc. — founded by Terry Beard, Terry H. Joy
- **Year Created:** 1993 (original DTS Coherent Acoustics / DTS Digital Surround)
- **Original Purpose:** Provide high-quality multi-channel digital audio for cinema (first deployed in Jurassic Park, 1993) as an alternative to Dolby Digital (AC-3) on laserdisc and later DVD/Blu-ray
- **Problem with Predecessors:** Analog optical soundtracks on film were limited in dynamic range and frequency response. Dolby's AC-3 (1991) was already deployed in cinemas. DTS differentiated with higher bitrate and a perceived quality advantage at equivalent channel counts.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| DTS Coherent Acoustics 1.0 | 1993 | Initial cinema release; 5.1 support, 1509.75 kbps |
| DTS-ES | 2000 | Added 6.1 channels (Matrix and Discrete variants) |
| DTS 96/24 | 2001 | 96 kHz / 24-bit support for 5.1 channels |
| DTS-HD High Resolution | 2005 | XBR extension; higher constant bitrates up to 6 Mbps |
| DTS-HD Master Audio (XLL) | 2005 | Lossless core + extension; up to 7.1 at 192 kHz |
| DTS:X | 2015 | Object-based audio; up to 32 channels; backward compatible via DTS-HD MA core |
| DTS:X Pro | 2019 | Extended DTS:X with more speaker configurations |

### 1.3 Current Adoption
- **Primary use cases today:** Blu-ray Disc (primary lossless audio alongside Dolby TrueHD), DVD-Video (legacy DTS audio tracks), cinema, streaming (some services), CD (raw DTS core at 1411.2 kbps)
- **Platforms with native support:** Windows, macOS, iOS, Android, Linux (via FFmpeg/libav), game consoles, AV receivers, Blu-ray players, set-top boxes
- **Major services using this format:** Cinema (DTS:X), some streaming services, legacy Blu-ray audio
- **Hardware support:** Universal in AV receivers, home theater systems, Blu-ray players, game consoles (PlayStation, Xbox), car audio
- **Status:** Declining for new content — DTS:X object audio has limited adoption vs. Dolby Atmos; DTS-HD MA remains dominant on Blu-ray alongside Dolby TrueHD

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Perceptual / transform / hybrid subband
- **Core algorithm:** QMF (Quadrature Mirror Filter) subband decomposition + ADPCM (Adaptive Differential PCM) in core; modified DCT for transform; extension layers for higher resolution
- **Loss mechanism:** Perceptual masking (core); lossless coding (DTS-HD MA XLL); ADPCM residual coding (core)
- **Frame-based vs sample-based:** Frame-based. Frame contains 1–32 PCM blocks (variable).
- **Fixed vs variable frame size:** Variable. Core frame duration: 512, 1024, or 2048 PCM samples depending on mode.

### 2.2 High-Level Encoding Flow (Block Diagram)
```
Input PCM Samples (8–192 kHz, up to 24-bit, up to 8 channels)
      │
      ▼
[Pre-processing: DC removal, high-pass filter]
      │
      ▼
[Core Path: 32-band QMF Analysis → ADPCM core coding]
      │
      ├──► Core bitstream (always present)
      │
      ▼
[Extension Path: Subtract decoded core from original → residual coding]
      │
      ├──► Extension bitstream (optional, appended to core)
      │
      ▼
[Combined Bitstream Packing]
      │
      ├──► SYNC word (32 bits)
      ├──► Frame header (N bits): FTYPE, NBLKS, FSIZE, AMODE, SFREQ, RATE, etc.
      ├──► Core audio data
      ├──► Extension audio data (XBR, XXCH, XLL, etc.)
      └──► CRC error detection
      │
      ▼
Output DTS Bitstream (synchronization frames)
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | Variable: 512–2048 samples depending on frame mode | Core frame duration varies |
| Frame size | Variable: 512, 1024, or 2048 samples per frame | See NBLKS field |
| Max channels | 8 (7.1 discrete) | Via XXCH extension |
| Max bit depth | 24-bit input (core); up to 24-bit output | Variable by format |
| Max sample rate | 192 kHz (DTS-HD MA, DTS 96/24) | Core: 48 kHz (44.1 kHz variant) |
| Bitrate range | 32 kbps – 24.5 Mbps (DTS-HD MA aggregate) | Varies by format |
| Complexity | O(N log N) per transform | Encoder significantly heavier than decoder |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes

```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       7F FE 80 01     —         Core sync word (big-endian 0x7FFE8001)
                                           Alternative encodings:
                                           0xFE7F0180 (little-endian / 16-bit words on IBM PC)
                                           0x1FFFE800 (14-bit big-endian packing)
                                           0xFF1F00E8 (14-bit little-endian packing)
```

#### Sync Word Variants (All Valid DTS Sync Words)
| Hex Value | Endianness | Storage Format | Notes |
|-----------|------------|----------------|-------|
| 0x7FFE8001 | Big-endian | 32-bit words | Standard DTS core; first 16 bits = 0x7FFE, second = 0x8001 |
| 0xFE7F0180 | Little-endian | 16-bit words on little-endian CPU | Same bits, reversed byte order |
| 0x1FFFE800 | 14-bit big-endian | 14-bit packed words | Reduced dynamic range (14-bit) format |
| 0xFF1F00E8 | 14-bit little-endian | 14-bit packed words | 14-bit format, little-endian |

**Extended sync word validation:** After SYNC word (0x7FFE8001), the next 6 bits (FTYPE + SHORT fields) should equal `0b111111` (0x3F) for normal frames. This forms a 38-bit extended sync word: `0x7FFE8001_3F`.

#### Extension Substream Sync Word
| Sync Word | Hex Value | Purpose |
|-----------|-----------|---------|
| Core substream | 0x7FFE8001 | Primary audio frame |
| Extension substream | 0x02B9261 | Core frame in extension substream |

### 3.2 Frame Header Layout (DTS Core)
```
Bit Offset   Bit Width   Field Name              Type      Description
----------   ---------   --------------------    ------    ---------------------------
0            32          SYNC                   uint32    Must be 0x7FFE8001 (big-endian)
32           1           FTYPE                  uint1     Frame type: 0=normal, 1=termination
33           5           SHORT                  uint5     Short block indicator
38           1           NBLKS                  uint1     Number of PCM blocks - 1
39           7           SAMPLES                uint7     Samples per channel in frame - 1
46           14          FSIZE                  uint14    Frame size in bytes - 1
60           6           AMODE                  uint6     Audio channel mode (see table)
66           4           SFREQ                  uint4     Sample frequency (see table)
70           1           RATE                   uint1     Bitrate modifier flag
71           3           DYNF                   uint1     Dynamic range flag
72           1           TIMEF                  uint1     Timestamp flag
73           1           AUXF                   uint1     Auxiliary data flag
74           3           HDCD                   uint3     High Definition CD flag
77           1           EXT_AUDIO_ID           uint1     Extension audio present flag
78           3           EXT_AUDIO_ID           uint3     Extension audio ID
81           1           EXT_AUDIO              uint1     Extension audio frame flag
82           4           VERNUM                 uint4     Encoder software version
86           2           CHIST                  uint2     Channel history
89           3           PCMR                   uint3     PCM source bit depth - 1
92           1           SUMF                   uint1     Sum/difference flag
93           1           SUMS                   uint1     Sum/difference all channels
94           4           DIALNORM               uint4     Dialog normalization (ver-dependent)
98           4           DNG                    uint4     Dialogue normalization gain
```

### 3.3 Frame Size (FSIZE) vs Bitrate for Core at 48 kHz
| RATE Field | Bitrate Modifier | Frame Mode | Samples/Frame | Frame Duration | Bitrate Formula |
|------------|-----------------|-----------|---------------|----------------|----------------|
| 0b0 | Standard | Normal | 512 | 10.67 ms | See RATE table |
| 0b1 | High | Normal | 1024 | 21.33 ms | See RATE table |
| — | — | Short | 256 | 5.33 ms | 50% of normal |
| — | — | Termination | variable | variable | Marks frame boundary |

### 3.4 Sample Rate (SFREQ) Table
| SFREQ Value | Sample Rate (Hz) | Notes |
|-------------|-------------------|-------|
| 0x0         | 8 kHz             | Low-rate mode |
| 0x1         | 16 kHz            | Low-rate mode |
| 0x2         | 32 kHz            | Broadcast |
| 0x3         | 64 kHz            | [NEEDS VERIFICATION] |
| 0x4         | 128 kHz           | [NEEDS VERIFICATION] |
| 0x5         | 22.05 kHz         | CD-compatible |
| 0x6         | 44.1 kHz          | CD audio |
| 0x7         | 88.2 kHz          | 2× CD audio |
| 0x8         | 12 kHz            | Low-rate |
| 0x9         | 24 kHz            | Low-rate |
| 0xA         | 48 kHz            | Standard (DVD, Blu-ray) |
| 0xB         | 96 kHz            | High-resolution (DTS 96/24) |
| 0xC–0xF     | Reserved           | — |

### 3.5 Audio Mode (AMODE) Table — Channel Configurations
| AMODE Value | Channels | Channel Layout | Notes |
|-------------|----------|----------------|-------|
| 0x00        | 1 (A)    | Mono            | Single channel |
| 0x01        | 2 (A, B) | Dual mono       | Two independent mono channels |
| 0x02        | 2        | L, R            | Stereo |
| 0x03        | 2        | (L+R), (L−R)   | Sum and difference |
| 0x04        | 2        | Lt, Rt          | Matrix surround (ProLogic compatible) |
| 0x05        | 3        | C, L, R         | Front 3.0 |
| 0x06        | 4        | L, R, SL, SR   | Front + back surround |
| 0x07        | 4        | L, R, BL, BR   | Front + rear (different angle) |
| 0x08        | 5        | C, L, R, SL, SR| 5.0 surround |
| 0x09        | 6        | C, L, R, BL, BR, BC | DTS-ES Matrix 6.1 |
| 0x0A        | 6        | C, L, R, SL, SR, LFE | 5.1 surround |
| 0x0B        | 7        | DTS-ES Discrete 6.1 | C, L, R, SL, SR, BC, LFE |
| 0x0C–0x3F  | Reserved or extended | — | XXCH extends to 8 channels |

### 3.6 Bitrate (RATE parameter in header) vs Target Bitrate
| RATE Value | Target Bitrate (kbps) | Notes |
|------------|----------------------|-------|
| 0x00       | 32                  | Lowest |
| 0x01       | 56                  |       |
| 0x02       | 64                  |       |
| 0x03       | 96                  |       |
| 0x04       | 112                 |       |
| 0x05       | 128                 |       |
| 0x06       | 192                 |       |
| 0x07       | 224                 |       |
| 0x08       | 256                 |       |
| 0x09       | 320                 |       |
| 0x0A       | 384                 |       |
| 0x0B       | 448                 |       |
| 0x0C       | 512                 |       |
| 0x0D       | 576                 |       |
| 0x0E       | 640                 |       |
| 0x0F       | 768                 |       |
| 0x10       | 1024                |       |
| 0x11       | 1152                |       |
| 0x12       | 1280                |       |
| 0x13       | 1344                |       |
| 0x14       | 1408                |       |
| 0x15       | 1411.2              | CD audio (75 frames × 16 bits × 2 ch × 44.1 kHz) |
| 0x16       | 1472                |       |
| 0x17       | 1536                | DVD/Blu-ray standard |
| 0x18       | 1920                |       |
| 0x19       | 2048                |       |
| 0x1A       | 3072                |       |
| 0x1B       | 3840                |       |
| 0x1C       | open mode           | Variable |
| 0x1D       | variable mode       | Variable |
| 0x1E       | lossless mode       | DTS-HD MA |
| 0x1F       | reserved            |       |

**Note:** 1411.2 kbps is derived from CD audio: 44100 Hz × 16 bits × 2 channels = 1,411,200 bps. This is the maximum rate for raw DTS on CD.

### 3.7 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 16-bit | Signed integer | Yes | Standard for CD, DVD-Video DTS core |
| 20-bit | Signed integer | Yes | Professional formats |
| 24-bit | Signed integer | Yes | High-resolution, DTS-HD |
| 32-bit | IEEE float | Partial | Decoding only in modern implementations |
| 8-bit | Unsigned integer | No | Not supported |

#### Supported Sample Rates
| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 8000 | Telephone | Yes | Low-rate mode |
| 16000 | Wideband | Yes | Low-rate mode |
| 22050 | — | Yes | 0.5× CD rate |
| 32000 | Broadcast | Yes | ATSC/DVB broadcast |
| 44100 | CD audio | Yes | CD audio, DVD-Audio |
| 48000 | Professional | Yes | Standard for DVD, Blu-ray |
| 88200 | 2× CD | Yes | Via DTS 96/24 extension |
| 96000 | High-res | Yes | DTS 96/24, DTS-HD MA |
| 176400 | 4× CD | No | Not in standard DTS |
| 192000 | High-res max | Yes | DTS-HD MA 7.1 or 5.1 |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Pre-Processing Stage
- **DC offset removal:** High-pass filter applied to input audio
- **Input formatting:** Input PCM separated into core (≤48 kHz) and extension (>48 kHz for high-resolution modes)
- **Windowing:** For transform stages (when used), analysis windows are applied
- **Level normalization:** Managed via dialog normalization (DIALNORM) in bitstream header

### 4.2 Analysis / Transform Stage

#### Core: QMF Subband + ADPCM
```
Parameters:
  QMF bands:       32 subbands (for core coding)
  Core sample rate: 44.1 or 48 kHz (full rate)
  Extension rate:   Up to 192 kHz (split into core + extension)
  Transform:        None in core (subband ADPCM)
  For high-res:    Subband ADPCM in core + residual coding in extension
```

The core encoding uses a 32-band QMF analysis filter bank, followed by ADPCM within each subband:

```
For each of 32 subbands:
  1. Compute prediction from previous samples
  2. Calculate prediction error
  3. Quantize prediction error (2–6 bits depending on step size)
  4. Transmit quantized error + scale factor
  5. Decoder reconstructs by adding quantized error to prediction
```

#### High-Resolution Path (DTS 96/24, DTS-HD):
```
96 kHz / 24-bit input encoding:
  1. Low-pass filter input at 48 kHz → core signal
  2. Encode core signal at 48 kHz using standard 32-band QMF + ADPCM
  3. Decode core locally → reconstructed core
  4. Subtract reconstructed core from original → residual (difference) signal
  5. Encode residual using separate QMF + ADPCM path → extension data
  6. Pack both core and extension into single bitstream

Result:
  - Core decoder sees normal 48 kHz / 16–20-bit audio
  - Extension decoder reconstructs 48–96 kHz (difference) + 0–48 kHz (reconstruction error)
  - Combined output: 96 kHz / 24-bit
```

### 4.3 Core Bit Allocation

The core bit allocation uses ADPCM step sizes and quantization:

```
Bit allocation per subband:
  Step size: derived from scale factor + bit allocation parameter
  Quantization levels: 2^M where M = 1 to 6 bits per sample
  Lower subbands: Higher bit allocation (more bits for bass)
  Higher subbands: Lower bit allocation (reduced accuracy at high frequencies)

Scale factors:
  - Transmitted as 8-bit exponents per subband
  - Range: -127 to +127
  - Represents 6 dB per step
```

### 4.4 DTS-HD Extension Coding

DTS-HD uses a layered extension architecture:

```
Extension Types:
  XBR  (Extended Bit Rate):    Additional bitrate for core audio resolution
  XXCH (Extra Channels):       Additional channels beyond 5.1 (up to 7.1/8 channels)
  XCH  (Extended Channel):     Single additional channel (DTS-ES back surround)
  XLL  (Lossless):            Lossless coding of core audio
  XSA  (Extended Surround):   Additional surround information
  X96  (96 kHz Extension):    Extension for 96 kHz support in core

Extension Data Packing:
  - Extensions are appended after core audio in the same frame
  - Extensions have their own sub-headers
  - Older decoders ignore extensions (backward compatibility)
```

### 4.5 Bit Allocation Algorithm

```
DTS Core Bit Allocation (simplified):

1. Input: 32 subband QMF coefficients per audio sample
2. Compute scale factors for each subband (normalization)
3. Compute bit allocation per subband based on:
   - Perceptual masking model (DTS proprietary)
   - Bit budget (determined by RATE field)
   - Subband importance weighting
4. Allocate bits:
   - Core bass subbands (0–7): Higher allocation
   - Core mid subbands (8–15): Medium allocation
   - Core high subbands (16–31): Lower allocation
5. Quantize subband coefficients with allocated bits
6. Pack quantized data + scale factors + bit allocation info into frame
```

### 4.6 Quantization
- **Type:** Uniform symmetric quantization for ADPCM residuals
- **Step sizes:** Adaptive based on signal amplitude and bit allocation
- **Core quantization:** 2 to 6 bits per ADPCM sample per subband
- **Extension quantization:** Variable precision for residual signals

### 4.7 Stereo Encoding Modes
| Mode | AMODE | Description | Notes |
|------|-------|-------------|-------|
| Independent L/R | 2 | L and R encoded independently | Default |
| Sum/Difference | 3 | (L+R)/2, (L−R)/2 | Matrix encoding |
| Lt/Rt | 4 | ProLogic-compatible matrix | DTS Surround matrix |

### 4.8 Encoder Settings / Quality Modes

#### DTS Core Bitrates (5.1 channel)
| Quality Setting | Bitrate | Channels | Use Case |
|----------------|----------|----------|----------|
| Lowest | 32–96 kbps | 1–2 | Low bandwidth |
| Low | 128–256 kbps | 1–2 | Voice, low quality |
| Medium | 384–512 kbps | 5.1 | Low-bitrate 5.1 |
| Standard | 768 kbps | 5.1 | [NEEDS VERIFICATION] |
| High | 1411.2 kbps | 5.1 | CD audio DTS |
| Standard DVD/Blu-ray | 1509.75 kbps | 5.1 | DVD, Blu-ray core |
| Highest | 1536 kbps | 5.1 | [NEEDS VERIFICATION] |

**Notes on quality:**
- DTS typically requires higher bitrates than AC-3 for equivalent quality
- The 1509.75 kbps rate (48 kHz) is standard for DVD/Blu-ray core
- DTS-HD MA provides lossless quality at variable bitrates up to 24.5 Mbps
- Subjective quality comparisons vary; DTS often rated higher at same bitrate vs. AC-3

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. Scan bitstream for SYNC word: 0x7FFE8001 (big-endian, 32 bits)
   For 14-bit packing: scan for 0x1FFFE800 or 0xFF1F00E8
2. Validate candidate frame:
   a. Check FTYPE + SHORT fields = 0b111111 (for normal frames)
   b. Parse FSIZE → determine frame byte size
   c. Compute expected frame end position
   d. Verify next frame at computed offset has valid SYNC
   e. Check CRC-16 of frame data
3. If validation fails: advance 1 byte (or 2 bits for 14-bit) and retry
4. Max failed sync attempts before error: implementation-defined
```

**Bit ordering note:** DTS uses big-endian bit ordering within bytes. The SYNC word 0x7FFE8001 means byte[0]=0x7F, byte[1]=0xFE, byte[2]=0x80, byte[3]=0x01.

#### Seeking
- **CBR seeking (core):** `byte_offset = floor(target_time_sec × bitrate_bps / 8)`
- **VBR seeking (DTS-HD MA):** Requires frame scan; XLL frame size varies
- **Seek table format:** Container-specific (Blu-ray uses BUV index table)
- **Seek precision:** Frame-accurate (±10.67 ms at 48 kHz normal mode)

### 5.2 Core Decode Pipeline
```
1. Read SYNC word — 4 bytes (32 bits)
   ├── Verify SYNC = 0x7FFE8001
   ├── Parse FTYPE, SHORT → determine frame type
   └── Validate extended sync (FTYPE + SHORT = 0x3F for normal frames)

2. Parse frame header — variable length
   ├── Parse NBLKS → determine number of PCM blocks (NBLKS+1)
   ├── Parse SAMPLES → samples per channel (SAMPLES+1)
   ├── Parse FSIZE → frame size in bytes (FSIZE+1)
   ├── Parse AMODE → channel configuration
   ├── Parse SFREQ → sample rate
   ├── Parse RATE → bitrate index
   ├── Parse DYNF → dynamic range flag
   ├── Parse DIALNORM → dialogue normalization level
   └── Parse extension flags (EXT_AUDIO_ID, etc.)

3. Decode core audio (for each of NBLKS+1 blocks):
   a. Parse bit allocation data per subband
   b. Parse scale factors per subband
   c. Decode ADPCM residuals for 32 subbands
   d. Reconstruct subband samples from prediction + residual
   e. Apply QMF synthesis filter bank (inverse QMF)
   f. Output PCM samples for all channels

4. Decode extension data (if present and decoder supports):
   ├── XBR: decode additional resolution for core
   ├── XXCH: decode additional channels (up to 8 total)
   ├── XCH: decode DTS-ES back surround channel
   ├── XLL: decode lossless extension (bit-exact core reconstruction)
   └── X96: decode 96 kHz extension

5. Combine core + extension:
   ├── For DTS 96/24: core (0–48 kHz) + X96 (48–96 kHz difference)
   ├── For DTS-HD MA: core + XLL = lossless reconstruction
   └── For DTS-ES: core + XCH = 6.1/7.1 discrete

6. Apply metadata:
   ├── Dialogue normalization: gain adjustment
   ├── Dynamic range compression: apply if signaled
   └── Downmix: reduce channel count if needed

7. Format output:
   └── Interleaved or planar PCM (16/20/24-bit integer or float)
```

### 5.3 Error Concealment
- **Corrupt frame detection:** CRC-16 mismatch; SYNC word loss; invalid header fields
- **Concealment method:** Mute frame or repeat last frame with gain reduction
- **Maximum consecutive errors before silence:** Implementation-defined; typically 3 frames
- **DSYNC word:** Optional 0xFFFF marker in bitstream for bit error detection

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** None — DTS is a bare bitstream (elementary stream)
- **Overhead:** ~16–32 bytes per frame (sync word + header + CRC)
- **Seeking in native container:** Not natively; requires container-level index
- **Multiple streams:** Not in DTS itself; containers multiplex multiple programs

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| MPEG-2 TS | Yes (stream ID 0x82–0x85) | Yes (by PCR/PTS) | Via PMT/PES | Digital broadcast |
| MPEG-2 PS | Yes | Limited | Via IFO | DVD-Video |
| Blu-ray (m2ts) | Yes (PCR stream 0x82) | Yes (via timestamps) | Limited | Primary audio |
| DVD-Audio | Yes (MLP/DTS) | Yes | Full (WAVEFORMATEX) | DVD-Audio standard |
| Matroska/MKV | Yes (codec ID: A_DTS) | Yes (via cues) | Full Vorbis-style | |
| WAV (RF64) | Yes (limited) | No | Limited (INFO chunks) | Usually raw DTS in DATA chunk |
| AIFF | No | — | — | |
| OGG | No | — | — | |
| WebM | No | — | — | |
| CD-DA (raw DTS) | Yes | No | No | Raw DTS bitstream on CD-ROM |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** None — DTS Coherent Acoustics has no native metadata tags
- **Metadata block location:** Within frame header (DIALNORM, DYNF, CHIST, etc.)
- **User-accessible metadata:** Limited to bitstream metadata; arbitrary tags not supported

### 7.2 Standard Tag Fields — Bitstream Metadata Reference
| Field | Location | Encoding | Range | Description |
|-------|----------|----------|-------|-------------|
| Dialog Normalization (DIALNORM) | Frame header | uint4 | 0–15 | Dialogue level (gain adjustment) |
| Dialogue Normalization Gain (DNG) | Frame header | uint4 | varies | Actual gain in dBFS |
| Dynamic Range Flag (DYNF) | Frame header | bit | 0/1 | DRC present flag |
| Channel History (CHIST) | Frame header | uint2 | 0–3 | Source/recorded |
| HDCD | Frame header | uint3 | 0–7 | High Definition CD flag |
| PCM Bit Depth (PCMR) | Frame header | uint3 | 0–7 | Source bit depth = PCMR+1 |
| Timestamp Flag (TIMEF) | Frame header | bit | 0/1 | Timestamp present |
| Auxiliary Flag (AUXF) | Frame header | bit | 0/1 | Auxiliary data present |

### 7.3 DTS 96/24 Format
```
DTS 96/24 uses the core + X96 extension:
  - Core: 48 kHz / 24-bit equivalent (quantized to core precision)
  - Extension: 48 kHz residual (difference signal)
  - Combined: 96 kHz / 24-bit (perceptually equivalent)

Frame structure:
  - Same FSIZE as normal DTS core
  - X96 extension appended after core
  - Backward compatible: older decoders ignore X96
```

### 7.4 Metadata Compatibility Matrix
| Tag System | Read | Write | Notes |
|------------|------|-------|-------|
| DTS Bitstream | N/A | N/A | Proprietary metadata only |
| ID3v2 | N/A | N/A | Not native to DTS |
| Vorbis Comments | N/A | N/A | Not native to DTS |
| Container tags | Via container | Via container | DVD-Audio, Blu-ray, MKV have their own metadata systems |

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   dca                 # DTS Coherent Acoustics decoder/encoder
AV_CODEC_ID:        AV_CODEC_ID_DTS    # (in libavcodec/codec_id.h)
Format Name (CLI):  dts                 # raw DTS elementary stream
Demuxer(s):         dts, matroska, mpegts, mpeg (auto-detects based on sync word)
Muxer(s):           dts, mpegts, matroska  # raw DTS elementary stream
Decoder(s):         dca                  # FFmpeg's DTS decoder (libdcadec-based)
Encoder(s):         dca (experimental, discouraged) # Native DTS encoder (limited)
```

**Important:** FFmpeg has a native DTS encoder (`dca`) but it is highly experimental and produces lower quality than reference encoders. FFmpeg cannot encode DTS-HD formats. For production DTS encoding, use external tools (dcaenc, Surcode, DTS-HD Master Audio Suite).

### 8.2 FFmpeg Encoding — Full CLI Reference

```bash
# Basic DTS encoding (discouraged — quality is inferior)
ffmpeg -i input.wav \
  -c:a dca \
  -b:a 1411k \
  -ar 48000 \
  -ac 6 \
  output.dts

# DTS encoding with explicit bitrate (standard DVD/Blu-ray rate)
ffmpeg -i input.wav \
  -c:a dca \
  -b:a 1509k \
  -ar 48000 \
  -ac 6 \
  output.dts

# Encode stereo as DTS core
ffmpeg -i input.wav \
  -c:a dca \
  -b:a 1411k \
  -ar 48000 \
  -ac 2 \
  output.dts

# WARNING: FFmpeg does NOT support DTS-HD MA or DTS-HD HRA encoding.
#   FFmpeg can only encode DTS core at specific bitrates.
#   For DTS-HD formats, use external encoders (DTS-HD MA Suite, Surcode).
```

#### Complete FFmpeg DTS Encoder Option Table
| Option | Type | Default | Range / Valid Values | Effect on Output |
|--------|------|---------|---------------------|-----------------|
| `-b:a` | int | auto | 32k–1509k | Target bitrate (encoder will select nearest) |
| `-ac` | int | auto | 1–6 | Output channel count |
| `-ar` | int | auto | 8000–96000 | Output sample rate |
| `-channel_layout` | string | auto | standard layouts | Output channel arrangement |
| `-dca_adpx` | int | — | — | [NEEDS VERIFICATION] |
| `-dca_adxt` | int | — | — | [NEEDS VERIFICATION] |

**Critical Warning:** FFmpeg's DTS encoder is not production-quality. It is suitable only for testing or legacy compatibility. For high-quality DTS encoding, use the DTS-HD Master Audio Suite or dcaenc.

### 8.3 FFmpeg Encoding — libavcodec C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/opt.h>
#include <libswresample/swresample.h>

// ─── 1. Find DTS encoder ─────────────────────────────────────────────────────
const AVCodec *codec = avcodec_find_encoder(AV_CODEC_ID_DTS);
if (!codec) { fprintf(stderr, "DTS encoder not found\n"); exit(1); }

// ─── 2. Allocate codec context ───────────────────────────────────────────────
AVCodecContext *ctx = avcodec_alloc_context3(codec);
if (!ctx) { fprintf(stderr, "OOM\n"); exit(1); }

// ─── 3. Set required parameters ─────────────────────────────────────────────
ctx->bit_rate     = 1509750;                 // 1509.75 kbps standard DTS core
ctx->sample_fmt   = AV_SAMPLE_FMT_S32;       // REQUIRED: signed 32-bit integer
ctx->sample_rate  = 48000;                   // Standard rate
av_channel_layout_default(&ctx->ch_layout, 6);  // 5.1 channels
// ctx->ch_layout = (AVChannelLayout)AV_CHANNEL_LAYOUT_5POINT1;

// ─── 4. Open codec ──────────────────────────────────────────────────────────
int ret = avcodec_open2(ctx, codec, NULL);
if (ret < 0) {
    char errbuf[128]; av_strerror(ret, errbuf, sizeof(errbuf));
    fprintf(stderr, "avcodec_open2 failed: %s\n", errbuf);
    exit(1);
}

// ─── 5. Allocate frame and packet ──────────────────────────────────────────
AVFrame *frame = av_frame_alloc();
frame->format      = ctx->sample_fmt;
frame->sample_rate = ctx->sample_rate;
av_channel_layout_copy(&frame->ch_layout, &ctx->ch_layout);
frame->nb_samples  = ctx->frame_size;  // Variable for DTS; check ctx->frame_size
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

// ─── 8. Cleanup ─────────────────────────────────────────────────────────────
av_frame_free(&frame);
av_packet_free(&pkt);
avcodec_free_context(&ctx);
```

**Critical Notes:**
- FFmpeg's DTS encoder (`dca`) is EXPERIMENTAL and DISCOURAGED for production use
- Only supports a limited set of bitrates: 754.5, 1509.75, 1234.8 kbps (44.1 kHz variant)
- Only supports up to 5.1 channels
- Does NOT support DTS-HD MA, DTS-HD HRA, DTS-ES, DTS 96/24
- For production work, use external DTS encoding tools

### 8.4 FFmpeg Decoding — Full CLI Reference

```bash
# Decode DTS to raw PCM WAV
ffmpeg -i input.dts \
  -c:a pcm_s16le \
  output.wav

# Decode DTS at native sample rate
ffmpeg -i input.dts \
  -c:a pcm_s16le \
  -ar 48000 \
  output.wav

# Decode DTS-HD (decodes core, ignores extensions — outputs core quality)
ffmpeg -i input.dtshd \
  -c:a pcm_s16le \
  output.wav

# Decode DTS and force specific sample rate (resample)
ffmpeg -i input.dts \
  -c:a pcm_s16le \
  -ar 44100 \
  output.wav

# Decode and downmix to stereo
ffmpeg -i input_5.1.dts \
  -c:a pcm_s16le \
  -ac 2 \
  output_stereo.wav

# Decode and output 24-bit PCM
ffmpeg -i input.dts \
  -c:a pcm_s24le \
  output_24bit.wav

# Probe DTS file info
ffprobe -v quiet -print_format json -show_streams -show_format input.dts | jq .streams[0]

# Extract DTS elementary stream from container
ffmpeg -i input.mkv -c:a copy -map 0:a:0 output.dts

# Decode DTS-ES 6.1 Discrete (FFmpeg outputs 7 channels or downmixes)
ffmpeg -i input_dts_es_discrete.dts \
  -c:a pcm_s16le \
  output_7ch.wav
```

### 8.5 FFmpeg Decoding — libavcodec C API

```c
// ─── Complete decode program skeleton ────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.dts", NULL, NULL);
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
            // frm->data[0..channels-1] contain audio samples
            // frm->nb_samples = sample count per channel
            // frm->format = AVSampleFormat (S32 for DTS)
            // frm->sample_rate = actual rate (may be 48000 or 96000 for DTS-HD)
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
# Read DTS metadata
ffprobe -v quiet -print_format json -show_format input.dts | jq .format.tags

# DTS bitstream metadata (read-only from bitstream):
# - dialnorm: Dialog normalization level
# - core_period: DTS core period information

# Strip metadata
ffmpeg -i input.dts -c:a copy -map_metadata -1 output.dts

# DTS does not support arbitrary metadata tags in the bitstream.
# Use container-level metadata (MKV, MP4, etc.) for tagging DTS files.
```

### 8.7 Quality / Fidelity Decision Table
| Use Case | Settings | Expected Size (min) | Notes |
|----------|----------|---------------------|-------|
| Archival / lossless | DTS-HD MA via external encoder | up to 9 MB/min | Requires licensed encoder |
| High quality lossy | `-c:a dca -b:a 1509k -ar 48000 -ac 6` | ~5.9 MB/min (5.1) | Standard DVD/Blu-ray core |
| Standard | `-c:a dca -b:a 768k -ar 48000 -ac 6` | ~3 MB/min | Low-bitrate DTS core |
| Stereo music | `-c:a dca -b:a 1411k -ar 48000 -ac 2` | ~5.5 MB/min | CD-quality DTS |
| Streaming (standard) | `-c:a dca -b:a 768k` | ~3 MB/min | Network streaming |
| Mobile / low bandwidth | `-c:a dca -b:a 192k` | ~0.75 MB/min | Low bandwidth |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
```
DTS has no native seek table. Seeking is handled by:
  - Container-level index (MPEG-TS PSI, Matroska cues, Blu-ray index)
  - CBR byte offset calculation for raw DTS elementary streams

CBR seek formula (DTS core at fixed bitrate):
  byte_offset = floor(target_time_seconds × bitrate_bps / 8)
  frame_number = floor(target_time_seconds / frame_duration)
  byte_offset = frame_number × frame_size_bytes

DTS-HD MA (VBR) seek:
  Requires scan or seek table from container
  FFprobe can build frame index: ffprobe -select_streams a -show_frames input.dtshd
```

### 9.2 Gapless Playback Data
```
Encoder delay:   Variable — depends on encoder implementation
                 Core frame processing introduces inherent delay
Padding:        Variable — depends on encoder

For DTS core (normal mode, 512 samples at 48 kHz):
  Encoder delay ≈ 512 samples = 10.67 ms
  Padding ≈ 0 samples (no encoder padding added)

For DTS-HD MA:
  Additional delay from XLL processing
  Encoder delay stored in DTS-HD MA header

Storage location: Not in DTS bitstream.
                   Managed by container (e.g., Blu-ray playlist timestamps)

FFmpeg gapless handling:
  FFmpeg decoders output all samples including delay
  User must trim based on frame timing information
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | Frame-based; can decode after first frame (~10.67 ms) |
| Algorithmic encoder delay | 512 samples / 10.67 ms (normal mode) | At 48 kHz |
| Live encoding feasible | Yes | Real-time encoding possible with hardware |
| HTTP progressive download | Yes | Via containers (MKV, M2TS) or raw DTS |
| HTTP Live Streaming (HLS) | Yes | Via MPEG-TS segments |
| DASH streaming | Yes | Via MPEG-TS segments |
| WebRTC / RTP transport | Yes | Via MPEG-TS or RTP (requires muxing) |
| Minimum decode buffer | 1 frame / 10.67 ms | At 48 kHz normal mode |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | Layout Name | Channel Order | FFmpeg Layout Constant | AMODE |
|----------|-------------|---------------|-----------------------|-------|
| 1 | Mono | C | AV_CHANNEL_LAYOUT_MONO | 0 |
| 2 | Stereo | L, R | AV_CHANNEL_LAYOUT_STEREO | 2 |
| 2 | Dual Mono | A, B | — | 1 |
| 2 | Lt/Rt (Matrix) | Lt, Rt | — | 4 |
| 3 | 3.0 | C, L, R | — | 5 |
| 4 | 4.0 | L, R, SL, SR | AV_CHANNEL_LAYOUT_QUAD | 6 |
| 5 | 5.0 | FL, FR, C, SL, SR | AV_CHANNEL_LAYOUT_5POINT0 | 8 |
| 6 | 5.1 | FL, FR, C, LFE, SL, SR | AV_CHANNEL_LAYOUT_5POINT1 | 10 |
| 6 | DTS-ES Matrix 6.1 | C, L, R, SL, SR, BC | — | 9 |
| 7 | DTS-ES Discrete 6.1 | C, L, R, SL, SR, BC, LFE | — | 11 |
| 8 | 7.1 | FL, FR, C, LFE, BL, BR, SL, SR | — | via XXCH |
| 8 | 7.1 (Blu-ray) | FL, FR, C, LFE, FLC, FRC, BL, BR | — | via XXCH |

### 11.2 Downmix Coefficients (5.1 → Stereo)
```
Standard DTS downmix:
  L_out = L + 0.7071 × C + 0.7071 × SL
  R_out = R + 0.7071 × C + 0.7071 × SR
  LFE:  discarded

FFmpeg auto-downmix (default coefficients):
  ffmpeg -i input_5.1.dts -c:a pcm_s16le -ac 2 output_stereo.wav
    # Uses standard Lt/Rt downmix coefficients

Custom downmix with pan filter:
  ffmpeg -i input_5.1.dts -c:a pcm_s16le \
    -af "pan=stereo|FL=FC+0.707*FC+0.707*BL|FR=FR+0.707*FC+0.707*BR" \
    output_stereo.wav
```

### 11.3 DTS-ES Variants

#### DTS-ES Matrix (6.1)
```
- Adds a back surround channel (BC) via matrix encoding into SL and SR
- Compatible with standard 5.1 DTS decoders (back channel ignored)
- Requires DTS-ES capable receiver to decode the extra channel
- AMODE = 9 (C, L, R, SL, SR, BC)
```

#### DTS-ES Discrete (6.1 / 7.1)
```
- Adds a truly discrete back surround channel (not matrixed)
- Back channel is encoded separately in the bitstream (XCH extension)
- Compatible with standard 5.1 DTS decoders (XCH extension ignored)
- AMODE = 11 (C, L, R, SL, SR, BC, LFE)
- Standard on many Blu-ray titles
```

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | DTS Core | DTS 96/24 | DTS-HD MA | Notes |
|------|----------|-----------|-----------|-------|
| Max bit depth | 24-bit input | 24-bit | 24-bit | Variable by format |
| Max sample rate | 48 kHz | 96 kHz | 192 kHz | Per channel |
| Float support | No | No | No | Fixed-point throughout |
| Lossless | No | No | Yes (XLL) | DTS-HD MA provides lossless |
| Channels | 5.1 | 5.1 | 7.1/5.1 | DTS-HD MA supports up to 8 discrete |
| Aggregate bitrate | 1.5 Mbps | 1.5 Mbps core | 24.5 Mbps | DTS-HD MA for Blu-ray |

```bash
# High-resolution DTS-HD MA (lossless) — decoding via FFmpeg:
# Note: FFmpeg decodes core only; DTS-HD extensions are ignored
ffmpeg -i input_dtshd_ma.dts -c:a pcm_s24le output_24bit.wav

# DTS 96/24 decoding:
ffmpeg -i input_dts_96_24.dts -c:a pcm_s24le output_96khz.wav
```

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

| Platform / API | Encode | Decode | FFmpeg Flag | Notes |
|----------------|--------|--------|-------------|-------|
| NVIDIA NVENC | No | — | — | NVENC does not support DTS |
| NVIDIA NVDEC | — | Yes | `-hwaccel cuda` | GPU decode for video pipelines |
| Intel QSV | No | Yes | `-hwaccel qsv` | Via Quick Sync Video |
| Apple AudioToolbox | No | Yes | — | Hardware DTS decode on macOS/iOS |
| Android MediaCodec | No | Yes | — | DTS decoder required on Android |
| VA-API (Linux) | No | Yes | `-hwaccel vaapi` | Decode via VA-API |
| DSP/Embedded | No | Yes | — | Many SoCs have DTS decoder hardware |

```bash
# Android: DTS is typically decoded via MediaCodec
#   Android supports DTS natively on most devices with appropriate licensing

# Build FFmpeg with DTS support:
git clone https://github.com/FFmpeg/FFmpeg.git
cd FFmpeg
./configure --enable-decoder=dca --enable-demuxer=dts
make -j$(nproc)
sudo make install

# Verify decoder is available
./ffmpeg -decoders | grep dca
# Should show:  DEA..D dca             DTS Coherent Acoustics
```

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Encoder-Specific Issues
| Issue | FFmpeg Versions Affected | Workaround |
|-------|------------------------|------------|
| DTS encoder quality far below reference | All versions | Use external DTS encoder (Surcode, dcaenc, DTS-HD MA Suite) |
| DTS encoder only supports limited bitrates | All versions | Only 754.5, 1509.75, 1234.8 kbps available |
| DTS encoder does not support DTS-HD formats | All versions | No workaround; external encoder required |
| DTS encoder produces audible artifacts | All versions | Use reference encoder for production |
| Cannot encode DTS-ES or DTS 96/24 | All versions | External encoder required |
| DTS encoder frame size inconsistencies | Some versions | [NEEDS VERIFICATION] |

### 14.2 Interoperability Issues
- **FFmpeg DTS → consumer AV receivers:** FFmpeg-encoded DTS core is generally compatible, but some receivers reject non-standard bitrates
- **DTS-HD MA passthrough:** Many receivers require original DTS-HD MA bitstream; re-encoded files may not pass through
- **14-bit DTS:** Some CD-ROM drives and older players do not support 14-bit DTS format
- **DTS-ES Discrete on older receivers:** Not all DTS decoders support DTS-ES Discrete; they may only decode core 5.1
- **Sample rate at 44.1 kHz:** Some older AV receivers do not support DTS at 44.1 kHz

### 14.3 Edge Cases to Handle in Converter
- **Empty file / 0-sample stream:** Return empty output; no error
- **File < 1 frame:** Encode minimum 1 frame; pad with silence if needed
- **All-silence audio:** DTS encoder handles gracefully
- **DC offset (non-zero mean):** Encoder applies high-pass filter; no special handling needed
- **Full-scale sine (0 dB):** Output samples may clip; encoder does not apply makeup gain
- **File with corrupt sync:** Search for next valid sync word; report frame count
- **Truncated file:** Decode up to last complete frame; report frame count mismatch
- **Sample rate not supported:** Resample to 48 kHz or 44.1 kHz as appropriate
- **Channel count not supported:** Downmix to stereo or mono

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM DTS

| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → FLAC | `ffmpeg -i in.dts -c:a flac -compression_level 8 out.flac` | No (no DTS tags) | Lossless decode; ReplayGain scan needed |
| → ALAC | `ffmpeg -i in.dts -c:a alac out.m4a` | No | Lossless decode |
| → MP3 | `ffmpeg -i in.dts -c:a libmp3lame -q:a 0 out.mp3` | No | Generation loss (lossy→lossy) |
| → AAC | `ffmpeg -i in.dts -c:a aac -b:a 256k out.m4a` | No | Generation loss |
| → Opus | `ffmpeg -i in.dts -c:a libopus -b:a 128k out.opus` | No | Generation loss |
| → WAV | `ffmpeg -i in.dts -c:a pcm_s16le out.wav` | No | Lossless decode |
| → AC3 | `ffmpeg -i in.dts -c:a ac3 -b:a 448k out.ac3` | No | Transcode DTS→AC3 |

### 15.2 Converting TO DTS

| Source | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| FLAC → | `ffmpeg -i in.flac -c:a dca -b:a 1509k -ar 48000 out.dts` | No | FFmpeg encoder (experimental) |
| WAV → | `ffmpeg -i in.wav -c:a dca -b:a 1509k -ar 48000 out.dts` | No | FFmpeg encoder (experimental) |
| DTS-HD MA → | External encoder | No | FFmpeg cannot encode DTS-HD; use DTS-HD MA Suite |
| AC3 → | `ffmpeg -i in.ac3 -c:a dca -b:a 1509k out.dts` | No | Double lossy encoding |

**Important:** FFmpeg's DTS encoder is experimental. For production DTS encoding, use:
- **dcaenc** (freeware, command-line DTS encoder)
- **Surcode DVD-DTS** (commercial, professional)
- **DTS-HD Master Audio Suite** (commercial, DTS, Inc.)

### 15.3 Lossless Round-Trip Verification
DTS core is lossy. DTS-HD MA is lossless (requires external encoder).
```bash
# For DTS-HD MA (lossless):
# Encode with external DTS-HD MA encoder
# Decode with FFmpeg
ffmpeg -i original.wav -c:a dca_hd -b:a 1509k -ar 48000 out.dtshd
ffmpeg -i out.dtshd -c:a pcm_s24le decoded.wav
md5sum original.wav decoded.wav  # Should match for DTS-HD MA

# For DTS core (lossy):
# Verify decoded quality vs original
ffmpeg -i original.wav -c:a pcm_s16le original_raw.wav
ffmpeg -i original.wav -c:a dca -b:a 1509k -ar 48000 out.dts
ffmpeg -i out.dts -c:a pcm_s16le decoded.wav
# Compare (there will be differences — DTS core is lossy)
md5sum original_raw.wav decoded.wav

# Use EBU R128 for loudness comparison
ffmpeg -i original.wav -af loudnorm=print_format=json -f null - 2>&1 | jq .input_i
ffmpeg -i out.dts -af loudnorm=print_format=json -f null - 2>&1 | jq .input_i
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| DTS Reference Encoder | C/C++ | Proprietary | 10/10 (reference) | — | Licensed via DTS, Inc. |
| FFmpeg (libavcodec) | C | LGPL 2.1+ | 3/10 (experimental) | 9/10 | https://ffmpeg.org |
| libdcadec | C | BSD | — | 9/10 | Merged into FFmpeg (2016) |
| dcaenc | C | Freeware | 7/10 | — | Third-party DTS core encoder |
| Surcode DVD-DTS | Proprietary | Commercial | 9/10 | — | Licensed software |
| DTS-HD MA Suite | Proprietary | Commercial | 10/10 | — | DTS, Inc. official |
| Sonic DTS decoder | C++ | Proprietary | — | 7/10 | Historical; slow |

### Build Instructions
```bash
# Build FFmpeg with DTS support (DTS is built-in, no external library needed)
git clone https://github.com/FFmpeg/FFmpeg.git
cd FFmpeg
./configure --enable-decoder=dca --enable-demuxer=dts --enable-muxer=dts
make -j$(nproc)
sudo make install

# Verify decoder and encoder are available
./ffmpeg -decoders | grep dca
./ffmpeg -encoders | grep dca
# Decoder should show:  DEA..D dca
# Encoder may show:     DA.... dca  (experimental)

# For dcaenc (third-party DTS encoder):
git clone https://github.com/foo86/dcaenc.git
cd dcaenc
make
# Usage: ./dcaenc input.wav output.dts -br 1509 -ch 6 -sm 48
```

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **ETSI TS 102 114 V1.6.1 (2019):** DTS Coherent Acoustics; Core and Extensions with Additional Profiles — https://www.etsi.org/deliver/etsi_ts/102100_102199/102114/01.06.01_60/ts_102114v010601p.pdf
- **ETSI TS 102 114 V1.4.1 (2012):** Previous version with additional profile details

### Technical Resources
- FFmpeg DCA decoder options: `ffmpeg -h decoder=dca`
- Multimedia Wiki DTS: https://wiki.multimedia.cx/index.php/DTS
- Multimedia Wiki DTS-HD: https://wiki.multimedia.cx/index.php/DTS-HD
- Hydrogenaudio DTS: https://hydrogenaudio.org/wiki/DTS
- DTS Whitepaper: https://www.mp3-tech.org/programmer/docs/dts_whitepaper.pdf
- DTS-HD Whitepaper: https://www.fast-and-wide.com/images/stories/White_papers/dts_hd_whitepaper.pdf

### Academic Papers
- Todd D. M. et al., "A Digital Compatible Surround Sound System for DVD-Audio," AES 107th Convention, 1999
- Grant Davidson, "Multiple Channel Audio Coding and Playback," AES 117th Convention, 2004
- Various AES papers on DTS Coherent Acoustics psychoacoustic model (proprietary)

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] FFmpeg `./configure` flags: `--enable-decoder=dca --enable-demuxer=dts --enable-muxer=dts`
- [ ] Verify `ffmpeg -decoders` output confirms dca decoder is available
- [ ] Verify `ffmpeg -encoders` output confirms dca encoder (experimental) is available
- [ ] Note: FFmpeg DTS encoder is NOT production-quality; recommend external encoder
- [ ] No external library dependency for DTS core decoding; libdcadec is merged into FFmpeg
- [ ] Handle platform licensing (DTS patents — commercial licensing may be needed for distribution)

### Encoding Pipeline
- [ ] Convert input sample format to `AV_SAMPLE_FMT_S32` using libswresample (DTS encoder requires S32)
- [ ] DTS frame size is variable; check `ctx->frame_size` after opening codec
- [ ] Implement proper flush/drain at end of stream (send NULL frame)
- [ ] Validate that input sample rate is supported ({8000, 11025, 12000, 16000, 22050, 24000, 32000, 44100, 48000})
- [ ] Validate that channel layout is supported; auto-downmix if needed
- [ ] FFmpeg cannot encode DTS-HD MA, DTS-HD HRA, DTS-ES, or DTS 96/24 — use external tools

### Decoding Pipeline
- [ ] Implement sync word search (0x7FFE8001, big-endian; also check variants for 14-bit)
- [ ] Handle AVERROR(EAGAIN) in receive_frame loop correctly
- [ ] Flush decoder at EOF (send NULL packet)
- [ ] DTS decoder delay is variable; managed by FFmpeg internally
- [ ] Handle sample format conversion from S32 to target format (S16/S24/FLOAT)
- [ ] Respect dynamic range metadata from bitstream (DYNF flag)

### Metadata
- [ ] DTS has no user-metadata tag system; metadata is in bitstream only
- [ ] Read dialog normalization level from DIALNORM field
- [ ] Read dynamic range flag (DYNF)
- [ ] Apply or bypass DRC based on user preference
- [ ] Container-level tags are not preserved through DTS encode/decode

### Quality & Verification
- [ ] Verify decoded audio quality meets expectations for source format
- [ ] Test with silence, full-scale sine, transient-rich, dialogue-heavy content
- [ ] Verify frame size is valid for the declared bitrate
- [ ] Verify sample rate is correctly decoded from SFREQ field
- [ ] Verify channel count and AMODE are correctly decoded
- [ ] For DTS-HD MA files: FFmpeg will decode core only; extensions are ignored

### Edge Cases
- [ ] Handle files with corrupt or missing sync words (search forward for next 0x7FFE8001)
- [ ] Handle files with 0 samples (produce empty output)
- [ ] Handle sample rate mismatch (resample using libswresample)
- [ ] Handle channel count mismatch (downmix via pan filter or auto-downmix)
- [ ] Handle very short files (< 1 frame) — pad with silence
- [ ] Handle 14-bit DTS format (alternative sync words: 0x1FFFE800, 0xFF1F00E8)

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
