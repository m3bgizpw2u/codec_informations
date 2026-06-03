# FFmpeg Quality Modes: CBR, VBR, ABR, CRF — Deep Technical Reference

> **Category:** FFmpeg API
> **File Extensions:** (applies to all audio formats)
> **MIME Types:** `audio/*`
> **Standardization Body:** FFmpeg Project / codec-specific bodies
> **Primary Specification:** https://ffmpeg.org/ffmpeg-codecs.html, https://github.com/FFmpeg/FFmpeg/blob/master/doc/encoders.texi
> **Patent Status:** Per-codec (MP3: Expired patents, AAC: Patented, Opus: Royalty-free, Vorbis: Royalty-free)
> **License:** FFmpeg: LGPL 2.1+; codec-specific licenses apply
> **Current Version:** Active development (rolling release)
> **Active Development:** Yes — rolling release

---

## 1. HISTORICAL CONTEXT & FUNDAMENTALS

### 1.1 Rate Control Overview

Audio compression encodes audio signals using fewer bits than the original PCM representation. Rate control determines how those bits are allocated. The choice of rate control mode profoundly affects output quality, file size, encoding consistency, and stream suitability for specific applications.

```
┌─────────────────────────────────────────────────────────────────┐
│                    RATE CONTROL MODES                            │
├──────────────────┬──────────────────┬──────────────────────────┤
│      CBR         │       VBR        │         ABR              │
│ Constant Bitrate │  Variable Bitrate │   Average Bitrate        │
│                  │                  │                          │
│ Same bitrate     │ Quality-targeted  │ Target average bitrate   │
│ throughout       │ bit usage        │ with rate fluctuation    │
│ entire file      │ throughout file  │ throughout file          │
│                  │                  │                          │
│ Fixed frame size │ Variable frame   │ Fixed average frame     │
│ allocation       │ allocation       │ allocation              │
└──────────────────┴──────────────────┴──────────────────────────┘
```

### 1.2 Why Rate Control Matters

| Concern | CBR | VBR | ABR |
|---------|-----|-----|-----|
| File size predictability | Exact | Variable | Approximate |
| Consistent stream bitrate | Yes | No | Approximate |
| Quality across content | Variable | Consistent | Moderate |
| Streaming suitability | Excellent | Good | Good |
| Storage planning | Easy | Difficult | Moderate |
| Transcoding efficiency | High | Moderate | High |

### 1.3 FFmpeg Rate Control Architecture

FFmpeg's libavcodec implements rate control through the `AVCodecContext` fields and codec-specific options:

```
AVCodecContext rate control fields:
├── bit_rate           → Primary bitrate target (bits/sec)
├── rc_min_rate        → Minimum bitrate floor
├── rc_max_rate        → Maximum bitrate ceiling
├── rc_buffer_size     → Rate control buffer size
├── rc_initial_buffer_occupancy → Initial buffer fill
├── global_quality     → Quality-based mode (for VBR/CRF)
└── compression_level  → Encoder speed/compression tradeoff
```

---

## 2. CBR — CONSTANT BITRATE

### 2.1 What CBR Is

CBR maintains a fixed bitrate throughout the entire encoded stream. Every frame uses exactly (or approximately) the same number of bits. This predictability makes CBR ideal for streaming protocols that require constant throughput.

```bash
# FFmpeg CBR examples
ffmpeg -i input.wav -c:a libmp3lame -b:a 192k output.mp3
ffmpeg -i input.wav -c:a aac -b:a 192k output.m4a
ffmpeg -i input.wav -c:a ac3 -b:a 448k output.ac3
```

### 2.2 Per-Codec CBR Implementation

#### MP3 (libmp3lame) — CBR
```bash
ffmpeg -i input.wav -c:a libmp3lame -b:a 320k output.mp3
ffmpeg -i input.wav -c:a libmp3lame -b:a 192k output.mp3
```

Valid bitrates: 8, 16, 24, 32, 40, 48, 64, 80, 96, 112, 128, 160, 192, 224, 256, 320 kbps

MP3 CBR actually uses fixed frame sizes — each frame contains exactly 1152 PCM samples encoded at the specified bitrate. The frame bit size is `frame_size = (144 × bitrate) / sample_rate`. For 44.1kHz at 192kbps: frame = (144 × 192000) / 44100 ≈ 626 bytes.

#### AAC — CBR
```bash
ffmpeg -i input.wav -c:a aac -b:a 192k output.m4a
ffmpeg -i input.wav -c:a libfdk_aac -b:a 256k output.m4a
```

AAC in FFmpeg is more flexible than MP3 — AAC frames have variable sizes within limits. True AAC CBR is achieved by maintaining a tight rate control window.

#### AC-3 (Dolby Digital) — CBR
```bash
ffmpeg -i input.wav -c:a ac3 -b:a 448k output.ac3
```

AC-3 supports these bitrates: 32, 40, 48, 56, 64, 80, 96, 112, 128, 160, 192, 224, 256, 320, 384, 448, 512, 640 kbps

#### DTS — CBR
```bash
ffmpeg -i input.wav -c:a dca -b:a 1536k output.dts
```

DTS bitrates: 32, 56, 64, 96, 112, 128, 192, 224, 256, 320, 384, 448, 640, 768, 960, 1024, 1152, 1280, 1344, 1408, 1411, 1509, 1536 kbps

### 2.3 CBR Use Cases

| Use Case | Recommended Bitrate | Rationale |
|----------|-------------------|-----------|
| Legacy streaming (RTMP) | 128–192 kbps | Protocol requires fixed bitrate |
| Digital radio broadcast | 128–192 kbps | Regulatory and infrastructure constraints |
| Music on legacy devices | 192–320 kbps | Maximum compatibility |
| VOIP/Voice chat | 32–64 kbps mono | Bandwidth-constrained |
| Audiobooks on mobile | 64–96 kbps mono | Storage and bandwidth |
| Satellite radio | 64–128 kbps | Transmission constraints |
| Broadcasting (legacy) | 256 kbps | Standard broadcast quality |

### 2.4 CBR Trade-offs

**Advantages:**
- Predictable file size: `file_size = bitrate × duration / 8`
- No bitrate spikes — ideal for streaming over fixed-bandwidth connections
- Simplest for bandwidth-constrained delivery
- Decoder buffer requirements are minimal and fixed

**Disadvantages:**
- Quality varies across content complexity
- Simple passages get same bits as complex passages — wasted bits on silence/silence
- Cannot allocate extra bits for demanding passages
- Achieves lower average quality than VBR at the same average bitrate
- Not optimal for storage efficiency

---

## 3. VBR — VARIABLE BITRATE

### 3.1 What VBR Is

VBR allocates bits based on the complexity of each passage. Complex audio (orchestral crescendos, dense harmonics) gets more bits; simple audio (quiet passages, speech pauses) gets fewer. The result is consistent perceptual quality across the file.

```
┌─────────────────────────────────────────────────────────────────┐
│              VBR BIT ALLOCATION (Example)                       │
├─────────────────────────────────────────────────────────────────┤
│ Bitrate                                                         │
│  ▲                                                              │
│  │     ╭──╮          ╭─────╮        ╭──╮                      │
│  │    ╱    ╲        ╱       ╲      ╱    ╲                     │
│  │───╱──────╲──────╱─────────╲────╱──────╲──────► Time       │
│  │  quiet   complex passage   complex   quiet                   │
│  └──────────────────────────────────────────────────────────── │
│                                                                 │
│  Lower bitrate on simple passages, higher on complex ones.      │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 FFmpeg VBR Syntax

In FFmpeg, VBR is typically enabled via `-q:a` (quality-based) or `-vbr` options:

```bash
# LAME VBR
ffmpeg -i input.wav -c:a libmp3lame -q:a 2 output.mp3

# AAC VBR (libfdk_aac)
ffmpeg -i input.wav -c:a libfdk_aac -vbr 4 output.m4a

# Vorbis VBR
ffmpeg -i input.wav -c:a libvorbis -q:a 6 output.ogg

