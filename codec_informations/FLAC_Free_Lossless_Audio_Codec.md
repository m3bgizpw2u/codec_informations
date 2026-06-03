# FLAC (Free Lossless Audio Codec) — Deep Technical Reference

> **Category:** Lossless Audio
> **File Extensions:** .flac, .fla
> **MIME Types:** audio/flac, audio/x-flac
> **Standardization Body:** Xiph.org / IETF CELLAR Working Group
> **Specification Document:** https://xiph.org/flac/format.html (RFC 9639)
> **Patent Status:** Patent-free
> **License:** BSD 3-Clause / Xiph.org

---

## 1. HISTORICAL CONTEXT & ORIGIN

- Created by Josh Coalson at Xiph.org (2001)
- FLAC stands for Free Lossless Audio Codec
- Design goals: fast encoding/decoding, open specification, streaming support, seekable files
- FLAC 1.0 (2001), 1.1 (2004), 1.2 (2007), 1.3 (2013), 1.4 (2021), 1.5 (2025)
- Version 1.3: stable format, added window types for LPC
- Version 1.4: added LPC with window types, faster and better compression
- Version 1.5 (February 2025): introduced multi-threaded encoding
- Current status: actively maintained by Xiph.org, standardized as RFC 9639 (December 2024)
- Wide industry adoption: streaming (Spotify uses FLAC for internal transcoding), archival, music distribution
- Supported by: FFmpeg, foobar2000, VLC, JRiver Media Center, dBpoweramp, Apple Music (not natively), etc.
- The FLAC format was first specified in December 2000; the bitstream format was considered frozen with the release of FLAC 1.0 in July 2001
- Only changes made since this first stable release are considered normative in the specification

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

- Codec category: lossless, block-based, predictive coding
- Core algorithm: linear prediction (fixed or LPC) + Rice/Golomb coding of residuals
- No proprietary extensions; fully open specification published as RFC 9639
- Subset mode vs full format: subset enforces encoder limits for maximum compatibility and true "streamability"
- Bitstream is structured with frames; each frame is independently decodable (crucial for streaming)
- No built-in metadata: uses Vorbis Comments (external metadata structure shared with Ogg Vorbis)
- Native streaming support: each frame can be decoded independently without waiting for previous frames
- All samples encoded to and decoded from FLAC MUST be in a signed representation
- Unsigned audio formats (e.g., standard WAV with unsigned 8-bit or unsigned 16-bit PCM) must be transformed to signed representation before encoding
- Integer PCM only in the bitstream; no floating-point representation at any level
- All numbers in the FLAC bitstream are big-endian, except Vorbis comment field lengths (little-endian)
- All arithmetic in encoding/decoding should use integer types to eliminate floating-point rounding errors

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File / Stream Structure

Complete byte-level layout:

```
+---------------------------+
| "fLaC" marker            |  0x66 0x4C 0x61 0x43 (4 bytes) — always present
+---------------------------+
| STREAMINFO metadata block |  Mandatory, always first metadata block (34 bytes data + 4-byte header)
| [other metadata blocks]   |  Optional: APPLICATION, SEEKTABLE, VORBIS_COMMENT, CUESHEET, PICTURE, PADDING
| [audio frames]           |  One or more frames, each independently decodable
+---------------------------+
```

Metadata block header (4 bytes): `is-last (1 bit) | block-type (7 bits) | length (24 bits)`
- `is-last = 1`: this is the last metadata block
- `is-last = 0`: more metadata blocks follow

Metadata block types (block-type field):
| Value | Block Type |
|-------|------------|
| 0 | STREAMINFO |
| 1 | PADDING |
| 2 | APPLICATION |
| 3 | SEEKTABLE |
| 4 | VORBIS_COMMENT |
| 5 | CUESHEET |
| 6 | PICTURE |
| 7-126 | Reserved |
| 127 | Forbidden (avoids confusion with frame sync code) |

### 3.2 STREAMINFO Block Layout

Complete field-by-field byte layout — total 34 bytes (excluding the 4-byte metadata header):

| Field | Bit Width | Offset | Description |
|-------|-----------|--------|------------|
| minimum block size | u(16) | 0 | Min block size in samples (16-65535, excludes last block) |
| maximum block size | u(16) | 16 | Max block size in samples (16-65535) |
| minimum frame size | u(24) | 32 | Min frame size in bytes (0 = unknown) |
| maximum frame size | u(24) | 56 | Max frame size in bytes (0 = unknown) |
| sample rate | u(20) | 80 | Sample rate in Hz (1-1048575) |
| channels - 1 | u(3) | 100 | Number of channels minus 1 (0=mono, 1=stereo, ...) |
| bits per sample - 1 | u(5) | 103 | Bit depth minus 1 (4-32 bits) |
| total samples | u(36) | 108 | Total interchannel samples (0 = unknown) |
| MD5 checksum | u(128) | 144 | MD5 of unencoded audio (0 = not computed) |

Example STREAMINFO hex (first 34 bytes after metadata header) for 44.1kHz stereo 16-bit:
```
00 10 10 00 00 0F 00 00 0F 0A C4 42 F0 00 00 00 01
3E 84 B4 18 07 DC 69 03 07 58 6A 3D AD 1A 2E 0F
```
- Min block size: 0x1000 = 4096
- Max block size: 0x1000 = 4096
- Min frame size: 0x00000F = 15
- Max frame size: 0x00000F = 15
- Sample rate: 0x0AC4 = 44100 Hz
- Channels: (0b001) + 1 = 2 channels
- Bits per sample: (0b01111) + 1 = 16 bits
- Total samples: 0x00000001 = 1
- MD5: 3E 84 B4 18 07 DC 69 03 07 58 6A 3D AD 1A 2E 0F

### 3.3 Frame Structure

A frame consists of (in order):
1. **Frame header** — variable length, contains sync code + metadata + CRC-8
2. **Subframes** — one per channel, NOT interleaved, serially coded
3. **Zero-padding bits** — as needed to byte-align
4. **Frame footer** — CRC-16 (2 bytes)

#### Frame Header Breakdown:

| Component | Bit Width | Notes |
|-----------|-----------|-------|
| Sync code | 14 bits | Always `0b11111111111110` (0x3FFE) |
| Blocking strategy | 1 bit | 0 = fixed block size, 1 = variable block size |
| Block size bits | 4 bits | Lookup table; uncommon sizes stored separately |
| Sample rate bits | 4 bits | Lookup table; uncommon rates stored separately |
| Channel assignment | 4 bits | Number of channels + stereo decorrelation type |
| Sample size bits | 3 bits | Bits per sample |
| Reserved bit | 1 bit | MUST be 0 |
| Coded number | variable | Frame number (fixed block) or sample number (variable block), UTF-8-like variable-length encoding |
| Uncommon block size | 0/8/16 bits | If block size bits = 0b0110 (8-bit) or 0b0111 (16-bit) |
| Uncommon sample rate | 0/8/16 bits | If sample rate bits = 0b1100 (8-bit kHz), 0b1101 (16-bit Hz), or 0b1110 (16-bit Hz/10) |
| Frame header CRC | 8 bits | CRC-8 with polynomial x^8 + x^2 + x^1 + x^0, initialized to 0 |

**Block Size Encoding (block size bits):**
| Value | Block Size |
|-------|------------|
| 0b0000 | Reserved |
| 0b0001 | 192 |
| 0b0010 | 576 |
| 0b0011 | 1152 |
| 0b0100 | 2304 |
| 0b0101 | 4608 |
| 0b0110 | (stored as 8-bit) + 1 |
| 0b0111 | (stored as 16-bit) + 1 |
| 0b1000-0b1111 | 256, 512, 1024, 2048, 4096, 8192, 16384, 32768 |

**Sample Rate Encoding (sample rate bits):**
| Value | Sample Rate |
|-------|------------|
| 0b0000 | From STREAMINFO only |
| 0b0001 | 88200 Hz |
| 0b0010 | 176400 Hz |
| 0b0011 | 192000 Hz |
| 0b0100 | 8000 Hz |
| 0b0101 | 16000 Hz |
| 0b0110 | 22050 Hz |
| 0b0111 | 24000 Hz |
| 0b1000 | 32000 Hz |
| 0b1001 | 44100 Hz |
| 0b1010 | 48000 Hz |
| 0b1011 | 96000 Hz |
| 0b1100 | (stored 8-bit) kHz |
| 0b1101 | (stored 16-bit) Hz |
| 0b1110 | (stored 16-bit) Hz/10 |
| 0b1111 | Forbidden |

### 3.4 Subframe Types

Each subframe has its own header specifying the encoding type. Subframe header format:

| Field | Bit Width | Description |
|-------|-----------|-------------|
| Zero bit | 1 bit | MUST be 0 (differentiates from metadata) |
| Type bits | 6 bits | Subframe type code |
| Wasted bits flag | 1 bit | 0 = no wasted bits; 1 = wasted bits follow in unary |
| Wasted bits (if flag=1) | variable | Unary-coded k-1, where k = number of wasted bits per sample |

**Subframe Type Codes (type field, bits 1-6):**
| Type Code (binary) | Type | Description |
|-------------------|------|-------------|
| 0b000000 | SUBFRAME_CONSTANT | Single sample value repeated |
| 0b000001 | SUBFRAME_VERBATIM | Uncompressed samples |
| 0b000010-0b000111 | Reserved | |
| 0b001000 | SUBFRAME_FIXED (order 0) | Fixed predictor, order 0 |
| 0b001001 | SUBFRAME_FIXED (order 1) | Fixed predictor, order 1 |
| 0b001010 | SUBFRAME_FIXED (order 2) | Fixed predictor, order 2 |
| 0b001011 | SUBFRAME_FIXED (order 3) | Fixed predictor, order 3 |
| 0b001100 | SUBFRAME_FIXED (order 4) | Fixed predictor, order 4 |
| 0b001101-0b011111 | Reserved | |
| 0b100000-0b111111 | SUBFRAME_LPC (order 1-32) | LPC with order = type - 31 |

**SUBFRAME_CONSTANT:** Stores one sample value as a signed integer with the subframe's bit depth. The decoder repeats this value for all samples in the block.

**SUBFRAME_VERBATIM:** Stores all samples unencoded, directly as signed integers. No prediction or residual coding is used.

**SUBFRAME_FIXED:** Uses a predefined predictor (coefficients are fixed, not stored). Only the predictor order (0-4) is stored in the subframe header. Warm-up samples (equal to predictor order) are stored unencoded, followed by the coded residual.

**SUBFRAME_LPC:** Uses a generic linear predictor. The subframe header stores the order (1-32). The subframe body stores:
1. Warm-up samples (order × subframe bit depth bits)
2. Coefficient precision: 4 bits, value = (precision in bits) - 1 (0b1111 is forbidden)
3. Prediction shift: 5 bits, signed two's complement, MUST be non-negative
4. Coefficients: (precision × order) bits, signed two's complement
5. Coded residual

### 3.5 Complete Field-by-Field Bitstream Map (STREAMINFO + Frame)

