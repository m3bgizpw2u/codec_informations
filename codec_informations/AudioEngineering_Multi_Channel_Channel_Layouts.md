# Audio Engineering: Multi-Channel & Channel Layouts — Deep Technical Reference
> **Category:** AudioEngineering
> **File Extensions:** N/A (theory reference document)
> **MIME Types:** N/A
> **Standardization Body:** ITU-R, SMPTE, Dolby, DTS, AES
> **Primary Specification:** ITU-R BS.775-3, SMPTE ST 428-12, Dolby Metadata Guide, DTS Core Metadata
> **Patent Status:** Channel layout specifications are public domain; proprietary implementations (Dolby Atmos, DTS:X)
> **License:** Layout standards are public; surround encoding/decoding algorithms carry their own licenses
> **Current Version:** Active — object-based audio (Dolby Atmos, DTS:X) extends beyond channel-based layouts
> **Active Development:** Yes — object-based and immersive audio (7.1.4, 22.2) evolving

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Origins of Multichannel Audio
- **Creator(s):** Various — quadraphonic experiments (1970s), Dolby Stereo (1976), DVD-Audio/SACD (1999-2000)
- **Year Created:** 1970s onward
- **Original Purpose:** To provide immersive, spatial audio experiences beyond stereo's 2-channel limitation, initially for cinema and later for home entertainment.
- **Problem with Predecessors:** Stereo provided limited spatial imaging; monophonic was spatially flat. Multichannel opened new dimensions for sound placement and immersion.

### 1.2 Key Historical Milestones
|| Year | Milestone | Significance |
|------|---------|-------------|
| 1970s | Quadraphonic (4.0) | First consumer multichannel; failed commercially |
| 1976 | Dolby Stereo Matrix | 4→2 matrix encoding for film |
| 1982 | Dolby Pro Logic | Consumer decode of Lt/Rt |
| 1995 | Dolby Digital (AC-3) | Discrete 5.1 digital codec |
| 1996 | DTS Coherent Acoustics | Discrete 5.1 codec |
| 1999 | DVD-Audio | High-resolution multichannel |
| 1999 | SACD | Direct Stream Digital multichannel |
| 2012 | Dolby Atmos | Object-based audio |
| 2015 | DTS:X | Object-based audio |

### 1.3 Current Adoption
- **Primary use cases:** Cinema, home theater, game audio, VR/AR, music production, streaming (Dolby Digital Plus, Dolby Atmos)
- **Platforms with native support:** AV receivers, soundbars, game consoles (PS5, Xbox), streaming services (Netflix, Disney+)
- **Major services using surround:** Netflix, Disney+, Apple TV+, Blu-ray, 4K Ultra HD Blu-ray
- **Hardware support:** 7.1.4 AV receivers, soundbars with upfiring speakers, gaming headsets with 7.1
- **Status:** Standard in home theater; transitioning to object-based for premium content

---

## 2. ITU-R BS.775 — THE FOUNDATIONAL STANDARD

### 2.1 Overview
ITU-R BS.775 defines the reference multichannel speaker layout for programme production and monitoring. It establishes the foundational 5.1 and 7.1 layouts used throughout the industry.

### 2.2 5.1 Speaker Layout (Primary Configuration)
```
                    [C] Center
                      │
                      │
[Ls]──────[L]────────┼────────[R]──────[Rs]
  │                                      │
  │                                      │
  │          [Screen/TV]                 │
  │                                      │
  └──────────────────────────────────────┘

Angle Reference (from center front):
  L:  –30°
  C:    0°
  R:  +30°
  Ls: –110° to –120°
  Rs: +110° to +120°
```

**Precise ITU-R BS.775 Angles:**
| Speaker | Angle from center (°) | Elevation | Distance |
|---------|----------------------|-----------|----------|
| L | –30° | 0° (ear height) | Reference |
| C | 0° | 0° | Reference |
| R | +30° | 0° | Reference |
| Ls | –100° to –120° | 0° (or slightly elevated) | Same or further than L/R |
| Rs | +100° to +120° | 0° (or slightly elevated) | Same or further than L/R |

### 2.3 LFE Channel
The Low-Frequency Effects (LFE) channel is a dedicated bass channel:

| Property | Value | Notes |
|----------|-------|-------|
| Frequency range | 20–120 Hz | Subwoofer range |
| Bandwidth | 100 Hz typical | Limited in most codecs |
| Gain | +10 dB relative to main channels | "Double bass" |
| Content | Bass effects, sub-bass | Not full-range bass |
| Optional | Yes | 5.0 (without LFE) is valid |

**LFE Myths Debunked:**
- LFE is NOT a subwoofer output — it's a separate channel
- Main speakers should still handle bass (especially in small rooms)
- LFE content should be specifically mixed for the .1 channel

### 2.4 Screen Height Speakers
For cinema/production monitoring with an acoustically transparent screen:
```
[CH] Center High (optional)
         ↑
  [C] Center (behind screen)
```

**When screen is not acoustically transparent:**
- Center speaker placed immediately above or below the screen
- Height compensation may be needed for monitoring

---

## 3. CHANNEL LAYOUT STANDARDS

### 3.1 ITU-R BS.775 Hierarchy
```
Level 0: Mono (1.0) — 1 channel
Level 1: Stereo (2.0) — 2 channels
Level 2: 3/0 — L, C, R (3 channels)
Level 3: 3/1 — L, C, R, center surround
Level 4: 3/2 — L, C, R, Ls, Rs (5.0) — PRIMARY STANDARD
Level 5: 3/2+LFE — L, C, R, Ls, Rs + LFE (5.1)
Level 6: 5/2 — Extended with additional front/rear speakers
```

### 3.2 Standard Channel Layout Reference Table
|| Channels | Layout Name | Channel Order | FFmpeg Constant |
|---------|-----------|-------------|----------------|
| 1 | Mono | C | AV_CHANNEL_LAYOUT_MONO |
| 2 | Stereo | L, R | AV_CHANNEL_LAYOUT_STEREO |
| 2.1 | 2.1 | L, R, LFE | AV_CHANNEL_LAYOUT_2POINT1 |
| 3 | 3.0 | L, C, R | — |
| 3 | 3.0(back) | L, R, Cs | — |
| 4 | 4.0 | FL, FR, RL, RR | AV_CHANNEL_LAYOUT_QUAD |
| 4 | 4.0(side) | FL, FR, SL, SR | — |
| 4 | Quadrophonic | FL, FR, BL, BR | — |
| 5 | 5.0 | FL, FR, FC, BL, BR | AV_CHANNEL_LAYOUT_5POINT0 |
| 5 | 5.0(side) | FL, FR, FC, SL, SR | AV_CHANNEL_LAYOUT_5POINT0_SIDE |
| 5 | 5.0(back) | FL, FR, FC, BL, BR | — |
| 6 | 5.1 | FL, FR, FC, LFE, BL, BR | AV_CHANNEL_LAYOUT_5POINT1 |
| 6 | 5.1(side) | FL, FR, FC, LFE, SL, SR | AV_CHANNEL_LAYOUT_5POINT1_SIDE |
| 6 | 5.1(back) | FL, FR, FC, LFE, BL, BR | — |
| 7 | 6.0 | FL, FR, FC, BC, SL, SR | — |
| 7 | 6.0(front) | FL, FR, FC, FLC, FRC, BC | — |
| 8 | 6.1 | FL, FR, FC, LFE, BC, SL, SR | — |
| 8 | 6.1(back) | FL, FR, FC, LFE, BL, BC, BR | — |
| 8 | 6.1(front) | FL, FR, FC, LFE, FLC, FRC, BC | — |
| 8 | 7.0 | FL, FR, FC, BL, BR, SL, SR | AV_CHANNEL_LAYOUT_7POINT0 |
| 8 | 7.0(back) | FL, FR, FC, BL, BR, SL, SR | — |
| 8 | 7.0(wide) | FL, FLC, FR, FRC, FC, BL, BR | — |
| 8 | 7.0(wide-side) | FL, FLC, FR, FRC, FC, SL, SR | — |
| 8 | 7.1 | FL, FR, FC, LFE, BL, BR, SL, SR | AV_CHANNEL_LAYOUT_7POINT1 |
| 8 | 7.1(wide) | FL, FLC, FR, FRC, FC, LFE, BL, BR | — |
| 8 | 7.1(wide-side) | FL, FLC, FR, FRC, FC, LFE, SL, SR | — |

