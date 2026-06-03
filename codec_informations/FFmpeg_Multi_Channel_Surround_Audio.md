# FFmpeg Multi-Channel and Surround Audio — Deep Technical Reference

> **Category:** FFmpeg API
> **File Extensions:** N/A (FFmpeg API reference)
> **MIME Types:** N/A
> **Standardization Body:** FFmpeg Project / ITU-R / Dolby / DTS / Xiph.org
> **Primary Specification:** FFmpeg source code (`libavutil/channel_layout.h`, `libavutil/channel_layout.c`)
> **Patent Status:** Channel layouts themselves are not patented; specific encodings (Dolby, DTS) are patented
> **License:** LGPL 2.1+ (FFmpeg core)
> **Current Version:** FFmpeg 7.x (ongoing development)
> **Active Development:** Yes — ongoing channel layout API improvements

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation

- **Creator(s):** FFmpeg Project; channel layouts from ITU-R BS.775, Dolby, DTS, Xiph.org (Vorbis)
- **Year Created:** FFmpeg channel layout system evolved from early 2000s; AVChannelLayout API introduced in FFmpeg 4.x
- **Original Purpose:** Represent multi-channel audio configurations for proper encoding, decoding, mixing, and routing
- **Problem with Predecessors:** Legacy FFmpeg used simple channel count integers which couldn't distinguish between different 6-channel configurations (e.g., 5.1 back vs 5.1 side)

### 1.2 Version History

| Version | Year | Key Changes |
|---------|------|-------------|
| FFmpeg 0.6 | 2010 | Basic channel count support (mono, stereo, 5.1) |
| FFmpeg 2.0 | 2013 | Named channel layouts added to CLI |
| FFmpeg 3.0 | 2016 | Extended layout support, improved channel mapping |
| FFmpeg 4.0 | 2018 | AVChannelLayout struct introduced (replaces uint64_t masks) |
| FFmpeg 5.0 | 2022 | Full AVChannelLayout adoption, custom layouts |
| FFmpeg 6.0 | 2023 | Improved ambisonic support, additional layouts |
| FFmpeg 7.0 | 2024 | Continued API refinements |

### 1.3 Current Adoption

- **Primary use cases today:** Home theater, streaming services, gaming audio, VR/360 audio
- **Platforms with native support:** All platforms with FFmpeg (home theaters, streaming servers, game engines)
- **Major services using multichannel:** Netflix, Disney+, Apple TV+ (Dolby Atmos), Blu-ray discs
- **Hardware support:** A/V receivers, soundbars, gaming consoles, mobile devices
- **Status:** Growing complexity with object-based audio (Dolby Atmos, DTS:X)

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Channel Layout Fundamentals

FFmpeg uses `AVChannelLayout` to describe audio channel configurations. This structure replaced the older uint64_t bitmask system in FFmpeg 4.0+.

```c
struct AVChannelLayout {
    enum AVChannelOrder order;       // Channel ordering method
    int nb_channels;                  // Number of channels
    union {
        uint64_t mask;               // For AV_CHANNEL_ORDER_NATIVE
        AVChannelCustom *map;         // For AV_CHANNEL_ORDER_CUSTOM
        // ...
    } u;
    void *opaque;                    // Private data
};
```

### 2.2 Channel Order Types

| Order Type | Description | Use Case |
|------------|-------------|----------|
| `AV_CHANNEL_ORDER_NATIVE` | Standard layout using channel mask | All standard configurations |
| `AV_CHANNEL_ORDER_CUSTOM` | User-defined channel positions | Non-standard, VR/360 audio |
| `AV_CHANNEL_ORDER_UNSPEC` | Unknown order, only channel count known | Raw PCM, some formats |

### 2.3 Standard Channel Identifiers

FFmpeg defines channels using the `AVChannel` enum:

```c
enum AVChannel {
    AV_CHAN_FRONT_LEFT,           // FL
    AV_CHAN_FRONT_RIGHT,          // FR
    AV_CHAN_FRONT_CENTER,         // FC
    AV_CHAN_LOW_FREQUENCY,        // LFE
    AV_CHAN_BACK_LEFT,            // BL
    AV_CHAN_BACK_RIGHT,           // BR
    AV_CHAN_FRONT_LEFT_OF_CENTER, // FLC
    AV_CHAN_FRONT_RIGHT_OF_CENTER,// FRC
    AV_CHAN_BACK_CENTER,          // BC
    AV_CHAN_SIDE_LEFT,            // SL
    AV_CHAN_SIDE_RIGHT,           // SR
    AV_CHAN_TOP_CENTER,           // TC
    AV_CHAN_TOP_FRONT_LEFT,       // TFL
    AV_CHAN_TOP_FRONT_CENTER,     // TFC
    AV_CHAN_TOP_FRONT_RIGHT,      // TFR
    AV_CHAN_TOP_BACK_LEFT,        // TBL
    AV_CHAN_TOP_BACK_CENTER,      // TBC
    AV_CHAN_TOP_BACK_RIGHT,       // TBR
    AV_CHAN_STEREO_LEFT,          // DL (downmix left)
    AV_CHAN_STEREO_RIGHT,         // DR (downmix right)
    // ... and many more
};
```

### 2.4 Channel Mask Flags

