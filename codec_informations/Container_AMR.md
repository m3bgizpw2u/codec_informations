# AMR (Adaptive Multi-Rate) Container — Deep Technical Reference

> **Category:** Container
> **File Extensions:** `.amr`, `.awb`, `.3gp`, `.3g2`
> **MIME Types:** `audio/amr`, `audio/amr-wb`, `audio/3gpp`, `audio/3gpp2`
> **Standardization Body:** ETSI (European Telecommunications Standards Institute)
> **Primary Specification:** 3GPP TS 26.090 (AMR-NB), 3GPP TS 26.190 (AMR-WB), RFC 4867
> **Patent Status:** Patented — 3GPP patent pool
> **License:** Royalty-bearing for commercial implementations
> **Current Version:** AMR-NB (stable), AMR-WB (stable), AMR-WB+ (discontinued)
> **Active Development:** No — standards finalized; only bug fixes

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** ETSI (European Telecommunications Standards Institute), adopted by 3GPP
- **Year Created:** 1998 (AMR-NB); 2001 (AMR-WB)
- **Original Purpose:** Create a speech codec for GSM/UMTS mobile networks that adapts its bitrate based on channel conditions — using more bits when the radio link is good, fewer when it's poor
- **Problem with Predecessors:** Fixed-rate codecs (GSM-EFR, PDC-EFR) couldn't adapt to varying radio conditions; AMR solved this by allowing the network to dynamically select the best mode

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| AMR-NB v1.0 | 1998 | Initial narrowband speech codec, 8 modes |
| AMR-NB v2.0 | 1999 | Refinements, improved error concealment |
| AMR-WB v1.0 | 2001 | Wideband extension, 9 modes, 16 kHz |
| AMR-WB+ v1.0 | 2004 | Extended wideband, super-wideband to 32 kHz |
| AMR-WB+ discontinued | 2012 | EVS codec superseded AMR-WB+ |

### 1.3 Current Adoption
- **Primary use cases today:** Mobile voice calls, VoLTE, VoNR, voicemail, MMS audio attachments
- **Platforms with native support:** All mobile phones since 2000, VoIP applications
- **Major services using this format:** Mobile carriers worldwide, WhatsApp voice messages
- **Hardware support:** Universal in mobile SoCs
- **Status:** Ubiquitous — dominant voice codec for mobile networks

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Speech coding (parametric)
- **Core algorithm:** ACELP (Algebraic Code Excited Linear Prediction) with additional modes
- **Loss mechanism:** Lossy — optimized for speech, not music
- **Frame-based vs sample-based:** Frame-based — fixed 20 ms frames
- **Fixed vs variable frame size:** Fixed — 160 samples (8 kHz NB) or 320 samples (16 kHz WB)

### 2.2 High-Level Encoding Flow (Block Diagram)
```
Input PCM Samples (16-bit, 8 kHz NB / 16 kHz WB)
      │
      ▼
[Pre-processing: Pre-emphasis, windowing]
      │
      ▼
[LP Analysis: Linear Prediction (10th order NB, 16th order WB)]
      │
      ▼
[Open-loop Pitch Analysis]
      │
      ▼
[Closed-loop Codebook Search]
      │
      ├── Adaptive Codebook (pitch)
      └── Fixed Algebraic Codebook (innovation)
      │
      ▼
[Quantization: Scalar + Vector quantization]
      │
      ▼
[Bitstream Packing: Frame header + speech bits]
      │
      ▼
Output AMR Frame (13–31 bytes)
```

### 2.3 Key Design Parameters
| Parameter | AMR-NB | AMR-WB | Notes |
|-----------|--------|--------|-------|
| Sample rate | 8,000 Hz | 16,000 Hz | |
| Frame duration | 20 ms | 20 ms | |
| Samples per frame | 160 | 320 | |
| Bitrates | 4.75–12.2 kbps | 6.6–23.85 kbps | 8 modes (NB), 9 modes (WB) |
| LP order | 10th order | 16th order | |
| Algorithmic delay | 5 ms lookahead + 5 ms frame | ~25 ms total | |
| Complexity (MOS) | ~5 MIPS | ~15 MIPS | |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
```
Raw AMR file has no magic bytes.
File type is identified by extension (.amr).

3GPP/3GPP2 container uses ISOBMFF (see Container_3GP_3GPP.md).
```

