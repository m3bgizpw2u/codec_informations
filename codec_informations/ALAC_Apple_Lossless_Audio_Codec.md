# ALAC (Apple Lossless Audio Codec) — Deep Technical Reference

> **Category:** Lossless Audio
> **File Extensions:** .m4a, .alac, .caf
> **MIME Types:** audio/x-m4a, audio/mp4, audio/alac
> **Standardization Body:** Apple Inc. (originally proprietary, reverse-engineered)
> **Specification Document:** De facto specification via reverse engineering; FFmpeg source code is the most authoritative reference
> **Patent Status:** Patent-free (Apple released specification in 2011 without patent claims)
> **License:** Apache 2.0 (Apple's reference implementation)

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Origins and Evolution

Apple Lossless Audio Codec (ALAC) was developed by Apple Inc. as a proprietary lossless audio encoding format for use within the iTunes ecosystem and Apple Lossless encoded songs sold on the iTunes Store. The codec was introduced in 2004 alongside the iTunes 4.5 release, representing Apple's answer to the growing demand for lossless digital audio distribution.

The codec was originally known as "Apple Lossless" or "ALAC" and shipped as an integral part of QuickTime and iTunes. For many years it remained a closed, proprietary format — Apple never published a formal specification document. However, because it was embedded in an open operating system (macOS) and shipped with publicly available development tools (Xcode/Developer SDK), the format was fully reverse-engineered by the open-source community. The reference implementation was first reconstructed by the FFmpeg project, followed by other projects such as FAAD2 (decoder only), and later the reference decoder published by Apple.

### 1.2 Patent Disclosure (2011)

On October 29, 2011, Apple took the significant step of publishing the full source code for both the encoder and decoder under the Apache 2.0 license. This release included complete header files, implementation files, and a detailed technical description of the codec parameters. The publication effectively placed ALAC in the public domain for anyone to use without licensing fees or patent concerns.

The published source code is available in the `apple-alac` repository (sometimes referred to as `ALACunity` or the Apple Lossless Audio Codec project on `opensource.apple.com`). This source serves as the definitive specification for the codec.

### 1.3 Current Status

ALAC remains in active use as the lossless audio format of choice within the Apple ecosystem. It is supported natively in:
- macOS / iTunes (native playback and encoding)
- iOS / iPadOS (native playback)
- Apple TV
- QuickTime framework
- FFmpeg (via `libavcodec`)
- foobar2000 (via `foo_dumb` component)
- VLC media player
- JRiver Media Center
- dBpoweramp

With the introduction of Apple Music Lossless (2021), Apple expanded beyond ALAC to offer Dolby Atmos and Lossless variants using the same `.m4a` container but with different audio codecs. ALAC continues to be the standard for CD-quality lossless audio within the Apple ecosystem.

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Category and Classification

ALAC is a **lossless, block-based, predictive audio codec**. It uses linear prediction to model the audio signal, computes the prediction error (residual), and then entropy-encodes that residual using a variant of Golomb-Rice coding. The key architectural decisions are:

- **Predictive coding**: An adaptive finite impulse response (FIR) filter predicts each audio sample from previous samples.
- **Lossless**: The original PCM signal can be perfectly reconstructed from the encoded bitstream.
- **Block-based**: Audio is processed in fixed-length blocks (frames), typically 4096 samples per channel.
- **Entropy coding**: Residuals are encoded using a parameterized Golomb-Rice code — the Rice parameter (k) adapts per block.
- **Mid-side stereo**: Stereo channels are transformed to mid-side representation before encoding, improving compression on correlated stereo content.
- **No external dependencies**: The decoder is self-contained; it requires no lookup tables or external state beyond the per-frame parameters.

### 2.2 Compression Philosophy

ALAC was designed with **simplicity and decoder efficiency** as primary goals, rather than maximum compression ratio. This is a deliberate design choice: Apple wanted a codec that could be decoded in real time on relatively modest hardware (the original iPod and iPhone generations had constrained CPU resources). The encoder is allowed to be more complex (it searches for optimal parameters), but the decoder is intentionally simple.

As a result, ALAC compression ratios are typically **10-15% worse** than FLAC on the same material, and significantly worse than APE or WavPack in their highest compression modes. For a CD-quality 44.1kHz/16-bit stereo source, ALAC typically achieves compression ratios of 55-65%, compared to 60-70% for FLAC at default settings.

### 2.3 Bit Depth and Sample Rate Support

ALAC supports the following bit depths and sample rates:

| Bit Depth | Sample Rates | Notes |
|-----------|-------------|-------|
| 16-bit | 8–384 kHz | Standard CD quality |
| 20-bit | 8–384 kHz | Enhanced precision |
| 24-bit | 8–384 kHz | Studio quality |
| 32-bit | 8–384 kHz | Float/integer support, rarely used |

The maximum supported sample rate is 384,000 Hz (384 kHz), which covers DXD (352.8 kHz) and DSD rates. Note that FFmpeg's ALAC encoder defaults to 16-bit unless explicitly configured otherwise.

### 2.4 Endianness and PCM Representation

All multi-byte integers in the ALAC bitstream are stored in **big-endian (network byte order)** format, consistent with QuickTime/ISO base media conventions. The encoder accepts PCM in signed form; unsigned PCM (e.g., standard WAV 16-bit PCM) must be converted to signed representation before encoding.

### 2.5 Frame-Based Processing Model

ALAC processes audio in discrete, self-contained frames. Each frame encodes a fixed number of audio samples (determined by the `frameLength` parameter in the magic cookie). This design choice has important implications:

**Advantages of frame-based processing**:
1. **Memory efficiency**: The decoder only needs to buffer one frame at a time.
2. **Random access**: Frame boundaries provide natural seek points (though no seek index is built into the codec itself).
3. **Error containment**: Bitstream errors are isolated to the frame in which they occur.
4. **Parallelism**: Multiple frames can theoretically be processed in parallel (though the adaptive predictor requires sequential processing within a frame).

**Frame lifecycle**:
```
[Input PCM: frameLength × numChannels samples]
        ↓
[Mid-Side Transform (if stereo)]
        ↓
[Shift Optimization (wasted bits detection)]
        ↓
[Adaptive FIR Prediction]
        ↓
[Residual Computation: r[n] = x[n] − x̂[n]]
        ↓
[k-Parameter Optimization (encoder only)]
        ↓
[Golomb-Rice Entropy Encoding]
        ↓
[Frame Header: shift + nbytes]
        ↓
[Output: nbytes bytes of encoded frame data]
```

### 2.6 State Management

The ALAC decoder maintains the following state variables that persist across frames (but not across seeks):

| State Variable | Type | Purpose |
|---------------|------|---------|
| `predictor_coeffs[32]` | int32[32] | FIR filter coefficients (LMS-updated) |
| `history_buffer` | int32[] per channel | Circular buffer of recent samples |
| `k` | uint8 | Current Rice parameter |
| `history` | uint32 | Accumulated absolute residual sum |
| `block_size` | uint32 | Frame length (from magic cookie) |
| `num_channels` | uint8 | Number of active channels |

The initial values of these state variables are set from the magic cookie parameters (`initialHistory`, `maxK`, `historyMultipler`) at the start of decoding. After each frame, the state is carried forward to the next frame without reset.

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 Stream Structure Overview

ALAC itself does not define a standalone file format. Instead, it is always embedded within a **container format**. The primary containers are:

- **QuickTime / ISO Base Media (.m4a, .mp4)**: The most common container for ALAC. The audio track uses the `alac` codec four-character code (FOURCC: `'a' 'l' 'a' 'c'`).
- **Core Audio Format (.caf)**: Apple's own container, supports ALAC natively.
- **QuickTime File Format (.mov)**: ALAC audio tracks in video files.

The audio elementary stream (the encoded ALAC bitstream itself) consists of:

```
+------------------------------------------+
|  per-frame header (3-8 bytes)            |  Parameters that control this frame's decode
|  + residual data (variable)              |  The compressed audio samples
+------------------------------------------+
|  [repeat for each frame]                 |
+------------------------------------------+
```

There is **no global stream header** in ALAC; all codec parameters are either fixed (baked into the decoder) or transmitted per-frame in the frame header.

### 3.2 The ALAC Magic Number

Every ALAC elementary stream begins with a **32-byte magic cookie** (sometimes called the "ALAC magic number" or "ALAC specific configuration"). This magic cookie is stored in the QuickTime `alac` atom (`kuki` atom) within the media header. It must be passed to the ALAC decoder before any frames can be decoded.

The magic cookie is **NOT optional** — it configures the decoder's fundamental behavior. A corrupted or missing magic cookie will cause decoding to fail or produce garbage output.

Complete 32-byte magic cookie layout:

```
 Byte:  0      [1]     [2]     [3]     [4]     [5]     [6]     [7]
       +-------+-------+-------+-------+-------+-------+-------+-------+
  0x00 | 'a'   | 'l'   | 'a'   | 'c'   |  0x00 |  0x00 |  0x00 |  0x00 |
       +-------+-------+-------+-------+-------+-------+-------+-------+

 Byte:  [8]    [9]     [10]    [11]    [12]    [13]    [14]    [15]
       +-------+-------+-------+-------+-------+-------+-------+-------+
  0x08 |  frameLength (24 bits, big-endian)                          |  0x00
       +-------+-------+-------+-------+-------+-------+-------+-------+
  0x10 |  0x00 | compatible[0]| compatible[1]| compatible[2]| compatible[3]|
       +-------+-------+-------+-------+-------+-------+-------+-------+
  0x18 | compatible[4]| compatible[5]| compatible[6]| maxK    |  0x00  |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  0x20 | historyMultipler (8 bits)                                    |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  0x28 | initialHistory (8 bits)                                      |
       +-------+-------+-------+-------+-------+-------+-------+-------+
  0x30 | kModifier (8 bits)                                           |
       +-------+-------+-------+-------+-------+-------+-------+-------+
```

### 3.3 Magic Cookie Field Descriptions

| Field | Byte Offset | Bit Width | Type | Default | Description |
|-------|------------|-----------|------|---------|-------------|
| `format[4]` | 0 | 32 | FOURCC | `'alac'` | Format identifier — always `'alac'` |
| `flags` | 4 | 32 | uint32 | `0x00000000` | Reserved flags (always 0) |
| `frameLength` | 8 | 24 | uint24 | `0x0400` (1024) | Frame size in samples per channel. Typical: 4096. Must be a multiple of 8. |
| `compatible[7]` | 11 | 56 | uint8[7] | all `0` | Reserved compatibility bytes — all zeros in standard ALAC |
| `maxK` | 18 | 8 | uint8 | `0x14` (20) | Maximum Rice parameter k. Controls the range of the entropy coder. Valid range: 0-31. |
| `historyMultipler` | 20 | 8 | uint8 | `0x40` (64) | History multiplier (x100). Controls the rate of the adaptive Rice parameter. Value = multiplier × 100. Range: 1-255. |
| `initialHistory` | 21 | 8 | uint8 | `0x0E` (14) | Initial history value. Controls the starting value of the adaptive entropy parameter. Range: 0-255. |
| `kModifier` | 22 | 8 | uint8 | `0x08` (8) | k modifier. Added to the Rice parameter on each search step. Range: 0-255. |
| `searchNegCeil` | 23 | 8 | uint8 | `0x08` (8) | Negative search ceiling. Controls the lower bound of the k search range. Range: 0-255. |
| `searchPosCeil` | 24 | 8 | uint8 | `0x10` (16) | Positive search ceiling. Controls the upper bound of the k search range. Range: 0-255. |

The first 8 bytes are the `'alac'` magic word and flags, followed by the 24-byte ALAC-specific configuration block.

**Example magic cookie** (hex dump, 32 bytes total):

```
61 6c 61 63  00 00 00 00   alac........
00 10 00 00  00 00 00 00   ........    frameLength = 0x00001000 = 4096
00 00 00 00  00 00 00 14   ........    maxK = 0x14 = 20
40 0e 08 08  10            @.......    historyMultipler=64, initialHistory=14, kModifier=8, searchNegCeil=8, searchPosCeil=16
```

### 3.4 Per-Frame Header

Each ALAC frame begins with a **3-byte frame header** (sometimes 4 bytes depending on channel configuration). This header contains parameters that **override or supplement** the magic cookie settings for this specific frame.

The frame header consists of three unsigned 8-bit integers, all in big-endian order:

- **`shift`** (byte 0): Shift amount applied to input samples before prediction. This compensates for wasted bits in the input PCM. For 16-bit audio, shift=0 typically. For 24-bit audio with lower 8 bits always zero, shift might be 8.
- **`nbytes`** (bytes 1-2, combined as 16-bit big-endian): The total size of the compressed frame data in bytes, including the 3-byte header itself. So the actual residual data size is `nbytes - 3`.

For **interleaved / composite frames** (stereo where both channels share one frame), an additional 8-bit field follows the 3-byte header:

```
byte 0: shift
byte 1: nbytes_high
byte 2: nbytes_low
byte 3: numChannels (for composite frames, this byte is present)
```

For non-interleaved frames (each channel has its own frame), the header is exactly 3 bytes.

### 3.5 Frame Header Byte Layout Diagram

Complete byte-level diagram of the ALAC frame header:

```
+------------------------------------------------------------------+
|  Byte 0: shift (8 bits)                                          |
|  +------------------------------------------------------------+ |
|  | b7   | b6   | b5   | b4   | b3   | b2   | b1   | b0     | |
|  +------------------------------------------------------------+ |
|  |  0   |  0   |  0   |  0   |  0   |  0   |  0   |  0     | |  (example: shift = 0)
|  +------------------------------------------------------------+ |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
|  Byte 1: nbytes_high (8 bits)                                    |
|  Contains the upper 8 bits of the 16-bit frame byte count.       |
|  Combined with Byte 2 to form: nbytes = (byte1 << 8) | byte2    |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
|  Byte 2: nbytes_low (8 bits)                                    |
|  Contains the lower 8 bits of the 16-bit frame byte count.       |
+------------------------------------------------------------------+

+------------------------------------------------------------------+
|  Byte 3: numChannels (8 bits) — ONLY present for composite frames |
|  For stereo composite frames: numChannels = 2 (0x02)             |
|  For mono frames: this byte is absent (header is only 3 bytes)  |
+------------------------------------------------------------------+
```

**Example frame header** (hex):

```
00              — shift = 0 (no wasted bits)
10 27           — nbytes = 0x1027 = 4143 bytes total
                  residual data = 4143 - 3 = 4140 bytes
[02]            — numChannels = 2 (composite stereo frame)
                  (this byte would be absent for mono/non-interleaved stereo)
```

### 3.6 Frame Size Calculation

The `nbytes` field specifies the total frame size including the frame header. This is critical for the decoder to know where one frame ends and the next begins:

```
frame_data_start = current_bitstream_position (byte-aligned)
frame_data_end = frame_data_start + nbytes
next_frame_start = frame_data_end (byte-aligned)
```

Because `nbytes` includes the 3-byte header, the actual entropy-coded residual data occupies `nbytes - 3` bytes. The decoder reads exactly `nbytes` bytes from the stream, regardless of how many bits of those bytes were actually consumed during decoding. Any leftover bits at the end of the `nbytes` region are padding and must be discarded.

### 3.7 Residual Data Layout

Within the `nbytes - 3` bytes of residual data, the bitstream is organized as follows:

```
For each channel (in order):
  For each sample in the frame (0 to frameLength - 1):
    Decode one signed residual using Golomb-Rice with current k
    Update adaptive k
    Apply predictor
    Store reconstructed sample

For stereo composite frames (channels interleaved):
  For each sample index (0 to frameLength - 1):
    For each channel (0 to numChannels - 1):
      Decode one signed residual
      Update per-channel k and predictor
      Store reconstructed sample
```

The bitstream does not contain any explicit markers between channels or samples. The decoder implicitly knows how many residuals to decode based on `frameLength` and `numChannels` from the magic cookie and frame header.

---

## 4. COMPRESSION ALGORITHM — ADAPTIVE FIR PREDICTOR

### 4.1 Overview

The core compression algorithm in ALAC consists of two stages:

1. **Linear prediction**: An adaptive FIR filter computes a predicted value for each input sample based on previous samples. The prediction error (residual) is the difference between the actual and predicted value.
2. **Entropy coding**: The residuals are encoded using a parameterized Golomb-Rice code, where the Rice parameter (k) adapts based on the residual statistics.

This is conceptually similar to FLAC's fixed and LPC predictors, but ALAC uses a **single adaptive predictor** that adjusts its coefficients per frame, rather than a fixed-coefficient predictor.

### 4.2 The Adaptive FIR Filter

The ALAC predictor is a **fully adaptive FIR filter** with a history buffer. Unlike FLAC's fixed-order predictors, the ALAC predictor's coefficients are recomputed adaptively based on the recent residual statistics. The predictor uses a **signed 32-bit integer arithmetic** throughout.

The predictor order is not explicitly transmitted — it is implicitly determined by the codec parameters and the sample count. The codec maintains a circular history buffer of previously decoded residual values.

**Predictor computation** (at the encoder, where samples are known):

For each sample `x[n]`, the predicted value `x̂[n]` is computed as a weighted sum of the previous `m` residual values:

```
x̂[n] = −Σ_{i=1}^{m} a_i · x[n − i]
```

Where `a_i` are the adaptive filter coefficients. The negative sign reflects that the predictor works on residuals in an autoregressive model.

**At the decoder**, the same computation is performed. Because both encoder and decoder use the same adaptive algorithm (initialized identically with the same parameters from the magic cookie), they converge to the same coefficients and therefore produce identical residuals from the same encoded data.

### 4.3 Coefficient Adaptation

The filter coefficients `a_i` are adapted after each sample using a **normalized LMS (Least Mean Squares)** style update. The adaptation rate is controlled by the `historyMultipler` parameter from the magic cookie.

The adaptation mechanism works as follows:

```
For each sample n:
  1. Compute prediction x̂[n] using current coefficients
  2. Compute residual r[n] = x[n] − x̂[n]  (at encoder)
  3. Update coefficients:
     a_i(new) = a_i(old) + μ · r[n] · x[n − i]
```

Where `μ` is the adaptation step size, derived from `historyMultipler`. The `historyMultipler` parameter (default 64) scales the adaptation rate: higher values produce faster adaptation but may cause instability; lower values produce slower adaptation but may not track non-stationary signals well.

### 4.4 Default Parameters and Ranges

The magic cookie encodes parameters that control the predictor and entropy coder:

| Parameter | Default | Range | Effect |
|-----------|---------|-------|--------|
| `frameLength` | 4096 | 0–65535 (multiple of 8) | Number of samples per frame |
| `maxK` | 20 | 0–31 | Maximum Rice parameter k |
| `historyMultipler` | 64 | 1–255 | Predictor adaptation rate |
| `initialHistory` | 14 | 0–255 | Initial entropy state |
| `kModifier` | 8 | 0–255 | k search step size |
| `searchNegCeil` | 8 | 0–255 | Lower bound offset for k search |
| `searchPosCeil` | 16 | 0–255 | Upper bound offset for k search |

The `initialHistory` value seeds the entropy coder's internal state before processing the frame. The `searchNegCeil` and `searchPosCeil` define the range around the estimated optimal k within which the encoder performs a brute-force search for the best k value. This search optimizes compression for the specific frame's residual statistics.

### 4.5 LMS Algorithm Mathematics

The Least Mean Squares (LMS) algorithm used in ALAC is a simplified variant of the classic Widrow-Hoff LMS algorithm. The key equations are:

**Prediction Error**:
```
e[n] = d[n] − x̂[n]
```
Where `d[n]` is the desired (actual) sample value and `x̂[n]` is the predicted value.

**Filter Coefficient Update**:
```
a_i[n+1] = a_i[n] + 2μ · e[n] · x[n−i]
```

The factor of 2 in the standard LMS update is folded into the `historyMultipler` scaling. In the actual ALAC implementation:

```
μ_eff = (historyMultipler * error) >> 9
for i in range(32):
    if sample_idx > i:
        predictor_coeffs[i] += μ_eff * history_buffer[(sample_idx - 1 - i) % frame_size]
```

The `>> 9` shift (division by 512) serves as the normalization factor that prevents coefficient overflow and ensures stable convergence. This value was chosen empirically by Apple.

### 4.6 History Buffer Management

The predictor maintains a **circular (ring) buffer** of the most recent samples for each channel. The buffer size equals `frameLength`, which means the predictor can reference any sample from the previous `frameLength` samples:

```
buffer_index = sample_index % frame_length
history_buffer[buffer_index] = current_sample
```

This circular buffer design serves two purposes:
1. It limits memory usage to a fixed, predictable amount (frameLength × numChannels × 4 bytes).
2. It naturally limits the predictor order — samples older than `frameLength` are no longer directly accessible (they have been overwritten).

The 32-coefficient limit means the predictor can use at most 32 samples in the past for any single prediction, even though the history buffer can hold `frameLength` samples. This is a deliberate complexity/performance tradeoff: 32 coefficients provide sufficient modeling capability for most audio signals while keeping the multiply-accumulate operation fast.

### 4.7 Coefficient Initialization and Convergence

At the start of decoding (or after a seek), the predictor coefficients are initialized to zero. This means the first samples in a stream (or after a seek) are encoded with a trivial predictor (prediction = 0, so the residual equals the sample value). As more samples are processed, the coefficients converge toward values that model the signal's statistical properties.

The convergence rate depends on the `historyMultipler` parameter:
- **High values (e.g., 128-255)**: Fast convergence, but potential overshoot and instability on signals with sudden transients.
- **Low values (e.g., 1-16)**: Slow convergence, may not adapt quickly enough to changing signal characteristics.
- **Default (64)**: Balanced — converges within 1000-2000 samples for typical music signals.

For typical music with stationary characteristics (consistent spectral content over hundreds of milliseconds), the predictor converges within the first few frames. For signals with rapid changes (drum hits, transients), the predictor re-converges after each change.

### 4.8 Why Adaptive Coefficients?

The choice of an adaptive predictor (rather than a fixed-coefficient predictor like FLAC's fixed-order predictors) has important implications:

**Benefits**:
1. **Adaptation to signal characteristics**: Different instruments and frequency content produce different optimal predictors. An adaptive predictor automatically adjusts.
2. **Better compression on diverse material**: A single predictor configuration cannot be optimal for both a sustained violin note and a percussive drum hit. The adaptive mechanism bridges this gap.
3. **No need to transmit coefficients**: Because both encoder and decoder run the same adaptation algorithm, no predictor coefficients need to be stored in the bitstream. This saves bits.

**Tradeoffs**:
1. **Encoding complexity**: The encoder must simulate the predictor forward through the entire frame to compute residuals before entropy encoding.
2. **State dependence**: The predictor's state at the start of a frame must match between encoder and decoder. This is guaranteed by the LMS algorithm's determinism, but it means seeking within a stream requires either resetting the state and re-decoding from the nearest keyframe, or storing predictor state at seek points.
3. **Numerical precision**: Using 32-bit integer arithmetic with LMS can cause coefficient drift over very long streams. The division by 512 (`>> 9`) in the update formula provides stability at the cost of some adaptation precision.

---

## 5. GOLOMB-RICE ENTROPY CODING

### 5.1 Encoding Process

The ALAC entropy coder uses a **Golomb-Rice code** parameterized by a Rice parameter `k`. This is a subset of Golomb coding where the divisor is always a power of 2 (`2^k`), making encoding and decoding extremely fast — division and modulo reduce to bit shifts.

For a given residual value `r` and parameter `k`:

1. Compute the **unary prefix**: `q = |r| >> k` (quotient). Encode `q` as `q` zero bits followed by a one bit.
2. Compute the **binary suffix**: `b = |r| & ((1 << k) − 1)` (the lower k bits). Output `b` as a k-bit binary number.
3. If `r` is negative, XOR the suffix with `(1 << k) − 1` (bit inversion for signed residuals).

**Output format**: `[unary(q)][k-bit suffix]`

**Example**: k=3, r=5:
- q = |5| >> 3 = 0, prefix = "1"
- b = |5| & 7 = 5, suffix = "101"
- Output: "1101" (4 bits)

**Example**: k=3, r=−5:
- q = 0, prefix = "1"
- b = 5 XOR 7 = 2, suffix = "010"
- Output: "1010" (4 bits)

### 5.2 Adaptive k Parameter

The optimal `k` varies with the residual distribution. ALAC uses an **adaptive** scheme where `k` is updated after processing each residual:

```
expectedAbsDiff += historyMultipler × (|r| − expectedAbsDiff) / 256
if expectedAbsDiff >= (1 << k):
    while expectedAbsDiff >= (1 << k):
        k += 1
else:
    while expectedAbsDiff < (1 << (k − 1)):
        k -= 1
```

The `historyMultipler` from the magic cookie scales the adaptation speed. Higher values make `k` respond more aggressively to changes in residual magnitude.

### 5.3 k Optimization Search

The encoder performs an **exhaustive search** to find the optimal initial `k` for each frame, within the range bounded by `searchNegCeil` and `searchPosCeil` around an estimated starting point. The encoder encodes the frame with each candidate `k` and selects the one that produces the fewest total bits.

The search range for initial k is:

```
k_min = max(0, estimated_k − searchNegCeil)
k_max = min(maxK, estimated_k + searchPosCeil)
```

The optimal `k` found during this search is not explicitly transmitted — instead, the encoder picks the `k` that minimizes encoded size, and the decoder independently computes `k` from the residual statistics using the same adaptive algorithm, converging to the same value.

### 5.4 Golomb-Rice Code Properties

Golomb-Rice codes have several important mathematical properties that make them ideal for audio residual encoding:

**Optimality**: For a memoryless source with a geometric distribution of residual magnitudes, Golomb-Rice coding is provably optimal. Audio prediction residuals closely approximate a Laplacian distribution (a special case of the geometric distribution), making Golomb-Rice nearly optimal for this use case.

**Code Length**: The average code length for a residual `r` with Rice parameter `k` is approximately:
```
L(r, k) ≈ 1 + |r|/2^k + log2(e)/2^k + k
```
The optimal `k` minimizes this expression given the actual distribution of `|r|`.

**Symbol Independence**: Each residual is encoded independently — no context from previous residuals affects the encoding of the current residual. This property means the decoder can recover from bit errors at the next residual symbol boundary (assuming it can resynchronize its bit position).

**Exponential Distribution Fit**: The residual magnitude distribution in audio coding typically follows an exponential (or Laplace) distribution:
```
P(|r| = n) ∝ e^(−λ·n)
```
The optimal `k` for this distribution satisfies:
```
2^k ≈ 1/λ
```
where `λ` is the rate parameter of the exponential distribution.

### 5.5 Bit-Level Representation

Understanding the bit-level representation is crucial for implementing an ALAC decoder:

**Bit ordering within bytes**: ALAC reads bits MSB-to-LSB within each byte (big-endian bit ordering). This is consistent with how JPEG, MPEG, and most video codecs work, but differs from FLAC which uses LSB-to-MSB within bytes.

**Bit reading pseudocode**:
```
bit_buffer: holds current byte (refreshed when bit_position crosses byte boundary)
bit_position: tracks position within the current byte (0 = MSB, 7 = LSB)

ReadBit():
    if bit_position == 0:
        bit_buffer = ReadNextByte()
    bit = (bit_buffer >> (7 - bit_position)) & 1
    bit_position = (bit_position + 1) % 8
    return bit

ReadBits(n):
    value = 0
    for i in range(n):
        value = (value << 1) | ReadBit()
    return value
```

**Byte alignment**: ALAC frames are byte-aligned at the start (the 3-byte frame header provides the alignment point), but individual samples within the frame are NOT byte-aligned — they span arbitrary bit boundaries within bytes. This bit-level interleaving is what makes the codec efficient for small residuals (which may only need 1-3 bits each).

### 5.6 Unary Coding

The unary prefix portion of Golomb-Rice coding is conceptually simple: for a quotient `q`, output `q` zero bits followed by a single one bit. This is equivalent to outputting the binary representation of `q+1` with the most significant bit stripped.

**Unary examples** (the delimiter "|" marks the transition from prefix to suffix):

| q | Unary prefix | Output |
|---|-------------|--------|
| 0 | `1` | `1|` |
| 1 | `01` | `01|` |
| 2 | `001` | `001|` |
| 3 | `0001` | `0001|` |
| 4 | `00001` | `00001|` |
| n | `0` repeated n times, then `1` | `0...01|` |

The decoder's `while ReadBit() == 0: q++` loop reads consecutive zero bits until it encounters a one, counting the zeros as the quotient `q`.

**Efficiency note**: Unary coding is very efficient when `q` is small (which is the common case for residuals near zero), but very inefficient when `q` is large. For audio residuals, `q` is typically 0 or 1, making unary coding highly effective.

### 5.7 Signed Residual Mapping

The Golomb-Rice code as described so far only encodes non-negative integers. ALAC must encode **signed** residuals (which can be positive or negative). The mapping from signed `r` to unsigned `magnitude` is:

1. Take the absolute value: `magnitude = |r|`
2. Encode `magnitude` using the unary+binary scheme
3. Use an extra sign bit: `0` for positive, `1` for negative

This means the total encoded size for signed residual `r` is:
- **Prefix**: unary编码(`|r| >> k`) — proportional to the shifted magnitude
- **Suffix**: `k` bits — the lower-order bits of the magnitude
- **Sign**: 1 bit — positive or negative

**Combined encoding**: In some implementations, the sign bit is merged with the suffix by XORing the suffix with `(1 << k) - 1` for negative values. This is equivalent to a sign-magnitude representation with the suffix bits inverted for negative numbers.

### 5.8 Encoder-Only Optimization: The k Search

The decoder has no choice in its `k` value — it must follow the adaptive update formula exactly. The encoder, however, has the freedom to choose an initial `k` for each frame. This is the primary source of ALAC's encoding complexity.

**Search algorithm**:
```
best_k = 0
best_size = infinity

for candidate_k in range(max(0, estimated_k - searchNegCeil), min(maxK, estimated_k + searchPosCeil) + 1):
    # Simulate encoding with this k
    size = 0
    reset_entropy_state()

    for each sample in frame:
        # Compute residual (requires simulating the predictor)
        residual = compute_residual(sample)

        # Encode with candidate_k and count bits
        size += encode_bits_needed(residual, candidate_k)

        # Update k adaptively (same as decoder)
        candidate_k = adaptive_update(candidate_k, residual)

    if size < best_size:
        best_size = size
        best_k = candidate_k

# Encode the frame using best_k
```

The `searchNegCeil` and `searchPosCeil` parameters limit the search range. The `estimated_k` is computed from the current entropy state before the frame begins. This search ensures the encoder picks the initial `k` that produces the smallest encoded frame, but the decoder arrives at the same `k` through the adaptive mechanism alone.

**Performance note**: The search is `O(frame_length × search_range)` complexity. For `frame_length=4096` and `search_range=25` (searchNegCeil=8, searchPosCeil=16), this is roughly 100,000 iterations per frame. At 44.1kHz, encoding at real-time requires processing ~10 frames per second, making this computationally intensive but feasible.

### 5.9 Bitrate and Compression Ratio

The compressed bitrate for ALAC depends on the entropy of the residual signal:

```
bitrate = (sample_rate × num_channels × bit_depth) × compression_ratio
```

Where `compression_ratio` is the ratio of compressed bits to uncompressed bits.

**Example**: 44.1kHz, 16-bit stereo:
- Raw bitrate: 44100 × 2 × 16 = 1,411,200 bits/second = 176,400 bytes/second
- ALAC typical: 60% compression ratio → 846,720 bits/second = 105,840 bytes/second
- ALAC efficient: 50% compression ratio → 705,600 bits/second = 88,200 bytes/second

The variable-length nature of Golomb-Rice coding means the instantaneous bitrate fluctuates frame by frame. High-entropy frames (transients, noise) produce larger frames; low-entropy frames (sustained tones, silence) produce smaller frames.

---

## 6. MID-SIDE STEREO ENCODING

### 6.1 Channel Decorrelation

ALAC supports **mid-side (M/S) stereo encoding** as an optional transformation applied before prediction. This transformation decorrelates the left and right stereo channels, typically producing smaller residuals for material with strong stereo correlation (most music has highly correlated L/R channels).

The mid-side transformation is defined as:

```
mid  = (L + R) / 2        (integer division, rounded toward zero)
side = L − R
```

The decoder reconstructs:

```
L = mid + side
R = mid − side
```

Note: When using integer arithmetic with odd L/R values, the division by 2 for computing mid requires rounding. The standard approach rounds toward zero (truncation), which is equivalent to `mid = (L + R) >> 1` for signed integers.

### 6.2 Integer Arithmetic Considerations

When implementing the M/S transform in integer arithmetic, several edge cases must be handled:

**Signed division by 2**: For signed integers, right-shift `(L + R) >> 1` is NOT equivalent to division by 2 when negative values are involved, due to round-toward-negative-infinity behavior of right-shift versus round-toward-zero for C's integer division. The standard ALAC implementation uses:

```c
// Compute mid with correct rounding
int32_t sum = L + R;
// For odd sum, add 1 if positive, subtract 1 if negative (round toward zero)
int32_t mid = (sum >= 0) ? (sum >> 1) : -((-sum) >> 1);
// Equivalent to: (sum >= 0) ? (sum / 2) : -((-sum) / 2)
```

This ensures that `mid = round_toward_zero((L + R) / 2)`.

**Overflow in side calculation**: When computing `side = L - R`, the result can overflow if L and R are at opposite extremes. For 16-bit audio, the range is -65536 to +65535, so `side` can be in range -131071 to +131071, requiring 18 bits to represent without overflow. The ALAC predictor operates on 32-bit integers, so this overflow is not an issue in practice.

**Range of mid**: `mid` is always in the range [-32768, +32767] for 16-bit audio, since it's the average of two values each in [-32768, +32767].

### 6.3 Implementation Details

The mid-side transformation in ALAC is **frame-level**: the decision to use M/S stereo is made once per frame and applies to all samples in that frame. The transformation is applied before the adaptive predictor runs.

In the ALAC elementary stream, stereo content can be encoded in two ways:

1. **Non-interleaved (separate frames)**: Left and right channels are encoded as independent mono frames, each with its own frame header and residual data. This is indicated by the container format (e.g., two mono samples in the mdat atom).
2. **Interleaved (composite frame)**: A single frame contains both channels interleaved. The frame header includes a `numChannels` byte (value=2) and the residual data is interleaved mid/side values.

The QuickTime container signals the channel configuration separately. The ALAC bitstream itself does not carry a channel mode flag — this information comes from the container's `sample description` atom (`stsd`).

### 6.4 Compression Benefit

For typical music material with strong L/R correlation (e.g., a centered vocal, kick drum, snare), M/S encoding typically reduces the side channel residual energy significantly, resulting in better compression. The mid channel captures the sum (mono content), while the side channel captures the difference (stereo spatial information).

For highly uncorrelated stereo content (e.g., a binaural recording, true stereo with completely different signals on L/R), M/S encoding may actually increase file size, but ALAC applies it uniformly because the typical use case benefits.

### 6.5 Energy Distribution Analysis

The compression benefit of M/S encoding can be understood by analyzing the energy distribution:

**Without M/S (L/R encoding)**:
```
Energy_L ≈ Energy_R ≈ σ²          (both channels have similar variance)
Residual energy: 2 × σ²
```

**With M/S encoding**:
```
Energy_mid ≈ 4 × σ²               (sum of correlated signals → high energy)
Energy_side ≈ ε² << σ²            (difference of correlated signals → low energy)
Residual energy: 4 × σ² + ε²
```

For highly correlated L/R (correlation coefficient ρ ≈ 1):
```
Energy_side ≈ σ² × (1 − ρ) ≈ 0
Compression improvement: ~50% on the side channel
```

This explains why M/S encoding provides substantial compression gains on typical music: most musical energy is centered (mono) and the L/R channels are highly correlated, resulting in near-zero side channel energy.

### 6.6 Inverse Transform and Reconstruction

The inverse M/S transform (applied at the decoder) is:

```c
void inverse_ms_transform(int32_t *mid, int32_t *side, int32_t *L, int32_t *R, int frame_length) {
    for (int i = 0; i < frame_length; i++) {
        int32_t m = mid[i];
        int32_t s = side[i];
        L[i] = m + s;   // = ((L+R)/2) + (L-R) = L
        R[i] = m - s;   // = ((L+R)/2) - (L-R) = R
    }
}
```

Note: The inverse transform does not require division by 2. Adding `mid + side` reconstructs `L` directly, and `mid - side` reconstructs `R`. The division by 2 only appears in the forward transform; the inverse uses only addition and subtraction.

---

## 7. QUICKTIME CONTAINER — ATOM LAYOUT

### 7.1 File Structure

An ALAC file (`.m4a`) is a QuickTime / ISO Base Media file. The top-level structure consists of a hierarchy of **atoms** (also called "boxes" in ISO BMFF terminology). Each atom has a 4-byte size field and a 4-byte type field, followed by the atom's payload.

```
TopLevel.m4a
├── ftyp (file type)
│   ├── major_brand: 'M4A '
│   ├── minor_version: 0
│   └── compatible_brands: ['M4A ', 'mp42', 'isom', ...]
├── moov (movie atom)
│   ├── mvhd (movie header)
│   ├── trak (track atom)
│   │   ├── tkhd (track header)
│   │   ├── mdia (media atom)
│   │   │   ├── mdhd (media header)
│   │   │   ├── hdlr (handler reference)
│   │   │   └── minf (media information)
│   │   │       ├── smhd (sound media header)
│   │   │       ├── dinf (data information)
│   │   │       │   └── dref (data reference)
│   │   │       └── stbl (sample table)
│   │   │           ├── stsd (sample description) ← contains 'alac' codec
│   │   │           ├── stts (time-to-sample)
│   │   │           ├── stsc (sample-to-chunk)
│   │   │           ├── stsz (sample sizes)
│   │   │           └── stco (chunk offset)
│   └── udta (user data, optional)
└── mdat (media data atom)
    ├── [ALAC frame 1] ← the actual encoded audio data
    ├── [ALAC frame 2]
    └── ...
```

### 7.2 The 'alac' Codec Atom (kuki)

The most critical atom for ALAC is the `'alac'` codec-specific atom embedded within the `'stsd'` (sample description) atom. This atom contains the 32-byte magic cookie that configures the ALAC decoder.

Structure of the `alac` codec atom:

```
Size (4 bytes, big-endian uint32) — total atom size including header
Type (4 bytes): 'kuki' (indicates a user-defined codec)

Payload (within 'kuki'):
  Format (4 bytes): 'alac'
  Reserved (4 bytes): 0x00000000
  ALAC Magic Cookie (32 bytes): as described in Section 3.2
```

In ISO base media / QuickTime terminology, the `'kuki'` atom is the container for a format-specific atom. The format-specific content begins with a format 4-byte code (`'alac'`), followed by reserved bytes and the codec configuration.

**Full `stsd` entry for ALAC** (from an actual file, hex representation):

```
00 00 00 5C           — size = 92 bytes
'stsd'                — sample description atom type
00 00 00 00           — version + flags (0)
00 00 00 01           — entry count = 1

  00 00 00 40         — size = 64 bytes (this entry)
  'alac'              — codec type (not 'kuki' at this level)
  00 00               — reserved
  00 01               — data-reference index
  00 00               — reserved
  00 00               — reserved
  00 00 00 00         — reserved
  00 00               — reserved
  00 00               — channel count (2)
  00 00               — sample size (16 bits)
  00 00               — compression ID
  00 00               — packet size (0 = variable)
  00 00 00 00         — sample rate (fixed-point 16.16)

  'alac'              — format ('alac')
  00 00 00 00         — reserved/flags
  00 00 10 00         — size of ALAC atom (4096)
  00 00 00 00         — compatible[0-3]
  00 00 00 14         — maxK = 20
  40 0E 08 08 10      — historyMultipler=64, initialHistory=14, kModifier=8, searchNegCeil=8, searchPosCeil=16
```

### 7.3 The 'alac' Atom (Full Codec Configuration)

In some `.m4a` files, the codec configuration appears directly in an `'alac'` atom rather than wrapped in a `'kuki'` container. The structure is:

```
Size (4 bytes)
Type: 'alac'
Payload: the same 32-byte magic cookie
```

Both representations are equivalent; the difference is how QuickTime-generators package the codec info. FFmpeg's muxer writes the codec info in a `'kuki'` container within `'alac'`.

### 7.4 Sound Media Header (smhd)

The `smhd` atom describes audio-specific properties:

```
Size (4 bytes)
Type: 'smhd'
Version (1 byte): 0
Flags (3 bytes): 0
Balance (2 bytes): signed int8 — stereo balance, 0=center
Reserved (2 bytes): 0
```

Balance values: 0 = centered, positive = pan left, negative = pan right. Most ALAC files have balance=0.

### 7.5 Media Header (mdhd)

```
Size (4 bytes)
Type: 'mdhd'
Version (1 byte): 0 or 1
Flags (3 bytes): 0
Creation time (4 or 8 bytes)
Modification time (4 or 8 bytes)
Time scale (4 bytes): 44100 or 48000 (samples per second)
Duration (4 or 8 bytes): total number of samples
Language (2 bytes): packed language code (e.g., 0x55C4 = 'und')
Reserved (2 bytes): 0
```

### 7.6 Sample Table (stbl)

The `stbl` atom is the heart of the sample mapping:

- **`stsd`**: Sample description — codec configuration and format details
- **`stts`**: Time-to-sample — maps decode time to sample index
- **`stsc`**: Sample-to-chunk — maps samples to chunks for efficient seeking
- **`stsz`**: Sample size — size of each compressed ALAC frame in bytes
- **`stco`**: Chunk offset — byte offset of each chunk in the `mdat` atom
- **`co64`**: 64-bit chunk offset (used when file > 4GB)

The `stsz` atom is particularly important for ALAC because each frame's size is needed to parse the elementary stream. ALAC frames are **variable-length** — the size is determined by the entropy coding, not fixed.

```
stsz atom:
  Size (4 bytes)
  Type: 'stsz'
  Version (1 byte): 0
  Flags (3 bytes): 0
  Sample size (4 bytes): 0 (means variable size per sample)
  Sample count (4 bytes): number of ALAC frames
  Entry size[0..N-1] (4 bytes each): size of each frame in bytes
```

---

## 8. FFmpeg / libavcodec INTEGRATION

### 8.1 Encoder and Decoder Availability

FFmpeg provides full ALAC encoding and decoding support via `libavcodec`. The codec is registered with the four-character code `'alac'`.

**Encoder**: `libavcodec/alacenc.c` — native FFmpeg ALAC encoder
**Decoder**: `libavcodec/alac.c` — native FFmpeg ALAC decoder
**Muxer**: `libavformat/movenc.c` (for `.m4a` / `.mp4` output)
**Demuxer**: `libavformat/mov.c` (for `.m4a` / `.mp4` input)

### 8.2 FFmpeg Command-Line Usage

**Encoding** (PCM to ALAC):

```bash
ffmpeg -i input.wav -c:a alac output.m4a
```

**Encoding with explicit codec options**:

```bash
ffmpeg -i input.wav -c:a alac -frame_size 4096 output.m4a
```

**Decoding** (ALAC to PCM):

```bash
ffmpeg -i input.m4a -c:a pcm_s16le output.wav
ffmpeg -i input.m4a -c:a pcm_s24le output_24bit.wav
ffmpeg -i input.m4a -c:a pcm_f32le output_float.wav
```

**Demuxing without decoding** (extract raw ALAC stream):

```bash
ffmpeg -i input.m4a -c:a copy -f alac output.alac
```

**Inspecting codec info**:

```bash
ffprobe -v trace -i input.m4a 2>&1 | grep -A 50 'Stream'
ffprobe -v trace -i input.m4a 2>&1 | grep -i 'alac\|magic'
```

### 8.3 FFmpeg Codec Options

The FFmpeg ALAC encoder accepts the following private options:

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `frame_size` | int | 4096 | Number of samples per frame per channel |
| `compression_level` | int | 0 | Encoding effort (0=fast, higher=slower but better compression). Note: ALAC is always lossless regardless of this setting; higher values search more exhaustively for optimal parameters |
| `min_frames` | int | 0 | Minimum number of frames to analyze |
| `max_frames` | int | 0 | Maximum number of frames to analyze |

The `compression_level` option is analogous to FLAC's encoding level — higher values enable more thorough parameter optimization searches, potentially improving compression ratio by 1-5% at the cost of much slower encoding.

### 8.4 FFmpeg Magic Cookie Extraction

FFmpeg automatically generates and writes the ALAC magic cookie when muxing to `.m4a`. The cookie is stored in the `kuki` atom within the `stsd` entry.

To extract the magic cookie using FFmpeg:

```bash
# Method 1: Parse container metadata
ffprobe -v quiet -print_format json -show_streams input.m4a | jq '.streams[0].codec_priv_data'

# Method 2: Using the binary codec data
ffprobe -v trace -i input.m4a 2>&1 | grep -A 20 'codec_tag=alac'

# Method 3: Extract the raw 'alac' atom
ffmpeg -i input.m4a -c:v copy -c:a copy -map 0:v -f framecrc - 2>/dev/null | head -1
```

### 8.5 libavcodec API Usage

**Decoder initialization**:

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>

AVCodec *codec = avcodec_find_decoder_by_name("alac");
AVCodecContext *ctx = avcodec_alloc_context3(codec);
avcodec_open2(ctx, codec, NULL);

// Read codec initialization data from the container's kuki atom
// This is available as ctx->extradata (size = ctx->extradata_size)
// The extradata is the 32-byte ALAC magic cookie

AVFrame *frame = av_frame_alloc();
AVPacket *pkt = av_packet_alloc();

// Decode loop
while (av_read_frame(fmt_ctx, pkt) >= 0) {
    if (pkt->stream_index == audio_stream_index) {
        avcodec_send_packet(ctx, pkt);
        while (avcodec_receive_frame(ctx, frame) >= 0) {
            // frame->data[0] contains PCM samples
            // frame->nb_samples = number of samples
            // frame->format = AV_SAMPLE_FMT_S16, S32, or FLTP
            process_pcm(frame);
        }
    }
    av_packet_unref(pkt);
}
```

**Encoder initialization**:

```c
AVCodec *codec = avcodec_find_encoder_by_name("alac");
AVCodecContext *ctx = avcodec_alloc_context3(codec);
ctx->sample_rate = 44100;
ctx->channels = 2;
ctx->sample_fmt = AV_SAMPLE_FMT_S16P;  // planar signed 16-bit
ctx->bit_rate = 0;  // lossless, no bitrate target

// Set ALAC private options
av_opt_set(ctx->priv_data, "frame_size", "4096", 0);
av_opt_set(ctx->priv_data, "compression_level", "1", 0);

avcodec_open2(ctx, codec, NULL);

// The muxer (e.g., MOV/MOV_MUXER) will automatically write the magic cookie
// as the kuki atom in the stsd entry.
```

### 8.6 extradata — The Magic Cookie in libavcodec

The ALAC decoder requires the 32-byte magic cookie as `extradata`. This is passed from the demuxer to the decoder through the `AVCodecContext.extradata` field:

```
ctx->extradata      → pointer to 32-byte magic cookie
ctx->extradata_size → 32
```

FFmpeg's MOV demuxer (`mov.c`) parses the `kuki`/`alac` atom from the `stsd` entry and populates `extradata` accordingly. If the container is missing the magic cookie, the decoder will fail to initialize.

When encoding, the MOV muxer (`movenc.c`) generates the magic cookie from the encoder's `priv_data` and writes it to the output file's `kuki` atom.

### 8.7 Sample Format Support

FFmpeg's ALAC decoder outputs the following sample formats:

| Sample Format | Bit Depth | Planar | Notes |
|---------------|-----------|--------|-------|
| `AV_SAMPLE_FMT_S16` | 16-bit | No (interleaved) | Default for 16-bit input |
| `AV_SAMPLE_FMT_S32` | 32-bit | No (interleaved) | For 20/24/32-bit input |
| `AV_SAMPLE_FMT_FLTP` | 32-bit float | Yes (planar) | Floating-point output |

The encoder accepts:
- `AV_SAMPLE_FMT_S16P`: 16-bit planar (L PCM)
- `AV_SAMPLE_FMT_S32P`: 32-bit integer planar
- `AV_SAMPLE_FMT_FLTP`: 32-bit float planar

---

## 9. BIT DEPTH AND PCM HANDLING

### 9.1 PCM Representation in ALAC

ALAC processes audio in **signed integer form**. The codec treats all samples as signed 32-bit integers internally, regardless of the nominal bit depth. Higher bit depths (20-bit, 24-bit) are represented by using more significant bits of the 32-bit integer; the lower unused bits are typically zero.

For a 16-bit source:
- PCM samples are in the range −32768 to +32767
- Represented as signed 16-bit values
- Shift = 0 in the frame header

For a 24-bit source:
- PCM samples are in the range −8388608 to +8388607
- Represented as signed 24-bit values within 32-bit containers
- Shift = 0 (no wasted bits to signal)

For a 24-bit source where the lower 8 bits are always zero (e.g., audio originally recorded at 16-bit, upsampled to 24-bit):
- The frame header `shift` field may be set to 8
- This signals to the decoder that the 8 least-significant bits of each sample are not encoded (they are known to be zero)

### 9.2 Wasted Bits (shift Parameter)

The `shift` field in the per-frame header indicates the number of trailing zero bits in each PCM sample. This optimization avoids encoding bits that are known to be zero:

- `shift = 0`: No wasted bits, all bits of the input are significant
- `shift = 8`: 8 LSBs are zero (16-bit samples stored in 24-bit format)
- `shift = N`: `N` LSBs are zero

The encoder computes the shift as the number of trailing zeros in the binary representation of all samples (checked across the frame). The decoder right-shifts reconstructed samples by `shift` before output.

---

## 10. ERROR HANDLING AND RECOVERY

### 10.1 Bitstream Errors

ALAC has **no built-in error correction** mechanism. Like most entropy-coded streams, it is highly sensitive to bit errors. A single flipped bit in the residual data can cause:

1. **Incorrect Rice decoding** for the affected sample and all subsequent samples in the frame
2. **Desynchronization** between encoder and decoder state (adaptive k parameter gets corrupted)
3. **Propagation** of errors until the end of the frame

This is a fundamental characteristic of entropy coding: the decoder relies on the encoded data to maintain its internal state. Once corrupted, it cannot self-correct.

### 10.2 Frame Boundary Recovery

The decoder recovers at the next frame boundary. Each frame's header contains enough information (`nbytes`) to locate the start of the next frame, assuming the byte count is correct. If `nbytes` itself is corrupted, the decoder will misalign for the remainder of the file.

**Mitigation strategies**:
- Container-level checksums (e.g., MP4's `pdin`, or using a container with integrity checks)
- Forward error correction at the storage/transmission layer
- For streaming: periodic resynchronization points (not natively supported in ALAC, but can be simulated by splitting the stream into independently decodable segments)

### 10.3 Decoder Robustness

A well-implemented ALAC decoder should:
1. Validate the magic cookie before starting decode
2. Check that `frameLength` is a multiple of 8 and within reasonable bounds (e.g., ≤ 65535)
3. Validate `nbytes` is non-zero and sufficient to contain the frame header
4. Check that decoded sample values do not overflow 32-bit signed integer range
5. Detect if the decoder state (history buffer, k parameter) goes out of reasonable bounds

### 10.4 Known Corruptions and Symptoms

| Symptom | Likely Cause |
|---------|-------------|
| Audio cuts out mid-playback | Corrupted frame header, `nbytes` too small |
| Static noise burst at frame boundary | Bit error in residual data |
| Entire track plays as noise | Magic cookie corrupted or missing |
| Decoder hangs | `nbytes` is 0 or wraps around |
| Volume spike at end of track | Residual accumulation near frame boundary |
| One channel drops out | Asymmetric corruption in stereo interleaved frame |

---

## 11. PERFORMANCE CHARACTERISTICS

### 11.1 Encoding Speed

ALAC encoding is significantly slower than decoding due to the adaptive parameter search. The encoder must:

1. Run the adaptive predictor forward through the frame
2. Perform the k-parameter optimization search (up to `searchNegCeil + searchPosCeil` iterations)
3. Encode the frame for each candidate k to find the minimum size

Approximate relative encoding speeds (FFmpeg, single-threaded):

| Codec | Relative Speed | Notes |
|-------|---------------|-------|
| ALAC | 1x (baseline) | Single-threaded reference |
| ALAC (compression_level=5) | ~0.3x | More exhaustive parameter search |
| FLAC (level 0) | ~3x | Fast encoding, minimal compression |
| FLAC (level 5) | ~0.8x | Balanced |
| FLAC (level 12) | ~0.1x | Maximum compression |
| WavPack (fast) | ~2x | Fast mode |
| APE (fast) | ~0.5x | Moderate compression |

### 11.2 Decoding Speed

ALAC decoding is extremely fast and suitable for real-time playback on any modern hardware. The decoder's operations are:

1. Parse frame header (3 bytes)
2. Initialize k parameter from magic cookie defaults
3. Adaptive Rice decode each residual (bit-shift operations only)
4. Apply adaptive FIR predictor (integer multiply-accumulate)
5. Reconstruct PCM samples

A single-threaded ALAC decoder on a modern CPU can decode at **500-1000x real-time** or faster, meaning it easily handles 8+ simultaneous streams.

### 11.3 Memory Footprint

The decoder requires:
- **History buffer**: `frameLength` samples × `numChannels` × 4 bytes (for 32-bit samples)
  - For `frameLength=4096` and stereo: 4096 × 2 × 4 = 32 KB
- **Filter coefficients**: ~32 coefficients × 4 bytes = 128 bytes
- **Entropy state**: a few hundred bytes
- **Output buffer**: `frameLength` × `numChannels` × 4 bytes

Total per-channel-instance memory: **~40 KB**, which is extremely small.

### 11.4 Compression Ratio Benchmarks

Compression ratios measured on CD-quality 44.1kHz/16-bit stereo audio (diverse music corpus):

| Codec | Mode | Ratio | Notes |
|-------|------|-------|-------|
| ALAC | Default | 58-65% | 16-bit stereo |
| ALAC | High compression | 55-62% | compression_level > 0 |
| FLAC | Level 0 | 65-70% | Fast encoding |
| FLAC | Level 5 | 60-66% | Balanced |
| FLAC | Level 12 | 58-64% | Maximum compression |
| WavPack | Fast | 60-66% | |
| WavPack | High | 52-58% | |
| APE | Fast | 55-62% | |
| APE | Insane | 50-56% | Very slow |
| TAK | Fast | 58-64% | |

ALAC's compression is competitive with FLAC at similar encoding effort levels, but lags behind specialized high-compression codecs like WavPack and APE.

---

## 12. COMPARISON WITH OTHER LOSSLESS CODECS

### 12.1 ALAC vs FLAC

| Aspect | ALAC | FLAC |
|--------|------|------|
| Container | QuickTime/MP4 (ISO BMFF) | Native stream or OGG |
| Metadata | iTunes-style atoms | Vorbis Comments |
| Predictor | Adaptive FIR, per-frame optimization | Fixed-order or LPC, per-subframe |
| Entropy coding | Golomb-Rice (adaptive k) | Golomb-Rice (fixed k per subframe) |
| Block size | Fixed per file (magic cookie) | Variable (per frame) |
| Compression | Moderate | Similar to slightly better |
| Encoding speed | Slow | Fast to moderate |
| Decoding speed | Very fast | Very fast |
| Streaming support | Requires container framing | Native frame-level independence |
| Patent status | Free (Apache 2.0) | Free (BSD 3-Clause) |
| Ecosystem | Apple devices, iTunes | Universal (all platforms) |

### 12.2 ALAC vs WavPack

WavPack offers significantly better compression (especially in high modes) but at the cost of much slower encoding and higher decoder complexity. WavPack also supports a hybrid lossy+lossless mode that ALAC does not have.

### 12.3 ALAC vs APE

Monkey's Audio (APE) achieves the best compression ratios among mainstream lossless codecs but requires prohibitively slow encoding (100x slower than ALAC for the highest modes). APE uses a different entropy coding scheme (Range Coder) compared to ALAC's Golomb-Rice.

### 12.4 ALAC vs DTS-HD MA

DTS-HD Master Audio is a lossless codec used in Blu-ray discs. It supports higher bitrates and is designed for cinema-grade audio. It is not typically used for music distribution and has a much more complex codec structure.

---

## 13. KNOWN ISSUES AND LIMITATIONS

### 13.1 Silent Gap Between Tracks

In early versions of iTunes-encoded ALAC files, there could be a 1-sample silence gap between tracks when concatenating ALAC files. This is a metadata/muxer issue (not a codec issue) related to how the QuickTime container handles sample boundaries.

### 13.2 Floating-Point PCM

Standard ALAC does not support floating-point PCM input. Some implementations (including FFmpeg) can convert float to integer before encoding, but this is not part of the original specification. When processing float sources, you must quantize to integer before ALAC encoding.

### 13.3 32-bit Integer Overflow

The adaptive FIR predictor uses 32-bit signed integer arithmetic. With very loud signals (approaching 0 dBFS) on 24-bit audio, the predictor's internal state may overflow 32-bit integers. The original Apple implementation handles this via undefined behavior in C (wrapping is implementation-defined), but well-behaved implementations should detect and handle potential overflow.

### 13.4 No Seeking Index

The ALAC elementary stream itself contains no seek table. Seeking within an ALAC stream requires scanning from the beginning of the stream or relying on the container's seek table (e.g., QuickTime's `cslg` atom or custom seek table). This is not a codec limitation per se — it is a container issue.

### 13.5 Limited Channel Support

ALAC natively supports up to 8 audio channels (7.1 surround). Beyond that, it falls back to multiple mono streams, which is not standardized. The channel layout must be communicated via the container (e.g., the `chpl` channel layout atom in MP4).

---

## 14. REFERENCES AND RESOURCES

### 14.1 Official Apple Sources

- **Apple Open Source — ALAC**: `https://opensource.apple.com/source/AppleLosslessDecoder/`
- **Apple ALAC Encoder**: `https://opensource.apple.com/source/AppleLosslessEncoder/`
- **QuickTime File Format Specification**: Available from Apple Developer Documentation

### 14.2 FFmpeg Sources

- **ALAC decoder** (`libavcodec/alac.c`): Primary reference implementation
- **ALAC encoder** (`libavcodec/alacenc.c`): Reference encoder
- **MOV muxer** (`libavformat/movenc.c`): Writes ALAC magic cookie to `kuki` atom
- **MOV demuxer** (`libavformat/mov.c`): Reads ALAC magic cookie from `kuki`/`alac` atom

### 14.3 Reverse Engineering Sources

- **alac-utils**: Community project implementing the reverse-engineered ALAC codec
- **FAAD2**: ALAC decoder-only implementation (useful reference for decoder simplicity)

### 14.4 Related Standards

- **ISO/IEC 14496-12** (ISO Base Media File Format): Defines the `.m4a` container structure
- **ITU-R BS.1196**: ATSC Digital Audio Compression (describes similar predictive coding techniques)
- **RFC 6386**: VP8 Data Format and Decoding Guide (Golomb-Rice coding reference)

---

## 15. TOOLS AND UTILITIES

### 15.1 FFmpeg

Primary tool for ALAC encoding, decoding, transcoding, and metadata manipulation:

```bash
# Basic transcode
ffmpeg -i input.flac -c:a alac output.m4a

# Extract ALAC stream without re-encoding
ffmpeg -i input.m4a -c:a copy output.alac

# Decode to WAV
ffmpeg -i input.m4a output.wav

# Encode with higher compression effort
ffmpeg -i input.wav -c:a alac -compression_level 5 output.m4a

# Encode 24-bit ALAC
ffmpeg -i input_24bit.wav -c:a alac -sample_size 24 output.m4a

# Verify codec info
ffprobe -show_streams -select_streams a:0 input.m4a

# Extract album art
ffmpeg -i input.m4a -c:v copy cover.jpg
```

### 15.2 MediaInfo

For inspecting ALAC file metadata and codec parameters:

```bash
mediainfo input.m4a
mediainfo --Full input.m4a
```

### 15.3 AtomicParsley

For editing iTunes/metadata atoms in M4A files without touching the audio:

```bash
# View all atoms
AtomicParsley input.m4a -t

# Edit metadata
AtomicParsley input.m4a --artist "Artist Name" --album "Album Name" --track 1
```

### 15.4 MP4Box

GPAC's MP4Box for advanced container manipulation:

```bash
# Extract raw ALAC stream
MP4Box -raw 1 input.m4a

# Extract specific track
MP4Box -raw item=1 input.m4a

# Get file info
MP4Box -info input.m4a
```

### 15.5 kid3

For viewing and editing ALAC metadata tags:

```bash
kid3-cli -c "get" input.m4a
kid3-cli -c "set artist 'Artist'" input.m4a
```

---

## 16. IMPLEMENTATION CHECKLIST

This checklist is for developers building ALAC encoding/decoding pipelines.

### 16.1 Decoder Implementation Checklist

```
[ ] Parse and validate the 32-byte ALAC magic cookie
    [ ] Verify format field == 'alac'
    [ ] Validate frameLength is non-zero and a multiple of 8
    [ ] Validate maxK is in range 0-31
    [ ] Validate historyMultipler is in range 1-255
    [ ] Validate all other parameters are in valid ranges

[ ] Initialize decoder state
    [ ] Allocate history buffer: frameLength × numChannels × sizeof(int32_t)
    [ ] Initialize entropy state with initialHistory value
    [ ] Initialize k parameter to 0
    [ ] Reset predictor coefficients to zero

[ ] For each frame:
    [ ] Read 3-byte (or 4-byte for stereo) frame header
    [ ] Parse shift, nbytes from frame header
    [ ] Locate frame data boundary using nbytes
    [ ] Initialize k from estimated value (using initial search range)
    [ ] For each sample:
        [ ] Read unary-coded quotient from bitstream (Golomb-Rice decode)
        [ ] Read k-bit suffix from bitstream
        [ ] Reconstruct signed residual r[n]
        [ ] Update adaptive k parameter based on |r[n]|
        [ ] Compute predicted sample x̂[n] using adaptive FIR
        [ ] Reconstruct original sample: x[n] = x̂[n] + r[n]
        [ ] Update predictor coefficients
        [ ] Store x[n] in output buffer
    [ ] Apply shift to output samples (right-shift by shift bits)
    [ ] If stereo M/S: transform mid/side back to L/R
    [ ] Output frame of PCM samples

[ ] Handle error cases
    [ ] Detect and report corrupted frame headers
    [ ] Handle nbytes == 0 (empty frame = silence)
    [ ] Detect entropy state overflow (k exceeding maxK)
    [ ] Handle end-of-stream gracefully

[ ] Validate output
    [ ] Verify PCM samples are within expected bit depth range
    [ ] Verify no NaN or infinity in floating-point paths
    [ ] Verify channel count matches container specification
```

### 16.2 Encoder Implementation Checklist

```
[ ] Read magic cookie parameters (or use defaults)
    [ ] Set frameLength from container spec
    [ ] Set maxK, historyMultipler, initialHistory, kModifier, searchNegCeil, searchPosCeil

[ ] For each input frame:
    [ ] Read PCM samples from source
    [ ] Convert to signed representation if needed
    [ ] If stereo: transform L/R to mid/side
    [ ] Apply shift optimization:
        [ ] Find minimum trailing zeros across all samples in frame
        [ ] Set shift field to this value
        [ ] Right-shift all samples by shift bits

    [ ] For k optimization search:
        [ ] For each candidate k in [estimated_k − searchNegCeil, estimated_k + searchPosCeil]:
            [ ] Simulate encoder with this k
            [ ] Count total encoded bits
            [ ] Track minimum
        [ ] Select k that minimizes encoded size

    [ ] Encode frame:
        [ ] Write frame header (shift + nbytes placeholder)
        [ ] For each sample:
            [ ] Compute predicted value using adaptive FIR
            [ ] Compute residual r[n] = x[n] − x̂[n]
            [ ] Encode r[n] using Golomb-Rice with optimal k
            [ ] Update k adaptively (same as decoder)
            [ ] Update predictor coefficients
        [ ] Fill in actual nbytes value

    [ ] Write encoded frame to output stream

[ ] Generate magic cookie for container
    [ ] Write 32-byte cookie to stsd/kuki atom
    [ ] Ensure format='alac', flags=0
    [ ] Fill all parameter fields

[ ] Container integration
    [ ] Write ALAC frames to mdat atom
    [ ] Write frame sizes to stsz atom entries
    [ ] Write codec configuration to alac/kuki atom in stsd
    [ ] Set correct sample rate and channel count in mdhd and smhd
```

### 16.3 Container (Muxer/Demuxer) Implementation Checklist

```
[ ] Writing ALAC files (.m4a):
    [ ] Create ftyp atom with brand 'M4A ' and compatible brands
    [ ] Create moov/mvhd with correct time scale and duration
    [ ] Create moov/trak/mdia/mdhd with sample rate, channel count, total samples
    [ ] Create moov/trak/mdia/minf/smhd with balance=0
    [ ] Create moov/trak/mdia/minf/stbl/stsd with 'alac' entry:
        [ ] Embed 32-byte magic cookie in 'kuki' sub-atom
        [ ] Set correct channel count and sample size
        [ ] Set sample_rate in fixed-point 16.16 format
    [ ] Create moov/trak/mdia/minf/stbl/stts with frame count and durations
    [ ] Create moov/trak/mdia/minf/stbl/stsc with chunk mapping
    [ ] Create moov/trak/mdia/minf/stbl/stsz with frame sizes
    [ ] Create moov/trak/mdia/minf/stbl/stco with chunk offsets
    [ ] Write ALAC frames to mdat atom
    [ ] Update mdat size after writing all frames

[ ] Reading ALAC files (.m4a):
    [ ] Parse ftyp to verify M4A brand
    [ ] Locate audio track in moov/trak
    [ ] Extract sample rate, channel count from mdhd
    [ ] Extract magic cookie from stsd entry (kuki/alac atom)
    [ ] Pass magic cookie as extradata to ALAC decoder
    [ ] Read frame sizes from stsz
    [ ] Read frame data from mdat using offsets from stco
    [ ] Map timestamps using stts

[ ] Metadata handling:
    [ ] Preserve all iTunes/metadata atoms (covr, ©nam, ©ART, etc.)
    [ ] Do not modify metadata atoms during audio transcoding
    [ ] Copy artwork (covr) from source to destination
```

---

## 17. APPENDIX — BYTE-LEVEL DECODE ALGORITHM

### 17.1 Complete Decoder Pseudocode

This pseudocode provides a complete reference for implementing an ALAC decoder from scratch:

```
function decode_alac_frame(bitstream, magic_cookie, num_channels, output_buffer):
    # --- Parse frame header ---
    shift = read_bits(bitstream, 8)
    nbytes_high = read_bits(bitstream, 8)
    nbytes_low = read_bits(bitstream, 8)
    nbytes = (nbytes_high << 8) | nbytes_low

    if num_channels == 2:
        num_ch = read_bits(bitstream, 8)  # Usually 2 for stereo composite
    else:
        num_ch = 1

    frame_size = magic_cookie.frameLength
    expected_residuals = frame_size * num_ch

    # --- Initialize entropy state ---
    k = 0
    history = magic_cookie.initialHistory
    block_size = 0
    cur_frame = 0

    # --- Initialize predictor ---
    # predictor_coeffs[] = array of 32 int32 values, initialized to 0
    # history_buffer[] = array of frame_size int32 values per channel, initialized to 0

    # --- Decode residuals ---
    for sample_idx in range(frame_size):
        for ch in range(num_ch):
            # --- Golomb-Rice decode one residual ---
            # Read unary-coded quotient
            q = 0
            while read_bit(bitstream) == 0:
                q += 1
                if q > 1000:  # Safety limit for corrupted streams
                    break

            # Read k-bit suffix
            if k > 0:
                suffix = read_bits(bitstream, k)
            else:
                suffix = 0

            # Reconstruct residual magnitude
            abs_residual = (q << k) | suffix

            # Handle sign
            sign_bit = read_bit(bitstream)
            if sign_bit == 1:
                residual = -abs_residual
            else:
                residual = abs_residual

            # --- Update adaptive k ---
            history += abs_residual
            expected = history >> 9  # Equivalent to history / 512
            if expected >= (1 << k):
                while expected >= (1 << k):
                    k += 1
                    if k > magic_cookie.maxK:
                        k = magic_cookie.maxK
            else:
                while expected < (1 << (k - 1)) and k > 0:
                    k -= 1

            # --- Apply predictor ---
            predicted = 0
            for i in range(32):
                if sample_idx > i:
                    predicted += predictor_coeffs[i] * history_buffer[ch][(sample_idx - 1 - i) % frame_size]

            # Reconstruct sample (at encoder: x = x̂ + r; at decoder: same formula)
            reconstructed = predicted + residual

            # --- LMS coefficient update ---
            error = residual  # Same as reconstructed - predicted
            for i in range(32):
                if sample_idx > i:
                    # LMS update: coeff += μ * error * sample[n-i-1]
                    mu = magic_cookie.historyMultipler * error >> 9
                    predictor_coeffs[i] += mu * history_buffer[ch][(sample_idx - 1 - i) % frame_size]

            # --- Store and output ---
            # Apply shift (undo wasted bits)
            if shift > 0:
                reconstructed = reconstructed << shift
                # For signed right-shift (implementation-dependent):
                reconstructed = reconstructed / (1 << shift)  # Must use arithmetic right-shift

            output_buffer[ch][sample_idx] = reconstructed
            history_buffer[ch][sample_idx % frame_size] = reconstructed

    # --- Stereo reconstruction ---
    if num_ch == 2:
        for i in range(frame_size):
            mid = output_buffer[0][i]   # This is actually 'mid'
            side = output_buffer[1][i]  # This is actually 'side'
            left = mid + side
            right = mid - side
            output_buffer[0][i] = left
            output_buffer[1][i] = right

    return output_buffer
```

### 17.2 Golomb-Rice Decoding in Detail

The Golomb-Rice decoder is the innermost loop and must be highly optimized. The key operations are:

```
RiceDecode(bitstream, k):
    # Step 1: Read unary-coded quotient
    q = 0
    while ReadBit(bitstream) == 0:
        q += 1

    # Step 2: Read k-bit suffix
    suffix = (k > 0) ? ReadBits(bitstream, k) : 0

    # Step 3: Reconstruct value
    magnitude = (q << k) | suffix

    # Step 4: Apply sign
    if ReadBit(bitstream) == 1:
        return -magnitude
    else:
        return magnitude
```

The `ReadBit` function reads one bit from the bitstream buffer, advancing the bit position. The `ReadBits(n)` function reads `n` bits as a binary integer (MSB first, consistent with big-endian bit ordering within bytes).

### 17.3 k Adaptation Algorithm

The adaptive k parameter update must be identical in both encoder and decoder:

```
UpdateK(history, k, maxK):
    # history is the running sum of |residuals|
    # This is equivalent to: history += |r[n]|
    expected = history >> 9  # Divide by 512

    if expected >= (1 << k):
        while expected >= (1 << k) and k < maxK:
            k += 1
    else:
        while expected < (1 << (k - 1)) and k > 0:
            k -= 1

    return k
```

The shift by 9 bits (division by 512) maps the history sum to an expected absolute residual value at the current Rice parameter level. This is a form of exponential moving average.

### 17.4 LMS Predictor Update

The LMS (Least Mean Squares) predictor update uses the following formula:

```
μ = historyMultipler * error >> 9
for i = 1 to 32:
    coeff[i] += μ * sample[n - i]
```

Where `error` is the prediction residual `r[n] = x[n] - x̂[n]`. The shift by 9 implements division by 512, which scales the adaptation rate to a reasonable step size. The `historyMultipler` parameter (1-255, default 64) scales this rate.

The `>> 9` operation is equivalent to division by 512, which is the normalization factor that keeps the LMS algorithm stable.

---

*Document Version: 1.0*
*Last Updated: June 2026*
*Target Audience: Audio codec developers building conversion pipelines*
