# 72_Open_Source_Equivalent_Analysis.md

## Open-Source Equivalent Analysis

> **Purpose:** Deep analysis of how open-source tools implement DBpoweramp-equivalent behavior. Which tool does each feature best? How close does each come to DBpoweramp's behavior?

---

## SECTION 1: OPEN-SOURCE TOOL OVERVIEW

### 1.1 Tools Analyzed

| Tool | Language | Tag Library | Best For |
|------|----------|------------|----------|
| **fre:ac** | C++ (wxWidgets) | TagLib | CD ripping, batch conversion |
| **MusicBrainz Picard** | Python | mutagen | Metadata lookup, tagging |
| **beets** | Python | mediafile.py (mutagen) | Library management, auto-tagging |
| **Mp3tag** | C++ (MFC) | TagLib | Tag editing, batch operations |
| **TagLib** | C++ | N/A | Library, not a tool |
| **mutagen** | Python | N/A | Library, not a tool |

### 1.2 Comparison Matrix

| Feature | DBpoweramp | fre:ac | Picard | beets | Mp3tag | TagLib | mutagen |
|---------|------------|--------|--------|-------|--------|--------|---------|
| Cross-format conversion | ✓ | ✓ | ✗ | ✗ | ✗ | ✓ | ✓ |
| CD ripping | ✓ | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ |
| AccurateRip verification | ✓ | ✗ | ✗ | ✗ | ✗ | ✗ | ✗ |
| Multi-format metadata lookup | ✓ (5 sources) | ✓ (2-3) | ✓ (MB + AcoustID) | ✓ (MB) | ✗ | ✗ | ✗ |
| Tag field mapping | ✓ | ~ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Cover art handling | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| ReplayGain support | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| R128 (Opus) | ✓ | ✓ | ✓ | ✓ | ✗ | ✓ | ✓ |
| Batch processing | ✓ | ✓ | ✓ | ✓ | ✓ | Via lib | Via lib |
| Scripting/automation | DSP | Limited | ✓ (Python) | ✓ (Python) | Limited | Via lib | Via lib |
| Atomic writes | ✓ | ✓ | ~ | ~ | ✓ | Via lib | Via lib |

---

## SECTION 2: FRE:AC ANALYSIS

### 2.1 Architecture Overview

**fre:ac** (formerly BonkEnc) is a free audio converter and CD ripper for Windows, macOS, Linux, and FreeBSD.

**Repository:** https://github.com/enzo1982/freac

**Architecture:**
```
┌─────────────────────────────────────────────────────┐
│                    fre:ac UI                        │
│         (wxWidgets, cross-platform GUI)             │
└──────────────────┬──────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────┐
│              BoCA Audio Processing                  │
│  ┌─────────────────────────────────────────────┐  │
│  │  Pipeline: Source → Decoder → DSP → Encoder  │  │
│  └─────────────────────────────────────────────┘  │
│                                                     │
│  ┌─────────────────────────────────────────────┐  │
│  │  Tag Processing Layer                        │  │
│  │  (based on TagLib + custom extensions)      │  │
│  └─────────────────────────────────────────────┘  │
└──────────────────┬──────────────────────────────────┘
                   │
┌──────────────────▼──────────────────────────────────┐
│              TagLib Library                         │
│         (tag reading/writing core)                  │
└─────────────────────────────────────────────────────┘
```

### 2.2 BoCA Audio Conversion Architecture

**BoCA** (Bontempi Clearlooks Audio) is fre:ac's internal processing framework:

```cpp
// Simplified BoCA pipeline concept
class ConversionJob {
    SourceFile* source;
    TargetFormat* target;
    std::vector<DSP::Component*> dsp_chain;
    
    void Execute() {
        // 1. Open source
        Decoder* decoder = CreateDecoder(source);
        
        // 2. Decode to PCM
        Buffer* buffer = decoder->Read();
        
        // 3. Apply DSP chain
        for (auto& dsp : dsp_chain) {
            buffer = dsp->Process(buffer);
        }
        
        // 4. Encode to target format
        Encoder* encoder = CreateEncoder(target);
        encoder->Write(buffer);
        
        // 5. Copy tags
        TagHandler* handler = GetTagHandler(target->format);
        handler->CopyTags(source, target);
    }
};
```

**Tag handling in BoCA:**
- Uses TagLib for low-level tag I/O
- Custom tag mapping layer for cross-format conversion
- DSP framework for tag manipulation during conversion

### 2.3 fre:ac Tag Handling Approach

