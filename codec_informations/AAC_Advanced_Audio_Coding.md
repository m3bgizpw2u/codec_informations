# AAC (Advanced Audio Coding) — Deep Technical Reference
> **Category:** Lossy Audio
> **File Extensions:** .aac, .m4a, .mp4, .m4b, .m4p
> **MIME Types:** audio/aac, audio/aacp, audio/mp4, audio/x-m4a
> **Standardization Body:** ISO/IEC 14496-3 (MPEG-4 Audio)
> **Specification Document:** ISO/IEC 14496-3:2009 (and amendments)
> **Patent Status:** Patented (MPEG LA patent pool)
> **License:** Patent-encumbered — requires licensing for commercial encoding

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 The MP3 Problem and the Need for Successor Codecs

The success of MP3 (MPEG-1 Audio Layer III) established perceptual audio coding as the dominant paradigm for digital music distribution in the late 1990s. However, MP3 had fundamental architectural limitations that constrained further efficiency gains. The MP3 bitstream format was frozen in 1993 as part of MPEG-1, with a maximum sample rate of 48 kHz, a fixed 1152-sample frame structure, and a psychoacoustic model that, while effective, left significant coding redundancy unexploited. By the late 1990s, the audio coding research community had developed a mature understanding of these limitations and produced multiple candidate technologies that offered substantial improvements.

The Moving Picture Experts Group (MPEG) initiated the MPEG-2 AAC (Advanced Audio Coding) project as MPEG-2 Part 7 (ISO/IEC 13818-7) in 1994, published in 1997. This first-generation AAC codec offered significantly improved efficiency over MP3 — approximately 30% better compression at equivalent perceived quality — along with support for up to 48 audio channels, higher sample rates up to 96 kHz, and a more flexible tool set. However, MPEG-2 AAC was still fundamentally constrained by its parent standard's architecture. The bitstream syntax was a direct extension of MPEG-1 audio, limiting future extensibility.

MPEG-4 Audio (ISO/IEC 14496-3), published in 1999, represented a complete architectural redesign of audio coding within a unified multimedia framework. Rather than defining audio codecs as standalone entities, MPEG-4 defined a modular tool-based architecture in which AAC became one component within a larger system. This modularity enabled graceful addition of new coding tools and profiles without disrupting existing decoders. The AAC codec as defined in MPEG-4 Part 3 incorporated all MPEG-2 AAC tools and added numerous extensions.

### 1.2 Standardization Timeline

The evolution of AAC spans multiple standardization milestones, each adding capabilities while maintaining backward compatibility at the profile level:

**MPEG-2 AAC (1997):** The initial standard (ISO/IEC 13818-7) defined the AAC-LC (Low Complexity) profile, the Main profile with backward-compatible prediction, and the Scaleable Sampling Rate (SSR) profile. The bitstream syntax used ADTS (Audio Data Transport Stream) framing for raw access units and was designed for streaming and broadcast applications.

**MPEG-4 Audio (1999, 2001, 2003):** The MPEG-4 Audio specification (ISO/IEC 14496-3) absorbed MPEG-2 AAC and extended it substantially. MPEG-4 defined the AAC-LC, HE-AAC (with SBR), HE-AAC v2 (with SBR and Parametric Stereo), AAC-LD (Low Delay), and AAC-ELD (Enhanced Low Delay) profiles. It introduced a new bitstream format using the MP4/ISO Base Media container, replacing the ADTS framing of MPEG-2 AAC with the more flexible LATM (Low-overhead Audio Transport Multiplex) or audioSpecificConfig + raw access units format.

**Amendments and Extensions:** Subsequent amendments to MPEG-4 added improved psychoacoustic models, error resilience tools, and parametric coding extensions. The 2005 amendment introduced USAC (Unified Speech and Audio Coding), a unified codec that could efficiently encode both speech and general audio, though USAC is outside the scope of this document's AAC focus.

### 1.3 Design Philosophy

AAC was designed with several core principles that differentiate it from MP3:

**Modular Tool Architecture:** Unlike MP3's monolithic single-profile design, AAC provides a toolkit of coding tools from which specific profiles are constructed. A decoder conforming to a particular profile must implement all tools in that profile but may optionally implement additional tools. This allows incremental deployment and backward compatibility.

**Perceptual Coding Efficiency:** AAC's psychoacoustic model operates with finer frequency resolution than MP3's, using a filter bank with better spectral characteristics and more sophisticated masking threshold calculation. The result is lower audibility of quantization noise at equivalent bitrates.

**Stereo Efficiency:** AAC provides multiple stereo coding modes — M/S (Mid/Side), Intensity Stereo, and parametric approaches — that can reduce stereo redundancy more effectively than MP3's joint stereo modes.

**Scalability:** The SSR profile in MPEG-2 AAC and the embedded nature of HE-AAC's SBR layer provide bitrate scalability without the complexity of true scalable coding in the main profile.

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 High-Level Encoding Pipeline

The AAC encoder processes audio in a pipeline of distinct stages, each contributing to the overall compression efficiency. Understanding this pipeline is essential for comprehending how each coding tool interacts with the signal and where quality decisions are made.

```
Input PCM (32/44.1/48/64/88.2/96 kHz, 16/24-bit)
    │
    ▼
┌─────────────────────────────────────────────┐
│           PRE-PROCESSING STAGE              │
│  ─ SBR analysis filterbank (HE-AAC only)    │
│  ─ Temporal Pre-filter (TNT)                │
└─────────────────┬───────────────────────────┘
                  ▼
┌─────────────────────────────────────────────┐
│         FILTER BANK & T/F TRANSFORM         │
│  ─ MDCT (128/960/1024 samples)              │
│  ─ Kaiser-Bessel Derived window             │
└─────────────────┬───────────────────────────┘
                  ▼
┌─────────────────────────────────────────────┐
│          PSYCHOACOUSTIC ANALYSIS            │
│  ─ FFT-based spectral analysis             │
│  ─ Perceptual entropy calculation           │
│  ─ Global masking threshold                 │
└─────────────────┬───────────────────────────┘
                  ▼
┌─────────────────────────────────────────────┐
│       TNS (Temporal Noise Shaping)          │
│  ─ LPC analysis on MDCT spectral lines      │
│  ─ Quantization in temporal domain          │
└─────────────────┬───────────────────────────┘
                  ▼
┌─────────────────────────────────────────────┐
│      INTENSITY / M/S STEOREO PROCESSING     │
│  ─ M/S decision per band                   │
│  ─ Intensity stereo coupling               │
└─────────────────┬───────────────────────────┘
                  ▼
┌─────────────────────────────────────────────┐
│      PNS (Perceptual Noise Substitution)    │
│  ─ Spectral flatness measure per band       │
│  ─ Noise energy estimation                 │
└─────────────────┬───────────────────────────┘
                  ▼
┌─────────────────────────────────────────────┐
│       QUANTIZATION & HUFFMAN CODING          │
│  ─ Non-uniform quantizer (4th-root law)     │
│  ─ 1/2/3/4-dimensional Huffman tables       │
│  ─ Rate-distortion loop (outer iteration)   │
│  ─ Noise shaping loop (inner iteration)      │
└─────────────────┬───────────────────────────┘
                  ▼
┌─────────────────────────────────────────────┐
│        BITSTREAM MULTIPLEXING               │
│  ─ Single-channel (SCE)                     │
│  ─ Channel pair (CPE)                      │
│  ─ LFE channel                             │
│  ─ Data stream elements                    │
└─────────────────┬───────────────────────────┘
                  ▼
Output Bitstream (ADTS or MP4 container)
```

### 2.2 Core AAC System Components

The AAC system is composed of several interacting components that must be understood as an integrated whole. Each component's configuration affects the behavior and output of subsequent stages.

**Filter Bank:** The MDCT (Modified Discrete Cosine Transform) forms the frequency-domain representation of the audio signal. AAC uses a critically sampled, perfectly reconstructing filter bank with windowing to eliminate blocking artifacts. The filter bank is the foundation upon which all other coding tools operate.

**Psychoacoustic Model:** The psychoacoustic model computes the threshold of masking in each frequency band, representing the amount of quantization noise that can be introduced without becoming audible. This threshold drives the quantization step size in each spectral region.

**Temporal Coding Tools:** TNS (Temporal Noise Shaping) and PNS (Perceptual Noise Substitution) address specific signal characteristics that the basic transform coding cannot handle efficiently. TNS controls the temporal shape of quantization noise in transient regions; PNS replaces tonal components with synthetic noise in noise-dominated regions.

**Stereo Coding:** AAC supports multiple stereo coding modes that exploit inter-channel redundancy and irrelevance. M/S stereo encodes the mid and side channels instead of left and right; Intensity stereo uses coupling for high-frequency channels where inter-channel phase is perceptually irrelevant.

**Bitstream Syntax:** The encoded spectral data and side information are packaged into a structured bitstream using AAC-specific syntax elements (SCE, CPE, LFE, DSE) that describe the channel configuration and data layout.

### 2.3 Sample Rate and Channel Configuration

AAC supports a wide range of sampling frequencies and channel configurations. The sampling frequency is signaled in the bitstream header and determines the MDCT window size and filter bank characteristics.

**Supported Sample Rates:** 96000, 88200, 64000, 48000, 44100, 32000, 24000, 22050, 16000, 12000, 11025, 8000 Hz. These are a subset of the rates available in MP3, designed for broadcast, DVD, Blu-ray, and mobile applications.

**Channel Configurations:** AAC supports configurations from mono (1 channel) to 7.1 surround (8 channels) and beyond. The channel configuration is signaled in the bitstream, with predefined mappings for common layouts (mono, stereo, 5.1, 7.1). Custom channel mappings are possible using the Program Config Element (PCE) in MPEG-2 AAC or channel configuration fields in MPEG-4.

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 ADTS Header (for Raw .aac Files)

Raw AAC access units intended for streaming or storage in generic containers use the ADTS (Audio Data Transport Stream) framing format, defined in ISO/IEC 14496-3. Each ADTS frame prepends a 7-byte (or 9-byte with CRC) header to the raw AAC bitstream data.

#### 3.1.1 ADTS Header Structure

The ADTS header consists of 28 bits of fixed fields followed by 16 bits of variable fields (header length: 7 bytes = 56 bits total, or 9 bytes = 72 bits with CRC). The structure is as follows:

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|F|  Syncword 11bits |ID|     Layer      | -protection_absent | 0-1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|      profile     |  sampling_frequency_index   | priiviate   | 2-3
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|       channel_configuration     |    original/copy    | home   | 3-4
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                  copyright_identification_bit_start           | 4-5
+-+-+-+-+-+-+-+-+                 |     copyright_identification|
+                                  |         bit_start            |
+                                 +-+                            |
                                 |     aac_frame_length 14bits  | 5-6
+                                 +-+----------------------------+
                                 |      adts_buffer_fullness    |
+                                 +-+  11 bits                  |
+                                 |  number_of_raw_data_blocks  |
+                                 +-+  2 bits                   |
+                                 |    (optional) crc_check      |
+                                 +-+  16 bits (if prot_abs=0)  |
```

#### 3.1.2 Field-by-Field Specification

**Syncword (12 bits):** Fixed value `0xFFF` (binary: `111111111111`). Marks the start of an ADTS frame. The syncword must be byte-aligned; decoders search for this pattern to locate frame boundaries.

**ID (1 bit):** MPEG identifier. `0` = MPEG-4, `1` = MPEG-2. In MPEG-4 ADTS, this field is always `0`. This field was retained for backward compatibility with MPEG-2 stream identification.

**Layer (2 bits):** Always `00`. Indicates the audio layer. AAC always uses layer 0 (the highest layer number, corresponding to the least protection).

**protection_absent (1 bit):** Error protection flag. `1` = no CRC checksum follows the header (7-byte header). `0` = 16-bit CRC checksum follows the header (9-byte header total).

**profile (2 bits):** MPEG-4 audio profile indication. Encodes the AAC profile as `profile - 1`, meaning:
- `00`: AAC Main profile
- `01`: AAC Low Complexity (AAC-LC)
- `10`: AAC SSR (Scaleable Sampling Rate)
- `11`: AAC LTP (Long Term Prediction, defined in MPEG-2)
For MPEG-4 audio, decoders must support at minimum AAC-LC. The profile field has been largely superseded by the AudioSpecificConfig in MPEG-4 bitstreams.

**sampling_frequency_index (4 bits):** Index into the sampling rate table:
```
0: 96000 Hz    8: 7350 Hz
1: 88200 Hz    9: Reserved
2: 64000 Hz   10: Reserved
3: 48000 Hz   11: Reserved
4: 44100 Hz   12: Reserved
5: 32000 Hz   13: Reserved
6: 24000 Hz   14: Reserved
7: 22050 Hz   15: Reserved
```

**private_bit (1 bit):** Reserved for private use. Must be ignored by decoders.

**channel_configuration (3 bits):** Indicates the number and arrangement of audio channels:
```
0: Defined in AOT Specific Config
1: 1 channel (mono)
2: 2 channels (stereo, left/right)
3: 3 channels (C)
4: 4 channels (C, rear left, rear right)
5: 5 channels (C, L, R, rear left, rear right)
6: 6 channels (5.1: C, L, R, LFE, rear left, rear right)
7: 8 channels (7.1: C, L, R, LFE, side left, side right, rear left, rear right)
```

**original/copy (1 bit):** Indicates if the content is original (`0`) or a copy (`1`). Used for broadcast and recording applications.

**home (1 bit):** Reserved for home use applications.

**copyright_identification_bit (1 bit) and copyright_identification_bit_start (3 bits):** Together form a 4-bit field indicating the start of a copyright identification bit string. These bits have been deprecated in newer specifications.

**aac_frame_length (13 bits):** The total length of the ADTS frame in bytes, including the header and the raw data block(s). Must be at least 7 (header only) and at most 8191. Calculated as: `header_length + raw_data_length`. Where `header_length` is 7 if `protection_absent = 1`, or 9 if `protection_absent = 0`.

**adts_buffer_fullness (11 bits):** Indicates the bit reservoir fullness for variable bitrate (VBR) encoding. The value `0x7FF` indicates a fixed bitrate stream (CBR). For VBR streams, this field indicates the buffer fullness of the encoder's bit reservoir.

**number_of_raw_data_blocks_in_frame (2 bits):** Indicates the number of additional raw data blocks in this frame beyond the mandatory first block. Value `N` means there are `N + 1` blocks total. This allows multiple short frames to be aggregated into a single ADTS frame, providing another form of temporal multiplexing.

**crc_check (16 bits, conditional):** Present only when `protection_absent = 0`. A CRC-16 checksum covering the entire ADTS frame following the CRC field.

#### 3.1.3 ADTS Frame Length Calculation

The raw data payload size within an ADTS frame is computed as:

```
raw_data_block_count = number_of_raw_data_blocks_in_frame + 1
payload_bytes = aac_frame_length - (7 or 9)  // subtract header (with or without CRC)
bytes_per_block = payload_bytes // raw_data_block_count
```

The resulting `bytes_per_block` value is the length of each individual raw AAC access unit within the ADTS frame.

### 3.2 MP4/ISO Base Media Container Structure

When AAC audio is stored in MP4/M4A containers (defined in ISO/IEC 14496-12 and used by ISO/IEC 14496-14 for MP4 audio), the raw AAC bitstream is stored as binary data within the MP4's box/atom structure without ADTS framing. The container provides sync, timing, and metadata services that raw ADTS files lack.

#### 3.2.1 Relevant MP4 Atom Hierarchy for AAC

```
ftyp (File Type Box)
  ├─ major_brand: "M4A " or "mp4a"
  ├─ minor_version: 0
  └─ compatible_brand[0]: "M4A "
  └─ compatible_brand[1]: "mp42"
  └─ compatible_brand[2]: "isom"
  └─ ...

