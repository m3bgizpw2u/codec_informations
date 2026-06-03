# FFmpeg Error Detection and Recovery — Deep Technical Reference

> **Category:** FFmpeg API
> **File Extensions:** N/A (FFmpeg API reference)
> **MIME Types:** N/A
> **Standardization Body:** FFmpeg Project
> **Primary Specification:** FFmpeg source code (`libavutil/error.h`, `libavutil/error.c`, `libavcodec/`)
> **Patent Status:** N/A
> **License:** LGPL 2.1+
> **Current Version:** FFmpeg 7.x (ongoing development)
> **Active Development:** Yes — ongoing

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation

- **Creator(s):** FFmpeg Project
- **Year Created:** FFmpeg error handling evolved from early 2000s as the library matured
- **Original Purpose:** Provide consistent, informative error reporting across FFmpeg's modular architecture
- **Problem with Predecessors:** Early multimedia libraries had inconsistent error codes; FFmpeg unified error handling with AVERROR system

### 1.2 Version History

| Version | Year | Key Changes |
|---------|------|-------------|
| FFmpeg 0.4 | 2003 | Basic error codes introduced |
| FFmpeg 0.6 | 2010 | Expanded AVERROR constants |
| FFmpeg 2.0 | 2013 | Improved error context reporting |
| FFmpeg 4.0 | 2018 | Enhanced error concealment APIs |
| FFmpeg 5.0 | 2022 | Better interrupt callback handling |
| FFmpeg 6.0 | 2023 | Improved error diagnostics |
| FFmpeg 7.0 | 2024 | Continued error handling improvements |

### 1.3 Current Adoption

- **Primary use cases today:** Robust media processing applications, streaming services, video editors, transcoding pipelines
- **Platforms with native support:** All platforms with FFmpeg
- **Major tools using this:** FFmpeg itself, libav, VLC, HandBrake, GStreamer integration
- **Status:** Mature, well-documented error system

---

## 2. FFMPEG ERROR CODE SYSTEM

### 2.1 AVERROR Macro System

FFmpeg uses a unified error code system based on the `AVERROR` macro:

```c
#include <libavutil/error.h>

// Macro to create error codes
#define AVERROR(e) (e)

// Example error codes
#define AVERROR_EOF           FFERRTAG('E','O','F',' ')     // End of file
#define AVERROR_INVALIDDATA   FFERRTAG('I','N','D','A')    // Invalid data
#define AVERROR_DECODER_NOT_FOUND FFERRTAG(0xF8,'D','E','C')  // Decoder not found
#define AVERROR_ENCODER_NOT_FOUND FFERRTAG(0xF8,'E','N','C')  // Encoder not found
#define AVERROR_DEMUXER_NOT_FOUND FFERRTAG(0xF8,'D','E','M') // Demuxer not found
#define AVERROR_MUXER_NOT_FOUND   FFERRTAG(0xF8,'M','U','X')  // Muxer not found
#define AVERROR_STREAM_NOT_FOUND  FFERRTAG(0xF8,'S','T','R')  // Stream not found
#define AVERROR_FILTER_NOT_FOUND  FFERRTAG(0xF8,'F','I','L')  // Filter not found
#define AVERROR_BSF_NOT_FOUND     FFERRTAG(0xF8,'B','S','F')  // BSF not found
#define AVERROR_OPTION_NOT_FOUND  FFERRTAG(0xF8,'O','P','T')  // Option not found
#define AVERROR_PROTOCOL_NOT_FOUND FFERRTAG(0xF8,'P','R','O') // Protocol not found
#define AVERROR_UNKNOWN          FFERRTAG('U','N','K','N')    // Unknown error
#define AVERROR_EXTERNAL         FFERRTAG('E','X','T',' ')   // External library error
#define AVERROR_INPUT_CHANGED    FFERRTAG('I','N','C','H')   // Input changed
#define AVERROR_OUTPUT_CHANGED  FFERRTAG('O','U','C','H')   // Output changed
#define AVERROR_BUG              FFERRTAG('B','U','G','!')   // Internal bug
#define AVERROR_BUG2            FFERRTAG('B','U','G','2')   // Another internal bug
#define AVERROR_BUFFER_TOO_SMALL FFERRTAG('B','U','F','S')  // Buffer too small
#define AVERROR_ENCODER_NOT_FOUND FFERRTAG(0xF8,'E','N','C') // Encoder not found
#define AVERROR_EXIT            FFERRTAG('E','X','I','T')  // Immediate exit requested
#define AVERROR_PATCHWELCOME    FFERRTAG('P','A','T','C')  // Not implemented, patches welcome
```

### 2.2 Complete AVERROR Code Table

| Constant | Tag String | Numeric Value | Meaning |
|----------|------------|---------------|---------|
| `AVERROR_EOF` | "EOF " | Various | End of file reached |
| `AVERROR_INVALIDDATA` | "INDA" | Various | Invalid data in input |
| `AVERROR_DECODER_NOT_FOUND` | "DEC " | Various | Requested decoder unavailable |
| `AVERROR_ENCODER_NOT_FOUND` | "ENC " | Various | Requested encoder unavailable |
| `AVERROR_DEMUXER_NOT_FOUND` | "DEM " | Various | Demuxer not found |
| `AVERROR_MUXER_NOT_FOUND` | "MUX " | Various | Muxer not found |
| `AVERROR_STREAM_NOT_FOUND` | "STR " | Various | Requested stream not found |
| `AVERROR_FILTER_NOT_FOUND` | "FIL " | Various | Filter not found |
| `AVERROR_BSF_NOT_FOUND` | "BSF " | Various | Bitstream filter not found |
| `AVERROR_OPTION_NOT_FOUND` | "OPT " | Various | Option not found |
| `AVERROR_PROTOCOL_NOT_FOUND` | "PRO " | Various | Protocol not found |
| `AVERROR_UNKNOWN` | "UNKN" | Various | Unknown/internal error |
| `AVERROR_EXTERNAL` | "EXT " | Various | Error from external library |
| `AVERROR_INPUT_CHANGED` | "INCH" | Various | Input parameters changed |
| `AVERROR_OUTPUT_CHANGED` | "OUCH" | Various | Output parameters changed |
| `AVERROR_BUG` | "BUG!" | Various | Internal FFmpeg bug detected |
| `AVERROR_BUG2` | "BUG2" | Various | Another internal bug |
| `AVERROR_BUFFER_TOO_SMALL` | "BUFS" | Various | Supplied buffer too small |
| `AVERROR_EXIT` | "EXIT" | Various | Immediate exit requested |
| `AVERROR_PATCHWELCOME` | "PATC" | Various | Feature not implemented |
| `AVERROR_HTTP_NOT_FOUND` | "HTNF" | Various | HTTP resource not found |
| `AVERROR_HTTP_UNAUTHORIZED` | "HT AU" | Various | HTTP unauthorized |
| `AVERROR_HTTP_FORBIDDEN` | "HT FD" | Various | HTTP forbidden |

### 2.3 FFERRTAG Implementation

```c
// FFERRTAG creates unique error tags from 4 characters
#define MKTAG(a, b, c, d) ((a) | ((b) << 8) | ((c) << 16) | ((unsigned)(d) << 24))
#define FFERRTAG(a, b, c, d) (-(int)MKTAG(a, b, c, d))

// Example:
// AVERROR_EOF = FFERRTAG('E','O','F',' ')
//           = -(int)MKTAG('E','O','F',' ')
//           = -(int)(0x45 | (0x4F << 8) | (0x46 << 16) | (0x20 << 24))
```

### 2.4 Error Code to String Conversion

```c
#include <libavutil/error.h>

char error_buffer[AV_ERROR_MAX_STRING_SIZE];

// Convert error code to human-readable string
int av_strerror(int errnum, char *errbuf, size_t errbuf_size);

// Example usage
int error_code = AVERROR_INVALIDDATA;
av_strerror(error_code, error_buffer, sizeof(error_buffer));
printf("Error: %s\n", error_buffer);  // "Invalid data found when processing input"

// Convenience macro
#define av_err2str(errnum) av_make_error_string((char[AV_ERROR_MAX_STRING_SIZE]){0}, \
                                                AV_ERROR_MAX_STRING_SIZE, errnum)

// Usage with fprintf
fprintf(stderr, "Error: %s\n", av_err2str(error_code));
```

---

## 3. COMMON ERROR HANDLING PATTERNS

