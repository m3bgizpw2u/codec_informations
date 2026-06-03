# Sony ATRAC — Deep Technical Reference
> **Category:** Lossy
> **File Extensions:** `.oma`, `.aa3`, `.at3`, `.wav` (ATRAC-in-WAV), `.atrac` [NEEDS VERIFICATION]
> **MIME Types:** `audio/x-sony-oma`, `audio/x-sony-atrac`
> **Standardization Body:** Sony (proprietary — format never fully publicly documented)
> **Primary Specification:** No public specification — reverse-engineered by FFmpeg and Rockbox
> **Patent Status:** Patented — Sony proprietary; patent status expired for early versions
> **License:** Proprietary — licensing via Sony; decoders reverse-engineered for open-source
> **Current Version:** ATRAC9 (PlayStation 5/4/Vita) — codec family still in active use
> **Active Development:** Minimal for legacy versions; ATRAC9 still actively used in Sony games

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Sony Corporation of America / Sony Audio Lab
- **Year Created:** 1992 (original ATRAC), with major revisions through 2006
- **Original Purpose:** Enable the Sony MiniDisc format to store the same amount of music as a CD (~80 minutes) on a much smaller 64mm magneto-optical disc, by compressing the 1411 kbps CD audio stream to ~292 kbps (a ~5:1 compression ratio)
- **Problem with Predecessors:** Analog tape had limited dynamic range and noise. PCM digital recording required too much storage for portable media. ATRAC was specifically designed to exploit auditory masking to reduce bitrate while maintaining perceptually transparent quality.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| ATRAC (original) | 1992 | First version for MiniDisc; 292 kbps; hybrid subband/MDCT |
| ATRAC Type R | ~1996 | Improved bit allocation, better quality at same bitrate |
| ATRAC Type S | ~1998 | Further refinements; used in Sony Walkman |
| ATRAC3 | 1999 | New codec (not a version of ATRAC); 66/105/132 kbps; MDLP modes |
| ATRAC3plus | 2003 | Improved compression; 48–352 kbps; Hi-MD, PSP, PS3 |
| ATRAC Advanced Lossless | 2006 | Hybrid lossless (lossy core + lossless correction) |
| ATRAC9 | 2012+ | Current generation; used in PS5, PS4, PS Vita; 4–8 channels |

### 1.3 Current Adoption
- **Primary use cases today:** Legacy (MiniDisc-era audio), Sony game audio (ATRAC9), PlayStation platform audio, some Sony Walkman legacy devices
- **Platforms with native support:** PlayStation 5, PlayStation 4, PlayStation Vita (ATRAC9), older Sony Walkman (ATRAC3/ATRAC3plus), SonicStage/Media Go software
- **Major services using this format:** Sony PlayStation Store (ATRAC9 in game bundles), legacy Sony Connect Music Store [discontinued], MiniDisc recordings
- **Hardware support:** Limited — billions of MiniDisc units exist but are largely retired; PlayStation hardware supports ATRAC9
- **Status:** Largely deprecated for ATRAC/ATRAC3/ATRAC3plus; ATRAC9 remains active in Sony gaming ecosystem

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Hybrid subband/transform coder
- **Core algorithm:** ATRAC: QMF (Quadrature Mirror Filter) polyphase filter bank + MDCT on selected subbands. ATRAC3: similar but with different block sizes. ATRAC3plus: 16-channel PQF + 128-point MDCT + Generalized Harmonic Analysis (GHA).
- **Loss mechanism:** Psychoacoustic masking via subband quantization — each version improved the balance between tonal extraction and noise allocation
- **Frame-based vs sample-based:** Frame-based (variable frame sizes)
- **Fixed vs variable frame size:** Variable frame sizes per version: ATRAC ~1024 samples, ATRAC3 ~2048 samples, ATRAC3plus ~2048 samples

### 2.2 High-Level Encoding Flow (Block Diagram)

**ATRAC (MiniDisc original):**
```
Input PCM Samples (~1024 samples per frame)
      │
      ▼
[Pre-processing: Split into 2 frequency regions via QMF]
      │
      ▼
[Low-frequency region: MDCT with window switching]
      │
      ▼
[High-frequency region: Band-split and requantization]
      │
      ▼
[Bit allocation: Per-region MNR optimization]
      │
      ▼
[Quantization: Uniform quantization per subband]
      │
      ▼
[Bitstream packing: Frame header + spectral data + parameters]
      │
      ▼
Output Encoded ATRAC Bitstream
```

