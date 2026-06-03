# Audio Engineering: Dynamic Range & Normalization — Deep Technical Reference
> **Category:** AudioEngineering
> **File Extensions:** N/A (theory reference document)
> **MIME Types:** N/A
> **Standardization Body:** EBU, ITU-R, AES
> **Primary Specification:** EBU R128, ITU-R BS.1770-4, ITU-R BS.1771-2
> **Patent Status:** Standards are public domain; implementations carry their own licenses
> **License:** Standards are freely available; proprietary metering implementations
> **Current Version:** EBU R128 v5.0 (2023), ITU-R BS.1770-4 (2015)
> **Active Development:** Yes — streaming platforms continue to evolve normalization targets

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 The Loudness War
- **Background:** In the 1990s–2010s, commercial music recordings became progressively louder due to competitive loudness dynamics in radio and CD releases. This "loudness war" led to heavily compressed, dynamically constrained master recordings that sacrificed musical expression for perceived loudness.
- **Consequences:** Listener fatigue, reduced dynamic range, hearing damage risk from elevated playback levels, poor playback compatibility across devices.
- **Solution:** Industry-wide adoption of loudness normalization standards to ensure consistent playback levels regardless of the recording's original dynamic range.

### 1.2 Historical Timeline
|| Year | Event | Significance |
|------|---------|-------------|
| 1963 | Fletcher-Munson equal-loudness contours | Basis for loudness measurement |
| 1997 | ISO 532-1 (Zwicker method) | Standardized loudness calculation |
| 2006 | ITU-R BS.1770-1 | First standardized loudness measurement algorithm |
| 2010 | EBU R128 published | Broadcast loudness standard (–23 LUFS target) |
| 2011 | BS.1770-2 | Added gating provisions |
| 2015 | BS.1770-4 | Current version; level measurement standardization |
| 2012+ | Streaming services adopt | YouTube, Spotify, Apple Music adopt normalization |

### 1.3 Current Adoption
- **Primary use cases:** Broadcast (EBU R128), streaming (Spotify, Apple Music, YouTube), film/TV post-production (ATSC A/85), podcast distribution
- **Platforms with native support:** All major streaming services, DAWs with EBU R128 metering, broadcast automation systems
- **Major services using normalization:** Spotify, Apple Music, YouTube, Amazon Music, Tidal, Netflix, Disney+
- **Hardware support:** Modern AV receivers with auto-room correction, smart speakers
- **Status:** Industry standard for audio delivery; loudness normalization is now the norm rather than the exception

---

## 2. DYNAMIC RANGE FUNDAMENTALS

### 2.1 Definition of Dynamic Range
**Dynamic range** is the ratio between the loudest and quietest sounds in an audio signal, typically expressed in decibels (dB):

```
Dynamic Range = 20 × log10(Peak / RMS_Quiet)  [dB]
```

Or equivalently, using the peak level and noise floor:
```
Dynamic Range = Peak_Level_dBFS - Noise_Floor_dBFS  [dB]
```

### 2.2 Dynamic Range of Common Sources
|| Source | Dynamic Range (dB) | Notes |
|--------|------------------|-------|
| Typical pop music | 6–12 | Highly compressed |
| Rock music | 10–15 | Moderate compression |
| Classical symphony | 20–30 | Full dynamic expression |
| Jazz (acoustic) | 15–22 | Natural dynamics |
| Spoken word | 20–30 | Voice only |
| Orchestral forte | 25–35 | Highest peaks |
| Silence (recording) | 50–70 | Background noise floor |
| Best studio recordings | 18–24 | Balanced production |

### 2.3 Dynamic Range vs Loudness
These are related but distinct concepts:

| Property | Dynamic Range | Loudness |
|----------|--------------|----------|
| Definition | Ratio of loud to quiet | Perceived intensity of the whole signal |
| Units | dB (ratio) | LUFS (absolute) |
| Scope | Difference within signal | Average of entire signal |
| Affects | Musical expression, contrast | How loud the playback sounds |
| Normalization target | Preserved or reduced | Standardized across platforms |

**Key insight:** A recording with high dynamic range can have low integrated loudness (e.g., quiet classical music). Loudness normalization aims to make playback levels consistent, not to remove all dynamics.

### 2.4 Crest Factor
**Crest factor** is the ratio between the peak level and the RMS (average) level of a signal:

```
Crest Factor = Peak / RMS  [linear ratio]

Crest Factor_dB = 20 × log10(Peak / RMS)  [dB]
```

**Crest factor examples:**
| Signal Type | Crest Factor (dB) | Notes |
|-------------|------------------|-------|
| Sine wave | 3.01 dB | Single frequency, constant amplitude |
| Pink noise | 6.0–9.0 dB | Random, natural variation |
| White noise | 3.0–6.0 dB | Flat spectrum |
| Percussion (kick) | 12–18 dB | High peaks, short transients |
| Classical music | 15–25 dB | Large variations over time |
| Highly compressed pop | 4–8 dB | Minimal variation |
| Speech | 10–15 dB | Varies with phonemes |

---

## 3. LOUDNESS MEASUREMENT — ITU-R BS.1770

### 3.1 Overview
ITU-R BS.1770 defines the international standard for measuring audio programme loudness. It is the foundation for EBU R128, ATSC A/85, and all streaming platform normalization standards.

### 3.2 K-Weighting Filter
Before loudness calculation, the signal passes through a **K-weighting filter**:

**High-pass filter:**
```
H(z) = (1 - z^(-1)) / (1 - 0.9816 × z^(-1))
  (2nd order IIR, high-pass at ~38 Hz)
```

**High-shelf filter:**
```
G = 4.0 dB boost at 1681.97 Hz
```

**Purpose:** K-weighting approximates the frequency response of the human auditory system for complex sounds and aligns with equal-loudness contours.

### 3.3 Integrated Loudness (Programme Loudness)
The integrated loudness measurement computes the long-term average loudness of the entire programme:

```
Loudness = -0.691 + 10 × log10( Σ_k (G_k × Σ_n (x_k[n]²)) / T )  [LUFS]

where:
  k = channel index
  n = sample index
  G_k = channel weight (see channel weights below)
  x_k[n] = K-weighted sample
  T = measurement duration
```

**Channel weights (per BS.1770-4):**
| Channel Configuration | Channel Weights |
|----------------------|-----------------|
| Stereo (L, R) | G_L = 1.0, G_R = 1.0 |
| 5.1 (L, R, C, LFE, LS, RS) | G_L = 1.0, G_R = 1.0, G_C = 1.0, G_LFE = 0 (ignored), G_LS = 1.41, G_RS = 1.41 |
| 7.1 (standard) | Same as 5.1 with additional channels at 1.0 |
| Mono | G = 1.0 |

### 3.4 Gating
BS.1770 includes two stages of gating to exclude silent or near-silent segments:

**Absolute (or ungated) threshold:** Exclude blocks below –70 LUFS (relative to full scale)

**Relative threshold:** After computing loudness of segments above –70 LUFS, apply a second gate at –10 LU below the ungated value. Only segments above this relative threshold contribute to the final integrated loudness.

```
Gating formula:
  1. Compute absolute gated loudness L_ungated (exclude blocks < –70 LUFS)
  2. Compute relative threshold: T_rel = L_ungated - 10 LU
  3. Final integrated loudness: exclude blocks < T_rel
```

