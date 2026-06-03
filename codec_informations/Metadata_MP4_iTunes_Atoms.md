# MP4/iTunes Metadata Atoms — Deep Technical Reference
> **Category:** Metadata
> **File Extensions:** `.mp4`, `.m4a`, `.m4b`, `.m4p`, `.m4v`, `.m4r`
> **MIME Types:** `audio/mp4`, `audio/x-m4a`, `video/mp4`
> **Standardization Body:** ISO/IEC 14496-12 (ISO Base Media File Format), Apple (iTunes extensions)
> **Primary Specification:** ISO/IEC 14496-12; Apple iTunes metadata specification
> **Patent Status:** Patented (MPEG-4), royalty-bearing for commercial use
> **License:** Proprietary (MP4), Open (ISO base format)
> **Current Version:** iTunes 12.x (latest)
> **Active Development:** Yes — maintained by Apple and ISO

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Apple Computer (iTunes metadata), ISO (MPEG-4 Part 12/14)
- **Year Created:** 2001 (iTunes 1.0), 2003 (iTunes 4 with album art)
- **Original Purpose:** Store rich metadata in MPEG-4 audio files for iTunes Store and iPod playback
- **Problem with Predecessors:** MP3's ID3v2 was inadequate for complex metadata, video metadata, and proprietary iTunes Store identifiers. Apple needed a hierarchical, extensible system that could store track numbers, album art, purchase info, and proprietary identifiers.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| iTunes 1.0 | 2001 | Basic text metadata (title, artist, album) |
| iTunes 2.0 | 2002 | Added genre, BPM, compilation flags |
| iTunes 3.0 | 2003 | Added album art (covr), playlist support |
| iTunes 4.0 | 2003 | Added iTunes Store atoms (cnID, geID, etc.) |
| iTunes 6.0 | 2005 | Video support, enhanced podcast metadata |
| iTunes 7.0 | 2006 | Gapless playback (pgap), artwork improvements |
| iTunes 8.0 | 2008 | Enhanced sorting, TV shows, movie rentals |
| iTunes 12.x | 2015+ | Current — maintains backward compatibility |

### 1.3 Current Adoption
- **Primary use cases today:** iTunes Store, Apple Music, podcast distribution, audiobooks, video
- **Platforms with native support:** macOS, iOS, iPod, Apple TV, Windows (iTunes/Apple Music app)
- **Major services using this format:** Apple Music, iTunes Store, Podcasts, audiobooks, video content
- **Hardware support:** All Apple devices, many third-party players (VLC, mpv, etc.)
- **Status:** Dominant in the Apple ecosystem

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 MP4/ISO Base Media File Structure
MP4 files use a hierarchical atom (box) structure. Every element is an atom/box:

```
┌─────────────────────────────────────────┐
│  ftyp (File Type)                       │  First atom, identifies format
│  ├── major_brand                        │
│  ├── minor_version                      │
│  └── compatible_brands[]                │
├─────────────────────────────────────────┤
│  moov (Movie)                           │  Contains all metadata and timing
│  ├── mvhd (Movie Header)               │
│  ├── trak[] (Tracks)                   │
│  │   ├── tkhd (Track Header)          │
│  │   ├── mdia (Media)                 │
│  │   │   ├── minf (Media Info)        │
│  │   │   └── ...                      │
│  │   └── ...                          │
│  └── udta (User Data)                  │  ← Metadata lives here
│      └── meta (Metadata)               │  ← iTunes metadata container
│          ├── hdlr (Handler)            │
│          ├──ilst (Item List)           │  ← All metadata fields
│          └── [other metadata atoms]     │
├─────────────────────────────────────────┤
│  mdat (Media Data)                      │  Raw audio/video data
├─────────────────────────────────────────┤
│  free (Free Space)                      │  Padding, can be reused
└─────────────────────────────────────────┘
```

### 2.2 Metadata Hierarchy
```
moov → udta → meta → ilst
                          ├── ©nam → data
                          ├── ©ART → data
                          ├── ©alb → data
                          ├── aART → data
                          ├── trkn → data
                          ├── disk → data
                          ├── covr → data[] (multiple)
                          ├── ---- → mean → data
                          │        └── name → data
                          │        └── data
                          └── ...
```

### 2.3 Atom Naming Conventions
| Pattern | Type | Example | Description |
|---------|------|---------|-------------|
| `©xxx` | Text | `©nam` (title) | Copyright symbol prefix for text |
| `xxxx` | Numeric | `trkn` (track number) | Four-character code |
| `----` | Free-form | `----:domain:name` | Reverse-DNS custom fields |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 Atom Structure (Standard Atom — 8-byte header)
```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------------
0x0000   4B      Atom Size              uint32 BE   Size of entire atom (header + data)
0x0004   4B      Atom Type              FOURCC      ASCII four-character type code
0x0008   N       Atom Data              BYTE[]      Atom-specific data
```

**Critical notes:**
- Size is stored in **big-endian** (network byte order) — opposite of WAV/RIFF!
- Size includes the 8-byte header
- If size = 1, the actual size is stored in extended size field (8 bytes after type)
- Atoms can be nested; hierarchy determines meaning

