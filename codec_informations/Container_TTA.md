# TTA (True Audio) Container — Deep Technical Reference

> **Category:** Container
> **File Extensions:** `.tta`
> **MIME Types:** `audio/x-tta`, `audio/tta`
> **Standardization Body:** None — proprietary format by Tau Software
> **Primary Specification:** Tau Software documentation (tausoft.org)
> **Patent Status:** Patent-free — open specification
> **License:** Free — freely usable for any purpose
> **Current Version:** TTA1 (original format, stable)
> **Active Development:** No — format stable since ~2000

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Tau Software (Russian company), lead developer is the author of the True Audio codec
- **Year Created:** ~2000
- **Original Purpose:** Create a simple, efficient, real-time-capable lossless audio codec with a minimal container format — no unnecessary complexity, just audio data with essential metadata and frame-level integrity checking
- **Problem with Predecessors:** Existing lossless formats (FLAC, WavPack) were either too complex or not optimized for real-time encoding/decoding; TTA was designed specifically for realtime use with fixed frame sizes

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| TTA1 | ~2000 | Initial format — lossless codec with fixed frames, CRC checksums |
| TTA1 (no change) | — | Format has remained stable; no versioning changes needed |

### 1.3 Current Adoption
- **Primary use cases today:** Archival of audio in a simple lossless format, niche audiophile community
- **Platforms with native support:** Limited — most players use FFmpeg for TTA support
- **Major services using this format:** Very few; essentially a niche format
- **Hardware support:** Minimal hardware support outside of software players
- **Status:** Niche / legacy — active development ceased, but format remains stable and usable

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Lossless waveform compression
- **Core algorithm:** Adaptive linear prediction (FIR/IIR filters) combined with entropy coding
- **Loss mechanism:** Lossless only — adaptive prediction + lossless coding
- **Frame-based vs sample-based:** Frame-based — fixed number of samples per frame
- **Fixed vs variable frame size:** Fixed frame size (approximately 1.4 seconds of audio per frame)

### 2.2 High-Level Encoding Flow (Block Diagram)
```
Input PCM Samples (stereo or mono)
      │
      ▼
[Pre-processing: Split into channel frames]
      │
      ▼
[Frame Buffering: Accumulate N samples per channel]
      │
      ▼
[Linear Prediction: Adaptive FIR filter estimation]
      │
      ▼
[Entropy Coding: Range coder for residuals]
      │
      ▼
[Frame CRC: 32-bit CRC for integrity]
      │
      ▼
[Seek Point: Record frame data length]
      │
      ▼
Output Encoded Frame (repeat for all frames)
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | ~1.4 seconds (one frame) | Enables real-time operation |
| Frame size | 65536 samples (at 48 kHz = 1.37 sec) | Fixed regardless of sample rate |
| Max channels | 2 (stereo) | Format supports mono and stereo only |
| Max bit depth | 16-bit, 24-bit | Defined per file |
| Max sample rate | 192 kHz (theoretically) | Frame size in samples is fixed |
| Bitrate range | ~500–1400 kbps | Depends on source content and bit depth |
| Complexity | O(n) linear | Encoder designed for speed |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       54 54 41 31     'TTA1'   TTA format magic + version
```

The magic bytes `TTA1` identify this as a TTA format file. The `1` indicates the format version number.

### 3.2 File-Level Header Layout
```
Full byte-map of the TTA1 file header (22 bytes total):
Offset   Size    Field Name              Type        Valid Range   Description
-------  ------  ----------------------  ----------  -----------   ---------------------------
0x0000   4B      Format Signature        char[4]     'TTA1'       Magic bytes, version indicator
0x0004   2B      Audio Format            uint16 LE   0x0001       Format identifier (always 1)
0x0006   2B      Number of Channels      uint16 LE   1–2          1=mono, 2=stereo
0x0008   2B      Bits Per Sample        uint16 LE   16 or 24     Bit depth of audio
0x000A   4B      Sample Rate             uint32 LE   8000–192000  Samples per second
0x000E   4B      Total Samples           uint32 LE   > 0          Total samples per channel
0x0012   4B      Header CRC32            uint32 LE   CRC-32       CRC-32 of bytes 0x00–0x11
```

### 3.3 Frame Header Layout (No Explicit Frame Header)
TTA frames do not have a dedicated header. Each frame is preceded by a 4-byte frame data length stored in the seek table. The frame data itself contains compressed audio, followed by a 4-byte frame CRC.

```
Per-Frame Data Layout:
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------
0x0000   N       Frame Data             byte[]      Compressed audio data
N        4B      Frame CRC32             uint32 LE   CRC-32 of frame data bytes
```

