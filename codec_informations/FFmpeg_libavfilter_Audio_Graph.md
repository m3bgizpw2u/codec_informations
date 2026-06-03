# FFmpeg libavfilter — Audio Filter Graph — Deep Technical Reference
> **Category:** FFmpeg API
> **File Extensions:** N/A (FFmpeg library)
> **MIME Types:** N/A (FFmpeg library)
> **Standardization Body:** FFmpeg Project
> **Primary Specification:** https://ffmpeg.org/doxygen/nav.html?index.html#Files
> **Patent Status:** N/A (API library)
> **License:** LGPL 2.1+ / GPL 2+ (depending on components)
> **Current Version:** FFmpeg n8.1.1 (as of 2026)
> **Active Development:** Yes — active maintenance and releases

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** FFmpeg Project (originally based on Audio/video/filters from various sources)
- **Year Created:** 2001–2004 (libavfilter separated from libavcodec during FFmpeg fork around 2005)
- **Original Purpose:** Provide a graph-based audio and video processing pipeline within FFmpeg. libavfilter allows chaining arbitrary processing stages (resampling, volume adjustment, mixing, effects) without requiring re-encoding between each operation.
- **Problem with Predecessors:** Before libavfilter, FFmpeg applied filters as standalone pre/post-processing steps. Users needed external tools for complex chains (volume normalization → resampling → mixing). libavfilter unified video and audio filtering under one graph-based architecture.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| Initial libavfilter | 2005 | Graph-based filter architecture, basic audio/video filters |
| Filter registration API | 2006–2008 | Stable filter registration, AVFilter ecosystem |
| Complex filtergraphs | 2009 | `-filter_complex` for multi-input/output graphs |
| Dynamic filters | 2012–2015 | Dynamic channel layouts, filter graph reconfiguration |
| Segment API | 2018–2020 | Filter graph segment parsing and manipulation |
| Thread safety improvements | 2022–2024 | Multi-threaded filter graph processing |

### 1.3 Current Adoption
- **Primary use cases today:** Audio normalization, resampling, channel mixing, volume adjustment, multi-track audio processing, video filtering, complex filter graphs in transcoding pipelines, live streaming with real-time effects
- **Platforms with native support:** All platforms supported by FFmpeg (Linux, macOS, Windows, mobile OSes)
- **Major services using this format:** YouTube (transcoding), Twitch (live filtering), FFmpeg-based transcoding services, broadcast automation
- **Hardware support:** FFmpeg filtergraphs can incorporate hardware-accelerated filters where available
- **Status:** Ubiquitous in FFmpeg-based audio/video processing pipelines

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Architecture type:** Graph-based filter pipeline (directed acyclic graph, DAG)
- **Core concepts:** Filter pads (inputs/outputs), filter chains, filter graphs
- **Processing model:** Frames (AVFrame) flow through filters; each filter transforms input frames into output frames
- **Audio-specific processing:** Sample-based processing, format negotiation, channel layout handling

### 2.2 High-Level Filter Graph Architecture
```
Input Audio Stream (decoded)
        │
        ▼
┌─────────────────────────────────────────────┐
│  Buffer Source (buffersrc)                   │
│  - Generates frames from decoded audio        │
│  - Sets format context                       │
└─────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────┐
│  Filter 1: aformat                          │
│  - Converts sample format                    │
│  - Negotiates channel layout                 │
│  - Handles sample rate if needed             │
└─────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────┐
│  Filter 2: volume                           │
│  - Adjusts audio level                      │
│  - Supports dB and linear gain               │
└─────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────┐
│  Filter 3: aresample                        │
│  - Sample rate conversion                    │
│  - Uses libswresample internally            │
└─────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────┐
│  Filter N: pan / amix / etc.                │
│  - Channel mixing / combining                │
└─────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────────────┐
│  Buffer Sink (buffersink)                   │
│  - Collects processed frames                │
│  - Returns frames to caller                 │
└─────────────────────────────────────────────┘
        │
        ▼
Output (encoded or raw audio)
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Filter graph depth | Unlimited (DAG) | Acyclic; cycles require feedback loops |
| Max inputs per filter | Unlimited (per filter definition) | Most audio filters have 1 input |
| Max outputs per filter | Unlimited (per filter definition) | split, ashuffle support N outputs |
| Frame buffer size | Per-filter internal | Configurable via filter options |
| Thread safety | Varies by filter | Check individual filter documentation |
| Format negotiation | Automatic | Most filters auto-negotiate input/output formats |

---

## 3. FILTER GRAPH DESCRIPTION STRING SYNTAX

### 3.1 Filter Graph String Format
The filter graph description string uses two separator characters with different precedence:

```
Filter Chain Separator: ";" (semicolon)
  - Separates independent filter chains
  - Each chain is processed separately
  - Output of chain N does not flow to chain N+1 automatically

Filter Separator: "," (comma)
  - Connects filters within a chain
  - Output of filter N becomes input of filter N+1
```

#### Syntax Rules
```
[input_label]filter_name=options[output_label];[input_label]filter_name=options[output_label]

Where:
  input_label  = optional named input pad (for complex filtergraphs)
  filter_name  = name of the filter (e.g., "volume", "aresample")
  options      = filter-specific options in key=value or positional format
  output_label = optional named output pad (for complex filtergraphs)
```

### 3.2 Filter Graph Parsing Examples

#### Simple Chain
```bash
# "volume=0.5,aresample=48000" means:
# Input → volume(filter) → aresample(filter) → Output
ffmpeg -i input.wav -af "volume=0.5,aresample=48000" output.wav
```

#### Multiple Chains
```bash
# "volume=0.5;amerge" means:
# Chain 1: Input → volume → (nothing connected to next)
# Chain 2: Input → amerge → Output
# Note: For amerge to work, it needs multiple inputs from the original
```

#### Complex Filtergraph with Named Pads
```bash
ffmpeg -i input.wav -filter_complex \
  "[0:a]volume=0.5[vol];[vol]aresample=48000[out]" \
  -map "[out]" output.wav
```

### 3.3 Filter Graph Parsing in C API
```c
#include <libavfilter/avfilter.h>
#include <libavfilter/buffersrc.h>
#include <libavfilter/buffersink.h>
#include <libavfilter/graphparser.h>

// Method 1: avfilter_graph_parse_ptr (legacy, deprecated in newer FFmpeg)
AVFilterGraph *graph = avfilter_graph_alloc();

// Create source and sink filters manually
AVFilterContext *src_ctx, *sink_ctx;
avfilter_graph_create_filter(&src_ctx, avfilter_get_by_name("abuffer"),
                              "in", NULL, NULL, graph);
avfilter_graph_create_filter(&sink_ctx, avfilter_get_by_name("abuffersink"),
                              "out", NULL, NULL, graph);

// Prepare AVFilterInOut for inputs/outputs
AVFilterInOut *inputs = avfilter_inout_alloc();
AVFilterInOut *outputs = avfilter_inout_alloc();

outputs->filter_ctx = src_ctx;
outputs->pad_idx = 0;
outputs->name = av_strdup("in");

inputs->filter_ctx = sink_ctx;
inputs->pad_idx = 0;
inputs->name = av_strdup("out");

// Parse filter string
ret = avfilter_graph_parse_ptr(graph, "volume=0.5,aresample=48000",
                                &inputs, &outputs, NULL);
if (ret < 0) {
    char errbuf[128];
    av_strerror(ret, errbuf, sizeof(errbuf));
    fprintf(stderr, "Parse error: %s\n", errbuf);
}

// Configure the graph
ret = avfilter_graph_config(graph, NULL);
if (ret < 0) { /* error */ }

// Method 2: avfilter_graph_segment_* (newer API, FFmpeg 6.0+)
// Parse into segments without creating filters yet
AVFilterGraphSegment *seg = NULL;
ret = avfilter_graph_segment_parse(graph, "volume=0.5,aresample=48000",
                                   0, &seg);
```

---

## 4. KEY AUDIO FILTERS — COMPLETE REFERENCE

### 4.1 aresample — Audio Resampling

**Purpose:** Convert audio sample rate and optionally channel layout using libswresample.

```bash
# Basic resampling
ffmpeg -i input.wav -af "aresample=48000" output.wav

# With resampler options
ffmpeg -i input.wav -af "aresample=48000:resampler=soxr:precision=28" output.wav
```

**aresample Options:**
| Option | Type | Default | Range | Description |
|--------|------|---------|-------|-------------|
| sample_rate | int | 0 | >0 | Output sample rate (Hz) |
| resampler | string | soxr | soxr,swr | Resampling engine |
| dither_method | int | 0 | 0–9 | Dithering algorithm |
| filter_size | int | 16 | 4–4096 | Polyphase filter size |
| phase_shift | int | 10 | 4–30 | Number of phases |
| linear_interp | bool | false | true/false | Linear interpolation |
| cutoff | float | 0.8 | 0.0–1.0 | Cutoff frequency (relative to Nyquist) |

**Supported Sample Rates:**
All standard rates from 1 Hz to 384000 Hz are accepted. Common rates: 8000, 11025, 16000, 22050, 32000, 44100, 48000, 64000, 88200, 96000, 176400, 192000, 352800, 384000.

### 4.2 aformat — Audio Format Conversion

**Purpose:** Convert sample format, channel layout, and sample rate to constrain output format.

```bash
# Convert to specific format
ffmpeg -i input.wav -af "aformat=sample_fmts=fltp:sample_rates=48000:channel_layouts=stereo" output.wav

