# FFmpeg CLI Audio Conversion Reference — Deep Technical Reference

> **Category:** FFmpeg API
> **File Extensions:** (applies to all audio formats)
> **MIME Types:** `audio/*` (all audio MIME types)
> **Standardization Body:** FFmpeg Project / FFmpeg.org
> **Primary Specification:** https://ffmpeg.org/ffmpeg.html, https://ffmpeg.org/ffmpeg-codecs.html, https://ffmpeg.org/ffmpeg-resampler.html
> **Patent Status:** FFmpeg is LGPL/GPL; individual codec encoders carry their own licenses
> **License:** LGPL 2.1+ (core) + per-codec licenses (libmp3lame: LGPL, libfdk-aac: Fraunhofer proprietary, libopus: BSD, libvorbis: BSD)
> **Current Version:** Active development (rolling release)
> **Active Development:** Yes — rolling release

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Fabrice Bellard and the FFmpeg developer community
- **Year Created:** 2000
- **Original Purpose:** Provide a complete, open-source solution for audio/video recording, conversion, and streaming
- **Problem with Predecessors:** Prior to FFmpeg, audio conversion required separate, proprietary tools for each codec (lame for MP3, oggenc for Vorbis, faac for AAC, etc.). FFmpeg unified all codecs under one CLI and one C API.

### 1.2 Version History
|| Version | Year | Key Changes |
|---------|------|-------------|
| 0.1 | 2000 | Initial release, basic format support |
| 0.4 | 2002 | Added libavcodec, libavformat architecture |
| 0.5 | 2004 | Multi-threading, many codec additions |
| 1.0 | 2013 | API stabilization, semantic versioning |
| 2.0 | 2013 | Libavcodec/libavformat split stabilized |
| 4.0 | 2018 | NVIDIA NVENC/NVDEC, VAAPI, Telestrator |
| 5.0 | 2022 | Bitstream filter API rewrite, hwcontext |
| 6.0 | 2023 | scene-sad assembly optimization, hwdevice API |
| 7.0 | 2024 | Enhanced filter graph threading, improved audio filters |

### 1.3 Current Adoption
- **Primary use cases today:** Audio transcoding, streaming (HLS/DASH), broadcast encoding, archival pipeline, research, mobile app backends, server-side media processing
- **Platforms with native support:** Linux, Windows, macOS, Android, iOS, BSD, embedded systems
- **Major services using FFmpeg:** YouTube, Twitch, Netflix, Spotify (backend), most streaming services
- **Hardware support:** NVENC/NVDEC (NVIDIA), QSV (Intel), VAAPI (Linux), VideoToolbox (macOS), MediaCodec (Android), AMC (Raspberry Pi)
- **Status:** Dominant open-source multimedia framework

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 FFmpeg Architecture — Component Stack

```
┌─────────────────────────────────────────────────┐
│              ffmpeg CLI Tool                    │
├─────────────────────────────────────────────────┤
│     libavformat   │   libavcodec  │ libavfilter │
│   (muxing/demux) │ (encode/decode)│(audio/video)│
├─────────────────────────────────────────────────┤
│           libswresample  │  libavutil         │
│        (resampling/mixing)│ (core utilities)   │
├─────────────────────────────────────────────────┤
│         libpostproc │  libavdevice            │
│      (post-processing)│ (device I/O)           │
└─────────────────────────────────────────────────┘
```

### 2.2 FFmpeg Processing Pipeline for Audio

```
Input File(s)
     │
     ▼
avformat_open_input() ─── demuxer reads container
     │                      parses streams, metadata
     ▼
av_read_frame() ──── returns AVPacket (encoded frame)
     │
     ▼
avcodec_send_packet() ── feeds packet to decoder
     │
     ▼
avcodec_receive_frame() ── returns AVFrame (decoded PCM)
     │
     ▼
[Optional: libswresample] ─── resample, remap, convert format
     │
     ▼
avcodec_send_frame() ──── feeds PCM to encoder
     │
     ▼
avcodec_receive_packet() ── returns encoded AVPacket
     │
     ▼
avformat_write_header() ─── mux into output container
     │
     ▼
av_interleaved_write_frame() ── write packets
     │
     ▼
av_write_trailer() ─── finalize container
     │
     ▼
Output File(s)
```

### 2.3 Key Design Parameters
|| Parameter | Value | Notes |
|-----------|-------|-------|
| Audio sample formats | u8, s16, s32, flt, dbl, u8p, s16p, s32p, fltp, dblp | Planar and interleaved variants |
| Max sample rate | 384000 Hz | Hardware-dependent |
| Max channels | 64 | Via channel layouts |
| Frame sizes | Codec-dependent | MP3: 1152, AAC: 1024, Opus: variable |
| Threading | Per-frame, per-stream, filter-graph | `-thread` flag |
| Hardware acceleration | cuda, qsv, vaapi, videotoolbox, opencl, mediacodec | Per-platform |

---

## 3. CORE CLI FLAGS — COMPLETE AUDIO REFERENCE

### 3.1 Essential Audio Encoding Flags

#### `-c:a` (Codec Selection)
Selects the audio encoder or decoder. Accepts codec names or special values:

| Codec Name | Type | Description | Notes |
|------------|------|-------------|-------|
| `libmp3lame` | encoder | MP3 via LAME | Requires `--enable-libmp3lame` |
| `libvorbis` | encoder | Vorbis via libvorbis | Requires `--enable-libvorbis` |
| `libopus` | encoder | Opus via libopus | Requires `--enable-libopus` |
| `libfdk_aac` | encoder | AAC via Fraunhofer FDK | Requires `--enable-libfdk-aac --enable-nonfree`; NOT in vanilla builds |
| `aac` | encoder | Native FFmpeg AAC encoder | Built-in; lower quality than libfdk_aac |
| `alac` | encoder | Apple Lossless | Built-in, lossless |
| `flac` | encoder | FLAC | Built-in, lossless |
| `pcm_s16le` | encoder | 16-bit signed PCM | Uncompressed |
| `pcm_s24le` | encoder | 24-bit signed PCM | Uncompressed |
| `pcm_s32le` | encoder | 32-bit signed PCM | Uncompressed |
| `pcm_f32le` | encoder | 32-bit float PCM | Uncompressed |
| `copy` | special | Stream copy (no re-encode) | Only for output; skips decode/encode |
| `ac3` | encoder | Dolby Digital (AC-3) | Built-in |
| `eac3` | encoder | Dolby Digital Plus (E-AC-3) | Built-in |
| `libtwolame` | encoder | MP2 via TwoLAME | Requires `--enable-libtwolame` |
| `libshine` | encoder | MP3 via Shine | Fixed-point, ARM-optimized |
| `libspeex` | encoder | Speex | Requires `--enable-libspeex` |
| `libwavpack` | encoder | WavPack | Requires `--enable-libwavpack` |
| `wmav1` | encoder | Windows Media Audio 1 | Built-in |
| `wmav2` | encoder | Windows Media Audio 2 | Built-in |
| `adpcm_ima_wav` | encoder | IMA ADPCM | Built-in |
| `gsm` | encoder | GSM 6.10 | Built-in |