moov (Movie Box)
  └─ mvhd (Movie Header Box)
  └─ trak (Track Box)
      └─ tkhd (Track Header Box)
          ├─ track_ID: 1
          ├─ duration
          ├─ width/height (for video) = 0 for audio
          └─ volume (for audio) = 0x0100 (1.0)
      └─ mdia (Media Box)
          ├─ mdhd (Media Header Box)
          │   ├─ version
          │   ├─ creation_time / modification_time
          │   ├─ timescale: 1000 or 44100 (for audio)
          │   ├─ duration (in timescale units)
          │   └─ language: "und" (undefined)
          └─ minf (Media Information Box)
              ├─ smhd (Sound Media Header Box)
              │   └─ balance: 0 (center)
              ├─ dinf (Data Information Box)
              │   └─ dref → url/urn for data location
              └─ stbl (Sample Table Box)
                  ├─ stsd (Sample Description Box)
                  │   └─ mp4a (Audio Sample Entry)
                  │       ├─ version: 0
                  │       ├─ revision_level: 0
                  │       ├─ vendor: "Apple"
                  │       ├─ channel_count: 1 or 2
                  │       ├─ sample_size: 16 bits
                  │       ├─ compression_id: 0
                  │       ├─ packet_size: 0
                  │       ├─ sample_rate (fixed-point 16.16)
                  │       └─ esds (Elementary Stream Descriptor)
                  │           ├─ ES_ID
                  │           ├─ decoderConfigDescriptor
                  │           │   ├─ objectTypeIndication: 0x40 (AAC)
                  │           │   ├─ streamType: 0x05 (audio)
                  │           │   ├─ bufferSizeDB
                  │           │   ├─ maxBitrate
                  │           │   ├─ avgBitrate
                  │           │   └─ audioSpecificConfig
                  │           └─ SL descriptor (optional)
                  ├─ stts (Time-to-Sample)
                  ├─ stsc (Sample-to-Chunk)
                  ├─ stsz (Sample Size)
                  ├─ stco/co64 (Chunk Offset)
                  └─ stsd (again if needed)
mdat (Media Data Box)
  └─ raw AAC access units (no ADTS header)
```

#### 3.2.2 AudioSpecificConfig Binary Structure

The AudioSpecificConfig is the critical binary structure embedded within the `esds` box that fully describes the AAC stream's configuration. It is encoded directly from the encoder's parameters and must be parsed by decoders before any audio can be decoded.

```
 0                   1                   2
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  audioObjectType 5bits  |  samplingFrequencyIndex 3bits |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   samplingFrequencyIndex 3bits  |   channelConfiguration 3bits   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|   GASpecificConfig (variable)                                  |
+   ...                                                         |
+                                                               |
+   +----------------------------------------------------------+
+   |  frameLengthFlag    1 bit  |  dependsOnCoreCoder  1 bit  |
+   |  extensionFlag      1 bit  |  (aacSectionDataResilience 1bit) |
+   |  (aacScalefactorDataResilience 1bit) (aacSpectralDataResilience 1bit) |
+   +----------------------------------------------------------+
+                                                               |
+   (Extension Audio Object Type / SBR / PS if present)         |
+                                                               |
```

**audioObjectType (5 bits):** Identifies the coding profile/scheme:
```
1:   AAC Main
2:   AAC LC (Low Complexity)
3:   AAC SSR (Scalable Sample Rate)
4:   AAC LTP (Long Term Prediction)
5:   SBR (Spectral Band Replication, used in HE-AAC)
6:   AAC Scalable
17:  ER-AAC LC (Error Resilient AAC-LC)
19:  ER-AAC LTP
23:  ER-AAC LD
29:  ER-AAC ELD
32:  AAC LD
33:  AAC ELD
```

**samplingFrequencyIndex (4 bits in audioSpecificConfig, but 3 bits if extension):** Index into the sampling rate table. If index is `0xF`, the actual sample rate is encoded as a 24-bit value following the index field.

**channelConfiguration (4 bits, actually 3 bits):** Identical in meaning to the ADTS `channel_configuration` field.

**GASpecificConfig:** Contains additional configuration for the General Audio profile. The `frameLengthFlag` indicates 1024 (0) or 960 (1) sample window. The `dependsOnCoreCoder` flag indicates whether the stream depends on a core coder (used in scalable contexts). The `extensionFlag` indicates the presence of an extension payload.

### 3.3 AAC Element Types (ID_SCE, ID_CPE, ID_LFE, etc.)

The AAC bitstream is organized into a hierarchical sequence of elements. Each element represents a collection of encoded audio data. The bitstream parser identifies element types by their element IDs before decoding.

**ID_SCE (Single Channel Element, ID=0):** Encodes a single audio channel independently. Used for mono content, center channel in surround mixes, or any independently coded channel. Contains a single set of spectral data.

**ID_CPE (Channel Pair Element, ID=1):** Encodes a pair of channels together, enabling joint stereo coding. Both channels share common side information (scale factors, M/S flags, TNS parameters) while maintaining independent spectral data or sharing it via M/S or Intensity stereo coding. The CPE is the fundamental building block for stereo encoding in AAC.

**ID_CCE (Channel Coupling Element, ID=2):** Used for coupling multiple channel elements together. Rarely used in practice, as the CPE handles most multi-channel coupling scenarios.

**ID_LFE (Low Frequency Effects Element, ID=3):** Encodes the LFE (subwoofer) channel. The LFE contains very limited bandwidth (typically below 120 Hz) and is encoded with fewer spectral lines than a full channel. Its presence is signaled in the channel configuration.

**ID_DSE (Data Stream Element, ID=4):** Carries ancillary data that is not audio. Can be used for metadata, DRM information, or other side-channel data. Decoders should skip DSE elements.

**ID_PCE (Program Config Element, ID=5):** Defines a custom channel configuration when the standard channel configurations (1-7) are insufficient. Contains explicit mapping of element tags to speaker positions. Used primarily in MPEG-2 AAC broadcast streams.

**ID_FIL (Fill Element, ID=6):** Contains fill data used for bitrate padding, extension payload (SBR/PS), and ancillary data. Fill elements can carry SBR data (in HE-AAC) or padding to reach a target bitrate.

**ID_END (Element ID=7):** Marks the end of the audio frame. Used in error-resilient bitstreams.

The element sequence within each AAC frame follows a predefined order: SCE, CPE, CCE, LFE, DSE, PCE, FIL. Multiple elements of the same type can appear in sequence, distinguished by their instance tags.

### 3.4 Complete Field-by-Field Bitstream Map

A complete AAC bitstream frame (raw_data_block) consists of a sequence of audio elements, each containing multiple data structures that collectively describe the encoded audio.

#### 3.4.1 Single Channel Element (SCE) Bit Layout

```
[Element ID: 4 bits] = 0b0000 (ID_SCE)
[Instance tag: 4 bits] = 0..15 (allows multiple SCEs in one frame)

[Common Window]
├─ global_gain: 8 bits (scalar applied to all scale factors)
├─ ics_reserved_bit: 1 bit
├─ window_sequence: 2 bits
│   0b00 = ONLY_LONG_SEQUENCE
│   0b01 = LONG_START_SEQUENCE
│   0b10 = EIGHT_SHORT_SEQUENCE
│   0b11 = LONG_STOP_SEQUENCE
├─ window_shape: 1 bit (0=Sine, 1=Kaiser-Bessel Derived)
├─ max_sfb: 6 bits (maximum scale factor bands per window group)
├─ scale_factor_grouping: 7 bits (present only if window_sequence=EIGHT_SHORT)
└─ predictor_data_present: 1 bit
    ├─ (TNS data if TNS present)
    ├─ (ms_mask_present: 1 bit) (only for CPE; not in SCE)
    └─ (section_data per window group)
        ├─ global_gain: 8 bits (already read above)
        ├─ section_offset: variable (Huffman coded)
        └─ (spectral_data per band, Huffman coded)
```

#### 3.4.2 Channel Pair Element (CPE) Bit Layout

The CPE shares the common window information between its two channels (left and right), then encodes per-channel spectral data with optional M/S and Intensity stereo processing.

```
[Element ID: 4 bits] = 0b0001 (ID_CPE)
[Instance tag: 4 bits]

[Common Window] (shared by both channels)
├─ global_gain: 8 bits
├─ window_sequence: 2 bits
├─ window_shape: 1 bit
├─ max_sfb: 6 bits
├─ scale_factor_grouping: 7 bits (if short blocks)
└─ predictor_data_present: 1 bit

[TNS data] (per channel if TNS present)
├─ TNS_present: 1 bit (per window in common window)
│   └─ n_filt[i]: 2 bits (0-3 filters per window)
│   └─ coef_res[i]: 1 bit (coefficient resolution)
│   └─ length[i]: 6 bits per filter
│   └─ order[i]: 5 bits per filter
│   └─ direction[i]: 1 bit per filter
│   └─ coef_compress[i]: 1 bit per filter
│   └─ coef[i]: variable length per filter
└─ ...

[ms_mask_present: 2 bits] (0=ms_used[n]=0 for all bands, 1=all ms_used[n]=1, 2=explicit)
├─ If ms_mask_present == 2:
│   └─ ms_used[0..max_sfb-1]: 1 bit each

