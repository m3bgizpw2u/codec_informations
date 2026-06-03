# Musepack (MPC) — Deep Technical Reference
> **Category:** Lossy
> **File Extensions:** `.mpc`, `.mp+`, `.mpp`
> **MIME Types:** `audio/x-musepack`, `audio/musepack`
> **Standardization Body:** Open source / community-developed (no formal ISO/IEC standardization)
> **Primary Specification:** Musepack SV8 specification (public), SV7 specification (public)
> **Patent Status:** Patent-free — the format is open and unencumbered by known patents
> **License:** Open source, free for all uses
> **Current Version:** SV8 (StreamVersion 8), finalized August 10, 2011
> **Active Development:** Minimal — format is stable; bug fixes only

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Andree Buschmann, Frank Klemm, and the open-source Musepack community
- **Year Created:** 1997–1998 (originally as MPEG Plus / MPEG+)
- **Original Purpose:** An open-source, patent-free perceptual audio codec optimized for transparent encoding of stereo audio at bitrates around 160–180 kbps — addressing the patent and licensing concerns of MP3 at the time
- **Problem with Predecessors:** MP3 was encumbered by patents and licensing fees. AAC was also patented. Vorbis was in development but early. Musepack (originally "MPEG+") aimed to be a fully open alternative that could achieve transparency at lower bitrates than MP2 while remaining patent-free.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| SV4 (StreamVersion 4) | 1998 | Initial release, MPEG+ codec |
| SV5 (StreamVersion 5) | 1999 | Improved psychoacoustic model |
| SV6 (StreamVersion 6) | 2001 | Further quality improvements |
| SV7 (StreamVersion 7) | ~2004 | Huffman coding refinement, APEv2 tagging, widely used |
| SV8 (StreamVersion 8) | 2011 | Container-independent, packetized, sample-accurate seeking, chapters, cleaned up bitstream |

### 1.3 Current Adoption
- **Primary use cases today:** Archival of high-quality lossy audio, niche audio community, game audio (historically), personal music collections
- **Platforms with native support:** Limited — no native OS support; requires third-party decoders (foobar2000, Winamp, VLC, FFmpeg)
- **Major services using this format:** None commercial; niche community use
- **Hardware support:** Very limited — some high-end DAPs with open-source codec support; not mainstream
- **Status:** Niche / legacy — surpassed by more widely-supported formats (AAC, Opus) but respected in the audiophile community for its high quality

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Perceptual audio coder / subband transform hybrid
- **Core algorithm:** MPEG-1 Audio Layer II (MP2) derivative with hybrid filter bank + Huffman coding + additional psychoacoustic optimizations
- **Loss mechanism:** Psychoacoustic masking — based on MP2's subband quantization with improved bit allocation, additional noise substitution for bands below threshold, and Huffman entropy coding for improved compression
- **Frame-based vs sample-based:** Frame-based (1152 samples per frame, identical to MP2)
- **Fixed vs variable frame size:** Variable bitrate — frames are variable in size; quality-based (not bitrate-based) encoding

### 2.2 High-Level Encoding Flow (Block Diagram)
```
Input PCM Samples (variable count per frame, ~1152 base)
      │
      ▼
[Pre-processing: Subband filtering via 32-band QMF polyphase filter bank]
      │
      ▼
[Hybrid Transform: Additional MDCT on selected subbands for better frequency resolution]
      │
      ▼
[Psychoacoustic Analysis: Per-subband masking threshold computation]
      │
      ▼
[Bit Allocation: Mask-to-noise optimization per subband, improved over MP2]
      │
      ▼
[Quantization: Uniform quantization per subband with block companding (scalefactors)]
      │
      ▼
[Huffman Coding: Canonical Huffman tables (vector Huffman, escape codes)]
      │
      ▼
[Noise Substitution: Substitute noise below masking threshold with energy-allocated bits]
      │
      ▼
[Stereo Processing: MS stereo, intensity stereo, channel coupling]
      │
      ▼
[Bitstream Packing: Variable-length frames with packet headers]
      │
      ▼
Output Encoded MPC Bitstream
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | ~1000 samples / ~23 ms | Higher than MP2 due to hybrid filter bank |
| Frame size | 1152 samples (base) | Variable; frames may span multiple blocks |
| Max channels | 2 (stereo) | Mono supported in decode |
| Max bit depth | 16-bit input | Internal 32-bit floating-point precision |
| Max sample rate | 48,000 Hz | SV7/SV8 support up to 48 kHz |
| Bitrate range | ~120–320 kbps | Quality-based; typical transparency ~175–185 kbps |
| Complexity | O(N log N) per sample | Higher than MP2 due to hybrid approach |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes

**SV8 format (`.mpc` files from 2011+):**
```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       4D 50 43 4B     MPCK     SV8 file signature ("MPCK")
0x0004  1       —                —        SH packet type (Stream Header)
...
```

**SV7 format (`.mp+` / older `.mpc`):**
```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       4D 50 2B        MP+      SV7 file signature
0x0004  4       —                —        Version info + song metadata
...
```

### 3.2 File-Level Header Layout

**SV8 Packet Structure:**
```
Offset   Size    Field Name              Type        Valid Range   Description
-------  ------  ----------------------  ----------  -----------   ---------------------------
0x0000   4B      MPCK Magic             char[4]     "MPCK"       SV8 stream signature
Packet header:
  Key:   2B      Packet Key            string      "SH"/"RG"/"EI"/"SO"/"AP"/"ST"/"CT"/"SE"
  Size:  variable Size Field             vbv         variable    Packet size in bytes