### 3.2 Full Box Structure (Extended Atom — 12-byte header)
Full boxes add a version and flags field:
```
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------------
0x0000   4B      Box Size               uint32 BE   Size of entire box
0x0004   4B      Box Type               FOURCC      ASCII four-character type code
0x0008   1B      Version                uint8       Format version (0, 1, 2, 3)
0x0009   3B      Flags                  uint24 BE   Flags (24-bit big-endian)
0x000C   N       Box Data               BYTE[]      Box-specific data
```

### 3.3 Data Atom Structure (for text/numeric metadata)
Metadata items (like `©nam`, `©ART`) contain a `data` sub-atom:
```
Parent Atom (e.g., ©nam):
├── data (sub-atom)
│   ├── Type/Class (2 bytes)    ← Identifies data format
│   ├── Locale (2 bytes)       ← Language/country code (usually 0)
│   └── Value Data (N bytes)   ← Actual content
```

### 3.4 Data Atom Class Codes (Type Field)
| Class Code | Type | Description |
|-----------|------|-------------|
| 0x00 | Implicit | Type determined by parent atom (default for text) |
| 0x01 | UTF-8 | UTF-8 encoded text |
| 0x02 | UTF-16 | UTF-16 encoded text (big-endian) |
| 0x03 | Reserved/Sort | Used for sort strings |
| 0x0D | JPEG | JPEG image data |
| 0x0E | PNG | PNG image data |
| 0x15 | signed int | Signed 8-bit integer |
| 0x16 | signed int | Signed 16-bit BE integer |
| 0x17 | signed int | Signed 32-bit BE integer |
| 0x18 | signed int | Signed 64-bit BE integer |
| 0x1B | unsigned int | Unsigned 8-bit integer |
| 0x1C | unsigned int | Unsigned 16-bit BE integer |
| 0x1D | unsigned int | Unsigned 32-bit BE integer |
| 0x1E | unsigned int | Unsigned 64-bit BE integer |

---

## 4. COMPLETE ITUNES METADATA ATOM REFERENCE

### 4.1 Text Metadata Atoms
All text atoms follow the same pattern:
```
Atom ©xxx:
  Size:    4 bytes BE
  Type:    ©xxx (4 bytes)
  Data:    data sub-atom:
            Offset 0: Type (2 bytes) = 0x0001 (UTF-8)
            Offset 2: Locale (2 bytes) = 0x0000
            Offset 4: Value (N bytes) = UTF-8 text
```

| Atom Code | Meaning | FFmpeg Key | iTunes Label | Max Length | Notes |
|-----------|---------|------------|--------------|------------|-------|
| `©nam` | Track title | title | Name | 256 chars | Primary identifier |
| `©ART` | Track artist | artist | Artist | 256 chars | Performer |
| `©alb` | Album name | album | Album | 256 chars | |
| `aART` | Album artist | album_artist | Album Artist | 256 chars | |
| `©wrt` | Composer | composer | Composer | 256 chars | Writer |
| `©grp` | Grouping | grouping | Grouping | 256 chars | Work grouping |
| `©gen` | Custom genre | genre | Genre | 256 chars | Freeform genre text |
| `©day` | Release date | date | Year | 4 chars | Year only (YYYY) |
| `©cmt` | Comment | comment | Comment | 256 chars | |
| `©too` | Encoder | encoder | Part of | Encoder/encoded by | Used tool |
| `©enc` | Encoded by | encoded_by | Encoded By | 256 chars | Encoder software |
| `©des` | Description | description | Description | 256 chars | Longer description |
| `cprt` | Copyright | copyright | Copyright | 256 chars | |
| `ldes` | Long description | — | — | 2GB | Long text, no limit |
| `©lyr` | Lyrics | lyrics | Lyrics | 2GB | No 256-byte limit |
| `©pub` | Publisher | publisher | Publisher | 256 chars | |
| `©swr` | Software | software | Software | 256 chars | |
| `©dsc` | Description | — | — | 256 chars | |
| `catg` | Category | — | — | 256 chars | Podcast category |
| `©sol` | Sort Album | — | — | 256 chars | Sort string |
| `©sor` | Sort Artist | — | — | 256 chars | Sort string |
| `©son` | Sort Title | — | — | 256 chars | Sort string |
| `©sow` | Sort Composer | — | — | 256 chars | Sort string |

### 4.2 Numeric Metadata Atoms

#### trkn (Track Number)
```
Atom trkn:
  Size:    4 bytes BE
  Type:    trkn
  Data:    data sub-atom (15 bytes total)
            Offset 0: Type (2 bytes) = 0x0015 (BE uint16)
            Offset 2: Locale (2 bytes) = 0x0000
            Offset 4: Track Number (2 bytes) = uint16 BE
            Offset 6: Total Tracks (2 bytes) = uint16 BE
            Offset 8: Disk Number (2 bytes) = 0 (unused here)
            Offset 10: Total Disks (2 bytes) = 0 (unused here)
```
Note: Some implementations use a compact 8-byte form with just track/total.

#### disk (Disc Number)
```
Atom disk:
  Size:    4 bytes BE
  Type:    disk
  Data:    data sub-atom (compact form, 10 bytes)
            Offset 0: Type (2 bytes) = 0x0015 (BE uint16)
            Offset 2: Locale (2 bytes) = 0x0000
            Offset 4: Disc Number (2 bytes) = uint16 BE
            Offset 6: Total Discs (2 bytes) = uint16 BE
```

