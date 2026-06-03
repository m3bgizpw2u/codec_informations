# RIFF WAV — Deep Technical Reference
> **Category:** Container
> **File Extensions:** `.wav`, `.wav`, `.bwf` (Broadcast Wave), `.rf64`
> **MIME Types:** `audio/x-wav`, `audio/wav`, `audio/vnd.wave`
> **Standardization Body:** Microsoft (RIFF/WAVE), EBU (BWF)
> **Primary Specification:** Microsoft Multimedia Programming Interface and Data Specifications 1.0; EBU Tech 3285 (BWF); EBU Tech 3306 (RF64)
> **Patent Status:** Patent-free
> **License:** Open (Microsoft specs published openly)
> **Current Version:** BWF Version 2 (EBU R98), RF64 Version 1 (EBU Tech 3306)
> **Active Development:** Yes — maintained by EBU and ITU-R (RF64/BW64)

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Microsoft Corporation, in collaboration with IBM
- **Year Created:** 1991 (RIFF/WAVE), 1997 (BWF by EBU), 2007 (RF64 by EBU)
- **Original Purpose:** Store digital audio on IBM PC compatibles and in broadcast environments
- **Problem with Predecessors:** Existing audio file formats lacked standardization, chunk extensibility, and support for non-PCM audio. RIFF provided a generic container framework extensible to any data type.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| WAVE 1.0 | 1991 | Initial release, PCM only, basic fmt/data chunks |
| WAVE 2.0 | 1993 | Added WAVEFORMATEX, extensible format descriptor |
| WAVEFORMATEXTENSIBLE | 1999 | GUID-based subformat identification, channel masks, >16-bit samples |
| BWF (EBU R98) | 1997 | Added bext chunk for broadcast metadata |
| BWF Version 2 | 2001 | Enhanced broadcast extensions, ds64 precursor |
| RF64 (EBU Tech 3306) | 2007 | 64-bit file and data sizes, ds64 chunk, 4GB+ support |
| BW64 (ITU-R BS.2088) | 2015 | ITU's equivalent of RF64 |

### 1.3 Current Adoption
- **Primary use cases today:** Professional audio recording, broadcast workflows, CD-quality archival, game audio, Windows system sounds
- **Platforms with native support:** Windows (native), macOS (QuickTime), Linux (via FFmpeg/libav), iOS/Android (MediaPlayer framework)
- **Major services using this format:** Broadcast stations (BWF), game developers, podcast production, audio editors
- **Hardware support:** Nearly universal — virtually every audio device reads WAV
- **Status:** Dominant for uncompressed/intermediate audio

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 RIFF Framework
RIFF (Resource Interchange File Format) is a tagged container format. Every element is a **chunk** with a FOURCC identifier. WAVE is one specific RIFF form type.

```
RIFF[WAVE]
 ├── fmt  (format descriptor)
 ├── fact (sample count — required for non-PCM)
 ├── data  (audio samples)
 ├── cue   (cue point list)
 ├── plst  (playlist — CD track indices)
 ├── list→adtl (associated data list: labl, note, ltxt, file, cue)
 ├── bext  (BWF broadcast extension)
 ├── iXML  (BWF XML metadata)
 ├── ds64  (RF64 size extension)
 ├── Junk  (padding/reserved space)
 └── [other chunks]
```

### 2.2 High-Level File Layout
```
┌─────────────────────────────────────────┐
│  RIFF / RF64  (file header)            │  12 bytes
│  ├── chunk ID: "RIFF" or "RF64"        │
│  ├── chunk size (32-bit or -1)         │
│  └── form type: "WAVE"                │
├─────────────────────────────────────────┤
│  fmt  (format chunk)                   │  24+ bytes (minimum)
│  ├── WAVEFORMATEX or WAVEFORMATEXTENSIBLE
├─────────────────────────────────────────┤
│  fact (sample count)                   │  4 bytes — required for non-PCM
│  └── total sample count                │
├─────────────────────────────────────────┤
│  data (audio data)                     │  variable
│  └── raw PCM or compressed audio frames
├─────────────────────────────────────────┤
│  [bext] (BWF broadcast extension)       │  602+ bytes
├─────────────────────────────────────────┤
│  [ds64] (RF64 size table)              │  32+ bytes
├─────────────────────────────────────────┤
│  [cue]  (cue point list)               │  variable
├─────────────────────────────────────────┤
│  [list→adtl] (associated data list)    │  variable
│  ├── labl (labels)                     │
│  ├── note (notes)                      │
│  ├── ltxt (labelled text)              │
│  └── file (referenced file)            │
├─────────────────────────────────────────┤
│  [JUNK] (padding/reserved)             │  28+ bytes
└─────────────────────────────────────────┘
```

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       52 49 46 46     "RIFF"    RIFF form identifier (standard)
0x0000  4       52 46 36 34     "RF64"    RIFF form identifier (RF64 extended)
0x0004  4       XX XX XX XX     —         File size minus 8 (32-bit), or -1 (0xFFFFFFFF) for RF64
0x0008  4       57 41 56 45     "WAVE"    Form type: WAVE format
```

For RF64 files, the `RIFF` FOURCC is replaced by `RF64` and the file size at offset 0x0004 is set to -1 (0xFFFFFFFF). The actual 64-bit file size is stored in the mandatory `ds64` chunk.

### 3.2 Chunk Structure (Generic RIFF Chunk)
Every chunk follows this exact layout:

```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------------
0x0000   4B      Chunk ID (ckID)        FOURCC      ASCII four-character code (e.g., "fmt ", "data")
0x0004   4B      Chunk Size (ckSize)    uint32 LE   Size of chunk data in bytes (excludes ID + size fields)
0x0008   N       Chunk Data              BYTE[]      Actual data; padded to WORD (2-byte) boundary
```

**Critical Rules:**
- `ckSize` does NOT include the 8-byte header (ckID + ckSize), nor any padding byte
- If `ckSize` is odd, a single padding byte (0x00) is appended after the data
- Reading code must handle the padding byte — skip it before reading the next chunk

### 3.3 RIFF Form Header Layout
```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------------
0x0000   4B      RIFF/RF64 ID           FOURCC      "RIFF" (0x52494646) or "RF64" (0x52463634)
0x0004   4B      File Size              uint32 LE   Size of rest of file (excl. 8 header bytes)
                                         OR -1 (0xFFFFFFFF) if RF64 with ds64
