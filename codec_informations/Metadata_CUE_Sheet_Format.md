# CUE Sheet Format — Deep Technical Reference
> **Category:** Metadata
> **File Extensions:** `.cue`
> **MIME Types:** `application/x-cue`, `text/x-cue`
> **Standardization Body:** None (de facto standard, originally from CDRDAO/EAC)
> **Primary Specification:** CDRDAO / Exact Audio Copy CUE sheet format
> **Patent Status:** Patent-free
> **License:** Open
> **Current Version:** Informal (no formal version, evolved from CDRDAO)
> **Active Development:** Stable — maintained by CD-DA and digital audio community

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Originally CDRDAO (CD Record Disk At Once) and Exact Audio Copy (EAC)
- **Year Created:** ~1999–2001 (EAC popularized the format)
- **Original Purpose:** Describe the layout of audio CDs for extraction, burning, and archival — including pregap regions, track boundaries, ISRC codes, and CD-text
- **Problem with Predecessors:** CD-ROMs lacked a way to describe CD structure beyond raw sector data. CUE filled the gap by providing a human-readable, text-based format for describing track layouts, pregaps, and metadata

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| CDRDAO TOC | ~1998 | Original binary/ASCII format for CDRDAO |
| EAC CUE | ~1999 | Adapted by EAC, added pregap/INDEX handling |
| Standard CUE | ~2000+ | Extended with FLAGS, ISRC, CDTEXTFILE, global commands |
| Modern CUE | Present | Added PERFORMER/SONGWRITER in global section |

### 1.3 Current Adoption
- **Primary use cases today:** CD ripping (EAC, XLD, CUETools), CD burning (ImgBurn, Brasero), CD-image archival (FLAC+CUE), music server metadata
- **Platforms with native support:** All major CD rippers and burners, music servers
- **Major services using this format:** MusicBrainz (for CD lookups), archival audio collections
- **Hardware support:** Limited to software — no hardware reads CUE sheets natively
- **Status:** Declining (CD sales declining) but critical for archival of existing CD collections

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 CUE Sheet Structure
A CUE sheet has two sections:

```
┌─────────────────────────────────────────┐
│  GLOBAL SECTION (optional)               │
│  ├── CATALOG                            │
│  ├── CDTEXTFILE                         │
│  ├── PERFORMER (global)                 │
│  ├── SONGWRITER (global)                │
│  └── TITLE (global)                     │
├─────────────────────────────────────────┤
│  TRACK SECTION (1–99 tracks)            │
│  ├── FILE reference                     │
│  ├── Track 1: TRACK AUDIO DATA         │
│  │   ├── FLAGS                          │
│  │   ├── ISRC                           │
│  │   ├── PERFORMER                      │
│  │   ├── SONGWRITER                     │
│  │   ├── TITLE                          │
│  │   ├── PREGAP                         │
│  │   ├── POSTGAP                        │
│  │   └── INDEX 00, INDEX 01, ...       │
│  ├── Track 2: TRACK AUDIO DATA         │
│  └── ...                                │
└─────────────────────────────────────────┘
```

### 2.2 Key Concepts
- **FILE:** References an audio file (WAVE, AIFF, BINARY, MOTOROLA, MP3). All subsequent tracks belong to this file until a new FILE is declared.
- **TRACK:** Defines a track number and mode (AUDIO, MODE1/2048, etc.)
- **INDEX:** Defines a time offset within the referenced FILE. INDEX 01 is the track start.
- **PREGAP:** Silence at the start of a track (data not stored in FILE)
- **INDEX 00:** Pregap data stored in the FILE itself (pregap with data)
- **Global commands:** Apply to the entire disc if placed before any FILE declaration

---

## 3. CUE SHEET SPECIFICATION

### 3.1 Character Encoding & Syntax
```
Character encoding:  ASCII (ISO-8859-1 recommended for CDTEXTFILE)
Line terminator:    LF (\n) or CRLF (\r\n)
Case sensitivity:   Keywords are case-insensitive, but conventionally UPPERCASE
Quoted strings:     Double quotes "..." for filenames and text containing spaces
                    Inside quotes: escape double quote with \"
Line continuation:  Backslash \ at line end (for readability, not required)
Comments:           Line starting with // or # (tool-dependent)
```

### 3.2 Global Commands
Global commands define disc-level metadata and must appear before the first FILE declaration.

#### CATALOG
```
CATALOG <ean/upc_code>
```
- **Purpose:** 13-digit EAN-13 or 14-digit UPC-A code (with leading zero)
- **Format:** Numeric string, may include hyphens or spaces
- **Example:** `CATALOG 0123456789012`
- **Notes:** Required for commercially released CDs; identifies the disc globally

