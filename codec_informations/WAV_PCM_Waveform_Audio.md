# WAV/PCM Waveform Audio — Deep Technical Reference

> **Category:** Lossless Audio (uncompressed PCM container)
> **File Extensions:** .wav, .wave, .bwf
> **MIME Types:** audio/wave, audio/wav, audio/x-wav, audio/vnd.wave
> **Standardization Body:** Microsoft + EBU (BWF extension)
> **Specification Document:** ITU-R BS.2088 (BWF), Microsoft multimedia standards
> **Patent Status:** Patent-free
> **License:** Public domain / Microsoft contributed to ITU

---

## 1. HISTORICAL CONTEXT & ORIGIN

WAV (Waveform Audio File Format) was developed by Microsoft and IBM for Windows 3.1 audio support in 1991, as a specific application of the RIFF (Resource Interchange File Format) container. The format was designed to provide a standardized container for PCM audio on the Windows platform, replacing the various proprietary formats that existed at the time. IBM contributed the IBM WAVE file format specification, and Microsoft implemented it as part of the Multimedia Extensions for Windows 3.1. The RIFF format was chosen as the foundation because it provided a flexible, chunked structure that could accommodate future extensions without breaking compatibility.

The European Broadcasting Union (EBU) adopted WAV as the basis for the **Broadcast Wave Format (BWF)** in 1997, documented in EBU Tech 3285 and later ETSI TS 102 821. BWF was specifically designed to meet the archival and exchange requirements of broadcast organizations. The original BWF Version 0 (1997) introduced the `bext` chunk for carryover of essential broadcast metadata. BWF Version 1 (2001) added UMID (Unique Material Identifier) support using 64 bytes from the reserved portion of the bext chunk. BWF Version 2 (2011) added loudness metadata compliant with EBU R 128, including integrated loudness, loudness range, and maximum true peak level.

The **RF64 format** (EBU Tech 3306, 2009) extended the WAV specification to support files larger than 4GB, which was a critical limitation for professional broadcast and post-production applications handling long recordings or high-resolution audio. RF64 uses a `ds64` chunk to store 64-bit chunk sizes while maintaining backward compatibility with standard WAV players. **BW64** (ITU-R BS.2088, 2015) further extended the format to include ADM (Audio Definition Model) support for object-based audio, representing a significant step toward immersive audio workflows in broadcast.

Industry adoption has been extensive across multiple domains: broadcast and television production (BWF is mandatory in many EBU member organizations), film and video post-production (DAW software universally supports WAV import/export), game audio pipelines (simple to implement, zero decode overhead), sound design and sample libraries (Samplitude, Sound Forge, Pro Tools all use WAV as interchange), and general-purpose audio storage. The format remains the canonical reference format for audio quality comparisons because it imposes no lossy compression.

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

WAV is a **chunked container format** built on the RIFF (Resource Interchange File Format) specification. Every element in a WAV file is a chunk: a self-describing block with a 4-byte identifier, a 4-byte unsigned little-endian size, and the data payload.

The WAV file structure is always:

```
RIFF (WAVE) [
    fmt  (format specification)
    data (audio samples)
    [fact]   (sample count — required for non-PCM formats)
    [cue ]   (cue points)
    [plst]   (playlist/play order)
    [smpl]   (sampler information)
    [labl]   (text labels)
    [LIST]   (metadata — INFO or other sub-chunks)
    [bext]   (BWF extension — broadcast metadata)
    [umid]   (unique material identifier)
    [axml]   (XML metadata container)
    [levl]   (peak envelope chunk)
    [rgad]   (ReplayGain data)
    [iXML]   (BWF XML extension)
    ...
]
```

Key architectural properties:

- **Little-endian byte order** throughout (the RIFX variant uses big-endian but is rarely encountered)
- **Word alignment**: all chunks start on even byte offsets; if a chunk's data size is odd, a single 0x00 padding byte is appended after the data
- **Chunk order**: while the RIFF specification does not mandate ordering, fmt must precede data for valid playback; BWF bext typically appears before data
- **Extensible format**: the `WAVE_FORMAT_EXTENSIBLE` tag (0xFFFE) allows arbitrary format subtypes via a 16-byte GUID, supporting non-PCM audio codecs within the WAV container
- **No built-in seek index**: byte offset for any sample must be calculated manually using `data_offset + sample_number × block_align`

Mandatory chunks for a valid standard PCM WAV file:
1. `RIFF` header chunk with form type `"WAVE"` (at bytes 0–11)
2. `fmt ` chunk (format specification — must appear before data)
3. `data` chunk (raw PCM samples)

Optional chunks that are commonly present:
- `fact` chunk: required for compressed formats, also written by some tools for PCM
- `LIST`/`INFO`: text metadata (title, artist, copyright, etc.)
- `cue `: cue points and markers
- `bext`: BWF broadcast extension

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 Complete File Header Layout

The following is the byte-exact layout of a standard PCM WAV file header (44 bytes for basic format, up to 78 bytes for extensible format):

```
Offset  Size  Type    Field                    Value / Notes
------  ----  -----   -----                    -------------
0x00    4     FOURCC  Chunk ID                 52 49 46 46 ("RIFF")
0x04    4     UINT32  RIFF chunk size          file_size - 8 (LE)
0x08    4     FOURCC  Form Type                57 41 56 45 ("WAVE")
0x0C    4     FOURCC  fmt chunk ID             66 6D 74 20 ("fmt ")
0x10    4     UINT32  fmt chunk size           16 (PCM), 18 (float), or 40 (extensible)
0x14    2     UINT16  Audio Format             0x0001 (PCM), 0x0003 (IEEE float), 0xFFFE (Extensible)
0x16    2     UINT16  nChannels                1–65536
0x18    4     UINT32  nSamplesPerSec           e.g., 44100, 48000, 96000
0x1C    4     UINT32  nAvgBytesPerSec          sample_rate × nBlockAlign
0x20    2     UINT16  nBlockAlign              nChannels × ceil(wBitsPerSample / 8)
0x22    2     UINT16  wBitsPerSample           8, 16, 20, 24, 32, 64
--- Extensible format continuation (fmt size = 40) ---
0x24    2     UINT16  cbSize                   22 (size of extension after this field)
0x26    2     UINT16  wValidBitsPerSample       actual bit depth (e.g., 20 for 20-bit PCM)
0x28    4     UINT32  dwChannelMask             speaker position bitmask
0x2C    16    GUID    SubFormat                format subtype GUID
--- End of fmt chunk ---
0x3C    4     FOURCC  data chunk ID            64 61 74 61 ("data")
0x40    4     UINT32  data chunk size          total_audio_bytes
0x44    N     BYTE[] Audio data                interleaved samples
0x44+N  0/1   BYTE    Padding byte            0x00 if data chunk size is odd
```

**RIFF Header Verification Steps:**
1. Read bytes 0–3: must equal `52 49 46 46` (ASCII "RIFF"). If "RIFX", byte-swap all subsequent integers.
2. Read bytes 8–11: must equal `57 41 56 45` (ASCII "WAVE").
3. Verify RIFF size field: `reported_RIFF_size + 8` should equal `file_size` (within ±1 for padded files).
4. Minimum valid WAV file: 44 bytes (empty data chunk with 16-byte fmt) or 58 bytes (extensible fmt + minimal data).

### 3.2 "fmt " Chunk — Complete Field Reference

The format chunk describes all audio parameters. For standard PCM WAV files, the fmt chunk is exactly 16 bytes (not counting the 8-byte chunk header):

| Field | Byte Offset | Size (bytes) | Type | Valid Range | Notes |
|-------|-------------|--------------|------|-------------|-------|
| `ckID` | 0x00 | 4 | FOURCC | `"fmt "` (0x666D7420) | Literal space character at byte 3 |
| `cksize` | 0x04 | 4 | UINT32 LE | 16, 18, or 40 | Payload size only, not including 8-byte header |
| `wFormatTag` | 0x08 | 2 | UINT16 LE | See format tag table | Primary format identifier |
| `nChannels` | 0x0A | 2 | UINT16 LE | 1–65536 | Channel count |
| `nSamplesPerSec` | 0x0C | 4 | UINT32 LE | > 0 | Sample rate in Hz |
| `nAvgBytesPerSec` | 0x10 | 4 | UINT32 LE | > 0 | Bytes per second = sample_rate × block_align |
| `nBlockAlign` | 0x14 | 2 | UINT16 LE | > 0 | Bytes per complete sample frame |
| `wBitsPerSample` | 0x16 | 2 | UINT16 LE | 8, 16, 20, 24, 32, 64 | Container size in bits |

**Critical Relationships (must all hold true):**
```
nBlockAlign        = nChannels × ceil(wBitsPerSample / 8)
nAvgBytesPerSec    = nSamplesPerSec × nBlockAlign
data_chunk_size    = num_samples × nBlockAlign
file_size          = 8 (RIFF header) + RIFF_size
RIFF_size          = 4 ("WAVE") + sum of all chunk sizes + chunk headers
```

**Complete Format Tag Registry:**

| wFormatTag | Name | Description | Requires fact chunk? |
|------------|------|-------------|----------------------|
| 0x0001 | WAVE_FORMAT_PCM | Pulse Code Modulation (integer) | No |
| 0x0002 | WAVE_FORMAT_ADPCM | Microsoft ADPCM | Yes |
| 0x0003 | WAVE_FORMAT_IEEE_FLOAT | IEEE 754 float | No |
| 0x0006 | WAVE_FORMAT_ALAW | A-law companded (ITU G.711) | No |
| 0x0007 | WAVE_FORMAT_MULAW | μ-law companded (ITU G.711) | No |
| 0x0011 | WAVE_FORMAT_IMA_ADPCM | IMA ADPCM | Yes |
| 0x0022 | WAVE_FORMAT_EXTENSIBLE | Extensible format (tag also = 0xFFFE) | Depends |
| 0xFFFE | WAVE_FORMAT_UNKNOWN | Placeholder for extensible — actual format in GUID | Depends |
| 0x0030 | WAVE_FORMAT_DIVUS | GSM 6.10 | Yes |
| 0x0040 | WAVE_FORMAT_G723 | G.723.1 | Yes |
| 0x0101 | WAVE_FORMAT_G726 | G.726 ADPCM | Yes |

### 3.3 Extensible Format Chunk (WAVE_FORMAT_EXTENSIBLE)

The extensible format (wFormatTag = 0xFFFE, or 0x0022 in the original Microsoft registration) extends the standard fmt chunk to support arbitrary audio formats, explicit bit depth specification, and multi-channel speaker mapping. The fmt chunk size is always 40 bytes for extensible format.

**Complete Extensible fmt Chunk (40 bytes payload + 8 byte header = 48 bytes total):**

```
Offset  Size  Type    Field                     Description
------  ----  -----   -----                     -----------
0x00    4     FOURCC  ckID                      "fmt "
0x04    4     UINT32  cksize                    40
0x08    2     UINT16  wFormatTag                0xFFFE (or 0x0022)
0x0A    2     UINT16  nChannels                 Channel count
0x0C    4     UINT32  nSamplesPerSec            Sample rate
0x10    4     UINT32  nAvgBytesPerSec           Bytes per second
0x14    2     UINT16  nBlockAlign              Channel × container_bytes
0x16    2     UINT16  wBitsPerSample            Container size (bits)
0x18    2     UINT16  cbSize                    22 (size of extension = this field onward)
0x1A    2     UINT16  wValidBitsPerSample       Actual bit depth
0x1C    4     UINT32  dwChannelMask             Speaker position bitmask
0x20    16    GUID    SubFormat                 Format subtype
```

**SubFormat GUID Definitions:**

| SubFormat | GUID Bytes (hex) | Format |
|-----------|-----------------|--------|
| KSDATAFORMAT_SUBTYPE_PCM | 01 00 00 00 00 00 10 80 00 00 AA 00 38 9B 71 | PCM integer |
| KSDATAFORMAT_SUBTYPE_IEEE_FLOAT | 03 00 00 00 00 00 10 80 00 00 AA 00 38 9B 71 | IEEE 754 float |

The GUID structure follows the standard Microsoft GUID layout: the first two bytes match the equivalent wFormatTag value (0x0001 for PCM, 0x0003 for float). The remaining 14 bytes are the KSDATAFORMAT_SUBTYPE vendor-specific identifier.

**Important Notes on Extensible Format:**
- `wBitsPerSample` is the **container size** (how many bytes each sample occupies)
- `wValidBitsPerSample` is the **actual precision** (may be less than container for non-byte-aligned depths)
- `nBlockAlign` is always calculated from the container size, not the valid bits
- `dwChannelMask` with value 0 means channels map directly to output ports in order

### 3.4 Channel Mask — Complete Bit Definitions

The `dwChannelMask` is a 32-bit bitmask (stored as UINT32 LE) where each bit represents a specific speaker position. The bit order determines the channel ordering in the interleaved audio data — the lowest set bit corresponds to the first channel, the next lowest set bit to the second channel, and so on.

**Complete Channel Mask Bit Table:**

