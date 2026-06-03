# Metadata: ReplayGain & EBU R128 — Deep Technical Reference

> **Category:** Metadata / Volume Normalization System
> **File Extensions:** N/A (metadata tags for various formats)
> **MIME Types:** N/A
> **Standardization Body:** ReplayGain.org (informal), EBU (R128)
> **Specification Document:** http://wiki.hydrogenaudio.org/index.php?title=ReplayGain_specification, EBU R128
> **Patent Status:** Patent-free
> **License:** Public domain

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 The Loudness Problem

Before ReplayGain existed, audio normalization was a crude affair. The CD era gave us track gain (pre-emphasis) stored in the CD's Q subcode channel — a simple ±1 dB adjustment applied uniformly across an entire track. This was designed for a different era when mastering engineers needed to compensate for cut lathe calibration, not for the wide variation in perceived loudness that digital audio would eventually create.

The problem that ReplayGain solved is fundamental to how digital audio is mastered and consumed. In the early 2000s, with the rise of MP3 players and CD ripping, listeners began noticing that albums from different artists, genres, or eras required significantly different volume settings on their player. A quiet classical recording might be 20 dB quieter than a heavily compressed modern rock album. Switching between them was jarring. The loudness war — the practice of maximizing perceived loudness through heavy dynamic range compression — had made this problem endemic. The solution was not to compress dynamics further but to measure and normalize loudness at playback time.

### 1.2 ReplayGain Origins

ReplayGain was created by Glen Sawyer and others at Hydrogenaudio, an audio engineering community, around 2001–2002. The specification was developed as an informal open standard, never standardized by any formal body, but it was widely adopted because it solved a real problem. The original ReplayGain 1.0 specification defined:

- A reference level of 89 dB SPL, based on psychoacoustic research from the early days of CD production
- A scanning algorithm using 50ms RMS windows with 75% overlap
- A gain adjustment value in dB, plus a peak value to prevent clipping

The 89 dB SPL reference level was chosen based on the assumption that typical commercial CD recordings averaged around that loudness, which was roughly accurate at the time but would prove to be too quiet as the loudness war escalated through the 2000s.

### 1.3 ReplayGain 2.0

ReplayGain 2.0 emerged around 2010 as a refinement addressing known limitations of v1.0. The primary changes were:

- Reference level raised from 89 dB SPL to 95 dB SPL, reflecting the reality that modern recordings were significantly louder
- Introduction of equal-loudness contour weighting in the filter chain (making measurements more perceptually accurate)
- Addition of true peak detection using oversampling (addressing inter-sample peaks that simple sample-peak measurement misses)
- New fields for minimum and maximum sample values across the track

### 1.4 EBU R128 Origins

The European Broadcasting Union (EBU) introduced R128 in 2010 as a comprehensive loudness standard for broadcast audio. Unlike ReplayGain, which was community-developed, EBU R128 was the product of formal standardization work building on ITU-R BS.1770, a recommendation from the International Telecommunication Union.

The EBU recognized that broadcast audio faced two distinct problems: excessive loudness variation between programs (viewers reaching for the remote control during commercial breaks), and the loudness war's effect on program material. R128 addressed both through a standardized measurement methodology (ITU-R BS.1770) and target loudness levels designed for broadcast chains.

The key insight behind R128 was that the traditional peak measurement (PPM, VU) did not correlate well with perceived loudness. A heavily compressed track with high peaks but low average energy sounds quieter than a dynamic track with lower peaks but higher average energy. R128 introduced integrated loudness (measured in LUFS) as the primary metric, along with true peak limiting to prevent inter-sample clipping at the output stage.

### 1.5 Adoption Landscape

ReplayGain remains the de facto standard for volume normalization in consumer music applications. Most music players (foobar2000, VLC, mpv, JRiver Media Center, Clementine, and others) implement ReplayGain scanning and playback gain application. Music management software (MusicBrainz Picard, beets) can write ReplayGain tags during the tagging pipeline.

EBU R128 is the dominant standard in broadcast (TV, radio, streaming platforms) and is increasingly used in music mastering as well. The streaming platforms — Spotify, Apple Music, YouTube, TIDAL, Amazon Music, Netflix — each target specific LUFS values that are derived from or consistent with EBU R128 methodology. Understanding R128 is essential for any mastering engineer targeting these platforms.

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 The Two-System Architecture

ReplayGain and EBU R128 both solve the same fundamental problem — consistent perceived loudness across different audio programs — but they operate at different stages of the production and consumption pipeline.

**ReplayGain** operates at the playback stage. The source audio is analyzed once (or the tags are pre-written by a scanner), and the playback application applies the gain adjustment in real time. The source file itself is not modified. This approach preserves the original audio quality and allows for dynamic adjustment (different preamp settings, album vs track mode).

**EBU R128** operates primarily at the mastering and distribution stage. The audio is analyzed using standardized measurement, and a linear gain adjustment plus a true peak limiter are applied to produce output that meets target loudness specifications. The result is baked into the distributed file. While R128 loudness metadata can also be embedded for downstream adjustment, the primary use case is production-side normalization.

Both systems share the same core measurement concept: converting audio to a loudness value in decibels relative to a reference, then computing a gain offset to reach a target reference level.

### 2.2 Core Measurement Pipeline

Both systems follow a similar measurement pipeline at a high level:

1. **Input**: Audio samples (PCM) at the native sample rate and bit depth
2. **Pre-conditioning**: Convert to a standard reference format (e.g., 32-bit float PCM at the original sample rate)
3. **Filter application**: Apply weighting filters to model human loudness perception
4. **Windowed RMS computation**: Divide into time windows, compute RMS per window
5. **Statistical aggregation**: Combine window values (mean for ReplayGain; gated mean for R128)
6. **Reference comparison**: Subtract the computed loudness from the target reference level to obtain the gain offset

The critical differences lie in the filter characteristics, the window sizes, the aggregation method (especially the gating in R128), and whether peak limiting is applied.

### 2.3 The dBFS and dB SPL Relationship

dBFS (decibels relative to Full Scale) measures digital signal level. 0 dBFS is the maximum representable sample value. dB SPL (decibels Sound Pressure Level) measures acoustic loudness in the physical world.

The correspondence between dBFS and dB SPL is calibrated: a sine wave at -18 dBFS produces a long-term RMS loudness of approximately 89 dB SPL when played back through a system calibrated to a standard listening level. This -18 dBFS = 89 dB SPL correspondence is the fundamental calibration anchor for ReplayGain 1.0.

ReplayGain 2.0 uses a reference level of 95 dB SPL, which corresponds to approximately -15 dBFS for pink noise or typical program material (the exact value depends on the crest factor of the signal).

### 2.4 LUFS vs dBFS

LUFS (Loudness Units relative to Full Scale) is the measurement unit for absolute loudness in EBU R128. It is identical in concept to dBFS when measuring the same signal, but LUFS explicitly indicates that the measurement has been weighted (K-weighted) and gated according to the ITU-R BS.1770-4 specification.

LU (Loudness Units) is a relative unit used for the gain adjustment itself. A change of 1 LU corresponds to 1 dB, but LU specifically denotes a perceptually weighted loudness change rather than an unweighted level change. In practice, 1 LU ≈ 1 dB for most purposes.

---

## 3. REPLAYGAIN SPECIFICATION

### 3.1 ReplayGain Tag Fields

ReplayGain tags are stored as text strings in the metadata container of the audio file. The format is standardized across all container types, though the underlying tag storage mechanism differs (Vorbis Comments, ID3v2 TXXX frames, MP4 freeform atoms, APEv2 tags).

#### 3.1.1 REPLAYGAIN_TRACK_GAIN

- **Purpose**: The gain adjustment (in dB) to apply to this track to normalize its loudness to the reference level
- **Format**: A signed decimal number with up to 2 decimal places, prefixed by a sign character, followed by " dB"
- **Examples**: `+3.20 dB`, `-1.50 dB`, `+0.00 dB`
- **Positive value**: Track is quieter than reference; apply positive gain to bring it up
- **Negative value**: Track is louder than reference; apply negative gain (attenuation) to bring it down
- **Calculation**: `gain = reference_level - measured_loudness` where both are in dBFS

#### 3.1.2 REPLAYGAIN_TRACK_PEAK

- **Purpose**: The maximum absolute sample value (peak) encountered in the track, used to determine if the gain-adjusted signal would clip
- **Format**: An unsigned decimal number representing the peak as a fraction of full scale, with 6 decimal places
- **Examples**: `0.987654`, `1.000000`, `0.500000`
- **Values > 1.000000**: Indicate that the true peak (with oversampling) exceeded the sample peak
- **Values of exactly 1.000000**: Peak reached exactly 0 dBFS
- **Note**: Values are stored as linear amplitude ratios, not in dB

#### 3.1.3 REPLAYGAIN_ALBUM_GAIN