**Purpose:** Gating removes silence and background noise from the loudness measurement, focusing on foreground content.

---

## 4. EBU R128 — EUROPEAN BROADCAST STANDARD

### 4.1 Core Requirements
EBU R128 specifies:

| Parameter | Target | Tolerance | Notes |
|-----------|--------|-----------|-------|
| Integrated Loudness | –23.0 LUFS | ±0.5 LU | ±1.0 LU for live programmes |
| Maximum True Peak | –1 dBTP | ±0.3 dB | For linear audio production |
| Maximum Momentary Loudness | –23 LUFS + 4 LU | — | Warning threshold |
| Maximum Short-term Loudness | –23 LUFS + 3 LU | — | Warning threshold |
| Loudness Range (LRA) | No limit | — | Informational only |

### 4.2 LUFS vs LU
| Unit | Full Name | Meaning |
|------|-----------|---------|
| LUFS | Loudness Units, Full Scale | Absolute loudness referenced to digital full scale |
| LU | Loudness Unit | Relative loudness difference (1 LU = 1 dB change in perceived loudness) |

LUFS is the absolute measurement; LU is the relative unit used for comparing loudness differences.

**Conversion:**
```
Loudness_dBFS = Loudness_LUFS  [when reference is 0 dBFS]
```

### 4.3 Loudness Range (LRA)
Loudness Range measures the dynamic range of the programme in LU:

```
LRA = L_high - L_low  [LU]

where:
  L_high = 95th percentile loudness
  L_low  = 5th percentile loudness
```

**LRA guidelines:**
| Programme Type | Typical LRA (LU) | Notes |
|---------------|-----------------|-------|
| Heavy commercials | 1–3 | Minimal dynamic range |
| Pop music | 3–8 | Moderate compression |
| TV drama | 8–15 | Natural dynamics |
| Classical music | 15–25 | Full orchestral range |
| Film soundtrack | 20–35 | Wide range for cinematic effect |

### 4.4 EBU R128 Mode Meters
EBU Tech 3341 specifies the "EBU Mode" metering:

| Meter Type | Window | Description |
|------------|--------|-------------|
| Momentary | 400 ms (rectangular) | Fast response, shows peaks and dips |
| Short-term | 3-second (sliding window) | Medium-term loudness |
| Integrated | From start to stop | Overall programme loudness |

---

## 5. TRUE PEAK MEASUREMENT

### 5.1 What Is True Peak?
**True Peak** measures the inter-sample peak level of a signal. Standard sample-rate peak meters only measure sample values; true peak reconstructs what would occur between samples at higher sample rates.

```
True Peak = max |x_reconstructed(t)|  for all t in [0, duration]

where:
  x_reconstructed(t) = bandlimited interpolation of samples at 192 kHz+
```

### 5.2 Why True Peak Matters
Digital-to-analog conversion (DAC) reconstructs a continuous signal from discrete samples. A bandlimited signal can have peaks between samples that exceed the sampled peak:

```
Example: 19 kHz sine wave at 44.1 kHz sampling
  - Sampled peak: 0 dBFS
  - True peak: Can exceed 0 dBFS by up to ~3.5 dB
```

This overshoot can cause:
- **DAC clipping** when converting to analog
- **Inter-sample clipping** in digital processing
- **Distortion** in downstream processing

### 5.3 True Peak Measurement Algorithm
```python
# Simplified true peak calculation:
# 1. Oversample to 4× or 192 kHz (whichever is higher)
# 2. Apply K-weighting filter at oversampled rate
# 3. Find maximum absolute value
# 4. Convert to dBTP: TP_dBTP = 20 × log10(max |sample|)

# True Peak = -1.0 dBTP (EBU R128 requirement)
# Measured with 4× oversampling and K-weighting
```

### 5.4 dBTP vs dBFS
| Unit | Definition | Reference Point | Typical Maximum |
|------|-----------|----------------|----------------|
| dBFS | Decibels relative to Full Scale | Digital full scale (0 dBFS) | 0 dBFS (clipping point) |
| dBTP | Decibels (True Peak) | Digital full scale | –1 dBTP (allowed max per R128) |

**Note:** True peak can exceed 0 dBFS but is still measured in dBTP relative to the digital full-scale reference.

---

## 6. FFmpeg LOUDNORM FILTER

### 6.1 Overview
FFmpeg's `loudnorm` filter implements EBU R128 compliant loudness measurement and normalization.

### 6.2 First-Pass Measurement
```bash
# Measure loudness without modifying audio:
ffmpeg -i input.wav -af loudnorm=print_format=json -f null -
```

**Sample output:**
```json
{
  "input_i": "-17.42",
  "input_tp": "-1.20",
  "input_lra": "8.50",
  "input_thresh": "-28.31",
  "output_i": "-23.00",
  "output_tp": "-1.00",
  "output_lra": "8.50",
  "output_thresh": "-33.31",
  "delta_i": "-5.58",
  "delta_tp": "0.20",
  "delta_lra": "0.00"
}
```

### 6.3 Two-Pass Loudness Normalization
For the most accurate normalization, use two passes:

**Pass 1: Measure**
```bash
ffmpeg -i input.wav -af loudnorm=I=-23:TP=-1:LRA=11:print_format=json -f null - 2>&1 | \
  grep -E '"input_i"|"input_tp"|"input_lra"|"input_thresh"'
```

**Pass 2: Normalize**
```bash
ffmpeg -i input.wav \
  -af loudnorm=\
    I=-23:\
    TP=-1:\
    LRA=11:\
    measured_I=-17.42:\
    measured_TP=-1.20:\
    measured_LRA=8.50:\
    measured_thresh=-28.31:\
    offset=0:\
    linear=true:\
    print_format=summary \
  -ar 48000 \
  output.wav
```

### 6.4 Loudnorm Parameters
| Parameter | Default | Description |
|-----------|---------|-------------|
| I | — | Integrated loudness target (LUFS) |
| TP | — | Maximum true peak (dBTP) |
| LRA | — | Loudness range target (LU) |
| measured_I | — | Measured integrated loudness from first pass |
| measured_TP | — | Measured true peak from first pass |
| measured_LRA | — | Measured loudness range from first pass |
| measured_thresh | — | Measured threshold from first pass |
| offset | 0 | Gain offset in dB (for level adjustment) |
| linear | false | Use linear normalization instead of gamma |
| dual_mono | false | Treat dual mono as mono |
| print_format | none | Output format: none, json, summary |

### 6.5 Single-Pass Normalization
For real-time or quick normalization:
```bash
# Single-pass (less accurate, faster):
ffmpeg -i input.wav \
  -af loudnorm=I=-23:TP=-1:LRA=11:linear=true \
  output.wav
```

### 6.6 Streaming Platform Targets
| Platform | Target LUFS | True Peak Limit | Notes |
|----------|-------------|------------------|-------|
| Spotify | –14 | –1 dBTP | User-selectable: –11, –14, –19 LUFS |
| YouTube | –14 | –1 dBTP | Downward only |
| Apple Music | –16 | –1 dBTP | iTunes Sound Check |
| Amazon Music | –11 to –13 | –1 dBTP | Genre-dependent |
| Tidal | –14 | –1 dBTP | Masters tier uses higher |
| Qobuz | –18 | –1 dBTP | Hi-Fi standard |
| Netflix | –27 (film), –24 (TV) | –2 dBTP | Programme-dependent |