#### `-b:a` (Target Bitrate)
Sets the target bitrate in bits per second. Used for CBR and ABR encoding.

```bash
ffmpeg -i input.wav -c:a aac -b:a 192k output.m4a
ffmpeg -i input.wav -c:a libmp3lame -b:a 320k output.mp3   # CBR
ffmpeg -i input.wav -c:a libopus -b:a 128k output.opus     # bitrate is max for Opus
```

Valid bitrate values per codec:
|| Codec | Valid Bitrates (kbps) |
|--------|------------------------|
| MP3 (libmp3lame) | 8, 16, 24, 32, 40, 48, 64, 80, 96, 112, 128, 160, 192, 224, 256, 320 |
| AAC | 8–512 (encoder-dependent) |
| AC-3 | 32, 40, 48, 56, 64, 80, 96, 112, 128, 160, 192, 224, 256, 320, 384, 448, 512, 640 |
| Opus | 6–256 (mono: 6–128, stereo: 16–256) |
| Vorbis | 45–500 |
| WMA | 4.8–768 |
| MP2 | 32–384 |

#### `-q:a` (VBR Quality)
Sets quality-based variable bitrate for codecs that support it. Scale is codec-specific; lower values generally mean higher quality.

```bash
ffmpeg -i input.wav -c:a libmp3lame -q:a 2 output.mp3   # LAME VBR quality 2 (~190kbps)
ffmpeg -i input.wav -c:a libvorbis -q:a 6 output.ogg    # Vorbis quality 6
ffmpeg -i input.wav -c:a libopus -vbr on -b:a 0 output.opus  # Opus CRF (use -b:a for target)
```

#### `-compression_level` (Lossless Codec Compression)
Sets compression level for lossless codecs (FLAC, ALAC). Range is codec-dependent.

```bash
ffmpeg -i input.wav -c:a flac -compression_level 8 output.flac   # Maximum compression
ffmpeg -i input.wav -c:a alac output.m4a                         # ALAC compression (default)
ffmpeg -i input.wav -c:a flac -compression_level 0 output.flac  # Fastest encoding
```

#### `-ar` (Audio Sample Rate)
Sets the output audio sample rate in Hz. If not specified, FFmpeg uses the input sample rate.

```bash
ffmpeg -i input.wav -ar 44100 output.wav          # Resample to 44.1 kHz
ffmpeg -i input.wav -ar 48000 output.wav          # Resample to 48 kHz
ffmpeg -i input.wav -ar 96000 output.wav          # Upsample to 96 kHz
ffmpeg -i input.wav -ar 8000 -ac 1 output.wav     # Low-quality voice: 8kHz mono
```

Standard sample rates and their common uses:

| Sample Rate | Common Name | Use Case |
|-------------|-------------|----------|
| 8000 | Telephone | Voice, GSM |
| 11025 | — | Low-quality legacy audio |
| 16000 | Wideband | VoIP, speech |
| 22050 | — | AM radio quality |
| 32000 | — | Broadcast |
| 44100 | CD | Consumer audio |
| 48000 | Professional | Video, DVD, Blu-ray |
| 88200 | 2× CD | High-res |
| 96000 | — | High-res, DVD-Audio |
| 176400 | 4× CD | DXD |
| 192000 | — | High-res max |
| 384000 | — | Ultra-high-res |

#### `-ac` (Audio Channel Count)
Sets the output number of audio channels.

```bash
ffmpeg -i input.wav -ac 1 output.wav      # Mono
ffmpeg -i input.wav -ac 2 output.wav      # Stereo
ffmpeg -i input.wav -ac 6 output.wav      # 5.1 surround
ffmpeg -i input.wav -ac 8 output.wav      # 7.1 surround
```

#### `-channel_layout` / `-layout`
Sets the output channel layout explicitly. Use named layouts:

```bash
ffmpeg -i input.wav -channel_layout stereo output.wav
ffmpeg -i input.wav -channel_layout 5.1 output.wav
ffmpeg -i input.wav -channel_layout mono output.wav
ffmpeg -i input.wav -channel_layout downmix output.wav
```

Supported channel layouts:

| Layout Name | Channels | Order |
|-------------|----------|-------|
| `mono` | 1 | C |
| `stereo` | 2 | L R |
| `2.1` | 3 | L R LFE |
| `4.0` / `quad` | 4 | FL FR RL RR |
| `5.0` | 5 | FL FR FC RL RR |
| `5.1` | 6 | FL FR FC LFE RL RR |
| `6.1` | 7 | FL FR FC LFE RL RR RC |
| `7.1` | 8 | FL FR FC LFE RL RR RL RR |
| `7.1(wide)` | 8 | FL FR FC LFE BL BR SL SR |

#### `-sample_fmt` (Sample Format)
Sets the output audio sample format.

```bash
ffmpeg -i input.wav -sample_fmt s16le output.wav    # 16-bit signed LE
ffmpeg -i input.wav -sample_fmt s32le output.wav    # 32-bit signed LE
ffmpeg -i input.wav -sample_fmt fltp output.wav     # 32-bit float planar
```

Sample format naming convention: `[u][s][f]p[b[l][e]]`
- `u` = unsigned, `s` = signed (default)
- `f` = float, no prefix = integer
- `p` = planar (separate buffer per channel), no `p` = interleaved
- `b[l]` = big-endian, no `b` or `b` alone = platform native, `le` = little-endian

| Format | Bytes/Sample | Type | Notes |
|--------|-------------|------|-------|
| `u8` | 1 | Unsigned integer | |
| `s16` / `s16le` | 2 | Signed 16-bit | CD quality |
| `s16be` | 2 | Signed 16-bit BE | |
| `s32` / `s32le` | 4 | Signed 32-bit | |
| `s32be` | 4 | Signed 32-bit BE | |
| `flt` / `fltle` | 4 | 32-bit float | Range -1.0 to 1.0 |
| `fltbe` | 4 | 32-bit float BE | |
| `dbl` / `dblle` | 8 | 64-bit double | Range -1.0 to 1.0 |
| `dblbe` | 8 | 64-bit double BE | |
| `s16p` | 2 | Planar 16-bit | |
| `s32p` | 4 | Planar 32-bit | |
| `fltp` | 4 | Planar float | Required by most modern encoders |
| `dblp` | 8 | Planar double | |

#### `-frames:a` (Limit Output Frames)
Limits the number of audio frames to encode.

```bash
ffmpeg -i input.wav -frames:a 1000 output.mp3   # Encode only first 1000 frames
ffmpeg -i input.wav -t 30 -c:a flac output.flac  # Encode first 30 seconds
```

