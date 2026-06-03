# Windows Media Audio Professional (WMA Pro) — Deep Technical Reference
> **Category:** Lossy
> **File Extensions:** `.wma`, `.asf`
> **MIME Types:** `audio/x-wma-pro`, `audio/wma`, `audio/x-ms-wma`
> **Standardization Body:** Microsoft
> **Primary Specification:** Microsoft ASF Specification + proprietary codec (not publicly documented)
> **Patent Status:** Patented — expires [varies by patent; Microsoft holds core patents]
> **License:** Proprietary (royalty-bearing; licensing via Microsoft)
> **Current Version:** WMA 10 Pro (also known as WMA 9 Professional)
> **Active Development:** No — last release ~2003–2005, deprecated in favor of successor codecs

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Microsoft Corporation
- **Year Created:** 2003 (Windows Media 9 Series)
- **Original Purpose:** Provide a high-quality lossy audio codec for multichannel surround sound and high-resolution audio, competing with AAC, Dolby Digital, and DTS
- **Problem with Predecessors:** WMA Standard (v1/v2) limited to stereo, 16-bit, 48 kHz; no multichannel support; audio quality not competitive with AAC at equivalent bitrates

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| WMA 9 Professional | 2003 | Initial release; multichannel, 24-bit, 96 kHz support |
| WMA 10 Professional | 2004 | Improved quality; profile levels P1, P2, P3 |
| WMA 10 Professional P1 | 2004 | Low-complexity profile for mobile devices |
| WMA 10 Professional P2 | 2004 | Medium-complexity profile |
| WMA 10 Professional P3 | 2004 | High-complexity profile for maximum quality |
| WMA 11 | 2008 | Minor improvements; Windows Media Player 11 integration |

### 1.3 Current Adoption
- **Primary use cases today:** Home theater systems, gaming (Xbox 360 supported WMA Pro), streaming with appropriate hardware, archival with multichannel
- **Platforms with native support:** Windows (native), Xbox 360 (native), some Blu-ray players, PS3 (via firmware), Linux (via FFmpeg/GStreamer), iOS/Android (limited/none)
- **Major services using this format:** Historically: Netflix (early), Xbox Live Marketplace, some streaming radio. As of 2024: Very limited commercial use; Qobuz offers WMA Pro as download option
- **Hardware support:** Many AV receivers and Blu-ray players from early-to-mid 2010s; declining support in favor of AAC/FLAC
- **Status:** Declining/Legacy — no new content encoding; existing hardware may not decode newer WMA Pro profiles

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Transform codec (perceptual audio coding)
- **Core algorithm:** MDCT (Modified Discrete Cosine Transform) with variable block length
- **Loss mechanism:** Psychoacoustic masking, subband/bit allocation, quantization, stereo coding
- **Frame-based vs sample-based:** Frame-based with variable-length subframes within each frame
- **Fixed vs variable frame size:** Variable; supports both CBR and VBR modes; block sizes adapt to audio content

### 2.2 High-Level Encoding Flow (Block Diagram)
```
Input PCM Samples (up to 8 channels, 24-bit, 96 kHz)
      │
      ▼
[Pre-processing: DC removal, channel reordering]
      │
      ▼
[Subband Analysis: QMF filterbank decomposition]
      │
      ▼
[MDCT: Per-subband MDCT transform with variable block sizes]
      │
      ▼
[Transient Detection: Short vs long block decision]
      │
      ▼
[Psychovisual/ Psychoacoustic Model: Masking thresholds per band]
      │
      ▼
[Bit Allocation: Per-band adaptive bit assignment]
      │
      ▼
[Quantization: Non-uniform scalar quantization with noise shaping]
      │
      ▼
[Huffman Coding: Vector Huffman coding of coefficients]
      │
      ▼
[Stereo Decorrelation: Joint stereo, channel coupling]
      │
      ▼
[Bitstream Packing: Frame headers, subframe layout, scale factors]
      │
      ▼
Output WMA Pro Bitstream (within ASF container)
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | Variable: 512–8192 samples | Longer for long blocks; ~46–170 ms at 48 kHz |
| Block sizes | 2^N samples: N=7–13 | Variable per subframe |
| Max channels | 8 (7.1 surround) | Mono, stereo, 5.1, 6.1, 7.1 |
| Max bit depth | 24-bit | Internal precision may be higher |
| Max sample rate | 96000 Hz | |
| Bitrate range | 24–768 kbps (stereo) | Up to 1500 kbps for multichannel |
| Complexity | O(N log N) | Encode significantly heavier than decode |
| Profile levels | P1 (mobile), P2 (desktop), P3 (high-quality) | Complexity/compression trade-offs |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
WMA Pro is not a standalone file format; it is always contained within ASF (Advanced Systems Format). The audio stream is identified by the codec FourCC.

```
WMA Pro (ASF Audio Stream):
  FourCC: 0x0162 (WMA Professional)
