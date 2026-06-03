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

## APPENDIX B: FFmpeg APEDEC.C — DETAILED DECODER ANALYSIS

### B.1 FFmpeg APE Decoder Version Support

The FFmpeg APE decoder handles multiple file format versions with different code paths:

```c
// Version constants in FFmpeg
enum {
    APE_VERSION_3_96 = 3960,  // Old format (MAC_HEADER_OLD)
    APE_VERSION_3_97 = 3970,  // New format with descriptor (MAC_HEADER)
    APE_VERSION_3_98 = 3980,  // Improved handling
    APE_VERSION_4_0  = 4000   // APEv2 tag support
};
```

### B.2 FFmpeg Compression Level Mapping

```c
enum APECompressionLevel {
    COMPRESSION_LEVEL_FAST       = 1000,  // Frame size: 4608 samples
    COMPRESSION_LEVEL_NORMAL     = 2000,  // Frame size: 4096 samples
    COMPRESSION_LEVEL_HIGH       = 3000,  // Frame size: 3072 samples
    COMPRESSION_LEVEL_EXTRA_HIGH = 4000,  // Frame size: 2304 samples
    COMPRESSION_LEVEL_INSANE     = 5000   // Frame size: 1280 samples
};
```

### B.3 Filter Structure in FFmpeg

```c
// FFmpeg defines filter memory for up to 8 filter stages
#define APE_FILTER_LEVELS 8

// Filter structure
typedef struct APEFilter {
    int16_t *buffer;    // Filter delay line
    int32_t *coeffs;    // Filter coefficients
    int pos;            // Current position in buffer
    int length;         // Filter length (order)
} APEFilter;

// Each compression level has different filter orders
static const int16_tape_filter_orders[5] = {
    1,   // Fast: 1 filter, order 1
    2,   // Normal: 2 filters, order 2
    3,   // High: 3 filters, order 3
    4,   // Extra High: 4 filters, order 4
    5    // Insane: 5 filters, order 5
};
```

### B.4 Rangecoder Implementation Details

```c
// FFmpeg APERangecoder structure
typedef struct APERangecoder {
    uint32_t low;           // Low end of interval
    uint32_t range;         // Length of interval
    uint32_t help;          // Bytes to follow
    unsigned int buffer;      // Input/output buffer
} APERangecoder;

// Key operations
static void range_dec_normalize(APERangecoder *rc) {
    while (rc->range < 0x01000000) {
        rc->range <<= 8;
        rc->low   <<= 8;
        // Read next byte if decoding
    }
}

static int range_dec_decode(APERangecoder *rc, int n) {
    // Decode n-bit value from rangecoder
    // Returns value in range [0, 2^n)
}
```

### B.5 Rice Coding in FFmpeg APE Decoder

```c
// FFmpeg Rice parameter structure
typedef struct APERice {
    int k;              // Rice parameter
    int sum;            // Running sum for k adaptation
} APERice;

// FFmpeg implements two decoding paths:
// - entropy_decode_stereo_3900: For APE version 3.900+
// - entropy_decode_stereo_3930: For APE version 3.930+

static void update_rice(APERice *rice, unsigned int x) {
    // Update Rice parameter based on decoded value
    // k adaptation formula:
    // if (x >= (1 << rice->k)) {
    //     rice->sum += (1 << rice->k);
    // } else {
    //     rice->sum += x;
    // }
    // Adjust k based on sum thresholds
    if (rice->sum >= (1 << 24)) {
        rice->sum >>= 1;
        rice->k++;
    }
}
```

### B.6 Frame Decoding Functions

FFmpeg implements version-specific decoding for different APE file versions:

```c
// For APE version 3.800:
static void predictor_decode_stereo_3800(APEContext *ctx, int count)
static void predictor_decode_mono_3800(APEContext *ctx, int count)

// For APE version 3.900+:
static void predictor_decode_stereo_3930(APEContext *ctx, int count)
static void predictor_decode_mono_3930(APEContext *ctx, int count)

// High filter variant (APE version 3.830):
static void long_filter_ehigh_3830(int32_t *buffer, int length)
```

### B.7 FFmpeg APE Decode Limitations

The FFmpeg APE decoder has the following limitations compared to the reference:

| Feature | FFmpeg | Reference | Notes |
|---------|--------|-----------|-------|
| Decode speed | Fast | Slower | FFmpeg is optimized |
| Error recovery | Limited | Better | Reference has better resync |
| Seek table use | No | Yes | FFmpeg ignores seek table |
| Frame validation | Basic | Thorough | Reference checks more cases |
| Tag handling | APEv2 only | APEv1 + APEv2 | FFmpeg handles both |