#### CDTEXTFILE
```
CDTEXTFILE "filename.cdt"
```
- **Purpose:** Path to an external CD-TEXT file (binary format, not plain text)
- **Example:** `CDTEXTFILE "data.cdt"`
- **Notes:** CD-TEXT is binary data embedded in the CD's lead-in area; CDTEXTFILE in CUE references external storage

#### PERFORMER (Global)
```
PERFORMER "performer_name"
```
- **Purpose:** Default performer for all tracks that don't specify their own
- **Example:** `PERFORMER "The Beatles"`
- **Notes:** Written to CD-TEXT if burning

#### SONGWRITER (Global)
```
SONGWRITER "songwriter_name"
```
- **Purpose:** Default songwriter for all tracks without their own SONGWRITER
- **Example:** `SONGWRITER "John Lennon"`
- **Notes:** Related to CD-TEXT songwriter field

#### TITLE (Global)
```
TITLE "disc_title"
```
- **Purpose:** Disc/album title
- **Example:** `TITLE "Abbey Road"`
- **Notes:** Default title for all tracks without individual titles

### 3.3 FILE Declaration
```
FILE "filename" [filetype]
```
- **Purpose:** Associates subsequent tracks with a specific audio file
- **filetype:** BINARY, MOTOROLA, AIFF, WAVE, MP3
  - `BINARY`: Little-endian raw PCM
  - `MOTOROLA`: Big-endian raw PCM
  - `AIFF`: Apple AIFF format
  - `WAVE`: Microsoft WAV format
  - `MP3`: MPEG-1 Layer 3
- **Example:** `FILE "track01.wav" WAVE`
- **Notes:** All subsequent TRACK declarations belong to this file until a new FILE is declared

### 3.4 Track Commands
#### TRACK
```
TRACK <track_number> <mode>
```
- **Purpose:** Defines a new track
- **track_number:** Integer 1–99, must be sequential
- **mode:** AUDIO, CDG, MODE1/2048, MODE1/2352, MODE2/2336, MODE2/2352, CDI/2336, CDI/2352
  - `AUDIO`: CD-DA audio track
  - `CDG`: Karaoke CD+G track
  - `MODE1/2048`: Data track, 2048 bytes/sector
  - `MODE1/2352`: Data track with CD-ROM sync/header/ECC
  - `MODE2/2336`: CD-ROM XA data
  - `MODE2/2352`: CD-ROM XA with form 1/2
  - `CDI/2336`: CDI data track
  - `CDI/2352`: CDI data with sync
- **Example:** `TRACK 01 AUDIO`

#### FLAGS
```
FLAGS [flag] [flag] [...]
```
- **Purpose:** Sets copy/scanner protection flags
- **Available flags:**
  - `DCP` (Digital Copy Permitted): No copy protection, can be digitally copied
  - `4CH` (Four Channel): Track contains four-channel audio (rare, quadraphonic)
  - `PRE` (Pre-emphasis): Track was recorded with pre-emphasis; de-emphasis needed on playback
  - `SCMS` (Serial Copy Management System): Prohibits further digital copies
- **Example:** `FLAGS DCP PRE`
- **Notes:** DCP and SCMS are mutually exclusive. PRE flag tells decoder to apply de-emphasis.

#### ISRC
```
ISRC <isrc_code>
```
- **Purpose:** International Standard Recording Code for this track
- **Format:** 12-character alphanumeric string: CC-OOO-YY-SSSSS
  - CC: Country code (2 letters)
  - OOO: Registrant code (3 alphanumeric)
  - YY: Last two digits of year (2 digits)
  - SSSSS: Unique recording identifier (5 digits)
- **Example:** `ISRC USRC17700001`
- **Validation:** Must be exactly 12 characters; stored in CD-TEXT

#### PERFORMER (Track)
```
PERFORMER "performer_name"
```
- **Purpose:** Performer for this specific track (overrides global PERFORMER)
- **Example:** `PERFORMER "The Beatles"`

#### SONGWRITER (Track)
```
SONGWRITER "songwriter_name"
```
- **Purpose:** Songwriter for this specific track
- **Example:** `SONGWRITER "Lennon-McCartney"`

#### TITLE (Track)
```
TITLE "track_title"
```
- **Purpose:** Title for this specific track
- **Example:** `TITLE "Come Together"`

