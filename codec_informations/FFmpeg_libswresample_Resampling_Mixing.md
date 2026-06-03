# FFmpeg libswresample — Sample Resampling and Channel Mixing — Deep Technical Reference
> **Category:** FFmpeg API
> **File Extensions:** N/A (FFmpeg library)
> **MIME Types:** N/A (FFmpeg library)
> **Standardization Body:** FFmpeg Project
> **Primary Specification:** https://ffmpeg.org/doxygen/nav.html?index.html#Files
> **Patent Status:** N/A (API library)
> **License:** LGPL 2.1+
> **Current Version:** FFmpeg n8.1.1 (as of 2026)
> **Active Development:** Yes — active maintenance and releases

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** FFmpeg Project (initially based on libavcodec resampling code, later refactored into dedicated library)
- **Year Created:** 2004–2006 (libswresample separated from libswscale/libavcodec during FFmpeg 0.6/0.7 refactoring)
- **Original Purpose:** Provide highly optimized audio sample rate conversion, sample format conversion, and channel mixing in a standalone library. Before libswresample, these operations were embedded within libavcodec and libavfilter, making them difficult to use independently.
- **Problem with Predecessors:** Audio conversion required linking against libavcodec (which was large and codec-focused). Applications that only needed resampling or mixing (e.g., audio players, streaming servers) had to link the entire codec infrastructure. libswresample provides these operations with minimal dependencies.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| Initial integration | 2004 | Resampling functions embedded in libavcodec |
| libswresample separation | 2006–2008 | Split into dedicated library |
| AVChannelLayout API | 2015–2017 | Replaced channel reordering arrays with structured channel layout API |
| swr_alloc_set_opts2 | 2019 | Simplified context initialization API |
| SIMD optimization | 2008–2024 | SSE/AVX optimizations for conversion performance |
| Float/double planar support | 2012 | Added all planar float/double format conversions |

### 1.3 Current Adoption
- **Primary use cases today:** Audio transcoding pipelines, streaming servers, audio playback applications, video editing software, broadcast automation, real-time audio processing
- **Platforms with native support:** All platforms supported by FFmpeg (Linux, macOS, Windows, Android, iOS via FFmpeg)
- **Major services using this format:** FFmpeg CLI tool, Libav, GStreamer (via FFmpeg plugin), MPlayer, VLC (via FFmpeg), broadcast streaming servers
- **Hardware support:** SIMD acceleration (SSE, SSE2, SSE3, SSSE3, SSE4.1, SSE4.2, AVX, AVX2, NEON on ARM)
- **Status:** Ubiquitous in audio processing pipelines; essential component of FFmpeg's audio handling

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Library Purpose
libswresample performs three primary audio conversions:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    libswresample Capabilities                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. SAMPLE RATE CONVERSION (Resampling)                             │
│     Input:  48000 Hz                                                 │
│     Output: 44100 Hz  (or any other rate)                           │
│     Method: Polyphase FIR filter with SIMD optimization              │
│                                                                      │
│  2. SAMPLE FORMAT CONVERSION                                        │
│     Input:  s16 (signed 16-bit interleaved)                         │
│     Output: fltp (32-bit float planar)                              │
│     Also:   Interleaved ↔ Planar conversion                         │
│                                                                      │
│  3. CHANNEL MIXING (Rematrixing)                                    │
│     Input:  Stereo (L, R)                                           │
│     Output: Mono (M)   or   5.1 Surround                            │
│     Method: Custom mixing matrix with user-specified coefficients     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 High-Level Processing Flow
```
Input Audio Buffer
       │
       ▼
┌─────────────────────────────────────────────┐
│  SwrContext Initialization                   │
│  - Set input/output sample rate              │
│  - Set input/output sample format            │
│  - Set input/output channel layout           │
│  - Configure mixing matrix (optional)        │
└─────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────┐
│  swr_init()                                  │
│  - Validates configuration                   │
│  - Allocates internal buffers               │
│  - Builds resampling filterbank             │
│  - Sets up SIMD code paths                  │
└─────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────┐
│  swr_convert()                              │
│  - Reads input samples                      │
│  - Applies sample format conversion         │
│  - Performs channel mixing (rematrix)      │
│  - Performs sample rate conversion          │
│  - Outputs converted samples               │
└─────────────────────────────────────────────┘
       │
       ▼
Output Audio Buffer
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Max channels | 64 | Internal limit based on channel array size |
| Max sample rate | 384000 Hz | Supports high-resolution audio |
| Sample format support | All AV_SAMPLE_FMT_* | u8, s16, s32, flt, dbl, u8p, s16p, s32p, fltp, dblp |
| Resampling methods | FIR polyphase | with optional linear interpolation |
| Mixing matrix size | max_channels × max_channels | Up to 64×64 |
| Thread safety | Not thread-safe | One SwrContext per thread/stream |
| Buffer strategy | On-demand allocation | Internal ring buffer for resampling delay |

---

## 3. CORE API FUNCTIONS

### 3.1 Context Allocation and Initialization

```c
#include <libswresample/swresample.h>

// ─── Method 1: swr_alloc() with AVOptions ──────────────────────────────────

SwrContext *swr_alloc(void);

// Set options via AVOptions API
int swr_alloc_set_opts(SwrContext *s,
                        int64_t out_ch_layout, enum AVSampleFormat out_sample_fmt, int out_sample_rate,
                        int64_t in_ch_layout, enum AVSampleFormat in_sample_fmt, int in_sample_rate,
                        int log_offset, void *log_ctx);

// Example:
SwrContext *swr = swr_alloc();
swr_alloc_set_opts(swr,
    AV_CHANNEL_LAYOUT_STEREO, AV_SAMPLE_FMT_FLTP, 48000,  // Output
    AV_CHANNEL_LAYOUT_MONO, AV_SAMPLE_FMT_S16, 44100,    // Input
    0, NULL);
swr_init(swr);

// ─── Method 2: swr_alloc_set_opts2 (FFmpeg 4.4+) ──────────────────────────

SwrContext *swr_alloc_set_opts2(SwrContext **ps,
                                 const AVChannelLayout *out_ch_layout,
                                 enum AVSampleFormat out_sample_fmt,
                                 int out_sample_rate,
                                 const AVChannelLayout *in_ch_layout,
                                 enum AVSampleFormat in_sample_fmt,
                                 int in_sample_rate,
                                 int log_offset,
                                 void *log_ctx);

// Example (recommended):
SwrContext *swr = NULL;
swr_alloc_set_opts2(&swr,
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO, AV_SAMPLE_FMT_FLTP, 48000,
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_MONO, AV_SAMPLE_FMT_S16, 44100,
    0, NULL);
swr_init(swr);

// ─── Initialize ─────────────────────────────────────────────────────────────

int swr_init(SwrContext *s);

// ─── Free ───────────────────────────────────────────────────────────────────

SwrContext *swr_free(SwrContext **s);
```

### 3.2 Sample Conversion (Main Function)

```c
// ─── Primary conversion function ────────────────────────────────────────────

int swr_convert(SwrContext *s,
                 uint8_t **out,
                 int out_count,
                 const uint8_t **in,
                 int in_count);

// Parameters:
//   s          - SwrContext (must be initialized)
//   out        - Output buffer pointers (array of pointers, one per channel)
//   out_count  - Maximum number of samples per channel to output
//   in         - Input buffer pointers (array of pointers, one per channel)
//   in_count   - Number of input samples per channel to process

// Returns: Number of samples produced, or negative error code

// Returns AVERROR(EAGAIN) if output buffer is insufficient (call again when output consumed).
// Returns 0 when input is consumed and no more output is pending.
// Returns positive value when output is available.

