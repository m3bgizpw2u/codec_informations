# Musepack (MPC) SV7/SV8 — Deep Technical Reference
> **Category:** Container
> **File Extensions:** `.mpc`, `.mp+`, `.mpp`
> **MIME Types:** `audio/x-musepack`, `audio/musepack`
> **Standardization Body:** Musepack.net (open specification)
> **Primary Specification:** SV8 Specification — http://mutagen-specs.readthedocs.io/en/latest/mpc/sv8.html
> **Patent Status:** Patent-free — no known patent claims
> **License:** BSD / Open source (Musepack SV8 is free software)
> **Current Version:** SV8 (r475, August 2011)
> **Active Development:** Limited — stable and mature; active maintenance only

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Andree Buschmann, Frank Klemm, and the Musepack development team
- **Year Created:** 1997 (originally as MPEGplus/MPP), evolved through SV4–SV7, SV8 released 2009
- **Original Purpose:** Provide high-quality lossy audio compression optimized for music, inspired by MPEG-2 AAC but with improvements for the critical mid-bitrate range. Initially developed as a free alternative to proprietary formats.
- **Problem with Predecessors:** Early SV4–SV6 formats had quality issues at very low bitrates. SV7 improved significantly. SV8 was a complete rewrite to improve streaming, seeking, and compression efficiency.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| SV4 | 1998 | Early MPEGplus implementation |
| SV5 | 1999 | Improved psychoacoustic model |
| SV6 | 2001 | Further quality improvements |
| SV7 | 2002 | Stable release; Huffman coding, SV8 announced |
| SV8 | 2009 | Container-independent, packetized, compressed bitstream, chapters, sample-accurate seeking |

### 1.3 Current Adoption
- **Primary use cases today:** Music archiving and playback, Rockbox firmware, foobar2000, professional audio workflows
- **Platforms with native support:** Rockbox (firmware),foobar2000, VLC (via FFmpeg), JRiver Media Center, DBPowerAmp
- **Major services using this format:** Historically used by some archival projects; niche community
- **Hardware support:** Rockbox-based DAPs (iRiver, Sansa, etc.), some Car MP3 players
- **Status:** Niche but well-regarded for quality; SV8 adoption growing

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Musepack Architecture
Musepack is a **subband-based** perceptual audio codec, closely related to MPEG-2 AAC's structure. Audio is processed in 32 subbands with 36 samples each, totaling **1152 samples per frame**.

```
Musepack Frame Structure:
  ┌──────────────────────────────────────────┐
  │ Frame Header                             │
  │  ├── Sync word (0x000)                   │
  │  ├── Bitrate/index information            │
  │  └── Frame-specific parameters           │
  ├──────────────────────────────────────────┤
  │ Scale Factor Bands (32 bands × 3 frames) │
  │  └── Per-band scale factors (DSCF)      │
  ├──────────────────────────────────────────┤
  │ Subband Data (32 bands × 36 samples)      │
  │  ├── Band type per subband               │
  │  ├── Quantized samples per band          │
  │  └── Huffman/VLC-coded                  │
  ├──────────────────────────────────────────┤
  │ CRC (SV7 only)                           │
  └──────────────────────────────────────────┘
```

### 2.2 SV7 vs SV8 Differences
| Aspect | SV7 | SV8 |
|--------|-----|-----|
| Container | Binary with MP+ magic | Packetized (MPCK magic) |
| Bitstream coding | Uncompressed | Compressed (Huffman tables) |
| File size | Baseline | ~2% smaller |
| Seeking | Slow (frame scan) | Sample-accurate (seek table) |
| Streaming | Limited | Full support |
| Chapters | No | Yes |
| Error resilience | CRC per frame | Better error concealment |

