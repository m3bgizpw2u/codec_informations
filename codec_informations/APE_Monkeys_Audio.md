# Monkey's Audio (APE) — Deep Technical Reference
> **Category:** Lossless
> **File Extensions:** `.ape`
> **MIME Types:** `audio/x-ape`, `audio/ape`
> **Standardization Body:** None (proprietary, developer Matthew T. Ashland)
> **Primary Specification:** http://www.monkeysaudio.com/theory.html (unofficial)
> **Patent Status:** Patented (algorithm is proprietary)
> **License:** Proprietary freeware
> **Current Version:** 4.67 (2024)
> **Active Development:** Yes (maintenance mode; last significant update ~2017)

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Matthew T. Ashland
- **Year Created:** 2000
- **Original Purpose:** Create a lossless audio codec optimized for maximum compression ratio, prioritizing file size reduction over encoding speed
- **Problem with Predecessors:** Existing lossless codecs (FLAC, Shorten) offered either fast encoding or good compression, but not both. APE was designed to push compression to the limit even at extreme CPU cost

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| 3.0 | 2000 | Initial public release |
| 3.1 | 2001 | Improved compression, bug fixes |
| 3.2 | 2002 | Filter improvements, better compression |
| 3.3 | 2003 | Faster decoding |
| 3.4 | 2004 | Stability improvements |
| 3.5 | 2004 | Further optimization |
| 3.6 | 2005 | Enhanced predictors |
| 3.7 | 2006 | Refined entropy coding |
| 3.8 | 2007 | Final pre-4.0 release |
| 3.9 | 2009 | Pre-4.0 version with improved tagging |
| 4.0 | 2009 | Major rewrite, APEv2 tags, seek table format change |
| 4.1 | 2010 | Minor improvements |
| 4.2 | 2012 | Performance optimizations |
| 4.3 | 2014 | Bug fixes, 64-bit support |
| 4.4 | 2015 | Compatibility improvements |
| 4.5 | 2016 | Minor enhancements |
| 4.6 | 2017 | Last major update |
| 4.67 | 2024 | Latest version (maintenance) |

### 1.3 Current Adoption
- **Primary use cases today:** Archival of music collections where maximum compression is prioritized over encoding time, niche enthusiast communities
- **Platforms with native support:** Windows (primary), macOS, Linux, BSD
- **Major services using this format:** Very limited; APE is largely deprecated in streaming and professional contexts
- **Hardware support:** Very limited. Rockbox has partial support. Most DAPs do not support APE natively
- **Status:** Declining. FLAC dominates the lossless archival space due to its open nature and comparable (within ~2%) compression at dramatically lower CPU cost

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Predictive (lossless only)
- **Core algorithm:** Adaptive linear prediction (IIR filters) + rangecoder (arithmetic coding variant) + channel decorrelation
- **Loss mechanism:** Lossless only; no lossy mode exists
- **Frame-based vs sample-based:** Frame-based (fixed block size per compression level)
- **Fixed vs variable frame size:** Fixed block size within a file, varies by compression level (smaller blocks = better compression, slower)

### 2.2 High-Level Encoding Flow (Block Diagram)
```
Input PCM Samples
      │
      ▼
[Pre-processing: Delta encoding between samples]
      │
      ▼
[Channel Decorrelation: L/R → L, R' = R - floor(L/2)]
      │
      ▼
[IIR Filtering: Adaptive linear prediction (1–3 stages)]
      │  Each filter: out[n] = in[n] + Σ(coeff[i] × in[n-i])
      │  Coefficients updated based on residual sign
      ▼
[Entropy Coding: Rangecoder (arithmetic coding variant) + Rice coding]
      │
      ▼
[Frame Packing: Header + compressed data + CRC]
      │
      ▼
Output .APE File (with optional APEv2 tag at end)
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | ~4096 samples (1 block) | Affects real-time use |
| Frame size | 128–4608 samples | Varies by compression level |
| Max channels | 2 | Only stereo or mono supported in file format |
| Max bit depth | 32-bit integer | 8, 16, 24-bit input supported |
| Max sample rate | 192 kHz | [NEEDS VERIFICATION] |
| Bitrate range | ~300–900 kbps (CD audio) | Depends on compression level |
| Complexity | O(n) — extremely high | Insane mode is the slowest lossless codec |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       4D 41 43 44    "MAC "   APE file magic
0x0004  2       XX XX           —         Version (e.g., 0x0FA0 = 4.00, LE uint16)
```

**Version-specific structures:**
- **Version 3.96 and earlier:** Uses MAC_HEADER_OLD structure
- **Version 3.97 and later:** Uses MAC_HEADER with APE_DESCRIPTOR preamble

### 3.2 File-Level Header Layout (Version 3.97+)

