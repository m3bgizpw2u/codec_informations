# 3GP / 3GPP Multimedia Container — Deep Technical Reference

> **Category:** Container
> **File Extensions:** `.3gp`, `.3gpp`, `.3g2`
> **MIME Types:** `video/3gpp`, `audio/3gpp`, `video/3gpp2`
> **Standardization Body:** 3GPP (3GPP2 for .3g2 variant)
> **Primary Specification:** 3GPP TS 26.244; ISO/IEC 14496-12 (ISO Base Media File Format)
> **Patent Status:** Patented — 3GPP patent pool managed by ETSI
> **License:** Royalty-bearing for commercial implementations
> **Current Version:** 3GPP Release 17 (2022)
> **Active Development:** Yes — maintained as part of 3GPP specifications

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Third Generation Partnership Project (3GPP), a collaboration between ARIB, ATIS, CCSA, ETSI, TTA, and TTC
- **Year Created:** 1998 (original 3GPP); 3GPP file format formally specified ~2000
- **Original Purpose:** Define a compact multimedia container for 3G mobile phones supporting video, audio, and timed text over UMTS networks, optimized for limited bandwidth, storage, and processing power
- **Problem with Predecessors:** Existing file formats (AVI, MOV) were too large, complex, and bandwidth-hungry for mobile networks; a simplified ISOBMFF derivative was needed

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| Release 4 | 2001 | Initial 3GP specification — H.263, AMR-NB, AAC |
| Release 5 | 2002 | Added AMR-WB, H.263 profile extensions |
| Release 6 | 2005 | Added H.264/AVC, timed text, HE-AAC |
| Release 7 | 2007 | Added H.264 Baseline Profile improvements, AMR-WB+ |
| Release 8 | 2008 | Added H.265/HEVC (TV profile) |
| Release 9–17 | 2010–2022 | Continued additions: HEVC, EVS codec, additional metadata extensions |

### 1.3 Current Adoption
- **Primary use cases today:** Mobile phone video, MMS attachments, video messaging, early smartphone video recording
- **Platforms with native support:** Symbian, early Android, feature phones, some smartphone platforms
- **Major services using this format:** Mobile carriers (MMS), early smartphone video
- **Hardware support:** Broad legacy support in mobile SoCs; declining for new devices
- **Status:** Legacy — largely superseded by MP4/M4A for new content, but still widely encountered in archives

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Container type:** ISO Base Media File Format (ISO/IEC 14496-12) derivative
- **Box-based structure:** Hierarchical composition of "boxes" (also called atoms in MP4)
- **No audio codec per se:** 3GP is a container; it wraps audio and video codecs

### 2.2 High-Level Container Structure (Block Diagram)
```
3GP File (.3gp)
│
├── ftyp box (File Type Box) — brand identification
│     offset 0x0000, mandatory, first box in file
│
├── moov box (Movie Box) — metadata container
│     ├── mvhd (Movie Header Box)
│     ├── trak (Track Box, one per track)
│     │     ├── tkhd (Track Header Box)
│     │     ├── mdia (Media Box)
│     │     │     ├── mdhd (Media Header Box)
│     │     │     ├── hdlr (Handler Box)
│     │     │     └── minf (Media Information Box)
│     │     │           ├── smhd (Sound Media Header, audio)
│     │     │           ├── dinf (Data Information Box)
│     │     │           └── stbl (Sample Table Box)
│     │     │                 ├── stsd (Sample Description)
│     │     │                 ├── stts (Time to Sample)
│     │     │                 ├── stsc (Sample to Chunk)
│     │     │                 ├── stsz (Sample Size)
│     │     │                 └── stco (Chunk Offset)
│     │     └── udta (User Data Box, optional metadata)
│     └── meta (Metadata Box, optional)
│
├── moof box (Movie Fragment Box, optional) — for streaming
│     ├── mfhd (Movie Fragment Header)
│     └── traf (Track Fragment Box)
│
├── mdat box (Media Data Box) — raw audio/video payload
│
└── [Additional boxes]
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Container basis | ISO/IEC 14496-12 | Same family as MP4, MOV, M4A |
| Box alignment | 8-byte aligned for full-box, 4-byte for basic-box | |
| File extension | .3gp, .3gpp, .3g2 | .3g2 is 3GPP2 variant |
| Maximum file size | Implementation-dependent | No hard limit in spec |
| Streamable | Yes (with moof fragments) | Progressive download supported |
| Seeking | By index table in stbl | Requires moov present |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       00 00 XX XX      ....      First box size (varies)
0x0004  4       66 74 79 70     'ftyp'    Box type: File Type Box
```

No global magic bytes exist for 3GP. File identification relies on the `ftyp` box's brand field.

