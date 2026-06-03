# FFmpeg Metadata Mapping: All Containers — Deep Technical Reference

> **Category:** FFmpeg API
> **File Extensions:** (all containers)
> **MIME Types:** `audio/*`
> **Standardization Body:** FFmpeg Project / codec-specific bodies
> **Primary Specification:** https://ffmpeg.org/ffmpeg-formats.html, https://github.com/FFmpeg/FFmpeg/blob/master/doc/metadata.texi
> **Patent Status:** Per-codec
> **License:** FFmpeg: LGPL 2.1+; codec-specific licenses
> **Current Version:** Active development
> **Active Development:** Yes — rolling release

---

## 1. FFMPEG INTERNAL METADATA SYSTEM

### 1.1 Metadata Architecture Overview

FFmpeg uses a unified internal key-value system for metadata that is agnostic to the underlying container format. During demuxing, FFmpeg reads container-native metadata and maps it to FFmpeg's canonical keys. During muxing, FFmpeg writes its canonical keys back to the container's native format.

```
┌─────────────────────────────────────────────────────────────────┐
│              FFMPEG METADATA FLOW                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  CONTAINER FILE                                                   │
│       │                                                           │
│       ▼                                                           │
│  DEMUXER parses native tags                                       │
│       │                                                           │
│       ▼                                                           │
│  FFmpeg internal key-value dictionary                             │
│       │                                                           │
│       ▼                                                           │
│  CODEC/SUBTITLE STREAM TAGS                                       │
│       │                                                           │
│       ▼                                                           │
│  MUXER writes native tags to output container                      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 FFmpeg Standard Metadata Keys

FFmpeg defines a set of canonical keys that are used internally regardless of container format:

| Key | Description | Format | Multi-value |
|-----|-------------|--------|-------------|
| `title` | Track title | Text | No |
| `artist` | Track artist | Text | No |
| `album` | Album name | Text | No |
| `album_artist` | Album artist (distinct from track artist) | Text | No |
| `composer` | Composer | Text | No |
| `track` | Track number | "N" or "N/Total" | No |
| `disc` | Disc number | "N" or "N/Total" | No |
| `genre` | Genre | Text | No |
| `date` | Release date | YYYY or YYYY-MM-DD | No |
| `year` | Year (deprecated alias for date) | YYYY | No |
| `comment` | Comment | Text | No |
| `lyrics` | Lyrics | Text | No |
| `copyright` | Copyright notice | Text | No |
| `encoder` | Encoder software | Text | No |
| `encoded_by` | Encoder operator | Text | No |
| `filename` | Original filename | Text | No |
| `language` | ISO 639-1 language code | Text | No |
| `publisher` | Record label/publisher | Text | No |
| `BPM` | Beats per minute | Integer | No |
| `ISRC` | International Standard Recording Code | Text | No |
| `barcode` | EAN/UPC barcode | Text | No |
| `catalog_number` | Label catalog number | Text | No |
| `recordings` | MusicBrainz recordings | Text | No |
| `MusicBrainz Track Id` | MusicBrainz track UUID | Text | No |
| `MusicBrainz Artist Id` | MusicBrainz artist UUID | Text | No |
| `MusicBrainz Album Id` | MusicBrainz album UUID | Text | No |
| `MusicBrainz Album Artist Id` | MusicBrainz album artist UUID | Text | No |
| `MusicBrainz Release Group Id` | MusicBrainz release group UUID | Text | No |
| `MusicBrainz Work Id` | MusicBrainz work UUID | Text | No |
| `MusicBrainz TRM` | MusicBrainz TRM ID | Text | No |
| `REPLAYGAIN_TRACK_GAIN` | ReplayGain track gain | "+/-X.XX dB" | No |
| `REPLAYGAIN_TRACK_PEAK` | ReplayGain track peak | "0.XXXXXXX" | No |
| `REPLAYGAIN_ALBUM_GAIN` | ReplayGain album gain | "+/-X.XX dB" | No |
| `REPLAYGAIN_ALBUM_PEAK` | ReplayGain album peak | "0.XXXXXXX" | No |
| `REPLAYGAIN_ORIGINAL_GAIN` | Original gain for RG-compatible files | Text | No |
| `REPLAYGAIN_REFERENCE_LOUDNESS` | Reference loudness | "-XX.Y dB" | No |

---

## 2. CONTAINER-SPECIFIC METADATA SYSTEMS

### 2.1 MP4/iTunes Atoms

MP4/M4A containers use a hierarchical atom (box) structure for metadata. Tags are stored as `udta` → `meta` → `ilst` atoms.

#### MP4 Atom Structure for Metadata
```
moov
 └── udta
      ├── meta
      │    ├── hdlr (handler = "mdir" for metadata)
      │    └── ilst
      │         ├── ©nam   (title)
      │         ├── ©ART   (artist)
      │         ├── ©alb   (album)
      │         ├── aART   (album artist)
      │         ├── ©day   (date/year)
      │         ├── ©day   (date/year)
      │         ├── trkn   (track number)
      │         ├── disc   (disc number)
      │         ├── ©gen   (genre)
      │         ├── ©cmt   (comment)
      │         ├── ©lyr   (lyrics)
      │         ├── ©cprt  (copyright)
      │         ├── ----   (custom/user atom)
      │         ├── covr   (cover art — binary JPEG/PNG)
      │         └── [other atoms]
      └── (other user data)