```
Offset   Size    Field Name              Type        Valid Range   Description
-------  ------  ----------------------  ----------  ------------  ---------------------------
0x0000   4B      Magic                  char[4]     "MAC "        File identifier
0x0004   2B      Version                uint16 LE   0x0970–      APE format version (×1000)
0x0006   4B      Parameter              uint32 LE   —             Compression level descriptor
0x000A   4B      Format Flags           uint32 LE   bitfield      See format flags below
0x000E   4B      Channel Count          uint32 LE   1–2           Number of channels
0x0012   4B      Sample Rate            uint32 LE   8000–192000  Samples per second
0x0016   8B      Total Samples          uint64 LE   0–2^63        Total audio samples
0x001E   8B      First Block Location   uint64 LE   —             Offset of first audio block
0x0026   8B      Total Blocks          uint64 LE   —             Number of data blocks
0x002E   4B      Final Block Samples    uint32 LE   0–65535       Samples in last block
0x0032   4B      Total Bit Rate         uint32 LE   —             Average bitrate (bits/sec)
```

### 3.3 Format Flags (32-bit bitfield)
```
Bits     Name                          Description
--------  ----------------------------  ------------------------------------------
0        HAS_PEAK_LEVEL                Peak audio level stored (obsolete)
1        HAS_REAL_PEAK_LEVEL           Real peak audio level stored
2        HAS_SEEK_ELEMENTS             Number of seek elements stored
3        SET_PEAK_LEVEL                Peak level is set
4        RESET_PEAK_LEVEL              Reset peak level
5        IGNORE_DULPEAK_CHUNK          Ignore "DUPPEAK" chunk
6        CREATE_PEAK_LEVEL             Create peak level chunk
7–23     —                             Reserved
24       MPEG_WALL                    [NEEDS VERIFICATION]
25       CDROM_IMAGE                   [NEEDS VERIFICATION]
26       Reserved (deprecated)          Reserved
27       NEW_DEMUX_VERSION             New demuxer version (v3.97+)
```

### 3.4 Frame Header Layout
Each audio frame begins with:
```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------
0x0000   4B      Frame CRC              uint32 LE   CRC-32 of decoded frame
0x0004   4B      Frame Size             uint32 LE   Compressed frame size in bytes
```

**Version 3.96 and earlier:** Frame size is stored in 4 bytes before the CRC.

### 3.5 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Unsigned integer | Yes | Stored as signed internally |
| 16-bit | Signed integer | Yes | Standard CD audio |
| 20-bit | Signed integer | [NEEDS VERIFICATION] | [NEEDS VERIFICATION] |
| 24-bit | Signed integer | Yes | High-res |
| 32-bit | Signed integer | [NEEDS VERIFICATION] | [NEEDS VERIFICATION] |
| 32-bit | IEEE float | No | Not supported |
| 64-bit | IEEE double | No | Not supported |

#### Supported Sample Rates
| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 8000 | Telephone | Yes | |
| 11025 | — | Yes | |
| 16000 | Wideband | Yes | |
| 22050 | — | Yes | |
| 32000 | Broadcast | Yes | |
| 44100 | CD audio | Yes | |
| 48000 | Professional | Yes | |
| 88200 | 2× CD | Yes | |
| 96000 | High-res | Yes | |
| 176400 | 4× CD | Yes | |
| 192000 | High-res max | Yes | |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Pre-Processing Stage
- **DC offset removal:** Not explicitly performed; handled by the predictor
- **Pre-emphasis filter:** None
- **Windowing function:** None; APE is a time-domain codec
- **Level normalization:** Input samples normalized to internal precision; APE uses 32-bit signed integers internally
- **Stereo decorrelation pre-step:** L/R channels are decorrelated before main encoding:
  ```
  L_out = L
  R_out = R - floor(L / 2)
  ```
  This provides initial compression for correlated stereo signals.

### 4.2 Analysis / Transform Stage

#### Transform Type: None (pure time-domain predictive)

APE uses three processing stages applied sequentially:

1. **Frame blocking:** Audio is divided into fixed-size blocks
2. **Channel decorrelation:** L/R → L, R' transformation
3. **IIR filtering:** Multiple adaptive filter stages
4. **Entropy coding:** Rangecoder compression of residuals

### 4.3 IIR Filter Stage (Adaptive Linear Prediction)

The core of APE's compression is the IIR (Infinite Impulse Response) filter stage. APE uses multiple stages of first-order IIR filters with coefficient adaptation.

**Filter structure:**
```
For each sample:
  out[n] = in[n] + Σ(coeff[i] × in[n-i]) for i = 1 to order

Coefficient update:
  if in[n] > 0:  coeff[i] -= in[n-i]
  if in[n] < 0:  coeff[i] += in[n-i]
  (scaled by a small learning rate)
```