Stream Header Packet (SH):
  [0]    4B      CRC-32                uint32     0 or valid   CRC-32 of packet (0=invalid)
  [4]    1B      Stream Version        uint8      8            Must be 8
  [5]    8B      Sample Count          varint     0–2^63       Total audio samples
  [13]   1B      Flags                 uint8      bitfield     Sample rate, channels, etc.
  [14]   1B      Max Band              uint8      0–31         Highest coded subband
  [17]   4B      Sample Rate           uint32     44100,48000,37800,32000  Samples per second
  [21]   1B      Channels              uint8      1–2          1=mono, 2=stereo
  [22]   1B      Flags                 uint8      bitfield     MS stereo, etc.
```

**SV7 Header Structure:**
```
Offset   Size    Field Name              Type        Valid Range   Description
-------  ------  ----------------------  ----------  -----------   ---------------------------
0x0000   4B      Signature               char[4]     "MP+"         SV7 signature
0x0004   4B      Version                 uint32 LE   7.0–7.15     Version number
0x0008   4B      Header Size            uint32 LE   0x0024        Header size (36 bytes)
0x000C   4B      Frame Count            uint32 LE   variable      Number of frames
0x0010   4B      Nav Info Offset        uint32 LE   variable      Seek table location
0x0014   4B      Nav Info Size          uint32 LE   variable      Seek table size
0x0018   4B      Free Bytes            uint32 LE   variable      Free format space
0x001C   2B      Sample Frequency       uint16 LE   44100,48000   Sample rate
0x001E   1B      Max Band               uint8      0–31         Highest coded subband
0x001F   1B      Channels               uint8      1–2          1=mono, 2=stereo
0x0020   2B      Emphasis               uint16 LE   0            Not used
0x0022   2B      Reserved              uint16 LE   0            Reserved
```

### 3.3 Frame / Block Header Layout

**SV8 Audio Packet (AP):**
```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------
[Key]    2B      Packet Key             "AP"         Audio packet identifier
[Size]   n       Size Field              variable     Packet size
[Payload]         Audio Data             bits         Huffman-coded subband samples
```

**SV7 Frame Structure:**
```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------
[0]      variable Frame Header           bits        Key, size, data length
[Data]   variable Audio Data             bits        Huffman-coded subband samples
```

### 3.4 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Unsigned integer | No | Not defined |
| 16-bit | Signed integer | Yes | Standard input/output |
| 20-bit | Signed integer | Partial | Encoder may quantize to 16-bit internally |
| 24-bit | Signed integer | No | Not supported by standard encoder |
| 32-bit | Signed integer | No | Not supported |
| 32-bit | IEEE float | Partial | Input accepted but quantized to 16-bit |

#### Supported Sample Rates
| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 8000 | Telephone | No | Not defined |
| 11025 | — | No | Not defined |
| 16000 | Wideband | No | Not defined |
| 22050 | — | No | Not defined |
| 32000 | Broadcast | Yes | SV7/SV8 supported |
| 44100 | CD audio | Yes | Primary rate |
| 48000 | Professional | Yes | Primary rate |
| 88200 | 2× CD | No | Not defined |
| 96000 | High-res | No | Not defined |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Pre-Processing Stage
- **DC offset removal:** Applied by encoder
- **Pre-emphasis filter:** Not applied
- **Windowing function:** Applied within the QMF polyphase filter bank
- **Level normalization:** Block companding via scalefactors (similar to MP2 but with finer granularity)
- **Stereo decorrelation pre-step:** MS stereo encoding optionally applied before quantization

### 4.2 Analysis / Transform Stage

#### Transform Type: Hybrid QMF + MDCT
```
Parameters:
  Primary filter:     32-band QMF polyphase filter bank (same as MP2)
  Secondary transform: MDCT on selected subbands (hybrid approach)
  Window size:        1152 samples (base frame)
  Overlap:           Critically sampled QMF + 50% MDCT overlap
  Window function:    Cosine-modulated filter + sine/cosine MDCT windows
