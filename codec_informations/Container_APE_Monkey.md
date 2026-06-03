# Monkey's Audio (APE) Container — Deep Technical Reference

> **Category:** Container
> **File Extensions:** `.ape`
> **MIME Types:** `audio/x-ape`, `audio/ape`, `audio/x-monkey`
> **Standardization Body:** None — proprietary format by Matthew T. Ashland
> **Primary Specification:** Monkey's Audio SDK documentation; Hydrogenaudio wiki
> **Patent Status:** Proprietary — no known patents
> **License:** Freeware — free for personal and commercial use
> **Current Version:** 3.99 (3990) / 4.0 development
> **Active Development:** Minimal — stable but not actively developed

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Matthew T. Ashland
- **Year Created:** 2000
- **Original Purpose:** Create a high-compression lossless audio codec for music archival that rivals FLAC and WavPack in compression ratio while maintaining reasonable encode/decode speeds
- **Problem with Predecessors:** Other lossless codecs either sacrificed compression for speed or were too slow; APE aimed to balance both

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| 3.81 | 2000 | Initial release |
| 3.97 | 2004 | New header format, improved compression |
| 3.98 | 2005 | Bug fixes, minor compression improvements |
| 3.99 | 2008 | Compression improvements, multichannel support |
| 4.0 | Unreleased | Planned improvements [NEEDS VERIFICATION] |