**Filter orders by compression level:**
| Level | Filter Order | Notes |
|-------|-------------|-------|
| Fast (1000) | 1 stage | Single first-order filter |
| Normal (2000) | 2 stages | Two first-order filters |
| High (3000) | 3 stages | Three first-order filters |
| Extra High (4000) | 4 stages | [NEEDS VERIFICATION] |
| Insane (5000) | 5+ stages | [NEEDS VERIFICATION] |

**Internal representation:**
- Coefficients are stored as 32-bit signed integers with implied fractional precision
- Initial coefficients are zero; they adapt quickly to the audio signal

### 4.4 Entropy Coding — Rangecoder

APE uses a **rangecoder**, which is an arithmetic coding variant optimized for audio data.

**Range coding fundamentals:**
1. Maintain an interval [low, high) representing all possible output codes
2. For each symbol, subdivide the interval proportionally to the symbol's probability
3. Output the shortest prefix that uniquely identifies the interval
4. The decoder reconstructs the interval and derives symbols

**APE's rangecoder specifics:**
```
State maintained per frame:
  uint32_t low;     // Low end of interval
  uint32_t range;   // Length of interval
  uint32_t help;    // Intermediate value (bytes_to_follow)
  uint8_t  buffer;  // Input/output buffer

Normalize: ensure range stays within power-of-two boundaries
  while (range < 0x01000000) {
      range <<= 8;
      low   <<= 8;
      if (output_pending) output byte
  }
```

**Rice parameter adaptation:**
- For each block, the optimal Rice parameter k is computed from the mean absolute value of residuals
- k is computed as: k = floor(log2(mean_of_residuals))
- The decoder derives k from the same algorithm, so no explicit transmission is needed

**Residual encoding:**
1. Compute Rice parameter k from recent samples
2. Encode quotient: unary-coded (k+1 zeroes followed by 1)
3. Encode remainder: k-bit binary code
4. Append sign bit
5. Update k based on new residuals (rolling average)

### 4.5 Encoder Settings / Quality Modes

APE's compression levels are encoded as the Parameter field in the header (multiples of 1000):

| Compression Level | Parameter | Frame Size | Encoding Speed | Decode Speed | Compression Ratio |
|---|---|---|---|---|---|
| Fast | 1000 | 4608 | ~3× realtime | ~15× realtime | ~55–60% |
| Normal | 2000 | 4096 | ~1.5× realtime | ~10× realtime | ~52–56% |
| High | 3000 | 3072 | ~0.7× realtime | ~7× realtime | ~50–54% |
| Extra High | 4000 | 2304 | ~0.3× realtime | ~5× realtime | ~48–52% |
| Insane | 5000 | 1280 | ~0.05× realtime | ~3× realtime | ~46–50% |

**Notes on compression ratios:**
- Ratios are for CD audio (16-bit/44.1kHz stereo)
- Actual ratios vary significantly based on source material
- Classical/minimal music compresses better (~40%) than complex rock/pop (~55%)
- APE "Insane" mode is often only 1–2% smaller than "Extra High" but takes 5–10× longer

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. Read magic bytes: "MAC " at offset 0x0000
2. Read version: validate in range 0x0970–0x1000+
3. Read format flags: verify valid flag bits
4. Validate sample rate: must be > 0
5. Validate channel count: must be 1 or 2
6. For each frame:
   a. Read frame CRC (4 bytes)
   b. Read frame size (4 bytes)
   c. Read and decompress frame
   d. Verify CRC of decoded samples
7. If CRC mismatch: attempt to resync or mute frame
```

#### Seeking
- **Seek table:** Embedded in file header (if HAS_SEEK_ELEMENTS flag set)
- **Seek table format:** Array of uint32 values, one per frame, storing byte offsets
- **Seek precision:** Frame-accurate (block-based)
- **Note:** FFmpeg's APE decoder ignores the seek table and performs linear scan

### 5.2 Core Decode Pipeline
```
1. Read file header
   ├── Validate magic "MAC "
   ├── Parse version and format flags
   ├── Extract channel count, sample rate
   └── Compute total blocks from total_samples / frame_size

2. Initialize rangecoder
   ├── Set initial low = 0, range = 0xFFFFFFFF
   └── Read initial bytes from compressed stream

3. For each frame to decode:
   a. Read frame header (CRC + size)
   b. Initialize entropy decoder state
   c. Initialize IIR filter state
   d. For each sample in frame:
      - Decode residual from rangecoder
      - Decode Rice parameter k (rolling adaptation)
      - Reconstruct sample:
        residual = unary_part × 2^k + binary_part
        if signed: residual = -residual if sign_bit else residual
        sample = residual + filter_output
      - Apply IIR filters in reverse order
      - Update filter coefficients
      - De-correlate channels (if stereo)

4. Verify frame CRC
   ├── Compare stored CRC with computed CRC
   └── On mismatch: mute frame, continue