```c
#define AV_CH_FRONT_LEFT           0x00000001ULL
#define AV_CH_FRONT_RIGHT          0x00000002ULL
#define AV_CH_FRONT_CENTER         0x00000004ULL
#define AV_CH_LOW_FREQUENCY        0x00000008ULL
#define AV_CH_BACK_LEFT            0x00000010ULL
#define AV_CH_BACK_RIGHT           0x00000020ULL
#define AV_CH_FRONT_LEFT_OF_CENTER 0x00000040ULL
#define AV_CH_FRONT_RIGHT_OF_CENTER 0x00000080ULL
#define AV_CH_BACK_CENTER          0x00000100ULL
#define AV_CH_SIDE_LEFT            0x00000200ULL
#define AV_CH_SIDE_RIGHT           0x00000400ULL
#define AV_CH_TOP_CENTER           0x00000800ULL
// ... and more
```

---

## 3. STANDARD CHANNEL LAYOUTS

### 3.1 Complete FFmpeg Channel Layout Table

| Layout Name | Channels | Channel Order | AV_CHANNEL_LAYOUT_* |
|-------------|----------|---------------|---------------------|
| mono | 1 | C | `AV_CHANNEL_LAYOUT_MONO` |
| stereo | 2 | FL, FR | `AV_CHANNEL_LAYOUT_STEREO` |
| 2.1 | 3 | FL, FR, LFE | `AV_CHANNEL_LAYOUT_2POINT1` |
| 2.0 | 2 | FL, FR | `AV_CHANNEL_LAYOUT_STEREO` |
| 3.0 | 3 | FL, FR, FC | `AV_CHANNEL_LAYOUT_SURROUND` |
| 3.0(back) | 3 | FL, FR, BC | [custom] |
| 4.0 | 4 | FL, FR, FC, BC | `AV_CHANNEL_LAYOUT_4POINT0` |
| quad | 4 | FL, FR, BL, BR | `AV_CHANNEL_LAYOUT_QUAD` |
| quad(side) | 4 | FL, FR, SL, SR | [custom] |
| 5.0 | 5 | FL, FR, FC, SL, SR | `AV_CHANNEL_LAYOUT_5POINT0` |
| 5.0(back) | 5 | FL, FR, FC, BL, BR | [custom] |
| 5.1 | 6 | FL, FR, FC, LFE, SL, SR | `AV_CHANNEL_LAYOUT_5POINT1` |
| 5.1(back) | 6 | FL, FR, FC, LFE, BL, BR | `AV_CHANNEL_LAYOUT_5POINT1_BACK` |
| 5.1(side) | 6 | FL, FR, FC, LFE, SL, SR | `AV_CHANNEL_LAYOUT_5POINT1` |
| 6.0 | 6 | FL, FR, FC, BC, SL, SR | `AV_CHANNEL_LAYOUT_6POINT0` |
| 6.0(front) | 6 | FL, FR, FLC, FRC, SL, SR | [custom] |
| 6.1 | 7 | FL, FR, FC, LFE, BC, SL, SR | `AV_CHANNEL_LAYOUT_6POINT1` |
| 6.1(back) | 7 | FL, FR, FC, LFE, BC, BL, BR | [custom] |
| 6.1(front) | 7 | FL, FR, FLC, FRC, LFE, SL, SR | [custom] |
| 7.0 | 7 | FL, FR, FC, BL, BR, SL, SR | `AV_CHANNEL_LAYOUT_7POINT0` |
| 7.0(front) | 7 | FL, FR, FC, FLC, FRC, SL, SR | [custom] |
| 7.1 | 8 | FL, FR, FC, LFE, BL, BR, SL, SR | `AV_CHANNEL_LAYOUT_7POINT1` |
| 7.1(wide) | 8 | FL, FR, FC, LFE, BL, BR, FLC, FRC | `AV_CHANNEL_LAYOUT_7POINT1_WIDE` |
| 7.1(wide-back) | 8 | FL, FR, FC, LFE, BL, BR, FLC, FRC | `AV_CHANNEL_LAYOUT_7POINT1_WIDE_BACK` |
| 7.1(back) | 8 | FL, FR, FC, LFE, BL, BR, SL, SR | `AV_CHANNEL_LAYOUT_7POINT1` |
| 7.1(wide-side) | 8 | FL, FR, FC, LFE, FLC, FRC, SL, SR | [custom] |
| octagonal | 8 | FL, FR, FC, BL, BR, BC, SL, SR | `AV_CHANNEL_LAYOUT_OCTAGONAL` |
| hexadecagonal | 16 | 16 channels | `AV_CHANNEL_LAYOUT_HEXADECAGONAL` |
| stereo(2.1) | 3 | FL, FR, LFE | via custom |
| downmix | 2 | DL, DR | via custom |

### 3.2 Channel Order by Layout

**WAV/FFmpeg Standard Order (most common):**

```
Mono:          [C]
Stereo:        [FL, FR]
2.0:           [FL, FR]
2.1:           [FL, FR, LFE]
3.0:           [FL, FR, FC]
4.0:           [FL, FR, FC, BC]
5.0:           [FL, FR, FC, SL, SR]
5.1:           [FL, FR, FC, LFE, SL, SR]
5.1(back):     [FL, FR, FC, LFE, BL, BR]
7.0:           [FL, FR, FC, BL, BR, SL, SR]
7.1:           [FL, FR, FC, LFE, BL, BR, SL, SR]
7.1(wide):     [FL, FR, FC, LFE, BL, BR, FLC, FRC]
```

### 3.3 LFE Channel Handling

The LFE (Low Frequency Effects) channel is always at position index 3 (zero-based) in 5.1+ layouts:

```c
// In FFmpeg channel layout:
// Position 0: FL
// Position 1: FR
// Position 2: FC (Center)
// Position 3: LFE  <-- Always here in 5.1+ layouts
// Position 4: SL/BL (Side or Back Left)
// Position 5: SR/BR (Side or Back Right)
```

---

## 4. FFMPEG CHANNEL LAYOUT API

### 4.1 Querying Standard Layouts

