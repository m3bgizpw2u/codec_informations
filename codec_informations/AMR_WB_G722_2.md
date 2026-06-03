# AMR-WB / G.722.2 — Deep Technical Reference
> **Category:** Lossy
> **File Extensions:** `.awb`, `.3gpp`, `.3gp`, `.gsm`
> **MIME Types:** `audio/amr-wb`, `audio/3gpp`, `audio/gsm`
> **Standardization Body:** ITU-T | 3GPP
> **Primary Specification:** ITU-T G.722.2 | 3GPP TS 26.190 | 3GPP TS 26.171
> **Patent Status:** Patented — licensing via VoiceAge Corporation and 3GPP essential patents
> **License:** Royalty-bearing (3GPP licensing terms, VoiceAge licensing)
> **Current Version:** G.722.2 (2003), AMR-WB+ extended version (2004)
> **Active Development:** No — standardized in 2003, stable and widely deployed

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** 3GPP (3rd Generation Partnership Project) in collaboration with ITU-T Study Group 16
- **Year Created:** 2001–2003 (3GPP TS 26.190 finalized 2001; ITU-T G.722.2 approved July 2003)
- **Original Purpose:** Provide wideband speech coding (50–7000 Hz) for 3G wireless networks, VoLTE (Voice over LTE), VoIP applications, and wireline conferencing. The primary motivation was improving speech intelligibility and naturalness over narrowband codecs (G.711, AMR NB) by doubling the audio bandwidth.
- **Problem with Predecessors:** Narrowband codecs (300–3400 Hz) like G.711 and AMR-NB produced muffled, artificial-sounding speech. Users and equipment manufacturers demanded wideband quality (50–7000 Hz) for improved speech clarity, especially in conferencing and music-coded audio segments. Existing wideband codecs like G.722 (64 kbps SB-ADPCM) were too high-bitrate for cellular networks.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| AMR-WB (Initial) | 2001 | 3GPP TS 26.190, 9 bitrates (6.6–23.85 kbps), ACELP, VAD/DTX/CNG |
| ITU-T G.722.2 | 2003 | Converged standard, identical to 3GPP AMR-WB, approved July 2003 |
| AMR-WB+ (Plus) | 2004 | Extended bitrates up to 48 kbps, stereo support, higher sample rates (up to 48 kHz), mode switching between ACELP and TCX |
| RFC 3267 | 2002 | RTP payload format specification for AMR and AMR-WB |
| RFC 4867 | 2007 | Updated RTP payload format (mode adaptation, redundant coding) |

### 1.3 Current Adoption
- **Primary use cases today:** VoLTE (Voice over LTE), VoIP (SIP-based conferencing), 3GPP packet-switched streaming service (PSS), wireless voice over Wi-Fi (VoWLAN), PacketCable 2.0 (cable VoIP)
- **Platforms with native support:** iOS (Core Audio framework), Android (MediaCodec), BlackBerry, Windows Phone, most modern smartphones via baseband/RIL
- **Major services using this format:** 4G/LTE mobile voice (VoLTE), Wi-Fi calling, 3GPP multimedia messaging, video telephony fallback
- **Hardware support:** Built into cellular modem chipsets (Qualcomm, MediaTek, Samsung, Intel), most smartphone application processors for hardware decode
- **Status:** Dominant wideband speech codec in mobile networks globally; deployed on billions of devices

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Parametric / Model-based (CELP family)
- **Core algorithm:** ACELP — Algebraic Code-Excited Linear Prediction
- **Loss mechanism:** Perceptual quantization of LPC parameters, pitch parameters, and algebraic codebook indices. The codec operates entirely in the parametric domain — no transform coding of the full spectrum.
- **Frame-based vs sample-based:** Frame-based. All processing operates on 20 ms frames.
- **Fixed vs variable frame size:** Fixed frame size of 20 ms (320 samples at 16 kHz), with internal 5 ms sub-frames (four sub-frames per frame for ACELP processing).

### 2.2 High-Level Encoding Flow (Block Diagram)
```
Input Speech (16 kHz, 16-bit PCM)
         │
         ▼
[Pre-processing: DC removal, High-pass filter 50 Hz]
         │
         ▼
[Open-loop Pitch Analysis (every 5 ms sub-frame)]
         │
         ▼
[LPC Analysis (10th order ISP on 20 ms frame)]
         │
         ▼
[ISP Quantization (Split Vector Quantization, 46 bits)]
         │
         ▼
[ISP to LSP Conversion and Interpolation]
         │
         ▼
[Per Sub-frame (4× per frame):]
 ├── Closed-loop Pitch Search (Fractional pitch, 1/4 sample)
 ├── Algebraic Codebook Search (17-bit ACELP, 4 subframes)
 ├── Gain Quantization (Scalar, 7 bits per subframe)
 └── High-band Excitation (23.85 kbps mode only)
         │
         ▼
[Bitstream Packing: Mode, ISP, pitch, codebook, gain]
         │
         ▼
Output AMR-WB Bitstream (6.60–23.85 kbps variable)
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | 25 ms total | 20 ms frame + 5 ms look-ahead |
| Frame size | 320 samples | 20 ms at 16 kHz |
| Sub-frame size | 80 samples | 5 ms at 16 kHz |
| Max channels | 1 (mono) | AMR-WB+ supports stereo |
| Max bit depth | 16-bit PCM input | Internal 32-bit float/computation |
| Max sample rate | 16000 Hz | AMR-WB; AMR-WB+ extends to 48000 Hz |
| Bitrate range | 6.60–23.85 kbps | 9 discrete modes |
| Complexity | 38 WMOPS | Weighted million operations per second |
| RAM usage | 5.3 K words | Fixed-point implementation |
| Frequency bandwidth | 50–7000 Hz | All modes; 23.85 kbps adds 6400–7000 Hz |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  1       0x23             #         AMR-WB file header magic ('#!AMR-WB\n')
0x0001  7       ...              !AMR-WB   Codec identifier
0x0008  1       0x0A             \n        Line feed terminator
```

