# Opus — Deep Technical Reference

> **Category:** Lossy/Hybrid Audio Codec (Hybrid: SILK + CELT)
> **File Extensions:** .opus, .oga
> **MIME Types:** audio/opus, audio/ogg (when in Ogg container)
> **Standardization Body:** IETF RFC 6716 / Xiph.org
> **Patent Status:** Royalty-free, covered by the Opus License (BSD-like)
> **License:** BSD 3-Clause / IETF
> **Specification:** RFC 6716 (September 2012); RFC 7845 (Ogg encapsulation, April 2016)
> **Reference Implementation:** libopus (Xiph.org) + libopusenc

---

## 1. HISTORICAL CONTEXT & ORIGIN

Opus is a lossy audio codec designed from the ground up for interactive real-time communication over the internet. It emerged from the merger of two separate IETF working group efforts: **CELT** (Constrained Energy Lapped Transform, from the Xiph.org Foundation) and **SILK** (developed by Skype / Microsoft). The working group selected the two technologies and combined them into a single codec that could span the full range from very low bitrates (~6 kbps) to near-transparent quality (~510 kbps).

### 1.1 Timeline

| Year | Milestone |
|------|-----------|
| 2002 | SILK development begins at Skype |
| 2007 | CELT development begins at Xiph.org |
| 2009 | IETF Codec Working Group formed; calls for codec proposals |
| 2010 | SILK and CELT submitted as candidates; hybrid proposal accepted |
| 2012 | RFC 6716 published (September 2012) — Opus is standardized |
| 2012 | Opus 1.0 released in libopus |
| 2014 | Opus selected as mandatory-to-implement audio codec for WebRTC (RFC 7874) |
| 2016 | RFC 7875 (Opus FEC), RFC 7845 (Ogg Opus) published |
| 2017 | Opus 1.2 — improvements to SILK, CELT, and stereo handling |
| 2018 | Opus 1.3 — enhanced stereo, better PLC, lower complexity |
| 2020 | Opus 1.4 — further encoding improvements |
| 2024 | Opus 1.5 — improved music performance, multi-channel |

### 1.2 Design Goals

Opus was explicitly designed to replace both Vorbis (for music streaming) and Speex (for voice) with a single codec. The primary design constraints were:

1. **Low algorithmic latency** — targeting 5 ms, 10 ms, 20 ms, 40 ms, or 60 ms frame sizes for real-time communication.
2. **Full audio bandwidth** — from 4 kHz (narrowband voice) to 20 kHz (fullband music).
3. **Scalability** — one codec that handles speech, music, and mixed content.
4. **Interoperability** — single bitstream format that any conforming decoder can decode.
5. **Royalty-free** — necessary for mandatory-to-implement (MTI) status in WebRTC.

### 1.3 SILK and CELT — The Two Engines

**SILK** is a speech-focused codec based on:
- Linear Prediction (LP) analysis, similar to CELP codecs
- Long-term prediction (LTP) for pitch harmonics
- Signal-to-noise ratio (SNR)-maximizing quantization
- Optimized for speech signals (voice fundamental ~300 Hz to 3.4 kHz)

**CELT** is a music-focused codec based on:
- Modified Discrete Cosine Transform (MDCT)
- Per-band energy (PBE) encoding
- Vector quantization with a coefficient of variation (CV) decision
- Optimized for music and general audio signals

The **hybrid mode** combines both: SILK handles the low frequencies (up to ~8 kHz) while CELT handles the high frequencies (above ~8 kHz). The decoder can seamlessly switch modes frame-by-frame.

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Dual-Mode Architecture

Opus is architecturally two codecs in one:

```
Input PCM (16-bit signed, 48 kHz, up to 8 channels)
        |
        v
+-------------------+
|  Bandwidth Check  |
+-------------------+
        |
   +----+----+
   |         |
   v         v
 SILK      CELT
 Narrowband/    MDCT-based
 Wideband/      fullband
 FB-MWV         transform
 (LP-based)     coding
   |              |
   +-------+------+
           |
           v
    +------+------+
    |  Opus Packet  |
    |  (entropy-coded |
    |   payload)     |
    +----------------+
```

### 2.2 Mode Decision

The encoder chooses which mode to use based on:

- **Input signal characteristics**: Speech vs. music classification
- **Bitrate**: Low bitrates favor SILK; high bitrates favor CELT
- **Application mode**: `OPUS_APPLICATION_RESTRICTED_LOWDELAY` forces CELT-only mode for ultra-low latency
- **Explicit mode signaling**: The `config` field in the Opus head packet specifies the mode

### 2.3 Frame Structure

Opus always operates at **48 kHz sample rate**. Input audio is resampled to 48 kHz internally. The codec supports multiple frame sizes:

| Frame Size (ms) | SILK Samples | CELT Samples | Notes |
|----------------|--------------|--------------|-------|
| 2.5 | 120 | — | SILK-only, very low latency |
| 5 | 240 | — | SILK-only |
| 10 | 480 | 512 (MDCT with 50% overlap) | Standard low-latency |
| 20 | 960 | 512 (×2 frames) | Default for most applications |
| 40 | 1920 | 512 (×4 frames) | Higher latency, better compression |
| 60 | 2880 | 512 (×6 frames) | Maximum latency |

Effective frame duration in CELT mode is always 512 samples (10.67 ms at 48 kHz), but the encoder can code multiple CELT frames per Opus packet.

### 2.4 Bitstream vs. Container

The Opus **bitstream format** (RFC 6716) is the wire/codec-level representation: a sequence of Opus packets. The **container format** (RFC 7845 for Ogg, WebM/Matroska) wraps the bitstream with additional headers, timing, and seeking information. Always distinguish between:
- **Opus bitstream** — raw Opus packets
- **Ogg Opus** — Ogg container with OpusHead + OpusTags + Opus packets
- **Opus in WebM/Matroska** — MKA container with CodecPrivate + Opus packets

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 Opus Head Packet (Container Header)

The OpusHead packet is the identification header placed at the beginning of an Opus stream in Ogg or WebM containers. It is **not** part of the codec bitstream itself — it is container-level metadata.

**Size:** 19 bytes (fixed)

**Byte layout:**

```
 Byte:  0        1-7                8           9         10-11    12-18
       +---------+------------------+-----------+----------+--------+
       | 'OpusH' | Magic signature  |  Version  | Channels | Pre-skip|
       |(0x4F70757348 "OpusHead")  | (0x01)    |(u8)      | (u16 LE)|
       +---------+------------------+-----------+----------+--------+

 Bytes 12-17: pre-skip (continued)
 Bytes 18-25: input sample rate (u32 LE)
 Bytes 26-27: output gain (s16 LE)
 Byte  28:    channel mapping family (u8)
 [Bytes 29+: channel mapping table — only present if channel mapping family != 0]
```

| Field | Type | Description |
|-------|------|-------------|
| `magic` | 8 bytes | ASCII "OpusHead" (no null terminator). Offset 0x00 |
| `version` | u8 | OpusHead version. Must be 1. All other values reserved. Offset 0x08 |
| `channels` | u8 | Number of coded audio channels (1-255). Offset 0x09 |
| `pre-skip` | u16 LE | **Critical.** Number of 48 kHz samples to discard from the **decoder output** before starting playback. Accounts for encoder look-ahead delay. Required for gapless playback. Offset 0x0A |
| `input_sample_rate` | u32 LE | Original input sample rate (before any internal resampling). RFC 7845 mandates this should be the actual sample rate, but decoders are not required to use it. Offset 0x0C |
| `output_gain` | s16 LE | Gain to apply to decoded output, in Q7.8 dB (1 dB = 256). Negative values allowed. Offset 0x10 |
| `channel_mapping_family` | u8 | 0 = no channel mapping (mono/stereo); 1 = Vorbis channel ordering with mapping table; 2+ = reserved for future use. Offset 0x12 |
| `channel_mapping[]` | varies | Only present if `channel_mapping_family != 0`. See Section 12. |

**OpusHead example hex** (stereo, 48 kHz, pre-skip 3840, no mapping):

```
 4F 70 75 73 48 65 61 64   # "OpusHead" magic
 01                        # version = 1
 02                        # channels = 2
 0F 00                     # pre-skip = 3840 (0x0F00)
 80 BB 00 00               # input_sample_rate = 48000 (0x0000BB80)
 00 00                     # output_gain = 0
 00                        # channel_mapping_family = 0 (no mapping)
```

**Pre-skip explanation:** SILK introduces encoder lookahead (up to 512 samples) and CELT uses MDCT with overlapping windows. The combined encoder lookahead can be 512–960 samples. The pre-skip value tells the decoder how many decoded samples to discard from the beginning of the stream to achieve sample-accurate alignment. The typical pre-skip value is **3840 samples** (80 ms at 48 kHz) for libopus at 20 ms frame size, but it varies by encoder settings. **Always use the actual pre-skip value from the OpusHead** — never hard-code a value.