[Per-channel spectral data]
├─ Channel 0 (ch1): Huffman-coded spectral data + scale factors
└─ Channel 1 (ch2): Huffman-coded spectral data + scale factors (or M/S derived)
```

#### 3.4.3 Scale Factor Band Layout

AAC divides the frequency spectrum into scale factor bands (sfb) whose boundaries are defined relative to the MDCT frequency resolution. The band boundaries are defined differently for long blocks (1024 samples, 128 max sfb) and short blocks (128 samples, grouped into 8 windows, 1024/128=8 groups). The actual band boundaries depend on the sampling rate and are looked up from Table 4.5.3 in ISO/IEC 14496-3.

For 44.1 kHz (the most common sample rate):
- **Long blocks:** 1024 MDCT lines, divided into 49-60 scale factor bands depending on profile
- **Short blocks:** 128 MDCT lines per window, divided into 14 scale factor bands

The scale factor encoding encodes the quantization step size for each scale factor band. Each scale factor is encoded as a difference from the previous band (or global gain for the first band) using a dedicated Huffman table optimized for small differences (most scale factors differ by 0-2 steps).

### 3.5 Sample Format Support

AAC does not define a fixed input or output sample format; the codec operates on internally transformed spectral data. However, the surrounding system (encoder input, decoder output) typically uses one of a few standard formats.

**Input to Encoder:** PCM audio in signed 16-bit or 24-bit integer format, or 32-bit floating point. The encoder may optionally accept floating-point input directly. Sample rates of 8, 11.025, 12, 16, 22.05, 24, 32, 44.1, 48, 64, 88.2, 96 kHz are supported, with 44.1 kHz and 48 kHz being the most common.

**Internal Processing:** Most modern AAC encoders use 32-bit floating-point or 64-bit double-precision arithmetic internally. Fixed-point AAC encoders exist for embedded applications and typically use 32-bit or 16-bit integer arithmetic with appropriate scaling.

**Output from Decoder:** PCM audio matching the input sample rate, in signed 16-bit, 24-bit, or 32-bit floating-point format. The output format is determined by the decoder implementation and is typically configurable.

---

## 4. ENCODING ALGORITHM (DEEP DETAIL)

### 4.1 Pre-Processing Stage

Before the audio enters the core transform coding pipeline, several optional pre-processing operations may be applied depending on the encoder configuration and signal characteristics.

**DC Offset Removal:** Many audio sources (particularly from analog-to-digital conversion) contain a small DC (zero-frequency) component. This DC component wastes encoding dynamic range without contributing to perceived audio quality. Most AAC encoders apply a high-pass filter with a very low cutoff frequency (typically 2-5 Hz) to remove DC offset before encoding.

**Pre-filter / Anti-Aliasing:** The MDCT filter bank requires input windowing that can introduce pre-ringing artifacts at block boundaries in transient signals. Some encoders apply a pre-emphasis filter that boosts high-frequency content before encoding, which is compensated by de-emphasis in the decoder. This technique reduces pre-ringing by spreading the transient's energy more evenly across frequency bins.

**SBR Analysis (HE-AAC only):** When encoding with HE-AAC (AAC + SBR), the input signal is analyzed by a QMF (Quadrature Mirror Filter) bank that splits the signal into 64 sub-bands. The SBR encoder analyzes the low-frequency portion (roughly up to Nyquist/2) and estimates parametric parameters that describe the high-frequency content. These parameters include spectral envelope, noise floor estimate, and tonal map. The SBR analysis operates at 2x the sample rate of the baseband AAC encoding.

**Channel Routing:** For multi-channel input (e.g., 5.1 surround), the encoder maps the input channels to the output channel configuration. A 5.1 input might be mapped to a CPE (for L/R stereo), an SCE (for center), an LFE (which is passed through with minimal encoding), and potentially a coupling strategy for rear channels.

### 4.2 Modified Discrete Cosine Transform (MDCT)

The MDCT is the heart of AAC's transform coding engine. Unlike the DCT used in MP3, the MDCT has the critical property of being critically sampled — the transform produces the same number of frequency coefficients as there were input samples — and uses time-domain aliasing cancellation (TDAC) to achieve perfect reconstruction.

#### 4.2.1 MDCT Mathematical Definition

The MDCT forward transform for a block of `N` input samples `x[n]` producing `N/2` coefficients `X[k]` is defined as:

```
X[k] = sum(n=0 to N-1) { x[n] * w[n] * cos(pi/N * (n + 0.5 + N/2) * (k + 0.5)) }
       for k = 0, 1, ..., N/2 - 1
```

Where `w[n]` is the analysis window function. The inverse MDCT (IMDCT) reconstructs `N` samples from `N/2` coefficients:

```
y'[n] = (2/N) * sum(k=0 to N/2-1) { X[k] * cos(pi/N * (n + 0.5 + N/2) * (k + 0.5)) }
        for n = 0, 1, ..., N-1