### 3.2 Frame Header Layout
```
AMR-WB uses a 1-byte frame header per frame:
Bit Offset   Bit Width   Field Name              Type      Description
----------   ---------   --------------------    ------    ---------------------------
0            1           Frame quality flag     uint1     0 = corrupted, 1 = good
1            4           Mode (bitrate)          uint4     0–8 (see Mode Table below)
5            3           Frame padding           uint3     Reserved, set to 0

Note: In octet-aligned storage (.awb files), the frame header occupies
the first byte of each frame. In RTP transport (RFC 3267), the CMR
(Command Marker) field occupies the first 4 bits of each payload.
```

#### Mode (Bitrate) Index Table
| Mode | Bitrate (kbps) | Frame Bits | Frame Bytes | Frame Quality | Bandwidth |
|------|-----------------|------------|-------------|--------------|-----------|
| 0 | 6.60 | 132 | 17 | Good | Narrowband-equivalent |
| 1 | 8.85 | 177 | 23 | Good | At/below G.722 at 48 kbps |
| 2 | 12.65 | 253 | 32 | Good | Main anchor, clean speech |
| 3 | 14.25 | 285 | 36 | Good | |
| 4 | 15.85 | 317 | 40 | Good | >= G.722 at 56 kbps |
| 5 | 18.25 | 365 | 46 | Good | |
| 6 | 19.85 | 397 | 50 | Good | |
| 7 | 23.05 | 461 | 58 | Good | |
| 8 | 23.85 | 477 | 60 | Best | >= G.722 at 64 kbps, comfortable wideband |

### 3.3 AMR-WB+ Extended Format
```
AMR-WB+ uses a different frame structure:
Offset  Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------
0x0000   4B      Sync word              uint32      0x4E4F4345 ("ONEC")
0x0004   4B      Frame size            uint32      Size of this frame in bytes
0x0008   4B      Config/Side info      uint32      Bitrate mode, stereo flag, etc.
0x000C   N B     Audio data            uint8[]     AMR-WB+ encoded payload
```

### 3.4 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Unsigned integer | No | Not supported |
| 16-bit | Signed integer | Yes | Primary input format |
| 20-bit | Signed integer | Yes | Supported in some implementations |
| 24-bit | Signed integer | Yes | Accepted but truncated to 16-bit internally |
| 32-bit | Signed integer | Partial | May be accepted via sample format conversion |
| 32-bit | IEEE float | Partial | Requires conversion to S16 or S32 |
| 64-bit | IEEE float (double) | No | Not directly supported |

#### Supported Sample Rates
| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 8000 | Telephone | No | Not native; resample to 16 kHz |
| 16000 | Wideband voice | Yes | **Native input/output rate** |
| 24000 | Super-wideband | Partial | AMR-WB+ supports up to 24 kHz |
| 32000 | Broadcast | Partial | AMR-WB+ supports up to 32 kHz |
| 44100 | CD audio | Partial | AMR-WB+ decodes but internally downsamples |
| 48000 | Professional | Partial | AMR-WB+ supports up to 48 kHz |
| Higher | High-res | No | Not supported |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Pre-Processing Stage
- **DC offset removal:** Subtraction of the mean (DC component) from the input signal before LP analysis. The DC offset is estimated over the current frame.
- **Pre-emphasis filter:** `H_pre(z) = 1 - 0.68·z^(-1)` applied to the speech signal before analysis. This spectral flattening emphasizes higher frequencies and improves LP model accuracy for voiced sounds.
- **Windowing function:** Asymmetric Hamming window applied to the 20 ms frame (320 samples). The window has different shapes for the forward and backward portions to minimize frame boundary artifacts.
- **Level normalization:** Not typically applied; the encoder handles wide dynamic range through its internal arithmetic. Some implementations apply automatic level control before fixed-point conversion.
- **Stereo decorrelation pre-step:** Not applicable in AMR-WB (mono only). AMR-WB+ handles stereo through side-channel encoding and independent mono encoding of channels.

### 4.2 Analysis / Transform Stage

#### Transform Type: LPC (Linear Predictive Coding) — No MDCT/FFT Transform
AMR-WB uses pure LPC analysis rather than transform coding. There is no MDCT or FFT of the full spectrum.

```
Parameters:
  ISP (Immittance Spectrum Pairs) order: 16 (10th order in some references, but 16 ISP
  parameters are quantized and transmitted)
  Window: Asymmetric Hamming, 20 ms frame + 5 ms look-ahead context
  Algorithm: Levinson-Durbin recursion in fixed-point
  Coefficient precision: 16-bit Q15 fixed-point arithmetic throughout
  Look-ahead: 5 ms (80 samples) for LP analysis and pitch estimation
  Sub-frame size: 5 ms (80 samples) for ACELP parameter estimation
```

**LPC Analysis Mathematical Definition:**

The LP analysis finds coefficients `a_i` that minimize the prediction error:

```
E = Σ(n=1 to N) [s(n) - Σ(i=1 to P) a_i · s(n-i)]²

where:
  s(n) = pre-emphasized speech sample at time n
  P    = 16 (LP order)
  E    = prediction error energy
  a_i  = LP coefficients (minimizing E)

The autocorrelation method (Levinson-Durbin) is used:
  E^(0) = R(0)
  A_i^(i) = a_i for the i-th order predictor
  E^(i) = E^(i-1) · (1 - k_i²)   where k_i are PARCOR/LSP reflection coefficients
```

ISP (Immittance Spectrum Pairs) are derived from the LP coefficients and then vector-quantized:

```
ISP Parameters: 16 values per 20 ms frame
Quantization: Split Vector Quantization (SVQ)
  - Sub-vector 1: ISP[0:7] → 8 bits
  - Sub-vector 2: ISP[8:15] → 8 bits
  Plus 1 VAD (Voice Activity Detection) flag bit
Total ISP + VAD: 46 bits per frame at 23.85 kbps mode
```

#### ACELP Codebook Structure
AMR-WB uses a **17-bit algebraic codebook** with a sparse structure optimized for efficient search:

```
Algebraic Codebook Parameters:
  Pulse count per subframe: 6 (in most modes)
  Pulse positions: 80 samples per 5 ms subframe
  Track structure: 4 tracks, 20 positions each
  Pulse signs: + or - for each pulse
  Bit allocation per subframe (varies by mode):
    23.85 kbps: 88 bits for codebook (352 bits/frame)
    12.65 kbps: 72 bits for codebook (288 bits/frame)
    6.60 kbps:  44 bits for codebook (176 bits/frame)

Codebook structure (17-bit representation):
  The codebook is "algebraic" — pulses are described by position and sign,
  not stored as explicit vectors. Position and sign indices are encoded
  directly into the bitstream.
```

#### Open-loop Pitch Analysis
```
Open-loop pitch analysis (every 5 ms sub-frame):
  Search range: 17.5 ms to 143.4 ms (280 to 2293 samples at 16 kHz)
  Algorithm: Three-tap normalized cross-correlation
  Output: Integer pitch period estimate (T0) for closed-loop search starting point
  Purpose: Reduces complexity of closed-loop pitch search
```

#### High-Band Extension (23.85 kbps Mode)
At the maximum 23.85 kbps mode, AMR-WB includes a **high-band excitation** component:
```
High-band processing:
  - Full-band signal encoded using ACELP (50–7000 Hz)
  - Bandwidth extension signal: 6400–7000 Hz (only in 23.85 kbps mode)
  - High-band energy: 16 bits per frame (4 bits × 4 subframes)
  - This enables "comfortable" wideband quality equivalent to G.722 at 64 kbps
```

### 4.3 Psychoacoustic Model (Lossy Only)