**Universal master recommendation:** –14 LUFS, –1 dBTP maximum (covers Spotify, YouTube, Tidal; Apple Music will normalize –16 automatically)

---

## 7. PEAK NORMALIZATION

### 7.1 Peak Normalization Types
| Type | Target | Description |
|------|--------|-------------|
| Maximum Peak | 0 dBFS | Normalize so highest peak = 0 dBFS |
| Peak at –1 dBFS | –1 dBFS | Leave 1 dB headroom for safety |
| Peak at –3 dBFS | –3 dBFS | Common in mastering for dynamics |
| Peak at –6 dBFS | –6 dBFS | Professional broadcast headroom |

### 7.2 FFmpeg Volume and Peak Normalization
```bash
# Normalize peak to 0 dBFS:
ffmpeg -i input.wav -af "volume=normalize=start" output.wav

# Normalize peak to –1 dBFS using volumedetect:
# Step 1: Measure peak
ffmpeg -i input.wav -af volumedetect -f null -
# Output: mean_volume: -12.5 dB, max_volume: -0.5 dB

# Step 2: Calculate gain and apply
GAIN=-0.5  # Target –1 dBFS, current max is –0.5 dBFS
ffmpeg -i input.wav -af "volume=${GAIN}dB" output.wav

# Normalize to –3 dBFS:
ffmpeg -i input.wav -af "volume=${GAIN}dB" output.wav
# GAIN = -3 - (-0.5) = -2.5 dB
```

### 7.3 FFmpeg volumedetect Filter
```bash
ffmpeg -i input.wav -af volumedetect -f null -
```

**Sample output:**
```
[Parsed_volumedetect_0 @ 0x7f...] mean_volume: -18.5 dB
[Parsed_volumedetect_0 @ 0x7f...] max_volume: -1.0 dB
```

**Interpretation:**
- `mean_volume`: Average (RMS-like) level
- `max_volume`: Peak level in dBFS

### 7.4 Peak Normalization vs Loudness Normalization
| Aspect | Peak Normalization | Loudness Normalization |
|--------|-------------------|----------------------|
| Target | Single peak value | Integrated loudness |
| Unit | dBFS | LUFS |
| Preserves dynamics | Yes (scales uniformly) | Partially (LUFS fixed) |
| Simple to implement | Yes | Requires measurement |
| Streaming optimized | No | Yes |
| Album dynamics preserved | Yes | Yes |

---

## 8. RMS NORMALIZATION

### 8.1 RMS Definition
**RMS (Root Mean Square)** represents the average power of a signal:

```
RMS = sqrt(1/N × Σ x[i]²)

RMS_dBFS = 20 × log10(RMS / Max_Integer)  [dBFS]
```

### 8.2 RMS vs Loudness
| Aspect | RMS | Integrated Loudness |
|--------|-----|-------------------|
| Scope | Entire signal | Entire signal, K-weighted |
| Weighting | Flat | K-weighting filter |
| Gating | None | Gating at –70 LUFS, –10 LU |
| Perceptual accuracy | Lower | Higher (matches human hearing) |
| BS.1770 compliant | No | Yes |

### 8.3 FFmpeg RMS Normalization
```bash
# Calculate RMS manually:
# RMS_dBFS = mean_volume from volumedetect
# To normalize RMS to target X dBFS:
GAIN=$(echo "TARGET - MEAN" | bc)
# e.g., TARGET = -20 dBFS, MEAN = -18.5 dBFS
GAIN=-1.5
ffmpeg -i input.wav -af "volume=${GAIN}dB" output.wav
```

---

## 9. REPLAYGAIN

### 9.1 Overview
ReplayGain is a volunteer-developed standard for album/track loudness normalization, predating EBU R128. It stores gain and peak values as metadata tags.

### 9.2 ReplayGain Tags
| Tag | Format | Description |
|-----|--------|-------------|
| REPLAYGAIN_TRACK_GAIN | "+X.XX dB" or "-X.XX dB" | Gain to apply to match –18 LUFS |
| REPLAYGAIN_TRACK_PEAK | "0.XXXXXXX" | Peak value (0.0 to 1.0) |
| REPLAYGAIN_ALBUM_GAIN | "+X.XX dB" or "-X.XX dB" | Album-level gain |
| REPLAYGAIN_ALBUM_PEAK | "0.XXXXXXX" | Album peak |
| REPLAYGAIN_REFERENCE_LOUDNESS | "18 dB" | Standard reference (18 LUFS) |

### 9.3 FFmpeg ReplayGain Scanning
```bash
# Scan album for ReplayGain:
ffmpeg -i *.flac -af "areplaygain=sort=album" -f null -

# Scan individual tracks:
ffmpeg -i track.flac -af "replaygain" -f null -
```

### 9.4 ReplayGain Limitations
- **Pre-scan required:** Must measure each file before applying gain
- **Not K-weighted:** Less perceptually accurate than EBU R128
- **Inconsistent players:** Support varies across media players
- **Single gain value:** Cannot match modern streaming loudness targets

### 9.5 Converting ReplayGain to EBU R128
ReplayGain targets –18 LUFS; streaming targets vary:

| Platform Target | Adjustment from ReplayGain |
|-----------------|---------------------------|
| Spotify (–14 LUFS) | Add +4 dB to ReplayGain track gain |
| Apple Music (–16 LUFS) | Add +2 dB to ReplayGain track gain |
| EBU R128 (–23 LUFS) | Add +5 dB to ReplayGain track gain |

---

## 10. DYNAMIC RANGE COMPRESSION IN LOSSY CODECS

### 10.1 How Codecs Affect Dynamics
Lossy codecs can reduce dynamic range through:
1. **Bit starvation:** Insufficient bits allocated to low-level signals
2. **Pre-echo artifacts:** Quantization noise before transients
3. **Stereo image collapse:** Joint stereo coding reduces spatial dynamics
4. **High-frequency rolloff:** Bandwidth limitation affects attack transients

### 10.2 Dynamic Range Preservation by Codec
| Codec | Bitrate | Dynamic Range Preservation | Notes |
|-------|---------|---------------------------|-------|
| MP3 | 128 kbps | 80% | Some HF loss |
| MP3 | 320 kbps | 95% | Near-transparent |
| AAC | 256 kbps | 95% | Excellent |
| Opus | 128 kbps | 90% | CELT mode handles dynamics well |
| Vorbis | 192 kbps | 90% | Good at low bitrates |
| FLAC | lossless | 100% | No loss |

### 10.3 Normalization vs Codec Artifacts
Even with normalization, codec artifacts can affect perceived dynamics:
- **Heavy limiting:** Streaming platforms may apply limiting to prevent peaks
- **Transcoding:** Multiple encode/decode cycles increase artifact accumulation
- **Dynamic processing:** Some platforms apply additional processing for consistency

---

## 11. PEAK LIMITING

### 11.1 What Is a Peak Limiter?
A **peak limiter** prevents signal peaks from exceeding a specified level:

```
Input → Gain Controller → Output
           ↑
         Control Signal
           ↑
    Look-ahead Buffer → Peak Detector

Limiter behavior:
  if |input[n]| > threshold:
    gain = threshold / |input[n]|
  else:
    gain = 1.0
```