# Opus VBR (default)
ffmpeg -i input.wav -c:a libopus -vbr on -b:a 128k output.opus
```

### 3.3 Per-Codec VBR Quality Scales

#### libmp3lame — LAME VBR

The LAME VBR scale ranges from `-V 0` (highest quality) to `-V 9` (lowest quality). In FFmpeg, use `-q:a`:

```bash
# Quality 0 — highest, ~245 kbps average
ffmpeg -i input.wav -c:a libmp3lame -q:a 0 output.mp3

# Quality 1 — ~225 kbps
ffmpeg -i input.wav -c:a libmp3lame -q:a 1 output.mp3

# Quality 2 — ~190 kbps (recommended for transparency)
ffmpeg -i input.wav -c:a libmp3lame -q:a 2 output.mp3

# Quality 3 — ~175 kbps
ffmpeg -i input.wav -c:a libmp3lame -q:a 3 output.mp3

# Quality 4 — ~165 kbps (LAME default)
ffmpeg -i input.wav -c:a libmp3lame -q:a 4 output.mp3

# Quality 5 — ~130 kbps
ffmpeg -i input.wav -c:a libmp3lame -q:a 5 output.mp3
```

LAME VBR bitrate table:

| FFmpeg `-q:a` | LAME `-V` | Avg Bitrate (kbps) | Bitrate Range (kbps) | Transparency |
|---------------|-----------|---------------------|----------------------|--------------|
| 0 | V0 | ~245 | 220–260 | Yes |
| 1 | V1 | ~225 | 190–250 | Yes |
| 2 | V2 | ~190 | 170–210 | Yes |
| 3 | V3 | ~175 | 150–195 | Near-transparent |
| 4 | V4 | ~165 | 140–185 | Good |
| 5 | V5 | ~130 | 120–150 | Acceptable |
| 6 | V6 | ~115 | 100–130 | Acceptable |
| 7 | V7 | ~100 | 80–120 | Low |
| 8 | V8 | ~85 | 70–105 | Low |
| 9 | V9 | ~65 | 45–85 | Very low |

#### libvorbis — Vorbis VBR

Vorbis uses a quality scale from `-1.0` to `10.0` (floating point). In FFmpeg, use `-q:a`:

```bash
# Quality 10 — maximum (~256+ kbps)
ffmpeg -i input.wav -c:a libvorbis -q:a 10 output.ogg

# Quality 6 — high quality (~192 kbps) [recommended]
ffmpeg -i input.wav -c:a libvorbis -q:a 6 output.ogg

# Quality 3 — default (~128 kbps)
ffmpeg -i input.wav -c:a libvorbis -q:a 3 output.ogg

# Quality 0 — low (~64 kbps)
ffmpeg -i input.wav -c:a libvorbis -q:a 0 output.ogg
```

Vorbis VBR bitrate table:

| FFmpeg `-q:a` | Quality | Avg Bitrate (kbps) | Target Use | Transparency |
|---------------|---------|---------------------|------------|--------------|
| -1.0 | Lowest | ~45 | Low bandwidth | No |
| 0 | Very low | ~64 | Voice | No |
| 1 | Low | ~80 | Voice/mobile | Marginal |
| 2 | Medium | ~96 | Mobile streaming | No |
| 3 | Good | ~128 | Standard streaming | Marginal |
| 4 | High | ~160 | High quality | Near |
| 5 | Very high | ~192 | Audiophile streaming | Yes |
| 6 | Excellent | ~224 | Near-transparent | Yes |
| 7 | Near-perfect | ~256 | Transparent | Yes |
| 8 | Maximum | ~320 | Maximum quality | Yes |
| 9 | Extreme | ~400 | Lossless approximation | Yes |
| 10 | Unrestricted | ~500+ | Theoretical max | Yes |

#### libopus — Opus VBR

Opus uses VBR by default. The quality/size tradeoff is controlled by bitrate (`-b:a`):

```bash
# Default VBR with bitrate cap
ffmpeg -i input.wav -c:a libopus -b:a 128k output.opus

# Explicit VBR mode
ffmpeg -i input.wav -c:a libopus -vbr on -b:a 128k output.opus

# CVBR (constrained VBR) — targets bitrate but allows some fluctuation
ffmpeg -i input.wav -c:a libopus -vbr cvbr -b:a 128k output.opus

# CBR (rarely used)
ffmpeg -i input.wav -c:a libopus -vbr off -b:a 128k output.opus
```

Opus bitrate table:

| `-b:a` | Mono (kbps) | Stereo (kbps) | Application | Transparency |
|--------|-------------|---------------|-------------|--------------|
| 6 | 6 | — | Very low bandwidth | No |
| 12 | 12 | — | Voice | Marginal |
| 16 | 16 | 16 | Standard voice | Near |
| 24 | 24 | 24 | HD voice | Yes (voice) |
| 32 | 32 | 32 | Wideband | Yes (voice) |
| 48 | 48 | 48 | High quality | Yes (voice) |
| 64 | — | 64 | Music streaming | Yes |
| 96 | — | 96 | High quality | Yes |
| 128 | — | 128 | Near-transparent | Yes |
| 160 | — | 160 | Transparent | Yes |
| 192 | — | 192 | Maximum quality | Yes |
| 256 | — | 256 | Ultra high quality | Yes |

#### libfdk_aac — Fraunhofer AAC VBR

```bash
# VBR quality 1–5 (1=lowest, 5=highest)
ffmpeg -i input.wav -c:a libfdk_aac -vbr 1 output.m4a  # ~32 kbps
ffmpeg -i input.wav -c:a libfdk_aac -vbr 2 output.m4a  # ~64 kbps
ffmpeg -i input.wav -c:a libfdk_aac -vbr 3 output.m4a  # ~96 kbps
ffmpeg -i input.wav -c:a libfdk_aac -vbr 4 output.m4a  # ~128 kbps
ffmpeg -i input.wav -c:a libfdk_aac -vbr 5 output.m4a  # ~192+ kbps
```

Fraunhofer AAC VBR quality table:

| `-vbr` | Avg Bitrate (kbps) | Quality Level | Transparency |
|--------|--------------------|---------------|--------------|
| 1 | ~32 | Lowest | No |
| 2 | ~64 | Low | Marginal |
| 3 | ~96 | Medium | Near (voice) |
| 4 | ~128 | High | Yes (most content) |
| 5 | ~192+ | Highest | Yes |

#### Native FFmpeg AAC VBR

The native FFmpeg AAC encoder uses `-q:a` for quality mode (not true VBR, but similar):

```bash
ffmpeg -i input.wav -c:a aac -q:a 1 output.m4a   # Highest quality
ffmpeg -i input.wav -c:a aac -q:a 2 output.m4a   # High quality
ffmpeg -i input.wav -c:a aac -q:a 4 output.m4a   # Medium quality
```

Note: The native AAC encoder's quality mode does not produce true VBR — it uses a quality-based quantization parameter but may not achieve the same bitrate variability as libfdk_aac or libopus.

### 3.4 VBR Quality vs File Size Comparison

| Codec | Quality Setting | Avg Bitrate | File Size (3-min song) | Quality |
|-------|----------------|-------------|------------------------|---------|
| MP3 | `-q:a 0` (V0) | ~245 kbps | ~5.5 MB | Highest |
| MP3 | `-q:a 2` (V2) | ~190 kbps | ~4.3 MB | Transparent |
| MP3 | `-q:a 4` (V4) | ~165 kbps | ~3.7 MB | Good |
| Vorbis | `-q:a 6` | ~224 kbps | ~5.0 MB | Transparent |
| Vorbis | `-q:a 3` | ~128 kbps | ~2.9 MB | Good |
| Opus | `-b:a 128k` | ~128 kbps | ~2.9 MB | Transparent |
| Opus | `-b:a 64k` | ~64 kbps | ~1.4 MB | Good (voice) |
| AAC | `-vbr 5` | ~192 kbps | ~4.3 MB | Transparent |
| AAC | `-vbr 3` | ~96 kbps | ~2.2 MB | Good |

---

## 4. ABR — AVERAGE BITRATE

### 4.1 What ABR Is

ABR (Average Bitrate) targets a specified average bitrate but allows the encoder to vary the bitrate up and down as needed. Unlike VBR (which targets a quality level and produces variable file size), ABR targets an average bitrate and allows rate fluctuation. Unlike CBR (which forces a constant bitrate), ABR can allocate more bits to complex passages.

ABR is essentially a compromise mode: it gives better quality than CBR at the same average bitrate, while providing more predictable file sizes than VBR.

### 4.2 FFmpeg ABR Implementation

#### MP3 (libmp3lame) — ABR
In FFmpeg, ABR for MP3 is specified with `-b:a` combined with `-q:a` or `-abr`:

```bash
# ABR 192 kbps
ffmpeg -i input.wav -c:a libmp3lame -b:a 192k output.mp3