```

**Key difference from MP2:** Musepack uses a hybrid approach where some subbands receive additional MDCT-based transform coding for better frequency resolution on transient or tonal content, rather than relying solely on the 32-band QMF.

### 4.3 Psychoacoustic Model (Lossy Only)

#### Model Version
- **Model:** Custom Musepack psychoacoustic model (derived from MP2 Annex D, extended)
- **Analysis window:** Per-frame FFT analysis (variable window size)

#### Masking Thresholds
```
Absolute Threshold of Hearing (ATH):
  ATH(f) = 3.64·(f/1000)^(-0.8) - 6.5·exp(-0.6·(f/100.0 - 3.3)^2) + 10^(-3)·(f/1000)^4 dB

Simultaneous Masking:
  Masking slope (upward):   ~5 dB/Bark
  Masking slope (downward): ~15 dB/Bark

Temporal Masking:
  Pre-masking:  ~20 ms
  Post-masking: ~150 ms
```

#### Bit Allocation Algorithm
```
1. Compute FFT of input frame
2. Compute masking threshold per critical band
3. Map subband energies to masking thresholds
4. Apply improved MNR optimization with noise substitution:
   - For bands below threshold: noise substitution (reuse bits elsewhere)
   - For bands above threshold: bit allocation based on SMR
5. Select Huffman codebook per subband based on expected coefficient distribution
6. Encode quantized coefficients using Huffman coding with escape codes
```

### 4.4 Quantization
- **Type:** Uniform quantization with block companding (scalefactors)
- **Step sizes:** Variable per subband based on bit allocation
- **Block companding:** Per-subband scalefactors with finer granularity than MP2
- **Noise substitution:** Technique where bands below the masking threshold are replaced with shaped noise rather than encoded zeros, improving perceptual quality

### 4.5 Stereo Encoding Modes
| Mode | Description | Condition for Selection | Notes |
|------|-------------|------------------------|-------|
| M/S Stereo | Mid/Side encoding | When S has less energy than M | Reduced bitrate for stereo |
| Intensity Stereo | High-frequency coupling | High compression | Limited to high frequencies |
| Channel Coupling | L/R coupling at high frequencies | Low bitrate | |

### 4.6 Entropy / Lossless Coding Stage
```
Method: Canonical Huffman coding (vector Huffman tables)

SV8 improvements:
  - Optimized canonical Huffman tables — 2% smaller files vs SV7
  - Faster decoding due to canonical table structure
  - Escape codes for out-of-range coefficients

Huffman codebook selection:
  - Multiple codebooks selected per subband
  - Codebooks optimized for different probability distributions
  - Vector Huffman for grouped coefficients
```

### 4.7 Encoder Settings / Quality Modes
| Quality Setting | Bitrate Range | Intended Use Case | Transparent? |
|-----------------|---------------|-------------------|--------------|
| Insane (q=10) | ~250–320 kbps | Highest quality | Yes |
| Standard (q=7) | ~175–185 kbps | Transparency target | Yes (most content) |
| Medium (q=5) | ~150–165 kbps | General use | Marginal |
| Low (q=3) | ~130–145 kbps | Low-bitrate | No |
| Radio (q=0) | ~100–120 kbps | Very low bitrate | No |

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
SV8:
1. Scan for "MPCK" magic bytes at file start
2. Parse SH (Stream Header) packet — mandatory first packet
3. Read audio packets (AP) sequentially
4. Each packet contains a variable number of coded frames

SV7:
1. Read file header (36 bytes)
2. Extract seek table from header
3. Read frames sequentially from data section
```

#### Seeking
- **SV8 seeking:** Sample-accurate seeking via seek table (ST packet) or by scanning audio packets
- **SV7 seeking:** Seek table in file header provides frame offsets; sample-accurate within frames
- **Seek precision:** Sample-accurate (SV8), frame-accurate (SV7)

