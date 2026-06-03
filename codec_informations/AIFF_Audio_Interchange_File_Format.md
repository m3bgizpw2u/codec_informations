# AIFF / AIFC (Audio Interchange File Format) — Deep Technical Reference

> **Category:** Audio Container / Interchange Format
> **File Extensions:** .aiff, .aif, .aifc
> **MIME Types:** audio/aiff, audio/x-aiff, audio/aiffc
> **Standardization Body:** Apple Computer, Inc. (published as EA IFF 85 standard)
> **Specification Document:** Apple AIFF Specification, AIFC extension (1991)
> **Patent Status:** Patent-free, open specification
> **License:** Public domain / Apple contributed to public domain

---

## 1. HISTORICAL CONTEXT & ORIGIN

The Audio Interchange File Format (AIFF) was developed by Apple Computer, Inc. in the late 1980s, with the goal of creating a standard file format for exchanging audio data between different computer platforms and audio applications. The original AIFF specification was published as part of the Electronic Arts Interchange File Format (EA IFF 85) standard, a generalized container format designed to accommodate many different data types — audio, video, text, and binary data — under a single structural framework.

Apple published the AIFF specification publicly, and it rapidly became the de facto standard for professional audio interchange on classic Mac OS and later NeXT, SGI, and other UNIX workstations. Its adoption in professional audio stems from several deliberate design decisions: big-endian byte order (matching the Motorola 68000 architecture of early Mac and SGI workstations), a clean chunk-based structure that allowed new metadata types to be added without breaking parsers, and support for high-quality audio at a time when most platforms were limited to 8-bit mono.

The AIFF format underwent a significant extension in 1991 with the introduction of AIFF-C (AIFF with Compression), which added support for compressed audio data while maintaining the same overall container structure. This extension allowed AIFF to serve not only as a high-quality interchange format for uncompressed PCM but also as a carrier format for common audio codecs such as G.711 μ-law and A-law, IMA ADPCM variants, and Apple proprietary codecs.

Over time, AIFF was largely supplanted by more efficient container formats — MP4/AAC for distribution, FLAC for lossless archival, and CAF (Core Audio Format) for macOS professional audio — but it remains important in conversion pipelines because it serves as a universal lossless intermediate between platforms and applications. Virtually every audio tool, operating system, and library can read and write AIFF, making it the most portable of the traditional high-quality audio formats. FFmpeg, SoX, Audacity, Adobe Audition, Pro Tools, Logic Pro, and nearly every other audio application supports AIFF as both input and output.

The format's historical significance extends beyond mere interchange. Many of its design choices — chunked architecture, big-endian numeric encoding, support for 8-bit unsigned samples, marker-based timestamps, instrument chunks — reflect the constraints and requirements of 1980s-era professional audio workstations. Understanding AIFF in depth illuminates the foundations on which modern audio container design was built.

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

AIFF is fundamentally a chunked container format built on the EA IFF 85 framework. Every piece of information in an AIFF file, whether it is audio sample data, format metadata, markers, or application-specific annotations, is stored as a discrete chunk. Each chunk has a type identifier, a size field, and a payload. The container itself is identified by a top-level FORM header that declares the overall form type (AIFF or AIFC), which determines the interpretation of contained chunks.

The architecture distinguishes clearly between AIFF (uncompressed PCM only) and AIFC (AIFF with Compression), which can carry encoded audio data. The presence or absence of a FVER chunk is the primary discriminator: an AIFF file never contains FVER, while an AIFC file always does. The FORM type identifier further clarifies this: `FORM` followed by `AIFF` or `FORM` followed by `AIFC`.

All numeric values in AIFF and AIFC are stored in big-endian (also known as network byte order or Motorola byte order) byte orientation. This applies to all chunk sizes, sample counts, sample rates, bit depths, channel counts, audio sample data, marker positions, instrument parameters, and any other numeric fields. This is in direct contrast to the RIFF/WAV format used on Intel-based platforms, which uses little-endian byte order. This difference is the single most important consideration when implementing cross-platform AIFF reading or writing code.

AIFF supports both integer PCM sample formats and a limited set of compressed formats in AIFC. The native integer PCM formats include 8-bit unsigned (0 to 255, with 128 as silence) and any bit depth from 1 to 32 bits per sample, stored as signed two's complement integers. Multi-channel audio is supported, with channels stored interleaved (L0, R0, L1, R1, ...) in the SSND chunk data. There is no defined maximum channel count or sample rate within the AIFF specification itself, though practical implementations may impose limits.

The chunk ordering rules in AIFF are not strictly enforced by all readers, but the canonical ordering — established by the original Apple specification — places the FORM header first, followed by the required chunks in a specific sequence, followed by optional chunks in any order. COMM (format chunk) must precede SSND (sound data chunk), and FVER (format version chunk, required for AIFC) must appear before any other chunk. This ordering enables streaming readers to discover the audio format before encountering the potentially large audio data.

AIFF does not define a built-in metadata system. Metadata is carried through several dedicated chunk types: NAME for a title, AUTHOR for an author/artist attribution, COPYRIGHT for a rights notice, ANNO for general annotations, MARK for named timestamp positions in the audio, INST for MIDI-style instrument parameters, and COMMENT for application-specific text. There is no standardized tag naming convention comparable to ID3 in MP3 or Vorbis Comments in Ogg. This makes AIFF metadata handling application-specific and often inconsistent across tools.

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File / Stream Structure

An AIFF or AIFC file is organized as a sequence of chunks, all contained within a top-level FORM container. The FORM container itself follows the standard IFF chunk layout and serves as the root of the file hierarchy.

```
┌──────────────────────────────────────────────────────────────────────┐
│                          FORM CONTAINER                               │
│  Offset  Bytes  Value              Description                        │
│  ─────   ─────  ─────              ───────────                        │
│  0       4      "FORM" (0x464F524D)  IFF type identifier              │
│  4       4      uint32              Size of remaining file (big-endian)│
│  8       4      "AIFF" or "AIFC"    Form type (big-endian ASCII)      │
│  ─────   ─────  ─────              ──────────────────────────────────  │
│  12+     ...    chunk               First chunk (COMM required)        │
│          ...    chunk               Additional chunks                  │
└──────────────────────────────────────────────────────────────────────┘
```