```

#### Common MP4 Atoms and Their Meanings
| Atom | Meaning | FFmpeg Key |
|------|---------|------------|
| `©nam` | Title | `title` |
| `©ART` | Artist | `artist` |
| `©alb` | Album | `album` |
| `aART` | Album artist | `album_artist` |
| `©day` | Date | `date` |
| `©lyr` | Lyrics | `lyrics` |
| `©cmt` | Comment | `comment` |
| `©gen` | Genre | `genre` |
| `©ART` | Artist | `artist` |
| `trkn` | Track number | `track` |
| `disc` | Disc number | `disc` |
| `©wrt` | Composer | `composer` |
| `©cprt` | Copyright | `copyright` |
| `©lab` | Label | `publisher` |
| `tmpo` | BPM | `BPM` |
| `shani` | Sound copyright | copyright |
| `covr` | Cover art | (binary) |
| `----` | Freeform atom | (custom keys) |
| `cpil` | Compilation flag | `compilation` |
| `pgap` | Gapless playback flag | `gapless_playback` |
| `tmpo` | BPM | `BPM` |

### 2.2 OGG Vorbis Comments

OGG containers use the Xiph/Vorbis comment format (defined in the Vorbis I spec). These are also used by FLAC (within the STREAMINFO metadata block) and Opus.

#### Vorbis Comment Structure
```
┌────────────────────────────────────────────────────────────────┐
│              VORBIS COMMENT PACKET                              │
├────────────────────────────────────────────────────────────────┤
│ vendor_length (uint32 LE)                                       │
│ vendor_string (UTF-8)                                          │
│ comment_list_length (uint32 LE)                                │
│ comment_list[] {                                               │
│   comment_length (uint32 LE)                                   │
│   comment_string "KEY=value" (UTF-8)                          │
│ }                                                              │
│ framing_bit (1 bit = 1)                                        │
└────────────────────────────────────────────────────────────────┘
```

#### Standard Vorbis Comment Keys
| Vorbis Key | FFmpeg Key | Notes |
|------------|------------|-------|
| `TITLE` | `title` | |
| `ARTIST` | `artist` | |
| `ALBUM` | `album` | |
| `ALBUMARTIST` | `album_artist` | |
| `COMPOSER` | `composer` | |
| `TRACKNUMBER` | `track` | Often "N" or "N/M" |
| `TRACKTOTAL` | — | Often encoded in TRACKNUMBER as "N/M" |
| `DISCNUMBER` | `disc` | Often "N" or "N/M" |
| `DISCTOTAL` | — | Often encoded in DISCNUMBER as "N/M" |
| `DATE` | `date` | |
| `GENRE` | `genre` | |
| `COMMENT` | `comment` | |
| `LYRICS` | `lyrics` | |
| `COPYRIGHT` | `copyright` | |
| `ENCODER` | `encoder` | |
| `ISRC` | `ISRC` | |
| `BPM` | `BPM` | |
| `MUSICBRAINZ_TRACKID` | `MusicBrainz Track Id` | UUID |
| `MUSICBRAINZ_ARTISTID` | `MusicBrainz Artist Id` | UUID |
| `MUSICBRAINZ_ALBUMID` | `MusicBrainz Album Id` | UUID |
| `MUSICBRAINZ_ALBUMARTISTID` | `MusicBrainz Album Artist Id` | UUID |
| `MUSICBRAINZ_RELEASEGROUPID` | `MusicBrainz Release Group Id` | UUID |
| `REPLAYGAIN_TRACK_GAIN` | `REPLAYGAIN_TRACK_GAIN` | |
| `REPLAYGAIN_TRACK_PEAK` | `REPLAYGAIN_TRACK_PEAK` | |
| `REPLAYGAIN_ALBUM_GAIN` | `REPLAYGAIN_ALBUM_GAIN` | |
| `REPLAYGAIN_ALBUM_PEAK` | `REPLAYGAIN_ALBUM_PEAK` | |
| `METADATA_BLOCK_PICTURE` | cover art | Base64-encoded FLAC picture block |
| `COVERART` | cover art | Base64-encoded JPEG (legacy) |
| `CATALOG` | `catalog_number` | Label catalog number |
| `BARCODE` | `barcode` | EAN/UPC |
| `PUBLISHER` | `publisher` | Label name |
| `LABEL` | `publisher` | Label name |
| `LABELNO` | `catalog_number` | Label catalog number |

### 2.3 FLAC Metadata

FLAC uses Vorbis comments within the STREAMINFO block (type 4) of the metadata block chain.

#### FLAC Metadata Block Types
| Block Type | Description |
|------------|-------------|
| 0 | STREAMINFO (mandatory, first) |
| 1 | PADDING |
| 2 | APPLICATION |
| 3 | SEEKTABLE |
| 4 | VORBIS_COMMENT |
| 5 | CUESHEET |
| 6 | PICTURE |

#### FLAC PICTURE Block
```
┌────────────────────────────────────────────────────────────────┐
│              FLAC PICTURE BLOCK                                  │
├────────────────────────────────────────────────────────────────┤
│ picture_type (uint32)    // 0=Other, 3=Front cover, etc.        │
│ mime_length (uint32)    // MIME string length                  │
│ mime_type (bytes)       // "image/jpeg" or "image/png"         │
│ description_length (uint32)                                     │
│ description (bytes)     // UTF-8 description                   │
│ width (uint32)          // Image width in pixels                │
│ height (uint32)         // Image height in pixels               │
│ depth (uint32)          // Color depth in bits                  │
│ colors (uint32)         // Number of colors (0=unspecified)    │
│ picture_data_length (uint32)                                     │
│ picture_data (bytes)    // Binary image data                   │
└────────────────────────────────────────────────────────────────┘
```

FLAC picture types:
| Value | Meaning |
|-------|---------|
| 0 | Other |
| 1 | 32×32 file icon |
| 2 | Other file icon |
| 3 | Front cover |
| 4 | Back cover |
| 5 | Leaflet page |
| 6 | Media (CD, vinyl, etc.) |
| 7 | Lead artist |
| 8 | Artist |
| 9 | Conductor |
| 10 | Band/Orchestra |
| 11 | Composer |
| 12 | Lyricist |
| 13 | Recording location |
| 14 | During recording |
| 15 | During performance |
| 16 | Movie/video screen capture |
| 17 | Bright colored fish |
| 18 | Illustration |
| 19 | Band/artist logotype |
| 20 | Publisher/studio logotype |

### 2.4 WAV RIFF INFO

WAV files use RIFF INFO chunks for metadata, following the INFO chunk specification:

| RIFF INFO ID | FFmpeg Key | Notes |
|--------------|------------|-------|
| `INAM` | `title` | |
| `IART` | `artist` | |
| `IPRD` | `album` | |
| `ICRD` | `date` | |
| `IGNR` | `genre` | |
| `ICMT` | `comment` | |
| `ITRK` | `track` | |
| `ISFT` | `encoder` | Software name |
| `ICOP` | `copyright` | |
| `IWRI` | `comment` | Writer |
| `ILNG` | `language` | |
| `IRTD` | `rating` | |
| `IKEY` | `keywords` | Keywords |
| `IWEB` | `publisher` | URL |
| `ISBJ` | `subject` | Subject |

### 2.5 AIFF Chunks

AIFF uses FORM-based chunks for metadata:

| Chunk ID | Meaning | FFmpeg Key |
|----------|---------|------------|
| `NAME` | Track name | `title` |
| `AUTH` | Artist/author | `artist` |
| `(c)` | Copyright | `copyright` |
| `ANNO` | Annotation | `comment` |
| `ID3` | ID3v2 tag (at end of file) | (standard ID3v2) |

Note: Standard AIFF chunks are limited. For full metadata support (including cover art, MusicBrainz IDs, etc.), AIFF-C files can contain an `ID3` chunk at the end of the file with ID3v2 tags.

### 2.6 Matroska EBML Tags

Matroska/WebM uses an EBML-based tag structure:

```
┌────────────────────────────────────────────────────────────────┐
│              MATROSKA TAG STRUCTURE                             │
├────────────────────────────────────────────────────────────────┤
│ Tag                                                           │
│ ├── Targets                                                   │
│ │    ├── TrackUID[]                                           │
│ │    ├── EditionUID[]                                         │
│ │    └── ChapterUID[]                                        │
│ └── SimpleTags[]                                              │
│      ├── TagName                                             │
│      ├── TagString                                           │
│      ├── TagLanguage                                         │
│      └── TagDefault                                          │
└────────────────────────────────────────────────────────────────┘
```

#### Matroska Tag Name Mapping
| Matroska Tag Name | FFmpeg Key | Notes |
|-------------------|------------|-------|
| `TITLE` | `title` | |
| `ARTIST` | `artist` | |
| `ALBUM` | `album` | |
| `ALBUMARTIST` | `album_artist` | |
| `COMPOSER` | `composer` | |
| `TRACKNUMBER` | `track` | |
| `PART_NUMBER` | `track` | |
| `DISCNUMBER` | `disc` | |
| `DATE_RELEASED` | `date` | |
| `YEAR` | `date` | |
| `GENRE` | `genre` | |
| `COMMENT` | `comment` | |
| `LYRICS` | `lyrics` | |
| `COPYRIGHT` | `copyright` | |
| `ENCODER` | `encoder` | |
| `ISRC` | `ISRC` | |
| `BPM` | `BPM` | |
| `PUBLISHER` | `publisher` | |
| `LABEL` | `publisher` | |
| `CATALOGNUMBER` | `catalog_number` | |
| `REPLAYGAIN_TRACK_GAIN` | `REPLAYGAIN_TRACK_GAIN` | |
| `REPLAYGAIN_TRACK_PEAK` | `REPLAYGAIN_TRACK_PEAK` | |
| `REPLAYGAIN_ALBUM_GAIN` | `REPLAYGAIN_ALBUM_GAIN` | |
| `REPLAYGAIN_ALBUM_PEAK` | `REPLAYGAIN_ALBUM_PEAK` | |
| `COVERART` | cover art | Base64 |
| `ATTACHED_PICTURE` | cover art | Binary |

### 2.7 ASF/WMA Metadata

ASF (Advanced Systems Format) uses a binary attribute system:

| ASF Attribute | Meaning | FFmpeg Key |
|--------------|---------|------------|
| `Title` | Track title | `title` |
| `Author` | Artist | `artist` |
| `WM/AlbumTitle` | Album | `album` |
| `WM/AlbumArtist` | Album artist | `album_artist` |
| `WM/Year` | Year | `date` |
| `WM/Genre` | Genre | `genre` |
| `WM/TrackNumber` | Track number | `track` |
| `WM/PartOfSet` | Disc number | `disc` |
| `WM/Composer` | Composer | `composer` |
| `WM/Writer` | Writer/lyricist | `lyrics` |
| `WM/Copyright` | Copyright | `copyright` |
| `WM/Publisher` | Publisher | `publisher` |
| `WM/Comment` | Comment | `comment` |
| `WM/EncodedBy` | Encoded by | `encoder` |
| `WM/Encoder` | Encoder | `encoder` |
| `WM/MediaClassPrimaryID` | MusicBrainz Release Group ID | UUID |
| `WM/MusicBrainzTrackID` | MusicBrainz Track ID | UUID |
| `WM/MusicBrainzAlbumID` | MusicBrainz Album ID | UUID |
| `WM/MusicBrainzArtistIDs` | MusicBrainz Artist ID | UUID |
| `WM/Picture` | Cover art | Binary |
| `WM/ReplayGain_Track_Gain` | ReplayGain track gain | |
| `WM/ReplayGain_Track_Peak` | ReplayGain track peak | |

---

## 3. PER-CONTAINER MAPPING TABLES

### 3.1 Complete Mapping: MP4/M4A

| Standard Field | FFmpeg Key | MP4 Native Atom | Notes |
|---------------|------------|-----------------|-------|
| Title | `title` | `©nam` | UTF-8 encoded |
| Artist | `artist` | `©ART` | |
| Album | `album` | `©alb` | |
| Album Artist | `album_artist` | `aART` | |
| Composer | `composer` | `©wrt` | |
| Genre | `genre` | `©gen` | ID3 genre or freeform |
| Year/Date | `date` | `©day` | YYYY or YYYY-MM-DD |
| Track Number | `track` | `trkn` | uint16[2]: N, total |
| Disc Number | `disc` | `disk` | uint16[2]: N, total |
| Comment | `comment` | `©cmt` | |
| Lyrics | `lyrics` | `©lyr` | |
| Copyright | `copyright` | `©cprt` | |
| Encoder | `encoder` | `©too` | Auto-set by encoder |
| BPM | `BPM` | `tmpo` | uint16 |
| Compilation | `compilation` | `cpil` | uint8 (0/1) |
| Encoder Settings | `encoder` | `©enc` | |
| ISRC | `ISRC` | `©isr` | |
| Catalog Number | `catalog_number` | `cat#` | Freeform |
| Label | `publisher` | `©lab` | |
| Barcode | `barcode` | `ubik` [NEEDS VERIFICATION] | |
| MusicBrainz Track ID | `MusicBrainz Track Id` | `----:com.apple.iTunes:MUSICBRAINZ_TRACKID` | UUID |
| MusicBrainz Artist ID | `MusicBrainz Artist Id` | `----:com.apple.iTunes:MUSICBRAINZ_ARTISTID` | UUID |
| MusicBrainz Album ID | `MusicBrainz Album Id` | `----:com.apple.iTunes:MUSICBRAINZ_ALBUMID` | UUID |
| MusicBrainz Release Group ID | `MusicBrainz Release Group Id` | `----:com.apple.iTunes:MUSICBRAINZ_RELEASEGROUPID` | UUID |
| ReplayGain Track Gain | `REPLAYGAIN_TRACK_GAIN` | `----:org.i.sevy.RG_TRACK_GAIN` [NEEDS VERIFICATION] | Format: "-6.20 dB" |
| ReplayGain Track Peak | `REPLAYGAIN_TRACK_PEAK` | `----:org.i.sevy.RG_TRACK_PEAK` [NEEDS VERIFICATION] | Format: "0.998459" |
| ReplayGain Album Gain | `REPLAYGAIN_ALBUM_GAIN` | `----:org.i.sevy.RG_ALBUM_GAIN` [NEEDS VERIFICATION] | |
| ReplayGain Album Peak | `REPLAYGAIN_ALBUM_PEAK` | `----:org.i.sevy.RG_ALBUM_PEAK` [NEEDS VERIFICATION] | |
| Cover Art | — | `covr` | Binary JPEG/PNG |
| Sort Album | `sort_album` | `soal` | |
| Sort Artist | `sort_artist` | `soar` | |
| Sort Title | `sort_title` | `sonm` | |
| Sort Album Artist | `sort_album_artist` | `soaa` | |
| Sort Composer | `sort_composer` | `soco` | |

