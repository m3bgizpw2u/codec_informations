# FFmpeg Build Configuration for Audio Codecs — Deep Technical Reference
> **Category:** FFmpeg API
> **File Extensions:** N/A (build configuration topic)
> **MIME Types:** N/A
> **Standardization Body:** FFmpeg Project
> **Primary Specification:** [FFmpeg Configure Script](https://github.com/FFmpeg/FFmpeg/blob/master/configure), [FFmpeg Compilation Guide](https://trac.ffmpeg.org/wiki/CompilationGuide)
> **Patent Status:** Varies by codec — see license section
> **License:** FFmpeg core: LGPL 2.1+; GPL 3+ required for --enable-gpl; proprietary codecs require --enable-nonfree
> **Current Version:** FFmpeg 7.x (as of 2024-2025)
> **Active Development:** Yes — active development on master branch

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Fabrice Bellard and the FFmpeg developer community
- **Year Created:** 2000
- **Original Purpose:** FFmpeg's `./configure` script was created to solve the problem of building a monolithic multimedia framework supporting hundreds of codecs, formats, and filters with diverse external library dependencies. The build system needed to detect host platform, optional dependencies, and license constraints automatically.
- **Problem with Predecessors:** Pre-configure build systems required manual editing of Makefiles, making it difficult to enable/disable codecs or link against optional libraries without deep knowledge of FFmpeg's internal architecture.

### 1.2 Version History
|| Version | Year | Key Changes |
|---------|------|-------------|
| Initial | 2000 | Basic configure script with enable/disable flags |
| 2003-2005 | 0.5–0.9 | External library support added (libavcodec separation) |
| 2009–2012 | 0.6–1.2 | pkg-config integration, hwaccel framework |
| 2015 | 2.8+ | Unified format/hwaccel API, cuvid/nvenc support |
| 2020 | 4.x | Meson build system added as alternative |
| 2023 | 6.x | Advanced bitstream reader, improved hwaccel |
| 2024 | 7.x | Enhanced ARM support, AV1 improvements |

### 1.3 Current Adoption
- **Primary use cases today:** Audio/video transcoding, streaming server builds, embedded multimedia, mobile app packaging, broadcast automation
- **Platforms with native support:** Linux, macOS, Windows, Android, iOS, BSD, embedded Linux
- **Major services using FFmpeg:** YouTube, Netflix, Twitch, Spotify (internal processing), FFmpeg-based CD ripper conversions
- **Hardware support:** NVIDIA (NVENC/NVDEC), Intel (QSV/VAAPI), AMD (AMF/VAAPI), Apple (VideoToolbox/AudioToolbox), Android (MediaCodec)
- **Status:** Dominant open-source multimedia framework; de facto standard for CLI audio/video conversion

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Build System Architecture
The FFmpeg build system has two main components:

1. **The configure script** (`./configure`): A shell script that:
   - Detects the host platform (Linux, macOS, Windows, etc.)
   - Detects available compilers (gcc, clang, cl, etc.)
   - Detects optional external libraries via `pkg-config` or manual probe
   - Checks for required header files and functions
   - Generates `config.mak` and `config.h` from template files
   - Determines which codecs, muxers, demuxers, filters, and protocols to build

2. **Makefiles**: Generated from templates, invoke the C compiler and linker

3. **Meson build system** (optional, introduced 2020): Alternative to traditional Makefiles

### 2.2 Configure Flow Diagram
```
./configure [options]
     │
     ▼
[Parse command-line flags]
     │
     ▼
[Detect host platform & compiler]
     │
     ▼
[Check required tools (yasm/nasm, pkg-config)]
     │
     ▼
[Detect external libraries via pkg-config]
     │
     ▼
[Check each library's headers and functions]
     │
     ▼
[Check external library compatibility flags]
     │
     ▼
[Generate config.mak and config.h]
     │
     ▼
[Print configuration summary]
```

### 2.3 Key Design Parameters
|| Parameter | Value | Notes |
|-----------|-------|-------|
| Number of configure flags | ~500+ | Covers codecs, formats, hwaccel, tools |
| External library slots | ~80+ | Each requires --enable-lib{Name} |
| License modes | 3 | LGPL-only, GPL, GPL+nonfree |
| Build types | 2 | Static (.a) or shared (.so/.dll) |
| Supported platforms | 10+ | Linux, macOS, Windows, BSD, etc. |

---

## 3. AUDIO CODEC CONFIGURE FLAGS — COMPLETE REFERENCE

### 3.1 Native FFmpeg Audio Codecs (Enabled by Default)
These codecs require no external library — they are built into FFmpeg automatically:

```
Native Audio Encoders (always available):
  - aac          : AAC-LC/HE/HEv2 (native, lower quality than libfdk-aac)
  - ac3          : Dolby Digital (AC-3)
  - eac3         : Dolby Digital Plus (E-AC-3)
  - mp3          : MP3 via built-in encoder (limited quality)
  - alac         : Apple Lossless
  - flac         : Free Lossless Audio Codec
  - vorbis       : Vorbis (native encoder, lower quality than libvorbis)
  - ape          : Monkey's Audio
  - nellymoser   : Nellymoser ASAO
  - comfortnoise  : Comfort noise generation
  - s302m        : SMPTE 302M
  - adpcm_*      : Various ADPCM variants
  - g722         : G.722 wideband codec
  - g726         : G.726 ADPCM
  - g729         : G.729 CELP

Native Audio Decoders (always available):
  - All of the above plus: DTS, TrueHD, MLP, WMA (standard/pro/lossless)
```

### 3.2 External Library Audio Codec Flags
|| Configure Flag | Library Name | Codec(s) | License | Quality |
||----------------|-------------|----------|---------|---------|
| `--enable-libmp3lame` | LAME 3.100+ | MP3 | LGPL | Reference quality |
| `--enable-libvorbis` | libvorbis 1.3+ | Vorbis | BSD | Reference quality |
| `--enable-libopus` | libopus 1.3+ | Opus | BSD | Reference quality |
| `--enable-libfdk-aac` | Fraunhofer FDK AAC | AAC-LC/HE/HEv2 | Non-free | Highest quality |
| `--enable-libspeex` | libspeex 1.2+ | Speex | BSD | Voice optimized |
| `--enable-libshine` | libshine 3.1+ | MP3 (fixed-point) | GPL | Lower quality |
| `--enable-libtwolame` | TwoLAME 0.4+ | MP2 | LGPL | MPEG-1 Layer II |
| `--enable-libwavpack` | WavPack 5+ | WavPack | BSD | Lossless/hybrid |
| `--enable-libopencore-amrnb` | opencore-amrnb | AMR-NB | Apache 2.0 | Voice |
| `--enable-libopencore-amrwb` | opencore-amrwb | AMR-WB | Apache 2.0 | Wideband voice |
| `--enable-libgsm` | libgsm | GSM 06.10 | Proprietary | Voice |
| `--enable-libwebp` | libwebp | WebP (image frames) | BSD | Image codec |
| `--enable-libx264` | x264 | H.264/AVC video | GPL | — |
| `--enable-libx265` | x265 | HEVC/H.265 video | GPL | — |
| `--enable-libvpx` | libvpx | VP8/VP9 video | BSD | — |
| `--enable-libsvtav1` | SVT-AV1 | AV1 video | BSD | — |
| `--enable-libaom` | libaom | AV1 decoder | BSD | — |

### 3.3 Essential Audio-Only Build Flags
For a complete audio converter build, enable at minimum:

```bash
# Core audio codecs (native — always available)
# aac, ac3, eac3, alac, flac, vorbis, ape, nellymoser, g722, g726, g729

# External library audio codecs (requires installing libraries first):
./configure \
  --enable-libmp3lame \
  --enable-libvorbis \
  --enable-libopus \
  --enable-libfdk-aac \
  --enable-libwavpack \
  --enable-libspeex \
  --enable-libtwolame \
  --enable-libopencore-amrnb \
  --enable-libopencore-amrwb \
  --enable-libshine \
  # Add license flags below:
  --enable-gpl \
  --enable-nonfree
```

### 3.4 License Flags
```
--enable-gpl
  Required for: libx264, libx265, libshine, libtwolame, GPL filters
  License impact: Resulting binary is GPL, not LGPL
  Use when: Building GPL-licensed distributions or applications

--enable-version3
  Upgrades LGPL from v2.1 to v2.1+ (avoids some LGPLv3 compatibility issues)
  Required for: libopencore-amrnb, libopencore-amrwb
  Recommended: Always use with --enable-gpl

--enable-nonfree
  Required for: libfdk-aac (incompatible with GPL)
  Warning: Resulting binary cannot be redistributed in open-source projects
  Use when: Building proprietary or commercial applications
```

### 3.5 License Compatibility Matrix
|| Configuration | Redistributable | Open Source Compatible | FFmpeg License |
||--------------|-----------------|----------------------|----------------|
| Minimal (native only) | Yes | Yes | LGPL 2.1 |
| + libmp3lame | Yes | Yes | LGPL 2.1 |
| + libvorbis | Yes | Yes | LGPL 2.1 |
| + libopus | Yes | Yes | LGPL 2.1 |
| + libfdk-aac | No | No | Non-free |
| + libx264 | No | No | GPL |
| + libshine | No | No | GPL |
| + libtwolame | Yes | Yes | LGPL |
| + libwavpack | Yes | Yes | BSD |

---

## 4. BUILD SYSTEM DEEP DETAIL

### 4.1 pkg-config Integration
FFmpeg uses `pkg-config` to find library flags for external dependencies:

```bash
# pkg-config discovers library flags automatically:
pkg-config --cflags --libs libmp3lame
# Output: -I/usr/include/lame -L/usr/lib -lmp3lame

# For static builds, use --pkg-config-flags="--static":
./configure \
  --pkg-config-flags="--static" \
  --enable-libmp3lame
```

**Prepending custom pkg-config paths:**
```bash
export PKG_CONFIG_PATH="$HOME/ffmpeg_build/lib/pkgconfig:$PKG_CONFIG_PATH"
./configure --prefix="$HOME/ffmpeg_build" --enable-libmp3lame
```

**Troubleshooting pkg-config:**
```bash
# Check if a library is detected:
pkg-config --exists libmp3lame && echo "Found" || echo "Not found"

# List all available libraries:
pkg-config --list-all | grep -E "(mp3|vorbis|opus|aac|wavpack)"

# Check library version:
pkg-config --modversion libmp3lame
```

### 4.2 Static vs Shared Builds
```
--enable-static
  Builds: libavcodec.a, libavformat.a, libavutil.a, libswresample.a, etc.
  Output: Static libraries (.a files) in $PREFIX/lib/
  Use case: Embedding FFmpeg in applications, minimal distribution
  Linker: -lavcodec -lavformat -lavutil -lswresample (in any order)

--enable-shared
  Builds: libavcodec.so, libavformat.so, libavutil.so, etc.
  Output: Shared libraries (.so on Linux, .dylib on macOS, .dll on Windows)
  Use case: System-wide installation, plugin systems
  Note: Requires RPATH configuration for portable distributions

--disable-static --disable-shared
  Builds: No libraries, only the ffmpeg/ffprobe/ffplay executables
  Use case: Minimal build with just the CLI tools
```

**Typical distribution build (static, LGPL-compatible):**
```bash
./configure \
  --prefix=/usr/local \
  --pkg-config-flags="--static" \
  --enable-static \
  --disable-shared \
  --enable-libmp3lame \
  --enable-libvorbis \
  --enable-libopus \
  --enable-libwavpack \
  --disable-programs \
  --disable-doc
```

### 4.3 Compilation Process
```bash
# 1. Run configure
./configure [flags]

# 2. Compile (parallel jobs)
make -j$(nproc)

# 3. Install to prefix
make install

# 4. For distribution packaging (DESTDIR):
make install DESTDIR=$PKG_CONFIG
# Results in: $PKG_CONFIG/usr/local/lib/libavcodec.a
```

**Useful make targets:**
```bash
make            # Build everything
make -j8       # Parallel build (8 jobs)
make ffmpeg    # Build only ffmpeg binary
make tools/qt-faststart  # Build qt-faststart tool
make install   # Install to --prefix
make clean     # Remove build artifacts
make distclean # Remove everything including config
make check     # Run regression tests
```

### 4.4 Hardware Acceleration Flags
```
Video Hardware Acceleration (for transcoding pipelines):
  --enable-cuda-nvcc     Enable NVIDIA CUDA compiler (nvcc)
  --enable-nvenc         NVIDIA NVENC encoder (H.264/H.265)
  --enable-nvdec         NVIDIA NVDEC decoder
  --enable-amf           AMD AMF encoder (H.264/H.265)
  --enable-videotoolbox  Apple VideoToolbox (macOS/iOS)
  --enable-mediacodec    Android MediaCodec
  --enable-vaapi         VA-API (Linux)
  --enable-vdpau        VDPAU (Linux older NVIDIA/AMD)
  --enable-rkmpp         Rockchip RGA (embedded)

Audio Hardware Acceleration (limited):
  --enable-audiotoolbox  Apple AudioToolbox (macOS/iOS)
  --enable-openssl / --enable-gnutls / --enable-mbedtls  TLS for streaming
```

**Cross-compilation examples:**
```bash
# Raspberry Pi (ARMhf):
./configure \
  --arch=armhf \
  --target-os=linux \
  --enable-armv6t2 \
  --enable-armvfp \
  --disable-avdevice \
  --enable-libmp3lame

# Android (NDK):
./configure \
  --arch=aarch64 \
  --target-os=android \
  --sysroot=$ANDROID_NDK_ROOT/sysroot \
  --enable-libfdk-aac

# macOS universal binary:
./configure \
  --arch="arm64 x86_64" \
  --enable-audiotoolbox \
  --enable-libmp3lame
```

### 4.5 Audio-Only Build Optimization
Reduce build size and compile time by disabling unnecessary components:

```bash
./configure \
  --disable-doc \
  --disable-htmlpages \
  --disable-manpages \
  --disable-txtpages \
  --disable-programs \
  --disable-avdevice \
  --disable-avfilter \
  --disable-postproc \
  --disable-network \
  --disable-dct \
  --disable-celt \
  --disable-dwt \
  --disable-error-resilience \
  --disable-faandct \
  --disable-faan \
  --disable-hwaccels \
  --disable-muxers \
  --disable-demuxers \
  --enable-muxer=wav,flac,ogg,mp3,m4a,matroska,mp4 \
  --enable-demuxer=wav,flac,ogg,mp3,m4a,matroska,mp4,avi \
  --enable-encoder=aac,mp3,libmp3lame,libopus,libvorbis,libfdk-aac,flac,alac,pcm_* \
  --enable-decoder=aac,mp3,libmp3lame,libopus,libvorbis,libfdk-aac,flac,alac,pcm_*,ac3,eac3 \
  --enable-libmp3lame \
  --enable-libopus \
  --enable-libvorbis \
  --enable-libfdk-aac \
  --enable-gpl --enable-nonfree \
  --enable-static \
  --disable-shared
```

---

## 5. EXTERNAL LIBRARY DEPENDENCY DETAILS

### 5.1 Installing Dependencies Before Configure

#### Ubuntu/Debian (apt):
```bash
sudo apt-get update
sudo apt-get install \
  build-essential \
  yasm \
  nasm \
  pkg-config \
  libmp3lame-dev \
  libvorbis-dev \
  libopus-dev \
  libfdk-aac-dev \
  libwavpack-dev \
  libspeex-dev \
  libtwolame-dev \
  libopencore-amrnb-dev \
  libopencore-amrwb-dev \
  libass-dev \
  libfreetype-dev \
  libx264-dev \
  libx265-dev \
  libvpx-dev \
  libwebp-dev \
  libaom-dev \
  libdav1d-dev \
  libsvtav1-dev \
  libssl-dev \
  libgnutls28-dev
```

#### macOS (Homebrew):
```bash
brew install \
  nasm \
  pkg-config \
  lame \
  libvorbis \
  opus \
  fdk-aac \
  wavpack \
  speex \
  libass \
  freetype \
  x264 \
  x265 \
  libvpx \
  webp \
  aom \
  dav1d \
  libarchive
```

#### Fedora/RHEL/CentOS (dnf):
```bash
sudo dnf install \
  gcc \
  make \
  nasm \
  yasm \
  pkg-config \
  lame-devel \
  libvorbis-devel \
  opus-devel \
  fdk-aac-devel \
  wavpack-devel \
  speex-devel \
  libass-devel \
  freetype-devel \
  x264-devel \
  x265-devel \
  libvpx-devel \
  openssl-devel
```

### 5.2 libfdk-aac Specific Configuration
Fraunhofer FDK AAC is the highest-quality AAC encoder available for FFmpeg:

```bash
# libfdk-aac is incompatible with GPL
# Must use --enable-gpl --enable-nonfree together:
./configure \
  --enable-gpl \
  --enable-nonfree \
  --enable-libfdk-aac

# VBR (variable bitrate) support is available:
ffmpeg -i input.wav -c:a libfdk-aac -vbr 5 output.m4a
# VBR quality: 0 (worst) to 5 (best)

# HE-AAC v2 (aacHeV2) requires -profile:a aac_he_v2:
ffmpeg -i input.wav -c:a libfdk-aac -profile:a aac_he_v2 -b:a 64k output.m4a

# Note: FFmpeg's native aac encoder does NOT support HE-AAC well
# For HE-AAC, use libfdk-aac or libopus instead
```

### 5.3 libmp3lame Configuration
```bash
# LAME provides highest quality MP3 encoding:
./configure --enable-libmp3lame

# Encoding options:
ffmpeg -i input.wav -c:a libmp3lame -q:a 2 output.mp3
# -q:a: VBR quality (0=best/slowest, 9=worst/fastest)
# -b:a: CBR bitrate in kbps (96, 128, 192, 256, 320)

# ABR (average bitrate):
ffmpeg -i input.wav -c:a libmp3lame -b:a 192k output.mp3

# --enable-libshine provides alternative fixed-point MP3 (lower quality, faster):
# Use libshine for embedded/mobile platforms where LAME is unavailable
```

### 5.4 libopus Configuration
```bash
# Opus supports CBR, VBR, and constrained VBR:
./configure --enable-libopus

# Encoding modes:
ffmpeg -i input.wav -c:a libopus -b:a 128k output.opus
# Default: VBR mode

# Constrained VBR (guarantees bitrate ceiling):
ffmpeg -i input.wav -c:a libopus -vbr on -b:a 96k output.opus

# Application type hints:
#   audio: general music (default)
#   voip: voice optimization
#   lowdelay: lowest latency
ffmpeg -i input.wav -c:a libopus -application audio -b:a 128k output.opus
ffmpeg -i input.wav -c:a libopus -application voip -b:a 32k output.opus

# Bitrate range: 6 kbps (narrowband) to 510 kbps (lossless-ish)
# Sample rate: Always 48 kHz internally (resamples automatically)
# Frame size: 2.5, 5, 10, 20, 40, 60 ms
```

### 5.5 libvorbis Configuration
```bash
# Vorbis is the free/libre alternative to MP3:
./configure --enable-libvorbis

# Quality-based encoding (VBR):
ffmpeg -i input.wav -c:a libvorbis -q:a 6 output.ogg
# -q:a: VBR quality -0.1 to 10.0 (6 is transparency threshold)

# Bitrate-based encoding (ABR):
ffmpeg -i input.wav -c:a libvorbis -b:a 192k output.ogg

# Note: Vorbis in OGG container uses Vorbis Comments for metadata
# Metadata is embedded in the OGG bitstream, not a separate tag block
```

---

## 6. VERIFYING THE BUILD

### 6.1 Post-Configure Verification
```bash
# Check enabled codecs:
ffmpeg -codecs 2>/dev/null | grep -E "DE.*EA.*A.*(mp3|vorbis|opus|aac|wavpack|flac)"

# Sample output showing enabled encoders:
# DEA..S aac                  AAC (Advanced Audio Coding)
# DEA..S ac3                  ATSC A/52A (AC-3)
# D.A..S alac                 ALAC (Apple Lossless Audio Codec)
# DEA..S eac3                 ATSC A/52B (E-AC-3)
# D.A..S flac                 FLAC (Free Lossless Audio Codec)
# DEA..S libfdk_aac           AAC (Free Development Kit)
# DEA..S libmp3lame           MP3 (LAME)
# DEA..S libopus             Opus (libopus)
# DEA..S libvorbis           Vorbis (libvorbis)
# DEA..S libwavpack          WavPack (encoders)
```

### 6.2 Verifying Decoder Availability
```bash
# List all audio decoders:
ffmpeg -decoders 2>/dev/null | grep -E "^ A.*"

# Verify specific codecs:
ffmpeg -codecs 2>/dev/null | grep -i "wma"
# Should show: D.A... wmalossless, D.A... wmapro, D.A... wmav1, D.A... wmav2
```

### 6.3 Format and Protocol Verification
```bash
# List supported formats:
ffmpeg -formats 2>/dev/null | grep -E "^ D.*A.*WMA|^ D.*A.*ASF|^ D.*A.*OGG"

# Sample output:
# DE wma_asf           ASF (Advanced Systems Format)
# D... wav             WAV / WAVE (Waveform Audio)
# DE ogg               OGG
# DE matroska,webm     Matroska / WebM
```

### 6.4 Test Encoding
```bash
# Test each codec after build:
ffmpeg -f lavfi -i "sine=frequency=440:duration=5" -c:a pcm_s16le /tmp/test.wav

# Test each enabled encoder:
ffmpeg -i /tmp/test.wav -c:a libmp3lame -b:a 192k /tmp/test.mp3
ffmpeg -i /tmp/test.wav -c:a libvorbis -q:a 6 /tmp/test.ogg
ffmpeg -i /tmp/test.wav -c:a libopus -b:a 128k /tmp/test.opus
ffmpeg -i /tmp/test.wav -c:a libfdk-aac -b:a 192k /tmp/test.m4a
ffmpeg -i /tmp/test.wav -c:a flac -compression_level 8 /tmp/test.flac
ffmpeg -i /tmp/test.wav -c:a alac /tmp/test.m4a

# Verify each output:
ffprobe /tmp/test.mp3  # Check duration, bitrate, codec
ffprobe /tmp/test.ogg
ffprobe /tmp/test.opus
ffprobe /tmp/test.m4a
ffprobe /tmp/test.flac

# Cleanup
rm -f /tmp/test.wav /tmp/test.mp3 /tmp/test.ogg /tmp/test.opus /tmp/test.m4a /tmp/test.flac
```

---

## 7. MESON BUILD SYSTEM

### 7.1 Meson Overview
Since FFmpeg 4.4, Meson is an alternative build system alongside the traditional Makefile approach:

```bash
# Install Meson and Ninja (required):
pip3 install meson ninja

# Configure with Meson:
meson setup build --prefix=/usr/local \
  -Dlibmp3lame=enabled \
  -Dlibvorbis=enabled \
  -Dlibopus=enabled \
  -Dlibfdk-aac=enabled \
  -Dgpl=enabled \
  -Dnonfree=enabled

# Build:
meson compile -C build

# Install:
meson install -C build
```

### 7.2 Meson Advantages
- Faster incremental builds
- Better cross-platform support
- Native Ninja performance
- Easier integration with IDEs
- Better dependency resolution via pkg-config

### 7.3 Meson Limitations
- Some FFmpeg options may not yet be exposed as Meson options
- External library detection may differ from configure script
- Less documented than traditional approach

---

## 8. DISTRIBUTION & BUNDLING

### 8.1 Static Build vs Shared Libraries
| Aspect | Static (.a) | Shared (.so/.dylib) |
|--------|-------------|---------------------|
| Binary size | Larger | Smaller |
| Deployment | Single executable | Requires .so files |
| Updates | Rebuild app | Update shared libs |
| LGPL compliance | Requires object file linking | Dynamic linking is fine |
| Startup time | Faster (no dlopen) | Slower |

### 8.2 Complete Audio-Only Static Build Script
```bash
#!/bin/bash
# build_audio_ffmpeg.sh - Build static FFmpeg for audio conversion

set -e

PREFIX="$HOME/ffmpeg_audio_build"
BUILD_DIR="$HOME/ffmpeg_build_tmp"

mkdir -p "$BUILD_DIR"
cd "$BUILD_DIR"

# Download and extract FFmpeg:
if [ ! -d "FFmpeg-n7.1" ]; then
  wget -O ffmpeg.tar.xz https://ffmpeg.org/releases/ffmpeg-7.1.tar.xz
  tar xf ffmpeg.tar.xz
fi

cd FFmpeg-n7.1

# Configure for static audio build:
./configure \
  --prefix="$PREFIX" \
  --pkg-config-flags="--static" \
  --enable-static \
  --disable-shared \
  --disable-doc \
  --disable-htmlpages \
  --disable-manpages \
  --disable-txtpages \
  --disable-programs \
  --disable-avdevice \
  --disable-avfilter \
  --disable-postproc \
  --disable-network \
  --disable-dct \
  --disable-dwt \
  --disable-error-resilience \
  --disable-faan \
  --disable-hwaccels \
  --disable-muxers \
  --disable-demuxers \
  --enable-muxer=wav,flac,ogg,mp3,m4a,matroska,mp4,aiff \
  --enable-demuxer=wav,flac,ogg,mp3,m4a,matroska,mp4,aiff \
  --enable-encoder=aac,mp3,libmp3lame,libopus,libvorbis,libfdk-aac,flac,alac,pcm_*,ac3,eac3,ape,wavpack \
  --enable-decoder=aac,mp3,libmp3lame,libopus,libvorbis,libfdk-aac,flac,alac,pcm_*,ac3,eac3,ape,wavpack,wmav1,wmav2,wmalossless,wmapro \
  --enable-libmp3lame \
  --enable-libvorbis \
  --enable-libopus \
  --enable-libfdk-aac \
  --enable-libwavpack \
  --enable-libspeex \
  --enable-libtwolame \
  --enable-gpl \
  --enable-version3 \
  --enable-nonfree

# Compile and install:
make -j$(nproc)
make install

echo "Build complete. FFmpeg installed to: $PREFIX"
echo "Binary location: $PREFIX/bin/ffmpeg"
```

### 8.3 Application Bundling Best Practices
```bash
# For distributing with an application:
APP_DIR="/path/to/my-audio-app"
FFMPEG_DIR="$APP_DIR/ffmpeg"

# Create app bundle structure:
mkdir -p "$APP_DIR"
mkdir -p "$FFMPEG_DIR"

# Copy static ffmpeg binary:
cp "$PREFIX/bin/ffmpeg" "$FFMPEG_DIR/"

# Copy required libraries (if shared build):
# ldd ffmpeg  # Shows library dependencies
# cp lib*.so "$FFMPEG_DIR/"  # For Linux

# Set RPATH for portable deployment:
patchelf --set-rpath '$ORIGIN/' "$FFMPEG_DIR/ffmpeg"
```

---

## 9. TROUBLESHOOTING BUILD ISSUES

### 9.1 Common Configure Errors
```
Error: "libmp3lame >= 3.98.3 not found"
  Solution: sudo apt install libmp3lame-dev

Error: "libfdk-aac not found"
  Solution: sudo apt install libfdk-aac-dev
  Note: May require adding multiverse/universe repositories on Ubuntu

Error: "X11 not found"
  Solution: sudo apt install libx11-dev
  Or: --disable-xlib if X11 is not needed

Error: "nasm/yasm not found"
  Solution: sudo apt install nasm yasm
  Or: --disable-yasm if assembler is unavailable (slower build)
```

### 9.2 Linking Errors After Configure
```
Error: "libavcodec/libavcodec.a(codec_*.o): undefined reference to 'x264'"
  Cause: --enable-libx264 was used but library not linked
  Solution: Re-run configure with --enable-libx264 AFTER installing libx264-dev

Error: "libavcodec.so: undefined reference to 'fdk_aac'"
  Cause: --enable-libfdk-aac combined with --enable-gpl but linking order wrong
  Solution: Ensure --enable-nonfree is set, check linker order
```

### 9.3 Runtime Library Issues
```
Error: "ffmpeg: error while loading shared libraries: libmp3lame.so.0"
  Cause: Library not in LD_LIBRARY_PATH
  Solution: 
    export LD_LIBRARY_PATH="/usr/local/lib:$LD_LIBRARY_PATH"
    # Or: sudo ldconfig

Error: "libavcodec.so.60: cannot open shared object file"
  Cause: Multiple FFmpeg versions installed, version mismatch
  Solution: Check with: which ffmpeg; ldd $(which ffmpeg)
```

### 9.4 License Compliance Checklist
```
[ ] Identify which external libraries are used
[ ] Determine license of each library
[ ] For LGPL libraries: ensure dynamic linking OR provide object files
[ ] For GPL libraries: comply with GPL (provide source, allow modification)
[ ] For non-free libraries (libfdk-aac): cannot redistribute in open-source projects
[ ] For commercial products: consult licensing attorney for specific guidance
```

---

## 10. FFMPEG CLI REFERENCE FOR AUDIO BUILD

### 10.1 Common Audio Encoding Commands
```bash
# MP3 (LAME, VBR):
ffmpeg -i input.wav -c:a libmp3lame -q:a 2 output.mp3

# MP3 (LAME, CBR 320k):
ffmpeg -i input.wav -c:a libmp3lame -b:a 320k output.mp3

# AAC (FDK-AAC, VBR):
ffmpeg -i input.wav -c:a libfdk-aac -vbr 4 output.m4a

# AAC (FDK-AAC, CBR):
ffmpeg -i input.wav -c:a libfdk-aac -b:a 256k output.m4a

# AAC (FDK-AAC, HE-AAC v2):
ffmpeg -i input.wav -c:a libfdk-aac -profile:a aac_he_v2 -b:a 64k output.m4a

# Opus:
ffmpeg -i input.wav -c:a libopus -b:a 128k output.opus

# Vorbis:
ffmpeg -i input.wav -c:a libvorbis -q:a 6 output.ogg

# FLAC:
ffmpeg -i input.wav -c:a flac -compression_level 8 output.flac

# ALAC:
ffmpeg -i input.wav -c:a alac output.m4a

# WavPack:
ffmpeg -i input.wav -c:a wavpack -compression_level 6 output.wv

# AC-3 (Dolby Digital):
ffmpeg -i input.wav -c:a ac3 -b:a 640k output.ac3

# E-AC-3 (Dolby Digital Plus):
ffmpeg -i input.wav -c:a eac3 -b:a 256k output.eac3
```

### 10.2 Audio Transcoding with Metadata
```bash
# Transcode with metadata preservation:
ffmpeg -i input.flac \
  -c:a libmp3lame -b:a 320k \
  -metadata title="Song Title" \
  -metadata artist="Artist Name" \
  -metadata album="Album Name" \
  -metadata date="2024" \
  -metadata track="1" \
  -metadata genre="Rock" \
  output.mp3

# Transcode with cover art:
ffmpeg -i input.flac -i cover.jpg \
  -c:a libfdk-aac -b:a 256k \
  -c:v copy \
  -map 0:a -map 1:v \
  -metadata:s:v title="Album cover" \
  -metadata:s:v comment="Cover (front)" \
  output.m4a
```

### 10.3 Audio Quality Verification
```bash
# Measure loudness (EBU R128):
ffmpeg -i input.mp3 -af loudnorm=print_format=json -f null -

# Check codec info:
ffprobe -v quiet -print_format json -show_format -show_streams input.mp3

# Verify bitrate:
ffprobe -v quiet -show_entries format=bit_rate -of default=noprint_wrappers=1 input.mp3

# Extract audio info:
ffprobe -v quiet -show_entries stream=codec_name,sample_rate,channels,bits_per_sample -of default=noprint_wrappers=1 input.mp3
```

---

## 11. REFERENCE IMPLEMENTATIONS & LIBRARIES

|| Library | Language | License | Encoder Quality | Decoder Quality | URL |
|---------|----------|---------|-----------------|-----------------|-----|
| FFmpeg native | C | LGPL 2.1+ | 6/10 (varies) | 9/10 | https://ffmpeg.org |
| LAME | C | LGPL | 10/10 | 10/10 | https://lame.sourceforge.io |
| libvorbis | C | BSD | 10/10 | 10/10 | https://xiph.org/vorbis |
| libopus | C | BSD | 10/10 | 10/10 | https://opus-codec.org |
| libfdk-aac | C | Non-free | 10/10 | 10/10 | https://android.googlesource.com/platform/external/fdk-aac |
| libwavpack | C | BSD | 9/10 | 9/10 | https://www.wavpack.com |
| TwoLAME | C | LGPL | 9/10 | 9/10 | https://www.twolame.org |

---

## 12. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specifications
- **FFmpeg Configure Script:** https://github.com/FFmpeg/FFmpeg/blob/master/configure
- **FFmpeg Compilation Guide:** https://trac.ffmpeg.org/wiki/CompilationGuide
- **FFmpeg Encode Guide:** https://trac.ffmpeg.org/wiki/Encode/AAC
- **FFmpeg MP3 Encoding Guide:** https://trac.ffmpeg.org/wiki/Encode/MP3
- **FFmpeg Opus Encoding Guide:** https://trac.ffmpeg.org/wiki/Encode/Opus
- **FFmpeg Vorbis Encoding Guide:** https://trac.ffmpeg.org/wiki/Encode/Vorbis
- **ASF Specification:** https://exse.eyewated.com/fls/54b3ed95bbfb1a92.pdf

### Technical Resources
- FFmpeg Codec Documentation: `ffmpeg -h encoder={name}`
- FFmpeg Muxer Documentation: `ffmpeg -h muxer={name}`
- Hydrogenaudio Forums: https://hydrogenaud.io/
- Multimedia Wiki: https://wiki.multimedia.cx/

---

## 13. IMPLEMENTATION CHECKLIST (Audio Converter Developer)

### Build & Environment
- [ ] Identify FFmpeg `./configure` flags needed for target codecs
- [ ] Install all required external library development packages
- [ ] Run `./configure` and verify codec availability with `ffmpeg -codecs`
- [ ] Test encoding with each enabled codec before deployment
- [ ] Document which build flags were used for reproducibility
- [ ] For distribution: choose static or shared based on LGPL compliance needs
- [ ] For commercial use: verify license compliance for all linked libraries

### Dependency Management
- [ ] Document minimum library versions (e.g., LAME 3.98.3+)
- [ ] Provide installation instructions for each supported platform
- [ ] Consider using conda or vcpkg for cross-platform dependency management
- [ ] For Windows: provide pre-built dependencies or build script
- [ ] For macOS: provide Homebrew tap or build script
- [ ] For Linux: provide distribution-specific packages or PPA

### Quality & Verification
- [ ] Test encoding with reference samples
- [ ] Verify bitrate/quality modes are accessible via CLI
- [ ] Test VBR, CBR, and ABR modes for each codec
- [ ] Verify metadata handling (title, artist, album, cover art)
- [ ] Test multichannel audio handling (5.1, 7.1)
- [ ] Test high-resolution audio (24-bit, 96kHz)

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
