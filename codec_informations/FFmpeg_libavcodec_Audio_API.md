# FFmpeg libavcodec Audio API — Deep Technical Reference

> **Category:** FFmpeg API
> **File Extensions:** N/A (API)
> **MIME Types:** N/A
> **Standardization Body:** FFmpeg Project
> **Specification Document:** https://ffmpeg.org/ffmpeg-codecs.html
> **Patent Status:** N/A
> **License:** LGPL 2.1+

---

## 1. HISTORICAL CONTEXT & ORIGIN

### Origins of FFmpeg and libavcodec

FFmpeg was originally created by Fabrice Bellard in 2000 as a command-line utility for recording, converting, and streaming audio and video. The project evolved from an earlier project called "FFmpeg" which was part of the "MPlayer" movie player project. The name "FFmpeg" stands for "Fast Forward MPEG," reflecting its original focus on MPEG video codec handling.

The libavcodec library emerged as the core audio and video codec implementation within FFmpeg. It was designed from the ground up to provide a unified, portable interface for audio and video encoding and decoding operations. The library grew organically as contributors added support for an ever-expanding array of codec formats.

### The Problem It Solved

Before libavcodec's existence, developers who wanted to decode or encode audio in their applications had limited options. They would need to:

1. Implement codecs from scratch (enormous engineering effort)
2. Use platform-specific APIs (Windows Media Foundation, Core Audio, DirectShow, etc.)
3. License proprietary codec implementations at significant cost
4. Use fragmented, inconsistent APIs across different operating systems

Libavcodec solved these problems by providing:

- **Cross-platform consistency**: The same API works identically on Linux, macOS, Windows, BSD, and embedded systems
- **Comprehensive codec coverage**: Support for hundreds of audio and video codecs in a single library
- **Open-source accessibility**: LGPL licensing allowed both proprietary and open-source usage
- **Performance optimization**: Hand-optimized assembly code for critical paths on multiple architectures

### Key Version History

| Version | Release Date | Major Changes |
|---------|-------------|---------------|
| 0.1 | 2000 | Initial release, basic codec support |
| 0.4 | 2001-2002 | Expanded format support, initial API stability |
| 0.5 "Lavc" | 2004-2005 | Complete API redesign,分离的libavcodec structure |
| 0.6 | 2006-2007 | Improved seeking, better error handling |
| 0.7 | 2009-2010 | Added H.264 in hardware, improved threading |
| 1.0 | 2012 | Major API cleanup, introduction of AVFrame |
| 2.0 | 2013-2014 | Better API for frame handling, avcodec_send_frame introduced |
| 3.0 | 2015 | Memory management improvements |
| 3.4 | 2017 | Legacy API marked deprecated, send/receive model emphasized |
| **4.0** | **2018** | **Send/receive API became the primary model** |
| 4.4 | 2021 | Improved channel layout API |
| 5.0 | 2022 | Further API refinements |
| 6.0 | 2023 | Modern channel layout functions |
| 7.0 | 2024 | Current stable, full modern API support |

### Current Status

FFmpeg and libavcodec are **actively maintained** with regular releases. The project has thousands of contributors and is used in countless production systems worldwide. The send/receive API model introduced in FFmpeg 4.0 (2018) is now the standard approach, with the legacy encode/decode functions officially deprecated.

### Industry Adoption

FFmpeg libavcodec is used in:

- **Video editing software**: Kdenlive, Shotcut, HandBrake
- **Streaming platforms**: YouTube, Twitch back-end processing
- **Audio applications**: Audacity (for some codecs), various DAWs
- **Mobile applications**: Many iOS and Android video apps
- **Web browsers**: Chrome, Firefox use FFmpeg components
- **Server-side transcoding**: Netflix, streaming services use FFmpeg for encoding
- **Embedded systems**: Smart TVs, set-top boxes, automotive entertainment

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### API Design Philosophy: The Send/Receive Paradigm

The FFmpeg libavcodec audio API follows a **send/receive paradigm** that decouples input from output. This design philosophy separates the concerns of data submission and data retrieval, allowing for:

1. **Asynchronous operation**: Input and output operations can be performed independently
2. **Buffered processing**: The codec can buffer multiple frames/packets internally
3. **Clean resource management**: Clear ownership semantics for memory
4. **Flexible flow control**: Applications can choose when to send and when to receive

The key functions in this model are:

```
avcodec_send_frame() → avcodec_receive_packet()  // Encoding
avcodec_send_packet() → avcodec_receive_frame()  // Decoding
```

This contrasts with the older **push/pull model** used by the deprecated APIs (`avcodec_encode_audio2()`, `avcodec_decode_audio4()`), where each call immediately processed one unit of data.

### Core Components

#### AVCodec Structure

The `AVCodec` structure represents a codec definition. It is a read-only structure that describes:

- The codec's capabilities and properties
- The codec's name and type (encoder/decoder)
- Pointers to the codec's internal functions
- Supported pixel/sample formats and channel layouts
- Required hardware acceleration support

You never allocate or modify an `AVCodec` directly. Instead, you obtain pointers to codec structures using discovery functions.

Key fields:

```c
const char *name;              // Codec name (e.g., "libfdk_aac", "pcm_s16le")
enum AVMediaType type;         // AVMEDIA_TYPE_AUDIO for audio codecs
int capabilities;              // AV_CODEC_CAP_* flags
const AVRational *supported_framerates;  // For video
const enum AVPixelFormat *pix_fmts;     // For video
const enum AVSampleFormat *sample_fmts;  // For audio
const AVChannelLayout *ch_layouts;      // For audio
int frame_sizes;                        // Typical frame sizes
int max_lowres;                         // Maximum resolution for lowres decoding
```

#### AVCodecContext Structure

The `AVCodecContext` is the central context object for codec operations. You allocate one per codec instance, configure it, and use it for all encoding/decoding operations.

Critical audio-specific fields:

```c
enum AVMediaType codec_type;        // Set automatically from codec
const AVCodec *codec;               // The codec being used
int bit_rate;                       // Target bitrate in bits/second
int sample_rate;                    // Samples per second (e.g., 44100, 48000)
enum AVSampleFormat sample_fmt;     // Audio sample format
AVChannelLayout ch_layout;         // Channel layout structure
int frame_size;                     // Samples per channel per frame (set after open)
int channels;                       // Number of channels (deprecated, use ch_layout)
int frame_number;                    // Frame counter
int delay;                          // Encoding/decoding delay
int64_t profile;                    // Codec profile
int64_t level;                      // Codec level
```

#### AVFrame Structure

The `AVFrame` represents uncompressed audio or video data. For audio, it contains the raw PCM samples to be encoded or the decoded samples from a packet.

Audio-specific fields:

```c
int nb_samples;              // Number of samples per channel
int format;                  // Sample format (AVSampleFormat as int)
int channels;                // Number of channels (deprecated)
uint8_t *data[AV_NUM_DATA_POINTERS];           // Sample data pointers
int linesize[AV_NUM_DATA_POINTERS];            // Line size (buffer size)
uint8_t **extended_data;    // Extended data pointers for many channels
AVChannelLayout ch_layout;  // Channel layout
int sample_rate;             // Sample rate
int64_t pts;                 // Presentation timestamp
int64_t best_effort_timestamp;  // Best effort timestamp
int64_t pkt_duration;        // Duration from packet
int key_frame;               // Key frame flag
enum AVFrameSideDataType;    // Side data types
AVBufferRef *buf[AV_NUM_DATA_POINTERS];         // Buffer references
int nb_extended_buf;         // Number of extended buffers
AVBufferRef **extended_buf;  // Extended buffer references
```

#### AVPacket Structure

The `AVPacket` represents compressed audio or video data. For audio, it typically contains one or more encoded audio frames.

Audio-specific fields:

```c
uint8_t *data;              // Compressed data
int size;                   // Size of data in bytes
int64_t pts;                // Presentation timestamp (AV_NOPTS_VALUE if unknown)
int64_t dts;                // Decompression timestamp
int stream_index;            // Stream index in the format context
int flags;                   // AV_PKT_FLAG_* flags
int64_t duration;            // Duration in stream timebase units
void *opaque;               // User data
AVBufferRef *buf;           // Reference to the data buffer
AVBufferRef *side_data;     // Side data
int side_data_elems;        // Number of side data elements
```

### The Send/Receive API Model vs. Deprecated APIs

The modern API provides several advantages over the legacy functions:

| Aspect | Legacy API | Modern API |
|--------|------------|------------|
| Function calls | `avcodec_encode_audio2()` | `avcodec_send_frame()` + `avcodec_receive_packet()` |
| Buffer management | Manual packet allocation | Reference counting with AVFrame/AVPacket |
| Multiple outputs | Single output per call | Can produce multiple outputs per input |
| Draining | Not directly supported | Clean draining via NULL frame |
| Flushing | Complex state management | Simple flush packet mechanism |
| Error handling | Mixed success/error returns | Consistent error codes |
| Thread safety | Limited | Better isolation of operations |

**Deprecated functions (do not use in new code):**

```c
// DEPRECATED - Do not use
int avcodec_encode_audio2(AVCodecContext *avctx, AVPacket *avpkt,
                          const AVFrame *frame, int *got_packet_ptr);
int avcodec_decode_audio4(AVCodecContext *avctx, AVFrame *frame,
                          int *got_frame_ptr, const AVPacket *avpkt);
int avcodec_encode_video2(AVCodecContext *avctx, AVPacket *avpkt,
                         const AVFrame *frame, int *got_packet_ptr);
int avcodec_decode_video2(AVCodecContext *avctx, AVFrame *frame,
                          int *got_frame_ptr, const AVPacket *avpkt);
```

### Codec Discovery

FFmpeg provides several functions for finding codecs:

```c
// Find encoder by category and codec ID
AVCodec *avcodec_find_encoder(enum AVMediaType type, enum AVCodecID id);

// Find decoder by codec ID
AVCodec *avcodec_find_decoder(enum AVCodecID id);

// Find encoder by name
AVCodec *avcodec_find_encoder_by_name(const char *name);

// Find decoder by name
AVCodec *avcodec_find_decoder_by_name(const char *name);

// Find encoder by category
AVCodec *avcodec_find_encoder_by_name(const char *name);

// Iterate through all codecs
const AVCodec *av_codec_iterate(void *opaque);
```

Common audio codec IDs:

```c
AV_CODEC_ID_PCM_S16LE    // Uncompressed 16-bit little-endian PCM
AV_CODEC_ID_PCM_S16BE    // Uncompressed 16-bit big-endian PCM
AV_CODEC_ID_PCM_F32LE    // Uncompressed 32-bit float little-endian
AV_CODEC_ID_AAC          // AAC-LC
AV_CODEC_ID_AC3          // Dolby Digital AC-3
AV_CODEC_ID_EAC3         // Dolby Digital Plus
AV_CODEC_ID_MP3          // MP3 (MPEG-1 Layer III)
AV_CODEC_ID_FLAC         // Free Lossless Audio Codec
AV_CODEC_ID_OPUS         // Opus
AV_CODEC_ID_VORBIS       // Vorbis
AV_CODEC_ID_ALAC         // Apple Lossless
AV_CODEC_ID_WMAV1        // Windows Media Audio v1
AV_CODEC_ID_WMAV2        // Windows Media Audio v2
AV_CODEC_ID_WMAVOICE     // Windows Media Voice
```

### Packet vs. Frame Granularity

Understanding the difference between packets and frames is crucial:

- **AVPacket**: A compressed data unit from a container format. A packet may contain one complete frame, multiple frames (common in audio), or a fragment of a frame (rare for audio).

- **AVFrame**: An uncompressed unit of audio samples. For audio, a frame typically contains a fixed number of samples per channel.

In the send/receive API:

```c
// Decoding: Packet → Frame(s)
avcodec_send_packet(ctx, packet);    // Send compressed packet
avcodec_receive_frame(ctx, frame);   // Receive uncompressed frame(s)

// Encoding: Frame → Packet(s)
avcodec_send_frame(ctx, frame);      // Send uncompressed frame
avcodec_receive_packet(ctx, packet); // Receive compressed packet(s)
```