```bash
# List all available channel layouts
ffmpeg -layouts

# Output shows:
# Channel layout: mono
# Channel layout: stereo
# Channel layout: 5.1
# Channel layout: 7.1
# ...
```

### 4.2 Setting Channel Layout in FFmpeg CLI

```bash
# Set output to 5.1 surround
ffmpeg -i input.wav -channel_layout 5.1 output.wav

# Set output to 7.1 with specific side/back variant
ffmpeg -i input.wav -channel_layout "7.1(wide)" output.wav

# Force mono output
ffmpeg -i input.wav -ac 1 output.wav

# Force stereo output
ffmpeg -i input.wav -ac 2 output.wav

# Use short form -ch_layout
ffmpeg -i input.wav -ch_layout 5.1 output.wav
```

### 4.3 Channel Layout String Names

FFmpeg recognizes these channel layout strings:

```bash
# Valid layout strings in FFmpeg
mono           # 1 channel
stereo         # 2 channels
stereo+FL      # 3 channels (stereo + front left)
stereo+LFE     # 3 channels (stereo + LFE)
2.1            # 3 channels
3.0            # 3 channels
3.0(back)      # 3 channels
4.0            # 4 channels
quad           # 4 channels
quad(side)     # 4 channels
5.0            # 5 channels
5.0(back)      # 5 channels
5.1            # 6 channels
5.1(back)      # 6 channels
5.1(side)      # 6 channels
6.0            # 6 channels
6.0(front)     # 6 channels
6.1            # 7 channels
6.1(back)      # 7 channels
6.1(front)     # 7 channels
7.0            # 7 channels
7.0(front)     # 7 channels
7.1            # 8 channels
7.1(wide)      # 8 channels
7.1(wide-back) # 8 channels
7.1(back)      # 8 channels
7.1(wide-side) # 8 channels
octagonal      # 8 channels
```

---

## 5. CHANNEL REMAPPING AND MIXING

### 5.1 The pan Filter

The `pan` filter provides precise control over channel mixing:

```bash
# Syntax: pan=layout|out_def1|out_def2|...
# Where out_def: out_channel=[gain*]in_channel[(+|-)[gain*]in_channel...]

# Extract specific channels from 5.1
ffmpeg -i input_5.1.mkv -af "pan=stereo|c0=FL|c1=FR" output_stereo.wav

# Convert 5.1 to stereo with center channel
ffmpeg -i input_5.1.mkv -af "pan=stereo|FL=FL+0.5*FC|FR=FR+0.5*FC" output.wav

# Full 5.1 to stereo downmix with LFE
ffmpeg -i input_5.1.mkv \
  -af "pan=stereo|FL=FL+0.707*FC+0.5*LFE|FR=FR+0.707*FC+0.5*LFE" \
  output.wav

# Rearrange channels: swap front left and front right
ffmpeg -i input_5.1.mkv -af "pan=5.1|c0=c1|c1=c0|c2=c2|c3=c3|c4=c4|c5=c5" output_5.1.wav
```

### 5.2 Professional Downmix Coefficients

**ITU-R BS.775 Standard Downmix (5.1 → Stereo):**

```bash
# ITU standard downmix coefficients
ffmpeg -i input_5.1.mkv \
  -af "pan=stereo|FL=0.374107*FC+0.529067*FL+0.458186*BL+0.264534*BR+0.374107*LFE|FR=0.374107*FC+0.529067*FR+0.458186*BR+0.264534*BL+0.374107*LFE" \
  output_stereo.wav
```

**Simplified Downmix (commonly used):**

```bash
# Simple stereo downmix without LFE
ffmpeg -i input_5.1.mkv \
  -af "pan=stereo|FL=FL+0.707*FC+0.5*BL|FR=FR+0.707*FC+0.5*BR" \
  output_stereo.wav

# With LFE included (0.5 coefficient)
ffmpeg -i input_5.1.mkv \
  -af "pan=stereo|FL=FL+0.707*FC+0.5*LFE+0.707*BL|FR=FR+0.707*FC+0.5*LFE+0.707*BR" \
  output_stereo.wav
```

### 5.3 Automatic Renormalization

Use `<` to auto-renormalize when mixing:

```bash
# Auto-renormalize to prevent clipping
ffmpeg -i input_5.1.mkv \
  -af "pan=stereo|FL < FL+0.5*FC+0.5*LFE|FR < FR+0.5*FC+0.5*LFE" \
  output.wav
```

### 5.4 Channel Number Reference

| Channel Name | c0 | c1 | c2 | c3 | c4 | c5 | c6 | c7 |
|-------------|-----|-----|-----|-----|-----|-----|-----|-----|
| mono | C | | | | | | | |
| stereo | FL | FR | | | | | | |
| 5.1 | FL | FR | FC | LFE | SL | SR | | |
| 5.1(back) | FL | FR | FC | LFE | BL | BR | | |
| 7.1 | FL | FR | FC | LFE | BL | BR | SL | SR |

### 5.5 Extracting Individual Channels

```bash
# Extract each channel of 5.1 to separate mono files
ffmpeg -i input_5.1.wav \
  -map_channel 0.0.0 FL.wav \      # Front Left
  -map_channel 0.0.1 FR.wav \      # Front Right
  -map_channel 0.0.2 FC.wav \      # Front Center
  -map_channel 0.0.3 LFE.wav \     # LFE
  -map_channel 0.0.4 SL.wav \      # Side/Back Left
  -map_channel 0.0.5 SR.wav        # Side/Back Right

# Alternative: use pan to extract single channel
ffmpeg -i input_5.1.wav -af "pan=1c|c0=FC" center.wav
ffmpeg -i input_5.1.wav -af "pan=1c|c0=LFE" lfe.wav
```