// Example usage:
uint8_t *output_planes[8];  // Output buffer planes
int out_samples = 1024;      // Desired output samples

// Allocate output buffers (per-channel)
for (int ch = 0; ch < out_channels; ch++) {
    output_planes[ch] = av_malloc(out_samples * av_get_bytes_per_sample(out_fmt));
}

// Convert
int produced = swr_convert(swr, output_planes, out_samples,
                           input_planes, in_samples);
if (produced < 0) {
    // Handle error
}

// ─── Flush remaining output ─────────────────────────────────────────────────

// When all input is consumed, call swr_convert with NULL input to flush
// remaining samples from resampling delay buffer:
int flushed = swr_convert(swr, output_planes, out_samples, NULL, 0);

// ─── Frame-based conversion ─────────────────────────────────────────────────

int swr_convert_frame(SwrContext *swr,
                      AVFrame *output,
                      const AVFrame *input);

// This function handles frame allocation and conversion in one call.
// Output frame will have nb_samples set to the number of samples produced.
```

### 3.3 Context Configuration Options

```c
// ─── Set/get options via AVOptions ─────────────────────────────────────────

// Set integer option
int av_opt_set_int(void *obj, const char *name, int64_t val, int search_flags);

// Set double option
int av_opt_set_double(void *obj, const char *name, double val, int search_flags);

// Set string option
int av_opt_set(void *obj, const char *name, const char *val, int search_flags);

// Get option value
int av_opt_get_int(void *obj, const char *name, int search_flags, int64_t *out_val);

// Query available options
const AVOption *av_opt_next(const void *obj, const AVOption *prev);

// ─── SwrContext Options ─────────────────────────────────────────────────────

// Core parameters (can also be set via swr_alloc_set_opts):
av_opt_set_int(swr, "in_channel_count", 2, AV_OPT_SEARCH_CHILDREN);     // Input channels
av_opt_set_int(swr, "out_channel_count", 1, AV_OPT_SEARCH_CHILDREN);    // Output channels
av_opt_set_s_fmt(swr, "in_sample_fmt", AV_SAMPLE_FMT_S16, AV_OPT_SEARCH_CHILDREN);
av_opt_set_s_fmt(swr, "out_sample_fmt", AV_SAMPLE_FMT_FLTP, AV_OPT_SEARCH_CHILDREN);
av_opt_set_int(swr, "in_sample_rate", 44100, AV_OPT_SEARCH_CHILDREN);
av_opt_set_int(swr, "out_sample_rate", 48000, AV_OPT_SEARCH_CHILDREN);
av_opt_set_chlayout(swr, "in_chlayout", &in_layout, AV_OPT_SEARCH_CHILDREN);
av_opt_set_chlayout(swr, "out_chlayout", &out_layout, AV_OPT_SEARCH_CHILDREN);

// Resampling options:
av_opt_set_int(swr, "filter_size", 16, AV_OPT_SEARCH_CHILDREN);         // FIR filter size
av_opt_set_int(swr, "phase_shift", 10, AV_OPT_SEARCH_CHILDREN);          // Number of phases
av_opt_set_double(swr, "linear_interp", 0, AV_OPT_SEARCH_CHILDREN);     // Linear interp
av_opt_set_double(swr, "cutoff", 0.8, AV_OPT_SEARCH_CHILDREN);           // Cutoff (0-1)

// Rematrixing options:
av_opt_set_int(swr, "rematrix", 1, AV_OPT_SEARCH_CHILDREN);             // Enable rematrixing
av_opt_set_double(swr, "matrix_encoding", AV_MATRIX_ENCODING_NONE, AV_OPT_SEARCH_CHILDREN);
// Other matrix encodings: AV_MATRIX_ENCODING_DOLBY, AV_MATRIX_ENCODING_DPLII

// Internal processing:
av_opt_set_int(swr, "internal_sample_count", 1024, AV_OPT_SEARCH_CHILDREN);
av_opt_set_int(swr, "dither_method", SWR_DITHER_NONE, AV_OPT_SEARCH_CHILDREN);
```

### 3.4 Query and Utility Functions

```c
// ─── Get resampling delay ──────────────────────────────────────────────────

int64_t swr_get_delay(SwrContext *s, int64_t base);

// Returns the number of samples that were buffered due to resampling
// (the delay between input and output).
// 'base' is the timebase (usually 1 or sample_rate).

// Example:
int64_t delay = swr_get_delay(swr, 1000000000); // Returns in nanoseconds
printf("Resampling delay: %lld samples\n", delay);

// ─── Estimate output buffer size ───────────────────────────────────────────

int swr_get_out_samples(SwrContext *s, int in_samples);

// Returns an estimate of the number of output samples that will be produced
// when 'in_samples' input samples are converted.
// This accounts for resampling delay and any pending buffered samples.

// Example:
int estimated_out = swr_get_out_samples(swr, 1024);
printf("May produce up to %d output samples\n", estimated_out);

// ─── Drop samples ───────────────────────────────────────────────────────────

int swr_drop_output(SwrContext *s, int count);

// Drop 'count' output samples from the internal buffer.
// Useful for real-time synchronization.

// ─── Injection (for testing) ───────────────────────────────────────────────

int swr_inject_silence(SwrContext *s, int count);

// Inject 'count' silence samples into the output stream.

// ─── Convert timestamps ─────────────────────────────────────────────────────

int64_t swr_next_pts(SwrContext *s, int64_t *pts);

// Convert next input timestamp to output timestamp.
// Timebase is 1/(in_sample_rate * out_sample_rate) units.
// Returns the converted timestamp or negative error.

// ─── Maximum output samples ─────────────────────────────────────────────────

int swr_get_out_samples_buffer_size(SwrContext *s, int out_samples);

// Get the required buffer size for 'out_samples' output samples.
```

---

## 4. SAMPLE FORMAT CONVERSION

### 4.1 Supported Sample Formats

```c
// ─── All AVSampleFormat values ─────────────────────────────────────────────

enum AVSampleFormat {
    AV_SAMPLE_FMT_NONE = -1,
    
    // Non-planar (interleaved) formats
    AV_SAMPLE_FMT_U8,      // unsigned 8-bit, interleaved
    AV_SAMPLE_FMT_S16,     // signed 16-bit, interleaved
    AV_SAMPLE_FMT_S32,     // signed 32-bit, interleaved
    AV_SAMPLE_FMT_FLT,     // 32-bit float, interleaved
    AV_SAMPLE_FMT_DBL,     // 64-bit double, interleaved
    
    // Planar formats (one buffer per channel)
    AV_SAMPLE_FMT_U8P,     // unsigned 8-bit, planar
    AV_SAMPLE_FMT_S16P,    // signed 16-bit, planar
    AV_SAMPLE_FMT_S32P,    // signed 32-bit, planar
    AV_SAMPLE_FMT_FLTP,    // 32-bit float, planar
    AV_SAMPLE_FMT_DBLP,    // 64-bit double, planar
    
    AV_SAMPLE_FMT_NB        // Total number of formats
};
```

### 4.2 Format Conversion Matrix

```c
// Conversion between all format pairs is supported.
// FFmpeg automatically selects the optimal conversion path.

// Example conversions:
//   s16 → fltp: Convert signed 16-bit interleaved to float planar
//   fltp → s16: Convert float planar to signed 16-bit interleaved
//   s16p → flt: Convert signed 16-bit planar to float interleaved
//   flt → dblp: Convert float interleaved to double planar
```

### 4.3 Buffer Layouts

```
INTERLEAVED (AV_SAMPLE_FMT_S16, AV_SAMPLE_FMT_FLT, etc.):
┌─────────────────────────────────────────────────────────────────┐
│ Sample 0          │ Sample 1          │ Sample 2          │ ... │
│ [ch0][ch1][ch2]  │ [ch0][ch1][ch2]  │ [ch0][ch1][ch2]  │     │
└─────────────────────────────────────────────────────────────────┘
Memory layout: ch0_s0, ch1_s0, ch2_s0, ch0_s1, ch1_s1, ch2_s1, ...