For audio codecs, a single packet often produces multiple frames upon decoding. The API handles this transparently—you call `avcodec_receive_frame()` repeatedly until it returns `AVERROR(EAGAIN)`.

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 Key Data Structures

#### AVCodec Structure Fields

The `AVCodec` structure (defined in `libavcodec/avcodec.h`) contains the following critical fields:

```c
struct AVCodec {
    const char *name;                    // Internally registered name
    const char *long_name;               // Human-readable name
    enum AVMediaType type;               // AVMEDIA_TYPE_AUDIO or AVMEDIA_TYPE_VIDEO
    enum AVCodecID id;                   // Unique codec identifier
    int capabilities;                    // Capability flags

    // Supported formats - set by codec registration
    const enum AVSampleFormat *sample_fmts;        // Audio sample formats
    const int *supported_samplerates;             // Supported sample rates
    const uint64_t *channel_layouts;               // Supported channel layouts
    const enum AVPixelFormat *pix_fmts;            // Video pixel formats
    int max_lowres;                                // Max lowres value

    // Internal data
    const struct AVCodec *next;                    // Linked list pointer
    const AVClass *priv_class;                    // Private options class
    const int *profiles;                           // Supported profiles

    // Initialization and capabilities
    int (*init_thread_copy)(AVCodecContext*);
    int (*update_thread_context)(AVCodecContext* dst, const AVCodecContext* src);
    const AVCodecDefault *defaults;

    // Codec-specific functions
    int (*init)(AVCodecContext*);                 // Codec initialization
    int (*encode_sub)(AVCodecContext*, uint8_t *buf, int buf_size,
                      const struct AVSubtitle *sub);
    int (*encode2)(AVCodecContext *avctx, AVPacket *pkt,
                   const AVFrame *frame, int *got_packet_ptr);
    int (*decode)(AVCodecContext *avctx, void *outdata,
                  int *outdata_size, AVPacket *avpkt);
    int (*close)(AVCodecContext*);                 // Codec close/deinit

    // Private data size
    int priv_data_size;
};
```

#### AVCodecContext: Audio-Specific Fields

```c
// Audio format parameters
int sample_rate;                    // Samples per second (e.g., 44100, 48000)
enum AVSampleFormat sample_fmt;    // Sample format (packed/planar)
AVChannelLayout ch_layout;         // Channel layout structure
int channels;                      // DEPRECATED: Use ch_layout.nb_channels

// Encoding parameters
int64_t bit_rate;                  // Target bitrate in bits/second
int frame_size;                    // Samples per channel per frame (set after open)
int frame_number;                  // Frame counter

// Codec control
int sample_fmt;                    // Requested sample format
int request_sample_fmt;             // Desired sample format for decoder output

// Encoder-specific
int compression_level;             // Compression level (0-12 for some codecs)
float frame_rate;                  // Target frame rate (video)
int max_b_frames;                  // Maximum B-frames (video)

// Quality vs. size tradeoffs
int qmin;                          // Minimum quantization parameter
int qmax;                          // Maximum quantization parameter

// Threading
int thread_count;                  // Number of threads to use
int thread_type;                   // FF_THREAD_SLICE or FF_THREAD_FRAME
int active_thread_type;            // Actual threading type used

// Timing
int64_t delay;                     // Encoding/decoding delay
int seek_preroll;                  // Seek preroll in milliseconds
```

#### AVFrame: Audio-Specific Fields

```c
// Audio sample data
int nb_samples;                    // Number of samples per channel in this frame
int format;                        // Sample format as integer
int channels;                      // DEPRECATED: Use ch_layout.nb_channels
uint8_t *data[AV_NUM_DATA_POINTERS];  // Pointer to sample data
int linesize[AV_NUM_DATA_POINTERS];   // Size of each channel plane

// Extended data for many channels
uint8_t **extended_data;           // All data pointers (includes extended)
int nb_extended_buf;               // Number of extended buffers
AVBufferRef **extended_buf;        // Extended buffer references

// Buffer management
AVBufferRef *buf[AV_NUM_DATA_POINTERS];  // Buffer references
int buf_alignment;                 // Buffer alignment requirement

// Audio metadata
AVChannelLayout ch_layout;         // Channel layout
int sample_rate;                   // Sample rate

// Timing
int64_t pts;                       // Presentation timestamp
int64_t best_effort_timestamp;    // Best effort timestamp
int64_t pkt_duration;              // Duration from source packet
int64_t pkt_pos;                   // Position in file
int64_t pkt_size;                  // Size of source packet

// Frame properties
int key_frame;                     // 1 if keyframe, 0 otherwise
enum AVPictureType pict_type;      // Picture type (video)
int coded_picture_number;           // Coded picture number
int display_picture_number;         // Display picture number
int quality;                       // Quality (for quality-based encoding)

// Side data
AVFrameSideData *side_data;        // Side data array
int nb_side_data;                  // Number of side data elements
int flags;                         // Frame flags
```

#### AVPacket: Audio-Specific Fields

```c
// Compressed data
uint8_t *data;                    // Pointer to compressed data
int size;                         // Size of data in bytes

// Timing (in stream's time base)
int64_t pts;                      // Presentation timestamp
int64_t dts;                      // Decompression timestamp
int64_t duration;                 // Duration in time base units

// Stream association
int stream_index;                  // Index in AVFormatContext

// Flags and properties
int flags;                        // AV_PKT_FLAG_* flags
#define AV_PKT_FLAG_KEY     0x0001  // Key packet
#define AV_PKT_FLAG_CORRUPT 0x0002  // Corrupted packet
#define AV_PKT_FLAG_DISCARD 0x0004  // Discard this packet

// Buffer management
AVBufferRef *buf;                  // Reference to data buffer
void *opaque;                      // User data

// Side data
AVBufferRef *side_data;           // Side data buffer
int side_data_elems;               // Number of side data elements

// Destructors
void (*destruct)(AVPacket*);       // Custom destructor
void (*destruct2)(AVPacket*);     // Alternative destructor
void *opaque_ref;                  // Opaque reference
```

### 3.2 Sample Format System (AVSampleFormat)

#### The Complete Format Enum

```c
enum AVSampleFormat {
    AV_SAMPLE_FMT_NONE = -1,       // Undefined format

    // Packed (interleaved) formats - all channels in one buffer
    AV_SAMPLE_FMT_U8,             // unsigned 8 bits, interleaved
    AV_SAMPLE_FMT_S16,            // signed 16 bits, interleaved
    AV_SAMPLE_FMT_S32,            // signed 32 bits, interleaved
    AV_SAMPLE_FMT_FLT,            // float (32-bit), interleaved
    AV_SAMPLE_FMT_DBL,            // double (64-bit), interleaved
    AV_SAMPLE_FMT_S64,            // signed 64 bits, interleaved

    // Planar formats - each channel in separate buffer
    AV_SAMPLE_FMT_U8P,            // unsigned 8 bits, planar
    AV_SAMPLE_FMT_S16P,           // signed 16 bits, planar
    AV_SAMPLE_FMT_S32P,           // signed 32 bits, planar
    AV_SAMPLE_FMT_FLTP,           // float (32-bit), planar
    AV_SAMPLE_FMT_DBLP,           // double (64-bit), planar
    AV_SAMPLE_FMT_S64P,           // signed 64 bits, planar

    AV_SAMPLE_FMT_NB              // Number of formats (not a valid format)
};
```

#### Planar vs. Packed Layouts

The distinction between planar and packed formats is fundamental to understanding FFmpeg's audio handling:

**Packed Format (Interleaved):**

```
Memory Layout for Stereo (2 channels):
[data[0] format: S16]

Address:  0    1    2    3    4    5    6    7
Content:  L0   L1   R0   R1   L2   L3   R2   R3
          └────┬────┘└────┬────┘└────┬────┘
          Sample 0       Sample 1    Sample 2

For 16-bit stereo PCM:
- Sample 0: bytes 0-1 (left), bytes 2-3 (right)
- Sample 1: bytes 4-5 (left), bytes 6-7 (right)
```

**Planar Format (Non-Interleaved):**

```
Memory Layout for Stereo (2 channels):
[data[0]] = left channel plane
[data[1]] = right channel plane

Address (data[0]):  0    1    2    3    4    5
Content:             L0   L1   L2   L3   L4   L5

Address (data[1]):  0    1    2    3    4    5
Content:             R0   R1   R2   R3   R4   R5

For planar float (FLTP):
- data[0] contains all left channel samples consecutively
- data[1] contains all right channel samples consecutively
```

#### Float Formats Range

For floating-point formats, sample values are normalized to a specific range:

- **Float (FLT/FLTP)**: Range [-1.0, 1.0]
- **Double (DBL/DBLP)**: Range [-1.0, 1.0]

Values outside this range may cause clipping during encoding or processing.

#### Format Conversion Functions

```c
// Get packed alternative (interleaved)
enum AVSampleFormat av_get_packed_sample_fmt(enum AVSampleFormat sample_fmt);

// Get planar alternative (non-interleaved)
enum AVSampleFormat av_get_planar_sample_fmt(enum AVSampleFormat sample_fmt);

// Get alternative (packed <-> planar)
enum AVSampleFormat av_get_alt_sample_fmt(enum AVSampleFormat sample_fmt, int planar);

// Get name from format
const char *av_get_sample_fmt_name(enum AVSampleFormat sample_fmt);

// Get format from name
enum AVSampleFormat av_get_sample_fmt(const char *name);

// Get planar format for a packed format
enum AVSampleFormat av_get_planar_sample_fmt(enum AVSampleFormat sample_fmt);
```

### 3.3 Channel Layout System (AVChannelLayout)

#### The AVChannelLayout Structure

```c
struct AVChannelLayout {
    enum AVChannelOrder order;     // Channel ordering method

    int nb_channels;               // Number of channels

    union {
        uint64_t mask;            // For AV_CHANNEL_ORDER_NATIVE
        AVChannelCustom *map;     // For AV_CHANNEL_ORDER_CUSTOM
    } u;

    void *opaque;                 // Private data
};
```

#### Channel Order Enumeration

```c
enum AVChannelOrder {
    AV_CHANNEL_ORDER_UNSPEC,      // Only channel count is known
    AV_CHANNEL_ORDER_NATIVE,       // Native channel order (use mask)
    AV_CHANNEL_ORDER_CUSTOM,       // Custom channel order (use map)
    AV_CHANNEL_ORDER_AMBISONIC,    // Ambisonic WXYZs order
};
```

#### Predefined Channel Layouts

FFmpeg provides predefined channel layouts for common configurations:

```c
// Macro for defining channel layouts
#define AV_CHANNEL_LAYOUT_MASK(nb, m) \
    { .order = AV_CHANNEL_ORDER_NATIVE, \
      .nb_channels = (nb), \
      .u.mask = (m), \
      .opaque = NULL }

// Common layouts
#define AV_CHANNEL_LAYOUT_MONO \
    AV_CHANNEL_LAYOUT_MASK(1, AV_CH_LAYOUT_MONO)           // 1 channel

#define AV_CHANNEL_LAYOUT_STEREO \
    AV_CHANNEL_LAYOUT_MASK(2, AV_CH_LAYOUT_STEREO)         // 2 channels (L,R)

#define AV_CHANNEL_LAYOUT_2POINT1 \
    AV_CHANNEL_LAYOUT_MASK(3, AV_CH_LAYOUT_2POINT1)        // L,R,Sub

#define AV_CHANNEL_LAYOUT_5POINT0 \
    AV_CHANNEL_LAYOUT_MASK(5, AV_CH_LAYOUT_5POINT0)        // L,R,C,BL,BR

#define AV_CHANNEL_LAYOUT_5POINT1 \
    AV_CHANNEL_LAYOUT_MASK(6, AV_CH_LAYOUT_5POINT1)        // L,R,C,BL,BR,Sub

#define AV_CHANNEL_LAYOUT_5POINT0_BACK \
    AV_CHANNEL_LAYOUT_MASK(5, AV_CH_LAYOUT_5POINT0_BACK)   // L,R,C,SL,SR

#define AV_CHANNEL_LAYOUT_5POINT1_BACK \
    AV_CHANNEL_LAYOUT_MASK(6, AV_CH_LAYOUT_5POINT1_BACK)   // L,R,C,SL,SR,Sub

#define AV_CHANNEL_LAYOUT_7POINT1 \
    AV_CHANNEL_LAYOUT_MASK(8, AV_CH_LAYOUT_7POINT1)        // L,R,C,BL,BR,SL,SR,Sub
```

