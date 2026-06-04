# ReplayGain Scanning Pipeline (EBU R128)
*Generated: 2026-06-04 | Sources: 10 | Confidence: High*

## Executive Summary

ReplayGain is a perceived loudness measurement standard that calculates volume adjustment values stored as metadata tags, allowing media players to normalize playback loudness. DBpoweramp implements ReplayGain scanning as a DSP effect that decodes audio to PCM, applies the specified loudness algorithm, and writes gain/peak values to output files. The software supports both the original ReplayGain 1.0 specification (95th percentile, -14 dB reference) and the newer EBU R128 standard (ITU-R BS.1770, -23 LUFS target). PerfectTUNES offers advanced ReplayGain with 4x oversampling for true peak detection.

## 1. ReplayGain Overview

### 1.1 What is ReplayGain?
ReplayGain is a technical standard proposed by David Robinson in 2001 for measuring and normalizing perceived loudness of digital audio files. Key principles:
- Measures perceived loudness, not just amplitude
- Stores adjustment values as metadata tags
- Players apply adjustment during playback
- Preserves original audio data (non-destructive)

### 1.2 Versions
| Version | Year | Standard | Reference Level |
|---------|------|----------|-----------------|
| ReplayGain 1.0 | 2001 | Proprietary | -14 dB (89 dB SPL) |
| ReplayGain 2.0 | 2011 | EBU R128/ITU BS.1770 | -18 LUFS |

ReplayGain 2.0 aligns with broadcast standards while maintaining backward compatibility with RG 1.0 tags.

## 2. ReplayGain 1.0 Algorithm

### 2.1 Step-by-Step Algorithm
The original ReplayGain algorithm:

1. **Pre-filter**: Apply equal-loudness contour filter
   - 10th-order IIR Yulewalk filter
   - 2nd-order Butterworth high-pass at 150 Hz
   - Simulates human frequency sensitivity

2. **Window RMS Calculation**: For each 50ms block:
   ```
   RMS[i] = sqrt(sum(samples²) / window_size)
   ```

3. **Statistical Processing**:
   - Sort all RMS values ascending
   - Select 95th percentile value
   - This represents overall perceived loudness

4. **Calibration**:
   - Reference: Pink noise file calibrated to 89 dB SPL
   - Stored reference level: -31.49 dB (from pink.wav)

5. **Gain Calculation**:
   ```
   ReplayGain = reference_level - measured_loudness
   ```

### 2.2 Pre-emphasis Filter Coefficients
The equal-loudness filter simulates Fletcher-Munson contours:
- Combined high-pass + 10th-order Yulewalk
- Frequency-dependent amplitude adjustment
- Compensates for human ear sensitivity

### 2.3 Why 95th Percentile?
From David Robinson's research:
> "For highly compressed pop music, the choice makes little difference. For speech and classical music, the choice makes a huge difference. The value which most accurately matches human perception of perceived loudness is around 95%."

### 2.4 Peak Amplitude Detection
ReplayGain stores peak values:
- Maximum absolute sample value per channel
- Stored as floating-point (1.0 = digital full scale)
- Used to prevent clipping during playback

## 3. EBU R128 / ITU-R BS.1770 Algorithm

### 3.1 Overview
EBU R128 is the European Broadcasting Union standard for audio loudness measurement, using the ITU-R BS.1770 algorithm.

### 3.2 Key Parameters
| Parameter | Value |
|-----------|-------|
| Target Loudness | -23 LUFS |
| Short-term Max | -18 LUFS |
| True Peak Max | -1 dBTP |
| Gating Block | 400ms (75% overlap) |
| Absolute Gate | -70 LKFS |
| Relative Gate | -10 dB from ungated |

### 3.3 Algorithm Steps
1. **K-weighting Filter**:
   - High-pass at ~38 Hz (pre-filter)
   - High-shelf at ~1681 Hz (+4 dB)
   - Applied per channel

2. **Block Gating** (400ms blocks, 75% overlap):
   - First gate: -70 LKFS absolute threshold
   - Second gate: -10 dB relative to ungated level

3. **Loudness Calculation**:
   ```
   Integrated Loudness = mean(loudness of ungated blocks)
   ```

4. **True Peak Detection**:
   - Oversampled peak detection (4x minimum)
   - Detects inter-sample peaks

### 3.4 LUFS vs dB
| Unit | Description |
|------|-------------|
| LUFS | Loudness Units relative to Full Scale |
| LKFS | Loudness K-weighted relative to Full Scale |
| LU | Loudness Unit (relative, 1 LU = 1 dB) |