AMR-WB does **not** use a psychoacoustic model in the traditional sense (like MP3's MPEG Psychoacoustic Model 1 or 2). Instead, it relies on the **perceptual sensitivity of the LPC residual**:

```
AMR-WB Perceptual Processing Strategy:
1. The ACELP model assumes the ear is most sensitive to errors in the
   perceptually-weighted spectral domain (controlled by the LP synthesis
   and weighting filter W(z)).
   
2. The perceptual weighting filter W(z) = A(z/γ) / A(z) shapes quantization
   noise to be masked by the formants in the speech spectrum.
   - Default γ = 0.92 (broadband weighting)
   
3. The gain quantization uses a scalar quantizer with a non-uniform
   step size that reflects perceptual sensitivity to level changes.
   
4. Unlike transform codecs (MP3/AAC), there is no explicit bit allocation
   based on masking thresholds per frequency band. The ACELP codebook
   search implicitly optimizes for perceptually weighted error minimization.
   
5. VAD (Voice Activity Detection) uses energy-based and spectral
   characterization to detect speech vs. silence/noise, enabling DTX.
```

### 4.4 Quantization
- **Type:** Multi-stage quantization with SVQ (Split Vector Quantization) for LP parameters, algebraic codebook for the excitation, scalar quantization for gains and pitch.
- **Step sizes:** Non-uniform, mode-dependent. The codebook bit allocation adapts to the chosen bitrate mode.
- **Dequantization formula:** Integer lookup tables in fixed-point. No closed-form formula; codebook indices map to pulse positions directly.
- **Noise shaping:** The perceptual weighting filter `W(z) = A(z/0.92) / A(z)` provides implicit noise shaping within the LP framework.

### 4.5 Stereo Encoding Modes
| Mode | Description | Condition for Selection | Bitrate |
|------|-------------|------------------------|---------|
| Mono | Single channel, standard AMR-WB | Default | 6.60–23.85 kbps |
| AMR-WB+ Stereo | Independent mono encoding of L/R | Stereo flag in frame header | Up to 2× mono bitrate |

AMR-WB itself is mono-only. Stereo support was introduced in AMR-WB+ (the extended version), which can encode stereo with independent left and right channels.

### 4.6 Entropy / Lossless Coding Stage
```
Method: No entropy coding (Huffman/Rice/Arithmetic)

AMR-WB bitstream characteristics:
  - Raw binary parameters packed directly into the frame
  - No variable-length coding applied to parameter indices
  - Bit-level packing is fixed and deterministic per mode
  - All parameters are quantized to integer indices before packing
  
Bitstream packing example (23.85 kbps mode, 477 bits per frame):
  [Bit 0]       VAD flag (1 bit)
  [Bit 1–46]    ISP quantized indices (46 bits)
  [Bit 47–76]   Pitch delay (30 bits across 4 subframes)
  [Bit 77–428]  Algebraic codebook indices (352 bits)
  [Bit 429–456] Gain indices (28 bits across 4 subframes)
  [Bit 457–472] High-band energy (16 bits)
  [Bit 473–476] Reserved/padding (4 bits)
```

### 4.7 Encoder Settings / Quality Modes

#### For Lossy Codecs (AMR-WB is lossy)
| Quality Setting | Bitrate | Intended Use Case | Transparent? |
|---|---|---|---|
| Minimum | 6.60 kbps | Emergency/backup channel, severe congestion | No |
| Low | 8.85 kbps | Temporary rate reduction, circuit-switched fallback | No |
| Standard | 12.65 kbps | Main anchor bitrate, VoLTE default in many deployments | Near-transparent for speech |
| Good | 14.25–15.85 kbps | Enhanced quality, noisy environments | Near-transparent for speech |
| High | 18.25–19.85 kbps | High-quality voice, some music | Near-transparent for music-coded speech |
| Maximum | 23.05–23.85 kbps | Comfortable wideband, music segments | Best wideband quality |

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. Scan bitstream for AMR-WB frame header: 0x23 (decimal 35, '#')
2. Validate candidate frame:
   a. Read frame quality flag (bit 7 of header byte)
   b. Read mode field (bits 6–3 of header byte)
   c. Check mode value is 0–8 (valid range)
   d. Compute expected frame size from mode table
   e. Verify next frame header is valid at computed offset
3. If validation fails: advance 1 byte and retry
4. Max failed sync attempts before error: 1024 bytes (recommended)
```

#### Seeking
- **File-based seeking:** AMR-WB .awb files have no seek index. Seeking requires scanning from the nearest keyframe or estimating position based on bitrate.
- **RTP/streaming seeking:** Not natively supported; requires RTSP/RTCP signaling or out-of-band signaling for seeking.
- **Seek precision:** Frame-level accuracy. With known bitrate: `byte_offset ≈ (target_time_sec × bitrate_bps) / 8`.

### 5.2 Core Decode Pipeline
```
1. Read frame header (1 byte)
   ├── Verify frame quality flag
   ├── Read mode (bitrate index 0–8)
   └── Compute frame size from mode table

2. Parse bitstream parameters
   ├── VAD flag (1 bit)
   ├── ISP indices (mode-dependent, 46–38 bits)
   ├── Pitch delay per subframe (mode-dependent, 30 bits total)
   ├── Algebraic codebook indices per subframe (variable, 44–88 bits)
   └── Gain indices per subframe (28 bits), high-band energy (16 bits)

3. ISP Dequantization
   └── Split Vector Dequantization → 16 ISP parameters

4. ISP to LP Coefficients Conversion
   └── ISP → LP coefficients a_i via spectral conversion

5. Per Sub-frame (4×):
   ├── Pitch pre-filter using decoded pitch delay
   ├── Algebraic codebook vector generation (from decoded indices)
   ├── Excitation signal: pitch_contribution + codebook_contribution
   ├── Gain dequantization
   └── Synthesis filter: s[n] = Σ(i=1 to 16) a_i · s[n-i] + e[n]

6. Post-processing
   ├── Adaptive gain control (AGC) to match signal level
   ├── High-pass filter (50 Hz cutoff) to remove rumble
   └── De-emphasis filter: H_de(z) = 1 / (1 - 0.68·z^(-1))

7. Output
   └── 320 samples of decoded 16-bit PCM at 16 kHz
```

### 5.3 Error Concealment
- **Corrupt frame detection:** Frame quality flag = 0 in the frame header indicates a corrupted frame. Additionally, invalid mode values, sync word loss, and CRC failures (if used in transport) indicate corruption.
- **Concealment method:** The decoder performs **frame erasure concealment**:
  - For 1–3 consecutive bad frames: Repeat pitch period from last good frame, with exponential decay of energy.
  - For 4+ consecutive bad frames: Fade to silence gradually.
  - The algebraic codebook is replaced with a comfort noise generator when DTX is active.
- **Maximum consecutive errors before silence:** Implementation-dependent; typically 20–40 frames (400–800 ms) before full silence insertion.

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** None — AMR-WB is a bare bitstream format (.awb) or carried within 3GPP container files (.3gp, .3gpp2).
- **Overhead:** Minimal. The .awb file adds 9 bytes header per file (magic bytes).
- **Seeking in native container:** Not supported; 3GPP containers use a track-based structure that supports seeking.
- **Multiple streams in native container:** AMR-WB streams can be multiplexed in MP4/M4A containers with other tracks.

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| 3GP / 3GPP2 | Yes | Yes (track-based) | Limited (3GPP metadata) | Native container |
| MP4 / M4A | Yes | Yes | Full | Store as audio track |
| Matroska/MKA | Yes | Yes | Full | Via FFmpeg/libavformat |
| AMR (bare) | Yes | No | None | Raw bitstream format |
| WebM | No | — | — | WebM uses Opus/VP8/VP9 |
| OGG | No | — | — | OGG uses Vorbis/Opus |
| WAV | No | — | — | Not a valid WAV codec |
| FLAC | No | — | — | Not applicable |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** 3GPP-specific metadata stored in the container (not the AMR-WB bitstream itself).
- **Tag block location:** Within the 3GPP/3GP container file structure, not within the AMR-WB audio frames.
- **Tag block identifier:** 3GPP uses a proprietary metadata box within the MP4/3GP structure.

### 7.2 Standard Tag Fields — Complete Reference
| Tag Field | Internal Key (this format) | Max Length | Character Encoding | Multi-value | Notes |
|-----------|---------------------------|------------|-------------------|-------------|-------|
| Title | Title | Variable | UTF-8 | No | From 3GPP metadata |
| Artist | Performer | Variable | UTF-8 | No | |
| Album | Album | Variable | UTF-8 | No | |
| Copyright | Copyright | Variable | UTF-8 | No | |
| Encoder | Encoder | Variable | UTF-8 | No | Usually "AMR-WB" |
| Encoder Settings | Encoder Settings | Variable | UTF-8 | No | Bitrate mode |
| Geographic Location | Location | Variable | UTF-8 | No | GPS coordinates if available |

**Note:** AMR-WB is primarily a transport codec for real-time voice. Comprehensive metadata tagging is container-dependent (3GP, MP4) rather than codec-native.

### 7.3 Cover Art Storage
- **Storage:** Not natively supported in AMR-WB. Cover art is stored in the container (3GP, MP4) using the standard image attachment mechanism of that container.
- **Max image size:** Container-dependent (typically 1–10 MB).
- **Max dimensions:** Container-dependent (typically up to 4096×4096).

### 7.4 Metadata Compatibility Matrix
| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| 3GPP Native | ✓ | ✓ | ✓ | Highest |
| ID3v2 | ✗ | ✗ | ✗ | N/A |
| Vorbis Comments | ✗ | ✗ | ✗ | N/A |
| MP4 Atoms | ✓ | ✓ | ✓ | (when in MP4 container) |

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   amr_wb             # used with -c:a amr_wb
AV_CODEC_ID:        AV_CODEC_ID_AMR_WB # C constant in libavcodec/codec_id.h
Format Name (CLI):  amr                 # used with -f amr
Encoder(s):         libopencore_amrwb  # Note: Not available in this FFmpeg build
Decoder(s):         amrwb, libopencore_amrwb
Muxer(s):           amr, amrwb          # Raw AMR-WB storage
Demuxer(s):         amr, amrwb         # Raw AMR-WB reading
```

**Important Note:** This FFmpeg build was configured with `--enable-libopencore_amrwb` but the encoder is not available (likely due to patent/licensing restrictions). Only the AMR-WB decoder is typically available in standard FFmpeg distributions.

### 8.2 FFmpeg Encoding — Full CLI Reference

```bash
# NOTE: AMR-WB encoder may not be available in standard FFmpeg builds
# due to patent licensing restrictions. Only decoding is typically supported.

# If available, encoding to AMR-WB:
ffmpeg -i input.wav \
  -c:a libopencore_amrwb \
  -b:a 23.85k \           # Bitrate: 6.60k, 8.85k, 12.65k, 14.25k, 15.85k,
                          # 18.25k, 19.85k, 23.05k, 23.85k
  -ar 16000 \             # AMR-WB native sample rate
  -ac 1 \                 # Mono only
  output.awb

# Decoding AMR-WB to WAV:
ffmpeg -i input.awb \
  -c:a pcm_s16le \
  -ar 16000 \
  -ac 1 \
  output.wav

# Decoding AMR-WB with automatic format detection:
ffmpeg -i input.3gp \
  -c:a pcm_s16le \
  output.wav

# Converting 3GP container with AMR-WB to MP4 with AAC:
ffmpeg -i input.3gp \
  -vn \
  -c:a libopus -b:a 128k \
  -map_metadata 0:g \
  output.opus

# Extracting AMR-WB audio stream from 3GP:
ffmpeg -i input.3gp \
  -vn -c:a copy \
  -f amrwb \
  output.awb
```

#### Complete FFmpeg Option Table
| Option | Type | Default | Range / Valid Values | Effect on Output |
|--------|------|---------|---------------------|-----------------|
| `-b:a` | int | auto | 6.6k, 8.85k, 12.65k, 14.25k, 15.85k, 18.25k, 19.85k, 23.05k, 23.85k | Target bitrate mode |
| `-ar` | int | 16000 | 16000 only | Output sample rate (must be 16 kHz) |
| `-ac` | int | 1 | 1 | Output channel count (mono only) |
| `-frame_size` | int | 320 | 320 | Samples per frame (fixed) |

### 8.3 FFmpeg Encoding — libavcodec C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/opt.h>

// AMR-WB Encoding (if encoder is available in the FFmpeg build)
// Note: Many FFmpeg builds exclude the AMR-WB encoder due to patents.

const AVCodec *codec = avcodec_find_encoder_by_name("libopencore_amrwb");
if (!codec) {
    fprintf(stderr, "AMR-WB encoder not available in this FFmpeg build\n");
    fprintf(stderr, "Patent licensing restrictions may prevent encoder inclusion\n");
    exit(1);
}

AVCodecContext *ctx = avcodec_alloc_context3(codec);
if (!ctx) { fprintf(stderr, "OOM\n"); exit(1); }

ctx->bit_rate = 23850;              // 23.85 kbps = highest quality
ctx->sample_fmt = AV_SAMPLE_FMT_S16; // S16 is required for AMR-WB
ctx->sample_rate = 16000;            // AMR-WB native rate only
av_channel_layout_default(&ctx->ch_layout, 1); // Mono only

int ret = avcodec_open2(ctx, codec, NULL);
if (ret < 0) {
    char errbuf[128];
    av_strerror(ret, errbuf, sizeof(errbuf));
    fprintf(stderr, "avcodec_open2 failed: %s\n", errbuf);
    exit(1);
}

// AMR-WB frame size is always 320 samples (20 ms at 16 kHz)
ctx->frame_size = 320;

AVFrame *frame = av_frame_alloc();
frame->format = ctx->sample_fmt;
frame->sample_rate = ctx->sample_rate;
av_channel_layout_copy(&frame->ch_layout, &ctx->ch_layout);
frame->nb_samples = ctx->frame_size; // 320
av_frame_get_buffer(frame, 0);

AVPacket *pkt = av_packet_alloc();

// Encoding loop
void encode_frame(AVCodecContext *ctx, AVFrame *frame, AVPacket *pkt, FILE *outfile) {
    int ret = avcodec_send_frame(ctx, frame);
    if (ret < 0) { return; }

    while (ret >= 0) {
        ret = avcodec_receive_packet(ctx, pkt);
        if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF) return;
        if (ret < 0) { exit(1); }
        // Write with AMR-WB header byte
        fwrite(pkt->data, 1, pkt->size, outfile);
        av_packet_unref(pkt);
    }
}

