# Windows Media Audio Lossless (WMA Lossless) — Deep Technical Reference
> **Category:** Lossless
> **File Extensions:** `.wma`, `.asf`
> **MIME Types:** `audio/x-wma-lossless`, `audio/wma`, `audio/x-ms-wma`
> **Standardization Body:** Microsoft
> **Primary Specification:** Proprietary (not publicly documented)
> **Patent Status:** Patented — expires [varies by patent; Microsoft holds core patents]
> **License:** Proprietary (royalty-bearing; licensing via Microsoft)
> **Current Version:** WMA 9 Lossless (also known as WMAL)
> **Active Development:** No — last release ~2003–2005, deprecated

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Microsoft Corporation
- **Year Created:** 2003 (Windows Media 9 Series)
- **Original Purpose:** Provide a lossless audio compression codec for high-fidelity archival and playback, competing with FLAC, ALAC, and DVD-Audio/SACD formats
- **Problem with Predecessors:** WMA Standard was lossy; no native Windows lossless audio option; users wanted bit-perfect archival within the Windows Media ecosystem

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| WMA 9 Lossless | 2003 | Initial release; up to 24-bit/96 kHz, 5.1 surround |
| WMA 9.1 Lossless | 2004 | Improved compression efficiency |
| WMA 9.2 Lossless | 2005 | Minor improvements; bug fixes |
| Extended Lossless | ~2005+ | Extended to higher sample rates [NEEDS VERIFICATION] |

### 1.3 Current Adoption
- **Primary use cases today:** High-fidelity archival within Windows ecosystem, legacy collections, audiophile archives
- **Platforms with native support:** Windows (via Windows Media Player, Media Foundation), macOS (Windows Media Player for Mac OS X, discontinued), Linux (via FFmpeg)
- **Major services using this format:** Historically: Napster (original lossless offering), Yahoo! Music. As of 2024: Qobuz offers WMA Lossless downloads; very limited adoption otherwise
- **Hardware support:** Some high-end AV receivers and Blu-ray players from early-to-mid 2010s; declining in favor of FLAC/ALAC
- **Status:** Declining/Legacy — no new encoding tools from Microsoft; Qobuz remains the primary commercial user

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Predictive lossless codec (integer-based)
- **Core algorithm:** Integer-based LMS (Least Mean Square) prediction + linear prediction + arithmetic coding
- **Loss mechanism:** None (bit-perfect reconstruction guaranteed by design)
- **Frame-based vs sample-based:** Frame/tile-based; each tile processed independently
- **Fixed vs variable frame size:** Variable; tiles have configurable size (typically 2048–8192 samples)

### 2.2 High-Level Encoding Flow (Block Diagram)
```
Input PCM Samples (up to 8 channels, 24-bit, 96 kHz)
      │
      ▼
[Pre-processing: Integer conversion, channel reordering]
      │
      ▼
[Tile/Block Segmentation: Divide into independent tiles]
      │
      ▼
[Inter-Channel Decorrelation: M/S or channel prediction]
      │
      ▼
[LMS Adaptive Filtering: Per-channel LMS predictors]
      │
      ▼
[LPC Linear Prediction: Higher-order linear prediction]
      │
      ▼
[Residue Encoding: Arithmetic coding of residuals]
      │
      ▼
[Bitstream Packing: Tiles, coefficients, parameters]
      │
      ▼
Output WMA Lossless Bitstream (within ASF container)
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | Variable: tile-dependent | No lookahead required for lossless |
| Tile size | 2048–16384 samples | Configurable per stream |
| Max channels | 8 (7.1 surround) | Mono through 7.1 |
| Max bit depth | 24-bit | Integer only |
| Max sample rate | 96000 Hz | Standard version; extended may support higher |
| Compression ratio | 1.5–2.5× (30–50% reduction) | Content-dependent |
| Complexity | O(N × predictor_order) | Encode heavier than decode |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
WMA Lossless is not a standalone file format; it is always contained within ASF (Advanced Systems Format). The audio stream is identified by the codec FourCC.

```
WMA Lossless (ASF Audio Stream):
  FourCC: 0x0163 (WMA Lossless)
```

### 3.2 ASF Container Structure (Relevant Fields)
```
ASF File Header (GUID: 8CABDCA1-A947-11CF-8EE4-00C00C205365):
  ├── Object Size (uint64 LE)
  ├── Number of Header Objects (uint32)
  └── Header Objects[]
       ├── File Properties Object
       │    ├── File ID (GUID)
       │    ├── File Size (uint64)
       │    ├── Creation Date (uint64)
       │    ├── Data Packets Count (uint64)
       │    └── Play Duration (uint64)
       ├── Stream Properties Object (audio stream)
       │    ├── Stream Type: {AUDIO_OBJECT}
       │    ├── Error Correction Type
       │    ├── Time Offset (uint64)
       │    ├── Type-Specific Data Length (uint32)
       │    └── Type-Specific Data (WMA Lossless WAVEFORMATEX)
       │         ├── wFormatTag: 0x0163 (WMA Lossless)
       │         ├── nChannels: 1–8
       │         ├── nSamplesPerSec: 8000–96000
       │         ├── nAvgBytesPerSec: variable (lower than PCM)
       │         ├── nBlockAlign: varies
       │         ├── wBitsPerSample: 16 or 24
       │         ├── cbSize: variable
       │         └── Extra Data (WMA Lossless codec initialization)
       ├── Content Description Object (metadata)
       └── other optional objects...
