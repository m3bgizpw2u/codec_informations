# WavPack — Deep Technical Reference
> **Category:** Lossless
> **File Extensions:** `.wv`, `.wvc`
> **MIME Types:** `audio/x-wavpack`, `audio/wavpack`
> **Standardization Body:** None (open standard, developer David Bryant)
> **Primary Specification:** https://www.wavpack.com/WavPack.pdf; https://github.com/dbry/WavPack
> **Patent Status:** Patent-free
> **License:** BSD 3-Clause
> **Current Version:** 5.9.0
> **Active Development:** Yes — last release 2026-01-18

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** David Bryant
- **Year Created:** 1998
- **Original Purpose:** Provide a high-performance, patent-unencumbered lossless audio codec with a unique hybrid mode that bridges the gap between pure lossless and efficient lossy delivery
- **Problem with Predecessors:** Predecessors either lacked compression efficiency, required expensive licensing, or relied on patented algorithms. WavPack was designed from the ground up using only well-known, public-domain techniques (linear prediction with LMS adaptation, Elias and Golomb codes) while avoiding patented methods like arithmetic coding and LZW compression

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| 1.0 | 1998 | Initial release |
| 2.0 | 2000 | Improved decorrelation, joint stereo |
| 3.0 | 2002 | Hybrid mode introduced |
| 4.0 | 2005 | New block format, APEv2 tagging, MD5 checksums |
| 4.3 | 2007 | Multichannel support, improved float handling |
| 4.4 | 2008 | Block checksums, RF64 support |
| 4.5 | 2009 | Improved DSD support (early) |
| 5.0 | 2016 | Native DSD audio compression (DSDIFF/DSF), WAVE64/CAF/AIFF wrappers, block checksums, new metadata format |
| 5.5 | 2020 | Up to 256 channels, improved AIFF/CAF support |
| 5.6 | 2022 | Enhanced DSD handling |
| 5.7 | 2023 | Multithreading support, optimize-int32 feature, BW64 support |
| 5.8 | 2025 | Bug fixes, performance improvements |
| 5.9 | 2026 | Latest stable release |

### 1.3 Current Adoption
- **Primary use cases today:** High-resolution audio archival, hybrid lossy/lossless delivery, DSD preservation, streaming with correction file upgrades
- **Platforms with native support:** Windows, macOS, Linux, Android, iOS, BSD, Haiku
- **Major services using this format:** Niche archival and enthusiast communities; not dominant in mainstream streaming
- **Hardware support:** Rockbox firmware, select DAPs (iBasso, Fiio, Cowon), Android/iOS players via apps, some AV receivers via DLNA/UPnP
- **Status:** Niche but respected; the most feature-rich open-source lossless codec

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Predictive (lossless / lossy / hybrid)
- **Core algorithm:** Adaptive decorrelation via LMS (Least Mean Squares) single-tap filters, combined with recursive Golomb coding
- **Loss mechanism:** Lossless by default; lossy via controlled residual truncation; hybrid via bitstream splitting
- **Frame-based vs sample-based:** Block-based. Each block is typically 0.5 seconds of audio (sample rate dependent), but block size is flexible
- **Fixed vs variable frame size:** Variable block size; encoder selects optimal block length per block

### 2.2 High-Level Encoding Flow (Block Diagram)
```
Input PCM Samples (mono or stereo per block)
      │
      ▼
[Joint Stereo Processing: L/R → Mid/Side]
      │
      ▼
[Multipass Decorrelation: 1–16 passes of single-tap LMS filters]
      │  each pass: e(k) = s(k) - w(k) * u(k)
      │  weight update: w(k+1) = w(k) + d * sgn(u(k)) * sgn(e(k))
      ▼
[Entropy Coding: Recursive Golomb with adaptive median]
      │
      ▼
[Bitstream Splitting — HYBRID MODE ONLY]
      ├── Main bitstream (.wv): lossy data (magnitude + sign, partial remainder)
      └── Correction bitstream (.wvc): full remainder refinement
      │
      ▼
[Block Packing: wvpk header + metadata + audio data + CRC]
      │
      ▼
Output .WV File (+ optional .WVC Correction File)
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | 0 samples | Block-based, no lookahead required |
| Block size | 0.5s typical (flexible) | Configurable; smaller blocks = faster seek, less compression |
| Max channels | 256 | Limited by uint8 channel index |
| Max bit depth | 32-bit integer + 32-bit IEEE float | DSD: 1-bit |
| Max sample rate | 1 GHz (integer steps) | 2 GHz tested |
| Bitrate range | 196–1411 kbps (CD audio hybrid) | Lossless: 705–1200 kbps typical |
| Complexity | O(n) | Decoding ~3× FLAC at default settings |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       77 76 70 6B     "wvpk"   WavPack block identifier
0x0004  4       XX XX XX XX      —         Block size (bytes, not including id+size)
0x0008  2       XX XX            —         Version (0x402–0x410, LE uint16)
0x000A  1       XX               —         Track number (0 if single)
0x000B  1       XX               —         Track sub-index
0x000C  4       XX XX XX XX      —         Total samples (0xFFFFFFFF = unknown)
0x0010  4       XX XX XX XX      —         Block index in samples
0x0014  4       XX XX XX XX      —         Samples in this block
0x0018  4       XX XX XX XX      —         Flags (bitfield)
0x001C  4       XX XX XX XX      —         CRC-32 of decoded samples
```

### 3.2 File-Level Header Layout
Full byte-map of the WavPack block header (preamble to every block):
```
Offset   Size    Field Name              Type        Valid Range       Description
-------  ------  ----------------------  ----------  ---------------   ---------------------------
0x0000   4B      Block ID                char[4]     "wvpk"            Magic identifier
0x0004   4B      Block Size              uint32 LE   24–16777215       Excludes ID + this field
0x0008   2B      Version                 uint16 LE   0x402–0x410       0x402 = 4.02, etc.
0x000A   1B      Track Number            uint8       0–255             0 = not used
0x000B   1B      Track Sub-Index         uint8       0–255             0 = not used
0x000C   4B      Total Samples           uint32 LE   0–4294967295      0xFFFFFFFF = unknown
0x0010   4B      Block Index             uint32 LE   0–4294967295      Sample offset of block start
0x0014   4B      Block Samples           uint32 LE   0–131072          Samples in this block
0x0018   4B      Flags                   uint32 LE   bitfield          See flags below
0x001C   4B      CRC                     uint32 LE   CRC-32            Of decoded samples (WavPack 5+)
```

