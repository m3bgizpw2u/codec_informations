# Windows Media Audio Standard — Deep Technical Reference
> **Category:** Lossy
> **File Extensions:** `.wma`, `.asf`
> **MIME Types:** `audio/x-wma`, `audio/wma`, `audio/x-ms-wma`
> **Standardization Body:** Microsoft
> **Primary Specification:** Microsoft ASF Specification (revision 01.20.03, December 2004)
> **Patent Status:** Patented — expires [varies by patent; Microsoft holds core patents]
> **License:** Proprietary (royalty-bearing; licensing via Microsoft)
> **Current Version:** WMA 9 (final major version; considered legacy)
> **Active Development:** No — last release ~2003–2005, deprecated in favor of WMA Pro

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Microsoft Corporation
- **Year Created:** 1999
- **Original Purpose:** Provide a proprietary, MP3-competitive audio codec optimized for streaming and download delivery via Windows Media technologies
- **Problem with Predecessors:** MP3 licensing uncertainty (Fraunhofer patents), lack of integrated DRM, no native Windows integration

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| WMA v1 (0x0160) | 1999 | Initial release; 48–192 kbps stereo; ASF container |
| WMA v2 (0x0161) | 2000 | Improved psychoacoustics, better stereo coding, VBR support |
| WMA 9 Standard | 2003 | Part of Windows Media 9 Series; improved quality at all bitrates |
| WMA 9.1 | 2004 | Minor improvements; bug fixes |
| WMA 9.2 | 2005 | Final release; optimized encoder |

### 1.3 Current Adoption
- **Primary use cases today:** Legacy streaming (radio stations), archival content, Windows Media Player library compatibility
- **Platforms with native support:** Windows (native), macOS (Windows Media Player for Mac OS X), Linux (via FFmpeg/GStreamer), iOS/Android (limited/none)
- **Major services using this format:** Historically: Napster (original), Yahoo! Music, MTV, some internet radio. As of 2024: Qobuz offers WMA Lossless for download; most major services have abandoned WMA lossy in favor of AAC/MP3
- **Hardware support:** Many legacy DAPs and car audio systems; declining as devices move to AAC/FLAC
- **Status:** Declining/Legacy — superseded by WMA Pro and AAC; no new encoding tools from Microsoft

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Transform codec (perceptual audio coding)
- **Core algorithm:** MDCT (Modified Discrete Cosine Transform) with filterbank
- **Loss mechanism:** Psychoacoustic masking, subband/bit allocation, quantization
- **Frame-based vs sample-based:** Frame-based; frames contain N samples encoded as a unit
- **Fixed vs variable frame size:** Variable; supports both CBR and VBR modes

### 2.2 High-Level Encoding Flow (Block Diagram)
```
Input PCM Samples
      │
      ▼
[Pre-processing: DC removal, high-pass filter]
      │
      ▼
[Subband Splitting: QMF analysis filterbank (32 subbands)]
      │
      ▼
[MDCT: Per-subband MDCT transform]
      │
      ▼
[Psychovisual/ Psychoacoustic Model: Masking thresholds]
      │
      ▼
[Bit Allocation: Per-subband bit assignment]
      │
      ▼
[Quantization: Non-uniform scalar quantization]
      │
      ▼
[Huffman Coding: Entropy coding of quantized coefficients]
      │
      ▼
[MS Stereo Coding: Optional mid/side encoding]
      │
      ▼
[Bitstream Packing: Frame headers, scale factors]
      │
      ▼
Output WMA Bitstream (within ASF container)
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | ~185 ms | 512-sample lookahead at 44.1 kHz |
| Frame size | Variable: 512–8192 samples | Adaptive based on bitrate |
| Max channels | 2 (stereo) | Mono and stereo only; use WMA Pro for multichannel |
| Max bit depth | 16-bit | Internal precision may be higher |
| Max sample rate | 48000 Hz | Standard rates: 8, 11.025, 16, 22.05, 32, 44.1, 48 kHz |
| Bitrate range | 48–192 kbps (stereo) | Lower in mono; 128 kbps typical for streaming |
| Complexity | O(N log N) | Encode significantly heavier than decode |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
WMA Standard is not a standalone file format; it is always contained within ASF (Advanced Systems Format). The audio stream within ASF is identified by the codec FourCC.

```
WMAv1 (ASF Audio Stream):
  FourCC: 0x0160 (WMA version 1)

WMAv2 (ASF Audio Stream):
  FourCC: 0x0161 (WMA version 2)
```

### 3.2 ASF Container Structure (Relevant Fields)
```
ASF File Header (GUID: 8CABDCA1-A947-11CF-8EE4-00C00C205365):
  ├── Object Size (uint64 LE)
  ├── Number of Header Objects (uint32)
  └── Header Objects[]
       ├── File Properties Object
       ├── Stream Properties Object (audio stream)
       │    ├── Stream Type: {AUDIO_OBJECT}
       │    ├── Error Correction Type
       │    ├── Time Offset
       │    ├── Type-Specific Data Length
       │    └── Type-Specific Data (contains WAVEFORMATEX)
       │         ├── wFormatTag: 0x0160 (WMAv1) or 0x0161 (WMAv2)
       │         ├── nChannels: 1 or 2
       │         ├── nSamplesPerSec: 8000–48000
       │         ├── nAvgBytesPerSec: bitrate/8
       │         ├── nBlockAlign: varies
       │         ├── wBitsPerSample: 16
       │         ├── cbSize: 4 (WMAv1) or 6 (WMAv2)
       │         └── Extra Data (codec-specific initialization)
       ├── Content Description Object (metadata)
       └── other optional objects...