```

### 3.3 WMA Lossless Bitstream Structure
```
WMA Lossless Frame (within ASF Packet):
  ├── Frame Header
  │    ├── Frame size (variable)
  │    ├── Samples in frame (from extradata)
  │    └── Configuration flags
  ├── Tile Header
  │    ├── Tile size
  │    ├── Channel configuration
  │    └── Prediction mode flags
  ├── Filter Coefficients
  │    ├── LMS filter orders and coefficients
  │    ├── LPC filter orders and coefficients
  │    └── AC filter coefficients
  └── Encoded Residuals
       ├── Arithmetic-coded integer residuals
       └── Channel residues
```

### 3.4 Arithmetic Coder Specification
WMA Lossless uses adaptive arithmetic coding for efficient residual encoding.

```
Arithmetic Coder Parameters:
  Probability Model:  Adaptive (updates after each symbol)
  Precision:         16-bit internal (carry-free)
  Escape Handling:   Special escape sequences for rare symbols
  
Key Operations:
  1. Range encoding:
     - Maintain low and high probability bounds
     - Output bits as range narrows
     
  2. Probability update:
     - Increment count for decoded symbol
     - Renormalize probability model
     
  3. Escape handling:
     - When probability < threshold:
       - Output escape code
       - Encode raw binary value
```

### 3.5 LMS Filter Coefficient Encoding
```
LMS Coefficient Encoding:
  1. Differential encoding: Transmit differences from previous tile
  2. Huffman coding: Use short codes for common differences
  3. Precision: Fixed-point representation (Q10 format)
  
Coefficient Update Rules:
  - Adaptation speed: Variable per tile
  - Update direction: Sign-sign LMS (simplest)
  - Coefficient bounding: Clamp to prevent overflow
```

### 3.4 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Unsigned integer | No | Not supported |
| 16-bit | Signed integer | Yes | Primary format |
| 20-bit | Signed integer | Yes | Common in professional audio |
| 24-bit | Signed integer | Yes | Primary high-resolution format |
| 32-bit | Signed integer | No | Not supported |
| 32-bit | IEEE float | No | Not supported |
| 64-bit | IEEE float (double) | No | Not supported |

### 3.4.1 Supported Sample Rates
| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 8000 | Telephone | Yes | Voice mode |
| 11025 | — | Yes | |
| 16000 | Wideband voice | Yes | |
| 22050 | — | Yes | |
| 32000 | Broadcast | Yes | |
| 44100 | CD audio | Yes | Primary rate |
| 48000 | Professional | Yes | Standard high-res |
| 88200 | 2× CD | Yes | High-resolution |
| 96000 | High-res max | Yes | Maximum supported |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Pre-Processing Stage
- **Integer conversion:** Input PCM is converted to integer representation for all processing
- **DC offset handling:** Not applicable (lossless codec preserves all data including DC)
- **Channel reordering:** Rearrange input channels to standard order for encoding
- **No windowing:** No window function applied (predictive codecs don't use transforms)

### 4.2 Prediction / Transform Stage

#### Core Algorithm: Integer LMS + LPC Hybrid
```
WMA Lossless uses a multi-stage prediction approach:

Stage 1: Inter-channel prediction
  - M/S (Mid/Side) transformation for stereo
  - Or LMS-based inter-channel prediction for multichannel
  - Reduces correlation between channels

Stage 2: LMS (Least Mean Square) adaptive filtering
  - Per-channel adaptive predictor
  - LMS order: up to 512 coefficients [NEEDS VERIFICATION]
  - Adapts to local signal characteristics
  - Coefficients updated sample-by-sample

Stage 3: LPC (Linear Predictive Coding) residual encoding
  - Fixed-order linear prediction
  - LPC order: up to 128 [NEEDS VERIFICATION]
  - Encodes the remaining prediction error
  - Coefficients transmitted in bitstream

Stage 4: AC (Adaptive Coding) filter
  - Additional noise-shaping filter
  - Improves compression of residual signal
  - Optional per configuration
```

#### Mathematical Definition
```
LMS Prediction:
  p[n] = Σ(i=0 to order-1) w[i] × x[n-i]
  e[n] = x[n] - p[n]  (prediction error)
  w[i] = w[i] + μ × e[n] × x[n-i]  (weight update)
  where:
    p[n] = predicted sample
    e[n] = residual (to be encoded)
    w[i] = LMS coefficients
    μ = step size (learning rate)