### 3.1 Basic Error Handling Pattern

```c
#include <libavformat/avformat.h>
#include <libavcodec/avcodec.h>
#include <libavutil/error.h>

int open_input_file(const char *filename, AVFormatContext **fmt_ctx_out) {
    AVFormatContext *fmt_ctx = NULL;
    int ret;
    
    // Open input file
    ret = avformat_open_input(&fmt_ctx, filename, NULL, NULL);
    if (ret < 0) {
        char errbuf[AV_ERROR_MAX_STRING_SIZE];
        av_strerror(ret, errbuf, sizeof(errbuf));
        fprintf(stderr, "Cannot open input file '%s': %s\n", filename, errbuf);
        return ret;
    }
    
    // Get stream info
    ret = avformat_find_stream_info(fmt_ctx, NULL);
    if (ret < 0) {
        char errbuf[AV_ERROR_MAX_STRING_SIZE];
        av_strerror(ret, errbuf, sizeof(errbuf));
        fprintf(stderr, "Cannot find stream info: %s\n", errbuf);
        avformat_close_input(&fmt_ctx);
        return ret;
    }
    
    *fmt_ctx_out = fmt_ctx;
    return 0;
}
```

### 3.2 Decoder Error Handling Pattern

```c
#include <libavcodec/avcodec.h>

int decode_audio(AVCodecContext *dec_ctx, AVPacket *pkt, AVFrame *frame,
                 FILE *outfile) {
    int ret;
    
    // Send packet to decoder
    ret = avcodec_send_packet(dec_ctx, pkt);
    if (ret < 0) {
        if (ret == AVERROR_EOF) {
            // Normal decoder flush
            return 0;
        } else if (ret == AVERROR_INVALIDDATA) {
            // Corrupt packet, skip it
            fprintf(stderr, "Warning: Skipping invalid packet\n");
            return 0;
        } else {
            char errbuf[AV_ERROR_MAX_STRING_SIZE];
            av_strerror(ret, errbuf, sizeof(errbuf));
            fprintf(stderr, "Error sending packet to decoder: %s\n", errbuf);
            return ret;
        }
    }
    
    // Receive decoded frames
    while (ret >= 0) {
        ret = avcodec_receive_frame(dec_ctx, frame);
        if (ret == AVERROR_EOF) {
            // Decoder flushed completely
            break;
        } else if (ret == AVERROR(EAGAIN)) {
            // Need more input
            return 0;
        } else if (ret < 0) {
            char errbuf[AV_ERROR_MAX_STRING_SIZE];
            av_strerror(ret, errbuf, sizeof(errbuf));
            fprintf(stderr, "Error receiving decoded frame: %s\n", errbuf);
            return ret;
        }
        
        // Process the decoded frame
        process_audio_frame(frame, outfile);
        av_frame_unref(frame);
    }
    
    return 0;
}
```

### 3.3 Encoder Error Handling Pattern

```c
#include <libavcodec/avcodec.h>

int encode_audio(AVCodecContext *enc_ctx, AVFrame *frame, AVPacket *pkt,
                  FILE *outfile) {
    int ret;
    
    // Send frame to encoder
    ret = avcodec_send_frame(enc_ctx, frame);
    if (ret < 0) {
        if (ret == AVERROR_EOF) {
            return 0;
        } else {
            char errbuf[AV_ERROR_MAX_STRING_SIZE];
            av_strerror(ret, errbuf, sizeof(errbuf));
            fprintf(stderr, "Error sending frame to encoder: %s\n", errbuf);
            return ret;
        }
    }
    
    // Receive encoded packets
    while (ret >= 0) {
        ret = avcodec_receive_packet(enc_ctx, pkt);
        if (ret == AVERROR_EOF) {
            break;
        } else if (ret == AVERROR(EAGAIN)) {
            // Need more input
            return 0;
        } else if (ret < 0) {
            char errbuf[AV_ERROR_MAX_STRING_SIZE];
            av_strerror(ret, errbuf, sizeof(errbuf));
            fprintf(stderr, "Error receiving encoded packet: %s\n", errbuf);
            return ret;
        }
        
        // Write encoded packet
        if (pkt->size > 0) {
            fwrite(pkt->data, 1, pkt->size, outfile);
        }
        av_packet_unref(pkt);
    }
    
    return 0;
}
```

### 3.4 Format Context Error Handling

```c
#include <libavformat/avformat.h>

int open_output_file(const char *filename, AVFormatContext *in_fmt_ctx,
                    const AVCodec *enc, AVFormatContext **out_fmt_ctx_out) {
    AVFormatContext *out_fmt_ctx = NULL;
    int ret;
    
    // Create output context
    ret = avformat_alloc_output_context2(&out_fmt_ctx, NULL, NULL, filename);
    if (ret < 0 || !out_fmt_ctx) {
        fprintf(stderr, "Could not create output context\n");
        if (out_fmt_ctx) avformat_free_context(out_fmt_ctx);
        return ret < 0 ? ret : AVERROR(ENOMEM);
    }
    
    // Add stream
    AVStream *out_stream = avformat_new_stream(out_fmt_ctx, NULL);
    if (!out_stream) {
        fprintf(stderr, "Could not create output stream\n");
        avformat_free_context(out_fmt_ctx);
        return AVERROR(ENOMEM);
    }
    
    // Copy codec parameters
    ret = avcodec_parameters_from_context(out_stream->codecpar, 
                                           in_fmt_ctx->streams[0]->codecpar);
    if (ret < 0) {
        char errbuf[AV_ERROR_MAX_STRING_SIZE];
        av_strerror(ret, errbuf, sizeof(errbuf));
        fprintf(stderr, "Could not copy codec parameters: %s\n", errbuf);
        avformat_free_context(out_fmt_ctx);
        return ret;
    }
    
    // Open output file
    if (!(out_fmt_ctx->oformat->flags & AVFMT_NOFILE)) {
        ret = avio_open(&out_fmt_ctx->pb, filename, AVIO_FLAG_WRITE);
        if (ret < 0) {
            char errbuf[AV_ERROR_MAX_STRING_SIZE];
            av_strerror(ret, errbuf, sizeof(errbuf));
            fprintf(stderr, "Could not open output file '%s': %s\n", filename, errbuf);
            avformat_free_context(out_fmt_ctx);
            return ret;
        }
    }
    
    // Write header
    ret = avformat_write_header(out_fmt_ctx, NULL);
    if (ret < 0) {
        char errbuf[AV_ERROR_MAX_STRING_SIZE];
        av_strerror(ret, errbuf, sizeof(errbuf));
        fprintf(stderr, "Error writing header: %s\n", errbuf);
        if (!(out_fmt_ctx->oformat->flags & AVFMT_NOFILE))
            avio_closep(&out_fmt_ctx->pb);
        avformat_free_context(out_fmt_ctx);
        return ret;
    }
    
    *out_fmt_ctx_out = out_fmt_ctx;
    return 0;
}
```

---

## 4. INVALID DATA HANDLING

### 4.1 Detecting Invalid Data

```c
#include <libavcodec/avcodec.h>

// Check frame for decode errors
void process_decoded_frame(AVFrame *frame, AVCodecContext *dec_ctx) {
    // Check decode error flags
    if (frame->flags & AV_FRAME_FLAG_CORRUPT) {
        fprintf(stderr, "Warning: Frame is marked as corrupted\n");
    }
    
    // Check decode error concealment
    if (frame->decode_error_flags & AV_DECODE_ERROR_ICONCEAL) {
        fprintf(stderr, "Warning: Errors were concealed in this frame\n");
    }
    
    // Additional checks
    if (frame->nb_samples == 0) {
        fprintf(stderr, "Warning: Frame has zero samples\n");
    }
    
    if (frame->format < 0) {
        fprintf(stderr, "Warning: Frame has invalid format\n");
    }
}
```

### 4.2 Decoder Error Concealment

```c
#include <libavcodec/avcodec.h>

// Enable error concealment for decoders
void configure_decoder_error_handling(AVCodecContext *dec_ctx) {
    // Most decoders handle errors internally
    // Options to consider:
    
    // Set thread count for parallel processing
    dec_ctx->thread_count = 4;
    
    // Set threading type
    dec_ctx->thread_type = FF_THREAD_FRAME | FF_THREAD_SLICE;
    
    // For buggy streams, try different approaches:
    
    // Option 1: Skip corrupted frames (MP3, AAC)
    // Most decoders do this automatically
    
    // Option 2: Use error concealment
    // Enabled by default for most formats
    
    // Option 3: Set flags
    dec_ctx->flags |= AV_CODEC_FLAG_OUTPUT_CORRUPT;
}
```