0x0008   4B      WAVE Form Type         FOURCC      "WAVE" (0x57415645)
```

### 3.4 Complete WAVEFORMATEX Structure (in fmt Chunk)
The `fmt` chunk contains a WAVEFORMATEX structure. This is the foundational audio descriptor for all WAV files.

```
Offset   Size    Field Name              Type         Description
-------  ------  ----------------------  -----------  ---------------------------------
0x0000   2B      wFormatTag              uint16 LE    Format type identifier
                                             0x0001 = WAVE_FORMAT_PCM
                                             0x0003 = WAVE_FORMAT_IEEE_FLOAT
                                             0x0006 = WAVE_FORMAT_ALAW
                                             0x0007 = WAVE_FORMAT_MULAW
                                             0x0011 = WAVE_FORMAT_IMA_ADPCM
                                             0x0022 = WAVE_FORMAT_MP3
                                             0x0040 = WAVE_FORMAT_WMAVOICE
                                             0x0061 = WAVE_FORMAT_G726_ADPCM
                                             0x0062 = WAVE_FORMAT_G722
                                             0x0063 = WAVE_FORMAT_G726
                                             0x0064 = WAVE_FORMAT_G726LE
                                             0x0065 = WAVE_FORMAT_ATRAC3
                                             0x0071 = WAVE_FORMAT_G729A
                                             0x007F = WAVE_FORMAT_GSM610
                                             0x0080 = WAVE_FORMAT_GSM620
                                             0x0081 = WAVE_FORMAT_ATRAC3P
                                             0x008A = WAVE_FORMAT_DOLBY_AC3
                                             0x00FF = WAVE_FORMAT_AAC
                                             0x0110 = WAVE_FORMAT_WMAV1
                                             0x0160 = WAVE_FORMAT_WMAV2
                                             0x0130 = WAVE_FORMAT_WMAV3
                                             0xFFFE = WAVE_FORMAT_EXTENSIBLE (use SubFormat GUID)
                                             0xFFFA = WAVE_FORMAT_MP3 (alternate)
                                             0xFFFB = WAVE_FORMAT_MP3 (alternate)
0x0002   2B      nChannels              uint16 LE    Number of audio channels (1–65535)
0x0004   4B      nSamplesPerSec         uint32 LE    Sample rate in Hz
0x0008   4B      nAvgBytesPerSec        uint32 LE    Bytes per second (for buffer sizing)
                                             PCM: nSamplesPerSec × nBlockAlign
                                             MP3: from bitrate
0x000C   2B      nBlockAlign            uint16 LE    Bytes per sample block
                                             PCM: nChannels × (bitsPerSample/8)
                                             MP3: from encoding
0x000E   2B      wBitsPerSample         uint16 LE    Bits per sample (container size)
                                             PCM: 8, 16, 20, 24, or 32
                                             For compressed formats: size of a sample frame
0x0010   2B      cbSize                 uint16 LE    Extra format bytes (0 for PCM)
                                             IMA ADPCM: 2  (contains wSamplesPerBlock)
                                             MS ADPCM: 32  (contains adapter table)
                                             WAVEFORMATEXTENSIBLE: 22
                                             MP3: 0 or 12
[variable]  N    Extra format data       BYTE[]       Format-specific additional fields
```

**Total minimum size for PCM WAVEFORMATEX: 16 bytes (offsets 0x0000–0x000F)**
**Total size for WAVEFORMATEXTENSIBLE: 40 bytes (16 base + 2 cbSize + 22 extensible)**

### 3.5 WAVEFORMATEXTENSIBLE Structure (in fmt Chunk)
WAVEFORMATEXTENSIBLE is used when:
- More than 2 channels are present
- Sample resolution exceeds 16 bits
- A specific subformat GUID is needed
- A channel mask must be specified

```
Offset   Size    Field Name              Type         Description
-------  ------  ----------------------  -----------  ---------------------------------
0x0000   16B     WAVEFORMATEX base      —            Same as above, but wFormatTag=0xFFFE
0x0010   2B      cbSize                  uint16 LE    MUST be at least 22 (0x16)
0x0012   2B      wValidBitsPerSample     uint16 LE    Valid bits in container (≤ container size)
                                              OR wSamplesPerBlock (for some compressed formats)
                                              OR wReserved (0x0000) if unused
0x0014   4B      dwChannelMask           uint32 LE    Speaker position bitmask
0x0018   16B     SubFormat               GUID         16-byte little-endian GUID

SubFormat GUID values (from KSDATAFORMAT_SUBTYPE_* in ksmedia.h):
  PCM:            {00000001-0000-0010-8000-00AA00389B71}  (KSDATAFORMAT_SUBTYPE_PCM)
  IEEE Float:     {00000003-0000-0010-8000-00AA00389B71}  (KSDATAFORMAT_SUBTYPE_IEEE_FLOAT)
  A-law:          {00000006-0000-0010-8000-00AA00389B71}  (KSDATAFORMAT_SUBTYPE_ALAW)
  mu-law:         {00000007-0000-0010-8000-00AA00389B71}  (KSDATAFORMAT_SUBTYPE_MULAW)
  IMA ADPCM:      {00000011-0000-0010-8000-00AA00389B71}  (KSDATAFORMAT_SUBTYPE_ADPCM)
  WMA:            {00000161-0000-0010-8000-00AA00389B71}  (KSDATAFORMAT_SUBTYPE_MPEG)
  ATRAC3:         {00000170-0000-0010-8000-00AA00389B71}
  MP3:            {00000055-0000-0010-8000-00AA00389B71}  (KSDATAFORMAT_SUBTYPE_MPEG)
  Dolby AC3:      {00002000-0000-0010-8000-00AA00389B71}