| Location | Field | Bit Offset | Bit Width | Type | Valid Range | Meaning |
|----------|-------|-----------|-----------|------|-------------|---------|
| Stream start | fLaC marker | 0 | 32 | u(32) | 0x664C6143 | File signature |
| Metadata block header | is-last | 32 | 1 | u(1) | 0 or 1 | Last metadata block? |
| Metadata block header | block-type | 33 | 7 | u(7) | 0-126 | Type of metadata block |
| Metadata block header | length | 40 | 24 | u(24) | varies | Bytes of metadata data |
| STREAMINFO | min block size | 64 | 16 | u(16) | 16-65535 | Minimum block size in samples |
| STREAMINFO | max block size | 80 | 16 | u(16) | 16-65535 | Maximum block size in samples |
| STREAMINFO | min frame size | 96 | 24 | u(24) | 0-16777215 | Minimum frame size in bytes |
| STREAMINFO | max frame size | 120 | 24 | u(24) | 0-16777215 | Maximum frame size in bytes |
| STREAMINFO | sample rate | 144 | 20 | u(20) | 1-1048575 | Sample rate in Hz |
| STREAMINFO | channels-1 | 164 | 3 | u(3) | 0-7 | Channels minus 1 |
| STREAMINFO | bits/sample-1 | 167 | 5 | u(5) | 3-31 | Bits per sample minus 1 |
| STREAMINFO | total samples | 172 | 36 | u(36) | 0-2^36-1 | Total interchannel samples |
| STREAMINFO | MD5 | 208 | 128 | u(128) | any | MD5 checksum of audio |
| Frame header | sync code | variable | 14 | constant | 0b11111111111110 | Frame start marker |
| Frame header | blocking strategy | variable+14 | 1 | u(1) | 0 or 1 | Fixed (0) or variable (1) block size |
| Frame header | block size bits | variable+15 | 4 | u(4) | 0b0000-0b1111 | Block size lookup |
| Frame header | sample rate bits | variable+19 | 4 | u(4) | 0b0000-0b1111 | Sample rate lookup |
| Frame header | channel bits | variable+23 | 4 | u(4) | 0b0000-0b1010 | Channels + decorrelation |
| Frame header | bit depth bits | variable+27 | 3 | u(3) | 0b000-0b111 | Bit depth lookup |
| Frame header | reserved | variable+30 | 1 | u(1) | 0 | MUST be zero |
| Frame header | coded number | variable+31 | variable | variable | UTF-8-like | Frame/sample number |
| Frame header | uncommon block size | after coded | 0/8/16 | u(8/16) | varies | If 0b0110 or 0b0111 |
| Frame header | uncommon sample rate | after block size | 0/8/16 | u(8/16) | varies | If 0b1100-0b1110 |
| Frame header | CRC-8 | end of header | 8 | u(8) | computed | Frame header checksum |
| Subframe header | zero | start | 1 | constant | 0 | MUST be 0 |
| Subframe header | type | start+1 | 6 | u(6) | 0-63 | Subframe type |
| Subframe header | wasted flag | start+7 | 1 | u(1) | 0 or 1 | Wasted bits present? |
| Subframe header | wasted k | after flag | unary | u(n) | 0+ | Wasted bits = k, stored as k-1 unary |
| Subframe LPC | coeff precision | after warmup | 4 | u(4) | 0b0000-0b1110 | (precision bits)-1 |
| Subframe LPC | prediction shift | after precision | 5 | s(5) | 0-15, non-neg | Right shift amount |
| Subframe LPC | coefficients | after shift | varies | s(n) | signed | Coefficients array |
| Residual | coding method | after subframe header | 2 | u(2) | 0b00=4-bit Rice, 0b01=5-bit Rice | Residual coding type |
| Residual | partition order | after method | 4 | u(4) | 0-8 (subset) | Partition count = 2^order |
| Residual partition | rice param | start of partition | 4 or 5 | u(4/5) | 0-14 (4-bit) or 0-30 (5-bit), 0b1111/0b11111=escape | Rice parameter |
| Residual partition | escape bits | after escape | 5 | u(5) | 0-31 | Bits per sample for raw |
| Residual partition | raw samples | after param | varies | s(n) | signed | Encoded residual samples |
| Frame footer | CRC-16 | end of frame | 16 | u(16) | computed | Frame checksum |

### 3.6 Sample Format Support

- **Bit depths:** 4-bit to 32-bit integer PCM (4-bit quantized PCM is valid per spec; -1 encoded as `bits-1 = 3`)
- **Sample rates:** 1 Hz to 1,048,575 Hz (subset limits to multiples of 10 for rates < 655360 Hz)
- **Channels:** 1 to 8 channels (max 8 in subset)
- **Integer PCM only:** No floating-point representation anywhere in the bitstream
- **Signed representation required:** All samples must be converted from unsigned to signed before encoding
- **Sample rate 0:** Allowed only for non-audio data; MUST NOT be 0 for audio content

---

## 4. ENCODING ALGORITHM (DEEP DETAIL)

### 4.1 Pre-Processing Stage

**Blocking:** Dividing input into contiguous blocks (frames). The block size directly affects compression ratio:
- Block size too small → many frames → disproportionate frame header overhead
- Block size too large → signal characteristics vary too much → poor predictor fit
- Minimum block size: 16 samples (except last block which can be smaller)
- Maximum block size: 65535 samples
- Minimum for subset: 16 samples; maximum for subset at ≤48kHz: 4608 samples
- Common block sizes: 1152 (CD-DA friendly), 4096, 8192

**Blocking strategy:** FLAC files use either fixed block size throughout or variable block size:
- **Fixed block size:** Frame headers contain frame numbers (0, 1, 2, ...). Simpler, more compatible.
- **Variable block size:** Frame headers contain sample numbers. Allows optimal block size per region. More complex, less compatible.

The blocking strategy bit in the frame header MUST NOT change during the stream.

**Unsigned to signed conversion:** Audio with unsigned representation (most formats) must be converted. For signed representation, values should be centered around zero. For example, unsigned 16-bit PCM (range 0-65535) becomes signed 16-bit PCM (range -32768 to +32767) by subtracting 32768.

### 4.2 Channel Coupling (Interchannel Decorrelation)

For stereo (2-channel) content, FLAC can transform the left-right signal before encoding to remove redundancy:

**Independent:** Both channels coded independently without transformation. Mandatory for non-stereo (3+ channels).

**Mid-Side (MS) stereo:**
- Mid = (Left + Right) >> 1
- Side = Left - Right
- On decode: Mid << 1, and if Side is odd, add 1 to Mid. Left = (Mid + Side) >> 1, Right = (Mid - Side) >> 1
- Mid channel shifted right 1 bit loses 1 LSB, but that bit is recoverable from the Side channel

**Left-Side (LS) stereo:**
- Left channel stored as-is (independent)
- Side = Left - Right
- On decode: Right = Left - Side

**Side-Right (SR) stereo:**
- Right channel stored as-is (independent)
- Side = Left - Right
- On decode: Left = Right + Side

**Bit depth implications:** The Side channel needs one extra bit of bit depth because Left - Right can be twice as large as individual samples.

**Channel assignment codes in frame header:**
| Code | Meaning |
|------|---------|
| 0b0000 | 1 channel: mono |
| 0b0001 | 2 channels: left, right (independent) |
| 0b0010 | 3 channels: left, right, center |
| 0b0011 | 4 channels: FL, FR, BL, BR |
| 0b0100 | 5 channels: FL, FR, FC, SL, SR |
| 0b0101 | 6 channels: FL, FR, FC, LFE, SL, SR |
| 0b0110 | 7 channels: FL, FR, FC, LFE, BC, SL, SR |
| 0b0111 | 8 channels: FL, FR, FC, LFE, BL, BR, SL, SR |
| 0b1000 | 2 channels: left, right (left-side stereo) |
| 0b1001 | 2 channels: left, right (side-right stereo) |
| 0b1010 | 2 channels: left, right (mid-side stereo) |
| 0b1011-0b1111 | Reserved |

### 4.3 Linear Prediction (LPC Encoding)

**The prediction model:** For a sample at position n, the predicted value is:
```
s'[n] = sum(a[i] * s[n-i]) for i=1 to order
```
Then the residual: `r[n] = s[n] - s'[n]`

The prediction is scaled by right-shifting: `s'[n] = (sum(a[i] * s[n-i])) >> shift`

**Fixed predictors (orders 0-4):** Coefficients are predefined and not stored in the bitstream:

| Order | Prediction Formula | Description |
|-------|-------------------|-------------|
| 0 | 0 | No prediction (always zero) |
| 1 | a(n-1) | First-order (similar to DPCM) |
| 2 | 2*a(n-1) - a(n-2) | Linear extrapolation |
| 3 | 3*a(n-1) - 3*a(n-2) + a(n-3) | Second-order polynomial |
| 4 | 4*a(n-1) - 6*a(n-2) + 4*a(n-3) - a(n-4) | Third-order polynomial |

**LPC predictors (orders 1-32):** Generic coefficients stored in the bitstream. The encoder:
1. Computes autocorrelation of the input signal
2. Uses Levinson-Durbin recursion to solve for coefficients
3. Quantizes coefficients to configurable precision (1-15 bits each)
4. Determines optimal shift amount
5. Encodes warm-up samples (first `order` samples stored raw)
6. Encodes coefficients with precision and shift parameters

**Coefficient precision:** 4-bit field, value = (precision in bits) - 1. Range: 1-15 bits per coefficient. Value 0b1111 (precision = 16) is **forbidden**.

**Prediction shift:** 5-bit signed field. MUST NOT be negative (shifting right by a negative amount is undefined behavior in C).

**Coefficient storage:** Stored big-endian, signed two's complement. The first coefficient multiplies the most recent past sample (s[n-1]), the second multiplies s[n-2], and so on.

### 4.4 Residual Encoding (Rice/Golomb Coding)

FLAC uses Rice coding, a subset of Golomb coding, for residuals. Rice coding efficiently represents numbers that are typically small with occasional outliers.

**Process for each residual sample:**
1. **Zigzag encoding (folding):** Convert signed residual to unsigned "folded" value:
   - Even folded value: `folded = residual >> 1`
   - Odd folded value: `folded = -(residual >> 1)`
   - Equivalently: `folded = (residual >= 0) ? (2 * residual) : (-2 * residual - 1)`

2. **Split by Rice parameter:** Divide folded value by `2^rice_param`:
   - Quotient = `folded >> rice_param`
   - Remainder = `folded & ((1 << rice_param) - 1)`

3. **Unary encoding of quotient:** Store `quotient` zeros followed by a one bit
   - Quotient of 0 → `1`
   - Quotient of 3 → `0001`
   - This prevents sync code patterns (`111111...`) from appearing in unary data

4. **Binary encoding of remainder:** Store remainder in `rice_param` bits

**Rice parameter selection:** Encoder chooses the parameter that minimizes coded size (estimated by residual entropy). Two methods:
- **4-bit Rice (0b00):** Parameter encoded in 4 bits (0-14, 15 = escape code)
- **5-bit Rice (0b01):** Parameter encoded in 5 bits (0-30, 31 = escape code)

**Partitioning:** Residuals are divided into `2^partition_order` partitions. Each partition has its own Rice parameter. This adapts to varying residual statistics within a block.

**Escape codes:** If a partition's residuals don't fit well in Rice coding (e.g., parameter would need to exceed 14 or 30), the encoder can store them raw:
- 4-bit: parameter = 0b1111 (escape), followed by 5-bit field count, then raw signed samples
- 5-bit: parameter = 0b11111 (escape), followed by 5-bit field count, then raw signed samples

