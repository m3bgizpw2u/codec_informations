# FFmpeg libavformat Muxing and Demuxing — Deep Technical Reference

> **Category:** FFmpeg API
> **File Extensions:** N/A (API)
> **MIME Types:** N/A
> **Standardization Body:** FFmpeg Project
> **Specification Document:** https://ffmpeg.org/ffmpeg-formats.html
> **Patent Status:** N/A
> **License:** LGPL 2.1+

---

## 1. HISTORICAL CONTEXT & ORIGIN

libavformat is the multimedia container (demuxer/muxer) library within the FFmpeg project, a free software project initiated by Fabrice Bellard in 2000. The project grew from an earlier effort called "FFmpeg" (Fast Forward MPEG) and quickly became the de facto open-source standard for audio and video format handling. The library was designed to provide a unified, extensible abstraction layer for reading and writing media containers, independent of the underlying codec logic (which lives in libavcodec). The architecture separates concerns cleanly: libavcodec handles compression/decompression of individual streams, while libavformat manages the container format — the structural wrapper that holds those streams together with timing, metadata, and synchronization information.

The FFmpeg project is maintained under the LGPL 2.1+ license, with contributing organizations including MPlayer, Videolan, and numerous individual developers. Over two decades, libavformat has grown to support over 300 demuxers (input formats) and 300 muxers (output formats), encompassing essentially every media container format in common use: MP4, MKV, FLAC, OGG, WAV, AVI, MOV, WebM, MPEG-TS, and many others. The API has remained remarkably stable while gaining capabilities, making FFmpeg-based applications highly portable across platforms and reliable for long-term use.

The design philosophy emphasizes generality over specificity. Rather than building one-off parsers for each format, the library defines a set of abstract data structures (`AVFormatContext`, `AVStream`, `AVPacket`, `AVFrame`, `AVIOContext`) that represent all formats uniformly. Each format's demuxer and muxer implements a common interface, registering itself with a global table at startup. This plugin-like architecture means that adding support for a new format requires implementing the interface without modifying any central code.

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

libavformat provides the I/O and container abstraction layer in the FFmpeg architecture. The architecture follows a layered model:

```
┌─────────────────────────────────────────────┐
│           Application Layer                  │
│   (your code using libavformat/libavcodec)  │
├─────────────────────────────────────────────┤
│            libavformat                       │
│  ┌─────────────┐  ┌─────────────────────┐  │
│  │  Demuxers   │  │       Muxers         │  │
│  │  (parsers)  │  │    (writers)         │  │
│  └─────────────┘  └─────────────────────┘  │
│  ┌─────────────────────────────────────────┐│
│  │     AVIOContext / URLProtocol           ││
│  │  (I/O abstraction: file, http, rtmp)   ││
│  └─────────────────────────────────────────┘│
├─────────────────────────────────────────────┤
│            libavcodec                        │
│    (encoding / decoding of individual       │
│     audio/video streams)                    │
├─────────────────────────────────────────────┤
│            libavutil                         │
│    (common utilities: buffers, math, etc.)  │
└─────────────────────────────────────────────┘
```

The key architectural principles are:

1. **Format independence:** All containers are accessed through the same API. Switching from MP4 to Matroska input requires only changing the filename — the API calls remain identical.

2. **Lazy initialization:** Format contexts are allocated and populated incrementally. The header is read only when `avformat_open_input` is called. Codec probing happens only when `avformat_find_stream_info` is called, and only if necessary.

3. **Separation of muxing and demuxing:** These are distinct operations. Demuxing reads a container to produce packets; muxing receives packets and writes them into a container. The two are never mixed in a single pipeline.

4. **Reference-counted packet management:** `AVPacket` uses reference counting for its data buffer. This prevents unnecessary copies and enables zero-copy sharing between streams (e.g., when muxing video and audio from the same source without re-encoding).

5. **AVIOContext abstraction:** All I/O goes through a buffered I/O layer. This layer can be backed by a file, a memory buffer, a network socket, a custom callback, or a protocol handler (http, rtmp, pipe, etc.).

6. **Automatic format detection:** If the input format (`fmt`) is not specified, libavformat probes the input by reading the first few kilobytes and matching against known magic bytes and layout patterns. The probe buffer is scored against each format's probe function.

7. **Timestamp management:** Every packet carries presentation timestamp (`pts`) and decompression timestamp (`dts`) in the container's time base. The library provides utilities for converting between time bases and for handling timestamp wrap-around (when a 33-bit PTS counter rolls over in MPEG-TS).

---

## 3. CORE DATA STRUCTURES

### 3.1 AVFormatContext

`AVFormatContext` is the top-level structure representing a container. It holds everything about an open media file or stream: all streams, global metadata, the I/O context, and format-specific options.

```c
typedef struct AVFormatContext {
    const AVClass *av_class;
    struct AVInputFormat *iformat;  // NULL for output
    struct AVOutputFormat *oformat; // NULL for input
    void *priv_data;                // format-specific private data
    AVIOContext *pb;                // I/O context
    int ctx_flags;
    unsigned int nb_streams;         // number of streams
    AVStream **streams;              // array of stream pointers
    char filename[1024];
    int64_t start_time;             // first timestamp in microseconds
    int64_t duration;               // total duration in microseconds
    int64_t bit_rate;               // total bitrate in bits/second
    unsigned int packet_size;
    int max_delay;
    int flags;
    int64_t probesize;
    int64_t max_analyze_duration;
    const uint8_t *key;
    int key_len;
    unsigned int nb_programs;
    AVProgram **programs;
    AVDiscard *video_codec_id_override;
    struct AVCodec *video_codec;
    struct AVCodec *audio_codec;
    struct AVCodec *subtitle_codec;
    int64_t max_index_probe;        // max frames to analyze before aborting
    int error_recognition;
    void (*interrupt_callback)(void);
    AVDictionary *metadata;         // global metadata
    // ... additional fields in newer API versions
} AVFormatContext;
```

Key fields for demuxing: `iformat` (input format), `pb` (I/O context), `streams` array, `duration`, `bit_rate`, `metadata`.

Key fields for muxing: `oformat` (output format), `pb` (output I/O context), `streams` array.

`priv_data` is a pointer to format-specific internal state. For example, when opening an MP4 file, `priv_data` points to an internal `MOVContext` structure that holds MP4-specific state like the atom parsing tree, edit lists, and chunk offsets.

`pb` is the `AVIOContext` — the buffered I/O layer. Every `AVFormatContext` has exactly one `AVIOContext` (or `NULL` for some special cases). This context owns the buffering and read/write callbacks.

