# OGG Vorbis — Deep Technical Reference

> **Category:** Lossy Audio
> **File Extensions:** .ogg, .oga
> **MIME Types:** audio/ogg, audio/vorbis, audio/x-vorbis
> **Standardization Body:** Xiph.org / IETF CELLAR Working Group
> **Specification Document:** https://xiph.org/vorbis/doc/Vorbis_I_spec.html (RFC 5215 for Ogg mapping)
> **Patent Status:** Patent-free (royalty-free, open specification)
> **License:** BSD 3-Clause / Xiph.org

---

## 1. HISTORICAL CONTEXT & ORIGIN

Ogg Vorbis was created by the Xiph.org Foundation as a free, open-source alternative to patented audio codecs like MP3 and AAC. The project began around 1993-1994 when Xiph.org founder Chris Montgomery started research into perceptual audio coding. The first public release of the Vorbis encoder came in 2000, and it quickly gained traction as a royalty-free alternative for open-source and commercial applications alike.

The name "Ogg" refers to the container format, while "Vorbis" specifically refers to the audio codec. The Xiph.org team designed both as companion technologies: Ogg as a general-purpose multimedia container and Vorbis as its primary audio codec. This bundled approach was intentional — Ogg provides robust framing, seeking, and chaining infrastructure that Vorbis relies on for streaming and seeking support.

Key milestones in Vorbis history:

- **1993-1998:** Research and prototyping phase by Chris Montgomery
- **2000:** First public beta release of libvorbis
- **2002:** Vorbis 1.0 specification finalized and frozen
- **2002-2004:** Wide adoption in open-source software (Linux distributions, games, streaming servers)
- **2005:** AoTuV beta tuning releases by Aoyumi — community-tuned builds that significantly improved quality at low bitrates
- **2007:** Xiph.Org merges with the Xiph Fund; libvorbis enters maintenance mode
- **2010:** Opus (successor codec) begins development, but Vorbis remains widely deployed
- **2012-2025:** Continued maintenance, AoTuV improvements integrated, Xiph.org IETF CELLAR working group standards efforts

The codec was specifically designed to avoid the patent thickets surrounding MP3 and AAC, making it the default audio format for many open-source operating systems (Linux, BSD), game engines (Unreal, Unity), and streaming platforms. It is the audio backbone of WebM video. The spec was frozen at version 1.0 to ensure long-term stability and decoder compatibility — no breaking changes have been introduced since the 1.0 specification.

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

Vorbis is a **lossy, perceptual audio codec** built around a transform coding architecture. Its encoding pipeline can be summarized as:

1. **PCM input:** Audio is divided into overlapping blocks (frames) of 64–8192 samples.
2. **MDCT transform:** Each block is transformed to the frequency domain via a Modified Discrete Cosine Transform.
3. **Floor estimation:** A psychoacoustic model estimates the masking threshold per spectral line.
4. **Floor curve:** A parametric curve (Type 0: Householder / Type 1: Compressed) describes the masking threshold across frequency.
5. **Residue encoding:** The spectral residue (actual MDCT values minus floor) is approximated using vector quantization with codebooks.
6. **Bitpacker:** All encoded data is assembled into a binary packet structure.

Vorbis is unique among lossy codecs in that it uses **vector quantization (VQ)** for residue encoding rather than scalar Huffman coding (MP3/AAC) or arithmetic coding (AAC-HE). This choice simplifies the decoder significantly at the cost of some compression efficiency. The encoder, however, must perform expensive search operations to find optimal VQ representations.

Key architectural characteristics:

- **Block switching:** Vorbis supports multiple block sizes (64–8192 samples) per stream, chosen adaptively based on signal characteristics. Transient signals use short blocks; tonal signals use long blocks.
- **Channel coupling:** Vorbis supports channel coupling modes (same-frequency inter-channel coupling) to exploit stereo correlation.
- **Packet chaining:** Multiple Vorbis logical bitstreams can be chained within a single Ogg physical bitstream, enabling gapless concatenation and multi-section streams.
- **VBR throughout:** Vorbis has no CBR mode. Bitrate varies frame-by-frame based on signal complexity.
- **Three-header structure:** Every logical Vorbis stream begins with three mandatory setup headers (identification, comment, setup) before any audio data.

The decoder is deliberately simple: parse headers, read codebook entries, decode floor, decode residue, combine, apply inverse MDCT, output PCM. This simplicity makes Vorbis decoding one of the fastest among perceptual codecs.

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 Ogg Container Page Structure

Every Ogg page begins with a 27-byte fixed header, followed by a variable-length segments table and payload data. Pages are the fundamental transport unit in Ogg — they can contain partial or complete Vorbis packets, and a single Vorbis packet can span multiple Ogg pages.

```
[Page Header: 27 bytes] [Segment Table: variable] [Page Payload: variable]
```

**Page Header Layout (27 bytes, all fields little-endian):**

| Byte Offset | Field Name         | Width (bytes) | Description                                                    |
|-------------|-------------------|---------------|----------------------------------------------------------------|
| 0           | capture_pattern    | 4             | Magic bytes: `O` `g` `g` `S` (0x4F 0x67 0x67 0x53)           |
| 4           | stream_structure_version | 1      | Ogg spec version; MUST be 0                                    |
| 5           | header_type_flag  | 1             | Bit 0: continuation, Bit 1: BOS (beginning of stream), Bit 2: EOS |
| 6           | granule_position  | 8             | Absolute sample position of last sample in page (signed int64) |
| 14          | serial_number     | 4             | Logical stream identifier (unique within physical bitstream)    |
| 18          | page_sequence_no  | 4             | Sequential page counter within logical stream                   |
| 22          | page_checksum     | 4             | CRC-32 (polynomial 0x04C11DB7) of entire page                  |
| 26          | page_segments     | 1             | Number of segment entries (1–255)                              |

**Header Type Flag bit meanings:**

| Bit | Value | Meaning                                                      |
|-----|-------|--------------------------------------------------------------|
| 0   | 0x01  | Continuation: page is a continuation of incomplete packet    |
| 1   | 0x02  | BOS (Beginning of Stream): this page starts a logical stream  |
| 2   | 0x04  | EOS (End of Stream): this page ends a logical stream          |

**Segment Table (1–255 entries, 1 byte each):**

Each segment entry specifies the length of one fragment of the packet data:

- If a segment entry is `< 255`: that fragment is the last segment of the current packet.
- If a segment entry is `== 255`: the packet continues in the next segment.
- A packet can span an arbitrary number of segments (up to 255 segments per page).
- A page can contain multiple complete packets, or a partial packet (continuation).

**Granule Position Encoding:**

The granule position (absolute sample number) in a Vorbis page encodes two values in a compound way:

```
granule_pos = last_decoded_sample + pre-roll samples
```

For Vorbis audio, the granule position represents the sample number of the first sample that cannot yet be decoded (the decoder needs `pre_roll` samples of look-ahead). The `pre_roll` value for Vorbis is codec-specific — it is declared in the identification header as `blocksize_1 / 2` (the encoder sets this implicitly by the block sizes chosen).

**Example page header hex dump:**

```
56 6f 67 42   00   02   00 00 00 00 00 00 80 01   12 34 56 78
[ capture ][ver][type=BOS][ granule_position=65536 ][  serial=0x12345678 ]

  00 00 01 5c   a4 f2 1c b3   02
[ page_seq=300 ][ CRC-32 ][ seg_count=2 ]

[ ff 01 ]  [ packet_data_1 (255 bytes) ][ packet_data_2 (1 byte) ]
[seg=255 ][ seg=1                  ]
```

### 3.2 Vorbis Packet Types

Vorbis packets are classified by their first byte:

| First Byte Value | Packet Type | Location |
|------------------|-------------|----------|
| 0x01             | Identification header | First packet of logical stream (BOS page) |
| 0x03             | Comment header | Second packet of logical stream |
| 0x05             | Setup header | Third packet of logical stream |
| 0x00–0x7F        | Audio data packet | Following packets (first bit = 0) |
| 0x80–0xFF        | Reserved / invalid | Should not appear in valid streams |

The first bit (MSB) of the first byte distinguishes audio packets from headers: `0` = audio data, `1` = setup/identification/comment header.

### 3.3 Identification Header (0x01 packet)

The identification header immediately follows the `0x01` packet type byte. Total minimum size: 30 bytes.

```
[0x01] [vorbis_type=0] [channels:u8] [sample_rate:u32_le] [bitrate_upper:u32_le]
       [bitrate_nominal:u32_le] [bitrate_lower:u32_le] [blocksize_0:u8] [framing_flag:u8]
```

| Field              | Type      | Offset (from 0x01) | Description                                                     |
|--------------------|-----------|--------------------|-----------------------------------------------------------------|
| vorbis_type        | uint8     | 1                  | MUST be 0 (future versions may define non-zero values)          |
| vorbis_version     | uint32    | 2                  | Audio codec version (MUST be 0 for Vorbis 1)                   |
| channels           | uint8     | 6                  | Number of audio channels (1–255, typically 1–8)                 |
| sample_rate        | uint32 LE | 7                  | Sample rate in Hz (any value 1–2^32-1)                         |
| bitrate_upper      | int32 LE  | 11                 | Upper bitrate limit (bytes/s). 0 = unknown                     |
| bitrate_nominal    | int32 LE  | 15                 | Nominal bitrate (bytes/s). 0 = unknown                         |
| bitrate_lower      | int32 LE  | 19                 | Lower bitrate limit (bytes/s). 0 = unknown                     |
| blocksize_0        | uint8     | 23                 | Smallest block size as power of 2 (e.g., 6 = 2^6 = 64 samples)|
| blocksize_1        | uint8     | 24                 | Largest block size as power of 2 (e.g., 12 = 2^12 = 4096)      |
| framing_flag       | uint8     | 25                 | MUST be 1. Bit indicating page checksum presence                |

**Blocksize rules:**
- `blocksize_0` MUST be ≤ `blocksize_1`
- Valid blocksize exponents range from 6 (64 samples) to 13 (8192 samples)
- Typical values: `0x08`/`0x0A` (256/1024) for music, `0x08`/`0x0B` (256/2048) for voice

**Example identification header hex:**

```
01 00 00 00 00   02   44 ac 00 00   ff ff ff ff   00 77 d0 00   00 00 00 00   08 0c 01
[pkt=1][ver=0    ][ch=2][rate=44100 ][upper=unk    ][nominal=128000][lower=unk    ][b0=8][b1=12][frame=1]
```

### 3.4 Comment Header (0x03 packet)

The comment header uses the Vorbis Comment format (identical to the Vorbis comment block in FLAC). See Section 7 for the full specification.

### 3.5 Setup Header (0x05 packet)

The setup header is the most complex of the three mandatory headers. It contains:

1. **Codec setup descriptor:** Number of floor types, residue types, mapping types, transform types
2. **Floor0 parameters** (if floor type 0 present)
3. **Floor1 parameters** (if floor type 1 present)
4. **Residue0 parameters** (if residue type 0 present)
5. **Residue1 parameters** (if residue type 1 present)
6. **Residue2 parameters** (if residue type 2 present)
7. **Mapping0 parameters** (if mapping type 0 present)
8. **Codebooks:** One or more codebook entries (huffman VQ)
9. **Time-domain transforms:** For block sizes requiring windowing

The setup header is parsed sequentially — each section declares its presence via a bit-field, then the decoder reads parameters for that section.

---

## 4. ENCODING ALGORITHM (DEEP DETAIL)

### 4.1 Block Size Determination

Vorbis supports block size switching on a frame-by-frame basis. The encoder selects between the declared `blocksize_0` and `blocksize_1` based on signal characteristics:

**Short blocks** (blocksize_0): Used for signals with rapid transients. A shorter window provides better time resolution but worse frequency resolution. The MDCT leakage is reduced, preserving transient attacks.

**Long blocks** (blocksize_1): Used for stationary, tonal signals. A longer window provides finer frequency resolution, enabling better psychoacoustic modeling and compression of tonal components.

**Block size decision algorithm:**
```
1. Compute spectral flux (rate of change of spectral energy) over the window
2. Compute LTP (long-term prediction) gain — how well the current block is predicted from past
3. If (high flux AND low LTP gain) → short block (transient)
4. If (low flux AND high LTP gain) → long block (tonal)
5. Handle overlapping: short blocks use 50% overlap; long blocks use 75% overlap
```

The encoder signals block size in the audio packet header: bit `0x10` in the audio packet type byte (high bit of low nibble) indicates long block (`blocksize_1`), while `0x00` indicates short block (`blocksize_0`).

### 4.2 MDCT Transform

The Modified Discrete Cosine Transform (MDCT) is the frequency-domain transform at the heart of Vorbis encoding. Unlike the DFT, the MDCT has the critical property of being **critically sampled** — no redundancy between overlapping blocks — making it ideal for perfect reconstruction systems.

**MDCT definition:**
```
X[k] = sum(n=0 to N-1) of x[n] * cos(pi/N * (n + 0.5 + N/2) * (k + 0.5))
```
for k = 0 to N/2 - 1, where N is the block size.

**Vorbis MDCT specifics:**
- N = blocksize (64–8192, always a power of 2)
- Output: N/2 spectral coefficients per channel
- Vorbis uses the **MDCT-IV** variant with a sine window
- The window function is: `sin(pi/N * (n + 0.5))`
- Overlap-add reconstruction guarantees perfect reconstruction when the same window is applied at encoder and decoder

**Windowed MDCT encoding:**
```
1. Apply window: x'[n] = x[n] * window[n]
2. Compute MDCT: get N/2 spectral coefficients
3. Encode spectral coefficients (see floor and residue sections)
```

**Inverse MDCT (decoding):**
```
1. Decode floor curve
2. Decode residue
3. Add residue to floor: spectral[i] = floor[i] + residue[i]
4. Compute inverse MDCT: get windowed time-domain samples
5. Overlap-add with previous block's second half
```

The MDCT is a linear transform, so it can be computed efficiently via FFT using standard algorithms. Vorbis uses a custom FFT-based MDCT implementation optimized for each supported block size.

### 4.3 Psychoacoustic Model and Floor Estimation

Vorbis's psychoacoustic model operates on the MDCT spectral data. The floor curve represents an approximation of the masking threshold at each spectral line — spectral energy below the floor is considered perceptually irrelevant and can be heavily quantized or zeroed.

**Floor computation steps:**

1. **Bark scale partitioning:** MDCT spectral lines are grouped into Bark-scale bands to match critical band perception.
2. **Global masking threshold:** Compute the absolute hearing threshold based on the MPEG 3 model.
3. **Tonal/anise detection:** Identify tonal components (pure sine-like spectral peaks) and noise-like components.
4. **Noise masking:** Compute how much energy each noise component can mask.
5. **Tonal masking:** Compute how much each tonal component can mask nearby bands.
6. **Simultaneous masking:** Sum all masking contributions per Bark band.
7. **Floor curve generation:** The combined masking threshold becomes the floor curve.

The floor curve is transmitted to the decoder as a compact parametric representation, not as per-spectral-line values. This is what gives Vorbis its excellent compression — a 1024-point curve is transmitted with perhaps 50–100 parameters.

### 4.4 Floor Curves

Vorbis defines two floor types. In practice, **Floor Type 1** (the "multipulse" or "compressed" floor) is used in virtually all Vorbis streams. Floor Type 0 (the older "Householder" floor) is rarely encountered in modern files.

#### Floor Type 1 (the standard floor)

Floor Type 1 uses a piecewise linear curve represented by:

1. **Partition classification:** The spectrum is divided into partitions of 2–4 Bark-scale bark-wide regions. Each partition is classified as:
   - Class 0: Low-energy, no encoding needed
   - Class 1: Low-energy, simple representation
   - Class 2+: Higher-energy, more precise representation

2. **Coefficient representation:** Each partition class uses a different number of bits to represent the curve:
   - A **vector of coefficients** per partition
   - Coefficient precision varies by class
   - Coefficients are **delta-encoded** within a partition

3. **Final curve reconstruction:** The decoder builds the floor by:
   - Reconstructing the curve values per partition from the coefficient vector
   - Interpolating between partition boundaries
   - Applying a amplitude multiplier (from the residue coupling stage)

**Floor Type 1 packet structure:**
```
[floor1_type=1: 1 bit] [partitions: 6 bits] [partition_class_list: variable]
[class_dimensions: per class, 4 bits] [class_subclasses: per class, 3 bits]
[class_masterbooks: per class, 8 bits] [pair_books: per class, 6 bits]
[coeff_list: variable]
```

#### Floor Type 0 (legacy Householder floor)

Floor Type 0 represents the floor curve using Householder reflections and a fixed codebook. It is significantly more complex and less efficient. It is included in the spec for completeness but is not used in practice.

### 4.5 Residue Encoding and Vector Quantization

After the floor curve is established, the **residue** (the difference between actual MDCT values and the floor curve) is encoded using **vector quantization (VQ)**.

**VQ basics:** The residue spectrum is partitioned into `N` vectors of `dim` spectral coefficients. A **codebook** contains candidate vectors. The encoder selects the best matching vector from the codebook for each residue partition, and the codebook index is transmitted to the decoder.

**Why VQ for residue?** The residue is typically noise-like and diffuse. Scalar Huffman coding would be inefficient for such distributed data. VQ captures the gross shape of the residue in a single lookup, at the cost of some quantization noise.

**Residue type 0 (non-coupled):**
- Each channel's residue is encoded independently
- Suitable for mono or when channels are not coupled

**Residue type 1 (residue-coupled):**
- Residues from multiple channels are encoded together as interleaved vectors
- Exploits inter-channel correlation in stereo/multi-channel

**Residue type 2 (stream-coupled):**
- All channels' residues are encoded jointly using interleaved vectors across all channels
- Maximum coupling, used for highly correlated multi-channel content

**Residue encoding pipeline:**
```
1. Partition the residue into groups of `dim` spectral coefficients
2. For each partition, select the best codebook entry (minimize squared error)
3. Encode the codebook index using a secondary VQ stage (sequence books)
4. For residue type 1/2: stages 1-3 are applied per-stage with progressive refinement
```

**Multiple residue stages:** Vorbis supports up to 3 encoding stages per residue type. The first stage uses the coarsest codebook; subsequent stages use finer codebooks to refine the approximation. This cascaded VQ approach achieves quality close to scalar quantization while maintaining the simplicity of VQ lookups.

### 4.6 Codebooks (Vector Quantization)

Codebooks are the fundamental building block of Vorbis's entropy coding. Each codebook defines a set of **codewords** (binary sequences) that map to **values** (scalar or vector numbers). Unlike Huffman coding (which uses prefix codes), Vorbis codebooks are structured VQ codebooks with specific properties.

**Codebook structure:**
```
[codebook_dimensions: u32] [codebook_entries: u32] [ordered: u1] [sparse: u1]
[length_list: u[entries]] [lookup_type: u4] [...lookup data...]
```

**Codebook properties:**
- **Dimensions:** Each codebook entry is a vector of `dimensions` scalar values (typically 2–8 for residue VQ)
- **Entries:** Number of codewords in the codebook (up to 2^32, practical limit ~16K)
- **Lengths:** Each entry has a codeword length (used for Huffman-like construction)
- **Ordered vs. sparse:** If ordered=1, lengths are stored sequentially (length 1, then length 2, etc.). If ordered=0, each length is stored individually.

**Codebook construction (for encoder):**
The encoder generates codebooks using **split-by-3 (Lloyd-Max) VQ training**:
1. Collect training vectors from many representative audio samples
2. Use generalized Lloyd iteration to find optimal cluster centers
3. Produce codebook entries (centroids)
4. Assign codewords to entries using the V皇子-Huffman construction

**Codebook lookup types:**
- Type 0: No lookup (codebook indexes directly map to values)
- Type 1: Explicitly stored values (array of values, indexed by entry)
- Type 2: Value vectors computed via sequential cascade (most common for residue)
- Type 3: Value vectors computed via parallel cascade

The most common lookup for residue encoding is **Type 2**: each codebook entry is a vector of `dimensions` values, generated by cascading: the first value is a base offset, subsequent values are cumulative deltas. This reduces the storage requirement per codebook.

**Example residue codebook entry (dim=4, type 2):**
```
codebook_entry[i] = [v_0, v_0+v_1, v_0+v_1+v_2, v_0+v_1+v_2+v_3]
```

### 4.7 Channel Coupling (Stereo)

Vorbis supports **same-frequency inter-channel coupling** (often called "channel mixing" or "mid-side"). This transforms stereo channels before encoding to exploit inter-channel redundancy.

**Coupling procedure:**
```
1. Convert L/R to M/S (mid/side):
   mid = (L + R) / 2
   side = (L - R) / 2
2. Encode mid channel normally (high precision)
3. Encode side channel with reduced precision (often heavily quantized or zeroed)
4. On decode: reconstruct L = mid + side, R = mid - side
```

The Vorbis encoder decides coupling per block based on analysis of inter-channel correlation. High correlation → strong coupling → high compression. Low correlation (e.g., highly different L/R content) → minimal coupling → preserve independence.

**Vorbis mapping structures** define:
- Which channels are coupled and in what manner
- Channel submaps (grouping of channels for floor and residue encoding)
- Phase status for each coupled channel pair

The standard Vorbis mapping (mapping 0) defines two modes:
- **Independent:** Channels encoded separately
- **Stereo:** L/R coupled into M/S as described above

### 4.8 Bitrate Management

Vorbis has no built-in bitrate target. Bitrate is a consequence of quality and signal complexity. However, encoders can enforce bitrate constraints through feedback loops:

**Quality-based encoding (reference encoder -q):**
- Encoder targets a quality level (0–10, where 10 is best)
- Bitrate is unbounded and varies frame-by-frame
- Produces files with consistent quality but variable size

**Bitrate-constrained encoding:**
- Encoder enforces upper/lower bitrate limits from identification header
- Uses rate-distortion optimization: if a frame exceeds budget, increase quantization
- May result in quality fluctuations near bitrate boundaries

The Vorbis bitstream format does not mandate any particular bitrate management strategy. Both approaches produce standard-compliant streams.

---

## 5. DECODING ALGORITHM

### 5.1 Packet Synchronization

Vorbis decoding begins with the three mandatory setup headers:

**Header 1 — Identification:**
```
1. Read byte 0: MUST be 0x01 (identification packet type)
2. Read 4-byte version: MUST be 0x00000000
3. Read channels (uint8), sample_rate (uint32 LE), bitrates (3×int32 LE)
4. Read blocksizes (2×uint8): extract exponents, verify blocksize_0 ≤ blocksize_1
5. Read framing_flag: MUST be 1
```

**Header 2 — Comment:**
```
1. Read byte 0: MUST be 0x03
2. Read vendor string length + vendor string (UTF-8)
3. Read comment list count (uint32 LE)
4. For each comment: read length + comment string
5. Validate UTF-8 encoding of all strings
```

**Header 3 — Setup:**
```
1. Read byte 0: MUST be 0x05
2. Read codec setup descriptors (bit-fields indicating which types are present)
3. For each floor type present: read floor parameters
4. For each residue type present: read residue parameters
5. For each mapping present: read mapping parameters
6. Read codebooks: for each codebook, read dimensions, entries, lengths, lookup type
7. Build Huffman decode trees from codebook lengths
8. Build VQ lookup tables from codebook entries
```

### 5.2 Audio Packet Decoding Pipeline

Once all three headers are processed, the decoder can process audio packets:

```
FOR each audio packet:
    1. Decode block size indicator → select blocksize_0 or blocksize_1
    2. Decode window type (normal or special)
    3. FOR each channel:
        a. Decode floor curve:
           - Read floor partition classes
           - For each partition, decode coefficient values
           - Reconstruct piecewise linear floor curve
        b. Decode residue:
           - Partition residue vectors
           - For each residue partition:
               i. Decode VQ index using Huffman codebook
               ii. Look up codebook entry
               iii. Add to running residue accumulator
           - Apply residue stages (multiple passes if configured)
        c. Combine floor + residue:
           - residue[i] = residue[i] + floor[i]  (per spectral line)
        d. Decode side info (if coupled):
           - Decode M/S components separately
           - Convert back: L = mid + side, R = mid - side
    4. Apply inverse MDCT:
       - For each channel, compute inverse MDCT of (floor + residue)
    5. Apply window function:
       - Multiply by sine window
    6. Overlap-add:
       - Add current block's second half to previous block's first half (50% overlap)
    7. Output PCM samples
```

### 5.3 Inverse MDCT

The inverse MDCT reconstructs time-domain samples from spectral coefficients:

```
x[n] = (2/N) * sum(k=0 to N/2-1) of X[k] * cos(pi/N * (n + N/2) * (k + 0.5))
```

**Implementation via FFT:**
The MDCT can be computed via FFT using standard algorithms:
1. Pre-twiddle: multiply input by window and sine-cosine factors
2. Compute FFT of length N
3. Post-twiddle: extract real parts and apply inverse trigonometric factors
4. The total complexity is O(N log N)

**Overlap-add:**
Vorbis uses 50% overlap between consecutive blocks. Each block is windowed before MDCT, and the inverse MDCT produces windowed time-domain samples. Adjacent blocks overlap by 50%:
```
output[n] = block_k[n] + block_{k+1}[n - N/2]
```
This guarantees perfect reconstruction (PR) when using properly matched analysis/synthesis windows.

### 5.4 Window Function and Overlap-Add

Vorbis uses a **sine window** (also called the Vorbis window):

```
window[n] = sin(pi/N * (n + 0.5))  for n = 0 to N-1
```

**Overlap-add reconstruction:**
```
FOR n = 0 to N/2 - 1:
    output[n] = prev_windowed[n] + curr_windowed[n]

prev_windowed = next_windowed shifted by N/2
```

This 50% overlap ensures that the sine windows form a perfect reconstruction pair:
```
window[n]^2 + window[n + N/2]^2 = 1
```

The result is a critically sampled, perfect-reconstruction, lapped transform. Every time-domain sample appears exactly once in the output (after overlap-add), and the reconstruction is guaranteed to be exact (barring quantization error).

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Ogg Container

The Ogg container is the **primary and canonical container** for Vorbis. An Ogg/Vorbis file is a physical bitstream of Ogg pages, each containing Vorbis packets (or partial packets).

**Ogg/Vorbis file structure:**
```
[Page 1: BOS, Identification Header] [Page 2: Comment Header]
[Page 3: Setup Header] [Page 4+: Audio Data Pages] ... [Page N: EOS]
```

**Seeking in Ogg/Vorbis:**
- Seeking relies on the **granule position** in Ogg pages and a **seek index** table (optional but recommended)
- The seek index maps time positions to byte offsets via a seek table in the comment header area
- Without a seek table, seeking is performed via bisection and CRC validation
- The granule position encodes the sample number, enabling time-accurate seeking

**Packet chaining:**
Ogg supports chaining of logical bitstreams. A chained bitstream:
- Each logical stream starts with a BOS page (identification header)
- A new BOS page implicitly terminates the previous stream
- Allows concatenation of multiple audio sections without transcoding
- Essential for gapless playback and multi-track streams

### 6.2 Other Containers Supporting Vorbis

| Container    | Method                      | Notes                                             |
|--------------|-----------------------------|---------------------------------------------------|
| Ogg (native) | Native Ogg page framing     | Reference, most efficient                         |
| Matroska/MKA | CodecPrivate block          | Vorbis headers stored in CodecPrivate             |
| WebM         | Same as Matroska            | Audio track codec ID: "A_VORBIS"                  |
| MP4/M4A      | Not natively supported      | Would require transcoding to AAC/ALAC            |
| FLAC         | Not applicable              | FLAC has its own codec; use native FLAC           |
| AVI          | Supported via FourCC "vorb" | Non-standard, limited support                     |
| WAV          | Not applicable              | WAV is PCM-only; use external encoder            |

**Matroska/MKA mapping (RFC 6381):**
- Codec ID: `A_VORBIS`
- CodecPrivate: concatenation of the three Vorbis headers (identification + comment + setup)
- Clusters: each Matroska cluster contains Ogg pages or Vorbis packets
- Timecode: derived from sample_rate and granule position

---

## 7. METADATA ARCHITECTURE

### 7.1 Vorbis Comment Format

Vorbis Comments (also known as "Vorbis comment headers" or "Vorbis tags") are the standard metadata format for Vorbis and many other Xiph.org codecs (FLAC, Opus, Speex, Ogg Theora, Ogg VP8/VP9).

**Structure (all integer fields are little-endian uint32):**
```
[Vendor String Length: u32] [Vendor String: UTF-8 bytes]
[Comment List Length: u32]
FOR each comment:
    [Comment Length: u32] [Comment String: UTF-8 bytes "KEY=value"]
```

**Comment string encoding:**
- Format: `KEY=value` where KEY is ASCII uppercase letters, numbers, and underscores only
- Values are UTF-8 encoded text
- There is no maximum field length (except practical limits of the bitstream)
- Keys are case-insensitive for parsing but conventionally uppercase first letter + uppercase after underscores

**Field name rules (strict):**
- Characters allowed in field names: A-Z, 0-9, underscore
- No spaces, no special characters, no lowercase in the key portion
- The `=` character separates key from value
- Multiple values for the same key are allowed (repeated tags)

**Recommended tag fields:**

| Field Name            | Description                                    | Multi-value |
|-----------------------|------------------------------------------------|-------------|
| TITLE                 | Track title                                    | No          |
| ARTIST                | Primary artist                                 | No          |
| ALBUM                 | Album name                                     | No          |
| ALBUMARTIST           | Album-level artist                             | No          |
| COMPOSER              | Composer                                       | Yes         |
| GENRE                 | Genre classification                           | Yes         |
| DATE                  | Release date                                   | No          |
| TRACKNUMBER           | Track number within album                      | No          |
| TRACKTOTAL            | Total tracks in album                          | No          |
| DISCNUMBER            | Disc number                                    | No          |
| DISCTOTAL             | Total discs                                    | No          |
| COMMENT               | User comment                                   | Yes         |
| LYRICS                | Lyrics text                                    | Yes         |
| BPM                   | Beats per minute                               | No          |
| CATALOGNUMBER         | Label catalog number                           | Yes         |
| BARCODE               | UPC/EAN barcode                                | No          |
| ISRC                  | International Standard Recording Code           | Yes         |
| LABEL                 | Record label                                   | Yes         |
| ENCODER               | Encoding software/version                      | No          |
| REPLAYGAIN_TRACK_GAIN | ReplayGain track gain adjustment (dB)          | No          |
| REPLAYGAIN_TRACK_PEAK | ReplayGain track peak amplitude                | No          |
| REPLAYGAIN_ALBUM_GAIN | ReplayGain album gain adjustment (dB)          | No          |
| REPLAYGAIN_ALBUM_PEAK | ReplayGain album peak amplitude                | No          |
| MUSIC_BRAINZ_TRACKID  | MusicBrainz track MBID                         | No          |
| MUSIC_BRAINZ_ALBUMID  | MusicBrainz album MBID                         | No          |
| ARTISTSORT            | Sort key for artist                            | No          |
| TITLESORT             | Sort key for title                             | No          |
| ALBUMARTISTSORT       | Sort key for album artist                      | No          |
| SOURCE                | Source medium                                  | Yes         |

### 7.2 Binary Metadata

Vorbis does not natively support binary metadata within the Vorbis comment format. For embedded cover art in Ogg Vorbis:

**Cover art via Ogg comment header:**
Vorbis comments cannot directly embed binary picture data. Cover art in Ogg Vorbis is typically stored as an attachment in the Ogg container (not within the Vorbis bitstream itself). Alternative approaches:

1. **Xiph comment extension:** A vendor string that indicates a cover art MIME type/URL follows (not standardized)
2. **Cover art in metadata:COMMENTS:** A vendor string starting with "coverart=" followed by base64-encoded binary data (non-standard extension used by some tools)
3. **External files:** Store cover art as a separate file alongside the Ogg file

The standard approach for cover art in Ogg Vorbis is to embed it within the comment header using a vendor string extension:
```
VENDOR=reference libVorbis I 20060307; coverart=image/jpeg,<base64>
```
This is a de facto convention, not part of the official Vorbis spec.

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 Codec Identifiers