# Multiple allowed values (FFmpeg picks best)
ffmpeg -i input.wav -af "aformat=sample_fmts=u8\|s16\|fltp:channel_layouts=mono\|stereo" output.wav
```

**aformat Options:**
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| sample_fmts | string | all | '\|'-separated list: u8, s16, s32, flt, dbl, u8p, s16p, s32p, fltp, dblp |
| sample_rates | string | all | '\|'-separated list of sample rates |
| channel_layouts | string | all | '\|'-separated channel layouts |

**Sample Format Meanings:**
| Format | Meaning | Planar |
|--------|---------|--------|
| u8 | unsigned 8-bit | No |
| s16 | signed 16-bit | No |
| s32 | signed 32-bit | No |
| flt | 32-bit float | No |
| dbl | 64-bit float | No |
| u8p | unsigned 8-bit | Yes |
| s16p | signed 16-bit | Yes |
| s32p | signed 32-bit | Yes |
| fltp | 32-bit float | Yes |
| dblp | 64-bit float | Yes |

### 4.3 volume — Audio Volume Adjustment

**Purpose:** Adjust audio volume (gain) in linear or logarithmic (dB) scale.

```bash
# Linear gain
ffmpeg -i input.wav -af "volume=2.0" output.wav      # Double volume

# dB gain
ffmpeg -i input.wav -af "volume=3dB" output.wav       # +3 dB

# Expression (variables: tc, az, sa, smax, smin, peak)
ffmpeg -i input.wav -af "volume='min(1.0,peak*1.5)':eval=frame" output.wav

# ReplayGain-like normalization
ffmpeg -i input.wav -af "volumedetect" -f null -    # Check peak level first
ffmpeg -i input.wav -af "volume=-3dB" output.wav    # Then apply
```

**volume Options:**
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| volume | string | 1.0 | Volume adjustment (linear or dB) |
| precision | string | fixed | fixed or float precision mode |
| eval | string | once | once or frame (per-frame evaluation) |
| maxval | float | 1.0 | Maximum allowed value |
| replaygain | string | drop | off, drop, overwrite, ignore |
| replaygain_preamp | float | 0.0 | ReplayGain pre-amp in dB |
| replaygain_noclip | bool | true | Prevent clipping |

**dB to Linear Conversion:**
```
linear = 10^(dB/20)
  -6 dB  → 0.501
  -3 dB  → 0.708
   0 dB  → 1.000
  +3 dB  → 1.413
  +6 dB  → 2.000
```

### 4.4 pan — Channel Remapping and Mixing

**Purpose:** Remap channels, downmix, or upmix audio channels with custom coefficients.

```bash
# Stereo to mono (average)
ffmpeg -i stereo.wav -af "pan=mono|c0=0.5*c0+0.5*c1" output.wav

# 5.1 to stereo with LFE
ffmpeg -i 5.1.wav -af "pan=stereo|FL=FL+0.5*FC+0.5*LFE|FR=FR+0.5*FC+0.5*LFE" output.wav

# Upmix mono to stereo
ffmpeg -i mono.wav -af "pan=stereo|c0=c0|c1=c0" output.wav

# Remap specific channels
ffmpeg -i 5.1.wav -af "pan=3c|c0=c2|c1=c0|c2=c1" output.wav   # Swap front channels

# Use < instead of = for renormalization
ffmpeg -i stereo.wav -af "pan=stereo|FL<FL+FC|FR<FR+FC" output.wav   # Add center to sides
```

**pan Syntax:**
```
pan syntax: "l|outdef1|outdef2|..."

Where:
  l           = output channel layout (stereo, 5.1, mono, or N channels)
  outdefN     = output channel definition: "out_name=[gain*]in_name[(+-)[gain*]in_name...]"

Gain modifiers:
  =           = set channel (clipping possible if sum > 1.0)
  <           = set channel with automatic renormalization to sum = 1.0
  +           = add to existing channel
  -           = subtract from existing channel
```

**Standard Channel Names:**
| Abbreviation | Full Name | Channel Number |
|--------------|-----------|----------------|
| FL | Front Left | c0 |
| FR | Front Right | c1 |
| FC | Front Center | c2 |
| LFE | Low Frequency Effects | c3 |
| BL | Back Left | c4 |
| BR | Back Right | c5 |
| SL | Side Left | c6 |
| SR | Side Right | c7 |
| FLC | Front Left of Center | c8 |
| FRC | Front Right of Center | c9 |
| BC | Back Center | c10 |

### 4.5 channelsplit — Split Audio into Per-Channel Streams

**Purpose:** Split multi-channel audio into separate mono streams for individual processing.

```bash
# Split stereo into two mono streams
ffmpeg -i stereo.wav -filter_complex "channelsplit=channel_layout=stereo[fl][fr]" \
  -map "[fl]" left.wav -map "[fr]" right.wav

# Split 5.1 into 6 mono streams
ffmpeg -i 5.1.wav -filter_complex "channelsplit=channel_layout=5.1[fl][fr][fc][lfe][bl][br]" \
  -map "[fl]" front_left.wav \
  -map "[fr]" front_right.wav \
  -map "[fc]" front_center.wav \
  -map "[lfe]" lfe.wav \
  -map "[bl]" back_left.wav \
  -map "[br]" back_right.wav
```

**channelsplit Options:**
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| channel_layout | string | input | Input channel layout |
| channels | string | all | Specific channels to extract |

### 4.6 achannelmix — Channel Mixing with Matrix

**Purpose:** Mix channels using a user-specified matrix, with support for normalization.

```bash
# 5.1 to stereo with mix levels
ffmpeg -i 5.1.wav -af "achannelmix=mix=FL+0.707*FC+0.5*LFE:FR+0.707*FC+0.5*LFE" output.wav

# Upmix with normalization
ffmpeg -i mono.wav -af "achannelmix=mix=mono*2:normalize=1" output.wav
```

**achannelmix Options:**
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| in | string | all | Input channel specification |
| out | string | all | Output channel specification |
| mix | string | normalized | Mix formula per output channel |
| normalize | bool | true | Renormalize output to prevent clipping |

### 4.7 asplit / split — Multi-Output Splitter

**Purpose:** Duplicate input to multiple outputs for parallel processing.

```bash
# Split into 3 identical outputs
ffmpeg -i input.wav -af "asplit=3[out1][out2][out3]" \
  -map "[out1]" out1.wav \
  -map "[out2]" out2.wav \
  -map "[out3]" out3.wav

# One output for monitoring, one for file
ffmpeg -i input.wav -af "asplit=2[monitor][file]" \
  -map "[file]" output.wav \
  -map "[monitor]" -f null -
```

**asplit Options:**
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| outputs | int | 2 | Number of output streams (1–64) |

### 4.8 amerge — Merge Multiple Audio Streams

**Purpose:** Combine multiple mono or multi-channel audio streams into a single multi-channel stream.

```bash
# Merge two stereo files into 4-channel audio
ffmpeg -i left.wav -i right.wav -filter_complex "amerge=inputs=2" output_4ch.wav

# Merge 6 mono files into 5.1
ffmpeg -i fl.wav -i fr.wav -i fc.wav -i lfe.wav -i bl.wav -i br.wav \
  -filter_complex "[0:a][1:a][2:a][3:a][4:a][5:a]amerge=inputs=6" output_5.1.wav
```

**amerge Options:**
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| inputs | int | 2 | Number of input streams |

### 4.9 amix — Mix Multiple Audio Streams

**Purpose:** Mix multiple audio inputs into one with optional weighting and dropout handling.

```bash
# Mix two audio files with equal gain
ffmpeg -i file1.wav -i file2.wav -filter_complex "amix=inputs=2:duration=first" output.wav

# Mix with dropout handling
ffmpeg -i file1.wav -i file2.wav -filter_complex "amix=inputs=2:dropout_transition=2" output.wav

# Weighted mixing
ffmpeg -i file1.wav -i file2.wav -filter_complex "amix=inputs=2:weights=2:1" output.wav
```

**amix Options:**
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| inputs | int | 2 | Number of inputs |
| duration | string | longest | How long to mix (longest, shortest, first) |
| dropout_transition | int | 2 | Time in seconds for input transitions |
| weights | string | 1:1 | Weight for each input (colon-separated) |
| normalize | bool | true | Normalize output to prevent clipping |

### 4.10 adelay — Audio Delay

**Purpose:** Add delay to audio channels (useful for alignment or effects).

```bash
# Delay left channel by 100ms
ffmpeg -i stereo.wav -af "adelay=100|0" output.wav

# Delay both channels
ffmpeg -i stereo.wav -af "adelay=100|100" output.wav

# Different delay per channel (100ms left, 200ms right)
ffmpeg -i stereo.wav -af "adelay=100|200" output.wav
```

### 4.11 aecho — Echo Effect

**Purpose:** Add echo (reflections) to audio.

```bash
# Simple echo
ffmpeg -i input.wav -af "aecho=0.8:0.9:500|1000:0.3|0.25" output.wav