#### Channel Bit Masks

The channel mask uses these bit definitions:

```c
// Individual channel positions
#define AV_CH_FRONT_LEFT           0x00000001ULL
#define AV_CH_FRONT_RIGHT          0x00000002ULL
#define AV_CH_FRONT_CENTER         0x00000004ULL
#define AV_CH_LOW_FREQUENCY        0x00000008ULL  // Subwoofer
#define AV_CH_BACK_LEFT            0x00000010ULL
#define AV_CH_BACK_CENTER          0x00000020ULL
#define AV_CH_BACK_RIGHT           0x00000040ULL
#define AV_CH_SIDE_LEFT            0x00000080ULL
#define AV_CH_SIDE_RIGHT           0x00000100ULL
#define AV_CH_TOP_CENTER           0x00000200ULL
#define AV_CH_TOP_FRONT_LEFT       0x00000400ULL
#define AV_CH_TOP_FRONT_CENTER     0x00000800ULL
#define AV_CH_TOP_FRONT_RIGHT      0x00001000ULL
#define AV_CH_TOP_BACK_LEFT        0x00002000ULL
#define AV_CH_TOP_BACK_CENTER      0x00004000ULL
#define AV_CH_TOP_BACK_RIGHT       0x00008000ULL
#define AV_CH_STEREO_LEFT          0x20000000ULL  // DPLII "left"
#define AV_CH_STEREO_RIGHT         0x40000000ULL  // DPLII "right"
#define AV_CH_WIDE_LEFT            0x0000002000000000ULL
#define AV_CH_WIDE_RIGHT           0x0000004000000000ULL
#define AV_CH_SURROUND_DIRECT_LEFT 0x0000008000000000ULL
#define AV_CH_SURROUND_DIRECT_RIGHT 0x0000010000000000ULL
#define AV_CH_LOW_FREQUENCY_2      0x0000020000000000ULL

// Layout compositions
#define AV_CH_LAYOUT_MONO          (AV_CH_FRONT_CENTER)
#define AV_CH_LAYOUT_STEREO        (AV_CH_FRONT_LEFT|AV_CH_FRONT_RIGHT)
#define AV_CH_LAYOUT_2POINT1       (AV_CH_LAYOUT_STEREO|AV_CH_LOW_FREQUENCY)
#define AV_CH_LAYOUT_5POINT0       (AV_CH_LAYOUT_STEREO|AV_CH_FRONT_CENTER|AV_CH_SIDE_LEFT|AV_CH_SIDE_RIGHT)
#define AV_CH_LAYOUT_5POINT1       (AV_CH_LAYOUT_5POINT0|AV_CH_LOW_FREQUENCY)
#define AV_CH_LAYOUT_5POINT0_BACK  (AV_CH_LAYOUT_STEREO|AV_CH_FRONT_CENTER|AV_CH_BACK_LEFT|AV_CH_BACK_RIGHT)
#define AV_CH_LAYOUT_5POINT1_BACK  (AV_CH_LAYOUT_5POINT0_BACK|AV_CH_LOW_FREQUENCY)
#define AV_CH_LAYOUT_7POINT1       (AV_CH_LAYOUT_5POINT1|AV_CH_SIDE_LEFT|AV_CH_SIDE_RIGHT)
```

#### Channel Layout Functions

```c
// Get default layout for number of channels
void av_channel_layout_default(AVChannelLayout *ch_layout, int nb_channels);

// Initialize from mask
int av_channel_layout_from_mask(AVChannelLayout *ch_layout, uint64_t mask);

// Initialize from string
int av_channel_layout_from_string(AVChannelLayout *ch_layout,
                                  const char *str);

// Uninitialize and free resources
void av_channel_layout_uninit(AVChannelLayout *ch_layout);

// Copy layout
int av_channel_layout_copy(AVChannelLayout *dst, const AVChannelLayout *src);

// Check if layouts are compatible
int av_channel_layout_compare(const AVChannelLayout *a, const AVChannelLayout *b);

// Get channel index from string
int av_channel_from_string(const char *name);

// Get channel name from index
const char *av_channel_name(enum AVChannel channel);

// Get channel description
const char *av_channel_description(enum AVChannel channel);

// Get channel from layout
enum AVChannel av_channel_layout_channel_from_index(const AVChannelLayout *ch_layout, int idx);

// Check if layout is valid
int av_channel_layout_check(const AVChannelLayout *ch_layout);
```

#### How FFmpeg Represents Channel Order

FFmpeg follows the **WAVE_FORMAT_CHANNEL_ORDER** standard for channel ordering:

| Channel Position | WAVE Order | FFmpeg Position |
|------------------|------------|----------------|
| Mono | 1 | 0 |
| Stereo Left | 1 | 0 |
| Stereo Right | 2 | 1 |
| Front Center | 1 | 0 |
| LFE | 1 | Last |
| Back Surround Left | 5 | n-2 |
| Back Surround Right | 6 | n-1 |
| Side Surround Left | 5 | n-2 |
| Side Surround Right | 6 | n-1 |

---

## 4. ENCODING ALGORITHM (API WORKFLOW)

### 4.1 Complete Encoding Workflow

The following step-by-step process describes the complete encoding workflow using the modern send/receive API:

#### Step 1: Find the Encoder

```c
AVCodec *codec = avcodec_find_encoder_by_name("libfdk_aac");
if (!codec) {
    // Handle error - codec not found
}
```

#### Step 2: Allocate the Codec Context

```c
AVCodecContext *ctx = avcodec_alloc_context3(codec);
if (!ctx) {
    // Handle error - allocation failed
}
```

#### Step 3: Configure Audio Parameters

```c
ctx->sample_rate = 44100;                    // Set sample rate
ctx->bit_rate = 128000;                     // Set bitrate (128 kbps)
ctx->ch_layout = (AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO;
ctx->sample_fmt = AV_SAMPLE_FMT_S16;        // Request input format

// For VBR encoding (AAC)
ctx->flags |= AV_CODEC_FLAG_QSCALE;          // Enable quality-based encoding
ctx->global_quality = 100;                   // Quality level (0-500, higher = better)
```

#### Step 4: Open the Codec

```c
int ret = avcodec_open2(ctx, codec, NULL);
if (ret < 0) {
    // Handle error - failed to open codec
    char errbuf[128];
    av_strerror(ret, errbuf, sizeof(errbuf));
    fprintf(stderr, "Failed to open codec: %s\n", errbuf);
}
```

#### Step 5: Allocate the Frame

```c
AVFrame *frame = av_frame_alloc();
if (!frame) {
    // Handle error
}

frame->nb_samples = ctx->frame_size;         // Set frame size
frame->format = ctx->sample_fmt;             // Set format
frame->ch_layout = ctx->ch_layout;           // Set channel layout
```

#### Step 6: Allocate Frame Buffers

```c
ret = av_frame_get_buffer(frame, 0);         // Allocate with default alignment
if (ret < 0) {
    // Handle error
}
```

#### Step 7: Fill Frame with Audio Data

```c
// Option A: Direct pointer access (for packed format)
int16_t *samples = (int16_t *)frame->data[0];
for (int i = 0; i < frame->nb_samples * 2; i++) {
    samples[i] = read_sample_from_source();
}

// Option B: For planar format, fill each plane separately
float *left = (float *)frame->data[0];
float *right = (float *)frame->data[1];
```

#### Step 8: Send Frame to Encoder

```c
ret = avcodec_send_frame(ctx, frame);
if (ret < 0) {
    // Handle error
    av_frame_unref(frame);
    // Don't call av_frame_free() if reusing the frame
}
```

#### Step 9: Receive Encoded Packets

```c
AVPacket *pkt = av_packet_alloc();

while (ret >= 0) {
    ret = avcodec_receive_packet(ctx, pkt);
    if (ret == AVERROR(EAGAIN)) {
        // No packet ready - need to send more input
        ret = 0;
        break;
    } else if (ret == AVERROR_EOF) {
        // Encoder fully flushed
        break;
    } else if (ret < 0) {
        // Error
        break;
    }

    // Process the encoded packet
    write_packet_to_output(pkt);

    // Unref packet for reuse
    av_packet_unref(pkt);
}
```

#### Step 10: Drain the Encoder (End of Stream)

```c
// Send NULL frame to enter draining mode
ret = avcodec_send_frame(ctx, NULL);
if (ret < 0) {
    // Handle error
}

// Receive remaining packets
while (1) {
    ret = avcodec_receive_packet(ctx, pkt);
    if (ret == AVERROR(EAGAIN)) {
        // No more packets
        break;
    } else if (ret == AVERROR_EOF) {
        // Fully drained
        break;
    } else if (ret < 0) {
        // Error
        break;
    }

    // Process final packets
    write_packet_to_output(pkt);
    av_packet_unref(pkt);
}
```

#### Step 11: Cleanup

```c
av_frame_free(&frame);
av_packet_free(&pkt);
avcodec_free_context(&ctx);
```

### 4.2 Frame Size Requirements

Most audio encoders require input frames to have a specific number of samples. This is determined by the `frame_size` field in `AVCodecContext`, which is set after `avcodec_open2()`.

#### When nb_samples Must Be Set

```c
// After opening the codec
avcodec_open2(ctx, codec, NULL);
int required_samples = ctx->frame_size;  // e.g., 1024 for AAC

// Frame allocation for encoding
AVFrame *frame = av_frame_alloc();
frame->nb_samples = required_samples;
frame->format = ctx->sample_fmt;
frame->ch_layout = ctx->ch_layout;
av_frame_get_buffer(frame, 0);
```

#### Handling Variable Frame Sizes

For codecs with `AV_CODEC_CAP_VARIABLE_FRAME_SIZE`:

```c
if (ctx->codec->capabilities & AV_CODEC_CAP_VARIABLE_FRAME_SIZE) {
    // Can use any frame size
    frame->nb_samples = arbitrary_size;
}
```

For the final frame (when no more input is available), the frame can be smaller than `frame_size`:

```c
// Last frame - can be smaller
frame->nb_samples = remaining_samples;
```

#### Padding Samples

Some encoders require padding samples to handle filter latency. The `initial_padding` and `trailing_padding` fields in the codec context provide this information:

```c
// After codec open
int padding_samples = ctx->initial_padding;  // Samples of encoder delay
```

### 4.3 Error Handling

#### Complete AVERROR Reference

FFmpeg error codes are negative values. The `AVERROR()` macro converts POSIX errors to FFmpeg format:

```c
// POSIX errno values used
AVERROR(EAGAIN)    // Resource temporarily unavailable
AVERROR(ENOMEM)    // Cannot allocate memory
AVERROR(EINVAL)    // Invalid argument
AVERROR_EOF        // End of file (special FFmpeg constant)
AVERROR_EXIT       // Immediate exit requested

// FFmpeg-specific errors
#define AVERROR_BSF_NOT_FOUND      FFERRTAG('B','S','F')  // Bitstream filter not found
#define AVERROR_DECODER_NOT_FOUND FFERRTAG('D','E','C')   // Decoder not found
#define AVERROR_ENCODER_NOT_FOUND FFERRTAG('E','N','C')   // Encoder not found
#define AVERROR_DEMUXER_NOT_FOUND FFERRTAG('D','E','M')   // Demuxer not found
#define AVERROR_MUXER_NOT_FOUND   FFERRTAG('M','U','X')   // Muxer not found
#define AVERROR_INVALIDDATA       FFERRTAG('I','N','D')   // Invalid data
#define AVERROR_BUG                FFERRTAG('B','U','G')  // Internal bug
#define AVERROR_UNKNOWN            FFERRTAG('U','N','K')  // Unknown error
#define AVERROR_EXTERNAL           FFERRTAG('E','X','T')  // External library error
```

