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

## 14. ADVANCED DITHERING TOPICS

### 14.1 n-th Order Dither (nRPDF)

The concept of TPDF (2RPDF) can be extended to higher orders:
- **1RPDF:** Rectangular (uniform) — first-order, flat spectrum
- **2RPDF:** Triangular (TPDF) — second-order, first spectral moment = 0
- **3RPDF:** Third-order, second spectral moment = 0
- **nRPDF:** n-th order, (n-1)-th spectral moment = 0

Higher-order dither provides better linearization at the cost of higher noise amplitude:
$$V_{nRPDF} = \sum_{i=1}^{n} RPDF_i$$

For nRPDF with range $[-n\Delta/2, n\Delta/2]$, the noise amplitude grows as $\sqrt{n}$ but the linearization quality improves.

### 14.2 Signal-Dependent Dither

Traditional dither adds a fixed-amplitude noise regardless of signal level. Signal-dependent dither varies the dither amplitude based on the input signal's proximity to quantization boundaries:
- Less dither for signals well within a quantization step
- Full dither for signals near step boundaries
- Reduces the added noise floor for signals that are already well-resolved

### 14.3 Blurred Dither (Perceptual Dither)

Blurred dither uses a noise signal that is pre-filtered to match the noise masking curve of the human ear:
- Less noise energy in the 1–6 kHz region (most sensitive)
- More noise energy at DC and above 15 kHz (less sensitive)
- Can achieve the same perceptual quality with lower total noise power
- Also called "perceptually shaped dither" or "ATH-weighted dither"

### 14.4 Multi-Stage Dithering

For cascading format conversions (e.g., 24-bit → 16-bit → 12-bit), dither should be applied:
1. At each quantization boundary where bit depth is reduced
2. With appropriate amplitude for each stage's LSB size

**Example: 24-bit → 16-bit → 12-bit**
```
Stage 1: Add TPDF with amplitude 0.5 LSB (24-bit), quantize to 16-bit
Stage 2: Add TPDF with amplitude 0.5 LSB (16-bit), quantize to 12-bit
```

### 14.5 Dithering in Lossy Codecs

Lossy codecs (MP3, AAC, Opus) perform internal quantization that generates quantization noise. Whether to apply dithering before encoding:

| Scenario | Recommendation |
|----------|---------------|
| High bitrate (320 kbps AAC, FLAC) | Dither optional — codec's internal quantization dominates |
| Medium bitrate (192 kbps AAC) | Dither recommended for quiet passages |
| Low bitrate (128 kbps MP3) | Dither can help preserve low-level dynamics |
| Very low bitrate (< 64 kbps) | Dither minimal benefit — codec's artifacts dominate |

### 14.6 Floating-Point Dithering

When converting from 32-bit floating point to 24-bit integer:
- The floating-point mantissa has non-uniform precision (relative to signal level)
- The "effective LSB" varies with the exponent
- Dither amplitude should match the effective LSB at the current level
- Simple fixed-amplitude dither may over-dither quiet signals

**Proper floating-point dithering:**
```c
// Calculate effective LSB for current sample
float effective_lsb = ldexpf(1.0f, -24.0f - floorf(log2f(fabsf(sample))));

// Dither with level-proportional amplitude
float dither = tpdf_sample() * effective_lsb * 0.5f;
```

### 14.7 Dither and Bit-Shifting

When converting between bit depths by bit-shifting (rather than quantization):
- Bit-shifting is a lossless operation — dither is NOT needed
- Example: Converting 24-bit to 16-bit by right-shifting 8 bits is lossless for the top 16 bits
- Example: Converting 32-bit float to 24-bit by mantissa truncation — dither IS needed

### 14.8 Dithering for A/B Comparisons

When testing dither audibility:
- Use A/B/X testing methodology
- Test with signals near quantization boundary: sine waves at -80 to -70 dBFS
- Test material: solo piano, solo violin (tonal content reveals harmonic distortion)
- Without dither: harmonics are clearly audible
- With TPDF: harmonics replaced by noise floor

---

## 15. NOISE SHAPING — DEEP MATHEMATICS

### 15.1 Noise Transfer Function

