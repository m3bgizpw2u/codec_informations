# FFmpeg Gapless Playback Implementation — Deep Technical Reference

> **Category:** FFmpeg API
> **File Extensions:** (format-agnostic)
> **MIME Types:** `audio/*`
> **Standardization Body:** FFmpeg Project / codec-specific bodies (Xiph.org, LAME, FLAC project)
> **Primary Specification:** FFmpeg source (libavcodec, libavformat), Xiph OGG spec, LAME tag format
> **Patent Status:** Per-codec
> **License:** FFmpeg: LGPL 2.1+; codec-specific licenses
> **Current Version:** Active development
> **Active Development:** Yes — rolling release

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 What Gapless Playback Is

Gapless playback is the seamless reproduction of consecutive audio tracks without audible gaps, clicks, or silences between them. When a listener queues multiple tracks — especially albums recorded as continuous performances — the absence of inter-track silence is critical for preserving the listening experience.

```
┌─────────────────────────────────────────────────────────────────┐
│                    WITH GAPLESS PLAYBACK                          │
├─────────────────────────────────────────────────────────────────┤
│ Track 1 ████████████|███████████████|███████████████           │
│                                 │                                 │
│ Track 2 ████████████|███████████████|███████████████            │
│                                                                   │
│ Zero silence between tracks. Decoder padding is trimmed.          │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                   WITHOUT GAPLESS PLAYBACK                        │
├─────────────────────────────────────────────────────────────────┤
│ Track 1 ████████████|███████████████|███████████████             │
│                                 │    SILENCE    │                 │
│ Track 2                   ░░░░░░░|███████████████|████████     │
│                                                                   │
│ Audible gap (~10–50ms) between tracks. Listener hears silence.   │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Why Gapless Playback is Necessary

**The problem arises from:**
1. **Encoder delay:** Transform-based codecs (MP3, AAC, Vorbis, Opus) use overlapping windowed transforms (MDCT). The encoder adds "padding" samples at the start and end of the input to allow the transform to work correctly.
2. **Decoder lookahead:** Decoders need to look ahead into future samples to properly window and reconstruct the audio.
3. **Window alignment:** Codecs like MP3 use 1152-sample frames with 50% overlap; the first and last frames require extra samples that aren't part of the original audio.

**The result:**
- The encoded audio is longer than the original source
- Without metadata signaling, decoders output silence where the encoder added padding
- Players must know how many samples to skip and trim to achieve gapless playback

### 1.3 History

| Year | Development |
|------|------------|
| 1993 | MPEG-1 audio finalized; encoder delay problem identified |
| 1998 | LAME encoder adds delay/padding to LAME tag |
| ~2003 | Apple introduces iTunSMPB tag for iTunes gapless |
| 2004 | Xiph standardizes Vorbis gapless via OGG granule positions |
| 2007 | FLAC adds total samples field to STREAMINFO block |
| 2017 | FFmpeg parses iTunSMPB tag for MP3 gapless |
| 2022 | FFmpeg improves iTunSMPB parsing in mp3dec |

---

## 2. ENCODER DELAY AND PADDING — TECHNICAL DEEP DIVE

### 2.1 The MDCT and Windowed Transform Problem

Most modern audio codecs use a Modified Discrete Cosine Transform (MDCT) with window overlapping. This creates delay because:

1. **MDCT requires 50% overlap:** The transform works on N samples but produces N frequency coefficients. To reconstruct the original signal, 2N samples must be processed at a time.
2. **Encoder lookahead:** The encoder must see samples beyond the frame boundaries to compute correct window overlaps.
3. **Decoder lookahead:** The decoder must look ahead to properly reconstruct the beginning and end of each frame.

```
┌────────────────────────────────────────────────────────────────┐
│              MDCT Windowing and Padding                         │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Encoder Input (source samples):                                 │
│  [  S1  S2  S3  S4  S5  S6  S7  S8  S9  S10 ... ]            │
│                                                                 │
│  Window 1: [PAD PAD PAD] [ S1  S2  S3  S4  S5  S6 ]          │
│            ↑ delay                     (2N samples, N overlap)   │
│                                                                 │
│  Window 2:          [ S5  S6  S7  S8  S9  S10 ] [PAD PAD PAD] │
│                                 ↑ end padding                   │
│                                                                 │
│  Output: [DELAY_FRAME] [FRAME1] [FRAME2] ... [FRAME_N] [PAD_FRAME] │
│          ↑ skip on decode          ↑ trim on decode               │
└────────────────────────────────────────────────────────────────┘
```

### 2.2 Per-Codec Encoder Delay Values

| Codec | Encoder Delay | Padding | Total Extra | Notes |
|-------|-------------|---------|-------------|-------|
| MP3 (LAME) | 1056 samples | 528 samples | 1584 samples | Most problematic |
| AAC (FDK) | encoderDelay | encoderDelayTail | Variable | Reported via `initial_padding` |
| Vorbis | 0 | 0 | 0 | Implicit — handled in OGG granule |
| Opus | 0 | 0 | 0 | Implicit — 2.5ms look-ahead, reported |
| FLAC | 0 | 0 | 0 | Total samples field handles it |
| ALAC | 0 | 0 | 0 | No padding needed |
| WAV (PCM) | 0 | 0 | 0 | Uncompressed, no padding |

### 2.3 LAME/MP3 Delay Details

LAME's encoder delay is particularly complex:

```c
// From libavcodec/libmp3lame.c
avctx->initial_padding = lame_get_encoder_delay(s->gfp) + 528 + 1;
```

Breaking this down:
- `lame_get_encoder_delay(gfp)`: LAME's internal delay (typically 1056 samples at 44.1kHz)
- `+ 528`: MDCT lookahead/overlap compensation
- `+ 1`: Frame boundary adjustment

**Total delay for LAME:** approximately **1584 samples** at 44.1kHz, or approximately **35.9ms**.

The end padding is **528 samples** (approximately **12ms**).

The iTunSMPB format encodes these values as:
- `start_pad`: The delay at the beginning
- `end_pad`: The padding at the end

### 2.4 AAC Delay Details

The Fraunhofer FDK AAC encoder reports delay through the encoder API:

```c
// From libavcodec/libfdk-aacenc.c
avctx->initial_padding = info.nDelay;  // or info.encoderDelay on older versions
```

FDK AAC's delay varies based on profile and configuration:
- **AAC-LC:** ~2048 samples (~46ms at 44.1kHz)
- **HE-AAC:** Variable depending on SBR configuration
- **HE-AAC v2:** Variable depending on PS configuration

The end padding is handled via `discard_padding` in the last encoded packet.

### 2.5 Vorbis/Opus Delay

Vorbis and Opus handle gapless differently from MP3/AAC:

- **Vorbis:** The Vorbis encoder does not add explicit delay/padding. Instead, the exact number of PCM samples encoded is known, and the OGG container stores the final granule position in each page header. The player calculates the actual audio length by reading the final granule position and comparing it with the stream's declared sample count.

- **Opus:** Opus also uses implicit gapless signaling. The container stores the total number of PCM samples in the header. Since Opus uses a lookahead of only 2.5ms (120 samples at 48kHz), the delay is minimal and consistent.

---

## 3. CONTAINER-LEVEL GAPLESS SIGNALING

### 3.1 How Containers Store Delay/Padding Information

Each container format handles gapless signaling differently:

| Container | Signaling Method | FFmpeg Field | Notes |
|-----------|-----------------|-------------|-------|
| MP3 | iTunSMPB ID3v2 tag | `initial_padding`, `discard_padding` | String format |
| MP4/M4A | Edit lists / `decodable` | `initial_padding`, `discard_padding` | Built-in MP4 |
| OGG | Granule position | `start_trimming`, `end_trimming` | Implicit |
| FLAC | STREAMINFO total samples | Total samples field | Implicit |
| WAV | cue chunks | Manual cue parsing | Non-standard |
| AIFF | ANNO chunk | Manual parsing | Non-standard |
| Matroska | Cluster timestamps | `start_trimming`, `end_trimming` | Via codec private |

### 3.2 MP3: iTunSMPB Tag

The iTunSMPB tag (also called the iTunes gapless info tag) is an ID3v2 comment frame that stores encoder delay and padding in a specific hexadecimal format.

**Tag structure:**
```
ID3v2 Frame: "COMM" (Comment)
Language: "eng"
Descriptor: "iTunSMPB"
Value format: "00000000 {start_pad_hex} {end_pad_hex} {total_valid_samples_hex} 00000000 {last_eight_frames_hex}"
```

**Example:**
```
iTunSMPB= 00000000 00000B40 000001E0 00000000000E2C40 00000000 00000000
```

Decoding:
| Field | Hex Value | Decimal | Meaning |
|-------|-----------|---------|---------|
| `start_pad` | `00000B40` | 2880 | Delay at start (samples) |
| `end_pad` | `000001E0` | 480 | Padding at end (samples) |
| `total_valid` | `00000000000E2C40` | 925,120 | Valid audio samples |
| `last_eight` | `00000000` | 0 | Last eight frames offset |

The actual `end_pad` in FFmpeg's `initial_padding`/`discard_padding` fields is computed as:
```
discard_padding = end_pad - (528 + 1)   // For LAME
```

**FFmpeg's handling of iTunSMPB** (from libavformat/mp3dec.c):

```c
// FFmpeg parses iTunSMPB and sets:
st->codecpar->initial_padding = mp3->start_pad;
st->codecpar->trailing_padding = end_pad - (528 + 1);
```

### 3.3 MP4/M4A: Edit Lists and Codec Private

MP4 containers support gapless playback through two mechanisms:

#### Mechanism 1: Edit Lists
Edit lists in MP4 (`edts` atom) define timeline mappings:

```
Edit List Atom (edts):
├── Duration: total duration in movie timescale
├── Media Time: start time in media timescale
├── Media Rate: playback speed (1.0 = normal)
└── Varies with sample durations
```

For gapless, the edit list specifies:
- `media_time`: The time to start playback (after encoder delay)
- The edit list excludes the delay samples from the declared duration

#### Mechanism 2: `decodable` Duration (AAC-specific)
iTunes and Apple devices use a `decodable` field to report the total decodable samples. The actual playable duration is:
```
playable_samples = total_samples - initial_padding - trailing_padding
```

#### FFmpeg's Handling of MP4 Gapless
```c
// AAC: initial_padding set during decode
avctx->initial_padding = info.nDelay;  // From libfdk-aacenc.c