```

### 3.2 ASF Container Structure (Relevant Fields)
```
ASF File Header (GUID: 8CABDCA1-A947-11CF-8EE4-00C00C205365):
  ├── Object Size (uint64 LE)
  ├── Number of Header Objects (uint32)
  └── Header Objects[]
       ├── File Properties Object
       │    ├── File ID (GUID)
       │    ├── File Size (uint64)
       │    ├── Creation Date (uint64)
       │    ├── Data Packets Count (uint64)
       │    └── Play Duration (uint64)
       ├── Stream Properties Object (audio stream)
       │    ├── Stream Type: {AUDIO_OBJECT}
       │    ├── Error Correction Type
       │    ├── Time Offset (uint64)
       │    ├── Type-Specific Data Length (uint32)
       │    └── Type-Specific Data (WMA Pro WAVEFORMATEX)
       │         ├── wFormatTag: 0x0162 (WMA Pro)
       │         ├── nChannels: 1–8
       │         ├── nSamplesPerSec: 8000–96000
       │         ├── nAvgBytesPerSec: bitrate/8
       │         ├── nBlockAlign: varies
       │         ├── wBitsPerSample: 16 or 24
       │         ├── cbSize: variable (up to 10 bytes)
       │         └── Extra Data (WMA Pro codec initialization)
       ├── Content Description Object (metadata)
       └── other optional objects...
```

### 3.3 WMA Pro Frame Structure
```
WMA Pro Frame (within ASF Packet):
  ├── Frame Header
  │    ├── Frame size (variable)
  │    ├── Samples in frame (from extradata)
  │    └── Configuration flags
  ├── Subframes[] (1–N per frame)
  │    ├── Subframe header
  │    │    ├── Block size (2^B samples)
  │    │    ├── Channel transform info
  │    │    └── Scale factors
  │    ├── Scale factors (per band, per channel)
  │    └── Spectral coefficients (Huffman-coded)
  └── Frame-level side information
```

### 3.6 WMA Pro Profile Specifications
WMA Pro defines three profile levels for different complexity and quality requirements.

| Parameter | P1 (Mobile) | P2 (Desktop) | P3 (High-Quality) |
|-----------|-------------|---------------|-------------------|
| Max Bitrate (stereo) | 384 kbps | 768 kbps | 768 kbps |
| Max Bitrate (5.1) | 512 kbps | 1500 kbps | 1500 kbps |
| Max Block Size | 4096 | 8192 | 8192 |
| Max Bands | 29 | 29 | 29 |
| VLC Bits | 8 | 9 | 9 |
| Prediction Order | 512 | 1024 | 2048 |
| Channel Transform | Limited | Full | Full |
| MS Stereo | Yes | Yes | Yes |
| Channel Coupling | No | Yes | Yes |

### 3.7 Vector Huffman Coding Tables
WMA Pro uses vector Huffman coding with multiple codebooks for efficient coefficient encoding.

```
Codebook Types:
  CB1:  1 coefficient per symbol
  CB2:  2 coefficients per symbol (vec2)
  CB4:  4 coefficients per symbol (vec4)
  
Each codebook has:
  - Escape code for out-of-range values
  - Magnitude categories
  - Sign bits

Magnitude Categories:
  0:    Values in range [-1, 1]
  1:    Values in range [-3, 3]
  2:    Values in range [-7, 7]
  3:    Values in range [-15, 15]
  4:    Values in range [-31, 31]
  5:    Values in range [-63, 63]
  6:    Values in range [-127, 127]
  7:    Values in range [-255, 255]
  8:    Escape + raw 8-bit value
  9:    Escape + raw 16-bit value
  10+:  Escape + raw 32-bit value
```

### 3.8 Channel Transform Modes
WMA Pro supports various channel transform modes for efficient multichannel encoding.

```
Channel Transform Types:
  Type 0: No transform (independent channels)
  Type 1: M/S stereo (for stereo pairs)
  Type 2: LFE channel (low-frequency effects)
  Type 3: Channel coupling with prediction
  
Channel Ordering (5.1 example):
  Index 0: Front Left (FL)
  Index 1: Front Right (FR)
  Index 2: Center (C)
  Index 3: LFE
  Index 4: Surround Left (SL)
  Index 5: Surround Right (SR)
  
Transform Priority:
  1. LFE flag (if channel is LFE)
  2. M/S stereo pair (FL/FR or SL/SR)
  3. Independent channels
```

### 3.9 ASF Stream Configuration Object
The WMA Pro codec configuration is stored in the extradata of the ASF stream properties object.

```
Extradata Structure (10 bytes):
  Offset  Size  Field                   Description
  ------  ----  ---------------------   ----------------------------
  0       2     Samples Per Frame       Samples encoded per frame
  2       2     Maximum Packet Size      Maximum packet size
  4       2     Max Bitrate             Maximum bitrate in bps
  6       2     Average Bitrate          Average bitrate in bps
  8       1     Channel Configuration   Channel layout flags
  9       1     Codec Profile           0=P1, 1=P2, 2=P3