### 3.2 Raw AMR Frame Layout — IF2 Format (Octet-Aligned)

#### AMR-NB Frame (IF2 — Interface Format 2)
```
Offset   Bytes   Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------
0x0000   1       Frame Type + Padding    uint8       4-bit frame type + QI bit + padding
0x0001   N       Speech bits            byte[]      Class A + B + C bits, padded to byte
```

#### AMR Frame Type Table (NB and WB)
| Frame Type | AMR-NB Content | AMR-WB Content | Total Bytes (NB) | Total Bytes (WB) |
|------------|---------------|----------------|-----------------|------------------|
| 0 | AMR 4.75 kbps | AMR-WB 6.60 kbps | 13 | 17 |
| 1 | AMR 5.15 kbps | AMR-WB 8.85 kbps | 14 | 19 |
| 2 | AMR 5.90 kbps | AMR-WB 12.65 kbps | 16 | 23 |
| 3 | AMR 6.70 kbps | AMR-WB 14.25 kbps | 18 | 27 |
| 4 | AMR 7.40 kbps | AMR-WB 15.85 kbps | 19 | 30 |
| 5 | AMR 7.95 kbps | AMR-WB 18.25 kbps | 21 | 32 |
| 6 | AMR 10.20 kbps | AMR-WB 19.85 kbps | 26 | 37 |
| 7 | AMR 12.20 kbps | AMR-WB 23.05 kbps | 31 | 40 |
| 8 | AMR SID (CNG) | AMR-WB SID | 6 | 10 |
| 9 | GSM-EFR SID | AMR-WB SID (2) | 6 | 9 |
| 10 | TDMA-EFR SID | AMR-WB SID (3) | 6 | 9 |
| 11 | PDC-EFR SID | Reserved | 6 | 9 |
| 14 | SPEECH_LOST | SPEECH_LOST | — | — |
| 15 | No Data | No Data | 1 | 1 |

### 3.3 AMR-NB Bit Allocation by Mode

#### IF2 Frame Bit Layout (All modes byte-aligned)
| Mode (kbps) | Frame Type | Total Bits | Class A Bits | Class B Bits | Class C Bits | Octets |
|--------------|-----------|-----------|-------------|-------------|-------------|--------|
| 4.75 | 0 | 95 | 42 | 53 | 0 | 13 |
| 5.15 | 1 | 103 | 49 | 54 | 0 | 14 |
| 5.90 | 2 | 118 | 55 | 63 | 0 | 16 |
| 6.70 | 3 | 134 | 58 | 76 | 0 | 18 |
| 7.40 | 4 | 148 | 61 | 87 | 0 | 19 |
| 7.95 | 5 | 159 | 75 | 84 | 0 | 21 |
| 10.20 | 6 | 204 | 65 | 99 | 40 | 26 |
| 12.20 | 7 | 244 | 81 | 103 | 60 | 31 |
| SID | 8 | 39 | 39 | 0 | 0 | 6 |

#### Class Bit Priority
- **Class A:** Most important (pitch, LSP, gains) — protected by CRC
- **Class B:** Important (innovation shape)
- **Class C:** Less important (innovation gain)

### 3.4 AMR-WB Bit Allocation by Mode

| Mode (kbps) | Frame Type | Total Bits | Octets |
|--------------|-----------|-----------|--------|
| 6.60 | 0 | 132 | 17 |
| 8.85 | 1 | 177 | 23 |
| 12.65 | 2 | 253 | 32 |
| 14.25 | 3 | 285 | 36 |
| 15.85 | 4 | 317 | 40 |
| 18.25 | 5 | 365 | 46 |
| 19.85 | 6 | 397 | 50 |
| 23.05 | 7 | 461 | 58 |
| 23.85 | — | 477 | 60 |
| SID | 8 | 80 | 10 |

### 3.5 Raw AMR File Structure
```
Raw AMR File (.amr):
[Signature: "#!AMR\n"] (5 bytes, optional header for players)
[Frame 1: 13-31 bytes]
[Frame 2: 13-31 bytes]
[Frame 3: 13-31 bytes]
...
[ID3v1 tag: 128 bytes, optional, at end]

Total file: Frame size × number of frames + optional header
```