// Trailing padding via packet side data
AV_PKT_DATA_SKIP_SAMPLES:
  bytes[0-3]: initial_padding (uint32 LE)
  bytes[4-7]: discard_padding (uint32 LE)

// The muxer writes this into the MP4 container
```

### 3.4 OGG: Granule Positions

OGG containers store gapless information implicitly through granule positions in page headers.

**OGG page header structure:**
```
Byte 0-4:    Capture pattern (0x4F 0x67 0x67 0x53 0x00)
Byte 5:      Stream structure version
Byte 6:      Header type (0x01 = continuation, 0x02 = BOS, 0x04 = EOS)
Byte 7-13:   Granule position (variable-length integer)
Byte 14-17:  Serial number
Byte 18-21:  Page sequence number
Byte 22-25:  CRC checksum
Byte 26:     Number of segments
Byte 27+:    Segment table
```

**Granule position semantics for Vorbis:**
- For audio streams, granule position = sample index of the LAST valid sample in the page
- If the page contains 2048 samples ending at sample 10000, granule = 10000
- If the final page is shorter than a full block, granule position indicates where the stream actually ends
- The decoder calculates: `actual_samples = granule_position - encoder_delay`

**FFmpeg's handling (from libavformat/oggdec.c):**

```c
struct ogg_stream {
    uint64_t granule;           // Granule position
    int start_trimming;         // Samples to drop from start
    int end_trimming;           // Samples to drop from end
    // ...
};