#### Handling AVERROR(EAGAIN)

`AVERROR(EAGAIN)` indicates that the operation cannot complete in the current state:

**When sending a frame:**
```c
ret = avcodec_send_frame(ctx, frame);
if (ret == AVERROR(EAGAIN)) {
    // Output buffer is full - must receive packets first
    while (1) {
        ret = avcodec_receive_packet(ctx, pkt);
        if (ret == AVERROR(EAGAIN)) break;
        if (ret < 0) goto error;
        // Process packet...
        av_packet_unref(pkt);
    }
    // Retry send after draining
    ret = avcodec_send_frame(ctx, frame);
}
```

**When receiving a packet:**
```c
ret = avcodec_receive_packet(ctx, pkt);
if (ret == AVERROR(EAGAIN)) {
    // Need to send more input frames
    // This is normal - just send more input
}
```

### 4.4 Memory Management

#### Reference Counting with AVFrame

AVFrame uses reference counting to manage memory efficiently:

```c
// AVFrame internal buffer references
AVBufferRef *buf[AV_NUM_DATA_POINTERS];      // Main buffers
int nb_extended_buf;                          // Extended buffer count
AVBufferRef **extended_buf;                   // Extended buffers
```

#### Reference Operations

```c
// Create a new reference to the frame
int av_frame_ref(AVFrame *dst, const AVFrame *src);

// Free one reference (decrement refcount)
void av_frame_unref(AVFrame *frame);

// Free frame and all its buffers
void av_frame_free(AVFrame **frame);

// Make the frame writable (copy if shared)
int av_frame_make_writable(AVFrame *frame);
```

#### When to Use Each Function

| Function | Use Case | Effect |
|----------|----------|--------|
| `av_frame_ref()` | Share frame data | Increments refcount, shares buffers |
| `av_frame_unref()` | Release frame after use | Decrements refcount, frees if zero |
| `av_frame_make_writable()` | Before modifying frame data | Copies if refcount > 1 |
| `av_frame_free()` | Final cleanup | Frees frame structure and buffers |

#### Packet Reference Counting

```c
// Similar to frame references
int av_packet_ref(AVPacket *dst, const AVPacket *src);
void av_packet_unref(AVPacket *pkt);
void av_packet_free(AVPacket **pkt);

// Clone entire packet with new allocation
int av_packet_clone(AVPacket **dst, const AVPacket *src);
```

#### Thread Safety

The reference counting mechanism is thread-safe:

- Each `AVBufferRef` has its own reference count
- Operations on different buffers are independent
- Same buffer accessed from multiple threads requires external synchronization

---

## 5. DECODING ALGORITHM (API WORKFLOW)

### 5.1 Complete Decoding Workflow

#### Step 1: Find the Decoder

```c
AVCodec *codec = avcodec_find_decoder(AV_CODEC_ID_AAC);
if (!codec) {
    // Handle error
}
```

#### Step 2: Allocate Codec Context

```c
AVCodecContext *ctx = avcodec_alloc_context3(codec);
if (!ctx) {
    // Handle error
}
```

#### Step 3: Open the Codec

```c
// Optionally copy parameters from stream
avcodec_parameters_to_context(ctx, stream->codecpar);

int ret = avcodec_open2(ctx, codec, NULL);
if (ret < 0) {
    // Handle error
}
```

#### Step 4: Allocate Packet and Frame

```c
AVPacket *pkt = av_packet_alloc();
AVFrame *frame = av_frame_alloc();

if (!pkt || !frame) {
    // Handle error
}
```

#### Step 5: Read Encoded Packets

```c
while (av_read_frame(fmt_ctx, pkt) >= 0) {
    if (pkt->stream_index != audio_stream_index) {
        av_packet_unref(pkt);
        continue;
    }

    // Process this packet
    break;
}
```

#### Step 6: Send Packet to Decoder

```c
ret = avcodec_send_packet(ctx, pkt);
if (ret < 0) {
    // Handle error
    av_packet_unref(pkt);
}
```

#### Step 7: Receive Decoded Frames

```c
while (ret >= 0) {
    ret = avcodec_receive_frame(ctx, frame);
    if (ret == AVERROR(EAGAIN)) {
        // Need to send more packets
        ret = 0;
        break;
    } else if (ret == AVERROR_EOF) {
        // Decoder fully flushed
        break;
    } else if (ret < 0) {
        // Error
        break;
    }

    // Process decoded frame
    process_audio_frame(frame);

    // Unref frame for next use
    av_frame_unref(frame);
}

// Packet consumed, unref for next packet
av_packet_unref(pkt);
```

#### Step 8: Drain the Decoder

```c
// Send flush packet
avcodec_send_packet(ctx, NULL);

while (1) {
    ret = avcodec_receive_frame(ctx, frame);
    if (ret == AVERROR(EAGAIN)) {
        break;
    } else if (ret == AVERROR_EOF) {
        break;
    } else if (ret < 0) {
        break;
    }

    process_audio_frame(frame);
    av_frame_unref(frame);
}
```

#### Step 9: Cleanup

```c
av_frame_free(&frame);
av_packet_free(&pkt);
avcodec_free_context(&ctx);
```

### 5.2 Side Data

Side data provides additional metadata for frames and packets.

#### AVFrameSideDataType Enum Values

```c
enum AVFrameSideDataType {
    AV_FRAME_DATA_PANSCAN,           // Pan scan
    AV_FRAME_DATA_A53_CC,            // ATSC A53 Part 4 Closed Captions
    AV_FRAME_DATA_STEREO3D,          // Stereo 3D info
    AV_FRAME_DATA_MATRIXENCODING,    // Matrix encoding (Dolby Surround)
    AV_FRAME_DATA_DOWNMIX_INFO,       // Downmix info
    AV_FRAME_DATA_REPLAYGAIN,        // ReplayGain info
    AV_FRAME_DATA_DISPLAYMATRIX,      // Display orientation
    AV_FRAME_DATA_AFD,               // Active Format Description
    AV_FRAME_DATA_MOTION_VECTORS,     // Motion vectors
    AV_FRAME_DATA_SKIP_SAMPLES,       // Skip samples info
    AV_FRAME_DATA_AUDIO_SERVICE_TYPE, // Audio service type
    AV_FRAME_DATA_MASTERING_DISPLAY_METADATA, // HDR mastering
    AV_FRAME_DATA_GOP_TIMECODE,       // GOP timecode
    AV_FRAME_DATA_REGIONS_OF_INTEREST, // Regions of interest
};
```

#### AVPacketSideDataType for Audio

```c
enum AVPacketSideDataType {
    AV_PKT_DATA_PALETTE,
    AV_PKT_DATA_NEW_EXTRADATA,
    AV_PKT_DATA_PARAM_CHANGE,
    AV_PKT_DATA_H263_MB_INFO,
    AV_PKT_DATA_REPLAYGAIN,           // ReplayGain for audio
    AV_PKT_DATA_DISPLAYMATRIX,
    AV_PKT_DATA_STEREO3D,
    AV_PKT_DATA_AUDIO_SERVICE_TYPE,
    AV_PKT_DATA_QUALITY_STATS,
    AV_PKT_DATA_FALLBACK_TRACK,
    AV_PKT_DATA_CPB_PROPERTIES,
    AV_PKT_DATA_SKIP_SAMPLES,
    AV_PKT_DATA_JP_DUALMONO,
    AV_PKT_DATA_STRINGS_METADATA,
    AV_PKT_DATA_SUBTITLE_POSITION,
    AV_PKT_DATA_MATROSKA_BLOCKADDITIONAL,
    AV_PKT_DATA_WEBVTT_IDENTIFIER,
    AV_PKT_DATA_WEBVTT_SETTINGS,
    AV_PKT_DATA_PCaptionRects,
    AV_PKT_DATA_DOVI_CONF,
    AV_PKT_DATA_STEREO3D_INCONSISTENT,
    AV_PKT_DATA_RECTANGLE_RECT,
};
```

#### Accessing Side Data

```c
// Create new side data
AVFrameSideData *sd = av_frame_new_side_data(
    frame,
    AV_FRAME_DATA_REPLAYGAIN,
    sizeof(AVReplayGain)
);

// Get existing side data
AVFrameSideData *sd = av_frame_get_side_data(frame, AV_FRAME_DATA_REPLAYGAIN);
if (sd) {
    AVReplayGain *rg = (AVReplayGain *)sd->data;
    // Access track_gain, track_peak, album_gain, album_peak
}

// Free side data
void av_frame_free_side_data(AVFrame *frame);
```

### 5.3 Flushing / Codec Reset

Flushing is necessary when:

1. Seeking to a new position in the stream
2. Recovering from a decoding error
3. Resuming decoding after a pause
4. Switching between different sections

#### Flushing Procedure

```c
// Method 1: Send NULL packet (FFmpeg 4.0+)
avcodec_flush_buffers(ctx);

// Method 2: Using flush packet (legacy but still works)
AVPacket flush_pkt;
av_init_packet(&flush_pkt);
flush_pkt.data = NULL;
flush_pkt.size = 0;
avcodec_send_packet(ctx, &flush_pkt);

// Clear any buffered frames
while (avcodec_receive_frame(ctx, frame) != AVERROR(EAGAIN)) {
    av_frame_unref(frame);
}
```

---

## 6. SAMPLE RATE CONVERSION & RESAMPLING

### 6.1 SwrContext Setup

The `SwrContext` is FFmpeg's resampling context for audio conversion operations.

#### Allocation Methods

```c
// Method 1: Allocate and configure separately
SwrContext *swr = swr_alloc();
if (!swr) {
    // Handle error
}

// Configure using AVOptions
av_opt_set_channel_layout(swr, "in_channel_layout", AV_CH_LAYOUT_STEREO, 0);
av_opt_set_channel_layout(swr, "out_channel_layout", AV_CH_LAYOUT_5POINT1, 0);
av_opt_set_int(swr, "in_sample_rate", 44100, 0);
av_opt_set_int(swr, "out_sample_rate", 48000, 0);
av_opt_set_sample_fmt(swr, "in_sample_fmt", AV_SAMPLE_FMT_S16P, 0);
av_opt_set_sample_fmt(swr, "out_sample_fmt", AV_SAMPLE_FMT_FLTP, 0);

// Initialize
int ret = swr_init(swr);
if (ret < 0) {
    // Handle error
}

// Method 2: Allocate and configure in one call
SwrContext *swr2 = NULL;
ret = swr_alloc_set_opts2(
    &swr2,                           // Output context pointer
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO,  // Output layout
    AV_SAMPLE_FMT_S16,               // Output format
    44100,                           // Output sample rate
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_5POINT1,  // Input layout
    AV_SAMPLE_FMT_FLTP,              // Input format
    48000,                           // Input sample rate
    0,                               // Log offset
    NULL                             // Log context
);
if (ret < 0) {
    // Handle error
}
```

### 6.2 Sample Format Conversion

#### Basic Conversion with swr_convert

```c
// Prepare input buffer (planar float, stereo, 1000 samples)
float *input_planes[2] = { left_channel, right_channel };
int input_samples = 1000;

// Calculate output buffer size
int64_t delay = swr_get_delay(swr, input_sample_rate);
int max_output = av_rescale_rnd(
    delay + input_samples,           // Total input with pending delay
    output_sample_rate,              // Target sample rate
    input_sample_rate,               // Source sample rate
    AV_ROUND_UP                     // Round up to avoid truncation
);

// Allocate output buffer
uint8_t **output_planes;
int output_samples = max_output;
av_samples_alloc_array_and_samples(
    &output_planes,
    NULL,                            // linesize (unused)
    output_channels,
    output_samples,
    output_format,
    0                                // alignment
);

// Perform conversion
int converted = swr_convert(
    swr,
    output_planes,                   // Output buffers
    output_samples,                  // Output capacity
    input_planes,                    // Input buffers
    input_samples                    // Input sample count
);

// Check converted count
if (converted < 0) {
    // Handle error
}
```