```

### 3.3 WMA Frame Header Layout (within ASF Packet Payload)
```
Bit Offset   Bit Width   Field Name              Type      Description
----------   ---------   --------------------    ------    ---------------------------
0            16          Sync Word              uint      0x0100 (frame sync)
16           16          Frame Size             uint      Depends on bitrate/sample rate
32           4           Bit Rate Index         uint      0–15 (see bitrate table)
36           2           Sample Rate Index      uint      0–3 (see sample rate table)
38           1           Pad Bits Flag          bool      1 if padding present
39           1           Encoder Version        bool      0=WMAv1, 1=WMAv2
40           5           Number of Channels     uint      0=mono, 1=stereo (encoded as n-1)
45           2           MS Stereo Flag         uint      0=no MS, 1=MS used
47           1           Encoder Delay          bool      Extra data follows
...          ...         Scale Factors         variable  Huffman-coded
...          ...         Spectral Data          variable  Huffman-coded
```

### 3.3.1 Bitrate Index Table
| Index | Value (kbps per channel) | Notes |
|-------|-------------------------|-------|
| 0x0 | Free bitrate | Variable; controlled by encoder |
| 0x1 | 5 (mono only) | Very low bitrate |
| 0x2 | 8 (mono only) | Low bitrate |
| 0x3 | 16 | |
| 0x4 | 22.05 kHz SR | Sample rate override |
| 0x5 | 24 | |
| 0x6 | 32 | |
| 0x7 | 44.1 kHz SR | Sample rate override |
| 0x8 | 48 | |
| 0x9 | 64 | |
| 0xA | 80 | |
| 0xB | 96 | |
| 0xC | 128 | CD-quality approximation |
| 0xD | 160 | High quality |
| 0xE | 192 | Maximum for WMA Standard |
| 0xF | Reserved | |

### 3.3.2 Sample Rate Index Table
| Index | Sample Rate (Hz) | Notes |
|-------|-----------------|-------|
| 0x0 | 8000 | Voice |
| 0x1 | 11025 | Low |
| 0x2 | 16000 | Wideband |
| 0x3 | 22050 | |

### 3.4 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Unsigned integer | No | Not supported by WMA Standard |
| 16-bit | Signed integer | Yes | Primary input format |
| 20-bit | Signed integer | Partial | Encoded within 24-bit container; limited support |
| 24-bit | Signed integer | Partial | Encoded within 24-bit container; limited support |
| 32-bit | Signed integer | No | Not supported |
| 32-bit | IEEE float | No | Not supported |
| 64-bit | IEEE float (double) | No | Not supported |

### 3.4.1 Supported Sample Rates
| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 8000 | Telephone | Yes | Voice mode |
| 11025 | — | Yes | |
| 16000 | Wideband voice | Yes | |
| 22050 | — | Yes | |
| 32000 | Broadcast | Yes | |
| 44100 | CD audio | Yes | Primary rate |
| 48000 | Professional | Yes | Highest supported |

---

### 3.4 Audio Frame Data Structure (Detailed)
The audio frame in WMA Standard contains the encoded spectral data following the frame header.

```
Byte Offset   Size    Field Name              Type        Description
------------  ------  ----------------------  ----------  --------------------------------
0x0000        4B      Frame Sync             uint32      Must be 0x01000000
0x0004        2B      Frame Size Low         uint16      Lower 16 bits of frame size
0x0006        1B      Bitrate Index          uint8       Index into bitrate table (0–15)
0x0007        1B      Sample Rate Index      uint8       Index into sample rate table
0x0008        2B      Frame Size High        uint16      Upper 16 bits of frame size + flags
0x000A        2B      Channel Configuration  uint16      Number of channels - 1
0x000C        2B      MS Stereo Flags        uint16      Bit 0: MS stereo enabled
0x000E        2B      Scale Factor Band Count uint16     Number of scale factor bands
0x0010        var     Scale Factors         variable    Huffman-coded scale factor deltas
0xXXXX        var     Spectral Data          variable    Huffman-coded MDCT coefficients
```

### 3.5 Scale Factor Band Structure
The scale factors are grouped into bands corresponding to critical bands of human hearing.

```
Band Number   Frequency Range (Hz)   MDCT Bins   Bit Allocation Bits
-----------   --------------------   ----------  --------------------
0             0 – 69               0–3         0–4
1             69 – 138              4–7         2–6
2             138 – 207             8–15        3–8
3             207 – 276             16–23       4–10
4             276 – 345             24–31       5–12
5             345 – 414             32–47       6–14
6             414 – 552             48–63       7–16
7             552 – 690             64–79       8–18
8             690 – 828             80–95       9–20
9             828 – 966             96–111      10–22
10            966 – 1104            112–127     11–24
...           ...                   ...         ...
```

### 3.6 Huffman Code Tables
WMA Standard uses multiple Huffman codebooks for different types of data.

```
Codebook    Purpose                    Entries    Max Code Length
---------   ------------------------   --------   ---------------
CB0         Zero coefficients          1          1 bit
CB1         Small positive values      16         4 bits
CB2         Small negative values      16         4 bits
CB3         Mixed small values        32         5 bits
CB4         Scale factor deltas       64         6 bits
CB5         Large magnitude values    128        7 bits
CB6         Escape + magnitude        256        8+ bits
```

### 3.7 ASF Packet Structure
ASF packets contain multiple WMA frames with overhead for streaming purposes.

```
Byte Offset   Size    Field Name              Type        Description
------------  ------  ----------------------  ----------  --------------------------------
0x0000        1B      Payload Count           uint8       Number of payloads in packet
0x0001        1B      Sequence                uint8       Packet sequence number
0x0002        2B      Unknown                 uint16      Reserved/unknown
0x0004        4B      Packet Length           uint32      Total packet size
0x0008        var     Padding                variable    Padding bytes
0xXXXX        var     Payloads               variable    One or more frame payloads
```

### 3.8 CRCs and Error Detection
```
CRC Coverage:
  Frame-level CRC:  16-bit CRC covering frame header and audio data
  Packet-level CRC:  32-bit CRC covering entire ASF packet (optional)
  