### 11.2 Brick-Wall Limiters
Professional mastering limiters have:
- **Look-ahead:** 1–5 ms of delay to "see" peaks before they occur
- **Fast attack:** < 0.1 ms attack time
- **Gradual release:** Smooth gain recovery to avoid distortion
- **Ceiling:** Set at –0.1 to –0.3 dBTP

### 11.3 FFmpeg Loudnorm as Limiter
```bash
# Apply limiting (brick-wall effect):
ffmpeg -i input.wav \
  -af "loudnorm=I=-14:TP=-1:LRA=0:linear=true" \
  output.wav

# LRA=0 forces near-constant loudness (limiting, not normalization)
```

---

## 12. DBFS VS DBVU

### 12.1 dBFS (Digital Full Scale)
- **Reference:** Digital full scale (maximum representable sample value)
- **0 dBFS:** Maximum peak before clipping
- **Negative values:** Signal below maximum
- **Formula:** `dBFS = 20 × log10(Peak / 2^(bit_depth-1))`

### 12.2 dBVU (Digital Volume Unit)
- **Reference:** Studio reference level (typically –18 dBFS or –20 dBFS)
- **0 VU:** Reference level, not clipping
- **Positive values:** Above reference level
- **History:** Analog VU meters calibrated to 0 VU = +4 dBu

### 12.3 dBFS vs dBVU Conversion
| dBFS | dBVU (at –18 dBFS = 0 VU) |
|------|----------------------------|
| 0 dBFS | +18 dBVU |
| –1 dBFS | +17 dBVU |
| –6 dBFS | +12 dBVU |
| –18 dBFS | 0 dBVU |
| –24 dBFS | –6 dBVU |

### 12.4 Headroom Considerations
| Application | Recommended Headroom | Peak Ceiling |
|-------------|---------------------|--------------|
| Broadcast (EBU R128) | — | –1 dBTP |
| CD Mastering | 1–2 dB headroom | –0.1 dBFS |
| Streaming master | 3–6 dB headroom | –3 to –6 dBFS |
| Live sound | 6–12 dB headroom | Variable |
| Vinyl mastering | Special considerations | Lower (groove limits) |

---

## 13. IMPLEMENTATION GUIDE

### 13.1 FFmpeg Complete Workflow
```bash
#!/bin/bash
# normalize_for_streaming.sh

INPUT="$1"
TARGET_I="${2:--14}"  # Default: Spotify/YouTube target
TARGET_TP="${3:--1}"  # Default: –1 dBTP
TARGET_LRA="${4:-11}" # Default: moderate dynamic range

# Pass 1: Measure
MEASURE=$(ffmpeg -i "$INPUT" -af \
  "loudnorm=I=${TARGET_I}:TP=${TARGET_TP}:LRA=${TARGET_LRA}:print_format=json" \
  -f null - 2>&1)

# Extract values (using grep and sed)
I_VAL=$(echo "$MEASURE" | grep -oP '"input_i":\s*"\K[^"]+' | head -1)
TP_VAL=$(echo "$MEASURE" | grep -oP '"input_tp":\s*"\K[^"]+' | head -1)
LRA_VAL=$(echo "$MEASURE" | grep -oP '"input_lra":\s*"\K[^"]+' | head -1)
THRESH=$(echo "$MEASURE" | grep -oP '"input_thresh":\s*"\K[^"]+' | head -1)

# Pass 2: Normalize
OUTPUT="${INPUT%.wav}_normalized.wav"
ffmpeg -i "$INPUT" -af \
  "loudnorm=I=${TARGET_I}:TP=${TARGET_TP}:LRA=${TARGET_LRA}:\
   measured_I=${I_VAL}:measured_TP=${TP_VAL}:\
   measured_LRA=${LRA_VAL}:measured_thresh=${THRESH}:offset=0:linear=true" \
  -ar 48000 \
  "$OUTPUT"

echo "Normalized to ${TARGET_I} LUFS. Output: $OUTPUT"
```

### 13.2 Album Normalization Workflow
```bash
# Measure album loudness
ALBUM_I=$(ffmpeg -i "album/"*.flac -filter_complex \
  "concat=n=50:v=0:a=1, loudnorm=I=${TARGET_I}:TP=${TARGET_TP}:print_format=json" \
  -f null - 2>&1 | grep -oP '"input_i":\s*"\K[^"]+')

# Calculate album gain
ALBUM_GAIN=$(echo "${TARGET_I} - ${ALBUM_I}" | bc)

# Apply to all tracks
for track in album/*.flac; do
  ffmpeg -i "$track" -af "volume=${ALBUM_GAIN}dB" "normalized/$(basename $track)"
done
```

### 13.3 Verification Commands
```bash
# Verify output loudness
ffmpeg -i output.wav -af loudnorm=I=-14:TP=-1:print_format=summary -f null -

# Compare input vs output loudness
echo "Input:" && ffmpeg -i input.wav -af volumedetect -f null - 2>&1 | grep max_volume
echo "Output:" && ffmpeg -i output.wav -af volumedetect -f null - 2>&1 | grep max_volume
```

---

## 14. LOUDNESS METERS COMPARISON

### 14.1 FFmpeg Built-in Tools
| Tool | Coverage | Gating | Notes |
|------|----------|--------|-------|
| `volumedetect` | Peak only | None | Simple peak measurement |
| `loudnorm` | Full EBU R128 | Yes | Most accurate |

### 14.2 Third-Party Tools
| Tool | License | EBU R128 | Notes |
|------|---------|----------|-------|
| `ebur128` (libebur128) | BSD | Full | Reference implementation |
| `ffmpeg-normalize` | GPL | Full | Python wrapper |
| Youlean Loudness Meter | Free/Paid | Full | Popular, accurate |
| Waves WLM | Commercial | Full | Professional |
| iZotope Insight | Commercial | Full | Comprehensive |
| Nugen Audio VisLM | Commercial | Full | Broadcast standard |

### 14.3 libebur128 Integration
```c
// Basic EBU R128 measurement with libebur128:
#include <ebur128.h>

ebur128_state *ebur = ebur128_init(
    2,          // channels
    44100,      // sample rate
    EBUR128_MODE_LUFS // measure LUFS
);

ebur128_add_frames_float(ebur, buffer, frame_count);
ebur128_loudness_global(ebur, &loudness);

double tp;
ebur128_true_peak(ebur, 0, &tp); // Channel 0

ebur128_destroy(&ebur);
```

---

## 15. STREAMING PLATFORM INTEGRATION

### 15.1 Platform-Specific Master Preparation
| Platform | Target LUFS | Max True Peak | Key Notes |
|----------|-------------|---------------|----------|
| Spotify | –14 | –1 dBTP | Check "loud" normalization setting |
| YouTube | –14 | –1 dBTP | Normalizes on ingest; no boost |
| Apple Music | –16 | –1 dBTP | iTunes Sound Check |
| Amazon Music | –11 to –13 | –1 dBTP | Genre-dependent targets |
| Tidal | –14 | –1 dBTP | Masters: FLAC up to 24/192 |
| Qobuz | –18 | –1 dBTP | Hi-Fi: no compression preferred |
| Netflix | –24 to –27 | –2 dBTP | Programme-specific |