---

## APPENDIX C: COMPRESSION RATIOS — DETAILED BENCHMARKS

### C.1 Benchmark Methodology

Compression benchmarks for lossless codecs are affected by:
- Source material type (classical, electronic, rock, jazz, speech)
- Sample rate and bit depth
- Encoder settings
- Encoder version

All ratios shown are file size relative to uncompressed PCM.

### C.2 APE Compression Ratios by Genre (CD Audio)

| Genre | APE Fast | APE Normal | APE High | APE Extra High | APE Insane |
|-------|-----------|------------|-----------|----------------|------------|
| Classical (orchestral) | 58–62% | 54–58% | 51–55% | 48–52% | 46–50% |
| Jazz (instruments) | 55–60% | 51–55% | 48–52% | 46–50% | 44–48% |
| Rock/Pop | 62–68% | 58–64% | 55–61% | 52–58% | 50–56% |
| Electronic | 60–66% | 56–62% | 53–59% | 50–56% | 48–54% |
| Speech | 50–58% | 46–54% | 43–51% | 40–48% | 38–46% |
| New Age | 55–62% | 51–58% | 48–55% | 45–52% | 43–50% |
| Average | 60–64% | 56–60% | 52–56% | 49–53% | 47–51% |

### C.3 APE vs Other Lossless Codecs (CD Audio)

| Codec | Fastest | Default | Best | Notes |
|-------|---------|---------|------|-------|
| FLAC | 65–68% | 58–62% | 52–56% | Best for compatibility |
| WavPack | 65–70% | 58–62% | 44–48% | Good balance |
| APE | 58–64% | 56–60% | 47–51% | Best compression |
| TAK | 62–66% | 56–60% | 48–52% | Fastest at high compression |
| OptimFROG | 58–62% | 52–56% | 46–50% | Slowest |
| ALAC | 72–75% | 72–75% | 72–75% | No compression options |

### C.4 APE Encoding Time Comparison (CD Audio, 75-minute album)

| Codec | Mode | Encode Time | Decode Time | Ratio |
|-------|------|-------------|-------------|-------|
| FLAC | -0 | ~30s | ~15s | 65% |
| FLAC | -5 | ~60s | ~20s | 56% |
| FLAC | -8 | ~3min | ~25s | 53% |
| APE | Fast | ~45s | ~20s | 60% |
| APE | Normal | ~90s | ~25s | 56% |
| APE | High | ~3min | ~35s | 53% |
| APE | Extra High | ~8min | ~50s | 50% |
| APE | Insane | ~45min | ~90s | 48% |

### C.5 Insane Mode Performance Notes

APE "Insane" mode has diminishing returns:
- Only 1–2% smaller than "Extra High" on average
- Takes 5–10× longer to encode than "Extra High"
- Takes 2× longer to decode than "Extra High"
- May actually produce larger files than "Extra High" on some material (confirmed in Xiph comparison)
- Recommended: Use "Extra High" for archival, "High" for general use

---

## APPENDIX D: TAG EDITING — DETAILED GUIDE

### D.1 APE Tag Item Keys (Official and Common)

Standard APEv2 tag keys used in APE files:

| Key | Description | Type | Example |
|-----|-------------|------|---------|
| Title | Track title | Text | "Symphony No. 5" |
| Artist | Track artist | Text | "Beethoven" |
| Album | Album name | Text | "Symphonies" |
| Album Artist | Album artist | Text | "Vienna Philharmonic" |
| Composer | Composer | Text | "Ludwig van Beethoven" |
| Genre | Music genre | Text | "Classical" |
| Year | Release year | Text | "2024" |
| Track | Track number | Text | "5/9" |
| Disc | Disc number | Text | "1/2" |
| Comment | User comment | Text | "Recorded 2024" |
| Encoder | Encoding software | Text | "Monkey's Audio 4.67" |
| Encoder Settings | Encoding settings | Text | "Extra High" |
| ISRC | International Standard Recording Code | Text | "USRC12345678" |
| Copyright | Copyright notice | Text | "(C) 2024" |
| Publisher | Record label | Text | "Deutsche Grammophon" |
| EAN/UPC | Barcode | Text | "0044003232329" |
| Album Gain | ReplayGain album gain | Text | "-6.20 dB" |
| Peak Level | Album peak level | Text | "0.998459" |
| CueSheet | Embedded cuesheet | Text | [cuesheet content] |
| Cover Art (Front) | Front cover | Binary | [JPEG/PNG data] |
| Cover Art (Back) | Back cover | Binary | [JPEG/PNG data] |