### 5.2 Core Decode Pipeline
```
SV8 Decode:
1. Read MPCK header — verify "MPCK" signature
2. Parse SH packet:
   ├── Read CRC-32
   ├── Read stream version (must be 8)
   ├── Read sample count
   ├── Read flags (sample rate, channels, MS stereo)
   └── Read max band
3. Parse optional packets (RG=ReplayGain, EI=EncoderInfo, SO=SeekOffset, ST=SeekTable)
4. Parse AP (Audio Packet):
   ├── Read packet size
   ├── Decode Huffman-coded subband data
   ├── Apply inverse quantization
   ├── Apply MS stereo decode if flagged
5. Inverse hybrid transform:
   ├── QMF synthesis for base subbands
   └── Inverse MDCT for hybrid-coded subbands
6. Apply scalefactor decompanding
7. Output to PCM buffer

SV7 Decode:
1. Read "MP+" header (4 bytes) + version
2. Read stream header (36 bytes)
3. Read seek table (nav info)
4. Decode frames sequentially:
   ├── Read frame header
   ├── Decode Huffman-coded samples
   ├── Inverse QMF synthesis
   └── Apply scalefactors
5. Output PCM samples
```

### 5.3 Error Concealment
- **Corrupt frame detection:** CRC-32 check in SV8; frame header validation in SV7
- **Concealment method:** Muting or interpolation from neighboring frames
- **Maximum consecutive errors before silence:** Implementation-defined

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** MPC file (`.mpc`) is itself a container for the Musepack stream
- **Overhead:** ~0% (SV8) to ~1% (SV7) depending on seek table size
- **Seeking in native container:** Yes — seek table in SV8 (ST packet) and SV7 (header nav info)
- **Multiple streams in native container:** No — single audio stream per file

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| MP4/M4A | No | — | — | |
| Matroska/MKA | Yes | Yes | Full | Via Matroska Audio codec ID |
| OGG | No | — | — | |
| WAV | No | — | — | |
| AIFF | No | — | — | |
| NUT | Yes | Yes | Full | Open container format |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** APEv2 (for SV7 and SV8 `.mpc` files)
- **Tag block location:** End of file (APEv2 standard placement)
- **Tag block identifier:** `APETAGEX` at the end of the file

### 7.2 Standard Tag Fields — Complete Reference
| Tag Field | Internal Key (this format) | Max Length | Character Encoding | Multi-value | Notes |
|-----------|---------------------------|------------|-------------------|-------------|-------|
| Title | Title | unlimited | UTF-8 | No | APEv2 text frame |
| Artist | Artist | unlimited | UTF-8 | No | |
| Album | Album | unlimited | UTF-8 | No | |
| Album Artist | Album Artist | unlimited | UTF-8 | No | |
| Composer | Composer | unlimited | UTF-8 | Yes | |
| Genre | Genre | unlimited | UTF-8 | No | |
| Year / Date | Year | unlimited | UTF-8 | No | |
| Track Number | Track | unlimited | UTF-8 | No | Format: "N" or "N/Total" |
| Disc Number | Disc | unlimited | UTF-8 | No | |
| Comment | Comment | unlimited | UTF-8 | Yes | |
| ReplayGain Track Gain | REPLAYGAIN_TRACK_GAIN | — | ASCII | No | Format: "-6.20 dB" |
| ReplayGain Track Peak | REPLAYGAIN_TRACK_PEAK | — | ASCII | No | Format: "0.998459" |
| ReplayGain Album Gain | REPLAYGAIN_ALBUM_GAIN | — | ASCII | No | |
| ReplayGain Album Peak | REPLAYGAIN_ALBUM_PEAK | — | ASCII | No | |
| Cover Art | METADATA_BLOCK_PICTURE | up to several MB | Binary | No | Vorbis Comment style picture block |
| Arbitrary/Custom | any key | unlimited | UTF-8 | Yes | APEv2 allows any key |

### 7.3 Cover Art Storage
```
Cover art storage format in MPC:
  SV8 (APEv2):     METADATA_BLOCK_PICTURE frame (identical to Vorbis/Opus)
  SV7 (APEv2):     Same — METADATA_BLOCK_PICTURE frame

  Binary layout (METADATA_BLOCK_PICTURE):
    [0x00-0x03]  Type        uint32     Picture type (0=Other, 1=32x32, 3=Front cover)
    [0x04-0x??]  MIME length  uint32     Length of MIME string
    [0x??-0x??]  MIME         string     Image MIME type (e.g., "image/jpeg")
    [0x??-0x??]  Desc length  uint32     Length of description string
    [0x??-0x??]  Description  string     UTF-8 description
    [0x??-0x??]  Width        uint32     Image width in pixels
    [0x??-0x??]  Height       uint32     Image height in pixels
    [0x??-0x??]  Depth        uint32     Color depth (bits per pixel)
    [0x??-0x??]  Colors       uint32     Number of colors (0=unknown)
    [0x??-0x??]  Data length  uint32     Length of image data
    [0x??-end]   Image data   binary     Raw JPEG/PNG data
```