#### Flush Remaining Samples

```c
// After all input is converted, flush remaining samples
int flushed = swr_convert(
    swr,
    output_planes,
    output_samples,
    NULL,                            // NULL input = flush
    0                                // 0 input samples
);
```

### 6.3 Channel Layout Conversion

#### Downmixing (More channels to fewer)

```c
// 5.1 to stereo downmix
SwrContext *swr = swr_alloc_set_opts2(
    &swr,
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO,  // Output: Stereo
    AV_SAMPLE_FMT_S16,
    48000,
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_5POINT1, // Input: 5.1
    AV_SAMPLE_FMT_S16,
    48000,
    0, NULL
);
swr_init(swr);

// Convert - SwrContext handles the downmix automatically
swr_convert(swr, stereo_output, output_samples, channel_51_input, input_samples);
```

#### Upmixing (Fewer channels to more)

```c
// Stereo to 5.1 upmix
SwrContext *swr = swr_alloc_set_opts2(
    &swr,
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_5POINT1, // Output: 5.1
    AV_SAMPLE_FMT_S16,
    48000,
    &(AVChannelLayout)AV_CHANNEL_LAYOUT_STEREO,  // Input: Stereo
    AV_SAMPLE_FMT_S16,
    48000,
    0, NULL
);
swr_init(swr);

// Convert - silent channels are added automatically
swr_convert(swr, channel_51_output, output_samples, stereo_input, input_samples);
```

#### Custom Channel Mapping

```c
// Use custom channel mapping (e.g., for different speaker arrangements)
int channel_map[] = { 1, 0, 2, 3, 4, 5 };  // Swap left/right, keep others
swr_set_channel_mapping(swr, channel_map);
swr_init(swr);
```

#### Custom Mixing Matrix

```c
// Set custom mixing coefficients
// matrix[i + stride * o] = weight of input channel i for output channel o
double matrix[6 * 2];  // 6 input, 2 output

// Set custom coefficients (normalized)
for (int i = 0; i < 6; i++) {
    for (int o = 0; o < 2; o++) {
        matrix[i * 2 + o] = 0.0;  // Initialize
    }
}
// Custom downmix implementation...

swr_set_matrix(swr, matrix, 2);  // stride = 2
swr_init(swr);
```

---

## 7. METADATA ARCHITECTURE

### 7.1 AVDictionary

The `AVDictionary` is FFmpeg's key-value store for metadata.

#### Dictionary Structure

```c
// Opaque structure - use API functions only
typedef struct AVDictionary AVDictionary;

// Entry structure (read-only)
typedef struct AVDictionaryEntry {
    char *key;
    char *value;
} AVDictionaryEntry;
```

#### Setting Metadata

```c
// Set string value
int av_dict_set(AVDictionary **pm, const char *key,
                const char *value, int flags);

// Flags for av_dict_set
#define AV_DICT_MATCH_CASE      1   // Match key case-sensitive
#define AV_DICT_IGNORE_SUFFIX   2   // Ignore suffix in key matching
#define AV_DICT_DONT_STRDUP_KEY 4   // Don't duplicate key (caller manages)
#define AV_DICT_DONT_STRDUP_VAL 8   // Don't duplicate value
#define AV_DICT_DONT_OVERWRITE 16  // Don't overwrite existing entry
#define AV_DICT_APPEND          32  // Append to existing entry

// Example: Basic set
AVDictionary *metadata = NULL;
av_dict_set(&metadata, "title", "My Song", 0);
av_dict_set(&metadata, "artist", "Artist Name", 0);

// Example: Take ownership of string (no copy)
char *key = av_strdup("album");
char *value = av_strdup("Album Name");
av_dict_set(&metadata, key, value, AV_DICT_DONT_STRDUP_KEY | AV_DICT_DONT_STRDUP_VAL);

// Example: Append to existing key
av_dict_set(&metadata, "genre", "Rock", 0);
av_dict_set(&metadata, "genre", "/Jazz", AV_DICT_APPEND);  // Results in "Rock/Jazz"

// Set integer value
int av_dict_set_int(AVDictionary **pm, const char *key, int64_t value, int flags);
```

#### Getting Metadata

```c
// Get single entry
AVDictionaryEntry *av_dict_get(const AVDictionary *m, const char *key,
                                const AVDictionaryEntry *prev, int flags);

// Example: Get by exact key
AVDictionaryEntry *entry = av_dict_get(metadata, "title", NULL, 0);
if (entry) {
    printf("Title: %s\n", entry->value);
}

// Iterate through all entries
AVDictionaryEntry *entry = NULL;
while ((entry = av_dict_get(metadata, "", entry, AV_DICT_IGNORE_SUFFIX))) {
    printf("%s = %s\n", entry->key, entry->value);
}

// Flags for av_dict_get
// AV_DICT_MATCH_CASE - match key case-sensitively
// AV_DICT_IGNORE_SUFFIX - prefix match (key="" with this flag iterates all)
```

#### Freeing Dictionary

```c
void av_dict_free(AVDictionary **m);

// Example
av_dict_free(&metadata);  // Sets metadata to NULL
```

#### Common Audio Metadata Keys

| Key | Description | Example Value |
|-----|-------------|---------------|
| title | Track title | "Song Name" |
| artist | Artist name | "Artist" |
| album | Album name | "Album Name" |
| album_artist | Album artist | "Album Artist" |
| composer | Composer | "Composer Name" |
| genre | Genre | "Rock" |
| date | Release date | "2024" |
| track | Track number | "1" or "1/10" |
| disc | Disc number | "1" or "1/2" |
| comment | Comment | "Recorded at..." |
| copyright | Copyright | "2024 Artist" |
| encoder | Encoder used | "Lavf60.0" |
| language | Language | "eng" |
| BPM | Beats per minute | "120" |

### 7.2 ReplayGain Side Data

ReplayGain stores volume normalization information.

#### AVReplayGain Structure

```c
typedef struct AVReplayGain {
    int32_t track_gain;    // Track gain in microbels (divide by 100000 for dB)
    uint32_t track_peak;   // Peak track amplitude (100000 = full scale)
    int32_t album_gain;    // Album gain in microbels
    uint32_t album_peak;   // Peak album amplitude
} AVReplayGain;
```

#### Reading ReplayGain from Frame

```c
AVFrameSideData *sd = av_frame_get_side_data(frame, AV_FRAME_DATA_REPLAYGAIN);
if (sd && sd->data && sd->size >= sizeof(AVReplayGain)) {
    AVReplayGain *rg = (AVReplayGain *)sd->data;

    if (rg->track_gain != INT32_MIN) {
        double gain_db = rg->track_gain / 100000.0;
        printf("Track gain: %.2f dB\n", gain_db);
    }

    if (rg->track_peak != 0) {
        double peak = rg->track_peak / 100000.0;
        printf("Track peak: %.6f\n", peak);
    }
}
```

#### Writing ReplayGain to Frame

```c
AVFrameSideData *sd = av_frame_new_side_data(
    frame,
    AV_FRAME_DATA_REPLAYGAIN,
    sizeof(AVReplayGain)
);

if (sd) {
    AVReplayGain *rg = (AVReplayGain *)sd->data;
    rg->track_gain = -6000000;     // -6 dB in microbels
    rg->track_peak = 98000;        // 0.98 peak
    rg->album_gain = INT32_MIN;    // Unknown
    rg->album_peak = 0;            // Unknown
}
```

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 Thread Model

FFmpeg supports two threading models for codec operations:

#### Frame-Level Threading

Frame threading decodes multiple complete frames in parallel. It introduces a delay of N-1 frames where N is the thread count.

```c
// Configure for frame threading
ctx->thread_count = 4;                    // Use 4 threads
ctx->thread_type = FF_THREAD_FRAME;       // Frame-level parallelism

// Open codec
avcodec_open2(ctx, codec, NULL);

// Check which threading was actually used
if (ctx->active_thread_type & FF_THREAD_FRAME) {
    // Frame threading is active
}
```

#### Slice-Level Threading

Slice threading decodes parts of a single frame in parallel.

```c
// Configure for slice threading
ctx->thread_count = 4;                    // Use 4 threads
ctx->thread_type = FF_THREAD_SLICE;       // Slice-level parallelism

// Open codec
avcodec_open2(ctx, codec, NULL);

// Check which threading was actually used
if (ctx->active_thread_type & FF_THREAD_SLICE) {
    // Slice threading is active
}
```

#### Threading Requirements

| Threading Type | Requirements | Delay |
|----------------|--------------|-------|
| Frame | Entire frames passed to codec | N-1 frames |
| Slice | get_buffer2() thread-safe | Minimal |
| Both | Custom callbacks must be thread-safe | Variable |

### 8.2 Frame Count Handling

#### Encoder Delay

Encoders may buffer initial frames that are not immediately output:

```c
// After codec open
int encoder_delay = ctx->delay;

// For AAC, delay is typically 1024 samples (one frame)
```

#### Padding for Encoder Delay

```c
// When encoding, prepend zeros equal to encoder delay
// After encoding, discard first 'delay' samples from output
```

#### PTS/DTS Computation

```c
// For encoded packets
pkt->pts = frame->pts;                                    // Use frame PTS
pkt->dts = frame->pts;                                    // Audio: pts = dts
pkt->duration = av_rescale_q(                             // Duration in stream TB
    frame->nb_samples,
    (AVRational){1, frame->sample_rate},
    stream->time_base
);
```

### 8.3 Complete Working C Program