### 3.6 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Unsigned integer | No | AMR uses 16-bit PCM internally |
| 16-bit | Signed integer | Yes | Input/output format |
| 24-bit | Signed integer | No | Not supported |
| 32-bit | Signed integer | No | Not supported |
| 32-bit | IEEE float | No | Not supported |

#### Supported Sample Rates
| Rate (Hz) | AMR-NB | AMR-WB | Notes |
|-----------|--------|--------|-------|
| 8000 | Yes | No | AMR-NB native rate |
| 16000 | No | Yes | AMR-WB native rate |
| 11025 | No | No | Not supported |
| 22050 | No | No | Not supported |
| 44100 | No | No | Not supported |
| 48000 | No | No | Not supported |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Pre-Processing Stage
- **Pre-emphasis filter:** H(z) = 1 - 0.68 × z^(-1) (NB), similar for WB
- **DC offset removal:** Implicit in LP analysis
- **Windowing:** 30 ms window ( Hamming ) for LP analysis, centered on frame
- **Look-ahead:** 5 ms (40 samples NB, 80 samples WB)

### 4.2 LP Analysis
```
AMR-NB:
  LP order:        10th order
  Algorithm:       Levinson-Durbin recursion
  Quantization:    Split vector quantization (SVQ) of LSP coefficients
  LSP to LP:       Conversion for each subframe

AMR-WB:
  LP order:        16th order
  Algorithm:       Levinson-Durbin recursion
  Quantization:    Two-stage SVQ with prediction
```

### 4.3 Per-Frame Encoding (ACELP)
```
Each 20 ms frame is divided into 4 subframes (5 ms each):

For each subframe:
  1. Open-loop pitch analysis (every 2nd subframe)
  2. Closed-loop pitch search (adaptive codebook)
  3. Algebraic codebook search (fixed codebook)
  4. Quantization of gains
  5. Memory update
```

### 4.4 Adaptive Codebook (Pitch)
```
AMR-NB pitch delay range:  18–145 samples (2.25–18.125 ms)
AMR-WB pitch delay range:  34–231 samples (2.125–14.4 ms)

Search method:  Focused search with limited range around OL pitch estimate
```

### 4.5 Algebraic Codebook Structure
```
AMR-NB: 17-bit algebraic codebook
  - 4 pulses per track, 2 tracks
  - Each pulse can have ±1 amplitude
  - Positions constrained to allow fast search

AMR-WB: 17-bit (lower rates) to 33-bit (higher rates) algebraic codebook
  - Variable pulse count per mode
  - More pulses at higher bitrates
```

### 4.6 Encoder Modes / Bitrates

#### AMR-NB Modes
| Mode | Bitrate | Primary Use | Typical Application |
|------|---------|-------------|--------------------|
| 0 | 4.75 kbps | Poor conditions | Congested cells |
| 1 | 5.15 kbps | Poor-good | Adaptive |
| 2 | 5.90 kbps | Moderate | Standard |
| 3 | 6.70 kbps | Good | Higher quality |
| 4 | 7.40 kbps | Good | TDMA-EFR replacement |
| 5 | 7.95 kbps | Very good | Higher quality |
| 6 | 10.20 kbps | Very good | Higher quality |
| 7 | 12.20 kbps | Best | GSM-EFR replacement |

#### AMR-WB Modes
| Mode | Bitrate | Primary Use | Bandwidth |
|------|---------|-------------|-----------|
| 0 | 6.60 kbps | Poor | Wideband |
| 1 | 8.85 kbps | Moderate | Wideband |
| 2 | 12.65 kbps | Good | Wideband |
| 3 | 14.25 kbps | Good | Wideband |
| 4 | 15.85 kbps | Very good | Wideband |
| 5 | 18.25 kbps | Very good | Wideband |
| 6 | 19.85 kbps | Very good | Wideband |
| 7 | 23.05 kbps | Best | Wideband |
| 8 | SID | Comfort noise | — |

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization

#### Sync Strategy for Raw AMR
```
1. Optional: Read "#!AMR\n" header (5 bytes)
2. For each frame:
     a. Read first byte: extract frame type (4 bits)
     b. Read remaining bytes based on frame type
     c. Validate frame type is in valid range (0–15)
     d. Decode speech parameters from bits
3. End of file or ID3v1 tag (128 bytes)
```

### 5.2 Core Decode Pipeline
```
1. Read frame header (first byte or parsed from container)
     ├── Extract frame type (4 bits)
     ├── Extract quality indicator (1 bit)
     └── Determine frame size from frame type table

2. Read frame data
     ├── Read Class A bits (speech class A parameters)
     ├── Read Class B bits (innovation bits)
     ├── Read Class C bits (innovation gain, lower modes)
     └── Byte-align for next frame

3. Decode LSP coefficients (Class A)
     └── Depacketize, interpolate with previous frame

4. Decode pitch (adaptive codebook)
     ├── Extract pitch delay and gain
     └── Build excitation

5. Decode innovation (fixed codebook)
     ├── Extract algebraic codebook indices
     ├── Reconstruct pulse positions and signs
     └── Apply gain

6. Build excitation
     ├── excitation = adaptive + fixed × gain

7. Synthesis filtering
     ├── LP synthesis filter: H(z) = 1 / A(z)
     └── 10th order (NB) or 16th order (WB)

8. Post-processing
     ├── Adaptive post-filter (NB: pitch sharpening + formants)
     ├── High-pass filter (NB: 80 Hz cutoff)
     └── De-emphasis (NB)

9. Output 160 (NB) or 320 (WB) samples
```

### 5.3 Error Concealment
- **Corrupt frame detection:** CRC on Class A bits
- **Concealment method:** Repetition of previous frame, with attenuation
- **Maximum consecutive errors:** Undefined — decoder degrades gracefully
- **SID frames:** Used to update comfort noise parameters during DTX

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** Raw bitstream (.amr) — no structure
- **Overhead:** ~1 byte per frame (frame type indicator)
- **Seeking:** No — must scan frame by frame
- **Multiple streams:** No

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| Raw AMR (.amr) | AMR-NB | No | ID3v1 | Simple bitstream |
| Raw AWB (.awb) | AMR-WB | No | ID3v1 | Simple bitstream |
| 3GP (.3gp) | AMR-NB, AMR-WB | Yes | 3GP metadata | ISOBMFF |
| 3G2 (.3g2) | AMR-NB, AMR-WB | Yes | 3GP metadata | ISOBMFF |
| MP4/M4A | AMR-NB, AMR-WB | Yes | iTunes atoms | Via esds box |
| Matroska/MKA | AMR-NB, AMR-WB | Yes | Vorbis Comments | Via webm |
| AMR-WB+ in MP4 | AMR-WB+ | Yes | iTunes atoms | Discontinued |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** None for raw AMR; 3GP metadata for containerized AMR
- **Tag block location:** N/A for raw AMR
- **External storage:** ID3v1 at end of file (optional)

### 7.2 Standard Tag Fields — AMR in 3GP
| Tag Field | Internal Key | Max Length | Notes |
|-----------|-------------|------------|-------|
| Title | ©nam | variable | |
| Artist | ©ART | variable | |
| Album | ©alb | variable | |
| All standard fields | Via 3GP metadata | — | See Container_3GP_3GPP.md |

### 7.3 Cover Art Storage
```
Raw AMR files: No cover art support
AMR in 3GP: Cover art via 'covr' atom (see Container_3GP_3GPP.md)
```

### 7.4 Metadata Compatibility Matrix
| Tag System | AMR (.amr) | AMR in 3GP | Priority |
|------------|------------|------------|----------|
| Native | ✗ | ✓ | Highest |
| ID3v1 | ✓ (end) | ✗ | Low |
| ID3v2 | No | No | N/A |
| APEv2 | No | No | N/A |

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   amr_nb (AMR narrow-band)
                    amr_wb (AMR wide-band)
                    libopencore_amrnb (OpenCore AMR-NB)
                    libopencore_amrwb (OpenCore AMR-WB)
AV_CODEC_ID:        AV_CODEC_ID_AMR_NB
                    AV_CODEC_ID_AMR_WB