// When reading a page:
if (os->codec && os->codec->granule_is_start)
    pts = ogg_gptopts(s, idx, os->granule, dts);
```

### 3.5 FLAC: Total Samples Field

FLAC's STREAMINFO block contains a 36-bit field for total sample count:

```
┌────────────────────────────────────────────────────────────────┐
│              FLAC STREAMINFO Block                             │
├────────────────────────────────────────────────────────────────┤
│ Offset   Size   Field                     Notes                 │
│ 0        1     BLOCK_TYPE (STREAMINFO)   = 0                   │
│ 1        2     MIN_BLOCK_SIZE            in samples            │
│ 3        2     MAX_BLOCK_SIZE            in samples            │
│ 5        3     MIN_FRAME_SIZE            in bytes              │
│ 8        3     MAX_FRAME_SIZE            in bytes              │
│ 11       8     SAMPLE_RATE/CHANNELS/etc  combined field        │
│ 19       4     TOTAL_SAMPLES[0..31]      36-bit, upper 4 bits  │
│ 23       16    MD5 signature             full file checksum     │
└────────────────────────────────────────────────────────────────┘
```

The `total_samples` field directly tells the decoder how many PCM samples are in the stream. Since FLAC doesn't add encoder delay, `total_samples` = actual audio samples.

**FFmpeg's handling:**
```c
// From libavformat/flacdec.c
st->duration = av_rescale_q(flac->total_samples,
                            (AVRational){1, s->sample_rate},
                            st->time_base);
```

### 3.6 WAV: Cue Chunks

WAV files can signal gapless information via the `cue ` chunk (RIFF INFO):

```
┌────────────────────────────────────────────────────────────────┐
│              WAV cue chunk structure                           │
├────────────────────────────────────────────────────────────────┤
│ Offset   Size   Field               Notes                       │
│ 0        4     "cue "              Chunk ID                    │
│ 4        4     chunk_size          Size of rest                │
│ 8        4     num_cues           Number of cue points         │
│ 12       24*N  cue points          N cue points                │
└────────────────────────────────────────────────────────────────┘
```

WAV does not natively support encoder delay signaling. For gapless WAV, the `cue ` chunk is sometimes used to mark the start and end of the "playable" region, but this is non-standard and not universally supported.

### 3.7 AIFF: Annotation Chunks

AIFF files can store gapless information in annotation chunks (ANNO). The format is similar to WAV's cue system and is also non-standard.

---

## 4. HOW FFMPEG HANDLES GAPLESS

### 4.1 Input-Side: Reading Delay Information

FFmpeg reads gapless information during demuxing and stores it in codec context and stream fields:

```c
// Key fields populated by FFmpeg demuxers:
AVCodecContext *avctx;  // Decoder context

avctx->initial_padding  // Samples of encoder delay at start
avctx->trailing_padding // Samples of encoder padding at end

// For MP3:
avctx->initial_padding = mp3->start_pad;
avctx->trailing_padding = mp3->end_pad - (528 + 1);

// For AAC:
avctx->initial_padding = info.nDelay;  // From FDK encoder