**Strengths:**
1. **Clean pipeline**: Tag copying is integrated into the conversion pipeline
2. **TagLib foundation**: Leverages TagLib's format coverage
3. **DSP tags**: Can apply DSP effects to tags (not just audio)

**Weaknesses:**
1. **TagLib limitations**: Inherits TagLib's multi-value tag handling issues
2. **No metadata lookup during conversion**: Must pre-fetch metadata
3. **Less sophisticated tag mapping** than DBpoweramp

### 2.4 fre:ac vs DBpoweramp Tag Preservation

| Feature | DBpoweramp | fre:ac | Gap |
|---------|------------|--------|-----|
| Basic fields | ✓ | ✓ | None |
| Multi-artist | ✓ | ✓ | None |
| Track/disc totals | ✓ | ✓ | None |
| Sort fields | ✓ | ✓ | None |
| Cover art type | 3 (front) | ? | Unknown |
| ReplayGain | ✓ | ✓ | None |
| R128 | ✓ | ✓ | None |
| Work/Movement | ✓ | Limited | Moderate |
| Custom fields | ✓ | Limited | Moderate |
| Atomic writes | ✓ | ✓ | None |

### 2.5 fre:ac DSP Framework

fre:ac includes a DSP framework that can manipulate tags:

```cpp
// Example DSP component for tag manipulation
class NormalizeTagDSP : public DSP::Component {
    void ProcessTags(Tag* tag) override {
        // Normalize artist capitalization
        // Resolve genre numbers
        // Fix encoding issues
    }
};
```

**Available DSP effects for tags:**
- Genre normalization
- Character encoding correction
- Multi-value tag handling
- Case normalization

### 2.6 fre:ac Tag Mapping Reference

Based on source code analysis of fre:ac's tag handling:

| Source Field | Target Field | Notes |
|-------------|--------------|-------|
| TITLE | TITLE | Direct |
| ARTIST | ARTIST | Direct |
| ALBUM | ALBUM | Direct |
| ALBUMARTIST | ALBUMARTIST | Direct |
| COMPOSER | COMPOSER | Direct |
| TRACKNUMBER | TRACKNUMBER | Parses "5/12" format |
| DISCNUMBER | DISCNUMBER | Parses "1/3" format |
| DATE/YEAR | DATE | Normalizes format |
| GENRE | GENRE | Resolves ID3v1 numbers |
| COVER | COVER | Embedded art |
| REPLAYGAIN_* | REPLAYGAIN_* | Preserved |

---

## SECTION 3: MUSICBRAINZ PICARD ANALYSIS

### 3.1 Architecture Overview

**MusicBrainz Picard** is a cross-platform music tagger written in Python (Qt GUI).

**Repository:** https://github.com/metabrainz/picard

**Key components:**
```python
# Picard architecture (simplified)
class Picard:
    def __init__(self):
        self.tagger = Tagger()  # Core tagging engine
        self.metadata = Metadata()  # Metadata store
        
    def load_file(self, path):
        # 1. Detect format
        # 2. Read existing tags
        # 3. Store in Metadata object
        pass
    
    def lookup_metadata(self, fingerprint):
        # 1. AcoustID fingerprint
        # 2. Query MusicBrainz
        # 3. Return metadata
        pass
    
    def save_file(self):
        # 1. Apply metadata
        # 2. Write tags using mutagen
        pass
```

### 3.2 Picard Tag Mapping Completeness

Picard uses **mutagen** for tag I/O and implements comprehensive tag mapping:

**Tag Mapping Table (from Picard source):**