# Multiple echoes
ffmpeg -i input.wav -af "aecho=0.8:0.9:500|1000|1500:0.3|0.25|0.2" output.wav
```

**aecho Options:**
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| in_gain | float | 0.6 | Input gain (0.0–1.0) |
| out_gain | float | 0.3 | Output gain (0.0–1.0) |
| delays | string | 1000 | Delay times in ms (pipe-separated) |
| decays | string | 0.5 | Decay factors per delay (pipe-separated) |

### 4.12 afade — Audio Fade In/Out

**purpose:** Apply fade-in or fade-out effects.

```bash
# Fade in over 3 seconds
ffmpeg -i input.wav -af "afade=t=in:ss=0:d=3" output.wav

# Fade out over 2 seconds starting at 30 seconds
ffmpeg -i input.wav -af "afade=t=out:st=30:d=2" output.wav

# Crossfade between two files
ffmpeg -i file1.wav -i file2.wav -filter_complex \
  "[0:a]afade=t=out:st=0:d=2[fa];[1:a]afade=t=in:st=0:d=2[fb];[fa][fb]acrossfade=d=2" \
  output.wav
```

**afade Options:**
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| type | string | in | in, out, or ts (transfers shape) |
| start_sample | int64 | 0 | Start sample position |
| nb_samples | int64 | 0 | Number of samples to fade |
| start_time | string | 0 | Start time in seconds |
| duration | string | 0 | Duration in seconds |
| curve | string | tri | Fade curve (tri, hsin, etc.) |

### 4.13 astats — Audio Statistics

**Purpose:** Display or extract audio statistics (RMS, peak, DC offset, etc.).

```bash
# Show statistics
ffmpeg -i input.wav -af "astats=metadata=1" -f null - 2>&1 | grep -A 20 "astats"

# Reset on new frame boundaries
ffmpeg -i input.wav -af "astats=reset=1:metadata=1" -f null -
```

**astats Output Fields:**
| Field | Description | Units |
|-------|-------------|-------|
| DC offset | Mean amplitude offset from zero | -1 to 1 |
| Min level | Minimum sample value | -1 to 1 |
| Max level | Maximum sample value | -1 to 1 |
| Peak level | Absolute maximum | dBFS |
| RMS level | Root mean square | dBFS |
| RMS peak | Peak RMS over window | dBFS |
| Crest factor | Peak/RMS ratio | ratio |
| Flat factor | Measure of tonality | dB |
| Noise floor | Estimated noise floor | dBFS |
| Entropy | Spectral entropy | bits |
| Bitcrush | Deviation from previous | % |

### 4.14 atempo — Tempo/Speed Adjustment

**Purpose:** Adjust playback tempo without changing pitch (time stretching).

```bash
# Double tempo (2x faster)
ffmpeg -i input.wav -af "atempo=2.0" output.wav

# Half tempo (2x slower)
ffmpeg -i input.wav -af "atempo=0.5" output.wav

# Tempo 1.5x with slight pitch preservation
ffmpeg -i input.wav -af "atempo=1.5" output.wav
```

**atempo Options:**
| Option | Type | Default | Range | Description |
|--------|------|---------|-------|-------------|
| tempo | float | 1.0 | 0.5–2.0 | Tempo multiplier |

**Note:** The valid range is 0.5 to 2.0. For values outside this range, chain multiple atempo filters.

### 4.15 atrim — Trim Audio

**Purpose:** Trim audio to a specific time range.

```bash
# Trim from 10s to 30s
ffmpeg -i input.wav -af "atrim=start=10:end=30" output.wav

# Trim with sample precision
ffmpeg -i input.wav -af "atrim=start_sample=100000:end_sample=500000" output.wav

# Keep only 5 seconds from position
ffmpeg -i input.wav -af "atrim=start=10:duration=5" output.wav
```

**atrim Options:**
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| start | string | 0 | Start time in seconds |
| end | string | INT_MAX | End time in seconds |
| start_sample | int64 | -1 | Start sample index |
| end_sample | int64 | -1 | End sample index |
| duration | string | 0 | Duration in seconds |
| buf_size | string | 10240 | Buffer size for look-ahead |

### 4.16 asetnsamples — Set Number of Samples

**Purpose:** Set the number of samples per output frame.

```bash
# Set output to 1024 samples per frame
ffmpeg -i input.wav -af "asetnsamples=n=1024" output.wav
```

### 4.17 asetpts — Presentation Timestamp Manipulation

**Purpose:** Modify audio PTS (presentation timestamps).

```bash
# Silence first 5 seconds
ffmpeg -i input.wav -af "asetpts=N/A" -t 5 -f null -

# Double playback speed (affects pitch)
ffmpeg -i input.wav -af "asetpts=2*PTS" output.wav
```

### 4.18 asetrate — Set Sample Rate (without resampling)

**Purpose:** Change sample rate metadata without actual resampling.

```bash
# Change sample rate tag to 48000 (no actual conversion)
ffmpeg -i input.wav -af "asetrate=48000" output.wav
```

**Warning:** Using asetrate without proper resampling can cause pitch shifts and playback speed changes. For actual resampling, use aresample instead.

### 4.19 asettb — Set Timebase

**Purpose:** Change the timebase of audio timestamps.

```bash
# Set timebase to 1/48000
ffmpeg -i input.wav -af "asettb=1/48000" output.wav
```

### 4.20 acontrast — Audio Contrast (Compression)

**Purpose:** Simple audio dynamic range compression/expansion.

```bash
ffmpeg -i input.wav -af "acontrast=contrast=1.5" output.wav
```

### 4.21 compand — Compression and Expansion

**Purpose:** Multi-band dynamic range compressor with attack, decay, and soft-knee.

```bash
# Simple compressor
ffmpeg -i input.wav -af "compand=attacks=0.3:decays=0.8:points=-50/-50|-6/-6|0/-3" output.wav

# Noise gate with compressor
ffmpeg -i input.wav -af "compand=0|0:1|-90/-90|0/0" output.wav
```

### 4.22 agate — Gate (Noise Gate)

**Purpose:** Audio gate for noise reduction.

```bash
ffmpeg -i input.wav -af "agate=threshold=-20dB:ratio=3" output.wav
```

---

## 5. AUDIO NORMALIZATION FILTERS

### 5.1 loudnorm — EBU R128 Loudness Normalization

**Purpose:** ITU-R BS.1770-4 / EBU R128 compliant loudness normalization (industry standard for broadcast).

```bash
# Two-pass loudness normalization
ffmpeg -i input.wav -af "loudnorm=I=-16:LRA=11:TP=-1.5:print_format=summary" -f null -

# First pass: measure
ffmpeg -i input.wav -af "loudnorm=I=-16:LRA=11:TP=-1.5:print_format=json" -f null - 2>&1 > log.txt

# Second pass: apply with measured values
ffmpeg -i input.wav -af "loudnorm=I=-16:LRA=11:TP=-1.5:measured_I=-20:measured_LRA=15:measured_TP=-3:measured_thresh=-40:print_format=summary" output.wav
```

**loudnorm Options:**
| Option | Type | Default | Range | Description |
|--------|------|---------|-------|-------------|
| I | float | -16 | -70 to -5 | Integrated loudness target (LUFS) |
| LRA | float | -11 | 0 to 20 | Loudness range target (LU) |
| TP | float | -1.0 | -9 to 0 | True peak limit (dBTP) |
| measured_I | float | none | -70 to -5 | Measured integrated loudness |
| measured_LRA | float | none | 0 to 20 | Measured loudness range |
| measured_TP | float | none | -9 to 0 | Measured true peak |
| measured_thresh | float | none | -70 to 0 | Measured threshold |
| offset | float | 0 | -inf to inf | Loudness offset |
| linear | bool | true | true/false | Linear (vs dynamic) normalization |
| print_format | string | none | none/json/summary | Output format |

**Common Broadcast Targets:**
| Standard | I (LUFS) | LRA (LU) | TP (dBTP) |
|----------|----------|----------|------------|
| EBU R128 | -23 | 23 (max) | -1 |
| ATSC A/85 | -24 | — | -2 |
| Netflix | -27 | — | -14 (programs), -2 (feature) |
| Apple Music | -16 | 11 | -1 |
| Spotify | -14 | — | -1 |
| YouTube | -14 | — | -1 |

### 5.2 dynaudnorm — Dynamic Audio Normalizer

**Purpose:** Windowed dynamic range normalization that preserves dynamics while evening out volume.

```bash
# Basic usage
ffmpeg -i input.wav -af "dynaudnorm" output.wav

# Aggressive normalization
ffmpeg -i input.wav -af "dynaudnorm=p=0.95:s=10:g=15" output.wav

# Gentle normalization
ffmpeg -i input.wav -af "dynaudnorm=p=0.5:s=100:g=3" output.wav
```

**dynaudnorm Options:**
| Option | Type | Default | Range | Description |
|--------|------|---------|-------|-------------|
| f | int | 500 | 1–10000 | Filter size in samples |
| g | int | 5 | 1–100 | Gain smoothing (frames) |
| p | float | 0.0 | 0–1 | Target peak (0 = auto) |
| r | float | 0.95 | 1–100 | Maximum gain factor |
| m | float | 10.0 | 1–100 | Maximum gain |
| n | bool | true | true/false | Normalize DC offset |
| c | bool | false | true/false | Correct DC offset |
| b | bool | false | true/false | Boundary mode |
| s | int | 5 | 0–1000 | Smoothing window |

---

## 6. LADSPA AND LV2 PLUGIN SUPPORT

### 6.1 LADSPA (Linux Audio Developer's Simple Plugin API)

**Purpose:** Load external LADSPA plugin effects within FFmpeg filter graphs.

```bash
# Load a LADSPA plugin
ffmpeg -i input.wav -af "ladspa=plugin_name[:label]" output.wav

