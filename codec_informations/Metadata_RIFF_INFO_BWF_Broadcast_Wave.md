# RIFF INFO Tags and BWF (Broadcast Wave Format) — Deep Technical Reference
> **Category:** Metadata
> **File Extensions:** `.wav`
> **MIME Types:** `audio/x-wav`, `audio/wav`
> **Standardization Body:** Microsoft (RIFF), EBU (BWF)
> **Primary Specification:** EBU Tech 3285 (BWF v2), EBU Tech 3306 (RF64), Microsoft RIFF Specification
> **Patent Status:** Open — RIFF is documented, BWF is EBU standard
> **License:** Open
> **Current Version:** BWF Version 2 (EBU Tech 3285s1)
> **Active Development:** Yes — EBU continues to maintain the specification

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Microsoft and IBM (RIFF), European Broadcasting Union (BWF)
- **Year Created:** RIFF: 1991 (introduced with Windows 3.1 Multimedia Extensions); BWF: 1995 (EBU Tech 3285)
- **Original Purpose:** RIFF provides a general-purpose container for multimedia data (audio, video, text). BWF was created specifically for broadcast environments, adding professional metadata for program exchange between broadcasters.
- **Problem with Predecessors:** Early WAV files had no standardized metadata for professional use. Broadcasters needed reliable originator identification, timestamps, and loudness measurements — none of which existed in plain RIFF/WAVE.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| EBU Tech 3285 v1 | 1995 | Initial BWF specification with bext chunk |
| EBU Tech 3285 v2 | 2006 | Added UMID, loudness fields, iXML chunk |
| EBU Tech 3306 (RF64) | 2009 | 64-bit chunk sizes, ds64 chunk for files > 4GB |
| EBU R98 | 2008 | Practice guideline for BWF usage in production |
| EBU R68 | 2000 | Level naming convention for broadcast audio |

### 1.3 Current Adoption
- **Primary use cases today:** Professional audio production, broadcast workflows, archival storage, DAW sessions
- **Platforms with native support:** Windows (native), macOS (native via Core Audio), Linux (via FFmpeg/libav), embedded systems
- **Major services using this format:** Broadcasters (BBC, ARD, France Télévisions), film post-production studios, archives
- **Hardware support:** Nearly all professional audio equipment supports WAV/BWF natively
- **Status:** Dominant in professional audio, legacy in consumer music distribution

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 RIFF Container Architecture

RIFF (Resource Interchange File Format) is a tagged chunk architecture. Every element in a RIFF file is a **chunk** with the following structure:

```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       52 49 46 46     "RIFF"   Chunk ID
0x0004  4       XX XX XX XX     uint32    Chunk size (little-endian)
0x0008  4       XX XX XX XX     uint32    Form type (e.g., "WAVE")
0x000C  n       ...             ...       Chunk data
```

A WAVE file is a RIFF form with type `WAVE`. It contains sub-chunks for format, audio data, and metadata.

### 2.2 WAVE File Structure (Byte-Level)