### 5.6 Joining Multiple Mono Files to Multi-Channel

```bash
# Join 6 mono files into 5.1
ffmpeg -i FL.wav -i FR.wav -i FC.wav -i LFE.wav -i BL.wav -i BR.wav \
  -filter_complex "[0:a][1:a][2:a][3:a][4:a][5:a]join=inputs=6:channel_layout=5.1" \
  output_5.1.wav

# Join stereo + center + LFE into 5.1
ffmpeg -i stereo.wav -i center.wav -i lfe.wav -i surround_mono.wav \
  -filter_complex "[0:a][1:a][2:a][3:a]join=inputs=4:channel_layout=5.1" \
  output_5.1.wav
```

---

## 6. UPMIXING AND STEREO TO SURROUND

### 6.1 Stereo to 5.1 Upmix

FFmpeg supports simple upmixing but for quality results, external filters or manual mixing are recommended:

```bash
# Simple stereo to 5.1 using channel duplication
ffmpeg -i stereo.wav \
  -af "pan=5.1|c0=FL|c1=FR|c2=FC|c3=LFE|c4=BL|c5=BR" \
  output_5.1.wav

# Upmix with center extraction
ffmpeg -i stereo.wav \
  -af "pan=5.1|c0=FL|c1=FR|c2=0.5*FL+0.5*FR|c3=LFE|c4=BL|c5=BR" \
  output_5.1.wav
```

### 6.2 Dolby Pro Logic II Encoding

For proper Pro Logic encoding, use specialized tools. FFmpeg can perform basic matrix encoding:

```bash
# Basic Lt/Rt (Left-total/Right-total) encoding
# Lt = L + j*C + k*SL + m*SR
# Rt = R + j*C + k*SL + m*SR
# Note: Requires external PLII encoder for proper results
```

### 6.3 Mono to Stereo Upmix

```bash
# Duplicate mono to both channels
ffmpeg -i mono.wav -af "pan=stereo|c0=FC|c1=FC" stereo.wav

# Add slight delay to one channel for width
ffmpeg -i mono.wav -af "astats=metadata=1:reset=1,pan=stereo|c0=FC|c1=FC" stereo.wav

# Simple mono to 2.0
ffmpeg -i mono.wav -af "apan=1d|FL=FC|FR=FC" stereo.wav
```

---

## 7. MULTICHANNEL IN CONTAINERS

### 7.1 AC3 (Dolby Digital) Multichannel

AC3 supports up to 5.1 channels:

```bash
# Encode 5.1 to AC3
ffmpeg -i input_5.1.wav -c:a ac3 -b:a 640k output.ac3

# E-AC3 (Dolby Digital Plus) supports more channels
ffmpeg -i input_7.1.wav -c:a eac3 -b:a 1024k output.eac3

# Check AC3 encoder capabilities
ffmpeg -h encoder=ac3
```

### 7.2 DTS (Digital Theater Systems) Multichannel

```bash
# Encode to DTS
ffmpeg -i input_5.1.wav -c:a dca -b:a 1536k output.dts

# Passthrough DTS-HD
ffmpeg -i input.mkv -c:a copy output.mkv

# Fix channel order if needed
ffmpeg -i input_dts.mkv -af "channelmap=0|1|4|5|2|3:5.1" output_ac3.ac3
```

### 7.3 AAC Multichannel

```bash
# AAC-LC supports up to 8 channels (ffmpeg-native)
ffmpeg -i input_5.1.wav -c:a aac -b:a 256k output.m4a

# Use libfdk-aac for better quality (if available)
ffmpeg -i input_5.1.wav -c:a libfdk_aac -profile:a aac_he_v2 -b:a 128k output.m4a

# Note: ffmpeg-native AAC has channel limitations
# For 7.1, may need to downmix first
```

### 7.4 Opus Multichannel

Opus supports up to 255 channels via mapping family 255:

```bash
# Opus with automatic channel mapping
ffmpeg -i input_5.1.opus -c:a copy output_5.1.ogg

# Explicit 5.1 encoding with surround mapping
ffmpeg -i input_5.1.wav -c:a libopus -mapping_family 1 output.opus

# Independent channels (no speaker layout)
ffmpeg -i input_8ch.wav -c:a libopus -mapping_family 255 output.opus
```

### 7.5 FLAC Multichannel

```bash
# FLAC supports up to 8 channels natively
ffmpeg -i input_5.1.wav -c:a flac -compression_level 8 output_5.1.flac

# Verify channel layout
ffprobe -v error -select_streams a:0 \
  -show_entries stream=channels,channel_layout \
  -of default=noprint_wrappers=1 output_5.1.flac
```

### 7.6 Matroska (MKA) Multichannel

```bash
# MKA preserves any channel configuration
ffmpeg -i input_7.1.wav -c:a flac output_7.1.mka

# Verify in container
ffprobe -v error -select_streams a:0 \
  -show_entries stream=channels,channel_layout \
  -of default=noprint_wrappers=1 output_7.1.mka
```

### 7.7 MP4/M4A Multichannel

MP4 supports multichannel AAC, but channel configuration is limited:

```bash
# MP4 with 5.1 AAC
ffmpeg -i input_5.1.wav -c:a aac -b:a 512k -ar 48000 output_5.1.m4a

# Note: Some players may not support 5.1 in MP4
# May need to use M4A container or stay with AAC-LC stereo
```

### 7.8 Vorbis (OGG) Multichannel

```bash
# Vorbis in OGG supports multichannel
ffmpeg -i input_5.1.wav -c:a libvorbis -q:a 6 output_5.1.ogg

# Verify channel count
ffprobe -v error -select_streams a:0 \
  -show_entries stream=channels \
  -of default=noprint_wrappers=1 output_5.1.ogg
```