#### tmpo (BPM/Tempo)
```
Atom tmpo:
  Size:    4 bytes BE
  Type:    tmpo
  Data:    data sub-atom (6 bytes)
            Offset 0: Type (2 bytes) = 0x0015 (BE uint16)
            Offset 2: Locale (2 bytes) = 0x0000
            Offset 4: Tempo (2 bytes) = BPM (0-65535)
```

#### gnre (Genre — Pre-defined)
```
Atom gnre:
  Size:    4 bytes BE
  Type:    gnre
  Data:    data sub-atom
            Offset 0: Type (2 bytes) = 0x0015 (BE uint16)
            Offset 2: Locale (2 bytes) = 0x0000
            Offset 4: Genre Number (2 bytes) = ID3v1 genre index + 1
```
**Genre Mapping (ID3v1 to gnre):**
| gnre | Genre | gnre | Genre |
|------|-------|------|-------|
| 1 | Blues | 17 | Rock |
| 2 | Classic Rock | 18 | Alternative |
| 3 | Country | 19 | Rock & Roll |
| 4 | Dance | 20 | Heavy Metal |
| 5 | Disco | 21 | Swing |
| 6 | Funk | 22 | Bluegrass |
| 7 | Grunge | 23 | Bass |
| 8 | Hip-Hop | 24 | Soul/R&B |
| 9 | Jazz | 25 | Rap |
| 10 | Metal | 26 | Reggae |
| 11 | New Age | 27 | Classical |
| 12 | Oldies | 28 | Soundtrack |
| 13 | Other | 29 | Opera |
| 14 | Pop | 30 | Chamber Music |
| 15 | R&B | 31 | Nature |
| 16 | Rap | 32 | Jazz |

### 4.3 Boolean/Flag Atoms

#### cpil (Compilation)
```
Atom cpil:
  Size:    4 bytes BE
  Type:    cpil
  Data:    data sub-atom (4 bytes)
            Offset 0: Type (2 bytes) = 0x0015 (BE uint8, stored as uint16)
            Offset 2: Locale (2 bytes) = 0x0000
            Offset 4: Value (1 byte) = 0x00 (not part of compilation) or 0x01 (part of compilation)
```

#### pgap (Gapless Playback)
```
Atom pgap:
  Size:    4 bytes BE
  Type:    pgap
  Data:    data sub-atom (4 bytes)
            Offset 0: Type (2 bytes) = 0x0015 (BE uint8)
            Offset 2: Locale (2 bytes) = 0x0000
            Offset 4: Value (1 byte) = 0x00 (no gapless) or 0x01 (gapless)
```
Note: Used for gapless albums in iTunes. Indicates the album should be played without gaps between tracks.

#### pcst (Podcast Flag) [NEEDS VERIFICATION]
```
Atom pcst:
  Size:    4 bytes BE
  Type:    pcst
  Data:    data sub-atom
            Offset 0: Type (2 bytes) = 0x0015
            Offset 2: Locale (2 bytes) = 0x0000
            Offset 4: Value (1 byte) = 0x01 (podcast)
```

#### hdvd (HD Video)
```
Atom hdvd:
  Size:    4 bytes BE
  Type:    hdvd
  Data:    data sub-atom
            Offset 0: Type (2 bytes) = 0x0015
            Offset 2: Locale (2 bytes) = 0x0000
            Offset 4: Value (1 byte) = 0x00 (SD) or 0x01 (HD)
```

#### stik (Media Type / Video)
```
Atom stik:
  Size:    4 bytes BE
  Type:    stik
  Data:    data sub-atom (4 bytes)
            Offset 0: Type (2 bytes) = 0x0015 (BE uint8)
            Offset 2: Locale (2 bytes) = 0x0000
            Offset 4: Value (1 byte) = Media type
```
| Value | Meaning |
|-------|---------|
| 0 | Movie |
| 1 | Normal (music) |
| 2 | Audiobook |
| 5 | Whacked! |
| 6 | Music Video |
| 9 | Movie (iTunes Store) |
| 10 | Podcast |
| 11 | Rental (deprecated) |
| 14 | Video Podcast |
| 24 | HD Video |

#### rtng (Content Rating)
```
Atom rtng:
  Size:    4 bytes BE
  Type:    rtng
  Data:    data sub-atom (4 bytes)
            Offset 0: Type (2 bytes) = 0x0015
            Offset 2: Locale (2 bytes) = 0x0000
            Offset 4: Value (1 byte) = 0x00 (none) or 0x02 (clean) or 0x04 (explicit)
```

### 4.4 Cover Art (covr)

Cover art is the most complex metadata atom:
```
Atom covr:
  ├── data (image 1) — JPEG or PNG
  │   ├── Type: 0x0D (JPEG) or 0x0E (PNG)
  │   ├── Locale: 0x0000
  │   └── Value: Raw image bytes
  ├── data (image 2) — Optional second image
  └── ... (multiple data atoms allowed)
```

**Rules:**
- Multiple data atoms are allowed within a single covr atom
- Each data atom can contain a different image (front cover, back cover, etc.)
- JPEG is most common; PNG also supported
- Maximum image size: implementation-dependent (recommend < 10MB)
- Recommended: 600×600 to 3000×3000 pixels

