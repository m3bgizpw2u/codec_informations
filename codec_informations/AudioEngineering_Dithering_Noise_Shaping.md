# Dithering and Noise Shaping in Audio — Deep Technical Reference
> **Category:** AudioEngineering
> **Topic:** Dithering, Noise Shaping, Quantization, Bit-Depth Reduction
> **Related Codecs:** PCM, DSD, WAV, AIFF, FLAC, ALAC, AAC, MP3, Opus
> **FFmpeg Modules:** libswresample (dither options), libavresample
> **Standards:** IEC 60958, AES3, EBU R68, ITU-R BS.775
> **References:** Wannamaker, Lipshitz, Vanderkooy — "Theory of Dithered Quantization"

---

## 1. HISTORICAL CONTEXT & MOTIVATION

### 1.1 The Quantization Problem

When converting from a high-resolution to a lower-resolution digital format — for example, from 24-bit to 16-bit PCM, or from 32-bit floating point to 16-bit integer — the number of available discrete amplitude levels decreases. Each input sample must be mapped to the nearest available output level. This process is called **quantization**.

**Quantization step size (LSB — Least Significant Bit):**
$$\Delta = \frac{2^{X_{max}}}{2^{N}} = \frac{2 \times X_{max}}{2^N}$$

For signed 16-bit PCM with range [-1, 1):
$$\Delta = \frac{2}{2^{16}} = \frac{1}{32768} \approx 0.0000305$$

For signed 24-bit PCM:
$$\Delta = \frac{2}{2^{24}} = \frac{1}{8388608} \approx 0.000000119$$

**The quantization error** (also called quantization noise) is:
$$e[n] = x_Q[n] - x[n]$$

where $x_Q[n]$ is the quantized value and $x[n]$ is the original.

### 1.2 Why Dither Is Necessary

Without dither, quantization error is **correlated** with the input signal. This correlation manifests as harmonic distortion products — the quantization error tracks the input waveform and introduces spectral components that are mathematically related to the input frequencies.

**Example:** A 997 Hz sine wave quantized to 16-bit without dither produces harmonics at 1994 Hz, 2991 Hz, 3988 Hz... These are not random noise; they are deterministic artifacts that are easily heard, especially on low-frequency tonal content.

**The correlation problem:**
$$e[n] = x_Q[n] - x[n]$$
$$e[n] \text{ is NOT independent of } x[n] \text{ without dither}$$

Dither breaks this correlation by adding a small amount of random noise before quantization. The quantization error becomes decorrelated from the input signal, turning harmonic distortion into broadband noise that is less perceptually disturbing.

### 1.3 When Dithering Is Required

Dithering is required whenever:
1. **Bit depth is reduced** — from 24-bit to 16-bit, from 32-bit float to 24-bit PCM, etc.
2. **Format changes** — floating point to integer
3. **Sample rate conversion** involves intermediate format conversion that reduces effective precision
4. **Lossy re-encoding** — any lossy codec reduces effective bit depth at each generation

**Dithering is NOT needed when:**
- Bit depth is increased (no quantization occurs)
- Output format has same or higher effective precision than input
- The conversion is from integer PCM to the same integer PCM at the same bit depth
- Lossless-to-lossless conversion at same or higher bit depth

---

## 2. THE MATHEMATICS OF DITHERED QUANTIZATION

### 2.1 Quantization Model

The uniform quantizer with input range $[-X_{max}, X_{max}]$ and $N$ bits:

$$Q(x) = \Delta \cdot \left\lfloor \frac{x}{\Delta} + \frac{1}{2} \right\rfloor$$

This is the **mid-tread** quantizer (round to nearest). For mid-rise quantizers, the rounding offset is $\Delta/2$ instead.

**Without dither:**
$$x_Q[n] = Q(x[n])$$

**With dither:**
$$x_Q[n] = Q(x[n] + d[n])$$
where $d[n]$ is the dither signal, added before quantization.

### 2.2 Probability Density Functions

**RPDF (Rectangular Probability Density Function):**
$$p_{RPDF}(x) = \begin{cases} 1 & \text{for } -\frac{\Delta}{2} \leq x \leq \frac{\Delta}{2} \\ 0 & \text{otherwise} \end{cases}$$

Also called **uniform dither**. Each value in the range $[-\Delta/2, \Delta/2]$ is equally likely.

**TPDF (Triangular Probability Density Function):**
The TPDF is the convolution of two RPDFs:
$$p_{TPDF}(x) = p_{RPDF} * p_{RPDF}(x)$$

Resulting in a triangular distribution:
$$p_{TPDF}(x) = \begin{cases} \frac{x + \Delta}{\Delta^2} + \frac{1}{\Delta} & \text{for } -\Delta \leq x \leq 0 \\ \frac{\Delta - x}{\Delta^2} + \frac{1}{\Delta} & \text{for } 0 \leq x \leq \Delta \\ 0 & \text{otherwise} \end{cases}$$

**Visual comparison:**
```
RPDF:           TPDF:
 █              █
 █              ███
 █████         ███████
[-Δ/2, Δ/2]   [-Δ, Δ]
```

### 2.3 Why TPDF Over RPDF?

**RPDF dither** (rectangular/uniform) provides first-order noise shaping but has a limitation: it only linearizes quantization when the input is uniformly distributed over the quantization interval. For tonal signals (sine waves), RPDF dither can leave residual harmonic distortion because the quantization error distribution depends on where the signal sits within the quantization step.