### 7.4 Metadata Compatibility Matrix
| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| ID3v1 | ✗ | ✗ | ✗ | N/A |
| ID3v2.3 | ✗ | ✗ | ✗ | N/A |
| ID3v2.4 | ✗ | ✗ | ✗ | N/A |
| APEv2 | ✓ | ✓ | ✓ | Primary |
| Vorbis Comments | ✓ (SV8) | ✓ (SV8) | ✓ | Compatibility |
| MP4 Atoms | ✗ | ✗ | ✗ | N/A |

**Note:** Musepack natively uses APEv2 tags. FFmpeg reads/writes APEv2 tags in MPC files. Some tag editors may convert to/from ID3v2 for cross-format compatibility.

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   mpc7                          # SV7 decoder
                    mpc8                          # SV8 decoder (preferred)
AV_CODEC_ID:        AV_CODEC_ID_MUSEPACK7         # SV7
                    AV_CODEC_ID_MUSEPACK8         # SV8
Format Name (CLI):  mpc, musepack                   # file format
Encoder(s):         NONE — FFmpeg does NOT encode Musepack
Decoder(s):         mpc7, mpc8                    # Both SV7 and SV8
Muxer(s):           mpc, matroska, nut             # Raw MPC, Matroska, NUT
Demuxer(s):         mpc, musepack                 # Auto-detects SV7/SV8
```

**Critical:** FFmpeg has **no Musepack encoder**. To create MPC files, use the official Musepack command-line tools (`mpcenc`) or foobar2000's encoder plugin.

### 8.2 FFmpeg Encoding — Full CLI Reference

**FFmpeg does NOT encode Musepack.** Use external tools:
- **Official mpcenc** (SV8 encoder): Available from musepack.net
- **foobar2000** with Maspuck encoder component
- **LAME-based alternatives** are not applicable

```bash
# FFmpeg CANNOT encode Musepack — only decodes

# Correct encoding workflow (external tool required):
# 1. Use mpcenc from musepack.net:
mpcenc input.wav --standard output.mpc    # ~175-185 kbps
mpcenc input.wav --extreme output.mpc    # Higher quality
mpcenc input.wav --braindead output.mpc   # Lowest quality

# 2. Or use foobar2000 with Musepack encoder component
```

### 8.3 FFmpeg Decoding — Full CLI Reference

```bash
# Decode MPC (SV7 or SV8) to WAV
ffmpeg -i input.mpc \
  -c:a pcm_s16le \
  -ar 44100 \
  -ac 2 \
  output.wav

# Decode with automatic SV7/SV8 detection
ffmpeg -i input.mpc \
  -c:a pcm_s16le \
  output.wav

# Extract specific stream
ffmpeg -i input.mpc -map 0:a:0 -c:a copy output.mpc

# Probe format without decoding
ffprobe -v quiet -print_format json -show_streams -show_format input.mpc
```

### 8.4 FFmpeg Decoding — libavcodec C API

```c
// ─── Complete decode program skeleton ────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.mpc", NULL, NULL);
avformat_find_stream_info(fmt_ctx, NULL);

// Find audio stream
int audio_idx = av_find_best_stream(fmt_ctx, AVMEDIA_TYPE_AUDIO, -1, -1, NULL, 0);
AVStream *stream = fmt_ctx->streams[audio_idx];

// Find and open decoder (mpc8 for SV8, mpc7 for SV7)
const AVCodec *dec = avcodec_find_decoder(stream->codecpar->codec_id);
if (!dec) {
    // Try both SV7 and SV8 decoders
    dec = avcodec_find_decoder_by_name("mpc8");  // Prefer SV8
    if (!dec) dec = avcodec_find_decoder_by_name("mpc7");  // Fallback SV7
}
AVCodecContext *dec_ctx = avcodec_alloc_context3(dec);
avcodec_parameters_to_context(dec_ctx, stream->codecpar);
avcodec_open2(dec_ctx, dec, NULL);

// Decode loop
AVPacket *pkt = av_packet_alloc();
AVFrame  *frm = av_frame_alloc();

while (av_read_frame(fmt_ctx, pkt) >= 0) {
    if (pkt->stream_index == audio_idx) {
        avcodec_send_packet(dec_ctx, pkt);
        while (avcodec_receive_frame(dec_ctx, frm) == 0) {
            // frm->data[0..channels-1] contain audio samples (planar S16)
            // frm->nb_samples = sample count per channel
            // frm->format = AV_SAMPLE_FMT_S16P
            // frm->sample_rate = actual rate
            process_audio_frame(frm);
            av_frame_unref(frm);
        }
    }
    av_packet_unref(pkt);
}