### 3.2 File-Level Header Layout — ftyp Box
```
Offset   Size    Field Name              Type        Valid Range         Description
-------  ------  ---------------------  ----------  -----------------   ---------------------------
0x0000   4B      Box Size               uint32 BE   > 0                 Total box size in bytes
0x0004   4B      Box Type               char[4]     'ftyp'             Box identifier
0x0008   4B      Major Brand            char[4]     '3gp4','3gp5',     Primary brand identifier
                                                   '3gp6','3gp7',     (Release 4-7)
                                                   '3g2a','3g2b',     (3GPP2 brands)
                                                   'isom','mp42'      (MP4 compatibility)
0x000C   4B      Minor Version          uint32 BE   varies             Brand version number
0x0010   4B×N    Compatible Brands      char[4]×N   various           List of compatible brands
```

#### Common 3GPP Brand Identifiers
| Brand | Specification | Description |
|-------|---------------|-------------|
| `3gp4` | Release 4 | H.263 and AMR, basic profile |
| `3gp5` | Release 5 | Extensions for H.263 and AMR |
| `3gp6` | Release 6 | H.264/AVC support, timed text |
| `3gp7` | Release 7 | Enhanced H.264, AMR-WB+ |
| `3gr6` | Release 6 | GSM-evolved profile (H.263 only) |
| `3gr7` | Release 7 | GSM-evolved profile |
| `isom` | ISO Base | Compatibility with MP4 |
| `mp42` | MP4 v2 | Compatibility brand |

#### 3GPP2 Brand Identifiers
| Brand | Specification | Description |
|-------|---------------|-------------|
| `3g2a` | 3GPP2 C.S0050-0 | 3GPP2 Media, version 1.0 |
| `3g2b` | 3GPP2 C.S0050-A | 3GPP2 Media, version A |
| `3g2c` | 3GPP2 C.S0050-B | 3GPP2 Media, version B |

### 3.3 moov/mvhd Box Layout
```
Offset   Size    Field Name              Type        Description
-------  ------  ---------------------  ----------  ---------------------------
0x0000   4B      Box Size               uint32 BE   Total box size
0x0004   4B      Box Type              char[4]     'moov'
0x0008   4B      Box Size               uint32 BE   'mvhd' sub-box size
0x000C   4B      Box Type              char[4]     'mvhd'
0x0010   1B      Version                uint8       0 or 1
0x0011   3B      Flags                  uint24      Bit 0: track_in_poster
0x0014   4/16B   Creation Time          uint32/64   Seconds since midnight Jan 1 1904
0x0018   4/16B   Modification Time      uint32/64   Same timebase
0x001C   4B      Time Scale            uint32       e.g., 90000 for video, 44100/48000 for audio
0x0020   8/16B   Duration              uint32/64   In timescale units
0x0028   4B      Rate                  fixed32      e.g., 0x00010000 = 1.0
0x002C   2B      Volume                fixed16      e.g., 0x0100 = 1.0 (full volume)
0x002E   10B     Reserved               —           Zeros
0x0038   36B     Matrix                uint32[9]   Transformation matrix
0x0060   24B     Pre-defined            —           Zeros
0x0078   4B      Next Track ID         uint32      Incrementing ID
```

### 3.4 Sample Entry Boxes for Audio in 3GP

#### AMRSampleEntry (for AMR-NB, AMR-WB)
```
Box type: 'samr' (AMR-NB) or 'sawb' (AMR-WB)
Parent: stsd inside stbl

Offset   Size    Field Name              Type        Description
-------  ------  ---------------------  ----------  ---------------------------
0x0000   4B      Box Size               uint32 BE   Total box size
0x0004   4B      Box Type               char[4]     'samr' or 'sawb'
0x0008   6B      Reserved               uint8[6]    Zeros
0x000E   2B      Data Reference Index   uint16 BE   Index into dref box
0x0010   2B      Reserved               uint8[2]    Zeros
0x0012   2B      Channel Count          uint16 BE   1 (AMR is mono)
0x0014   2B      Sample Size            uint16 BE   16 (bits per sample)
0x0018   2B      Pre-defined            int16       0
0x001A   4B      Reserved               int32       0
0x001E   4B      AMR Specific Box Type   char[4]    'damr'
0x0022   4B      AMR Specific Box Size  uint32 BE  Size of AMR box
0x0026   4B      Vendor Code            uint32 LE  Encoder vendor
0x002A   1B      Encoder Version        uint8       0
0x002B   1B      Mode Set               uint16 LE   Bitmask of enabled modes
0x002C   1B      Mode Change Period     uint8       Mode change frequency
0x002D   1B      Frames Per Sample      uint8       Usually 1
```

#### MP4AudioSampleEntry (for AAC, HE-AAC)
```
Box type: 'mp4a'
Same as ISO Base Media File Format for AAC audio.
Contains:
  - esds box (Elementary Stream Descriptor) with ASC (Audio Specific Config)
  - Specific Box for sample rate and channel configuration
```

### 3.5 AMR Frame Storage in 3GP

AMR audio in 3GP uses the AMR IF2 (Interface Format 2) storage format, which is byte-aligned:

```
AMR Frame in 3GP (IF2 format):
  Byte 0: Frame Type (4 bits) + Quality Indicator (1 bit) + Padding (3 bits)
  Bytes 1-N: AMR speech bits (Class A, Class B, Class C)
  
Frame Type values (4 bits):
  0x0 = AMR 4.75 kbps    — 13 octets total
  0x1 = AMR 5.15 kbps    — 14 octets total
  0x2 = AMR 5.90 kbps    — 16 octets total
  0x3 = AMR 6.70 kbps    — 18 octets total
  0x4 = AMR 7.40 kbps    — 19 octets total
  0x5 = AMR 7.95 kbps    — 21 octets total
  0x6 = AMR 10.20 kbps   — 26 octets total
  0x7 = AMR 12.20 kbps   — 31 octets total
  0x8 = AMR SID (CNG)    — 6 octets total
  0xF = No Data          — 1 octet total
```

---

## 4. ENCODING ALGORITHM — CONTAINER OPERATIONS

### 4.1 Box Writing Order
```
1. Write ftyp box (first, mandatory)
2. Write moov box (metadata, second)
     ├── mvhd (movie header)
     ├── trak (track, repeated)
     │     ├── tkhd (track header)
     │     ├── mdia (media)
     │     │     ├── mdhd (media header)
     │     │     ├── hdlr (handler = 'soun' for audio)
     │     │     └── minf
     │     │           ├── smhd (sound media header)
     │     │           ├── dinf → dref → url (data reference)
     │     │           └── stbl
     │     │                 ├── stsd (sample description)
     │     │                 ├── stts (time-to-sample)
     │     │                 ├── stsc (sample-to-chunk)
     │     │                 ├── stsz (sample sizes)
     │     │                 └── stco (chunk offsets)
     │     └── udta (user data, optional)
3. Write mdat box (media data)
```

### 4.2 Sample Entry Identification in 3GP
| Audio Format | Sample Entry Type | Box Code | Notes |
|-------------|-------------------|----------|-------|
| AMR-NB | AMRSampleEntry | 'samr' | 3GPP-specific |
| AMR-WB | AMRSampleEntry | 'sawb' | 3GPP-specific |
| AMR-WB+ | AMRWPSampleEntry | 'sawp' | 3GPP-specific |
| AAC-LC | MP4AudioSampleEntry | 'mp4a' | Standard ISOBMFF |
| HE-AAC | MP4AudioSampleEntry | 'mp4a' | Same entry, SBR in ESDS |
| MP3 | MPEGAudioSampleEntry | 'mp4a' | Standard ISOBMFF |

### 4.3 muxer Settings / Profiles

#### 3GPP Release 4 Profile Constraints
| Feature | Constraint |
|---------|------------|
| Video | H.263 Profile 0 Level 10 or MPEG-4 SP |
| Audio | AMR-NB only |
| Max video bitrate | 64 kbps |
| Max audio bitrate | 12.2 kbps |

#### 3GPP Release 6 Profile Constraints
| Feature | Constraint |
|---------|------------|
| Video | H.264/AVC Baseline Profile Level 1B |
| Audio | AMR-NB, AMR-WB, AAC-LC, HE-AAC v1, HE-AAC v2 |
| Max video frame size | QCIF (176×144) |
| Max frame rate | 15 fps |

---

## 5. DECODING ALGORITHM — CONTAINER PARSING

### 5.1 Stream Synchronization & File Identification

#### File Identification
```
1. Read first 8 bytes: box size + box type
2. If type == 'ftyp', this is a valid ISOBMFF-family file
3. Parse major_brand: '3gp4','3gp5','3gp6','3gp7' → 3GP
                                '3g2a','3g2b','3g2c' → 3G2
4. Check compatible_brands list for 3GPP/3GPP2 brand presence
5. Parse moov box to extract stream parameters
```

#### Seeking in 3GP
```
Seeking requires:
  1. moov box must be present and parsed
  2. stbl contains:
       - stsz: sample sizes
       - stco/stsc: chunk-to-offset mapping
       - stts: time-to-sample mapping
  3. Compute byte offset = stco[chunk] + stsc[sample].offset
  4. Read mdat payload at computed offset
```

### 5.2 Core Container Decode Pipeline
```
1. Read ftyp box
     ├── Validate major brand
     └── Read compatible brands list

2. Read moov box
     ├── Read mvhd: timescale, duration, next_track_id
     ├── For each trak:
     │     ├── Read tkhd: track_id, enabled, volume
     │     ├── Read mdia → hdlr: handler_type ('soun' for audio)
     │     ├── Read mdhd: time_scale, duration
     │     └── Read minf → smhd: balance (L/R), reserved
     └── Read stbl:
           ├── stsd: codec type, sample entry
           ├── stts: sample count, time delta table
           ├── stsc: chunk/sample mapping
           ├── stsz: sample size array
           └── stco: chunk offset array

3. Read mdat box
     └── Extract raw codec frames

4. Decode audio frames using identified codec
```