### 4.3 Handling Corrupt Files

```c
#include <libavformat/avformat.h>
#include <libavcodec/avcodec.h>

typedef struct {
    int frames_processed;
    int frames_corrupted;
    int frames_skipped;
} DecodeStats;

int decode_with_error_tracking(const char *input_file, const char *output_file,
                               DecodeStats *stats) {
    AVFormatContext *fmt_ctx = NULL;
    AVCodecContext *dec_ctx = NULL;
    const AVCodec *dec = NULL;
    int ret;
    int stream_index;
    
    // Initialize stats
    memset(stats, 0, sizeof(DecodeStats));
    
    // Open input
    ret = avformat_open_input(&fmt_ctx, input_file, NULL, NULL);
    if (ret < 0) {
        fprintf(stderr, "Cannot open input: %s\n", av_err2str(ret));
        return ret;
    }
    
    ret = avformat_find_stream_info(fmt_ctx, NULL);
    if (ret < 0) {
        fprintf(stderr, "Cannot find stream info: %s\n", av_err2str(ret));
        goto end;
    }
    
    // Find audio stream
    stream_index = av_find_best_stream(fmt_ctx, AVMEDIA_TYPE_AUDIO, -1, -1, &dec, 0);
    if (stream_index < 0) {
        fprintf(stderr, "Cannot find audio stream\n");
        ret = AVERROR_STREAM_NOT_FOUND;
        goto end;
    }
    
    // Setup decoder
    dec_ctx = avcodec_alloc_context3(dec);
    if (!dec_ctx) {
        ret = AVERROR(ENOMEM);
        goto end;
    }
    
    ret = avcodec_parameters_to_context(dec_ctx, fmt_ctx->streams[stream_index]->codecpar);
    if (ret < 0) goto end;
    
    // Open decoder
    ret = avcodec_open2(dec_ctx, dec, NULL);
    if (ret < 0) {
        fprintf(stderr, "Cannot open decoder: %s\n", av_err2str(ret));
        goto end;
    }
    
    // Main decode loop
    AVPacket *pkt = av_packet_alloc();
    AVFrame *frame = av_frame_alloc();
    
    while (av_read_frame(fmt_ctx, pkt) >= 0) {
        if (pkt->stream_index == stream_index) {
            stats->frames_processed++;
            
            ret = avcodec_send_packet(dec_ctx, pkt);
            av_packet_unref(pkt);
            
            if (ret < 0) {
                if (ret == AVERROR_INVALIDDATA) {
                    stats->frames_skipped++;
                    fprintf(stderr, "Skipping invalid packet\n");
                    continue;
                } else if (ret == AVERROR_EOF) {
                    break;
                }
            }
            
            while (ret >= 0) {
                ret = avcodec_receive_frame(dec_ctx, frame);
                if (ret == AVERROR(EAGAIN)) {
                    ret = 0;
                    break;
                } else if (ret == AVERROR_EOF) {
                    ret = 0;
                    break;
                } else if (ret < 0) {
                    fprintf(stderr, "Decode error: %s\n", av_err2str(ret));
                    goto decode_loop_end;
                }
                
                // Check for corrupted frames
                if (frame->decode_error_flags & AV_DECODE_ERROR_ICONCEAL) {
                    stats->frames_corrupted++;
                }
                
                // Process frame
                process_audio_frame(frame);
                av_frame_unref(frame);
            }
        } else {
            av_packet_unref(pkt);
        }
    }
    
decode_loop_end:
    // Flush decoder
    avcodec_send_packet(dec_ctx, NULL);
    while (avcodec_receive_frame(dec_ctx, frame) == 0) {
        process_audio_frame(frame);
        av_frame_unref(frame);
    }
    
    av_frame_free(&frame);
    av_packet_free(&pkt);
    
end:
    avcodec_free_context(&dec_ctx);
    avformat_close_input(&fmt_ctx);
    
    // Print stats
    fprintf(stderr, "Decode complete: %d processed, %d corrupted, %d skipped\n",
            stats->frames_processed, stats->frames_corrupted, stats->frames_skipped);
    
    return ret < 0 ? ret : 0;
}
```

---

## 5. MUXER AND DEMUXER ERRORS

### 5.1 Muxer Error Handling

```c
#include <libavformat/avformat.h>

int mux_with_error_handling(AVFormatContext *in_fmt_ctx, 
                            AVFormatContext *out_fmt_ctx,
                            AVCodecContext *enc_ctx) {
    int ret;
    AVPacket *pkt = av_packet_alloc();
    AVFrame *frame = av_frame_alloc();
    int frame_count = 0;
    
    while (av_read_frame(in_fmt_ctx, pkt) >= 0) {
        if (pkt->stream_index != 0) {
            av_packet_unref(pkt);
            continue;
        }
        
        // Decode
        ret = avcodec_send_packet(enc_ctx, pkt);
        av_packet_unref(pkt);
        
        while (ret >= 0) {
            ret = avcodec_receive_frame(enc_ctx, frame);
            if (ret == AVERROR(EAGAIN)) {
                ret = 0;
                break;
            } else if (ret == AVERROR_EOF) {
                break;
            } else if (ret < 0) {
                fprintf(stderr, "Decode error: %s\n", av_err2str(ret));
                goto end;
            }
            
            // Encode
            ret = avcodec_send_frame(enc_ctx, frame);
            av_frame_unref(frame);
            
            while (ret >= 0) {
                ret = avcodec_receive_packet(enc_ctx, pkt);
                if (ret == AVERROR(EAGAIN)) {
                    ret = 0;
                    break;
                } else if (ret == AVERROR_EOF) {
                    break;
                } else if (ret < 0) {
                    fprintf(stderr, "Encode error: %s\n", av_err2str(ret));
                    goto end;
                }
                
                // Write packet
                pkt->stream_index = 0;
                av_packet_rescale_ts(pkt, enc_ctx->time_base, 
                                    out_fmt_ctx->streams[0]->time_base);
                
                ret = av_interleaved_write_frame(out_fmt_ctx, pkt);
                if (ret < 0) {
                    fprintf(stderr, "Muxer error: %s\n", av_err2str(ret));
                    // Check for specific errors
                    if (ret == AVERROR(EIO)) {
                        fprintf(stderr, "I/O error during muxing\n");
                    } else if (ret == AVERROR_INVALIDDATA) {
                        fprintf(stderr, "Invalid packet data\n");
                    }
                    goto end;
                }
                
                frame_count++;
            }
        }
    }
    
    // Write trailer
    ret = av_write_trailer(out_fmt_ctx);
    if (ret < 0) {
        fprintf(stderr, "Error writing trailer: %s\n", av_err2str(ret));
    }
    
end:
    fprintf(stderr, "Encoded %d frames\n", frame_count);
    av_frame_free(&frame);
    av_packet_free(&pkt);
    
    return ret < 0 ? ret : 0;
}
```

### 5.2 Demuxer Error Handling

```c
#include <libavformat/avformat.h>

int demux_with_recovery(const char *filename) {
    AVFormatContext *fmt_ctx = NULL;
    int ret;
    int error_count = 0;
    const int max_errors = 100;
    
    // Try to open file
    ret = avformat_open_input(&fmt_ctx, filename, NULL, NULL);
    if (ret < 0) {
        fprintf(stderr, "Cannot open '%s': %s\n", filename, av_err2str(ret));
        
        // Check specific error types
        if (ret == AVERROR_ENOENT) {
            fprintf(stderr, "File not found\n");
        } else if (ret == AVERROR(EACCES)) {
            fprintf(stderr, "Permission denied\n");
        } else if (ret == AVERROR_INVALIDDATA) {
            fprintf(stderr, "Invalid data or unsupported format\n");
        }
        return ret;
    }
    
    // Try to find stream info with error tolerance
    AVDictionary *opts = NULL;
    av_dict_set(&opts, "timeout", "5000000", 0);  // 5 second timeout
    av_dict_set(&opts, "reconnect", "1", 0);
    av_dict_set(&opts, "reconnect_streamed", "1", 0);
    
    ret = avformat_find_stream_info(fmt_ctx, opts);
    if (ret < 0) {
        fprintf(stderr, "Cannot find stream info: %s\n", av_err2str(ret));
        
        // For some formats, this is acceptable
        if (fmt_ctx->nb_streams > 0) {
            fprintf(stderr, "Continuing anyway, found %d streams\n", fmt_ctx->nb_streams);
        } else {
            avformat_close_input(&fmt_ctx);
            av_dict_free(&opts);
            return ret;
        }
    }
    av_dict_free(&opts);
    
    // Read packets with error handling
    AVPacket *pkt = av_packet_alloc();
    
    while (av_read_frame(fmt_ctx, pkt) >= 0) {
        // Process packet
        // ...
        
        av_packet_unref(pkt);
        
        // Check error count
        if (error_count > max_errors) {
            fprintf(stderr, "Too many errors (%d), stopping\n", error_count);
            ret = AVERROR_INVALIDDATA;
            break;
        }
    }
    
    if (ret < 0 && ret != AVERROR_EOF) {
        fprintf(stderr, "Error reading: %s\n", av_err2str(ret));
    }
    
    av_packet_free(&pkt);
    avformat_close_input(&fmt_ctx);
    
    return 0;
}
```