```

**GUID byte layout (little-endian):**
```
Byte  0–3:  Data1   (uint32 LE) — first 4 bytes of GUID
Byte  4–5:  Data2   (uint16 LE)
Byte  6–7:  Data3   (uint16 LE)
Byte  8–15: Data4   (uint8[8])
```

### 3.6 Channel Mask (dwChannelMask) — Speaker Positions
The `dwChannelMask` field is a 32-bit bitmask defining which speaker positions are present:

| Bit | Position | Description |
|-----|----------|-------------|
| 0   | SPEAKER_FRONT_LEFT | Front Left |
| 1   | SPEAKER_FRONT_RIGHT | Front Right |
| 2   | SPEAKER_FRONT_CENTER | Front Center |
| 3   | SPEAKER_LOW_FREQUENCY | LFE (Subwoofer) |
| 4   | SPEAKER_BACK_LEFT | Back Left |
| 5   | SPEAKER_BACK_RIGHT | Back Right |
| 6   | SPEAKER_FRONT_LEFT_OF_CENTER | Front Left of Center |
| 7   | SPEAKER_FRONT_RIGHT_OF_CENTER | Front Right of Center |
| 8   | SPEAKER_BACK_CENTER | Back Center |
| 9   | SPEAKER_SIDE_LEFT | Side Left |
| 10  | SPEAKER_SIDE_RIGHT | Side Right |
| 11  | SPEAKER_TOP_CENTER | Top Center |
| 12  | SPEAKER_TOP_FRONT_LEFT | Top Front Left |
| 13  | SPEAKER_TOP_FRONT_CENTER | Top Front Center |
| 14  | SPEAKER_TOP_FRONT_RIGHT | Top Front Right |
| 15  | SPEAKER_TOP_BACK_LEFT | Top Back Left |
| 16  | SPEAKER_TOP_BACK_CENTER | Top Back Center |
| 17  | SPEAKER_TOP_BACK_RIGHT | Top Back Right |
| 18–31 | Reserved | |

**Common channel mask values:**
| Channels | Mask (hex) | Layout |
|----------|-----------|--------|
| 1 (Mono) | 0x0004 | FC |
| 2 (Stereo) | 0x0003 | FL, FR |
| 2.1 | 0x000B | FL, FR, LFE |
| 4 (Quad) | 0x0033 | FL, FR, RL, RR |
| 5.1 | 0x003F | FL, FR, FC, LFE, RL, RR |
| 7.1 | 0x00FF | FL, FR, FC, LFE, RL, RR, FLC, FRC |
| 7.1 (surround) | 0x063F | FL, FR, FC, LFE, BL, BR, SL, SR |

### 3.7 All Supported Audio Formats in WAV

| Format Tag | Name | Bits/Sample | Channels | Compressed | Requires fact Chunk |
|-----------|------|-------------|----------|------------|--------------------|
| 0x0001 | PCM | 8, 16, 20, 24, 32 | 1–N | No | No |
| 0x0003 | IEEE Float | 32, 64 | 1–N | No | No |
| 0x0006 | A-law (G.711) | 8 | 1–N | Yes | Yes |
| 0x0007 | mu-law (G.711) | 8 | 1–N | Yes | Yes |
| 0x0011 | IMA ADPCM | 4 | 1–N | Yes | Yes |
| 0x0022 | MP3 | Variable | 1–2 | Yes | Yes |
| 0x0023 | MPEG | Variable | 1–N | Yes | Yes |
| 0x0040 | WMA Voice | Variable | 1 | Yes | Yes |
| 0x0061 | G.726 ADPCM | 4 | 1 | Yes | Yes |
| 0x0062 | G.722 | 4 | 1 | Yes | Yes |
| 0x0063 | G.726 | 4 | 1 | Yes | Yes |
| 0x0065 | ATRAC3 | Variable | 1–2 | Yes | Yes |
| 0x0071 | G.729A | Variable | 1 | Yes | Yes |
| 0x008A | Dolby AC3 | Variable | 1–6 | Yes | Yes |
| 0x00FF | AAC | Variable | 1–N | Yes | Yes |
| 0x0110 | WMAV1 | Variable | 1–N | Yes | Yes |
| 0x0130 | WMAV3 | Variable | 1–N | Yes | Yes |
| 0x0160 | WMAV2 | Variable | 1–N | Yes | Yes |
| 0xFFFE | WAVEFORMATEXTENSIBLE | Any | 1–N | Varies | Varies |

### 3.8 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Unsigned integer (PCM) | Yes | Range: 0–255, center at 128 |
| 16-bit | Signed integer (PCM) | Yes | Standard CD audio |
| 20-bit | Signed integer (PCM) | Yes | Stored in 24-bit or 32-bit container |
| 24-bit | Signed integer (PCM) | Yes | Common in professional audio |
| 32-bit | Signed integer (PCM) | Yes | High-resolution standard |
| 32-bit | IEEE float | Yes | Range: -1.0 to +1.0 |
| 64-bit | IEEE float (double) | Yes | Rare in practice |

#### Supported Sample Rates
| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 8000 | Telephone | Yes | |
| 11025 | — | Yes | |
| 16000 | Wideband voice | Yes | |
| 22050 | — | Yes | |
| 32000 | Broadcast | Yes | |
| 44100 | CD audio | Yes | Most common |
| 48000 | Professional | Yes | Standard video |
| 88200 | 2× CD | Yes | |
| 96000 | High-res | Yes | DVD-Audio |
| 176400 | 4× CD | Yes | |
| 192000 | High-res max | Yes | DVD-Audio, BD |
| 352800 | DXD | Yes | Pro audio |
| 384000 | Ultra-high-res | Yes | |

### 3.9 The fact Chunk
The `fact` chunk stores the total number of audio samples per channel. It is **required** for all non-PCM formats (MP3, IEEE Float, ADPCM, AAC, WMA, AC3, etc.) and optional for PCM.

```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------------
0x0000   4B      dwSampleLength          uint32 LE   Total number of samples per channel
```

**Byte layout:**
```
52 49 46 46           ; "RIFF" chunk ID
XX XX XX XX           ; RIFF chunk size (rest of file)
57 41 56 45           ; "WAVE" form type
66 6D 74 20           ; "fmt " chunk ID
10 00 00 00           ; fmt chunk size = 16
01 00                 ; wFormatTag = PCM
02 00                 ; nChannels = 2
44 AC 00 00           ; nSamplesPerSec = 44100
10 B1 02 00           ; nAvgBytesPerSec = 176400
04 00                 ; nBlockAlign = 4
20 00                 ; wBitsPerSample = 32
66 61 63 74           ; "fact" chunk ID
04 00 00 00           ; fact chunk size = 4
00 70 05 00           ; dwSampleLength = 352000 (4 min @ 44.1kHz stereo)
64 61 74 61           ; "data" chunk ID
00 70 05 00           ; data chunk size = 352000 bytes
[audio samples...]
```

### 3.10 Cue Chunk (cue)
The `cue` chunk stores cue points for seeking, silence detection, and track boundary markers.

```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------------
0x0000   4B      cue ID                 FOURCC      "cue " (0x63756520)
0x0004   4B      chunk size             uint32 LE   Size of rest of cue chunk
0x0008   4B      dwCuePoints            uint32 LE   Number of cue points (N)
[cue point array — N entries of 24 bytes each]
```

**Cue Point Structure (24 bytes each):**
```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------------
0x0000   4B      dwName/ID              uint32 LE   Unique identifier for this cue point
0x0004   4B      dwPosition             uint32 LE   Position in samples from file start
0x0008   4B      fccChunk               FOURCC      Chunk ID this cue refers to (usually "data")
0x000C   4B      dwChunkStart           uint32 LE   Byte offset to start of containing chunk
0x0010   4B      dwBlockStart           uint32 LE   Byte offset within chunk to start of block
0x0014   4B      dwSampleOffset         uint32 LE   Byte offset to sample (if PCM) or
                                                      frame (if compressed)