### 2.3 Key Design Parameters
|| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | 0 samples | No lookahead required |
| Frame size | 1152 samples | 32 subbands × 36 samples |
| Max channels | 2 | Stereo only (SV7); SV8 supports up to 2 |
| Bit depth | 16-bit (internal 32-bit) | |
| Bitrate range | 0–1300 kbps | Pure variable bitrate |
| Compression | Lossy (no lossless mode) | |
| Complexity | Encode: high; Decode: low | |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 SV7 File Signature / Magic Bytes
```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       4D 50 2B 20      MP+      Musepack SV7 magic ("MP+ " in ASCII)
```

### 3.2 SV7 Header Layout (32 bytes)
```
Offset  Size   Field Name              Type        Valid Range   Description
------  -----  ---------------------  ----------  ------------  ---------------------------
0x0000  4B     Magic                   char[4]     "MP+ "        SV7 signature
0x0004  2B     Stream version          uint16 LE   0x0007        Must be 7 (SV7)
0x0006  2B     Sample frequency       uint16 LE   44100/48000   Samples per second
0x0008  4B     Max band (0–31)        uint32 LE   0–31          Highest coded subband
0x000C  4B     Intensity stereo pos   uint32 LE   varies        Joint stereo position
0x0010  4B     Side info checksum     uint32 LE   any           CRC of side information
0x0014  4B     CRC of header         uint32 LE   any           Header CRC
0x0018  4B     Number of frames       uint32 LE   any           Total frames in file
0x001C  4B     Last frame length      uint32 LE   any           Bits in last frame
0x0020  8B     Title ref              uint64 LE   any           Internal title reference
0x0028  4B     Artist ref             uint32 LE   any           Internal artist reference
0x002C  4B     Comment ref            uint32 LE   any           Internal comment reference
0x0030  4B     Album ref              uint32 LE   any           Internal album reference
0x0034  4B     Year ref              uint32 LE   any           Internal year reference
0x0038  4B     Track ref              uint32 LE   any           Internal track reference
0x003C  4B     Genre ref              uint32 LE   any           Internal genre reference
0x0040  8B     Duration              uint64 LE   any           Duration in ms (packed)
0x0048  8B     File size             uint64 LE   any           Total file size
```

### 3.3 SV7 Frame Format (per-frame structure)
```
Offset  Bit    Field Name              Bit Width   Type       Description
------  -----  ---------------------  ----------  ---------  ---------------------------
0       12     Frame length             12          uint       Number of bits in this frame
12      12     Frame length (copy)     12          uint       Must match first 12 bits
24      1      Key frame flag          1           uint       1=key frame, 0=not
25      1      Stereo band flag        1           uint       1=has stereo bands
26      6      Max band index          6           uint       max_band + 1 (1–32)
--- Per subband data (max_band + 1 bands) ---
?       1      MS stereo flag           1           uint       1=mid-side stereo for this band
?       2      Band type (lower)        2           uint       Quantizer type
?       2      Band type (upper)        2           uint       Quantizer type
--- Per non-zero band ---
?       3      Scale factor coding      3           VLC         DSCF or raw value
?       ?      Quantized samples        variable    VLC/huffman Encoded samples
--- CRC ---
?       16     Frame CRC               16          uint16 LE  CRC-16 of frame data
```

### 3.4 SV8 Packet Structure (Key/Size/Payload)
SV8 uses a **packetized container format** with the "MPCK" magic:

```
Offset  Size   Field Name              Type        Description
------  -----  ---------------------  ----------  ---------------------------
0       4B     Magic                   char[4]    "MPCK" (0x4D50434B)
--- Packets follow in sequence ---
N       2B     Packet Key              uint16 BE  Packet type identifier (see below)
N+2     N      Variable-size Size       uintN      Packet size (see variable-length encoding)
N+2+N   M      Packet Payload          bytes      Packet data
```

**SV8 Packet Key/Size/Payload:**
```
Size field encoding (variable-length):
  The size is encoded as a series of bytes, each with 7 bits of size + 1 continuation bit
  - If bit 7 = 0: this is the last size byte; bits 0–6 = size value
  - If bit 7 = 1: more size bytes follow; bits 0–6 = upper bits of size
  Maximum size: 10 bytes → up to 2^63 bytes
```