PLANAR (AV_SAMPLE_FMT_S16P, AV_SAMPLE_FMT_FLTP, etc.):
┌─────────────────────────────────────────────────────────────────┐
│ Plane 0 (ch0):  [s0][s1][s2][s3][s4][s5][s6][s7]...          │
├─────────────────────────────────────────────────────────────────┤
│ Plane 1 (ch1):  [s0][s1][s2][s3][s4][s5][s6][s7]...          │
├─────────────────────────────────────────────────────────────────┤
│ Plane 2 (ch2):  [s0][s1][s2][s3][s4][s5][s6][s7]...          │
└─────────────────────────────────────────────────────────────────┘
Memory layout: All ch0 samples, then all ch1 samples, then all ch2 samples, ...

In swr_convert():
- For interleaved: out[0] points to the interleaved buffer, out[1..n] are ignored
- For planar: out[0], out[1], out[2], ... each point to separate channel buffers
```

### 4.4 Format Conversion Examples

```c
// ─── Example 1: s16 → fltp conversion ─────────────────────────────────────

SwrContext *swr = NULL;
swr_alloc_set_opts2(&swr,
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO, AV_SAMPLE_FMT_FLTP, 48000,
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO, AV_SAMPLE_FMT_S16, 48000,
    0, NULL);
swr_init(swr);

// Input: interleaved s16 stereo, 1024 samples
int16_t input_interleaved[1024 * 2];  // 2 channels
const uint8_t *in_planes[2] = { (uint8_t*)input_interleaved, NULL };

// Output: planar float stereo, allocate buffers
float *output_fl[2];
output_fl[0] = av_malloc(1024 * sizeof(float));
output_fl[1] = av_malloc(1024 * sizeof(float));
uint8_t *out_planes[2] = { (uint8_t*)output_fl[0], (uint8_t*)output_fl[1] };

// Convert
int out_samples = swr_convert(swr, out_planes, 1024, in_planes, 1024);
printf("Produced %d float samples\n", out_samples);

// ─── Example 2: fltp → s16 conversion ─────────────────────────────────────

SwrContext *swr2 = NULL;
swr_alloc_set_opts2(&swr2,
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO, AV_SAMPLE_FMT_S16, 48000,
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO, AV_SAMPLE_FMT_FLTP, 48000,
    0, NULL);
swr_init(swr2);

// Input: planar float stereo, 1024 samples
float *input_fl[2] = { /* ... */ };
uint8_t *in_planes2[2] = { (uint8_t*)input_fl[0], (uint8_t*)input_fl[1] };

// Output: interleaved s16 stereo
int16_t output_interleaved[1024 * 2];
uint8_t *out_planes2[2] = { (uint8_t*)output_interleaved, NULL };

int out_samples2 = swr_convert(swr2, out_planes2, 1024, in_planes2, 1024);
printf("Produced %d s16 samples\n", out_samples2);
```

---

## 5. SAMPLE RATE CONVERSION

### 5.1 Resampling Fundamentals

libswresample uses **polyphase FIR filter** resampling:

```
Resampling Process:
1. Input samples are placed into an input buffer
2. FIR filter coefficients are applied via polyphase filters
3. Output samples are generated by interpolation between filter phases
4. The polyphase structure ensures high-quality conversion

Quality vs. Performance Trade-offs:
- Larger filter_size: Better quality, more CPU
- More phase_shift: Better quality, more memory
- Linear interpolation: Faster, lower quality
- Higher cutoff: Less band-limiting, more aliasing
```

### 5.2 Resampling Parameters

```c
// ─── Filter Size ───────────────────────────────────────────────────────────

// Number of taps per phase in the polyphase filter
// Higher = better quality, more memory
// Range: 4 to 4096, default: 16
av_opt_set_int(swr, "filter_size", 32, AV_OPT_SEARCH_CHILDREN);

// ─── Phase Shift ───────────────────────────────────────────────────────────

// log2 of the number of phases
// More phases = smoother interpolation
// Range: 4 to 30, default: 10 (1024 phases)
// Higher values use more memory but can reduce aliasing
av_opt_set_int(swr, "phase_shift", 12, AV_OPT_SEARCH_CHILDREN);  // 4096 phases

// ─── Cutoff Frequency ─────────────────────────────────────────────────────

// Normalized cutoff frequency (relative to Nyquist)
// Lower values = more band-limiting, less aliasing, but reduced high frequencies
// Range: 0.0 to 1.0, default: 0.8
av_opt_set_double(swr, "cutoff", 0.9, AV_OPT_SEARCH_CHILDREN);

// ─── Linear Interpolation ──────────────────────────────────────────────────

// Use linear interpolation for faster (but lower quality) resampling
// Default: disabled (0)
av_opt_set_double(swr, "linear_interp", 1, AV_OPT_SEARCH_CHILDREN);

// ─── Dithering ─────────────────────────────────────────────────────────────

// Dithering method for noise shaping during sample rate conversion
av_opt_set_int(swr, "dither_method", SWR_DITHER_NONE, AV_OPT_SEARCH_CHILDREN);

// Options:
//   SWR_DITHER_NONE           - No dithering
//   SWR_DITHER_RECTANGULAR    - Rectangular (fastest)
//   SWR_DITHER_TRIANGULAR      - Triangular (good quality)
//   SWR_DITHER_TRIANGULAR_HIGHPASS - Triangular with HPF (best quality)
```

### 5.3 Resampling Ratio

```c
// The resampling ratio is computed as: out_sample_rate / in_sample_rate
// For integer ratios (e.g., 44100 → 48000), the filter is pre-computed at init.
// For non-integer ratios (e.g., 44100 → 48001), conversion happens at runtime.

// Common conversion ratios:
//   44100 → 48000: ratio = 48000/44100 = 1.0884...
//   48000 → 44100: ratio = 44100/48000 = 0.91875
//   44100 → 96000: ratio = 96000/44100 = 2.176...
//   48000 → 192000: ratio = 192000/48000 = 4.0

// The resampling creates a delay equal to:
//   delay = filter_size * phase_count / min(in_rate, out_rate)
```

### 5.4 Resampling Examples

```c
// ─── Example: 44.1kHz to 48kHz ────────────────────────────────────────────

SwrContext *swr = NULL;
swr_alloc_set_opts2(&swr,
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO, AV_SAMPLE_FMT_FLTP, 48000,
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO, AV_SAMPLE_FMT_FLTP, 44100,
    0, NULL);
swr_init(swr);

// Input at 44100 Hz, 1024 samples = ~23.2ms of audio
float *input[2] = { /* input data */ };
uint8_t *in_planes[2] = { (uint8_t*)input[0], (uint8_t*)input[1] };

// Output at 48000 Hz: will produce ~1116 samples (1024 * 48000/44100)
float *output[2];
av_mallocz_array(2, 1024 * 5 * sizeof(float));  // Buffer for ~5x upsampling
uint8_t *out_planes[2] = { (uint8_t*)output[0], (uint8_t*)output[1] };

// First call: might not produce all output due to buffering
int produced1 = swr_convert(swr, out_planes, 1024 * 5, in_planes, 1024);

// Flush remaining output (resampling delay buffer)
int produced2 = swr_convert(swr, out_planes, 1024 * 5, NULL, 0);

printf("Total output: %d samples\n", produced1 + produced2);