```

### 3.11 plst (Playlist) Chunk
The playlist chunk defines segments of the audio for CD track simulation within a single WAV file.

```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------------
0x0000   4B      plst ID                FOURCC      "plst" (0x706C7374)
0x0004   4B      chunk size             uint32 LE   Size of rest of chunk
0x0008   4B      dwSegments             uint32 LE   Number of segments
[segment array]
```

**Playlist Segment Structure (16 bytes each):**
```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------------
0x0000   4B      dwIdentifier           uint32 LE   Cue point ID this segment starts at
0x0004   4B      dwLength               uint32 LE   Length in samples
0x0008   4B      dwSkipIdentifier       uint32 LE   Cue point ID to skip to (0 = none)
0x000C   4B      dwPlayIdentifier       uint32 LE   Cue point ID for playback override
```

### 3.12 LIST → adtl Chunk Hierarchy
The associated data list contains metadata annotations linked to cue points.

```
LIST chunk ID:    "list" (0x4C495354)
LIST form type:   "adtl" (0x6164746C)
```

**labl (Label) Chunk:**
```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------------
0x0000   4B      labl ID                FOURCC      "labl" (0x6C61626C)
0x0004   4B      chunk size             uint32 LE   Size of rest
0x0008   4B      dwName                  uint32 LE   Cue point ID this label belongs to
0x000C   N       bText                   char[]      Null-terminated label string
```

**note (Note) Chunk:** Same structure as labl, ID = "note" (0x6E6F7465), for longer annotations.

**ltxt (Labelled Text) Chunk:**
```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------------
0x0000   4B      ltxt ID                FOURCC      "ltxt" (0x6C747874)
0x0004   4B      chunk size             uint32 LE   Size of rest
0x0008   4B      dwIdentifier           uint32 LE   Cue point ID
0x000C   4B      dwSampleLength         uint32 LE   Sample count for this region
0x0010   4B      dwCountry              uint16 LE   Country code
0x0012   4B      dwLanguage            uint16 LE   Language code
0x0014   4B      dwDialect              uint16 LE   Dialect code
0x0016   4B      dwCodePage             uint16 LE   Character encoding code page
0x0018   N       bText                   char[]      Null-terminated text
```

**file (Referenced File) Chunk:**
```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------------
0x0000   4B      file ID                FOURCC      "file" (0x66696C65)
0x0004   4B      chunk size             uint32 LE   Size of rest
0x0008   4B      fccType                FOURCC      File type (e.g., "WAVE")
0x000C   N       szPathname              char[]      Null-terminated file path
```

---

## 4. BROADCAST WAVE FORMAT (BWF) EXTENSIONS

### 4.1 bext Chunk
The `bext` (Broadcast Extension) chunk stores professional broadcast metadata. It is defined in EBU R98 and EBU Tech 3285.

```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------------
0x0000   4B      bext ID                FOURCC      "bext" (0x62657874)
0x0004   4B      chunk size             uint32 LE   Size of rest (typically 602+)
0x0008   602B    Description             char[602]   UTF-8 description (0-padded)
0x0266   32B     Originator              char[32]    Name of facility that created file
0x0286   32B     OriginatorReference     char[32]    Unique identifier from originator
0x02A6   10B     OriginationDate        char[10]    YYYY-MM-DD (ISO 8601 date)
0x02B0   8B      OriginationTime         char[8]     HH:MM:SS (24-hour time)
0x02B8   4B      TimeReference           uint32 LE   First sample count since midnight (sample 0)
0x02BC   2B      Version                 uint16 LE   BWF version: 1 or 2
0x02BE   1B      UMIDFlag               uint8       0 = no UMID follows, 1 = UMID follows
[if UMIDFlag == 1]
0x02BF   64B     UMID                   uint8[64]   SMPTE UMID (Unique Material Identifier)
[Version 2 fields]
[4B]      CodingHistoryERP            variable     Coding history — extensible string
```

**UMID (Unique Material Identifier):** A 64-byte SMPTE standard identifier for broadcast material. Basic UMID is 32 bytes; full UMID is 64 bytes with the pack header.

### 4.2 ds64 Chunk (RF64 Extension)
The `ds64` (Data Size 64) chunk is the cornerstone of the RF64 format. It stores 64-bit sizes that overcome the 4GB RIFF limitation.

```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------------
0x0000   4B      ds64 ID                FOURCC      "ds64" (0x64733634)
0x0004   4B      chunk size             uint32 LE   Size of rest (typically 28 + 4×numTableEntries)
0x0008   8B      riffSize64             uint64 LE   True 64-bit file size (excl. first 12 bytes)
0x0010   8B      dataSize64             uint64 LE   True 64-bit data chunk size
0x0018   8B      sampleCount64          uint64 LE   True 64-bit total sample count
0x0020   4B      tableLength            uint32 LE   Number of additional chunk size entries
[table array]    additional sizes       uint64[]    64-bit sizes for other chunks
```

**Rules for ds64:**
- Must be the first chunk immediately after the RF64 header
- If a 32-bit field (RIFF size, data size, fact sample count) equals -1 (0xFFFFFFFF), the corresponding 64-bit value in ds64 is used
- The `tableLength` field allows any number of additional chunks to have their sizes reported in 64-bit

### 4.3 iXML Chunk
The `iXML` chunk embeds XML-encoded metadata. It is defined in EBU Tech 3285s1.

```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------------
0x0000   4B      iXML ID                FOURCC      "iXML" (0x69584D4C)
0x0004   4B      chunk size             uint32 LE   Size of XML data
0x0008   N       XML data               char[]      UTF-8 encoded XML document
```

**Common iXML contents:** SCMS (Serial Copy Management System) info, additional broadcast metadata, production notes.

### 4.4 BWF Timestamp Encoding
BWF timestamps follow ISO 8601:
- **Date:** `YYYY-MM-DD` (10 bytes, zero-padded, e.g., `2024-01-15`)
- **Time:** `HH:MM:SS` (8 bytes, 24-hour, zero-padded, e.g., `14:30:00`)
- **Time Reference:** Samples since midnight (dwTimeReference, 4 bytes at 48kHz)
  - Formula: `seconds_since_midnight = dwTimeReference / 48000`
  - For 44.1kHz files: `dwTimeReference × 1000 / 44100`

### 4.5 BWF Compliance — EBU R98 and EBU R68
| Compliance Level | Requirements |
|-----------------|--------------|
| **EBU R98 (BWF)** | fmt chunk present, bext chunk present with description and originator |
| **EBU R68 (Level A)** | Full R98 + iXML with SCMS info, UMID, full originator reference |
| **EBU R68 (Level B)** | R98 with partial iXML, UMID not required |
| **EBU Tech 3306 (RF64)** | RIFF→RF64, ds64 mandatory, backward compatible with BWF |

---

## 5. RIFF INFO TAGS

RIFF INFO is a metadata system stored within LIST→INFO subchunks. Unlike ID3v1/v2 or APE tags, RIFF INFO is embedded directly in the RIFF container.

### 5.1 RIFF INFO Chunk Structure
```
LIST chunk ID:    "list" (0x4C495354)
LIST form type:   "INFO" (0x494E464F)
```

### 5.2 Standard RIFF INFO Keys
| Chunk ID | Meaning | Max Length | Type |
|----------|---------|-----------|------|
| IARL | Archival Location | — | text |
| IART | Artist/Performer | — | text |
| ICMS | Commissioned | — | text |
| ICMT | Comment | — | text |
| ICOP | Copyright | — | text |
| ICRD | Creation Date | 10 bytes | text (YYYY-MM-DD) |
| IENG | Engineer | — | text |
| IGNR | Genre | — | text |
| IKEY | Keywords | — | text |
| IMED | Medium | — | text |
| INAM | Name/Title | — | text |
| IPRD | Product/Album | — | text |
| ISBJ | Subject | — | text |
| ISFT | Software/Encoder | — | text |
| ISHP | Sharpness | — | text |
| ISRC | Source | — | text |
| ISRF | Source Form | — | text |
| ITCH | Technician | — | text |
| IDIT | Digitization Date | — | text (same as ICRD) |
| IMUS | Composer | — | text (unofficial) |
| IPRO | Producer | — | text (unofficial) |
| IWRI | Writer | — | text (unofficial) |

### 5.3 RIFF INFO Chunk Layout
```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------------
0x0000   4B      chunk ID                FOURCC      e.g., "INAM", "IART"
0x0004   4B      chunk size             uint32 LE   Size of text (excluding header)
0x0008   N       text                   char[]      Null-terminated UTF-8 string (padded)
```

---

## 6. RF64 EXTENSION — DETAILED

### 6.1 Why RF64 Exists
Standard RIFF/WAVE uses 32-bit unsigned integers for file sizes and chunk sizes. The maximum representable value is 4,294,967,295 bytes (4GB - 1 byte). For multichannel high-resolution audio, files routinely exceed this limit:
- 7.1 surround at 24-bit/192kHz: 7.1 × 3 bytes/sample × 192,000 samples/sec = 4.1 MB/sec
- One hour of 7.1/192kHz/24-bit: ~14.8 GB

### 6.2 RF64 Header Layout
```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------------
0x0000   4B      RF64 ID                FOURCC      "RF64" (0x52463634) — replaces "RIFF"
0x0004   4B      File Size              uint32 LE   MUST be -1 (0xFFFFFFFF)
0x0008   4B      WAVE Form Type        FOURCC      "WAVE" (0x57415645)
```

### 6.3 RF64 File Layout
```
0x0000: 52 46 36 34           ; "RF64"
0x0004: FF FF FF FF           ; file size = -1 (use ds64)
0x0008: 57 41 56 45           ; "WAVE"
        ──── ds64 ────        ; FIRST chunk after header (mandatory)
        ──── fmt ────         ; Format chunk
        ──── fact ────        ; Sample count (32-bit = -1 in RF64)
        ──── data ────        ; Audio data (32-bit size = -1)
        ──── [other chunks] ────