CRC Polynomial:  x^16 + x^12 + x^5 + 1  (CRC-16-CCITT)
CRC Initial:    0xFFFF
```

### 3.9 Bitstream Example (Hex Dump)
```
Example WMAv2 frame header (bytes):
  00: 01 00 00 00              ; Sync word (0x0100 in little-endian)
  04: 20 03                    ; Frame size = 0x0320 = 800 bytes
  06: 4B                      ; Bitrate index 9 = 64 kbps per channel
  07: 03                      ; Sample rate index 3 = 22050 Hz
  08: 01 00                    ; Channel config = stereo
  0A: 00 00                    ; MS stereo flags
  0C: 20 00                    ; 32 scale factor bands
```

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Detailed Pre-Processing Stage
- **DC offset removal:** Subtraction of running mean using 1-pole IIR high-pass filter with ~4 Hz cutoff. Formula: `y[n] = x[n] - x[n-1] + 0.999 × y[n-1]`
- **Pre-emphasis filter:** Optional 1-pole high-shelf with coefficient ~0.8; applied only to certain profiles [NEEDS VERIFICATION]
- **Windowing function:** Sine window: `w[n] = sin(π × (n + 0.5) / N)` where N is window size
- **Level normalization:** Peak normalization to prevent clipping in fixed-point domain
- **Stereo decorrelation pre-step:** Optional M/S (mid/side) transformation: `M = (L + R) / 2`, `S = (L - R) / 2`

### 4.2 Detailed QMF Filterbank Analysis
The 32-band QMF (Quadrature Mirror Filter) analysis filterbank decomposes the input signal into subbands.

```
QMF Filterbank Specifications:
  Number of bands:     32
  Bandwidth per band:  SR / 64 Hz
  Input window:        1024 samples
  Output:              32 subband samples per analysis block
  Filter coefficients: 512 taps (per band)
  
Mathematical operation:
  For each band k (0–31):
    X[k][n] = Σ(m=0 to 1023) x[m] × h[k][m mod 1024]
  where h[k] is the k-th QMF analysis filter
```

### 4.2 Analysis / Transform Stage

#### Transform Type: MDCT (Modified Discrete Cosine Transform)
```
Parameters:
  Window size:     2048 samples (long block) [NEEDS VERIFICATION]
                   512 samples (short block) [NEEDS VERIFICATION]
  Overlap:         50% (MDCT standard)
  Window function: Sine window
  FFT size:        2048-point FFT for analysis filterbank
```

**Mathematical definition (MDCT forward):**
```
MDCT[k] = Σ(n=0 to 2N-1) x[n] · w[n] · cos(π/N · (n + N/2 + 1/2) · (k + 1/2))
  where:
    k  = frequency bin index (0 to N-1)
    N  = half-frame size (512 for short, 2048 for long)
    w[n] = sine window function value at sample n
    x[n] = input samples after windowing
```

**Block switching (transient detection):**
- Long block: 2048 samples → triggered for steady-state audio
- Short block: 512 samples → triggered by transient detection
- Switch sequence: LONG → SHORT(×8) → LONG (transient handling)

#### QMF (Quadrature Mirror Filter) Analysis
```
32-band polyphase filterbank for initial subband decomposition
Each subband: SR/64 Hz bandwidth
Critical for frequency-domain bit allocation
```

### 4.3 Psychoacoustic Model (Lossy Only)

#### Model Version
- **Model:** Custom Microsoft psychoacoustic model (not ISO/IEC 11172-3)
- **Analysis window:** 1024-point FFT at input sample rate
- **ATH (Absolute Threshold of Hearing):** Approximated via table lookup

#### Masking Thresholds
```
Simultaneous Masking:
  Masking slope (upward):   ~15 dB/Bark
  Masking slope (downward): ~30 dB/Bark
  Spreading function:       1 Bark bandwidth

Temporal Masking:
  Pre-masking:  ~50 ms
  Post-masking: ~200 ms
  Decay:        Exponential