### 3.2 Complete Mapping: OGG (Vorbis/Opus)

| Standard Field | FFmpeg Key | Vorbis Comment Key | Notes |
|---------------|------------|-------------------|-------|
| Title | `title` | `TITLE` | |
| Artist | `artist` | `ARTIST` | |
| Album | `album` | `ALBUM` | |
| Album Artist | `album_artist` | `ALBUMARTIST` | |
| Composer | `composer` | `COMPOSER` | |
| Genre | `genre` | `GENRE` | |
| Year/Date | `date` | `DATE` | |
| Track Number | `track` | `TRACKNUMBER` | Often "N" or "N/M" |
| Total Tracks | — | `TRACKTOTAL` / `TOTALTRACKS` | Separate field |
| Disc Number | `disc` | `DISCNUMBER` | Often "N" or "N/M" |
| Total Discs | — | `DISCTOTAL` / `TOTALDISCS` | Separate field |
| Comment | `comment` | `COMMENT` | |
| Lyrics | `lyrics` | `LYRICS` | |
| Copyright | `copyright` | `COPYRIGHT` | |
| Encoder | `encoder` | `ENCODER` | |
| Encoder Settings | `encoder` | `ENCODER_OPTIONS` | |
| BPM | `BPM` | `BPM` | |
| ISRC | `ISRC` | `ISRC` | |
| Catalog Number | `catalog_number` | `CATALOGNUMBER` | |
| Label | `publisher` | `LABEL` / `PUBLISHER` | |
| Barcode | `barcode` | `BARCODE` | |
| MusicBrainz Track ID | `MusicBrainz Track Id` | `MUSICBRAINZ_TRACKID` | UUID |
| MusicBrainz Artist ID | `MusicBrainz Artist Id` | `MUSICBRAINZ_ARTISTID` | UUID |
| MusicBrainz Album ID | `MusicBrainz Album Id` | `MUSICBRAINZ_ALBUMID` | UUID |
| MusicBrainz Album Artist ID | `MusicBrainz Album Artist Id` | `MUSICBRAINZ_ALBUMARTISTID` | UUID |
| MusicBrainz Release Group ID | `MusicBrainz Release Group Id` | `MUSICBRAINZ_RELEASEGROUPID` | UUID |
| MusicBrainz Work ID | `MusicBrainz Work Id` | `MUSICBRAINZ_WORKID` | UUID |
| ReplayGain Track Gain | `REPLAYGAIN_TRACK_GAIN` | `REPLAYGAIN_TRACK_GAIN` | |
| ReplayGain Track Peak | `REPLAYGAIN_TRACK_PEAK` | `REPLAYGAIN_TRACK_PEAK` | |
| ReplayGain Album Gain | `REPLAYGAIN_ALBUM_GAIN` | `REPLAYGAIN_ALBUM_GAIN` | |
| ReplayGain Album Peak | `REPLAYGAIN_ALBUM_PEAK` | `REPLAYGAIN_ALBUM_PEAK` | |
| ReplayGain Reference Loudness | `REPLAYGAIN_REFERENCE_LOUDNESS` | `REPLAYGAIN_REFERENCE_LOUDNESS` | |
| Cover Art (Vorbis) | — | `METADATA_BLOCK_PICTURE` | Base64 FLAC picture |
| Cover Art (legacy) | — | `COVERART` | Base64 JPEG |
| Cover Art MIME | — | `COVERARTMIME` | "image/jpeg" or "image/png" |

### 3.3 Complete Mapping: FLAC

| Standard Field | FFmpeg Key | Vorbis Comment Key | Notes |
|---------------|------------|-------------------|-------|
| Title | `title` | `TITLE` | |
| Artist | `artist` | `ARTIST` | |
| Album | `album` | `ALBUM` | |
| Album Artist | `album_artist` | `ALBUMARTIST` | |
| Composer | `composer` | `COMPOSER` | |
| Genre | `genre` | `GENRE` | |
| Year/Date | `date` | `DATE` | |
| Track Number | `track` | `TRACKNUMBER` | |
| Total Tracks | — | `TRACKTOTAL` / `TOTALTRACKS` | |
| Disc Number | `disc` | `DISCNUMBER` | |
| Total Discs | — | `DISCTOTAL` / `TOTALDISCS` | |
| Comment | `comment` | `COMMENT` | |
| Lyrics | `lyrics` | `LYRICS` | |
| Copyright | `copyright` | `COPYRIGHT` | |
| Encoder | `encoder` | `ENCODER` | |
| BPM | `BPM` | `BPM` | |
| ISRC | `ISRC` | `ISRC` | |
| MusicBrainz Track ID | `MusicBrainz Track Id` | `MUSICBRAINZ_TRACKID` | UUID |
| MusicBrainz Artist ID | `MusicBrainz Artist Id` | `MUSICBRAINZ_ARTISTID` | UUID |
| MusicBrainz Album ID | `MusicBrainz Album Id` | `MUSICBRAINZ_ALBUMID` | UUID |
| ReplayGain Track Gain | `REPLAYGAIN_TRACK_GAIN` | `REPLAYGAIN_TRACK_GAIN` | |
| ReplayGain Track Peak | `REPLAYGAIN_TRACK_PEAK` | `REPLAYGAIN_TRACK_PEAK` | |
| Cover Art | — | (PICTURE metadata block) | Binary image |

