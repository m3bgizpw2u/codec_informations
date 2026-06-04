# DSP Effects Chain Order and Behavior
*Generated: 2026-06-04 | Sources: 8 | Confidence: High*

## Executive Summary

DBpoweramp's DSP (Digital Signal Processing) system processes audio through a configurable chain of effects applied in top-to-bottom order. The critical insight is that the order of effects significantly impacts audio quality. The recommended chain for maximum quality is: 1) Bit Depth to 32-bit Float (no dither), 2) Sample Rate Conversion, 3) DSP Effects (EQ, normalization, etc.), 4) Bit Depth to target (with dither). This order ensures maximum processing precision before final output. The resampling algorithm uses ARDFTSRC, and dithering employs Triangular PDF (TPDF) for 16-bit output.

## 1. DSP System Overview

### 1.1 What are DSP Effects?
DSP effects in audio conversion modify the audio signal in real-time during encoding. Unlike metadata tags, DSP effects actually alter the audio data.

### 1.2 DBpoweramp DSP Capabilities
DBpoweramp includes over 20 DSP effects:
- Volume normalization (peak, RMS, EBU R128)
- Graphic/parametric equalizer
- Sample rate conversion (resampling)
- Bit depth conversion (with dithering)
- Channel remix/upmix/downmix
- DC offset removal
- Compression/limiting
- Low-pass/high-pass filters
- ReplayGain calculation
- Copy/delete source file
- Multi-CPU force
- And more...

### 1.3 Order is Critical
DSP effects are processed **strictly in the order listed**, from top to bottom. Reordering effects can produce dramatically different results.

## 2. Complete DSP Effect Order

### 2.1 Default Processing Pipeline
The typical processing sequence:

```
Input: Raw source audio (any format → decoded to PCM)
│
├─► 1. Channel Remix/Upmix/Downmix
│
├─► 2. DC Offset Removal
│
├─► 3. Pre-Gain (volume adjustment before processing)
│
├─► 4. ReplayGain Pre-Gain (if applying gain during encode)
│
├─► 5. DSP Effects (user-configurable):
│    ├─► Graphic/Parametric Equalizer
│    ├─► Volume Normalize (Peak/RMS/EBU R128)
│    ├─► Compressor/Limiter
│    ├─► Low-pass/High-pass Filters
│    └─► Other effects
│
├─► 6. Sample Rate Conversion (if needed)
│    └─► Algorithm: ARDFTSRC (bandlimited sinc)
│
├─► 7. Bit Depth Conversion
│    └─► Noise shaping (for reduction)
│    └─► Dithering (TPDF for 16-bit)
│
└─► Output: PCM at target format → Encoder → Output file
```

### 2.2 Input: Decoded PCM
Source audio is decoded to PCM (float32) before any DSP processing:
- FLAC → PCM
- WAV → PCM
- MP3 → PCM
- AAC → PCM
- All formats unified to common internal representation

### 2.3 Floating Point Headroom
When processing at 32-bit float:
- Values can exceed ±1.0 (digital full scale)
- Allows processing without clipping
- Final stage brings values back to target range

## 3. Bit Depth to Float (First)

### 3.1 Purpose
Convert to 32-bit floating point before any DSP:
- Provides maximum precision for calculations
- Prevents accumulation of quantization errors
- Allows headroom above 0 dBFS

### 3.2 Configuration
Bit Depth DSP effect settings:
- **Convert to**: 32-bit Floating Point
- **Dither**: None (no dither needed at this stage)
- **Purpose**: Internal processing only

### 3.3 Why First?
Converting to float BEFORE other effects:
- Equalizer calculations more precise
- Volume normalization more accurate
- Prevents multiple integer rounding errors

## 4. Channel Remix/Upmix/Downmix

### 4.1 Channel Remix
Remap channels to different configurations:
- Stereo to Mono
- 5.1 to Stereo
- Custom channel routing

### 4.2 Upmix/Downmix
Convert between channel configurations:
- **Upmix**: 2.0 → 5.1, 2.0 → 7.1
- **Downmix**: 5.1 → 2.0, 7.1 → 5.1