| MusicBrainz Field | FLAC/Vorbis | MP3 | MP4 | Notes |
|-------------------|-------------|-----|-----|-------|
| title | TITLE | TIT2 | ©nam | |
| artist | ARTIST | TPE1 | ©ART | |
| album | ALBUM | TALB | ©alb | |
| albumartist | ALBUMARTIST | TPE2 | aART | |
| composer | COMPOSER | TCOM | ©wrt | |
| conductor | CONDUCTOR | TPE3 | ©con | |
| tracknumber | TRACKNUMBER | TRCK | trkn | (n, m) tuple |
| discnumber | DISCNUMBER | TPOS | diskn | (n, m) tuple |
| date | DATE | TDRC/TYER | ©day | Normalized to YYYY-MM-DD |
| genre | GENRE | TCON | ©gen | |
| lyricist | LYRICIST | TEXT | ©lyr | |
| producer | PRODUCER | TIPL/producer | ©prd | |
| arranger | ARRANGER | TIPL/arranger | ©prd | |
| engineer | ENGINEER | TIPL/engineer | ©eng | |
| writer | WRITER | TIPL/writer | ©lyr | |
| isrc | ISRC | TSRC | ©isr | |
| copyright | COPYRIGHT | TCOP | cprt | |
| comment | COMMENT | COMM | ©cmt | |
| mood | MOOD | TMOO | ©mood | |
| discsubtitle | DISCSUBTITLE | TSST | ©sdt | |
| encoder | ENCODER | TENC | ©enc | |
| encodersettings | ENCODERSETTINGS | TSSE | | |
| albumtype | ALBUMTYPE | TIT2 | ©mvc | Special handling |
| releasestatus | RELEASESTATUS | | | Custom |
| releasetype | RELEASETYPE | | | Custom |
| barcode | BARCODE | TXXX:BARCODE | | |
| catalognumber | CATALOGNUMBER | TXXX:CATALOGNUMBER | | |
| discid | DISCID | TXXX:DISCID | | |
| MusicBrainz Recording ID | MUSICBRAINZ_RECORDINGID | TXXX:MUSICBRAINZ_RELEASetrackid | -- |
| MusicBrainz Track ID | MUSICBRAINZ_TRACKID | TXXX:MUSICBRAINZ_TRACKID | | |
| MusicBrainz Album ID | MUSICBRAINZ_ALBUMID | TXXX:MUSICBRAINZ_ALBUMID | | |
| MusicBrainz Artist ID | MUSICBRAINZ_ARTISTID | TXXX:MUSICBRAINZ_ARTISTID | | |
| MusicBrainz Release Group ID | MUSICBRAINZ_RELEASEGROUPID | TXXX:MUSICBRAINZ_RELEASEGROUPID | | |
| MusicBrainz Release ID | MUSICBRAINZ_ALBUMID | TXXX:MUSICBRAINZ_ALBUMID | | |

### 3.3 Picard Cross-Format Tag Preservation

**Strengths:**
1. **Comprehensive field coverage**: Maps all MusicBrainz fields
2. **Format normalization**: Consistent field representation internally
3. **Scripting support**: PICARD_SCRIPT, Python plugins

**Weaknesses:**
1. **Not a converter**: Only tags existing files, doesn't change audio format
2. **No batch conversion pipeline**: Must use external tools for format conversion
3. **Genre handling**: Uses freeform strings, no genre number support

### 3.4 Picard Metadata Normalization

Picard normalizes all metadata internally:

```python
# Picard internal Metadata class (simplified)
class Metadata:
    def __init__(self):
        self.multi_values = {}  # {field: [values]}
        self.single_values = {}  # {field: value}
    
    def normalize(self):
        # 1. Join multi-values for single-value contexts
        # 2. Split single-values for multi-value contexts
        # 3. Normalize date formats
        # 4. Resolve genre numbers
        pass
```

**Normalization rules:**
- Date: All formats → "YYYY-MM-DD"
- Track number: All formats → tuple (track, total)
- Disc number: All formats → tuple (disc, total)
- Genre: ID3v1 numbers → strings

### 3.5 Picard vs DBpoweramp Comparison

| Feature | DBpoweramp | Picard | Assessment |
|---------|------------|--------|------------|
| Field coverage | Excellent | Excellent | Equal |
| Multi-artist | ✓ | ✓ | Equal |
| Sort fields | ✓ | ✓ | Equal |
| Cover art | ✓ | ✓ | Equal |
| Custom fields | ✓ | ✓ | Equal |
| MusicBrainz integration | Via metadata lookup | Native | Picard wins |
| CD ripping | ✓ | ✗ | DBpoweramp wins |
| Format conversion | ✓ | ✗ | DBpoweramp wins |
| Scripting | DSP, scripting | Python, PICARD_SCRIPT | Equal |
| Batch processing | ✓ | ✓ | Equal |

---

## SECTION 4: BEETS ANALYSIS

### 4.1 Architecture Overview

**beets** is a music library manager written in Python. Its tag handling is done through **mediafile.py**, which wraps mutagen.

**Repository:** https://github.com/beetbox/beets
**MediaFile:** https://github.com/beetbox/mediafile

### 4.2 mediafile.py Tag Mapping

mediafile.py is **the most comprehensive open-source tag mapping reference**. It defines how each format stores each field.

**Core architecture:**