**Hex dump example:**
```
00000000  00 00 00 3C 63 6F 76 72                      ; Size=60, "covr"
00000008  00 00 00 38 64 61 74 61                      ; Size=56, "data"
00000010  00 0D 00 00                                   ; Type=0x0D (JPEG), Locale=0
00000014  FF D8 FF E0 00 10 4A 46 49 46 00 01 01 ...  ; JPEG magic + data
```

### 4.5 Free-Form Atoms (----)

Free-form atoms use a reverse-DNS namespace for custom fields:
```
Atom ----:
  ├── mean: issuer domain (e.g., "com.apple.iTunes")
  │   └── data: "com.apple.iTunes" (UTF-8)
  ├── name: field descriptor
  │   └── data: "iTunNORM" (UTF-8)
  └── data: actual value
      └── data: "00000A5E ..." (UTF-8 or binary)
```

**Common ---- atoms:**
| Mean (Issuer) | Name (Field) | Description | Value Type |
|--------------|--------------|-------------|------------|
| com.apple.iTunes | iTunNORM | Sound Check normalization | ASCII hex |
| com.apple.iTunes | iTunPGAP | Gapless flag | "0" or "1" |
| com.apple.iTunes | iTunSMPB | Gapless encoding padding | Hex sample counts |
| com.apple.iTunes | GAPLESS | Gapless flag | "1" |
| com.apple.iTunes | LANGUAGE | Track language | ISO 639-2 code |
| com.apple.iTunes | MOVIEID | iTunes Store movie ID | Integer |
| com.apple.iTunes | TVSHOWID | TV show ID | Integer |
| com.musicbrainz | albumid | MusicBrainz album ID | UUID |
| com.musicbrainz | trackid | MusicBrainz track ID | UUID |
| com.musicbrainz | artistid | MusicBrainz artist ID | UUID |
| com.musicbrainz | releasegroupid | MusicBrainz release group ID | UUID |

### 4.6 iTunes Store Atoms (cnID, geID, plID, sfID, apID, atID)

These atoms store iTunes Store identifiers:

#### cnID (Catalog ID)
```
Atom cnID:
  Size:    4 bytes BE
  Type:    cnID
  Data:    data sub-atom (10 bytes)
            Offset 0: Type (2 bytes) = 0x001D (BE uint32)
            Offset 2: Locale (2 bytes) = 0x0000
            Offset 4: Catalog ID (4 bytes) = uint32 BE
```
Used to identify the track in the iTunes Store catalog.

#### geID (Genre ID)
```
Atom geID:
  Size:    4 bytes BE
  Type:    geID
  Data:    data sub-atom (10 bytes)
            Offset 0: Type (2 bytes) = 0x001D (BE uint32)
            Offset 2: Locale (2 bytes) = 0x0000
            Offset 4: Genre ID (4 bytes) = uint32 BE
```

#### sfID (Storefront ID)
```
Atom sfID:
  Size:    4 bytes BE
  Type:    sfID
  Data:    data sub-atom (10 bytes)
            Offset 0: Type (2 bytes) = 0x001D (BE uint32)
            Offset 2: Locale (2 bytes) = 0x0000
            Offset 4: Storefront ID (4 bytes) = uint32 BE
```
| sfID | Country |
|------|---------|
| 143441 | United States |
| 143442 | France |
| 143443 | Germany |
| 143444 | United Kingdom |
| 143445 | Austria |
| 143446 | Belgium (Dutch) |
| 143447 | Belgium (French) |
| 143448 | Finland |
| 143449 | Netherlands |
| 143450 | Sweden |
| 143461 | Canada |
| 143462 | Japan |
| 143489 | Australia |

#### plID (Playlist ID) [NEEDS VERIFICATION]
```
Atom plID:
  Size:    4 bytes BE
  Type:    plID
  Data:    data sub-atom (12 bytes)
            Offset 0: Type (2 bytes) = 0x001E (BE uint64)
            Offset 2: Locale (2 bytes) = 0x0000
            Offset 4: Playlist ID (8 bytes) = uint64 BE
```

#### apID (Account ID)
```
Atom apID:
  Size:    4 bytes BE
  Type:    apID
  Data:    data sub-atom
            Offset 0: Type (2 bytes) = 0x0001 (UTF-8)
            Offset 2: Locale (2 bytes) = 0x0000
            Offset 4: Account name (UTF-8 string)
```

#### atID (Artist ID)
```
Atom atID:
  Size:    4 bytes BE
  Type:    atID
  Data:    data sub-atom (10 bytes)
            Offset 0: Type (2 bytes) = 0x001D (BE uint32)
            Offset 2: Locale (2 bytes) = 0x0000
            Offset 4: Artist ID (4 bytes) = uint32 BE
```

### 4.7 Chapter Atoms (chpl)

Chapter metadata uses the `chpl` atom:
```
Atom chpl:
  Size:    4 bytes BE
  Type:    chpl
  Data:    Full box with version/flags
            Offset 0: Version (1 byte) = 0x00
            Offset 1: Flags (3 bytes) = 0x000000
            Offset 4: Reserved (1 byte) = 0x00
            Offset 5: Chapter count (1 byte) = N
            [N chapters:]
              - Start time (8 bytes) = uint64 BE (ms)
              - Title length (1 byte) = L
              - Title (L bytes) = UTF-8 string
```

