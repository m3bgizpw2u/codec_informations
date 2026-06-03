# Audio Engineering: Psychoacoustics & Perceptual Coding — Deep Technical Reference
> **Category:** AudioEngineering
> **File Extensions:** N/A (theory reference document)
> **MIME Types:** N/A
> **Standardization Body:** ISO/IEC, ITU-R, AES
> **Primary Specification:** ISO/IEC 11172-3 (MPEG-1 Audio), ISO/IEC 13818-7 (AAC), Johnston "Estimation of Perceptual Entropy Using Noise Masking Criteria" (IEEE ICASSP 1986)
> **Patent Status:** Varies by codec implementation
> **License:** Theory/standards are public domain; codec implementations carry their own licenses
> **Current Version:** Theoretical framework well-established; active in codec development
> **Active Development:** Yes — codec algorithms continue to evolve

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** James A. Moorer, James D. Johnston (Bell Labs), Heinrich von Helmholtz (original hearing research), Harvey Fletcher (masking studies)
- **Year Created:** 1970s–1980s (modern perceptual audio coding: 1986 onward)
- **Original Purpose:** To understand and exploit the limitations of human hearing to compress digital audio beyond what purely mathematical (lossless) compression could achieve, enabling near-transparent audio at 128–192 kbps compared to 1,411 kbps for CD-DA.
- **Problem with Predecessors:** Early audio codecs (e.g., PCM, ADPCM) treated audio as pure information theory without considering human perception, wasting bits on inaudible details.

### 1.2 Key Historical Milestones
|| Year | Milestone | Significance |
|------|---------|-------------|
| 1863 | Helmholtz "On the Sensations of Tone" | First comprehensive theory of auditory perception |
| 1920s–1950s | Fletcher & Munson hearing curves | Absolute threshold of hearing, equal-loudness contours |
| 1960s | Critical band theory (Zwicker) | Understanding frequency resolution of the cochlea |
| 1986 | Johnston "Perceptual Entropy" paper | First formal framework for quantifying audible information |
| 1988 | MPEG-1 Audio Layer III (MP3) | First widely-deployed perceptual audio codec |
| 1994 | MPEG-2 AAC | Extended bandwidth, improved psychoacoustic model |
| 1999 | MP3 becomes ISO/IEC standard | Global adoption of perceptual coding |

### 1.3 Current Adoption
- **Primary use cases today:** Streaming (Spotify, Apple Music, YouTube), digital radio (DAB+), TV broadcast (AAC), VoIP (Opus), game audio, AR/VR
- **Platforms with native support:** Every modern device — smartphones, smart speakers, streaming platforms, game consoles
- **Major services using perceptual coding:** All lossy audio services (Spotify, Apple Music, Amazon Music, Tidal, YouTube, Netflix, Disney+)
- **Hardware support:** Dedicated DSP chips in smartphones, AV receivers, car audio systems, Bluetooth audio codecs
- **Status:** Foundational technology for all modern digital audio delivery

---

## 2. THE HUMAN AUDITORY SYSTEM — ANATOMY & PHYSIOLOGY

### 2.1 Outer Ear Anatomy
```
Pinna (Auricle)
    │
    ▼
External Auditory Canal (ear canal)
    │  ~2.5 cm length
    │  Resonant frequency: ~2.5–3.5 kHz (boosts ~10–15 dB)
    ▼
Tympanic Membrane (eardrum)
```

The outer ear performs three acoustic functions:
1. **Sound collection:** Pinna funnels sound into the ear canal
2. **Frequency filtering:** Head-related transfer function (HRTF) provides spatial cues
3. **Resonance boost:** Ear canal acts as a closed tube resonator at ~2.7 kHz

### 2.2 Middle Ear Anatomy
```
Tympanic Membrane (eardrum)
    │
    ▼
Malleus (hammer) → Incus (anvil) → Stapes (stirrup)
    │                    │              │
    └────────────────────┴──────────────┘
                    Ossicle chain
                          │
                          ▼
               Oval Window (fenestra ovalis)
                          │
                          ▼
                    Inner Ear (cochlea)
```

**Middle ear transformer function:**
- The ossicle chain provides ~1.5:1 mechanical advantage
- Area ratio of eardrum (~55 mm²) to oval window (~3.2 mm²) = ~17:1
- Total pressure gain: ~1.5 × 17 = ~25:1 (≈28 dB)
- **Purpose:** Match impedance of air (low) to cochlear fluid (high)

### 2.3 Inner Ear — The Cochlea
The cochlea is the critical organ for frequency analysis:

```
Cochlea (spiral, ~35 mm uncoiled)
    │
    ├── Vestibular Canal (scala vestibuli)
    │         │
    │         │ ← Filled with perilymph (high Na⁺, low K⁺)
    │         ▼
    ├── Reissner's Membrane (separates vestibular from media)
    │         │
    │         ▼
    ├── Media Canal (scala media)
    │         │ ← Filled with endolymph (low Na⁺, high K⁺, +80 mV)
    │         │
    │         ▼
    │  Basilar Membrane
    │    │
    │    │ ← Frequency selectivity; stiff at base, compliant at apex
    │    ▼
    └── Tympanic Canal (scala tympani)
              │ ← Filled with perilymph
              ▼
         Round Window (releases pressure)
```

### 2.4 The Basilar Membrane — Frequency Analysis
The basilar membrane is the key frequency-analyzing structure:

**Location-dependent frequency tuning:**
| Position | Frequency | Characteristics |
|----------|-----------|----------------|
| Base (near oval/round windows) | 20,000 Hz | Narrow, stiff, fast response |
| Middle | 1,000–4,000 Hz | Moderate width |
| Apex (cochlear tip) | 20–200 Hz | Wide, compliant, slow response |

**Response characteristics:**
- Low frequencies (20–500 Hz): Broad response region, many hair cells stimulated
- High frequencies (5,000–20,000 Hz): Narrow response region, precise localization
- **Critical bandwidth:** The bandwidth of the region maximally stimulated by a tone

---

## 3. FREQUENCY PERCEPTION & CRITICAL BANDS

### 3.1 The Bark Scale (Critical Band Rate)
The Bark scale divides the audible frequency range (20 Hz–16 kHz) into 24 critical bands plus a partial 25th band. It models the frequency resolution of the auditory system:

**Bark Scale Formula:**
```
z(f) = 13 × arctan(0.00076 × f) + 3.5 × arctan((f/7500)²)

where:
  z = critical band rate in Barks
  f = frequency in Hz
```

**Simplified approximation:**
```
z(f) ≈ (26.81 × f) / (1960 + f) - 0.53
```

**Critical Band Frequencies (Zwicker/Terhardt):**
|| Band | Center Freq (Hz) | Lower Edge (Hz) | Upper Edge (Hz) | Bandwidth (Hz) |
||------|-----------------|-----------------|-----------------|----------------|
| 1 | 50 | 20 | 100 | 80 |
| 2 | 150 | 100 | 200 | 100 |
| 3 | 250 | 200 | 300 | 100 |
| 4 | 350 | 300 | 400 | 100 |
| 5 | 450 | 400 | 510 | 110 |
| 6 | 570 | 510 | 630 | 120 |
| 7 | 700 | 630 | 770 | 140 |
| 8 | 840 | 770 | 920 | 150 |
| 9 | 1000 | 920 | 1080 | 160 |
| 10 | 1170 | 1080 | 1270 | 190 |
| 11 | 1370 | 1270 | 1480 | 210 |
| 12 | 1600 | 1480 | 1720 | 240 |
| 13 | 1850 | 1720 | 2000 | 280 |
| 14 | 2150 | 2000 | 2320 | 320 |
| 15 | 2500 | 2320 | 2700 | 380 |
| 16 | 2900 | 2700 | 3150 | 450 |
| 17 | 3400 | 3150 | 3700 | 550 |
| 18 | 4000 | 3700 | 4400 | 700 |
| 19 | 4800 | 4400 | 5300 | 900 |
| 20 | 5800 | 5300 | 6400 | 1100 |
| 21 | 7000 | 6400 | 7700 | 1300 |
| 22 | 8500 | 7700 | 9500 | 1800 |
| 23 | 10500 | 9500 | 12000 | 2500 |
| 24 | 13500 | 12000 | 15500 | 3500 |