// ─── Example: 48kHz to 44.1kHz (downsample) ──────────────────────────────

SwrContext *swr_down = NULL;
swr_alloc_set_opts2(&swr_down,
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO, AV_SAMPLE_FMT_FLTP, 44100,
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO, AV_SAMPLE_FMT_FLTP, 48000,
    0, NULL);
swr_init(swr_down);

// Input at 48000 Hz, 1024 samples
// Output at 44100 Hz: will produce ~941 samples (1024 * 44100/48000)
```

---

## 6. CHANNEL MIXING (REMATRIXING)

### 6.1 Channel Layout API

```c
#include <libavutil/channel_layout.h>

// ─── Standard Channel Layouts ───────────────────────────────────────────────

// Mono
AV_CHANNEL_LAYOUT_MONO  // 1 channel: C

// Stereo
AV_CHANNEL_LAYOUT_STEREO  // 2 channels: FL, FR

// 2.1
AV_CHANNEL_LAYOUT_2POINT1  // 3 channels: FL, FR, LFE

// 5.0
AV_CHANNEL_LAYOUT_5POINT0  // 5 channels: FL, FR, FC, BL, BR
AV_CHANNEL_LAYOUT_5POINT0_BACK  // 5 channels: FL, FR, FC, BL, BR (alternative order)

// 5.1
AV_CHANNEL_LAYOUT_5POINT1  // 6 channels: FL, FR, FC, LFE, BL, BR
AV_CHANNEL_LAYOUT_5POINT1_BACK  // 6 channels: FL, FR, FC, LFE, BL, BR (alternative)

// 7.0
AV_CHANNEL_LAYOUT_7POINT0  // 8 channels
AV_CHANNEL_LAYOUT_7POINT0_WIDE  // 8 channels

// 7.1
AV_CHANNEL_LAYOUT_7POINT1  // 8 channels: FL, FR, FC, LFE, BL, BR, FLC, FRC
AV_CHANNEL_LAYOUT_7POINT1_WIDE  // 8 channels

// Surround
AV_CHANNEL_LAYOUT_SURROUND   // 3 channels: FL, FR, FC
AV_CHANNEL_LAYOUT_3POINT1    // 4 channels: FL, FR, FC, LFE

// Quad
AV_CHANNEL_LAYOUT_QUAD  // 4 channels: FL, FR, BL, BR
AV_CHANNEL_LAYOUT_QUAD_SIDE  // 4 channels: FL, FR, SL, SR

// ─── AVChannelLayout Structure ──────────────────────────────────────────────

typedef struct AVChannelLayout {
    int nb_channels;           // Number of channels
    enum AVChannelOrder order; // Channel order (native, audio, or custom)
    uint64_t u_mask;          // Bit mask of present channels
    const uint64_t *mask;     // Optional explicit mask array
    char *description;        // Human-readable description
    // ... internal fields
} AVChannelLayout;

// Channel order options:
AV_CHANNEL_ORDER_NATIVE      // Default channel order (e.g., stereo = FL, FR)
AV_CHANNEL_ORDER_CUSTOM      // Custom order specified by user
AV_CHANNEL_ORDER_UNSPEC      // Unspecified order

// ─── Working with Channel Layouts ──────────────────────────────────────────

// Copy channel layout
int av_channel_layout_copy(AVChannelLayout *dst, const AVChannelLayout *src);

// Compare channel layouts
int av_channel_layout_compare(const AVChannelLayout *a, const AVChannelLayout *b);

// Describe channel layout as string
int av_channel_layout_describe(const AVChannelLayout *ch_layout, char *buf, size_t buf_size);

// Parse channel layout from string
int av_channel_layout_from_string(AVChannelLayout *ch_layout, const char *str);
// Examples: "stereo", "5.1", "mono", "FL+FR+FC"

// Get channel index from layout
int av_channel_layout_index_from_channel(const AVChannelLayout *ch_layout, enum AVChannel channel);
int av_channel_layout_channel_from_index(const AVChannelLayout *ch_layout, int idx);

// Check if channel is present
int av_channel_layout_has_channel(const AVChannelLayout *ch_layout, enum AVChannel channel);

// Get channel name
const char *av_channel_name(enum AVChannel channel);

// ─── Default Layout ────────────────────────────────────────────────────────

void av_channel_layout_default(AVChannelLayout *ch_layout, int nb_channels);
// Sets the default layout for the given number of channels:
//   1 channel → MONO
//   2 channels → STEREO
//   3 channels → 2.1
//   4 channels → 5.0 (or QUAD if appropriate)
//   6 channels → 5.1
//   8 channels → 7.1

// ─── Query Functions ───────────────────────────────────────────────────────

int av_get_channel_layout_nb_channels(uint64_t channel_layout);
// Count channels from layout mask

const char *av_get_channel_layout_string(char *buf, int buf_size,
                                         int nb_channels, uint64_t channel_layout);
```

### 6.2 Default Mixing Matrix

When converting between channel layouts, libswresample uses a **default mixing matrix**:

```
MONO → STEREO:
  FL = M
  FR = M

STEREO → MONO:
  M = (FL + FR) / 2

5.1 → STEREO (with LFE mixed out):
  FL = FL + 0.707*FC + 0.5*LFE
  FR = FR + 0.707*FC + 0.5*LFE

5.1 → MONO:
  M = (FL + FR + FC) / 3 + 0.5*LFE

7.1 → STEREO:
  FL = FL + 0.707*FC + 0.5*BL + 0.5*BR + 0.5*LFE
  FR = FR + 0.707*FC + 0.5*BL + 0.5*BR + 0.5*LFE

Upmixing (mono → 5.1):
  FL = 0.5*M
  FR = 0.5*M
  FC = M
  LFE = 0 (silent)
  BL = 0.3*M
  BR = 0.3*M
```

### 6.3 Custom Mixing Matrix

```c
// ─── Set custom mixing matrix ──────────────────────────────────────────────

int swr_set_matrix(SwrContext *s, const double *matrix, int stride);

// Set a custom N×M mixing matrix where N = output channels, M = input channels
// matrix[i*M + j] = coefficient for mixing input channel j into output channel i

// Example: Create stereo to mono mixer with custom weights
double matrix[1 * 2];  // 1 output, 2 inputs
matrix[0] = 0.6;      // FL → M: 0.6
matrix[1] = 0.4;      // FR → M: 0.4

SwrContext *swr = NULL;
swr_alloc_set_opts2(&swr,
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_MONO, AV_SAMPLE_FMT_FLTP, 48000,
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO, AV_SAMPLE_FMT_FLTP, 48000,
    0, NULL);
swr_init(swr);

swr_set_matrix(swr, matrix, 1);  // stride = 1 output channel

// ─── Get current mixing matrix ────────────────────────────────────────────

int swr_get_matrix(SwrContext *s, double *matrix, int stride);

// ─── Build matrix helper ───────────────────────────────────────────────────

int swr_build_matrix(uint64_t in_layout, uint64_t out_layout,
                     double *matrix, int stride,
                     double center_mix_level, double surround_mix_level,
                     double lfe_mix_level, double rematrix_volume,
                     enum AVMatrixEncoding matrix_encoding, void *log_ctx);

// Build standard downmix/upmix matrices
// center_mix_level: Mix level for center channel in stereo downmix (default 0.707)
// surround_mix_level: Mix level for surround in stereo downmix (default 0.5)
// lfe_mix_level: Mix level for LFE in stereo downmix (default 0.0)
// matrix_encoding: Dolby, Pro Logic II, or none

// ─── Enable/disable rematrixing ───────────────────────────────────────────