```

### 3.5 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Unsigned integer | No | Not supported |
| 16-bit | Signed integer | Yes | Primary input format |
| 20-bit | Signed integer | Yes | Supported in later versions |
| 24-bit | Signed integer | Yes | Primary high-resolution format |
| 32-bit | Signed integer | No | Not supported |
| 32-bit | IEEE float | No | Not supported |
| 64-bit | IEEE float (double) | No | Not supported |

### 3.5.1 Supported Sample Rates
| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 8000 | Telephone | Yes | Voice mode |
| 11025 | — | Yes | |
| 16000 | Wideband voice | Yes | |
| 22050 | — | Yes | |
| 32000 | Broadcast | Yes | |
| 44100 | CD audio | Yes | Primary rate |
| 48000 | Professional | Yes | Highest standard |
| 88200 | 2× CD | Yes | High-resolution |
| 96000 | High-res max | Yes | Maximum supported |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Pre-Processing Stage
- **DC offset removal:** Subtraction of running mean (high-pass filter with configurable cutoff)
- **Pre-emphasis filter:** Optional; configurable per profile
- **Windowing function:** Sine window applied before MDCT
- **Channel remapping:** Rearrange input channels to standard order for encoding
- **Level normalization:** Peak normalization to prevent clipping in fixed-point domain

### 4.2 Analysis / Transform Stage

#### Transform Type: MDCT (Modified Discrete Cosine Transform)
```
Parameters:
  Window size:     Variable: 64–8192 samples (power of 2)
  Overlap:         50% (MDCT standard)
  Window function: Sine window
  FFT size:        Matches window size

Block size selection (per subframe):
  - Long blocks (2048–8192 samples): Steady-state audio
  - Short blocks (64–512 samples): Transients
  - Block switching based on transient detection threshold
```

**Mathematical definition (MDCT forward):**
```
MDCT[k] = Σ(n=0 to 2N-1) x[n] · w[n] · cos(π/N · (n + N/2 + 1/2) · (k + 1/2))
  where:
    k  = frequency bin index (0 to N-1)
    N  = half-frame size (window_size / 2)
    w[n] = sine window function
    x[n] = input samples after windowing
```

**Inverse MDCT (IMDCT):**
```
IMDCT[n] = (1/N) · Σ(k=0 to N-1) X[k] · cos(π/N · (k + 1/2) · (n + N/2 + 1/2))
  for n = 0 to 2N-1
```

#### QMF (Quadrature Mirror Filter) Analysis
```
64-band polyphase filterbank [NEEDS VERIFICATION]
  or
32-band polyphase filterbank with MDCT [VERIFIED]
Each subband: SR/64 Hz bandwidth (for 64-band)
Critical for frequency-domain bit allocation
```

### 4.3 Psychoacoustic Model (Lossy Only)

#### Model Version
- **Model:** Custom Microsoft psychoacoustic model (improved over WMA Standard)
- **Analysis window:** Adaptive; matches current block size
- **ATH (Absolute Threshold of Hearing):** Based on ISO 3897/ITU-R BS.1387

#### Masking Thresholds
```
Simultaneous Masking:
  Masking slope (upward):   ~10 dB/Bark
  Masking slope (downward): ~25 dB/Bark
  Spreading function:       Advanced spreading model
  Tonality detection:       Yes; affects masking efficiency

Temporal Masking:
  Pre-masking:  ~20 ms
  Post-masking: ~150 ms
  Decay:        Exponential decay
```

#### Bit Allocation Algorithm
```
1. Compute FFT of input block
2. Apply bark spectral integration
3. Compute masking threshold T[k] for each frequency bin
4. Detect tonal vs noise components
5. Compute signal-to-mask ratio: SMR[k] = Energy[k] - T[k]
6. Allocate bits per subband using water-filling:
   - Iterative allocation based on SMR
   - Priority: maximize audible quality per bit
   - Constraints: bit budget, maximum bit per band
7. Quantize coefficients in each subband
8. Encode with vector Huffman coding
```

### 4.4 Quantization
- **Type:** Non-uniform scalar quantization with noise shaping
- **Step sizes:** Variable; determined by bit allocation
- **Dequantization formula:** `reconstructed = sign(code) × |code|^(4/3) × 2^(gain_factor)` [NEEDS VERIFICATION]
- **Noise shaping:** Applied implicitly via psychoacoustic model and bit allocation

### 4.5 Stereo and Multichannel Encoding Modes
| Mode | Description | Condition for Selection | Channels |
|------|-------------|------------------------|----------|
| Independent | Each channel encoded separately | Default; high bitrates | Any |
| MS Stereo | M=(L+R)/2, S=(L-R)/2 | Per-frame decision | Stereo only |
| Channel Coupling | Group channels, transmit shared + residuals | Complex scenes | 5.1, 7.1 |
| LFE Channel | LFE encoded with reduced bandwidth | Always for LFE | 5.1, 7.1 |

**Multichannel Decorrelation:**
- WMA Pro uses inter-channel prediction for multichannel encoding
- LMS (Least Mean Square) prediction for channel decorrelation
- Reduces redundancy between channels (e.g., front left/right similarity)

### 4.6 Entropy / Lossless Coding Stage
```
Method: Vector Huffman coding (improved over WMA Standard)

Huffman codebook selection:
  - Multiple codebooks for different coefficient distributions
  - Vector coding: groups of 1, 2, or 4 coefficients per symbol
  - Selection method: Per-band codebook index in bitstream
  - Escape code: For out-of-range values

Scale factor coding:
  - Differential coding: Scale factors transmitted as deltas
  - Huffman-coded: Variable-length delta sequences
  - Joint coding: Scale factors shared across channels when possible