In a noise-shaped quantizer:

$$Y(z) = X(z) + E(z) \cdot (1 - H(z))$$

Where:
- $X(z)$ = input signal
- $Y(z)$ = output signal
- $E(z)$ = quantization error
- $H(z)$ = noise shaping filter transfer function

The noise spectrum at the output:
$$N(z) = E(z) \cdot (1 - H(z))$$

To minimize noise in a frequency band $[f_1, f_2]$:
$$|1 - H(f)| \approx 0 \text{ for } f \in [f_1, f_2]$$
$$|1 - H(f)| \approx 1 \text{ for } f \notin [f_1, f_2]$$

### 15.2 First-Order Noise Shaping Transfer Function

$$H(z) = z^{-1}$$
$$1 - H(z) = 1 - z^{-1}$$

Frequency response:
$$|1 - e^{-j\omega}| = 2\sin(\omega/2)$$

This gives a high-shelf response — noise is amplified at high frequencies, reduced at low frequencies.

### 15.3 Second-Order Noise Shaping

$$H(z) = 2z^{-1} - z^{-2}$$
$$1 - H(z) = 1 - 2z^{-1} + z^{-2} = (1 - z^{-1})^2$$

Frequency response:
$$|1 - e^{-j\omega}|^2 = 4\sin^2(\omega/2)$$

More aggressive high-frequency boosting.

### 15.4 Optimal Noise Shaping Filter Design

For psychoacoustic optimization, design $H(z)$ such that the shaped noise spectrum $N(f)$ follows the inverse of the ATH curve:

$$|N(f)|^2 \propto \frac{1}{ATH(f)}$$

Where $ATH(f)$ is the Absolute Threshold of Hearing.

### 15.5 IIR vs FIR Noise Shaping Filters

**FIR noise shapers:**
- Linear phase
- Always stable
- Can have very steep transitions
- Higher computational cost

**IIR noise shapers:**
- Can be minimum phase
- May be unstable (pole placement critical)
- Lower computational cost
- Commonly used in professional dithering systems

### 15.6 Noise Shaping and Dither Interaction

The choice of noise shaping is tightly coupled with dither:
- TPDF dither is required for effective noise shaping above first-order
- RPDF dither cannot provide linearization beyond first-order
- Higher-order dither (3RPDF) enables more aggressive noise shaping

---

## 16. DITHER IN BROADCAST AND ARCHIVAL

### 16.1 EBU R68 Requirements

EBU R68 (2000) specifies:
- Audio for digital broadcasting shall be quantized to at least 16 bits
- Dithering shall be applied when reducing bit depth
- TPDF dither is recommended
- Peak level calibration: 0 dBFS = -18 dB on a PPM (Peak Programme Meter)

### 16.2 Archival Best Practices

For long-term digital archival:

| Principle | Implementation |
|-----------|---------------|
| Preserve dynamic range | Use highest practical bit depth (24-bit minimum) |
| Dither at delivery only | Apply dither only when creating distribution copies |
| Use TPDF | Second-order dither provides best tonal linearity |
| Consider noise shaping | Mild noise shaping for 16-bit delivery |
| Document conversion chain | Record all SRC and dither operations |
| Verify with test signals | Include calibration tones in archival files |

### 16.3 Archival Format Recommendations

| Use Case | Format | Bit Depth | Sample Rate | Dither |
|----------|--------|-----------|-------------|--------|
| Archival master | FLAC | 24-bit | 96 kHz | No (keep pristine) |
| Distribution copy | FLAC | 24-bit | 96 kHz | No |
| CD-ready copy | FLAC | 24-bit | 44.1 kHz | Yes (TPDF) |
| Streaming master | FLAC | 24-bit | 44.1 or 48 kHz | Yes (TPDF+NS) |
| Broadcast | WAV/BWF | 20-bit | 48 kHz | Yes (TPDF) |

### 16.4 Test Tones for Dither Verification

Include in archival files:
- 997 Hz sine wave at -1 dBFS
- 0 Hz (DC) offset at -20 dBFS
- 0 Hz (DC) offset at -40 dBFS
- Silence (all zeros)

These enable verification of dither effectiveness in future conversions.

