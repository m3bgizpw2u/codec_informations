# Audio Sample Rate Conversion (SRC) — Deep Technical Reference
> **Category:** AudioEngineering
> **Topic:** Audio Resampling, Sample Rate Conversion, Polyphase FIR Interpolation
> **Related Codecs:** PCM, DSD, WAV, AIFF, FLAC, ALAC, AAC, MP3, Opus
> **FFmpeg Modules:** libswresample, libavresample, libsoxr
> **External Libraries:** libsamplerate (Secret Rabbit Code)
> **Standards:** IEC 60958, AES5, AES17, ITU-R BS.775

---

## 1. HISTORICAL CONTEXT & FUNDAMENTALS

### 1.1 Why Sample Rate Conversion Is Needed

Sample rate conversion (SRC) is the process of converting audio from one sample rate to another. It arises in virtually every audio conversion chain because different parts of the digital audio ecosystem operate at different rates.

**Common sample rate mismatches requiring SRC:**

| Source | Destination | Ratio Required | Difficulty |
|--------|-------------|----------------|-------------|
| CD Audio (44100 Hz) | DVD-Video Audio (48000 Hz) | 160/147 ≈ 1.08844 | Irrational — requires polyphase |
| CD Audio (44100 Hz) | DVD-Audio (96000 Hz) | 96000/44100 = 320/147 | Requires polyphase |
| 48000 Hz | 44100 Hz | 147/160 ≈ 0.91875 | Common in video production |
| DSD64 (2822400 Hz) | PCM 176400 Hz | 1/16 (integer) | Simple decimation |
| DSD64 (2822400 Hz) | PCM 352800 Hz | 1/8 (integer) | Simple decimation |
| 44100 Hz | 88200 Hz | 2/1 (integer) | Simple interpolation |
| 96000 Hz | 44100 Hz | 147/320 ≈ 0.45937 | Severe downsampling, anti-alias required |
| 192000 Hz | 48000 Hz | 5/1 (integer) | Simple ratio |

**Key insight:** Only when the ratio of output rate to input rate is a rational number with small integer numerators and denominators (e.g., 2/1, 3/1, 1/2, 4/3) can true integer-ratio resampling be used. In practice, the ratios between 44100 and 48000 (and their multiples) are irrational and require sophisticated polyphase filter banks.

### 1.2 Mathematical Foundation of Sample Rate Conversion

**The Shannon-Nyquist Sampling Theorem** states that a bandlimited signal sampled at rate $f_s$ can be perfectly reconstructed if it contains no frequencies at or above $f_s / 2$ (the Nyquist frequency).

**Upsampling by integer L:** Given an input signal $x[n]$, upsampling by integer $L$ produces:
$$y[n] = x[n/L] \text{ for } n = 0, L, 2L, \ldots \text{ (zeroes in between)}$$

This creates spectral images at multiples of the original sample rate, which must be removed by a low-pass filter with cutoff at $\pi/L$ radians/sample.

**Downsampling by integer M:** Downsampling by integer $M$ requires pre-filtering to avoid aliasing. The input must be low-pass filtered to $\pi/M$ radians/sample first:
$$y[n] = x[nM]$$

**General rational ratio P/Q:** Convert by upsampling by $P$, then downsampling by $Q$. The combined filter must have cutoff $\pi / \max(P, Q)$.

---

## 2. INTEGER-RATIO RESAMPLING

### 2.1 Upsampling by Integer L

Upsampling inserts $L-1$ zero-valued samples between each pair of input samples, then applies an interpolation filter. For $L=2$ (doubling the rate):

```
Input:     x[0]  x[1]  x[2]  x[3]  ...
Upsampled: x[0]   0   x[1]   0   x[2]   0   x[3]  ...
           ↑           ↑
         sample    zero-inserted
```

The ideal upsampling filter is a low-pass FIR with frequency response:
$$H(\omega) = \begin{cases} L & |\omega| \leq \frac{\pi}{L} \\ 0 & \frac{\pi}{L} < |\omega| \leq \pi \end{cases}$$

### 2.2 Downsampling by Integer M

Downsampling by $M$ requires a pre-filter to prevent aliasing. The anti-alias filter must attenuate all frequencies above $\pi/M$ radians/sample before decimation:

```
Input:     x[0]  x[1]  x[2]  x[3]  x[4]  x[5]  x[6]  ...
           ↓
Anti-alias filter (LPF at π/M)
           ↓
Downsampled: y[0]=filtered[0]  y[1]=filtered[M]  y[2]=filtered[2M] ...
```

**Filter design for downsampling:** The anti-alias filter must have:
- Passband: 0 to $\pi/M$ (ideally)
- Stopband: $\pi/M$ to $\pi$
- Stopband attenuation: ≥ 80 dB for high-quality conversion

### 2.3 Combined Rational Ratio P/Q

For a rational ratio $f_{out}/f_{in} = P/Q$, the optimal approach is:
1. Upsample by $P$
2. Apply a low-pass filter with cutoff $\pi / \max(P, Q)$
3. Downsample by $Q$

The combined filter design determines quality. The filter must be wide enough to pass the audio band (up to approximately 0.9 × Nyquist of the input) while sufficiently attenuating images/aliases.

### 2.4 Real-World Rational Ratios

The ratios between common audio sample rates are derived from their greatest common divisors:

| From → To | GCD | Upsample P | Downsample Q | Result |
|-----------|-----|-----------|-------------|--------|
| 44100 → 48000 | 147 | 160 | 147 | 160/147 ≈ 1.08844 |
| 48000 → 44100 | 147 | 147 | 160 | 147/160 ≈ 0.91875 |
| 44100 → 96000 | 441 | 640 | 294 | 640/294 ≈ 2.17687 |
| 96000 → 44100 | 294 | 147 | 640 | 147/640 ≈ 0.22969 |
| 48000 → 96000 | 48 | 2 | 1 | 2/1 = integer |
| 88200 → 48000 | 6 | 8 | 15 | 8/15 ≈ 0.53333 |
| 176400 → 44100 | 147 | 1 | 4 | 1/4 = integer |
| 192000 → 48000 | 48 | 1 | 4 | 1/4 = integer |

The ratio 160/147 is particularly important since it appears in CD-to-DVD/video conversions. These two numbers (160 and 147) are prime to each other, meaning a polyphase filter bank with 160 upsampling phases and 147 downsampling phases is needed for perfect conversion.

---

## 3. POLYPHASE FIR FILTERS

### 3.1 The Polyphase Decomposition

A polyphase FIR filter decomposes a filter's coefficients into $P$ phase groups, where $P$ is the number of output samples per input sample period (for upsampling) or the number of input samples consumed per output sample (for downsampling).

Given a filter $h[n]$ of length $N$, the polyphase decomposition is:
$$h_k[n] = h[nP + k] \quad \text{for } k = 0, 1, \ldots, P-1$$

This allows computing each output sample by selecting the appropriate phase and performing a dot product, rather than computing the full convolution each time.

**Polyphase filter bank for 44100 → 48000 conversion (P=160, Q=147):**
- The prototype filter has length $N = P \times Q \times L_{tap}$ where $L_{tap}$ is the number of input samples processed per output
- A practical implementation uses 160 polyphase branches for upsampling
- Each output sample uses branch $m \mod 160$
- The interpolation filter within each branch has $Q \times L_{coeff}$ taps

### 3.2 Filter Implementation

The upsampling process using polyphase filters:

```
Input samples: ... x[n-1]  x[n]  x[n+1] ...
                    ↓
           Polyphase demultiplexer
                    ↓
            ┌───────┬───────┬─────┬───────┐
Phase 0:    │ h[0]  │ h[P]  │...  │ h[(L-1)P] │
Phase 1:    │ h[1]  │ h[P+1]│...  │           │
...
Phase P-1:  │ h[P-1]│h[2P-1]│...  │           │
            └───────┴───────┴─────┴───────┘
                    ↓
           Each phase drives output at its time slot
                    ↓
Output samples: ... y[mP/denom]  y[(m+1)P/denom]  ...
```

### 3.3 libsamplerate (Secret Rabbit Code) — Complete Reference

libsamplerate, authored by Erik de Castro Lopo, is the gold-standard open-source SRC library. It implements the Julius O. Smith bandlimited interpolation algorithm.

#### 3.3.1 Available Converter Types

```c
enum SRC_CONVERTER_TYPE {
    SRC_SINC_BEST_QUALITY   = 0,   // Highest quality, slowest
    SRC_SINC_MEDIUM_QUALITY = 1,   // Good quality, moderate speed
    SRC_SINC_FASTEST        = 2,   // Fastest sinc, lower quality
    SRC_ZERO_ORDER_HOLD     = 3,   // Step interpolation (not recommended)
    SRC_LINEAR              = 4,   // Linear interpolation (not recommended)
};
```

#### 3.3.2 Quality Specifications

| Converter | SNR | Bandwidth | Polyphase Count | Filter Taps | Speed |
|-----------|-----|----------|----------------|-------------|-------|
| SRC_SINC_BEST_QUALITY | 144 dB | 96% of Nyquist | 2381 phases | ~64 taps per phase | ~40× realtime |
| SRC_SINC_MEDIUM_QUALITY | 121 dB | 90% of Nyquist | 491 phases | ~32 taps per phase | ~140× realtime |
| SRC_SINC_FASTEST | 97 dB | 80% of Nyquist | 64 phases | ~16 taps per phase | Fastest |
| SRC_LINEAR | ~60 dB | Full Nyquist | — | — | Fastest |
| SRC_ZERO_ORDER_HOLD | ~40 dB | Full Nyquist | — | — | Fastest |

**Bandwidth definition:** Bandwidth here refers to the percentage of the output Nyquist frequency that is preserved without attenuation. A 96% bandwidth means the filter passes frequencies up to 0.96 × (output_sample_rate / 2) with minimal attenuation.

#### 3.3.3 SNR Derivation

The Signal-to-Noise Ratio of bandlimited sinc interpolation depends on:
$$SNR \approx 6.02 \times B - 10 \log_{10}(\Delta_f) - 20 \log_{10}(\delta) \text{ dB}$$

Where:
- $B$ = number of bits in the input signal
- $\Delta_f$ = normalized transition width of the filter
- $\delta$ = filter passband/stopband ripple

The high-quality sinc converters achieve 144 dB SNR by using very narrow transition bands and long filters with low ripple.

#### 3.3.4 Algorithm Details — Julius O. Smith Method

The algorithm works by:

1. **Prototype filter design:** A bandlimited FIR filter is designed with a very narrow transition band. This filter is oversampled by a factor of $P$ (number of phases), producing $P$ sets of coefficients.

2. **Phase selection:** For each output sample at fractional position $\mu$, the algorithm selects the nearest phase $p = \lfloor \mu \times P \rfloor$ from the polyphase bank.

3. **Linear interpolation between phases:** Since $\mu$ rarely lands exactly on a phase boundary, the algorithm linearly interpolates between phase $p$ and phase $p+1$ (mod $P$):
$$h_\mu[n] = (1 - f) \cdot h_p[n] + f \cdot h_{p+1}[n]$$
where $f = (\mu \times P) - p$ is the fractional part.