**ATRAC3plus (Hi-MD / PSP):**
```
Input PCM Samples (~2048 samples per frame per channel block)
      │
      ▼
[16-channel Polyphase Quadrature Filter (PQF)]
      │
      ▼
[Generalized Harmonic Analysis (GHA) — extract tonal components]
      │
      ▼
[128-point MDCT on remaining non-tonal components]
      │
      ▼
[Gain control windows: Pre-echo prevention]
      │
      ▼
[Bit allocation: MNR optimization across all spectral components]
      │
      ▼
[Huffman coding: Variable-length codes for spectral data]
      │
      ▼
[Channel coupling: MS stereo or intensity stereo]
      │
      ▼
Output Encoded ATRAC3plus Bitstream
```

### 2.3 Key Design Parameters
| Parameter | ATRAC | ATRAC3 | ATRAC3plus |
|-----------|-------|--------|------------|
| Algorithmic delay | ~58 ms [NEEDS VERIFICATION] | ~58 ms | ~58 ms [NEEDS VERIFICATION] |
| Frame size | ~1024 samples | ~2048 samples | 2048 samples per channel block |
| Max channels | 2 | 2 (stereo) | 8 (channel blocks) |
| Max bit depth | 16-bit | 16-bit | 16-bit |
| Max sample rate | 44,100 Hz | 44,100 Hz | 48,000 Hz |
| Bitrate range | 292 kbps | 66/105/132 kbps | 48–352 kbps |
| Complexity | Moderate | Moderate | Higher |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes

**ATRAC3 / ATRAC3plus in WAV:**
```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       52 49 46 46    RIFF     RIFF container
0x0008  4       57 41 56 45    WAVE     WAVE format chunk
0x0012  4       66 6D 74 20    fmt      Format chunk identifier
0x0016  4       XX XX XX XX    —        Chunk size
0x001A  2       XX XX          —        Audio format:
                                          0xFFFE = WAVE_FORMAT_EXTENSIBLE
                                          In GUID field: ATRAC3/3+ GUID
0x0024  2       XX XX          —        Number of channels
0x0026  4       XX XX XX XX    —        Sample rate (44100/48000)
...
```

**OMA file (Sony OpenMG Audio Container):**
```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       53 4F 4E 59    SONY     OMA file signature
0x0004  4       XX XX XX XX    —        File version
0x0008  4       XX XX XX XX    —        Header size
...
```

**AA3 file (ATRAC Audio File):**
```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       66 4D 74 41    fMtA     AA3 file signature [NEEDS VERIFICATION]
...
```

### 3.2 File-Level Header Layout

**ATRAC3 / ATRAC3plus in WAV (WAVE_FORMAT_EXTENSIBLE):**
```
Offset   Size    Field Name              Type        Valid Range   Description
-------  ------  ----------------------  ----------  -----------   ---------------------------
0x0000   4B      RIFF Magic             char[4]     "RIFF"       File container magic
0x0004   4B      File Size              uint32 LE   —            File size minus 8 bytes
0x0008   4B      WAVE Magic           char[4]     "WAVE"       RIFF wave chunk
0x000C   4B      fmt  Magic           char[4]     "fmt "       Format chunk identifier
0x0010   4B      fmt  Chunk Size       uint32 LE   40           Chunk size for extensible
0x0014   2B      Audio Format          uint16 LE   0xFFFE       WAVE_FORMAT_EXTENSIBLE
0x0016   2B      Number of Channels   uint16 LE   1–2          1=mono, 2=stereo
0x0018   4B      Sample Rate           uint32 LE   44100,48000  Samples per second
0x001C   4B      Byte Rate             uint32 LE   dependent    Bytes per second
0x0020   2B      Block Align           uint16 LE   dependent    Frame size in bytes
0x0022   2B      Bits Per Sample       uint16 LE   0            0=variable
0x0024   2B      Extra Size            uint16 LE   22           Size of extension
0x0026   2B      Valid Bits Per Sample uint16 LE   16           Actual bit depth
0x0028   4B      Channel Mask          uint32 LE   0x00000003   Stereo mask
0x002C   16B     SubFormat GUID         uint8[16]   —            ATRAC3/ATRAC3+ GUID
```

### 3.3 Frame / Block Header Layout

**ATRAC3plus Frame Structure:**
```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------
[0]      variable Frame Header           bits        Sync + frame size + parameters
[Data]   variable Spectral Data          bits        Huffman-coded MDCT coefficients
[End]    variable CRC                   bits        Optional frame CRC
```