**SV8 Packet Types:**
| Key | Name | Mandatory | Description |
|-----|------|-----------|-------------|
| `SH` | Stream Header | Yes | Stream parameters |
| `RG` | ReplayGain | Yes | Loudness metadata |
| `EI` | Encoder Info | No | Encoder identification |
| `SO` | Seek Table Offset | No | Location of seek table |
| `AP` | Audio Packet | Yes | Compressed audio data |
| `ST` | Seek Table | No | Byte offsets for seeking |
| `CT` | Chapter Tag | No | Chapter markers |
| `SE` | Stream End | Yes | End of stream marker |

### 3.5 SV8 Stream Header Packet (SH)
```
Offset  Size   Field Name              Type        Valid Range   Description
------  -----  ---------------------  ----------  ------------  ---------------------------
0       4B     CRC-32                  uint32 BE  any           CRC of entire packet (this field=0)
4       1B     Stream version          uint8      8             Must be 8
5       8B     Sample count            var        any            Total audio samples
13      4B     Sample frequency        uint32 BE  44100/48000   Samples per second
17      2B     Channel count           uint16 BE  1–2           1=mono, 2=stereo
19      1B     Precision               uint8      16/32         Internal precision (bits)
20      4B     Format flags            uint32 BE  varies        Format-specific flags
```

### 3.6 Sample Format Support
|| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Signed integer | No | Not supported |
| 16-bit | Signed integer | Yes | Primary input/output format |
| 20-bit | Signed integer | Yes | Internal precision |
| 24-bit | Signed integer | Yes | Internal precision |
| 32-bit | Signed integer | Yes | Internal precision (SV8) |
| 32-bit | IEEE float | No | Not supported |

### 3.7 Supported Sample Rates
|| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 44100 | CD audio | Yes | Most common |
| 48000 | Professional | Yes | Supported in SV8 |

Note: Musepack only officially supports 44100 Hz and 48000 Hz. Other rates require resampling.

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Musepack Encoding Pipeline
```
Input PCM Samples (16-bit, 32-bit float, 1152 samples per frame)
      │
      ▼
[Subband Analysis — 32-band PQMF]
Polyphase Quadrature Mirror Filterbank
Divides audio into 32 subbands, 36 samples each
      │
      ▼
[psychoacoustic Model]
FFT-based masking threshold calculation
Determines quantizer noise floor per subband
      │
      ▼
[Stereo Processing]
Mid-side (MS) stereo or independent stereo
Determined per-band by encoder
      │
      ▼
[Scale Factor Coding]
DSCF (Differential Scale Factor) coding for scale factors
Entropy-coded differences between adjacent scale factors
      │
      ▼
[Sample Quantization]
Per-band non-uniform quantization
Huffman-coded quantized samples (SV7: raw bits; SV8: Huffman tables)
      │
      ▼
[Frame Assembly]
Sync word + frame data + CRC
SV8: Packetize with Key/Size/Payload structure
      │
      ▼
Output: Musepack Frame
```

### 4.2 Subband Processing Details
- **32 subbands:** Each covering 1/32 of the Nyquist frequency
- **36 samples per subband:** Total 1152 samples per frame
- **Scale factor bands:** Groups of subbands with shared scale factors
- **M/S stereo:** Applied per subband, typically in upper frequency bands
- **Quantizer types:** Variable coding depending on signal complexity

### 4.3 Encoder Settings / Quality Modes
Musepack uses a **quality parameter** rather than bitrate:

|| Setting | Bitrate Range | Intended Use | Notes |
|---------|--------------|-------------|-------|
| --quality 0 | ~70 kbps | Lowest quality | Heavily filtered |
| --quality 1 | ~100 kbps | Low quality | |
| --quality 2 | ~130 kbps | Medium quality | Default |
| --quality 3 | ~160 kbps | Good quality | |
| --quality 4 | ~190 kbps | High quality | |
| --quality 5 | ~220 kbps | Very high | |
| --quality 6 | ~260 kbps | Transparent | Transparency threshold |
| --quality 7 | ~300 kbps | Very high | |
| --quality 8 | ~350 kbps | Maximum | |

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### SV7 Sync Strategy
```
1. Read "MP+ " magic at offset 0
2. Parse header (32 bytes): verify stream version = 7
3. Read frame count from header
4. For each frame:
   a. Read 12-bit frame length (bits 0–11)
   b. Read another 12-bit copy (bits 12–23) for verification
   c. Validate both lengths match
   d. Read frame data for frame_length bits
   e. Decode subband data, scale factors, quantized samples
   f. Verify frame CRC-16
   g. Apply inverse PQMF synthesis
5. Interleave stereo channels
```

#### SV8 Sync Strategy
```
1. Read "MPCK" magic at offset 0
2. Parse packets in sequence:
   a. Read 2-byte packet key
   b. Read variable-length size
   c. Read payload data
   d. Process packet based on key:
      - SH: Parse stream parameters, store for audio decode
      - RG: Parse ReplayGain metadata
      - AP: Decode audio data for this packet
      - ST: Build seek table from seek points
      - SE: End of stream reached
```

#### Seeking
- **SV7:** Linear scan through frames; slow for large files
- **SV8:** Seek table provides byte offsets; O(1) seek via binary search + decode

### 5.2 Core Decode Pipeline
```
SV8 Decode:
1. Read "MPCK" magic
2. Parse SH packet:
   ├── Verify version = 8
   ├── Extract sample_rate, channels, precision
   └── Store sample_count for duration
3. Parse optional packets (RG, EI, SO, ST)
4. For each AP packet:
   ├── Decode compressed audio data
   ├── Apply Huffman decoding (SV8) or direct VLC (SV7)
   ├── Dequantize samples per subband
   ├── Apply scale factors
   └── Apply stereo decoding (MS → L/R)
5. Apply inverse PQMF synthesis filterbank
6. Output 1152 PCM samples per frame
7. Parse SE packet → end of stream
```

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** Musepack MPC (SV7: binary, SV8: packetized)
- **Overhead:** ~0.1–0.5% (header, packet overhead)
- **Seeking in native container:** Yes (SV8 seek table, SV7 slow scan)
- **Multiple streams in native container:** No — single audio stream

### 6.2 Codec-to-Container Compatibility
|| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| MPC (native SV7) | Yes | No (slow) | APEv2 (at end) | Limited |
| MPC (native SV8) | Yes | Yes (seek table) | APEv2, chapters | Preferred |
| Matroska/MKA | Yes | Yes | Full | Via EBML |
| OGG | No | N/A | N/A | Not supported |
| MP4/M4A | No | N/A | N/A | Not supported |
| WAV | No | N/A | N/A | Not applicable |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** APEv2 tags (at end of file for SV7; as CT packet for SV8)
- **Tag block location:** End of file (SV7) or within stream (SV8 CT packet)
- **Tag block identifier:** `APEv2` tag format

### 7.2 Standard Tag Fields — Complete Reference
|| Tag Field | Internal Key (MPC/APEv2) | Max Length | Character Encoding | Multi-value | Notes |
|-----------|---------------------------|------------|-------------------|-------------|---|
| Title | TITLE | unlimited | UTF-8 | Yes | |
| Artist | ARTIST | unlimited | UTF-8 | Yes | |
| Album | ALBUM | unlimited | UTF-8 | Yes | |
| Album Artist | ALBUMARTIST | unlimited | UTF-8 | Yes | |
| Composer | COMPOSER | unlimited | UTF-8 | Yes | |
| Genre | GENRE | unlimited | UTF-8 | Yes | |
| Year | YEAR | 4 bytes | ASCII | No | Four-digit year |
| Track Number | TRACK | unlimited | ASCII | No | Plain number |
| Track Total | TRACKTOTAL | unlimited | ASCII | No | |
| Disc Number | DISCNUMBER | unlimited | ASCII | No | |
| Disc Total | DISCTOTAL | unlimited | ASCII | No | |
| Comment | COMMENT | unlimited | UTF-8 | Yes | |
| ReplayGain Track Gain | REPLAYGAIN_TRACK_GAIN | unlimited | ASCII | No | Format: "-6.20 dB" |
| ReplayGain Track Peak | REPLAYGAIN_TRACK_PEAK | unlimited | ASCII | No | Format: "0.998459" |
| Encoder | ENCODER | unlimited | UTF-8 | No | Software name |