- **Purpose**: The gain adjustment to apply to normalize the loudness of the entire album (all tracks collectively) to the reference level
- **Format**: Identical to REPLAYGAIN_TRACK_GAIN: signed decimal with " dB" suffix
- **Calculation**: Same as track gain, but the measurement is performed across the concatenated audio of all tracks in the album, preserving track boundaries for peak detection
- **Use case**: When playing an album in sequence, applying album gain prevents the listener from having to adjust volume between tracks

#### 3.1.4 REPLAYGAIN_ALBUM_PEAK

- **Purpose**: The maximum absolute sample value across the entire album, used for peak-aware playback
- **Format**: Identical to REPLAYGAIN_TRACK_PEAK

#### 3.1.5 REPLAYGAIN_TRACK_MINMAX (ReplayGain 2.0 only)

- **Purpose**: Stores the minimum and maximum sample values for the track, enabling more accurate peak prediction after gain application
- **Format**: Two decimal values separated by a space: `min max`
- **Example**: `0.000000 0.987654`
- **Legacy**: Not present in ReplayGain 1.0

#### 3.1.6 REPLAYGAIN_ALBUM_MINMAX (ReplayGain 2.0 only)

- **Purpose**: Stores the minimum and maximum sample values for the entire album
- **Format**: Identical to REPLAYGAIN_TRACK_MINMAX

### 3.2 Reference Level

#### 3.2.1 ReplayGain 1.0 Reference Level

The reference loudness for ReplayGain 1.0 is **89 dB SPL**. This corresponds to approximately **-18 dBFS** for typical program material. The value was derived from calibration measurements using pink noise and a panel of listeners, and it reflected the average loudness of commercial CDs at the time.

The correspondence can be expressed as: a sine wave of -18 dBFS produces a calibrated SPL of approximately 89 dB when played through a system calibrated so that 0 dBFS corresponds to 107 dB SPL peak.

The reference level establishes the absolute anchor for the loudness measurement. If a track measures at -21 dBFS loudness (mean RMS level relative to full scale), the gain adjustment is:

```
gain = 89 dB SPL (reference) - (-21 dBFS adjusted to SPL) = +21 dBFS adjustment
```

In practice, ReplayGain implementations convert everything to dBFS internally, so the reference becomes -18 dBFS, and:

```
gain_dBFS = -18 - measured_loudness_dBFS
```

#### 3.2.2 ReplayGain 2.0 Reference Level

ReplayGain 2.0 raised the reference level to **95 dB SPL**, reflecting the fact that commercial recordings had gotten louder over the decade since v1.0 was specified. This means that ReplayGain 2.0 tags (with the higher reference) will result in less overall attenuation for most modern music compared to v1.0 tags, which is the correct behavior.

#### 3.2.3 Track vs Album Gain

**Track gain** is computed independently for each track. Each track is analyzed in isolation, and its individual loudness is compared to the reference. This means that when tracks from the same album are played in random order, each track will be normalized to the same perceived loudness.

**Album gain** is computed by analyzing the entire album as one continuous program. The mean RMS is computed across all tracks concatenated together, then compared to the reference. When playing an album in track order, applying album gain ensures the relative loudness differences between tracks (e.g., quiet intros vs loud choruses) are preserved, while the overall album is normalized to the reference level.

The choice between track and album gain is a playback-side decision. Most players offer both options. Album gain is generally preferred when listening to an album in order; track gain is preferred for shuffled playback or playlists containing tracks from multiple albums.

### 3.3 Calculation Method (ReplayGain 1.0)

The ReplayGain 1.0 algorithm processes audio in the following stages:

#### Step 1: Convert to 16-bit PCM

If the input audio is not already 16-bit PCM (e.g., 24-bit FLAC, 32-bit float WAV), it is converted to 16-bit signed PCM before analysis. This conversion uses standard rounding (halfway values rounding away from zero). Dithering is not applied during analysis because the measurement does not need to preserve the quantized representation — it only needs a consistent 16-bit reference.

#### Step 2: RMS Window Definition

The audio is divided into overlapping windows. Each window spans **50 milliseconds** of audio. Windows are positioned every 12.5 milliseconds (giving a **75% overlap**). At 44100 Hz, each window contains 2,205 samples (50 ms × 44100 / 1000 = 2,205), and the hop between windows is 551.25 samples (rounded to 551 samples with a fractional step accumulated). At 48000 Hz, windows are 2,400 samples wide with 600-sample hops.

The window function used is a Hann window (raised cosine), which tapers the edges of each window to reduce spectral leakage:

```
w(n) = 0.5 * (1 - cos(2πn / (N-1)))
```

where N is the window length (number of samples in 50 ms).

#### Step 3: Compute RMS per Window

For each window, the RMS (root mean square) value is computed:

```
RMS_i = sqrt((1/N) * Σ(s(n)^2))  for n = 0 to N-1
```

where s(n) are the windowed samples (sample values in the range [-32768, 32767] for 16-bit signed PCM).

The result is a sequence of RMS values, one per window position across the track.

#### Step 4: Compute Mean of Squared RMS Values

The overall loudness is computed as the mean of the squared RMS values across all windows, then the square root is taken:

```
mean_of_squares = (1/M) * Σ(RMS_i^2)  for i = 1 to M
loudness_dBFS = 10 * log10(mean_of_squares)
```

This double-RMS computation (RMS per window, then mean across windows) weights the loudness toward sustained energy rather than transient peaks. It is conceptually equivalent to a long-term RMS with temporal smoothing.

The resulting `loudness_dBFS` is a negative value (e.g., -21.5 dBFS for a quiet track, -15.0 dBFS for a loud track).

#### Step 5: Compute Gain

```
gain_dB = reference_level_dBFS - loudness_dBFS
```

For ReplayGain 1.0, `reference_level_dBFS = -18`. If the measured loudness is -21 dBFS, the gain is `-18 - (-21) = +3 dB`.

#### Step 6: Peak Value

The peak value is the maximum absolute sample value across the entire track:

```
peak = max(|s(n)|)  for all samples n
peak_normalized = peak / 32768  (for 16-bit PCM)
```

The peak is stored with 6 decimal places as a linear amplitude ratio. A peak of 0.707107 corresponds to -3 dBFS. A peak of 1.000000 corresponds to 0 dBFS (clipping).

### 3.4 Filter Specifications

#### 3.4.1 Pre-Filter

ReplayGain 1.0 does not mandate a specific pre-filter. The original specification references the IEC 60268-1 standard curve but does not require its application in the v1.0 scanner. This is a known limitation — without weighting, the measurement does not account for the ear's frequency-dependent sensitivity.

ReplayGain 2.0 addresses this by specifying an equal-loudness contour weighting, which brings the measurement closer to what EBU R128 uses (K-weighting).

#### 3.4.2 RMS Weighting

In ReplayGain 1.0, the RMS computation is performed without any frequency weighting. All frequencies contribute equally to the loudness estimate. This means that a track with strong bass content but little high-frequency energy might measure louder than a perceptually equivalent track with more high-frequency content, because bass energy contributes disproportionately to the RMS value despite being less audible at the same physical SPL.

This is the primary reason ReplayGain 2.0 and EBU R128 use frequency weighting — to match the measurement to human loudness perception.

#### 3.4.3 The -18 dBFS = 89 dB SPL Correspondence

The -18 dBFS level corresponds to 89 dB SPL under specific calibration conditions:

- A continuous sine wave at -18 dBFS produces a measured SPL of 89 dB when the playback system is calibrated so that 0 dBFS equals 107 dB SPL.
- This calibration assumes pink noise or typical program material has a crest factor (peak-to-RMS ratio) of approximately 12–15 dB.
- The actual SPL produced by a -18 dBFS signal depends on the playback system's gain staging, speaker sensitivity, and room acoustics — but the ReplayGain calibration provides a consistent reference point for relative measurements.

### 3.5 True Peak Detection

#### 3.5.1 Why Sample Peak Is Insufficient

A digital audio signal is a sequence of samples taken at discrete time intervals (e.g., every 1/44100 second at 44.1 kHz). The maximum sample value in a track tells you the largest instantaneous amplitude that was captured in the sample sequence. However, the actual analog waveform between samples can exceed the sample values due to the limited bandwidth of the digital-to-analog reconstruction filter.

Consider a sine wave whose analog peak occurs between two sample points. The samples might read 0.707 at both adjacent samples, but the actual peak of the analog waveform is 1.0. This is called an **inter-sample peak**. When the gain-adjusted signal is converted back to analog, the inter-sample peaks can exceed 0 dBFS and cause clipping in the reconstruction filter, even though all individual samples are below 0 dBFS.

The difference between the sample peak and the true peak can be 1–3 dB depending on the signal content and sample rate. For 44.1 kHz audio, the inter-sample peak of a full-scale sine wave at 997 Hz can be approximately +3 dB relative to the sample peak.