### 4.3 Placement
Channel processing should occur:
- Early in chain (before EQ)
- After float conversion
- Before volume normalization

## 5. DC Offset Removal

### 5.1 What is DC Offset?
DC offset is a constant (non-zero) average amplitude in audio:
- Causes low-frequency rumble
- Reduces usable dynamic range
- Can affect loudness measurements

### 5.2 Removal Algorithm
High-pass filter at very low frequency:
- Typical cutoff: 20-30 Hz
- Preserves musical content
- Removes only DC component

### 5.3 When to Apply
DC offset removal should occur:
- Early in processing chain
- Before volume normalization
- After channel processing

## 6. Volume Normalization

### 6.1 Peak to Peak
Simple peak normalization:
1. Find maximum absolute sample value
2. Calculate scaling factor to reach target
3. Apply uniform gain to all samples
4. Can be set to not exceed target (Reduce if Above)

### 6.2 RMS Normalization
Loudness-based normalization:
1. Calculate RMS power of entire file
2. Find scaling factor for target RMS
3. Apply uniform gain

### 6.3 EBU R128 Mode
BS.1770-compliant normalization:
1. Measure integrated loudness (LUFS)
2. Calculate gain to reach -23 LUFS
3. Apply with true peak limiting

### 6.4 Volume Normalize Options
| Option | Behavior |
|--------|----------|
| Peak to Peak | Scale to touch target, never exceed |
| Reduce if Above | Scale down only, never amplify |
| Maximum Amplification | Limit max gain applied |
| Fixed Amplification | Apply exact dB adjustment |

## 7. Sample Rate Conversion

### 7.1 When Applied
SRC typically occurs:
- After DSP effects
- Before final bit depth conversion
- Only if output rate differs from source

### 7.2 DBpoweramp's SRC: ARDFTSRC
DBpoweramp uses the ARDFTSRC resampling library:
- Based on bandlimited sinc interpolation
- High-quality rational and irrational ratio conversion
- Also used in foobar2000
- Available at hydrogenaudio.org test suite

### 7.3 Algorithm Details
ARDFTSRC characteristics:
- **Type**: Bandlimited interpolation
- **Quality**: Very high (comparable to SoX)
- **Pre/post ringing**: Linear phase
- **Stopband attenuation**: >180 dB

### 7.4 Comparison with Other Resamplers
| Resampler | Quality | Latency | Notes |
|-----------|---------|---------|-------|
| ARDFTSRC | Very High | Medium | DBpoweramp/foobar2000 |
| SoX | High | Variable | VHQ mode very high |
| Secret Rabbit Code | High | Medium | Good all-rounder |
| Speex | Low-Medium | Low | Real-time optimized |
| r8brain | Very High | Medium | Commercial quality |

### 7.5 SRC Quality Testing
Hydrogenaudio maintains comparison results:
- Test suite at src.hydrogenaudio.org
- Shows frequency response, aliasing, impulse response
- ARDFTSRC performs at top tier

## 8. Bit Depth Conversion and Dithering

### 8.1 Why Dither?
When reducing bit depth:
- 24-bit → 16-bit
- 32-bit float → 16-bit integer
- Quantization introduces distortion
- Dithering linearizes quantization

### 8.2 TPDF Dither
Triangular Probability Density Function dither:
- Generated by summing two RPDF sources
- 2 LSB amplitude range
- Eliminates harmonic distortion
- Adds noise floor without distortion

### 8.3 DBpoweramp Dither Options
Available dithering options:
- **None**: For reduction to 24-bit or higher
- **Rectangular**: Basic, less preferred
- **Triangular (TPDF)**: Recommended for 16-bit
- **Noise Shaping**: Advanced profiles

### 8.4 Noise Shaping Profiles
Advanced dithering can include noise shaping:
- **F-weighted**: Frequency-weighted noise
- **Minimum phase**: Reduced pre-ringing
- **Various profiles**: Different spectral shapes

### 8.5 When to Apply Dither
**Apply dither ONLY when reducing to ≤20 bits**:
- 24-bit → 16-bit: Apply TPDF dither
- 24-bit → 24-bit: No dither needed
- 32-bit float → 32-bit float: No dither needed
- 32-bit float → 16-bit: Apply TPDF dither