LUFS/LKFS are equivalent in practice.

## 4. DBpoweramp ReplayGain Implementation

### 4.1 DSP Effect Integration
In dBpoweramp's DSP chain:
1. Decode source audio to PCM
2. Apply ReplayGain/R128 algorithm
3. Store calculated values in memory
4. Write to output file metadata

### 4.2 Tag Storage Format
ReplayGain tags written to output files:
| Tag | Description |
|-----|-------------|
| `REPLAYGAIN_REFERENCE_LOUDNESS` | Reference level (e.g., -14 dB or -18 LUFS) |
| `REPLAYGAIN_TRACK_GAIN` | Volume adjustment for track (dB) |
| `REPLAYGAIN_TRACK_PEAK` | Track peak amplitude |
| `REPLAYGAIN_ALBUM_GAIN` | Album-level adjustment (dB) |
| `REPLAYGAIN_ALBUM_PEAK` | Album peak amplitude |

### 4.3 Supported Formats
ReplayGain tags supported in:
- FLAC (Vorbis comments)
- Ogg Vorbis (Vorbis comments)
- MP3 (ID3v2 / LAME tag)
- AAC (iTunes-style tags)
- Others (format-dependent)

### 4.4 PerfectTUNES ReplayGain
PerfectTUNES offers enhanced ReplayGain:
- EBU R128 algorithm option
- 4x oversampling for true peak
- Batch processing entire libraries
- Album or track mode

## 5. Album vs Track Scanning

### 5.1 Track Mode
Each track analyzed independently:
- Individual loudness measurement
- Individual gain value per track
- Preserves relative loudness between tracks
- Album dynamics intact

### 5.2 Album Mode
All tracks analyzed together:
1. Scan all tracks in album
2. Calculate loudness for each track
3. Find highest 95th percentile (RG 1.0) or integrated (R128)
4. Apply same gain to all tracks
5. Preserves album dynamics

### 5.3 Album Gain Calculation
For ReplayGain 1.0:
- Track RMS values combined, sorted
- 95th percentile of combined distribution
- Single gain applied to all tracks

For EBU R128:
- Weighted average of track loudness
- Minimum block requirement per track
- Integrated loudness of album

## 6. Batch Album Grouping

### 6.1 How Files are Grouped
DBpoweramp groups files for album mode by:
- **Folder structure**: Files in same folder
- **Album tag**: Files with matching album name
- **Manual selection**: User-selected tracks

### 6.2 Folder-Based Grouping
Default behavior:
1. Read first file's album tag
2. Find all files in same folder
3. Scan as single album
4. Apply album gain to all

### 6.3 Multi-Disc Albums
For box sets and multi-disc albums:
- Each disc typically separate folder
- Cross-disc album gain requires manual selection
- Disc number tracked via `[disc]` tag

### 6.4 Compilation Albums
Special handling for compilations:
- Various Artists grouping
- May require separate album gain
- User intervention may be needed

## 7. True Peak vs Sample Peak

### 7.1 Sample Peak
Simple maximum absolute sample value:
```
Peak = max(|samples|)
```
Does not detect inter-sample peaks.

### 7.2 True Peak (TP)
Oversampled peak detection:
- Resamples signal at higher rate (4x-8x)
- Detects peaks between samples
- More accurate for DAC reconstruction

### 7.3 DBpoweramp/PerfectTUNES True Peak
PerfectTUNES implements true peak detection:
- 4x oversampling minimum
- ITU-R BS.1770 compliant
- Detects peaks that would clip in playback

### 7.4 Peak Values in Tags
| Type | Typical Value | Detection |
|------|---------------|-----------|
| Sample Peak | 0.95 | Direct sample |
| True Peak | 0.96-0.98 | Oversampled |

True peak values are typically 0.5-1.5 dB higher.

## 8. Reference Loudness Levels

### 8.1 ReplayGain 1.0 Reference
- **Digital Reference**: -14 dB (= 89 dB SPL on SMPTE RP 200)
- **Purpose**: Loud albums normalized to average
- **Typical Results**: -6 to +12 dB adjustment

### 8.2 EBU R128 Reference
- **Target Loudness**: -23 LUFS
- **Purpose**: Broadcast standardization
- **Typical Results**: -9 to +6 dB adjustment

### 8.3 ReplayGain 2.0 (Hybrid)
- **Target Loudness**: -18 LUFS
- **Purpose**: Compatibility with RG 1.0 players
- **Reference**: -18 dB (closer to RG 1.0)