| Bit Position | Hex Value | Constant | Speaker Position |
|--------------|-----------|----------|-----------------|
| 0 | 0x00000001 | SPEAKER_FRONT_LEFT | Front Left (FL) |
| 1 | 0x00000002 | SPEAKER_FRONT_RIGHT | Front Right (FR) |
| 2 | 0x00000004 | SPEAKER_FRONT_CENTER | Front Center (FC) |
| 3 | 0x00000008 | SPEAKER_LOW_FREQUENCY | Low Frequency (LFE / Subwoofer) |
| 4 | 0x00000010 | SPEAKER_BACK_LEFT | Back Left (BL) |
| 5 | 0x00000020 | SPEAKER_BACK_RIGHT | Back Right (BR) |
| 6 | 0x00000040 | SPEAKER_FRONT_LEFT_OF_CENTER | Front Left of Center (FLC) |
| 7 | 0x00000080 | SPEAKER_FRONT_RIGHT_OF_CENTER | Front Right of Center (FRC) |
| 8 | 0x00000100 | SPEAKER_BACK_CENTER | Back Center (BC) |
| 9 | 0x00000200 | SPEAKER_SIDE_LEFT | Side Left (SL) |
| 10 | 0x00000400 | SPEAKER_SIDE_RIGHT | Side Right (SR) |
| 11 | 0x00000800 | SPEAKER_TOP_CENTER | Top Center (TC) |
| 12 | 0x00001000 | SPEAKER_TOP_FRONT_LEFT | Top Front Left (TFL) |
| 13 | 0x00002000 | SPEAKER_TOP_FRONT_CENTER | Top Front Center (TFC) |
| 14 | 0x00004000 | SPEAKER_TOP_FRONT_RIGHT | Top Front Right (TFR) |
| 15 | 0x00008000 | SPEAKER_TOP_BACK_LEFT | Top Back Left (TBL) |
| 16 | 0x00010000 | SPEAKER_TOP_BACK_CENTER | Top Back Center (TBC) |
| 17 | 0x00020000 | SPEAKER_TOP_BACK_RIGHT | Top Back Right (TBR) |
| 22–31 | 0x00400000–0x80000000 | — | Reserved for future use |

**Standard Channel Mask Reference:**

| Configuration | Channel Mask (hex) | Set Bits | Channel Order in Data |
|---------------|-------------------|----------|----------------------|
| Mono | 0x0004 | FC | [FC] |
| Stereo | 0x0003 | FL, FR | [FL, FR] |
| 2.1 (Stereo + Sub) | 0x000B | FL, FR, LFE | [FL, FR, LFE] |
| 3.0 (3.0 surround) | 0x0007 | FL, FR, FC | [FL, FR, FC] |
| 4.0 Quad | 0x0033 | FL, FR, BL, BR | [FL, FR, BL, BR] |
| 4.0 (Octagonal) | 0x0099 | FL, FR, BL, BR, BC | [FL, FR, BC, BL, BR] |
| 5.0 | 0x0037 | FL, FR, FC, BL, BR | [FL, FR, FC, BL, BR] |
| 5.1 (Standard) | 0x003F | FL, FR, FC, LFE, BL, BR | [FL, FR, FC, LFE, BL, BR] |
| 5.1 (Side-surround) | 0x060F | FL, FR, FC, LFE, SL, SR | [FL, FR, FC, LFE, SL, SR] |
| 6.1 | 0x013F | FL, FR, FC, LFE, BL, BR, BC | [FL, FR, FC, LFE, BC, BL, BR] |
| 7.1 (Standard) | 0x00FF | FL, FR, FC, LFE, BL, BR, FLC, FRC | [FL, FR, FC, LFE, FLC, FRC, BL, BR] |
| 7.1 (Wide) | 0x06FF | FL, FR, FC, LFE, BL, BR, SL, SR | [FL, FR, FC, LFE, SL, SR, BL, BR] |
| 7.1.4 Atmos | 0x0BFF | FL, FR, FC, LFE, BL, BR, FLC, FRC + 4 heights | [FL, FR, FC, LFE, FLC, FRC, TFL, TFR, TBL, TBR, BL, BR] |

### 3.5 "data" Chunk — Complete Reference

The data chunk contains the raw audio samples. All samples are stored in interleaved format, meaning all channels for sample 0 are stored together, then all channels for sample 1, and so on.

| Field | Byte Offset (relative to chunk start) | Size | Type | Description |
|-------|---------------------------------------|------|------|-------------|
| `ckID` | 0x00 | 4 | FOURCC | `"data"` (0x64617461) |
| `cksize` | 0x04 | 4 | UINT32 LE | Size of audio data in bytes |
| `audio_data` | 0x08 | `cksize` | BYTE[] | Interleaved samples |
| `pad_byte` | `cksize` (if odd) | 1 | BYTE | 0x00 padding to word-align next chunk |

**Core Calculations:**

```
bytes_per_sample_container = ceil(wBitsPerSample / 8)
num_samples                = data_chunk_size / nBlockAlign
duration_seconds          = num_samples / nSamplesPerSec
duration_ms               = (num_samples * 1000) / nSamplesPerSec
bitrate_bps               = nSamplesPerSec × nChannels × wBitsPerSample
```

**Interleaved Sample Layout (Stereo 16-bit):**
```
Byte 0: Sample 0, Channel 0 (Left), Low byte
Byte 1: Sample 0, Channel 0 (Left), High byte
Byte 2: Sample 0, Channel 1 (Right), Low byte
Byte 3: Sample 0, Channel 1 (Right), High byte
Byte 4: Sample 1, Channel 0 (Left), Low byte
...
```

**Interleaved Sample Layout (5.1 24-bit):**
```
Byte 0–2: Sample 0, Channel 0 (FL), 3 bytes LE
Byte 3–5: Sample 0, Channel 1 (FR), 3 bytes LE
Byte 6–8: Sample 0, Channel 2 (FC), 3 bytes LE
Byte 9–11: Sample 0, Channel 3 (LFE), 3 bytes LE
Byte 12–14: Sample 0, Channel 4 (BL), 3 bytes LE
Byte 15–17: Sample 0, Channel 5 (BR), 3 bytes LE
Byte 18–20: Sample 1, Channel 0 (FL), 3 bytes LE
...
```

### 3.6 Additional Standard Chunks

**"LIST" Chunk with "INFO" Sub-chunks:**

| Chunk ID | Max Length | Encoding | Description |
|----------|-----------|----------|-------------|
| `INAM` | 256 chars | ASCII/ZSTR | Title / Name |
| `IART` | 256 chars | ASCII/ZSTR | Artist / Author |
| `ICMT` | unlimited | ASCII | Comment text |
| `ICRD` | 10 chars | ASCII | Creation date (YYYY-MM-DD) |
| `IGNR` | 256 chars | ASCII/ZSTR | Genre |
| `IPRD` | 256 chars | ASCII/ZSTR | Product / Album |
| `ISFT` | 256 chars | ASCII/ZSTR | Software / Encoder |
| `ISRC` | 256 chars | ASCII | Source supplier |
| `ICOP` | 256 chars | ASCII | Copyright notice |
| `ISBJ` | 256 chars | ASCII/ZSTR | Subject / Description |
| `IENG` | 256 chars | ASCII | Engineer name(s) |
| `ITCH` | 256 chars | ASCII | Technician name |
| `IKEY` | 256 chars | ASCII | Search keywords |
| `IMED` | 256 chars | ASCII | Original medium |
| `ISRF` | 256 chars | ASCII | Source form |
| `IARL` | 256 chars | ASCII | Archival location |
| `ICMS` | 256 chars | ASCII | Commissioned by |
| `ICRP` | 256 chars | ASCII | Cropping information |
| `IDIM` | 256 chars | ASCII | Original dimensions |
| `IDPI` | 256 chars | ASCII | Digitization DPI |
| `ILGT` | 256 chars | ASCII | Lightness settings |

All INFO chunks use null-terminated ASCII strings. The LIST chunk itself has:
```
Offset 0: FOURCC "LIST" (0x4C495354)
Offset 4: UINT32 size_of_LIST_payload
Offset 8: FOURCC "INFO" (0x494E464F)
Offset 12+: sub-chunks
```

**"fact" Chunk (Required for non-PCM formats, optional for PCM):**

| Field | Size | Type | Description |
|-------|------|------|-------------|
| `dwSampleLength` | 4 | UINT32 | Total number of audio samples per channel |

The fact chunk provides the sample count, which is useful when the data chunk size is not evenly divisible or when decoding to know the total number of samples in advance.

**"cue " Chunk (Cue Points):**

Each cue point occupies 24 bytes:

| Field | Size | Type | Description |
|-------|------|------|-------------|
| `dwName` | 4 | UINT32 | Unique cue point identifier |
| `dwPosition` | 4 | UINT32 | Position in play order |
| `fccChunk` | 4 | FOURCC | Chunk ID containing the cue point (usually "data") |
| `dwChunkStart` | 4 | UINT32 | Byte offset to the start of the chunk containing the cue |
| `dwBlockStart` | 4 | UINT32 | Byte offset to the start of the block containing the cue |
| `dwSampleOffset` | 4 | UINT32 | Sample offset within the block |

The cue chunk header specifies the number of cue points: `dwCuePoints` as UINT32 at the beginning of the chunk data.

**"smpl" Chunk (Sampler / Loop Information):**

| Field | Size | Description |
|-------|------|-------------|
| `dwManufacturer` | 4 | MIDI manufacturer ID |
| `dwProduct` | 4 | MIDI product ID |
| `dwSamplePeriod` | 4 | Sample period in nanoseconds (e.g., 22675 for 44.1kHz) |
| `dwMIDIPhraseNote` | 4 | Base MIDI note number |
| `dwMIDIPitchFraction` | 4 | MIDI pitch fraction |
| `dwSMPTEFormat` | 4 | SMPTE format (24, 25, 29, 30 fps) |
| `dwSMPTEOffset` | 4 | SMPTE offset |
| `cSampleLoops` | 4 | Number of loop definitions |
| `cbSamplerData` | 4 | Size of sampler-specific data |
| Loop structures[] | variable | Each loop: dwIdentifier, dwType, dwStart, dwEnd, dwFraction, dwPlayCount |

**"labl" Chunk (Text Label for a Cue Point):**

| Field | Size | Type | Description |
|-------|------|------|-------------|
| `dwName` | 4 | UINT32 | Cue point identifier (matches cue point dwName) |
| `text` | variable | ASCII | Null-terminated label string |

**"ltxt" Chunk (Labeled Text for a Cue Point):**

| Field | Size | Type | Description |
|-------|------|------|-------------|
| `dwName` | 4 | UINT32 | Cue point identifier |
| `dwLength` | 4 | UINT32 | Text length in bytes |
| `dwUsage` | 4 | UINT32 | Usage type identifier |
| `dwCountry` | 2 | UINT16 | Country code |
| `dwLanguage` | 2 | UINT16 | Language code |
| `dwDialect` | 2 | UINT16 | Dialect code |
| `dwCodePage` | 2 | UINT16 | Character code page |
| `text` | variable | BYTE[] | Label text |

**"plst" Chunk (Playlist / Play Order):**

| Field | Size | Description |
|-------|------|-------------|
| `dwSegments` | 4 | Number of segments |
| Per segment: dwCuePointID, dwLength (samples), dwLoops | 12 bytes each | Segment definitions |

### 3.7 Complete Bit-Level Field Map

This table provides the complete byte-offset, size, and type reference for every field in a standard extensible-format WAV file, from the RIFF header through the fmt chunk:

| Field | File Offset | Size (bytes) | Type | Valid Range / Notes |
|-------|-------------|--------------|------|---------------------|
| RIFF ID | 0x00 | 4 | FOURCC | "RIFF" (0x52494646) |
| RIFF Size | 0x04 | 4 | UINT32 LE | ≥ 12 (file size minus 8) |
| WAVE ID | 0x08 | 4 | FOURCC | "WAVE" (0x57415645) |
| fmt ID | 0x0C | 4 | FOURCC | "fmt " (0x666D7420) |
| fmt Size | 0x10 | 4 | UINT32 LE | 16 (PCM), 18 (float), 40 (extensible) |
| wFormatTag | 0x14 | 2 | UINT16 LE | 0x0001, 0x0003, 0xFFFE |
| nChannels | 0x16 | 2 | UINT16 LE | 1–65536 |
| nSamplesPerSec | 0x18 | 4 | UINT32 LE | > 0 (Hz) |
| nAvgBytesPerSec | 0x1C | 4 | UINT32 LE | > 0 |
| nBlockAlign | 0x20 | 2 | UINT16 LE | > 0 |
| wBitsPerSample | 0x22 | 2 | UINT16 LE | 8, 16, 20, 24, 32, 64 |
| cbSize | 0x24 | 2 | UINT16 LE | 0 (PCM), 22 (extensible) |
| wValidBitsPerSample | 0x26 | 2 | UINT16 LE | ≤ wBitsPerSample |
| dwChannelMask | 0x28 | 4 | UINT32 LE | 0x0000–0x0FFFF |
| SubFormat (bytes 0–1) | 0x2C | 2 | UINT16 LE | 0x0001 (PCM), 0x0003 (float) |
| SubFormat (bytes 2–15) | 0x2E | 14 | BYTE[] | Vendor GUID bytes |
| data ID | variable | 4 | FOURCC | "data" (0x64617461) |
| data Size | variable+4 | 4 | UINT32 LE | ≥ 0 |
| Audio Data | variable+8 | N | BYTE[] | Interleaved samples |
| Pad Byte | variable+8+N | 0 or 1 | BYTE | 0x00 if data size is odd |

### 3.8 Sample Format — Complete Encoding Reference

**8-bit PCM (unsigned integer):**
- Container: 1 byte per sample
- Encoding: unsigned integer, 0–255
- Zero crossing: 128 (silence = 128, not 0)
- Maximum positive: 255, Maximum negative: 0
- Range: silence at 128, full-scale positive at 255, full-scale negative at 0
- Formula: `sample_value = raw_byte - 128`