# Example: Apply VLevel loudness normalizer
ffmpeg -i input.wav -af "ladspa=vlevel-ladspa:vlevel_mono" output.wav

# Example: CAPS EQ
ffmpeg -i input.wav -af "ladspa=revamp:EQ" output.wav

# List available plugins (if help is specified)
ffmpeg -i input.wav -af "ladspa=help" -f null -
```

**ladspa Options:**
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| file | string | none | LADSPA plugin .so file path |
| plugin | string | none | Plugin name |
| controls | string | none | '\|'-separated control values |

### 6.2 LV2 (LADSPA Version 2)

**Purpose:** Load LV2 plugins (more feature-rich than LADSPA).

```bash
# Load a Calf plugin
ffmpeg -i input.wav -af "lv2=http://calf.sourceforge.net/plugins/BassEnhancer:c=amount=2" output.wav

# Vinyl effect
ffmpeg -i input.wav -af "lv2=http://calf.sourceforge.net/plugins/Vinyl:c=drone=0.2|aging=0.5" output.wav

# Bit crusher
ffmpeg -i input.wav -af "lv2=http://www.openavproductions.com/artyfx#bitta:c=crush=0.3" output.wav
```

**lv2 Options:**
| Option | Type | Default | Description |
|--------|------|---------|-------------|
| plugin | string | none | Plugin URI |
| controls | string | none | '\|'-separated control values |
| sample_rate | int | 44100 | Sample rate for generator plugins |
| nb_samples | int | 1024 | Samples per frame |
| duration | string | 0 | Duration for generator plugins |

---

## 7. FILTER COMPLEX AND COMPLEX FILTERGRAPHS

### 7.1 Filter Complex Overview
`-filter_complex` enables multi-input and multi-output filter graphs that cannot be expressed with simple `-af` syntax.

```bash
# Basic filter complex
ffmpeg -i input.wav -filter_complex "[0:a]volume=0.5[out]" -map "[out]" output.wav

# Multi-input filter complex
ffmpeg -i left.wav -i right.wav -filter_complex "[0:a][1:a]amix=inputs=2[out]" \
  -map "[out]" output.wav
```

### 7.2 Multi-Output Filters

#### split (video/audio)
```bash
# Split to 4 outputs
ffmpeg -i input.wav -filter_complex "split=4[o1][o2][o3][o4]" \
  -map "[o1]" out1.wav \
  -map "[o2]" out2.wav \
  -map "[o3]" out3.wav \
  -map "[o4]" out4.wav
```

#### ashuffle (audio channel shuffle)
```bash
# Interleave channels
ffmpeg -i input.wav -af "ashuffle=shufflein=0,1,0" output.wav
```

### 7.3 Complex Examples

#### Upmix Stereo to 5.1
```bash
ffmpeg -i stereo.wav \
  -filter_complex "stereotools=mol=1:telephone=1[fln];[fln]pan=5.1|c0=0.707*FL+0.707*FC|c1=0.707*FR+0.707*FC|c2=FC|c3=0.5*LFE|c4=BL|c5=BR" \
  output_5.1.wav
```

#### Side-by-Side Mixing
```bash
# Mix left and right channels from different files
ffmpeg -i vocals.wav -i instrumental.wav \
  -filter_complex "[0:a][1:a]amix=inputs=2:duration=first:weights=1:1[out]" \
  -map "[out]" output.wav
```

#### Multi-Band Compression
```bash
# Using acrossover for multiband processing
ffmpeg -i input.wav \
  -filter_complex "acrossover=split='200|2000'[lo][mid][hi];[lo]volume=1.5[lo_vol];[mid]volume=1.2[mid_vol];[hi]volume=0.8[hi_vol];[lo_vol][mid_vol][hi_vol]amerge=inputs=3" \
  output.wav
```

---

## 8. FILTER GRAPH C API — COMPLETE REFERENCE

### 8.1 Key Data Structures

```c
#include <libavfilter/avfilter.h>
#include <libavfilter/buffersrc.h>
#include <libavfilter/buffersink.h>
#include <libavfilter/graphparser.h>

// AVFilter - Filter definition
typedef struct AVFilter {
    const char *name;              // Filter name (e.g., "volume", "aresample")
    const char *description;       // Human-readable description
    
    const AVFilterPad *inputs;     // Array of input pads (NULL-terminated)
    const AVFilterPad *outputs;    // Array of output pads (NULL-terminated)
    
    int (*init)(AVFilterContext *ctx);              // Filter initialization
    int (*init_dict)(AVFilterContext *ctx, AVDictionary **options);
    void (*uninit)(AVFilterContext *ctx);           // Cleanup
    
    int (*process_command)(AVFilterContext *ctx, const char *cmd, const char *arg,
                           char *res, int res_len, int flags);
    
    // ... internal fields
} AVFilter;

// AVFilterContext - Instance of a filter
typedef struct AVFilterContext {
    const AVFilter *filter;        // Pointer to filter definition
    char *name;                    // Instance name
    
    AVFilterPad **inputs;          // Input pads
    AVFilterPad **outputs;         // Output pads
    unsigned nb_inputs;             // Number of inputs
    unsigned nb_outputs;           // Number of outputs
    
    void *priv;                    // Private data for this instance
    
    struct AVFilterGraph *graph;   // Parent filter graph
    // ... internal fields
} AVFilterContext;

// AVFilterLink - Connection between two filters
typedef struct AVFilterLink {
    AVFilterContext *src;          // Source filter
    int srcpad;                    // Source output pad index
    AVFilterContext *dst;          // Destination filter
    int dstpad;                    // Destination input pad index
    
    enum AVMediaType type;         // Media type (AUDIO or VIDEO)
    int w, h;                      // For video: width and height
    AVSampleFormat format;          // Sample format (for audio)
    int sample_rate;               // Sample rate (for audio)
    AVChannelLayout ch_layout;     // Channel layout (for audio)
    AVRational time_base;          // Time base
    
    // ... internal fields
} AVFilterLink;

// AVFilterGraph - The complete filter graph
typedef struct AVFilterGraph {
    const AVClass *av_class;
    AVFilterContext **filters;     // All filters in graph
    unsigned nb_filters;           // Number of filters
    
    char *scale_sws_opts;          // sws options for auto-inserted scale filters
    int thread_type;               // Threading model (e.g., SLICE_THREAD)
    int nb_threads;                // Number of threads
    int thread_count;              // Current thread count
    
    // ... internal fields
} AVFilterGraph;

// AVFilterInOut - Linked list for inputs/outputs in parse functions
typedef struct AVFilterInOut {
    char *name;                    // Filter name pattern
    AVFilterContext *filter_ctx;   // Associated filter context
    int pad_idx;                   // Pad index
    
    struct AVFilterInOut *next;    // Next input/output in list
} AVFilterInOut;
```

### 8.2 Core API Functions

```c
// ─── Filter Graph Allocation ────────────────────────────────────────────────

// Allocate a new filter graph
AVFilterGraph *avfilter_graph_alloc(void);

// Free a filter graph and all its filters
void avfilter_graph_free(AVFilterGraph **graph);

// Get the AVClass for AVFilterGraph (for options)
const AVClass *avfilter_graph_get_class(AVFilterGraph *graph);

// Set maximum number of threads
void avfilter_graph_set_thread_types(AVFilterGraph *graph, int thread_type);
int avfilter_graph_set_max_threads(AVFilterGraph *graph, int max_threads);


// ─── Filter Creation ────────────────────────────────────────────────────────

// Create and add a filter to a graph
int avfilter_graph_create_filter(AVFilterContext **filt_ctx,
                                 const AVFilter *filt,
                                 const char *name,
                                 const char *args,
                                 void *opaque,
                                 AVFilterGraph *graph_ctx);

// Convenience wrapper for avfilter_graph_create_filter
AVFilterContext *avfilter_graph_alloc_filter(AVFilterGraph *graph_ctx,
                                              const AVFilter *filt,
                                              const char *name);


// ─── Filter Graph Configuration ─────────────────────────────────────────────

// Configure all filters in the graph (after all links are set up)
int avfilter_graph_config(AVFilterGraph *graphctx, void *log_ctx);

// Validate and configure the graph, checking format compatibility
int avfilter_graph_config_formats(AVFilterGraph *graph);

// Check for unused inputs/outputs
int avfilter_graph_check_validity(AVFilterGraph *graph_ctx, void *log_ctx);

// Configure all formats
int avfilter_graph_config_links(AVFilterGraph *graph_ctx);


// ─── Filter Graph Parsing ───────────────────────────────────────────────────

// Parse a filtergraph string (legacy API)
int avfilter_graph_parse(AVFilterGraph *graph,
                         const char *filters,
                         AVFilterInOut *inputs,
                         AVFilterInOut *outputs,
                         void *log_ctx);

// Parse with explicit input/output handling
int avfilter_graph_parse_ptr(AVFilterGraph *graph,
                              const char *filters,
                              AVFilterInOut **inputs,
                              AVFilterInOut **outputs,
                              void *log_ctx);

