# MP3 (MPEG-1 Audio Layer III) — Deep Technical Reference

> **Category:** Lossy Audio
> **File Extensions:** .mp3
> **MIME Types:** audio/mpeg, audio/mp3, audio/mpeg3, audio/x-mpeg-3
> **Standardization Body:** ISO/IEC 11172-3 (MPEG-1 Audio), ISO/IEC 13818-3 (MPEG-2)
> **Specification Document:** ISO/IEC 11172-3:1993, ISO/IEC 13818-3:1995
> **Patent Status:** Patented (many patent pools: MPEG LA, VIA Licensing)
> **License:** Patent-encumbered — requires licensing for commercial encoding

---

## 1. HISTORICAL CONTEXT & ORIGIN

MPEG-1 Audio Layer III (MP3) was developed by the Moving Picture Experts Group (MPEG), an ISO/IEC working group, with the first standard published in 1993 as part of MPEG-1 (ISO/IEC 11172-3). The standard defined three audio coding layers with increasing complexity and compression efficiency:

- **Layer I:** The simplest scheme, derived from PASC (Precision Subband Coding). Designed for digital compact cassette quality at high bitrates (192–384 kbps). Uses 384-sample frames with a 32-subband polyphase filter bank.
- **Layer II (MP2):** Enhanced version with better psychoacoustic modeling. Used in DAB (Digital Audio Broadcasting), Video CDs, and DVD-Video audio tracks. Bitrates of 128–256 kbps.
- **Layer III (MP3):** The most complex and efficient layer. Combines the polyphase filter bank with MDCT, Huffman coding, and advanced psychoacoustic modeling to achieve 10–12× compression with acceptable quality at 128 kbps.