**TPDF dither** (triangular) provides **second-order noise shaping**. The extra integration of two RPDF sources means:
1. The quantization error $e[n] = Q(x[n] + d_1[n] + d_2[n]) - x[n]$ is uncorrelated with the input signal to a higher degree
2. The error spectrum is whitened — all harmonics are removed, replaced by noise floor
3. The probability of the quantized value landing on exactly the right side of the true value is proportional to the distance from the true value, providing proper linearization

**Formal result (Wannamaker, Lipshitz, Vanderkooy, 1992):**
For TPDF dither with sufficient amplitude (range $[-\Delta, \Delta]$), the quantized signal $x_Q$ is a **linearly related** function of the input $x$. Specifically:
$$E\{x_Q[n]\} = E\{x[n]\}$$
$$E\{x_Q[n] \cdot x[m]\} \propto E\{x[n] \cdot x[m]\} \text{ (for uncorrelated dither)}$$

This means the dithered quantizer is a **linear system** for signals whose spectral content is below the dither noise floor — no harmonic distortion, only added noise.

### 2.4 Dither Amplitude

The dither amplitude must span **exactly one quantization step** (TPDF: from $-\Delta$ to $+\Delta$, which is $2 \times \Delta$ total range) to provide optimal linearization:

- **TPDF amplitude:** Range of $[-\Delta, \Delta]$, peak probability at 0
- **RPDF amplitude:** Range of $[-\Delta/2, \Delta/2]$, uniform probability

For 16-bit PCM with full-scale range $[-1, +1)$:
$$\Delta_{16bit} = \frac{2}{2^{16}} = \frac{1}{32768}$$

Dither must span approximately ±1 LSB (for TPDF) or ±0.5 LSB (for RPDF).

---

## 3. DITHER IMPLEMENTATION

### 3.1 Generating TPDF Dither

**Method 1: Sum of two uniform random numbers**

```c
#include <stdlib.h>

// Generate one TPDF sample from two RPDF samples
float tpdf_sample(void) {
    // RPDF1: uniform in [-0.5, +0.5) LSB
    float r1 = ((float)rand() / RAND_MAX) - 0.5f;
    // RPDF2: uniform in [-0.5, +0.5) LSB
    float r2 = ((float)rand() / RAND_MAX) - 0.5f;
    // TPDF: sum of two RPDFs = triangular distribution
    return r1 + r2;  // Range: [-1, +1) LSB
}
```

**Method 2: Box-Muller for Gaussian (professional implementations)**

Some high-end dithering systems use Gaussian-shaped dither rather than pure TPDF. The rationale is that the ear's sensitivity to noise near 0 Hz is reduced, and Gaussian dither concentrates more energy around the center where the ear is less sensitive per unit energy.

```c
#include <math.h>

float gaussian_sample(void) {
    // Box-Muller transform for Gaussian noise
    float u1 = (float)rand() / RAND_MAX;
    float u2 = (float)rand() / RAND_MAX;
    return sqrtf(-2.0f * logf(u1)) * cosf(2.0f * M_PI * u2);
}
```

**Proper scaling:**

```c
float dither_amplitude_s16 = 0.5f;      // ±0.5 LSB for TPDF
float dither_amplitude_s24 = 0.5f;      // ±0.5 LSB in 24-bit context
float dither_amplitude_s32 = 0.5f;      // ±0.5 LSB in 32-bit context

float dithered_sample = input_sample + (tpdf_sample() * dither_amplitude);
int16_t quantized = (int16_t)lrintf(dithered_sample);
```

### 3.2 Complete Dither Pipeline

```c
typedef enum {
    DITHER_NONE,
    DITHER_RECTANGULAR,     // RPDF — flat noise, first-order
    DITHER_TRIANGULAR,      // TPDF — triangular, second-order
    DITHER_TRIANGULAR_HP,   // TPDF with high-pass filter (dc removed)
    DITHER_TRIANGULAR_NS,   // TPDF with noise shaping
} DitherMethod;

typedef struct {
    DitherMethod method;
    float amplitude;           // Dither amplitude in LSBs
    // State for TPDF (two random generators per channel)
    uint32_t rng_state[2][8];   // Max 8 channels
    float prev_dither[8];       // Previous dither for HP filter
    // Noise shaping coefficients (NS variant)
    float ns_b[3];              // Feedforward coefficients
    float ns_a[3];              // Feedback coefficients
    float ns_delay[8][2];       // Filter delay states
} DitherState;

float dither_generate(DitherState *state, int channel) {
    float d = 0.0f;

    switch (state->method) {
    case DITHER_RECTANGULAR:
        d = ((float)rand_r(&state->rng_state[0][channel]) / RAND_MAX) - 0.5f;
        break;
    case DITHER_TRIANGULAR:
        d = ((float)rand_r(&state->rng_state[0][channel]) / RAND_MAX) - 0.5f;
        d += ((float)rand_r(&state->rng_state[1][channel]) / RAND_MAX) - 0.5f;
        break;
    case DITHER_TRIANGULAR_HP: {
        float tpdf = ((float)rand_r(&state->rng_state[0][channel]) / RAND_MAX) - 0.5f;
        tpdf += ((float)rand_r(&state->rng_state[1][channel]) / RAND_MAX) - 0.5f;
        // High-pass filter: remove DC
        d = tpdf - state->prev_dither[channel];
        state->prev_dither[channel] = tpdf;
        break;
    }
    default:
        break;
    }
    return d * state->amplitude;
}
```

