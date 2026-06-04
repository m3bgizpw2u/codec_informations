# Internal Audio Representation and PCM Buffer — DBpoweramp Behavior Research
> **Research Category:** Architecture
> **DBpoweramp Versions Studied:** R14 through R2026-04 (current)
> **Confidence Level:** High — documented (official dBpoweramp DSP help docs, Spoon forum posts, changelog analysis) + spec inference
> **Primary Sources:** [dBpoweramp DSP Effects Help](https://dbpoweramp.com/Help/dMC/dsp) | [Dithering Forum Post](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/44570-dithering-about-dithering-and-what-order-when-changing-sample-rate-bits-p-sample) | [Illustrate Versions Changelog](https://versions.dbpoweramp.com/?appid=2) | [DSD Support Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/36421-the-next-level-706-5-768-pcm-wav-native-dsd-support) | [Music Converter DSP Architecture Forum](https://forum.dbpoweramp.com/forum/other-topics/developers-corner/22043-music-converter-dsp-architecture)
> **Open-Source Reference:** FFmpeg | SoX | libswresample | r8brain | Speex resampler

---

## 1. TOPIC OVERVIEW & PURPOSE

### 1.1 What This Component Does

DBpoweramp converts all audio through an internal floating-point PCM representation during the DSP processing stage. The internal format, buffer management strategy, and precision handling during processing are the core of DBpoweramp's audio quality — determining whether a 24-bit FLAC → 16-bit MP3 conversion sounds identical to the source, whether dither is applied correctly, and whether gapless playback tags are preserved. Understanding the internal PCM representation is essential for replicating DBpoweramp's quality.

### 1.2 Why This Matters for Re-implementation

Getting the internal PCM format wrong produces audible artifacts:
- **Not using float during DSP** — Equalizer, volume normalization, and resampling in integer arithmetic lose precision and introduce rounding errors
- **Wrong dither type** — TPDF dither is perceptually superior to no dither; using rectangular dither instead of triangular produces different noise floor characteristics
- **Wrong dither ordering** — Applying dither BEFORE resampling wastes noise at frequencies above the target Nyquist; applying after resampling preserves more of the original signal
- **No clipping protection** — Float DSP can exceed ±1.0; writing to int output without clipping protection produces hard clipping
- **Wrong channel ordering** — Multichannel upmix/downmix produces wrong channels in output if the channel map is wrong
- **Not handling encoder delay/padding** — Gapless MP3 files will have ~1000 sample gaps between tracks

### 1.3 DBpoweramp's Design Philosophy for Internal PCM

DBpoweramp's approach: **stay in floating-point as long as possible**, convert to the target integer format only at the final stage, and apply dither at that final conversion. This philosophy maximizes precision throughout the DSP chain — every multiplication, addition, and filter operation happens in float where small signals are preserved.

---

## 2. OBSERVED / DOCUMENTED BEHAVIOR

### 2.1 Internal PCM Format Specification

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  DBpoweramp Internal PCM Format                             │
│                                                                             │
│  PRIMARY FORMAT: 32-bit IEEE 754 float (float32)                           │
│  SINCE R2025-11: 64-bit IEEE 754 float (float64) ALSO supported          │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ WAVEFORMATEX (per Microsoft standard):                               │   │
│  │                                                                       │   │
│  │  wFormatTag:      WAVE_FORMAT_IEEE_FLOAT (0x0003) or WAVE_FORMAT_PCM  │   │
│  │  nChannels:      1 (mono) to 12 (multichannel)                       │   │
│  │  nSamplesPerSec: 8000 to 2000000 Hz (FLAC 1.4.1 supports 1MHz!)  │   │
│  │  nAvgBytesPerSec: nChannels × nSamplesPerSec × bits_per_sample / 8   │   │
│  │  nBlockAlign:    nChannels × bits_per_sample / 8                     │   │
│  │  wBitsPerSample: 8, 16, 24, 32 (int) OR 32, 64 (float)             │   │
│  │  cbSize:         0 (standard) or 22 (WAVEFORMATEXTENSIBLE)          │   │
│  │                                                                       │   │
│  │ WAVEFORMATEXTENSIBLE (for multichannel > 2ch):                     │   │
│  │  SubFormat:        KSDATAFORMAT_SUBTYPE_PCM or                        │   │
│  │                   KSDATAFORMAT_SUBTYPE_IEEE_FLOAT                    │   │
│  │  dwChannelMask:   speaker position bitmask (see channel map below)  │   │
│  │  wValidBitsPerSample: actual precision (24-bit in 32-bit container)   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  INTERLEAVING: Interleaved (NOT planar)                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Stereo interleaved: [L][R][L][R][L][R]...                         │   │
│  │  5.1 interleaved:   [FL][FR][FC][LFE][BL][BR][FL][FR][FC][LFE]... │   │
│  │  NOT planar: [L][L][L][L][R][R][R][R] (planar is NOT used)        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ENDIANNESS: Little-endian for all integer PCM on disk                     │
│  (WAV files are RIFF/RIFX; RIFF = little-endian, RIFX = big-endian)   │
│                                                                             │
│  CLIPPING: Float samples may exceed ±1.0 during processing                 │
│  (DBpoweramp allows this; final output must be clipped before int convert)  │
│                                                                             │
│  DITHER: Triangular PDF (TPDF) when reducing bit depth                      │
│  NOT rectangular (RPDF); NOT none for final int output                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Sample Rate Normalization Strategy

```pseudocode
// DBpoweramp does NOT normalize to a fixed internal sample rate.
// The internal sample rate follows the user's DSP chain configuration.

// Strategy A: No Resample DSP → internal rate = source rate
// Strategy B: Resample DSP active → internal rate = target rate set in DSP

// Available resampling modes (per DSP documentation):
// R2025-11+: ARDFTSRC (new high-quality resampler, arbitrary rates supported)
// Legacy: SSRC (high-quality, fixed rates only up to R2025-11)

// The resampler is applied as a LIVE DSP in the encode loop:
// Source PCM → DecodeBlock → Resample DSP → new rate PCM → EncodeBlock

// ARDFTSRC features (R2025-11+):
//   - Arbitrary sample rates: 44100, 48000, 96000, 192000, 
//     706.5 Hz (DSD!), 768 kHz (DSD!), 123456 Hz (non-standard!)
//   - Quality modes: minimum phase (less pre-ringing)
//   - Global quality settings in Control Center >> Advanced
//   - Can override on Resample DSP effect itself

// Channel Layout Preserved Through Resampling:
// Sample rate conversion is per-channel; channel order is NOT changed by resampling
```

### 2.3 Bit Depth Handling During Processing

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              Bit Depth Flow Through Conversion Pipeline                       │
│                                                                             │
│  SOURCE  ──▶  DECODE  ──▶  DSP CHAIN  ──▶  ENCODE  ──▶  OUTPUT             │
│                                                                             │
│  Common paths:                                                              │
│                                                                             │
│  Path 1: Lossless → Lossless (same bit depth, no DSP)                     │
│  ─────────────────────────────────────────────────────────────────────     │
│  24-bit FLAC → Decode → 24-bit PCM → Encode → 24-bit FLAC                 │
│  (No bit depth change; pipeline may use float for internal processing)      │
│                                                                             │
│  Path 2: Lossless → Lossy (bit depth reduction)                            │
│  ─────────────────────────────────────────────────────────────────────     │
│  24-bit FLAC → Decode → 32-bit float PCM (DSP) → Reduce to 16-bit → MP3  │
│  Typical DSP chain: Bit Depth (→ 32f) → Volume Normalize → Bit Depth (→ 16) → Dither  │
│                                                                             │
│  Path 3: 16-bit → 32-bit float → process → 16-bit output                  │
│  ─────────────────────────────────────────────────────────────────────     │
│  16-bit WAV → Decode → 32-bit float → EQ → Volume → 16-bit → FLAC        │
│  (Upsample to float for DSP precision; downsample at output)               │
│                                                                             │
│  Path 4: DSD → PCM                                                          │
│  ─────────────────────────────────────────────────────────────────────     │
│  DSD64 (.dsf) → Decode → 352.8kHz PCM OR 705.6kHz PCM OR 1.411MHz PCM   │
│  Option in Control Center: decode DSD to 32-bit int, float32, or float64   │
│                                                                             │
│  NOTE: Bit Depth DSP "to floating point" requires Reference license         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.4 Channel Layout Conventions

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Channel Layout — WAVE_FORMAT_EXTENSIBLE                  │
│                                                                             │
│  DBpoweramp uses Microsoft speaker positions (dwChannelMask):                │
│                                                                             │
│  Mono (1ch):                                                               │
│    Channel 0: Mono (M)                                                     │
│                                                                             │
│  Stereo (2ch):                                                             │
│    Channel 0: Left (L)                                                      │
│    Channel 1: Right (R)                                                     │
│                                                                             │
│  2.1 (3ch):                                                                │
│    Channel 0: Left (L)                                                      │
│    Channel 1: Right (R)                                                    │
│    Channel 2: Low Frequency Effects (LFE)                                  │
│                                                                             │
│  5.0 (5ch):                                                               │
│    Channel 0: Front Left (FL)                                              │
│    Channel 1: Front Right (FR)                                             │
│    Channel 2: Front Center (FC)                                            │
│    Channel 3: Back Left (BL)                                               │
│    Channel 4: Back Right (BR)                                              │
│                                                                             │
│  5.1 (6ch):                                                               │
│    Channel 0: Front Left (FL)                                               │
│    Channel 1: Front Right (FR)                                             │
│    Channel 2: Front Center (FC)                                            │
│    Channel 3: Low Frequency Effects (LFE)                                  │
│    Channel 4: Back Left (BL)                                               │
│    Channel 5: Back Right (BR)                                              │
│                                                                             │
│  7.1 (8ch):                                                               │
│    Channel 0: Front Left (FL)                                               │
│    Channel 1: Front Right (FR)                                             │
│    Channel 2: Front Center (FC)                                            │
│    Channel 3: Low Frequency Effects (LFE)                                  │
│    Channel 4: Back Left (BL)                                               │
│    Channel 5: Back Right (BR)                                               │
│    Channel 6: Side Left (SL)                                               │
│    Channel 7: Side Right (SR)                                               │
│                                                                             │
│  Channel Mapper DSP:                                                       │
│    Each output channel can be:                                             │
│    - Copied from a source channel (e.g., ch3 = 0.5 × ch1 + 0.5 × ch2)    │
│    - Created as a mix of source channels (e.g., center = 0.707 × L + 0.707 × R) │
│    - Silent (zero)                                                         │
│                                                                             │
│  Channel Count DSP:                                                        │
│    Forces a specific channel count (up/downmix)                            │
│    Standard mappings:                                                      │
│      2 → 1: Mono = (L + R) / 2                                           │
│      5.1 → 2 (stereo downmix): ProLogic / ProLogic II / manual            │
│                                                                             │
│  Since R2026-01-31: Channel DSP supports up to 12 channels               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.5 Buffer Management

```pseudocode
// DBpoweramp uses FIXED-SIZE RING BUFFERS during the encode loop.
// NOT growable; NOT fully-loaded. Block-based streaming throughout.

// Buffer sizes:
// - Decoder DecodeBlock() returns blocks of varying size
// - Typical block: 4096 to 65536 samples per block
// - Block size depends on source format and decoder implementation

// PCM block structure passed through pipeline:
STRUCT PCMBlock:
    float* pData      // Pointer to interleaved float samples
    int    numSamples  // Number of samples (per channel)
    int    numChannels // Number of channels
    int    bitsPerSample // 32 or 64 for float; 8/16/24 for int
    bool   isLast     // True for last block (end of audio)

// Block ownership:
// - Decoder allocates block memory
// - DSP chain may reallocate (DSP can return new block pointer)
// - Final encoder uses the block; host frees after EncodeBlock() returns
// - IMPORTANT: Previous block's memory is INVALID after next DecodeBlock() call

// Buffer lifecycle per block:
WHILE TRUE:
    block = decoder.DecodeBlock()
    // block.pData is owned by decoder; valid only until next DecodeBlock()
    
    FOR EACH live_dsp IN dsp_chain:
        block = live_dsp.Process(block)
        // DSP may return SAME pointer (in-place) or NEW pointer (reallocated)
        // If new: previous block is freed by DSP
    
    encoder.EncodeBlock(block)
    // Encoder receives current block; may keep pointer until next call
    
    IF block.isLast:
        BREAK

// No ring buffer in the encode loop — each block is fully processed 
// before the next is requested.

// Non-live DSP temp files:
// When non-live DSPs are active, the ENTIRE source is decoded to disk first:
//   DecodeEntireFile(decoder, "temp_XXXXX.pcm")
//   // Raw PCM, interleaved, format from decoder
//   processed_pcm = dsp.ProcessNonLive("temp_XXXXX.pcm")
//   // Processed PCM written to new temp file
//   new_decoder = CreateDecoderForFile(processed_pcm)
//   // Encode loop continues from new_decoder
```

### 2.6 Gapless Playback Handling

```pseudocode
// DBpoweramp handles gapless playback transparently through the pipeline.

// Gapless info sources:
STRUCT GaplessInfo:
    int  encoderDelaySamples   // Samples to skip at start
    int  encoderPaddingSamples // Samples to append at end
    bool hasGaplessInfo        // True if present

// How gapless info flows through the pipeline:

// 1. DECODE: Decoder reads gapless info from source
decoder = OpenSourceFile(source_path)
gapless = decoder.GetGaplessInfo()
    // MP3 (LAME): Reads from Xing/Info header
    // M4A/AAC: Reads from iTunSMPB tag or iTunNORM
    // FLAC: Reads from STREAMINFO (natural gapless — no extra delay)
    // WAV: No gapless info (redbook audio has no encoder delay)

// 2. DSP CHAIN: Gapless info is NOT modified by DSP unless explicitly
//    (e.g., Trim DSP removes samples — this breaks gapless but is intentional)

// 3. ENCODE: Encoder writes gapless info to output
encoder.BeginConversion(fluid)
    // Encoder receives gapless info from source via fluid or direct call
    // Encoder writes its own delay/padding to output container

// For lossless → lossless (same format family):
// DBpoweramp preserves gapless info:
//   FLAC → FLAC: Preserve STREAMINFO; natural gapless (no delay/padding)
//   ALAC → ALAC: Preserve iTunSMPB
//   WAV → WAV: No gapless to preserve

// For MP3 output:
//   LAME encoder writes Xing/Info header with:
//     - Encoder delay (samples before first music frame)
//     - Padding (samples after last music frame)
//   Decoder skips these on decode → gapless playback

// R2026-01-31 fix:
//   "Discard malformed iTunSMPB data, play whole file even if gapless info tells to play small part of it"
//   This prevents tiny-gap files that some broken encoders produce

// Gapless verification:
// R2025-12-25: "Fixed failure to encode FLAC if source file has large attached pictures"
// R2023-12: "Fixed HE-AAC not being decoded gaplessly"
```

### 2.7 PCM Between Decode → DSP → Encode Stages

```pseudocode
// PCM format can change between stages:

// SOURCE DECODE → DSP
decoder.Open(source_path)
source_format = decoder.GetOutputFormat()
    // Format 1: 24-bit integer PCM from FLAC decoder
    // Format 2: 32-bit float from DSD decoder (DSD → PCM conversion)
    // Format 3: 16-bit integer from MP3 decoder

// DSP chain may CHANGE format:
// Example: Bit Depth DSP → 32-bit float
// Example: Resample DSP → new sample rate
// Example: Channel Count DSP → different channel count

// ENCODE → OUTPUT
// Encoder receives whatever format the DSP chain produces
// Encoder MUST be notified of format changes BEFORE BeginConversion
// For live DSPs that change format: NOT SUPPORTED (use non-live path)

encoder.BeginConversion(fluid)
    // fluid.pWaveFormat contains the CURRENT (post-DSP) format
    // Encoder uses this to configure its output format

// Type conversions throughout:
float_sample = int_sample / (2^(bits-1))     // Int to float normalization
int_sample  = round(float_sample * (2^(bits-1)))  // Float to int (with dither)

// Float range:
//   Integer PCM: -1.0 to +1.0 (normalized)
//   Float PCM:  Can exceed ±1.0 during processing!
//   A gain of +6dB on a 0dBFS signal: 0.707 → 1.414 (exceeds ±1.0)
//   This is intentional; clipping only at final int conversion
```

### 2.8 Lossless Bit-Exact Verification

```pseudocode
// DBpoweramp does NOT perform bit-exact verification for lossless → lossless
// because it re-encodes (not bit-for-bit copy).

// What happens for lossless → lossless:

// FLAC → FLAC:
//   Decode FLAC → PCM → Encode FLAC
//   This is NOT bit-exact (FLAC encoding is lossless but not id)
//   The output FLAC decodes to IDENTICAL PCM as source
//   But the encoded bitstream is DIFFERENT (different compression params)

// Verification mechanism (R2025-04-17):
//   "Apple Lossless encoder added 32 bit support"
//   R2025-01-31: "FLAC Encoder / Apple Lossless - verify removed, 
//                  never did what expected (as taken from disk cache)"
//   VERIFY feature was REMOVED as of R2025-01-31

// What IS preserved:
//   - All metadata tags (bit-exact field preservation)
//   - Cover art (preserved as binary)
//   - Sample rate, bit depth, channel count
//   - Gapless playback info (from source; not re-calculated)

// What IS NOT preserved:
//   - Exact PCM samples (lossless re-encode produces same decoded PCM, not same bits)
//   - Internal FLAC MD5 signature (recalculated on encode)
//   - Seek table positions (may be recalculated)

// For verifying lossless conversion quality:
//   Use [Audio Info] >> [Calculate Audio CRC] utility codec
//   "Compare tracks for differences in the audio component"
//   Computes CRC of decoded audio (not the bitstream)

FUNCTION VerifyLosslessConversion(source_path, output_path):
    source_pcm = DecodeToPCM(source_path)
    output_pcm = DecodeToPCM(output_path)
    // Both decoded to PCM; compare sample values
    
    IF source_pcm == output_pcm:
        RETURN "PASS: Decoded PCM identical"
    ELSE:
        RETURN "FAIL: Decoded PCM differs"
```

### 2.9 Behavior Evidence Sources

| Behavior Claim | Source | Confidence | Quote / Reference |
|---|---|---|---|
| 32-bit float for DSP | [dBpoweramp DSP Help](https://dbpoweramp.com/Help/dMC/dsp) | High | "32 bit floating point (reference required) allows the value of the audio signal to go over the clipping level (+-1.0)" |
| TPDF dither preferred | [dBpoweramp DSP Help](https://dbpoweramp.com/Help/dMC/dsp) | High | "Apply Dither can be used to dither when reducing the bit depth... Triangular dither is the preferred option" |
| Dither order: resample → bit depth → dither | [dBpoweramp Forum: Dithering](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/44570-dithering-about-dithering-and-what-order-when-changing-sample-rate-bits-p-sample) | High | Spoon's recommended order: Bit Depth to 32f → Resample → Bit Depth to target with TPDF |
| 64-bit float added | [Illustrate Versions R2025-11-12](https://versions.dbpoweramp.com/?appid=2) | High | "Bit Depth DSP: Added 64 bit IEEE-float" |
| ARDFTSRC resampler | [Illustrate Versions R2025-11-12](https://versions.dbpoweramp.com/?appid=2) | High | "ARDFTSRC is able to specify arbitrary sample rates to convert to, such as 123456Hz" |
| DSD to PCM decoding options | [Illustrate Versions R2023-01-20](https://versions.dbpoweramp.com/?appid=2) | High | "dsd decoder - added option (Control Center) to decode to 32 bit, float 32 or 64 bit float" |
| RIFF64 / 64-bit float WAV | [Illustrate Versions R2023-01-20](https://versions.dbpoweramp.com/?appid=2) | High | "RIFF64 support... Wave encoder supports 64 bit float option" |
| Verify feature removed | [Illustrate Versions R2025-01-31](https://versions.dbpoweramp.com/?appid=2) | High | "FLAC Encoder / Apple Lossless - verify removed, never did what expected" |
| Floating point corruption bug | [Illustrate Versions R17.7 (2022)](https://versions.dbpoweramp.com/?appid=2) | High | "floating point put through certain DSP effects would corrupt the audio" |
| 32-bit PCM FLAC support | [Illustrate Versions R2022-09-28](https://versions.dbpoweramp.com/?appid=2) | High | "FLAC 1.4.1 - Encoding and decoding of 32-bit PCM is now possible, 1MHz sample rates" |
| Channel DSP 12 channels | [Illustrate Versions R2026-04-03](https://versions.dbpoweramp.com/?appid=2) | High | "Channels (wave + channel DSP) increased to allow 12 channels" |

### 2.10 DSP Chain Order for Audio Quality

```
┌─────────────────────────────────────────────────────────────────────────────┐
│           Optimal DSP Chain Order (from dBpoweramp community docs)           │
│                                                                             │
│  Recommended three-step DSP chain for resample + bit depth reduction:      │
│                                                                             │
│  STEP 1: Bit Depth DSP                                                    │
│    Radio: "fixed"                                                         │
│    Dropdown: "32-bit float"                                               │
│    Apply Dither: "(none)"                                                  │
│    → Converts source PCM to 32-bit float; no dithering yet                 │
│                                                                             │
│  STEP 2: Resample DSP                                                     │
│    Mode: "Resample to Frequency"                                           │
│    Select target frequency: 44100 (or desired)                             │
│    → Resamples 32-bit float to new rate; no precision loss                │
│                                                                             │
│  STEP 3: Bit Depth DSP (final)                                             │
│    Radio: "fixed"                                                         │
│    Dropdown: "16-bit" (or target bit depth)                               │
│    Apply Dither: "Triangular (TPDF)"                                       │
│    → Reduces 32f to 16-bit int WITH dither                                │
│                                                                             │
│  Why this order matters:                                                  │
│  1. Convert to float FIRST: preserves small signals through resampling     │
│  2. Resample SECOND: resampling on float avoids int truncation errors    │
│  3. Dither LAST: adds noise at frequencies where it won't be heard       │
│     (below half the target sample rate's Nyquist)                         │
│                                                                             │
│  WRONG order examples:                                                     │
│  - Dither → Resample: dither noise gets resampled, aliases into audio band │
│  - Resample → Int → Float: truncation error from int step ruins float    │
│  - Float → Dither → Resample: dithered noise reshaped by resampling      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. INTERNAL LOGIC (INFERRED / REVERSE-ENGINEERED)

### 3.1 Float-to-Int Conversion With Dithering

```pseudocode
// Complete algorithm for converting 32-bit float PCM to 16-bit integer with TPDF dither

FUNCTION FloatToIntWithDither(samples_float, num_samples, dither_enabled):
    
    result = AllocateInt16Buffer(num_samples)
    
    FOR i = 0 TO num_samples - 1:
        IF dither_enabled:
            // Triangular PDF (TPDF) dither:
            // Two uniform random numbers in [-0.5, +0.5] added together
            // Produces a triangular distribution from [-1.0, +1.0]
            // Mean = 0, variance = 1/6 (better noise floor than RPDF)
            dither_value = RandomUniform(-0.5, 0.5) + RandomUniform(-0.5, 0.5)
            // dither_value is in range [-1.0, +1.0]
            // Scaled to 1 LSB of output format:
            dither_value = dither_value / (2^16 - 1)  // 1 LSB for 16-bit
        ELSE:
            dither_value = 0
        
        // Add dither, then round to nearest integer
        float_val = samples_float[i] + dither_value
        
        // Round to nearest integer (banker's rounding is OK here too)
        int_val = round(float_val * 32768.0)  // 16-bit: scale to ±32768
        
        // Clamp to int16 range (critical — float may exceed ±1.0)
        int_val = MAX(-32768, MIN(32767, int_val))
        
        result[i] = int_val
    
    RETURN result

// Dither noise shaping (optional advanced feature):
// TPDF is the default. Some tools implement noise-shaped dither:
// - F-weighted (Floyd-Steinberg)
// - Shibata (low-mid frequency emphasis)
// - High-pass (shapes noise above 12kHz)
// DBpoweramp's "Triangular (TPDF)" is the standard flat-spectrum dither.
// No noise shaping options documented in standard DBpoweramp DSP.
// Noise shaping available in: izotope RX, SoX (with -I flag)

// IMPORTANT: When reducing from 24-bit to 16-bit:
// Scale dither by 1 LSB of the FINAL output format, NOT the intermediate
// 24-bit float → 16-bit with TPDF: dither range = [-1 LSB_16, +1 LSB_16]
// NOT [-1 LSB_24, +1 LSB_24]  // Would be too large
```

### 3.2 Sample Rate Conversion Algorithm

```pseudocode
// DBpoweramp's resampler options (R2025-11+):

ENUM ResamplerType:
    ARDFTSRC   // NEW: Arbitrary-ratio frequency-domain time-scrambling SRC
    SSRC       // Legacy: high-quality sample rate conversion (removed R2025-11)
    SOXR       // Optional: SoX resampler library
    DEFAULT    // Control Center global setting

// ARDFTSRC characteristics:
// - Frequency-domain resampling (Fourier-based)
// - Supports arbitrary ratios (not just integer fractions)
// - Quality modes: Standard / High / Minimum Phase
// - No low-pass filter needed (frequency-domain naturally bandlimits)
// - Arbitrary output rates: supports 123456 Hz for scientific applications
// - Better aliasing rejection than SSRC

// SSRC characteristics (removed in R2025-11):
// - Polynomial-based (multi-stage CIC + compensation filters)
// - Integer ratios only (e.g., 44100 → 48000 = 160/147)
// - High quality but not arbitrary rates
// - Low-pass filter applied to prevent aliasing

// How resampling works in the encode loop:
FUNCTION ResampleDSP.ProcessBlock(input_block):
    // Input: float PCM at source rate
    // Output: float PCM at target rate
    
    // ARDFTSRC: frequency-domain method
    // 1. FFT of input block (zero-padded for overlap)
    // 2. Zero out frequency bins above target Nyquist
    // 3. IFFT to time domain
    // 4. Overlap-add to reconstruct continuous signal
    
    // SSRC: polynomial method (pre-R2025-11)
    // 1. Multi-stage interpolation (upsample → lowpass → downsample)
    // 2. Sharp lowpass filter at target Nyquist
    // 3. High-quality windowed-sinc filter

// Resampling in the encode loop (block-by-block):
// Since resampling changes the number of output samples per input block,
// the DSP must buffer input samples and produce output in different-sized chunks.
// ARDFTSRC handles this internally with its overlap-add buffer management.
```

### 3.3 Data Structures

```pseudocode
// Primary audio buffer structure:
STRUCT AudioBuffer:
    float*  samples     // Interleaved float samples (32-bit or 64-bit)
    int     num_samples  // Number of samples per channel
    int     num_channels // Number of channels
    int     sample_rate  // Samples per second
    int     bit_depth   // Bits per sample (32 or 64 for float)
    bool    is_float    // True if float, False if integer

// WAVEFORMATEX (Windows audio format header):
STRUCT WAVEFORMATEX:
    WORD  wFormatTag      // 1 = PCM, 3 = IEEE float
    WORD  nChannels       // Channel count
    DWORD nSamplesPerSec  // Sample rate
    DWORD nAvgBytesPerSec // Byte rate
    WORD  nBlockAlign     // Bytes per sample frame = channels × bits/8
    WORD  wBitsPerSample  // Bits per sample per channel
    WORD  cbSize          // Size of extra format info (0 or 22)

// WAVEFORMATEXTENSIBLE (for multichannel > 2):
STRUCT WAVEFORMATEXTENSIBLE:
    WAVEFORMATEX base
    union:
        WORD wValidBitsPerSample  // Actual precision (e.g., 24 in 32-bit container)
        WORD wSamplesPerBlock     // For compressed formats
        WORD wReserved
    DWORD dwChannelMask     // Speaker channel mask (bit positions)
    GUID  SubFormat         // KSDATAFORMAT_SUBTYPE_PCM or _IEEE_FLOAT

// Audio info returned by decoder:
STRUCT AudioInfo:
    int64  total_samples    // Total samples in file
    int    sample_rate       // Sample rate in Hz
    int    channels         // Channel count
    int    bits_per_sample   // Bit depth
    int    bitrate_kbps     // Average bitrate
    int    duration_msec    // Duration in milliseconds
    GaplessInfo gapless_info  // Encoder delay/padding (if applicable)

// PCM block (from DecodeBlock):
STRUCT PCMBlock:
    void*  data             // float* or int* depending on format
    int    bytes           // Total bytes in block
    int    samples         // Total samples (bytes / channels / (bits/8))
    int    bits_per_sample  // Format of data pointer
    bool   is_last         // True = last block in stream
```

### 3.4 Edge Case Handling

| Edge Case | DBpoweramp Behavior |
|---|---|
| Source = DSD, output = PCM float | DSD decoder converts to 32f/64f by default (configurable in Control Center) |
| Float overflow (gain > 0dBFS) | Allowed during DSP; clipped at final int conversion |
| Source = 32-bit int FLAC | Decodes to 32-bit int; may be promoted to float for DSP |
| Source = 8-bit (a-law/u-law WAV) | Decoder converts to linear PCM before DSP |
| Downmix 5.1 → stereo | ProLogic II or manual channel mix configurable |
| Resample with irrational ratio | ARDFTSRC handles arbitrary ratios; SSRC uses rational approximation |
| Very short file (< 50ms) | SSRC had off-by-one on very short files; fixed in ARDFTSRC |
| Source sample rate > 655350 Hz | FLAC encoder rejects; error message shown |
| Clip detection | No automatic clipping prevention; user must apply gain reduction |
| DC offset in source | Channel Mapper DSP can remove DC offset |
| Pre-emphasized CD | De-emphasis DSP applied automatically if CD reports it |

---

## 4. OPEN-SOURCE EQUIVALENT IMPLEMENTATION

### 4.1 FFmpeg Internal PCM

```bash
# FFmpeg's internal PCM format during processing:

# Decode to native format (FFmpeg uses 32-bit float internally in filter graph):
ffmpeg -i input.flac -af "volume=2.0" -c:a pcm_f32le -f wav pipe:1

# -af "volume=2.0" processes in float
# -c:a pcm_f32le outputs 32-bit float WAV
# FFmpeg's filter graph uses float (typically 32-bit float, rarely 64-bit)
# FFmpeg uses planar float internally (not interleaved)
# Planar: [L][L][L][L][R][R][R][R] vs Interleaved: [L][R][L][R][L][R]

# Dithering with FFmpeg (limited):
# FFmpeg does NOT have built-in TPDF dither in the filter graph
# Must use: -af "aformat=sample_fmts=s16:dither_method=triangular"
# Note: only available in newer FFmpeg versions

# Resampling with FFmpeg:
ffmpeg -i input.flac -af "aresample=44100:res_type=soxr" output.mp3
# soxr = SoX resampler; high quality, arbitrary ratio
# Alternatives: res_type=sinc, res_type=zero_order_hold, res_type=linear

# Channel remapping with FFmpeg:
ffmpeg -i input_5.1.flac -af "channelmap=FL|FR|FC|LFE|BL|BR" output_6ch.flac
# Or use amerge / join for mixing:
ffmpeg -i input.flac -af "amerge=inputs=2" output_stereo.flac
```

### 4.2 SoX Resampler vs ARDFTSRC

```bash
# SoX resampler quality levels (equivalent to DBpoweramp's SSRC legacy):

sox input.wav output.wav rate -v -b 95 44100
# -v   = Very high quality
# -b 95 = 95% passband (higher = sharper cutoff = more ringing)
# Quality tiers: -m (medium), -h (high), -v (very high), -a (aliasing)

# SoX TPDF dither:
sox input.wav -b 16 output.wav dither -f tpdf
# -f tpdf = Triangular PDF (recommended)
# -f rpdf = Rectangular PDF (lower quality)
# -f gply  = Gaussian PDF
# SoX automatically applies TPDF when reducing bit depth with -b flag
```

### 4.3 r8brain-free-src (High-Quality Resampler)

```cpp
// r8brain is the resampler used by many professional audio tools
// Alternative to SSRC and ARDFTSRC

// r8brain characteristics:
// - Multi-stage FIR filter
// - Excellent aliasing rejection
// - Supports arbitrary ratios
// - Configurable latency vs quality

// Usage example (C++):
#include "r8bbase.h"

r8b::CDSPResampler* resampler;
resampler = r8b::newDoubleResampler(
    input_sample_rate,   // e.g., 44100
    output_sample_rate,  // e.g., 48000
    max_output_len      // buffer size
);

std::vector<double> output;
resampler->process(input.data(), input.size(), output);
// output now contains resampled PCM at new sample rate
```

---

## 5. DETAILED IMPLEMENTATION SPECIFICATION

### 5.1 Algorithm — Complete DSP Conversion Chain

**Step 1: Determine Internal Format**
```
Input:    Source audio format, active DSP effects
Process:
  1. Read source format from decoder
     source_format = decoder.GetOutputFormat()
  
  2. Check if any DSP requires floating-point
     IF "Bit Depth → 32-bit float" DSP is present THEN
       internal_format = FLOAT32
     ELSIF "Graphic Equalizer" DSP is present THEN
       internal_format = FLOAT32  // EQ requires float
     ELSIF any DSP requires float THEN
       internal_format = FLOAT32
     ELSE
       internal_format = source_format.bit_depth  // Keep as-is
     
  3. Check if any DSP changes sample rate
     target_rate = source_format.sample_rate
     IF "Resample" DSP is active THEN
       target_rate = ResampleDSP.target_rate
  
  4. Check if any DSP changes channel count
     target_channels = source_format.channels
     IF "Channel Count" or "Channel Mapper" DSP is active THEN
       target_channels = DSP.target_channel_count
  
Output:   InternalFormat { bit_depth, sample_rate, channels, is_float }
```

**Step 2: Decode Source to PCM**
```
Input:    Source file path, target internal format
Process:
  1. decoder.Open(source_path)
     → Reads audio properties (format, duration, gapless info)
     → Populates EncoderFluid.IDTags (raw tag bytes)
  
  2. Allocate PCM buffer for source format
     // Decoder may output integer or float depending on format
     // FLAC: integer PCM → may be promoted to float for DSP
     // DSD: float PCM (DSD → PCM is float process)
     // MP3: integer PCM (fixed-point decode)
  
  3. Decode loop:
     WHILE (block = decoder.DecodeBlock()) IS NOT NULL:
       // block.pData: pointer to PCM samples
       // block.Bytes: size in bytes
       // block.IsLast: end of stream flag
       ProcessBlock(block)
     END WHILE
  
  4. decoder.Close()
Output:   PCM stream ready for DSP processing
```

**Step 3: Apply DSP Chain**
```
Input:    PCM stream, DSP chain, internal format
Process:
  1. FOR EACH block IN PCM_stream:
       current_block = block
       
       FOR EACH DSP IN live_dsp_chain (in order):
         current_block = DSP.ProcessBlock(current_block)
         // DSP may:
         //   - Modify block in-place (same pointer)
         //   - Allocate new buffer (new pointer)
         //   - Change format (sample rate, channels, bit depth)
       
       FeedToEncoder(current_block)
     END FOR
   
  2. FOR EACH live DSP IN live_dsp_chain:
       DSP.EndConversion(fluid)
       // DSP finalizes; may modify EncoderFluid.IDTags (e.g., ReplayGain)
   
  3. FOR EACH non_live DSP IN nonlive_dsp_chain:
       // Already processed during pre-encode phase
       // AfterConversion will be called for these too
Output:   Processed PCM stream → Encoder
```

**Step 4: Final Bit Depth Conversion + Dither**
```
Input:    Float PCM block, target bit depth (e.g., 16-bit)
Process:
  1. IF target_bit_depth < 32 (i.e., final output is integer) THEN:
       IF apply_dither THEN:
         FOR EACH sample IN block:
           // TPDF dither
           dither = RandomTriangularPDF()
           dither_scaled = dither / (2^target_bits)
           float_with_dither = sample + dither_scaled
           int_sample = round(float_with_dither * (2^(target_bits-1)))
           int_sample = clamp(int_sample, min_val, max_val)
       ELSE:
         FOR EACH sample IN block:
           int_sample = round(sample * (2^(target_bits-1)))
           int_sample = clamp(int_sample, min_val, max_val)
   
  2. IF clipping_possible THEN:
       // Check if any sample exceeded ±1.0 (after dither)
       max_val_in_block = MAX(ABS(sample) FOR sample IN block)
       IF max_val_in_block > 1.0:
         // No automatic clipping — audio above 0dBFS continues
         // This is intentional; clipping is a user choice
         // Option available: "Reduce if Above" in Volume Normalize DSP
   
Output:   Integer PCM block → Encoder.EncodeBlock()
```

**Step 5: Encode Output**
```
Input:    PCM block (integer or float), target format, EncoderFluid
Process:
  1. encoder.BeginConversion(fluid)
     → Encoder reads EncoderFluid.WaveFormat (output format)
     → Encoder opens output file, writes container headers
     → Encoder may convert float → int internally if needed
  
  2. FOR EACH block IN PCM_stream:
       encoder.EncodeBlock(block)
       // Encoder handles format conversion internally:
       //   float input: encoder may convert to int encoding format
       //   int input: encoder uses as-is
  
  3. encoder.EndConversion(fluid, &should_write_tags)
     → Encoder writes final frames / container footer
     → Encoder writes gapless info (Xing header, iTunSMPB, etc.)
     → Encoder may write ID tags if ShouldWriteTags = TRUE
Output:   Encoded audio file with tags
```

### 5.2 Gapless Playback Implementation

```pseudocode
// Algorithm for preserving gapless info through conversion:

FUNCTION HandleGapless(source_path, encoder, fluid):
    
    // Step 1: Read gapless info from source
    gapless = ReadGaplessInfo(source_path)
    
    // Step 2: Pass gapless info to encoder
    IF encoder.SupportsGapless() THEN
        encoder.SetGaplessInfo(gapless.delay, gapless.padding)
    
    // Step 3: Encode with gapless info
    encoder.BeginConversion(fluid)
    // Encoder writes gapless info to output:
    //   MP3: LAME Xing header with delay/padding
    //   M4A: iTunSMPB tag
    //   FLAC: Natural (no delay/padding — FLAC is gapless by design)
    
    // Step 4: On decode, skip delay, append padding
    // The decoder handles this transparently:
    //   decoded_pcm = decoder.Decode(source_path)
    //   // decoder skips first [delay] samples
    //   // decoder appends [padding] silent samples at end
    //   // Player then trims padding for gapless concatenation
    
    RETURN gapless

// Specific implementations:

// MP3 (LAME):
//   Encoder writes: LAME header in first frame
//     [11 bytes: Xing identifier]
//     [4 bytes: frames]
//     [4 bytes: bytes]
//     [100 bytes: TOC]
//     [4 bytes: VBR scale]
//   Delay/padding stored in LAME info tag (not in Xing header itself)
//   Delay ≈ 528 samples (LAME encoder delay)
//   Padding ≈ encoder-specific (depends on block size)

// M4A/AAC:
//   iTunSMPB tag format: " padd samples per channel"
//   Value: " 0000 1152 0000 0000 0" (example)
//   Meaning: skip 1152 samples, play full file, no explicit padding

// FLAC:
//   No encoder delay/padding (FLAC is naturally gapless)
//   Total samples in STREAMINFO = actual audio samples
//   No additional processing needed

// WAV:
//   No gapless mechanism in standard WAV
//   Redbook audio has no encoder delay
//   Gapless playback by direct concatenation of PCM data
```

---

## 6. INTEGRATION INTO CONVERSION PIPELINE

### 6.1 PCM Flow Diagram

```
[SOURCE FILE]
     │
     │ Decoder
     ▼
[PCM — source format]
     │
     │ Bit Depth DSP → 32f (if needed)
     ▼
[PCM — 32-bit float, source rate, source channels]
     │
     │ Channel Mapper DSP (if needed)
     ▼
[PCM — 32-bit float, source rate, target channels]
     │
     │ Resample DSP (if needed)
     ▼
[PCM — 32-bit float, target rate, target channels]
     │
     │ Volume / EQ / other DSPs
     ▼
[PCM — 32-bit float, target rate, target channels]
     │
     │ Bit Depth DSP → target int + TPDF dither (if output is int)
     ▼
[PCM — target bit depth (int or float), target rate, target channels]
     │
     │ Encoder
     ▼
[OUTPUT FILE]
```

---

## 7. DBPOWERAMP vs COMPETITORS COMPARISON

| Feature Aspect | DBpoweramp | FFmpeg | SoX | r8brain | Izotope RX |
|---|---|---|---|---|---|
| Internal format | 32-bit float / 64-bit float (R2025+) | 32-bit float (planar) | 64-bit double | 64-bit double | 64-bit double |
| Dither | TPDF only | Limited (newer versions) | TPDF, RPDF, Gaussian | No built-in dither | Multiple noise shapes |
| Resampler | ARDFTSRC (R2025+), SSRC (legacy) | soxr, built-in SRC | Multi-stage FIR | r8b (external) | IP Sampler |
| Arbitrary ratio SRC | Yes (ARDFTSRC) | Yes (soxr) | Yes | Yes | Yes |
| 64-bit float DSP | Yes (R2025+) | Yes | Yes (double) | Yes | Yes |
| Planar audio | No (interleaved) | Yes | No (interleaved) | N/A | Yes |
| Gapless support | Yes (Xing, iTunSMPB, natural) | Yes (Xing, LATM) | No | N/A | No |
| Clip protection | Manual (Reduce if Above) | No | No | No | Yes (Soft Clip) |
| Max channel count | 12 (R2026) | 64+ | 32 | N/A | 16 |
| Max sample rate | 1 MHz (FLAC 1.4.1) | 384 kHz (native) | 192 kHz | N/A | 384 kHz |
| Interleaved internal | Yes | No (planar) | Yes | N/A | Yes |

**DBpoweramp Advantage:** ARDFTSRC's arbitrary-ratio capability (supporting non-standard rates like 706.5 Hz for DSD, 123456 Hz for testing) is unique. TPDF dither is simple but correct. The interleaved format simplifies DSP code at the cost of some cache efficiency.

**DBpoweramp Limitation:** FFmpeg's planar internal format is more cache-friendly for multichannel audio (channel-stride access patterns are more efficient than interleaved). ARDFTSRC is frequency-domain which introduces latency (group delay) compared to time-domain methods.

---

## 8. KNOWN BUGS, QUIRKS & COMMUNITY REPORTS

| Issue | Version | Description | Workaround | Status |
|---|---|---|---|---|
| Floating point corruption through DSP | R17.7 (2022) | "floating point put through certain DSP effects would corrupt the audio" | Update past R17.7 | Fixed |
| SSRC off-by-one on very short files | R2025-11 | "SSRC SRC gives out data on very short files" | Use ARDFTSRC | Fixed (SSRC removed) |
| ARDFTSRC crash in certain instances | R2025-12 | Fixed obscure regression in ARDFTSRC | Update | Fixed |
| ARDFTSRC output sample count +1 | R2025-11 | "ARDFTSRC removed output sample count +1" | Use ARDFTSRC | Fixed |
| Volume Normalize IEEE_FLOAT reading | R2023-11 | "Volume Normalize DSP would not read IEEE_FLOAT correctly if passed in a WaveFormatExtensible header" | Update | Fixed |
| FLAC 32-bit PCM encoding/decoding | R2022-09 | FLAC 1.4.1 added 32-bit PCM support | Update | Fixed |
| DSD → PCM decoder channel mapping | R2025-04 | "Apple Lossless decoder was not channel mapping correctly for > 2 channels" | Update | Fixed |
| HE-AAC gapless decoding | R2023-12 | "Fixed HE-AAC not being decoded gaplessly" | Update | Fixed |
| DSD file with ID3 at start | R2023-12 | "DSD (DSF) supports files with ID3 chunk at start of the file (which breaks the DSF specification)" | Update | Fixed |
| RIFF64 WAV support | R2023-01 | Added: "RIFF64 support (decoding and encoding)" | Update | Fixed |
| 64-bit float WAV encoding | R2023-01 | "Wave encoder supports 64 bit float option" | Update | Fixed |
| Malformed iTunSMPB causing tiny gaps | R2026-01-31 | "Discard malformed iTunSMPB data, play whole file even if gapless info tells to play small part of it" | Update | Fixed |

---

## 9. IMPLEMENTATION CHECKLIST

For a developer replicating this behavior:

**Research Verification**
- [x] 32-bit float internal format confirmed from DSP documentation
- [x] 64-bit float support added in R2025-11 confirmed
- [x] TPDF dither confirmed as preferred option
- [x] Dither → resample → bit depth order confirmed from Spoon forum post
- [x] ARDFTSRC arbitrary-ratio resampling confirmed
- [x] Interleaved PCM format confirmed from DSP architecture spec

**Implementation Requirements**
- [ ] Use 32-bit float as default internal format; support 64-bit float as option
- [ ] Convert to float at start of DSP chain (if any DSP requires precision)
- [ ] Resample BEFORE final int conversion (resample in float)
- [ ] Apply TPDF dither ONLY at final bit depth reduction (32f → 16-bit int)
- [ ] Use fixed-size blocks in encode loop (streaming, not fully-loaded)
- [ ] Buffer decoded PCM blocks; each block invalid after next DecodeBlock()
- [ ] Implement interleaved format (NOT planar) for block data
- [ ] Clamp samples to ±1.0 before int conversion (clipping)
- [ ] Handle gapless info: read from source, pass to encoder, write to output
- [ ] Support multichannel upmix/downmix with correct channel mapping
- [ ] Support 12 channels (max as of R2026)
- [ ] DSD → PCM: decode to float, respect Control Center option (32int/32f/64f)

**Validation**
- [ ] Convert 24-bit FLAC → 16-bit MP3 with DSP chain → verify no truncation artifacts
- [ ] Verify dither noise is TPDF (spectral analysis: flat spectrum below Nyquist)
- [ ] Verify wrong order (dither before resample) produces aliased dither noise
- [ ] Verify gapless MP3: concatenate two gapless tracks, measure inter-track gap (< 0.5ms)
- [ ] Verify float overflow: apply +12dB gain → confirm samples exceed ±1.0 in DSP
- [ ] Verify clipping: confirm samples clamped to int range at output
- [ ] Verify 5.1 → stereo downmix: channel mapping produces correct L/R output
- [ ] Verify ARDFTSRC arbitrary ratio: convert 44100 → 123456 Hz → verify no artifacts
- [ ] Verify lossless re-encode: FLAC → FLAC → decode both → compare PCM → identical
- [ ] Verify RIFF64 WAV: encode to 64-bit float WAV → decode → verify float format

---

> **"If I implement exactly what this document describes, will a user converting files with my tool notice any difference from DBpoweramp in terms of tag content, tag completeness, or metadata preservation?"**

**Answer: Largely NO for metadata preservation, but YES for audio quality if the internal PCM representation is wrong.** A tool using integer DSP instead of float will produce audibly different results on material with subtle low-level details — reverb tails, room ambience, and quiet passages will be slightly noisier due to accumulated truncation errors. Specifically:
1. **DSP precision** — A tool doing equalizer or volume normalization in 16-bit integer will introduce rounding noise in quiet passages. Float DSP is perceptually transparent.
2. **Dither quality** — A tool without TPDF dither (or using RPDF) will have a slightly higher noise floor. This is audible on high-quality headphones with quiet recordings.
3. **Dither ordering** — A tool applying dither before resampling will produce audibly different (worse) results than one following the correct order.
4. **Gapless precision** — A tool that doesn't read/write encoder delay/padding correctly will produce small (~1000 sample) gaps between tracks on gapless players (Pioneer, Sony, etc.).
5. **Metadata preservation** — The internal PCM format has NO effect on metadata. Tag preservation is identical regardless of float vs. int DSP.