// Flush decoder
avcodec_send_packet(dec_ctx, NULL);
while (avcodec_receive_frame(dec_ctx, frm) == 0) {
    process_audio_frame(frm);
    av_frame_unref(frm);
}

// Cleanup
av_frame_free(&frm);
av_packet_free(&pkt);
avcodec_free_context(&dec_ctx);
avformat_close_input(&fmt_ctx);
```

### 8.5 FFmpeg Metadata Handling

```bash
# Read all metadata as JSON
ffprobe -v quiet -print_format json -show_format input.mpc | jq .format.tags

# Write metadata (copy audio, update tags)
# Note: FFmpeg handles APEv2 tags in MPC
ffmpeg -i input.mpc \
  -c copy \
  -metadata title="Song Title" \
  -metadata artist="Artist Name" \
  -metadata album="Album Name" \
  -metadata track="5/12" \
  -metadata date="2024" \
  -metadata genre="Electronic" \
  -metadata REPLAYGAIN_TRACK_GAIN="-6.20 dB" \
  -metadata REPLAYGAIN_TRACK_PEAK="0.998459" \
  output.mpc

# Strip all metadata
ffmpeg -i input.mpc -c copy -map_metadata -1 output.mpc

# Embed cover art (APEv2 METADATA_BLOCK_PICTURE)
ffmpeg -i input.mpc -i cover.jpg \
  -c copy \
  -map 0:a -map 1:v \
  -metadata:s:v title="Album cover" \
  -disposition:v attached_pic \
  output.mpc
```

#### FFmpeg Internal Metadata Key Mapping
| Standard Field | FFmpeg Key | MPC Native Key | Notes |
|----------------|------------|--------------------------|-------|
| Title | title | Title | APEv2 |
| Artist | artist | Artist | |
| Album | album | Album | |
| Album Artist | album_artist | Album Artist | |
| Track Number | track | Track | FFmpeg uses "N/Total" format |
| Disc Number | disc | Disc | |
| Genre | genre | Genre | |
| Date/Year | date | Year | |
| Comment | comment | Comment | |
| ReplayGain Track Gain | REPLAYGAIN_TRACK_GAIN | REPLAYGAIN_TRACK_GAIN | |
| ReplayGain Track Peak | REPLAYGAIN_TRACK_PEAK | REPLAYGAIN_TRACK_PEAK | |
| Composer | composer | Composer | |
| Copyright | copyright | Copyright | |

### 8.6 Quality / Fidelity Decision Table
| Use Case | Settings | Expected Size (min) | Notes |
|----------|----------|---------------------|-------|
| Archival / audiophile | External encoder `--extreme` / `--insane` | ~250–320 kbps | Best possible MPC |
| Standard quality | External encoder `--standard` | ~175–185 kbps | Transparency target |
| Streaming (high) | External encoder `--standard` | ~175–185 kbps | |
| Streaming (standard) | External encoder `--medium` | ~150–165 kbps | |
| Podcast / voice | External encoder `--radio` | ~100–120 kbps | Not recommended |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure

**SV8 Seek Table (ST packet):**
```
Location:    Within MPCK stream (ST packet after SH)
Magic:       "ST" (0x5354)
Entry size:  Variable (packed offsets)

SV8 ST Packet format:
  [0x00]     Key     "ST" (2 bytes)
  [Size]     Size    Variable-size integer
  [Payload]   Entries Packed seek entries
              Each entry: sample_position (varint) + offset (varint)
```

**SV7 Seek Table (NAV info):**
```
Location:    Header at offset specified by NavInfoOffset
Size:        NavInfoSize bytes
Entry size:  8 bytes per entry (frame offset + sample offset)

SV7 entry format:
  [0x00-0x03]  byte_offset  (uint32 LE)  Byte offset of frame in file
  [0x04-0x07]  sample_offset (uint32 LE)  Sample number of frame start