### 7.3 Cover Art Storage
```
Cover art storage format in Musepack:
  Container type:  APEv2 tag item with binary value
  Image formats:   JPEG, PNG
  Max image size:  No hard limit (practical: 16 MB)
  
  APEv2 item for cover art:
    Key:  "Cover Art (Front)" or "Cover Art (Back)" or "Cover art"
    Value: [binary image data]
```

### 7.4 Metadata Compatibility Matrix
|| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| APEv2 | ✓ | ✓ | ✓ | Highest (native) |
| SV8 Stream metadata (RG) | ✓ | Partial | Partial | Medium |
| SV8 Chapters (CT) | ✓ | ✓ | ✓ | High (SV8 only) |

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   mpc7 (SV7), mpc8 (SV8)   # used with -c:a
AV_CODEC_ID:        AV_CODEC_ID_MUSEPACK7, AV_CODEC_ID_MUSEPACK8
Format Name (CLI):   mpc                         # used with -f
Encoder(s):          none                         # FFmpeg does not encode Musepack
Decoder(s):          mpc7, mpc8                  # ffmpeg -decoders | grep mpc
Muxer(s):           mpc                          # ffmpeg -muxers | grep mpc
Demuxer(s):          mpc                         # ffmpeg -demuxers | grep mpc
```

Note: FFmpeg can **decode** Musepack but cannot **encode** it. Use standalone mpcenc for encoding.

### 8.2 FFmpeg Decoding — Full CLI Reference

```bash
# Decode Musepack to WAV (auto-detects SV7/SV8)
ffmpeg -i input.mpc \
  -c:a pcm_s16le \
  output.wav

# Explicitly specify decoder version
ffmpeg -i input.mpc \
  -c:a mpc7 \              # SV7 decoder
  -c:a pcm_s16le \
  output_sv7.wav

ffmpeg -i input.mpc \
  -c:a mpc8 \              # SV8 decoder
  -c:a pcm_s16le \
  output_sv8.wav

# Extract with metadata copy
ffmpeg -i input.mpc -c:a copy output.mpc

# Probe format
ffprobe -v quiet -print_format json -show_streams -show_format input.mpc
```

### 8.3 FFmpeg Decoding — libavcodec C API

```c
// ─── Complete decode program skeleton ─────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.mpc", NULL, NULL);
avformat_find_stream_info(fmt_ctx, NULL);

int audio_idx = av_find_best_stream(fmt_ctx, AVMEDIA_TYPE_AUDIO, -1, -1, NULL, 0);
AVStream *stream = fmt_ctx->streams[audio_idx];

// FFmpeg auto-selects mpc7 or mpc8 based on stream
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
            // Musepack outputs: 1152 samples per frame
            // Format: AV_SAMPLE_FMT_S16 (16-bit signed)
            // frm->nb_samples = 1152
            // frm->sample_rate = 44100 or 48000
            // frm->ch_layout = mono or stereo
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

### 8.4 FFmpeg Metadata Handling

```bash
# Read all metadata as JSON
ffprobe -v quiet -print_format json -show_format input.mpc | jq .format.tags

# Write metadata (APEv2 tags — works for output files)
ffmpeg -i input.mpc \
  -c:a copy \
  -metadata title="Song Title" \
  -metadata artist="Artist Name" \
  output.mpc

# Strip all metadata
ffmpeg -i input.mpc -c:a copy -map_metadata -1 output.mpc
```