```

### 6.4 JUNK Chunk (RF64_AUTO Mode)
In `RF64_AUTO` mode (FFmpeg default), FFmpeg writes a JUNK chunk as a placeholder that gets overwritten with the actual ds64 chunk when the file size exceeds 4GB during encoding.

```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------------
0x0000   4B      JUNK ID                FOURCC      "JUNK" (0x4A554E4B)
0x0004   4B      chunk size             uint32 LE   Size of junk data (typically 28)
0x0008   28B     Reserved               uint8[28]   Zero-padded (reserved for ds64)
```

### 6.5 Multiple fact Chunks
WAV/RF64 files may contain multiple `fact` chunks. Each represents sample count for a different audio encoding stage or track. When reading, use the fact chunk closest to the corresponding data reference.

---

## 7. FFmpeg WAV MUXER/DEMUXER REFERENCE

### 7.1 FFmpeg Identifiers
```
Codec Name (CLI):   pcm_s16le, pcm_s24le, pcm_s32le, pcm_f32le, pcm_f64le
AV_CODEC_ID:        AV_CODEC_ID_PCM_S16LE, AV_CODEC_ID_PCM_S24LE, etc.
Format Name (CLI):  wav, riff (both accepted)
Encoder(s):         all PCM codecs, all lossy codecs when wrapped in WAV
Decoder(s):         all WAV-supported codecs
Muxer(s):           wav
Demuxer(s):         wav, bwf (BWF detection)
```

### 7.2 FFmpeg WAV Encoding — Full CLI Reference

```bash
# Basic PCM WAV encoding (CD quality)
ffmpeg -i input.wav -c:a pcm_s16le output.wav