```

### 9.2 Gapless Playback Data
```
Encoder delay:   ~1393 samples [NEEDS VERIFICATION] (pre-echo look-ahead in psychoacoustic model)
Padding:         ~1393 samples [NEEDS VERIFICATION] (encoder delay compensation)
Storage location: Not stored in bitstream — player must handle
Example value:    Not applicable — gapless requires player-level handling
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | SV8 is fully packetized and streamable |
| Algorithmic encoder delay | ~1000 samples / ~23 ms | Higher than MP2 |
| Live encoding feasible | Yes (with external encoder) | Not native to FFmpeg |
| HTTP progressive download | Yes | Frame-based, seekable with index |
| HTTP Live Streaming (HLS) | No | Not a standard HLS codec |
| DASH streaming | Yes (theoretical) | Can be muxed into Matroska for DASH |
| WebRTC / RTP transport | No | Not standard RTP codec |
| Minimum decode buffer | 1 frame / ~23 ms | |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | Layout Name | Channel Order | FFmpeg Layout Constant |
|----------|-------------|---------------|----------------------|
| 1 | Mono | C | AV_CHANNEL_LAYOUT_MONO |
| 2 | Stereo | L, R | AV_CHANNEL_LAYOUT_STEREO |

**Note:** Musepack was designed exclusively for stereo (and mono) audio. Multi-channel support is not defined in the specification.

### 11.2 Downmix Coefficients (Stereo → Mono)
```
Mono = (L + R) / 2
```

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 16-bit | Fixed at 16-bit PCM |
| Max sample rate | 48,000 Hz | SV7/SV8 support |
| Float support | No | Only integer PCM |
| DSD support | No | |
| 24-bit support | No | Not defined |

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

| Platform / API | Encode | Decode | FFmpeg Flag | Notes |
|----------------|--------|--------|-------------|-------|
| NVIDIA NVENC | No | — | — | No hardware MPC encoder |
| NVIDIA NVDEC | — | No | — | No hardware MPC decoder |
| Intel QSV | No | No | — | Not supported |
| Apple VideoToolbox | No | No | — | No hardware MPC support |
| Android MediaCodec | No | No | — | Not supported |
| VA-API (Linux) | No | No | — | Not supported |

**Note:** Musepack is decoded exclusively in software. No hardware acceleration is available on any platform.

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Encoder-Specific Issues
| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| No encoder exists in FFmpeg | All | Use official mpcenc or foobar2000 |
| SV7/SV8 auto-detection may fail on edge cases | Some | Use explicit codec selection |
| APEv2 tag handling varies by FFmpeg version | Older versions | Update FFmpeg |

### 14.2 Interoperability Issues
- **Musepack encoder vs FFmpeg decoder:** Generally excellent compatibility; SV7 and SV8 both fully supported
- **Files tagged with multiple tag systems:** APEv2 is primary; FFmpeg handles APEv2 correctly
- **SV7 files vs SV8 decoders:** FFmpeg's mpc8 decoder can decode SV7 files (backwards compatible)

### 14.3 Edge Cases to Handle in Converter
- **Empty file / 0-sample stream:** Return empty output
- **File < 1 frame:** Decode as much as possible
- **All-silence audio:** Decodes correctly
- **Corrupt frame:** Skip frame, mute, or interpolate
- **Unknown stream version:** Try SV7 then SV8 decoder
- **Very large files:** Seek table enables efficient seeking

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM Musepack

| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|--------------|
| → FLAC | `ffmpeg -i in.mpc -c:a flac -compression_level 8 out.flac` | All tags (via APEv2→Vorbis rewrite) | Lossless decode |
| → ALAC | `ffmpeg -i in.mpc -c:a alac out.m4a` | Partial | Lossless decode |
| → MP3 | `ffmpeg -i in.mpc -c:a libmp3lame -q:a 0 out.mp3` | Tags via ID3v2.3 | Generation loss |
| → AAC | `ffmpeg -i in.mpc -c:a aac -b:a 256k out.m4a` | Partial | Generation loss |
| → Opus | `ffmpeg -i in.mpc -c:a libopus -b:a 128k out.opus` | Vorbis Comments | Generation loss |
| → WAV | `ffmpeg -i in.mpc -c:a pcm_s16le out.wav` | APEv2 tags | Lossless decode |
| → OGG Vorbis | `ffmpeg -i in.mpc -c:a libvorbis -q:a 6 out.ogg` | Vorbis Comments | Generation loss |

### 15.2 Converting TO Musepack

| Source | Command | Metadata Preserved | Quality Notes |
|--------|---------|-------------------|--------------|
| FLAC → | External mpcenc (not FFmpeg) | Vorbis → APEv2 | Lossless decode, lossy encode |
| WAV → | External mpcenc | None (no tags) | Lossless decode, lossy encode |
| MP3 → | External mpcenc | ID3v2 → APEv2 | Generation loss |
| AAC → | External mpcenc | Partial | Generation loss |
| Vorbis → | External mpcenc | Vorbis → APEv2 | Generation loss |