The MP3 format gained early industry backing from Fraunhofer IIS and Thomson Multimedia (now Technicolor), who held critical patents. The reference encoder/decoder implementation by the Fraunhofer Institute became the de facto quality benchmark. The LAME project (LAME Ain't an MP3 Encoder), launched in 1998, reverse-engineered and significantly improved upon the reference encoders, producing near-transparent quality at VBR settings and eventually surpassed all proprietary encoders in quality. The "bitrate wars" of the early 2000s—where competing encoders were continuously benchmarked on Hydrogenaudio forums—drove rapid quality improvements across the ecosystem.

Napster (1999) popularized MP3 for peer-to-peer music sharing, bringing the format into mainstream culture despite widespread piracy concerns. Apple's iTunes Store (2003) then legitimized per-track MP3 sales. Today, MP3 remains the most universally supported audio format, found in streaming services, podcasts, mobile devices, gaming consoles, and automotive entertainment systems. Many patent licenses have expired or are offered royalty-free for certain uses, and encoding tools are freely available.

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

MP3 is classified as a **lossy, perceptual audio codec**. Its core principle is to remove or reduce audio information that is imperceptible to human hearing, using a sophisticated model of human auditory perception (the psychoacoustic model).

The codec architecture follows these key stages:

1. **Input:** 16-bit signed PCM audio, sampled at one of the defined MPEG sample rates.
2. **Filter Bank:** A two-stage hybrid filter bank divides the audio into frequency subbands.
3. **MDCT:** A Modified Discrete Cosine Transform further subdivides each subband into finer frequency resolution.
4. **Psychoacoustic Analysis:** Parallel FFT analysis computes masking thresholds.
5. **Quantization & Coding:** Non-uniform quantization with iterative rate/distortion loops, followed by Huffman coding.
6. **Bitstream Packing:** Frame assembly with headers, side information, and encoded data.

Key architectural characteristics:
- **Frame-based:** Each frame encodes 1152 PCM samples (MPEG-1 Layer III). Frames are independently decodable for streaming and random access.
- **Granules:** Each frame contains 2 granules (time slots). Granules allow temporal resolution adaptation within a frame.
- **Bit Reservoir:** A dynamic buffer allowing frames to borrow capacity from neighboring frames, enabling VBR without breaking frame structure.
- **Joint Stereo:** Optional channel coupling modes to reduce stereo redundancy.

MPEG-2 (ISO/IEC 13818-3) extended the standard with lower sample rates (22.05, 16, 11.025, 12, 8 kHz) and up to 5.1 multichannel audio. MPEG-2.5 is an unofficial but widely supported extension for very low sample rates (8, 11.025, 12 kHz), commonly used in voice recording and very low-bitrate applications.

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 MP3 File / Stream Structure

An MP3 file is a raw bitstream composed of consecutive MPEG audio frames, with optional metadata tags:

```
[ID3v2 Tag] [MPEG Frame 1] [MPEG Frame 2] ... [MPEG Frame N] [ID3v1 Tag]
```

**ID3v2 tag** is placed at the very beginning of the file (before the first audio frame). **ID3v1 tag** is placed at the very end (after the last audio frame). Both are optional; the audio data is valid without either.

Each MPEG audio frame has the following byte-level layout:

```
[Frame Header] [CRC (optional, 2 bytes)] [Side Information] [Main Data]
```

- **Frame Header:** Exactly 4 bytes (32 bits), present in every frame.
- **CRC:** 2 bytes only if the protection bit in the header is 0.
- **Side Information:** Variable length (17 bytes for mono, 32 bytes for stereo in MPEG-1 Layer III).
- **Main Data:** Variable length — contains scalefactors and Huffman-coded spectral data.

Frames are byte-aligned. Padding is achieved via the pad bit in the frame header, which adds one extra slot (1 byte for Layer II/III) to the frame length.

### 3.2 Frame Header Layout (4 bytes, 32 bits)

The bit layout follows the pattern: `AAAAAAAA AAABBCCD EEEEFFGH IIJJKLMM`

| Bit Position | Field Name | Width | Description |
|---|---|---|---|
| 31–21 | Frame Sync | 11 bits | Always `0x7FF` (all 11 bits set to 1) |
| 20–19 | Version (ID) | 2 bits | `00` = MPEG-2.5 (unofficial), `01` = reserved, `10` = MPEG-2, `11` = MPEG-1 |
| 18–17 | Layer | 2 bits | `00` = reserved, `01` = Layer III, `10` = Layer II, `11` = Layer I |
| 16 | CRC Protection | 1 bit | `0` = protected by CRC, `1` = not protected |
| 15–12 | Bitrate Index | 4 bits | See bitrate table below |
| 11–10 | Sample Rate Index | 2 bits | See sample rate table below |
| 9 | Padding Bit | 1 bit | `1` = frame is padded with one extra slot |
| 8 | Private Bit | 1 bit | Free for application use |
| 7–6 | Channel Mode | 2 bits | `00` = Stereo, `01` = Joint Stereo, `10` = Dual Mono, `11` = Mono |
| 5–4 | Mode Extension | 2 bits | Joint stereo method (intensity stereo, MS stereo) |
| 3 | Copyright | 1 bit | `1` = copyrighted |
| 2 | Original | 1 bit | `1` = original media |
| 1–0 | Emphasis | 2 bits | De-emphasis type: `00` = none, `01` = 50/15μs, `10` = reserved, `11` = CCIT J.17 |

**Frame sync detection:** In MPEG-1/MPEG-2, the sync word is 11 bits (`0x7FF`). For MPEG-2.5 compatibility, some implementations use a 12-bit sync pattern. The decoder searches for a byte of `0xFF` followed by a byte whose top 3–4 bits are set, then validates the rest of the header fields.

### 3.3 Complete Bitrate Table

| Bitrate Index | MPEG-1 Layer I | MPEG-1 Layer II | MPEG-1 Layer III | MPEG-2 Layer I | MPEG-2/2.5 Layer II/III | Unit |
|---|---|---|---|---|---|---|
| `0000` | free | free | free | free | free | — |
| `0001` | 32 | 32 | 32 | 32 | 8 | kbps |
| `0010` | 64 | 48 | 40 | 48 | 16 | kbps |
| `0011` | 96 | 56 | 48 | 56 | 24 | kbps |
| `0100` | 128 | 64 | 56 | 64 | 32 | kbps |
| `0101` | 160 | 80 | 64 | 80 | 40 | kbps |
| `0110` | 192 | 96 | 80 | 96 | 48 | kbps |
| `0111` | 224 | 112 | 96 | 112 | 56 | kbps |
| `1000` | 256 | 128 | 112 | 128 | 64 | kbps |
| `1001` | 288 | 160 | 128 | 144 | 80 | kbps |
| `1010` | 320 | 192 | 160 | 160 | 96 | kbps |
| `1011` | 352 | 224 | 192 | 176 | 112 | kbps |
| `1100` | 384 | 256 | 224 | 192 | 128 | kbps |
| `1101` | 416 | 320 | 256 | 224 | 144 | kbps |
| `1110` | 448 | 384 | 320 | 256 | 160 | kbps |
| `1111` | bad | bad | bad | bad | bad | — |

MPEG-1 max Layer III bitrate is 320 kbps. MPEG-2/2.5 max Layer III is 160 kbps.

### 3.4 Complete Sample Rate Table

| Version | Index `00` | Index `01` | Index `10` | Index `11` |
|---|---|---|---|---|
| MPEG-1 | 44100 Hz | 48000 Hz | 32000 Hz | reserved |
| MPEG-2 | 22050 Hz | 24000 Hz | 16000 Hz | reserved |
| MPEG-2.5 | 11025 Hz | 12000 Hz | 8000 Hz | reserved |

Frame length calculation (Layer III, MPEG-1):

```
frame_length_bytes = 144 × bitrate / sample_rate + padding_bit
```

For example: 128 kbps / 44100 Hz → `144 × 128000 / 44100 + 0 = 417` bytes. With padding: 418 bytes.

### 3.5 Side Information (MPEG-1 Layer III)

Side information immediately follows the frame header (and optional 2-byte CRC) and precedes the main data. Its size is **17 bytes for mono**, **32 bytes for stereo** (MPEG-1 Layer III).

Key fields:

| Field | Width (mono) | Width (stereo) | Description |
|---|---|---|---|
| `main_data_begin` | 9 bits | 9 bits | Negative byte offset from frame sync to where main data begins (bit reservoir pointer) |
| `private_bits` | 5 bits | 3 bits | Private use bits (round to byte boundary) |
| `scfsi` (channel × 4) | 4 × 1 bit | 4 × 1 bit | Scalefactor selection info: which granule shares scalefactors with granule 0 |
| Per granule, per channel: | | | |
| `part2_3_length` | 12 bits | 12 bits | Total bits for scalefactors (part 2) + Huffman data (part 3) |
| `big_values` | 9 bits | 9 bits | Number of coded frequency line pairs (×2 gives index) in big_values region (max 288 pairs) |
| `global_gain` | 8 bits | 8 bits | Quantizer step size |
| `scalefac_compress` | 4 bits | 4 bits | Determines slen1/slen2 for scalefactor encoding |
| `window_switching_flag` | 1 bit | 1 bit | Whether block type is non-standard |
| `block_type` | 2 bits | 2 bits | `0`=normal, `1`=start, `2`=short, `3`=stop |
| `mixed_block_flag` | 1 bit | 1 bit | Mix of long and short blocks |
| `table_select[3]` | 3 × 5 bits | 3 × 5 bits | Huffman table number for each region (0–31) |
| `subblock_gain[3]` | 3 × 3 bits | 3 × 3 bits | Gain offset for short blocks |
| `region0_count` | 4 bits | 4 bits | One less than scalefactor bands in region 0 |
| `region1_count` | 3 bits | 3 bits | One less than scalefactor bands in region 1 |
| `preflag` | 1 bit | 1 bit | Additional high-frequency scalefactor amplification |
| `scalefac_scale` | 1 bit | 1 bit | `0` = ×1, `1` = ×2 step size multiplier |
| `count1table_select` | 1 bit | 1 bit | Which count1 Huffman table (0 or 1) |

### 3.6 XING / VBRI Headers (VBR)

VBR MP3 files embed metadata headers in the first frame's main data area to enable seeking and duration calculation without scanning all frames.

**XING header** (used by LAME, FhG, and most encoders):
- Placed after side information in the first frame's main data
- Identified by ASCII string `"Xing"` (0x58 0x69 0x6E 0x67) or `"Info"` (for CBR files with info tag)
- Offset location: after side info in the first frame

| Offset | Size | Field | Description |
|---|---|---|---|
| 0 | 4 | Header ID | `"Xing"` or `"Info"` |
| 4 | 4 | Flags | Flags indicating present fields (OR'd): `0x0001`=frames, `0x0002`=bytes, `0x0004`=TOC, `0x0008`=quality |
| 8 | 4 | Frames | Total number of frames (Big-Endian DWORD) |
| 12 | 4 | Bytes | Total file size in bytes (Big-Endian DWORD) |
| 16 | 100 | TOC | Table of Contents: 100 entries, each 1 byte. TOC[i] = approximate file position for play position i/100 |
| 116 | 4 | Quality | Quality indicator: 0 (best) to 100 (worst) |

**VBRI header** (Fraunhofer-specific):
- Located at a fixed offset of 32 bytes after the first MPEG audio header (inside the first frame)
- Identified by `"VBRI"` (0x56 0x42 0x52 0x49)
- Contains: version, delay, quality, total bytes, total frames, TOC entries, TOC scale factor, TOC entry size, frames per TOC entry

Seeking via TOC: For position `p` (0–255), calculate file byte offset as `(TOC[p] / 256) × file_size`. Then resync to the nearest frame header.

### 3.7 Sample Format Support

| Parameter | MPEG-1 | MPEG-2 | MPEG-2.5 |
|---|---|---|---|
| Sample rates (Hz) | 32000, 44100, 48000 | 16000, 22050, 24000 | 8000, 11025, 12000 |
| Bitrates (kbps) | 8–320 (L3: 32–320) | 8–160 | 8–64 |
| Channels | 1 (mono), 2 (stereo), 1+1 (dual) | Up to 5.1 (MPEG-2 LS) | Up to 5.1 |
| Frame samples | 1152 (Layer III) | 1152 (Layer III) | 1152 (Layer III) |
| Frame duration | ~26ms (44.1kHz) | ~52ms (22.05kHz) | ~104ms (11.025kHz) |

---

## 4. ENCODING ALGORITHM (DEEP DETAIL)

### 4.1 Pre-Processing Stage

Input audio is PCM at 16-bit signed integer or 32-bit float precision. The encoding pipeline processes the audio in chunks of **1152 samples per channel** per frame (MPEG-1 Layer III).

Steps:
1. **Input buffering:** 1152 PCM samples per channel are accumulated.
2. **Pre-emphasis filter (optional):** A first-order high-shelf filter (coefficient ~0.5–0.7) that boosts high frequencies before quantization. Reversed at decode. Controlled by the `emphasis` flag, but modern encoders typically use `00` (none).
3. **Split into granules:** The 1152 samples are divided into two granules of 576 samples each.

Note: MP3 does **not** include Spectral Band Replication (SBR). SBR is a separate tool used in HE-AAC (MPEG-4 High Efficiency AAC).

### 4.2 Hybrid Filter Bank + MDCT

MP3 uses a two-stage hybrid filter bank to achieve both coarse frequency splitting and fine spectral resolution:

**Stage 1 — Polyphase Quadrature Mirror Filter (PQMF):**
- The input 576-sample granule passes through a 32-channel polyphase analysis filter bank.
- Each subband has bandwidth of `sample_rate / 64` Hz.
- Output: 32 subband samples per time slot (36 time slots per granule = 1152 total).
- The filter is implemented as a 512-sample windowed convolution followed by a 32-point DCT.
- Critically sampled: 576 input → 32 × 18 output per granule.

**Stage 2 — Modified Discrete Cosine Transform (MDCT):**
- Applied individually to each of the 32 subbands within a granule.
- Standard MP3 uses **18-point MDCT** per subband, producing 18 frequency coefficients per subband.
- Total: 32 × 18 = **576 frequency lines** per granule per channel.

**MDCT Window Sizes:**
| Block Type | Window | Coefficients | Use Case |
|---|---|---|---|
| 0 (normal) | Long window | 36 | Stationary signals |
| 1 (start) | Long window start | 36 + 12 | Transient start |
| 2 (short) | Short window | 12 × 3 | Transients |
| 3 (stop) | Long window stop | 12 + 36 | Transient end |

The **window switching** decision is made by the psychoacoustic model based on transient detection. Short blocks (block type 2) improve pre-echo control by reducing temporal spreading of quantization noise, at the cost of reduced frequency resolution.

### 4.3 Psychoacoustic Model

MPEG specifies two psychoacoustic models (Psychoacoustic Model 1 and Model 2), both producing identical output semantics. Model 1 is simpler (1024-point FFT on input), while Model 2 is more complex (2048-point FFT) and slightly more accurate.

Model 2 analysis steps:
1. Compute a 2048-point FFT on the input samples (centered on the granule being coded).
2. Apply a spreading function to simulate critical band masking.
3. Compute the **Absolute Threshold of Hearing (ATH)** — the minimum audible level per frequency, derived from an empirical formula that reaches its minimum (~3.5 dB SPL) around 2–4 kHz.
4. Combine simultaneous masking from tonal and noise-like components.
5. Calculate the **Signal-to-Mask Ratio (SMR)** per critical band — the ratio between signal energy and masking threshold.
6. Output: SMR per scalefactor band, which is passed to the bit allocation algorithm.

The psychoacoustic model also determines:
- Window type selection (long vs. short blocks)
- Threshold for pre-echo detection
- Bit allocation per band

### 4.4 Masking Threshold Calculation

**Absolute Threshold of Hearing (ATH):** The minimum audible sound pressure level, modeled empirically as:

```
ATH(f) ≈ 3.64 × (f/1000)^(-0.8) − 6.5 × e^(-0.6 × (f/1000−3.3)^2) + ATH_min
```

Where `ATH_min ≈ 0 dB` at 2–4 kHz. The ATH rises steeply at very low frequencies (~60 dB SPL at 20 Hz) and at high frequencies (~40 dB SPL at 16 kHz).

**Simultaneous (frequency-domain) masking:** A strong tone at frequency `f` creates a masking threshold that spreads upward and downward in frequency. The spreading function follows approximately a `1/3-critical-band` width dependency, creating a broader upward spread than downward.

**Temporal (time-domain) masking:** A loud sound masks sounds that occur up to ~200ms before (backward masking) and up to ~50–100ms after (forward masking) the masker. Forward (post-masking) temporal masking is significantly weaker than backward (pre-masking).

### 4.5 Bit Allocation Algorithm

The bit allocation in MP3 encoding is performed by an iterative nested-loop process:

1. **Outer loop (distortion control):** Starts with a set of quantization step sizes. Computes the resulting quantization noise per band. Compares noise against the SMR threshold. If noise exceeds threshold in any band, increases quantization step size (reduces quality slightly) to free bits.
2. **Inner loop (rate control):** Huffman encodes the quantized coefficients and counts the bits produced. If bits exceed budget, increases the global quantizer step.

The algorithm iterates between loops until both constraints are satisfied: quantization noise is below masking threshold AND bit consumption fits within the frame budget (including bit reservoir). This is the core of MP3's perceptual quality — it dynamically trades frequency resolution for bitrate within each frame.

### 4.6 Quantization

MP3 uses **compound non-uniform quantization** where the quantized value `v` is:

```
v = sign(x) × |x|^(3/4)
```

The `3/4` power compacts the dynamic range of audio coefficients, which tend to have Laplacian (exponential) distribution. The inverse operation at decode is `x = sign(v) × |v|^(4/3)`.

**Global gain** (8 bits): Controls the overall quantizer step size. Applied uniformly across all frequency lines.

**Scalefactors** (per-scalefactor-band): Amplify or attenuate groups of frequency lines before quantization. Scalefactor bands are aligned with critical bands at low frequencies and become wider at high frequencies. There are 21 scalefactor bands in long blocks, 12 in short blocks. Each scalefactor is transmitted as a small integer (4–11 bits depending on `scalefac_compress`). Applied as a multiplicative factor in the requantization formula.

**Scalefactor scaling:** `scalefac_scale = 0` → step size multiplier = 1; `scalefac_scale = 1` → step size multiplier = 2.

**Preflag:** For long blocks, if set, adds predefined amplification values (from a table) to high-frequency scalefactors, equivalent to a high-shelf pre-emphasis.

### 4.7 Huffman Coding

After quantization, the 576 spectral coefficients per granule per channel are Huffman coded. The frequency spectrum is divided into regions:

**Big values region** (largest magnitude coefficients):
- Covers the lowest-frequency portion of the spectrum.
- Divided into 3 sub-regions (region0, region1, region2) based on `region0_count` and `region1_count`.
- Each region uses one of **32 Huffman code tables** selected via `table_select[]`.
- Tables are optimized for different amplitude distributions (small values, large values, etc.).
- Maximum absolute value per coefficient: **8191**.
- Escaped values: coefficients exceeding the table's maximum are stored as escape codes followed by raw bits.

**Count1 region**:
- Covers the next portion of the spectrum (lower amplitude coefficients).
- Encodes **quadruples of values** (4 coefficients at a time), each being {-1, 0, +1}.
- Uses one of **2 Huffman tables** selected by `count1table_select`.
- Each quadruple consumes 1–2 Huffman codewords.

**Rzero region**:
- Remaining high-frequency coefficients are assumed to be zero.
- Not explicitly coded — simply the decoder fills this region with zeros.
- Encoder counts consecutive zero pairs from the high-frequency end.

### 4.8 Joint Stereo Encoding

MP3 supports three channel mode configurations:

| Mode | Code | Description |
|---|---|---|
| Stereo | `00` | Left and right channels encoded independently |
| Joint Stereo | `01` | Mid/Side or Intensity stereo per subband |
| Dual Mono | `10` | Two independent mono channels |
| Mono | `11` | Single channel |

In **Joint Stereo** mode (`mode_extension` bits control details):

- **Mid/Side (MS) stereo:** Encode M = (L+R)/2, S = (L−R)/2. More bits are allocated to the mid channel than the side channel, which often has lower energy. MS stereo is particularly effective for centered content (vocals, bass).

- **Intensity stereo:** For high-frequency bands only. Instead of encoding individual channel coefficients, encode a single mono signal plus an intensity parameter and a position indicator. The decoder reconstructs stereo by applying the intensity to the appropriate frequency region. Lossy — destroys phase information at high frequencies.

Modern encoders like LAME use **adaptive joint stereo**: they switch between MS stereo and LR stereo on a frame-by-frame and band-by-band basis, applying MS stereo only where it provides a net quality benefit.

### 4.9 Bitrate and Quality Modes

**CBR (Constant Bitrate):** Every frame uses the same bitrate. Simple for streaming and seeking, but cannot adapt to signal complexity. Produces inefficient encoding on simple passages and may be insufficient on complex passages.

**VBR (Variable Bitrate):** Quality target is specified (e.g., LAME's `-V` option, 0=best to 9=worst). The encoder dynamically allocates more bits to complex passages and fewer to simple passages, producing a file whose size depends on content complexity at the target quality level. LAME VBR presets:

| Preset | Approx. Avg. Bitrate | Target Quality |
|---|---|---|
| `-V 0` | ~245 kbps | Transparent (except most critical) |
| `-V 1` | ~225 kbps | Near-transparent |
| `-V 2` | ~190 kbps | Very high |
| `-V 3` | ~175 kbps | High |
| `-V 4` | ~165 kbps | Standard |
| `-V 5` | ~130 kbps | Medium |
| `-V 6` | ~115 kbps | Low |
| `-V 7` | ~100 kbps | Poor |
| `-V 8` | ~85 kbps | Very poor |
| `-V 9` | ~65 kbps | Worst |

**ABR (Average Bitrate):** Target average bitrate specified (e.g., `-b 192k --abr`). Bitrate can vary between frames but maintains the specified average. Uses less bit reservoir than VBR, making it more compatible with hardware players that have limited buffering.

---

## 5. DECODING ALGORITHM

### 5.1 Stream Synchronization

Decoding begins by locating the start of the first valid MPEG audio frame:

1. Search for `0xFF` byte in the file (most significant 3 bits of next byte must be `0x7`).
2. Read the 4-byte frame header and validate all fields.
3. Validate by checking: version not `01` (reserved), layer not `00` (reserved), bitrate not `1111` (bad), sample rate not `11` (reserved), emphasis not `10` (reserved).
4. For robustness, confirm at least 2–3 consecutive valid frame headers before committing to a sync position.
5. Skip any ID3v2 tag at the beginning (identified by `"ID3"` magic bytes).

**Resynchronization after errors:** After a CRC failure or decode error, search forward for the next `0x7FF` sync pattern and validate the new frame header.

### 5.2 Core Decoding Pipeline

The complete decode pipeline for each frame:

```
1. Frame sync → read 4-byte header
2. Parse header: version, layer, bitrate, sample rate, channel mode, padding
3. Read optional CRC (2 bytes) if protection bit = 0
4. Read side information (17 bytes mono, 32 bytes stereo)
5. Read scalefactors from main data (granule 0, granule 1)
6. Read Huffman-coded spectral data
7. Huffman decode → quantized MDCT coefficients
8. Requantize → apply global_gain, scalefactors, preflag
9. Reorder short blocks (if block_type == 2)
10. Inverse MDCT → time-frequency transform
11. Frequency-to-time aliasing reduction (butterfly operations)
12. Polyphase synthesis filter bank → 32 subband samples
13. Output: 1152 PCM samples per channel
```

### 5.3 Inverse MDCT

The inverse MDCT (IMDCT) transforms frequency-domain coefficients back to time-domain subband samples:

For each block of N/2 output samples from a block of N input coefficients:

```
y[n] = w[n] × Σ(k=0 to N-1) X[k] × cos(π/N × (n + 0.5) × (k + 0.5))
```

Where `w[n]` is the sine window function:

```
w[n] = sin(π/N × (n + 0.5))
```

For long blocks (N=36): A 50% overlap-add between consecutive IMDCT outputs reconstructs the time domain. Each block produces N output samples, but consecutive blocks overlap by N/2, so only half of each block's output is new.

For short blocks (N=12): Three short blocks are processed per long block period, each producing 6 new samples (with 50% overlap within the short block). The three short block outputs are windowed and overlapped-add within each scalefactor band region before the final polyphase stage.

### 5.4 Synthesis Filter Bank (Polyphase)

The synthesis filter bank reconstructs 32 time-domain subband samples from the 32 frequency-domain subband coefficients produced by the inverse MDCT.

Implementation uses an inverse PQMF (polyphase quadrature mirror filter) combined with a windowed 512-point synthesis stage:

1. **DCT-based synthesis:** The 32 subband samples are transformed using a 32-point IDCT (Type-IV DCT).
2. **Overlap-add:** The output is a 512-sample windowed vector. Successive windows are overlapped and summed, producing continuous output.
3. **Aliasing reduction:** Butterfly operations in the filter bank undo the aliasing introduced by the analysis PQMF.
4. **Output:** 32 new PCM samples per subband iteration.

The filter bank produces 32 output samples per subband, and the decoding process iterates 18 times per granule (18 × 32 = 576 samples), then twice for two granules = 1152 output samples per frame per channel.

### 5.5 Output Reconstruction

Standard output is **16-bit signed PCM** (little-endian or big-endian depending on container). Internally, decoders often use 32-bit float precision for intermediate calculations to avoid clipping and preserve quality during processing.

Clipping can occur when:
- Encoded material has peaks near or at 0 dBFS
- ReplayGain replay adjustment boosts the signal
- Multiple decoded frames are summed during overlap-add

Decoders may apply soft clipping or limiting to prevent harsh digital clipping artifacts.

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native MP3 Container

The native "container" for MP3 is a raw bitstream — no wrapper is required. Frames are simply concatenated sequentially. Metadata is embedded via optional tags:

- **ID3v2:** Placed at the file beginning, before audio frames. Tag data follows the frame sync, so players must skip it during playback.
- **ID3v1:** Placed at the file end, after all audio frames. 128 bytes.
- **LYRICS3 v1 and v2:** Optional lyric extension placed before or after ID3v1.
- **APE tag:** Can be appended after ID3v1 in some implementations.
- **PRI tag:** LAME encoder info stored as an INFO tag in the first frame's main data.

### 6.2 MP3 in Other Containers

| Container | Metadata System | Seeking | Notes |
|---|---|---|---|
| Raw bitstream (.mp3) | ID3v2 + ID3v1 | Via XING/VBRI TOC or CBR formula | Native |
| MP4 (.m4a) | ID3v2 in mdat or `moov`/`udta` | Supported | Rare; non-standard |
| Matroska (.mka) | Matroska Tags (EBML) | Full seeking | Supported; high quality |
| OGG | Not recommended | Limited | Not standardized |
| ADTS (.aac stream) | No tag system | Via TOC or scan | Raw AAC frames for streaming |
| MPEG-TS | PAT/PMT descriptors | Frame-level | Broadcasting |

MP3 embedded in MP4 is non-standard and typically not recommended. For surround audio or advanced metadata, AAC in MP4 is the proper successor format.

---

## 7. METADATA ARCHITECTURE

### 7.1 ID3v2 Tag

The ID3v2 tag is placed at the beginning of the file. It was designed to be safe for streaming applications — the tag does not contain MPEG sync bytes, so players can simply skip it to reach the audio.

**ID3v2 Header (10 bytes):**

| Offset | Size | Field | Value |
|---|---|---|---|
| 0 | 3 | Identifier | `"ID3"` (0x49 0x44 0x33) |
| 3 | 1 | Version | 0x03 (v2.3) or 0x04 (v2.4) |
| 4 | 1 | Revision | 0x00–0xFF |
| 5 | 1 | Flags | b7=footer present, b6=experimental, b5=extended header |
| 6 | 4 | Size | Synchsafe integer (v2.4) or big-endian (v2.3) |

**Synchsafe integer (ID3v2.4):** Each byte uses only 7 bits (MSB always 0). A 32-bit synchsafe stores 28 effective bits (max ~256 MB). To decode: `value = (b0 << 21) | (b1 << 14) | (b2 << 7) | b3`.

**ID3v2.3 vs v2.4 key differences:**
- v2.3: Frame size is big-endian 32-bit. Frames can have 2-byte flags. Terminates text strings with `$00`.
- v2.4: Frame size is synchsafe 32-bit. Frames can have 2-byte status/format flags. Supports multiple text encodings. Terminates strings per encoding rule.
- v2.4 also allows frames to be unsynchronized (bytes `0xFF 0x00` inserted to prevent false sync).

### 7.2 Supported ID3v2 Frames (ID3v2.3/2.4)

| Frame ID | Meaning | Description |
|---|---|---|
| TIT2 | Title | Song title |
| TPE1 | Lead Artist | Primary performer(s), separated by `/` |
| TALB | Album | Album name |
| TRCK | Track Number | Track number and optional total (e.g., `4/9`) |
| TPOS | Part of Set | Disc number in a multi-disc album |
| TYER | Year | Recording year (4-digit string) |
| TDRC | Recording Time | Recording time (ID3v2.4, supersedes TYER, TDAT, TIME) |
| TCON | Genre | Genre as text or numeric index (ID3v1.1 genre list) |
| TPE2 | Band/Orchestra | Additional performer info |
| TPE3 | Conductor | Conductor/performer refinement |
| TPE4 | Remixer | Interpreted, remixed, or modified by |
| TCOM | Composer | Music composer(s) |
| TEXT | Lyricist | Lyricist/text writer |
| TKEY | Initial Key | Initial musical key (e.g., `C#m`) |
| TBPM | BPM | Beats per minute |
| TLEN | Length | Length in milliseconds |
| TMED | Media Type | Source media type (e.g., `CD`) |
| TENC | Encoded By | Software/hardware used for encoding |
| TPUB | Publisher | Record label/publisher |
| TCOP | Copyright | Copyright message |
| TSOA | Album Sort Order | Album name for sorting (ID3v2.4) |
| TSOP | Artist Sort Order | Artist name for sorting (ID3v2.4) |
| TSOT | Title Sort Order | Title for sorting (ID3v2.4) |
| APIC | Attached Picture | Cover art: encoding byte, MIME type, picture type (0–20), description, binary image data |
| COMM | Comment | Encoding byte, language (3 chars), short description, full comment text |
| USLT | Unsynchronized Lyrics | Unsynchronized lyrics text |
| SYLT | Synchronized Lyrics | Synchronized lyrics with timestamps |
| UFID | Unique File Identifier | Owner ID + identifier bytes |
| MCDI | Music CD Identifier | Binary TOC data from CD |
| PRIV | Private Frame | Private use frame: owner ID + private data |
| TXXX | User-Defined Text | User-defined text info frame |
| WXXX | User-Defined Link | User-defined URL frame |
| WCOM | Commercial Info | Commercial information URL |
| WOAR | Official Artist URL | Artist's official website |
| WOAS | Official Audio Source URL | Official audio source website |
| WORS | Official Radio Station | Official radio station URL |

Text frames begin with a **text encoding byte**: `$00` = ISO-8859-1, `$01` = UTF-16 with BOM, `$02` = UTF-16BE, `$03` = UTF-8.

### 7.3 ID3v1 Tag

Located at the last 128 bytes of the file. Identified by `"TAG"` magic bytes at offset 0.

| Offset | Size | Field | Notes |
|---|---|---|---|
| 0 | 3 | Identifier | `"TAG"` |
| 3 | 30 | Title | Padded with spaces or null |
| 33 | 30 | Artist | Padded with spaces or null |
| 63 | 30 | Album | Padded with spaces or null |
| 93 | 4 | Year | 4-character string |
| 97 | 30 | Comment | Padded with spaces. In v1.1: byte 97 = `$00`, bytes 98–99 = track number |
| 127 | 1 | Genre | Index into ID3v1 genre list (0–79 standard, 80–255 extended by WinAmp) |

### 7.4 LAME / Encoder Tags

**LAME info tag (stored as INFO tag in first frame's main data):**
- ASCII string `"LAME"` followed by version (e.g., `LAME3.99r`)
- Contains: encoder version, tag revision, VBR quality, lowpass filter value, replaygain fields, encoder delay/padding

**ReplayGain in MP3:**
- Stored in an `RGAD` (ReplayGain Adjustment Data) subframe within an `INFO` or `TXXX` frame
- Fields: peak amplitude, track gain (dB), album gain (dB)
- ReplayGain 1.0 specification: track gain +89 dB = reference level (83 dB SPL perceived)

**iTunSMPB tag** (Apple gapless metadata, stored in a comment frame):
- Hex-encoded string: `00000000 00000000 00000000 00000000 NNNNNNNN NNNNNNNN PPPPPPPP`
- Format: `encoder delay` (hex), `encoder padding` (hex), `num samples` (hex)
- All values are in samples at 44.1 kHz

### 7.5 Metadata Encoding Matrix

| Tag System | Read Support | Write Support | Lossless Round-trip | Notes |
|---|---|---|---|---|
| ID3v2.4 | Universal | Universal | Yes (preserve all frames) | Most common |
| ID3v2.3 | Universal | Universal | Partial (size encoding differs) | Widely compatible |
| ID3v1.1 | Universal | Universal | No (limited fields, truncation) | Legacy fallback |
| LAME INFO | LAME/FFmpeg | LAME/FFmpeg | Yes | Encoder info + gapless |
| iTunSMPB | Apple software | Apple software | Yes | Apple gapless |
| APE v2 | Limited | Limited | Yes | Rare, third-party |

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 Codec Identifiers

| FFmpeg Name | Codec ID | Description |
|---|---|---|
| `libmp3lame` | `AV_CODEC_ID_MP3` (encoder) | LAME-based MP3 encoder |
| `mp3` | `AV_CODEC_ID_MP3` (decoder) | Native MP3 decoder (fixed-point) |
| `mp3adufloat` | `AV_CODEC_ID_MP3ADU` | MP3 Audio Decoding Unit (for RTP) |
| `mp3on4float` | `AV_CODEC_ID_MP3ON4` | MP3 in MP4 container |

### 8.2 FFmpeg Encoding — Full CLI Reference

```bash
# Basic MP3 encoding (LAME, default quality)
ffmpeg -i input.wav -c:a libmp3lame output.mp3

# CBR at 320 kbps (maximum quality)
ffmpeg -i input.wav -c:a libmp3lame -b:a 320k output.mp3

# VBR quality (0=best, 9=worst)
ffmpeg -i input.wav -c:a libmp3lame -q:a 2 output.mp3

# ABR at 192 kbps (average bitrate)
ffmpeg -i input.wav -c:a libmp3lame -b:a 192k -abr 1 output.mp3

# Joint stereo explicitly enabled (default)
ffmpeg -i input.wav -c:a libmp3lame -joint_stereo 1 output.mp3

# Plain stereo (no joint stereo)
ffmpeg -i input.wav -c:a libmp3lame -joint_stereo 0 output.mp3

# Disable bit reservoir (strict CBR frame sizes)
ffmpeg -i input.wav -c:a libmp3lame -reservoir 0 output.mp3

# Mark as copyrighted
ffmpeg -i input.wav -c:a libmp3lame -copyright 1 output.mp3

# Mark as non-original (copy)
ffmpeg -i input.wav -c:a libmp3lame -original 0 output.mp3

# Combine options: ABR with joint stereo disabled
ffmpeg -i input.wav -c:a libmp3lame -b:a 256k -abr 1 -joint_stereo 0 output.mp3

# Encode with ReplayGain (track gain applied during decode)
ffmpeg -i input.wav -af replaygain=track=1 -c:a libmp3lame output.mp3
```

**Full table of libmp3lame encoder options:**

| Option | Type | Default | Range | Effect |
|---|---|---|---|---|
| `b` (`-b:a`) | integer | 0 (VBR default) | 8–320 kbps | Target bitrate for CBR/ABR |
| `q` (`-q:a`) | float | 2 (FFmpeg) | 0.0–9.999 | VBR quality (0=best) |
| `abr` | boolean | 0 | 0–1 | Enable ABR mode |
| `joint_stereo` | boolean | 1 | 0–1 | Enable joint stereo encoding |
| `reservoir` | boolean | 1 | 0–1 | Enable bit reservoir usage |
| `copyright` | boolean | 0 | 0–1 | Set copyright flag in frame header |
| `original` | boolean | 1 | 0–1 | Set original flag in frame header |

Note: `-q:a` maps to LAME's `-V` option (inverted: LAME `-V 0` ≈ FFmpeg `-q:a 0`). FFmpeg's default VBR quality of 2 corresponds roughly to LAME `-V 2` (~190 kbps average).

### 8.3 FFmpeg Decoding

```bash
# Decode to 16-bit PCM WAV
ffmpeg -i input.mp3 -c:a pcm_s16le output.wav

# Decode to 24-bit PCM
ffmpeg -i input.mp3 -c:a pcm_s24le output.wav

# Decode to float WAV
ffmpeg -i input.mp3 -c:a pcm_f32le output.wav

# Probe stream info
ffprobe -v quiet -print_format json -show_streams input.mp3

# Probe format and metadata
ffprobe -v quiet -show_format -show_streams input.mp3

# Extract audio without re-encoding
ffmpeg -i input.mp3 -c:a copy output_stream.mp3

# Extract with format conversion
ffmpeg -i input.mp3 -f s16le -acodec pcm_s16le -
```

### 8.4 FFmpeg Metadata Handling

```bash
# Read all metadata
ffprobe -v quiet -show_format input.mp3

# Write specific tags during encoding
ffmpeg -i input.wav -c:a libmp3lame -metadata title="Song Title" \
  -metadata artist="Artist Name" -metadata album="Album Name" \
  -metadata year=2024 output.mp3

# Copy metadata during transcoding
ffmpeg -i input.mp3 -i source.wav -map_metadata 0 \
  -c:a libmp3lame -q:a 2 output.mp3

# Remove all metadata
ffmpeg -i input.mp3 -map_metadata -1 -c:a copy output.mp3
```

FFmpeg handles ID3v2.4 tags natively. For ID3v1, it reads and writes automatically when the output format supports it. The `-id3v2_version` option controls which ID3v2 version to use (default 3).

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seeking Architecture

MP3 does not have a built-in seek index. Seeking is achieved through three methods:

1. **XING/VBRI TOC (VBR files):** The TOC table in the first frame provides 100 (XING) or variable (VBRI) position markers. Interpolate between markers to estimate the byte offset for a given play position. Accuracy: typically within ~0.1 seconds.

2. **CBR formula (CBR files):** Frame N's byte offset = `N × frame_size`. Since frame size is constant, this is deterministic. No TOC needed.

3. **Frame scanning:** For VBR files without TOC, scan sequentially from the beginning or from a known reference point, counting frames. Very slow but always works.

The seeking algorithm with XING TOC:
```
target_percent = desired_playback_position / total_duration
toc_entry = TOC[int(target_percent * 100)]
byte_offset = (toc_entry / 256) × total_file_size
→ seek to byte_offset
→ find nearest frame sync
→ decode from that frame
```

### 9.2 Gapless Playback

MP3 introduces **encoder delay** and **encoder padding** that create silent portions at the start and end of decoded output. These are inherent to the frame-based processing and filter bank initialization.

**LAME encoder behavior:**
- Delay: **576 samples** (before the first audio sample)
- Padding: **1152 samples** (after the last audio sample)

The LAME tag stores these values so compliant decoders/players can trim them:

```
trim_start = encoder_delay / sample_rate  # seconds to skip from beginning
trim_end = (encoder_samples - encoder_delay - valid_samples) / sample_rate  # seconds to skip from end
```

**iTunSMPB format:** Apple stores gapless metadata as a 36-character hex string in the `iTunSMPB` comment frame:
```
00000300 = 768 samples delay (standard iTunes)
00000800 = 2048 samples padding (standard iTunes)
```

**Rule of thumb for gapless:** When concatenating LAME-encoded MP3s, ignore the first 528 samples and last 696 samples of each file's decoded output. This corresponds to the 48-sample pre-FFT delay and the padding region.

---

## 10. STREAMING & REAL-TIME

- **Streaming:** Fully supported. Frames are independently decodable (granules within a frame may depend on bit reservoir, but frames themselves are self-contained for header parsing).
- **Latency:** ~26ms per frame (1152 samples at 44.1 kHz). Total encoder latency including psychoacoustic look-ahead: ~50–100ms.
- **HTTP streaming:** Supported via HTTP range requests. Servers can serve byte ranges, and clients request specific frames.
- **HLS/DASH:** MP3 is a valid codec for streaming playlists. Support varies by player.
- **RTP:** MP3 frames can be packetized directly into RTP. `AV_CODEC_ID_MP3ADU` handles MP3 Audio Decoding Units for RTP applications.

---

## 11. MULTI-CHANNEL & SURROUND

- **MPEG-1:** Stereo and mono only. No multichannel support.
- **MPEG-2 LS (Low Sampling Rate):** Extension for multichannel audio (up to 5.1) at reduced sample rates (max 32/48 kHz). Uses matrixing (Dolby Surround compatible) and reduced spectral resolution at high frequencies. Rarely used.
- **MPEG-2 BC (Backward Compatible):** Original MPEG-2 multichannel extension with full backward compatibility to stereo decoders.

In practice, MP3 is not used for surround audio. The industry standard for multichannel audio is AAC (MP4/M4A), AC3 (Dolby Digital), DTS, or Dolby Digital Plus.

---

## 12. TRANSPARENT BITRATE & QUALITY

### 12.1 Transparent Bitrate

Transparency (perceptually lossless) means that a trained listener cannot distinguish the MP3 from the original in an ABX test.

| Content Type | Transparent Bitrate | Notes |
|---|---|---|
| General music (LAME) | ~192–256 kbps | VBR preferred |
| LAME V0 (≈245 kbps avg) | Near-transparent | Most critical material may still be detectable |
| LAME V2 (≈190 kbps avg) | High quality | Transparent for ~90% of listeners |
| LAME V5 (≈130 kbps avg) | Standard quality | Noticeable artifacts on difficult material |
| Classical / acoustic | 256–320 kbps | Full orchestral dynamics require higher rates |
| Voice / speech | 64–96 kbps | More than sufficient |
| Lossless reference | N/A | 1411 kbps (CD audio) |

### 12.2 Listening Tests

The **Hydrogenaudio** community has conducted extensive MP3 listening tests since 2000. Key findings:
- LAME 3.90+ achieves transparency at **128 kbps** for most content (v5 = VBR quality target).
- Critical content (acoustic instruments, spatial cues, high-frequency detail) requires 192+ kbps.
- Reference encoders (FhG, Xing) need 192–256 kbps for equivalent quality.
- Modern LAME VBR is superior to all pre-2005 proprietary encoders at any bitrate.

### 12.3 Artifact Description

| Artifact | Cause | Characteristics |
|---|---|---|
| **Pre-echo / Ringing** | Long MDCT blocks capturing transients; quantization noise spread before the attack | Audible noise preceding a sharp attack (e.g., drums, castanets). Solution: short blocks. |
| **Birdie noise** | Bit-allocation threshold shifting between frames causes spectral coefficients to appear/disappear | Random high-frequency chirping or twittering, especially at 100–200 kbps |
| **Swishy/wet artifacts** | Aggressive quantization of sibilants, cymbals, and high-frequency transient content | Hiss-like distortion on "s" sounds, cymbals sound grainy or watery |
| **Pumping** | Over-aggressive bit allocation; bass or overall level varies with the bit budget | Perceptible volume fluctuation, often in sync with beat |
| **Stereo unmasking / Imaging** | At very low bitrates, joint stereo intensity coding destroys stereo separation | Loss of stereo image, reduced spatial width, center-channel effect |
| **Metallic noise** | Quantization noise landing in critical frequency bands near masking threshold | Harsh, tinny quality on certain instruments |
| **Silence truncation** | Encoder padding not handled by player | Tiny gaps between tracks, audible clicks at track boundaries |

---

## 13. RE-ENCODING DEGRADATION

Transcoding (MP3 → MP3) causes **generational quality loss** because each lossy encode discards information irreversibly. The process:

1. **Decode MP3** → PCM (approximate original, already lossy approximation)
2. **Re-encode PCM** → MP3 (further quantization, new artifacts)

Each generation:
- Accumulates quantization artifacts (pre-echo, birdie noise)
- Loses high-frequency detail
- Reduces effective bitrate (even at same nominal bitrate)
- Widens quantization noise bands

After 2–3 generations at 128 kbps, artifacts become clearly audible to most listeners. After 5+ generations, the audio quality can degrade to near-AM radio quality.

**Rule:** Never transcode lossy sources. Re-encode from the original lossless source whenever possible.

---

## 14. KNOWN ISSUES & EDGE CASES

- **Bit reservoir overflow:** If a frame needs more bits than the reservoir can supply (max 511 bytes back-referenced), excess bits spill into subsequent frames, causing frame-dependent data. This breaks strict frame independence for editing.
- **Malformed headers:** Non-MPEG data can contain false sync bytes. Robust players check header validity across multiple consecutive frames before committing to a sync position.
- **Crippled audio (censorship):** Some "censored" MP3s inject silence frames into specific portions while keeping file duration unchanged. Detected by analyzing frame bitrate — silence frames use minimum bitrate (32 kbps for MPEG-1 L3).
- **ID3v2.4 unsynchronization:** When encoding to ID3v2.4, the tag data may include `0xFF 0x00` sequences that must be skipped during playback. Some decoders handle this incorrectly.
- **Frame CRC errors:** Many decoders silently skip corrupted frames rather than applying concealment. This produces audible glitches or dropouts.
- **ID3v2 footer:** If the footer flag is set, a 10-byte footer is appended after the tag data. Some tag editors incorrectly parse this as frame data.
- **Mixed MPEG versions:** Some files have MPEG-2 frames at the beginning (LAME sometimes prepends a silent/vbr-info frame at MPEG-2 rate) and MPEG-1 frames thereafter. Players must handle version switching.
- **Bitrate switching:** In VBR files, not all frames have the same bitrate. Seeking without TOC requires scanning or interpolation.
- **Dual-mono:** Dual channel mode encodes two independent mono signals. Joint stereo cannot be used. Some decoders incorrectly treat dual-mono as stereo.

---

## 15. CONVERSION GUIDE (FFmpeg Context)

### 15.1 Converting FROM MP3

| Target Format | Recommended Command | Quality Notes | Metadata Preservation |
|---|---|---|---|
| **FLAC** | `ffmpeg -i input.mp3 -c:a flac -compression_level 8 output.flac` | Lossless; no quality loss | `-c copy` preserves all ID3v2 frames |
| **WAV (PCM)** | `ffmpeg -i input.mp3 -c:a pcm_s16le output.wav` | Perfect decode to PCM | No metadata (wav doesn't support ID3) |
| **AAC (M4A)** | `ffmpeg -i input.mp3 -c:a aac -b:a 256k output.m4a` | Depends on bitrate; aac is successor | Use `-c copy` for lossless metadata transfer |
| **Opus** | `ffmpeg -i input.mp3 -c:a libopus -b:a 192k output.opus` | Excellent at low bitrates | Opus comments (Vorbis-style) |
| **OGG Vorbis** | `ffmpeg -i input.mp3 -c:a libvorbis -q:a 6 output.ogg` | Comparable quality to MP3 at lower bitrate | Vorbis comments |

### 15.2 Converting TO MP3

| Source Format | Recommended Command | Quality Notes | Metadata Preservation |
|---|---|---|---|
| **FLAC** → | `ffmpeg -i input.flac -c:a libmp3lame -q:a 2 output.mp3` | VBR ~190 kbps; very high quality | ID3v2.4 written automatically |
| **WAV** → | `ffmpeg -i input.wav -c:a libmp3lame -b:a 320k output.mp3` | Maximum quality CBR at 320 kbps | No metadata unless added |
| **AAC/M4A** → | `ffmpeg -i input.m4a -c:a libmp3lame -q:a 2 output.mp3` | Transcoding from lossy: avoid if possible | Preserve with `-map_metadata 0` |
| **AIFF** → | `ffmpeg -i input.aiff -c:a libmp3lame -q:a 2 output.mp3` | Same as WAV; AIFF is lossless PCM | No metadata unless added |

### 15.3 ReplayGain

ReplayGain 1.0 (EBU R128 successor) stores gain and peak values in MP3 tags:

```bash
# Calculate and write ReplayGain (track gain + peak)
ffmpeg -i input.mp3 -af replaygain=track=1:drop_version=0 -c:a copy output.mp3

# Calculate album gain (all tracks as an album)
ffmpeg -i track01.mp3 -i track02.mp3 -filter_complex \
  "areplaygain=album=1:drop_version=0" \
  -c:a copy -map_metadata 0 output.mp3

# Probe existing ReplayGain values
ffprobe -v quiet -show_format input.mp3 | grep -E "REPLAYGAIN"
```

**ReplayGain values in tags:**
- `REPLAYGAIN_TRACK_GAIN`: Track adjustment in dB (e.g., `-4.5 dB`)
- `REPLAYGAIN_TRACK_PEAK`: Peak amplitude as a decimal (e.g., `0.56478912`)
- `REPLAYGAIN_ALBUM_GAIN`: Album adjustment in dB
- `REPLAYGAIN_ALBUM_PEAK`: Album peak amplitude

Players apply gain by adjusting playback volume. If peak × gain exceeds 0 dBFS, clipping occurs. FFmpeg's `areplaygain` filter prevents clipping by limiting the gain.

---

## 16. REFERENCE IMPLEMENTATIONS

| Library | Language | License | Quality | Notes |
|---|---|---|---|---|
| **LAME** | C | LGPL v2+ | Best | Reference-quality encoder; de facto standard for high-quality encoding. Active development since 1998. |
| **FFmpeg native (MP3)** | C | LGPL/GPL | High (decoder) | Fixed-point decoder; encoder uses built-in version. Excellent for transcoding. |
| **MAD (libmad)** | C | GPL | High | Fixed-point MP3 decoder; very accurate, no float dependencies. Popular for embedded systems. |
| **mpg123** | C | GPL | High | Highly optimized decoder; widely used in Linux and embedded applications. |
| **TwoLAME** | C | LGPL | High | MPEG-1 Layer II only (not Layer III). For MP2 encoding. |
| **FAAD2** | C | BSD | High | Open-source AAC/MP4 decoder; MP3 is not its focus. |
| **FAAD** | C | Custom | Medium | Older open-source AAC decoder. |

---

## 17. SPECIFICATIONS & FURTHER READING

### Primary Standards
- **ISO/IEC 11172-3:1993** — MPEG-1 Audio Layer I, II, III. The definitive specification for MPEG-1 audio.
- **ISO/IEC 13818-3:1995** — MPEG-2 Audio (backward compatible). Extensions for multichannel and lower sample rates.
- **ISO/IEC 14496-3** — MPEG-4 Audio. Covers AAC and its extensions (HE-AAC, LD-AAC).

### Reference Documents
- **LAME Technical Docs:** `https://lame.cvs.sourceforge.net/viewvc/lame/lame/doc/html/` — LAME encoding internals, switches, and technical descriptions.
- **MP3 Technology:** `http://www.mp3-tech.org/` — MP3 programmer documentation, frame header reference, filter bank analysis.
- **ID3v2 Specification:** `https://id3.org/` — Official ID3v2.3 and ID3v2.4 standards.
- **Hydrogenaudio Knowledgebase:** `https://wiki.hydrogenaudio.org/` — Community-curated codec quality comparisons, listening tests, encoding guides.
- **Xiph.org:** `https://xiph.org/` — Vorbis/Opus documentation provides useful psychoacoustic background.

### Academic Resources
- **MP3 Theory PDF:** `http://www.mp3-tech.org/programmer/docs/mp3_theory.pdf` — Comprehensive academic treatment of the encoding algorithm.
- **Stanford Audio Compression Course:** `https://cs.stanford.edu/people/eroberts/courses/soco/projects/data-compression/lossy/mp3/` — Accessible explanation of hybrid filter bank and MDCT.
- **Akamai MP3 Security Research:** `https://www.akamai.com/blog/security-research/` — Detailed analysis of Huffman decoding and security implications.

---

## 18. IMPLEMENTATION CHECKLIST (for the Converter Developer)

For developers building MP3 conversion tools, the following elements must be correctly implemented:

- [ ] **Frame sync detection:** Search for `0xFF` followed by valid header fields across multiple consecutive frames
- [ ] **Header parsing:** Correctly interpret all 32 bits: version, layer, bitrate index, sample rate index, padding, channel mode, mode extension, copyright, original, emphasis
- [ ] **Frame length calculation:** `frame_size = 144 × bitrate / sample_rate + padding`
- [ ] **CRC validation:** Read and verify 16-bit CRC when protection bit = 0
- [ ] **Side info parsing:** Handle 17-byte (mono) and 32-byte (stereo) side info; parse `main_data_begin`, `scfsi`, all granule/channel fields
- [ ] **Bit reservoir:** Use `main_data_begin` as negative byte offset; store and retrieve data from reservoir buffer (max 511 bytes)
- [ ] **Scalefactor decoding:** Parse `scalefac_compress` to determine `slen1`/`slen2`; decode per-band scalefactors for long and short blocks
- [ ] **Huffman decoding:** Decode big_values region (with escape codes for values > table max), count1 region (quadruples), and rzero region (fill with zeros)
- [ ] **Requantization:** Apply `global_gain`, `scalefactors`, `preflag`, `scalefac_scale`; inverse of `pow(v, 4/3)` operation
- [ ] **Reorder short blocks:** Reorder frequency lines for block type 2 (short windows) per scalefactor band
- [ ] **Inverse MDCT:** Apply sine window, 50% overlap-add for long blocks; handle short block windows
- [ ] **Aliasing reduction:** Apply butterfly operations in each subband to reduce PQMF aliasing
- [ ] **Polyphase synthesis:** 32-band inverse filter bank, 18 iterations per granule, 2 granules per frame → 1152 output samples
- [ ] **Output formatting:** Output 16-bit signed PCM or 32-bit float as specified
- [ ] **Clipping prevention:** Detect and handle output values exceeding [-1.0, +1.0] range
- [ ] **Joint stereo decoding:** Correctly decode MS stereo (mid/side → left/right) and intensity stereo bands
- [ ] **ID3v2 reading:** Parse header, skip frames by size, handle synchsafe integers (v2.4), unsynchronization, extended header, footer
- [ ] **ID3v2 writing:** Pack frames with correct size encoding, set flags, handle text encoding bytes
- [ ] **ID3v1 reading/writing:** Handle 128-byte footer tag; ID3v1.1 track number in comment field
- [ ] **XING/VBRI TOC seeking:** Parse first frame's main data for TOC identifier; use TOC for VBR seeking
- [ ] **Gapless playback:** Report or apply encoder delay/padding trimming from LAME tag or iTunSMPB
- [ ] **ReplayGain:** Read `REPLAYGAIN_*` frames; apply during decode if requested
- [ ] **Bit-exact verification:** Run decode twice on same input and confirm bit-exact output (test for determinism)
- [ ] **Encoder identification:** Detect LAME, FhG, iTunes, and other encoders from tag signatures
- [ ] **Error handling:** Gracefully handle corrupted frames, CRC failures, malformed headers, truncated files
- [ ] **VBR file handling:** Don't assume constant frame size; recalculate for each frame
- [ ] **Mixed MPEG versions:** Handle files that switch between MPEG-1 and MPEG-2 frame headers within the same stream