#### `-map_channel` (Channel Mapping)
Maps specific channels from input to output. Useful for extracting or rearranging channels.

```bash
# Extract left channel from stereo
ffmpeg -i stereo.wav -map_channel 0.0.0 left.wav

# Extract right channel from stereo
ffmpeg -i stereo.wav -map_channel 0.0.1 right.wav

# Create stereo from two mono files with explicit channel mapping
ffmpeg -i left.wav -i right.wav \
  -filter_complex "[0:a][1:a]amerge=inputs=2[aout]" \
  -map "[aout]" output.wav

# Swap left and right channels
ffmpeg -i stereo.wav -af "channelmap=map=R:L:FL=R:FR=L" output.wav
```

### 3.2 Audio Filter (`-af` / `-filter:a`) Reference

The `-af` flag applies an audio filtergraph. Simple filters use `-af`, complex graphs use `-filter_complex`.

#### Resampling Filters

| Filter | Description | Example |
|--------|-------------|---------|
| `aresample` | Audio resampling | `-af aresample=48000` |
| `aformat` | Convert sample format | `-af aformat=sample_fmts=fltp:sample_rates=48000` |
| `channelsplit` | Split channels | `-af "channelsplit=channel_layout=stereo"` |
| `channelmap` | Remap channels | `-af "channelmap=map=FL:FR:FC:C:LFE:SL:SR"` |
| `pan` | Custom channel mixing | `-af "pan=stereo|c0=c0+c1|c1=c0-c1"` |
| `aformat` | Format conversion | `-af aformat=s32p` |

```bash
# Resample to 48kHz with dithering
ffmpeg -i input.wav -af "aresample=48000,dither=srect" output.wav

# Convert to planar float for encoder
ffmpeg -i input.wav -af "aformat=sample_fmts=fltp:sample_rates=48000" output.wav

# Downmix 5.1 to stereo
ffmpeg -i input_51.wav -af "aformat=sample_fmts=fltp:channel_layouts=stereo" \
  -c:a libopus output.opus
```

#### Volume and Level Filters

| Filter | Description | Example |
|--------|-------------|---------|
| `volume` | Adjust volume | `-af "volume=0.5"` (half) or `+3dB` |
| `acompressor` | Dynamics compression | `-af "acompressor=threshold=0.5"` |
| `aexpander` | Dynamic expansion | `-af "aexpander=threshold=0.2"` |
| `alimiter` | Brickwall limiter | `-af "alimiter=limit=0.9"` |
| `alangdetec` | Language detection | `-af "alangdetec=en"` |
| `volumedetect` | Analyze volume | `-af volumedetect` (for stats) |

```bash
# Normalize to -1 dB
ffmpeg -i input.wav -af "volume=-1dB" output.wav

# Apply 2:1 compression with -12dB threshold
ffmpeg -i input.wav -af "acompressor=threshold=0.25:ratio=2:attack=5:release=50" output.wav

# Detect volume statistics
ffmpeg -i input.wav -af volumedetect -f null -
```

#### EQ and Tone Filters

| Filter | Description | Example |
|--------|-------------|---------|
| `aecho` | Add echo/reverb | `-af "aecho=0.8:0.88:60:0.4"` |
| `aequalizer` | Parametric EQ | `-af "aequalizer=f=1000:t=h:width=0.1:g=-6"` |
| `aphaser` | Phaser effect | `-af "aphaser=type=sine"` |
| `tremolo` | Tremolo effect | `-af "tremolo=f=5:d=0.5"` |
| `chorus` | Chorus effect | `-af "chorus=0.5:0.9:50:0.4:0.25:2"` |

#### Denoising Filters

| Filter | Description | Example |
|--------|-------------|---------|
| `anlmdn` | Non-local means denoising | `-af "anlmdn=s=7:m=5"` |
| `hqdn3d` | High-quality denoiser | `-af "hqdn3d=4:3:6:4.5"` |
| `afftfilt` | FFT filter | `-af "afftfilt=win_size=512:expr=0.5*(1-cos(2*PI*n/512))"` |

### 3.3 Per-Codec Encoder Selection

#### libmp3lame (MP3)
```bash
# VBR — recommended for quality
ffmpeg -i input.wav -c:a libmp3lame -q:a 2 output.mp3   # ~190kbps VBR
ffmpeg -i input.wav -c:a libmp3lame -q:a 0 output.mp3   # Highest VBR (~245kbps)

# CBR
ffmpeg -i input.wav -c:a libmp3lame -b:a 192k output.mp3

# ABR (average bitrate)
ffmpeg -i input.wav -c:a libmp3lame -b:a 192k -q:a 0 output.mp3
```

#### libvorbis (Ogg Vorbis)
```bash
# VBR quality mode (0–10, default 3)
ffmpeg -i input.wav -c:a libvorbis -q:a 6 output.ogg    # High quality (~192kbps)
ffmpeg -i input.wav -c:a libvorbis -q:a 10 output.ogg   # Maximum quality

# ABR bitrate mode
ffmpeg -i input.wav -c:a libvorbis -b:a 192k output.ogg
```

#### libopus (Opus)
```bash
# Default VBR (recommended)
ffmpeg -i input.wav -c:a libopus -b:a 128k output.opus

# Constrained VBR with bitrate cap
ffmpeg -i input.wav -c:a libopus -vbr on -b:a 128k -cutoff 20000 output.opus

# CBR (rarely needed)
ffmpeg -i input.wav -c:a libopus -vbr off -b:a 128k output.opus

# CVBR (constrained VBR)
ffmpeg -i input.wav -c:a libopus -vbr cvbr -b:a 128k output.opus
```

#### libfdk_aac (Fraunhofer AAC)
```bash
# VBR quality mode (1–5)
ffmpeg -i input.wav -c:a libfdk_aac -vbr 4 output.m4a   # ~128kbps VBR
ffmpeg -i input.wav -c:a libfdk_aac -vbr 5 output.m4a   # Highest quality

# ABR
ffmpeg -i input.wav -c:a libfdk_aac -b:a 256k output.m4a

# HE-AAC v2 (parametric stereo)
ffmpeg -i input.wav -c:a libfdk_aac -profile:a aac_he_v2 -b:a 48k output.m4a
```

#### Native FFmpeg AAC
```bash
# Note: native AAC encoder is lower quality than libfdk_aac
ffmpeg -i input.wav -c:a aac -b:a 192k output.m4a
ffmpeg -i input.wav -c:a aac -q:a 2 -profile:a aac_low output.m4a
```

Available native AAC profiles:
| Profile | Description | Notes |
|---------|-------------|-------|
| `aac_low` | AAC-LC (default) | Most compatible |
| `aac_he` | HE-AAC v1 | ~64kbps stereo |
| `aac_he_v2` | HE-AAC v2 | ~48kbps stereo |
| `aac_ld` | AAC-LD | Low-delay |
| `aac_eld` | AAC-ELD | Enhanced low-delay |