```python
# mediafile.py architecture (simplified)
class MediaFile:
    """Represents an audio file with accessible metadata."""
    
    def __init__(self, path, id3v23=False):
        self._path = path
        self._mutagen_file = mutagen.File(path)
        
    # Each field is a MediaField descriptor
    title = MediaField(
        MP3StorageStyle('TIT2'),
        MP4StorageStyle('©nam'),
        VorbisStorageStyle('TITLE'),
        FLACStorageStyle('TITLE'),
        # ...
    )
```

**Field definitions:**

```python
# Complete field definitions from mediafile.py

class MediaFile:
    # Basic identification
    title = MediaField(...)
    artist = MediaField(...)
    album = MediaField(...)
    albumartist = MediaField(...)
    composer = MediaField(...)
    lyricist = MediaField(...)
    arranger = MediaField(...)
    producer = MediaField(...)
    engineer = MediaField(...)
    encoder = MediaField(...)
    
    # Numbering
    track = MediaField(...)  # track number
    tracktotal = MediaField(...)  # total tracks
    disc = MediaField(...)  # disc number
    disctotal = MediaField(...)  # total discs
    discsubtitle = MediaField(...)
    
    # Date
    year = MediaField(...)
    month = MediaField(...)
    day = MediaField(...)
    
    # Classification
    genre = MediaField(...)
    style = MediaField(...)
    mood = MediaField(...)
    
    # Technical
    comments = MediaField(...)
    bpm = MediaField(...)
    initial_key = MediaField(...)
    
    # Identifiers
    isrc = MediaField(...)
    asin = MediaField(...)
    barcode = MediaField(...)
    catalognum = MediaField(...)
    
    # Covers
    images = MediaField(...)  # List of Image objects
    
    # ReplayGain
    rg_track_gain = MediaField(...)
    rg_track_peak = MediaField(...)
    rg_album_gain = MediaField(...)
    rg_album_peak = MediaField(...)
    r128_track_gain = MediaField(...)  # Opus-specific
    
    # MusicBrainz
    mb_trackid = MediaField(...)
    mb_albumid = MediaField(...)
    mb_artistid = MediaField(...)
    mb_releasetrackid = MediaField(...)
    mb_albumtype = MediaField(...)
    mb_albumstatus = MediaField(...)
    mb_albumdiscid = MediaField(...)
    
    # Work/Movement
    work = MediaField(...)
    movement = MediaField(...)
    movementtotal = MediaField(...)
    work_disambig = MediaField(...)
```

### 4.3 StorageStyle Classes

mediafile.py defines StorageStyle subclasses for each format family:

```python
# Storage styles from mediafile.py

# MP3 (ID3v2)
class MP3StorageStyle(StorageStyle):
    formats = ['MPEG']

# Vorbis family (FLAC, OGG, Opus)
class VorbisStorageStyle(StorageStyle):
    formats = ['FLAC', 'OggTheora', 'OggSpeex', 'OggVorbis', 'OggFlac']

# MP4 family
class MP4StorageStyle(StorageStyle):
    formats = ['MPEG-4', 'iTunes']

# Musepack, WavPack, etc.
class APEv2StorageStyle(StorageStyle):
    formats = ['Musepack', 'WavPack', 'APEv2File']

# Key operations
class StorageStyle:
    def get(self, mutagen_file):
        # Get value from mutagen file
        
    def set(self, mutagen_file, value):
        # Set value in mutagen file
        
    def delete(self, mutagen_file):
        # Remove value from mutagen file
```

### 4.4 beets Convert Plugin

The beets convert plugin handles format conversion:

```python
# beets convert plugin (simplified)
class ConvertPlugin(BeetsPlugin):
    def convert(self, source, dest, fmt):
        # 1. Copy audio data
        subprocess.run(['ffmpeg', '-i', source, '-codec:a', fmt, dest])
        
        # 2. Copy tags
        source_mf = MediaFile(source)
        dest_mf = MediaFile(dest)
        
        # 3. Transfer fields
        for field in self.config['fields']:
            setattr(dest_mf, field, getattr(source_mf, field))
        
        # 4. Handle cover art
        for img in source_mf.images:
            dest_mf.add_image(img)
        
        # 5. Save
        dest_mf.save()
```

### 4.5 beets vs DBpoweramp Comparison