---

## 8. FFMPEG CHANNEL LAYOUT C API

### 8.1 Initializing Standard Layouts

```c
#include <libavutil/channel_layout.h>
#include <stdio.h>

void initialize_standard_layouts(void) {
    AVChannelLayout layout;
    
    // Initialize stereo layout
    av_channel_layout_default(&layout, 2);  // Sets to stereo
    char buf[256];
    av_channel_layout_describe(&layout, buf, sizeof(buf));
    printf("Default for 2 channels: %s\n", buf);  // "stereo"
    av_channel_layout_uninit(&layout);
    
    // Initialize 5.1 layout
    av_channel_layout_default(&layout, 6);  // Sets to 5.1
    av_channel_layout_describe(&layout, buf, sizeof(buf));
    printf("Default for 6 channels: %s\n", buf);  // "5.1"
    av_channel_layout_uninit(&layout);
    
    // Direct initialization
    av_channel_layout_from_string(&layout, "5.1");
    printf("5.1 has %d channels\n", layout.nb_channels);  // 6
    av_channel_layout_uninit(&layout);
}
```

### 8.2 Querying Channel Information

```c
#include <libavutil/channel_layout.h>

void query_channel_info(const AVChannelLayout *layout) {
    printf("Channels: %d\n", layout->nb_channels);
    
    // Get channel index
    int fl_idx = av_channel_layout_index_from_channel(layout, AV_CHAN_FRONT_LEFT);
    int fr_idx = av_channel_layout_index_from_channel(layout, AV_CHAN_FRONT_RIGHT);
    int lfe_idx = av_channel_layout_index_from_channel(layout, AV_CHAN_LOW_FREQUENCY);
    
    printf("FL index: %d\n", fl_idx);
    printf("FR index: %d\n", fr_idx);
    printf("LFE index: %d\n", lfe_idx);
    
    // Get channel from index
    enum AVChannel ch = av_channel_layout_channel_from_index(layout, 0);
    printf("Channel at index 0: %d\n", ch);
    
    // Check if layout has specific channel
    int has_lfe = av_channel_layout_has_channel(layout, AV_CHAN_LOW_FREQUENCY);
    printf("Has LFE: %s\n", has_lfe ? "Yes" : "No");
}
```

### 8.3 Custom Channel Layouts

```c
#include <libavutil/channel_layout.h>

int create_custom_layout(void) {
    AVChannelLayout layout;
    int ret;
    
    // Create custom 4-channel layout with specific channels
    AVChannelCustom custom_channels[4] = {
        { .id = AV_CHAN_FRONT_LEFT,  .name = "FL" },
        { .id = AV_CHAN_FRONT_RIGHT, .name = "FR" },
        { .id = AV_CHAN_BACK_LEFT,   .name = "BL" },
        { .id = AV_CHAN_BACK_RIGHT,  .name = "BR" },
    };
    
    ret = av_channel_layout_custom_init(&layout, 4);
    if (ret < 0) return ret;
    
    // Copy custom channel info
    for (int i = 0; i < 4; i++) {
        layout.u.map[i].id = custom_channels[i].id;
    }
    
    // Describe the layout
    char buf[256];
    av_channel_layout_describe(&layout, buf, sizeof(buf));
    printf("Custom layout: %s\n", buf);
    
    av_channel_layout_uninit(&layout);
    return 0;
}
```

### 8.4 Channel Remapping with libswresample

```c
#include <libswresample/swresample.h>

SwrContext* create_channel_remapper(
    const AVChannelLayout *in_layout,
    const AVChannelLayout *out_layout) {
    
    SwrContext *swr = swr_alloc();
    if (!swr) return NULL;
    
    // Set channel layouts
    av_opt_set_chlayout(swr, "in_ch_layout", in_layout, 0);
    av_opt_set_chlayout(swr, "out_ch_layout", out_layout, 0);
    
    // Initialize
    int ret = swr_init(swr);
    if (ret < 0) {
        swr_free(&swr);
        return NULL;
    }
    
    return swr;
}

int remap_5_1_to_stereo(SwrContext *swr, AVFrame *in_frame, AVFrame *out_frame) {
    int ret;
    
    // Calculate output samples
    int out_samples = av_rescale_rnd(
        swr_get_delay(swr, in_frame->sample_rate) + in_frame->nb_samples,
        out_frame->sample_rate, in_frame->sample_rate, AV_ROUND_UP);
    
    if (out_samples > out_frame->nb_samples) {
        av_frame_unref(out_frame);
        out_frame->nb_samples = out_samples;
        ret = av_frame_get_buffer(out_frame, 0);
        if (ret < 0) return ret;
    }
    
    // Perform remapping/downmixing
    ret = swr_convert(swr,
                      out_frame->extended_data, out_frame->nb_samples,
                      (const uint8_t **)in_frame->extended_data, in_frame->nb_samples);
    if (ret < 0) return ret;
    
    out_frame->nb_samples = ret;
    return 0;
}
```

### 8.5 Using the pan Filter Programmatically