**16-bit PCM (signed integer, two's complement):**
- Container: 2 bytes per sample
- Encoding: signed two's complement, little-endian
- Zero crossing: 0
- Maximum positive: 32767, Maximum negative: -32768
- Range: ±32768 (centered at 0)
- Formula: `sample_value = int16_t(low_byte | (high_byte << 8))`

**20-bit PCM (packed):**
- Container: 3 bytes per sample (or 4 bytes with zero-padding in 32-bit container)
- Encoding: signed integer, little-endian 3-byte packing
- Value range: ±524287 (20-bit signed range)
- Storage: left-justified within container (MSB at bit 19)
- If in 24-bit container: bits 20–23 are padding (typically 0)
- If in 32-bit container: bits 20–31 are padding (typically 0)
- Formula (24-bit container): `sample = int32_t(data[0] | (data[1] << 8) | (data[2] << 16))`
- Sign extension: if bit 19 is set, extend sign to bits 20–31

**24-bit PCM (signed integer, three-byte packing):**
- Container: 3 bytes per sample
- Encoding: signed two's complement, little-endian three-byte integer
- Zero crossing: 0
- Maximum positive: 8388607, Maximum negative: -8388608
- Range: ±8388608 (centered at 0)
- Formula: `sample = int32_t(data[0] | (data[1] << 8) | (data[2] << 16))`
- Sign extension required when promoting to 32-bit

**32-bit PCM (signed integer, four-byte):**
- Container: 4 bytes per sample
- Encoding: signed two's complement, little-endian
- Zero crossing: 0
- Maximum positive: 2147483647, Maximum negative: -2147483648
- Range: ±2147483648 (centered at 0)
- Formula: `sample = int32_t(data[0] | (data[1] << 8) | (data[2] << 16) | (data[3] << 24))`

**32-bit IEEE Float (single precision):**
- Container: 4 bytes per sample
- Encoding: IEEE 754 binary32, little-endian
- Value range: normalized to [-1.0, +1.0] for audio; values outside this range cause clipping
- Zero crossing: 0x00000000
- Maximum positive: 1.0 (0x3F800000)
- Maximum negative: -1.0 (0xBF800000)
- Special values: NaN and infinity should not occur in audio data
- Denormals are possible near silence and may cause performance issues on some hardware

**64-bit IEEE Float (double precision):**
- Container: 8 bytes per sample
- Encoding: IEEE 754 binary64, little-endian
- Value range: normalized to [-1.0, +1.0] for audio
- Used for intermediate processing precision in DAWs

**Sample Value Mapping:**

| Bit Depth | Type | Zero Level | Max Positive | Max Negative | Midpoint |
|-----------|------|------------|--------------|--------------|----------|
| 8-bit PCM | unsigned | 128 | 255 | 0 | 128 |
| 16-bit PCM | signed | 0 | 32767 | -32768 | 0 |
| 20-bit PCM | signed | 0 | 524287 | -524288 | 0 |
| 24-bit PCM | signed | 0 | 8388607 | -8388608 | 0 |
| 32-bit PCM | signed | 0 | 2147483647 | -2147483648 | 0 |
| 32-bit float | IEEE float | 0.0 | 1.0 | -1.0 | 0.0 |
| 64-bit float | IEEE float | 0.0 | 1.0 | -1.0 | 0.0 |

---

## 4. ENCODING ALGORITHM

### 4.1 Complete WAV Encoding Pipeline

Encoding raw PCM audio into a WAV file involves five stages: format preparation, header calculation, RIFF header writing, fmt chunk writing, and data chunk writing.

**Stage 1: Format Preparation**

```
Input:  Array of audio samples (one array per channel)
Output: Interleaved byte stream

For each sample frame (one sample from every channel):
    For each channel in channel_order (per dwChannelMask):
        Convert sample to the target container format:
            For 16-bit signed:   pack as int16 → 2 bytes LE
            For 24-bit:          pack as 3-byte int → 3 bytes LE
            For 32-bit signed:   pack as int32 → 4 bytes LE
            For 32-bit float:    pack as IEEE 754 binary32 → 4 bytes LE
        Append to output buffer
```

**Stage 2: Header Value Calculation**

```
1. bytes_per_sample = ceil(wBitsPerSample / 8)
2. nBlockAlign = nChannels × bytes_per_sample
3. nAvgBytesPerSec = nSamplesPerSec × nBlockAlign
4. num_samples = total_samples_in_input
5. audio_data_bytes = num_samples × nBlockAlign
6. data_chunk_size = audio_data_bytes
7. RIFF_size = 4 + 12 (fmt chunk header + data chunk header) + fmt_data_size + audio_data_bytes
   (add padding contributions if any)
8. file_size = RIFF_size + 8
```

**Stage 3: RIFF Header (12 bytes)**

```
Offset 0: "RIFF" (52 46 49 46 in LE due to ASCII storage)
Offset 4: RIFF_size as UINT32 LE (file_size - 8)
Offset 8: "WAVE" (57 41 56 45)
```

**Stage 4: fmt Chunk**

For standard PCM (16 bytes):
```
Offset 0:  "fmt " (66 6D 74 20)
Offset 4:  16 as UINT32 LE
Offset 8:  wFormatTag = 0x0001 as UINT16 LE
Offset 10: nChannels as UINT16 LE
Offset 12: nSamplesPerSec as UINT32 LE
Offset 16: nAvgBytesPerSec as UINT32 LE
Offset 20: nBlockAlign as UINT16 LE
Offset 22: wBitsPerSample as UINT16 LE
```

For extensible format (40 bytes):
```
Offset 0:  "fmt " (66 6D 74 20)
Offset 4:  40 as UINT32 LE
Offset 8:  wFormatTag = 0xFFFE as UINT16 LE
Offset 10: nChannels as UINT16 LE
Offset 12: nSamplesPerSec as UINT32 LE
Offset 16: nAvgBytesPerSec as UINT32 LE
Offset 20: nBlockAlign as UINT16 LE
Offset 22: wBitsPerSample as UINT16 LE (container size)
Offset 24: cbSize = 22 as UINT16 LE
Offset 26: wValidBitsPerSample as UINT16 LE
Offset 28: dwChannelMask as UINT32 LE
Offset 32: SubFormat GUID (16 bytes)
```

**Stage 5: data Chunk**

```
Offset 0:  "data" (64 61 74 61)
Offset 4:  audio_data_bytes as UINT32 LE
Offset 8:  [interleaved audio samples]
[If audio_data_bytes is odd, append 0x00 padding byte]
```

### 4.2 Complete Encoding Implementation

```c
#include <stdint.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

typedef struct {
    uint16_t wFormatTag;
    uint16_t nChannels;
    uint32_t nSamplesPerSec;
    uint32_t nAvgBytesPerSec;
    uint16_t nBlockAlign;
    uint16_t wBitsPerSample;
    uint16_t cbSize;           // 0 for PCM, 22 for extensible
    uint16_t wValidBitsPerSample;
    uint32_t dwChannelMask;
    uint8_t  SubFormat[16];
} WaveFormat;

static void write_fourcc(FILE *fp, const char *id) {
    fwrite(id, 1, 4, fp);
}

static void write_uint16_le(FILE *fp, uint16_t val) {
    uint8_t buf[2] = { val & 0xFF, (val >> 8) & 0xFF };
    fwrite(buf, 1, 2, fp);
}

static void write_uint32_le(FILE *fp, uint32_t val) {
    uint8_t buf[4] = {
        val & 0xFF, (val >> 8) & 0xFF,
        (val >> 16) & 0xFF, (val >> 24) & 0xFF
    };
    fwrite(buf, 1, 4, fp);
}

int write_wav_file(const char *filename, WaveFormat *fmt,
                   const uint8_t *audio_data, uint64_t num_samples) {
    FILE *fp = fopen(filename, "wb");
    if (!fp) return -1;

    uint32_t bytes_per_sample = (fmt->wBitsPerSample + 7) / 8;
    uint32_t block_align = fmt->nChannels * bytes_per_sample;
    uint32_t audio_bytes = num_samples * block_align;
    uint32_t fmt_size = (fmt->cbSize > 0) ? 40 : 16;

    // RIFF header
    write_fourcc(fp, "RIFF");
    uint32_t riff_size = 4 + 8 + fmt_size + 8 + audio_bytes;
    if (audio_bytes % 2) riff_size++;  // padding byte
    write_uint32_le(fp, riff_size);
    write_fourcc(fp, "WAVE");

    // fmt chunk
    write_fourcc(fp, "fmt ");
    write_uint32_le(fp, fmt_size);
    write_uint16_le(fp, fmt->wFormatTag);
    write_uint16_le(fp, fmt->nChannels);
    write_uint32_le(fp, fmt->nSamplesPerSec);
    write_uint32_le(fp, fmt->nBlockAlign);
    write_uint16_le(fp, fmt->wBitsPerSample);

    if (fmt_size == 40) {
        write_uint16_le(fp, fmt->cbSize);
        write_uint16_le(fp, fmt->wValidBitsPerSample);
        write_uint32_le(fp, fmt->dwChannelMask);
        fwrite(fmt->SubFormat, 1, 16, fp);
    }

    // data chunk
    write_fourcc(fp, "data");
    write_uint32_le(fp, audio_bytes);
    fwrite(audio_data, 1, audio_bytes, fp);
    if (audio_bytes % 2) {
        uint8_t pad = 0;
        fwrite(&pad, 1, 1, fp);  // padding byte
    }

    fclose(fp);
    return 0;
}
```

### 4.3 BWF Extension Handling

The `bext` chunk is the primary broadcast extension. It is written between the fmt chunk and the data chunk.

**bext Chunk Structure (Version 2 — 1202+ bytes):**

| Field | Size (bytes) | Type | Description |
|-------|-------------|------|-------------|
| Description | 256 | ASCII | Free-form text description of recording |
| Originator | 32 | ASCII | Name of the origination system |
| OriginatorReference | 32 | ASCII | Unique reference code from originator |
| OriginationDate | 10 | ASCII | Recording date in YYYY-MM-DD format |
| OriginationTime | 8 | ASCII | Recording time in HH:MM:SS format |
| TimeReference (low) | 4 | UINT32 LE | Sample count since midnight UTC (lower 32 bits) |
| TimeReference (high) | 4 | UINT32 LE | Sample count since midnight UTC (upper 32 bits) |
| Version | 2 | UINT16 LE | BWF version: 0 (602 bytes), 1 (602+64=666 bytes), 2 (602+64+8=1234 bytes) |
| UMID | 64 | BYTE[64] | SMPTE UMID (basic=32, extended=64). Present only if Version ≥ 1 |
| LoudnessValue | 2 | INT16 LE | Integrated loudness × 100 LUFS. Present only if Version = 2 |
| LoudnessRange | 2 | INT16 LE | Loudness range × 100 LU. Present only if Version = 2 |
| MaxTruePeakLevel | 2 | INT16 LE | Maximum true peak level × 100 dBTP. Present only if Version = 2 |
| Reserved | 190 | BYTE[190] | Reserved for future use |
| CodingHistory | N | ASCII | Coding history, each operation on a new line |

**CodingHistory String Format:**
Each encoding/transformation is recorded as a comma-separated record: `Coding=, Format=, ...`
Example: `CodingHistory=AMR, 48000Hz, 64kbps, stereo\nPCM, 48000Hz, 16-bit, stereo\n` — each entry terminated by CR+LF (0x0D 0x0A).

### 4.4 Creating BWF Files with Loudness Metadata

```c
void write_bext_chunk(FILE *fp, const char *description,
                      const char *originator, const char *originator_ref,
                      const char *date, const char *time,
                      uint64_t time_reference, int version,
                      float integrated_lufs, float loudness_range_db,
                      float max_true_peak_db) {

    write_fourcc(fp, "bext");

    uint32_t base_size = 602;  // Version 0 base
    if (version >= 1) base_size += 64;   // UMID
    if (version >= 2) base_size += 8;    // Loudness metadata

    uint32_t coding_history_size = 0;  // filled by caller or estimated
    write_uint32_le(fp, base_size + coding_history_size);

    // Description (256 bytes)
    memset(write_ptr, 0, 256);
    strncpy(write_ptr, description, 255);

    // Originator (32 bytes)
    memset(write_ptr, 0, 32);
    strncpy(write_ptr, originator, 31);

    // OriginatorReference (32 bytes)
    memset(write_ptr, 0, 32);
    strncpy(write_ptr, originator_ref, 31);

    // OriginationDate (10 bytes): "YYYY-MM-DD\0"
    memcpy(write_ptr, date, 10);

    // OriginationTime (8 bytes): "HH:MM:SS\0"
    memcpy(write_ptr, time, 8);

    // TimeReference (8 bytes: high + low UINT32)
    write_uint32_le(fp, (time_reference >> 32) & 0xFFFFFFFF);
    write_uint32_le(fp, time_reference & 0xFFFFFFFF);

    // Version (2 bytes)
    write_uint16_le(fp, version);

    // UMID (64 bytes) if version >= 1
    if (version >= 1) {
        fwrite(umid_bytes, 1, 64, fp);
    }

    // Loudness (8 bytes) if version >= 2
    if (version >= 2) {
        int16_t loudness_val = (int16_t)(integrated_lufs * 100.0);
        int16_t range_val = (int16_t)(loudness_range_db * 100.0);
        int16_t peak_val = (int16_t)(max_true_peak_db * 100.0);
        write_uint16_le(fp, loudness_val);
        write_uint16_le(fp, range_val);
        write_uint16_le(fp, peak_val);
        fwrite("\x00\x00", 1, 2, fp);  // Reserved (2 bytes)
    }

    // Reserved (190 bytes)
    fseek(fp, 190, SEEK_CUR);
}
```

---

## 5. DECODING ALGORITHM

### 5.1 Complete Parsing Algorithm

Decoding a WAV file requires reading and validating the RIFF/WAVE structure, parsing the fmt chunk to determine audio parameters, locating the data chunk, and reading samples.

```c
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define FOURCC(a,b,c,d) (((uint32_t)(a)) | ((uint32_t)(b) << 8) | \
    ((uint32_t)(c) << 16) | ((uint32_t)(d) << 24))

#define RIFF_ID   FOURCC('R','I','F','F')
#define WAVE_ID   FOURCC('W','A','V','E')
#define FMT__ID   FOURCC('f','m','t',' ')
#define DATA_ID   FOURCC('d','a','t','a')
#define LIST_ID   FOURCC('L','I','S','T')
#define BEXT_ID   FOURCC('b','e','x','t')
#define FACT_ID   FOURCC('f','a','c','t')
#define CUE_ID    FOURCC('c','u','e',' ')
#define SMPL_ID   FOURCC('s','m','p','l')

typedef struct {
    uint32_t fourcc;
    uint32_t size;
    uint64_t file_offset;
} ChunkHeader;

typedef struct {
    uint16_t wFormatTag;
    uint16_t nChannels;
    uint32_t nSamplesPerSec;
    uint32_t nAvgBytesPerSec;
    uint16_t nBlockAlign;
    uint16_t wBitsPerSample;
    uint16_t cbSize;
    uint16_t wValidBitsPerSample;
    uint32_t dwChannelMask;
    uint8_t  SubFormat[16];
    int is_extensible;
    int is_float;
} WAVFormat;

typedef struct {
    FILE *fp;
    uint64_t file_size;
    WAVFormat format;
    ChunkHeader data_chunk;
    ChunkHeader fact_chunk;
    uint64_t total_samples;
} WAVFile;

static uint32_t read_uint32(FILE *fp) {
    uint32_t val;
    fread(&val, 4, 1, fp);
    return val;  // Already LE from disk
}

static uint16_t read_uint16(FILE *fp) {
    uint16_t val;
    fread(&val, 2, 1, fp);
    return val;
}

static void read_fourcc(FILE *fp, char id[5]) {
    fread(id, 1, 4, fp);
    id[4] = '\0';
}

WAVFile* wav_open(const char *filename) {
    WAVFile *wav = calloc(1, sizeof(WAVFile));
    wav->fp = fopen(filename, "rb");
    if (!wav->fp) { free(wav); return NULL; }

    fseek(wav->fp, 0, SEEK_END);
    wav->file_size = ftell(wav->fp);
    fseek(wav->fp, 0, SEEK_SET);

    // Read RIFF header
    char riff_id[5];
    read_fourcc(wav->fp, riff_id);
    uint32_t riff_size = read_uint32(wav->fp);
    char wave_id[5];
    read_fourcc(wav->fp, wave_id);

    if (memcmp(riff_id, "RIFF", 4) != 0 || memcmp(wave_id, "WAVE", 4) != 0) {
        fprintf(stderr, "Invalid WAV file: missing RIFF/WAVE header\n");
        fclose(wav->fp); free(wav); return NULL;
    }

    // Read all chunks
    uint64_t pos = 12;  // After RIFF header
    while (pos < riff_size + 8 && ftell(wav->fp) < (long)wav->file_size) {
        ChunkHeader chunk;
        chunk.file_offset = ftell(wav->fp);

        char chunk_id[5];
        read_fourcc(wav->fp, chunk_id);
        chunk.fourcc = *(uint32_t*)chunk_id;
        chunk.size = read_uint32(wav->fp);

        if (chunk.fourcc == FMT__ID) {
            // Parse format chunk
            wav->format.wFormatTag = read_uint16(wav->fp);
            wav->format.nChannels = read_uint16(wav->fp);
            wav->format.nSamplesPerSec = read_uint32(wav->fp);
            wav->format.nAvgBytesPerSec = read_uint32(wav->fp);
            wav->format.nBlockAlign = read_uint16(wav->fp);
            wav->format.wBitsPerSample = read_uint16(wav->fp);

            if (wav->format.wFormatTag == 0xFFFE && chunk.size >= 40) {
                wav->format.cbSize = read_uint16(wav->fp);
                wav->format.wValidBitsPerSample = read_uint16(wav->fp);
                wav->format.dwChannelMask = read_uint32(wav->fp);
                fread(wav->format.SubFormat, 1, 16, wav->fp);
                wav->format.is_extensible = 1;
                wav->format.is_float = (wav->format.SubFormat[0] == 0x03);
            } else {
                wav->format.is_extensible = 0;
                wav->format.is_float = (wav->format.wFormatTag == 0x0003);
                wav->format.cbSize = 0;
            }

            // Skip to end of fmt chunk data
            if (chunk.size > 16) {
                fseek(wav->fp, chunk.file_offset + 8 + chunk.size, SEEK_SET);
            }
        } else if (chunk.fourcc == DATA_ID) {
            wav->data_chunk = chunk;
            wav->total_samples = chunk.size / wav->format.nBlockAlign;
            // Don't skip — this is where audio data starts
            break;  // data chunk is typically last or we want to start reading here
        } else if (chunk.fourcc == FACT_ID) {
            wav->fact_chunk = chunk;
            wav->total_samples = read_uint32(wav->fp);
            fseek(wav->fp, chunk.file_offset + 8 + chunk.size, SEEK_SET);
        } else {
            // Unknown chunk — skip it
            fseek(wav->fp, chunk.file_offset + 8 + chunk.size, SEEK_SET);
        }

        // Word-align to next chunk
        if (chunk.size % 2) {
            fseek(wav->fp, 1, SEEK_CUR);
        }

        pos = ftell(wav->fp);
    }

    // Seek to start of audio data
    fseek(wav->fp, wav->data_chunk.file_offset + 8, SEEK_SET);

    return wav;
}

void wav_close(WAVFile *wav) {
    if (wav) {
        fclose(wav->fp);
        free(wav);
    }
}
```

### 5.2 Sample-Level Access

After parsing, individual samples or sample ranges can be read:

```c
// Calculate byte offset for a specific sample
uint64_t wav_sample_offset(WAVFile *wav, uint64_t sample_num) {
    uint64_t bytes_per_sample = wav->format.nBlockAlign;
    return sample_num * bytes_per_sample;
}

// Seek to a specific sample
void wav_seek(WAVFile *wav, uint64_t sample_num) {
    uint64_t offset = wav->data_chunk.file_offset + 8 + wav_sample_offset(wav, sample_num);
    fseek(wav->fp, offset, SEEK_SET);
}

// Read N samples
size_t wav_read_samples(WAVFile *wav, void *buffer, size_t num_samples) {
    uint64_t max_samples = wav->total_samples;
    if (sample_num + num_samples > max_samples) {
        num_samples = max_samples - sample_num;
    }
    size_t bytes = num_samples * wav->format.nBlockAlign;
    return fread(buffer, 1, bytes, wav->fp);
}
```

### 5.3 Multi-Format Handling

**Detecting PCM vs IEEE Float:**
```c
int wav_is_float(WAVFile *wav) {
    if (wav->format.wFormatTag == 0x0003) return 1;
    if (wav->format.wFormatTag == 0xFFFE) {
        return (wav->format.SubFormat[0] == 0x03);
    }
    return 0;
}

int wav_is_pcm(WAVFile *wav) {
    if (wav->format.wFormatTag == 0x0001) return 1;
    if (wav->format.wFormatTag == 0xFFFE) {
        return (wav->format.SubFormat[0] == 0x01);
    }
    return 0;
}
```

**Sample Format Normalization:**
```c
// Convert any WAV sample format to 32-bit signed integer or float
void wav_to_normalized(WAVFile *wav, const uint8_t *src, float *dest, size_t count) {
    size_t bps = (wav->format.wBitsPerSample + 7) / 8;  // container bytes

    if (wav_is_float(wav)) {
        // IEEE float → normalized float
        for (size_t i = 0; i < count; i++) {
            float f;
            memcpy(&f, src + i * 4, 4);
            dest[i] = f;  // Already normalized
        }
    } else {
        // PCM integer → normalized float
        int valid_bits = (wav->format.wValidBitsPerSample > 0)
            ? wav->format.wValidBitsPerSample : wav->format.wBitsPerSample;
        double scale = 1.0 / (double)(1 << (valid_bits - 1));

        for (size_t i = 0; i < count; i++) {
            int32_t sample = 0;
            memcpy(&sample, src + i * bps, bps);
            // Sign extend for non-32-bit sizes
            if (bps == 3) {
                if (sample & 0x800000) sample |= ~0xFFFFFF;  // Sign extend 24-bit
            } else if (bps == 2) {
                if (sample & 0x8000) sample |= ~0xFFFF;      // Sign extend 16-bit
            }
            dest[i] = sample * scale;
        }
    }
}
```

---

## 6. RIFF CONTAINER INTEGRATION

### 6.1 RIFF Container Architecture

RIFF (Resource Interchange File Format) is the parent container for WAV. Understanding RIFF is essential for building robust WAV parsers that must handle unknown chunks gracefully.

**RIFF Chunk Anatomy:**
```
+--------+--------+-------------------------+
| ckID   | ckSize | ckData                  |
| 4 bytes| 4 bytes| ckSize bytes            |
+--------+--------+-------------------------+
Offset 0   Offset 4 Offset 8 (aligned to ckSize bytes)
```

**ckSize Field Rules:**
- The `ckSize` field stores the size of `ckData` only — it does not include the 8-byte chunk header
- A chunk with `ckSize = 0` is valid (empty chunk)
- For odd `ckSize`, a single 0x00 padding byte follows `ckData` (not counted in `ckSize`)
- Total file space occupied by chunk: `8 + ckSize + (ckSize % 2)`

**RIFF Form Structure:**
```
RIFF (form_type) [
    sub_chunk_1
    sub_chunk_2
    ...
]
```
- The top-level RIFF chunk's `ckData` contains a 4-byte form type identifier followed by all sub-chunks
- For WAV: form type = "WAVE" (0x57415645)
- Sub-chunks are direct children of the RIFF form; they are not nested within each other

**WAV Chunk Ordering Convention:**
While RIFF does not mandate chunk ordering, the conventional order for maximum compatibility is:
1. `fmt ` chunk (required)
2. `fact` chunk (if non-PCM)
3. `cue ` chunk (optional)
4. `plst` chunk (optional)
5. `LIST` chunks (`LIST INFO`, `LIST rlvm`, etc.)
6. `bext` chunk (BWF)
7. Other metadata chunks
8. `data` chunk (required, typically last — contains the bulk of file)

**RIFX (Big-Endian Variant):**
The `RIFX` variant (bytes 0–3 = "RIFX" instead of "RIFF") stores all multi-byte integers in big-endian byte order. The rest of the structure is identical. RIFX files are extremely rare in practice.

### 6.2 Chunk Alignment and Padding

Word-alignment padding is one of the most common sources of WAV parsing bugs. The rule: **every chunk starts on an even byte offset**. If a chunk's `ckSize` is odd, append a single 0x00 byte after the data.

**Correct chunk traversal loop:**
```c
void traverse_wav_chunks(WAVFile *wav) {
    fseek(wav->fp, 12, SEEK_SET);  // After RIFF header
    uint64_t end_pos = 8 + read_uint32_le(wav->fp, 4);  // RIFF end

    while (ftell(wav->fp) < end_pos && ftell(wav->fp) < wav->file_size) {
        uint64_t chunk_start = ftell(wav->fp);
        uint32_t ckID = read_uint32(wav->fp);
        uint32_t ckSize = read_uint32(wav->fp);

        // Process chunk at chunk_start + 8 to chunk_start + 8 + ckSize
        process_chunk(ckID, chunk_start + 8, ckSize);

        // Advance to next chunk with padding
        fseek(wav->fp, chunk_start + 8 + ckSize, SEEK_SET);
        if (ckSize % 2) {
            fseek(wav->fp, 1, SEEK_CUR);  // Skip padding byte
        }
    }
}
```

**Padding Byte Handling:**
- The padding byte (0x00) is not part of the chunk data
- Readers must skip the padding byte when present
- Writers must add the padding byte when `ckSize` is odd
- Some tools omit the padding byte on the final chunk; this is technically non-compliant but widely tolerated

---

## 7. METADATA ARCHITECTURE

### 7.1 LIST/INFO Metadata System

The LIST/INFO chunk provides text metadata in a nested structure. The LIST chunk acts as a container for multiple INFO sub-chunks, each of which is itself a chunk.

**LIST Chunk Structure:**
```
Offset 0:  FOURCC "LIST" (0x4C495354)
Offset 4:  UINT32 size  (size of type + all sub-chunks)
Offset 8:  FOURCC type  ("INFO" = 0x494E464F for standard metadata)
Offset 12: sub-chunks (each is a standard chunk)
```

**Sub-chunk Structure (within LIST/INFO):**
```
Offset 0:  FOURCC subchunk ID (e.g., "INAM", "IART")
Offset 4:  UINT32 subchunk size (size of data)
Offset 8:  data (ASCII/ZSTR, variable length)
[pad byte if odd]
```

**Text Encoding Rules:**
- Standard WAV metadata uses ASCII or null-terminated strings (ZSTR)
- Multi-byte characters should use UTF-16LE with BOM for international characters
- Each sub-chunk field should be null-terminated (the null is not counted in size)
- Empty strings are permitted (size = 0, no data)

**Complete INFO Sub-chunk Table:**

| Subchunk ID | Description | Typical Content |
|-------------|-------------|-----------------|
| INAM | Name / Title | Track title |
| IART | Artist | Artist, performer, or creator |
| ICMT | Comment | Free-form comment text |
| ICRD | Creation Date | YYYY-MM-DD format |
| IGNR | Genre | Music genre or content category |
| IPRD | Product / Album | Album title or product name |
| ISFT | Software | Encoding software name and version |
| ISRC | Source | Original supplier or catalog number |
| ICOP | Copyright | Copyright notice |
| ISBJ | Subject | Description of contents |
| IENG | Engineer | Recording engineer name |
| ITCH | Technician | Technician name |
| IKEY | Keywords | Comma-separated search terms |
| IMED | Medium | Original recording medium |
| ISRF | Source Form | Form of original material |
| IARL | Archival Location | Archive identification |
| ICMS | Commissioned | Commissioning entity |
| ICRP | Cropped | Cropping information |
| IDIM | Dimensions | Physical dimensions |
| IDPI | DPI | Digitization resolution |
| ILGT | Lightness | Lightness adjustment settings |

### 7.2 BWF Metadata (bext) — Complete Reference

The `bext` chunk (Broadcast Wave Extension) provides standardized broadcast metadata. It was designed to ensure that essential information about the recording survives format conversions and archive migrations.

**bext Version Detection Logic:**
```
if (chunk_size >= 602)  → Version 0
if (chunk_size >= 602+64=666) → Version 1
if (chunk_size >= 602+64+8=1234) → Version 2
```

**UMID (Unique Material Identifier) Structure:**
BWF Version 1 introduced UMID support. The UMID is a 64-byte SMPTE Universal Material Identifier that provides a globally unique identifier for the audio content.

Basic UMID (32 bytes):
| Field | Size | Description |
|-------|------|-------------|
| Universal Label | 12 | Registered ISO label (06 0A 2B 34 04 01 01 01 01 01 02 00 for BWF audio) |
| Length | 1 | 0x13 (basic UMID indicator) |
| Instance Number | 3 | Instance identifier |
| Material Number | 16 | Material-specific identifier |

Extended UMID (64 bytes):
| Field | Size | Description |
|-------|------|-------------|
| Basic UMID | 32 | Full basic UMID |
| Length | 1 | 0x33 (extended UMID indicator) |
| Source Pack | 31 | Extended metadata (time offset, geographic location, etc.) |

**Source Pack (31 bytes):**
| Field | Size | Description |
|-------|------|-------------|
| TimeOffset | 8 | Time offset since midnight (BCD or binary) |
| Date | 3 | Year (2 bytes), month, day |
| Geographic Location | 12 | Country (3 chars), region (3 chars), user (6 chars) coded per SMPTE RP 2057 |
| Spare | 8 | Reserved bytes |

### 7.3 Peak Metadata

**levl Chunk (BWF Peak Envelope):**

The `levl` chunk stores a peak envelope for each channel, enabling accurate gain riding during playback without scanning the entire file.

| Field | Size | Type | Description |
|-------|------|------|-------------|
| dwVersion | 4 | UINT32 | Format version (typically 1) |
| dwFormat | 4 | UINT32 | 1=UINT8 (0–255), 2=UINT16 |
| dwPointsPerValue | 4 | UINT32 | 1 or 2 peak values per entry |
| dwBlockSize | 4 | UINT32 | Number of audio samples per peak block |
| dwPeakBlockSize | 4 | UINT32 | Bytes per peak block |
| dwPeakChannels | 4 | UINT32 | Number of channels |
| dwSampleRate | 4 | UINT32 | Sample rate for time calculations |
| dwNumPeakBlocks | 4 | UINT32 | Number of peak blocks |
| PeakFrames | variable | variable | Peak envelope data |

**ReplayGain rgad Chunk (Legacy):**

| Field | Size | Type | Description |
|-------|------|------|-------------|
| fPeakAmplitude | 4 | float32 | Peak level (1.0 = full scale, 0.5 = -6 dBFS) |
| wRadioRgAdjust | 2 | UINT16 | Radio ReplayGain adjustment (dB × 100, sign in bit 15) |
| wAudiophileRgAdjust | 2 | UINT16 | Audiophile ReplayGain adjustment (dB × 100, sign in bit 15) |

Gain value encoding: `dB_gain = value / 100.0`, with bit 15 as the sign (0=positive, 1=negative).

### 7.4 Supported Metadata Fields Matrix

| Metadata Field | WAV Chunk | Max Length | Encoding | BWF Equivalent | Notes |
|---------------|-----------|-----------|----------|---------------|-------|
| Title | INAM | 256 | ASCII/ZSTR | — | Track name |
| Artist | IART | 256 | ASCII/ZSTR | — | Creator/performer |
| Album | IPRD | 256 | ASCII/ZSTR | — | Album title |
| Genre | IGNR | 256 | ASCII/ZSTR | — | Content category |
| Date | ICRD | 10 | ASCII | OriginationDate | YYYY-MM-DD |
| Comment | ICMT | varies | ASCII | Description (bext) | Free-form |
| Copyright | ICOP | 256 | ASCII | — | Legal notice |
| Software | ISFT | 256 | ASCII/ZSTR | — | Encoder ID |
| Source | ISRC | 256 | ASCII | OriginatorReference | Supplier |
| Description | — | — | — | bext Description | BWF-only |
| Originator | — | — | — | bext Originator | BWF-only |
| UMID | — | — | — | bext UMID | BWF-only |
| Integrated Loudness | — | — | — | bext LoudnessValue | BWF v2 |
| Loudness Range | — | — | — | bext LoudnessRange | BWF v2 |
| Max True Peak | — | — | — | bext MaxTruePeakLevel | BWF v2 |

### 7.5 Metadata Preservation During Conversion

When converting between formats, the following metadata preservation strategy applies for WAV targets:

- **INAM, IART, ICMT, IPRD, IGNR, ICOP, ISFT, ISRC, ICRD**: Map directly to LIST/INFO sub-chunks
- **bext metadata**: If converting from BWF source, create bext chunk in output
- **cue points**: Preserve `cue `, `labl`, `ltxt` chunks — these reference sample positions
- **sampler info**: Preserve `smpl` chunk for sampler instrument loops
- **Unknown chunks**: Copy verbatim as unknown chunks to preserve non-standard metadata

FFmpeg's `-map_metadata` option controls metadata transfer:
```bash
# Preserve all metadata from source
ffmpeg -i input.flac -map_metadata 0 output.wav

# Only preserve format-level metadata
ffmpeg -i input.flac -map_metadata 0:s output.wav

# Strip all metadata
ffmpeg -i input.flac -map_metadata -1 output.wav

# Copy BWF bext metadata specifically
ffmpeg -i input.bwf -c:a pcm_s24le -map_metadata 0 output.wav
```

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 Codec Identifiers

FFmpeg provides PCM codecs for every standard bit depth and byte order combination. The naming convention is `pcm_<bits><signed/unsigned><byte_order>`:

| FFmpeg Codec Name | Codec ID | Bits | Type | Byte Order | Value Range |
|-------------------|----------|------|------|------------|-------------|
| pcm_u8 | AV_CODEC_ID_PCM_U8 | 8 | unsigned | — | 0–255 |
| pcm_s8 | AV_CODEC_ID_PCM_S8 | 8 | signed | — | -128–127 |
| pcm_u16le | AV_CODEC_ID_PCM_U16LE | 16 | unsigned | Little-endian | 0–65535 |
| pcm_u16be | AV_CODEC_ID_PCM_U16BE | 16 | unsigned | Big-endian | 0–65535 |
| pcm_s16le | AV_CODEC_ID_PCM_S16LE | 16 | signed | Little-endian | -32768–32767 |
| pcm_s16be | AV_CODEC_ID_PCM_S16BE | 16 | signed | Big-endian | -32768–32767 |
| pcm_u24le | AV_CODEC_ID_PCM_U24LE | 24 | unsigned | Little-endian | 0–16777215 |
| pcm_u24be | AV_CODEC_ID_PCM_U24BE | 24 | unsigned | Big-endian | 0–16777215 |
| pcm_s24le | AV_CODEC_ID_PCM_S24LE | 24 | signed | Little-endian | -8388608–8388607 |
| pcm_s24be | AV_CODEC_ID_PCM_S24BE | 24 | signed | Big-endian | -8388608–8388607 |
| pcm_u32le | AV_CODEC_ID_PCM_U32LE | 32 | unsigned | Little-endian | 0–4294967295 |
| pcm_u32be | AV_CODEC_ID_PCM_U32BE | 32 | unsigned | Big-endian | 0–4294967295 |
| pcm_s32le | AV_CODEC_ID_PCM_S32LE | 32 | signed | Little-endian | ±2147483647 |
| pcm_s32be | AV_CODEC_ID_PCM_S32BE | 32 | signed | Big-endian | ±2147483647 |
| pcm_f32le | AV_CODEC_ID_PCM_F32LE | 32 | IEEE float | Little-endian | ±1.0 (normalized) |
| pcm_f32be | AV_CODEC_ID_PCM_F32BE | 32 | IEEE float | Big-endian | ±1.0 (normalized) |
| pcm_f64le | AV_CODEC_ID_PCM_F64LE | 64 | IEEE float | Little-endian | ±1.0 (normalized) |
| pcm_f64be | AV_CODEC_ID_PCM_F64BE | 64 | IEEE float | Big-endian | ±1.0 (normalized) |

FFmpeg muxer options for WAV:

| Muxer Option | Type | Default | Description |
|-------------|------|---------|-------------|
| write_bext | boolean | 0 | Write BWF bext chunk |
| write_peak | string | off | Write peak envelope (off/on/only) |
| peak_format | int | 2 | Peak format: 1=UINT8, 2=UINT16 |
| peak_block_size | int | 256 | Peak block size in samples |
| rf64 | string | auto | RF64 handling: never/auto/always |

### 8.2 Complete FFmpeg Encoding Commands

```bash
# =====================================================================
# BASIC WAV ENCODING
# =====================================================================

# 16-bit PCM (default — maximum compatibility)
ffmpeg -i input.wav -c:a pcm_s16le output.wav

# 24-bit PCM (professional audio)
ffmpeg -i input.wav -c:a pcm_s24le output.wav

# 32-bit PCM (high precision)
ffmpeg -i input.wav -c:a pcm_s32le output.wav

# 32-bit IEEE float (for intermediate processing)
ffmpeg -i input.wav -c:a pcm_f32le output.wav

# 64-bit IEEE float (double precision)
ffmpeg -i input.wav -c:a pcm_f64le output.wav

# =====================================================================
# SAMPLE RATE AND CHANNEL CONTROL
# =====================================================================

# 96 kHz, 24-bit (high-resolution)
ffmpeg -i input.wav -ar 96000 -c:a pcm_s24le output.wav

# 192 kHz, 32-bit float (studio quality)
ffmpeg -i input.wav -ar 192000 -c:a pcm_f32le output.wav

# Convert to mono
ffmpeg -i input.wav -ac 1 -c:a pcm_s16le output.wav

# 5.1 surround, 48 kHz, 24-bit
ffmpeg -i input.wav -ar 48000 -ac 6 -c:a pcm_s24le output.wav

# =====================================================================
# BWF BROADCAST EXTENSION
# =====================================================================

# Write BWF bext chunk with description
ffmpeg -i input.wav -c:a pcm_s24le -write_bext 1 \
  -metadata description="Interview Recording" \
  -metadata originator="Recording Studio" \
  -metadata originator_reference="2024-INT-001" \
  -metadata date="$(date +%Y-%m-%d)" \
  -metadata time="$(date +%H:%M:%S)" \
  output.bwf

# Write BWF with loudness metadata (Version 2)
ffmpeg -i input.wav -c:a pcm_s24le -write_bext 1 \
  -metadata description="Studio Recording" \
  -metadata originator="DAW System" \
  output_loudness.bwf

# =====================================================================
# PEAK ENVELOPE (REPLAYGAIN-COMPATIBLE)
# =====================================================================

# Write peak data (for volume normalization tools)
ffmpeg -i input.wav -c:a pcm_s16le -write_peak on output.wav

# Peak as UINT8 format (smaller but less precise)
ffmpeg -i input.wav -c:a pcm_s16le -write_peak on -peak_format 1 output.wav

# =====================================================================
# RF64 FOR LARGE FILES (> 4GB)
# =====================================================================

# Always use RF64 format
ffmpeg -i input.wav -c:a pcm_s24le -rf64 always output.rf64

# Auto (default): RF64 when needed
ffmpeg -i input.wav -c:a pcm_s24le -rf64 auto output.wav

# Never use RF64 (may truncate at 4GB boundary)
ffmpeg -i input.wav -c:a pcm_s24le -rf64 never output.wav

# =====================================================================
# EXPLICIT WAVEFORMATEXTENSIBLE
# =====================================================================

# FFmpeg does not natively generate WAVEFORMATEXTENSIBLE for standard PCM,
# but third-party tools or manual construction is needed for:
# - Non-standard channel masks (custom configurations)
# - Explicit wValidBitsPerSample specification
# - Specific GUID requirements
#
# Example: Create 5.1 with explicit channel mask using sox + manual muxing
sox input.wav -b 24 -r 48000 -c 6 temp.wav
```

### 8.3 FFmpeg Decoding and Inspection

```bash
# =====================================================================
# DECODING
# =====================================================================

# Decode to 16-bit PCM
ffmpeg -i input.wav -c:a pcm_s16le output.pcm

# Decode to 24-bit PCM (preserve precision)
ffmpeg -i input.wav -c:a pcm_s24le output.pcm

# Decode to 32-bit float (for processing)
ffmpeg -i input.wav -c:a pcm_f32le output.pcm

# Decode to raw PCM with explicit format
ffmpeg -i input.wav -f s16le -acodec pcm_s16le output.raw

# =====================================================================
# INSPECTION AND PROBING
# =====================================================================

# Stream information only
ffprobe -v quiet -show_streams -print_format compact input.wav

# Format and stream details
ffprobe -v error -show_entries stream=codec_name,sample_rate,channels,bits_per_sample,bit_rate \
  -of default=noprint_wrappers=1 input.wav

# Full format information
ffprobe -v quiet -show_format -show_streams input.wav

# Complete hex dump of header (first 512 bytes)
ffmpeg -i input.wav -f s16le -t 0 -i /dev/null -map 0:a -t 0 -f null -
# Instead: use hexdump
xxd input.wav | head -40

# Extract BWF metadata
ffprobe -v quiet -show_entries format_tags=description,originator,origination_date,origination_time \
  -of default=noprint_wrappers=1 input.bwf

# Extract all format tags
ffprobe -v quiet -show_entries format_tags -of compact input.wav

# =====================================================================
# METADATA EXTRACTION AND PRESERVATION
# =====================================================================

# Extract all metadata to text file
ffprobe -v quiet -show_format input.wav | grep -v "^#" > metadata.txt

# Convert with full metadata preservation
ffmpeg -i input.wav -c:a pcm_s24le -map_metadata 0 output.wav

# Convert and add new metadata
ffmpeg -i input.wav -c:a pcm_s24le \
  -metadata title="New Title" \
  -metadata artist="New Artist" \
  -metadata album="Album Name" \
  -metadata date="2024" \
  output.wav

# Strip metadata
ffmpeg -i input.wav -c:a pcm_s24le -map_metadata -1 output.wav
```

### 8.4 FFmpeg Metadata Mapping

FFmpeg maps WAV metadata according to the following conventions:

| FFmpeg Tag Key | WAV Chunk | Notes |
|---------------|-----------|-------|
| title | INAM | Track title |
| artist | IART | Performer/artist |
| album | IPRD | Album name |
| genre | IGNR | Genre |
| date | ICRD | Date (YYYY-MM-DD) |
| comment | ICMT | Comments |
| copyright | ICOP | Copyright notice |
| encoder | ISFT | Encoding software |
| language | — | Not standard in WAV |
| track | — | Non-standard in WAV (some tools use ITRK) |
| performer | IART | Alias for artist |

**BWF-specific mapping:**

| FFmpeg Property | bext Field | Notes |
|-----------------|------------|-------|
| — | Description | Via `-metadata description=...` |
| — | Originator | Via `-metadata originator=...` |
| — | OriginationDate | Via `-metadata date=...` (YYYY-MM-DD) |
| — | OriginationTime | Via `-metadata time=...` (HH:MM:SS) |
| — | OriginatorReference | Via `-metadata originator_reference=...` |

### 8.5 RF64 Deep Dive

RF64 (EBU Tech 3306) was designed to overcome the 4GB file size limitation of standard RIFF/WAV. It is the default output format when FFmpeg detects that the file would exceed 4GB.

**RF64 File Identification:**
- Standard WAV: bytes 0–3 = "RIFF", bytes 8–11 = "WAVE"
- RF64: bytes 0–3 = "RF64", bytes 8–11 = "WAVE"
- RF64 chunks use size = 0xFFFFFFFF (0xFFFFFFFF = ~4GB) as a sentinel value
- The `ds64` chunk provides the actual 64-bit sizes

**ds64 Chunk Structure (24 bytes minimum + table):**

| Field | Size | Type | Description |
|-------|------|------|-------------|
| chunk ID | 4 | FOURCC | "ds64" |
| chunk size | 4 | UINT32 | 24 + table_size |
| riffSizeHigh | 4 | UINT32 | Upper 32 bits of file size |
| riffSizeLow | 4 | UINT32 | Lower 32 bits of file size |
| dataSizeHigh | 4 | UINT32 | Upper 32 bits of data chunk size |
| dataSizeLow | 4 | UINT32 | Lower 32 bits of data chunk size |
| sampleCountHigh | 4 | UINT32 | Upper 32 bits of sample count |
| sampleCountLow | 4 | UINT32 | Lower 32 bits of sample count |
| tableLength | 4 | UINT32 | Number of entries in size table |
| table[] | variable | 3×UINT32 | Additional chunk sizes (chunk ID, size high, size low) |

**RF64 Detection and Handling:**
```bash
# Check file type
xxd input.wav | head -1
# RF64: 00000000: 5246 3432 0000 0000 0000 0000 0000 0000  RF64..........
# WAV:  00000000: 5249 4646 0000 0000 0000 0000 0000 0000  RIFF.........

# FFmpeg automatically handles RF64 on read
ffprobe input.rf64

# Force RF64 output for large files
ffmpeg -i large_input.wav -rf64 always large_output.wav
```

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seeking Architecture

WAV has **no native seek index**. Every seek operation requires a direct byte offset calculation from the sample position, which is only possible for constant-bitrate (CBR) audio where each sample frame occupies exactly `nBlockAlign` bytes.

**Byte Offset Formulas:**

```
sample_byte_offset = data_chunk_start_offset + (sample_number × nBlockAlign)

sample_number = (byte_offset - data_chunk_start_offset) / nBlockAlign

duration_seconds = data_chunk_size / nAvgBytesPerSec

duration_samples = data_chunk_size / nBlockAlign
```

**CBR Seeking Algorithm:**
```c
int wav_seek_time_ms(WAVFile *wav, uint32_t time_ms) {
    uint64_t target_sample = (uint64_t)(
        (uint64_t)time_ms * wav->format.nSamplesPerSec / 1000
    );
    if (target_sample >= wav->total_samples) return -1;

    uint64_t byte_offset = wav->data_chunk.file_offset + 8
                          + target_sample * wav->format.nBlockAlign;
    return fseek(wav->fp, byte_offset, SEEK_SET);
}
```

**VBR Considerations:**
Standard PCM WAV is always CBR because each sample occupies a fixed number of bytes. However, if custom chunks or metadata cause irregular spacing between chunks, byte offset calculation remains valid for the data region. For compressed WAV variants (ADPCM, MP3-in-WAV), seeking requires scanning or a seek table.

### 9.2 Building Seek Tables

For applications that need fast seeking (e.g., DAWs, media players), a seek table can be precomputed and stored using the `cue ` chunk combined with an index:

```c
// Build a simple seek table: one entry per N samples
typedef struct {
    uint64_t sample_position;
    uint64_t byte_offset;
} SeekTableEntry;

void build_seek_table(WAVFile *wav, uint32_t interval_samples,
                      SeekTableEntry **entries, size_t *num_entries) {
    *num_entries = (wav->total_samples + interval_samples - 1) / interval_samples;
    *entries = malloc(*num_entries * sizeof(SeekTableEntry));

    for (size_t i = 0; i < *num_entries; i++) {
        uint64_t sample = i * interval_samples;
        (*entries)[i].sample_position = sample;
        (*entries)[i].byte_offset = wav->data_chunk.file_offset + 8
                                   + sample * wav->format.nBlockAlign;
    }
}

SeekTableEntry* find_closest_seek_entry(SeekTableEntry *entries,
                                        size_t num_entries,
                                        uint64_t target_sample) {
    // Binary search for closest entry not exceeding target
    size_t lo = 0, hi = num_entries;
    while (lo < hi) {
        size_t mid = lo + (hi - lo) / 2;
        if (entries[mid].sample_position < target_sample) {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    return (lo > 0) ? &entries[lo - 1] : &entries[0];
}
```

### 9.3 Gapless Playback Considerations

WAV files do not natively carry encoder delay/padding metadata (unlike MP3's LAME tag or AAC's iTunSMPB). For gapless concatenation of multiple WAV files, the following approaches are used:

1. **Trimmed source files**: Remove silence from the beginning and end of each track before concatenation
2. **Cue point markers**: Use `cue ` chunks to define track boundaries within a single long recording
3. **Pre-roll detection**: Some DAWs scan a few milliseconds of audio before the intended start point to handle any DC offset or pre-ringing

For professional broadcast workflows (BWF), the `iXML` chunk can carry EBU R 128 loudness metadata that helps ensure seamless transitions between segments.

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

### 10.1 Streaming Architecture

WAV's chunked structure presents both advantages and challenges for streaming applications:

**Streaming-Friendly Properties:**
- Frames are self-contained and byte-aligned
- Sample rate and format are declared in the header before any audio data
- Sequential reading requires only forward seeking within the data chunk
- No entropy coding or frame interdependency (unlike MP3 bit reservoir)

**Streaming Challenges:**
- The complete file structure must be known before writing the RIFF header
- The `RIFF size` field at byte 4 requires knowing the total file size in advance
- For true streaming output, use chunked transfer encoding or pipe through a process that buffers and updates the header

**Streaming Solutions:**
```bash
# For HTTP streaming, serve WAV files with Range request support
# Most HTTP servers handle WAV range requests correctly because
# byte offsets are deterministic

# For RTP/UDP streaming, raw PCM is preferred over WAV wrapping
# because WAV headers add overhead and require pre-buffering

# Pipe WAV data without headers (raw PCM streaming)
ffmpeg -i input.wav -c:a pcm_s16le -f s16le - | \
  nc -l -p 5000

# Stream with WAV headers (delay before first audio)
# Use FFmpeg's stream-copy mode
ffmpeg -i input.wav -c:a copy -f wav - | \
  nc -l -p 5000
```

### 10.2 Real-Time Constraints

| Parameter | Typical Value | Impact |
|-----------|--------------|--------|
| Decode latency | ~0ms (no decode needed) | Audio playback can begin immediately after header |
| Header parsing | < 1ms | Fast initial response |
| Sample access | O(1) | Direct seek by byte offset |
| Memory for playback | Buffer: 1–5 seconds | Low memory requirement |
| I/O bandwidth | 1.4 Mbps (16-bit/44.1kHz/stereo) | Achievable on any network |

The primary latency in WAV playback comes from the audio buffer on the playback device, not from decoding overhead. PCM WAV has the lowest possible decode latency of any audio format, making it ideal for:
- Real-time audio processing pipelines
- Live sound reinforcement
- Low-latency monitoring applications
- Digital audio workstations (DAWs) that need immediate sample-accurate access

### 10.3 HTTP Streaming Behavior

Modern HTTP-based audio players (HLS, DASH, web browsers) typically do not use WAV as a streaming format because:
1. WAV files do not support byte-range seeking for VBR content (though WAV is always CBR)
2. The WAV header must be received before audio playback can begin
3. No support for adaptive bitrate switching

For HTTP streaming, WAV is used only as a download format, not as a progressive streaming format. The typical workflow:
1. Client initiates download
2. Server sends WAV header
3. Client begins playback as data streams in
4. Seeking is supported by calculating byte offset from sample position

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Channel Ordering in Interleaved Data

The `dwChannelMask` bitmask determines which speaker positions are present and defines their order in the interleaved sample stream. The channel order follows the numeric order of set bits in the mask (lowest bit first).

**Common Channel Orders:**

| Configuration | Channel Mask (hex) | Interleaved Channel Order |
|---------------|-------------------|--------------------------|
| Stereo | 0x0003 | [FL, FR] |
| 2.1 | 0x000B | [FL, FR, LFE] |
| 5.1 (standard) | 0x003F | [FL, FR, FC, LFE, BL, BR] |
| 5.1 (side) | 0x060F | [FL, FR, FC, LFE, SL, SR] |
| 7.1 | 0x00FF | [FL, FR, FC, LFE, FLC, FRC, BL, BR] |
| 7.1 (wide) | 0x06FF | [FL, FR, FC, LFE, SL, SR, BL, BR] |

**Extracting Channels from Interleaved Data:**
```c
// Extract a specific channel from interleaved multi-channel data
void extract_channel(const uint8_t *interleaved, size_t num_frames,
                     uint16_t nChannels, size_t bytes_per_sample,
                     uint8_t *channel_out, uint8_t channel_index) {
    for (size_t f = 0; f < num_frames; f++) {
        size_t sample_offset = (f * nChannels + channel_index) * bytes_per_sample;
        memcpy(channel_out + f * bytes_per_sample,
               interleaved + sample_offset, bytes_per_sample);
    }
}

// De-interleave to planar format
void deinterleave(const uint8_t *interleaved, size_t num_frames,
                  uint16_t nChannels, size_t bytes_per_sample,
                  uint8_t **planar_out) {
    for (uint16_t ch = 0; ch < nChannels; ch++) {
        for (size_t f = 0; f < num_frames; f++) {
            size_t src = (f * nChannels + ch) * bytes_per_sample;
            size_t dst = ch * num_frames * bytes_per_sample + f * bytes_per_sample;
            memcpy(planar_out[ch] + dst, interleaved + src, bytes_per_sample);
        }
    }
}

// Interleave from planar format
void interleave(const uint8_t **planar_in, size_t num_frames,
                uint16_t nChannels, size_t bytes_per_sample,
                uint8_t *interleaved_out) {
    for (size_t f = 0; f < num_frames; f++) {
        for (uint16_t ch = 0; ch < nChannels; ch++) {
            size_t src = ch * num_frames * bytes_per_sample + f * bytes_per_sample;
            size_t dst = (f * nChannels + ch) * bytes_per_sample;
            memcpy(interleaved_out + dst, planar_in[ch] + src, bytes_per_sample);
        }
    }
}
```

### 11.2 Channel Matrix Operations

**Stereo to 5.1 Expansion (standard ITU BS.775):**
```c
void stereo_to_5point1(const float *stereo_in, float *surround_out,
                       size_t num_frames) {
    for (size_t i = 0; i < num_frames; i++) {
        float L = stereo_in[i * 2 + 0];
        float R = stereo_in[i * 2 + 1];

        // Apply standard downmix matrix
        surround_out[i * 6 + 0] = L;                           // FL
        surround_out[i * 6 + 1] = R;                           // FR
        surround_out[i * 6 + 2] = 0.7071f * (L + R);            // FC (–3 dB)
        surround_out[i * 6 + 3] = 0;                            // LFE (bass management elsewhere)
        surround_out[i * 6 + 4] = 1.0f * L;                    // BL
        surround_out[i * 6 + 5] = 1.0f * R;                    // BR
    }
}
```

### 11.3 Validation of Multi-Channel Files

```c
int validate_channel_mask(WAVFormat *fmt) {
    uint32_t mask = fmt->dwChannelMask;
    uint32_t set_bits = __builtin_popcount(mask);

    // For extensible format, set bits must equal nChannels
    if (fmt->is_extensible && set_bits != fmt->nChannels) {
        fprintf(stderr, "Warning: channel mask has %u bits but %u channels\n",
                set_bits, fmt->nChannels);
        return 0;
    }

    // For non-extensible standard format, validate common configurations
    if (!fmt->is_extensible) {
        if (fmt->nChannels == 1 && mask != 0x0000 && mask != 0x0004) {
            fprintf(stderr, "Warning: mono with unexpected mask 0x%X\n", mask);
        }
        if (fmt->nChannels == 2 && mask != 0x0000 && mask != 0x0003) {
            fprintf(stderr, "Warning: stereo with unexpected mask 0x%X\n", mask);
        }
    }

    // Check for reserved bits set
    if (mask & 0xFFF00000) {
        fprintf(stderr, "Warning: channel mask has reserved bits set\n");
    }

    return 1;
}
```

---

## 12. KNOWN ISSUES, BUGS & EDGE CASES

### 12.1 The 4GB File Size Limit

Standard RIFF/WAV uses 32-bit size fields throughout, limiting individual chunk sizes to 4,294,967,295 bytes (~4GB). This affects the data chunk most severely, as high-resolution multi-channel audio can exceed this limit:

| Format | Channels | Sample Rate | Bit Depth | Bytes/Second | 4GB Limit |
|--------|----------|-------------|-----------|--------------|-----------|
| CD quality | 2 | 44.1 kHz | 16-bit | 176,400 | ~6.8 hours |
| CD quality | 2 | 44.1 kHz | 24-bit | 264,600 | ~4.5 hours |
| DVD audio | 6 | 48 kHz | 24-bit | 864,000 | ~1.4 hours |
| Studio 96kHz | 8 | 96 kHz | 24-bit | 2,304,000 | ~52 minutes |
| Studio 192kHz | 16 | 192 kHz | 32-bit float | 12,288,000 | ~10 minutes |

**RF64 Solution:** FFmpeg automatically switches to RF64 when the output would exceed 4GB. For maximum compatibility with legacy players, the file should be created as RF64 from the start.

### 12.2 20-bit PCM Edge Cases

20-bit PCM is the most problematic standard bit depth because it does not fit evenly into byte boundaries. The correct specification requires a 24-bit or 32-bit container with 20 valid bits:

**Correct 20-bit Specification:**
```
wBitsPerSample = 24 (container size)
wValidBitsPerSample = 20 (actual precision)
nBlockAlign = nChannels × 3
data samples: 3 bytes each, left-justified
```

**Incorrect Specifications Found in the Wild:**
```
wBitsPerSample = 20 (incorrect — violates byte alignment)
wBitsPerSample = 24, wValidBitsPerSample = 0 (ambiguous — assume 24-bit)
```

**Robust 20-bit Reading:**
```c
int32_t read_20bit_sample(const uint8_t *data) {
    // Read as 32-bit, then sign-extend from bit 19
    int32_t val = (int32_t)(data[0] | (data[1] << 8) | (data[2] << 16));
    // Sign extend from bit 19 (0-indexed)
    if (val & 0x80000) {
        val |= ~0x7FFFF;  // Extend sign to bits 20–31
    }
    return val;
}

void write_20bit_sample(uint8_t *data, int32_t sample) {
    // Clamp to 20-bit range
    if (sample > 524287) sample = 524287;
    if (sample < -524288) sample = -524288;
    // Pack as 3 bytes, left-justified
    data[0] = sample & 0xFF;
    data[1] = (sample >> 8) & 0xFF;
    data[2] = (sample >> 16) & 0xFF;
}
```

### 12.3 Invalid Chunk Size Handling

Corrupted WAV files often have incorrect chunk sizes. A robust parser must handle these cases:

**Recovery Strategies:**

1. **Chunk extends past file**: Reduce chunk size to fit within the file
2. **Chunk overlaps next chunk**: Trust the chunk boundary and skip to next chunk
3. **Missing padding byte**: Check if next chunk's FOURCC is readable; if not, backtrack
4. **fmt after data**: Non-compliant but tolerated — parse fmt from wherever it appears

```c
int validate_chunk_bounds(uint64_t chunk_start, uint32_t chunk_size,
                          uint64_t file_size) {
    uint64_t chunk_data_end = chunk_start + 8 + chunk_size;
    uint64_t total_end = chunk_data_end + (chunk_size % 2);  // padding

    if (chunk_data_end > file_size) {
        fprintf(stderr, "Warning: chunk extends past file\n");
        return 0;
    }

    // Check if next chunk's FOURCC is valid
    fseek(wav->fp, chunk_data_end, SEEK_SET);
    char next_id[4];
    fread(next_id, 1, 4, wav->fp);

    // Valid FOURCC bytes are all printable ASCII or known patterns
    int valid = 1;
    for (int i = 0; i < 4; i++) {
        unsigned char c = next_id[i];
        if (c < 32 && c != 0) valid = 0;  // Non-printable control chars
        if (c > 126) valid = 0;             // Non-ASCII
    }
    if (!valid) {
        fprintf(stderr, "Warning: possible missing padding byte\n");
    }

    return 1;
}
```

### 12.4 Byte Order Detection and Handling

```c
int detect_wav_byte_order(FILE *fp) {
    char id[4];
    fseek(fp, 0, SEEK_SET);
    fread(id, 1, 4, fp);

    if (memcmp(id, "RIFF", 4) == 0) return 0;   // Little-endian
    if (memcmp(id, "RIFX", 4) == 0) return 1;   // Big-endian

    fprintf(stderr, "Unknown byte order, not a valid WAV/RIFF file\n");
    return -1;
}

uint32_t read_wav_uint32(FILE *fp, int big_endian) {
    uint32_t val;
    fread(&val, 4, 1, fp);
    return big_endian ? be32toh(val) : le32toh(val);
}

uint16_t read_wav_uint16(FILE *fp, int big_endian) {
    uint16_t val;
    fread(&val, 2, 1, fp);
    return big_endian ? be16toh(val) : le16toh(val);
}
```

### 12.5 Malformed fmt Chunk Issues

**Issue: wBitsPerSample = 0**
- Some encoders write 0 for wBitsPerSample (ambiguous)
- Default to assuming 8 bits per byte for block alignment calculation

**Issue: nBlockAlign inconsistency**
- Some files have `nBlockAlign` that doesn't equal `nChannels × ceil(wBitsPerSample/8)`
- When in doubt, trust `nBlockAlign` for reading sample frames

**Issue: nAvgBytesPerSec mismatch**
- Some files have incorrect `nAvgBytesPerSec`
- Recalculate: `nAvgBytesPerSec = nSamplesPerSec × nBlockAlign`
- Use the recalculated value for duration verification

**Issue: fact chunk with wrong sample count**
- Some encoders write an incorrect `dwSampleLength` in fact chunk
- Cross-verify with: `data_chunk_size / nBlockAlign`

---

## 13. RE-ENCODING DEGRADATION

### 13.1 Understanding Re-encoding Loss

Unlike lossy formats (MP3, AAC, Opus), WAV/PCM is a **lossless container** — re-encoding PCM data to WAV preserves bit-exact samples with no quality loss. There is no "re-encoding degradation" in the traditional sense for PCM-to-PCM conversions.

However, quality degradation **can** occur in the following scenarios:

**Bit Depth Reduction (Dithering Required):**
When reducing bit depth (e.g., 24-bit → 16-bit), the quantization noise introduced must be shaped by dithering to prevent tonal artifacts. Simply truncating samples introduces a systematic distortion pattern.

| Source Bit Depth | Target Bit Depth | Dithering Required | Method |
|-----------------|-----------------|-------------------|--------|
| 32-bit float | 24-bit PCM | Recommended | Triangular PDF dither |
| 32-bit float | 16-bit PCM | Required | TPDF dither + noise shaping |
| 24-bit PCM | 16-bit PCM | Recommended | TPDF dither |
| 24-bit PCM | 16-bit PCM | Required for professional | TPDF + MBIT+ noise shaping |
| 16-bit PCM | 8-bit PCM | Required | Dithering + requantization |

```bash
# Correct 24-bit to 16-bit conversion with dithering
ffmpeg -i input_24bit.wav -c:a pcm_s24le output_24bit.wav  # No change

# Convert with dithering (using high-quality SRC for sample rate)
ffmpeg -i input_24bit.wav -af "aformat=sample_fmts=s16:dither_method=tpdf" \
  -c:a pcm_s16le output_16bit_dithered.wav

# Destructive 24-bit to 16-bit (no dithering — NOT recommended)
ffmpeg -i input_24bit.wav -c:a pcm_s16le output_16bit_no_dither.wav
```

**Sample Rate Conversion (Resampling):**
Changing sample rate requires resampling, which introduces interpolation. Low-quality resampling introduces aliasing and phase distortion:

| Resampling Quality | Filter Type | Aliasing | Phase Distortion |
|-------------------|-------------|----------|-----------------|
| Highest (soxr_vhq) | Linear-phase sinc | None | Minimal |
| High (soxr_hq) | Linear-phase | None | Minimal |
| Medium (SWr) | Intermediate | Low | Moderate |
| Low (basic linear) | None | Significant | None |
| Nearest-neighbor | None | Severe | None |

```bash
# High-quality resampling with soxr
ffmpeg -i input_44.1k.wav -af "aresample=48000:resampler=soxr" \
  -c:a pcm_s24le output_48k.wav

# Standard quality resampling
ffmpeg -i input_44.1k.wav -ar 48000 -c:a pcm_s24le output_48k.wav
```

### 13.2 Lossless Round-Trip Verification

Since WAV/PCM is bit-exact, any round-trip conversion (WAV → WAV) should produce identical audio data:

```bash
# Verify bit-exact round-trip (16-bit PCM)
ffmpeg -i original.wav -c:a pcm_s16le temp.wav
ffmpeg -i temp.wav -c:a pcm_s16le output.wav
diff <(xxd original.wav) <(xxd output.wav) && echo "Bit-exact match"

# Verify with MD5 checksum
ffmpeg -i original.wav -f hash -hash md5
ffmpeg -i output.wav -f hash -hash md5

# For floating-point WAV: expect near-identical, not bit-exact
# (floating-point operations may produce slightly different results)
```

### 13.3 Cross-Format Lossless Round-Trips

WAV ↔ FLAC, WAV ↔ ALAC, and WAV ↔ W64 are all lossless. Any audio data decoded from these formats and re-encoded to WAV produces bit-exact output (assuming identical sample format).

```bash
# FLAC → WAV → FLAC: should be bit-exact
ffmpeg -i original.flac -c:a pcm_s24le temp.wav
ffmpeg -i temp.wav -c:a flac -compression_level 8 reconstructed.flac

# Compare
ffmpeg -i original.flac -i reconstructed.flac -filter_complex \
  "md5" -f null - 2>&1 | grep md5

# If MD5 matches across both streams: bit-exact
```

---

## 14. CONVERSION GUIDE (DBpoweramp Context)

### 14.1 Converting FROM WAV

| Target Format | Recommended Command | Quality Notes | Metadata Preservation |
|---------------|---------------------|---------------|----------------------|
| **FLAC** (level 8) | `ffmpeg -i input.wav -c:a flac -compression_level 8 output.flac` | Lossless, ~50–60% size reduction | Full via Vorbis comments |
| **FLAC** (level 0) | `ffmpeg -i input.wav -c:a flac -compression_level 0 output.flac` | Fastest encoding, slightly larger | Full via Vorbis comments |
| **MP3** (320k CBR) | `ffmpeg -i input.wav -c:a libmp3lame -b:a 320k output.mp3` | Transparent at 320k | Via ID3v2.3 `-map_metadata 0` |
| **MP3** (V0 VBR) | `ffmpeg -i input.wav -c:a libmp3lame -q:a 0 output.mp3` | ~245k VBR, near-transparent | Via ID3v2.3 |
| **AAC** (256k) | `ffmpeg -i input.wav -c:a aac -b:a 256k output.m4a` | Good quality, wide support | Via MP4 metadata |
| **AAC** (VBR q2) | `ffmpeg -i input.wav -c:a aac -q:a 2 output.m4a` | Variable quality, ~190k avg | Via MP4 metadata |
| **Opus** (128k) | `ffmpeg -i input.wav -c:a libopus -b:a 128k output.opus` | Excellent at low bitrates | Via Opus comments |
| **AIFF** | `ffmpeg -i input.wav -c:a pcm_s16be output.aiff` | Bit-exact, big-endian | Full preservation |
| **ALAC** | `ffmpeg -i input.wav -c:a alac output.m4a` | Apple lossless | Full via MP4 metadata |
| **W64** | `ffmpeg -i input.wav -c:a pcm_s24le output.w64` | Sonic Foundry 64-bit |
| **CAF** | `ffmpeg -i input.wav -c:a pcm_s24le output.caf` | Core Audio Format |

### 14.2 Converting TO WAV

| Source Format | Recommended Command | Notes |
|---------------|---------------------|-------|
| **FLAC** | `ffmpeg -i input.flac -c:a pcm_s24le output.wav` | Specify target bit depth |
| **FLAC** (16-bit) | `ffmpeg -i input.flac -c:a pcm_s16le output.wav` | Reduce file size |
| **MP3** | `ffmpeg -i input.mp3 -c:a pcm_s16le output.wav` | Decode to PCM |
| **AAC/M4A** | `ffmpeg -i input.m4a -c:a pcm_s16le output.wav` | Lossless decode |
| **OGG Vorbis** | `ffmpeg -i input.ogg -c:a pcm_s16le output.wav` | Vorbis decode |
| **Opus** | `ffmpeg -i input.opus -c:a pcm_s16le output.wav` | Opus decode |
| **AIFF** | `ffmpeg -i input.aiff -c:a pcm_s16le output.wav` | Explicit PCM, big→LE |
| **W64** | `ffmpeg -i input.w64 -c:a pcm_s16le output.wav` | Sonic Foundry format |
| **RF64** | `ffmpeg -i input.rf64 output.wav` | Auto-detect RF64 |
| **APE** | `ffmpeg -i input.ape -c:a pcm_s24le output.wav` | Monkey's Audio decode |
| **TTA** | `ffmpeg -i input.tta -c:a pcm_s16le output.wav` | True Audio decode |

### 14.3 Lossless Round-Trip Verification

```bash
# Extract MD5 of original audio data (skip WAV header)
ffmpeg -i original.wav -f s16le -acodec pcm_s16le - | md5sum

# Extract MD5 of converted audio data
ffmpeg -i converted.wav -f s16le -acodec pcm_s16le - | md5sum

# Bit-exact file comparison
cmp original.wav converted.wav && echo "Files are identical"

# Detailed comparison with ffprobe
ffprobe -v error -show_entries stream=codec_name,sample_rate,channels,bits_per_sample \
  -of default=noprint_wrappers=1 original.wav
ffprobe -v error -show_entries stream=codec_name,sample_rate,channels,bits_per_sample \
  -of default=noprint_wrappers=1 converted.wav

# Cross-verify duration
ffprobe -v error -show_entries format=duration -of default=noprint_wrappers=1 input.wav
```

---

## 15. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | WAV Support | Notes |
|---------|----------|---------|------------|-------|
| **FFmpeg libavformat** | C | LGPL/GPL | Full | Reference WAV, RF64, W64 implementation |
| **libsndfile** | C | LGPL | Full | Multi-format audio I/O; handles 20/24-bit PCM, IEEE float, RF64 |
| **SoX (libsox)** | C | GPL/ISC | Full | Sound processing, format conversion, resampling |
| **libsamplerate** | C | BSD-2-Clause | — | High-quality sample rate conversion |
| **PortAudio** | C | MIT | Full | Cross-platform audio I/O, reads WAV files |
| **dr_wav** | C | Public Domain / CC0 | PCM only | Single-file header library, easy integration |
| **miniaudio** | C | MIT | Full | Single-file audio library, WAV decoding |
| **soundfile** | C/Python | BSD | Full | libsndfile bindings for Python |
| **audioread** | Python | MIT | Full | Cross-library audio reading for Python |
| **wave (stdlib)** | Python | PSF | PCM only | Built-in Python module for 8/16-bit WAV |
| **alsa-lib** | C | LGPL | Via file I/O | Linux audio development |
| **Xuggler** | Java | LGPL | Full | FFmpeg bindings for Java |
| **JAVE2** | Java | LGPL | Full | FFmpeg Java wrapper |
| **JAudiotagger** | Java | MPL/LGPL | Limited | Tag reading, not audio decoding |

### 15.1 dr_wav Quick Reference

dr_wav is a popular single-header library for WAV reading/writing:

```c
#define DR_WAV_IMPLEMENTATION
#include "dr_wav.h"

// Reading
drwav wav;
drwav_init_file(&wav, "input.wav", NULL);

drwav_int16 *pDecodedData = drwav_malloc_and_read_s16(&wav, NULL);
if (pDecodedData) {
    // pDecodedData contains interleaved int16 samples
    size_t numSamples = wav.totalSampleCount;
    drwav_free(pDecodedData, NULL);
}
drwav_uninit(&wav);

// Writing
drwav wav_out;
drwav_data_format format = {
    .container     = drwav_container_riff,
    .format        = DR_WAVE_FORMAT_PCM,
    .channels      = 2,
    .sampleRate    = 44100,
    .bitsPerSample = 16
};
drwav_init_file_write(&wav_out, "output.wav", &format, NULL);

drwav_int16 buffer[1024];
// ... fill buffer ...
drwav_write(&wav_out, buffer, 1024);
drwav_uninit(&wav_out);
```

---

## 16. RELEVANT SPECIFICATIONS & FURTHER READING

### Core Specifications

| Document | Organization | Description |
|----------|--------------|-------------|
| ITU-R BS.2088 | ITU | BW64 file format — latest BWF with ADM support |
| EBU Tech 3285 | EBU | Broadcast Wave Format (BWF) v2 — mandatory for EBU members |
| EBU Tech 3306 | EBU | RF64: Extended file format for files > 4GB |
| SMPTE ST 330M | SMPTE | UMID (Unique Material Identifier) |
| SMPTE 336M | SMPTE | KLV Data Encoding for metadata |
| Microsoft RIFF | Microsoft | RIFF container specification |
| AES3-2015 | AES | Professional audio digital interface standard |

### BWF Supplements

| Document | Description |
|----------|-------------|
| EBU Tech 3285-s1 | MPEG audio in BWF files |
| EBU Tech 3285-s2 | Unicode use in BWF metadata |
| EBU Tech 3285-s3 | Peak Envelope Chunk (levl) specification |
| EBU Tech 3285-s4 | Link Chunk for multi-file sequences |
| EBU Tech 3285-s5 | Cue Chunk enhancements |
| EBU Tech 3285-s6 | Data Chunk Extension mechanism |

### Reference Documentation

- **McGill University WAVE Specification**: `https://www.mmsp.ece.mcgill.ca/Documents/AudioFormats/WAVE/WAVE.html` — authoritative technical reference for WAV format details
- **Microsoft RIFF Documentation**: `https://learn.microsoft.com/en-us/windows/win32/xaudio2/resource-interchange-file-format--riff-` — official Microsoft reference
- **Kaitai WAVE Format**: `https://formats.kaitai.io/wav/` — visual format explorer with binary layout
- **ReplayGain WAV Extension**: `https://replaygain.hydrogenaud.io/file_format_wav.html` — ReplayGain in WAV files
- **EBU BWF Guidelines**: `https://tech.ebu.ch/docs/tech/tech3285.pdf` — official EBU BWF specification PDF
- **FFmpeg WAV Demuxer**: `https://github.com/FFmpeg/FFmpeg/blob/master/libavformat/wavdec.c` — reference implementation
- **FFmpeg WAV Muxer**: `https://github.com/FFmpeg/FFmpeg/blob/master/libavformat/wavenc.c` — reference implementation
- **dr_wav Library**: `https://github.com/mackron/dr_libs` — single-file WAV reader/writer in C

---

## 17. IMPLEMENTATION CHECKLIST (for the Converter Developer)

This checklist covers every aspect that a production-quality WAV encoder/decoder must handle:

### Pre-Reading Phase
- [ ] Open file in binary mode (`"rb"` for reading, `"wb"` for writing)
- [ ] Read and validate RIFF/WAVE header (bytes 0–11): verify `"RIFF"` and `"WAVE"` FOURCCs
- [ ] Check for `"RIFX"` variant and byte-swap all subsequent integers
- [ ] Verify chunk order: fmt chunk must appear before data chunk
- [ ] Build a complete chunk lookup table (hash map by FOURCC)
- [ ] Handle unknown chunks: copy them verbatim during conversion

### Format Parsing
- [ ] Parse fmt chunk: extract `wFormatTag`, `nChannels`, `nSamplesPerSec`
- [ ] Validate `nBlockAlign`: must equal `nChannels × ceil(wBitsPerSample/8)`
- [ ] Validate `nAvgBytesPerSec`: must equal `nSamplesPerSec × nBlockAlign`
- [ ] Handle standard PCM (`wFormatTag = 0x0001`)
- [ ] Handle IEEE float (`wFormatTag = 0x0003` or SubFormat = {03...})
- [ ] Handle extensible format (`wFormatTag = 0xFFFE`): parse cbSize, wValidBitsPerSample, dwChannelMask, SubFormat GUID
- [ ] Extract `wValidBitsPerSample` for non-byte-aligned bit depths (20-bit)
- [ ] Extract `dwChannelMask` for multi-channel layouts
- [ ] Detect byte order: RIFF = little-endian, RIFX = big-endian
- [ ] Verify fmt chunk size is valid: 16 (PCM), 18 (float), or 40 (extensible)

### Audio Data Reading
- [ ] Locate data chunk offset and size
- [ ] Calculate total sample count: `samples = data_chunk_size / nBlockAlign`
- [ ] Calculate duration: `duration_sec = samples / sample_rate`
- [ ] Read interleaved samples into appropriately sized buffer
- [ ] Handle padding byte if data chunk size is odd
- [ ] Verify fact chunk `dwSampleLength` if present (cross-check with calculation)

### Metadata Handling
- [ ] Parse LIST/INFO chunks: iterate through sub-chunks, extract each INFO field
- [ ] Map INFO subchunk IDs to standardized tag names
- [ ] Handle null-terminated ASCII strings (ZSTR) correctly
- [ ] Parse bext chunk: detect version (0, 1, or 2) from chunk size
- [ ] Extract bext Description (256 bytes), Originator (32 bytes), OriginationDate (10 bytes), OriginationTime (8 bytes)
- [ ] Extract TimeReference (8 bytes: high + low uint32) if present
- [ ] Extract UMID (64 bytes) if bext version ≥ 1
- [ ] Extract loudness metadata (LoudnessValue, LoudnessRange, MaxTruePeakLevel) if version = 2
- [ ] Parse cue chunks: build cue point table with sample offsets
- [ ] Parse labl chunks: associate text labels with cue point IDs
- [ ] Parse smpl chunks: extract sampler information, loop definitions
- [ ] Parse levl chunks: extract peak envelope data for normalization tools
- [ ] Preserve all unknown chunks during conversion

### Format Validation
- [ ] Verify block alignment: `nBlockAlign = nChannels × ceil(wBitsPerSample/8)`
- [ ] Verify avg bytes per second: `nAvgBytesPerSec = nSamplesPerSec × nBlockAlign`
- [ ] Validate channel mask: number of set bits equals `nChannels` for extensible format
- [ ] Check for 20-bit PCM edge cases: expect container ≥ 20 bits
- [ ] Detect invalid `wBitsPerSample = 0` and default to byte-aligned assumption
- [ ] Detect and report invalid chunk sizes (chunk extends past file)
- [ ] Handle missing padding byte gracefully (check next FOURCC validity)
- [ ] Verify RIFF size field matches actual file size

### Writing Phase
- [ ] Write RIFF header: `"RIFF"` FOURCC, correct file size, `"WAVE"` form type
- [ ] Write fmt chunk with all parameters correctly packed
- [ ] For extensible format: include `cbSize=22`, `wValidBitsPerSample`, `dwChannelMask`, 16-byte SubFormat GUID
- [ ] Write fact chunk if required (non-PCM formats)
- [ ] Write bext chunk for BWF output with correct version field
- [ ] Write LIST/INFO chunks: correct LIST header ( FOURCC + size + "INFO"), each INFO subchunk with correct FOURCC, size, null-terminated data
- [ ] Write data chunk: `"data"` FOURCC, audio byte count, interleaved samples
- [ ] Add padding byte (0x00) if data chunk size is odd
- [ ] Handle > 4GB files with RF64: write `"RF64"` header, ds64 chunk with 64-bit sizes, set data chunk size to 0xFFFFFFFF
- [ ] Update RIFF size field after all chunks are written

### Seeking & Playback
- [ ] Calculate byte offset: `offset = data_start + sample_number × nBlockAlign`
- [ ] Implement seek using `fseek()` or equivalent
- [ ] Read samples from seeked position
- [ ] Handle end-of-file gracefully (fewer samples than expected)
- [ ] Support reverse seeking (from end of file): `SEEK_END`
- [ ] Implement binary search seek table for fast seeking in large files

### Multi-Channel Considerations
- [ ] Interleave channels in correct order per `dwChannelMask` bit positions
- [ ] Validate channel mask: set bit count must equal channel count (extensible)
- [ ] Support common configurations: mono, stereo, 5.1, 7.1, 7.1.4
- [ ] Handle custom channel layouts with arbitrary mask values
- [ ] Implement channel extraction (de-interleaving) for processing
- [ ] Implement channel insertion (re-interleaving) for output

### Lossless Metadata Preservation
- [ ] Copy all unknown chunks during format conversion (preserves custom metadata)
- [ ] Map source metadata to appropriate destination chunks
- [ ] Preserve BWF bext if input has it (including version number)
- [ ] Preserve UMID, loudness metadata during BWF-to-WAV conversions
- [ ] Preserve cue points and markers (convert sample offsets if sample rate changes)
- [ ] Preserve sampler information (smpl chunk)
- [ ] Preserve peak envelope data (levl chunk)
- [ ] Test complete round-trip: input → output → compare byte-exact

### Bit Depth Conversion
- [ ] When reducing bit depth: apply triangular PDF (TPDF) dithering
- [ ] When reducing bit depth: apply noise shaping for professional applications
- [ ] When reducing bit depth from float: apply dithering before integer quantization
- [ ] When increasing bit depth: zero-pad LSBs (lossless expansion)
- [ ] Avoid truncation without dithering in any production conversion

### Sample Rate Conversion
- [ ] Use high-quality resampler (soxr or similar) for SRC
- [ ] Avoid low-quality linear interpolation for professional audio
- [ ] When changing sample rate: recalculate all cue point sample offsets
- [ ] Update `nSamplesPerSec` in fmt chunk after SRC

### Error Handling
- [ ] Handle truncated files (incomplete chunks at end of file)
- [ ] Handle corrupted chunk sizes (claimed size > actual file data)
- [ ] Handle missing required chunks (fmt or data not found)
- [ ] Handle unsupported format tags gracefully with informative error
- [ ] Validate all numeric fields are within expected ranges
- [ ] Handle empty data chunks (zero-length audio)
- [ ] Handle files with extra bytes after last chunk (extra padding or garbage)

### Performance
- [ ] Use memory-mapped I/O (`mmap`) for large files (> 100 MB)
- [ ] Buffer audio data for streaming playback (buffer 1–5 seconds)
- [ ] Avoid scanning entire file for simple seek operations (direct calculation)
- [ ] Consider SIMD-optimized sample format conversion (SSE/AVX for int-to-float)
- [ ] Use 64-bit file offset calculations for files approaching 4GB
- [ ] Pre-allocate output buffers to exact size to avoid reallocation