### 3.3 Block Flags (32-bit bitfield)
```
Bits     Name                     Description
--------  ----------------------  --------------------------------------------------
0        WV_MONO                Block is mono (1 channel)
1        WV_HYBRID_MODE         Hybrid mode (lossy + correction)
2        WV_HYBRID_SHAPE        Noise shaping enabled (hybrid)
3        WV_HYBRID_BITRATE     Bitrate-based hybrid mode
4        WV_HYBRID_BALANCE      Balance control (joint stereo hybrid)
5        WV_CROSS_DECORR        Cross-decorrelation active
6        WV_NOISE_SHAPING       Second-order noise shaping (hybrid)
7        WV_JOINT_STEREO        Joint stereo encoding
8        WV_INITIAL_BLOCK       First block of a stream
9        WV_FINAL_BLOCK         Last block of a stream
10–15    —                       Reserved
16–21    —                       Left shift for non-8-bit depths (e.g., 20-bit = 4)
22–25    Sample Rate Index       Index into wv_rates[] table (15 = custom)
26       —                       Reserved
27       WV_FLOAT_DATA           32-bit IEEE float samples
28       —                       Reserved
29       WV_DSD_DATA             DSD (1-bit PCM) data
30       WV_INT32_DATA          32-bit integer data
31       WV_FALSE_STEREO         Block is "false stereo" (identical L+R)
```

### 3.4 Metadata Sub-Block Header
Metadata items follow the block header. Each has a variable-length header:
```
Bit 7 (0x80)   Long form: size is 3 bytes (24 bits)
Bit 6 (0x40)   Size is odd (add 1 to actual size)
Bit 0–4 (0x3F) Metadata ID

Size encoding:
  - If bit 7 clear: 1-byte size field (byte value = size)
  - If bit 7 set: 2-byte size field follows (uint16 LE, value in words)
```

### 3.5 Metadata ID Reference
| ID (hex) | Name | Description |
|----------|------|-------------|
| 0x01 | ID_DUMMY | No-op / padding |
| 0x02 | ID_ENCINFO | Encoder info: block size, sample rate, etc. |
| 0x03 | ID_DECTERMS | Decorrelation terms (filter lengths) |
| 0x04 | ID_DECWEIGHTS | Decorrelation filter weights |
| 0x05 | ID_DECSAMPLES | Decorr samples buffer (initial values) |
| 0x06 | ID_ENTROPY | Entropy variables (median values) |
| 0x07 | ID_HYBRID | Hybrid mode control (bitrate, shaping) |
| 0x08 | ID_SHAPING | Noise shaping data |
| 0x09 | ID_FLOATINFO | Float sample parameters |
| 0x0A | ID_INT32INFO | 32-bit integer parameters |
| 0x0B | ID_DATA | Compressed audio data |
| 0x0C | ID_CORR | Cross-correlation (stereo decorrelation) |
| 0x0D | ID_EXTRABITS | Extra bits for sub-8-bit data |
| 0x0E | ID_CHANINFO | Channel ordering info |
| 0x0F | ID_DSD_DATA | DSD audio data |
| 0x21 | ID_RIFF_HEADER | RIFF/WAVE header (from source .wav) |
| 0x22 | ID_RIFF_TRAILER | RIFF/WAVE trailer (from source .wav) |
| 0x26 | ID_MD5 | 16-byte MD5 checksum of raw audio |
| 0x27 | ID_SAMPLE_RATE | Custom sample rate (replaces index) |

### 3.6 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Unsigned integer | Yes | Stored as signed internally (shifted) |
| 16-bit | Signed integer | Yes | Standard CD audio |
| 20-bit | Signed integer | Yes | Extra bit flag (shift = 4) |
| 24-bit | Signed integer | Yes | Standard high-res |
| 32-bit | Signed integer | Yes | Extended precision |
| 32-bit | IEEE float | Yes | Range -1.0 to +1.0, no infinities/NaN in hybrid |
| 64-bit | IEEE double | No | Not supported |

#### Supported Sample Rates
| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 8000 | Telephone | Yes | |
| 11025 | — | Yes | |
| 16000 | Wideband voice | Yes | |
| 22050 | — | Yes | |
| 32000 | Broadcast | Yes | |
| 44100 | CD audio | Yes | |
| 48000 | Professional | Yes | |
| 88200 | 2× CD | Yes | |
| 96000 | High-res | Yes | |
| 176400 | 4× CD | Yes | |
| 192000 | High-res max | Yes | |
| 352800 | DXD | Yes | |
| 384000 | Ultra-high-res | Yes | |
| 705600 | DSD64×4 | Yes | WavPack 5 DSD |
| 1411200 | DSD128 | Yes | |
| Any integer | Custom | Yes | Up to 1 GHz, via ID_SAMPLE_RATE (0x27) |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Pre-Processing Stage
- **DC offset removal:** Not explicitly performed. The LMS decorrelation naturally adapts to DC offsets via the weight adaptation mechanism. Any residual DC is encoded as part of the signal.
- **Pre-emphasis filter:** None. WavPack relies entirely on adaptive linear prediction.
- **Windowing function:** None. WavPack is a time-domain codec with no windowed transforms.
- **Level normalization:** Input samples are presented as 32-bit signed integers to the encoder. The encoder shifts input data internally for bit depths that are not multiples of 8.
- **Stereo decorrelation pre-step:** L/R channels are immediately converted to Mid/Side:
  - `M(k) = int((L(k) + R(k)) / 2)`  — integer division discards LSB (recoverable from difference)
  - `S(k) = L(k) - R(k)`
  - The LSB of M is encoded implicitly from S, providing free compression for stereo.

### 4.2 Analysis / Transform Stage

#### Transform Type: None (pure time-domain predictive)

WavPack uses no frequency-domain transforms. All compression occurs in the time domain through:

1. **Stereo decorrelation** (joint stereo): M/S encoding eliminates inter-channel redundancy
2. **Sample decorrelation**: Multiple passes of single-tap LMS adaptive filters remove intra-channel redundancy
3. **Entropy coding**: Recursive Golomb coding compresses the resulting residuals

```
LPC Analysis (for predictive codecs):
  Type:             Single-tap LMS (Least Mean Squares)
  Window:           None (sample-by-sample adaptation)
  Algorithm:        Sign-sign LMS (avoids multiplication overflow)
  Coefficient precision: 16-bit signed, 10-bit fraction (range ±4.0)
  Step-size range:  1–7 units (default: 2)
  Max terms:        16 passes per block
```

**Filter term definitions:**

For single-channel (terms 1–18):
| Term Value | Input u(k) | Description |
|------------|------------|-------------|
| 1–8 | s(k - term) | Previous samples at various delays |
| 17 | 2s(k-1) - s(k-2) | Linear extrapolation |
| 18 | 3s(k-1) - s(k-2)/2 | Quadratic extrapolation |

For dual-channel (additional terms -1, -2, -3):
| Term Value | Input u(k) | Description |
|------------|------------|-------------|
| -1 | s_ch1(k-1), s_ch2(k-1) | Cross-channel (predict ch2 from ch1) |
| -2 | s_ch1(k-1), s_ch2(k) | Cross-channel (predict ch2 from current ch1) |
| -3 | s_ch1(k-1), s_ch1(k-1) | Same-channel (force weight to ±1) |

**Mode-specific filter configurations:**
```
Fast mode:       terms = {17, 17}           (2 passes)
Default mode:    terms = {18, 18, 2, 3, -2} (5 passes)
High mode:       terms = {18, 18, 2, 3, -2, 18, 2, 4, 7, 5, 3, 6, 8, -1, 18, 2} (16 passes)
```

### 4.3 Entropy Coding — Recursive Golomb