The FORM header occupies 12 bytes total: 4 bytes for the "FORM" magic, 4 bytes for the size of everything after the size field (i.e., the FORM header's data portion), and 4 bytes for the form type identifier. Note that the size field in the FORM header includes the 4-byte form type plus all subsequent chunks, but does NOT include the 8 bytes consumed by the FORM chunk's own ID and size fields. This is consistent with the general IFF chunk layout.

The FORM size field is a uint32 big-endian integer. In practice, many AIFF files exceed 4 GB, and implementations use various strategies to handle this: either storing a size of 0xFFFFFFFF (indicating unknown size, as commonly done in IFF files) or relying on the fact that most tools truncate the FORM size field to this maximum value. True 64-bit extended size support was never standardized in the original AIFF specification, though AIFC references the 64-bit extended integer format for the sound data chunk size in some documentation.

### 3.2 IFF Chunk Layout (Universal)

Every chunk in an AIFF file follows the identical 8-byte header followed by a payload:

```
┌──────────────────────────────────────────────────────────────────────┐
│                         IFF CHUNK LAYOUT                              │
│  Offset  Bytes  Type              Description                         │
│  ─────   ─────  ───              ───────────                         │
│  0       4      char[4]           Chunk ID (4 ASCII characters)       │
│  4       4      uint32            Chunk size (big-endian)             │
│  8       N      uint8[]           Chunk data (N = chunk size)         │
└──────────────────────────────────────────────────────────────────────┘
```

The chunk size field specifies the exact number of bytes in the chunk's data portion, NOT including the 8-byte header. After the data portion, if the chunk size is odd, a single padding byte (0x00) is appended to maintain alignment to even byte boundaries. This padding byte is not included in the chunk size count. All AIFF chunks are therefore aligned to 2-byte (even) boundaries.

The 4-byte chunk ID is a case-sensitive ASCII identifier. Unlike some IFF variants, AIFF uses upper-case identifiers exclusively: `COMM`, `SSND`, `FVER`, `MARK`, `INST`, `NAME`, `AUTHOR`, `COPYRIGHT`, `ANNO`, `COMMENT`. Any chunk ID that is not recognized by a parser should be skipped by reading the declared size, adding padding if odd, and advancing to the next chunk. This forward-compatibility mechanism is central to the IFF design philosophy.

### 3.3 Required Chunks

**AIFF (uncompressed) requires exactly two chunks:**
- `COMM` — the Common chunk, which describes the audio format (sample rate, bit depth, channel count, sample count).
- `SSND` — the Sound Data chunk, which contains the actual audio sample data.

**AIFC (compressed) requires three chunks:**
- `COMM` — same as AIFF, with the addition of a compression type and name field.
- `SSND` — the compressed audio data.
- `FVER` — the Format Version chunk, which specifies the AIFC revision number.

In both cases, the `FORM` header surrounds everything. The absence of `FVER` definitively identifies an AIFF file; its presence identifies an AIFC file.

---

## 4. CHUNK TYPE SPECIFICATIONS

### 4.1 FORM Chunk

The FORM chunk is the top-level container for the entire file. It is identical in structure to any other IFF chunk, with the ID "FORM" and a form type that specifies the semantic interpretation of the enclosed chunks.

| Field         | Offset | Size | Type       | Value                |
|---------------|--------|------|------------|----------------------|
| Chunk ID      | 0      | 4    | char[4]    | "FORM" (0x464F524D)  |
| Chunk Size    | 4      | 4    | uint32 BE  | Size of form data    |
| Form Type     | 8      | 4    | char[4]    | "AIFF" or "AIFC"     |

The Form Type `"AIFF"` (0x41494646) indicates an AIFF file containing uncompressed PCM audio. The Form Type `"AIFC"` (0x41494643) indicates an AIFF-C file that may contain compressed audio. No other form types are standard, though some tools produce non-standard variants.

The Form Size field, like all IFF chunk sizes, does not include the 8-byte header. It equals 4 (for the form type) plus the sum of all chunk sizes (plus padding bytes for odd-sized chunks).

### 4.2 COMM Chunk (Common)

The Common chunk is the primary format descriptor. Every valid AIFF and AIFC file must contain exactly one COMM chunk, and it must appear before the SSND chunk.

**AIFF COMM Layout (26 bytes):**

```
┌──────────────────────────────────────────────────────────────────────┐
│  Offset  Bytes  Type       Description                               │
│  ─────   ─────  ────       ───────────                               │
│  0       2      int16 BE   Number of audio channels (nchannels)     │
│  2       4      int32 BE   Number of sample frames (numSampleFrames) │
│  6       2      int16 BE   Bits per sample (sampleSize)             │
│  8       10     uint80 BE   Sample rate (Extended float, see §4.2.1)│
└──────────────────────────────────────────────────────────────────────┘
Total fixed fields: 18 bytes
```

**AIFC COMM Layout (additional fields after the AIFF fields):**

```
┌──────────────────────────────────────────────────────────────────────┐
│  Offset  Bytes  Type       Description                               │
│  ─────   ─────  ────       ───────────                               │
│  18      4      char[4]    Compression type identifier               │
│  22      1      int8       Compression name length (strlen + 1)      │
│  23      N      char[]     Compression name (N = name length)         │
│                                (padded to even boundary)            │
└──────────────────────────────────────────────────────────────────────┘
```

**nchannels** (channels): A 16-bit big-endian signed integer specifying the number of audio channels. Value 1 = mono, 2 = stereo, 6 = 5.1 surround, etc. There is no defined maximum in the specification; practical limits are imposed by the file size (sample frames × channels × bit depth must fit in the SSND chunk) and the application consuming the file.

**numSampleFrames**: A 32-bit big-endian signed integer specifying the total number of sample frames in the file. A sample frame is defined as one sample from each channel at a given point in time. For stereo audio at 44100 Hz, 16-bit, with 10 seconds of audio, this would be 44100 × 10 = 441,000 sample frames. The total byte count of audio data in the SSND chunk is `numSampleFrames × nchannels × ((sampleSize + 7) / 8)`, rounded up to the nearest even byte boundary.

**sampleSize**: A 16-bit big-endian signed integer specifying the number of bits used to represent each sample. For standard PCM, this is typically 8, 16, 20, 24, or 32. A value of 8 represents unsigned 8-bit PCM where the silence point is at 128. Values greater than 8 represent signed two's complement integers. For compressed AIFC formats, this field indicates the native bit depth of the decompressed output.

#### 4.2.1 Extended Precision 80-bit IEEE 754 Float (Sample Rate)

The sample rate is stored as an 80-bit (10-byte) extended precision IEEE 754 floating-point number, also known as "extended" or "long double" format on x87 floating-point units. This is a critical and unusual choice that differentiates AIFF from virtually every other audio format.

**80-bit Extended Format Layout (big-endian):**

```
┌──────────────────────────────────────────────────────────────────────┐
│  Byte 0    Byte 1    Byte 2    ...  Byte 8    Byte 9                │
│  ┌────────┬─────────────────────────────────────────────────────────┐ │
│  │ Sign+  │  Exponent (15 bits)  │  Integer bit │  Fraction (63 bits)│ │
│  │Exponent │                      │  (1 bit)     │                   │ │
│  │ (1 bit)│                      │              │                   │ │
│  └────────┴─────────────────────────────────────────────────────────┘ │
│  Total: 1 + 15 + 1 + 63 = 80 bits                                     │
└──────────────────────────────────────────────────────────────────────┘
```

The 10-byte layout, from most significant byte to least significant byte:

| Byte Position | Bits         | Field                  | Description                                      |
|---------------|--------------|------------------------|--------------------------------------------------|
| 0             | Bits 0–7     | Exponent MSB           | Upper 8 bits of 15-bit exponent (biased by 16383)|
| 1             | Bits 0–7     | Exponent LSB + Sign    | Lower 7 bits of exponent + sign bit (bit 7)     |
| 2–9           | Bits 0–7     | Mantissa               | 8 bytes of mantissa (integer bit + 63 fraction) |

The sign and exponent fields occupy the first 2 bytes. The exponent is 15 bits wide, biased by 16383. The sign bit is bit 7 of byte 1. The remaining 8 bytes (bytes 2–9) hold the mantissa, where the most significant bit (the integer bit, also called the "j-bit") is explicit (unlike in 64-bit double where it is implicit). This explicit integer bit means the extended format has 64 bits of mantissa precision (1 explicit + 63 fraction), effectively giving approximately 19 decimal digits of precision.

**Common Sample Rates encoded as 80-bit extended:**

| Sample Rate | Hexadecimal Representation (10 bytes, big-endian) |
|-------------|---------------------------------------------------|
| 8000 Hz     | 40E2 0000 0000 0000 0000                           |
| 11025 Hz    | 40AC 2C00 0000 0000 0000                           |
| 22050 Hz    | 40D6 5800 0000 0000 0000                           |
| 44100 Hz    | 40E5 8C00 0000 0000 0000                           |
| 48000 Hz    | 40E8 AC00 0000 0000 0000                           |
| 88200 Hz    | 40FA 1800 0000 0000 0000                           |
| 96000 Hz    | 40FB 5800 0000 0000 0000                           |
| 176400 Hz   | 40FD 8C00 0000 0000 0000                           |
| 192000 Hz   | 40FE AC00 0000 0000 0000                           |

To extract the sample rate as a double for computation, extract the biased exponent, subtract 16383 to get the true exponent, and compute `2^exponent × (1 + fraction / 2^63)`. Most programming environments provide a way to reinterpret 10 bytes as an extended precision float or to manually decode the value.

**Implementation Note:** When converting from AIFF to other formats, extract the sample rate by reading the 10-byte extended float. Do not assume standard IEEE 754 double conversion will work — the extended format's byte order and layout differ from both standard double (64-bit) and single (32-bit). Some implementations incorrectly attempt to read the 80-bit value as a 64-bit double, producing garbage sample rate values. The correct approach is to manually decode the 80-bit representation or use a library that provides native 80-bit extended float support.

### 4.3 SSND Chunk (Sound Data)

The Sound Data chunk contains the actual audio sample data. It must appear after the COMM chunk in a valid AIFF/AIFC file.

**SSND Chunk Layout:**

```
┌──────────────────────────────────────────────────────────────────────┐
│  Offset  Bytes  Type       Description                               │
│  ─────   ─────  ────       ───────────                               │
│  0       4      uint32 BE  Offset to sound data (blockSize)           │
│  4       4      uint32 BE  Block size (blockAlign)                    │
│  8       N      uint8[]    Sound data (N = chunk size - 8)           │
└──────────────────────────────────────────────────────────────────────┘
```

**blockSize (offset):** This field, confusingly named "offset" in the specification, specifies the number of bytes preceding the sound data that should be skipped when reading. In practice, virtually all AIFF files set this to 0. Some very old applications set it to a non-zero value for alignment purposes, but this is extremely rare. If non-zero, the reader should skip `blockSize` bytes of offset data before beginning to read sample data. The offset data is not included in the chunk's data size in the standard formula for computing audio duration.

**blockAlign (block size):** This field specifies the number of bytes per "block" of audio data, where a block is defined as one sample frame (all channels at one point in time). For PCM audio with `sampleSize` bits per sample, `blockAlign = nchannels × ((sampleSize + 7) / 8)`, rounded up to the nearest byte. For example, for 16-bit stereo: `blockAlign = 2 × 2 = 4 bytes`. For 24-bit mono: `blockAlign = 1 × 3 = 3 bytes`. The blockAlign field exists to support block-structured compression formats where multiple sample frames are encoded as a unit, but for linear PCM, it is simply the frame size. When reading PCM audio, the blockAlign value can be used to verify consistency with the COMM chunk parameters.

**Sound Data:** The actual audio samples, stored as a continuous interleaved stream. For AIFF (PCM), each sample is a signed two's complement integer of `sampleSize` bits, padded to the nearest byte. For 8-bit PCM, samples are unsigned with 128 as silence. For all other bit depths, samples are signed. Channels are interleaved: all channels for sample frame 0, then all channels for sample frame 1, etc.

**Byte packing for non-8-bit samples:** For sample sizes that are not a multiple of 8 (e.g., 12-bit, 20-bit), samples are packed into bytes with the most significant bit of the first byte being the most significant bit of the sample. A 20-bit sample occupies 3 bytes (bits 19–0), with bit 19 at the MSB of byte 0 and bit 0 at the LSB of byte 2. Unused padding bits within a byte are set to zero and are ignored on read.

**Multi-channel ordering:** AIFF does not define a canonical channel ordering. For stereo, the convention is channel 0 = left, channel 1 = right, but this is not enforced by the specification. For multi-channel audio (5.1, 7.1), implementations vary widely. When converting to formats with defined channel layouts (such as WAV's speaker positions or FFmpeg's channel layout conventions), care must be taken to map channels correctly, or the user should be warned about ambiguous channel ordering.

### 4.4 FVER Chunk (Format Version)

The Format Version chunk is required for all AIFC files and must not appear in AIFF files. It specifies the AIFC specification revision that the file conforms to.

**FVER Chunk Layout (4 bytes data):**

```
┌──────────────────────────────────────────────────────────────────────┐
│  Offset  Bytes  Type       Description                               │
│  ─────   ─────  ────       ───────────                               │
│  0       4      uint32 BE  AIFC version timestamp                    │
└──────────────────────────────────────────────────────────────────────┘
```

The version is stored as a decimal number representing a date/time in the format YYYYMMDD. For example, the value `0x0002A461` (big-endian uint32) equals 173,857 decimal, which as a date corresponds to November 24, 1991 — the date of the AIFC 1.0 specification. The specification defines only one version value: `173857` (0x0002A861 in the hex representation shown in the original docs — note the specification document sometimes uses confusing byte order examples). The canonical decimal value is 173857.

FFmpeg and most modern tools write `173857` (0x0002A861) as the FVER timestamp. Any other value should be treated as non-compliant or as a future version.

### 4.5 MARK Chunk (Marker)

The Marker chunk stores named positions (timestamps) within the audio stream. Markers are stored as a list, each with a unique 16-bit ID, a name string, and a sample frame position.

**MARK Chunk Layout:**

```
┌──────────────────────────────────────────────────────────────────────┐
│  Offset  Bytes  Type       Description                               │
│  ─────   ─────  ────       ───────────                               │
│  0       2      int16 BE   Number of markers (markerCount)           │
│  4       N      ...        Marker entries (see below)               │
└──────────────────────────────────────────────────────────────────────┘
```

Each marker entry has the following layout:

```
┌──────────────────────────────────────────────────────────────────────┐
│  Offset  Bytes  Type       Description                               │
│  ─────   ─────  ────       ───────────                               │
│  0       2      int16 BE   Marker ID (unique, used by INST chunk)    │
│  2       4      int32 BE   Position in sample frames                │
│  6       2      int16 BE   Marker name length (strlen + 1)           │
│  8       N      char[]     Marker name (N = name length, padded)    │
└──────────────────────────────────────────────────────────────────────┘
```

Marker IDs are signed 16-bit integers, ranging from -32768 to 32767. They need not be sequential or consecutive. The value 0 is reserved and should not be used as a marker ID. Marker IDs are referenced by the INST chunk to indicate attack and release positions.

Marker positions are expressed as sample frame numbers (zero-based indexing from the start of the audio data). The position value must not exceed `numSampleFrames` from the COMM chunk. Markers positioned at the very start of the audio have position 0; markers at the very end have position `numSampleFrames - 1`.

The marker name is a Pascal-style string (length byte followed by characters), padded to an even byte boundary. For example, a marker named "Verse" would be stored as: length byte (0x05), 'V' (0x56), 'e' (0x65), 'r' (0x72), 's' (0x73), 'e' (0x65), plus one padding byte (0x00) to reach the even boundary.

### 4.6 INST Chunk (Instrument)

The Instrument chunk stores parameters describing how the audio might be played as a MIDI-compatible instrument. This is an optional metadata chunk that provides information such as MIDI note numbers, velocity ranges, and ADSR envelope parameters.

**INST Chunk Layout (20 bytes):**

```
┌──────────────────────────────────────────────────────────────────────┐
│  Offset  Bytes  Type       Description                               │
│  ─────   ─────  ────       ───────────                               │
│  0       1      int8        Base frequency (semitones relative to A) │
│  1       1      int8        Detune (-50 to +50 cents)                │
│  2       1      int8        Volume scale (0–127)                     │
│  3       2      int16 BE    Sustain loop play mode (see below)       │
│  5       2      int16 BE    Sustain loop begin marker ID             │
│  7       2      int16 BE    Sustain loop end marker ID               │
│  9       2      int16 BE    Release loop play mode                   │
│  11      2      int16 BE    Release loop begin marker ID             │
│  13      2      int16 BE    Release loop end marker ID               │
└──────────────────────────────────────────────────────────────────────┘
```

**baseFrequency:** Signed byte indicating the MIDI note number of the base frequency of the sound, relative to the A above middle C (MIDI note 69, 440 Hz). The value is stored as -127 to +127, but the spec defines it as the number of semitones above or below middle A (A-49 in the original spec's numbering system). Converting to a MIDI note: `midiNote = 69 + baseFrequency`. A value of 0 means the audio is pitched at the same frequency as middle A (440 Hz).

**detune:** Signed byte indicating fine tuning from -50 to +50 cents (hundredths of a semitone). The value 0 means no detuning. Positive values detune upward; negative values detune downward.

**volumeScale:** Unsigned byte (0–127) indicating a scaling factor to apply to the overall gain when playing the instrument.

**Loop play modes:** Four loop modes are defined:
- 0 = No looping
- 1 = Forward looping (play forward, then jump back to start)
- 2 = Forward/backward looping (ping-pong)
- 3 = Backward looping

The sustain loop is the loop that plays while a key is held (after the attack and decay phases). The release loop plays after the key is released. Both loops reference marker IDs from the MARK chunk. A marker ID of 0 means the loop point is not specified.

### 4.7 NAME Chunk

A simple text chunk containing a name or title for the audio content. Follows standard IFF text chunk format: a Pascal-style string (byte count followed by characters), padded to an even byte boundary.

```
┌──────────────────────────────────────────────────────────────────────┐
│  Offset  Bytes  Type       Description                               │
│  ─────   ─────  ────       ───────────                               │
│  0       2      int16 BE   Text length (strlen)                      │
│  2       N      char[]     Text data (N = length, padded to even)   │
└──────────────────────────────────────────────────────────────────────┘
```

The NAME chunk stores the audio's title, analogous to the Title field in other formats. Many applications embed it but others ignore it.

### 4.8 AUTHOR Chunk

Similar to NAME, but specifically for the author or artist attribution. Standard Pascal-style string layout.

### 4.9 COPYRIGHT Chunk

A standard text chunk containing the copyright notice. Standard Pascal-style string layout. The text typically follows the format "(C) YYYY Name" or similar.

### 4.10 ANNO Chunk (Annotation)

A general-purpose annotation text chunk. Unlike NAME and AUTHOR, which are single fields, a file may contain multiple ANNO chunks, each holding a separate annotation. Standard Pascal-style string layout.

### 4.11 COMMENT Chunk (Application-Specific)

The COMMENT chunk provides a place for applications to store arbitrary text annotations with a timestamp. This is distinct from ANNO in that it includes an additional time field.

**COMMENT Chunk Layout:**

```
┌──────────────────────────────────────────────────────────────────────┐
│  Offset  Bytes  Type       Description                               │
│  ─────   ─────  ────       ───────────                               │
│  0       4      int32 BE   Timestamp (seconds since 1904-01-01)      │
│  4       2      int16 BE   Comment text length                       │
│  6       N      char[]     Comment text (Pascal-style, padded)       │
└──────────────────────────────────────────────────────────────────────┘
```

The timestamp is stored as a 32-bit big-endian signed integer representing seconds since midnight, January 1, 1904 (the Macintosh date/time epoch, shared with Mac OS ResourceFork timestamps and MIDI File timestamps). This is different from the Unix epoch (January 1, 1970) used in most other systems. Converting: `unix_timestamp = mac_timestamp - 2082844800`.

---

## 5. BYTE ORDER AND ENDIANNESS

Big-endian byte order is used uniformly throughout AIFF and AIFC files. Every numeric value — chunk sizes, sample counts, sample data, floating-point sample rates, marker positions, instrument parameters — is stored with the most significant byte first.

This is a direct consequence of AIFF's origin on Motorola 68000-based platforms (Macintosh, NeXT, SGI), which use big-endian byte order natively. The big-endian storage was a pragmatic choice for the original target platforms but creates a compatibility challenge on modern little-endian x86/x64 systems.

**Key implications for implementation:**

On little-endian systems, every multi-byte integer read or written must be byte-swapped. This applies to all uint32, int32, uint16, int16, and uint80 values. The only exception is the audio sample data in SSND, which may contain odd bit-depth samples that are packed across byte boundaries — those bit fields must be unpacked byte-by-byte.

The 80-bit extended float used for sample rate is particularly important: it is a 10-byte big-endian value with a non-standard layout (explicit integer bit, 15-bit biased exponent). On little-endian systems, care must be taken to read the bytes in the correct order and to not accidentally attempt to interpret the 10-byte value as a native `long double` (which on x86 uses a different internal representation, typically 80-bit but with a different memory layout and potentially different endianness within the mantissa bytes depending on the platform).

**Byte-swapping reference for 16-bit and 32-bit values:**

```python
# Big-endian to host byte order (assumes little-endian host)
def be16_to_host(data: bytes) -> int:
    return int.from_bytes(data[:2], byteorder='big', signed=True)

def be32_to_host(data: bytes) -> int:
    return int.from_bytes(data[:4], byteorder='big', signed=True)

def be16_to_host_uint(data: bytes) -> int:
    return int.from_bytes(data[:2], byteorder='big', signed=False)

def be32_to_host_uint(data: bytes) -> int:
    return int.from_bytes(data[:4], byteorder='big', signed=False)

def host_to_be16(value: int) -> bytes:
    return value.to_bytes(2, byteorder='big', signed=True)

def host_to_be32(value: int) -> bytes:
    return value.to_bytes(4, byteorder='big', signed=True)
```

When writing a complete AIFF encoder, all multi-byte integer fields must be byte-swapped on output, and all multi-byte integer fields must be byte-swapped on input. The SSND sample data, for integer PCM, follows the same rule: each integer sample occupies (sampleSize + 7) / 8 bytes, stored big-endian (most significant byte first).

---

## 6. SAMPLE FORMATS AND BIT DEPTHS

### 6.1 8-bit Unsigned PCM

8-bit audio in AIFF uses unsigned representation, where the silence point is at 128 (mid-scale), full negative amplitude is 0, and full positive amplitude is 255. This is fundamentally different from every other common audio format, which uses signed representation.

```
Range:       0x00 (negative full scale)  ...  0x80 (silence)  ...  0xFF (positive full scale)
Interpretation: Full negative       →     Silence      →     Full positive
```

This unsigned encoding was chosen because early audio hardware (particularly on the Mac) used 8-bit unsigned DACs. When converting 8-bit AIFF to signed formats (like WAV 8-bit PCM, which uses signed -128 to +127), a simple offset adjustment is required: `signed_sample = unsigned_sample - 128`. Conversely, when converting signed 8-bit to AIFF unsigned: `unsigned_sample = signed_sample + 128`.

**Silence level:** In 8-bit AIFF, silence is exactly 128 (0x80). Any DC offset in the audio will cause the silence level to deviate from this value.

### 6.2 16-bit Signed PCM

16-bit is the most common bit depth for AIFF files. Samples are stored as signed two's complement integers, big-endian:

```
Range:       0x8000 (negative full scale: -32768)  ...  0x0000 (silence)  ...  0x7FFF (positive full scale: +32767)
```

Silence is at 0 (zero crossing). The full scale values are symmetric: -32768 and +32767, which is standard two's complement asymmetry.

### 6.3 20-bit PCM

20-bit audio occupies 3 bytes (24 bits) per sample, with the top 20 bits being the sample value and the bottom 4 bits of the last byte being unused padding (should be zero).

```
Byte layout (big-endian):  [S19-S12] [S11-S4] [S3-S0 | PADDING]
```

The 20-bit sample value is a signed integer, so the most significant bit (bit 19) is the sign bit. When converting to a 32-bit signed integer, the sample is typically left-aligned: `(sample_20bit << 12)`, preserving the sign. When converting from 32-bit to 20-bit, right-shift by 12: `sample_20bit = sample_32bit >> 12`, with clipping to the -2^19 to +2^19-1 range.

### 6.4 24-bit PCM

24-bit audio occupies 3 bytes per sample, using all 24 bits for the signed sample value.

```
Byte layout (big-endian):  [S23-S16] [S15-S8] [S7-S0]
```

When converting to 32-bit, left-align within 32 bits: `(sample_24bit << 8)`, which preserves the sign bit in bit 31. When converting from 32-bit to 24-bit, right-shift by 8: `sample_24bit = sample_32bit >> 8`, with clipping.

### 6.5 32-bit PCM

32-bit audio occupies 4 bytes per sample, stored as a signed 32-bit integer. This is rare in AIFF but fully supported by the specification. Full-scale values: -2147483648 to +2147483647.

### 6.6 Sample Interleaving

Regardless of bit depth, all channels for a given sample frame are stored consecutively before moving to the next frame. For a 4-channel file at 24-bit:

```
Frame 0: [Ch0: 3 bytes][Ch1: 3 bytes][Ch2: 3 bytes][Ch3: 3 bytes]
Frame 1: [Ch0: 3 bytes][Ch1: 3 bytes][Ch2: 3 bytes][Ch3: 3 bytes]
...
```

The total audio data size (excluding SSND header fields) must equal `numSampleFrames × nchannels × bytesPerSample`, where `bytesPerSample = ceil(sampleSize / 8)`. If this does not match the declared chunk size minus 8 (for the SSND header), the file is malformed.

---

## 7. AIFC COMPRESSION TYPES

AIFC (AIFF with Compression) extends AIFF to support compressed audio data within the same chunk-based container. The compression type is declared in the COMM chunk via the `compressionType` field (4 bytes) and the `compressionName` string. All compressed audio still uses the SSND chunk for storage, but the data within SSND is codec-specific.

### 7.1 Compression Type Registry

| Compression ID | Compression Name       | Description                                | Lossless? | Typical Use Case       |
|----------------|-----------------------|--------------------------------------------|-----------|------------------------|
| `NONE`         | not specified         | No compression (same as AIFF PCM)          | Yes       | Backward compatibility |
| `sowt`         | not specified         | Little-endian 16-bit PCM (byte-swapped AIFF)| Yes      | Cross-platform PCM     |
| `twos`         | not specified         | Big-endian 16-bit signed PCM (standard AIFF)| Yes      | Standard AIFF PCM      |
| `alaw`         | not specified         | G.711 A-law (8-bit)                         | No        | Telecom, legacy systems |
| `ulaw`         | not specified         | G.711 μ-law (8-bit)                         | No        | Telecom, legacy systems |
| `G722`         | not specified         | G.722 ADPCM (16 kHz sample rate)            | No        | Wideband telecom       |
| `G726`         | not specified         | G.726 ADPCM (variable bit rate)             | No        | Telecom                |
| `ima4`         | not specified         | IMA ADPCM (4-bit, cross-platform)           | No        | General purpose ADPCM  |
| `MAC3`         | not specified         | Apple MACE Type 3 (3:1 compression)        | No        | Apple legacy           |
| `MAC6`         | not specified         | Apple MACE Type 6 (6:1 compression)        | No        | Apple legacy           |
| `fl32`         | not specified         | 32-bit IEEE float (big-endian)              | Yes       | Floating-point audio   |
| `fl64`         | not specified         | 64-bit IEEE double (big-endian)             | Yes       | High-precision audio   |
| `ulaw`         | not specified         | G.711 μ-law / ITU G.711                    | No        | Telecom, telephony     |
| `alaw`         | not specified         | G.711 A-law / ITU G.711                     | No        | Telecom, telephony     |

### 7.2 NONE (Uncompressed PCM)

Compression ID: `NONE` (0x4E4F4E45)

This is not truly a compression format. The `NONE` type indicates that the audio data in the SSND chunk is uncompressed PCM, identical in layout to standard AIFF. When a file declares `NONE` compression, it is effectively an AIFC file with no compression — typically used when a tool needs to indicate it is using the AIFC extension (perhaps to carry a compression name string or other AIFC-specific metadata) but storing uncompressed audio. The SSND data is the same as AIFF PCM.

### 7.3 sowt (Little-Endian PCM)

Compression ID: `sowt` (0x736F7774)

This compression type signals that the audio data in the SSND chunk is 16-bit signed PCM, but stored in little-endian byte order instead of the standard big-endian. This is a compatibility hack: AIFF was designed for big-endian platforms, but on little-endian systems, writing big-endian PCM requires byte-swapping. Some tools store little-endian PCM and mark it as `sowt` to avoid the byte-swap overhead on those platforms. When reading `sowt`, treat the SSND data as little-endian 16-bit PCM and byte-swap on big-endian hosts. When writing `sowt`, write 16-bit PCM in little-endian.

### 7.4 twos (Big-Endian Signed PCM)

Compression ID: `twos` (0x74776F73)

This explicitly indicates 16-bit signed big-endian PCM, identical to standard AIFF. The `twos` compression type was defined to differentiate from the `sowt` variant and to allow AIFC files to carry PCM audio while explicitly naming the format.

### 7.5 alaw (G.711 A-Law)

Compression ID: `alaw` (0x616C6177)

G.711 A-law encoding. A-law is a companding (compressing-expanding) algorithm used primarily in European telecommunications. It compresses 16-bit linear PCM to 8 bits using a piecewise linear approximation of the A-law curve. The compression ratio is 2:1 (16 bits → 8 bits).

A-law encoding uses a specific ITU-T G.711 table. The 16-bit signed PCM input is converted to an 8-bit A-law codeword. A-law has a slightly different dynamic range and quantization characteristics compared to μ-law, with a different sign bit encoding and a different compression curve near the origin.

In AIFC, the A-law data in the SSND chunk is stored as 8-bit A-law codewords. The sample rate in the COMM chunk reflects the input sample rate (e.g., 8000 Hz for narrowband telephony, 16000 Hz for wideband if using G.711 at higher rates). The number of sample frames equals the number of A-law codewords (one per sample frame).

### 7.6 ulaw / ulaw (G.711 μ-Law)

Compression ID: `ulaw` (0x756C6177)

G.711 μ-law (mu-law) encoding. μ-law is used primarily in North America and Japan. Like A-law, it compresses 16-bit PCM to 8 bits at a 2:1 ratio using a logarithmic companding curve.

μ-law encoding follows the ITU-T G.711 specification. The 16-bit signed PCM input is converted to an 8-bit μ-law codeword using the standard μ-law table. The compression curve provides approximately 14–15 bits of effective dynamic range for the most important portions of the audio signal (near silence).

The μ-law format encodes sign and magnitude differently from A-law. The conversion between μ-law and linear PCM requires a table lookup or computation. Common implementations use a 256-entry lookup table for μ-law → linear and a formula for linear → μ-law.

### 7.7 G722 (ITU-T G.722 ADPCM)

Compression ID: `G722` (0x47373232)

G.722 is a wideband (7 kHz audio bandwidth) speech coding standard that uses Sub-Band ADPCM (SB-ADPCM). The audio signal is split into two sub-bands (low and high), and each is encoded using ADPCM. The total bit rate is 64 kbps. For AIFF/AIFC, G.722 encoded data is stored in the SSND chunk with the sample rate declared in the COMM chunk reflecting the input sample rate (typically 16000 Hz output from the decoder, since G.722 processes 16 kHz audio).

### 7.8 G726 (ITU-T G.726 ADPCM)

Compression ID: `G726` (0x47373236)

G.726 ADPCM supports multiple bit rates: 40, 32, 24, and 16 kbps. The G.726 codec operates on G.711 μ-law or A-law input (64 kbps PCM). The AIFC declaration may not always specify the exact bit rate within the compression name, making it ambiguous. The SSND chunk contains the raw G.726 bitstream.

### 7.9 ima4 (IMA ADPCM)

Compression ID: `ima4` (0x696D6134)

The IMA ADPCM codec used in AIFF-C files is distinct from Microsoft's DVI ADPCM variant, though they share the same conceptual approach: 4-bit adaptive differential encoding with a step size table and predictor. The IMA ADPCM used by Apple is based on the IMA's recommended practice for ADPCM encoding.

**IMA ADPCM encoding algorithm:**
1. The predictor maintains an estimate of the next sample based on previous samples.
2. The difference between the actual sample and the predictor is quantized to 4 bits using a step size derived from a 16-entry step size table indexed by the previous quantized difference.
3. The 4-bit code is stored, and the predictor is updated based on the step size and the quantized difference.
4. Initial step size is determined by the input bit depth (e.g., 16-bit uses an initial step size based on the variance of the first few samples).

**IMA ADPCM decoding algorithm:**
1. Read the 4-bit code.
2. Look up the step size from the table using the previous step size and the current code.
3. Reconstruct the sample by adding the signed difference (computed from the code and step size) to the previous predicted sample.
4. Clamp the result to the valid range.

IMA ADPCM typically achieves 4:1 compression (16-bit → 4-bit). In AIFC files, the compressed data is stored in the SSND chunk. The block size for IMA ADPCM is typically 34 bytes per block for mono (containing 64 compressed samples), with a header of 2 bytes per block indicating the initial predictor state. This block structure must be accounted for when computing sample frame counts from the SSND data size.

**Compressed block format for IMA4 mono:**
- 2-byte block header: initial predictor value (16-bit big-endian signed).
- Remaining 32 bytes: 64 × 4-bit samples, packed two samples per byte (odd/even nibble ordering follows IMA specification — first sample in high nibble, second in low nibble).

### 7.10 MAC3 and MAC6 (Apple MACE)

Compression IDs: `MAC3` (0x4D414333) and `MAC6` (0x4D414336)

Apple MACE (Macintosh Audio Compression and Expansion) is a proprietary compression scheme developed by Apple. MAC3 provides approximately 3:1 compression (16-bit → ~5.3 bits per sample), while MAC6 provides approximately 6:1 compression (16-bit → ~2.7 bits per sample). Both are lossy. These formats were used in older Mac applications but are now obsolete. Modern decoders for MACE are rare outside of Apple's legacy code.

---

## 8. SSND CHUNK DATA ORGANIZATION

The SSND chunk contains the audio sample data, organized according to the format declared in the COMM chunk. Understanding the precise byte layout is essential for correct implementation.

### 8.1 PCM Data Layout

For uncompressed PCM (AIFF or AIFC with `NONE`, `twos`, `fl32`, `fl64` compression), the SSND data is a straightforward interleaved stream:

```
SSND data bytes = numSampleFrames × nchannels × ceil(sampleSize / 8)
```

For 16-bit stereo at 44100 Hz, 10 seconds: `441000 frames × 2 channels × 2 bytes = 1,764,000 bytes`.

Each sample frame occupies `nchannels × bytesPerSample` bytes. Within a frame, channels are ordered sequentially: channel 0 sample first, channel 1 second, etc. For multi-byte samples (bit depth > 8), bytes are stored big-endian.

### 8.2 Block Alignment

The `blockSize` field in the SSND header enables block-structured data. For PCM, blockAlign (confusingly named `blockSize` in the header) should equal the frame size. For compressed formats like IMA ADPCM, the blockAlign specifies the number of bytes per compressed block, and numSampleFrames may not be derivable from the raw data size alone — compressed blocks have a specific structure.

When writing a compliant AIFF, set both the `offset` and `blockAlign` fields to 0 if the data is not block-structured. This is the most common case and ensures maximum compatibility.

### 8.3 SSND Chunk Size and Padding

The chunk size field in the SSND header is the size of the data portion only (offset + blockAlign fields plus all sound data), excluding the 8-byte chunk header. If the total data size is odd, a single padding byte (0x00) is appended and the chunk size field is set to the unpadded size (the padding byte is not included in the size). This is standard IFF behavior.

---

## 9. CHUNK ORDERING RULES

The Apple AIFF specification defines a recommended chunk ordering. While not all parsers enforce strict ordering, adhering to it ensures maximum compatibility and enables streaming (reading format information before encountering large audio data).

**Canonical ordering:**

```
FORM (header, always first)
  └── FVER (AIFC only, must be first chunk after FORM header)
  └── COMM (format descriptor, must precede SSND)
  └── SSND (audio data, must follow COMM)
  └── MARK (markers)
  └── INST (instrument)
  └── NAME (title)
  └── AUTHOR (artist)
  └── COPYRIGHT (rights notice)
  └── ANNO (annotations, may appear multiple times)
  └── COMMENT (application annotations, may appear multiple times)
  └── [Application-specific chunks]
```

**Rules:**
- FVER must appear before any other chunk in an AIFC file.
- COMM must appear before SSND.
- There should be exactly one of each: FORM, FVER, COMM, SSND.
- There may be multiple: MARK, INST, NAME, AUTHOR, COPYRIGHT, ANNO, COMMENT.
- Unknown chunks should be skipped, maintaining forward compatibility.

Some applications are lenient about chunk ordering and can read files where chunks appear in non-standard order (e.g., NAME before COMM). However, strict conformance requires parsers to search for the required chunks regardless of order.

---

## 10. MARKER AND INSTRUMENT CHUNKS

### 10.1 Marker IDs

Marker IDs are signed 16-bit integers ranging from -32768 to 32767, but in practice, applications typically use small positive values (1, 2, 3, ...) or values assigned sequentially. ID 0 is reserved and must not be used. IDs need not be unique across all markers — loop start and loop end in the INST chunk can reference the same marker ID if needed, though this is unusual.

### 10.2 Marker Use Cases

Markers are commonly used to denote:
- Loop start and end points (referenced by INST chunk)
- Section boundaries (verse, chorus, bridge, etc.)
- Edit points
- Tempo change locations
- Regions for无损切割

When the INST chunk references marker IDs, those markers must exist in a MARK chunk for the references to be meaningful. If a referenced marker ID is not found, the loop configuration should be treated as invalid or the reference as unresolved.

### 10.3 INST Chunk Loops

The INST chunk's loop configuration references MARK chunk entries by ID. The relationship is:

- `sustainLoop.playMode` defines how the sustain loop is played: 0 = no sustain loop, 1 = forward, 2 = forward/backward, 3 = backward.
- `sustainLoop.beginMarker` and `sustainLoop.endMarker` hold marker IDs pointing to MARK chunk entries that define the loop start and end sample frame positions.
- `releaseLoop` has the same structure for the release phase of the envelope.

When implementing a player or sampler that reads AIFF with INST chunks, the loop playback should:
1. Read the sustain loop play mode.
2. Look up the begin and end marker IDs in the MARK chunk.
3. If both markers exist, use their positions as loop points.
4. If either marker is missing (ID 0 or not found), treat the loop configuration as undefined.

### 10.4 baseFrequency and MIDI Mapping

The `baseFrequency` field in the INST chunk defines the pitch of the audio relative to middle A (440 Hz). To convert to a MIDI note number:

```python
def aiff_base_freq_to_midi(base_frequency: int) -> int:
    """Convert AIFF INST baseFrequency to MIDI note number."""
    # base_frequency is signed: number of semitones from middle A
    # Middle A is MIDI note 69
    return 69 + base_frequency

def midi_to_aiff_base_freq(midi_note: int) -> int:
    """Convert MIDI note number to AIFF INST baseFrequency."""
    return midi_note - 69
```

The `detune` field adds fine-tuning in cents (-50 to +50). To compute the actual frequency:

```python
def compute_frequency(base_midi_note: int, detune_cents: int, base_a4: float = 440.0) -> float:
    """Compute the actual frequency of a note with detuning."""
    import math
    # MIDI note 69 = A4 = 440 Hz
    semitones_from_a4 = base_midi_note - 69 + (detune_cents / 100.0)
    return base_a4 * math.pow(2.0, semitones_from_a4 / 12.0)
```

---

## 11. 8-BIT AUDIO ENCODING

The 8-bit unsigned encoding in AIFF is a historical artifact that requires careful handling in conversion pipelines. It is fundamentally incompatible with the signed representation used by virtually every other audio format.

### 11.1 Encoding Details

In 8-bit AIFF, each sample occupies exactly 1 byte. The value 128 (0x80) is the silence/midpoint. Values below 128 represent negative amplitudes (with 0 as minimum amplitude, not minimum value). Values above 128 represent positive amplitudes (with 255 as maximum amplitude).

| Byte Value | Interpretation                          | Signed Equivalent |
|------------|-----------------------------------------|-------------------|
| 0x00       | Minimum amplitude (full negative)        | -128              |
| 0x01       | Very low amplitude (negative)            | -127              |
| ...        | ...                                     | ...               |
| 0x7F       | Low positive amplitude                  | -1                |
| 0x80       | Silence (zero crossing)                 | 0                 |
| 0x81       | Low positive amplitude                  | +1                |
| ...        | ...                                     | ...               |
| 0xFE       | High positive amplitude                  | +126              |
| 0xFF       | Maximum amplitude (full positive)        | +127              |

### 11.2 Conversion to Signed PCM (WAV, FLAC, etc.)

When converting 8-bit AIFF to a signed format, apply the offset:

```python
def aiff8_to_signed(aiff_byte: int) -> int:
    """Convert 8-bit AIFF unsigned byte to signed 8-bit."""
    return aiff_byte - 128

def aiff8_to_signed16(aiff_byte: int) -> int:
    """Convert 8-bit AIFF unsigned byte to signed 16-bit, scaled to full range."""
    signed_8 = aiff_byte - 128
    # Scale to 16-bit range: multiply by 256
    return signed_8 * 256
```

The multiplication by 256 when scaling to 16-bit preserves the relative amplitude levels correctly. The value 128 in AIFF maps to 0 in signed 16-bit; the value 0 in AIFF maps to -32768; the value 255 in AIFF maps to +32512 (not +32767, because the signed 8-bit range is -128 to +127, not symmetric).

### 11.3 Conversion from Signed PCM to 8-bit AIFF

```python
def signed_to_aiff8(signed_sample: int) -> int:
    """Convert signed 8-bit sample to 8-bit AIFF unsigned."""
    # Clamp to valid signed 8-bit range first
    clamped = max(-128, min(127, signed_sample))
    return clamped + 128

def signed16_to_aiff8(signed_16: int) -> int:
    """Convert signed 16-bit sample to 8-bit AIFF unsigned."""
    # Divide by 256 (right shift) and clamp
    scaled = signed_16 >> 8
    return signed_to_aiff8(scaled)
```

### 11.4 Practical Implications

The 8-bit unsigned format was designed for 8-bit DAC hardware that represented voltage levels linearly from ground to Vref. On modern systems, 8-bit AIFF files are rarely encountered, but they do exist in legacy archives and are still generated by some applications. Any robust AIFF decoder must handle 8-bit audio correctly, or the output will be severely distorted (every sample will be off by 128 units).

FFmpeg handles 8-bit AIFF correctly by internally converting to its signed representation. When using FFmpeg for conversions, specify `-sample_fmt s16p` (or appropriate output format) to ensure correct signed output.

---

## 12. AIFF-C VERSION TRACKING

The FVER chunk encodes the AIFC specification version as a date/timestamp. This mechanism was designed to allow future revisions of the AIFC specification while maintaining backward compatibility.

### 12.1 Version Values

| Version Value | Decimal | Hex (BE uint32) | Description                   |
|---------------|---------|-----------------|--------------------------------|
| AIFC version 1 | 173857  | 0x0002A861       | Original AIFC specification (Nov 24, 1991) |

The timestamp 173857 in decimal corresponds to the date November 24, 1991. When encoded as a big-endian uint32, this is `0x0002A861`. The original AIFC specification document uses both decimal and hexadecimal representations, sometimes creating confusion.

### 12.2 Handling Unknown Versions

When reading an AIFC file with an unknown or future FVER timestamp, the reader should:
1. Check if the compression type is supported.
2. Attempt to decode the audio data.
3. If decoding fails, report the file as unsupported due to an incompatible AIFC version.

A reader should never refuse to open a file simply because the FVER timestamp is greater than 173857, as future revisions may maintain backward compatibility.

### 12.3 Version Interaction with Compression

The FVER chunk's presence confirms the file uses the AIFC extension. Even if the compression type is `NONE` (no compression), the presence of FVER means the file follows AIFC rules for the COMM chunk (which includes the compression type field). An AIFF file with compression type `NONE` in the COMM chunk but no FVER is technically an AIFF file (not AIFC), which may confuse some strict parsers. For maximum compatibility, if writing AIFC with compression information, include the FVER chunk.

---

## 13. FILE SIZE LIMITS AND EXTENSIONS

### 13.1 The 4 GB Limit

The original AIFF/AIFC specification uses uint32 for chunk sizes, which imposes a hard limit of 4,294,967,295 bytes (~4 GB) for any individual chunk, including the FORM container's size field. In practice, the FORM size field is often set to 0xFFFFFFFF (indicating unknown size) because the actual file size may exceed what can be stored in 32 bits.

### 13.2 Handling Large Files

When encountering a file with FORM size = 0xFFFFFFFF:
1. Read chunks sequentially until EOF.
2. For each chunk, use the chunk's own size field to skip to the next chunk.
3. The total audio data size can be computed from the COMM chunk: `numSampleFrames × nchannels × ceil(sampleSize / 8)`.

### 13.3 AIFC and 64-bit Sizes

The AIFC specification references 64-bit extended integers for sound data chunk sizes, but this was never widely implemented. Modern AIFC files with audio data exceeding 4 GB typically use the 0xFFFFFFFF placeholder in the FORM size field and rely on sequential reading.

### 13.4 RF64 and BWF Comparison

For reference, the EBU's RF64 specification extends the BWF (Broadcast Wave Format) to handle files over 4 GB using a `ds64` chunk that carries 64-bit sizes. AIFF has no equivalent standardized extension. If you need to store more than 4 GB of audio in a format with chunked structure, CAF (Core Audio Format) is the recommended modern alternative.

---

## 14. METADATA HANDLING

### 14.1 Metadata Chunk Summary

| Chunk ID   | Cardinality | Content                   | Encoding            |
|------------|-------------|---------------------------|---------------------|
| NAME       | 0 or 1      | Title / name              | Pascal string       |
| AUTHOR     | 0 or 1      | Artist / author           | Pascal string       |
| COPYRIGHT  | 0 or 1      | Copyright notice          | Pascal string       |
| ANNO       | 0 or many   | General annotations       | Pascal string       |
| COMMENT    | 0 or many   | Timestamped annotations   | Pascal + timestamp  |
| MARK       | 0 or 1      | Named positions           | Structured list     |
| INST       | 0 or 1      | Instrument parameters     | Binary structured   |

### 14.2 Pascal String Encoding

All text chunks in AIFF use Pascal-style strings: a length byte (0–255) followed by the character data. The length byte counts the characters, not including the length byte itself. The string is padded to an even byte boundary with a 0x00 padding byte if necessary.

```
Example: "Hello" stored in AIFF text chunk
Byte 0: 0x05 (length = 5)
Bytes 1-5: 'H' 'e' 'l' 'l' 'o'
Byte 6: 0x00 (padding to even boundary)
```

The maximum text length is 255 characters (limited by the length byte). This is a significant constraint compared to modern tag formats that support arbitrary-length strings.

### 14.3 Cross-Format Metadata Mapping

When converting between AIFF and other formats, the following mappings are conventional:

| AIFF Chunk    | WAV LIST INFO    | MP3 ID3v2     | Vorbis Comment   |
|---------------|------------------|---------------|-------------------|
| NAME          | INAM / TTITL     | TIT2          | TITLE             |
| AUTHOR        | IART             | TPE1 / TPE2   | ARTIST            |
| COPYRIGHT     | ICOP             | TCOP          | COPYRIGHT         |
| ANNO / COMMENT| ICMT / ICRM      | COMM          | COMMENT           |
| MARK          | cue (position)   | —             | —                 |

These mappings are not standardized and vary between tools. When building a conversion pipeline, define explicit mapping rules and document them.

### 14.4 Cover Art in AIFF

AIFF has no standard mechanism for embedded cover art or artwork. This is a significant limitation compared to MP4 (which uses the `covr` atom) or Vorbis/FLAC (which uses `METADATA_BLOCK_PICTURE`). Some applications have used private chunks (chunks with application-specific IDs starting with a registered creator ID) to embed artwork, but this is non-standard and not portable.

When converting audio that has embedded artwork from a format with artwork (MP4, FLAC, Vorbis) to AIFF, the artwork must be discarded or stored in a private chunk with clear documentation of its format and the application that wrote it.

---

## 15. FFmpeg AIFF/AIFC SUPPORT

### 15.1 FFmpeg Format Identification

FFmpeg identifies AIFF files by the `FORM` header followed by `AIFF` or `AIFC`. The demuxer (`aiff` for FFmpeg) handles both AIFF and AIFC.

**FFmpeg format names:**
- `aiff` — AIFF (uncompressed PCM)
- `aifc` — AIFC (compressed, including `sowt`)

In practice, FFmpeg uses a single `aiff` demuxer for both, distinguished by the FORM type.

### 15.2 FFmpeg AIFF/AIFC Encoding Options

Key FFmpeg options for AIFF/AIFC output:

```
FFmpeg AIFF/AIFC Encoding Options
================================

-audio_codec (codec selection):
  pcm_s16be    → 16-bit big-endian PCM (standard AIFF)
  pcm_s8       → 8-bit signed PCM (NOT standard AIFF; use pcm_u8 for unsigned)
  pcm_u8       → 8-bit unsigned PCM (standard AIFF 8-bit)
  pcm_s24be    → 24-bit big-endian PCM
  pcm_s32be    → 32-bit big-endian PCM
  alaw         → G.711 A-law (AIFC)
  ulaw         → G.711 μ-law (AIFC)
  adpcm_ima_ws → IMA ADPCM (AIFC, compatible with ima4)

-bits_per_coded_sample:
  Override the bits per sample written to the COMM chunk.
  Default is derived from the audio codec.

-sample_rate:
  Set the sample rate. Stored as 80-bit extended float in AIFF.
  Common values: 44100, 48000, 96000, 192000.

-channel_layout:
  Specify channel arrangement. AIFF does not define channel layouts,
  so FFmpeg uses the channel count only.

-write_id3v2:
  Enable ID3v2 tag writing on AIFF files (non-standard extension).
  Allows ID3 tags to be embedded as a private chunk for cross-tool
  compatibility with MP3 metadata tools.
```

### 15.3 Common FFmpeg AIFF Commands

**Convert WAV to AIFF (16-bit stereo, 44100 Hz):**
```bash
ffmpeg -i input.wav -c:a pcm_s16be -ar 44100 -ac 2 output.aiff
```

**Convert WAV to AIFF with ID3 tags preserved:**
```bash
ffmpeg -i input.wav -c:a pcm_s16be -write_id3v2 1 -metadata title="Track Title" \
       -metadata artist="Artist Name" output.aiff
```
Note: ID3v2 in AIFF is non-standard. FFmpeg stores it in a private `ID3 ` chunk.

**Convert AIFF to WAV (automatic byte order handling):**
```bash
ffmpeg -i input.aiff -c:a pcm_s16le output.wav
```
FFmpeg automatically handles big-endian to little-endian conversion.

**Convert AIFF to FLAC:**
```bash
ffmpeg -i input.aiff -c:a flac -compression_level 8 output.flac
```

**Convert AIFF to AAC (lossy):**
```bash
ffmpeg -i input.aiff -c:a aac -b:a 256k output.m4a
```

**Convert WAV to AIFC (IMA ADPCM):**
```bash
ffmpeg -i input.wav -c:a adpcm_ima_ws output.aifc
```

### 15.4 FFmpeg Byte Order Behavior

FFmpeg's AIFF muxer writes PCM data in the native byte order of the selected codec. For `pcm_s16be`, data is written big-endian (standard AIFF). For `pcm_s16le`, data is written little-endian (which is non-standard and should be avoided). The muxer handles the SSND header's blockAlign and offset fields automatically.

When using `-c:a pcm_s16be` (big-endian), FFmpeg writes audio data in big-endian format, which is correct for AIFF. When using `-c:a pcm_s16le` (little-endian), FFmpeg writes little-endian data, which is technically non-compliant for AIFF and should not be used for standard AIFF output.

---

## 16. IMPLEMENTATION CHECKLIST

### 16.1 AIFF Reader Implementation

**Header Parsing:**
- [ ] Read first 12 bytes: verify "FORM" magic, read FORM size, verify "AIFF" or "AIFC" form type.
- [ ] If form type is "AIFC": search for FVER chunk. If "AIFF": FVER must not exist. If FVER exists for AIFF, treat as error.
- [ ] Search for COMM chunk. If not found, file is invalid.
- [ ] Search for SSND chunk. If not found, file is invalid.
- [ ] For all remaining chunks, read ID and size, skip to next chunk (with padding).
- [ ] Handle FORM size = 0xFFFFFFFF by reading until EOF.

**COMM Chunk Parsing:**
- [ ] Read nchannels (int16 BE).
- [ ] Read numSampleFrames (int32 BE).
- [ ] Read sampleSize (int16 BE).
- [ ] Read sampleRate (uint80 BE extended float). Decode correctly — do not treat as native float.
- [ ] For AIFC: read compression type (4 bytes). Read compression name length. Read compression name string (Pascal string).
- [ ] Validate: nchannels >= 1, numSampleFrames >= 0, sampleSize >= 1.

**SSND Chunk Parsing:**
- [ ] Read blockSize (offset) and blockAlign (blockSize field) — both uint32 BE.
- [ ] Compute expected data size: `numSampleFrames × nchannels × ceil(sampleSize / 8)`.
- [ ] If chunk size - 8 does not equal expected data size, log a warning (possible non-standard alignment or embedded cover art in a private chunk).
- [ ] Skip offset bytes before reading audio data.
- [ ] Read audio data in frame order, deinterleaving if needed.

**Sample Rate Decoding (80-bit extended float):**
- [ ] Read 10 bytes big-endian.
- [ ] Extract sign bit (bit 7 of byte 1).
- [ ] Extract exponent: combine bits 0–6 of byte 1 with byte 0, subtract 16383 bias.
- [ ] Extract mantissa: 64-bit value from bytes 2–9, with explicit integer bit.
- [ ] Compute: `result = (-1)^sign × 2^(exponent-16383) × (1 + mantissa/2^63)`.
- [ ] Validate: exponent of 0 with mantissa of 0 = zero. Exponent of all 1s = infinity or NaN.

**8-bit Audio Handling:**
- [ ] Check sampleSize == 8.
- [ ] When converting to signed output, subtract 128 from every sample.
- [ ] When scaling 8-bit to 16-bit, multiply by 256 after offset correction.
- [ ] Never treat 8-bit AIFF samples as signed without the offset.

**Multi-Byte Sample Handling:**
- [ ] For sampleSize <= 8: each sample is 1 byte. Signed for sampleSize == 8 (unsigned offset applied). For other values, zero-pad.
- [ ] For 8 < sampleSize <= 16: each sample is 2 bytes, big-endian signed.
- [ ] For 16 < sampleSize <= 24: each sample is 3 bytes, big-endian signed.
- [ ] For 24 < sampleSize <= 32: each sample is 4 bytes, big-endian signed.
- [ ] For 32 < sampleSize: each sample is ceil(sampleSize/8) bytes, big-endian signed.
- [ ] Implement sign extension correctly for values that don't fill all bytes of their storage (e.g., 20-bit samples in 24-bit storage).

**Marker Parsing:**
- [ ] Read marker count (int16 BE).
- [ ] For each marker: read ID (int16 BE), position (int32 BE), name length (int16 BE), name (Pascal string).
- [ ] Validate marker IDs are unique within the chunk.
- [ ] Validate marker positions do not exceed numSampleFrames.
- [ ] Convert marker positions to time (seconds) using sample rate: `time_seconds = position / sampleRate`.

**Metadata Reading:**
- [ ] Read NAME, AUTHOR, COPYRIGHT as Pascal strings.
- [ ] Read ANNO as Pascal strings (may have multiple ANNO chunks).
- [ ] Read COMMENT chunks with timestamp (convert Mac epoch to Unix epoch: subtract 2082844800).
- [ ] All text encoding is ASCII (7-bit). Handle gracefully; some files may contain Latin-1 or MacRoman characters.

**Error Handling:**
- [ ] If COMM chunk is missing or invalid: reject file.
- [ ] If SSND chunk is missing: reject file.
- [ ] If audio data size does not match declared parameters: log warning, attempt best-effort reading with declared parameters.
- [ ] If sample rate extended float is invalid (NaN, infinity): reject file or use fallback sample rate (44100 Hz) with warning.
- [ ] If chunk size is odd: skip padding byte correctly.
- [ ] If chunk ID is unknown: skip chunk using its declared size + padding.

### 16.2 AIFF Writer Implementation

**File Header:**
- [ ] Write "FORM" (0x464F524D).
- [ ] Reserve 4 bytes for FORM size (fill with 0 or 0xFFFFFFFF for large files).
- [ ] Write "AIFF" (0x41494646) or "AIFC" (0x41494643).
- [ ] If AIFC: write FVER chunk with version 173857.

**COMM Chunk:**
- [ ] Write "COMM" (0x434F4D4D).
- [ ] Compute and write nchannels (int16 BE).
- [ ] Compute and write numSampleFrames (int32 BE).
- [ ] Write sampleSize (int16 BE).
- [ ] Encode sample rate as 80-bit extended float (big-endian). Validate encoding.
- [ ] If AIFC: write compression type (4 bytes) and compression name (Pascal string).

**Sample Rate Encoding (80-bit extended float):**
- [ ] Convert double sample rate to 80-bit extended format.
- [ ] Use appropriate rounding to nearest representable value.
- [ ] Store as 10 bytes big-endian.

**SSND Chunk:**
- [ ] Write "SSND" (0x53534E44).
- [ ] Write offset = 0 (most common case).
- [ ] Write blockAlign = frame size.
- [ ] Write audio data interleaved, big-endian, with correct byte packing.
- [ ] If total audio data size is odd, write one padding byte (0x00) and include it in the SSND chunk size.

**8-bit Output:**
- [ ] For 8-bit AIFF: convert signed samples to unsigned by adding 128.
- [ ] Validate: input range must be -128 to +127 after conversion; clamp if necessary.

**Chunk Ordering:**
- [ ] Write FVER (if AIFC) first.
- [ ] Write COMM before SSND.
- [ ] Write SSND immediately after COMM.
- [ ] Write optional chunks after SSND in canonical order.

**FORM Size Computation:**
- [ ] Sum all chunk sizes (including padding bytes) plus 4 bytes for form type.
- [ ] Write the FORM size to the reserved bytes at offset 4.
- [ ] If total exceeds 0xFFFFFFFF, write 0xFFFFFFFF and handle at read time.

### 16.3 AIFC Compressed Audio Implementation

**Compression Type Selection:**
- [ ] For cross-platform compatibility with maximum lossless quality: use `NONE` (uncompressed PCM) with AIFC header.
- [ ] For standard big-endian PCM with explicit compression type: use `twos`.
- [ ] For little-endian platforms that want to avoid byte-swapping: use `sowt`.
- [ ] For cross-platform IMA ADPCM: use `ima4`.
- [ ] For G.711 telephony audio: use `alaw` or `ulaw`.

**IMA ADPCM Encoding:**
- [ ] Initialize predictor with 16-bit signed value (typically 0).
- [ ] Process samples in blocks of 64 (for mono).
- [ ] For each block: write 2-byte header (initial predictor, big-endian). Write 32 bytes of compressed nibbles.
- [ ] Use IMA step size table: index = clamp(|difference| / 2, 0, 88).
- [ ] Output 4-bit codes using IMA nibble ordering (high nibble first, low nibble second).

**IMA ADPCM Decoding:**
- [ ] For each block: read 2-byte header to get initial predictor.
- [ ] Read nibbles: first sample from high nibble, second from low nibble.
- [ ] Look up step size from table using current index.
- [ ] Reconstruct sample: `predicted += step_size × (code - (code & 8 ? 8 : 0))`.
- [ ] Clamp predicted value to -32768..32767.
- [ ] Update index: `index = clamp(index + table[code & 0x07], 0, 88)`.

**G.711 μ-law / A-law:**
- [ ] Use ITU-T G.711 tables for encode and decode.
- [ ] μ-law: input 14-bit signed linear, output 8-bit codeword.
- [ ] A-law: input 13-bit signed linear (14-bit with sign), output 8-bit codeword.
- [ ] G.711 operates at 8000 Hz sample rate. Set COMM sample rate accordingly.
- [ ] For 16 kHz (G.722), declare 16000 Hz in COMM.

### 16.4 Conversion Pipeline Integration

**AIFF to WAV:**
- [ ] Read AIFF as described above.
- [ ] Write WAV RIFF header with little-endian byte order.
- [ ] Write fmt chunk with audio format 0x0001 (PCM) or 0x0003 (IEEE float) if appropriate.
- [ ] Convert 8-bit unsigned AIFF to signed WAV.
- [ ] Convert big-endian multi-byte samples to little-endian.
- [ ] Compute and write the fact chunk if format is not standard PCM (e.g., 20-bit, compressed).

**WAV to AIFF:**
- [ ] Read WAV fmt chunk to determine format, channels, sample rate, bit depth.
- [ ] Write FORM header, COMM chunk, SSND chunk.
- [ ] Convert signed WAV samples to unsigned AIFF for 8-bit output.
- [ ] Convert little-endian WAV samples to big-endian AIFF.
- [ ] Encode sample rate as 80-bit extended float.
- [ ] Write markers (if present in WAV cue chunk) to MARK chunk.
- [ ] Write text metadata to NAME, AUTHOR, COPYRIGHT, ANNO chunks.

**AIFF to FLAC:**
- [ ] Read AIFF PCM samples.
- [ ] Convert 8-bit unsigned to signed.
- [ ] Write FLAC frames with correct channel count and sample rate.
- [ ] Write Vorbis Comment metadata for NAME, AUTHOR, COPYRIGHT, ANNO.
- [ ] For MARK/INST chunks: convert to FLAC CUE sheet or embedded markers if supported.

**FLAC to AIFF:**
- [ ] Read FLAC frame data (decode to PCM).
- [ ] Write AIFF COMM with sample rate, channels, bit depth.
- [ ] For 16-bit output: dither if desired.
- [ ] Write SSND with interleaved PCM.
- [ ] Map Vorbis Comment fields to AIFF text chunks.

---

## 17. COMMON PITFALLS AND IMPLEMENTATION NOTES

### 17.1 The 80-bit Sample Rate Trap

The single most common bug in AIFF implementations is mishandling the 80-bit extended float sample rate. Many implementations incorrectly read the 10-byte value as if it were a native float or double, resulting in garbage sample rate values (e.g., 44100 being read as some large garbage number). Always implement correct 80-bit extended float decoding.

### 17.2 The 8-bit Signed Offset Trap

Converting 8-bit AIFF without accounting for the unsigned offset produces audio that sounds like heavy static or distortion (everything is off by 128 units). Always subtract 128 when reading 8-bit AIFF and add 128 when writing.

### 17.3 Byte Order Confusion

The big-endian byte order of AIFF means every multi-byte field must be byte-swapped on little-endian platforms. Forgetting to swap the sample rate, sample count, or audio data produces garbled output. Implement a clear endianness handling layer and test with both big-endian and little-endian sample data.

### 17.4 Odd-Sized Chunk Padding

IFF chunks are always aligned to even byte boundaries. If a chunk's data size is odd, a padding byte (0x00) is appended. When skipping unknown chunks or computing file offsets, always account for this padding rule. Failing to do so causes the parser to misalign with subsequent chunks.

### 17.5 Compression Type vs. Form Type

The distinction between AIFF (FORM = AIFF, no compression) and AIFC (FORM = AIFC, may have compression) is sometimes confused. A file with FORM = "AIFC" but compression type "NONE" is still an AIFC file. A file with FORM = "AIFF" and no FVER chunk is AIFF. Do not use the presence or absence of compression to determine the form type.

### 17.6 Non-Standard Extensions

Many applications embed non-standard chunks in AIFF files. Common extensions include:
- `ID3 ` (0x49443320): ID3v2 tags (used by iTunes and others).
- `CHAN` (0x4348414E): Channel layout information (Apple extension).
- Private chunks starting with a registered 4-character creator ID (e.g., `appl`, `musi`).

A robust reader should skip unknown chunks rather than rejecting the file. A robust writer should avoid creating non-standard chunks unless required for compatibility with a specific tool.

### 17.7 Marker Time vs. Frame Position

Markers store positions as sample frame numbers, not as time in seconds. When exposing markers to users or converting between formats, always divide by the sample rate to get time: `time_seconds = position / sample_rate`. Relying on the raw position number without accounting for sample rate will produce incorrect times for files with non-44.1 kHz sample rates.

### 17.8 INST Chunk Loop Validation

When implementing loop playback from AIFF INST chunks, validate that referenced marker IDs exist before using them. If the INST chunk references marker ID 5 for the loop start, but marker ID 5 does not exist in the MARK chunk, the loop configuration is invalid and should be ignored or reported as an error.

### 17.9 SSND Offset Field

The `blockSize` field in the SSND header (misleadingly named "offset to sound data") is almost always zero. Some very old files may have non-zero values. If non-zero, skip exactly that many bytes before reading audio data. The offset bytes, if present, are not part of the audio data and should not contribute to the sample count.

### 17.10 numSampleFrames as uint32

The `numSampleFrames` field is a 32-bit signed integer, limiting AIFF files to 2,147,483,647 sample frames. For stereo 16-bit 44100 Hz audio, this translates to a maximum duration of approximately 13.5 hours (2,147,483,647 / 44100 / 2 ≈ 24,341 seconds ≈ 6.76 hours per channel). For most practical purposes, this is not a limitation, but high-resolution long-form audio (e.g., 8-channel 192 kHz 24-bit recordings) could approach this limit in files longer than a few hours.

---

## 18. DETAILED IMPLEMENTATION CODE EXAMPLES

### 18.1 Python AIFF Header Parser

The following implementation demonstrates the core reading logic for AIFF headers. This is production-quality code suitable for integration into conversion pipelines.

```python
import struct
import io
from dataclasses import dataclass
from typing import BinaryIO, Optional, List

@dataclass
class AIFFCommonChunk:
    num_channels: int
    num_sample_frames: int
    sample_size: int
    sample_rate: float
    # AIFC-specific
    is_aifc: bool = False
    compression_type: Optional[bytes] = None
    compression_name: Optional[str] = None

@dataclass
class AIFFMarker:
    marker_id: int
    position: int
    name: str

@dataclass
class AIFFHeader:
    form_type: str
    is_aifc: bool
    common: Optional[AIFFCommonChunk] = None
    markers: List[AIFFMarker] = []
    name: Optional[str] = None
    author: Optional[str] = None
    copyright: Optional[str] = None
    annotations: List[str] = []
    instrument: Optional[bytes] = None

def read_uint16_be(f: BinaryIO) -> int:
    return struct.unpack('>H', f.read(2))[0]

def read_int16_be(f: BinaryIO) -> int:
    return struct.unpack('>h', f.read(2))[0]

def read_uint32_be(f: BinaryIO) -> int:
    return struct.unpack('>I', f.read(4))[0]

def read_int32_be(f: BinaryIO) -> int:
    return struct.unpack('>i', f.read(4))[0]

def read_extended_float(f: BinaryIO) -> float:
    """
    Decode the 80-bit IEEE 754 extended precision float used for AIFF sample rates.
    Layout: 1 sign+exponent byte, 1 exponent byte, 8 mantissa bytes (all big-endian).
    """
    exp_sign_byte = f.read(2)
    mantissa_bytes = f.read(8)
    
    sign = (exp_sign_byte[1] >> 7) & 1
    exponent = ((exp_sign_byte[0] << 1) | ((exp_sign_byte[1] >> 7) & 1)) & 0x7FFF
    
    # Reconstruct mantissa as a 64-bit integer (includes explicit integer bit)
    mantissa = int.from_bytes(mantissa_bytes, byteorder='big')
    
    if exponent == 0 and mantissa == 0:
        return 0.0
    if exponent == 0x7FFF:
        return float('nan') if mantissa != 0 else float('inf') * (-1 if sign else 1)
    
    true_exp = exponent - 16383
    significand = mantissa / (1 << 63)
    result = (1.0 + significand) * (2.0 ** true_exp)
    return -result if sign else result

def read_pascal_string(f: BinaryIO) -> str:
    """Read a Pascal-style string: length byte followed by characters."""
    length = read_uint16_be(f)  # Note: AIFF uses int16 for string length
    data = f.read(length)
    if length % 2 == 0:
        f.read(1)  # padding
    return data.decode('latin-1', errors='replace')

def read_aiff_header(stream: BinaryIO) -> AIFFHeader:
    """Parse AIFF/AIFC file header and metadata chunks."""
    magic = stream.read(4)
    if magic != b'FORM':
        raise ValueError(f"Invalid AIFF magic: {magic}")
    
    form_size = read_uint32_be(stream)
    form_type = stream.read(4).decode('ascii')
    
    is_aifc = form_type == 'AIFC'
    header = AIFFHeader(form_type=form_type, is_aifc=is_aifc)
    
    while stream.tell() < form_size + 12:
        chunk_id = stream.read(4)
        if len(chunk_id) < 4:
            break
        
        chunk_size = read_uint32_be(stream)
        chunk_start = stream.tell()
        
        if chunk_id == b'COMM':
            num_channels = read_int16_be(stream)
            num_sample_frames = read_int32_be(stream)
            sample_size = read_int16_be(stream)
            sample_rate = read_extended_float(stream)
            
            compression_type = None
            compression_name = None
            if is_aifc:
                compression_type = stream.read(4)
                name_length = ord(stream.read(1))
                compression_name = stream.read(name_length).decode('latin-1')
                if name_length % 2 == 0:
                    stream.read(1)  # padding
            
            header.common = AIFFCommonChunk(
                num_channels=num_channels,
                num_sample_frames=num_sample_frames,
                sample_size=sample_size,
                sample_rate=sample_rate,
                is_aifc=is_aifc,
                compression_type=compression_type,
                compression_name=compression_name
            )
        
        elif chunk_id == b'FVER':
            version = read_uint32_be(stream)
        
        elif chunk_id == b'SSND':
            offset = read_uint32_be(stream)
            block_align = read_uint32_be(stream)
        
        elif chunk_id == b'MARK':
            marker_count = read_int16_be(stream)
            for _ in range(marker_count):
                marker_id = read_int16_be(stream)
                position = read_int32_be(stream)
                name = read_pascal_string(stream)
                header.markers.append(AIFFMarker(marker_id, position, name))
        
        elif chunk_id == b'NAME':
            header.name = read_pascal_string(stream)
        
        elif chunk_id == b'AUTHOR':
            header.author = read_pascal_string(stream)
        
        elif chunk_id == b'COPY':
            header.copyright = read_pascal_string(stream)
        
        elif chunk_id == b'ANNO':
            header.annotations.append(read_pascal_string(stream))
        
        elif chunk_id == b'INST':
            header.instrument = stream.read(20)
        
        # Skip to next chunk, accounting for padding
        consumed = (stream.tell() - chunk_start)
        padding = (chunk_size - consumed + 1) & ~1
        stream.seek(chunk_start + chunk_size)
        if chunk_size % 2 == 1:
            stream.read(1)
    
    return header
```

### 18.2 Python AIFF Sample Reader

```python
import struct
import io
import math

class AIFFSampleReader:
    """
    Read audio samples from an AIFF/AIFC file.
    Handles big-endian byte order, 8-bit unsigned, and multi-channel interleaving.
    """
    
    def __init__(self, filepath: str):
        self.filepath = filepath
        with open(filepath, 'rb') as f:
            from .parser import read_aiff_header
            self.header = read_aiff_header(f)
            
            if self.header.common is None:
                raise ValueError("AIFF file has no COMM chunk")
            
            # Find SSND chunk
            f.seek(0)
            while True:
                pos = f.tell()
                chunk_id = f.read(4)
                if not chunk_id or chunk_id == b'\x00\x00\x00\x00':
                    break
                chunk_size = struct.unpack('>I', f.read(4))[0]
                if chunk_id == b'SSND':
                    self.ssnd_offset = struct.unpack('>I', f.read(4))[0]
                    self.block_align = struct.unpack('>I', f.read(4))[0]
                    self.data_start = f.tell() + self.ssnd_offset
                    self.data_size = chunk_size - 8
                    break
                f.seek(pos + 8 + chunk_size + (chunk_size % 2))
    
    @property
    def sample_rate(self) -> float:
        return self.header.common.sample_rate
    
    @property
    def channels(self) -> int:
        return self.header.common.num_channels
    
    @property
    def bit_depth(self) -> int:
        return self.header.common.sample_size
    
    @property
    def duration_seconds(self) -> float:
        return self.header.common.num_sample_frames / self.sample_rate
    
    def bytes_per_sample(self) -> int:
        return (self.header.common.sample_size + 7) // 8
    
    def frame_size(self) -> int:
        return self.channels * self.bytes_per_sample()
    
    def read_frames(self, count: int, offset: int = 0) -> List[List[int]]:
        """Read 'count' sample frames, returning a list of channel samples."""
        with open(self.filepath, 'rb') as f:
            f.seek(self.data_start + offset * self.frame_size())
            frames = []
            for _ in range(count):
                frame = []
                for ch in range(self.channels):
                    sample_bytes = f.read(self.bytes_per_sample())
                    sample = self._bytes_to_sample(sample_bytes)
                    frame.append(sample)
                frames.append(frame)
            return frames
    
    def _bytes_to_sample(self, data: bytes) -> int:
        """Convert big-endian bytes to signed integer sample."""
        if self.bit_depth == 8:
            # 8-bit AIFF is unsigned, silence at 128
            unsigned = data[0]
            signed = unsigned - 128
            return signed
        else:
            # Multi-byte signed big-endian
            value = int.from_bytes(data, byteorder='big', signed=False)
            bits = self.bit_depth
            # Sign-extend if the high bit is set
            if bits < 8 * len(data):
                mask = 1 << bits
                if value >= mask // 2:
                    value -= mask
            return value
    
    def _sample_to_bytes(self, sample: int) -> bytes:
        """Convert signed integer sample to big-endian bytes."""
        if self.bit_depth == 8:
            unsigned = max(0, min(255, sample + 128))
            return bytes([unsigned])
        else:
            bits = self.bit_depth
            mask = 1 << bits
            if sample < 0:
                sample += mask
            byte_count = self.bytes_per_sample()
            return sample.to_bytes(byte_count, byteorder='big')
```

### 18.3 Python AIFF Writer

```python
import struct
import io
from typing import List, Optional

class AIFFWriter:
    """
    Write audio samples to an AIFF/AIFC file.
    Produces standard AIFF with big-endian PCM, or AIFC with compression.
    """
    
    def __init__(
        self,
        filepath: str,
        sample_rate: float,
        channels: int,
        bit_depth: int,
        is_aifc: bool = False,
        compression_type: Optional[bytes] = None,
        compression_name: Optional[str] = None,
    ):
        self.filepath = filepath
        self.sample_rate = sample_rate
        self.channels = channels
        self.bit_depth = bit_depth
        self.is_aifc = is_aifc
        self.compression_type = compression_type or b'NONE'
        self.compression_name = compression_name or 'not specified'
        self.frames: List[List[int]] = []
        self.markers: List[tuple] = []
        self.name: Optional[str] = None
        self.author: Optional[str] = None
        self.copyright: Optional[str] = None
        self.annotations: List[str] = []
    
    def write_frame(self, frame: List[int]):
        """Append a single sample frame (one sample per channel)."""
        if len(frame) != self.channels:
            raise ValueError(
                f"Frame has {len(frame)} channels, expected {self.channels}"
            )
        self.frames.append(frame)
    
    def write_frames(self, frames: List[List[int]]):
        """Append multiple sample frames."""
        for frame in frames:
            self.write_frame(frame)
    
    def add_marker(self, marker_id: int, position: int, name: str):
        """Add a named marker at a sample frame position."""
        self.markers.append((marker_id, position, name))
    
    def encode_extended_float(self, value: float) -> bytes:
        """Encode a float as an 80-bit IEEE 754 extended precision value."""
        if value == 0.0:
            return b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        
        sign = 1 if value < 0 else 0
        value = abs(value)
        
        # Find the exponent
        exp = math.floor(math.log2(value))
        significand = value / (2.0 ** exp)
        
        biased_exp = exp + 16383
        
        # Convert to 80-bit format
        # The extended float has: 1 sign+exponent byte, 1 exponent byte, 8 mantissa bytes
        # Mantissa: 1 explicit integer bit + 63 fraction bits
        
        # Pack the mantissa (64 bits = 8 bytes)
        scaled = significand - 1.0  # Remove the implicit 1
        mantissa_int = round(scaled * (1 << 63))
        mantissa_bytes = mantissa_int.to_bytes(8, byteorder='big')
        
        # Pack exponent (15 bits) and sign
        exp_msb = (biased_exp >> 1) & 0xFF
        exp_lsb_sign = ((biased_exp & 1) << 7) | (sign << 7)
        
        return bytes([exp_msb, exp_lsb_sign]) + mantissa_bytes
    
    def _write_pascal_string(self, f: io.RawIOBase, text: str):
        """Write a Pascal-style string to the output."""
        data = text.encode('latin-1')
        f.write(struct.pack('>H', len(data)))
        f.write(data)
        if len(data) % 2 == 0:
            f.write(b'\x00')
    
    def _write_text_chunk(self, f: io.RawIOBase, chunk_id: bytes, text: str):
        """Write a generic text chunk (NAME, AUTHOR, COPYRIGHT, ANNO)."""
        f.write(chunk_id)
        size_pos = f.tell()
        f.write(struct.pack('>I', 0))  # placeholder
        start = f.tell()
        self._write_pascal_string(f, text)
        end = f.tell()
        size = end - start
        f.seek(size_pos)
        f.write(struct.pack('>I', size))
        f.seek(end)
        if size % 2 == 1:
            f.write(b'\x00')
    
    def save(self):
        """Write the complete AIFF/AIFC file."""
        num_frames = len(self.frames)
        bytes_per_sample = (self.bit_depth + 7) // 8
        frame_size = self.channels * bytes_per_sample
        audio_data_size = num_frames * frame_size
        
        with open(self.filepath, 'wb') as f:
            # FORM header
            f.write(b'FORM')
            form_size_pos = f.tell()
            f.write(struct.pack('>I', 0))  # placeholder
            f.write(b'AIFC' if self.is_aifc else b'AIFF')
            
            if self.is_aifc:
                # FVER chunk
                f.write(b'FVER')
                f.write(struct.pack('>I', 4))
                f.write(struct.pack('>I', 173857))  # AIFC version 1
            
            # COMM chunk
            f.write(b'COMM')
            comm_data_start = f.tell()
            f.write(struct.pack('>I', 0))  # placeholder size
            
            f.write(struct.pack('>h', self.channels))
            f.write(struct.pack('>i', num_frames))
            f.write(struct.pack('>h', self.bit_depth))
            f.write(self.encode_extended_float(self.sample_rate))
            
            if self.is_aifc:
                f.write(self.compression_type)
                self._write_pascal_string(f, self.compression_name)
            
            comm_data_end = f.tell()
            comm_size = comm_data_end - comm_data_start - 4
            f.seek(comm_data_start)
            f.write(struct.pack('>I', comm_size))
            f.seek(comm_data_end)
            
            # SSND chunk
            f.write(b'SSND')
            f.write(struct.pack('>I', audio_data_size + 8))
            f.write(struct.pack('>I', 0))   # offset
            f.write(struct.pack('>I', frame_size))  # block align
            
            for frame in self.frames:
                for ch in range(self.channels):
                    sample = frame[ch]
                    if self.bit_depth == 8:
                        unsigned = max(0, min(255, sample + 128))
                        f.write(bytes([unsigned]))
                    else:
                        bits = self.bit_depth
                        mask = 1 << bits
                        if sample < 0:
                            sample += mask
                        byte_count = bytes_per_sample
                        f.write(sample.to_bytes(byte_count, byteorder='big'))
            
            if audio_data_size % 2 == 1:
                f.write(b'\x00')
            
            # MARK chunk
            if self.markers:
                f.write(b'MARK')
                mark_start = f.tell()
                f.write(struct.pack('>I', 0))
                f.write(struct.pack('>h', len(self.markers)))
                for marker_id, position, name in self.markers:
                    f.write(struct.pack('>h', marker_id))
                    f.write(struct.pack('>i', position))
                    self._write_pascal_string(f, name)
                mark_end = f.tell()
                f.seek(mark_start)
                f.write(struct.pack('>I', mark_end - mark_start - 4))
                f.seek(mark_end)
            
            # Text chunks
            if self.name:
                self._write_text_chunk(f, b'NAME', self.name)
            if self.author:
                self._write_text_chunk(f, b'AUTH', self.author)
            if self.copyright:
                self._write_text_chunk(f, b'COPY', self.copyright)
            for ann in self.annotations:
                self._write_text_chunk(f, b'ANNO', ann)
            
            # Compute and write FORM size
            file_end = f.tell()
            form_size = file_end - 12
            f.seek(form_size_pos)
            f.write(struct.pack('>I', form_size))
```

---

## 19. AIFC COMPRESSION TYPE DEEP DIVE

### 19.1 G.711 A-Law Encoding Algorithm

The ITU-T G.711 A-law compression algorithm maps 16-bit signed linear PCM samples to 8-bit A-law codewords. The algorithm uses a piecewise linear approximation of the logarithmic A-law curve.

**A-law encoding rules:**
1. The sign bit is preserved: positive and negative values are encoded symmetrically.
2. The magnitude is compressed using a 13-segment (15-segment including the origin) piecewise linear approximation.
3. The 16-bit input is first converted to 14-bit magnitude (by inverting the MSB after the sign bit).
4. The 14-bit magnitude is then compressed to 4-bit segment index and 3-bit step index.
5. The final 8-bit codeword is: `sign_bit ^ segment_code ^ step_code`, with the MSB inverted for A-law (this creates the characteristic A-law asymmetry where even codewords have inverted MSB).

The A-law provides a signal-to-quantization-noise ratio of approximately 37 dB across its dynamic range, with better resolution at low amplitudes (near silence) where human hearing is most sensitive.

**A-law decode formula (simplified):**
```python
def alaw_decode(code: int) -> int:
    """Decode G.711 A-law 8-bit codeword to 16-bit signed linear PCM."""
    # Invert MSB (A-law characteristic)
    code ^= 0x55
    sign = (code >> 7) & 1
    exp = (code >> 4) & 0x07
    step = code & 0x0F
    # Reconstructed magnitude
    magnitude = (exp * 16) + step + (16 if exp > 0 else 0)
    # Scale by segment size and add segment bias
    if exp == 0:
        magnitude = (magnitude * 2) + 1
    else:
        magnitude = (magnitude + 8) * (1 << (exp - 1))
    return -magnitude if sign else magnitude
```

### 19.2 G.711 μ-Law Encoding Algorithm

The μ-law (mu-law) compression algorithm uses a slightly different curve from A-law. The encoding uses the formula:

```
output_code = sign(m) × ln(1 + μ × |m| / m_max) / ln(1 + μ)
```

Where μ = 255, m_max = 32767 (for 16-bit input), and the result is quantized to 8 bits. The μ-law encoding provides approximately 14 bits of effective dynamic range in the compressed representation.

**μ-law decode formula:**
```python
def ulaw_decode(code: int) -> int:
    """Decode G.711 μ-law 8-bit codeword to 16-bit signed linear PCM."""
    BIAS = 0x84  # 132 for μ-law
    code = ~code  # Invert all bits
    sign = (code >> 7) & 1
    exp = (code >> 4) & 0x07
    step = code & 0x0F
    # Reconstructed value
    magnitude = (exp * 16 + step) * 4 + BIAS
    return -magnitude if sign else magnitude
```

### 19.3 IMA ADPCM Block Encoding

IMA ADPCM (Adaptive Differential Pulse Code Modulation) achieves 4:1 compression by storing only the difference between consecutive samples, quantized to 4 bits.

**IMA ADPCM encoder state:**
```python
class IMAEncoderState:
    def __init__(self):
        self.predictor: int = 0       # 16-bit signed
        self.index: int = 0           # Step size table index (0–88)
        self.prev_input: int = 0      # Previous input sample (16-bit)

IMA_STEP_TABLE = [
    7, 8, 9, 10, 11, 12, 13, 14,
    16, 17, 19, 21, 23, 25, 28, 31,
    34, 37, 41, 45, 50, 55, 60, 66,
    73, 80, 88, 97, 107, 118, 130, 143,
    157, 173, 190, 209, 230, 253, 279, 307,
    337, 371, 408, 449, 494, 544, 598, 658,
    724, 796, 876, 963, 1060, 1166, 1282, 1411,
    1552, 1707, 1878, 2066, 2272, 2499, 2749, 3024,
    3327, 3660, 4026, 4428, 4871, 5358, 5894, 6484,
    7132, 7845, 8630, 9493, 10442, 11487, 12635, 13899,
    15289, 16818, 18500, 20350, 22385, 24623, 27086, 29794,
    32767
]

def imadpcm_encode_sample(state: IMAEncoderState, sample: int) -> tuple:
    """
    Encode a single 16-bit PCM sample as an IMA ADPCM 4-bit code.
    Returns (code, updated_state).
    """
    diff = sample - state.predictor
    step = IMA_STEP_TABLE[state.index]
    
    # Quantize: find the closest step multiple
    if diff < 0:
        code = 0 if diff > -step else (4 if diff < -step*2 else 8)
    else:
        code = 2 if diff < step else (6 if diff < step*2 else 10)
    
    # Compute quantized difference
    if diff < 0:
        if code >= 8:
            code = 16 - code
        delta = -(code * step) // 2
    else:
        delta = (code * step) // 2
    
    # Update predictor
    state.predictor += delta
    state.predictor = max(-32768, min(32767, state.predictor))
    
    # Update step size index
    state.index += [0, -1, -1, -1, -1, 2, 4, 6, 8][code & 0x07]
    state.index = max(0, min(88, state.index))
    
    # The 4-bit code is stored with some transformations
    stored_code = (code & 0x0F)
    
    return stored_code, state
```

### 19.4 Audio Duration Calculation

Correctly computing audio duration from AIFF header information is essential for validation and playback:

```python
def compute_duration_seconds(
    num_sample_frames: int,
    sample_rate: float,
) -> float:
    """Calculate audio duration in seconds from AIFF parameters."""
    if sample_rate <= 0:
        raise ValueError(f"Invalid sample rate: {sample_rate}")
    return num_sample_frames / sample_rate

def validate_audio_data_size(
    declared_frames: int,
    channels: int,
    bit_depth: int,
    actual_data_size: int,
) -> bool:
    """Validate that the declared audio parameters match the actual data size."""
    bytes_per_sample = (bit_depth + 7) // 8
    expected = declared_frames * channels * bytes_per_sample
    # Allow for minor padding discrepancies
    return abs(actual_data_size - expected) <= channels
```

---

## 20. INTEROPERABILITY AND EDGE CASES

### 20.1 Apple Core Audio AIFF Handling

macOS Core Audio handles AIFF/AIFC through the ExtAudioFile API. Core Audio enforces a strict interpretation of the AIFF specification but adds its own extensions. Notably, Core Audio supports reading and writing AIFF files with embedded ID3v2 tags stored in a private `ID3 ` chunk (note the trailing space in the chunk ID, which is ASCII 0x20). This non-standard extension is widely used by applications on macOS that need to carry ID3 metadata in AIFF containers.

When reading AIFF files produced by macOS applications, check for the presence of an `ID3 ` chunk after the SSND chunk. The chunk structure follows the standard IFF layout with chunk ID `ID3 ` and data containing the full ID3v2 tag (header + frames).

### 20.2 SoX AIFF Handling

SoX (Sound eXchange) supports AIFF reading and writing. When converting to AIFF with SoX, it defaults to 16-bit big-endian PCM. Key SoX options:

```bash
# Convert to AIFF (16-bit, 44100 Hz)
sox input.wav output.aiff

# Convert with specific parameters
sox input.wav -b 24 -r 96000 -c 6 output.aiff

# Convert AIFF to WAV with automatic endianness handling
sox input.aiff output.wav

# Convert AIFF with markers preserved
sox input.aiff output.aiff

# Read AIFF metadata
soxi input.aiff
```

SoX stores markers in a format compatible with the AIFF MARK chunk but may not preserve all INST chunk parameters. When using SoX in conversion pipelines, verify that markers and instrument data survive the round-trip.

### 20.3 FFprobe AIFF Metadata

FFprobe can extract comprehensive AIFF metadata:

```bash
# Show detailed format information
ffprobe -show_format -show_streams input.aiff

# Show packet/frame-level details
ffprobe -show_frames input.aiff

# Show all metadata tags (including ID3 in AIFF)
ffprobe -show_private_data input.aiff

# Extract COMM chunk parameters as JSON
ffprobe -v quiet -print_format json -show_format input.aiff
```

### 20.4 Common Non-Standard Extensions

Several non-standard chunk types are encountered in real AIFF files:

| Chunk ID   | Bytes (ASCII) | Description                                     | Used By        |
|------------|--------------|------------------------------------------------|----------------|
| `ID3 `     | 0x49443320  | ID3v2 tag (with trailing space)               | iTunes, macOS  |
| `CHAN`     | 0x4348414E  | Channel layout (macOS extension)              | Apple apps     |
| `MATL`     | 0x4D41544C  | Material tag (non-standard)                   | Rare           |
| `appl`     | 0x6170706C | Apple-specific application data                | Various        |

The `CHAN` chunk specifies channel ordering using the standard Core Audio channel layout tags (e.g., kChannelLayoutTag_Stereo, kChannelLayoutTag_5_1, etc.). This is Apple-specific and not portable to other platforms.

### 20.5 File Identification

AIFF/AIFC files can be identified by their magic bytes:

```
Offset 0: "FORM" (0x46 0x4F 0x52 0x4D)
Offset 8: "AIFF" (0x41 0x49 0x46 0x46) — uncompressed
          "AIFC" (0x41 0x49 0x46 0x43) — compressed
```

The presence of the `FVER` chunk confirms AIFC. The absence of `FVER` confirms AIFF. The FORM type alone is sufficient for file identification in most cases.

### 20.6 Streamed Reading (No Seeking)

For networked AIFF streams or sequential access without seeking, implement a two-pass strategy:

**Pass 1 (header discovery):**
- Read chunks sequentially from the start.
- Record the offset of the SSND chunk data.
- Extract all metadata and format parameters.
- Stop after reading the SSND chunk header (don't read all audio data yet).

**Pass 2 (data reading):**
- Seek to the SSND data offset.
- Read audio data in chunks.
- If seeking is needed during playback, maintain a frame index or seek table.

For truly non-seekable streams, the header information must be received first (over a control channel or as a preamble), and the SSND chunk must be streamed sequentially.

### 20.7 Audio Corruption Scenarios

Several common corruption patterns affect AIFF files:

**Truncated SSND chunk:** If the file is truncated mid-audio, the numSampleFrames field will not match the actual available data. Implement best-effort reading: read as many complete frames as possible from the available bytes.

**Byte-swapped audio:** Some tools accidentally write little-endian PCM data into an AIFF file. The audio will sound like severe distortion or noise. Detecting this requires comparing the byte order against the file's declared format — if the file claims to be AIFF (big-endian) but the audio data has a byte-swapped pattern (e.g., each 16-bit sample has its bytes reversed), the data is byte-swapped.

**Incorrect numSampleFrames:** If numSampleFrames is too small, trailing silence is lost. If too large, the reader may attempt to read beyond the file or read garbage bytes. Always validate: `computed_data_size = numSampleFrames × channels × bytes_per_sample` against the actual SSND data size.

**Corrupted sample rate extended float:** If the 80-bit sample rate field is corrupted, the decoded value may be NaN, infinity, or a large garbage number. Validate: `0 < sample_rate < 1_000_000` (reasonable range for audio). Reject or use a fallback rate with a warning if validation fails.

**Duplicate chunks:** Having multiple COMM or SSND chunks is non-standard. A robust reader should use the first valid instance and ignore or warn about duplicates.

### 20.8 Cross-Platform Byte Order Verification

Testing AIFF byte order handling requires comparing the same audio encoded on big-endian and little-endian systems:

```python
def verify_byte_order_consistency(filepath: str, expected_channels: int, expected_bit_depth: int):
    """Verify that a file's byte order produces consistent results on any platform."""
    with open(filepath, 'rb') as f:
        # Read first few samples
        reader = AIFFSampleReader(filepath)
        frames = reader.read_frames(10)
        
        # Verify no sample values are impossible
        for frame in frames:
            for ch, sample in enumerate(frame):
                if expected_bit_depth == 8:
                    if not (-128 <= sample <= 127):
                        print(f"Warning: 8-bit sample out of range: {sample}")
                else:
                    max_val = (1 << expected_bit_depth) - 1
                    if not (-max_val <= sample <= max_val):
                        print(f"Warning: {expected_bit_depth}-bit sample out of range: {sample}")
        
        # Verify sample rate is reasonable
        if not (1000 <= reader.sample_rate <= 1000000):
            raise ValueError(f"Suspicious sample rate: {reader.sample_rate}")
```

---

## 21. COMPARISON WITH RELATED FORMATS

### 21.1 AIFF vs. WAV

| Aspect                  | AIFF                          | WAV                           |
|-------------------------|-------------------------------|-------------------------------|
| Byte order              | Big-endian                    | Little-endian                 |
| Origin platform         | Macintosh, NeXT, SGI          | IBM PC (Intel)                |
| Max file size           | ~4 GB (chunk size limit)      | ~4 GB (RIFF limit, RF64 extends)|
| Built-in metadata       | Basic (NAME, AUTHOR, etc.)    | Basic (LIST INFO), extensible  |
| Cover art              | No standard mechanism         | Via LIST/art                  |
| Compression            | AIFC extension (various)      | Extensible via format tags    |
| Sample rate format     | 80-bit extended float         | uint32 (integer Hz)           |
| Standard sample rates  | Any (floating-point)         | Integer Hz values            |
| 8-bit encoding         | Unsigned (silence at 128)    | Unsigned (silence at 128)     |
| Loop markers           | MARK + INST chunks            | cue + plst chunks             |
| Float audio            | Via AIFC (fl32, fl64)        | Via WAVE_FORMAT_IEEE_FLOAT    |
| Industry use            | Professional Mac/Unix audio   | General Windows audio         |

### 21.2 AIFF vs. FLAC

| Aspect                  | AIFF                          | FLAC                          |
|-------------------------|-------------------------------|-------------------------------|
| Compression             | Uncompressed (or lossy AIFC) | Lossless, block-based         |
| Byte order              | Big-endian                    | Little-endian (stream)        |
| Standard sample rates   | Any floating-point            | Integer Hz (up to 655350 Hz)  |
| Bit depths              | 1–32 bits                     | 4–32 bits (native)            |
| Metadata               | Basic chunks                  | Vorbis Comments (extensible)  |
| Streaming support       | Chunks, no native seek table | Frame-based, seekable         |
| Cover art              | No standard                   | METADATA_BLOCK_PICTURE        |
| 8-bit handling         | Unsigned                      | Converted to signed internally|
| Float audio            | Via AIFC fl32/fl64           | Not natively supported         |
| Cross-platform         | Universal                     | Universal                     |
| File size (typical)    | Uncompressed                  | ~50–70% of AIFF size          |

### 21.3 AIFF vs. CAF (Core Audio Format)

| Aspect                  | AIFF                          | CAF                           |
|-------------------------|-------------------------------|-------------------------------|
| File size              | ~4 GB limit                  | Virtually unlimited           |
| Metadata               | Basic chunks                 | Dictionary-based             |
| Byte order             | Big-endian                   | Configurable (usually LE)    |
| Sample rate            | 80-bit extended float         | 64-bit float                  |
| Chunk structure        | Fixed (IFF)                  | 64-bit offsets                |
| Channel layout         | No standard                  | AudioChannelLayout blob      |
| Industry use           | Legacy interchange           | Modern macOS professional     |

---

## 22. PERFORMANCE CONSIDERATIONS

### 22.1 Memory-Mapped I/O

For large AIFF files (hundreds of megabytes or gigabytes), memory-mapped I/O provides efficient random access without loading the entire file into memory:

```python
import mmap
import os

def memory_map_aiff_samples(filepath: str, max_frames: int = 1000000):
    """Memory-map the audio data region of an AIFF file for fast access."""
    with open(filepath, 'rb') as f:
        # Find SSND data start
        header = read_aiff_header(open(filepath, 'rb'))
        # ... (find data_start as before)
        
        fd = os.open(filepath, os.O_RDONLY)
        mm = mmap.mmap(fd, length=data_size, access=mmap.ACCESS_READ, offset=data_start)
        
        # Now reads are done through the memory map
        # Each frame read: mm.read(frame_size)
        
        yield mm  # Caller uses and closes
        
        mm.close()
        os.close(fd)
```

### 22.2 Batch Conversion Pipeline

When processing large numbers of AIFF files, use batch processing with multiprocessing to maximize throughput:

```python
import concurrent.futures
from pathlib import Path

def batch_convert_aiff(
    input_files: list,
    output_dir: Path,
    target_format: str,
    max_workers: int = 4,
) -> list:
    """Convert multiple AIFF files in parallel."""
    results = []
    
    with concurrent.futures.ThreadPoolExecutor(max_workers=max_workers) as executor:
        futures = {
            executor.submit(
                convert_single_file,
                infile,
                output_dir / f"{infile.stem}.{target_format}",
            ): infile
            for infile in input_files
        }
        
        for future in concurrent.futures.as_completed(futures):
            infile = futures[future]
            try:
                result = future.result()
                results.append(result)
            except Exception as e:
                results.append({'file': infile, 'error': str(e)})
    
    return results
```

### 22.3 Streaming Decode

For real-time or low-latency applications, implement a streaming decoder that reads and processes audio in chunks without buffering the entire file:

```python
class AIFFStreamingDecoder:
    """Decode AIFF files in a streaming fashion without full-file buffering."""
    
    def __init__(self, filepath: str, chunk_frames: int = 8192):
        self.filepath = filepath
        self.chunk_frames = chunk_frames
        
        # Parse header only
        self.header = read_aiff_header(open(filepath, 'rb'))
        self.frame_size = self.header.common.num_channels * \
                          ((self.header.common.sample_size + 7) // 8)
        self.total_frames = self.header.common.num_sample_frames
        self.samples_read = 0
        
        # Open file for streaming reads
        self.file = open(filepath, 'rb')
        self.file.seek(self.data_start)
    
    def read_chunk(self) -> Optional[List[List[int]]]:
        """Read the next chunk of frames, or None if at EOF."""
        if self.samples_read >= self.total_frames:
            return None
        
        frames = []
        frames_to_read = min(
            self.chunk_frames,
            self.total_frames - self.samples_read
        )
        
        for _ in range(frames_to_read):
            frame = []
            for _ in range(self.header.common.num_channels):
                data = self.file.read(self.bytes_per_sample)
                sample = self._bytes_to_sample(data)
                frame.append(sample)
            frames.append(frame)
        
        self.samples_read += frames_to_read
        return frames
    
    def close(self):
        self.file.close()
    
    def __enter__(self):
        return self
    
    def __exit__(self, *args):
        self.close()
```

---

*Document version: 1.0*
*Target audience: Audio codec developers building AIFF/AIFC conversion pipelines*
*Related specifications: Apple AIFF Specification (1991), EA IFF 85, ITU-T G.711, ITU-T G.722, ITU-T G.726, IMA ADPCM Recommended Practice*