5. Output samples
   ├── Interleave channels if stereo
   └── Clip to output bit depth
```

### 5.3 Frame Recovery from Corruption

APE frames are independent (no inter-frame dependencies), which enables recovery from corruption:

**CRC checking:** Each frame has a 32-bit CRC of the decoded samples. On CRC failure:
1. The decoder attempts to find the next valid frame by scanning for the next frame's CRC/size header
2. If found, decoding resumes from the next frame
3. If not found within a reasonable range, the file is considered corrupt

**Partial recovery:**
- Audio up to the corrupted frame is preserved
- Corrupted frames are replaced with silence
- Subsequent frames are decoded normally

**Known issues:**
- Seek table corruption can cause catastrophic seeking failures
- Tag operations that corrupt the APEv2 tag can render the file unplayable if the tag writer is buggy

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** APE (.ape) is the native format; it is a self-contained file with audio frames + optional APEv2 tag
- **Overhead:** ~0.1–1% (frame CRCs + headers)
- **Seeking in native format:** Yes — by seek table index or linear scan
- **Multiple streams:** No — one audio stream per .ape file

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| .APE (native) | Yes | Yes — seek table | APEv2 | Primary format |
| Matroska/MKA | Yes | Yes | APEv2 | Via LIBMATROSKA | [NEEDS VERIFICATION] |
| RIFF/WAVE | No | — | — | Not supported |
| AIFF | No | — | — | Not supported |
| MP4/M4A | No | — | — | Not supported |
| OGG | No | — | — | Not supported |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** APEv2 (in .ape files, at end of file)
- **Tag block location:** End of file, after audio data
- **Tag block identifier:** `APETAGEX` (0x41 50 45 54 41 47 45 58)
- **APEv1:** Used in files created with APE 3.x and earlier (deprecated)

### 7.2 Standard Tag Fields — Complete Reference
| Tag Field | Internal Key | Max Length | Char Encoding | Multi-value | Notes |
|-----------|---------------------------|------------|---------------|-------------|-------|
| Title | Title | 512 bytes | UTF-8 | No | |
| Artist | Artist | 512 bytes | UTF-8 | No | |
| Album | Album | 512 bytes | UTF-8 | No | |
| Album Artist | Album Artist | 512 bytes | UTF-8 | No | |
| Composer | Composer | 512 bytes | UTF-8 | No | |
| Genre | Genre | 256 bytes | UTF-8 | No | |
| Year / Date | Year | 32 bytes | UTF-8 | No | Format: YYYY |
| Track Number | Track | 32 bytes | UTF-8 | No | Format: "N" or "N/Total" |
| Disc Number | Disc | 32 bytes | UTF-8 | No | Format: "N" or "N/Total" |
| Comment | Comment | 512 bytes | UTF-8 | No | |
| ReplayGain Track Gain | REPLAYGAIN_TRACK_GAIN | 32 bytes | ASCII | No | Format: "-6.20 dB" |
| ReplayGain Track Peak | REPLAYGAIN_TRACK_PEAK | 32 bytes | ASCII | No | Format: "0.998459" |
| Cover Art | Cover Art (Front) | 16 MB | Binary | No | JPEG/PNG embedded |
| Encoder | Encoder | 256 bytes | UTF-8 | No | Software name + version |

### 7.3 Cover Art Storage
Cover art in APE files is stored as a standard APEv2 binary item:
```
APEv2 Item structure:
  [Key length: 4 bytes LE]     Key string (e.g., "Cover Art (Front)")
  [Value length: 8 bytes LE]   Value length (bytes)
  [Flags: 4 bytes LE]          Item flags (0x03 = binary)
  [Value data]                  Binary image data (JPEG/PNG)
```

### 7.4 APEv2 Tag Structure

```
Offset   Size    Field Name              Description
-------  ------  ----------------------  ----------------------------------
0x0000   8B      Tag ID                 "APETAGEX" (0x41 50 45 54 41 47 45 58)
0x0008   4B      Version                APE tag version (2000 = v2.0)
0x000C   4B      Tag Size               Total tag data size (excluding header/footer)
0x0010   4B      Item Count             Number of tag items
0x0014   4B      Flags                  0x00000000 = header, 0x00000002 = footer
0x0018   variable Tag Items             Tag items (sorted by size, ascending)
         variable Padding               Null bytes (for alignment)
0x0000   8B      Tag ID                 "APETAGEX" (footer only)
0x0008   4B      Version                APE tag version
0x000C   4B      Tag Size               Same as header
0x0010   4B      Item Count             Same as header
0x0014   4B      Flags                  0x00000002 = footer
```

### 7.5 APE Tag Item Structure

```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  --------------------------
0x0000   4B      Key Length             uint32 LE   Length of key string
Variable  Key Length Key                 char[]       Tag key (e.g., "Artist")
0x0000   8B      Value Length           uint64 LE    Length of value data
0x0008   4B      Flags                  uint32 LE    Item flags
  bit 0         Is UTF-8               Text stored as UTF-8
  bit 1         Is binary              Binary data (e.g., cover art)
  bit 2         Is external            External data (file path reference)