#### 3.5.2 Oversampling Method for True Peak Detection

True peak detection requires reconstructing the analog waveform by oversampling the digital signal. The recommended method is 4x oversampling (upsampling by a factor of 4 using linear interpolation or, preferably, a windowed sinc filter), then finding the maximum absolute sample value in the oversampled signal.

The process:

1. Upsample the signal from its native rate (e.g., 44.1 kHz) to 4x (e.g., 176.4 kHz)
2. Find the maximum absolute value in the upsampled signal
3. Report this as the true peak

ReplayGain 2.0 mandates true peak detection. ReplayGain 1.0 only stores the sample peak.

#### 3.5.3 ITU-R BS.1770-4 True Peak Algorithm

The ITU-R BS.1770-4 specification defines true peak measurement as the maximum value of the oversampled signal. The standard recommends 4x oversampling using a linear-phase filter. The oversampling filter must have a passband that covers at least 0 Hz to 20 kHz (for audio applications) with less than 0.1 dB ripple, and sufficient stopband attenuation to avoid introducing spurious peaks.

The true peak value is reported in dBTP (decibels relative to true peak), where 0 dBTP = 0 dBFS sample peak (not the true peak — this is the convention used to distinguish the measurement).

---

## 4. EBU R128 SPECIFICATION

### 4.1 Loudness Measurement (ITU-R BS.1770-4)

The EBU R128 loudness measurement is based on ITU-R BS.1770-4, which defines the algorithm for measuring audio loudness in a way that correlates with human perception. The algorithm consists of four stages: K-weighting filtering, channel weighting, block-wise loudness computation, and gated loudness aggregation.

#### 4.1.1 K-Weighting Filter

The K-weighting filter combines two cascaded stages: a high-shelf pre-filter and a Revised Low-frequency B-weighting (RLB) high-pass filter.

**High-shelf pre-filter**: This filter provides a +4 dB boost at 1681.97 Hz, rolling off toward lower and higher frequencies. The transfer function is:

```
H_pre(s) = (s + 2π × 1681.97) / (s + 2π × 1681.97 / 2)
```

The shelving frequency of approximately 1682 Hz was derived from psychoacoustic equal-loudness contour data. The shelving gain of +4 dB at this frequency means that frequencies around 1682 Hz contribute more to the perceived loudness than lower frequencies at the same physical level.

**RLB (Revised Low-frequency B-weighting) high-pass filter**: This stage removes very low frequency content (below approximately 38 Hz) that contributes disproportionately to acoustic measurements but is poorly perceived by the human ear. The transfer function is:

```
H_hp(s) = s^2 / (s^2 + (2π × 38.13547) × s + (2π × 38.13547)^2)
```

The corner frequency of approximately 38.13547 Hz is the RLB's defining parameter.

**Combined K-weighting**: The total K-weighting response is the cascade of the pre-filter and the RLB filter. The name "K-weighting" derives from the "K" coefficient used in the EBU Tech 3341 document that formalizes the measurement.

The combined K-weighting curve has:
- A +4 dB peak around 1682 Hz
- Gradual rolloff below 500 Hz (approximately -2 dB at 250 Hz, -5 dB at 100 Hz)
- Steep rolloff below 40 Hz (below the RLB cutoff)
- Gradual rolloff above 5 kHz (approximately -2 dB at 10 kHz)

This frequency response models the ear's sensitivity across the audible range, with the 2–4 kHz range being the most sensitive and the extreme lows and highs contributing less.

#### 4.1.2 Channel Weighting

In stereo or multichannel programs, each channel is processed independently through the K-weighting filter, then all channels are combined with equal weight. The LFE (Low-Frequency Effects) channel is excluded — its weight is 0 in the loudness computation.

For a 5.1 surround signal (Left, Right, Center, Left Surround, Right Surround, LFE):
- Channels 1–5 (L, R, C, Ls, Rs): weight = 1.0
- Channel 6 (LFE): weight = 0.0

For stereo (Left, Right):
- Both channels: weight = 1.0

This weighting is based on the assumption that all loudspeaker positions contribute equally to the listening experience, and the LFE channel's content is already accounted for in the bass management of the reproduction system.

#### 4.1.3 Block-Wise Loudness Computation

After K-weighting and channel combination, the signal is divided into overlapping blocks:

- **Block duration**: 400 milliseconds
- **Hop size**: 75 milliseconds (giving 75% overlap)
- **Window function**: Hann window (same as ReplayGain)
- **Per block**: Compute the mean of the squared K-weighted, channel-combined samples

```
L_k(t) = (1/N) * Σ(s_k(n)^2)  for n in block starting at t
```

The block length of 400 ms was chosen to approximate the integration time of human loudness perception — the ear's ability to integrate loudness over time is not instantaneous but also not infinitely long.

#### 4.1.4 Gating

Gating is the process of excluding blocks that correspond to silence or near-silence from the loudness computation. This is the most important conceptual difference between ReplayGain and EBU R128.

**Absolute gate (first stage)**: Any block with loudness below **-70 LUFS** is excluded. This threshold is far below any program material of interest and effectively removes blocks that correspond to complete silence. It prevents silent portions of a track from dragging down the integrated loudness.

**Relative gate (second stage)**: After computing the mean loudness of all non-gated blocks, a relative gate is applied at **-10 LU** relative to the resulting integrated loudness value. Only blocks that are within 10 LU of the computed loudness are included in the final integrated loudness calculation.

The two-stage gating process ensures that:
1. Silence is excluded (absolute gate)
2. Especially quiet segments (e.g., quiet passages in an otherwise loud film) are excluded from the integrated loudness (relative gate)

The iterative nature of the relative gate (it requires computing the initial loudness before determining which blocks to include) means that the algorithm typically requires two passes over the block data.

#### 4.1.5 Integrated Loudness (Loudness, Integrated)

The integrated loudness is the gated mean loudness across the entire program, reported in LUFS. It represents the overall perceived loudness of the complete audio program, excluding silence and quiet passages.

```
L_KG = 10 * log10(mean(L_k) over gated blocks)
```

The term "integrated" refers to the fact that this single number summarizes the loudness of the entire program, integrating across time.

#### 4.1.6 Loudness Range (LRA)

The Loudness Range (LRA) measures the dynamic range of the program — how much the loudness varies from quiet to loud sections. It is calculated as the difference between the 95th percentile and the 10th percentile of the loudness distribution of the gated blocks.

**Algorithm**:
1. Take all gated blocks above the absolute and relative gates
2. Compute the loudness of each block (L_k values)
3. Sort the block loudness values
4. LRA = 95th percentile value - 10th percentile value

The LRA is reported in LU (Loudness Units), not LUFS. It is a relative measure — it tells you the range of loudness variation, not an absolute level. A typical film soundtrack might have an LRA of 15–20 LU. A heavily compressed pop song might have an LRA of 4–6 LU.

### 4.2 EBU R128 Loudness Levels

#### 4.2.1 Target Integrated Loudness

The primary target for EBU R128 broadcast is **-23 LUFS**. This value was chosen based on:
- Extensive listening tests showing that -23 LUFS corresponds to a comfortable listening level for broadcast content
- Compatibility with existing calibration practices (which often used -23 LUFS as a reference)
- Sufficient headroom to accommodate peaks without excessive limiting

#### 4.2.2 Target Loudness Range (LRA)

The maximum recommended LRA for EBU R128 is **20 LU**. This is not a hard target but a guideline — content with higher LRA is acceptable if the program design requires it. However, the LRA limit ensures that dynamic range compression is not being applied as a workaround to hit the integrated loudness target.

#### 4.2.3 Maximum True Peak

The maximum true peak for EBU R128 broadcast is **-1 dBTP**. This means that after loudness normalization, the true peak (including inter-sample peaks) must not exceed -1 dBTP. The -1 dB headroom accounts for the reconstruction filter's behavior in the playback DAC.

Streaming platforms often use different true peak limits (e.g., -2 dBTP, or no limit at all).

#### 4.2.4 Momentary Loudness

Momentary loudness is the loudness measured in a **400 ms** window, updated every 100 ms (or with the same 75% overlap as the block computation). It is a short-term loudness measurement that tracks the envelope of the program's loudness.

Momentary loudness is useful for:
- Real-time loudness meters in broadcast equipment
- Detecting sudden loudness changes (e.g., explosions in film sound)
- Ensuring compliance with momentary loudness limits in some broadcast standards

#### 4.2.5 Short-Term Loudness

Short-term loudness is the loudness measured in a **3-second** sliding window. It provides a medium-term loudness measurement that smooths out momentary fluctuations while still tracking program-level changes.

Short-term loudness is useful for:
- Assessing the overall loudness of program segments
- Detecting loudness drift within a program
- Supplementing the integrated loudness measurement

#### 4.2.6 LUFS (Loudness Units Full Scale)

