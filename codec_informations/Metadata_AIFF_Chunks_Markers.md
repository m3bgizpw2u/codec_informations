# AIFF Metadata Chunks: MARK, INST, COMT, FVER — Deep Technical Reference
> **Category:** Metadata
> **File Extensions:** `.aiff`, `.aifc`
> **MIME Types:** `audio/x-aiff`, `audio/aiff`
> **Standardization Body:** Apple Computer (AIFF), AIIM/EAI (AIFF-C)
> **Primary Specification:** AIFF 1.3 Specification, https://www.mmsp.ece.mcgill.ca/Documents/AudioFormats/AIFF/Docs/AIFF-1.3.pdf
> **Patent Status:** Open — AIFF is a published specification
> **License:** Open
> **Current Version:** AIFF 1.3, AIFF-C 1.0
> **Active Development:** No — legacy format, maintained for compatibility

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Apple Computer, Inc. (AIFF); Apple, IBM, Microsoft (AIFF-C)
- **Year Created:** 1988 (AIFF 1.0), 1991 (AIFF 1.3), 1991 (AIFF-C)
- **Original Purpose:** Provide a high-quality audio file format for professional audio applications on Apple Macintosh computers, competing with Electronic Arts' IFF format.
- **Problem with Predecessors:** Before AIFF, audio files were typically stored in proprietary formats. AIFF standardized audio metadata alongside audio data in a well-defined IFF-based container.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| AIFF 1.0 | 1988 | Initial specification |
| AIFF 1.3 | 1991 | Added MARK, INST, COMT chunks |
| AIFF-C 1.0 | 1991 | Added compression support (ALaw, uLaw, FLAC) |
| Current | — | No further updates, legacy format |

### 1.3 Current Adoption
- **Primary use cases today:** Professional audio production, archival storage, cross-platform audio interchange
- **Platforms with native support:** macOS (native), Windows (via QuickTime/FFmpeg), Linux (via FFmpeg), iOS (native)
- **Major software using this format:** Pro Tools, Logic Pro, Cubase, Adobe Audition
- **Hardware support:** CD-DA production, professional audio equipment
- **Status:** Legacy but widely supported; being superseded by newer formats in streaming

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 AIFF/AIFF-C Container Architecture