---

## 5. COMPLETE METADATA HIERARCHY EXAMPLE

### 5.1 Minimal iTunes Metadata Structure
```
moov
 └── udta
      └── meta
           ├── hdlr (Handler: "mdta")
           ├── ilst
           │    ├── ©nam
           │    │    └── data (UTF-8 text)
           │    ├── ©ART
           │    │    └── data (UTF-8 text)
           │    ├── ©alb
           │    │    └── data (UTF-8 text)
           │    ├── trkn
           │    │    └── data (uint16 array)
           │    └── covr
           │         └── data (JPEG/PNG binary)
           └── [other metadata]
```

### 5.2 Complete Atom Hierarchy with All Fields
```
moov (Movie)
 └── udta (User Data)
      └── meta (Metadata)
           ├── hdlr (Handler)
           │    └── Data: "mdta" (metadata handler)
           ├── ilst (Item List) ← Main metadata container
           │    ├── ©nam → data ("Song Title")
           │    ├── ©ART → data ("Artist Name")
           │    ├── ©alb → data ("Album Name")
           │    ├── aART → data ("Album Artist")
           │    ├── ©wrt → data ("Composer")
           │    ├── ©grp → data ("Work/Grouping")
           │    ├── ©gen → data ("Custom Genre")
           │    ├── gnre → data (uint16 genre index + 1)
           │    ├── ©day → data ("2024")
           │    ├── trkn → data (track/total array)
           │    ├── disk → data (disc/total array)
           │    ├── tmpo → data (BPM as uint16)
           │    ├── ©cmt → data ("Comment text")
           │    ├── ©too → data ("Encoder info")
           │    ├── cprt → data ("Copyright notice")
           │    ├── pgap → data (0x01 gapless)
           │    ├── cpil → data (0x01 compilation)
           │    ├── stik → data (0x01 normal audio)
           │    ├── rtng → data (0x00 none)
           │    ├── covr → data[] (JPEG/PNG images)
           │    ├── ----
           │    │    ├── mean → data ("com.apple.iTunes")
           │    │    ├── name → data ("iTunNORM")
           │    │    └── data ("00000A5E ...")
           │    ├── cnID → data (iTunes catalog ID)
           │    ├── geID → data (genre ID)
           │    ├── sfID → data (storefront ID)
           │    └── apID → data (account name)
           └── [reserved-free space]
```

### 5.3 Hex Dump Example — Simple Title + Artist
```
00000000  6D 6F 6F 76                                    ; "moov"
[moov contents...]
         75 64 74 61                                    ; "udta"
         6D 65 74 61                                    ; "meta"
         00 00 00 18 68 64 6C 72                        ; Size=24, "hdlr"
         00 00 00 00 00 00 00 00                        ; Version=0, Flags=0, PreUsel
         6D 64 74 61                                    ; "mdta" handler type
         00 00 00 5C 69 6C 73 74                        ; Size=92, "ilst"
         
         00 00 00 19 A9 6E 61 6D                        ; Size=25, "©nam"
         00 00 00 11 64 61 74 61                        ; Size=17, "data"
         00 01 00 00                                    ; Type=UTF-8, Locale=0
         54 65 73 74 20 53 6F 6E 67                     ; "Test Song"
         
         00 00 00 1D A9 41 52 54                        ; Size=29, "©ART"
         00 00 00 15 64 61 74 61                        ; Size=21, "data"
         00 01 00 00                                    ; Type=UTF-8, Locale=0
         54 65 73 74 20 41 72 74 69 73 74              ; "Test Artist"
```

---

## 6. FFmpeg MP4 METADATA REFERENCE

### 6.1 FFmpeg Identifiers
```
Codec Name (CLI):   aac, alac, mp3 (in MP4 container)
Format Name (CLI):  mp4, ipod, iphone, m4a, f4v
Demuxer(s):         mov (MP4), ipod
Muxer(s):           ipod, mov, mp4, f4v
```

### 6.2 FFmpeg Encoding with Metadata
```bash
# Basic MP4 encoding with metadata
ffmpeg -i input.wav -c:a aac -b:a 256k \
  -metadata title="Song Title" \
  -metadata artist="Artist Name" \
  -metadata album="Album Name" \
  -metadata date="2024" \
  -metadata track="5/12" \
  -metadata disc="1/2" \
  -metadata genre="Electronic" \
  output.m4a

# iTunes-compatible encoding
ffmpeg -i input.wav -c:a aac -b:a 256k \
  -movflags +ipod \
  -metadata title="Title" \
  -metadata artist="Artist" \
  output.m4a

# Gapless encoding
ffmpeg -i input.wav -c:a alac \
  -movflags +gapless \
  output.m4a
```

### 6.3 Complete FFmpeg Metadata Option Table
| Option | Type | Description |
|--------|------|-------------|
| `-metadata` | key=value | Set metadata |
| `-movflags` | flags | MP4/mov format flags |
| `-write_colr` | bool | Write color information |
| `-write_gama` | bool | Write gamma information |
| `-brand` | string | Set major brand |