Variable  Value Length Value             uint8[]       Value data
```

### 7.6 Metadata Compatibility Matrix
| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| APEv1 | ✓ | ✓ (legacy) | ✓ | Lowest |
| APEv2 | ✓ | ✓ | ✓ | Highest |

**Note:** APEv2 is the only tag format used in APE files version 4.0+. Tagging in APE is notoriously fragile; buggy tag editors can corrupt the file.

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   ape
AV_CODEC_ID:        AV_CODEC_ID_APE
Format Name (CLI):  ape
Encoder(s):         NONE (FFmpeg does NOT support APE encoding)
Decoder(s):         ape (native)
Muxer(s):           ape (native .ape muxer — writes, no encoder!)
Demuxer(s):         ape (native .ape demuxer)
```

**CRITICAL:** FFmpeg does NOT encode APE files. The APE codec is decode-only in FFmpeg. Use the reference Monkey's Audio CLI for encoding.

### 8.2 FFmpeg Encoding — NOT SUPPORTED

FFmpeg cannot encode APE files. To create APE files, use the reference Monkey's Audio GUI or CLI.

### 8.3 FFmpeg Decoding — Full CLI Reference

```bash
# Decode to raw PCM WAV
ffmpeg -i input.ape \
  -c:a pcm_s16le \
  output.wav

# Decode with automatic format detection
ffmpeg -i input.ape output.wav

# Decode with higher bit depth output
ffmpeg -i input.ape -c:a pcm_s24le output_24bit.wav
ffmpeg -i input.ape -c:a pcm_s32le output_32bit.wav

# Decode and resample
ffmpeg -i input.ape -ar 48000 output_48k.wav

# Decode multi-channel APE (stereo) to mono
ffmpeg -i input.ape -ac 1 output_mono.wav

# Extract APE stream without re-encoding
ffmpeg -i input.ape -c:a copy output.ape

# Probe format details
ffprobe -v quiet -print_format json -show_streams -show_format input.ape
```

### 8.4 FFmpeg Decoding — libavcodec C API

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

// Find and open decoder
const AVCodec *dec = avcodec_find_decoder(stream->codecpar->codec_id);
if (!dec) {
    fprintf(stderr, "APE decoder not found\n");
    exit(1);
}

AVCodecContext *dec_ctx = avcodec_alloc_context3(dec);
avcodec_parameters_to_context(dec_ctx, stream->codecpar);
avcodec_open2(dec_ctx, dec, NULL);

// Decode loop
AVPacket *pkt = av_packet_alloc();
AVFrame  *frm = av_frame_alloc();