### 3.2 Opus Tags Packet (Container Header)

The OpusTags packet is the metadata/comment header. It follows Vorbis comment conventions (RFC 7845 Section 5.2).

**Byte layout:**

```
 Bytes 0-7:   'OpusTags' magic (0x4F 0x70 0x75 0x73 0x54 0x61 0x67 0x73)
 Bytes 4-7:   vendor string length (u32 LE)
 Bytes 8-...: vendor string (UTF-8)
 ...:         user comment list length (u32 LE)
 [repeats]:   each comment: length (u32 LE) + UTF-8 string "KEY=VALUE"
```

**Vendor string:** Identifies the encoder software. libopus typically sets this to `"libopus <version>"` (e.g., `"libopus 1.5"`). The length field includes the full vendor string without a null terminator.

**Comment strings:** Each comment follows the Vorbis comment format: `KEY=VALUE`, encoded as UTF-8. Common fields:

| Field | Description | Example |
|-------|-------------|---------|
| `TITLE` | Track title | `TITLE=Sympathy for the Devil` |
| `ARTIST` | Artist name | `ARTIST=The Rolling Stones` |
| `ALBUM` | Album name | `ALBUM=Beggar's Banquet` |
| `DATE` | Release year | `DATE=1968` |
| `TRACKNUMBER` | Track number | `TRACKNUMBER=7` |
| `GENRE` | Genre | `GENRE=Rock` |
| `COMMENT` | Free text | `COMMENT=Live at Altamont` |
| `ENCODER` | Encoder identification | `ENCODER=Lavf60.12.100` |

**Opus Tags example:**

```
 4F 70 75 73 54 61 67 73   # "OpusTags" magic
 08 00 00 00               # vendor string length = 8
 6C 69 62 6F 70 75 73 20   # "libopus "
 31 2E 33                   # "1.3"
 02 00 00 00               # comment list length = 2

 # Comment 1
 11 00 00 00               # length = 17
 54 49 54 4C 45 3D 47 6F   # "TITLE=Go"
 6F 64 20 57 69 6C 6C 20   # "d Will"
 41 63 74 69 6F 6E         # "Action"

 # Comment 2
 0D 00 00 00               # length = 13
 41 52 54 49 53 54 3D 52   # "ARTIST=R"
 61 64 69 6F 68 65 61 64   # "adiohead"
```

### 3.3 Opus Packet Format (Bitstream)

An Opus packet is the core codec payload. The format is defined in RFC 6716 Section 3. The packet structure depends on the **TOC byte** (Table of Contents), which is the first byte of every Opus packet.

#### 3.3.1 TOC Byte Layout

```
  7   6   5   4   3   2   1   0
+-------+-------+---------------+
| config| stereo|  codebook /   |
|  code | flag  | frame count  |
| (5b)  | (1b)  |   (2b)       |
+-------+-------+---------------+

 config_code: mode(2) + bandwidth(3)
   mode:       0 = SILK-only, 1 = hybrid, 2 = CELT-only, 3 = reserved
   bandwidth:  0 = NB (4kHz), 1 = MB (6kHz), 2 = WB (8kHz),
               3 = SWB (12kHz), 4 = FB (20kHz)
               (Note: FB is SILK FB-MWV or CELT fullband)
 codebook/stereo + frame_count:
   0x = 1 frame in packet
   10 = 2 frames in packet
   11 = multiple frames (frames coded in the payload)
```

**Config code to mode mapping (first 3 bits of config_code):**

| Config Code | Mode | SILK Bandwidth | CELT Bandwidth |
|-------------|------|---------------|----------------|
| 0 | SILK-only | NB | N/A |
| 1 | SILK-only | MB | N/A |
| 2 | SILK-only | WB | N/A |
| 3 | SILK-only | SWB | N/A |
| 4 | SILK-only | FB-MWV | N/A |
| 5 | Hybrid | SWB | FB |
| 6 | Hybrid | WB | FB |
| 7 | Hybrid | MB | FB |
| 8 | Hybrid | NB | FB |
| 9 | CELT-only | N/A | FB |
| 10 | CELT-only | N/A | FB (low complexity) |
| 11 | CELT-only | N/A | FB (low complexity variant) |
| 12 | CELT-only | N/A | FB (low complexity variant) |
| 13 | CELT-only | N/A | FB (low complexity variant) |
| 14 | CELT-only | N/A | FB (low complexity variant) |
| 15 | CELT-only | N/A | FB (low complexity variant) |
| 16-19 | CELT-only | N/A | SWB |
| 20-23 | CELT-only | N/A | SWB (low complexity) |
| 24-27 | CELT-only | N/A | WB |
| 28-31 | CELT-only | N/A | WB (low complexity) |

The stereo flag (bit 5): 0 = mono, 1 = stereo. For SILK-only and hybrid modes, this field is present in the SILK header. For CELT-only, it is explicitly coded in the TOC byte.

#### 3.3.2 Frame Count and Size Encoding

When the frame count field in the TOC byte is `10` (2 frames):
- Both frames are the **same size** (derived from `v` in the packet)
- The size is encoded after the optional `c` and `m` fields (see Section 3.3.4)

When the frame count field is `11` (multiple frames):
- The first byte after the TOC byte is a **padding count** (`p`), indicating how many bytes of padding are present
- The frame count is computed as: `count = (byte - p) / 2`
- Each frame size is stored as a 2-byte big-endian value in the frame size table

**Frame size table (for `11` case):**
```
 Byte 0:     frame_size_table_byte (num_frames + padding encoded)
 Bytes 1-2:  size of frame 1 (u16 BE)
 Bytes 3-4:  size of frame 2 (u16 BE)
 ...
 Bytes N-2:  padding (p bytes)
```

#### 3.3.3 Codebooks

The remaining 2 bits of the TOC byte (`codebook / frame count` field, bits 0-1) encode the **codebook** for the current mode:

For **SILK-only mode**: these bits carry additional SILK parameters.

For **CELT-only mode**: these 2 bits encode the **codebook index** (0-3), which determines the pulse entropy coding table used in the MDCT coefficient quantization. The codebook index is combined with the frame size to derive the total bits available for the spectral representation.

#### 3.3.4 Optional Fields: `c` (CBR) and `m` (VBR/MRRP)

After the TOC byte, additional control bytes may be present:

**`c` (CBR flag):** Present in SILK-only and hybrid modes. 1 bit.
- `c = 0`: Variable bitrate (VBR)
- `c = 1`: Constant bitrate (CBR)

**`m` (VBR/metadata flag):** 1 bit, present when `c = 0`.
- `m = 0`: VBR, metadata is present
- `m = 1`: VBR, metadata is not present (decoder can skip analysis)

**`silk_vad`: (in SILK-only VBR)** 1 bit, present in SILK-only VBR mode:
- `silk_vad = 0`: SILK VAD is active (voice activity detection)
- `silk_vad = 1`: SILK VAD is disabled (all frames are voice)

**`silk_dyn`: (in SILK-only)** 1 bit:
- `silk_dyn = 0`: SILK uses fixed rate allocation
- `silk_dyn = 1`: SILK uses dynamic rate allocation

These optional bytes are followed by the **payload size** (for CBR or two-frame modes), and then the **actual compressed audio data**.

### 3.4 Byte-Level Packet Layout Examples

#### Example 1: Mono, SILK-only, VBR, 1 frame, 20 ms WB

```
 Byte 0:     0x42 (config=8, frame_count=0) → SILK-only, WB, 1 frame
 Byte 1:     0x00 (c=0, m=0)                → VBR, metadata present
 Bytes 2-3:  0xNN 0xNN (payload size u16)  → frame payload size
 Bytes 4+:   SILK compressed data           → N bytes
```

#### Example 2: Stereo, CELT-only, 1 frame, FB

```
 Byte 0:     0xC9 (config=25, stereo=1)     → CELT, FB, 1 frame, stereo
 Bytes 1+:   CELT compressed data          → N bytes
```

#### Example 3: Stereo, hybrid, 2 frames, same size

```
 Byte 0:     0x88 (config=17, stereo=1, frame_count=1)
 Bytes 1-2:  0xMM 0xMM (size v for both frames)
 Bytes 3+:   frame 1 data (v bytes) + frame 2 data (v bytes)
```

---

## 4. SILK — SPEECH CODING LAYER

### 4.1 SILK Overview

SILK (originally "Skype Low-Light Kernel") is a speech codec based on linear prediction. It was designed for voice communication and handles the lower frequency portion of the audio signal in hybrid mode, or the full signal in SILK-only mode.

SILK operates in four bandwidth modes:

| Mode | Bandwidth | Frequency Range | SILK Sample Rate | Frame Size (samples) |
|------|-----------|-----------------|-------------------|---------------------|
| NB | Narrowband | 0–4 kHz | 8 kHz | 120 (2.5ms), 240 (5ms), 480 (10ms) |
| MB | Medium Band | 0–6 kHz | 12 kHz | 360 (3ms), 720 (6ms) |
| WB | Wideband | 0–8 kHz | 16 kHz | 480 (3ms), 960 (6ms) |
| FB-MWV | Fullband Modified WV | 0–12 kHz | 24 kHz | 720 (3ms), 1440 (6ms) |

**FB-MWV:** "Modified Wideband with extended frequency response." This is SILK's fullband mode, which covers approximately 12 kHz audio bandwidth. It is not true fullband (20 kHz) — that range is handled by CELT in hybrid mode or by CELT alone in CELT-only mode.

### 4.2 Linear Prediction Architecture

SILK uses a **Linear Predictive Coding (LPC)** framework similar to classical CELP codecs but with significant enhancements:

1. **LPC Analysis Order:** 12th-order LPC for NB, 16th-order for WB/FB-MWV
2. **Levinson-Durbin Recursion:** Standard algorithm to solve the Toeplitz normal equations for LPC coefficients
3. **Perceptual Weighting Filter:** Applies spectral shaping to the quantization error to exploit masking
4. **Long-Term Prediction (LTP):** Handles pitch periodicity (fundamental frequency ~50 Hz to 500 Hz)
5. **Noise Shaping (NS):** An additional shaping filter applied to the excitation

### 4.3 SILK Encoding Pipeline

```
Input: Audio at SILK internal sample rate (8/12/16/24 kHz depending on bandwidth)
        |
        v
+--------------------+
| High-pass filter   |  Removes DC and low-frequency rumble (8 Hz cutoff)
+--------------------+
        |
        v
+--------------------+
| LPC Analysis       |  Compute a_k coefficients (12 for NB, 16 for WB/FB)
| (Levinson-Durbin)   |  Use autocorrelation + Hamming window
+--------------------+
        |
        v
+--------------------+
| LTP Analysis       |  Find pitch lag (4-16 ms) and LTP gain (0-1)
|                    |  5th-order LTP filter
+--------------------+
        |
        v
+--------------------+
| Noise Shaping      |  Shape quantization noise in frequency domain
| Analysis           |  Split into 4 frequency bands, analyze energy
+--------------------+
        |
        v
+--------------------+
|残差计算: r[n] = x[n] - sum(a_k * x[n-k]) - sum(b_m * r[n-LTP_m]) |
|                    |  Compute residual after LP and LTP
+--------------------+
        |
        v
+--------------------+
|残差量化:           |  Scalar quantization of residual
| Noise shaping      |  Gains per band, entropy-coded
| parameter encoding |  a_k, LTP params, noise shaping, gains all encoded
+--------------------+
```

### 4.4 SILK Bit Allocation

The SILK encoder distributes bits among several parameter classes:

| Parameter | Encoding Method | Bits (estimated, varies by mode) |
|-----------|----------------|--------------------------------|
| LPC coefficients | LAR (Log Area Ratios) | ~20-40 bits |
| LTP lag | Scalar + Huffman | ~7-9 bits |
| LTP gains | Vector quantization | ~3 bits per subframe |
| Noise shaping gains | Scalar quantization | ~3-5 bits per band |
| Residual | Scalar quantization | Remaining bits (VBR) |

### 4.5 SILK in Hybrid Mode

In hybrid mode, SILK encodes the low frequencies (typically up to 8 kHz) and CELT encodes the high frequencies (typically 8–20 kHz). The SILK encoder must account for the frequency range overlap:

- SILK produces a decoded signal up to its bandwidth limit
- A **band-split filter** separates input into low and high frequency bands
- SILK encodes the low band
- The SILK **decoder output** (low band) is subtracted from the original signal to produce a **high-band residual**
- CELT encodes the high-band residual
- The decoder combines SILT and CELT outputs

The crossfade between SILK and CELT bands is typically at 6–8 kHz to avoid audible artifacts at the band boundary.

---

## 5. CELT — TRANSFORM-BASED LAYER

### 5.1 CELT Overview

CELT (Constrained Energy Lapped Transform) is a MDCT-based audio coder optimized for music and general audio. It operates independently in CELT-only mode or as the high-band encoder in hybrid mode.

CELT operates at the native 48 kHz sample rate and covers:
- **Narrowband (NB):** 0–4 kHz (CELT config codes 24-27)
- **Wideband (WB):** 0–8 kHz (CELT config codes 20-23)
- **Super Wideband (SWB):** 0–12 kHz (CELT config codes 16-19)
- **Fullband (FB):** 0–20 kHz (CELT config codes 9-15)

### 5.2 MDCT Transform

The Modified Discrete Cosine Transform is the core transform in CELT. CELT uses a **low-overhead MDCT** with a **pre/post rotation** to enable 50% overlap with only N extra bits per frame.

#### 5.2.1 Standard MDCT

The forward MDCT for a block of 2N samples is:

```
X[k] = sum(n=0 to 2N-1) { x[n] * cos(pi/N * (k + 0.5) * (n - N + 0.5)) }
       for k = 0, 1, ..., N-1
```

The inverse (IMDCT):

```
y[n] = (1/N) * sum(k=0 to N-1) { X[k] * cos(pi/N * (k + 0.5) * (n - N + 0.5)) }
       for n = 0, 1, ..., 2N-1
```

With 50% overlap and a sinusoidal window, perfect reconstruction is achieved.

#### 5.2.2 CELT's Low-Overhead MDCT

Standard MDCT requires N spectral coefficients for 2N time-domain samples, meaning it **loses** N degrees of freedom. CELT compensates for this using a **pre-rotation/post-rotation** technique (also called "anti-symmetric/symmetric" decomposition) that encodes the extra N constraints in the bitstream at very low cost.

The transform works as follows:

1. **Pre-rotation:** Apply an orthonormal rotation to the 2N input samples before the DCT
2. **DCT-IV:** Apply an N-point DCT-IV (which is its own inverse) to produce N coefficients
3. **Post-rotation:** The remaining N "lost" degrees of freedom are encoded as the **energy** (norm) of the band in the first N/2 coefficients

Concretely, the low-overhead MDCT encoding process:
```
1. Take N+1 overlapping blocks of N samples each
2. Apply MDCT to get N coefficients per block
3. For the overlap region, constrain the coefficients to be consistent
4. Encode the band energies (N/2 values) separately as coarse quantizers
5. Encode the remaining fine details with entropy coding
```

#### 5.2.3 CELT Frame Size and MDCT Parameters

CELT always uses a fixed MDCT size of **512 samples** (10.67 ms at 48 kHz), regardless of frame size. For longer Opus frames (20 ms, 40 ms, 60 ms), the encoder codes multiple MDCT frames per packet.

| Opus frame size | CELT MDCT blocks | Total samples |
|----------------|-----------------|---------------|
| 10 ms | 1 block | 512 |
| 20 ms | 2 blocks | 1024 |
| 40 ms | 4 blocks | 2048 |
| 60 ms | 6 blocks | 3072 |

The encoder groups consecutive MDCT blocks and jointly optimizes the bit allocation across them.

### 5.3 Per-Band Energy Encoding

CELT organizes frequency content into **bands** following a pseudo-Bark scale (human ear frequency resolution):

| Band Index | Frequency Range (approx.) | MDCT bins |
|-----------|--------------------------|-----------|
| 0 | 0–250 Hz | 0-4 |
| 1 | 250–500 Hz | 4-8 |
| 2 | 500–1 kHz | 8-16 |
| 3 | 1–2 kHz | 16-32 |
| 4 | 2–4 kHz | 32-64 |
| 5 | 4–6.5 kHz | 64-108 |
| 6 | 6.5–10 kHz | 108-174 |
| 7 | 10–15 kHz | 174-272 |
| 8 | 15–20 kHz | 272-512 |
| 9 | 20+ kHz | 272-512 (folded) |

Each band has:
1. **Coarse energy:** First 2-3 bits per band for the band energy (dB value, quantized)
2. **Fine energy:** Residual quantization refinement
3. **Spectral details:** The MDCT coefficients within the band

### 5.4 Coefficient Quantization and Entropy Coding

The MDCT coefficients in each band are quantized using a **split vector quantizer (SVQ)**:

1. **Band normalization:** Coefficients are normalized by the band energy
2. **Pulse vector:** A sparse representation using unit pulses (1 or -1) placed at different positions
3. **Codebook index:** The 2 LSBs of the TOC byte select the **codebook** that determines how many pulses and which entropy coding table is used

**Codebook selection:**