LUFS is the unit of absolute loudness in the R128 system. The "Full Scale" designation means that the measurement is referenced to the digital full-scale range (like dBFS), but the measurement has been perceptually weighted (K-weighted) and gated. The LUFS value directly indicates the loudness level relative to the reference.

A program at -23 LUFS has the same perceived loudness as a -23 dBFS K-weighted pink noise signal. A program at -14 LUFS sounds approximately 9 dB louder.

### 4.3 EBU R128 Loudness Meters

#### 4.3.1 Type I Meters

Type I loudness meters implement the full ITU-R BS.1770-4 algorithm, including K-weighting, channel weighting, block-wise computation, and gating. They display:
- Integrated loudness (LUFS)
- True peak (dBTP)
- Loudness Range (LU)
- Momentary loudness (optional)
- Short-term loudness (optional)

Type I meters are used for compliance measurement and authoritative loudness certification.

#### 4.3.2 Type II Meters

Type II meters are simplified peak-only meters that estimate loudness from the peak level using a calibration curve. They do not implement the full K-weighting and gating algorithm. Their accuracy is lower than Type I meters, but they are less complex and less expensive. They are useful for real-time monitoring but not for compliance measurement.

### 4.4 Program Loudness Standards by Region/Platform

| Platform | Target LUFS | Max True Peak | Max LRA | Notes |
|---|---|---|---|---|
| EBU R128 | -23 | -1 dBTP | 20 LU | European broadcast standard |
| ATSC A/85 | -24 | -2 dBTP | N/A | US broadcast standard |
| Netflix | -27 | -14 dBFS | N/A | Streaming, dialogue-normalized |
| Spotify | -14 | 0 dBTP | N/A | Streaming, loudness penalty |
| Apple Music | -16 | -1 dBTP | N/A | Streaming |
| YouTube | -14 | 0 dBTP | N/A | Streaming, uses loudnorm |
| TIDAL | -14 | -1 dBTP | N/A | Streaming |
| Amazon Music | -11 to -16 | 0 dBTP | N/A | Streaming, genre-dependent |
| Apple TV+ | -14 | -1 dBTP | N/A | Streaming originals |
| Disney+ | -14 | -2 dBTP | N/A | Streaming |
| ARD/ZDF (Germany) | -23 | -1 dBTP | 20 LU | German public broadcast |
| BBC (UK) | -23 | -1 dBTP | N/A | UK public broadcast |
| France Télévisions | -24 | -1 dBTP | 20 LU | French public broadcast |
| Spotify Loud | -11 | 0 dBTP | N/A | High-loudness option |
| Spotify Normal | -14 | 0 dBTP | N/A | Default setting |
| Spotify Quiet | -21 | 0 dBTP | N/A | Reduced-loudness option |

**Spotify's Loudness Penalty**: Spotify applies a loudness penalty system where tracks louder than -14 LUFS are not further boosted — instead, they are left as-is but can be subjected to compression/limiting if too loud. Tracks quieter than -14 LUFS are boosted. The "Spotify Loud" mode targets -11 LUFS with additional limiting.

**Netflix's Dialogue Normalization**: Netflix's -27 LUFS target is calibrated for dialogue intelligibility. The standard includes specific guidance on dialogue gating — the loudness measurement can focus on dialogue-frequency content to ensure speech is not masked.

---

## 5. GAIN APPLICATION

### 5.1 Applying ReplayGain in Playback

#### 5.1.1 How Players Apply the Gain Adjustment

When a player encounters a ReplayGain-tagged file, it reads the `REPLAYGAIN_TRACK_GAIN` value (or `REPLAYGAIN_ALBUM_GAIN` if album mode is selected) and applies the gain to the audio during playback. The gain is applied as a linear multiplier to the PCM sample values:

```
gain_linear = 10^(gain_dB / 20)
output_sample = input_sample * gain_linear
```

For example, a gain of +3.20 dB translates to a linear multiplier of `10^(3.20/20) = 10^0.16 ≈ 1.445`. Each sample value is multiplied by 1.445 before being sent to the output buffer.

The gain application happens in the player's DSP pipeline, typically after decoding and before output resampling. For floating-point pipelines, the computation is exact. For fixed-point pipelines, the gain multiplier is computed as a fixed-point Q-value.

#### 5.1.2 Peak Limiting / Soft Clipping

If applying the gain adjustment would cause any sample to exceed 0 dBFS (i.e., `peak * gain_linear > 1.0`), the player has several options:

**Option 1: Apply gain as-is (accept clipping)**: The player applies the full gain, and samples that exceed the clipping threshold are hard-clipped to 0 dBFS. This introduces distortion and is generally not recommended.

**Option 2: Scaled gain to prevent clipping**: The player computes the maximum safe gain: `safe_gain_dB = 20 * log10(1.0 / peak)`. If the track gain exceeds this value, the player applies only `safe_gain_dB` instead. This preserves the peak at 0 dBFS but reduces the effective loudness normalization.

**Option 3: Soft clipping / limiting**: The player applies a limiter that gradually compresses peaks above a threshold (e.g., -0.3 dBFS) with a fast attack and moderate release. This allows the full gain to be applied while preventing hard clipping artifacts.

**Option 4: Peak-aware gain application**: The player uses the `REPLAYGAIN_TRACK_PEAK` value (which may already reflect true peak) to pre-calculate whether the gain would cause clipping, and selects the appropriate strategy.

Most modern players implement Option 2 or 3. The ReplayGain specification recommends that players apply the gain as-is and let the user manage the preamp level.

#### 5.1.3 The Preamp Control

Most ReplayGain-aware players expose a "preamp" control — an additional gain offset (in dB) that is applied in addition to the ReplayGain value. This allows the user to fine-tune the normalization behavior:

- **Positive preamp**: Increase the loudness of all normalized tracks (useful in quiet listening environments)
- **Negative preamp**: Decrease the loudness (useful in noisy environments or with sensitive speakers)

The effective gain applied to a track is: `effective_gain = track_gain + preamp_gain`.

#### 5.1.4 Fallback Gain for Files Without Tags

For files that lack ReplayGain tags, most players apply a **fallback gain** value. The fallback is typically configurable:

- **0 dB (default)**: No adjustment is made. The track plays at its original loudness.
- **Some players use -6 dB or -12 dB** as a conservative fallback to prevent unexpected clipping when mixing tagged and untagged files.

The fallback behavior is player-specific. foobar2000, VLC, and mpv all provide a configurable fallback gain setting.

### 5.2 Applying EBU R128 in Mastering

#### 5.2.1 Linear Gain Change to Hit Target LUFS

The first step in R128 mastering is to measure the program's integrated loudness, then calculate a linear gain adjustment to bring it to the target LUFS.

```
target_LUFS = -23  (for broadcast)
measured_LUFS = <from measurement>
gain_dB = target_LUFS - measured_LUFS
gain_linear = 10^(gain_dB / 20)
```

This linear gain is applied to the entire program. For a program at -19 LUFS, the required gain is `-23 - (-19) = -4 dB` (attenuation). For a program at -27 LUFS, the required gain is `-23 - (-27) = +4 dB` (boost).

#### 5.2.2 True Peak Limiting

After the linear gain adjustment, the true peak must be checked against the platform's maximum true peak limit. If any inter-sample peaks exceed the limit (e.g., -1 dBTP for broadcast), a true peak limiter must be applied.

The true peak limiter is different from a conventional dynamic range limiter:

- **Brick-wall limiter**: An infinitely hard limiter that cuts any sample above the threshold to exactly the threshold value. Produces the cleanest true peak control but can introduce significant distortion on transient material.

- **Look-ahead limiter**: Examines the signal a few milliseconds ahead before applying gain reduction. Allows the limiter to respond to peaks without the distortion that would be caused by a slow attack.

- **Transparent limiter**: A limiter designed to operate below the threshold, applying gentle gain reduction only when necessary. The goal is to keep the true peak below the limit while minimizing audible artifacts.