4. **Convolution:** The interpolated filter is convolved with the input buffer to produce the output sample.

#### 3.3.5 libsamplerate API

```c
#include <samplerate.h>

// Simple API — one-shot conversion
int src_simple(float *data_out, float *data_in,
               int converter_type, int channels,
               long input_frames, long output_frames,
               double input_ratio);

// Full API — streaming/consecutive conversion
SRC_STATE *src_new(int converter_type, int channels, int *error);
long src_process(SRC_STATE *state, SRC_DATA *data);
void src_delete(SRC_STATE *state);

typedef struct {
    float  *data_in;       // Input buffer
    float  *data_out;      // Output buffer
    long    input_frames;   // Frames in input buffer
    long    output_frames;  // Frames in output buffer
    long    input_frames_used;  // Frames actually consumed
    long    output_frames_gen;  // Frames actually produced
    double  src_ratio;          // output_sample_rate / input_sample_rate
    int     end_of_input;       // Set to 1 when no more input
} SRC_DATA;
```

**Example usage:**

```c
#include <samplerate.h>
#include <stdio.h>

int convert_samplerate(const float *input, float *output,
                       int channels, long input_frames,
                       double from_rate, double to_rate)
{
    SRC_DATA src_data = {0};
    src_data.data_in = (float *)input;
    src_data.data_out = output;
    src_data.input_frames = input_frames;
    src_data.src_ratio = to_rate / from_rate;

    // Determine output buffer size
    src_data.output_frames = (long)(input_frames * src_data.src_ratio) + 100;

    SRC_STATE *state = src_new(SRC_SINC_BEST_QUALITY, channels, &(int){0});
    if (!state) return -1;

    int error = src_process(state, &src_data);
    if (error) { src_delete(state); return error; }

    printf("Consumed %ld input frames, generated %ld output frames\n",
           src_data.input_frames_used, src_data.output_frames_gen);

    src_delete(state);
    return 0;
}
```

#### 3.3.6 Channel Handling

libsamplerate handles multichannel audio by processing each channel independently through the same SRC state. Channels are interleaved in the data buffers:

```c
// For stereo (channels = 2), data layout is:
// [L0, R0, L1, R1, L2, R2, ...]
```

The library supports up to 64 channels, and the quality of the conversion is identical across all channels.

---

## 4. INTERPOLATION METHODS

### 4.1 Zero Order Hold (ZOH)

Zero order hold is the simplest interpolation method. Each input sample is held constant until the next input sample arrives:

$$y[t] = x[\lfloor t \rfloor]$$

**Frequency response of ZOH:**
$$H_{ZOH}(\omega) = e^{-j\omega/2} \cdot \frac{\sin(\omega/2)}{\omega/2}$$

The sinc-shaped rolloff of ZOH causes significant high-frequency attenuation. The -3 dB point occurs at approximately 0.443 × Nyquist. This method is NOT suitable for audio.

### 4.2 Linear Interpolation

Linear interpolation connects adjacent samples with straight lines:

$$y[t] = x[\lfloor t \rfloor] + (t - \lfloor t \rfloor) \cdot (x[\lfloor t \rfloor + 1] - x[\lfloor t \rfloor])$$

**Frequency response:** Linear interpolation acts as a sinc-squared filter:
$$H_{LIN}(\omega) = \left(\frac{\sin(\omega/2)}{\omega/2}\right)^2$$

The -3 dB point is at approximately 0.67 × Nyquist. Linear interpolation produces a gentle rolloff but introduces inter-symbol interference and group delay distortion.

### 4.3 Lagrange Interpolation

Lagrange interpolation fits a polynomial through surrounding samples. The $N$-point Lagrange interpolator is:

$$y(t) = \sum_{k=0}^{N-1} x[k] \cdot L_k(t)$$

where $L_k(t)$ is the Lagrange basis polynomial. For 4-point Lagrange (cubic):

$$L_k(t) = \prod_{\substack{0 \leq m < N \\ m \neq k}} \frac{t - m}{k - m}$$

**Coefficients for 4-point cubic Lagrange (with t in [0,1]):**

| Index | Coefficient |
|-------|------------|
| $L_0(t)$ | $-t(t-1)(t-2)/6$ |
| $L_1(t)$ | $(t+1)(t-1)(t-2)/2$ |
| $L_2(t)$ | $-t(t+1)(t-2)/2$ |
| $L_3(t)$ | $t(t+1)(t-1)/6$ |

Lagrange interpolation is computationally efficient and was used in some early digital audio equipment. However, it is not bandlimited and produces aliasing artifacts.

### 4.4 Bandlimited Interpolation (Sinc-Based)

Bandlimited interpolation is the gold standard for high-quality SRC. It uses the sinc function as the interpolation kernel:

$$\text{sinc}(x) = \frac{\sin(\pi x)}{\pi x}$$

The ideal reconstruction of a bandlimited signal from samples is:

$$x(t) = \sum_{n=-\infty}^{\infty} x[n] \cdot \text{sinc}(t - n)$$

**Practical sinc interpolation** truncates the infinite sinc to a finite window of $2M+1$ samples and applies a window function:

$$y(t) = \sum_{n=-M}^{M} x[n] \cdot w[n] \cdot \text{sinc}(t - n)$$

Common window functions:

| Window | Formula | Stopband Attenuation | Mainlobe Width |
|--------|---------|---------------------|----------------|
| Rectangular | $w[n] = 1$ | -21 dB | Narrowest |
| Hann | $w[n] = 0.5(1 - \cos(2\pi n/N))$ | -44 dB | Medium |
| Hamming | $w[n] = 0.54 - 0.46\cos(2\pi n/N)$ | -53 dB | Medium |
| Kaiser-Bessel | $w[n] = I_0(\beta\sqrt{1-(2n/N)^2})/I_0(\beta)$ | Variable (60–100+ dB) | Variable |