| Codebook Index | Description | Used for |
|---------------|-------------|----------|
| 0 | Normal mode | General audio |
| 1 | Split mode | Fine-grained control at low bitrates |
| 2 | QR mode | Further variants |
| 3 | No pulse mode | Very low bitrate or silence |

The pulse positions are encoded with a **logarithmic pulse count** encoding — more pulses require more bits but provide better quality. The entropy coding uses a **range coder** (a form of arithmetic coding) for the coefficient data.

### 5.5 Stereo Coding in CELT

CELT supports both **intensity stereo** and **joint stereo** coding:

**Intensive stereo (also called "mid-side simplification"):** In CELT, the stereo field is progressively reduced at low bitrates. The encoder can choose to code only the **mid channel** (L+R)/2 for certain high-frequency bands, discarding the side information (L-R). This is signaled per-band in the bitstream.

**Fine-grained stereo:** At higher bitrates, the full L/R spectrum is coded independently with some band-level coupling.

**Stereo decision threshold:** The encoder decides per-frame whether to use independent stereo or mid-only coding for each band based on bitrate and the correlation between channels.

---

## 6. HYBRID MODE — SILK + CELT INTEGRATION

### 6.1 Hybrid Mode Operation

Hybrid mode is engaged when the encoder chooses a configuration code from the range 5-8 (see Section 3.3.1). In this mode:

- **SILK** encodes the low-frequency portion of the signal
- **CELT** encodes the high-frequency portion
- A **band-split filter** separates the input into low and high bands before encoding
- A **band-merge/synthesis** process combines the two decoded streams

### 6.2 Band Split

The input signal (at 48 kHz) is split at a crossover frequency:

| Hybrid Config | Crossover Frequency |
|--------------|---------------------|
| Config 5 (SWB) | SILK covers up to ~12 kHz, minimal CELT high-band |
| Config 6 (WB) | SILK covers up to ~8 kHz, CELT covers ~8-20 kHz |
| Config 7 (MB) | SILK covers up to ~6 kHz, CELT covers ~6-20 kHz |
| Config 8 (NB) | SILK covers up to ~4 kHz, CELT covers ~4-20 kHz |

The band split uses a **low-pass filter** (for SILK's input) and a **high-pass filter** (for CELT's input), both designed to have minimal phase distortion.

### 6.3 SILK/CELT Transition

In hybrid mode, SILK produces its output at its native sample rate (8/12/16/24 kHz). This must be upsampled to 48 kHz before combining with the CELT output. The upsampling uses a **linear interpolation filter** followed by a **CIC (Cascaded Integrator-Comb)** filter for efficiency.

The band-gap around the crossover frequency is carefully managed to avoid comb-filtering artifacts. A small overlap region allows for smooth crossfading between SILK and CELT outputs.

### 6.4 Hybrid Bitrate Allocation

The encoder distributes the total bit budget between SILK and CELT based on:
- The bitrate point (lower bitrates give more to SILK for speech quality)
- Signal classification (more SILK for speech, more CELT for music)
- Perceptual importance (low frequencies are perceptually more important)

Typical allocation at 32 kbps hybrid:
- SILK: ~20-24 kbps
- CELT: ~8-12 kbps

Typical allocation at 64 kbps hybrid:
- SILK: ~32 kbps
- CELT: ~32 kbps

---

## 7. VBR (VARIABLE BITRATE) AND CBR (CONSTANT BITRATE)

### 7.1 VBR Mode

Opus supports true variable bitrate encoding, where the bitrate adapts to the complexity of the input signal. This is critical for:

- **Voice Activity Detection (VAD):** Silence and low-complexity passages get fewer bits
- **Dynamic bit allocation:** More bits allocated to complex transients, fewer to steady-state
- **Perceptual optimization:** Bits are distributed to the perceptually most important components

VBR is the **default mode** in libopus. The encoder uses a **rate-distortion optimization (RDO)** loop to find the best quantization step for each parameter at a given target quality/bitrate.

**VBR parameter:** `OPUS_SET_VBR_REQUEST` (config) or `-b:v` in FFmpeg with `bitrate=XXk`.

### 7.2 CBR Mode