Note: FLAC cover art is stored in a dedicated PICTURE metadata block, not in Vorbis comments. The `METADATA_BLOCK_PICTURE` Vorbis comment is used for embedding FLAC pictures in OGG containers.

### 3.4 Complete Mapping: WAV/RIFF

| Standard Field | FFmpeg Key | RIFF INFO Chunk | Notes |
|---------------|------------|-----------------|-------|
| Title | `title` | `INAM` | |
| Artist | `artist` | `IART` | |
| Album | `album` | `IPRD` | |
| Year/Date | `date` | `ICRD` | |
| Genre | `genre` | `IGNR` | |
| Comment | `comment` | `ICMT` | |
| Track Number | `track` | `ITRK` | |
| Encoder | `encoder` | `ISFT` | |
| Copyright | `copyright` | `ICOP` | |
| Writer | `comment` | `IWRI` | |
| Subject | `comment` | `ISBJ` | |
| Encoder URL | `publisher` | `IWEB` | |
| Label URL | `publisher` | `IWEB` | |

Note: WAV/RIFF INFO is very limited. It does not support: album artist, disc number, lyrics, cover art, MusicBrainz IDs, ReplayGain, or multi-value fields.

### 3.5 Complete Mapping: AIFF

| Standard Field | FFmpeg Key | AIFF Chunk | Notes |
|---------------|------------|------------|-------|
| Title | `title` | `NAME` | |
| Artist | `artist` | `AUTH` | |
| Copyright | `copyright` | `(c)` | |
| Comment | `comment` | `ANNO` | |
| Encoder | `encoder` | `ID3` (ID3v2 in ID3 chunk) | |

Note: Standard AIFF chunks are extremely limited. Full metadata support requires an `ID3` chunk at the end of the file containing ID3v2 tags.

### 3.6 Complete Mapping: Matroska/MKA

| Standard Field | FFmpeg Key | Matroska Tag Name | Notes |
|---------------|------------|-------------------|-------|
| Title | `title` | `TITLE` | |
| Artist | `artist` | `ARTIST` | |
| Album | `album` | `ALBUM` | |
| Album Artist | `album_artist` | `ALBUMARTIST` | |
| Composer | `composer` | `COMPOSER` | |
| Genre | `genre` | `GENRE` | |
| Year/Date | `date` | `DATE` | |
| Track Number | `track` | `TRACKNUMBER` | |
| Disc Number | `disc` | `DISCNUMBER` | |
| Comment | `comment` | `COMMENT` | |
| Lyrics | `lyrics` | `LYRICS` | |
| Copyright | `copyright` | `COPYRIGHT` | |
| Encoder | `encoder` | `ENCODER` | |
| BPM | `BPM` | `BPM` | |
| ISRC | `ISRC` | `ISRC` | |
| Catalog Number | `catalog_number` | `CATALOGNUMBER` | |
| Publisher | `publisher` | `PUBLISHER` / `LABEL` | |
| ReplayGain Track Gain | `REPLAYGAIN_TRACK_GAIN` | `REPLAYGAIN_TRACK_GAIN` | |
| ReplayGain Track Peak | `REPLAYGAIN_TRACK_PEAK` | `REPLAYGAIN_TRACK_PEAK` | |
| Cover Art | — | `COVERART` / `ATTACHED_PICTURE` | Binary |
| MusicBrainz IDs | — | `MUSICBRAINZ_*` | Custom tags |

---

## 4. LOSSY METADATA ROUND-TRIP

### 4.1 What Survives Stream Copy (`-c:a copy`)

Stream copy is the most lossless metadata operation because it does not re-encode the audio stream. All container-level metadata survives stream copy.

| Container | Metadata Survives `-c:a copy`? | Notes |
|-----------|------------------------------|-------|
| MP4/M4A | Yes | All atoms survive |
| OGG | Yes | Vorbis comments survive |
| FLAC | Yes | Vorbis comments survive |
| WAV | Yes | RIFF INFO survives |
| AIFF | Yes | Standard chunks survive |
| Matroska | Yes | EBML tags survive |
| MP3 | Yes (mostly) | ID3 tags survive; iTunSMPB may be stripped |

### 4.2 What Survives Transcoding

When re-encoding (not stream copying), metadata is written by the muxer to the output container. Survival depends on:

1. **Whether the output container supports the tag**
2. **Whether FFmpeg maps the key correctly to the container**
3. **Whether the tag value format is compatible**

| Tag Type | MP4 | OGG | FLAC | WAV | Matroska |
|----------|-----|-----|------|-----|---------|
| Standard fields (title, artist, etc.) | Yes | Yes | Yes | Partial | Yes |
| Cover art | Yes | Yes (via METADATA_BLOCK_PICTURE) | Yes | No | Yes |
| MusicBrainz IDs | Yes (custom atoms) | Yes | Yes | No | Partial |
| ReplayGain | Partial [NEEDS VERIFICATION] | Yes | Yes | No | Yes |
| Sort fields | Yes | Yes | Yes | No | Yes |
| Custom tags | Yes (freeform) | Yes | Yes | No | Yes |
| BPM | Yes | Yes | Yes | No | Yes |
| ISRC | Yes | Yes | Yes | No | Yes |

### 4.3 Tags Lost During Transcoding

The following are commonly lost when transcoding between different containers:

| Tag | Lost When | Reason |
|-----|-----------|--------|
| iTunSMPB | Re-encode | Only valid for MP3 |
| Sort fields | → WAV | WAV doesn't support sort |
| Cover art (WAV) | → WAV | WAV RIFF INFO doesn't support images |
| MusicBrainz IDs | → WAV | WAV doesn't support UUID fields |
| ReplayGain | → WAV | WAV doesn't support replaygain tags |
| ID3v1 | Any transcoding | ID3v1 is stripped by most muxers |
| APEv2 | Any transcoding | APE tags are format-specific |

### 4.4 Preserving Metadata During Transcoding

#### Using `-map_metadata`
```bash
# Copy all metadata from input to output
ffmpeg -i input.flac -c:a libopus -b:a 128k \
  -map_metadata 0 \
  output.opus

# Copy only global metadata
ffmpeg -i input.flac -c:a libopus -b:a 128k \
  -map_metadata 0:g \
  output.opus

# Copy only audio stream metadata
ffmpeg -i input.flac -c:a libopus -b:a 128k \
  -map_metadata 0:a \
  output.opus

# Copy nothing
ffmpeg -i input.flac -c:a libopus -b:a 128k \
  -map_metadata -1 \
  output.opus
```

#### Using `-metadata` with Selective Override
```bash
# Copy all metadata, override specific fields
ffmpeg -i input.flac -c:a libopus -b:a 128k \
  -map_metadata 0 \
  -metadata title="New Title" \
  -metadata encoder="My Encoder" \
  output.opus
```

#### Using ffmetadata File
```bash
# Extract metadata to file
ffmpeg -i input.flac -f ffmetadata metadata.txt

# Edit metadata.txt (manual editing)

# Re-apply metadata during transcode
ffmpeg -i input.flac -i metadata.txt \
  -c:a libopus -b:a 128k \
  -map_metadata 1 \
  output.opus
```

---

## 5. COVER ART MAPPING

### 5.1 Cover Art Storage Across Containers

| Container | Storage Method | Format | FFmpeg Method |
|-----------|---------------|--------|---------------|
| MP4/M4A | `covr` atom | JPEG, PNG, BMP | `-attach` or `-disposition` |
| OGG | `METADATA_BLOCK_PICTURE` | JPEG, PNG | Via FFmpeg's cover art handling |
| FLAC | PICTURE metadata block | JPEG, PNG | Via FFmpeg's cover art handling |
| WAV | None (RIFF INFO) | N/A | Not supported |
| AIFF | None (standard) / ID3 | JPEG, PNG | Via ID3 chunk |
| Matroska | `ATTACHED_PICTURE` | JPEG, PNG | `-attach` |
| MP3 | APIC frame | JPEG, PNG | `-attach` |

### 5.2 FFmpeg Cover Art Commands

#### Embedding Cover Art
```bash
# MP4: embed cover from file
ffmpeg -i input.m4a -i cover.jpg \
  -c:a copy \
  -map 0:a -map 1:v \
  -disposition:v attached_pic \
  output_with_cover.m4a

# OGG: embed cover
ffmpeg -i input.ogg -i cover.jpg \
  -c:a copy \
  -map 0:a -map 1:v \
  -metadata:s:v title="Cover" \
  -metadata:s:v comment="Cover (front)" \
  output_with_cover.ogg

# Matroska: embed cover
ffmpeg -i input.mka -i cover.jpg \
  -c:a copy \
  -map 0:a -map 1:v \
  -metadata:s:v title="Cover" \
  output_with_cover.mka

# Strip cover art
ffmpeg -i input.m4a -c:a copy -disposition:v none output_no_cover.m4a
```

