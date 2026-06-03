# Audio Engineering: Float vs Integer Audio Precision — Deep Technical Reference
> **Category:** AudioEngineering
> **File Extensions:** N/A (theory reference document)
> **MIME Types:** N/A
> **Standardization Body:** IEEE, AES
> **Primary Specification:** IEEE 754-2019 (Floating-Point Arithmetic), AES5-2020 (Digital Audio)
> **Patent Status:** IEEE 754 is a published standard; implementations carry their own licenses
> **License:** Standards are public domain; implementations subject to their own terms
> **Current Version:** IEEE 754-2019, AES5-2020
> **Active Development:** Yes — 32-bit float recording becoming standard in professional audio

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Origins of Digital Audio Representation
- **Creator(s):** Various — pulse-code modulation (PCM) developed by Alec Reeves (1937); IEEE 754 standardized by various contributors (1985)
- **Year Created:** PCM concepts: 1937; IEEE 754-1985 first standard; current IEEE 754-2019
- **Original Purpose:** PCM converts analog signals to discrete numerical representations for digital processing, transmission, and storage. IEEE 754 standardized how computers represent real numbers.
- **Problem with Predecessors:** Early digital systems used inconsistent number representations. IEEE 754 unified floating-point representation across platforms.

### 1.2 Key Historical Milestones
|| Year | Milestone | Significance |
|------|---------|-------------|
| 1937 | PCM concept (Alec Reeves) | Foundational digital audio idea |
| 1948 | First PCM demonstration (Oliver, Pierce) | Proof of concept |
| 1957 | First commercial PCM system (CSIRAC) | Audio applications begin |
| 1977 | First digital audio recording (2-bit, 30 kHz) | Studio recording emergence |
| 1982 | CD-DA standard: 16-bit/44.1 kHz | Consumer digital audio |
| 1985 | IEEE 754-1985 | Floating-point standard |
| 1990s | 24-bit recording in studios | Professional resolution |
| 2000s | 32-bit float internal processing | Professional DAWs |
| 2010s | 32-bit float file recording (Sound Devices, Zoom) | Consumer 32-bit float |
| 2019 | IEEE 754-2019 | Current standard |

### 1.3 Current Adoption
- **Primary use cases:** Recording, mixing, mastering, real-time DSP, audio plugins, broadcast
- **Platforms with native support:** All DAWs, audio interfaces, streaming platforms, game engines
- **Major services:** All digital audio services use PCM (converted from lossy codecs)
- **Hardware support:** Audio interfaces (24-bit ADC/DAC standard), AD/DA converters
- **Status:** Integer PCM dominant in files; 32-bit float dominant in processing

---

## 2. INTEGER PCM REPRESENTATION

### 2.1 Signed Integer Representation
Digital audio uses signed two's complement integers:

```
Two's complement representation:
  For N bits:
    Range: -(2^(N-1)) to +(2^(N-1) - 1)
    
  Example (8-bit signed):
    01111111 = +127
    00000001 = +1
    00000000 = 0
    11111111 = -1
    10000000 = -128
```

### 2.2 Bit Depth Specifications
| Bit Depth | Bits Used | Range | Steps | Dynamic Range | CD Quality |
|-----------|-----------|-------|-------|--------------|-----------|
| 8-bit unsigned | 8 | 0–255 | 256 | 48 dB | No |
| 8-bit signed | 8 | –128 to +127 | 256 | ~42 dB | No |
| 16-bit signed | 16 | –32,768 to +32,767 | 65,536 | 96 dB | Yes |
| 20-bit signed | 20 | –524,288 to +524,287 | 1,048,576 | 120 dB | Yes |
| 24-bit signed | 24 | –8,388,608 to +8,388,607 | 16,777,216 | 144 dB | Yes |
| 32-bit signed | 32 | –2,147,483,648 to +2,147,483,647 | 4,294,967,296 | 192 dB | Yes |

### 2.3 16-bit PCM (CD Quality)
```
Format: Signed 16-bit, little-endian, stereo interleaved

Bytes per second (stereo, 44.1 kHz):
  2 bytes × 2 channels × 44,100 samples/s = 176,400 bytes/s

Bitrate:
  16 bits × 44,100 Hz × 2 channels = 1,411,200 bps = 1,411.2 kbps
```

### 2.4 24-bit PCM
```
Format: Signed 24-bit, typically stored in 32-bit container
  Sample stored: [8 bits unused][24 bits sample] or [24 bits sample][8 bits unused]
  
Byte order: Usually little-endian in WAV files

Bytes per second (stereo, 96 kHz):
  3 bytes × 2 channels × 96,000 samples/s = 576,000 bytes/s
  
  OR in 32-bit container (padded):
  4 bytes × 2 channels × 96,000 samples/s = 768,000 bytes/s
```

### 2.5 Clipping in Integer PCM
Integer PCM clips at the maximum representable value:

```
For 16-bit signed:
  Maximum positive:  +32,767 (0x7FFF)
  Maximum negative:  –32,768 (0x8000)
  Clipping: Any value > +32,767 is clipped to +32,767
            Any value < –32,768 is clipped to –32,768

Clipping distortion:
  - Hard clipping creates high-order harmonics
  - Always destructive to audio quality
  - Must be avoided at recording/input stage
```

---

## 3. IEEE 754 FLOATING-POINT

### 3.1 IEEE 754 Single Precision (32-bit Float)
```
Layout: 1 bit (sign) + 8 bits (exponent) + 23 bits (fraction/mantissa)

Formula:
  value = (-1)^sign × 1.fraction × 2^(exponent - 127)

Special values:
  exponent = 0, fraction = 0: Zero
  exponent = 0, fraction ≠ 0: Denormalized
  exponent = 255, fraction = 0: Infinity
  exponent = 255, fraction ≠ 0: NaN
```

### 3.2 32-bit Float in Audio Context
Audio uses normalized float representation:

```
Normalized audio range: –1.0 to +1.0
  - 0 dBFS = 1.0 (maximum)
  - Full-scale sine wave: ±1.0
  
Exponent range:
  Minimum (denormalized): ~1.4 × 10^(-45) [subnormal]
  Normal minimum: 2^(-126) ≈ 1.2 × 10^(-38)
  Maximum: 2^127 ≈ 3.4 × 10^38
```

### 3.3 Dynamic Range of 32-bit Float
```
Effective precision at unity gain (|x| ≈ 1.0):
  Mantissa: 23 bits ≈ 24-bit precision
  Dynamic range: 23 × 6.02 ≈ 138 dB (practical)
  Theoretical: 144 dB (24 bits)

Below unity gain, precision decreases:
  |x| = 0.5: ~23 bits + 1 bit (implicit) - 1 = ~23 bits
  |x| = 0.25: ~22 bits effective
  |x| = 10^(-6): ~3 bits effective
  
Total dynamic range (including exponent):
  Can represent: 2^127 ≈ 1.7 × 10^38
  In dB: 20 × log10(2^127) ≈ 762 dBFS
  
But effective SNR at any given level:
  ~144 dB at unity gain
  Degrades at lower signal levels
```

### 3.4 64-bit Double Precision (Double)
```
Layout: 1 bit (sign) + 11 bits (exponent) + 52 bits (fraction/mantissa)

Formula:
  value = (-1)^sign × 1.fraction × 2^(exponent - 1023)

Effective mantissa: 53 bits (52 + implicit 1) ≈ 54-bit precision
Dynamic range: 53 × 6.02 ≈ 319 dB

Use cases:
  - High-precision mixing and mastering
  - Accumulation of many channels
  - Mastering bus processing
  - Not typically used in real-time DSP (too slow)
```