### 15.2 Multi-Platform Master Strategy
**Recommended approach:** Master for –14 LUFS, –1 dBTP
- Covers: Spotify, YouTube, Tidal (primary)
- Apple Music will apply –2 dB gain (acceptable)
- Qobuz/Amazon will be slightly louder (acceptable)

### 15.3 Mastering Checklist
- [ ] Analyze true peak of master
- [ ] Ensure no peaks exceed –1 dBTP (ideally –1.2 dBTP for safety)
- [ ] Verify integrated loudness matches target
- [ ] Check LRA for genre-appropriate dynamics
- [ ] Preview on reference monitors at calibrated level
- [ ] Check with Youlean or equivalent for BS.1770 compliance

---

## 16. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specifications
- **EBU R128:** https://tech.ebu.ch/docs/r/r128.pdf
- **EBU Tech 3341:** EBU Mode Loudness Metering
- **EBU Tech 3342:** Loudness Range
- **EBU Tech 3401:** Radio Guidelines
- **ITU-R BS.1770-4:** https://www.itu.int/rec/R-REC-BS.1770-4
- **ITU-R BS.1771-2:** Peak programme loudness meters

### Technical Resources
- EBU Tech Portal: https://tech.ebu.ch/
- Youlean Loudness Meter: https://youlean.co/loudness-meter/
- Audio Mastering Blog: https://www.travsonic.com/mastering-audio-streaming/

---

## 17. IMPLEMENTATION CHECKLIST (Audio Converter Developer)

### Loudness Measurement
- [ ] Implement EBU R128 compliant loudness calculation or use libebur128
- [ ] Implement K-weighting filter (high-pass + shelf)
- [ ] Implement absolute and relative gating
- [ ] Calculate integrated loudness (LUFS)
- [ ] Calculate true peak (dBTP)
- [ ] Calculate loudness range (LRA)

### Normalization
- [ ] Implement two-pass loudness normalization
- [ ] Support single-pass for real-time use
- [ ] Handle true peak limiting (brick-wall if needed)
- [ ] Preserve album-level dynamics where appropriate

### Streaming Integration
- [ ] Support configurable target LUFS
- [ ] Support configurable true peak limit
- [ ] Provide presets for major streaming platforms
- [ ] Verify output loudness after normalization

### Quality Assurance
- [ ] Test with various dynamic range materials
- [ ] Verify no clipping after normalization
- [ ] Check true peak compliance
- [ ] Compare against reference loudness meters

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*

---

## 16. ADVANCED LOUDNESS MEASUREMENT

### 16.1 K-Weighting Filter Design
The K-weighting filter is fundamental to BS.1770 loudness measurement:

```python
def k_weighting_filter(sample_rate):
    """
    Design K-weighting filter for loudness measurement.
    Based on ITU-R BS.1770-4.
    
    Args:
        sample_rate: Sample rate in Hz
    
    Returns:
        filter coefficients (b, a)
    """
    from scipy.signal import butter, lfilter, sosfilt
    
    # High-pass filter (first stage):
    # H(z) = (1 - z^(-1)) / (1 - 0.9816 * z^(-1))
    b_hp = [1, -1]
    a_hp = [1, -0.9816]
    
    # High-shelf filter (second stage):
    # +4 dB boost at 1681.97 Hz
    # This is a simplified approximation
    
    return b_hp, a_hp

def apply_k_weighting(audio, sample_rate):
    """
    Apply K-weighting to audio signal.
    """
    from scipy.signal import lfilter
    
    # High-pass filter coefficients
    b = [1.0, -1.0]
    a = [1.0, -0.9816]
    
    # Apply filter
    filtered = lfilter(b, a, audio)
    
    # Apply pre-filter (simplified):
    # In practice, use proper biquad design
    # for the +4 dB shelf at ~1.7 kHz
    
    return filtered

def k_weighting_sos(sample_rate):
    """
    Generate K-weighting filter as second-order sections.
    
    Based on ITU-R BS.1770-4 Annex 2.
    """
    from scipy.signal import butter, sosfilt
    
    # Stage 1: High-shelf (+4 dB at 1681.97 Hz)
    # Approximated as biquad
    f0 = 1681.97
    Q = 0.707
    gain_db = 4.0
    
    # Convert to digital filter coefficients
    # (simplified - actual implementation more complex)
    
    # Stage 2: High-pass (38.13547 Hz, Q = 0.5002)
    # Stage 3: High-pass (38.13547 Hz, Q = 0.5002) [second section]
    # Stage 4: High-pass (149.097 Hz, Q = 0.5005)
    
    # These form the complete K-weighting filter
    # See BS.1770-4 Annex 2 for exact coefficients
```

### 16.2 Gating Algorithm Implementation
```python
def calculate_integrated_loudness(audio, sample_rate, channel_weights=None):
    """
    Calculate integrated loudness according to ITU-R BS.1770-4.
    
    Args:
        audio: Audio samples (numpy array)
        sample_rate: Sample rate in Hz
        channel_weights: List of channel weights (default: [1.0, 1.0])
    
    Returns:
        Integrated loudness in LUFS
    """
    from scipy.signal import sosfilt
    
    # Default stereo weights
    if channel_weights is None:
        n_channels = audio.shape[1] if len(audio.shape) > 1 else 1
        channel_weights = [1.0] * n_channels
    
    # Step 1: Apply K-weighting
    weighted = apply_k_weighting(audio, sample_rate)
    
    # Step 2: Block into 400ms windows with 75% overlap
    block_size = int(0.4 * sample_rate)  # 400ms
    hop = int(0.1 * sample_rate)  # 75% overlap
    
    # Step 3: Calculate mean square for each block
    block_loudness = []
    for start in range(0, len(weighted) - block_size, hop):
        block = weighted[start:start + block_size]
        
        # Apply channel weights and sum
        if len(block.shape) > 1:
            mean_sq = sum(
                channel_weights[i] * np.mean(block[:, i]**2)
                for i in range(block.shape[1])
            )
        else:
            mean_sq = channel_weights[0] * np.mean(block**2)
        
        # Convert to loudness (simplified)
        if mean_sq > 0:
            loudness = -0.691 + 10 * np.log10(mean_sq)
            block_loudness.append(loudness)
    
    block_loudness = np.array(block_loudness)
    
    # Step 4: Absolute gating (exclude blocks < -70 LUFS)
    absolute_threshold = -70.0
    gated_blocks = block_loudness[block_loudness > absolute_threshold]
    
    if len(gated_blocks) == 0:
        return -70.0  # Silence
    
    # Step 5: Relative gating
    # Calculate ungated loudness
    ungated_loudness = 10 * np.log10(
        np.mean(10**(gated_blocks / 10))
    )
    
    relative_threshold = ungated_loudness - 10.0
    final_gated_blocks = gated_blocks[gated_blocks > relative_threshold]
    
    if len(final_gated_blocks) == 0:
        return ungated_loudness  # No blocks pass relative gate
    
    # Step 6: Final integrated loudness
    integrated = 10 * np.log10(
        np.mean(10**(final_gated_blocks / 10))
    )
    
    return integrated
```