```

```
LPC Prediction:
  p[n] = Σ(i=1 to P) a[i] × x[n-i]
  e[n] = x[n] - p[n]  (prediction error)
  where:
    a[i] = LPC coefficients (Levinson-Durbin solved)
    P = LPC order
    e[n] = residual to be encoded
```

### 4.3 Residual Encoding (Lossless Stage)

#### Arithmetic Coding
```
Method: Adaptive arithmetic coding (binary arithmetic coder)

Residual coding process:
1. Integer residuals from LMS/LPC prediction
2. Map residuals to binary representation
3. Arithmetic code each bit plane
4. Adaptive probability estimation

Key features:
  - Zero-frequency handling: Special codes for runs of zeros
  - Signed residuals: Encode sign separately
  - Escape codes: For large residuals exceeding table
  - Adaptive: Probabilities updated as encoding progresses
```

#### Filter Coefficient Coding
```
LMS Coefficients:
  - Transmitted as differences from previous tile
  - Huffman-coded for compact representation
  - DPCM encoding with adaptive quantization

LPC Coefficients:
  - Levinson-Durbin recursion solved on encoder side
  - Reflection coefficients encoded
  - PARCOR coefficients (partial correlation)

AC Filter Coefficients:
  - Transmitted as integer values
  - Fixed-point representation
```

### 4.4 Encoder Settings / Quality Modes

#### Compression Levels
| Compression Level | Encoding Speed | Compression Ratio | Notes |
|---|---|---|---|
| Fast | Very fast | Lower (~1.5×) | Simple LMS predictor |
| Normal | Medium | Medium (~1.8×) | Balanced |
| High | Slow | Higher (~2.2×) | Complex prediction |
| Maximum | Very slow | Highest (~2.5×) | Full algorithm |

#### Bitrate Examples (44.1 kHz Stereo)
| Bit Depth | PCM Bitrate | WMA Lossless Bitrate | Compression |
|----------|-------------|---------------------|------------|
| 16-bit | 1411 kbps | ~700–900 kbps | ~50% reduction |
| 24-bit | 2117 kbps | ~1000–1400 kbps | ~40% reduction |

#### Tile Size Configuration
```
Tile Size Options:
  Size 0:     2048 samples
  Size 1:     4096 samples (default)
  Size 2:     8192 samples
  Size 3:     16384 samples

Tile Size Selection Criteria:
  - Longer tiles: Better compression, higher latency
  - Shorter tiles: Lower compression, lower latency
  - Default: 4096 samples at 44.1/48 kHz
```

### 4.8 Prediction Order Specifications
```
LMS Filter Orders:
  Channel LMS Order:    16–512 (adaptive per tile)
  Inter-channel LMS:    4–32 coefficients

LPC Filter Orders:
  Per-channel LPC:      8–128 coefficients
  Levinson-Durbin recursion for coefficient computation
  Reflection coefficients encoded in bitstream

AC Filter:
  Order:               1–16
  Coefficients:         Integer (Q15 format)
  Application:         Noise shaping in residual domain
```

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. ASF container parsed for audio stream properties
2. WMA Lossless extradata extracted (codec initialization data)
3. Verify FourCC is 0x0163
4. Parse tile structure
5. For each tile:
   a. Decode filter coefficients
   b. Decode arithmetic-coded residuals
   c. Apply inverse prediction (LPC + LMS)
   d. Apply inter-channel reconstruction
   e. Output decoded samples
```

#### Seeking
- **ASF seeking:** Index-based seeking using ASF index object
- **Seek table:** Stored in optional Data Object Index
- **Precision:** Millisecond accuracy (tile boundaries)
- **Tile-based seeking:** Decoder can seek to tile boundaries efficiently

### 5.2 Core Decode Pipeline
```
1. Parse ASF packet structure
   ├── ASF packet header
   ├── Payload parse flags
   └── Payload data (WMA Lossless tile payload)

2. Decode tile header
   ├── Tile size and configuration
   ├── Channel setup
   └── Prediction mode flags

3. Decode filter coefficients
   ├── LMS coefficients (if enabled)
   ├── LPC coefficients (if enabled)
   └── AC filter coefficients (if enabled)

4. Decode residuals
   ├── Arithmetic decode bit planes
   ├── Reconstruct integer residuals
   └── Handle signed values

5. Inverse prediction
   ├── Apply LPC: x[n] = p[n] + e[n]
   ├── Apply LMS: refine prediction
   └── Apply AC filter (if enabled)

6. Inter-channel reconstruction
   ├── If M/S: reconstruct L, R from M, S
   ├── If inter-channel prediction: reconstruct dependent channels
   └── Channel remapping

7. Output formatting
   ├── Clip to valid range
   └── Format as PCM samples
```

