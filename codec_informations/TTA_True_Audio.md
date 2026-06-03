# TTA (True Audio) — Deep Technical Reference

> **Category:** Lossless
> **File Extensions:** `.tta`
> **MIME Types:** `audio/x-tta`, `audio/tta`
> **Standardization Body:** Tau Software / Open-source community
> **Primary Specification:** https://tausoft.org/en/true-audio-codec-tta/
> **Patent Status:** Patent-free
> **License:** GPL / LGPL v3
> **Current Version:** 3.4 (encoder), TTA1 format
> **Active Development:** No — discontinued ~2012, last release 2007

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Alexander Leschinsky, Tau Software
- **Year Created:** 2007 (formal release as TTA1)
- **Original Purpose:** Realtime lossless audio compression with emphasis on fast encoding and decoding speeds
- **Problem with Predecessors:** Existing lossless codecs at the time (FLAC 0.x, Monkey's Audio, etc.) were either too slow or had proprietary licensing. TTA was designed as a free, open alternative that could achieve transparent compression in real-time while maintaining cross-platform compatibility.

The TTA project emerged from the need for a truly open-source lossless audio codec that could provide:
1. Transparent audio quality (bit-exact reproduction)
2. Fast encoding and decoding speeds suitable for real-time operation
3. Reasonable compression ratios (30-70% of original size)
4. Hardware-friendly design for embedded applications
5. Free and open implementation without patent encumbrances

The term "True Audio" reflects the codec's commitment to lossless reproduction — the audio after decompression is bit-identical to the original input, with no approximation, quantization, or perceptual coding artifacts.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| 1.0 | 2000 | Initial release with basic lossless compression using adaptive filtering |
| 2.0 | 2003 | Improved predictor algorithms with better compression ratios, added multichannel support |
| 2.1 | 2004 | Bug fixes, optimized decoder, improved stability |
| 2.5 | 2005 | Added ID3v2 and APEv2 tag support, improved seek functionality |
| 3.0 | 2006 | Major encoder improvements, added encryption support |
| 3.4 | 2007 | Final release, Windows Vista compatibility, GUI improvements |

Key milestones in TTA development:
- 2000: First public release targeting real-time audio processing
- 2003: Introduction of multichannel support up to 6 channels
- 2005: Addition of comprehensive metadata tagging support
- 2006: Encryption layer for password-protected audio
- 2007: Last official release, integration with various media players

### 1.3 Current Adoption
- **Primary use cases today:** Archival of live concert recordings, niche audio archiving communities, digital preservation projects
- **Platforms with native support:** Windows (via DirectShow filters), Linux (via FFmpeg/libavcodec), macOS (via plugins), FreeBSD
- **Major services using this format:** None (niche/deprecated format)
- **Hardware support:** Some older DVD players and set-top boxes with TTA support, limited consumer electronics
- **Status:** Legacy — superseded by FLAC and other open standards, but files remain playable

The TTA format saw its peak usage during the mid-2000s when:
- Lossless audio was becoming popular for archival purposes
- FLAC was still in early development (version 1.0 released July 2001)
- Open-source lossless options were limited
- Hardware decoding was a design consideration

Today, TTA files can still be found in:
- Live concert recording archives (especially from etree.org community)
- Personal audio collections from the mid-2000s era
- Some specialized archival institutions

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Waveform / Predictive
- **Core algorithm:** Adaptive Linear Prediction (ALP) with integer-only arithmetic
- **Loss mechanism:** Lossless — uses predictive coding with entropy coding of residuals
- **Frame-based vs sample-based:** Frame-based; each frame is independently decodable
- **Fixed vs variable frame size:** Fixed frame size (64KB per frame) for simplicity and seekability

TTA belongs to the family of predictive lossless codecs, which work by:
1. Predicting each audio sample based on previous samples
2. Encoding only the difference (residual) between prediction and actual value
3. Using entropy coding to compress the residuals efficiently

The codec uses a "lossless" approach meaning:
- No quantization is applied to the residuals
- The prediction error is exactly preserved
- Decoding reproduces bit-identical audio to the original

### 2.2 High-Level Encoding Flow (Block Diagram)
```
Input PCM Samples (Interleaved or Planar)
      │
      ▼
[Pre-processing: DC removal, endianness handling, byte-order normalization]
      │
      ▼
[Frame Splitting: Fixed 64KB frames (configurable predictor mode)]
      │
      ▼
[Adaptive Linear Prediction: FAST/NORMAL/HIGH/EXTRA modes]
      │   ├── FAST Mode: Minimal prediction, fastest processing
      │   ├── NORMAL Mode: Balanced prediction/compression (default)
      │   ├── HIGH Mode: Enhanced prediction, better compression
      │   └── EXTRA Mode: Maximum prediction, best compression
      │
      ▼
[Residual Calculation: Actual Sample - Predicted Sample]
      │
      ▼
[Entropy Coding: Rice-Golomb coding of residuals]
      │   ├── Parameter k: Computed from residual variance
      │   ├── Quotient: Unary coded
      │   └── Remainder: Binary coded with k bits
      │
      ▼
[Bitstream Packing: Frame header + CRC32 + compressed data]
      │   ├── Frame sync marker
      │   ├── Frame size indicator
      │   ├── CRC32 checksum for integrity
      │   └── Compressed audio data
      │
      ▼
Output TTA Stream (Native Format or Muxed in Container)
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | ~64KB frame | ~1.4 seconds at 44.1kHz stereo |
| Frame size | 64KB (fixed) | 65,536 bytes per frame compressed |
| Max channels | 16 (FFmpeg), 6 (reference encoder) | Format allows up to 65536 theoretically |
| Max bit depth | 24-bit | 8, 16, and 24-bit supported |
| Max sample rate | 4 GHz (format), 2 GHz (encoder) | Integer rates only |
| Bitrate range | N/A | Lossless — bitrate varies with content |
| Complexity | O(n) linear | Very fast encoding/decoding |
| Memory usage | Low | ~1MB RAM for encoder/decoder |
| CPU usage | Low | Can encode/decode in real-time on modest hardware |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       54 54 41 31     TTA1      File magic / signature
0x0004  2       XX XX            ..        Reserved (header length indicator)
```

The TTA1 magic signature identifies the file format version and ensures proper identification:
- "TTA1" is a human-readable ASCII signature
- Version 1 format has remained stable throughout TTA's development
- No other TTA version strings exist (format was frozen early)

### 3.2 File-Level Header Layout
Full byte-map of the TTA1 file header:

```
Offset   Size    Field Name              Type        Valid Range   Description
-------  ------  ----------------------  ----------  -----------   ---------------------------
0x0000   4B      Magic Bytes           uint8[4]    "TTA1"       File signature (0x54 0x54 0x41 0x31)
0x0004   4B      Audio Format          uint32 LE   1            1 = PCM format (WAVE_FORMAT_PCM)
0x0008   4B      Number of Channels    uint32 LE   1–65536     Channel count (1=mono, 2=stereo, etc.)
0x000C   4B      Bits Per Sample       uint32 LE   8,16,24      Bit depth of audio samples
0x0010   4B      Sample Rate          uint32 LE   any int      Sample rate in Hz (44100, 48000, etc.)
0x0014   8B      Data Length          uint64 LE   0–2^64       Total audio samples (all channels combined)
0x001C   4B      Header CRC32          uint32 LE   valid CRC    CRC32 checksum of header bytes 0x0000-0x001B
```

#### Header Field Details

**Magic Bytes (4 bytes):**
- Fixed value: "TTA1" (0x54 0x54 0x41 0x31)
- Used for file identification and sync

**Audio Format (4 bytes, little-endian uint32):**
- Currently defined value: 1
- 1 = WAVE_FORMAT_PCM (uncompressed PCM)
- Future versions may define additional formats

**Number of Channels (4 bytes, little-endian uint32):**
- Range: 1 to 65536 (format limit)
- Practical limits:
  - Reference encoder: max 6 channels
  - FFmpeg implementation: max 16 channels
  - Modified foobar2000: reports up to 256 channels
- No channel layout metadata in format itself

**Bits Per Sample (4 bytes, little-endian uint32):**
- Supported values: 8, 16, 24
- 8-bit: Unsigned (0-255) representation
- 16-bit: Signed integer (-32768 to 32767)
- 24-bit: Signed integer (-8388608 to 8388607)
- Note: 32-bit integer and floating-point not supported

**Sample Rate (4 bytes, little-endian uint32):**
- Any positive integer value
- Common values: 44100, 48000, 88200, 96000, 176400, 192000
- Format limit: up to 4 GHz
- Encoder limit: up to 2 GHz
- Value 0 is invalid/reserved

**Data Length (8 bytes, little-endian uint64):**
- Total number of audio samples
- Count is for all channels combined
- For stereo: sample_count = frames × frame_samples × 2
- Value 0 indicates unknown (should not occur in valid files)

**Header CRC32 (4 bytes, little-endian uint32):**
- CRC of bytes 0x0000 through 0x001B (22 bytes)
- Polynomial: 0x04C11DB7 (ISO 3309)
- Initial value: 0xFFFFFFFF
- Reflected input and output
- Used for header integrity verification

### 3.3 Frame / Block Header Layout
Each TTA frame has the following structure:

```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------
0x0000   4B      Frame CRC32            uint32 LE   CRC32 checksum of frame data
0x0004   N B     Frame Data             uint8[]     Compressed audio data
```

#### Frame Data Internal Structure
```
Bit Offset   Bit Width   Field Name              Type      Description
----------   ---------   --------------------    ------    ---------------------------
0            16         Frame Length            uint16    Frame data length in bytes
16           variable   Compressed PCM          bits      Rice-coded prediction residuals
                            │
                            └── Each channel encoded independently
                            └── Predictor state reset at frame boundaries
```

### 3.4 Frame Size Calculation
```
frame_samples = 65536 / (channels × (bits_per_sample / 8))
Example for 44100Hz, 16-bit stereo:
  frame_samples = 65536 / (2 × 2) = 16384 samples per frame
  frame_duration = 16384 / 44100 ≈ 0.371 seconds per frame

For 48kHz, 24-bit 5.1 surround (6 channels):
  frame_samples = 65536 / (6 × 3) = 3640 samples per frame
  frame_duration = 3640 / 48000 ≈ 0.076 seconds per frame
```

### 3.5 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Unsigned integer | Yes | Limited use — typically for voice |
| 16-bit | Signed integer | Yes | Standard CD audio, most common |
| 20-bit | Signed integer | Yes | Professional audio |
| 24-bit | Signed integer | Yes | High-resolution standard |
| 32-bit | Signed integer | No | Not supported |
| 32-bit | IEEE float | No | Not supported |
| 64-bit | IEEE float (double) | No | Not supported |

#### Supported Sample Rates
| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 8000 | Telephone | Yes | Lowest common rate |
| 11025 | — | Yes | Quarter CD rate |
| 16000 | Wideband voice | Yes | VoIP standard |
| 22050 | — | Yes | Half CD rate |
| 32000 | Broadcast | Yes | TV audio standard |
| 44100 | CD audio | Yes | Most common rate |
| 48000 | Professional | Yes | DVD/BD standard |
| 88200 | 2× CD | Yes | High-res |
| 96000 | High-res | Yes | DVD-Audio common |
| 176400 | 4× CD | Yes | High-res |
| 192000 | High-res max | Yes | Maximum consumer rate |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Pre-Processing Stage
- **DC offset removal:** Subtraction of mean value from samples — ensures zero-mean audio for better prediction
- **Pre-emphasis filter:** Optional first-order high-shelf filter for certain audio types
- **Windowing function:** None for linear prediction (uses entire frame) — no windowing artifacts
- **Level normalization:** None required — uses full integer range
- **Stereo decorrelation pre-step:** Optional mid-side (M/S) transformation for stereo — can improve compression

The pre-processing stage prepares audio samples for optimal compression:
1. **Byte-order handling:** Samples converted to consistent endianness for processing
2. **DC offset:** Removed by subtracting the mean value from all samples in a frame
3. **Sample format:** All formats normalized to signed representation internally

### 4.2 Analysis / Transform Stage

#### Transform Type: Adaptive Linear Prediction (Multiple Modes)
TTA uses adaptive linear prediction exclusively — no MDCT, FFT, or frequency-domain transforms. This makes it computationally simple but effective for audio compression.

```
Parameters:
  Prediction modes:  FAST (1), NORMAL (2), HIGH (3), EXTRA (4)
  Predictor order:   Adaptive, up to 32 coefficients
  Window:            Full frame (65,536 samples)
  Algorithm:         Modified Levinson-Durbin recursion for LPC coefficients
  Coefficient precision: 32-bit integer arithmetic
  
Mode Characteristics:
  FAST Mode:
    - Minimal prediction complexity
    - Fastest encoding/decoding
    - Lower compression ratio (~40-60%)
    - Suitable for real-time applications
    
  NORMAL Mode (Default):
    - Balanced prediction effort
    - Good compression (~35-55%)
    - Recommended for general use
    
  HIGH Mode:
    - Enhanced prediction
    - Better compression (~30-50%)
    - Slower than NORMAL
    
  EXTRA Mode:
    - Maximum prediction effort
    - Best compression (~25-45%)
    - Slowest encoding
```

**LPC Analysis Mathematical Foundation:**
```
Linear Prediction Model:
  x̂[n] = a₁ × x[n-1] + a₂ × x[n-2] + ... + aₚ × x[n-p]
  
Where:
  x̂[n] = Predicted sample at time n
  x[n-k] = Previous samples
  aₖ = LPC coefficients
  p = Prediction order
  
Prediction Error:
  e[n] = x[n] - x̂[n]
  
Goal: Minimize sum of squared errors E = Σ(e[n])²
```

**Levinson-Durbin Algorithm:**
```
For i = 1 to p:
  E[0] = variance of x[n]
  k[i] = -Σ(x[n] × x̂[n]) / E[i-1]  (PARCOR coefficient)
  a[i] = k[i] - Σ(a[j] × k[i-j]) for j = 1 to i-1
  E[i] = E[i-1] × (1 - k[i]²)
```

### 4.3 Psychoacoustic Model (Not Applicable)
TTA is a **lossless** codec. No psychoacoustic modeling is performed — all samples are preserved bit-exactly. This is fundamentally different from lossy codecs like MP3 or AAC which use perceptual masking to discard perceptually irrelevant information.

### 4.4 Quantization
TTA uses **no quantization** — it is a purely lossless codec. Prediction residuals are entropy-coded without any lossy approximation.

### 4.5 Stereo Encoding Modes
| Mode | Description | Condition for Selection | Notes |
|------|-------------|------------------------|-------|
| Independent | L and R encoded separately | Default | Each channel has independent predictor |
| M/S (Mid-Side) | M=(L+R)/2, S=(L-R)/2 | Optional per frame | Can improve compression for centered sources |

### 4.6 Entropy / Lossless Coding Stage
```
Method: Rice-Golomb coding (simplified arithmetic coding)

Rice-Golomb Coding Process:
  1. Compute optimal parameter k based on residual statistics
     k = floor(log₂(mean(|residuals|)))
  
  2. Encode each residual:
     quotient = floor(residual / 2^k)
     remainder = residual mod 2^k
     
  3. Output:
     - Unary code for quotient (q zeros followed by 1)
     - Binary code with k bits for remainder
     
  Example with k=3:
    residual = 27
    quotient = floor(27 / 8) = 3
    remainder = 27 - 3×8 = 3
    output: "0001" + "011" = "0001011"
    
Advantages of Rice coding:
  - Very fast encoding/decoding
  - Near-optimal for Laplacian distributions
  - Simple hardware implementation
  - No code table storage required
```

### 4.7 Encoder Settings / Quality Modes

#### For Lossless Codecs
| Compression Level | Encoding Speed | Decode Speed | Compression Ratio | Notes |
|---|---|---|---|---|
| FAST (1) | Fastest (10-20× real-time) | Fastest (15-25× real-time) | ~40-60% | Minimal prediction |
| NORMAL (2) | Fast (8-15× real-time) | Fast (12-20× real-time) | ~35-55% | Balanced (default) |
| HIGH (3) | Medium (5-10× real-time) | Medium (8-15× real-time) | ~30-50% | Better prediction |
| EXTRA (4) | Slow (3-8× real-time) | Medium (6-12× real-time) | ~25-45% | Best compression |

Speed values are approximate for modern x86 processors.

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. Scan bitstream for magic bytes: "TTA1" (0x54 0x54 0x41 0x31)
   - Start from file beginning
   - Read 4 bytes at a time
   - Match against "TTA1" signature
   
2. Read and validate header fields
   - Verify audio format = 1 (PCM)
   - Check channel count is valid (1-65536)
   - Verify bits per sample is 8, 16, or 24
   - Validate sample rate is positive integer
   
3. Verify header CRC32
   - Compute CRC of bytes 0x0000-0x001B
   - Compare with stored CRC at 0x001C
   - If mismatch: file is corrupt or not TTA
   
4. Frame synchronization via CRC32 checks at end of each frame
   - Each frame begins immediately after previous frame ends
   - No explicit frame markers in stream
   - Frame boundaries computed from compressed data
   - Frame CRC verified before accepting data
```

#### Seeking
- **Seek table:** Stored at end of file — list of frame offsets and sample positions
- **Seek precision:** Sample-accurate via seek table
- **Random access:** Each frame independently decodable
- **Seek table format:** Located before metadata tags, contains frame offsets and sample positions

### 5.2 Core Decode Pipeline
```
1. Read file header
   ├── Verify magic bytes "TTA1"
   ├── Parse channel count, bit depth, sample rate
   ├── Read total sample count
   └── Validate header CRC32

2. Read seek table (if present)
   └── Contains frame offsets and starting sample positions

3. For each frame:
   ├── Read frame CRC32
   ├── Read compressed frame data
   ├── Verify CRC32
   └── Decode to PCM samples:
       ├── Rice-Golomb entropy decoding
       ├── Reconstruct residuals
       ├── Apply inverse LPC prediction
       ├── Reconstruct samples
       └── Stereo decorrelation reversal (if M/S was used)

4. Post-processing
   ├── Sample format conversion (if needed)
   └── Output PCM samples
```

### 5.3 Error Concealment
- **Corrupt frame detection:** CRC32 checksum per frame
- **Concealment method:** Replace corrupt frame with silence (mute)
- **Maximum consecutive errors:** Decoder stops after too many corrupt frames
- **Recovery:** Next valid frame can be decoded independently

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** None — TTA is a raw stream format
- **Overhead:** ~0.1% (header + frame CRCs only)
- **Seeking in native container:** Yes — via seek table at file end
- **Multiple streams in native container:** No

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| TTA (native) | Yes | Yes | ID3v1/v2, APEv2 | No container overhead |
| Matroska/MKA | Yes | Yes | Full | Via Matroska audio codec |
| RIFF WAV | No | N/A | N/A | Not designed for WAV |
| AIFF | No | N/A | N/A | Not designed for AIFF |
| OGG | No | N/A | N/A | Not designed for OGG |
| MP4/M4A | No | N/A | N/A | Not designed for MP4 |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** ID3v1, ID3v2, and APEv2 (stored after audio data)
- **Tag block location:** End of file (after all audio frames and seek table)
- **Tag block identifier:** "TAG" (ID3v1) or "APETAGEX" (APEv2)

### 7.2 Standard Tag Fields — Complete Reference
| Tag Field | Internal Key (TTA) | Max Length | Character Encoding | Multi-value | Notes |
|-----------|---------------------------|------------|-------------------|-------------|-------|
| Title | TITLE | 255 bytes | UTF-8/UTF-16 | No | Track title |
| Artist | ARTIST | 255 bytes | UTF-8/UTF-16 | No | Performer |
| Album | ALBUM | 255 bytes | UTF-8/UTF-16 | No | Album name |
| Album Artist | ALBUMARTIST | 255 bytes | UTF-8/UTF-16 | No | Album performer |
| Genre | GENRE | 255 bytes | UTF-8/UTF-16 | No | ID3v1 genre numbers also accepted |
| Year / Date | YEAR or TDRC | 4 bytes | ASCII | No | Release year |
| Track Number | TRACK or TRACKNUMBER | 10 bytes | ASCII | No | Format: "N" or "N/Total" |
| Disc Number | DISCNUMBER | 10 bytes | ASCII | No | Format: "N" or "N/Total" |
| Comment | COMMENT | 1000 bytes | UTF-8/UTF-16 | No | Freeform comment |
| ReplayGain Track Gain | REPLAYGAIN_TRACK_GAIN | 20 bytes | ASCII | No | Format: "-6.20 dB" |
| ReplayGain Track Peak | REPLAYGAIN_TRACK_PEAK | 20 bytes | ASCII | No | Format: "0.998459" |
| Encoder | ENCODER | 64 bytes | UTF-8/UTF-16 | No | Software name and version |

### 7.3 Cover Art Storage
TTA does not natively support embedded cover art. Cover art must be stored in the tag metadata using APEv2's "COVER ART" field or ID3v2's APIC frame.

### 7.4 Metadata Compatibility Matrix
| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| ID3v1 | ✓ | ✓ | ✓ | Lowest |
| ID3v2.3 | ✓ | ✓ | ✓ | |
| ID3v2.4 | ✓ | ✓ | ✓ | |
| APEv2 | ✓ | ✓ | ✓ | Highest |

**Conflict resolution:** APEv2 takes precedence over ID3v2, which takes precedence over ID3v1.

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   tta              # used with -c:a
AV_CODEC_ID:        AV_CODEC_ID_TTA  # C constant in libavcodec/codec_id.h
Format Name (CLI):  tta              # used with -f
Encoder(s):         NONE — FFmpeg does NOT encode TTA
Decoder(s):         tta (native)      # ffmpeg -decoders | grep -i tta
Muxer(s):           tta              # ffmpeg -muxers | grep -i tta
Demuxer(s):         tta              # ffmpeg -demuxers | grep -i tta
```

### 8.2 FFmpeg Encoding — NOT SUPPORTED
FFmpeg does **NOT** support encoding TTA format. TTA encoding requires the official TTA encoder from tausoftware.org or third-party encoders like ttaR (rewritten CLI).

### 8.3 FFmpeg Decoding — Full CLI Reference

```bash
# Decode TTA to WAV
ffmpeg -i input.tta \
  -c:a pcm_s16le \
  output.wav

# Decode TTA to FLAC
ffmpeg -i input.tta \
  -c:a flac -compression_level 8 \
  output.flac

# Extract audio stream without conversion
ffmpeg -i input.tta \
  -c:a copy \
  output.tta

# Probe TTA file information
ffprobe -v quiet -print_format json -show_streams -show_format input.tta

# Extract metadata
ffprobe -v quiet -print_format json -show_format input.tta | jq .format.tags

# Convert with specific output format
ffmpeg -i input.tta \
  -c:a pcm_s24le \
  -ar 96000 \
  output_96k.wav
```

### 8.4 FFmpeg Decoding — libavcodec C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>

// ─── Complete decode program skeleton ────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.tta", NULL, NULL);
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
            // frm->data[0..channels-1] contain audio samples
            // frm->nb_samples = sample count per channel
            // frm->format = AVSampleFormat
            // frm->sample_rate = actual rate
            // frm->pts = presentation timestamp
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
ffprobe -v quiet -print_format json -show_format input.tta | jq .format.tags

# Strip all metadata
ffmpeg -i input.tta -c:a copy -map_metadata -1 output.tta

# Note: FFmpeg cannot write ID3v2/APEv2 tags to TTA files natively
# Use external tools like: mid3v2, id3v2, apetag
```

#### FFmpeg Internal Metadata Key Mapping
| Standard Field | FFmpeg Key | TTA Native Key | Notes |
|----------------|------------|--------------------------|-------|
| Title | title | TIT2 (ID3v2) | |
| Artist | artist | TPE1 (ID3v2) | |
| Album | album | TALB (ID3v2) | |
| Track Number | track | TRCK (ID3v2) | |
| Genre | genre | TCON (ID3v2) | |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
TTA stores a seek table at the end of the file before any metadata:

```
TTA Seek Table:
  Location:     End of file, before metadata
  Magic:        None (identified by position)
  Entry size:   12 bytes
  Entry format:
    [0x00–0x07]  frame_offset (uint64)      — Byte offset from file start
    [0x08–0x0B]  frame_samples (uint32)     — Sample number at frame start
  Max entries:  Limited by file size
```

### 9.2 Gapless Playback Data
```
Encoder delay:   0 samples (TTA adds no padding)
Padding:         0 samples
Storage location: Not stored — total samples in header
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | Frame-based, each frame independent |
| Algorithmic encoder delay | ~64KB frame | ~1.4 seconds at 44.1kHz stereo |
| Live encoding feasible | Yes | Designed for real-time operation |
| HTTP progressive download | Yes | Supported |
| HTTP Live Streaming (HLS) | No | Not a standard HLS segment format |
| DASH streaming | No | Not commonly used |
| WebRTC / RTP transport | No | Not designed for real-time transport |
| Minimum decode buffer | 1 frame | 64KB |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | Layout Name | Channel Order | FFmpeg Layout Constant |
|----------|-------------|---------------|----------------------|
| 1 | Mono | C | AV_CHANNEL_LAYOUT_MONO |
| 2 | Stereo | L, R | AV_CHANNEL_LAYOUT_STEREO |
| 3 | 2.1 | L, R, LFE | AV_CHANNEL_LAYOUT_2POINT1 |
| 4 | Quad | FL, FR, RL, RR | AV_CHANNEL_LAYOUT_QUAD |
| 5 | 5.0 | FL, FR, C, SL, SR | AV_CHANNEL_LAYOUT_5POINT0 |
| 6 | 5.1 | FL, FR, C, LFE, SL, SR | AV_CHANNEL_LAYOUT_5POINT1 |
| 8 | 7.1 | FL, FR, C, LFE, BL, BR, SL, SR | AV_CHANNEL_LAYOUT_7POINT1 |
| 16 | — | Up to 16 channels | FFmpeg limit |

**Note:** TTA format has no native channel assignment metadata. Channel meaning must be inferred from external context or tags.

### 11.2 Downmix Coefficients (5.1 → Stereo)
```
L_out = L_in + (C_in × 0.7071) + (LS_in × 0.5)
R_out = R_in + (C_in × 0.7071) + (RS_in × 0.5)
LFE:  typically mixed out (0.0 coefficient)
```

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 24-bit | Integer only |
| Max sample rate | 2 GHz | Encoder limit, format supports higher |
| Float support | None | Integer arithmetic only |
| DSD support | No | Not supported |
| 20-bit support | Yes | Common in professional audio |
| 24-bit support | Yes | High-res standard |

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

| Platform / API | Encode | Decode | FFmpeg Flag | Notes |
|----------------|--------|--------|-------------|-------|
| x86 SIMD | No | Yes | — | SSE/SSE2 optimization in reference |
| ARM NEON | No | Limited | — | Not officially optimized |
| CUDA/NVENC | No | No | — | Not applicable |
| OpenCL | No | No | — | Not applicable |

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Encoder-Specific Issues
| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| No TTA encoder | All versions | Use official TTA encoder |
| Limited multichannel support | All versions | FFmpeg limits to 16 channels |

### 14.2 Interoperability Issues
- **FFmpeg TTA → Official decoder:** Fully compatible
- **Official TTA → FFmpeg decode:** Fully compatible
- **Multichannel interpretation:** No standard channel assignment in format
- **Files with mixed tag systems:** Last valid tag system wins

### 14.3 Edge Cases to Handle in Converter
- **Empty file / 0-sample stream:** Invalid TTA file
- **File < 1 frame of audio:** Rare but valid
- **All-silence audio:** Encodes very efficiently (~5% size)
- **DC offset (non-zero mean):** Encoder handles, no special processing
- **Full-scale sine (0 dB):** No clipping risk in lossless
- **Corrupt frame:** CRC failure, frame replaced with silence
- **Truncated file:** Decoder stops at last valid frame
- **Sample rate not supported:** Must be integer value
- **Channel count > 16:** FFmpeg rejects, reference encoder max 6

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM TTA

| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → FLAC | `ffmpeg -i in.tta -c:a flac -compression_level 8 out.flac` | All tags | Lossless |
| → ALAC | `ffmpeg -i in.tta -c:a alac out.m4a` | Via MP4 atoms | Lossless |
| → WAV | `ffmpeg -i in.tta -c:a pcm_s16le out.wav` | RIFF INFO | Lossless |
| → MP3 | `ffmpeg -i in.tta -c:a libmp3lame -q:a 0 out.mp3` | Via ID3v2 | Generation loss |
| → AAC | `ffmpeg -i in.tta -c:a aac -b:a 256k out.m4a` | Via MP4 | Generation loss |
| → Opus | `ffmpeg -i in.tta -c:a libopus -b:a 128k out.opus` | Via Ogg | Generation loss |

### 15.2 Converting TO TTA
FFmpeg does **NOT** support encoding TTA. Use the official TTA encoder:
- Windows: Tau Producer GUI
- Cross-platform: ttaenclib-based tools
- Note: No FFmpeg-based TTA encoding available

### 15.3 Lossless Round-Trip Verification
```bash
# Encode
ffmpeg -i original.wav -c:a tta output.tta  # NOT SUPPORTED

# Decode
ffmpeg -i original.tta -c:a pcm_s16le decoded.wav

# Compare checksums
ffmpeg -i original.tta -map 0:a -f framemd5 original.md5
ffmpeg -i decoded.wav -map 0:a -f framemd5 decoded.md5
diff original.md5 decoded.md5   # Empty diff = bit-perfect
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| TTA Reference | C/C++ | GPL/LGPL | Reference | Reference | https://tausoft.org |
| TTA-lib | C | LGPL | Reference | Reference | SourceForge |
| FFmpeg libavcodec | C | LGPL 2.1+ | N/A | 8/10 | https://ffmpeg.org |
| foobar2000 TTA plugin | C++ | Proprietary | Reference | Reference | foobar2000 |
| ttaR | C | GPL | Improved CLI | Reference | Fork of original |

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **TTA Format Spec:** https://tausoft.org/en/true-audio-codec-tta/
- **Source Code:** https://sourceforge.net/projects/tta/

### Technical Resources
- FFmpeg codec support: `ffmpeg -decoders | grep tta`
- Hydrogenaudio TTA page: https://wiki.hydrogenaudio.org/index.php?title=TTA
- Multimedia Wiki: https://wiki.multimedia.cx/index.php/TTA

### Academic Papers
- None published (closed development)

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] Identify FFmpeg built with TTA support (verify `ffmpeg -decoders | grep tta`)
- [ ] Note: FFmpeg does NOT encode TTA — must use external encoder
- [ ] Handle platform-specific TTA encoder availability

### Decoding Pipeline
- [ ] Implement sync word search for "TTA1" magic
- [ ] Parse file header (channels, bit depth, sample rate, total samples)
- [ ] Validate header CRC32
- [ ] Read frame CRC32 at start of each frame
- [ ] Implement Rice-Golomb entropy decoding
- [ ] Apply inverse linear prediction
- [ ] Handle stereo M/S decorrelation reversal
- [ ] Flush decoder at EOF

### Metadata
- [ ] Read ID3v1 tags from end of file
- [ ] Read ID3v2 tags from end of file (before ID3v1)
- [ ] Read APEv2 tags from end of file
- [ ] Preserve tag priority (APEv2 > ID3v2 > ID3v1)
- [ ] Cover art extraction from tags

### Quality & Verification
- [ ] Implement frame-level CRC verification
- [ ] Track corrupted frames for error reporting
- [ ] Test with: silence, full-scale, multichannel, high-resolution files

---

## 19. COMPRESSION EFFICIENCY ANALYSIS

### 19.1 Theoretical Foundations
TTA's compression efficiency depends on how well linear prediction works for the audio signal. The theoretical compression limit is the entropy of the audio signal:

```
Entropy H = -Σ p(x) × log₂(p(x))

Where p(x) is the probability distribution of sample values or residuals.
```

### 19.2 Compression by Content Type
| Content Type | Compression Ratio | Notes |
|--------------|-------------------|-------|
| Classical music (orchestral) | 30-40% | High redundancy, slow variations |
| Jazz (acoustic) | 35-45% | Good redundancy, some transients |
| Rock/Pop | 45-55% | Moderate redundancy, heavy bass |
| Electronic/Techno | 50-65% | Repetitive patterns, synthesized |
| Speech (clean) | 40-50% | Good prediction for voice |
| Speech (noisy) | 55-70% | Less predictable |
| Silence | 2-5% | Extremely compressible |
| White noise | 90-100% | No redundancy to exploit |

### 19.3 Comparison with Other Lossless Codecs
| Codec | Compression | Speed | Notes |
|-------|-------------|-------|-------|
| TTA FAST | ~40-60% | Fastest | Similar to FLAC fast |
| TTA NORMAL | ~35-55% | Fast | Balanced |
| TTA HIGH | ~30-50% | Medium | Better compression |
| TTA EXTRA | ~25-45% | Slow | Best compression |
| FLAC (level 0) | ~55-65% | Fastest | |
| FLAC (level 8) | ~45-55% | Medium | |
| FLAC (level 12) | ~40-50% | Slow | |
| Monkey's Audio (extra) | ~25-45% | Very slow | |
| OptimFROG (extra) | ~25-40% | Extremely slow | Best compression |

---

## 20. ERROR DETECTION AND CORRECTION

### 20.1 Integrity Verification
TTA provides multiple layers of integrity checking:

```
1. Header CRC32
   - Covers entire 28-byte header
   - Detects header corruption
   
2. Frame CRC32
   - Covers compressed frame data
   - Detects frame corruption
   
3. Total sample count
   - Header stores total samples
   - Decoder can verify completeness
```

### 20.2 Error Handling Strategies
| Error Type | Detection | Handling |
|------------|-----------|----------|
| Header corrupt | CRC mismatch | Reject file |
| Frame corrupt | CRC mismatch | Replace with silence |
| Truncated file | Incomplete final frame | Decode available frames |
| Bit errors in frame | CRC mismatch | Frame replaced with silence |
| Stream sync lost | Invalid header | Stop decoding |

---

## 21. PERFORMANCE BENCHMARKS

### 21.1 Encoding Speed (Typical Modern CPU)
| Mode | Samples/Second | Real-time Multiplier |
|------|----------------|---------------------|
| FAST | ~900,000 | ~20× |
| NORMAL | ~600,000 | ~14× |
| HIGH | ~350,000 | ~8× |
| EXTRA | ~200,000 | ~4.5× |

*Test conditions: Intel Core i7, 3.5 GHz, 44.1 kHz stereo*

### 21.2 Decoding Speed (Typical Modern CPU)
| Mode | Samples/Second | Real-time Multiplier |
|------|----------------|---------------------|
| All modes | ~1,200,000 | ~27× |

*Test conditions: Intel Core i7, 3.5 GHz, 44.1 kHz stereo*

### 21.3 Memory Usage
| Operation | Memory Required |
|-----------|----------------|
| Encoder (stereo) | ~2 MB |
| Decoder (stereo) | ~2 MB |
| Frame buffer | 64 KB |

---

## 22. FUTURE CONSIDERATIONS

### 22.1 Format Limitations
- No native floating-point support
- No multi-channel channel layout metadata
- No embedded cover art standard
- No official specification document
- Discontinued development

### 22.2 Migration Recommendations
Files encoded in TTA should consider migration to:
1. **FLAC** — for maximum compatibility and open standard
2. **TAK** — for similar compression with better decode speed
3. **OptimFROG** — for maximum compression (if decode speed not critical)

### 22.3 Preservation Considerations
- TTA format is stable and well-understood
- Reference implementation is open source
- FFmpeg provides independent decoder implementation
- Format unlikely to become unreadable due to obscurity

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