**Partition layout:**
- First partition: `(block_size >> partition_order) - predictor_order` samples
- Subsequent partitions: `block_size >> partition_order` samples each

**Partition order constraints:**
- Must result in equal partition sizes (block_size must be divisible by 2^order)
- Must satisfy: `(block_size >> order) > predictor_order`
- Subset limit: partition_order ≤ 8

### 4.5 Compression Settings / Levels

FLAC compression levels 0-8 (reference encoder) control encoding aggressiveness:

| Level | Focus | Block Size | Prediction | Notes |
|-------|-------|------------|------------|-------|
| 0 | Speed | 1152 | None/Fixed | Fastest, largest files |
| 1 | Speed | 1152 | Fixed | Very fast |
| 2 | Speed | 1152 | Fixed | Fast |
| 3 | Balanced | 1152 | Fixed | Moderate-fast |
| 4 | Balanced | 4096 | Fixed | Moderate |
| 5 (default) | Balanced | 4096 | LPC | Best balance — recommended |
| 6 | Compression | 4096 | LPC | Slower, marginal improvement |
| 7 | Compression | 8192 | LPC | Significantly slower |
| 8 | Maximum | 8192 | LPC | Slowest, best compression |

**FFmpeg compression levels 0-12:**
- FFmpeg's FLAC encoder uses compression_level 0-12 (default: 5)
- Levels 0-4 map roughly to reference encoder levels 0-4
- Levels 5-12 enable LPC with increasing analysis passes
- FFmpeg adds options like `ch_mode` (stereo decorrelation), `lpc_passes` (Cholesky passes), `multi_dim_quant` (2-stage LPC)

**Encoding speed vs compression tradeoffs:**
- Level 0 → Level 8 saves approximately 8% file size on typical album
- Level 8 encoding takes 5-10x longer than Level 0
- Decoding speed is essentially constant across all levels (150-200x real-time)
- Level 5 is the industry standard default

---

## 5. DECODING ALGORITHM

### 5.1 Stream Synchronization

**Finding frame sync:** The decoder scans for the 15-bit sync code `0b11111111111110` (0x3FFE / 0xFFF8 as first two bytes with strategy bit):
- Fixed block size: first two bytes = `0xFFF8`
- Variable block size: first two bytes = `0xFFF9`
- The blocking strategy bit differentiates: byte 2 bit 7 is 0 for fixed, 1 for variable

**CRC validation:** After parsing the frame header, the decoder computes CRC-8 and compares against the stored value. If mismatch, the sync point was a false positive — continue scanning for next sync code.

**Resync strategy:** On CRC failure, skip to the next potential sync code position. In practice, the decoder scans byte-by-byte or more efficiently by looking for the unique `0xFF` followed by `0xF8` or `0xF9` pattern.

### 5.2 Core Decoding Pipeline

1. **Find and validate frame header:**
   - Locate sync code `0xFF F8` or `0xFF F9`
   - Parse blocking strategy, block size, sample rate, channels, bit depth
   - Decode variable-length frame/sample number
   - Parse uncommon block size or sample rate if indicated
   - Validate CRC-8; if invalid, resync

2. **Decode each channel's subframe:**
   - Read subframe header (type, wasted bits flag)
   - Skip wasted bits if indicated (restore by left-shifting after decode)
   - Decode subframe type:
     - CONSTANT: read one sample, repeat for all blocks
     - VERBATIM: read all samples raw
     - FIXED: read warm-up samples, decode residual, apply fixed predictor
     - LPC: read warm-up samples, read precision/shift/coeffs, decode residual, apply LPC predictor

3. **Apply linear prediction:**
   - For FIXED: add residual to predefined predictor output
   - For LPC: compute dot product of coefficients with previous samples, right-shift, add residual

4. **Restore channel decorrelation (if applicable):**
   - For mid-side: reconstruct left/right from mid/side
   - For left-side: reconstruct right from left/side
   - For side-right: reconstruct left from right/side

5. **Restore wasted bits:** Left-shift samples by the wasted bits count (if any)

6. **Validate frame CRC-16:** Compute CRC over entire frame (excluding the CRC-16 itself), compare to stored value

7. **Output samples:** Interleave channels (if needed) and deliver to audio output

### 5.3 Inverse Linear Prediction

**Fixed predictors (exact reversal):**
```
s[n] = r[n] + pred(n)
```
Where `pred(n)` uses the predefined coefficients. Order 0 is always zero.

**LPC predictors:**
```
s[n] = r[n] + (sum(coeff[i] * s[n-1-i] for i in 0..order-1) >> shift)
```

**Clipping prevention:** After final restoration, samples must be clipped to the valid range for the bit depth. For example, 16-bit PCM must be in range [-32768, 32767].

**Warm-up samples:** The first `order` samples in an LPC/FIXED subframe are stored unencoded (as warm-up), providing the initial state for the predictor chain.

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native FLAC Container

**Standalone FLAC:** The native `.flac` container provides:
- Metadata blocks (STREAMINFO, Vorbis Comments, SeekTable, etc.)
- Frame-based audio data
- Frame-level seeking via SeekTable
- No additional framing needed

**OGG_FLAC (OGG mapping):**
- First packet: STREAMINFO in OGG headers (79 bytes fixed)
- Codec ID in OGG: `vorbis` identification header replaced by `fLaC` signature
- Granule position: sample number of last completed frame
- Metadata: Vorbis comment follows first header packet
- RFC 5334 defines the media type `audio/ogg` for Ogg-encapsulated FLAC

**Matroska/MKA:**
- Codec ID: `A_FLAC`
- CodecPrivate: all metadata blocks before first audio frame
- Each FLAC frame treated as single Matroska frame/block
- Supports variable block size and mid-stream property changes via chaining

**FLAC in MP4:** Not natively supported. A separate specification exists for encapsulating FLAC in ISO Base Media File Format (MP4/M4A), storing FLAC frames within `alc` boxes. This is defined in `doc/isoflac.txt` from the FLAC project.

### 6.2 Compatible Containers

| Container | Metadata System | Seeking | Streaming | Notes |
|-----------|---------------|---------|----------|-------|
| Native `.flac` | Vorbis Comments | Frame-based (SeekTable) | Native | Reference container |
| OGG | Vorbis Comments | Via granule position | Native | OGG_FLAC variant, RFC 5334 |
| Matroska/MKA | Matroska Tags | Block-based | Native | CodecPrivate stores metadata |
| WebM | Matroska Tags | Block-based | Native | Same as Matroska |
| RF64/WAV | N/A | No | No | Not recommended; use native FLAC |
| AIFF | ID3/NONE | No | No | Foreign metadata stored via APPLICATION block |

---

## 7. METADATA ARCHITECTURE

### 7.1 Vorbis Comments in FLAC

Follows the Vorbis Comment specification (same structure as Ogg Vorbis) but stored within the FLAC metadata block format. A FLAC file MUST NOT contain more than one Vorbis comment metadata block.

**Structure (within the metadata block):**
```
4 bytes: vendor string length (little-endian u32)
N bytes: vendor string (UTF-8)
4 bytes: comment count (little-endian u32)
For each comment:
  4 bytes: field length (little-endian u32)
  M bytes: field value "KEY=value" (UTF-8)
```

**Field name rules:**
- ASCII printable characters only: U+0020 to U+007E (space through tilde)
- Excludes U+003D (= equals sign)
- Case-insensitive for parsing, preserved exactly as written
- Maximum 255 bytes per field name

**Field value rules:**
- Any UTF-8 characters allowed
- No length limit (except metadata block size limit of ~16 MiB)
- Fields can contain multiple values (repeated keys) or single values

**Vendor string:** Identifies the encoding software (e.g., "reference libFLAC 1.3.4 20190929")

### 7.2 Supported Tag Fields

| Tag Field | Vorbis Comment Key | Max Length | Encoding | Multi-value | Notes |
|-----------|-------------------|-----------|----------|-------------|-------|
| Title | TITLE | Unlimited | UTF-8 | No | Track title |
| Artist | ARTIST | Unlimited | UTF-8 | No | Primary artist |
| Album | ALBUM | Unlimited | UTF-8 | No | Album name |
| Album Artist | ALBUMARTIST | Unlimited | UTF-8 | No | Album-level artist |
| Composer | COMPOSER | Unlimited | UTF-8 | No | Composer |
| Genre | GENRE | Unlimited | UTF-8 | Yes | Multiple genres possible |
| Date | DATE | Unlimited | UTF-8 | No | Release date |
| Year | YEAR | 4 chars | ASCII | No | Year only (legacy) |
| Track Number | TRACKNUMBER | Unlimited | ASCII | No | Track number |
| Track Total | TRACKTOTAL | Unlimited | ASCII | No | Total tracks |
| Disc Number | DISCNUMBER | Unlimited | ASCII | No | Disc number |
| Disc Total | DISCTOTAL | Unlimited | ASCII | No | Total discs |
| Comment | COMMENT | Unlimited | UTF-8 | Yes | User comments |
| Lyrics | LYRICS | Unlimited | UTF-8 | Yes | Synchronized/unsynced |
| BPM | BPM | Unlimited | ASCII | No | Beats per minute |
| Compilation | COMPILATION | 1 char | ASCII | No | 0 or 1 |
| Copyright | COPYRIGHT | Unlimited | UTF-8 | No | Copyright info |
| ISRC | ISRC | 12 chars | ASCII | Yes | Track ISRC codes |
| Label | LABEL | Unlimited | UTF-8 | Yes | Record label |
| Catalog Number | CATALOGNUMBER | Unlimited | UTF-8 | Yes | Label catalog # |
| Barcode | BARCODE | Unlimited | ASCII | No | UPC/EAN |
| ASIN | ASOC | 10 chars | ASCII | No | Amazon ASIN |
| Album Sort | ALBUMSORT | Unlimited | UTF-8 | No | For alphabetical sorting |
| Artist Sort | ARTISTSORT | Unlimited | UTF-8 | No | Sort key for artist |
| Title Sort | TITLESORT | Unlimited | UTF-8 | No | Sort key for title |
| Album Artist Sort | ALBUMARTISTSORT | Unlimited | UTF-8 | No | Sort key for album artist |
| ReplayGain Track Gain | REPLAYGAIN_TRACK_GAIN | Unlimited | ASCII | No | e.g., "+1.23 dB" |
| ReplayGain Track Peak | REPLAYGAIN_TRACK_PEAK | Unlimited | ASCII | No | e.g., "0.567890" |
| ReplayGain Album Gain | REPLAYGAIN_ALBUM_GAIN | Unlimited | ASCII | No | e.g., "-3.21 dB" |
| ReplayGain Album Peak | REPLAYGAIN_ALBUM_PEAK | Unlimited | ASCII | No | e.g., "0.789012" |
| MusicBrainz Track ID | MUSICBRAINZ_TRACKID | 36 chars | ASCII | No | MB Track ID |
| MusicBrainz Album ID | MUSICBRAINZ_ALBUMID | 36 chars | ASCII | No | MB Album ID |
| MusicBrainz Artist ID | MUSICBRAINZ_ARTISTID | 36 chars | ASCII | Yes | MB Artist IDs |
| MusicBrainz Release ID | MUSICBRAINZ_RELEASEGROUPID | 36 chars | ASCII | No | MB Release Group ID |
| MusicBrainz Track MBID | MUSICIP_PUID | 32 chars | ASCII | No | AcoustID equivalent |
| Encoder | ENCODER | Unlimited | UTF-8 | No | Encoding software |
| Encoder Settings | ENCODEDBY | Unlimited | UTF-8 | No | Encoding settings |
| Channel Mask | WAVEFORMATEXTENSIBLE_CHANNEL_MASK | 5 chars | ASCII | No | Hex mask (non-streamable) |
| Source | SOURCE | Unlimited | UTF-8 | Yes | Source medium |

