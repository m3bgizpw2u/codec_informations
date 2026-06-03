# Audio Engineering: PCM Fundamentals, Bit Depth, and Sample Rate — Deep Technical Reference

> **Category:** Audio Fundamentals / PCM (Pulse Code Modulation)
> **File Extensions:** N/A (raw audio encoding, used in .wav, .aiff, .caf, raw PCM)
> **MIME Types:** audio/raw, audio/x-raw, audio/pcm
> **Standardization Body:** Multiple (IEC 60906, ITU-R BS.468, IEEE 754 for float)
> **Patent Status:** Patent-free
> **License:** Public domain

---

## 1. HISTORICAL CONTEXT & ORIGIN

Pulse Code Modulation is the foundational method for digitally representing analog audio signals. The theory was first described mathematically by Alec Reeves in 1937, while working at Standard Telecommunications Laboratories in England. Reeves filed a patent for PCM in 1938, conceptualizing the encoding of voice signals into a serial binary data stream for transmission over copper wire. However, the practical implementation of PCM required electronic circuits that did not exist at that time.

The first practical demonstration of PCM occurred in 1948 at Bell Labs, where researchers built a working system using vacuum tube technology. The early Bell Labs PCM system encoded voice into a 7-bit, 8 kHz stream (56 kbps) transmitted over a single copper pair. This was groundbreaking — it demonstrated that analog voice could be faithfully reconstructed from a digital bitstream, establishing the theoretical foundation for all digital audio.

The introduction of the compact disc (CD) in 1982 standardized PCM at 16-bit, 44.1 kHz stereo, transforming PCM from a professional studio tool into a consumer format. The CD specification was developed jointly by Sony and Philips, with the choice of 44.1 kHz driven by the need to accommodate both 48 kHz (professional video equipment) and 44.056 kHz (NTSC color burst frequency) sample rates through integer-ratio oversampling. The Sony PCM 501 processor (1977) was the first commercial PCM encoder/decoder system for consumer use, recording stereo PCM onto Beta-format video cassettes.

Modern PCM audio operates across an enormous range of bit depths (from 8-bit telephony to 32-bit and 64-bit float in professional DAWs) and sample rates (from 8 kHz telephony to 384 kHz in ultra-high-resolution audio). The underlying principle, however, remains exactly as Reeves described it in 1937: periodic sampling of an analog signal's amplitude, quantization to discrete levels, and binary encoding of those quantized values.

The transition from 16-bit CD audio to higher bit depths (24-bit, 32-bit float) in professional environments was driven by the needs of mixing, mastering, and signal processing. During intermediate processing stages, arithmetic operations (gain, EQ, compression) produce values that exceed the original signal's range. Higher bit depths provide headroom and maintain precision through these operations. Dithering, discovered in the 1970s and formalized in the 1990s, allows the preservation of sub-LSB (least significant bit) information even at 16-bit output.

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 What PCM Actually Is

PCM is a method for representing a continuous analog signal as a sequence of discrete binary values. The process involves three fundamental operations: sampling, quantization, and encoding.

**Sampling** measures the instantaneous amplitude of an analog waveform at regular time intervals. The sample rate (measured in samples per second, or Hertz) determines how frequently these measurements occur. According to the Nyquist-Shannon sampling theorem, a band-limited signal can be perfectly reconstructed from its samples if the sample rate is at least twice the highest frequency present in the signal. The Nyquist frequency is exactly half the sample rate — for CD audio at 44.1 kHz, the Nyquist frequency is 22.05 kHz, providing a small margin above the 20 kHz limit of human hearing.

**Quantization** maps each continuous sample value to one of a finite set of discrete levels. The number of available levels is determined by the bit depth: for N bits, there are 2^N discrete levels. A 16-bit PCM signal has 65,536 possible values. Quantization inherently introduces error — the difference between the original analog value and the quantized digital value. This error manifests as quantization noise, a form of distortion that is spectrally shaped by the signal itself.

**Encoding** converts each quantized sample value into its binary representation. The encoding scheme (signed vs. unsigned, integer vs. floating-point, endianness) determines how binary values map to actual signal amplitudes. Different audio containers and standards use different encoding schemes.

### 2.2 PCM in the Conversion Pipeline

PCM occupies a unique position in audio codec pipelines — it is the universal interchange format. Every lossy codec (MP3, AAC, Opus, Vorbis) accepts PCM as input and produces PCM as output during decode. Lossless codecs (FLAC, ALAC, WavPack) encode PCM to compressed form and decode back to PCM. The quality of PCM representation at the input and output of a codec chain determines the ceiling for overall quality.

Consider a conversion pipeline: FLAC → decode to PCM → encode to AAC. The output AAC quality is limited by the quality of the PCM produced by the FLAC decoder. If the FLAC decoder outputs 16-bit PCM and the source was 24-bit, 8 bits of precision have already been lost before AAC encoding begins. Conversely, if the FLAC decoder outputs 32-bit float and the source was 16-bit, the additional bits contain only numerical noise (upconverted from quantization noise), not actual signal information.

Understanding PCM at the bit level is essential for building conversion pipelines that preserve the maximum possible signal quality at every stage.

### 2.3 PCM Representation Space

For a given bit depth N, the PCM representation space is a range of integers (for integer PCM) or a range of floating-point values (for float PCM):

| Bit Depth | Type | Value Range | Peak SNR (theoretical) |
|---|---|---|---|
| 8 | unsigned integer | 0 to 255 | 48.16 dB |
| 16 | signed integer | −32,768 to +32,767 | 96.33 dB |
| 20 | signed integer | −524,288 to +524,287 | 120.41 dB |
| 24 | signed integer | −8,388,608 to +8,388,607 | 144.49 dB |
| 32 | signed integer | −2,147,483,648 to +2,147,483,647 | 192.66 dB |
| 32 | IEEE 754 float | −3.4028235e+38 to +3.4028235e+38 | N/A (dynamic range) |
| 64 | IEEE 754 float | −1.7976931e+308 to +1.7976931e+308 | N/A (dynamic range) |

The theoretical peak signal-to-noise ratio (SNR) for N-bit PCM is approximately `6.02 × N + 1.76` dB, derived from quantizing a full-scale sine wave. For 16-bit, this is 98.09 dB; for 24-bit, this is 146.25 dB. Real-world performance is lower due to converter imperfections, clock jitter, and thermal noise.

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 PCM Encoding Schemes

PCM audio can be encoded in several fundamentally different schemes. The choice affects how binary data maps to signal amplitudes, and mismatches between encoding schemes are among the most common sources of audio corruption in conversion pipelines.

#### 3.1.1 Unsigned Integer PCM (offset binary)

In unsigned integer PCM, all values are interpreted as positive integers. The signal is centered around the midpoint of the range. This encoding is unusual for professional audio but appears in:

- 8-bit WAV files (PCMs教科)
- Some raw audio formats
- Older telephony systems

For 8-bit unsigned with range 0–255, the midpoint value 128 represents zero amplitude. The range 0–127 represents the negative half-cycle, and 129–255 represents the positive half-cycle. Value 0 is the most negative signal, and value 255 is the most positive signal.

**Mapping formula (unsigned → normalized):**

```
normalized = (unsigned_sample - 128) / 127.0
```

#### 3.1.2 Signed Integer PCM (two's complement)

Signed integer PCM uses two's complement representation, which is the standard for professional audio. Two's complement allows simple binary addition and subtraction while representing both positive and negative values symmetrically around zero.

For N-bit signed integer:

| Value | Representation |
|---|---|
| 0 | 0x0000... (all zeros) |
| Maximum positive | 0x7FFF... (sign bit 0, all others 1) |
| Minimum negative | 0x8000... (sign bit 1, all others 0) |
| −1 | 0xFFFF... (all ones) |

For 16-bit signed integer:
- Range: −32,768 to +32,767
- Zero amplitude: 0x0000
- Positive full-scale: 0x7FFF (+1.0 in normalized float)
- Negative full-scale: 0x8000 (−1.0 in normalized float)

**Important:** The negative and positive ranges are asymmetric in two's complement. There is one more negative value than positive value. This asymmetry means that a full-scale sine wave encoded in two's complement uses the range −32,767 to +32,767, leaving 0x8000 (the most negative value) unused. Some encoders use 0x8000 as a special "clip" indicator, while others simply never generate it.

**Mapping formula (signed → normalized float):**

```
normalized = signed_sample / 32768.0
```

#### 3.1.3 IEEE 754 Floating-Point PCM

Floating-point PCM represents values as scientific notation in binary. The IEEE 754 standard defines two formats commonly used in audio:

**IEEE 754-1985 Single Precision (32-bit float):**

```
Sign (1 bit) | Exponent (8 bits) | Mantissa (23 bits)
```

The value is computed as:

```
(-1)^sign × 2^(exponent - 127) × 1.mantissa
```

The mantissa is normalized to have an implicit leading 1 (except for denormals). The exponent bias is 127.

For audio use, 32-bit float typically uses the range −1.0 to +1.0 as the nominal full-scale signal range. Values outside this range represent clipping. The exponent field allows values far outside this range (for representing overflow during intermediate computation), but audio content is expected to stay within ±1.0.

**IEEE 754-1985 Double Precision (64-bit float):**

```
Sign (1 bit) | Exponent (11 bits) | Mantissa (52 bits)
```

Bias is 1023. Even larger range and precision than 32-bit float.

**Key properties of floating-point PCM:**

- The representation space is not uniformly distributed. Resolution is finest near zero and coarser at larger magnitudes.
- There is no fixed "full-scale" value. ±1.0 is a convention, not a hard limit.
- Denormalized numbers (exponent = 0, non-zero mantissa) exist near zero but are problematic on many CPUs (slow or flushed to zero).
- NaN and infinity can theoretically appear but should never occur in valid audio data.
- Rounding during arithmetic operations can introduce very small errors.

**32-bit float vs. 64-bit float in audio:**
- 32-bit float provides approximately 24 bits of mantissa precision (including the implicit bit).
- 64-bit float provides approximately 53 bits of precision.
- The practical benefit of 64-bit float is primarily in accumulated computation (e.g., summing thousands of samples), where rounding errors in 32-bit float could accumulate to audible levels.
- For a single pass of gain or EQ, 32-bit float is generally sufficient.

#### 3.1.4 μ-Law and A-Law Companding (ITU-T G.711)

μ-law (used in North America and Japan) and A-law (used in Europe and internationally) are non-linear PCM encoding schemes designed to improve the effective dynamic range of 8-bit PCM for voice telephony.

Instead of uniform quantization, companding (compressing-expanding) uses logarithmic quantization. Near-zero amplitudes use fine quantization steps; large amplitudes use coarser steps. This achieves approximately 12–13 bits of effective dynamic range in an 8-bit word.