#### Extracting Cover Art
```bash
# Extract first cover art from MP4
ffmpeg -i input.m4a -an -vcodec copy cover.jpg

# Extract from OGG
ffmpeg -i input.ogg -an -vcodec copy cover.jpg

# Extract from FLAC
ffmpeg -i input.flac -an -vcodec copy cover.png

# Extract from MP3
ffmpeg -i input.mp3 -an -vcodec copy cover.jpg
```

### 5.3 Converting Cover Art Between Formats
```bash
# MP4 cover (extract + re-embed as PNG)
ffmpeg -i input.m4a -an -vcodec copy cover.jpg
ffmpeg -i input.m4a -i cover.jpg \
  -c:a copy \
  -map 0:a -map 1:v \
  -disposition:v attached_pic \
  output.png_cover.m4a

# OGG cover (METADATA_BLOCK_PICTURE conversion)
# FFmpeg handles this automatically when embedding
ffmpeg -i input.flac -i cover.png \
  -c:a copy \
  -map 0:a -map 1:v \
  output_with_png.ogg
```

---

## 6. REPLAYGAIN TAG MAPPING

### 6.1 ReplayGain Standard Format

ReplayGain stores loudness normalization data in specific formats:

| Tag | Format | Example |
|-----|--------|---------|
| `REPLAYGAIN_TRACK_GAIN` | `"+/-X.XX dB"` | `"-6.20 dB"` |
| `REPLAYGAIN_TRACK_PEAK` | `"0.XXXXXXX"` | `"0.998459"` |
| `REPLAYGAIN_ALBUM_GAIN` | `"+/-X.XX dB"` | `"-5.80 dB"` |
| `REPLAYGAIN_ALBUM_PEAK` | `"0.XXXXXXX"` | `"0.998459"` |
| `REPLAYGAIN_REFERENCE_LOUDNESS` | `"-XX.Y dB"` | `"-18 dB"` |

### 6.2 ReplayGain Across Containers

| Container | Tag Format | FFmpeg Support |
|-----------|-----------|---------------|
| MP4/M4A | Custom freeform atoms | Partial [NEEDS VERIFICATION] |
| OGG | Vorbis comments | Full |
| FLAC | Vorbis comments | Full |
| WAV | Not supported | None |
| AIFF | Not supported | None |
| Matroska | EBML tags | Full |
| MP3 | ID3v2 RVA2 frame / TXXX | Full |

### 6.3 FFmpeg ReplayGain Commands

```bash
# Copy ReplayGain through transcode
ffmpeg -i input.flac -c:a libopus -b:a 128k \
  -map_metadata 0 \
  output.opus

# Scan and apply ReplayGain (requires external tool like `audiotools` or `r128gain`)
# Example with r128gain (not native FFmpeg):
r128gain input.flac

# Scan using ffmpeg (for analysis only, not writing)
ffmpeg -i input.flac -af "ebur128=metadata=1" -f null -

# Manual ReplayGain writing
ffmpeg -i input.flac -c:a copy \
  -metadata REPLAYGAIN_TRACK_GAIN="-6.20 dB" \
  -metadata REPLAYGAIN_TRACK_PEAK="0.998459" \
  output_with_rg.flac
```

---

## 7. MUSICBRAINZ ID MAPPING

### 7.1 MusicBrainz IDs in FFmpeg

MusicBrainz IDs are stored as UUID strings in tags. FFmpeg uses the exact field names:

| Field | FFmpeg Key | Format |
|-------|------------|--------|
| Track ID | `MusicBrainz Track Id` | UUID |
| Artist ID | `MusicBrainz Artist Id` | UUID |
| Album ID | `MusicBrainz Album Id` | UUID |
| Album Artist ID | `MusicBrainz Album Artist Id` | UUID |
| Release Group ID | `MusicBrainz Release Group Id` | UUID |
| Work ID | `MusicBrainz Work Id` | UUID |
| TRM ID | `MusicBrainz TRM` | UUID (legacy) |
| Disc ID | `MusicBrainz Disc ID` | String (CD TOC) |
| AR ID | `MusicBrainz AR ID` | UUID |

### 7.2 MusicBrainz ID Preservation

| Container | MusicBrainz ID Support | FFmpeg Support |
|-----------|----------------------|---------------|
| MP4/M4A | Custom freeform atoms (`----`) | Partial |
| OGG/Vorbis | Native `MUSICBRAINZ_*` | Full |
| FLAC | Native `MUSICBRAINZ_*` | Full |
| WAV | Not supported | None |
| AIFF | Via ID3 | Full |
| Matroska | Custom tags | Partial |

### 7.3 Copying MusicBrainz IDs
```bash
# Copy all MusicBrainz IDs through transcode
ffmpeg -i input.flac -c:a libopus -b:a 128k \
  -map_metadata 0 \
  output.opus

# Manual MusicBrainz ID writing
ffmpeg -i input.flac -c:a copy \
  -metadata "MusicBrainz Track Id"="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" \
  -metadata "MusicBrainz Album Id"="yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy" \
  output_with_mb.flac
```

---

## 8. FFMPEG `-METADATA` FLAGS

### 8.1 Global vs Per-Stream Metadata

```bash
# Global metadata (applies to entire file)
ffmpeg -i input.wav -metadata title="Song Title" output.mp3

# Per-stream metadata (audio stream)
ffmpeg -i input.wav -metadata:s:a:0 title="Song Title" output.mp3

# Per-stream metadata (video stream — cover art)
ffmpeg -i input.wav -i cover.jpg \
  -c:a copy \
  -map 0:a -map 1:v \
  -metadata:s:v title="Cover" \
  output.m4a
```

### 8.2 Stream Specifiers for Metadata

| Specifier | Meaning |
|-----------|---------|
| `-metadata:g` | Global metadata |
| `-metadata:s:a` | Audio stream metadata |
| `-metadata:s:a:0` | First audio stream |
| `-metadata:s:a:1` | Second audio stream |
| `-metadata:s:v` | Video stream metadata |
| `-metadata:s:v:0` | First video stream |

### 8.3 Deleting Metadata
```bash
# Delete all metadata
ffmpeg -i input.wav -c:a copy -map_metadata -1 output.wav

# Delete specific metadata field
ffmpeg -i input.wav -c:a copy -metadata:s:a:0 title="" output.wav

# Delete all audio stream metadata, keep global
ffmpeg -i input.wav -c:a copy -map_metadata:s:a -1 output.wav
```

---

## 9. COMMON PITFALLS

### 9.1 Case Sensitivity

**Vorbis comments are case-sensitive:**

| Tag | Valid | Notes |
|-----|-------|-------|
| `TITLE` | Yes | Correct |
| `title` | No | Not recognized by most players |
| `Title` | No | Not recognized |
| `Artist` | Yes | Correct |
| `ARTIST` | Yes | Correct |

**MP4 atoms are NOT case-sensitive (they use 4-char codes):**

| Atom | Valid |
|------|-------|
| `©nam` | Yes (standard) |
| `©NAM` | Yes (also valid) |

### 9.2 Unicode Handling

| Container | Encoding | Notes |
|-----------|---------|-------|
| MP4 | UTF-8 or UTF-16 | FFmpeg uses UTF-8 |
| Vorbis | UTF-8 | Always UTF-8 |
| FLAC | UTF-8 | Always UTF-8 |
| WAV | Latin-1 | Limited to ISO-8859-1 |
| AIFF | ASCII (ID3 in separate chunk) | Via ID3v2 |
| Matroska | UTF-8 | Full Unicode |

### 9.3 Multiple Values in Single Fields

Vorbis comments support multiple values by repeating the key:

```
TITLE=Track 1
TITLE=Live Version
ARTIST=Artist Name
```

MP4 does not natively support multiple values per field. FFmpeg may encode multiple values as a single string with separators.

### 9.4 Unknown/Custom Tags

| Container | Custom Tag Support | Method |
|-----------|------------------|--------|
| MP4 | Yes | Freeform `----` atoms with namespace |
| Vorbis | Yes | Any key not in spec |
| FLAC | Yes | Any key not in spec |
| WAV | No | Not supported |
| Matroska | Yes | Custom tag names |

### 9.5 FFmpeg Metadata Pitfalls