---

## 6. FORMAT PROBING AND DETECTION

### 6.1 Format Probing with avformat_find_stream_info

```c
#include <libavformat/avformat.h>

int probe_with_limits(const char *filename, int max_analyze_duration) {
    AVFormatContext *fmt_ctx = NULL;
    int ret;
    
    // Allocate context
    fmt_ctx = avformat_alloc_context();
    if (!fmt_ctx) {
        return AVERROR(ENOMEM);
    }
    
    // Set probing limits
    // Default is 5 seconds (AVFMT_NOHEADER) or 8 seconds
    // Set in microseconds
    fmt_ctx->max_analyze_duration = max_analyze_duration * 1000000;
    fmt_ctx->max_analyze_duration_fixed = max_analyze_duration * 1000000;
    
    // Set probesize for format detection
    fmt_ctx->probesize = 5 * 1024 * 1024;  // 5 MB probe size
    fmt_ctx->format_probesize = 5 * 1024 * 1024;
    
    // Open input
    ret = avformat_open_input(&fmt_ctx, filename, NULL, NULL);
    if (ret < 0) {
        avformat_free_context(&fmt_ctx);
        return ret;
    }
    
    // Find stream info
    ret = avformat_find_stream_info(fmt_ctx, NULL);
    if (ret < 0) {
        fprintf(stderr, "Warning: Could not find complete stream info: %s\n", 
                av_err2str(ret));
        // This might be acceptable for some formats
        if (ret == AVERROR_INVALIDDATA) {
            fprintf(stderr, "Invalid data encountered during probing\n");
        }
    }
    
    // Print format info
    printf("Format: %s (%s)\n", fmt_ctx->iformat->name, 
           fmt_ctx->iformat->long_name);
    printf("Duration: %lld ms\n", fmt_ctx->duration / 1000);
    printf("Streams: %d\n", fmt_ctx->nb_streams);
    
    avformat_close_input(&fmt_ctx);
    avformat_free_context(&fmt_ctx);
    
    return 0;
}
```

### 6.2 Format Guessing

```c
#include <libavformat/avformat.h>

const AVInputFormat* guess_format_with_hints(const char *filename,
                                             const char *mime_type,
                                             const uint8_t *buf,
                                             int buf_size) {
    const AVInputFormat *fmt = NULL;
    
    // Try guessing from filename
    fmt = av_guess_format(filename, NULL, mime_type);
    if (fmt) {
        printf("Guessed format from filename: %s\n", fmt->name);
        return fmt;
    }
    
    // Try guessing from MIME type
    if (mime_type) {
        fmt = av_find_input_format(mime_type);
        if (fmt) {
            printf("Found format for MIME type: %s\n", fmt->name);
            return fmt;
        }
    }
    
    // Try probing from buffer
    if (buf && buf_size > 0) {
        AVProbeData pd = {
            .filename = filename ? filename : "",
            .buf = buf,
            .buf_size = buf_size,
            .mime_type = mime_type
        };
        
        fmt = av_probe_input_format(&pd, 1);  // 1 = fully greedy
        if (fmt) {
            printf("Probed format: %s (confidence high)\n", fmt->name);
            return fmt;
        }
        
        // Try with lower confidence
        fmt = av_probe_input_format2(&pd, 1, NULL);
        if (fmt) {
            printf("Probed format: %s\n", fmt->name);
            return fmt;
        }
    }
    
    return NULL;  // Could not determine format
}
```

### 6.3 Manual Format Detection

```c
#include <libavformat/avformat.h>

int detect_format_manually(const char *filename) {
    FILE *fp = fopen(filename, "rb");
    if (!fp) {
        perror("fopen");
        return AVERROR(errno);
    }
    
    // Read magic bytes
    uint8_t buf[32];
    size_t n = fread(buf, 1, sizeof(buf), fp);
    fclose(fp);
    
    if (n < 12) {
        return AVERROR_INVALIDDATA;
    }
    
    // Check magic bytes
    if (memcmp(buf, "RIFF", 4) == 0 && memcmp(buf + 8, "WAVE", 4) == 0) {
        printf("Detected: WAV (RIFF/WAVE)\n");
        return 0;
    }
    
    if (memcmp(buf, "FORM", 4) == 0 && memcmp(buf + 8, "AIFF", 4) == 0) {
        printf("Detected: AIFF\n");
        return 0;
    }
    
    if (memcmp(buf, "ID3", 3) == 0) {
        printf("Detected: MP3 with ID3v2 tag\n");
        return 0;
    }
    
    if (buf[0] == 0xFF && (buf[1] & 0xE0) == 0xE0) {
        printf("Detected: MP3 frame sync\n");
        return 0;
    }
    
    if (memcmp(buf, "fLaC", 4) == 0) {
        printf("Detected: FLAC\n");
        return 0;
    }
    
    if (memcmp(buf, "OggS", 4) == 0) {
        printf("Detected: OGG container\n");
        return 0;
    }
    
    // Check for common audio formats
    const char *ext = strrchr(filename, '.');
    if (ext) {
        if (strcasecmp(ext, ".mp3") == 0) {
            printf("Assuming: MP3 (from extension)\n");
            return 0;
        }
        if (strcasecmp(ext, ".flac") == 0) {
            printf("Assuming: FLAC (from extension)\n");
            return 0;
        }
        if (strcasecmp(ext, ".m4a") == 0 || strcasecmp(ext, ".mp4") == 0) {
            printf("Assuming: MP4/M4A (from extension)\n");
            return 0;
        }
    }
    
    printf("Unknown format\n");
    return AVERROR_INVALIDDATA;
}
```

---

## 7. RECOVERY STRATEGIES

### 7.1 Decoder Reinitialization

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>

typedef struct {
    int reinit_count;
    int errors_recovered;
} RecoveryStats;