#### FLAC (Free Lossless Audio Codec)
```bash
# Maximum compression (default)
ffmpeg -i input.wav -c:a flac -compression_level 8 output.flac

# Fast encoding (level 0)
ffmpeg -i input.wav -c:a flac -compression_level 0 output.flac

# Medium compression (level 5)
ffmpeg -i input.wav -c:a flac -compression_level 5 output.flac
```

Compression levels:
| Level | Encode Speed | Compression | Notes |
|-------|-------------|-------------|-------|
| 0 | Fastest | Lowest | Use for real-time |
| 1–3 | Fast | Low | |
| 4–6 | Medium | Medium | |
| 7–8 | Slow | High | |
| 12 | Maximum | Highest | Default |

#### ALAC (Apple Lossless)
```bash
ffmpeg -i input.wav -c:a alac output.m4a
```

#### PCM (Uncompressed)
```bash
ffmpeg -i input.wav -c:a pcm_s16le output.wav    # 16-bit signed LE
ffmpeg -i input.wav -c:a pcm_s24le output.wav    # 24-bit signed LE
ffmpeg -i input.wav -c:a pcm_s32le output.wav    # 32-bit signed LE
ffmpeg -i input.wav -c:a pcm_f32le output.wav    # 32-bit float LE
ffmpeg -i input.wav -c:a pcm_f64le output.wav    # 64-bit float LE
```

---

## 4. LOSSLESS ENCODING GUIDE

### 4.1 FLAC
FLAC (Free Lossless Audio Codec) is the recommended lossless format for most use cases. It offers fast encoding/decoding, seeking support, and wide hardware compatibility.

```bash
# Maximum compression
ffmpeg -i input.wav -c:a flac -compression_level 8 output.flac

# Default (level 5)
ffmpeg -i input.wav -c:a flac output.flac

# Streamable (level 7, no seeking table)
ffmpeg -i input.wav -c:a flac -compression_level 7 -streamable 1 output.flac

# With MD5 checksum verification
ffmpeg -i input.wav -c:a flac -compression_level 8 -fletcher16 1 output.flac
```

### 4.2 ALAC
Apple Lossless is identical in quality to FLAC but limited to M4A containers. Use when targeting Apple ecosystems.

```bash
ffmpeg -i input.wav -c:a alac output.m4a
```

### 4.3 TAK (Tom's Lossless Audio Kompressor)
TAK requires an external TAK encoder. FFmpeg does not natively support TAK encoding. Use the standalone `takc` encoder, then wrap with FFmpeg:

```bash
# Encode with takc
takc -p4 input.wav output.tak

# Decode with FFmpeg
ffmpeg -i output.tak -c:a pcm_s32le output.wav
```

### 4.4 WavPack
WavPack is a hybrid lossless format that can also produce correction files.

```bash
# Pure lossless
ffmpeg -i input.wav -c:a libwavpack -compression_level 6 output.wv

# Hybrid mode (lossless + correction file)
ffmpeg -i input.wav -c:a libwavpack -compression_level 3 output.wv

# Decode
ffmpeg -i output.wv output.wav
```

### 4.5 WavPack Pipeline (for maximum compatibility)
```bash
# WavPack lossless with correction file for hybrid mode
ffmpeg -i input.wav -c:a pcm_s24le -f wav - | \
  wvunpack - -o output.wav

# Or decode back from WavPack
ffmpeg -i output.wv -c:a pcm_s32le decoded.wav
```

### 4.6 Lossless Bit-Exact Verification
```bash
# Step 1: Encode
ffmpeg -i original.wav -c:a flac -compression_level 8 output.flac

# Step 2: Decode back
ffmpeg -i output.flac -c:a pcm_s32le decoded.wav

# Step 3: Compare using MD5
md5sum original.wav decoded.wav

# Step 4: Using FFmpeg framemd5 for frame-level verification
ffmpeg -i original.wav -f framemd5 original.md5
ffmpeg -i output.flac -f framemd5 output.md5
diff original.md5 output.md5   # Empty diff = bit-perfect

# Step 5: Using sha256sum on raw PCM
ffmpeg -i original.wav -c:a pcm_s24le -f wav - | sha256sum
ffmpeg -i output.flac -c:a pcm_s24le -f wav - | sha256sum
```

---

## 5. HIGH-QUALITY LOSSY ENCODING

### 5.1 Per-Codec Recommended Settings for Transparency

| Codec | Recommended Setting | Expected Bitrate | Notes |
|-------|--------------------|------------------|-------|
| libopus | `-b:a 128k` (VBR on) | ~128 kbps | Transparent for most content |
| libfdk_aac | `-vbr 4` | ~128 kbps | Best AAC encoder |
| libvorbis | `-q:a 6` | ~192 kbps | Good transparency |
| libmp3lame | `-q:a 0` | ~245 kbps | Highest quality MP3 |
| native aac | `-b:a 256k` | ~256 kbps | Lower quality than libfdk |

### 5.2 libopus — Best Quality Settings
```bash
# Recommended for music
ffmpeg -i input.wav -c:a libopus \
  -vbr on \
  -b:a 128k \
  -frame_duration 20 \
  -application audio \
  output.opus

# For speech/voice
ffmpeg -i input.wav -c:a libopus \
  -vbr on \
  -b:a 48k \
  -frame_duration 60 \
  -application voip \
  output.opus

# For low-latency (gaming, live)
ffmpeg -i input.wav -c:a libopus \
  -vbr on \
  -b:a 96k \
  -frame_duration 10 \
  -application lowdelay \
  output.opus

# Bitrate-explicit constrained VBR
ffmpeg -i input.wav -c:a libopus \
  -vbr cvbr \
  -b:a 192k \
  -max_delay 4000000 \
  output.opus
```

### 5.3 libfdk_aac — Best Quality Settings
```bash
# High quality VBR
ffmpeg -i input.wav -c:a libfdk_aac -vbr 5 output.m4a

# ABR with target
ffmpeg -i input.wav -c:a libfdk_aac -b:a 256k -afterburner 1 output.m4a

# HE-AAC v2 for low bitrates
ffmpeg -i input.wav -c:a libfdk_aac -profile:a aac_he_v2 -vbr 3 output.m4a

# LD-ELD for low-delay applications
ffmpeg -i input.wav -c:a libfdk_aac -profile:a aac_eld -b:a 64k output.m4a
```

### 5.4 libvorbis — Best Quality Settings
```bash
# High quality VBR
ffmpeg -i input.wav -c:a libvorbis -q:a 6 output.ogg    # ~192kbps
ffmpeg -i input.wav -c:a libvorbis -q:a 8 output.ogg    # ~225kbps
ffmpeg -i input.wav -c:a libvorbis -q:a 10 output.ogg   # ~256kbps

# ABR mode
ffmpeg -i input.wav -c:a libvorbis -b:a 256k output.ogg
```