# 24-bit high-resolution WAV
ffmpeg -i input.wav -c:a pcm_s24le -ar 96000 output.wav

# IEEE float WAV (32-bit)
ffmpeg -i input.wav -c:a pcm_f32le output.wav

# RF64 for files >4GB
ffmpeg -i input.wav -rf64 auto -c:a pcm_s16le output.wav

# BWF with broadcast extension (bext)
ffmpeg -i input.wav \
  -rf64 auto \
  -write_bext 1 \
  -metadata title="Broadcast Title" \
  -metadata artist="Artist Name" \
  -metadata description="Program description" \
  -c:a pcm_s16le output.wav

# WAV with peak envelope metadata
ffmpeg -i input.wav \
  -c:a pcm_s16le \
  -write_peak on \
  -peak_format uint16 \
  -peak_block_size 256 \
  -peak_ppv 2 \
  output.wav
```

### 7.3 Complete FFmpeg WAV Muxer Options
| Option | Type | Default | Range | Effect |
|--------|------|---------|-------|--------|
| `rf64` | int | never | auto/always/never | RF64 header mode |
| `write_bext` | bool | 0 | 0/1 | Write bext broadcast chunk |
| `write_peak` | int | off | off/on/only | Write Peak Envelope chunk |
| `peak_block_size` | int | 256 | 1–65536 | Samples per peak frame |
| `peak_format` | int | uint16 | uint8/uint16 | Peak envelope data format |
| `peak_ppv` | int | 2 | 1–2 | Peak points per value |

**RF64 mode values:**
- `never`: Never use RF64, even if file exceeds 4GB (may produce corrupt file)
- `auto`: Use RF64 only if file grows beyond 4GB during writing (default)
- `always`: Always write RF64 header regardless of file size

### 7.4 FFmpeg WAV Demuxer Options
```bash
# Basic demuxing
ffmpeg -i input.wav -f null -

# Probe format details
ffprobe -v quiet -print_format json -show_streams -show_format input.wav

# Extract format info
ffprobe -v error -show_entries format=duration,bit_rate,size -of default=noprint_wrappers=1 input.wav

# Read RIFF INFO tags
ffprobe -v quiet -show_entries format_tags -of default=noprint_wrappers=1 input.wav

# Read BWF bext chunk
ffprobe -v quiet -show_entries format_tags=description,originator,origination_date -of default input.wav
```

### 7.5 FFmpeg WAV Metadata Handling
```bash
# Write RIFF INFO tags via -metadata
ffmpeg -i input.wav -c copy \
  -metadata title="Song Title" \
  -metadata artist="Artist" \
  -metadata album="Album" \
  -metadata comment="Comment text" \
  -metadata INAM="File Title" \
  -metadata IART="Artist" \
  -metadata ICMT="Comment" \
  -metadata ISFT="FFmpeg" \
  output.wav

# Write BWF metadata
ffmpeg -i input.wav -c copy \
  -metadata description="Broadcast description" \
  -metadata originator="Radio Station" \
  -metadata originator_reference="REF001" \
  -metadata date="2024-01-15" \
  -metadata time="10:30:00" \
  -write_bext 1 \
  output.wav