### 3.2 Why Critical Bands Matter for Coding
- **Bit allocation:** Allocate more bits to critical bands where the ear is most sensitive
- **Masking analysis:** Spreading function operates on Bark scale, not linear Hz
- **Threshold calculation:** Perceptual entropy computed per critical band
- **Stereo redundancy:** Inter-channel correlation varies by critical band

---

## 4. ABSOLUTE THRESHOLD OF HEARING (ATH)

### 4.1 Definition
The Absolute Threshold of Hearing (ATH) is the minimum sound pressure level (SPL) detectable by a healthy young listener in a noiseless environment, presented binaurally via headphones, for each frequency.

### 4.2 The ATH Formula (Terhardt et al., 1979)
```
ATH(f) = 3.64 × (f/1000)^(-0.8) - 6.5 × exp(-0.6 × (f/1000 - 3.3)²) + 10^(-3) × (f/1000)^4  [dB SPL]

where:
  f = frequency in Hz (valid range: 20 Hz to 16 kHz)
  ATH(f) = absolute threshold in dB SPL
```

**Simplified form (from MPEG psychoacoustic model 1):**
```
ATH(f) ≈ 3.64 × (f/1000)^(-0.8)  [dB SPL]
```

### 4.3 ATH Values at Standard Frequencies
|| Frequency (Hz) | ATH (dB SPL) | Notes |
|---------------|---------------|-------|
| 20 | ~78 | Near limit of hearing |
| 100 | ~26 | Good sensitivity |
| 1000 | ~4 | Most sensitive region |
| 2000 | ~3 | Near minimum |
| 3000 | ~3 | Near minimum |
| 4000 | ~4 | Near minimum |
| 8000 | ~12 | Decreased sensitivity |
| 16000 | ~20 | High-frequency rolloff |

### 4.4 ATH in Perceptual Coding
The ATH serves as a lower bound for all masking calculations:

```
Global Masking Threshold = max(ATH(f), sum of all individual masking thresholds)
```

In codec implementation:
- ATH sets the minimum audibility floor
- Any quantization noise below ATH is perceptually irrelevant
- For a 16-bit PCM signal normalized to ±1.0:
  - Smallest sine wave amplitude: 1/2^15
  - Power: (1/2^15)²/2 = 1/2^31 ≈ -93.32 dB
  - Model ATH ≈ -90 dB at 1 kHz, so signal is near the threshold
  - Codec can potentially ignore quantization noise below ~-90 dB at this frequency

### 4.5 Factors Affecting ATH
- **Age:** Hearing loss reduces sensitivity, especially above 4 kHz
- **Gender:** Slight differences in average ATH
- **Listening conditions:** Anechoic vs. normal room (room adds ~5–10 dB)
- **Presentation level:** Stark's law — threshold rises at very low levels
- **Duration:** Signals shorter than ~200 ms require higher SPL to reach threshold

---

## 5. MASKING PHENOMENA

### 5.1 Definition of Masking
**Masking** occurs when the presence of one sound (the **masker**) reduces the audibility of another sound (the **maskee**). The **masking threshold** is the level at which the maskee becomes just audible.

**Key terminology:**
- **Masker:** The sound that interferes with perception of another sound
- **Maskee:** The sound being masked
- **Masking threshold:** The level at which the maskee is just detectable
- **Signal-to-Mask Ratio (SMR):** Signal level minus masking threshold (in dB)

### 5.2 Simultaneous (Frequency-Domain) Masking
Simultaneous masking occurs when masker and maskee are present at the same time.

**Types of simultaneous masking:**
1. **Tonal masking noise:** A pure tone masks nearby noise
2. **Noise masking tone:** Broadband noise masks a tonal signal
3. **Tone-on-tone:** One tone masks another nearby tone

**Masking spread characteristics:**
- **Upward spread (downward masking):** A low-frequency masker spreads to affect higher frequencies more strongly than the reverse
- **Downward spread (upward masking):** A high-frequency masker has less effect on lower frequencies
- **Spread asymmetry:** The spread is approximately 100 dB/decade toward higher frequencies and 25 dB/decade toward lower frequencies

### 5.3 The Spreading Function
The cochlear spreading function models how masker energy spreads across critical bands:

**Mathematical model ( Johnston 1988, simplified):**
```
S(i, j) = 15.81 + 7.5 × (j - i + 0.474) - 17.5 × sqrt(1 + (j - i + 0.474)²)  [dB]

where:
  i = Bark index of the masker band
  j = Bark index of the masked band
  S(i, j) = masking contribution from band i to band j (dB, negative)
```

**Simplified triangular spreading:**
```
For |i - j| < 0.5: S(i,j) =  0 dB  (direct masking)
For 0.5 < |i - j| < 1.5: S(i,j) = -6 dB per Bark
For |i - j| > 1.5: S(i,j) = -24 dB per Bark
```

**Visualization of spreading function:**
```
         |     /\
Masker→  |    /  \         ← Spreads asymmetrically
         |   /    \____
         |  /          \____
         └──────────────────────
              Bark index →
         
         More spread upward (higher Bark = higher frequency)
```

### 5.4 Temporal (Time-Domain) Masking
Temporal masking occurs when masker and maskee are separated in time.

**Pre-masking (backward masking):**
- Maskee occurs BEFORE the masker
- Maximum pre-masking: ~50 ms before masker onset
- Effect decays rapidly as time gap increases
- **Explanation:** The auditory system "expects" the masker and suppresses prior perception

**Post-masking (forward masking):**
- Masker occurs BEFORE the maskee
- Maximum post-masking: ~200 ms after masker offset
- Effect decays slowly
- **Explanation:** Cochlear excitation persists after stimulus; neural adaptation

**Temporal masking curves:**
```
Masker
Start →───────────────┐
                      │
Duration               │ ← Pre-masking zone (~50 ms)
                      │
                    ──┘
                      │
                    ──┐ ← Post-masking zone (~200 ms)
                      │
                    ──┘
                    
Time →
    ← Pre-masking   ← Post-masking →
```

### 5.5 Combined Masking Model
The total masking threshold is computed as:
```
T_total(f) = max(ATH(f), T_masked(f))

where:
  T_masked(f) = 10 × log10( Σ 10^(S_i/10) ) + T_i
  S_i = spreading function contribution from band i
  T_i = individual masking threshold of band i
```

---

## 6. EXCITATION PATTERNS & TONAL VS NOISE COMPONENTS

### 6.1 Excitation Patterns
An **excitation pattern** is the pattern of neural activity along the basilar membrane produced by a sound. It is the physiological correlate of the perceptual masking pattern.

**Computing excitation patterns:**
```
E(f) = Σ S(f, f_i) × P(f_i)

where:
  E(f) = excitation level at frequency f (Bark scale)
  S(f, f_i) = spreading function
  P(f_i) = power spectrum of input signal at frequency f_i
```

**Excitation pattern analysis in codecs:**
- Johnston's Perceptual Entropy uses excitation patterns to estimate audibility
- High excitation regions → more masking available → fewer bits needed
- Low excitation regions → near ATH → more bits required

### 6.2 Tonal Component Detection
Perceptual codecs must distinguish between tonal (sinusoidal) and noise-like components because they have different masking characteristics.