### 5.5 libmp3lame — Best Quality Settings
```bash
# VBR highest quality
ffmpeg -i input.wav -c:a libmp3lame -q:a 0 output.mp3    # ~245kbps VBR

# VBR transparent quality (recommended)
ffmpeg -i input.wav -c:a libmp3lame -q:a 2 output.mp3    # ~190kbps VBR

# CBR 320kbps (maximum bitrate)
ffmpeg -i input.wav -c:a libmp3lame -b:a 320k output.mp3

# ABR with quality hint
ffmpeg -i input.wav -c:a libmp3lame -b:a 192k -q:a 2 output.mp3
```

### 5.6 Waveform Representation and Dithering
When converting from high bit-depth sources to lower bit-depth outputs (e.g., 24-bit to 16-bit), dithering is essential to avoid quantization distortion.

```bash
# Explicit dithering with aformat
ffmpeg -i input_24bit.wav -af "aformat=s16p" -c:a flac output.flac

# Dither with noise shaping (for quiet passages)
ffmpeg -i input_24bit.wav -af "aformat=sample_fmts=s16p:dither_method=shaped_via_a" output.wav

# Available dither methods:
#   rectangular / none / triangular / triangular_hp / lipshitz / shibata / low_shibata / high_shibata / weighted / modified_e / improved_e / uniform
```

---

## 6. BATCH CONVERSION & PERFORMANCE

### 6.1 Hardware Acceleration
```bash
# CUDA (NVIDIA) — decode + encode
ffmpeg -hwaccel cuda -i input.mov -c:a pcm_s16le output.wav

# VAAPI (Linux Intel/AMD)
ffmpeg -hwaccel vaapi -hwaccel_device /dev/dri/renderD128 \
  -i input.mov -c:a pcm_s16le output.wav

# Quick Sync Video (Intel)
ffmpeg -hwaccel qsv -i input.mov -c:a pcm_s16le output.wav

# VideoToolbox (macOS)
ffmpeg -hwaccel videotoolbox -i input.mov -c:a pcm_s16le output.wav
```

### 6.2 Threading
```bash
# Single-threaded
ffmpeg -i input.wav -c:a flac -threads 1 output.flac

# Auto-detect (default)
ffmpeg -i input.wav -c:a flac -threads 0 output.flac

# Explicit thread count
ffmpeg -i input.wav -c:a flac -threads 8 output.flac

# Filter threading (for complex filtergraphs)
ffmpeg -i input.wav -af "superequalizer=2b=1" -c:a libopus -threads 4 output.opus
```

### 6.3 Progress Reporting
```bash
# Pipe progress to terminal
ffmpeg -i input.wav -c:a flac -compression_level 8 -progress pipe:1 output.flac

# Pipe progress to file
ffmpeg -i input.wav -c:a flac -compression_level 8 -progress progress.txt output.flac

# Parse progress (for applications)
ffmpeg -i input.wav -c:a flac -progress pipe:1 -nostats output.flac 2>/dev/null | \
  while read line; do
    case "$line" in
      frame=*) frame="${line#frame=}";;
      fps=*) fps="${line#fps=}";;
      time=*) time="${line#time=}";;
      bitrate=*) bitrate="${line#bitrate=}";;
      speed=*) speed="${line#speed=}";;
    esac
    echo "Frame: $frame | FPS: $fps | Time: $time | Bitrate: $bitrate | Speed: $speed"
  done
```

### 6.4 Multiple Input Files
```bash
# Concat two files (files must have same format)
ffmpeg -i "concat:input1.wav|input2.wav" -c:a flac output.flac

# With intermediate filter
ffmpeg -i input1.wav -i input2.wav \
  -filter_complex "[0:a][1:a]concat=n=2:v=0:a=1[outa]" \
  -map "[outa]" -c:a flac output.flac

# Process all files in directory (bash)
for f in *.wav; do
  ffmpeg -i "$f" -c:a libopus -b:a 128k "${f%.wav}.opus"
done

# Parallel batch processing
find . -name "*.wav" -print0 | xargs -0 -P 4 -I{} \
  ffmpeg -i {} -c:a flac -compression_level 8 {}.flac
```

---

## 7. STREAM SELECTION

### 7.1 Stream Specifiers
Stream specifiers follow options and are separated by colon: `-codec:a:1` selects the second audio stream.

| Specifier | Meaning |
|-----------|---------|
| `:a` | All audio streams |
| `:a:0` | First audio stream |
| `:a:1` | Second audio stream |
| `:a:mime:audio/aac` | Streams by MIME type |
| `:a:language=eng` | Streams by language |
| `:a:title="Commentary"` | Streams by title metadata |

```bash
# Select first audio stream
ffmpeg -i input.mkv -map 0:a:0 -c:a copy output.mka

# Select English audio
ffmpeg -i input.mkv -map 0:a:language=eng -c:a copy output.mka

# Select by MIME type (for obscure formats)
ffmpeg -i input.mp4 -map 0:a:mime:audio/aac -c:a copy output.aac

# Multiple stream selection
ffmpeg -i input.mkv \
  -map 0:a:0 -map 0:a:1 \
  -c:a:0 libopus -b:a:0 128k \
  -c:a:1 libopus -b:a:1 64k \
  output_eng.opus output_ commentary.opus
```

### 7.2 Stream Copy
Use `-c:a copy` to skip re-encoding entirely (bit-exact copy).

```bash
# Copy audio stream from MP4 to MKV
ffmpeg -i input.mp4 -map 0:a -c:a copy output.mka

# Copy audio, re-encode video
ffmpeg -i input.mp4 -map 0:v -map 0:a -c:v libx264 -c:a copy output.mkv

# Copy audio with metadata update
ffmpeg -i input.mp4 -map 0:a -c:a copy \
  -metadata title="Track Title" \
  output.mka
```

---

## 8. CONVERSION VERIFICATION

### 8.1 Probe Metadata
```bash
# Full stream information
ffprobe -v quiet -print_format json -show_streams -show_format input.flac | jq .

# Audio stream details only
ffprobe -v error -show_entries stream=codec_name,sample_rate,channels,bit_rate \
  -of default=noprint_wrappers=1 input.flac

# Show all metadata tags
ffprobe -v quiet -print_format json -show_format input.flac | jq .format.tags

# Show chapter information
ffprobe -i input.flac -show_chapters

# Probe codec capabilities
ffmpeg -encoders 2>/dev/null | grep -i "^ A"
ffmpeg -decoders 2>/dev/null | grep -i "^ A"
```