### 5.3 Error Detection and Concealment
- **Corrupt tile detection:** Arithmetic decode errors, invalid coefficients
- **CRC checking:** Per-tile CRC for error detection [NEEDS VERIFICATION]
- **Concealment method:** 
  - Repeat last good tile
  - Interpolation for gaps
  - Mute after extended errors
- **Maximum consecutive errors:** Implementation-dependent

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** ASF (Advanced Systems Format)
- **Overhead:** ~1–3% (ASF header objects + packet headers)
- **Seeking in native container:** Yes — by index object (if present)
- **Multiple streams in native container:** Yes (audio + video + metadata)

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| ASF/WMA | Yes (native) | Yes | Full | WMA Lossless = ASF with WLAC stream |
| AVI | No | — | — | Not supported |
| Matroska/MKA | No | — | — | Not supported |
| MP4/M4A | No | — | — | Not supported |
| OGG | No | — | — | Not supported |
| WAV | No | — | — | Not supported |
| AIFF | No | — | — | Not supported |
| WebM | No | — | — | Not supported |

**Note:** WMA Lossless requires ASF container. There is no raw WMA Lossless stream format.

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** ASF Content Description Object + Extended Content Description Object
- **Tag block location:** Within ASF header object, before audio data
- **Tag block identifier:** GUID-based objects in ASF

### 7.2 Standard Tag Fields — Complete Reference
| Tag Field | Internal Key (ASF) | Max Length | Character Encoding | Multi-value | Notes |
|-----------|------------------------|------------|-------------------|-------------|-------|
| Title | Title | 1024 bytes | UTF-16LE | No | |
| Artist | Author | 1024 bytes | UTF-16LE | No | |
| Album | WM/AlbumTitle | 1024 bytes | UTF-16LE | No | |
| Album Artist | WM/AlbumArtist | 1024 bytes | UTF-16LE | No | |
| Composer | WM/Composer | 1024 bytes | UTF-16LE | No | |
| Genre | WM/Genre | 1024 bytes | UTF-16LE | No | |
| Year / Date | WM/Year | 4 bytes | ASCII | No | 4-digit year |
| Track Number | WM/TrackNumber | 4 bytes | ASCII | No | Integer |
| Disc Number | WM/PartOfSet | 4 bytes | ASCII | No | Integer |
| Comment | Description | 1024 bytes | UTF-16LE | No | |
| Lyrics | WM/Lyrics | 1024 bytes | UTF-16LE | No | |
| BPM | WM/BeatsPerMinute | 4 bytes | ASCII | No | Integer |
| Compilation | WM/Compilation | 1 byte | ASCII | No | 0 or 1 |
| Copyright | Copyright | 1024 bytes | UTF-16LE | No | |
| Publisher/Label | WM/Publisher | 1024 bytes | UTF-16LE | No | |
| Encoder | WM/EncodedBy | 1024 bytes | UTF-16LE | No | Software name |
| Encoder Settings | WM/EncodingSettings | 1024 bytes | UTF-16LE | No | Compression level |
| ISRC | WM/ISRC | 12 bytes | ASCII | No | |
| MusicBrainz Track ID | MusicBrainz/TrackId | 36 bytes | ASCII | No | UUID format |
| MusicBrainz Artist ID | MusicBrainz/ArtistId | 36 bytes | ASCII | No | UUID format |
| MusicBrainz Album ID | MusicBrainz/AlbumId | 36 bytes | ASCII | No | UUID format |
| ReplayGain Track Gain | REPLAYGAIN_TRACK_GAIN | 16 bytes | ASCII | No | Format: "-6.20 dB" |
| ReplayGain Track Peak | REPLAYGAIN_TRACK_PEAK | 16 bytes | ASCII | No | Format: "0.998459" |
| ReplayGain Album Gain | REPLAYGAIN_ALBUM_GAIN | 16 bytes | ASCII | No | Format: "-5.80 dB" |
| ReplayGain Album Peak | REPLAYGAIN_ALBUM_PEAK | 16 bytes | ASCII | No | Format: "0.998459" |
| Cover Art | WM/Picture | up to 10 MB | Binary | No | Binary blob with MIME header |
| Arbitrary/Custom | WM/Custom* | 1024 bytes | UTF-16LE | Yes | Custom namespace |

### 7.3 Cover Art Storage
```
Cover art storage format in ASF/WMA Lossless:
  Container type:  WM/Picture attribute (binary)
  Image formats:   JPEG (recommended), PNG
  Max image size:  10 MB (implementation limit)
  Max dimensions:  No hard limit (recommend 3000×3000)
  
  Binary layout of WM/Picture:
    [0x00]        Picture type (1 byte): 0=Other, 1=32x32, 2=Other, 3=Front cover, 4=Back cover
    [0x01-N]      MIME type (null-terminated UTF-16LE string)
    [N+1-M]       Description (null-terminated UTF-16LE string)
    [M+1-...]     Binary image data (JPEG/PNG)
```