**ATRAC3plus Frame Sizes:**
| Bitrate (kbps) | Frame Size (bytes) | Notes |
|----------------|-------------------|-------|
| 48 | 280 | Hi-LP mode |
| 64 | 376 | |
| 96 | 560 | |
| 128 | 744 | |
| 160 | 936 | |
| 192 | 1120 | |
| 256 | 1488 | |
| 320 | 1864 | |
| 352 | 2048 | Hi-SP mode |

### 3.4 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Unsigned integer | No | Not defined |
| 16-bit | Signed integer | Yes | Standard input/output |
| 20-bit | Signed integer | No | Not supported |
| 24-bit | Signed integer | No | Not supported |
| 32-bit | Signed integer | No | Not supported |
| 32-bit | IEEE float | No | Not supported |
| 64-bit | IEEE float | No | Not supported |

#### Supported Sample Rates
| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 32000 | Broadcast | Yes (ATRAC3plus) | |
| 44100 | CD audio | Yes | Primary rate for ATRAC/ATRAC3 |
| 48000 | Professional | Yes (ATRAC3plus) | PSP/PS3 support |
| 88200 | 2× CD | No | Not defined |
| 96000 | High-res | No | Not defined |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Pre-Processing Stage
- **DC offset removal:** Applied by encoder
- **Pre-emphasis filter:** Not applied
- **Windowing function:** MDCT windows for frequency-domain coding; sine/cosine windows
- **Level normalization:** Gain control blocks applied to prevent pre-echo
- **Stereo decorrelation pre-step:** MS stereo or intensity stereo encoding before quantization

### 4.2 Analysis / Transform Stage

#### ATRAC3plus Transform Type: 16-channel PQF + 128-point MDCT + GHA
```
Parameters:
  Primary filter:     16-channel Polyphase Quadrature Filter (PQF)
  Secondary transform: 128-point MDCT per PQF band
  Tonal extraction:   Generalized Harmonic Analysis (GHA)
  Window sizes:       Variable — window switching for transient handling
  Block size:        2048 samples per channel block
  Overlap:           50% for MDCT, critically sampled for PQF
```