// Parse into segments (newer API, FFmpeg 6.0+)
int avfilter_graph_segment_parse(AVFilterGraph *graph,
                                  const char *graph_str,
                                  int flags,
                                  AVFilterGraphSegment **seg);

// Create filters from segments
int avfilter_graph_segment_create_filters(AVFilterGraphSegment *seg, int flags);


// ─── Buffer Source / Sink ──────────────────────────────────────────────────

// Get buffer source filter
const AVFilter *avfilter_get_by_name(const char *name);
// "abuffer" for audio, "buffer" for video

// Helper to create audio buffer source
AVFilterContext *avfilter_graph_add_buffer_source(AVFilterGraph *graph,
                                                   const char *name,
                                                   AVDictionary *options,
                                                   int *ret);

// Helper to create audio buffer sink
AVFilterContext *avfilter_graph_add_buffer_sink(AVFilterGraph *graph,
                                                 const char *name,
                                                 AVDictionary *options,
                                                 int *ret);


// ─── Filter Linking ─────────────────────────────────────────────────────────

// Link two filters together
int avfilter_link(AVFilterContext *src, int srcpad,
                  AVFilterContext *dst, int dstpad);

// Unlink two filters
void avfilter_link_free(AVFilterContext *src, int srcpad,
                        AVFilterContext *dst, int dstpad);


// ─── Query Functions ────────────────────────────────────────────────────────

// Get the format that will be used between two linked filters
AVSampleFormat avfilter_config_links(AVFilterContext *filter);


// ─── Filter Query ───────────────────────────────────────────────────────────

// Get next filter in registered list
const AVFilter *avfilter_next(const AVFilter *prev);

// Get nth input or output pad of a filter
const AVFilterPad *avfilter_pad_get_nth(const AVFilter *filter, int pad_idx, int is_output);
#define avfilter_pad_get_input(f, i) avfilter_pad_get_nth(f, i, 0)
#define avfilter_pad_get_output(f, i) avfilter_pad_get_nth(f, i, 1)

// Get pad count
int avfilter_pad_count(const AVFilterPad *pads);


// ─── Dynamic Filter Graph Operations ───────────────────────────────────────

// Send a command to a filter
int avfilter_graph_send_command(AVFilterGraph *graph,
                                const char *target,
                                const char *cmd,
                                const char *arg,
                                char *res,
                                int res_len,
                                int flags);

// Queue a command to be executed during filter processing
int avfilter_graph_queue_command(AVFilterGraph *graph,
                                  const char *target,
                                  const char *cmd,
                                  const char *arg,
                                  int flags,
                                  int lock);


// ─── Utility Functions ──────────────────────────────────────────────────────

// Dump filter graph to string
char *avfilter_graph_dump(AVFilterGraph *graph, const char *options);

// Free AVFilterInOut linked list
void avfilter_inout_free(AVFilterInOut **inout);

// Request a frame from the oldest link
int avfilter_graph_request_oldest(AVFilterGraph *graph);
```

### 8.3 Complete C API Example

```c
#include <libavformat/avformat.h>
#include <libavcodec/avcodec.h>
#include <libavfilter/avfilter.h>
#include <libavfilter/buffersrc.h>
#include <libavfilter/buffersink.h>
#include <libavutil/opt.h>
#include <libavutil/channel_layout.h>

int example_filter_audio(const char *input_file, const char *output_file) {
    AVFormatContext *fmt_ctx = NULL;
    AVCodecContext *dec_ctx = NULL, *enc_ctx = NULL;
    AVFilterGraph *filter_graph = NULL;
    AVFilterContext *buffersrc_ctx = NULL;
    AVFilterContext *buffersink_ctx = NULL;
    AVFrame *frame = NULL, *filt_frame = NULL;
    AVPacket *pkt = NULL;
    int ret = 0;

    // ─── Open input ─────────────────────────────────────────────────────────
    if ((ret = avformat_open_input(&fmt_ctx, input_file, NULL, NULL)) < 0) {
        av_log(NULL, AV_LOG_ERROR, "Cannot open input file\n");
        return ret;
    }
    avformat_find_stream_info(fmt_ctx, NULL);

    // Find audio stream
    int audio_stream_idx = av_find_best_stream(fmt_ctx, AVMEDIA_TYPE_AUDIO, -1, -1, NULL, 0);
    if (audio_stream_idx < 0) {
        av_log(NULL, AV_LOG_ERROR, "Cannot find audio stream\n");
        return AVERROR_STREAM_NOT_FOUND;
    }
    AVStream *in_stream = fmt_ctx->streams[audio_stream_idx];

    // Open decoder
    const AVCodec *dec = avcodec_find_decoder(in_stream->codecpar->codec_id);
    dec_ctx = avcodec_alloc_context3(dec);
    avcodec_parameters_to_context(dec_ctx, in_stream->codecpar);
    avcodec_open2(dec_ctx, dec, NULL);

    // ─── Build filter graph ──────────────────────────────────────────────────
    filter_graph = avfilter_graph_alloc();
    if (!filter_graph) {
        ret = AVERROR(ENOMEM);
        goto end;
    }

    // Build filter description string
    const char *filter_descr = "volume=0.5,aresample=48000,aformat=sample_fmts=fltp";

    // Create buffer source
    char args[512];
    snprintf(args, sizeof(args),
             "time_base=%d/%d:sample_rate=%d:sample_fmt=%s:channel_layout=%s",
             in_stream->time_base.num, in_stream->time_base.den,
             dec_ctx->sample_rate,
             av_get_sample_fmt_name(dec_ctx->sample_fmt),
             av_channel_layout_describe(&dec_ctx->ch_layout, (char[64]){0}));

    ret = avfilter_graph_create_filter(&buffersrc_ctx,
                                         avfilter_get_by_name("abuffer"),
                                         "in", args, NULL, filter_graph);
    if (ret < 0) {
        av_log(NULL, AV_LOG_ERROR, "Cannot create buffer source\n");
        goto end;
    }

    // Create buffer sink
    ret = avfilter_graph_create_filter(&buffersink_ctx,
                                         avfilter_get_by_name("abuffersink"),
                                         "out", NULL, NULL, filter_graph);
    if (ret < 0) {
        av_log(NULL, AV_LOG_ERROR, "Cannot create buffer sink\n");
        goto end;
    }

    // Set sink options
    static const enum AVSampleFormat sink_fmts[] = { AV_SAMPLE_FMT_FLTP, AV_SAMPLE_FMT_NONE };
    ret = av_opt_set_int_list(buffersink_ctx, "sample_fmts", sink_fmts, -1, AV_OPT_SEARCH_CHILDREN);
    if (ret < 0) {
        av_log(NULL, AV_LOG_ERROR, "Cannot set output sample format\n");
        goto end;
    }

    // Parse filter graph
    AVFilterInOut *inputs = avfilter_inout_alloc();
    AVFilterInOut *outputs = avfilter_inout_alloc();
    if (!inputs || !outputs) {
        ret = AVERROR(ENOMEM);
        goto end;
    }

    outputs->name = av_strdup("in");
    outputs->filter_ctx = buffersrc_ctx;
    outputs->pad_idx = 0;
    outputs->next = NULL;

    inputs->name = av_strdup("out");
    inputs->filter_ctx = buffersink_ctx;
    inputs->pad_idx = 0;
    inputs->next = NULL;

    ret = avfilter_graph_parse_ptr(filter_graph, filter_descr, &inputs, &outputs, NULL);
    if (ret < 0) {
        av_log(NULL, AV_LOG_ERROR, "Cannot parse filter graph\n");
        goto end;
    }

    ret = avfilter_graph_config(filter_graph, NULL);
    if (ret < 0) {
        av_log(NULL, AV_LOG_ERROR, "Cannot configure filter graph\n");
        goto end;
    }

    // ─── Allocate frames ─────────────────────────────────────────────────────
    frame = av_frame_alloc();
    filt_frame = av_frame_alloc();
    pkt = av_packet_alloc();

    // ─── Decode and filter loop ─────────────────────────────────────────────
    while (av_read_frame(fmt_ctx, pkt) >= 0) {
        if (pkt->stream_index != audio_stream_idx) {
            av_packet_unref(pkt);
            continue;
        }

        avcodec_send_packet(dec_ctx, pkt);
        while (avcodec_receive_frame(dec_ctx, frame) == 0) {
            frame->pts = frame->best_effort_timestamp;

            // Push frame to filter graph
            ret = av_buffersrc_add_frame(buffersrc_ctx, frame);
            if (ret < 0) {
                av_log(NULL, AV_LOG_ERROR, "Error while feeding the filtergraph\n");
                break;
            }

            // Pull filtered frames
            while (ret >= 0) {
                ret = av_buffersink_get_frame(buffersink_ctx, filt_frame);
                if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF)
                    break;

                // Process filt_frame (write to output, encode, etc.)
                printf("Filtered frame: %d samples, %s, %d Hz\n",
                       filt_frame->nb_samples,
                       av_get_sample_fmt_name(filt_frame->format),
                       filt_frame->sample_rate);

                av_frame_unref(filt_frame);
            }
        }
        av_packet_unref(pkt);
    }

    // Flush filter graph
    av_buffersrc_add_frame(buffersrc_ctx, NULL);
    while (av_buffersink_get_frame(buffersink_ctx, filt_frame) >= 0) {
        printf("Flushed frame: %d samples\n", filt_frame->nb_samples);
        av_frame_unref(filt_frame);
    }