### 3.3 Channel Naming Conventions
| Abbreviation | Full Name | Notes |
|-------------|-----------|-------|
| FL | Front Left | — |
| FR | Front Right | — |
| FC | Front Center / Center | — |
| C | Center | Same as FC |
| LFE | Low Frequency Effects | Subwoofer |
| SL | Surround Left | Side surround (preferred over BL) |
| SR | Surround Right | Side surround (preferred over BR) |
| BL | Back Left | Rear surround |
| BR | Back Right | Rear surround |
| BC | Back Center | Center rear |
| FLC | Front Left of Center | Between FL and FC |
| FRC | Front Right of Center | Between FR and FC |

---

## 4. CHANNEL ORDERING STANDARDS

### 4.1 ITU-R BS.775 Channel Order
The original standard defines channel order for 3/2 configuration:
```
Order: L, C, R, LS, RS
       (then + LFE if present)
```

### 4.2 SMPTE Channel Order
SMPTE ST 428-12 defines the channel order for cinema:
```
5.1 Order: L, C, LFE, R, LS, RS
         1   2   3    4   5    6
```

### 4.3 WAV/AIFF/FFmpeg Channel Order
FFmpeg and WAV files use a different order for 5.1:
```
5.1 Order: FL, FR, FC, LFE, BL, BR
         1    2    3    4    5    6
```
This matches SMPTE ST 428-12 EXCEPT for BL/BR vs SL/SR positioning.

### 4.4 Microsoft Windows WAVEFORMATEXTENSIBLE Channel Order
Microsoft defines WAVE_FORMAT_EXTENSIBLE channel mappings:
```
5.1 Order: FL, FR, FC, LFE, BL, BR  (matches WAV standard)
         1    2    3    4    5    6

7.1 Microsoft Order (non-standard):
   FL, FR, FC, LFE, BL, BR, SL, SR
   (rear surrounds before side surrounds)
```

### 4.5 Channel Position Bitmask (WAVEFORMATEXTENSIBLE)
```c
// Speaker position bitmask definitions:
#define SPEAKER_FRONT_LEFT      0x00000001  // 1
#define SPEAKER_FRONT_RIGHT     0x00000002  // 2
#define SPEAKER_FRONT_CENTER    0x00000004  // 4
#define SPEAKER_LOW_FREQUENCY   0x00000008  // 8
#define SPEAKER_BACK_LEFT       0x00000010  // 16
#define SPEAKER_BACK_RIGHT      0x00000020  // 32
#define SPEAKER_FRONT_LEFT_OF_CENTER  0x00000100  // 256
#define SPEAKER_FRONT_RIGHT_OF_CENTER 0x00000200  // 512
#define SPEAKER_BACK_CENTER     0x00000100  // 256
#define SPEAKER_SIDE_LEFT       0x00000200  // 512
#define SPEAKER_SIDE_RIGHT      0x00000400  // 1024
#define SPEAKER_TOP_CENTER      0x00000800  // 2048
#define SPEAKER_TOP_FRONT_LEFT  0x00001000  // 4096
#define SPEAKER_TOP_FRONT_CENTER 0x00002000  // 8192
#define SPEAKER_TOP_FRONT_RIGHT 0x00004000  // 16384
#define SPEAKER_TOP_BACK_LEFT   0x00008000  // 32768
#define SPEAKER_TOP_BACK_CENTER 0x00010000  // 65536
#define SPEAKER_TOP_BACK_RIGHT  0x00020000  // 131072
```

---

## 5. STEREO MATRIX ENCODING — Lt/Rt

### 5.1 Dolby Pro Logic Matrix
Dolby Pro Logic encodes 4 channels (L, C, R, S) into 2 channels (Lt, Rt) for backward compatibility with stereo:

```
Lt = L + 0.707 × C + 0.707 × S
Rt = R + 0.707 × C - 0.707 × S

Decoded (Pro Logic):
  L' = Lt
  R' = Rt
  C' = 0.707 × (Lt + Rt)
  S' = 0.707 × (Lt - Rt)
```

### 5.2 Lt/Rt vs Lo/Ro
| Aspect | Lt/Rt (Matrixed) | Lo/Ro (Discrete) |
|--------|-----------------|------------------|
| Encoding | Matrix encoding | No encoding |
| Channels | 4→2 matrix | 4 discrete channels |
| Quality | Some separation loss | Full separation |
| Compatibility | Stereo playback | Requires 4+ channel system |
| Use case | DVDs, broadcast | Blu-ray, modern streaming |

### 5.3 Dolby Pro Logic II/IIx
Pro Logic II improves upon original Pro Logic:
- Full-bandwidth surround channel
- Improved separation (30 dB vs 3 dB)
- Music mode with stereo surround option
- Cinema mode optimized for film

Pro Logic IIx extends to 6.1/7.1:
```
Pro Logic IIx 5.1 → 6.1/7.1:
  Lt/Rt → L, C, R, SL, SR, (BC for 6.1)
```

---

## 6. FFmpeg CHANNEL LAYOUT REFERENCE

### 6.1 FFmpeg Channel Layout Constants
```c
// From libavutil/channel_layout.h
AV_CHANNEL_LAYOUT_MONO          // 1 channel: C
AV_CHANNEL_LAYOUT_STEREO        // 2 channels: L, R
AV_CHANNEL_LAYOUT_2POINT1       // 3 channels: L, R, LFE
AV_CHANNEL_LAYOUT_2_1           // 3 channels: L, R, Cs
AV_CHANNEL_LAYOUT_QUAD          // 4 channels: FL, FR, RL, RR
AV_CHANNEL_LAYOUT_4POINT0       // 4 channels: FL, FR, FC, RC
AV_CHANNEL_LAYOUT_5POINT0       // 5 channels: FL, FR, FC, BL, BR
AV_CHANNEL_LAYOUT_5POINT0_SIDE  // 5 channels: FL, FR, FC, SL, SR
AV_CHANNEL_LAYOUT_4POINT1       // 5 channels: FL, FR, FC, LFE, RC
AV_CHANNEL_LAYOUT_5POINT1       // 6 channels: FL, FR, FC, LFE, BL, BR
AV_CHANNEL_LAYOUT_5POINT1_SIDE  // 6 channels: FL, FR, FC, LFE, SL, SR
AV_CHANNEL_LAYOUT_7POINT0       // 7 channels: FL, FR, FC, BL, BR, SL, SR
AV_CHANNEL_LAYOUT_7POINT0_FRONT // 7 channels: FL, FLC, FRC, FR, FC, BL, BR
AV_CHANNEL_LAYOUT_7POINT1       // 8 channels: FL, FR, FC, LFE, BL, BR, SL, SR
AV_CHANNEL_LAYOUT_7POINT1_WIDE  // 8 channels: FL, FLC, FRC, FR, FC, LFE, BL, BR
AV_CHANNEL_LAYOUT_7POINT1_WIDE_SIDE // 8 channels: FL, FLC, FR, FRC, FC, LFE, SL, SR
```

### 6.2 FFmpeg Channel Layout CLI
```bash
# Specify channel layout:
ffmpeg -i input.wav -af "channelmap=channel_layout=5.1" output.wav

# Convert 5.1 to stereo:
ffmpeg -i input_5_1.wav -af "pan=stereo|c0=c0+c2+0.707*c4+0.707*c5|c1=c1+c2+0.707*c4+0.707*c5" output_stereo.wav

# List channel layouts:
ffmpeg -layouts

# Sample output:
# mono
# stereo
# quad
# 5.0
# 5.1
# 5.0(side)
# 5.1(side)
# 7.1
# 7.1(wide)
```

### 6.3 FFmpeg Channel Remapping
```bash
# Remap channels (5.1 to different order):
ffmpeg -i input_5_1.wav \
  -af "pan=5.1|c0=c0|c1=c1|c2=c2|c3=c4|c4=c5|c5=c3" \
  output.wav

# Extract specific channels:
ffmpeg -i input_5_1.wav -map_channel 0.0.0 -map_channel 0.0.1 left_right.wav
ffmpeg -i input_5_1.wav -map_channel 0.0.2 center.wav
ffmpeg -i input_5_1.wav -map_channel 0.0.3 lfe.wav
```