while (av_read_frame(fmt_ctx, pkt) >= 0) {
    if (pkt->stream_index == audio_idx) {
        int ret = avcodec_send_packet(dec_ctx, pkt);
        if (ret < 0) {
            fprintf(stderr, "Error sending packet: %s\n", av_err2str(ret));
            continue;
        }
        while (ret >= 0) {
            ret = avcodec_receive_frame(dec_ctx, frm);
            if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF) break;
            if (ret < 0) {
                fprintf(stderr, "Error receiving frame: %s\n", av_err2str(ret));
                break;
            }
            // frm->data[0] contains interleaved PCM samples
            // frm->nb_samples = samples in this frame
            // frm->sample_rate = sample rate
            // frm->format = AV_SAMPLE_FMT_S16 (typically)
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

# Strip all metadata
ffmpeg -i input.ape -c:a copy -map_metadata -1 output_stripped.ape

# NOTE: FFmpeg cannot write APE tags directly
# Use external tools: APE Tag Editor, Mp3tag, etc.
```

#### FFmpeg Internal Metadata Key Mapping
| Standard Field | FFmpeg Key | APE Native Key | Notes |
|----------------|------------|---------------|-------|
| Title | title | Title | |
| Artist | artist | Artist | |
| Album | album | Album | |
| Album Artist | album_artist | Album Artist | |
| Track Number | track | Track | |
| Disc Number | disc | Disc | |
| Genre | genre | Genre | |
| Date/Year | date | Year | |
| Comment | comment | Comment | |
| Encoder | encoder | Encoder | Auto-set by reference encoder |

### 8.6 Quality / Fidelity Decision Table
| Use Case | Settings | Expected Size | Notes |
|----------|----------|---------------|-------|
| Archival / maximum compression | Reference "Insane" (5000) | ~46–50% of PCM | Extremely slow encode |
| Good compression, reasonable speed | Reference "Extra High" (4000) | ~48–52% of PCM | Recommended |
| Balanced | Reference "High" (3000) | ~50–54% of PCM | Good choice |
| Fast encoding | Reference "Normal" (2000) | ~52–56% of PCM | FLAC -5 is faster |
| Quick encode | Reference "Fast" (1000) | ~55–60% of PCM | Not competitive with FLAC |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
```
Seek table structure (stored in file header):
  Location:    File header (version 3.97+)
  Entry size:  4 bytes (uint32)
  Entry count: Number of audio frames
  Entry format:
    [0x00–0x03]  frame_byte_offset (uint32 LE)
  Storage:      In header if HAS_SEEK_ELEMENTS flag is set

Maximum entries: Limited by file size (one uint32 per frame)
```

**Note:** The seek table stores byte offsets for each frame. This enables O(1) seeking to any frame, but requires the seek table to be read into memory first.

### 9.2 Gapless Playback Data
```
Encoder delay:   ~4096 samples (1 block at highest compression)
Padding:        ~4096 samples (1 block at highest compression)
Storage:        Total samples in file header
                 = exact count of audio samples
                 (encoder delay is implicitly the first frame's worth)

FFmpeg gapless flags: N/A
Gapless detection:    Compare file header total_samples with expected
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | Frame-based; can start after first frame |
| Algorithmic encoder delay | ~4096 samples | 1 block at max compression |
| Live encoding feasible | No | Requires entire file for optimal compression |
| HTTP progressive download | Yes | Frame-based seeking works over HTTP |
| HTTP Live Streaming (HLS) | No | No native HLS support |
| DASH streaming | No | No DASH muxing for APE |
| WebRTC / RTP transport | No | Not designed for real-time transport |
| Minimum decode buffer | 1 frame | Variable by level (~128–4608 samples) |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | Layout Name | Channel Order | FFmpeg Layout Constant | Supported |
|----------|-------------|---------------|------------------------|-----------|
| 1 | Mono | C | AV_CHANNEL_LAYOUT_MONO | Yes |
| 2 | Stereo | L, R | AV_CHANNEL_LAYOUT_STEREO | Yes |
| >2 | Multi-channel | — | — | **No** |

**Note:** APE files only support mono (1 channel) or stereo (2 channels). Multi-channel audio must be downmixed to stereo before encoding.

### 11.2 Downmix Coefficients (Not Applicable)
APE does not support multi-channel input, so no downmix coefficients are defined.

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 32-bit integer | [NEEDS VERIFICATION] |
| Max sample rate | 192 kHz | [NEEDS VERIFICATION] |
| Float support | No | Not supported |
| DSD support | No | Not supported |
| 20-bit support | [NEEDS VERIFICATION] | [NEEDS VERIFICATION] |
| 24-bit support | Yes | Standard high-res |

```bash
# High-res 24-bit/96kHz decoding (FFmpeg)
ffmpeg -i input_24bit_96k.ape -c:a pcm_s24le output_hires.wav

# NOTE: FFmpeg cannot ENCODE APE. Use reference encoder for high-res APE.
```

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

| Platform / API | Encode | Decode | FFmpeg Flag | Notes |
|----------------|--------|--------|-------------|-------|
| Native (reference) | Yes | Yes | N/A | Monkey's Audio CLI/GUI |
| FFmpeg native | **No** | Yes | `-c:a ape` | Decode only |
| NVIDIA NVENC/NVDEC | No | No | — | Not applicable |
| Intel QSV | No | No | — | Not applicable |
| Apple AudioToolbox | No | Yes | Via AudioQueue | Some players |
| Rockbox | Partial | Yes | — | Limited codec support |
| Android | No | Yes | Via FFmpeg | Via third-party apps |

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 Known Bugs

| Issue | Affected Versions | Details | Workaround |
|-------|------------------|---------|------------|
| Seek table corruption | All versions | Seek table can become corrupted if tags are edited by buggy software; causes seeking to fail or produce garbage | Use Verify mode in official software; avoid third-party tag editors |
| Frame CRC errors | All versions | Single-bit errors in compressed data cause entire frame to decode as noise | Use Verify mode; recover from backups |
| Tag count mismatch | Some tag editors | Tag item count in footer does not match actual items; causes parsers to fail | Use Mutagen's fix or manually correct in hex editor |
| External cover art issue | All versions | Cover art marked as "external" with a file path becomes orphaned | Embed cover art instead |
| Long filename overflow | APEv1 | Very long tag keys could overflow | Use APEv2 |

### 14.2 Tag Corruption Issues

APE tagging is notoriously fragile compared to other formats. Known issues:
- **Mutagen library issue:** Early versions would throw exceptions when tag count in footer exceeded actual items
- **PhotoWatermark corruption:** Some taggers incorrectly parse binary items as text
- **Unicode in keys:** Some readers fail to handle Unicode-encoded tag keys

**Best practices:**
1. Always use the official Monkey's Audio Tag Editor or foobar2000 for tagging APE files
2. Keep backups before re-tagging
3. Use the Verify function in Monkey's Audio to check file integrity after any operation
4. Avoid third-party tools that directly modify the APE tag structure

### 14.3 Interoperability Issues
- **FFmpeg → reference decoder:** FFmpeg produces valid but suboptimal APE frames; reference decoder handles them correctly
- **Reference encoder → FFmpeg:** FFmpeg decodes correctly
- **Corrupted seek table → FFmpeg:** FFmpeg ignores seek table and does linear scan; playback works but seeking is slow
- **Multi-channel → APE:** Refuse; APE only supports mono and stereo
- **APEv2 tag with 0 items:** Some parsers fail; should have at least a footer

### 14.4 Edge Cases to Handle in Converter
- **Empty file / 0-sample stream:** Refuse to encode
- **File < 1 frame:** Handled normally
- **All-silence audio:** Encodes very efficiently; large blocks of zero residuals
- **Corrupt frame:** Replace with silence; continue decoding
- **Truncated file:** Decode available frames; report truncated warning
- **Seek table corruption:** Ignore seek table; perform linear scan seeking
- **Invalid tag keys:** Skip invalid keys; log warning
- **APE tag at beginning (non-standard):** Some files have APEv2 at start; handle both start and end

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM APE

| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|---------------------|----------------|
| → FLAC | `ffmpeg -i in.ape -c:a flac -compression_level 8 out.flac` | All tags via rewrite | Lossless |
| → WAV | `ffmpeg -i in.ape out.wav` | RIFF INFO (limited) | Lossless |
| → ALAC | `ffmpeg -i in.ape -c:a alac out.m4a` | Partial | Lossless if source is lossless |
| → MP3 | `ffmpeg -i in.ape -c:a libmp3lame -q:a 0 out.mp3` | Tags via ID3v2.3 | Generation loss |
| → AAC | `ffmpeg -i in.ape -c:a aac -b:a 256k out.m4a` | Partial | Generation loss |
| → Opus | `ffmpeg -i in.ape -c:a libopus -b:a 128k out.opus` | Vorbis Comments | Generation loss |
| → WavPack | `ffmpeg -i in.ape -c:a wavpack -compression_level 8 out.wv` | APEv2 → APEv2 | Lossless |
| → OGG Vorbis | `ffmpeg -i in.ape -c:a libvorbis -q:a 6 out.ogg` | Vorbis Comments | Generation loss |

### 15.2 Converting TO APE

**CRITICAL:** FFmpeg cannot encode APE. You must use the reference Monkey's Audio CLI/GUI.

```bash
# Using reference encoder (NOT FFmpeg):
# Install Monkey's Audio from https://monkeysaudio.com/

# Fast encoding
MAC in.wav out.ape 1000

# Normal encoding
MAC in.wav out.ape 2000

# High encoding
MAC in.wav out.ape 3000

# Extra High encoding
MAC in.wav out.ape 4000

# Insane encoding (slowest, best compression)
MAC in.wav out.ape 5000

# Decode with FFmpeg
ffmpeg -i in.ape -c:a pcm_s16le out.wav
```

### 15.3 Lossless Round-Trip Verification
```bash
# Encode with reference (requires Monkey's Audio installed)
MAC original.wav original.ape 4000

# Decode with FFmpeg
ffmpeg -i original.ape -c:a pcm_s16le decoded.wav

# Compare checksums
ffmpeg -i original.wav -c:a pcm_s16le original_raw.wav
md5sum original_raw.wav decoded.wav   # Must match for true lossless

# Using Monkey's Audio verification
MAC original.wav original.ape 4000 -v  # Verify mode
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Encoder | Decoder | URL |
|---------|----------|---------|---------|---------|-----|
| Monkey's Audio | C++ | Proprietary | Reference | Reference | https://monkeysaudio.com/ |
| FFmpeg native | C | LGPL 2.1+ | None | Good | https://ffmpeg.org |
| libcue | C | GPL | Via ref | Via ref | https://github.com/lipnitsk/libcue |
| APEv2 (mutagen) | Python | GPL | Via ref | Via ref | https://github.com/quodlibet/mutagen |

### Build Instructions
```bash
# Monkey's Audio is not open source for encoding.
# The official binaries are freeware but source is not available.

# FFmpeg already includes APE decoding:
# Compile FFmpeg with APE support:
./configure --enable-decoder=ape
make -j$(nproc)

# Verify decoder is present:
ffmpeg -decoders | grep ape
```

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **Monkey's Audio Theory:** https://www.monkeysaudio.com/theory.html (official, sparse)
- **APE Format:** No official specification; reverse-engineered by FFmpeg and Mutagen developers

### Technical Resources
- FFmpeg APE decoder: `libavcodec/apedec.c`
- FFmpeg APE demuxer: `libavformat/ape.c`
- Multimedia Wiki: https://wiki.multimedia.cx/index.php/Monkey%27s_Audio
- Hydrogenaudio Wiki: https://wiki.hydrogenaudio.org/index.php?title=Monkey%27s_Audio
- Mutagen APEv2 spec: https://mutagen-specs.readthedocs.io/en/latest/apev2/apev2.html

### Academic Papers
- No formal academic papers published on APE (proprietary algorithm)
- Compression theory applies: see Rice coding, arithmetic coding references from FLAC document

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] APE encoding requires Monkey's Audio CLI (not open-source)
- [ ] FFmpeg `ffmpeg -decoders | grep ape` confirms decode is available
- [ ] FFmpeg `ffmpeg -encoders | grep ape` confirms encode is NOT available
- [ ] For APE encoding: bundle reference `MAC.exe` (Windows) or macOS/Linux binary

### Encoding Pipeline
- [ ] FFmpeg cannot encode APE — invoke reference `MAC` CLI for encoding
- [ ] Map source metadata to APEv2 tag format
- [ ] Handle mono and stereo input only (refuse multi-channel)
- [ ] For high-resolution: verify support for 24-bit input
- [ ] Compression level mapping:
  - 0 (fast) → MAC parameter 1000
  - 5 (high) → MAC parameter 3000
  - 10 (max) → MAC parameter 5000

### Decoding Pipeline
- [ ] Implement frame CRC verification
- [ ] Handle CRC failure: mute frame, continue decoding
- [ ] Handle seek table corruption: fall back to linear scan
- [ ] Flush decoder at EOF (send NULL packet)
- [ ] Skip encoder-delay samples: block-aligned (first frame)
- [ ] Handle variable frame sizes between compression levels

### Metadata
- [ ] Read APEv2 tags from end of file
- [ ] Handle both APEv1 (old files) and APEv2 (modern files)
- [ ] Map APE tag keys to standard keys
- [ ] Read cover art as binary JPEG/PNG from APEv2
- [ ] Write APEv2 tags (requires external tool or direct binary writing)
- [ ] Handle tag item count mismatch (known Mutagen issue)
- [ ] Preserve ReplayGain tags through conversion
- [ ] Handle UTF-8 encoding in tag keys and values

### Quality & Verification
- [ ] Implement Verify mode (via reference MAC -v command)
- [ ] Provide bit-exact verification for lossless conversions
- [ ] Implement error detection: CRC per frame
- [ ] Implement partial-file recovery: decode available frames
- [ ] Test with: silence, full-scale sine, stereo content, corrupted frames

### Edge Cases
- [ ] Handle files with corrupt or missing file headers
- [ ] Handle files with 0 samples
- [ ] Handle channel count ≠ 1 or 2: refuse to encode
- [ ] Handle tag at beginning of file (non-standard but possible)
- [ ] Handle APEv2 tag with zero items
- [ ] Handle seek table overflow (too many frames)
- [ ] Handle truncated files: decode available frames

---

## APPENDIX A: FFmpeg APEDEC.C INTERNALS

### A.1 Compression Level to Filter/Frame Mapping
```c
enum APECompressionLevel {
    COMPRESSION_LEVEL_FAST       = 1000,  // frame=4608, filter order=1
    COMPRESSION_LEVEL_NORMAL     = 2000,  // frame=4096, filter order=2
    COMPRESSION_LEVEL_HIGH       = 3000,  // frame=3072, filter order=3
    COMPRESSION_LEVEL_EXTRA_HIGH = 4000,  // frame=2304, filter order=4+
    COMPRESSION_LEVEL_INSANE     = 5000   // frame=1280, filter order=5+
};
```

### A.2 FFmpeg APE Version Handling
```c
// APE versions:
enum {
    APE_VERSION_3_96 = 3960,  // Old format
    APE_VERSION_3_97 = 3970,  // New format
    APE_VERSION_3_98 = 3980,  // Further improvements
    APE_VERSION_4_0  = 4000   // APEv2 tag support
};
```

### A.3 Filter Orders by Compression Level
```c
// Filter orders for each compression level (per channel)
static const int8_t filter_orders[APE_FILTER_LEVELS] = {
    [COMPRESSION_LEVEL_FAST]       = 1,
    [COMPRESSION_LEVEL_NORMAL]     = 2,
    [COMPRESSION_LEVEL_HIGH]       = 3,
    [COMPRESSION_LEVEL_EXTRA_HIGH] = 4,
    [COMPRESSION_LEVEL_INSANE]     = 5
};
```

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