#### PREGAP
```
PREGAP <mm:ss:ff>
```
- **Purpose:** Length of silence before the track starts (not stored in audio file)
- **Format:** MSF (Minutes:Seconds:Frames)
- **Notes:** Mutually exclusive with INDEX 00. PREGAP data is generated silence.
- **Example:** `PREGAP 00:02:00` (2 seconds of silence)

#### POSTGAP
```
POSTGAP <mm:ss:ff>
```
- **Purpose:** Length of silence after the track ends (not stored in audio file)
- **Format:** MSF
- **Notes:** Typically 0 or 2 seconds. Data is generated silence.
- **Example:** `POSTGAP 00:02:00`

#### INDEX
```
INDEX <index_number> <mm:ss:ff>
```
- **Purpose:** Defines a point within the FILE (offset from file start)
- **index_number:** 00–99 (two digits)
  - `INDEX 00`: Start of pregap (data present in FILE, before track content)
  - `INDEX 01`: Track start point (where CD player starts playing)
  - `INDEX 02–99`: Sub-indexes within a track
- **Format:** mm:ss:ff (MSF)
  - mm: Minutes (0–99)
  - ss: Seconds (0–59)
  - ff: Frames (0–74)
  - 75 frames = 1 second (1 second = 75 frames)
- **Example:** `INDEX 00 00:00:00` followed by `INDEX 01 00:02:00`
- **Notes:** INDEX 01 is mandatory for each track. INDEX 00 is optional (use instead of PREGAP to include pregap data from the FILE).

---

## 4. TIME FORMAT SPECIFICATION

### 4.1 MSF (Minutes:Seconds:Frames) Format
CD audio uses a specific time format inherited from CD-DA:

```
MSF = MM:SS:FF
  MM = minutes (00–99)
  SS = seconds (00–59)
  FF = frames  (00–74)
  
Total frames = (MM × 60 × 75) + (SS × 75) + FF
Total samples at 44.1kHz = total_frames × 588 samples
```

**Conversion Examples:**
| MSF | Total Frames | Total Samples (44.1kHz) | Total Time |
|-----|-------------|------------------------|-----------|
| 00:00:00 | 0 | 0 | 0:00 |
| 00:00:01 | 1 | 588 | 0:00.023 |
| 00:01:00 | 75 | 44,100 | 0:01.000 |
| 00:02:30 | 187 | 110,250 | 0:02.500 |
| 01:00:00 | 4,500 | 2,646,000 | 1:00.000 |
| 03:12:45 | 14,520 | 8,541,760 | 3:12.600 |

### 4.2 MSF to Byte Offset
To convert MSF to byte offset in a WAVE file:
```python
def msf_to_samples(mm, ss, ff, sample_rate=44100):
    total_frames = (mm * 60 * 75) + (ss * 75) + ff
    samples_per_frame = sample_rate / 75  # 588 samples/frame at 44.1kHz
    return int(total_frames * samples_per_frame)

def samples_to_bytes(samples, channels=2, bits_per_sample=16):
    return samples * channels * (bits_per_sample // 8)
```

### 4.3 Frame vs. Sample Rate Tables
| Sample Rate | Samples/Frame | Bytes/Sample (stereo 16-bit) |
|------------|--------------|-------------------------------|
| 44100 Hz | 588 samples | 1176 bytes |
| 48000 Hz | 640 samples | 1280 bytes |
| 96000 Hz | 1280 samples | 2560 bytes |

### 4.4 Special Time Values
| Time | Meaning |
|------|---------|
| 00:00:00 | File start / track 1 start |
| INDEX 00 00:00:00 | Pregap start (first track must have this) |
| INDEX 01 00:02:00 | Track data start (2-second pregap in file) |
| 79:59:74 | Maximum CD time (just before lead-out) |

---

## 5. PREGAP HANDLING — DETAILED

### 5.1 Pregap Methods
There are two ways to encode a track's pregap in a CUE sheet:

**Method A: INDEX 00 (Pregap data stored in FILE)**
```
INDEX 00 00:00:00    ; Pregap start (at file position 00:00:00)
INDEX 01 00:02:00    ; Track starts at file position 00:02:00
```
- Pregap audio data exists in the file between INDEX 00 and INDEX 01
- The difference in timecodes = pregap length (2 seconds in this example)
- Player must seek to INDEX 01 when starting playback

**Method B: PREGAP command (Pregap is silence, not stored)**
```
PREGAP 00:02:00      ; 2 seconds of silence before track
INDEX 01 00:00:00    ; Track starts at file position 00:00:00
```
- Pregap audio is not stored in the file
- Player generates silence for the pregap duration
- INDEX 01 points to where the actual track data starts