// For OGG/Vorbis:
os->start_trimming  // Set during demuxing
os->end_trimming    // Set during demuxing
```

### 4.2 Output-Side: Writing Delay Information

When encoding, FFmpeg writes gapless information to the output container:

```c
// Key fields for encoding:
AVCodecContext *avctx;  // Encoder context

avctx->initial_padding  // Set to encoder's delay
avctx->frame_size       // Encoder frame size

// The muxer writes these values appropriately:
// - MP4: edit lists + decodable duration
// - OGG: granule positions
// - MP3: iTunSMPB tag (if configured)
```

### 4.3 FFmpeg Flags for Gapless

| Flag | Purpose | Notes |
|------|---------|-------|
| `-avoid_negative_ts make_zero` | Shift timestamps to start at 0 | Helps with gapless demuxing |
| `-avoid_negative_ts make_non_negative` | Shift timestamps but don't go below 0 | |
| `-map_metadata -1` | Don't copy metadata | Prevents iTunSMPB loss if not wanted |
| `-movflags +gaplessinfo` | MP4: Write gapless info atom | |
| `-movflags +frag_keyframe` | Fragmented MP4 | For streaming |
| `-flags +bitexact` | Bit-exact output | For verification |

### 4.4 FFmpeg Side Data for Gapless

FFmpeg uses packet side data to communicate delay/padding to muxers:

```c
// AV_PKT_DATA_SKIP_SAMPLES side data
// Written by encoders and read by muxers

uint8_t *side_data = av_packet_new_side_data(pkt,
    AV_PKT_DATA_SKIP_SAMPLES, 10);

// First 4 bytes: initial_padding (samples to skip at start)
AV_WL32(side_data + 0, avctx->initial_padding);

// Next 4 bytes: discard_padding (samples to discard at end)
AV_WL32(side_data + 4, discard_padding);
```

### 4.5 Gapless Seeking

For gapless seeking, FFmpeg uses `av_seek_frame()` with specific flags:

```c
// Seek to timestamp with gapless consideration
int av_seek_frame(AVFormatContext *s, int stream_index,
                  int64_t timestamp, int flags);

// Key flags:
#define AVSEEK_FLAG_BACKWARD 0x01  // Seek to nearest keyframe before timestamp
#define AVSEEK_FLAG_BYTE     0x02  // Byte position seek
#define AVSEEK_FLAG_ANY       0x04  // Seek to any frame (not just keyframe)
#define AVSEEK_FLAG_FRAME     0x08  // Frame number seek
```

For gapless playback from a specific timestamp, FFmpeg:
1. Seeks to the nearest frame before the timestamp
2. Decodes frames until reaching the exact timestamp
3. Skips `initial_padding` samples from the beginning
4. Tracks `trailing_padding` for end-of-file handling

---

## 5. CONVERTING GAPLESS FROM ONE FORMAT TO ANOTHER

### 5.1 The Challenge

When converting between formats, gapless information must be:
1. Read from the source format
2. Properly interpreted
3. Written to the destination format

**Not all formats support all types of gapless signaling:**

| Source | Destination | Gapless Preserved? | Method |
|--------|-------------|-------------------|--------|
| MP3 | FLAC | Yes | Decode to PCM, re-encode |
| MP3 | MP3 | Yes (copy) | Stream copy preserves iTunSMPB |
| FLAC | MP3 | Yes | Decode, re-encode, new iTunSMPB needed |
| MP3 | AAC | Yes | Decode to PCM, re-encode |
| AAC | MP3 | Yes | Decode to PCM, re-encode |
| FLAC | FLAC | Yes | Stream copy or decode/re-encode |
| Vorbis | MP3 | Yes | Decode to PCM, re-encode |

### 5.2 Lossless Transcoding (Container Remux)

For bit-identical transcoding (e.g., FLAC → FLAC with different compression):

```bash
# Stream copy preserves all gapless info
ffmpeg -i input.flac -c:a copy output.flac

# This preserves:
# - FLAC total_samples (in STREAMINFO)
# - Vorbis comments
# - All metadata
```

For MP3 → MP3 remux:
```bash
# Stream copy preserves iTunSMPB tag
ffmpeg -i input.mp3 -c:a copy output.mp3
```

### 5.3 Re-encoding with Gapless Preservation

When re-encoding, the decoder must handle source delay and the encoder must set proper destination delay:

```bash
# Example: MP3 → AAC, preserving gapless
ffmpeg -i input.mp3 \
  -c:a libfdk_aac -vbr 4 \
  -movflags +gaplessinfo \
  output.m4a

# FFmpeg handles:
# 1. Read iTunSMPB from MP3 → initial_padding/trailing_padding
# 2. Decode MP3, skip initial_padding samples
# 3. Trim trailing_padding samples
# 4. Encode AAC with proper delay
# 5. Write AAC with gapless info
```

### 5.4 Bit-Exact Verification of Gapless

```bash
# Step 1: Extract gapless audio from source
ffmpeg -i source.flac -c:a pcm_s32le -ar 44100 source_raw.wav

# Step 2: Transcode to target format
ffmpeg -i source.flac -c:a libfdk_aac -vbr 4 output.m4a