### 8.2 Frame-Level Checksumming
```bash
# Generate framemd5 for bit-exact comparison
ffmpeg -i original.wav -f framemd5 original.md5
ffmpeg -i encoded.flac -f framemd5 encoded.md5

# Compare
diff original.md5 encoded.md5 && echo "BIT-IDENTICAL" || echo "DIFFERENT"

# Verify against known good
ffmpeg -i output.flac -f framemd5 output.md5
md5sum -c reference.md5
```

### 8.3 Bitrate Analysis
```bash
# Report frame sizes and bitrate over time
ffmpeg -i input.mp3 -b:a 192k -f bitstreamout - | \
  ./analyze_bitstream.py

# Check actual bitrate with ffprobe
ffprobe -v error -show_entries format=bit_rate:stream=bit_rate \
  -of default=noprint_wrappers=1 input.mp3

# Analyze VBR quality distribution
ffprobe -v error -select_streams a:0 -show_entries stream=bit_rate \
  -of csv=p=0 input.mp3 | sort -u
```

### 8.4 Audio Quality Metrics
```bash
# Using ffmpeg's internal analysis
ffmpeg -i input.wav -af "astat=metadata=1:reset=1" -f null -

# Compute SNR (Signal-to-Noise Ratio) by comparing with original
ffmpeg -i original.wav -i encoded.wav -filter_complex \
  "[0:a][1:a]abconsistency=metadata=1:file=/dev/stdout" \
  -f null - 2>&1 | grep -i snr

# VMAF (Video Multimethod Assessment Fusion) for audio [NEEDS VERIFICATION]
ffmpeg -i original.wav -i encoded.opus -filter_complex \
  "[0:a][1:a]libvmaf=model=audio/vmaf_v0.3.7b.json" \
  -f null - 2>&1 | grep VMAF
```

---

## 9. KNOWN CODEC AVAILABILITY DIFFERENCES

### 9.1 Codecs Requiring External Libraries

| Codec | Library | License | Configure Flag | Not in Vanilla |
|-------|---------|---------|---------------|----------------|
| libmp3lame | LAME | LGPL | `--enable-libmp3lame` | No — often included |
| libfdk_aac | Fraunhofer FDK | Proprietary | `--enable-libfdk-aac --enable-nonfree` | YES — requires custom build |
| libopus | libopus | BSD | `--enable-libopus` | Usually included |
| libvorbis | libvorbis | BSD | `--enable-libvorbis` | Usually included |
| libspeex | Speex | BSD | `--enable-libspeex` | No |
| libwavpack | WavPack | BSD | `--enable-libwavpack` | No |
| libtwolame | TwoLAME | GPL | `--enable-libtwolame` | No |
| libshine | Shine | GPL | `--enable-libshine` | No |
| libwebp | WebP | BSD | `--enable-libwebp` | No |
| libx264 | x264 | GPL | `--enable-libx264` | No |
| libx265 | x265 | Commercial | `--enable-libx265` | No |

### 9.2 Detecting Available Encoders
```bash
# List all audio encoders
ffmpeg -encoders 2>/dev/null | grep "^ A"

# Check for specific encoder
ffmpeg -encoders 2>/dev/null | grep -i "fdk\|libfdk"

# List all audio decoders
ffmpeg -decoders 2>/dev/null | grep "^ A"

# Check encoder capabilities
ffmpeg -h encoder=libfdk_aac
ffmpeg -h encoder=libopus
ffmpeg -h encoder=flac
```

### 9.3 Cross-Platform Build Notes
- **macOS:** FFmpeg from Homebrew includes all encoders including libfdk-aac
- **Windows:** Builds from ffmpeg.org or Zeranoe include most encoders; libfdk-aac requires custom build
- **Linux (Debian/Ubuntu):** `apt install ffmpeg` includes most codecs; `libfdk-aac-dev` package needed for libfdk-aac
- **Linux (RPM):** RPMFusion builds include most codecs
- **Android/iOS:** Cross-compile with NDK or use mobile-optimized FFmpeg forks

---

## 10. COMPLETE CLI REFERENCE TABLES

### 10.1 Global Flags (Apply to Entire Command)

| Flag | Description | Example |
|------|-------------|---------|
| `-i <file>` | Input file | `-i input.wav` |
| `-y` | Overwrite output without asking | `-y` |
| `-n` | Do not overwrite output | `-n` |
| `-v <level>` | Set verbosity | `-v quiet`, `-v error`, `-v debug` |
| `-loglevel <level>` | Same as -v | `-loglevel info` |
| `-stats` | Show encoding progress | `-stats` |
| `-progress <url>` | Write progress to file/pipe | `-progress pipe:1` |
| `-benchmark` | Show benchmark results | `-benchmark` |
| `-threads <n>` | Set thread count | `-threads 8` |
| `-filter_threads <n>` | Filter thread count | `-filter_threads 4` |
| `-filter_script <file>` | Filtergraph from file | `-filter_script filters.txt` |
| `-re` | Read input at native frame rate | `-re` (for live input) |
| `-stream_loop <n>` | Loop input n times | `-stream_loop 3` |
| `-f <format>` | Force format (input or output) | `-f wav` |

### 10.2 Per-Stream Audio Flags

| Flag | Description | Example |
|------|-------------|---------|
| `-c:a <codec>` | Audio codec | `-c:a libopus` |
| `-b:a <rate>` | Audio bitrate | `-b:a 192k` |
| `-q:a <quality>` | Audio VBR quality | `-q:a 4` |
| `-ar <rate>` | Audio sample rate | `-ar 48000` |
| `-ac <n>` | Audio channels | `-ac 2` |
| `-af <filters>` | Audio filters | `-af "volume=2"` |
| `-sample_fmt <fmt>` | Audio sample format | `-sample_fmt fltp` |
| `-channel_layout <layout>` | Audio channel layout | `-channel_layout 5.1` |
| `-frames:a <n>` | Limit audio frames | `-frames:a 1000` |
| `-map_channel <spec>` | Map specific channel | `-map_channel 0.0.0` |
| `-metadata:s:a <kv>` | Stream metadata | `-metadata:s:a title="Audio"` |
| `-disposition:a <disp>` | Stream disposition | `-disposition:a default` |

### 10.3 Metadata Flags

| Flag | Description | Example |
|------|-------------|---------|
| `-metadata <k=v>` | Global metadata | `-metadata title="Song"` |
| `-metadata:s:a:0 <k=v>` | Per-stream metadata | `-metadata:s:a:0 title="Track"` |
| `-map_metadata <in>` | Copy metadata from input | `-map_metadata 0` |
| `-map_metadata:s:v <in>` | Copy video stream metadata | `-map_metadata:s:v 0` |
| `-map_metadata:s:a <in>` | Copy audio stream metadata | `-map_metadata:s:a 0` |
| `-movflags +gaplessinfo` | MP4 gapless info | `-movflags +gaplessinfo` |
| `-id3v2_version 4` | ID3v2 version | `-id3v2_version 4` |

---

## 11. STREAM SELECTION EXAMPLES