```c
/**
 * Complete FFmpeg Audio Encode-Decode Pipeline
 *
 * This example demonstrates:
 * 1. Opening an input audio file
 * 2. Decoding to raw PCM
 * 3. Re-encoding with a different codec
 * 4. Writing to output file
 */

#include <libavutil/opt.h>
#include <libavutil/channel_layout.h>
#include <libavutil/samplefmt.h>
#include <libavutil/frame.h>
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>
#include <libswresample/swresample.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define ERROR_BUFFER_SIZE 256

static void error_exit(const char *msg, int err) {
    char errbuf[ERROR_BUFFER_SIZE];
    av_strerror(err, errbuf, sizeof(errbuf));
    fprintf(stderr, "%s: %s\n", msg, errbuf);
    exit(1);
}

static int decode_packet(AVCodecContext *dec_ctx, AVPacket *pkt,
                         AVFrame *frame, FILE *outfile) {
    int ret;

    ret = avcodec_send_packet(dec_ctx, pkt);
    if (ret < 0) {
        fprintf(stderr, "Error sending packet to decoder: %d\n", ret);
        return ret;
    }

    while (ret >= 0) {
        ret = avcodec_receive_frame(dec_ctx, frame);
        if (ret == AVERROR(EAGAIN) || ret == AVERROR_EOF) {
            return 0;
        } else if (ret < 0) {
            fprintf(stderr, "Error receiving frame from decoder: %d\n", ret);
            return ret;
        }

        // Write PCM data (interleaved)
        int data_size = av_get_bytes_per_sample(dec_ctx->sample_fmt) *
                        frame->nb_samples *
                        dec_ctx->ch_layout.nb_channels;

        if (dec_ctx->sample_fmt == AV_SAMPLE_FMT_FLTP ||
            dec_ctx->sample_fmt == AV_SAMPLE_FMT_S16P ||
            dec_ctx->sample_fmt == AV_SAMPLE_FMT_DBLP ||
            dec_ctx->sample_fmt == AV_SAMPLE_FMT_S32P ||
            dec_ctx->sample_fmt == AV_SAMPLE_FMT_U8P ||
            dec_ctx->sample_fmt == AV_SAMPLE_FMT_S64P) {
            // Planar format - need to interleave
            uint8_t *interleaved = av_malloc(data_size);
            if (!interleaved) {
                return AVERROR(ENOMEM);
            }

            int planes = av_sample_fmt_is_planar(dec_ctx->sample_fmt) ?
                         dec_ctx->ch_layout.nb_channels : 1;
            int bps = av_get_bytes_per_sample(dec_ctx->sample_fmt);

            if (planes > 1) {
                for (int s = 0; s < frame->nb_samples; s++) {
                    for (int c = 0; c < dec_ctx->ch_layout.nb_channels; c++) {
                        memcpy(interleaved + (s * dec_ctx->ch_layout.nb_channels + c) * bps,
                               frame->extended_data[c] + s * bps,
                               bps);
                    }
                }
            }

            if (fwrite(interleaved, 1, data_size, outfile) != (size_t)data_size) {
                fprintf(stderr, "Error writing audio data\n");
                av_free(interleaved);
                return AVERROR_UNKNOWN;
            }
            av_free(interleaved);
        } else {
            // Packed format - direct write
            if (fwrite(frame->data[0], 1, data_size, outfile) != (size_t)data_size) {
                fprintf(stderr, "Error writing audio data\n");
                return AVERROR_UNKNOWN;
            }
        }

        av_frame_unref(frame);
    }

    return 0;
}

static int encode_frame(AVCodecContext *enc_ctx, AVFrame *frame,
                        AVPacket *pkt, FILE *outfile) {
    int ret;

    ret = avcodec_send_frame(enc_ctx, frame);
    if (ret < 0) {
        fprintf(stderr, "Error sending frame to encoder: %d\n", ret);
        return ret;
    }

    while (ret >= 0) {
        ret = avcodec_receive_packet(enc_ctx, pkt);
        if (ret == AVERROR(EAGAIN)) {
            return 0;
        } else if (ret == AVERROR_EOF) {
            return 0;
        } else if (ret < 0) {
            fprintf(stderr, "Error receiving packet from encoder: %d\n", ret);
            return ret;
        }

        // Write encoded packet to output
        fwrite(pkt->data, 1, pkt->size, outfile);
        av_packet_unref(pkt);
    }

    return 0;
}

int main(int argc, char **argv) {
    const char *input_filename = NULL;
    const char *output_filename = NULL;
    AVFormatContext *ifmt_ctx = NULL;
    AVFormatContext *ofmt_ctx = NULL;
    AVCodecContext *dec_ctx = NULL;
    AVCodecContext *enc_ctx = NULL;
    const AVCodec *dec_codec = NULL;
    const AVCodec *enc_codec = NULL;
    AVFrame *frame = NULL;
    AVPacket *pkt = NULL;
    AVPacket *enc_pkt = NULL;
    int ret;
    int audio_stream_idx = -1;
    AVStream *out_stream = NULL;
    FILE *pcm_outfile = NULL;
    FILE *encoded_outfile = NULL;

    if (argc < 3) {
        fprintf(stderr, "Usage: %s <input_file> <output_prefix>\n", argv[0]);
        return 1;
    }

    input_filename = argv[1];
    output_filename = argv[2];

    char pcm_filename[512];
    char encoded_filename[512];
    snprintf(pcm_filename, sizeof(pcm_filename), "%s_decoded.pcm", output_filename);
    snprintf(encoded_filename, sizeof(encoded_filename), "%s_encoded.aac", output_filename);

    // Open input file
    ret = avformat_open_input(&ifmt_ctx, input_filename, NULL, NULL);
    if (ret < 0) {
        error_exit("Could not open input file", ret);
    }

    ret = avformat_find_stream_info(ifmt_ctx, NULL);
    if (ret < 0) {
        error_exit("Could not find stream info", ret);
    }

    // Find audio stream
    for (unsigned int i = 0; i < ifmt_ctx->nb_streams; i++) {
        if (ifmt_ctx->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_AUDIO) {
            audio_stream_idx = i;
            break;
        }
    }

    if (audio_stream_idx == -1) {
        fprintf(stderr, "Could not find audio stream\n");
        return 1;
    }

    AVStream *in_stream = ifmt_ctx->streams[audio_stream_idx];

    // Find decoder
    dec_codec = avcodec_find_decoder(in_stream->codecpar->codec_id);
    if (!dec_codec) {
        fprintf(stderr, "Could not find decoder for codec %s\n",
                avcodec_get_name(in_stream->codecpar->codec_id));
        return 1;
    }

    // Allocate decoder context
    dec_ctx = avcodec_alloc_context3(dec_codec);
    if (!dec_ctx) {
        fprintf(stderr, "Could not allocate decoder context\n");
        return 1;
    }

    ret = avcodec_parameters_to_context(dec_ctx, in_stream->codecpar);
    if (ret < 0) {
        error_exit("Could not copy decoder parameters", ret);
    }

    // Open decoder
    ret = avcodec_open2(dec_ctx, dec_codec, NULL);
    if (ret < 0) {
        error_exit("Could not open decoder", ret);
    }

    // Find encoder (AAC)
    enc_codec = avcodec_find_encoder(AV_CODEC_ID_AAC);
    if (!enc_codec) {
        fprintf(stderr, "Could not find AAC encoder\n");
        return 1;
    }

    // Allocate encoder context
    enc_ctx = avcodec_alloc_context3(enc_codec);
    if (!enc_ctx) {
        fprintf(stderr, "Could not allocate encoder context\n");
        return 1;
    }

    // Configure encoder
    enc_ctx->sample_rate = dec_ctx->sample_rate;
    enc_ctx->bit_rate = 128000;
    enc_ctx->ch_layout = dec_ctx->ch_layout;
    enc_ctx->sample_fmt = AV_SAMPLE_FMT_FLTP;

    // Check if encoder supports this format
    const AVSampleFormat *p = enc_codec->sample_fmts;
    while (*p != AV_SAMPLE_FMT_NONE) {
        if (*p == enc_ctx->sample_fmt) break;
        p++;
    }
    if (*p == AV_SAMPLE_FMT_NONE) {
        enc_ctx->sample_fmt = enc_codec->sample_fmts[0];
    }

    // Open encoder
    ret = avcodec_open2(enc_ctx, enc_codec, NULL);
    if (ret < 0) {
        error_exit("Could not open encoder", ret);
    }

    // Allocate frames and packets
    frame = av_frame_alloc();
    pkt = av_packet_alloc();
    enc_pkt = av_packet_alloc();

    if (!frame || !pkt || !enc_pkt) {
        fprintf(stderr, "Could not allocate frames/packets\n");
        return 1;
    }

    // Open output files
    pcm_outfile = fopen(pcm_filename, "wb");
    if (!pcm_outfile) {
        fprintf(stderr, "Could not open PCM output file\n");
        return 1;
    }

    encoded_outfile = fopen(encoded_filename, "wb");
    if (!encoded_outfile) {
        fprintf(stderr, "Could not open encoded output file\n");
        return 1;
    }

    // Decode and write PCM
    printf("Decoding %s...\n", input_filename);
    while (av_read_frame(ifmt_ctx, pkt) >= 0) {
        if (pkt->stream_index != audio_stream_idx) {
            av_packet_unref(pkt);
            continue;
        }

        ret = decode_packet(dec_ctx, pkt, frame, pcm_outfile);
        if (ret < 0) {
            error_exit("Error decoding", ret);
        }

        av_packet_unref(pkt);
    }

    // Flush decoder
    pkt->data = NULL;
    pkt->size = 0;
    ret = decode_packet(dec_ctx, pkt, frame, pcm_outfile);
    if (ret < 0) {
        error_exit("Error flushing decoder", ret);
    }

    fclose(pcm_outfile);
    printf("Decoded PCM written to %s\n", pcm_filename);

    // Write header for AAC (ADTS)
    // ADTS header will be written by encoder

    // Reopen input PCM for encoding
    FILE *pcm_infile = fopen(pcm_filename, "rb");
    if (!pcm_infile) {
        fprintf(stderr, "Could not reopen PCM file for encoding\n");
        return 1;
    }

    printf("Encoding to AAC...\n");

    // Prepare frame for encoding
    frame->nb_samples = enc_ctx->frame_size;
    frame->format = enc_ctx->sample_fmt;
    frame->ch_layout = enc_ctx->ch_layout;

    ret = av_frame_get_buffer(frame, 0);
    if (ret < 0) {
        error_exit("Could not allocate frame buffer", ret);
    }

    // Read PCM and encode
    int frame_count = 0;
    while (1) {
        // Read samples from PCM file
        int bps = av_get_bytes_per_sample(dec_ctx->sample_fmt);
        int channels = dec_ctx->ch_layout.nb_channels;
        int samples_to_read = frame->nb_samples;

        // Read into temporary buffer first
        uint8_t *temp_buf = av_malloc(samples_to_read * channels * bps);
        if (!temp_buf) {
            fprintf(stderr, "Could not allocate temp buffer\n");
            break;
        }

        size_t bytes_read = fread(temp_buf, 1, samples_to_read * channels * bps, pcm_infile);
        int samples_read = bytes_read / (channels * bps);

        if (samples_read == 0) {
            av_free(temp_buf);
            break;
        }

        // Convert to planar format for encoder
        if (enc_ctx->sample_fmt == AV_SAMPLE_FMT_FLTP) {
            float *out_left = (float *)frame->data[0];
            float *out_right = (float *)frame->data[1];
            int16_t *in = (int16_t *)temp_buf;

            for (int i = 0; i < samples_read; i++) {
                out_left[i] = in[i * 2] / 32768.0f;
                out_right[i] = in[i * 2 + 1] / 32768.0f;
            }

            // Zero-pad remaining samples
            memset(out_left + samples_read, 0, (frame->nb_samples - samples_read) * sizeof(float));
            memset(out_right + samples_read, 0, (frame->nb_samples - samples_read) * sizeof(float));
        }

        av_free(temp_buf);

        frame->nb_samples = samples_read;

        ret = encode_frame(enc_ctx, frame, enc_pkt, encoded_outfile);
        if (ret < 0) {
            fprintf(stderr, "Error encoding frame %d\n", frame_count);
            break;
        }

        frame_count++;
        frame->nb_samples = enc_ctx->frame_size;  // Reset for next frame
    }

    // Flush encoder
    ret = encode_frame(enc_ctx, NULL, enc_pkt, encoded_outfile);
    if (ret < 0) {
        fprintf(stderr, "Error flushing encoder\n");
    }

    printf("Encoded %d frames to %s\n", frame_count, encoded_filename);

    // Cleanup
    fclose(pcm_infile);
    fclose(encoded_outfile);

    av_frame_free(&frame);
    av_packet_free(&pkt);
    av_packet_free(&enc_pkt);
    avcodec_free_context(&dec_ctx);
    avcodec_free_context(&enc_ctx);
    avformat_close_input(&ifmt_ctx);

    printf("Done!\n");
    return 0;
}
```

---

## 9. SEEKING & TIMESTAMP HANDLING

### 9.1 PTS / DTS Model

In audio, the relationship between PTS and DTS is typically simple:

- For audio: **PTS = DTS** (audio frames are decoded in order)
- For video: PTS may differ from DTS (B-frame reordering)

#### Accessing Timestamps

```c
// From decoded frame
int64_t pts = frame->pts;

// Best effort timestamp (handles edge cases)
int64_t best_effort = av_frame_get_best_effort_timestamp(frame);

// Alternative: calculate from packet
int64_t packet_pts = pkt->pts;
int64_t packet_dts = pkt->dts;

// Convert to seconds
double pts_seconds = pts * av_q2d(stream->time_base);
```

#### Duration Calculation

```c
// Frame duration from sample count
int frame_duration_samples = frame->nb_samples;

// Duration in stream timebase
int64_t frame_duration_tb = av_rescale_q(
    frame->nb_samples,
    (AVRational){1, frame->sample_rate},
    stream->time_base
);

// Duration in seconds
double frame_duration_sec = (double)frame->nb_samples / frame->sample_rate;
```

### 9.2 Seeking Considerations

#### Basic Seek Operation

