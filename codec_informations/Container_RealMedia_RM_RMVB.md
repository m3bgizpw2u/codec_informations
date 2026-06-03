# RealMedia (RM/RMVB) — Deep Technical Reference
> **Category:** Container
> **File Extensions:** `.rm`, `.ra`, `.rmv`, `.rmvb`, `.rmhd`, `.rma`, `.rmi`, `.ram`
> **MIME Types:** `application/vnd.rn-realmedia`, `audio/x-pn-realaudio`, `video/x-pn-realvideo`
> **Standardization Body:** RealNetworks (proprietary)
> **Primary Specification:** RealMedia File Format (RMFF) — draft IETF RFC (draft-heftagaub-rmff-00)
> **Patent Status:** Patented — proprietary RealNetworks technology
> **License:** Proprietary
> **Current Version:** RMFF 0 (RM) / RMFF 1 (RMVB/RMHD)
> **Active Development:** No — deprecated; RealPlayer discontinued 2012; maintained for backward compatibility

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** RealNetworks Inc. (Rob Glaser, et al.)
- **Year Created:** 1993 (RealAudio 1.0), 1997 (RealMedia video)
- **Original Purpose:** Enable streaming audio and video over dial-up internet connections with low bandwidth, using proprietary codecs optimized for network delivery and real-time playback
- **Problem with Predecessors:** Early internet audio required full download before playback. RealAudio pioneered streaming with real-time encoding/decoding, bitrate adaptation, and efficient multiplexing for the emerging internet audio market.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| RealAudio 1.0 | 1993 | VSELP codec, 14.4 kbps dialup |
| RealAudio 2.0 | 1995 | LD-CELP codec, 28.8 kbps |
| RealAudio 3.0 | 1996 | Sipro/ACELP, higher quality |
| RealMedia 0 | 1997 | Added video streams (RV10) |
| RealMedia RMVB | 2002 | Variable bitrate, higher quality |
| RealMedia HD (RMHD) | 2006 | H.264 support, modern codecs |
| Discontinued | 2012 | RealPlayer end-of-life |

### 1.3 Current Adoption
- **Primary use cases today:** Legacy streaming content, archival of old internet radio/TV, some legacy media archives
- **Platforms with native support:** RealPlayer (discontinued), some Linux players via FFmpeg
- **Major services using this format:** Historical: BBC Radio streaming, CNN video streaming (1990s–2000s)
- **Hardware support:** Very limited; mostly read via software
- **Status:** Deprecated / obsolete — replaced by MP4/HLS/RTMP in all modern applications

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 RealMedia Architecture
RealMedia uses a **chunk-based container** with big-endian byte ordering. Files contain a header section with property and stream metadata, followed by interleaved media data packets.

```
RealMedia File:
  [.RMF / .RMP Header]     ← File identification, version
  [PROP]                   ← File properties (play time, bitrate, etc.)
  [MDPR × N]               ← Media properties (one per stream)
  [CONT]                   ← Content description (title, author, copyright)
  [DATA Header]             ← Data section header
  [DATA Packets]           ← Interleaved audio/video packets
  [INDX × N]               ← Index entries (one per stream)
```

### 2.2 Supported Codecs

#### Audio Codecs
| Codec ID | Name | Description | Quality |
|----------|------|-------------|---------|
| `lpcJ` | RealAudio 1.0 | VSELP, 8 kHz mono | Low |
| `28_8` | RealAudio 2.0 | LD-CELP, 28.8 kbps | Low |
| `dnet` | AC3 | Dolby Digital | High |
| `sipr` | Sipro ACELP | Speech codec | Medium |
| `cook` | COOK | G2 codec, multirate | Medium-High |
| `atrc` | ATRAC3 | Sony ATRAC3 | High |
| `ralf` | RALF | RealAudio Lossless Format | Lossless |
| `raac` | LC-AAC | Low Complexity AAC | High |
| `racp` | HE-AAC | High Efficiency AAC | Very High |

#### Video Codecs
| Codec ID | Name | Description |
|----------|------|-------------|
| `RV10` | RealVideo 1 | H.263 derivative |
| `RV13` | RealVideo 1.3 | H.263+ derivative |
| `RV20` | RealVideo G2 | H.263+ improved |
| `RV30` | RealVideo 8 | H.264 precursor |
| `RV40` | RealVideo 9/10 | H.264 variant |
| `RVTR` | RealVideo TrueMotion | |
| `H263` | H.263 | Standard H.263 |
| `RMHD` | RealMedia HD | H.264/AVC |