### 1.3 Current Adoption
- **Primary use cases today:** Music archival, CD ripping, audiophile storage where maximum compression matters
- **Platforms with native support:** Windows (Monkey's Audio application), cross-platform via FFmpeg
- **Major services using this format:** Some download sites, audiophile communities
- **Hardware support:** Limited — primarily software playback
- **Status:** Legacy / declining — superseded by FLAC for most use cases due to FLAC's open nature

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Lossless waveform compression
- **Core algorithm:** Modified ANSI-C code model with block-wise compression
- **Loss mechanism:** Lossless only — no psychoacoustic model
- **Frame-based vs sample-based:** Frame-based — variable number of samples per frame
- **Fixed vs variable frame size:** Variable per compression level

### 2.2 High-Level Encoding Flow (Block Diagram)
```
Input PCM Samples
      │
      ▼
[Anti-Predictor: Remove obvious patterns]
      │
      ▼
[Block Decomposition: Split into 16-1024 sample blocks]
      │
      ▼
[Compressor Stage 1: Initial compression]
      │
      ▼
[Compressor Stage 2: Secondary compression]
      │
      ▼
[Entropy Coding: Range coder for final output]
      │
      ▼
[CRC: 32-bit CRC per frame for integrity]
      │
      ▼
Output Encoded Stream
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | Variable | Based on block size |
| Frame/block size | Variable (16–1024 samples) | Depends on compression level |
| Max channels | 8 (v3.99+) | Monophonic and multichannel |
| Max bit depth | 32-bit (internal) | 8, 16, 24, 32-bit input |
| Max sample rate | 192,000 Hz | Supported |
| Bitrate range | ~400–1400 kbps | Depends on content and preset |
| Compression levels | 5 levels | 1000–5000 |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       4D 41 43 20     'MAC '   APE file magic (with trailing space)
```

### 3.2 File-Level Layout — Version 3.99+ (New Format)

```
APE File Structure (v3.99+):

[Descriptor Block]     — 52 bytes minimum
[Header Block]         — 24 bytes
[Seek Table]           — variable
[WAV Header]           — variable (original WAV header if present)
[Audio Data]           — compressed frames
[Old Metadata]         — ID3v1 tags (optional)
[APE Tag Footer]       — 32 bytes (APEv2 tags, optional)
```

### 3.3 Descriptor Block Layout (Version 3790+)
```
Offset   Size    Field Name              Type        Valid Range   Description
-------  ------  ----------------------  ----------  -----------   ---------------------------
0x0000   4B      Magic                 char[4]     'MAC '        File signature
0x0004   2B      Version               uint16 LE   3790–3990    Version × 1000
0x0006   2B      Padding               uint16 LE   0            Padding to 8-byte align
0x0008   4B      Descriptor Length      uint32 LE   >= 52        Length of this block
0x000C   4B      Header Length          uint32 LE   24           Length of header block
0x0010   4B      Seek Table Length      uint32 LE   variable     Length of seek table
0x0014   4B      WAV Header Length      uint32 LE   variable     Original WAV header size
0x0018   4B      Audio Data Length      uint32 LE   variable     Compressed audio size
0x001C   4B      Audio Data Length High uint32 LE   0            High 32 bits for >4GB
0x0020   4B      WAV Tail Length        uint32 LE   variable     Data after audio (tags)
0x0024   4B      Reserved               uint32 LE   0            Must be zero
0x0028   2B      Format Flags           uint16 LE   bit field    See format flags table
0x002A   2B      Compression Level      uint16 LE   1000–5000    See compression levels
0x002C   4B      Sample Count           uint32 LE   > 0          Total audio samples
0x0030   4B      Block Count            uint32 LE   > 0          Total blocks/frames
0x0034   4B      Blocks Per Frame       uint32 LE   varies        Samples per frame
0x0038   2B      Bits Per Sample        uint16 LE   8,16,24,32   Input bit depth
0x003A   2B      Channels              uint16 LE   1–8          Number of channels
0x003C   4B      Sample Rate           uint32 LE   any           Samples per second
0x0040   16B     MD5                   uint8[16]   hash         Full audio MD5 hash
```

### 3.4 Format Flags (Descriptor Block)
| Bit | Name | Description |
|-----|------|-------------|
| 0 | MAC_FORMAT_FLAG_8_BIT | Audio is 8-bit (obsolete) |
| 1 | MAC_FORMAT_FLAG_CRC | Uses CRC-32 error detection (obsolete) |
| 2 | MAC_FORMAT_FLAG_PEAK_LEVEL | Peak level stored in header (obsolete) |
| 3 | MAC_FORMAT_FLAG_24_BIT | Audio is 24-bit (obsolete) |
| 4 | MAC_FORMAT_FLAG_SEEK_ELEMENTS | Seek elements present |
| 5 | MAC_FORMAT_FLAG_PEAK_LEVEL2 | Peak level stored in header v2 (obsolete) |
| 6 | MAC_FORMAT_FLAG_32_BIT | Audio is 32-bit |

### 3.5 Compression Levels
| Level | Value | Name | Block Size | Encode Speed | Notes |
|-------|-------|------|------------|--------------|-------|
| Fast | 1000 | COMPRESSION_LEVEL_FAST | 9216 samples | Fastest | Lowest compression |
| Normal | 2000 | COMPRESSION_LEVEL_NORMAL | 9216 samples | Fast | Default |
| High | 3000 | COMPRESSION_LEVEL_HIGH | 16384 samples | Medium | |
| Extra High | 4000 | COMPRESSION_LEVEL_EXTRA_HIGH | 36864 samples | Slow | |
| Insane | 5000 | COMPRESSION_LEVEL_INSANE | 73728 samples | Slowest | Maximum compression |

For version 3950+:
- Blocks per frame = 73728 × 4 = 294912 samples for Extra High and higher

### 3.6 Old Format Header (Version < 3790)
```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------
0x0000   4B      Magic                 char[4]     'MAC '
0x0004   2B      Version               uint16 LE   Version × 1000
0x0006   2B      Compression Level      uint16 LE   1000–5000
0x0008   2B      Format Flags          uint16 LE   Bit flags
0x000A   2B      Channels              uint16 LE   1 or 2
0x000C   4B      Sample Rate           uint32 LE   Samples per second
0x0010   4B      WAV Header Length      uint32 LE   Original WAV header bytes
0x0014   4B      WAV Tail Length        uint32 LE   Bytes after audio
0x0018   4B      Total Frames          uint32 LE   Number of frames
0x001C   4B      Final Frame Blocks     uint32 LE   Blocks in last frame
```

### 3.7 Audio Frame Structure
```
Each compressed frame:
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------
0x0000   N       Frame Data             byte[]      Compressed audio data
N        4B      Frame CRC              uint32 LE   CRC-32 of frame data
```

### 3.8 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Unsigned integer | Yes | Supported (obsolete format) |
| 16-bit | Signed integer | Yes | Primary format |
| 20-bit | Signed integer | Yes | Internal 24-bit processing |
| 24-bit | Signed integer | Yes | Full quality |
| 32-bit | Signed integer | Yes | Supported in v3.99+ |
| 32-bit | IEEE float | No | Not supported |

#### Supported Sample Rates
| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 8000 | Telephone | Yes | |
| 11025 | — | Yes | |
| 16000 | Wideband voice | Yes | |
| 22050 | — | Yes | |
| 32000 | Broadcast | Yes | |
| 44100 | CD audio | Yes | Most common |
| 48000 | Professional | Yes | |
| 88200 | 2× CD | Yes | |
| 96000 | High-res | Yes | |
| 176400 | 4× CD | Yes | |
| 192000 | High-res max | Yes | |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Pre-Processing Stage
- **Anti-predictor:** Removes obvious patterns from audio
- **Block decomposition:** Audio is split into blocks for compression
- **No pre-emphasis:** APE does not apply any pre-emphasis

### 4.2 Analysis / Transform Stage

#### Transform Type: Time-Domain Compression
```
APE uses a sophisticated time-domain compression approach:
  1. Anti-predictor: Removes obvious statistical patterns
  2. Block compression: Processes audio in variable-sized blocks
  3. Range coder: Final entropy coding stage

Parameters vary by compression level:
  - Compression 1000 (Fast):   Smaller blocks, simpler models
  - Compression 5000 (Insane): Larger blocks, complex models
```

### 4.3 No Psychoacoustic Model
APE is a lossless codec. There is no quality setting — the output is always bit-exact.

### 4.4 Entropy Coding — Range Coder
```
Method: Range coder (similar to arithmetic coding)

The range coder processes the residual signal after prediction.
Key properties:
  - Near-optimal compression
  - Fast encoding and decoding
  - Adaptive probability modeling
```

### 4.5 Encoder Settings / Quality Modes

#### For Lossless Codec
| Compression Level | Encoding Speed | Decode Speed | Compression Ratio | Block Size |
|------------------|---------------|--------------|------------------|------------|
| 1000 (Fast) | Fastest | Fastest | ~60–70% | 9216 samples |
| 2000 (Normal) | Fast | Fast | ~55–65% | 9216 samples |
| 3000 (High) | Medium | Medium | ~50–60% | 16384 samples |
| 4000 (Extra High) | Slow | Medium | ~45–55% | 36864 samples |
| 5000 (Insane) | Slowest | Medium | ~40–50% | 73728 samples |

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. Read first 4 bytes: 'MAC ' magic
2. Read version number
3. For v3790+:
     a. Read descriptor block (52+ bytes)
     b. Read header block
     c. Read seek table
     d. Skip WAV header
4. For v<3790:
     a. Read 24-byte header
     b. Read seek table
     c. Skip WAV header
5. Validate MD5 checksum if present
```

#### Seeking
```
APE seek table maps frame index to byte offset and sample position.

Seek to time T:
1. Compute target block: block = T × sample_rate
2. Binary search seek table for nearest keyframe
3. Decode from keyframe position
4. Decode forward until target time reached

Seek precision: Frame/block-accurate
```

### 5.2 Core Decode Pipeline
```
1. Read and validate descriptor block
     ├── Verify 'MAC ' signature
     ├── Extract: version, compression level, channels, sample rate
     └── Read format flags

2. Read seek table (if present)
     ├── Number of entries = total_frames + 1
     ├── Each entry: frame offset (uint32), frame size (uint32)

3. Read WAV header (if present)
     └── Skip original WAV header

4. For each frame (i = 0 to total_frames - 1):
     ├── Read frame data
     ├── Verify frame CRC (if format flag set)
     ├── Decode range coder output
     ├── Reconstruct audio block
     └── Output samples

5. Verify total MD5 checksum

6. Post-processing: Format output as PCM
```

### 5.3 Error Concealment
- **Corrupt frame detection:** CRC-32 check on each frame (if enabled)
- **Concealment method:** None — decode stops on error
- **Maximum consecutive errors before silence:** 1

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** APE — standalone file format
- **Overhead:** ~52+ bytes header + seek table + per-frame CRC
- **Seeking in native container:** Yes — seek table provides direct frame access
- **Multiple streams in native container:** No — single audio stream

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| APE (.ape) | APE audio | Yes | APEv2, ID3v1 | Native format |
| WAV | No | — | — | Raw PCM only |
| Matroska/MKA | No | — | — | No support |
| MP4/M4A | No | — | — | No support |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** APEv2 (at end of file)
- **Tag block location:** End of file (APEv2), beginning (ID3v1)
- **Native APE metadata:** Encoder version stored in file header

### 7.2 Standard Tag Fields — APEv2 in APE Files
| Tag Field | Internal Key (APEv2) | Max Length | Character Encoding | Multi-value | Notes |
|-----------|----------------------|------------|-------------------|-------------|-------|
| Title | Title | variable | UTF-8 | No | |
| Artist | Artist | variable | UTF-8 | No | |
| Album | Album | variable | UTF-8 | No | |
| Album Artist | Album Artist | variable | UTF-8 | No | |
| Composer | Composer | variable | UTF-8 | No | |
| Genre | Genre | variable | UTF-8 | No | |
| Year / Date | Year | 4 bytes | ASCII | No | YYYY format |
| Track Number | Track | variable | UTF-8 | No | Format "N" or "N/Total" |
| Disc Number | Disc | variable | UTF-8 | No | Format "N" or "N/Total" |
| Comment | Comment | variable | UTF-8 | No | |
| Cover Art | Cover Art (front) | variable | Binary | No | JPEG or PNG |
| ReplayGain Track Gain | REPLAYGAIN_TRACK_GAIN | variable | ASCII | No | Format "-6.20 dB" |
| ReplayGain Track Peak | REPLAYGAIN_TRACK_PEAK | variable | ASCII | No | Format "0.998459" |
| ReplayGain Album Gain | REPLAYGAIN_ALBUM_GAIN | variable | ASCII | No | |
| ReplayGain Album Peak | REPLAYGAIN_ALBUM_PEAK | variable | ASCII | No | |

### 7.3 APEv2 Tag Structure (32-byte Footer)
```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------
0x0000   8B      Preamble               char[8]     'APETAGEX'
0x0008   4B      Version                uint32 LE   2000 (APEv2)
0x000C   4B      Tag Size               uint32 LE   Size including footer, excluding header
0x0010   4B      Item Count             uint32 LE   Number of tag items
0x0014   4B      Flags                  uint32 LE   Bit 31=has header, 30=has footer, 29=is header
0x0018   8B      Reserved               uint64 LE   Must be zero
```

### 7.4 APE Tag Item Structure
```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------
0x0000   4B      Value Size             uint32 LE   Length of value in bytes
0x0004   4B      Item Flags            uint32 LE   Bit 0=read-only, bits 1-2=data type
0x0008   N       Key                   char[N]     Item key (ASCII, 2-255 chars)
N        1B      Null Terminator        uint8       0x00
N+1      M       Value                  byte[M]     Item value

Item flags data type bits:
  00 = text (UTF-8)
  01 = binary
  10 = external (file reference)
  bit 0 = read-only flag
```

### 7.5 Metadata Compatibility Matrix
| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| APEv2 | ✓ | ✓ | ✓ | Highest |
| ID3v1 | ✓ | ✓ | ✓ | Lowest |

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   ape           # used with -c:a
AV_CODEC_ID:        AV_CODEC_ID_APE  # C constant in libavcodec/codec_id.h
Format Name (CLI):  ape           # used with -f
Encoder(s):         None          # FFmpeg does NOT encode APE natively
Decoder(s):         ape           # FFmpeg has APE decoder
Muxer(s):          ape           # FFmpeg has APE muxer
Demuxer(s):        ape           # FFmpeg has APE demuxer
```

### 8.2 FFmpeg Encoding — Not Supported
```bash
# FFmpeg cannot encode APE natively
# Use Monkey's Audio application or MAC (mac-port) encoder
# Available at: https://www.monkeysaudio.com/
```

### 8.3 FFmpeg Decoding — Full CLI Reference

```bash
# Decode APE to WAV (lossless)
ffmpeg -i input.ape \
  -c:a pcm_s16le \
  output.wav

# Decode APE to FLAC (lossless to lossless)
ffmpeg -i input.ape \
  -c:a flac -compression_level 8 \
  output.flac

# Decode APE to any format
ffmpeg -i input.ape \
  -c:a libmp3lame -q:a 2 \
  output.mp3

# Decode with error concealment (continue on error)
ffmpeg -err_detect ignore_err -i input.ape -c:a pcm_s16le output.wav

# Probe format details
ffprobe -v quiet -print_format json -show_streams -show_format input.ape
```

### 8.4 FFmpeg Decoding — libavformat C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>

// ─── Complete decode program skeleton ────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.ape", NULL, NULL);
avformat_find_stream_info(fmt_ctx, NULL);

// Find audio stream
int audio_idx = av_find_best_stream(fmt_ctx, AVMEDIA_TYPE_AUDIO, -1, -1, NULL, 0);
AVStream *stream = fmt_ctx->streams[audio_idx];

// Find APE decoder
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

### 8.5 FFmpeg Metadata Handling

```bash
# Read all metadata as JSON
ffprobe -v quiet -print_format json -show_format input.ape | jq .format.tags

# APE files typically have APEv2 tags
# FFmpeg reads APEv2 tags from APE files

# Strip all metadata
ffmpeg -i input.ape -c copy -map_metadata -1 output.ape

# Note: FFmpeg cannot write APE tags
# Use external tools like apetag or monkeysaudio for tag editing
```

### 8.6 Quality / Fidelity Decision Table
| Use Case | Settings | Expected Size | Notes |
|----------|----------|---------------|-------|
| Archival | COMPRESSION_LEVEL_INSANE (5000) | 40–50% of WAV | Maximum compression |
| General archival | COMPRESSION_LEVEL_HIGH (3000) | 50–60% of WAV | Good balance |
| Space-saving | COMPRESSION_LEVEL_NORMAL (2000) | 55–65% of WAV | Fast |
| Speed priority | COMPRESSION_LEVEL_FAST (1000) | 60–70% of WAV | Fastest |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
```
APE Seek Table:
  Location:    After descriptor/header block, before audio data
  Entry count: total_frames + 1 (includes end marker)
  Entry size:  8 bytes (2 × uint32)
  Entry format:
    [0x00-0x03]  frame_position (uint32 LE) — byte offset from audio start
    [0x04-0x07]  frame_size (uint32 LE) — frame size in bytes
  
  Max entries: Limited by file size
```

### 9.2 Gapless Playback Data
```
APE does not store gapless playback data.
Encoder delay:   Not stored
Padding:         Not stored
Storage:         None
Note:            Gapless playback may have slight overlap between tracks
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | Seek table enables streaming |
| Algorithmic encoder delay | Variable | Based on block size |
| Live encoding feasible | No | Designed for file-based compression |
| HTTP progressive download | Yes | With seek table |
| HTTP Live Streaming (HLS) | No | Not segmented |
| WebRTC / RTP transport | No | Not typical transport |
| Minimum decode buffer | 1 block | ~100–500 ms |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | Layout Name | Channel Order | FFmpeg Layout Constant | Notes |
|----------|-------------|---------------|----------------------|-------|
| 1 | Mono | C | AV_CHANNEL_LAYOUT_MONO | Supported |
| 2 | Stereo | L, R | AV_CHANNEL_LAYOUT_STEREO | Supported |
| 3 | — | — | — | Limited support |
| 4 | Quad | FL, FR, RL, RR | AV_CHANNEL_LAYOUT_QUAD | Supported (v3.99+) |
| 5+ | — | — | — | Limited support |

### 11.2 Downmix Coefficients
```
APE multichannel support is limited.
Most APE files are stereo.
Downmix not natively supported; use FFmpeg filters.
```

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 32-bit (internal) | 8, 16, 24, 32-bit input supported |
| Max sample rate | 192,000 Hz | Supported |
| Float support | No | Integer PCM only |
| DSD support | No | Not applicable |

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

| Platform / API | Encode | Decode | FFmpeg Flag | Notes |
|----------------|--------|--------|-------------|-------|
| Native | Windows only | Windows only | N/A | Monkey's Audio app |
| FFmpeg | No | Yes | Built-in | Best cross-platform |
| Software-only | Yes | Yes | N/A | No hardware APE support |

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Decoder-Specific Issues
| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| No encoder | All | Use Monkey's Audio app |
| Large file support | < 4.0 | Use v3.99 format for >4GB |
| 32-bit support | < 2.8 | Decode to 24-bit only |

### 14.2 Interoperability Issues
- **APE tag writing:** FFmpeg cannot write APEv2 tags
- **Cross-platform:** Official encoder Windows-only
- **CUE sheet splitting:** APE + CUE for album splitting requires external tools

### 14.3 Edge Cases to Handle in Converter
- **Corrupt frame:** Report error, stop decode
- **Truncated file:** Decode available frames
- **Missing seek table:** Scan for frame sync codes
- **Very large files (>4GB):** Use v3.99 format

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM APE
| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → FLAC | `ffmpeg -i in.ape -c:a flac -compression_level 8 out.flac` | No (FFmpeg can't write APEv2) | Lossless decode |
| → WAV | `ffmpeg -i in.ape -c:a pcm_s16le out.wav` | No | Lossless decode |
| → ALAC | `ffmpeg -i in.ape -c:a alac out.m4a` | No | Lossless decode |
| → MP3 | `ffmpeg -i in.ape -c:a libmp3lame -q:a 0 out.mp3` | No | Generation loss |

### 15.2 Converting TO APE
| Source | Tool | Metadata Preserved | Quality Notes |
|--------|------|-------------------|---------------|
| FLAC → | Monkey's Audio app | No | Lossless |
| WAV → | Monkey's Audio app | No | Lossless |
| ALAC → | Monkey's Audio app | No | Lossless |

### 15.3 Lossless Round-Trip Verification
```bash
# Decode APE to WAV
ffmpeg -i input.ape -c:a pcm_s16le decoded.wav

# Compare checksums
md5sum decoded.wav

# Verify MD5 from APE file
ffprobe -v quiet -show_format input.ape | grep md5
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Encoder | Decoder | URL |
|---------|----------|---------|---------|---------|-----|
| Monkey's Audio | C++ | Freeware | Reference | Reference | monkeysaudio.com |
| FFmpeg APE | C | LGPL 2.1+ | No | Good | ffmpeg.org |
| MAC (mac-port) | C | GPL | Good | Reference | sourceforge |
| libapetag | C | BSD | No | Yes | github |

### Build Instructions
```bash
# Build FFmpeg with APE support
./configure --enable-decoder=ape --enable-demuxer=ape --enable-muxer=ape
make -j$(nproc)
make install

# Build MAC (mac-port) encoder for Linux/macOS
git clone https://sourceforge.net/projects/mac-port/
cd mac-port
mkdir build && cd build
cmake .. && make
```

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **Monkey's Audio:** https://www.monkeysaudio.com/
- **APE Format (Hydrogenaudio):** https://wiki.hydrogenaud.io/index.php?title=APE
- **APEv2 Specification:** https://wiki.hydrogenaud.io/index.php?title=APEv2_specification

### Technical Resources
- FFmpeg decoder: `ffmpeg -decoders | grep ape`
- FFmpeg muxer: `ffmpeg -muxers | grep ape`
- Multimedia Wiki: https://wiki.multimedia.cx/index.php?title=APE

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] Identify FFmpeg `./configure` flags: `--enable-decoder=ape --enable-demuxer=ape`
- [ ] Verify `ffmpeg -decoders` output confirms APE decoder is available
- [ ] Note: FFmpeg does NOT have APE encoder
- [ ] For encoding, use Monkey's Audio application or MAC port

### Encoding Pipeline
- [ ] Use official Monkey's Audio encoder for APE encoding
- [ ] Select appropriate compression level
- [ ] Note: FFmpeg cannot be used for APE encoding

### Decoding Pipeline
- [ ] Read descriptor block (v3790+) or old header (v<3790)
- [ ] Extract compression level, channels, sample rate, total samples
- [ ] Read seek table if present
- [ ] Decode each frame using APE range decoder
- [ ] Verify frame CRC if enabled
- [ ] Validate MD5 checksum at end

### Metadata
- [ ] Read APEv2 tags from end of file
- [ ] Map APEv2 keys to standard field names
- [ ] Note: FFmpeg cannot write APEv2 tags
- [ ] Preserve cover art as binary APEv2 item

### Quality & Verification
- [ ] APE is lossless — verify bit-exact decode
- [ ] Test with all compression levels
- [ ] Test with old and new format versions

### Edge Cases
- [ ] Handle files with missing seek table
- [ ] Handle corrupt frames gracefully
- [ ] Handle very large files (>4GB)
- [ ] Handle multichannel files

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