For EBU R128 broadcast mastering, the standard approach is a look-ahead true peak limiter with:
- **Threshold**: -1 dBTP (or the platform's maximum true peak)
- **Attack time**: Very short (≤ 1 ms) to catch fast transients
- **Release time**: Moderate (50–200 ms) to avoid "pumping" artifacts
- **Linkage**: Channel-linking so that multi-channel content is limited uniformly across all channels (prevents stereo image shifts from asymmetric limiting)

#### 5.2.3 Limiter Attack and Release Time Considerations

The attack time of a true peak limiter must be extremely short because inter-sample peaks can occur within a single sample period. A limiter with a 10 ms attack time cannot respond quickly enough to catch a peak that exists for less than that time.

For true peak limiting, the limiter must:
- Respond within 0.5–1 ms to catch brief inter-sample peaks
- Use look-ahead (typically 1–5 ms) to detect approaching peaks before they occur
- Apply gain reduction uniformly across all channels to maintain stereo balance

The release time controls how quickly the limiter releases the gain reduction after the peak has passed. Too short a release time causes "pumping" — audible modulation of the background noise at the release rate. Too long a release time causes the limiter to "ride" the program, reducing the effective loudness. A release time of 100–200 ms with a variable (program-dependent) release curve is typical.

### 5.3 Gain in dB vs Linear

The relationship between gain in decibels and the linear amplitude multiplier is:

```
gain_linear = 10^(gain_dB / 20)
```

The exponent of 20 reflects the fact that power is proportional to the square of amplitude, and decibels are fundamentally a power ratio. When converting from amplitude gain to decibels: `gain_dB = 20 * log10(gain_linear)`.

**Examples**:

| Gain (dB) | Linear Multiplier | Effect |
|---|---|---|
| +6 | 1.9953 | Doubles perceived loudness |
| +3 | 1.4125 | Moderate increase |
| +1 | 1.1220 | Small increase |
| 0 | 1.0000 | No change |
| -1 | 0.8913 | Small decrease |
| -3 | 0.7079 | Halves loudness |
| -6 | 0.5012 | Quarter loudness |

**Practical implementation**: For integer PCM (e.g., 16-bit signed samples in range [-32768, 32767]), the gain application is:

```python
gain_linear = 10 ** (gain_dB / 20.0)
output_sample = int(input_sample * gain_linear)
output_sample = max(-32768, min(32767, output_sample))  # clip to range
```

For floating-point PCM (32-bit float in range [-1.0, 1.0]):

```python
gain_linear = 10 ** (gain_dB / 20.0)
output_sample = input_sample * gain_linear
```

Floating-point pipelines can typically handle values outside the [-1.0, 1.0] range during intermediate computation, with final clipping applied at the output stage.

---

## 6. TAG FIELD NAMES PER FORMAT

### 6.1 ReplayGain in Vorbis Comments (FLAC, OGG)

Vorbis Comments use key=value text pairs stored as a flattened list. ReplayGain fields are stored as plain text strings:

```
REPLAYGAIN_TRACK_GAIN=+3.20 dB
REPLAYGAIN_TRACK_PEAK=0.876543
REPLAYGAIN_ALBUM_GAIN=-1.50 dB
REPLAYGAIN_ALBUM_PEAK=0.987654
```

FLAC, Ogg Vorbis, and Opus all use Vorbis Comments for metadata, so this format applies to all three.

The field names are case-sensitive. The " dB" suffix (with a leading space before "dB") is required. Values must not contain leading or trailing whitespace beyond the space before "dB".

### 6.2 ReplayGain in ID3v2 (MP3)

ID3v2 uses frames to store metadata. ReplayGain fields are stored in **TXXX** (user-defined text) frames:

```
Frame ID: TXXX
Text encoding: $03 (UTF-8)
Description: REPLAYGAIN_TRACK_GAIN
Value: +3.20 dB
```

Each ReplayGain field requires a separate TXXX frame. The description field must exactly match the ReplayGain field name (case-sensitive). Multiple TXXX frames with the same description are not allowed (ID3v2 requires unique description values).

```
Frame 1: TXXX, description="REPLAYGAIN_TRACK_GAIN", value="+3.20 dB"
Frame 2: TXXX, description="REPLAYGAIN_TRACK_PEAK", value="0.876543"
Frame 3: TXXX, description="REPLAYGAIN_ALBUM_GAIN", value="-1.50 dB"
Frame 4: TXXX, description="REPLAYGAIN_ALBUM_PEAK", value="0.987654"
```

### 6.3 ReplayGain in MP4/iTunes Atoms

MP4/M4A uses a hierarchical atom (box) structure. ReplayGain is stored as **freeform atoms** within the 'ilst' container, using the `----` atom type (mean: "com.apple.iTunes"):

```
'----' atom:
  mean: "com.apple.iTunes" (4 bytes)
  name: "REPLAYGAIN_TRACK_GAIN" (variable-length string)
  data: "+3.20 dB" (UTF-8 string)
```

The full atom path is: `/moov/udta/meta/ilst/----[mean=com.apple.iTunes,name=REPLAYGAIN_TRACK_GAIN]/data`

Multiple freeform atoms can share the same name within the `----` parent, but ReplayGain implementations typically use one atom per field.

### 6.4 ReplayGain in APEv2 (Musepack, WMA, WAV)

APEv2 tags use binary key=value items. ReplayGain fields are stored as text:

```
Item: REPLAYGAIN_TRACK_GAIN
Value: +3.20 dB
Flags: UTF-8 encoded text
```

APEv2 supports both text and binary item types. All ReplayGain values are text items. The item key is case-sensitive and must match the standard field name exactly.

### 6.5 EBU R128 in FLAC

EBU R128 fields in FLAC (stored as Vorbis Comments) use the following naming convention:

```
REPLAYGAIN_REFERENCE_LOUDNESS=89 dB        # or "95 dB" for RG 2.0
REPLAYGAIN_ALBUM_GAIN=+3.20 dB
REPLAYGAIN_ALBUM_PEAK=0.987654
REPLAYGAIN_TRACK_GAIN=+2.10 dB
REPLAYGAIN_TRACK_PEAK=0.876543
LOUDNESS=-23.4 LUFS                      # Integrated loudness
LOUDNESS_RANGE=12.3 LU                   # LRA
MOMENTARY=-21.0 LUFS                      # Momentary loudness
SHORT-TERM=-22.5 LUFS                     # Short-term loudness
TRUE_PEAK=-0.8 dBTP                       # True peak
INTEGRATED=-23.4 LUFS                     # Integrated loudness (alias)
```

The EBU R128 specific tags (LOUDNESS, LOUDNESS_RANGE, etc.) are stored alongside the standard ReplayGain tags in FLAC files. These tags are written by R128-aware scanners (e.g., FFmpeg's loudnorm, R128GAIN) and are read by R128-aware players.

Note: The LOUDNESS tag format varies between implementations. Some use "LUFS" suffix (e.g., "-23.4 LUFS"), while others omit it. Always check the scanner's output format.

### 6.6 EBU R128 in MP4

In MP4/M4A containers, EBU R128 loudness data is stored using Apple's freeform atom mechanism, similar to ReplayGain. However, there is no standardized atom name for loudness tags — implementations vary:

```
'----' atom (mean: "com.apple.iTunes"):
  name: "LOUDNESS"
  data: "-23.4 LUFS"

'----' atom (mean: "com.apple.iTunes"):
  name: "LOUDNESS_RANGE"
  data: "12.3 LU"

'----' atom (mean: "com.apple.iTunes"):
  name: "TRUE_PEAK"
  data: "-0.8 dBTP"
```

Some implementations also use the '©too' (encoder) or '©cmt' (comment) atoms to carry loudness metadata in a standardized way, though this is not formally specified.

FFmpeg's loudnorm filter embeds R128 data using the `REPLAYGAIN_*` and `LOUDNESS_*` tags in Vorbis Comment format, which map to the equivalent freeform atoms in MP4 through FFmpeg's metadata handling.

---

## 7. FFMPEG IMPLEMENTATION REFERENCE

### 7.1 ReplayGain Scanning with FFmpeg

FFmpeg's `loudnorm` filter can perform loudness measurement and output ReplayGain-compatible values. The `print_format` option controls the output format.

#### Measuring loudness and printing ReplayGain-style values:

```bash
ffmpeg -i input.flac -af loudnorm=print_format=json:iframe=1:lagate=1:推=1:i=-23:lra=20:tp=-1 output.wav
```

This command measures the input file and outputs JSON with measured loudness values that can be used to compute the gain adjustment.

#### Measuring with summary output:

```bash
ffmpeg -i input.flac -af loudnorm=I=-23:TP=-1.5:LRA=11:print_format=summary -f null -
```

This outputs a human-readable summary with measured integrated loudness, true peak, and LRA.

### 7.2 EBU R128 Loudness Measurement

The `loudnorm` filter implements ITU-R BS.1770-4 loudness measurement. The basic measurement command:

```bash
ffmpeg -i input.flac -af loudnorm=I=-23:TP=-1.5:LRA=11:print_format=summary -f null -
```

Output example:
```
[Parsed_loudnorm_0 @ 0x55a3b8c0e8c0] Input Integrated: -20.3 LUFS
[Parsed_loudnorm_0 @ 0x55a3b8c0e8c0] Input True Peak: -0.5 dBTP
[Parsed_loudnorm_0 @ 0x55a3b8c0e8c0] Input LRA: 8.2 LU
[Parsed_loudnorm_0 @ 0x55a3b8c0e8c0] Input Threshold: -30.1 LUFS
```

### 7.3 Applying Loudness Normalization (Two-Pass)

FFmpeg's `loudnorm` filter operates in two passes for accurate normalization:

**Pass 1 — Measure:**
```bash
ffmpeg -i input.flac -af loudnorm=I=-16:TP=-1:LRA=11:print_format=json -f null -
```

This outputs JSON with the measured loudness values.

**Pass 2 — Apply:**
```bash
ffmpeg -i input.flac -af loudnorm=I=-16:TP=-1:LRA=11:measured_I=-20.3:measured_TP=-0.5:measured_LRA=8.2:measured_thresh=-30.1:linear=true:print_format=summary output.flac
```

The second pass reads the measured values from Pass 1 and applies the appropriate gain and limiting. The `linear=true` parameter applies a linear gain (without limiting) for the first pass, and the second pass applies both the gain and peak limiting.

### 7.4 Reading Loudness with ffprobe

ffprobe can extract loudness information from streams:

```bash
ffprobe -show_streams -select_streams a:0 -read_intervals "%+60" -count_frames -of json input.flac
```

The `-read_intervals "%+60"` flag reads the first 60 seconds of the file (for faster analysis). For full-file analysis, use the entire file or a larger interval.

### 7.5 loudnorm Filter Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| I | float | -24 | Target integrated loudness (LUFS) |
| TP | float | -1.0 | Maximum true peak (dBTP) |
| LRA | float | N/A | Target loudness range (LU) |
| linear | bool | false | Apply gain linearly without limiting |
| Iconst | bool | true | Constant integrated loudness (anchor-based) |
| measured_I | float | N/A | Measured integrated loudness (from Pass 1) |
| measured_TP | float | N/A | Measured true peak (from Pass 1) |
| measured_LRA | float | N/A | Measured loudness range (from Pass 1) |
| measured_thresh | float | N/A | Measured threshold (from Pass 1) |
| offset | float | 0.0 | Gain offset applied after normalization |
| dual_mono | bool | false | Treat input as dual mono |
| print_format | str | summary | Output format: summary, json, none |
| iframe | int | 0 | Frame interval for logging (0=off) |
| lagate | bool | false | Enable loudness-gated measurement |
| ebuf | bool | false | Enable extended buffer delay analysis |
| peak | str | sample | Peak type: sample or true |

---

## 8. TOOLS REFERENCE

### 8.1 ReplayGain Scanners

#### R128GAIN

R128GAIN is a command-line tool that performs EBU R128 loudness measurement and gain application. It implements the full ITU-R BS.1770-4 algorithm and supports batch processing of entire directories.

```bash
r128gain -m -12 /path/to/album/
```

The `-m` flag applies the gain (modifies files in place or creates new files). The `-12` flag sets the target LUFS to -12.

R128GAIN writes ReplayGain-style tags (`REPLAYGAIN_TRACK_GAIN`, `REPLAYGAIN_ALBUM_GAIN`, `REPLAYGAIN_TRACK_PEAK`, `REPLAYGAIN_ALBUM_PEAK`) along with R128-specific tags (`LOUDNESS`, `TRUE_PEAK`, `LOUDNESS_RANGE`).

#### FFmpeg (loudnorm filter)

FFmpeg's `loudnorm` filter performs both R128 measurement and normalization. See Section 7 for detailed usage.

#### aacgain / mp3gain

`mp3gain` and `aacgain` are ReplayGain scanners specifically for MP3 and AAC files. They modify the files in-place using:
- **MP3**: Stores ReplayGain tags in ID3v2 TXXX frames
- **AAC**: Stores ReplayGain tags in iTunes-compatible metadata atoms

mp3gain can also apply the gain by modifying the MP3 frames directly (using the SCFT ramped gain feature), though this is less common as it permanently alters the audio.

```bash
mp3gain -c -r -k -d 3.0 /path/to/album/*.mp3
```

- `-c`: Create a backup before modifying
- `-r`: Apply track gain automatically
- `-k`: Apply album gain (keeps album gain, adjusts track gain to prevent clipping)
- `-d 3.0`: Set the reference level to 89 + 3 = 92 dB SPL

#### vorbisgain

`vorbisgain` scans FLAC and Ogg Vorbis files and writes ReplayGain tags:

```bash
vorbisgain -a /path/to/album/*.flac
```

The `-a` flag applies album gain (writes both track and album tags). Without `-a`, only track gain is written.

#### metaflac

`metaflac` is the FLAC utility that can set ReplayGain tags directly:

```bash
metaflac --set-tag=REPLAYGAIN_TRACK_GAIN=+3.20+dB \
         --set-tag=REPLAYGAIN_TRACK_PEAK=0.876543 \
         --set-tag=REPLAYGAIN_ALBUM_GAIN=-1.50+dB \
         --set-tag=REPLAYGAIN_ALBUM_PEAK=0.987654 \
         input.flac
```

### 8.2 Loudness Meters

#### TT DR (Type I Loudness Meter)

TT DR is a hardware and software Type I loudness meter that implements ITU-R BS.1770-4. It displays:
- Integrated loudness (LUFS)
- True peak (dBTP)
- Loudness Range (LU)
- Momentary loudness (LUFS)
- Short-term loudness (LUFS)
- LRA bar graph

TT DR is widely used in mastering and broadcast facilities for compliance measurement.

#### Nugen Audio VisLM

VisLM is a plugin-based loudness meter (VST/AU/AAX) from Nugen Audio. It provides real-time and offline loudness analysis with EBU R128, ATSC A/85, and custom target support. Features include:
- Real-time loudness display
- History logging
- Short-term and momentary meters
- Peak limiting with true peak detection

#### Waves WLM

The Waves WLM (Waves Loudness Meter) plugin provides a simplified loudness meter with Type II accuracy. It is less accurate than Type I meters but is widely available as part of the Waves plugin bundle.

#### YouLean

YouLean is a free, open-source loudness meter available as a plugin and standalone application. It implements EBU R128 measurement and is useful for quick loudness checks without purchasing commercial software.

---

## 9. TRUE PEAK DETECTION ALGORITHM

### 9.1 Oversampling Method

True peak detection reconstructs the analog waveform by interpolating between the discrete digital samples. The algorithm works as follows:

#### 9.1.1 Oversampling Process

1. **Upsample** the signal from its native sample rate (e.g., 44100 Hz) to 4x the rate (e.g., 176400 Hz) using a high-quality interpolation filter
2. **Find the maximum absolute value** in the oversampled signal
3. **Convert back to linear amplitude** as a ratio relative to full scale

The interpolation method affects the accuracy:
- **Linear interpolation**: Fast but imprecise; underestimates true peak by 0.5–1 dB for high-frequency content
- **Cubic spline**: Better accuracy than linear, moderate computational cost
- **Windowed sinc**: Best accuracy; recommended for production tools. The sinc function is the ideal lowpass filter, and windowing (e.g., Hann window) truncates the infinite sinc to a finite length

The standard recommendation is a **4x oversampling ratio** using a **linear-phase FIR filter** with:
- Passband: 0 Hz to 0.9 × Nyquist of the original signal (e.g., 0–19.8 kHz for 44.1 kHz input)
- Stopband attenuation: > 60 dB
- Transition bandwidth: sufficient to achieve the stopband attenuation

#### 9.1.2 Finding Peak Sample Value

After oversampling, the maximum absolute value across all samples (including oversampled values) is:

```python
true_peak = max(abs(s_oversampled[n])) for all n
true_peak_ratio = true_peak / (2^(bit_depth - 1))  # normalize to full scale
true_peak_dBTP = 20 * log10(true_peak_ratio)
```

For 16-bit audio, the full-scale peak value is 32767. For 24-bit, it is 8388607.

#### 9.1.3 Inter-Sample Peak Detection

The oversampled waveform may have peaks at positions that were not sampled in the original signal. These inter-sample peaks can be significantly higher than the sample peaks, especially for:

- **High-frequency content**: A sine wave at 997 Hz (a common test frequency) with samples at -3 dBFS can have true peaks at +0.5 dBFS when oversampled.
- **Transient content**: Percussive sounds with fast attack can create inter-sample peaks in the attack transient.
- **Stereophonic content**: Phase relationships between channels can create inter-sample peaks that don't appear in individual channels.

The worst-case inter-sample peak for a full-scale sine wave at 44.1 kHz sampling rate is approximately +3.01 dB relative to the sample peak. This means a track with a sample peak of -0.5 dBFS could have true peaks of +2.5 dBTP.

### 9.2 ITU-R BS.1770-4 True Peak

The ITU-R BS.1770-4 specification defines true peak measurement as follows:

1. **Channel-dependent K-weighting**: Each audio channel is processed through the K-weighting filter independently.
2. **Oversampling**: Each channel's K-weighted signal is upsampled to 4× the original sample rate using a linear-phase lowpass filter.
3. **Maximum detection**: For each channel, find the maximum absolute value in the oversampled signal.
4. **Combined true peak**: The overall true peak is the maximum across all channels.

The K-weighting filter is applied before oversampling because the true peak should reflect the perceived loudness-relevant content, not just raw sample amplitudes. High-frequency content above the K-weighting cutoff contributes less to true peak for loudness purposes.

The specification requires the oversampling filter to have:
- Passband ripple: ≤ 0.1 dB in the audio band (0–20 kHz)
- Stopband attenuation: ≥ 60 dB
- Phase response: linear phase (symmetric FIR)

---

## 10. PLAYBACK CONSIDERATIONS

### 10.1 Peak Limiting

#### 10.1.1 When Gain Would Cause Clipping

When the ReplayGain track gain (plus any preamp offset) would push any sample above 0 dBFS after application, the player must decide how to handle the excess:

**Hard clipping**: Any sample above 0 dBFS is truncated to 0 dBFS. This introduces 2nd and 3rd harmonic distortion at every clipping event. Perceptually, hard clipping sounds harsh and brittle, especially on transient material. It is generally unacceptable for music playback.

**Soft clipping**: Samples above the clipping threshold are smoothly compressed using a nonlinear function. A typical soft-clipper uses a tanh() function:

```python
output = threshold * tanh(input / threshold)
```

This produces gradual compression that sounds less harsh than hard clipping but still introduces some distortion.

**Peak limiting with gain reduction**: The player's DSP applies a gain reduction signal to the entire track whenever the peak would exceed the threshold. This is the cleanest approach but requires real-time dynamic processing.

**Gain scaling to prevent clipping**: The player reduces the overall gain so that the track's peak (after gain application) reaches exactly 0 dBFS. This is the most common player behavior.

#### 10.1.2 Attack and Release Time Constants

For dynamic peak limiting (as opposed to simple gain scaling), the limiter uses:

- **Look-ahead**: 1–5 ms of signal buffering allows the limiter to detect approaching peaks before they arrive, preventing the limiter from "missing" the attack.
- **Attack time**: 0.1–1.0 ms. The limiter must respond almost instantly to catch inter-sample peaks.
- **Release time**: 50–200 ms. The gain reduction is released gradually to avoid pumping artifacts.

For ReplayGain playback, most players use simple gain scaling (Option 2 from Section 5.1.2) rather than a full limiter, because ReplayGain gain values are typically small (usually within ±6 dB), and the `REPLAYGAIN_TRACK_PEAK` value provides advance warning of potential clipping.

### 10.2 Album vs Track Gain

#### 10.2.1 Track Gain Mode

In track gain mode, each track's `REPLAYGAIN_TRACK_GAIN` value is applied independently. When playing a playlist of tracks from different albums:

- All tracks normalize to the same perceived loudness (the ReplayGain reference level)
- The listener does not need to adjust volume between tracks
- Any dynamic range differences within each track are preserved

This is the correct mode for **shuffled playlists**, **mixed-genre playlists**, and **radio-style playback**.

#### 10.2.2 Album Gain Mode

In album gain mode, the player's `REPLAYGAIN_ALBUM_GAIN` value is applied to all tracks in the album, and the individual `REPLAYGAIN_TRACK_GAIN` values are ignored. The album's internal loudness relationships are preserved:

- Quiet intros remain quieter than loud sections within the same album
- The listener experiences the album as the mastering engineer intended in terms of dynamics
- However, when switching between albums, volume adjustment may still be needed if albums have very different average loudness

This is the correct mode for **listening to an album in order**.

#### 10.2.3 When to Use Which

| Scenario | Recommended Mode | Reason |
|---|---|---|
| Playing an album track-by-track | Album gain | Preserves internal dynamics and album flow |
| Shuffled playlist | Track gain | Normalizes across all tracks |
| Podcast playback | Track gain (or fixed gain) | Speech content needs consistent loudness |
| Classical music | Track gain (or album gain carefully) | Classical music's wide dynamics are part of the art |
| Live recordings | Album gain | Maintains the live concert experience |

### 10.3 Fallback Gain

#### 10.3.1 Default Behavior

For tracks without ReplayGain tags, most players offer these fallback options:

- **0 dB (default)**: Play at original loudness. The listener may need to manually adjust volume when switching to untagged tracks.
- **-6 dB**: Conservative attenuation. Prevents clipping if the untagged track is significantly louder than the normalized tracks.
- **Player-specific**: Some players (notably older implementations) apply a fixed gain of -6 dB or use a detection algorithm to estimate loudness from peak levels.

#### 10.3.2 Handling Mixed Playlists

When a playlist contains both tagged and untagged tracks, the fallback gain applies only to untagged tracks. The tagged tracks play at their normalized level. If the fallback gain is 0 dB, switching between tagged and untagged tracks can be jarring if their loudness differs significantly.

The best practice for playlist management is to run ReplayGain scanning on all tracks before creating playlists, ensuring consistent loudness across the entire playlist.

---

## 11. REPLAYGAIN 2.0

### 11.1 Changes from ReplayGain 1.0

ReplayGain 2.0, developed circa 2010, made several important changes to address the limitations of v1.0:

#### 11.1.1 Reference Level Change

- **ReplayGain 1.0**: 89 dB SPL reference level (approximately -18 dBFS)
- **ReplayGain 2.0**: 95 dB SPL reference level (approximately -15 dBFS)

The 6 dB increase reflects the reality that commercial recordings had become significantly louder due to the loudness war. Using the v1.0 reference level would cause excessive attenuation for most modern music. The v2.0 reference level provides more appropriate normalization for contemporary content.

Note: A gain value computed with a v2.0 scanner is **not directly comparable** to a gain value computed with a v1.0 scanner. The v2.0 gain value is typically 6 dB smaller (less gain adjustment needed) than the v1.0 value for the same track.

#### 11.1.2 Filter Specification

ReplayGain 1.0 did not mandate specific filters. ReplayGain 2.0 specifies:
- **Pre-filter**: High-shelf at approximately 1682 Hz (+4 dB) — matching the K-weighting pre-filter from ITU-R BS.1770
- **High-pass filter**: RLB weighting at approximately 38 Hz — matching the K-weighting high-pass

These filter specifications bring ReplayGain 2.0's loudness measurement closer to ITU-R BS.1770-4, improving correlation with perceived loudness.

#### 11.1.3 True Peak Detection

ReplayGain 1.0 only stores the **sample peak** — the maximum absolute sample value. ReplayGain 2.0 requires **true peak** measurement using 4x oversampling, matching ITU-R BS.1770-4 true peak methodology.

A ReplayGain 2.0 tag's peak value can exceed 1.000000, indicating that the true peak was higher than the sample peak:

```
REPLAYGAIN_TRACK_PEAK=1.023456  # True peak exceeded sample peak by ~0.2 dB
```

Players can use this information to make better decisions about clipping prevention.

#### 11.1.4 New Fields: MINMAX

ReplayGain 2.0 adds `REPLAYGAIN_TRACK_MINMAX` and `REPLAYGAIN_ALBUM_MINMAX` fields:

```
REPLAYGAIN_TRACK_MINMAX=0.000000 0.987654
```

The first value is the minimum sample value (normally 0.000000 for symmetric audio), and the second is the maximum. These values allow players to compute the peak after arbitrary gain adjustments more precisely.

For example, if a player wants to apply +6 dB of gain, it can check: `0.987654 × 1.9953 = 1.9709 > 1.0`, indicating clipping would occur.

### 11.2 Backward Compatibility

#### 11.2.1 Tag Format Compatibility

ReplayGain 2.0 uses the same tag field names as v1.0 (`REPLAYGAIN_TRACK_GAIN`, `REPLAYGAIN_TRACK_PEAK`, etc.). This means v2.0 tags are readable by v1.0 players, though the v1.0 player will:
- Ignore the new MINMAX fields
- Use the true peak value (stored as sample peak) as a sample peak estimate

#### 11.2.2 Cross-Version Incompatibility

A v2.0 tag written to a file does **not** overwrite a v1.0 tag — both can coexist. Most tagging libraries write both when scanning, but some implementations may have specific rules about which version's tags take precedence.

The key incompatibility is in the **gain values**: a +3 dB gain from a v1.0 scanner is computed relative to a -18 dBFS reference, while a +3 dB gain from a v2.0 scanner is computed relative to a -15 dBFS reference. For the same input track, the v1.0 and v2.0 measurements would differ by approximately 3 dB.

#### 11.2.3 Reference Loudness Tag

To distinguish between v1.0 and v2.0 scans, implementations often write a `REPLAYGAIN_REFERENCE_LOUDNESS` tag:

```
REPLAYGAIN_REFERENCE_LOUDNESS=89 dB     # ReplayGain 1.0
REPLAYGAIN_REFERENCE_LOUDNESS=95 dB     # ReplayGain 2.0
```

Players can use this tag to determine whether the gain values are v1.0 or v2.0, and adjust accordingly.

---

## 12. IMPLEMENTATION CHECKLIST (for the Converter Developer)

### 12.1 Planning Phase

- [ ] Determine whether the converter will write ReplayGain tags, EBU R128 tags, or both
- [ ] For ReplayGain: decide on v1.0 or v2.0 reference level (v2.0 recommended for new implementations)
- [ ] For R128: verify the target platform's loudness specification (LUFS target, true peak limit, LRA limit)
- [ ] Determine whether peak values should be sample peak or true peak (true peak recommended)
- [ ] Plan the two-pass pipeline: measurement pass, then gain application pass
- [ ] Choose the output container format and verify its metadata tag support for ReplayGain/R128 fields

### 12.2 Measurement Implementation

- [ ] Read audio as PCM: decode to the native sample rate and bit depth of the source
- [ ] Convert to 32-bit float for processing
- [ ] For ReplayGain: implement 50ms RMS windows with 75% overlap and Hann windowing
- [ ] For EBU R128: implement K-weighting filter (high-shelf + RLB high-pass) per ITU-R BS.1770-4
- [ ] For EBU R128: implement gating (absolute gate at -70 LUFS, relative gate at -10 LU)
- [ ] Implement 4x oversampling for true peak detection (use windowed sinc or equivalent)
- [ ] Compute mean of squared RMS values → convert to dBFS → compute gain offset
- [ ] For album gain: concatenate all track PCM data and process as a single program

### 12.3 Gain Application Implementation

- [ ] Compute linear gain multiplier: `gain_linear = 10^(gain_dB / 20)`
- [ ] Apply gain to PCM: `output = input * gain_linear`
- [ ] Check for clipping: if `max_abs(output) > 1.0`, apply peak limiting or gain scaling
- [ ] Implement true peak limiting if required by the target specification (-1 dBTP for broadcast, etc.)
- [ ] Encode output to the target format (FLAC, AAC, MP3, etc.)
- [ ] Write ReplayGain and/or R128 tags to the output file's metadata container

### 12.4 Verification Checklist

- [ ] Use `kid3-cli` to verify all tag fields are written correctly (see workspace rule for verification steps)
- [ ] Verify no `Invalid UTF-8` errors in kid3 output
- [ ] Verify no `Ignoring duplicate atom` warnings
- [ ] Verify all standard tags match exactly (Title, Artist, Album, Date, Track, Genre, etc.)
- [ ] Verify every custom/unknown field from source appears in destination with the exact same name and value
- [ ] Verify Sort atoms appear with their kid3 labels (Sort Album, etc.) not as raw freeform names
- [ ] Verify numeric IDs (Artist ID, Album ID) are present if source had them
- [ ] Verify Publisher (©pub) is present if source had it
- [ ] Verify cover art (Picture) is present and shows `[JPEG/PNG, N bytes]`
- [ ] Verify ReplayGain fields use correct format: `+XdB` for gain, `X.XXXXXX` for peak
- [ ] Verify R128 fields: integrated loudness in LUFS, LRA in LU, true peak in dBTP
- [ ] Measure the output loudness with FFmpeg loudnorm to verify it matches the target
- [ ] Check the output true peak with a true peak meter to verify it is within the platform limit

### 12.5 FFmpeg loudnorm Integration Reference

The recommended approach for a converter is to use FFmpeg's loudnorm filter for both measurement and normalization, as it implements the complete ITU-R BS.1770-4 algorithm:

```python
import subprocess
import json

def measure_and_normalize(input_path, output_path, target_lufs=-16, max_tp=-1.0):
    # Pass 1: Measure
    measure_cmd = [
        'ffmpeg', '-i', input_path,
        '-af', f'loudnorm=I={target_lufs}:TP={max_tp}:print_format=json',
        '-f', 'null', '-'
    ]
    result = subprocess.run(measure_cmd, capture_output=True, text=True)
    # Parse JSON output from stderr
    # ...

    # Pass 2: Normalize using measured values
    normalize_cmd = [
        'ffmpeg', '-i', input_path,
        '-af', f'loudnorm=I={target_lufs}:TP={max_tp}:'
               f'measured_I={measured_i}:measured_TP={measured_tp}:'
               f'measured_LRA={measured_lra}:measured_thresh={measured_thresh}:'
               f'linear=false:print_format=summary',
        '-c:a', 'flac', '-compression_level', '8',
        output_path
    ]
    subprocess.run(normalize_cmd, capture_output=True)
```

### 12.6 Metadata Tag Writing Reference by Format

| Format | Container | Tag System | ReplayGain Fields | R128 Fields |
|---|---|---|---|---|
| FLAC | Native | Vorbis Comments | `REPLAYGAIN_TRACK_GAIN` | `LOUDNESS`, `LOUDNESS_RANGE`, `TRUE_PEAK` |
| Ogg Vorbis | OGG | Vorbis Comments | `REPLAYGAIN_TRACK_GAIN` | `LOUDNESS`, `LOUDNESS_RANGE`, `TRUE_PEAK` |
| MP3 | MPEG Audio | ID3v2 TXXX | `REPLAYGAIN_TRACK_GAIN` (in TXXX desc) | Not standardized |
| AAC | MP4 | iTunes Atoms (----) | `REPLAYGAIN_TRACK_GAIN` (freeform) | Varies by encoder |
| ALAC | MP4 | iTunes Atoms (----) | `REPLAYGAIN_TRACK_GAIN` (freeform) | Varies by encoder |
| WAV | RIFF | INFO tags | Limited support | Not standardized |
| WMA | ASF | ASF metadata | `REPLAYGAIN_TRACK_GAIN` | Not standardized |
| MusePack | MPC | APEv2 | `REPLAYGAIN_TRACK_GAIN` | Not standardized |

---

## APPENDIX A: EQUATION REFERENCE

### Loudness in dBFS from RMS

```
loudness_dBFS = 10 × log10((1/N) × Σ(s(n)²) for n in window)
```

### Gain calculation (ReplayGain)

```
gain_dB = reference_level_dBFS - loudness_dBFS
gain_linear = 10^(gain_dB / 20)
```

### Gain calculation (EBU R128)

```
gain_dB = target_LUFS - measured_LUFS
gain_linear = 10^(gain_dB / 20)
```

### True peak in dBTP

```
true_peak_dBTP = 20 × log10(true_peak_linear)
```

### K-weighting pre-filter transfer function

```
H_pre(s) = (s + 2π × 1681.97) / (s + 2π × 841.97)
```

### K-weighting RLB filter transfer function

```
H_hp(s) = s² / (s² + (2π × 38.13547) × s + (2π × 38.13547)²)
```

### Loudness Range calculation

```
LRA = L_95 - L_10
```

where L_95 is the 95th percentile and L_10 is the 10th percentile of the gated block loudness values.

---

## APPENDIX B: GLOSSARY

| Term | Definition |
|---|---|
| **dBFS** | Decibels relative to Full Scale. Measures digital signal level. 0 dBFS is the maximum representable level. |
| **dB SPL** | Decibels Sound Pressure Level. Measures acoustic loudness in the physical world. |
| **dBTP** | Decibels relative to True Peak. Used for true peak measurement, where 0 dBTP = 0 dBFS sample peak. |
| **LUFS** | Loudness Units relative to Full Scale. The unit of absolute weighted loudness (ITU-R BS.1770). |
| **LU** | Loudness Units. A relative unit for loudness differences. 1 LU ≈ 1 dB. |
| **LRA** | Loudness Range. The dynamic range of a program, measured as the difference between 95th and 10th percentile loudness. |
| **RMS** | Root Mean Square. The square root of the mean of squared values. Used to compute the effective amplitude of a signal. |
| **True Peak** | The maximum peak level of the analog reconstruction of a digital signal, including inter-sample peaks. Detected via oversampling. |
| **Sample Peak** | The maximum absolute sample value in the digital representation. Does not capture inter-sample peaks. |
| **K-weighting** | The frequency weighting filter defined in ITU-R BS.1770-4: high-shelf (+4 dB at 1682 Hz) + RLB high-pass (38 Hz). |
| **Gating** | The process of excluding silence/near-silence blocks from loudness measurement. Absolute gate at -70 LUFS, relative gate at -10 LU. |
| **ReplayGain** | A community-developed volume normalization system that measures and stores loudness adjustment values in file metadata. |
| **EBU R128** | A formal loudness standard from the European Broadcasting Union, based on ITU-R BS.1770-4. |
| **Inter-sample peak** | A peak in the analog waveform that occurs between two digital sample points. Not captured by sample-peak measurement. |
| **Preamp** | A user-adjustable gain offset applied in addition to ReplayGain gain. Allows per-listener preference tuning. |
| **IEC 60268-1** | International standard for sound system equipment. Referenced in early ReplayGain documentation. |
| **ITU-R BS.1770-4** | ITU Radiocommunication Sector recommendation defining the algorithm for measuring audio loudness. |