```c
// Seek to timestamp in stream
int64_t target_ts = av_rescale_q(
    target_seconds * AV_TIME_BASE,  // Target in AV_TIME_BASE_Q units
    AV_TIME_BASE_Q,
    stream->time_base
);

ret = av_seek_frame(fmt_ctx, audio_stream_index, target_ts, AVSEEK_FLAG_BACKWARD);
if (ret < 0) {
    // Handle seek error
}

// Flush decoder after seek
avcodec_flush_buffers(dec_ctx);
```

#### Post-Seek Handling

```c
void handle_seek(AVFormatContext *fmt_ctx, AVCodecContext *dec_ctx,
                 int stream_index, int64_t seek_ts) {
    // Perform seek
    int ret = av_seek_frame(fmt_ctx, stream_index, seek_ts, AVSEEK_FLAG_BACKWARD);
    if (ret < 0) {
        return ret;
    }

    // Flush codec buffers
    avcodec_flush_buffers(dec_ctx);

    // Discard packets before seek point
    AVPacket *pkt = av_packet_alloc();
    while (1) {
        ret = av_read_frame(fmt_ctx, pkt);
        if (ret < 0) break;
        if (pkt->stream_index == stream_index) {
            if (pkt->pts < seek_ts) {
                av_packet_unref(pkt);
            } else {
                break;
            }
        } else {
            av_packet_unref(pkt);
        }
    }
    av_packet_free(&pkt);

    return 0;
}
```

#### Timestamp Rescaling

```c
// Convert between timebases
int64_t av_rescale_q(int64_t a, AVRational bq, AVRational cq);

// Example: Rescale audio timestamp to video timebase
int64_t audio_pts_in_video_tb = av_rescale_q(
    audio_frame->pts,
    audio_stream->time_base,
    video_stream->time_base
);

// Example: Convert seconds to stream timestamp
int64_t seconds_to_pts(double seconds, AVStream *stream) {
    return av_rescale_q(
        seconds * AV_TIME_BASE,
        AV_TIME_BASE_Q,
        stream->time_base
    );
}
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

### Latency in the Send/Receive Model

The send/receive API introduces buffering at the codec level:

| Operation | Typical Latency |
|-----------|----------------|
| Encoder buffering | 1-2 frames |
| Decoder buffering | 0-1 frames |
| Resampler buffering | Variable |

#### Minimizing Latency

```c
// For low-latency encoding
ctx->flags |= AV_CODEC_FLAG_LOW_DELAY;

// Disable frame threading (adds delay)
ctx->thread_count = 1;

// Use smallest possible frame size
if (codec->capabilities & AV_CODEC_CAP_VARIABLE_FRAME_SIZE) {
    ctx->frame_size = 256;  // Minimum for many codecs
}
```

### Buffer Sizing for Real-Time

```c
// Calculate buffer requirements for real-time capture
int calculate_buffer_size(int sample_rate, int channels,
                          enum AVSampleFormat format) {
    int bytes_per_sample = av_get_bytes_per_sample(format);
    int samples_per_ms = sample_rate / 1000;
    int buffer_samples = samples_per_ms * 10;  // 10ms buffer
    return buffer_samples * channels * bytes_per_sample;
}

// Ensure adequate buffering
int required_samples = swr_get_out_samples(swr, input_samples);
int output_buffer = av_rescale_rnd(
    required_samples + swr_get_delay(swr, input_sample_rate),
    output_sample_rate,
    input_sample_rate,
    AV_ROUND_UP
);
```

### avcodec_send_packet Ordering Guarantees

The send/receive API provides strict ordering guarantees:

1. **Order preservation**: Frames are decoded in packet order
2. **Output order**: Frames are output in presentation order
3. **No reordering for audio**: Unlike video, audio maintains order

---

## 11. PLATFORM & BUILD NOTES

### 11.1 Build Configuration

#### Required ./configure Flags

```bash
# Basic audio support (always enabled)
./configure

# Additional audio codecs
./configure \
    --enable-libfdk-aac \      # High-quality AAC encoder
    --enable-libmp3lame \      # MP3 encoding
    --enable-libvorbis \      # Vorbis encoder
    --enable-libopus \        # Opus encoder
    --enable-libflac \        # FLAC encoder
    --enable-nonfree \        # Required for libfdk-aac
    --enable-gpl              # Required for some encoders
```

#### Codec Support Flags

| Flag | Codec | License |
|------|-------|---------|
| (built-in) | PCM, MP3, AAC, AC3 | Various |
| --enable-libfdk-aac | AAC-LC/HE | Non-free |
| --enable-libmp3lame | MP3 | LGPL |
| --enable-libvorbis | Vorbis | BSD |
| --enable-libopus | Opus | BSD |
| --enable-libflac | FLAC | BSD |

### 11.2 Compiler & Linker

#### Required Header Files

```c
#include <libavcodec/avcodec.h>     // Main codec API
#include <libavutil/frame.h>        // AVFrame
#include <libavutil/buffer.h>       // Buffer management
#include <libavutil/channel_layout.h> // Channel layouts
#include <libavutil/samplefmt.h>     // Sample formats
#include <libavutil/dict.h>          // Metadata
#include <libavutil/error.h>         // Error codes
```

#### Linker Flags

```bash
# Using pkg-config
pkg-config --cflags --libs libavcodec libavutil libswresample

# Manual linking
-lavcodec -lavutil -lswresample

# Static linking (all dependencies)
-pthread -lm -lpthread -ldl
```

#### pkg-config Names

| Library | pkg-config Name |
|---------|-----------------|
| libavcodec | libavcodec |
| libavutil | libavutil |
| libswresample | libswresample |
| libavformat | libavformat |
| libavfilter | libavfilter |
| libavdevice | libavdevice |
| libavresample | libavresample |

---

## 12. VERSIONING & API CHANGES

### API Deprecation Timeline

| Version | Date | Changes |
|---------|------|---------|
| 0.1 | 2000 | Initial API |
| 0.5 | 2004 | Major redesign |
| 0.6 | 2006 | Frame refactoring |
| 1.0 | 2012 | AVFrame redesign |
| 2.0 | 2013 | AVFrame improvements |
| 3.0 | 2015 | Memory management |
| 3.4 | 2017 | Legacy API deprecated |
| **4.0** | **2018** | **Send/receive API introduced** |
| 4.4 | 2021 | Channel layout API |
| 6.0 | 2023 | Enhanced channel layout |
| 7.0 | 2024 | Current stable |

### Legacy API Migration

**Before (FFmpeg 3.x and earlier):**

```c
int got_frame;
while (av_read_frame(fmt_ctx, pkt) >= 0) {
    ret = avcodec_decode_audio4(ctx, frame, &got_frame, pkt);
    if (got_frame) {
        // Process frame
    }
    av_packet_unref(pkt);
}
```

**After (FFmpeg 4.0+):**

```c
while (av_read_frame(fmt_ctx, pkt) >= 0) {
    ret = avcodec_send_packet(ctx, pkt);
    if (ret < 0) {
        // Handle error
    }
    av_packet_unref(pkt);

    while (1) {
        ret = avcodec_receive_frame(ctx, frame);
        if (ret == AVERROR(EAGAIN)) break;
        if (ret == AVERROR_EOF) break;
        if (ret < 0) goto error;
        // Process frame
        av_frame_unref(frame);
    }
}
```

### Version Information

```c
// Get library version
unsigned avformat_version(void);
unsigned avcodec_version(void);
unsigned avutil_version(void);

// Get version as string
const char *avformat_configuration(void);
const char *avformat_license(void);

// Version macros
#define AV_VERSION_INT(a, b, c) (((a)<<16) | ((b)<<8) | (c))
#define AV_VERSION_DOT(a, b, c) a.b.c