### 7.3 SeekTable Metadata Block

Each seekpoint is exactly 18 bytes:

| Field | Width | Description |
|-------|-------|-------------|
| Sample number | 64 bits (u64) | First sample of target frame; 0xFFFFFFFFFFFFFFFF = placeholder |
| Stream offset | 64 bits (u64) | Bytes from first frame header to target frame header |
| Frame samples | 16 bits (u16) | Number of samples in target frame |

**Properties:**
- Sorted ascending by sample number
- All sample numbers must be unique (except placeholder points)
- All placeholder points must be at the end of the table
- 1% resolution seek table adds less than 2 KB per file

### 7.4 CUE Sheet Metadata Block

Stores CD-DA cue sheet for precise track/index navigation:

| Field | Width | Description |
|-------|-------|-------------|
| Media catalog number | 128 bytes (ASCII) | Right-padded with 0x00; CD-DA: 13 digits + 115 nulls |
| Lead-in samples | 64 bits (u64) | Samples from first sample to first INDEX 01 |
| CD-DA flag | 1 bit | 1 = CD-DA, 0 = other |
| Reserved | 7 + 258*8 bits | Must be zero |
| Track count | 8 bits (u8) | 1-100 (CD-DA), at least 1 (lead-out required) |

**Cuesheet Track:**
| Field | Width | Description |
|-------|-------|-------------|
| Track offset | 64 bits (u64) | Samples relative to stream start |
| Track number | 8 bits (u8) | 1-99, 170 for CD-DA lead-out, 255 for non-CD-DA |
| ISRC | 12 bytes (ASCII) | 12-digit alphanumeric or 12×0x00 |
| Track type | 1 bit | 0 = audio, 1 = non-audio |
| Pre-emphasis | 1 bit | 0 = none, 1 = pre-emphasis |
| Reserved | 6 + 13*8 bits | Must be zero |
| Index count | 8 bits (u8) | 0 for lead-out, 1-100 for tracks |

**Index Point:**
| Field | Width | Description |
|-------|-------|-------------|
| Offset | 64 bits (u64) | Samples relative to track start |
| Index number | 8 bits (u8) | 0 = pre-gap, 1+ = normal indices |
| Reserved | 3 bytes | Must be zero |

### 7.5 Picture Metadata Block

Stores embedded images (album art, etc.):

| Field | Width | Description |
|-------|-------|-------------|
| Picture type | 32 bits (u32) | See picture type table |
| MIME type length | 32 bits (u32) | Bytes of MIME type string |
| MIME type | variable | e.g., "image/jpeg", "image/png", or "-->" for URI |
| Description length | 32 bits (u32) | Bytes of description string |
| Description | variable | UTF-8 description |
| Width | 32 bits (u32) | Pixels (0 if unknown) |
| Height | 32 bits (u32) | Pixels (0 if unknown) |
| Color depth | 32 bits (u32) | Bits per pixel (0 if unknown) |
| Colors used | 32 bits (u32) | 0 for non-indexed, palette size for indexed |
| Picture data length | 32 bits (u32) | Bytes of picture data |
| Picture data | variable | Raw image bytes or URI |

**Picture Types:**
| Value | Type |
|-------|------|
| 0 | Other |
| 1 | 32×32 PNG file icon |
| 2 | General file icon |
| 3 | Front cover |
| 4 | Back cover |
| 5 | Liner notes |
| 6 | Media label |
| 7 | Lead artist |
| 8 | Artist/performer |
| 9 | Conductor |
| 10 | Band/orchestra |
| 11 | Composer |
| 12 | Lyricist |
| 13 | Recording location |
| 14 | During recording |
| 15 | During performance |
| 16 | Movie/video capture |
| 17 | Fish (from ID3v2 compatibility) |
| 18 | Illustration |
| 19 | Band/artist logotype |
| 20 | Publisher/studio logotype |

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 Codec Identifiers

- **FFmpeg format name:** `flac`
- **Codec ID:** `AV_CODEC_ID_FLAC`
- **Encoder name:** `flac`
- **Decoder names:** `flac` (native), `flacnative`

### 8.2 FFmpeg Encoding — Full CLI Reference

```bash
# Basic FLAC encoding
ffmpeg -i input.wav -c:a flac output.flac

# Compression levels (0-12, default 5)
ffmpeg -i input.wav -c:a flac -compression_level 0 output.flac  # fastest
ffmpeg -i input.wav -c:a flac -compression_level 5 output.flac  # default
ffmpeg -i input.wav -c:a flac -compression_level 12 output.flac  # best compression

# Block size control (frame_size option)
ffmpeg -i input.wav -c:a flac -frame_size 4096 output.flac

# LPC algorithm selection
ffmpeg -i input.wav -c:a flac -lpc_type none output.flac    # no LPC
ffmpeg -i input.wav -c:a flac -lpc_type fixed output.flac   # fixed coefficients
ffmpeg -i input.wav -c:a flac -lpc_type levinson output.flac # Levinson recursion
ffmpeg -i input.wav -c:a flac -lpc_type cholesky output.flac # Cholesky decomposition

# LPC passes (for cholesky)
ffmpeg -i input.wav -c:a flac -lpc_type cholesky -lpc_passes 4 output.flac

# LPC coefficient precision (1-15, default 15)
ffmpeg -i input.wav -c:a flac -lpc_coeff_precision 12 output.flac

# Partition order range (-1=auto, 0-8)
ffmpeg -i input.wav -c:a flac -min_partition_order 2 -max_partition_order 8 output.flac

# Prediction order method
ffmpeg -i input.wav -c:a flac -prediction_order_method 4level output.flac

# Stereo decorrelation mode
ffmpeg -i input.wav -c:a flac -ch_mode auto output.flac      # encoder decides
ffmpeg -i input.wav -c:a flac -ch_mode indep output.flac    # independent
ffmpeg -i input.wav -c:a flac -ch_mode left_side output.flac
ffmpeg -i input.wav -c:a flac -ch_mode mid_side output.flac

# Specify sample format explicitly (16-bit or 32-bit)
ffmpeg -i input.wav -c:a flac -sample_fmts s16 output.flac   # 16-bit
ffmpeg -i input.wav -c:a flac -sample_fmts s32 output.flac   # 32-bit

# Multi-channel with explicit layout
ffmpeg -i input.wav -c:a flac -channel_layout 5.1 output.flac

# Copy metadata from source (no re-encoding of audio)
ffmpeg -i input.flac -c:a copy -metadata title="New Title" output.flac

# Encode from ALAC source (lossless round-trip)
ffmpeg -i input.m4a -c:a flac output.flac

# Encode with verification (compare MD5)
ffmpeg -i input.wav -c:a flac -f md5 - < /dev/null  # encode and check
```

**Full FLAC encoder options table:**

| Option | Type | Default | Range | Effect |
|--------|------|---------|-------|--------|
| compression_level | int | 5 | 0-12 | Overall encoding effort (sets defaults for other options) |
| frame_size | int | auto | 16-65535 | Samples per block per channel |
| lpc_type | int | auto | -1 to 3 | LPC algorithm: -1=auto, 0=none, 1=fixed, 2=levinson, 3=cholesky |
| lpc_passes | int | 2 | 1-INT_MAX | Cholesky passes for LPC analysis |
| lpc_coeff_precision | int | 15 | 0-15 | Bits per LPC coefficient (0=auto) |
| min_prediction_order | int | auto | -1 to 15 | Minimum predictor order (-1=auto) |
| max_prediction_order | int | auto | -1 to 15 | Maximum predictor order (-1=auto) |
| prediction_order_method | int | auto | -1 to 5 | Search method: -1=auto, 0=estimation, 1=2level, 2=4level, 3=8level, 4=full search, 5=log search |
| min_partition_order | int | -1 | -1 to 8 | Minimum partition order (-1=auto) |
| max_partition_order | int | -1 | -1 to 8 | Maximum partition order (-1=auto) |
| ch_mode | int | -1 | -1 to 4 | Stereo mode: -1=auto, 0=indep, 1=left_side, 2=side_right, 3=mid_side |
| exact_rice_parameters | int | 0 | 0 or 1 | Use exact Rice parameter estimation |
| multi_dim_quant | int | 0 | 0 or 1 | Two-stage LPC quantization |

### 8.3 FFmpeg Decoding

```bash
# Decode to WAV (uncompressed)
ffmpeg -i input.flac output.wav

# Decode to specific format
ffmpeg -i input.flac -c:a pcm_s16le output.wav    # 16-bit signed LE PCM
ffmpeg -i input.flac -c:a pcm_s24le output.wav    # 24-bit signed LE PCM
ffmpeg -i input.flac -c:a pcm_f32le output.wav    # 32-bit float PCM

# Decode with channel layout
ffmpeg -i input.flac -channel_layout 5.1 output.wav

# Stream info only (no decode)
ffprobe -v quiet -show_format -show_streams input.flac

# Detailed stream info
ffprobe -v verbose -show_format -show_streams -show_frames input.flac

# Extract frame count
ffprobe -v quiet -show_entries stream=nb_frames -of default=nokey=1:noprint_wrappers=1 input.flac

# Extract sample count
ffprobe -v quiet -show_entries stream=duration -of default=nokey=1:noprint_wrappers=1 input.flac

# Decode and compute MD5 for verification
ffmpeg -i input.flac -f md5 - 2>/dev/null

# Decode specific time range
ffmpeg -i input.flac -ss 00:01:30 -t 00:00:30 output.wav
```

### 8.4 FFmpeg Metadata Handling

**Reading metadata:**
```bash
# List all metadata
ffprobe -v quiet -show_format input.flac

# Read specific tag
ffprobe -v quiet -show_entries format_tags=title,artist,album -of json input.flac
```

**Writing metadata:**
```bash
# Add/replace tags
ffmpeg -i input.flac -c:a copy -metadata title="Track Title" \
       -metadata artist="Artist Name" \
       -metadata album="Album Name" \
       -metadata date="2024" \
       -metadata track="1" \
       -metadata genre="Electronic" \
       output.flac

# Copy all metadata from source
ffmpeg -i input.flac -c:a copy -map_metadata 0 output.flac

# Clear all metadata
ffmpeg -i input.flac -c:a copy -map_metadata -1 output.flac

# Embed cover art (via PICTURE block)
ffmpeg -i input.flac -i cover.jpg -c:a copy -c:v copy \
       -metadata:s:v title="Album Art" \
       -metadata:s:v comment="Cover (Front)" \
       output.flac
```

**Preserving metadata during transcode:**
```bash
# Transcode but preserve all metadata and artwork
ffmpeg -i input.flac -c:a flac -compression_level 8 \
       -map_metadata 0 -id3v2_version 0 \
       output.flac
```

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seeking Architecture

**Frame-based seeking:** Each FLAC frame starts with a 15-bit sync code and contains enough information (sample rate, bit depth, block size) to decode independently. This makes seeking possible without decoding from the beginning.

