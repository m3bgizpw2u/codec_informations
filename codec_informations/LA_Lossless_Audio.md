# LA (Lossless Audio) — Deep Technical Reference

> **Category:** Lossless
> **File Extensions:** `.la`
> **MIME Types:** `audio/x-la`, `audio/la`
> **Standardization Body:** Marc Heuser (abandoned project)
> **Primary Specification:** No public specification — reverse-engineered format
> **Patent Status:** Unknown
> **License:** Proprietary (source not publicly available)
> **Current Version:** 0.4 (final)
> **Active Development:** No — abandoned ~2005

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Marc Heuser [NEEDS VERIFICATION]
- **Year Created:** 2003-2004
- **Original Purpose:** Maximum-compression lossless audio codec for archival
- **Problem with Predecessors:** Existing codecs (FLAC early versions, Monkey's Audio) didn't compress as well. LA was designed to compete with OptimFROG in compression ratio.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| LA0 | 2003 | Initial release |
| LA1 | 2003 | Bug fixes |
| LA2 | 2003 | Improved compression |
| LA3 | 2004 | Further improvements |
| LA4 | 2004 | Final version with all features |

### 1.3 Current Adoption
- **Primary use cases today:** Historical — files still exist in archives
- **Platforms with native support:** Windows (original encoder), limited cross-platform
- **Major services using this format:** None
- **Hardware support:** No hardware support
- **Status:** Deprecated/Abandoned — superseded by FLAC and other codecs

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Waveform / Predictive
- **Core algorithm:** Adaptive LMS (Least Mean Squares) filtering with arithmetic coding
- **Loss mechanism:** Lossless — uses sophisticated multi-filter prediction with arithmetic coding
- **Frame-based vs sample-based:** Frame-based; fixed-size frames
- **Fixed vs variable frame size:** Fixed frame size (16 blocks × 73,728 samples per block)

### 2.2 High-Level Encoding Flow (Block Diagram)
```
Input PCM Samples
      │
      ▼
[Pre-processing: Format validation, channel detection]
      │
      ▼
[Frame Splitting: Fixed-size blocks]
      │
      ▼
[Channel Decorrelation: Stereo decorrelation for multi-channel]
      │
      ▼
[Adaptive LMS Filtering: Multiple filter stages]
      │
      ▼
[Entropy Coding: Arithmetic coding]
      │
      ▼
[Bitstream Packing: Header + frames + seek table]
      │
      ▼
Output LA Stream
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | ~2.4 seconds per block | 16 blocks of 73,728 samples |
| Block size | 73,728 samples (models 1-2) | 61440 samples (model 4) |
| Frame size | 16 blocks per frame | Very large frame size |
| Max channels | 2 (stereo) | [NEEDS VERIFICATION] |
| Max bit depth | 24-bit | [NEEDS VERIFICATION] |
| Max sample rate | 192 kHz | [NEEDS VERIFICATION] |
| Bitrate range | N/A | Lossless — varies with content |
| Complexity | O(n × complexity) | Slow encoding |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  3       4C 41 30        LA0       File magic / signature
0x0003  1       XX               ..        Model version ('2'-'4')
```

### 3.2 File-Level Header Layout
```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------
0x0000   3B      Magic                  char[3]     "LA0"
0x0003   1B      Model Version          char        '2', '3', or '4'
0x0004   4B      Raw Audio Size         uint32 LE   Uncompressed audio size in bytes
0x0008   32B     WAV Header             uint8[32]   First 32 bytes of WAV header
0x0028   4B      Number of Samples      uint32 LE   Total coded samples (all channels)
0x002C   1B      Flags                  uint8       Bit 0: seek table present, Bit 1: model 4 extra filters
0x002D   4B      Header CRC             uint32 LE   CRC32 of header
```

### 3.3 Frame / Block Structure
```
Frame Structure:
  Each frame contains 16 blocks of audio data
  Block size: 73,728 samples (models 1-2) or 61,440 samples (model 4)
  
  Frame header: None explicit
  Frame boundary: Determined by sample count
```

### 3.4 Filter Architecture (Model 4)
Based on reverse-engineering:

```
Filter Stages (Model 4):
  1. Channel Decorrelator (stereo only)
     - Predicts one channel from the other
     - Reduces inter-channel redundancy
     
  2. Delta Filter (single channel)
     - Simple predictor: pred = last_sample × weight >> 8
     - Weight selected adaptively
     
  3. Quick Filter (order 16)
     - Adaptive LMS filter with 16 coefficients
     
  4. Big Filter (order 512)
     - High-order LMS filter
     - Single channel
     
  5. Big Filter (order 16) [if extra compression bit set]
  6. Big Filter (order 288) [if extra compression bit set]
  7. Big Filter (order 96)
  8. Big Filter (order 16)
```

### 3.5 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Unsigned integer | Yes [NEEDS VER] | Rarely used |
| 16-bit | Signed integer | Yes | Standard CD audio |
| 20-bit | Signed integer | Yes [NEEDS VER] | Professional audio |
| 24-bit | Signed integer | Yes | High-resolution standard |
| 32-bit | Signed integer | No | Not supported |
| 32-bit | IEEE float | No | Not supported |

#### Supported Sample Rates
| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 44100 | CD audio | Yes | Most common |
| 48000 | Professional | Yes | Standard |
| 88200 | 2× CD | Yes | |
| 96000 | High-res | Yes | |
| 176400 | 4× CD | Yes | |
| 192000 | High-res max | Yes | |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Pre-Processing Stage
- **DC offset removal:** Implicit in linear prediction
- **Pre-emphasis filter:** None
- **Windowing function:** None (block-based)
- **Level normalization:** None
- **Stereo decorrelation pre-step:** Channel decorrelator for stereo

### 4.2 Analysis / Transform Stage

#### Adaptive LMS Filters
```
LMS (Least Mean Squares) Filter:
  - Coefficients adapt based on prediction error
  - Multiple filter orders for different signal components
  - Fast adaptation for transient content
  - Slow adaptation for stationary content
  
Filter Adaptation:
  error = actual_sample - predicted_sample
  coefficients[i] += mu × error × input[i]
  where mu is the adaptation rate
```

### 4.3 Psychoacoustic Model (Not Applicable)
LA is a **lossless** codec. No psychoacoustic modeling is performed.

### 4.4 Quantization
LA uses **no quantization** — it is a purely lossless codec.

### 4.5 Stereo Encoding Modes
| Mode | Description | Implementation |
|------|-------------|----------------|
| Independent | L and R encoded separately | Default |
| Channel Decorrelation | One channel predicts the other | Model 4 only |

### 4.6 Entropy / Lossless Coding Stage
```
Method: Arithmetic Coding

Arithmetic coding properties:
  - Uses precise probability models
  - Achieves near-entropy coding efficiency
  - More complex than Huffman or Rice coding
  - Provides optimal compression for symbol streams
  
Context modeling:
  - Probabilities adapt to local statistics
  - Better compression than static models
```

### 4.7 Encoder Settings / Quality Modes

LA was primarily designed for maximum compression with no speed options documented.

| Mode | Encoding Speed | Compression | Notes |
|------|---------------|-------------|-------|
| Default | Slow | Maximum | Single mode |

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. Scan for magic bytes: "LA0" followed by model version
2. Read and validate header
3. Validate header CRC32
4. Read seek table (if present in flags)
5. Decode frames sequentially
```

#### Seeking
- **Seek table:** Optional, stored in file
- **Seek table format:** Frame end positions (uint32 per entry)
- **Precision:** Frame-level

### 5.2 Core Decode Pipeline
```
1. Read file header
   ├── Verify magic "LA0"
   ├── Parse model version
   ├── Read WAV header duplicate
   ├── Validate header CRC32
   └── Determine if seek table present

2. Read seek table (if present)
   └── Frame end positions

3. Decode frames
   ├── Arithmetic decode residuals
   ├── Apply inverse filters in reverse order
   ├── Reconstruct channel data
   └── Output PCM samples

4. Arithmetic decode
   ├── Maintain probability state
   └── Reconstruct filter coefficients and residuals
```

### 5.3 Error Concealment
- **Corrupt frame detection:** CRC32 on header only [NEEDS VER]
- **Concealment method:** Likely replace with silence
- **Error tolerance:** Not designed for error resilience

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** LA is a raw stream format
- **Overhead:** Very low (header + CRCs)
- **Seeking in native container:** Via seek table
- **Multiple streams in native container:** No

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| LA (native) | Yes | Yes (if seek table) | None | No container |
| Matroska/MKA | Unlikely | — | — | Not commonly muxed |
| Other containers | No | — | — | Not designed for |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** None native
- **Tag storage:** LA has its own simple tag format [NEEDS VER]
- **External tags:** ID3v1, ID3v2, or APEv2 may be appended

### 7.2 Standard Tag Fields
LA has no documented native tag system. Metadata is typically stored in external tag formats.

### 7.3 Cover Art Storage
No native cover art support.

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   la              # [NEEDS VER] - may not exist
AV_CODEC_ID:        AV_CODEC_ID_LAF  # [NEEDS VER]
Format Name (CLI):  la               # [NEEDS VER]
Encoder(s):         NONE — FFmpeg does NOT support LA encoding
Decoder(s):         NONE or LIMITED — FFmpeg has no native LA decoder
```

### 8.2 FFmpeg Decoding — NOT SUPPORTED
FFmpeg does **NOT** support LA decoding. LA files must be decoded using the original LA decoder or converted before processing.

### 8.3 Alternative Decoding Methods
```
1. Use original LA decoder (Windows only likely)
2. Convert LA files to another format using original software
3. Use foobar2000 with LA plugin (if available)
```

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
```
LA Seek Table (if present):
  Location:     After header
  Entry size:   4 bytes (uint32)
  Entry format:
    [0x00–0x03]  frame_end_position (uint32) — Byte position after frame
  Seek table size: Inferred from header flags
```

### 9.2 Gapless Playback Data
```
Encoder delay:   0 samples (assumed)
Padding:         0 samples (assumed)
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable | No | Designed for file-based archival |
| Live encoding | No | Too slow |
| HTTP progressive download | Yes | After full download |
| HLS / DASH | No | Not applicable |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | Layout Name | Notes |
|----------|-------------|-------|
| 1 | Mono | Supported |
| 2 | Stereo | Primary design target |

**Note:** LA was primarily designed for stereo audio. Multi-channel support is uncertain.

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 24-bit [NEEDS VER] | Integer only |
| Max sample rate | 192 kHz [NEEDS VER] | |
| Float support | No | Not supported |

---

## 13. KNOWN ISSUES, BUGS & EDGE CASES

### 13.1 Critical Issues
| Issue | Description | Workaround |
|-------|-------------|------------|
| No FFmpeg support | Cannot decode with standard tools | Use original decoder |
| Abandoned project | No development since 2005 | Convert to other format |
| Rare format | Limited software support | May need custom solutions |

### 13.2 Edge Cases
- **Corrupt files:** No recovery mechanism documented
- **Truncated files:** Decoder likely fails
- **Missing seek table:** Slower seeking by frame scan

---

## 14. CONVERSION GUIDE (DBpoweramp Context)

### 14.1 Converting FROM LA

Since FFmpeg does not support LA, alternative methods are required:

1. **Use original LA decoder** (if available)
2. **Use foobar2000 with plugin** (if available)
3. **Custom conversion pipeline**

### 14.2 Converting TO LA
Not recommended. LA is deprecated and has no advantages over FLAC or other modern codecs.

---

## 15. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Notes |
|---------|----------|---------|-------|
| LA Reference | Unknown | Proprietary | Not publicly available |
| FFmpeg | — | — | No support |
| foobar2000 plugin | — | — | Unknown if exists |

---

## 16. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **Multimedia Wiki:** https://wiki.multimedia.cx/index.php/La_Lossless_Audio
- **No official documentation available**

### Technical Resources
- Hydrogenaudio Lossless Comparison: https://wiki.hydrogenaudio.org/index.php?title=Lossless

---

## 17. IMPLEMENTATION CHECKLIST (Converter Developer)

### Critical Issues
- [ ] FFmpeg has no LA decoder — alternative solution required
- [ ] Original LA decoder may be Windows-only
- [ ] No cross-platform decoding solution in standard tools
- [ ] Recommend converting LA files to FLAC as first step

### Alternative Approaches
1. **Bundled decoder:** Include original LA decoder as external tool
2. **Pre-conversion:** Convert LA to WAV using original decoder, then process with FFmpeg
3. **Plugin integration:** Use foobar2000 SDK or similar

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