# Step 3: Decode target back to raw PCM
ffmpeg -i output.m4a -c:a pcm_s32le -ar 44100 target_raw.wav

# Step 4: Compare checksums (after trimming silence from both)
ffmpeg -i source_raw.wav -ss 1584 -t $((TOTAL - 1584 - 528)) source_trimmed.wav
ffmpeg -i target_raw.wav -ss $((TARGET_DELAY)) -t $((TARGET_DURATION)) target_trimmed.wav

md5sum source_trimmed.wav target_trimmed.wav
# Must match for bit-exact gapless
```

### 5.5 FFmpeg Metadata Preservation

```bash
# Extract metadata from source
ffprobe -v quiet -print_format json -show_format input.flac > metadata.json

# Transcode with metadata
ffmpeg -i input.flac \
  -c:a libfdk_aac -vbr 4 \
  -movflags +gaplessinfo \
  -metadata title="$(jq -r '.format.tags.title' metadata.json)" \
  -metadata artist="$(jq -r '.format.tags.artist' metadata.json)" \
  output.m4a
```

---

## 6. iTunSMPB TAG — DETAILED REFERENCE

### 6.1 Format Specification

The iTunSMPB tag is an ID3v2.4 `COMM` frame with:
- **Frame ID:** `COMM`
- **Text encoding:** ISO-8859-1 (or UTF-8)
- **Language:** `eng`
- **Descriptor:** `iTunSMPB` (null-terminated)
- **Text:** Hexadecimal values separated by spaces

**Value field format:**
```
00000000 {start_pad:8} {end_pad:8} {total_valid:10} 00000000 {last_eight_frames:8}
```

| Position | Bytes | Field | Notes |
|----------|-------|-------|-------|
| 0 | 8 | Version/magic | Always `00000000` |
| 9 | 8 | start_pad | Hex samples of encoder delay |
| 18 | 8 | end_pad | Hex samples of end padding |
| 27 | 10 | total_valid | Hex total valid audio samples |
| 38 | 8 | Reserved | Always `00000000` |
| 47 | 8 | last_eight_frames | Offset to last 8 frames |

### 6.2 LAME Delay Breakdown

For LAME-encoded MP3s:

| Parameter | Value (44.1kHz) | Notes |
|-----------|-----------------|-------|
| Internal LAME delay | 1056 samples | `lame_get_encoder_delay()` |
| MDCT lookahead | 528 samples | Fixed overhead |
| Frame boundary | 1 sample | |
| **Total start_pad** | **1584 samples** | `1056 + 528 + 1` |
| End padding | 528 samples | `lame_get_encoder_padding()` |
| **FFmpeg trailing_padding** | **-1** | `528 - 528 - 1 = -1` [NEEDS VERIFICATION] |

### 6.3 FFmpeg iTunSMPB Parsing

From the FFmpeg source (libavformat/mp3dec.c):

```c
static void mp3_parse_itunes_tag(AVFormatContext *s, AVStream *st,
                                 int64_t base, int vbrtag_size,
                                 unsigned int *size, uint64_t *duration)
{
    // Parse the iTunSMPB tag value
    uint32_t zero, start_pad, end_pad, zero2;
    uint64_t temp_duration, last_eight_frames_offset;
    AVDictionaryEntry *de;

    de = av_dict_get(s->metadata, "iTunSMPB", NULL, 0);
    if (!de) return;

    sscanf(de->value,
           "%"PRIx32" %"PRIx32" %"PRIx32" %"PRIx64" %"PRIx32" %"PRIx64,
           &zero, &start_pad, &end_pad, &temp_duration,
           &zero2, &last_eight_frames_offset);

    // Validate ranges
    if (start_pad > (576 * 2 * 32) ||
        end_pad > (576 * 2 * 64)) {
        *duration = 0;
        return;
    }

    // Compute actual padding
    mp3->start_pad = start_pad;
    mp3->end_pad = end_pad - (528 + 1);  // Subtract LAME's internal padding

    // Set stream duration
    *duration = temp_duration;
}
```

---

## 7. LIBavformat GAPLESS SEEKING

### 7.1 av_seek_frame with AVSEEK_FLAG_BACKWARD

For gapless playback, use `AVSEEK_FLAG_BACKWARD` to seek to the nearest keyframe before the target:

```c
#include <libavformat/avformat.h>

// Gapless seek for playback
int64_t seek_timestamp = target_time;

// Seek to nearest keyframe BEFORE target
int ret = av_seek_frame(fmt_ctx, audio_stream_idx,
                        seek_timestamp,
                        AVSEEK_FLAG_BACKWARD);

if (ret < 0) {
    // Handle seek error
}

// Now decode from this point
// FFmpeg will handle:
// 1. Skipping initial_padding samples
// 2. Tracking trailing_padding for stream end
```

### 7.2 Manual Gapless Seeking

For precise gapless seeking with known delay values:

```c
// Given:
//   initial_padding: samples to skip from start
//   trailing_padding: samples to trim from end

// Seek past initial padding
int64_t skip_time = av_rescale_q(initial_padding,
                                  (AVRational){1, sample_rate},
                                  time_base);