- **FFmpeg format name:** `ogg` (for Ogg container)
- **Vorbis codec names:** `vorbis` (native FFmpeg decoder/encoder)
- **Codec ID:** `AV_CODEC_ID_VORBIS`
- **Encoder name:** `libvorbis` (FFmpeg's libvorbis wrapper)

FFmpeg contains two Vorbis implementations:
1. **Native (native):** Written from scratch for FFmpeg, not bit-exact compatible with libvorbis
2. **libvorbis:** Wrapper around Xiph.org's reference libvorbis library

**Important:** The native FFmpeg encoder produces files that are **not bit-exact** with libvorbis encoder output. For maximum compatibility, use `libvorbis` encoder (requires libvorbis to be installed).

### 8.2 FFmpeg Encoding — Full CLI Reference

```bash
# Basic Vorbis encoding (native, default quality)
ffmpeg -i input.wav -c:a vorbis output.ogg

# High quality using libvorbis (recommended)
ffmpeg -i input.wav -c:a libvorbis output.ogg

# Quality-based encoding with libvorbis (-q:a 0-10, default ~3)
ffmpeg -i input.wav -c:a libvorbis -q:a 6 output.ogg   # very high quality
ffmpeg -i input.wav -c:a libvorbis -q:a 4 output.ogg   # good quality (default ~3)
ffmpeg -i input.wav -c:a libvorbis -q:a 2 output.ogg   # lower quality, smaller

# Bitrate-based encoding (approximate target)
ffmpeg -i input.wav -c:a libvorbis -b:a 192k output.ogg

# Variable bitrate with upper limit
ffmpeg -i input.wav -c:a libvorbis -vbr 4 output.ogg  # 4=ABR ~192k

# Force two-pass encoding (not natively supported by libvorbis in ffmpeg)
# Use oggenc for two-pass: https://libmpg123.org/oggenc.1.html
oggenc -q 6 --managed -b 192 input.wav -o output.ogg

# Specify audio channels
ffmpeg -i input.wav -c:a libvorbis -ac 2 output.ogg   # stereo
ffmpeg -i input.wav -c:a libvorbis -ac 1 output.ogg   # mono

# Specify sample rate
ffmpeg -i input.wav -c:a libvorbis -ar 48000 output.ogg

# Preserve metadata (Vorbis comments)
ffmpeg -i input.wav -c:a libvorbis -map_metadata 0 output.ogg

# Strip all metadata
ffmpeg -i input.wav -c:a libvorbis -map_metadata -1 output.ogg

# Encode from FLAC
ffmpeg -i input.flac -c:a libvorbis -q:a 6 output.ogg

# Encode with custom vendor string
ffmpeg -i input.wav -c:a libvorbis -vendor "MyEncoder/1.0" output.ogg
```

**libvorbis encoder options:**

| Option            | Type   | Default     | Description                              |
|-------------------|--------|-------------|------------------------------------------|
| -q:a              | float  | 3.0         | Quality (0–10)                           |
| -b:a              | int    | N/A         | Bitrate in bits/s (alternative to -q)   |
| -vbr              | int    | 2 (CVBR)   | VBR mode: 0=CBR, 1=ABR, 2=CVBR, 3=low   |
| -mg               | int    | 0           | Managed bitrate (1=unrestrained VBR)     |
| -maxrate          | int    | 0           | Maximum bitrate (for streaming)          |
| -minrate          | int    | 0           | Minimum bitrate                          |
| -bufsize          | int    | N/A         | Decoder buffer size                      |
| -ac               | int    | auto        | Number of output channels                |
| -ar               | int    | auto        | Sample rate                              |
| -vendor           | string | "Lavf57.xx" | Vendor string                            |

### 8.3 FFmpeg Decoding

```bash
# Decode to WAV
ffmpeg -i input.ogg output.wav

# Decode to specific format
ffmpeg -i input.ogg -c:a pcm_s16le output.wav     # 16-bit signed LE PCM
ffmpeg -i input.ogg -c:a pcm_s24le output.wav     # 24-bit signed LE PCM
ffmpeg -i input.ogg -c:a pcm_f32le output.wav     # 32-bit float PCM

# Decode with channel layout
ffmpeg -i input.ogg -channel_layout stereo output.wav

# Stream info (no decode)
ffprobe -v quiet -show_format -show_streams input.ogg

# Detailed stream info
ffprobe -v verbose -show_format -show_streams input.ogg

# Extract duration
ffprobe -v quiet -show_entries format=duration -of default=nokey=1:noprint_wrappers=1 input.ogg

# Decode specific time range
ffmpeg -i input.ogg -ss 00:01:00 -t 00:00:30 output.wav

# Decode and verify with ffmd5
ffmpeg -i input.ogg -f md5 - 2>/dev/null
```

### 8.4 FFmpeg Metadata Handling

**Reading metadata:**
```bash
# List all Vorbis comments
ffprobe -v quiet -show_format input.ogg

# Read specific tag
ffprobe -v quiet -show_entries format_tags=title,artist,album -of json input.ogg
```

**Writing metadata:**
```bash
# Add/replace tags
ffmpeg -i input.wav -c:a libvorbis -q:a 6 \
       -metadata title="Track Title" \
       -metadata artist="Artist" \
       -metadata album="Album" \
       output.ogg

# Copy metadata from source during transcode
ffmpeg -i input.ogg -c:a libvorbis -q:a 6 -map_metadata 0 output.ogg

# Clear all metadata
ffmpeg -i input.ogg -c:a libvorbis -q:a 6 -map_metadata -1 output.ogg

# Extract cover art from Ogg (workaround — stored as attachment)
ffmpeg -i input.ogg -attach cover.jpg -metadata:s:t mimetype=image/jpeg output2.ogg
```

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seeking Architecture

Ogg/Vorbis seeking relies on two mechanisms:

**1. Granule position indexing:**
- Each Ogg page's granule position encodes the sample number
- Converting sample number to time: `time = sample_number / sample_rate`
- The decoder maintains a mapping from byte offsets to granule positions

**2. Seek table (index):**
- Optional index stored in the comment header (non-standard extension)
- Maps time positions to byte offsets for fast seeking
- Not all Ogg/Vorbis files contain a seek table
- Without seek table, seeking uses bisection: binary search + CRC validation

**Seeking algorithm:**
```
1. If seek table present: binary search in seek table for nearest page ≤ target time
2. If no seek table: bisection search
   a. Start at byte 0 and end of file
   b. Bisect: check page at midpoint
   c. Read granule position of midpoint page
   d. If granule > target: move end backward
   e. If granule < target: move start forward
   f. Repeat until within tolerance
3. Decode forward from seek point to exact target sample
4. Handle block overlap: account for MDCT look-ahead (pre-roll samples)
```

**Pre-roll compensation:**
Vorbis uses overlapping blocks. The granule position marks the first sample not yet decodable. To seek to a specific sample:
```
actual_output_sample = seek_target - pre_roll_samples
pre_roll = blocksize_1 / 2
```

### 9.2 Gapless Playback

Gapless playback with Ogg/Vorbis is achieved through:

1. **Packet chaining:** Multiple logical bitstreams can be chained within a single Ogg stream
2. **Granule position tracking:** Exact sample positions are preserved across page boundaries
3. **Pre-roll handling:** Decoder outputs silence for pre-roll samples at the start of the first block
4. **Encoder delay tagging:** The Vorbis encoder embeds delay information in the comment header

For true gapless concatenation, the encoder must ensure that the first audio sample has granule position 0, and the encoder delay (samples before first MDCT block) is compensated.

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

**Latency:**
- Minimum block size of 64 samples → ~1.5ms at 44.1kHz (short block)
- Typical block size of 1024 samples → ~23ms at 44.1kHz (long block)
- Total codec latency ≈ blocksize + pre-roll (blocksize/2) ≈ 1.5× blocksize
- Real-world latency: 30–50ms typical, making it unsuitable for real-time two-way communication

**Streaming architecture:**
- Ogg pages are self-contained with CRC validation
- Each page can be decoded independently (within the context of the bitstream headers)
- HTTP streaming works naturally: Range requests fetch individual pages
- Icecast/Shoutcast servers commonly use Ogg/Vorbis for open-source streaming

**Real-time encoding:**
- libvorbis encoding: 10–30x real-time at 44.1kHz stereo at quality 6
- opusenc-based encoding (Opus, not Vorbis): faster
- Decoding: 80–150x real-time at 44.1kHz stereo
- Memory usage: modest (50–100 MB for encoding a typical album)

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Channel Layout Support

Vorbis supports up to **255 channels** in theory. In practice:
- 1 channel (mono): `channel_mapping = 0`
- 2 channels (stereo): `channel_mapping = 0` (or coupling mapping)
- 5.1 channels (6 channels): `channel_mapping = 1` (defined in Vorbis I spec)
- Higher channel counts: custom mapping with explicit channel coupling

**Standard Vorbis channel orderings (mapping 1):**

| Channels | Ordering                          |
|----------|-----------------------------------|
| 1        | Mono (C)                          |
| 2        | Left (L), Right (R)               |
| 3        | Left (L), Center (C), Right (R)   |
| 4        | Left (L), Center (C), Right (R), Back Center (Bc) |
| 5        | Left (L), Center (C), Right (R), Back Left (Bl), Back Right (Br) |
| 6        | Left (L), Center (C), Right (R), Back Left (Bl), Back Right (Br), LFE |
| 7        | Not standard (requires custom mapping) |
| 8+       | Custom (Vorbis II channel mapping) |

**Note:** Vorbis uses a unique channel ordering that differs from both DVD-Audio (SN48) and WAV/DTS (SDSS). For maximum compatibility, check the container's channel layout metadata (e.g., Matroska's AudioTrack with ChannelLayout element).

### 11.2 Channel Coupling in Multi-Channel

The same mid-side coupling technique applies to multi-channel audio. Vorbis can couple adjacent channel pairs (L/R, SL/SR, etc.) using the residue coupling mechanism defined in the mapping structure.

For 5.1 content, the encoder typically:
- Couples L/R into M/S (primary stereo image)
- Couples SL/SR into a secondary M/S pair
- Leaves center and LFE uncoupled (critical for dialogue and bass)

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

### 12.1 Sample Rate and Bit Depth