```
Offset   Size    Field Name              Type        Valid Range   Description
-------  ------  ----------------------  ----------  -----------   ---------------------------
0x0000   4B      Chunk ID                char[4]     "RIFF"        Must be "RIFF" (0x52494646)
0x0004   4B      Chunk Size              uint32 LE   varies        Size of rest of file minus 8
0x0008   4B      Form Type               char[4]     "WAVE"        Must be "WAVE" (0x57415645)
0x000C   4B      Subchunk1 ID            char[4]     "fmt "        Format chunk (space after)
0x0010   4B      Subchunk1 Size          uint32 LE   16/18/40      16 for PCM, 18 for extensible
0x0014   2B      Audio Format            uint16 LE   1, 3, 0xFFFE  1=PCM, 3=IEEE float, 0xFFFE=extensible
0x0016   2B      Num Channels            uint16 LE   1–65536       Number of audio channels
0x0018   4B      Sample Rate             uint32 LE   various       Samples per second
0x001C   4B      Byte Rate               uint32 LE   varies        Sample Rate × Num Channels × Bits/8
0x0020   2B      Block Align             uint16 LE   varies        Num Channels × Bits/8
0x0022   2B      Bits Per Sample         uint16 LE   8, 16, 24, 32 Bits per sample per channel
[0x0024] 2B      Extra Param Size        uint16 LE   0 or 22       For extensible format only
[0x0026] 2B      Valid Bits Per Sample   uint16 LE   8–32          Actual resolution (extensible only)
[0x0028] 4B      Channel Mask             uint32 LE   bit mask      Speaker positions (extensible only)
[0x002C] 16B     SubFormat               GUID        16 bytes      Format GUID (extensible only)
0x0024   4B      data ID                 char[4]     "data"        Audio data chunk
0x0028   4B      data Size               uint32 LE   varies        Size of audio data in bytes
0x002C   n       Audio Data              binary      ...           PCM or compressed samples
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Max chunk size (32-bit) | 4 GB | Limited by uint32 in standard RIFF |
| Max chunk size (RF64) | 16 EB | Limited by uint64 in ds64 chunk |
| Max file size (standard) | ~4 GB | RIFF/WAVE limit |
| Max file size (RF64) | 16 EB | With ds64 chunk |
| Character encoding | ANSI/UTF-8 | Historically ANSI, modern tools prefer UTF-8 |
| Byte order | Little-endian | RIFF/WAVE specific |

---

## 3. RIFF INFO CHUNK — DETAILED SPECIFICATION

### 3.1 LIST INFO Container

The `LIST INFO` chunk is a standard RIFF mechanism for embedding text metadata. It follows this structure:

```
Offset  Bytes   Field Name          Type        Description
------  ------  ------------------  ----------  ----------------------------------
0x00    4       Chunk ID            char[4]     "LIST" (0x4C495354)
0x04    4       Chunk Size          uint32 LE   Size of list data minus 8
0x08    4       List Type ID        char[4]     "INFO" (0x494E464F)
0x0C    n       Subchunk Data       varies      INFO subchunks
```

### 3.2 INFO Subchunk Structure

Each INFO subchunk has the following layout:

```
Offset  Bytes   Field Name          Type        Description
------  ------  ------------------  ----------  ----------------------------------
0x00    4       Subchunk ID         char[4]     4-character key (e.g., "INAM")
0x04    4       Subchunk Size       uint32 LE   Size of subchunk data in bytes
0x08    n       Subchunk Data       varies      Text data (null-terminated or raw)
```

**Critical Rule:** All INFO subchunk data must be written as **null-terminated ASCII or UTF-8 strings**. If the text length is odd, a single null padding byte is added to maintain even alignment at the chunk level.

### 3.3 Standard INFO Subchunk Reference

| Subchunk ID | ExifTool Name | Description | Max Length | Example |
|-------------|---------------|-------------|------------|---------|
| `INAM` | Title | Track/song title | 32KB | "Symphony No. 5 in C minor" |
| `IART` | Artist | Performer, creator | 32KB | "Beethoven" |
| `ICMT` | Comment | General comments | 32KB | "Recorded at Vienna Konzerthaus" |
| `ICRD` | DateCreated | Creation date | 32KB | "2024-01-15" |
| `IGNR` | Genre | Genre category | 32KB | "Classical" |
| `ISFT` | Software | Encoder software | 32KB | "Adobe Audition 24.0" |
| `ITCH` | Technician | Engineer/technician | 32KB | "John Smith" |
| `IPRD` | Product | Album/product name | 32KB | "Complete Symphonies" |
| `ICOP` | Copyright | Copyright notice | 32KB | "© 2024 Classic Records" |
| `ISBJ` | Subject | Subject description | 32KB | "Symphony performance" |
| `IMED` | Medium | Original medium | 32KB | "CD" |
| `IHLN` | Length | Track length | 32KB | "00:05:32" |
| `IKEY` | Keywords | Keywords/tags | 32KB | "orchestra, classical, symphony" |
| `IAUT` | Author | Author/composer | 32KB | "Ludwig van Beethoven" |
| `IASU` | Artist | Artist alias | 32KB | (same as IART) |
| `IENG` | Engineer | Engineer alias | 32KB | (same as ITCH) |
| `ISRC` | Source | Source media | 32KB | "Original master tape" |
| `ISRF` | SourceForm | Source format | 32KB | "Analog tape" |
| `ICNM` | Cinematographer | Camera/recording person | 32KB | (video-related) |
| `IPDS` | ProductionDesigner | Production designer | 32KB | (video-related) |
| `ISTD` | ProductionStudio | Studio name | 32KB | "Abbey Road Studios" |
| `ISTR` | Starring | Cast/starring | 32KB | (video-related) |
| `ICDS` | CostumeDesigner | Costume designer | 32KB | (video-related) |
| `IMUS` | MusicBy | Music credits | 32KB | "Berlin Philharmonic" |
| `ICNT` | Country | Country of origin | 32KB | "Germany" |
| `IENC` | EncodedBy | Encoded by | 32KB | "Audio Engineer" |
| `IRIP` | RippedBy | Ripped by | 32KB | "CD ripper software" |
| `IPRO` | ProducedBy | Produced by | 32KB | "Producer name" |
| `IWMU` | WatermarkURL | Watermark URL | 32KB | "https://..." |
| `IWRI` | WrittenBy | Written by | 32KB | "Composer name" |
| `IARL` | ArchivalLocation | Archive location | 32KB | "Library Archive" |
| `ICRP` | Cropped | Cropping info | 32KB | (video-related) |
| `IDIM` | Dimensions | Dimensions | 32KB | (video-related) |
| `IDPI` | DotsPerInch | DPI | 32KB | (video-related) |
| `ICDS` | CostumeDesigner | Costume designer | 32KB | (video-related) |
| `IDST` | DistributedBy | Distributed by | 32KB | "Distributor" |
| `IEDT` | EditedBy | Edited by | 32KB | "Editor name" |
| `ICMS` | Commissioned | Commissioned by | 32KB | "Commissioning org" |

### 3.4 Complete INFO Chunk Byte Map Example

```
52 49 46 46           ; Chunk ID: "RIFF"
24 00 00 00           ; Chunk size: 36 bytes
57 41 56 45           ; Form type: "WAVE"
4C 49 53 54           ; Subchunk ID: "LIST"
18 00 00 00           ; LIST size: 24 bytes
49 4E 46 4F           ; List type: "INFO"
49 4E 41 4D           ; Subchunk ID: "INAM"
0C 00 00 00           ; Subchunk size: 12 bytes
53 79 6D 70 68 6F 6E 79 20 4E 6F 2E 00  ; Data: "Symphony No.\0"
```

### 3.5 RIFF INFO Chunk Limitations

| Limitation | Value | Notes |
|------------|-------|-------|
| Max subchunk size | 4,294,967,295 bytes | Limited by uint32 |
| Practical max | ~32 KB | Most tools truncate longer values |
| Character encoding | Historically ASCII/ANSI | Modern tools support UTF-8 |
| Null termination | Required | Must be null-terminated string |
| Alignment | Even bytes | Padding byte if odd length |
| Ordering | Any order allowed | Tools may enforce specific order |

---

## 4. BWF (BROADCAST WAVE FORMAT) — DETAILED SPECIFICATION

### 4.1 BWF Overview

BWF is a superset of standard WAV that adds professional broadcast metadata via the `bext` chunk. A BWF file is a valid WAV file with mandatory `bext` chunk.

### 4.2 BWF File Structure

```
RIFF(WAVE
    fmt  -- Format chunk (mandatory)
    bext -- Broadcast Extension chunk (mandatory for BWF compliance)
    fact -- Fact chunk (required for MPEG, optional for PCM)
    data -- Audio data chunk (mandatory)
    LIST INFO -- Optional RIFF INFO metadata
    iXML -- Optional XML metadata chunk
    ds64 -- RF64 data size extension (for files > 4GB)
)
```

### 4.3 bext Chunk — Broadcast Audio Extension

The `bext` chunk is the defining feature of BWF. It contains professional metadata required for broadcast program exchange.

#### bext Byte Map

```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ----------------------------------
0x00     4B      Chunk ID                char[4]     "bext" (0x62657874)
0x04     4B      Chunk Size              uint32 LE   Typically 602 or 602+coding_history
0x08     256B    Description             char[256]   ASCII: description of sound content
0x108    32B     Originator              char[32]    ASCII: name of originator (equipment)
0x128    32B     OriginatorReference     char[32]    ASCII: unique reference from originator
0x148    10B     OriginationDate         char[10]    ASCII: YYYY-MM-DD (broadcast date)
0x152    8B      OriginationTime         char[8]     ASCII: HH:MM:SS (UTC time)
0x15A    8B      TimeReference           uint48 LE   First sample count since midnight UTC
0x162    2B      Version                 uint16 LE   0x0001=Version 1, 0x0002=Version 2
0x164    64B     UMID                    binary      SMPTE 330M Universal Material Identifier
0x1A4    4B      LoudnessValue           int16 LE    round(100 × LUFS integrated loudness)
0x1A8    4B      LoudnessRange           int16 LE    round(100 × LU loudness range)
0x1AC    2B      MaxTruePeakLevel        int16 LE    round(100 × dBTP max true peak)
0x1AE    2B      MaxMomentaryLoudness    int16 LE    round(100 × LUFS momentary max)
0x1B0    2B      MaxShortTermLoudness    int16 LE    round(100 × LUFS short-term max)
0x1B2    180B    Reserved                binary      Set to NULL (zero) for v1 and v2
0x262    n       CodingHistory           char[]      ASCII: coding history string
```

#### bext Field Details

**Description Field (256 bytes):**
- ASCII null-terminated string
- Describes the sound content
- If less than 256 characters, remaining bytes are NULL
- Example: `"Opening movement of Beethoven's Fifth Symphony"`

**Originator Field (32 bytes):**
- ASCII null-terminated string
- Identifies the recording equipment/operator
- Example: `"TASCAM DA-6400"`, `"Pro Tools 2024"`

**OriginatorReference Field (32 bytes):**
- ASCII null-terminated string
- Unique identifier assigned by the originator
- Format defined by the originating organization
- Example: `"PROJ-2024-001-A1B2"`

**OriginationDate Field (10 bytes):**
- ASCII format: `YYYY-MM-DD` (exactly 10 characters including hyphens)
- Represents the broadcast/recording date
- Example: `"2024-03-15"`

**OriginationTime Field (8 bytes):**
- ASCII format: `HH:MM:SS` (exactly 8 characters including colons)
- UTC time of recording
- Example: `"14:30:00"`

**TimeReference Field (8 bytes):**
- 48-bit unsigned integer (little-endian across 8 bytes)
- Sample count at the first sample of the file since midnight UTC
- Used to synchronize with other recordings
- For 48 kHz, 24-hour range: 0 to 4,294,967,295 samples

**Version Field (2 bytes):**
- `0x0001`: Version 1 (original BWF)
- `0x0002`: Version 2 (added loudness fields, UMID required)
- Version 2 added: LoudnessValue, LoudnessRange, MaxTruePeakLevel, MaxMomentaryLoudness, MaxShortTermLoudness

**UMID Field (64 bytes):**
- SMPTE 330M Universal Material Identifier
- First 32 bytes: Basic UMID (mandatory)
- Last 32 bytes: Extension (set to NULL if only basic UMID)
- Format: 16-byte UUID + 16-byte material number
- Used for unique identification of broadcast content

**Loudness Fields (Version 2 only):**
- All stored as signed 16-bit integers
- Value = round(100 × actual_value)
- LoudnessValue: Integrated loudness in LUFS (-70 to 0 range typical)
- LoudnessRange: Loudness range in LU (0 to 20 typical)
- MaxTruePeakLevel: Maximum true peak in dBTP (-70 to 0 typical)
- MaxMomentaryLoudness: Maximum momentary loudness in LUFS (short window, ~400ms)
- MaxShortTermLoudness: Maximum short-term loudness in LUFS (~3s window)

**CodingHistory Field (variable):**
- ASCII string (no null termination required if at chunk end)
- Contains human-readable encoding chain
- Format defined by EBU R98
- Example: `"A=DPPM,B=FFmpeg,L=24,T=PCM,F=48000"