**μ-law formula (μ = 255):**

```
sign = sample < 0 ? -1 : +1
magnitude = abs(sample)
compressed = sign × (ln(1 + μ × normalized_magnitude) / ln(1 + μ))
```

**A-law formula (A = 87.6):**

```
For |x| < 1/A:    compressed = A × |x|
For |x| >= 1/A:   compressed = sign × (1 + ln(A × |x|)) / (1 + ln(A))
```

PCM converters must properly handle μ-law and A-law source material by expanding it back to linear PCM before processing with other codecs.

### 3.2 Endianness

The byte order of multi-byte integer PCM values varies by platform and container format:

**Little-endian (LE):** Least significant byte stored first. Used by:
- WAV files on all platforms (RIFF/WAVE standard)
- Raw PCM on x86, x86-64, and ARM (little-endian) platforms
- AIFF files on Mac (historically big-endian, but AIFF-C can be LE)

**Big-endian (BE):** Most significant byte stored first. Used by:
- AIFF/AIFC files (Audio Interchange File Format)
- CAF files (Core Audio Format, optionally)
- Raw PCM on big-endian platforms (PowerPC, SPARC historically)
- MP3 frame headers (and most MPEG structures)

**Byte-swap formula for converting between endianness:**

```python
def swap_endian_16(value: int) -> int:
    return ((value & 0xFF) << 8) | ((value >> 8) & 0xFF)

def swap_endian_24(value: int) -> int:
    return ((value & 0xFF) << 16) | ((value >> 8) & 0xFF) | ((value >> 16) & 0xFF)

def swap_endian_32(value: int) -> int:
    return ((value & 0xFF) << 24) | ((value >> 8) & 0xFF00) | \
           ((value >> 16) & 0xFF0000) | ((value >> 24) & 0xFF000000)
```

### 3.3 Sample Frame Layout (Interleaved vs. Planar)

PCM audio can be stored in two fundamentally different memory layouts, which matters enormously for buffer handling in conversion pipelines.

**Interleaved (most common):**

```
[L0][R0][L1][R1][L2][R2]...
```

Samples from all channels are interleaved in memory, one sample frame at a time. This is the standard layout for WAV, AIFF, MP3 output, and most audio hardware interfaces. Access pattern for sample n, channel c in an interleaved buffer with `channels` channels:

```python
byte_offset = (n * channels + c) * bytes_per_sample
sample_value = read_value_from_buffer(buffer, byte_offset, bytes_per_sample)
```

**Planar (also called non-interleaved):**

```
[L0][L1][L2]...[R0][R1][R2]...
```