---

## 7. DOWNMIX ALGORITHMS

### 7.1 5.1 to Stereo Downmix
Standard downmix formula (ITU-R BS.775-3):

```
L_out = L_in + 0.707 × C_in + 0.707 × LS_in
R_out = R_in + 0.707 × C_in + 0.707 × RS_in
LFE:  Typically discarded (or mixed at 0.5× if preferred)
```

**FFmpeg pan filter for 5.1 → Stereo:**
```bash
ffmpeg -i input_5_1.wav \
  -af "pan=stereo|FL=FC+0.707*FL+0.707*BL|FR=FC+0.707*FR+0.707*BR" \
  output_stereo.wav
```

### 7.2 5.1 to Mono Downmix
```
Mono = 0.707 × L + C + 0.707 × R + 0.5 × LFE + 0.5 × LS + 0.5 × RS
```

**FFmpeg:**
```bash
ffmpeg -i input_5_1.wav \
  -af "pan=mono|c0=0.707*FL+FC+0.707*FR+0.5*BL+0.5*BR+LFE" \
  output_mono.wav
```

### 7.3 7.1 to 5.1 Downmix
```
5.1(L) = 7.1(L) + 0.707 × 7.1(FLC)
5.1(R) = 7.1(R) + 0.707 × 7.1(FRC)
5.1(C) = 7.1(C)
5.1(LFE) = 7.1(LFE)
5.1(BL) = 7.1(BL) + 0.707 × 7.1(SL)
5.1(BR) = 7.1(BR) + 0.707 × 7.1(SR)
```

### 7.4 Lo/Ro vs Lt/Rt Downmix
| Type | Formula | Use Case |
|------|---------|----------|
| Lo = L + 0.7C + 0.7S | Discrete-compatible | DVD-Audio |
| Lt = L + 0.7C + 0.7S | Matrix-compatible | Dolby Digital |

---

## 8. DOLBY ATMOS & OBJECT-BASED AUDIO

### 8.1 Object vs Channel Audio
| Aspect | Channel-Based | Object-Based |
|--------|--------------|--------------|
| Audio stored as | Fixed channel assignment | Individual audio elements with metadata |
| Metadata includes | Channel name only | Position (X, Y, Z), size, gain |
| Renderer | Built into the audio | Renderer interprets metadata |
| Playback | Fixed speaker layout | Adapts to available speakers |
| Scalability | Limited by channel count | Infinitely scalable |

### 8.2 Dolby Atmos Bed + Objects
```
Bed Audio (up to 7.1.2):
  Channels: 7.1.2
  7: FL, FR, FC, LFE, BL, BR, BC
  1: LFE
  2: Height (FL, FR) or (TML, TMR)

Object Audio:
  Individual elements with 3D position metadata
  Examples: "dialogue", "footsteps", "explosion"
```

### 8.3 Atmos Speaker Configurations
| Config | Channels | Height Speakers |
|--------|----------|----------------|
| 5.1.2 | 8 | 2 (front height) |
| 5.1.4 | 10 | 4 (4 corners) |
| 7.1.2 | 10 | 2 (front height) |
| 7.1.4 | 12 | 4 (4 corners) |
| 9.1.6 | 16 | 6 |
| 22.2 | 24 | Multiple layers |

### 8.4 FFmpeg and Atmos
```bash
# Check for Atmos metadata:
ffprobe -v quiet -show_entries stream=channels,channel_layout,codec_name input_eac3_atmos.mkv

# Atmos is typically carried in:
# - E-AC-3 (Dolby Digital Plus) with JOC
# - TrueHD with JOC
# - PCM with Dolby Atmos XML metadata
```

---

## 9. DTS:X & MPEG-H

### 9.1 DTS:X
- **Format:** Object-based audio
- **Channel beds:** Up to 7.1 or 9.1
- **Objects:** Up to 32 simultaneous objects
- **Metadata:** 3D position (X, Y, Z), size, elevation
- **Rendering:** Automatically adapts to speaker layout

### 9.2 MPEG-H
- **Standard:** ITU-T H.265 / ISO/IEC 23008-3 (Part 3)
- **Channel beds:** Up to 7.1.4
- **Objects:** Full object support
- **Use case:** ATSC 3.0 broadcast, 3D Audio for BDAV

---

## 10. BINAURAL AUDIO

### 10.1 Overview
Binaural audio uses stereo headphone playback to simulate 3D spatial audio by capturing or synthesizing HRTF (Head-Related Transfer Functions).

### 10.2 HRTF Fundamentals
HRTF describes how sound is filtered by the head, pinna, and torso before reaching the eardrums:

```
HRTF(θ, φ) = H(θ, φ, source) / H(direct)

where:
  θ = azimuth angle (–180° to +180°)
  φ = elevation angle (–90° to +90°)
  H = acoustic transfer function
```

**Key HRTF characteristics:**
- **Interaural time difference (ITD):** 0–0.7 ms based on head width
- **Interaural level difference (ILD):** 0–20 dB, especially at high frequencies
- **Pinna cues:** Elevation and front/back discrimination

### 10.3 FFmpeg Binaural Processing
```bash
# Apply binaural downmix from 5.1:
ffmpeg -i input_5_1.wav \
  -af " binauralout=fc=0|w=0.8|h=0.5 " \
  output_binaural.wav
```

---

## 11. AMBISONICS

### 11.1 Overview
Ambisonics is a full-sphere surround sound format that encodes spatial information independently of speaker layout.

### 11.2 B-Format
First-order Ambisonics uses 4 channels:
| Channel | Description |
|---------|-------------|
| W | Omnidirectional (sum of all directions) |
| X | Figure-eight, forward-facing |
| Y | Figure-eight, left-facing |
| Z | Figure-eight, upward-facing |

### 11.3 Ambisonics Channel Order (FuMa)
```
1: W (omnidirectional)
2: X (forward-back)
3: Y (left-right)
4: Z (up-down)
```

### 11.4 FFmpeg Ambisonics Support
```bash
# Check for ambisonics metadata:
ffprobe -v quiet -show_entries stream=channels,channel_layout input_ambisonics.wav

# Decode ambisonics to stereo (first-order):
ffmpeg -i input_ambisonics.wav \
  -af "aformat=channel_layouts=4.0,amix=inputs=4:duration=first:weights=1 1 1 1" \
  output_stereo.wav
```

---

## 12. LFE CROSSOVER FREQUENCY

### 12.1 Theory
The LFE channel covers only low frequencies (typically 20–120 Hz). Main speakers should be crossed over to a subwoofer at a frequency where:
- Main speakers cannot reproduce bass efficiently
- Room acoustics cause problems at low frequencies
- LFE can extend system bass response