```

### 4.7 Encoder Settings / Quality Modes

#### Profile Levels
| Profile | Complexity | Max Bitrate | Typical Use |
|---------|------------|-------------|-------------|
| P1 (Mobile) | Low | 384 kbps (stereo) | Mobile devices, limited CPU |
| P2 (Desktop) | Medium | 768 kbps (stereo) | Desktop, streaming |
| P3 (High-Quality) | High | 768 kbps (stereo) | Maximum quality |

#### Bitrate Table (Stereo)
| Bitrate (kbps) | Quality | Typical Use |
|----------------|---------|-------------|
| 24–32 | Very low | Voice, extreme bandwidth saving |
| 48–64 | Low | Low-bandwidth streaming |
| 96–128 | Medium | Standard streaming |
| 160–192 | High | High-quality streaming |
| 256–384 | Very high | Near-transparent |
| 512–768 | Maximum | Maximum quality, archival lossy |

#### Bitrate Table (Multichannel)
| Bitrate (kbps) | Channels | Typical Use |
|----------------|----------|-------------|
| 128–192 | 5.1 | Standard surround |
| 256–384 | 5.1 | High-quality surround |
| 512–768 | 5.1 | Maximum surround quality |
| 768–1500 | 7.1 | High-quality 7.1 |

#### VBR Mode Parameters
```
WMA Pro VBR Modes:
  Mode 0:  CBR (Constant Bitrate)
  Mode 1:  VBR (Quality-based)
  Mode 2:  VBR (Peak bitrate constrained)
  Mode 3:  VBR (Average bitrate constrained)

Quality Levels:
  Q0:      Highest quality (largest file)
  Q1:      High quality
  Q2:      Medium quality
  Q3:      Low quality
  Q4:      Voice quality
```

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. ASF container parsed for audio stream properties
2. WMA Pro extradata extracted (codec initialization data)
3. Frame sync within ASF packets
4. Validate frame by checking:
   a. Subframe structure matches extradata configuration
   b. Huffman decode succeeds
   c. Dequantized values within valid range
5. Frame boundary determined by frame_size field in extradata
```

#### Seeking
- **ASF seeking:** Index-based seeking using ASF index object
- **Seek table:** Stored in optional Data Object Index; entries point to packet offsets
- **Precision:** Millisecond accuracy (not sample-accurate for WMA Pro)

### 5.2 Core Decode Pipeline
```
1. Parse ASF packet structure
   ├── ASF packet header (variable size)
   ├── Payload parse flags
   └── Payload data (WMA Pro frame payload)

2. Parse WMA Pro frame
   ├── Read frame size from packet header
   ├── Parse subframe count and configuration
   └── Loop through subframes

3. For each subframe:
   a. Read block size and transform type
   b. Decode scale factors (differential + Huffman)
   c. Decode spectral coefficients (vector Huffman)
   d. Dequantize coefficients
   e. Apply inverse quantization

4. Inter-channel processing
   ├── If coupled channels: decode residual + prediction
   └── Apply LMS decorrelation

5. Inverse MDCT (IMDCT) per subframe
   └── Window + transform

6. Overlap-add (50% overlap between consecutive subframes)
   └── Reconstruct time-domain signal

7. QMF synthesis filterbank

8. Post-processing
   ├── Channel remapping
   ├── LFE low-pass filtering
   └── Clip to target bit depth
```

### 5.3 Error Concealment
- **Corrupt frame detection:** Huffman decode failure, invalid quantized values, CRC check
- **Concealment method:** 
  - Frame repeat for brief errors
  - Interpolation for longer error bursts
  - Muting after ~3 consecutive errors
- **Packet loss concealment:** WMA Pro supports PLC via ASF streaming protocol

### 5.4 Subframe Processing Details
Each WMA Pro frame contains one or more subframes with variable block sizes.

```
Subframe Header Structure:
  Byte Offset  Field               Bits    Description
  -----------  -----------------  ------  --------------------------------
  0            Block Size          6       Size index (64–8192 samples)
  6            Channel Transform   3       Transform type for this subframe
  9            Scale Factor Present 1      Scale factors follow header
  10           Spectral Data Size  12      Bits for spectral data
  22           Reserved            10      Reserved for future use
  
Subframe Priority:
  Subframes are processed in order of decreasing block size
  Large blocks (low frequency) processed first
  Small blocks (high frequency) processed later
```

### 5.5 Inter-Channel Decorrelation
WMA Pro uses LMS-based inter-channel prediction for multichannel content.

```
LMS Inter-Channel Prediction:
  For each pair of channels (e.g., FL, FR):
    1. Compute cross-correlation
    2. Estimate LMS filter coefficients
    3. Predict one channel from the other
    4. Encode residual (difference)
    
LMS Parameters:
  Filter Order:     4–16 coefficients
  Step Size:        Adaptive (0.01–0.1)
  Adaptation:       Forward and backward adaptation
  Convergence:      ~1000 samples
```

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** ASF (Advanced Systems Format)
- **Overhead:** ~1–3% (ASF header objects + packet headers)
- **Seeking in native container:** Yes — by index object (if present)
- **Multiple streams in native container:** Yes (audio + video + metadata)

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| ASF/WMA | Yes (native) | Yes | Full | WMA Pro = ASF with WMA Pro audio stream |
| AVI | Partial | Yes | Limited | FourCC 0x0162; limited metadata |
| Matroska/MKA | Partial | Yes | Full | Via LAV Filters or FFmpeg |
| MP4/M4A | No | — | — | Not supported |
| OGG | No | — | — | Not supported |
| WAV | No | — | — | Not supported |
| AIFF | No | — | — | Not supported |
| WebM | No | — | — | Not supported |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** ASF Content Description Object + Extended Content Description Object
- **Tag block location:** Within ASF header object, before audio data
- **Tag block identifier:** GUID-based objects in ASF