---

## 4. DYNAMIC RANGE & SIGNAL-TO-NOISE RATIO

### 4.1 Theoretical Dynamic Range Formula
```
Dynamic Range = 6.02 × bit_depth + 1.76  [dB]

For integer formats (assuming dithered quantization):
  16-bit: 6.02 × 16 + 1.76 = 96.32 + 1.76 = 98.08 dB
  20-bit: 6.02 × 20 + 1.76 = 120.4 + 1.76 = 122.16 dB
  24-bit: 6.02 × 24 + 1.76 = 144.48 + 1.76 = 146.24 dB
  32-bit: 6.02 × 32 + 1.76 = 192.64 + 1.76 = 194.4 dB

For 32-bit float (at unity gain):
  ~144 dB (24-bit equivalent mantissa)
```

### 4.2 Dynamic Range Comparison Table
| Format | Bit Depth | Theoretical DR | Practical DR | Notes |
|--------|-----------|--------------|--------------|-------|
| 8-bit unsigned | 8 | 48 dB | 42 dB | Telephone quality |
| 8-bit signed | 8 | 48 dB | 42 dB | Better than unsigned |
| 12-bit | 12 | 72 dB | 66 dB | Early digital audio |
| 16-bit signed | 16 | 98 dB | 93 dB | CD quality |
| 20-bit signed | 20 | 122 dB | 117 dB | Professional |
| 24-bit signed | 24 | 146 dB | 140 dB | Studio standard |
| 32-bit signed | 32 | 194 dB | 192 dB | Beyond practical need |
| 32-bit float | 32 | 1528 dB | ~144 dB at unity | Internal DSP |
| 64-bit float | 64 | 7600 dB | ~330 dB at unity | High precision |

### 4.3 Why 16-bit Is "CD Quality"
```
CD specification: 16-bit, 44.1 kHz

Dynamic range of 16-bit:
  98 dB theoretical maximum
  93 dB practical (accounting for converter noise)
  
Human hearing dynamic range:
  Minimum audible: ~0 dB SPL (threshold)
  Maximum safe: ~120 dB SPL (pain threshold)
  Practical range: ~100–120 dB
  
Conclusion: 16-bit PCM covers the full practical hearing range
  44.1 kHz sample rate covers up to 22.05 kHz (half of Nyquist)
  Human hearing: ~20 Hz to ~20 kHz
```

### 4.4 Why 24-bit Is Standard in Studios
```
Recording headroom:
  - 16-bit provides only ~20 dB headroom above 0 dBFS
  - Transient peaks can exceed this easily
  - 24-bit provides ~50 dB headroom
  
Noise floor:
  - 16-bit noise floor: –93 dBFS (can be audible in quiet passages)
  - 24-bit noise floor: –141 dBFS (far below hearing threshold)
  - Recording at 24-bit ensures quantization noise is inaudible
  
Processing margin:
  - Each mix operation can add quantization noise
  - 24-bit provides margin for multiple generations
  - 32-bit float in DAW prevents accumulation
```

---

## 5. DITHERING

### 5.1 When to Dither
Dithering is required when **reducing bit depth** to prevent quantization distortion:

```
NEVER dither when:
  - Reducing from 32-bit float to another float format
  - The destination has equal or greater bit depth
  - The signal is already below the noise floor

ALWAYS dither when:
  - Reducing from 32-bit float to 24-bit integer
  - Reducing from 24-bit to 16-bit integer
  - Reducing from 16-bit to 8-bit integer
```

### 5.2 Dithering Theory
Without dithering, reducing bit depth creates quantization distortion (harmonics of the signal). Dithering converts distortion into random noise:

```
Without dithering (N-bit quantization):
  Error = quantization noise (correlated with signal, sounds harsh)
  
With dithering:
  Error = random noise (sounds like hiss, less annoying)
```

### 5.3 TPDF (Triangular Probability Density Function)
TPDF dither provides optimal noise shaping:

```
TPDF dither: Two uniformly distributed random values added together
  - PDF: triangular shape
  - Range: –1 to +1 LSB
  - Standard deviation: 1/sqrt(6) ≈ 0.408 LSB
  - Effective: 1.5 bits of noise reduction

Algorithm:
  1. Generate two uniform random values: r1, r2 ∈ [-0.5, +0.5]
  2. Add to sample: x' = x + r1 + r2
  3. Round to target bit depth

Python example:
  import random
  r1 = random.uniform(-0.5, 0.5)
  r2 = random.uniform(-0.5, 0.5)
  x_dithered = x + r1 + r2
  x_quantized = round(x_dithered)
```

### 5.4 Noise Shaping
Noise shaping moves quantization noise away from sensitive frequency bands:

```
Flat dither: White noise across all frequencies
Noise shaping: Shaped noise, reduced at critical frequencies

Example: Simple first-order noise shaper:
  out[n] = in[n] + (in[n] - in[n-1]) × amount
  This reduces low-frequency quantization noise

Professional implementations:
  - Apogee UV22HR: Proprietary noise shaping
  - iZotope MBIT+ : Psychoacoustically optimized
  - Sony/Philips Super Bit Mapping: 16-to-20 bit mapping
```

### 5.5 Dither in FFmpeg
```bash
# Dither when converting bit depths:
ffmpeg -i input_24bit.wav -c:a pcm_s16le -dither_method rectangular output_16bit.wav

# Available dither methods:
#  -dither_method rectangular   # Uniform PDF
#  -dither_method triangular    # TPDF (recommended)
#  -dither_method shiblis      # Shibata low-pass noise shaping
#  -dither_method modified_e   # Modified E-weighted
#  -dither_method lipshitz     # Lipshitz low-noise shaping
#  -dither_method f_weighted   # F-weighted
#  -dither_method improved_e   # Improved E-weighted

# Default: triangular (TPDF)
```

---

## 6. CLIPPING IN FLOAT VS INTEGER

### 6.1 Integer Clipping
Integer formats have a hard clipping point:

```c
// 16-bit integer clipping example:
int16_t clip_16bit(float sample) {
    int32_t s = (int32_t)(sample * 32768.0f);
    if (s > 32767) s = 32767;
    if (s < -32768) s = -32768;
    return (int16_t)s;
}
```

### 6.2 Float Clipping
Float can exceed ±1.0 without clipping internally:

```
Float range at unity gain:
  - 1.0 = 0 dBFS (digital maximum)
  - Values > 1.0 are above 0 dBFS
  - No hard clipping until you convert to integer!

Example:
  Internal float: 1.5 (corresponds to +3.5 dBFS)
  Converted to 16-bit: 1.5 × 32768 = 49152 → clipped to 32767
  
Conclusion: Float protects against clipping during processing,
  but you must normalize before integer conversion
```

### 6.3 32-bit Float Recording
Modern recorders (Sound Devices, Zoom) record in 32-bit float:

```
Benefits:
  1. No clipping during recording
     - Can capture +3 dBFS, +6 dBFS, +10 dBFS without clipping
     - 32-bit float can represent levels far above 0 dBFS
     
  2. Large internal dynamic range
     - 144 dB at unity gain
     - Adequate for any real-world signal
     
  3. Simple gain staging
     - Don't need to set perfect recording levels
     - Can fix level in post without quality loss
```

---

## 7. FLOAT VS INTEGER IN DSP

### 7.1 Why DSP Uses Float
Floating-point offers advantages for audio processing:

| Aspect | Fixed-Point | Floating-Point |
|--------|------------|----------------|
| Overflow risk | High (need careful scaling) | Low (exponent handles range) |
| Precision | Fixed (limited mantissa) | Variable (degrades at low levels) |
| Complexity | High (scaling, saturation) | Low (normalized math) |
| Speed (modern CPU) | Similar (SIMD optimized) | Similar (SIMD optimized) |
| Audio quality | Excellent (with proper design) | Excellent |

### 7.2 Accumulation Precision
Summing many channels requires precision:

```c
// Fixed-point accumulation (16-bit inputs):
int32_t sum = 0;
for (int i = 0; i < 100; i++) {
    sum += input[i];  // May overflow if not 32-bit accumulator
}

// Float accumulation (32-bit inputs):
float sum = 0.0f;
for (int i = 0; i < 100; i++) {
    sum += input[i];  // No overflow, precision degrades slightly
}

// Double accumulation (64-bit):
double sum = 0.0;
for (int i = 0; i < 100; i++) {
    sum += input[i];  // Very high precision
}
```

### 7.3 Gain Staging in Float
```c
// Float gain staging (no clipping risk):
float apply_gain(float input, float gain_db) {
    float linear_gain = powf(10.0f, gain_db / 20.0f);
    return input * linear_gain;  // May exceed ±1.0, that's fine
}

// Convert to integer at the end:
int16_t to_int16(float sample) {
    float clipped = fmaxf(-1.0f, fminf(1.0f, sample));
    return (int16_t)(clipped * 32767.0f);
}
```

### 7.4 Coefficient Precision
DSP coefficients (filter coefficients, etc.) have precision requirements:

```
Filter coefficient precision requirements:
  - Simple EQ: 24-bit float (32-bit) sufficient
  - Precision filters: 32-bit float minimum
  - Measurement instruments: 64-bit double
  
IIR filter stability:
  - Fixed-point: Requires careful pole placement
  - Float: Poles stable with 32-bit float
  - Double: Maximum precision for critical filters
```

---

## 8. NORMALIZATION

### 8.1 Why Normalize
Normalization scales audio to a target level:

```
Common normalization targets:
  - Peak normalization: Scale so peak = 0 dBFS (–1 dBFS, –3 dBFS)
  - RMS normalization: Scale so RMS = target level
  - Loudness normalization: Scale so integrated loudness = target LUFS
```

### 8.2 Peak Normalization Formula
```python
# Peak normalization to target dBFS:
def peak_normalize(samples, target_db=-1.0):
    peak = max(abs(samples))
    target_linear = 10 ** (target_db / 20.0)
    gain = target_linear / peak
    return samples * gain

# Example:
# samples = [-0.5, 0.7, -0.8, 0.9]
# peak = 0.9
# target = -1 dBFS = 0.891 (linear)
# gain = 0.891 / 0.9 = 0.99
# output = [-0.495, 0.693, -0.792, 0.891]
```

### 8.3 Normalization in Float vs Integer
```
Integer normalization issues:
  - Must calculate gain carefully
  - Rounding introduces quantization
  - Gain may be approximate due to integer division
  
Float normalization:
  - Gain is floating-point precise
  - No quantization until converted to integer
  - Can apply gain without precision loss
```

---

## 9. DC OFFSET

### 9.1 DC Offset in Integer PCM
DC offset is a constant non-zero component in the signal:

```c
// DC offset in 16-bit PCM:
int16_t sample_with_dc = 256;  // DC offset of 256 counts = 256/32768 = 0.78%
int16_t dc_free = sample_with_dc - dc_offset;  // Remove DC

// DC offset in float:
float sample_float = 0.01f;  // 1% DC offset (no scaling needed)
float dc_free_float = sample_float - 0.01f;
```

### 9.2 Representing DC in Float vs Integer
| Aspect | Integer | Float |
|--------|---------|-------|
| DC resolution | 1 LSB | Very fine (mantissa precision) |
| DC range | ±32767 counts | ±1.0 (normalized) |
| DC stability | Good | Excellent |
| DC removal | Simple subtraction | Simple subtraction |

### 9.3 DC Offset in Audio Files
```bash
# Detect DC offset with sox:
sox input.wav -n stat

# Remove DC offset:
ffmpeg -i input.wav -af "highpass=f=5,lowpass=f=0" output.wav

# High-pass filter (removes DC):
# cutoff at 5 Hz removes DC while preserving bass
```

---

## 10. FFmpeg SAMPLE FORMATS

### 10.1 FFmpeg Sample Format Codes
| Format | Bits | Type | Description |
|--------|------|------|-------------|
| `u8` | 8 | Unsigned | 0–255, not used for audio |
| `s16` | 16 | Signed | Standard CD quality |
| `s32` | 32 | Signed | Professional (padded) |
| `s64` | 64 | Signed | Very high precision |
| `flt` | 32 | Float | IEEE 754 single precision |
| `dbl` | 64 | Double | IEEE 754 double precision |
| `s16p` | 16 | Planar | Planar 16-bit |
| `s32p` | 32 | Planar | Planar 32-bit signed |
| `fltp` | 32 | Planar | Planar float |
| `dblp` | 64 | Planar | Planar double |

### 10.2 Planar vs Interleaved
```
Interleaved (all FFmpeg formats except *p):
  [L0][R0][L1][R1][L2][R2]...
  Memory layout: Left, Right, Left, Right, ...
  Access pattern: Good for sequential access
  
Planar (*p formats):
  [L0][L1][L2]...[R0][R1][R2]...
  Memory layout: All Left, then all Right
  Access pattern: Good for SIMD (all left samples processed together)
```

### 10.3 FFmpeg Sample Format Conversion
```bash
# Convert between formats:
ffmpeg -i input.wav -c:a pcm_s32le output_s32.wav    # To 32-bit signed
ffmpeg -i input.wav -c:a pcm_f32le output_f32.wav    # To 32-bit float
ffmpeg -i input.wav -c:a pcm_s24le output_s24.wav    # To 24-bit (stored in 32-bit)
ffmpeg -i input.wav -c:a pcm_s16le output_s16.wav    # To 16-bit signed

# Specify planar format:
ffmpeg -i input.wav -c:a pcm_fltp output_planar.wav   # Planar float
```

### 10.4 SwrContext for Format Conversion
```c
#include <libswresample/swresample.h>

// Initialize resampler with format conversion:
struct SwrContext *swr = swr_alloc();
av_opt_set_channel_layout(swr, "in_ch_layout", AV_CH_LAYOUT_STEREO, 0);
av_opt_set_channel_layout(swr, "out_ch_layout", AV_CH_LAYOUT_STEREO, 0);
av_opt_set_int(swr, "in_sample_rate", 44100, 0);
av_opt_set_int(swr, "out_sample_rate", 48000, 0);
av_opt_set_sample_fmt(swr, "in_sample_fmt", AV_SAMPLE_FMT_S16, 0);
av_opt_set_sample_fmt(swr, "out_sample_fmt", AV_SAMPLE_FMT_FLTP, 0);
swr_init(swr);

// Convert:
int out_samples = swr_convert(swr, out_buffer, out_samples_max,
                             (const uint8_t**)in_buffer, in_samples);

// Cleanup:
swr_free(&swr);
```

---

## 11. PRACTICAL IMPLICATIONS

### 11.1 Recording Workflow
```
Step 1: Capture at 24-bit, 48/96 kHz
  - Use 24-bit for headroom and noise floor
  - Sample rate based on frequency content needed

Step 2: Mix in 32-bit float
  - DAW processing at float precision
  - No accumulation loss from summing channels

Step 3: Export at 24-bit
  - Preserve resolution for mastering
  - Convert to 16-bit only for distribution
```