**ATRAC vs ATRAC3plus difference:** ATRAC used a simpler QMF split + MDCT approach. ATRAC3plus introduced the 16-channel PQF (vs ATRAC's 3-band QMF) and GHA for better tonal extraction.

### 4.3 Psychoacoustic Model (Lossy Only)

#### Model Version
- **Model:** Sony proprietary psychoacoustic model
- **Analysis window:** Variable per version

#### Masking Thresholds
```
Absolute Threshold of Hearing (ATH):
  ATH(f) = 3.64·(f/1000)^(-0.8) - 6.5·exp(-0.6·(f/1000 - 3.3)^2) + 10^(-3)·(f/1000)^4 dB

Simultaneous Masking:
  Masking slope (upward):   ~5 dB/Bark
  Masking slope (downward): ~15 dB/Bark

Temporal Masking:
  Pre-masking:  ~20 ms (via gain control windows)
  Post-masking: ~100 ms
```

#### Bit Allocation Algorithm
```
1. Split signal into 16 subbands via PQF
2. Apply GHA to extract tonal components from each subband
3. Classify remaining energy as noise-like (non-tonal)
4. Compute masking threshold for tonal and noise components
5. Allocate bits to minimize MNR across all components
6. Apply Huffman coding to quantized coefficients
```

### 4.4 Quantization
- **Type:** Uniform quantization with variable step sizes per subband
- **Step sizes:** Per-subband based on bit allocation
- **Block companding:** Scalefactors per block for gain control
- **Noise shaping:** Gain control windows for pre-echo prevention

### 4.5 Stereo Encoding Modes
| Mode | Description | Condition for Selection |
|------|-------------|------------------------|
| MS Stereo | Mid/Side encoding | When side channel has less energy |
| Intensity Stereo | High-freq coupling | Low bitrate modes |
| Channel Coupling | L/R coupling at high frequencies | ATRAC3plus |

### 4.6 Entropy / Lossless Coding Stage
```
Method: Huffman coding (ATRAC3plus)

ATRAC3plus Huffman coding:
  - Variable-length codes for spectral coefficients
  - Multiple codebooks selected per spectral region
  - Escape codes for large coefficients
```

### 4.7 Encoder Settings / Quality Modes

**ATRAC (MiniDisc):**
| Quality Setting | Bitrate | Notes |
|-----------------|---------|-------|
| High | 292 kbps | Standard MiniDisc quality |

**ATRAC3:**
| Quality Setting | Bitrate | Notes |
|-----------------|---------|-------|
| LP2 | 132 kbps | 2× recording time on MD |
| LP4 | 66 kbps | 4× recording time on MD |

**ATRAC3plus:**
| Quality Setting | Bitrate | Notes |
|-----------------|---------|-------|
| Hi-SP | 352 kbps | Highest quality on Hi-MD |
| Hi-LP | 48–80 kbps | Low-bitrate modes |
| Standard | 128–192 kbps | Default quality |

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
ATRAC3plus:
1. Read WAV/OMA header and extract ATRAC3plus configuration
2. Extract extradata from container format
3. Parse frame header for frame size and parameters
4. Decode Huffman-coded spectral data
5. Reconstruct MDCT coefficients
6. Apply inverse MDCT and PQF synthesis
7. Output PCM samples
```

#### Seeking
- **CBR seeking:** Frame-based seeking — `byte_offset = frame_number × frame_size`
- **VBR seeking:** Requires seek table or scan
- **Seek precision:** Frame-accurate

### 5.2 Core Decode Pipeline
```
ATRAC3plus Decode:
1. Read container header (WAV, OMA, AA3)
2. Extract ATRAC3plus configuration (extradata)
3. Parse frame header:
   ├── Frame size in bytes
   ├── Number of coded channels
   └── Joint stereo parameters
4. Decode Huffman-coded spectral data per channel block
5. Dequantize spectral coefficients
6. Reconstruct tonal components (from GHA parameters)
7. Apply inverse MDCT to spectral coefficients
8. Reconstruct subband signals from MDCT output
9. Apply inverse PQF synthesis
10. Apply gain control for pre-echo prevention
11. Output to PCM buffer
```

### 5.3 Error Concealment
- **Corrupt frame detection:** Frame header validation, CRC checks
- **Concealment method:** Muting or interpolation
- **Maximum consecutive errors before silence:** Implementation-defined

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** OMA (Sony OpenMG Audio) — proprietary container format
- **Also stored in:** WAV (via WAVE_FORMAT_EXTENSIBLE), AA3 (ATRAC Audio File)
- **Overhead:** Container-dependent
- **Seeking in native container:** WAV seek table, OMA proprietary index

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| MP4/M4A | No | — | — | |
| Matroska/MKA | Yes | Yes | Full | Via Matroska Audio codec ID |
| OGG | No | — | — | |
| WAV | Yes (ATRAC3/3+) | Yes | Limited | WAVE_FORMAT_EXTENSIBLE |
| AIFF | No | — | — | |
| OMA | Yes | Yes | Full | Sony OpenMG container |
| AA3 | Yes | Yes | Full | Sony ATRAC Audio |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** ID3v2 (in OMA/WAV containers), OpenMG proprietary metadata (in OMA)
- **Tag block location:** Varies by container

### 7.2 Standard Tag Fields — Complete Reference
| Tag Field | Internal Key | Max Length | Character Encoding | Multi-value | Notes |
|-----------|-------------|------------|-------------------|-------------|-------|
| Title | TIT2 | — | UTF-16 | No | ID3v2 in WAV |
| Artist | TPE1 | — | UTF-16 | No | |
| Album | TALB | — | UTF-16 | No | |
| Year | TYER | — | UTF-16 | No | |
| Track Number | TRCK | — | UTF-16 | No | |
| Genre | TCON | — | UTF-16 | No | |
| Comment | COMM | — | UTF-16 | Yes | |
| Cover Art | APIC | — | Binary | No | ID3v2 |
| OpenMG | Various | — | — | — | Proprietary in OMA |

### 7.3 Cover Art Storage
```
Cover art storage format in ATRAC containers:
  Via ID3v2 APIC frame (when in WAV or OMA container)
  Via OpenMG proprietary format (OMA native)

  Binary layout (ID3v2 APIC frame):
    [0x00-0x03]  Frame ID: "APIC" (4 bytes)
    [0x04-0x07]  Frame size (4 bytes, big-endian)
    [0x08-0x09]  Frame flags (2 bytes)
    [0x0A]       Text encoding (1 byte: 1=UTF-16)
    [0x0B-...]   MIME type (null-terminated UTF-16)
    [...]         Picture type (1 byte)
    [...]         Description (null-terminated UTF-16)
    [...]         Binary image data
```

### 7.4 Metadata Compatibility Matrix
| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| ID3v1 | ✗ | ✗ | ✗ | N/A |
| ID3v2.3 | ✓ | ✓ | ✓ | High |
| ID3v2.4 | ✓ | ✓ | ✓ | High |
| OpenMG | ✓ | ✗ | — | Compatibility |
| APEv2 | ✗ | ✗ | ✗ | N/A |

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   atrac3                         # ATRAC3 decoder
                    atrac3al                      # ATRAC3 Advanced Lossless decoder
                    atrac3p                       # ATRAC3plus decoder
                    atrac3pal                     # ATRAC3plus Advanced Lossless decoder
                    atrac9                        # ATRAC9 decoder
AV_CODEC_ID:        AV_CODEC_ID_ATRAC3             # ATRAC3
                    AV_CODEC_ID_ATRAC3AL            # ATRAC3 AL
                    AV_CODEC_ID_ATRAC3P             # ATRAC3plus
                    AV_CODEC_ID_ATRAC3PAL           # ATRAC3plus AL
                    AV_CODEC_ID_ATRAC9              # ATRAC9
Format Name (CLI):  atrac3, oma, aa3, atrac           # file formats
Encoder(s):         NONE — FFmpeg does NOT encode any ATRAC variant
Decoder(s):         atrac3, atrac3al, atrac3p, atrac3pal, atrac9
Muxer(s):           oma (Sony OpenMG Audio)
Demuxer(s):         oma, aa3, atrac (WAV ATRAC)
```

**Critical:** FFmpeg has **no ATRAC encoder**. ATRAC is a proprietary Sony format. FFmpeg decoders are reverse-engineered. There is no open-source encoder.

### 8.2 FFmpeg Encoding — Full CLI Reference

**FFmpeg does NOT encode ATRAC.** Sony licensing required for ATRAC encoding. Sony SonicStage and Connect software provide proprietary encoding capabilities.

```bash
# FFmpeg CANNOT encode ATRAC — only decodes

# ATRAC encoding requires:
# - Sony SonicStage software (Windows, discontinued)
# - Sony Connect software (Windows, discontinued)
# - Sony Media Go software (Windows, discontinued)
# No open-source ATRAC encoder exists
```

### 8.3 FFmpeg Decoding — Full CLI Reference

```bash
# Decode ATRAC3 (WAV-in-ATRAC3) to WAV
ffmpeg -i input.at3 \
  -c:a pcm_s16le \
  -ar 44100 \
  -ac 2 \
  output.wav

# Decode ATRAC3plus (WAV-in-ATRAC3+) to WAV
ffmpeg -i input.at3 \
  -c:a pcm_s16le \
  output.wav

# Decode OMA file to WAV
ffmpeg -i input.oma \
  -c:a pcm_s16le \
  output.wav

# Decode AA3 file to WAV
ffmpeg -i input.aa3 \
  -c:a pcm_s16le \
  output.wav

# Probe format without decoding
ffprobe -v quiet -print_format json -show_streams -show_format input.oma
```

### 8.4 FFmpeg Decoding — libavcodec C API

```c
// ─── ATRAC3 / ATRAC3plus decode program skeleton ──────────────────────────────

// ─── 1. Open file and find stream ───────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.oma", NULL, NULL);
avformat_find_stream_info(fmt_ctx, NULL);

// Find audio stream
int audio_idx = av_find_best_stream(fmt_ctx, AVMEDIA_TYPE_AUDIO, -1, -1, NULL, 0);
AVStream *stream = fmt_ctx->streams[audio_idx];

// ─── 2. Find and open decoder based on codec ID ──────────────────────────────
const AVCodec *dec = NULL;
switch (stream->codecpar->codec_id) {
    case AV_CODEC_ID_ATRAC3:
        dec = avcodec_find_decoder_by_name("atrac3");
        break;
    case AV_CODEC_ID_ATRAC3P:
        dec = avcodec_find_decoder_by_name("atrac3p");
        break;
    case AV_CODEC_ID_ATRAC3AL:
        dec = avcodec_find_decoder_by_name("atrac3al");
        break;
    case AV_CODEC_ID_ATRAC3PAL:
        dec = avcodec_find_decoder_by_name("atrac3pal");
        break;
    case AV_CODEC_ID_ATRAC9:
        dec = avcodec_find_decoder_by_name("atrac9");
        break;
    default:
        fprintf(stderr, "Unknown ATRAC codec ID\n");
        exit(1);
}

if (!dec) { fprintf(stderr, "Decoder not found\n"); exit(1); }

AVCodecContext *dec_ctx = avcodec_alloc_context3(dec);
avcodec_parameters_to_context(dec_ctx, stream->codecpar);
avcodec_open2(dec_ctx, dec, NULL);

// ─── 3. Decode loop ───────────────────────────────────────────────────────────
AVPacket *pkt = av_packet_alloc();
AVFrame  *frm = av_frame_alloc();

while (av_read_frame(fmt_ctx, pkt) >= 0) {
    if (pkt->stream_index == audio_idx) {
        int ret = avcodec_send_packet(dec_ctx, pkt);
        if (ret < 0) {
            fprintf(stderr, "Error sending packet to decoder\n");
            continue;
        }
        while (avcodec_receive_frame(dec_ctx, frm) == 0) {
            // frm->data[0..channels-1] contain audio samples
            // frm->nb_samples = sample count per channel
            // frm->format = AV_SAMPLE_FMT_FLTP (float planar) for ATRAC3+
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

// ─── 4. Cleanup ───────────────────────────────────────────────────────────────
av_frame_free(&frm);
av_packet_free(&pkt);
avcodec_free_context(&dec_ctx);
avformat_close_input(&fmt_ctx);
```

### 8.5 FFmpeg Metadata Handling

```bash
# Read all metadata as JSON
ffprobe -v quiet -print_format json -show_format input.oma | jq .format.tags

# Write metadata (copy audio, update tags)
# Note: Metadata support varies by container format
ffmpeg -i input.oma \
  -c copy \
  -metadata title="Song Title" \
  -metadata artist="Artist Name" \
  -metadata album="Album Name" \
  -metadata track="5" \
  output.oma

# Strip all metadata
ffmpeg -i input.oma -c copy -map_metadata -1 output.oma

# Embed cover art
ffmpeg -i input.oma -i cover.jpg \
  -c copy \
  -map 0:a -map 1:v \
  -metadata:s:v title="Album cover" \
  -disposition:v attached_pic \
  output.oma
```

#### FFmpeg Internal Metadata Key Mapping
| Standard Field | FFmpeg Key | ATRAC Native Key | Notes |
|----------------|------------|--------------------------|-------|
| Title | title | TIT2 | |
| Artist | artist | TPE1 | |
| Album | album | TALB | |
| Track Number | track | TRCK | |
| Genre | genre | TCON | |
| Date/Year | date | TYER | |
| Comment | comment | COMM | |

### 8.6 Quality / Fidelity Decision Table
| Use Case | Settings | Expected Size (min) | Notes |
|----------|----------|---------------------|-------|
| Hi-SP (ATRAC3plus) | 352 kbps | ~23 MB/min | Best quality on Hi-MD |
| Standard (ATRAC3plus) | 128–192 kbps | ~8–12 MB/min | Default |
| LP2 (ATRAC3) | 132 kbps | ~8.5 MB/min | 2× MD recording time |
| LP4 (ATRAC3) | 66 kbps | ~4 MB/min | 4× MD recording time |
| PSP audio (ATRAC3plus) | 128 kbps | ~8 MB/min | Portable standard |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
ATRAC files use container-specific seeking mechanisms:
- **WAV:** Standard WAV seek table (idx1 chunk)
- **OMA:** Proprietary Sony index structure within OMA container
- **AA3:** Minimal container — frame-based seeking

### 9.2 Gapless Playback Data
```
Encoder delay:   ~1024–2048 samples [NEEDS VERIFICATION]
Padding:        ~1024–2048 samples [NEEDS VERIFICATION]
Storage location: Not standardized — container-dependent
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | Frame-based |
| Algorithmic encoder delay | ~58 ms | Estimate based on frame size |
| Live encoding feasible | No | No encoder available |
| HTTP progressive download | Yes | Frame-based |
| HTTP Live Streaming (HLS) | No | Not standard HLS codec |
| DASH streaming | No | Not standard DASH codec |
| WebRTC / RTP transport | No | Not standard RTP codec |
| Minimum decode buffer | 1 frame / ~58 ms | |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | Layout Name | Channel Order | Notes |
|----------|-------------|---------------|-------|
| 1 | Mono | C | |
| 2 | Stereo | L, R | |
| 4 | — | L, R, SL, SR | ATRAC-X multichannel [NEEDS VERIFICATION] |
| 6 | 5.1 | FL, FR, C, LFE, SL, SR | ATRAC-X [NEEDS VERIFICATION] |
| 8 | — | Multiple | ATRAC3plus supports up to 8 channels in channel blocks |

### 11.2 ATRAC3plus Channel Blocks
```
ATRAC3plus encodes audio in channel blocks of 1-2 channels:
  - Channel block 0: channels 0–1 (stereo pair)
  - Channel block 1: channels 2–3
  - Channel block 2: channels 4–5
  - Channel block 3: channels 6–7
  
Maximum 4 channel blocks = 8 channels
Each channel block encoded independently at the same bitrate
```

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 16-bit | Fixed at 16-bit PCM |
| Max sample rate | 48,000 Hz | ATRAC3plus supports 48 kHz |
| Float support | Partial | Internal processing may use float |
| DSD support | No | |
| 24-bit support | No | Not defined |

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

| Platform / API | Encode | Decode | FFmpeg Flag | Notes |
|----------------|--------|--------|-------------|-------|
| NVIDIA NVENC | No | — | — | No hardware ATRAC encoder |
| NVIDIA NVDEC | — | No | — | No hardware ATRAC decoder |
| Intel QSV | No | No | — | Not supported |
| Apple VideoToolbox | No | No | — | No hardware ATRAC support |
| Android MediaCodec | No | No | — | Not supported |
| VA-API (Linux) | No | No | — | Not supported |
| Sony PSP Media Engine | — | Yes | — | Internal hardware decode |
| Sony PS3 Cell | — | Yes | — | Software decode via Cell SPU |

**Note:** ATRAC decoding is done in software on most platforms. Sony PSP had a dedicated Media Engine processor for ATRAC3plus decode. Sony PS3 used the Cell Broadband Engine's SPU for software decode.

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Decoder-Specific Issues
| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| ATRAC requires extradata from container | All | Ensure proper container format |
| ATRAC9 support incomplete | Some | Use latest FFmpeg |
| ATRAC Advanced Lossless incomplete | Some | Decode lossy core only |

### 14.2 Interoperability Issues
- **Files encoded with SonicStage → FFmpeg:** Generally compatible if proper container format used
- **Encrypted OMA files:** FFmpeg cannot decode encrypted OpenMG content
- **Different bitrate modes:** Some hardware players only support specific bitrate/format combinations
- **Hi-MD recordings:** Require proper ATRAC3plus configuration from container

### 14.3 Edge Cases to Handle in Converter
- **Empty file / 0-sample stream:** Return empty output
- **File < 1 frame:** Decode as much as possible
- **All-silence audio:** Decodes correctly
- **Corrupt frame:** Skip frame or mute
- **Encrypted content:** FFmpeg cannot decode — report unsupported
- **Unknown ATRAC variant:** Try each decoder in sequence

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM ATRAC

| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|--------------|
| → FLAC | `ffmpeg -i in.oma -c:a flac -compression_level 8 out.flac` | All tags (via rewrite) | Lossless decode |
| → ALAC | `ffmpeg -i in.oma -c:a alac out.m4a` | Partial | Lossless decode |
| → MP3 | `ffmpeg -i in.oma -c:a libmp3lame -q:a 0 out.mp3` | Tags via ID3v2.3 | Generation loss |
| → AAC | `ffmpeg -i in.oma -c:a aac -b:a 256k out.m4a` | Partial | Generation loss |
| → Opus | `ffmpeg -i in.oma -c:a libopus -b:a 128k out.opus` | Vorbis Comments | Generation loss |
| → WAV | `ffmpeg -i in.oma -c:a pcm_s16le out.wav` | RIFF INFO | Lossless decode |
| → OGG Vorbis | `ffmpeg -i in.oma -c:a libvorbis -q:a 6 out.ogg` | Vorbis Comments | Generation loss |

### 15.2 Converting TO ATRAC

| Source | Command | Metadata Preserved | Quality Notes |
|--------|---------|-------------------|--------------|
| Any → | **Not possible** | — | FFmpeg has no ATRAC encoder; Sony SonicStage required |

### 15.3 Lossless Round-Trip Verification
```bash
# Decode to WAV (lossless decode)
ffmpeg -i input.oma -c:a pcm_s16le decoded.wav

# Lossless verification (ATRAC is lossy — no true lossless round-trip)
# Compare decoded output with reference decode
ffmpeg -i input.oma -map 0:a -f framemd5 input.md5
diff input.md5 reference.md5
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| FFmpeg (ATRAC3/3+) | C | LGPL 2.1+ | — (no encoder) | 8/10 | https://ffmpeg.org |
| FFmpeg (ATRAC9) | C | LGPL 2.1+ | — (no encoder) | 7/10 | https://ffmpeg.org |
| Rockbox ATRAC3 | C | GPL | — | 8/10 | https://www.rockbox.org |
| Sony SonicStage | Proprietary | Proprietary | Reference | Reference | Discontinued |
| Atracdenc | C | BSD | 7/10 | — | Open ATRAC3 encoder |
| sceAtrac (PSP) | C (PSP library) | Proprietary | Reference | Reference | PPSSPP emulator |

### Build Instructions
```bash
# FFmpeg includes ATRAC decoders by default
# No additional configure flags needed:
./configure
make -j$(nproc)

# ATRAC9 decoder support requires recent FFmpeg (2023+)
ffmpeg -decoders | grep atrac

# Build Atracdenc (open ATRAC3 encoder — NOT Sony ATRAC3plus):
git clone https://github.com/Distensive/atracdenc.git
cd atracdenc
mkdir build && cd build
cmake ..
make -j$(nproc)
# Note: Atracdenc encodes ATRAC3, NOT ATRAC3plus
```

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **Sony ATRAC3plus Technical Documentation:** http://www.sony.net/Products/ATRAC3/tech/ (discontinued link)
- **Multimedia Wiki ATRAC3plus:** https://wiki.multimedia.cx/index.php/ATRAC3plus
- **Multimedia Wiki ATRAC3:** https://wiki.multimedia.cx/index.php/ATRAC3
- **PPSSPP sceAtrac documentation:** https://www.ppsspp.org/docs/development/ppsspp-internals/atrac/

### Technical Resources
- FFmpeg ATRAC documentation: https://ffmpeg.org/ffmpeg-codecs.html
- Hydrogenaudio ATRAC discussion: https://hydrogenaud.io/
- Sony MiniDisc Wikipedia: https://en.wikipedia.org/wiki/MiniDisc
- ATRAC Wikipedia: https://en.wikipedia.org/wiki/Adaptive_Transform_Acoustic_Coding
- HandWiki ATRAC: https://handwiki.org/wiki/Adaptive_Transform_Acoustic_Coding

### Academic Papers
- *Adaptive Transform Acoustic Coding (ATRAC) — Sony Technical Paper*, Sony Corporation
- Prasada, B. & Wilson, G., *Analysis of Sony's ATRAC Compression Algorithm*, AES Convention, 1996
- Nakagawa, Y. et al., *ATRAC3: A High-Quality Audio Coding System for Network Distribution*, AES 103rd Convention, 1997

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] FFmpeg decoders enabled by default — verify with `ffmpeg -decoders | grep atrac`
- [ ] FFmpeg has NO ATRAC encoder — conversion TO ATRAC is not possible
- [ ] Verify `ffprobe` correctly identifies ATRAC3, ATRAC3plus, ATRAC9 files
- [ ] Handle OMA, AA3, ATRAC-in-WAV containers
- [ ] Note: Encrypted OpenMG content is not decodable