### 7.2 Standard Tag Fields — Complete Reference
| Tag Field | Internal Key (ASF) | Max Length | Character Encoding | Multi-value | Notes |
|-----------|------------------------|------------|-------------------|-------------|-------|
| Title | Title | 1024 bytes | UTF-16LE | No | |
| Artist | Author | 1024 bytes | UTF-16LE | No | |
| Album | WM/AlbumTitle | 1024 bytes | UTF-16LE | No | |
| Album Artist | WM/AlbumArtist | 1024 bytes | UTF-16LE | No | |
| Composer | WM/Composer | 1024 bytes | UTF-16LE | No | |
| Genre | WM/Genre | 1024 bytes | UTF-16LE | No | |
| Year / Date | WM/Year | 4 bytes | ASCII | No | 4-digit year |
| Track Number | WM/TrackNumber | 4 bytes | ASCII | No | Integer |
| Disc Number | WM/PartOfSet | 4 bytes | ASCII | No | Integer |
| Comment | Description | 1024 bytes | UTF-16LE | No | |
| Lyrics | WM/Lyrics | 1024 bytes | UTF-16LE | No | |
| BPM | WM/BeatsPerMinute | 4 bytes | ASCII | No | Integer |
| Compilation | WM/Compilation | 1 byte | ASCII | No | 0 or 1 |
| Copyright | Copyright | 1024 bytes | UTF-16LE | No | |
| Publisher/Label | WM/Publisher | 1024 bytes | UTF-16LE | No | |
| Encoder | WM/EncodedBy | 1024 bytes | UTF-16LE | No | Software name |
| Encoder Settings | WM/EncodingSettings | 1024 bytes | UTF-16LE | No | Bitrate/profile params |
| ISRC | WM/ISRC | 12 bytes | ASCII | No | |
| MusicBrainz Track ID | MusicBrainz/TrackId | 36 bytes | ASCII | No | UUID format |
| MusicBrainz Artist ID | MusicBrainz/ArtistId | 36 bytes | ASCII | No | UUID format |
| MusicBrainz Album ID | MusicBrainz/AlbumId | 36 bytes | ASCII | No | UUID format |
| ReplayGain Track Gain | REPLAYGAIN_TRACK_GAIN | 16 bytes | ASCII | No | Format: "-6.20 dB" |
| ReplayGain Track Peak | REPLAYGAIN_TRACK_PEAK | 16 bytes | ASCII | No | Format: "0.998459" |
| ReplayGain Album Gain | REPLAYGAIN_ALBUM_GAIN | 16 bytes | ASCII | No | Format: "-5.80 dB" |
| ReplayGain Album Peak | REPLAYGAIN_ALBUM_PEAK | 16 bytes | ASCII | No | Format: "0.998459" |
| Cover Art | WM/Picture | up to 10 MB | Binary | No | Binary blob with MIME header |
| Arbitrary/Custom | WM/Custom* | 1024 bytes | UTF-16LE | Yes | Custom namespace |

### 7.3 Cover Art Storage
```
Cover art storage format in ASF/WMA Pro:
  Container type:  WM/Picture attribute (binary)
  Image formats:   JPEG (recommended), PNG
  Max image size:  10 MB (implementation limit)
  Max dimensions:  No hard limit (recommend 3000×3000)
  
  Binary layout of WM/Picture:
    [0x00]        Picture type (1 byte): 0=Other, 1=32x32, 2=Other, 3=Front cover, 4=Back cover
    [0x01-N]      MIME type (null-terminated UTF-16LE string)
    [N+1-M]       Description (null-terminated UTF-16LE string)
    [M+1-...]     Binary image data (JPEG/PNG)
```

### 7.4 Metadata Compatibility Matrix
| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| ASF Native | ✓ | ✓ | ✓ | Highest |
| ID3v1 | ✗ | ✗ | ✗ | N/A |
| ID3v2.3 | ✗ | ✗ | ✗ | N/A |
| APEv2 | ✗ | ✗ | ✗ | N/A |
| Vorbis Comments | ✗ | ✗ | ✗ | N/A |
| MP4 Atoms | ✗ | ✗ | ✗ | N/A |

**Conflict resolution:** Only ASF metadata is native; no conflicts.

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   wmapro         # used with -c:a
AV_CODEC_ID:        AV_CODEC_ID_WMAPRO
Format Name (CLI):  asf             # ASF container
Encoder(s):         NONE            # FFmpeg CANNOT encode WMA Pro
Decoder(s):         wmapro          # Native FFmpeg decoder
Muxer(s):           asf, asfw        # ASF muxer
Demuxer(s):         asf             # ASF demuxer
```

**CRITICAL:** FFmpeg supports WMA Pro decoding only. There is no native WMA Pro encoder in FFmpeg.

### 8.2 FFmpeg Decoding — Full CLI Reference

```bash
# Decode WMA Pro to raw PCM WAV
ffmpeg -i input.wma \
  -c:a pcm_s16le \
  output.wav

# Decode 24-bit WMA Pro to 24-bit WAV
ffmpeg -i input.wma \
  -c:a pcm_s24le \
  output.wav

# Decode with resampling
ffmpeg -i input.wma \
  -c:a pcm_s16le \
  -ar 48000 \
  output.wav

# Extract specific stream (multi-stream ASF)
ffmpeg -i input.wma -map 0:a:0 -c:a copy output.wma