// Check version
#if LIBAVCODEC_VERSION_INT < AV_VERSION_INT(4, 0, 0)
#error "FFmpeg 4.0 or later required"
#endif
```

---

## 13. COMPLETE FUNCTION REFERENCE

### Codec Discovery Functions

| Function | Parameters | Returns | Notes |
|----------|------------|---------|-------|
| `avcodec_find_encoder` | enum AVMediaType type, enum AVCodecID id | AVCodec* | Find encoder by codec ID |
| `avcodec_find_decoder` | enum AVCodecID id | AVCodec* | Find decoder by codec ID |
| `avcodec_find_encoder_by_name` | const char *name | AVCodec* | Find encoder by name string |
| `avcodec_find_decoder_by_name` | const char *name | AVCodec* | Find decoder by name string |
| `av_codec_iterate` | void *opaque | const AVCodec* | Iterate all registered codecs |

### Context Allocation & Setup Functions

| Function | Parameters | Returns | Notes |
|----------|------------|---------|-------|
| `avcodec_alloc_context3` | const AVCodec *codec | AVCodecContext* | Allocate codec context |
| `avcodec_free_context` | AVCodecContext **avctx | void | Free codec context |
| `avcodec_open2` | AVCodecContext *avctx, const AVCodec *codec, AVDictionary **options | int | Open codec |
| `avcodec_parameters_to_context` | AVCodecContext *codec, const AVCodecParameters *par | int | Copy codec parameters |
| `avcodec_parameters_from_context` | AVCodecParameters *par, const AVCodecContext *codec | int | Extract parameters |
| `avcodec_close` | AVCodecContext *avctx | int | Close codec (deprecated) |

### Encoder Functions

| Function | Parameters | Returns | Notes |
|----------|------------|---------|-------|
| `avcodec_send_frame` | AVCodecContext *avctx, const AVFrame *frame | int | Send frame to encoder |
| `avcodec_receive_packet` | AVCodecContext *avctx, AVPacket *avpkt | int | Receive encoded packet |
| `avcodec_encode_audio2` | AVCodecContext *avctx, AVPacket *avpkt, const AVFrame *frame, int *got_packet_ptr | int | **Deprecated** encoder |

### Decoder Functions

| Function | Parameters | Returns | Notes |
|----------|------------|---------|-------|
| `avcodec_send_packet` | AVCodecContext *avctx, const AVPacket *avpkt | int | Send packet to decoder |
| `avcodec_receive_frame` | AVCodecContext *avctx, AVFrame *frame | int | Receive decoded frame |
| `avcodec_receive_frame_flags` | AVCodecContext *avctx, AVFrame *frame, unsigned flags | int | Receive with flags |
| `avcodec_decode_audio4` | AVCodecContext *avctx, AVFrame *frame, int *got_frame_ptr, const AVPacket *avpkt | int | **Deprecated** decoder |
| `avcodec_flush_buffers` | AVCodecContext *avctx | void | Flush codec buffers |

### Frame Functions

| Function | Parameters | Returns | Notes |
|----------|------------|---------|-------|
| `av_frame_alloc` | void | AVFrame* | Allocate frame structure |
| `av_frame_free` | AVFrame **frame | void | Free frame and buffers |
| `av_frame_ref` | AVFrame *dst, const AVFrame *src | int | Create frame reference |
| `av_frame_unref` | AVFrame *frame | void | Unref frame buffers |
| `av_frame_make_writable` | AVFrame *frame | int | Ensure frame is writable |
| `av_frame_get_buffer` | AVFrame *frame, int align | int | Allocate frame buffers |
| `av_frame_clone` | const AVFrame *frame | AVFrame* | Create frame copy |
| `av_frame_new_side_data` | AVFrame *frame, enum AVFrameSideDataType type, int size | AVFrameSideData* | Add side data |
| `av_frame_get_side_data` | const AVFrame *frame, enum AVFrameSideDataType type | AVFrameSideData* | Get side data |

### Packet Functions

| Function | Parameters | Returns | Notes |
|----------|------------|---------|-------|
| `av_packet_alloc` | void | AVPacket* | Allocate packet |
| `av_packet_free` | AVPacket **pkt | void | Free packet |
| `av_init_packet` | AVPacket *pkt | void | Initialize packet |
| `av_packet_ref` | AVPacket *dst, const AVPacket *src | int | Create packet reference |
| `av_packet_unref` | AVPacket *pkt | void | Unref packet data |
| `av_packet_clone` | const AVPacket *src | AVPacket* | Clone packet |
| `av_packet_make_refcounted` | AVPacket *pkt | int | Ensure refcounted |
| `av_packet_move_ref` | AVPacket *dst, AVPacket *src | void | Move packet reference |

### Sample Format Functions

| Function | Parameters | Returns | Notes |
|----------|------------|---------|-------|
| `av_get_sample_fmt_name` | enum AVSampleFormat sample_fmt | const char* | Get format name |
| `av_get_sample_fmt` | const char *name | enum AVSampleFormat | Get format from name |
| `av_get_packed_sample_fmt` | enum AVSampleFormat sample_fmt | enum AVSampleFormat | Get packed alternative |
| `av_get_planar_sample_fmt` | enum AVSampleFormat sample_fmt | enum AVSampleFormat | Get planar alternative |
| `av_sample_fmt_is_planar` | enum AVSampleFormat sample_fmt | int | Check if planar |
| `av_get_bytes_per_sample` | enum AVSampleFormat sample_fmt | int | Bytes per sample |
| `av_samples_alloc` | uint8_t **audio_data, int *linesize, int nb_channels, int nb_samples, enum AVSampleFormat sample_fmt, int align | int | Allocate sample buffer |
| `av_samples_fill_arrays` | uint8_t **audio_data, int *linesize, const uint8_t *src, int nb_channels, int nb_samples, enum AVSampleFormat sample_fmt, int align | int | Fill from source |
| `av_samples_alloc_array_and_samples` | uint8_t ***audio_data, int *linesize, int nb_channels, int nb_samples, enum AVSampleFormat sample_fmt, int align | int | Allocate array and buffers |

### Channel Layout Functions

| Function | Parameters | Returns | Notes |
|----------|------------|---------|-------|
| `av_channel_layout_default` | AVChannelLayout *ch_layout, int nb_channels | void | Get default layout |
| `av_channel_layout_from_mask` | AVChannelLayout *ch_layout, uint64_t mask | int | Initialize from mask |
| `av_channel_layout_from_string` | AVChannelLayout *ch_layout, const char *str | int | Parse from string |
| `av_channel_layout_uninit` | AVChannelLayout *ch_layout | void | Free layout resources |
| `av_channel_layout_copy` | AVChannelLayout *dst, const AVChannelLayout *src | int | Copy layout |
| `av_channel_layout_check` | const AVChannelLayout *ch_layout | int | Validate layout |
| `av_channel_layout_compare` | const AVChannelLayout *a, const AVChannelLayout *b | int | Compare layouts |
| `av_channel_name` | enum AVChannel channel | const char* | Get channel name |
| `av_channel_description` | enum AVChannel channel | const char* | Get channel description |
| `av_channel_from_string` | const char *name | enum AVChannel | Get channel from name |

### SwrContext Functions

| Function | Parameters | Returns | Notes |
|----------|------------|---------|-------|
| `swr_alloc` | void | SwrContext* | Allocate context |
| `swr_alloc_set_opts2` | SwrContext **ps, const AVChannelLayout *out_ch_layout, enum AVSampleFormat out_sample_fmt, int out_sample_rate, const AVChannelLayout *in_ch_layout, enum AVSampleFormat in_sample_fmt, int in_sample_rate, int log_offset, void *log_ctx | int | Allocate and configure |
| `swr_init` | SwrContext *s | int | Initialize context |
| `swr_is_initialized` | SwrContext *s | int | Check if initialized |
| `swr_free` | SwrContext **s | void | Free context |
| `swr_close` | SwrContext *s | void | Close context |
| `swr_convert` | SwrContext *s, uint8_t *const *out, int out_count, const uint8_t *const *in, int in_count | int | Convert samples |
| `swr_get_delay` | SwrContext *s, int64_t base | int64_t | Get conversion delay |
| `swr_get_out_samples` | SwrContext *s, int in_samples | int | Calculate output size |
| `swr_set_channel_mapping` | SwrContext *s, const int *channel_map | int | Set channel mapping |
| `swr_set_matrix` | SwrContext *s, const double *matrix, int stride | int | Set mixing matrix |
| `swr_convert_frame` | SwrContext *swr, AVFrame *output, const AVFrame *input | int | Convert using frames |
| `swr_config_frame` | SwrContext *swr, const AVFrame *out, const AVFrame *in | int | Configure from frames |

### Utility Functions

| Function | Parameters | Returns | Notes |
|----------|------------|---------|-------|
| `av_strerror` | int errnum, char *errbuf, size_t errbuf_size | int | Get error string |
| `av_err2str` | int errnum | char[] | Macro for error string |
| `av_rescale_q` | int64_t a, AVRational bq, AVRational cq | int64_t | Rescale with timebase |
| `av_rescale_rnd` | int64_t a, int64_t b, int64_t c, enum AVRounding rnd | int64_t | Rescale with rounding |
| `av_q2d` | AVRational a | double | Rational to double |

### AVDictionary Functions

| Function | Parameters | Returns | Notes |
|----------|------------|---------|-------|
| `av_dict_set` | AVDictionary **pm, const char *key, const char *value, int flags | int | Set entry |
| `av_dict_set_int` | AVDictionary **pm, const char *key, int64_t value, int flags | int | Set integer entry |
| `av_dict_get` | const AVDictionary *m, const char *key, const AVDictionaryEntry *prev, int flags | AVDictionaryEntry* | Get entry |
| `av_dict_iterate` | const AVDictionary *m, const AVDictionaryEntry *prev | AVDictionaryEntry* | Iterate entries |
| `av_dict_free` | AVDictionary **m | void | Free dictionary |
| `av_dict_copy` | AVDictionary **dst, const AVDictionary *src, int flags | int | Copy dictionary |
| `av_dict_get_string` | const AVDictionary *m, char **buffer, const char key_val_sep, const char pairs_sep | int | Get as string |
| `av_dict_parse_string` | AVDictionary **pm, const char *str, const char *key_val_sep, const char *pairs_sep, int flags | int | Parse from string |

---

## 14. IMPLEMENTATION CHECKLIST (for the Converter Developer)

### Pre-Implementation Verification

- [ ] Verify FFmpeg is installed: `ffmpeg -version`
- [ ] Check available codecs: `ffmpeg -codecs | grep -i audio`
- [ ] Confirm pkg-config is available: `pkg-config --modversion libavcodec`

### FFmpeg Build Flags

- [ ] `--enable-libavcodec` (default)
- [ ] `--enable-libswresample` (for sample rate conversion)
- [ ] `--enable-libfdk-aac` (if using high-quality AAC)
- [ ] `--enable-libmp3lame` (if encoding MP3)
- [ ] `--enable-libvorbis` (if encoding Vorbis)
- [ ] `--enable-libopus` (if encoding Opus)
- [ ] `--enable-nonfree` (required for libfdk-aac)
- [ ] `--enable-gpl` (required for some encoders)

### Encoder Initialization Sequence

1. [ ] `avcodec_find_encoder()` or `avcodec_find_encoder_by_name()`
2. [ ] `avcodec_alloc_context3(codec)`
3. [ ] Set `codec_context->sample_rate`
4. [ ] Set `codec_context->bit_rate` or `codec_context->global_quality`
5. [ ] Set `codec_context->ch_layout`
6. [ ] Set `codec_context->sample_fmt`
7. [ ] `avcodec_open2(codec_context, codec, NULL)`
8. [ ] Record `codec_context->frame_size`

### Decoder Initialization Sequence

1. [ ] `avcodec_find_decoder()` or from `AVStream->codecpar`
2. [ ] `avcodec_alloc_context3(codec)`
3. [ ] Copy parameters: `avcodec_parameters_to_context()`
4. [ ] `avcodec_open2(codec_context, codec, NULL)`
5. [ ] Allocate `AVPacket` with `av_packet_alloc()`
6. [ ] Allocate `AVFrame` with `av_frame_alloc()`

### Frame Size Handling

- [ ] After `avcodec_open2()`, read `codec_context->frame_size`
- [ ] Set `frame->nb_samples = frame_size` for encoding
- [ ] For final frame, `frame->nb_samples < frame_size` is acceptable
- [ ] If `AV_CODEC_CAP_VARIABLE_FRAME_SIZE`, any size is allowed
- [ ] Allocate frame buffers: `av_frame_get_buffer(frame, 0)`

### Sample Format Conversion

- [ ] Use `av_sample_fmt_is_planar()` to check format type
- [ ] For planar formats, fill `frame->extended_data[c][s]` for each channel
- [ ] For packed formats, interleaved data goes in `frame->data[0]`
- [ ] Use `swr_convert()` for format conversion
- [ ] Check `swr_get_delay()` to determine output buffer size

### Channel Layout Handling

- [ ] Use `AVChannelLayout` struct (not deprecated `channels` field)
- [ ] Initialize with predefined macros: `AV_CHANNEL_LAYOUT_STEREO`
- [ ] Or use `av_channel_layout_from_mask()` for custom layouts
- [ ] Use `av_channel_layout_default()` for default based on channel count
- [ ] Free with `av_channel_layout_uninit()` when done

### Memory Management Sequence

```
Encode:
  1. av_frame_alloc()
  2. av_packet_alloc()
  3. [Loop] av_frame_unref() each iteration
  4. avcodec_send_frame()
  5. avcodec_receive_packet()
  6. [Loop] av_packet_unref()
  7. av_frame_free()
  8. av_packet_free()
  9. avcodec_free_context()

Decode:
  1. av_packet_alloc()
  2. av_frame_alloc()
  3. [Loop] av_read_frame()
  4. avcodec_send_packet()
  5. avcodec_receive_frame()
  6. [Loop] av_frame_unref()
  7. av_packet_unref()
  8. av_frame_free()
  9. av_packet_free()
  10. avcodec_free_context()
```

### Error Recovery

- [ ] Check all function return values for `< 0`
- [ ] Handle `AVERROR(EAGAIN)`: send more input OR receive more output
- [ ] Handle `AVERROR_EOF`: end of stream, stop processing
- [ ] Handle `AVERROR(ENOMEM)`: memory allocation failed
- [ ] Handle `AVERROR(EINVAL)`: invalid argument or codec state
- [ ] Log errors with `av_strerror()` for debugging

### Seeking Support

- [ ] Use `av_seek_frame()` for timestamp-based seeking
- [ ] Always call `avcodec_flush_buffers()` after seeking
- [ ] Rescale timestamps with `av_rescale_q()`
- [ ] Handle partial frames at seek boundaries

### Thread Safety Verification

- [ ] Frame threading: `thread_count > 1` + `thread_type = FF_THREAD_FRAME`
- [ ] Slice threading: `thread_count > 1` + `thread_type = FF_THREAD_SLICE`
- [ ] Verify codec supports threading: check `codec->capabilities & AV_CODEC_CAP_*`
- [ ] For custom `get_buffer2()`, ensure thread-safe implementation
- [ ] Reference counting is thread-safe for separate buffers

---

## Quick Reference: Common Patterns

### Minimal Decode Loop

```c
while (av_read_frame(fmt_ctx, pkt) >= 0) {
    avcodec_send_packet(ctx, pkt);
    av_packet_unref(pkt);
    while (!avcodec_receive_frame(ctx, frame)) {
        process(frame);
        av_frame_unref(frame);
    }
}
```

### Minimal Encode Loop

```c
while (have_input) {
    fill(frame);
    avcodec_send_frame(ctx, frame);
    av_frame_unref(frame);
    while (!avcodec_receive_packet(ctx, pkt)) {
        write(pkt);
        av_packet_unref(pkt);
    }
}
// Drain
avcodec_send_frame(ctx, NULL);
while (!avcodec_receive_packet(ctx, pkt)) {
    write(pkt);
    av_packet_unref(pkt);
}
```

### Minimal Resampling

```c
swr_alloc_set_opts2(&swr, &out_layout, out_fmt, out_rate,
                    &in_layout, in_fmt, in_rate, 0, NULL);
swr_init(swr);
swr_convert(swr, out_buf, out_samples, in_buf, in_samples);
swr_free(&swr);
```