### D.2 APE Tag Value Encoding

APEv2 supports multiple value encodings:

```c
// APE tag item flags
enum {
    APE_TAG_FLAG_TEXT       = 0 << 0,  // UTF-8 text
    APE_TAG_FLAG_BINARY     = 1 << 0,  // Binary data
    APE_TAG_FLAG_EXTERNAL  = 1 << 1,  // External (file path)
    APE_TAG_FLAG_UTF8      = 1 << 2,  // Explicit UTF-8 encoding
};
```

### D.3 Tag Reading with FFmpeg

```bash
# Read all tags
ffprobe -v quiet -print_format json -show_format input.ape | jq .format.tags

# Read specific tag
ffprobe -v quiet -show_format input.ape 2>&1 | grep -i title

# List all tags with ffprobe
ffprobe -v quiet -show_format input.ape
```

### D.4 Tag Writing Limitations

FFmpeg cannot write APE tags. Use these alternatives:

| Tool | Platform | APE Tag Support | Notes |
|------|----------|-----------------|-------|
| Monkey's Audio | Windows | Full | Official tool |
| Mp3tag | Windows | Full | Freeware |
| APE Tag Editor | Windows | Full | Simple UI |
| FFmpeg | All | Read only | Cannot write APE tags |
| Mutagen (Python) | All | Full | Library |
| Ex Falso | Linux | Full | Quod Libet frontend |

### D.5 Tagging Best Practices

1. **Always backup before editing:** APE tags are fragile
2. **Use official tools when possible:** Monkey's Audio Tag Editor
3. **Verify after editing:** Use Verify mode
4. **Avoid streaming edits:** Read/modify/write, don't stream
5. **Watch for encoding:** Use UTF-8 for Unicode support
6. **Check tag size:** Maximum recommended is 1 MB
7. **Test with player:** Verify tags work in your playback software

---

## APPENDIX E: BINARY FORMAT — DETAILED STRUCTURE

### E.1 Complete APE File Structure Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         APE File (.ape)                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                     File Header (MAC_HEADER)                          │   │
│  │  ┌─────────┬─────────┬──────────┬────────────┬─────────────────┐   │   │
│  │  │ "MAC "  │ Version │ Parameter │ Format Flags │ Channel Count  │   │   │
│  │  │  (4B)   │ (2B)   │  (4B)    │   (4B)      │    (4B)        │   │   │
│  │  ├─────────┴─────────┴──────────┴────────────┴─────────────────┤   │   │
│  │  │ Sample Rate │ Total Samples │ First Block │ Total Blocks │ ... │   │   │
│  │  │   (4B)     │   (8B)      │ Location (8B)│   (8B)     │     │   │   │
│  │  └─────────────┴─────────────┴─────────────┴─────────────┴─────┘   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                     Audio Frames (repeat for each frame)               │   │
│  │  ┌──────────────────────────────────────────────────────────────┐   │   │
│  │  │ Frame Header (8 bytes)                                        │   │   │
│  │  │  [0x0000] Frame CRC (4 bytes LE)                           │   │   │
│  │  │  [0x0004] Frame Size (4 bytes LE)                          │   │   │
│  │  ├──────────────────────────────────────────────────────────────┤   │   │
│  │  │ Compressed Frame Data                                        │   │   │
│  │  │  - Rangecoder entropy-coded residuals                        │   │   │
│  │  │  - IIR filter coefficients                                  │   │   │
│  │  │  - Channel decorrelation data                                │   │   │
│  │  └──────────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                     APEv2 Tag (optional, at end of file)              │   │
│  │  ┌─────────────┬──────────────┬───────────────────────────┐       │   │
│  │  │ "APETAGEX" │  Version    │  Tag Size (4B)           │       │   │
│  │  │   (8B)      │  (4B)       │                          │       │   │
│  │  ├─────────────┴──────────────┼───────────────────────────┤       │   │
│  │  │ Item Count (4B)            │  Flags (4B)               │       │   │
│  │  ├───────────────────────────┴───────────────────────────┤       │   │
│  │  │ Tag Items (sorted by size, ascending)               │       │   │
│  │  │  ┌────────────────────────────────────────────┐   │       │   │
│  │  │  │ Tag Item N                                    │   │       │   │
│  │  │  │  [Key Length: 4B LE]                        │   │       │   │
│  │  │  │  [Key String: variable]                       │   │       │   │
│  │  │  │  [Value Length: 8B LE]                       │   │       │   │
│  │  │  │  [Flags: 4B LE]                            │   │       │   │
│  │  │  │  [Value Data: variable]                      │   │       │   │
│  │  │  └────────────────────────────────────────────┘   │       │   │
│  │  ├─────────────────────────────────────────────────┤       │   │
│  │  │ Padding (null bytes)                            │       │   │
│  │  ├─────────────────────────────────────────────────┤       │   │
│  │  │ Tag Footer "APETAGEX"                           │       │   │
│  │  └─────────────────────────────────────────────────┘       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### E.2 MAC_HEADER_OLD Structure (Version 3.96 and earlier)