**Tonal detection algorithm (simplified):**
```
1. Compute FFT of input block (1024 or 2048 points at 44.1 kHz)
2. Find local maxima in magnitude spectrum
3. For each local maximum:
   a. Check if it exceeds neighboring bins by threshold (e.g., 7 dB)
   b. If yes → classify as tonal
   c. If no → classify as noise
4. Remove tonal components from spectrum for noise floor estimation
5. Compute SFM (Spectral Flatness Measure) for each critical band
   SFM = Gm / Am  (geometric mean / arithmetic mean of power spectrum)
   SFM_dB = 10 × log10(SFM)
   Low SFM → tonal, High SFM → noise
```

**Spectral Flatness Measure formula:**
```
SFM = (Π P[k])^(1/N) / (Σ P[k] / N)  [ratio]

SFM_dB = 10 × log10(SFM)  [dB, typically -60 to 0 dB]

where:
  P[k] = power spectrum bins in the critical band
  N = number of bins in the critical band
  SFM_dB → 0 dB: pure tone (all energy in one frequency)
  SFM_dB → -60 dB: broadband noise (geometric mean approaches 0)
```

### 6.3 Tonal vs Noise Masking Thresholds
Once components are classified, different masking indices are applied:

| Masker Type | Masking Noise Threshold | Masking Tone Threshold |
|-------------|-------------------------|------------------------|
| Tonal (tone mask noise) | 14.5 + i dB below excitation | — |
| Noise (noise mask tone) | — | 5.5 dB below excitation |

where i = Bark frequency index (approximately 0–25)

**Implication:** A tonal component masks noise more effectively than a noise component masks tones. This asymmetry is exploited in perceptual codecs.

---

## 7. PERCEPTUAL ENTROPY (PE) — JOHNSTON'S FRAMEWORK

### 7.1 Concept Definition
Perceptual Entropy (PE), introduced by James D. Johnston in 1988, quantifies the minimum number of bits per sample required to encode an audio signal transparently using psychoacoustic masking constraints.

**Definition:**
```
PE = Σ max(0, E_j - 10 × log10(TH_j)) × (bandwidth_j / total_bandwidth)

where:
  PE = Perceptual Entropy (bits per sample)
  E_j = excitation level in critical band j (dB)
  TH_j = masking threshold in critical band j (dB)
  The sum is over all critical bands where E_j > TH_j
```

### 7.2 Interpretation
- **PE ≈ 1.0:** Efficient coding possible (e.g., easy music, sparse spectrum)
- **PE ≈ 2.0:** Average music material
- **PE > 2.5:** Difficult material (e.g., complex orchestral, percussive)
- **PE ≈ 4.0:** Maximum (corresponds to 16-bit PCM capacity)

**Johnston's key finding:** Most audio material has PE between 1.8 and 2.2 bits/sample, suggesting that transparent coding at 128–192 kbps (1.5–2.3 bits/sample for stereo at 44.1 kHz) is theoretically possible.

### 7.3 PE vs Bitrate Calculation
```
Required Bitrate = PE × Sample_Rate × Channels  [bps]

Example:
  PE = 2.0
  Sample Rate = 44100 Hz
  Channels = 2 (stereo)
  
  Required Bitrate = 2.0 × 44100 × 2 = 176,400 bps ≈ 172 kbps
```

This matches the practical experience that 192 kbps MP3 is generally transparent for most music.

### 7.4 PE in Modern Codecs
- **MP3 (MPEG-1 Layer III):** Uses PE-like calculations in psychoacoustic model 2
- **AAC (MPEG-2/4):** Advanced psychoacoustic model, PE integrated into bit allocation
- **Opus:** Uses SILK (voice) and CELT (music) models, PE-inspired algorithms
- **Vorbis:** DynamicPsy model, similar to Johnston's approach

---

## 8. NOISE-TO-MASK RATIO (NMR)

### 8.1 Definition
The Noise-to-Mask Ratio (NMR) quantifies the audibility of quantization noise:

```
NMR = Noise_Level - Masking_Threshold  [dB]

NMR < 0: Noise is masked (inaudible)
NMR = 0: Noise is at threshold of audibility
NMR > 0: Noise is audible (quality degradation)
```

### 8.2 Computing NMR in a Codec
```
1. Input: Audio block → FFT → Spectrum P[k]
2. Compute masking threshold T[k] using psychoacoustic model
3. After quantization: Reconstructed spectrum R[k]
4. Quantization error: E[k] = P[k] - R[k]
5. Noise in critical band j:
   N_j = 10 × log10( Σ 10^(E[k]/10) )  [dB]
6. NMR_j = N_j - T_j  [dB]
7. Goal: Minimize bits while keeping max(NMR_j) ≈ 0 dB
```

### 8.3 NMR in Quality Control
Codecs can use NMR as a quality metric:
```
If max(NMR) > 3 dB: Allocate more bits to problematic bands
If max(NMR) < -10 dB: Wasteful — can reduce bits
```

---

## 9. BIT ALLOCATION ALGORITHMS

### 9.1 Overview
Perceptual audio codecs must allocate a limited bit budget across frequency bands to minimize audible distortion. The bit allocation algorithm is the heart of the encoder's psychoacoustic integration.

### 9.2 MPEG-1 Layer III Bit Allocation (Overview)
```
1. Compute SMR for each scale-factor band:
   SMR[band] = Signal_Level[band] - Masking_Threshold[band]

2. Iterate bit allocation:
   while (bits_remaining > 0):
     a. Find band with maximum SMR (most "audible" if quantized poorly)
     b. Allocate one quantization step more to that band
     c. Update SMR based on new step size
     d. Decrement bits_remaining
     e. Continue until bit budget exhausted or all bands exceed threshold

3. Apply outer iteration (if needed):
   - Adjust global gain
   - Redistribute bits across bands
   - Repeat until stable
```

### 9.3 VBR vs CBR Bit Allocation
| Mode | Description | Typical Usage |
|------|-------------|---------------|
| CBR | Fixed bitrate; bit allocation varies to fit budget | Streaming, broadcast |
| VBR | Fixed quality; bitrate varies with content | Archival, on-demand |
| ABR | Average bitrate; hybrid approach | General purpose |

**CBR bit allocation trade-offs:**
- Easy passages: Wasted bits (more than needed for transparency)
- Difficult passages: Insufficient bits (may sound worse than easier material at same bitrate)

**VBR bit allocation trade-offs:**
- Consistent quality across material
- Variable file size
- Requires two-pass encoding for best results (or look-ahead)

### 9.4 Bit Allocation in Opus (SILK + CELT)
Opus uses a rate-distortion optimized bit allocation:
```
R-D optimization criterion:
  Minimize: Σ D_i + λ × R_i

where:
  D_i = perceptual distortion in band i
  R_i = bits allocated to band i
  λ = Lagrange multiplier (controls quality/bitrate trade-off)
```

The algorithm considers:
- Tonal vs noise nature of each band
- Inter-frame correlation (redundancy)
- Inter-channel correlation (stereo redundancy)
- Bitrate constraints from mode (SILK vs CELT)

---

## 10. MDCT AND PSYCHOACOUSTICS

### 10.1 Why MDCT for Audio Coding
The Modified Discrete Cosine Transform is the dominant transform in modern audio codecs because:

1. **Energy compaction:** Most audio signals have energy concentrated in a few transform coefficients
2. **Perfect reconstruction:** TDAC (Time-Domain Aliasing Cancellation) property
3. **44.1 kHz alignment:** MDCT window sizes (512, 1024, 2048 at 44.1 kHz) align with critical bands
4. **Oversampling:** Transform provides frequency resolution matching perceptual needs
5. **Overlap-add:** Smooths quantization artifacts between frames