```c
#include <libavfilter/avfilter.h>
#include <libavfilter/buffersrc.h>
#include <libavfilter/buffersink.h>

AVFilterGraph* create_pan_filter_graph(
    const AVChannelLayout *in_layout,
    const AVChannelLayout *out_layout,
    const char *pan_expression) {
    
    AVFilterGraph *graph = avfilter_graph_alloc();
    if (!graph) return NULL;
    
    // Create buffer source
    AVFilter *buffersrc = avfilter_get_by_name("abuffer");
    AVFilterContext *buffersrc_ctx;
    
    char args[512];
    snprintf(args, sizeof(args), "time_base=1/%d:sample_rate=%d:ch_layout=%dch",
             48000, 48000, in_layout->nb_channels);
    
    int ret = avfilter_graph_create_filter(&buffersrc_ctx, buffersrc, "in",
                                            args, NULL, graph);
    if (ret < 0) return NULL;
    
    // Create pan filter
    AVFilter *pan = avfilter_get_by_name("apad");
    AVFilterContext *pan_ctx;
    
    ret = avfilter_graph_create_filter(&pan_ctx, pan, "pan", pan_expression, NULL, graph);
    if (ret < 0) return NULL;
    
    // Create buffer sink
    AVFilter *buffersink = avfilter_get_by_name("abuffersink");
    AVFilterContext *buffersink_ctx;
    
    ret = avfilter_graph_create_filter(&buffersink_ctx, buffersink, "out", NULL, NULL, graph);
    if (ret < 0) return NULL;
    
    // Set output channel layout
    int channel_count = out_layout->nb_channels;
    av_opt_set_int(buffersink_ctx, "channel_count", channel_count, AV_OPT_SEARCH_CHILDREN);
    
    // Connect filters
    avfilter_link(buffersrc_ctx, 0, pan_ctx, 0);
    avfilter_link(pan_ctx, 0, buffersink_ctx, 0);
    
    // Configure graph
    ret = avfilter_graph_config(graph, NULL);
    if (ret < 0) return NULL;
    
    return graph;
}
```

---

## 9. DOWNMIX COEFFICIENTS REFERENCE

### 9.1 Standard Downmix Matrices

**5.1 → Stereo (ITU-R BS.775):**

```
FL_out = 1.0×FL + 0.7071×FC + 0.7071×BL + 0.5×LFE
FR_out = 1.0×FR + 0.7071×FC + 0.7071×BR + 0.5×LFE
```

**5.1 → Stereo (Pro Logic II compatible):**

```
FL_out = 1.0×FL + 0.7071×FC + 0.5×SL + 0.7071×BL + 0.5×LFE
FR_out = 1.0×FR + 0.7071×FC + 0.5×SR + 0.7071×BR + 0.5×LFE
```

**7.1 → 5.1:**

```
FL_out = FL
FR_out = FR
FC_out = FC + 0.7071×FLC + 0.7071×FRC
LFE_out = LFE
SL_out = 1.0×SL + 0.7071×BL
SR_out = 1.0×SR + 0.7071×BR
```

### 9.2 LFE Considerations

| Scenario | LFE Recommendation |
|----------|-------------------|
| Home theater | Include LFE (typically at 0.5×) |
| Music stereo | Exclude LFE (not present in stereo source) |
| Cinema | Include LFE at full level |
| Portable devices | Exclude (often no subwoofer) |

---

## 10. LFE HANDLING

### 10.1 LFE Channel Position

In FFmpeg's WAV-order channel layouts, LFE is always at index 3 (zero-based):

```c
// 5.1 channel order in FFmpeg:
static const int ff_avapi_channel_map_5_1[6] = {
    0,  // FL - Front Left
    1,  // FR - Front Right
    2,  // FC - Front Center
    3,  // LFE - Low Frequency Effects (position 3)
    4,  // SL - Side Left (or Back Left)
    5,  // SR - Side Right (or Back Right)
};
```

### 10.2 LFE in Different Containers

| Container | LFE Position | Notes |
|-----------|--------------|-------|
| WAV | Index 3 | Standard WAV order |
| AIFF | Index 3 | Same as WAV |
| AC3 | Index 3 | Standard |
| DTS | Index 3 | Standard |
| FLAC | Index 3 | Standard |
| MP4/M4A | Index 3 | AAC channel order |
| OGG/Vorbis | Index 3 | Standard |
| Opus | Depends on mapping | See RFC 7845 |

### 10.3 LFE Filter for Low-Pass

```bash
# Extract and process LFE separately
ffmpeg -i input_5.1.wav -af "asplit[fc][lfe];[lfe]lowpass=f=120,volume=1.5" lfe_processed.wav

# Apply lowpass to LFE channel in surround
ffmpeg -i input_5.1.wav \
  -af "pan=5.1|c0=FL|c1=FR|c2=FC|c3<LFE:c4=SL|c5=SR,lowpass=f=120" \
  output.wav
```

---

## 11. FFmpeg -ac Flag vs pan Filter

### 11.1 The -ac Flag

The `-ac` flag performs automatic channel count reduction:

```bash
# Reduce to 2 channels (stereo)
ffmpeg -i input_5.1.wav -ac 2 output_stereo.wav

# Reduce to 1 channel (mono)
ffmpeg -i input_5.1.wav -ac 1 output_mono.wav
```

**Note:** The `-ac` flag does NOT use the pan filter. It uses FFmpeg's internal automatic downmix which:
- May discard LFE
- Uses standard matrix coefficients
- Cannot include custom mixing

### 11.2 When to Use -ac vs pan

| Use Case | Recommended Method |
|----------|-------------------|
| Simple channel reduction | `-ac 2` |
| Need LFE included | `pan` filter |
| Custom coefficients | `pan` filter |
| Professional downmix | `pan` filter with ITU coefficients |
| Quick stereo extraction | `-ac 2` |
| Preserving all channels | `-c:a copy` |

---

## 12. BIT-EXACT MULTICHANNEL

### 12.1 Preserving All Channels

```bash
# Copy audio stream (preserves all channels)
ffmpeg -i input_7.1.mkv -c:a copy output_7.1.mkv

# Verify channel count
ffprobe -v error -select_streams a:0 \
  -show_entries stream=channels \
  -of default=noprint_wrappers=1 output_7.1.mkv
```