### 5.2 Mutual Exclusivity
```
INDEX 00 and PREGAP are MUTUALLY EXCLUSIVE for a single track.
```
- If you use INDEX 00, do NOT use PREGAP
- If you use PREGAP, do NOT use INDEX 00
- The choice depends on whether the source material has pregap data

### 5.3 First Track Pregap
The first track of a CD always has a pregap of at least 2 seconds (required by the Red Book standard):
```
FILE "disc.wav" WAVE
  TRACK 01 AUDIO
    INDEX 00 00:00:00    ; Pregap start
    INDEX 01 00:02:00    ; Track 01 starts here (2-second pregap)
```

### 5.4 Pregap Extraction
To extract pregap audio from a CUE sheet:
```bash
# Extract pregap between INDEX 00 and INDEX 01 as silence
ffmpeg -i source.wav -ss 00:00:00 -t 00:02:00 -c:a copy pregap.wav

# For CUE sheets with PREGAP (no data):
# Pregap must be generated as silence
ffmpeg -f lavfi -i anullsrc=r=44100:cl=stereo -t 00:02:00 pregap_silence.wav
```

---

## 6. COMPLETE CUE SHEET EXAMPLES

### 6.1 Single-Track Disc (Image)
```
CATALOG 0123456789012
TITLE "Greatest Hits"
PERFORMER "The Artist"
FILE "disc.wav" WAVE
  TRACK 01 AUDIO
    TITLE "Song One"
    PERFORMER "The Artist"
    INDEX 01 00:00:00
```

### 6.2 Multi-Track with Pregaps (INDEX 00)
```
CATALOG 0123456789012
TITLE "Live Concert"
PERFORMER "The Band"
FILE "live_show.wav" WAVE
  TRACK 01 AUDIO
    TITLE "Intro"
    FLAGS DCP
    INDEX 00 00:00:00
    INDEX 01 00:03:24
  TRACK 02 AUDIO
    TITLE "Song One"
    FLAGS DCP
    ISRC USRC17700001
    INDEX 00 00:03:24
    INDEX 01 00:05:48
  TRACK 03 AUDIO
    TITLE "Song Two"
    FLAGS DCP PRE
    ISRC USRC17700002
    INDEX 00 00:05:48
    INDEX 01 00:08:12
```

### 6.3 Multi-Track with PREGAP (Silence)
```
CATALOG 0123456789012
TITLE "Album Title"
PERFORMER "Artist Name"
FILE "album.wav" WAVE
  TRACK 01 AUDIO
    TITLE "Track 1"
    PREGAP 00:02:00
    INDEX 01 00:00:00
  TRACK 02 AUDIO
    TITLE "Track 2"
    PREGAP 00:02:00
    INDEX 01 00:00:00
  TRACK 03 AUDIO
    TITLE "Track 3"
    PREGAP 00:02:00
    INDEX 01 00:00:00
```

### 6.4 Multi-File CUE Sheet
```
TITLE "Compilation CD"
PERFORMER "Various Artists"
FILE "side_a.wav" WAVE
  TRACK 01 AUDIO
    TITLE "Track A1"
    PERFORMER "Artist 1"
    INDEX 01 00:00:00
  TRACK 02 AUDIO
    TITLE "Track A2"
    PERFORMER "Artist 1"
    INDEX 01 03:45:00
FILE "side_b.wav" WAVE
  TRACK 03 AUDIO
    TITLE "Track B1"
    PERFORMER "Artist 2"
    INDEX 01 00:00:00
  TRACK 04 AUDIO
    TITLE "Track B2"
    PERFORMER "Artist 2"
    INDEX 01 04:12:00
```

### 6.5 CD with CDTEXTFILE
```
CATALOG 0123456789012
TITLE "Classic Album"
PERFORMER "Orchestra"
CDTEXTFILE "orchestra.cdt"
FILE "classical.wav" WAVE
  TRACK 01 AUDIO
    TITLE "Movement 1"
    PERFORMER "London Symphony Orchestra"
    SONGWRITER "Beethoven"
    ISRC GBAUV1900001
    INDEX 01 00:00:00
  TRACK 02 AUDIO
    TITLE "Movement 2"
    PERFORMER "London Symphony Orchestra"
    SONGWRITER "Beethoven"
    ISRC GBAUV1900002
    INDEX 01 12:34:56
```

### 6.6 Data + Audio Mixed Disc
```
CATALOG 9876543210987
FILE "mixed_mode.iso" BINARY
  TRACK 01 MODE1/2048
    TITLE "Data Session"
    INDEX 01 00:00:00
FILE "audio.wav" WAVE
  TRACK 02 AUDIO
    TITLE "Bonus Track"
    INDEX 01 00:00:00
```