# ABR with quality hint (better than pure CBR)
ffmpeg -i input.wav -c:a libmp3lame -b:a 192k -q:a 2 output.mp3
```

LAME's ABR mode uses a minimum bitrate floor and a target average. It allocates bits based on the complexity of each frame while attempting to maintain the specified average over the entire file.

Important: In LAME/FFmpeg, using `-b:a` alone does NOT guarantee true CBR — it is ABR behavior. For strict CBR, additional options may be needed depending on the encoder version.

#### AAC — ABR
```bash
ffmpeg -i input.wav -c:a aac -b:a 192k output.m4a
ffmpeg -i input.wav -c:a libfdk_aac -b:a 192k output.m4a
```

#### Vorbis — ABR
```bash
ffmpeg -i input.wav -c:a libvorbis -b:a 192k output.ogg
```

#### Opus — ABR via CVBR
Opus uses CVBR (constrained VBR) as its closest ABR-equivalent:

```bash
ffmpeg -i input.wav -c:a libopus -vbr cvbr -b:a 128k output.opus
```

### 4.3 ABR Trade-offs

| Aspect | ABR vs CBR | ABR vs VBR |
|--------|------------|------------|
| Quality | Better than CBR at same avg bitrate | Slightly lower than VBR at same avg bitrate |
| File size | More predictable than VBR | Less predictable than CBR |
| Streaming | Good for HTTP streaming | Better than pure VBR |
| Complexity | Simpler than VBR | More complex than CBR |

### 4.4 ABR Bitrate Selection

| ABR Target | Typical Use | Quality Notes |
|------------|-------------|---------------|
| 64 kbps | Mobile voice | Good for voice, poor for music |
| 96 kbps | Low bandwidth | Acceptable for casual listening |
| 128 kbps | Standard streaming | Good for most listeners |
| 160 kbps | High quality | Near-transparent for many |
| 192 kbps | High quality | Transparent for most content |
| 256 kbps | Audiophile | Transparent |
| 320 kbps | Maximum | Transparent with margin |

---

## 5. CRF — CONSTANT RATE FACTOR

### 5.1 What CRF Is

CRF (Constant Rate Factor) is the mode used by modern video codecs (x264, x265, VP9) and Opus. In CRF mode, the encoder targets a specific quality level and adjusts bitrate as needed to achieve that quality. The bitrate is allowed to vary freely — unlike VBR which may target a bitrate range, CRF adjusts quality to maintain a fixed quality score.

**Key distinction:**
- **VBR:** "Use approximately N bits per second, but adjust up/down for quality"
- **CRF:** "Maintain quality level Q, use whatever bitrate is needed"

### 5.2 CRF in FFmpeg

#### Opus — CRF Mode

Opus uses CRF internally. The CLI represents it as VBR with a quality target:

```bash
# Opus CRF mode (target quality, bitrate varies)
ffmpeg -i input.wav -c:a libopus \
  -vbr on \
  -b:a 0 \
  -vbr_quality 10 \
  output.opus