### 11.1 Basic Stream Selection
```bash
# Copy first audio stream (default behavior)
ffmpeg -i input.mkv -c:a copy output.mkv

# Explicitly select first audio stream
ffmpeg -i input.mkv -map 0:a:0 -c:a copy output.mkv

# Select all audio streams
ffmpeg -i input.mkv -map 0:a -c:a copy output.mkv
```

### 11.2 Complex Multi-Track Scenarios
```bash
# 5.1 surround to stereo downmix with metadata
ffmpeg -i input_51.wav \
  -af "aformat=sample_fmts=fltp:channel_layouts=stereo" \
  -c:a libopus -b:a 128k \
  -metadata title="Stereo Mix" \
  output_stereo.opus

# Extract multiple commentary tracks
ffmpeg -i input.mkv \
  -map 0:a:0 -map 0:a:1 \
  -c:a:0 libopus -b:a:0 96k commentary_eng.opus \
  -c:a:1 libopus -b:a:1 96k commentary_spa.opus

# Reorder channels (5.1 → stereo + LFE channel)
ffmpeg -i input_51.wav \
  -af "pan=stereo|c0=FL|c1=FR,amerge" \
  -c:a libopus -b:a 128k output_stereo.opus
```

### 11.3 Stream Selection by Metadata
```bash
# Select audio by language tag
ffmpeg -i input.mkv \
  -map 0:a:language=jpn \
  -map 0:a:language=eng \
  -c:a:0 libopus -b:a:0 128k \
  -c:a:1 libopus -b:a:1 128k \
  output_jpn.opus output_eng.opus

# Select audio by title
ffmpeg -i input.mkv \
  -map 0:a:title="Director's Commentary" \
  -c:a copy \
  commentary.mka
```

---

## 12. FFmpeg C API — QUICK REFERENCE

### 12.1 Audio Encoding Skeleton
```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libavutil/opt.h>
#include <libswresample/swresample.h>

// ─── 1. Find encoder ─────────────────────────────────────────────────────────
const AVCodec *codec = avcodec_find_encoder_by_name("libopus");
if (!codec) { /* handle error */ }

// ─── 2. Create context ───────────────────────────────────────────────────────
AVCodecContext *ctx = avcodec_alloc_context3(codec);

// ─── 3. Set parameters ───────────────────────────────────────────────────────
ctx->bit_rate = 128000;
ctx->sample_rate = 48000;
ctx->ch_layout = (AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO;
ctx->sample_fmt = AV_SAMPLE_FMT_FLTP; // Planar float (required by Opus)
ctx->frame_size = 960; // Opus: 960 samples at 48kHz = 20ms
ctx->compression_level = FF_COMPRESSION_DEFAULT;

// ─── 4. Open codec ───────────────────────────────────────────────────────────
avcodec_open2(ctx, codec, NULL);

// ─── 5. Create frame ─────────────────────────────────────────────────────────
AVFrame *frame = av_frame_alloc();
frame->format = ctx->sample_fmt;
frame->nb_samples = ctx->frame_size;
frame->sample_rate = ctx->sample_rate;
av_channel_layout_copy(&frame->ch_layout, &ctx->ch_layout);
av_frame_get_buffer(frame, 0);

// ─── 6. Encode loop ──────────────────────────────────────────────────────────
AVPacket *pkt = av_packet_alloc();
while (has_input_samples) {
    // Fill frame->data with PCM samples
    encode_frame(ctx, frame, pkt, output_file);
}
encode_frame(ctx, NULL, pkt, output_file); // Flush

// ─── 7. Cleanup ─────────────────────────────────────────────────────────────
av_frame_free(&frame);
av_packet_free(&pkt);
avcodec_free_context(&ctx);
```

### 12.2 Audio Decoding Skeleton
```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>

AVFormatContext *fmt = avformat_alloc_context();
avformat_open_input(&fmt, "input.opus", NULL, NULL);
avformat_find_stream_info(fmt, NULL);

int idx = av_find_best_stream(fmt, AVMEDIA_TYPE_AUDIO, -1, -1, NULL, 0);
AVStream *st = fmt->streams[idx];
const AVCodec *dec = avcodec_find_decoder(st->codecpar->codec_id);
AVCodecContext *ctx = avcodec_alloc_context3(dec);
avcodec_parameters_to_context(ctx, st->codecpar);
avcodec_open2(ctx, dec, NULL);

AVPacket *pkt = av_packet_alloc();
AVFrame *frame = av_frame_alloc();
while (av_read_frame(fmt, pkt) >= 0) {
    if (pkt->stream_index == idx) {
        avcodec_send_packet(ctx, pkt);
        while (avcodec_receive_frame(ctx, frame) == 0) {
            // frame->data contains decoded PCM
            process_samples(frame);
            av_frame_unref(frame);
        }
    }
    av_packet_unref(pkt);
}
avcodec_send_packet(ctx, NULL); // Flush
while (avcodec_receive_frame(ctx, frame) == 0) {
    process_samples(frame);
    av_frame_unref(frame);
}

av_frame_free(&frame);
av_packet_free(&pkt);
avcodec_free_context(&ctx);
avformat_close_input(&fmt);
```

### 12.3 Resampling with libswresample
```c
SwrContext *swr = swr_alloc();
av_opt_set_int(swr, "in_sample_rate", 44100, 0);
av_opt_set_sample_fmt(swr, "in_sample_fmt", AV_SAMPLE_FMT_S16, 0);
av_opt_set_chlayout(swr, "in_chlayout", &(AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO, 0);

av_opt_set_int(swr, "out_sample_rate", 48000, 0);
av_opt_set_sample_fmt(swr, "out_sample_fmt", AV_SAMPLE_FMT_FLTP, 0);
av_opt_set_chlayout(swr, "out_chlayout", &(AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO, 0);

swr_init(swr);

// Convert
uint8_t *out_planes[2];
int out_linesize;
av_samples_alloc(out_planes, &out_linesize, 2, 960, AV_SAMPLE_FMT_FLTP, 0);
swr_convert(swr, out_planes, 960, (const uint8_t **)in_planes, 441);

swr_free(&swr);
```

---

## 13. PRACTICAL WORKFLOWS

### 13.1 CD to Archive (Lossless)
```bash
# Rip + encode to FLAC with metadata preservation
ffmpeg -i input.wav \
  -c:a flac -compression_level 8 \
  -metadata title="Album Title" \
  -metadata artist="Artist Name" \
  -metadata album="Album Name" \
  -metadata date="2024" \
  -metadata track="1/12" \
  -metadata genre="Jazz" \
  output.flac
```

### 13.2 High-Quality Streaming Transcode
```bash
# Transcode FLAC to AAC for streaming
ffmpeg -i input.flac \
  -c:a libfdk_aac -vbr 4 \
  -movflags +faststart \
  -metadata title="Track Title" \
  output.m4a

# Transcode to Opus for web
ffmpeg -i input.flac \
  -c:a libopus \
  -vbr on \
  -b:a 128k \
  -frame_duration 20 \
  -application audio \
  output.opus
```