```

The critical sampling property means `N` input samples produce `N/2` output coefficients, and `N/2` coefficients produce `N` output samples. Adjacent blocks overlap by 50% (N/2 samples), and the TDAC property ensures that the overlapping portions sum to reconstruct the original signal.

#### 4.2.2 Window Functions

AAC supports two window shapes that determine the spectral characteristics of the filter bank:

**Sine Window:** The default window, providing good spectral selectivity with moderate sidelobe levels. Defined as:

```
w[n] = sin(pi/N * (n + 0.5))
```

**Kaiser-Bessel Derived (KBD) Window:** Provides better spectral selectivity (narrower mainlobes, lower sidelobes) at the cost of slightly higher overlapping aliasing. Derived from a Kaiser-Bessel window function:

```
w[n] = I0(beta * sqrt(1 - (2n/N - 1)^2)) / I0(beta)
```

Where `I0` is the modified Bessel function of the first kind, and `beta` is typically 4-6 for audio applications. The KBD window provides approximately 3-6 dB better spectral isolation than the sine window.

**Window Sequence Types:** AAC supports four window sequences that control the time-frequency resolution trade-off:

| Window Sequence | Description | Transform Size | Use Case |
|---|---|---|---|
| ONLY_LONG_SEQUENCE | Single long window | 2048 → 1024 MDCT | Stationary signals |
| LONG_START_SEQUENCE | Long start → short stop | 2048+256 → 1024+128 | Transients (start) |
| EIGHT_SHORT_SEQUENCE | Eight short windows | 8×256 → 8×128 | Transient regions |
| LONG_STOP_SEQUENCE | Long start → short stop | 2048+256 → 1024+128 | Transients (end) |

**Short Block Transition:** When a transient is detected (typically by measuring the increase in perceptual entropy between consecutive short segments), the encoder switches from long to short windows. The LONG_START_SEQUENCE processes the first block with a long window (to allow smooth transition from the previous block), followed by EIGHT_SHORT_SEQUENCE blocks (to achieve fine temporal resolution on the transient), and finally a LONG_STOP_SEQUENCE to transition back to long windows. This approach minimizes both pre-ringing (from long-window processing of transient energy) and post-ringing artifacts.

**Block Switching Decision:** The encoder decides when to switch window types based on several heuristics:
- **Perceptual Entropy Jump:** A large increase in perceptual entropy between consecutive frames indicates a transient.
- **Energy Distribution:** High energy concentrated in a short time span relative to the block size.
- **Bitrate Demand:** Short blocks can sometimes reduce bitrate demand for transient-heavy content because quantization errors in one short block don't propagate to others.
- The decision is made independently for each channel in a CPE (channel pair), and the common window flag determines whether both channels use the same window sequence.

#### 4.2.3 Frequency Resolution

For a 1024-sample MDCT at 44.1 kHz:
- Frequency resolution: 44100 / 1024 ≈ 43 Hz per bin
- Nyquist frequency: 22050 Hz
- Total coefficient bins: 512 (positive frequencies)

For an 8×128-sample short block MDCT:
- Frequency resolution: 44100 / 128 ≈ 345 Hz per bin
- The trade-off is temporal resolution vs. frequency resolution

The MDCT frequency bins are not perceptually uniform — bass frequencies are spread across relatively few bins, while treble frequencies occupy many bins. The psychoacoustic model accounts for this by analyzing the spectrum in bark-scale bands.

### 4.3 Psychoacoustic Model

The psychoacoustic model is the perceptual foundation of AAC encoding. It computes the threshold of audible distortion — the maximum amount of quantization noise that can be introduced without being perceived — for each frequency region of the audio signal.

#### 4.3.1 Masking Fundamentals

Auditory masking occurs when a sound (the masker) renders another sound (the maskee) inaudible. Two types of masking are relevant for audio coding:

**Simultaneous (Spectral) Masking:** A masker tone at frequency `f` masks other sounds occurring at the same time, within a critical band around `f`. The masking effect is strongest for maskees close in frequency and drops off with a roughly triangular spread in the bark domain. The threshold of masking depends on the masker's level, type (tone vs. noise), and frequency.

**Temporal Masking:** A masker also affects sounds before and after its occurrence. Pre-masking (forward masking) extends approximately 20-50 ms before the masker, while post-masking (backward masking) extends 50-200 ms after. AAC does not directly exploit temporal masking in its standard psychoacoustic model, though temporal coding tools like TNS indirectly affect the temporal shape of quantization noise.

#### 4.3.2 Hearing Threshold and Absolute Threshold

Below the absolute threshold of hearing (ATH), sound is inaudible regardless of masking. The ATH varies with frequency, reaching its minimum around 2-4 kHz (where human hearing is most sensitive) and rising at both lower and higher frequencies. The ATH is approximately:

```
ATH(f_bark) ≈ 3.64 * (f_kHz)^(-0.8) - 6.5 * exp(-0.6 * (f_kHz - 3.3)^2) + 10^(-3) * (f_kHz)^4
```

This formula produces values in dB SPL (Sound Pressure Level). The encoder uses this as a floor — if the quantization noise floor is below the ATH in a given band, it is inaudible regardless of masking considerations.

#### 4.3.3 Perceptual Entropy

The psychoacoustic model in AAC computes a quantity called Perceptual Entropy (PE) for each block. PE is a measure (in bits) of the minimum number of bits required to encode the audio perceptually losslessly — that is, with quantization noise below the masking threshold. PE drives both window switching decisions and the bit allocation in the rate-distortion loop.

```
PE = sum(band) { N_b * log2(SMR_b + 1) }
```

Where `N_b` is the number of spectral bins in band `b`, and `SMR_b` is the Signal-to-Mask Ratio in band `b` (the ratio of signal energy to masking threshold energy in that band).

High PE indicates that the block is perceptually complex and will require many bits to encode transparently. When PE increases dramatically between frames, the encoder switches to short blocks to maintain quality.

#### 4.3.4 Global Masking Threshold

For each scale factor band, the encoder computes the global masking threshold as:

```
T_gm[b] = max(T_hearing[b], T_tonal[b] * T_noise[b])
```

Where:
- `T_hearing[b]` is the absolute threshold of hearing in band `b`
- `T_tonal[b]` is the sum of masking contributions from tonal (sine-like) maskers in band `b`
- `T_noise[b]` is the masking contribution from noise maskers in band `b`

The distinction between tonal and noise maskers matters because their masking functions have different shapes. Tonal maskers produce narrow-band masking with a steep falloff; noise maskers produce broader masking with a gentler falloff.

### 4.4 TNS (Temporal Noise Shaping)

TNS is one of AAC's most important coding tools, specifically addressing the temporal characteristics of quantization noise in transient regions. Without TNS, quantization noise in a transform-coded block has a fixed temporal envelope — the noise is distributed uniformly across the block's duration. For stationary signals, this uniform distribution is perceptually optimal. But for transient signals, it creates audible pre-ringing artifacts: the quantization noise preceding the transient is clearly audible because the signal energy is low before the transient.

#### 4.4.1 TNS Concept

TNS applies linear prediction to the spectral coefficients of the MDCT. Rather than encoding the spectral coefficients directly, TNS encodes a residual spectral envelope. This is equivalent to shaping the quantization noise in the temporal domain.

The key insight: if the MDCT spectral coefficients `X[k]` can be predicted from neighboring coefficients using linear prediction (LP), then the prediction residual `R[k] = X[k] - sum(i=1 to order) { a[i] * X[k-i] }` has a flatter temporal envelope (in the autocorrelation sense) than the original `X[k]`. When `R[k]` is quantized and the prediction coefficients `a[i]` are transmitted as side information, the decoder applies inverse prediction, which reconstructs `X'[k]`. The quantization noise in `X'[k]` is shaped to follow the temporal envelope of the original signal — it increases during transients and decreases during quiet periods. This temporal shaping aligns the noise with high signal energy, where it is masked, minimizing audibility.

#### 4.4.2 TNS Implementation

TNS encoding proceeds as follows:

1. **LP Analysis:** For the current window of `N` MDCT coefficients, perform linear prediction analysis. The prediction order ranges from 0 (no TNS) to 12. Higher order provides more precise noise shaping but requires more bits for the prediction coefficients.

2. **Coefficient Quantization:** The LP coefficients `a[i]` (or PARCOR coefficients for the reflection coefficient representation) are quantized with a fixed resolution (1 bit for coefficient resolution, variable bits for coefficient values). The `coef_res` flag indicates whether coefficients use 3 or 4 bits of precision.

3. **Residual Encoding:** The prediction residual `R[k]` is quantized using the same Huffman/quantization pipeline as the regular spectral data. The residual typically has lower energy than the original coefficients, which can improve coding efficiency.

4. **Band Limitation:** TNS is typically applied only to the lower frequency portion of the spectrum (e.g., bands 0-31 out of 49-60 total bands). High-frequency coefficients are encoded directly because their temporal characteristics are less perceptually relevant and the LP analysis overhead would not be justified.

The TNS filter can be applied in either forward or backward direction, controlled by the `direction` flag. The filter length is specified separately for each window in the sequence.

#### 4.4.3 TNS in Short Blocks

For EIGHT_SHORT_SEQUENCE, TNS operates on each of the 8 short windows independently. This means TNS can provide very fine temporal resolution — with 128-sample windows at 44.1 kHz, each window covers approximately 2.9 ms. This enables precise noise shaping around transients at the cost of increased side information overhead (TNS data must be transmitted for each short window).

### 4.5 Quantization and Coding

The quantization and coding stage is where the actual bitrate reduction occurs. AAC uses a non-uniform quantizer followed by Huffman coding, with a rate-distortion optimization loop controlling the bit allocation.

#### 4.5.1 Non-Uniform Quantizer

AAC uses a 4th-root (quartic) non-uniform quantizer. The spectral coefficients `X[k]` are quantized as:

```
Y[k] = round( sign(X[k]) * |X[k]|^0.25 )
```

This non-uniform quantization has several advantages:
- **Perceptual Weighting:** The 4th-root law approximates the perceptual loudness curve, giving more resolution to low-amplitude coefficients where the ear is more sensitive to amplitude differences.
- **Reduced Dynamic Range:** The 4th-root compression reduces the range of coefficient magnitudes, requiring fewer bits per Huffman symbol.
- **Huffman Efficiency:** The distribution of 4th-root quantized coefficients is more concentrated (peaked at small absolute values), which is better matched to Huffman coding tables.

The quantized values `Y[k]` are integers, typically in the range [-8191, +8191] for 13-bit signed representation. Zero is the most frequent value.

#### 4.5.2 Huffman Coding

The quantized spectral coefficients are entropy-coded using Huffman coding. AAC defines 12 Huffman codebooks for different types of spectral data:

| Codebook | Dimensions | Spectral Data Type |
|---|---|---|
| 0 | 1 | Escape-coded (unscaled) values |
| 1 | 2 | Low-energy stereo pairs |
| 2 | 2 | General pairs (low values) |
| 3 | 2 | General pairs (moderate values) |
| 4 | 2 | General pairs (high values) |
| 5 | 2 | General pairs (with sign) |
| 6 | 2 | General pairs (higher values) |
| 7 | 2 | General pairs (maximum values) |
| 8 | 4 | Four-tuples (low) |
| 9 | 4 | Four-tuples (moderate) |
| 10 | 4 | Four-tuples (high) |
| 11 | 4 | Four-tuples (maximum) |
| 12 | 2 | Reserved (Spectral Pairs — SSR profile) |
| 13 | 4 | Reserved (Spectral Lines — SSR profile) |

The encoder selects the optimal codebook for each group of spectral coefficients based on the magnitude of the values. Codebook 0 indicates that the value is too large to code with a Huffman table and instead uses an escape sequence: a prefix of all zeros followed by a direct binary encoding of the value.

#### 4.5.3 Rate-Distortion Loop (Outer Iteration)

AAC uses a two-loop rate-distortion optimization process to allocate bits optimally across scale factor bands:

**Outer Loop (Rate Loop):** Adjusts the global quantization step size until the total bit consumption is within the target bit budget. At each iteration, the quantization step size is adjusted, scale factors are recomputed, and the resulting bitrate is estimated. If bits remain, the loop continues; if bits are exceeded, the loop reduces quality in bands with the lowest perceptual impact.

**Inner Loop (Noise Shaping Loop):** Within the rate loop, an inner iteration adjusts the quantization step size per scale factor band to shape the quantization noise to follow the masking threshold. For each band, the quantization step is adjusted based on the ratio of actual signal energy to the masking threshold. Bands where the signal is well above the threshold get coarser quantization; bands near the threshold get finer quantization.

The combined effect is bit allocation that:
1. Respects the total bit budget
2. Shapes quantization noise to be below the masking threshold in each band
3. Maximizes perceptual quality for the given bitrate

The number of iterations in each loop is typically limited (e.g., 2 outer iterations, 4 inner iterations per outer iteration) to control encoder complexity.

#### 4.5.4 Scale Factor Encoding

Scale factors are encoded differentially — each band's scale factor is encoded as the difference from the previous band's scale factor. This exploits the strong correlation between adjacent scale factor bands. The difference values are Huffman-coded using a dedicated table optimized for small integers (typically -4 to +4). A special escape code represents larger differences.

### 4.6 M/S Stereo (Mid/Side)

M/S stereo coding exploits the perceptual redundancy between the left and right channels in stereo recordings. When two channels are similar (which is common in most music), encoding them as mid and side channels rather than left and right can reduce bitrate.

#### 4.6.1 M/S Transform

```
M = (L + R) / 2    (mid channel — the sum)
S = (L - R) / 2    (side channel — the difference)
```

At the decoder, reconstruction is:

```
L = M + S
R = M - S
```

When L and R are identical (mono content), the side channel S is zero and no bits are needed to encode it. When L and R are very different (highly uncorrelated), the mid channel carries most of the energy and the side channel is small, but not zero. The worst case is when L = -R (perfectly out of phase), which gives M = 0 and S = L, doubling the energy in the side channel relative to the mid.

#### 4.6.2 M/S Decision and Bit Savings

The encoder makes the M/S decision per scale factor band. For each band, it compares the bits required to encode L and R independently versus the bits required to encode M and S. The cheaper option is chosen.

```
Cost_Independent = bits(L) + bits(R)
Cost_MS = bits(M) + bits(S)
Use_M/S if Cost_MS < Cost_Independent
```

The `ms_used[n]` flag (transmitted per scale factor band) signals to the decoder which mode was chosen. `ms_mask_present = 0` means no M/S coding is used; `ms_mask_present = 1` means M/S is used for all bands; `ms_mask_present = 2` means per-band `ms_used` flags are transmitted.

In practice, M/S coding typically saves 10-30% on stereo channel data for well-correlated stereo content.

### 4.7 Intensity Stereo Coding

Intensity stereo coding (also called coupling) exploits a perceptual property of human hearing at high frequencies: above approximately 2 kHz, the ear is insensitive to the phase difference between left and right channels. Only the overall envelope (intensity) matters. This enables a more aggressive form of channel coupling.

#### 4.7.1 Intensity Stereo Mechanism

In Intensity Stereo mode, only a single downmix channel is encoded for a group of high-frequency bands. The decoded stereo signal is reconstructed by applying intensity (scale factor) information to the mono signal for each channel. The decoder synthesizes the stereo image by scaling the mono signal differently for left and right channels:

```
L'[band] = scaleL[band] * mono[band]
R'[band] = scaleR[band] * mono[band]
```

Where `scaleL` and `scaleR` are the intensity coefficients transmitted per band. The sum `scaleL + scaleR = 1` is maintained to preserve overall energy.

#### 4.7.2 When to Use Intensity Stereo

Intensity Stereo is applied at high frequencies (typically above 6-8 kHz for 44.1 kHz audio) where the phase irrelevance property holds. It provides significant bitrate savings for high-frequency content because only one channel's spectral data needs to be transmitted. However, it destroys phase information, making it unsuitable for low-frequency content or for content where stereo phase differences are perceptually important (e.g., certain drum patterns, stereo widening effects).

### 4.8 Perceptual Noise Substitution (PNS)

PNS addresses a fundamental limitation of transform coding: transform coders are efficient for tonal (sinusoidal) signals but are inefficient for noise-like signals. A pure noise signal has no structure for the transform to exploit — every spectral coefficient carries roughly equal energy, and the entropy is high. PNS sidesteps this by detecting noise-dominated spectral regions and replacing them with synthetic noise at the decoder, freeing bits for tonal regions.

#### 4.8.1 PNS Detection

PNS identifies spectral regions that are more efficiently represented as noise than as coded spectral coefficients. The detection is based on the spectral flatness measure (SFM) within each scale factor band:

```
SFM = G_m / A_m
G_m = (product(k in band) { |X[k]| })^(1/N)   // geometric mean
A_m = (1/N) * sum(k in band) { |X[k]| }       // arithmetic mean
```

A low SFM (approaching 0) indicates a tonal signal — the energy is concentrated in a few dominant spectral components. A high SFM (approaching 1) indicates a noise-like signal — all components have similar energy. If SFM exceeds a threshold, the band is classified as noise-like and is a candidate for PNS.

Additionally, the total energy of the band must be below a threshold relative to the masking threshold — if the band is already near or below the masking threshold, it doesn't matter whether it's coded as tonal or noise.

#### 4.8.2 PNS Encoding and Decoding

When a band is coded with PNS:
- The encoder transmits the total energy of the band (quantized) instead of individual spectral coefficients.
- The `pns_used` flag for the band is set.
- The decoder synthesizes a noise signal with the transmitted energy, using a pseudo-random number generator seeded consistently (or using a deterministic noise pattern).
- The synthetic noise is added to the decoded output in place of the original spectral data.

The PNS energy is transmitted as a differential value relative to the previous band's PNS energy, using a dedicated Huffman table. This makes PNS bands very cheap to encode — only the energy needs to be transmitted, not the spectral structure.

### 4.9 Long Term Prediction (LTP)

Long Term Prediction (LTP) was defined in MPEG-2 AAC but is rarely used in practice. It provides a time-domain prediction mechanism that can exploit periodic structures in the audio signal (e.g., pitched instruments, vibrato).

#### 4.9.1 LTP Mechanism

LTP works similarly to the long-term prediction in speech codecs (e.g., CELP). It predicts the current block's time-domain samples from a previous block using a pitch lag (lag) and gain coefficients. The prediction residual is then transform-coded using the regular MDCT pipeline.

```
x_predicted[n] = sum(i=0 to lag-1) { b[i] * x[n - pitch_lag - i] }
```

The LTP parameters (pitch lag, gain, filter coefficients) are transmitted as side information. LTP can significantly reduce bitrate for signals with strong periodic components, but the prediction overhead and complexity have limited its adoption.

Most modern AAC encoders (including those in FFmpeg and Apple's encoder) do not implement LTP due to its complexity and marginal benefit for general music content.

### 4.10 AAC Profiles and Bitrate Settings

AAC defines multiple profiles that specify which coding tools must/may be implemented. The choice of profile determines the encoder complexity, decoder complexity, and the achievable bitrate range.

| Profile | Main Tools | Typical Bitrate | Use Case |
|---|---|---|---|
| AAC-LC | MDCT, TNS, M/S, PNS | 64-320 kbps per channel | General audio (mobile to HD) |
| HE-AAC v1 | AAC-LC + SBR | 32-128 kbps per channel | Streaming, mobile |
| HE-AAC v2 | HE-AAC v1 + PS | 16-64 kbps per channel | Ultra-low bitrate streaming |
| AAC-LD | AAC-LC + Low-delay tools | 64-256 kbps | Intercom, telepresence |
| AAC-ELD | AAC-LD + SBR | 32-128 kbps | VoIP, broadcast |

The encoder's bitrate is controlled via:
- **CBR (Constant Bitrate):** Fixed bits per frame. Simple but may waste bits on simple content and struggle with complex content.
- **VBR (Variable Bitrate):** Target average bitrate. The encoder adjusts bits per frame based on perceptual complexity. Achieves better quality-per-bitrate than CBR at the cost of variable file size.
- **ABR (Average Bitrate):** A compromise — targets an average bitrate while allowing per-frame variation. Better than CBR for quality but more predictable than VBR for file size.
- **CRF (Constant Rate Factor):** Used in some encoders (not native AAC) — maintains constant quality by varying bitrate as needed. FFmpeg's native AAC encoder supports CRF-like quality targeting.

---

## 5. DECODING ALGORITHM

### 5.1 Stream Synchronization

Before any audio can be decoded, the decoder must locate and synchronize with the AAC bitstream. The synchronization process differs between raw ADTS streams and MP4/M4A container-wrapped streams.

#### 5.1.1 ADTS Synchronization

In raw ADTS streams, the decoder searches for the sync word `0xFFF`. The search must handle:
- False positives (arbitrary data that happens to contain `0xFFF`)
- Bit errors that corrupt the sync word
- Variable bitrate streams with padding

The decoder reads the ADTS header, extracts the frame length, and advances to the next frame. If the frame length is invalid (too small, too large, or inconsistent with the bitstream's inner structure), the sync search continues. Most decoders maintain a sync state machine with hysteresis — once synchronized, they remain synchronized unless multiple consecutive frame failures occur.

#### 5.1.2 MP4 Container Synchronization

In MP4 containers, the raw AAC bitstream is stored as a series of `mdat` payloads referenced by the `stbl` (Sample Table Box). Each sample in the `stbl` points to a raw AAC access unit. The container provides:
- Sample-accurate timing via the `stts` (Time-to-Sample) box
- Byte-accurate seeking via the `stco` (Chunk Offset) box
- Sample size information via the `stsz` (Sample Size) box

The decoder parses the `AudioSpecificConfig` from the `esds` box once during stream initialization to determine the profile, sample rate, and channel configuration. It then processes raw access units sequentially or at the requested seek position.

### 5.2 Core Decoding Pipeline

The AAC decoder reverses the encoder pipeline. Given a valid bitstream, it reconstructs the quantized spectral data, applies inverse quantization, inverse TNS, inverse M/S (if applicable), and synthesizes time-domain audio via the IMDCT filter bank.

```
Input: AAC Bitstream
    │
    ▼
┌─────────────────────────────────────────────┐
│        BITSTREAM UNPACKING                 │
│  ─ Parse element IDs and instance tags     │
│  ─ Extract Huffman-coded spectral data     │
│  ─ Decode scale factors                    │
│  ─ Extract TNS parameters                 │
│  ─ Extract M/S and PNS flags               │
└─────────────────┬───────────────────────────┘
                  ▼
┌─────────────────────────────────────────────┐
│        HUFFMAN DECODING                     │
│  ─ Decode Huffman codewords → quantized    │
│    spectral coefficients                   │
│  ─ Handle escape sequences for large values │
└─────────────────┬───────────────────────────┘
                  ▼
┌─────────────────────────────────────────────┐
│        SCALE FACTOR APPLICATION             │
│  ─ Apply scale factors to each band        │
│  ─ Reconstruct quantized spectral values   │
└─────────────────┬───────────────────────────┘
                  ▼
┌─────────────────────────────────────────────┐
│        INVERSE QUANTIZATION                 │
│  ─ Apply inverse 4th-root law              │
│  ─ Y[k]^4 → reconstructed |X'[k]|          │
│  ─ Restore sign from Huffman decode        │
└─────────────────┬───────────────────────────┘
                  ▼