### 5.3 Error Concealment
- **Corrupt box detection:** Box size of 0, size < header, or size extending past EOF → skip or abort
- **Missing moov:** Progressive playback impossible; some players attempt streaming from moof
- **Concealment method:** For audio, mute on decode error; for video, freeze previous frame

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** 3GP itself is an ISOBMFF container
- **Overhead:** ~50–200 bytes per track (moov structure), independent of media bitrate
- **Seeking in native container:** Yes — by index table in stbl
- **Multiple streams in native container:** Yes — audio + video + timed text simultaneously

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| 3GP (.3gp) | AMR-NB | Yes | Limited (3GP boxes) | Native |
| 3GP (.3gp) | AMR-WB | Yes | Limited | Native |
| 3GP (.3gp) | AAC-LC | Yes | Limited | Native |
| 3GP (.3gp) | HE-AAC | Yes | Limited | Via ESDS SBR |
| 3GP (.3gp) | H.263 | Yes | Limited | Native |
| 3GP (.3gp) | H.264/AVC | Yes | Limited | Via avcC box |
| 3G2 (.3g2) | EVRC | Yes | Limited | 3GPP2-specific |
| 3G2 (.3g2) | SMV | Yes | Limited | 3GPP2-specific |
| 3G2 (.3g2) | QCELP | Yes | Limited | 3GPP2-specific |
| MP4/M4A | All 3GP codecs | Yes | Full (iTunes atoms) | Compatible |
| Matroska/MKA | Some 3GP codecs | Yes | Full | Not recommended |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** 3GPP-specific metadata boxes within the moov/udta hierarchy
- **Tag block location:** Inside moov/udta box, or standalone meta box
- **Tag block identifier:** `meta` box containing `keys` and `ilst` sub-boxes (iTunes-style)
- **3GPP-specific metadata:** `UCDT` (User-Defined Characteristic Tags), `TITL`, `AUTH`, etc.

### 7.2 Standard Tag Fields — 3GP/iTunes-style Metadata
| Tag Field | Internal Key (3GP) | Max Length | Character Encoding | Multi-value | Notes |
|-----------|---------------------|------------|-------------------|-------------|-------|
| Title | '©nam' | variable | UTF-8 | No | |
| Artist | '©ART' | variable | UTF-8 | No | |
| Album | '©alb' | variable | UTF-8 | No | |
| Album Artist | 'aART' | variable | UTF-8 | No | |
| Genre | '©gen' | variable | UTF-8 | No | |
| Year / Date | '©day' | 4 bytes | ASCII | No | YYYY format |
| Track Number | 'trkn' | 4 bytes | binary | No | 2-byte track + 2-byte total |
| Disc Number | 'disk' | 4 bytes | binary | No | 2-byte disc + 2-byte total |
| Comment | '©cmt' | variable | UTF-8 | No | |
| Copyright | 'cprt' | variable | UTF-8 | No | |
| Encoder | '©enc' | variable | UTF-8 | No | Software name |
| Cover Art | 'covr' | up to 10 MB | Binary JPEG/PNG | No | Embedded in mdat |
| Description | 'desc' | variable | UTF-8 | No | 3GPP-specific |
| Classification | 'clsf' | variable | UTF-8 | No | 3GPP-specific |
| Rating | 'rtng' | 1 byte | binary | No | 3GPP-specific |
| Location | 'loc ' | variable | UTF-8 | Yes | GPS metadata |

### 7.3 Cover Art Storage in 3GP
```
Cover art in 3GP using iTunes-style metadata:
  Container: meta/ilst/covr
  Image formats: JPEG (preferred), PNG, BMP
  Max image size: ~10 MB (implementation limit)
  Max dimensions: 2048×2048 pixels (recommendation)
  
  Binary layout (covr item):
    [0x00-0x03]  Item flags (4 bytes, big-endian)
                   bit 0: read-only flag
                   bits 1-31: reserved
    [0x04-0x07]  Data length (4 bytes, big-endian)
    [0x08...]    Binary image data (JPEG/PNG)
```

### 7.4 Metadata Compatibility Matrix
| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| 3GP native boxes | ✓ | ✓ | ✓ | Highest |
| iTunes atoms | ✓ | ✓ | ✓ | High |
| ID3v2 (at end) | ✗ | ✗ | ✗ | N/A |
| APEv2 (at end) | ✗ | ✗ | ✗ | N/A |

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   amr_nb (AMR narrow-band)
                    amr_wb (AMR wide-band)
                    aac (AAC in 3GP)
                    libfdk_aac (Fraunhofer AAC)
AV_CODEC_ID:        AV_CODEC_ID_AMR_NB
                    AV_CODEC_ID_AMR_WB
                    AV_CODEC_ID_AAC
                    AV_CODEC_ID_AAC_LC
Format Name (CLI):  3gp (3GPP format)
                    3g2 (3GPP2 format)
Encoder(s):         amr_nb (decoder only in free FFmpeg)
                    aac
                    libfdk_aac