| Feature | DBpoweramp | beets/mediafile | Assessment |
|---------|------------|-----------------|------------|
| Field coverage | Excellent | Excellent | Equal |
| MP3 tag mapping | ✓ | ✓ | Equal |
| FLAC tag mapping | ✓ | ✓ | Equal |
| MP4 tag mapping | ✓ | ✓ | Equal |
| Vorbis tag mapping | ✓ | ✓ | Equal |
| Opus R128 | ✓ | ✓ | Equal |
| Custom fields | ✓ | ✓ | Equal |
| Sort fields | ✓ | ✓ | Equal |
| Work/Movement | MP4 only | MP4 only | Equal |
| Cover art | ✓ | ✓ | Equal |
| ReplayGain | ✓ | ✓ | Equal |
| Convert plugin | Basic | Basic | DBpoweramp wins (DSP) |
| Library management | Limited | Excellent | beets wins |
| CD ripping | ✓ | ✗ | DBpoweramp wins |
| Metadata lookup | Via PerfectMeta | Via MusicBrainz | Equal (different sources) |

---

## SECTION 5: MP3TAG ANALYSIS

### 5.1 Architecture Overview

**Mp3tag** is a Windows-only tag editor written in C++ using MFC. It uses TagLib for format handling.

**Website:** https://www.mp3tag.de/en/

### 5.2 Mp3tag Field Name Standardization

Mp3tag normalizes field names internally:

```cpp
// Mp3tag field name normalization (conceptual)
class CTag {
    static String NormalizeFieldName(String name) {
        // Remove special characters
        // Uppercase
        // Standardize aliases
    }
    
    // Field name mappings
    static const map<String, String> FieldAliases = {
        {"ALBUM ARTIST", "ALBUMARTIST"},
        {"ALBUMARTIST", "ALBUMARTIST"},
        {"BAND", "ALBUMARTIST"},
        {"COMPOSER", "COMPOSER"},
        {"LYRICIST", "LYRICIST"},
        {"CONDUCTOR", "CONDUCTOR"},
        {"PUBLISHER", "PUBLISHER"},
        {"LABEL", "LABEL"},
        {"ENCODEDBY", "ENCODER"},
        {"COPYRIGHT", "COPYRIGHT"},
    };
};
```

### 5.3 Mp3tag Cross-Format Tag Mapping

**Strengths:**
1. **Consistent field names**: Normalizes across all formats
2. **Extended tag support**: Can edit any TXXX frame in MP3
3. **Batch operations**: Tag many files at once

**Weaknesses:**
1. **Windows only**: No cross-platform support
2. **No conversion**: Can't change audio format
3. **Proprietary format handling**: Some quirky behaviors

### 5.4 Mp3tag vs DBpoweramp Field Handling

| Field | Mp3tag | DBpoweramp | Notes |
|-------|--------|------------|-------|
| ALBUMARTIST | ✓ | ✓ | Both normalize |
| COMPOSER | ✓ | ✓ | |
| BAND | ✓ (→ALBUMARTIST) | ✓ | Mp3tag alias |
| CONDUCTOR | ✓ | ✓ | |
| TRACKTOTAL | ✓ (via TRACK) | ✓ | Both parse "5/12" |
| DISCTOTAL | ✓ (via DISC) | ✓ | Both parse "1/3" |
| WORK | MP4 only | MP4 only | |
| MOVEMENT | MP4 only | MP4 only | |
| REPLAYGAIN | ✓ | ✓ | |
| R128 | Not native | ✓ | Requires manual |
| Cover art | ✓ | ✓ | |

---

## SECTION 6: TAGLIB ANALYSIS

### 6.1 Architecture Overview

**TagLib** is a C++ library for reading and writing audio metadata. It powers many of the tools analyzed above.

**Repository:** https://github.com/taglib/taglib

**Key classes:**
```cpp
// TagLib class hierarchy
class File {
    virtual Tag* tag() = 0;
    virtual AudioProperties* audioProperties() = 0;
};

class Tag {
    String title() const;
    void setTitle(const String&);
    // ... basic fields
    
    PropertyMap properties() const;
    void setProperties(const PropertyMap&);
};

class AudioFile : public File {
    Tag* tag() override;
};

class MPEG::File : public AudioFile { /* ... */ };
class FLAC::File : public AudioFile { /* ... */ };
class MP4::File : public AudioFile { /* ... */ };
class Vorbis::File : public AudioFile { /* ... */ };
```

### 6.2 TagLib Format Coverage