`ctx_flags` controls various behaviors: `AVFMT_FLAG_GENPTS` (generate PTS if missing), `AVFMT_FLAG_IGNIDX` (ignore corrupted index), `AVFMT_FLAG_NONBLOCK` (don't block on I/O), `AVFMT_FLAG_IGNDTS` (ignore DTS), `AVFMT_FLAG_NOFILLIN` (don't fill in missing values), `AVFMT_FLAG_NOPARSE` (don't parse — use with custom parsers), `AVFMT_FLAG_CUSTOM_IO` (AVIOContext is user-provided), `AVFMT_FLAG_DISCARD_CORRUPT` (discard corrupted packets).

`interrupt_callback` enables cooperative cancellation. Set this to a callback that checks a flag or pipe. If the callback returns non-zero, blocking operations abort with `AVERROR_EXIT`.

`probesize` (default 5 MB) controls how many bytes the format probe reads before deciding on a format. `max_analyze_duration` (default 5 seconds) controls how long `avformat_find_stream_info` spends analyzing frames. Increase these for very large files with low-bitrate streams that need more data to determine codec parameters.

### 3.2 AVStream

`AVStream` represents a single media stream within a container — one audio track, one video track, one subtitle track, etc.

```c
typedef struct AVStream {
    int index;           // stream index in the parent AVFormatContext.streams
    int id;              // format-specific stream ID
    AVCodecContext *codec;  // DEPRECATED: use codecpar instead
    void *priv_data;    // format-specific private data
    int64_t first_dts;  // first DTS value seen
    int64_t start_time; // first timestamp in microseconds (container time base)
    int64_t duration;   // duration in stream time base
    int64_t nb_frames;  // total number of frames (may be 0 if unknown)
    int disposition;
    AVDiscard discard;   // demuxer discard level
    AVRational sample_aspect_ratio;
    int64_t pts_wrap_bits;
    int64_t cur_dts;
    int64_t last_IP_pts;
    int last_IP_duration;
    int probe_packets;
    int codec_info_nb_frames;
    AVRational time_base;    // the time base for this stream
                            // e.g., {1, 90000} for video, {1, 48000} for audio
    int pts_correction_num_faulty_pts;
    int pts_correction_last_dts;
    int64_t reference_dts;
    int64_t avg_frame_rate;     // measured/smoothed frame rate
    AVPacket *cur_pkt;          // private: current packet being processed
    AVCodecParameters *codecpar;  // codec parameters extracted from the container
    // ... additional fields
} AVStream;
```

The most important field is `codecpar` (added in FFmpeg 4.0, replacing the deprecated `codec` pointer). This `AVCodecParameters` struct holds all codec-agnostic information about the stream: codec type, codec ID, bitrate, sample rate, channel count, frame size, etc. The separation of `codecpar` from the `AVCodecContext` reflects a deliberate design: the container's metadata about a stream does not depend on whether a codec is actually opened for decoding.

`time_base` is critical. For video, it is commonly `{1, 90000}` (the MPEG/PTS time base, based on a 90 kHz clock) or `{1, 1000}`. For audio, it equals the sample rate: `{1, 48000}` for 48 kHz audio. All timestamps in `AVPacket` (`pts`, `dts`, `duration`) are expressed in this time base and must be scaled using `av_rescale_q` to convert to/from other time bases.

`first_dts` records the DTS of the first packet and is used to compute the initial presentation offset.

`disposition` encodes stream flags: `AV_DISPOSITION_DEFAULT`, `AV_DISPOSITION_DUB`, `AV_DISPOSITION_ORIGINAL`, `AV_DISPOSITION_COMMENT`, `AV_DISPOSITION_LYRICS`, `AV_DISPOSITION_KARAOKE`, `AV_DISPOSITION_FORCED`, `AV_DISPOSITION_HEARING_IMPAIRED`, `AV_DISPOSITION_VISUAL_IMPAIRED`, `AV_DISPOSITION_CLEAN_EFFECTS`, `AV_DISPOSITION_ATTACHED_PIC`, `AV_DISPOSITION_NON_STREAMING`, `AV_DISPOSITION_CANONICAL`.

The `discard` field controls which packets are returned: `AVDISCARD_NONE` (keep all), `AVDISCARD_DEFAULT` (skip some), `AVDISCARD_NONKEY` (skip non-keyframes), `AVDISCARD_ALL` (skip everything).

### 3.3 AVCodecParameters

`AVCodecParameters` stores all codec-specific metadata that describes the stream's encoding parameters. This struct is codec-agnostic — it contains fields common to all codecs, and the meaning of each field depends on the `codec_type`.

```c
typedef struct AVCodecParameters {
    enum AVMediaType codec_type;   // audio, video, subtitle, data, attachment
    enum AVCodecID   codec_id;    // e.g., AV_CODEC_ID_AAC, AV_CODEC_ID_FLAC
    uint32_t         codec_tag;   // fourcc for legacy containers
    uint8_t         *extradata;  // codec-specific initialization data
    int              extradata_size;
    int64_t          bit_rate;
    int              bits_per_coded_sample;
    int              bits_per_raw_sample;
    int              profile;
    int              level;
    int              width;       // video: width in pixels
    int              height;      // video: height in pixels
    int              sample_aspect_ratio_num;
    int              sample_aspect_ratio_den;
    enum AVPixelFormat pix_fmt;   // video: pixel format
    enum AVColorSpace colorspace; // video: color space
    enum AVColorRange color_range;
    enum AVChromaLocation chroma_location;
    int              field_order; // video: progressive, top-first, bottom-first
    int              refs;       // video: number of reference frames
    // Audio-specific:
    int              sample_rate;     // audio: samples per second
    int              channels;        // audio: number of channels
    uint64_t         channel_layout;  // audio: channel layout mask
    int              frame_size;      // audio: frame size in samples
    int              initial_padding; // audio: samples of delay before first sample
    int              trailing_padding;
    int64_t          seek_preroll;    // audio: samples to discard after seek
} AVCodecParameters;
```

For audio streams, the critical fields are: `codec_type` (should be `AVMEDIA_TYPE_AUDIO`), `codec_id` (e.g., `AV_CODEC_ID_FLAC`, `AV_CODEC_ID_AAC`, `AV_CODEC_ID_MP3`), `sample_rate` (Hz), `channels` (count), `channel_layout` (bitmask, e.g., `AV_CH_LAYOUT_STEREO`), `bit_rate` (bps), `extradata` / `extradata_size` (codec-specific init data like ADTS headers for AAC, FLAC metadata block for FLAC).

`extradata` is particularly important for audio formats where the codec needs initialization metadata that lives in the container but is not part of the compressed audio frames. For example:
- **AAC in ADTS:** The ADTS header contains 7 bytes per frame but the decoder needs the AudioSpecificConfig (ASC) which is stored in a separate atom in MP4.
- **FLAC:** The `STREAMINFO` metadata block is stored as `extradata`.
- **Opus:** The OpusHead and OpusTags headers.
- **Vorbis:** The identification header and comment header.

`avcodec_parameters_copy()` copies all fields from one `AVCodecParameters` to another — useful when setting up an output stream from an input stream.

### 3.4 AVPacket

`AVPacket` carries compressed data for one access unit (AU) — one encoded audio frame (or a fragment thereof) from a specific stream.

```c
typedef struct AVPacket {
    uint8_t *data;        // compressed data buffer
    int      size;        // size in bytes
    int64_t  pts;        // presentation timestamp in stream time_base
    int64_t  dts;        // decompression timestamp in stream time_base
    int      stream_index;  // which stream this packet belongs to
    int      flags;
    int64_t  duration;   // duration in stream time_base
    void    *priv;
    int64_t  pos;        // byte position in the file (-1 if unknown)
    int64_t  convergence_duration;
    // Reference counting (FFmpeg 4.0+):
    AVBufferRef *buf;
    // Side data:
    int64_t  side_data_elems;
    AVPacketSideData *side_data;
} AVPacket;
```

**Memory ownership model (FFmpeg 4.0+):** `data` is backed by a reference-counted buffer referenced by `buf`. When you call `av_read_frame()`, you receive a packet whose buffer is owned by the caller. You must call `av_packet_unref()` to release it when done. This is the standard pattern:

```c
AVPacket *pkt = av_packet_alloc();
while (av_read_frame(fmt_ctx, pkt) >= 0) {
    // use pkt->data, pkt->size, pkt->pts, etc.
    av_packet_unref(pkt); // free the packet's reference
}
av_packet_free(&pkt);
```

**PTS vs DTS:** For audio, PTS and DTS are almost always equal because audio frames are independently decodable (no temporal prediction). For video with B-frames, DTS can be less than PTS. For purely audio streams, you can safely treat them as identical.

**flags:** `AV_PKT_FLAG_KEY` indicates a keyframe (random access point). For audio, this is less critical than for video, but some formats mark every frame as keyframe.

**side data:** `AVPacketSideData` carries additional per-packet information. Notable types:
- `AV_PKT_DATA_NEW_EXTRADATA`: replacement extradata discovered during parsing.
- `AV_PKT_DATA_PALETTE`: embedded palette for video (rare).
- `AV_PKT_DATA_H263_MB_INFO`: H.263-specific macroblock info.
- `AV_PKT_DATA_SKIP_SAMPLES`: samples to skip at start/end of decoded frame.
- `AV_PKT_DATA_AUDIO_SERVICE_TYPE`: audio service type (for AC3/DTS in MPEG-TS).
- `AV_PKT_DATA_METADATA_UPDATE`: metadata changed between packets.

**Interleaving note:** When muxing with `av_interleaved_write_frame()`, the muxer may hold onto packets and reorder them to ensure proper temporal ordering. When using `av_write_frame()` (non-interleaved), the caller is responsible for correct ordering.

### 3.5 AVFrame

`AVFrame` holds decoded, uncompressed data — the output of the decoder after calling `avcodec_receive_frame()`. While `AVFrame` belongs to libavcodec rather than libavformat, it is the natural destination of demuxed-and-decoded data and the source for encoded-and-muxed data.

```c
typedef struct AVFrame {
    uint8_t *data[AV_NUM_DATA_POINTERS];  // planes for planar formats
    int      linesize[AV_NUM_DATA_POINTERS];
    uint8_t **extended_data;
    int      width, height;
    int      nb_samples;        // number of audio samples (per channel)
    int      format;            // pixel format (video) or sample format (audio)
    int      key_frame;
    enum AVPictureType pict_type;
    int64_t  pts;
    int64_t  pkt_pts;
    int64_t  pkt_dts;
    int      coded_picture_number;
    int      display_picture_number;
    int      quality;
    void    *opaque;
    int      repeat_pict;
    int      interlaced_frame;
    int      sample_rate;
    uint64_t channel_layout;
    AVBufferRef *buf[AV_NUM_DATA_POINTERS];
    AVBufferRef **extended_buf;
    int      nb_extended_buf;
    int      best_effort_timestamp;
    int64_t  pkt_pos;
    int64_t  pkt_duration;
    AVDictionary *metadata;
    int      decode_error_flags;
    int      channels;
    int      pkt_size;
    // ... more fields
} AVFrame;
```

For audio, `nb_samples` is the number of samples per channel in this frame. `format` is an `AVSampleFormat` value (e.g., `AV_SAMPLE_FMT_FLTP` for float planar, `AV_SAMPLE_FMT_S16` for signed 16-bit interleaved). `channel_layout` specifies which channels are present. `sample_rate` is the sampling frequency.

`pts` in the decoded frame is set by the decoder based on the packet's PTS and the known frame duration (`nb_samples / sample_rate`). The `best_effort_timestamp` field provides a best-effort PTS reconstruction for formats where the decoder needs to infer the timestamp.

### 3.6 AVIOContext and I/O

`AVIOContext` is the buffered I/O abstraction in libavformat. It sits between the format layer and the actual data source/sink, providing buffering, seeking, and a uniform callback interface.

```c
typedef struct AVIOContext {
    unsigned char *buffer;  // the internal buffer
    int buffer_size;
    int pos;                // current position in the buffer
    int eof_reached;         // end of file reached
    int error;               // contains error code or 0
    void *opaque;            // user data passed to callbacks
    int (*read_packet)(void *opaque, uint8_t *buf, int buf_size);
    int (*write_packet)(void *opaque, uint8_t *buf, int buf_size);
    int64_t (*seek)(void *opaque, int64_t offset, int whence);
    int64_t pos;
    int must_flush;          // must flush data on next write
    int write_flag;
    int max_packet_size;
    unsigned long checksum;
    unsigned char *checksum_ptr;
    unsigned long (*update_checksum)(unsigned long checksum,
                                     const uint8_t *buf, int size);
    int (*read_pause)(void *opaque, int pause);
    int64_t (*read_seek)(void *opaque, int stream_index,
                         int64_t timestamp, int flags);
    int seekable;
    int max_file_pos;
    int64_t bytes_read;
    int64_t seek_threshold;
    int64_t logical_pos;
    int logical_size;
} AVIOContext;
```

**Standard initialization:** `avio_open()` allocates an `AVIOContext` and connects it to a URL (file path, HTTP URL, etc.):

```c
AVFormatContext *fmt_ctx = NULL;
int ret = avformat_open_input(&fmt_ctx, "input.mp4", NULL, NULL);
// ...
// Then open output:
ret = avformat_alloc_output_context2(&fmt_ctx, NULL, NULL, "output.flac");
ret = avio_open(&fmt_ctx->pb, "output.flac", AVIO_FLAG_WRITE);
```

**Custom I/O:** You can provide your own read/write/seek callbacks by allocating an `AVIOContext` manually with `avio_alloc_context()`:

```c
unsigned char *buffer = av_malloc(buffer_size);
AVIOContext *avio = avio_alloc_context(
    buffer, buffer_size,     // internal buffer
    0,                       // write_flag = 0 (read-only)
    my_opaque,               // passed to callbacks
    my_read_packet,          // read callback
    my_write_packet,         // write callback (can be NULL)
    my_seek                  // seek callback (can be NULL for streaming)
);
fmt_ctx->pb = avio;
fmt_ctx->flags |= AVFMT_FLAG_CUSTOM_IO;
```

This is essential for non-file sources: reading from memory buffers, network streams, custom hardware interfaces, or decompressing data on the fly.

**Seekable:** The `seekable` field indicates the I/O capabilities:
- `0`: Not seekable (e.g., pipe, HTTP stream without range support)
- `AVIO_SEEKABLE_NORMAL`: Fully seekable (file)
- `AVIO_SEEKABLE_DEVICE`: Seekable within a device buffer

### 3.7 AVOutputFormat / AVInputFormat

These are the format descriptors — plugin registration structures. Every demuxer and muxer registers itself at startup through a static constructor mechanism.

```c
typedef struct AVInputFormat {
    const char *name;
    const char *long_name;
    int priv_data_size;
    int flags;
    const char **extensions;
    const struct AVCodecTag *const *codec_tag;
    const AVFormatProbeFunc proto_func;
    int (*read_probe)(AVProbeData *);
    int (*read_header)(struct AVFormatContext *);
    int (*read_packet)(struct AVFormatContext *, AVPacket *pkt);
    int (*read_close)(struct AVFormatContext *);
    int (*read_seek)(struct AVFormatContext *,
                     int stream_index, int64_t timestamp, int flags);
    int64_t (*read_timestamp)(struct AVFormatContext *, int,
                              int64_t *, int64_t);
    int (*read_play)(struct AVFormatContext *);
    int (*read_pause)(struct AVFormatContext *);
    // ... more fields
} AVInputFormat;

typedef struct AVOutputFormat {
    const char *name;
    const char *long_name;
    const char *mime_type;
    const char *const *extensions;
    uint8_t codec_tag;
    int priv_data_size;
    int (*write_header)(struct AVFormatContext *);
    int (*write_packet)(struct AVFormatContext *, AVPacket *);
    int (*write_trailer)(struct AVFormatContext *);
    int (*query_codec)(enum AVCodecID id, int std_compliance, ...);
    void (*get_output_timestamp)(struct AVFormatContext *, int, int64_t *, int64_t *);
    int flags;
    // ... more fields
} AVOutputFormat;
```

The `flags` field on `AVInputFormat` includes:
- `AVFMT_NO_BYTE_SEEK`: Don't seek by byte offset
- `AVFMT_GENERIC_INDEX`: Use generic index building
- `AVFMT_TS_DISCONT`: Timestamp discontinuity possible
- `AVFMT_NOBINSEARCH`: Don't use binary search in `av_seek_frame`
- `AVFMT_NOGENSEARCH`: Don't use generic seek algorithm
- `AVFMT_NO_BYTE_SEEK`: Byte seeking not supported
- `AVFMT_SEEK_TO_PTS`: Seek to PTS rather than DTS

The muxer/output format flags include:
- `AVFMT_NOFILE`: Doesn't need a file (e.g., image2)
- `AVFMT_NEEDNUMBER`: Needs numbered output (e.g., `%03d` patterns)
- `AVFMT_GLOBALHEADER`: Format requires global header in `write_header`
- `AVFMT_NOTIMESTAMPS`: Timestamps have no meaning
- `AVFMT_VARIABLE_FPS`: Variable frame rate
- `AVFMT_NOSTREAMS`: No streams in this format (e.g., raw images)

---

## 4. DEMUXING ALGORITHM (READING)

### 4.1 Complete Demuxing Workflow

Demuxing is the process of reading a container file and extracting individual compressed packets. The standard workflow is:

```
avformat_open_input()
    ↓
avformat_find_stream_info()
    ↓
av_read_frame() ←─── loop ───┐
    ↓                         │
avcodec_send_packet()         │
    ↓                         │
avcodec_receive_frame()       │
    ↓                         │
process decoded frame ────────┘
    ↓
avformat_close_input()
```

**Step-by-step breakdown:**

1. **`avformat_open_input()`** — Opens the input URL, creates the `AVFormatContext`, detects the format (or uses the specified format), calls the format's `read_header()` which parses the container header and populates the `AVStream` array. Does NOT probe codec parameters yet.

```c
AVFormatContext *fmt_ctx = NULL;
int ret = avformat_open_input(&fmt_ctx, "input.mp4", NULL, NULL);
if (ret < 0) {
    // handle error: ret is a negative AVERROR code
    av_log(NULL, AV_LOG_ERROR, "Cannot open file: %s\n", av_err2str(ret));
    return ret;
}
```

2. **`avformat_find_stream_info()`** — Probes the streams to determine codec parameters. For each stream without an identified codec, it reads packets, sends them to the decoder, and reads back decoded frames to extract parameters. This step can be slow — it may read the entire file for low-bitrate streams. Set `fmt_ctx->max_analyze_duration` and `fmt_ctx->probesize` to control behavior.

```c
AVDictionary *opts = NULL;
av_dict_set(&opts, "max_analyze_duration", "5000000", 0); // 5 seconds
av_dict_set(&opts, "probesize", "5000000", 0); // 5 MB
ret = avformat_find_stream_info(fmt_ctx, &opts);
av_dict_free(&opts);
```

You can disable this step with `fmt_ctx->max_analyze_duration = 0`, but then some streams may have unknown codec parameters.

3. **`av_read_frame()`** — Reads one packet at a time from the container. The packet belongs to one stream (`pkt->stream_index`). Returns `0` on success, `AVERROR_EOF` when end of file is reached, or a negative error code. The caller owns the returned packet — must call `av_packet_unref()` to free it.

```c
AVPacket *pkt = av_packet_alloc();
while (1) {
    ret = av_read_frame(fmt_ctx, pkt);
    if (ret < 0) {
        if (ret == AVERROR_EOF) {
            // normal end of file
        } else {
            // error
        }
        break;
    }
    // pkt contains compressed data for stream pkt->stream_index
    av_packet_unref(pkt);
}
```

4. **`avcodec_send_packet()`** — Sends the compressed packet to the decoder. The decoder may hold onto the packet (for formats that need multiple packets per frame) or return a frame immediately.

```c
AVCodecContext *dec_ctx = /* from stream */;
AVFrame *frame = av_frame_alloc();
ret = avcodec_send_packet(dec_ctx, pkt);
if (ret < 0) {
    av_log(NULL, AV_LOG_ERROR, "Error sending packet to decoder: %s\n", av_err2str(ret));
}
// pkt can now be unreferenced
```

5. **`avcodec_receive_frame()`** — Receives a decoded frame from the decoder. Call this in a loop until `AVERROR(EAGAIN)` is returned, indicating the decoder needs more input:

```c
while (ret >= 0) {
    ret = avcodec_receive_frame(dec_ctx, frame);
    if (ret == AVERROR(EAGAIN)) {
        ret = 0;
        break;
    } else if (ret < 0) {
        break;
    }
    // frame contains decoded PCM/audio data
    // frame->nb_samples, frame->data[0], etc.
    av_frame_unref(frame);
}
```

6. **`avformat_close_input()`** — Closes the input, frees all allocated memory, closes the `AVIOContext`, and closes the codec contexts:

```c
avformat_close_input(&fmt_ctx); // sets pointer to NULL
```

### 4.2 Stream Discovery

When `avformat_open_input()` calls the format's `read_header()`, it creates `AVStream` objects for each stream found in the container. The container header contains a stream directory — a table listing all streams and where their data is located.

For example, in **MP4/MOV**, the `moov` atom contains a `trak` box for each stream, which specifies the codec type, codec configuration (e.g., ESDS for AAC, `hvcC`/`avcC` for HEVC/H.264), and the `mdia` box which points to the `mdat` chunk data.

In **FLAC**, the `fLaC` marker is followed by `STREAMINFO` metadata block, then optionally `METADATA_BLOCK_DATA_VORBIS_COMMENT` for comments and `PICTURE` blocks. FLAC is unique among lossless formats in that it has a native streaming format with embedded metadata — there is no separate container.

In **OGG**, the first page of each logical bitstream contains a BOS (Beginning Of Stream) header that identifies the codec (e.g., `vorbis`, `speex`, `opus`). The container can interleave multiple logical bitstreams by alternating pages.

The `fmt_ctx->nb_streams` field tells you how many streams were found. You can iterate through them:

```c
for (unsigned int i = 0; i < fmt_ctx->nb_streams; i++) {
    AVStream *st = fmt_ctx->streams[i];
    AVCodecParameters *par = st->codecpar;
    if (par->codec_type == AVMEDIA_TYPE_AUDIO) {
        printf("Stream %u: audio, codec=%s, %d Hz, %d channels\n",
               i, avcodec_get_name(par->codec_id),
               par->sample_rate, par->channels);
    }
}
```

### 4.3 Codec Parameter Extraction

The `AVCodecParameters` in each `AVStream` contains all the information needed to open a decoder for that stream. The typical pattern:

```c
AVStream *audio_stream = NULL;
for (int i = 0; i < fmt_ctx->nb_streams; i++) {
    if (fmt_ctx->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_AUDIO) {
        audio_stream = fmt_ctx->streams[i];
        break;
    }
}

if (!audio_stream) {
    // no audio stream found
    return AVERROR_STREAM_NOT_FOUND;
}

const AVCodec *dec = avcodec_find_decoder(audio_stream->codecpar->codec_id);
if (!dec) {
    return AVERROR_DECODER_NOT_FOUND;
}

AVCodecContext *dec_ctx = avcodec_alloc_context3(dec);
if (!dec_ctx) {
    return AVERROR(ENOMEM);
}

ret = avcodec_parameters_to_context(dec_ctx, audio_stream->codecpar);
if (ret < 0) {
    avcodec_free_context(&dec_ctx);
    return ret;
}

// Open the decoder
ret = avcodec_open2(dec_ctx, dec, NULL);
if (ret < 0) {
    avcodec_free_context(&dec_ctx);
    return ret;
}
```

The key is `avcodec_parameters_to_context()`, which copies the codec parameters from the stream into the codec context, including:
- Sample rate, channel count, channel layout
- Bit rate
- Extradata (e.g., AAC AudioSpecificConfig)
- Codec-specific profile and level

You do NOT need to call `avformat_find_stream_info()` to get these parameters if you only need the codec configuration. `avformat_open_input()` (via `read_header()`) already populates the `codecpar` fields directly from the container metadata.

### 4.4 Probe Buffers and Format Detection

Format detection happens when `avformat_open_input()` is called with `fmt = NULL` (no specified format). The library:

1. Allocates a probe buffer of size `max_probe_size` (default 5 MB).
2. Reads data from the URL into the buffer.
3. Scores the buffer against each registered format's `read_probe()` function.
4. Picks the highest-scoring format and proceeds.

The probe function receives an `AVProbeData` structure:

```c
typedef struct AVProbeData {
    const char *filename;
    const char *buf;
    int buf_size;
    const char *mime_type;
} AVProbeData;
```

The probe function returns a score from 0 to 100. Higher scores mean a better match. FFmpeg uses several techniques:
- **Magic bytes:** Checking the first few bytes for known file signatures (e.g., `fLaC`, `OggS`, `ID3`, `ftyp`, `\x1aE\xdf\xa3` for Matroska).
- **Structural patterns:** Validating expected byte offsets and sizes (e.g., checking that atom sizes in MP4 are consistent).
- **Probabilistic matching:** For ambiguous formats, using partial data to estimate likelihood.

A score of `100` means the format is certain. A score of `0` means no match. A partial score (e.g., `50`) means the format is possible but uncertain — this is used for formats like MPEG-TS that have no unique magic bytes.

You can force a format with `avformat_open_input(&fmt_ctx, url, input_format, NULL)` where `input_format` is obtained from `av_find_input_format("mp4")` or `av_guess_format("mp4", NULL, NULL)`.

For streaming protocols, you can also set `AVFMT_FLAG_NOPARSE` to skip the probe phase entirely and rely on the protocol's content-type or format hint.

### 4.5 URLContext and Protocol Handlers

`AVIOContext` wraps a `URLContext`, which represents the underlying protocol:

```c
typedef struct URLContext {
    const struct URLProtocol *prot;
    char *filename;
    int flags;
    int is_streamed;       // 1 if no seeking possible
    int max_packet_size;
    char *headers;         // HTTP headers to send
    int64_t pos;           // current position
    int64_t eof;
    int64_t bytes_read;
    int64_t bytes_written;
    struct URLContext *next;
    void *priv_data;
    char *protocol_buffer;
    int protocol_buffer_size;
    int min_packet_size;
} URLContext;
```

Protocol handlers implement the actual I/O. Each registered protocol provides `url_open()`, `url_read()`, `url_write()`, `url_seek()`, `url_close()` functions. Built-in protocols include:

| Protocol | Prefix | Description |
|---|---|---|
| file | `file://` | Local filesystem |
| http | `http://`, `https://` | HTTP/HTTPS |
| pipe | `pipe://` | Anonymous pipe (read or write) |
| crypto | `crypto://` | AES-encrypted stream |
| tcp | `tcp://` | TCP socket |
| udp | `udp://` | UDP datagrams |
| rtmp | `rtmp://` | Real-Time Messaging Protocol |
| rtmpt | `rtmpt://` | RTMP over HTTP |
| rtsp | `rtsp://` | Real Time Streaming Protocol |
| srt | `srt://` | Secure Reliable Transport |
| concat | `concat://` | Virtual concatenation of streams |
| data | `data:` | Inline data (RFC 2397) |

Custom protocols can be registered via `ffmpeg_register_protocol2()` with custom read/write/seek callbacks.

### 4.6 Seeking in Containers

libavformat provides several seeking interfaces:

**`av_seek_frame()`** — The classic seeking function. Seeks to a timestamp in a specific stream:

```c
int av_seek_frame(AVFormatContext *s, int stream_index,
                  int64_t timestamp, int flags);
```

- `stream_index`: The stream to seek in. Use `-1` for arbitrary seeking (uses default stream).
- `timestamp`: The target timestamp in the stream's `time_base`.
- `flags`: `AVSEEK_FLAG_BACKWARD` (seek to nearest keyframe before timestamp), `AVSEEK_FLAG_BYTE` (seek by byte offset), `AVSEEK_FLAG_ANY` (seek to any frame — may not be decodable for video), `AVSEEK_FLAG_FRAME` (seek by frame number).

**`avformat_seek_file()`** — More powerful seeking that can seek within a range and returns whether the seek was successful:

```c
int avformat_seek_file(AVFormatContext *s, int stream_index,
                       int64_t min_ts, int64_t ts, int64_t max_ts, int flags);
```

**`av_seek_frame()` behavior by format:**
- **MP4/MOV:** Uses the `ctts` (composition time-to-sample) and `stss` (sync sample) tables. Seeking to a timestamp involves finding the nearest sync sample before the target and then adjusting the PTS using the `ctts` table.
- **FLAC:** Uses the `SEEKTABLE` metadata block (if present) for O(1) seeks. Without a seektable, performs a binary search.
- **OGG:** Uses the `KEYFRAME` flag in page headers. Pages are scanned and decoded until the target timestamp is found.
- **WAV:** Uses the `data` chunk's `data_size` and sample count to compute byte offsets directly.
- **Matroska (MKV):** Uses the `CuePoint` entries in the `Cues` element. Each `CuePoint` references cluster positions.

**Seekable range:** The range of seekable timestamps is `fmt_ctx->start_time` to `fmt_ctx->start_time + fmt_ctx->duration`. If `start_time` is `AV_NOPTS_VALUE`, the start time is unknown.

### 4.7 Packet Ordering and DTS/PTS Handling

Packets from `av_read_frame()` are returned in **stream byte order** — the order they appear in the file, which is typically interleaved by presentation time (especially in well-formed MP4 and MKV). However, for formats with B-frame video coding (e.g., H.264), the PTS and DTS of video packets are NOT in the same order as the byte stream.

For audio-only containers, DTS equals PTS for every packet (audio frames are always independently decodable). The DTS field is often set to `AV_NOPTS_VALUE` for audio.

For muxing, you must ensure that packets are written in the correct DTS/PTS order. `av_interleaved_write_frame()` handles this automatically by buffering packets and reordering them as needed. `av_write_frame()` writes packets in the order you provide — which is only safe if you guarantee correct ordering.

**Timestamp wrap-around:** In MPEG-TS, PTS/DTS are 33-bit values that wrap around every ~26.5 hours at a 90 kHz clock. FFmpeg handles this internally using its timestamp comparison functions (`av_compare_ts()`, `av_rescale_delta()`, etc.) which account for unsigned wrap-around.

**Negative timestamps:** Some formats produce negative first DTS/PTS (e.g., files with encoder delay stored as negative offset). FFmpeg handles this, but when muxing, you may need to shift all timestamps to start from 0 or a positive value using `av_interleaved_write_frame()`'s `pts_wrap` or by manually adjusting.

### 4.8 Memory Management in Demuxing

Memory management follows a strict ownership model:

| Object | Allocated By | Freed By |
|---|---|---|
| `AVFormatContext` | `avformat_open_input()` | `avformat_close_input()` |
| `AVIOContext` | `avio_open()` or `avio_alloc_context()` | `avio_closep()` or `av_freep(&ctx->pb)` |
| `AVStream` | Format's `read_header()` | `avformat_close_input()` |
| `AVCodecParameters` | Part of `AVStream` allocation | Part of `AVStream` free |
| `AVPacket` (owned) | `av_read_frame()` caller (ref-counted via `buf`) | `av_packet_unref()` |
| `AVPacket` (allocated) | `av_packet_alloc()` | `av_packet_free()` |
| Extradata | Format's `read_header()` (may be reallocated by decoder) | `avformat_close_input()` |
| `AVCodecContext` | `avcodec_alloc_context3()` | `avcodec_free_context()` |

**Critical rule:** Never call `av_free()` on `AVPacket->data` directly. Always use `av_packet_unref()` because the data may be backed by a shared buffer reference (`buf`). The `unref` call decrements the reference count and frees the buffer only when it reaches zero.

**Example of proper cleanup sequence:**

```c
avcodec_free_context(&dec_ctx);
avformat_close_input(&fmt_ctx);
// Never: av_free(fmt_ctx); // already freed by close_input
```

---

## 5. MUXING ALGORITHM (WRITING)

### 5.1 Complete Muxing Workflow

Muxing is the process of taking compressed packets and writing them into a container. The standard workflow is:

```
avformat_alloc_output_context2()
    ↓
av_guess_format() / avformat_alloc_output_context2(fmt_name)
    ↓
avio_open()
    ↓
avformat_new_stream() × N streams
    ↓
avcodec_parameters_from_context() / avcodec_parameters_copy()
    ↓
avformat_write_header()
    ↓
av_interleaved_write_frame() ←─── loop ───┐
    ↓                                      │
process and write next packet ─────────────┘
    ↓
av_write_trailer()
    ↓
avio_closep()
```

**Step-by-step breakdown:**

1. **`avformat_alloc_output_context2()`** — Allocates the output format context and guesses the format from the filename extension:

```c
AVFormatContext *ofmt_ctx = NULL;
int ret = avformat_alloc_output_context2(&ofmt_ctx, NULL, NULL, "output.flac");
if (ret < 0 || !ofmt_ctx) {
    av_log(NULL, AV_LOG_ERROR, "Could not create output context\n");
    return ret;
}
```

2. **`av_guess_format()`** — If you need to specify the format explicitly (or when the filename doesn't indicate the format):

```c
const AVOutputFormat *fmt = av_guess_format("flac", NULL, "audio/flac");
if (!fmt) {
    return AVERROR_DECODER_NOT_FOUND;
}
// Then set it in the context:
ofmt_ctx->oformat = fmt;
```

3. **`avio_open()`** — Opens the output URL for writing:

```c
ret = avio_open(&ofmt_ctx->pb, "output.flac", AVIO_FLAG_WRITE);
if (ret < 0) {
    av_log(NULL, AV_LOG_ERROR, "Could not open output file: %s\n", av_err2str(ret));
    return ret;
}
```

4. **`avformat_new_stream()`** — Adds a new stream for each audio track:

```c
AVStream *out_stream = avformat_new_stream(ofmt_ctx, NULL);
if (!out_stream) {
    return AVERROR(ENOMEM);
}
// out_stream->index will be set automatically
// out_stream->id should be set if the format requires stream IDs
out_stream->id = 0; // for FLAC, MP3, etc.
```

5. **`avcodec_parameters_from_context()`** — Copies codec parameters from the codec context into the output stream:

```c
AVCodecContext *enc_ctx = /* from encoder */;
ret = avcodec_parameters_from_context(out_stream->codecpar, enc_ctx);
if (ret < 0) {
    return ret;
}
// Set the time base to match the audio sample rate
out_stream->time_base = (AVRational){1, enc_ctx->sample_rate};
```

Or for a copy (pass-through) operation:

```c
AVStream *in_stream = fmt_ctx->streams[audio_stream_index];
ret = avcodec_parameters_copy(out_stream->codecpar, in_stream->codecpar);
if (ret < 0) {
    return ret;
}
out_stream->time_base = in_stream->time_base;
```

6. **`avformat_write_header()`** — Writes the container header/metadata. Some formats (e.g., MP4, MOV) need this to be called before any packets. Others (e.g., raw FLAC) write minimal or no header here:

```c
ret = avformat_write_header(ofmt_ctx, NULL);
if (ret < 0) {
    av_log(NULL, AV_LOG_ERROR, "Error writing header: %s\n", av_err2str(ret));
    return ret;
}
```

For formats that support global metadata, you can pass options:

```c
AVDictionary *opts = NULL;
av_dict_set(&opts, "title", "Album Title", 0);
av_dict_set(&opts, "artist", "Artist Name", 0);
ret = avformat_write_header(ofmt_ctx, &opts);
av_dict_free(&opts);
```

7. **`av_interleaved_write_frame()`** — Writes a packet with automatic interleaving. This is the preferred method because it handles packet reordering and ensures that the output file is seekable:

```c
AVPacket *pkt = /* packet with encoded audio data */;
// Convert timestamps from input time base to output time base
pkt->stream_index = out_stream->index;
// For a copy operation, scale timestamps:
av_packet_rescale_ts(pkt, in_stream->time_base, out_stream->time_base);
ret = av_interleaved_write_frame(ofmt_ctx, pkt);
if (ret < 0) {
    av_log(NULL, AV_LOG_ERROR, "Error writing frame: %s\n", av_err2str(ret));
}
```

8. **`av_write_trailer()`** — Finalizes the container. This writes any footer data, updates indexes, computes checksums, etc.:

```c
ret = av_write_trailer(ofmt_ctx);
if (ret < 0) {
    av_log(NULL, AV_LOG_ERROR, "Error writing trailer: %s\n", av_err2str(ret));
}
```

9. **`avio_closep()`** — Closes the output I/O context and frees resources:

```c
avio_closep(&ofmt_ctx->pb);
```

### 5.2 Codec Parameter Setup

Setting up codec parameters correctly is the most critical step in muxing. The output stream's `codecpar` must exactly describe the encoded data that will be written to the stream.

For a typical pass-through (copy) scenario:

```c
AVStream *in_stream = fmt_ctx->streams[audio_idx];
AVStream *out_stream = avformat_new_stream(ofmt_ctx, NULL);
avcodec_parameters_copy(out_stream->codecpar, in_stream->codecpar);
out_stream->codecpar->codec_tag = 0; // Let the muxer choose codec tag
```

For an encoding scenario:

```c
AVCodecContext *enc_ctx = avcodec_alloc_context3(enc);
enc_ctx->codec_id = AV_CODEC_ID_FLAC;
enc_ctx->sample_rate = 48000;
enc_ctx->channels = 2;
enc_ctx->channel_layout = AV_CH_LAYOUT_STEREO;
enc_ctx->bit_rate = 0; // FLAC is lossless, bitrate is variable
enc_ctx->compression_level = 8; // Maximum compression

avcodec_parameters_from_context(out_stream->codecpar, enc_ctx);
out_stream->time_base = (AVRational){1, 48000};
```

**Important:** The `time_base` of the output stream must be set correctly before `avformat_write_header()` is called. For audio, the time base should match the sample rate: `{1, sample_rate}`. The muxer uses this to calculate byte offsets from timestamps and to compute the duration of the last packet.

### 5.3 Container Header Writing

Each container format has its own header structure. The muxer writes the header in `avformat_write_header()` and the footer in `av_write_trailer()`.

For **FLAC**, `write_header` writes the `fLaC` marker and `STREAMINFO` block (including the `min_blocksize`, `max_blocksize`, sample rate, channel count, bits per sample, total samples). The `write_trailer` writes a `write_trailer` block if any metadata blocks remain to be written.

For **MP4/MOV**, `write_header` writes the `ftyp` brand, `moov` atom (containing `mvhd`, `trak`, `udta`), and the beginning of `mdat`. The `write_trailer` updates the `moov` atom with final sample table information (`stsz`, `stco`/`co64`, `stsc`, `stts`, `ctts`, `stss`) and moves the `moov` atom to a better position (for `+faststart`).

For **OGG**, `write_header` writes BOS (Beginning Of Stream) pages for each stream containing the codec identification headers. The `write_trailer` writes EOS (End Of Stream) pages.

For **Matroska**, `write_header` writes the EBML header, segment info, and cluster start. The `write_trailer` finalizes the last cluster and writes the `Cues` element.

### 5.4 Interleaving Strategy

Interleaving is the technique of ordering packets in the output file so that they can be read and decoded in the correct temporal order without buffering. For audio/video, this means alternating video and audio packets by presentation time.

**`av_interleaved_write_frame()`** handles this automatically:
- It buffers packets internally per-stream.
- It compares the PTS of the incoming packet with buffered packets from other streams.
- It writes out all packets that are now "due" (their presentation time has passed).
- This ensures that seeking to any point in the file can begin decoding immediately.

**`av_write_frame()`** is non-interleaved:
- It writes packets in the exact order you provide.
- You are responsible for correct ordering.
- Generally only safe for single-stream outputs (e.g., raw FLAC, WAV).

For audio-only pipelines, interleaving provides minimal benefit since there is only one stream. However, using `av_interleaved_write_frame()` is still recommended for consistency and future-proofing (if you later add video or subtitle streams).

**Interleaving also affects seeking.** A non-interleaved file may still be seekable if the format has an index, but interleaved files are generally more seek-friendly because the first data encountered during a sequential read is typically near the start of the timeline.

### 5.5 Writing Packets

The core packet-writing functions:

**`av_interleaved_write_frame()`** — The standard, safe option:
```c
int av_interleaved_write_frame(AVFormatContext *s, AVPacket *pkt);
```

**`av_write_frame()`** — For non-interleaved, direct writing:
```c
int av_write_frame(AVFormatContext *s, AVPacket *pkt);
```

**Timestamp conversion** before writing is mandatory when muxing from decoded/re-encoded data:

```c
// Input packet from one file, output to another file with different time base
av_packet_rescale_ts(pkt,
    in_stream->time_base,   // source time base (e.g., {1, 44100})
    out_stream->time_base   // destination time base (e.g., {1, 48000})
);
pkt->stream_index = out_stream->index;
```

**DTS and PTS:** For audio, DTS is usually `AV_NOPTS_VALUE` or equals PTS. The muxer handles DTS assignment when needed (e.g., for video with B-frames).

**Duration:** `pkt->duration` must be set correctly. For audio, this is typically `nb_samples` in the stream's time base. When muxing with `av_interleaved_write_frame()`, the duration is used for packet scheduling.

**Keyframe flag:** For video streams, setting `pkt->flags |= AV_PKT_FLAG_KEY` marks the packet as a keyframe (random access point). This is critical for seeking — the muxer may use this to build an index. For audio, this flag is less critical but some formats use it for stream segmentation.

### 5.6 Writing Headers and Trailers

**`avformat_write_header()`** behavior by format:**

| Format | Header Behavior |
|---|---|
| FLAC | Writes `fLaC` + `STREAMINFO` + any metadata blocks from `av_dict_set()` |
| MP4/MOV | Writes `ftyp` + `moov` (empty tables) + begins `mdat` |
| OGG | Writes BOS pages (codec headers) |
| Matroska | Writes EBML header + Segment header + SegmentInfo |
| WAV | Writes `RIFF` header + `fmt ` chunk; `data` size set to 0 |
| AIFF | Writes `FORM` + `AIFF` + `COMM` + empty `SSND` |
| MP3 | No header (raw frames); writes ID3v2 tag if requested |
| MPEG-TS | Writes PAT/PMT tables + video/audio PES headers |
| CAF | Writes `CAF` header + `desc` + `fmt ` chunk |

**`av_write_trailer()`** behavior by format:**

| Format | Trailer Behavior |
|---|---|
| FLAC | Writes any remaining metadata blocks; pad to block alignment |
| MP4/MOV | Updates `moov` with final sample tables; may relocate `moov` for `+faststart` |
| WAV | Updates `data` chunk size in header |
| AIFF | Updates `SSND` chunk size and offset |
| MPEG-TS | Writes NULL PID packets for PCR alignment |
| Matroska | Finalizes last cluster; writes `Cues` element |

**`flush` parameter:** When calling `avformat_write_header()`, you can pass a pointer to an `AVDictionary` to set format-specific options. Options that are not recognized are returned in the dictionary after the call.

### 5.7 Memory Management in Muxing

| Object | Allocated By | Freed By |
|---|---|---|
| `AVFormatContext` (output) | `avformat_alloc_output_context2()` | `avformat_free_context()` |
| `AVIOContext` | `avio_open()` | `avio_closep()` |
| `AVStream` | `avformat_new_stream()` | Part of `AVFormatContext` free |
| `AVPacket` (owned) | `av_read_frame()` or `av_packet_alloc()` | `av_packet_unref()` |
| `AVPacket` (written) | Caller | `av_interleaved_write_frame()` takes ownership and unreferences it internally |

**Key difference from demuxing:** When you call `av_interleaved_write_frame()`, the function takes ownership of the packet's reference. You should NOT call `av_packet_unref()` after a successful write. However, if the write fails, the packet is NOT consumed — you must unref it yourself.

**Proper error handling pattern:**

```c
ret = av_interleaved_write_frame(ofmt_ctx, pkt);
if (ret < 0) {
    av_log(NULL, AV_LOG_ERROR, "Error writing frame: %s\n", av_err2str(ret));
    av_packet_unref(pkt); // unref on error
    // possibly break and go to trailer/cleanup
}
```

**Cleanup order:**
```c
// Write remaining packets
av_interleaved_write_frame(ofmt_ctx, NULL); // drain the interleaver
av_write_trailer(ofmt_ctx);
avio_closep(&ofmt_ctx->pb);
avformat_free_context(ofmt_ctx);
```

Calling `av_interleaved_write_frame()` with `NULL` packet drains any internally buffered packets — essential to flush the last packets before the trailer.

---

## 6. AUDIO-SPECIFIC DETAILS

### 6.1 Audio Stream Setup

Setting up an audio stream for muxing requires matching the encoder's output format:

```c
AVCodecContext *enc_ctx = avcodec_alloc_context3(enc);
enc_ctx->codec_type = AVMEDIA_TYPE_AUDIO;
enc_ctx->sample_rate = 48000;
enc_ctx->channel_layout = AV_CH_LAYOUT_STEREO;
enc_ctx->channels = av_get_channel_layout_nb_channels(enc_ctx->channel_layout);
enc_ctx->bit_rate = 0; // or specific bitrate for lossy codecs
enc_ctx->time_base = (AVRational){1, 48000};
enc_ctx->frame_size = enc->capabilities & AV_CODEC_CAP_VARIABLE_FRAME_SIZE
                      ? 0 : 1152; // set appropriately for the codec

ret = avcodec_open2(enc_ctx, enc, NULL);
```

For the output stream:
```c
AVStream *out_stream = avformat_new_stream(ofmt_ctx, NULL);
avcodec_parameters_from_context(out_stream->codecpar, enc_ctx);
out_stream->time_base = enc_ctx->time_base;
```

### 6.2 AAC in MP4 Muxing

AAC encoding produces raw Access Units (AUs). When muxing into MP4, these AUs need to be wrapped in **LATM** (Low Overhead Audio Transport Multiplex) or **ADTS** (Audio Data Transport Stream) framing, depending on the muxer.

**MP4/AAC** uses LATM framing. The `esds` atom stores the `AudioSpecificConfig` that describes the AAC profile, sample rate, and channel configuration. This is derived from the encoder's `extradata`.

**ADTS** is used for raw `.aac` files. Each AAC frame is prefixed with a 7-byte ADTS header containing the MPEG-4 Audio Object Type, sampling frequency index, channel configuration, and frame length.

FFmpeg's AAC encoder can be configured with:
```bash
ffmpeg -i input.wav -c:a aac -profile:a aac_lc -b:a 256k output.m4a
ffmpeg -i input.wav -c:a aac -profile:a aac_he_v2 -b:a 64k output.m4a  # HE-AAC v2
```

**Muxing raw AAC to MP4** requires proper setup of the `esds` atom:
```c
// The encoder's extradata contains the AudioSpecificConfig
out_stream->codecpar->extradata = av_malloc(enc_ctx->extradata_size);
memcpy(out_stream->codecpar->extradata, enc_ctx->extradata, enc_ctx->extradata_size);
out_stream->codecpar->extradata_size = enc_ctx->extradata_size;
```

For HE-AAC, the `sbr` flag in the AudioSpecificConfig must be set. FFmpeg handles this automatically.

**Gapless AAC:** MP4 supports gapless playback through the `edit list` (`elst` atom), which specifies encoder delay and padding as a track edit. The encoder's `initial_padding` field and the `seek_preroll` field should be reflected in the output container's edit list.

### 6.3 FLAC Muxing

FLAC is unique because its native format is a self-contained streaming format with no outer container required. The FLAC bitstream format is:

```
fLaC marker (4 bytes)
METADATA_BLOCK header (4 bytes)
  METADATA_BLOCK_DATA (variable)
    STREAMINFO (required, first)
    ...
METADATA_BLOCK header
  METADATA_BLOCK_DATA
    ... (VORBIS_COMMENT, PICTURE, etc.)
audio frames...
```

When muxing to **native FLAC**, FFmpeg's FLAC muxer (`flac` format) writes:
1. `fLaC` marker
2. `STREAMINFO` block (from encoder's output parameters)
3. Any metadata blocks set via `av_dict_set()` (Vorbis comments)
4. Audio frames with FLAC frame headers

```c
AVCodecContext *enc_ctx = avcodec_alloc_context3(avcodec_find_encoder(AV_CODEC_ID_FLAC));
enc_ctx->sample_rate = 48000;
enc_ctx->channels = 2;
enc_ctx->compression_level = 8; // 0-12, default 5

avformat_alloc_output_context2(&ofmt_ctx, NULL, "flac", "output.flac");
avio_open(&ofmt_ctx->pb, "output.flac", AVIO_FLAG_WRITE);
// ... setup streams and write_header ...
```

**FLAC in other containers:** FLAC can be embedded in OGG, Matroska, and other containers. In these cases, the FLAC frames are wrapped in the container's framing, and the `STREAMINFO` may be duplicated as container metadata.

**Metadata:** FLAC uses Vorbis comments — the same comment format as OGG Vorbis and Opus. Set with:
```c
av_dict_set(&ofmt_ctx->metadata, "TITLE", "Track Title", 0);
av_dict_set(&ofmt_ctx->metadata, "ARTIST", "Artist", 0);
av_dict_set(&ofmt_ctx->metadata, "ALBUM", "Album", 0);
av_dict_set(&ofmt_ctx->metadata, "DATE", "2024", 0);
av_dict_set(&ofmt_ctx->metadata, "TRACKNUMBER", "1", 0);
av_dict_set(&ofmt_ctx->metadata, "GENRE", "Electronic", 0);
```

**ReplayGain in FLAC:** Set as Vorbis comment fields:
```
REPLAYGAIN_TRACK_GAIN=-4.5 dB
REPLAYGAIN_TRACK_PEAK=0.56478912
```

**Seektable:** FLAC supports an optional seektable metadata block. The FLAC muxer can generate one automatically with `-write_seektable` option or you can provide one via `AVDictionary` options.

### 6.4 MP3 Muxing

MP3 raw frames have no container structure. FFmpeg's MP3 muxer handles:
1. **ID3v2 tag** at the beginning (if requested via `-id3v2_version` or metadata).
2. **MPEG audio frames** written sequentially.
3. **ID3v1 tag** at the end (optional, via `write_id3v1` option).

```bash
ffmpeg -i input.wav -c:a libmp3lame -q:a 2 output.mp3
```

For VBR MP3 with seeking support:
```bash
ffmpeg -i input.wav -c:a libmp3lame -q:a 2 -movflags +faststart output.mp3
```

**MP3 muxing with LAME:** The LAME encoder embeds the `LAME` info tag in the first frame's main data area, containing encoder version, VBR quality, replaygain info, and encoder delay/padding.

**Metadata in MP3:** FFmpeg automatically writes ID3v2.3 tags for compatibility (or ID3v2.4 if specified). The metadata is converted to ID3v2 frames:

```c
av_dict_set(&ofmt_ctx->metadata, "title", "Song Title", 0);
av_dict_set(&ofmt_ctx->metadata, "artist", "Artist", 0);
// FFmpeg handles the conversion to ID3v2 frames
```

**MP3 in MP4:** When muxing MP3 into MP4 (`.m4a`), FFmpeg wraps the MP3 frames in an LATM muxed stream (`mpa` RTP payload type). The output is technically not pure MP3 — it is an MPEG-4 audio stream using MP3 as the codec. Players handle this via the `mp4a` atom with codec tag `0x66` (MP3).

### 6.5 WAV Muxing

WAV is a simple RIFF container for PCM audio. The FLAC/PCM muxer writes:

```
RIFF header (12 bytes)
  'WAVE' form type
fmt  chunk (24 bytes) — PCM format descriptor
data chunk (8 + N bytes) — raw PCM samples
```

```c
avformat_alloc_output_context2(&ofmt_ctx, NULL, NULL, "output.wav");
// Automatic format detection from extension
avio_open(&ofmt_ctx->pb, "output.wav", AVIO_FLAG_WRITE);
```

**WAV limitations:**
- Limited to PCM formats (8/16/24/32-bit PCM, IEEE float, and some compressed variants like ADPCM).
- Maximum file size: 4 GB (RIFF format uses 32-bit size fields; use `RF64` format for larger files).
- Single audio stream only (though some extensions allow additional chunks).

**Writing WAV with FFmpeg:**
```bash
ffmpeg -i input.flac -c:a pcm_s24le output.wav  # 24-bit PCM
ffmpeg -i input.flac -c:a pcm_f32le output.wav  # 32-bit float
```

**WAV metadata:** WAV supports limited metadata through the `LIST INFO` chunk. FFmpeg writes standard tags here, but the support is incomplete for extended fields.

**Sample rate:** WAV stores the sample rate as a 32-bit unsigned integer, supporting up to 4,294,967,295 Hz (theoretical maximum, well beyond any practical need).

### 6.6 OGG Muxing

OGG is a multimedia container built on the Xiph.org codec family (Vorbis, FLAC, Speex, Opus, Theora). The OGG container organizes data as **pages**, with each page belonging to a specific logical bitstream (identified by a serial number). Pages can be interleaved across multiple logical bitstreams.

**OGG structure:**
```
Page = page_header (26 bytes) + segment_table + segments
  BOS (Beginning Of Stream) page: first page of each logical bitstream
  EOS (End Of Stream) page: last page of each logical bitstream
```

**Vorbis/Opus in OGG:**
```bash
ffmpeg -i input.wav -c:a libvorbis -q:a 6 output.ogg
ffmpeg -i input.wav -c:a libopus -b:a 192k output.ogg
```

**Metadata:** OGG uses Vorbis comments. FFmpeg's OGG muxer converts `AVDictionary` metadata to Vorbis comment headers:

```c
av_dict_set(&ofmt_ctx->metadata, "TITLE", "Track Title", 0);
av_dict_set(&ofmt_ctx->metadata, "ARTIST", "Artist", 0);
```

**Seeking in OGG:** OGG seeking uses the `KEYFRAME` flag in page headers. Without a seek index, OGG seeking requires scanning from the last keyframe — which can be slow for large files. Some implementations embed a seek index in a special metadata packet.

**Opus in OGG:** Opus in OGG uses two header packets in the BOS page: `OpusHead` (8 bytes + channel count + sample rate + initial padding + seek pre-roll) and `OpusTags` (Vorbis comment). The OGG muxer handles this automatically.

### 6.7 Matroska Muxing

Matroska (`.mkv`) is a general-purpose container based on EBML (Extensible Binary Meta Language). It supports virtually any codec, arbitrary numbers of streams, advanced features like chapters, attachments, and segment linking.

**MKV structure:**
```
EBML Header (identifies the file as EBML/Matroska)
Segment
  Segment Info (Seek, TimecodeScale, Duration, Title, ...)
  Cluster(s) (containing Block elements with packet data)
    Block (stream_index, timecode, flags, data)
    BlockGroup (Block + BlockAdditional + ReferenceBlock)
  Tracks (track metadata)
  Cues (seek index)
  Chapters
  Tags
  Attachments
```

**Muxing to Matroska:**
```bash
ffmpeg -i input.wav -c:a flac output.mkv
ffmpeg -i input.m4a -c:a copy output.mka  # extract audio from MP4 to MKA
```

**Matroska-specific options:**
```bash
ffmpeg -i input.wav -c:a flac -metadata title="Track" \
  - EBML.head.docType="matroska" -EBML.head.docTypeVersion=4 \
  output.mkv
```

**MKA (Matroska Audio):** `.mka` is the Matroska audio-only variant. It supports the same features as MKV but is typically used for audio-only content.

**Key features of Matroska for audio:**
- Chapter marks (stored as ChapterAtom elements)
- Attachment support (embed cover art as attachments)
- Segment linking (for multi-file productions)
- Track delay and seek preroll fields for gapless playback
- Full Unicode support in all metadata fields

---

## 7. METADATA HANDLING

### 7.1 AVDictionary for Metadata

`AVDictionary` is FFmpeg's key-value metadata structure. It is a simple, efficient string dictionary used throughout the API for both input and output metadata.

```c
typedef struct AVDictionary {
    int count;
    AVDictionaryEntry *elems;
} AVDictionary;

typedef struct AVDictionaryEntry {
    char *key;
    char *value;
    int flags;
} AVDictionaryEntry;
```

Key operations:

```c
// Set a metadata value
av_dict_set(&dict, "title", "My Song", 0);

// Flags: 0 = replace if exists, AV_DICT_DONT_STRDUP_KEY/VALUE to avoid copy
av_dict_set(&dict, "key", "value", AV_DICT_DONT_STRDUP_KEY | AV_DICT_DONT_STRDUP_VALUE);

// Get a metadata value
AVDictionaryEntry *e = NULL;
while ((e = av_dict_get(dict, "", e, AV_DICT_IGNORE_SUFFIX)) != NULL) {
    printf("%s = %s\n", e->key, e->value);
}

// Get specific key
const char *title = av_dict_get(dict, "title", NULL, 0)->value;

// Free all entries
av_dict_free(&dict);
```

**Flags for `av_dict_set`:**
- `AV_DICT_DONT_STRDUP_KEY`: Don't copy the key (use the provided pointer directly)
- `AV_DICT_DONT_STRDUP_VALUE`: Don't copy the value
- `AV_DICT_DONT_OVERWRITE`: Don't overwrite an existing key
- `AV_DICT_APPEND`: Append to existing key (for multi-value fields like multiple artists)

### 7.2 Reading Metadata

Metadata is accessible at three levels:

1. **Global metadata** (`AVFormatContext->metadata`): Applies to the entire file/stream.
2. **Stream metadata** (`AVStream->metadata`): Applies to a specific stream.
3. **Chapter metadata** (`AVChapter->metadata`): Per-chapter metadata.

```c
// Read global metadata
AVDictionaryEntry *tag = NULL;
while ((tag = av_dict_get(fmt_ctx->metadata, "", tag, AV_DICT_IGNORE_SUFFIX)) != NULL) {
    printf("Global: %s = %s\n", tag->key, tag->value);
}

// Read stream-specific metadata
for (unsigned int i = 0; i < fmt_ctx->nb_streams; i++) {
    AVDictionaryEntry *title = av_dict_get(fmt_ctx->streams[i]->metadata,
                                            "title", NULL, 0);
    if (title) {
        printf("Stream %u title: %s\n", i, title->value);
    }
}
```

**Metadata keys vary by format:**

| Format | Metadata System | Key Style |
|---|---|---|
| MP3 | ID3v2 | `TIT2`, `TPE1`, `TALB` (4-char frame IDs) |
| FLAC | Vorbis Comment | `TITLE`, `ARTIST`, `ALBUM` (title-case) |
| OGG | Vorbis Comment | `TITLE`, `ARTIST`, `ALBUM` (title-case) |
| MP4/MOV | iTunes/Atom | `©nam`, `©ART`, `©alb` (ATOM keys) |
| WAV | RIFF INFO / BEXT | `INAM`, `IART`, `ICRD` (RIFF INFO) |
| Matroska | EBML Tags | `TITLE`, `ARTIST`, `ALBUM` (title-case) |

FFmpeg normalizes many common tags when reading, mapping format-specific keys to canonical keys (e.g., `©nam` → `title`). Use `av_metadata_conv()` or check the format's metadata mapping table.

### 7.3 Writing Metadata

When muxing, set metadata on `AVFormatContext->metadata` before calling `avformat_write_header()`:

```c
av_dict_set(&ofmt_ctx->metadata, "title", "Track Title", 0);
av_dict_set(&ofmt_ctx->metadata, "artist", "Artist Name", 0);
av_dict_set(&ofmt_ctx->metadata, "album", "Album Name", 0);
av_dict_set(&ofmt_ctx->metadata, "date", "2024", 0);
av_dict_set(&ofmt_ctx->metadata, "track", "1/10", 0);
av_dict_set(&ofmt_ctx->metadata, "genre", "Electronic", 0);
av_dict_set(&ofmt_ctx->metadata, "composer", "Composer", 0);
av_dict_set(&ofmt_ctx->metadata, "copyright", "(c) 2024", 0);
av_dict_set(&ofmt_ctx->metadata, "comment", "Recorded at 96kHz", 0);
av_dict_set(&ofmt_ctx->metadata, "encoder", "FFmpeg 6.0", 0);
```

The muxer converts these keys to the format's native metadata system.

**ReplayGain metadata:**
```c
av_dict_set(&ofmt_ctx->metadata, "REPLAYGAIN_TRACK_GAIN", "-4.5 dB", 0);
av_dict_set(&ofmt_ctx->metadata, "REPLAYGAIN_TRACK_PEAK", "0.56478912", 0);
```

**Cover art:** Embedding cover art is format-specific:
- **FLAC:** Write a `PICTURE` metadata block with MIME type, picture type, description, and binary image data.
- **MP4:** Write an `APIC` atom or embed the image as a separate stream with `disposition = AV_DISPOSITION_ATTACHED_PIC`.
- **OGG:** Write picture data as a Vorbis comment with embedded base64 data or use a separate stream.

### 7.4 Format-Specific Metadata Keys

FFmpeg maintains internal metadata conversion tables that map canonical keys to format-specific keys. This enables generic metadata handling in applications while supporting format-specific storage.

**Metadata conversion when reading:**

When you read from an MP4 file and access the metadata:
```c
// Reading MP4 metadata — FFmpeg maps ATOM keys to canonical keys
av_dict_get(fmt_ctx->metadata, "title", NULL, 0);  // from ©nam
av_dict_get(fmt_ctx->metadata, "artist", NULL, 0); // from ©ART
av_dict_get(fmt_ctx->metadata, "album", NULL, 0);  // from ©alb
av_dict_get(fmt_ctx->metadata, "genre", NULL, 0);  // from ©gen
av_dict_get(fmt_ctx->metadata, "date", NULL, 0);   // from ©day
av_dict_get(fmt_ctx->metadata, "track", NULL, 0);  // from ©trk
```

**Custom/private metadata:** Use the `priv` or custom key namespace. Some formats support arbitrary user-defined text frames (`TXXX` in ID3v2, `COMM` in Vorbis comments, `©cmt` in MP4).

**Binary metadata:** For binary data (e.g., cover art, iTXt chunks), metadata is stored as `AVPacketSideData` on packets or as `AVDictionaryEntry` with binary values. Cover art in MP3 is stored as an `APIC` frame. In FLAC, it's a `PICTURE` metadata block. In Matroska, it's an `Attachment` element.

---

## 8. FFMPEG CLI REFERENCE

### 8.1 Format Detection

```bash
# Detect format without decoding
ffprobe -show_format input.mp3

# List all supported formats, muxers, and demuxers
ffmpeg -formats

# List only muxers (output formats)
ffmpeg -muxers

# List only demuxers (input formats)
ffmpeg -demuxers

# List all protocols
ffmpeg -protocols

# Probe specific format with detailed stream info
ffprobe -v quiet -print_format json -show_streams -show_format input.flac

# Show format details
ffprobe -show_format input.mp4 | grep -E "format_name|duration|bit_rate|nb_streams"
```

### 8.2 Demuxing CLI

```bash
# Basic demuxing — extract audio stream without re-encoding
ffmpeg -i input.mp4 -vn -c:a copy output.aac

# Extract specific audio stream by index
ffmpeg -i input.mp4 -map 0:a:0 -c:a copy output.flac

# Extract all audio streams
ffmpeg -i input.mp4 -map 0:a -c:a copy output_%03d.m4a

# Extract audio and force format based on extension
ffmpeg -i input.mp4 -vn -c:a pcm_s16le -f wav output.wav

# Demux only (no decoding) — useful for probing
ffmpeg -i input.mp4 -c copy -f null /dev/null

# Extract raw packets to file (for debugging)
ffmpeg -i input.mp4 -c copy -f mpegts output.ts

# Read metadata from file
ffprobe -v quiet -print_format json -show_format -show_streams input.flac
```

### 8.3 Muxing CLI

```bash
# Basic WAV to FLAC
ffmpeg -i input.wav -c:a flac output.flac

# With maximum compression
ffmpeg -i input.wav -c:a flac -compression_level 12 output.flac

# With metadata
ffmpeg -i input.wav \
  -metadata title="Song Title" \
  -metadata artist="Artist" \
  -metadata album="Album" \
  -metadata date="2024" \
  -metadata track="1" \
  -metadata genre="Electronic" \
  -c:a flac output.flac

# AAC in MP4 with explicit profile
ffmpeg -i input.wav \
  -c:a aac -profile:a aac_lc -b:a 256k \
  -movflags +faststart \
  output.m4a

# OPUS in OGG
ffmpeg -i input.wav -c:a libopus -b:a 192k output.ogg

# Multiple audio streams in one container
ffmpeg -i input.wav -i commentary.wav \
  -map 0:a:0 -c:a:0 flac \
  -map 1:a:0 -c:a:1 libopus -b:a:1 64k \
  output.mkv

# Force container format
ffmpeg -i input.wav -c:a copy -f flac output.bin

# Copy metadata from source
ffmpeg -i input.flac -i source.mp3 -map_metadata 0 -c:a copy output.mp3
```

### 8.4 Container Format Options

**FLAC options:**

| Option | Type | Default | Description |
|---|---|---|---|
| `compression_level` | int | 5 | 0 (fast) to 12 (best) |
| `lpc_coeff_precision` | int | 15 | LPC coefficient precision (4–15) |
| `lpc_type` | int | `FLAC_LPC_TYPE_LEVINSON` | LPC algorithm |
| `lpc_passes` | int | 2 | Number of passes |
| `predicates` | int | 0 | Number of level predictors |
| `min_partition_order` | int | 0 | Min partition order |
| `max_partition_order` | int | 6 | Max partition order |
| `write_crcsum` | int | 1 | Write CRC-32 in frame header |
| `write_seektable` | str | NULL | Seektable template |

**MP4/MOV options:**

| Option | Type | Default | Description |
|---|---|---|---|
| `movflags` | flags | 0 | `+faststart`, `+frag_keyframe`, `+default_base_moof`, `+delay_moov`, `+negative_cts_offsets` |
| `brand` | str | `isom` | MP4 brand (e.g., `iso2`, `iso5`, `M4A`) |
| `encryption_scheme` | int | 0 | Enable encryption (CENC) |
| `legacy_movformat` | int | 0 | Use legacy MOV/MP4 format |
| `write_tmcd` | int | 1 | Write timecode track |
| `movie_timescale` | int | 1000 | Timescale for `moov` atom |
| `track_timescale` | int | 0 | Use stream's own timescale |

```bash
# Fast-start MP4 (moov before mdat) — enables streaming
ffmpeg -i input.mov -c:a aac -movflags +faststart output.mp4

# Fragmented MP4 (for DASH/HLS streaming)
ffmpeg -i input.mov -c:a aac -movflags +frag_keyframe+default_base_moof \
  -frag_duration 1000000 output.m4a
```

**OGG options:**

| Option | Type | Default | Description |
|---|---|---|---|
| `ogg_delay` | int | 0 | Page granule position delay (ns) |
| `ogg_serial` | int | random | Serial number for the stream |

**Matroska options:**

| Option | Type | Default | Description |
|---|---|---|---|
| `EBML.head.docType` | str | `matroska` | EBML DocType |
| `EBML.head.docTypeVersion` | int | 4 | DocType version |
| `EBML.head.docTypeReadVersion` | int | 2 | Minimum DocType version for reading |

**WAV options:**

| Option | Type | Default | Description |
|---|---|---|---|
| `write_bext` | bool | 0 | Write Broadcast Extension (`bext`) chunk |
| `write_peak` | bool | 0 | Write Peak Envelope (`peak`) chunk |
| `peak_block_size` | int | 0 | Peak block size |
| `peak_format` | int | 0 | Peak format (`0`=uint8, `1`=uint16, `2`=float16) |

**MP3 options:**

| Option | Type | Default | Description |
|---|---|---|---|
| `id3v2_version` | int | 3 | ID3v2 version (3 or 4) |
| `write_id3v1` | bool | 1 | Write ID3v1 tag at end |
| `audio_service_type` | int | 0 | ATSC A/53 AC-3 service type |

---

## 9. COMPLETE WORKING C PROGRAM

The following is a complete, production-quality C program that opens an MP4 file, extracts the audio stream, decodes it to PCM, and writes it to a WAV file. It demonstrates proper initialization, error handling, memory management, and the complete demux-decode-write pipeline.

```c
#include <libavutil/log.h>
#include <libavutil/opt.h>
#include <libavutil/channel_layout.h>
#include <libavformat/avformat.h>
#include <libavcodec/avcodec.h>
#include <libavutil/samplefmt.h>
#include <stdio.h>
#include <stdlib.h>

#define AUDIO_IN_BUFF_SAMPLES 4096
#define AUDIO_OUT_BUFF_RAW_SAMPLES (AUDIO_IN_BUFF_SAMPLES * 8)

static int g_log_level = AV_LOG_INFO;

static int write_wav_header(AVFormatContext *s, int sample_rate,
                             int channels, int bits_per_sample) {
    uint8_t header[44];
    int byte_rate = sample_rate * channels * bits_per_sample / 8;
    int block_align = channels * bits_per_sample / 8;
    int data_size = 0; /* unknown yet */

    /* RIFF chunk descriptor */
    header[0] = 'R'; header[1] = 'I'; header[2] = 'F'; header[3] = 'F';
    /* chunk size — placeholder, filled at the end */
    header[4] = 0; header[5] = 0; header[6] = 0; header[7] = 0;
    header[8] = 'W'; header[9] = 'A'; header[10] = 'V'; header[11] = 'E';

    /* fmt sub-chunk */
    header[12] = 'f'; header[13] = 'm'; header[14] = 't'; header[15] = ' ';
    header[16] = 16;  header[17] = 0;  header[18] = 0;  header[19] = 0; /* subchunk1 size */
    header[20] = 1;   header[21] = 0;  /* audio format: PCM = 1 */
    header[22] = (uint8_t)(channels & 0xFF);
    header[23] = (uint8_t)((channels >> 8) & 0xFF);
    header[24] = (uint8_t)(sample_rate & 0xFF);
    header[25] = (uint8_t)((sample_rate >> 8) & 0xFF);
    header[26] = (uint8_t)((sample_rate >> 16) & 0xFF);
    header[27] = (uint8_t)((sample_rate >> 24) & 0xFF);
    header[28] = (uint8_t)(byte_rate & 0xFF);
    header[29] = (uint8_t)((byte_rate >> 8) & 0xFF);
    header[30] = (uint8_t)((byte_rate >> 16) & 0xFF);
    header[31] = (uint8_t)((byte_rate >> 24) & 0xFF);
    header[32] = (uint8_t)(block_align & 0xFF);
    header[33] = (uint8_t)((block_align >> 8) & 0xFF);
    header[34] = (uint8_t)(bits_per_sample & 0xFF);
    header[35] = (uint8_t)((bits_per_sample >> 8) & 0xFF);

    /* data sub-chunk */
    header[36] = 'd'; header[37] = 'a'; header[38] = 't'; header[39] = 'a';
    /* subchunk2 size — placeholder */
    header[40] = 0; header[41] = 0; header[42] = 0; header[43] = 0;

    if (fwrite(header, 1, 44, (FILE *)s->pb->opaque) != 44) {
        av_log(NULL, AV_LOG_ERROR, "Failed to write WAV header\n");
        return AVERROR(EIO);
    }
    return 0;
}

static int finish_wav_header(AVFormatContext *s, int64_t data_size) {
    uint8_t header[8];
    uint8_t riff_size[4];
    uint8_t data_size_bytes[4];
    uint32_t riff_chunk_size = (uint32_t)(36 + data_size);
    uint32_t data_chunk_size = (uint32_t)data_size;

    /* RIFF chunk size: file size - 8 */
    riff_size[0] = (uint8_t)(riff_chunk_size & 0xFF);
    riff_size[1] = (uint8_t)((riff_chunk_size >> 8) & 0xFF);
    riff_size[2] = (uint8_t)((riff_chunk_size >> 16) & 0xFF);
    riff_size[3] = (uint8_t)((riff_chunk_size >> 24) & 0xFF);

    /* data chunk size */
    data_size_bytes[0] = (uint8_t)(data_chunk_size & 0xFF);
    data_size_bytes[1] = (uint8_t)((data_chunk_size >> 8) & 0xFF);
    data_size_bytes[2] = (uint8_t)((data_chunk_size >> 16) & 0xFF);
    data_size_bytes[3] = (uint8_t)((data_chunk_size >> 24) & 0xFF);

    /* Seek to RIFF size field (offset 4) and write */
    if (fseek((FILE *)s->pb->opaque, 4, SEEK_SET) != 0) return AVERROR(EIO);
    if (fwrite(riff_size, 1, 4, (FILE *)s->pb->opaque) != 4) return AVERROR(EIO);

    /* Seek to data size field (offset 40) and write */
    if (fseek((FILE *)s->pb->opaque, 40, SEEK_SET) != 0) return AVERROR(EIO);
    if (fwrite(data_size_bytes, 1, 4, (FILE *)s->pb->opaque) != 4) return AVERROR(EIO);

    return 0;
}

int main(int argc, char **argv) {
    if (argc < 3) {
        av_log(NULL, AV_LOG_ERROR,
               "Usage: %s <input.mp4> <output.wav>\n", argv[0]);
        return 1;
    }

    const char *input_path = argv[1];
    const char *output_path = argv[2];

    /* ─── 1. Open input ─────────────────────────────────────────── */
    AVFormatContext *fmt_ctx = NULL;
    int ret = avformat_open_input(&fmt_ctx, input_path, NULL, NULL);
    if (ret < 0) {
        av_log(NULL, AV_LOG_ERROR, "Cannot open input '%s': %s\n",
               input_path, av_err2str(ret));
        return 1;
    }

    /* ─── 2. Find stream info ──────────────────────────────────── */
    ret = avformat_find_stream_info(fmt_ctx, NULL);
    if (ret < 0) {
        av_log(NULL, AV_LOG_ERROR, "Cannot find stream info: %s\n",
               av_err2str(ret));
        goto cleanup_open;
    }

    /* ─── 3. Find audio stream ─────────────────────────────────── */
    int audio_stream_idx = -1;
    for (unsigned int i = 0; i < fmt_ctx->nb_streams; i++) {
        if (fmt_ctx->streams[i]->codecpar->codec_type == AVMEDIA_TYPE_AUDIO) {
            audio_stream_idx = i;
            break;
        }
    }

    if (audio_stream_idx < 0) {
        av_log(NULL, AV_LOG_ERROR, "No audio stream found in '%s'\n",
               input_path);
        ret = AVERROR_STREAM_NOT_FOUND;
        goto cleanup_open;
    }

    AVStream *in_stream = fmt_ctx->streams[audio_stream_idx];
    AVCodecParameters *in_par = in_stream->codecpar;

    av_log(NULL, AV_LOG_INFO, "Audio stream: %s, %d Hz, %d channels, "
           "bitrate %lld bps\n",
           avcodec_get_name(in_par->codec_id),
           in_par->sample_rate, in_par->channels,
           (long long)in_par->bit_rate);

    /* ─── 4. Find decoder ──────────────────────────────────────── */
    const AVCodec *dec = avcodec_find_decoder(in_par->codec_id);
    if (!dec) {
        av_log(NULL, AV_LOG_ERROR, "Unsupported codec: %s\n",
               avcodec_get_name(in_par->codec_id));
        ret = AVERROR_DECODER_NOT_FOUND;
        goto cleanup_open;
    }

    /* ─── 5. Allocate codec context ─────────────────────────────── */
    AVCodecContext *dec_ctx = avcodec_alloc_context3(dec);
    if (!dec_ctx) {
        av_log(NULL, AV_LOG_ERROR, "Cannot allocate codec context\n");
        ret = AVERROR(ENOMEM);
        goto cleanup_open;
    }

    ret = avcodec_parameters_to_context(dec_ctx, in_par);
    if (ret < 0) {
        av_log(NULL, AV_LOG_ERROR, "Cannot copy codec parameters: %s\n",
               av_err2str(ret));
        goto cleanup_dec_ctx;
    }

    /* Force planar sample format for easier processing */
    dec_ctx->request_sample_fmt = AV_SAMPLE_FMT_FLTP;

    ret = avcodec_open2(dec_ctx, dec, NULL);
    if (ret < 0) {
        av_log(NULL, AV_LOG_ERROR, "Cannot open codec: %s\n",
               av_err2str(ret));
        goto cleanup_dec_ctx;
    }

    /* ─── 6. Open output WAV file ──────────────────────────────── */
    FILE *out_file = fopen(output_path, "wb");
    if (!out_file) {
        av_log(NULL, AV_LOG_ERROR, "Cannot open output '%s'\n", output_path);
        ret = AVERROR(EIO);
        goto cleanup_dec_ctx;
    }

    /* Determine bits per sample from codec context or input params */
    int bits_per_sample = av_get_bytes_per_sample(dec_ctx->sample_fmt) * 8;
    if (bits_per_sample <= 0) bits_per_sample = 16;
    /* Clamp to supported WAV PCM bit depths */
    if (bits_per_sample < 16) bits_per_sample = 16;

    /* Write WAV header with unknown data size (will fix up later) */
    ret = write_wav_header(fmt_ctx,
                           dec_ctx->sample_rate > 0 ? dec_ctx->sample_rate : in_par->sample_rate,
                           dec_ctx->channels > 0 ? dec_ctx->channels : in_par->channels,
                           bits_per_sample);
    if (ret < 0) {
        av_log(NULL, AV_LOG_ERROR, "Cannot write WAV header\n");
        goto cleanup_out;
    }

    /* ─── 7. Demux + Decode loop ───────────────────────────────── */
    AVPacket *pkt = av_packet_alloc();
    AVFrame *frame = av_frame_alloc();
    if (!pkt || !frame) {
        av_log(NULL, AV_LOG_ERROR, "Cannot allocate packet or frame\n");
        ret = AVERROR(ENOMEM);
        goto cleanup_pkt_frame;
    }

    int64_t total_samples = 0;
    int eof = 0;

    while (1) {
        if (!eof) {
            ret = av_read_frame(fmt_ctx, pkt);
            if (ret < 0) {
                if (ret == AVERROR_EOF) {
                    eof = 1;
                    av_log(NULL, AV_LOG_INFO, "End of input reached\n");
                } else {
                    av_log(NULL, AV_LOG_ERROR,
                           "Error reading frame: %s\n", av_err2str(ret));
                }
                /* Send a NULL packet to flush the decoder */
                avcodec_send_packet(dec_ctx, NULL);
            } else if (pkt->stream_index != audio_stream_idx) {
                av_packet_unref(pkt);
                continue;
            } else {
                ret = avcodec_send_packet(dec_ctx, pkt);
                av_packet_unref(pkt);
                if (ret < 0) {
                    av_log(NULL, AV_LOG_ERROR,
                           "Error sending packet to decoder: %s\n",
                           av_err2str(ret));
                    break;
                }
            }
        } else {
            /* At EOF, try to drain the decoder */
            ret = avcodec_send_packet(dec_ctx, NULL);
            if (ret < 0) {
                if (ret == AVERROR_EOF) {
                    break;
                }
            }
        }

        while (ret >= 0) {
            ret = avcodec_receive_frame(dec_ctx, frame);
            if (ret == AVERROR(EAGAIN)) {
                ret = 0;
                break;
            } else if (ret == AVERROR_EOF) {
                av_log(NULL, AV_LOG_INFO, "Decoder flushed, EOF\n");
                eof = 1;
                break;
            } else if (ret < 0) {
                av_log(NULL, AV_LOG_ERROR,
                       "Error receiving decoded frame: %s\n",
                       av_err2str(ret));
                eof = 1;
                break;
            }

            /* Write decoded PCM samples to WAV */
            int nb_samples = frame->nb_samples;
            int channels = frame->channels;
            if (frame->format == AV_SAMPLE_FMT_FLTP) {
                /* Planar float — deinterleave to interleaved 16-bit */
                for (int s = 0; s < nb_samples; s++) {
                    for (int ch = 0; ch < channels; ch++) {
                        float *plane = (float *)frame->data[ch];
                        float sample = plane[s];
                        /* Clamp and convert to 16-bit */
                        if (sample > 1.0f) sample = 1.0f;
                        if (sample < -1.0f) sample = -1.0f;
                        int16_t out = (int16_t)(sample * 32767.0f);
                        if (fwrite(&out, 2, 1, out_file) != 1) {
                            av_log(NULL, AV_LOG_ERROR,
                                   "Write error at sample %lld\n",
                                   (long long)total_samples);
                            ret = AVERROR(EIO);
                            eof = 1;
                            break;
                        }
                    }
                    total_samples++;
                }
            } else if (frame->format == AV_SAMPLE_FMT_FLT) {
                /* Interleaved float */
                float *data = (float *)frame->data[0];
                for (int s = 0; s < nb_samples * channels; s++) {
                    float sample = data[s];
                    if (sample > 1.0f) sample = 1.0f;
                    if (sample < -1.0f) sample = -1.0f;
                    int16_t out = (int16_t)(sample * 32767.0f);
                    if (fwrite(&out, 2, 1, out_file) != 1) {
                        av_log(NULL, AV_LOG_ERROR, "Write error\n");
                        ret = AVERROR(EIO);
                        eof = 1;
                        break;
                    }
                }
                total_samples += nb_samples;
            } else if (frame->format == AV_SAMPLE_FMT_S16P) {
                /* Planar signed 16-bit */
                for (int s = 0; s < nb_samples; s++) {
                    for (int ch = 0; ch < channels; ch++) {
                        int16_t *plane = (int16_t *)frame->data[ch];
                        if (fwrite(&plane[s], 2, 1, out_file) != 1) {
                            ret = AVERROR(EIO);
                            eof = 1;
                            break;
                        }
                    }
                    total_samples++;
                }
            } else if (frame->format == AV_SAMPLE_FMT_S16) {
                /* Interleaved signed 16-bit — direct copy */
                size_t written = fwrite(frame->data[0], 2,
                                        nb_samples * channels, out_file);
                if (written != (size_t)(nb_samples * channels)) {
                    ret = AVERROR(EIO);
                    eof = 1;
                }
                total_samples += nb_samples;
            } else {
                /* Fallback: use swr_convert for any other format */
                static struct SwrContext *swr = NULL;
                if (!swr) {
                    swr = swr_alloc();
                    av_opt_set_int(swr, "in_channel_layout",
                                   frame->channel_layout, 0);
                    av_opt_set_int(swr, "in_sample_fmt",
                                   frame->format, 0);
                    av_opt_set_int(swr, "in_sample_rate",
                                   frame->sample_rate, 0);
                    av_opt_set_int(swr, "out_channel_layout",
                                   frame->channel_layout, 0);
                    av_opt_set_int(swr, "out_sample_fmt",
                                   AV_SAMPLE_FMT_S16, 0);
                    av_opt_set_int(swr, "out_sample_rate",
                                   frame->sample_rate, 0);
                    swr_init(swr);
                }
                uint8_t *out_buf = NULL;
                int out_samples = swr_convert(swr, &out_buf,
                                              nb_samples,
                                              (const uint8_t **)frame->data,
                                              nb_samples);
                if (out_samples > 0) {
                    fwrite(out_buf, 2, out_samples * channels, out_file);
                    total_samples += out_samples;
                }
            }

            av_frame_unref(frame);
            if (eof) break;
        }

        if (eof) break;
    }

    av_log(NULL, AV_LOG_INFO, "Total samples written: %lld\n",
           (long long)total_samples);

    /* ─── 8. Fix up WAV header with final data size ─────────── */
    int64_t data_size = total_samples *
                        (dec_ctx->channels > 0 ? dec_ctx->channels :
                         in_par->channels) *
                        2; /* 2 bytes per sample (16-bit) */
    ret = finish_wav_header(fmt_ctx, data_size);
    if (ret < 0) {
        av_log(NULL, AV_LOG_WARNING, "Could not update WAV header size\n");
    }

    /* ─── Cleanup ─────────────────────────────────────────────── */
cleanup_pkt_frame:
    av_frame_free(&frame);
    av_packet_free(&pkt);

cleanup_out:
    if (out_file) fclose(out_file);

cleanup_dec_ctx:
    avcodec_free_context(&dec_ctx);

cleanup_open:
    avformat_close_input(&fmt_ctx);

    if (ret < 0 && ret != AVERROR_EOF) {
        av_log(NULL, AV_LOG_ERROR, "Conversion failed: %s\n",
               av_err2str(ret));
        return 1;
    }

    av_log(NULL, AV_LOG_INFO, "Done. Wrote '%s'\n", output_path);
    return 0;
}
```

Compile with:
```bash
gcc -o mp4_to_wav mp4_to_wav.c \
  $(pkg-config --cflags --libs libavformat libavcodec libavutil libswresample) \
  -lavformat -lavcodec -lavutil -lswresample -lm -lz -lpthread
```

---

## 10. PROTOCOLS AND I/O

### 10.1 URLProtocol Handlers

FFmpeg registers protocol handlers at compile time via static constructors. Each protocol implements the `URLProtocol` interface:

```c
typedef struct URLProtocol {
    const char *name;
    int (*url_open)(URLContext *h, const char *url, int flags,
                    AVDictionary **options);
    int (*url_read)(URLContext *h, unsigned char *buf, int size);
    int (*url_write)(URLContext *h, const unsigned char *buf, int size);
    int64_t (*url_seek)(URLContext *h, int64_t pos, int whence);
    int (*url_close)(URLContext *h);
    int (*url_read_pause)(URLContext *h, int pause);
    int64_t (*url_read_seek)(URLContext *h, int stream_index,
                              int64_t timestamp, int flags);
    int (*url_get_file_handle)(URLContext *h);
    int (*url_shutdown)(URLContext *h, int type, int press);
    int flags;
    int (*url_check)(URLContext *h, int mask, int flags);
    int (*url_open_dir)(URLContext *h);
    int (*url_read_dir)(URLContext *h, AVIODirEntry **next);
    int (*url_close_dir)(URLContext *h);
} URLProtocol;
```

The `flags` in `url_open` include `AVIO_FLAG_READ`, `AVIO_FLAG_WRITE`, and `AVIO_FLAG_NONBLOCK`.

### 10.2 Custom I/O (AVIOContext)

Implementing custom I/O is essential for non-file sources. The most common use cases are:

**Memory-backed I/O:** Reading from or writing to a memory buffer:
```c
static int mem_read(void *opaque, uint8_t *buf, int buf_size) {
    MemContext *ctx = opaque;
    int len = FFMIN(buf_size, ctx->size - ctx->pos);
    if (len <= 0) return AVERROR_EOF;
    memcpy(buf, ctx->data + ctx->pos, len);
    ctx->pos += len;
    return len;
}

static int64_t mem_seek(void *opaque, int64_t pos, int whence) {
    MemContext *ctx = opaque;
    int64_t new_pos;
    if (whence == SEEK_SET) new_pos = pos;
    else if (whence == SEEK_CUR) new_pos = ctx->pos + pos;
    else if (whence == SEEK_END) new_pos = ctx->size + pos;
    else return AVERROR(EINVAL);
    if (new_pos < 0 || new_pos > ctx->size) return AVERROR(EINVAL);
    ctx->pos = new_pos;
    return new_pos;
}

uint8_t *buffer = av_malloc(max_size);
AVIOContext *avio = avio_alloc_context(
    buffer, max_size, 0, ctx,
    mem_read, NULL, mem_seek);
fmt_ctx->pb = avio;
fmt_ctx->flags |= AVFMT_FLAG_CUSTOM_IO;
```

**Decrypting I/O:** Wrap the underlying I/O with a decryption layer that intercepts reads and decrypts data in-place.

**Network streaming:** Implement a custom protocol that reads from a network socket, reassembles MPEG-TS packets, and passes them to the demuxer.

### 10.3 Pipes and Streaming

Pipes are a common I/O model for inter-process communication in FFmpeg:

```bash
# Encode and pipe to another process
ffmpeg -i input.flac -c:a pcm_f32le -f f32le pipe:1 | some_processor

# Decode from pipe
ffmpeg -i pipe:0 -c:a flac output.flac

# Stream over network using UDP
ffmpeg -i input.wav -c:a flac -f flac udp://localhost:1234

# Stream over TCP
ffmpeg -i input.wav -c:a flac -f flac tcp://localhost:1234?listen

# RTSP streaming
ffmpeg -i input.wav -c:a aac -f rtsp -rtsp_transport tcp rtsp://server/live
```

The `pipe:` protocol uses anonymous pipes (on Unix) or stdin/stdout (in FFmpeg's model). The buffer size for pipe I/O is typically 32 KB.

### 10.4 Byte Serving (HTTP Range Requests)

HTTP streaming with seeking requires Range request support:

```bash
# Read a segment of a file via HTTP range
curl -H "Range: bytes=0-1023" http://example.com/audio.flac -o first_kb.flac
```

FFmpeg's HTTP protocol supports Range requests automatically when seeking in an HTTP stream that supports byte serving. Set `seekable=0` to disable seeking if the server doesn't support it:

```c
av_dict_set(&opts, "seekable", "0", 0);
avformat_open_input(&fmt_ctx, "http://...", NULL, &opts);
```

For HLS streaming, FFmpeg handles segment downloading and concatenation automatically:

```bash
ffmpeg -i "https://example.com/playlist.m3u8" -c:a copy output.aac
```

---

## 11. SEEKING IN CONTAINERS

### 11.1 av_seek_frame Behavior

`av_seek_frame` is the primary seeking interface:

```c
int av_seek_frame(AVFormatContext *s, int stream_index,
                  int64_t timestamp, int flags);
```

The function behavior depends on the `flags`:

| Flag | Behavior |
|---|---|
| `AVSEEK_FLAG_BACKWARD` | Seek to nearest keyframe BEFORE the target. The decoder starts from there and discards until the target. Most reliable. |
| `AVSEEK_FLAG_BYTE` | Seek by byte offset. `timestamp` is treated as a byte offset. Sets `fmt_ctx->pb->pos` directly. |
| `AVSEEK_FLAG_ANY` | Seek to any frame (keyframe or non-keyframe). May produce garbled output for video with temporal prediction. Use only for audio where all frames are independently decodable. |
| `AVSEEK_FLAG_FRAME` | Seek to nearest frame number. `timestamp` is interpreted as a frame number in the target stream. |

For audio streams, `AVSEEK_FLAG_BACKWARD` is the standard choice because audio frames are independently decodable (no temporal prediction dependency), so the decoder can start from the nearest prior keyframe and produce correct output.

### 11.2 Seeking in MP4

MP4 seeking relies on several atom tables:

- **`stss` (Sync Sample):** Lists which samples are keyframes (random access points).
- **`stsc` (Sample-to-Chunk):** Maps samples to chunks.
- **`stco` / `co64` (Chunk Offset):** Byte offsets of each chunk in the file.
- **`stsz` (Sample Size):** Size of each sample.
- **`stts` (Time-to-Sample):** Maps sample numbers to timestamps (duration table).
- **`ctts` (Composition Time-to-Sample):** Offset between decode time and presentation time (for B-frames).

To seek to timestamp `T`:
1. Use `stts` to find the sample number `S` whose PTS contains `T`.
2. Use `stss` to find the nearest sync sample `K` before or at `S`.
3. Use `stsc` to find the chunk containing `K`.
4. Use `stco`/`co64` to find the byte offset of that chunk.
5. Seek the I/O context to that byte offset.
6. Resume reading frames from there, discarding until `S`.

The `AVFMT_FLAG_NOFILLIN` flag can be used to skip building some of these tables, trading seeking speed for faster startup.

**`+faststart` reordering:** By default, the `moov` atom is written at the end of the file. The `+faststart` flag (via `movflags`) relocates the `moov` to the beginning after `write_trailer`, enabling streaming playback without downloading the entire file.

### 11.3 Seeking in FLAC

FLAC seeking uses the **SEEKTABLE** metadata block (optional but recommended):

```
seektable block:
  seek point 1:  sample_number (64-bit), stream_offset (64-bit), frame_size (16-bit)
  seek point 2:  ...
  ...
```

Without a seektable, FLAC seeking requires:
1. Binary search within the file: seek to midpoint, read frame headers until you find a sync pattern.
2. Use the `min_blocksize` and `max_blocksize` from `STREAMINFO` to bound the search.
3. Resync and decode until the target sample number is reached.

This can be slow for large files. Always encode FLAC with a seektable:

```bash
ffmpeg -i input.wav -c:a flac -write_seektable 128 output.flac
```

The number `128` specifies the number of seek points. More points = faster seeking but larger seektable metadata block.

### 11.4 Seeking in OGG

OGG seeking scans pages from the last keyframe. Each OGG page header contains:
- `granulepos`: The presentation time of the last sample in the page (in the codec's time base).
- `page_sequence`: Sequential page number.
- `BOS` flag: Beginning of stream (first page of a logical bitstream).
- `EOS` flag: End of stream.
- `keyframe` flag: Whether this page starts with a keyframe.

Seeking in OGG:
1. The muxer stores a "skew" factor (the first granule position's time).
2. Binary search through pages using `page_sequence`.
3. Use `granulepos` to compute the page's timestamp.
4. When the target is bracketed, scan forward to the nearest keyframe page.

OGG seeking without a seek index is inherently imprecise and slow. The `ogg` muxer in FFmpeg does not write a seek index.

### 11.5 Seeking in WAV

WAV seeking is straightforward because WAV is essentially a sequential byte stream of PCM samples:

1. Use the `fmt ` chunk to determine `nChannels`, `bitsPerSample`, `sampleRate`.
2. Calculate bytes per sample: `nChannels × bitsPerSample / 8`.
3. Use the `data` chunk's size to determine total sample count.
4. `byte_offset = data_start + sample_number × bytes_per_sample`.
5. `fseek()` to the computed offset and read from there.

No index is needed because every sample in WAV is a random access point (PCM has no temporal compression).

### 11.6 Keyframe-Seek Requirements

For video, seeking requires keyframe alignment because P-frames and B-frames depend on previously decoded frames. The decoder cannot reconstruct a non-keyframe without first decoding all prior reference frames.

For audio, most formats have independently decodable frames, making every frame a potential seek point. However, there are exceptions:

- **Opus:** Uses lookahead, so a small number of samples before the seek point must be discarded.
- **AAC HE-AAC:** Uses SBR (Spectral Band Replication), which requires the SBR data preceding the target frame.
- **MP3 with bit reservoir:** The bit reservoir allows bits from one frame to be used in another, meaning a frame may depend on prior frame data. However, the dependency is minor — most decoders handle this automatically.
- **ATRAC9:** Uses backward prediction, requiring prior frames.

FFmpeg handles these edge cases via the `seek_preroll` field in `AVCodecParameters` and the `AV_PKT_DATA_SKIP_SAMPLES` side data.

---

## 12. INTERLEAVING

### 12.1 Interleaved vs Non-Interleaved Writing

**Non-interleaved writing** writes all data for stream 0, then all data for stream 1, etc. This produces a file where all audio frames are consecutive, followed by all video frames. This is simple but breaks playback: the player would need to buffer all audio frames before starting to play, or would need to seek to find the start of the audio data.

**Interleaved writing** mixes packets from different streams by presentation time. This ensures that a sequential read of the file naturally produces packets in the correct temporal order, enabling immediate playback start and seamless seeking.

```
Non-interleaved (2 audio streams, 3 packets each):
  A1 A2 A3 B1 B2 B3

Interleaved (alternating by timestamp):
  A1 B1 A2 B2 A3 B3
```

### 12.2 av_interleaved_write_frame vs av_write_frame

**`av_interleaved_write_frame()`** — The safe, recommended choice:
- Internally buffers one packet per stream.
- Before writing, compares the incoming packet's PTS with buffered packets.
- Writes out all packets whose PTS is now in the past.
- Buffers the incoming packet until its presentation time arrives.
- Handles DTS computation for formats that need it.

**`av_write_frame()`** — For advanced use:
- Writes packets in the order you call it.
- You must ensure correct temporal ordering yourself.
- Bypasses the interleaver's buffering.
- Useful for:
  - Single-stream outputs (no interleaving needed).
  - Custom packet scheduling (e.g., writing all keyframes before non-keyframes for archival purposes).
  - Testing or debugging.
  - When the caller manages its own buffering.

### 12.3 Packet Scheduling

The interleaver uses a simple algorithm:

```
For each incoming packet P with PTS = t:
  1. Determine the target stream S = P->stream_index.
  2. Compare t with the buffered packet's PTS for other streams.
  3. If any other stream has a buffered packet with PTS < t,
     write that packet (and remove it from the buffer).
  4. If the target stream S has a buffered packet,
     compare it with P.
     - If P's PTS < buffered PTS, write buffered and store P.
     - If P's PTS >= buffered PTS, store P and write P.
  5. If no comparison needed (first packet), store it.
```

The muxer guarantees that the output file's packets are ordered by non-decreasing PTS, with the constraint that each stream's relative ordering is preserved.

**Performance implications:** The interleaver's internal buffer can grow if packets arrive out of order. For real-time streaming, this can cause unbounded delay. Use `av_write_frame()` for real-time applications where latency matters more than perfect interleaving.

---

## 13. ERROR HANDLING

### 13.1 Error Codes

FFmpeg uses negative AVERROR codes, defined in `libavutil/error.h`:

| Error Code | Value | Meaning |
|---|---|---|
| `AVERROR_EOF` | `FFERRTAG('E','O','F',' ')` | End of file reached |
| `AVERROR_DEMUXER_ERROR` | Custom | Demuxer encountered an error |
| `AVERROR_ENCODER_NOT_FOUND` | `MKTAG('E','N','C','N')` | No encoder found for codec |
| `AVERROR_DECODER_NOT_FOUND` | `MKTAG('D','E','C','N')` | No decoder found for codec |
| `AVERROR_INVALIDDATA` | `FFERRTAG('I','N','D','A')` | Invalid data found (malformed input) |
| `AVERROR_NOMEM` | `MKTAG('N','O','M','E')` | Not enough memory |
| `AVERROR_UNKNOWN` | `MKTAG('U','N','K','N')` | Unknown error |
| `AVERROR_STREAM_NOT_FOUND` | Custom | No stream matching criteria |
| `AVERROR_DECODE_ERROR` | Custom | Decoder error (corrupt frame) |
| `AVERROR_ENCODER_ERROR` | Custom | Encoder error |
| `AVERROR_EXIT` | `FFERROR_REDO` | Operation interrupted by callback |
| `AVERROR(EAGAIN)` | `FFERRTAG('E','A','G','A')` | Resource temporarily unavailable — retry |

**Checking errors:**
```c
if (ret < 0) {
    if (av_err2str(ret)) {
        av_log(NULL, AV_LOG_ERROR, "Error: %s\n", av_err2str(ret));
    }
    // Determine error category
    if (ret == AVERROR_EOF) { /* normal end */ }
    else if (ret == AVERROR_INVALIDDATA) { /* corrupt file */ }
    else if (ret == AVERROR_NOMEM) { /* memory failure */ }
}
```

### 13.2 Recovery Strategies

**Corrupt packets:** When `av_read_frame()` returns `AVERROR_INVALIDDATA`, the packet is corrupt. Options:
- Skip the packet and continue.
- Attempt to resync by searching for the next valid sync pattern.
- Mark the stream as corrupted and stop.

```c
ret = av_read_frame(fmt_ctx, pkt);
if (ret == AVERROR_INVALIDDATA) {
    av_log(NULL, AV_LOG_WARNING, "Skipping corrupt packet\n");
    av_packet_unref(pkt);
    continue;
} else if (ret < 0) {
    break;
}
```

**Incomplete streams:** When a stream is truncated, `av_read_frame()` eventually returns `AVERROR_EOF`. The muxer may have buffered packets — call `av_interleaved_write_frame(NULL)` to drain them before writing the trailer.

**Decoder errors:** When `avcodec_receive_frame()` returns `AVERROR_INVALIDDATA` or `AVERROR_DECODE_ERROR`, the frame is corrupt. Skip it and continue:

```c
ret = avcodec_receive_frame(dec_ctx, frame);
if (ret == AVERROR_INVALIDDATA || ret == AVERROR_DECODE_ERROR) {
    av_log(NULL, AV_LOG_WARNING, "Skipping corrupt frame\n");
    av_frame_unref(frame);
    continue;
}
```

**Custom interrupt callback:** Implement cooperative cancellation:

```c
static volatile int g_cancel = 0;

static int interrupt_callback(void *ctx) {
    return g_cancel;
}

fmt_ctx->interrupt_callback.callback = interrupt_callback;
fmt_ctx->interrupt_callback.opaque = NULL;

// In another thread or signal handler:
g_cancel = 1;
```

### 13.3 Incomplete Files

Incomplete files (still being written, or cut off mid-download) require careful handling:

1. **Probe cautiously:** Set `probesize` to a small value to avoid reading too far.
2. **Check `fmt_ctx->duration`:** May be unreliable for incomplete files.
3. **Use `AVFMT_FLAG_IGNIDX`:** Ignore corrupted indexes.
4. **Handle partial metadata:** Some metadata may be readable even if the file is truncated.

For writing, if the process is interrupted mid-write:
1. The container footer may not be written.
2. The file may be unreadable by players.
3. Some formats (WAV, FLAC) have size fields in the header that must be updated at the end.

**Defensive writing:** For critical applications, write to a temporary file and rename on successful completion:

```bash
ffmpeg -i input.wav -c:a flac output.flac.tmp && mv output.flac.tmp output.flac
```

---

## 14. PLATFORM & BUILD NOTES

### 14.1 Required ./configure Flags

When building FFmpeg from source, ensure these flags are enabled for full audio format support:

```bash
./configure \
  --enable-gpl \
  --enable-libx264 \
  --enable-libx265 \
  --enable-libvpx \
  --enable-libmp3lame \
  --enable-libvorbis \
  --enable-libopus \
  --enable-libspeex \
  --enable-libflac \
  --enable-libfdk-aac \
  --enable-libaacplus \
  --enable-libass \
  --enable-libbluray \
  --enable-libwebp \
  --enable-librtmp \
  --enable-libsoxr \
  --enable-version3 \
  --enable-nonfree \
  --disable-doc \
  --disable-programs
```

For a minimal build with audio focus:

```bash
./configure \
  --enable-gpl \
  --enable-libmp3lame \
  --enable-libvorbis \
  --enable-libopus \
  --enable-libflac \
  --enable-libfdk-aac \
  --enable-libspeex \
  --enable-libsoxr \
  --enable-version3 \
  --disable-doc
```

### 14.2 Linker Flags

When linking an application against libavformat:

```bash
# Using pkg-config
pkg-config --cflags --libs libavformat libavcodec libavutil libswresample

# Direct linking
-lavformat -lavcodec -lavutil -lswresample -lm -lz -lpthread

# Full static linking (requires all dependencies statically linked)
-Wl,--whole-archive -lavformat -lavcodec -lavutil -lswresample \
  -Wl,--no-whole-archive -lm -lz -lpthread -lstdc++
```

### 14.3 Header Files

The core headers for libavformat programming:

```c
#include <libavformat/avformat.h>    // AVFormatContext, avformat_open_input, etc.
#include <libavcodec/avcodec.h>       // AVCodec, AVCodecContext, AVPacket
#include <libavutil/avutil.h>         // av_err2str, av_malloc, etc.
#include <libavutil/channel_layout.h> // AV_CH_LAYOUT_*, av_get_channel_layout_nb_channels
#include <libavutil/samplefmt.h>      // AV_SAMPLE_FMT_*, av_get_bytes_per_sample
#include <libavutil/opt.h>            // av_dict_set, av_opt_*
#include <libswresample/swresample.h> // SwrContext, swr_convert
```

---

## 15. VERSIONING & API CHANGES

### 15.1 Major API Changes

**FFmpeg 4.0 (2018):**
- Introduced `AVCodecParameters` (`codecpar`) to replace the deprecated `codec` pointer in `AVStream`. `avcodec_parameters_to_context()` and `avcodec_parameters_copy()` were added.
- Changed `AVPacket` to use reference-counted buffers (`AVBufferRef *buf`) instead of direct allocation. `av_packet_unref()` became mandatory.
- Added `av_packet_alloc()` and `av_packet_free()` for packet lifecycle management.
- Deprecated `av_free()` for `AVPacket->data`.

**FFmpeg 4.1 (2019):**
- Added `avformat_flush()` for non-destructive flushing.
- Added `avformat_close_input()` (already existed but behavior was clarified).
- Added `AVStreamGroup` for grouped streams (e.g., multiple audio channels as one group).

**FFmpeg 4.2 (2019):**
- Added `av_stream_get_first_dts()` to retrieve the first DTS.
- Added support for `side_data` injection in `avformat_write_header()`.

**FFmpeg 4.3 (2020):**
- `avcodec_send_packet()` can now receive `NULL` to signal EOF to the decoder, equivalent to flushing.
- Added `av_interleaved_write_frame()` can be called with `NULL` to drain the interleaver.

**FFmpeg 4.4 (2021):**
- Added `avformat_network_init()` for explicit network initialization (needed on some platforms).
- Added `av_stream_side_data_new()` for managing stream side data.

**FFmpeg 5.0 (2022):**
- New `AVCodec` plugin API (`AVCodecHWConfigInternal`, `AVHWDeviceContext`).
- `avcodec_decode_audio4()` was fully removed (it was deprecated since 4.0).
- `avcodec_decode_video2()` was fully removed.

**FFmpeg 6.0 (2023):**
- Removed deprecated `avcodec_decode_audio4()` remnants.
- Removed old channel layout API (`av_get_channel_layout_*` was replaced by newer functions).
- `AVPixelFormat` was replaced by `enum AVPixelFormat` renamed to `AVPixelFormat` throughout.

**FFmpeg 7.0 (2024):**
- `AVFrame` `channel_layout` replaced with per-plane channel layout.
- New send/receive API is now the only decode/encode interface.
- `av_read_play()` and `av_read_pause()` are deprecated for most formats.

### 15.2 Migration Guide

**From FFmpeg < 4.0 to 4.0+:**

Old pattern:
```c
AVCodecContext *dec_ctx = fmt_ctx->streams[i]->codec;
```

New pattern:
```c
AVCodecParameters *par = fmt_ctx->streams[i]->codecpar;
AVCodecContext *dec_ctx = avcodec_alloc_context3(dec);
avcodec_parameters_to_context(dec_ctx, par);
```

Old packet handling:
```c
AVPacket pkt;
// Initialize and use directly
av_free(pkt.data); // WRONG in FFmpeg 4.0+
```

New packet handling:
```c
AVPacket *pkt = av_packet_alloc();
// ...
av_packet_unref(pkt);
av_packet_free(&pkt);
```

**From FFmpeg < 3.0 to 4.0+:**

Old `av_register_all()` was deprecated in 3.0 and removed in 4.0. Formats are now auto-registered via static constructors. Remove any `av_register_all()` calls.

---

## 16. REFERENCE: ALL FORMAT MUXER/DEMUXER OPTIONS

Complete table of format-specific options available for each major format:

### FLAC

| Option | Type | Default | Description |
|---|---|---|---|
| `compression_level` | int | 5 | Encoding compression (0–12) |
| `lpc_coeff_precision` | int | 15 | LPC coefficient precision (4–15) |
| `lpc_type` | int | `FLAC_LPC_TYPE_LEVINSON` | LPC algorithm type |
| `lpc_passes` | int | 2 | Number of LPC analysis passes |
| `min_partition_order` | int | 0 | Minimum partition order |
| `max_partition_order` | int | 6 | Maximum partition order |
| `write_crcsum` | int | 1 | Write CRC-32 in frame header |
| `write_seektable` | str | NULL | Seektable content or template |
| `vendor` | str | "Lavf" | Vendor string in vorbis comment |

### OGG

| Option | Type | Default | Description |
|---|---|---|---|
| `ogg_delay` | int | 0 | Page granule position delay in nanoseconds |
| `ogg_serial` | int | auto | Serial number for the stream |

### MP4 / MOV

| Option | Type | Default | Description |
|---|---|---|---|
| `movflags` | flags | 0 | `+faststart`, `+frag_keyframe`, `+default_base_moof`, `+delay_moov`, `+negative_cts_offsets`, `+disable_chpl`, `+omit_tfhd_offset`, `+non_dash`, `+cast_meta` |
| `brand` | str | `isom` | Major brand (e.g., `M4A`, `iso2`, `qt`) |
| `encryption_scheme` | int | 0 | CENC encryption scheme (4 = AES-CTR) |
| `encryption_key` | bin | NULL | Encryption key (hex) |
| `encryption_kid` | bin | NULL | Key ID (hex) |
| `movie_timescale` | int | 1000 | Timescale for the moov atom |
| `track_timescale` | int | 0 | Use stream's own timescale |
| `trellis` | int | 0 | Enable trellis quantization |
| `write_tmcd` | int | 1 | Write timecode track |
| `frag_size` | int | 0 | Fragment size (for `+frag_keyframe`) |
| `frag_duration` | int | 0 | Fragment duration in time base units |

### Matroska / WebM

| Option | Type | Default | Description |
|---|---|---|---|
| `EBML.head.docType` | str | `matroska` | EBML DocType |
| `EBML.head.docTypeVersion` | int | 4 | DocType version |
| `EBML.head.docTypeReadVersion` | int | 2 | Minimum readable DocType version |
| `timecode_scale` | int | 1000000 | Timestamp scale (nanoseconds) |
| `Cluster_size_limit` | int | -1 | Max cluster size in bytes |
| `Cluster_time_limit` | int | -1 | Max cluster duration in time base |
| `livestream` | int | 0 | Enable live streaming mode |
| `write_cues` | int | 1 | Write Cues element |
| `default_mode` | str | `infer` | Default mode for tracks: `infer`, `infer_no_data`, `probe`, `full` |

### WAV / RF64

| Option | Type | Default | Description |
|---|---|---|---|
| `write_bext` | bool | 0 | Write Broadcast Extension chunk |
| `bext_ description` | str | "" | BEXT description |
| `bext_originator` | str | "" | BEXT originator code |
| `bext_orig_ref` | str | "" | BEXT originator reference |
| `bext_umid` | bin | NULL | BEXT UMID |
| `bext_timestamp` | str | NULL | BEXT timestamp (YYYY-MM-DDTHH:MM:SS) |
| `write_peak` | bool | 0 | Write Peak Envelope chunk |
| `peak_block_size` | int | 0 | Peak envelope block size |
| `peak_format` | int | 2 | Peak format (0=u8, 1=u16, 2=float16) |

### AIFF

| Option | Type | Default | Description |
|---|---|---|---|
| `platform` | str | "FFmpeg" | Platform identifier |
| `write_id3v1` | bool | 0 | Write ID3v1 tag |

### MP3

| Option | Type | Default | Description |
|---|---|---|---|
| `id3v2_version` | int | 3 | ID3v2 version (3 or 4) |
| `write_id3v1` | bool | 1 | Write ID3v1 tag |
| `audio_service_type` | int | 0 | ATSC A/53 AC-3 service type (0=main, 6=visually impaired, etc.) |

### MPEG-TS

| Option | Type | Default | Description |
|---|---|---|---|
| `mpegts_flags` | flags | 0 | `resend_headers`, `latm`, `pat_pmt_at_frames` |
| `mpegts_start_pid` | int | 256 | First PID for streams |
| `mpegts_service_type` | int | `mpegts_services` | Service type: `digital_tv`, `digital_radio`, etc. |
| `service_name` | str | NULL | Service name |
| `provider_name` | str | NULL | Provider name |
| `pcr_period` | int | auto | PCR repetition period |
| `pat_period` | int | 0.1 | PAT repetition period |
| `tsid` | int | 0 | Transport Stream ID |
| `tsid` | int | 0 | Transport Stream ID |
| `onid` | int | 0 | Original Network ID |
| `sid` | int | 0 | Service ID |

### AVI

| Option | Type | Default | Description |
|---|---|---|---|
| `reserve_index_space` | int | 0 | Reserve space for the index at the end |
| `write_index` | int | 1 | Write index at the end |
| `max_index_size` | int | 1048576 | Maximum index size |

### NUT

| Option | Type | Default | Description |
|---|---|---|---|
| `syncpoints` | flags | `default` | `none`, `default`, `timestamped` |
| `write_index` | int | 1 | Write index at end |
| `nuspect_switch` | int | 0 | Switch to NUSPEC |

### CAF

| Option | Type | Default | Description |
|---|---|---|---|
| `caf_version` | int | 1 | CAF file version |
| `mark_sparse` | int | 0 | Mark packets as sparse |
| `packet_size` | int | 0 | Packet size (0 = variable) |

### AAC (raw ADTS)

| Option | Type | Default | Description |
|---|---|---|---|
| `id3v2_version` | int | 3 | ID3v2 version |
| `write_id3v1` | bool | 0 | Write ID3v1 tag |

### AC3

| Option | Type | Default | Description |
|---|---|---|---|
| `per_frame_metadata` | bool | 0 | Allow per-frame metadata |
| `write_aesub` | bool | 1 | Write audio production information |
| `dvm_interpolation` | bool | 1 | Enable dialog normalization |
| `drc_scale` | float | 1.0 | Dynamic Range Control scale factor |

### DTS

| Option | Type | Default | Description |
|---|---|---|---|
| `crc_check` | bool | 1 | Check CRC of core stream |
| `core_only` | bool | 0 | Output core stream only (no extensions) |

---

## 17. IMPLEMENTATION CHECKLIST (for the Converter Developer)

This checklist covers all critical requirements for building a production-grade audio format converter using FFmpeg libavformat:

### Pre-Operation

- [ ] Verify FFmpeg libraries are installed and linked correctly (`pkg-config --modversion libavformat`)
- [ ] Check `avformat_configuration()` output confirms required formats are enabled
- [ ] Test with a small input file before processing the full archive
- [ ] Create a backup of the source file or verify `MusicFolder/` is read-only
- [ ] Implement cooperative interrupt callback for cancellation support
- [ ] Set appropriate `probesize` and `max_analyze_duration` for the input format
- [ ] Handle the case where `avformat_open_input()` returns `AVERROR_INVALIDDATA` (corrupt file)
- [ ] Set `interrupt_callback` on `AVFormatContext` for long-running operations

### Stream Discovery

- [ ] Iterate all streams and filter by `codec_type == AVMEDIA_TYPE_AUDIO`
- [ ] Log codec ID, sample rate, channel count, bitrate, and codec name for each audio stream
- [ ] Handle the case where no audio stream is found (`AVERROR_STREAM_NOT_FOUND`)
- [ ] Check `codecpar->extradata` and `codecpar->extradata_size` for codec initialization data
- [ ] Verify the codec is supported (`avcodec_find_decoder()` returns non-NULL)
- [ ] Check `codecpar->profile` and `codecpar->level` for codec-specific constraints

### Decoder Setup

- [ ] Allocate `AVCodecContext` with `avcodec_alloc_context3()`
- [ ] Copy parameters with `avcodec_parameters_to_context()` (not direct field access)
- [ ] Open decoder with `avcodec_open2()` and handle failure
- [ ] Set `request_sample_fmt` to a known output format if format conversion is needed
- [ ] Handle `AV_CODEC_CAP_VARIABLE_FRAME_SIZE` for codecs with variable frame sizes

### Encoding (if transcoding)

- [ ] Allocate and configure encoder `AVCodecContext`
- [ ] Set correct `sample_rate`, `channel_layout`, `bit_rate`, and `time_base`
- [ ] Set `compression_level` for lossless codecs (FLAC: 8, TAK: max)
- [ ] Set codec-specific options via `AVDictionary` passed to `avcodec_open2()`
- [ ] Set `frame_size` if the encoder requires a specific frame size
- [ ] Handle encoder initialization failures gracefully

### Output Container Setup

- [ ] Use `avformat_alloc_output_context2()` with guessed format from extension
- [ ] If format is ambiguous (e.g., `.mka` vs `.flac`), specify explicitly
- [ ] Open output with `avio_open()` and check for I/O errors
- [ ] Call `avformat_new_stream()` for each output stream
- [ ] Copy codec parameters with `avcodec_parameters_copy()` for pass-through
- [ ] Copy codec parameters with `avcodec_parameters_from_context()` for encoding
- [ ] Set `out_stream->time_base` to the correct value before `write_header()`
- [ ] Set `out_stream->id` if the format requires stream IDs (MP4, MPEG-TS)
- [ ] Copy metadata with `av_dict_copy()` or set explicitly with `av_dict_set()`
- [ ] Write ReplayGain fields if source had them
- [ ] Copy cover art / attached pictures if source had them

### Metadata Handling

- [ ] Map source metadata keys to destination format's key conventions
- [ ] Handle `AV_DICT_APPEND` for multi-value fields
- [ ] Verify that all critical metadata fields (title, artist, album) are preserved
- [ ] Handle custom/private metadata fields (TXXX, PRIV, etc.)
- [ ] Set `encoder` metadata field to identify the conversion tool
- [ ] Handle cover art embedding: FLAC → PICTURE block, MP4 → APIC/attached pic, OGG → Vorbis comment

### Packet Processing

- [ ] Allocate `AVPacket` with `av_packet_alloc()` (never stack-allocate)
- [ ] Call `av_packet_unref()` after every `av_read_frame()` — ALWAYS
- [ ] Scale timestamps with `av_packet_rescale_ts()` before writing
- [ ] Handle `AV_PKT_FLAG_KEY` for keyframe marking
- [ ] Handle `pkt->side_data` — copy or process side data as needed
- [ ] Call `av_interleaved_write_frame()` instead of `av_write_frame()` for multi-stream output
- [ ] Check the return value of every `write_frame()` call
- [ ] On error, unref the packet AND handle cleanup
- [ ] After the loop, drain the interleaver with `av_interleaved_write_frame(NULL)`

### Muxer Finalization

- [ ] Call `av_write_trailer()` — ALWAYS, even on error (attempt it)
- [ ] Close `AVIOContext` with `avio_closep()` — sets pointer to NULL
- [ ] Free `AVFormatContext` with `avformat_free_context()`
- [ ] Free codec contexts with `avcodec_free_context()`
- [ ] Free packets and frames with `av_packet_free()` and `av_frame_free()`
- [ ] Free dictionaries with `av_dict_free()`

### Verification

- [ ] Compare source and destination metadata with `kid3-cli` (see workspace rules)
- [ ] Check for `Invalid UTF-8` or `Ignoring duplicate atom` errors
- [ ] Verify all tag fields are preserved with exact same names and values
- [ ] Verify cover art is present and not corrupted
- [ ] Verify audio duration matches (with tolerance for encoder delay/padding)
- [ ] Run `generate_sfv.py` to verify output integrity
- [ ] Test seeking in the output file (if applicable)
- [ ] Test gapless playback markers if the format supports them

### Error Recovery

- [ ] Handle `AVERROR_INVALIDDATA` (corrupt packets) — skip and continue
- [ ] Handle `AVERROR_EOF` (end of file) — normal completion
- [ ] Handle `AVERROR_NOMEM` — report and abort
- [ ] Handle `AVERROR_DECODER_NOT_FOUND` — report and abort with clear message
- [ ] Implement cleanup-on-error path that frees ALL allocated resources
- [ ] Ensure partial output files are removed on error (or accept them as-is)
- [ ] Log all errors with `av_err2str()` for debugging

### Performance Considerations

- [ ] Set appropriate buffer sizes for `AVIOContext` (default 32 KB, increase for HDD)
- [ ] Use `avformat_flush()` if restarting a decode from a seek point
- [ ] For large files, process in chunks rather than loading entire file into memory
- [ ] Consider `avcodec_send_packet()` / `avcodec_receive_frame()` efficiency: send multiple packets before receiving to fill the decoder buffer
- [ ] Use `av_frame_unref()` immediately after processing a frame (don't hold references)
- [ ] Use `-threads 1` for the decoder if thread safety is a concern in your application