## 9. Complete Recommended Order

### 9.1 Optimal DSP Chain
For highest quality output:

```
1. Bit Depth → 32-bit Float (No Dither)
   └─► Purpose: Internal processing precision

2. Resample → [Target Rate] (e.g., 48000 Hz)
   └─► Note: Some prefer resample BEFORE other effects
   └─► Reason: EQ and normalize at native rate first

3. Graphic Equalizer → [Settings]
   └─► Note: EQ at high precision

4. Volume Normalize → Peak to Peak -1 dB
   └─► Purpose: Prevent clipping from other effects

5. Bit Depth → 16-bit (Apply Dither: Triangular)
   └─► Purpose: Final output with proper dithering
```

### 9.2 Alternative Order (Resample First)
Some users prefer:

```
1. Bit Depth → 32-bit Float
2. Resample → Target Rate
3. [Other DSP Effects]
4. Bit Depth → 16-bit (Dither)
```

### 9.3 Forum Consensus
From dBpoweramp forum discussions:
- "The first effect to float allows the values to go above clipping (+1)"
- "Volume normalize would bring that back under"
- Effects are processed in listed order

## 10. Edge Cases

### 10.1 Clipping During Processing
When DSP effects exceed 0 dBFS:
- Float processing allows headroom
- Volume normalize can bring back
- Check for clipping before output

### 10.2 Multiple Volume Effects
Applying multiple volume/normalization effects:
- Order affects final result
- Duplicate processing wastes CPU
- Choose single best method

### 10.3 Resampling Before vs After EQ
Resample first vs resample last:
- **Resample first**: EQ at target rate
- **Resample last**: EQ at source rate
- Both valid, slight quality differences
- Resample last often preferred for precision

### 10.4 DSD to PCM Conversion
DSD (Direct Stream Digital):
- Must convert to PCM first
- Apply DSP at PCM stage
- Then convert to target format

### 10.5 Very Low Sample Rates
Processing at 96 kHz or higher:
- More processing overhead
- Longer filter tails
- May exceed EQ filter design limits

### 10.6 Multi-Channel to Stereo Downmix
When downmixing surround to stereo:
- Apply appropriate matrix
- Check for phase issues
- Consider LFE channel handling

## 11. Would a User Notice a Difference?

### From Different DSP Orders

| Order Change | Audible Impact |
|--------------|-----------------|
| Float vs Integer | Subtle (integer may clip) |
| Dither vs None | Subtle at 16-bit |
| EQ before vs after normalize | Significant |
| SRC algorithm | Varies by quality level |

### From Different Resamplers

| Resampler | Quality Level | Potential Audible |
|-----------|---------------|-------------------|
| ARDFTSRC | Very High | None (very high quality) |
| SoX VHQ | Very High | None |
| Speex | Low-Medium | Potential artifacts |
| None (builtin) | Unknown | May vary |

### From Applying vs Not Applying DSP

| Effect | Audible Impact |
|--------|-----------------|
| Volume normalize | Significant (loudness change) |
| EQ | Significant (frequency response) |
| Dither | Minimal (noise floor) |
| DC removal | Minimal (unless DC present) |

## Sources

1. [dBpoweramp DSP Help](https://dbpoweramp.com/Help/dMC/dsp)
2. [dBpoweramp Music Converter Help](https://dbpoweramp.com/Help/dMC/dMC)
3. [Dithering - dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/44570-dithering-about-dithering-and-what-order-when-changing-sample-rate-bits-p-sample)
4. [Resampling - dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/batch-converter/44854-resampling)
5. [DSP Effect Order - dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/batch-converter/330321-dsp-effect-order-advice)
6. [SRC Comparison - Hydrogenaudio](https://src.hydrogenaudio.org/compareresults?id1=0a5d115f-a928-459d-844f-fdb3b7b6501f&id2=0)
7. [SoX Resampling](https://web.archive.org/web/20150420140757/sox.sourceforge.net/SoX/Resampling)
8. [Dither - Wikipedia](https://en.wikipedia.org/wiki/Dither)