### 4.5 Windowed Sinc Interpolation Algorithm

For fractional delay $\mu$ (where $0 \leq \mu < 1$) and filter half-length $M$:

```
For each output sample:
    1. Determine the center sample index: n0 = floor(t)  (t is continuous time)
    2. For k = -M to M:
         mu_k = mu - k
         h[k] = sinc(mu_k) × window(k)
    3. y[t] = Σ(k=-M to M) x[n0 - k] × h[k]
```

The window controls the trade-off between:
- **Transition width:** How sharply the filter transitions from passband to stopband
- **Stopband attenuation:** How much aliasing energy is suppressed
- **Passband ripple:** How flat the frequency response is in the passband

---

## 5. ANTI-ALIASING FILTER DESIGN

### 5.1 The Need for Anti-Aliasing

When downsampling, any frequency content above the new Nyquist frequency (output_rate / 2) must be removed before decimation. Failure to do so causes aliasing — frequencies fold back into the audible range as inharmonic, non-musical artifacts.

**Aliasing formula:**
$$f_{alias} = |f_{signal} - k \cdot f_s|$$

where $f_s$ is the output sample rate and $k$ is the nearest integer to $f_{signal} / f_s$.

### 5.2 FIR Filter Design for Decimation

The anti-aliasing filter for downsampling by $M$ must satisfy:

1. **Passband:** 0 to $f_c = f_{in} / (2M)$ (the output Nyquist)
2. **Transition band:** $f_c$ to $f_{transition}$ (typically $f_c + \Delta f$)
3. **Stopband:** $f_{transition}$ to $f_s/2$ (the input Nyquist)

The transition band width $\Delta f$ determines the filter length:
$$N \approx \frac{D \cdot f_s}{\Delta f}$$

Where $D$ is the parameter related to stopband attenuation:
- 60 dB attenuation: $D \approx 4.0$
- 80 dB attenuation: $D \approx 5.0$
- 100 dB attenuation: $D \approx 6.0$

**Example:** Downsampling from 192000 Hz to 48000 Hz (M=4):
- $f_c = 192000 / 8 = 24000$ Hz (output Nyquist)
- For 80 dB stopband with $\Delta f = 4000$ Hz: $N \approx 5.0 \times 192000 / 4000 = 240$ taps

### 5.3 Kaiser Window Parameters

The Kaiser window is commonly used for audio resampling filter design:

$$\beta = \begin{cases} 0.1102(A - 8.7) & A > 50 \\ 0.5842(A - 21)^{0.4} + 0.07886(A - 21) & 21 \leq A \leq 50 \\ 0.0 & A < 21 \end{cases}$$

Where $A$ = required stopband attenuation in dB.

| Stopband Attenuation | β (Kaiser parameter) | Filter Length Factor |
|---------------------|---------------------|---------------------|
| 40 dB | 2.12 | 2.0 |
| 50 dB | 3.40 | 2.5 |
| 60 dB | 4.54 | 3.0 |
| 70 dB | 5.68 | 3.5 |
| 80 dB | 6.76 | 4.0 |
| 90 dB | 7.86 | 4.5 |
| 100 dB | 8.96 | 5.0 |

### 5.4 Polyphase Implementation of Anti-Aliasing

For rational-ratio resampling, the anti-aliasing filter and interpolation are combined into a single polyphase filter bank. This is computationally efficient because:

1. Only the active polyphase branch is computed per output sample
2. No temporary high-rate signal buffer is needed
3. The filter computation and decimation/interpolation happen simultaneously

The combined filter design must satisfy both:
- Passband ripple: < 0.01 dB for transparent quality
- Stopband attenuation: ≥ 100 dB to prevent audible aliasing
- Transition width: narrow enough to preserve full audio bandwidth

---

## 6. PHASE RESPONSE IN RESAMPLING

### 6.1 Linear Phase Resampling

Most high-quality resamplers (including libsamplerate sinc converters) use linear phase filters. A linear phase filter has a group delay that is constant across frequency:

$$\tau_g(\omega) = -\frac{d\phi(\omega)}{d\omega} = \frac{N-1}{2} \text{ samples (constant)}$$

This means all frequency components of the signal are delayed by the same amount. The output is a time-shifted version of the ideal resampled signal.

**Advantage:** No phase distortion — the waveform shape is preserved.
**Disadvantage:** Linear phase filters have symmetric coefficients, requiring pre-ringing (pre-echo) on transient signals.

### 6.2 Minimum Phase Resampling

Minimum phase filters have all energy as early as possible. They are used when:
- Pre-ringing is undesirable (percussive audio)
- Time alignment with other tracks is critical
- Lower algorithmic latency is needed

A minimum phase filter is derived from a linear phase filter by:
1. Taking the square root of the squared magnitude response
2. Converting back to time domain using homomorphic techniques
3. Ensuring all zeros are inside or on the unit circle

**Note:** libsamplerate uses linear phase sinc filters. Minimum phase resampling is not natively supported.

### 6.3 Constant Group Delay Approximation

Even in linear phase filters, the actual group delay can vary slightly from the ideal constant due to:
- Finite filter length
- Windowing effects
- Transition band shape

For audio applications, this variation is typically < 0.1 samples and is not perceptible. In measurement applications (e.g., acoustic measurement), minimum phase corrections may be applied.

---

## 7. SAMPLE RATE CONVERSION ARTIFACTS