WavPack uses a proprietary **Recursive Golomb Coding** scheme that generalizes traditional Golomb coding to handle non-ideal distributions.

**Standard Golomb coding:**
- Parameter m (a positive integer) determines the coding efficiency
- For m = 2^k (Rice coding): quotient in unary, remainder in k bits
- For m ≠ 2^k: adjusted binary coding for the remainder

**Recursive refinement:**
WavPack extends Golomb by recursively partitioning the distribution:
1. First median m partitions residuals into 1/2 and 1/2
2. Second median m' partitions upper half into 1/4 and 1/4
3. Third median m'' partitions next quarter into 1/8 and 1/8
4. Continue recursively

This handles non-exponential distributions efficiently without the complexity of arithmetic coding.

**Median adaptation per sample:**
```
if e(k) >= m(k):   m(k+1) = m(k) + int((m(k) + 127) / 128)
else:              m(k+1) = m(k) - int((m(k) + 126) / 128)
```

**Overflow protection:**
When a residual requires ≥16 consecutive unary 1's, WavPack emits 16 ones, then an Elias gamma code for the actual count. This prevents catastrophic codeword expansion.

**Signed residual encoding:**
1. Convert to unsigned: if e < 0, encode(-e - 1)
2. Encode magnitude using recursive Golomb
3. Append sign bit

**Zero-run encoding:**
When medians are very small, runs of zero residuals are encoded as: 1 + Elias-gamma(length). The leading 1 distinguishes from normal value encoding.

### 4.4 Hybrid Mode — Bitrate-Constrained Quantization

In hybrid mode, WavPack imposes a bitrate constraint that forces quantization:

**Minimum lossy bitrate:** 2.22 bits/sample (~196 kbps for CD audio)
- Achieved by expanding the first partition to 5/7 of the distribution (instead of 1/2)
- Uses unary-to-binary translation to compress the magnitude prefix

**Bitrate control:**
For each sample, WavPack calculates:
1. The magnitude range from encoded unary prefix
2. The required remainder bits to reach the target error limit
3. If range < error limit: no additional data needed
4. Otherwise: binary search refinement bits until error limit met

**Correction file generation:**
- Remainder refinement bits (beyond the magnitude prefix) are routed to the `.wvc` correction file
- The `.wvc` file contains only correction data; it cannot be decoded independently
- Combined `.wv + .wvc` = bit-exact original

**Dynamic noise shaping (DNS):**
In hybrid mode, WavPack can apply first-order noise shaping to push quantization noise into frequency bands where it is less perceptible. The shaping filter coefficient is transmitted in the WP_ID_SHAPING metadata block.

### 4.5 Encoder Settings / Quality Modes

#### For Lossless Compression (FFmpeg + Reference)
| Compression Level | Encoding Speed | Decode Speed | Compression Ratio | Notes |
|---|---|---|---|---|
| 0 (fast) | ~0.5× realtime | ~8× realtime | ~65–70% | 2 decorr passes |
| 1 | ~0.6× realtime | ~7× realtime | ~62–67% | 2 passes + delta |
| 2 | ~0.8× realtime | ~6× realtime | ~58–63% | 4 passes |
| 3 | ~1× realtime | ~5× realtime | ~55–60% | 5 passes + delta |
| 4 | ~1.5× realtime | ~4× realtime | ~52–58% | 5 passes + delta |
| 5 | ~2× realtime | ~3× realtime | ~50–55% | 5 passes + delta + branches |
| 6 | ~3× realtime | ~2× realtime | ~48–53% | 5 passes + delta + sort |
| 7 | ~5× realtime | ~1.5× realtime | ~46–51% | 5 passes + delta + sort + branches |
| 8 (high) | ~10× realtime | ~1× realtime | ~44–49% | 16 passes + all extras |

#### For Hybrid Mode (Reference Only)
| Bitrate (bps) | Quality | .WV Size (CDDA) | Notes |
|---|---|---|---|
| 2.0 | Minimum lossy | ~176 kbps | 2.22 bits/sample floor |
| 3.0 | Very low | ~265 kbps | |
| 4.0 | Low | ~353 kbps | |
| 6.0 | Medium | ~529 kbps | Transparent for most content |
| 8.0 | High | ~706 kbps | Near-transparent |
| 10.0 | Very high | ~882 kbps | |
| 12.0 | Near lossless | ~1058 kbps | Very close to lossless |
| 24.0 | Maximum | ~2117 kbps | Nearly lossless |

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. Scan for magic bytes: "wvpk" (0x77 76 70 6B)
2. Read block header: version, block size, sample count, flags
3. Validate version: must be 0x402–0x410
4. Validate block_samples: ≤ 131072
5. Compute next block offset = current_pos + 4 (id) + 4 (size) + size
6. Verify at least one metadata block or audio data present
7. Optional: verify CRC-32 checksum (WavPack 5+)
8. If invalid: advance 1 byte, retry
9. Max failed sync attempts: implementation-defined
```

#### Seeking
- **Seek table:** Embedded in audio data as ID_DECSAMPLES and ID_ENTROPY metadata blocks
- **Seek precision:** Sample-accurate within a block; block size is typically ~22,050 samples at 44.1kHz
- **Hybrid seeking:** Correction file must be present for lossless seeking
- **Fast seek (WavPack 5+):** Block-level checksums enable corruption detection without full decode

### 5.2 Core Decode Pipeline
```
1. Read block header (32 bytes minimum)
   ├── Verify "wvpk" magic
   ├── Parse version, block_samples, flags
   └── Compute CRC for verification (WavPack 5+)

2. Parse metadata sub-blocks
   ├── ID_ENCINFO: block size, sample rate, bits per sample
   ├── ID_DECTERMS: number and types of decorrelation passes
   ├── ID_DECWEIGHTS: initial filter weights (16-bit fixed-point)
   ├── ID_DECSAMPLES: initial sample history buffer
   ├── ID_ENTROPY: initial median values for each channel
   ├── ID_SHAPING: noise shaping filter state
   └── ID_FLOATINFO: float exponent parameters

3. Read compressed audio data (ID_DATA)
   └── Bit-level recursive Golomb decoding per channel

4. Reconstruct residuals
   └── Apply sign and convert from unsigned to signed

5. Entropy decoding (reverse of encoding)
   └── Recursive Golomb with medians tracked per sample

6. Apply decorrelation passes (in reverse order of encoding)
   └── For each pass: apply LMS filter using stored weights
       s_recon(k) = s(k) + w(k) * u(k)

7. Weight adaptation
   └── w(k+1) = w(k) + d * sgn(u(k)) * sgn(e(k))

8. Stereo reconstruction
   └── L = M + (S >> 1)
       R = M - (S >> 1)
   Or if joint stereo: decode L/R directly

9. Output formatting
   └── Clip to output bit depth, convert to output format