### 4.4 Coding History Format (EBU R98)

The coding history string records the encoding chain. Format follows EBU R98:

```
A=<source_type>,B=<encoder>,C=<bit_depth>,D=<sample_rate>,
E=<codec_type>,F=<sample_rate_reconf>,G=<word_length>,
H=<speed_mode>,I=<features>,...
```

Common fields:
| Field | Description | Example |
|-------|-------------|---------|
| A | Source type | DPPM (digital production), ANALOGUE |
| B | Encoder/software | FFmpeg, Adobe Audition |
| C | Bit depth | 16, 24, 32 |
| D | Sample rate | 48000, 96000 |
| E | Codec | PCM, FLAC |
| F | Sample rate reconfirmation | (empty or reconfirmed rate) |
| L | Word length | 24 |

Example coding history:
```
A=DPPM,B=FFmpeg v6.0,L=24,T=PCM,F=48000,A=DPPM
```

### 4.5 ds64 Chunk — RF64 Extension

RF64 (Radio 64) extends RIFF/WAVE to support files larger than 4GB. The `ds64` chunk provides 64-bit sizes where the standard 32-bit fields would overflow.

```
Offset  Bytes   Field Name              Type        Description
------  ------  ------------------  ----------  ----------------------------------
0x00    4       Chunk ID                char[4]     "ds64" (0x64733634)
0x04    4       Chunk Size              uint32 LE   28 + 4×table_length
0x08    8       RIFF Size               uint64 LE   Total file size minus 8
0x10    8       Data Size               uint64 LE   Size of 'data' chunk data
0x18    8       Sample Count            uint64 LE   Total samples in file
0x20    4       Table Length             uint32 LE   Number of size table entries
0x24    n       Size Table               varies      Additional chunk size entries
```