### 10.2 MDCT Properties
```
MDCT definition (forward):
  X[k] = Σ(n=0 to 2N-1) x[n] · w[n] · cos(π/N × (n + N/2 + 1/2) × (k + 1/2))
  
  where:
    k = 0 to N-1 (frequency bin index)
    N = half-window size (e.g., 512 for 50% overlap, 1024 samples → 512 bins)
    w[n] = window function (sine, KBD, etc.)

IMDCT definition (inverse):
  y[n] = (1/N) × Σ(k=0 to N-1) X[k] · cos(π/N × (n + N/2 + 1/2) × (k + 1/2))
  
  Overlap-add for perfect reconstruction:
    output[n] = y_current[n] + y_previous[n + N]
```

### 10.3 Window Functions
| Window Type | Formula | Characteristics |
|-------------|---------|----------------|
| Sine | sin(π/N × (n + 0.5)) | Standard MDCT |
| KBD (Kaiser-Bessel Derived) | Derived from K-Bessel window | Better spectral leakage control |
| Hann | 0.5 × (1 - cos(2π/N × n)) | Smooth, used in some codecs |
| None (rectangular) | w[n] = 1 | For stationary signals only |

### 10.4 Block Switching
Transitional block types prevent pre-echo artifacts on transient signals:

**Block types in MP3/AAC:**
| Block Type | Window Length | Samples (44.1kHz) | Use Case |
|------------|--------------|-------------------|----------|
| LONG | 2048 | 46.4 ms | Stationary signals |
| START | 2048 | 46.4 ms | Transition after SHORT |
| SHORT | 256 × 8 | 5.8 ms each × 8 | Transient detection |
| STOP | 256 | 5.8 ms | Transition to LONG |

**Pre-echo prevention:**
```
Transient detected → Switch to SHORT blocks
  - Short blocks have better time resolution (5.8 ms)
  - Each block has separate quantization/bit allocation
  - Pre-masking (~50 ms) covers quantization noise before transient
  - Audible artifacts prevented

After transient subsides → Switch to LONG blocks
  - Better frequency resolution for stationary content
```

---

## 11. STEREO REDUNDANCY & JOINT STEREO CODING

### 11.1 Interaural Redundancy
Stereo channels contain correlated information. Exploiting this redundancy reduces bitrate:

**Left/Right vs Mid/Side representation:**
```
M = (L + R) / 2    [Mid — correlated]
S = (L - R) / 2    [Side — often small, high entropy]

L = M + S
R = M - S
```

### 11.2 Stereo Coding Modes
| Mode | Description | Bitrate Savings | Quality Impact |
|------|-------------|-----------------|----------------|
| Independent L/R | Encode each channel separately | 0% (baseline) | Best quality at given bitrate |
| MS (Mid-Side) Stereo | Encode M and S separately | 10–30% | Negligible if side channel is small |
| Intensity Stereo | High-frequency bins merged, position cues | 30–50% | Some spatial quality loss |
| Parametric Stereo | Code L/R correlation as parameters | 50–70% | Significant quality reduction |

### 11.3 When to Use MS Stereo
MS stereo coding is most effective when:
- Low frequencies (< 4 kHz): Where spatial perception is less critical
- Center-panned content (vocals, bass): Side channel is small
- Orchestral/classical music: High inter-channel correlation

MS stereo should be avoided when:
- High-frequency content is uncorrelated (cymbals, room ambience)
- Heavy stereo separation effects are used in the mix

---

## 12. HEARING DAMAGE & NOISE-INDUCED HEARING LOSS

### 12.1 Noise-Induced Hearing Loss (NIHL)
NIHL occurs when the cochlear hair cells are damaged by excessive sound exposure:

**Mechanism:**
- Loud sounds cause excessive basilar membrane vibration
- Hair cell stereocilia are overstressed and break
- Permanent threshold shift (PTS): Irreversible hearing loss
- Temporary threshold shift (TTS): Recovery after rest (warning sign)

**NIHL progression:**
1. **Temporary threshold shift (TTS):** Temporary elevation of hearing threshold after loud exposure
2. **Permanent threshold shift (PTS):** Cumulative, irreversible damage
3. **Notched audiogram:** Characteristic 4 kHz dip from repeated noise exposure

### 12.2 Safe Exposure Limits (OSHA/NIOSH)
| Duration | Permissible Sound Level (dBA) |
|----------|-------------------------------|
| 8 hours | 85 |
| 4 hours | 88 |
| 2 hours | 91 |
| 1 hour | 94 |
| 30 minutes | 97 |
| 15 minutes | 100 |
| 7.5 minutes | 103 |

**Formula:** For every 3 dB increase, safe exposure time halves.

### 12.3 Relevance to Audio Coding
- **Codec loudness wars:** Louder masters → listeners increase volume → potential NIHL
- **Dynamic range reduction:** Heavily compressed music encourages louder playback levels
- **Headphone safety:** Personal audio devices can exceed safe levels
- **Streaming normalization:** Reduces listener need to adjust volume, may help prevent NIHL

---

## 13. THEORETICAL LIMITS OF PERCEPTUAL CODING

### 13.1 Information-Theoretic Bounds
The theoretical minimum bitrate for transparent coding is determined by:
```
R_min = Perceptual Entropy × Sample_Rate × Channels  [bps]
```

**Empirical observations (Johnston, 1988):**
- Mean PE ≈ 2.0 bits/sample
- 98th percentile PE ≈ 2.23 bits/sample
- Corresponds to: ~172 kbps (mean) to ~195 kbps (98th percentile) for stereo 44.1 kHz

### 13.2 Practical Limits
| Bitrate | Theoretical Quality | Practical Quality |
|---------|-------------------|-------------------|
| 64 kbps | Transparent for simple material | Good for voice, poor for music |
| 128 kbps | Transparent for most material | Acceptable for casual listening |
| 192 kbps | Transparent for nearly all material | Excellent for most listeners |
| 256 kbps | Perceptually lossless | Indistinguishable from lossless |
| 320 kbps | Beyond perception threshold | Transparent for all |

### 13.3 Why Transparent Coding Is Hard
- **Critical listening:** Some listeners can detect artifacts at 192 kbps with trained ears
- **Sensitive material:** Pitched solo instruments, sine wave test tones, speech
- **Transients:** Pre-echo artifacts on attack transients
- **Stereo image:** Joint stereo artifacts can affect spatial perception
- **High frequencies:** cymbal crashes, orchestral air, high harmonics

---

## 14. FFMPEG PSYCHOACOUSTIC MODEL REFERENCE

### 14.1 FFmpeg Native AAC Psychoacoustic Model
FFmpeg's native AAC encoder uses a simplified psychoacoustic model:

```bash
# FFmpeg native AAC encoding:
ffmpeg -i input.wav -c:a aac -b:a 256k output.m4a

# The native encoder's psychoacoustic model:
# - Computes ATH for each frequency
# - Applies spreading function
# - Allocates bits based on SMR
# - Quality is lower than libfdk-aac or reference encoders
```

### 14.2 libfdk-aac Psychoacoustic Model
Fraunhofer FDK AAC uses a more sophisticated model:

```bash
# FDK AAC encoding:
ffmpeg -i input.wav -c:a libfdk-aac -vbr 5 output.m4a

# VBR quality levels:
# 1: 32 kbps  - Voice
# 2: 64 kbps  -对话
# 3: 96 kbps  - Standard music
# 4: 128 kbps - High quality
# 5: 192 kbps - Very high quality
```

### 14.3 libopus Psychoacoustic Model
Opus combines two models:

```bash
# Opus encoding (CELT mode for music):
ffmpeg -i input.wav -c:a libopus -b:a 128k -vbr on output.opus

# Application modes:
# -audio: Full bandwidth, all signal types
# -voip: Optimized for speech
# -lowdelay: Minimum algorithmic delay
```

---

## 15. PRACTICAL APPLICATIONS