| Format | Reading | Writing | Notes |
|--------|---------|---------|-------|
| MP3 (ID3v1/v2) | ✓ | ✓ | Complete |
| FLAC (Vorbis) | ✓ | ✓ | Complete |
| Ogg Vorbis | ✓ | ✓ | Complete |
| Ogg Opus | ✓ | ✓ | Complete |
| MP4/M4A | ✓ | ✓ | Complete |
| WAV | ✓ | ✓ | RIFF INFO + ID3v2 |
| AIFF | ✓ | ✓ | ID3v2 in COMM chunk |
| WMA/ASF | ✓ | ✓ | |
| APEv2 | ✓ | ✓ | |
| WavPack | ✓ | ✓ | |
| Musepack | ✓ | ✓ | |
| Monkeys Audio | ✓ | ✓ | |
| OptimFROG | ✓ | ✓ | |
| Shorten | ✓ | ✗ | Read-only |
| TAAC | ✓ | ✓ | |
| DSF (DSD) | ✓ | ✓ | |
| DFF (DSD) | ✓ | ✓ | |
| CAF | ✓ | ✓ | |

### 6.3 TagLib Tag Reading Quirks

**Issue #162: Multi-value tag inconsistency**

```cpp
// Problem: TagLib::Tag generic getters are inconsistent
tag->artist();  // ID3v2.4: returns all joined by space
                // Vorbis: returns first only

// Solution: Use properties() and setProperties()
PropertyMap props = tag->properties();
StringList artists = props["ARTIST"];  // Always returns all values
```

**Historical behavior:**
- TagLib 1.7-1.8: Inconsistent multi-value handling
- TagLib 1.8+: Fixed to use " / " as separator for generic getters
- TagLib 2.x: Uses StringList consistently

### 6.4 TagLib Field Mapping Per Format

**ID3v2 (MP3):**

| Property Key | Frame | Notes |
|-------------|-------|-------|
| TITLE | TIT2 | |
| ARTIST | TPE1 | |
| ALBUM | TALB | |
| ALBUMARTIST | TPE2 | |
| COMPOSER | TCOM | |
| CONDUCTOR | TPE3 | |
| LYRICIST | TEXT | |
| RECORDINGDATE | TDRC | ID3v2.4 only |
| YEAR | TYER | ID3v2.3 |
| GENRE | TCON | Resolves numbers |
| TRACKNUMBER | TRCK | |
| TRACKTOTAL | TPOS | Shared with disc |
| DISCNUMBER | TPOS | |
| DISCTOTAL | TPOS | |
| COMMENT | COMM | |
| LYRICS | USLT | |
| BPM | TBPM | |
| ENCODER | TSSE | |
| ISRC | TSRC | |
| COPYRIGHT | TCOP | |
| ALBUMSORT | TSOA | ID3v2.4 |
| ARTISTSORT | TSOP | ID3v2.4 |
| TITLESORT | TSOT | ID3v2.4 |
| COMPILATION | TCMP | |
| REPLAYGAIN_TRACK_GAIN | TXXX:REPLAYGAIN_TRACK_GAIN | |
| REPLAYGAIN_TRACK_PEAK | TXXX:REPLAYGAIN_TRACK_PEAK | |

**Vorbis Comment (FLAC, OGG, Opus):**

| Property Key | Vorbis Comment | Notes |
|--------------|----------------|-------|
| TITLE | TITLE | |
| ARTIST | ARTIST | |
| ALBUM | ALBUM | |
| ALBUMARTIST | ALBUMARTIST | |
| COMPOSER | COMPOSER | |
| DATE | DATE | ISO 8601 |
| GENRE | GENRE | |
| TRACKNUMBER | TRACKNUMBER | May include total |
| TOTALTRACKS | TOTALTRACKS | |
| DISCNUMBER | DISCNUMBER | May include total |
| TOTALDISCS | TOTALDISCS | |
| REPLAYGAIN_TRACK_GAIN | REPLAYGAIN_TRACK_GAIN | dB string |
| REPLAYGAIN_TRACK_PEAK | REPLAYGAIN_TRACK_PEAK | Float |

**MP4 (M4A):**

| Property Key | MP4 Atom | Notes |
|--------------|----------|-------|
| TITLE | ©nam | |
| ARTIST | ©ART | |
| ALBUM | ©alb | |
| ALBUMARTIST | aART | |
| COMPOSER | ©wrt | |
| DATE | ©day | |
| GENRE | ©gen | |
| TRACKNUMBER | trkn | (n, m) tuple |
| DISCNUMBER | diskn | (n, m) tuple |
| COMMENT | ©cmt | |
| LYRICS | ©lyr | |
| BPM | tmpo | Int |
| ENCODER | ©enc | |
| COVER | covr | List of images |
| COMPILATION | cpil | Bool |
| WORK | ©wrk | |
| MOVEMENT | ©mvn | |
| MOVEMENTNUMBER | ©mvi | Int |
| MOVEMENTTOTAL | ©mvc | Int |
| REPLAYGAIN | ----:com.apple.iTunes:REPLAYGAIN_* | |