Format Name (CLI):  amr (raw AMR-NB)
                    amrwb (raw AMR-WB)
                    3gp (3GP container)
Encoder(s):         amr_nb (limited — requires libopencore-amrnb)
                    amr_wb (limited — requires libopencore-amrwb)
Decoder(s):         amr_nb, amr_wb
Muxer(s):          amr, amrnb, amrwb, 3gp
Demuxer(s):        amr, amrnb, amrwb, 3gp, 3g2
```

### 8.2 FFmpeg Encoding — Full CLI Reference

```bash
# Encode to AMR-NB (requires libopencore-amrnb)
ffmpeg -i input.wav \
  -ac 1 \
  -ar 8000 \
  -c:a libopencore_amrnb \
  -b:a 12.2k \
  -f amr \
  output.amr

# Encode to AMR-WB (requires libopencore-amrwb)
ffmpeg -i input.wav \
  -ac 1 \
  -ar 16000 \
  -c:a libopencore_amrwb \
  -b:a 23.85k \
  -f amrwb \
  output.awb

# Encode to 3GP with AMR audio
ffmpeg -i input.wav \
  -ac 1 \
  -ar 8000 \
  -c:a libopencore_amrnb \
  -f 3gp \
  output.3gp
```

#### Complete FFmpeg Option Table
| Option | Type | Default | Range / Valid Values | Effect on Output |
|--------|------|---------|---------------------|-----------------|
| `-b:a` | int | varies | 4.75k–12.2k (NB), 6.6k–23.85k (WB) | Target bitrate |
| `-ar` | int | 8000/16000 | 8000 (NB), 16000 (WB) | Sample rate |
| `-ac` | int | 1 | 1 | Channels (mono only) |
| `-f amr` | flag | — | — | Force raw AMR format |
| `-f amrwb` | flag | — | — | Force raw AMR-WB format |
| `-f 3gp` | flag | — | — | Force 3GP container |

### 8.3 FFmpeg Decoding — Full CLI Reference

```bash
# Decode AMR-NB to WAV
ffmpeg -i input.amr \
  -ar 8000 \
  -ac 1 \
  -c:a pcm_s16le \
  output.wav

# Decode AMR-WB to WAV
ffmpeg -i input.awb \
  -ar 16000 \
  -ac 1 \
  -c:a pcm_s16le \
  output.wav

# Decode 3GP to WAV
ffmpeg -i input.3gp \
  -c:a pcm_s16le \
  output.wav

# Probe format
ffprobe -v quiet -print_format json -show_streams input.amr
```

### 8.4 FFmpeg Decoding — libavformat C API

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>

// ─── Complete decode program skeleton ────────────────────────────────────────
AVFormatContext *fmt_ctx = NULL;
avformat_open_input(&fmt_ctx, "input.amr", NULL, NULL);
avformat_find_stream_info(fmt_ctx, NULL);

// Find audio stream
int audio_idx = av_find_best_stream(fmt_ctx, AVMEDIA_TYPE_AUDIO, -1, -1, NULL, 0);
AVStream *stream = fmt_ctx->streams[audio_idx];

// Find decoder
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
            // AMR-NB: 160 samples per frame at 8 kHz
            // AMR-WB: 320 samples per frame at 16 kHz
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

### 8.5 Quality / Fidelity Decision Table
| Use Case | Settings | Expected Quality | Notes |
|----------|----------|------------------|-------|
| Mobile voice | AMR-NB 7.95–12.2 kbps | Good (speech) | Standard quality |
| VoIP | AMR-NB 5.9–7.4 kbps | Acceptable | Bandwidth saving |
| VoLTE | AMR-NB 12.2 kbps | Good | HD voice |
| VoNR / HD voice | AMR-WB 12.65–23.85 kbps | Very good | Wideband |
| Music / audio | Not recommended | Poor | Use AAC or Opus instead |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
```
Raw AMR: No seek table — frames have no absolute time markers
  Seeking requires: Scanning from nearest frame boundary

AMR in 3GP: stbl provides seek index
  Seeking: Frame-accurate via stbl time-to-sample mapping