int64_t seek_target = target_time + skip_time;

// Seek
av_seek_frame(fmt_ctx, stream_idx, seek_target, AVSEEK_FLAG_BACKWARD);

// When decoding, first initial_padding frames belong to "silence"
// Discard them and start playback from the real audio start
```

### 7.3 Gapless Playback with OGG/Vorbis

OGG/Vorbis uses granule positions for gapless:

```c
// From libavformat/oggdec.c

// Calculate start_trimming and end_trimming
if (os->codec && os->codec->granule_is_start)
    pts = ogg_gptopts(s, idx, os->granule, dts);

// For Vorbis, gptopts translates granule position to sample count
static uint64_t ogg_gptopts(AVFormatContext *s, int i,
                             uint64_t gp, int64_t *dts)
{
    struct ogg *ogg = s->priv_data;
    struct ogg_stream *os = ogg->streams + i;

    if (os->codec && os->codec->granule_is_start)
        return os->codec->gptopts(s, i, gp, dts);

    // Granule is end position
    return gp - ogg->curidx;
}

// The player calculates:
// actual_audio_samples = final_granule_position
// The difference between stored and declared is implicitly trimmed
```

---

## 8. IMPLEMENTATION GUIDE FOR GAPLESS-AWARE CONVERTER

### 8.1 Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                  GAPLESS-AWARE CONVERTER                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  SOURCE ──► DEMUX ──► DECODE ──► TRIM DELAY ──► ENCODE ──► MUX  │
│             │         │           │              │         │       │
│             │         │           │              │         │       │
│         Read iTunSMPB │    Skip padding     Set delay     Write │
│         Read edit list │    from start        in output    delay │
│         Read granule   │    Trim padding                    info │
│         position       │    from end                         │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### 8.2 Step-by-Step Implementation

#### Step 1: Open source and read delay info
```c
// Open source file
AVFormatContext *src_fmt = NULL;
avformat_open_input(&src_fmt, "source.mp3", NULL, NULL);
avformat_find_stream_info(src_fmt, NULL);

// Find audio stream
int audio_idx = av_find_best_stream(src_fmt, AVMEDIA_TYPE_AUDIO, -1, -1, NULL, 0);
AVStream *src_stream = src_fmt->streams[audio_idx];

// Read delay/padding from codec context
AVCodecParameters *src_par = src_stream->codecpar;
int64_t initial_padding = src_par->initial_padding;  // Samples to skip at start
int64_t trailing_padding = src_par->trailing_padding; // Samples to trim at end

// For MP3, also check iTunSMPB tag directly
AVDictionaryEntry *tag = av_dict_get(src_fmt->metadata, "iTunSMPB", NULL, 0);
if (tag) {
    // Parse iTunSMPB manually if needed
}
```

#### Step 2: Set up decoder
```c
const AVCodec *dec = avcodec_find_decoder(src_stream->codecpar->codec_id);
AVCodecContext *dec_ctx = avcodec_alloc_context3(dec);
avcodec_parameters_to_context(dec_ctx, src_stream->codecpar);
avcodec_open2(dec_ctx, dec, NULL);
```

#### Step 3: Set up encoder with delay
```c
const AVCodec *enc = avcodec_find_encoder_by_name("libfdk_aac");
AVCodecContext *enc_ctx = avcodec_alloc_context3(enc);

enc_ctx->sample_rate = src_par->sample_rate;
av_channel_layout_copy(&enc_ctx->ch_layout, &src_stream->codecpar->ch_layout);
enc_ctx->sample_fmt = AV_SAMPLE_FMT_S16P;

// Preserve delay info for encoder
enc_ctx->initial_padding = initial_padding;
enc_ctx->trailing_padding = trailing_padding;

avcodec_open2(enc_ctx, enc, NULL);
```

#### Step 4: Decode, skip delay, encode, write
```c
AVFrame *dec_frame = av_frame_alloc();
AVFrame *enc_frame = av_frame_alloc();
AVPacket *pkt = av_packet_alloc();
int64_t samples_decoded = 0;
int64_t samples_to_skip = initial_padding;
int64_t total_samples = 0;  // Track for end trimming

while (av_read_frame(src_fmt, pkt) >= 0) {
    if (pkt->stream_index == audio_idx) {
        avcodec_send_packet(dec_ctx, pkt);
        while (avcodec_receive_frame(dec_ctx, dec_frame) == 0) {
            samples_decoded += dec_frame->nb_samples;

            // Skip initial padding
            if (samples_to_skip > 0) {
                int skip = FFMIN(samples_to_skip, dec_frame->nb_samples);
                // Advance data pointer by skip samples
                // (or use swr to resample with offset)
                samples_to_skip -= skip;
                dec_frame->nb_samples -= skip;
                dec_frame->data[0] += skip * bytes_per_sample;
                // ... advance all channels
            }

            if (dec_frame->nb_samples > 0) {
                // Encode
                avcodec_send_frame(enc_ctx, dec_frame);
                while (avcodec_receive_packet(enc_ctx, pkt) == 0) {
                    av_interleaved_write_frame(dst_fmt, pkt);
                    av_packet_unref(pkt);
                }
            }
            av_frame_unref(dec_frame);
        }
    }
    av_packet_unref(pkt);
}