### 6.5 TagLib Changelog (Relevant to Tag Preservation)

| Version | Change | Impact |
|---------|--------|--------|
| 1.7 | Initial multi-format support | Baseline |
| 1.8 | Fixed multi-value inconsistency | Better cross-format |
| 1.11 | Fixed ID3v2.4 genre parsing | Genre numbers |
| 1.12 | Added ASF support | WMA coverage |
| 1.13 | Fixed ID3v2 unsynchronization | MP3 stability |
| 2.0 | Complete rewrite | Performance, API changes |
| 2.1-2.4 | Incremental improvements | Bug fixes |
| Current | Active development | |

---

## SECTION 7: MUTAGEN ANALYSIS

### 7.1 Architecture Overview

**mutagen** is a Python library for handling audio metadata. It powers beets and Picard.

**Repository:** https://github.com/quodlibet/mutagen

### 7.2 Mutagen Format Coverage

| Format | Mutagen Class | Features |
|--------|---------------|----------|
| MP3 | mutagen.mp3.MP3 | ID3v1/v2, Xing/Lame |
| FLAC | mutagen.flac.FLAC | Vorbis, METADATA_BLOCK_PICTURE |
| Ogg Vorbis | mutagen.oggvorbis.OggVorbis | Vorbis comments |
| Ogg Opus | mutagen.oggopus.OggOpus | Opus comments, R128 |
| MP4 | mutagen.mp4.MP4 | iTunes atoms, covr |
| WAV | mutagen.wave.WAVE | RIFF INFO, ID3v2 |
| AIFF | mutagen.aiff.AIFF | ID3v2 in COMM |
| WMA | mutagen.asf.ASF | ASF attributes |
| WavPack | mutagen.wvpk.WavPack | APEv2 tags |
| Musepack | mutagen.musepack.Musepack | APEv2 tags |
| Monkeys Audio | mutagen.monkeysaudio.MonkeysAudio | APEv2 tags |
| OptimFROG | mutagen.optimfrog.OptimFROG | APEv2 tags |
| APE | mutagen.apev2.APEv2 | Standalone APE |
| ID3 | mutagen.id3.ID3 | Standalone ID3 |

### 7.3 Mutagen Tag Access Patterns

```python
# Basic tag access (all formats)
from mutagen import File

f = File("song.mp3")
print(f.tags.title)  # Title frame value
print(f.tags.artist)

# Format-specific access
from mutagen.mp3 import MP3
from mutagen.flac import FLAC
from mutagen.mp4 import MP4

# MP3
mp3 = MP3("song.mp3")
print(mp3.tags["TIT2"].text[0])  # Title
print(mp3.tags["APIC:"].data)  # Cover art

# FLAC
flac = FLAC("song.flac")
print(flac["TITLE"])  # Vorbis comment
for pic in flac.pictures:
    print(pic.data)  # METADATA_BLOCK_PICTURE

# MP4
mp4 = MP4("song.m4a")
print(mp4["©nam"][0])  # Title
print(mp4["covr"])  # Cover art

# Opus (with R128)
opus = OggOpus("song.opus")
print(opus["R128_TRACK_GAIN"])  # Q7.8 int
print(opus["REPLAYGAIN_TRACK_GAIN"])  # dB string
```

### 7.4 Mutagen Cross-Format Behavior

**Multi-value handling:**
```python
# Reading multi-value
flac = FLAC("song.flac")
artists = flac["ARTIST"]  # Returns ['Artist1', 'Artist2']

# Writing multi-value
flac["ARTIST"] = ['Artist1', 'Artist2']  # Multiple ARTIST comments
flac.save()

# MP3 multi-value (via TXXX or separate frames)
mp3 = MP3("song.mp3")
mp3.tags.add(TPE1(encoding=3, text="Artist1"))
mp3.tags.add(TPE1(encoding=3, text="Artist2"))  # Multiple TPE1 frames
# Note: This is technically invalid ID3v2, but some tools accept it
```

**Cover art:**
```python
# FLAC - METADATA_BLOCK_PICTURE
flac = FLAC("song.flac")
for pic in flac.pictures:
    print(pic.type)  # Picture type
    print(pic.mime)  # MIME type
    print(pic.data)  # Image data

# MP3 - APIC
mp3 = MP3("song.mp3")
for frame in mp3.tags.getall('APIC'):
    print(frame.type)  # Picture type (0-20)
    print(frame.mime)
    print(frame.data)

# MP4 - covr
mp4 = MP4("song.m4a")
for img_data in mp4["covr"]:
    print(len(img_data))  # Raw image data
```