av_opt_set_int(swr, "rematrix", 1, AV_OPT_SEARCH_CHILDREN);  // Enable (default)
av_opt_set_int(swr, "rematrix", 0, AV_OPT_SEARCH_CHILDREN);  // Disable

// ─── Set volume for rematrixing ──────────────────────────────────────────

av_opt_set_double(swr, "rematrix_volume", 1.0, AV_OPT_SEARCH_CHILDREN);
// Scale all mixing coefficients by this value
```

### 6.4 Channel Mixing Examples

```c
// ─── Example 1: 5.1 to Stereo Downmix ─────────────────────────────────────

SwrContext *swr = NULL;
swr_alloc_set_opts2(&swr,
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO, AV_SAMPLE_FMT_FLTP, 48000,
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_5POINT1, AV_SAMPLE_FMT_FLTP, 48000,
    0, NULL);
swr_init(swr);

// Default downmix coefficients:
// FL = FL + 0.707*FC + 0.5*BL + 0.5*BR + 0.5*LFE
// FR = FR + 0.707*FC + 0.5*BL + 0.5*BR + 0.5*LFE

// ─── Example 2: Custom 5.1 to Stereo Downmix ──────────────────────────────

SwrContext *swr2 = NULL;
swr_alloc_set_opts2(&swr2,
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO, AV_SAMPLE_FMT_FLTP, 48000,
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_5POINT1, AV_SAMPLE_FMT_FLTP, 48000,
    0, NULL);
swr_init(swr2);

// Custom matrix: 2 outputs × 6 inputs
double matrix[2 * 6] = {
    // FL           FR            FC             LFE           BL           BR
    1.0,           0.0,          0.5,          0.0,         0.3,         0.3,   // Out FL
    0.0,           1.0,          0.5,          0.0,         0.3,         0.3    // Out FR
};

swr_set_matrix(swr2, matrix, 2);

// ─── Example 3: Mono to 5.1 Upmix ────────────────────────────────────────

SwrContext *swr3 = NULL;
swr_alloc_set_opts2(&swr3,
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_5POINT1, AV_SAMPLE_FMT_FLTP, 48000,
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_MONO, AV_SAMPLE_FMT_FLTP, 48000,
    0, NULL);
swr_init(swr3);

// Custom matrix: 6 outputs × 1 input
double matrix[6 * 1] = {
    0.5,   // FL
    0.5,   // FR
    1.0,   // FC (center)
    0.0,   // LFE
    0.3,   // BL
    0.3    // BR
};

swr_set_matrix(swr3, matrix, 6);

// ─── Example 4: Stereo Swap (L↔R) ─────────────────────────────────────────

SwrContext *swr4 = NULL;
swr_alloc_set_opts2(&swr4,
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO, AV_SAMPLE_FMT_FLTP, 48000,
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO, AV_SAMPLE_FMT_FLTP, 48000,
    0, NULL);
swr_init(swr4);

// Swap left and right
double matrix[2 * 2] = {
    0.0, 1.0,  // FL gets input from FR
    1.0, 0.0   // FR gets input from FL
};

swr_set_matrix(swr4, matrix, 2);

// ─── Example 5: Extract specific channel ───────────────────────────────────

SwrContext *swr5 = NULL;
swr_alloc_set_opts2(&swr5,
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_MONO, AV_SAMPLE_FMT_FLTP, 48000,
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_5POINT1, AV_SAMPLE_FMT_FLTP, 48000,
    0, NULL);
swr_init(swr5);

// Extract front center only
double matrix[1 * 6] = {
    0.0,  // FL
    0.0,  // FR
    1.0,  // FC
    0.0,  // LFE
    0.0,  // BL
    0.0   // BR
};

swr_set_matrix(swr5, matrix, 1);
```

---

## 7. FRAME-BASED CONVERSION API

### 7.1 swr_convert_frame

```c
// ─── Frame-based conversion ────────────────────────────────────────────────

int swr_convert_frame(SwrContext *swr,
                      AVFrame *output,
                      const AVFrame *input);

// Parameters:
//   swr     - SwrContext (must be initialized)
//   output  - Output frame (will be allocated if needed)
//   input   - Input frame

// Returns: 0 on success, negative error code
//   AVERROR(EAGAIN): Output not available yet, call again
//   AVERROR_EOF: Input consumed, no more output
//   AVERROR_OUTPUT_CHANGED: Output frame format changed
//   AVERROR_INPUT_CHANGED: Input frame format changed

// Example:
AVFrame *in_frame = av_frame_alloc();
AVFrame *out_frame = av_frame_alloc();