---

## 7. FFMPEG CUE SHEET INTEGRATION

### 7.1 FFmpeg CUE Demuxer
FFmpeg has a native CUE demuxer that reads CUE files:
```bash
# Split by cue sheet into individual tracks
ffmpeg -i "disc.cue" -f segment -segment_time 180 track_%02d.wav

# Use segment muxer with cue
ffmpeg -i "input.wav" -f cue -scanning 1 -i "disc.cue" -c:a flac output_%03d.flac
```

### 7.2 CUE-Based Splitting with FFmpeg
```bash
# Method 1: Manual split using cue info
# Extract track 2 using start time and duration
ffmpeg -i disc.wav -ss 00:05:48 -t 00:03:00 -c:a pcm_s16le track02.wav

# Method 2: Using cue2tracks (CUETools)
cue2tracks disc.cue

# Method 3: Using EAC + CUETools
# EAC can export with embedded cuesheet
# CUETools can split based on cue

# Method 4: Using FFmpeg concat with cue
# Parse cue file to generate segment times
```

### 7.3 CUE Sheet Parsing with FFmpeg
```bash
# FFmpeg's cue demuxer options:
# -i "file.cue" 
#   -f cue activates cue demuxer
#   -scan_file <file> use audio file
#   -boundary_scan <n> scanning offset

# Example with explicit audio file
ffmpeg -i disc.cue \
  -f s16le -ar 44100 -ac 2 \
  -ss 00:05:48 \
  -t 00:03:00 \
  -c:a pcm_s16le \
  track02.wav
```

### 7.4 Converting CUE + Audio to Individual Tracks
```bash
# Using cuetools.sh (from CUETools package)
cuetools.sh split --cue disc.cue --input disc.wav --output track --format track%02d.wav

# Using shnsplit (from soundlib)
shnsplit -f disc.cue -t "%n - %t" -o wav disc.ape

# Using bchunk (for BIN/CUE)
bchunk -w disc.bin disc.cue track

# Using FFmpeg + manual parsing
# Requires scripting to extract start times from CUE
```

### 7.5 Gapless Extraction with CUE
For gapless extraction of a CUE+image pair:
```bash
# Extract full disc image
ffmpeg -i disc.wav -c:a copy disc_copy.wav

# Extract individual tracks with proper trimming
# Track 1: start=0, end=INDEX01 of track 2
# Track 2: start=INDEX01 of track 2, end=INDEX01 of track 3
# etc.

# Proper pregap handling
# If INDEX 00 exists: include pregap data
# If PREGAP exists: generate silence before INDEX 01
```

---

## 8. CD-IMAGE FORMATS WITH CUE

### 8.1 Supported Image Formats
| Format | CUE Support | Notes |
|--------|-------------|-------|
| WAVE (.wav) | Yes | Uncompressed PCM |
| AIFF (.aiff) | Yes | Apple format |
| FLAC (.flac) | Yes | Lossless compressed |
| WV (.wv) | Yes | WavPack |
| APE (.ape) | Yes | Monkey's Audio |
| ISO (.iso) | Partial | Data+audio mixed mode |
| BIN (.bin) + CUE | Yes | Raw CD image + CUE |

### 8.2 FLAC + CUE for Archival
```
# Create FLAC image from WAV + CUE
ffmpeg -i disc.wav -c:a flac -compression_level 8 disc.flac

# Split FLAC using CUE
# Method 1: cuetools
flac2cue disc.flac disc.cue
# Then use cuetools to split

# Method 2: shnsplit
shnsplit -f disc.cue -o flac -t "%n - %t" disc.flac

# Method 3: FFmpeg + manual
# Parse CUE to get track boundaries
```

### 8.3 BIN + CUE for CD Archival
```
# BIN is a raw dump of CD sectors (2352 bytes/sector for audio)
# CUE describes the sector layout

# Extract audio tracks from BIN/CUE
bchunk -w disc.bin disc.cue track

# Convert to WAV
bchunk disc.bin disc.cue track
# Produces: track01.wav, track02.wav, ...

# Verify sector alignment
# Audio sector: 2352 bytes = 588 samples × 4 bytes (stereo 16-bit)
# DATA sector: 2048 bytes = MODE1
```

---

## 9. MUSICBrainz CUE SHEET CONVENTIONS