### 2.3 Key Design Parameters
|| Parameter | Value | Notes |
|-----------|-------|-------|
| Max streams | Multiple | Audio + video + subtitle |
| Byte ordering | Big-endian | All multi-byte values |
| Timestamp resolution | ms | Packet timestamps in milliseconds |
| Packet interleaving | Yes | Audio/video interleaved in DATA |
| Seeking | Via INDX | Index-based seeking |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       2E 52 4D 46   .RMF     RealMedia File Header (standard)
0x0000  4       2E 52 4D 50   .RMP     RealMedia HD File Header
```

### 3.2 RealMedia File Header (.RMF) Layout
```
Offset  Size   Field Name              Type        Valid Range   Description
------  -----  ---------------------  ----------  ------------  ---------------------------
0x0000  4B     Chunk Type             char[4]    ".RMF"        RealMedia File header
0x0004  4B     Chunk Size             uint32 BE  18           Size of this chunk (always 18)
0x0008  2B     File Version           uint16 BE  0            Version (0 for RM, 1 for RMHD)
0x000A  2B     File Revision          uint16 BE  any          Revision number
0x000C  4B     Num Headers            uint32 BE  1–∞         Number of headers in file
```

### 3.3 File Properties Header (PROP) Layout
```
Offset  Size   Field Name              Type        Valid Range   Description
------  -----  ---------------------  ----------  ------------  ---------------------------
0x0000  4B     Chunk Type             char[4]    "PROP"        File properties
0x0004  4B     Chunk Size             uint32 BE  56           Size of this chunk
0x0008  2B     Object Version         uint16 BE  0            Version
0x000A  4B     Max Bitrate           uint32 BE  any          Maximum combined bitrate (bps)
0x000E  4B     Average Bitrate        uint32 BE  any          Average bitrate (bps)
0x0012  4B     Max Packet Size        uint32 BE  any          Largest data packet size (bytes)
0x0016  4B     Average Packet Size    uint32 BE  any          Average packet size (bytes)
0x001A  4B     Duration              uint32 BE  any          Playback duration (ms)
0x001E  4B     Preroll               uint32 BE  any          Preroll time (ms)
0x0022  4B     Index Offset          uint32 BE  any          Byte offset to first INDX chunk
0x0026  4B     Data Offset          uint32 BE  any          Byte offset to DATA chunk
0x002A  2B     Number of Streams     uint16 BE  1–16         Number of media streams
0x002C  4B     Flags                 uint32 BE  bitfield      0x01=saveable, 0x02=perfect play
```

### 3.4 Media Properties Header (MDPR) Layout
```
Offset  Size   Field Name              Type        Valid Range   Description
------  -----  ---------------------  ----------  ------------  ---------------------------
0x0000  4B     Chunk Type             char[4]    "MDPR"        Media properties
0x0004  4B     Chunk Size             uint32 BE  varies        Size of this chunk
0x0008  2B     Object Version         uint16 BE  0            Version
0x000A  4B     Max Bitrate           uint32 BE  any          Stream bitrate (bps)
0x000E  4B     Average Bitrate        uint32 BE  any          Stream average bitrate (bps)
0x0012  4B     Largest Packet Size    uint32 BE  any          Largest packet size (bytes)
0x0016  4B     Average Packet Size    uint32 BE  any          Average packet size (bytes)
0x001A  4B     Duration              uint32 BE  any          Stream duration (ms)
0x001E  4B     preroll               uint32 BE  any          Preroll time (ms)
0x0022  4B     Index Offset          uint32 BE  any          Stream-specific index offset
0x0026  2B     Stream Number          uint16 BE  1–255       Stream identifier
0x0028  4B     Stream Name Length     uint32 BE  0–N         Length of stream name
0x002C  N      Stream Name            string     any          Human-readable name
N+0x002C  4B   Mime Type Length     uint32 BE  0–N         Length of MIME type string
N+0x0030  M    MIME Type            string     any          MIME type (e.g., "audio/x-pn-realaudio")
M+N+0x0030  4B Type Specific Len     uint32 BE  0–N         Length of codec-specific data
M+N+0x0034  P  Type Specific Data    bytes      varies        Codec initialization data
```

### 3.5 Data Chunk Header Layout
```
Offset  Size   Field Name              Type        Valid Range   Description
------  -----  ---------------------  ----------  ------------  ---------------------------
0x0000  4B     Chunk Type             char[4]    "DATA"        Data chunk
0x0004  4B     Chunk Size             uint32 BE  varies        Total size of data chunk
0x0008  2B     Object Version         uint16 BE  0            Version
0x000A  4B     Num Packets           uint32 BE  any          Number of packets in chunk
0x000E  4B     Next Data Header       uint32 BE  0            Offset to next data chunk
```

### 3.6 Media Packet Header Layout (per packet)
```
Offset  Size   Field Name              Type        Valid Range   Description
------  -----  ---------------------  ----------  ------------  ---------------------------
0x0000  2B     Object Version         uint16 BE  0            Version
0x0002  2B     Packet Length          uint16 BE  1–65507     Packet data size
0x0004  2B     Stream Number          uint16 BE  1–255       Which stream this belongs to
0x0006  4B     Timestamp             uint32 BE  any          Presentation timestamp (ms)
0x000A  1B     Reserved              uint8      0xFF         Reserved byte
0x000B  1B     Flags                 uint8      bitfield      0x01=key frame, 0x02=end of stream
0x000C  N      Packet Data            bytes      varies        Encoded audio/video data
```

### 3.7 Index Chunk (INDX) Layout
```
Offset  Size   Field Name              Type        Valid Range   Description
------  -----  ---------------------  ----------  ------------  ---------------------------
0x0000  4B     Chunk Type             char[4]    "INDX"        Index chunk
0x0004  4B     Chunk Size             uint32 BE  varies        Size of this chunk
0x0008  2B     Object Version         uint16 BE  0            Version
0x000A  4B     Num Entries           uint32 BE  any          Number of index entries
0x000E  2B     Stream Number          uint16 BE  1–255       Which stream this index is for
0x0010  4B     Next INDX Offset       uint32 BE  any          Offset to next INDX chunk
--- Index Entries follow ---
0x0014  N      Index Entries          variable    N × 14 bytes See below
```

**Index Entry (14 bytes each):**
```
Offset  Size   Field Name              Type        Description
------  -----  ---------------------  ----------  ---------------------------
0       4B     Timestamp             uint32 BE   Packet timestamp (ms)
4       4B     Packet Offset         uint32 BE   Offset from DATA chunk start
8       2B     Packet Length         uint16 BE   Packet size
12      2B     Packet Flags           uint16 BE   0x0001=key frame
```

### 3.8 Sample Format Support
|| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | PCM (G.711) | Yes | For voice codecs |
| 16-bit | Signed integer | Yes | For PCM-based codecs |
| 24-bit | Signed integer | Partial | Some codecs |
| 32-bit | Signed integer | Partial | Some codecs |

### 3.9 Supported Sample Rates
|| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 8000 | Telephone | Yes | G.711 ulaw/alaw |
| 11025 | — | Yes | Older RealAudio |
| 16000 | Wideband | Yes | Cook, AAC |
| 22050 | — | Yes | |
| 32000 | Broadcast | Yes | |
| 44100 | CD audio | Yes | Most audio codecs |
| 48000 | Professional | Yes | AAC, ATRAC |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Encoding Pipeline (varies by codec)

#### RealAudio G2 (Cook) Encoding
```
Input PCM Samples
      │
      ▼