Decoder(s):         amr_nb, amr_wb, aac, libfdk_aac, libfaad
Muxer(s):           3gp, 3g2
Demuxer(s):         3gp, 3g2, ipod (MP4-compatible)
```

### 8.2 FFmpeg Encoding — Full CLI Reference

#### Encoding to 3GP with AMR-NB
```bash
# Encode audio to AMR-NB in 3GP container
ffmpeg -i input.wav \
  -c:a amr_nb \
  -ar 8000 \
  -ac 1 \
  -f 3gp \
  output.3gp

# Encode audio to AMR-WB in 3GP container
ffmpeg -i input.wav \
  -c:a amr_wb \
  -ar 16000 \
  -ac 1 \
  -f 3gp \
  output.3gp

# Encode audio to AAC in 3GP container
ffmpeg -i input.wav \
  -c:a aac \
  -b:a 48k \
  -ar 44100 \
  -ac 2 \
  -f 3gp \
  output.3gp
```

#### Encoding Video + Audio to 3GP
```bash
# Encode H.263 video + AMR audio to 3GP
ffmpeg -i input.avi \
  -c:v h263p \
  -b:v 64k \
  -r 15 \
  -c:a amr_nb \
  -ar 8000 \
  -ac 1 \
  -f 3gp \
  -brand 3gp6 \
  output.3gp

# Encode H.264 video + AAC audio to 3GP
ffmpeg -i input.avi \
  -c:v libx264 \
  -profile:v baseline \
  -level 1.1 \
  -b:v 256k \
  -r 15 \
  -vf "scale=176:144" \
  -c:a aac \
  -b:a 48k \
  -ar 44100 \
  -f 3gp \
  output.3gp
```

#### Complete FFmpeg Option Table
| Option | Type | Default | Range / Valid Values | Effect on Output |
|--------|------|---------|---------------------|-----------------|
| `-f 3gp` | flag | — | — | Force 3GP container format |
| `-f 3g2` | flag | — | — | Force 3GPP2 container format |
| `-brand` | string | auto | 3gp4, 3gp5, 3gp6, 3gp7, 3gr6, 3g2a | Override major brand |
| `-ar` | int | 8000/16000 | 8000, 16000 | Audio sample rate |
| `-ac` | int | 1 | 1, 2 | Audio channel count |
| `-b:a` | int | varies | 4.75k–12.2k for AMR | Audio bitrate |
| `-movflags` | flags | — | +faststart, +disabledgp | MP4-family flags |

### 8.3 FFmpeg Encoding — libavformat C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/opt.h>

// ─── 1. Allocate output context ──────────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_alloc_output_context2(&fmt_ctx, NULL, "3gp", "output.3gp");
if (!fmt_ctx) { fprintf(stderr, "Could not allocate context\n"); exit(1); }

// ─── 2. Add audio stream ────────────────────────────────────────────────────
AVStream *audio_st = avformat_new_stream(fmt_ctx, NULL);
if (!audio_st) { exit(1); }

// Find and open AMR-NB encoder
const AVCodec *audio_codec = avcodec_find_encoder(AV_CODEC_ID_AMR_NB);
AVCodecContext *audio_ctx = avcodec_alloc_context3(audio_codec);
audio_ctx->sample_rate = 8000;
audio_ctx->channels = 1;
av_channel_layout_default(&audio_ctx->ch_layout, 1);
audio_ctx->bit_rate = 12200; // AMR 12.2 kbps mode

AVDictionary *opts = NULL;
av_dict_set(&opts, "amr_mode", "12", 0); // Set AMR mode
int ret = avcodec_open2(audio_ctx, audio_codec, &opts);

// Copy codec params to stream
avcodec_parameters_from_context(audio_st->codecpar, audio_ctx);
audio_st->time_base = (AVRational){1, 8000};

// ─── 3. Open output file ────────────────────────────────────────────────────
if (!(fmt_ctx->oformat->flags & AVFMT_NOFILE)) {
    ret = avio_open(&fmt_ctx->pb, "output.3gp", AVIO_FLAG_WRITE);
}

// ─── 4. Write header ────────────────────────────────────────────────────────
// Set 3GP-specific brand
AVDictionary *fmt_opts = NULL;
av_dict_set(&fmt_opts, "brand", "3gp6", 0);

avformat_write_header(fmt_ctx, &fmt_opts);

// ─── 5. Encode and write frames ─────────────────────────────────────────────
AVFrame *frame = av_frame_alloc();
frame->format = AV_SAMPLE_FMT_S16;
frame->sample_rate = 8000;
frame->ch_layout.nb_channels = 1;
frame->nb_samples = audio_ctx->frame_size;
av_frame_get_buffer(frame, 0);

AVPacket *pkt = av_packet_alloc();
while (reading_samples) {
    // Fill frame->data with PCM samples
    encode_audio_frame(audio_ctx, frame, pkt);
    if (pkt->size > 0) {
        pkt->stream_index = audio_st->index;
        av_interleaved_write_frame(fmt_ctx, pkt);
    }
}

// Flush encoder
avcodec_send_frame(audio_ctx, NULL);
while (avcodec_receive_packet(audio_ctx, pkt) == 0) {
    av_interleaved_write_frame(fmt_ctx, pkt);
}

// ─── 6. Write trailer ───────────────────────────────────────────────────────
av_write_trailer(fmt_ctx);

// ─── 7. Cleanup ────────────────────────────────────────────────────────────
av_frame_free(&frame);
av_packet_free(&pkt);
avcodec_free_context(&audio_ctx);
avio_closep(&fmt_ctx->pb);
avformat_free_context(fmt_ctx);
```