```

Note: In current FFmpeg, Opus's CRF behavior is achieved by setting `-b:a 0` with `-vbr on`, which tells Opus to use variable bitrate with quality-based allocation.

#### libvorbis — Quality-Based Mode

libvorbis's `-q:a` is effectively a CRF-like quality target:

```bash
ffmpeg -i input.wav -c:a libvorbis -q:a 6 output.ogg
```

#### x264/x265 Audio (CRF for video)

While not strictly audio, FFmpeg's video encoding with CRF is relevant for transcoding pipelines:

```bash
# Video CRF (not audio, but relevant for A/V pipelines)
ffmpeg -i input.mkv -c:v libx264 -crf 23 -c:a copy output.mkv
```

### 5.3 CRF vs VBR in Audio Context

| Feature | VBR | CRF |
|---------|-----|-----|
| Primary target | Bitrate range | Quality score |
| Bitrate variability | Within range | Unconstrained |
| Quality consistency | Moderate | High |
| File size predictability | Moderate | Low |
| FFmpeg representation | `-q:a` / `-vbr` | `-b:a 0` + `-vbr on` |

---

## 6. PER-CODEC BITRATE TABLES — COMPLETE REFERENCE

### 6.1 MP3 (libmp3lame)

| Quality | Avg Bitrate | Range | Transparency | Notes |
|---------|-------------|-------|--------------|-------|
| `-q:a 0` (V0) | ~245 kbps | 220–260 | Yes | Highest quality VBR |
| `-q:a 1` (V1) | ~225 kbps | 190–250 | Yes | Very high quality |
| `-q:a 2` (V2) | ~190 kbps | 170–210 | Yes | Recommended transparent |
| `-q:a 3` (V3) | ~175 kbps | 150–195 | Near | Very good |
| `-q:a 4` (V4) | ~165 kbps | 140–185 | Good | LAME default |
| `-q:a 5` (V5) | ~130 kbps | 120–150 | Acceptable | Good for voice |
| `-q:a 6` (V6) | ~115 kbps | 100–130 | Acceptable | |
| `-q:a 7` (V7) | ~100 kbps | 80–120 | Low | |
| `-q:a 8` (V8) | ~85 kbps | 70–105 | Low | |
| `-q:a 9` (V9) | ~65 kbps | 45–85 | Very low | Minimum quality |
| `-b:a 320k` | 320 kbps | 320 CBR | Yes | Maximum CBR |
| `-b:a 192k` | 192 kbps | 192 CBR | Yes | Standard CBR |

### 6.2 AAC

#### Fraunhofer FDK AAC (`-vbr` mode)

| `-vbr` | Avg Bitrate | Quality | Transparency |
|--------|-------------|---------|--------------|
| 1 | ~32 kbps | Lowest | No |
| 2 | ~64 kbps | Low | Marginal |
| 3 | ~96 kbps | Medium | Near (voice) |
| 4 | ~128 kbps | High | Yes (most content) |
| 5 | ~192 kbps | Highest | Yes |

#### Fraunhofer FDK AAC (`-b:a` mode — ABR/CBR)

| `-b:a` | Type | Quality | Transparency |
|--------|------|---------|--------------|
| 8 kbps | CBR | Very low | No |
| 32 kbps | CBR | Low | No |
| 64 kbps | ABR | Medium | No |
| 96 kbps | ABR | Good | Marginal |
| 128 kbps | ABR | High | Yes (most) |
| 192 kbps | ABR | Very high | Yes |
| 256 kbps | ABR | Near-transparent | Yes |
| 512 kbps | CBR | Transparent | Yes |

#### Native FFmpeg AAC

| `-q:a` | Quality | Approx Bitrate | Transparency |
|--------|---------|----------------|--------------|
| 0.1 | Highest | ~250 kbps | Yes |
| 0.2 | High | ~200 kbps | Yes |
| 0.5 | Medium | ~150 kbps | Near |
| 1.0 | Standard | ~128 kbps | Yes |
| 2.0 | Low | ~96 kbps | Marginal |

### 6.3 Vorbis (libvorbis)

| `-q:a` | Quality | Avg Bitrate | Transparency |
|--------|---------|-------------|--------------|
| -1.0 | Lowest | ~45 kbps | No |
| 0 | Very low | ~64 kbps | No |
| 1 | Low | ~80 kbps | Marginal |
| 2 | Medium-low | ~96 kbps | No |
| 3 | Medium | ~128 kbps | Near |
| 4 | Medium-high | ~160 kbps | Near |
| 5 | High | ~192 kbps | Yes (voice) |
| 6 | Very high | ~224 kbps | Yes |
| 7 | Excellent | ~256 kbps | Yes |
| 8 | Near-perfect | ~320 kbps | Yes |
| 9 | Extreme | ~400 kbps | Yes |
| 10 | Maximum | ~500+ kbps | Yes |

### 6.4 Opus (libopus)

| `-b:a` | Mono | Stereo | Application | Transparency |
|--------|------|--------|-------------|--------------|
| 6 kbps | 6 | — | Emergency | No |
| 12 kbps | 12 | — | Voice | No |
| 16 kbps | 16 | 16 | Standard voice | Marginal |
| 24 kbps | 24 | 24 | HD voice | Yes (voice) |
| 32 kbps | 32 | 32 | High voice | Yes (voice) |
| 48 kbps | 48 | 48 | Very high voice | Yes |
| 64 kbps | — | 64 | Music (low) | Yes |
| 96 kbps | — | 96 | Music (high) | Yes |
| 128 kbps | — | 128 | Music (transparent) | Yes |
| 160 kbps | — | 160 | High quality | Yes |
| 192 kbps | — | 192 | Very high | Yes |
| 256 kbps | — | 256 | Maximum | Yes |

### 6.5 AC-3 (Dolby Digital)

| Bitrate (kbps) | Channels | Typical Use | Transparency |
|----------------|----------|-------------|--------------|
| 32 | 1 | Voice mono | No |
| 64 | 1–2 | Voice stereo | No |
| 128 | 1–2 | Voice high | Marginal |
| 192 | 5.1 | Standard broadcast | Yes |
| 256 | 5.1 | High broadcast | Yes |
| 384 | 5.1 | HD broadcast | Yes |
| 448 | 5.1 | Cinema/broadcast | Yes |
| 640 | 5.1 | Maximum | Yes |

### 6.6 DTS

| Bitrate (kbps) | Channels | Typical Use | Transparency |
|----------------|----------|-------------|--------------|
| 1411 | 5.1 | DTS-CD | Yes |
| 1509 | 5.1 | DTS-ES | Yes |
| 768 | 5.1 | Standard | Yes |
| 512 | 5.1 | Core | Yes |
| 384 | 5.1 | Low | Near |
| 256 | 5.1 | Very low | Marginal |
| 64 | Mono | Voice | No |

---

## 7. BITRATE SELECTION FOR TRANSPARENCY

### 7.1 Hydrogenaudio Transparency Guidelines

Per Hydrogenaudio community consensus:

| Codec | Transparent Bitrate | Recommended Setting | Notes |
|-------|--------------------|---------------------|-------|
| MP3 (LAME) | ≥192 kbps VBR | `-q:a 2` (~190kbps) | 192kbps VBR is widely accepted as transparent |
| AAC | ≥128 kbps VBR | `-vbr 4` (~128kbps) | Modern AAC at 128kbps rivals MP3 at 192kbps |
| Vorbis | ≥160 kbps VBR | `-q:a 5` (~192kbps) | Slightly higher than AAC needed |
| Opus | ≥64 kbps stereo | `-b:a 128k` | Most efficient modern codec |
| FLAC | Lossless | `-compression_level 8` | 100% lossless |
| ALAC | Lossless | (default) | 100% lossless |
| WavPack | Lossless | `-compression_level 6` | 100% lossless |

### 7.2 Factors Affecting Transparency

Transparency is not a binary state — it depends on:

1. **Listener sensitivity:** Trained ears detect artifacts at lower thresholds
2. **Equipment quality:** High-end headphones reveal artifacts masked on consumer gear
3. **Content type:** Transient-rich music (percussion, plucked strings) is harder to encode transparently
4. **Listening volume:** Artifacts become audible at higher volumes
5. **Content familiarity:** Listeners familiar with source material detect artifacts more easily
6. **Encoder implementation:** Same codec, different encoders may differ in quality

### 7.3 Per-Genre Bitrate Recommendations

| Genre | MP3 (LAME VBR) | AAC (libfdk) | Vorbis | Opus |
|-------|----------------|---------------|--------|------|
| Speech/Audiobook | `-q:a 6` (115k) | `-vbr 3` (96k) | `-q:a 3` (128k) | `-b:a 24k` mono |
| Pop/Electronic | `-q:a 2` (190k) | `-vbr 4` (128k) | `-q:a 5` (192k) | `-b:a 128k` |
| Classical/Jazz | `-q:a 0` (245k) | `-vbr 5` (192k) | `-q:a 7` (256k) | `-b:a 192k` |
| Rock/Metal | `-q:a 2` (190k) | `-vbr 4` (128k) | `-q:a 5` (192k) | `-b:a 128k` |
| World/Folk | `-q:a 1` (225k) | `-vbr 4` (128k) | `-q:a 6` (224k) | `-b:a 160k` |
| Lossless Archive | FLAC/ALAC | FLAC/ALAC | FLAC | FLAC |

### 7.4 Codec Efficiency Comparison

| Codec | Efficiency (quality per bitrate) | Transparency Threshold | Notes |
|-------|--------------------------------|------------------------|-------|
| Opus | Highest | ~64 kbps stereo | Best modern codec |
| AAC | High | ~128 kbps | Mature, widely supported |
| Vorbis | High | ~160 kbps | Open, well-regarded |
| MP3 | Moderate | ~192 kbps | Widest compatibility |
| FLAC | Lossless | N/A | Reference quality |
| ALAC | Lossless | N/A | Apple ecosystem |

---

## 8. FFMPEG LIBEXCODEC AVOPTION RATE CONTROL TABLES

### 8.1 Global Codec Options (all encoders)

| Option | Type | Default | Range | Description |
|--------|------|---------|-------|-------------|
| `b` | int | varies | codec-dependent | Target bitrate in bits/s |
| `q` | float | varies | codec-dependent | Quality-based mode (VBR) |
| `compression_level` | int | varies | codec-dependent | Encode speed vs compression ratio |
| `flags` | flags | 0 | +qscale | Enable VBR mode |
| `profile` | string | varies | codec-specific | Encoder profile selection |
| `level` | int | varies | codec-specific | Encoder level selection |

### 8.2 libmp3lame Options

| Option | CLI Flag | Type | Default | Range | Description |
|--------|----------|------|---------|-------|-------------|
| `b` | `-b:a` | int | 128k | 8k–320k | CBR/ABR bitrate |
| `q` | `-q:a` | float | 4.0 | 0–9 | VBR quality (0=best) |
| `abr` | — | int | 0 | 0/1 | Enable ABR mode |
| `joint_stereo` | — | int | 1 | 0/1 | Enable MS-stereo |
| `drm` | — | int | 0 | 0/1 | Digital Radio Mondiale |
| `mono` | — | int | 0 | 0/1 | Encode as mono |
| `mode` | — | int | auto | 0–3 | Stereo mode |
| `padding` | — | int | auto | -1–2 | Padding mode |
| `快` [NEEDS VERIFICATION] | — | — | — | — | — |

### 8.3 libvorbis Options

| Option | CLI Flag | Type | Default | Range | Description |
|--------|----------|------|---------|-------|-------------|
| `b` | `-b:a` | int | 128k | 0–infinity | ABR target bitrate |
| `q` | `-q:a` | float | 3.0 | -1.0–10.0 | VBR quality (higher=better) |
| `maxrate` | — | int | 0 | 0–infinity | Maximum bitrate |
| `minrate` | — | int | 0 | 0–infinity | Minimum bitrate |
| `timelength` [NEEDS VERIFICATION] | — | — | — | — | — |
| `pages` [NEEDS VERIFICATION] | — | — | — | — | — |

### 8.4 libopus Options

| Option | CLI Flag | Type | Default | Range | Description |
|--------|----------|------|---------|-------|-------------|
| `b` | `-b:a` | int | 0 | 0–infinity | Bitrate (0=unlimited VBR) |
| `vbr` | `-vbr` | int | on | off/on/cvbr | VBR mode |
| `vbr_constraint` | — | int | unconstrained | unconstrained/tight | VBR constraint |
| `cutoff` | — | int | 0 | 0–20000 | Bandwidth cutoff (Hz) |
| `frame_duration` | — | float | 20.0 | 2.5–120.0 | Frame duration (ms) |
| `application` | — | string | audio | audio/voip/lowdelay | Application mode |
| `mapping_family` | — | int | 255 (auto) | 0–255 | Channel mapping |
| `signal` | — | string | auto | auto/music/speech | Signal type hint |
| `comp` [NEEDS VERIFICATION] | — | — | — | — | — |
| `vbr_quality` | — | float | — | 0–10 | VBR quality target |

### 8.5 libfdk_aac Options

| Option | CLI Flag | Type | Default | Range | Description |
|--------|----------|------|---------|-------|-------------|
| `b` | `-b:a` | int | 128k | 8k–512k | Target bitrate |
| `vbr` | `-vbr` | int | 0 (off) | 0–5 | VBR quality (0=off/CBR, 5=highest) |
| `afterburner` | — | int | 1 | 0/1 | Enable afterburner (higher quality) |
| `profile` | — | string | auto | aac_low/he/he_v2/etc. | AAC profile |
| `signaling` | — | int | auto | auto/explicit/sbr/explicit_sbr | SBR signaling |
| `latm` | — | int | auto | 0/1/auto | LATM output |
| `header_period` | — | int | 0 | 0–65535 | ID3v2 header period |

### 8.6 FLAC Options

| Option | CLI Flag | Type | Default | Range | Description |
|--------|----------|------|---------|-------|-------------|
| `compression_level` | `-compression_level` | int | 5 | 0–12 | Compression level |
| `frame_size` | — | int | 0 | 0–65535 | Blocksize |
| `lpc_coeff_precision` | — | int | 15 | 0–15 | LPC coefficient precision |
| `lpc_type` | — | int | 2 | 0–6 | LPC algorithm |
| `lpc_passes` | — | int | 2 | 1–20 | Number of passes |
| `min_partition_order` | — | int | 2 | 0–15 | Minimum partition order |
| `max_partition_order` | — | int | 6 | 0–15 | Maximum partition order |
| `prediction_order_method` | — | int | -1 | -1–5 | Prediction order method |
| `ch_mode` | — | int | auto | auto/explicit | Channel mode |
| `exact_rice_parameters` | — | int | 0 | 0/1 | Exact Rice params |
| `multi_dim_quant` | — | int | 0 | 0/1 | Multi-dimensional quantization |
| `streamable` | — | int | 0 | 0/1 | Streamable subset |

### 8.7 AC-3 Options

| Option | CLI Flag | Type | Default | Range | Description |
|--------|----------|------|---------|-------|-------------|
| `b` | `-b:a` | int | 128k | 32k–640k | Bitrate |
| `bit_rate` | — | int | 128k | 32k–640k | Alias for `b` |
| `acodec` | — | — | — | — | — |

---

## 9. MEMORY AND CPU IMPLICATIONS

### 9.1 VBR vs CBR: Computational Cost

| Mode | Encode CPU | Decode CPU | Memory (Encode) | Memory (Decode) |
|------|-----------|-----------|-----------------|-----------------|
| CBR | Moderate | Low | Low | Low |
| VBR | Higher | Low | Moderate | Low |
| ABR | Moderate-High | Low | Moderate | Low |
| CRF | Higher | Low | Moderate | Low |

**Notes:**
- VBR/CRF require more CPU because the encoder must analyze content complexity before committing to a bit allocation
- Decode cost is generally independent of rate control mode (decoder processes whatever bits are present)
- Memory usage is primarily for the psychoacoustic model (VBR/CRF) and bit reservoir (CBR)

### 9.2 Encoding Speed by Quality Setting

| Codec | Quality Level | Encode Speed | Notes |
|-------|--------------|-------------|-------|
| FLAC | Level 0 | ~50× realtime | Very fast |
| FLAC | Level 8 | ~10× realtime | Good compression |
| FLAC | Level 12 | ~2× realtime | Maximum compression |
| MP3 | `-q:a 0` (V0) | ~10× realtime | Slowest VBR |
| MP3 | `-q:a 5` (V5) | ~30× realtime | Fast VBR |
| MP3 | CBR 320k | ~40× realtime | Fastest |
| AAC (libfdk) | VBR 5 | ~5× realtime | Slowest |
| AAC (libfdk) | VBR 1 | ~15× realtime | Fastest |
| Vorbis | `-q:a 10` | ~5× realtime | Slowest |
| Vorbis | `-q:a 3` | ~15× realtime | Fastest |
| Opus | 256k | ~8× realtime | Variable |

### 9.3 Buffer Requirements

For streaming, rate control requires decoder buffers:

| Codec | Min Buffer | Recommended Buffer | Notes |
|-------|-----------|-------------------|-------|
| MP3 | 1152 samples | 4608 samples | Small |
| AAC | 1024 samples | 4096 samples | Small |
| Vorbis | Variable | 2048 samples | Variable |
| Opus | 960 samples | 5760 samples | 60ms lookahead |
| AC-3 | 1536 samples | 6144 samples | 32ms blocks |

---

## 10. FFmpeg C API Rate Control

### 10.1 Setting CBR via libavcodec
```c
AVCodecContext *ctx = avcodec_alloc_context3(codec);