### 7.1 Imaging (Upsonic Artifacts)

When upsampling, the zero-inserted signal has spectral images at multiples of the original Nyquist frequency. For 2× upsampling:

```
Original spectrum (input at fs):
  |-----bandwidth-----|

Upsampled spectrum (after zero-insertion):
  |bandwidth|  0  |bandwidth|  0  |bandwidth|
  ←──── image 1 ────→    ←──── image 2 ────→
```

The anti-alias/interpolation filter must attenuate these images below the noise floor. Insufficient stopband attenuation causes "birdie" artifacts — audible spectral copies of the audio content at high frequencies.

**Detection:** Analyze the output with a spectrogram. Images appear as mirrored copies of the audio band above the new Nyquist.

### 7.2 Aliasing (Downsonic Artifacts)

When downsampling, insufficient pre-filtering causes frequency content above the new Nyquist to fold back into the audible band. For 4× downsampling from 192000 to 48000 Hz:

- Nyquist of 48000 Hz = 24000 Hz
- Frequencies between 24000 and 96000 Hz in the input will alias
- A 36000 Hz tone (well above 24000 Hz Nyquist) aliases to 12000 Hz (audible):
  $$f_{alias} = |36000 - 2 \times 24000| = |36000 - 48000| = 12000 \text{ Hz}$$

This creates entirely new frequency components not present in the original — a form of distortion that is especially noticeable on high-frequency content like cymbals.

### 7.3 Pre-Ringing and Post-Ringing

Linear phase FIR filters cause symmetric ringing around transients:

```
Input:   |    pulse    |
         ↓
Output:  |~pre-ring~ pulse ~post-ring~|
```

The number of ringing cycles is approximately $N / 2$ where $N$ is the filter length in samples. For a 192000 Hz input with a 256-tap filter:
- Ringing duration = 256 / 2 / 192000 ≈ 0.67 ms

This pre-ringing is sometimes audible as a "smearing" of transient attacks (drums, plucked strings, consonants in vocals).

**Solutions:**
- Use shorter filters (but sacrifices stopband attenuation)
- Use minimum phase filters (reduces pre-ringing at cost of phase distortion)
- Over-sample before filtering to spread ringing over fewer audible cycles

### 7.4 Frequency Response Irregularities

Imperfect filter design causes:
- **Passband ripple:** Variation of ±0.01 to ±0.1 dB in the passband — may be audible as a slight comb filtering
- **Rolloff near Nyquist:** Most filters gently roll off approaching the passband edge
- **Group delay variation:** Slight smearing at the top of the passband

For the highest-quality converters (SRC_SINC_BEST_QUALITY), the passband ripple is typically < 0.0005 dB, which is below perceptual threshold.

---

## 8. HIGH-QUALITY RESAMPLING PARAMETERS

### 8.1 libsamplerate Quality Selection Guide

| Use Case | Converter | Bandwidth | Stopband | Notes |
|----------|-----------|-----------|----------|-------|
| Archival / mastering | SRC_SINC_BEST_QUALITY | 96% | -144 dB | Transparent for 24-bit |
| Critical listening | SRC_SINC_MEDIUM_QUALITY | 90% | -121 dB | Excellent for 16-bit |
| Real-time playback | SRC_SINC_FASTEST | 80% | -97 dB | Acceptable for playback |
| Quick preview | SRC_LINEAR | Full | -60 dB | Noticeable artifacts |
| Benchmark only | SRC_ZERO_ORDER_HOLD | Full | -40 dB | Never use for audio |

### 8.2 Varispeed vs Sample Rate Conversion

**Varispeed** (playing audio at a different rate than it was recorded) changes both pitch and tempo:

```
varispeed: output_rate = input_rate × speed_multiplier
pitch_change = speed_multiplier - 1 (in semitones: 12 × log2(speed_multiplier))
```

**SRC** (true sample rate conversion) changes only the sample rate while preserving pitch and tempo. The output audio is a reconstruction of what the original analog signal would look like if it had been recorded at the target sample rate.

**Distinction in FFmpeg:**

```bash
# Varispeed: changes pitch and tempo (2× playback = pitch up one octave)
ffmpeg -i input.wav -filter:a "atempo=2.0" output.wav

# True SRC: preserves pitch and tempo
ffmpeg -i input.wav -ar 96000 output.wav
```

### 8.3 Bit-Depth Considerations During SRC

SRC does not change bit depth, but the internal computation must use sufficient precision:
- Input: 16-bit PCM → internal: 32-bit float minimum
- Input: 24-bit PCM → internal: 32-bit float recommended
- Input: 32-bit float → internal: 64-bit double recommended

The filter computations in libsamplerate use 32-bit floating point by default. For 24-bit or higher source material, the intermediate accumulation uses 64-bit floating point to prevent rounding errors from accumulating in the long filter convolutions.

---

## 9. FFMPEG RESAMPLING

### 9.1 FFmpeg Resampler Architecture

FFmpeg provides two resampling engines through libswresample:

1. **swr (Native SW Resampler):** The default, built-in resampler. Uses FIR filters with configurable parameters.
2. **soxr (SoX Resampler Library):** Optional, higher-quality resampler based on libsoxr.

```bash
# Default resampler (swr)
ffmpeg -i input.wav -ar 48000 output.wav

# Explicitly use swr
ffmpeg -i input.wav -af aresample=resampler=swr:out_sample_rate=48000 output.wav

# Use soxr (higher quality)
ffmpeg -i input.wav -af aresample=resampler=soxr:out_sample_rate=48000 output.wav
```

### 9.2 FFmpeg aresample Filter Options