### 13.3 Batch Convert to Multiple Formats
```bash
#!/bin/bash
INPUT="$1"
BASENAME="${INPUT%.*}"

ffmpeg -i "$INPUT" -c:a flac -compression_level 8 "${BASENAME}.flac"
ffmpeg -i "$INPUT" -c:a libmp3lame -q:a 2 "${BASENAME}.mp3"
ffmpeg -i "$INPUT" -c:a libopus -b:a 128k "${BASENAME}.opus"
ffmpeg -i "$INPUT" -c:a alac "${BASENAME}.m4a"
ffmpeg -i "$INPUT" -c:a libvorbis -q:a 6 "${BASENAME}.ogg"
```

### 13.4 Gapless-Aware Re-encode
```bash
# Preserve gapless info through transcoding
ffmpeg -i input.mp3 -c:a flac -compression_level 8 \
  -movflags +gaplessinfo \
  output.flac

# Strip gapless info (for editing)
ffmpeg -i input.mp3 -c:a flac -compression_level 8 \
  -map_metadata -1 \
  output_no_gapless.flac
```

---

## 14. ERROR HANDLING & TROUBLESHOOTING

### 14.1 Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `Encoder (codec) not found` | Codec not compiled in | Check `ffmpeg -encoders`, recompile with `--enable-libXXX` |
| `No supported sample format` | Encoder requires specific format | Use `-af "aformat=sample_fmts=fltp"` before encode |
| `Channel layout not supported` | Encoder doesn't support layout | Add `-af "aformat=channel_layouts=stereo"` |
| `Unsupported sample rate` | Sample rate not in encoder's supported list | Resample first: `-ar 48000` |
| `bitrate not allowed` | Bitrate out of valid range | Check valid ranges, use `-q:a` instead |
| `Error while decoding` | Corrupt input or unsupported codec | Verify input: `ffprobe input`, use different decoder |
| `Encoder延时 not supported` | libfdk_aac not available | Use native `aac` encoder instead |

### 14.2 Debugging Commands
```bash
# Verbose logging
ffmpeg -v debug -i input.wav output.mp3 2>&1 | head -50

# Check packet info
ffprobe -v debug -show_packets input.mp3 | grep -i "pts\|dts\|flags"

# Check stream timebase
ffprobe -v error -show_entries stream=time_base \
  -of default=noprint_wrappers=1 input.m4a

# Verify codec capabilities
ffmpeg -h encoder=libopus | head -30
```

---

## 15. REFERENCE COMMANDS BY USE CASE

### 15.1 Use Case: Universal Archive Format
```bash
ffmpeg -i input.wav -c:a flac -compression_level 8 \
  -movflags +gaplessinfo \
  -fletcher16 1 \
  archive.flac
```

### 15.2 Use Case: Portable Player (MP3)
```bash
ffmpeg -i input.wav -c:a libmp3lame -q:a 2 \
  -id3v2_version 4 \
  portable.mp3
```

### 15.3 Use Case: Streaming Server (Opus)
```bash
ffmpeg -i input.wav -c:a libopus \
  -vbr on \
  -b:a 128k \
  -frame_duration 20 \
  -application audio \
  -movflags +faststart \
  stream.opus
```

### 15.4 Use Case: Apple Ecosystem (AAC/ALAC)
```bash
# ALAC lossless
ffmpeg -i input.wav -c:a alac output.m4a

# AAC lossy
ffmpeg -i input.wav -c:a libfdk_aac -vbr 4 \
  -movflags +faststart \
  output.m4a
```

### 15.5 Use Case: Space-Constrained (OGG Vorbis)
```bash
ffmpeg -i input.wav -c:a libvorbis -q:a 4 output.ogg
```

### 15.6 Use Case: Maximum Quality (No Compression)
```bash
ffmpeg -i input.wav -c:a pcm_s32le -ar 96000 output.wav
```

### 15.7 Use Case: Multi-Channel Downmix
```bash
ffmpeg -i input_51.flac \
  -af "aformat=sample_fmts=fltp:channel_layouts=stereo" \
  -c:a libopus -b:a 192k \
  stereo.opus
```

### 15.8 Use Case: Extract Audio from Video
```bash
ffmpeg -i input.mkv -map 0:a -c:a flac -compression_level 8 audio.flac
```

### 15.9 Use Case: Trim Audio (No Re-encode)
```bash
# Copy 30 seconds starting at 1:00
ffmpeg -i input.flac -ss 60 -t 30 -c:a copy output.flac

# Copy with re-encode (for format that doesn't support precise copy)
ffmpeg -i input.flac -ss 60 -t 30 -c:a flac output.flac
```

### 15.10 Use Case: Normalize Audio (ReplayGain Scan)
```bash
# Analyze peak and gain
ffmpeg -i input.wav -af volumedetect -f null /dev/null

# Apply gain (example: -6 dB)
ffmpeg -i input.wav -af "volume=-6dB" -c:a flac output.flac
```

---

## 16. IMPLEMENTATION CHECKLIST

### Build & Environment
- [ ] Identify which audio encoders are available: `ffmpeg -encoders | grep "^ A"`
- [ ] Check for libfdk_aac availability (best AAC encoder): `ffmpeg -encoders | grep fdk`
- [ ] Identify libswresample support for format conversion
- [ ] Test threading with `-threads` flag for performance tuning

### Encoding Pipeline
- [ ] Validate input sample format matches encoder requirements
- [ ] Insert `aformat` filter to convert to encoder-preferred format
- [ ] Handle VBR vs CBR mode selection per codec
- [ ] Implement progress reporting via `-progress pipe:1`
- [ ] Set correct channel layouts for multi-channel audio
- [ ] Validate sample rate is in encoder's supported list

### Metadata
- [ ] Copy metadata through conversion: `-map_metadata 0`
- [ ] Handle cover art with `-map 0:a -map 1:v` and `-disposition:v attached_pic`
- [ ] Strip metadata if needed: `-map_metadata -1`
- [ ] Write container-specific metadata (e.g., `-movflags +faststart` for MP4)

### Quality & Verification
- [ ] Generate framemd5 for bit-exact verification
- [ ] Compare checksums before/after lossless conversion
- [ ] Measure actual bitrate with ffprobe for lossy encodes
- [ ] Test multiple sample rates and formats

### Edge Cases
- [ ] Handle corrupt input: check with `ffprobe` before encoding
- [ ] Handle very short files (<1 second): add padding if needed
- [ ] Handle very long files: set appropriate buffer sizes
- [ ] Handle mismatched channel counts: downmix or upmix
- [ ] Handle unsupported sample rates: resample with aresample

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete FFmpeg audio CLI reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
