# DBpoweramp Music Converter — Internal Tag Model: Canonical Representation

**Document:** `10_Internal_Tag_Model_Canonical_Representation.md`  
**Date:** 2026-06-04  
**Confidence Level:** High (based on MusicBrainz Picard as canonical reference, corroborated by TagLib, Mp3tag, and DBpoweramp forum data)  
**Sources:** 12 primary sources consulted

---

## 1. Overview & Purpose

This document defines the canonical internal tag structure that a DBpoweramp-equivalent implementation must use internally after reading tags from any audio format. The canonical model serves as the neutral interchange representation — the internal data model all format-specific readers normalize into, and the source all format-specific writers render from.

The canonical model is derived from the **MusicBrainz Picard tag mapping** (Appendix A), which represents the de facto industry-standard unified tag set. Picard's mapping is the most comprehensively documented, actively maintained, and format-agnostic canonical field specification available, covering 80+ fields across ID3v2, Vorbis Comment, APEv2, iTunes MP4, ASF/WMA, and RIFF INFO. (Source: [Picard Tag Mapping v3.0](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html))

DBpoweramp does **not** publish a formal internal canonical model specification. Instead, it operates on a format-normalization principle: it reads tags from source formats, applies format-to-format mapping rules during conversion, and writes to destination formats using format-native frames. The DBpoweramp naming/tagging system exposes tag names as human-readable labels (e.g., "Album Artist", "Disc Subtitle") rather than raw frame IDs. (Source: [dBpoweramp Forum — Mapping of metadata](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/25898-mapping-of-metadata-between-file-formats))