end:
    av_frame_free(&frame);
    av_frame_free(&filt_frame);
    av_packet_free(&pkt);
    avcodec_free_context(&dec_ctx);
    avcodec_free_context(&enc_ctx);
    avformat_close_input(&fmt_ctx);
    avfilter_graph_free(&filter_graph);
    avfilter_inout_free(&inputs);
    avfilter_inout_free(&outputs);

    return ret < 0 ? 1 : 0;
}
```

### 8.4 Buffer Source and Sink API

```c
// ─── Buffer Source (Input to Filter Graph) ──────────────────────────────────

// Add frame to buffer source
int av_buffersrc_add_frame(AVFilterContext *buffer_src, AVFrame *frame);

// Add frame with flags
int av_buffersrc_add_frame_flags(AVFilterContext *buffer_src, AVFrame *frame, int flags);
// Flags: AV_BUFFERSRC_FLAG_NO_CHECK_FORMAT, AV_BUFFERSRC_FLAG_PUSH,
//        AV_BUFFERSRC_FLAG_KEEP_REF

// Get the number of buffered frames in buffer source
int av_buffersrc_get_nb_failed_requests(AVFilterContext *buffer_src);

// Close buffer source (flush)
int av_buffersrc_close(AVFilterContext *buffer_src, int64_t pts, uint32_t flags);

// ─── Buffer Sink (Output from Filter Graph) ─────────────────────────────────

// Get frame from buffer sink
int av_buffersink_get_frame(AVFilterContext *buffer_sink, AVFrame *frame);

// Get frame with blocking
int av_buffersink_get_frame_flags(AVFilterContext *ctx, AVFrame *frame, int flags);
// Flags: AV_BUFFERSINK_FLAG_NO_REQUEST

// Get frame count before request returns EAGAIN
int av_buffersink_poll_frame(AVFilterContext *ctx);

// Get/set buffer sink frame properties
int av_buffersink_get_channels(AVFilterContext *ctx);
void av_buffersink_set_frame_size(AVFilterContext *ctx, unsigned frame_size);

// Get specific properties of the sink
int64_t av_buffersink_get_time(AVFilterContext *ctx);
AVRational av_buffersink_get_time_base(AVFilterContext *ctx);
AVSampleFormat av_buffersink_get_format(AVFilterContext *ctx);
int av_buffersink_get_sample_rate(const AVFilterContext *ctx);
const AVChannelLayout *av_buffersink_get_ch_layout(const AVFilterContext *ctx);
int av_buffersink_get_frame_rate(const AVFilterContext *ctx);
```

---

## 9. PERFORMANCE AND THREAD SAFETY

### 9.1 Filter Graph Threading Model

```c
// Enable multi-threaded filter processing
AVFilterGraph *graph = avfilter_graph_alloc();

// Set thread type (SLICE_THREAD for slice-based parallelism)
avfilter_graph_set_thread_types(graph, AVFILTER_THREAD_SLICE);

// Set maximum thread count
avfilter_graph_set_max_threads(graph, 8);

// Or set via options:
av_opt_set(graph, "thread_type", "slice", AV_OPT_SEARCH_CHILDREN);
av_opt_set_int(graph, "nb_threads", 8, AV_OPT_SEARCH_CHILDREN);
```

### 9.2 Thread Safety by Component

| Component | Thread Safety | Notes |
|-----------|---------------|-------|
| AVFilterGraph | Yes | Thread-safe allocation and configuration |
| AVFilterContext | Varies | Check individual filter documentation |
| avfilter_graph_parse | No | Call from single thread |
| avfilter_graph_config | No | Call from single thread |
| avfilter_link | Yes | Safe for concurrent calls |
| av_buffersrc_add_frame | No | Buffer source is single-threaded per instance |
| av_buffersink_get_frame | No | Buffer sink is single-threaded per instance |

### 9.3 Buffer Management

```c
// ─── Buffer Size Considerations ─────────────────────────────────────────────

// Buffer source buffer size (in samples per channel)
av_opt_set(buffersrc_ctx, "buffer_size", "1024", AV_OPT_SEARCH_CHILDREN);

// Maximum number of frames buffered
av_opt_set(buffersrc_ctx, "max_nb_frames", "32", AV_OPT_SEARCH_CHILDREN);

// Per-filter buffering
// Some filters (aresample) maintain internal buffers for resampling.
// The swresample library buffers input when sample rates differ.
// Always call swr_convert with NULL input to flush remaining samples at EOF.
```

### 9.4 Memory Considerations

```c
// Filter graph memory allocation
// - Each filter allocates its own private data (sizeof(priv_data_size))
// - Frame buffers are allocated on demand
// - Maximum memory usage depends on:
//   1. Number of filters
//   2. Frame buffer sizes
//   3. Internal filter buffers (resampling, delay lines)

// Profile memory usage:
av_opt_set(buffersrc_ctx, "dump_input", "1", AV_OPT_SEARCH_CHILDREN);
```

---

## 10. PRACTICAL FILTER GRAPH EXAMPLES

### 10.1 Audio Normalization Pipeline
```bash
# EBU R128 loudness normalization (two-pass)
ffmpeg -i input.wav -af "loudnorm=I=-16:LRA=11:TP=-1.5:print_format=summary" -f null - 2>&1 | \
  grep -E "Input|Output|Change"

# Single-pass with defaults
ffmpeg -i input.wav -af "loudnorm=I=-16:LRA=11:TP=-1.5" output.wav
```

### 10.2 Channel Format Conversion
```bash
# Convert 5.1 to stereo with center mixing
ffmpeg -i 5.1.wav \
  -af "pan=stereo|FL=FL+0.707*FC+0.5*LFE|FR=FR+0.707*FC+0.5*LFE,volume=1.5" \
  output.wav

# Upmix mono to stereo
ffmpeg -i mono.wav -af "pan=stereo|c0=c0|c1=c0" output.wav
```

### 10.3 Resampling and Format Conversion
```bash
# Convert to 48kHz float planar
ffmpeg -i input.wav -af "aformat=sample_fmts=fltp:sample_rates=48000" output.wav

# Chain resampling and format
ffmpeg -i input.wav -af "aresample=48000,aformat=sample_fmts=s16:channel_layouts=stereo" output.wav
```

### 10.4 Multi-Track Processing
```bash
# Extract and process channels separately
ffmpeg -i 5.1.wav \
  -filter_complex "channelsplit=channel_layout=5.1[fl][fr][fc][lfe][bl][br];[fc]volume=2[fc_loud]" \
  -map "[fl]" fl.wav \
  -map "[fr]" fr.wav \
  -map "[fc_loud]" fc_loud.wav \
  -map "[lfe]" lfe.wav \
  -map "[bl]" bl.wav \
  -map "[br]" br.wav

# Mix two files with volume balance
ffmpeg -i music.wav -i vocals.wav \
  -filter_complex "[0:a]volume=0.7[music];[1:a]volume=1.3[vocals];[music][vocals]amix=inputs=2[out]" \
  -map "[out]" mixed.wav
```

### 10.5 Advanced Audio Effects
```bash
# Crossfade between two tracks
ffmpeg -i track1.wav -i track2.wav \
  -filter_complex "acrossfade=d=3:c1=tri:c2=tri" output.wav

# Echo/reverb effect
ffmpeg -i input.wav -af "aecho=0.8:0.9:500|1000:0.4|0.3,aecho=0.8:0.9:250|500:0.3|0.2" output.wav

# Noise gate
ffmpeg -i input.wav -af "agate=threshold=-40dB:ratio=4:attack=10:release=200" output.wav
```

---

## 11. ERROR HANDLING AND TROUBLESHOOTING

### 11.1 Common Error Codes

| Error | Meaning | Solution |
|-------|---------|----------|
| AVERROR(EINVAL) | Invalid filter or options | Check filter name spelling, valid option values |
| AVERROR(ENOMEM) | Out of memory | Reduce filter complexity or buffer sizes |
| AVERROR(EIO) | I/O error | Check input file validity |
| EAGAIN (via AVERROR) | No frame available yet | Continue calling filter, frames not ready |
| EOF (via AVERROR_EOF) | End of stream | No more frames to process |

### 11.2 Format Negotiation Failures

```c
// If avfilter_graph_config fails due to format issues:
// 1. Insert aformat filter to force compatible formats
const char *filter_descr = "aformat=sample_fmts=fltp:sample_rates=48000,volume=0.5";

// 2. Or use auto-inserted format conversion
avfilter_graph_set_auto_convert(graph, AVFILTER_AUTO_CONVERT_ALL);

// 3. Check format compatibility
printf("Buffer src format: %s, %d Hz, %s\n",
       av_get_sample_fmt_name(buffersrc_ctx->inputs[0]->format),
       buffersrc_ctx->inputs[0]->sample_rate,
       av_channel_layout_describe(&buffersrc_ctx->inputs[0]->ch_layout, (char[64]){0}));
```

### 11.3 Debugging Filter Graphs

```bash
# Dump filter graph to see actual configuration
ffmpeg -i input.wav -af "volume=0.5,aresample=48000" -f null - 2>&1 | grep -A 100 "Filter Graph"