### 12.2 Verifying Channel Layout

```bash
# Detailed channel layout information
ffprobe -v error -select_streams a:0 \
  -show_entries stream=channel_layout,channels,bits_per_raw_sample \
  -of default=noprint_wrappers=1 input_5.1.flac

# Output example:
# channel_layout=5.1
# channels=6
# bits_per_raw_sample=24
```

### 12.3 Common Layout Issues

| Issue | Symptom | Solution |
|-------|---------|----------|
| Wrong side/back designation | Speakers play wrong channels | Use correct layout variant (5.1 vs 5.1(back)) |
| LFE missing | No subwoofer output | Include LFE in pan filter explicitly |
| Center channel quiet | Dialogue hard to hear | Adjust center channel coefficient in pan |
| Channels swapped | L/R reversed | Use pan to reorder: `c0=c1|c1=c0` |

---

## 13. MULTICHANNEL IN SPECIFIC FORMATS

### 13.1 AAC Channel Configurations

| Channels | Configuration | Notes |
|----------|--------------|-------|
| 1 | SCE (Single Channel Element) | Mono |
| 2 | SCE + CPE (Channel Pair Element) | Stereo |
| 3 | SCE + CPE | 3.0 |
| 4 | SCE + CPE + SCE | 4.0 |
| 5 | SCE + CPE + SCE | 5.0 |
| 6 | SCE + CPE + LFE | 5.1 |

### 13.2 DTS Channel Configurations

| DTS Mode | Channels | Configuration |
|----------|----------|---------------|
| DTS 1/0/0 | 1 | Mono |
| DTS 2/0/0 | 2 | Stereo |
| DTS 3/2/0 | 5 | L, R, C, SL, SR |
| DTS 3/2/1 | 6 | L, R, C, SL, SR, LFE |
| DTS 3/4/0 | 6 | L, R, C, TFL, TFR, BC |
| DTS 3/4/1 | 7 | L, R, C, TFL, TFR, BC, LFE |
| DTS 7.1 | 8 | + Back channels |

### 13.3 AC3 Channel Modes

| Mode | Value | Channels | Configuration |
|------|-------|----------|---------------|
| AC3_CHMODE_DUALMONO | 0 | 2 | Dual mono |
| AC3_CHMODE_MONO | 1 | 1 | C |
| AC3_CHMODE_STEREO | 2 | 2 | L, R |
| AC3_CHMODE_3F | 3 | 3 | L, R, C |
| AC3_CHMODE_2F1R | 4 | 3 | L, R, SR |
| AC3_CHMODE_3F1R | 5 | 4 | L, R, C, SR |
| AC3_CHMODE_2F2R | 6 | 4 | L, R, SL, SR |
| AC3_CHMODE_3F2R | 7 | 5 | L, R, C, SL, SR |

---

## 14. STREAMING MULTICHANNEL

### 14.1 HLS with Multichannel Audio

```bash
# Create HLS with 5.1 AC3 audio
ffmpeg -i input_5.1.mkv \
  -c:v libx264 -c:a ac3 -b:a 640k \
  -hls_time 4 -hls_list_size 0 \
  -hls_segment_filename segment_%03d.ts \
  playlist.m3u8
```

### 14.2 DASH with Multichannel

```bash
# DASH with multichannel audio
ffmpeg -i input_5.1.mkv \
  -c:v libx264 -c:a ac3 -b:a 640k \
  -f dash \
  -init_seg_name init_$RepresentationID$.m4s \
  -media_seg_name segment_$RepresentationID$_$Number$.m4s \
  output.mpd
```

---

## 15. KNOWN ISSUES AND TROUBLESHOOTING

### 15.1 Common Problems

| Problem | Cause | Solution |
|---------|-------|----------|
| "Layout not supported" | Encoder doesn't support channel count | Downmix or use different encoder |
| Channels in wrong order | Container uses different ordering | Use pan filter to remap |
| No LFE output | LFE filtered out by default | Use pan filter with explicit LFE |
| 5.1 not working in MP4 | Player limitation | Use M4A or add stereo fallback |
| Surround sounds like mono | All channels mixed to one | Verify channel layout is correct |

### 15.2 Channel Order Discrepancies

Different formats use different channel orders:

| Format | 5.1 Order |
|--------|------------|
| WAV/AIFF | FL, FR, FC, LFE, SL, SR |
| DTS | FL, FR, FC, LFE, SL, SR |
| AAC (MPEG-2) | FL, FR, FC, LFE, SL, SR |
| AAC (MPEG-4) | FL, FR, FC, LFE, SL, SR |
| Vorbis | FL, FR, FC, LFE, SL, SR |

**Note:** Some DTS sources may have back/side confusion. Always verify with test tones.

### 15.3 Decoder-Specific Issues

| Decoder | Issue | Workaround |
|---------|-------|------------|
| DTS-MA passthrough | Limited player support | Decode to PCM |
| E-AC3 | Some players downmix to stereo | Provide stereo track |
| AAC 7.1 | Limited codec support | Use 5.1 or stereo |
| MLP | Only available in TrueHD | N/A |

---

## 16. REFERENCE COMMANDS

### 16.1 Common Multichannel Operations