### 11.2 Mastering Workflow
```
Step 1: Import 24-bit mix
  - Process with 32-bit float plugins
  - Apply EQ, compression, limiting

Step 2: Dither on export
  - Dither to 16-bit only for CD/streaming
  - Keep 24-bit master for archival

Step 3: Normalize for distribution
  - Peak normalize to –0.3 to –1.0 dBFS
  - Or loudness normalize to streaming target
```

### 11.3 Real-Time Audio Processing
```
Constraints:
  - Latency: Must process within buffer size
  - Bit depth: Usually 32-bit float internally
  - Output: Typically 24-bit DAC

Best practices:
  - Process in float at all stages
  - Convert to integer only at audio output
  - Use high-quality dither at conversion
  - Monitor peak levels to prevent clipping
```

---

## 12. MYTHS & MISCONCEPTIONS

### 12.1 "32-bit float has 32-bit precision"
False. 32-bit float has 24-bit equivalent precision (23-bit mantissa + 1 implicit bit). The exponent allows representing a large range, but precision is limited.

### 12.2 "24-bit recording is 24 times better than 16-bit"
False. 24-bit provides ~50 dB more dynamic range than 16-bit, but human hearing has ~100–120 dB range. 16-bit already covers most of this; 24-bit adds headroom and reduces quantization noise.

### 12.3 "Dithering adds noise to the recording"
True technically, but the noise is below hearing threshold and prevents quantization distortion. The trade-off is beneficial.

### 12.4 "Float can clip without distortion"
False. Float can represent values beyond ±1.0, but when converted to integer, any value beyond ±1.0 clips. Float protects during processing but not at output.

### 12.5 "Higher sample rate means higher quality"
Not necessarily. Higher sample rates extend bandwidth (beyond 20 kHz) but don't improve resolution within the audible band. 44.1/48 kHz is sufficient for full-range audio.

---

## 13. REFERENCE IMPLEMENTATIONS

### 13.1 IEEE 754 Reference
```
IEEE 754-2019:
  - Single precision (binary32): 1+8+23 bits
  - Double precision (binary64): 1+11+52 bits
  - Extended precision: Optional 80-bit format

Bit layout for single precision:
  S EEEEEEEE FFFFFFFFFFFFFFFFFFFFFFF
  | |________| |____________________|
  sign  exponent      fraction (23 bits)
```

### 13.2 Audio Precision Reference
```
AES5-2020 (Digital Audio Recording):
  - Recommended practice for PCM measurement
  - Dithering recommendations
  - Word length selection
```

---

## 14. IMPLEMENTATION CHECKLIST

### Bit Depth Handling
- [ ] Support 16-bit, 24-bit, and 32-bit integer PCM
- [ ] Support 32-bit float WAV files
- [ ] Implement proper conversion between formats
- [ ] Apply dither when reducing bit depth
- [ ] Prevent clipping during format conversion

### Float Processing
- [ ] Use 32-bit float for internal DSP
- [ ] Use 64-bit double for accumulation or critical processing
- [ ] Normalize to ±1.0 range before output
- [ ] Handle values beyond ±1.0 without silent clipping

### Quality Verification
- [ ] Test with full-scale sine waves at various frequencies
- [ ] Verify no clipping in normal operation
- [ ] Test dither quality with low-level signals
- [ ] Compare float vs integer processing quality

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*

---

## 15. FIXED-POINT ARITHMETIC IN AUDIO DSP

### 15.1 Fixed-Point Representation
Fixed-point numbers use integer storage with an implicit scaling factor:

```python
class FixedPointQ:
    """
    Fixed-point number with Q-format.
    
    Qm.n format:
        m = integer bits (including sign)
        n = fractional bits
        Total bits = m + n
    """
    
    def __init__(self, value, m=16, n=16):
        """
        Args:
            value: Floating-point value
            m: Integer bits (including sign)
            n: Fractional bits
        """
        self.m = m
        self.n = n
        self.scale = 2**n
        
        # Convert to fixed-point
        self.value = int(round(value * self.scale))
    
    @property
    def float_value(self):
        """Convert back to float."""
        return self.value / self.scale
    
    def __add__(self, other):
        """Addition (no overflow check)."""
        result = FixedPointQ(0, self.m, self.n)
        result.value = self.value + other.value
        return result
    
    def __mul__(self, other):
        """Multiplication (may need shifting)."""
        result = FixedPointQ(0, self.m, self.n)
        # Qm.n × Qm.n → Q(2m).(2n)
        product = self.value * other.value
        # Shift right by fractional bits to get back to Qm.n
        result.value = product >> self.n
        return result
```

### 15.2 Common Q-Formats in Audio
| Format | Total Bits | Fractional Bits | Range | Typical Use |
|--------|-----------|----------------|------|-------------|
| Q0.15 | 16 | 15 | -1 to +0.999 | Audio samples |
| Q1.14 | 16 | 14 | -2 to +1.999 | Filter coefficients |
| Q8.23 | 32 | 23 | -128 to +127.999 | 32-bit fixed |
| Q4.27 | 32 | 27 | -8 to +7.999 | Accumulator |

### 15.3 Audio DSP in Fixed-Point
```python
class FixedPointBiquad:
    """
    Biquad filter in fixed-point arithmetic.
    """
    
    def __init__(self, coefficients, q_format=16):
        """
        Args:
            coefficients: [b0, b1, b2, a1, a2] (normalized)
            q_format: Number of fractional bits
        """
        self.q = q_format
        self.scale = 2**q_format
        
        # Convert to fixed-point
        self.b0 = int(coefficients[0] * self.scale)
        self.b1 = int(coefficients[1] * self.scale)
        self.b2 = int(coefficients[2] * self.scale)
        self.a1 = int(coefficients[3] * self.scale)
        self.a2 = int(coefficients[4] * self.scale)
        
        # State variables (need extra bits for overflow prevention)
        self.x1 = 0
        self.x2 = 0
        self.y1 = 0
        self.y2 = 0
    
    def process_sample(self, x0):
        """Process single sample in fixed-point."""
        # Convert input to fixed-point
        x0_fp = int(x0 * self.scale)
        
        # Direct Form II Transposed
        # y = b0*x + b1*x1 + b2*x2 - a1*y1 - a2*y2
        y0 = (self.b0 * x0_fp + 
              self.b1 * self.x1 + 
              self.b2 * self.x2 -
              self.a1 * self.y1 - 
              self.a2 * self.y2) >> self.q
        
        # Update state
        self.x2 = self.x1
        self.x1 = x0_fp
        self.y2 = self.y1
        self.y1 = y0
        
        # Convert back to float
        return y0 / self.scale
```

### 15.4 Overflow Handling
```python
def saturate(value, bits):
    """
    Saturating arithmetic for fixed-point.
    
    Args:
        value: Fixed-point value
        bits: Number of bits
    
    Returns:
        Saturated value
    """
    max_val = (1 << (bits - 1)) - 1
    min_val = -(1 << (bits - 1))
    
    if value > max_val:
        return max_val
    elif value < min_val:
        return min_val
    else:
        return value

def wrap(value, bits):
    """
    Wrapping arithmetic (modular arithmetic).
    
    Args:
        value: Fixed-point value
        bits: Number of bits
    
    Returns:
        Wrapped value
    """
    mask = (1 << bits) - 1
    return value & mask
```

---

## 16. ROUNDING AND QUANTIZATION