### 15.1 Optimal Bitrate Selection
| Use Case | Recommended Format | Bitrate | Rationale |
|----------|-------------------|---------|----------|
| Archival master | FLAC | N/A (lossless) | No quality loss |
| High-quality streaming | Opus | 128–160 kbps | Transparent for most |
| Standard streaming | AAC-LC | 128–192 kbps | Universal compatibility |
| Mobile/low bandwidth | AAC-HE | 48–64 kbps | 2× efficiency |
| Voice messages | Opus | 24–48 kbps | Speech optimized |
| Bluetooth | SBC/aptX | 328 kbps / 350 kbps | Codec dependent |

### 15.2 Perceptual Coding Artifacts
| Artifact | Cause | Detection | Mitigation |
|----------|-------|-----------|------------|
| Pre-echo | Transient quantized before detection | High-frequency "ringing" before drums | Block switching to SHORT |
| Birdie artifacts | Tonal components quantized poorly | Resonating tones, metallic sound | Increase bits for tonal bands |
| Phasing/stereo | Mid-side coding errors | Stereo image distortion | Switch to L/R stereo |
| High-frequency loss | Low-pass filter or bit starvation | Dull, muffled sound | Increase bitrate or cutoff |
| Burble | Bit allocation oscillation | Undulating distortion on transients | Smoother bit allocation |

### 15.3 Listening Test Conditions
For valid perceptual audio evaluation:
- **Environment:** Quiet, treated listening room or anechoic chamber
- **Equipment:** Flat-frequency-response monitors or calibrated headphones
- **Level:** Realistic listening level (~75–85 dBA)
- **Reference:** A/B comparison with lossless source
- **Blindness:** Double-blind testing preferred
- **Training:** Listeners should be trained in artifact recognition
- **Duration:** Tests should be long enough to assess all artifact types

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

|| Library | Language | License | Notes |
|---------|----------|---------|-------|
| ISO MPEG Audio | C | Patent encumbered | Reference implementation |
| LAME | C | LGPL | Reference MP3 quality |
| libvorbis | C | BSD | Reference Vorbis quality |
| libopus | C | BSD | Reference Opus quality |
| FFmpeg native | C | LGPL | Various quality |
| libfdk-aac | C | Non-free | Reference AAC quality |

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specifications
- **ISO/IEC 11172-3:** MPEG-1 Audio (Layer I, II, III)
- **ISO/IEC 13818-7:** MPEG-2 AAC
- **ISO/IEC 23003-1:** USAC (Unified Speech and Audio Coding)
- **ITU-R BS.1387:** Methods for the subjective assessment of audio quality
- **ITU-R BS.1770:** Algorithms for loudness measurement

### Academic Papers
- Johnston, J.D. "Estimation of Perceptual Entropy Using Noise Masking Criteria," IEEE ICASSP, 1986
- Johnston, J.D. "Transform Coding of Audio Signals Using Perceptual Noise Criteria," IEEE JSAC, 1988
- Zwicker, E. & Fastl, H. "Psychoacoustics: Facts and Models," Springer, 1990
- Brandenburg, K. & Bosi, M. "Overview of MPEG Audio: Past, Present and Future," AES preprint, 1993
- Painter, T. & Spanias, A. "Perceptual Coding of Digital Audio," Proc. IEEE, 2000

### Technical Resources
- Hydrogenaudio Knowledgebase: https://wiki.hydrogenaud.io/
- MP3 Tech: http://www.mp3-tech.org/
- IEEE Audio, Speech, and Language Processing: https://ieeexplore.ieee.org/

---

## 18. IMPLEMENTATION CHECKLIST (Audio Converter Developer)

### Psychoacoustic Integration
- [ ] Understand critical band masking for frequency-domain codecs
- [ ] Implement or use ATH computation
- [ ] Implement spreading function for mask estimation
- [ ] Detect tonal vs noise components
- [ ] Compute SMR for each band
- [ ] Allocate bits based on SMR
- [ ] Implement block switching for transient handling
- [ ] Test for pre-echo artifacts on test signals

### Quality Verification
- [ ] Conduct listening tests with diverse material
- [ ] Use standardized test signals (sinusoids, sweeps, transients)
- [ ] Compare against reference encoders (LAME, FDK AAC)
- [ ] Test corner cases: solo instruments, full orchestras, voice, silence
- [ ] Implement ABX testing framework for developer verification

### Bitrate Optimization
- [ ] Analyze PE distribution across your audio library
- [ ] Choose bitrate that achieves PE ≈ 2.0 for transparency target
- [ ] Consider VBR for consistent quality
- [ ] Profile difficult material (high PE) to understand limits

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*

---

## 17. MATHEMATICAL MODELS — DETAILED FORMULAS

### 17.1 Complete ATH Formula Derivation
The Absolute Threshold of Hearing formula has been refined over decades:

```python
def ATH_terhardt(f):
    """
    Calculate ATH using Terhardt's formula.
    
    Args:
        f: Frequency in Hz
    
    Returns:
        ATH in dB SPL
    """
    f_kHz = f / 1000.0
    
    # Terhardt's formula (1979):
    ath = 3.64 * f_kHz**(-0.8) \
          - 6.5 * np.exp(-0.6 * (f_kHz - 3.3)**2) \
          + 10**(-3) * f_kHz**4
    
    return ath

def ATH_rubinstein(f):
    """
    Calculate ATH using Rubinstein's approximation.
    Used in some perceptual audio coders.
    
    Args:
        f: Frequency in Hz
    
    Returns:
        ATH in dB SPL
    """
    f_kHz = f / 1000.0
    
    # Rubinstein approximation:
    if f < 2000:
        ath = 0.0
    else:
        ath = 0.85 * np.log10(f_kHz) + 0.8
    
    return ath
```

### 17.2 Critical Band Calculation
The Bark scale conversion formulas:

```python
def hz_to_bark(f):
    """
    Convert frequency (Hz) to critical band rate (Bark).
    Formula from Zwicker & Terhardt (1980).
    
    Args:
        f: Frequency in Hz
    
    Returns:
        Critical band rate in Barks
    """
    if f < 0.5:
        return 0.0
    
    z = 13.0 * np.arctan(0.00076 * f) \
        + 3.5 * np.arctan((f / 7500.0)**2)
    
    return z

def bark_to_hz(z):
    """
    Convert critical band rate (Bark) to frequency (Hz).
    Inverse of hz_to_bark.
    
    Args:
        z: Critical band rate in Barks
    
    Returns:
        Frequency in Hz
    """
    f1 = 1960.0 / (13.0 - z) * z
    f2 = (7500.0**0.5) * np.tan(z / 3.5)
    
    # Blend the two formulas
    f = 0.5 * (f1 + f2)
    
    return f

def critical_bandwidth(f):
    """
    Calculate critical bandwidth at a given frequency.
    
    Args:
        f: Frequency in Hz
    
    Returns:
        Critical bandwidth in Hz
    """
    z = hz_to_bark(f)
    
    # Approximation using Bark scale:
    # BW ≈ 52548 / (z² - 40z + 1100)
    if z > 0:
        bw = 52548.0 / (z**2 - 40*z + 1100)
    else:
        bw = 100  # Minimum bandwidth
    
    return bw
```

### 17.3 Spreading Function Implementation
The auditory spreading function models masking spread:

```python
def spreading_function(z_masker, z_target, spreading_type='johnston'):
    """
    Calculate masking spread from masker at z_masker to target at z_target.
    
    Args:
        z_masker: Masker position in Bark
        z_target: Target position in Bark
        spreading_type: 'johnston', 'zwicker', 'moore_glasberg'
    
    Returns:
        Masking level in dB (negative)
    """
    dz = z_masker - z_target  # Bark difference
    
    if spreading_type == 'johnston':
        # Johnston (1988) spreading function:
        S = 15.81 + 7.5 * (dz + 0.474) \
            - 17.5 * np.sqrt(1 + (dz + 0.474)**2)
    
    elif spreading_type == 'zwicker':
        # Zwicker & Fastl approximation:
        if dz >= 0:
            S = 27.0 * dz  # Downward spread (more spread)
        else:
            S = -17.0 * dz  # Upward spread (less spread)
        
        # Apply masker-dependent level
        S = np.clip(S, -60, 0)
    
    elif spreading_type == 'moore_glasberg':
        # Moore & Glasberg (1990) model:
        if dz >= 0:
            S = 0.3 * 15 + 24.0 * (1 - np.exp(-dz / 0.17))
        else:
            S = 0.3 * 15 + 2.5 * dz
    
    return S

def global_spreading_function(z, maskers, masker_levels):
    """
    Calculate global spreading function from multiple maskers.
    
    Args:
        z: Target Bark position
        maskers: List of masker Bark positions
        masker_levels: List of masker levels in dB
    
    Returns:
        Global masking level in dB
    """
    total_power = 0
    
    for zi, Li in zip(maskers, masker_levels):
        Si = spreading_function(zi, z)
        total_power += 10**((Li + Si) / 10)
    
    global_level = 10 * np.log10(total_power + 1e-10)
    
    return global_level
```

### 17.4 Masking Index Calculation
Computing masking indices for tonal vs noise maskers:

```python
def masking_index(f, masker_type='tone'):
    """
    Calculate masking index based on masker frequency and type.
    
    Args:
        f: Masker frequency in Hz
        masker_type: 'tone' or 'noise'
    
    Returns:
        Masking index in dB
    """
    z = hz_to_bark(f)
    
    if masker_type == 'tone':
        # Tonal masker masks noise
        # Masking index varies with frequency
        if z <= 1:
            mi = 6.0
        elif z <= 13:
            mi = 6.0 - 0.25 * (z - 1)
        elif z <= 24:
            mi = 3.0 - 0.175 * (z - 13)
        else:
            mi = 1.0
    
    else:  # noise
        # Noise masker masks tone
        # More constant masking index
        mi = 5.5
    
    return mi

def spectral_flatness_measure(power_spectrum):
    """
    Calculate Spectral Flatness Measure (SFM).
    Used to distinguish tonal from noise-like components.
    
    Args:
        power_spectrum: Power spectrum values
    
    Returns:
        SFM in dB (0 = pure tone, negative = tonal)
    """
    # Geometric mean
    log_spectrum = np.log(power_spectrum + 1e-10)
    geometric_mean = np.exp(np.mean(log_spectrum))
    
    # Arithmetic mean
    arithmetic_mean = np.mean(power_spectrum)
    
    # SFM
    if arithmetic_mean > 0 and geometric_mean > 0:
        sfm = geometric_mean / arithmetic_mean
        sfm_db = 10 * np.log10(sfm)
    else:
        sfm_db = -60  # Minimum value
    
    return sfm_db

def classify_tonal_components(spectrum, threshold_db=-60):
    """
    Classify spectral components as tonal or noise.
    
    Args:
        spectrum: Magnitude spectrum (complex or absolute)
        threshold_db: Threshold for tonal detection
    
    Returns:
        Dictionary with tonal_bins and noise_floor
    """
    power = np.abs(spectrum)**2
    threshold = 10**(threshold_db / 10)
    
    # Find local maxima
    n = len(power)
    is_local_max = np.zeros(n, dtype=bool)
    for i in range(1, n-1):
        if power[i] > power[i-1] and power[i] > power[i+1]:
            is_local_max[i] = True
    
    # Calculate SFM for entire spectrum
    sfm_db = spectral_flatness_measure(power)
    
    # Tonal detection threshold
    sfm_threshold = -30  # dB
    
    if sfm_db < sfm_threshold:
        # Spectrum is tonal
        tonal_bins = np.where(is_local_max & (power > threshold))[0]
        noise_floor = np.zeros(n)
    else:
        # Spectrum is noise-like
        tonal_bins = np.array([])
        noise_floor = power
    
    return {
        'tonal_bins': tonal_bins,
        'noise_floor': noise_floor,
        'sfm_db': sfm_db
    }
```

### 17.5 Perceptual Entropy Calculation
Johnston's Perceptual Entropy computation:

```python
def perceptual_entropy(signal, sample_rate=44100):
    """
    Calculate Perceptual Entropy of an audio signal.
    Based on Johnston (1988).
    
    Args:
        signal: Audio samples
        sample_rate: Sample rate in Hz
    
    Returns:
        Perceptual entropy (bits per sample)
    """
    N = 2048  # FFT size
    overlap = 0.75  # 75% overlap
    hop = int(N * (1 - overlap))
    
    # Window function (Hann)
    window = np.hanning(N)
    
    # Calculate FFT
    spectrum = np.fft.rfft(signal[:N] * window)
    power = np.abs(spectrum)**2 / np.sum(window**2)
    
    # Convert to Bark bands
    freqs = np.fft.rfftfreq(N, 1.0 / sample_rate)
    
    # Calculate masking threshold
    # (simplified - full implementation would include all steps)
    threshold = calculate_masking_threshold(power, freqs)
    
    # Calculate PE
    pe = 0
    for i in range(len(power)):
        if power[i] > threshold[i]:
            signal_power = 10 * np.log10(power[i] + 1e-10)
            threshold_db = 10 * np.log10(threshold[i] + 1e-10)
            smr = signal_power - threshold_db
            pe += max(0, smr) / 10.0
    
    # Convert to bits per sample
    pe_per_sample = pe / (N / 2)
    
    return pe_per_sample

def calculate_masking_threshold(power, freqs):
    """
    Calculate simultaneous masking threshold.
    Simplified implementation.
    
    Args:
        power: Power spectrum
        freqs: Frequency values in Hz
    
    Returns:
        Masking threshold per frequency bin
    """
    # Calculate ATH
    n = len(freqs)
    ath = np.zeros(n)
    for i in range(n):
        if freqs[i] > 20:
            ath[i] = ATH_terhardt(freqs[i])
        else:
            ath[i] = 100  # High threshold below 20 Hz
    
    # Convert ATH to power
    ath_power = 10**((ath - 90) / 10)  # Reference to digital full scale
    
    # Simplified spreading
    # (Full implementation would convolve with spreading function)
    threshold = ath_power
    
    return threshold
```

---

## 18. TEMPORAL MASKING — DETAILED MODELS

### 18.1 Temporal Masking Functions
```python
def temporal_masking_pre(t, t_masker, duration_masker, strength=50e-3):
    """
    Pre-masking (backward masking) function.
    Maskee occurs before masker.
    
    Args:
        t: Time of maskee (relative to masker end)
        t_masker: Masker duration
        strength: Maximum pre-masking strength in dB/ms
    
    Returns:
        Pre-masking threshold multiplier
    """
    if t < -50e-3:  # Before 50ms before masker
        return 0
    elif t < 0:
        # Linear decay from strength to 0
        return strength * abs(t) / 50e-3
    else:
        return 0

def temporal_masking_post(t, t_masker, duration_masker, strength=200e-3):
    """
    Post-masking (forward masking) function.
    Maskee occurs after masker.
    
    Args:
        t: Time of maskee (relative to masker end)
        t_masker: Masker duration
        strength: Maximum post-masking duration in seconds
    
    Returns:
        Post-masking threshold multiplier
    """
    if t > strength:
        return 0
    elif t > 0:
        # Exponential decay
        return np.exp(-t / (strength / 3))
    else:
        return 1.0

def combined_temporal_masking(t, masker_duration):
    """
    Combined temporal masking (pre + post).
    
    Args:
        t: Time of maskee relative to masker center
        masker_duration: Duration of masker in seconds
    
    Returns:
        Combined masking threshold
    """
    # Pre-masking: ~50ms before
    pre_mask = temporal_masking_pre(t, 0, masker_duration)
    
    # Post-masking: ~200ms after
    post_mask = temporal_masking_post(t, 0, masker_duration)
    
    return max(pre_mask, post_mask)
```