// Flush encoder
encode_frame(ctx, NULL, pkt, outfile);

av_frame_free(&frame);
av_packet_free(&pkt);
avcodec_free_context(&ctx);
```

**Critical Notes:**
- The AMR-WB encoder is often **not available** in standard FFmpeg builds due to patent licensing restrictions from VoiceAge Corporation and 3GPP. If `avcodec_find_encoder_by_name("libopencore_amrwb")` returns NULL, encoding is not possible with this FFmpeg build.
- The AMR-WB decoder (`amrwb`) is typically available and works correctly.
- Sample rate **must be 16000 Hz** — AMR-WB only operates at 16 kHz.
- Channel count **must be 1 (mono)** — stereo is not supported in AMR-WB (use AMR-WB+ for stereo).
- `ctx->frame_size` is always 320 samples for AMR-WB.

### 8.4 FFmpeg Decoding — Full CLI Reference

```bash
# Basic AMR-WB to WAV decoding
ffmpeg -i input.awb \
  -c:a pcm_s16le \
  output.wav

# Decode and resample to 48 kHz for compatibility
ffmpeg -i input.awb \
  -c:a pcm_s16le \
  -ar 48000 \
  output_48k.wav

# Decode from 3GP container
ffmpeg -i input.3gp \
  -vn \
  -c:a pcm_s16le \
  output.wav

# Decode and transcode to Opus
ffmpeg -i input.awb \
  -c:a libopus -b:a 64k -application voip \
  output.opus

# Decode to FLAC (lossless re-encode of decoded audio)
ffmpeg -i input.awb \
  -c:a pcm_s16le \
  -c:a flac -compression_level 8 \
  output.flac

# Probe AMR-WB file information
ffprobe -v quiet -print_format json -show_streams input.awb