The frame CRC covers only the compressed frame data, not the frame length itself.

### 3.4 Seek Table Layout
```
Location: Immediately after the 22-byte file header
Entry count: equal to total number of frames

Each seek point entry:
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------
0x0000   4B      Frame Data Length       uint32 LE   Length of compressed frame data in bytes

The seek table is followed by a 4-byte CRC:
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------
0x0000   4B      Seek Table CRC32        uint32 LE   CRC-32 of the entire seek table
```

### 3.5 Complete File Layout
```
File Layout:
[0x0000] 22-byte file header
           ├── 'TTA1' signature (4 bytes)
           ├── Audio format (2 bytes)
           ├── Channel count (2 bytes)
           ├── Bits per sample (2 bytes)
           ├── Sample rate (4 bytes)
           ├── Total samples (4 bytes)
           └── Header CRC32 (4 bytes)
           
[0x0016] Seek Table (total_frames × 4 bytes)
           ├── Frame 0 data length (4 bytes)
           ├── Frame 1 data length (4 bytes)
           ├── ...
           └── Frame N-1 data length (4 bytes)
           
           Seek Table CRC32 (4 bytes)

[0x0016 + N×4 + 4] Audio Frames (repeated for each frame)
           For each frame:
           ├── Frame data (variable bytes, length from seek table)
           └── Frame CRC32 (4 bytes)

[End of file] Last frame CRC32

Total file structure:
  File Header:      22 bytes
  Seek Table:       total_frames × 4 bytes + 4 bytes CRC
  Audio Frames:     sum of all frame data + (total_frames × 4 bytes CRC)
```

### 3.6 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Unsigned integer | No | TTA supports 16-bit and 24-bit only |
| 16-bit | Signed integer | Yes | Primary format |
| 20-bit | Signed integer | No | Not officially supported |
| 24-bit | Signed integer | Yes | Secondary format |
| 32-bit | Signed integer | No | Not supported |
| 32-bit | IEEE float | No | Not supported |

#### Supported Sample Rates
| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 8000 | Telephone | Yes | Frame = 8 sec (65536 samples) |
| 11025 | — | Yes | Frame = 5.94 sec |
| 16000 | Wideband voice | Yes | Frame = 4.1 sec |
| 22050 | — | Yes | Frame = 2.97 sec |
| 32000 | Broadcast | Yes | Frame = 2.05 sec |
| 44100 | CD audio | Yes | Frame = 1.49 sec |
| 48000 | Professional | Yes | Frame = 1.37 sec |
| 88200 | 2× CD | Yes | Frame = 0.74 sec |
| 96000 | High-res | Yes | Frame = 0.68 sec |
| 176400 | 4× CD | Yes | Frame = 0.37 sec |
| 192000 | High-res max | Yes | Frame = 0.34 sec |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Pre-Processing Stage
- **Channel separation:** Left and right channels are encoded independently
- **Sample alignment:** All samples within a frame are encoded together
- **No pre-emphasis:** TTA does not apply any pre-emphasis filter
- **No DC offset removal:** Not performed by TTA encoder

### 4.2 Analysis / Transform Stage

#### Transform Type: None (Time-Domain Prediction)
```
TTA does not use a frequency-domain transform.
Instead, it uses adaptive linear prediction in the time domain.

Parameters:
  Prediction order:    Adaptive (determined by encoder)
  Window size:         Equal to frame size (65536 samples)
  Overlap:             None (frame-based)
  Window function:     None
```

**Linear Prediction Model:**
```
For each sample x[n], the encoder predicts:
  x̂[n] = Σ(i=1 to order) a[i] × x[n-i]
  
The residual r[n] = x[n] - x̂[n] is encoded losslessly.

The prediction coefficients a[i] are adaptively updated per frame.
```

### 4.3 No Psychoacoustic Model
TTA is a lossless codec and does not use psychoacoustic modeling. There is no quality setting — the output is always bit-exact.

### 4.4 No Quantization Stage
Since TTA is lossless, no quantization occurs. The residuals are encoded using entropy coding without any lossy approximation.

### 4.5 Stereo Encoding Modes
| Mode | Description | Condition for Selection | Notes |
|------|-------------|------------------------|-------|
| Independent | L and R encoded independently | Default | Both channels use same predictor |
| No joint stereo | No M/S encoding | Not applicable | TTA does not use joint stereo |

### 4.6 Entropy Coding
```
Method: Range coder (also called arithmetic coding)

The range coder processes the residual signal from the linear predictor.
It uses a table-driven approach for symbol probability estimation.

Key properties:
  - Adaptive probability modeling
  - Variable-length output
  - Near-optimal compression for the given probability model
```