```bash
# 1. Extract stereo from 5.1
ffmpeg -i input_5.1.mkv -af "pan=stereo|FL=FL+0.707*FC|FR=FR+0.707*FC" output_stereo.wav

# 2. Convert 7.1 to 5.1
ffmpeg -i input_7.1.mkv -af "pan=5.1|c0=FL|c1=FR|c2=FC|c3=LFE|c4=BL+0.5*SL|c5=BR+0.5*SR" output_5.1.wav

# 3. Fix channel order in DTS
ffmpeg -i input_dts.mkv -af "channelmap=0|1|4|5|2|3:5.1" output_fixed.ac3

# 4. Join multiple mono files to 5.1
ffmpeg -i FL.wav -i FR.wav -i FC.wav -i LFE.wav -i SL.wav -i SR.wav \
  -filter_complex "[0:a][1:a][2:a][3:a][4:a][5:a]join=inputs=6:channel_layout=5.1" \
  output_5.1.wav

# 5. Extract LFE for separate processing
ffmpeg -i input_5.1.mkv -af "pan=1c|c0=LFE" lfe_only.wav

# 6. Create stereo from 5.1 with LFE
ffmpeg -i input_5.1.mkv -af "pan=stereo|FL=FL+0.707*FC+0.5*LFE|FR=FR+0.707*FC+0.5*LFE" output.wav
```

### 16.2 Format-Specific Commands

```bash
# AC3 5.1 encoding
ffmpeg -i input.wav -c:a ac3 -b:a 640k output.ac3

# E-AC3 7.1 encoding
ffmpeg -i input.wav -c:a eac3 -b:a 1024k output.eac3

# DTS encoding
ffmpeg -i input.wav -c:a dca -b:a 1536k output.dts

# FLAC 5.1 (lossless)
ffmpeg -i input.wav -c:a flac -compression_level 8 output_5.1.flac

# Opus 5.1
ffmpeg -i input.wav -c:a libopus -mapping_family 1 output.opus
```

---

## 17. IMPLEMENTATION CHECKLIST

### Build & Environment
- [ ] Verify FFmpeg supports desired channel layouts
- [ ] Check encoder support for multichannel (`ffmpeg -encoders | grep ac3`)
- [ ] Verify container supports multichannel

### Encoding Pipeline
- [ ] Set correct channel layout (`-channel_layout 5.1`)
- [ ] Verify encoder supports channel count
- [ ] Use appropriate bitrate for multichannel
- [ ] Apply proper downmix coefficients if needed

### Decoding/Processing
- [ ] Handle channel layout from source correctly
- [ ] Apply correct channel remapping if needed
- [ ] Include/exclude LFE as appropriate
- [ ] Verify output has expected channel count

### Quality & Verification
- [ ] Test with multichannel audio files
- [ ] Verify all channels are present
- [ ] Check LFE is functioning
- [ ] Test playback on multichannel system

### Edge Cases
- [ ] Handle files with unknown channel order
- [ ] Handle mismatched channel layouts
- [ ] Handle too many channels for encoder
- [ ] Handle LFE-only streams

---

## 18. CHANNEL LAYOUT CONSTANTS REFERENCE

### 18.1 Predefined Layout Macros

```c
// From libavutil/channel_layout.h

// Single channel
#define AV_CHANNEL_LAYOUT_MONO { AV_CHANNEL_ORDER_NATIVE, 1, { .mask = AV_CH_FRONT_CENTER }, NULL }

// Stereo
#define AV_CHANNEL_LAYOUT_STEREO { AV_CHANNEL_ORDER_NATIVE, 2, { .mask = AV_CH_FRONT_LEFT | AV_CH_FRONT_RIGHT }, NULL }

// 2.1
#define AV_CHANNEL_LAYOUT_2POINT1 { AV_CHANNEL_ORDER_NATIVE, 3, { .mask = AV_CH_FRONT_LEFT | AV_CH_FRONT_RIGHT | AV_CH_LOW_FREQUENCY }, NULL }

// 5.0
#define AV_CHANNEL_LAYOUT_5POINT0 { AV_CHANNEL_ORDER_NATIVE, 5, { .mask = AV_CH_FRONT_LEFT | AV_CH_FRONT_RIGHT | AV_CH_FRONT_CENTER | AV_CH_SIDE_LEFT | AV_CH_SIDE_RIGHT }, NULL }

// 5.1
#define AV_CHANNEL_LAYOUT_5POINT1 { AV_CHANNEL_ORDER_NATIVE, 6, { .mask = AV_CH_FRONT_LEFT | AV_CH_FRONT_RIGHT | AV_CH_FRONT_CENTER | AV_CH_LOW_FREQUENCY | AV_CH_SIDE_LEFT | AV_CH_SIDE_RIGHT }, NULL }

// 5.1 (back)
#define AV_CHANNEL_LAYOUT_5POINT1_BACK { AV_CHANNEL_ORDER_NATIVE, 6, { .mask = AV_CH_FRONT_LEFT | AV_CH_FRONT_RIGHT | AV_CH_FRONT_CENTER | AV_CH_LOW_FREQUENCY | AV_CH_BACK_LEFT | AV_CH_BACK_RIGHT }, NULL }

// 7.1
#define AV_CHANNEL_LAYOUT_7POINT1 { AV_CHANNEL_ORDER_NATIVE, 8, { .mask = AV_CH_FRONT_LEFT | AV_CH_FRONT_RIGHT | AV_CH_FRONT_CENTER | AV_CH_LOW_FREQUENCY | AV_CH_BACK_LEFT | AV_CH_BACK_RIGHT | AV_CH_SIDE_LEFT | AV_CH_SIDE_RIGHT }, NULL }

// And more...
```

### 18.2 Quick Reference: CLI Channel Layout Options

```bash
# Show all layouts
ffmpeg -layouts

# Set layout
-channel_layout <name>
-ch_layout <name>

# Set by channel count (auto-selects default)
-ac <count>

# Examples
-channel_layout 5.1
-channel_layout "7.1(wide)"
-ch_layout stereo
-ac 2
```

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