# Probe 3GP container with AMR-WB track
ffprobe -v quiet -print_format json -show_streams -show_format input.3gp
```

### 8.5 FFmpeg Decoding — libavcodec C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>

AVFormatContext *fmt_ctx = NULL;
if (avformat_open_input(&fmt_ctx, "input.awb", NULL, NULL) < 0) {
    fprintf(stderr, "Could not open input file\n");
    exit(1);
}

avformat_find_stream_info(fmt_ctx, NULL);

// Find AMR-WB audio stream
int audio_idx = av_find_best_stream(fmt_ctx, AVMEDIA_TYPE_AUDIO, -1, -1, NULL, 0);
if (audio_idx < 0) {
    fprintf(stderr, "Could not find AMR-WB audio stream\n");
    exit(1);
}

AVStream *stream = fmt_ctx->streams[audio_idx];
printf("Codec: %s, Bitrate: %lld, Sample Rate: %d, Channels: %d\n",
       stream->codecpar->codec ? stream->codecpar->codec->name : "unknown",
       stream->codecpar->bit_rate,
       stream->codecpar->sample_rate,
       stream->codecpar->ch_layout.nb_channels);

// Find and open decoder
const AVCodec *dec = avcodec_find_decoder(stream->codecpar->codec_id);
if (!dec) {
    // Fallback to libopencore_amrwb if available
    dec = avcodec_find_decoder_by_name("libopencore_amrwb");
}
if (!dec) {
    fprintf(stderr, "No AMR-WB decoder available\n");
    exit(1);
}

AVCodecContext *dec_ctx = avcodec_alloc_context3(dec);
avcodec_parameters_to_context(dec_ctx, stream->codecpar);
avcodec_open2(dec_ctx, dec, NULL);

// Decode loop
AVPacket *pkt = av_packet_alloc();
AVFrame *frm = av_frame_alloc();

while (av_read_frame(fmt_ctx, pkt) >= 0) {
    if (pkt->stream_index == audio_idx) {
        avcodec_send_packet(dec_ctx, pkt);
        while (avcodec_receive_frame(dec_ctx, frm) == 0) {
            // frm->sample_rate = 16000 (always)
            // frm->ch_layout.nb_channels = 1 (always)
            // frm->nb_samples = 320 (fixed per AMR-WB frame)
            // frm->format = AV_SAMPLE_FMT_S16
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

av_frame_free(&frm);
av_packet_free(&pkt);
avcodec_free_context(&dec_ctx);
avformat_close_input(&fmt_ctx);
```

### 8.6 FFmpeg Metadata Handling

```bash
# Read metadata from 3GP file
ffprobe -v quiet -print_format json -show_format input.3gp | jq .format.tags

# AMR-WB files typically have limited metadata
# Most metadata is container-specific (3GP, MP4)

# Strip metadata
ffmpeg -i input.awb -c copy -map_metadata -1 output.awb

# Copy metadata from 3GP to AMR-WB (when re-muxing)
ffmpeg -i input.3gp -c copy -map_metadata 0 output.awb
```

#### FFmpeg Internal Metadata Key Mapping
| Standard Field | FFmpeg Key | AMR-WB/3GPP Native Key | Notes |
|----------------|------------|------------------------|-------|
| Title | title | Title | From 3GPP metadata |
| Artist | artist | Performer | |
| Copyright | copyright | Copyright | |
| Encoder | encoder | AMR-WB | Codec identifier |

### 8.7 Quality / Fidelity Decision Table
| Use Case | Settings | Expected Size | Notes |
|----------|----------|---------------|-------|
| VoLTE (mobile voice) | 12.65 kbps default | ~20 KB/minute | Industry standard |
| High-quality voice | 23.85 kbps | ~36 KB/minute | Comfortable wideband |
| Wireless conferencing | 12.65–23.05 kbps | variable | Adaptive based on bandwidth |
| Archival (speech only) | 12.65 kbps | ~950 KB/hour | Sufficient for speech |
| Music segments in voice | 23.85 kbps | ~1.4 MB/hour | Music-coded quality |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
```
AMR-WB Native Seek Index: None

AMR-WB has no built-in seek table or index structure.
Seeking requires:

1. Bitrate-aware seeking (CBR assumption):
   byte_offset = (target_time_sec × bitrate_bps) / 8
   
2. For VBR (mode-switching) streams:
   - Parse frames sequentially until reaching target time
   - Average bitrate estimation for initial seek point
   - Frame-level scan for precise seeking

3. Container-based seeking (3GP/MP4):
   - Use the container's seek table (ctts, stsc, stsz boxes in MP4)
   - Map container time to AMR-WB frame offset
```

### 9.2 Gapless Playback Data
```
Encoder delay:   80 samples (5 ms look-ahead, included at start of output)
Padding:         80 samples (5 ms, due to look-ahead buffer)
Storage location: Not stored in AMR-WB bitstream; determined by decoder implementation

FFmpeg gapless flags: Not directly applicable to AMR-WB

Gapless handling:
  - The 5 ms look-ahead delay causes 80 samples of pre-ring at the start
  - Most implementations trim these 80 samples from output
  - The de-emphasis filter and HPF also contribute small delay
  - Total encoder-decoder latency: approximately 25 ms (20 ms frame + 5 ms look-ahead)
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | Frame-by-frame decoding |
| Algorithmic encoder delay | 25 ms total | 20 ms frame + 5 ms look-ahead |
| Live encoding feasible | Yes | Real-time encoding at 38 WMOPS complexity |
| HTTP progressive download | Yes | With container wrapper (3GP/MP4) |
| HTTP Live Streaming (HLS) | Yes | AMR-WB audio track in 3GP segments |
| DASH streaming | Yes | 3GP/DASH segment format |
| WebRTC / RTP transport | Yes | RFC 3267 / RFC 4867 payload format |
| Minimum decode buffer | 1 frame (320 samples) | 20 ms at 16 kHz |

### RTP Payload Format (RFC 3267 / RFC 4867)
```
RTP Header for AMR-WB:
  Payload type: 96 (dynamic, assigned via SDP)
  Timestamp: 16 kHz clock (each sample = 1 timestamp unit)
  Marker bit: Set to 1 for first frame of talkspurt (silence-to-speech transition)
  
AMR-WB Payload Structure (RFC 3267):
  [ CMR (4 bits) ] [ Table of Contents (ToC) entries... ]
  
  CMR (Codec Mode Request): Requests a specific bitrate mode from encoder
    0xF = No mode request (no request)
    0x0-0x8 = Request mode 0-8
    
  ToC Entry per frame:
    [ F (1 bit) ] [ Mode (4 bits) ] [ CMR (3 bits, stored if F=1) ]
    
    F = 0: Last frame in payload
    F = 1: More frames follow