| Pitfall | Symptom | Solution |
|---------|---------|----------|
| Tag case mismatch | Tags lost on read | Use uppercase for Vorbis |
| UTF-16 in MP4 | Garbled text | Use UTF-8 encoding |
| Multi-value handling | Tags concatenated | Check FFmpeg version |
| Stream copy stripping | Metadata lost | Don't use `-c:a copy` + metadata override; use `-map_metadata` |
| Re-encoding metadata loss | Custom atoms lost | Use stream copy or preserve via `-map_metadata` |
| Non-ASCII filenames | Encoding issues | Use `-metadata:s:a:0 title=""` with proper quoting |

---

## 10. CODEC-SPECIFIC METADATA CONSIDERATIONS

### 10.1 MP3

MP3 files can contain multiple tag systems simultaneously:

| Tag System | Location | FFmpeg Behavior |
|------------|---------|-----------------|
| ID3v2.4 | Beginning of file | Read/write by default |
| ID3v2.3 | Beginning of file | Read/write |
| ID3v1 | End of file | Read/write (may need `-id3v2_version 3`) |
| APEv2 | End of file | Read/write |

FFmpeg's read priority for MP3: ID3v2 > APEv2 > ID3v1

FFmpeg's write behavior: Writes ID3v2.4 at start + ID3v1 at end (if enabled)

```bash
# Write ID3v2.4 (default)
ffmpeg -i input.wav -id3v2_version 4 output.mp3

# Write ID3v2.3 (maximum compatibility)
ffmpeg -i input.wav -id3v2_version 3 output.mp3

# Disable ID3v2
ffmpeg -i input.wav -id3v2_version 0 output.mp3

# Disable ID3v1
ffmpeg -i input.wav -write_id3v1 0 output.mp3
```

### 10.2 AAC/MP4

AAC in MP4 containers have all MP4 atom capabilities. Additional considerations:

| Feature | MP4 Support | FFmpeg Command |
|---------|------------|---------------|
| Sort fields | Yes | `-metadata sort_album="Album Name"` |
| MusicBrainz IDs | Partial (custom atoms) | `-metadata "----:com.apple.iTunes:MUSICBRAINZ_TRACKID=..."` |
| Gapless info | Yes (built-in) | `-movflags +gaplessinfo` |
| Track gain | Custom | Via freeform atoms |
| iTunes metadata | Yes | Standard atoms |

### 10.3 FLAC

FLAC uses Vorbis comments for text metadata and PICTURE blocks for images:

| Feature | FLAC Support | Notes |
|---------|-------------|-------|
| Vorbis comments | Yes | Standard |
| Cover art | Yes | PICTURE metadata block (not comment) |
| MusicBrainz IDs | Yes | `MUSICBRAINZ_*` keys |
| ReplayGain | Yes | `REPLAYGAIN_*` keys |
| CUE sheets | Yes | Embedded CUE block |
| Seek tables | Yes | SEEKTABLE block |

### 10.4 Opus

Opus in OGG containers uses Vorbis comments. Special considerations:

| Feature | Opus Support | Notes |
|---------|-------------|-------|
| Pre-skip | Yes | In OpusHead header, not metadata |
| Input sample rate | Yes | In OpusHead header |
| Gain | Yes | Via replaygain or container tags |
| Channel mapping | Yes | In OpusHead header |

### 10.5 WAV

WAV/RIFF INFO is extremely limited. For full metadata in WAV files, use Broadcast Wave Format (BWF):

| Extension | Metadata Support | FFmpeg Behavior |
|-----------|----------------|-----------------|
| `.wav` | RIFF INFO only | Very limited |
| `.bwf` | BWF extensions | Supports more fields |

```bash
# Create BWF with BWF metadata
ffmpeg -i input.wav -metadata title="Title" \
  -metadata artist="Artist" \
  -metadata recording_year="2024" \
  -metadata origin_site="Studio" \
  output.bwf
```

---

## 11. PRACTICAL COMMANDS

### 11.1 Copy All Metadata
```bash
ffmpeg -i input.flac -c:a libopus -b:a 128k -map_metadata 0 output.opus
```

### 11.2 Copy Metadata, Strip Cover Art
```bash
ffmpeg -i input.flac -c:a libopus -b:a 128k \
  -map_metadata 0 -map 0:a \
  output.opus
```

### 11.3 Add/Override Specific Fields
```bash
ffmpeg -i input.flac -c:a libopus -b:a 128k \
  -map_metadata 0 \
  -metadata title="New Title" \
  -metadata genre="Electronic" \
  output.opus
```

### 11.4 Copy All Metadata to MP4
```bash
ffmpeg -i input.flac -c:a libfdk_aac -vbr 4 \
  -movflags +faststart \
  -map_metadata 0 \
  output.m4a
```

### 11.5 Export Metadata to File
```bash
ffmpeg -i input.flac -f ffmetadata metadata.txt
# Edit metadata.txt as needed
ffmpeg -i input.flac -i metadata.txt -c:a libopus -b:a 128k \
  -map_metadata 1 \
  output.opus
```

### 11.6 Strip All Metadata
```bash
ffmpeg -i input.flac -c:a flac -compression_level 8 \
  -map_metadata -1 \
  output_clean.flac
```

### 11.7 Merge Metadata from Multiple Sources
```bash
# Take audio from input1, metadata from input2
ffmpeg -i input1.flac -i input2.flac -c:a copy \
  -map 0:a -map_metadata 1 \
  output_merged.flac
```

### 11.8 Convert Metadata Between Containers
```bash
# FLAC (Vorbis comments) → MP4 (iTunes atoms)
ffmpeg -i input.flac -c:a alac \
  -map_metadata 0 \
  output.m4a

# MP3 (ID3) → FLAC (Vorbis comments)
ffmpeg -i input.mp3 -c:a flac -compression_level 8 \
  -map_metadata 0 \
  output.flac
```

---

## 12. IMPLEMENTATION CHECKLIST

### Metadata Reading
- [ ] Read all standard tag fields from container
- [ ] Read cover art as binary JPEG/PNG
- [ ] Read ReplayGain tags in correct format
- [ ] Read MusicBrainz UUIDs
- [ ] Read sort fields if present
- [ ] Handle multiple tag systems (ID3v1 + ID3v2 + APE for MP3)
- [ ] Handle case sensitivity (Vorbis comments are case-sensitive)

### Metadata Writing
- [ ] Write standard fields to container-native format
- [ ] Embed cover art in container-native format
- [ ] Write ReplayGain tags
- [ ] Write MusicBrainz IDs
- [ ] Write sort fields
- [ ] Handle multi-value fields
- [ ] Set correct character encoding per container

### Container-Specific
- [ ] MP4: Write iTunes atoms correctly (`©nam`, `©ART`, etc.)
- [ ] MP4: Write custom freeform atoms (`----`) for MusicBrainz IDs
- [ ] OGG/FLAC: Write Vorbis comments (uppercase keys)
- [ ] OGG/FLAC: Handle METADATA_BLOCK_PICTURE for cover art
- [ ] WAV: Write RIFF INFO chunks (limited field set)
- [ ] Matroska: Write EBML tags with correct names

### Lossless Round-trip
- [ ] Test: FLAC → MP3 → FLAC: all tags preserved
- [ ] Test: MP4 → OGG → MP4: all tags preserved
- [ ] Test: Cover art survives through all conversions
- [ ] Test: ReplayGain survives through all conversions
- [ ] Test: MusicBrainz IDs survive through all conversions

### Edge Cases
- [ ] Handle UTF-8 metadata in all containers
- [ ] Handle non-ASCII characters in tag values
- [ ] Handle very long tag values (per container limits)
- [ ] Handle unknown/custom tag fields
- [ ] Handle multiple cover art images (front, back, etc.)
- [ ] Handle missing optional fields (do not write empty tags)

---

## 13. FFMPEG METADATA IN PIPELINES

### 13.1 Reading Metadata with ffprobe

```bash
# JSON format (best for programmatic use)
ffprobe -v quiet -print_format json -show_format -show_streams input.flac | jq .

# Extract all tags
ffprobe -v quiet -show_entries format_tags -of default=noprint_wrappers=1 input.flac

# Extract specific tag
ffprobe -v quiet -show_entries format_tags=title,artist,album \
  -of default=noprint_wrappers=1:nokey=1 input.flac

# Extract cover art as binary
ffprobe -v quiet -show_entries stream=codec_type,codec_name \
  -of default=noprint_wrappers=1 input.m4a

# Extract chapter information
ffprobe -v quiet -show_chapters input.flac

# Extract all metadata including chapters and streams
ffprobe -v quiet -show_format -show_streams -show_chapters \
  -print_format json input.flac | jq .
```