```

#### Bit Allocation Algorithm
```
1. Compute 1024-point FFT of input block
2. Apply bark spectral integration for spreading
3. Compute masking threshold T[k] for each frequency bin
4. Compute signal-to-mask ratio: SMR[k] = Energy[k] - T[k]
5. Allocate bits per subband:
   - Per-subband bit allocation table
   - Water-filling algorithm for bit distribution
   - Priority: maximize audible quality per bit
6. Quantize coefficients in each subband
7. Encode with Huffman coding
```

### 4.4 Quantization
- **Type:** Non-uniform scalar quantization (companded)
- **Step sizes:** Variable; determined by bit allocation
- **Dequantization formula:** `reconstructed = sign(code) × |code|^(4/3) × 2^(gain_factor)` [NEEDS VERIFICATION]
- **Noise shaping:** Applied implicitly via psychoacoustic model

### 4.5 Stereo Encoding Modes
| Mode | Description | Condition for Selection | Bitrate Range |
|------|-------------|------------------------|---------------|
| Independent Stereo | L and R encoded separately | Default; better at high bitrates | >128 kbps |
| MS Stereo | M=(L+R)/2, S=(L-R)/2 encoded | Per-frame decision by encoder | <128 kbps |
| Mono | Single channel encoded | Input is mono or forced | any |

**MS Stereo Benefit:** Reduces correlation in signals with centered vocals and ambient stereo content; saves bits.

### 4.6 Entropy / Lossless Coding Stage
```
Method: Huffman coding

Huffman codebook selection:
  - Number of codebooks: Multiple (different for spectral coefficients, scale factors)
  - Selection method: Per-subband codebook index encoded in bitstream
  - Escape code: For out-of-range values; prefix + magnitude + residual

Scale factor coding:
  - Differential coding: Scale factors transmitted as deltas
  - Huffman-coded: Short delta sequences
```

### 4.7 Encoder Settings / Quality Modes

#### For Lossy Codecs
| Quality Setting | Bitrate Range (stereo) | Intended Use Case | Transparent? |
|---|---|---|---|
| Voice | 16–32 kbps | Speech, low bandwidth | No |
| Low | 48–64 kbps | Internet radio, streaming | No |
| Medium | 80–96 kbps | General streaming | Marginal |
| High | 128 kbps | High-quality streaming | Near-transparent |
| Highest | 160–192 kbps | Maximum quality | Yes (most content) |

#### Variable Bitrate (VBR) Encoding
```
VBR Modes in WMA Standard:
  Mode 0:     CBR (Constant Bitrate)
  Mode 1:     VBR (Quality-based)
  Mode 2:     VBR (Peak bitrate constrained)
  Mode 3:     VBR (Average bitrate constrained)

Quality Levels (for VBR Mode 1):
  Level 0:    Highest quality, variable bitrate
  Level 1:    High quality
  Level 2:    Medium quality
  Level 3:    Low quality
  Level 4:    Voice quality
```

#### Frame Size Calculation
```
Frame size in WMA Standard is determined by:
  - Sample rate (determines frame length bits)
  - Bitrate (determines bits per frame)
  - Channel mode (mono/stereo)

Formula:
  if sample_rate <= 16000:
    frame_length_bits = 9
  else if sample_rate <= 22050 or (WMAv1 and sample_rate <= 32000):
    frame_length_bits = 10
  else:
    frame_length_bits = 11
  
  frame_size_bytes = (bitrate × (2^frame_length_bits) / sample_rate) / 8
```

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. ASF container parsed for audio stream properties
2. Frame sync within ASF packets: 0x0100 sync word
3. Validate frame by checking:
   a. Frame size matches expected from header fields
   b. Huffman decode succeeds
   c. Dequantized values within valid range
4. Frame boundary determined by block_align field in WAVEFORMATEX
```

#### Seeking
- **ASF seeking:** Index-based seeking using ASF index object
- **Seek table:** Stored in optional Data Object Index; entries point to packet offsets
- **Precision:** Millisecond accuracy (not sample-accurate)

### 5.2 Detailed Bit Allocation Algorithm
The bit allocation algorithm determines how many bits to allocate to each scale factor band.

```
Bit Allocation Algorithm (Step-by-Step):

1. Compute FFT of input block (1024-point FFT)

2. Compute bark spectrum:
   for bark_band b:
     bark_energy[b] = Σ(k in bark_band[b]) Energy[k]

3. Compute masking thresholds:
   for each bark band:
     spreading[b] = Σ(other_b) bark_energy[other_b] × spreading_function[b][other_b]
     threshold[b] = bark_energy[b] / spreading[b]

4. Compute bit allocation:
   available_bits = frame_bit_budget - header_bits - scale_factor_bits
   
   repeat until all bits allocated or no more SMR improvement:
     for each band b:
       SMR[b] = bark_energy[b] / threshold[b]
     find band with maximum SMR
     allocate 1 bit to that band
     update threshold[b] using water-filling

5. Encode scale factors and coefficients with allocated bits
```

### 5.3 Inverse Quantization Details
After Huffman decoding, the quantized spectral coefficients are inverse quantized.