### Encoding Pipeline
- [ ] **CRITICAL:** FFmpeg cannot encode ATRAC — Sony SonicStage or Connect software required
- [ ] Cannot implement ATRAC encoding in open-source converter
- [ ] Warn user that ATRAC output is not supported

### Decoding Pipeline
- [ ] Auto-detect ATRAC variant based on codec_id from container
- [ ] Handle ATRAC3, ATRAC3plus, ATRAC3 AL, ATRAC3plus AL, ATRAC9 decoders
- [ ] Handle AVERROR(EAGAIN) in receive_frame loop correctly
- [ ] Flush decoder at EOF (send NULL packet)
- [ ] Handle float planar output format (AV_SAMPLE_FMT_FLTP) for ATRAC3+
- [ ] Handle signed 16-bit output for ATRAC3

### Metadata
- [ ] Read ID3v2 tags from OMA/WAV containers
- [ ] Map ID3v2 fields to standard names (Section 7.2)
- [ ] Read cover art (APIC frame) and preserve as JPEG/PNG binary
- [ ] Write ID3v2 tags to output container
- [ ] Handle OpenMG proprietary metadata (read-only)
- [ ] Handle UTF-16 encoded metadata from Sony software

### Quality & Verification
- [ ] Lossless verification only applies to decode step (ATRAC is lossy)
- [ ] Implement progress reporting from FFmpeg
- [ ] Test with ATRAC3, ATRAC3plus, ATRAC9 files
- [ ] Test with encrypted vs unencrypted OMA files

### Edge Cases
- [ ] Handle files with corrupt or missing headers
- [ ] Handle files with 0 samples
- [ ] Handle sample rate mismatch (resample via libswresample)
- [ ] Handle mono vs stereo vs multichannel
- [ ] Handle encrypted OpenMG content (reject — not supported)
- [ ] Handle unknown ATRAC variant (try decoders in sequence)
- [ ] Handle channel block configurations in multichannel ATRAC3plus

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