```

### 9.2 Gapless Playback Data
```
AMR is a speech codec — gapless playback is not applicable.
Encoder delay:  ~25 ms (5 ms lookahead + 20 ms frame)
Padding:        0 samples
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable | Yes | Frame-by-frame decode |
| Algorithmic delay | ~25 ms | Low latency |
| Live encoding | Yes | Designed for real-time |
| RTP payload | Yes | RFC 4867 defines AMR RTP |
| HTTP streaming | Yes | Progressive download |
| WebRTC | Yes | Opus preferred now |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | AMR-NB | AMR-WB | Notes |
|----------|--------|--------|-------|
| 1 | Yes | Yes | Mono only |
| 2 | No | No | No stereo mode |

### 11.2 Downmix
```
AMR encodes mono audio only.
Stereo requires two AMR streams or a different codec.
No native stereo mode.
```

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 16-bit | Internal precision |
| Max sample rate | 16,000 Hz | AMR-WB only |
| Float support | No | Fixed-point DSP |
| DSD support | No | Not applicable |

---

## 13. KNOWN ISSUES, BUGS & EDGE CASES

### 13.1 FFmpeg Issues
| Issue | Versions Affected | Workaround |
|-------|------------------|------------|
| AMR encoder requires libopencore | All | Use libopencore-amrnb |
| Limited AMR-NB encode modes | All | Set mode explicitly |

### 13.2 Edge Cases
- **SID frames:** Comfort noise — decode as silence with CNG
- **No Data frames:** DTX mode — output silence
- **Frame type 14/15:** SPEECH_LOST / No Data — handle gracefully
- **Sample rate mismatch:** AMR requires exact rate (8k NB, 16k WB)

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM AMR
| Target | FFmpeg Command | Quality Notes |
|--------|---------------|---------------|
| → WAV | `ffmpeg -i in.amr -ar 8000 -ac 1 -c:a pcm_s16le out.wav` | Lossless decode |
| → MP3 | `ffmpeg -i in.amr -ar 8000 -ac 1 -c:a libmp3lame -q:a 2 out.mp3` | Re-encode |
| → Opus | `ffmpeg -i in.amr -ar 8000 -ac 1 -c:a libopus -b:a 24k out.opus` | Re-encode |

### 15.2 Converting TO AMR
| Source | FFmpeg Command | Quality Notes |
|--------|---------------|---------------|
| WAV → AMR-NB | `ffmpeg -i in.wav -ar 8000 -ac 1 -c:a libopencore_amrnb -b:a 12.2k out.amr` | Lossy |
| WAV → AMR-WB | `ffmpeg -i in.wav -ar 16000 -ac 1 -c:a libopencore_amrwb -b:a 23.85k out.awb` | Lossy |

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Encoder | Decoder | URL |
|---------|----------|---------|---------|---------|-----|
| libopencore-amr | C/C++ | Apache 2.0 | Good | Reference | sourceforge |
| FFmpeg AMR | C | LGPL 2.1+ | Limited | Good | ffmpeg.org |
| 3GPP reference | C | 3GPP | Reference | Reference | 3GPP member portal |

---

## 17. RELEVANT SPECIFICATIONS

### Primary Specification
- **3GPP TS 26.090:** AMR-NB Transcoding Functions
- **3GPP TS 26.101:** AMR-NB Frame Structure
- **3GPP TS 26.190:** AMR-WB Transcoding Functions
- **3GPP TS 26.201:** AMR-WB Frame Structure
- **RFC 4867:** RTP Payload Format for AMR/AMR-WB

---

## 18. IMPLEMENTATION CHECKLIST

### Build & Environment
- [ ] Identify FFmpeg flags: `--enable-libopencore-amrnb --enable-libopencore-amrwb`
- [ ] Verify AMR-NB/WB demuxers and decoders

### Encoding Pipeline
- [ ] Validate input sample rate: 8 kHz (NB) or 16 kHz (WB)
- [ ] Set channel count to 1 (mono only)
- [ ] Select appropriate bitrate mode

### Decoding Pipeline
- [ ] Handle frame type parsing from raw bitstream
- [ ] Decode Class A/B/C bits correctly
- [ ] Handle SID and No Data frames

### Metadata
- [ ] Raw AMR: Read ID3v1 if present
- [ ] AMR in 3GP: Parse 3GP metadata boxes

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