### 3.3 Applying Dither Before Quantization

```c
void quantize_with_dither(float *input, int16_t *output,
                          long frames, int channels,
                          DitherState *dither) {
    for (long f = 0; f < frames; f++) {
        for (int ch = 0; ch < channels; ch++) {
            float sample = input[f * channels + ch];
            float d = dither_generate(dither, ch);
            float dithered = sample + d;
            output[f * channels + ch] = (int16_t)lrintf(dithered);
        }
    }
}
```

**Key principle:** Dither is added to the input signal in the input's numeric domain, THEN quantized. The amplitude of the dither must match the LSB size of the OUTPUT format.

---

## 4. NOISE SHAPING

### 4.1 Concept: Frequency-Dependent Error Weighting

Noise shaping extends the dithering concept by applying a frequency weighting to the quantization error. The idea is simple: the ear is less sensitive to noise at high frequencies and near 20 kHz, and more sensitive in the midrange (2–5 kHz) and low frequencies. Noise shaping pushes quantization noise into the less-sensitive frequency bands.

**Basic feedback structure:**
```
Input x[n] ──→(+)──→ Quantizer Q ──→ Output y[n]
              ↑                   │
              │                   │
              └────── Error e[n] ─┘
                      │
                      ▼
              Noise Shaping Filter H(z)
                      │
                      ▼
              Shaped noise n[n]
```

**Transfer functions:**
$$Y(z) = X(z) + E(z) \cdot (1 - H(z))$$
$$N(z) = E(z) \cdot (1 - H(z))$$

Where $H(z)$ is the noise shaping filter. If $|H(z)| \approx 1$ in the band we want to suppress noise, then $N(z) \approx 0$ in that band — noise is "shaped" away.

### 4.2 First-Order Noise Shaping

First-order noise shaping uses a single-sample delay in the feedback path:
$$H(z) = z^{-1}$$

Therefore:
$$N(z) = E(z) \cdot (1 - z^{-1})$$

In the frequency domain:
$$|1 - e^{-j\omega}| = 2|\sin(\omega/2)|$$

This gives a **high-shelf** noise shaping — noise is boosted at high frequencies and reduced at low frequencies. This is the opposite of what we want for audio (we want to reduce midrange and high-frequency noise where the ear is most sensitive).

### 4.3 Second-Order Noise Shaping

Second-order shaping uses:
$$H(z) = 2z^{-1} - z^{-2}$$

Resulting in:
$$N(z) = E(z) \cdot (1 - 2z^{-1} + z^{-2}) = E(z) \cdot (1 - z^{-1})^2$$

Frequency response:
$$|1 - e^{-j\omega}|^2 = 4\sin^2(\omega/2)$$

This provides even more aggressive high-frequency noise boosting — still not ideal for audio applications.

### 4.4 Psychoacoustic Noise Shaping

For audio, we need the inverse: noise should be **reduced** in the most sensitive frequency bands (roughly 1–6 kHz) and **moved** to less sensitive bands (low frequencies < 200 Hz and very high frequencies > 15 kHz).

The Absolute Threshold of Hearing (ATH) model defines the minimum audible level at each frequency. The noise shaping filter should produce less noise where ATH is lower (more sensitive) and more noise where ATH is higher (less sensitive).

**ATH formula (ISO 226:2003, simplified):**
$$ATH(f) = 3.64 \cdot \left(\frac{f}{1000}\right)^{-0.8} - 6.5 \cdot e^{-0.6\left(\frac{f}{1000} - 3.3\right)^2} + 10^{-3} \cdot \left(\frac{f}{1000}\right)^4 \text{ dB SPL}$$

| Frequency | ATH (dB SPL) | Ear Sensitivity |
|-----------|---------------|-----------------|
| 20 Hz | ~78 dB | Very insensitive |
| 100 Hz | ~38 dB | Low sensitivity |
| 1 kHz | ~4.8 dB | Most sensitive |
| 3 kHz | ~0 dB | Most sensitive |
| 10 kHz | ~15 dB | Moderate sensitivity |
| 15 kHz | ~25 dB | Less sensitive |
| 20 kHz | ~50 dB | Very insensitive |

A well-designed noise shaper reduces noise in the 2–5 kHz region by 10–15 dB relative to flat dither, at the cost of increasing noise above 15 kHz (where the ear is less sensitive) by a similar amount.

### 4.5 Industry Noise Shaping Curves

Several commercial noise shaping curves have been developed:

| Name | Developer | Shape | Peak Attenuation |
|------|-----------|-------|-----------------|
| POW-r 1 | Weiss Engineering | Mild high-shelf | ~3 dB at 1 kHz |
| POW-r 2 | Weiss Engineering | Medium | ~6 dB at 2 kHz |
| POW-r 3 | Weiss Engineering | Aggressive | ~10 dB at 2 kHz |
| MBIT+ | iZotope | Psychoacoustic | Variable |
| TPDF | Standard | Flat (no shaping) | 0 dB |
| HP-TPDF | Standard | High-pass only | DC removed |