### 16.1 Rounding Modes
```python
def round_truncate(value, bits):
    """Truncate (round toward zero)."""
    scale = 2**bits
    return int(value * scale) / scale

def round_floor(value, bits):
    """Round down (floor)."""
    scale = 2**bits
    return math.floor(value * scale) / scale

def round_ceil(value, bits):
    """Round up (ceiling)."""
    scale = 2**bits
    return math.ceil(value * scale) / scale

def round_nearest(value, bits):
    """Round to nearest (ties to even)."""
    scale = 2**bits
    return round(value * scale) / scale

def round_nearest_away(value, bits):
    """Round to nearest (ties away from zero)."""
    scale = 2**bits
    scaled = value * scale
    if scaled >= 0:
        return math.floor(scaled + 0.5) / scale
    else:
        return math.ceil(scaled - 0.5) / scale

def round_stochastic(value, bits, rng):
    """
    Stochastic rounding (probabilistic).
    Useful for hardware and neural audio.
    
    Args:
        value: Floating-point value
        bits: Number of output bits
        rng: Random number generator
    """
    scale = 2**bits
    floor_val = math.floor(value * scale)
    prob = (value * scale) - floor_val  # Fractional part
    
    if rng.random() < prob:
        return (floor_val + 1) / scale
    else:
        return floor_val / scale
```

### 16.2 Mid-Tread vs Mid-Rise Quantizers
```python
def quantize_midtread(x, bits):
    """
    Mid-tread (mid-tread) quantizer.
    Includes zero as a quantization level.
    
    For N bits:
        - Has exact zero
        - Symmetric around zero
        - Common in audio (linear PCM)
    """
    levels = 2**(bits - 1)  # Number of positive levels
    scale = 2**(bits - 1) - 1  # Max positive value
    
    # Clamp
    x_clamped = max(-1, min(1, x))
    
    # Quantize
    x_q = round(x_clamped * scale) / scale
    
    return x_q

def quantize_midrise(x, bits):
    """
    Mid-rise (mid-rise) quantizer.
    Zero is NOT a quantization level.
    
    For N bits:
        - No exact zero
        - Symmetric, but zero falls between levels
        - Used in some DSP applications
    """
    levels = 2**bits  # Total levels (even)
    scale = levels / 2  # Range from -1 to +1
    
    # Scale and round
    x_scaled = x * scale
    x_q = (math.floor(x_scaled) + 0.5) / scale
    
    return x_q
```

### 16.3 Noise Shaping
```python
def noise_shape(audio, target_bits, shaping_filter):
    """
    Apply noise shaping when reducing bit depth.
    
    Args:
        audio: Input audio (float)
        target_bits: Target bit depth
        shaping_filter: Frequency response target (e.g., 'high_emphasis')
    
    Returns:
        Quantized audio
    """
    # Design shaping filter
    if shaping_filter == 'high_emphasis':
        # Reduce noise at high frequencies
        b = [1.0, -0.5]  # Simple high-frequency emphasis
        a = [1.0, 0.0]
    elif shaping_filter == 'flat':
        b = [1.0]
        a = [1.0]
    else:
        b = [1.0]
        a = [1.0]
    
    # Calculate quantization error
    error = 0
    output = np.zeros_like(audio)
    
    for i in range(len(audio)):
        # Input minus shaped previous error
        x = audio[i] - np.dot(b, error_hist)
        
        # Quantize
        output[i] = quantize(x, target_bits)
        
        # Calculate error
        error = x - output[i]
        
        # Update error history
        error_hist = np.roll(error_hist, 1)
        error_hist[0] = error
    
    return output
```

---

## 17. AUDIO FILE FORMAT SPECIFICATIONS

### 17.1 WAV (RIFF) Bit Depths
```python
WAV_FORMATS = {
    8: {
        'type': 'unsigned',
        'signed_range': None,
        'unsigned_range': (0, 255),
        'use': ' telephony, basic',
    },
    16: {
        'type': 'signed',
        'range': (-32768, 32767),
        'use': 'CD-quality audio',
    },
    20: {
        'type': 'signed',
        'range': (-524288, 524287),
        'storage': 'padded to 24-bit or 32-bit',
        'use': 'Professional',
    },
    24: {
        'type': 'signed',
        'range': (-8388608, 8388607),
        'storage': 'padded to 32-bit',
        'use': 'Studio recording',
    },
    32: {
        'type': 'signed',
        'range': (-2147483648, 2147483647),
        'use': 'Processing, archival',
    },
    32: {
        'type': 'float',
        'range': (-1.0, 1.0),
        'standard': 'IEEE 754',
        'use': 'Internal DSP, recording',
    },
    64: {
        'type': 'float',
        'range': (-1.0, 1.0),
        'standard': 'IEEE 754',
        'use': 'High-precision processing',
    },
}
```

### 17.2 AIFF Bit Depths
```python
AIFF_FORMATS = {
    8: {
        'type': 'unsigned',
        'range': (0, 255),
        'note': 'Same as WAV',
    },
    16: {
        'type': 'signed',
        'range': (-32768, 32767),
        'note': 'Big-endian',
    },
    24: {
        'type': 'signed',
        'range': (-8388608, 8388607),
        'note': 'Packed 3 bytes',
    },
    32: {
        'type': 'signed',
        'range': (-2147483648, 2147483647),
        'note': 'Big-endian',
    },
}
```

### 17.3 FLAC Bit Depths
```python
FLAC_FORMATS = {
    8: {'support': True, 'storage': '32-bit internally'},
    16: {'support': True, 'storage': 'Native'},
    20: {'support': True, 'storage': '32-bit internally'},
    24: {'support': True, 'storage': 'Native'},
    32: {'support': False, 'note': 'Not supported for lossless'},
}
```

---

## 18. ANALOG VS DIGITAL DYNAMIC RANGE

### 18.1 Vinyl Records
```python
VINYL_SPECS = {
    'frequency_range': '30 Hz - 15 kHz',
    'dynamic_range': '60-70 dB',
    'channel_separation': '30 dB',
    'wow_and_flutter': '0.03-0.15%',
    'noise_floor': '-60 to -70 dB below maximum',
    'notes': [
        'Low-frequency noise (rumble) from turntable motor',
        'High-frequency noise from dust and wear',
        'Dynamic range varies with frequency',
        'Inner grooves have more distortion',
    ],
}
```

### 18.2 Magnetic Tape
```python
TAPE_SPECS = {
    'frequency_range': '30 Hz - 18 kHz (high bias)',
    'dynamic_range': '55-65 dB',
    'snr': '60-70 dB (Dolby noise reduction)',
    'wow_and_flutter': '0.1-0.3%',
    'noise_floor': '-55 to -65 dB (without NR)',
    'types': {
        'cassette': {'snr': '50-60 dB', 'type': 'Type I/II/IV'},
        'reel_to_reel': {'snr': '55-65 dB', 'type': 'Professional'},
    },
}
```

### 18.3 Digital Audio Comparison
```python
DIGITAL_COMPARISON = {
    '8-bit PCM': {
        'snr': '48 dB',
        'use': 'Early digital audio, telephone',
        'equivalent': 'AM radio quality',
    },
    '12-bit PCM': {
        'snr': '72 dB',
        'use': 'Early digital broadcast',
        'equivalent': 'FM radio quality',
    },
    '16-bit CD': {
        'snr': '96 dB',
        'use': 'CD-quality audio',
        'equivalent': 'Exceeds vinyl and tape',
    },
    '24-bit': {
        'snr': '144 dB',
        'use': 'Professional recording',
        'note': 'Far exceeds hearing range',
    },
    '32-bit float': {
        'snr': '~144 dB effective',
        'use': 'DSP processing',
        'note': 'Headroom, not SNR',
    },
}
```