[Filterbank — MDCT]
128-point MDCT with overlap-add
      │
      ▼
[Bit Allocation]
Psychoacoustic model-based bit allocation
      │
      ▼
[Quantization]
Vector quantization of MDCT coefficients
      │
      ▼
[Entropy Coding]
Huffman coding of quantized indices
      │
      ▼
[Bit Packing]
Frame assembly with stream parameters
      │
      ▼
Output: RealMedia Packet
```

### 4.2 Encoder Settings / Quality Modes
|| Setting | Bitrate | Quality | Notes |
|---------|----------|---------|-------|
| Voice | 8–32 kbps | Low | Speech-optimized |
| Music Low | 64 kbps | Medium | HE-AAC |
| Music High | 128 kbps | High | AAC |
| Music Highest | 192+ kbps | Very High | AAC+ |

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. Read .RMF header at offset 0
2. Parse PROP chunk: get duration, packet sizes, stream count
3. Parse MDPR chunks (one per stream): build stream table
4. Read DATA chunk: iterate through interleaved packets
5. For each packet:
   a. Read packet header (stream_number, timestamp, flags)
   b. Look up codec from stream table
   c. Decode payload using appropriate codec
   d. Apply timestamp offset (subtract preroll)
6. Continue until end of DATA
```

#### Seeking
Seeking requires the INDX chunk:

```
1. Read PROP → Index Offset
2. Seek to INDX chunk
3. Parse index entries (timestamp, packet_offset, packet_length, flags)
4. Binary search: find entry where timestamp ≤ target_time
5. Seek to: DATA_start + entry.packet_offset
6. Decode forward to target frame
7. Apply preroll correction to timestamps
```

### 5.2 Core Decode Pipeline
```
1. Read .RMF header
2. Parse PROP: get file-level properties
3. Parse all MDPR chunks: build stream table
4. Parse CONT: read content metadata
5. Seek to DATA chunk start
6. For each packet:
   ├── Read packet header (version, length, stream_number, timestamp)
   ├── Look up codec from stream table by stream_number
   ├── Decode payload with appropriate codec
   ├── Apply preroll offset to timestamp
   └── Queue for playback (interleave audio/video by timestamp)
7. Parse INDX for seeking (optional)
```

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** RealMedia (RM/RMVB)
- **Overhead:** ~1–3% (headers, packet overhead)
- **Seeking in native container:** Yes — via INDX chunk
- **Multiple streams in native container:** Yes — audio + video + subtitle

### 6.2 Codec-to-Container Compatibility
|| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| RealMedia (native) | Yes (all Real codecs) | Via INDX | Limited (CONT) | Preferred |
| Matroska/MKA | No | N/A | N/A | Not supported |
| MP4/M4A | No | N/A | N/A | Not supported |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** RealMedia CONT (Content Description) chunk
- **Tag block location:** Within header section, before DATA chunk
- **Tag block identifier:** "CONT" chunk type code

### 7.2 Content Description (CONT) Chunk Layout
```
Offset  Size   Field Name              Type        Valid Range   Description
------  -----  ---------------------  ----------  ------------  ---------------------------
0x0000  4B     Chunk Type             char[4]    "CONT"        Content description
0x0004  4B     Chunk Size             uint32 BE  varies        Size of this chunk
0x0008  2B     Object Version         uint16 BE  0            Version
0x000A  2B     Title Length          uint16 BE  0–N         Length of title string
0x000C  N      Title                 string     any          Title text
N+0x000C  2B  Author Length         uint16 BE  0–N         Length of author string
N+0x000E  M    Author                string     any          Author text
M+N+0x000E  2B Copyright Length     uint16 BE  0–N         Length of copyright string
M+N+0x0010  P  Copyright            string     any          Copyright text
P+M+N+0x0012 2B Comment Length      uint16 BE  0–N         Length of comment string
P+M+N+0x0014  Q  Comment              string     any          Comment text
```

### 7.3 Standard Tag Fields
|| Tag Field | Internal Key (CONT) | Max Length | Notes |
|-----------|---------------------------|------------|-------|
| Title | Title (CONT) | any | CONT chunk |
| Author | Author (CONT) | any | CONT chunk |
| Copyright | Copyright (CONT) | any | CONT chunk |
| Comment | Comment (CONT) | any | CONT chunk |
| Rating | Rating | any | CONT chunk extension |

Note: RealMedia has very limited metadata compared to modern containers.

### 7.4 Metadata Compatibility Matrix
|| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| RealMedia CONT | ✓ | ✓ | ✓ | Highest (native) |
| ID3v1 | ✗ | ✗ | N/A | Not supported |
| APEv2 | ✗ | ✗ | N/A | Not supported |

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   real_108 (RV10), real_208 (RV20), real_210 (RV30/RV40), cook, atrac3, sipr, ralf, raac, racp
AV_CODEC_ID:        AV_CODEC_ID_RV10, AV_CODEC_ID_RV20, etc.
Format Name (CLI):   rm, rtsp                          # used with -f
Encoder(s):          none (FFmpeg cannot encode RealMedia)
Decoder(s):          many (see below)                  # ffmpeg -decoders | grep -i real
Muxer(s):           rm                                # ffmpeg -muxers | grep -i rm
Demuxer(s):          rm                               # ffmpeg -demuxers | grep -i rm
```

### 8.2 FFmpeg Decoding — Full CLI Reference

```bash
# Decode RealMedia to WAV
ffmpeg -i input.rm \
  -c:a pcm_s16le \
  output.wav

# Decode RealMedia to MKV (remux without re-encoding)
ffmpeg -i input.rm \
  -c copy \
  output.mkv