**Three seeking strategies:**

1. **With SeekTable:** Most efficient. SeekTable contains pre-computed offsets to each frame (or every Nth frame). Decoder jumps to nearest seekpoint, then decodes forward/backward to exact position. Complexity: O(1) to locate seekpoint + O(N) to nearest frame.

2. **Without SeekTable — binary search:** Decoder performs binary search on frame positions, checking each frame header's CRC-8. This requires scanning but is faster than linear search. Complexity: O(log N) frames checked.

3. **Without SeekTable — linear scan:** Decoder scans byte-by-byte looking for sync codes, then validates with CRC-8. Slowest but most robust. Used as fallback.

**SeekTable resolution:** Each seekpoint is 18 bytes. A seekpoint per frame at 44.1kHz/1152 samples means ~38 seekpoints per second. At 1% resolution, a 5-minute album needs ~300 seekpoints (~5.4 KB).

**Sample-accurate seeking:** Frame headers contain either frame number (fixed block) or sample number (variable block), enabling sample-accurate positioning.

### 9.2 Gapless Playback

FLAC natively supports gapless playback without any special metadata:

- Each frame knows its starting sample number
- Frames are byte-aligned and independently decodable
- The encoder delay (samples before first audio in the source) is handled by the encoder writing a pre-roll of silent samples or by the decoder understanding the source format's delay
- Players can calculate exact sample positions for seamless track transitions
- No special tags or metadata needed; the frame numbering provides everything needed

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

**Latency:**
- Minimum block size of 16 samples → ~0.36ms at 44.1kHz
- Minimum for streaming subset: 192 samples → ~4.4ms at 44.1kHz
- Common block size 1152 → ~26ms at 44.1kHz
- This makes FLAC viable for live monitoring and low-latency applications

**Streaming architecture:**
- Frames are independent — no inter-frame dependencies
- A decoder can start decoding mid-stream after receiving any frame header + sync validation
- HTTP Range requests work naturally: each frame can be fetched independently
- Ogg FLAC variant provides additional streaming infrastructure (granule positions, page boundaries)

**Real-time encoding:**
- Level 0 encoding achieves 100-150x real-time at 44.1kHz stereo
- This means a single modern CPU core can encode faster than real-time even for multi-channel 192kHz audio
- Decoding is even faster: 150-200x real-time across all compression levels

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Channel Layout Support

FLAC supports 1-8 channels with standard layouts:

| Channels | Default Layout | Bits in Frame Header |
|----------|---------------|---------------------|
| 1 | Mono | 0b0000 |
| 2 | Left, Right (stereo) | 0b0001 |
| 3 | Left, Right, Center | 0b0010 |
| 4 | FL, FR, BL, BR | 0b0011 |
| 5 | FL, FR, FC, SL, SR | 0b0100 |
| 6 | FL, FR, FC, LFE, SL, SR | 0b0101 |
| 7 | FL, FR, FC, LFE, BC, SL, SR | 0b0110 |
| 8 | FL, FR, FC, LFE, BL, BR, SL, SR | 0b0111 |

For non-standard layouts (e.g., ambisonics), the `WAVEFORMATEXTENSIBLE_CHANNEL_MASK` Vorbis comment field can specify the exact channel mapping. Files using this tag are **not streamable** (the mask is not repeated in every frame header).

### 11.2 Channel Coupling

Stereo decorrelation (mid-side, left-side, side-right) is applied per-frame:
- The encoder chooses the best decorrelation method for each block
- The frame header's channel assignment bits indicate which method is used
- The decoder automatically reverses the transformation
- Benefits: typically 5-15% smaller stereo files
- Non-stereo (3+ channels): always independent coding
- Not all frames use decorrelation; some blocks encode channels independently

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

- **Bit depths:** 4-bit to 32-bit integer PCM natively supported
  - 4-bit: exotic, used for companded audio
  - 8-bit: unsigned standard, must convert to signed
  - 12-bit: DAT, DV, some broadcast formats
  - 16-bit: CD-DA standard
  - 20-bit: DVD-Audio, some professional formats
  - 24-bit: Studio standard, high-resolution
  - 32-bit: Maximum supported (stored as 32-bit integer, no float)
- **Sample rates:** 1 Hz to 1,048,575 Hz (theoretical max)
  - Subset limit at ≤48kHz: max 4608 samples/block
  - Subset limit for >48kHz: max 16384 samples/block
  - Subset requires sample rate to be a multiple of 10 Hz (or from lookup table)
- **32-bit audio:** Supported natively (no floating-point involved — pure 32-bit integer PCM)
- **DXD (352.8kHz):** Fully supported (exceeds subset limits but valid)
- **DSD-to-FLAC:** Not directly supported (DSD is 1-bit sigma-delta, not PCM). DSD must be converted to PCM (e.g., DSD2PCM) before FLAC encoding.

---

## 13. COMPRESSION RATIOS & BENCHMARKS

### 13.1 Typical Compression Ratios

| Content Type | Typical Ratio | Notes |
|--------------|---------------|-------|
| Classical music | 45-55% | High redundancy, long quiet passages, orchestral dynamics |
| Jazz | 50-60% | Acoustic instruments, moderate dynamics |
| Speech/audiobook | 40-50% | Very high redundancy, consistent spectral content |
| Rock/pop | 55-65% | Moderate redundancy, compressed dynamics |
| Electronic/EDM | 60-70% | Synthesized, less redundancy, heavy compression |
| Metal | 60-70% | High dynamics and distortion, less predictable |
| Folk/acoustic | 50-58% | Natural instruments, moderate complexity |

**WAV to FLAC at Level 5:**
- 60-minute album (CD-DA, 16-bit/44.1kHz stereo): ~330 MB WAV → ~180 MB FLAC (55%)
- 60-minute album at Level 8: ~165 MB FLAC (52%)

### 13.2 Encoding/Decoding Speed

| Level | Encode Speed (x RT @ 44.1kHz stereo) | Decode Speed (x RT) | Memory Usage |
|-------|-------------------------------------|---------------------|-------------|
| 0 | 100-150x RT | 150-200x RT | Low |
| 1 | 80-120x RT | 150-200x RT | Low |
| 2 | 60-100x RT | 150-200x RT | Low |
| 3 | 50-80x RT | 150-200x RT | Low-Medium |
| 4 | 40-60x RT | 150-200x RT | Medium |
| 5 (default) | 30-50x RT | 150-200x RT | Medium |
| 6 | 15-30x RT | 150-200x RT | Medium-High |
| 7 | 10-20x RT | 150-200x RT | High |
| 8 | 8-15x RT | 150-200x RT | High |

**Notes:**
- RT = Real-Time (1x = playback speed)
- Higher sample rates reduce effective multiples (192kHz is ~4.4x more samples than 44.1kHz)
- Multi-threaded encoding (FLAC 1.5+) significantly improves encode speed on multi-core
- Decoding speed is largely independent of compression level

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

**Negative LPC shift bits:**
- Early specification allowed negative prediction right shift (effectively left shift)
- Reference encoder was changed in 2007 to never generate negative shifts
- Some older decoders cannot handle negative shifts
- Modern specification forbids negative shifts
- **Recommendation:** Never encode with negative shifts; accept files with negative shifts with caution

**Non-zero FIR precision for FIXED subframes:**
- Some legacy files may have incorrect coefficient precision fields in FIXED subframes
- FIXED subframes have no coefficients in the bitstream — the precision field should be ignored
- Ensure decoders don't mistakenly read coefficient data for FIXED subframes

**Frame sync false positives:**
- The 15-bit sync code `11111111111110` can appear in encoded audio data
- CRC-8 validation of frame header is essential before trusting sync
- Implement robust resync: after CRC failure, continue scanning for next sync marker

**Side channel bit depth overflow:**
- Mid-side stereo: side channel has one extra bit of bit depth
- When combined with high-order LPC and high bit depths, intermediate calculations can overflow 32-bit
- Use 64-bit arithmetic for intermediate dot products in LPC decoding
- The spec requires 32-bit signed integers for residual processing, but predictor arithmetic needs wider types

**Variable block size interoperability:**
- Variable block size streams are less widely supported than fixed block size
- Some hardware players and older software only support fixed block size
- **Recommendation:** Use fixed block size for maximum compatibility

**Wasted bits per sample:**
- 14-bit audio in 16-bit container: 2 wasted bits per sample
- FLAC efficiently compresses these zeros
- Some decoders don't properly handle wasted bits — validate output with known test files
- Padding must be restored (left-shift) before channel decorrelation undo

**Rice escape codes:**
- Escaped partitions (raw residual storage) are rarely used
- Some decoders cannot handle escaped partitions
- Reference encoder rarely produces escaped partitions except for pathological input
- **Recommendation:** Avoid escaped partitions for maximum compatibility

**MD5 checksum edge cases:**
- MD5 is computed on interleaved, signed, little-endian PCM
- Non-byte-aligned bit depths are sign-extended to byte boundaries before MD5 computation
- The MD5 can be 0 (not computed) — this is valid
- A correct decode produces audio that matches the stored MD5

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM FLAC

| Target Format | Recommended Command | Quality Notes | Metadata Preservation |
|---------------|--------------------|--------------|---------------------|
| WAV | `ffmpeg -i input.flac output.wav` | Bit-exact (use `-acodec pcm_s16le`) | Preserve with `-map_metadata 0` |
| WAV (24-bit) | `ffmpeg -i input.flac -acodec pcm_s24le output.wav` | Bit-exact at 24-bit | Metadata in separate .cue or .txt |
| MP3 (320kbps) | `ffmpeg -i input.flac -b:a 320k output.mp3` | Perceptually transparent | Use `-map_metadata 0` |
| MP3 (V0) | `ffmpeg -i input.flac -q:a 0 output.mp3` | ~245kbps VBR | Use `-map_metadata 0` |
| AAC (256kbps) | `ffmpeg -i input.flac -b:a 256k -aac_coder treble output.aac` | High quality | Use `-map_metadata 0` |
| Opus (128kbps) | `ffmpeg -i input.flac -b:a 128k -c:a libopus output.opus` | Transparent for most | Use `-map_metadata 0` |
| ALAC | `ffmpeg -i input.flac -c:a alac output.m4a` | Bit-exact | Use `-c:a copy` on existing ALAC |
| FLAC (higher level) | `ffmpeg -i input.flac -c:a flac -compression_level 8 output.flac` | Bit-exact after re-encode | Use `-map_metadata 0` |

### 15.2 Converting TO FLAC

| Source Format | Recommended Command | Quality Notes | Metadata Preservation |
|---------------|--------------------|--------------|---------------------|
| WAV (16-bit) → | `ffmpeg -i input.wav -c:a flac output.flac` | Bit-exact | Use `-map_metadata 0` |
| WAV (24-bit) → | `ffmpeg -i input.wav -c:a flac -sample_fmts s32 output.flac` | Bit-exact at 24-bit | Use `-map_metadata 0` |
| MP3 → | `ffmpeg -i input.mp3 -c:a flac output.flac` | Source is lossy, result is lossless copy of lossy | Some tags lost, audio unchanged |
| AAC → | `ffmpeg -i input.m4a -c:a flac output.flac` | Lossless round-trip of lossy source | Some tags may not copy |
| AIFF → | `ffmpeg -i input.aiff -c:a flac output.flac` | Bit-exact | Use `-map_metadata 0` |
| ALAC → | `ffmpeg -i input.m4a -c:a flac output.flac` | Bit-exact | Use `-c:a copy` where possible |