int decode_with_recovery(const char *input, const char *output,
                        RecoveryStats *stats) {
    AVFormatContext *fmt_ctx = NULL;
    AVCodecContext *dec_ctx = NULL;
    const AVCodec *dec = NULL;
    int stream_idx;
    int ret;
    
    memset(stats, 0, sizeof(RecoveryStats));
    
    ret = avformat_open_input(&fmt_ctx, input, NULL, NULL);
    if (ret < 0) return ret;
    
    ret = avformat_find_stream_info(fmt_ctx, NULL);
    if (ret < 0) goto end;
    
    stream_idx = av_find_best_stream(fmt_ctx, AVMEDIA_TYPE_AUDIO, -1, -1, &dec, 0);
    if (stream_idx < 0) {
        ret = AVERROR_STREAM_NOT_FOUND;
        goto end;
    }
    
    dec_ctx = avcodec_alloc_context3(dec);
    if (!dec_ctx) {
        ret = AVERROR(ENOMEM);
        goto end;
    }
    
    ret = avcodec_parameters_to_context(dec_ctx, 
                                       fmt_ctx->streams[stream_idx]->codecpar);
    if (ret < 0) goto end;
    
    // Open decoder
    ret = avcodec_open2(dec_ctx, dec, NULL);
    if (ret < 0) goto end;
    
    AVPacket *pkt = av_packet_alloc();
    AVFrame *frame = av_frame_alloc();
    
    while (av_read_frame(fmt_ctx, pkt) >= 0) {
        if (pkt->stream_index == stream_idx) {
        send_packet:
            ret = avcodec_send_packet(dec_ctx, pkt);
            av_packet_unref(pkt);
            
            if (ret < 0) {
                if (ret == AVERROR_INVALIDDATA) {
                    // Skip corrupt packet
                    continue;
                } else if (ret == AVERROR_EOF) {
                    break;
                } else {
                    // Try decoder reinitialization
                    if (stats->reinit_count < 3) {
                        fprintf(stderr, "Decoder error, attempting recovery...\n");
                        
                        // Flush decoder
                        avcodec_flush_buffers(dec_ctx);
                        
                        // Try reopening decoder
                        avcodec_free_context(&dec_ctx);
                        dec_ctx = avcodec_alloc_context3(dec);
                        if (!dec_ctx) continue;
                        
                        avcodec_parameters_to_context(dec_ctx,
                            fmt_ctx->streams[stream_idx]->codecpar);
                        
                        ret = avcodec_open2(dec_ctx, dec, NULL);
                        if (ret < 0) continue;
                        
                        stats->reinit_count++;
                        stats->errors_recovered++;
                        
                        // Retry with same packet (need to seek back or re-read)
                        // For simplicity, just continue
                        continue;
                    }
                }
            }
            
            while (ret >= 0) {
                ret = avcodec_receive_frame(dec_ctx, frame);
                if (ret == AVERROR(EAGAIN)) {
                    ret = 0;
                    break;
                } else if (ret == AVERROR_EOF) {
                    break;
                } else if (ret < 0) {
                    if (ret == AVERROR_INVALIDDATA) {
                        // Skip bad frame
                        continue;
                    }
                    break;
                }
                
                process_audio_frame(frame);
                av_frame_unref(frame);
            }
        }
    }
    
    av_frame_free(&frame);
    av_packet_free(&pkt);
    
end:
    fprintf(stderr, "Recovery stats: %d reinit, %d recovered\n",
            stats->reinit_count, stats->errors_recovered);
    
    avcodec_free_context(&dec_ctx);
    avformat_close_input(&fmt_ctx);
    
    return 0;
}
```

### 7.2 avcodec_flush_buffers

```c
#include <libavcodec/avcodec.h>

void flush_decoder_state(AVCodecContext *dec_ctx) {
    // Flush decoder buffers
    // This should be called:
    // 1. After seeking
    // 2. Before decoder reinit
    // 3. When resuming after error
    
    avcodec_flush_buffers(dec_ctx);
}

// Example: Flush after seek
int seek_and_resume(AVFormatContext *fmt_ctx, AVCodecContext *dec_ctx,
                    int64_t timestamp) {
    int ret;
    
    // Seek to position
    ret = av_seek_frame(fmt_ctx, -1, timestamp, AVSEEK_FLAG_BACKWARD);
    if (ret < 0) {
        fprintf(stderr, "Seek failed: %s\n", av_err2str(ret));
        return ret;
    }
    
    // Flush decoder to remove stale frames
    avcodec_flush_buffers(dec_ctx);
    
    return 0;
}
```

### 7.3 Stream Recovery

```c
#include <libavformat/avformat.h>

int resync_stream(AVFormatContext *fmt_ctx, int stream_index) {
    int ret;
    
    // Try to resync by seeking slightly forward
    int64_t current_pos = avio_tell(fmt_ctx->pb);
    int64_t target_pos = current_pos + 1024;  // Skip 1KB
    
    ret = avio_seek(fmt_ctx->pb, target_pos, SEEK_SET);
    if (ret < 0) {
        fprintf(stderr, "Cannot seek forward: %s\n", av_err2str(ret));
        return ret;
    }
    
    // Clear any buffered data
    avformat_flush(fmt_ctx);
    
    // Find sync point manually if needed
    // This is format-dependent
    
    return 0;
}
```

---

## 8. INTERRUPT AND TIMEOUT HANDLING

### 8.1 AVIOInterruptCB Setup

```c
#include <libavformat/avformat.h>
#include <stdbool.h>
#include <stdatomic.h>

// Global interrupt flag
static volatile sig_atomic_t g_interrupted = 0;

// Interrupt callback function
static int interrupt_callback(void *opaque) {
    // Check opaque for custom state
    if (opaque) {
        // Could be a pointer to custom state
        // For example:
        // MyContext *ctx = (MyContext *)opaque;
        // return ctx->should_interrupt;
    }
    
    // Return 1 to abort, 0 to continue
    return g_interrupted;
}

// Set up interrupt callback
void setup_interrupt_handling(AVFormatContext *fmt_ctx) {
    fmt_ctx->interrupt_callback.callback = interrupt_callback;
    fmt_ctx->interrupt_callback.opaque = NULL;  // Or custom context
}

// Call this to request interruption
void request_interrupt(void) {
    g_interrupted = 1;
}

// Reset interrupt flag
void reset_interrupt(void) {
    g_interrupted = 0;
}
```

### 8.2 Timeout Handling with Custom Context

```c
#include <libavformat/avformat.h>
#include <time.h>

typedef struct {
    int timeout_seconds;
    time_t start_time;
    int should_abort;
} TimeoutContext;

static int timeout_callback(void *opaque) {
    TimeoutContext *ctx = (TimeoutContext *)opaque;
    
    if (ctx->should_abort) {
        return 1;  // Explicit abort requested
    }
    
    if (ctx->timeout_seconds > 0) {
        time_t elapsed = time(NULL) - ctx->start_time;
        if (elapsed >= ctx->timeout_seconds) {
            fprintf(stderr, "Timeout after %d seconds\n", ctx->timeout_seconds);
            return 1;  // Timeout
        }
    }
    
    return 0;  // Continue
}

int open_with_timeout(const char *url, int timeout_sec,
                      AVFormatContext **fmt_ctx_out) {
    AVFormatContext *fmt_ctx = NULL;
    TimeoutContext timeout_ctx = {
        .timeout_seconds = timeout_sec,
        .start_time = time(NULL),
        .should_abort = 0
    };
    int ret;
    
    fmt_ctx = avformat_alloc_context();
    if (!fmt_ctx) {
        return AVERROR(ENOMEM);
    }
    
    fmt_ctx->interrupt_callback.callback = timeout_callback;
    fmt_ctx->interrupt_callback.opaque = &timeout_ctx;
    
    ret = avformat_open_input(&fmt_ctx, url, NULL, NULL);
    if (ret < 0) {
        if (timeout_ctx.should_abort) {
            fprintf(stderr, "Open interrupted\n");
        } else {
            fprintf(stderr, "Cannot open: %s\n", av_err2str(ret));
        }
        avformat_free_context(&fmt_ctx);
        return ret;
    }
    
    *fmt_ctx_out = fmt_ctx;
    return 0;
}
```

### 8.3 Handling AVERROR_EXIT

```c
#include <libavutil/error.h>

int handle_operation_with_interrupt(const char *filename) {
    AVFormatContext *fmt_ctx = NULL;
    int ret;
    
    // Setup interrupt callback
    // (see previous section)
    
    ret = avformat_open_input(&fmt_ctx, filename, NULL, NULL);
    if (ret < 0) {
        if (ret == AVERROR_EXIT) {
            fprintf(stderr, "Operation was interrupted\n");
            // Clean up and return gracefully
        } else {
            fprintf(stderr, "Error: %s\n", av_err2str(ret));
        }
        if (fmt_ctx) {
            avformat_close_input(&fmt_ctx);
        }
        return ret;
    }
    
    // Normal processing...
    
    avformat_close_input(&fmt_ctx);
    return 0;
}
```

### 8.4 Network Stream Timeout

```bash
# FFmpeg CLI with timeout
timeout 60 ffmpeg -i rtsp://example.com/stream -c copy output.mp4

# With reconnect
ffmpeg -i rtsp://example.com/stream \
  -timeout 10000000 \
  -reconnect 1 \
  -reconnect_streamed 1 \
  -reconnect_delay_max 5 \
  -c copy output.mp4

# Protocol-specific options
ffmpeg -i http://example.com/stream.m3u8 \
  -rw_timeout 10000000 \
  -c copy output.mp4
```

---

## 9. av_read_frame ERROR HANDLING

### 9.1 Complete av_read_frame Loop with Error Handling

```c
#include <libavformat/avformat.h>