Vorbis uses **32-bit floating-point** internal representation and **integer PCM** for input/output. The specification places no hard upper limit on sample rate (the identification header's uint32 field supports up to 4.29 GHz). In practice:

- **Sample rates observed:** 8000, 12000, 16000, 22050, 24000, 32000, 44100, 48000, 72000, 96000, 176400, 192000 Hz
- **Bit depths:** Input/output as 16-bit or 24-bit integer PCM. Internal precision is 32-bit float.
- **High-resolution (96kHz+):** Fully supported. libvorbis handles rates up to 2^32-1 Hz.

**Encoding high-resolution:**
```bash
# 96kHz, 24-bit, very high quality
ffmpeg -i input_96k.wav -c:a libvorbis -q:a 8 -ar 96000 output.ogg

# Verify properties
ffprobe -v quiet -show_streams input_96k.wav | grep -E "sample_rate|channels"
```

### 12.2 Wide Dynamic Range

Vorbis's floating-point internal processing naturally handles wide dynamic ranges. The psychoacoustic model accounts for both very quiet and very loud passages. For high-dynamic-range content (classical, jazz):

- Use longer block sizes (4096 or 8192) to capture low-frequency sustained tones
- Use quality ≥ 6 to ensure sufficient bitrate for complex passages
- Consider using `-q:a 10` for archival applications

---

## 13. COMPRESSION RATIOS & BENCHMARKS

### 13.1 Typical Compression Ratios

| Content Type    | Quality Level | Typical Ratio (vs WAV) | Bitrate (44.1kHz stereo) |
|-----------------|---------------|------------------------|--------------------------|
| Speech/podcast  | q=2           | 10–12×                 | 48–64 kbps              |
| Speech/podcast  | q=4           | 8–10×                  | 80–96 kbps              |
| Pop/rock        | q=4           | 10–12×                 | 112–128 kbps            |
| Pop/rock        | q=6           | 8–10×                  | 160–192 kbps            |
| Classical       | q=6           | 7–9×                   | 192–256 kbps            |
| Classical       | q=8           | 6–8×                   | 256–320 kbps            |
| Hi-res (96kHz)  | q=8           | 6–8×                   | 384–512 kbps            |

**Transparency threshold:** At q=6 (128–160 kbps stereo), Vorbis is perceptually transparent for most listeners and program material. The AoTuV-tuned encoders achieve transparency at lower bitrates (80–100 kbps) through improved psychoacoustic tuning.

**Comparison with other codecs:**
- At equivalent quality, Vorbis bitrate is comparable to LAME MP3 VBR (quality 2–4, ~130–180 kbps for music)
- Vorbis generally achieves 10–15% smaller files than MP3 at equivalent quality
- AAC (HE-AAC) achieves better quality at very low bitrates (<64 kbps)
- Opus outperforms Vorbis at all bitrates, especially below 128 kbps

### 13.2 Encoding/Decoding Speed

| Quality | Encode Speed (×RT @ 44.1kHz stereo) | Decode Speed (×RT) |
|---------|-----------------------------------|-------------------|
| q=2     | 30–50x                            | 80–120x           |
| q=4     | 20–40x                            | 80–120x           |
| q=6     | 15–30x                            | 80–120x           |
| q=8     | 10–20x                            | 80–120x           |
| q=10    | 5–15x                             | 80–120x           |

**Notes:**
- RT = Real-Time (1x = playback speed)
- Decode speed is essentially constant regardless of quality level
- High sample rates (96kHz+) reduce effective speed proportionally
- AoTuV encoders are slightly slower than stock libvorbis due to additional analysis passes

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 Encoder Compatibility Issues

**Native FFmpeg encoder vs. libvorbis:**
- FFmpeg's native Vorbis encoder produces bitstreams that may not decode correctly on all players
- Some hardware players (especially older ones) only accept libvorbis-encoded streams
- **Recommendation:** Always use `libvorbis` encoder for production files

**AoTuV tuning incompatibilities:**
- AoTuV is a third-party tuning of libvorbis that changes the psychoacoustic model
- AoTuV-encoded files are fully compliant Vorbis but may produce different bitstreams than stock libvorbis
- Some decoders may exhibit slightly different output (audible differences are rare)

### 14.2 Metadata Edge Cases

**Invalid UTF-8 in comments:**
- Some tools write non-UTF-8 data in Vorbis comment fields
- The Vorbis spec mandates UTF-8, but decoders often accept ISO-8859-1 or binary
- FFmpeg's vorbis parser is strict; non-UTF-8 fields may cause parsing failures

**Vendor string parsing:**
- Some encoders write extremely long vendor strings
- FFmpeg and other tools may truncate or reject long vendor strings
- Safe vendor string length: <128 bytes

### 14.3 Seeking Edge Cases

**Pre-roll handling at seek boundaries:**
- After seeking, the decoder outputs some pre-roll samples that should be discarded
- Pre-roll length = blocksize_1 / 2
- Most players handle this correctly, but custom implementations must account for it

**Short blocks at stream boundaries:**
- The first few frames of a Vorbis stream may use short blocks
- Granule position encoding at the start of the stream can be confusing
- Ensure decoder handles granule position = 0 correctly

### 14.4 Invalid Bitstream Handling

**Zero-length packets:**
- A Vorbis packet of zero length should be skipped
- This can occur at the end of an Ogg page with padding

**Invalid codebook entries:**
- Some malformed files contain invalid codebook configurations
- Decoder must validate: codebook_dimensions ≥ 1, codebook_entries ≥ 1
- Codebook length list must sum to exactly `ceil(log2(codebook_entries))` bits

**Floor curve extremes:**
- Very low floors (near-zero amplitude): can cause division-by-zero in inverse floor computation
- Very high floors (near-maximum amplitude): residue becomes zero, effectively lossless encoding
- Validate floor values before using as divisors in residue reconstruction

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM Vorbis

| Target Format   | Recommended Command                                   | Notes                                    |
|-----------------|------------------------------------------------------|------------------------------------------|
| WAV (16-bit)    | `ffmpeg -i input.ogg output.wav`                    | Bit-exact if using same sample rate      |
| WAV (24-bit)    | `ffmpeg -i input.ogg -acodec pcm_s24le output.wav` | Upscale from internal 32-bit float       |
| FLAC            | `ffmpeg -i input.ogg -c:a flac output.flac`         | Lossless, slightly larger than Ogg       |
| MP3 (320kbps)   | `ffmpeg -i input.ogg -b:a 320k output.mp3`          | Lossy transcode, quality loss            |
| AAC (256kbps)   | `ffmpeg -i input.ogg -b:a 256k output.aac`          | Lossy, good for Apple ecosystem          |
| Opus            | `ffmpeg -i input.ogg -c:a libopus -b:a 192k output.opus` | Generally better than Vorbis at same bitrate |
| ALAC            | `ffmpeg -i input.ogg -c:a alac output.m4a`          | Lossless in MP4 container                |

### 15.2 Converting TO Vorbis

| Source Format   | Recommended Command                                   | Quality Notes                            |
|-----------------|------------------------------------------------------|------------------------------------------|
| WAV (CD-DA)     | `ffmpeg -i input.wav -c:a libvorbis -q:a 6 output.ogg` | Transparent at q=6                      |
| WAV (96kHz)     | `ffmpeg -i input.wav -c:a libvorbis -q:a 8 -ar 96000 output.ogg` | Hi-res capable                           |
| FLAC            | `ffmpeg -i input.flac -c:a libvorbis -q:a 6 output.ogg` | Lossy re-encode of lossless             |
| MP3             | `ffmpeg -i input.mp3 -c:a libvorbis -q:a 6 output.ogg` | Double-lossy; avoid                      |
| AAC             | `ffmpeg -i input.m4a -c:a libvorbis -q:a 6 output.ogg` | Lossy re-encode; quality loss expected  |
| AIFF            | `ffmpeg -i input.aiff -c:a libvorbis -q:a 6 output.ogg` | Direct conversion                        |

### 15.3 Metadata Preservation

```bash
# Read metadata from source Vorbis
ffprobe -v quiet -show_format input.ogg

# Preserve all Vorbis comments during conversion
ffmpeg -i input.ogg -c:a flac -map_metadata 0 output.flac

# Convert and preserve all tags (including replaygain)
ffmpeg -i input.ogg -c:a libvorbis -q:a 6 \
       -metadata title="..." -metadata artist="..." output.ogg

# Verify round-trip (decode back and compare)
ffmpeg -i input.ogg -f md5 - 2>/dev/null
ffmpeg -i output.ogg -f md5 - 2>/dev/null
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library          | Language | License    | Quality | Notes                                       |
|------------------|----------|------------|---------|---------------------------------------------|
| libvorbis        | C        | BSD        | Reference | Official Xiph.org implementation            |
| libvorbisfile    | C        | BSD        | Reference | File/streaming decoding API                |
| libvorbisenc     | C        | BSD        | Reference | Encoding API                               |
| FFmpeg (native)  | C        | LGPL       | High     | Complete but not libvorbis-compatible      |
| FFmpeg libvorbis | C        | LGPL       | Reference | libvorbis wrapper                          |
| AoTuV            | C        | BSD        | High     | Community-tuned libvorbis (better low-bitrate) |
| Tizen VPB        | C        | Proprietary| High     | Samsung's hardware Vorbis decoder           |
| tremor           | C        | BSD        | Medium   | Fixed-point decoder (no float)              |

**libvorbis API reference:**

```c
// Initialization
vorbis_info vi;
vorbis_comment vc;
vorbis_dsp_state vd;
vorbis_block vb;

vorbis_info_init(&vi);
vorbis_encode_init_vbr(&vi, channels, sample_rate, 0.4); // q=6 quality
vorbis_comment_init(&vc);
vorbis_comment_add_tag(&vc, "ENCODER", "MyEncoder/1.0");

// Encoding loop
ogg_packet op;
while (get_pcm_samples(&pcm, &samples)) {
    vorbis_analysis_init(&vd, &vi);
    vorbis_block_init(&vd, &vb);
    float **buffer = vorbis_analysis_buffer(&vd, samples);
    // copy PCM into buffer
    vorbis_analysis(&vb, NULL);
    vorbis_bitrate_addblock(&vb, &op);
    // write op.packet to Ogg page
}
```

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### 17.1 Official Specifications

- **Vorbis I Specification:** https://xiph.org/vorbis/doc/Vorbis_I_spec.html — The canonical bitstream specification
- **Ogg Logical Bitstream Framing:** https://xiph.orgogg/doc/oggstream.html — Ogg container specification
- **Vorbis Comment Specification:** https://xiph.org/vorbis/doc/v-comment.html — Vorbis comment header format
- **RFC 5215:** "Ogg Vorbis Audio Codec" — IETF standard (informational)
- **Xiph.org Vorbis Homepage:** https://xiph.org/vorbis/
- **libvorbis source code:** https://github.com/xiph/libvorbis

### 17.2 Reference Implementations

- **libvorbis:** https://git.xiph.org/?p=vorbis.git — Reference encoder/decoder
- **AoTuV:** https://www.geocities.jp/aoyoume/vorbis/ — Community-tuned encoder improvements
- **FFmpeg:** https://ffmpeg.org/ — Native Vorbis decoder/encoder

### 17.3 Additional Resources

- **Xiph.org wiki:** https://wiki.xiph.org/Vorbis — Community documentation
- **Hydrogenaudio Knowledgebase:** https://wiki.hydrogenaud.io/ — Detailed codec comparisons and tests
- **Vorbis quality comparisons:** Hydrogenaudio blind tests comparing Vorbis to MP3 and AAC

---

## 18. BINARY FORMAT EXAMPLES

### 18.1 Complete Ogg Page Dissection

Consider an Ogg page containing the Vorbis identification header:

```
4f 67 67 53   00   02   00 00 00 00 00 00 80 01   12 34 56 78   00 00 00 01   d5 a3 f4 12   02   ff ff 01
[capture    ][ver][BOS][ granule=65536         ][ serial=0x12345678 ][ page_seq=1 ][  CRC-32   ][N=2][seg=255][seg=1]
```

**Field-by-field:**
| Offset | Field            | Value       | Description                           |
|--------|------------------|-------------|---------------------------------------|
| 0–3    | capture_pattern  | 0x4F676753  | "OggS" magic                          |
| 4      | version          | 0x00        | Ogg version 0                         |
| 5      | header_type      | 0x02        | BOS (Beginning of Stream)             |
| 6–13   | granule_position | 0x0001800000000000 → 65536 | Last sample number   |
| 14–17  | serial_number    | 0x78563412  | Stream ID                             |
| 18–21  | page_sequence    | 0x01000000  | Page #1                               |
| 22–25  | page_checksum    | 0x12F4A3D5  | CRC-32                                |
| 26     | page_segments    | 0x02        | 2 segments                            |
| 27     | segment_table[0] | 0xFF        | 255 bytes, packet continues           |
| 28     | segment_table[1] | 0x01        | 1 byte, end of packet                 |
| 29     | payload          | (256 bytes) | Identification header data            |

### 18.2 Identification Header Dissection

For the identification header from the page above:

```
01 00 00 00 00   02   44 ac 00 00   00 77 d0 00   00 77 d0 00   00 00 00 00   08 0c 01
[pkt=1][ver=0    ][ch=2][rate=44100 ][upper=~128k][nom=128k   ][lower=0     ][b0=8][b1=12][frame=1]
```

**Decoded values:**
- Packet type: 0x01 (identification header)
- Vorbis version: 0 (version 0)
- Channels: 2 (stereo)
- Sample rate: 44100 Hz
- Bitrate upper: 128000 bytes/s
- Bitrate nominal: 128000 bytes/s
- Bitrate lower: 0 (unknown)
- Block size 0: 2^8 = 256 samples
- Block size 1: 2^12 = 4096 samples
- Framing flag: 1 (checksums present)

### 18.3 Audio Packet Structure

A Vorbis audio packet begins with:

```
[0x00-0xFF] — first byte: packet type (audio = 0, headers = 1/3/5)
```

For a stereo audio frame using long blocks:

```
Byte 0: 0x08  (0b00001000)
         └─ Block flag: bit 4 set = long block (blocksize_1)
         └─ Packet type: 0 = audio data

Bytes 1–N: Encoded audio data (floor, residue, coupling info)
```

**Packet decoding at byte level:**
```
1. Read packet_type (1 byte): determines block size
2. Read floor data: partition class list + coefficient values
3. Read residue data: VQ indexes + stages
4. Read coupling data: M/S mapping (if coupled)
5. Remaining bits: padding to byte boundary
```

### 18.4 Setup Header Codebook Example

A minimal Vorbis codebook in the setup header:

```
[codebook_dimensions=4: 4 bytes] [codebook_entries=256: 4 bytes]
[ordered=0: 1 bit] [sparse=1: 1 bit] [lengths: 256 bits]
[lookup_type=2: 4 bits] [quant_min=-32768: 4 bytes]
[quant_delta=1: 4 bytes] [quant=16: 4 bits] [seq_p=0: 1 bit]
[lookup_values=65536: 4 bytes] [cascade: variable]
```

**Decoding:**
```
1. Build Huffman tree from length list
2. Allocate VQ entry array of 256 × 4 values
3. Read cascade values (quantized deltas)
4. Compute VQ entries: entry[i] = v_0 + v_1 + v_2 + v_3 (cumulative)
5. Use entries to decode residue vectors
```

---

## 19. ADVANCED ALGORITHMIC DETAIL

### 19.1 The MDCT Window and Perfect Reconstruction

Vorbis uses a critically sampled, lapped transform. Perfect reconstruction requires that the analysis and synthesis windows satisfy the **Nuttall condition**:

```
w[n]^2 + w[n + N/2]^2 = 1  for n = 0 to N/2 - 1
```

The Vorbis sine window satisfies this exactly:

```
w[n] = sin(pi/N * (n + 0.5))
w[n + N/2]^2 = sin^2(pi/N * (n + N/2 + 0.5))
             = sin^2(pi - pi/N * (n + 0.5))
             = sin^2(pi/N * (n + 0.5))
w[n]^2 + w[n+N/2]^2 = sin^2(x) + sin^2(pi/2 - x) = sin^2(x) + cos^2(x) = 1
```

This guarantees that overlap-add reconstruction produces the original signal exactly (ignoring quantization).

### 19.2 Residue Cascade Encoding Detail

Vorbis residue encoding uses multiple cascaded VQ stages. Each stage refines the approximation from previous stages:

```
Stage 0: R_approx = codebook_0.best_match(residue)
Stage 1: R_approx += codebook_1.best_match(residue - R_approx)
Stage 2: R_approx += codebook_2.best_match(residue - R_approx)
```

This cascade approach is analogous to **successive refinement** or **layered VQ**. It allows the encoder to use:
- A coarse codebook for the bulk of the energy (stage 0)
- A medium-precision codebook for the medium-scale structure (stage 1)
- A fine codebook for the remaining details (stage 2)

The number of cascade stages and which codebooks are used at each stage is configured in the setup header.

### 19.3 Floor Reconstruction Algorithm

For Floor Type 1 (the standard floor), reconstruction proceeds as:

```
1. For each partition:
   a. Read partition class
   b. Look up class dimensions and sub-class book
   c. Read coefficient values using the sub-class VQ book
   d. Compute partition value from coefficients
2. For each spectral line in partition:
   a. Interpolate between partition endpoints
   b. Apply any channel coupling phase
   c. Result: floor_value[line]
3. Final floor curve: floor_value[i] for all i
```

The interpolation between partitions uses linear interpolation with Bark-scale weighting, ensuring that the floor curve is smooth within each critical band but can jump between bands.

---

## 20. IMPLEMENTATION CHECKLIST (for the Converter Developer)

### Encoding Pipeline

- [ ] Input validation: PCM sample rate, bit depth, channel count
- [ ] Use `libvorbis` encoder (not native FFmpeg) for compatibility
- [ ] Set quality target: `-q:a 6` for transparent, `-q:a 8` for archival
- [ ] Block size selection: verify `blocksize_0 ≤ blocksize_1`, powers of 2 in range [64, 8192]
- [ ] Channel coupling decision: encoder selects per-block based on stereo correlation
- [ ] Metadata mapping: Vorbis comment construction (vendor string + field count + fields)
- [ ] UTF-8 encoding: validate all comment strings before embedding
- [ ] Ogg page assembly: capture pattern, CRC-32, segment table, granule position
- [ ] Bitrate signaling: populate identification header bitrate fields (or leave as 0)
- [ ] Framing flag: set to 1 in identification header

### Decoding Pipeline

- [ ] Stream synchronization: locate capture pattern "OggS" at page boundaries
- [ ] CRC-32 validation: compute and compare against page_checksum field
- [ ] BOS/EOS handling: track logical stream boundaries
- [ ] Serial number tracking: distinguish between multiple logical streams
- [ ] Granule position interpretation: convert to sample number for seeking
- [ ] Three-header parsing: validate identification header fields
- [ ] Setup header parsing: build Huffman trees and VQ lookup tables from codebooks
- [ ] Audio packet parsing: extract floor, residue, coupling data per channel
- [ ] Floor reconstruction: build floor curve from partition coefficients
- [ ] Residue decoding: decode VQ indexes, look up codebook entries, cascade stages
- [ ] Combine floor + residue: per-spectral-line addition
- [ ] Channel decoupling: convert M/S back to L/R if coupling was used
- [ ] Inverse MDCT: transform spectral data to time domain per channel
- [ ] Window and overlap-add: apply sine window, overlap 50%, sum
- [ ] Output: deliver PCM samples, handle clipping (clamp to [-1.0, 1.0] for float)
- [ ] Pre-roll handling: discard first `blocksize_1/2` samples after seeking

### Metadata Handling

- [ ] Read Vorbis comments: parse vendor string, field count, field strings
- [ ] Validate field name format: uppercase letters, numbers, underscores only
- [ ] UTF-8 validation: reject or sanitize non-UTF-8 fields
- [ ] Field name normalization: convert to standard casing (TITLE, ARTIST, etc.)
- [ ] ReplayGain tags: preserve REPLAYGAIN_TRACK_GAIN/PEAK during transcode
- [ ] Multi-value fields: handle repeated field names (GENRE, COMMENT, etc.)
- [ ] Cover art: handle via Ogg attachment or vendor string extension
- [ ] Write Vorbis comments: construct bitstream with vendor + count + fields
- [ ] Field value escaping: no special escaping needed for UTF-8 text

### Format Compliance

- [ ] Capture pattern "OggS" present at every page start
- [ ] Ogg version field = 0 in all pages
- [ ] Segment table entries: 1–255 bytes per segment, 1–255 segments per page
- [ ] Identification header: version = 0, channels ≥ 1, blocksizes are powers of 2
- [ ] Comment header: valid UTF-8, vendor string present, field names valid
- [ ] Setup header: all referenced codebooks exist and are valid
- [ ] Audio packets: first bit = 0, valid floor/residue/coupling data
- [ ] Granule positions: monotonically non-decreasing within a logical stream
- [ ] CRC-32: computed over entire page including payload

### Seeking Implementation

- [ ] Seek table parsing: read index entries from comment header extension (if present)
- [ ] Binary search: bisection on byte offsets + CRC validation
- [ ] Granule-to-time conversion: `time = granule_position / sample_rate`
- [ ] Pre-roll compensation: discard first `blocksize_1/2` output samples
- [ ] Block size at seek point: decode first packet, determine blocksize_0 or blocksize_1
- [ ] Backward seeking: decode forward from nearest seek point to exact position
- [ ] Handle chained bitstreams: reset decoder state at each BOS page

### Known Compatibility Notes

- [ ] Hardware players: prefer `libvorbis` encoder over FFmpeg native
- [ ] Streaming servers (Icecast): use `libvorbis`, ensure bitrate ≤ configured max
- [ ] Matroska/MKA containers: include all three Vorbis headers in CodecPrivate
- [ ] WebM containers: verify channel layout matches Vorbis channel mapping
- [ ] Gapless playback: verify encoder delay is properly compensated
- [ ] AoTuV files: decode correctly with any standard libvorbis decoder

---

## 21. OGG CRC-32 SPECIFICATION

### 21.1 CRC-32 Polynomial and Initialization

The Ogg container uses CRC-32 with polynomial `0x04C11DB7` (the same as IEEE 802.3 Ethernet). The CRC is computed over the **entire page** including the page header, but the checksum field itself is treated as zero during computation.

**CRC-32 computation algorithm:**
```
1. Initialize CRC register to all 1s (0xFFFFFFFF)
2. For each byte of the page (including header, excluding checksum bytes):
   a. XOR the byte with the low byte of the CRC register
   b. For each of the 8 bits:
      - If the MSB of CRC is 1: CRC = (CRC << 1) XOR 0x04C11DB7
      - Else: CRC = CRC << 1
3. Finalize: invert all bits (XOR with 0xFFFFFFFF)
4. Result: 32-bit CRC checksum
```

**Implementation note:** The checksum bytes (bytes 22-25 of the page header) must be treated as zero when computing the CRC. After computing the CRC, write it into those bytes as little-endian uint32.

### 21.2 CRC-32 Verification

To verify an Ogg page:
```
1. Extract the stored CRC from bytes 22-25 (little-endian)
2. Set bytes 22-25 to 0x00, 0x00, 0x00, 0x00 in a copy of the page
3. Compute CRC-32 over the entire modified page
4. Compare computed CRC with stored CRC
5. If equal: page is intact. If not: page is corrupted
```

**Example CRC computation (pseudocode):**
```python
def compute_ogg_crc(page_bytes):
    crc = 0xFFFFFFFF
    for byte in page_bytes:
        if 22 <= page_bytes.index(byte) <= 25:
            continue  # Skip checksum bytes (treat as 0)
        crc ^= byte
        for _ in range(8):
            if crc & 0x80000000:
                crc = ((crc << 1) ^ 0x04C11DB7) & 0xFFFFFFFF
            else:
                crc = (crc << 1) & 0xFFFFFFFF
    return crc ^ 0xFFFFFFFF
```

### 21.3 Common CRC-32 Implementation Pitfalls

**Pitfall 1: Byte ordering confusion**
- The CRC polynomial is big-endian (0x04C11DB7), not little-endian
- The result is stored little-endian in the page header
- Always store/compare in the correct endianness

**Pitfall 2: Treating checksum as zeros**
- The spec requires treating checksum bytes as zero during computation
- This means you cannot compute CRC over the raw page bytes without masking

**Pitfall 3: Final XOR omission**
- Many CRC-32 implementations include a final XOR step
- The Ogg CRC requires final XOR with 0xFFFFFFFF

---

## 22. OGG PAGE SEGMENTATION RULES

### 22.1 Packet-to-Page Mapping

A Vorbis packet can span multiple Ogg pages, and an Ogg page can contain multiple Vorbis packets. The mapping is governed by:

**Rule 1: Segment table interpretation**
- Each segment table entry is 1 byte (0–255)
- Value < 255: this is the last segment of the current packet
- Value = 255: packet continues in next segment

**Rule 2: Page type flags**
- `BOS` (bit 1): First page of a logical stream. Must contain the beginning of a packet.
- `EOS` (bit 2): Last page of a logical stream. May end a packet.
- `Continuation` (bit 0): This page continues an incomplete packet from the previous page.

**Rule 3: Packet continuation**
- If a page ends with a segment value of 255, the packet continues on the next page
- The next page must have the `Continuation` flag set
- A BOS page can never have the Continuation flag

**Rule 4: Empty packets**
- A zero-length packet is encoded as: one segment entry with value 0
- Zero-length packets are valid but discouraged

### 22.2 Maximum Page Size

Ogg pages have a maximum theoretical size of:
```
Max page = 27 (header) + 255 (segment table) + 255 * 255 (payload)
         = 27 + 255 + 65,025 = 65,307 bytes
```

In practice, Vorbis pages are typically 4–8 KB. Very large pages (>64KB) may not be supported by all implementations.

### 22.3 Complete Segmentation Examples

**Example 1: Single packet fits in one page**
```
Page header: page_segments = 2
Segment table: [150, 0]  # Packet is 150 bytes, followed by empty packet
Payload: [150 bytes of data][empty]
```

**Example 2: Single packet spans two pages**
```
Page 1 header: page_segments = 3
Page 1 segment table: [255, 255, 100]
Page 1 payload: [255 + 255 + 100 = 610 bytes of packet data]
Page 1 header_type: 0x00 (no flags)

Page 2 header: page_segments = 1
Page 2 segment table: [200]
Page 2 payload: [200 bytes, continuation of packet]
Page 2 header_type: 0x01 (continuation)
```

**Example 3: Multiple packets in one page**
```
Page header: page_segments = 5
Segment table: [50, 255, 255, 30, 10]
Payload interpretation:
  - Packet 1: segments 0 = 50 bytes (ends with 50 < 255)
  - Packet 2: segments 1+2 = 255 + 255 = 510 bytes (ends with 255 = continues, 255 = continues, 30 < 255)
  - Packet 3: segment 3 = 30 bytes (ends with 30 < 255)
  - Packet 4: segment 4 = 10 bytes (ends with 10 < 255)
```

---

## 23. CODEBOOK HUFFMAN TREE CONSTRUCTION

### 23.1 Building the Decode Tree

Vorbis codebooks are constructed by building a binary Huffman tree from the codeword length list. The encoder assigns shorter codewords to more frequently used entries.

**Algorithm for building a Huffman tree from lengths:**
```
1. Create a list of entries with their codeword lengths
2. Sort by length ascending (shorter lengths first)
3. For each entry, assign a binary codeword:
   - Maintain a running counter for the current codeword
   - Codeword length determines how many bits to use
   - Pad shorter codewords with leading zeros as needed
4. Verify the Kraft-McMillan inequality: sum(2^-length[i]) = 1
5. Build the decode tree: traverse and assign left/right branches
```

**Example construction:**
```
Entries: A (len=1), B (len=2), C (len=3), D (len=3)
Kraft check: 2^-1 + 2^-2 + 2^-3 + 2^-3 = 0.5 + 0.25 + 0.125 + 0.125 = 1.0 ✓

Codeword assignment:
  A: 0          (length 1)
  B: 10         (length 2)
  C: 110        (length 3)
  D: 111        (length 3)

Decode tree:
        [root]
        /    \
       A      [node]
              /    \
             B    [node]
                  /    \
                 C      D
```

### 23.2 Sparse vs. Ordered Codebooks

**Ordered codebooks (flag = 1):**
- Lengths are stored as a sequence: first N1 entries have length 1, next N2 entries have length 2, etc.
- Reduces storage overhead when codebook entries cluster at specific lengths
- Format: `[N1: u32][N2: u32]...[Ni: u32]` for each distinct length

**Sparse codebooks (flag = 0):**
- Every entry's length is stored explicitly
- More storage but simpler to encode/decode
- Used for irregular length distributions

**Typical sparse codebook storage:**
```
For each entry i from 0 to codebook_entries - 1:
    [length_i: u8]  # 1–32 bits per codeword
```

### 23.3 Codebook Lookup Type 2 (Cumulative Cascade)

Lookup type 2 is the most common for residue encoding. Each VQ entry is computed as a cumulative sum:

**Entry format:**
```
entry_vector[j] = v_0 + v_1 + v_2 + ... + v_j
```

**Storage requirements:**
- Store: `v_0, v_1, v_2, ..., v_(dim-1)` (one value per dimension)
- For dim=4, entries=256: need 4 × 256 = 1024 values
- Each value is quantized to `2^quant_bits` levels

**Quantization:**
```
stored_value[i] = round(entry[i] / quant_delta) + quant_min
decoded_value[i] = (stored_value[i] - quant_min) * quant_delta
```

The cumulative sum provides an efficient representation of smooth vectors, reducing the storage required for related entries.

---

## 24. RESIDUE TYPE SPECIFICATIONS

### 24.1 Residue Type 0 — Non-Coupled (Independent)

In residue type 0, each channel's residue is encoded independently. This is the simplest residue type.

**Encoding process:**
```
For each channel c:
    1. Extract residue[0..end-1] for channel c
    2. Partition residue into vectors of dim coefficients
    3. For each partition:
        a. Look up best codebook entry (minimum squared error)
        b. Encode codebook index using cascade books
        c. Add quantized vector to output
    4. Repeat for cascade stages (stages 1, 2, ...)
```

**Use cases:**
- Mono audio
- Stereo with low inter-channel correlation
- Multi-channel where channels are independent
- High-bitrate encoding where coupling overhead exceeds benefit

### 24.2 Residue Type 1 — Residue-Coupled

In residue type 1, multiple channels' residues are interleaved into combined vectors before VQ encoding.

**Interleaving for 2 channels:**
```
Original: L[0..N-1], R[0..N-1]
Interleaved: [L[0], R[0], L[1], R[1], ..., L[N-1], R[N-1]]
```

**Encoding process:**
```
1. Interleave residues from 2 or more channels
2. Partition into vectors of dim coefficients
3. Encode using type 0 process
4. On decode: deinterleave to restore channel residues
```

**Use cases:**
- Stereo with moderate inter-channel correlation
- Surround channels with partial coupling
- When coupling helps but full stream coupling is excessive

### 24.3 Residue Type 2 — Stream-Coupled

In residue type 2, all channels are treated as a single interleaved stream. Maximum coupling is applied.

**Interleaving for N channels:**
```
Original: Ch0[0..N-1], Ch1[0..N-1], ..., ChN-1[0..N-1]
Interleaved: [Ch0[0], Ch1[0], ..., ChN-1[0], Ch0[1], Ch1[1], ...]
```

**Encoding process:**
```
1. Interleave ALL channels into single residue stream
2. Partition into vectors of dim coefficients
3. Encode using type 0 process
4. On decode: deinterleave back to individual channels
```

**Use cases:**
- Stereo with high inter-channel correlation (identical L/R)
- Surround sound with strong channel correlation
- Maximum compression scenarios

### 24.4 Residue Cascade Stages

Vorbis supports up to 3 cascade stages per residue. Each stage uses a different codebook to refine the approximation:

**Stage configuration in setup header:**
```
For each residue type present:
    [begin: u32] [end: u32] [partition_size: u32]
    [classifications: u8] [classbook: u8]  # Huffman book for classification
    [stage_books[0..2]: u8]  # VQ book per stage (-1 if not used)
```

**Multi-stage encoding:**
```
residue_approx = 0
For stage s = 0 to 2:
    If stage_book[s] >= 0:
        For each partition:
            residual = true_residue - residue_approx
            partition_class = classify(partition)
            book_index = decode_huffman(classbook, classification)
            codebook_entry = vorbis_codebook_decode(stage_book[s], book_index)
            residue_approx += codebook_entry
```

---

## 25. CHANNEL MAPPING SPECIFICATIONS

### 25.1 Vorbis I Channel Mapping (Mapping 0)

The standard Vorbis I channel mapping defines how audio channels are processed for encoding:

**Mapping structure in setup header:**
```
[mapping: u8] [submaps: u8] [coupling_steps: u8]
[channel_mux[0..channels-1]: u8]  # Channel to submux mapping
[ submux[0..submaps-1]: struct ]  # Submap configurations
```

**Coupling (M/S stereo):**
```
For i in 0 to coupling_steps-1:
    [magnitude[0]: u8]  # Channel to couple (magnitude)
    [angle[0]: u8]     # Channel to couple (angle/phase)
```

**Channel decoupling on decode:**
```
If channel i is magnitude:
    magnitude_channel = channel[i]
    angle_channel = channel[angle[i]]
    # Convert M/S back to L/R:
    channel[i] = (magnitude_channel + angle_channel) / 2
    channel[angle[i]] = (magnitude_channel - angle_channel) / 2
```

### 25.2 Vorbis II Channel Mapping (Custom)

For channel counts beyond 5.1, Vorbis uses an extended channel mapping (Vorbis II):

**Vorbis II channel ordering:**
| Channel | Name | Description |
|---------|------|-------------|
| 0 | L | Left |
| 1 | C | Center |
| 2 | R | Right |
| 3 | SL | Surround Left |
| 4 | SR | Surround Right |
| 5 | LFE | Low Frequency Effects |
| 6 | SLc | Back Surround Left (alternative) |
| 7 | SRc | Back Surround Right (alternative) |
| 8 | B | Back Center |
| 9 | LC | Left Width |
| 10 | RC | Right Width |

**Vorbis II coupling:**
- Pairs are defined explicitly: (L, R), (SL, SR), (Lc, Rc)
- Each pair can be independently coupled
- LFE is never coupled

### 25.3 Floor and Residue Submaps

When multiple submaps are defined, each submap has its own:
- **Floor:** Decoded floor curve for channels in the submap
- **Residue:** Encoded residue for channels in the submap

**Submap selection per channel:**
```
channel_submap[channel] = mapping[channel]
```

**Per-submap configuration:**
```
For each submap m:
    [floor[0]: u8]  # Floor type index for submap m
    [residue[0]: u8]  # Residue type index for submap m
```

This allows different psychoacoustic models for different frequency ranges or channel groups.

---

## 26. EDGE CASE HANDLING

### 26.1 Silent Blocks

When the input audio is silent (or nearly silent), Vorbis encodes efficiently:

**Floor encoding:**
- All floor partitions are classified as class 0 (low energy)
- The floor curve is encoded with minimal bits
- Most coefficients are zero

**Residue encoding:**
- Residue vectors are all zeros
- The encoder may transmit zero-length residue data
- Decoder outputs zeros

**MDCT output:**
- All spectral coefficients are zero
- After inverse MDCT, output is zero

### 26.2 Pure Tone Signals

Pure tone signals (single-frequency sine waves) present a challenge for Vorbis:

**Floor encoding:**
- The psychoacoustic model detects the tonal component
- Floor may be set low for the tonal frequency band
- Adjacent bands may be masked

**Residue encoding:**
- Residue concentrates around the tonal frequency
- VQ must represent the concentrated spectral peak accurately
- Low-bitrate encoding may smear the tone

**Block size selection:**
- For sustained tones: use long blocks (blocksize_1 = 2048 or 4096)
- Long blocks provide better frequency resolution
- Reduces spectral smearing artifacts

### 26.3 Transient Signals

Percussion, attacks, and other transients require special handling:

**Block size selection:**
- Use short blocks (blocksize_0 = 64 or 128)
- Short windows reduce pre-echo artifacts
- More frames per second increases bitrate

**Floor encoding:**
- Pre-echo masking: floor curve accounts for forward masking
- Post-echo masking: floor may be elevated after transient
- More bits allocated to the attack region

**Residue encoding:**
- Residue concentrates in early part of block
- Cascade stages may be used to refine temporal precision

### 26.4 Clipping Prevention

Vorbis operates in floating-point, so clipping can occur:

**Encoder-side:**
- Monitor peak levels during encoding
- Apply gain reduction if peaks exceed 0 dBFS
- Typical headroom: -0.3 dBFS

**Decoder-side:**
- After inverse MDCT, clamp samples to [-1.0, 1.0]
- For integer output (16-bit PCM): clamp to [-32768, 32767]
- Clipping introduces distortion but prevents audio artifacts

---

## 27. MEMORY USAGE CALCULATIONS

### 27.1 Decoder Memory Requirements

**Per-channel buffers:**
```
MDCT coefficient buffer: blocksize / 2 * sizeof(float) bytes
  - 4096 samples / 2 = 2048 coefficients × 4 bytes = 8 KB

Floor curve buffer: blocksize / 2 * sizeof(float) bytes
  - Same as MDCT: 8 KB per channel

Residue buffer: blocksize / 2 * sizeof(float) bytes
  - Same as MDCT: 8 KB per channel

Windowed input buffer: blocksize * sizeof(float) bytes
  - 4096 samples × 4 bytes = 16 KB per channel

Overlap buffer: blocksize / 2 * sizeof(float) bytes
  - 2048 samples × 4 bytes = 8 KB per channel
```

**Total per channel (4096-sample block):**
```
8 + 8 + 8 + 16 + 8 = 48 KB per channel
```

**Total for stereo (4096-sample block):**
```
48 × 2 = 96 KB
```

### 27.2 Encoder Memory Requirements

**Encoder requires additional buffers:**
```
Psychoacoustic model buffer: blocksize * sizeof(float) bytes
  - 4096 × 4 = 16 KB per channel

Bit allocation buffer: variable (depends on quality/bitrate)
  - Typically 32–128 KB

Codebook storage: depends on stream complexity
  - Typically 64–256 KB (all codebooks combined)

MDCT forward transform buffer: blocksize * sizeof(float) bytes
  - 4096 × 4 = 16 KB per channel
```

**Total encoder overhead per channel:**
```
16 (psychoacoustic) + 16 (MDCT) + overhead (bit allocation, codebooks)
≈ 64–128 KB per channel
```

### 27.3 Practical Memory Estimates

| Scenario | Block Size | Channels | Memory (KB) |
|----------|-----------|----------|-------------|
| Decode mono | 1024 | 1 | ~50 |
| Decode stereo | 1024 | 2 | ~100 |
| Decode stereo | 4096 | 2 | ~200 |
| Decode 5.1 | 4096 | 6 | ~600 |
| Encode mono | 1024 | 1 | ~100 |
| Encode stereo | 4096 | 2 | ~400 |
| Encode 5.1 | 4096 | 6 | ~1200 |

---

## 28. PERFORMANCE OPTIMIZATION

### 28.1 MDCT Optimizations

**FFT-based MDCT:**
The MDCT can be computed via FFT using standard O(N log N) algorithms:

```
MDCT(x, N):
    1. Pre-twiddle: y[n] = x[n] * cos(pi/N * (n + 0.5)) for n = 0..N-1
    2. Compute FFT(y, N)
    3. Post-twiddle: extract real parts, apply scaling
    4. Fold into N/2 output coefficients
```

**Optimizations:**
- Use split-radix FFT for sizes that are powers of 2
- Pre-compute twiddle factors (window coefficients)
- SIMD (SSE/AVX) vectorization of butterfly operations
- Cache-friendly memory access patterns

### 28.2 VQ Decode Optimizations

**Lookup table approach:**
- Pre-compute all codebook entry values
- Store as aligned arrays for SIMD access
- Direct index → pointer arithmetic for VQ lookup

**Cascade optimization:**
- Process multiple stages in a single pass
- Fused add-multiply operations
- Early termination for zero-valued entries

### 28.3 Memory Bandwidth Reduction

**Streaming decode:**
- Reuse buffers between frames (ping-pong buffers)
- Avoid allocations in the decode loop
- Pre-allocate all buffers at initialization

**Cache efficiency:**
- Process spectral data in cache-friendly order
- Use blocking for large block sizes
- Minimize cache misses in VQ lookup loops

---

## 29. TEST VECTORS AND VERIFICATION

### 29.1 Reference Test Files

The Xiph.org project provides reference test files for conformance testing:

**Test file categories:**
1. **Reference streams:** Known-good encoded files for decoder testing
2. **Bit-exact streams:** Files designed to test specific codec features
3. **Stress tests:** Edge cases and boundary conditions
4. **Round-trip tests:** Encode/decode pairs for regression testing

### 29.2 Decoder Conformance Testing

**Testing procedure:**
```
1. Obtain reference test vectors from Xiph.org
2. Decode each test file using your implementation
3. Compare output PCM against reference decoder
4. Verify sample-by-sample exactness (bit-exact match required)
5. Test error handling: corrupt test files should fail gracefully
```

**Critical test cases:**
- All block size combinations (64/256, 256/1024, etc.)
- Both floor types (0 and 1)
- All residue types (0, 1, 2)
- Channel couplings (independent, M/S, coupled)
- Edge cases: silence, pure tones, transients, clipping

### 29.3 Encoder Conformance Testing

**Testing procedure:**
```
1. Encode known test signals (sine waves, noise, music)
2. Decode with reference decoder
3. Verify output meets Vorbis specification
4. Check that decoded signal maintains expected characteristics
5. Verify bitstream can be played by third-party decoders
```

**Reference test signals:**
- 1 kHz sine wave at -3 dBFS
- Pink noise (ANSI/IES S-14)
- White noise
- Swept sine (20 Hz to 20 kHz)
- Impulse response

---

## 30. GLOSSARY OF TERMS

| Term | Definition |
|------|------------|
| MDCT | Modified Discrete Cosine Transform — lapped perfect-reconstruction frequency-domain transform |
| VQ | Vector Quantization — encoding technique that maps vectors to codebook entries |
| Codebook | A table of representative vectors used for VQ encoding/decoding |
| Floor | Perceptual masking threshold curve in the frequency domain |
| Residue | The difference between actual MDCT values and the floor curve |
| Pre-roll | Look-ahead samples required by the MDCT windowing |
| Coupling | Combining channels to exploit inter-channel redundancy |
| M/S | Mid/Side — stereo coding where mid = (L+R)/2, side = (L-R)/2 |
| BOS | Beginning of Stream — Ogg page flag indicating logical stream start |
| EOS | End of Stream — Ogg page flag indicating logical stream end |
| Granule | The sample number encoded in an Ogg page header |
| Packet | A discrete unit of Vorbis data (header or audio) |
| Page | An Ogg container unit containing one or more packet segments |
| Codeword | A binary sequence assigned to a codebook entry |
| Cascade | Multi-stage refinement encoding using multiple codebooks |
| Submap | A grouping of channels for floor and residue encoding |
| Framing flag | Bit in identification header indicating page structure |
| Window | A time-domain weighting function applied before MDCT |
| Psychoacoustic | Relating to human auditory perception and masking |
| Bark | A perceptual frequency scale approximating critical bands |
| Packet chaining | Linking multiple Vorbis logical bitstreams in one Ogg stream |