// Flush decoder
avcodec_send_packet(dec_ctx, NULL);
while (avcodec_receive_frame(dec_ctx, dec_frame) == 0) {
    // Process remaining frames, trim trailing_padding
    av_frame_unref(dec_frame);
}

// Flush encoder
avcodec_send_frame(enc_ctx, NULL);
while (avcodec_receive_packet(enc_ctx, pkt) == 0) {
    // pkt may have AV_PKT_DATA_SKIP_SAMPLES side data
    av_interleaved_write_frame(dst_fmt, pkt);
    av_packet_unref(pkt);
}
```

### 8.3 FFmpeg CLI Gapless Workflow

```bash
# Simple gapless transcode (FFmpeg handles delay automatically)
ffmpeg -i source.mp3 -c:a libfdk_aac -vbr 4 -movflags +gaplessinfo output.m4a

# With explicit metadata preservation
ffmpeg -i source.mp3 \
  -c:a libfdk_aac -vbr 4 \
  -movflags +gaplessinfo \
  -map_metadata 0 \
  output.m4a

# Remux FLAC to FLAC (bit-exact)
ffmpeg -i source.flac -c:a copy output.flac

# Re-encode FLAC to AAC with gapless
ffmpeg -i source.flac \
  -c:a libfdk_aac -vbr 4 \
  -movflags +gaplessinfo \
  -metadata title="Album Title" \
  -metadata artist="Artist" \
  output.m4a
```

---

## 9. PRACTICAL TESTING

### 9.1 Creating a Gapless Test File

```bash
# Create a test signal (1kHz sine wave, 5 seconds)
ffmpeg -f lavfi -i "sine=frequency=1000:duration=5" \
  -c:a pcm_s16le -ar 44100 test_sine.wav

# Verify the file has no silence at start or end
ffprobe -v quiet -show_entries format=duration:stream=nb_frames \
  -of default=noprint_wrappers=1 test_sine.wav

# Now encode with a format that adds delay
ffmpeg -i test_sine.wav -c:a libmp3lame -q:a 0 test_encoded.mp3

# Decode back
ffmpeg -i test_encoded.mp3 -c:a pcm_s16le test_decoded.wav

# Check if there's silence at the boundaries
ffmpeg -i test_decoded.wav -af "astat" -f null /dev/null
```

### 9.2 Verifying Gapless Information

```bash
# Check MP3 for iTunSMPB tag
ffprobe -v quiet -show_entries format_tags=iTunSMPB \
  -of default=noprint_wrappers=1 input.mp3

# Check FLAC total samples
ffprobe -v quiet -show_entries stream=nb_samples \
  -of default=noprint_wrappers=1 input.flac

# Check MP4/AAC duration vs sample count
ffprobe -v quiet -show_entries format=duration:stream=nb_samples,sample_rate \
  -of default=noprint_wrappers=1 input.m4a

# For OGG/Vorbis, check granule position
ffprobe -v debug -show_packets input.ogg 2>&1 | grep granule
```

### 9.3 Bit-Exact Gapless Verification

```bash
#!/bin/bash
# gapless_verify.sh — Verify gapless conversion

SOURCE="$1"
TARGET="$2"

# Get sample rates
SRC_RATE=$(ffprobe -v error -show_entries stream=sample_rate \
  -of default=noprint_wrappers=1:nokey=1 "$SOURCE")
TGT_RATE=$(ffprobe -v error -show_entries stream=sample_rate \
  -of default=noprint_wrappers=1:nokey=1 "$TARGET")

# Resample to common rate if needed
if [ "$SRC_RATE" != "$TGT_RATE" ]; then
  ffmpeg -y -i "$TARGET" -ar "$SRC_RATE" target_resampled.wav
  TARGET="target_resampled.wav"
fi

# Get source initial padding
SRC_PAD=$(ffprobe -v error -select_streams a:0 -show_entries \
  stream=initial_padding -of default=noprint_wrappers=1:nokey=1 "$SOURCE")

# Decode both to raw PCM
ffmpeg -y -i "$SOURCE" -c:a pcm_s32le source_raw.wav 2>/dev/null
ffmpeg -y -i "$TARGET" -c:a pcm_s32le target_raw.wav 2>/dev/null