int read_packets_with_handling(AVFormatContext *fmt_ctx, int audio_stream_idx) {
    AVPacket *pkt = av_packet_alloc();
    int ret;
    int error_count = 0;
    int eof_count = 0;
    const int max_errors = 100;
    
    while (1) {
        ret = av_read_frame(fmt_ctx, pkt);
        
        if (ret < 0) {
            if (ret == AVERROR_EOF) {
                fprintf(stderr, "End of file\n");
                break;
            } else if (ret == AVERROR_EXIT) {
                fprintf(stderr, "Read interrupted\n");
                break;
            } else if (ret == AVERROR_INVALIDDATA) {
                fprintf(stderr, "Invalid data, skipping...\n");
                error_count++;
                if (error_count > max_errors) {
                    fprintf(stderr, "Too many errors, giving up\n");
                    ret = AVERROR_INVALIDDATA;
                    break;
                }
                continue;  // Skip bad packet and continue
            } else if (ret == AVERROR(EAGAIN)) {
                // Should not happen with av_read_frame, but handle anyway
                fprintf(stderr, "EAGAIN, retrying...\n");
                continue;
            } else if (ret == AVERROR_EOF) {
                break;
            } else {
                // Unknown error
                fprintf(stderr, "Unknown read error: %s\n", av_err2str(ret));
                error_count++;
                if (error_count > max_errors) {
                    break;
                }
                continue;
            }
        }
        
        // Reset error count on successful read
        error_count = 0;
        
        // Process packet
        if (pkt->stream_index == audio_stream_idx) {
            process_audio_packet(pkt);
        }
        
        av_packet_unref(pkt);
    }
    
    av_packet_free(&pkt);
    
    if (ret == AVERROR_EOF || ret == AVERROR_EXIT) {
        return 0;  // Normal termination
    }
    
    return ret;
}
```

### 9.2 Non-blocking Read with Polling

```c
#include <libavformat/avformat.h>

int nonblocking_read_loop(AVFormatContext *fmt_ctx) {
    AVPacket *pkt = av_packet_alloc();
    int ret;
    
    // Make read non-blocking
    // Note: This doesn't fully work with all protocols
    // For full async support, use separate thread
    
    while (1) {
        ret = av_read_frame(fmt_ctx, pkt);
        
        if (ret == AVERROR(EAGAIN)) {
            // No packet available, wait and retry
            // In real code, use proper waiting mechanism
            struct timespec ts = {0, 10000000};  // 10ms
            nanosleep(&ts, NULL);
            continue;
        } else if (ret < 0) {
            if (ret == AVERROR_EOF) {
                fprintf(stderr, "EOF reached\n");
            } else {
                fprintf(stderr, "Read error: %s\n", av_err2str(ret));
            }
            break;
        }
        
        // Process packet
        process_packet(pkt);
        av_packet_unref(pkt);
    }
    
    av_packet_free(&pkt);
    return ret;
}
```

---

## 10. MEMORY MANAGEMENT AND LEAKS

### 10.1 Proper Reference Management

```c
#include <libavutil/frame.h>
#include <libavcodec/avcodec.h>

void demonstrate_reference_management(void) {
    AVFrame *frame = av_frame_alloc();
    
    // Frame holds references to buffers
    // Always unref when done
    
    av_frame_unref(frame);  // Release buffer references
    av_frame_free(&frame);  // Free frame structure
    
    // AVFrame used as local variable in decode loop:
    AVFrame *local_frame = av_frame_alloc();
    while (decoding) {
        ret = avcodec_receive_frame(dec_ctx, local_frame);
        if (ret >= 0) {
            // Process frame...
            // Important: unref before next iteration
            av_frame_unref(local_frame);
        }
    }
    av_frame_free(&local_frame);
    
    // Similar for packets
    AVPacket *pkt = av_packet_alloc();
    av_packet_unref(pkt);  // Clean any internal references
    av_packet_free(&pkt);  // Free packet
}

void demonstrate_packet_unref(AVPacket *pkt) {
    // Always unref packet data after use
    av_packet_unref(pkt);
    
    // When reusing a packet, unref old data first
    av_packet_unref(pkt);
    int ret = av_read_frame(fmt_ctx, pkt);
    if (ret >= 0) {
        // Process...
        av_packet_unref(pkt);  // Don't forget
    }
}
```

### 10.2 Complete Cleanup Sequence

```c
#include <libavcodec/avcodec.h>
#include <libavformat/avformat.h>

void cleanup_all_resources(AVFormatContext *ifmt_ctx,
                           AVFormatContext *ofmt_ctx,
                           AVCodecContext *dec_ctx,
                           AVCodecContext *enc_ctx,
                           SwrContext *swr_ctx,
                           AVPacket *pkt,
                           AVFrame *frame) {
    // Order matters: always clean up in reverse order of allocation
    
    // 1. Free frames
    if (frame) {
        av_frame_free(&frame);
    }
    
    // 2. Free packets
    if (pkt) {
        av_packet_free(&pkt);
    }
    
    // 3. Free swr context
    if (swr_ctx) {
        swr_free(&swr_ctx);
    }
    
    // 4. Close encoders
    if (enc_ctx) {
        avcodec_free_context(&enc_ctx);
    }
    
    // 5. Close decoders
    if (dec_ctx) {
        avcodec_free_context(&dec_ctx);
    }
    
    // 6. Close output file
    if (ofmt_ctx) {
        if (!(ofmt_ctx->oformat->flags & AVFMT_NOFILE)) {
            avio_closep(&ofmt_ctx->pb);
        }
        avformat_free_context(ofmt_ctx);
    }
    
    // 7. Close input file
    if (ifmt_ctx) {
        avformat_close_input(&ifmt_ctx);
    }
}

int safe_transcode(const char *input, const char *output) {
    AVFormatContext *ifmt_ctx = NULL, *ofmt_ctx = NULL;
    AVCodecContext *dec_ctx = NULL, *enc_ctx = NULL;
    SwrContext *swr_ctx = NULL;
    AVPacket *pkt = NULL;
    AVFrame *frame = NULL;
    int ret;
    
    // Allocate
    pkt = av_packet_alloc();
    frame = av_frame_alloc();
    if (!pkt || !frame) {
        ret = AVERROR(ENOMEM);
        goto cleanup;
    }
    
    // Open input/output
    ret = avformat_open_input(&ifmt_ctx, input, NULL, NULL);
    if (ret < 0) goto cleanup;
    
    // ... setup decoders, encoders, etc.
    
cleanup:
    cleanup_all_resources(ifmt_ctx, ofmt_ctx, dec_ctx, enc_ctx,
                         swr_ctx, pkt, frame);
    return ret;
}
```

### 10.3 Buffer Reference Management

```c
#include <libavutil/buffer.h>

void demonstrate_buffer_unref(void) {
    // For AVBuffer
    AVBufferRef *buf = av_buffer_alloc(1024);
    if (buf) {
        // Use buffer...
        av_buffer_unref(&buf);  // Release reference
    }
    
    // For frame data
    AVFrame *frame = av_frame_alloc();
    if (frame) {
        // After avcodec_receive_frame, frame owns buffer references
        ret = avcodec_receive_frame(dec_ctx, frame);
        if (ret >= 0) {
            // Use frame->data, frame->buf, etc.
            
            // Important: unref to release references
            av_frame_unref(frame);
        }
        av_frame_free(&frame);
    }
}
```

---

## 11. avformat_close_input AND PROPER CLEANUP

### 11.1 Complete Cleanup Pattern

```c
#include <libavformat/avformat.h>

void demonstrate_proper_cleanup(void) {
    AVFormatContext *fmt_ctx = avformat_alloc_context();
    
    // Open input
    int ret = avformat_open_input(&fmt_ctx, "input.mp3", NULL, NULL);
    if (ret < 0) {
        avformat_free_context(&fmt_ctx);  // Only free context if open failed
        return;
    }
    
    // Get stream info
    ret = avformat_find_stream_info(fmt_ctx, NULL);
    if (ret < 0) {
        // Note: close_input frees the context
        avformat_close_input(&fmt_ctx);
        return;
    }
    
    // Process...
    
    // Clean up: close_input frees context automatically
    avformat_close_input(&fmt_ctx);
    // fmt_ctx is now NULL
    
    // If you need to keep the context but close the input:
    avformat_close_input(&fmt_ctx);
    // fmt_ctx is still allocated but input is closed
    // You can call avformat_open_input again
    // When done for real:
    avformat_free_context(&fmt_ctx);
}
```

### 11.2 Multiple Context Cleanup

```c
#include <libavformat/avformat.h>