### 13.2 Batch Metadata Extraction

```bash
#!/bin/bash
# extract_metadata.sh — Extract metadata from all audio files

for file in *.flac *.mp3 *.m4a *.ogg *.opus; do
    if [ -f "$file" ]; then
        echo "=== $file ==="
        ffprobe -v quiet -show_entries format_tags=title,artist,album,date,genre \
          -of default=noprint_wrappers=1:nokey=1 "$file"
        echo ""
    fi
done

# Using jq for JSON output
for file in *.flac; do
    if [ -f "$file" ]; then
        echo "$file:"
        ffprobe -v quiet -print_format json -show_format "$file" | \
          jq -r '.format.tags | "\(.title // "N/A") by \(.artist // "N/A")"'
    fi
done
```

### 13.3 Writing Metadata in Pipelines

```bash
# Read metadata from source, write to destination
SOURCE="source.flac"
ffprobe -v quiet -show_entries format_tags=title,artist,album,genre,date \
  -of default=noprint_wrappers=1 "$SOURCE" > tags.txt

ffmpeg -i "$SOURCE" -c:a libopus -b:a 128k \
  -i tags.txt -map_metadata 1 output.opus

# Inline metadata writing
ffmpeg -i input.wav -c:a libopus -b:a 128k \
  -metadata title="Track Title" \
  -metadata artist="Artist Name" \
  -metadata album="Album Name" \
  -metadata date="2024" \
  -metadata genre="Electronic" \
  output.opus

# Conditional metadata (using shell variables)
TITLE="My Song"
ARTIST="My Artist"
ffmpeg -i input.wav -c:a libopus -b:a 128k \
  -metadata title="$TITLE" \
  -metadata artist="$ARTIST" \
  output.opus
```

### 13.4 Metadata Normalization

```bash
# Normalize metadata: trim whitespace, fix capitalization
#!/bin/bash
for file in *.mp3; do
    # Extract tags
    TITLE=$(ffprobe -v quiet -show_entries format_tags=title \
      -of default=noprint_wrappers=1:nokey=1 "$file")
    ARTIST=$(ffprobe -v quiet -show_entries format_tags=artist \
      -of default=noprint_wrappers=1:nokey=1 "$file")

    # Normalize (trim, capitalize)
    TITLE=$(echo "$TITLE" | xargs | sed 's/\b\(.\)/\U\1/g')
    ARTIST=$(echo "$ARTIST" | xargs | sed 's/\b\(.\)/\U\1/g')

    # Write back
    ffmpeg -y -i "$file" -c:a copy \
      -metadata title="$TITLE" \
      -metadata artist="$ARTIST" \
      "${file%.mp3}_normalized.mp3"
done
```

---

## 14. COVER ART HANDLING

### 14.1 Cover Art Extraction and Replacement

```bash
# Extract first cover art (all formats)
ffmpeg -i input.m4a -an -vcodec copy cover.jpg

# Extract from FLAC
ffmpeg -i input.flac -an -vcodec copy cover.png

# Extract from OGG
ffmpeg -i input.ogg -an -vcodec copy cover.jpg

# Extract from MP3
ffmpeg -i input.mp3 -an -vcodec copy cover.jpg

# Extract all images (multiple covers)
# Note: ffmpeg extracts only the first; use ffprobe for counting
ffprobe -v quiet -show_entries stream=codec_type,width,height \
  -of default=noprint_wrappers=1 input.m4a | grep video
```

### 14.2 Embedding Cover Art

```bash
# Embed in MP4/M4A
ffmpeg -i input.m4a -i cover.jpg \
  -c:a copy \
  -map 0:a -map 1:v \
  -disposition:v attached_pic \
  output_with_cover.m4a

# Embed in OGG
ffmpeg -i input.ogg -i cover.jpg \
  -c:a copy \
  -map 0:a -map 1:v \
  -metadata:s:v title="Cover" \
  -metadata:s:v comment="Cover (front)" \
  output_with_cover.ogg

# Embed in Matroska
ffmpeg -i input.mka -i cover.png \
  -c:a copy \
  -map 0:a -map 1:v \
  -metadata:s:v title="Cover" \
  output_with_cover.mka

# Embed in MP3
ffmpeg -i input.mp3 -i cover.jpg \
  -c:a copy \
  -map 0:a -map 1:v \
  output_with_cover.mp3
```

### 14.3 Cover Art Conversion

```bash
# Convert JPEG to PNG for FLAC
ffmpeg -i cover.jpg -f image2 cover.png

# Resize cover to standard size (500x500 max)
ffmpeg -i cover.jpg -vf "scale='min(500,iw)':min'(500,ih)':force_original_aspect_ratio=decrease" \
  cover_500.jpg

# Convert to specific format for container
# FLAC prefers PNG or JPEG
ffmpeg -i cover.png cover_for_flac.png
```

### 14.4 Multiple Cover Art

Some containers support multiple cover images:

| Container | Multiple Covers | Method |
|-----------|--------------|--------|
| MP4 | Yes | Multiple `covr` atoms |
| Matroska | Yes | Multiple `ATTACHED_PICTURE` tags |
| Vorbis | Yes | Multiple `METADATA_BLOCK_PICTURE` |
| FLAC | Yes | Multiple PICTURE blocks |
| MP3 | Yes | Multiple APIC frames |
| WAV | No | Not supported |
| AIFF | No | Via ID3 chunk |

```bash
# Embed multiple covers in Matroska
ffmpeg -i input.mka -i front.jpg -i back.jpg -i booklet.png \
  -c:a copy \
  -map 0:a -map 1:v -map 2:v -map 3:v \
  -metadata:s:v:0 title="Front cover" \
  -metadata:s:v:1 title="Back cover" \
  -metadata:s:v:2 title="Booklet" \
  output_multi_cover.mka
```

---

## 15. REPLAYGAIN IMPLEMENTATION

### 15.1 ReplayGain Scanning

ReplayGain requires scanning the audio to measure loudness:

```bash
# EBU R128 loudness scanning (FFmpeg built-in)
ffmpeg -i input.wav -af loudnorm=print_format=json -f null -

# Output example:
# {
#   "input_i": "-16.42",
#   "input_tp": "-1.24",
#   "input_lra": "6.52",
#   "input_thresh": "-26.84",
#   "output_i": "-16.00",
#   "output_tp": "-1.00",
#   "output_lra": "6.52",
#   "output_thresh": "-25.84",
#   "normalization_type": "dynamic",
#   "target_offset": "0.00"
# }
```

### 15.2 Applying ReplayGain

```bash
# Write ReplayGain tags
ffmpeg -i input.wav -c:a copy \
  -metadata REPLAYGAIN_TRACK_GAIN="-6.20 dB" \
  -metadata REPLAYGAIN_TRACK_PEAK="0.998459" \
  output_with_rg.wav

# Scan and write in one pass (requires external tool like r128gain)
# r128gain input.wav
```

### 15.3 Applying Loudness Normalization

```bash
# Normalize to -16 LUFS (streaming standard)
ffmpeg -i input.wav -af loudnorm=I=-16:TP=-1:LRA=11 \
  -c:a libopus -b:a 128k normalized.opus

# Two-pass normalization (for maximum accuracy)
ffmpeg -i input.wav -af loudnorm=I=-16:TP=-1:LRA=11:print_format=json -f null - 2>&1 | \
  grep -E "I:|TP:|LRA:|Thresh:" > stats.txt

# Apply with measured values
ffmpeg -i input.wav \
  -af loudnorm=I=-16:TP=-1:LRA=11:measured_I=-16.42:measured_TP=-1.24:measured_LRA=6.52:measured_thresh=-26.84 \
  -c:a libopus -b:a 128k normalized.opus
```

---

## 16. MUSICBRAINZ INTEGRATION

### 16.1 MusicBrainz IDs in Metadata

MusicBrainz IDs are UUIDs that uniquely identify entities:

```bash
# Read MusicBrainz IDs from FLAC
ffprobe -v quiet -show_entries format_tags= \
  -of default=noprint_wrappers=1 input.flac | grep MUSICBrainz

# Write MusicBrainz IDs
ffmpeg -i input.wav -c:a copy \
  -metadata "MusicBrainz Track Id"="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx" \
  -metadata "MusicBrainz Album Id"="yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy" \
  -metadata "MusicBrainz Artist Id"="zzzzzzzz-yyyy-yyyy-yyyy-yyyyyyyyyyyy" \
  output_with_mb.flac
```

### 16.2 Using MusicBrainz Lookup