// CBR mode
ctx->bit_rate = 192000;          // Target bitrate
ctx->rc_min_rate = 192000;       // Floor = target
ctx->rc_max_rate = 192000;       // Ceiling = target
ctx->rc_buffer_size = 0;          // Disable buffer sizing
```

### 10.2 Setting VBR via libavcodec
```c
// LAME VBR
av_opt_set_int(ctx, "q", 2.0, AV_OPT_SEARCH_CHILDREN);  // Quality 2

// libfdk_aac VBR
av_opt_set_int(ctx, "vbr", 4, AV_OPT_SEARCH_CHILDREN); // Quality 4

// libvorbis VBR
av_opt_set_double(ctx, "q", 6.0, AV_OPT_SEARCH_CHILDREN); // Quality 6

// libopus VBR (default)
av_opt_set_int(ctx, "vbr", 1, AV_OPT_SEARCH_CHILDREN);    // VBR on
av_opt_set_int(ctx, "b", 128000, AV_OPT_SEARCH_CHILDREN);  // Target
```

### 10.3 Setting ABR via libavcodec
```c
// ABR mode
ctx->bit_rate = 192000;          // Average target
ctx->rc_min_rate = 0;            // Allow fluctuation below
ctx->rc_max_rate = 256000;       // Allow some peaks
ctx->rc_buffer_size = 0;
```

### 10.4 Encoder Initialization for VBR
```c
const AVCodec *codec = avcodec_find_encoder_by_name("libfdk_aac");
AVCodecContext *ctx = avcodec_alloc_context3(codec);