**Size Table Entry (each 16 bytes):**
```
4B      Chunk FourCC        Chunk ID of oversized chunk
4B      Chunk Size Low      Lower 32 bits of 64-bit size
4B      Chunk Size High     Upper 32 bits of 64-bit size
4B      Reserved            Set to zero
```

### 4.6 iXML Chunk — XML Metadata

The `iXML` chunk (introduced in BWF Version 2) contains additional metadata in XML format. It provides a standardized way to include production metadata that doesn't fit in the fixed `bext` fields.

**Common iXML Elements:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<BEXT>
    <DESCRIPTION>Program description</DESCRIPTION>
    <ORIGINATOR>Equipment name</ORIGINATOR>
</BEXT>
<PRODUCER>
    <PRODUCER_NAME>Producer name</PRODUCER_NAME>
    <PRODUCER_BOSCH_CODE>...</PRODUCER_BOSCH_CODE>
</PRODUCER>
<PROJECT>
    <PROJECT_NAME>Project identifier</PROJECT_NAME>
    <PROJECT_NOTE>Notes</PROJECT_NOTE>
</PROJECT>
<SCENARIO>
    <SCENARIO_NAME>Scenario name</SCENARIO_NAME>
</SCENARIO>
<TAKE>
    <TAKE_NAME>Take number</TAKE_NAME>
    <TAPE>...</TAPE>
    <CIRCLED>false</CIRCLED>