### 16.3 True Peak Detection Algorithm
```python
def calculate_true_peak(audio, sample_rate, oversample_factor=4):
    """
    Calculate true peak of audio signal.
    
    Algorithm:
    1. Oversample the signal
    2. Apply K-weighting
    3. Find maximum absolute value
    4. Convert to dBTP
    
    Args:
        audio: Audio samples
        sample_rate: Original sample rate
        oversample_factor: Oversampling factor (4x or higher)
    
    Returns:
        True peak in dBTP
    """
    from scipy.signal import resample_poly
    
    # Oversample
    oversampled = resample_poly(
        audio,
        oversample_factor,
        1,
        axis=0
    )
    
    # Apply K-weighting at oversampled rate
    new_rate = sample_rate * oversample_factor
    k_weighted = apply_k_weighting(oversampled, new_rate)
    
    # Find maximum absolute value
    true_peak = np.max(np.abs(k_weighted))
    
    # Convert to dBTP
    if true_peak > 0:
        true_peak_db = 20 * np.log10(true_peak)
    else:
        true_peak_db = -np.inf
    
    return true_peak_db

def true_peak_limit(audio, target_db=-1.0):
    """
    Apply true peak limiting to audio.
    
    Args:
        audio: Input audio
        target_db: Target true peak in dBTP
    
    Returns:
        Limited audio
    """
    # Calculate current true peak
    current_tp = calculate_true_peak(audio, 48000)
    
    # Calculate required gain
    gain_db = target_db - current_tp
    
    # Apply gain
    if gain_db < 0:
        linear_gain = 10**(gain_db / 20)
        audio_limited = audio * linear_gain
    
    # Re-check true peak
    # May need multiple iterations for brick-wall limiting
    
    return audio_limited
```

---

## 17. LOUDNESS RANGE (LRA) CALCULATION

### 17.1 LRA Algorithm
```python
def calculate_loudness_range(audio, sample_rate):
    """
    Calculate Loudness Range (LRA) per EBU Tech 3342.
    
    Args:
        audio: Audio samples
        sample_rate: Sample rate
    
    Returns:
        LRA in LU
    """
    # Step 1: Calculate momentary loudness for each block
    # Same as integrated loudness but without gating
    block_size = int(0.4 * sample_rate)
    hop = int(0.4 * sample_rate)  # No overlap for LRA
    
    momentary_loudness = []
    for start in range(0, len(audio) - block_size, hop):
        block = audio[start:start + block_size]
        mean_sq = np.mean(block**2)
        
        if mean_sq > 0:
            loudness = -0.691 + 10 * np.log10(mean_sq)
            momentary_loudness.append(loudness)
    
    momentary = np.array(momentary_loudness)
    
    # Step 2: Apply gating (absolute: -70 LUFS, relative: -10 LU)
    absolute_threshold = -70.0
    relative_threshold = np.mean(momentary[momentary > absolute_threshold]) - 10.0
    
    gated = momentary[(momentary > absolute_threshold) & 
                      (momentary > relative_threshold)]
    
    if len(gated) == 0:
        return 0.0
    
    # Step 3: Calculate histogram
    # Divide range into bins
    n_bins = 100
    hist, bin_edges = np.histogram(gated, bins=n_bins)
    
    # Step 4: Find 5th and 95th percentile
    cumsum = np.cumsum(hist) / len(gated)
    
    # Find L5 (5th percentile)
    L5_idx = np.searchsorted(cumsum, 0.05)
    L5 = bin_edges[L5_idx]
    
    # Find L95 (95th percentile)
    L95_idx = np.searchsorted(cumsum, 0.95)
    L95 = bin_edges[L95_idx]
    
    # LRA = L95 - L5
    LRA = L95 - L5
    
    return LRA
```

### 17.2 LRA Interpretation
| LRA Range | Interpretation | Programme Type |
|-----------|---------------|----------------|
| 0–4 LU | Minimal dynamics | Commercials, heavily compressed pop |
| 4–8 LU | Moderate dynamics | Pop/rock with some dynamics |
| 8–15 LU | Natural dynamics | Acoustic music, drama |
| 15–25 LU | Wide dynamics | Classical, jazz, orchestral |
| >25 LU | Very wide | Uncompressed, live recordings |

---

## 18. STREAMING PLATFORM DETAILED ANALYSIS

### 18.1 Spotify Loudness Normalization
```python
SPOTIFY_SPECS = {
    'target_lufs': -14.0,
    'true_peak_limit': -1.0,
    'normalization_type': 'both',  # Can normalize up or down
    'user_selectable': [-19, -14, -11],  # User can choose
    'analysis_method': 'ITU-R BS.1770',
    'gain_range': 'unlimited',  # Can boost or attenuate
}
```

### 18.2 YouTube Audio Processing
```python
YOUTUBE_SPECS = {
    'target_lufs': -14.0,
    'true_peak_limit': -1.0,
    'normalization_type': 'downward_only',  # Only attenuate
    'analysis_method': 'Internal',
    'processing_chain': [
        'Input audio analysis',
        'Loudness measurement (EBU R128)',
        'True peak detection',
        'Gain calculation',
        'Brick-wall limiting',
        'Output'
    ],
}
```

### 18.3 Apple Music Audio Processing
```python
APPLE_MUSIC_SPECS = {
    'target_lufs': -16.0,
    'true_peak_limit': -1.0,
    'normalization_type': 'both',
    'analysis_method': 'Internal (Sound Check successor)',
    'format_support': 'AAC-LC, ALAC, Dolby Atmos',
    'spatial_audio': True,  # Supports Dolby Atmos spatial
}
```

### 18.4 Multi-Platform Master Checklist
```python
def streaming_master_checklist():
    """
    Checklist for mastering for multiple streaming platforms.
    """
    return [
        # 1. Loudness Analysis
        {
            'check': 'Integrated loudness',
            'target': '-14 LUFS',
            'tolerance': '±0.5 LU',
        },
        {
            'check': 'True peak',
            'target': '-1.0 dBTP',
            'tolerance': '±0.5 dBTP',
        },
        {
            'check': 'Loudness range (LRA)',
            'target': 'Genre-appropriate',
            'typical': '5-15 LU',
        },
        
        # 2. Spectral Balance
        {
            'check': 'Low frequency content',
            'note': 'Check bass in headphones and small speakers',
        },
        {
            'check': 'High frequency clarity',
            'note': 'Avoid excessive sibilance',
        },
        
        # 3. Dynamics
        {
            'check': 'Dynamic range',
            'note': 'Preserve natural dynamics where appropriate',
        },
        {
            'check': 'Peak control',
            'note': 'Use brick-wall limiter for headroom',
        },
        
        # 4. Monitoring
        {
            'check': 'Reference monitors',
            'level': '85 dB SPL (calibrated)',
        },
        {
            'check': 'Headphones',
            'level': 'Various for checking',
        },
        {
            'check': 'Consumer playback',
            'devices': 'Phone, earbuds, laptop speakers',
        },
    ]
```

---

## 19. LOUDNESS METERS — IMPLEMENTATION