**FFmpeg noise shaping coefficients for triangular_ns:**

The triangular_ns (noise-shaped triangular) dither in FFmpeg uses different filter coefficients depending on sample rate:

```c
// For 44100 Hz sample rate — 256-coefficient FIR noise shaper
static const float ns_44_coef_b[] = { /* 256 filter coefficients */ };
static const float ns_44_coef_a[] = { /* FIR — a[0]=1 only */ };

// For 48000 Hz — slightly different coefficients
static const float ns_48_coef_b[] = { /* 256 filter coefficients */ };
static const float ns_48_coef_a[] = { /* FIR */ };
```

[NEEDS VERIFICATION] The exact numerical values of FFmpeg's triangular_ns filter coefficients are not publicly documented. The filter is a 256-tap FIR design optimized for psychoacoustic weighting at 44100 and 48000 Hz.

### 4.6 Noise Shaping Filter Design

A practical audio noise shaper can be designed as:

```c
// Second-order IIR noise shaper with psychoacoustic weighting
typedef struct {
    float b0, b1, b2;  // Feedforward coefficients
    float a1, a2;      // Feedback coefficients
    float x1, x2;      // Input delay states
    float y1, y2;      // Output delay states
} NoiseShaper;

// Design a shaper that attenuates 3–5 kHz by ~8 dB
NoiseShaper ns_create(float atten_db, float center_hz, float fs) {
    NoiseShaper ns = {0};

    // Convert to angular frequency
    float w0 = 2.0f * M_PI * center_hz / fs;
    float alpha = sinf(w0) / (2.0f * powf(10.0f, atten_db / 40.0f));

    // Peaking EQ coefficients (for noise shaping)
    ns.b0 = 1.0f + alpha;
    ns.b1 = -2.0f * cosf(w0);
    ns.b2 = 1.0f - alpha;
    ns.a1 = ns.b1;
    ns.a2 = -alpha * 2.0f;

    // Normalize
    float norm = 1.0f / (1.0f + alpha);
    ns.b0 *= norm; ns.b1 *= norm; ns.b2 *= norm;

    return ns;
}

float ns_process(NoiseShaper *ns, float error) {
    float output = ns->b0 * error + ns->x1 * ns->b1 + ns->x2 * ns->b2
                 - ns->y1 * ns->a1 - ns->y2 * ns->a2;

    ns->x2 = ns->x1; ns->x1 = error;
    ns->y2 = ns->y1; ns->y1 = output;

    return output;
}
```

---

## 5. TONAL CONTENT AND DITHER

### 5.1 The DC and Low-Frequency Problem

The most challenging case for undithered quantization is a **DC signal** (constant value) or a **low-frequency sine wave**. With no dither:

- A DC signal at exactly 0.25 LSB above a quantization boundary will always round UP
- A DC signal at exactly 0.75 LSB below a quantization boundary will always round DOWN
- A low-frequency sine wave near 0 Hz will have its negative half quantized more heavily than its positive half (asymmetric clipping)

**Example:** A 1 Hz sine wave at 16-bit, with peak amplitude of 0.5 LSB, will:
- Be quantized entirely to 0 (silence) without dither
- With TPDF dither, produce a correct average level of -3 dB relative to the undithered output

### 5.2 Tonal Distortion Without Dither

A sine wave at frequency $f_0$ quantized without dither produces harmonic distortion at multiples of $f_0$:

$$e_{harmonic}[n] \approx \sum_{k=1}^{\infty} a_k \sin(2\pi k f_0 n / f_s)$$

The amplitude of harmonic $k$ relative to the fundamental depends on the sine wave's amplitude relative to the quantization step $\Delta$. A sine at 99% of the quantization threshold will produce very little distortion, while a sine at 50% of the threshold will produce significant 2nd and 3rd harmonics.

### 5.3 Tonal Distortion With TPDF Dither

With properly scaled TPDF dither, the harmonic distortion is completely eliminated. The quantization error becomes a white noise process (uniform power spectral density) that is uncorrelated with the input signal. The output spectrum consists of:
1. The original signal (preserved)
2. A flat noise floor (added by dither)
3. NO harmonic distortion products

This is the key advantage of dithering for music with tonal content: distortion products that would obscure quiet passages are replaced by a barely-audible noise floor.

### 5.4 Optimal Dither for Tonal Material

For material with significant tonal content (classical music, acoustic instruments, vocals):

| Dither Type | Harmonic Distortion | Noise Floor | Recommendation |
|-------------|---------------------|-------------|----------------|
| None | Strong | None | Never use |
| RPDF | Weak | Flat | Minimum acceptable |
| TPDF | None | Flat | Recommended minimum |
| TPDF + HP | None | Flat (no DC) | Recommended |
| TPDF + NS | None | Shaped | Best for music |

For material with mostly noise-like content (drums, percussion, noise):

| Dither Type | Notes |
|-------------|-------|
| RPDF | Acceptable — less dither noise energy |
| TPDF | Fine — tonal content is minimal |

---

## 6. WORD LENGTH REDUCTION

### 6.1 From 24-bit to 16-bit PCM

The most common word length reduction scenario. 24-bit recording at -60 dBFS has 8 bits of headroom above 16-bit's noise floor. Proper dithering preserves this dynamic range as a noise floor rather than losing it as distortion.