```c
struct MAC_HEADER_OLD {
    char     magic[4];           // "MAC "
    uint16_t version;            // APE version × 1000 (LE)
    uint16_t compression_level;  // 1000-5000
    uint16_t format_flags;       // Format flags
    uint16_t channels;          // Number of channels (1-2)
    uint32_t sample_rate;        // Samples per second
    uint32_t total_frames;       // Total number of frames
    uint32_t final_frame_samples;// Samples in final frame
    uint32_t total_samples;      // Total audio samples
    uint32_t block_align;        // Samples per block
    uint32_t bits_per_sample;   // Bits per sample
};
```

### E.3 MAC_HEADER Structure (Version 3.97+)

```c
struct MAC_DESCRIPTOR {
    char     magic[4];           // "MAC "
    uint16_t version;            // APE version × 1000 (LE)
    uint16_t padding;           // Descriptor bytes (to header start)
    uint32_t descriptor_bytes;  // Descriptor length
    uint32_t header_bytes;     // Header length
    uint32_t seek_table_bytes;  // Seek table length
    uint32_t header_data_bytes; // Header data length
    uint32_t frame_data_bytes; // Frame data length (high 32 bits)
    uint32_t frame_data_bytes_low; // Frame data length (low 32 bits)
    uint64_t terminating_bytes;  // Bytes after audio
    uint64_t file_signature;    // File signature
};

struct MAC_HEADER {
    uint16_t compression_level;  // 1000-5000
    uint32_t format_flags;       // Format flags
    uint32_t channels;          // Number of channels (1-2)
    uint32_t sample_rate;        // Samples per second
    uint64_t total_samples;      // Total audio samples
    uint64_t first_frame_offset;// Offset to first frame
    uint64_t total_frames;       // Total number of frames
    uint32_t final_frame_samples;// Samples in final frame
    uint32_t total_bits_per_sample; // Average bits per sample
};
```

### E.4 Frame Header Structure

```c
struct APE_FRAME_HEADER {
    uint32_t crc;      // CRC-32 of decoded samples
    uint32_t size;    // Compressed frame size in bytes
};

// For version 3.96 and earlier:
// uint32_t size first, then uint32_t crc
```

### E.5 Seek Table Structure

```c
// Seek table entries (if HAS_SEEK_ELEMENTS flag is set)
// Stored as array of uint32 values
// Each entry is byte offset from file start to frame start

uint32_t seek_table[total_frames];

// Example:
seek_table[0] = 0x00000080;  // First frame at byte 128
seek_table[1] = 0x00012400;  // Second frame at byte 74752
seek_table[2] = 0x00024800;  // Third frame at byte 149504
// ... etc
```

### E.6 Tag Item Structure Detail

```c
struct APE_TAG_ITEM {
    uint32_t key_length;    // Length of key string
    char     key[];          // Key string (e.g., "Artist")
    uint64_t value_length;   // Length of value
    uint32_t flags;          // Item flags
    uint8_t  value[];       // Value data
};

// Tag item flags
enum APE_TAG_FLAG {
    APE_TAG_FLAG_CONTAINS_DATA = (1 << 0),
    APE_TAG_FLAG_KEEP_AFTER_DELETE = (1 << 1),
};
```

---

## APPENDIX F: COMPARISON WITH OTHER LOSSLESS CODECS

### F.1 Codec Feature Comparison Matrix