### 4.7 Encoder Settings / Quality Modes

#### For Lossless Codec
| Quality Setting | Encoding Speed | Decode Speed | File Size | Notes |
|----------------|---------------|--------------|-----------|-------|
| Single mode | ~1× realtime | ~1× realtime | Minimum | Only mode |

TTA has no quality/compression levels — it is exclusively lossless.

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. Read first 22 bytes (file header)
2. Validate 'TTA1' magic at offset 0
3. Validate header CRC32 at bytes 18-21
4. Validate audio format = 1
5. Compute total frames from:
     total_frames = ceil(total_samples / 65536)
6. Read seek table: 4 bytes × total_frames
7. Validate seek table CRC
8. Frame positions are computed cumulatively:
     frame_position[i] = header_size + seek_table_size + sum(frame_data_length[j] for j < i)
```

#### Seeking
```
TTA provides efficient seeking through the seek table.
Each seek point entry contains the frame data length (in bytes).

Seeking to a specific time:
1. Calculate target frame: frame = target_sample / 65536
2. Look up frame position from cumulative sum of seek table entries
3. Skip to frame position in file
4. Read frame data and decode

Seek precision: Frame-accurate (within one frame = ~1.4 seconds at 48 kHz)
```

### 5.2 Core Decode Pipeline
```
1. Read and validate 22-byte file header
   ├── Verify 'TTA1' signature
   ├── Extract: channels, bits_per_sample, sample_rate, total_samples
   └── Verify header CRC32

2. Read seek table
   ├── Compute total_frames = (total_samples + 65535) / 65536
   ├── Read 4 bytes × total_frames for frame lengths
   └── Verify seek table CRC32

3. For each frame (i = 0 to total_frames - 1):
   ├── Read frame data (length from seek_table[i])
   ├── Verify frame CRC32
   ├── Decode range coder output to get residuals
   ├── Reconstruct prediction coefficients
   ├── Apply linear prediction: x[n] = x̂[n] + r[n]
   └── Output 65536 samples (or fewer for last frame)

4. Post-processing
   └── Format samples as PCM (16-bit or 24-bit signed integer)
```

### 5.3 Error Concealment
- **Corrupt frame detection:** CRC-32 check on each frame
- **Concealment method:** None — corrupt frame causes decode error
- **Maximum consecutive errors before silence:** 1 (decode stops on first error)

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** TTA — standalone file format, no external container
- **Overhead:** ~4 bytes per frame (seek table entry + CRC) + 22 byte header
- **Seeking in native container:** Yes — seek table provides direct frame access
- **Multiple streams in native container:** No — single audio stream only

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| TTA (.tta) | TTA audio | Yes (seek table) | None | Native format |
| OGG (.oga) | TTA via OGG container | Yes | Vorbis Comments | OGG wrapping |
| Matroska/MKA | No native support | — | — | No known support |
| MP4/M4A | No | — | — | Not supported |
| WAV | No | — | — | Raw PCM only |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** None — TTA has no native metadata support
- **Tag block location:** N/A — no metadata in TTA file
- **External storage:** Metadata handled by filename or external database

### 7.2 Standard Tag Fields — Not Supported in TTA
| Tag Field | Supported | Notes |
|-----------|-----------|-------|
| Title | No | Use external database |
| Artist | No | Use external database |
| Album | No | Use external database |
| All standard tags | No | TTA has no metadata support |

### 7.3 Cover Art Storage
```
Cover art: Not supported in TTA format
Workaround: Use separate cover file, or convert to FLAC with embedded art
```

### 7.4 Metadata Compatibility Matrix
| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| Native TTA | ✗ | ✗ | N/A | N/A |
| ID3v1 (before TTA) | ✗ | ✗ | N/A | N/A |
| ID3v2 (before TTA) | ✗ | ✗ | N/A | N/A |
| APEv2 (after TTA) | Some players | Some players | Not native | N/A |

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   tta           # used with -c:a
AV_CODEC_ID:        AV_CODEC_ID_TTA  # C constant in libavcodec/codec_id.h
Format Name (CLI):  tta           # used with -f
Encoder(s):         None          # FFmpeg does NOT encode TTA natively
Decoder(s):         tta           # FFmpeg has TTA decoder
Muxer(s):          tta           # FFmpeg has TTA muxer
Demuxer(s):        tta           # FFmpeg has TTA demuxer
```