### 9.1 MusicBrainz Disc ID
MusicBrainz calculates a disc ID from the TOC (table of contents):
```
CD TOC entries: track 1 start, track 2 start, ..., lead-out start
Disc ID = SHA1 of TOC entries
```
CUE sheets exported from MusicBrainz include the CATALOG number and accurate pregap information.

### 9.2 MusicBrainz CUE Export Format
```
CATALOG <ean>
<global metadata>
FILE "filename.wav" WAVE
  TRACK 01 AUDIO
    FLAGS DCP
    INDEX 00 <pregap_frames>
    INDEX 01 <track_start_frames>
  TRACK 02 AUDIO
    FLAGS DCP
    INDEX 00 <pregap_frames>
    INDEX 01 <track_start_frames>
  ...
```

### 9.3 Pregap Detection
MusicBrainz uses offsets from the lead-out:
- Track N start = lead-out offset - sum of all track durations after N
- This is more accurate than relying on audio content

---

## 10. CUE PARSER IMPLEMENTATION

### 10.1 Tokenizer
```python
import re

def tokenize_cue(content):
    """Tokenize CUE sheet content into commands."""
    lines = content.split('\n')
    tokens = []
    
    for line in lines:
        # Remove comments
        line = re.sub(r'//.*$|#.*$', '', line).strip()
        if not line:
            continue
        
        # Tokenize: keyword + quoted strings + unquoted values
        parts = re.split(r'(\s+)', line)
        tokens.append(parts)
    
    return tokens

def parse_msf(value):
    """Parse MSF format: MM:SS:FF"""
    match = re.match(r'(\d+):(\d+):(\d+)', value)
    if not match:
        raise ValueError(f"Invalid MSF: {value}")
    mm, ss, ff = map(int, match.groups())
    if ss > 59 or ff > 74:
        raise ValueError(f"Invalid MSF components: {mm}:{ss}:{ff}")
    return (mm, ss, ff)

def msf_to_sectors(mm, ss, ff):
    """Convert MSF to CD sectors (75 sectors/second)."""
    return (mm * 60 + ss) * 75 + ff

def msf_to_samples(mm, ss, ff, sample_rate=44100, channels=2):
    """Convert MSF to byte offset in WAV."""
    sectors = msf_to_sectors(mm, ss, ff)
    samples = sectors * (sample_rate // 75)
    return samples * channels * 2  # 2 bytes per sample (16-bit)
```

### 10.2 Complete CUE Parser
```python
class CueTrack:
    def __init__(self, number, mode='AUDIO'):
        self.number = number
        self.mode = mode
        self.title = None
        self.performer = None
        self.songwriter = None
        self.isrc = None
        self.flags = []
        self.pregap = None
        self.postgap = None
        self.indexes = {}  # {index_number: msf_tuple}

class CueSheet:
    def __init__(self):
        self.catalog = None
        self.cdtextfile = None
        self.performer = None
        self.songwriter = None
        self.title = None
        self.files = []  # [('filename', 'filetype')]
        self.tracks = []
    
    def parse_file(self, filename):
        with open(filename, 'r', encoding='utf-8', errors='replace') as f:
            content = f.read()
        
        current_file = None
        current_track = None
        
        for line in content.split('\n'):
            line = re.sub(r'//.*$|#.*$', '', line).strip()
            if not line:
                continue
            
            # Parse command
            parts = re.split(r'\s+(?=(?:[^"]*"[^"]*")*$)', line)
            cmd = parts[0].upper()
            
            if cmd == 'CATALOG':
                self.catalog = parts[1].strip('"')
            elif cmd == 'TITLE':
                self.title = parts[1].strip('"')
            elif cmd == 'PERFORMER':
                self.performer = parts[1].strip('"')
            elif cmd == 'SONGWRITER':
                self.songwriter = parts[1].strip('"')
            elif cmd == 'CDTEXTFILE':
                self.cdtextfile = parts[1].strip('"')
            elif cmd == 'FILE':
                filename = parts[1].strip('"')
                filetype = parts[2].upper() if len(parts) > 2 else 'WAVE'
                self.files.append((filename, filetype))
                current_file = len(self.files) - 1
            elif cmd == 'TRACK':
                track_num = int(parts[1])
                mode = parts[2].upper()
                current_track = CueTrack(track_num, mode)
                self.tracks.append(current_track)
            elif cmd == 'FLAGS':
                if current_track:
                    current_track.flags = parts[1:]
            elif cmd == 'ISRC':
                if current_track:
                    current_track.isrc = parts[1]
            elif cmd == 'TITLE' and current_track:
                current_track.title = parts[1].strip('"')
            elif cmd == 'PERFORMER' and current_track:
                current_track.performer = parts[1].strip('"')
            elif cmd == 'SONGWRITER' and current_track:
                current_track.songwriter = parts[1].strip('"')
            elif cmd == 'PREGAP':
                if current_track:
                    current_track.pregap = parse_msf(parts[1])
            elif cmd == 'POSTGAP':
                if current_track:
                    current_track.postgap = parse_msf(parts[1])
            elif cmd == 'INDEX':
                if current_track and len(parts) >= 3:
                    idx_num = int(parts[1])
                    msf = parse_msf(parts[2])
                    current_track.indexes[idx_num] = msf
        
        return self
```