#### FFmpeg Internal Metadata Key Mapping
|| Standard Field | FFmpeg Key | MPC Native Key | Notes |
|----------------|------------|--------------------------|-------|
| Title | title | TITLE | |
| Artist | artist | ARTIST | |
| Album | album | ALBUM | |
| Genre | genre | GENRE | |
| Date | date | YEAR | |
| Track Number | track | TRACK | |
| Comment | comment | COMMENT | |

### 8.5 Quality / Fidelity Decision Table
|| Use Case | Settings | Expected Size | Notes |
|----------|----------|---------------|-------|
| High-quality archival | SV8 encoder, quality 8 | ~350 kbps | Maximum quality |
| Transparent encoding | SV8 encoder, quality 6 | ~260 kbps | Transparency threshold |
| Standard streaming | SV8 encoder, quality 4 | ~190 kbps | Good balance |
| Low bandwidth | SV8 encoder, quality 2 | ~130 kbps | Default |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
```
SV8 Seek Table (ST packet):
  Location:     Within MPC stream (referenced by SO packet, or standalone)
  Magic:       Key = "ST"
  Entry size:  8 bytes per seek point
  Entry format:
    [0x00–0x03]  Frame number (uint32 BE) — key frame index
    [0x04–0x07]  Byte offset (uint32 BE) — from packet start to frame
```

### 9.2 Gapless Playback Data
```
Musepack does not store explicit gapless metadata.
Encoder delay:  0 samples — PQMF synthesis has symmetric delay
Padding:        0 samples — frames are independently decodable
SV8: Sample count field in SH packet gives exact total.
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

|| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes (SV8) | Packetized format |
| Algorithmic encoder delay | 0 samples | PQMF is symmetric |
| Live encoding feasible | Yes | SV8 supports streaming |
| HTTP progressive download | Yes | Common |
| HTTP Live Streaming (HLS) | No | Not natively supported |
| DASH streaming | Yes | Via Matroska/MKA |
| WebRTC / RTP transport | No | Not natively supported |
| Minimum decode buffer | 1 frame | 1152 samples |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
|| Channels | Layout Name | Channel Order | Notes |
|----------|-------------|---------------|-------|
| 1 | Mono | C | |
| 2 | Stereo | L, R | |
| 2 | Joint Stereo | M/S | Mid-side stereo |

Note: Musepack only supports up to 2 channels. No multi-channel (5.1) support.

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

|| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 16-bit (input), 32-bit (internal) | |
| Max sample rate | 48000 Hz | Only 44100 and 48000 Hz supported |
| Float support | No | Not supported |
| DSD support | No | Not applicable |

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

|| Platform / API | Encode | Decode | Notes |
|----------------|--------|--------|-------|
| Rockbox firmware | Yes | Yes | Native on supported DAPs |
| FFmpeg native | No (encode) | Yes | mpc7/mpc8 decoders |
| foobar2000 | Yes | Yes | Native |
| VLC | No (encode) | Yes | Via FFmpeg |
| JRiver Media Center | Yes | Yes | Native |

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Encoder-Specific Issues
|| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| No Musepack encoder in FFmpeg | All | Use standalone mpcenc |
| SV8 seek table not auto-built | All | Use mpcenc with seek table |
| APEv2 tag writing limited | All | Use dedicated tag editor |

### 14.2 Interoperability Issues
- **SV7 vs SV8:** Some players only support SV7; SV8 is backward-compatible
- **Sample rate:** Only 44100 and 48000 Hz are officially supported
- **MPEGplus legacy files:** SV4–SV6 files may not decode correctly

### 14.3 Edge Cases to Handle in Converter
- **Empty file / 0-sample stream:** Valid MPC with no frames; skip gracefully
- **Corrupt frame:** CRC-16 detects; skip frame, mute
- **SV7 seek:** Warn user about slow seeking
- **Missing APEv2 tags:** Not an error; proceed without tags
- **Full-scale sine (0 dB):** May clip in quantized subbands

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM Musepack

|| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → FLAC | `ffmpeg -i in.mpc -c:a flac -compression_level 8 out.flac` | APEv2 → Vorbis | Lossless (re-encode) |
| → WAV | `ffmpeg -i in.mpc -c:a pcm_s16le out.wav` | Limited | Lossless (re-encode) |
| → MP3 | `ffmpeg -i in.mpc -c:a libmp3lame -q:a 0 out.mp3` | APEv2 → ID3v2.3 | Generation loss |
| → AAC | `ffmpeg -i in.mpc -c:a aac -b:a 256k out.m4a` | Via -metadata | Generation loss |
| → Opus | `ffmpeg -i in.mpc -c:a libopus -b:a 128k out.opus` | Limited | Generation loss |

### 15.2 Converting TO Musepack

Note: FFmpeg cannot encode Musepack. Use standalone `mpcenc` from the Musepack package.

```bash
# Musepack SV8 encoding (standalone mpcenc)
mpcenc --quality 6 input.wav output.mpc