</TAKE>
<UMID>
    <PREVIOUS_UMID>...</PREVIOUS_UMID>
</UMID>
```

### 4.7 BWF Compliance Levels

| Level | Requirements | Use Case |
|-------|-------------|----------|
| Level A | bext chunk mandatory, Version 2, full loudness metadata | Archival/exchange requiring maximum metadata |
| Level B | bext chunk mandatory, Version 1 acceptable | Standard broadcast exchange |
| Level C | bext chunk strongly recommended | General production |

---

## 5. METADATA ARCHITECTURE

### 5.1 Metadata Chunk Priority

When both `bext` and `LIST INFO` are present:

| Priority | Chunk | Notes |
|----------|-------|-------|
| 1 | `bext` | BWF primary metadata — read first |
| 2 | `iXML` | Extended XML metadata |
| 3 | `LIST INFO` | Standard RIFF INFO tags |

**Conflict Resolution:**
- `bext` OriginatorDate vs INFO ICRD: `bext` takes precedence
- `bext` Description vs INFO INAM: Both may coexist; `bext` for technical, `INAM` for artistic title
- When in doubt, `bext` is the authoritative source in BWF files

### 5.2 RIFF INFO Field Mapping

| RIFF INFO | ExifTool Tag | FFmpeg Key | Notes |
|------------|-------------|------------|-------|
| INAM | Title | title | Track title |
| IART | Artist | artist | Performer/artist |
| ICMT | Comment | comment | General comments |
| ICRD | DateCreated | date | Creation date |
| IGNR | Genre | genre | Genre category |
| ISFT | Software | encoder | Encoder software |
| ITCH | Technician | — | Engineer name |
| IPRD | Product | album | Album/product name |
| ICOP | Copyright | copyright | Copyright notice |
| ISBJ | Subject | — | Subject description |
| IMED | Medium | — | Original medium |
| IHLN | Length | — | Track length |
| IKEY | Keywords | — | Keywords/tags |
| ISRC | Source | — | Source media |
| IARL | ArchivalLocation | — | Archive location |

### 5.3 Cover Art in WAV

WAV files do not have a standard cover art mechanism. However, several conventions exist:

| Method | Description | Tool Support |
|--------|-------------|--------------|
| `LIST INFO` DISP | Display title (not cover art) | Limited |
| `fmt ` extension | Private codec extensions | Rare |
| Private chunks | Format-specific extensions | Tool-dependent |

**Recommendation:** For cover art in WAV, use embedded album art in an associated metadata system or convert to a format with native cover art support.

---

## 6. FFMPEG IMPLEMENTATION REFERENCE

### 6.1 FFmpeg Identifiers

```
Format Name (CLI):   wav, riff        # used with -f
Format Demuxer:      wav.c, riff.c    # in libavformat
Format Muxer:         wav.c, riff.c    # in libavformat
Codec Support:        PCM (all formats), IEEE float, various compressed codecs
```

### 6.2 FFmpeg Writing RIFF INFO Metadata

```bash
# Basic metadata write to WAV
ffmpeg -i input.wav \
  -metadata title="Symphony No. 5" \
  -metadata artist="Beethoven" \
  -metadata album="Complete Symphonies" \
  -metadata date="2024" \
  -metadata genre="Classical" \
  -metadata comment="Recorded live" \
  output.wav

# Write with specific INFO chunk IDs (using -metadata with underscores)
ffmpeg -i input.wav \
  -metadata title="Track Title" \
  -metadata author="Composer Name" \
  -metadata copyright="© 2024 Label" \
  -metadata software="FFmpeg" \
  output.wav

# Write BWF metadata (bext chunk)
ffmpeg -i input.wav \
  -write_bext true \
  -metadata description="Concert recording" \
  -metadata originator="FFmpeg" \
  -metadata originator_reference="REF-001" \
  -metadata originator_date="2024-03-15" \
  -metadata originator_time="14:30:00" \
  output.wav