### 15.3 Lossless Round-Trip Verification

**Frame count verification:**
```bash
# Get frame count from source
ffprobe -v quiet -select_streams a:0 -count_packets -show_entries stream=nb_read_frames -of csv=p=0 input.wav

# Compare with decoded FLAC
ffprobe -v quiet -select_streams a:0 -count_packets -show_entries stream=nb_read_frames -of csv=p=0 input.flac
```

**MD5 checksum comparison:**
```bash
# Get MD5 of original WAV
ffmpeg -i original.wav -f md5 - 2>/dev/null

# Get MD5 of decoded FLAC
ffmpeg -i encoded.flac -f md5 - 2>/dev/null

# Compare (should be identical for lossless round-trip)
```

**Binary comparison with external reference:**
```bash
# Decode FLAC and compare to original
ffmpeg -i encoded.flac -af "aformat=s16:${sample_rate}:${channels}" /tmp/decoded.wav
diff original.wav /tmp/decoded.wav && echo "IDENTICAL" || echo "DIFFERENT"
cmp original.wav /tmp/decoded.wav
```

**Expected behavior:** Bit-exact match after FLAC decode. Any difference indicates either:
1. The FLAC encoder added padding or processed samples differently
2. Sample rate or bit depth conversion was applied
3. The "lossless" comparison was done between non-identical source and decoded formats

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Quality | Notes |
|---------|----------|---------|---------|-------|
| libFLAC | C | BSD | Reference | Official Xiph.org implementation; most complete |
| libFLAC++ | C++ | BSD | High | C++ wrapper around libFLAC |
| FFmpeg libavcodec | C | LGPL | High | Native implementation; excellent integration |
| Flake | C | BSD | High | libFLAC-based with improved compression |
| libavcodec (flacenc) | C | LGPL | High | FFmpeg's encoder; supports levels 0-12 |
| CUETools.Flake | C# | LGPL | High | .NET binding via CUETools |
| tone送你 | Various | Various | Varies | Many third-party implementations |

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

- **Official FLAC format specification:** https://xiph.org/flac/format.html
- **RFC 9639 (current):** https://datatracker.ietf.org/doc/html/rfc9639 — Authoritative standard
- **FLAC on Xiph.org:** https://xiph.org/flac/
- **Reference encoder source:** https://github.com/xiph/flac
- **FLAC test files:** https://github.com/ietf-wg-cellar/flac-test-files
- **FLAC in MP4 specification:** https://github.com/xiph/flac/blob/master/doc/isoflac.txt
- **FLAC interoperability wiki:** https://github.com/xiph/flac-specification/wiki/Interoperability-considerations
- **Hydrogenaudio Knowledgebase:** https://wiki.hydrogenaud.io/ — Community FLAC documentation
- **RFC 5334 (Ogg mapping):** https://www.rfc-editor.org/rfc/rfc5334.txt
- **RFC 9559 (Matroska mapping):** https://www.rfc-editor.org/rfc/rfc9559.txt

---

## 18. IMPLEMENTATION CHECKLIST (for the Converter Developer)

### Encoding Pipeline

- [ ] FFmpeg build flags needed (default: enabled in most builds; verify with `ffmpeg -formats | grep flac`)
- [ ] Input sample format validation (FLAC bitstream requires signed PCM; convert unsigned formats)
- [ ] Encoder initialization: block size (`-frame_size`), compression level (`-compression_level`), LPC settings (`-lpc_type`, `-lpc_passes`)
- [ ] Frame size handling: verify block size divides evenly, handle final partial block correctly
- [ ] Encoder delay handling: some encoders add zero-padding at the start; track and compensate
- [ ] Multi-channel channel layout validation: verify channel count supported (1-8)
- [ ] Bit depth handling: FLAC supports 4-32 bits; ensure encoder handles the target bit depth
- [ ] Sample rate validation: verify within 1-1048575 Hz range
- [ ] Output file verification: compute MD5 of decoded output, compare to expected

### Metadata Handling

- [ ] Metadata read: all Vorbis Comment fields mapped (see Section 7.2)
- [ ] Metadata write: Vorbis Comment construction (vendor string, field count, field lengths — all little-endian)
- [ ] Cover art embed/extract: PICTURE metadata block (type 5)
- [ ] SeekTable generation: 18-byte seekpoints, sorted by sample number
- [ ] ReplayGain scan and tag write: REPLAYGAIN_TRACK_GAIN, REPLAYGAIN_TRACK_PEAK, etc.
- [ ] PADDING block: reserve space for future metadata editing without full file rewrite

### Decoding Pipeline

- [ ] Stream synchronization: find 15-bit sync code `11111111111110`, validate with CRC-8
- [ ] Error recovery: on CRC failure, resync to next frame
- [ ] Frame header parsing: all variable-length fields (UTF-8-like coded number)
- [ ] Subframe decoding: handle all 4 types (CONSTANT, VERBATIM, FIXED, LPC)
- [ ] Inverse linear prediction: 64-bit arithmetic for dot products to prevent overflow
- [ ] Channel decorrelation undo: restore left/right from mid/side (or LS/SR)
- [ ] Wasted bits restoration: left-shift after decode but before decorrelation
- [ ] Clipping prevention: clamp final samples to valid range for bit depth
- [ ] Frame CRC-16 validation: validate entire frame before output
- [ ] Bit-exact output: use integer PCM, verify MD5 against stored value

### Format Compliance

- [ ] `fLaC` marker (0x664C6143) present at stream start
- [ ] STREAMINFO block (type 0) is first metadata block
- [ ] STREAMINFO minimum block size ≤ maximum block size
- [ ] Block sizes within 16-65535 range (except last block can be smaller)
- [ ] Frame headers: sync code not corrupted, CRC-8 valid
- [ ] Subframes: type codes within valid ranges, no forbidden patterns
- [ ] LPC subframes: coefficient precision ≠ 15, shift ≥ 0
- [ ] Residual coding: partition sizes valid, Rice/escape codes valid
- [ ] Frame footer: CRC-16 valid
- [ ] Reserved bits: mandatory zero bits are zero

---

## 19. ADVANCED ALGORITHMIC DETAIL

### 19.1 The Levinson-Durbin Recursion for LPC Coefficient Computation

The FLAC reference encoder uses the Levinson-Durbin recursion to compute optimal LPC coefficients from autocorrelation coefficients. This algorithm efficiently solves the Yule-Walker equations for linear prediction.

**Autocorrelation computation:**
```
R[k] = sum(n=0 to N-k-1) of s[n] * s[n+k]
```
Where `s[n]` is the input signal and `N` is the block size.

**Levinson-Durbin recursion:**

```
Input: R[0..p] where p is the desired prediction order

a[0] = 1
e = R[0]

for m = 1 to p:
    lambda = (R[m] - sum(k=1 to m-1) of a[k] * R[m-k]) / e
    a[m] = -lambda
    for k = 1 to m-1:
        a[k] = a[k] - lambda * a[m-k]
    e = e * (1 - lambda^2)

Output: LPC coefficients a[1..p] (from a[1] = -lambda at each step)
```

**Quantization of coefficients:**
- After computing floating-point coefficients, the encoder quantizes them to a fixed precision
- The precision is encoded as `(bits - 1)` in 4 bits, allowing 1-15 bits per coefficient
- Coefficients are quantized by: `quantized = round(float_coeff * (2^precision))`
- The prediction shift is adjusted to compensate for quantization scale

**Coefficient precision selection:**
- The encoder tests multiple precision values (1-15 bits)
- For each precision, it computes the residual variance
- The precision that minimizes residual entropy is selected
- Higher precision costs more bits but produces smaller residuals

### 19.2 Rice Parameter Estimation

The optimal Rice parameter for a partition minimizes the expected coded length. For a partition with residual samples, the encoder estimates entropy and selects the parameter that minimizes:

**Theoretical optimal parameter:**
```
k_opt = round(log2(2^mean_abs_residual))
```

**Empirical estimation:**
1. Compute the sum of absolute residual values in the partition
2. Estimate mean absolute value: `m = sum|r[i]| / n`
3. Set Rice parameter: `k = ceil(log2(m + 1))`
4. Verify by computing actual coded size for neighboring parameters
5. Select the parameter with minimum total size

**4-bit vs 5-bit parameter selection:**
- 4-bit Rice (0b00): parameters 0-14, escape at 15
- 5-bit Rice (0b01): parameters 0-30, escape at 31
- The encoder may use 5-bit parameters when residuals have high dynamic range (e.g., 24-bit audio)
- Some decoders don't support 5-bit parameters; use 4-bit only for maximum compatibility

**Escape code threshold:**
- When `2^parameter > max_residual`, Rice coding is inefficient
- The encoder switches to escape coding: parameter = escape code + raw bits
- Escape code: raw samples stored with `5` bits specifying bits-per-sample
- Escape partitions are rare in practice; most encoders avoid them

### 19.3 Wasted Bits Handling in Detail

Wasted bits per sample optimize storage when audio has fewer bits than the container provides.

**Example: 14-bit PCM in 16-bit container:**
1. The WAV file stores 14-bit samples padded to 16 bits with 2 LSB zeros
2. FLAC detects these zeros and signals `wasted bits flag = 1`
3. The unary-coded value: `k - 1 = 2 - 1 = 1`, stored as `0b01`
4. The encoder codes 14-bit samples (not 16-bit)
5. The decoder shifts left by `k = 2` after decoding

**Bit depth calculation:**
```
effective_bits = frame_header_bit_depth - wasted_bits
if side_channel:
    effective_bits = effective_bits + 1
```

**Constraints:**
- Effective bit depth MUST be > 0
- Wasted bits can vary between frames and between subframes
- If wasted bits change within a frame, the subframe uses the minimum value
- This ensures lossless restoration (no non-zero bits are discarded)

### 19.4 Mid-Side Decoration Mathematics

The mid-side transformation exploits stereo correlation:

**Encoding:**
```
mid[n] = (left[n] + right[n]) >> 1    # Integer division by 2, rounding toward -inf
side[n] = left[n] - right[n]
```

**Decoding (with lossless reconstruction):**
```
# Reconstruct with full precision
if side[n] is odd:
    mid[n] = (mid[n] << 1) | 1    # Add recovered LSB
else:
    mid[n] = mid[n] << 1

left[n] = (mid[n] + side[n]) >> 1
right[n] = (mid[n] - side[n]) >> 1
```

**Why this is lossless:**
- `left + right` produces an even number when both are integers
- The right-shift loses 1 bit, but the lost LSB is recovered from `side` being odd
- This is mathematically equivalent to: `left = (mid*2 + side) / 2`

**Left-Side stereo:**
```
left[n] = stored directly
side[n] = left[n] - right[n]
right[n] = left[n] - side[n]    # Decoded
```

**Side-Right stereo:**
```
right[n] = stored directly
side[n] = left[n] - right[n]
left[n] = side[n] + right[n]    # Decoded
```

---

## 20. BINARY LAYOUT EXAMPLES

### 20.1 Complete Frame Header Dissection

Consider a frame with these bytes (hex dump):
```
FF F8 69 18 01 02 A4 02 C3 82 ...
```

**Bit-by-bit breakdown:**