**`-movflags` options:**
| Flag | Effect |
|------|--------|
| `+gapless` | Enable gapless playback support |
| `+ipod` | Enable iPod/iPhone compatibility |
| `+isml` | Enable ISMA compatibility |
| `+frag_keyframe` | Fragment at keyframes |
| `+faststart` | Move moov atom before mdat |
| `+default_base_moof` | Use default base MOOF offset |

### 6.4 FFmpeg Metadata Reading
```bash
# Read all metadata
ffprobe -v quiet -print_format json -show_format input.m4a | jq .format.tags

# Read specific fields
ffprobe -v quiet -show_entries format_tags=title,artist,album -of default input.m4a

# Read cover art
ffprobe -v quiet -show_entries format_tags -of json input.m4a
ffmpeg -i input.m4a -an -vcodec copy cover.jpg

# Read all atoms (for debugging)
ffprobe -v debug -show_packets input.m4a 2>&1 | grep "tag"
```

### 6.5 FFmpeg Internal Metadata Key Mapping (MP4/iTunes)
| Standard Field | FFmpeg Key | MP4 Atom | Notes |
|----------------|------------|----------|-------|
| Title | title | ©nam | UTF-8 text |
| Artist | artist | ©ART | UTF-8 text |
| Album | album | ©alb | UTF-8 text |
| Album Artist | album_artist | aART | UTF-8 text |
| Composer | composer | ©wrt | UTF-8 text |
| Grouping | grouping | ©grp | UTF-8 text |
| Genre | genre | ©gen / gnre | Custom or pre-defined |
| Year/Date | date | ©day | UTF-8 text (YYYY) |
| Track Number | track | trkn | Compact uint16 array |
| Disc Number | disc | disk | Compact uint16 array |
| Comment | comment | ©cmt | UTF-8 text |
| BPM/Tempo | tempo | tmpo | uint16 |
| Compilation | compilation | cpil | Boolean (0/1) |
| Gapless | gapless | pgap | Boolean (0/1) |
| Encoder | encoder | ©too | UTF-8 text |
| Copyright | copyright | cprt | UTF-8 text |
| Cover Art | — | covr | Binary JPEG/PNG |
| Description | description | ©des | UTF-8 text |
| Long Description | — | ldes | UTF-8 text (no limit) |
| Lyrics | lyrics | ©lyr | UTF-8 text (no limit) |
| Sort Album | album_sort | ©sor | UTF-8 text |
| Sort Artist | artist_sort | ©soa | UTF-8 text |
| Sort Title | title_sort | ©son | UTF-8 text |
| Sort Composer | composer_sort | ©sow | UTF-8 text |

### 6.6 Tag Round-Trip with FFmpeg
```bash
# Copy all tags from one MP4 to another
ffmpeg -i input.m4a -c:a copy -c:v copy \
  -map_metadata 0 \
  output.m4a

# Copy specific tags
ffmpeg -i input.m4a -c:a copy \
  -metadata title="New Title" \
  -metadata artist="New Artist" \
  -movflags +faststart \
  output.m4a

# Embed cover art
ffmpeg -i input.m4a -i cover.jpg \
  -c:a copy \
  -c:v copy \
  -map 0:a -map 1:v \
  -metadata:s:v title="Cover" \
  -metadata:s:v comment="Cover (front)" \
  output.m4a

# Strip all metadata
ffmpeg -i input.m4a -c:a copy -c:v copy \
  -map_metadata -1 \
  -map_chapters -1 \
  output.m4a
```

---

## 7. CHAPTER ATOMS

### 7.1 Chapter List Structure (chpl)
```
Atom chpl:
  Size:    4 bytes BE
  Type:    chpl
  Version: 0x00
  Flags:   0x000000
  Reserved: 0x00
  Chapter Count: N (1 byte)
  
  [Repeat N times:]
  ├── Start Time: uint64 BE (milliseconds)
  ├── Title Length: uint8
  └── Title: UTF-8 string (Title Length bytes)
```

### 7.2 Chapter Track Atoms (chap within trak)
For gapless chapter markers, chapters can also be stored as track atoms:
```
moov
 └── trak[] (one per chapter)
      ├── tkhd (track header, chapter track)
      ├── mdia
      │    └── hdlr (chapter handler)
      └── meta
           └── ilst
                └── chpl (chapter list)
```

### 7.3 Reading/Writing Chapters with FFmpeg
```bash
# FFmpeg cannot write chapters directly in MP4
# Use a tool like mp4chaps, AtomicParsley, or mp4art

# Extract chapters to file
mp4chaps --extract input.m4a

# Import chapters from file
mp4chaps --import chapters.txt input.m4a

# List chapters
mp4chaps --list input.m4a

# FFmpeg can burn chapters into output
ffmpeg -i input.m4a -c:a copy \
  -f ffmetadata chapters.txt \
  output.m4a
```

---

## 8. GAPLESS PLAYBACK IN MP4

### 8.1 Gapless Mechanisms
iTunes uses multiple mechanisms for gapless playback:

1. **pgap atom:** Boolean flag indicating the album should play gaplessly
2. **iTunSMPB atom:** Contains encoder delay and padding in samples
   ```
   Format: "00000000 00000B40 000001E0 00000000 00000200 00000000"
           (8 hex chars = 4 bytes each)
   
   Fields:
   - Encoder delay (in samples, hex)
   - Playback start trim (in samples, hex)
   - Playback end trim (in samples, hex)
   - Total samples (hex)
   - Reserved
   - Reserved
   ```