---

## 17. DITHERING IN DISTRIBUTED SYSTEMS

### 17.1 DAW Internal Processing

Modern DAWs process at 32-bit or 64-bit floating point internally:
- No quantization occurs during mixing, EQ, or effects
- Dither is applied only at:
  1. Export to integer format
  2. Real-time playback to hardware with limited bit depth

### 17.2 Real-Time Playback Dithering

When streaming to hardware DACs:
- Most modern DACs (24-bit) do not need output dithering
- Some 16-bit DACs benefit from TPDF dither
- Hardware with volume control may re-quantize at each volume step

### 17.3 Network Audio Streaming

In networked audio systems (AES67, Dante, etc.):
- Audio is transmitted at full precision (32-bit float or 24-bit)
- Dithering occurs only at the final output stage
- Sample rate conversion may occur at network bridges — verify SRC quality

### 17.4 Multi-Rate Audio Systems

When mixing sources at different sample rates:
- Each source is resampled to the DAW's internal rate
- Dither should be applied during SRC if the intermediate format has reduced precision
- Final output dither is applied once at delivery

---

## 18. PSYCHOACOUSTIC FUNDAMENTALS FOR NOISE SHAPING

### 18.1 Critical Bands

The ear's frequency resolution follows critical bands (Bark scale):
- Critical band $k$: $f_c = 25 + 75 \times [(1 + 1.4 \times k^2)^{0.69} - 1]$ Hz
- Bandwidth doubles approximately every 3–4 bands

| Critical Band | Center Frequency | Bandwidth |
|--------------|-----------------|-----------|
| 1 | 50 Hz | 100 Hz |
| 2 | 150 Hz | 100 Hz |
| 3 | 250 Hz | 100 Hz |
| 4 | 350 Hz | 100 Hz |
| 8 | 1050 Hz | ~150 Hz |
| 15 | 4500 Hz | ~400 Hz |
| 24 | 15500 Hz | ~2300 Hz |

### 18.2 Simultaneous Masking

A masker tone at level $L$ (dB SPL) can mask a signal at frequency $f$ if:
$$L_{masker} - S(f) > ATH(f)$$

Where $S(f)$ is the masking index, approximately:
$$S(f) = 14.5 + 0.4 \times L_{masker} - \text{masking_index}(f - f_{masker})$$

### 18.3 Noise Shaping Design Based on Masking

A practical noise shaping filter should:
1. Reduce noise power in critical bands where ATH is low (1–5 kHz)
2. Allow noise power in bands where ATH is high (< 100 Hz, > 15 kHz)
3. Have smooth transitions between bands to avoid spectral artifacts

### 18.4 Effective SNR After Noise Shaping

For a 16-bit system with noise shaping:
- Flat noise floor: -93 dB (theoretical)
- With 6 dB noise shaping in critical band: effective SNR ≈ -99 dB in that band
- Perceptually equivalent to ~17–18 bits in the shaped region

---

## 19. DITHERING CODE EXAMPLES

### 19.1 Complete Dither State Machine