### 7.4 Metadata Compatibility Matrix
| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| ASF Native | ✓ | ✓ | ✓ | Highest |
| ID3v1 | ✗ | ✗ | ✗ | N/A |
| ID3v2.3 | ✗ | ✗ | ✗ | N/A |
| APEv2 | ✗ | ✗ | ✗ | N/A |
| Vorbis Comments | ✗ | ✗ | ✗ | N/A |
| MP4 Atoms | ✗ | ✗ | ✗ | N/A |

**Conflict resolution:** Only ASF metadata is native; no conflicts.

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   wmalossless    # used with -c:a
AV_CODEC_ID:        AV_CODEC_ID_WMALOSSLEX
Format Name (CLI):  asf              # ASF container
Encoder(s):         NONE             # FFmpeg CANNOT encode WMA Lossless
Decoder(s):         wmalossless      # Native FFmpeg decoder
Muxer(s):           asf, asfw         # ASF muxer
Demuxer(s):         asf              # ASF demuxer
```

**CRITICAL:** FFmpeg supports WMA Lossless decoding only. There is no native WMA Lossless encoder in FFmpeg.

### 8.2 FFmpeg Decoding — Full CLI Reference

```bash
# Decode WMA Lossless to raw PCM WAV (16-bit)
ffmpeg -i input.wma \
  -c:a pcm_s16le \
  output.wav

# Decode 24-bit WMA Lossless to 24-bit WAV
ffmpeg -i input.wma \
  -c:a pcm_s24le \
  output.wav

# Decode with resampling
ffmpeg -i input.wma \
  -c:a pcm_s16le \
  -ar 48000 \
  output.wav

# Extract specific stream (multi-stream ASF)
ffmpeg -i input.wma -map 0:a:0 -c:a copy output.wma

# Probe format without decoding
ffprobe -v quiet -print_format json -show_streams -show_format input.wma

# Decode with channel downmix (5.1 to stereo)
ffmpeg -i input_5.1.wma \
  -c:a pcm_s16le \
  -ac 2 \
  output_stereo.wav
```

### 8.3 FFmpeg Decoding — libavcodec C API

```c
// ─── Complete decode program skeleton ────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.wma", NULL, NULL);
avformat_find_stream_info(fmt_ctx, NULL);

// Find audio stream
int audio_idx = av_find_best_stream(fmt_ctx, AVMEDIA_TYPE_AUDIO, -1, -1, NULL, 0);
AVStream *stream = fmt_ctx->streams[audio_idx];

// Verify codec is WMA Lossless
if (stream->codecpar->codec_id != AV_CODEC_ID_WMALOSSLEX) {
    fprintf(stderr, "Not a WMA Lossless stream\n");
    exit(1);
}

// Find and open decoder
const AVCodec *dec = avcodec_find_decoder(stream->codecpar->codec_id);
AVCodecContext *dec_ctx = avcodec_alloc_context3(dec);
avcodec_parameters_to_context(dec_ctx, stream->codecpar);
avcodec_open2(dec_ctx, dec, NULL);

// Print stream info
printf("Channels: %d\n", dec_ctx->ch_layout.nb_channels);
printf("Sample rate: %d Hz\n", dec_ctx->sample_rate);
printf("Bit depth: %d bits\n", av_get_bytes_per_sample(dec_ctx->sample_fmt) * 8);

// Decode loop
AVPacket *pkt = av_packet_alloc();
AVFrame  *frm = av_frame_alloc();