---

## 19. QUANTIZATION DISTORTION

### 19.1 Distortion Characteristics
```python
def quantization_distortion(quantization_step, signal_amplitude):
    """
    Calculate quantization distortion characteristics.
    
    Args:
        quantization_step: LSB size
        signal_amplitude: Signal RMS amplitude
    
    Returns:
        Dictionary with distortion metrics
    """
    # Theoretical SNR for uniform quantizer
    signal_power = signal_amplitude**2
    noise_power = (quantization_step**2) / 12
    
    if signal_power > 0:
        snr_db = 10 * np.log10(signal_power / noise_power)
    else:
        snr_db = 0
    
    # Quantization noise spectrum
    # For uniform quantizer: white noise
    # For shaped quantizer: shaped noise
    
    return {
        'snr_db': snr_db,
        'noise_power': noise_power,
        'noise_floor_db': 10 * np.log10(noise_power),
    }
```

### 19.2 Harmonic Distortion
```python
def harmonic_distortion_test():
    """
    Test for harmonic distortion from quantization.
    """
    return [
        '1. Generate a pure sine wave',
        '2. Quantize to target bit depth',
        '3. Analyze output spectrum',
        '4. Measure harmonic levels',
        '5. Calculate THD (Total Harmonic Distortion)',
    ]

def thd_calculation(harmonics, fundamental):
    """
    Calculate THD from harmonic levels.
    
    Args:
        harmonics: List of harmonic amplitudes
        fundamental: Fundamental frequency amplitude
    
    Returns:
        THD as percentage
    """
    harmonic_power = sum(h**2 for h in harmonics)
    fundamental_power = fundamental**2
    
    thd = np.sqrt(harmonic_power / fundamental_power) * 100
    
    return thd
```

---

## 20. DITHERING VARIANTS

### 20.1 Dither Types
```python
DITHER_TYPES = {
    'rectangular': {
        'pdf': 'Uniform',
        'range': '-0.5 to +0.5 LSB',
        'sigma': '1/sqrt(12) ≈ 0.289 LSB',
        'use': 'Basic dithering',
    },
    'triangular': {
        'pdf': 'Triangular',
        'range': '-1 to +1 LSB',
        'sigma': '0.5 / sqrt(6) ≈ 0.204 LSB',
        'use': 'Recommended for audio',
    },
    'gaussian': {
        'pdf': 'Normal',
        'sigma': 'Variable',
        'use': 'Noise shaping applications',
    },
    'tpdf_2nd_order': {
        'pdf': 'Double triangular',
        'range': '-2 to +2 LSB',
        'use': 'Higher-order dither',
    },
}
```

### 20.2 Professional Dither Algorithms
```python
class ProfessionalDither:
    """
    Professional dither implementations.
    """
    
    @staticmethod
    def uv22hr(input_bits, output_bits):
        """
        Apogee UV22HR-style noise shaping.
        Reduces audible noise by shaping it to less sensitive frequencies.
        
        Args:
            input_bits: Input bit depth
            output_bits: Output bit depth
        
        Returns:
            Dithered output
        """
        # UV22HR is a proprietary algorithm
        # Approximation based on公开 information
        pass
    
    @staticmethod
    def mbite(input_bits, output_bits):
        """
        iZotope MBIT+ noise shaping.
        Psychoacoustically optimized.
        """
        pass
    
    @staticmethod
    def shibata(input_bits, output_bits):
        """
        Shibata low-pass noise shaping.
        Used in Sony/Philips Super Bit Mapping.
        """
        pass
```

---

## 21. FLOAT vs INTEGER IN MACHINE LEARNING

### 21.1 Neural Audio Processing
```python
def audio_nn_precision():
    """
    Precision requirements for neural audio processing.
    """
    return {
        'inference': {
            'float16': 'Good for GPU inference',
            'float32': 'Standard for quality',
            'int8': 'Quantized inference (quality loss)',
        },
        'training': {
            'float32': 'Standard for training',
            'float16': 'Mixed precision training',
            'float64': 'For gradient precision',
        },
        'audio_specific': {
            'pre-trained models': 'float32 typically',
            'on-device inference': 'int8 quantization possible',
            'quality_impact': 'Varies by model and content',
        },
    }
```

### 21.2 Quantization for Audio Models
```python
def quantize_audio_model(model, target_precision):
    """
    Quantize an audio processing neural network.
    
    Args:
        model: PyTorch/TensorFlow model
        target_precision: 'int8', 'float16', etc.
    
    Returns:
        Quantized model
    """
    if target_precision == 'int8':
        # Post-training quantization
        # May require calibration with representative data
        pass
    elif target_precision == 'float16':
        # Half precision
        pass
    
    return model
```

---

## 22. IMPLEMENTATION EXAMPLES

### 22.1 Audio Buffer Management
```python
class AudioBuffer:
    """
    Fixed-size audio buffer for DSP.
    """
    
    def __init__(self, channels, max_samples, bit_depth=32):
        """
        Args:
            channels: Number of audio channels
            max_samples: Maximum buffer size
            bit_depth: Bit depth (16, 24, 32)
        """
        self.channels = channels
        self.max_samples = max_samples
        self.bit_depth = bit_depth
        
        # Allocate based on type
        if bit_depth == 16:
            self.dtype = np.int16
            self.max_value = 32767
        elif bit_depth == 24:
            self.dtype = np.int32  # Stored as 32-bit
            self.max_value = 8388607
        elif bit_depth == 32:
            self.dtype = np.float32
            self.max_value = 1.0
        else:
            raise ValueError(f"Unsupported bit depth: {bit_depth}")
        
        # Create buffer
        self.buffer = np.zeros((channels, max_samples), dtype=self.dtype)
        self.write_pos = 0
        self.read_pos = 0
        self.samples_in_buffer = 0
    
    def write(self, data):
        """Write samples to buffer."""
        n_samples = min(data.shape[1], self.max_samples - self.samples_in_buffer)
        
        self.buffer[:, self.write_pos:self.write_pos + n_samples] = data[:, :n_samples]
        
        self.write_pos = (self.write_pos + n_samples) % self.max_samples
        self.samples_in_buffer += n_samples
        
        return n_samples
    
    def read(self, n_samples):
        """Read samples from buffer."""
        n_to_read = min(n_samples, self.samples_in_buffer)
        
        data = np.zeros((self.channels, n_to_read), dtype=self.dtype)
        
        end_pos = (self.read_pos + n_to_read) % self.max_samples
        
        if end_pos > self.read_pos:
            data = self.buffer[:, self.read_pos:end_pos]
        else:
            # Wrap around
            part1 = self.buffer[:, self.read_pos:]
            part2 = self.buffer[:, :end_pos]
            data = np.concatenate([part1, part2], axis=1)
        
        self.read_pos = end_pos
        self.samples_in_buffer -= n_to_read
        
        return data
```

### 22.2 Sample Rate Conversion
```python
def convert_bit_depth(audio, from_depth, to_depth):
    """
    Convert audio between bit depths.
    
    Args:
        audio: Input audio
        from_depth: Source bit depth
        to_depth: Target bit depth
    
    Returns:
        Converted audio
    """
    if from_depth == to_depth:
        return audio
    
    # Normalize to float
    if from_depth == 16:
        audio_float = audio.astype(np.float32) / 32768.0
    elif from_depth == 24:
        audio_float = audio.astype(np.float32) / 8388608.0
    elif from_depth == 32 and audio.dtype == np.float32:
        audio_float = audio
    else:
        audio_float = audio.astype(np.float32)
    
    # Convert to target format
    if to_depth == 16:
        output = np.clip(audio_float, -1.0, 1.0)
        output = (output * 32768.0).astype(np.int16)
    elif to_depth == 24:
        output = np.clip(audio_float, -1.0, 1.0)
        output = (output * 8388608.0).astype(np.int32)
    elif to_depth == 32:
        output = np.clip(audio_float, -1.0, 1.0).astype(np.float32)
    else:
        raise ValueError(f"Unsupported target bit depth: {to_depth}")
    
    return output
```