```c
#include <stdint.h>
#include <stdlib.h>
#include <math.h>
#include <string.h>

typedef enum {
    DITHER_NONE = 0,
    DITHER_RECTANGULAR = 1,
    DITHER_TRIANGULAR = 2,
    DITHER_TRIANGULAR_HP = 3,
    DITHER_TRIANGULAR_NS = 4,
} DitherMethod;

// Linear feedback shift register (LFSR) for fast PRNG
typedef struct {
    uint32_t state;
    int method;
    float amplitude;     // In LSBs of output format
    float prev_dither;   // For high-pass dither
    // Noise shaping state (2nd order)
    float ns_b[3];      // FIR coefficients
    float ns_x[2];      // Input delay
    float ns_y[2];      // Output delay
    float ns_prev_err;  // Previous error
} DitherContext;

uint32_t lfsr_next(uint32_t state) {
    // Fibonacci LFSR with period 2^32-1
    uint32_t bit = ((state >> 0) ^ (state >> 2) ^
                    (state >> 3) ^ (state >> 5)) & 1;
    return (state >> 1) | (bit << 31);
}

DitherContext *dither_init(DitherMethod method, float output_lsb) {
    DitherContext *ctx = calloc(1, sizeof(DitherContext));
    ctx->method = method;
    ctx->amplitude = output_lsb * 0.5f; // 0.5 LSB for TPDF
    ctx->state = 0xACE1; // Non-zero seed

    // 2nd order noise shaping coefficients
    // (simplified — flat at DC, -6dB at Nyquist)
    ctx->ns_b[0] = 1.0f;
    ctx->ns_b[1] = -2.0f;
    ctx->ns_b[2] = 1.0f;

    return ctx;
}

float dither_generate(DitherContext *ctx) {
    float d = 0.0f;

    switch (ctx->method) {
    case DITHER_RECTANGULAR: {
        // RPDF: uniform in [-0.5, +0.5) * LSB
        ctx->state = lfsr_next(ctx->state);
        float u = (float)ctx->state / (float)0xFFFFFFFF;
        d = u - 0.5f;
        break;
    }

    case DITHER_TRIANGULAR: {
        // TPDF: sum of two RPDF samples
        ctx->state = lfsr_next(ctx->state);
        float u1 = (float)ctx->state / (float)0xFFFFFFFF - 0.5f;
        ctx->state = lfsr_next(ctx->state);
        float u2 = (float)ctx->state / (float)0xFFFFFFFF - 0.5f;
        d = u1 + u2;
        break;
    }

    case DITHER_TRIANGULAR_HP: {
        // TPDF with DC removal
        ctx->state = lfsr_next(ctx->state);
        float u1 = (float)ctx->state / (float)0xFFFFFFFF - 0.5f;
        ctx->state = lfsr_next(ctx->state);
        float u2 = (float)ctx->state / (float)0xFFFFFFFF - 0.5f;
        float tpdf = u1 + u2;
        d = tpdf - ctx->prev_dither; // High-pass
        ctx->prev_dither = tpdf;
        break;
    }

    default:
        break;
    }

    return d * ctx->amplitude;
}

void dither_process(DitherContext *ctx,
                    const float *input,
                    int16_t *output,
                    size_t samples) {
    for (size_t i = 0; i < samples; i++) {
        float dither = dither_generate(ctx);
        float dithered = input[i] + dither;
        int32_t quantized = (int32_t)lrintf(dithered);
        // Clamp to prevent overflow
        if (quantized > 32767) quantized = 32767;
        if (quantized < -32768) quantized = -32768;
        output[i] = (int16_t)quantized;
    }
}

void dither_free(DitherContext *ctx) {
    free(ctx);
}
```

### 19.2 Python Dither Verification Script

```python
import numpy as np
import struct
import wave

def generate_tpdf(n_samples):
    """Generate TPDF dither."""
    u1 = np.random.uniform(-0.5, 0.5, n_samples)
    u2 = np.random.uniform(-0.5, 0.5, n_samples)
    return u1 + u2

def quantize_to_16bit(samples):
    """Quantize float samples to 16-bit PCM."""
    dither = generate_tpdf(len(samples))
    dithered = samples + dither * (1.0 / 32768.0)
    quantized = np.round(dithered * 32768.0).astype(np.int16)
    return np.clip(quantized, -32768, 32767)

def test_dither():
    """Test dithering on 1 kHz sine wave at -80 dBFS."""
    sr = 44100
    freq = 1000
    duration = 1.0

    # Generate -80 dBFS sine wave
    t = np.arange(int(sr * duration)) / sr
    amplitude = 10 ** (-80 / 20)  # -80 dBFS
    sine = amplitude * np.sin(2 * np.pi * freq * t)

    # Quantize with dither
    quantized = quantize_to_16bit(sine)

    # Compute FFT
    N = len(quantized)
    window = np.hanning(N)
    fft_result = np.fft.rfft(quantized.astype(np.float32) * window)
    freqs = np.fft.rfftfreq(N, 1/sr)
    magnitude = np.abs(fft_result) / N * 2

    # Find fundamental and harmonics
    fundamental_idx = np.argmax(magnitude[(freqs > 900) & (freqs < 1100)])
    harmonic_2 = np.max(magnitude[(freqs > 1900) & (freqs < 2100)])
    harmonic_3 = np.max(magnitude[(freqs > 2900) & (freqs < 3100)])

    fundamental = magnitude[(freqs > 900) & (freqs < 1100)][fundamental_idx]
    thd = 20 * np.log10(np.sqrt(harmonic_2**2 + harmonic_3**2) / fundamental)

    print(f"Fundamental: {20*np.log10(fundamental/32768):.1f} dBFS")
    print(f"2nd Harmonic: {20*np.log10(harmonic_2/32768):.1f} dBFS")
    print(f"3rd Harmonic: {20*np.log10(harmonic_3/32768):.1f} dBFS")
    print(f"THD: {thd:.1f} dB")

    # Without dither, THD would be ~-50 to -60 dB
    # With TPDF dither, THD should be below -100 dB
    if thd < -80:
        print("Dither appears to be working (low harmonic distortion)")
    else:
        print("WARNING: High harmonic distortion detected")

if __name__ == "__main__":
    test_dither()
```