### 12.2 Typical Crossover Frequencies
| Speaker Size | Recommended Crossover | Notes |
|-------------|---------------------|-------|
| Large (8"+ woofers) | 40–60 Hz | Full-range main speakers |
| Medium (5–6" woofers) | 60–80 Hz | Common for bookshelf speakers |
| Small (3–4" woofers) | 80–120 Hz | Desktop/satellite systems |
| Bookshelf | 80 Hz | Standard THX crossover |

### 12.3 LFE in Codecs
| Codec | LFE Handling | Notes |
|-------|-------------|-------|
| AC-3 (Dolby Digital) | LFE channel present | 3–120 Hz bandwidth |
| DTS | LFE channel present | 20–80 Hz bandwidth |
| AAC | LFE channel present | Limited in some profiles |
| Opus | No dedicated LFE | Mono bass extraction required |
| WAV/PCM | No standard LFE | Channel mapping defines it |

---

## 13. IMPLEMENTATION CHECKLIST

### 13.1 Channel Layout Handling
- [ ] Support all standard channel layouts (mono, stereo, 5.0, 5.1, 7.0, 7.1)
- [ ] Handle LFE channel correctly (gain +10 dB, bandwidth)
- [ ] Implement channel remapping between different standards
- [ ] Detect channel layout from container metadata
- [ ] Handle files with ambiguous or non-standard channel counts

### 13.2 Downmix Implementation
- [ ] Implement 5.1 → stereo with proper coefficients
- [ ] Implement 5.1 → mono
- [ ] Implement 7.1 → 5.1
- [ ] Support Lt/Rt (Pro Logic) downmix for matrix-encoded audio
- [ ] Preserve LFE in downmix where appropriate

### 13.3 FFmpeg Integration
- [ ] Use FFmpeg channel layout constants correctly
- [ ] Implement channel remapping with `pan` and `channelmap` filters
- [ ] Handle upmix (stereo → 5.1) with care
- [ ] Verify channel order consistency across formats

### 13.4 Quality Verification
- [ ] Test downmix with known reference material
- [ ] Verify LFE gain is correct
- [ ] Check for phase issues in downmix
- [ ] Validate speaker position metadata in output

---

## 14. REFERENCE SPECIFICATIONS & FURTHER READING

### Primary Specifications
- **ITU-R BS.775-3:** https://www.itu.int/dms_pubrec/itu-r/rec/bs/R-REC-BS.775-3-201208-S!!PDF-E.pdf
- **ITU-R BS.2159-6:** Multichannel sound technology
- **SMPTE ST 428-12:** D-Cinema Audio Channel Assignment and Soundfield Level
- **Dolby Metadata Guide:** https://professionalsupport.dolby.com/

### Technical Resources
- Audio Channel Layout Reference: https://一致
- FFmpeg Channel Layout Documentation
- Dolby Atmos Home Entertainment Installation Guide

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*

---

## 13. IMMERSIVE AUDIO — HEIGHT CHANNELS & 3D LAYOUTS

### 13.1 Overview of Immersive Audio
Immersive audio extends traditional channel-based layouts with height channels, creating a three-dimensional soundfield. The most common configurations are based on the 5.1/7.1 foundation with additional overhead speakers.

### 13.2 Height Channel Placement Standards
Height channel placement follows specific angle and elevation standards:

| Configuration | Height Speaker Count | Elevation Angle | Positioning |
|--------------|---------------------|----------------|-------------|
| 5.1.2 | 2 | +30° to +45° | Front height (L, R) |
| 5.1.4 | 4 | +30° to +45° | Front and rear height |
| 7.1.2 | 2 | +30° to +45° | Front height (L, R) |
| 7.1.4 | 4 | +30° to +45° | Front and rear height |
| 9.1.6 | 6 | +30° to +60° | Front, side, and rear height |

### 13.3 Speaker Position Mathematical Calculation
Speaker positions can be calculated using spherical coordinates:

```python
# Calculate speaker position from angle and distance:
import math

def speaker_position(azimuth_deg, elevation_deg, distance_m):
    """
    Convert spherical to Cartesian coordinates.
    
    Args:
        azimuth_deg: Horizontal angle from front (degrees)
        elevation_deg: Vertical angle from horizontal plane (degrees)
        distance_m: Distance from listening position (meters)
    
    Returns:
        (x, y, z) Cartesian coordinates
    """
    az_rad = math.radians(azimuth_deg)
    el_rad = math.radians(elevation_deg)
    
    x = distance_m * math.cos(el_rad) * math.cos(az_rad)
    y = distance_m * math.cos(el_rad) * math.sin(az_rad)  # Depth
    z = distance_m * math.sin(el_rad)  # Height
    
    return (x, y, z)

# Example: Front left height speaker
# Azimuth: -45° (left side)
# Elevation: +30° (30 degrees up)
# Distance: 1.5 meters
x, y, z = speaker_position(-45, 30, 1.5)
print(f"FLH position: x={x:.2f}m, y={y:.2f}m, z={z:.2f}m")
```

### 13.4 Dolby Atmos Home Speaker Configurations
Dolby specifies speaker configurations for home Atmos:

#### 5.1.2 Configuration (Minimum Atmos)
```
Top-down view:
       [TFL] [TFR]
           \ /
  [BL]---[C]---[BR]
   |     / \     |
   |    /   \    |
  [L]-----------[R]
  
Front view:
    [TFL]       [TFR]
      \          /
       \        /
        [C]  [C]
       /        \
  [L]------------[R]
  
Side view:
    [TFL]
       |
       | height
       |
      [L]
```

### 13.5 Atmos Rendering to Speaker Configurations
Atmos renders object audio to any speaker configuration:

```python
# Object rendering algorithm (simplified):
def render_object_to_speakers(obj_x, obj_y, obj_z, speaker_positions):
    """
    Render an audio object to speakers based on position.
    
    Args:
        obj_x, obj_y, obj_z: Object position in 3D space
        speaker_positions: List of (azimuth, elevation, gain) tuples
    
    Returns:
        Dictionary mapping speaker name to gain
    """
    gains = {}
    for name, azimuth, elevation in speaker_positions:
        # Calculate angular distance from object
        angle_diff = angular_distance(
            (obj_x, obj_y, obj_z),
            (azimuth, elevation)
        )
        
        # Use panning law (e.g., VBAP, ACN/SN3D)
        gain = max(0, 1 - angle_diff / 180)
        gains[name] = gain
    
    # Normalize gains
    total = sum(gains.values())
    if total > 0:
        gains = {k: v / total for k, v in gains.items()}
    
    return gains

def angular_distance(pos1, pos2):
    """Calculate angular distance between two 3D positions."""
    # Simplified - actual implementation uses proper spherical math
    az1, el1 = pos1[0], pos1[1]
    az2, el2 = pos2[0], pos2[1]
    
    # Haversine-like formula for spherical coordinates
    daz = abs(az1 - az2)
    delev = abs(el1 - el2)
    
    return math.sqrt(daz**2 + delev**2)
```

### 13.6 DTS:X Speaker Configurations
DTS:X uses flexible speaker configurations based on MDA (Multi-Dimensional Audio):

| Configuration | Description | Channels |
|--------------|-------------|----------|
| 5.1 | Standard surround | L, C, R, Ls, Rs, LFE |
| 7.1 | Extended surround | L, C, R, Ls, Rs, Lb, Rb, LFE |
| 9.1.6 | Full immersive | L, C, R, Ls, Rs, Lb, Rb, TFL, TFR, TSL, TSR, TBL, TBR, LFE |
| 11.1 | Enhanced front | L, C, R, TFC, Ls, Rs, Lb, Rb, TFL, TFR, TSL, TSR, LFE |

### 13.7 MPEG-H Speaker Layouts
MPEG-H (ITU-T H.265 audio part) defines flexible layouts:

| Layout Name | Channels | Description |
|-------------|----------|-------------|
| 0+5.0 | 5 | Standard 5.0 |
| 0+5.1 | 6 | Standard 5.1 |
| 2+5.1 | 8 | 5.1 + 2 height |
| 4+5.1 | 10 | 5.1 + 4 height |
| 4+5.1+7.1 | 18 | Dual 5.1 + 4 height |

### 13.8 Auro-3D Layouts
Auro-3D uses a three-layer approach:

| Layer | Height | Description |
|-------|--------|-------------|
| Ear level | 0° | Standard 5.1/7.1 base |
| Voice level | +30° | "Voice" channel for dialogue clarity |
| Overhead | +45° to +90° | "Heaven" layer for immersive effects |

```
Auro-3D 13.1 Layout:

Top-down:
         [TMR]
    [TFL]   [TFR]
      \       /
[TML]---(C)---[TMR]
       /   \
  [BL]       [BR]
   |    \ /    |
   |    (C)    |
  [L]---------[R]
  [LFE]

Side view:
    [T] (Heaven)
     |
    [V] (Voice)
     |
    [B] (Base)
```

---

## 14. CHANNEL REMAPPING & FORMAT CONVERSION

### 14.1 Channel Remapping Fundamentals
Channel remapping converts audio between different channel layouts:

```python
# Channel remapping lookup table:
REMAP_TABLE = {
    # From: To: Mapping
    ('stereo', 'mono'): {
        'mono': lambda L, R: 0.707 * L + 0.707 * R  # Equal mix
    },
    ('mono', 'stereo'): {
        'L': lambda M: M,
        'R': lambda M: M
    },
    ('5.1', 'stereo'): {
        'L': lambda FL, FR, FC, LFE, BL, BR: FL + 0.707 * FC + 0.707 * BL,
        'R': lambda FL, FR, FC, LFE, BL, BR: FR + 0.707 * FC + 0.707 * BR
    },
    ('stereo', '5.1'): {
        'FL': lambda L, R: L,
        'FR': lambda L, R: R,
        'FC': lambda L, R: 0.5 * L + 0.5 * R,
        'LFE': lambda L, R: 0.0,
        'BL': lambda L, R: 0.5 * L,
        'BR': lambda L, R: 0.5 * R
    }
}
```

### 14.2 FFmpeg Channel Remap Filter
```bash
# Remap channels using pan filter:
# Convert 5.1 to stereo with custom coefficients:
ffmpeg -i input_5.1.wav \
  -af "pan=stereo|FL=FL+0.5*FC+0.3*BL|FR=FR+0.5*FC+0.3*BR" \
  output_stereo.wav

# Swap front left and front right:
ffmpeg -i input_stereo.wav \
  -af "pan=stereo|FL=FR|FR=FL" \
  output_swapped.wav

# Extract center channel:
ffmpeg -i input_5.1.wav \
  -af "pan=mono|c0=FC" \
  output_center.wav

# Create 5.1 from stereo with matrix:
ffmpeg -i input_stereo.wav \
  -af "pan=5.1|c0=FL|c1=FR|c2=FC|c3=LFE|c4=BL|c5=BR" \
  output_5.1.wav
```

### 14.3 Surround Sound Upmixing
Upmixing creates additional channels from fewer:

```python
def upmix_stereo_to_5_1(L, R):
    """
    Upmix stereo to 5.1 using passive matrix.
    """
    # Calculate center and surround
    C = 0.707 * (L + R)  # Center mix
    S = 0.707 * (L - R)  # Difference (surround)
    
    # Calculate front left and right with center
    FL = L
    FR = R
    
    # Surround channels (delayed and filtered in practice)
    BL = 0.7 * S
    BR = -0.7 * S  # Opposite phase
    
    # No LFE in stereo upmix
    LFE = 0.0
    
    return FL, FR, C, LFE, BL, BR

def adaptive_upmix(L, R, analysis):
    """
    Adaptive upmix using spectral analysis.
    Uses more center channel for vocals, less for stereo instruments.
    """
    # Detect vocal/mono content
    vocal_present = detect_vocals(analysis)
    
    if vocal_present:
        # Use more center for vocals
        C = 0.8 * (L + R)
        FL = L - 0.4 * C
        FR = R - 0.4 * C
    else:
        # Minimal center for stereo instruments
        C = 0.5 * (L + R)
        FL = L - 0.2 * C
        FR = R - 0.2 * C
    
    return FL, FR, C, 0.0, 0.5 * (L - R), -0.5 * (L - R)
```

### 14.4 Pro Logic Decoding
```bash
# Decode Lt/Rt to 5.1 using FFmpeg:
ffmpeg -i input_prologic.wav \
  -af "extrastereo=m=1.5,stereotools=mode=sidefade" \
  output_5.1.wav

# Better: Use a proper PLII decoder
# Note: FFmpeg doesn't include PLII decoder
# Use SoX or a dedicated plugin
```

---

## 15. SPATIAL AUDIO & BINAURAL RENDERING

### 15.1 Head-Related Transfer Functions (HRTF)
HRTF describes how sound is modified by the head, pinna, and torso:

```python
class HRTF:
    """HRTF-based spatial audio renderer."""
    
    def __init__(self, hrtf_database='default'):
        self.hrtf_db = self.load_hrtf(hrtf_database)
    
    def render_binaural(self, audio, azimuth, elevation):
        """
        Render mono audio to binaural using HRTF.
        
        Args:
            audio: Mono audio samples
            azimuth: Horizontal angle (-180 to +180 degrees)
            elevation: Vertical angle (-90 to +90 degrees)
        
        Returns:
            Stereo audio (binaural)
        """
        # Find closest HRTF in database
        hrtf_l, hrtf_r = self.get_hrtf(azimuth, elevation)
        
        # Convolve with HRTF
        left = convolve(audio, hrtf_l)
        right = convolve(audio, hrtf_r)
        
        return interleave(left, right)
    
    def get_hrtf(self, azimuth, elevation):
        """Get HRTF filters for given position."""
        # Interpolation between database points
        pass  # Implementation-specific
```

### 15.2 Common HRTF Databases
| Database | Subjects | Resolution | Notes |
|---------|----------|------------|-------|
| CIPIC | 45 subjects | 25° azimuth, variable elevation | Widely used |
| KEMAR | 2 subjects | 5° azimuth, variable | Diffuse field compensated |
| IRCAM | 3 subjects | 5° azimuth, 5° elevation | High resolution |
| 3D3A | 2 subjects | 5° azimuth, 5° elevation | Recent, high quality |

### 15.3 SOFA File Format
SOFA (Spatially Oriented Format for Acoustics) is the standard for HRTF storage:

```bash
# Check SOFA support in FFmpeg:
# FFmpeg doesn't natively support SOFA
# Use libraries like pysofaconventions

# Example using Python:
from pysofaconventions import SOFAFile

# Load HRTF from SOFA file
sf = SOFAFile('subject_001.sofa', 'r')
data = sf.getDataIR()  # Impulse responses
sr = sf.getSamplingRate()
positions = sf.getSourcePosition()

# Convolve audio with HRTF
import numpy as np
audio = np.load('input_mono.wav')
left = np.convolve(audio, data[:, 0], mode='full')[:len(audio)]
right = np.convolve(audio, data[:, 1], mode='full')[:len(audio)]
```

---

## 16. SPHERE & CUBE MICROPHONE PROCESSING

### 16.1 First-Order Ambisonics (FOA)
First-order Ambisonics uses 4 channels (WXYZ):

| Channel | Pattern | Description |
|---------|---------|-------------|
| W | Omnidirectional | Sum of all directions |
| X | Figure-8 (forward) | Left-right intensity difference |
| Y | Figure-8 (left) | Forward-back intensity difference |
| Z | Figure-8 (up) | Up-down intensity difference |

### 16.2 Ambisonics Encoding
```python
def encode_ambisonics(sources, directions):
    """
    Encode multiple sources into Ambisonics format.
    
    Args:
        sources: List of audio signals
        directions: List of (azimuth, elevation) tuples in radians
    
    Returns:
        W, X, Y, Z channel audio
    """
    W = sum(sources)  # Omnidirectional
    
    W_out = np.zeros_like(sources[0])
    X_out = np.zeros_like(sources[0])
    Y_out = np.zeros_like(sources[0])
    Z_out = np.zeros_like(sources[0])
    
    for source, (az, el) in zip(sources, directions):
        # Spherical harmonics encoding
        cos_el = np.cos(el)
        
        W_out += source
        X_out += source * cos_el * np.sin(az)
        Y_out += source * cos_el * np.cos(az)
        Z_out += source * np.sin(el)
    
    # Normalize for unit amplitude
    W_out *= 0.5
    X_out *= 1.5
    Y_out *= 1.5
    Z_out *= 1.5
    
    return W_out, X_out, Y_out, Z_out
```

### 16.3 Ambisonics Decoding
```python
def decode_ambisonics(W, X, Y, Z, speaker_dirs):
    """
    Decode Ambisonics to speaker channels.
    
    Args:
        W, X, Y, Z: Ambisonics channels
        speaker_dirs: List of (azimuth, elevation) tuples
    
    Returns:
        Array of speaker signals [n_speakers, n_samples]
    """
    n_speakers = len(speaker_dirs)
    outputs = []
    
    for az, el in speaker_dirs:
        # Simple mode-matching decoder
        cos_el = np.cos(el)
        
        # Max_rE weighting (common decoder method)
        rE = np.sqrt(W**2 + X**2 + Y**2 + Z**2)
        
        # Calculate speaker signal
        speaker = (
            0.5 * W +
            1.5 * (X * cos_el * np.sin(az) +
                   Y * cos_el * np.cos(az) +
                   Z * np.sin(el))
        )
        
        outputs.append(speaker)
    
    return np.array(outputs)
```

### 16.4 Higher-Order Ambisonics
Higher-order Ambisonics adds more channels for improved spatial resolution:

| Order | Channels | Description |
|-------|----------|-------------|
| 0 | 1 (W) | Omnidirectional only |
| 1 | 4 (WXYZ) | First-order |
| 2 | 9 | 5 second-order channels |
| 3 | 16 | 7 third-order channels |
| 4 | 25 | 9 fourth-order channels |

```
Number of channels formula:
N_channels = (order + 1)²
```

---

## 17. COMPLETE CHANNEL LAYOUT REFERENCE

### 17.1 All Standard Layouts
| Channels | Layout | Channel Order |
|----------|--------|--------------|
| 1 | mono | C |
| 2 | stereo | L, R |
| 3 | 2.1 | L, R, LFE |
| 3 | 3.0 | L, C, R |
| 3 | 3.0(back) | L, R, Cs |
| 4 | 4.0 | FL, FR, RL, RR |
| 4 | quad | FL, FR, RL, RR |
| 4 | 4.0(side) | FL, FR, SL, SR |
| 4 | 3.1 | L, R, C, LFE |
| 5 | 5.0 | FL, FR, FC, BL, BR |
| 5 | 5.0(back) | FL, FR, FC, BL, BR |
| 5 | 5.0(side) | FL, FR, FC, SL, SR |
| 6 | 5.1 | FL, FR, FC, LFE, BL, BR |
| 6 | 5.1(back) | FL, FR, FC, LFE, BL, BR |
| 6 | 5.1(side) | FL, FR, FC, LFE, SL, SR |
| 7 | 6.0 | FL, FR, FC, BC, SL, SR |
| 7 | 6.0(front) | FL, FR, FC, FLC, FRC, BC |
| 7 | 7.0 | FL, FR, FC, BL, BR, SL, SR |
| 7 | 7.0(back) | FL, FR, FC, BL, BR, SL, SR |
| 7 | 7.0(front) | FL, FLC, FRC, FR, FC, BL, BR |
| 8 | 7.1 | FL, FR, FC, LFE, BL, BR, SL, SR |
| 8 | 7.1(back) | FL, FR, FC, LFE, BL, BR, SL, SR |
| 8 | 7.1(wide) | FL, FLC, FR, FRC, FC, LFE, BL, BR |
| 8 | 7.1(wide-side) | FL, FLC, FR, FRC, FC, LFE, SL, SR |
| 10 | 7.1.2 | FL, FR, FC, LFE, BL, BR, SL, SR, TFL, TFR |
| 12 | 7.1.4 | FL, FR, FC, LFE, BL, BR, SL, SR, TFL, TFR, TBL, TBR |
| 14 | 9.1.4 | FL, FLC, FR, FRC, FC, LFE, BL, BR, SL, SR, TFL, TFR, TBL, TBR |
| 16 | 9.1.6 | FL, FLC, FR, FRC, FC, LFE, BL, BR, SL, SR, TFL, TFR, TML, TMR, TBL, TBR |

### 17.2 Professional Studio Configurations
| Name | Channels | Typical Use |
|------|----------|-------------|
| Music studio (stereo) | 2.0 | Music mixing |
| Music studio (5.1) | 5.1 | Film music scoring |
| Film dubbing stage | 5.1 / 7.1 | Film post-production |
| Broadcast studio | 5.1 | TV production |
| Cinema | 7.1 / Atmos | Movie theater |

### 17.3 Home Theater Configurations
| Name | Channels | Typical Setup |
|------|----------|--------------|
| Basic | 2.0 | TV speakers |
| Entry | 3.1 | Soundbar + sub |
| Standard | 5.1 | 5.1 receiver + speakers |
| Premium | 7.1 | Extended surround |
| Immersive | 5.1.2 / 7.1.4 | Atmos/DTS:X capable |

---

## 18. IMPLEMENTATION CHECKLIST (EXPANDED)

### 18.1 Basic Channel Layout Support
- [x] Support mono (1.0)
- [x] Support stereo (2.0)
- [x] Support 2.1 (L, R, LFE)
- [x] Support 5.0 (FL, FR, FC, BL, BR)
- [x] Support 5.1 (FL, FR, FC, LFE, BL, BR)
- [x] Support 7.0 (FL, FR, FC, BL, BR, SL, SR)
- [x] Support 7.1 (FL, FR, FC, LFE, BL, BR, SL, SR)

### 18.2 Advanced Channel Layout Support
- [ ] Support 5.1.2, 5.1.4 (height channels)
- [ ] Support 7.1.2, 7.1.4 (height channels)
- [ ] Support side vs back channel differentiation
- [ ] Support wide channel configurations
- [ ] Support Auro-3D layouts

### 18.3 Downmix/Upmix Implementation
- [x] 5.1 → stereo downmix
- [x] 5.1 → mono downmix
- [x] 7.1 → 5.1 downmix
- [ ] Stereo → 5.1 upmix
- [ ] Lt/Rt Pro Logic encoding
- [ ] Lt/Rt Pro Logic II decoding

### 18.4 FFmpeg Integration
- [x] Use pan filter for channel remapping
- [x] Use channelmap filter for simple remapping
- [x] Detect layout from container metadata
- [ ] Implement custom downmix coefficients
- [ ] Handle LFE correctly (+10 dB, low-pass)

### 18.5 Spatial Audio Support
- [ ] First-order Ambisonics (WXYZ) encoding
- [ ] Ambisonics decoding to speaker layouts
- [ ] HRTF-based binaural rendering
- [ ] Atmos object audio processing
- [ ] DTS:X object audio processing

### 18.6 Quality Assurance
- [ ] Test with known reference material (5.1 music, film)
- [ ] Verify channel order consistency
- [ ] Verify LFE gain is correct (+10 dB)
- [ ] Test downmix quality with music and film
- [ ] Validate speaker metadata in output

---

## 19. REFERENCE IMPLEMENTATIONS

### 19.1 FFmpeg Channel Layout Constants (Complete)
```c
// From libavutil/channel_layout.h (FFmpeg 6.x)
enum {
    AV_CH_LAYOUT_MONO = (AV_CH_FRONT_CENTER),
    AV_CH_LAYOUT_STEREO = (AV_CH_FRONT_LEFT | AV_CH_FRONT_RIGHT),
    AV_CH_LAYOUT_2POINT1 = (AV_CH_FRONT_LEFT | AV_CH_FRONT_RIGHT | AV_CH_LOW_FREQUENCY),
    AV_CH_LAYOUT_2_1 = (AV_CH_FRONT_LEFT | AV_CH_FRONT_RIGHT | AV_CH_BACK_CENTER),
    AV_CH_LAYOUT_QUAD = (AV_CH_FRONT_LEFT | AV_CH_FRONT_RIGHT | AV_CH_BACK_LEFT | AV_CH_BACK_RIGHT),
    AV_CH_LAYOUT_4POINT0 = (AV_CH_FRONT_LEFT | AV_CH_FRONT_RIGHT | AV_CH_FRONT_CENTER | AV_CH_BACK_CENTER),
    AV_CH_LAYOUT_5POINT0 = (AV_CH_FRONT_LEFT | AV_CH_FRONT_RIGHT | AV_CH_FRONT_CENTER | AV_CH_BACK_LEFT | AV_CH_BACK_RIGHT),
    AV_CH_LAYOUT_5POINT0_SIDE = (AV_CH_FRONT_LEFT | AV_CH_FRONT_RIGHT | AV_CH_FRONT_CENTER | AV_CH_SIDE_LEFT | AV_CH_SIDE_RIGHT),
    AV_CH_LAYOUT_4POINT1 = (AV_CH_FRONT_LEFT | AV_CH_FRONT_RIGHT | AV_CH_FRONT_CENTER | AV_CH_BACK_CENTER | AV_CH_LOW_FREQUENCY),
    AV_CH_LAYOUT_5POINT1 = (AV_CH_FRONT_LEFT | AV_CH_FRONT_RIGHT | AV_CH_FRONT_CENTER | AV_CH_LOW_FREQUENCY | AV_CH_BACK_LEFT | AV_CH_BACK_RIGHT),
    AV_CH_LAYOUT_5POINT1_SIDE = (AV_CH_FRONT_LEFT | AV_CH_FRONT_RIGHT | AV_CH_FRONT_CENTER | AV_CH_LOW_FREQUENCY | AV_CH_SIDE_LEFT | AV_CH_SIDE_RIGHT),
    AV_CH_LAYOUT_7POINT0 = (AV_CH_FRONT_LEFT | AV_CH_FRONT_RIGHT | AV_CH_FRONT_CENTER | AV_CH_SIDE_LEFT | AV_CH_SIDE_RIGHT | AV_CH_BACK_LEFT | AV_CH_BACK_RIGHT),
    AV_CH_LAYOUT_7POINT0_FRONT = (AV_CH_FRONT_LEFT | AV_CH_FRONT_CENTER | AV_CH_FRONT_RIGHT | AV_CH_BACK_LEFT | AV_CH_BACK_RIGHT | AV_CH_SIDE_LEFT | AV_CH_SIDE_RIGHT),
    AV_CH_LAYOUT_7POINT1 = (AV_CH_FRONT_LEFT | AV_CH_FRONT_RIGHT | AV_CH_FRONT_CENTER | AV_CH_LOW_FREQUENCY | AV_CH_BACK_LEFT | AV_CH_BACK_RIGHT | AV_CH_SIDE_LEFT | AV_CH_SIDE_RIGHT),
    AV_CH_LAYOUT_7POINT1_WIDE = (AV_CH_FRONT_LEFT | AV_CH_FRONT_CENTER | AV_CH_FRONT_RIGHT | AV_CH_BACK_LEFT | AV_CH_BACK_RIGHT | AV_CH_LOW_FREQUENCY | AV_CH_SIDE_LEFT | AV_CH_SIDE_RIGHT),
    AV_CH_LAYOUT_7POINT1_WIDE_SIDE = (AV_CH_FRONT_LEFT | AV_CH_FRONT_CENTER | AV_CH_FRONT_RIGHT | AV_CH_BACK_LEFT | AV_CH_BACK_RIGHT | AV_CH_LOW_FREQUENCY | AV_CH_SIDE_LEFT | AV_CH_SIDE_RIGHT),
};
```

### 19.2 Channel Mask Definitions
```c
// Speaker position bitmasks
#define AV_CH_FRONT_LEFT             0x00000001
#define AV_CH_FRONT_RIGHT            0x00000002
#define AV_CH_FRONT_CENTER           0x00000004
#define AV_CH_LOW_FREQUENCY          0x00000008
#define AV_CH_BACK_LEFT              0x00000010
#define AV_CH_BACK_RIGHT             0x00000020
#define AV_CH_FRONT_LEFT_OF_CENTER   0x00000040
#define AV_CH_FRONT_RIGHT_OF_CENTER  0x00000080
#define AV_CH_BACK_CENTER            0x00000100
#define AV_CH_SIDE_LEFT              0x00000200
#define AV_CH_SIDE_RIGHT             0x00000400
#define AV_CH_TOP_CENTER             0x00000800
#define AV_CH_TOP_FRONT_LEFT         0x00001000
#define AV_CH_TOP_FRONT_CENTER       0x00002000
#define AV_CH_TOP_FRONT_RIGHT        0x00004000
#define AV_CH_TOP_BACK_LEFT          0x00008000
#define AV_CH_TOP_BACK_CENTER        0x00010000
#define AV_CH_TOP_BACK_RIGHT         0x00020000
#define AV_CH_STEREO_LEFT            0x80000000  // Difference channel
#define AV_CH_STEREO_RIGHT          0x40000000  // Sum channel
```

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*Additional sections: Immersive audio, spatial audio, Ambisonics, complete layout reference*

---

## 20. DOLBY PRO LOGIC — MATRIX DECODING

### 20.1 Matrix Encoding Fundamentals
Matrix encoding combines multiple audio channels into fewer channels for transmission or storage:

```python
def dolby_stereo_encode(L, C, R, S):
    """
    Dolby Stereo matrix encoding.
    Produces Lt/Rt for transmission.
    """
    Lt = L + 0.707 * C + 0.707 * S
    Rt = R + 0.707 * C - 0.707 * S
    
    return Lt, Rt

def dolby_stereo_decode(Lt, Rt):
    """
    Basic Dolby Stereo matrix decoding.
    """
    L = Lt
    R = Rt
    C = 0.707 * (Lt + Rt)
    S = 0.707 * (Lt - Rt)
    
    return L, C, R, S
```

### 20.2 Pro Logic II Decoding
Pro Logic II improves on basic matrix decoding:

```python
def pro_logic_ii_decode(Lt, Rt, mode='auto'):
    """
    Pro Logic II decoder with music/cinema modes.
    """
    # Determine dominant content
    if mode == 'auto':
        # Analyze Lt/Rt correlation
        correlation = calculate_correlation(Lt, Rt)
        if correlation > 0.7:
            mode = 'stereo'
        else:
            mode = 'cinema'
    
    if mode == 'cinema':
        # Stronger center channel
        C_gain = 1.0
        S_gain = 1.0
    else:  # music mode
        # More subtle center/surround
        C_gain = 0.7
        S_gain = 0.5
    
    # Decode with adaptive gains
    L = Lt
    R = Rt
    C = C_gain * 0.707 * (Lt + Rt)
    S = S_gain * 0.707 * (Lt - Rt)
    
    return L, C, R, S

def calculate_correlation(signal1, signal2):
    """Calculate correlation coefficient between signals."""
    mean1 = np.mean(signal1)
    mean2 = np.mean(signal2)
    
    n = len(signal1)
    cov = np.sum((signal1 - mean1) * (signal2 - mean2)) / n
    
    std1 = np.sqrt(np.sum((signal1 - mean1)**2) / n)
    std2 = np.sqrt(np.sum((signal2 - mean2)**2) / n)
    
    if std1 > 0 and std2 > 0:
        return cov / (std1 * std2)
    return 0
```

### 20.3 Lt/Rt vs Lo/Ro
| Aspect | Lt/Rt (Matrix) | Lo/Ro (Discrete) |
|--------|----------------|-----------------|
| Channels | 4 encoded to 2 | 4 discrete |
| Compatibility | Plays on stereo | Requires decoder |
| Quality | Some separation loss | Full separation |
| Codec | AC-3, DTS matrix | Discrete codecs |

### 20.4 Surround Channel Processing
```python
def process_surround(channel, sample_rate):
    """
    Apply Dolby surround processing to rear channel.
    Includes:
    - Bandpass filter (7kHz lowpass)
    - Rearward phase shift
    - Delay (20ms)
    """
    from scipy import signal
    
    # 7kHz lowpass filter
    sos = signal.butter(6, 7000, 'lowpass', fs=sample_rate, output='sos')
    channel = signal.sosfilt(sos, channel)
    
    # 20ms delay
    delay_samples = int(0.020 * sample_rate)
    channel = np.pad(channel, (delay_samples, 0))[:len(channel)]
    
    # Phase shift (simplified)
    # Actual implementation uses Hilbert transform
    channel = apply_phase_shift(channel, degrees=90)
    
    return channel

def apply_phase_shift(audio, degrees=90):
    """Apply phase shift to audio."""
    from scipy.signal import hilbert
    
    analytic = hilbert(audio)
    phase = np.angle(analytic)
    shifted_phase = phase + np.radians(degrees)
    
    return np.abs(analytic) * np.cos(shifted_phase)
```

---

## 21. DTS NEO ENCODING/DECODING

### 21.1 DTS Neo Overview
DTS Neo is the DTS equivalent of Dolby Pro Logic:

```python
def dts_neo_6_decode(Lt, Rt, mode='stereo'):
    """
    DTS Neo:6 decoder.
    Modes: 'stereo', 'cinema', 'music'
    """
    if mode == 'cinema':
        # 6-channel cinema mode
        C = 0.707 * (Lt + Rt)
        Ls = 0.707 * Lt
        Rs = 0.707 * Rt
        
    elif mode == 'music':
        # 6-channel music mode
        C = 0.5 * (Lt + Rt)
        Ls = 0.707 * Lt - 0.5 * Rt
        Rs = 0.707 * Rt - 0.5 * Lt
        
    else:  # stereo
        # Minimal processing
        C = 0.5 * (Lt + Rt)
        Ls = 0.7 * Lt
        Rs = 0.7 * Rt
    
    return Lt, Rt, C, Ls, Rs, 0.0  # No LFE
```

### 21.2 DTS Neo:6 vs Pro Logic II
| Feature | DTS Neo:6 | Dolby Pro Logic II |
|---------|-----------|-------------------|
| Channels | Up to 6.1 | Up to 7.1 |
| Cinema mode | Yes | Yes |
| Music mode | Yes | Yes |
| Center width control | No | Yes |
| Dimension control | No | Yes |
| Surround delay | Fixed | Adjustable |

---

## 22. LFE PROCESSING & CROSSOVER

### 22.1 LFE Channel Characteristics
The LFE (Low Frequency Effects) channel has specific requirements:

```python
LFE_SPECS = {
    'frequency_range': '20-120 Hz',  # Typical
    'bandwidth': '100 Hz',  # Limited in most codecs
    'gain_offset': '+10 dB',  # Relative to main channels
    'codec_support': {
        'AC-3': True,
        'DTS': True,
        'AAC': True,
        'Opus': False,  # No dedicated LFE
        'MP3': False,
        'Vorbis': False,
    }
}
```

### 22.2 LFE Crossover Configuration
```python
def calculate_crossover_settings(front_speaker_size, crossover_freq=80):
    """
    Calculate LFE crossover settings based on speaker size.
    
    Args:
        front_speaker_size: 'large', 'medium', 'small'
        crossover_freq: Target crossover frequency in Hz
    
    Returns:
        (lfe_enabled, lfe_cutoff, lfe_mix)
    """
    if front_speaker_size == 'large':
        # Full-range speakers, minimal LFE use
        lfe_enabled = True
        lfe_cutoff = 40  # Hz
        lfe_mix = 0.0  # Don't mix LFE to mains
    elif front_speaker_size == 'medium':
        lfe_enabled = True
        lfe_cutoff = 60
        lfe_mix = 0.0
    else:  # small
        lfe_enabled = True
        lfe_cutoff = crossover_freq
        lfe_mix = 0.0  # LFE doesn't feed main speakers
    
    return lfe_enabled, lfe_cutoff, lfe_mix
```

### 22.3 LFE in Various Codecs
| Codec | LFE Sample Rate | LFE Bandwidth | Notes |
|-------|---------------|---------------|-------|
| AC-3 | 1/192 of main | ~120 Hz | Highly limited |
| DTS | 1/64 of main | ~80 Hz | Limited |
| AAC | 1/4 of main | Variable | Profile-dependent |
| Opus | N/A | N/A | No dedicated LFE |
| WAV/PCM | Same as main | Full range | No standard LFE |

---

## 23. CHANNEL LAYOUT IN CONTAINERS

### 23.1 WAV/AIFF Channel Order
```
WAVEFORMATEX channel order for common layouts:

5.1 (Microsoft standard):
  Offset:  0  1  2  3  4  5
  Channel: FL FR FC LFE BL BR
  Bits:    1  2  3  4  5  6

7.1 (Microsoft standard):
  Offset:  0  1  2  3  4  5  6  7
  Channel: FL FR FC LFE BL BR SL SR
  Bits:    1  2  3  4  5  6  7  8
```

### 23.2 MP4/M4A Channel Order
```
MP4 channel layout (from ISO 14496-12):

Mono:     C
Stereo:   L  R
5.1:      FL FR FC LFE BL BR
7.1:      FL FR FC LFE BL BR SL SR
```

### 23.3 Matroska/MKA Channel Order
```
Matroska follows the same order as MP4:
  C, L, R, LFE, BL, BR, FL, FR (for some configs)

Check container documentation for specific codec support.
```

### 23.4 AC-3 Bitstream Channel Order
```
AC-3 (A52) channel order:
  Mode:    1+1  1/0  2/0  3/0  2/1  3/1  2/2  3/2  1/0+1  2/0+1  3/0+1  2/1+1  3/1+1  2/2+1  3/2+1
  Channels:L R   C    L R   L C R Ls Rs  L C R Ls Rs  C L R  C L R LFE LFE Ls Rs  L C R Ls Rs LFE LFE

AC-3 also uses LFE as channel 7 when present.
```

---

## 24. VERIFICATION & TESTING

### 24.1 Test Signals for Channel Layout
```python
def generate_channel_test_signals(sample_rate=48000, duration=1.0):
    """Generate test signals for each channel."""
    import numpy as np
    
    n_samples = int(sample_rate * duration)
    t = np.linspace(0, duration, n_samples, endpoint=False)
    
    # Channel-specific tones (different frequencies)
    signals = {
        'FL': np.sin(2 * np.pi * 440 * t),      # A4
        'FR': np.sin(2 * np.pi * 440 * t),
        'FC': np.sin(2 * np.pi * 880 * t),      # A5
        'LFE': np.sin(2 * np.pi * 60 * t),       # 60 Hz
        'BL': np.sin(2 * np.pi * 660 * t),       # E5
        'BR': np.sin(2 * np.pi * 660 * t),
        'SL': np.sin(2 * np.pi * 550 * t),       # C#5
        'SR': np.sin(2 * np.pi * 550 * t),
    }
    
    return signals
```

### 24.2 Channel Order Verification
```python
def verify_channel_order(audio, sample_rate, expected_layout):
    """
    Verify channel order by analyzing test signals.
    """
    from scipy import signal
    
    # Expected frequencies for each channel
    expected_freqs = {
        'FL': 440, 'FR': 440,
        'FC': 880, 'LFE': 60,
        'BL': 660, 'BR': 660,
        'SL': 550, 'SR': 550
    }
    
    # Detect frequency in each channel
    detected = {}
    for i, channel in enumerate(audio.T):
        freq = detect_frequency(channel, sample_rate)
        detected[i] = freq
    
    # Map detected frequencies to expected
    channel_map = {}
    for channel_name, expected_freq in expected_freqs.items():
        for idx, detected_freq in detected.items():
            if abs(detected_freq - expected_freq) < 10:  # Within 10 Hz
                channel_map[channel_name] = idx
                break
    
    return channel_map

def detect_frequency(signal_data, sample_rate):
    """Detect dominant frequency using FFT."""
    import numpy as np
    
    fft = np.fft.rfft(signal_data)
    freqs = np.fft.rfftfreq(len(signal_data), 1/sample_rate)
    
    # Find peak in meaningful range (20Hz-20kHz)
    mask = (freqs > 20) & (freqs < 20000)
    peak_idx = np.argmax(np.abs(fft[mask]))
    
    return freqs[mask][peak_idx]
```

### 24.3 Test Procedure
```bash
# 1. Generate test signals
python generate_test_wav.py --layout 5.1 --output test_5_1.wav

# 2. Verify with audio editor
ffmpeg -i test_5_1.wav -af "ashowinfo" -f null -

# 3. Check channel assignments
ffprobe -v quiet -show_streams test_5_1.wav | grep channels

# 4. Verify with known-good decoder
ffmpeg -i test_5_1.wav -af "pan=5.1|FL=FL|FR=FR|FC=FC|LFE=LFE|BL=BL|BR=BR" \
  -c:a pcm_s24le test_verify.wav

# 5. Check each channel individually
for ch in FL FR FC LFE BL BR; do
  ffmpeg -i test_verify.wav -af "pan=mono|c0=${ch}" "${ch}.wav"
  # Verify frequency in each file
done
```

---

## 25. SPECIFICATION REFERENCES

### 25.1 ITU Standards
| Standard | Title | Year |
|----------|-------|------|
| BS.775-1 | Multichannel stereophonic sound system | 1994 |
| BS.775-2 | idem | 1999 |
| BS.775-3 | idem | 2012 |
| BS.2159-6 | Multichannel sound technology | 2013 |

### 25.2 SMPTE Standards
| Standard | Title | Year |
|----------|-------|------|
| ST 428-12 | D-Cinema Audio Channel Assignment | 2006 |
| ST 432-1 | Digital Audio Bitstream Requirements | 2006 |

### 25.3 Dolby Specifications
| Specification | Description |
|--------------|-------------|
| Dolby Metadata Guide | Audio metadata for AC-3/E-AC-3 |
| Dolby Atmos白皮书 | Object-based audio specification |
| DD+ Bitstream Specification | Dolby Digital Plus specification |

### 25.4 DTS Specifications
| Specification | Description |
|--------------|-------------|
| DTS Coherent Acoustics | Core DTS specification |
| DTS-HD | High-resolution audio extension |
| DTS:X | Object-based audio specification |

---

*File expanded with: Pro Logic decoding, DTS Neo, LFE processing, container layouts, testing procedures, and full specification references*