# Strip all metadata
ffmpeg -i input.wav -c copy -map_metadata -1 output.wav
```

### 6.3 FFmpeg Reading RIFF INFO Metadata

```bash
# Read all format metadata
ffprobe -v quiet -print_format json -show_format input.wav | jq .format.tags

# Read specific fields
ffprobe -v quiet -show_format input.wav | grep -E "^tag_(title|artist|album)="

# Read BWF bext metadata
ffprobe -v quiet -show_format input.wav | grep -E "bext|description|originator"

# Read metadata from specific chunk (using exiftool for bext)
exiftool -BroadcastExtension input.wav
exiftool -Description -Originator -OriginationDate input.wav

# Read all RIFF tags including non-standard
exiftool -a input.wav
```

### 6.4 FFmpeg BWF-specific Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `-write_bext` | bool | false | Enable BWF bext chunk writing |
| `-write_bext true` | flag | — | Add bext chunk to output file |
| `-metadata description=` | string | — | Set bext description field |
| `-metadata originator=` | string | — | Set bext originator field |
| `-metadata originator_reference=` | string | — | Set bext reference field |
| `-metadata originator_date=` | string | — | Set bext date (YYYY-MM-DD) |
| `-metadata originator_time=` | string | — | Set bext time (HH:MM:SS) |

### 6.5 FFmpeg Encoding Examples

```bash
# Encode to WAV with metadata
ffmpeg -i input.flac -c:a pcm_s16le -ar 44100 -ac 2 output.wav

# High-resolution WAV (24-bit, 96kHz)
ffmpeg -i input.flac -c:a pcm_s24le -ar 96000 output.wav

# Float WAV (32-bit)
ffmpeg -i input.flac -c:a pcm_f32le output.wav

# WAV with BWF compliance
ffmpeg -i input.wav -write_bext true -metadata description="Program content" output_bwf.wav

# WAV with RF64 extension (for files > 4GB)
ffmpeg -i input.wav -rf64 auto output.wav
```

---

## 7. TOOL REFERENCE

### 7.1 exiftool RIFF/BWF Support

```bash
# Read all RIFF/BWF tags
exiftool input.wav

# Read bext (Broadcast Extension) specifically
exiftool -BroadcastExtension input.wav

# Read RIFF INFO tags
exiftool -Title -Artist -Album -Comment input.wav

# Read all tags including custom
exiftool -a input.wav

# Read ds64 (RF64 data)
exiftool -DataSize64 input.wav

# Write RIFF INFO tags
exiftool -Title="New Title" -Artist="New Artist" input.wav

# Write bext metadata
exiftool -Description="Recording" -Originator="Recorder" input.wav
```

### 7.2 kid3-cli RIFF/BWF Support

```bash
# Read RIFF INFO tags
kid3-cli -c "get" input.wav

# Read specific tag
kid3-cli -c "get :title" input.wav

# Write tag
kid3-cli -c "set title 'Track Title'" input.wav