### 10.3 CUE-Based Track Extraction
```python
def extract_tracks(cue_file, audio_file, output_dir):
    """Extract individual tracks based on CUE sheet."""
    import os
    import struct
    import wave
    
    cuesheet = CueSheet().parse_file(cue_file)
    
    # Get audio properties
    with wave.open(audio_file, 'rb') as w:
        channels = w.getnchannels()
        sampwidth = w.getsampwidth()
        framerate = w.getframerate()
    
    prev_index01_sectors = 0
    
    for i, track in enumerate(cuesheet.tracks):
        # Get INDEX 01
        if 1 not in track.indexes:
            print(f"Track {track.number}: No INDEX 01, skipping")
            continue
        
        # Start sector (INDEX 01 of this track)
        start_mm, start_ss, start_ff = track.indexes[1]
        start_sectors = msf_to_sectors(start_mm, start_ss, start_ff)
        
        # End sector (INDEX 01 of next track, or EOF)
        if i + 1 < len(cuesheet.tracks):
            next_track = cuesheet.tracks[i + 1]
            next_mm, next_ss, next_ff = next_track.indexes[1]
            end_sectors = msf_to_sectors(next_mm, next_ss, next_ff)
        else:
            # Read WAV header to get total samples
            with wave.open(audio_file, 'rb') as w:
                n_frames = w.getnframes()
            end_sectors = n_frames * 75 // framerate
        
        # Calculate durations
        track_sectors = end_sectors - start_sectors
        duration_samples = track_sectors * (framerate // 75)
        duration_secs = track_sectors / 75.0
        
        # Build output filename
        title = track.title or f"Track{track.number:02d}"
        safe_title = re.sub(r'[<>:"/\\|?*]', '_', title)
        output_file = os.path.join(output_dir, f"{track.number:02d} - {safe_title}.wav")
        
        # Extract with FFmpeg
        start_time = start_sectors / 75.0
        duration = duration_secs
        
        import subprocess
        subprocess.run([
            'ffmpeg', '-y', '-i', audio_file,
            '-ss', str(start_time),
            '-t', str(duration),
            '-c:a', 'pcm_s16le',
            output_file
        ], check=True)
        
        # Add metadata
        subprocess.run([
            'ffmpeg', '-y', '-i', output_file,
            '-metadata', f'title={title}',
            '-metadata', f'artist={track.performer or cuesheet.performer}',
            '-metadata', f'track={track.number}',
            '-c:a', 'copy',
            output_file + '.tmp'
        ], check=True)
        os.replace(output_file + '.tmp', output_file)
        
        print(f"Extracted: {output_file} ({duration_secs:.2f}s)")
```

---

## 11. EDGE CASES & KNOWN ISSUES

### 11.1 Common Pitfalls
| Issue | Cause | Solution |
|-------|-------|----------|
| Wrong sample rate | CUE uses 44.1kHz sectors, but file is 48kHz | Verify file sample rate matches CUE expectations |
| Pregap not extracted | INDEX 00 vs PREGAP confusion | Check if pregap data exists in file |
| Negative duration | INDEX ordering wrong | Ensure INDEX numbers are sequential |
| Wrong file reference | Multiple FILE declarations | Track belongs to most recent FILE |
| Binary vs Motorola | Endianness confusion | Check byte order for RAW CD images |

### 11.2 Compatibility Issues
| Tool | Issue | Workaround |
|------|-------|------------|
| EAC | Generates non-standard CUE | Post-process with cuetools |
| XLD | Strict about MSF format | Use exactly MM:SS:FF format |
| Brazero | Limited FLAC+CUE support | Convert to WAV+CUE |
| Windows Media Player | No CUE support | Use foobar2000 or mp3tag |
| MusicBee | May reorder tracks | Verify track numbering |