### 8.4 FFmpeg Decoding — Full CLI Reference

```bash
# Decode 3GP to WAV (all codecs)
ffmpeg -i input.3gp \
  -c:a pcm_s16le \
  output.wav

# Extract audio only, convert to MP3
ffmpeg -i input.3gp \
  -vn \
  -c:a libmp3lame \
  -q:a 2 \
  output.mp3

# Extract specific stream from multi-track 3GP
ffmpeg -i input.3gp -map 0:a:0 -c:a copy output.aac

# Probe format details
ffprobe -v quiet -print_format json -show_streams -show_format input.3gp
```

### 8.5 FFmpeg Decoding — libavformat C API

```c
// ─── Complete decode program skeleton ────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.3gp", NULL, NULL);
avformat_find_stream_info(fmt_ctx, NULL);

// Find audio stream
int audio_idx = av_find_best_stream(fmt_ctx, AVMEDIA_TYPE_AUDIO, -1, -1, NULL, 0);
AVStream *stream = fmt_ctx->streams[audio_idx];

// Find and open decoder
const AVCodec *dec = avcodec_find_decoder(stream->codecpar->codec_id);
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
            // frm->data[0] contains PCM samples
            // frm->nb_samples = sample count per channel
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

### 8.6 FFmpeg Metadata Handling

```bash
# Read all metadata as JSON
ffprobe -v quiet -print_format json -show_format input.3gp | jq .format.tags

# Write metadata to 3GP
ffmpeg -i input.3gp \
  -c copy \
  -metadata title="Song Title" \
  -metadata artist="Artist Name" \
  -metadata album="Album Name" \
  -metadata date="2024" \
  -metadata genre="Music" \
  -metadata description="Description" \
  output.3gp

# Strip all metadata
ffmpeg -i input.3gp -c copy -map_metadata -1 output.3gp

# Embed cover art (if supported)
ffmpeg -i input.3gp -i cover.jpg \
  -c copy \
  -map 0:a -map 1:v \
  -metadata:s:v title="Album cover" \
  output.3gp
```

#### FFmpeg Internal Metadata Key Mapping
| Standard Field | FFmpeg Key | 3GP Native Key | Notes |
|----------------|------------|------------------------|-------|
| Title | title | ©nam | |
| Artist | artist | ©ART | |
| Album | album | ©alb | |
| Album Artist | album_artist | aART | |
| Track Number | track | trkn | Binary 2+2 bytes |
| Disc Number | disc | disk | Binary 2+2 bytes |
| Genre | genre | ©gen | |
| Date/Year | date | ©day | YYYY format |
| Comment | comment | ©cmt | |
| Copyright | copyright | cprt | |
| Encoder | encoder | ©enc | |
| Description | description | desc | 3GPP-specific |

### 8.7 Quality / Fidelity Decision Table
| Use Case | Settings | Expected Size | Notes |
|----------|----------|----------------|-------|
| Mobile MMS | AMR-NB 4.75–7.95 kbps | ~5–10 KB/sec | Voice only |
| Mobile video (basic) | H.263 + AMR-NB | ~20–50 KB/sec | Low quality |
| Mobile video (good) | H.264 + AAC 48k | ~50–150 KB/sec | Modern phones |
| Archival conversion | AAC 128k + H.264 | >200 KB/sec | Use MP4 instead |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
```
3GP seek index: stbl box
  Location:    Within moov/trak/mdia/minf/stbl
  Magic:      N/A (no separate magic, part of box structure)
  Entry size: Variable per stbl sub-box
  
  stsz box (Sample Size):
    [0x00-0x03] 4 bytes: sample_size (0 = variable)
    [0x04-0x07] 4 bytes: sample_count
    [0x08-...]  4 bytes × N: array of sample sizes

  stco box (Chunk Offset):
    [0x00-0x03] 4 bytes: entry_count
    [0x04-...]  8 bytes × N: byte offsets from file start

  stsc box (Sample-to-Chunk):
    [0x00-0x03] 4 bytes: entry_count
    [0x04-...] 12 bytes × N: first_chunk, samples_per_chunk, sample_desc_index

  stts box (Time-to-Sample):
    [0x00-0x03] 4 bytes: entry_count
    [0x04-...]  8 bytes × N: sample_count, sample_delta