AIFF uses the IFF (Interchange File Format) container structure. All chunks are stored in **big-endian** byte order (unlike WAV's little-endian RIFF).

```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       46 4F 52 4D     "FORM"   Container chunk ID
0x0004  4       XX XX XX XX     uint32    Container size (big-endian)
0x0008  4       41 49 46 46     "AIFF"   Form type (AIFF or "AIFC")
0x000C  n       ...             ...       Chunk data
```

### 2.2 AIFF File Structure

```
FORM AIFF
    COMM        -- Common chunk (REQUIRED)
    SSND        -- Sound data chunk (REQUIRED)
    MARK        -- Marker chunk (optional)
    INST        -- Instrument chunk (optional)
    COMT        -- Comments chunk (optional)
    NAME        -- Name chunk (optional)
    AUTHOR      -- Author chunk (optional)
    COPYRIGHT   -- Copyright chunk (optional)
    ANNO        -- Annotation chunk (optional)
    FVER        -- Format version (AIFF-C only, required)
```

### 2.3 Generic Chunk Structure

All AIFF chunks follow this format:

```
Offset  Bytes   Field Name          Type        Description
------  ------  ------------------  ----------  ----------------------------------
0x00    4       Chunk ID            char[4]     Four-character chunk identifier
0x04    4       Chunk Size          int32 BE    Size of chunk data (big-endian)
0x08    n       Chunk Data          varies      Chunk-specific data
[n%2]   0 or 1  Pad Byte            uint8       Pad byte if chunk size is odd
```

**Important:** If chunk data length is odd, a single pad byte (0x00) is added to maintain even alignment.

### 2.4 AIFF vs AIFF-C Key Differences

| Feature | AIFF | AIFF-C |
|---------|------|--------|
| Audio format | PCM only | PCM + compressed |
| Compression types | None | ALaw, uLaw, FLAC, etc. |
| FVER chunk | Not required | Required |
| COMM compression fields | Not present | Present |
| File extension | .aiff, .aif | .aifc |

---

## 3. FVER CHUNK — FORMAT VERSION

### 3.1 FVER Overview

The FVER chunk exists only in AIFF-C files and indicates the format version of the AIFF-C specification used.

### 3.2 FVER Byte Structure

```
Offset  Bytes   Field Name          Type        Description
------  ------  ------------------  ----------  ----------------------------------
0x00    4       Chunk ID            char[4]     "FVER"
0x04    4       Chunk Size          int32 BE    Always 4
0x08    4       Timestamp           uint32 BE   Seconds since 1904-01-01 00:00:00 UTC
```

### 3.3 FVER Field Details

**Timestamp (4 bytes, big-endian uint32):**
- Unix timestamp offset by 2082844800 seconds (to account for Mac OS epoch: 1904-01-01)
- Mac OS epoch: January 1, 1904, 00:00:00 UTC
- Unix epoch: January 1, 1970, 00:00:00 UTC
- Offset: 2082844800 seconds between these epochs

**Standard FVER value for AIFF-C:**
- Timestamp: `0xA2805140` (2726318400 decimal)
- Represents: May 23, 1990, 14:40:00 UTC
- This is the official AIFF-C version timestamp

### 3.4 FVER Example

```
46 56 45 52           ; Chunk ID: "FVER"
00 00 00 04           ; Chunk size: 4 bytes
A2 80 51 40           ; Timestamp: 0xA2805140 (May 23, 1990)
```

---

## 4. MARK CHUNK — MARKER STRUCTURE

### 4.1 MARK Overview

The MARK chunk defines named positions (markers) within the audio data. Markers are used for:
- Loop points (start/end of loops)
- Region definitions (verse, chorus, etc.)
- Edit points
- Cues for CD track markers

### 4.2 MARK Byte Structure

```
Offset  Bytes   Field Name              Type        Description
------  ------  ------------------     ----------  ----------------------------------
0x00    4       Chunk ID                char[4]     "MARK"
0x04    4       Chunk Size              int32 BE    2 + (10 × numMarkers) + name lengths + pad
0x08    2       Number of Markers       uint16 BE   Count of markers (0 to 65535)
0x0A    n       Marker Array            varies      Array of marker structures
```

### 4.3 MARKER Structure (per marker)

```
Offset  Bytes   Field Name              Type        Description
------  ------  ------------------     ----------  ----------------------------------
0x00    2       Marker ID               uint16 BE   Unique identifier (1 to 65534)
0x02    4       Position                uint32 BE   Sample frame number (0-indexed)
0x06    2       Name Length             uint16 BE   Length of marker name in bytes
0x08    n       Marker Name             char[]      Pascal string (first byte = length)
```

**Pascal String:** The name is a Pascal string where the first byte contains the length, followed by the characters. Total size = 1 + nameLength bytes.

### 4.4 MARK Chunk Example

```
4D 41 52 4B           ; Chunk ID: "MARK"
00 00 00 2A           ; Chunk size: 42 bytes
00 03                 ; Number of markers: 3

; Marker 1: Start
00 01                 ; Marker ID: 1
00 00 00 00           ; Position: sample 0
06                    ; Name length: 6
00 53 74 61 72 74     ; Name: "\0Start" (null + "Start")

; Marker 2: Loop Start
00 02                 ; Marker ID: 2
00 00 1C 20           ; Position: sample 7200
0A                    ; Name length: 10
00 4C 6F 6F 70 20 53 74 61 72 74  ; Name: "\0Loop Start"

; Marker 3: End
00 03                 ; Marker ID: 3
00 00 3A 98           ; Position: sample 15000
04                    ; Name length: 4
00 45 6E 64           ; Name: "\0End"
```

### 4.5 MARK Chunk Restrictions

| Restriction | Value | Notes |
|-------------|-------|-------|
| Max markers | 65,535 | Limited by uint16 |
| Marker ID range | 0 to 65,534 | 0 is reserved |
| Position range | 0 to file samples | Must be within audio |
| Marker name length | Variable | Pascal string format |

### 4.6 Standard Marker IDs

| Marker ID | Meaning | Usage |
|-----------|---------|-------|
| 0 | Play entire file | Default, implicit |
| 1 | Default start | Play from beginning |
| 2+ | User-defined | Application-specific |

---

## 5. INST CHUNK — INSTRUMENT PARAMETERS

### 5.1 INST Overview

The INST chunk stores MIDI-based instrument parameters for sample-based instruments. It defines how a sound sample should be played back as an instrument.

### 5.2 INST Byte Structure

```
Offset  Bytes   Field Name              Type        Description
------  ------  ------------------     ----------  ----------------------------------
0x00    4       Chunk ID                char[4]     "INST"
0x04    4       Chunk Size              int32 BE    Always 20
0x08    1       MIDI Base Note           int8        Base MIDI note number (0-127)
0x09    1       Detune                  int8        Detune amount (-64 to +63 cents)
0x0A    1       Low Note                int8        Lowest playable note (0-127)
0x0B    1       High Note               int8        Highest playable note (0-127)
0x0C    1       Low Velocity            int8        Lowest velocity (0-127)
0x0D    1       High Velocity           int8        Highest velocity (0-127)
0x0E    2       Gain                    int16 BE    Gain in dB (0x0000 = 0dB)
0x10    4       Sustain Loop            Loop        Sustain loop structure
0x14    4       Release Loop            Loop        Release loop structure
```

**Total chunk size is always 20 bytes** (fixed for all AIFF files using INST).

### 5.3 LOOP Structure

```
Offset  Bytes   Field Name              Type        Description
------  ------  ------------------     ----------  ----------------------------------
0x00    2       Loop ID                 uint16 BE   Loop identifier
0x02    2       Loop Type               uint16 BE   Loop playback type
0x04    2       Begin Marker ID         uint16 BE   Marker ID for loop start
0x06    2       End Marker ID           uint16 BE   Marker ID for loop end
```

### 5.4 Loop Type Values

| Value | Type | Description |
|-------|------|-------------|
| 0 | No loop | Play sample once |
| 1 | Forward loop | Loop from begin to end, then restart |
| 2 | Forward-backward | Play forward then backward repeatedly |
| 3 | Release only | Loop during release phase only |

### 5.5 INST Chunk Fields Detail

**MIDI Base Note (1 byte):**
- Range: 0 to 127
- MIDI note number where sample is centered
- 60 = Middle C (C4)
- Example: A sample of a piano note at A4 (69) would set this to 69

**Detune (1 byte):**
- Range: -64 to +63
- Fine tuning in cents (1/100 of a semitone)
- Stored as signed integer
- Example: +10 means tune up by 10 cents

**Low Note / High Note (1 byte each):**
- Range: 0 to 127
- Define the note range where sample plays
- Sample will be pitch-shifted to play other notes

**Low Velocity / High Velocity (1 byte each):**
- Range: 0 to 127
- Define velocity (dynamics) range for the sample
- 0 = softest, 127 = loudest

**Gain (2 bytes, big-endian):**
- Signed 16-bit integer
- Value 0x0000 = 0 dB (no gain)
- Each unit = 1/256 dB
- Example: 0x0100 = +1 dB

### 5.6 INST Chunk Example

```
49 4E 53 54           ; Chunk ID: "INST"
00 00 00 14           ; Chunk size: 20 bytes
3C                    ; MIDI Base Note: 60 (Middle C)
00                    ; Detune: 0 cents
24                    ; Low Note: 36 (C2)
60                    ; High Note: 96 (C7)
00                    ; Low Velocity: 0
7F                    ; High Velocity: 127
00 00                 ; Gain: 0 dB

; Sustain Loop
00 01                 ; Loop ID: 1
00 01                 ; Loop Type: Forward
00 02                 ; Begin Marker ID: 2 (loop start)
00 03                 ; End Marker ID: 3 (loop end)

; Release Loop
00 00                 ; Loop ID: 0 (no release loop)
00 00                 ; Loop Type: No loop
00 00                 ; Begin Marker ID: 0
00 00                 ; End Marker ID: 0
```

### 5.7 INST and MARK Chunk Relationship

The INST chunk's loop structures reference marker IDs defined in the MARK chunk. This linkage connects loop points to named positions:

```
MARK chunk defines:
  Marker ID 2 -> "Loop Start" at sample position 1000
  Marker ID 3 -> "Loop End" at sample position 2000

INST chunk references:
  Sustain Loop Begin = 2 -> references "Loop Start"
  Sustain Loop End = 3 -> references "Loop End"
```

### 5.8 MIDI Note Number Reference

| Note | MIDI Number | Frequency (Hz) |
|------|-------------|---------------|
| C-1 | 0 | 8.1758 |
| A0 | 21 | 27.5 |
| C1 | 24 | 32.703 |
| C2 | 36 | 65.406 |
| C3 | 48 | 130.81 |
| C4 (Middle C) | 60 | 261.63 |
| A4 (Concert A) | 69 | 440.0 |
| C5 | 72 | 523.25 |
| C6 | 84 | 1046.5 |
| G10 | 127 | 12543.85 |

---

## 6. COMT CHUNK — COMMENTS

### 6.1 COMT Overview

The COMT chunk stores comments associated with the audio file. Each comment has a timestamp and can reference a marker for positioning.

### 6.2 COMT Byte Structure

```
Offset  Bytes   Field Name              Type        Description
------  ------  ------------------     ----------  ----------------------------------
0x00    4       Chunk ID                char[4]     "COMT"
0x04    4       Chunk Size              int32 BE    2 + comment_size × numComments
0x08    2       Number of Comments      uint16 BE   Count of comments
0x0A    n       Comment Array           varies      Array of comment structures
```

### 6.3 COMMENT Structure (per comment)

```
Offset  Bytes   Field Name              Type        Description
------  ------  ------------------     ----------  ----------------------------------
0x00    4       Timestamp               uint32 BE   Seconds since 1904-01-01 00:00:00 UTC
0x04    2       Marker ID               int16 BE    Associated marker (-1 if none)
0x06    2       Count                   uint16 BE   Number of characters
0x08    n       Text                    char[]      Comment text (not Pascal string)
```

**Note:** Unlike marker names, comment text is a **standard string** (not Pascal format), no length byte prefix.

### 6.4 COMT Chunk Example

```
43 4F 4D 54           ; Chunk ID: "COMT"
00 00 00 3C           ; Chunk size: 60 bytes
00 02                 ; Number of comments: 2

; Comment 1
A2 80 51 40           ; Timestamp: 0xA2805140 (May 23, 1990, 14:40:00)
FF FF                 ; Marker ID: -1 (no associated marker)
00 1C                 ; Count: 28 characters
52 65 63 6F 72 64 65 64 20 61 74 20 56 69 65 6E 6E 61 20 4B 6F 6E 7A 65 72 74 68 61 75 73  ; "Recorded at Vienna Konzert..."

; Comment 2
A2 80 51 41           ; Timestamp: next second
00 01                 ; Marker ID: 1 (associated with marker 1)
00 0F                 ; Count: 15 characters
43 68 6F 72 75 73 20 73 74 61 72 74 73 20 6E 6F 77  ; "Chorus starts now"
```

### 6.5 COMT Chunk Restrictions

| Restriction | Value | Notes |
|-------------|-------|-------|
| Max comments | 65,535 | Limited by uint16 |
| Timestamp format | Mac OS epoch | Seconds since 1904-01-01 |
| Marker reference | -1 or valid ID | -1 = no marker association |
| Text format | Plain string | Not Pascal, no null terminator |
| Character encoding | ASCII | Typically 7-bit ASCII |

### 6.6 COMT Timestamp Conversion

```python
# Mac OS to Unix timestamp
mac_epoch_offset = 2082844800  # seconds between 1904-01-01 and 1970-01-01
unix_timestamp = mac_timestamp - mac_epoch_offset

# Example
mac_timestamp = 0xA2805140  # 2726318400
unix_timestamp = 2726318400 - 2082844800 = 643473600
# Result: May 23, 1990, 14:40:00 UTC
```

---

## 7. NAME, AUTHOR, COPYRIGHT, ANNO CHUNKS

### 7.1 Overview

These four chunks provide basic text metadata about the audio file. They are simple text containers following a consistent format.

### 7.2 Common Structure (Name, Author, Copyright, Anno)

```
Offset  Bytes   Field Name              Type        Description
------  ------  ------------------     ----------  ----------------------------------
0x00    4       Chunk ID                char[4]     "NAME", "AUTH", "(c) ", "ANNO"
0x04    4       Chunk Size              int32 BE    Size of text data
0x08    n       Text Data               char[]      Pascal string format
```

**Note:** Text is stored as a Pascal string (first byte = length), same as marker names.

### 7.3 Chunk IDs

| Chunk ID | Content | Description |
|----------|---------|-------------|
| `NAME` | Title/Name | Name of the sound/document |
| `AUTH` | Author | Name of the author/creator |
| `(c) ` | Copyright | Copyright notice (note: 4 chars including spaces) |
| `ANNO` | Annotation | General annotation text |

### 7.4 Text Chunk Examples

**NAME chunk:**
```
4E 41 4D 45           ; Chunk ID: "NAME"
00 00 00 0C           ; Chunk size: 12 bytes
0B                    ; Name length: 11
53 79 6D 70 68 6F 6E 79 20 4E 6F 2E  ; "\0Symphony No. 5"
```

**AUTH chunk:**
```
41 55 54 48           ; Chunk ID: "AUTH"
00 00 00 0B           ; Chunk size: 11 bytes
0A                    ; Name length: 10
4C 75 64 77 69 67 20 76 61 6E  ; "\0Ludwig van"
```

**Copyright chunk:**
```
28 63 29 20           ; Chunk ID: "(c) " (copyright symbol, space)
00 00 00 1E           ; Chunk size: 30 bytes
1D                    ; Length: 29
C2 A9 20 32 30 32 34 20 43 6C 61 73 73 69 63 20 52 65 63 6F 72 64 73 20 4C 74 64 2E  ; "\0© 2024 Classic Records Ltd."
```

**ANNO chunk:**
```
41 4E 4E 4F           ; Chunk ID: "ANNO"
00 00 00 18           ; Chunk size: 24 bytes
17                    ; Length: 23
45 78 63 65 6C 6C 65 6E 74 20 63 6F 6E 63 65 72 74 20 72 65 63 6F 72 64 69 6E 67  ; "\0Excellent concert recording"
```

---

## 8. ID3V2 IN AIFF — EMBEDDED METADATA

### 8.1 ID3v2 in AIFF Overview

While not part of the original AIFF specification, it has become a common practice to embed ID3v2 metadata at the beginning of AIFF files (similar to how MP3 files use ID3v2).

### 8.2 ID3v2 Chunk Structure in AIFF

The ID3v2 chunk is placed at the very beginning of the file, before the FORM header:

```
Offset  Bytes   Field Name              Type        Description
------  ------  ------------------     ----------  ----------------------------------
0x00    3       ID3 Identifier          char[3]     "ID3"
0x03    2       Version                 uint8[2]    Version (e.g., 0x03 0x00 for v2.4.0)
0x05    1       Flags                   uint8       ID3v2 flags
0x06    4       Size                    uint32 BE   Synchsafe integer
0x0A    n       ID3v2 Data             varies      ID3v2 frames
```

### 8.3 ID3v2 Placement Considerations

| Position | Pros | Cons |
|----------|------|------|
| At file start | Standard, easy to find | Requires FORM to be after ID3 |
| Non-standard | Compatible with some players | May break others |

**The standard approach:** ID3v2 chunk should appear at the very start of the file, before the FORM chunk. The FORM chunk ID becomes data within the overall file structure.

### 8.4 AIFF File with ID3v2 Example

```
Offset   Content
-------  ------------------------------------
0x0000   ID3                             ; ID3v2 header
0x0003   03 00                           ; Version 2.4.0
0x0005   00                              ; Flags
0x0006   00 00 00 1F                     ; Size: 31 bytes
0x000A   TIT2...                         ; ID3v2 frames
0x0029   46 4F 52 4D                     ; FORM chunk (start of AIFF)
0x002D   XX XX XX XX                     ; FORM size
0x0031   41 49 46 46                     ; "AIFF"
...       ...                             ; Remaining AIFF chunks
```

### 8.5 APIC (Cover Art) in AIFF via ID3v2

Cover art can be embedded in AIFF files using the ID3v2 APIC frame:

```
Frame ID: "APIC"
Frame size: varies
Frame flags: 0x0000
Frame data:
  [0x00] Text encoding: 0x00 (ISO-8859-1)
  [0x01...] MIME type: "image/jpeg" or "image/png" (null-terminated)
  [0x0n] Picture type: 0x03 (Front cover) or 0x00-0x20
  [0x0n+1] Description (null-terminated)
  [0x0n+m] Image data (binary)
```

---

## 9. FFMPEG IMPLEMENTATION REFERENCE

### 9.1 FFmpeg AIFF Identifiers

```
Format Name (CLI):   aiff               # used with -f
AVFormat ID:         AVFORMAT_ID_AIFF   # in libavformat
Demuxer:             aiff.c              # in libavformat
Muxer:               aiff.c              # in libavformat
Codec Support:       All PCM formats, various compressed formats for AIFC
```

### 9.2 FFmpeg AIFF Metadata Reading

```bash
# Read all metadata
ffprobe -v quiet -print_format json -show_format input.aiff | jq .format.tags

# Read specific metadata
ffprobe -v quiet -show_format input.aiff | grep -E "^tag_"

# Read AIFF-specific chunks (using exiftool)
exiftool -a input.aiff

# Read MARK chunks
exiftool -MarkChunk input.aiff  # Shows markers if present

# Read INST chunk
exiftool -InstrumentChunk input.aiff

# Read COMT comments
exiftool -CommentsChunk input.aiff
```

### 9.3 FFmpeg AIFF Metadata Writing

```bash
# Write standard metadata
ffmpeg -i input.wav \
  -c:a pcm_s16be \
  -metadata title="Symphony No. 5" \
  -metadata artist="Beethoven" \
  -metadata album="Complete Symphonies" \
  -metadata date="2024" \
  -metadata genre="Classical" \
  -metadata comment="Concert recording" \
  output.aiff

# Write AIFF with ID3v2 (for cross-compatibility)
ffmpeg -i input.wav \
  -c:a pcm_s24be \
  -metadata title="Track Title" \
  -metadata author="Composer" \
  -metadata copyright="© 2024 Label" \
  output.aiff

# Strip all metadata
ffmpeg -i input.aiff -c copy -map_metadata -1 output.aiff
```

### 9.4 FFmpeg AIFF Encoding Options

```bash
# PCM AIFF (default)
ffmpeg -i input.wav -c:a pcm_s16be output.aiff

# 24-bit AIFF
ffmpeg -i input.wav -c:a pcm_s24be output.aiff

# 32-bit AIFF
ffmpeg -i input.wav -c:a pcm_s32be output.aiff

# AIFF-C (compressed)
ffmpeg -i input.wav -c:a alaw output.aifc

# Float AIFF
ffmpeg -i input.wav -c:a pcm_f32be output.aiff
```

### 9.5 AIFF Audio Format Options

| Parameter | CLI Option | Values | Notes |
|-----------|-----------|--------|-------|
| Sample format | `-c:a` | pcm_s16be, pcm_s24be, pcm_s32be, pcm_f32be | Big-endian |
| Sample rate | `-ar` | 8000-192000 | Hz |
| Channels | `-ac` | 1-64 | Channel count |
| Metadata | `-metadata` | key="value" | Standard tags |

---

## 10. TOOL REFERENCE

### 10.1 exiftool AIFF Support

```bash
# Read all AIFF metadata
exiftool input.aiff

# Read MARK chunks
exiftool -MarkChunk input.aiff

# Read INST chunk
exiftool -InstrumentChunk input.aiff

# Read COMT comments
exiftool -CommentsChunk input.aiff

# Read embedded ID3v2
exiftool -ID3v2 input.aiff

# Read cover art
exiftool -Picture input.aiff

# Write metadata
exiftool -Title="New Title" -Artist="New Artist" input.aiff

# Write comments
exiftool -Comment="New comment" input.aiff
```

### 10.2 kid3-cli AIFF Support

```bash
# Read all tags
kid3-cli -c "get" input.aiff

# Read specific tag
kid3-cli -c "get :title" input.aiff

# Write tag
kid3-cli -c "set title 'Track Title'" input.aiff

# Read AIFF-specific chunks
kid3-cli -c "get" input.aiff  # Shows MARK, INST, COMT if present
```

### 10.3 Limitations of AIFF Metadata

| Limitation | Description | Workaround |
|------------|-------------|------------|
| No standard cover art | AIFF doesn't define cover art | Use ID3v2 or convert to MP4 |
| Pascal strings | Text uses Pascal format | Tool handles conversion |
| Limited genre | No standard genre field | Use COMT or ID3v2 |
| No track/disc numbers | Not part of spec | Use ID3v2 or external |
| Big-endian | Different from WAV | Tool handles byte order |

---

## 11. BYTE-LEVEL STRUCTURE EXAMPLES

### 11.1 Minimal AIFF File Structure

```
Offset   Hex                                         ASCII        Description
-------  ------------------------------------------  ----------   ------------------------
0x0000   46 4F 52 4D                                 FORM         Chunk ID
0x0004   XX XX XX XX                                 ...          Chunk size
0x0008   41 49 46 46                                 AIFF         Form type
0x000C   43 4F 4D 4D                                 COMM         Common chunk
0x0010   XX XX XX XX                                 ...          Chunk size
0x0014   02 00                                       2            Channels
0x0016   XX XX XX XX XX XX XX XX XX XX               ...          Sample rate (80-bit)
0x0020   10 00                                       16           Frames (sample count high)
0x0022   XX XX                                       ...          Sample count low
0x0024   10 00                                       16           Bits per sample
0x0026   53 53 4E 44                                 SSND         Sound data chunk
0x002A   XX XX XX XX                                 ...          Chunk size
0x002E   00 00 00 00                                 0            Offset
0x0032   00 00 00 00                                 0            Block size
0x0036   ...                                         ...          Audio samples
```

### 11.2 AIFF with MARK Chunk Example

```
; FORM header
46 4F 52 4D           ; "FORM"
XX XX XX XX           ; Total size
41 49 46 46           ; "AIFF"

; MARK chunk
4D 41 52 4B           ; "MARK"
00 00 00 16           ; Size: 22 bytes
00 02                 ; 2 markers

; Marker 1
00 01                 ; ID: 1
00 00 00 00           ; Position: 0
05                    ; Name length: 5
00 53 74 61 72        ; "\0Star" (Start)

; Marker 2
00 02                 ; ID: 2
00 00 1C 20           ; Position: 7200
0B                    ; Name length: 11
00 4C 6F 6F 70 20 45 6E 64  ; "\0Loop End"

; COMM chunk (abbreviated)
43 4F 4D 4D           ; "COMM"
...                   ; Common chunk data

; SSND chunk (abbreviated)
53 53 4E 44           ; "SSND"
...                   ; Sound data
```

### 11.3 AIFF with INST Chunk Example

```
; INST chunk
49 4E 53 54           ; "INST"
00 00 00 14           ; Size: 20 bytes (always)
3C                    ; Base note: 60 (Middle C)
00                    ; Detune: 0
24                    ; Low note: 36 (C2)
60                    ; High note: 96 (C7)
00                    ; Low velocity: 0
7F                    ; High velocity: 127
00 00                 ; Gain: 0 dB

; Sustain loop
00 01                 ; ID: 1
00 01                 ; Type: forward loop
00 02                 ; Begin marker: 2
00 03                 ; End marker: 3

; Release loop
00 00                 ; ID: 0 (no release)
00 00                 ; Type: no loop
00 00                 ; Begin: 0
00 00                 ; End: 0
```

---

## 12. METADATA CHUNK PRIORITY & INTERACTION

### 12.1 AIFF Metadata Priority

When multiple metadata systems are present:

| Priority | System | Notes |
|----------|--------|-------|
| 1 | INST | Highest priority for instrument playback |
| 2 | MARK | Defines playback regions and loops |
| 3 | COMT | Comments for documentation |
| 4 | NAME/AUTH/ANNO | Basic metadata |
| 5 | ID3v2 | Cross-format compatibility layer |

### 12.2 Conflict Resolution

| Scenario | Resolution |
|----------|------------|
| INST + MARK conflict | INST references MARK IDs; ensure MARK IDs exist |
| COMT marker reference missing | Ignore reference, keep comment |
| ID3v2 + NAME conflict | ID3v2 takes precedence (more widely supported) |
| Multiple COMT entries | Preserve all, ordered by timestamp |

### 12.3 Cross-Format Metadata Mapping

| AIFF Chunk | Equivalent in WAV | Equivalent in MP3 |
|------------|-------------------|-------------------|
| NAME | INAM (title) | TIT2 (title) |
| AUTH | IART (artist) | TPE1 (artist) |
| (c)  | ICOP (copyright) | TCOP (copyright) |
| ANNO | ICMT (comment) | COMM (comment) |
| COMT | ICMT (comment) | COMM (comment) |
| MARK | cue 'cue ' (cue points) | -- |
| INST | inst (instrument) | -- |

---

## 13. PRACTICAL GUIDELINES

### 13.1 When to Use MARK Chunks

| Use Case | Recommendation |
|----------|---------------|
| CD track markers | Use MARK with standard IDs (1, 2, 3...) |
| Loop points | Use MARK + INST sustain loop |
| Region definitions | Use MARK for region boundaries |
| Edit points | Use MARK for cut/join points |
| Cue points | Use MARK for DJ cue points |

### 13.2 When to Use INST Chunks

| Use Case | Recommendation |
|----------|---------------|
| Sample-based instruments | Always use INST + MARK |
| Loop-enabled samples | Use INST + MARK with loop IDs |
| One-shot samples | No INST needed |
| Multi-sample instruments | Multiple INST chunks (one per sample) |

### 13.3 Recommended AIFF Metadata Workflow

```
1. Create AIFF with COMM + SSND chunks
2. Add MARK chunks for loop points (if applicable)
3. Add INST chunk referencing MARK IDs (if applicable)
4. Add NAME/AUTH/COPYRIGHT/ANNO chunks
5. Add COMT for comments (if needed)
6. Optionally add ID3v2 for cross-format compatibility
```

---

## 14. REFERENCE IMPLEMENTATIONS

### 14.1 Reference Specifications

| Document | Description | URL |
|----------|-------------|-----|
| AIFF 1.3 Spec | Apple AIFF specification | https://www.mmsp.ece.mcgill.ca/Documents/AudioFormats/AIFF/Docs/AIFF-1.3.pdf |
| AIFF-C Spec | AIFF for C (compressed) | https://www.mmsp.ece.mcgill.ca/Documents/AudioFormats/AIFF/AIFF.html |
| MIDI Spec | MIDI note mapping reference | https://www.midi.org/specifications |

### 14.2 Implementation Libraries

| Library | Language | AIFF Support | Notes |
|---------|----------|--------------|-------|
| libsndfile | C | Full | Comprehensive AIFF support |
| FFmpeg | C | Full | AIFF demuxing/muxing |
| SoX | C | Full | Audio processing |
| audioread | Python | Full | Via FFmpeg backend |

### 14.3 AIFF Validation Tools

| Tool | Description | AIFF Validation |
|------|-------------|-----------------|
| JHOVE | Format validator | Full AIFF-hul module |
| MediaInfo | Format analyzer | Basic metadata |
| exiftool | Metadata tool | Full metadata |
| ffprobe | FFmpeg analyzer | Full |

---

## 15. IMPLEMENTATION CHECKLIST

### 15.1 Reading AIFF Metadata Chunks

- [ ] Parse FORM container (verify "FORM" at offset 0)
- [ ] Verify form type ("AIFF" or "AIFC")
- [ ] For AIFC: parse FVER chunk first
- [ ] Parse required COMM chunk
- [ ] Locate optional MARK chunk (may be before or after SSND)
- [ ] Parse marker structures correctly (Pascal strings)
- [ ] Locate optional INST chunk
- [ ] Parse loop structures and validate marker references
- [ ] Locate optional COMT chunk
- [ ] Parse comment structures (note: not Pascal strings)
- [ ] Parse optional NAME, AUTH, (c) , ANNO chunks
- [ ] Check for ID3v2 at file start
- [ ] Handle pad bytes correctly

### 15.2 Writing AIFF Metadata Chunks

- [ ] Create/form validate chunk structure
- [ ] Write MARK with correct marker IDs
- [ ] Validate marker positions are within audio range
- [ ] Write INST with loop IDs matching MARK IDs
- [ ] Encode text as Pascal strings for NAME/AUTH/ANNO
- [ ] Encode text as plain strings for COMT
- [ ] Add pad bytes for odd-length chunks
- [ ] Update FORM chunk size correctly
- [ ] Use big-endian byte order for all multi-byte values

### 15.3 INST/MARK Integration

- [ ] Create MARK chunks before INST
- [ ] Assign unique marker IDs (avoid duplicates)
- [ ] Set INST loop Begin/End to valid MARK IDs
- [ ] Validate loop positions don't exceed audio length
- [ ] Set loop type correctly (forward, release, etc.)
- [ ] Handle case where loop markers are deleted

### 15.4 AIFF-C Specific

- [ ] Include FVER chunk with correct timestamp
- [ ] Set compression type in COMM chunk
- [ ] Provide compression name in COMM chunk
- [ ] Handle compressed audio data correctly

---

## 16. METADATA COMPATIBILITY MATRIX

### 16.1 AIFF Metadata Support by Tool

| Tag/Chunks | FFmpeg | exiftool | kid3 | JHOVE |
|------------|--------|----------|------|-------|
| COMM | Yes | Yes | Yes | Yes |
| NAME | Yes | Yes | Yes | Yes |
| AUTH | Yes | Yes | Yes | Yes |
| (c)  | Yes | Yes | Yes | Yes |
| ANNO | Yes | Yes | Yes | Yes |
| MARK | No | Partial | No | Yes |
| INST | No | Partial | No | Yes |
| COMT | No | Yes | No | Yes |
| ID3v2 | Yes | Yes | Yes | Yes |
| Cover Art | Via ID3v2 | Yes | No | Yes |

### 16.2 AIFF Metadata Fields Mapping

| AIFF Chunk | FFmpeg Key | exiftool Tag | Notes |
|------------|------------|-------------|-------|
| NAME | title | Title | Track title |
| AUTH | artist | Author | Artist/composer |
| (c)  | copyright | Copyright | Copyright notice |
| ANNO | comment | Comment | Annotation |
| COMM | -- | -- | Audio format info |
| MARK | -- | MarkChunk | Markers |
| INST | -- | InstrumentChunk | Instrument |
| COMT | -- | Comment | Comments |

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