```bash
# Install musicbrainz-cli or Picard for lookup
# Then tag from lookup results
musicbrainz-cli tag --file input.wav --album-id <MB_ALBUM_ID>

# Or use picard in batch
picard --no-player --no-singleton-albums *.wav
```

---

## 17. SPECIALIZED METADATA OPERATIONS

### 17.1 Handling Multi-Value Fields

Vorbis comments support multiple values per field:

```
ARTIST=Artist One
ARTIST=Artist Two
TITLE=Main Title
TITLE=Remastered Version
```

```bash
# Create multi-value tags
ffmpeg -i input.wav -c:a flac \
  -metadata TITLE="Main Title" \
  -metadata TITLE="Remastered Version" \
  output_multi.flac

# Read multi-value tags
ffprobe -v quiet -show_entries format_tags=TITLE \
  -of default=noprint_wrappers=1 input.flac
```

### 17.2 Sort Fields and Normalization

```bash
# Write sort fields for proper alphabetical sorting
ffmpeg -i input.wav -c:a copy \
  -metadata title="The Song Title" \
  -metadata sort_title="Song Title, The" \
  -metadata artist="The Artist Name" \
  -metadata sort_artist="Artist Name, The" \
  -metadata album="Album Name" \
  -metadata sort_album="Album Name" \
  output_sorted.flac
```

### 17.3 Custom/User-Defined Fields

```bash
# Write custom fields
ffmpeg -i input.wav -c:a copy \
  -metadata SOURCE="Vinyl (Original Pressing)" \
  -metadata ENCODER="My Encoder v1.0" \
  -metadata MY_CUSTOM_FIELD="Custom Value" \
  output_custom.flac

# MP4 uses freeform atoms for custom fields
ffmpeg -i input.wav -c:a alac \
  -metadata "----:com.apple.iTunes:SOURCE"="Vinyl (Original Pressing)" \
  -metadata "----:com.apple.iTunes:MY_CUSTOM_FIELD"="Custom Value" \
  output_custom.m4a
```

### 17.4 Removing Specific Tags

```bash
# Remove all tags
ffmpeg -i input.wav -c:a copy -map_metadata -1 output_clean.flac

# Remove specific tag (set to empty)
ffmpeg -i input.wav -c:a copy -metadata title="" output_notitle.flac

# Remove genre only
ffprobe -v quiet -show_entries format_tags -of default=noprint_wrappers=1 input.flac | \
  grep -v "^genre=" | ffmpeg -i input.flac -c:a copy -i - -map_metadata 0 output_nogenre.flac
```

---

## 18. METADATA VALIDATION

### 18.1 Validation Checklist

```bash
# Check all standard tags are present
ffprobe -v quiet -show_entries format_tags=title,artist,album,date,genre,track \
  -of default=noprint_wrappers=1 input.flac

# Check for invalid characters
ffprobe -v quiet -show_entries format_tags -of json input.flac | \
  jq '.format.tags | to_entries[] | select(.value | test("[\\x00-\\x1F]"))'

# Check tag length limits
# Vorbis: No hard limit (practical ~8192 bytes per tag)
# MP4: 255 bytes per atom value
# ID3v2: 64 KB per frame

# Check cover art size
ffprobe -v quiet -show_entries stream=width,height \
  -of default=noprint_wrappers=1 input.m4a | grep -E "width|height"

# Check ReplayGain format
ffprobe -v quiet -show_entries format_tags=REPLAYGAIN_TRACK_GAIN \
  -of default=noprint_wrappers=1 input.flac | \
  grep -E "^[+-][0-9]+\.[0-9]+ dB$" || echo "Invalid format"
```

### 18.2 Automated Validation Script

```bash
#!/bin/bash
# validate_metadata.sh

validate_file() {
    local file="$1"
    echo "Validating: $file"

    # Check required tags
    ffprobe -v quiet -show_entries format_tags=title,artist,album \
      -of default=noprint_wrappers=1 "$file" | \
      while IFS= read -r line; do
        if [ -z "$line" ]; then
          echo "  WARNING: Empty required tag"
        else
          echo "  $line"
        fi
      done

    # Check ReplayGain format
    RG_GAIN=$(ffprobe -v quiet -show_entries format_tags=REPLAYGAIN_TRACK_GAIN \
      -of default=noprint_wrappers=1:nokey=1 "$file")
    if [[ "$RG_GAIN" =~ ^[+-][0-9]+\.[0-9]+[[:space:]]dB$ ]]; then
        echo "  ReplayGain OK"
    elif [ -n "$RG_GAIN" ]; then
        echo "  WARNING: Invalid ReplayGain format: $RG_GAIN"
    fi

    echo ""
}

for f in "$@"; do
    if [ -f "$f" ]; then
        validate_file "$f"
    fi
done
```

---

## 19. CROSS-CONTAINER METADATA CONVERSION

### 19.1 FLAC to Any Format

```bash
# FLAC to MP3 (Vorbis -> ID3)
ffmpeg -i input.flac -c:a libmp3lame -q:a 2 \
  -map_metadata 0 \
  output.mp3

# FLAC to AAC (Vorbis -> iTunes atoms)
ffmpeg -i input.flac -c:a libfdk_aac -vbr 4 \
  -movflags +faststart \
  -map_metadata 0 \
  output.m4a

# FLAC to Opus (Vorbis -> Vorbis)
ffmpeg -i input.flac -c:a libopus -b:a 128k \
  -map_metadata 0 \
  output.opus
```

### 19.2 MP3 to Any Format

```bash
# MP3 to FLAC (ID3 -> Vorbis)
ffmpeg -i input.mp3 -c:a flac -compression_level 8 \
  -map_metadata 0 \
  output.flac

# MP3 to AAC (ID3 -> iTunes atoms)
ffmpeg -i input.mp3 -c:a libfdk_aac -vbr 4 \
  -movflags +faststart \
  -map_metadata 0 \
  output.m4a
```

### 19.3 Preserving All Metadata

```bash
# Use a metadata preservation workflow
# 1. Extract all metadata
ffprobe -v quiet -print_format json -show_format input.flac > metadata.json

# 2. Transcode
ffmpeg -i input.flac -c:a libopus -b:a 128k temp.opus

# 3. Apply metadata from JSON
TITLE=$(jq -r '.format.tags.title' metadata.json)
ARTIST=$(jq -r '.format.tags.artist' metadata.json)
ALBUM=$(jq -r '.format.tags.album' metadata.json)

ffmpeg -i temp.opus -c:a copy \
  -metadata title="$TITLE" \
  -metadata artist="$ARTIST" \
  -metadata album="$ALBUM" \
  output.opus

rm metadata.json temp.opus
```

---

## 20. APPENDIX: TAG LIMITS

### 20.1 Tag Value Length Limits

| Container | Tag Value Limit | Notes |
|-----------|----------------|-------|
| ID3v2.3 | 64 KB per frame | Maximum frame size |
| ID3v2.4 | 256 MB per frame | Maximum frame size |
| Vorbis | ~8 KB per tag (practical) | No spec limit |
| MP4 | 255 bytes per atom value | Hard limit |
| FLAC | Same as Vorbis | Uses Vorbis comments |
| WAV | 256 bytes per chunk | RIFF INFO limit |
| AIFF | 256 bytes per chunk | ANNO limit |
| Matroska | No limit | EBML allows large values |

### 20.2 Tag Name Length Limits

| Container | Tag Name Limit | Notes |
|-----------|----------------|-------|
| ID3v2 | 4 bytes (4 chars) | Fixed-length frame IDs |
| Vorbis | No limit | UTF-8 string |
| MP4 | 4 bytes | 4-char atom codes |
| FLAC | Same as Vorbis | Uses Vorbis comments |
| WAV | 4 bytes | Chunk ID length |
| AIFF | 4 bytes | Chunk ID length |
| Matroska | No limit | UTF-8 string |

### 20.3 Character Encoding Support

| Container | ASCII | Latin-1 | UTF-8 | UTF-16 | Notes |
|-----------|-------|---------|-------|--------|-------|
| ID3v2 | Yes | Yes | Yes | Yes | Text encoding byte per frame |
| Vorbis | Yes | Yes | Yes | No | UTF-8 only |
| MP4 | Yes | Yes | Yes | Yes | UTF-8 or UTF-16 |
| FLAC | Yes | Yes | Yes | No | UTF-8 only |
| WAV | Yes | Yes | No | No | Latin-1/ASCII only |
| AIFF | Yes | Yes | Yes | No | Via ID3 chunk |
| Matroska | Yes | Yes | Yes | No | UTF-8 only |

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete metadata mapping reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