### 8.4 Reference Level Differences
| Standard | Target LUFS | dB Reference |
|----------|-------------|--------------|
| ReplayGain 1.0 | ~89 dB SPL | -14 dB |
| EBU R128 | -23 LUFS | N/A |
| RG 2.0 | -18 LUFS | -18 dB |

## 9. Implementation Details

### 9.1 Processing Pipeline
```
1. Decode Source → PCM (float32)
2. Resample to 48 kHz (if needed)
3. Apply Equal-Loudness Filter
4. Calculate RMS per 50ms window
5. Sort and find 95th percentile
6. Calculate gain = reference - loudness
7. Find peak amplitude
8. Write tags to output file
```

### 9.2 libebur128 Reference Implementation
libebur128 is the reference open-source EBU R128 implementation:
- Portable ANSI C
- Supports M (momentary), S (short-term), I (integrated) modes
- Loudness range (LRA) measurement
- True peak scanning
- Used by FFmpeg, Audacity, and others

### 9.3 Essentia Reference Implementation
The MTG/essentia library provides ReplayGain 1.0:
- Equal-loudness filtering
- 50ms RMS windows
- 95th percentile selection
- Reference level from pink.wav

## 10. Edge Cases

### 10.1 Very Short Tracks
- Minimum length: 50ms (one window)
- Shorter tracks return silence value
- May skew album calculations

### 10.2 Silent or Near-Silent Tracks
- Silence contributes 0 to calculation
- Can affect album gain significantly
- Manual exclusion may be needed

### 10.3 Live/Concert Recordings
- Audience noise affects loudness
- 95th percentile may be too aggressive
- R128 may work better

### 10.4 Classical/Symphonic Music
- Large dynamic range
- Peak/RMS relationship varies
- Album mode recommended

### 10.5 Heavily Compressed Music
- Modern pop/EDM highly compressed
- 95th percentile close to maximum
- Gain adjustments minimal

### 10.6 Tracks with DCO (DC Offset)
- DC offset affects RMS calculation
- High-pass filter helps
- May need manual DC removal

### 10.7 Mixed Content Albums
- Talk + music albums
- Classical with applause
- Different genres in compilation
- Album mode may not suit all tracks

### 10.8 Multi-Channel Audio
- ReplayGain 1.0: Stereo primarily
- EBU R128: Full surround support
- Channel weighting for LFE

## 11. Would a User Notice a Difference?

### From DBpoweramp ReplayGain vs No ReplayGain

**Audible difference during playback**:
- Consistent volume across albums
- No manual volume adjustment needed
- Dynamic range preserved

**No difference in audio quality**:
- Original files unchanged
- Gain applied by player, not during conversion

### From ReplayGain 1.0 vs R128

| Aspect | RG 1.0 | R128 |
|--------|---------|------|
| Target Level | -14 dB | -23 LUFS |
| Algorithm | 95th percentile | Integrated |
| Player Support | Wide | Growing |
| Broadcast Use | No | Yes |

For most listeners: **Minimal perceived difference** if same source material.

### From True Peak vs Sample Peak

**No audible difference in files**:
- Peak stored only as tag
- Player uses for clipping prevention

**Potential audible difference**:
- If true peak > sample peak
- Player prevents clipping
- Slightly lower playback level

## Sources

1. [ReplayGain 1.0 Specification - Hydrogenaudio](https://wiki.hydrogenaudio.org/index.php/ReplayGain_1.0_specification)
2. [ReplayGain - Wikipedia](https://en.wikipedia.org/wiki/ReplayGain)
3. [ReplayGain - Grokipedia](https://grokipedia.com/page/ReplayGain)
4. [EBU R128 - Wikipedia](https://en.wikipedia.org/wiki/EBU_R_128)
5. [ITU-R BS.1770-5 (2023)](https://www.itu.int/dms_pubrec/itu-r/rec/bs/R-REC-BS.1770-5-202311-I!!PDF-E.pdf)
6. [EBU Tech 3341 - R128](https://tech.ebu.ch/docs/r/r128s1.pdf)
7. [libebur128 - GitHub](https://github.com/jiixyj/libebur128)
8. [Essentia ReplayGain - GitHub](https://github.com/MTG/essentia/blob/master/src/algorithms/standard/replaygain.cpp)
9. [dBpoweramp DSP Help](https://dbpoweramp.com/Help/dMC/dsp)
10. [PerfectTUNES](https://www.dbpoweramp.com/perfecttunes)