# Probe format without decoding
ffprobe -v quiet -print_format json -show_streams -show_format input.wma

# Decode with channel downmix (7.1 to stereo)
ffmpeg -i input_7.1.wma \
  -c:a pcm_s16le \
  -ac 2 \
  output_stereo.wav
```

### 8.3 FFmpeg Decoding — libavcodec C API

```c
// ─── Complete decode program skeleton ────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.wma", NULL, NULL);
avformat_find_stream_info(fmt_ctx, NULL);

// Find audio stream
int audio_idx = av_find_best_stream(fmt_ctx, AVMEDIA_TYPE_AUDIO, -1, -1, NULL, 0);
AVStream *stream = fmt_ctx->streams[audio_idx];

// Verify codec is WMA Pro
if (stream->codecpar->codec_id != AV_CODEC_ID_WMAPRO) {
    fprintf(stderr, "Not a WMA Pro stream\n");
    exit(1);
}

// Find and open decoder
const AVCodec *dec = avcodec_find_decoder(stream->codecpar->codec_id);
AVCodecContext *dec_ctx = avcodec_alloc_context3(dec);
avcodec_parameters_to_context(dec_ctx, stream->codecpar);
avcodec_open2(dec_ctx, dec, NULL);

// Print stream info
printf("Channels: %d\n", dec_ctx->ch_layout.nb_channels);
printf("Sample rate: %d Hz\n", dec_ctx->sample_rate);
printf("Bitrate: %d bps\n", dec_ctx->bit_rate);

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
            // frm->pts = presentation timestamp
            printf("Frame: %d samples at %d Hz, format=%d\n",
                   frm->nb_samples, frm->sample_rate, frm->format);
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

### 8.4 FFmpeg Metadata Handling

```bash
# Read all metadata as JSON
ffprobe -v quiet -print_format json -show_format input.wma | jq .format.tags

# Write metadata (copy audio, update tags)
ffmpeg -i input.wma \
  -c copy \
  -metadata title="Song Title" \
  -metadata author="Artist Name" \
  -metadata album="Album Name" \
  -metadata year="2024" \
  -metadata genre="Electronic" \
  output.wma

# Strip all metadata
ffmpeg -i input.wma -c copy -map_metadata -1 output.wma

# Embed cover art
ffmpeg -i input.wma -i cover.jpg \
  -c copy \
  -metadata:s:v title="Album cover" \
  -disposition:v attached_pic \
  output.wma
```

#### FFmpeg Internal Metadata Key Mapping
| Standard Field | FFmpeg Key | ASF Native Key | Notes |
|----------------|------------|--------------------------|-------|
| Title | title | Title | |
| Artist | artist | Author | |
| Album | album | WM/AlbumTitle | |
| Album Artist | album_artist | WM/AlbumArtist | |
| Track Number | track | WM/TrackNumber | |
| Disc Number | disc | WM/PartOfSet | |
| Genre | genre | WM/Genre | |
| Date/Year | date | WM/Year | |
| Comment | comment | Description | |
| Composer | composer | WM/Composer | |
| Copyright | copyright | Copyright | |
| Encoder | encoder | WM/EncodedBy | Auto-set by encoder |

### 8.5 Quality / Fidelity Decision Table
| Use Case | Settings | Expected Size | Notes |
|----------|----------|---------------|-------|
| Audiophile lossy (stereo) | FFmpeg decode only | — | No FFmpeg encoder; use Microsoft tools |
| Audiophile lossy (5.1) | FFmpeg decode only | — | No FFmpeg encoder |
| Streaming source | FFmpeg decode only | — | No FFmpeg encoder |
| Multichannel archival | FFmpeg decode only | — | No FFmpeg encoder |

**Note:** FFmpeg cannot encode WMA Pro. For encoding, use:
- Windows Media Encoder (Microsoft, discontinued)
- Expression Encoder (Microsoft, discontinued)
- Third-party encoders with WMA Pro support

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
```
ASF Index Object:
  Location:    End of file (after data objects)
  GUID:        D6E229D3-35DA-11D1-9034-00A0C90349BE
  Entry size:  12 bytes
  Entry format:
    [0x00–0x03]  Packet Number (uint32)
    [0x04–0x07]  Packet Count (uint32)
    [0x08–0x0B]  Time Offset (uint32, ms)
  Index intervals: 1 second default (configurable)
```