```

### 5.3 Error Concealment
- **Corrupt block detection:** CRC-32 checksum (WavPack 5+) verified before parsing
- **Concealment method:** Mute corrupted block; continue playback
- **Maximum consecutive errors:** Unlimited (plays through with silence)
- **Hybrid lossy fallback:** If .wvc correction file is missing, decoder gracefully produces lossy output from .wv alone

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** WavPack (.wv) is itself a container format. It stores audio blocks with embedded metadata. A RIFF/WAVE wrapper can be appended for compatibility.
- **Overhead:** ~0.5–2% (block headers + metadata)
- **Seeking in native format:** Yes — by block index and sample number
- **Multiple streams in native container:** No — one audio stream per .wv file

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| .WV (native) | Yes | Yes — block index | Full APEv2 | Primary format |
| Matroska/MKA | Yes | Yes — index | Full | Good support |
| RIFF/WAVE | Yes (hybrid) | Yes | Via .wvc sidecar | Limited |
| AIFF | Yes | Yes | Via .wvc sidecar | |
| CAF | Yes | Yes | Via .wvc sidecar | |
| MP4/M4A | No | — | — | Not supported |
| OGG | No | — | — | Not supported |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** APEv2
- **Tag block location:** End of file (after all audio blocks)
- **Tag block identifier:** `APETAGEX` (0x41 50 45 54 41 47 45 58)
- **ID3v1:** Read on decode if present (legacy), but APEv2 takes priority

### 7.2 Standard Tag Fields — Complete Reference
| Tag Field | Internal Key | Max Length | Char Encoding | Multi-value | Notes |
|-----------|---------------------------|------------|---------------|-------------|-------|
| Title | Title | 512 bytes | UTF-8 | No | |
| Artist | Artist | 512 bytes | UTF-8 | No | |
| Album | Album | 512 bytes | UTF-8 | No | |
| Album Artist | Album Artist | 512 bytes | UTF-8 | No | |
| Composer | Composer | 512 bytes | UTF-8 | No | |
| Genre | Genre | 256 bytes | UTF-8 | No | |
| Year / Date | Year | 32 bytes | UTF-8 | No | Format: YYYY |
| Track Number | Track | 32 bytes | UTF-8 | No | Format: "N" or "N/Total" |
| Disc Number | Disc | 32 bytes | UTF-8 | No | Format: "N" or "N/Total" |
| Comment | Comment | 512 bytes | UTF-8 | No | |
| ReplayGain Track Gain | REPLAYGAIN_TRACK_GAIN | 32 bytes | ASCII | No | Format: "-6.20 dB" |
| ReplayGain Track Peak | REPLAYGAIN_TRACK_PEAK | 32 bytes | ASCII | No | Format: "0.998459" |
| ReplayGain Album Gain | REPLAYGAIN_ALBUM_GAIN | 32 bytes | ASCII | No | |
| ReplayGain Album Peak | REPLAYGAIN_ALBUM_PEAK | 32 bytes | ASCII | No | |
| Cover Art | Cover Art (Front) | 16 MB | Binary | No | JPEG/PNG embedded |
| Encoder | Encoder | 256 bytes | UTF-8 | No | Software name + version |
| Cuesheet | Cuesheet | 16 MB | UTF-8 | No | Embedded cuesheet |
| ID | — | — | — | — | Unique file identifier |
| Source | Source | 256 bytes | UTF-8 | No | Orig. source medium |

### 7.3 Cover Art Storage
Cover art in WavPack is stored as a standard APEv2 item:
```
APEv2 Item for cover art:
  Key: "Cover Art (Front)"
  Value: Binary image data (JPEG or PNG)
  Type:  Binary (0x03)
  Size:  Image size in bytes
```

### 7.4 Metadata Compatibility Matrix
| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| ID3v1 | ✓ | ✓ | ✓ (legacy) | Lowest |
| APEv2 | ✓ | ✓ | ✓ | Highest |
| Vorbis Comments | ✗ | ✗ | ✗ | — |

**Conflict resolution:** APEv2 tags take absolute priority. ID3v1 is read only if no APEv2 tag is present.

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   wavpack
AV_CODEC_ID:        AV_CODEC_ID_WAVPACK
Format Name (CLI):  wavpack
Encoder(s):         wavpack (native, lossless-only)
Decoder(s):         wavpack (native, handles WavPack 4 and 5)
Muxer(s):          wv (native .wv muxer), matroska
Demuxer(s):         wv (native .wv demuxer), matroska
```

### 8.2 FFmpeg Encoding — Full CLI Reference

**IMPORTANT:** FFmpeg's native WavPack encoder is **lossless-only**. It does NOT support hybrid mode or correction files.

```bash
# Lossless encoding — complete options reference
ffmpeg -i input.wav \
  -c:a wavpack \
  -compression_level 8 \     # 0=fast (least), 8=high (most), default=0
  -ar 44100 \                # Output sample rate
  -ac 2 \                    # Output channel count
  -sample_fmt s32le \         # Output sample format: s16, s32, fltp
  output.wv

# Default encoding (fastest, least compression)
ffmpeg -i input.wav -c:a wavpack output.wv

# High compression (slowest, best ratio)
ffmpeg -i input.wav -c:a wavpack -compression_level 8 output.wv
```

#### Complete FFmpeg Option Table
| Option | Type | Default | Range / Valid Values | Effect on Output |
|--------|------|---------|---------------------|-----------------|
| `-compression_level` | int | 0 | 0–8 | Higher = better compression, slower encode |
| `-b:a` | int | — | — | Ignored for WavPack (not applicable) |
| `-q:a` | float | — | — | Ignored (no VBR quality mode) |
| `-ar` | int | auto | any supported | Output sample rate |
| `-ac` | int | input | 1–256 | Output channel count |
| `-sample_fmt` | string | s32 | s16, s32, fltp | Output bit depth/format |

### 8.3 FFmpeg Encoding — libavcodec C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/opt.h>
#include <libswresample/swresample.h>

// ─── 1. Find encoder ─────────────────────────────────────────────────────────
const AVCodec *codec = avcodec_find_encoder(AV_CODEC_ID_WAVPACK);
if (!codec) { fprintf(stderr, "Encoder not found\n"); exit(1); }

// ─── 2. Allocate codec context ───────────────────────────────────────────────
AVCodecContext *ctx = avcodec_alloc_context3(codec);
if (!ctx) { fprintf(stderr, "OOM\n"); exit(1); }