| Byte(s) | Bits | Value | Field |
|---------|------|-------|-------|
| 0xFF | 11111111 | 0xFF | First byte of sync (0xFF always) |
| 0xF8 | 11111000 | 0xF8 | Second byte: 1111 (sync) + 111 (block size) + 0 (strategy) |
| 0x69 | 01101001 | 0x69 | Block size bits (0110=8-bit uncommon) + Sample rate (1001=44.1k) |
| 0x18 | 00011000 | 0x18 | Channels (0001=stereo) + Bit depth (110=24-bit) + Reserved (0) |
| 0x01 | variable | UTF-8 | Frame number = 1 (fixed block size) |
| 0x02 | 8 bits | 0x02 | Uncommon block size = 3 (stored + 1 = 3+1=4? No, stored as block_size-1) |

Wait, let me reconsider with a clearer example. Let's use the example from RFC 9639 Appendix D:

```
FF F8 69 18 00 00 BF ...
```

**Frame header parsing step by step:**

1. **Sync code:** Bytes 0xFF 0xF8 = `11111111 11111000`
   - First 14 bits: `11111111111110` ✓ Correct sync code
   - Blocking strategy bit: `0` → Fixed block size stream

2. **Block size bits:** Next 4 bits after sync = `0110`
   - 0b0110 = uncommon block size, stored as 8-bit number + 1
   - See byte after coded number for the actual size

3. **Sample rate bits:** Next 4 bits = `1001`
   - 0b1001 = 44100 Hz (from lookup table)

4. **Channel bits:** Next 4 bits = `0001`
   - 0b0001 = 2 channels, left-right, independent stereo

5. **Bit depth bits:** Next 3 bits = `110`
   - 0b110 = 24 bits per sample

6. **Reserved bit:** Next 1 bit = `0` ✓ Must be 0

7. **Coded number:** After reserved bit
   - For fixed block size, this is a frame number
   - UTF-8-like encoding: single byte 0x00 = frame 0

8. **Uncommon block size:** Since block size bits = 0b0110
   - Next byte 0x00 = block size = 0 + 1 = 1 sample per block
   - (Actually, uncommon block size is stored as block_size - 1, so 0x00 means block_size = 1)

9. **Frame header CRC:** 8 bits
   - Stored at end of header
   - Polynomial: x^8 + x^2 + x^1 + x^0

### 20.2 Subframe Header Dissection

For a FIXED order 4 subframe:
```
8C ... (subframe data follows)
```

**Binary: 0x8C = 10001100**
- Bit 7: `1` → wasted bits flag = 1
- Bits 6-1: `00010` → Type code 0b000100 = SUBFRAME_FIXED order 4
- Bit 0: part of wasted bits unary encoding

Wait, let me parse correctly:
- 0x8C = 0b10001100
- Bit 7: `1` (wasted flag)
- Bits 6-1: `00010` = decimal 2? No, need to reconsider.

The subframe header format:
```
[1 bit: zero MUST be 0] [6 bits: type] [1 bit: wasted flag]
```

So for 0x8C = 0b10001100:
- Split: `1` `000110` `0`
- Zero: `1`? That's wrong, zero MUST be 0.

Let me use a better example. From the RFC 9639 example:
```
0x80 = 0b10000000
- Zero: 1? That's invalid.
```

Actually, let me use the verbatim subframe example from the RFC:
```
0x81 = 0b10000001
- Zero: 1 → Invalid!
```

The correct parsing needs proper byte alignment. In FLAC, subframes are NOT necessarily byte-aligned after the subframe header. The header is read bit-by-bit.

For a subframe starting at byte boundary with type VERBATIM and no wasted bits:
```
[0][000001][0] = 0x02
```

For VERBATIM with wasted bits:
```
[0][000001][1][k-1 in unary]...
```

For FIXED order 2, no wasted bits:
```
[0][000010][0] = 0x04
```

For FIXED order 4, wasted bits flag = 1, k = 2:
```
[0][000100][1][01] = 0x09 0x40? No...
```

Let me show it clearly:

```
Bit positions:  7   6-1    0
Values:        0  000100  0   = 0x08 for FIXED order 4, no wasted bits

With wasted bits k=3:
Bit positions:  7   6-1    0   [unary: k-1 = 2]
Values:        0  000100  1   001
Combined:     00010011 = 0x13
```

### 20.3 Complete Residual Decoding Walkthrough

Given a partition with Rice parameter 3 and residual samples:

**Encoded partition (bits):**
```
01 001 110 011 000 100 111
```

**Rice decoding step by step:**

1. Read unary part until 1:
   - `01` → count zeros = 1 → quotient = 1

2. Read 3 bits for remainder:
   - `001` → remainder = 1

3. Reconstruct: folded = (quotient << k) | remainder
   - folded = (1 << 3) | 1 = 8 + 1 = 9

4. Unfold (zigzag decode):
   - If folded is even: residual = folded >> 1
   - If folded is odd: residual = -(folded + 1) >> 1
   - 9 is odd → residual = -(9 + 1) >> 1 = -10 >> 1 = -5

**Verification:**
The original residual must have been -5.
- Original: -5
- Zigzag: odd → -(|-5|) * 2 - 1 = -10 - 1 = -11? No.

Let me recalculate:
- residual = -5
- If residual < 0: folded = -2 * residual - 1 = -2 * (-5) - 1 = 10 - 1 = 9 ✓
- If folded is odd: residual = -(folded + 1) / 2 = -(9 + 1) / 2 = -10 / 2 = -5 ✓

**Multiple samples in partition:**
```
Partition order 0: 1 partition, all samples in one partition
Rice parameter: 3
Encoded: [3 bits][sample1][sample2][sample3]...
```

---

## 21. STREAMABLE SUBSET COMPLIANCE

### 21.1 Subset Requirements Summary

For a FLAC stream to be "streamable" (compliant with the FLAC streamable subset):

| Requirement | Constraint | Rationale |
|-------------|-----------|-----------|
| Sample rate in frame header | MUST use lookup table (0b0001-0b1110) | Decoder may not have STREAMINFO |
| Bit depth in frame header | MUST use lookup table (0b001-0b111) | Decoder may not have STREAMINFO |
| Maximum block size | ≤ 16384 samples | Hardware decoder buffer limits |
| Block size at ≤48kHz | ≤ 4608 samples | Prevents excessive pre-roll |
| LPC order at ≤48kHz | ≤ 12 (order not 13-32) | Simplicity for hardware |
| Rice partition order | ≤ 8 | Limits decoder complexity |
| Channel ordering | Standard layouts only | No WAVEFORMATEXTENSIBLE_CHANNEL_MASK needed |

### 21.2 Subset Block Size Tables

**At sample rates ≤ 48000 Hz (most common):**

| Block Size Code | Block Size | Subset Compliant |
|----------------|-----------|-----------------|
| 0b0001 | 192 | Yes |
| 0b0010 | 576 | Yes |
| 0b0011 | 1152 | Yes (CD-DA friendly) |
| 0b0100 | 2304 | Yes |
| 0b0101 | 4608 | Yes (FFmpeg default at level 5) |
| 0b0110 | varies (8-bit) | Yes if ≤ 4608 |
| 0b0111 | varies (16-bit) | No (would exceed 4608) |
| 0b1000 | 256 | Yes |
| 0b1001 | 512 | Yes |
| 0b1010 | 1024 | Yes |
| 0b1011 | 2048 | Yes |
| 0b1100 | 4096 | No (exceeds 4608) |
| 0b1101 | 8192 | No |
| 0b1110 | 16384 | No |
| 0b1111 | 32768 | No |

**At sample rates > 48000 Hz:**

All block sizes up to 16384 are allowed.

### 21.3 Subset LPC Order Limits

For audio at ≤48000 Hz, LPC subframes MUST use prediction order ≤ 12:
- Order 1-12: Allowed in subset
- Order 13-32: Not allowed in subset at ≤48000 Hz

This corresponds to subframe type codes:
- 0b100100 to 0b101011: Order 13-32 (forbidden in subset at ≤48000 Hz)
- 0b100000 to 0b101011: Order 1-32

For audio at >48000 Hz, all LPC orders 1-32 are allowed.

---

## 22. ERROR HANDLING & RECOVERY

### 22.1 Frame Synchronization Loss Recovery

When CRC-8 validation fails on a suspected frame header:

**Algorithm:**
```
1. Start scanning from last known good position
2. Look for byte pattern 0xFF followed by 0xF8 or 0xF9
3. Parse potential frame header starting at that position
4. Validate all fields are within valid ranges
5. Compute and compare CRC-8
6. If CRC matches, accept as valid frame start
7. If CRC fails, increment scan position by 1 and repeat
8. After finding valid frame, continue normal decoding
```

**Optimization: Skip to next potential sync**
- Scan for 0xFF bytes (sync code starts every frame)
- Check next byte for 0xF8 or 0xF9
- Reduces scan overhead compared to byte-by-byte

**Maximum scan distance:**
- In worst case, the sync code could appear every byte
- Maximum distance to next valid sync is unbounded
- Practical decoders limit scan to prevent infinite loops
- If no valid frame found after scanning N bytes, report error

### 22.2 CRC Mismatch Handling

**Frame header CRC-8 failure:**
- Sync point was a false positive
- Continue scanning for next sync code
- Do not attempt to decode the invalid frame

**Frame footer CRC-16 failure:**
- Frame data may be corrupted
- Most decoders: stop at corrupted frame, output silence
- Some decoders: attempt to resync and continue with next frame
- Output frame is discarded; gap in audio output

**MD5 checksum mismatch:**
- Occurs after complete decode
- Indicates corrupted audio data (even if bitstream was valid)
- Some decoders: continue playback, report error
- Others: stop with error message

### 22.3 Overflow and Underflow

**Integer overflow in LPC computation:**
- Problem: `sum(coeff[i] * sample[n-i])` can exceed 32-bit range
- For 24-bit audio with 12th-order LPC and 15-bit coefficients: needs ~44 bits
- Solution: Use 64-bit arithmetic for intermediate results

**Clipping prevention:**
- After prediction restoration, samples may exceed valid range
- Example: 16-bit range is [-32768, 32767]
- If decoded value is -40000, it must be clamped to -32768
- Clipping introduces distortion but prevents audio artifacts from overflow

**Subframe bit depth calculation errors:**
- If wasted bits calculation produces zero or negative effective bits:
  - Invalid frame, reject and resync
- If side channel bit depth increase causes overflow:
  - Use wider arithmetic, clamp final result

---

## 23. PLATFORM-SPECIFIC CONSIDERATIONS

### 23.1 Big-Endian vs Little-Endian Systems

FLAC uses big-endian encoding throughout, which affects:
- **Big-endian systems (SPARC, some ARM modes):** Direct memory mapping of multi-byte fields
- **Little-endian systems (x86, x64, most ARM):** Byte-swapping required for 16/32/64-bit fields

**Critical fields requiring byte-swap on little-endian:**
- All metadata block header fields (32 bits)
- STREAMINFO fields (16/24/20/36/128-bit values)
- Frame header sync, blocking strategy, block size, sample rate, channel, bit depth
- All subframe headers and data
- CRC-8 and CRC-16 values
- Seekpoint sample numbers and offsets
- Cuesheet track offsets and index offsets

**Exception: Vorbis Comments**
- Field lengths are stored little-endian within the comment block
- This is intentional for compatibility with Vorbis format