while (av_read_frame(fmt_ctx, pkt) >= 0) {
    if (pkt->stream_index == audio_idx) {
        avcodec_send_packet(dec_ctx, pkt);
        while (avcodec_receive_frame(dec_ctx, frm) == 0) {
            // frm->data[0..channels-1] contain audio samples
            // frm->nb_samples = sample count per channel
            // frm->format = AVSampleFormat (S16, S16P, S32, S32P)
            // frm->sample_rate = actual rate
            // frm->pts = presentation timestamp
            printf("Frame: %d samples at %d Hz, format=%d\n",
                   frm->nb_samples, frm->sample_rate, frm->format);
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

### 8.4 FFmpeg Metadata Handling

```bash
# Read all metadata as JSON
ffprobe -v quiet -print_format json -show_format input.wma | jq .format.tags

# Write metadata (copy audio, update tags)
ffmpeg -i input.wma \
  -c copy \
  -metadata title="Song Title" \
  -metadata author="Artist Name" \
  -metadata album="Album Name" \
  -metadata year="2024" \
  -metadata genre="Electronic" \
  output.wma

# Strip all metadata
ffmpeg -i input.wma -c copy -map_metadata -1 output.wma

# Embed cover art
ffmpeg -i input.wma -i cover.jpg \
  -c copy \
  -metadata:s:v title="Album cover" \
  -disposition:v attached_pic \
  output.wma
```

#### FFmpeg Internal Metadata Key Mapping
| Standard Field | FFmpeg Key | ASF Native Key | Notes |
|----------------|------------|--------------------------|-------|
| Title | title | Title | |
| Artist | artist | Author | |
| Album | album | WM/AlbumTitle | |
| Album Artist | album_artist | WM/AlbumArtist | |
| Track Number | track | WM/TrackNumber | |
| Disc Number | disc | WM/PartOfSet | |
| Genre | genre | WM/Genre | |
| Date/Year | date | WM/Year | |
| Comment | comment | Description | |
| Composer | composer | WM/Composer | |
| Copyright | copyright | Copyright | |
| Encoder | encoder | WM/EncodedBy | Auto-set by encoder |

### 8.5 Quality / Fidelity Decision Table
| Use Case | Settings | Expected Size | Notes |
|----------|----------|---------------|-------|
| Bit-perfect archival | FFmpeg decode only | 40–60% of PCM | Lossless |
| High-fidelity playback | FFmpeg decode | Variable | Depends on source bit depth |
| Cross-platform archival | FFmpeg decode + convert to FLAC | ~50% of PCM | Convert to FLAC for wider support |

**Note:** FFmpeg cannot encode WMA Lossless. For encoding, use:
- Windows Media Encoder (Microsoft, discontinued)
- Expression Encoder (Microsoft, discontinued)
- Third-party encoders with WMA Lossless support

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
```
ASF Index Object:
  Location:    End of file (after data objects)
  GUID:        D6E229D3-35DA-11D1-9034-00A0C90349BE
  Entry size:  12 bytes
  Entry format:
    [0x00–0x03]  Packet Number (uint32)
    [0x04–0x07]  Packet Count (uint32)
    [0x08–0x0B]  Time Offset (uint32, ms)
  Index intervals: 1 second default (configurable)
```

### 9.2 Gapless Playback Data
```
Encoder delay:   0 samples (lossless codec)
Padding:         0 samples (lossless codec)
Storage location: ASF Presentation Time (implicit)

FFmpeg gapless flags:
  Input:  Automatic handling via ASF parsing
  Output: ASF index generated for seeking
  
Gapless detection:
  WMA Lossless: No delay or padding; gapless by design
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | ASF designed for streaming |
| Algorithmic encoder delay | ~0 samples | Lossless codec; no lookahead |
| Live encoding feasible | Yes | Real-time encoding possible |
| HTTP progressive download | Yes | ASF over HTTP |
| HTTP Live Streaming (HLS) | No | Not natively supported |
| DASH streaming | Partial | Via ASF segmenting |
| WebRTC / RTP transport | Yes | Via RTSP/MMS |
| Minimum decode buffer | 1 tile / ~46 ms | Depends on tile size |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | Layout Name | Channel Order | FFmpeg Layout Constant | WMA Lossless Support |
|----------|-------------|--------------|------------------------|---------------------| 
| 1 | Mono | C | AV_CHANNEL_LAYOUT_MONO | ✓ |
| 2 | Stereo | L, R | AV_CHANNEL_LAYOUT_STEREO | ✓ |
| 3 | 2.1 | L, R, LFE | AV_CHANNEL_LAYOUT_2POINT1 | ✓ |
| 4 | Quad | FL, FR, RL, RR | AV_CHANNEL_LAYOUT_QUAD | ✓ |
| 5 | 5.0 | FL, FR, C, SL, SR | AV_CHANNEL_LAYOUT_5POINT0 | ✓ |
| 6 | 5.1 | FL, FR, C, LFE, SL, SR | AV_CHANNEL_LAYOUT_5POINT1 | ✓ |
| 7 | 6.1 | FL, FR, C, LFE, BL, BC, BR | AV_CHANNEL_LAYOUT_6POINT1 | ✓ |
| 8 | 7.1 | FL, FR, C, LFE, BL, BR, SL, SR | AV_CHANNEL_LAYOUT_7POINT1 | ✓ |

### 11.2 Downmix Coefficients (5.1 → Stereo)
```
L_out = L + (C × 0.7071) + (SL × 0.7071)
R_out = R + (C × 0.7071) + (SR × 0.7071)
LFE:  Discarded (or mixed with coefficient 0.0–1.0 based on implementation)

FFmpeg automatic downmix:
  WMA Lossless decoder automatically downmixes when stereo output is requested

FFmpeg manual downmix command:
ffmpeg -i input_5.1.wma \
  -af "apan=stereo|FL=FC+0.707*FC+0.707*BL|FR=FC+0.707*FC+0.707*BR" \
  output_stereo.wav
```

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 24-bit | Integer only |
| Max sample rate | 96000 Hz | |
| Float support | None | |
| DSD support | No | |
| 20-bit support | Yes | |
| 24-bit support | Yes | Primary high-resolution format |

```bash
# High-res WMA Lossless decoding example
ffmpeg -i input_24bit_96k.wma \
  -c:a pcm_s24le \
  output_24_96.wav
```

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

| Platform / API | Encode | Decode | FFmpeg Flag | Notes |
|----------------|--------|--------|-------------|-------|
| Windows Media Foundation | Yes | Yes | Native | Microsoft's official API |
| FFmpeg native | No | Yes | None | Decode only; no encoder |
| NVIDIA NVENC | No | — | N/A | WMA Lossless not supported |
| NVIDIA NVDEC | — | No | N/A | Not implemented |
| Intel QSV | No | Yes | `-hwaccel qsv` | Decode only |
| Apple AudioToolbox | No | No | N/A | No WMA Lossless support |
| Android MediaCodec | No | Partial | N/A | OEM-dependent |
| VA-API (Linux) | No | Yes | `-hwaccel vaapi` | Decode only |

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Decoder-Specific Issues
| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| Windows 10 broken decoders | Windows 10 certain builds | Use FFmpeg for decode |
| No encoder available | All | Use Microsoft encoder or transcoding |
| 24-bit support inconsistent | <4.0 | Update FFmpeg version |
| Seeking accuracy | Some versions | Use ASF index if present |

### 14.2 Known Windows 10 Decoder Bug
Microsoft shipped several Windows 10 updates with broken WMA Lossless decoders that produced non-lossless output. FFmpeg's decoder produces correct lossless output and is recommended for bit-perfect archival.

### 14.3 Interoperability Issues
- **Microsoft encoder → FFmpeg decoder:** Generally compatible; bit-exact
- **FFmpeg encoder → Microsoft decoder:** Not applicable (no FFmpeg encoder)
- **Files with DRM:** FFmpeg cannot decode DRM-protected WMA Lossless files
- **High-resolution files:** Some hardware decoders may not support 24-bit/96 kHz
- **Cross-platform playback:** Limited; FFmpeg recommended for non-Windows

### 14.4 Edge Cases to Handle in Converter
- **Empty file / 0-sample stream:** Output empty file; no error
- **File < 1 tile of audio:** Output partial tile; decoder handles
- **All-silence audio:** WMA Lossless encodes as small file; trivial compression
- **Corrupt tiles:** Decoder may output partial frame; CRC checks if present
- **File with corrupt header:** FFmpeg may decode partially or fail with error
- **Truncated file:** Last tile may be corrupt; decoder outputs what it can
- **Sample rate not supported:** Error: "sample rate not supported"
- **Channel count not supported (>8):** Error: "unsupported number of channels"

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM WMA Lossless

| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → FLAC | `ffmpeg -i in.wma -c:a flac -compression_level 8 out.flac` | All ASF tags → Vorbis Comments | Lossless → Lossless ✓ |
| → ALAC | `ffmpeg -i in.wma -c:a alac out.m4a` | Partial (tag mapping) | Lossless → Lossless ✓ |
| → WAV | `ffmpeg -i in.wma -c:a pcm_s16le out.wav` | RIFF INFO tags | Lossless → Uncompressed |
| → WAV (24-bit) | `ffmpeg -i in.wma -c:a pcm_s24le out.wav` | RIFF INFO tags | Lossless → Uncompressed |
| → MP3 | `ffmpeg -i in.wma -c:a libmp3lame -q:a 0 out.mp3` | Tags via ID3v2.3 | Lossless → Lossy |
| → AAC | `ffmpeg -i in.wma -c:a aac -b:a 256k out.m4a` | Partial | Lossless → Lossy |
| → Opus | `ffmpeg -i in.wma -c:a libopus -b:a 128k out.opus` | Vorbis Comments (recreated) | Lossless → Lossy |
| → OGG Vorbis | `ffmpeg -i in.wma -c:a libvorbis -q:a 10 out.ogg` | Vorbis Comments (recreated) | Lossless → Lossy |

### 15.2 Converting TO WMA Lossless

**CRITICAL:** FFmpeg cannot encode WMA Lossless. Use third-party tools for encoding.

| Source | Tool | Command | Notes |
|--------|------|---------|-------|
| FLAC → | Windows Media Encoder | GUI tool | Requires Windows |
| WAV → | Expression Encoder | GUI tool | Requires Windows |
| Any → | dBpoweramp | GUI tool | Cross-platform encoder wrapper |
| Any → | Steinberg Wavelab | GUI tool | Professional |

### 15.3 Lossless Round-Trip Verification
```bash
# Encode (requires external encoder)
# Example: Convert FLAC to WMA Lossless using external tool
# then decode and verify

# Decode WMA Lossless to WAV
ffmpeg -i input.wma -c:a pcm_s16le decoded.wav

# Decode original FLAC to reference WAV
ffmpeg -i original.flac -c:a pcm_s16le reference.wav

# Compare checksums
md5sum reference.wav decoded.wav
# Output must MATCH for bit-perfect lossless

# Or use FFmpeg's built-in checksumming:
ffmpeg -i original.flac -map 0:a -f framemd5 original.md5
ffmpeg -i input.wma -map 0:a -f framemd5 decoded.md5
diff original.md5 decoded.md5   # Empty diff = bit-perfect
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| FFmpeg native | C | LGPL 2.1+ | N/A (no encoder) | 10/10 | https://ffmpeg.org |
| Windows Media Format SDK | C/C++ | Proprietary | 10/10 | 10/10 | Microsoft |
| LAV Filters | C/C++ | LGPL 2.1+ | N/A | 10/10 | https://github.com/Nevcairiel/LAVFilters |
| Media Foundation | C/C++ | Proprietary | Yes | Yes | Windows native |

### Build Instructions (for bundling in converter app)
```bash
# Build FFmpeg from source with WMA Lossless support
git clone https://github.com/FFmpeg/FFmpeg.git
cd FFmpeg
./configure --prefix=/usr/local \
  --enable-gpl --enable-libass --enable-libfreetype
make -j$(nproc)
make install

# Verify WMA Lossless decode support:
ffmpeg -decoders | grep -i wmalossless
# Output should show: wmalossless

# Note: No encoder will be listed for WMA Lossless
ffmpeg -encoders | grep -i wmalossless
# Output: (empty)
```

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **WMA Lossless Codec:** No public specification; reverse-engineered by FFmpeg
- **ASF Specification:** Microsoft ASF Specification (for container format)

### Technical Resources
- FFmpeg decoder: `ffmpeg -h decoder=wmalossless` or https://ffmpeg.org/ffmpeg-codecs.html
- FFmpeg ASF muxer: `ffmpeg -h muxer=asf` or https://ffmpeg.org/ffmpeg-formats.html
- Multimedia Wiki: https://wiki.multimedia.cx/index.php/Windows_Media_Audio
- Hydrogenaudio: https://wiki.hydrogenaudio.org/index.php?title=WMA
- Hydrogenaudio Lossless Comparison: https://wiki.hydrogenaudio.org/index.php?title=Lossless_comparison
- Microsoft WMA Codecs: https://learn.microsoft.com/en-us/windows/win32/medfound/about-the-windows-media-codecs

### Academic Papers
- No significant academic papers on WMA Lossless (proprietary codec)

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] Identify FFmpeg `./configure` flags (default build includes WMA Lossless decoder)
- [ ] Verify `ffmpeg -decoders` output confirms wmalossless decoder is available
- [ ] **IMPORTANT:** Note that NO WMA Lossless encoder exists in FFmpeg
- [ ] For encoding, recommend third-party tools or transcoding to alternative format
- [ ] Note platform restrictions (macOS/iOS may need transcoding)

### Decoding Pipeline
- [ ] ASF container parsing (use avformat)
- [ ] Verify stream codec_id is AV_CODEC_ID_WMALOSSLEX
- [ ] Handle AVERROR(EAGAIN) in receive_packet loop correctly
- [ ] Flush decoder at EOF (send NULL packet)
- [ ] Handle variable sample format (S16, S16P, S32, S32P)
- [ ] Handle variable channel layouts (1–8 channels)
- [ ] Handle high sample rates (up to 96 kHz)
- [ ] Implement bit-perfect verification (md5/framemd5)

### Metadata
- [ ] Read ASF Content Description Object fields
- [ ] Read ASF Extended Content Description Object fields
- [ ] Map all tag fields through standard key mapping
- [ ] Read WM/Picture cover art
- [ ] Write all standard tag fields to ASF container (for copy operations)
- [ ] Embed cover art as WM/Picture binary
- [ ] Preserve ReplayGain tags through conversion
- [ ] Handle UTF-16LE encoding (ASF native)

### Quality & Verification
- [ ] Implement bit-perfect verification for lossless conversions
- [ ] Use FFmpeg framemd5 for verification
- [ ] Compare original and decoded checksums
- [ ] Implement progress reporting from FFmpeg stats output
- [ ] Implement error detection and partial-file recovery
- [ ] Test with: silence, full-scale, multichannel, high-resolution files

### Edge Cases
- [ ] Handle files with corrupt or missing ASF headers
- [ ] Handle files with 0 samples
- [ ] Handle sample rate mismatch (trigger libswresample)
- [ ] Handle channel count mismatch (downmix if requested)
- [ ] Handle bit depth mismatch (convert from decoder output)
- [ ] Handle very short files (< 1 tile)
- [ ] Handle DRM-protected files (FFmpeg cannot decode)

### Encoding (Alternative Path)
- [ ] If encoding WMA Lossless is required, recommend using external tools
- [ ] Document that FFmpeg cannot encode WMA Lossless
- [ ] Provide guidance on Windows Media Encoder alternatives
- [ ] Consider transcoding to FLAC/ALAC as primary lossless path

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