# Extract specific audio stream
ffmpeg -i input.rm \
  -map 0:a:0 \
  -c:a copy \
  output.ra

# Extract specific video stream
ffmpeg -i input.rmv \
  -map 0:v:0 \
  -c:v copy \
  output.rv

# Probe format
ffprobe -v quiet -print_format json -show_streams -show_format input.rm
```

### 8.3 FFmpeg Decoding — libavcodec C API

```c
// ─── Complete decode program skeleton ─────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.rm", NULL, NULL);
avformat_find_stream_info(fmt_ctx, NULL);

int audio_idx = av_find_best_stream(fmt_ctx, AVMEDIA_TYPE_AUDIO, -1, -1, NULL, 0);
AVStream *stream = fmt_ctx->streams[audio_idx];

const AVCodec *dec = avcodec_find_decoder(stream->codecpar->codec_id);
AVCodecContext *dec_ctx = avcodec_alloc_context3(dec);
avcodec_parameters_to_context(dec_ctx, stream->codecpar);
avcodec_open2(dec_ctx, dec, NULL);

AVPacket *pkt = av_packet_alloc();
AVFrame  *frm = av_frame_alloc();

while (av_read_frame(fmt_ctx, pkt) >= 0) {
    if (pkt->stream_index == audio_idx) {
        avcodec_send_packet(dec_ctx, pkt);
        while (avcodec_receive_frame(dec_ctx, frm) == 0) {
            process_audio_frame(frm);
            av_frame_unref(frm);
        }
    }
    av_packet_unref(pkt);
}

avcodec_send_packet(dec_ctx, NULL);
while (avcodec_receive_frame(dec_ctx, frm) == 0) {
    process_audio_frame(frm);
    av_frame_unref(frm);
}

av_frame_free(&frm);
av_packet_free(&pkt);
avcodec_free_context(&dec_ctx);
avformat_close_input(&fmt_ctx);
```

### 8.4 FFmpeg Streaming

```bash
# RTSP streaming from RealMedia server
ffmpeg -rtsp_transport tcp \
  -i rtsp://server/stream.rm \
  -c:a pcm_s16le \
  output.wav

# MMS (Microsoft Media Server) streaming
ffmpeg -i mms://server/stream.wmv \
  -c:a pcm_s16le \
  output.wav
```

### 8.5 FFmpeg Metadata Handling

```bash
# Read all metadata as JSON
ffprobe -v quiet -print_format json -show_format input.rm | jq .format.tags

# FFmpeg metadata support for RealMedia is limited
# CONT chunk metadata may appear in ffprobe output
```

### 8.6 Quality / Fidelity Decision Table
|| Use Case | Settings | Notes |
|----------|----------|-------|
| Archival decode | Decode to WAV | Lossless decode to PCM |
| Remux to MKV | `-c copy` | Preserve quality, fix container |
| Convert to modern format | Transcode to AAC/FLAC | Generation loss |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
```
RealMedia Index (INDX chunk):
  Location:     End of file (after DATA)
  Magic:       "INDX" chunk type
  Entry size:  14 bytes per entry
  Entry format:
    [0x00–0x03]  Timestamp (uint32 BE) — ms from start
    [0x04–0x07]  Packet offset (uint32 BE) — from DATA start
    [0x08–0x09]  Packet length (uint16 BE)
    [0x0A–0x0B]  Flags (uint16 BE) — 0x0001=keyframe
```

### 9.2 Gapless Playback Data
```
RealMedia does not store explicit gapless metadata.
Encoder delay:  Varies by codec (typically ~20–100 ms)
Padding:        Varies by codec
Storage:        Not standardized
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

|| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | Designed for streaming |
| Algorithmic encoder delay | ~20–100 ms | Codec-dependent |
| Live encoding feasible | Yes | Real-time encoding available |
| HTTP progressive download | Yes | Via .rm files |
| HTTP Live Streaming (HLS) | No | Not supported |
| RTSP streaming | Yes | Real-time Streaming Protocol |
| MMS streaming | Yes | Microsoft Media Server |
| WebRTC / RTP transport | Yes | Via RTSP/RTP |