```
Inverse Quantization Formula:
  if code == 0:
    X = 0
  else:
    sign = -1 if code < 0 else 1
    magnitude = abs(code)
    X = sign × (magnitude)^(4/3) × 2^(gain / 4)

where:
  code = Huffman-decoded coefficient value
  gain = scale factor for this band (0–127)
  X = dequantized coefficient value
```

### 5.2 Core Decode Pipeline
```
1. Parse ASF packet structure
   ├── ASF packet header (variable size)
   ├── Payload parse flags
   └── Payload data (WMA frame payload)

2. Parse WMA frame header
   ├── Sync word verification (0x0100)
   ├── Bitrate, sample rate, channel mode
   └── Frame size from header fields

3. Decode Huffman-coded data
   ├── Read Huffman codebook index
   ├── Decode scale factors (differential)
   └── Decode spectral coefficients

4. Dequantization
   └── Apply inverse quantization with gain

5. Stereo processing (if MS stereo)
   ├── Decode M and S channels
   └── L = (M + S) / √2; R = (M - S) / √2

6. Inverse MDCT (IMDCT)
   └── x[n] = Σ(k=0 to N-1) X[k] · cos(π/N · (k + 1/2) · (n + N/2 + 1/2))

7. Overlap-add (50% overlap)
   └── output[n] = IMDCT_current[n] + IMDCT_previous[n + N]

8. QMF synthesis filterbank (32 bands)

9. Post-processing
   ├── DC offset correction
   └── Clip to 16-bit range
```

### 5.3 Error Concealment
- **Corrupt frame detection:** Huffman decode failure, invalid quantized values
- **Concealment method:** Repeat previous good frame (muted slightly)
- **Maximum consecutive errors before silence:** ~3 frames

### 5.4 Stereo Decoding (MS Mode)
When MS stereo is enabled, the mid and side channels are decoded and converted back to left and right.

```
MS to L/R Conversion:
  L = M + S
  R = M - S
  
or normalized:
  L = (M + S) / √2
  R = (M - S) / √2

Where:
  M = Mid channel = (L + R) / 2
  S = Side channel = (L - R) / 2
```

### 5.5 QMF Synthesis Filterbank
After inverse MDCT, the subband signals are reconstructed using the QMF synthesis filterbank.

```
QMF Synthesis Specifications:
  Number of bands:     32
  Output block:       1024 samples
  Overlap:            50% with previous block
  
Mathematical operation:
  x[n] = Σ(k=0 to 31) X[k][n] × h_synth[k][n mod 1024]
  where h_synth is the k-th QMF synthesis filter
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
| ASF/WMA | Yes (native) | Yes | Full | WMA = ASF with WMA audio stream |
| AVI | Partial | Yes | Limited | FourCC 0x0160/0x0161; limited metadata |
| Matroska/MKA | Partial | Yes | Full | Via LAV Filters or FFmpeg |
| MP4/M4A | No | — | — | Not supported |
| OGG | No | — | — | Not supported |
| WAV | No | — | — | WMA-in-WAV is different (multiple .wma in RIFF) |
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
| Encoder Settings | WM/EncodingSettings | 1024 bytes | UTF-16LE | No | Bitrate/quality params |
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
Cover art storage format in ASF/WMA:
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
Codec Name (CLI):   wmav1, wmav2     # used with -c:a
AV_CODEC_ID:        AV_CODEC_ID_WMAV1, AV_CODEC_ID_WMAV2
Format Name (CLI):  asf                # ASF container
Encoder(s):         wmav1, wmav2      # Native FFmpeg encoder
Decoder(s):         wmav1, wmav2      # Native FFmpeg decoder
Muxer(s):           asf, asfw          # ASF muxer
Demuxer(s):         asf                # ASF demuxer
```

### 8.2 FFmpeg Encoding — Full CLI Reference

```bash
# Encoding to WMA Standard (ASF container) — complete options reference
ffmpeg -i input.wav \
  -c:a wmav2 \
  -b:a 128k \                   # CBR bitrate — range: 48k–192k, default: 128k
  -ar 44100 \                  # Output sample rate (Hz)
  -ac 2 \                      # Output channel count (1=mono, 2=stereo)
  -sample_fmt s16le \          # Sample format: s16le only for WMA
  output.wma
```

#### Complete FFmpeg Option Table
| Option | Type | Default | Range / Valid Values | Effect on Output |
|--------|------|---------|---------------------|-----------------|
| `-b:a` | int | 128k | 48k–192k | Target bitrate (CBR only for WMA Standard) |
| `-ar` | int | 44100 | 8000–48000 | Output sample rate |
| `-ac` | int | from input | 1–2 | Channel count (2=max for WMA Standard) |
| `-sample_fmt` | string | s16le | s16le only | 16-bit signed PCM |
| `-cutoff` | int | auto | 0–24000 | High-frequency cutoff in Hz |
| `-joint_stereo` | bool | 1 | 0 or 1 | Enable joint (MS) stereo |

### 8.3 FFmpeg Encoding — libavcodec C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/opt.h>
#include <libswresample/swresample.h>

// ─── 1. Find encoder ─────────────────────────────────────────────────────────
const AVCodec *codec = avcodec_find_encoder(AV_CODEC_ID_WMAV2);
// Alternative by name: avcodec_find_encoder_by_name("wmav2");
if (!codec) { fprintf(stderr, "Encoder not found\n"); exit(1); }