### 9.2 Gapless Playback Data
```
Encoder delay:   Variable [NEEDS VERIFICATION]
Padding:         Variable [NEEDS VERIFICATION]
Storage location: ASF Presentation Time (implicit)

FFmpeg gapless flags:
  Input:  Automatic handling via ASF parsing
  Output: ASF index generated for seeking
  
Gapless detection:
  WMA Pro: No native gapless markers; use ASF presentation time
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | ASF designed for streaming |
| Algorithmic encoder delay | Variable: 46–170 ms | Depends on block size |
| Live encoding feasible | Yes | ASF supports live broadcast |
| HTTP progressive download | Yes | ASF over HTTP |
| HTTP Live Streaming (HLS) | No | Not natively supported |
| DASH streaming | Partial | Via ASF segmenting |
| WebRTC / RTP transport | Yes | Via RTSP/MMS |
| Minimum decode buffer | ~1 frame / ~10–170 ms | Depends on block size |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | Layout Name | Channel Order | FFmpeg Layout Constant | WMA Pro Support |
|----------|-------------|--------------|------------------------|-----------------| 
| 1 | Mono | C | AV_CHANNEL_LAYOUT_MONO | ✓ |
| 2 | Stereo | L, R | AV_CHANNEL_LAYOUT_STEREO | ✓ |
| 3 | 2.1 | L, R, LFE | AV_CHANNEL_LAYOUT_2POINT1 | ✓ |
| 4 | Quad | FL, FR, RL, RR | AV_CHANNEL_LAYOUT_QUAD | ✓ |
| 5 | 5.0 | FL, FR, C, SL, SR | AV_CHANNEL_LAYOUT_5POINT0 | ✓ |
| 6 | 5.1 | FL, FR, C, LFE, SL, SR | AV_CHANNEL_LAYOUT_5POINT1 | ✓ |
| 7 | 6.1 | FL, FR, C, LFE, BL, BC, BR | AV_CHANNEL_LAYOUT_6POINT1 | ✓ |
| 8 | 7.1 | FL, FR, C, LFE, BL, BR, SL, SR | AV_CHANNEL_LAYOUT_7POINT1 | ✓ |

### 11.2 Downmix Coefficients (5.1 → Stereo)
```
L_out = L + (C × 0.7071) + (SL × 0.7071)
R_out = R + (C × 0.7071) + (SR × 0.7071)
LFE:  Discarded (or mixed with coefficient 0.0–1.0 based on implementation)

FFmpeg automatic downmix:
  WMA Pro decoder automatically downmixes when stereo output is requested
  Coefficients depend on decoder implementation

FFmpeg manual downmix command:
ffmpeg -i input_5.1.wma \
  -af "pan=stereo|FL=FC+0.707*FC+0.707*BL|FR=FC+0.707*FC+0.707*BR" \
  output_stereo.wav
```

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 24-bit | Integer only |
| Max sample rate | 96000 Hz | |
| Float support | None | |
| DSD support | No | |
| 20-bit support | Yes | |
| 24-bit support | Yes | Primary high-resolution format |

```bash
# High-res WMA Pro decoding example
ffmpeg -i input_24bit_96k.wma \
  -c:a pcm_s24le \
  output_24_96.wav
```

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

| Platform / API | Encode | Decode | FFmpeg Flag | Notes |
|----------------|--------|--------|-------------|-------|
| Windows Media Foundation | Yes | Yes | Native | Microsoft's official API |
| FFmpeg native | No | Yes | None | Decode only; no encoder |
| NVIDIA NVENC | No | — | N/A | WMA Pro not supported |
| NVIDIA NVDEC | — | No | N/A | Not implemented |
| Intel QSV | No | Yes | `-hwaccel qsv` | Decode only |
| Apple VideoToolbox | No | No | N/A | No WMA Pro support |
| Android MediaCodec | No | Partial | N/A | OEM-dependent |
| VA-API (Linux) | No | Yes | `-hwaccel vaapi` | Decode only |

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Decoder-Specific Issues
| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| No encoder available | All | Use Microsoft encoder or transcoding |
| Seeking accuracy | Some versions | Use ASF index if present |
| High sample rate (>48k) decode | <4.0 | Update FFmpeg version |

### 14.2 Interoperability Issues
- **Microsoft encoder → FFmpeg decoder:** Generally compatible
- **FFmpeg encoder → Microsoft decoder:** Not applicable (no FFmpeg encoder)
- **Files with DRM:** FFmpeg cannot decode DRM-protected WMA Pro files
- **High-resolution files:** Some hardware decoders may not support 24-bit/96 kHz
- **Profile compatibility:** P1 profile may not decode on P2/P3-only decoders

### 14.3 Edge Cases to Handle in Converter
- **Empty file / 0-sample stream:** Output empty file; no error
- **File < 1 frame of audio:** Output partial frame; behavior depends on decoder
- **All-silence audio:** WMA Pro decoder handles silently
- **Corrupt frames:** Decoder outputs what it can; may contain artifacts
- **File with corrupt header:** FFmpeg may decode partially or fail with error
- **Truncated file:** Last frame may be corrupt; decoder outputs what it can
- **Sample rate not supported:** Error: "sample rate not supported"
- **Channel count not supported (>8):** Error: "unsupported number of channels"

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM WMA Pro

| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → FLAC | `ffmpeg -i in.wma -c:a flac -compression_level 8 out.flac` | All ASF tags → Vorbis Comments | Lossless decode |
| → ALAC | `ffmpeg -i in.wma -c:a alac out.m4a` | Partial (tag mapping) | Lossless decode |
| → MP3 | `ffmpeg -i in.wma -c:a libmp3lame -q:a 0 out.mp3` | Tags via ID3v2.3 | Generation loss |
| → AAC | `ffmpeg -i in.wma -c:a aac -b:a 256k out.m4a` | Partial | Generation loss |
| → Opus | `ffmpeg -i in.wma -c:a libopus -b:a 128k out.opus` | Vorbis Comments (recreated) | Generation loss |
| → WAV | `ffmpeg -i in.wma -c:a pcm_s16le out.wav` | RIFF INFO tags | Lossless decode |
| → WAV (24-bit) | `ffmpeg -i in.wma -c:a pcm_s24le out.wav` | RIFF INFO tags | Lossless decode |
| → OGG Vorbis | `ffmpeg -i in.wma -c:a libvorbis -q:a 6 out.ogg` | Vorbis Comments (recreated) | Generation loss |

### 15.2 Converting TO WMA Pro

**CRITICAL:** FFmpeg cannot encode WMA Pro. Use third-party tools for encoding.

| Source | Tool | Command | Notes |
|--------|------|---------|-------|
| FLAC → | Windows Media Encoder | GUI tool | Requires Windows |
| WAV → | Expression Encoder | GUI tool | Requires Windows |
| Any → | dBpoweramp | GUI tool | Cross-platform encoder wrapper |
| Any → | Steinberg Wavelab | GUI tool | Professional |

### 15.3 Lossless Round-Trip Verification
```bash
# Note: WMA Pro is lossy; no lossless round-trip possible
# For lossless: use WMA Lossless format instead