```

### 9.2 Gapless Playback Data
```
3GP does not natively store gapless playback data.
Encoder delay:  Not stored; application must handle
Padding:        Not stored
Storage:        No dedicated field
Workaround:     Convert to MP4/M4A for gapless support
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | Requires moov at start or streaming via moof |
| Algorithmic encoder delay | 0 ms for container | Codec delay varies |
| Live encoding feasible | Yes | With moof fragments |
| HTTP progressive download | Yes | If moov precedes mdat |
| HTTP Live Streaming (HLS) | No | Use MP4 segments instead |
| DASH streaming | Yes | With mp4dash or ismv |
| WebRTC / RTP transport | Yes | Via RFC 3267 for AMR |
| Minimum decode buffer | 1 frame | AMR: 20 ms |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | Layout Name | Channel Order | FFmpeg Layout Constant | Notes |
|----------|-------------|---------------|------------------------|-------|
| 1 | Mono | C | AV_CHANNEL_LAYOUT_MONO | AMR default |
| 2 | Stereo | L, R | AV_CHANNEL_LAYOUT_STEREO | AAC default |
| 1 | Mono | C | AV_CHANNEL_LAYOUT_MONO | AMR-WB default |

### 11.2 Downmix Coefficients
```
3GP audio is typically mono (AMR) or stereo (AAC).
No standardized downmix metadata within 3GP container.
Downmix handled at decoder/application level.
```

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 16-bit | AMR is 16-bit; AAC supports up to 24-bit in spec |
| Max sample rate | 48,000 Hz | AAC supports up to 48 kHz in 3GP |
| Float support | No | 3GP targets fixed-point mobile hardware |
| DSD support | No | Not applicable |
| 24-bit support | Partial | Only through AAC, not AMR |
| 20-bit support | Partial | Through AAC |

```bash
# High-res AAC in 3GP example
ffmpeg -i input_24bit_48k.wav \
  -c:a aac \
  -b:a 256k \
  -ar 48000 \
  -ac 2 \
  -f 3gp \
  output.3gp
```

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

| Platform / API | Encode | Decode | FFmpeg Flag | Notes |
|----------------|--------|--------|-------------|-------|
| Mobile SoC DSP | Yes | Yes | Hardware-specific | Most mobile phones have HW AMR |
| Android MediaCodec | Yes | Yes | `-c:a aac_mediacodec` | AAC HW encode on Android |
| Apple VideoToolbox | Partial | Yes | `-c:a aac_at` | AAC HW on iOS |
| Intel QSV | No | No | N/A | Not typical mobile use |
| VA-API (Linux) | No | No | N/A | Desktop only |

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Encoder-Specific Issues
| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| AMR encoder not in free build | All | Use libopencore-amrnb |
| 3GP muxer brand auto-selection | < 4.0 | Use `-brand` flag explicitly |
| Seeking in fragmented 3GP | < 5.0 | Rebuild with `-movflags +faststart` |
| HE-AAC SBR in 3GP | < 3.0 | Use MP4 container instead |

### 14.2 Interoperability Issues
- **3GP → specific players:** Some feature phones only play files with specific brand identifiers
- **AMR in 3GP:** Not all players support AMR-WB; use AAC for compatibility
- **H.264 in 3GP:** Baseline Profile required; High Profile will not play on some devices
- **Files with no moov:** Cannot be played; some streaming servers handle moof-only

### 14.3 Edge Cases to Handle in Converter
- **Empty file / 0-sample stream:** Return error or produce silence
- **File < 1 frame of audio:** Handle gracefully
- **All-silence audio:** AMR SID frames used for comfort noise
- **File with corrupt ftyp:** Attempt to identify by content scan
- **Truncated file:** Partial playback, no recovery
- **Sample rate not supported by codec:** AMR is fixed at 8 kHz (NB) or 16 kHz (WB)
- **Channel count not supported:** AMR mono only; AAC can be stereo

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM 3GP

| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → FLAC | `ffmpeg -i in.3gp -c:a flac -compression_level 8 out.flac` | Via tag mapping | Lossless decode |
| → ALAC | `ffmpeg -i in.3gp -c:a alac out.m4a` | Partial | Lossless decode |
| → MP3 | `ffmpeg -i in.3gp -c:a libmp3lame -q:a 0 out.mp3` | Tags via ID3v2.3 | Generation loss |
| → AAC | `ffmpeg -i in.3gp -c:a aac -b:a 256k out.m4a` | Partial | Re-encode |
| → WAV | `ffmpeg -i in.3gp -c:a pcm_s16le out.wav` | Limited | Lossless decode |
| → Opus | `ffmpeg -i in.3gp -c:a libopus -b:a 128k out.opus` | Vorbis Comments | Re-encode |

### 15.2 Converting TO 3GP