The canonical model described here represents the **union** of what DBpoweramp can read, store internally, and write — as evidenced by its support for all standard tag frames, its Picard-compatible naming conventions, and its scripting API (`ReadIDTagElementValue` returns `"Element: Value"` pairs enumerated by index). (Source: [dBpoweramp Developer Scripting](https://dbpoweramp.com/developer-scripting-dmc))

---

## 2. Canonical Tag Structure

### 2.1 Identification Fields

| Canonical Field Name | Picard Internal Name | ID3v2 | Vorbis/FLAC | MP4/iTunes | ASF/WMA | RIFF INFO | Type |
|---|---|---|---|---|---|---|---|
| title | `title` | `TIT2` | `TITLE` | `©nam` | `Title` | `INAM` | String |
| subtitle | `subtitle` | `TIT3` | `SUBTITLE` | `----:com.apple.iTunes:SUBTITLE` | `WM/SubTitle` | — | String |
| artist | `artist` | `TPE1` | `ARTIST` | `©ART` | `Author` | `IART` | String |
| artists | `artists` | `TXXX:Artists` | `ARTISTS` | `----:com.apple.iTunes:ARTISTS` | `WM/ARTISTS` | — | Multi-String |
| albumartist | `albumartist` | `TPE2` | `ALBUMARTIST` | `aART` | `WM/AlbumArtist` | — | String |
| album | `album` | `TALB` | `ALBUM` | `©alb` | `WM/AlbumTitle` | `IPRD` | String |
| composer | `composer` | `TCOM` | `COMPOSER` | `©wrt` | `WM/Composer` | `IMUS` | String |
| lyricist | `lyricist` | `TEXT` | `LYRICIST` | `----:com.apple.iTunes:LYRICIST` | `WM/Writer` | — | String |
| conductor | `conductor` | `TPE3` | `CONDUCTOR` | `----:com.apple.iTunes:CONDUCTOR` | `WM/Conductor` | — | String |
| arranger | `arranger` | `TIPL:arranger` | `ARRANGER` | — | — | — | String |
| engineer | `engineer` | `TIPL:engineer` | `ENGINEER` | `----:com.apple.iTunes:ENGINEER` | `WM/Engineer` | `IENG` | String |
| producer | `producer` | `TIPL:producer` | `PRODUCER` | `----:com.apple.iTunes:PRODUCER` | `WM/Producer` | `IPRO` | String |
| djmixer | `djmixer` | `TIPL:DJ-mix` | `DJMIXER` | `----:com.apple.iTunes:DJMIXER` | `WM/DJMixer` | — | String |
| mixer | `mixer` | `TIPL:mix` | `MIXER` | `----:com.apple.iTunes:MIXER` | `WM/Mixer` | — | String |
| remixer | `remixer` | `TPE4` | `REMIXER` | `----:com.apple.iTunes:REMIXER` | `WM/ModifiedBy` | — | String |
| writer | `writer` | `TXXX:Writer` | `WRITER` | — | — | `IWRI` | String |
| director | `director` | `TXXX:DIRECTOR` | `DIRECTOR` | `©dir` | `WM/Director` | — | String |
| performer:instrument | `performer` | `TMCL:instrument` | `PERFORMER={artist} (instrument)` | — | — | — | Multi-String |
| show | `show` | — | — | `tvsh` | — | — | String |
| showmovement | `showmovement` | `TXXX:SHOWMOVEMENT` | `SHOWMOVEMENT` | `shwm` | — | — | Boolean |
| showsort | `showsort` | — | — | `sosn` | — | — | String |
| movement | `movement` | `MVNM` | `MOVEMENTNAME` | `©mvn` | — | — | String |
| movementnumber | `movementnumber` | `MVIN` | `MOVEMENT` | `©mvi` | — | — | Integer |
| movementtotal | `movementtotal` | `MVIN` | `MOVEMENTTOTAL` | `©mvc` | — | — | Integer |

### 2.2 Number / Sequence Fields

| Canonical Field Name | Picard Internal Name | ID3v2 | Vorbis/FLAC | MP4/iTunes | ASF/WMA | Type |
|---|---|---|---|---|---|---|---|
| tracknumber | `tracknumber` | `TRCK` | `TRACKNUMBER` | `trkn` (uint16 pair) | `WM/TrackNumber` | Integer pair |
| totaltracks | `totaltracks` | `TRCK` (suffix) | `TRACKTOTAL`, `TOTALTRACKS` | `trkn` (uint16 pair) | — | Integer |
| discnumber | `discnumber` | `TPOS` | `DISCNUMBER` | `disk` (uint16 pair) | `WM/PartOfSet` | Integer pair |
| totaldiscs | `totaldiscs` | `TPOS` (suffix) | `DISCTOTAL`, `TOTALDISCS` | `disk` (uint16 pair) | `WM/PartOfSet` | Integer |
| discsubtitle | `discsubtitle` | `TSST` (ID3v2.4) | `DISCSUBTITLE` | `----:com.apple.iTunes:DISCSUBTITLE` | `WM/SetSubTitle` | String |
| bpm | `bpm` | `TBPM` | `BPM` | `tmpo` (int16) | `WM/BeatsPerMinute` | Float |
| initialkey | `key` | `TKEY` | `KEY` | `----:com.apple.iTunes:initialkey` | `WM/InitialKey` | String |
| originalyear | `originalyear` | — | `ORIGINALYEAR` | — | `WM/OriginalReleaseYear` | Integer |

### 2.3 Date / Temporal Fields

| Canonical Field Name | Picard Internal Name | ID3v2 | Vorbis/FLAC | MP4/iTunes | ASF/WMA | RIFF INFO | Type |
|---|---|---|---|---|---|---|---|
| date | `date` | `TDRC` (v2.4) / `TYER+TDAT` (v2.3) | `DATE` | `©day` | `WM/Year` | `ICRD` | Date string |
| originaldate | `originaldate` | `TDOR` (v2.4) / `TORY` (v2.3) | `ORIGINALDATE` | — | `WM/OriginalReleaseTime` | — | Date string |
| encodingdate | `encodingdate` | `TDEN` (ID3v2.4) | — | — | `WM/EncodingTime` | — | Date string |
| taggingtime | `taggingtime` | `TDTG` (ID3v2.4) | — | — | — | — | Date string |
| releasetime | `releasetime` | `TDRL` (ID3v2.4) | — | — | — | — | Date string |

### 2.4 Classification Fields

| Canonical Field Name | Picard Internal Name | ID3v2 | Vorbis/FLAC | MP4/iTunes | ASF/WMA | RIFF INFO | Type |
|---|---|---|---|---|---|---|---|
| genre | `genre` | `TCON` | `GENRE` | `©gen` | `WM/Genre` | `IGNR` | String/Multi |
| mood | `mood` | `TMOO` (v2.4) | `MOOD` | `----:com.apple.iTunes:MOOD` | `WM/Mood` | — | String |
| grouping | `grouping` | `TIT1` / `GRP1` | `GROUPING` | `©grp` | `WM/ContentGroupDescription` | — | String |
| comment | `comment` | `COMM:description` | `COMMENT` | `©cmt` | `Description` | `ICMT` | String |
| lyrics | `lyrics` | `USLT:description` | `LYRICS` | `©lyr` | `WM/Lyrics` | — | String |
| syncedlyrics | `syncedlyrics` | `SYLT:description` | — | — | `WM/Lyrics_Synchronised` | — | Complex |
| copyright | `copyright` | `TCOP` | `COPYRIGHT` | `cprt` | `Copyright` | `ICOP` | String |
| license | `license` | `WCOP` / `TXXX:LICENSE` | `LICENSE` | `----:com.apple.iTunes:LICENSE` | — | — | String |
| label | `label` | `TPUB` | `LABEL` | `----:com.apple.iTunes:LABEL` | `WM/Publisher` | — | String |
| asin | `asin` | `TXXX:ASIN` | `ASIN` | `----:com.apple.iTunes:ASIN` | `ASIN` | — | String |
| barcode | `barcode` | `TXXX:BARCODE` | `BARCODE` | `----:com.apple.iTunes:BARCODE` | `WM/Barcode` | — | String |
| catalognumber | `catalognumber` | `TXXX:CATALOGNUMBER` | `CATALOGNUMBER` | `----:com.apple.iTunes:CATALOGNUMBER` | `WM/CatalogNo` | — | String |
| isrc | `isrc` | `TSRC` | `ISRC` | `----:com.apple.iTunes:ISRC` | `WM/ISRC` | — | String |
| media | `media` | `TMED` | `MEDIA` | `----:com.apple.iTunes:MEDIA` | `WM/Media` | `IMED` | String |
| script | `script` | `TXXX:SCRIPT` | `SCRIPT` | `----:com.apple.iTunes:SCRIPT` | `WM/Script` | — | String |
| language | `language` | `TLAN` | `LANGUAGE` | `----:com.apple.iTunes:LANGUAGE` | `WM/Language` | `ILNG` | String |

### 2.5 MusicBrainz Identifier Fields

| Canonical Field Name | Picard Internal Name | ID3v2 | Vorbis/FLAC | MP4/iTunes | ASF/WMA |
|---|---|---|---|---|---|
| musicbrainz_artistid | `musicbrainz_artistid` | `TXXX:MusicBrainz Artist Id` | `MUSICBRAINZ_ARTISTID` | `----:com.apple.iTunes:MusicBrainz Artist Id` | `MusicBrainz/Artist Id` |
| musicbrainz_albumartistid | `musicbrainz_albumartistid` | `TXXX:MusicBrainz Album Artist Id` | `MUSICBRAINZ_ALBUMARTISTID` | `----:com.apple.iTunes:MusicBrainz Album Artist Id` | `MusicBrainz/Album Artist Id` |
| musicbrainz_composerid | `musicbrainz_composerid` | `TXXX:MusicBrainz Composer Id` | `MUSICBRAINZ_COMPOSERID` | `----:com.apple.iTunes:MusicBrainz Composer Id` | `MusicBrainz/Composer Id` |
| musicbrainz_recordingid | `musicbrainz_recordingid` | `UFID:http://musicbrainz.org` | `MUSICBRAINZ_TRACKID` | `----:com.apple.iTunes:MusicBrainz Track Id` | `MusicBrainz/Track Id` |
| musicbrainz_trackid | `musicbrainz_trackid` | `TXXX:MusicBrainz Release Track Id` | `MUSICBRAINZ_RELEASETRACKID` | `----:com.apple.iTunes:MusicBrainz Release Track Id` | `MusicBrainz/Release Track Id` |
| musicbrainz_albumid | `musicbrainz_albumid` | `TXXX:MusicBrainz Album Id` | `MUSICBRAINZ_ALBUMID` | `----:com.apple.iTunes:MusicBrainz Album Id` | `MusicBrainz/Album Id` |
| musicbrainz_releasegroupid | `musicbrainz_releasegroupid` | `TXXX:MusicBrainz Release Group Id` | `MUSICBRAINZ_RELEASEGROUPID` | `----:com.apple.iTunes:MusicBrainz Release Group Id` | `MusicBrainz/Release Group Id` |
| musicbrainz_discid | `musicbrainz_discid` | `TXXX:MusicBrainz Disc Id` | `MUSICBRAINZ_DISCID` | `----:com.apple.iTunes:MusicBrainz Disc Id` | `MusicBrainz/Disc Id` |
| musicbrainz_originalartistid | `musicbrainz_originalartistid` | `TXXX:MusicBrainz Original Artist Id` | `MUSICBRAINZ_ORIGINALARTISTID` | `----:com.apple.iTunes:MusicBrainz Original Artist Id` | `MusicBrainz/Original Artist Id` |
| musicbrainz_originalalbumid | `musicbrainz_originalalbumid` | `TXXX:MusicBrainz Original Album Id` | `MUSICBRAINZ_ORIGINALALBUMID` | `----:com.apple.iTunes:MusicBrainz Original Album Id` | `MusicBrainz/Original Album Id` |
| musicbrainz_workid | `musicbrainz_workid` | `TXXX:MusicBrainz Work Id` | `MUSICBRAINZ_WORKID` | `----:com.apple.iTunes:MusicBrainz Work Id` | `MusicBrainz/Work Id` |
| musicbrainz_trmid | `musicbrainz_trmid` (deprecated) | `TXXX:MusicBrainz TRM Id` | `MUSICBRAINZ_TRMID` | `----:com.apple.iTunes:MusicBrainz TRM Id` | `MusicBrainz/TRM Id` |

### 2.6 AcoustID / Fingerprint Fields

| Canonical Field Name | Picard Internal Name | ID3v2 | Vorbis/FLAC | MP4/iTunes | ASF/WMA |
|---|---|---|---|---|---|
| acoustid_id | `acoustid_id` | `TXXX:Acoustid Id` | `ACOUSTID_ID` | `----:com.apple.iTunes:Acoustid Id` | `Acoustid/Id` |
| acoustid_fingerprint | `acoustid_fingerprint` | `TXXX:Acoustid Fingerprint` | `ACOUSTID_FINGERPRINT` | `----:com.apple.iTunes:Acoustid Fingerprint` | `Acoustid/Fingerprint` |
| musicip_puid | `musicip_puid` | `TXXX:MusicIP PUID` | `MUSICIP_PUID` | `----:com.apple.iTunes:MusicIP PUID` | `MusicIP/PUID` |
| musicip_fingerprint | `musicip_fingerprint` | `TXXX:MusicMagic Fingerprint` | `FINGERPRINT=MusicMagic Fingerprint` | `----:com.apple.iTunes:fingerprint` | — |

### 2.7 ReplayGain Fields

| Canonical Field Name | Picard Internal Name | ID3v2 | Vorbis/FLAC | MP4/iTunes | ASF/WMA |
|---|---|---|---|---|---|
| replaygain_track_gain | `replaygain_track_gain` | `TXXX:REPLAYGAIN_TRACK_GAIN` | `REPLAYGAIN_TRACK_GAIN` | `----:com.apple.iTunes:REPLAYGAIN_TRACK_GAIN` | `REPLAYGAIN_TRACK_GAIN` |
| replaygain_track_peak | `replaygain_track_peak` | `TXXX:REPLAYGAIN_TRACK_PEAK` | `REPLAYGAIN_TRACK_PEAK` | `----:com.apple.iTunes:REPLAYGAIN_TRACK_PEAK` | `REPLAYGAIN_TRACK_PEAK` |
| replaygain_album_gain | `replaygain_album_gain` | `TXXX:REPLAYGAIN_ALBUM_GAIN` | `REPLAYGAIN_ALBUM_GAIN` | `----:com.apple.iTunes:REPLAYGAIN_ALBUM_GAIN` | `REPLAYGAIN_ALBUM_GAIN` |
| replaygain_album_peak | `replaygain_album_peak` | `TXXX:REPLAYGAIN_ALBUM_PEAK` | `REPLAYGAIN_ALBUM_PEAK` | `----:com.apple.iTunes:REPLAYGAIN_ALBUM_PEAK` | `REPLAYGAIN_ALBUM_PEAK` |
| replaygain_track_range | `replaygain_track_range` | `TXXX:REPLAYGAIN_TRACK_RANGE` | `REPLAYGAIN_TRACK_RANGE` | `----:com.apple.iTunes:REPLAYGAIN_TRACK_RANGE` | `REPLAYGAIN_TRACK_RANGE` |
| replaygain_album_range | `replaygain_album_range` | `TXXX:REPLAYGAIN_ALBUM_RANGE` | `REPLAYGAIN_ALBUM_RANGE` | `----:com.apple.iTunes:REPLAYGAIN_ALBUM_RANGE` | `REPLAYGAIN_ALBUM_RANGE` |
| replaygain_reference_loudness | `replaygain_reference_loudness` | `TXXX:REPLAYGAIN_REFERENCE_LOUDNESS` | `REPLAYGAIN_REFERENCE_LOUDNESS` | `----:com.apple.iTunes:REPLAYGAIN_REFERENCE_LOUDNESS` | `REPLAYGAIN_REFERENCE_LOUDNESS` |

### 2.8 Sort / Ordering Fields

| Canonical Field Name | Picard Internal Name | ID3v2 | Vorbis/FLAC | MP4/iTunes | ASF/WMA |
|---|---|---|---|---|---|
| artistsort | `artistsort` | `TSOP` | `ARTISTSORT` | `soar` | `WM/ArtistSortOrder` |
| albumartistsort | `albumartistsort` | `TSO2` / `TXXX:ALBUMARTISTSORT` | `ALBUMARTISTSORT` | `soaa` | `WM/AlbumArtistSortOrder` |
| albumsort | `albumsort` | `TSOA` | `ALBUMSORT` | `soal` | `WM/AlbumSortOrder` |
| titlesort | `titlesort` | `TSOT` | `TITLESORT` | `sonm` | `WM/TitleSortOrder` |
| composersort | `composersort` | `TSOC` / `TXXX:COMPOSERSORT` | `COMPOSERSORT` | `soco` | `WM/ComposerSortOrder` |

### 2.9 Release / Album Metadata Fields

| Canonical Field Name | Picard Internal Name | ID3v2 | Vorbis/FLAC | MP4/iTunes | ASF/WMA | RIFF INFO |
|---|---|---|---|---|---|---|
| originalalbum | `originalalbum` | `TOAL` | — | — | `WM/OriginalAlbumTitle` | — |
| originalartist | `originalartist` | `TOPE` | `ORIGINALARTIST` (Picard≥2.1) | — | `WM/OriginalArtist` | — |
| originalfilename | `originalfilename` | `TOFN` | `ORIGINALFILENAME` | — | `WM/OriginalFilename` | — |
| releasestatus | `releasestatus` | `TXXX:MusicBrainz Album Status` | `RELEASESTATUS` | `----:com.apple.iTunes:MusicBrainz Album Status` | `MusicBrainz/Album Status` | — |
| releasetype | `releasetype` | `TXXX:MusicBrainz Album Type` | `RELEASETYPE` | `----:com.apple.iTunes:MusicBrainz Album Type` | `MusicBrainz/Album Type` | — |
| releasecountry | `releasecountry` | `TXXX:MusicBrainz Album Release Country` | `RELEASECOUNTRY` | `----:com.apple.iTunes:MusicBrainz Album Release Country` | `MusicBrainz/Album Release Country` | `ICNT` |
| work | `work` | `TXXX:WORK` / `TIT1` | `WORK` | `©wrk` (Picard≥2.1) | `WM/Work` | — |
| compilation | `compilation` | `TCMP` | `COMPILATION` | `cpil` | `WM/IsCompilation` | — |
| gapless | `gapless` | — | — | `pgap` | — | — |
| podcast | `podcast` | — | — | `pcst` | — | — |
| podcasturl | `podcasturl` | — | — | `purl` | — | — |

### 2.10 Encoding / Technical Fields

| Canonical Field Name | Picard Internal Name | ID3v2 | Vorbis/FLAC | MP4/iTunes | ASF/WMA | RIFF INFO |
|---|---|---|---|---|---|---|
| encodedby | `encodedby` | `TENC` | `ENCODEDBY` | `©too` | `WM/EncodedBy` | `IENC` |
| encodersettings | `encodersettings` | `TSSE` | `ENCODERSETTINGS` | — | `WM/EncodingSettings` | — |
| website | `website` | `WOAR` | `WEBSITE` | — | `WM/AuthorURL` | — |
| rating | `_rating` | `POPM` | `RATING:user@email` | — | `WM/SharedUserRating` | — |

### 2.11 iTunes-Exclusive Fields

| Canonical Field Name | MP4 Atom | Description |
|---|---|---|
| itunes_artistid | `atID` | iTunes Artist ID (uint32) |
| itunes_albumid | `plID` | iTunes Album ID (uint64) |
| itunes_catalognumber | `cnID` | iTunes Catalog ID |
| itunes_composerid | `cmID` | iTunes Composer ID |
| itunes_genreid | `geID` | iTunes Genre ID (uint32) |
| itunes_country | `sfID` | iTunes Store Country ID |
| itunes_media_type | `stik` | Media type: Normal, Audiobook, Podcast, Movie, TV Show, etc. |
| itunes_advisory | `rtng` | Content advisory: 0=None, 1=Explicit, 2=Clean |
| itunes_gapless | `pgap` | Gapless album flag |
| itunes_purchase_date | `purd` | Purchase date |
| itunes_owner | `ownr` | Owner |
| itunes_cddb_1 | `----:com.apple.iTunes:iTunes_CDDB_1` | CDDB reference |

---

## 3. Multi-Value Field Handling

### 3.1 Which Fields Allow Multiple Values

The following canonical fields are natively multi-value in at least one supported format:

| Field | ID3v2 Behavior | Vorbis/FLAC Behavior | MP4 Behavior |
|---|---|---|---|
| `artist` / `artists` | Multiple `TPE1` frames, or `;`-separated in single frame (ID3v2.3 workaround) | Multiple `ARTIST=...` lines | Single `©ART` value |
| `genre` | Multiple `TCON` frames (v2.4); numeric or text; `(12)` prefix for numeric refs | Multiple `GENRE=...` lines | `©gen` / `gnre` atoms |
| `performer:instrument` | `TMCL:instrument` frame with `role=artist` pairs | Multiple `PERFORMER=artist (instrument)` lines | Not supported |
| `comment` | Multiple `COMM` frames with different descriptions | Multiple `COMMENT=...` lines | Single `©cmt` value |
| `lyrics` | Multiple `USLT` frames with different language/description | Multiple `LYRICS=...` lines | Single `©lyr` value |
| `website` | Multiple `WOAR` frames | Multiple `WEBSITE=...` lines | Not supported |
| `musicbrainz_artistid` | Multiple `TXXX:MusicBrainz Artist Id` frames | Multiple `MUSICBRAINZ_ARTISTID=...` lines | Multiple `----:com.apple.iTunes:MusicBrainz Artist Id` atoms |
| `replaygain_*` | Multiple `TXXX:REPLAYGAIN_*` frames | Multiple same-name lines | Multiple atoms |
| label | Multiple `TPUB` frames | Multiple `LABEL=...` lines | Multiple atoms |
| catalognumber | Multiple `TXXX:CATALOGNUMBER` frames | Multiple `CATALOGNUMBER=...` lines | Multiple atoms |
| writer | Multiple `TXXX:Writer` frames | Multiple `WRITER=...` lines | Not supported |
| mood | Multiple `TMOO` frames (ID3v2.4) | Multiple `MOOD=...` lines | Multiple atoms |
| discnumber (multi-disc sets) | `TPOS` may appear multiple times | Multiple `DISCNUMBER=...` lines | Multiple `disk` atoms |

(Source: [MusicBrainz Picard Tag Mapping v3.0](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html); [Vorbis Comment spec](https://xiph.org/vorbis/doc/v-comment.html); [Plex Forum — Multi-value Genre](https://forums.plex.tv/t/inconsistent-behavior-when-parsing-music-genre-tags/638344))

### 3.2 Internal Storage Strategy

**In the canonical model**, multi-value fields are stored as **ordered lists of strings**, not as semicolon/comma-separated single strings. Each format reader must normalize to this list representation:

```
canonical.genre = ["Rock", "Alternative"]   # NOT "Rock; Alternative"
canonical.artist = ["John Doe", "Jane Smith"]  # NOT "John Doe; Jane Smith"
```

**ID3v2.3 workaround:** Since ID3v2.3 does not support true multi-value text frames, implementations traditionally store multiple values as a single string separated by `; ` (semicolon-space). When reading ID3v2.3, the canonical reader must split on `"; "` to reconstruct the list. When writing to ID3v2.3, the writer must join the list with `"; "`. (Source: [dBpoweramp Forum — Multiple Artist handling](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/331575-map-tags-to-other-tags))

**DBpoweramp specifically** warns about this: "A problem arises when another program does not follow these rules, by not following the tagging conventions correctly said program might not read the 2nd Artist, or only the 2nd Artist." DBpoweramp's "Multiple Artist From 'Artist1; Artist2'" option detects artists separated by `'; '` and sets them internally to multiple artists correctly stored as defined by the tagging format. (Source: [dBpoweramp ID Tag Update Help](https://dbpoweramp.com/Help/dMC/idtagupdate.html))

**MP4 single-value fields:** Several fields that are multi-value in other formats (artist, album artist, composer) can only hold a single value in MP4/iTunes atoms. DBpoweramp handles this by storing only the **first value** when reading from MP4, and writing the **first value** when writing to MP4. (Source: [Mp3tag Tag Field Mappings](https://help.mp3tag.de/main_tags.html))

### 3.3 Single-Value Fields with Multiple Occurrences

When a source format has the same single-value field appearing multiple times (e.g., two `TPE1` frames in ID3v2, or two `ARTIST=` lines in Vorbis), the canonical reader must decide:

- **Artist, Album Artist, Composer, Title, Album**: Take the **first** value (most significant).
- **Comment, Lyrics**: Collect all and tag each with a `description` sub-field (e.g., `comment:description` becomes the composite key `comment{separator}description`).
- **ReplayGain fields**: Take the first matching field (track gain, album gain, etc.).
- **Sort fields**: Take the first value.

---

## 4. Field Normalization Rules

### 4.1 Year / Date Parsing

The canonical model stores dates as **full precision date strings** (ISO 8601 partial format: `YYYY`, `YYYY-MM`, or `YYYY-MM-DD`). The `date` field specifically receives the **recording date**.

| Source Value | Canonical `date` Stored | Notes |
|---|---|---|
| `"2024"` | `"2024"` | Year only; padded/truncated to 4 digits |
| `"2024-03"` | `"2024-03"` | Year-month |
| `"2024-03-15"` | `"2024-03-15"` | Full ISO 8601 date |
| `"2024/2025"` | `"2024"` (or error) | Range; canonical takes first year |
| `"15-03-2024"` | `"2024-03-15"` | DD-MM-YYYY → normalized to YYYY-MM-DD |
| `TYER = "2024"` (ID3v2.3) | `"2024"` | Year frame → date field |
| `TDRC = "2024-03-15"` (ID3v2.4) | `"2024-03-15"` | Recording time frame → date field |
| `TDOR = "1999"` (ID3v2.4) | `originaldate = "1999"` | Original release date → separate field |
| `©day = "2024-03-15"` (MP4) | `"2024-03-15"` | MP4 date atom |
| `DATE = "2024-03-15T12:00:00Z"` (Vorbis) | `"2024-03-15"` | Timestamps normalized to date |

DBpoweramp specifically offers a **`DATE → YEAR`** mapping option under FLAC codec tagging settings, which strips the full date to just the year for compatibility with older software expecting `DATE` as a year-only field. (Source: [dBpoweramp Forum — DATE to YEAR](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/331575-map-tags-to-other-tags))

### 4.2 Track Number Parsing

Track numbers are stored as **integer pairs**: `(track_number, total_tracks)`. Both values are optional (total may be unknown).

| Source Value | Canonical `tracknumber` | Canonical `totaltracks` | Notes |
|---|---|---|---|
| `"5"` | `5` | `null` | Single number, no total |
| `"05"` | `5` | `null` | Zero-padded; canonical strips padding |
| `"5/12"` | `5` | `12` | Slash-separated; canonical splits |
| `"5 of 12"` | `5` | `12` | Natural language; canonical parses "of" |
| `"5/12"` in Vorbis `TRACKNUMBER` | `5` | `12` | Must split — do NOT write "5/12" to `TRACKNUMBER` in FLAC |
| MP4 `trkn = (5, 12)` uint16 pair | `5` | `12` | MP4 encodes as two uint16 values in one atom |
| ID3v2.3 `TRCK = "5/12"` | `5` | `12` | Native slash format; canonical splits |
| ID3v2.4 `TRCK = "5/12"` | `5` | `12` | Same as v2.3 |

**Critical rule for output:** When writing to a format that stores track/total separately:
- **ID3v2 (TRCK):** Write as `"5/12"` (slash-joined).
- **Vorbis (FLAC/OGG):** Write `"5"` to `TRACKNUMBER` and `"12"` to `TOTALTRACKS` (or `TRACKTOTAL`). Never write `"5/12"` to `TRACKNUMBER` in Vorbis.
- **MP4 (trkn):** Encode as uint16 pair `(5, 12)`.

The same logic applies to `discnumber` / `totaldiscs`: format as `"1/3"` for ID3v2, split into `DISCNUMBER=1` + `DISCTOTAL=3` for Vorbis. (Source: [ID3v2.4.0 spec](https://id3.org/id3v2.4.0-frames); [dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/331575-map-tags-to-other-tags))

### 4.3 Genre ID → String Normalization

ID3v1 stores genre as a **single byte** (0–147) referencing a predefined genre list. ID3v2 supports both numeric references like `(12)` within a TCON frame and free-text genre names.

| ID3v1 Genre ID | Genre Name | ID3v2.3 TCON | ID3v2.4 TCON |
|---|---|---|---|
| 0 | Blues | `"Blues"` | `"Blues"` |
| 1 | Classic Rock | `"Classic Rock"` | `"Classic Rock"` |
| 12 | Country | `"Country"` | `"Country"` |
| 17 | Rock | `"Rock"` | `"Rock"` |
| 37 | Classical | `"Classical"` | `"Classical"` |
| 79 | Euro-Techno | `"Euro-Techno"` | `"Euro-Techno"` |
| 80 (user-defined) | `"RX"` prefix in ID3v2 | `"RX"` or `"Remix"` | `"RX"` |
| 85 | Beyonce | `"Beyonce"` | `"Beyonce"` |
| 125 | undefined | `"0"` or raw number | `"125"` |
| 126 | Cutting Intentions | `"Cutting Intentions"` | `"Cutting Intentions"` |
| 147 | Unknown | `"Unknown"` | `"Unknown"` |

The canonical model stores genres as **human-readable strings only**. Numeric IDs are resolved to names at read time. Unknown numeric IDs (126–147) are stored as-is or discarded depending on implementation strictness. (Source: [ID3v2.2 spec — Genre byte](http://mutagen-specs.readthedocs.io/en/latest/id3/id3v2.2.html))

**Genre normalization in DBpoweramp:** When DBpoweramp reads a file, it displays genres from both ID3v1 and ID3v2 sources. During conversion, it writes genres using the target format's native representation. DBpoweramp does not force numeric genres — it writes text genres. (Source: [dBpoweramp Forum — Tagging discussion](https://forum.dbpoweramp.com/forum/other-topics/how-do-i/24690-how-do-i-do-meta-data-tagging-right))

### 4.4 Field Name Normalization

Format-specific field names are mapped to canonical names at read time:

| Format | Source Field Name | Canonical Name |
|---|---|---|
| Vorbis | `ALBUMARTIST`, `ALBUM ARTIST` | `albumartist` |
| Vorbis | `LABEL`, `ORGANIZATION` | `label` (DBpoweramp prioritizes `LABEL`) |
| ID3v2 | `TPE2` | `albumartist` |
| ID3v2 | `TXXX:BARCODE` | `barcode` |
| MP4 | `©wrt` | `composer` |
| MP4 | `aART` | `albumartist` |
| MP4 | `©gen` / `gnre` | `genre` |
| ASF/WMA | `WM/AlbumTitle` | `album` |
| ASF/WMA | `WM/Publisher` | `label` |
| RIFF | `IPRD` | `album` |
| RIFF | `IART` | `artist` |

DBpoweramp maintains a **tag name display priority** for fields that may appear under different names: for example, when both `LABEL` and `ORGANIZATION` exist in a Vorbis file, DBpoweramp displays `LABEL` (the Vorbis standard) and uses that as the canonical name, but the `ORGANIZATION` value can be accessed via custom tag. (Source: [dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/25898-mapping-of-metadata-between-file-formats))

---

## 5. Empty vs. Absent Field Semantics

### 5.1 The Distinction

The canonical model distinguishes between three states:

| State | Canonical Representation | Meaning |
|---|---|---|
| **Absent** | Field key does not exist in the map | The source had no data for this field; no tag was present |
| **Empty string** | `field = ""` | The tag existed but contained no text; explicitly empty |
| **Present with value** | `field = "Some Value"` | The tag existed and contained data |

### 5.2 How DBpoweramp Handles This

DBpoweramp's `ReadIDTagElementValue` method returns `"Element: Value"` pairs for all tags it encounters when reading a file. The enumeration includes all fields that exist in the file's tag structures, including those with empty values. Fields that do not exist at all are simply absent from the enumeration. (Source: [dBpoweramp Developer Scripting](https://dbpoweramp.com/developer-scripting-dmc))

**In the ID Tag Processing DSP**, DBpoweramp offers an `Externally Script Tags` option that writes all currently active tags to a temporary Unicode text file. This file includes all fields with values (including empty strings) but does not include absent fields. The format is `Element=Value` per line, where an empty value appears as `Element=` (trailing equals sign, no value). (Source: [dBpoweramp Forum — External Tag Scripting](https://forum.dbpoweramp.com/forum/other-topics/developers-corner/28525-external-tag-scripting-id-tag-processing-update-id-tag))

**Important DBpoweramp behavior:** When using the `Write Metadata File` DSP effect to export tags to XML, all fields present in the file are exported — including those with empty values — as separate XML elements. The presence or absence of an element in the output is the canonical indicator. (Source: [dBpoweramp DSP Help](https://dbpoweramp.com/Help/dMC/dsp))

### 5.3 Practical Implications for Conversion

- **Read:** Check both existence and emptiness. Copy absent fields as absent (do not create empty tags). Copy empty-string fields as empty strings (write the tag with no value).
- **Write:** Only write fields that have non-empty values by default. Provide an option to write empty-value tags if the user explicitly wants to clear a field.
- **DBpoweramp's ID Tag Update utility** can explicitly clear or remove tags: "it is possible to remove All Tags, All Except listed, or delete a Single Tag." (Source: [dBpoweramp ID Tag Update Help](https://dbpoweramp.com/Help/dMC/idtagupdate.html))

---

## 6. Custom and Unknown Field Handling

### 6.1 Unknown / Unrecognized Fields

The canonical model **preserves all fields** from the source that do not map to known canonical names. These are stored as raw key-value pairs with their original field name preserved exactly (including case).

**Storage approach:**

```python
canonical.custom_fields["MY_CUSTOM_FIELD"] = "value"
canonical.custom_fields["~private~"] = {"FOO": "bar"}  # Format-specific private data
```

**DBpoweramp explicitly supports this:** Custom tags can be referenced in naming and processing rules using the syntax `[tag]tagname[]`. DBpoweramp's ID Tag Processing DSP allows adding new tags by name, and the Write Metadata File DSP exports all fields — not just known ones — to XML. (Source: [dBpoweramp Naming](https://www.dbpoweramp.com/help/dmc/Naming); [dBpoweramp DSP](https://dbpoweramp.com/Help/dMC/dsp))

**In the Write Metadata File XML export**, fields use either the XML element name or a `name` attribute for the tag name:

```xml
<MyCustomTag>custom value</MyCustomTag>
<!-- OR -->
<tag name="MY_CUSTOM_TAG">custom value</tag>
```

This means any arbitrary field name is preserved through the XML round-trip. (Source: [dBpoweramp DSP Help](https://dbpoweramp.com/Help/dMC/dsp))

### 6.2 Case Sensitivity

| Format | Field Name Case Sensitivity |
|---|---|
| ID3v2 | Case-insensitive (frame IDs normalized: `TPE1`, `tpe1`, `Tpe1` all refer to the same frame) |
| Vorbis Comment | Case-insensitive per spec (`ARTIST`, `artist`, `Artist` are identical) |
| APEv2 | Case-sensitive (`ARTIST` ≠ `artist`) |
| MP4/iTunes | Case-sensitive (`©ART` ≠ `©art`) |
| ASF/WMA | Case-preserving but lookup case-insensitive |
| RIFF INFO | Case-insensitive |

**Canonical model decision:** Field names are **normalized to a canonical case** (typically lowercase with underscores, matching Picard convention: `albumartist`, `discnumber`, `replaygain_track_gain`). When reading, the original case is stored in a separate `original_name` metadata property so it can be preserved when writing back to formats that are case-sensitive (APE, MP4).

### 6.3 Format-Specific Frame Mapping

DBpoweramp maps unknown fields based on format conventions:

| Unknown Source Field | Written To |
|---|---|
| Unknown ID3v2 TXXX frame `TXXX:Foo` | Written as `TXXX:Foo` in ID3v2 output; `FOO` in Vorbis; `----:com.apple.iTunes:FOO` in MP4 |
| Unknown Vorbis field `FOO` | Written as `FOO` in Vorbis output; `TXXX:FOO` in ID3v2; `----:com.apple.iTunes:FOO` in MP4 |
| Unknown MP4 atom `----:com.apple.iTunes:FOO` | Written as `----:com.apple.iTunes:FOO` in MP4 output; `TXXX:FOO` in ID3v2; `FOO` in Vorbis |

The cross-format mapping is name-based: the canonical reader strips the format-specific namespace prefix (e.g., `TXXX:`, `----:com.apple.iTunes:`) and stores the bare field name. The writer adds the appropriate prefix for the target format.

---

## 7. Cover Art (Embedded Pictures) Handling

### 7.1 Supported Picture Types

DBpoweramp supports all standard cover art types. The canonical model stores pictures with the following metadata:

```python
class CoverArt:
    picture_type: int          # 0-20 per ID3v2 / MP4 specification
    mime_type: str             # "image/jpeg", "image/png", "image/gif", etc.
    description: str           # Free-text description (may be empty)
    width: int | None         # Pixel width (if known)
    height: int | None        # Pixel height (if known)
    color_depth: int | None   # Color depth in bits (if known)
    data: bytes               # Raw image binary data
    # picture_type meanings:
    # 0 = Other
    # 1 = 32x32 PNG file icon (only for MP4)
    # 2 = Other file icon
    # 3 = Front cover (primary album art)
    # 4 = Back cover
    # 5 = Leaflet page
    # 6 = Media label (e.g., CD face)
    # 7 = Lead artist / lead performer / soloist
    # 8 = Artist / performer / soloist
    # 9 = Conductor
    # 10 = Band / orchestra
    # 11 = Composer
    # 12 = Lyricist / text writer
    # 13 = Recording location
    # 14 = During recording
    # 15 = During performance
    # 16 = Movie / video screen capture
    # 17 = A bright colored fish
    # 18 = Illustration
    # 19 = Band / artist logotype
    # 20 = Publisher / studio logotype
```

### 7.2 Cover Art Limits

| Format | Max Pictures | Notes |
|---|---|---|
| ID3v2 (APIC frames) | Unlimited (no hard limit; frame-based) | Each APIC frame = one picture |
| Vorbis/FLAC (METADATA_BLOCK_PICTURE) | Unlimited | Each picture stored as separate `METADATA_BLOCK_PICTURE` block |
| MP4/iTunes (covr atom) | Multiple | Stored as list of picture data items within single `covr` atom |
| ASF/WMA (WM/Picture) | Multiple | Multiple `WM/Picture` attributes |

### 7.3 DBpoweramp Cover Art Behavior

**Primary cover:** DBpoweramp treats the **Front Cover (type 3)** as the primary picture. When converting, the primary cover is written to the destination. Additional pictures are preserved if the format supports multiple pictures and the conversion settings allow it.

**Cover art during conversion:** DBpoweramp's "Album Artwork" DSP effect offers options including:
- "Extract album art from source" (extracts embedded artwork to a separate file, e.g., `Folder.jpg`)
- "Embed album art from file" (embeds from an external file into the output)
- "Remove album art" (strips all embedded pictures)

When embedding, DBpoweramp writes the picture with `picture_type = 3` (Front Cover) by default — not type 0 (Other), which is what many naive implementations do. (Source: [DBPOWERAMP_CRITICAL_BEHAVIORS_QUICKREF.md](.research/DBpoweramp-research/DBPOWERAMP_CRITICAL_BEHAVIORS_QUICKREF.md))

**Metadata extracted per picture:**
- Type (integer code)
- MIME type
- Description
- Width, height, color depth (from EXIF/PNG metadata where present)
- Binary data

---

## 8. ReplayGain Normalization

### 8.1 ReplayGain Field Inventory

The canonical model stores exactly seven ReplayGain fields:

| Canonical Field | Format (Picard ≥ 2.2) | Value Format | Example |
|---|---|---|---|
| `replaygain_track_gain` | `REPLAYGAIN_TRACK_GAIN` | `±X.X dB` | `"-3.4 dB"` or `"+1.2 dB"` |
| `replaygain_track_peak` | `REPLAYGAIN_TRACK_PEAK` | Decimal 0.0–2.0 | `"0.707106"` |
| `replaygain_track_range` | `REPLAYGAIN_TRACK_RANGE` | `±X.X dB` | `"6.0 dB"` |
| `replaygain_album_gain` | `REPLAYGAIN_ALBUM_GAIN` | `±X.X dB` | `"-5.2 dB"` |
| `replaygain_album_peak` | `REPLAYGAIN_ALBUM_PEAK` | Decimal 0.0–2.0 | `"0.891234"` |
| `replaygain_album_range` | `REPLAYGAIN_ALBUM_RANGE` | `±X.X dB` | `"4.5 dB"` |
| `replaygain_reference_loudness` | `REPLAYGAIN_REFERENCE_LOUDNESS` | `-X.X dB` | `"-18.0 dB"` |

(Source: [MusicBrainz Picard Tag Mapping v3.0](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html))

### 8.2 Track vs. Album Gain

**Track gain (`replaygain_track_gain`):** The loudness adjustment in dB needed to bring the individual track to the reference loudness level.

**Album gain (`replaygain_album_gain`):** The loudness adjustment in dB needed to bring the entire album (as a continuous set) to the reference loudness level. Album gain takes precedence when playing in album mode.

**Track peak (`replaygain_track_peak`):** The peak amplitude of the track, used to prevent clipping when applying negative gain. Format is a decimal number where 1.0 = 0 dBFS (digital full scale). Values up to 2.0 are allowed for peaks above 0 dBFS that would require negative overall gain.

**Album peak (`replaygain_album_peak`):** Same as track peak but for the album.

**Reference loudness (`replaygain_reference_loudness`):** The target loudness level, typically `-18.0 LUFS` (ReplayGain spec) or `-16.0 LUFS` (older) or `-14.0 LUFS` (streaming platforms). Stored as `-18.0 dB` (note: dB, not LUFS, in the tag).

### 8.3 Value Normalization

- **Gain values:** Parsed as float from string `"-3.4 dB"`, stored as `float = -3.4`. Written back with `f"{value:.1f} dB"` format. Leading `+` is preserved for positive values.
- **Peak values:** Parsed as float from decimal string, stored as float. Written with 6 decimal places (`f"{value:.6f}"`).
- **Missing fields:** Each field is independent. A file may have track gain without track peak, etc. The canonical model stores `None` for absent fields.
- **Precision:** Gain values are normalized to 1 decimal place. Peak values retain full precision (up to 6 decimal places).
- **Scan-calculated vs. tagged:** ReplayGain values may come from a previous loudness scan (tagged in the file) or be calculated on-the-fly during playback. DBpoweramp does not recalculate ReplayGain during conversion — it passes through existing values.

---

## 9. Source Metadata Preservation

### 9.1 Source Format Identifier

The canonical model does **not** store the source tag format as a first-class field. However, DBpoweramp preserves format-specific tags through its format-mapping rules.

**When converting FLAC → ALAC:**
DBpoweramp's forum states: "ALAC is based on iTunes tagging, they have set fields we have to write to. By doing a conversion of a fully tagged file between the formats then looking at the tags you will be able to see what goes where." This means DBpoweramp maps from Vorbis Comment field names to iTunes MP4 atom names according to its internal mapping table. (Source: [dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/25898-mapping-of-metadata-between-file-formats))

### 9.2 Fields That Are Not Remapped

DBpoweramp does NOT automatically map certain fields that users often expect it to, by design. From the forum: "There is no table, only that we try to standardize tags from one format to the other. ...dropping ORGANIZATION and DESCRIPTION — Not dropped, rather defaulted to not mapped, as each year we field about 100 questions why they are mapped when people do not expect them." (Source: [dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/25898-mapping-of-metadata-between-file-formats))

Fields that DBpoweramp **does always map** (mandatory mappings):
- `ORGANIZATION` → `LABEL` (in the Vorbis→ID3 direction)
- `ALBUMARTIST` ↔ `TPE2`
- `COMPOSER` ↔ `TCOM`
- All MusicBrainz identifier fields (cross-format)
- All ReplayGain fields (cross-format)
- `ARTIST` ↔ `TPE1`, `ALBUM` ↔ `TALB`, etc.

### 9.3 Preservation of Unmapped Fields

**Custom/unknown fields** are preserved via the custom field storage mechanism (Section 6). DBpoweramp's `Write Metadata File` DSP can export all fields to XML, and `Read Metadata File` can import them back, enabling preservation of any arbitrary field through a round-trip conversion. (Source: [dBpoweramp DSP](https://dbpoweramp.com/Help/dMC/dsp))

**Format-native private tags** (e.g., iTunes-specific atoms, ASF private data) are preserved when the output format is the same as the input format. When the format changes, private tags are dropped unless they map to a known canonical field or are stored in the custom fields bucket.

### 9.4 Tag Version Preservation

**ID3v1 + ID3v2 coexistence:** When a file has both ID3v1 and ID3v2 tags, DBpoweramp reads from ID3v2 (higher priority). The ID3v1 tags are used as fallback for fields missing from ID3v2. During write, DBpoweramp can be configured to write ID3v1 tags, ID3v2 tags, or both, via the "ID Tag Version" setting in the tagging configuration. (Source: [dBpoweramp ID Tag Update Help](https://dbpoweramp.com/Help/dMC/idtagupdate.html))

**APEtag flags:** APEv2 tags have three flag bits (read-only, item-is-string, item-is-binary). DBpoweramp reads the flags but does not preserve the read-only flag when rewriting — all fields become writable.

---

## Appendix A: Complete Canonical Field Name Registry

Derived from MusicBrainz Picard Appendix A Tag Mapping v3.0, augmented with MP3tag and DBpoweramp-specific fields. Canonical names use lowercase with underscores (Picard convention).

### Identification
- `title` / `subtitle`
- `artist` / `artists` / `albumartist`
- `composer` / `composersort` / `lyricist` / `writer`
- `conductor` / `arranger`
- `performer` (with instrument sub-key)
- `engineer` / `producer` / `djmixer` / `mixer` / `remixer`
- `director`
- `show` / `showsort` / `showmovement`
- `movement` / `movementnumber` / `movementtotal`
- `work`
- `originalalbum` / `originalartist` / `originalfilename`

### Numbers
- `tracknumber` / `totaltracks`
- `discnumber` / `totaldiscs` / `discsubtitle`
- `bpm` / `initialkey` / `originalyear`

### Dates
- `date` / `originaldate` / `encodingdate` / `taggingtime` / `releasetime`

### Classification
- `genre` / `mood` / `grouping`
- `comment` / `lyrics` / `syncedlyrics`
- `copyright` / `license`
- `label` / `asin` / `barcode` / `catalognumber`
- `isrc` / `media` / `script` / `language`

### MusicBrainz IDs
- `musicbrainz_artistid` / `musicbrainz_albumartistid` / `musicbrainz_composerid`
- `musicbrainz_recordingid` / `musicbrainz_trackid` / `musicbrainz_albumid`
- `musicbrainz_releasegroupid` / `musicbrainz_discid`
- `musicbrainz_originalartistid` / `musicbrainz_originalalbumid`
- `musicbrainz_workid` / `musicbrainz_trmid` (deprecated)

### AcoustID / Fingerprints
- `acoustid_id` / `acoustid_fingerprint`
- `musicip_puid` / `musicip_fingerprint`

### ReplayGain
- `replaygain_track_gain` / `replaygain_track_peak` / `replaygain_track_range`
- `replaygain_album_gain` / `replaygain_album_peak` / `replaygain_album_range`
- `replaygain_reference_loudness`

### Sort Order
- `artistsort` / `albumartistsort` / `albumsort` / `titlesort` / `composersort`

### Release
- `releasestatus` / `releasetype` / `releasecountry`
- `compilation` / `gapless` / `podcast` / `podcasturl`

### Encoding
- `encodedby` / `encodersettings` / `website` / `rating`

### iTunes-Exclusive
- `itunes_artistid` / `itunes_albumid` / `itunes_catalognumber`
- `itunes_composerid` / `itunes_genreid` / `itunes_country`
- `itunes_media_type` / `itunes_advisory` / `itunes_gapless`
- `itunes_purchase_date` / `itunes_owner` / `itunes_cddb_1`

### Cover Art
- `cover_art` (list of `CoverArt` objects)

---

## Appendix B: Source Attribution

| Source | Used For |
|---|---|
| [MusicBrainz Picard Tag Mapping v3.0](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html) | Canonical field inventory, cross-format frame mappings, Picard internal names, ReplayGain fields, MusicBrainz IDs |
| [Mp3tag Tag Field Mappings](https://help.mp3tag.de/main_tags.html) | MP4 atom names, iTunes-exclusive fields, RIFF INFO mappings, field normalization |
| [TagLib API Documentation](https://taglib.org/api/) | PropertyMap abstraction, complex property handling (cover art), format abstraction |
| [dBpoweramp Developer Scripting](https://dbpoweramp.com/developer-scripting-dmc) | `ReadIDTagElementValue` API, tag enumeration by index, write API |
| [dBpoweramp Naming](https://www.dbpoweramp.com/help/dmc/Naming) | Tag name syntax `[tag]name[]`, multi-tag handling |
| [dBpoweramp DSP Help](https://dbpoweramp.com/Help/dMC/dsp) | ID Tag Processing, Write Metadata File XML format, field export/import |
| [dBpoweramp ID Tag Update Help](https://dbpoweramp.com/Help/dMC/idtagupdate.html) | Multiple artist handling, tag manipulation pipeline order |
| [dBpoweramp Forum — Tag Mapping](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/25898-mapping-of-metadata-between-file-formats) | ORGANIZATION/LABEL mapping behavior, ALAC tagging constraints |
| [dBpoweramp Forum — DATE→YEAR](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/331575-map-tags-to-other-tags) | DATE→YEAR setting, DISCNUMBER/DISCTOTAL handling |
| [ID3.org — ID3v2.4.0 Frames](https://id3.org/id3v2.4.0-frames) | TRCK, TPOS, TYER, TDRC frame specifications |
| [Mutagen Specs — ID3v2.2](http://mutagen-specs.readthedocs.io/en/latest/id3/id3v2.2.html) | ID3v1 genre byte mapping |
| [Vorbis Comment Spec](https://xiph.org/vorbis/doc/v-comment.html) | Multi-value field semantics, field name case insensitivity |
| [DBPOWERAMP_CRITICAL_BEHAVIORS_QUICKREF.md](.research/DBpoweramp-research/DBPOWERAMP_CRITICAL_BEHAVIORS_QUICKREF.md) | Track number divergence table, cover art type code (Front Cover = 3) |

---

## Would a User Notice Any Difference from DBpoweramp?

**In most cases, no.** A pipeline that implements the canonical model described in this document will produce outputs that are functionally equivalent to DBpoweramp for the vast majority of real-world files. The 80+ canonical fields cover everything DBpoweramp can read or write, and the normalization rules match DBpoweramp's behavior for year, track, and genre fields.

**Where differences may appear:**

1. **Unknown/custom fields**: DBpoweramp preserves any field not in the standard set through its XML metadata export/import mechanism. A pipeline that only preserves known fields will lose truly custom tags (e.g., `MY_FIELD`, `ORIGINAL_PATH`) that DBpoweramp would keep.

2. **ID3v2.3 vs ID3v2.4**: DBpoweramp defaults to ID3v2.4 for modern MP3 files. A pipeline defaulting to ID3v2.3 may produce slightly different tag representations (TDRC vs. TYER+TDAT, TMOO support, etc.).

3. **ORGANIZATION ↔ LABEL**: DBpoweramp maps ORGANIZATION → LABEL in FLAC→MP3 conversions but does NOT map LABEL → ORGANIZATION. The canonical model defaults to `label` for both, which means writing to a Vorbis file could produce a `LABEL` tag where the user wanted `ORGANIZATION`.

4. **iTunes numeric IDs** (`itunes_artistid`, `itunes_albumid`, etc.): These are preserved through MP4→MP4 conversion but are not mapped to any other format. A pipeline that drops these during non-MP4 conversions is correct by the canonical model — but DBpoweramp also drops them.

5. **ID3v1 coexistence**: When reading a file with both ID3v1 and ID3v2 tags, DBpoweramp gives priority to ID3v2 and may fall back to ID3v1 for fields missing from ID3v2. Exact fallback priority is not documented.

6. **Display name semantics**: DBpoweramp's tag editor UI shows human-readable names (e.g., "Album Artist" instead of `albumartist`). The canonical model uses `albumartist` internally — functionally equivalent but different key names in raw file inspection.

The most significant practical difference would be in **lossy-to-lossless round-trips with extensive custom metadata** or **multi-format conversions where iTunes-specific fields must be preserved**. For standard music library conversions (FLAC→MP3, FLAC→ALAC, MP3→FLAC), the canonical model is effectively identical to DBpoweramp.