---

## 20. SUMMARY: DITHERING DECISION TREE

```
Is bit depth being reduced?
│
├── NO → No dithering needed
│
└── YES ↓
    │
    Is output format floating-point?
    │
    ├── YES → Dither at level proportional to effective LSB at each sample
    │
    └── NO ↓
        │
        Is the output format for measurement/acoustic analysis?
        │
        ├── YES → Use TPDF_HP (high-pass) to prevent DC accumulation
        │
        └── NO ↓
            │
            Is the content primarily tonal (classical, acoustic)?
            │
            ├── YES → Use TPDF (triangular) — flat noise floor
            │
            └── NO ↓
                │
                Is this the final delivery format (CD, streaming)?
                │
                ├── YES → Use TPDF+NS (noise shaping, mild POW-r)
                │
                └── NO → Use TPDF (standard triangular)
```

---

## 21. DITHERING IN SPECIFIC WORKFLOWS

### 21.1 Mastering Workflow

The professional mastering workflow for a 24-bit/96 kHz session to 16-bit/44.1 kHz CD:

```
Session: 32-bit float / 96 kHz (internal DAW processing)
         ↓
1. Final processing (EQ, compression, limiting)
   - Apply at 96 kHz internal rate
   - Do not reduce bit depth yet
         ↓
2. Export at 96 kHz (maintain resolution)
         ↓
3. High-quality SRC to 44100 Hz
   - Use libsamplerate SINC_BEST_QUALITY or soxr
   - Internal bit depth: 64-bit float
   - This step preserves all audible content
         ↓
4. Apply dithering for bit-depth reduction
   - Dither from float to 24-bit: optional (TPDF)
   - Dither from 24-bit to 16-bit: REQUIRED (TPDF or TPDF+NS)
   - Choose TPDF for classical/acoustic music
   - Choose TPDF+NS for pop/electronic
         ↓
5. Verify with spectral analysis
   - Check for harmonic distortion on test tones
   - Verify noise floor is flat (no distortion products)
         ↓
6. Output to CD image or streaming master
```

### 21.2 Podcast Production Workflow

Podcast audio typically ends up at 44.1 kHz (CD/export) or 48 kHz (video sync), at 16-bit or 24-bit:

```
Recording: 48 kHz / 24-bit (or higher)
         ↓
Editing: 32-bit float / 48 kHz (DAW)
         ↓
Export: 48 kHz / 24-bit WAV
         ↓
SRC (if needed): 48 kHz → 44.1 kHz for CD
   - Use FFmpeg: aresample=resampler=soxr:precision=28
   - Internal dithering not needed (bit depth unchanged)
         ↓
Dither: 24-bit → 16-bit (for MP3/AAC streaming)
   - Apply TPDF dither
   - FFmpeg: aformat=s16p:dither_method=triangular
         ↓
Encode: MP3 or AAC at target bitrate
```

### 21.3 Archival Workflow

For long-term digital preservation, follow these principles:

```
Capture: DSD64 or PCM 176.4 kHz / 24-bit
         ↓
Store as: FLAC level 8 at capture rate (lossless)
         ↓
Delivery copies: Only apply SRC + dither at delivery
   - Never modify the archive master
   - Apply dither only once, at final delivery stage
         ↓
Verify: Periodic checksum verification of archive files
```