### 23.2 Integer Size Requirements

| Operation | Minimum Bits | Notes |
|-----------|-------------|-------|
| Sample storage (32-bit audio) | 32 bits | Signed two's complement |
| Fixed predictor computation | +4 bits | For order 4 |
| LPC dot product (24-bit, 12th-order, 15-bit coeffs) | 44 bits | Use 64-bit intermediate |
| Rice residual storage | 32 bits | Spec requires this |
| Accumulator for MD5 | 64 bits | MD5 algorithm uses 128-bit state |
| Frame number (fixed block) | 31 bits max | Subset limit |
| Sample number (variable block) | 36 bits | Total samples field |

### 23.3 Memory Requirements for Decoding

**Per-frame buffer:**
```
Stereo 16-bit, 4096 samples/block:
- Left channel: 4096 * 4 bytes = 16 KB
- Right channel: 4096 * 4 bytes = 16 KB
- Residual buffer: 4096 * 4 bytes = 16 KB
- Coefficient buffer: 32 * 2 bytes = 64 bytes
- Total: ~48 KB per frame
```

**8-channel 24-bit, 16384 samples/block:**
```
- 8 channels: 8 * 16384 * 4 bytes = 524 KB
- Residual: 8 * 16384 * 4 bytes = 524 KB
- Total: ~1 MB per frame
```

---

## 24. METADATA EDITTING WITHOUT RE-ENCODING

### 24.1 PADDING Strategy

PADDING metadata blocks reserve space for future metadata additions without rewriting the entire file:

**Initial encoding with padding:**
```bash
ffmpeg -i input.wav -c:a flac -padding_size 8192 output.flac
```

This creates an 8 KB PADDING block at the end of metadata. When editing:
1. Replace part of PADDING with new metadata block
2. Reduce PADDING size accordingly
3. No audio data needs to be rewritten

**Without padding:** Adding metadata requires:
1. Reading entire audio data into memory
2. Rewriting entire file with new metadata
3. For large files, this can be slow and require temporary disk space

### 24.2 Safe Metadata Editing

**Always work on a copy:**
```bash
cp original.flac working.flac
# Edit working.flac with metadata tools
ffmpeg -i working.flac -c:a copy output.flac
```

**Verify audio integrity after editing:**
```bash
# Compare MD5 checksums
ffprobe -v quiet -show_entries format=format_md5 original.flac
ffprobe -v quiet -show_entries format=format_md5 edited.flac
# Should be identical if only metadata was changed
```

**Safe edit operations:**
- Adding/replacing Vorbis Comment fields
- Embedding new PICTURE blocks (using existing padding or replacing PADDING)
- Removing PICTURE blocks
- Adding APPLICATION blocks

**Unsafe operations (require re-encoding):**
- Changing audio sample rate
- Changing bit depth
- Changing channel count
- Any lossy conversion

### 24.3 COVER ART Best Practices

**Picture block ordering:**
- Most players display the first PICTURE block of type 3 (Front Cover)
- Additional covers can be included (Back Cover, etc.)
- File size increases with cover art (typically 50-500 KB per image)

**Recommended cover art format:**
- JPEG for photographs: smaller files, wide compatibility
- PNG for graphics with transparency: lossless, larger files
- Resolution: 500x500 to 1000x1000 is sufficient for display
- Excessive resolution (4K album art) increases file size without visual benefit

**Preserving cover art during conversion:**
```bash
# Extract cover from source FLAC
ffmpeg -i source.flac -c:v copy cover.jpg

# Embed in converted file
ffmpeg -i converted.flac -i cover.jpg -c:a copy -c:v copy output.flac
```

---

## 25. PERFORMANCE TUNING

### 25.1 Encoder Optimization Guide

**For fastest encoding (Level 0):**
- Use `-compression_level 0`
- Fixed predictors only (no LPC): `-lpc_type none`
- Small block size: `-frame_size 1152`
- Independent stereo: `-ch_mode indep`
- Expected: 100-150x real-time at 44.1kHz stereo

**For best compression (Level 8):**
- Use `-compression_level 8`
- Enable LPC: `-lpc_type levinson` or `-lpc_type cholesky`
- Increase passes: `-lpc_passes 4` or higher
- Larger block size: `-frame_size 8192`
- Auto stereo mode: `-ch_mode auto`
- Enable multi-dimensional quantization: `-multi_dim_quant 1`
- Expected: 8-15x real-time at 44.1kHz stereo

**For archival (balanced):**
- Use `-compression_level 5` (default)
- Enable LPC: `-lpc_type levinson`
- Moderate passes: `-lpc_passes 2`
- Medium block size: `-frame_size 4096`
- Expected: 30-50x real-time at 44.1kHz stereo

### 25.2 Block Size Selection Guide

| Scenario | Recommended Block Size | Rationale |
|---------|----------------------|-----------|
| Streaming/Internet | 1152-4096 | Balance of compression and random access |
| Archival/Storage | 4096-8192 | Better compression from larger blocks |
| Low-latency monitoring | 192-576 | Minimal delay |
| High sample rate (96kHz+) | 4096-8192 | Fewer frames, better compression |
| Classical/long movements | 8192-16384 | Long-term redundancy exploitation |
| Speech/podcasts | 2048-4096 | Moderate complexity, good compression |

**Why larger blocks compress better:**
- More samples per block = more redundancy to exploit
- Better predictor fit for sustained tonal content
- Fewer frame headers overhead

**Why smaller blocks compress faster:**
- Less data to analyze per block
- Parallelizable across blocks (multi-threaded)
- Lower memory usage

### 25.3 Multi-Threaded Encoding (FLAC 1.5+)

FLAC 1.5 introduced multi-threaded encoding using thread pools:
- Block-level parallelism: multiple blocks encoded simultaneously
- Near-linear scaling with CPU core count
- Enable with: reference encoder uses all cores automatically

**FFmpeg multi-threaded:**
```bash
# FFmpeg uses multiple threads automatically for encoding
ffmpeg -i input.wav -c:a flac -compression_level 8 output.flac
# Use -threads to control (auto-detected by default)
```

**Memory scaling:**
- Single-threaded: ~50 MB for typical stereo file
- Multi-threaded: scales linearly with thread count
- 8 threads: ~400 MB peak memory

---

## 26. TESTING & VALIDATION

### 26.1 Conformance Test Suite

The FLAC project provides a conformance test suite at:
`https://github.com/ietf-wg-cellar/flac-test-files`

**Test categories:**
- Valid streams with all subframe types
- Streams with various bit depths (4-32 bits)
- Streams with various sample rates
- Streams with various channel counts (1-8)
- Streams with all metadata block types
- Edge cases and extreme configurations

**Running tests:**
```bash
# Clone test suite
git clone https://github.com/ietf-wg-cellar/flac-test-files.git

# Test decoder against reference
for file in flac-test-files/**/*.flac; do
    reference_decoder "$file" > /tmp/ref_output.pcm
    your_decoder "$file" > /tmp/test_output.pcm
    diff /tmp/ref_output.pcm /tmp/test_output.pcm || echo "FAIL: $file"
done
```

### 26.2 Round-Trip Verification

**Complete verification pipeline:**
```bash
# 1. Generate reference
ffmpeg -i original.wav -f md5 - 2>/dev/null > original.md5

# 2. Encode to FLAC
ffmpeg -i original.wav -c:a flac -compression_level 8 test.flac

# 3. Decode back
ffmpeg -i test.flac test.wav

# 4. Verify
ffmpeg -i test.wav -f md5 - 2>/dev/null > decoded.md5
diff original.md5 decoded.md5 && echo "PASS" || echo "FAIL"
```

### 26.3 Stress Testing

**Extreme configurations to test:**
- 4-bit audio (minimum bit depth)
- 32-bit audio (maximum bit depth)
- 1 Hz sample rate (minimum)
- 1 MHz sample rate (above spec limit to test rejection)
- 65535-sample block (maximum)
- 1-sample block (minimum valid for non-last blocks)
- All LPC orders 1-32
- Mixed block sizes (variable block size stream)
- 8 channels with all channel assignments
- Maximum size metadata blocks

---

## 27. APPENDIX: COMPLETE OPCODE REFERENCE

### 27.1 Metadata Block Opcodes

| Op | Name | Required | Contents | Size |
|----|------|---------|---------|------|
| 0 | STREAMINFO | Yes (first) | Stream parameters | 34 bytes |
| 1 | PADDING | No | Null bytes | Variable |
| 2 | APPLICATION | No | Third-party data | Variable (≥4 bytes) |
| 3 | SEEKTABLE | No | Seek points | Multiple of 18 |
| 4 | VORBIS_COMMENT | No | Tags | Variable |
| 5 | CUESHEET | No | CD-DA cues | Variable |
| 6 | PICTURE | No | Images | Variable |

### 27.2 Frame Header Bit Layout

```
| Sync (14) | BS (4) | SR (4) | CH (4) | BD (3) | Res (1) | Number (var) | Opt BS | Opt SR | CRC8 (8) |
```

### 27.3 Subframe Type Opcodes

| Op | Type | Has Coeffs | Has Residual |
|----|------|-----------|--------------|
| 0b000000 | CONSTANT | No | No |
| 0b000001 | VERBATIM | No | No |
| 0b001000 | FIXED order 0 | No | Yes |
| 0b001001 | FIXED order 1 | No | Yes |
| 0b001010 | FIXED order 2 | No | Yes |
| 0b001011 | FIXED order 3 | No | Yes |
| 0b001100 | FIXED order 4 | No | Yes |
| 0b100000 | LPC order 1 | Yes | Yes |
| ... | ... | ... | ... |
| 0b101011 | LPC order 32 | Yes | Yes |

### 27.4 CRC Polynomials

**CRC-8 (Frame Header):**
- Polynomial: x^8 + x^2 + x^1 + x^0 = 0x107
- Initialization: 0
- Reflected: No
- Example generator: 0x07

**CRC-16 (Frame Footer):**
- Polynomial: x^16 + x^15 + x^2 + x^0 = 0x18005
- Initialization: 0
- Reflected: No
- Example generator: 0x18005

---

## 28. GLOSSARY OF TERMS

| Term | Definition |
|------|-----------|
| Block | A fixed number of consecutive samples from all channels, processed together |
| Frame | The encoded representation of a block, including header and subframes |
| Subblock | All samples within a block for one channel |
| Subframe | The encoded representation of a subblock |
| Interchannel samples | Sample count that applies to all channels |
| Block size | Number of interchannel samples in a block |
| Bit depth | Number of bits per sample per channel |
| Blocking strategy | Whether block size is fixed or variable throughout stream |
| Rice coding | Variable-length coding optimized for small numbers |
| Golomb coding | Family of codes that Rice coding belongs to |
| Linear prediction | Predicting samples using weighted sum of past samples |
| Fixed predictor | Linear predictor with predefined coefficients |
| LPC | Linear Predictive Coding |
| Warm-up samples | Initial samples stored unencoded to start prediction chain |
| Residual | Difference between original and predicted sample |
| Zigzag encoding | Converting signed integers to unsigned for Rice coding |
| Stereo decorrelation | Transforming stereo channels to exploit inter-channel redundancy |
| Mid-side stereo | Encoding average and difference of stereo channels |
| Streamable subset | Restricted FLAC features ensuring hardware decoder compatibility |