```bash
# Syntax
ffmpeg -i input -af aresample[=options] output

# Key options (swr engine)
aresample=resampler=swr
    filter_size=32         # Resampling filter length (default: 32)
    phase_shift=10         # log2 of phase count (default: 10, i.e., 1024 phases)
    linear_interp=0/1      # Use linear interpolation (default: 1)
    exact_rational=0/1     # Use exact phase count (default: 1)
    cutoff=0.97           # Cutoff frequency ratio (default: 0.97)
    filter_type=0         # 0=linear, 1=zero_hold, 2=kaiser (default: 0)
    kaiser_beta=16.0      # Kaiser window beta parameter (default: 16.0)
    dither_method=0       # Dithering method (default: 0=none)

# Key options (soxr engine)
aresample=resampler=soxr
    precision=28           # Computation precision in bits (default: 20, max: 28)
    cheby=0               # Chebyshev passband (default: 0=none)
    steep=0               # Steep rolloff (default: 0)
    cutoff=0.91           # Cutoff ratio (default: 0.91)
```

### 9.3 SWR Filter Design Parameters

**`filter_size`:** Number of taps in the resampling FIR filter. Default: 32. Range: 1–32 for swr.

- Larger = sharper transition band, more computation
- At filter_size=32, the transition band is very steep
- For input rates > 96000 Hz, a smaller filter may be acceptable

**`phase_shift`:** Controls the number of polyphase branches. phase_count = 2^phase_shift. Default: 10 → 1024 phases.

**`cutoff`:** The cutoff frequency as a ratio of the output Nyquist. Default: 0.97.
- swr: 0.97 means the filter passes 97% of the audio band (up to 0.97 × Nyquist)
- soxr: 0.91 means the filter passes 91% of the audio band to 20 kHz at 44100 Hz

### 9.4 soxr vs swr — Quality Comparison

| Property | swr (default) | soxr |
|----------|---------------|------|
| Algorithm | Polyphase FIR | Multi-stage (Polyphase + IIR) |
| Passband flatness | ±0.01 dB | ±0.001 dB |
| Stopband rejection | ~-100 dB | ~-200 dB |
| Alias rejection | Good | Excellent |
| Pre-ringing | Moderate | Low |
| Speed | Fast | Moderate |
| Precision setting | Fixed | 20 or 28 bits |
| Compensation | Supported | Not supported |

**FFmpeg soxr precision levels:**

| Precision | Equivalent Quality | Use Case |
|-----------|-------------------|---------|
| 20 bits (default) | High Quality | General purpose, 16-bit output |
| 28 bits | Very High Quality | Professional, 24-bit output |

```bash
# Very high quality conversion using soxr at 28-bit precision
ffmpeg -i input.flac \
  -af aresample=resampler=soxr:precision=28:out_sample_rate=96000 \
  -c:a pcm_s24le output.wav
```

### 9.5 FFmpeg `-ar` Flag vs `-af aresample`

| Feature | `-ar` flag | `-af aresample` |
|---------|-----------|-----------------|
| Position in pipeline | Automatic (optimal placement) | Explicit in filter graph |
| Quality control | Default swr | Engine and options selectable |
| Format conversion | Limited | Full control |
| Multi-stage processing | No | Yes |
| Dithering options | Limited | Full |

```bash
# Simple sample rate change (uses -ar internally)
ffmpeg -i input.wav -ar 48000 output.wav

# Full control with aresample filter
ffmpeg -i input.wav \
  -af "aresample=out_sample_rate=48000:resampler=soxr:precision=28" \
  -c:a pcm_s24le output.wav
```

---

## 10. DSD TO PCM CONVERSION

### 10.1 DSD (Direct Stream Digital) Overview

DSD stores audio as a 1-bit sigma-delta modulated bitstream at very high sample rates:
- DSD64: 2822400 Hz (64 × 44100)
- DSD128: 5644800 Hz (128 × 44100)
- DSD256: 11289600 Hz (256 × 44100)
- DSD512: 22579200 Hz (512 × 44100)

DSD's noise spectrum is shaped so that quantization noise is pushed to high frequencies, leaving a low-noise region in the audible band.

### 10.2 DSD to PCM Conversion Process

Converting DSD to PCM involves:
1. **Decimation:** The DSD bitstream is averaged over groups of bits to produce multi-bit samples at a lower rate
2. **Low-pass filtering:** Remove out-of-band noise above the new Nyquist
3. **Anti-alias filtering:** Prevent aliasing from any remaining high-frequency content

**DSD64 → PCM 28224 Hz:** Integer decimation by 100 (2822400/28224 = 100)
**DSD64 → PCM 352800 Hz:** Integer decimation by 8 (2822400/352800 = 8)
**DSD64 → PCM 176400 Hz:** Integer decimation by 16 (2822400/176400 = 16)

### 10.3 DSD64 to PCM 352.8 kHz (Industry Standard)

The most common high-resolution DSD-to-PCM conversion target is 352.8 kHz (8× CD rate):

$$352800 = \frac{2822400}{8}$$

This is an integer ratio — simple 8-sample averaging is technically possible. However, a proper reconstruction filter (not just averaging) produces better results because it accounts for the DSD noise shaping.

### 10.4 DSD to PCM in FFmpeg

```bash
# DSD to PCM conversion using FFmpeg
ffmpeg -i input.dsf -acodec pcm_s24le -ar 352800 output.wav

# DSD64 to PCM 176.4 kHz
ffmpeg -i input.dsf -acodec pcm_s24le -ar 176400 output.wav

# Multiple DSD-to-PCM conversion steps
ffmpeg -i input.dsf \
  -af "aresample=in_sample_rate=2822400:out_sample_rate=352800:resampler=soxr:precision=28" \
  output.wav
```