### 11.3 Special Cases
```
First track pregap: Always present, minimum 2 seconds (Red Book requirement)
Last track: INDEX 01 points to track start; no next INDEX
Lead-out: Not explicitly in CUE; implied by EOF or next file
Mode switches: Changing MODE in middle of file requires new FILE
```

---

## 12. CONVERSION GUIDE

### 12.1 CUE + WAV → Individual FLAC Tracks
```bash
# Extract with shnsplit
shnsplit -f disc.cue -o flac -t "%n - %t" disc.wav

# Add tags with cuetag
cuetag disc.cue track01.flac track02.flac ...

# Or use FFmpeg
# 1. Parse CUE to get track boundaries
# 2. Extract each track with -ss and -t
```

### 12.2 Individual Tracks → CUE + WAV
```bash
# Concatenate tracks into image
ffmpeg -i "concat:track01.wav|track02.wav|track03.wav" -c:a pcm_s16le disc.wav

# Generate CUE from track lengths
# Use cuetools or manual generation
```

### 12.3 BIN/CUE → FLAC/CUE
```bash
# Convert BIN to WAV first
bchunk -w disc.bin disc.cue disc

# Then FLAC the WAV image
ffmpeg -i disc.wav -c:a flac -compression_level 8 disc.flac

# Copy CUE (or convert TOC)
cp disc.cue disc.flac.cue
```

---

## 13. REFERENCE IMPLEMENTATIONS

| Tool | Language | License | Notes |
|------|----------|---------|-------|
| cuetools | Python | GPL | CUE parsing, splitting, verification |
| shntool | C | BSD | Audio splitting and manipulation |
| bchunk | C | GPL | BIN/CUE conversion |
| cdrdao | C | GPL | CD burning from TOC/CUE |
| EAC | C++ | Proprietary | CD ripping with CUE export |
| FFmpeg | C | LGPL | CUE demuxer, segment extraction |
| CUETools | C# | GPL | Advanced CUE processing |

---

## 14. SPECIFICATIONS & FURTHER READING

### Primary Specifications
- **CD-DA Red Book (IEC 60908):** Physical CD audio standard
- **CD-ROM/XA Green Book:** Extended architecture
- **CDTEXTFILE:** Binary CD-TEXT format
- **MusicBrainz CUE conventions:** https://musicbrainz.org/doc/CUE_Tutorial

### Technical Resources
- Hydrogenaudio CUE sheet: https://wiki.hydrogenaudio.org/index.php?title=Cue_sheet
- CDRDAO documentation: https://cdrdao.sourceforge.net/
- GNU ccd2cue: https://www.gnu.org/software/ccd2cue/
- libyal/libodraw: https://github.com/libyal/libodraw

---

## 15. IMPLEMENTATION CHECKLIST

### Parsing Pipeline
- [ ] Read entire CUE file as text
- [ ] Strip comments (// and #)
- [ ] Parse global commands (CATALOG, CDTEXTFILE, TITLE, PERFORMER, SONGWRITER)
- [ ] Track FILE declarations and associate subsequent tracks
- [ ] Parse TRACK commands (number, mode)
- [ ] Parse track-level commands (FLAGS, ISRC, TITLE, PERFORMER, SONGWRITER, PREGAP, POSTGAP, INDEX)
- [ ] Validate MSF format (MM:SS:FF, 75 frames/sec, 60 sec/min)
- [ ] Validate INDEX ordering (00 before 01, sequential)
- [ ] Handle PREGAP/INDEX 00 mutual exclusivity

### Metadata Pipeline
- [ ] Read global metadata (applies to all tracks without their own)
- [ ] Read track-level metadata (overrides global)
- [ ] Read FLAGS (DCP, 4CH, PRE, SCMS)
- [ ] Read ISRC codes
- [ ] Build complete metadata for each track

### Extraction Pipeline
- [ ] Calculate start/end sample positions from MSF times
- [ ] Handle PREGAP (generate silence vs. extract from file)
- [ ] Handle INDEX 00 (include pregap data from file)
- [ ] Verify audio file sample rate matches 44.1kHz assumption
- [ ] Extract audio segments using FFmpeg
- [ ] Apply metadata to extracted tracks
- [ ] Handle multi-file CUE sheets

### Edge Cases
- [ ] Handle missing INDEX 01
- [ ] Handle INDEX 00 without INDEX 01
- [ ] Handle PREGAP with zero duration
- [ ] Handle mixed audio/data tracks
- [ ] Handle non-standard MSF values (e.g., >74 frames)
- [ ] Handle non-ASCII characters in titles
- [ ] Handle missing CATALOG number
- [ ] Handle non-standard track numbers (not starting at 1)

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