All samples for channel 0 are stored contiguously, followed by all samples for channel 1, and so on. This layout is used by some lossless codecs (e.g., older FLAC implementations), some video codecs (e.g., FFmpeg's planar audio), and some DSP frameworks. Planar layouts can be more cache-friendly for channel-based processing but are less efficient for sample-based access.

FFmpeg distinguishes these layouts in its sample format enumeration:
- `AV_SAMPLE_FMT_S16` — signed 16-bit, interleaved
- `AV_SAMPLE_FMT_S16P` — signed 16-bit, planar
- `AV_SAMPLE_FMT_FLT` — 32-bit float, interleaved
- `AV_SAMPLE_FMT_FLTP` — 32-bit float, planar
- `AV_SAMPLE_FMT_S32` — signed 32-bit, interleaved

### 3.4 Complete PCM Buffer Calculation Formulas

```python
def calculate_pcm_buffer_size(
    sample_rate: int,
    duration_seconds: float,
    channels: int,
    bits_per_sample: int
) -> int:
    bytes_per_sample = (bits_per_sample + 7) // 8
    samples_per_channel = int(sample_rate * duration_seconds)
    total_samples = samples_per_channel * channels
    return total_samples * bytes_per_sample

def calculate_duration_samples(
    buffer_size_bytes: int,
    sample_rate: int,
    channels: int,
    bits_per_sample: int
) -> float:
    bytes_per_sample = (bits_per_sample + 7) // 8
    samples_per_channel = buffer_size_bytes // (channels * bytes_per_sample)
    return samples_per_channel / sample_rate

def calculate_bytes_per_second(
    sample_rate: int,
    channels: int,
    bits_per_sample: int
) -> int:
    bytes_per_sample = (bits_per_sample + 7) // 8
    return sample_rate * channels * bytes_per_sample
```

### 3.5 Data Rate Table for Common PCM Configurations

| Bit Depth | Channels | Sample Rate | Bytes/sec | Bits/sec | Data Rate |
|---|---|---|---|---|---|
| 8 | 1 (mono) | 8000 | 8,000 | 64,000 | 64 kbps |
| 8 | 2 (stereo) | 8000 | 16,000 | 128,000 | 128 kbps |
| 16 | 1 (mono) | 44100 | 88,200 | 705,600 | ~691 kbps |
| 16 | 2 (stereo) | 44100 | 176,400 | 1,411,200 | ~1.38 Mbps (CD audio) |
| 16 | 2 (stereo) | 48000 | 192,000 | 1,536,000 | 1.5 Mbps (DVD audio) |
| 24 | 2 (stereo) | 96000 | 576,000 | 4,608,000 | 4.5 Mbps (hi-res) |
| 24 | 8 (7.1 surround) | 96000 | 1,843,200 | 14,745,600 | 14.4 Mbps |
| 32 | 2 (stereo) | 192000 | 1,536,000 | 12,288,000 | 12 Mbps |
| 32 (float) | 2 (stereo) | 48000 | 384,000 | 3,072,000 | 3 Mbps |
| 64 (float) | 2 (stereo) | 48000 | 768,000 | 6,144,000 | 6 Mbps |

---

## 4. BIT DEPTH DETAILED SPECIFICATION

### 4.1 Quantization Error and LSB Size

The fundamental unit of PCM resolution is the Least Significant Bit (LSB). For N-bit PCM with a peak-to-peak range of 2^(N−1) to +(2^(N−1)−1), the size of one LSB is:

```
LSB_size = 2 / 2^N = 2^(1-N)
```

| Bit Depth | LSB Size (full-scale) | LSB Size (normalized dBFS) | Practical Implication |
|---|---|---|---|
| 8 | 1/128 | −42.1 dBFS | Very coarse; audible quantization noise |
| 12 | 1/2048 | −66.2 dBFS | Better than early digital telephone |
| 16 | 1/65536 | −90.3 dBFS | CD-quality; adequate for final delivery |
| 20 | 1/1,048,576 | −114.4 dBFS | DAT-quality; professional minimum |
| 24 | 1/16,777,216 | −138.5 dBFS | Industry standard for recording/mastering |
| 32 | 1/4,294,967,296 | −162.7 dBFS | Professional floating-point processing |
| 64 | 1/2^64 | −386.7 dBFS | Academic precision; overkill for audio |

The dBFS value is calculated as `20 × log10(LSB_size)` or `6.02 × N` (approximate SNR per bit).

### 4.2 8-Bit PCM (Unsigned)

8-bit PCM was the first widely deployed digital audio format, primarily in telephony and early consumer applications. The standard WAV format for 8-bit is unsigned (range 0–255), where 128 is the zero point.

**Byte layout for 8-bit unsigned mono:**
```
Byte 0: Sample 0 (0=minimum, 128=zero, 255=maximum)
Byte 1: Sample 1
...
```

**Normalized conversion:**

```python
# Unsigned 8-bit (0-255) to float (-1.0 to +1.0)
def uint8_to_float(sample: int) -> float:
    return (sample - 128) / 127.0

# Float (-1.0 to +1.0) to unsigned 8-bit
def float_to_uint8(sample: float) -> int:
    return int(round(sample * 127.0 + 128.0))
```

**SNR characteristics:** The theoretical SNR of 8-bit PCM is approximately 49.9 dB (using the full-scale sine wave formula: 6.02 × 8 + 1.76). However, because unsigned 8-bit PCM has a DC offset at 128, the actual dynamic range for AC signals is lower. In practice, 8-bit PCM quantization noise is clearly audible, especially on quiet passages. The noise floor is approximately at the −48 dBFS level.

**Applications:** Telephony (PCM at 8 kHz, μ-law), early computer audio (PC speaker, Sound Blaster 8-bit mode), and certain industrial audio applications. 8-bit audio at 8 kHz requires approximately 64 kbps per mono channel and is acceptable for voice but completely inadequate for music.

### 4.3 16-Bit PCM (Signed Integer)

16-bit signed PCM is the standard for CD audio and the most common interchange format in audio software. The range is −32,768 to +32,767, with 0 as the center point.

**Byte layout for 16-bit signed stereo (interleaved, little-endian):**

```
Byte 0: LSB of Sample 0, Channel 0
Byte 1: MSB of Sample 0, Channel 0
Byte 2: LSB of Sample 0, Channel 1
Byte 3: MSB of Sample 0, Channel 1
Byte 4: LSB of Sample 1, Channel 0
Byte 5: MSB of Sample 1, Channel 0
...
```

**Normalized conversion:**

```python
# Signed 16-bit to float (-1.0 to +1.0)
def int16_to_float(sample: int) -> float:
    return sample / 32768.0

# Float to signed 16-bit
def float_to_int16(sample: float) -> int:
    if sample >= 1.0:
        return 32767
    if sample <= -1.0:
        return -32768
    return int(round(sample * 32768.0))

# C implementation:
# int16_t float_to_int16_clamped(float s) {
#     if (s >= 1.0f) return 32767;
#     if (s <= -1.0f) return -32768;
#     return (int16_t)lrintf(s * 32768.0f);
# }
```

**Peak SNR:** The theoretical peak SNR is approximately 98.1 dB (6.02 × 16 + 1.76). In practice, high-quality 16-bit converters achieve 90–96 dB of dynamic range and THD+N (total harmonic distortion plus noise) of −90 to −100 dBFS.

**Aliasing and imaging in 16-bit:** Quantization noise in 16-bit PCM is spectrally white (uniformly distributed across frequency) when the input is uncorrelated noise or a full-scale signal. For signals below full-scale, the quantization noise is still white but its level is proportionally lower. This white noise floor provides a consistent background that does not modulate with the signal, making it perceptually less annoying than some other noise types.

**Clipping behavior:** When a 16-bit PCM value exceeds +32,767, it wraps around to negative values (in two's complement). This produces harsh digital distortion. In floating-point processing that targets 16-bit output, it is essential to check for and handle values outside the representable range before final conversion. The standard approach is to apply a ceiling function (clipping) or, preferably, apply gain reduction before clipping occurs.

### 4.4 20-Bit PCM

20-bit PCM was the standard for professional digital audio recording through the 1990s and into the 2000s, particularly in formats like DAT (Digital Audio Tape) at 48 kHz. It provides a theoretical SNR improvement of 24 dB over 16-bit.

**Byte layout for 20-bit signed stereo (3 bytes per sample, 6 bytes per frame):**

```
Byte 0: Bits 0-7 (LSB) of Sample 0, Channel 0
Byte 1: Bits 8-15 (middle) of Sample 0, Channel 0
Byte 2: Bits 16-19 (MSB, upper 4 bits of third byte) of Sample 0, Channel 0
Byte 3: Bits 0-7 (LSB) of Sample 0, Channel 1
Byte 4: Bits 8-15 (middle) of Sample 0, Channel 1
Byte 5: Bits 16-19 (MSB) of Sample 0, Channel 1
```

**Packing note:** The 20-bit value is left-aligned within the 24-bit storage space. The unused 4 bits (the 4 least significant bits of the third byte) are typically zeroed. This means the raw 24-bit file might show non-zero values in those unused bits if the encoding was done sloppily.

**Range:** −524,288 to +524,287.

**Conversion to/from float:**

```python
def int20_to_float(sample_3bytes: bytes, offset: int, endian: str = 'little') -> float:
    if endian == 'little':
        b0, b1, b2 = sample_3bytes[offset:offset+3]
        value = b0 | (b1 << 8) | ((b2 & 0xF0) << 12)
        if value >= 0x80000:  # sign extend
            value -= 0x100000
    else:
        b0, b1, b2 = sample_3bytes[offset:offset+3]
        value = ((b0 & 0xF0) << 8) | (b1 << 8) | b2
        if value >= 0x80000:
            value -= 0x100000
    return value / 524288.0
```

**Practical use:** 20-bit PCM is natively supported in some professional formats (AES3, also known as S/PDIF at professional levels). The AES3 standard allows both 20-bit and 24-bit payloads. In the WAV format, 20-bit PCM is supported through the extensible format chunk, where `wBitsPerSample = 20` and `wValidBitsPerSample = 20`.

### 4.5 24-Bit PCM

24-bit PCM is the current standard for professional audio recording, mixing, and mastering. The theoretical dynamic range of 24-bit is 144.5 dB — far beyond any practical converter's noise floor. The extra bits in 24-bit are used primarily for headroom during mixing and signal processing, not for capturing additional audio detail.

**Byte layout for 24-bit signed stereo (interleaved, little-endian):**

```
Byte 0: Bits 0-7 (LSB) of Sample 0, Channel 0
Byte 1: Bits 8-15 of Sample 0, Channel 0
Byte 2: Bits 16-23 (MSB) of Sample 0, Channel 0
Byte 3: Bits 0-7 (LSB) of Sample 0, Channel 1
Byte 4: Bits 8-15 of Sample 0, Channel 1
Byte 5: Bits 16-23 (MSB) of Sample 0, Channel 1
...
```

**Range:** −8,388,608 to +8,388,607.

**Conversion:**

```python
def int24_to_int32(sample_3bytes: bytes, offset: int) -> int:
    """Convert 3-byte signed 24-bit PCM to signed 32-bit integer."""
    b0, b1, b2 = sample_3bytes[offset:offset+3]
    value = b0 | (b1 << 8) | (b2 << 16)
    # Sign extend if negative (bit 23 set)
    if value >= 0x800000:
        value -= 0x1000000
    return value

def int32_to_int24(value: int) -> bytes:
    """Convert signed 32-bit integer to 3-byte signed 24-bit PCM."""
    # Clamp to 24-bit range
    if value > 8388607:
        value = 8388607
    elif value < -8388608:
        value = -8388608
    # Convert to unsigned 24-bit for packing
    if value < 0:
        value += 0x1000000
    return bytes([value & 0xFF, (value >> 8) & 0xFF, (value >> 16) & 0xFF])

def int24_to_float(sample: int) -> float:
    return sample / 8388608.0

def float_to_int24(sample: float) -> int:
    if sample >= 1.0:
        return 8388607
    if sample <= -1.0:
        return -8388608
    return int(round(sample * 8388608.0))
```

**Real-world dynamic range:** No audio converter achieves 144 dB of dynamic range. The best converters (e.g., ESS Sabre, AK4499, AKM AK4497) achieve approximately 130–133 dB of dynamic range and −110 to −120 dBFS THD+N. The practical conclusion: 24-bit recording provides ample headroom for mixing and processing. The SNR of the recording environment (acoustic noise, microphone self-noise, room acoustics) typically limits the effective dynamic range to 80–100 dB for most real-world recordings.

### 4.6 32-Bit Integer PCM

32-bit signed integer PCM is used in some professional formats but is relatively uncommon. Its theoretical dynamic range is 192.7 dB, which exceeds any practical use case. 32-bit integer is more commonly encountered as an intermediate format in fixed-point DSP processing (e.g., 32-bit accumulator for summing samples).

**Range:** −2,147,483,648 to +2,147,483,647.

**C code for 32-bit PCM read/write:**

```c
#include <stdint.h>
#include <string.h>

// Read 32-bit signed little-endian PCM from byte buffer
int32_t read_int32_le(const uint8_t* buffer) {
    return (int32_t)(buffer[0] | (buffer[1] << 8) |
                     (buffer[2] << 16) | (buffer[3] << 24));
}

// Write 32-bit signed little-endian PCM to byte buffer
void write_int32_le(uint8_t* buffer, int32_t value) {
    buffer[0] = value & 0xFF;
    buffer[1] = (value >> 8) & 0xFF;
    buffer[2] = (value >> 16) & 0xFF;
    buffer[3] = (value >> 24) & 0xFF;
}

// Read 32-bit signed big-endian PCM from byte buffer
int32_t read_int32_be(const uint8_t* buffer) {
    return (int32_t)((buffer[0] << 24) | (buffer[1] << 16) |
                     (buffer[2] << 8) | buffer[3]);
}

// Write 32-bit signed big-endian PCM to byte buffer
void write_int32_be(uint8_t* buffer, int32_t value) {
    buffer[0] = (value >> 24) & 0xFF;
    buffer[1] = (value >> 16) & 0xFF;
    buffer[2] = (value >> 8) & 0xFF;
    buffer[3] = value & 0xFF;
}
```

### 4.7 32-Bit Float PCM (IEEE 754)

32-bit float is the most important bit depth for audio processing because it is the native format of modern DAWs and audio DSP libraries. FFmpeg, libraries like libsoundio, and hardware interfaces like ASIO all work natively in 32-bit float.

**IEEE 754 binary32 layout:**

```
31 (MSB)                                     0 (LSB)
+-----------+-------------------+-------------------------+
| S (1 bit) |  E (8 bits)       |  M (23 bits)            |
+-----------+-------------------+-------------------------+
```

Value = (−1)^S × 2^(E − 127) × 1.M (where 1.M is the 24-bit mantissa with implicit leading 1)

**Special values:**
- S = 0, E = 0, M = 0: +0.0
- S = 1, E = 0, M = 0: −0.0
- S = any, E = 0, M != 0: Denormalized number (very small)
- S = any, E = 255, M = 0: +/− Infinity
- S = any, E = 255, M != 0: NaN (Not a Number)

**For audio:** Values outside ±1.0 represent signal above full-scale. These should be clipped or limited before converting to integer PCM. Many DAWs use ±1.0 as the nominal ceiling and apply soft clipping or limiting when processing requires it.

```c
#include <stdint.h>
#include <string.h>
#include <math.h>

// Read IEEE 754 float from byte buffer (big-endian)
float read_float_be(const uint8_t* buffer) {
    uint32_t bits = (buffer[0] << 24) | (buffer[1] << 16) |
                    (buffer[2] << 8) | buffer[3];
    float result;
    memcpy(&result, &bits, sizeof(float));
    return result;
}

// Write IEEE 754 float to byte buffer (big-endian)
void write_float_be(uint8_t* buffer, float value) {
    uint32_t bits;
    memcpy(&bits, &value, sizeof(float));
    buffer[0] = (bits >> 24) & 0xFF;
    buffer[1] = (bits >> 16) & 0xFF;
    buffer[2] = (bits >> 8) & 0xFF;
    buffer[3] = bits & 0xFF;
}

// Clamp float to [-1.0, +1.0] and convert to 16-bit integer
int16_t float_to_int16_safe(float value) {
    if (isnan(value)) return 0;
    if (value >= 1.0f) return 32767;
    if (value <= -1.0f) return -32768;
    // Round to nearest, ties to even
    return (int16_t)lrintf(value * 32768.0f);
}
```

### 4.8 Bit Depth Conversion (Requantization)

When converting between bit depths, the key operation is requantization. The correct approach depends on whether you are reducing (going from more bits to fewer) or expanding (going from fewer to more) the bit depth.

**Upsampling (e.g., 16-bit → 24-bit):** Zero-fill the lower bits. This is the correct approach because the original 16-bit values already have quantization error — filling in zeros does not introduce new information, and it preserves the exact signal amplitude.

```python
def upsample_16_to_24(int16_value: int) -> int:
    """Zero-fill lower 8 bits to convert 16-bit to 24-bit."""
    return int16_value << 8
```

**Downsampling (e.g., 24-bit → 16-bit):** Requires dithering (adding low-level noise before truncation) to linearize the quantization error and prevent it from becoming correlated with the signal. Simply truncating the lower bits produces harmonic distortion, especially on quiet signals.

```python
import random

def dither_and_truncate_24_to_16(float_sample: float) -> int:
    """
    Convert float to 16-bit with triangular dither.
    Triangular dither (PDF = 1 for values in [-1, +1]) has a
    triangular probability density function, which shapes
    the quantization noise spectrum to be less perceptible.
    """
    # Triangular dither: sum of two uniform random values
    dither = random.uniform(-1.0, 1.0) + random.uniform(-1.0, 1.0)
    dithered = float_sample + (dither / 65536.0)
    return float_to_int16(dithered)
```

The dither amplitude for 16-bit output should be approximately 1 LSB (1/65536 of full-scale). The triangular PDF (sum of two uniform random values) distributes the quantization noise energy across a wider frequency range, making it perceptually less obtrusive than simple noise dither.

---

## 5. SAMPLE RATE DETAILED SPECIFICATION

### 5.1 The Nyquist-Shannon Sampling Theorem

The Nyquist-Shannon sampling theorem, established mathematically by Harry Nyquist (1928) and conclusively proven by Claude Shannon (1949), states:

> A band-limited continuous signal can be perfectly reconstructed from its samples if and only if the sample rate is greater than twice the highest frequency component in the signal.

The Nyquist frequency is `F_nyquist = sample_rate / 2`. A signal with components up to `F_max` can be perfectly reconstructed if `sample_rate > 2 × F_max`.

**Why twice?** When a continuous signal is sampled, its frequency spectrum becomes periodic. The original baseband spectrum appears at its original position, and copies (called images or aliases) appear centered at integer multiples of the sample rate. If the signal contains energy at or above the Nyquist frequency, these spectral copies overlap, causing irreversible aliasing.

**Aliasing formula:**

```
f_alias = |f_signal - k × sample_rate|
```

where k is chosen such that `f_alias` falls within `[0, sample_rate/2]`.

For example, with sample_rate = 44100 Hz, a 30 kHz tone would alias to:

```
|30000 - 44100| = 14100 Hz
```

The 30 kHz signal would be heard as a 14.1 kHz tone — a completely wrong frequency. This is aliasing, and it cannot be undone.

### 5.2 Standard Sample Rates

| Sample Rate | Nyquist | Primary Use | Historical Context |
|---|---|---|---|
| 8000 Hz | 4.0 kHz | Telephony, VOIP | Minimum for voice intelligibility (4 kHz bandwidth) |
| 11025 Hz | 5.5 kHz | Early multimedia | 1/4 of CD rate; used in early sound cards |
| 16000 Hz | 8.0 kHz | VoIP, speech | Wideband speech; good for voice over 8 kHz |
| 22050 Hz | 11.0 kHz | Limited bandwidth audio | 1/2 of CD rate; acceptable for low-fi audio |
| 44100 Hz | 22.05 kHz | CD audio, general audio | Chosen to accommodate 48 kHz and NTSC color burst |
| 48000 Hz | 24.0 kHz | DVD, professional audio | Standard for video post-production |
| 88200 Hz | 44.1 kHz | CD remastering, hi-res | 2× CD rate |
| 96000 Hz | 48.0 kHz | DVD-Audio, hi-res, video | 2× standard rate |
| 176400 Hz | 88.2 kHz | Studio hi-res | 4× CD rate |
| 192000 Hz | 96.0 kHz | Professional hi-res | 4× standard rate; widely supported |
| 352800 Hz | 176.4 kHz | DXD (Digital eXtreme Definition) | 8× CD rate; used in DSD workflow |
| 384000 Hz | 192.0 kHz | Maximum hi-res standard | 8× standard rate; rare hardware support |

### 5.3 The 44.1 kHz Puzzle

The 44.1 kHz sample rate used for CDs is not a round number and warrants explanation. The CD specification was finalized in 1980, and the engineering team at Sony needed to accommodate two requirements:

1. **48 kHz compatibility:** Professional video equipment used 48 kHz as its standard sample rate (derived from the 1.536 MHz color subcarrier for NTSC: 1.536 MHz / 32 = 48,000 Hz).
2. **NTSC color burst compatibility:** The NTSC color subcarrier is 3.579545 MHz. The earliest digital audio recorders used video tape as a storage medium (the Sony PCM-1600 used a Betamax VCR).

The 44.1 kHz rate is derived as follows:

```
525 lines × 30 frames/sec = 15750 lines/sec (NTSC field rate)
15750 lines/sec × 588 samples/line = 9,261,000 (NTSC samples/sec)
9,261,000 / 3 × 14 = 44,100 Hz
```

Or alternatively, from the PAL system:

```
625 lines × 25 frames/sec = 15625 lines/sec (PAL field rate)
15625 × 588 = 9,187,500 / 3 × 14 = 44,100 Hz
```

The resulting 44.1 kHz accommodates both the NTSC and PAL video systems, which use 525/30 and 625/25 line/frame combinations respectively. Both systems produce approximately the same line rate × samples per line product that yields 44.1 kHz.

### 5.4 Anti-Aliasing Filters

Before analog audio is sampled (A-to-D conversion), an anti-aliasing filter must remove all frequency components above the Nyquist frequency. This filter is typically a steep low-pass filter with a cutoff at or slightly below the Nyquist frequency.

**Filter requirements:**

- Passband: flat response up to the desired audio bandwidth (e.g., 20 kHz)
- Transition band: rapid rolloff from the highest desired frequency to the Nyquist frequency
- Stopband: sufficient attenuation at and above the Nyquist frequency to prevent aliasing

A practical anti-aliasing filter for CD audio (44.1 kHz, Nyquist = 22.05 kHz) might have:
- Passband: 0 to 20 kHz (±0.1 dB ripple)
- Transition band: 20 kHz to 22.05 kHz
- Stopband: 22.05 kHz to infinity, with >80 dB attenuation

Modern sigma-delta ADC chips (such as those from ESS, AK, or Cirrus Logic) use oversampling techniques where the audio is initially sampled at a very high rate (e.g., 64× or 128× oversampling, producing a first-stage sample rate of 2.8224 MHz or 5.6448 MHz for CD audio). A digital decimation filter then downsamples to the target rate. This approach moves the anti-aliasing requirement into the digital domain, where perfect brick-wall filters can be implemented with linear-phase characteristics.

### 5.5 Reconstruction Filters (D-to-A Conversion)

After D-to-A conversion, the output contains not only the desired baseband signal but also spectral images at multiples of the sample rate. A reconstruction (anti-imaging) filter removes these images, producing a smooth analog output.

The reconstruction filter's job is the inverse of the anti-aliasing filter. In an ideal world, the output would be a perfect reconstruction of the original continuous signal, but practical filters have finite rolloff.

**ZOH (Zero-Order Hold) effect:** The simplest D-to-A conversion (holding each sample value constant until the next sample) produces a stair-step waveform. This is equivalent to convolving the ideal impulse response with a rectangular pulse, which multiplies the frequency spectrum by `sinc(f)` — attenuating high frequencies and introducing a gradual rolloff.

```
sinc(f) = sin(π × f / Fs) / (π × f / Fs)
```

At f = 0, sinc = 1. At f = Fs/2 (Nyquist), sinc ≈ 0.637 (about −3.92 dB). This means a simple zero-order hold DAC would attenuate the highest audio frequencies by nearly 4 dB, which is unacceptable.

Modern DACs use oversampling with multi-stage reconstruction filters:
1. The input PCM samples are upsampled (typically 8×) by inserting zeros between samples
2. A digital interpolation filter removes the spectral images
3. A multi-bit sigma-delta modulator produces a high-frequency pulse-density modulated bitstream
4. An analog low-pass filter removes the high-frequency modulation component

The result is a smooth analog output with minimal phase distortion and flat frequency response.

### 5.6 Sample Rate Conversion (SRC)

Sample rate conversion (SRC) is the process of changing the sample rate of a digital audio signal. It is one of the most critical and frequently misimplemented operations in audio conversion pipelines. There are two fundamental approaches:

#### 5.6.1 Integer Ratio SRC

When the ratio between input and output sample rates is an exact integer (e.g., 44100 → 88200, which is 2×), the process is called interpolation (for upsampling) or decimation (for downsampling).

**Upsampling by integer L:**
1. Insert (L−1) zero samples between every input sample
2. Apply a low-pass filter with cutoff at π/L radians (in normalized frequency) to remove spectral images

**Downsampling by integer M:**
1. Apply a low-pass filter with cutoff at π/M radians to prevent aliasing
2. Keep every M-th sample

The low-pass filter is the critical component. A poor filter will introduce aliasing (during downsampling) or allow imaging (during upsampling), and may introduce phase distortion or amplitude ripple in the passband.

#### 5.6.2 Rational Ratio SRC

When the ratio is a fraction (e.g., 44100 → 48000, which is 160/147), a polyphase filter bank is used:

1. Compute the least common multiple of both sample rates: LCM(44100, 48000) = 70560000 Hz
2. Upsample input to the LCM rate (70560000 / 44100 = 1600×)
3. Downsample to output rate (70560000 / 48000 = 1470×)

In practice, this is implemented as a polyphase filter with `P` phases (where P is typically 64 or more), using the formula:

```
output[n] = Σ(k=0 to N-1) input[polyphase_index(n, k)] × filter_coefficient[k]
```

where the polyphase index selects the appropriate phase of the filter for each output sample.

#### 5.6.3 Asynchronous SRC (ASRC)

For asynchronous sample rate conversion (where the input and output clocks are unrelated, as in S/PDIF connections between devices with different clocks), a sample rate converter must dynamically adapt to the continuously changing ratio. This requires:

1. A buffer to absorb timing differences between input and output clocks
2. Adaptive interpolation that adjusts the effective sample rate ratio in real time
3. Buffer management to prevent underrun or overflow

High-quality ASRC algorithms (used in devices like the Analog Devices AD1896 or the SRC4382) achieve THD+N better than −140 dB and flat frequency response within the audio band. The best ASRC algorithms use polynomial interpolation (Lagrange or B-spline) combined with high-quality low-pass filtering.

#### 5.6.4 SRC Quality Concerns

Poor SRC implementation manifests as:

| Issue | Symptom | Cause |
|---|---|---|
| Aliasing | High-frequency images appear at wrong frequencies | Insufficient stopband attenuation in decimation filter |
| Imaging | Spectral images in upsampled signal | Missing or weak anti-imaging filter |
| Passband ripple | Uneven frequency response | Poorly designed filter with ripple in passband |
| Phase distortion | Time-domain smearing | Non-linear phase filter |
| Group delay variation | Frequency-dependent delay | Same as above |
| Transients | Pre-echo or ringing on transients | Excessively long filter kernel |

FFmpeg's libswresample provides high-quality SRC using polyphase filter banks with up to 64 phases. The quality settings range from 0 (highest quality, most computationally expensive) to 9 (lowest quality, fastest). For professional work, use quality 0 or 1.

```bash
# Sample rate conversion with FFmpeg
ffmpeg -i input.wav -ar 48000 output.wav  # 44.1 → 48 kHz
ffmpeg -i input.flac -ar 96000 output.wav  # 44.1 → 96 kHz
ffmpeg -i input.wav -ar 44100 -af aresample=resampler=soxr output.wav  # SoX resampler
```

---

## 6. CHANNEL CONFIGURATIONS

### 6.1 Standard Channel Layouts

Audio channel layouts define which speaker positions are represented by which sample channels in the interleaved stream. Common layouts:

| Channels | Layout Name | Channel Order |
|---|---|---|
| 1 | Mono | C |
| 2 | Stereo | L, R |
| 2.1 | Stereo + LFE | L, R, LFE |
| 4 | Quad | FL, FR, RL, RR |
| 5 | 5.0 | FL, FR, FC, LFE, RL, RR |
| 5.1 | Standard surround | FL, FR, FC, LFE, RL, RR |
| 6 | 5.1 (alternate) | FL, FR, RL, RR, FC, LFE (WAVE_EX ordering) |
| 7.1 | 7.1 surround | FL, FR, FC, LFE, RL, RR, SL, SR |

**WAVE_FORMAT_EXTENSIBLE channel mask bit positions:**

| Bit | Position | Channel |
|---|---|---|
| 0 | Front Left | FL |
| 1 | Front Right | FR |
| 2 | Front Center | FC |
| 3 | LFE (Low Frequency Effects) | LFE |
| 4 | Back Left | BL |
| 5 | Back Right | BR |
| 6 | Front Left of Center | FLC |
| 7 | Front Right of Center | FRC |
| 8 | Back Center | BC |
| 9 | Side Left | SL |
| 10 | Side Right | SR |
| 11 | Top Center | TC |
| 12 | Top Front Left | TFL |
| 13 | Top Front Center | TFC |
| 14 | Top Front Right | TFR |
| 15 | Top Back Left | TBL |
| 16 | Top Back Center | TBC |
| 17 | Top Back Right | TBR |

The channel mask is a 32-bit unsigned integer where each bit represents a speaker position.

### 6.2 Stereo and Channel Pairing

For stereo PCM, the convention is:

**WAV/RIFF:** Left channel first, then Right channel (L, R order in interleaved stream).

**AIFF:** Left channel first, then Right channel (same as WAV, but big-endian).

**S/PDIF (consumer):** Left channel first, Right channel second.

**MPEG audio (MP3, AAC):** Left channel first, then Right channel. Joint stereo modes encode left+right as mid (M = (L+R)/2) and side (S = (L−R)/2).

When converting between mono and stereo:

```python
def stereo_to_mono(left: int, right: int, bit_depth: int) -> int:
    """Mix stereo to mono using equal gain."""
    return (left + right) // 2  # For integer PCM; exact formula depends on bit depth

def mono_to_stereo(mono: int, bit_depth: int) -> tuple[int, int]:
    """Duplicate mono to both channels."""
    return (mono, mono)

def split_stereo_to_interleaved(stereo_tuple: tuple[bytes, bytes]) -> bytes:
    """Convert separate channel buffers to interleaved."""
    left_buf, right_buf = stereo_tuple
    interleaved = bytearray(len(left_buf) * 2)
    for i in range(len(left_buf) // 2):  # assuming 16-bit
        interleaved[i * 4:i * 4 + 2] = left_buf[i * 2:i * 2 + 2]
        interleaved[i * 4 + 2:i * 4 + 4] = right_buf[i * 2:i * 2 + 2]
    return bytes(interleaved)
```

---

## 7. NORMALIZATION AND HEADROOM

### 7.1 Full-Scale Reference

In digital audio, **0 dBFS (decibels relative to full scale)** is the maximum representable level. All PCM values are expressed relative to this reference.

| dBFS | Linear Level | 16-bit Integer Value | Notes |
|---|---|---|---|
| 0 dBFS | 1.0 | 32767 | Maximum; no headroom |
| −1 dBFS | 0.891 | 29193 | 1 dB below max |
| −3 dBFS | 0.708 | 23170 | Quarter-power point |
| −6 dBFS | 0.501 | 16384 | Half-amplitude |
| −12 dBFS | 0.251 | 8192 | Common reference for unity |
| −20 dBFS | 0.100 | 3277 | 20 dB below max |
| −48 dBFS | 0.004 | 131 | 8-bit dynamic range equivalent |
| −90 dBFS | 0.0000316 | 1 | One LSB at 16-bit |
| −∞ dBFS | 0.0 | 0 | Silence |

**Critical rule:** In PCM conversion pipelines, never allow values to exceed the maximum representable level without explicit clipping. Clipping produces hard distortion that is far more perceptually objectionable than gentle gain reduction.

### 7.2 Headroom Strategy

When mixing multiple PCM channels or applying gain in a floating-point pipeline:

1. **Plan for headroom:** When mixing N signals, the combined peak could theoretically be N times the individual peaks. In practice, coherent signals (identical content) can sum to full-scale when each is at 0 dBFS. Reserve at least 1 dB of headroom.

2. **Measure true peaks:** Peak meters should measure true peaks (maximum absolute sample value), not RMS. For loudness normalization (EBU R128, ReplayGain), use integrated loudness (LUFS) measurement.

3. **Apply limiter:** Before converting float to integer PCM for output, apply a limiter that prevents any sample from exceeding ±1.0. The limiter can be:
   - Hard clip: `output = min(max(input, -1.0), +1.0)` — introduces distortion
   - Soft clip: applies a smooth non-linear function near the threshold
   - Look-ahead limiter: finds peaks in advance and applies gain reduction

```python
def apply_hard_clip(value: float) -> float:
    if value > 1.0:
        return 1.0
    if value < -1.0:
        return -1.0
    return value

def apply_soft_clip(value: float, threshold: float = 0.9) -> float:
    """Tanh soft clipper — smooth limiting without hard clipping."""
    if abs(value) <= threshold:
        return value
    # Tanh soft limiting above threshold
    if value > threshold:
        return threshold + (1.0 - threshold) * math.tanh((value - threshold) / (1.0 - threshold))
    else:
        return -threshold - (1.0 - threshold) * math.tanh((-value - threshold) / (1.0 - threshold))
```

---

## 8. DITHERING

### 8.1 Why Dither Is Necessary

When reducing bit depth (e.g., 24-bit → 16-bit), the quantization step size of the output is larger than the resolution of the input. Without dithering, the quantization error is deterministic and correlated with the signal — particularly problematic for low-level signals where the error can be a significant fraction of the signal itself.

Consider a sine wave at −60 dBFS (approximately 0.001 peak amplitude). At 16-bit, the LSB size is 1/32768 ≈ 0.0000305. The sine wave occupies only about 33 code points, and the quantization error between the true sine value and the nearest quantized value creates harmonic distortion.

**Dithering** solves this by adding a low-level random signal (dither noise) to the signal before quantization. The dither randomizes the quantization error, making it spectrally white noise instead of signal-correlated distortion. For a properly shaped dither (triangular PDF), the quantization noise floor becomes flat at the theoretical level of −90.3 dBFS for 16-bit PCM.

### 8.2 Dither PDF Types

| Dither Type | PDF | Noise Floor | Spectral Shape | Notes |
|---|---|---|---|---|
| None (truncate) | N/A | Correlated distortion | N/A | Only acceptable for non-critical applications |
| Uniform | Flat [−0.5, +0.5] LSB | −3 dB below truncation | White | Slightly better than none |
| Triangular | Two uniform samples summed | −4.8 dB below truncation | White | Standard for audio; flat noise floor |
| Gaussian | Normal distribution | Depends on σ | White | Good but requires RNG |
| High-pass | HPF applied to triangular | Shaped | Shaped | Psychoacoustically optimized |
| TPDF (TPDF + noise shaping) | Triangular | Shaped | Shaped | Industry standard |

### 8.3 Triangular Probability Density Function (TPDF) Dither

TPDF dither is the standard for professional audio. It is produced by summing two independent uniform random values, each in [−0.5, +0.5] LSB:

```python
import random

def generate_tpdf_dither_16bit() -> int:
    """Generate one triangular dither sample for 16-bit output."""
    d1 = random.uniform(-0.5, 0.5)
    d2 = random.uniform(-0.5, 0.5)
    return d1 + d2  # Range: [-1.0, +1.0] LSB

def dither_24_to_16(input_float: float, bit_depth: int = 16) -> int:
    """Apply TPDF dither and quantize to target bit depth."""
    lsb_size = 2.0 ** (1 - bit_depth)
    dither = generate_tpdf_dither_16bit() * lsb_size
    dithered = input_float + dither
    quantized = round(dithered / lsb_size) * lsb_size
    return int(quantized / lsb_size)
```

### 8.4 Noise Shaping

Noise shaping goes beyond flat dither by applying a filter to the quantization error, pushing it into frequency regions where the ear is less sensitive (typically the high frequencies, above 8–10 kHz).

The fundamental noise shaping feedback equation:

```
y[n] = x[n] + (x[n] - x_hat[n]) * H_shaping(z)
```

where `x_hat[n]` is the quantized output, `H_shaping(z)` is the shaping filter, and `y[n]` is the shaped error.

Simple first-order noise shaper (1-pole highshelf):

```
error_shaped[n] = error[n] - g × error[n-1]
```

where `g` is the high-frequency gain (>0 pushes noise to high frequencies).

Modern high-order noise shapers (e.g., in sigma-delta modulators) can push quantization noise above the audible range entirely, achieving >120 dB SNR in the audio band with only 1-bit output (as in DSD).

**Psychoacoustic noise shaping:** The ear's sensitivity to noise varies with frequency (approximately following the equal-loudness contour). Shaping noise to follow the inverse of the equal-loudness contour can make the same amount of noise perceptually less audible. This is the principle behind psychoacoustic-optimized dither.

---

## 9. PCM IN AUDIO CONTAINERS

### 9.1 PCM in WAV/RIFF

The WAV format (RIFF container with WAVE form type) is the most common container for PCM audio on Windows. It supports PCM at 8, 16, 20, 24, 32-bit integer, and 32/64-bit float.

The format chunk (`fmt `) specifies the PCM encoding parameters. For standard PCM, `wFormatTag = 0x0001`. For IEEE float, `wFormatTag = 0x0003`.

**WAV header for 44.1 kHz stereo 16-bit PCM:**
```
Offset  Value   Description
0x00    "RIFF"  Chunk ID
0x04    N+36    Chunk size (file size - 8)
0x08    "WAVE"  Form type
0x0C    "fmt "  Format chunk ID
0x10    16      Format chunk size (16 for PCM)
0x14    1       Audio format (PCM = 1)
0x16    2       Number of channels
0x18    44100   Sample rate
0x1C    176400  Bytes per second
0x20    4       Block align (channels × bytes per sample)
0x22    16      Bits per sample
0x24    "data"  Data chunk ID
0x28    N       Data chunk size (audio bytes)
0x2C    [audio data...]
```

### 9.2 PCM in AIFF

AIFF (Audio Interchange File Format) uses big-endian byte order throughout. The container structure consists of chunks, similar to RIFF but with different endianness.

**AIFF header for 44.1 kHz stereo 16-bit PCM:**
```
Offset  Value   Description
0x00    "FORM"  Chunk ID
0x04    N+46    File size - 8
0x08    "AIFF"  Form type
0x0C    "COMM"  Common chunk ID
0x10    18      Common chunk size
0x14    2       Number of channels
0x16    N        Number of sample frames (5 bytes, 80-bit integer)
0x1B    16       Bits per sample
0x1D    8E      Sample rate (80-bit IEEE 754 extended)
0x26    "SSND"  Sound data chunk ID
0x2A    N+8     Sound chunk size
0x2E    0       Offset (unused, set to 0)
0x32    0       Block size (unused, set to 0)
0x36    [audio data...]
```

AIFF uses 80-bit IEEE 754 extended precision for sample rate encoding (10 bytes). The sample rate value `8E` in the example is the hexadecimal encoding of 44100.0 in 80-bit format.

### 9.3 PCM in CAF

CAF (Core Audio Format) is Apple's professional container format. It supports arbitrarily large files (>4 GB), 32-bit float PCM, and 64-bit float PCM natively.

**CAF chunk types relevant to PCM:**
- `desc`: Audio description (format chunk)
- `data`: Audio sample data
- `uuid`: User-defined chunk type

### 9.4 PCM in Raw Files

Raw PCM files have no header — they contain only the audio sample data. The format parameters (sample rate, bit depth, channels, endianness) must be known from external metadata. This format is used in:

- FFmpeg pipe input/output (`-f f32le`, `-f s16le`)
- Embedded systems with fixed format
- Analysis tools that extract raw PCM from container files

FFmpeg sample format names for raw PCM:
```
s8    — signed 8-bit
s16_le — signed 16-bit little-endian
s16_be — signed 16-bit big-endian
s24_le — signed 24-bit little-endian
s32_le — signed 32-bit little-endian
f32_le — 32-bit float little-endian
f64_le — 64-bit float little-endian
```

---

## 10. CLOCKING, JITTER, AND ACCURACY

### 10.1 Clock Jitter

Clock jitter is the variation in the timing of sample instants from the ideal periodic schedule. In A-to-D and D-to-A conversion, jitter causes sample timing errors that translate into amplitude errors.

**The jitter-signal relationship:**

For a sinusoidal input `x(t) = A × sin(2πft)`, the sampled value at time `t + δ(t)` is approximately:

```
x_sampled ≈ A × sin(2πft + 2πf × δ(t))
```

For small jitter `δ(t)`, using `sin(a + b) ≈ sin(a) + b × cos(a)`:

```
x_sampled ≈ A × sin(2πft) + A × 2πf × δ(t) × cos(2πft)
```

The jitter-induced error is proportional to:
- The signal frequency `f` — higher frequencies are more susceptible
- The jitter amplitude `δ(t)`
- The signal amplitude `A`

**Jitter budget:** For a CD-quality system (44.1 kHz, 20 kHz bandwidth), jitter below approximately 10–20 nanoseconds RMS produces negligible distortion (below −100 dBFS). Jitter above 100 ns RMS becomes audible as increased high-frequency noise and loss of stereo imaging.

### 10.2 Sample Clock Accuracy

The sample rate must be maintained within a tight tolerance. For CD audio, the standard requires ±1000 ppm (parts per million) accuracy and ±50 ppm stability over the duration of a recording. This means:

- At 44.1 kHz, the acceptable range is 44,100 ± 44.1 Hz (approximately ±0.004%)
- A recording at 44,105 Hz played back at 44,100 Hz would be 0.01% flat
- For a 3-minute track, this would cause approximately 18 ms of time drift

**PLL (Phase-Locked Loop):** Professional digital audio equipment uses PLLs to synchronize to incoming digital audio clocks. The PLL tracks the incoming sample rate and locks the local oscillator to it, minimizing jitter and frequency error. The quality of the PLL determines how well the receiver tracks the transmitter's clock.

### 10.3 Asynchronous Sample Rate Conversion

When two digital audio devices operate with independent, non-synchronized clocks (as in a consumer S/PDIF connection), the receiving device must perform asynchronous sample rate conversion. The ASRC algorithm estimates the instantaneous input sample rate, maintains a buffer to absorb rate differences, and resamples to the local output rate.

ASRC quality is characterized by:
- **THD+N:** Total harmonic distortion plus noise in the output
- **Passband flatness:** Variation in frequency response within the audio band
- **Stopband attenuation:** Attenuation of alias images above Nyquist
- **Group delay variation:** Phase linearity across frequency
- **Transient handling:** Preservation of transient timing and shape

---

## 11. INTEROPERABILITY AND PIPELINE CONSIDERATIONS

### 11.1 Universal PCM Buffer Interface

For a conversion pipeline that handles arbitrary bit depths, sample rates, and channel configurations, a canonical buffer structure is essential:

```python
@dataclass
class PCMBuffer:
    """Canonical PCM audio buffer representation."""
    sample_rate: int          # Samples per second (Hz)
    bit_depth: int            # Bits per sample (8, 16, 20, 24, 32, 64)
    channels: int             # Number of channels
    channel_layout: str      # Layout name: "mono", "stereo", "5.1", etc.
    is_float: bool            # True if floating-point encoding
    is_signed: bool          # True if signed (False = unsigned)
    is_interleaved: bool     # True = interleaved, False = planar
    endianness: str          # "little" or "big"
    samples_per_channel: int # Total samples per channel (duration × sample_rate)
    data: bytes              # Raw sample data

    @property
    def duration_seconds(self) -> float:
        return self.samples_per_channel / self.sample_rate

    @property
    def bytes_per_sample(self) -> int:
        return (self.bit_depth + 7) // 8

    @property
    def bytes_per_frame(self) -> int:
        return self.bytes_per_sample * self.channels

    @property
    def bytes_per_second(self) -> int:
        return self.bytes_per_frame * self.sample_rate

    @property
    def total_bytes(self) -> int:
        return self.samples_per_channel * self.bytes_per_frame
```

### 11.2 Bit-Exact Conversion Rules

For a conversion pipeline to produce bit-exact output across multiple runs, the following conditions must be met:

1. **Identical input data:** The same bytes must be read from the same positions.
2. **Identical processing parameters:** Sample rate, bit depth, dither setting, channel layout.
3. **Deterministic arithmetic:** Floating-point operations must produce identical results (use of FMA, rounding mode, and compiler optimization can affect this).
4. **No random number generators:** Any dithering must use a seeded PRNG with a fixed seed, or dither must be disabled entirely.
5. **Consistent platform:** Different CPUs may produce slightly different floating-point results due to extended precision registers (x87 vs. SSE).

For bit-exact verification:

```python
def verify_bit_exact(pcm_a: PCMBuffer, pcm_b: PCMBuffer) -> bool:
    """Verify two PCM buffers produce bit-identical output."""
    if (pcm_a.sample_rate != pcm_b.sample_rate or
        pcm_a.bit_depth != pcm_b.bit_depth or
        pcm_a.channels != pcm_b.channels or
        pcm_a.endianness != pcm_b.endianness or
        pcm_a.samples_per_channel != pcm_b.samples_per_channel):
        return False
    return pcm_a.data == pcm_b.data
```

### 11.3 Metadata Preservation

PCM itself carries no metadata (no tags, no channel labels, no encoding information). Metadata must be preserved separately from the audio data and reattached to the output container.

When converting between formats, the following metadata should be tracked:

| Metadata | Source | Importance |
|---|---|---|
| Sample rate | Format header / demuxer | Critical — must match actual sample rate |
| Bit depth | Format header / demuxer | Critical — determines quantization |
| Channel layout | Format header | Important — affects playback |
| Encoding type | Format header | Important — signed vs. unsigned, float vs. int |
| Peak amplitude | Analysis | Important — for normalization |
| Loudness (LUFS) | EBU R128 / ReplayGain analysis | Important — for loudness normalization |
| Encoder identity | Tag | Useful for debugging |

FFmpeg's `-map_metadata` option handles tag copying between containers, but the audio data itself must be processed separately with appropriate sample format conversion.

---

## 12. SIGNAL PROCESSING OPERATIONS ON PCM

### 12.1 Gain (Amplitude Scaling)

```python
def apply_gain(samples: list[float], gain_db: float) -> list[float]:
    """Apply gain in dB to a float PCM buffer."""
    gain_linear = 10 ** (gain_db / 20.0)
    return [s * gain_linear for s in samples]

def apply_gain_integer(samples: list[int], gain_db: float, bit_depth: int) -> list[int]:
    """Apply gain to integer PCM, returning clipped values."""
    gain_linear = 10 ** (gain_db / 20.0)
    max_val = (1 << (bit_depth - 1)) - 1
    min_val = -(1 << (bit_depth - 1))
    result = []
    for s in samples:
        scaled = int(round(s * gain_linear))
        result.append(max(min_val, min(max_val, scaled)))
    return result
```

### 12.2 Peak Normalization

```python
def find_peak(samples: list[float]) -> float:
    """Find the maximum absolute sample value."""
    return max(abs(s) for s in samples) if samples else 0.0

def normalize_to_peak(samples: list[float], target_db: float = 0.0) -> tuple[list[float], float]:
    """Normalize peak to target dBFS."""
    peak = find_peak(samples)
    if peak == 0:
        return samples, 0.0
    target_linear = 10 ** (target_db / 20.0)
    gain = target_linear / peak
    return [s * gain for s in samples], gain
```

### 12.3 Mixing (Adding PCM Signals)

```python
def mix_buffers(buf_a: list[float], buf_b: list[float]) -> list[float]:
    """Mix two equal-length PCM buffers with equal gain."""
    assert len(buf_a) == len(buf_b)
    return [a + b for a, b in zip(buf_a, buf_b)]

def mix_with_gain(buf_a: list[float], buf_b: list[float],
                  gain_a_db: float, gain_b_db: float) -> list[float]:
    """Mix two buffers with individual gain controls."""
    gain_a = 10 ** (gain_a_db / 20.0)
    gain_b = 10 ** (gain_b_db / 20.0)
    return [gain_a * a + gain_b * b for a, b in zip(buf_a, buf_b)]
```

### 12.4 Crossfade

```python
import math

def linear_crossfade(buf_a: list[float], buf_b: list[float],
                     fade_samples: int) -> list[float]:
    """Crossfade between two buffers using linear interpolation."""
    assert len(buf_a) == len(buf_b)
    result = list(buf_a)
    for i in range(fade_samples):
        weight_b = i / (fade_samples - 1) if fade_samples > 1 else 1.0
        weight_a = 1.0 - weight_b
        result[i] = weight_a * buf_a[i] + weight_b * buf_b[i]
    return result

def equal_power_crossfade(buf_a: list[float], buf_b: list[float],
                          fade_samples: int) -> list[float]:
    """Crossfade using equal-power law (constant perceived loudness)."""
    assert len(buf_a) == len(buf_b)
    result = list(buf_a)
    for i in range(fade_samples):
        t = i / (fade_samples - 1) if fade_samples > 1 else 1.0
        # Equal power: sin/cos for constant power
        weight_a = math.cos(t * math.pi / 2)
        weight_b = math.sin(t * math.pi / 2)
        result[i] = weight_a * buf_a[i] + weight_b * buf_b[i]
    return result
```

---

## 13. LOUDNESS AND NORMALIZATION

### 13.1 RMS vs. Peak vs. True Peak

| Measurement | Definition | Use Case |
|---|---|---|
| **Peak** | Maximum absolute sample value | Clipping detection, normalization |
| **RMS (Root Mean Square)** | √(mean of squared samples) | Loudness, perceived volume |
| **True Peak** | Peak including inter-sample peaks | EBU R128, streaming loudness |
| **Integrated Loudness (LUFS)** | Loudness over entire program | Standardized loudness measurement |

**RMS formula:**

```
RMS = sqrt(1/N × Σ(i=1 to N) x[i]^2)
Loudness_dBFS = 20 × log10(RMS)
```

**True Peak:** Measured by oversampling the signal (typically 4×) before measuring peaks, to capture inter-sample peaks that can exceed the PCM sample values and cause clipping in D-to-A conversion.

### 13.2 ReplayGain 1.0

ReplayGain 1.0 applies gain to normalize perceived loudness to a reference level of 89 dB SPL (approximately equivalent to −18 dBFS in the digital domain for typical program material).

**ReplayGain calculation:**

```
pre_gain = -18 - integrated_loudness_dBFS
peak = max(abs(sample) for sample in program)
peak_gain = -1.0 - peak  # prevent clipping
applied_gain = min(pre_gain, peak_gain)
```

FFmpeg calculates and applies ReplayGain:

```bash
# Calculate ReplayGain
ffmpeg -i input.wav -af ebur128=peak=true -f null -

# Apply ReplayGain during playback/transcoding
ffmpeg -i input.wav -af replaygain=track=1 output.wav
```

### 13.3 EBU R128 (Loudness Normalization)

EBU R128 is the European broadcast standard for loudness normalization, replacing peak normalization with integrated loudness measurement.

**Key parameters:**
- Integrated loudness target: −23 LUFS
- Maximum true peak: −1 dBTP
- Loudness range (LRA): maximum 20 LU (for compliance)

**FFmpeg EBU R128 filter:**

```bash
# Measure loudness
ffmpeg -i input.wav -af loudnorm=print_format=json -f null -

# Apply loudness normalization
ffmpeg -i input.wav -af loudnorm=I=-16:TP=-1.5:LRA=11 output.wav
```

---

## 14. FORMATS AND SPECIFICATIONS REFERENCE

### 14.1 PCM Format Quick Reference

| Format | Extension | Bit Depths | Sample Rates | Channels | Endianness | Notes |
|---|---|---|---|---|---|---|
| WAV (PCM) | .wav | 8, 16, 20, 24, 32 | Any | Any | LE | RIFF container |
| AIFF (PCM) | .aif, .aiff | 8, 16, 20, 24 | Any | Any | BE | Apple format |
| CAF | .caf | 8–64 | Any | Any | BE/LE | Apple professional |
| RF64 | .wav | 16–64 | Any | Any | LE | WAV extension >4GB |
| BW64 | .wav | 16–64 | Any | Any | LE | BWF extension >4GB |
| AU | .au | 8, 16, 24, 32 | Any | Any | BE | Sun/NeXT format |
| AIFF-C | .aifc | 8–32 float | Any | Any | BE | Compressed AIFF |
| S24 | .s24, .raw | 24 | Any | Any | LE/BE | Raw 24-bit |
| S32 | .s32, .raw | 32 | Any | Any | LE/BE | Raw 32-bit |

### 14.2 Bit Depth Selection Guidelines

| Application | Recommended Bit Depth | Rationale |
|---|---|---|
| Telephony / VOIP | 8-bit μ-law | Sufficient for voice; minimizes bandwidth |
| Music streaming (MP3/AAC) | 16-bit source | CD quality; 16-bit input is the source standard |
| CD Audio | 16-bit | Format specification |
| DVD-Video | 16 or 24-bit | 48 kHz, up to 5.1 channels |
| DVD-Audio | 16, 20, 24-bit | Up to 192 kHz, MLP lossless |
| Blu-ray Audio | 16, 24-bit | Up to 192 kHz, Meridian Lossless Packing |
| Professional recording | 24-bit | Industry standard; headroom for processing |
| Mixing / mastering | 32-bit float | DAW internal format; no headroom concerns |
| Archival | 24-bit minimum | Preserve maximum detail; 32-bit preferred |
| Hi-Res delivery | 24 or 32-bit float | 96/192 kHz at 24-bit minimum |

---

## 15. PYTHON IMPLEMENTATION EXAMPLES

### 15.1 Complete PCM Reader

```python
import struct
from dataclasses import dataclass

@dataclass
class WAVHeader:
    channels: int
    sample_rate: int
    bits_per_sample: int
    byte_rate: int
    block_align: int
    data_size: int
    data_offset: int

def read_wav_header(filepath: str) -> WAVHeader:
    """Parse WAV file header and return format information."""
    with open(filepath, 'rb') as f:
        # Verify RIFF header
        riff = f.read(4)
        assert riff == b'RIFF', f"Expected RIFF, got {riff!r}"
        
        file_size = struct.unpack('<I', f.read(4))[0]
        wave = f.read(4)
        assert wave == b'WAVE', f"Expected WAVE, got {wave!r}"
        
        # Find fmt chunk
        while True:
            chunk_id = f.read(4)
            if not chunk_id:
                raise ValueError("No fmt chunk found")
            chunk_size = struct.unpack('<I', f.read(4))[0]
            
            if chunk_id == b'fmt ':
                fmt_data = f.read(chunk_size)
                # Parse fmt chunk
                audio_format = struct.unpack('<H', fmt_data[0:2])[0]
                channels = struct.unpack('<H', fmt_data[2:4])[0]
                sample_rate = struct.unpack('<I', fmt_data[4:8])[0]
                byte_rate = struct.unpack('<I', fmt_data[8:12])[0]
                block_align = struct.unpack('<H', fmt_data[12:14])[0]
                bits_per_sample = struct.unpack('<H', fmt_data[14:16])[0]
                assert audio_format == 1, f"Only PCM supported, got format {audio_format}"
                break
            else:
                f.seek(chunk_size, 1)  # Skip this chunk
        
        # Find data chunk
        while True:
            chunk_id = f.read(4)
            if not chunk_id:
                raise ValueError("No data chunk found")
            chunk_size = struct.unpack('<I', f.read(4))[0]
            
            if chunk_id == b'data':
                data_offset = f.tell()
                return WAVHeader(
                    channels=channels,
                    sample_rate=sample_rate,
                    bits_per_sample=bits_per_sample,
                    byte_rate=byte_rate,
                    block_align=block_align,
                    data_size=chunk_size,
                    data_offset=data_offset
                )
            else:
                f.seek(chunk_size, 1)
    
    raise ValueError("No data chunk found")

def read_pcm_samples(filepath: str, max_samples: int | None = None) -> tuple[WAVHeader, bytes]:
    """Read WAV file and return header + raw PCM data."""
    header = read_wav_header(filepath)
    
    with open(filepath, 'rb') as f:
        f.seek(header.data_offset)
        bytes_to_read = header.data_size
        if max_samples is not None:
            bytes_to_read = min(bytes_to_read, max_samples * header.block_align)
        
        data = f.read(bytes_to_read)
        return header, data
```

### 15.2 PCM Format Converter

```python
import struct

class PCMConverter:
    """Convert between different PCM formats."""
    
    def __init__(self, src_rate: int, src_bits: int, src_channels: int,
                 dst_rate: int, dst_bits: int, dst_channels: int):
        self.src_rate = src_rate
        self.src_bits = src_bits
        self.src_channels = src_channels
        self.dst_rate = dst_rate
        self.dst_bits = dst_bits
        self.dst_channels = dst_channels
    
    def convert_samples(self, samples: list[float]) -> list[float]:
        """Convert float samples (normalized -1.0 to +1.0) to destination format."""
        # Step 1: Handle sample rate conversion if needed
        if self.src_rate != self.dst_rate:
            samples = self._resample(samples)
        
        # Step 2: Handle channel conversion
        if self.src_channels != self.dst_channels:
            samples = self._convert_channels(samples)
        
        return samples
    
    def _resample(self, samples: list[float]) -> list[float]:
        """Simple linear interpolation resampling."""
        if self.dst_rate < self.src_rate:
            # Decimation path (simplified — would need anti-aliasing filter)
            ratio = self.dst_rate / self.src_rate
            indices = [int(i * ratio) for i in range(len(samples))]
            return [samples[i] for i in indices if i < len(samples)]
        else:
            # Interpolation path (simplified — would need anti-imaging filter)
            ratio = self.dst_rate / self.src_rate
            result = []
            for i in range(len(samples)):
                result.append(samples[i])
                # Insert zeros (simplified — proper interpolation would interpolate)
                for _ in range(int(ratio) - 1):
                    result.append(0)
            return result
    
    def _convert_channels(self, samples: list[float]) -> list[float]:
        """Convert between channel configurations."""
        src_frames = len(samples) // self.src_channels
        dst_frames = len(samples) * self.dst_channels // self.src_channels
        
        if self.dst_channels == 1 and self.src_channels == 2:
            # Stereo to mono
            result = []
            for i in range(dst_frames):
                l = samples[i * 2]
                r = samples[i * 2 + 1]
                result.append((l + r) / 2.0)
            return result
        elif self.dst_channels == 2 and self.src_channels == 1:
            # Mono to stereo
            result = []
            for i in range(len(samples)):
                result.append(samples[i])
                result.append(samples[i])
            return result
        else:
            return samples  # Placeholder for more complex conversions
    
    def float_to_int16(self, samples: list[float]) -> list[int]:
        """Convert float samples to signed 16-bit integers."""
        result = []
        for s in samples:
            s_clipped = max(-1.0, min(1.0, s))
            result.append(int(round(s_clipped * 32767.0)))
        return result
    
    def int16_to_float(self, samples: list[int]) -> list[float]:
        """Convert signed 16-bit integers to float samples."""
        return [s / 32768.0 for s in samples]
```

### 15.3 PCM Hex Dump Utility

```python
def pcm_hex_dump(data: bytes, bytes_per_sample: int = 2,
                 channels: int = 2, samples_per_line: int = 4) -> str:
    """Generate a hex dump of PCM data with sample decoding."""
    lines = []
    bytes_per_frame = bytes_per_sample * channels
    samples_total = len(data) // bytes_per_sample
    
    for frame_idx in range(0, samples_total, samples_per_line):
        hex_parts = []
        sample_vals = []
        
        for ch in range(channels):
            for s in range(samples_per_line):
                sample_idx = frame_idx + s
                if sample_idx >= samples_total:
                    break
                byte_offset = (sample_idx * channels + ch) * bytes_per_sample
                if byte_offset + bytes_per_sample > len(data):
                    break
                sample_bytes = data[byte_offset:byte_offset + bytes_per_sample]
                
                if bytes_per_sample == 2:
                    val = struct.unpack('<h', sample_bytes)[0]
                    hex_str = f"{val:06x}" if bytes_per_sample == 3 else f"{val:04x}"
                    sample_vals.append(f"{val:7d}")
                elif bytes_per_sample == 3:
                    val = sample_bytes[0] | (sample_bytes[1] << 8) | (sample_bytes[2] << 16)
                    if val >= 0x800000:
                        val -= 0x1000000
                    hex_str = f"{val:06x}"
                    sample_vals.append(f"{val:7d}")
                else:
                    val = struct.unpack('<h', sample_bytes)[0]
                    hex_str = sample_bytes.hex()
                    sample_vals.append(f"{val:4d}")
                
                hex_parts.append(sample_bytes.hex())
        
        line = " ".join(hex_parts)
        lines.append(f"{frame_idx:8d}: {line}  |  {' '.join(sample_vals)}")
    
    return "\n".join(lines)
```

---

## 16. FFMPEG AND PCM

### 16.1 FFmpeg PCM Decoder/Encoder

FFmpeg provides native PCM decoders for all common PCM formats. The PCM decoder simply reads the raw audio data from the container without decompression (since PCM is not compressed).

**Key FFmpeg PCM codec IDs:**

| Codec ID | FFmpeg Name | Description |
|---|---|---|
| `AV_CODEC_ID_PCM_S8` | s8 | Signed 8-bit PCM |
| `AV_CODEC_ID_PCM_U8` | pcm_u8 | Unsigned 8-bit PCM |
| `AV_CODEC_ID_PCM_S16LE` | pcm_s16le | Signed 16-bit PCM, little-endian |
| `AV_CODEC_ID_PCM_S16BE` | pcm_s16be | Signed 16-bit PCM, big-endian |
| `AV_CODEC_ID_PCM_S24LE` | pcm_s24le | Signed 24-bit PCM, little-endian |
| `AV_CODEC_ID_PCM_S24BE` | pcm_s24be | Signed 24-bit PCM, big-endian |
| `AV_CODEC_ID_PCM_S32LE` | pcm_s32le | Signed 32-bit PCM, little-endian |
| `AV_CODEC_ID_PCM_F32LE` | pcm_f32le | 32-bit float PCM, little-endian |
| `AV_CODEC_ID_PCM_F64LE` | pcm_f64le | 64-bit float PCM, little-endian |

### 16.2 FFmpeg Sample Format Conversions

```bash
# Decode to raw PCM (interleaved signed 16-bit LE, stereo, 44.1 kHz)
ffmpeg -i input.flac -f s16le -acodec pcm_s16le output.pcm

# Decode to float PCM
ffmpeg -i input.flac -f f32le -acodec pcm_f32le output.pcm

# Encode from raw PCM
ffmpeg -f s16le -ar 44100 -ac 2 -i input.pcm -c:a pcm_s16le output.wav

# Convert between sample formats in pipeline
ffmpeg -i input.wav -af "aformat=sample_fmts=s32:channel_layouts=stereo" \
  -c:a pcm_s32le output_s32.pcm

# Upsample during decode
ffmpeg -i input.flac -ar 96000 output_96k.wav

# Convert to planar float for processing
ffmpeg -i input.wav -af "aformat=sample_fmts=fltp:channel_layouts=stereo" \
  -c:a pcm_fltp output_planar.pcm
```

### 16.3 FFmpeg Audio Filters for PCM Processing

```bash
# Apply gain (in dB)
ffmpeg -i input.wav -af "volume=3dB" output.wav

# Normalize to peak
ffmpeg -i input.wav -af "volume=normalize=1" output.wav

# Apply ReplayGain
ffmpeg -i input.wav -af "replaygain=track=1" output.wav

# Apply EBU R128 loudness normalization
ffmpeg -i input.wav -af "loudnorm=I=-16:TP=-1.5:LRA=11" output.wav

# High-pass filter (remove DC offset and sub-bass)
ffmpeg -i input.wav -af "highpass=f=80" output.wav

# Low-pass filter
ffmpeg -i input.wav -af "lowpass=f=20000" output.wav

# Pan stereo to mono
ffmpeg -i input.wav -af "apan=pan=mono|c0=c0+c1" output.wav

# Convert 5.1 to stereo (with downmix coefficients)
ffmpeg -i input_51.flac -af "pan=stereo|c0=c2|c1=c2" output_stereo.flac
```

### 16.4 Probing PCM Stream Information

```bash
# Full stream information
ffprobe -v quiet -print_format json -show_streams input.wav

# Sample format details
ffprobe -v quiet -show_entries stream=sample_rate,channels,bits_per_sample,\
  codec_name,channel_layout input.wav

# Detailed audio analysis
ffmpeg -i input.wav -af "astat=metadata=1:reset=1" -f null -
```

---

## 17. IMPLEMENTATION CHECKLIST

For developers building PCM conversion pipelines, the following elements must be correctly implemented:

### 17.1 Input Handling

- [ ] Detect and validate container format (WAV, AIFF, CAF, raw PCM)
- [ ] Read and validate chunk IDs and sizes (RIFF/WAVE, FORM/AIFF)
- [ ] Parse format chunk: sample rate, bit depth, channel count, endianness
- [ ] Verify format is PCM (not a compressed format like GSM or a-law)
- [ ] Handle extensible format chunk (WAVE_FORMAT_EXTENSIBLE, wValidBitsPerSample, channel mask)
- [ ] Locate and verify audio data chunk (offset and size)
- [ ] Handle padding bytes (chunks with odd sizes)
- [ ] Detect and report truncated files (data chunk shorter than declared)
- [ ] Handle files > 4 GB (RF64, BW64 extension)

### 17.2 PCM Encoding and Decoding

- [ ] Implement signed/unsigned conversion for 8-bit PCM
- [ ] Implement little-endian and big-endian byte reading for 16, 24, 32-bit PCM
- [ ] Implement sign-extension for 20-bit and 24-bit PCM
- [ ] Implement IEEE 754 float read/write (32-bit and 64-bit)
- [ ] Handle planar and interleaved buffer layouts
- [ ] Validate that sample values are within the representable range for the declared bit depth
- [ ] Implement proper clipping for float-to-integer conversion

### 17.3 Sample Rate Conversion

- [ ] Detect when SRC is required (source rate ≠ output rate)
- [ ] Implement integer-ratio upsampling (interpolation + anti-imaging filter)
- [ ] Implement integer-ratio downsampling (anti-aliasing filter + decimation)
- [ ] Implement rational-ratio SRC using polyphase filter banks
- [ ] Use appropriate filter length (longer = better stopband, more latency)
- [ ] Implement linear-phase filtering (symmetric kernel) for minimal phase distortion
- [ ] Handle asynchronous SRC with buffer management
- [ ] Validate that no aliasing or imaging artifacts are introduced

### 17.4 Bit Depth Conversion

- [ ] Implement upsampling with zero-fill (24-bit → 16-bit is NOT upsampling)
- [ ] Implement downsampling with dither (24-bit → 16-bit requires TPDF dither)
- [ ] Select appropriate dither amplitude (1 LSB of output bit depth)
- [ ] Choose between triangular, uniform, or shaped dither based on quality requirements
- [ ] Implement noise shaping for perceptually optimized dither
- [ ] Prevent clipping during gain operations before bit depth reduction
- [ ] Verify that quantization noise floor is at the theoretical level

### 17.5 Channel Configuration

- [ ] Parse channel mask from extensible format chunk
- [ ] Map channels to correct speaker positions for the output layout
- [ ] Implement stereo-to-mono mixing (equal gain or custom coefficients)
- [ ] Implement mono-to-stereo duplication
- [ ] Implement 5.1-to-stereo downmix with standard ITU coefficients (or custom)
- [ ] Implement stereo-to-5.1 upmixing (only possible with lossless panning)
- [ ] Handle channel reordering for different format conventions
- [ ] Preserve LFE channel during conversions (don't discard unless specified)

### 17.6 Floating-Point Processing

- [ ] Handle denormalized numbers (flush to zero or preserve)
- [ ] Set appropriate rounding mode for float-to-int conversion
- [ ] Handle NaN and infinity values (clip to ±1.0 or report error)
- [ ] Preserve ±0.0 sign bit if important (usually not for audio)
- [ ] Use FMA (fused multiply-add) when available for reduced rounding error
- [ ] Verify that 32-bit float precision is sufficient for the processing chain
- [ ] Use 64-bit float for accumulated computation (e.g., summing thousands of samples)

### 17.7 Output Writing

- [ ] Write proper RIFF/WAVE header with correct chunk sizes
- [ ] Verify `nBlockAlign = nChannels × bytes_per_sample`
- [ ] Verify `nAvgBytesPerSec = sample_rate × nBlockAlign`
- [ ] Include `fact` chunk for non-PCM formats
- [ ] Write extensible format chunk when needed (bit depths not directly supported, non-standard channel layouts)
- [ ] Set correct channel mask in extensible format
- [ ] Write padding bytes if data size is odd
- [ ] Verify final file size matches declared RIFF size + 8
- [ ] Use atomic write (write to temp file, rename) to prevent corruption on interruption

### 17.8 Metadata Preservation

- [ ] Copy text metadata from source to destination
- [ ] Preserve cover art (APIC frame, covr atom)
- [ ] Copy ReplayGain tags if present
- [ ] Copy EBU R128 loudness metadata if present
- [ ] Copy encoder identity and version tags
- [ ] Copy track/album disc and number information
- [ ] Copy date and release information
- [ ] Preserve custom/unknown tags (passthrough)

### 17.9 Verification

- [ ] Run `ffprobe -show_streams` on output to verify format
- [ ] Use `kid3-cli` to verify metadata in output container
- [ ] Run bit-exact comparison when round-tripping (e.g., WAV → FLAC → WAV)
- [ ] Measure loudness (LUFS) of output to verify normalization
- [ ] Check for clipping: scan for any sample at ±full-scale
- [ ] Verify duration matches source (within one sample)
- [ ] Check phase correlation between channels
- [ ] Verify channel balance and stereo image preservation
- [ ] Test with edge cases: silence, full-scale sine, DC offset, high-frequency tones