### 10.5 Sigma-Delta Modulation Theory

The DSD bitstream is the output of a sigma-delta modulator:

```
         ┌─────────────┐
x[n] ──→ │  Σ (accum) │ ──→ ┌──────────┐ ──→ DSD bitstream
         └──────┬──────┘     │ Quantizer │     ↑
                │             └──────────┘     │
                │                  ↑           │
                └──────────────────┘           │
                   (feedback loop)            │
                                              │
            ←←← Low-pass filter ←←←←←←←←←←←←←←
```

The sigma-delta modulator's noise transfer function shapes quantization noise to high frequencies. The signal transfer function is unity in the passband.

---

## 11. ULTRA-HIGH SAMPLE RATE CONSIDERATIONS

### 11.1 Sample Rates Beyond 192 kHz

The primary argument for sample rates above 192 kHz (384000 Hz, 768000 Hz) is:
1. **Simpler analog anti-aliasing:** Wider transition band in the reconstruction DAC
2. **Reduced timing jitter sensitivity:** More samples per period of a given frequency
3. **Intermodulation products:** Some argue that ultrasound intermodulates down into the audible range

The counter-argument (supported by psychoacoustic research) is:
1. The human ear cannot detect frequencies above ~20 kHz
2. The audible benefit of >192 kHz is negligible even for ultrasound-sensitive individuals
3. Recording at >192 kHz increases file sizes and processing load without proportional benefit

### 11.2 DXD (Digital eXtreme Definition)

DXD uses 352800 Hz sample rate at 24-bit or 32-bit float. It was developed by Merging Technologies as a recording and editing format that avoids DSD's limitations while offering higher resolution than traditional PCM:

| Format | Sample Rate | Bit Depth | Data Rate (stereo) |
|--------|-------------|-----------|--------------------|
| CD | 44100 Hz | 16-bit | 1411 kbps |
| DVD-Audio | 192000 Hz | 24-bit | 9216 kbps |
| DXD | 352800 Hz | 24-bit | 16934 kbps |
| DSD64 | 2822400 Hz | 1-bit | 5645 kbps |
| DSD128 | 5644800 Hz | 1-bit | 11290 kbps |

### 11.3 Conversion Chain Fidelity

When converting through multiple sample rates, each SRC stage introduces a small amount of distortion. For the highest fidelity:

1. Convert directly from source rate to target rate in one step
2. Avoid cascading conversions (e.g., 44100 → 48000 → 96000 → 192000)
3. Use the highest-quality converter available (SRC_SINC_BEST_QUALITY or soxr 28-bit)
4. Use at least 24-bit precision throughout the chain
5. Apply dither when reducing bit depth

---

## 12. IMPLEMENTATION REFERENCE

### 12.1 libsamplerate Integration

```c
// Complete example: high-quality SRC with libsamplerate
#include <samplerate.h>
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    double input_rate;
    double output_rate;
    int channels;
    int quality;  // 0=best, 1=medium, 2=fastest
    SRC_STATE *state;
} SRC_Context;

SRC_Context *src_init(double from_rate, double to_rate,
                      int channels, int quality)
{
    SRC_Context *ctx = calloc(1, sizeof(SRC_Context));
    ctx->input_rate = from_rate;
    ctx->output_rate = to_rate;
    ctx->channels = channels;
    ctx->quality = quality;

    int error;
    static const int type_map[] = {
        SRC_SINC_BEST_QUALITY,
        SRC_SINC_MEDIUM_QUALITY,
        SRC_SINC_FASTEST
    };
    ctx->state = src_new(type_map[quality], channels, &error);
    if (!ctx->state) {
        fprintf(stderr, "SRC init failed: %s\n", src_strerror(error));
        free(ctx);
        return NULL;
    }
    return ctx;
}

long src_process_buffer(SRC_Context *ctx,
                        const float *input, long input_frames,
                        float *output, long output_frames_max)
{
    SRC_DATA data = {0};
    data.data_in = (float *)input;
    data.data_out = output;
    data.input_frames = input_frames;
    data.output_frames = output_frames_max;
    data.src_ratio = ctx->output_rate / ctx->input_rate;

    int error = src_process(ctx->state, &data);
    if (error) {
        fprintf(stderr, "SRC process failed: %s\n", src_strerror(error));
        return -1;
    }
    return data.output_frames_gen;
}

void src_free(SRC_Context *ctx)
{
    if (ctx) {
        if (ctx->state) src_delete(ctx->state);
        free(ctx);
    }
}
```

### 12.2 FFmpeg libswresample C API

```c
#include <libswresample/swresample.h>
#include <libavutil/opt.h>

SwrContext *swr_alloc_set_opts(
    NULL,                          //-or- existing SwrContext
    out_channel_layout,            // e.g., AV_CH_LAYOUT_STEREO
    out_sample_fmt,               // e.g., AV_SAMPLE_FMT_S32
    out_sample_rate,              // e.g., 48000
    in_channel_layout,            // e.g., AV_CH_LAYOUT_STEREO
    in_sample_fmt,                // e.g., AV_SAMPLE_FMT_FLTP
    in_sample_rate,               // e.g., 44100
    0, NULL);                     // log_offset, log_ctx

swr_init(ctx);

// Process:
int swr_convert(ctx, output, output_count,
                input, input_count);

// Cleanup:
swr_free(&ctx);
```

### 12.3 FFmpeg soxr Integration