---

*File expanded with: Fixed-point DSP, rounding modes, audio format specs, analog comparison, quantization distortion, dithering variants, ML audio processing, and implementation examples*

---

## 23. PLATFORM-SPECIFIC IMPLEMENTATIONS

### 23.1 macOS Audio Units
```c
// Audio Unit format handling in Core Audio
// macOS supports all standard bit depths through AudioUnit

// Sample format constants:
typedef enum {
    kAudioFormatLinearPCM,
    kAudioFormatAppleLossless,
    // ... other formats
} AudioFormatID;

// Linear PCM flags:
kAudioFormatFlagIsFloat           // Float vs integer
kAudioFormatFlagIsSignedInteger   // Signed vs unsigned
kAudioFormatFlagIsPacked          // Packed vs planar
kAudioFormatFlagIsNonInterleaved  // Interleaved vs planar

// Common formats:
kAudioFormatLinearPCM | 
kAudioFormatFlagIsFloat | 
kAudioFormatFlagIsPacked | 
kAudioFormatNativeEndian
// → Float32 interleaved

kAudioFormatLinearPCM | 
kAudioFormatFlagIsSignedInteger | 
kAudioFormatFlagIsPacked
// → Int16 interleaved
```

### 23.2 Windows Audio APIs
```c
// WAVEFORMATEX structure (Windows)
typedef struct {
    WORD  wFormatTag;      // Audio format (WAVE_FORMAT_PCM, etc.)
    WORD  nChannels;       // Number of channels
    DWORD nSamplesPerSec;  // Sample rate
    DWORD nAvgBytesPerSec; // Bytes per second
    WORD  nBlockAlign;     // Block alignment
    WORD  wBitsPerSample;  // Bits per sample
    WORD  cbSize;          // Extra data size
} WAVEFORMATEX;

// WAVEFORMATEXTENSIBLE for more channels/bit depths
typedef struct {
    WAVEFORMATEX Format;
    union {
        WORD wValidBitsPerSample;
        WORD wSamplesPerBlock;
        WORD wReserved;
    } Samples;
    DWORD dwChannelMask;   // Channel arrangement
    GUID  SubFormat;       // Format GUID
} WAVEFORMATEXTENSIBLE;
```

### 23.3 ALSA (Linux) Format Handling
```c
// snd_pcm_format_t (ALSA)
typedef enum {
    SND_PCM_FORMAT_S8,      // Signed 8-bit
    SND_PCM_FORMAT_S16,     // Signed 16-bit
    SND_PCM_FORMAT_S24,     // Signed 24-bit
    SND_PCM_FORMAT_S32,     // Signed 32-bit
    SND_PCM_FORMAT_FLOAT,   // Float32
    SND_PCM_FORMAT_FLOAT64, // Float64
    // ... other formats
} snd_pcm_format_t;

// Set hardware parameters
int set_hw_params(snd_pcm_t *pcm, int channels, int rate, int bits) {
    snd_pcm_hw_params_t *params;
    snd_pcm_hw_params_alloca(&params);
    
    snd_pcm_hw_params_any(pcm, params);
    snd_pcm_hw_params_set_access(pcm, params, SND_PCM_ACCESS_RW_INTERLEAVED);
    
    if (bits == 16)
        snd_pcm_hw_params_set_format(pcm, params, SND_PCM_FORMAT_S16);
    else if (bits == 24)
        snd_pcm_hw_params_set_format(pcm, params, SND_PCM_FORMAT_S24);
    else if (bits == 32)
        snd_pcm_hw_params_set_format(pcm, params, SND_PCM_FORMAT_FLOAT);
    
    snd_pcm_hw_params_set_channels(pcm, params, channels);
    snd_pcm_hw_params_set_rate_near(pcm, params, &rate, 0);
    
    return snd_pcm_hw_params(pcm, params);
}
```

### 23.4 Android Audio Track
```java
// Android AudioTrack (Java)
import android.media.AudioAttributes;
import android.media.AudioFormat;
import android.media.AudioTrack;

int sampleRate = 48000;
int channelConfig = AudioFormat.CHANNEL_OUT_STEREO;
int audioFormat = AudioFormat.ENCODING_PCM_16BIT;  // or PCM_16BIT, PCM_24BIT_PACKED, PCM_32BIT

int bufferSize = AudioTrack.getMinBufferSize(
    sampleRate, channelConfig, audioFormat);

AudioTrack track = new AudioTrack.Builder()
    .setAudioAttributes(new AudioAttributes.Builder()
        .setUsage(AudioAttributes.USAGE_MEDIA)
        .setContentType(AudioAttributes.CONTENT_TYPE_MUSIC)
        .build())
    .setAudioFormat(new AudioFormat.Builder()
        .setEncoding(audioFormat)
        .setSampleRate(sampleRate)
        .setChannelMask(channelConfig)
        .build())
    .setBufferSizeInBytes(bufferSize)
    .setTransferMode(AudioTrack.MODE_STREAM)
    .build();

// PCM_24BIT_PACKED: 3 bytes per sample, packed
// PCM_32BIT: 4 bytes per sample (24-bit padded to 32-bit)
```

---

## 24. HIGH-RESOLUTION AUDIO SPECIFICATIONS

### 24.1 High-Res Audio Standards
```python
HIGH_RES_STANDARDS = {
    'CD_quality': {
        'sample_rate': 44100,
        'bit_depth': 16,
        'channels': 2,
        'duration_74min': 783 MB,
    },
    'DVD_quality': {
        'sample_rate': 48000,
        'bit_depth': 24,
        'channels': 2,
    },
    '96_24': {
        'sample_rate': 96000,
        'bit_depth': 24,
        'channels': 2,
        'quality': 'High-resolution',
    },
    '192_24': {
        'sample_rate': 192000,
        'bit_depth': 24,
        'channels': 2,
        'quality': 'Studio quality',
    },
    'DXD': {
        'sample_rate': 352800,
        'bit_depth': 24,
        'channels': 2,
        'note': 'Digital eXtreme Definition',
    },
    'DSD64': {
        'type': 'DSD',
        'sample_rate': '2.8224 MHz',
        'note': '1-bit, direct stream digital',
    },
    'DSD128': {
        'type': 'DSD',
        'sample_rate': '5.6448 MHz',
        'note': 'Double DSD',
    },
    'DSD256': {
        'type': 'DSD',
        'sample_rate': '11.2896 MHz',
        'note': 'Quad DSD',
    },
}
```

### 24.2 Bitrate Calculations
```python
def calculate_bitrate(sample_rate, bit_depth, channels):
    """
    Calculate bitrate for uncompressed audio.
    """
    return sample_rate * bit_depth * channels

# Examples:
cd_rate = calculate_bitrate(44100, 16, 2)  # 1,411,200 bps
dvd_rate = calculate_bitrate(48000, 24, 2)  # 2,304,000 bps
 hires_96_24 = calculate_bitrate(96000, 24, 2)  # 4,608,000 bps
 hires_192_24 = calculate_bitrate(192000, 24, 2)  # 9,216,000 bps
 dxd = calculate_bitrate(352800, 24, 2)  # 16,934,400 bps

print(f"CD quality: {cd_rate:,} bps ({cd_rate/1000:.1f} kbps)")
print(f"96/24: {hires_96_24:,} bps ({hires_96_24/1000:.1f} kbps)")
print(f"192/24: {hires_192_24:,} bps ({hires_192_24/1000:.1f} kbps)")
print(f"DXD: {dxd:,} bps ({dxd/1000000:.1f} Mbps)")
```