# Read bext (if present)
kid3-cli -c "get" input.wav  # Shows bext if available
```

### 7.3 RIFF INFO Limitations in Tools

| Issue | Description | Workaround |
|-------|-------------|------------|
| Character encoding | Many tools write ANSI, not UTF-8 | Use UTF-8 aware tools |
| Field length | No enforced max, but practical limit ~32KB | Truncate long values |
| Case sensitivity | Keys are case-sensitive (INAM vs inam) | Use correct case |
| Duplicate keys | Multiple subchunks with same key possible | Read all, use first or last |
| Non-standard keys | Custom keys may not be preserved | Document custom extensions |

---

## 8. BYTE-LEVEL STRUCTURE EXAMPLES

### 8.1 Minimal WAV File Structure

```
Offset   Hex                                         ASCII        Description
-------  ------------------------------------------  ----------   ------------------------
0x0000   52 49 46 46                                 RIFF         Chunk ID
0x0004   24 00 00 00                                 36           Chunk size (little-endian)
0x0008   57 41 56 45                                 WAVE         Form type
0x000C   66 6D 74 20                                 fmt          Format chunk ID
0x0010   10 00 00 00                                 16           Chunk size (16 for PCM)
0x0014   01 00                                       1            Audio format (PCM)
0x0016   02 00                                       2            Channels (stereo)
0x0018   44 AC 00 00                                 44100        Sample rate
0x001C   10 B1 02 00                                 176400       Byte rate (44100×2×2)
0x0020   04 00                                       4            Block align
0x0022   10 00                                       16           Bits per sample
0x0024   64 61 74 61                                 data         Data chunk ID
0x0028   08 00 00 00                                 8            Data size (8 bytes)
0x002C   00 01 02 03 04 05 06 07                     ...          Audio samples
```

### 8.2 WAV with LIST INFO Structure

```
Offset   Hex                                         ASCII        Description
-------  ------------------------------------------  ----------   ------------------------
0x0000   52 49 46 46                                 RIFF         Chunk ID
0x0004   XX XX XX XX                                 ...          Chunk size
0x0008   57 41 56 45                                 WAVE         Form type
0x000C   66 6D 74 20                                 fmt          Format chunk
...       ...                                        ...          (format data)
0x0028   4C 49 53 54                                 LIST         LIST chunk ID
0x002C   XX XX XX XX                                 ...          LIST size
0x0030   49 4E 46 4F                                 INFO         LIST type (INFO)
0x0034   49 4E 41 4D                                 INAM         Title subchunk
0x0038   0C 00 00 00                                 12           Subchunk size
0x003C   54 65 73 74 00                              Test\0       Title text (null-terminated)
0x0044   49 41 52 54                                 IART         Artist subchunk
0x0048   08 00 00 00                                 8            Subchunk size
0x004C   41 72 74 69 73 74 00                        Artist\0     Artist text
0x0054   64 61 74 61                                 data         Data chunk
...       ...                                        ...          (audio data)
```

### 8.3 BWF bext Chunk Structure

```
Offset   Size    Field                       Hex/ASCII        Description
-------  ------  --------------------------  ---------------  ------------------------
0x0000   4B      Chunk ID                   62 65 78 74       "bext"
0x0004   4B      Chunk Size                 6A 02 00 00       618 bytes
0x0008   256B    Description                43 6F 6E 63...     "Concert recording..."
0x0108   32B     Originator                 54 41 53 43...     "TASCAM DA-6400\0..."
0x0128   32B     OriginatorReference        50 52 4F 4A...     "PROJ-001-A1B2\0..."
0x0148   10B     OriginationDate           32 30 32 34...     "2024-03-15"
0x0152   8B      OriginationTime           31 34 3A 33...     "14:30:00"
0x015A   8B      TimeReference             00 00 00 00...     0 (sample count)
0x0162   2B      Version                   02 00               0x0002 (Version 2)
0x0164   64B     UMID                      00 00 00 00...     SMPTE UMID binary
0x01A4   4B      LoudnessValue             90 FF               -112 (round(-1.12×100))
0x01A8   4B      LoudnessRange             08 00               8 LU
0x01AC   2B      MaxTruePeakLevel          CC FF               -52 dBTP
0x01AE   2B      MaxMomentaryLoudness      9A FF               -102 LUFS
0x01B0   2B      MaxShortTermLoudness      88 FF               -120 LUFS
0x01B2   180B    Reserved                  00 00 00 00...      NULL
0x0262   n       CodingHistory             41 3D 44 50...      "A=DPPM,B=FFmpeg..."
```

---

## 9. EBU R68 LOUDNESS STANDARD

### 9.1 EBU R68 Overview

EBU R68 (Recommendation 68) defines how broadcast audio loudness should be measured and reported. It standardizes the -23 LUFS target level for broadcast.

### 9.2 Loudness Measurement Parameters

| Parameter | Symbol | Typical Target | Unit |
|-----------|--------|---------------|------|
| Integrated Loudness | LUFS | -23 | LUFS (absolute) |
| Loudness Range | LRA | < 20 | LU |
| Maximum True Peak | dBTP | -1 | dBTP |
| Momentary Loudness | M | < -20 | LUFS |
| Short-Term Loudness | S | < -23 | LUFS |

### 9.3 BWF Loudness Field Format

The bext chunk stores loudness values as integers scaled by 100:

```
Actual LUFS value = Stored integer / 100
Example: -23.0 LUFS stored as -2300 (0xF8F4)
```

---

## 10. PRACTICAL GUIDELINES

### 10.1 When to Use BWF

| Scenario | Recommendation |
|----------|---------------|
| Broadcast production | Always use BWF with bext chunk |
| Archival | Use BWF Version 2 with loudness |
| Consumer audio | Standard WAV with LIST INFO sufficient |
| DAW sessions | BWF preferred for project files |
| Sound library distribution | BWF for professional, WAV for consumer |

### 10.2 Metadata Preservation Workflow

```
1. Extract source metadata
   exiftool -a input.wav

2. Preserve through conversion
   ffmpeg -i input.wav -i metadata.txt -map_metadata 0 ...

3. Verify output
   exiftool -a output.wav
   kid3-cli -c "get" output.wav