# Strip all metadata
ffmpeg -i input.wav -c copy -map_metadata -1 output.wav

# Gapless WAV concatenation (using concat demuxer)
ffmpeg -f concat -rf64 auto -i filelist.txt -c copy output.wav
```

### 7.6 FFmpeg Internal Metadata Key Mapping (WAV)
| Standard Field | FFmpeg Key | WAV/Riff INFO Key | Notes |
|----------------|------------|-------------------|-------|
| Title | title | INAM | |
| Artist | artist | IART | |
| Album | album | IPRD | |
| Comment | comment | ICMT | |
| Copyright | copyright | ICOP | |
| Software | encoder | ISFT | |
| Date | date | ICRD | Format: YYYY-MM-DD |
| Genre | genre | IGNR | |

---

## 8. GAPLESS PLAYBACK & WAV

### 8.1 Cue Points for Silence Detection
CD-quality WAV files often have cue points for silence intervals:
- INDEX 00: Pre-gap start (silence before track)
- INDEX 01: Track content start

### 8.2 Pad Blocks
Some encoders write artificial pad blocks (silent samples) at the end of a WAV file. The cue→plst system allows players to skip these:
```bash
# Pad block example: 529200 samples = 12ms @ 44.1kHz
# Player uses cue points to identify and skip pad
```

### 8.3 RF64 Seamless Gapless
RF64 and BWF files use the `fact` chunk's sample count and `cue` points to define the playable region. A gapless player reads `fact.dwSampleLength` to determine the total sample count and uses `cue` points for seeking.

---

## 9. MULTI-CHANNEL & SURROUND AUDIO

### 9.1 Supported Channel Layouts
| Channels | Layout Name | Channel Mask | FFmpeg Layout Constant |
|----------|-------------|-------------|------------------------|
| 1 | Mono | 0x0004 | mono |
| 2 | Stereo | 0x0003 | stereo |
| 3 | 2.1 | 0x000B | 2.1 |
| 4 | 4.0 (Quad) | 0x0033 | quad |
| 6 | 5.1 | 0x003F | 5.1 |
| 8 | 7.1 | 0x00FF | 7.1 |
| 8 | 7.1 (wide) | 0x063F | 7.1(wide) |

### 9.2 WAV Channel Interleaving
In multi-channel WAV files, samples are interleaved:
- Stereo (2ch, 16-bit): L R L R L R ...
- 5.1 (6ch, 16-bit): FL FR FC LFE RL RR FL FR ...
- Frame = nChannels × (bitsPerSample/8) bytes

---

## 10. CONVERSION GUIDE

### 10.1 Converting FROM WAV
| Target | FFmpeg Command | Notes |
|--------|---------------|-------|
| FLAC | `ffmpeg -i in.wav -c:a flac -compression_level 8 out.flac` | Lossless |
| MP3 | `ffmpeg -i in.wav -c:a libmp3lame -q:a 0 out.mp3` | ID3v2.3 tags |
| AAC | `ffmpeg -i in.wav -c:a aac -b:a 256k out.m4a` | |
| Opus | `ffmpeg -i in.wav -c:a libopus -b:a 128k out.opus` | Vorbis Comments |
| ALAC | `ffmpeg -i in.wav -c:a alac out.m4a` | |
| OGG | `ffmpeg -i in.wav -c:a libvorbis -q:a 6 out.ogg` | Vorbis Comments |

### 10.2 Converting TO WAV
| Source | FFmpeg Command | Notes |
|--------|---------------|-------|
| FLAC → | `ffmpeg -i in.flac -c:a pcm_s24le out.wav` | Preserve resolution |
| MP3 → | `ffmpeg -i in.mp3 -c:a pcm_s16le out.wav` | |
| ALAC → | `ffmpeg -i in.m4a -c:a pcm_s16le out.wav` | |
| AAC → | `ffmpeg -i in.m4a -c:a pcm_s16le out.wav` | |
| Opus → | `ffmpeg -i in.opus -c:a pcm_s16le out.wav` | |

### 10.3 Lossless Round-Trip Verification
```bash
# Encode to WAV and back
ffmpeg -i original.flac -c:a pcm_s24le original_raw.wav
ffmpeg -i original_raw.wav -c:a flac original_copy.flac

# Compare checksums
ffprobe -v quiet -show_entries format=format_name,duration,size -of default original.flac original_copy.flac
md5sum original.flac original_copy.flac