| Source | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| FLAC → | `ffmpeg -i in.flac -c:a amr_nb -f 3gp out.3gp` | Limited | Lossy re-encode |
| WAV → | `ffmpeg -i in.wav -c:a aac -b:a 64k -f 3gp out.3gp` | Limited | Re-encode |
| MP3 → | `ffmpeg -i in.mp3 -c:a amr_nb -f 3gp out.3gp` | None | Generation loss |
| AAC → | `ffmpeg -i in.m4a -c:a copy -f 3gp out.3gp` | Limited | Remux only |

### 15.3 Lossless Round-Trip Verification
```bash
# Decode 3GP to PCM
ffmpeg -i input.3gp -c:a pcm_s16le -ar 8000 decoded.wav

# Compare checksums (bit-exact decode)
md5sum decoded.wav   # For reference

# Verify bit-exact decoding
ffmpeg -i input.3gp -c:a pcm_s16le -f md5 -   # FFmpeg frame-level checksum
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| 3GPP TS 26.244 reference | C | 3GPP | Reference | Reference | 3GPP member portal |
| FFmpeg 3GP muxer/demuxer | C | LGPL 2.1+ | Good | Good | ffmpeg.org |
| libopencore-amr | C/C++ | Apache 2.0 | Good | Reference | sourceforge.net |
| GPAC (MP4Box) | C | LGPL 2.1+ | Good | Good | gpac.io |
| libavformat 3GP | C | LGPL 2.1+ | Good | Good | ffmpeg.org |

### Build Instructions (for bundling in converter app)
```bash
# Build FFmpeg with 3GP and AMR support
./configure --enable-muxer=3gp,3g2 \
  --enable-demuxer=3gp,3g2 \
  --enable-decoder=amr_nb,amr_wb,aac,h263,h264 \
  --enable-encoder=aac \
  --enable-libopencore-amrnb \
  --enable-libopencore-amrwb \
  --disable-doc
make -j$(nproc)
make install
```

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **3GPP TS 26.244:** 3GPP file format (3GP) — https://www.etsi.org/deliver/etsi_ts/126200_126299/126244/
- **ISO/IEC 14496-12:** ISO Base Media File Format — not publicly available
- **RFC 3839:** MIME Type Registrations for 3GPP files — https://www.rfc-editor.org/rfc/rfc3839
- **RFC 4867:** RTP Payload Format for AMR/AMR-WB — https://www.rfc-editor.org/rfc/rfc4867

### Technical Resources
- FFmpeg muxer options: `ffmpeg -h muxer=3gp` or https://ffmpeg.org/ffmpeg-formats.html
- FFmpeg 3GP demuxer: `ffmpeg -h demuxer=3gp`
- Multimedia Wiki: https://wiki.multimedia.cx/index.php?title=3GP
- 3GPP file format spec: https://www.etsi.org/deliver/etsi_ts/126200_126299/126244/

### Academic Papers
- "3GPP File Format for Multimedia Services" — 3GPP Technical Specification
- "Adaptive Multi-Rate (AMR) Speech Codec" — ETSI EN 301 704

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] Identify FFmpeg `./configure` flags: `--enable-muxer=3gp,3g2 --enable-libopencore-amrnb --enable-libopencore-amrwb`
- [ ] Verify `ffmpeg -muxers` output confirms 3GP muxer is available
- [ ] Verify `ffmpeg -demuxers` output confirms 3GP demuxer is available
- [ ] Note external library dependency: libopencore-amrnb, libopencore-amrwb for AMR

### Encoding Pipeline
- [ ] Set correct audio sample rate: AMR-NB = 8000 Hz, AMR-WB = 16000 Hz, AAC = 44100/48000 Hz
- [ ] Set correct channel count: AMR = mono (1), AAC = mono or stereo
- [ ] Select appropriate brand based on content and target device
- [ ] Map metadata fields to 3GP atom names
- [ ] Handle codec-specific parameters (AMR mode, AAC profile)

### Decoding Pipeline
- [ ] Parse ftyp box to identify 3GP variant and compatible brands
- [ ] Parse moov box to extract stream parameters
- [ ] Identify codec from stsd sample entry type ('samr', 'sawb', 'mp4a')
- [ ] Extract raw frames from mdat and decode using appropriate codec
- [ ] Handle audio-only vs video+audio files

### Metadata
- [ ] Read 3GP metadata from meta/ilst boxes
- [ ] Map 3GP atom keys to standard field names
- [ ] Handle cover art from 'covr' atom
- [ ] Write metadata back to 3GP format
- [ ] Preserve 3GPP-specific fields (description, classification, rating)

### Quality & Verification
- [ ] Verify 3GP brand compatibility with target device
- [ ] Test with: AMR-NB, AMR-WB, AAC, H.263, H.264 files
- [ ] Test seeking in files with and without index tables
- [ ] Implement error detection for corrupt box structures

### Edge Cases
- [ ] Handle files with no ftyp box (non-standard)
- [ ] Handle files with moov after mdat (need to seek/reorder)
- [ ] Handle files using 3GPP2 brands (EVRC, SMV audio)
- [ ] Handle AMR SID (comfort noise) frames during decode

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