// ─── 2. Allocate codec context ───────────────────────────────────────────────
AVCodecContext *ctx = avcodec_alloc_context3(codec);
if (!ctx) { fprintf(stderr, "OOM\n"); exit(1); }

// ─── 3. Set required parameters ──────────────────────────────────────────────
ctx->bit_rate    = 128000;               // bits/sec (48k–192k range)
ctx->sample_fmt  = AV_SAMPLE_FMT_S16;   // WMA Standard requires S16
ctx->sample_rate = 44100;               // Hz — must be in codec->supported_samplerates[]
av_channel_layout_copy(&ctx->ch_layout, &(AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO);

// Optional quality/mode settings:
av_opt_set_int(ctx, "joint_stereo", 1, AV_OPT_SEARCH_CHILDREN);

// ─── 4. Open codec ───────────────────────────────────────────────────────────
AVDictionary *opts = NULL;
int ret = avcodec_open2(ctx, codec, &opts);
if (ret < 0) {
    char errbuf[128]; av_strerror(ret, errbuf, sizeof(errbuf));
    fprintf(stderr, "avcodec_open2 failed: %s\n", errbuf);
    exit(1);
}

// ─── 5. Allocate frame and packet ─────────────────────────────────────────────
AVFrame  *frame = av_frame_alloc();
frame->format      = ctx->sample_fmt;
frame->sample_rate = ctx->sample_rate;
av_channel_layout_copy(&frame->ch_layout, &ctx->ch_layout);
frame->nb_samples  = ctx->frame_size;  // ctx->frame_size = 512 for WMA Standard
av_frame_get_buffer(frame, 0);

AVPacket *pkt = av_packet_alloc();

// ─── 6. Encode loop ──────────────────────────────────────────────────────────
void encode_frame(AVCodecContext *ctx, AVFrame *frame, AVPacket *pkt, FILE *outfile) {
    int ret = avcodec_send_frame(ctx, frame);
    if (ret < 0) { /* handle AVERROR(EAGAIN), AVERROR(EINVAL) */ }

    while (ret >= 0) {
        ret = avcodec_receive_packet(ctx, pkt);
        if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF) return;
        if (ret < 0) { /* fatal error */ exit(1); }
        // Write ASF packet to output
        fwrite(pkt->data, 1, pkt->size, outfile);
        av_packet_unref(pkt);
    }
}

// ─── 7. Flush encoder ───────────────────────────────────────────────────────
encode_frame(ctx, NULL, pkt, outfile); // NULL frame triggers flush/drain

// ─── 8. Cleanup ─────────────────────────────────────────────────────────────
av_frame_free(&frame);
av_packet_free(&pkt);
avcodec_free_context(&ctx);
```

**Critical Notes:**
- WMA Standard encoder requires `AV_SAMPLE_FMT_S16` (signed 16-bit PCM)
- `ctx->frame_size` is 512 samples for WMA Standard
- FFmpeg's WMA encoder quality is considered lower than Microsoft's native encoder
- ASF muxer (`-f asf`) must be used for proper container wrapping

### 8.4 FFmpeg Decoding — Full CLI Reference

```bash
# Decode to raw PCM WAV
ffmpeg -i input.wma \
  -c:a pcm_s16le \
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
```

### 8.5 FFmpeg Decoding — libavcodec C API

```c
// ─── Complete decode program skeleton ────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.wma", NULL, NULL);
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
            // frm->format = AVSampleFormat (S16 for WMA Standard)
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
| Encoder | encoder | WM/EncodedBy | Auto-set by FFmpeg |

### 8.7 Quality / Fidelity Decision Table
| Use Case | Settings | Expected Size (vs CD) | Notes |
|----------|----------|---------------------|-------|
| Archival / lossless | N/A | N/A | WMA Standard is lossy; use WMA Lossless |
| Audiophile lossy | `-c:a wmav2 -b:a 192k` | ~10% of CD | Maximum quality; near-transparent |
| Streaming (high) | `-c:a wmav2 -b:a 128k` | ~7% of CD | Standard streaming quality |
| Streaming (standard) | `-c:a wmav2 -b:a 96k` | ~5% of CD | |
| Podcast / voice | `-c:a wmav2 -b:a 48k -ac 1` | ~2.5% of CD | Mono |
| Mobile / low bandwidth | `-c:a wmav2 -b:a 64k` | ~3.5% of CD | |

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
Encoder delay:   ~512 samples (2 × frame_size/2) [NEEDS VERIFICATION]
Padding:         ~512 samples [NEEDS VERIFICATION]
Storage location: ASF Presentation Time (implicit)

FFmpeg gapless flags:
  Input:  Automatic handling via ASF parsing
  Output: ASF index generated for seeking
  