# Compare
cmp source_raw.wav target_raw.wav && echo "BIT-IDENTICAL" || echo "DIFFERENT"
```

### 9.4 Testing Container Support

| Container | iTunSMPB | Edit List | Granule | FFmpeg Support |
|-----------|-----------|-----------|---------|----------------|
| MP3 | Yes | No | No | Full (since 2017) |
| M4A/AAC | Yes | Yes | N/A | Full |
| OGG | N/A | N/A | Yes | Full |
| FLAC | N/A | N/A | Yes | Full |
| WAV | Partial | No | No | Manual |
| AIFF | Partial | No | No | Manual |
| Matroska | Yes | No | Yes | Partial |

---

## 10. PER-CODEC GAPLESS REFERENCE

### 10.1 MP3 (LAME)

| Property | Value | Notes |
|----------|-------|-------|
| Encoder delay | 1584 samples (35.9ms @ 44.1kHz) | LAME 3.99+ |
| End padding | 528 samples (12ms @ 44.1kHz) | |
| Tag format | iTunSMPB | ID3v2 COMM frame |
| FFmpeg support | Full | Since 2017 patch |
| Players supporting | Most modern players | Foobar2000, Winamp, iTunes |

### 10.2 AAC (FDK)

| Property | Value | Notes |
|----------|-------|-------|
| Encoder delay | Variable (profile-dependent) | ~2048 samples typical |
| End padding | Variable | |
| Tag format | Built-in MP4 | `decodable` field |
| FFmpeg support | Full | Via side data |
| Players supporting | Apple devices, most modern players | |

### 10.3 Vorbis

| Property | Value | Notes |
|----------|-------|-------|
| Encoder delay | 0 | Implicit via granule |
| End padding | 0 | Implicit via granule |
| Tag format | OGG granule position | Implicit in container |
| FFmpeg support | Full | |
| Players supporting | Most modern players | |

### 10.4 Opus

| Property | Value | Notes |
|----------|-------|-------|
| Encoder delay | 120 samples (2.5ms @ 48kHz) | Per OGG Opus spec |
| End padding | Variable | 2.5ms lookahead |
| Tag format | OGG granule / container | Implicit |
| FFmpeg support | Full | |
| Players supporting | Most modern players | |

### 10.5 FLAC

| Property | Value | Notes |
|----------|-------|-------|
| Encoder delay | 0 | No transform padding |
| End padding | 0 | No padding |
| Tag format | STREAMINFO total samples | Explicit field |
| FFmpeg support | Full | |
| Players supporting | All players | Native support |

### 10.6 ALAC

| Property | Value | Notes |
|----------|-------|-------|
| Encoder delay | 0 | Lossless, no transform |
| End padding | 0 | Lossless, no padding |
| Tag format | MP4 container | Native |
| FFmpeg support | Full | |
| Players supporting | Apple devices, most modern | |

---

## 11. EDGE CASES AND PITFALLS

### 11.1 Edge Cases

| Scenario | Issue | Solution |
|----------|-------|----------|
| Very short file (< delay samples) | Entire file is delay | Warn user; file too short |
| Multiple concatenated MP3s | Each has own delay | Decode each separately, trim, concatenate |
| Corrupt iTunSMPB | FFmpeg ignores tag | Fall back to default delay (1584) |
| Missing end padding | Truncated audio | Track actual frame count |
| Variable delay encoder | Delay varies per frame | Use maximum delay; trim excess |
| Non-iTunSMPB MP3 encoder | Unknown delay | Use LAME default (1584/528) |

### 11.2 Common Pitfalls

1. **Forgetting initial_padding:** Decoders output silence at the start. Always skip the first `initial_padding` samples.

2. **Forgetting trailing_padding:** Decoders output silence at the end. Always trim the last `trailing_padding` samples.

3. **Mixing delay from different sources:** When concatenating tracks, each track may have different delay values. Sum them appropriately.

4. **Assuming constant delay:** Not all encoders use the same delay. LAME is 1584 samples; FDK AAC varies; Vorbis is 0.

5. **Stream copying delay:** When using `-c:a copy`, the delay info is preserved but not re-computed. Ensure the destination container supports the source's delay format.

6. **Cross-container conversion:** Converting from MP3 (iTunSMPB) to OGG (granule) requires careful handling of delay values.

---

## 12. IMPLEMENTATION CHECKLIST

### Gapless Detection
- [ ] Read `AVCodecContext.initial_padding` from source
- [ ] Read `AVCodecContext.trailing_padding` from source
- [ ] Check for iTunSMPB tag in MP3 files
- [ ] Check for edit list in MP4 files
- [ ] Check granule position for OGG files
- [ ] Check STREAMINFO total_samples for FLAC files

### Gapless Encoding
- [ ] Set `AVCodecContext.initial_padding` on encoder
- [ ] Set `AVCodecContext.trailing_padding` on encoder (if applicable)
- [ ] Verify encoder supports delay reporting
- [ ] Write appropriate delay metadata to output container

### Gapless Decoding
- [ ] Skip `initial_padding` samples at stream start
- [ ] Trim `trailing_padding` samples at stream end
- [ ] Handle players that don't support gapless (provide non-gapless fallback)

### Container Support
- [ ] MP3: Write iTunSMPB tag if encoder supports it
- [ ] MP4: Write edit list or `decodable` field
- [ ] OGG: Set correct granule positions on pages
- [ ] FLAC: Set correct total_samples in STREAMINFO

### Verification
- [ ] Create test files with known silence at boundaries
- [ ] Verify no silence when concatenating gapless files
- [ ] Test with players known to support gapless
- [ ] Test with players that don't support gapless (fallback behavior)
- [ ] Verify bit-exact reconstruction via framemd5 comparison

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete gapless playback implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