### 21.4 Video Post-Production Workflow

Video requires strict sample rate synchronization. The challenge is that 44100 Hz is not an integer multiple of common video frame rates like 23.976 fps:

```
Video frame rate: 23.976 fps (film) or 24 fps
Audio sync requirement: Audio must match video duration exactly

Challenge: 44100 / 23.976 ≈ 1839.4 samples per frame (not integer!)
Solution 1: Use 48000 Hz audio (48000 / 24 = 2000 samples/frame, exact)
Solution 2: Accept that CD audio will not perfectly frame-align
Solution 3: Use pull-up/pull-down rates (e.g., 47.952 kHz for 23.976 fps)

For 48 kHz audio synced to 23.976 fps video:
- Apply SRC only if output needs to be 44100 Hz
- Maintain sample-accurate sync throughout the pipeline
```

### 21.5 Broadcast WAV Workflow

Broadcast uses BWF (Broadcast Wave Format) with specific requirements defined in EBU R68:

```
Sample rate: 48 kHz (EBU R68 standard)
Bit depth: 20-bit minimum
Dithering: TPDF applied at delivery stage
Peak level: -18 dBFS = 0 on PPM (per EBU R68)

When converting broadcast content to CD:
- SRC: 48000 → 44100 (FFmpeg with soxr)
- Dither: TPDF (broadcast standard)
- Verify against original EBU R68 calibration tones
```

---

## 22. DITHERING AND LOUDNESS

### 22.1 Dithering Does Not Affect Loudness

Dithering adds noise that is uncorrelated with the signal. The long-term RMS of the added noise adds to the signal's RMS, but this effect is imperceptible for TPDF dither (noise is below -93 dBFS for 16-bit). Without dithering, quantization adds harmonic distortion that is far more perceptually disturbing.

### 22.2 Dithering and Dynamic Range

**True dynamic range** is the ratio between the largest possible signal and the noise floor:

For a 16-bit system with TPDF dither:
- Maximum signal: 0 dBFS
- Noise floor: approximately -93 dBFS
- True dynamic range: about 93 dB

For a 16-bit system without dither:
- Maximum signal: 0 dBFS
- Noise + distortion floor: varies, may be higher in midrange
- True dynamic range: reduced due to harmonic distortion

### 22.3 Dithering and Loudness Normalization

Loudness normalization (EBU R128, ReplayGain) measures loudness in the presence of dither noise. The dither noise floor contributes to the measured loudness, but this contribution is negligible at normal listening levels.

---

## 23. THE HISTORY OF DITHERING

### 23.1 Early Digital Audio

Early digital audio systems from the 1970s and 1980s did not always use dithering, resulting in audible harmonic distortion on quiet passages, particularly for classical and acoustic recordings.

### 23.2 Key Research Milestones

| Year | Researcher | Contribution |
|------|-----------|-------------|
| 1972 | Schuchman | First formal treatment of dither in digital systems |
| 1984 | Vanderkooy, Lipshitz | Dither theory for audio |
| 1992 | Wannamaker, Lipshitz, Vanderkooy | Theory of Dithered Quantization |
| 1995 | Lipshitz, Vanderkooy | Practical TPDF dither implementation |
| 1998 | Ehmer | Psychoacoustic dithering |
| 2000 | Ehmer | TPDF dither widely adopted in DAWs |

### 23.3 Modern DAWs

All major DAWs apply dithering in their export and bounce processes. Each offers different dithering options:
- **Logic Pro:** POW-r dithering in bounce settings
- **Pro Tools:** TPDF and POW-r dithering options
- **Ableton Live:** Built-in dithering options
- **REAPER:** Multiple dithering options including TPDF and noise-shaped variants

---

## 24. DITHERING MYTHS AND FACTS

### Myth 1: Dithering adds noise that degrades quality

**Fact:** Dithering REPLACES harmonic distortion with broadband noise. The noise is at or below the theoretical quantization noise floor and is typically inaudible. Without dithering, quantization produces harmonic distortion that is far more perceptually disturbing.