### 10.1 Streaming Protocols
| Protocol | Description | FFmpeg Support |
|----------|-------------|----------------|
| RTSP (Real Time Streaming) | Standard streaming control | Yes (`-rtsp_transport`) |
| RTSP over HTTP | HTTP tunneling | Partial |
| MMS (MMSH/MMST) | Microsoft Media Server | Yes (`mms://`) |
| HTTP | Progressive download | Yes |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
|| Channels | Layout Name | Channel Order | Notes |
|----------|-------------|---------------|-------|
| 1 | Mono | C | Supported |
| 2 | Stereo | L, R | Supported |

Note: RealMedia audio codecs are limited to mono or stereo.

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

|| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 16-bit | Most codecs |
| Max sample rate | 48000 Hz | Limited |
| Float support | No | Not supported |
| DSD support | No | Not applicable |

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

|| Platform / API | Encode | Decode | Notes |
|----------------|--------|--------|-------|
| RealPlayer | Yes | Yes | Discontinued |
| FFmpeg native | No | Yes | Via libavformat/rmdec.c |
| VLC | No | Yes | Via FFmpeg |
| GStreamer | No | Yes | Via FFmpeg |

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Encoder-Specific Issues
|| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| No RealMedia encoder in FFmpeg | All | Not possible to create new RM files |
| Limited codec support | Older builds | Update FFmpeg |
| RTSP negotiation issues | Some servers | Use `-rtsp_transport tcp` |

### 14.2 Interoperability Issues
- **Legacy codec support:** Some old RealAudio codecs not fully reverse-engineered
- **DRM-protected files:** Cannot decode protected content
- **Corrupt headers:** Some .rm files have malformed headers; skip gracefully

### 14.3 Edge Cases to Handle in Converter
- **Empty file / 0-packet stream:** Valid RM with no media; skip gracefully
- **Corrupt packet:** Skip packet, continue to next
- **Unknown codec:** Log warning, skip stream
- **Missing INDX:** Seeking not possible; linear scan required

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM RealMedia

|| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → FLAC | `ffmpeg -i in.rm -c:a flac -compression_level 8 out.flac` | Limited | Lossless decode + re-encode |
| → WAV | `ffmpeg -i in.rm -c:a pcm_s16le out.wav` | None | Lossless decode |
| → MP3 | `ffmpeg -i in.rm -c:a libmp3lame -q:a 0 out.mp3` | Limited | Generation loss |
| → MKV | `ffmpeg -i in.rm -c copy out.mkv` | Partial | Remux only |

### 15.2 Converting TO RealMedia

Not supported. FFmpeg cannot encode RealMedia. Use legacy RealProducer software.

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

|| Library | Language | License | Encoder | Decoder | URL |
|---------|----------|---------|---------|---------|-----|
| RealNetworks SDK | C/C++ | Proprietary | Yes | Yes | Discontinued |
| FFmpeg native | C | LGPL 2.1+ | No | Yes | https://ffmpeg.org |
| VLC | C | GPL | No | Yes | https://videolan.org |

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **RealMedia File Format (draft):** https://datatracker.ietf.org/doc/html/draft-heftagaub-rmff-00
- **Multimedia Wiki RMFF:** https://multimedia.cx/rmff.htm

### Technical Resources
- FFmpeg demuxer options: `ffmpeg -h demuxer=rm` or https://ffmpeg.org/ffmpeg-formats.html
- Multimedia Wiki: https://wiki.multimedia.cx/index.php/RealMedia
- Hydrogenaudio: https://hydrogenaud.io/index.php/board,84.0.html

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] FFmpeg rm demuxer is built into default FFmpeg — no external dependency
- [ ] Verify `ffmpeg -decoders` output confirms RealMedia decoders are available
- [ ] Note: FFmpeg does NOT have a RealMedia encoder

### Decoding Pipeline
- [ ] Implement .RMF magic verification
- [ ] Parse PROP chunk for file-level properties
- [ ] Parse all MDPR chunks to build stream table
- [ ] Parse CONT chunk for content metadata
- [ ] Handle AVERROR(EAGAIN) in receive_packet loop correctly
- [ ] Flush decoder at EOF (send NULL packet)
- [ ] Handle preroll offset correction for timestamps

### Metadata
- [ ] Read CONT chunk for basic metadata (title, author, copyright, comment)
- [ ] Note: RealMedia metadata support is very limited
- [ ] Cannot write RealMedia metadata from FFmpeg

### Edge Cases
- [ ] Handle corrupt or malformed headers gracefully
- [ ] Handle missing INDX chunk (no seeking)
- [ ] Handle unknown codec IDs (skip stream)
- [ ] Handle DRM-protected files (cannot decode)

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