```c
// To use soxr, set the engine before initialization:
av_opt_set_int(ctx, "engine", SWR_ENGINE_SOXR, AV_OPT_SEARCH_CHILDREN);
av_opt_set_int(ctx, "precision", 28, AV_OPT_SEARCH_CHILDREN);
swr_init(ctx);
```

---

## 13. ARTIFACT DIAGNOSIS GUIDE

### 13.1 Identifying SRC Quality Problems

| Symptom | Likely Cause | Solution |
|---------|-------------|----------|
| High-frequency "birdie" tones | Insufficient image rejection in upsampler | Use higher-quality SRC |
| New mid/high frequencies not in original | Aliasing from downsampler | Use better anti-alias filter |
| Transient "smearing" | Pre-ringing from long FIR filter | Use minimum phase or shorter filter |
| Subtle pitch instability | Timing jitter in conversion | Use synchronous clock if possible |
| "Glassy" or "metallic" timbre | Insufficient stopband attenuation | Use SINC_BEST_QUALITY or soxr |
| Bass sounds "loose" | Phase distortion | Use linear phase (default) |
| Phase shift on transients | Minimum phase resampler | Use linear phase instead |

### 13.2 Listening Test Protocol

To verify SRC quality perceptually:
1. Use program material with known high-frequency content (acoustic instruments, cymbals)
2. Convert A→B→A (e.g., 44100 → 48000 → 44100) — should be perceptually identical to A
3. Use ABX testing with a reliable reference
4. Test at least 10 participants for statistical significance
5. A properly implemented SRC_SINC_BEST_QUALITY or soxr converter should be perceptually transparent for 16-bit and higher content

### 13.3 FFT Verification

```python
import numpy as np

def verify_src_quality(input_file, output_file, from_rate, to_rate):
    """Verify SRC by comparing spectral content in overlap region."""
    # Read both files (same duration)
    input_spectrum = compute_fft(input_file)
    output_spectrum = compute_fft(output_file)

    # For upsampling: check that original band matches
    # For downsampling: check no aliasing in preserved band

    # Compute ratio spectrum
    ratio = output_spectrum / input_spectrum

    # Report passband deviation
    print(f"Passband deviation: {20*np.log10(np.abs(ratio[passband])).std():.3f} dB")
    print(f"Stopband level: {20*np.log10(np.abs(ratio[stopband]).mean()):.1f} dB")
```

---

## 14. REFERENCE IMPLEMENTATIONS COMPARISON

| Library | Algorithm | Quality | Speed | License | Language |
|---------|-----------|---------|-------|---------|---------|
| libsamplerate (SRC) | Julius O. Smith bandlimited sinc | Excellent (144 dB SNR) | Moderate | BSD-2 | C |
| FFmpeg swr | Polyphase FIR | Good (~100 dB SNR) | Fast | LGPL 2.1+ | C |
| FFmpeg soxr | Multi-stage polyphase + IIR | Very good (~140 dB SNR) | Moderate | BSD-3 | C |
| SoX | Multi-stage resampling | Good (~130 dB SNR) | Fast | BSD-2 | C++ |
| r8brain (free) | Lagrange + IIR | Good | Fast | MIT | C++ |
| Secret Rabbit Code | See libsamplerate | See libsamplerate | — | BSD-2 | C |

---

## 15. MATHEMATICAL REFERENCE

### 15.1 Sinc Function

$$\text{sinc}(x) = \frac{\sin(\pi x)}{\pi x}$$

Properties:
- $\text{sinc}(0) = 1$ (by limit)
- $\text{sinc}(n) = 0$ for all non-zero integers $n$
- $\int_{-\infty}^{\infty} \text{sinc}(x) \, dx = 1$

### 15.2 Polyphase Filter Bank — Signal Flow

```
Input samples x[n]
     │
     ▼
┌────────────────────┐
│  Delay chain:      │
│  x[n] x[n-1] ...  │
│  x[n-M]            │
└────────┬───────────┘
         │
         ▼
   ┌───────────────┐
   │ Polyphase     │
   │ Demultiplexer │
   └───────┬───────┘
           │
    ┌──────┼──────┬──────┬───────┐
    ▼      ▼      ▼      ▼       ▼
  Phase0 Phase1 ... Phase(P-1)
    │      │      │      │       │
    └──────┴──────┴──────┴───────┘
              │
              ▼
       Multiply-Accumulate
              │
              ▼
     Output sample y[m]
```

### 15.3 Complete SRC Algorithm

```
Input: x[n], input_rate, output_rate, filter h[n]
Output: y[m]

1. Compute ratio: R = output_rate / input_rate
2. Find integer approximation: P/Q ≈ R
3. Build polyphase filter bank: h_k[n] = h[nP + k]
4. For each output sample m:
   a. Compute time position: t = m / output_rate
   b. Find input sample index: n0 = floor(t × input_rate)
   c. Compute fractional position: mu = (t × input_rate) - n0
   d. Select phase: p = round(mu × P) mod P
   e. Compute fractional part: frac = (mu × P) - p
   f. Interpolate filter: h_int[n] = (1-frac)×h_p[n] + frac×h_{p+1}[n]
   g. Compute output: y[m] = Σ(k) x[n0-k] × h_int[k]
```

---

*File generated for: Audio Engineering Knowledge Base*
*Topic: Sample Rate Conversion, Polyphase FIR, libsamplerate, FFmpeg aresample*
*[NEEDS VERIFICATION] markers indicate specific numerical values that require additional source confirmation*
*[NEEDS VERIFICATION]: libsamplerate SNR figures for SRC_SINC_MEDIUM_QUALITY and SRC_SINC_FASTEST converters — the GitHub source references 121 dB and 97 dB respectively, but these should be verified against the official API documentation at mega-nerd.com/SRC*