3. **Track ordering:** Tracks played in order within album

### 8.2 Gapless Extraction
```bash
# Extract gapless parameters
# Read iTunSMPB from file

# For proper gapless:
# 1. Track 1: skip first N samples (encoder delay)
# 2. All tracks: play until last sample before encoder padding
# 3. Final track: play all remaining samples including padding
```

---

## 9. FREE-FORM / CUSTOM ATOMS

### 9.1 iTunes Normalization Atom (iTunNORM)
```
----:com.apple.iTunes:iTunNORM
  └── value: "00000A5E 00000A5E 00000A5E 00000A5E..."
              (8-char hex pairs, one per channel)
```
Contains preamp values for Sound Check normalization.

### 9.2 MusicBrainz Atoms
```
----:com.musicbrainz:albumid
  └── value: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

----:com.musicbrainz:trackid
  └── value: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

----:com.musicbrainz:artistid
  └── value: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

----:com.musicbrainz:releasegroupid
  └── value: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

### 9.3 Rating Atom
```
----:com.apple.iTunes:Rating
  └── value: "0" to "100" (integer as string)
```

---

## 10. TOOLS FOR MP4 METADATA

### 10.1 Command-Line Tools
| Tool | Platform | License | Key Features |
|------|----------|---------|--------------|
| AtomicParsley | Cross | GPL | Full metadata editing, artwork |
| FFmpeg | Cross | LGPL | Encoding, metadata, format conversion |
| mp4art | Cross | GPL | Cover art only |
| mp4chaps | Cross | GPL | Chapter management |
| id3tool | Linux | GPL | ID3-like interface |
| mkvpropeds | Cross | GPL | Property editing |
| Bento4 | Cross | Custom | MP4 parsing and editing |

### 10.2 AtomicParsley Examples
```bash
# Set basic metadata
AtomicParsley input.m4a --artist "Artist" --title "Title" --album "Album" --tracknum 5 --artwork cover.jpg

# Set iTunes Store atoms
AtomicParsley input.m4a --cnID 123456789 --geID 20 --sfID 143441

# Remove specific atoms
AtomicParsley input.m4a --©ART "" --overwrite

# Remove all metadata
AtomicParsley input.m4a --metaEnema --overwrite

# List all atoms
AtomicParsley input.m4a --extract-atom

# Set free-form atoms
AtomicParsley input.m4a --reverseDNS --domain "com.musicbrainz" --freeform "trackid=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

### 10.3 FFmpeg Metadata Examples
```bash
# Set title
ffmpeg -i input.m4a -c copy -metadata title="New Title" output.m4a

# Set multiple tags
ffmpeg -i input.m4a -c copy \
  -metadata title="Title" \
  -metadata artist="Artist" \
  -metadata album="Album" \
  -metadata track="5" \
  output.m4a

# Embed artwork
ffmpeg -i input.m4a -i cover.jpg \
  -c copy -c:v copy \
  -map 0:a -map 1:v \
  -disposition:v attached_pic \
  output.m4a

# Copy tags during transcoding
ffmpeg -i input.m4a -c:a alac -map_metadata 0 output.m4a
```

---

## 11. EDGE CASES & KNOWN ISSUES

### 11.1 Common Pitfalls
| Issue | Cause | Solution |
|-------|-------|----------|
| Cover art not showing | covr atom wrong type | Use 0x0D (JPEG) or 0x0E (PNG) |
| Non-ASCII characters | UTF-8 encoding | Ensure proper UTF-8 throughout |
| Large files | moov at end | Use `-movflags +faststart` |
| Duplicate atoms | Re-encoding | Remove before adding |
| Genre conflict | gnre vs ©gen | Use ©gen for custom, gnre for standard |
| Sort fields not working | Wrong atom names | Use ©son, ©sor, ©soa, ©sow |

### 11.2 Interoperability Issues
| Tool | Issue | Workaround |
|------|-------|------------|
| QuickTime | Limited metadata support | Use iTunes-compatible atoms |
| VLC | May not read all atoms | Standard atoms work |
| Android | Variable support | Stick to standard ©xxx atoms |
| FFmpeg | Some custom atoms not preserved | Use -map_metadata 0 |
| AtomicParsley | Limited to 256 chars | Use ldes/©lyr for long text |

### 11.3 Atom Order
Atom order within ilst is not strictly enforced, but conventionally:
```
©nam, ©ART, ©alb, aART, ©wrt, ©grp, gnre, ©gen, ©day, trkn, disk, tmpo, ©cmt, ©too, cprt, pgap, cpil, covr, ----, cnID, geID, sfID, apID, atID, plID
```

---

## 12. CONVERSION GUIDE

### 12.1 Converting TO MP4 with Metadata
| Source | FFmpeg Command | Metadata |
|--------|---------------|----------|
| FLAC | `ffmpeg -i in.flac -c:a alac out.m4a` | Via -map_metadata |
| MP3 | `ffmpeg -i in.mp3 -c:a aac -b:a 256k out.m4a` | Via -map_metadata |
| WAV | `ffmpeg -i in.wav -c:a aac -b:a 256k out.m4a` | Via -metadata |
| OGG | `ffmpeg -i in.ogg -c:a aac -b:a 256k out.m4a` | Via -map_metadata |
| Opus | `ffmpeg -i in.opus -c:a aac -b:a 256k out.m4a` | Via -map_metadata |