In CBR mode, every encoded frame uses exactly the allocated bit budget (or as close as possible within the frame's complexity constraints). CBR is useful for:
- **Streaming with fixed bandwidth:** Consistent bitrate for stable network delivery
- **Multiplexing with other streams:** Fixed-size frames simplify container interleaving
- **Recording to fixed-size blocks:** Consistent disk/memory usage

**CBR parameter:** `OPUS_SET_VBR_REQUEST(0)` in libopus API.

### 7.3 VBR vs CBR in the Bitstream

The `c` bit in the Opus packet TOC byte signals CBR vs VBR:
- `c = 1`: CBR — the decoder knows each frame uses the same number of bits
- `c = 0`: VBR — frame sizes vary

For VBR encoding, the encoder communicates frame sizes to the decoder in several ways:
1. **Explicit size table:** In multi-frame packets, each frame's size is listed
2. **Single size field:** In two-frame mode, a single `v` field gives both frame sizes
3. **No size needed:** In single-frame mode, the decoder can infer the size from packet length

### 7.4 Bitrate Ranges

| Mode | Min Bitrate | Default | Max Bitrate |
|------|------------|---------|-------------|
| CELT-only (mono) | 6 kbps | 64 kbps | 256 kbps |
| CELT-only (stereo) | 8 kbps | 96 kbps | 512 kbps |
| SILK-only (mono) | 6 kbps | 20 kbps | 40 kbps |
| SILK-only (stereo) | 8 kbps | 24 kbps | 50 kbps |
| Hybrid (mono) | 12 kbps | 40 kbps | 192 kbps |
| Hybrid (stereo) | 16 kbps | 64 kbps | 256 kbps |

### 7.5 Bitrate Control in libopus API

```c
// Set VBR with target bitrate
opus_encoder_ctl(enc, OPUS_SET_BITRATE(bitrate));      // bitrate in bits/sec

// Enable/disable VBR
opus_encoder_ctl(enc, OPUS_SET_VBR(1));                // 1 = VBR, 0 = CBR

// Set VBR constraint (hard cap vs quality target)
opus_encoder_ctl(enc, OPUS_SET_VBR_CONSTRAINT(0));     // 0 = unconstrained VBR

// Set application mode (affects default rate-distortion tradeoffs)
opus_encoder_ctl(enc, OPUS_SET_APPLICATION(OPUS_APPLICATION_AUDIO));      // Music
opus_encoder_ctl(enc, OPUS_SET_APPLICATION(OPUS_APPLICATION_VOIP));      // Voice
opus_encoder_ctl(enc, OPUS_SET_APPLICATION(OPUS_APPLICATION_RESTRICTED_LOWDELAY)); // Low latency
```

---

## 8. FORWARD ERROR CORRECTION (FEC)

### 8.1 Opus FEC Overview

RFC 7875 defines the Opus FEC mechanism. The core idea is: when encoding frame N, the encoder also computes a **lower-quality representation of frame N+1** and transmits it as redundant data appended to frame N's packet.

If frame N+1 is lost at the network layer, the decoder can use the FEC data from frame N to reconstruct a **approximation** of N+1.

### 8.2 FEC Encoding Process

```
For each frame N:
  1. Encode frame N normally → payload P_N
  2. If N+1 has significant energy (not silence):
     a. Encode frame N+1 at LOWER bitrate (typically 1/2 or 1/4 of normal)
     b. Append FEC payload F_(N+1) after P_N
  3. Packet structure: [P_N] [F_(N+1)]
```

### 8.3 FEC in the Bitstream

The FEC payload is signaled in the Opus packet:

- In **SILK-only** mode: FEC is embedded as additional SILK data after the main SILK payload
- In **CELT-only** mode: CELT has an explicit FEC flag that tells the decoder to expect FEC data
- The FEC bitrate is controlled separately: `OPUS_SET_INBAND_FEC(1)` to enable

**Packet with FEC:**

```
 [TOC byte] [main payload for frame N] [FEC payload for frame N+1]
```

The decoder knows the FEC payload size because it decodes the main payload and the remaining bytes are the FEC data.

### 8.4 FEC Decoding

When the decoder detects a missing frame:
1. Check if the previous packet contained FEC data for this frame
2. If yes, decode the FEC payload using the **lower-complexity** decoder path
3. Use the FEC-reconstructed frame as a substitute
4. The FEC quality will be lower than the original, but better than PLC (Packet Loss Concealment)

### 8.5 FEC Performance Considerations

| Metric | Value |
|--------|-------|
| FEC overhead | ~25-50% additional bandwidth |
| FEC quality | ~3-6 dB worse than normal encoding |
| Best use case | Moderate packet loss (< 20%), voice |
| Worst use case | High packet loss, music (FEC wastes bits on transients) |

FFmpeg enables FEC with: `-opus_fec 1`

---

## 9. PACKET LOSS CONCEALMENT (PLC)

### 9.1 PLC Overview

Opus has built-in Packet Loss Concealment (PLC) that generates replacement audio when packets are lost. Unlike FEC, PLC does not require any redundant data — it synthesizes audio from previously received frames.

### 9.2 SILK PLC

SILK uses a **pitch waveform replication** approach:
1. Find the pitch period (fundamental frequency) in the last good frame
2. Replicate the pitch waveform and overlap it with the previous frame's tail
3. Apply a fade-out envelope to decay the signal
4. The result is a plausible-sounding continuation of the speech signal

```
Lost frame PLC in SILK:
  pitch_lag = detect_pitch(decoder_state)
  for n in 0..frame_size-1:
    plc_output[n] = decoder_state[(n - pitch_lag) mod pitch_lag] * fade[n]
    fade[n] = exp(-3.0 * n / frame_size)   # exponential decay
```

### 9.3 CELT PLC

CELT PLC uses a **band-extrapolation** technique:
1. Track the per-band energy of the last few decoded frames
2. Extrapolate the energy trajectory forward
3. Generate noise in each band, shaped to match the extrapolated energy
4. The noise has the correct spectral envelope, providing a natural-sounding "filler"

```
CELT PLC algorithm:
  for each band b:
    energy_extrapolated = extrapolate(energy_history[b])
    noise_band = generate_white_noise(frame_size)
    for n in 0..frame_size-1:
      noise_band[n] *= energy_extrapolated[n]
    plc_output += noise_band
```

### 9.4 Hybrid Mode PLC

In hybrid mode, SILK PLC handles the low-frequency content and CELT PLC handles the high-frequency content independently. The results are combined in the output.

### 9.5 PLC Quality and Limitations

- **PLC quality degrades rapidly** with consecutive lost frames (no reference for extrapolation)
- **Silence** frames are PLC'd as silence (no generation needed)
- **Music** has more diverse spectral characteristics, making PLC less convincing than for speech
- libopus's PLC was significantly improved in version 1.3+ with better energy tracking

FFmpeg PLC behavior: PLC is always active in the decoder (no explicit flag needed).

---

## 10. OGG CONTAINER — OGG OPUS

### 10.1 RFC 7845: Ogg Opus Mapping

RFC 7845 defines how Opus audio is packaged in Ogg containers. The structure is straightforward:

```
+---------------------------+
|      Ogg Page 1          |  OpusHead page (bos = 1)
|   [OpusHead magic + data] |
+---------------------------+
|      Ogg Page 2           |  OpusTags page
|  [OpusTags magic + data]  |
+---------------------------+
|      Ogg Page 3           |  Audio page 1 (contains 1+ Opus packets)
|  [Opus packet 1]          |
+---------------------------+
|      Ogg Page 4           |  Audio page 2
|  [Opus packet 2]          |
+---------------------------+
|      ...                  |
|      Ogg Page N           |  Audio page (eos = 1)
+---------------------------+
```

### 10.2 Ogg Page Header

Each Ogg page has a 27-byte header:

| Offset | Field | Size | Description |
|--------|-------|------|-------------|
| 0 | capture_pattern | 4 | "OggS" (0x4F 0x67 0x67 0x53) |
| 4 | stream_structure_version | 1 | Must be 0 |
| 5 | header_type_flag | 1 | bit 0 = BOS, bit 1 = EOS |
| 6 | absolute_position | 8 | Granule position (sample count) |
| 14 | stream_serial_number | 4 | Unique stream ID |
| 18 | page_sequence_no | 4 | Page counter within stream |
| 22 | page_checksum | 4 | CRC32 of page |
| 26 | page_segments | 1 | Number of segment entries |
| 27+ | segment_table | variable | Lacing values (1-255 each) |

### 10.3 Granule Position

The granule position in Ogg Opus pages encodes the **decoded sample count** at the **end** of the page:

```
granule = sum_of_all_decoded_samples_up_to_and_including_this_page
```

This is critical for seeking: to find the page containing sample N, binary search on granule positions.

### 10.4 OpusHead Page

The OpusHead page has the **BOS (Beginning of Stream)** flag set (header_type_flag bit 0 = 1). It contains exactly the 19-byte OpusHead structure described in Section 3.1. It may span multiple Ogg pages (unlikely for OpusHead, but possible in theory).

### 10.5 OpusTags Page

The OpusTags page follows the OpusHead page. It has the standard OpusTags structure (Section 3.2). Both OpusHead and OpusTags pages should be **unconditional** — they are always present and decodable without waiting for audio pages.

### 10.6 Audio Pages

Audio pages contain Opus packets. Key rules:
- Each Ogg page can contain **one or more** Opus packets
- A single Opus packet can span **one or more** Ogg pages (rare for Opus but valid)
- The segment table encodes the lacing: each entry is 1-255, where the sum of all entries in a page equals the page's payload size
- A value of 255 means the packet **continues** in the next segment; any value < 255 terminates the current packet

### 10.7 Seeking in Ogg Opus

Seeking in Ogg Opus requires:
1. A **seek table** (Ogg Skeleton or Opus-specific seek) or binary search
2. The **pre-skip** value from OpusHead to calculate the PCM sample offset
3. The decoder must be **primed** with enough context to decode from the seek point

**Seeking formula:**

```
target_sample = desired_PCM_sample
page_granule = target_sample + opus_head.pre_skip
# Find page with granule >= page_granule
# Decode from that page, discard first pre_skip samples of decoded output
```

FFmpeg automatically handles seeking in Ogg Opus files.

---

## 11. WEBM / MATROSKA CONTAINER — OPUS IN MKV

### 11.1 Matroska / WebM Opus Mapping

Opus in Matroska (and by extension WebM) follows the Matroska specification. The mapping is defined in the Matroska spec and is largely compatible with the signaling used in Ogg Opus, but with some important differences.

### 11.2 CodecPrivate Block

In Matroska/WebM, the Opus codec initializes using a `CodecPrivate` element. For Opus, this is structured as:

```
 CodecPrivate = OpusHead (19 bytes) + optional ChannelMapping (if family != 0)

 [OpusHead bytes 0-18] + [ChannelMappingTable bytes]
```

The `CodecPrivate` contains the full OpusHead plus the channel mapping table, which provides the Vorbis channel ordering and channel-to-speaker mapping for surround sound configurations.

### 11.3 Block Layout

Matroska stores Opus packets in `Block` elements with the following structure:

```
 Block:
   TrackNumber (EBML var-length)
   Timecode (s16, relative to Cluster timecode, in milliseconds / sample count)
   Flags (u8):
     bit 0: Keyframe flag (1 = keyframe, 0 = non-keyframe)
     bit 1: Invisible (1 = discard after rendering)
     bit 2: Lacing (0 = none, 1 = Xiph, 2 = EBML, 3 = fixed-size)
   [Block data] — Opus packet payload
```

**Key difference from Ogg:** In Matroska/WebM, every Opus `Block` is a **self-contained Opus packet**. The container does not need to encode packet boundaries (as Ogg's segment table does) because each Block is independently one packet.

### 11.4 Block Duration

Opus frames are **constant in duration** (determined by the config in CodecPrivate). Therefore Matroska Opus Blocks do not need an explicit duration field — the decoder knows the frame size from the config. The `BlockDuration` element on the `Cluster` level can be used for gaps, but for normal Opus, it's computed from frame count × frame size.

### 11.5 Channel Mapping in WebM

When `channel_mapping_family = 1`, the CodecPrivate includes a channel mapping table following the OpusHead:

```
 Bytes 0-18:     OpusHead (19 bytes)
 Byte 19:        channel_mapping = number of entries in mapping
 Bytes 20+:      stream_map[channel_mapping] — which output channel
                 reserved = 0 (must be 0)
```

The channel ordering follows Vorbis channel ordering:

| Channels | Vorbis Order | Speaker Position |
|---------|-------------|------------------|
| 1 | 0 | Mono (C) |
| 2 | 0, 1 | Front Left, Front Right |
| 5.1 | 0, 1, 2, 3, 4, 5 | FL, FR, FC, LFE, BL, BR |
| 7.1 | 0-7 | FL, FR, FC, LFE, BL, BR, SL, SR |
| 7.1 (WMP) | 0-7 | FL, FR, FC, LFE, BL, BR, FLC, FRC |

### 11.6 Seek Head and Cues

Matroska/WebM files use `Cues` elements for seeking. The Cues reference the `Cluster` and `Block` positions. For Opus, the Cues should reference keyframe Blocks. Opus CELT frames at the beginning of a packet sequence can be treated as keyframes.

---

## 12. CHANNEL MAPPING AND SURROUND SOUND

### 12.1 Channel Mapping Families

Opus supports three channel mapping families:

| Family | Value | Description |
|--------|-------|-------------|
| 0 | 0 | No channel mapping. Only mono (1ch) or stereo (2ch). Simple layout. |
| 1 | 1 | Vorbis channel ordering. Channel mapping table present. Supports 1-8 channels. |
| 2+ | 2+ | Reserved for future use |

### 12.2 Channel Mapping Family 0 (No Mapping)

For mono and stereo only:
- `channels = 1`: Single channel of audio
- `channels = 2`: Stereo. Channel 0 = left, Channel 1 = right.

No additional channel mapping data is present in the OpusHead.

### 12.3 Channel Mapping Family 1 (Vorbis Mapping)

For multichannel (3-8 channels), the OpusHead includes a channel mapping table:

```
 Byte 19:   stream_count = number of coded streams
 Byte 20:   coupled_count = number of coupled streams (stereo pairs)
 Bytes 21+: channel_map[channels] = array mapping output channel to stream

 stream_map[i] = which stream this output channel comes from
 reserved[i] = must be 0
```

**How it works:**
- Opus encodes audio in "streams" — each stream is independently coded mono or stereo
- The `channel_map` array tells the decoder which output channel each stream feeds into
- Coupled streams are stereo; uncoupled streams are mono
- The mapping allows arbitrary routing (e.g., a 5.1 source can be downmixed to 2 streams, or encoded as 6 mono streams)

### 12.4 Vorbis Channel Ordering Reference

| # | Vorbis Index | Channels |
|---|-------------|----------|
| 0 | Mono / C |
| 1 | Left |
| 2 | Center |
| 3 | Right |
| 4 | Back Left |
| 5 | Back Right |
| 6 | Side Left (5.1 side) |
| 7 | Side Right (5.1 side) |
| 8 | LFE |

### 12.5 Downmixing at the Encoder

Opus encoders can choose to downmix surround input to fewer streams before encoding. This reduces bitrate but loses spatial information. For example:
- 5.1 → 2 (stereo downmix)
- 5.1 → 4 (quadrophonic approximation)

The encoder reports the downmix as the actual number of coded streams. The decoder must **not** upmix to the original channel count — the output is the downmixed version.

### 12.6 Surround Sound Recommendations

| Input Format | Recommended Encoding | Bitrate (stereo equivalent) |
|-------------|---------------------|----------------------------|
| 5.1 | Stereo downmix + Parametric Stereo (PS) | 96-128 kbps |
| 5.1 | 4 streams (L R C LFE BL/BR as 2 mono) | 128-192 kbps |
| 7.1 | 6 streams (5.1 + side channels) | 192-256 kbps |
| 7.1 | Stereo downmix | 96-128 kbps |

---

## 13. RTP PAYLOAD FORMAT FOR OPUS

### 13.1 RFC 7587: RTP Payload Format for Opus

RFC 7587 defines the RTP payload format for Opus. Key points:

- **Payload type:** Dynamically assigned (typically 96-127 in SDP)
- **Timestamp rate:** 48,000 Hz
- **Encoding name:** "opus" (in SDP `a=rtpmap`)
- **ptime:** Recommended 20 ms (1 frame per packet) or 40/60 ms (multiple frames)

### 13.2 RTP Header for Opus

```
  0                   1                   2                   3
  0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |V=2|P|X|  CC   |M|     PT      |       sequence number         |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |                           timestamp                           |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |           synchronization source (SSRC) identifier            |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
 |            contributing source identifiers (CSRC)            |
 ++-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+++++++++++++-+
 |                           Opus payload                        |
 ++-+-+-+-+-+-+-+-+                                         |
 |                           ....                                |
 +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### 13.3 SDP Parameters

The SDP `a=fmtp` line for Opus RTP includes several parameters:

```
 a=rtpmap:96 opus/48000/2
 a=fmtp:96; maxplaybackrate=48000; sprop-maxcapturerate=48000;
          sprop-stereo=0; stereo=0; useinbandfec=0; usedtx=0
```

| Parameter | Description | Values |
|-----------|-------------|--------|
| `maxplaybackrate` | Maximum output sample rate decoder will accept | 8000-48000 |
| `sprop-maxcapturerate` | Maximum capture rate of sender | 8000-48000 |
| `sprop-stereo` | Sender signals if input is stereo | 0 or 1 |
| `stereo` | Sender requests stereo encoding | 0 or 1 |
| `useinbandfec` | Sender will include in-band FEC | 0 or 1 |
| `usedtx` | Sender will use discontinuous transmission | 0 or 1 |
| `cbr` | Sender requests constant bitrate | 0 or 1 |
| `ptime` | Packet duration | 5, 10, 20, 40, 60 |
| `maxptime` | Maximum packet duration | 5, 10, 20, 40, 60 |

### 13.4 Multiple Frames in RTP

Opus RTP packets can carry multiple Opus frames:
- `M = 1` flag in RTP header marks the **last** packet of a talkspurt (silence gap)
- Multiple frames are concatenated directly (no delimiter needed — frame sizes are derived from the TOC byte)
- Maximum payload size: **120 ms** of audio per packet (RFC 7587 limits this to prevent implausibly large packets)

### 13.5 DTX (Discontinuous Transmission)

DTX reduces bitrate during silence by not transmitting packets at all. The decoder fills gaps using PLC. To signal DTX:
- Do not send RTP packets during silence
- The RTP timestamp still increments by the expected sample count during the silence gap
- When sending resumes, `M = 1` marks the talkspurt boundary

---

## 14. SAMPLE RATE — 48 KHZ REQUIREMENT

### 14.1 The 48 kHz Mandate

Opus always operates internally at **48,000 Hz**. This is a hard requirement of the specification (RFC 6716). The reasons for this design choice are:

1. **CELT compatibility:** The MDCT size (512 samples) at 48 kHz gives a 10.67 ms frame, which aligns with standard RTP timestamps and provides good latency/quality tradeoff.

2. **Rational arithmetic:** 48 kHz has many convenient divisors (1, 2, 3, 4, 6, 8, 12, 16, 24, 32, 48, 64, 96, ... ms), making it easy to map to common packet sizes and sample counts.

3. **Resampling:** 48 kHz can exactly resample from/to 8, 12, 16, 24, 32, and 44.1 kHz using polyphase filters with integer ratio approximations (the 44.1 kHz case uses 160/147 ratio).

4. **SILK integration:** SILK's internal sample rates (8, 12, 16, 24 kHz) are exact divisors of 48 kHz, simplifying the upsampling in hybrid mode.

### 14.2 Input Resampling

Any input audio at a different sample rate must be **resampled to 48 kHz** before being passed to the Opus encoder. The resampler must:

- **High quality:** Use at least a 64-tap polyphase filter for 44.1 kHz → 48 kHz conversion
- **Low group delay:** Especially for real-time communication, the resampler should not add excessive latency
- **Anti-aliasing:** Prevent aliasing when downsampling from higher rates

libopus includes a built-in resampler (based on `libspeexdsp` resampler) that handles arbitrary input rates.

### 14.3 Output Handling

The Opus decoder always outputs **48 kHz PCM**. If the application needs a different sample rate, it must resample after decoding:

- Use `libswresample` (FFmpeg)
- Use `libresample` / `soxr`
- Use a custom polyphase resampler

### 14.4 Resampling Ratios

| Input Rate | Output Rate | Ratio | Complexity |
|-----------|------------|-------|-----------|
| 44100 | 48000 | 160/147 | High (large polyphase filter) |
| 48000 | 48000 | 1/1 | None (pass-through) |
| 44100 | 44100 | 1/1 | None |
| 44100 | 24000 | 160/294 | High (two-stage: 44100→48000→24000) |
| 8000 | 48000 | 6/1 | Low (integer ratio) |
| 16000 | 48000 | 3/1 | Low (integer ratio) |
| 32000 | 48000 | 3/2 | Medium |

### 14.5 FFmpeg Opus Encoding and Sample Rate

FFmpeg handles the 48 kHz requirement automatically. When encoding to Opus:

```bash
# FFmpeg automatically resamples input to 48 kHz for Opus encoding
ffmpeg -i input.wav -c:a libopus -b:a 128k output.opus

# Force output to 48 kHz explicitly
ffmpeg -i input.wav -ar 48000 -c:a libopus output.opus

# Decode and resample to 44100
ffmpeg -i input.opus -ar 44100 output.wav
```

The OpusHead's `input_sample_rate` field should reflect the **original** input sample rate, not 48 kHz. This allows decoders to know if the source was, for example, 44.1 kHz CD audio that was upsampled for encoding.

---

## 15. ENCODER PARAMETERS AND PRESETS

### 15.1 libopus API Encoder Controls

```c
#include <opus.h>

OpusEncoder *opus_encoder_create(
    opus_int32 Fs,           // Sample rate (must be 48000)
    int channels,            // 1 or 2 (1 for mono, 2 for stereo)
    int application,         // OPUS_APPLICATION_VOIP, AUDIO, or RESTRICTED_LOWDELAY
    int *error                // Error code output
);

// Bitrate control
opus_encoder_ctl(enc, OPUS_SET_BITRATE(bitrate));    // bits/sec (e.g., 64000)
opus_encoder_ctl(enc, OPUS_SET_VBR(0));               // 0 = CBR, 1 = VBR
opus_encoder_ctl(enc, OPUS_SET_VBR_CONSTRAINT(0));   // unconstrained VBR

// Complexity (CPU usage vs quality)
opus_encoder_ctl(enc, OPUS_SET_COMPLEXITY(10));        // 0-10, default 10

// Signal type hint (helps encoder make better decisions)
opus_encoder_ctl(enc, OPUS_SET_SIGNAL(OPUS_SIGNAL_VOICE));    // Hint: speech
opus_encoder_ctl(enc, OPUS_SET_SIGNAL(OPUS_SIGNAL_MUSIC));    // Hint: music
opus_encoder_ctl(enc, OPUS_SET_SIGNAL(OPUS_AUTO));            // Let encoder decide

// FEC and DTX
opus_encoder_ctl(enc, OPUS_SET_INBAND_FEC(0));         // 0 = no FEC, 1 = enable FEC
opus_encoder_ctl(enc, OPUS_SET_DTX(0));                 // 0 = no DTX, 1 = enable DTX

// Packet loss percentage (helps encoder add redundancy)
opus_encoder_ctl(enc, OPUS_SET_PACKET_LOSS_PERC(0));    // 0-100

// Maximum forward jump (for PLC quality)
opus_encoder_ctl(enc, OPUS_SET_LSB_DEPTH(24));          // 16-24, default 24

// Frame size
opus_encoder_ctl(enc, OPUS_SET_EXPERT_FRAME_DURATION(OPUS_FRAMESIZE_20_MS));
// Options: 2.5, 5, 10, 20, 40, 60 ms, or AUTO
```

### 15.2 Application Modes

| Mode | Description | Typical Use Case | SILK vs CELT |
|------|-------------|-----------------|---------------|
| `OPUS_APPLICATION_VOIP` | Optimized for speech | VoIP calls, voice chat | Favors SILK at low bitrates |
| `OPUS_APPLICATION_AUDIO` | General audio | Music, movies, streaming | Balances SILK and CELT |
| `OPUS_APPLICATION_RESTRICTED_LOWDELAY` | Minimum latency | Real-time gaming, live | Forces CELT-only mode |

### 15.3 FFmpeg Opus Encoding Options

```bash
# Basic encoding
ffmpeg -i input.wav -c:a libopus output.opus

# VoIP mode with low bitrate
ffmpeg -i input.wav -c:a libopus -application voip -b:a 24k output.opus

# Music/audio mode with high quality
ffmpeg -i input.wav -c:a libopus -application audio -b:a 192k output.opus

# Low-delay mode (CELT only)
ffmpeg -i input.wav -c:a libopus -application lowdelay -b:a 64k output.opus

# VBR with quality level (CRF-like)
ffmpeg -i input.wav -c:a libopus -vbr on -b:a 0 output.opus  # VBR, no bitrate cap

# CBR encoding
ffmpeg -i input.wav -c:a libopus -b:a 64k -cbr 1 output.opus

# With FEC (for lossy networks)
ffmpeg -i input.wav -c:a libopus -b:a 32k -application voip -opus_fec 1 output.opus

# With DTX (for silence suppression)
ffmpeg -i input.wav -c:a libopus -b:a 24k -opus_dtx 1 output.opus

# Multi-pass encoding (Opus supports two-pass via libopusenc)
```

### 15.4 Complexity Settings

| Complexity | Description | Approximate CPU Usage | Quality |
|-----------|-------------|----------------------|---------|
| 0 | Very fast, minimal quality | ~5% of real-time | Low |
| 1-3 | Fast | ~10-20% of real-time | Medium |
| 4-6 | Balanced | ~30-50% of real-time | Good |
| 7-9 | Slow | ~60-90% of real-time | Very Good |
| 10 | Slowest, best quality | ~100%+ of real-time | Best |

### 15.5 Quality vs Bitrate Reference

| Bitrate | Mode | Quality Level | Use Case |
|---------|------|--------------|----------|
| 6-8 kbps | CELT mono | Amateur radio quality | Extremely constrained |
| 12-16 kbps | SILK mono | Toll quality (PSTN-like) | VoIP with poor bandwidth |
| 24 kbps | SILK mono | Good voice quality | Standard VoIP |
| 32-48 kbps | Hybrid mono | Very good voice | High-quality VoIP |
| 64 kbps | Hybrid stereo | Good voice, acceptable music | Video conferencing |
| 96-128 kbps | CELT stereo | Good music quality | Streaming audio |
| 160-192 kbps | CELT stereo | Near-transparent music | High-quality streaming |
| 256-320 kbps | CELT stereo | Transparent (CD-like) | Archival replacement |

---

## 16. DECODER API AND FUNCTIONAL REFERENCE

### 16.1 Decoder Creation and State

```c
#include <opus.h>

OpusDecoder *opus_decoder_create(
    opus_int32 Fs,        // Sample rate (must be 48000)
    int channels,        // 1 or 2
    int *error            // Error code output
);

// Free decoder state
opus_decoder_destroy(decoder);
```

### 16.2 Basic Decoding

```c
// Decode a packet
opus_int32 opus_decode(
    OpusDecoder *dec,
    const unsigned char *data,     // Opus packet bytes
    opus_int32 len,                // Packet size in bytes (-1 = PLC mode)
    opus_int16 *pcm,               // Output buffer (interleaved 16-bit PCM)
    int frame_size,                // Number of samples per channel
    int decode_fec                 // 0 = normal, 1 = decode FEC for next frame
);

// frame_size must match the Opus frame size at 48 kHz:
//   2.5 ms  → 120 samples
//   5 ms    → 240 samples
//   10 ms   → 480 samples
//   20 ms   → 960 samples
//   40 ms   → 1920 samples
//   60 ms   → 2880 samples
```

### 16.3 PLC Decoding

To perform Packet Loss Concealment, pass `len = -1` and `decode_fec = 0`:

```c
// PLC: generate replacement for a lost packet
opus_int32 plc_size = opus_decode(
    decoder,
    NULL,               // No data (packet is lost)
    -1,                 // len = -1 signals PLC
    pcm_buffer,
    frame_size,
    0                   // decode_fec = 0 for PLC
);
```

### 16.4 FEC Decoding

When a packet is received after a loss, the decoder can use FEC to partially recover the lost frame:

```c
// On receiving frame N (after detecting N-1 was lost):
opus_int32 fec_size = opus_decode(
    decoder,
    data_N,             // Data for frame N (contains FEC for N-1)
    len_N,
    pcm_buffer,
    frame_size,
    1                   // decode_fec = 1: decode FEC for lost frame N-1
);
```

When `decode_fec = 1`, the decoder:
1. Decodes frame N normally
2. Simultaneously extracts the FEC data for frame N-1 from the current packet
3. Outputs the PLC-generated frame N-1 (or FEC-reconstructed frame N-1)

### 16.5 OpusPacketInfo API

```c
#include <opus.h>

// Get information about a packet without fully decoding it
int opus_packet_get_bandwidth(const unsigned char *data, opus_int32 len);
int opus_packet_get_nb_channels(const unsigned char *data, opus_int32 len);
int opus_packet_get_nb_frames(opus_int32 len, const unsigned char *data, opus_int32 len);
opus_int32 opus_packet_get_samples_per_frame(const unsigned char *data, opus_int32 Fs);
```

### 16.6 Decoder State Reset

```c
// Reset the decoder state (e.g., at a seek point)
opus_decoder_ctl(dec, OPUS_RESET_STATE);

// Get the decoder's final state after decoding a packet
opus_int32 final_range;
opus_decoder_ctl(dec, OPUS_GET_FINAL_RANGE(&final_range));
```

### 16.7 Multichannel Decoding

For multichannel Opus (> 2 channels), use the extended API:

```c
// Create decoder with channel count (must match OpusHead)
OpusDecoder *dec = opus_decoder_create(48000, num_channels, &error);

// Decode to pre-allocated buffer
opus_int32 samples = opus_multistream_decode(
    dec,
    data, len,
    pcm,             // num_channels * frame_size samples
    frame_size,
    decode_fec
);

// Destroy
opus_multistream_decoder_destroy(dec);
```

---

## 17. IMPLEMENTATION CHECKLIST

### 17.1 Encoder Implementation

- [ ] **Sample rate enforcement:** Confirm input is 48 kHz. If not, resample before encoding.
- [ ] **Bit depth:** Convert to 16-bit signed PCM if needed.
- [ ] **Channel count:** Verify channel count is supported (1-8 channels with appropriate mapping).
- [ ] **Frame size:** Select frame size based on latency requirements (2.5-60 ms).
- [ ] **Mode selection:** Choose SILK-only, hybrid, or CELT-only based on content type and bitrate.
- [ ] **Bitrate control:** Set VBR/CBR mode and target bitrate.
- [ ] **FEC:** Enable if the transport is lossy (enable in encoder AND decoder).
- [ ] **DTX:** Enable if silence suppression is desired.
- [ ] **Complexity:** Set based on available CPU budget.
- [ ] **OpusHead construction:** Fill all required fields, especially `pre-skip` and `channel_mapping_family`.
- [ ] **Channel mapping table:** For surround sound, construct the correct Vorbis channel mapping.
- [ ] **OpusTags:** Encode all metadata in Vorbis comment format.
- [ ] **Container encapsulation:** Wrap in Ogg or Matroska with proper page sequencing.
- [ ] **Pre-skip measurement:** Measure actual encoder lookahead to set correct pre-skip.

### 17.2 Decoder Implementation

- [ ] **Container parsing:** Extract OpusHead and OpusTags from Ogg/Matroska.
- [ ] **Validate OpusHead:** Check version, channel count, pre-skip, sample rate.
- [ ] **TOC parsing:** Parse first byte to determine mode, bandwidth, frame count, stereo.
- [ ] **Frame size extraction:** Derive frame size from TOC byte and packet length.
- [ ] **SILK decoding:** Implement SILK decoder (or use libopus) for SILK and hybrid modes.
- [ ] **CELT decoding:** Implement CELT decoder (or use libopus) for CELT and hybrid modes.
- [ ] **Hybrid merging:** Properly combine SILK and CELT outputs in hybrid mode with band-crossfade.
- [ ] **PLC:** Implement PLC using pitch waveform replication (SILK) and band-extrapolation (CELT).
- [ ] **FEC handling:** Extract FEC data from packets and use it when packets are lost.
- [ ] **Output resampling:** If needed, resample from 48 kHz to the target output rate.
- [ ] **Channel routing:** Apply the channel mapping table to route decoded streams to output channels.
- [ ] **Gain application:** Apply the `output_gain` from OpusHead to the decoded PCM.

### 17.3 Container Implementation (Ogg)

- [ ] **OggS capture pattern:** Verify "OggS" at start of every page.
- [ ] **Page checksum:** Validate CRC-32 of each page.
- [ ] **BOS page:** First page must contain OpusHead, with BOS flag set.
- [ ] **OpusTags page:** Second page must contain OpusTags.
- [ ] **Granule position:** Increment granule position by decoded samples per page.
- [ ] **EOS page:** Mark last page with EOS flag.
- [ ] **Segment table:** Properly decode continuation packets (255 values).
- [ ] **Seeking:** Use binary search on granule positions, add pre-skip offset.

### 17.4 Container Implementation (Matroska/WebM)

- [ ] **CodecPrivate:** Extract OpusHead from CodecPrivate element.
- [ ] **Channel mapping:** Parse extended channel mapping if family != 0.
- [ ] **Block parsing:** Parse Block elements, extract track number and timecode.
- [ ] **Keyframe flag:** Mark CELT frame starts as keyframes for seeking.
- [ ] **Cues:** Build cue entries referencing keyframe block positions.
- [ ] **Duration:** Compute from frame size × frame count (Opus frames are constant duration).

### 17.5 Metadata Preservation

- [ ] **Vorbis comments:** Preserve all user comment fields during transcoding.
- [ ] **Encoder identification:** Write `ENCODER` field with encoder name and version.
- [ ] **Gain normalization:** If re-encoding, check ReplayGain tags and apply gain to avoid loudness changes.
- [ ] **Cover art:** Opus in Ogg uses the standard Ogg封面 picture metadata block (type 6).
- [ ] **No tag stripping:** Do not strip vendor string, ReplayGain, or other extended fields.

### 17.6 Validation and Testing

- [ ] **Reference decoder:** Test against libopus reference implementation output.
- [ ] **Bit-exactness:** For lossless metadata pass-through, verify no changes to audio content.
- [ ] **Seeking test:** Seek to multiple positions and verify sample-accurate output.
- [ ] **PLC test:** Simulate packet loss and verify PLC audio quality.
- [ ] **FEC test:** Simulate packet loss with FEC enabled and verify recovery quality.
- [ ] **Multichannel test:** Verify correct channel routing for 5.1 and 7.1 sources.
- [ ] **Cross-container test:** Re-mux between Ogg Opus and WebM and verify bit-exact audio.
- [ ] **kid3 verification:** Use kid3-cli to inspect OpusHead, OpusTags, and any container-specific tags.
- [ ] **FFmpeg round-trip:** Encode with libopus, decode with FFmpeg, verify sample rate and channel count.

### 17.7 FFmpeg-Specific Commands Reference

```bash
# Encode WAV to Opus in Ogg container
ffmpeg -i input.wav -c:a libopus -b:a 128k output.opus

# Encode to WebM container
ffmpeg -i input.wav -c:a libopus -b:a 128k output.webm

# Extract Opus bitstream from container
ffmpeg -i input.opus -c:a copy -f opus output.bitstream

# Decode Opus to WAV
ffmpeg -i input.opus -c:a libopus output.wav

# Decode to different sample rate
ffmpeg -i input.opus -ar 44100 -c:a libopus output.wav

# Decode multichannel to 5.1 WAV
ffmpeg -i input.opus -c:a pcm_s16le -af "aresample=osf=flt" output_51.wav

# Get detailed codec information
ffprobe -show_streams -show_format input.opus

# Inspect OpusHead with hexdump
ffmpeg -i input.opus -c:v none -c:a copy -f data - | xxd | head -20

# Encode with FEC
ffmpeg -i input.wav -c:a libopus -b:a 32k -application voip -opus_fec 1 output.opus

# Encode with DTX
ffmpeg -i input.wav -c:a libopus -b:a 24k -opus_dtx 1 output.opus

# Two-pass VBR encoding (using libopusenc)
opusenc --bitrate 128 --vbr input.wav output.opus

# Encode with explicit frame size
ffmpeg -i input.wav -c:a libopus -frame_duration 20 output.opus
```

---

## APPENDIX A: REFERENCE IMPLEMENTATIONS

| Component | Implementation | Language | License |
|-----------|---------------|----------|---------|
| Opus codec | libopus | C | BSD |
| Ogg encapsulation | libopusenc | C | BSD |
| Matroska/WebM muxing | libmatroska + libebml | C++ | BSD |
| FFmpeg integration | FFmpeg (libavcodec) | C | LGPL/GPL |
| Python bindings | opuslib, pymedia | Python | Various |

## APPENDIX B: CRITICAL RFC REFERENCES

| RFC | Title | Purpose |
|-----|-------|---------|
| RFC 6716 | Opus Audio Codec | Core bitstream specification |
| RFC 7587 | RTP Payload Format for Opus | RTP encapsulation |
| RFC 7845 | Ogg Opus | Ogg container mapping |
| RFC 7874 | WebRTC Audio Codec Requirements | Mandatory-to-implement in WebRTC |
| RFC 7875 | Opus FEC | Forward Error Correction extension |

## APPENDIX C: COMMON PITFALLS

1. **Wrong pre-skip:** Many implementations hard-code pre-skip instead of reading it from OpusHead. This breaks gapless playback and causes sample-accurate seeking to fail.

2. **Ignoring input sample rate:** The OpusHead's `input_sample_rate` field is ignored by many implementations. If re-encoding, this information is needed to reconstruct the original sample rate.

3. **Multichannel channel routing:** Failing to apply the channel mapping table results in incorrect speaker assignment (e.g., center channel going to front left).

4. **CELT-only vs hybrid:** Treating all packets as CELT when the stream uses hybrid mode results in decode failures or corrupted output.

5. **48 kHz assumption:** Assuming Opus can encode at other sample rates. It cannot — the encoder must resample to 48 kHz first.

6. **FEC without FEC:** Enabling FEC on the encoder but not the decoder results in FEC data being silently discarded (the decoder ignores FEC if `decode_fec = 0`).

7. **Variable frame size:** Assuming all Opus frames are 20 ms. Frame size varies and must be derived from the TOC byte and config.

8. **Stereo in SILK:** SILK-only stereo uses mid-side coding, not independent channels. Decoders must handle this if decoding SILK-only streams.

---

*Document version: 1.0*
*Target audience: Audio codec developers building conversion pipelines*
*Specification: RFC 6716 (Opus), RFC 7845 (Ogg Opus), RFC 7587 (RTP), RFC 7875 (FEC)*