### 19.1 EBU R128 Meter Class
```python
class EBUR128Meter:
    """
    EBU R128 compliant loudness meter.
    """
    
    def __init__(self, sample_rate=48000):
        self.sample_rate = sample_rate
        self.block_size = int(0.4 * sample_rate)  # 400ms
        self.short_term_window = int(3.0 * sample_rate)  # 3s
        self.loudness_history = []
        self.momentary_history = []
        
    def update(self, audio_block):
        """
        Update meter with new audio block.
        
        Args:
            audio_block: Audio samples
        """
        # Calculate momentary loudness
        momentary = self._calculate_momentary(audio_block)
        self.momentary_history.append(momentary)
        
        # Calculate short-term loudness (last 3 seconds)
        short_term = self._calculate_short_term()
        
        # Update loudness history for integrated
        self.loudness_history.append(momentary)
    
    def _calculate_momentary(self, audio):
        """Calculate momentary loudness (400ms block)."""
        # Apply K-weighting
        weighted = apply_k_weighting(audio, self.sample_rate)
        
        # Calculate mean square
        mean_sq = np.mean(weighted**2)
        
        if mean_sq > 0:
            return -0.691 + 10 * np.log10(mean_sq)
        else:
            return -70.0  # Silence
    
    def _calculate_short_term(self):
        """Calculate short-term loudness (3s window)."""
        # Take last 3 seconds of momentary values
        n_blocks = int(3.0 / 0.4)  # 7.5 blocks = 3 seconds
        recent = self.momentary_history[-n_blocks:]
        
        if len(recent) == 0:
            return -70.0
        
        # Mean of powers
        mean_power = np.mean(10**(np.array(recent) / 10))
        
        if mean_power > 0:
            return 10 * np.log10(mean_power)
        else:
            return -70.0
    
    def get_integrated(self):
        """Calculate integrated loudness with gating."""
        # Apply absolute gating
        absolute_threshold = -70.0
        gated = [l for l in self.loudness_history if l > absolute_threshold]
        
        if len(gated) == 0:
            return -70.0
        
        # Calculate ungated loudness
        ungated = 10 * np.log10(np.mean(10**(np.array(gated) / 10)))
        
        # Apply relative gating
        relative_threshold = ungated - 10.0
        final_gated = [l for l in gated if l > relative_threshold]
        
        if len(final_gated) == 0:
            return ungated
        
        # Final integrated loudness
        return 10 * np.log10(np.mean(10**(np.array(final_gated) / 10)))
```

### 19.2 True Peak Meter
```python
class TruePeakMeter:
    """
    True peak meter per ITU-R BS.1770-4.
    """
    
    def __init__(self, sample_rate=48000, oversample_factor=4):
        self.sample_rate = sample_rate
        self.oversample_factor = oversample_factor
        self.peak_history = []
    
    def update(self, audio):
        """Update with new audio samples."""
        # Calculate true peak
        tp = calculate_true_peak(audio, self.sample_rate, 
                                self.oversample_factor)
        self.peak_history.append(tp)
    
    def get_max_peak(self):
        """Get maximum true peak."""
        return np.max(self.peak_history)
    
    def get_peak_at_time(self, time_seconds):
        """Get true peak at specific time."""
        # Calculate index from time
        idx = int(time_seconds * (self.sample_rate / self.block_size))
        
        if idx < len(self.peak_history):
            return self.peak_history[idx]
        else:
            return None
```

---

## 20. DYNAMIC RANGE COMPRESSION

### 20.1 Dynamic Range Compression Theory
```python
class DynamicRangeCompressor:
    """
    Dynamic range compressor for audio processing.
    """
    
    def __init__(self, sample_rate=48000):
        self.sample_rate = sample_rate
        self.threshold = -20.0  # dBFS
        self.ratio = 4.0  # 4:1
        self.attack = 0.01  # seconds
        self.release = 0.1  # seconds
        self.makeup_gain = 0.0  # dB
        
        # Calculate time constants
        self.attack_coeff = np.exp(-1.0 / (self.attack * sample_rate))
        self.release_coeff = np.exp(-1.0 / (self.release * sample_rate))
        
        self.envelope = 0.0
    
    def process(self, audio):
        """
        Process audio with compression.
        
        Args:
            audio: Input samples
        
        Returns:
            Compressed audio
        """
        output = np.zeros_like(audio)
        
        for i, sample in enumerate(audio):
            # Get absolute value
            abs_input = abs(sample)
            
            # Convert to dB
            if abs_input > 0:
                input_db = 20 * np.log10(abs_input)
            else:
                input_db = -100.0
            
            # Envelope follower
            if input_db > self.envelope:
                self.envelope = (self.attack_coeff * self.envelope + 
                               (1 - self.attack_coeff) * input_db)
            else:
                self.envelope = (self.release_coeff * self.envelope + 
                               (1 - self.release_coeff) * input_db)
            
            # Calculate gain reduction
            if self.envelope > self.threshold:
                # Above threshold: apply ratio
                excess = self.envelope - self.threshold
                reduction = excess * (1 - 1.0/self.ratio)
            else:
                reduction = 0.0
            
            # Total gain
            gain_db = -reduction + self.makeup_gain
            gain_linear = 10**(gain_db / 20)
            
            output[i] = sample * gain_linear
        
        return output
```

### 20.2 Limiting vs Compression
| Aspect | Compressor | Brick-Wall Limiter |
|--------|-----------|-------------------|
| Ratio | 2:1 to 10:1 | ∞:1 |
| Attack | Variable (ms) | Instantaneous (<1ms) |
| Release | Variable | Fast (ms) |
| Output | Limited headroom | Strict ceiling |
| Sound | Tonal character | Clean, transparent |

---

## 21. LOUDNESS VS DYNAMICS IN PRODUCTION

### 21.1 Production Guidelines
```python
PRODUCTION_GUIDELINES = {
    'broadcast': {
        'target': -23.0,
        'tp_limit': -1.0,
        'lra_range': '8-15 LU typical',
        'peak_type': 'True peak',
    },
    'streaming': {
        'spotify': -14.0,
        'youtube': -14.0,
        'apple': -16.0,
        'tp_limit': -1.0,
    },
    'cinema': {
        'target': -27.0,  # Dialogue
        'tp_limit': -2.0,
        'lra_range': 'Wide (film)',
    },
    'classical': {
        'target': 'No specific',
        'note': 'Preserve dynamics',
        'typical_lra': '15-25 LU',
    },
}
```

### 21.2 Genre-Specific Dynamics
| Genre | Typical LRA | Target LUFS | Notes |
|-------|-------------|-----------|-------|
| EDM/Techno | 3-6 LU | -8 to -12 | Loud, competitive |
| Pop | 5-10 LU | -10 to -14 | Balanced |
| Rock | 8-15 LU | -12 to -16 | Dynamic |
| Jazz | 12-20 LU | -14 to -18 | Natural |
| Classical | 15-30 LU | -18 to -23 | Full dynamics |
| Podcast | 5-10 LU | -14 to -16 | Consistent |

---

*File expanded with: K-weighting filter design, gating algorithm, true peak calculation, LRA computation, streaming platform details, compressor implementation, and production guidelines*

---

## 22. MEASUREMENT SOFTWARE COMPARISON

### 22.1 Open-Source Tools
| Tool | License | EBU R128 | True Peak | Notes |
|------|---------|-----------|-----------|-------|
| ffmpeg loudnorm | LGPL | Full | Yes | CLI tool |
| ebur128 | BSD | Full | Yes | Reference implementation |
| libloudness | BSD | Full | Yes | Library |
| ffmpeg-normalize | GPL | Full | Yes | Python wrapper |
| sox | GPL | Partial | Yes | Audio processing |