void cleanup_multiple_contexts(AVFormatContext **contexts, int count) {
    for (int i = 0; i < count; i++) {
        if (contexts[i]) {
            // close_input frees the context
            avformat_close_input(&contexts[i]);
            contexts[i] = NULL;
        }
    }
}

// Usage
void process_files(void) {
    AVFormatContext *inputs[3] = {NULL, NULL, NULL};
    AVFormatContext *output = NULL;
    
    // Open inputs
    avformat_open_input(&inputs[0], "file1.mp3", NULL, NULL);
    avformat_open_input(&inputs[1], "file2.mp3", NULL, NULL);
    
    // Open output
    avformat_alloc_output_context2(&output, NULL, NULL, "output.mp3");
    
    // Process...
    
    // Clean up
    cleanup_multiple_contexts(inputs, 3);
    
    if (output) {
        if (!(output->oformat->flags & AVFMT_NOFILE)) {
            avio_closep(&output->pb);
        }
        avformat_free_context(output);
    }
}
```

---

## 12. RESOURCE LIMITS

### 12.1 Setting Format Context Limits

```c
#include <libavformat/avformat.h>

void configure_resource_limits(AVFormatContext *fmt_ctx) {
    // Maximum time to analyze stream (microseconds)
    // Default: 5 seconds for no-header formats, 8 seconds otherwise
    fmt_ctx->max_analyze_duration = 10000000;  // 10 seconds
    fmt_ctx->max_analyze_duration_fixed = 10000000;
    
    // Size of probe buffer
    fmt_ctx->probesize = 5 * 1024 * 1024;  // 5 MB
    fmt_ctx->format_probesize = 5 * 1024 * 1024;
    
    // Maximum URL length
    // Default: 1024 bytes
    
    // Maximum packet size
    // Default: various
    
    // Maximum streams
    // Default: no limit
    
    // RTSP specific
    // fmt_ctx-> RTSPflags
}

void configure_decoding_limits(AVCodecContext *dec_ctx) {
    // Maximum threads
    dec_ctx->thread_count = 4;
    
    // Threading type
    dec_ctx->thread_type = FF_THREAD_FRAME | FF_THREAD_SLICE;
    
    // Skip loop filter (for faster but lower quality)
    // dec_ctx->skip_loop_filter = AVDISCARD_ALL;
    
    // Skip IDCT
    // dec_ctx->skip_idct = AVDISCARD_DEFAULT;
    
    // Skip frame
    // dec_ctx->skip_frame = AVDISCARD_DEFAULT;
}
```

### 12.2 Network Stream Limits

```c
#include <libavformat/avformat.h>

void configure_network_limits(AVFormatContext *fmt_ctx) {
    // RTSP timeout (microseconds)
    // Not directly available in AVFormatContext
    
    // TCP options
    AVDictionary *opts = NULL;
    
    // RTSP specific options
    av_dict_set(&opts, "stimeout", "10000000", 0);  // 10 second socket timeout
    av_dict_set(&opts, "timeout", "10000000", 0);
    
    // TCP no-delay (disable Nagle)
    av_dict_set(&opts, "tcp_nodelay", "1", 0);
    
    // Buffer size
    av_dict_set(&opts, "buffer_size", "1048576", 0);  // 1 MB
    
    // Maximum reconnect attempts
    av_dict_set(&opts, "reconnect", "1", 0);
    av_dict_set(&opts, "reconnect_on_network_error", "1", 0);
    av_dict_set(&opts, "reconnect_on_http_error", "1", 0);
    av_dict_set(&opts, "reconnect_streamed", "1", 0);
    av_dict_set(&opts, "reconnect_delay_max", "5", 0);
    
    // HTTP specific
    av_dict_set(&opts, "user-agent", "FFmpeg/7.0", 0);
    av_dict_set(&opts, "referer", "", 0);
    
    // Apply via AVFormatContext options
    // These need to be set before avformat_open_input
    // via AVDictionary parameter
}
```

---

## 13. LOGGING AND ERROR REPORTING

### 13.1 Configuring Log Levels

```c
#include <libavutil/log.h>

void configure_logging(void) {
    // Set minimum log level
    // Levels: AV_LOG_PANIC, AV_LOG_FATAL, AV_LOG_ERROR, AV_LOG_WARNING,
    //         AV_LOG_INFO, AV_LOG_VERBOSE, AV_LOG_DEBUG, AV_LOG_TRACE
    
    av_log_set_level(AV_LOG_WARNING);  // Only warnings and errors
    
    // Or use environment variable
    // export FFREPORT=level=warning
    
    // For specific component logging:
    // av_log_set_level_for_context(AV_LOG_ERROR, specific_context);
}

void log_with_context(void) {
    // Each FFmpeg function can log with context
    
    AVFormatContext *fmt_ctx = avformat_alloc_context();
    
    // Log to specific context
    av_log(fmt_ctx, AV_LOG_INFO, "Opening file\n");
    av_log(fmt_ctx, AV_LOG_ERROR, "Failed: %s\n", av_err2str(ret));
    
    // Log without context
    av_log(NULL, AV_LOG_INFO, "General info\n");
}
```

### 13.2 Custom Log Handler

```c
#include <libavutil/log.h>

static void custom_log_callback(void *ptr, int level, const char *fmt, va_list vl) {
    // Filter by level
    if (level > av_log_get_level()) {
        return;
    }
    
    // Format message
    char message[1024];
    vsnprintf(message, sizeof(message), fmt, vl);
    
    // Get component name
    const char *component = "ffmpeg";
    if (ptr) {
        AVClass *cls = *(AVClass **)ptr;
        if (cls && cls->item_name) {
            component = cls->item_name(ptr);
        }
    }
    
    // Get level name
    const char *level_name = "unknown";
    switch (level & 0xFF) {
        case AV_LOG_PANIC:   level_name = "PANIC"; break;
        case AV_LOG_FATAL:   level_name = "FATAL"; break;
        case AV_LOG_ERROR:   level_name = "ERROR"; break;
        case AV_LOG_WARNING: level_name = "WARN"; break;
        case AV_LOG_INFO:    level_name = "INFO"; break;
        case AV_LOG_VERBOSE: level_name = "VERB"; break;
        case AV_LOG_DEBUG:   level_name = "DEBUG"; break;
        case AV_LOG_TRACE:   level_name = "TRACE"; break;
    }
    
    // Print to stderr
    fprintf(stderr, "[%s] %s: %s", level_name, component, message);
}

void setup_custom_logging(void) {
    av_log_set_callback(custom_log_callback);
}

void restore_default_logging(void) {
    av_log_set_callback(av_default_log_callback);
}
```

---

## 14. PARTIAL OUTPUT AND INTERRUPTION

### 14.1 Handling Interrupted Conversion

```c
#include <libavformat/avformat.h>
#include <libavcodec/avcodec.h>

typedef struct {
    volatile int interrupted;
    volatile int output_written;
} ConversionState;

static int conversion_interrupt(void *arg) {
    ConversionState *state = (ConversionState *)arg;
    return state->interrupted;
}

int interrupted_transcode(const char *input, const char *output) {
    AVFormatContext *ifmt_ctx = NULL, *ofmt_ctx = NULL;
    ConversionState state = {0};
    int ret;
    
    ret = avformat_open_input(&ifmt_ctx, input, NULL, NULL);
    if (ret < 0) goto cleanup;
    
    ret = avformat_alloc_output_context2(&ofmt_ctx, NULL, NULL, output);
    if (ret < 0 || !ofmt_ctx) goto cleanup;
    
    // Setup interrupt callback
    ofmt_ctx->interrupt_callback.callback = conversion_interrupt;
    ofmt_ctx->interrupt_callback.opaque = &state;
    
    // Write header
    ret = avformat_write_header(ofmt_ctx, NULL);
    if (ret < 0) goto cleanup;
    
    // Main loop
    AVPacket *pkt = av_packet_alloc();
    while (!state.interrupted) {
        ret = av_read_frame(ifmt_ctx, pkt);
        if (ret < 0) {
            if (ret == AVERROR_EOF) {
                break;  // Normal end
            } else if (ret == AVERROR_EXIT) {
                fprintf(stderr, "Interrupted\n");
                break;
            }
            // Log and continue for other errors
            continue;
        }
        
        // Process packet
        ret = av_interleaved_write_frame(ofmt_ctx, pkt);
        av_packet_unref(pkt);
        
        if (ret >= 0) {
            state.output_written = 1;
        }
        
        if (ret < 0 && ret != AVERROR_EXIT) {
            fprintf(stderr, "Write error: %s\n", av_err2str(ret));
        }
    }
    
    // Write trailer if interrupted but have some output
    if (state.output_written && !state.interrupted) {
        av_write_trailer(ofmt_ctx);
    } else if (state.interrupted) {
        // Partial output - truncate or keep based on requirements
        fprintf(stderr, "Warning: Output may be incomplete\n");
        // Could delete partial output here
        // unlink(output);
    }
    
    av_packet_free(&pkt);
    
cleanup:
    if (ret < 0 && ret != AVERROR_EXIT) {
        fprintf(stderr, "Error: %s\n", av_err2str(ret));
    }
    
    if (ifmt_ctx) avformat_close_input(&ifmt_ctx);
    if (ofmt_ctx) {
        if (!(ofmt_ctx->oformat->flags & AVFMT_NOFILE)) {
            avio_closep(&ofmt_ctx->pb);
        }
        avformat_free_context(ofmt_ctx);
    }
    
    return ret;
}
```

### 14.2 CLI with -xerror

```bash
# Exit on first error
ffmpeg -i input.wav -c:a flac output.flac -xerror