```

### 10.3 Common Issues and Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| Metadata not written | Wrong -metadata syntax | Use key=value, not key:value |
| Chinese/Japanese garbled | ANSI encoding | Set locale to UTF-8, use modern tools |
| bext not created | -write_bext not specified | Add `-write_bext true` to ffmpeg |
| File > 4GB truncated | Standard RIFF limit | Use RF64 format with `-rf64 auto` |
| Duplicate metadata | Multiple tag systems | Strip old tags before writing new |

---

## 11. REFERENCE IMPLEMENTATIONS

### 11.1 Reference Specifications

| Document | Description | URL |
|----------|-------------|-----|
| EBU Tech 3285 | BWF Specification v2 | https://tech.ebu.ch/docs/tech/tech3285.pdf |
| EBU Tech 3306 | RF64/MBWF Specification | https://tech.ebu.ch/docs/tech/tech3306-2009.pdf |
| EBU R98 | Coding History Format | https://tech.ebu.ch/docs/r/r098.pdf |
| EBU R68 | Loudness Normalization | https://tech.ebu.ch/docs/r/r068.pdf |
| Microsoft RIFF | RIFF Specification | https://learn.microsoft.com/en-us/windows/win32/multimedia/multimedia-file-formats |

### 11.2 Implementation Libraries

| Library | Language | BWF Support | Notes |
|---------|----------|-------------|-------|
| libavformat | C | Yes | FFmpeg's WAV/BWF support |
| SoX | C | Yes | Audio processing with BWF |
| BWF MetaEdit | C++ | Full | EBU-developed validation tool |
| SoundFile | C | Yes | Cross-format audio library |

### 11.3 BWF Validation Tools

| Tool | Description | BWF Validation |
|------|-------------|---------------|
| BWF MetaEdit | EBU tool | Full Level A/B/C |
| EBUadm | ADM/BWF validator | Broadcast compliance |
| sox | Audio converter | bext reading/writing |
| exiftool | Metadata tool | Full RIFF INFO + bext |

---

## 12. METADATA COMPATIBILITY

### 12.1 WAV Metadata Support Matrix

| Tag System | Read | Write | Notes |
|------------|------|-------|-------|
| RIFF INFO (LIST INFO) | Yes | Yes | Standard WAV metadata |
| BWF bext | Yes | Partial | FFmpeg partial support |
| RF64 ds64 | Yes | Yes | Automatic with >4GB |
| iXML | Yes | Yes | Requires external tools |
| ID3v2 | Rare | Rare | Non-standard in WAV |

### 12.2 Cross-Tool Metadata Reading

| Tool | RIFF INFO | bext | ds64 | iXML |
|------|-----------|------|------|------|
| FFmpeg/ffprobe | Yes | Yes | Yes | No |
| exiftool | Yes | Yes | Yes | Yes |
| kid3-cli | Yes | Yes | No | No |
| mediainfo | Yes | Yes | Yes | Yes |

---

## 13. IMPLEMENTATION CHECKLIST

### 13.1 Reading RIFF INFO Metadata

- [ ] Parse RIFF container structure
- [ ] Locate LIST INFO chunk (0x4C495354)
- [ ] Verify list type is INFO (0x494E464F)
- [ ] Iterate through INFO subchunks
- [ ] Handle null-terminated strings
- [ ] Support UTF-8 encoding
- [ ] Handle missing/null subchunks gracefully
- [ ] Support non-standard INFO keys

### 13.2 Writing RIFF INFO Metadata

- [ ] Create LIST INFO chunk with proper structure
- [ ] Validate key names are 4-character ASCII
- [ ] Encode text as UTF-8 or ANSI
- [ ] Add null terminator to each string
- [ ] Pad to even byte boundary if odd length
- [ ] Update LIST chunk size field
- [ ] Update RIFF chunk size field

### 13.3 Reading BWF bext Metadata

- [ ] Locate bext chunk (0x62657874)
- [ ] Parse fixed-size fields (Description, Originator, etc.)
- [ ] Handle Version 1 vs Version 2 differences
- [ ] Parse UMID field correctly
- [ ] Decode loudness values (divide by 100)
- [ ] Parse coding history string
- [ ] Handle truncated or malformed fields

### 13.4 Writing BWF bext Metadata

- [ ] Set all mandatory fields for compliance level
- [ ] Validate date format (YYYY-MM-DD)
- [ ] Validate time format (HH:MM:SS)
- [ ] Encode UMID if available
- [ ] Calculate loudness values (multiply by 100)
- [ ] Format coding history per EBU R98
- [ ] Update RIFF chunk size after adding bext

### 13.5 Handling RF64 (ds64)

- [ ] Detect RF64 format (file size > 4GB or 'RF64' identifier)
- [ ] Parse ds64 chunk for 64-bit sizes
- [ ] Use 64-bit size fields where available
- [ ] Handle size table for multiple oversized chunks
- [ ] Ensure compatibility with non-RF64 readers

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