# Verbose logging
ffmpeg -loglevel debug -i input.wav -af "volume=0.5" -f null - 2>&1 | grep -E "filter|AVFilter"
```

---

## 12. REFERENCE IMPLEMENTATIONS

### 12.1 Official FFmpeg Examples
| Example | Path | Description |
|---------|------|-------------|
| decode_filter_audio | doc/examples/decode_filter_audio.c | Basic audio filtering with libavfilter |
| filter_audio | doc/examples/filter_audio.c | Advanced audio filter graph |
| transcoding | doc/examples/transcoding.c | Full encode-decode-filter pipeline |

### 12.2 Filter Registration (Custom Filters)

```c
// Register a custom filter
int avfilter_register(const AVFilter *filter);

// Find registered filter
const AVFilter *avfilter_next(const AVFilter *prev);

// Get filter by name
const AVFilter *avfilter_get_by_name(const char *name);
```

---

## 13. ADVANCED FILTER GRAPH TOPICS

### 13.1 Dynamic Filter Graphs

Dynamic filter graphs allow runtime reconfiguration of filter parameters without rebuilding the entire graph:

```c
// ─── Sending Commands to Filters ──────────────────────────────────────────────

// Send a command to a specific filter
int avfilter_graph_send_command(AVFilterGraph *graph,
                              const char *target,
                              const char *cmd,
                              const char *arg,
                              char *res,
                              int res_len,
                              int flags);

// Example: Change volume dynamically
char response[64];
avfilter_graph_send_command(graph, "volume", "volume", "0.5",
                           response, sizeof(response), 0);

// Flags:
//   AVFILTER_CMD_FLAG_ONE  - Stop after first successful command
//   AVFILTER_CMD_FLAG_FAST - Only fast commands

// ─── Queue Commands ──────────────────────────────────────────────────────────

// Queue a command to be executed at a specific time
int avfilter_graph_queue_command(AVFilterGraph *graph,
                               const char *target,
                               const char *cmd,
                               const char *arg,
                               int flags,
                               int lock);

// ─── Dynamic Format Changes ─────────────────────────────────────────────────

// Some filters support runtime format changes
// Enable auto-conversion if formats change dynamically
avfilter_graph_set_auto_convert(graph, AVFILTER_AUTO_CONVERT_ALL);
```

### 13.2 Multi-Threaded Filter Processing

Filter graphs can be processed in parallel for improved performance on multi-core systems:

```c
// ─── Thread Configuration ─────────────────────────────────────────────────────

// Set thread type for the filter graph
avfilter_graph_set_thread_types(graph, AVFILTER_THREAD_SLICE);

// Supported thread types:
//   AVFILTER_THREAD_SLICE - Process filters in parallel (slice-based)
//   AVFILTER_THREAD_FRAME - Process individual frames in parallel

// Set maximum threads
avfilter_graph_set_max_threads(graph, 8);

// Or via options:
av_opt_set_int(graph, "thread_type", AVFILTER_THREAD_SLICE,
               AV_OPT_SEARCH_CHILDREN);
av_opt_set_int(graph, "nb_threads", 8, AV_OPT_SEARCH_CHILDREN);

// ─── Thread Safety Considerations ────────────────────────────────────────────

/*
Thread Safety Rules:
1. Each filter graph instance should be accessed from a single thread
2. Multiple graphs can run in parallel in different threads
3. Buffer sources and sinks are not thread-safe per-instance
4. Use separate filter graph instances for parallel processing

Best Practices:
- Create one filter graph per processing pipeline
- Process audio in chunks/frames in a single thread
- For parallel processing, create multiple filter graphs
- Monitor CPU usage to determine optimal thread count
*/

// ─── Parallel Processing Example ─────────────────────────────────────────────

AVFilterGraph *create_graph_for_thread(int thread_id) {
    AVFilterGraph *graph = avfilter_graph_alloc();
    
    // Thread-specific configuration
    av_opt_set_int(graph, "nb_threads", 4, AV_OPT_SEARCH_CHILDREN);
    
    // ... configure filters ...
    
    return graph;
}

void process_parallel(AVFrame *frames, int num_frames, int num_threads) {
    AVFilterGraph *graphs[MAX_THREADS];
    
    // Create one graph per thread
    for (int i = 0; i < num_threads; i++) {
        graphs[i] = create_graph_for_thread(i);
    }
    
    // Distribute frames across threads
    #pragma omp parallel for
    for (int i = 0; i < num_frames; i++) {
        int thread_id = omp_get_thread_num();
        process_frame(graphs[thread_id], &frames[i]);
    }
    
    // Cleanup
    for (int i = 0; i < num_threads; i++) {
        avfilter_graph_free(&graphs[i]);
    }
}
```

### 13.3 Filter Graph Memory Profiling

Understanding and optimizing memory usage in filter graphs is important for large-scale processing:

```c
// ─── Memory Profiling ────────────────────────────────────────────────────────

/*
Filter Graph Memory Usage:

Per-Filter Memory:
  - AVFilterContext->priv: Filter-specific private data
  - Filter buffers: Per-input/output buffer pools
  - Internal state: Filters like aresample maintain resampling buffers

Per-Link Memory:
  - AVFilterLink buffers: Frame buffers for passing data between filters
  - Default buffer size: 8192 samples per link
  - Configurable via filter options (e.g., abuffersink frame_size)
*/

// ─── Buffer Size Configuration ────────────────────────────────────────────────

// Configure buffer sizes for memory optimization
// In buffer sink:
av_opt_set_int(buffersink_ctx, "frame_size", 512, AV_OPT_SEARCH_CHILDREN);

// In buffer source:
av_opt_set_int(buffersrc_ctx, "buffer_size", 4096, AV_OPT_SEARCH_CHILDREN);

// ─── Memory Monitoring Example ────────────────────────────────────────────────

void print_graph_memory_stats(AVFilterGraph *graph) {
    printf("Filter Graph Memory Statistics:\n");
    printf("  Number of filters: %u\n", graph->nb_filters);
    
    size_t total_priv_size = 0;
    for (unsigned i = 0; i < graph->nb_filters; i++) {
        AVFilterContext *ctx = graph->filters[i];
        printf("  Filter %s: priv_size = %zu bytes\n",
               ctx->filter->name,
               ctx->priv ? sizeof(ctx->priv) : 0);
    }
    
    // Estimate link buffer sizes
    // Each link has buffers for input and output frames
    printf("  Per-link buffer estimate: ~%d bytes per sample × channels × frame_size\n",
           4);  // Assuming float format
}
```

---

## 14. FILTER GRAPH DEBUGGING AND TROUBLESHOOTING

### 14.1 Common Filter Graph Issues and Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| No output | Format mismatch | Add aformat filter, check sample_fmt |
| Crash on init | Invalid options | Check option names and types |
| Memory leak | Frames not unreffed | Always call av_frame_unref() |
| High CPU | Too many filters | Simplify graph, reduce buffer sizes |
| Stuttering audio | Buffer underrun | Increase buffer sizes |
| Distorted audio | Sample format mismatch | Add aformat filter |
| Wrong channels | Layout not set | Set correct channel layout |

### 14.2 Filter Graph Logging

```c
// ─── Enable Debug Logging ────────────────────────────────────────────────────

// Set log level
av_log_set_level(AV_LOG_DEBUG);

// Set custom log callback
void custom_log_callback(void *ptr, int level, const char *fmt, va_list args) {
    if (level <= av_log_get_level()) {
        // Custom logging behavior
        char buffer[1024];
        vsnprintf(buffer, sizeof(buffer), fmt, args);
        printf("[FFmpeg] %s", buffer);
    }
}

av_log_set_callback(custom_log_callback);

// ─── Dump Filter Graph ────────────────────────────────────────────────────────

// Dump graph structure to string
char *dump = avfilter_graph_dump(graph, "full");
printf("%s\n", dump);
av_free(dump);

// Format options for dump:
//   "full" - Complete graph dump
//   "none" - Minimal dump

// ─── Validate Graph ──────────────────────────────────────────────────────────

// Check graph validity
int avfilter_graph_check_validity(AVFilterGraph *graphctx, void *log_ctx);

// Check all format configurations
int avfilter_graph_config_formats(AVFilterGraph *graph);

// Check all link configurations
int avfilter_graph_config_links(AVFilterGraph *graph);
```

---

## 15. REFERENCE IMPLEMENTATIONS

### 15.1 Complete Real-Time Processing Example

```c
#include <libavformat/avformat.h>
#include <libavcodec/avcodec.h>
#include <libavfilter/avfilter.h>
#include <libavfilter/buffersrc.h>
#include <libavfilter/buffersink.h>

#define MAX_FRAME_SIZE 48000  // 1 second at 48kHz

typedef struct {
    AVFilterGraph *graph;
    AVFilterContext *buffersrc_ctx;
    AVFilterContext *buffersink_ctx;
    AVFrame *in_frame;
    AVFrame *out_frame;
} AudioProcessor;