Gapless detection:
  WMA Standard: No native gapless markers; use ASF presentation time
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | ASF designed for streaming |
| Algorithmic encoder delay | ~185 ms | 512-sample lookahead |
| Live encoding feasible | Yes | ASF supports live broadcast |
| HTTP progressive download | Yes | ASF over HTTP |
| HTTP Live Streaming (HLS) | No | Not natively supported |
| DASH streaming | Partial | Via ASF segmenting |
| WebRTC / RTP transport | Yes | Via RTSP/MMS |
| Minimum decode buffer | ~2 frames / ~23 ms | |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | Layout Name | WMA Standard Support | Notes |
|----------|-------------|---------------------|-------|
| 1 | Mono | Yes | Mono encoding |
| 2 | Stereo | Yes | Primary mode |
| 3+ | Multichannel | No | Use WMA Pro for 5.1/7.1 |

### 11.2 Stereo Modes (WMA Standard Only)
```
Independent Stereo (default):
  Left and right channels encoded separately
  Maximum bitrate per channel: 96 kbps

MS Stereo (joint):
  Mid channel: M = (L + R) / 2
  Side channel: S = (L - R) / 2
  Benefits: Better for centered audio (vocals), saves bits
  Enabled by: joint_stereo=1 (FFmpeg default)
```

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 16-bit | WMA Standard limited to 16-bit |
| Max sample rate | 48000 Hz | |
| Float support | None | |
| DSD support | No | |
| 20-bit support | Partial | Only via WMA Pro |
| 24-bit support | Partial | Only via WMA Pro |

WMA Standard does not support high-resolution audio. For 24-bit/96 kHz audio, use WMA Pro (lossy) or WMA Lossless.

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

| Platform / API | Encode | Decode | FFmpeg Flag | Notes |
|----------------|--------|--------|-------------|-------|
| Windows Media Foundation | Yes | Yes | Native | Microsoft's official API |
| FFmpeg native | Yes | Yes | None | LGPL build |
| NVIDIA NVENC | No | — | N/A | WMA not supported |
| NVIDIA NVDEC | — | No | N/A | Not implemented |
| Intel QSV | No | Yes | `-hwaccel qsv` | Decode only |
| Apple VideoToolbox | No | No | N/A | No WMA support |
| Android MediaCodec | No | Partial | N/A | OEM-dependent |
| VA-API (Linux) | No | Yes | `-hwaccel vaapi` | Decode only |

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Encoder-Specific Issues
| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| Lower quality than Microsoft encoder | All | Use Microsoft encoder for production |
| VBR mode not working | Some versions | Use CBR mode |
| Incorrect sample rate detection | <4.0 | Specify `-ar` explicitly |

### 14.2 Interoperability Issues
- **Microsoft encoder → FFmpeg decoder:** Generally compatible; minor quality differences
- **FFmpeg encoder → Microsoft decoder:** Works but quality may be lower
- **Files with DRM:** FFmpeg cannot decode DRM-protected WMA files
- **Files with non-standard sample rates:** Some hardware decoders reject rates other than 44.1/48 kHz

### 14.3 Edge Cases to Handle in Converter
- **Empty file / 0-sample stream:** Output empty file; no error
- **File < 1 frame of audio:** Output partial frame; behavior depends on encoder
- **All-silence audio:** WMA encoder may use special silent-frame encoding
- **DC offset (non-zero mean):** Encoder applies high-pass filter; DC removed
- **Full-scale sine (0 dB):** Encoder may apply limiting; slight distortion possible
- **File with corrupt header:** FFmpeg may decode partially or fail with error
- **Truncated file:** Last frame may be corrupt; decoder outputs what it can
- **Sample rate not supported by codec:** Error: "sample rate not supported"
- **Channel count not supported (>2):** Error: "unsupported number of channels"

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM WMA Standard

| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → FLAC | `ffmpeg -i in.wma -c:a flac -compression_level 8 out.flac` | All ASF tags → Vorbis Comments | Lossless decode; lossy source |
| → ALAC | `ffmpeg -i in.wma -c:a alac out.m4a` | Partial (tag mapping) | Lossless decode; lossy source |
| → MP3 | `ffmpeg -i in.wma -c:a libmp3lame -q:a 0 out.mp3` | Tags via ID3v2.3 | Generation loss |
| → AAC | `ffmpeg -i in.wma -c:a aac -b:a 256k out.m4a` | Partial | Generation loss |
| → Opus | `ffmpeg -i in.wma -c:a libopus -b:a 128k out.opus` | Vorbis Comments (recreated) | Generation loss |
| → WAV | `ffmpeg -i in.wma -c:a pcm_s16le out.wav` | RIFF INFO tags | Lossless decode |
| → OGG Vorbis | `ffmpeg -i in.wma -c:a libvorbis -q:a 6 out.ogg` | Vorbis Comments (recreated) | Generation loss |
| → WMA Pro | `ffmpeg -i in.wma -c:a wmapro -b:a 256k out.wma` | Partial | Re-encoding lossy to lossy |

### 15.2 Converting TO WMA Standard

| Source | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| FLAC → | `ffmpeg -i in.flac -c:a wmav2 -b:a 192k out.wma` | Vorbis → ASF tags | Lossless decode |
| WAV → | `ffmpeg -i in.wav -c:a wmav2 -b:a 128k out.wma` | Limited | |
| MP3 → | `ffmpeg -i in.mp3 -c:a wmav2 -b:a 128k out.wma` | ID3v2 → ASF tags | Generation loss |
| AAC → | `ffmpeg -i in.m4a -c:a wmav2 -b:a 128k out.wma` | MP4 atoms → ASF | Generation loss |
| WMA Pro → | `ffmpeg -i in.wma -c:a wmav2 -b:a 192k out.wma` | Partial | Re-encode lossy to lossy |
| WMA Lossless → | `ffmpeg -i in.wma -c:a wmav2 -b:a 192k out.wma` | Partial | Lossless decode, lossy encode |