# With seek table
mpcenc --quality 6 --seekfile input.wav output.mpc

# With ReplayGain
mpcenc --quality 6 --replaygain input.wav output.mpc
```

### 15.3 Lossless Round-Trip Verification
Musepack is a **lossy codec** — no lossless round-trip is possible. Verify transparency:

```bash
# Decode to WAV
ffmpeg -i input.mpc -c:a pcm_s16le decoded.wav

# Compare with original (bit-exact is impossible due to lossy nature)
# Use perceptual comparison tools instead:
# - bs1770gain for loudness
# - audiotools for spectrogram comparison
# - Subjective listening tests
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

|| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| libmpcdec | C | BSD | — | Reference | https://www.musepack.net/ |
| libmpcenc | C | BSD | Reference | — | https://www.musepack.net/ |
| FFmpeg native | C | LGPL 2.1+ | — | 9/10 | https://ffmpeg.org |
| Rockbox | C | GPL | Native | Native | https://www.rockbox.org |

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **SV8 Specification:** http://mutagen-specs.readthedocs.io/en/latest/mpc/sv8.html
- **Musepack Official:** https://www.musepack.net/

### Technical Resources
- FFmpeg decoder options: `ffmpeg -h decoder=mpc7` or `ffmpeg -h decoder=mpc8`
- Multimedia Wiki: https://wiki.multimedia.cx/index.php/Musepack
- Hydrogenaudio: https://hydrogenaud.io/index.php/board,33.0.html

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] FFmpeg mpc7 and mpc8 decoders are built into default FFmpeg — no external dependency
- [ ] Verify `ffmpeg -decoders` output confirms mpc7 and mpc8 decoders are available
- [ ] Note: FFmpeg does NOT have a Musepack encoder — use standalone mpcenc

### Decoding Pipeline
- [ ] Implement Musepack sync search ("MP+ " for SV7, "MPCK" for SV8)
- [ ] Parse SV7 header (32 bytes) for stream parameters
- [ ] Parse SV8 packets (SH, AP, ST, SE, etc.)
- [ ] Handle AVERROR(EAGAIN) in receive_packet loop correctly
- [ ] Flush decoder at EOF (send NULL packet)
- [ ] Handle sample format conversion (Musepack outputs S16)
- [ ] Verify frame CRC-16 for error detection

### Metadata
- [ ] Read APEv2 tags from end of MPC file
- [ ] Read SV8 RG (ReplayGain) packet for loudness info
- [ ] Read SV8 CT (Chapter Tag) packet for chapters
- [ ] Write APEv2 tags via dedicated tag editor (not FFmpeg)
- [ ] Handle UTF-8 encoding in all tag fields

### Edge Cases
- [ ] Handle SV7 files without seek table (slow seeking)
- [ ] Handle SV8 files with seek table (fast seeking)
- [ ] Handle corrupt frames (CRC-16 detection, skip/mute)
- [ ] Handle only 44100 and 48000 Hz sample rates
- [ ] Handle very short files (< 1 frame)

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