### 18.2 Temporal Masking in Codec Design
```python
def temporal_masking_block_switch(fft_data, hop_size, sample_rate):
    """
    Determine block type for MDCT based on temporal masking.
    
    Args:
        fft_data: FFT of input block
        hop_size: Hop size in samples
        sample_rate: Sample rate in Hz
    
    Returns:
        Block type: 'long' or 'short'
    """
    # Detect transients
    energy = np.sum(np.abs(fft_data)**2)
    
    # Energy ratio between adjacent blocks
    # (simplified - actual implementation more complex)
    if energy_ratio > threshold:
        return 'short'  # Transient detected
    else:
        return 'long'  # Stationary

def pre_ech o_prevention(fft_data, block_type):
    """
    Pre-echo prevention based on temporal masking.
    
    When a transient is detected, use shorter blocks
    to prevent quantization noise from spreading before
    the transient (pre-masking covers ~50ms).
    """
    if block_type == 'short':
        # Short blocks have better time resolution
        # Pre-echo is masked by ~50ms pre-masking
        return True
    else:
        return False
```

---

## 19. STEREO AND MULTI-CHANNEL CODING

### 19.1 Intensity Stereo Coding
```python
def intensity_stereo_encode(L, R, sample_rate):
    """
    Intensity stereo encoding for high frequencies.
    Used in MP3, AAC, Vorbis at low bitrates.
    
    Args:
        L, R: Left and right channel samples
        sample_rate: Sample rate
    
    Returns:
        (sum_channel, position_info)
    """
    # Sum (mid) channel
    M = 0.5 * (L + R)
    
    # Position parameter (high frequencies only)
    # Position encodes the spatial location
    # Using energy ratio at high frequencies
    
    # Band-split: separate into low and high bands
    crossover_freq = 3500  # Hz
    
    # Calculate high-frequency energy
    L_hf, L_lf = band_split(L, crossover_freq, sample_rate)
    R_hf, R_lf = band_split(R, crossover_freq, sample_rate)
    
    # Position (intensity ratio)
    E_L = np.sum(L_hf**2)
    E_R = np.sum(R_hf**2)
    E_sum = E_L + E_R
    
    if E_sum > 0:
        position = E_L / E_sum  # 0 = full right, 1 = full left
    else:
        position = 0.5  # Center
    
    return M, position

def intensity_stereo_decode(M, position, band_limiter):
    """
    Decode intensity stereo signal.
    """
    # Reconstruct left and right from sum + position
    p = position  # 0 = full right, 1 = full left
    
    # High-frequency reconstruction
    L_hf = M * (2 * p)
    R_hf = M * (2 * (1 - p))
    
    # Low frequencies remain discrete
    L = band_merge(L_hf, M)
    R = band_merge(R_hf, M)
    
    return L, R
```

### 19.2 MS (Mid-Side) Stereo Coding
```python
def ms_encode(L, R):
    """
    Encode L/R as Mid/Side channels.
    
    M = (L + R) / 2  # Mid
    S = (L - R) / 2  # Side
    """
    M = 0.5 * (L + R)
    S = 0.5 * (L - R)
    
    return M, S

def ms_decode(M, S):
    """
    Decode Mid/Side to L/R.
    
    L = M + S
    R = M - S
    """
    L = M + S
    R = M - S
    
    return L, R

def ms_stereo_bit_allocation(M, S, bit_budget):
    """
    Allocate bits between Mid and Side channels.
    
    Args:
        M: Mid channel data
        S: Side channel data
        bit_budget: Total available bits
    
    Returns:
        (bits_for_M, bits_for_S)
    """
    # Calculate energy in each channel
    energy_M = np.sum(M**2)
    energy_S = np.sum(S**2)
    
    # Side channel often has less energy (high correlation)
    # Allocate bits proportionally to energy
    total_energy = energy_M + energy_S
    
    if total_energy > 0:
        ratio_M = energy_M / total_energy
        ratio_S = energy_S / total_energy
    else:
        ratio_M = 0.5
        ratio_S = 0.5
    
    # Apply some bias toward Mid channel
    # (mid is perceptually more important)
    ratio_M = 0.7 * ratio_M + 0.3
    ratio_S = 1 - ratio_M
    
    bits_M = int(bit_budget * ratio_M)
    bits_S = bit_budget - bits_M
    
    return bits_M, bits_S
```

### 19.3 Binaural Perception in Stereo Coding
```python
def itd_detection(L, R, sample_rate):
    """
    Detect Interaural Time Difference (ITD).
    
    Args:
        L, R: Left and right channel samples
        sample_rate: Sample rate in Hz
    
    Returns:
        ITD in seconds
    """
    # Cross-correlation
    correlation = np.correlate(L, R, mode='full')
    lags = np.arange(-len(L)+1, len(L))
    
    # Find lag at maximum correlation
    max_idx = np.argmax(correlation)
    lag_samples = lags[max_idx]
    
    itd_seconds = lag_samples / sample_rate
    
    return itd_seconds

def ild_calculation(L, R):
    """
    Calculate Interaural Level Difference (ILD).
    
    Args:
        L, R: Left and right channel samples
    
    Returns:
        ILD in dB
    """
    energy_L = np.sum(L**2)
    energy_R = np.sum(R**2)
    
    if energy_R > 0:
        ild = 10 * np.log10(energy_L / energy_R)
    else:
        ild = 0
    
    return ild
```

---

## 20. LOSSLESS PSYCHOACOUSTIC LIMITS

### 20.1 Theoretical Foundations
```python
def shannon_capacity(bandwidth, snr_db):
    """
    Shannon-Hartley theorem for channel capacity.
    
    C = B * log2(1 + S/N)
    
    Where:
        C = capacity in bits/second
        B = bandwidth in Hz
        S/N = signal-to-noise ratio (linear)
    """
    snr_linear = 10**(snr_db / 10)
    capacity = bandwidth * np.log2(1 + snr_linear)
    
    return capacity

def audio_capacity_bits_per_sample(bandwidth_hz, sample_rate, snr_db):
    """
    Calculate theoretical audio capacity.
    
    For CD-quality audio (44.1kHz, 16-bit):
        Bandwidth ≈ 22kHz
        SNR ≈ 96dB
    
    Args:
        bandwidth_hz: Audio bandwidth in Hz
        sample_rate: Sample rate in Hz
        snr_db: Signal-to-noise ratio in dB
    
    Returns:
        Capacity in bits per sample
    """
    # Capacity per second
    capacity_per_second = shannon_capacity(bandwidth_hz, snr_db)
    
    # Capacity per sample
    capacity_per_sample = capacity_per_second / sample_rate
    
    return capacity_per_sample

# Example calculations:
# CD-quality audio:
cd_capacity = audio_capacity_bits_per_sample(22050, 44100, 96)
print(f"CD-quality capacity: {cd_capacity:.2f} bits/sample")

# This represents the theoretical minimum bitrate
# for lossless coding of CD-quality audio
```

### 20.2 Entropy of Audio Signals
```python
def estimate_audio_entropy(signal, block_size=1024):
    """
    Estimate entropy of audio signal.
    
    Args:
        signal: Audio samples
        block_size: Block size for entropy calculation
    
    Returns:
        Average entropy in bits per sample
    """
    n_blocks = len(signal) // block_size
    entropies = []
    
    for i in range(n_blocks):
        block = signal[i*block_size:(i+1)*block_size]
        
        # Calculate histogram
        hist, _ = np.histogram(block, bins=256)
        
        # Normalize to probability
        probs = hist / np.sum(hist)
        probs = probs[probs > 0]  # Remove zeros
        
        # Calculate entropy
        entropy = -np.sum(probs * np.log2(probs))
        entropies.append(entropy)
    
    return np.mean(entropies)

def first_order_entropy(signal):
    """
    Calculate first-order entropy of audio signal.
    Assumes samples are independent (simplified).
    """
    hist, _ = np.histogram(signal, bins=65536)
    probs = hist / np.sum(hist)
    probs = probs[probs > 0]
    
    entropy = -np.sum(probs * np.log2(probs))
    
    return entropy
```