### 8.2 FFmpeg Encoding — Not Supported
```bash
# FFmpeg cannot encode TTA natively
# TTA encoding requires third-party tools:
#   - Tau Software TTA encoder (Windows)
#   - ttaenc (command-line encoder)
#   - Use FFmpeg for decoding only
```

### 8.3 FFmpeg Decoding — Full CLI Reference

```bash
# Decode TTA to WAV (lossless)
ffmpeg -i input.tta \
  -c:a pcm_s16le \
  output.wav

# Decode TTA to FLAC (lossless to lossless)
ffmpeg -i input.tta \
  -c:a flac -compression_level 8 \
  output.flac

# Decode TTA to any format
ffmpeg -i input.tta \
  -c:a libmp3lame -q:a 2 \
  output.mp3

# Probe format details
ffprobe -v quiet -print_format json -show_streams -show_format input.tta
```

### 8.4 FFmpeg Decoding — libavformat C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/opt.h>

// ─── Complete decode program skeleton ────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.tta", NULL, NULL);
avformat_find_stream_info(fmt_ctx, NULL);

// Find audio stream
int audio_idx = av_find_best_stream(fmt_ctx, AVMEDIA_TYPE_AUDIO, -1, -1, NULL, 0);
AVStream *stream = fmt_ctx->streams[audio_idx];

// Codec context is pre-populated by demuxer
AVCodecContext *dec_ctx = avcodec_alloc_context3(NULL);
avcodec_parameters_to_context(dec_ctx, stream->codecpar);

// Find TTA decoder
const AVCodec *dec = avcodec_find_decoder(stream->codecpar->codec_id);
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
# Read metadata (TTA has none)
ffprobe -v quiet -print_format json -show_format input.tta
# Output: format_name=tta, format_long_name="TTA (True Audio)"

# TTA files typically have no metadata
# Metadata must be stored externally or in filename
```

### 8.6 Quality / Fidelity Decision Table
| Use Case | Settings | Expected Size | Notes |
|----------|----------|---------------|-------|
| Archival | Lossless | 50–70% of WAV | TTA is lossless |
| Conversion intermediate | TTA → any | Lossless decode path | Good intermediate format |
| Streaming | Not ideal | N/A | Large frame size not optimal |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
```
TTA Seek Table:
  Location:    Immediately after 22-byte file header
  Magic:      N/A (no separate magic, follows header directly)
  Entry size: 4 bytes per frame
  Entry format:
    [0x00–0x03]  frame_data_length (uint32 LE)
  
  Max entries: ceil(total_samples / 65536)
  
  Seek Table is followed by a 4-byte CRC32:
    [0x00–0x03]  seek_table_crc (uint32 LE)
```

### 9.2 Gapless Playback Data
```
TTA does not store gapless playback data.
Encoder delay:   0 samples (no lookahead)
Padding:         0 samples (frame boundary aligned)
Storage:         None
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | Fixed frame size aids streaming |
| Algorithmic encoder delay | ~1.4 seconds | One frame delay |
| Live encoding feasible | Yes | Designed for real-time |
| HTTP progressive download | Yes | Seek table enables partial download |
| HTTP Live Streaming (HLS) | No | Not segmented |
| WebRTC / RTP transport | No | Not typical transport |
| Minimum decode buffer | 1 frame | ~1.4 seconds |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | Layout Name | Channel Order | FFmpeg Layout Constant | Notes |
|----------|-------------|---------------|----------------------|-------|
| 1 | Mono | C | AV_CHANNEL_LAYOUT_MONO | Supported |
| 2 | Stereo | L, R | AV_CHANNEL_LAYOUT_STEREO | Supported |
| >2 | Not supported | — | — | TTA limits to 2 channels |

### 11.2 Downmix Coefficients
```
TTA does not support multichannel, so no downmix is needed.
Mono TTA files: 1 channel
Stereo TTA files: 2 channels (L, R)
```

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 24-bit | TTA supports 16-bit and 24-bit |
| Max sample rate | 192,000 Hz | Frame size in samples is fixed |
| Float support | No | Integer PCM only |
| DSD support | No | Not applicable |
| 24-bit support | Yes | Fully supported |
| 20-bit support | No | Not officially supported |

---

## 13. HARDWARE ACCELERATION & PLATFORM NOTES

| Platform / API | Encode | Decode | FFmpeg Flag | Notes |
|----------------|--------|--------|-------------|-------|
| Native | Limited | Limited | N/A | Few players have native TTA |
| FFmpeg | No | Yes | Built-in | Most reliable decode path |
| Software-only | Yes | Yes | N/A | No hardware TTA support known |

---

## 14. KNOWN ISSUES, BUGS & EDGE CASES