### 15.3 Lossless Round-Trip Verification
```bash
# Note: WMA Standard is lossy; no lossless round-trip possible
# For lossless: use WMA Lossless format instead

# Decode WMA Standard
ffmpeg -i input.wma -c:a pcm_s16le decoded.wav

# Compare decoded to original (will NOT match due to lossy compression)
ffmpeg -i original.wav -c:a pcm_s16le original_raw.wav
md5sum original_raw.wav decoded.wav   # Will NOT match
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| FFmpeg native | C | LGPL 2.1+ | 6/10 | 8/10 | https://ffmpeg.org |
| libavcodec | C | LGPL 2.1+ | 6/10 | 8/10 | https://ffmpeg.org |
| Windows Media Format SDK | C/C++ | Proprietary | 9/10 | 10/10 | Microsoft |
| LAV Filters | C/C++ | LGPL 2.1+ | N/A | 8/10 | https://github.com/Nevcairiel/LAVFilters |

### Build Instructions (for bundling in converter app)
```bash
# Build FFmpeg from source with WMA support
git clone https://github.com/FFmpeg/FFmpeg.git
cd FFmpeg
./configure --prefix=/usr/local \
  --enable-gpl --enable-libass --enable-libfreetype \
  --enable-nonfree
make -j$(nproc)
make install

# FFmpeg with WMA encoder/decoder is built-in (no external dependencies)
# Verify WMA support:
ffmpeg -encoders | grep -i wma
ffmpeg -decoders | grep -i wma
```

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **ASF Specification:** Microsoft ASF Specification (revision 01.20.03, December 2004) — https://download.microsoft.com/download/7/9/0/790fecaa-f64a-4a5e-a430-0bccdab3f1b4/ASF_Specification.doc
- **WMA Codec:** No public specification; reverse-engineered by FFmpeg

### Technical Resources
- FFmpeg encoder options: `ffmpeg -h encoder=wmav2` or https://ffmpeg.org/ffmpeg-codecs.html
- FFmpeg ASF muxer: `ffmpeg -h muxer=asf` or https://ffmpeg.org/ffmpeg-formats.html
- Multimedia Wiki: https://wiki.multimedia.cx/index.php/Windows_Media_Audio
- Hydrogenaudio: https://wiki.hydrogenaudio.org/index.php?title=WMA
- Library of Congress: https://www.loc.gov/preservation/digital/formats/fdd/fdd000027.shtml

### Academic Papers
- No significant academic papers on WMA Standard (proprietary codec)

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] Identify FFmpeg `./configure` flags needed (default build includes WMA)
- [ ] Verify `ffmpeg -encoders` output confirms wmav1/wmav2 encoder is available
- [ ] Verify `ffmpeg -decoders` output confirms wmav1/wmav2 decoder is available
- [ ] No external library dependency (WMA codec is built into FFmpeg)
- [ ] Note platform restrictions (macOS/iOS may need transcoding)

### Encoding Pipeline
- [ ] Convert input sample format to required `AV_SAMPLE_FMT_S16` using libswresample
- [ ] Fixed-frame-size encoder (`ctx->frame_size` = 512)
- [ ] Implement proper flush/drain at end of stream (send NULL frame)
- [ ] Use ASF muxer (`-f asf`) for proper container wrapping
- [ ] Validate that input sample rate is in `codec->supported_samplerates[]`
- [ ] Validate that channel layout is 1 or 2; error or downmix for 3+ channels

### Decoding Pipeline
- [ ] ASF container parsing (use avformat)
- [ ] Handle AVERROR(EAGAIN) in receive_packet loop correctly
- [ ] Flush decoder at EOF (send NULL packet)
- [ ] Handle sample format conversion from decoder output format
- [ ] Skip encoder-delay samples at stream start (if detectable)

### Metadata
- [ ] Read ASF Content Description Object fields
- [ ] Read ASF Extended Content Description Object fields
- [ ] Map all tag fields through standard key mapping
- [ ] Read WM/Picture cover art
- [ ] Write all standard tag fields to ASF container
- [ ] Embed cover art as WM/Picture binary
- [ ] Preserve ReplayGain tags through conversion
- [ ] Handle UTF-16LE encoding (ASF native)

### Quality & Verification
- [ ] Note: WMA Standard is lossy; cannot achieve bit-perfect output
- [ ] Implement progress reporting from FFmpeg stats output
- [ ] Implement error detection and partial-file recovery
- [ ] Test with: silence, full-scale, clipped, stereo files

### Edge Cases
- [ ] Handle files with corrupt or missing ASF headers
- [ ] Handle files with 0 samples
- [ ] Handle sample rate mismatch (trigger libswresample)
- [ ] Handle channel count mismatch (error for >2 channels)
- [ ] Handle bit depth mismatch (convert from decoder output)
- [ ] Handle very short files (< 1 frame)
- [ ] Handle DRM-protected files (FFmpeg cannot decode)

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