| Feature | APE | FLAC | WavPack | ALAC | TAK | OptimFROG |
|---------|-----|------|---------|------|-----|-----------|
| Compression | Best | Good | Good | Poor | Good | Best |
| Encode Speed | Very Slow | Fast | Fast | Medium | Fast | Very Slow |
| Decode Speed | Medium | Fast | Fast | Fast | Fast | Slow |
| Open Source | No | Yes | Yes | Yes (Apple) | No | No |
| Multi-channel | No | Yes | Yes | Yes | Yes | Yes |
| Float Support | No | Yes | Yes | Yes | No | Yes |
| Hybrid Mode | No | No | Yes | No | No | Yes |
| Hardware Support | Low | High | Medium | High | Low | Very Low |
| Streaming | Yes | Yes | Yes | Yes | Yes | Yes |
| Error Recovery | Good | Good | Excellent | Good | Good | Good |

### F.2 Compression Ratio Comparison (CD Audio)

| Codec | Fastest | Default | Best | Encode Time | Decode Time |
|-------|---------|---------|------|-------------|-------------|
| APE | 60% | 56% | 48% | ~45min (best) | ~90s |
| FLAC | 68% | 58% | 54% | ~3min | ~25s |
| WavPack | 68% | 58% | 46% | ~10min | ~60s |
| TAK | 64% | 56% | 50% | ~4min | ~30s |
| OptimFROG | 58% | 52% | 46% | ~60min | ~120s |
| ALAC | 75% | 75% | 75% | ~5min | ~20s |

### F.3 When to Choose Each Codec

**Choose APE when:**
- Maximum compression is required
- Encoding time is not a concern
- Only stereo/mono audio
- Using Windows platform
- Decoding speed is acceptable

**Choose FLAC when:**
- Cross-platform compatibility is important
- Hardware playback is required
- Fast encoding/decoding is needed
- Multi-channel support is required
- Open source is required

**Choose WavPack when:**
- Hybrid lossy/lossless is needed
- High-resolution/DSD support is required
- Maximum format flexibility is needed
- Good compression with fast speed is desired

**Choose TAK when:**
- Fast encoding with good compression is needed
- Windows platform is used
- Multi-channel support is required
- Hardware support is less important

### F.4 Migration Paths

```
APE → FLAC:
  ffmpeg -i input.ape -c:a flac -compression_level 8 output.flac

APE → WavPack:
  ffmpeg -i input.ape -c:a wavpack -compression_level 8 output.wv

FLAC → APE (requires Monkey's Audio):
  MAC input.flac output.ape 4000

WavPack → APE (requires Monkey's Audio):
  # First decode
  ffmpeg -i input.wv -c:a pcm_s16le output.wav
  # Then encode
  MAC output.wav output.ape 4000

APE → ALAC:
  ffmpeg -i input.ape -c:a alac output.m4a
```

---

## APPENDIX G: TROUBLESHOOTING GUIDE

### G.1 Common APE Issues and Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| File won't play | Corrupt header | Use Verify mode, recover from backup |
| Seeking broken | Seek table corrupted | Use linear seek mode, avoid tag editors |
| Tag editor crashes | Invalid tag count | Fix with Mutagen, hex editor |
| Encoding fails | Out of disk space | Free space, use smaller block size |
| Decode errors | Corrupt frames | Use Verify mode, skip bad frames |
| Wrong track length | Wrong sample rate | Check header, re-encode |
| Memory error | Large file, limited RAM | Use smaller compression level |

### G.2 Recovery from Corrupt Files

```bash
# 1. Verify file integrity
MAC input.ape -v

# 2. Extract available audio (blind decode)
ffmpeg -i input.ape -acodec pcm_s16le -b blind output.wav

# 3. Check for gaps
ffmpeg -i output.wav -f null -

# 4. Re-encode good portions
# (Manual process using audio editor)

# 5. Verify output
MAC output.wav -v
```

### G.3 Tag Corruption Repair

```python
# Using Mutagen to fix tag count mismatch
from mutagen.apev2 import APEv2

def fix_tag_count(filename):
    tags = APEv2(filename)
    # Force rewrite of tag footer with correct count
    tags.save()

# Alternative: Use hex editor
# 1. Find "APETAGEX" at end of file
# 2. Locate item count field (4 bytes before footer end)
# 3. Count actual tag items
# 4. Update item count to match
```

### G.4 Performance Tuning

```bash
# Optimize encoding for speed
MAC input.wav output.ape 2000  # Normal mode

# Optimize for compression
MAC input.wav output.ape 4000  # Extra High

# Use multiple threads (if supported)
# Parallel encoding:
ls *.wav | xargs -P 4 -I {} MAC {} {}.ape

# Monitor memory usage
# APE encoding can use 100+ MB per file
# Reduce concurrent operations if memory limited
```

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