# Decode WMA Pro (lossy)
ffmpeg -i input.wma -c:a pcm_s16le decoded.wav

# Original WAV (lossy source decode only)
# Cannot compare due to lossy compression
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| FFmpeg native | C | LGPL 2.1+ | N/A (no encoder) | 9/10 | https://ffmpeg.org |
| Windows Media Format SDK | C/C++ | Proprietary | 10/10 | 10/10 | Microsoft |
| LAV Filters | C/C++ | LGPL 2.1+ | N/A | 9/10 | https://github.com/Nevcairiel/LAVFilters |
| Media Foundation | C/C++ | Proprietary | Yes | Yes | Windows native |

### Build Instructions (for bundling in converter app)
```bash
# Build FFmpeg from source with WMA Pro support
git clone https://github.com/FFmpeg/FFmpeg.git
cd FFmpeg
./configure --prefix=/usr/local \
  --enable-gpl --enable-libass --enable-libfreetype
make -j$(nproc)
make install

# Verify WMA Pro decode support:
ffmpeg -decoders | grep -i wmapro
# Output should show: wmapro

# Note: No encoder will be listed for WMA Pro
ffmpeg -encoders | grep -i wmapro
# Output: (empty)
```

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **ASF Specification:** Microsoft ASF Specification (revision 01.20.03, December 2004)
- **WMA Pro Codec:** No public specification; reverse-engineered by FFmpeg

### Technical Resources
- FFmpeg decoder options: `ffmpeg -h decoder=wmapro` or https://ffmpeg.org/ffmpeg-codecs.html
- FFmpeg ASF muxer: `ffmpeg -h muxer=asf` or https://ffmpeg.org/ffmpeg-formats.html
- Multimedia Wiki: https://wiki.multimedia.cx/index.php/Windows_Media_Audio
- Hydrogenaudio: https://wiki.hydrogenaudio.org/index.php?title=WMA
- Microsoft WMA Pro: https://learn.microsoft.com/en-us/windows/win32/medfound/about-the-windows-media-codecs

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] Identify FFmpeg `./configure` flags (default build includes WMA Pro decoder)
- [ ] Verify `ffmpeg -decoders` output confirms wmapro decoder is available
- [ ] **IMPORTANT:** Note that NO WMA Pro encoder exists in FFmpeg
- [ ] For encoding, recommend third-party tools or transcoding to alternative format
- [ ] Note platform restrictions (macOS/iOS may need transcoding)

### Decoding Pipeline
- [ ] ASF container parsing (use avformat)
- [ ] Verify stream codec_id is AV_CODEC_ID_WMAPRO
- [ ] Handle AVERROR(EAGAIN) in receive_packet loop correctly
- [ ] Flush decoder at EOF (send NULL packet)
- [ ] Handle variable sample format (S16, S16P, S32P)
- [ ] Handle variable channel layouts (1–8 channels)
- [ ] Handle high sample rates (up to 96 kHz)
- [ ] Skip encoder-delay samples at stream start (if detectable)

### Metadata
- [ ] Read ASF Content Description Object fields
- [ ] Read ASF Extended Content Description Object fields
- [ ] Map all tag fields through standard key mapping
- [ ] Read WM/Picture cover art
- [ ] Write all standard tag fields to ASF container (for copy operations)
- [ ] Embed cover art as WM/Picture binary
- [ ] Preserve ReplayGain tags through conversion
- [ ] Handle UTF-16LE encoding (ASF native)

### Quality & Verification
- [ ] Note: WMA Pro is lossy; cannot achieve bit-perfect output from lossy source
- [ ] Implement progress reporting from FFmpeg stats output
- [ ] Implement error detection and partial-file recovery
- [ ] Test with: silence, full-scale, multichannel, high-resolution files

### Edge Cases
- [ ] Handle files with corrupt or missing ASF headers
- [ ] Handle files with 0 samples
- [ ] Handle sample rate mismatch (trigger libswresample)
- [ ] Handle channel count mismatch (downmix if >2 for stereo target)
- [ ] Handle bit depth mismatch (convert from decoder output)
- [ ] Handle very short files (< 1 frame)
- [ ] Handle DRM-protected files (FFmpeg cannot decode)

### Encoding (Alternative Path)
- [ ] If encoding WMA Pro is required, recommend using external tools
- [ ] Document that FFmpeg cannot encode WMA Pro
- [ ] Provide guidance on Windows Media Encoder alternatives
- [ ] Consider transcoding to alternative format (AAC, Opus) as primary path

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