┌─────────────────────────────────────────────┐
│        INVERSE M/S STEREO                   │
│  ─ Convert Mid/Side back to Left/Right     │
│  ─ Use ms_used flags per band              │
└─────────────────┬───────────────────────────┘
                  ▼
┌─────────────────────────────────────────────┐
│        INVERSE TNS                          │
│  ─ Apply inverse LP filter to spectral data │
│  ─ Reconstruct temporal envelope           │
└─────────────────┬───────────────────────────┘
                  ▼
┌─────────────────────────────────────────────┐
│        INVERSE QUANTIZE PNS                 │
│  ─ Replace PNS bands with synthetic noise  │
│  ─ Energy-matched pseudo-random signal     │
└─────────────────┬───────────────────────────┘
                  ▼
┌─────────────────────────────────────────────┐
│        IMDCT + WINDOWING                    │
│  ─ Apply appropriate window (sine or KBD)  │
│  ─ Transform to time domain                │
└─────────────────┬───────────────────────────┘
                  ▼
┌─────────────────────────────────────────────┐
│        OVERLAP-ADD                          │
│  ─ Add 50% overlap with previous block     │
│  ─ TDAC cancels aliasing                   │
└─────────────────┬───────────────────────────┘
                  ▼
┌─────────────────────────────────────────────┐
│        OUTPUT PCM                           │
│  ─ 16/24-bit signed integer or float        │
│  ─ Matches input sample rate                │
└─────────────────────────────────────────────┘
```

### 5.3 Inverse Quantization

The inverse quantizer reconstructs the magnitude of spectral coefficients from their Huffman-decoded quantized values `Y[k]`. Since the forward quantizer applied `Y = sign(X) * |X|^0.25`, the inverse is:

```
|X'[k]| = |Y[k]|^4
X'[k] = sign(Y[k]) * |Y[k]|^4
```

For `Y[k] = 0`, the output is zero. For large values encoded via escape sequences, the decoded value is the direct binary representation (which represents the 4th-root of the original value, not the original value itself — so no further transformation is needed).

### 5.4 Inverse TNS

If TNS data is present in the current window, the decoder applies the inverse TNS filter. The filter order, direction, and quantized coefficients were transmitted as side information. The inverse filter reconstructs the original spectral shape that the TNS analysis filter removed:

```
R_reconstructed[k] = X'[k]  // from inverse quantizer
X''[k] = R_reconstructed[k] + sum(i=1 to order) { a[i] * X''[k-i] }
```

Where `a[i]` are the inverse of the transmitted prediction coefficients. This reconstructs the spectral envelope with the temporal shaping removed, leaving quantization noise that follows the original signal's temporal envelope.

### 5.5 IMDCT and Windowing

The inverse MDCT (IMDCT) transforms the spectral coefficients back to the time domain. For a block of `N/2` spectral coefficients, the IMDCT produces `N` time-domain samples:

```
y'[n] = (2/N) * sum(k=0 to N/2-1) { X''[k] * cos(pi/N * (n + 0.5 + N/2) * (k + 0.5)) }
```

The output `y'[n]` must be windowed and overlap-added with the previous block's output to achieve perfect reconstruction. The synthesis window function is typically the same shape as the analysis window used in the encoder, though the KBD window may have slightly different parameters.

### 5.6 Overlap-Add Reconstruction

The TDAC (Time-Domain Aliasing Cancellation) property of the MDCT/IMDCT pair ensures that when overlapping blocks are windowed and summed, the aliasing components cancel. For 50% overlap:

```
y_final[n] = y'[n] + y''[n + N/2]
```

Where `y'[n]` is the current block's windowed output and `y''[n + N/2]` is the second half of the previous block's windowed output (the part that overlapped with the current block).

For the first block, the previous block's overlap is zero. For blocks following a short-block sequence, the overlap is with the previous block's second half, which may itself be a short block.

**Window Switching:** When the window sequence changes (e.g., from ONLY_LONG to EIGHT_SHORT), the window shapes and sizes change. The decoder must apply the appropriate window for each block in the sequence. The LONG_START_SEQUENCE uses a long window; EIGHT_SHORT_SEQUENCE uses 8 short windows (each with its own window function); LONG_STOP_SEQUENCE uses a short window with overlap-add into the subsequent long window.

### 5.7 Output Reconstruction

After overlap-add, the decoder may apply post-processing:
- **De-emphasis:** If pre-emphasis was applied in the encoder.
- **Dithering:** For signals with very low amplitude in certain bands, adding a small amount of dither can reduce quantization artifacts from the inverse quantization process.
- **SBR Reconstruction (HE-AAC):** The SBR decoder synthesizes high-frequency content from the decoded baseband and the transmitted SBR parameters.
- **Parametric Stereo Reconstruction (HE-AAC v2):** If Parametric Stereo is used, the decoder synthesizes the stereo image from a mono downmix and the transmitted spatial parameters.

The final output is PCM audio at the decoded sample rate, in the configured bit depth (typically 16-bit for consumer applications, 24-bit for professional applications).

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Raw AAC (ADTS)

Raw AAC files with `.aac` extension use the ADTS (Audio Data Transport Stream) framing format. Each frame consists of an ADTS header followed by a raw data block. The format is defined in ISO/IEC 14496-3.