ctx->bit_rate = 0;               // Not used for VBR
ctx->sample_rate = 44100;
ctx->ch_layout = (AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO;
ctx->sample_fmt = AV_SAMPLE_FMT_S16P;

// Set VBR quality
av_opt_set_int(ctx, "vbr", 4, AV_OPT_SEARCH_CHILDREN);

// Open codec
int ret = avcodec_open2(ctx, codec, NULL);
if (ret < 0) {
    // Handle error
}

// Encode
AVFrame *frame = av_frame_alloc();
frame->nb_samples = ctx->frame_size;
frame->format = ctx->sample_fmt;
av_channel_layout_copy(&frame->ch_layout, &ctx->ch_layout);
av_frame_get_buffer(frame, 0);

AVPacket *pkt = av_packet_alloc();
while (has_input) {
    // Fill frame->data
    avcodec_send_frame(ctx, frame);
    while (avcodec_receive_packet(ctx, pkt) == 0) {
        write_packet(pkt);
        av_packet_unref(pkt);
    }
}
// Flush
avcodec_send_frame(ctx, NULL);
while (avcodec_receive_packet(ctx, pkt) == 0) {
    write_packet(pkt);
    av_packet_unref(pkt);
}
```

---

## 11. PRACTICAL GUIDES

### 11.1 Choosing the Right Mode

| Scenario | Recommended Mode | Setting |
|----------|-----------------|---------|
| Archival storage | VBR | Highest quality (lossless preferred) |
| Streaming server | VBR or ABR | `-q:a 2` MP3, `-b:a 128k` Opus |
| Legacy device compatibility | CBR | `-b:a 192k` |
| Mobile bandwidth-constrained | VBR | `-q:a 4` MP3, `-b:a 96k` Opus |
| Real-time voice | CBR | `-b:a 64k` Opus |
| CD ripping | VBR (lossless) | FLAC `-compression_level 8` |
| Podcast (mono voice) | VBR | `-q:a 6` MP3, `-b:a 32k` Opus |
| Multichannel music | VBR | `-vbr 4` AAC, `-b:a 192k` Opus |

### 11.2 Common Encoding Presets

```bash
# ==== MP3 VBR (recommended) ====
ffmpeg -i input.wav -c:a libmp3lame -q:a 2 output.mp3  # ~190kbps VBR

# ==== MP3 CBR (legacy/streaming) ====
ffmpeg -i input.wav -c:a libmp3lame -b:a 192k output.mp3

# ==== AAC VBR (best quality) ====
ffmpeg -i input.wav -c:a libfdk_aac -vbr 4 output.m4a  # ~128kbps VBR

# ==== AAC ABR (streaming) ====
ffmpeg -i input.wav -c:a libfdk_aac -b:a 192k output.m4a

# ==== Vorbis VBR ====
ffmpeg -i input.wav -c:a libvorbis -q:a 6 output.ogg  # ~224kbps

# ==== Opus VBR ====
ffmpeg -i input.wav -c:a libopus -vbr on -b:a 128k output.opus

# ==== FLAC Lossless ====
ffmpeg -i input.wav -c:a flac -compression_level 8 output.flac

# ==== ALAC Lossless ====
ffmpeg -i input.wav -c:a alac output.m4a
```

### 11.3 Conversion Between Modes

```bash
# Convert VBR MP3 to CBR AAC
ffmpeg -i input.mp3 -c:a libfdk_aac -b:a 192k output.m4a

# Convert CBR MP3 to VBR AAC
ffmpeg -i input.mp3 -c:a libfdk_aac -vbr 4 output.m4a

# Convert to transparent Opus
ffmpeg -i input.wav -c:a libopus -vbr on -b:a 128k output.opus

# Downgrade quality for space savings
ffmpeg -i input.flac -c:a libopus -vbr on -b:a 64k output.opus
```

---

## 12. QUALITY VERIFICATION

### 12.1 Measuring Actual Bitrate
```bash
ffprobe -v error -show_entries format=bit_rate:stream=bit_rate \
  -of default=noprint_wrappers=1 input.mp3
```

### 12.2 Bitrate Distribution (VBR Analysis)
```bash
# Extract bitrate per frame (MP3)
ffprobe -v debug -show_packets input.mp3 2>&1 | \
  grep "pkt_pts" | wc -l

# Check VBR quality distribution
ffprobe -v quiet -show_entries stream=bit_rate \
  -of csv=p=0 input.mp3 | sort -u
```

### 12.3 Perceptual Quality Testing
```bash
# Compare two encodings (requires original + references)
ffmpeg -i original.wav -i candidate1.mp3 -i candidate2.mp3 \
  -filter_complex "[1:a][2:a]ebur128=metadata=1" \
  -f null - 2>&1 | grep -i "I:\|Peak\|integrated"
```

---

## 13. IMPLEMENTATION CHECKLIST

### Mode Selection
- [ ] Determine use case: archival, streaming, mobile, voice
- [ ] Choose VBR for quality, CBR for streaming compatibility
- [ ] Select appropriate bitrate/quality level for target transparency

### Encoding Pipeline
- [ ] Validate encoder supports chosen rate control mode
- [ ] Check encoder availability: `ffmpeg -encoders | grep "libmp3lame\|libfdk\|libopus\|libvorbis"`
- [ ] Set correct quality/bitrate flag
- [ ] Monitor actual bitrate output for VBR modes
- [ ] Set reasonable buffer sizes for streaming

### Quality Verification
- [ ] Measure actual average bitrate: `ffprobe -show_entries format=bit_rate`
- [ ] Verify bitrate distribution for VBR: check for unexpected spikes
- [ ] Test with critical samples (transients, complex passages)
- [ ] Compare with transparent reference bitrates

## 13. ADVANCED RATE CONTROL STRATEGIES

### 13.1 Per-Frame Bit Allocation

Audio codecs allocate bits to individual frames based on perceptual importance.

```
┌─────────────────────────────────────────────────────────────────┐
│              FRAME-LEVEL BIT ALLOCATION                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  FRAME N: Transient (percussive)                                │
│  Bit allocation: High — peak bitrate exceeded                    │
│                                                                   │
│  FRAME N+1: Stationary (sustained tone)                        │
│  Bit allocation: Moderate — below average bitrate                │
│                                                                   │
│  FRAME N+2: Silence/quasi-silence                                │
│  Bit allocation: Very low — well below average                    │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 13.2 VBR Bitrate Variability

| Content Type | Bitrate Variability | Peak-to-Average Ratio |
|-------------|-------------------|----------------------|
| Classical music | High | ~2.5x average |
| Jazz | High | ~2.2x average |
| Rock/Pop | Moderate | ~1.8x average |
| Electronic | High (bass) | ~2.0x average |
| Speech | Low | ~1.3x average |
| Audiobook | Very low | ~1.1x average |

### 13.3 Constrained VBR (CVBR)

CVBR constrains bitrate to a maximum while allowing quality to vary within that constraint. Useful for streaming with bandwidth limits.

```bash
# Opus CVBR: targets 128k but allows some fluctuation
ffmpeg -i input.wav -c:a libopus -vbr cvbr -b:a 128k output.opus

# AAC: constrained bitrate with quality floor
ffmpeg -i input.wav -c:a libfdk_aac -b:a 192k output.m4a
```

### 13.4 Bit Reservoir (MP3)

MP3 uses a bit reservoir to smooth bit allocation. Complex frames borrow bits from future simple frames:

```
Frame 1: Complex → uses 200k of 192k budget
Frame 2: Simple → uses 150k of 192k budget, saves 42k
Frame 3: Complex → borrows 50k from Frame 2 savings
Over time, average bitrate stays ~192k.
LAME's reservoir can accumulate up to 768 bytes of borrowed bits.
```

### 13.5 ABR Bit Allocation Strategy

ABR uses rate-distortion optimization to allocate bits per frame:

```c
for each frame:
    complexity = measure_spectral_complexity(frame);
    target_bits = average_bitrate / frame_rate;
    if (complexity > threshold):
        allocated_bits = target_bits * complexity_factor;
    else:
        allocated_bits = target_bits * simplicity_factor;
    allocated_bits = clamp(allocated_bits, min, max);
    encode_frame(frame, allocated_bits);
```

---

## 14. QUALITY METRICS AND MEASUREMENT

### 14.1 Objective Quality Metrics

| Metric | Description | Range | Notes |
|--------|-------------|-------|-------|
| SNR | Signal-to-noise ratio | 0- infinity dB | Not perceptually accurate |
| Segmental SNR | SNR per segment | 0- infinity dB | Better for variable signals |
| PESQ | Perceptual Evaluation of Speech Quality | -0.5 to 4.5 | Speech-specific |
| POLQA | Perceptual Objective Listening Quality Assessment | 1 to 5 | Next-gen speech quality |
| PEAQ | Perceptual Evaluation of Audio Quality | -4 to 0 | Audio-specific |
| ODG | Objective Difference Grade | -4 to 0 | Based on PEAQ |

### 14.2 Psychoacoustic Quality Metrics

```bash
# EBU R128 loudness measurement
ffmpeg -i input.wav -af loudnorm=print_format=json -f null -

# Measure loudness stats
ffmpeg -i input.wav -af ebur128=metadata=1 -f null - 2>&1 | grep "I:"

# Basic SNR comparison
ffmpeg -i original.wav -i encoded.wav -filter_complex "[0:a][1:a]ssim" -f null -
```

### 14.3 ABX Testing for Rigorous Comparison

1. **A:** Original/reference file
2. **B:** Encoded file being tested
3. **X:** Randomly selected A or B
4. **Listener:** Identifies which is X

```bash
# Prepare test files (must match duration and sample rate)
ffmpeg -i original.wav -ar 44100 -ac 2 ref.wav
ffmpeg -i encoded.opus -ar 44100 -ac 2 encoded.wav

# Statistical significance: need 8/10 correct for p<0.05
# Use dedicated ABX tools: foobar2000, abx.sh, or web-based ABX
```

### 14.4 Bitrate vs Perceived Quality Curve

```
Quality (ODG)
  5 |                                           * Transparency
    |                                       *
  4 |                                  *
    |                             *
  3 |                        *  (diminishing returns)
    |                   *
  2 |              *
    |         *
  1 |     *
    | *
  0 +------------------------------------------------------- Bitrate
      32k   64k   96k   128k   192k   256k   320k
```

### 14.5 Spectral Analysis

```bash
# Generate spectrogram comparison
ffmpeg -i original.wav -lavfi "showspectrumpic=s=1920x1080" original_spec.png
ffmpeg -i encoded.opus -lavfi "showspectrumpic=s=1920x1080" encoded_spec.png

# FFT-based frequency display
ffplay -f lavfi "amovie=input.wav,showfreqs=mode=line:fscale=log"
```

---

## 15. STREAMING-SPECIFIC CONSIDERATIONS

### 15.1 Protocols and Bitrate Requirements

| Protocol | Typical Audio Bitrate | Notes |
|----------|----------------------|-------|
| HLS | 64-192 kbps | Adaptive bitrate |
| DASH | 64-256 kbps | Adaptive bitrate |
| RTMP | 64-192 kbps | Fixed bandwidth |
| Icecast/Shoutcast | 64-256 kbps | MP3/Ogg Vorbis |
| LL-HLS | 32-192 kbps | Low-latency variant |
| WebRTC | <500ms latency | Real-time |

### 15.2 ABR Streaming with FFmpeg

```bash
# Generate multiple quality levels
for BITRATE in 64k 128k 192k 256k; do
  ffmpeg -i input.wav \
    -c:a libopus -vbr on -b:a "$BITRATE" \
    "stream_${BITRATE}.m3u8"
done
```

### 15.3 Buffering and VBR

VBR requires larger buffers than CBR because peak bitrate can exceed average:

```
Example:
  Average: 128 kbps, Peak: 256 kbps (2x)
  Buffer for 10s: 256 kbps x 10s = 320 KB
vs. CBR 128 kbps x 10s = 160 KB
VBR requires ~2x the buffer of CBR.
```

### 15.4 Low-Latency Streaming

| Mode | Latency | Bitrate Strategy |
|------|---------|------------------|
| HLS | 6-30s | CBR preferred |
| DASH | 2-10s | VBR acceptable |
| LL-HLS | 0.5-2s | Low-bitrate VBR |
| WebRTC | <500ms | CBR/VBR low-bitrate |

```bash
# Low-latency Opus streaming
ffmpeg -i input.wav -c:a libopus \
  -vbr on -b:a 48k \
  -frame_duration 20 \
  -application lowdelay \
  -f rtp "rtp://localhost:5004"
```

---

## 16. PER-GENRE BITRATE REQUIREMENTS

### 16.1 Genre Analysis

| Genre | Characteristics | Minimum | Recommended |
|-------|----------------|---------|-------------|
| Piano solo | Sparse, dynamic range | 192 kbps | 256 kbps |
| Orchestral classical | Dense harmonics | 256 kbps | 320 kbps |
| Jazz combo | Transient-rich | 192 kbps | 256 kbps |
| Rock/Pop | Dense mix | 192 kbps | 256 kbps |
| Electronic/EDM | Synthetic, bass | 160 kbps | 192 kbps |
| Hip-hop | Bass-heavy vocals | 160 kbps | 192 kbps |
| Metal | Dense distorted guitars | 192 kbps | 256 kbps |
| Vocal/Acoustic | Sparse, voice | 128 kbps | 192 kbps |
| Audiobook/Speech | Monotone | 32-64 kbps | 96 kbps |

### 16.2 Critical Test Samples

| Sample | Why Difficult | Encoder Weakness |
|--------|--------------|-----------------|
| Castanets | High transients | Pre-echo in MP3 |
| Harpsichord | Complex harmonics | Spectral smearing |
| Glass harmonica | High-frequency sibilance | All encoders show artifacts |
| Pitch pipe | Pure sine wave | Quantization noise |
| Drum solo | Rapid transients | Bandwidth allocation |

---

## 17. C API: ADVANCED RATE CONTROL

### 17.1 Custom Rate Control Setup

```c
#include <libavcodec/avcodec.h>
#include <libavutil/opt.h>

AVCodecContext *ctx = avcodec_alloc_context3(codec);

// Configure bitrate
ctx->bit_rate = 192000;
ctx->rc_min_rate = 64000;
ctx->rc_max_rate = 384000;
ctx->rc_buffer_size = 0;

// VBR mode
ctx->global_quality = 0;
av_opt_set_int(ctx, "vbr", 4, AV_OPT_SEARCH_CHILDREN);
```

### 17.2 Frame-Level Quality Override

```c
AVFrame *frame = av_frame_alloc();
frame->format = ctx->sample_fmt;
frame->nb_samples = ctx->frame_size;
av_frame_get_buffer(frame, 0);

// Override quality for this frame
frame->quality = ctx->global_quality;

avcodec_send_frame(ctx, frame);
```

---

## 18. BITRATE MATH REFERENCE

### 18.1 Core Calculations

```bash
# File size = (bitrate x duration) / 8
# 3-min song at 192kbps: (192000 x 180) / 8 = 4.3 MB

# Raw PCM bitrate = sample_rate x channels x (bit_depth / 8)
# Stereo 44.1kHz 16-bit: 44100 x 2 x 2 = 1,411,200 bps
# 7.1 48kHz 32-bit float: 48000 x 8 x 4 = 12,288,000 bps
```

### 18.2 Compression Ratios

| Format | Ratio | Effective Bitrate |
|--------|--------|------------------|
| Uncompressed WAV | 1:1 | 1,411 kbps |
| FLAC level 8 | ~2.8:1 | ~500 kbps |
| MP3 192kbps | ~7.4:1 | 192 kbps |
| AAC 128kbps | ~11:1 | 128 kbps |
| Opus 64kbps | ~22:1 | 64 kbps |

---

## 19. EDGE CASES

### 19.1 Edge Case Handling

| Scenario | Issue | Solution |
|----------|-------|----------|
| Very short file (< delay) | Entire file is delay | Warn user |
| Multiple concatenated tracks | Each has own delay | Decode separately, trim, concatenate |
| Corrupt iTunSMPB | FFmpeg ignores | Fall back to default 1584/528 |
| Variable delay encoder | Delay varies per frame | Use maximum; trim excess |
| Non-iTunSMPB MP3 encoder | Unknown delay | Use LAME default 1584/528 |

---

## 20. IMPLEMENTATION CHECKLIST

### Mode Selection
- [ ] Determine use case: archival, streaming, mobile, voice
- [ ] Choose VBR for quality, CBR for streaming compatibility
- [ ] Select appropriate bitrate/quality level for target transparency

### Encoding Pipeline
- [ ] Validate encoder supports chosen rate control mode
- [ ] Check encoder availability: `ffmpeg -encoders | grep libfdk`
- [ ] Set correct quality/bitrate flag
- [ ] Monitor actual bitrate output for VBR modes
- [ ] Set reasonable buffer sizes for streaming

### Quality Verification
- [ ] Measure actual average bitrate: `ffprobe -show_entries format=bit_rate`
- [ ] Verify bitrate distribution for VBR
- [ ] Test with critical samples (transients, complex passages)
- [ ] Compare with transparent reference bitrates

### Edge Cases
- [ ] Handle CBR bitrate validation (must be in valid set)
- [ ] Handle VBR quality out-of-range (clamp to valid range)
- [ ] Handle encoder that does not support chosen mode (fallback)
- [ ] Handle very low bitrate scenarios (voice vs music)

## 20. BITRATE MATH REFERENCE

### 20.1 Core Calculations

```bash
# File size = (bitrate x duration) / 8
# 3-min song at 192kbps: (192000 x 180) / 8 = 4.3 MB
# Raw PCM bitrate = sample_rate x channels x (bit_depth / 8)
# Stereo 44.1kHz 16-bit: 44100 x 2 x 2 = 1,411,200 bps
```

### 20.2 Compression Ratios

| Format | Ratio | Effective Bitrate |
|--------|-------|-----------------|
| Uncompressed WAV | 1:1 | 1,411 kbps |
| FLAC level 8 | ~2.8:1 | ~500 kbps |
| MP3 192kbps | ~7.4:1 | 192 kbps |
| AAC 128kbps | ~11:1 | 128 kbps |
| Opus 64kbps | ~22:1 | 64 kbps |

## 21. DECODER BITRATE ANALYSIS

### 21.1 Measuring Actual Bitrate

```bash
# Get format-level bitrate
ffprobe -v error -select_streams a:0 -show_entries stream=bit_rate \
  -of default=noprint_wrappers=1:nokey=1 input.mp3

# Calculate from file size and duration
file_size=$(stat -c%s input.mp3)
duration=$(ffprobe -v error -show_entries format=duration \
  -of default=noprint_wrappers=1:nokey=1 input.mp3)
echo "Bitrate: $((file_size * 8 / duration / 1000)) kbps"
```

### 21.2 VBR Distribution Analysis

```bash
# Average bitrate
ffprobe -v quiet -select_streams a:0 -show_entries stream=bit_rate \
  -of csv=p=0 input.mp3 | awk '{s+=$1; n++} END {print s/n}'

# Bitrate histogram
ffprobe -v quiet -select_streams a:0 -show_entries stream=bit_rate \
  -of csv=p=0 input.mp3 | sort -n | uniq -c
```

## 22. ADVANCED FFmpeg C API

### 22.1 Complete Encode with Statistics

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>

int encode_with_stats(const char *input, const char *output, int bitrate) {
    AVFormatContext *ifmt = NULL, *ofmt = NULL;
    avformat_open_input(&ifmt, input, NULL, NULL);
    avformat_find_stream_info(ifmt, NULL);
    int idx = av_find_best_stream(ifmt, AVMEDIA_TYPE_AUDIO, -1, -1, NULL, 0);

    avformat_alloc_output_context2(&ofmt, NULL, "ogg", output);
    AVCodecContext *enc = avcodec_alloc_context3(
        avcodec_find_encoder_by_name("libopus"));
    enc->bit_rate = bitrate;
    enc->sample_rate = 48000;
    enc->ch_layout = (AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO;
    enc->sample_fmt = AV_SAMPLE_FMT_FLTP;
    avcodec_open2(enc, enc->codec, NULL);

    AVStream *ost = avformat_new_stream(ofmt, enc->codec);
    avcodec_parameters_from_context(ost->codecpar, enc);
    avio_open(&ofmt->pb, output, AVIO_FLAG_WRITE);
    avformat_write_header(ofmt, NULL);

    int64_t total_bytes = 0, frames = 0;
    AVPacket *pkt = av_packet_alloc();
    AVFrame *frame = av_frame_alloc();

    while (av_read_frame(ifmt, pkt) >= 0) {
        if (pkt->stream_index == idx) {
            avcodec_send_packet(enc, pkt);
            while (avcodec_receive_packet(enc, pkt) == 0) {
                av_interleaved_write_frame(ofmt, pkt);
                total_bytes += pkt->size;
                frames++;
                av_packet_unref(pkt);
            }
        }
        av_packet_unref(pkt);
    }
    avcodec_send_frame(enc, NULL);
    while (avcodec_receive_packet(enc, pkt) == 0) {
        av_interleaved_write_frame(ofmt, pkt);
        total_bytes += pkt->size;
        av_packet_unref(pkt);
    }
    av_write_trailer(ofmt);

    printf("Encoded %lld frames, %lld bytes\n", (long long)frames, (long long)total_bytes);
    av_frame_free(&frame);
    av_packet_free(&pkt);
    avcodec_free_context(&enc);
    avformat_free_context(ofmt);
    avformat_close_input(&ifmt);
    return 0;
}
```

### 22.2 Quality-Based Encoding Loop

```c
// Encode with quality targeting (VBR)
int encode_vbr(const char *input, const char *output, float quality) {
    // quality: 0.0 (best) to 1.0 (worst)

    AVCodecContext *ctx = /* setup encoder */;

    // LAME: 0-9, lower is better
    int lame_quality = (int)(quality * 9.0);
    av_opt_set_int(ctx, "q", 9 - lame_quality, AV_OPT_SEARCH_CHILDREN);

    // Vorbis: -1.0 to 10.0, higher is better
    double vorbis_quality = -1.0 + quality * 11.0;
    av_opt_set_double(ctx, "q", vorbis_quality, AV_OPT_SEARCH_CHILDREN);

    // Opus: bitrate target with VBR
    int opus_bitrate = (int)(320000 - quality * 300000); // 20k to 320k
    ctx->bit_rate = opus_bitrate;
    av_opt_set_int(ctx, "vbr", 1, AV_OPT_SEARCH_CHILDREN);

    // Encode loop...
    return 0;
}
```

## 23. APPENDIX: CODEC PROFILES AND LEVELS

### 23.1 AAC Profiles

| Profile | Full Name | Bitrate Range | Typical Use |
|---------|-----------|--------------|-------------|
| AAC-LC | Low Complexity | 8-320 kbps | General audio |
| HE-AAC | High Efficiency | 16-128 kbps | Low bitrate |
| HE-AAC v2 | High Efficiency v2 | 8-64 kbps | Very low bitrate |
| AAC-LD | Low Delay | 32-256 kbps | Interactive |
| AAC-ELD | Enhanced Low Delay | 32-128 kbps | Communication |
| xHE-AAC | Extended HE-AAC | 12-64 kbps | Unified speech/music |

### 23.2 Opus Application Modes

| Mode | Application | Frame Duration | Use Case |
|------|-------------|---------------|----------|
| `voip` | Voice over IP | 10-60ms | Telephony |
| `audio` | General audio | 10-120ms | Music |
| `lowdelay` | Low delay | 2.5-20ms | Real-time |

### 23.3 MP3 Frame Sizes

| Bitrate | Frame Size (44.1kHz) | Frame Duration |
|---------|---------------------|---------------|
| 128 kbps | 417 bytes | 26.1 ms |
| 192 kbps | 626 bytes | 26.1 ms |
| 256 kbps | 835 bytes | 26.1 ms |
| 320 kbps | 1044 bytes | 26.1 ms |

MP3 frames always contain 1152 PCM samples. Frame size = (144 x bitrate) / sample_rate.

## 24. ENERGY EFFICIENCY REFERENCE

### 24.1 Encoding Power Consumption by Mode

| Encoding Mode | CPU Usage | Power Draw | Use Case |
|--------------|----------|-----------|---------|
| FLAC level 0 | Low | ~5W | Real-time, low-power |
| FLAC level 8 | Medium | ~15W | Batch encoding |
| MP3 CBR | Medium | ~12W | Streaming server |
| MP3 VBR | High | ~18W | High-quality encoding |
| Opus | Medium | ~10W | Efficient, modern |
| AAC | High | ~20W | High-quality |

### 24.2 Storage Efficiency by Format

| Format | Size (3-min song) | Storage per 1000 songs | Energy per year |
|--------|------------------|----------------------|----------------|
| FLAC (level 8) | ~30 MB | 30 GB | ~5 kWh |
| Opus 128k | ~2.9 MB | 2.9 GB | ~0.5 kWh |
| MP3 192k VBR | ~4.3 MB | 4.3 GB | ~0.7 kWh |
| AAC 256k | ~5.7 MB | 5.7 GB | ~1 kWh |

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete FFmpeg rate control reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*

## 25. GLOSSARY

| Term | Definition |
|------|------------|
| ABR | Average Bitrate — encoding mode targeting average bitrate |
| CBR | Constant Bitrate — fixed bitrate throughout the file |
| CRF | Constant Rate Factor — quality-based encoding mode |
| VBR | Variable Bitrate — bitrate varies based on content complexity |
| CVBR | Constrained VBR — VBR with a maximum bitrate cap |
| MDCT | Modified Discrete Cosine Transform — core transform in many codecs |
| Transparency | Perceptually indistinguishable from original |
| Psy模型的 | Psychoacoustic model — perceptual masking analysis |
| Bit reservoir | MP3 mechanism allowing temporary bit borrowing |
| Frame | Fixed-size chunk of encoded audio samples |

---