### 7.5 Mutagen vs DBpoweramp Tag Handling

| Feature | DBpoweramp | Mutagen | Notes |
|---------|------------|---------|-------|
| Basic fields | ✓ | ✓ | Equal |
| Multi-value | ✓ | ✓ | Equal |
| Track parsing | "5/12" → track+total | Same | Equal |
| Disc parsing | "1/3" → disc+total | Same | Equal |
| Genre numbers | Resolves to string | Resolves to string | Equal |
| Cover art | Front cover (3) | Depends on source | Depends |
| ReplayGain | ✓ | ✓ | Equal |
| R128 | ✓ | ✓ | Equal |
| Sort fields | ✓ | ✓ | Equal |
| Work/Movement | MP4 | MP4 | Equal |
| Custom fields | TXXX/MP4 freeform | Same | Equal |
| Atomic writes | ✓ | Manual | DBpoweramp wins |

---

## SECTION 8: FEATURE COMPARISON TABLE

### 8.1 Which Open-Source Tool Does Each DBpoweramp Feature Best?

| DBpoweramp Feature | Best Open-Source Tool | Second Best | Notes |
|-------------------|----------------------|-------------|-------|
| Cross-format conversion | **TagLib** | **mutagen** | Libraries provide the actual conversion |
| Tag field mapping | **beets mediafile.py** | **Picard** | Most complete field definitions |
| Cover art handling | **TagLib** | **mutagen** | Full picture type support |
| ReplayGain support | **beets** | **Picard** | Complete RG + R128 |
| CD ripping | **fre:ac** | None | Only real open-source option |
| Metadata lookup | **Picard** | **beets** | MusicBrainz integration |
| Batch processing | **fre:ac** | **beets** | Both handle large batches |
| Scripting/automation | **beets** | **Picard** | Python plugin ecosystem |
| Atomic writes | **fre:ac** | None | Explicitly handles this |
| Sort fields | **beets** | **Picard** | Complete mapping |
| Work/Movement | **beets** | **Picard** | MP4 support |
| Custom fields | **beets** | **mutagen** | Flexible storage |
| Genre handling | **beets** | **Picard** | Number resolution |
| Date handling | **beets** | **Picard** | Normalization |
| Multi-artist | **beets** | **Picard** | Separator handling |

### 8.2 Gaps Between DBpoweramp and Open-Source

| Gap | Description | Impact |
|-----|-------------|--------|
| **AccurateRip** | No open-source implementation | CD quality verification unavailable |
| **PerfectMeta** | No multi-source metadata blending | Must manually combine sources |
| **Shell integration** | Windows-only, proprietary | Limited to GUI use |
| **DSP pipeline** | Basic in fre:ac, none in others | No real-time effects during convert |
| **Gapless playback** | LAME/Xing tags | Partially supported |
| **HDCD detection** | Proprietary | Not available |
| **DSD processing** | Limited support | Basic decode only |
| **Professional resampling** | SSRC/ARDFTSRC | Limited in open-source |

### 8.3 Recommendations for DBpoweramp-equivalent Implementation

**For tag handling, use beets/mediafile.py as the primary reference** because:
1. Most complete field definitions
2. Best cross-format consistency
3. Well-documented Python implementation
4. Active development

**For conversion, use FFmpeg with TagLib/mutagen for tags** because:
1. FFmpeg handles all audio encoding/decoding
2. TagLib/mutagen handle tags correctly
3. Combining them gives complete control

**For CD ripping, use fre:ac** because:
1. Only viable open-source option
2. Reasonable tag handling
3. Cross-platform support

---

## SECTION 9: SOURCE CITATIONS

| Tool | Source |
|------|--------|
| fre:ac architecture | https://github.com/enzo1982/freac |
| beets mediafile.py | https://github.com/beetbox/mediafile |
| Picard tag mapping | https://github.com/metabrainz/picard |
| TagLib source | https://github.com/taglib/taglib |
| Mutagen source | https://github.com/quodlibet/mutagen |
| Mp3tag help | https://help.mp3tag.de/main_tags.html |
| Hydrogenaudio comparisons | https://hydrogenaudio.org |
| fre:ac review | https://appmus.com/vs/dbpoweramp-vs-bonkenc |

---

*Document: 72_Open_Source_Equivalent_Analysis.md*
*Generated from: Open-source tool analysis*
*Version: 1.0*