# Or use framemd5 for bit-exact comparison
ffmpeg -i original.flac -map 0:a -f framemd5 original.md5
ffmpeg -i original_copy.flac -map 0:a -f framemd5 copy.md5
diff original.md5 copy.md5
```

---

## 11. CONTAINER / WRAPPER INTEGRATION

### 11.1 RIFF WAV as a Container
| Property | Value |
|----------|-------|
| Native container | RIFF (no external container needed) |
| Overhead | ~0.1% (just fmt + fact headers) |
| Seeking | Yes — by sample position via cue chunk |
| Multiple streams | No (single audio stream only) |
| Multi-track | Via playlist (plst) and cue chunks |
| Encryption | No (plaintext container) |

### 11.2 Codec-to-Container Compatibility
| Container | Can Store | Seeking | Metadata | Notes |
|-----------|-----------|---------|----------|-------|
| WAV (RIFF) | PCM, IEEE Float, MP3, WMA, AAC, ADPCM, AC3, ATRAC | Yes (PCM) | RIFF INFO + BWF | Standard |
| BWF | PCM, MPEG, Dolby | Yes (PCM) | bext + iXML + RIFF INFO | Broadcast |
| RF64 | Any WAV codec | Yes (PCM) | Same as BWF | >4GB files |
| AVI (RIFF) | Any WAV codec | Via idx1 | AVI metadata chunks | Video+Audio |
| W64 (Sony) | Any WAV codec | Yes | BWF-like | Sony's alternative to RF64 |

---

## 12. KNOWN ISSUES, BUGS & EDGE CASES

### 12.1 FFmpeg WAV-Specific Issues
| Issue | Affected Versions | Workaround |
|-------|-------------------|------------|
| RF64_AUTO: fmt chunk displaced by JUNK | Pre-patch FFmpeg | Use `-rf64 always` |
| RF64 file not recognized by `file` utility | Some versions | Use `mediainfo` instead |
| bext chunk with >602 bytes | FFmpeg limited | Write separate iXML for extended |
| fact chunk missing for some encoders | Older FFmpeg | Always specify output codec |

### 12.2 Interoperability Issues
- **Windows Media Player:** May not read RF64 unless Windows Media Foundation is updated
- **QuickTime:** Limited WAV support; prefer CAF for macOS
- **RF64 files >4GB:** Some legacy DAWs (Pro Tools <12) cannot read
- **Multiple fact chunks:** Behavior varies across implementations

### 12.3 Edge Cases
| Edge Case | Behavior |
|-----------|----------|
| Empty data chunk (0 bytes) | Valid file; produces silence |
| fmt chunk missing | Most players refuse to play |
| fact chunk wrong sample count | Players use fact; some ignore it |
| wBitsPerSample = 0 | Compressed format; block-aligned |
| Odd chunk sizes | Padding byte (0x00) appended |
| Non-PCM with fact chunk = -1 | Use ds64 sampleCount64 |
| Malformed channel mask | Player may fail or guess layout |

---

## 13. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Encoder | Decoder | URL |
|---------|----------|---------|---------|---------|-----|
| FFmpeg (libavformat) | C | LGPL 2.1+ | 10/10 | 10/10 | ffmpeg.org |
| libsndfile | C | LGPL 2.1 | 9/10 | 9/10 | github.com/erikd/libsndfile |
| SoX | C | GPL/BSD | 7/10 | 8/10 | sourceforge.net/projects/sox |
| WavPack | C | BSD | 8/10 | 8/10 | wavpack.com |
| BWF MetaEdit | C++ | Public Domain | — | — | sourceforge.net/projects/bwfmedit |
| SoundExchange (SxWAV) | C | Public Domain | — | — | github.com/SRI-CSL/soundcoop |

---

## 14. SPECIFICATIONS & FURTHER READING

### Primary Specifications
- **Microsoft RIFF/WAVE:** Multimedia Programming Interface and Data Specifications 1.0 — https://www.mmsp.ece.mcgill.ca/Documents/AudioFormats/WAVE/Docs/riffmci.pdf
- **EBU R98:** EBU Standard for Broadcast Wave Format — https://tech.ebu.ch/docs/r/r098.pdf
- **EBU Tech 3285:** BWF Supplement — https://tech.ebu.ch/docs/tech/tech3285.pdf
- **EBU Tech 3306:** RF64 Specification — https://tech.ebu.ch/docs/tech/tech3306v1_1.pdf
- **ITU-R BS.2088:** BW64 — https://www.itu.int/rec/R-REC-BS.2088/

### Technical Resources
- FFmpeg WAV muxer source: `libavformat/wavenc.c`
- FFmpeg WAV demuxer source: `libavformat/wavdec.c`
- Microsoft WAVEFORMATEX: `mmreg.h` / `mmeapi.h`
- Microsoft WAVEFORMATEXTENSIBLE: `ksmedia.h`
- Hydrogenaudio WAV article: https://hydrogenaudio.org

---

## 15. IMPLEMENTATION CHECKLIST

### Build & Environment
- [ ] FFmpeg compiled with WAV muxer and demuxer (default: enabled)
- [ ] Note external codec dependencies (libmp3lame for MP3 in WAV, etc.)

### Encoding Pipeline
- [ ] Convert input sample format to required output format using libswresample
- [ ] Set WAVEFORMATEX fields: wFormatTag, nChannels, nSamplesPerSec, nAvgBytesPerSec, nBlockAlign, wBitsPerSample
- [ ] Set WAVEFORMATEXTENSIBLE when needed: cbSize=22, SubFormat GUID, dwChannelMask, wValidBitsPerSample
- [ ] Include fact chunk for all non-PCM formats
- [ ] Use `-rf64 auto` to prevent corruption on files >4GB
- [ ] Use `-write_bext 1` for BWF compliance

### Decoding Pipeline
- [ ] Parse RIFF/RF64 header to identify file type
- [ ] Read fmt chunk into WAVEFORMATEX or WAVEFORMATEXTENSIBLE structure
- [ ] Handle WAVEFORMATEXTENSIBLE: extract SubFormat GUID
- [ ] Read fact chunk for non-PCM formats (total sample count)
- [ ] Read cue chunk for seeking/track boundary information
- [ ] Handle ds64 chunk for RF64 files (64-bit sizes)
- [ ] Read bext and iXML for BWF metadata

### Metadata
- [ ] Read all standard RIFF INFO keys (INAM, IART, ICMT, ICRD, etc.)
- [ ] Read BWF bext chunk fields
- [ ] Write RIFF INFO via metadata key mapping
- [ ] Preserve bext metadata when remuxing
- [ ] Handle multiple fact chunks (use closest to data)

### Quality & Verification
- [ ] Implement BWF/RF64 validation against EBU R98/R68
- [ ] Verify byte-level PCM data integrity
- [ ] Test with: silence, full-scale, multi-channel, high-resolution, >4GB files
- [ ] Use `file` utility and mediainfo for format validation

### Edge Cases
- [ ] Handle corrupt or missing fmt chunk
- [ ] Handle fact chunk with incorrect sample count
- [ ] Handle RF64 files on non-RF64-aware players
- [ ] Handle chunk padding bytes correctly
- [ ] Handle files with multiple fact chunks

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