### 20.3 Psychoacoustic Lossless Boundary
```python
def psychoacoustic_lossless_rate(audio, sample_rate):
    """
    Calculate the theoretical minimum rate for psychoacoustically
    lossless coding (transparent at all frequencies).
    
    Args:
        audio: Audio samples
        sample_rate: Sample rate in Hz
    
    Returns:
        Minimum bitrate in kbps
    """
    # Calculate Perceptual Entropy
    pe = perceptual_entropy(audio, sample_rate)
    
    # Minimum rate
    channels = 2  # Stereo
    rate = pe * sample_rate * channels / 1000  # kbps
    
    return rate

# Example:
# Average music has PE ≈ 2.0 bits/sample
# For stereo 44.1kHz:
# Rate = 2.0 * 44100 * 2 / 1000 = 176.4 kbps
```

---

## 21. CODEC IMPLEMENTATION — PRACTICAL GUIDE

### 21.1 MDCT Implementation
```python
def mdct(x, N, window='sine'):
    """
    Modified Discrete Cosine Transform.
    
    Args:
        x: Input samples (2N points)
        N: Half transform size
        window: Window type
    
    Returns:
        N-point MDCT coefficients
    """
    n = len(x)
    if n != 2 * N:
        raise ValueError(f"Input length must be 2N={2*N}, got {n}")
    
    # Window function
    if window == 'sine':
        w = np.sin(np.pi/(2*N) * (np.arange(2*N) + 0.5))
    elif window == 'kbd':
        # Kaiser-Bessel derived window
        w = kbd_window(2*N, N)
    else:
        w = np.ones(2*N)
    
    # Apply window
    xw = x * w
    
    # MDCT formula
    k = np.arange(N)
    n_idx = np.arange(2*N)
    
    X = np.zeros(N)
    for ki in k:
        X[ki] = np.sum(xw * np.cos(
            np.pi/N * (n_idx + 0.5 + N/2) * (ki + 0.5)
        ))
    
    return X

def imdct(X, N, window='sine'):
    """
    Inverse Modified Discrete Cosine Transform.
    Includes TDAC (Time-Domain Aliasing Cancellation).
    
    Args:
        X: MDCT coefficients (N points)
        N: Half transform size
        window: Window type
    
    Returns:
        2N-point output samples
    """
    # Window function
    if window == 'sine':
        w = np.sin(np.pi/(2*N) * (np.arange(2*N) + 0.5))
    else:
        w = np.ones(2*N)
    
    # IMDCT
    n_idx = np.arange(2*N)
    xw = np.zeros(2*N)
    
    for ki, Xk in enumerate(X):
        xw += Xk * np.cos(
            np.pi/N * (n_idx + 0.5 + N/2) * (ki + 0.5)
        )
    
    # Apply window
    x = xw * w
    
    return x / N

def kbd_window(N, alpha=4):
    """
    Generate Kaiser-Bessel derived window.
    
    Args:
        N: Window length
        alpha: Window parameter (4 is common for audio)
    
    Returns:
        KBD window of length N
    """
    # Standard Kaiser-Bessel window
    kaiser = np.kaiser(N, alpha * np.pi)
    
    # Integrate
    cumsum = np.cumsum(kaiser)
    
    # KBD: square root of integrated window
    kbd = np.sqrt(cumsum / cumsum[-1])
    
    # Symmetrize for MDCT use
    # (full KBD window is applied in two halves)
    
    return kbd
```

### 21.2 Bit Allocation Implementation
```python
def perceptual_bit_allocation(spectrum, smr, total_bits, n_bands):
    """
    Perceptual bit allocation algorithm.
    Allocates bits to frequency bands based on SMR.
    
    Args:
        spectrum: Power spectrum
        smr: Signal-to-mask ratio per band
        total_bits: Total available bits
        n_bands: Number of frequency bands
    
    Returns:
        bits_per_band array
    """
    bits = np.zeros(n_bands)
    remaining_bits = total_bits
    
    # Sort bands by SMR (most audible first)
    sorted_indices = np.argsort(smr)[::-1]
    
    # Iterative allocation
    for idx in sorted_indices:
        if remaining_bits <= 0:
            break
        
        # Minimum bits for this band
        min_bits = 1
        
        # Target bits based on SMR
        # Higher SMR = more audible = more bits
        target_bits = int(remaining_bits * (smr[idx] / np.sum(smr)))
        target_bits = max(min_bits, min(target_bits, remaining_bits))
        
        bits[idx] = target_bits
        remaining_bits -= target_bits
    
    return bits
```

### 21.3 Complete Encoder Flow
```python
def perceptual_audio_encoder(signal, target_bitrate, sample_rate):
    """
    Simplified perceptual audio encoder.
    
    Steps:
    1. MDCT
    2. Psychoacoustic analysis
    3. Bit allocation
    4. Quantization
    5. Entropy coding
    
    Args:
        signal: Audio samples
        target_bitrate: Target bitrate in bits/sample
        sample_rate: Sample rate
    
    Returns:
        Encoded bitstream
    """
    # Parameters
    N = 2048  # Transform size
    hop = N // 2
    
    # Frame processing
    n_frames = (len(signal) - N) // hop + 1
    
    for frame in range(n_frames):
        # Get frame
        start = frame * hop
        end = start + N
        frame_data = signal[start:end]
        
        # 1. MDCT
        mdct_coeffs = mdct(frame_data, N//2)
        
        # 2. Psychoacoustic analysis
        ath = calculate_ath(len(mdct_coeffs))
        threshold = calculate_masking_threshold(mdct_coeffs, ath)
        smr = calculate_smr(mdct_coeffs, threshold)
        
        # 3. Bit allocation
        bits_per_sample = target_bitrate
        total_bits = bits_per_sample * len(mdct_coeffs)
        allocated_bits = perceptual_bit_allocation(
            mdct_coeffs, smr, total_bits, len(mdct_coeffs)
        )
        
        # 4. Quantization
        quantized = quantize_mdct(mdct_coeffs, allocated_bits)
        
        # 5. Entropy coding
        bitstream = huffman_encode(quantized)
        
        yield bitstream

def quantize_mdct(coeffs, bits_per_bin):
    """
    Quantize MDCT coefficients.
    """
    step_sizes = 2**(-bits_per_bin)
    quantized = np.round(coeffs / step_sizes)
    
    return quantized
```

---

## 22. REFERENCE IMPLEMENTATIONS

### 22.1 Open-Source Codecs
| Codec | License | Quality | Notes |
|-------|---------|---------|-------|
| LAME | LGPL | Reference | Best MP3 quality |
| libvorbis | BSD | Reference | Best Vorbis quality |
| libopus | BSD | Reference | Best Opus quality |
| libfdk-aac | Non-free | Reference | Best AAC quality |
| FFmpeg | LGPL | Varies | Many codecs |
| FAAC | GPL | Lower | Older AAC encoder |
| TwoLAME | LGPL | Good | MP2 encoder |
| shine | GPL | Lower | Fixed-point MP3 |

### 22.2 Academic Implementations
| Name | Description | URL |
|------|-------------|-----|
| ISO MPEG reference | Official MPEG code | ISO standard |
| Jeff's AAC | Educational AAC | Various |
| OFA | Optimum Fletcher add | Research |

### 22.3 Psychoacoustic Model Libraries
| Library | Description | Language |
|--------|-------------|----------|
| Sound Synthesis | MPEG psychoacoustic | Python |
| Psyclone | FFmpeg psy model | C |
| LAME psy model | LAME psy model | C |

---

*File expanded with: Mathematical models, temporal masking, stereo coding, lossless limits, codec implementation guide, and reference implementations*