// Initialize the audio processor
int audio_processor_init(AudioProcessor *proc, int sample_rate, int channels) {
    int ret;
    
    // Allocate context
    *proc = (AudioProcessor){0};
    
    // Create filter graph
    proc->graph = avfilter_graph_alloc();
    if (!proc->graph) return AVERROR(ENOMEM);
    
    // Filter description: volume + resample + format
    const char *filter_descr = "volume=1.0,aformat=sample_fmts=fltp:sample_rates=48000";
    
    // Create buffer source
    char args[256];
    snprintf(args, sizeof(args),
             "time_base=1/%d:sample_rate=%d:sample_fmt=s16:channel_layout=stereo",
             sample_rate, sample_rate);
    
    ret = avfilter_graph_create_filter(&proc->buffersrc_ctx,
                                       avfilter_get_by_name("abuffer"),
                                       "in", args, NULL, proc->graph);
    if (ret < 0) goto error;
    
    // Create buffer sink
    ret = avfilter_graph_create_filter(&proc->buffersink_ctx,
                                       avfilter_get_by_name("abuffersink"),
                                       "out", NULL, NULL, proc->graph);
    if (ret < 0) goto error;
    
    // Set sink options
    static const enum AVSampleFormat out_fmts[] = { AV_SAMPLE_FMT_FLTP, AV_SAMPLE_FMT_NONE };
    ret = av_opt_set_int_list(proc->buffersink_ctx, "sample_fmts", out_fmts, -1,
                               AV_OPT_SEARCH_CHILDREN);
    if (ret < 0) goto error;
    
    // Parse filter graph
    AVFilterInOut *inputs = avfilter_inout_alloc();
    AVFilterInOut *outputs = avfilter_inout_alloc();
    if (!inputs || !outputs) { ret = AVERROR(ENOMEM); goto error; }
    
    outputs->name = av_strdup("in");
    outputs->filter_ctx = proc->buffersrc_ctx;
    outputs->pad_idx = 0;
    outputs->next = NULL;
    
    inputs->name = av_strdup("out");
    inputs->filter_ctx = proc->buffersink_ctx;
    inputs->pad_idx = 0;
    inputs->next = NULL;
    
    ret = avfilter_graph_parse_ptr(proc->graph, filter_descr,
                                   &inputs, &outputs, NULL);
    if (ret < 0) goto error;
    
    ret = avfilter_graph_config(proc->graph, NULL);
    if (ret < 0) goto error;
    
    // Allocate frames
    proc->in_frame = av_frame_alloc();
    proc->out_frame = av_frame_alloc();
    if (!proc->in_frame || !proc->out_frame) {
        ret = AVERROR(ENOMEM);
        goto error;
    }
    
    avfilter_inout_free(&inputs);
    avfilter_inout_free(&outputs);
    return 0;
    
error:
    avfilter_inout_free(&inputs);
    avfilter_inout_free(&outputs);
    audio_processor_free(proc);
    return ret;
}

// Process audio data
int audio_processor_process(AudioProcessor *proc,
                          const int16_t *input, int input_samples,
                          int16_t *output, int *output_samples) {
    int ret;
    
    // Prepare input frame
    proc->in_frame->nb_samples = input_samples;
    proc->in_frame->format = AV_SAMPLE_FMT_S16;
    proc->in_frame->sample_rate = 48000;
    av_channel_layout_default(&proc->in_frame->ch_layout, 2);
    
    ret = av_frame_get_buffer(proc->in_frame, 0);
    if (ret < 0) return ret;
    
    // Copy input data to frame
    memcpy(proc->in_frame->data[0], input, input_samples * 2 * sizeof(int16_t));
    
    // Push frame to filter graph
    ret = av_buffersrc_add_frame(proc->buffersrc_ctx, proc->in_frame);
    av_frame_unref(proc->in_frame);
    if (ret < 0) return ret;
    
    // Pull frames from filter graph
    *output_samples = 0;
    while (1) {
        ret = av_buffersink_get_frame(proc->buffersink_ctx, proc->out_frame);
        if (ret == AVERROR(EAGAIN)) {
            break;  // No more frames available
        } else if (ret == AVERROR_EOF) {
            break;  // End of stream
        } else if (ret < 0) {
            return ret;  // Error
        }
        
        // Convert float planar to int16 interleaved
        float *src = (float *)proc->out_frame->data[0];
        int samples = proc->out_frame->nb_samples;
        
        for (int i = 0; i < samples && (*output_samples + i) < MAX_FRAME_SIZE; i++) {
            float s = src[i];
            if (s > 1.0f) s = 1.0f;
            if (s < -1.0f) s = -1.0f;
            output[(*output_samples + i) * 2] = (int16_t)(s * 32767.0f);
            output[(*output_samples + i) * 2 + 1] = (int16_t)(s * 32767.0f);
        }
        *output_samples += samples;
        
        av_frame_unref(proc->out_frame);
    }
    
    return 0;
}

// Free resources
void audio_processor_free(AudioProcessor *proc) {
    if (proc->in_frame) av_frame_free(&proc->in_frame);
    if (proc->out_frame) av_frame_free(&proc->out_frame);
    if (proc->graph) avfilter_graph_free(&proc->graph);
}
```

### 15.2 Filter Graph Performance Benchmark

```c
#include <time.h>

typedef struct {
    const char *name;
    double encode_time_ms;
    double decode_time_ms;
    size_t memory_bytes;
} FilterBenchmark;

void benchmark_filter_graph(const char *filter_descr,
                          int sample_rate, int channels,
                          int total_samples,
                          FilterBenchmark *result) {
    AVFilterGraph *graph = avfilter_graph_alloc();
    AVFilterContext *src, *sink;
    AVFrame *in_frame, *out_frame;
    
    clock_t start = clock();
    
    // Create simple graph
    char args[256];
    snprintf(args, sizeof(args), "%d:%d:s16:stereo", sample_rate, sample_rate);
    avfilter_graph_create_filter(&src, avfilter_get_by_name("abuffer"),
                                 "in", args, NULL, graph);
    avfilter_graph_create_filter(&sink, avfilter_get_by_name("abuffersink"),
                                 "out", NULL, NULL, graph);
    
    AVFilterInOut *inputs = avfilter_inout_alloc();
    AVFilterInOut *outputs = avfilter_inout_alloc();
    outputs->name = av_strdup("in"); outputs->filter_ctx = src;
    inputs->name = av_strdup("out"); inputs->filter_ctx = sink;
    avfilter_graph_parse_ptr(graph, filter_descr, &inputs, &outputs, NULL);
    avfilter_graph_config(graph, NULL);
    
    in_frame = av_frame_alloc();
    out_frame = av_frame_alloc();
    
    // Process frames
    int frame_size = 1024;
    int frames = total_samples / frame_size;
    for (int i = 0; i < frames; i++) {
        // ... process frame ...
    }
    
    clock_t end = clock();
    result->encode_time_ms = (double)(end - start) * 1000.0 / CLOCKS_PER_SEC;
    
    avfilter_graph_free(&graph);
    av_frame_free(&in_frame);
    av_frame_free(&out_frame);
}

// Example usage
void run_benchmarks() {
    FilterBenchmark bench;
    
    printf("Filter Graph Performance Benchmarking\n");
    printf("====================================\n\n");
    
    benchmark_filter_graph("volume=1.0", 48000, 2, 480000, &bench);
    printf("volume=1.0: %.2f ms\n", bench.encode_time_ms);
    
    benchmark_filter_graph("volume=0.5,aresample=44100", 48000, 2, 480000, &bench);
    printf("volume+aresample: %.2f ms\n", bench.encode_time_ms);
    
    benchmark_filter_graph("aformat=sample_fmts=fltp", 48000, 2, 480000, &bench);
    printf("aformat: %.2f ms\n", bench.encode_time_ms);
}
```

---

## 16. SPECIFICATIONS AND FURTHER READING

### Primary Documentation
- FFmpeg Filters Documentation: https://ffmpeg.org/ffmpeg-filters.html
- libavfilter API: https://ffmpeg.org/doxygen/nav.html?index.html#Files
- FFmpeg Filters (HTML): https://ffmpeg.org/ffmpeg-filters.html

### Technical References
- EBU R128: https://tech.ebu.ch/standards/ebu-r128
- ITU-R BS.1770: Loudness measurement standard
- LADSPA API: http://www.ladspa.org/
- LV2 Plugin Standard: https://lv2plug.in/

---

## 14. IMPLEMENTATION CHECKLIST

### Build & Configuration
- [ ] Verify FFmpeg built with `--enable-libavfilter`
- [ ] Confirm audio filters available: `ffmpeg -filters | grep " A"`
- [ ] Check LADSPA/LV2 support if needed: `ffmpeg -filters | grep -E "ladspa|lv2"`

### Filter Graph Design
- [ ] Design filter chain from source to sink
- [ ] Plan format conversions (insert aformat where needed)
- [ ] Consider buffer sizes for real-time processing
- [ ] Handle multi-output with asplit if needed

### C API Integration
- [ ] Allocate filter graph with avfilter_graph_alloc
- [ ] Create buffer source and sink filters
- [ ] Set buffer source parameters (sample rate, format, channel layout)
- [ ] Parse filter description or create filters manually
- [ ] Configure graph with avfilter_graph_config
- [ ] Implement decode → filter → encode loop

### Memory Management
- [ ] Free frames after use with av_frame_unref
- [ ] Flush filter graph at end of stream
- [ ] Free filter graph and all resources at cleanup

### Error Handling
- [ ] Handle AVERROR(EAGAIN) in buffer sink loop
- [ ] Handle AVERROR_EOF for end of stream
- [ ] Check avfilter_graph_config return value
- [ ] Validate filter descriptions before parsing

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