### 15.3 Lossless Round-Trip Verification
```bash
# Decode to WAV (lossless decode)
ffmpeg -i input.mpc -c:a pcm_s16le decoded.wav

# Lossless verification (compare source decode with original WAV)
# Note: MPC is lossy — bit-exact match only if source was already MPC

# For lossy source comparison:
ffmpeg -i input.mpc -map 0:a -f framemd5 input.md5
ffmpeg -reference.mpc -map 0:a -f framemd5 reference.md5
diff input.md5 reference.md5
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| Official mpcenc/mpcdec | C | BSD | Reference | Reference | https://www.musepack.net |
| FFmpeg native (libavcodec) | C | LGPL 2.1+ | — (no encoder) | 10/10 | https://ffmpeg.org |
| libmpcdec (in FFmpeg) | C | LGPL 2.1+ | — | 10/10 | Part of FFmpeg |
| foobar2000 component | C++ | Proprietary | High | High | https://www.foobar2000.org |
| Rockbox | C | Various | — | 10/10 | https://www.rockbox.org |

### Build Instructions
```bash
# Build official Musepack tools from source
git clone https://github.com/rfc2190/musepack.git
cd musepack
# Note: Official source may not be on GitHub — download from musepack.net

# Download official release:
wget https://www.musepack.net/musepack-src.tar.gz
tar xzf musepack-src.tar.gz
cd musepack
make
sudo make install

# FFmpeg includes Musepack decoders by default (enabled by default)
# No additional configure flags needed:
./configure
make -j$(nproc)
```

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **Musepack SV8 Release Notes:** https://www.musepack.net/SV8_release.txt
- **SV8 Specification (Mutagen):** http://mutagen-specs.readthedocs.io/en/latest/mpc/sv8.html
- **SV7 Specification:** Available at musepack.net (historical)

### Technical Resources
- FFmpeg Musepack documentation: https://ffmpeg.org/ffmpeg-codecs.html
- Musepack official site: https://www.musepack.net
- Multimedia Wiki Musepack: https://wiki.multimedia.cx/index.php/Musepack
- Hydrogenaudio Musepack discussion: https://hydrogenaud.io/

### Academic Papers
- Klemm, F., *The MPEG+ Audio Codec — An Open Solution for Transparent Stereo Audio Coding*, 2001
- Buschmann, A., *Musepack SV8 Technical Description*, 2009

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] FFmpeg decoders enabled by default — verify with `ffmpeg -decoders | grep mpc`
- [ ] FFmpeg has NO encoder — must use external `mpcenc` tool
- [ ] Verify `ffprobe` correctly identifies SV7 and SV8 files
- [ ] Handle both `.mpc` and `.mp+` file extensions

### Encoding Pipeline
- [ ] **CRITICAL:** FFmpeg cannot encode Musepack — integrate external mpcenc tool
- [ ] Support quality presets: `--radio`, `--medium`, `--standard`, `--extreme`, `--insane`
- [ ] Support input from WAV or lossless codecs (FLAC, ALAC)
- [ ] Handle APEv2 metadata writing via external tool

### Decoding Pipeline
- [ ] Auto-detect SV7 vs SV8 based on file header
- [ ] Prefer mpc8 decoder for SV8, mpc7 decoder for SV7
- [ ] Handle AVERROR(EAGAIN) in receive_frame loop correctly
- [ ] Flush decoder at EOF (send NULL packet)
- [ ] Handle planar S16 output format (AV_SAMPLE_FMT_S16P)
- [ ] Convert to desired output format via libswresample

### Metadata
- [ ] Read APEv2 tags from MPC files
- [ ] Write APEv2 tags to MPC files (via external encoder or tag library)
- [ ] Map APEv2 fields to standard names (Section 7.2)
- [ ] Read cover art (METADATA_BLOCK_PICTURE) and preserve as JPEG/PNG binary
- [ ] Write cover art as METADATA_BLOCK_PICTURE frame
- [ ] Preserve ReplayGain tags through conversion

### Quality & Verification
- [ ] Lossless verification only applies to decode step (MPC is lossy)
- [ ] Implement progress reporting from external encoder
- [ ] Test with SV7 and SV8 files
- [ ] Test with files containing APEv2 tags and cover art

### Edge Cases
- [ ] Handle files with corrupt or missing headers
- [ ] Handle files with 0 samples
- [ ] Handle sample rate mismatch (resample via libswresample)
- [ ] Handle mono vs stereo conversion
- [ ] Handle very large files (test seek table)
- [ ] Handle files with no seek table (scan audio packets)

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