# With pipe and error checking
ffmpeg -i input.wav -c:a flac -progress pipe:1 output.flac 2>&1 || {
    echo "Conversion failed"
    rm -f output.flac
    exit 1
}

# Fail on any error in pipe chain
set -o pipefail
ffmpeg -i input.wav -f wav - 2>/dev/null | \
    ffmpeg -f wav -i - -c:a alac output.m4a -xerror
```

---

## 15. STREAM SELECTION ERRORS

### 15.1 Handling Stream Not Found

```c
#include <libavformat/avformat.h>
#include <libavcodec/avcodec.h>

int find_best_stream_robust(AVFormatContext *fmt_ctx, AVMediaType type) {
    int stream_index;
    const AVCodec *dec = NULL;
    int ret;
    
    // Find best stream
    stream_index = av_find_best_stream(fmt_ctx, type, -1, -1, &dec, 0);
    
    if (stream_index < 0) {
        switch (stream_index) {
            case AVERROR_STREAM_NOT_FOUND:
                fprintf(stderr, "No stream of type %d found\n", type);
                break;
            case AVERROR_DECODER_NOT_FOUND:
                fprintf(stderr, "No decoder found for stream\n");
                break;
            case AVERROR_INVALIDDATA:
                fprintf(stderr, "Invalid data in stream\n");
                break;
            default:
                fprintf(stderr, "Error finding stream: %s\n", 
                        av_err2str(stream_index));
        }
        return stream_index;
    }
    
    // Verify decoder is available
    if (!dec) {
        fprintf(stderr, "No suitable decoder found for stream %d\n", 
                stream_index);
        return AVERROR_DECODER_NOT_FOUND;
    }
    
    printf("Found stream %d with decoder %s\n", stream_index, dec->name);
    return stream_index;
}

void list_available_streams(AVFormatContext *fmt_ctx) {
    printf("Available streams:\n");
    for (unsigned int i = 0; i < fmt_ctx->nb_streams; i++) {
        AVStream *stream = fmt_ctx->streams[i];
        AVCodecParameters *par = stream->codecpar;
        
        const char *type_str = "unknown";
        switch (par->codec_type) {
            case AVMEDIA_TYPE_VIDEO: type_str = "video"; break;
            case AVMEDIA_TYPE_AUDIO: type_str = "audio"; break;
            case AVMEDIA_TYPE_SUBTITLE: type_str = "subtitle"; break;
            case AVMEDIA_TYPE_DATA: type_str = "data"; break;
            default: type_str = "other"; break;
        }
        
        const AVCodec *dec = avcodec_find_decoder(par->codec_id);
        const char *dec_name = dec ? dec->name : "none";
        
        printf("  Stream %d: %s, codec=%s (%s), %d Hz, %d channels\n",
               i, type_str, avcodec_get_name(par->codec_id), dec_name,
               par->sample_rate, par->ch_layout.nb_channels);
    }
}
```

---

## 16. CODEC COMPATIBILITY

### 16.1 Checking Codec Availability

```c
#include <libavcodec/avcodec.h>

void check_codec_availability(void) {
    // List all available encoders
    printf("Available audio encoders:\n");
    void *opaque = NULL;
    const AVCodec *c = NULL;
    while ((c = av_codec_iterate(&opaque))) {
        if (av_codec_is_encoder(c) && c->type == AVMEDIA_TYPE_AUDIO) {
            printf("  %s (%s)\n", c->name, c->long_name ? c->long_name : "");
        }
    }
    
    // Check specific encoder
    const AVCodec *flac = avcodec_find_encoder_by_name("flac");
    if (flac) {
        printf("FLAC encoder found\n");
        printf("  Supported sample formats: ");
        for (int i = 0; flac->sample_fmts && flac->sample_fmts[i] != AV_SAMPLE_FMT_NONE; i++) {
            printf("%s ", av_get_sample_fmt_name(flac->sample_fmts[i]));
        }
        printf("\n");
        
        printf("  Supported sample rates: ");
        for (int i = 0; flac->supported_samplerates && flac->supported_samplerates[i] > 0; i++) {
            printf("%d ", flac->supported_samplerates[i]);
        }
        printf("\n");
    } else {
        printf("FLAC encoder NOT found - recompile with FLAC support\n");
    }
    
    // Check decoder
    const AVCodec *dsd = avcodec_find_decoder_by_name("dsd_msbf");
    if (dsd) {
        printf("DSD MSBF decoder found\n");
    } else {
        printf("DSD decoder NOT found\n");
    }
}
```

### 16.2 Handling Unsupported Codecs

```c
#include <libavcodec/avcodec.h>

int handle_unsupported_codec(AVCodecParameters *par) {
    const AVCodec *dec = avcodec_find_decoder(par->codec_id);
    
    if (!dec) {
        fprintf(stderr, "Codec %s is not supported by this FFmpeg build\n",
                avcodec_get_name(par->codec_id));
        fprintf(stderr, "Build with additional codec support\n");
        return AVERROR_DECODER_NOT_FOUND;
    }
    
    // Check if decoder is experimental
    if (dec->capabilities & AV_CODEC_CAP_EXPERIMENTAL) {
        fprintf(stderr, "Warning: %s is experimental, enable experimental codecs\n",
                dec->name);
        // Decoder might still work
    }
    
    // Check capabilities
    if (par->codec_type == AVMEDIA_TYPE_AUDIO) {
        if (!(dec->capabilities & AV_CODEC_CAP_DR1)) {
            fprintf(stderr, "Warning: Decoder may require additional buffers\n");
        }
    }
    
    return 0;
}
```

---

## 17. QUICK REFERENCE

### 17.1 Common Error Handling Patterns

```c
// Pattern 1: Check and return
if (ret < 0) {
    fprintf(stderr, "Error: %s\n", av_err2str(ret));
    goto cleanup;
}

// Pattern 2: Log and continue
if (ret < 0) {
    fprintf(stderr, "Warning: %s\n", av_err2str(ret));
    continue;
}

// Pattern 3: Attempt recovery
if (ret == AVERROR_INVALIDDATA) {
    // Try to recover
    avcodec_flush_buffers(dec_ctx);
    continue;
}

// Pattern 4: Timeout handling
if (ret == AVERROR_EXIT) {
    fprintf(stderr, "Operation interrupted\n");
    return ret;
}
```

### 17.2 Error Code Quick Reference

```c
// Common error checks
if (ret == AVERROR_EOF)           // End of file/stream
if (ret == AVERROR_INVALIDDATA)   // Corrupt/invalid data
if (ret == AVERROR(EAGAIN))       // Resource temporarily unavailable
if (ret == AVERROR_ENOMEM)        // Out of memory
if (ret == AVERROR_EXIT)          // Interrupted
if (ret == AVERROR_STREAM_NOT_FOUND)  // No matching stream

// Error to string
fprintf(stderr, "%s\n", av_err2str(ret));
```

### 17.3 Cleanup Order

```
1. av_frame_free()
2. av_packet_free()
3. swr_free()
4. avcodec_free_context()  // for both enc and dec
5. avformat_close_input()
6. avio_closep()
7. avformat_free_context()
```

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