### 22.2 Commercial Tools
| Tool | Vendor | EBU R128 | True Peak | Notes |
|------|--------|-----------|-----------|-------|
| Youlean LM | Youlean | Full | Yes | Free/Pro |
| WLM | Waves | Full | Yes | Professional |
| VisLM | Nugen | Full | Yes | Broadcast standard |
| Insight | iZotope | Full | Yes | Comprehensive |
| Dorrough | Dorrough | Partial | Yes | Industry standard |

### 22.3 Measurement Accuracy
| Tool | Accuracy | Gating Support | Notes |
|------|---------|---------------|-------|
| Reference (PPM) | ±0.1 dB | No | Analog standard |
| BS.1770 meter | ±0.5 dB | Full | Digital standard |
| Integrated meter | ±0.3 dB | Full | EBU R128 |

---

## 23. LOUDNESS IN BROADCAST STANDARDS

### 23.1 ATSC A/85 (US Television)
```python
ATSC_A85 = {
    'target': -24.0,  # LUFS ( ATSC recommended)
    'range': -26.0 to -22.0,  # Acceptable range
    'tp_limit': -2.0,  # dBTP
    'dialogue_gate': True,  # Use dialogue-gated measurement
}
```

### 23.2 EBU R128s1 (Short-form)
```python
EBU_R128_S1 = {
    'target': -23.0,  # LUFS
    'tp_limit': -1.0,  # dBTP
    'short_form': True,  # For ads, promos
    'max_duration': 60.0,  # seconds
}
```

### 23.3 EBU R128s3 (Radio)
```python
EBU_R128_S3 = {
    'target': -23.0,  # LUFS
    'tp_limit': -1.0,  # dBTP
    'lra_limit': 18.0,  # Maximum LRA in LU
}
```

---

## 24. LOUDNESS IN FILM & VIDEO

### 24.1 Film Mixing Levels
| Stage | Target | Notes |
|-------|--------|-------|
| Mixing stage | -27 to -24 LUFS | Dialogue-centric |
| Review room | -23 to -21 LUFS | Match main stage |
| Theater | -85 dB SPL avg | Room-dependent |
| Home video | -24 LUFS | Streaming standard |

### 24.2 Streaming Video Platforms
| Platform | Audio Target | Notes |
|----------|-------------|-------|
| Netflix | -27 LUFS (film), -24 LUFS (TV) | Programme-specific |
| Amazon | -24 LUFS | Standard |
| Disney+ | -24 LUFS | Standard |
| Apple TV+ | -16 LUFS (music), -24 LUFS (dialogue) | Mixed |
| HBO Max | -24 LUFS | Standard |

---

## 25. PRACTICAL LOUDNESS WORKFLOW

### 25.1 Recording Workflow
```python
def recording_workflow():
    """
    Recommended loudness workflow for recording.
    """
    return [
        "1. Set recording level for headroom (peak at -12 to -6 dBFS)",
        "2. Record at 24-bit (noise floor well below hearing)",
        "3. Monitor levels but don't over-compress during recording",
        "4. Import to DAW at recorded level",
        "5. Mix without loudness target in mind",
        "6. Apply mastering for target loudness",
    ]

def mixing_workflow():
    """
    Recommended workflow for mixing with loudness in mind.
    """
    return [
        "1. Mix at a consistent, comfortable monitoring level (75-85 dB SPL)",
        "2. Reference against commercial releases",
        "3. Don't mix 'quiet' - mix at target level",
        "4. Use compression sparingly to control dynamics",
        "5. Avoid over-limiting just for loudness",
        "6. Preserve musical dynamics where appropriate",
    ]

def mastering_workflow():
    """
    Recommended workflow for mastering.
    """
    return [
        "1. Analyze with EBU R128 meter",
        "2. Target streaming platform LUFS",
        "3. Apply gentle compression if needed",
        "4. Use brick-wall limiter for headroom",
        "5. Verify true peak compliance",
        "6. Check LRA for genre-appropriate dynamics",
        "7. Create multiple masters for different platforms",
    ]
```

### 25.2 Multi-Format Mastering
```python
def create_streaming_masters():
    """
    Create masters for different streaming platforms.
    """
    return {
        'spotify': {
            'target': -14.0,
            'tp_limit': -1.0,
        },
        'apple_music': {
            'target': -16.0,
            'tp_limit': -1.0,
        },
        'youtube': {
            'target': -14.0,
            'tp_limit': -1.0,
        },
        'broadcast': {
            'target': -23.0,
            'tp_limit': -1.0,
        },
    }
```

---

## 26. LOUDNESS DEBUGGING

### 26.1 Common Issues
| Issue | Cause | Solution |
|-------|-------|----------|
| Too quiet after normalization | Input below target | Apply gain before loudnorm |
| Clipping after normalization | Target too high | Lower target or use limiter |
| Inconsistent loudness | Mixed sources | Apply consistent normalization |
| LRA too low | Over-compressed | Use gentler mastering |
| True peak exceeded | Insufficient headroom | Apply limiting |

### 26.2 Troubleshooting Guide
```python
def troubleshoot_loudness_issues():
    """
    Common loudness issues and solutions.
    """
    return {
        'issue_1': {
            'description': 'Output louder than target',
            'causes': ['Input already above target', 'Target misconfigured'],
            'solution': 'Apply attenuation before normalization',
        },
        'issue_2': {
            'description': 'True peak exceeded',
            'causes': ['Input peaks too high', 'Gain applied too aggressively'],
            'solution': 'Use brick-wall limiter, lower target',
        },
        'issue_3': {
            'description': 'Inconsistent loudness between tracks',
            'causes': ['Different mastering levels', 'Varying source quality'],
            'solution': 'Normalize each track individually',
        },
        'issue_4': {
            'description': 'Loss of dynamics',
            'causes': ['Heavy compression', 'Aggressive limiting'],
            'solution': 'Use lighter processing, accept louder master',
        },
    }
```

---

## 27. LOUDNESS TEST SIGNALS

### 27.1 Reference Signals
```python
def generate_test_signals():
    """
    Generate standard loudness test signals.
    """
    return {
        'silence': {
            'duration': 10.0,  # seconds
            'loudness': -np.inf,  # LUFS
        },
        'pink_noise': {
            'duration': 10.0,
            'loudness': -20.0,  # Target LUFS
            'description': 'Reference pink noise at -20 LUFS',
        },
        '1kHz_sine': {
            'duration': 10.0,
            'loudness': -3.0,  # Peak at -3 dBFS
            'description': '1 kHz sine wave for meter calibration',
        },
        'sweep': {
            'duration': 10.0,
            'description': '20 Hz to 20 kHz logarithmic sweep',
        },
    }
```

### 27.2 EBU Test CD Tracks
```python
EBU_TEST_CD = {
    'track_1': {
        'name': '500 Hz sine wave',
        'frequency': 500,
        'duration': 60.0,
        'level': -23 LUFS,
    },
    'track_2': {
        'name': 'Pink noise',
        'duration': 60.0,
        'level': -23 LUFS,
    },
    'track_3': {
        'name': 'PPM reference',
        'description': '1000 Hz sine, various levels',
    },
}
```

---

*File expanded with: Measurement tools comparison, broadcast standards, film/video loudness, practical workflows, debugging guide, and test signals*