**SNR improvement with dithering:**
- 16-bit without dither: ~78 dB SNR (harmonic distortion present)
- 16-bit with TPDF dither: ~93 dB SNR (noise floor only)
- Theoretical 16-bit SNR: 6.02 × 16 = 96.3 dB

The 3 dB difference between theoretical and achieved SNR with TPDF dither comes from the dither noise itself being spread across the full Nyquist bandwidth.

### 6.2 From 32-bit Float to 24-bit PCM

32-bit floating point has 24-bit mantissa precision (same as 24-bit PCM). However, floating point's mantissa is not uniformly distributed — precision is relative to signal level. When converting from 32-bit float to 24-bit PCM:

1. The floating-point mantissa must be rounded to 24 bits
2. This is equivalent to 24-bit quantization (one step up from 16-bit)
3. Dithering is still beneficial, especially for quiet signals
4. The effective SNR is approximately 144 dB (32-bit float's theoretical range)

```bash
# 32-bit float to 24-bit PCM with dithering
ffmpeg -i input_s32le.wav \
  -af "aformat=s32p:dither_method=triangular" \
  -c:a pcm_s24le output_s24.wav
```

### 6.3 From 24-bit to 20-bit PCM

Used in some broadcast and professional applications:

| Source Bit Depth | Target Bit Depth | Dither Recommended | Method |
|------------------|------------------|-------------------|--------|
| 32-bit float | 24-bit PCM | Yes | TPDF |
| 24-bit PCM | 20-bit PCM | Yes | TPDF |
| 24-bit PCM | 16-bit PCM | Yes | TPDF or TPDF+NS |
| 20-bit PCM | 16-bit PCM | Yes | TPDF |
| 16-bit PCM | 16-bit PCM | No | N/A (no reduction) |

### 6.4 Multiple-Generation Dithering

When audio undergoes multiple encoding passes (e.g., CD → MP3 → transcoded), dither should ideally be applied at each quantization step. However:
1. Each dithering stage adds noise
2. After 3–4 generations, the cumulative noise floor rises
3. Best practice: dither only at the final output stage (for delivery formats)
4. For intermediate processing, keep higher bit depth and dither only at the final conversion

---

## 7. FFMPEG DITHERING OPTIONS

### 7.1 FFmpeg Dither Methods

FFmpeg's `-af aresample` and format conversion pipeline support the following dither methods:

```bash
# Dither method syntax
ffmpeg -i input.wav \
  -af "aformat=s16p:dither_method=rectangular" \
  output.wav

ffmpeg -i input.wav \
  -af "aformat=s16p:dither_method=triangular" \
  output.wav

ffmpeg -i input.wav \
  -af "aformat=s16p:dither_method=triangular_hp" \
  output.wav

ffmpeg -i input.wav \
  -af "aformat=s16p:dither_method=triangular_ns" \
  output.wav
```

**Dither methods comparison:**

| Method | Description | Noise Type | Best For |
|--------|-------------|-----------|----------|
| none | No dither | — | Never for audio |
| rectangular | RPDF dither | White, uniform | Noise-like content |
| triangular | TPDF dither | White, triangular | Tonal content |
| triangular_hp | TPDF with DC removal | High-passed triangular | Measurement, DC-sensitive |
| triangular_ns | TPDF with noise shaping | Psychoacoustically shaped | Final delivery (music) |

### 7.2 aformat Filter — Bit Depth and Dither

```bash
# Convert from 24-bit to 16-bit with triangular dither
ffmpeg -i input_24bit.wav \
  -af "aformat=s16p:dither_method=triangular" \
  output_16bit.wav

# Convert from 32-bit float to 24-bit with triangular HP dither
ffmpeg -i input_f32.wav \
  -af "aformat=s24p:dither_method=triangular_hp" \
  output_24bit.wav

# Resample and reduce bit depth in one step with NS dither
ffmpeg -i input.wav \
  -af "aresample=out_sample_rate=48000,aformat=s16p:dither_method=triangular_ns" \
  output.wav
```

### 7.3 output_sample_bits Option

For SWR (native resampler), FFmpeg provides `output_sample_bits` which controls the effective output bit depth for dithering purposes:

```bash
# Use 20-bit output quantization with triangular dither
ffmpeg -i input.wav \
  -af "aresample=out_sample_rate=48000:output_sample_bits=20" \
  output.wav
```

This is useful when you want to apply dithering to a specific effective bit depth that differs from the actual output format's bit depth.

### 7.4 FFmpeg dither.c Implementation Details

FFmpeg's dither implementation in `libavresample/dither.c` (deprecated) and `libswresample/dither.c` (current) follows these steps:

```c
// 1. Generate two RPDF samples (one per random generator)
float r1 = lfg_get_float(lfg);  // Linear Feedback Shift Register
float r2 = lfg_get_float(lfg);

// 2. Sum for TPDF
float tpdf = r1 + r2;  // Range: approximately [-1, +1]

// 3. Scale to 0.5 LSB for the output bit depth
float dither = tpdf * (1.0f / (1ULL << (output_bits - 1)));

// 4. Add dither to the sample
float dithered_sample = sample + dither;

// 5. Quantize (round)
int32_t quantized = lrintf(dithered_sample);
```

For `triangular_hp`, an additional high-pass filter is applied to the TPDF dither:
```c
float hp_tpdf = tpdf - prev_tpdf;  // Single-pole high-pass
prev_tpdf = tpdf;
```

### 7.5 FFmpeg libswresample Dither API

```c
#include <libswresample/swresample.h>

SwrContext *swr = swr_alloc_set_opts(
    NULL,                           // existing context (or NULL)
    AV_CH_LAYOUT_STEREO,            // output channel layout
    AV_SAMPLE_FMT_S16P,             // output format (16-bit)
    48000,                          // output sample rate
    AV_CH_LAYOUT_STEREO,            // input channel layout
    AV_SAMPLE_FMT_FLTP,             // input format (32-bit float)
    44100,                          // input sample rate
    0, NULL);

// Set dither method
av_opt_set_int(swr, "dither_method", AV_RESAMPLE_DITHER_TRIANGULAR, 0);
av_opt_set_int(swr, "dither_retry", 1, 0);  // Allow retry on EAGAIN

swr_init(swr);

// Process (dithering happens internally during format conversion)
swr_convert(swr, output, output_frames,
            input, input_frames);
```

---

## 8. INDUSTRY STANDARDS AND GUIDELINES

### 8.1 EBU R68 — Dithering in Broadcast

The European Broadcasting Union (EBU) standard R68 specifies that audio for digital broadcasting should be:
- Dithered to 16 bits (or the target delivery format's bit depth)
- Peak level: 0 dBFS = -18 dB relative to full-scale PPM
- Dither should be TPDF, applied at the final output stage

**EBU R68 also specifies:**
- Silence or near-silence should be dithered to prevent DC accumulation
- Test tones should be dithered before quantization
- All processing stages should maintain at least 20-bit internal precision

### 8.2 AES3 / IEC 60958 — Digital Audio Interface

The AES3 standard for digital audio interface (S/PDIF) specifies 20-bit or 24-bit PCM transmission. When mixing between different sample rates or bit depths on a digital audio console, dither must be applied at each quantization boundary to prevent distortion from propagating through the signal chain.

### 8.3 Mastering Workflow

Industry-standard mastering workflow for 24-bit session → 16-bit CD:

```
1. Mix at 32-bit float / 96 kHz (or higher)
         ↓
2. Apply any processing (EQ, compression, limiting)
         ↓
3. Dither to 24-bit (TPDF) — if keeping 24-bit output
         ↓
4. Resample to 44100 Hz (if targeting CD)
         ↓
5. Dither to 16-bit (TPDF+NS) — for final CD output
         ↓
6. Dither to 16-bit (TPDF) — for streaming/digital delivery
```

### 8.4 Dither in Lossy Codecs

Lossy codecs (MP3, AAC, Opus) perform internal quantization as part of their encoding process. Dithering before lossy encoding can improve quality for material that will be encoded at low bitrates:

| Scenario | Dither Recommendation |
|----------|----------------------|
| 24-bit → MP3 320 kbps | Optional (bitrate is high enough) |
| 24-bit → AAC 128 kbps | Recommended (TPDF) |
| 24-bit → Opus 64 kbps | Recommended (TPDF+NS) |
| 16-bit → MP3 128 kbps | Not needed (MP3's own quantization dominates) |
| 16-bit → MP3 320 kbps | Optional |

---

## 9. NOISE SHAPING CURVES — DETAILED REFERENCE

### 9.1 Frequency-Domain Noise Shaping

A practical noise shaping filter is designed in the frequency domain using the ATH curve:

1. Define target noise floor: $N_{target}(f) = ATH(f) - SNR_{target}$
2. Design filter $H(f)$ such that $|1 - H(f)|^2 \propto \frac{N_0}{N_{target}(f)}$
3. Convert $H(f)$ to time-domain coefficients via IFFT

### 9.2 POW-r Noise Shaping

The POW-r (Psychoacoustically Optimized) curves developed by iZotope (originally by W. Jeff Test) provide three levels of noise shaping:

**POW-r 1 (Mild):** ~3 dB attenuation at 1–2 kHz, gradual high-frequency boost
- Use for: acoustic music, vocals, material with high-frequency content

**POW-r 2 (Medium):** ~6 dB attenuation at 2–4 kHz, more aggressive high-frequency boost
- Use for: general-purpose mastering

**POW-r 3 (Aggressive):** ~10 dB attenuation at 2–5 kHz, strong high-frequency boost above 12 kHz
- Use for: electronic music, material dominated by bass and percussion

### 9.3 Noise Shaping Trade-offs

| Shaping Strength | Midrange Attenuation | HF Noise Increase | Audible Character |
|-----------------|---------------------|-------------------|-------------------|
| None (flat) | 0 dB | 0 dB | Clean but less dynamic |
| Mild (POW-r 1) | ~3 dB | ~3 dB | Natural |
| Medium (POW-r 2) | ~6 dB | ~6 dB | Acceptable |
| Aggressive (POW-r 3) | ~10 dB | ~10 dB | Can sound "airy" or "bright" |
| Very Aggressive | >12 dB | >12 dB | Can sound harsh |

### 9.4 Optimal Noise Shaping for 16-bit Delivery

For CD/streaming delivery (16-bit), the recommended noise shaping targets:
- **2–4 kHz:** -6 to -8 dB relative to flat
- **5–10 kHz:** -3 to -5 dB relative to flat
- **12–18 kHz:** +3 to +6 dB relative to flat (acceptable)
- **Above 18 kHz:** noise floor is below hearing threshold anyway

---

## 10. DSD TO PCM DITHERING

### 10.1 DSD's Effective Bit Depth

DSD stores audio as a 1-bit sigma-delta modulated bitstream. The effective resolution varies with frequency due to noise shaping:
- In the 0–10 kHz band: equivalent to ~20–22 bits
- In the 10–20 kHz band: equivalent to ~18–20 bits
- Above 20 kHz: noise floor rises

### 10.2 DSD to PCM Conversion with Dither

When converting DSD to PCM (e.g., DSD64 to PCM 176400 Hz or 352800 Hz):

1. The sigma-delta decimation filter removes out-of-band noise
2. The output is multi-bit PCM at the target rate
3. Dithering is applied only if the output bit depth is lower than the input's effective precision
4. For DSD → 24-bit PCM: dither is optional (effective precision is ~20–22 bits)
5. For DSD → 16-bit PCM: dithering IS required

```bash
# DSD to 24-bit PCM — no dithering needed (24-bit > effective DSD resolution)
ffmpeg -i input.dsf -acodec pcm_s24le -ar 352800 output.wav

# DSD to 16-bit PCM — dithering required
ffmpeg -i input.dsf \
  -af "aformat=s16p:dither_method=triangular,aresample=in_sample_rate=2822400:out_sample_rate=176400" \
  -c:a pcm_s16le output.wav
```

---

## 11. PRACTICAL GUIDELINES FOR CONVERTERS

### 11.1 When to Apply Dither

| Conversion Type | Dither Required? | Method |
|----------------|-----------------|--------|
| 24-bit → 24-bit (same format) | No | N/A |
| 24-bit → 20-bit | Yes | TPDF |
| 24-bit → 16-bit | Yes | TPDF or TPDF+NS |
| 32-bit float → 24-bit PCM | Yes | TPDF |
| 32-bit float → 16-bit PCM | Yes | TPDF+NS |
| 24-bit → 16-bit (streaming) | Yes | TPDF+NS |
| DSD64 → 24-bit PCM | Optional | TPDF (if applied) |
| DSD64 → 16-bit PCM | Yes | TPDF |
| Any → integer (broadcast) | Yes | TPDF |

### 11.2 When NOT to Apply Dither

1. **Multiple generations of lossless encoding:** If you are processing a FLAC → decode → FLAC chain, only dither at the final output stage
2. **Broadcast/playout chains with automatic dithering:** Some broadcast equipment has built-in dithering — double-dithering adds unnecessary noise
3. **Lossy encoding at high bitrates:** If the lossy codec's quantization dominates, external dithering provides minimal benefit

### 11.3 Dither Amplitude Accuracy

The dither amplitude must be precisely 0.5 LSB of the OUTPUT format. Common errors:

| Error | Effect |
|-------|--------|
| Dither too strong (1.0 LSB) | Excess noise floor, ~6 dB higher than optimal |
| Dither too weak (0.25 LSB) | Residual harmonic distortion, incomplete linearization |
| Dither amplitude varies with level | Dynamic distortion artifacts |
| Dither not applied in correct domain | Wrong LSB size — distortion persists |

### 11.4 Multi-Channel Dithering

For stereo and multichannel audio, each channel must have its own independent dither generator. Using the same dither for multiple channels would create inter-channel correlation, which:
- Preserves stereo image distortion
- Can be heard as phantom images
- Violates the dithering assumption of independent noise

```c
// Each channel gets its own random number generator
for (int ch = 0; ch < channels; ch++) {
    // Separate RNG state per channel prevents correlation
    dither[ch] = tpdf_sample_from_channel_state(rng_state[ch]);
}
```

---

## 12. VERIFICATION AND TESTING

### 12.1 Spectral Test for Dither Verification

To verify that dither is working correctly, apply a very low-level sine wave (e.g., -80 dBFS) and check the output spectrum:

**Without dither:** You will see the sine wave plus harmonic distortion products at multiples of the fundamental frequency.

**With TPDF dither:** You will see only the sine wave plus a flat noise floor. No harmonics.

```python
import numpy as np

def test_dither_spectrum(wav_file, sine_hz=1000, sine_level_db=-80):
    """Test dithering quality by analyzing sine wave distortion."""
    samples, sr = read_wav(wav_file)

    # Compute FFT
    N = len(samples)
    spectrum = np.abs(np.fft.rfft(samples * np.hanning(N)))
    freqs = np.fft.rfftfreq(N, 1/sr)

    # Find sine wave peak
    sine_idx = np.argmax(spectrum[(freqs > sine_hz - 50) & (freqs < sine_hz + 50)])

    # Find harmonic peaks (2f, 3f, 4f, 5f)
    harmonics = [sine_hz * k for k in range(2, 6)]

    # Compute THD+N
    signal_power = spectrum[sine_idx]**2
    noise_power = np.sum([spectrum[np.argmax(np.abs(freqs - h))]**2
                          for h in harmonics])
    thd = 10 * np.log10(noise_power / signal_power)

    print(f"THD (without fundamental): {thd:.1f} dB")
    if thd < -80:
        print("Dither appears to be working (no harmonic distortion)")
    else:
        print("WARNING: Possible dithering issue (harmonic distortion detected)")
```

### 12.2 DC Test

Test with a DC offset or near-DC sine wave:

**Without dither:** The DC level will be quantized to discrete steps, creating a staircase pattern.

**With TPDF dither:** The DC level is correctly reproduced as an average value with noise modulation.

### 12.3 Histogram Test

```python
def test_dither_histogram(wav_file, test_signal_amplitude=0.25):
    """Verify dither by checking error histogram shape."""
    # Generate known test signal (uniform noise)
    test = np.random.uniform(-test_signal_amplitude, test_signal_amplitude, 10000)

    # Process through quantizer
    quantized = np.round(test * 32768) / 32768  # Simulate 16-bit

    # Compute error
    error = quantized - test

    # Plot histogram — should be triangular for TPDF
    # Should be rectangular for RPDF
    # Should show clear distortion pattern without dither
```

---

## 13. REFERENCE IMPLEMENTATIONS

### 13.1 Pure C TPDF Dither

```c
#include <stdlib.h>
#include <math.h>

// High-quality TPDF dither for audio conversion
// From float input to 16-bit PCM output

static inline float generate_tpdf(void) {
    // Two uniform RPDF samples in [0, 1)
    float u1 = (float)rand() / (float)RAND_MAX;
    float u2 = (float)rand() / (float)RAND_MAX;
    // Map to [-0.5, +0.5)
    u1 -= 0.5f;
    u2 -= 0.5f;
    // TPDF: sum of two RPDFs
    return u1 + u2;  // Range: [-1.0, +1.0)
}

void float_to_s16_with_dither(const float *input, int16_t *output,
                               size_t samples, float dither_scale) {
    for (size_t i = 0; i < samples; i++) {
        float dithered = input[i] + generate_tpdf() * dither_scale;
        // Round to nearest
        output[i] = (int16_t)lrintf(dithered);
        // Clamp to prevent overflow on full-scale signals
        if (output[i] > 32767) output[i] = 32767;
        if (output[i] < -32768) output[i] = -32768;
    }
}

// Usage for 16-bit output with 0.5 LSB TPDF:
float dither_scale = 1.0f / 32768.0f;  // ±0.5 LSB
float_to_s16_with_dither(float_buffer, s16_buffer, frame_count, dither_scale);
```

### 13.2 Noise Shaped Dither (POW-r Style)

```c
// Second-order noise shaping for 16-bit output
// Implements a psychoacoustically-shaped noise floor

void ns_dither_16bit(const float *input, int16_t *output,
                    size_t samples, const float *ns_coef, int coef_count) {
    float error_state[2] = {0};  // Feedback delay states
    float dither_state[2] = {0};  // TPDF state per channel

    for (size_t i = 0; i < samples; i++) {
        // TPDF dither generation
        float u1 = (float)rand() / (float)RAND_MAX - 0.5f;
        float u2 = (float)rand() / (float)RAND_MAX - 0.5f;
        float tpdf = u1 + u2;  // ±1 LSB

        // Apply noise shaping to error
        float shaped_error = 0.0f;
        shaped_error += ns_coef[0] * error_state[0];
        shaped_error += ns_coef[1] * error_state[1];

        // Add dither + shaped error
        float dithered = input[i] + tpdf * (1.0f/32768.0f) + shaped_error;

        // Quantize
        int16_t quantized = (int16_t)lrintf(dithered);

        // Update error state
        float quantized_float = (float)quantized / 32768.0f;
        float error = quantized_float - input[i];
        error_state[1] = error_state[0];
        error_state[0] = error;

        output[i] = quantized;
    }
}
```

---

## 14. SUMMARY: BEST PRACTICES

| Scenario | Recommended Dither | Notes |
|----------|-------------------|-------|
| Archival / mastering | TPDF + noise shaping | Use POW-r 2 or 3 |
| CD production | TPDF + NS (mild) | POW-r 1 or 2 |
| Streaming (MP3/AAC) | TPDF | At final output stage only |
| Video post-production | TPDF or TPDF_HP | HP variant prevents DC |
| Acoustic measurement | TPDF_HP | DC-free dither prevents measurement errors |
| Broadcast (EBU R68) | TPDF | Standard compliance |
| DSD → 16-bit PCM | TPDF + NS | Required |
| 24-bit → 16-bit (internal) | TPDF | For intermediate stages |
| 32-bit float → 24-bit PCM | TPDF | Optional but recommended |

### Critical Rules

1. **Always dither when reducing bit depth** — no exceptions for audio that will be heard
2. **Dither at the correct amplitude** — exactly 0.5 LSB of the OUTPUT format for TPDF
3. **Dither only once** at the final output stage — multiple dithering accumulates noise
4. **Use independent dither per channel** — correlated dither between channels creates phantom images
5. **Choose noise shaping appropriate to content** — aggressive shaping may be audible on classical music
6. **Verify dither implementation** with spectral tests — check for residual harmonics on low-level tones

---

*File generated for: Audio Engineering Knowledge Base*
*Topic: Dithering, Noise Shaping, Quantization, Bit-Depth Reduction*
*[NEEDS VERIFICATION] markers indicate specific numerical values that require additional source confirmation*
*[NEEDS VERIFICATION]: The exact numerical coefficients for FFmpeg's triangular_ns noise shaping filter — the source code contains these values but they are not documented in the public API and should be verified against the libswresample source*