// Configure input frame
in_frame->format = AV_SAMPLE_FMT_S16P;
in_frame->sample_rate = 44100;
av_channel_layout_copy(&in_frame->ch_layout, &(AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO);
in_frame->nb_samples = 1024;
av_frame_get_buffer(in_frame, 0);

// Fill input frame with data
// ...

// Convert
int ret = swr_convert_frame(swr, out_frame, in_frame);
if (ret < 0) {
    // Handle error
}

// Use out_frame
printf("Output: %d samples, %s, %d Hz\n",
       out_frame->nb_samples,
       av_get_sample_fmt_name(out_frame->format),
       out_frame->sample_rate);

// Clean up
av_frame_free(&in_frame);
av_frame_free(&out_frame);
```

### 7.2 Frame vs. Buffer API Comparison

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Buffer API vs. Frame API Comparison                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  BUFFER API (swr_convert)                 │  FRAME API (swr_convert_frame)│
│  ─────────────────────────────────────────  │  ────────────────────────────│
│  • Manual buffer management                  │  • Automatic allocation       │
│  • More control over memory                 │  • Simpler code               │
│  • Must track sample counts manually         │  • Frame metadata preserved   │
│  • Can reuse buffers for efficiency         │  • PTS/timestamp handling     │
│  • Lower level, more flexibility             │  • Higher level abstraction   │
│                                                                             │
│  Best for:                               │  Best for:                      │
│  • Real-time streaming                     │  • Batch processing            │
│  • Memory-constrained systems              │  • When frames have metadata   │
│  • Custom buffer management                 │  • Quick prototyping           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 8. ERROR HANDLING

### 8.1 Error Codes

```c
#include <libavutil/error.h>

// ─── Common Error Codes ────────────────────────────────────────────────────

AVERROR_EOF           // End of file
AVERROR(EINVAL)       // Invalid argument
AVERROR(ENOMEM)       // Out of memory
AVERROR_UNKNOWN        // Unknown error

// From swr_convert:
AVERROR(EAGAIN)       // Output not available, call again
AVERROR(EINVAL)       // Context not initialized, invalid format, etc.

// From swr_convert_frame:
AVERROR_OUTPUT_CHANGED   // Output format changed, reconfigure needed
AVERROR_INPUT_CHANGED    // Input format changed, reconfigure needed

// ─── Error Handling Example ────────────────────────────────────────────────

int ret = swr_convert(swr, out_planes, out_count, in_planes, in_count);
if (ret < 0) {
    char errbuf[128];
    av_strerror(ret, errbuf, sizeof(errbuf));
    
    if (ret == AVERROR(EAGAIN)) {
        // Need to call again after consuming output
        printf("Need to consume output before providing more input\n");
    } else {
        fprintf(stderr, "swr_convert error: %s\n", errbuf);
    }
} else {
    int produced = ret;
    printf("Produced %d samples\n", produced);
}
```

### 8.2 Validation at Initialization

```c
// ─── Validate context after init ──────────────────────────────────────────

SwrContext *swr = swr_alloc();
swr_alloc_set_opts2(&swr,
    &out_layout, out_fmt, out_rate,
    &in_layout, in_fmt, in_rate,
    0, NULL);

int ret = swr_init(swr);
if (ret < 0) {
    // Check what went wrong
    if (ret == AVERROR(EINVAL)) {
        fprintf(stderr, "Invalid configuration\n");
    }
    
    // Clean up
    swr_free(&swr);
    return ret;
}

// ─── Check format compatibility ────────────────────────────────────────────

// Get actual configured formats (may differ from requested if auto-conversion occurred)
AVSampleFormat actual_in_fmt = av_opt_get_sample_fmt(swr, "in_sample_fmt");
AVSampleFormat actual_out_fmt = av_opt_get_sample_fmt(swr, "out_sample_fmt");
printf("Actual in format: %s\n", av_get_sample_fmt_name(actual_in_fmt));
printf("Actual out format: %s\n", av_get_sample_fmt_name(actual_out_fmt));
```

---

## 9. PERFORMANCE CONSIDERATIONS

### 9.1 SIMD Acceleration

libswresample automatically uses SIMD instructions when available:

```
SIMD Optimizations:
• SSE (128-bit): 4 × 32-bit floats per operation
• SSE2 (128-bit): 2 × 64-bit doubles, or 8 × 16-bit integers
• SSE3: Additional horizontal operations
• SSSE3: Shuffle operations
• SSE4.1/SSE4.2: Dot products, rounding
• AVX (256-bit): 8 × 32-bit floats per operation
• AVX2 (256-bit): 8 × 32-bit floats with FMA
• NEON (ARM): 4 × 32-bit floats or 8 × 16-bit integers

FFmpeg selects the best code path at runtime based on CPU capabilities.
```

### 9.2 Buffer Management

```c
// ─── Internal buffer allocation ─────────────────────────────────────────────

// SwrContext maintains internal buffers for:
// 1. Input samples (for resampling delay)
// 2. Intermediate conversion results
// 3. Output samples (if buffer is full)

// The internal buffer size depends on:
// - Sample rate ratio (larger for up/down conversion)
// - Filter size
// - Number of channels

// ─── Reducing latency ─────────────────────────────────────────────────────

// For low-latency applications, you can reduce internal buffering:
// However, this may cause quality degradation for large rate conversions

av_opt_set_int(swr, "filter_size", 4, AV_OPT_SEARCH_CHILDREN);   // Minimal filter
av_opt_set_int(swr, "phase_shift", 4, AV_OPT_SEARCH_CHILDREN);    // Fewer phases

// ─── Getting buffer requirements ──────────────────────────────────────────

int64_t delay_samples = swr_get_delay(swr, 1);
// Returns delay in samples at output sample rate
```

### 9.3 Performance Tips

```c
// ─── Best practices for performance ────────────────────────────────────────

// 1. Reuse SwrContext for repeated conversions
// Creating a new context is expensive; reuse when possible.

SwrContext *swr = NULL;
swr_alloc_set_opts2(&swr, &out, out_fmt, out_rate, &in, in_fmt, in_rate, 0, NULL);
swr_init(swr);

for (int i = 0; i < many_files; i++) {
    // ... convert each file with the same context
}

swr_free(&swr);

// 2. Use planar formats for internal processing
// Planar formats are more efficient for channel-wise operations

// 3. Batch multiple samples
// Converting larger batches reduces per-call overhead

// 4. Use frame API for cleaner code when performance is not critical
// swr_convert_frame has slightly more overhead but is easier to use

// 5. Match input/output sample rates when possible
// No-resampling conversion is much faster

// 6. Use the native format for the operation
// For rematrixing without format change, keep the same format
swr_alloc_set_opts2(&swr, &out_layout, out_fmt, out_rate,
                     &in_layout, out_fmt, out_rate, 0, NULL);  // Same format

// 7. Consider format conversion separately from resampling
// Sometimes two SwrContexts are faster than one that does everything
```

---

## 10. COMPLETE C API REFERENCE

### 10.1 Header Files

```c
#include <libswresample/swresample.h>    // Main header
#include <libavutil/channel_layout.h>    // AVChannelLayout
#include <libavutil/opt.h>              // AVOptions API
#include <libavutil/samplefmt.h>        // AVSampleFormat utilities
```

### 10.2 Complete Function List

```c
// ─── Context Management ─────────────────────────────────────────────────────

SwrContext *swr_alloc(void);
SwrContext *swr_alloc_set_opts(SwrContext *s,
                               int64_t out_ch_layout, enum AVSampleFormat out_sample_fmt,
                               int out_sample_rate,
                               int64_t in_ch_layout, enum AVSampleFormat in_sample_fmt,
                               int in_sample_rate,
                               int log_offset, void *log_ctx);
int swr_alloc_set_opts2(SwrContext **ps,
                         const AVChannelLayout *out_ch_layout,
                         enum AVSampleFormat out_sample_fmt,
                         int out_sample_rate,
                         const AVChannelLayout *in_ch_layout,
                         enum AVSampleFormat in_sample_fmt,
                         int in_sample_rate,
                         int log_offset, void *log_ctx);
int swr_init(SwrContext *s);
void swr_free(SwrContext **s);
SwrContext *swr_allocged(void);

// ─── Sample Conversion ─────────────────────────────────────────────────────

int swr_convert(SwrContext *s, uint8_t **out, int out_count,
                const uint8_t **in, int in_count);
int swr_convert_frame(SwrContext *swr, AVFrame *output, const AVFrame *input);

// ─── Utility Functions ─────────────────────────────────────────────────────

int64_t swr_get_delay(SwrContext *s, int64_t base);
int swr_get_out_samples(SwrContext *s, int in_samples);
int swr_drop_output(SwrContext *s, int count);
int swr_inject_silence(SwrContext *s, int count);
int64_t swr_next_pts(SwrContext *s, int64_t *pts);

// ─── Mixing Matrix ─────────────────────────────────────────────────────────

int swr_set_matrix(SwrContext *s, const double *matrix, int stride);
int swr_get_matrix(SwrContext *s, double *matrix, int stride);
int swr_build_matrix(uint64_t in_layout, uint64_t out_layout,
                     double *matrix, int stride,
                     double center_mix_level, double surround_mix_level,
                     double lfe_mix_level, double rematrix_volume,
                     enum AVMatrixEncoding matrix_encoding, void *log_ctx);

// ─── Configuration Query ───────────────────────────────────────────────────

int swr_config_init(SwrContext *s);
int swr_pending_count(SwrContext *s);
int swr_pending_swr_delay(SwrContext *s);

// ─── Version Info ─────────────────────────────────────────────────────────

unsigned swresample_version(void);  // Returns version like avutil_version
const char *swresample_configuration(void);  // Build configuration
const char *swresample_license(void);        // License string
```

### 10.3 Constants and Enums

```c
// ─── Dither Methods ────────────────────────────────────────────────────────

enum SwrDitherOptions {
    SWR_DITHER_NONE = 0,
    SWR_DITHER_RECTANGULAR,
    SWR_DITHER_TRIANGULAR,
    SWR_DITHER_TRIANGULAR_HIGHPASS,
};

// ─── Matrix Encoding ───────────────────────────────────────────────────────

enum AVMatrixEncoding {
    AV_MATRIX_ENCODING_NONE,
    AV_MATRIX_ENCODING_DOLBY,
    AV_MATRIX_ENCODING_DPLII,
    AV_MATRIX_ENCODING_DPLIIX,
    AV_MATRIX_ENCODING_DPLIIZ,
    AV_MATRIX_ENCODING_NB,
};

// ─── Channel Names ─────────────────────────────────────────────────────────

enum AVChannel {
    AV_CHANNEL_NONE = -1,
    AV_CHANNEL_FRONT_LEFT,
    AV_CHANNEL_FRONT_RIGHT,
    AV_CHANNEL_FRONT_CENTER,
    AV_CHANNEL_LOW_FREQUENCY,
    AV_CHANNEL_BACK_LEFT,
    AV_CHANNEL_BACK_RIGHT,
    AV_CHANNEL_FRONT_LEFT_OF_CENTER,
    AV_CHANNEL_FRONT_RIGHT_OF_CENTER,
    AV_CHANNEL_BACK_CENTER,
    AV_CHANNEL_SIDE_LEFT,
    AV_CHANNEL_SIDE_RIGHT,
    AV_CHANNEL_TOP_CENTER,
    AV_CHANNEL_TOP_FRONT_LEFT,
    AV_CHANNEL_TOP_FRONT_CENTER,
    AV_CHANNEL_TOP_FRONT_RIGHT,
    AV_CHANNEL_TOP_BACK_LEFT,
    AV_CHANNEL_TOP_BACK_CENTER,
    AV_CHANNEL_TOP_BACK_RIGHT,
    AV_CHANNEL_STEREO_LEFT,
    AV_CHANNEL_STEREO_RIGHT,
    AV_CHANNEL_WIDE_LEFT,
    AV_CHANNEL_WIDE_RIGHT,
    AV_CHANNEL_SURROUND_DIRECT_LEFT,
    AV_CHANNEL_SURROUND_DIRECT_RIGHT,
    AV_CHANNEL_LOW_FREQUENCY_2,
    AV_CHANNEL_TOP_SIDE_LEFT,
    AV_CHANNEL_TOP_SIDE_RIGHT,
    AV_CHANNEL_BOTTOM_FRONT_CENTER,
    AV_CHANNEL_BOTTOM_FRONT_LEFT,
    AV_CHANNEL_BOTTOM_FRONT_RIGHT,
    AV_CHANNEL_NB,  // Total number of channels
};
```

---

## 11. PRACTICAL EXAMPLES

### 11.1 Complete Transcoding Example

```c
#include <libavformat/avformat.h>
#include <libavcodec/avcodec.h>
#include <libswresample/swresample.h>

int transcode_audio(const char *input_file, const char *output_file,
                   int out_sample_rate, enum AVSampleFormat out_sample_fmt,
                   uint64_t out_ch_layout) {
    AVFormatContext *ifmt_ctx = NULL, *ofmt_ctx = NULL;
    AVCodecContext *dec_ctx = NULL, *enc_ctx = NULL;
    SwrContext *swr = NULL;
    AVPacket *pkt = NULL;
    AVFrame *frame = NULL;
    int ret = 0;

    // ─── Open input ────────────────────────────────────────────────────────
    avformat_open_input(&ifmt_ctx, input_file, NULL, NULL);
    avformat_find_stream_info(ifmt_ctx, NULL);

    int audio_stream_idx = av_find_best_stream(ifmt_ctx, AVMEDIA_TYPE_AUDIO, -1, -1, NULL, 0);
    AVStream *in_stream = ifmt_ctx->streams[audio_stream_idx];

    const AVCodec *dec = avcodec_find_decoder(in_stream->codecpar->codec_id);
    dec_ctx = avcodec_alloc_context3(dec);
    avcodec_parameters_to_context(dec_ctx, in_stream->codecpar);
    avcodec_open2(dec_ctx, dec, NULL);

    // ─── Setup resampling ──────────────────────────────────────────────────
    swr = swr_alloc();
    swr_alloc_set_opts2(&swr,
        &(AVChannelLayout){0}, out_sample_fmt, out_sample_rate,
        &dec_ctx->ch_layout, dec_ctx->sample_fmt, dec_ctx->sample_rate,
        0, NULL);
    swr_init(swr);

    // ─── Open output ───────────────────────────────────────────────────────
    avformat_alloc_output_context2(&ofmt_ctx, NULL, NULL, output_file);
    AVStream *out_stream = avformat_new_stream(ofmt_ctx, NULL);

    const AVCodec *enc = avcodec_find_encoder(out_fmt_ctx->oformat->audio_codec);
    enc_ctx = avcodec_alloc_context3(enc);
    enc_ctx->sample_fmt = out_sample_fmt;
    enc_ctx->sample_rate = out_sample_rate;
    av_channel_layout_copy(&enc_ctx->ch_layout, &(AVChannelLayout){0});
    avcodec_open2(enc_ctx, enc, NULL);
    avcodec_parameters_from_context(out_stream->codecpar, enc_ctx);

    avformat_write_header(ofmt_ctx, NULL);

    // ─── Main loop ─────────────────────────────────────────────────────────
    pkt = av_packet_alloc();
    frame = av_frame_alloc();
    AVFrame *resampled_frame = av_frame_alloc();

    while (av_read_frame(ifmt_ctx, pkt) >= 0) {
        if (pkt->stream_index != audio_stream_idx) continue;

        avcodec_send_packet(dec_ctx, pkt);
        while (avcodec_receive_frame(dec_ctx, frame) >= 0) {
            // Resample
            uint8_t *resampled_data[8];
            int resampled_linesize;
            av_image_alloc(resampled_data, &resampled_linesize,
                          1, frame->nb_samples * 2,  // Allow extra for upsample
                          out_sample_fmt, 1);

            int resampled_samples = swr_convert(swr, resampled_data,
                                               frame->nb_samples * 2,
                                               (const uint8_t**)frame->data,
                                               frame->nb_samples);

            // Encode resampled
            avcodec_send_frame(enc_ctx, frame);  // In real code: resampled frame
            // ... encoding and muxing ...
        }
    }

    // Flush
    avcodec_send_packet(dec_ctx, NULL);
    while (avcodec_receive_frame(dec_ctx, frame) >= 0) {
        // Resample and encode remaining
    }

    av_write_trailer(ofmt_ctx);

    // ─── Cleanup ───────────────────────────────────────────────────────────
    swr_free(&swr);
    av_frame_free(&frame);
    av_frame_free(&resampled_frame);
    av_packet_free(&pkt);
    avcodec_free_context(&dec_ctx);
    avcodec_free_context(&enc_ctx);
    avformat_close_input(&ifmt_ctx);
    avformat_free_context(ofmt_ctx);

    return ret;
}
```

### 11.2 Multi-Channel Processing Example

```c
// Process 5.1 audio, extract center channel, convert to mono
void process_center_channel(const char *input, const char *output) {
    AVFormatContext *fmt_ctx = avformat_alloc_context();
    avformat_open_input(&fmt_ctx, input, NULL, NULL);
    avformat_find_stream_info(fmt_ctx, NULL);

    int audio_idx = av_find_best_stream(fmt_ctx, AVMEDIA_TYPE_AUDIO, -1, -1, NULL, 0);
    AVCodecContext *dec_ctx = avcodec_alloc_context3(
        avcodec_find_decoder(fmt_ctx->streams[audio_idx]->codecpar->codec_id));
    avcodec_parameters_to_context(dec_ctx, fmt_ctx->streams[audio_idx]->codecpar);
    avcodec_open2(dec_ctx, avcodec_find_decoder(dec_ctx->codec_id), NULL);

    // Resampler: 5.1 → Mono, S16 → FLTP
    SwrContext *swr = NULL;
    AVChannelLayout mono = AV_CHANNEL_LAYOUT_MONO;
    swr_alloc_set_opts2(&swr,
        &mono, AV_SAMPLE_FMT_FLTP, dec_ctx->sample_rate,
        &dec_ctx->ch_layout, dec_ctx->sample_fmt, dec_ctx->sample_rate,
        0, NULL);
    swr_init(swr);

    // Process
    AVFrame *frame = av_frame_alloc();
    AVPacket *pkt = av_packet_alloc();

    while (av_read_frame(fmt_ctx, pkt) >= 0) {
        if (pkt->stream_index != audio_idx) continue;

        avcodec_send_packet(dec_ctx, pkt);
        while (avcodec_receive_frame(dec_ctx, frame) >= 0) {
            float *mono_data = av_malloc(frame->nb_samples * sizeof(float));

            uint8_t *in_planes[8];
            for (int ch = 0; ch < dec_ctx->ch_layout.nb_channels; ch++)
                in_planes[ch] = frame->data[ch];
            uint8_t *out_planes[1] = { (uint8_t*)mono_data };

            int out_samples = swr_convert(swr, out_planes, frame->nb_samples,
                                         (const uint8_t**)in_planes, frame->nb_samples);

            // mono_data now contains the center channel extracted as mono
            process_mono_samples(mono_data, out_samples);

            av_free(mono_data);
        }
    }

    swr_free(&swr);
    av_frame_free(&frame);
    av_packet_free(&pkt);
    avcodec_free_context(&dec_ctx);
    avformat_close_input(&fmt_ctx);
}
```

---

## 12. PERFORMANCE OPTIMIZATION TECHNIQUES

### 12.1 SIMD Vectorization

libswresample automatically uses SIMD instructions when available. Here's how it works internally:

```
SIMD Optimization Overview:
─────────────────────────────────────────────────────────────────────────────

1. Sample Format Conversion (S16 ↔ FLT):
   • SSE: 4 samples per iteration (4 × 32-bit floats)
   • AVX: 8 samples per iteration (8 × 32-bit floats)
   • NEON: 4 samples per iteration (4 × 32-bit floats)

2. Channel Mixing (Rematrix):
   • Matrix multiplication with SIMD
   • Horizontal additions for summing
   • FMA (Fused Multiply-Add) when available

3. Resampling (Polyphase Filter):
   • Convolution with SIMD acceleration
   • Polyphase filter bank with vectorized coefficients
   • Interpolation with SIMD

4. Auto-Detection:
   FFmpeg runtime detects CPU capabilities and selects optimal code path
```

### 12.2 Buffer Size Optimization

```
Buffer Size Optimization:
─────────────────────────────────────────────────────────────────────────────

1. Input Buffer Size:
   • Large buffers reduce function call overhead
   • Too large = increased latency
   • Recommended: 1024-8192 samples per call

2. Output Buffer Size:
   • For upsample (44100 → 48000): need ~9% extra space
   • For downsample: need proportionally less

3. Internal Resampling Buffer:
   • Filter size affects internal buffer requirements
   • Smaller filter = less latency but lower quality
   • Larger filter = more latency but better quality
```

### 12.3 Benchmarking libswresample

```c
#include <time.h>
#include <stdio.h>

typedef struct {
    const char *description;
    double total_time_ms;
    double samples_per_sec;
    double realtime_factor;
    int total_samples;
} BenchmarkResult;

void benchmark_swr_conversion(int in_rate, int out_rate,
                             AVSampleFormat in_fmt, AVSampleFormat out_fmt,
                             int channels, int total_samples,
                             BenchmarkResult *result) {
    SwrContext *swr = NULL;
    
    AVChannelLayout layout;
    av_channel_layout_default(&layout, channels);
    
    swr_alloc_set_opts2(&swr,
        &layout, out_fmt, out_rate,
        &layout, in_fmt, in_rate,
        0, NULL);
    swr_init(swr);
    
    float *in_data = av_malloc(total_samples * channels * sizeof(float));
    for (int i = 0; i < total_samples * channels; i++) {
        in_data[i] = sinf(2.0 * M_PI * 440.0 * i / in_rate);
    }
    
    int max_out_samples = total_samples * out_rate / in_rate + 1024;
    float *out_data = av_malloc(max_out_samples * channels * sizeof(float));
    
    clock_t start = clock();
    
    const uint8_t *in_planes[1] = { (uint8_t*)in_data };
    uint8_t *out_planes[1] = { (uint8_t*)out_data };
    
    int out_samples_total = swr_convert(swr, out_planes, max_out_samples,
                                       in_planes, total_samples);
    
    clock_t end = clock();
    
    double elapsed = (double)(end - start) / CLOCKS_PER_SEC;
    double audio_duration = (double)total_samples / in_rate;
    
    result->description = "swr_convert";
    result->total_time_ms = elapsed * 1000.0;
    result->samples_per_sec = total_samples / elapsed;
    result->realtime_factor = audio_duration / elapsed;
    result->total_samples = total_samples;
    
    av_free(in_data);
    av_free(out_data);
    swr_free(&swr);
}
```

---

## 13. COMPARISON WITH LIBAVRESAMPLE (LEGACY)

FFmpeg previously had a library called `libavresample` (from the Libav fork). It has been deprecated in favor of libswresample:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   libavresample vs. libswresample                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  libavresample:                                                            │
│  • Created during FFmpeg/Libav fork (2011)                                  │
│  • Similar API to libswresample                                            │
│  • No longer maintained                                                    │
│  • Removed from FFmpeg 4.0+                                                │
│                                                                             │
│  libswresample:                                                            │
│  • FFmpeg's official audio conversion library                              │
│  • Active development                                                      │
│  • Better SIMD optimization                                               │
│  • Modern AVChannelLayout API                                              │
│  • Recommended for all new code                                           │
│                                                                             │
│  Migration from libavresample:                                            │
│  • Most function names are identical (swr_* vs. avr_*)                     │
│  • SwrContext replaces AVAudioResampleContext                              │
│  • Key differences: AVChannelLayout API, swr_alloc_set_opts2              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 13. SPECIFICATIONS AND FURTHER READING

### Primary Documentation
- libswresample API: https://ffmpeg.org/doxygen/trunk/group__lswr.html
- FFmpeg libswresample: https://ffmpeg.org/libswresample.html
- Channel Layout API: https://ffmpeg.org/doxygen/trunk/channel__layout_8h.html

### Technical References
- Polyphase Filter Banks: Crochiere & Rabiner, "Multirate Signal Processing"
- SIMD Optimization: Intel/AMD Architecture Manuals
- Audio Resampling: https://ccrma.stanford.edu/~jos/resample/

---

## 14. IMPLEMENTATION CHECKLIST

### Build & Configuration
- [ ] Link against libswresample: `-lswresample`
- [ ] Include header: `#include <libswresample/swresample.h>`
- [ ] Verify library available: `pkg-config --libs swresample`

### Context Setup
- [ ] Allocate SwrContext with `swr_alloc()` or `swr_alloc_set_opts2()`
- [ ] Set input/output sample formats, rates, and channel layouts
- [ ] Call `swr_init()` before first conversion
- [ ] Handle initialization errors

### Conversion Loop
- [ ] Call `swr_convert()` or `swr_convert_frame()` in a loop
- [ ] Handle AVERROR(EAGAIN) — indicates need to consume output first
- [ ] Flush remaining output with `swr_convert(ctx, out, count, NULL, 0)`
- [ ] Check for remaining input with `swr_get_delay()`

### Memory Management
- [ ] Allocate output buffers with sufficient size (use `swr_get_out_samples()` for estimate)
- [ ] Free SwrContext with `swr_free()`
- [ ] Unref frames with `av_frame_unref()`
- [ ] Free input/output buffers appropriately

### Channel Mixing
- [ ] Use standard layouts from `AV_CHANNEL_LAYOUT_*` constants
- [ ] Set custom matrix with `swr_set_matrix()` for non-standard mixing
- [ ] Consider rematrix_volume for output level control

### Performance
- [ ] Reuse SwrContext for repeated conversions
- [ ] Use planar formats for internal processing
- [ ] Match input/output rates when possible
- [ ] Use frame API for cleaner code when appropriate

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