### 24.3 Storage Requirements
```python
def storage_requirement(duration_minutes, sample_rate, bit_depth, channels):
    """
    Calculate storage requirement in MB.
    """
    duration_seconds = duration_minutes * 60
    bytes_per_sample = bit_depth / 8
    
    total_bytes = (sample_rate * bytes_per_sample * channels * 
                    duration_seconds)
    
    return total_bytes / (1024 * 1024)  # MB

# Examples:
storage_cd = storage_requirement(74, 44100, 16, 2)  # ~650 MB
storage_96_24 = storage_requirement(74, 96000, 24, 2)  # ~2.4 GB
storage_192_24 = storage_requirement(74, 192000, 24, 2)  # ~4.8 GB

print(f"74 min CD: {storage_cd:.1f} MB")
print(f"74 min 96/24: {storage_96_24:.1f} GB")
print(f"74 min 192/24: {storage_192_24:.1f} GB")
```

---

## 25. DAC/ADC PERFORMANCE VS BIT DEPTH

### 25.1 DAC Specifications
```python
DAC_PERFORMANCE = {
    'budget': {
        'snr_db': '90-100 dB',
        'thd_percent': '0.01-0.1%',
        'bit_depth_adequate': '16-bit',
    },
    'mid_range': {
        'snr_db': '100-110 dB',
        'thd_percent': '0.001-0.01%',
        'bit_depth_adequate': '24-bit',
    },
    'high_end': {
        'snr_db': '110-120 dB',
        'thd_percent': '0.0001-0.001%',
        'bit_depth_adequate': '24-bit',
    },
    'measurement': {
        'snr_db': '120+ dB',
        'thd_percent': '<0.0001%',
        'bit_depth_adequate': '24-bit',
    },
}
```

### 25.2 ADC Specifications
```python
ADC_PERFORMANCE = {
    'consumer': {
        'effective_bits': '16-18 bits',
        'snr_db': '90-100 dB',
        'typical_use': 'Consumer audio',
    },
    'semi_pro': {
        'effective_bits': '18-20 bits',
        'snr_db': '100-110 dB',
        'typical_use': 'Semi-professional recording',
    },
    'professional': {
        'effective_bits': '20-22 bits',
        'snr_db': '110-120 dB',
        'typical_use': 'Professional studio',
    },
    'measurement': {
        'effective_bits': '22-24 bits',
        'snr_db': '120+ dB',
        'typical_use': 'Measurement and analysis',
    },
}
```

### 25.3 Effective vs Actual Bit Depth
```python
def effective_bits(snr_db):
    """
    Calculate effective bit depth from SNR.
    
    Formula: Effective bits = (SNR - 1.76) / 6.02
    """
    return (snr_db - 1.76) / 6.02

# Examples:
snr_90 = effective_bits(90)  # ~14.7 bits
snr_100 = effective_bits(100)  # ~16.3 bits
snr_110 = effective_bits(110)  # ~18.0 bits
snr_120 = effective_bits(120)  # ~19.6 bits
snr_140 = effective_bits(140)  # ~23.0 bits

print(f"90 dB SNR: {snr_90:.1f} effective bits")
print(f"100 dB SNR: {snr_100:.1f} effective bits")
print(f"110 dB SNR: {snr_110:.1f} effective bits")
print(f"120 dB SNR: {snr_120:.1f} effective bits")
print(f"140 dB SNR: {snr_140:.1f} effective bits")
```

---

## 26. TRUNCATION AND DITHERING

### 26.1 When Truncation Is Acceptable
```python
def truncation_acceptable(input_bits, output_bits, signal_level_dbfs):
    """
    Determine if truncation is acceptable.
    
    Truncation is generally acceptable when:
    1. Signal level is well above truncation noise
    2. Dithering is not practical (real-time constraints)
    3. Content is not critical (background music)
    
    Args:
        input_bits: Source bit depth
        output_bits: Target bit depth
        signal_level_dbfs: Signal level in dBFS
    
    Returns:
        (acceptable, reason)
    """
    bits_lost = input_bits - output_bits
    noise_floor_db = -6.02 * bits_lost - 1.76  # Quantization noise
    
    if signal_level_dbfs > noise_floor_db + 30:  # 30 dB headroom
        return True, "Signal well above truncation noise"
    elif signal_level_dbfs > noise_floor_db + 20:
        return True, "Marginal, but acceptable for most content"
    else:
        return False, "Truncation may be audible"
```

### 26.2 Dither Requirements
```python
def dither_required(input_bits, output_bits, signal_characteristics):
    """
    Determine if dithering is required.
    
    Dither is required when:
    1. Reducing bit depth by 4+ bits
    2. Signal has quiet passages
    3. High-fidelity is required
    4. Program is for broadcast/release
    """
    bits_reduced = input_bits - output_bits
    
    if bits_reduced >= 4:
        return True, "Significant bit reduction"
    elif signal_characteristics['has_quiet_passages']:
        return True, "Quiet passages need dither"
    elif signal_characteristics['critical_listening']:
        return True, "Critical listening material"
    else:
        return False, "Dither optional"
```

---

## 27. MATHEMATICAL REFERENCE

### 27.1 Constants
```python
# Audio precision constants
AUDIO_CONSTANTS = {
    'PI': 3.141592653589793,
    'TWO_PI': 2 * 3.141592653589793,
    'HALF_PI': 3.141592653589793 / 2,
    
    # Decibel conversions
    'DB_6_02_PER_BIT': 6.02,  # dB per bit (linear)
    'DB_20_PER_DECADE': 20,  # dB per decade (voltage/power)
    'DB_10_PER_DECADE': 10,  # dB per decade (power)
    
    # Common audio levels
    'DBFS_SINE_MAX': 0.0,  # Full-scale sine
    'DBFS_SINE_PEAK': -3.01,  # Peak of sine at full scale
    'DBFS_SQUARE_MAX': 0.0,  # Full-scale square
    'DBFS_SQUARE_RMS': -3.01,  # RMS of full-scale square
    
    # Standard reference
    'DB_SPL_REF': 20e-6,  # 20 micropascals (0 dB SPL)
}
```

### 27.2 Conversion Formulas
```python
def linear_to_dbfs(linear):
    """Linear (0-1) to dBFS."""
    if linear <= 0:
        return -float('inf')
    return 20 * np.log10(linear)

def dbfs_to_linear(dbfs):
    """dBFS to linear (0-1)."""
    return 10 ** (dbfs / 20)

def amplitude_to_db(amplitude):
    """Amplitude to dB."""
    if amplitude <= 0:
        return -float('inf')
    return 20 * np.log10(amplitude)

def power_to_db(power):
    """Power to dB."""
    if power <= 0:
        return -float('inf')
    return 10 * np.log10(power)

def db_to_amplitude(db):
    """dB to amplitude."""
    return 10 ** (db / 20)

def db_to_power(db):
    """dB to power."""
    return 10 ** (db / 10)
```

---

*File expanded with: Platform-specific implementations, high-resolution audio specifications, DAC/ADC performance, truncation guidelines, and mathematical reference*