```

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | Layout Name | AMR-WB Support | AMR-WB+ Support |
|----------|-------------|---------------|-----------------|
| 1 | Mono | Yes (native) | Yes |
| 2 | Stereo | No | Yes (independent L/R) |
| 5.1 | Surround | No | No |
| 7.1 | Surround | No | No |

### 11.2 Stereo Support
AMR-WB itself is mono-only. For stereo applications:
- **Option 1:** Encode left and right channels as separate AMR-WB streams (double bandwidth)
- **Option 2:** Use AMR-WB+ for stereo (extended codec with stereo support)
- **Option 3:** Downmix stereo to mono for AMR-WB encoding

### 11.3 AMR-WB+ Stereo Extensions
AMR-WB+ (the extended version) supports stereo through:
```
Stereo encoding in AMR-WB+:
  - Independent mono encoding of L and R channels
  - Optional frequency-domain stereo coupling for bitrate reduction
  - Bitrates up to 48 kbps total for stereo
  - Sample rates up to 48 kHz in AMR-WB+ mode
```

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 16-bit | Internal 32-bit fixed-point computation |
| Max sample rate | 16000 Hz | AMR-WB native; AMR-WB+ extends to 48000 Hz |
| Float support | No | Fixed-point only |
| DSD support | No | Not applicable to speech codec |
| 20-bit support | Partial | May be accepted with truncation |
| 24-bit support | Partial | Accepted but truncated to 16-bit |

AMR-WB is a **wideband speech codec**, not a high-resolution audio codec. It is optimized for speech at 16 kHz sample rate, not for music reproduction. For high-quality music, use AAC, FLAC, Opus, or other general-purpose audio codecs.

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

| Platform / API | Encode | Decode | FFmpeg Flag | Notes |
|----------------|--------|--------|-------------|-------|
| iOS AudioToolbox | Yes | Yes | Native framework | Hardware decode on Apple SoCs |
| Android MediaCodec | Yes | Yes | amrwbdec / amrwbenc | Hardware decode on most Android devices |
| Qualcomm Hexagon DSP | Yes | Yes | Hardware DSP | Dedicated voice processing |
| MediaTek DSP | Yes | Yes | Hardware DSP | Dedicated voice processing |
| Windows Media Foundation | Yes | Yes | MFT_AMRWB | Software fallback available |
| FFmpeg (software) | No* | Yes | amrwb, libopencore_amrwb | Encoder often unavailable |

*Many FFmpeg builds exclude the AMR-WB encoder due to patent licensing restrictions. The decoder is universally available.

```bash
# iOS AudioToolbox encoding (via AVFoundation):
# Use AVAudioEngine with AVAudioConverter, or AVAssetWriter with AVFileType.amr

# Android MediaCodec encoding:
# MediaFormat.createAudioFormat("audio/amr-wb", 16000, 1)

# Hardware decode verification on Android:
ffmpeg -hwaccel mediacodec -i input.awb -c:a pcm_s16le output.wav
```

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Encoder-Specific Issues
| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| AMR-WB encoder not available | All standard builds | Patent restrictions; use external encoder or accept decoder-only |
| AMR-WB in MP4 container seeking | All versions | Seek to nearest keyframe, not sample-accurate |
| 3GP demuxing with interleaved audio/video | Older versions | Use `-max_interleave_delta` parameter |

### 14.2 Interoperability Issues
- **Interoperant encoders:** AMR-WB from different manufacturers (Qualcomm, MediaTek, Ericsson) should be fully interoperable due to standardized fixed-point reference code. Minor quality variations are possible due to different floating-point implementations.
- **Mode switching:** Rapid mode switching (e.g., every frame) can cause audible artifacts. Most implementations smooth transitions.
- **DTX/CNG quality:** The comfort noise generated during DTX periods varies significantly between implementations. Some produce audible artifacts.
- **High-band in 23.85 kbps mode:** Not all decoders correctly implement the high-band extension (6400–7000 Hz). This can cause slightly different frequency response compared to the reference.

### 14.3 Edge Cases to Handle in Converter
- **Empty file / 0-sample stream:** Output silence, no error.
- **File < 1 frame of audio:** Output whatever samples are available, may be corrupted.
- **All-silence audio:** Correctly encoded as SID (Silence Insertion Descriptor) frames in DTX mode, or full frames of silence.
- **DC offset (non-zero mean):** DC removal in pre-processing handles this; output should be DC-free.
- **Full-scale sine (0 dB):** AMR-WB fixed-point implementation clips at internal computation limits. Output may be slightly distorted for very loud signals.
- **File with corrupt header:** Decoder reports error, returns partial output with frame erasure concealment.
- **Truncated file:** Last incomplete frame handled as corrupted frame, concealment applied.
- **Sample rate not 16 kHz:** FFmpeg's libswresample automatically resamples to 16 kHz before encoding (if encoder is available).
- **Channel count > 1:** Stereo input is downmixed to mono before AMR-WB encoding.

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM AMR-WB

| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → FLAC | `ffmpeg -i in.awb -c:a flac -compression_level 8 out.flac` | Tags from container | Lossless re-encode of decoded audio |
| → WAV | `ffmpeg -i in.awb -c:a pcm_s16le out.wav` | None (bare bitstream) | Uncompressed, 16-bit/16kHz |
| → MP3 | `ffmpeg -i in.awb -c:a libmp3lame -q:a 2 out.mp3` | ID3v2 tags | Generation loss |
| → AAC | `ffmpeg -i in.awb -c:a aac -b:a 128k out.m4a` | Container tags | Generation loss |
| → Opus | `ffmpeg -i in.awb -c:a libopus -b:a 64k out.opus` | Container tags | Generation loss, -application voip recommended |
| → OGG Vorbis | `ffmpeg -i in.awb -c:a libvorbis -q:a 6 out.ogg` | Vorbis Comments | Generation loss |

### 15.2 Converting TO AMR-WB

| Source | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| WAV 16kHz mono → | `ffmpeg -i in.wav -c:a libopencore_amrwb -b:a 12.65k out.awb` | None | Requires mono 16kHz input |
| WAV 44.1kHz → | `ffmpeg -i in.wav -ar 16000 -ac 1 -c:a libopencore_amrwb out.awb` | None | Auto-resample to 16kHz |
| MP3 → | `ffmpeg -i in.mp3 -ar 16000 -ac 1 -c:a libopencore_amrwb out.awb` | None | Generation loss, then AMR-WB loss |
| AAC → | `ffmpeg -i in.m4a -ar 16000 -ac 1 -c:a libopencore_amrwb out.awb` | None | Generation loss, then AMR-WB loss |
| Opus → | `ffmpeg -i in.opus -ar 16000 -ac 1 -c:a libopencore_amrwb out.awb` | None | Double re-encode: Opus→PCM→AMR-WB |

### 15.3 Lossless Round-Trip Verification
```bash
# AMR-WB is lossy — no true lossless round-trip is possible
# However, you can verify bit-exact decode consistency:

# Encode to AMR-WB
ffmpeg -i original.wav -c:a libopencore_amrwb -b:a 23.85k temp.awb

# Decode back
ffmpeg -i temp.awb -c:a pcm_s16le decoded.wav

# Generate framemd5 for comparison
ffmpeg -i original.wav -map 0:a -f framemd5 original.md5
ffmpeg -i decoded.wav -map 0:a -f framemd5 decoded.md5

# Compare (will show differences due to AMR-WB lossy compression)
diff original.md5 decoded.md5
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| 3GPP Reference | C | 3GPP MSS license | Reference | Reference | 3GPP website (specs) |
| OpenCore AMR | C | Apache 2.0 | Production | Reference | opensource.google.com |
| FFmpeg (libavcodec) | C | LGPL 2.1+ | Not included* | 9/10 | https://ffmpeg.org |
| VoiceAge AMR-WB | C/C++ | Proprietary | Production | Production | voiceage.com |
| Android (AOSP) | Java/JNI | Apache 2.0 | Hardware | Hardware | source.android.com |
| iOS Core Audio | C | Proprietary | Hardware | Hardware | developer.apple.com |

*Many FFmpeg builds exclude the AMR-WB encoder due to patent licensing restrictions. Decoder is always available.

### Build Instructions (OpenCore AMR from source)
```bash
# Clone OpenCore AMR repository
git clone https://github.com/nicknisi/opencore-amr.git
cd opencore-amr

# Configure and build
./configure --enable-shared --disable-static
make -j$(nproc)
sudo make install

# Link into your project:
# LDFLAGS += -L/usr/local/lib -lopencore-amrwb
# CFLAGS  += -I/usr/local/include/opencore-amrnb
```

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specifications
- **ITU-T G.722.2 (2003):** Wideband coding of speech at around 16 kbit/s using Adaptive Multi-Rate Wideband (AMR-WB) — https://www.itu.int/rec/T-REC-G.722.2
- **3GPP TS 26.190:** AMR-WB speech codec; speech processing; transcoding functions
- **3GPP TS 26.194:** AMR-WB speech codec; Voice Activity Detector (VAD)
- **RFC 3267 (2002):** Real-Time Transport Protocol (RTP) Payload Format for AMR and AMR-WB
- **RFC 4867 (2007):** RTP Payload Format for AMR-WB with Annex Header

### Technical Resources
- FFmpeg AMR-WB decoder: `ffmpeg -h decoder=amrwb`
- FFmpeg AMR-WB encoder: `ffmpeg -h encoder=libopencore_amrwb`
- AMR-WB Wikipedia: https://en.wikipedia.org/wiki/Adaptive_Multi-Rate_Wideband
- VoiceAge AMR-WB page: https://voiceage.com/AMR-WB.G.722.2.html
- 3GPP specifications portal: https://www.3gpp.org/specifications/specifications

### Academic Papers
- Salami, R. et al., "AMR Wideband Speech Codec: Evolution from 3GPP to ITU-T," *IEEE Transactions on Audio, Speech, and Language Processing*, 2006
- Vary, P. et al., "Speech and Audio Coding," *Springer*, 2018 — Chapter on CELP and wideband coding

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] Verify `ffmpeg -decoders | grep -i amrwb` shows decoder availability
- [ ] Note that AMR-WB encoder may not be available (patent restrictions)
- [ ] If encoder is needed, investigate third-party libraries (OpenCore AMR, VoiceAge)
- [ ] Handle platform differences in AMR-WB availability

### Encoding Pipeline
- [ ] Resample input to 16 kHz if needed using libswresample
- [ ] Convert to mono if stereo input is provided
- [ ] Convert to S16 sample format for encoder compatibility
- [ ] Set frame_size = 320 (fixed for AMR-WB)
- [ ] Implement flush/drain at end of stream (send NULL frame)
- [ ] Pack AMR-WB frames with correct header byte (mode + quality flag)

### Decoding Pipeline
- [ ] Implement AMR-WB frame header parsing (sync word '#', mode field)
- [ ] Handle AVERROR(EAGAIN) in receive_frame loop correctly
- [ ] Flush decoder at EOF (send NULL packet)
- [ ] Resample output to desired rate (typically 16 kHz or 48 kHz)
- [ ] Handle frame erasure concealment for corrupted frames

### Metadata
- [ ] Read metadata from container (3GP, MP4) when present
- [ ] Map 3GPP metadata fields to standard keys
- [ ] Write metadata to output container
- [ ] Handle absence of metadata in bare .awb files

### Quality & Verification
- [ ] Note that AMR-WB is inherently lossy — no bit-exact verification possible
- [ ] Use PESQ or POLQA for subjective quality measurement
- [ ] Test with: silence, full-scale, speech, music-coded segments
- [ ] Test mode switching behavior across different bitrates

### Edge Cases
- [ ] Handle files with corrupt or missing headers
- [ ] Handle files with 0 samples (return silence)
- [ ] Handle truncated files gracefully
- [ ] Handle sample rate mismatch (auto-resample)
- [ ] Handle channel count mismatch (auto-downmix)
- [ ] Handle very short files (< 1 frame)

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