### 12.2 Converting FROM MP4
| Target | FFmpeg Command | Notes |
|--------|---------------|----------|
| FLAC | `ffmpeg -i in.m4a -c:a flac -compression_level 8 out.flac` | Lossless |
| WAV | `ffmpeg -i in.m4a -c:a pcm_s16le out.wav` | |
| MP3 | `ffmpeg -i in.m4a -c:a libmp3lame -q:a 0 out.mp3` | ID3v2.3 |
| OGG | `ffmpeg -i in.m4a -c:a libvorbis -q:a 6 out.ogg` | Vorbis Comments |

### 12.3 Metadata Preservation Matrix
| Source | MP4 | FLAC | MP3 | OGG | Notes |
|--------|-----|------|-----|-----|-------|
| ©nam | title | title | title | title | |
| ©ART | artist | artist | artist | artist | |
| ©alb | album | album | album | album | |
| aART | album_artist | album_artist | album_artist | album_artist | |
| trkn | track | track | track | track | |
| covr | covr | picture | APIC | picture | |
| gnre/©gen | gnre/©gen | genre | genre | genre | |
| REPLAYGAIN | ---- | replaygain_* | replaygain_* | replaygain_* | |

---

## 13. REFERENCE IMPLEMENTATIONS

| Library | Language | License | Notes |
|---------|----------|---------|-------|
| FFmpeg (libavformat) | C | LGPL 2.1+ | mov/mp4 muxing and demuxing |
| mp4v2 | C++ | MPL 1.1 | Full iTunes metadata support |
| AtomicParsley | C++ | GPL | CLI metadata editing |
| Bento4 (mp4parser) | C++ | Custom | MP4 parsing and editing |
| mp4parse-rust | Rust | Apache 2.0 | Mozilla's Rust implementation |
| TagLib | C++ | LGPL | Basic metadata support |

---

## 14. SPECIFICATIONS & FURTHER READING

### Primary Specifications
- **ISO/IEC 14496-12:** ISO Base Media File Format (MPEG-4 Part 12)
- **ISO/IEC 14496-14:** MP4 file format (MPEG-4 Part 14)
- **iTunes Metadata Format:** Apple specification (informal)
- **RFC 6381:** Content-Type for MP4

### Technical Resources
- AtomicParsley documentation: https://bitbucket.org/jonhander/atomicparsley
- mp4v2 library: https://code.google.com/archive/p/mp4v2/
- Bento4 documentation: https://www.bento4.com/documentation/
- Hydrogenaudio MP4: https://wiki.hydrogenaudio.org
- Mozilla mp4parse: https://github.com/mozilla/mp4parse-rust

---

## 15. IMPLEMENTATION CHECKLIST

### Parsing Pipeline
- [ ] Read ftyp atom to verify MP4/M4A format
- [ ] Locate moov → udta → meta → ilst hierarchy
- [ ] Parse each ilst child atom as a metadata item
- [ ] For text atoms (©xxx): read data sub-atom with UTF-8 type (0x01)
- [ ] For numeric atoms (trkn, disk, tmpo): read data sub-atom as uint
- [ ] For boolean atoms (cpil, pgap): read data sub-atom as uint8
- [ ] For covr: read multiple data sub-atoms as JPEG/PNG binary
- [ ] For ---- atoms: parse mean/name/data sub-hierarchy

### Writing Pipeline
- [ ] Locate or create moov → udta → meta → ilst hierarchy
- [ ] For text: create atom with data sub-atom (type=0x01, locale=0)
- [ ] For numeric: create atom with data sub-atom (type matches value size)
- [ ] For boolean: create atom with data sub-atom (type=0x15)
- [ ] For cover art: create covr with data sub-atom (type=0x0D or 0x0E)
- [ ] Use `-movflags +faststart` to move moov before mdat
- [ ] Validate UTF-8 encoding for all text values

### Metadata Fields
- [ ] Read/write standard ©xxx text atoms
- [ ] Read/write trkn (compact array format)
- [ ] Read/write disk (compact array format)
- [ ] Read/write gnre vs ©gen (standard vs custom genre)
- [ ] Read/write tmpo (BPM)
- [ ] Read/write cpil, pgap (boolean flags)
- [ ] Read/write covr (multiple JPEG/PNG images)
- [ ] Read/write ---- free-form atoms (mean/name structure)
- [ ] Read/write iTunes Store atoms (cnID, geID, sfID, apID, atID, plID)
- [ ] Read/write chapter atoms (chpl)
- [ ] Handle sort atoms (©son, ©sor, ©soa, ©sow)

### Edge Cases
- [ ] Handle missing ilst (no metadata present)
- [ ] Handle duplicate atoms (use first or last)
- [ ] Handle non-standard atom types
- [ ] Handle corrupted data sub-atoms
- [ ] Handle large files with moov at end
- [ ] Handle mixed encodings (UTF-8 vs UTF-16)
- [ ] Handle non-ASCII characters in keys
- [ ] Handle multiple cover art images

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