**Advantages of ADTS:**
- Simple, self-contained format
- Frame boundaries are explicit (via the header's frame length field)
- Compatible with low-complexity streaming parsers
- Frame-level random access is straightforward

**Disadvantages:**
- No metadata support (ID3 tags can be prepended, but this is non-standard)
- No timing information without external container
- No support for multiple audio streams or muxed content
- No muxing with video

**Typical File Structure:**
```
[Optional ID3v2 tag]
[ADTS Frame 1: 7-9 byte header + raw AAC data]
[ADTS Frame 2: 7-9 byte header + raw AAC data]
...
[ADTS Frame N: 7-9 byte header + raw AAC data]
```

### 6.2 MP4/M4A Container

MP4/M4A is the primary container for AAC audio. It provides a comprehensive framework for audio/video synchronization, metadata, scene description, and streaming.

**Advantages of MP4:**
- Complete timing and synchronization model
- Rich metadata support (iTunes-style atoms, iXML, etc.)
- Support for multiple audio streams and muxed audio+video
- Hint tracks for streaming (RTSP/HTTP)
- Fragmented MP4 for progressive streaming
- Sample-accurate seeking
- iTunes compatibility (for .m4a with specific metadata atoms)

**Atom Structure for AAC:**
- `ftyp`: File type brand identification
- `moov`: Movie container (metadata, timing, sample tables)
  - `mvhd`: Movie header (duration, timescale)
  - `trak`: Track container
    - `tkhd`: Track header (track ID, duration)
    - `mdia`: Media container
      - `mdhd`: Media header (sample rate, channel count)
      - `minf`: Media information
        - `smhd`: Sound media header
        - `stbl`: Sample table (decoding instructions, timing)
          - `stsd`: Sample description (mp4a box with AudioSpecificConfig)
          - `stts`: Time-to-sample mapping
          - `stsc`: Sample-to-chunk mapping
          - `stsz`: Sample size table
          - `stco`/`co64`: Chunk offset table
- `mdat`: Media data (raw AAC access units)

### 6.3 Other Containers

**3GP/3GPP:** Used for mobile content (3GPP release 4 and later). Uses a simplified atom structure derived from MP4. Supports AAC-LC and HE-AAC profiles.

**ADIF (AAC Data Interchange Format):** An alternative raw AAC format used primarily in DAB (Digital Audio Broadcasting). Unlike ADTS, ADIF uses a single header at the beginning of the file with stream configuration, followed by raw access units. This makes ADIF less suitable for random access but more efficient for broadcast applications where the stream configuration is fixed throughout.

**Matroska/WebM (MKA):** Matroska is a general-purpose container that supports AAC audio. The WebM subset restricts this to Vorbis and Opus. AAC in Matroska stores raw AAC access units in a Matroska cluster, with codec private data (AudioSpecificConfig) in the CodecPrivate element.

**MOV/QuickTime:** The original container format for AAC (introduced with iTunes 1.0). Essentially identical to MP4 in its atom structure for audio.

---

## 7. METADATA ARCHITECTURE

### 7.1 MP4 iTunes Metadata (iTunes-style Atoms)

Metadata in MP4 containers is stored as a hierarchy of boxes within the `moov` → `udta` (user data) atom. The iTunes metadata model defines a standard set of atom types, each with a 4-character code. The metadata atoms use a nested structure where each atom contains a data atom (`data`) holding the actual value.

```
moov
 └─ udta
     └─ meta
         └─ ilst
             ├─ ©nam (Title)
             │   └─ data (UTF-8 text)
             ├─ ©ART (Artist)
             │   └─ data (UTF-8 text)
             ├─ ©alb (Album)
             │   └─ data (UTF-8 text)
             ├─ ©day (Release Date)
             │   └─ data (UTF-8 text)
             ├─ ©cmt (Comment)
             │   └─ data (UTF-8 text)
             ├─ ©gen (Genre)
             │   └─ data (UTF-8 text)
             ├─ trkn (Track Number)
             │   └─ data (16-bit track, 16-bit total)
             ├─ disk (Disc Number)
             │   └─ data (16-bit disc, 16-bit total)
             ├─ ©wrt (Composer)
             │   └─ data (UTF-8 text)
             ├─ ©grp (Grouping)
             │   └─ data (UTF-8 text)
             ├─ covr (Cover Art)
             │   └─ data (image bytes: JPEG or PNG)
             ├─ aART (Album Artist)
             │   └─ data (UTF-8 text)
             ├─ ©prd (Producer)
             │   └─ data (UTF-8 text)
             ├─ ©lyr (Lyrics)
             │   └─ data (UTF-8 text)
             ├─ ©gen (Genre — numeric)
             │   └─ data (8-bit genre index)
             ├─ tmpo (BPM)
             │   └─ data (16-bit integer)
             ├─ cpil (Compilation)
             │   └─ data (8-bit boolean)
             ├─ tmpo (Beats-Per-Minute)
             │   └─ data (16-bit integer)
             ├─ ©too (Encoded By)
             │   └─ data (UTF-8 text)
             ├─ ©enc (Encoding Tool)
             │   └─ data (UTF-8 text)
             ├─ ---- (Freeform atoms)
             │   └─ mean (namespace)
             │   └─ name (key name)
             │   └─ data (value)
             └─ gnre (Genre — ID3-style numeric)
                 └─ data (8-bit genre index)
```

### 7.2 Supported Tag Fields

The following table summarizes the most commonly used MP4 metadata atoms:

| Atom | Content | Data Type | Example |
|---|---|---|---|
| `©nam` | Track title | UTF-8 text | "Bohemian Rhapsody" |
| `©ART` | Artist | UTF-8 text | "Queen" |
| `©alb` | Album | UTF-8 text | "A Night at the Opera" |
| `©day` | Release year | UTF-8 text | "1975" |
| `©gen` | Genre (text) | UTF-8 text | "Rock" |
| `©cmt` | Comment | UTF-8 text | "Live at Wembley" |
| `©wrt` | Composer | UTF-8 text | "Freddie Mercury" |
| `trkn` | Track number | (track, total) pair | (5, 12) |
| `disk` | Disc number | (disc, total) pair | (1, 1) |
| `covr` | Cover art | Binary JPEG/PNG | [image data] |
| `aART` | Album artist | UTF-8 text | "Queen" |
| `tmpo` | BPM | 16-bit integer | 72 |
| `cpil` | Compilation flag | 8-bit boolean | 1 |
| `©lyr` | Lyrics | UTF-8 text | [lyrics text] |
| `----` | Freeform custom | Variable | Custom atoms |
| `cprt` | Copyright | UTF-8 text | "(P) 1975" |
| `©pub` | Publisher/Label | UTF-8 text | "EMI" |
| `©enc` | Encoded by | UTF-8 text | "iTunes 12.3" |
| `soal` | Sort album | UTF-8 text | "Night at the Opera, A" |
| `soaa` | Sort album artist | UTF-8 text | "Queen" |
| `soar` | Sort artist | UTF-8 text | "Queen" |
| `soco` | Sort composer | UTF-8 text | "Mercury, Freddie" |
| `sonm` | Sort title | UTF-8 text | "Bohemian Rhapsody" |
| `isrf` | ISRC | UTF-8 text | "GBUM71502601" |
| `apID` | Apple Account ID | UTF-8 text | [account string] |
| `cnID` | iTunes Catalog ID | 32-bit integer | 1086851673 |
| `atID` | Album Artist ID | 32-bit integer | 32941 |
| `plID` | Playlist ID | 64-bit integer | 1000000001 |
| `geID` | Genre ID | 32-bit integer | 17 |
| `sfID` | Store Front ID | 32-bit integer | 143444 |
| `cmID` | Campaign ID | 64-bit integer | [campaign ID] |

### 7.3 Cover Art in MP4

Cover art is stored in the `covr` atom as raw binary image data. The image format is typically JPEG (most common) or PNG. Multiple images can be stored in a single `covr` atom, with different sizes (front cover, back cover, etc.).

**Covr Atom Structure:**
```
covr
 ├─ data[0]: [image/jpeg, binary JPEG data]
 ├─ data[1]: [image/png, binary PNG data]  (optional)
 └─ data[N]: ...                          (optional)
```

**Important constraints:**
- Cover art in MP4 must be stored as raw image data within the `covr` atom, NOT as a reference to an external file.
- The image type is determined by the first few bytes of the data: `0xFF 0xD8` for JPEG, `0x89 0x50 0x4E` for PNG.
- Many encoders write cover art as a freeform atom (`----:com.apple.iTunes:Cover`) instead of the native `covr` atom. This is technically non-standard and may not be recognized by all players.
- FFmpeg's native AAC encoder typically writes cover art correctly to the native `covr` atom when using the MP4 container.

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 Codec Identifiers

FFmpeg uses internal codec identifiers for each audio codec. For AAC, the relevant identifiers are:

| FFmpeg Codec Name | Codec ID | Description |
|---|---|---|
| `aac` | AV_CODEC_ID_AAC | Native AAC encoder/decoder |
| `aac_latm` | AV_CODEC_ID_AAC_LATM | AAC in LATM transport syntax |
| `aac_fixed` | AV_CODEC_ID_AAC_FIXED | Fixed-point AAC decoder |
| `libfdk_aac` | (external) | Fraunhofer FDK AAC library |
| `libfaac` | (external) | FAAC encoder (deprecated) |

The preferred encoder for quality is `libfdk_aac`, followed by FFmpeg's native `aac` encoder. The `libfaac` encoder is deprecated and produces lower quality output at equivalent bitrates.

### 8.2 FFmpeg Encoding — Full CLI Reference

#### 8.2.1 Basic Encoding

```bash
# Basic AAC-LC encoding at 128 kbps
ffmpeg -i input.wav -c:a aac -b:a 128k output.m4a

# AAC with explicit profile specification
ffmpeg -i input.wav -c:a aac -profile:a aac_low -b:a 192k output.m4a

# HE-AAC (AAC + SBR) at 64 kbps
ffmpeg -i input.wav -c:a aac -profile:a aac_he -b:a 64k output.m4a

# HE-AAC v2 (AAC + SBR + PS) at 48 kbps
ffmpeg -i input.wav -c:a aac -profile:a aac_he_v2 -b:a 48k output.m4a
```

#### 8.2.2 Quality-Based Encoding

```bash
# VBR encoding (target quality 2, range 0-10, lower = better)
ffmpeg -i input.wav -c:a aac -q:a 2 output.m4a

# ABR encoding (average bitrate)
ffmpeg -i input.wav -c:a aac -b:a 192k output.m4a

# High quality VBR
ffmpeg -i input.wav -c:a aac -q:a 1 output.m4a

# Transparent quality VBR (~200-256 kbps on complex music)
ffmpeg -i input.wav -c:a aac -q:a 0 output.m4a
```

#### 8.2.3 FDK AAC Encoding (Best Quality)

```bash
# FDK AAC-LC at 256 kbps VBR
ffmpeg -i input.wav -c:a libfdk_aac -vbr 4 output.m4a

# FDK AAC-LC at 128 kbps CBR
ffmpeg -i input.wav -c:a libfdk_aac -b:a 128k output.m4a

# FDK HE-AAC v1 at 64 kbps
ffmpeg -i input.wav -c:a libfdk_aac -profile:a aac_he -b:a 64k output.m4a

# FDK HE-AAC v2 at 48 kbps
ffmpeg -i input.wav -c:a libfdk_aac -profile:a aac_he_v2 -b:a 48k output.m4a
```

#### 8.2.4 Advanced Encoding Options

```bash
# Explicit sample rate
ffmpeg -i input.wav -c:a aac -ar 44100 -b:a 192k output.m4a

# Explicit channel layout
ffmpeg -i input.wav -c:a aac -ac 2 -b:a 192k output.m4a

# Force SBR (implicit with HE profiles)
ffmpeg -i input.wav -c:a aac -sbr 1 -profile:a aac_low output.m4a

# Explicit bitrate control
ffmpeg -i input.wav -c:a aac -b:a 256k -bufsize 256k -maxrate 256k output.m4a

# Metadata passthrough
ffmpeg -i input.wav -c:a aac -b:a 192k -movflags +use_metadata_tags output.m4a

# Embed cover art
ffmpeg -i input.wav -i cover.jpg -c:a aac -b:a 192k \
  -c:v copy -map 0:a -map 1:v output.m4a
```

#### 8.2.5 FFmpeg AAC Encoder Options (Native)

| Option | Type | Default | Description |
|---|---|---|---|
| `-b:a` | integer | 128k | Average bitrate |
| `-q:a` | float | 2.0 | VBR quality (0-10, lower=better) |
| `-profile:a` | string | aac_low | AAC profile: aac_low, aac_he, aac_he_v2, aac_ld, aac_eld |
| `-ar` | integer | 0 | Sample rate (0=input) |
| `-ac` | integer | 0 | Number of channels (0=input) |
| `-sbr` | integer | -1 | SBR (0=off, 1=on, -1=auto) |
| `-dspr` | integer | -1 | DSP (0=off, 1=on, -1=auto) |
| `-bufsize` | integer | auto | Bitrate buffer size |
| `-maxrate` | integer | auto | Maximum bitrate |
| `-movflags` | flags | none | MP4 container flags |

#### 8.2.6 FDK AAC Encoder Options

| Option | Type | Default | Description |
|---|---|---|---|
| `-b:a` | integer | 128k | Bitrate (CBR/ABR) |
| `-vbr` | integer | 0 | VBR quality (0-5, 0=CBR) |
| `-profile:a` | string | aac_low | Profile selection |
| `-aac_coder` | string | fast | Coding algorithm: fast, 2phase, twoloop |
| `-aac_ms` | integer | 1 | M/S stereo coding (0=off, 1=on) |
| `-aac_pns` | integer | 1 | PNS enable (0=off, 1=on) |
| `-aac_tns` | integer | 1 | TNS enable (0=off, 1=on) |
| `-eld_sbr` | integer | 0 | ELD SBR (for ELD profile) |

### 8.3 FFmpeg Decoding

```bash
# Decode AAC to WAV
ffmpeg -i input.aac -c:a pcm_s16le output.wav

# Decode and resample to 48 kHz
ffmpeg -i input.m4a -ar 48000 -c:a pcm_s16le output.wav

# Decode and convert to FLAC (transcode)
ffmpeg -i input.m4a -c:a flac output.flac

# Decode HE-AAC
ffmpeg -i input_he.m4a -c:a pcm_s16le output.wav

# Decode with explicit codec selection
ffmpeg -i input.m4a -c:a aac -c:a pcm_s16le output.wav

# Decode to float (high precision)
ffmpeg -i input.m4a -c:a pcm_f32le output.wav

# Decode multiple files
ffmpeg -i input1.m4a -i input2.m4a -map 0:a -map 1:a \
  -c:a pcm_s16le output1.wav -c:a pcm_s16le output2.wav
```

### 8.4 FFmpeg Metadata Handling

```bash
# Extract all metadata
ffprobe -show_format -show_streams input.m4a

# Extract metadata to JSON
ffprobe -print_format json -show_format -show_streams input.m4a

# Copy metadata from source to output
ffmpeg -i input.wav -i source.m4a -c:a aac -b:a 192k \
  -map_metadata 1 output.m4a

# Strip metadata
ffmpeg -i input.m4a -c:a copy -map_metadata -1 output.m4a

# Set specific metadata
ffmpeg -i input.wav -c:a aac -b:a 192k \
  -metadata title="Track Title" \
  -metadata artist="Artist Name" \
  -metadata album="Album Name" \
  -metadata date="2024" \
  output.m4a

# Extract cover art
ffmpeg -i input.m4a -an -c:v copy cover.jpg

# Embed cover art
ffmpeg -i input.wav -i cover.jpg -c:a aac -b:a 192k \
  -c:v mjpeg -disposition:v attached_pic output.m4a
```

### 8.5 Quality / Fidelity: Encoding Settings Decision Table

The following table provides guidance for selecting AAC encoding parameters based on target use case and quality requirements:

| Use Case | Profile | Bitrate | Quality Level | Notes |
|---|---|---|---|---|
| Mobile streaming | HE-AAC v2 | 32-48 kbps | Low | Acceptable for speech, poor for music |
| Mobile streaming | HE-AAC v1 | 48-64 kbps | Acceptable | Good for speech, fair for music |
| Mobile streaming | HE-AAC v1 | 64-96 kbps | Good | Viable for most music |
| Streaming (broadband) | AAC-LC | 96-128 kbps | Good | Transparent for most content |
| Streaming (broadband) | AAC-LC | 128-160 kbps | Very Good | Near-transparent |
| High quality archive | AAC-LC | 192-256 kbps | High | Transparent for nearly all content |
| Master quality archive | AAC-LC | 256-320 kbps | Very High | Considered transparent |
| Metadata verification ref | AAC-LC | 320 kbps | Reference | Nero AAC @ 256 kbps VBR as benchmark |

**Encoder Quality Ranking (at equivalent bitrates):**
1. **Fraunhofer FDK AAC** — Highest quality, best transparency at low bitrates. Not built into most FFmpeg binaries due to licensing.
2. **Nero AAC Encoder** — Historically considered the reference encoder. Produces excellent quality at medium-high bitrates. Available via `neroAacEnc` CLI tool.
3. **FFmpeg Native AAC** — Improved significantly in recent versions. Good quality at 128+ kbps. `q=0` VBR can approach Nero quality.
4. **Apple QuickTime/iTunes** — Produces high-quality AAC. Used in iTunes purchases.
5. **FAAC** — Deprecated encoder with lower quality than modern alternatives. Not recommended for new encodes.

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Random Access Points

AAC bitstreams support random access through two mechanisms:

**ADTS Streams:** Each ADTS frame begins with a sync word, making each frame independently searchable. For streams using only long windows (ONLY_LONG_SEQUENCE), each frame is independently decodable (IDR-like). For streams with short windows, random access points are limited to frames that start with a long window sequence — short-window sequences depend on the previous block's overlap for perfect reconstruction.

**MP4 Containers:** The sample table (`stbl`) provides byte offsets and timestamps for every sample, enabling efficient seeking to any frame. The decoder can start decoding from any sample boundary, though the output may contain a brief preroll artifact if the access point uses short windows. Many MP4 files include a `sync sample` table (`stss`) that explicitly marks the random access points.

### 9.2 Seeking Behavior by Container

**MP4/M4A:** Sample-accurate seeking is supported. The `stts` box provides the duration of each sample, and the `stco`/`co64` box provides byte offsets. The decoder begins outputting audio from the seek point, with potentially a few samples of preroll from the previous frame's overlap-add region.

**ADTS:** Seeking requires scanning for the next sync word and decoding the frame header to determine the frame length. This is O(n) in the worst case, but with a good sync search algorithm, practical seeking is achievable at reasonable speeds.

**Matroska (MKA):** Uses cluster-based seeking. Each cluster contains a blockgroup with a timestamp reference. Seeking is to the nearest cluster start, then decoding forward to the exact timestamp.

### 9.3 Gapless Playback

Gapless playback requires bit-exact concatenation of consecutive audio frames without introducing silence or overlap at track boundaries. AAC supports gapless playback when:
1. The encoder uses consistent window sequences (only long windows)
2. The decoder properly handles the overlap-add of the final block
3. The container provides accurate timing for the final sample
4. No encoder-specific padding or silence is inserted between frames

FFmpeg's native AAC decoder properly handles overlap-add, and the MP4 container accurately represents frame timing, enabling gapless playback. ADTS files may have framing inconsistencies that prevent gapless playback.

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

### 10.1 Latency

The inherent algorithmic latency of AAC depends on the window size and buffering:

- **AAC-LC (1024-sample MDCT):** Algorithmic delay = 1024 samples = ~23.2 ms at 44.1 kHz (the overlap-add requires the next block for complete reconstruction).
- **AAC-LD:** Reduced window size (480 samples) and modified buffering: ~20 ms end-to-end.
- **AAC-ELD:** Further optimized for low delay: ~15-20 ms with SBR.
- **HE-AAC:** SBR introduces additional buffering: ~40-50 ms total.
- **HE-AAC v2:** PS adds further processing: ~50-60 ms total.

These are algorithmic delays. Total system latency (input buffering + encoder + transmission + decoder + output buffering) is typically 2-5x higher.

### 10.2 Streaming Protocols

AAC is used in multiple streaming protocols:

**MPEG-2 TS (Transport Stream):** Classic broadcast transport. AAC audio is PES-packetized and multiplexed with video into MPEG-2 TS packets. Supports ADRIF for low-bitrate transport.

**HLS (HTTP Live Streaming):** Apple's streaming protocol. Segments contain MP4-wrapped AAC audio. The manifest (m3u8) specifies the codec parameters for each variant stream.

**DASH (Dynamic Adaptive Streaming over HTTP):** ISO standard for adaptive streaming. Uses MP4 or WebM containers. Supports HE-AAC and HE-AAC v2 profiles.

**LATM (Low-overhead Audio Transport Multiplex):** An alternative to ADTS framing, LATM packs multiple AAC frames into a single transport packet with shared configuration. Used in DVB-H and other broadcast applications where bandwidth efficiency is critical.

**RTMP (Real-Time Messaging Protocol):** Adobe's streaming protocol. AAC is supported for live streaming, using ADTS framing within RTMP packets.

### 10.3 Bitrate Adaptation

Adaptive bitrate streaming with AAC is typically implemented at the container/protocol level (HLS, DASH) rather than within the codec itself. However, the codec's variable bitrate (VBR) capability enables smooth quality variation within a single encoded stream.

Modern AAC encoders support:
- **CBR:** Fixed bits per frame. Simplest for streaming but inefficient.
- **ABR:** Targeted average bitrate with controlled per-frame variation.
- **VBR:** Quality-based encoding with variable bits per frame.
- **Peak-constrained VBR:** VBR with a peak bitrate cap, combining quality targeting with bandwidth control.

---

## 11. AAC PROFILES

### 11.1 AAC-LC (Low Complexity)

AAC-LC (Low Complexity) is the most widely deployed AAC profile. It provides the best balance of compression efficiency, encoder/decoder complexity, and compatibility. It is the baseline profile that all AAC decoders must support.

**Key Characteristics:**
- MDCT with 1024-sample long windows and 8×128-sample short windows
- TNS (Temporal Noise Shaping)
- M/S stereo coding
- Perceptual Noise Substitution (PNS)
- Huffman coding with 11 codebooks
- Bitrates from 8 kbps to 320 kbps per channel

**Typical Applications:** Digital music distribution (iTunes, Amazon Music), mobile music streaming, digital radio (DAB+), Blu-ray audio tracks, game audio, podcasting.

**Quality:** Transparent at 192-256 kbps for most music. Good quality at 128-160 kbps. Acceptable at 96-128 kbps for casual listening.

### 11.2 HE-AAC (SBR)

HE-AAC (High Efficiency AAC) combines AAC-LC with SBR (Spectral Band Replication). SBR is a parametric coding technology that reconstructs high-frequency content from a reduced baseband signal and transmitted parameters. This enables high-quality audio at roughly half the bitrate of AAC-LC.

**SBR Principle:** Instead of transmitting the full spectrum (e.g., 0-20 kHz), HE-AAC transmits only the baseband (e.g., 0-10 kHz) using AAC-LC, plus parametric side information that describes the high-frequency content (10-20 kHz). The SBR decoder analyzes the baseband's spectral envelope and uses this information to synthesize plausible high-frequency content.

**SBR Parameters Include:**
- Spectral envelope of the high-frequency bands (quantized energy per SBR band)
- Noise floor estimate (how much noise-like content is in the high frequencies)
- Tone map (which high-frequency components are tonal vs. noise-like)
- Transient detection flags
- Inverse filtering parameters

**Quality:** HE-AAC at 48-64 kbps approaches the quality of AAC-LC at 96-128 kbps. At very low bitrates (32-48 kbps), the parametric nature of SBR may introduce audible artifacts, particularly on complex music with sustained high-frequency content (cymbals, string sections).

**Compatibility:** HE-AAC decoders must decode both HE-AAC (profile 5 in AudioSpecificConfig) and AAC-LC (profiles 1-2). The SBR data is carried in the Fill Element (ID_FIL) within the AAC frame.

### 11.3 HE-AAC v2 (SBR + PS)

HE-AAC v2 extends HE-AAC with PS (Parametric Stereo). PS is a further bitrate reduction technique that encodes stereo as a mono downmix plus parametric stereo description. This enables near-stereo quality at extremely low bitrates.

**PS Principle:** Instead of encoding two independent (or M/S coded) stereo channels, HE-AAC v2 encodes:
1. A mono downmix of the stereo signal
2. Parametric stereo parameters describing the stereo image:
   - Inter-channel level difference (ICLD) per frequency band
   - Inter-channel correlation (ICC) per frequency band
   - Phase difference information (optional, higher bitrates)

The PS decoder reconstructs stereo from the mono signal using these parameters.

**Quality:** HE-AAC v2 at 32-48 kbps can provide acceptable stereo quality for many listeners, though audiophiles will notice the parametric reconstruction. At 16-24 kbps, it is primarily suitable for speech.

**Compatibility:** HE-AAC v2 decoders must support all lower profiles (AAC-LC, HE-AAC, and PS). HE-AAC v1 decoders that encounter PS data will decode the mono downmix and ignore the stereo parameters.

### 11.4 AAC-LD (Low Delay)

AAC-LD (Low Delay AAC) was designed for applications requiring minimal end-to-end latency, such as video conferencing, intercom systems, and live broadcast monitoring. It achieves reduced latency through modifications to the transform window size and buffering strategy.

**Key Modifications:**
- Reduced MDCT window size (512 samples, reduced from 1024)
- Modified frame structure with shorter lookahead
- Simplified psychoacoustic model
- Optional reduced block length (480 samples)
- No PNS (which requires additional buffering)

**Latency:** ~20 ms algorithmic delay, compared to ~23+ ms for standard AAC-LC. Total system latency is typically 40-60 ms.

**Quality Trade-off:** The reduced window size means coarser frequency resolution, which can lead to lower compression efficiency and slightly more audible pre-ringing artifacts at equivalent bitrates compared to AAC-LC. AAC-LD typically requires 20-30% higher bitrate to achieve equivalent perceptual quality.

### 11.5 AAC-ELD (Enhanced Low Delay)

AAC-ELD combines AAC-LD with SBR, providing low-latency encoding with improved efficiency through spectral band replication. It is the preferred profile for real-time communication applications that also require good audio quality at moderate bitrates.

**Key Characteristics:**
- AAC-LD core (512/480 sample windows)
- SBR operating at low delay mode
- Full bandwith reconstruction from reduced baseband
- Typical latency: 15-32 ms (depending on configuration)
- Bitrates: 32-128 kbps

**Quality:** AAC-ELD at 64 kbps provides quality comparable to AAC-LC at 96-128 kbps, with substantially lower latency. This makes it ideal for:
- VoIP and video conferencing (where lip-sync is critical)
- Live broadcast audio monitoring
- Professional in-ear monitoring systems
- Two-way communication applications

**Configuration:** AAC-ELD supports two modes:
- **ELD SBR:** Standard SBR integrated with the low-delay core
- **ELD without SBR:** AAC-LD-like behavior at higher bitrates

### 11.6 Profile Comparison Table

| Characteristic | AAC-LC | HE-AAC v1 | HE-AAC v2 | AAC-LD | AAC-ELD |
|---|---|---|---|---|---|
| Base Codec | AAC-LC | AAC-LC + SBR | AAC-LC + SBR + PS | AAC-LC (modified) | AAC-LD + SBR |
| MDCT Size | 1024 | 1024 (core) | 1024 (core) | 512/480 | 512/480 (core) |
| Frequency Range | Full | Full (reconstructed) | Full (reconstructed) | Full | Full (reconstructed) |
| Stereo Support | L/R, M/S, Inten. | L/R, M/S, Inten. | Mono + PS | L/R, M/S | L/R, M/S |
| Min Bitrate (stereo) | 64 kbps | 32 kbps | 16 kbps | 64 kbps | 32 kbps |
| Typical Bitrate | 128-256 kbps | 64-128 kbps | 32-64 kbps | 96-192 kbps | 48-96 kbps |
| Algorithmic Latency | ~23 ms | ~40 ms | ~50 ms | ~20 ms | 15-32 ms |
| PNS Support | Yes | Yes | Yes | No | No |
| TNS Support | Yes | Yes | Yes | Yes | Yes |
| Decoder Complexity | Low | Medium | Medium-High | Medium | Medium |
| Primary Use Case | Music, streaming | Mobile streaming | Ultra-low bitrate | Conferencing | VoIP, monitoring |
| iTunes Compatibility | Yes | Yes | Yes (mono) | Limited | Limited |
| DAB+ Support | Yes | Yes | Limited | No | No |
| 3GPP Support | Yes | Yes | Yes | Yes | Yes |

---

## 12. TRANSPARENT BITRATE & QUALITY

### 12.1 Transparent Bitrate

Transparency in perceptual audio coding means that the compressed output is perceptually indistinguishable from the original. For AAC-LC, transparency thresholds have been established through extensive listening tests:

**AAC-LC Transparency:**
- **Most listeners, most content:** 192-256 kbps
- **Critical listeners, all content:** 256-320 kbps
- **Average listeners, simple content:** 128-160 kbps

The actual bitrate required for transparency depends on:
- **Source complexity:** Orchestral music requires more bits than solo piano; electronic music with sibilants is challenging at any bitrate.
- **Listener sensitivity:** trained audiophiles can detect artifacts at higher bitrates than casual listeners.
- **Playback equipment:** High-fidelity playback chains reveal artifacts that are masked on consumer equipment.
- **Listening environment:** Controlled listening rooms reveal artifacts that are inaudible in noisy environments.

**HE-AAC Transparency:**
- **HE-AAC v1:** 64-96 kbps for most content, though some listeners report differences at this level.
- **HE-AAC v2:** 48-64 kbps for acceptable quality; transparency is not achieved even at 96 kbps due to PS limitations.

### 12.2 Listening Tests

Multiple independent listening tests have established AAC quality benchmarks:

**TTCD (Telecommunication Technology Association):** Concluded that HE-AAC at 48 kbps is perceptually equivalent to MP3 at 128 kbps, representing approximately 62% bitrate reduction.

**EPSMA (European Telecommunications Standards Institute):** Found that HE-AAC at 64 kbps provides quality comparable to MP3 at 128 kbps.

**Spirit of Engineering Listening Tests (multiple independent):** Consistently found that FDK AAC at 192-256 kbps VBR is indistinguishable from lossless on most program material. The Nero AAC encoder at 256 kbps VBR was considered the reference transparent encoder.

**Hydrogenaudio Listening Tests:** Community-driven tests at hydrogenaudio.org have extensively compared AAC encoders. Key findings:
- FDK AAC with VBR quality 4 (approximately 190-210 kbps) achieves transparency on 95%+ of program material.
- FFmpeg native AAC at `q=0` (VBR, very high quality) approaches Nero AAC quality but with slightly lower efficiency.
- HE-AAC v2 at 64 kbps is adequate for casual listening but not for critical listening.

### 12.3 Artifact Description

Understanding AAC encoding artifacts helps identify encoding quality issues:

**Pre-Ringing:** A faint echo preceding transient attacks. Most audible on percussion, plucked strings, and consonants in vocal recordings. Caused by the MDCT's spectral leakage in the time domain. More pronounced with long windows; short windows reduce but don't eliminate it. TNS helps mitigate pre-ringing by temporally shaping the noise.

**Post-Ringing:** Spectral leakage following transients. Less perceptually problematic than pre-ringing due to backward masking. Controlled by window shape and psychoacoustic model aggressiveness.

**Tonal Collapse:** At very low bitrates, AAC may merge multiple tonal components into fewer frequency bins, creating a听的, undifferentiated tonal quality. The Huffman coding of quantized coefficients loses the fine structure of the spectrum.

**Birdie Artifacts:** High-pitched, tonal artifacts that appear as small isolated whistles or chirps. Caused by the psychoacoustic model misclassifying noise as tonal, or vice versa, at bitrates near the transparency threshold. Characteristic of aggressive encoding.

**PNS Artifacts:** When PNS replaces a partially tonal band with synthetic noise, the transition between the original tonal component and the synthetic noise may be audible as a slight change in texture or timbre. Also, perfectly stationary noise can sound unnatural compared to natural noise that has slight temporal variation.

**SBR Artifacts in HE-AAC:** The parametric high-frequency reconstruction can introduce:
- **Metallic quality:** Overly bright or harsh high frequencies due to envelope misestimation
- **Temporal smearing:** Loss of high-frequency transients due to envelope smoothing
- **Spectral Ducking:** Low-frequency energy causing the SBR envelope to underestimate high-frequency content
- **Banding:** Distinct horizontal bands in the spectrogram of high-frequency content

**PS Artifacts in HE-AAC v2:** Parametric stereo reconstruction can introduce:
- **Center Channel Leakage:** A mono sum appears too wide or too narrow
- **Unnatural Stereo Width:** Either unnaturally wide (phase artifacts) or overly narrow (excessive inter-channel coupling)
- **Spatial Discontinuities:** Sudden changes in stereo image when PS parameters change between frames

---

## 13. KNOWN ISSUES & EDGE CASES

### 13.1 Encoder-Specific Issues

**FFmpeg Native AAC:**
- The native encoder's VBR mode (`-q:a`) can produce files that are incompatible with some hardware decoders (particularly older Sony, Samsung, and Nintendo devices). CBR encoding produces wider compatibility.
- At very low bitrates (under 64 kbps), the native encoder may produce audible birdie artifacts more frequently than FDK AAC.
- The default encoding mode (VBR with `-q:a 2`) may not produce consistent quality across different source material.

**FAAC:**
- Produces noticeably lower quality output than modern encoders at equivalent bitrates.
- Does not support HE-AAC profiles.
- Deprecated and unmaintained.
- Should not be used for new encodes.

**FDK AAC:**
- Requires FFmpeg to be compiled with `--enable-libfdk-aac`. Many distribution-provided FFmpeg builds do not include it due to licensing concerns.
- The VBR mode (`-vbr 4`) produces very high quality but variable file sizes.
- Older FDK library versions have known bugs with specific sample rate/channel configurations.

**Nero AAC:**
- The Nero AAC encoder is no longer actively maintained.
- Available only as a Windows GUI or CLI tool; no source code available for integration.
- Considered the reference encoder for high-quality AAC, but increasingly replaced by FDK AAC in open-source pipelines.

### 13.2 Container Issues

**Metadata Atom Order:** Some hardware decoders (particularly automotive and portable devices) are sensitive to the order of metadata atoms within the MP4 file. The `moov` atom should ideally precede the `mdat` atom. Use `ffmpeg -movflags +faststart` to move the moov atom to the beginning of the file for web streaming.

**Cover Art Handling:** Several issues arise with cover art in MP4:
- Some encoders write cover art as a freeform atom (`----:com.apple.iTunes:Cover`) instead of the native `covr` atom, making it invisible to many players.
- The `covr` atom must contain valid JPEG or PNG data; corrupted or truncated image data causes playback failure on some devices.
- Multiple images in a single `covr` atom may not be handled consistently across implementations.

**M4A File Size Limits:** Some very old hardware players have file size limitations (2 GB or 4 GB). Modern MP4/M4A files use 64-bit `co64` chunk offsets to support files larger than 4 GB.

**DRM (iTunes Plus vs. FairPlay):** Files purchased from iTunes before 2009 used FairPlay DRM. Modern iTunes Plus files are DRM-free AAC-LC at 256 kbps VBR. FFmpeg can decode both (DRM removal requires separate tools).

### 13.3 Bitstream Edge Cases

**Short Window Sequences:** Files using only EIGHT_SHORT_SEQUENCE windows without proper LONG_START_SEQUENCE and LONG_STOP_SEQUENCE transitions may produce artifacts at block boundaries. Some decoders are more robust to malformed window sequences than others.

**Zero-Length Frames:** At extremely low bitrates or with very quiet input, AAC encoders may produce frames with no spectral data. Decoders must handle these gracefully.

**Invalid Scale Factors:** If scale factors exceed the valid range (0-255 for AAC-LC), the decoder may produce NaN or infinity values during inverse quantization. FFmpeg's decoder handles this gracefully; some hardware decoders may crash.

**Mixed Block Types:** Streams that alternate between long and short windows without the proper transition sequence (LONG_START_SEQUENCE before first short, LONG_STOP_SEQUENCE after last short) may cause imperfect reconstruction on some decoders.

---

## 14. CONVERSION GUIDE (DBpoweramp Context)

### 14.1 Recommended Conversion Settings

For transcoding to AAC/M4A from any source format using FFmpeg:

**High Quality (Recommended):**
```bash
ffmpeg -i input.flac -c:a aac -profile:a aac_low -q:a 1 -ar 44100 -ac 2 \
  -movflags +faststart output.m4a
```

This produces VBR AAC-LC at approximately 192-256 kbps, which is transparent for nearly all music content. The `-movflags +faststart` optimizes the file for web streaming by moving the moov atom to the beginning.

**Standard Quality:**
```bash
ffmpeg -i input.flac -c:a aac -profile:a aac_low -b:a 192k -ar 44100 -ac 2 \
  -movflags +faststart output.m4a
```

CBR 192 kbps provides consistent quality and maximum compatibility with hardware players.

**FFDK AAC (Best Quality, if available):**
```bash
ffmpeg -i input.flac -c:a libfdk_aac -vbr 4 -ar 44100 -ac 2 \
  -movflags +faststart output.m4a
```

VBR quality 4 with FDK AAC produces the highest quality AAC output available from open-source encoders.

### 14.2 Passthrough Verification

After AAC encoding, verify metadata preservation using `kid3-cli` per the project metadata verification rules:

```bash
kid3-cli -c "get" "source.flac"
kid3-cli -c "get" "output.m4a"
```

Check for:
- All standard tags preserved (Title, Artist, Album, Date, Genre, Track, Disc)
- All freeform atoms preserved with exact names and values
- Cover art (covr) present as native Picture atom
- No `Invalid UTF-8` or `Ignoring duplicate atom` errors
- Sort atoms (soan, soaa, soal, soco) preserved as native atoms

### 14.3 Known Conversion Pitfalls

**Bitrate Selection for Transcoding:** When transcoding from a lossy source (e.g., MP3 at 128 kbps), encoding at 256 kbps AAC does not recover lost quality — only the information that survived the MP3 encoding can be encoded. Transcoding from MP3 to AAC at any bitrate will produce a result no better than the original MP3.

**Dithering on Low-Level Sources:** When converting from high-resolution (24-bit) source to AAC, the 16-bit default output may benefit from proper dithering. FFmpeg's native AAC encoder operates internally at 32-bit float precision, so dithering is applied during the float-to-output conversion.

**Sample Rate Conversion:** If the output must be at a specific sample rate (e.g., 44.1 kHz from a 96 kHz source), FFmpeg applies high-quality sample rate conversion (Sinc-based by default). The AAC encoder then operates at the output sample rate.

---

## 15. REFERENCE IMPLEMENTATIONS & LIBRARIES

### 15.1 Open-Source Implementations

**FFmpeg/libavcodec:** The most complete open-source AAC implementation. Includes both encoder (native) and decoder for all AAC profiles. Actively maintained. The native encoder quality has improved substantially since FFmpeg 4.0.

**Fraunhofer FDK AAC (libfdk_aac):** Considered the best-quality open-source AAC encoder. Provides CBR, ABR, and VBR modes. Supports AAC-LC, HE-AAC, HE-AAC v2, AAC-LD, and AAC-ELD. Available in FFmpeg when compiled with `--enable-libfdk-aac`. Requires Fraunhofer's AAC patent license for commercial use.

**FAAC:** An older open-source AAC encoder. Produces lower quality output than modern alternatives. Deprecated and unmaintained. Should not be used for new encodes.

**FAAD2:** Open-source AAC decoder. Supports all AAC profiles including HE-AAC and HE-AAC v2. Used as a reference decoder in many testing scenarios.

**MPEG reference software:** ISO provides reference implementations of AAC encoding and decoding in C, available from the MPEG website. These are educational references but not optimized for production use.

### 15.2 Commercial Implementations

**Apple (Apple Lossless Utility / AudioToolbox):** Apple's hardware-accelerated AAC encoder is used in macOS and iOS for encoding audio content. Available via the AudioToolbox framework. Produces high-quality output with good hardware decoder compatibility.

**Fraunhofer AAC Encoder:** The commercial reference implementation. Available as a library for licensing. Used in professional audio production tools, streaming servers, and broadcast equipment.

**Nero AG AAC Encoder:** The Nero AAC codec was widely considered the best-quality commercial encoder. Now discontinued and superseded by the Fraunhofer library. Available as a free download for encoding; decoder included in many multimedia software packages.

**Dolby (Dolby Digital Plus):** While technically based on AC-3 rather than AAC, Dolby Digital Plus (E-AC-3) is related and provides similar capability with additional features. Used in Blu-ray and streaming applications.

### 15.3 Hardware Support

**Chip-level support:** Most modern SoCs include hardware AAC decode support:
- Apple A-series and M-series chips: Hardware AAC decode/encode at low power
- Qualcomm Snapdragon: Hardware AAC decode
- ARM NEON: Optimized software decode fallback
- Intel/AMD x86: SSE-accelerated decode

**Hardware decoder compatibility considerations:** While nearly all modern hardware supports AAC-LC decoding, HE-AAC v2 support varies significantly on older devices. Some budget Android devices and older car stereos may decode HE-AAC v1 but fail on HE-AAC v2.

---

## 16. RELEVANT SPECIFICATIONS & FURTHER READING

### 16.1 Core Specifications

**ISO/IEC 14496-3:2009:** The definitive specification for MPEG-4 Audio, including all AAC profiles, tools, and bitstream syntax. This is the primary reference document for any AAC implementation.

**ISO/IEC 14496-12:2015:** ISO Base Media File Format specification. Defines the MP4 container structure, atom hierarchy, and file format rules.

**ISO/IEC 14496-14:2003:** MP4 file format for MPEG-4 content. Specifically defines the use of the ISO Base Media File Format for MPEG-4 audio content, including M4A files.

**ISO/IEC 13818-7:2006:** MPEG-2 AAC specification. The original AAC standard, now superseded by MPEG-4 but still referenced for ADTS bitstream format details.

**RFC 6381:** "The 'codecs' and 'profile' parameters for HTML video." Defines the codec parameter string format used in SDP and HTML5 video/audio elements for specifying AAC profiles.

### 16.2 Standards Bodies and Reference Material

**MPEG (Moving Picture Experts Group):** The working group responsible for developing MPEG-2 AAC, MPEG-4 Audio, and related standards. Official specifications available from ISO or through national standards bodies.

**ITU-R BS.1116:** Methods for the subjective assessment of small impairments in audio systems. The reference methodology for listening tests that establish transparency thresholds.

**ITU-R BS.1534:** Method for the subjective assessment of intermediate quality levels of audio systems. Defines the MUSHRA (MUlti Stimulus test with Hidden Reference and Anchor) methodology used in most modern audio codec listening tests.

**EBU Tech 3276:** Listening conditions for the subjective assessment of audio quality. Defines the standard listening room specifications used in European broadcast quality testing.

**EBU R128:** Loudness normalisation and permitted maximum level of audio signals. While not specific to AAC, relevant for audio level normalization in broadcast workflows that distribute AAC-encoded content.

### 16.3 Academic and Technical References

**Bos, M., & Quinn, B. (2005).** "Audio Coding." In I.Inf. (Ed.), Wiley Encyclopedia of Electrical and Electronics Engineering. A comprehensive overview of perceptual audio coding principles.

**Vinton, M., et al. (2005).** "Distortionless Audio Coding for Streaming and Converged Applications." AES Convention Paper. Discusses AAC-LD and AAC-ELD for real-time communication.

**Noll, P. (1997).** "MPEG Digital Audio Coding." IEEE Signal Processing Magazine. A foundational paper on MPEG audio coding, including AAC.

**Brandenburg, K., & Sporer, T. (1992).** "NMR and Masking Flag: Evaluation of Quality Using Perceptual Criteria." AES Convention. The perceptual evaluation methodology used in AAC development.

**ISO/IEC JTC 1/SC 29/WG 11 (MPEG):** Working documents, meeting reports, and reference software available through the MPEG website and various mirror archives. These provide historical context for design decisions in AAC.

---

## 17. IMPLEMENTATION CHECKLIST (for the Converter Developer)

### 17.1 Encoder Integration Checklist

- [ ] Select appropriate AAC profile based on target use case:
  - [ ] AAC-LC for general music at 128+ kbps
  - [ ] HE-AAC for mobile streaming at 48-96 kbps
  - [ ] HE-AAC v2 for voice-only at 16-48 kbps
  - [ ] AAC-ELD for real-time communication
- [ ] Configure bitrate control mode:
  - [ ] VBR for best quality-per-bitrate
  - [ ] ABR for predictable file sizes
  - [ ] CBR for maximum hardware compatibility
- [ ] Verify encoder version and configuration
- [ ] Test with problematic source material (transients, sibilants, orchestral)
- [ ] Benchmark against reference encoders (Nero, FDK AAC)

### 17.2 Container Integration Checklist

- [ ] Use MP4/M4A container for output (not raw ADTS)
- [ ] Embed AudioSpecificConfig correctly in esds box
- [ ] Set proper file type brand (M4A for audio-only)
- [ ] Include all required sample table boxes (stsd, stts, stsc, stsz, stco)
- [ ] Use moov-before-mdat ordering (`-movflags +faststart`)
- [ ] Set correct channel count and sample rate in mdhd
- [ ] Verify AudioSpecificConfig matches actual stream parameters

### 17.3 Metadata Integration Checklist

- [ ] Copy all standard tags from source to output
- [ ] Write native atoms (©ART, ©nam, ©alb, etc.) for standard fields
- [ ] Preserve freeform atoms (----:*) with exact original names
- [ ] Copy cover art to native `covr` atom (binary, not freeform)
- [ ] Handle trkn/disk as pair atoms, not as freeform strings
- [ ] Verify sort atoms (soan, soal, soaa, soco) are preserved
- [ ] Strip duplicate or conflicting metadata entries
- [ ] Validate UTF-8 encoding for all text fields
- [ ] Run kid3-cli verification after encoding

### 17.4 Verification Checklist

- [ ] Decode output and verify PCM matches expected sample rate
- [ ] Verify channel count and layout
- [ ] Check bitrate matches configured target
- [ ] Test playback on reference hardware/software players
- [ ] Verify gapless playback on consecutive tracks (MP4 container)
- [ ] Test seeking accuracy (random access points)
- [ ] Run kid3-cli comparison: source vs output metadata
- [ ] Verify no `Invalid UTF-8` in kid3 output
- [ ] Verify no `Ignoring duplicate atom` warnings
- [ ] Verify all freeform atoms from source appear in output with exact names
- [ ] Verify cover art appears as `covr` Picture atom, not freeform text
- [ ] Verify numeric IDs (atID, plID) present if source had them
- [ ] Verify Publisher (©pub) present if source had it

### 17.5 Quality Assurance Checklist

- [ ] Conduct listening tests at target bitrate with diverse program material
- [ ] Compare output with reference encoder (Nero/FDK AAC) at same bitrate
- [ ] Test edge cases: silence, pure tones, complex transients, extreme stereo
- [ ] Verify HE-AAC SBR reconstruction quality on reference decoder
- [ ] Verify HE-AAC v2 Parametric Stereo quality (if applicable)
- [ ] Test compatibility with target playback devices (iOS, Android, desktop)
- [ ] Measure encoding speed (real-time factor) for batch processing viability
- [ ] Monitor memory usage during batch encoding

---

*Document Version: 1.0*
*Last Updated: June 2026*
*Target Audience: Audio converter developers, codec engineers, streaming infrastructure teams*