// ─── 3. Set required parameters ──────────────────────────────────────────────
ctx->bit_rate    = 0;                          // WavPack is lossless; no bitrate
ctx->sample_fmt  = AV_SAMPLE_FMT_S32;          // 32-bit integer output
ctx->sample_rate = 44100;                      // Hz
av_channel_layout_copy(&ctx->ch_layout, &(AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO);

// ─── 4. Set compression level ────────────────────────────────────────────────
// 0 = fastest, 8 = highest compression
av_opt_set_int(ctx, "compression_level", 8, AV_OPT_SEARCH_CHILDREN);

// ─── 5. Open codec ───────────────────────────────────────────────────────────
int ret = avcodec_open2(ctx, codec, NULL);
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
frame->nb_samples  = ctx->frame_size;   // 0 for variable frame size (WavPack)
av_frame_get_buffer(frame, 0);

AVPacket *pkt = av_packet_alloc();

// ─── 7. Encode loop ──────────────────────────────────────────────────────────
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
encode_frame(ctx, NULL, pkt, outfile);  // NULL frame = drain

// ─── 9. Cleanup ──────────────────────────────────────────────────────────────
av_frame_free(&frame);
av_packet_free(&pkt);
avcodec_free_context(&ctx);
```

**Critical Notes:**
- FFmpeg's WavPack encoder does **NOT** support hybrid mode. Use the reference `wavpack` CLI for hybrid/correction files.
- FFmpeg produces **WavPack version 4** files, not version 5 (no block checksums, no fast verification)
- `ctx->frame_size = 0` for WavPack (variable block size based on sample rate)

### 8.4 FFmpeg Decoding — Full CLI Reference

```bash
# Decode to raw PCM WAV
ffmpeg -i input.wv \
  -c:a pcm_s32le \
  output.wav

# Decode with automatic format detection
ffmpeg -i input.wv output.wav

# Decode and resample to 48kHz
ffmpeg -i input.wv -ar 48000 output.wav

# Decode multi-channel to stereo
ffmpeg -i input_5_1.wv -ac 2 output.wav

# Extract WavPack stream without re-encoding
ffmpeg -i input.wv -c:a copy output.wv

# Probe format details
ffprobe -v quiet -print_format json -show_streams -show_format input.wv

# Decode DSD to PCM (WavPack 5 → PCM via FFmpeg)
ffmpeg -i input_dsd.wv -ar 2822400 output_pcm.wav
```

### 8.5 FFmpeg Decoding — libavcodec C API

```c
// ─── Complete decode program skeleton ────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.wv", NULL, NULL);
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
            // frm->format = AVSampleFormat (AV_SAMPLE_FMT_S32 typically)
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
ffprobe -v quiet -print_format json -show_format input.wv | jq .format.tags

# Write metadata (lossless transcode)
ffmpeg -i input.wv -c:a copy \
  -metadata title="Song Title" \
  -metadata artist="Artist Name" \
  -metadata album="Album Name" \
  -metadata album_artist="Album Artist" \
  -metadata track="5/12" \
  -metadata disc="1/2" \
  -metadata date="2024" \
  -metadata genre="Classical" \
  -metadata REPLAYGAIN_TRACK_GAIN="-6.20 dB" \
  -metadata REPLAYGAIN_TRACK_PEAK="0.998459" \
  output_tagged.wv

# Strip all metadata
ffmpeg -i input.wv -c:a copy -map_metadata -1 output_stripped.wv

# Embed cover art
ffmpeg -i input.wv -i cover.jpg \
  -c:a copy \
  -map 0:a -map 1:v \
  -metadata:s:v title="Album cover" \
  -disposition:v attached_pic \
  output_with_cover.wv
```

#### FFmpeg Internal Metadata Key Mapping
| Standard Field | FFmpeg Key | WavPack Native Key | Notes |
|----------------|------------|---------------------|-------|
| Title | title | Title | |
| Artist | artist | Artist | |
| Album | album | Album | |
| Album Artist | album_artist | Album Artist | |
| Track Number | track | Track | |
| Disc Number | disc | Disc | |
| Genre | genre | Genre | |
| Date/Year | date | Year | |
| Comment | comment | Comment | |
| Encoder | encoder | Encoder | Auto-set by FFmpeg |

### 8.7 Quality / Fidelity Decision Table
| Use Case | Settings | Expected Size | Notes |
|----------|----------|---------------|-------|
| Archival / lossless | `-c:a wavpack -compression_level 8` | ~45–50% of PCM | Use reference encoder for best |
| Fast archival | `-c:a wavpack -compression_level 0` | ~65–70% of PCM | FFmpeg default |
| Portable / streaming | `-c:a wavpack -compression_level 2` | ~60–65% of PCM | Good balance |
| Hybrid lossy (reference only) | `wavpack -b4.0 -c file.wav` | ~353 kbps | Use reference CLI, not FFmpeg |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
```
Embedded seek index (stored in ID_DATA metadata):
  Location:    Distributed within audio blocks
  Structure:   Each block header contains:
    - block_index: uint32 — starting sample number
    - block_samples: uint32 — samples in this block
  Seeking:     Binary search on block_index, then decode from block start
  Precision:   Sample-accurate within block
  Block size:  ~22050 samples at 44100 Hz (0.5 second)
```

### 9.2 Gapless Playback Data
```
Encoder delay:   0 samples (WavPack is block-based, no lookahead)
Padding:         0 samples (block boundaries are clean)
Storage:         Total samples in WavPack header (uint32 at offset 0x0C)
                 = exact count of audio samples in file

FFmpeg gapless flags: N/A (WavPack stores exact sample count)
Gapless detection:    Check block header's total_samples field
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | Block-based; can begin after first block |
| Algorithmic encoder delay | 0 samples | Block-based, no lookahead |
| Live encoding feasible | Yes | Reference encoder supports streaming mode |
| HTTP progressive download | Yes | Block-level seeking works over HTTP |
| HTTP Live Streaming (HLS) | No | No native HLS support |
| DASH streaming | Partial | Via Matroska/DASH if muxed |
| WebRTC / RTP transport | No | Not designed for real-time transport |
| Minimum decode buffer | 1 block | ~0.5 seconds at 44.1kHz |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | Layout Name | Channel Order | FFmpeg Layout Constant |
|----------|-------------|---------------|------------------------|
| 1 | Mono | C | AV_CHANNEL_LAYOUT_MONO |
| 2 | Stereo | L, R | AV_CHANNEL_LAYOUT_STEREO |
| 3 | 2.1 | L, R, LFE | AV_CHANNEL_LAYOUT_2POINT1 |
| 4 | Quad | FL, FR, RL, RR | AV_CHANNEL_LAYOUT_QUAD |
| 5 | 5.0 | FL, FR, C, BL, BR | AV_CHANNEL_LAYOUT_5POINT0 |
| 6 | 5.1 | FL, FR, C, LFE, BL, BR | AV_CHANNEL_LAYOUT_5POINT1 |
| 7 | 6.1 | FL, FR, C, LFE, BL, BR, BC | AV_CHANNEL_LAYOUT_6POINT1 |
| 8 | 7.1 | FL, FR, C, LFE, BL, BR, SL, SR | AV_CHANNEL_LAYOUT_7POINT1 |
| N (up to 256) | Custom | Arbitrary | Via WAVEFORMATEXTENSIBLE mask |

### 11.2 Downmix Coefficients (5.1 → Stereo)
```
L_out = FL + C×0.707 + BL×0.707
R_out = FR + C×0.707 + BR×0.707
LFE:  discarded

FFmpeg downmix command:
ffmpeg -i input_5_1.wv -af "pan=stereo|FL<FL+0.707*FC+0.707*BL|FR<FR+0.707*FC+0.707*BR" output.wv
```

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 32-bit integer + 32-bit float | |
| Max sample rate | 1 GHz (integer steps) | 2 GHz tested |
| Float support | 32-bit IEEE float only | No 64-bit double |
| DSD support | Yes (WavPack 5+) | DSD64 through DSD512 |
| 20-bit support | Yes | Via left-shift flag in flags field |
| 24-bit support | Yes | Native |
| 32-bit integer | Yes | Extended precision |

```bash
# High-res 24-bit/96kHz encoding
ffmpeg -i input_24bit_96k.wav \
  -c:a wavpack \
  -compression_level 8 \
  output_hires.wv

# High-res 32-bit float encoding
ffmpeg -i input_float.wav \
  -c:a wavpack \
  -sample_fmt fltp \
  output_float.wv

# DSD encoding (reference only)
wavpack input.dsf -o output.wv
```

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

| Platform / API | Encode | Decode | FFmpeg Flag | Notes |
|----------------|--------|--------|-------------|-------|
| Native (reference) | Yes | Yes | N/A | libwavpack |
| FFmpeg native | Yes (lossless only) | Yes | `-c:a wavpack` | |
| NVIDIA NVENC/NVDEC | No | No | — | Not applicable |
| Intel QSV | No | No | — | Not applicable |
| Apple AudioToolbox | No | Yes | Via AudioQueue | Some players |
| Android MediaCodec | No | Yes | Via MediaCodec | Via FFmpeg |
| VA-API (Linux) | No | No | — | Not applicable |

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Encoder-Specific Issues
| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| Produces WavPack v4 only (no v5 checksums) | All versions | Use reference encoder for WavPack 5 |
| No hybrid/correction file support | All versions | Use `wavpack` CLI for hybrid |
| Multi-channel > 16 channels: wrong format | < 6.1 | Use reference encoder for >16 ch |
| 8-bit PCM encoding produces broken files | < 6.1 | Use reference encoder for 8-bit |
| Decoding dropout bug (rare) | < 6.1 | Upgrade to 6.1+ |
| No MD5 verification | All versions | Use `wavpack -m` CLI |
| Cannot restore non-audio RIFF chunks | All versions | Use reference encoder |
| Correction files ignored | All versions | Use reference decoder or foobar2000 |

### 14.2 Interoperability Issues
- **FFmpeg → reference decoder:** FFmpeg produces WavPack 4 format; reference decoder handles it correctly
- **Reference encoder → FFmpeg:** FFmpeg reads WavPack 5 files but cannot write them
- **Hybrid files + FFmpeg:** FFmpeg decodes only the lossy .wv portion; .wvc correction file is ignored
- **DSD files + FFmpeg:** FFmpeg decodes DSD → PCM, cannot write DSD-in-WavPack
- **Files with 17+ channels:** FFmpeg produces non-compliant files (fixed in 6.1 for encode, but decode may vary)

### 14.3 Edge Cases to Handle in Converter
- **Empty file / 0-sample stream:** Refuse to encode; warn user
- **File < 1 block of audio:** Handled normally (minimum block is encoded)
- **All-silence audio:** Encodes very efficiently; large runs of zero residuals compressed to near-zero
- **DC offset (non-zero mean):** LMS filter adapts to DC; may slightly reduce compression ratio
- **Full-scale sine (0 dB):** No clipping risk; decorrelation handles well
- **Corrupt header:** Report error; do not attempt decode
- **Truncated file:** Decode available blocks; report truncated warning
- **Sample rate not supported:** Not applicable; WavPack supports all integer sample rates
- **Channel count > 255:** Refuse to encode (FFmpeg limitation)
- **Float with infinities/NaN:** Refuse hybrid encode; lossless float can encode but decoder may produce NaN

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM WavPack

| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|---------------------|----------------|
| → FLAC | `ffmpeg -i in.wv -c:a flac -compression_level 8 out.flac` | All tags via rewrite | Lossless |
| → ALAC | `ffmpeg -i in.wv -c:a alac out.m4a` | Partial (tag mapping) | Lossless if source is lossless |
| → WAV | `ffmpeg -i in.wv out.wav` | RIFF INFO (if reference used) | Lossless |
| → MP3 | `ffmpeg -i in.wv -c:a libmp3lame -q:a 0 out.mp3` | Tags via ID3v2.3 | Generation loss |
| → AAC | `ffmpeg -i in.wv -c:a aac -b:a 256k out.m4a` | Partial | Generation loss |
| → Opus | `ffmpeg -i in.wv -c:a libopus -b:a 128k out.opus` | Vorbis Comments | Generation loss |
| → OGG Vorbis | `ffmpeg -i in.wv -c:a libvorbis -q:a 6 out.ogg` | Vorbis Comments | Generation loss |
| → WavPack (lossless) | `ffmpeg -i in.wv -c:a wavpack -compression_level 8 out.wv` | APEv2 tags | Lossless |
| → WavPack (hybrid ref) | `wavpack in.wv -h -o out.wv` | APEv2 tags | Lossless or hybrid |

### 15.2 Converting TO WavPack

| Source | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|---------------------|----------------|
| FLAC → | `ffmpeg -i in.flac -c:a wavpack -compression_level 8 out.wv` | APEv2 via rewrite | Lossless |
| WAV → | `ffmpeg -i in.wav -c:a wavpack out.wv` | None (FFmpeg) | Lossless |
| WAV → (ref) | `wavpack in.wav -h -m -o out.wv` | RIFF INFO + APEv2 | Lossless + MD5 |
| MP3 → | `ffmpeg -i in.mp3 -c:a wavpack out.wv` | ID3 → APEv2 | Generation loss |
| AAC → | `ffmpeg -i in.m4a -c:a wavpack out.wv` | Partial | Generation loss |
| Vorbis → | `ffmpeg -i in.ogg -c:a wavpack out.wv` | Vorbis → APEv2 | Lossless |
| ALAC → | `ffmpeg -i in.m4a -c:a wavpack out.wv` | Partial | Lossless |
| DSD → (ref) | `wavpack in.dsf -o out.wv` | DSDIFF/DSF metadata | Lossless DSD |

### 15.3 Lossless Round-Trip Verification
```bash
# Encode
ffmpeg -i original.wav -c:a wavpack -compression_level 8 output.wv

# Decode back
ffmpeg -i output.wv -c:a pcm_s32le decoded.wav

# Compare checksums
ffmpeg -i original.wav -c:a pcm_s32le original_raw.wav
md5sum original_raw.wav decoded.wav   # Must match for true lossless

# Reference encoder verification with MD5
wavpack original.wav -h -m -o output.wv
wavunpack -m output.wv  # Computes and displays MD5 comparison
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| WavPack (reference) | C | BSD 3-Clause | Reference (10/10) | Reference (10/10) | https://www.wavpack.com |
| FFmpeg native | C | LGPL 2.1+ | Good (8/10) | Good (8/10) | https://ffmpeg.org |
| libwavpack | C | BSD 3-Clause | Reference | Reference | Bundled with WavPack |
| ruby-wavpack | Ruby | MIT | Via libwavpack | Via libwavpack | rubygems.org |

### Build Instructions (for bundling in converter app)
```bash
# Build WavPack from source
git clone https://github.com/dbry/WavPack.git
cd WavPack
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
make -j$(nproc)
make install

# Build with static library
cmake .. -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=OFF

# Link into your project:
# LDFLAGS += -L/usr/local/lib -lwavpack
# CFLAGS  += -I/usr/local/include/wavpack
```

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **WavPack Format Specification:** https://www.wavpack.com/WavPack.pdf (David Bryant, in Salomon "Data Compression: The Complete Reference")
- **WavPack 5 Library Documentation:** https://www.wavpack.com/WavPack5LibraryDoc.pdf
- **WavPack 5 File Format:** https://github.com/dbry/WavPack/blob/master/doc/WavPack5FileFormat.pdf
- **WavPack 5 Porting Guide:** https://www.wavpack.com/WavPack5PortingGuide.pdf

### Technical Resources
- FFmpeg encoder options: `ffmpeg -h encoder=wavpack`
- Multimedia Wiki: https://wiki.multimedia.cx/index.php/WavPack
- Hydrogenaudio Wiki: https://wiki.hydrogenaudio.org/index.php?title=WavPack
- GitHub: https://github.com/dbry/WavPack
- Official CLI docs: https://www.wavpack.com/wavpack_doc.html

### Academic Papers
- David Bryant, "WavPack: A Open-Source Audio Compression System" — AES Convention, 2006
- David Salomon, "Data Compression: The Complete Reference", 4th ed., Springer, 2007 — Chapter 7.11 (WavPack by Bryant)

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] Identify FFmpeg `./configure` flags: WavPack is built-in (no external flag needed)
- [ ] Verify `ffmpeg -encoders | grep wavpack` shows wavpack encoder
- [ ] Verify `ffmpeg -decoders | grep wavpack` shows wavpack decoder
- [ ] For hybrid mode: bundle reference `wavpack`/`wvunpack` CLI tools
- [ ] Note: FFmpeg does NOT support hybrid mode or correction files

### Encoding Pipeline
- [ ] Convert input sample format to `AV_SAMPLE_FMT_S32` using libswresample
- [ ] Handle variable frame size (`ctx->frame_size == 0` for WavPack)
- [ ] Implement proper flush/drain at end of stream (send NULL frame)
- [ ] For hybrid: invoke reference `wavpack` CLI with `-b` and `-c` flags (not FFmpeg)
- [ ] For MD5 verification: use reference `wavpack -m` (FFmpeg cannot compute MD5)
- [ ] Validate input sample rate is supported
- [ ] Validate channel count ≤ 255

### Decoding Pipeline
- [ ] Implement sync word search: scan for "wvpk" magic bytes
- [ ] Handle `AVERROR(EAGAIN)` in receive_frame loop correctly
- [ ] Flush decoder at EOF (send NULL packet)
- [ ] Skip encoder-delay samples: 0 (WavPack has no delay)
- [ ] Trim padding samples: 0 (WavPack has no padding)
- [ ] Handle sample format conversion from `AV_SAMPLE_FMT_S32` to output format
- [ ] For hybrid: detect and combine .wvc correction file (reference decoder only)

### Metadata
- [ ] Read all standard tag fields (see Section 7.2 table)
- [ ] Map all tag fields through standard key mapping (Section 8.6 table)
- [ ] Read cover art as binary JPEG/PNG from APEv2
- [ ] Write all standard tag fields to APEv2 in output
- [ ] Embed cover art as APEv2 binary item
- [ ] Preserve ReplayGain tags through conversion
- [ ] Handle tag encoding: UTF-8 for APEv2
- [ ] Handle ID3v1 fallback: read if no APEv2 present

### Quality & Verification
- [ ] Implement MD5 scan via reference `wavpack -m` (FFmpeg lacks this)
- [ ] Provide bit-exact verification for lossless conversions
- [ ] Implement error detection: CRC-32 in WavPack 5 block headers
- [ ] Implement partial-file recovery: decode available blocks, skip corrupted
- [ ] Test with: silence, full-scale sine, multi-channel, high-resolution, DSD files

### Edge Cases
- [ ] Handle files with corrupt or missing block headers
- [ ] Handle files with 0 samples
- [ ] Handle DSD input: route to reference encoder (FFmpeg cannot encode DSD)
- [ ] Handle channel count mismatch: trigger downmix or error
- [ ] Handle bit depth mismatch: trigger conversion via libswresample
- [ ] Handle very short files (< 1 block)
- [ ] Handle float with infinities/NaN in hybrid mode: refuse or warn

---

## APPENDIX A: REFERENCE ENCODER CLI — COMPLETE OPTIONS REFERENCE

### A.1 wavpack Command Reference
```
wavpack [options] infile [-o outfile]

COMPRESSION OPTIONS:
  -f         Fast mode (least compression, fastest)
  -h         High quality (better compression)
  -hh        Very high quality (best compression, slower)
  -hx[n]     Extra high quality with options (n=1-6)
  -0 to -8   Compression level 0-8 (default: 2)

HYBRID MODE OPTIONS:
  -b[n]      Enable hybrid mode, n = kbps (24-9600) or bits/sample (2.0-23.9)
  -c         Create correction file (.wvc) for hybrid mode
  -cc        Maximum hybrid compression (hurts quality)
  -cn        Hybrid noise shaping (n=0-2)

OUTPUT OPTIONS:
  -o file    Set output filename
  -O dir     Set output directory
  -w "key=value"  Write APEv2 tag
  -t         Copy input file timestamp to output
  -y         Yes to all (overwrite without asking)

VERIFICATION OPTIONS:
  -m         Calculate and store MD5 checksum
  -v         Verify after encoding
  -V         Verify only (no encode output)

INPUT OPTIONS:
  --raw-pcm  Input is raw PCM (specify format with --sr, --ch, --bps)
  --wav      Force WAV input
  --aiff     Force AIFF input
  --w64      Force WAVE64 input
  --caf      Force CAF input
  --rf64     Force RF64 input

DSD OPTIONS (WavPack 5):
  --dsd16    DSD 1-bit at 1x rate
  --dsd32    DSD 1-bit at 2x rate
  --dsd64    DSD 1-bit at 4x rate
  --dsd128   DSD 1-bit at 8x rate
  --dsdiff   Input is Philips DSDIFF
  --dsf      Input is Sony DSF

RIFF OPTIONS:
  --no-riiff Do not store RIFF chunks
  --import-id3  Import ID3v1/v2 tags to APEv2
```

### A.2 wvunpack Command Reference
```
wvunpack [options] infile [-o outfile]

EXTRACTION OPTIONS:
  -o file    Set output filename
  -O dir     Set output directory
  -t         Copy input file timestamp to output
  -y         Yes to all (overwrite without asking)

OUTPUT FORMAT OPTIONS:
  --wav      Output as WAV (default)
  --w64      Output as WAVE64
  --aif      Output as AIFF
  --caf      Output as CAF
  --raw-pcm  Output as raw PCM (headerless)
  --dts      Output as DTS (for DTS CDs)

DSD OUTPUT OPTIONS:
  --dsdiff   Output as Philips DSDIFF
  --dsf      Output as Sony DSF
  --dff      Synonym for --dsdiff

VERIFICATION OPTIONS:
  -m         Verify MD5 checksum (display only)
  -v         Verify only (no output file)
  -vv        Quick verify (WavPack 5+)
  -vvv       Detailed verify (WavPack 5+)

INFO OPTIONS:
  -s         Show summary info
  -ss        Show detailed info
  -s s       Show short summary
  -f         Format output for parsing
  -x field   Extract specific tag field
  -c         Extract cuesheet

GAPLESS OPTIONS:
  --skip n   Skip first n samples
  --until n  Decode until sample n
  --all      Decode all blocks

DECODING OPTIONS:
  -b         Blind decode (ignore errors)
  -d         Delete source file (use with caution!)
  --no-check Ignore corrupt blocks
```

### A.3 wvtag Command Reference
```
wvtag [options] infile

TAGGING OPTIONS:
  -a key=value  Add tag
  -d key        Delete tag
  -m key=value  Modify existing tag
  -l            List all tags
  -r            Remove all tags

IMPORT OPTIONS:
  --import-id3   Import ID3v1/v2 tags to APEv2
  --export-id3    Export APEv2 tags to ID3v1

MASS OPERATIONS:
  -r             Process subdirectories recursively
  -y             Yes to all (overwrite without asking)

COVER ART:
  -w "Cover Art (Front)=image.jpg"   Embed cover art
  --add-cover image.jpg               Add cover art
  --remove-cover                     Remove all cover art
```

### A.4 Compression Level Details

#### WavPack CLI Compression Levels (vs FFmpeg)
| WavPack CLI | Description | FFmpeg equiv | Frame size | Decorrelation passes |
|------------|-------------|--------------|-----------|---------------------|
| -f (fast) | Fastest | `-compression_level 0` | 44100 | 2 |
| Default | Balanced | `-compression_level 2` | 22050 | 5 |
| -h (high) | Better | `-compression_level 4` | 22050 | 5 |
| -hh (very high) | Best | `-compression_level 8` | 22050 | 16 |
| -hx1 | Extra 1 | `-compression_level 8` | 22050 | 16 |
| -hx2 | Extra 2 | `-compression_level 8` | 22050 | 16 |
| -hx3 | Extra 3 | `-compression_level 8` | 22050 | 16 |
| -hx4 | Extra 4 | `-compression_level 8` | 22050 | 16 |
| -hx5 | Extra 5 | `-compression_level 8` | 22050 | 16 |
| -hx6 | Extra 6 | `-compression_level 8` | 22050 | 16 |

### A.5 Hybrid Mode Bitrate Calculations

For CD audio (44.1 kHz, 16-bit stereo):
```
bits/sample → kbps calculation:
  kbps = bits/sample × sample_rate × channels / 1000
  Example: 4.0 bits/sample × 44100 × 2 / 1000 = 352.8 kbps
```

| Hybrid bitrate (bps) | CD audio kbps | Notes |
|---------------------|---------------|-------|
| 2.0 | ~176 kbps | Minimum practical |
| 2.22 | ~196 kbps | Absolute minimum (floor) |
| 3.0 | ~265 kbps | Very low quality |
| 4.0 | ~353 kbps | Low quality |
| 5.0 | ~441 kbps | Medium quality |
| 6.0 | ~529 kbps | Acceptable for most |
| 8.0 | ~706 kbps | Near-transparent |
| 10.0 | ~882 kbps | High quality |
| 12.0 | ~1058 kbps | Very high |
| 16.0 | ~1411 kbps | Nearly lossless |
| 24.0 | ~2117 kbps | Maximum quality |

---

## APPENDIX B: DSD COMPRESSION — WavPack 5 DETAIL

### B.1 DSD Audio Fundamentals

DSD (Direct Stream Digital) is a 1-bit audio format used in SACDs:
- Sample rate: 2.8224 MHz (DSD64), 5.6448 MHz (DSD128), etc.
- 1 bit per sample (sigma-delta modulation)
- WavPack treats 8 consecutive DSD bits as one "sample" for seeking/compression

### B.2 DSD Formats Supported

| Format | Extension | Description |
|--------|-----------|-------------|
| DSDIFF | .dff | Philips DSDIFF (used by Merging Technologies) |
| DSF | .dsf | Sony DSF (used by Sony/PS3) |
| Raw DSD | .wv | WavPack with DSD flag set |

### B.3 DSD Encoding with WavPack 5

```bash
# Encode DSDIFF to WavPack DSD
wavpack input.dff -o output.wv

# Encode DSF to WavPack DSD
wavpack input.dsf -o output.wv

# Decode DSD WavPack to DSDIFF
wvunpack input.wv --dsdiff -o output.dff

# Decode DSD WavPack to DSF
wvunpack input.wv --dsf -o output.dsf

# Decode DSD to PCM (lossy conversion)
wvunpack input.wv -o output.wav
```

### B.4 DSD Limitations in WavPack

- **No hybrid mode:** DSD cannot be encoded in hybrid mode
- **Lossless only:** DSD is always lossless in WavPack
- **No FFmpeg support:** FFmpeg cannot write DSD-in-WavPack
- **Requires WavPack 5:** DSD support added in version 5.0.0

### B.5 DSD Compression Ratios

| DSD Rate | Uncompressed Size | WavPack Default | WavPack High | Compression |
|----------|-----------------|-----------------|--------------|-------------|
| DSD64 | 2.8 Mbps | ~1.54 Mbps (45%) | ~1.34 Mbps (52%) | |
| DSD128 | 5.6 Mbps | ~3.08 Mbps (45%) | ~2.69 Mbps (52%) | |
| DSD256 | 11.2 Mbps | ~6.16 Mbps (45%) | ~5.38 Mbps (52%) | |

---

## APPENDIX C: ADVANCED USAGE & WORKFLOWS

### C.1 Automated Batch Encoding

```bash
#!/bin/bash
# Batch encode all WAV files to WavPack with hybrid mode
for f in *.wav; do
    wavpack -b6.0 -c -h -m "$f"
done
# Result: .wv (lossy) + .wvc (correction) files
```

### C.2 Multi-Channel Encoding

```bash
# Encode 5.1 surround to WavPack
wavpack -h "input_5.1.wav" -o "output_5.1.wv"

# Verify channel count
wvunpack -s output_5.1.wv

# Decode and downmix to stereo
ffmpeg -i output_5.1.wv -af "pan=stereo|c0=c0|c1=c1|c2=c2<0.707*c2+0.707*c0|c3=c3<0.707*c3+0.707*c1" stereo_output.wav
```

### C.3 Archival Workflow

```bash
# 1. Create archival copy with maximum compression and MD5
wavpack original.wav -hh -m -o archive.wv

# 2. Create verification copy
wavunpack -vm archive.wv
# Output: MD5 comparison result

# 3. Create distribution copy (hybrid)
wavpack original.wav -b6.0 -c -h -o distribution.wv
# Result: distribution.wv + distribution.wvc
```

### C.4 Recompression Workflow

```bash
# Recompress existing WavPack file with better settings
wavpack existing.wv -hh -m -o recompressed.wv

# WavPack 5.7+ automatically removes old .wvc if creating lossless
```

### C.5 Integrity Checking Workflow

```bash
# Quick verify (WavPack 5+, no decode)
wvunpack -vv archive.wv

# Full verify with MD5 (requires decode)
wvunpack -mvv archive.wv

# Check stored MD5 without decode
wvtag -l archive.wv | grep MD5
```

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