### Myth 2: Dithering is only needed for 16-bit output

**Fact:** Dithering is beneficial whenever bit depth is reduced, even from 24-bit to 20-bit. The improvement is more subtle at higher bit depths but still measurable and can be audible on critical material.

### Myth 3: TPDF dither is always better than RPDF

**Fact:** TPDF dither is better for tonal material where harmonic distortion would be audible. For primarily noise-like material (drums, noise textures), RPDF dither is acceptable and produces slightly less total noise.

### Myth 4: Noise shaping is always better than flat dither

**Fact:** Noise shaping trades noise in one frequency range for noise in another. On material with significant high-frequency content (cymbals, breath noise, tape hiss), aggressive noise shaping may actually be audible as extra brightness or a glassy character.

### Myth 5: You should dither at every stage

**Fact:** Dither should be applied ONCE at the final output stage. Applying dither at every intermediate conversion accumulates noise unnecessarily. The exception is when each intermediate stage's output will be heard or further processed independently.

### Myth 6: Dithering fixes everything

**Fact:** Dithering cannot recover information that was lost to prior quantization. If the source was recorded at 16-bit, dithering when converting from 24-bit to 16-bit cannot restore details already lost at the 16-bit recording stage.

---

## 25. SUMMARY REFERENCE TABLES

### 25.1 Dither Type Selection Guide

| Content Type | Dither Type | Noise Shaping | Notes |
|-------------|------------|--------------|-------|
| Classical music | TPDF | Optional (mild) | Preserves tonal quality |
| Jazz/acoustic | TPDF | Optional | |
| Pop/electronic | TPDF+NS | Yes (mild-medium) | Hides noise in HF |
| Rock | TPDF+NS | Yes (mild) | |
| Heavy metal | TPDF+NS | Yes (mild) | |
| Hip-hop/R&B | TPDF+NS | Yes (medium) | |
| Spoken word | RPDF or TPDF | No | Minimal tonal content |
| Nature recordings | TPDF | Optional | |
| Field recordings | TPDF | Optional | |
| Ambience | TPDF | Optional | |

### 25.2 Bit Depth and Dither Recommendations

| Source | Output | Dither | Method | Notes |
|--------|--------|--------|--------|-------|
| 32-bit float | 24-bit | Optional | TPDF | Internal precision is high |
| 32-bit float | 16-bit | Yes | TPDF+NS | REQUIRED |
| 24-bit | 24-bit | No | — | No reduction |
| 24-bit | 20-bit | Yes | TPDF | |
| 24-bit | 16-bit | Yes | TPDF+NS | Standard delivery |
| 20-bit | 16-bit | Yes | TPDF | |
| 16-bit | 16-bit | No | — | No reduction |
| DSD64 | 24-bit | Optional | TPDF | Optional |
| DSD64 | 16-bit | Yes | TPDF | REQUIRED |

### 25.3 Noise Shaping Strength Guide

| Strength | NS Curve | Use Case | HF Noise Increase |
|----------|---------|---------|------------------|
| None | Flat | Measurement, neutral | 0 dB |
| Mild (POW-r 1) | +3 dB at 15 kHz | Classical, acoustic | +3 dB |
| Medium (POW-r 2) | +6 dB at 15 kHz | General music | +6 dB |
| Aggressive (POW-r 3) | +10 dB at 15 kHz | Electronic, bass-heavy | +10 dB |
| Very aggressive | +15+ dB at 15 kHz | Only if needed | +15+ dB |

---

*File generated for: Audio Engineering Knowledge Base*
*Topic: Dithering, Noise Shaping, Quantization, Bit-Depth Reduction*
*[NEEDS VERIFICATION] markers indicate specific numerical values that require additional source confirmation*
*[NEEDS VERIFICATION]: The exact numerical coefficients for FFmpeg's triangular_ns noise shaping filter — the source code contains these values but they are not documented in the public API and should be verified against the libswresample source*
*[NEEDS VERIFICATION]: The exact values of ATH at specific frequencies in Section 18.1 — the critical band table values are approximations based on standard psychoacoustic research and should be verified against ISO 226:2003*