### 14.1 FFmpeg Encoder-Specific Issues
| Issue | FFmpeg Versions Affected | Workaround |
|-------|--------------------------|------------|
| No encoder | All | Use Tau Software encoder or third-party |

### 14.2 Interoperability Issues
- **TTA metadata:** No metadata support means album/track info must come from filename or external tags
- **Multichannel:** Only mono and stereo supported
- **24-bit audio:** Some older decoders only support 16-bit

### 14.3 Edge Cases to Handle in Converter
- **Empty file / 0-sample stream:** Return error
- **File < 1 frame of audio:** Valid but unusual
- **Corrupt frame CRC:** Report error, do not output audio
- **Truncated file:** Decode available frames, report incomplete
- **Sample rate not in table:** TTA supports a wide range; most should work

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM TTA

| Target | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| → FLAC | `ffmpeg -i in.tta -c:a flac -compression_level 8 out.flac` | No (no metadata) | Lossless |
| → WAV | `ffmpeg -i in.tta -c:a pcm_s16le out.wav` | No | Lossless decode |
| → ALAC | `ffmpeg -i in.tta -c:a alac out.m4a` | No | Lossless |
| → MP3 | `ffmpeg -i in.tta -c:a libmp3lame -q:a 0 out.mp3` | No | Generation loss |
| → AAC | `ffmpeg -i in.tta -c:a aac -b:a 256k out.m4a` | No | Re-encode |

### 15.2 Converting TO TTA

| Source | FFmpeg Command | Metadata Preserved | Quality Notes |
|--------|---------------|-------------------|---------------|
| FLAC → | Use Tau encoder | No | Lossless |
| WAV → | Use Tau encoder | No | Lossless |
| MP3 → | Not recommended | No | Generation loss |

### 15.3 Lossless Round-Trip Verification
```bash
# Decode TTA to WAV
ffmpeg -i input.tta -c:a pcm_s16le decoded.wav

# Compare checksums
md5sum decoded.wav

# Verify bit-exact decoding
ffmpeg -i input.tta -c:a pcm_s16le -f md5 -
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| TTA reference encoder | C | Free | Reference | Reference | tausoft.org |
| FFmpeg TTA decoder | C | LGPL 2.1+ | — | Good | ffmpeg.org |
| FFmpeg TTA muxer | C | LGPL 2.1+ | — | — | ffmpeg.org |
| ttaenc (Linux) | C | GPL | Reference | Reference | sourceforge |

### Build Instructions
```bash
# Build FFmpeg with TTA support
./configure --enable-decoder=tta --enable-demuxer=tta --enable-muxer=tta
make -j$(nproc)
make install
```

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **TTA Format:** https://tausoft.org/en/tta-%d0%be%d0%bf%d0%b8%d1%81%d0%b0%d0%bd%d0%b8%d0%b5-%d1%84%d0%be%d1%80%d0%bc%d0%b0%d1%82%d0%b0/
- **TTA Codec:** https://tausoft.org/en/true-audio-%d0%ba%d0%be%d0%b4%d0%b5%d0%ba-tta/

### Technical Resources
- FFmpeg decoder: `ffmpeg -decoders | grep tta`
- FFmpeg muxer: `ffmpeg -muxers | grep tta`
- Multimedia Wiki: https://wiki.multimedia.cx/index.php?title=TTA

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] Identify FFmpeg `./configure` flags: `--enable-decoder=tta --enable-muxer=tta`
- [ ] Verify `ffmpeg -decoders` output confirms TTA decoder is available
- [ ] Note: FFmpeg does NOT have a TTA encoder — use external tool if needed

### Encoding Pipeline
- [ ] TTA encoding requires third-party encoder (Tau Software)
- [ ] FFmpeg cannot be used for TTA encoding
- [ ] Configure external TTA encoder if encoding support needed

### Decoding Pipeline
- [ ] Read 22-byte file header and validate 'TTA1' magic
- [ ] Extract channel count, bit depth, sample rate, total samples
- [ ] Read seek table and validate CRC
- [ ] Decode each frame using TTA range decoder
- [ ] Verify frame CRC32 before outputting samples

### Metadata
- [ ] TTA has no native metadata support
- [ ] Metadata must be handled externally (filename, database)
- [ ] Consider storing metadata in companion file

### Quality & Verification
- [ ] TTA is lossless — verify bit-exact decode
- [ ] Test with 16-bit and 24-bit TTA files
- [ ] Test seeking at various positions

### Edge Cases
- [ ] Handle files with fewer samples than one frame
- [ ] Handle corrupt frame CRC gracefully
- [ ] Handle files with unexpected sample rates

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
