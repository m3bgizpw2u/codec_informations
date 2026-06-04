# DBpoweramp Tag Writing Pipeline: Full Format Coverage

*Generated: 2026-06-04 | Sources: 25+ | Confidence: High*

---

## 1. Overview & Purpose

This document exhaustively covers how DBpoweramp Music Converter writes metadata tags to every supported audio format. It is based on analysis of DBpoweramp's official help documentation, version changelogs, developer forum posts (DBpoweramp forum), and cross-referenced tag-format specifications (ID3.org, Xiph.org, RFC 9639).

DBpoweramp's approach is characterized by:

- **Format-native tagging**: Each output format uses its canonical tag system, never a foreign one (e.g., FLAC always gets Vorbis Comments, never ID3v2 as a primary).
- **User-configurable global defaults**: Tag creation settings are centralized in **dBpoweramp Configuration → Codecs → Advanced Options**, applied consistently across CD Ripper, Batch Converter, and command-line (`CoreConverter.exe`).
- **Clean-slate writes**: The **ID Tag Update** utility codec always removes existing tags and rewrites fresh ones using the current global configuration. It does not edit in-place.
- **Safe atomic writes**: CD Ripper writes to `.tmp` files first, then renames to the final extension only after all encoding and tagging are complete. This prevents incomplete files from being indexed by media players.
- **One active tag type per format at write time**: DBpoweramp does not write multiple independent tag systems to the same file simultaneously (except MP3, which has a configurable `ID3v2 & ID3v1` option).

---

## 2. Per-Format Tag System Selection

### 2.1 Complete Format Table

| Format | Primary Written | Secondary / Compatibility | Unicode | Album Art | Notes |
|---|---|---|---|---|---|
| **MP3** | **ID3v2** (default) | ID3v1, APEv2 (optional) | Yes (ID3v2) | Yes — APIC frame | Tag Creation is user-configurable per codec |
| **FLAC** | **Vorbis Comments** | ID3v2 / ID3v1 for compatibility only | Yes | Yes — METADATA_BLOCK_PICTURE (preferred) | ID3v2 written only if configured; Vorbis Comments is default/recommended |
| **OGG Vorbis** | **Vorbis Comments** | None | Yes | Yes — METADATA_BLOCK_PICTURE | ID3v2 not applicable |
| **Opus** | **Vorbis Comments** (in Ogg container) | None | Yes | Yes — METADATA_BLOCK_PICTURE | Standard Ogg Opus tagging |
| **AAC / M4A** | **iTunes MP4 atoms** | ID3v1 (Sony/Nokia compatibility only) | Yes | Yes — `covr` atom | ALAC shares same container/atoms |
| **WAV** | **RIFF LIST INFO** + **ID3v2 chunk** | ID3v1 (legacy) | Partial (RIFF) / Yes (ID3v2) | Yes (ID3v2 APIC) | Both systems written by default; ID3v2 is the full-featured layer |
| **AIFF** | **ID3v2 chunk** (iTunes-compatible) | None | Yes | Yes — APIC frame | Written inside `ID3 ` chunk in AIFF container |
| **WMA / ASF** | **ASF / Windows Media attributes** | None | Yes | Yes — embedded | Native ASF metadata |
| **WavPack WV** | **APEv2** | ID3v1 (read-only legacy) | Yes | Yes — APEv2 | ID3v1 not written |
| **APE (Monkeys Audio)** | **APEv2** (default) | APEv1 | Yes | Yes — APEv2 | APEv2 is native; APE was invented for this format |
| **TAK** | **Vorbis Comments** (native TAK tags) | Unknown | Yes | Yes | TAK has a proprietary tag system; DBpoweramp R2023+ added TAK tag editing |
| **TTA (True Audio)** | **APEv2** or **ID3v2** (per-container) | Unknown | Yes | Yes | TTA is a native container; APEv2 tags written at end of file |
| **MusePack MPC** | **APEv2** | ID3v1 (legacy) | Yes | Yes — APEv2 | SV8 format; R2023+ added ReplayGain info saving |

### 2.2 Key Behavioral Details Per Format

#### MP3
DBpoweramp's MP3 codec is the most configurable. The default `Tag Creation` setting is **ID3v2 only**. Users can also choose:
- `ID3v1` (v1.0 or v1.1)
- `ID3v2 & ID3v1`
- `APEv2` (rarely used)
- `APEv2 & ID3v1`

APEv2 and ID3v2 **cannot both be written simultaneously** to MP3. The DBpoweramp developer explicitly stated: *"we have never allowed our programs to tag ID3v2 and APE at the same time, really speaking, APE should no longer be used for mp3 files."* ([DBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/27227-mp3-with-id3v1-id3v2-3-and-apev2))

ID3v2 text encoding (for the text frames) is configurable as either **UTF-16** (default for Unicode support) or **ANSI/ISO-8859-1**. This is set in `dBpoweramp Configuration → Codecs → Advanced Options → mp3 ID Tagging → ID3v2 Text Encoding`.

#### FLAC
Vorbis Comments are the **only** tag format DBpoweramp writes by default. The official help states: *"Vorbis Comments are best for FLAC."* ([dbpoweramp.com](https://www.dbpoweramp.com/help/Codec/Flac/help)) DBpoweramp's FLAC codec can *read* ID3v2 and ID3v1 for backward compatibility with non-standard FLAC files, but it writes Vorbis Comments.

A notable advanced option is **Vorbis Comment Mapping**: DBpoweramp maps certain field names to the official Vorbis equivalents (e.g., `Label` → `Organization`) but this can be disabled to preserve field names as-is.

FLAC ID Tag Padding is automatic. The FLAC encoder *"will automatically set the padding to the expected ID Tag size, saving rewriting the whole file"* when tags change. ([dbpoweramp.com](https://www.dbpoweramp.com/help/Codec/Flac/help))

#### AAC / M4A (including ALAC)
DBpoweramp uses the **iTunes/MP4 atom tagging system** exclusively as the primary format. The help states: *"Adding ID tags to mp4 has standardized on the Apple iTunes format."* ([dbpoweramp.com](https://dbpoweramp.com/Help/Codec/mp4/help))

Album art is stored in the **`covr`** atom. A **compatibility option** exists for Sony and Nokia players, which require an ID3v1 tag (strictly non-standard for M4A) — DBpoweramp can write this on request.

Tag Padding is configurable: *"by using padding, minor tag changes save the whole file being shuffled (as tags are generally before the audio)."* ([dbpoweramp.com](https://dbpoweramp.com/Help/Codec/mp4/help))

M4A Layout has two modes: **Optimized for Streaming** (moov atom before audio data) and **Streaming Incompatible** (audio data first). This affects where tags are placed in the file.

#### WAV
DBpoweramp writes **two concurrent tag systems** to WAV files:
1. **RIFF LIST INFO chunks** (`IART`, `INAM`, `IPRD`, `IGNR`, etc.) — standard RIFF INFO tags
2. **ID3v2 chunk** (`ID3 `) — full ID3v2 metadata with APIC frames for album art

The RIFF LIST chunks are the "basic" layer. The ID3v2 chunk provides full Unicode support, custom fields (TXXX), and album art. Forum posts confirm: *"They are written in a specific wave riff chunk: 'ID3 ' and 'LIST' for the limited tag."* ([DBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/327194-id3-tags-in-wav))

As of R2023+: *"WAV: Change tag type if asked to save ReplayGain and existing tags can't hold ReplayGain."* — DBpoweramp will automatically add an ID3v2 chunk to WAV files if ReplayGain tags need to be written but only RIFF INFO tags exist.

#### AIFF
DBpoweramp writes **ID3v2 tags inside an `ID3 ` chunk** within the AIFF file structure, described as *"iTunes ID3 tags."* ([dbpoweramp.com](https://www.dbpoweramp.com/Help/Codec/Aiff/help)) Album art is stored in APIC frames.

The TagLib issue tracker confirms: *"AIFF files are usually written using ID3v2.3 tags (iTunes and dBPowerAmp don't support ID3v2.4)."* ([taglib/taglib#922](https://github.com/taglib/taglib/issues/922)) However, DBpoweramp's version history notes from R12+ indicate AIFF can now read ID3v2.4, though write defaults remain at v2.3.

#### WMA / ASF
DBpoweramp uses the **native ASF/Windows Media tagging format**. The help states: *"Windows Media Audio have their own tagging format, able to store fields of any name, length and embeddable album art. Fully Unicode compatible."* ([dbpoweramp.com](https://www.dbpoweramp.com/help/dMC/wma.html)) An advanced option controls how Track Number is stored (for compatibility with programs that don't correctly use `WM/Track` and `WM/TrackNumber`).

#### WavPack
DBpoweramp writes **APEv2 tags only** by default. The help states: `ID Tag Writing: Yes [APETagv2]`. ([dbpoweramp.com](https://www.dbpoweramp.com/help/dmc/wavpack.html)) ID3v1 is readable but **not written**. Album art and ReplayGain are supported within APEv2.

#### APE (Monkeys Audio)
APEv2 is the **native** and **default** tag system. DBpoweramp's help confirms: `ID Tag Writing: Yes [APETagv2 (default), APETagv1]`. APE was the format for which APEv2 was invented.

#### OGG Opus
Vorbis Comments in the Ogg container. ID Tag Reading and Writing: `Yes [Vorbis Comments]`. ([dbpoweramp.com](https://www.dbpoweramp.com/help/dmc/opus.html)) Album art is written as `METADATA_BLOCK_PICTURE` (Base64-encoded FLAC picture block). An advanced option allows forcing uppercase tag field names for compatibility with certain devices.

#### MusePack MPC
APEv2 tagging with ReplayGain info support added in R2023+. Prior to that, ReplayGain was not saved for MusePack SV8 files.

---

## 3. ID3v2 Version Strategy

### 3.1 Default Version: ID3v2.3

DBpoweramp defaults to **ID3v2.3** for all formats that use ID3v2 (MP3, WAV, AIFF). This is the de facto standard — the version with the broadest compatibility across all hardware players, car stereos, and software.

Evidence:
- DBpoweramp's MP3 codec help lists `ID3v2` as the recommended format, with ID3v1 and APEv2 as optional additions. ([dbpoweramp.com](https://www.dbpoweramp.com/help/dmc/mp3lame.html))
- The TagLib issue tracker explicitly states: *"AIFF files are usually written using ID3v2.3 tags (iTunes and dBPowerAmp don't support ID3v2.4)."* ([taglib/taglib#922](https://github.com/taglib/taglib/issues/922))
- Forum posts confirm: *"dBpoweramp writes id3v2.3 tags by default"* and *"if a file has a id3v2.4 tag, then dBpoweramp cannot write a v2.3 tag to it"* — meaning a read-modify-write cycle with a v2.4 source fails. ([DBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/19126-replaygain-tags-missing))

### 3.2 User Override: v2.3 or v2.4 Configurable

The `mp3 ID Tagging` section in **dBpoweramp Configuration → Codecs → Advanced Options** allows switching between **ID3v2.3** and **ID3v2.4**. However, this setting is **per-user-account** and requires administrator privileges to save (a known UX issue where running Configuration as a non-admin user silently fails to persist changes). ([DBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/19585-id3v2-tags))

Forum evidence shows users who configured v2.4 but saw v2.3 output — the root cause was running as non-admin. ([DBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/batch-converter/37605-how-to-batch-convert-mp3s-from-id3v2-3-to-id3v2-4))

### 3.3 What Changes Between v2.3 and v2.4

| Feature | ID3v2.3 | ID3v2.4 | DBpoweramp Default |
|---|---|---|---|
| Frame size encoding | 32-bit non-syncsafe | 32-bit synchsafe | v2.3 |
| Text encoding options | ISO-8859-1, UTF-16 | ISO-8859-1, UTF-16, **UTF-8** | v2.3 (UTF-16 default) |
| Date frame | `TYER` (year only) | `TDRC` (full ISO 8601) | v2.3 |
| Multi-value fields | `/` separator | Null (`\0`) separator | v2.3 |
| Album sort | `TSOA` | `TSOA` | v2.3 |
| Artist sort | `TSOP` | `TSOA` (album sort), `TSOP` (performer sort) | v2.3 |
| CRLF handling | Native | All CRLF converted to LF on write | v2.3 preferred |
| Synchronization | Removed in v2.4 | Present | Not applicable |
| Tag location | Prepended (required) | Can be appended | Both prepend |

### 3.4 What Breaks with Each Version

**ID3v2.4 issues:**
- Windows Media Player does not read ID3v2.4 tags at all. ([Hydrogenaudio Forum](https://hydrogenaud.io/index.php/topic,74407.0.html))
- Many car stereos, older hardware MP3 players, and budget Bluetooth speakers cannot parse v2.4 frames — they display blank fields or fall back to v1. ([AudioUtils](https://audioutils.com/blog/id3-tags-explained))
- foobar2000 reads v2.4 but defaults to writing v2.3 compatibility mode; when it writes v2.4 over files, it can create v2.3+v2.4 conflicts.
- MusicBee, JRiver, and other players auto-convert on sync.
- **DBpoweramp specifically**: Cannot write to a file that already has a v2.4 tag in-place. The workaround is the **ID Tag Update** utility codec, which strips all tags and rewrites using the current global version setting.

**ID3v2.3 issues:**
- Cannot natively store UTF-8 — relies on UTF-16 for international characters (wastes space for ASCII-heavy English tags).
- Date field limited to year (`TYER`) with no time-of-day or timezone support.
- Multi-value handling with `/` separator is less robust than v2.4's null-separator approach.

### 3.5 v2.3 → v2.4 Read-Modify-Write

DBpoweramp has a **critical limitation**: if a source MP3 has ID3v2.4 tags, DBpoweramp cannot directly overwrite them with v2.3 tags. Forum posts confirm: *"If a file has a id3v2.4 tag, then dBpoweramp cannot write a v2.3 tag to it."* The resolution is the **ID Tag Update** utility codec, which reads all tags, removes them completely, then rewrites using the configured version. ([DBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/19126-replaygain-tags-missing))

---

## 4. Atomic Write and File Safety

### 4.1 CD Ripper: `.tmp` → Rename Strategy

DBpoweramp CD Ripper implements a **strict atomic write pattern** for ripping:

1. Each track is encoded and written as `track001.tmp`, `track002.tmp`, etc.
2. Metadata, ReplayGain, and tagging are applied to the `.tmp` file.
3. Only after **all** tracks on the album have completed encoding and tagging does DBpoweramp rename all files to their final extensions (`.flac`, `.mp3`, etc.).
4. The rename is atomic at the filesystem level.

This is described by the developer as intentional: *"the idea is any player would not see the tracks until the whole album is ripped, especially if using ReplayGain which writes an album gain after all tracks are written."* ([DBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/328169-question-about-the-update-release-2024-09-30-windows))

A change in R2024 replaced `.` prefix filenames (e.g., `._track001.flac`) with `.tmp` extensions, which better signals temporary files and avoids programs incorrectly indexing `._` macOS resource fork files.

**Ripping modes** (configurable in CD Ripper → When Ripping):
- **Option 1** (default): Write to `.tmp`, rename after all tracks complete.
- **Option 2**: Write to a temporary folder, then move the entire album folder after completion.
- **Option 3**: `Rip direct to final filenames` — not recommended, as a failed rip leaves partial files visible to media libraries.

### 4.2 Batch Converter and Music Converter: Temp File Strategy

For format conversions (e.g., FLAC → MP3), DBpoweramp writes to a temp file first, then replaces the original. The temporary file location is configurable in **dBpoweramp Control Center → Advanced → Temporary Folder**, defaulting to `%TEMP%` (system temp folder).

When using `CoreConverter.exe` from the command line, the outfile is written first, then on successful completion the original can be overwritten or the output moved.

### 4.3 Failure Behavior

- If encoding or tagging fails, the `.tmp` file is abandoned and no rename occurs.
- The original source file is **never modified** during a conversion operation.
- For the ID Tag Update utility codec: tags are removed and rewritten in a single pass — if interrupted, the file would have no tags.

### 4.4 Padding as a Write Optimization

For **FLAC** specifically, DBpoweramp automatically calculates and embeds padding during encoding: *"as Vorbis Comments are at the beginning of the file it is advantageous to insert padding so if the ID Tags change then the whole file does not need rewriting. When encoding a file dBpoweramp's FLAC encoder will automatically set the padding to the expected ID Tag size."* ([dbpoweramp.com](https://www.dbpoweramp.com/help/Codec/Flac/help))

For **M4A/MP4**, a similar padding option exists: *"by using a padding, minor tag changes save the whole file being shuffled (as tags are generally before the audio)."* ([dbpoweramp.com](https://dbpoweramp.com/Help/Codec/mp4/help))

ID3v2 padding strategy follows the standard: padding (null bytes) may exist after the final frame, before the audio data. The ID3v2.4 spec states: *"A possible purpose of this padding is to allow for adding a few additional frames or enlarge existing frames within the tag without having to rewrite the entire file."* Padding bytes must be `$00`. ([ID3.org](https://id3.org/id3v2.3.0))

---

## 5. Tag Cleanup and Stripping Rules

### 5.1 The ID Tag Update Utility: Total Replacement

The **ID Tag Update** utility codec is DBpoweramp's primary mechanism for tag manipulation. Its behavior is **total replacement**:
> *"ID Tag Update will read all existing tags and write new tags to the file (old ones removed) using the tag creation settings from dBpoweramp Configuration."* ([dbpoweramp.com](https://dbpoweramp.com/Help/dMC/idtagupdate.html))

This means:
- It does not edit in-place.
- It does not preserve one tag system while updating another.
- All tag systems present in the source are **removed first**, then the configured output tag system(s) are written fresh.

This is confirmed by forum posts: *"This is how update ID tag works, it clears the existing tags totally then rewrites to the type as set in the dB Config."* ([DBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/27227-mp3-with-id3v1-id3v2-3-and-apev2))

### 5.2 During Format Conversion (FLAC → MP3, etc.)

When converting between formats, DBpoweramp:
1. **Reads all tag systems** from the source (Vorbis Comments from FLAC, ID3v2 from MP3, etc.).
2. **Maps** field names to the target format's tag system.
3. **Writes** only the target format's native tag system(s) to the output file.
4. **Strips** the source's tag system entirely from the output.

**Critical: The source format's tag system is not preserved in the output.** For example:
- Converting FLAC → MP3 does **not** carry Vorbis Comments into the MP3 as APEv2. Only ID3v2 (and optionally ID3v1) tags are written.
- Converting MP3 → FLAC does **not** carry ID3v2 into FLAC as ID3v2. Only Vorbis Comments are written.

This is a **clean conversion** — each output file has only its format's canonical tag system(s).

### 5.3 ID3v2.2 and Legacy Tag Stripping

DBpoweramp does not write ID3v2.2 frames (the older 3-character frame ID format). When reading a file with ID3v2.2 tags, DBpoweramp reads and translates the fields to the modern frame ID scheme, then writes the modern frame IDs in the output.

### 5.4 User Control Over Stripping

The **ID Tag Processing DSP effect** (usable in Batch Converter) provides granular control:
- **Map**: Copy a tag to another name without removing the original.
- **Deletion**: Remove specific tags, or remove all tags except listed ones, or remove all tags entirely.
- **Manipulation**: Rule-based modifications (character replacement, conditional changes).
- **Additions**: Add new tags.

The deletion options allow users to explicitly remove ReplayGain tags, custom fields, or any specific tag by name — but this requires a deliberate configuration, not automatic behavior.

---

## 6. ID3v1 Handling

### 6.1 When Is ID3v1 Written?

DBpoweramp writes ID3v1 **only when explicitly configured** for MP3, and as a compatibility layer for M4A (Sony/Nokia devices). For all other formats, ID3v1 is **never written**.

For MP3, the `Tag Creation` option in `dBpoweramp Configuration → Codecs → Advanced Options → mp3 ID Tagging` offers:
- `ID3v2` (default) — only v2
- `ID3v1` — only v1
- `ID3v2 & ID3v1` — both (most common full-compatibility setting)
- `APEv2` — alternate
- `APEv2 & ID3v1` — alternate

The `ID3v1 Version` option switches between **ID3v1.0** (track number in comment field) and **ID3v1.1** (dedicated 1-byte track number field). ([dbpoweramp.com](https://www.dbpoweramp.com/help/dmc/mp3lame.html))

### 6.2 ID3v1 Character Encoding Limitations

ID3v1 has no encoding byte — it is implicitly **ISO-8859-1** (Latin-1), which covers Western European characters but not Unicode. DBpoweramp's handling:
- For ID3v1, characters outside ISO-8859-1 are **truncated or replaced** at the byte level.
- For full Unicode support, **ID3v2 is required**. This is why DBpoweramp's default for MP3 is `ID3v2` or `ID3v2 & ID3v1` with UTF-16 encoding in v2.
- The ID3v1 field length limits (30 bytes for title/artist/album, 30 for comment, 4 for genre) mean long Unicode strings will be severely truncated or lost.

### 6.3 ID3v1 Read Priority

When DBpoweramp reads MP3 files with both ID3v2 and ID3v1, it prioritizes ID3v2 for display and editing. The ID3v1 tag serves as a fallback for very old players and legacy hardware.

---

## 7. Cover Art Writing Strategy

### 7.1 Per-Format Cover Art Storage

| Format | Cover Art Mechanism | DBpoweramp Default |
|---|---|---|
| MP3 | **APIC** frame inside ID3v2 | Yes (in ID3v2) |
| FLAC | **METADATA_BLOCK_PICTURE** Vorbis comment (binary block, not Base64 in native FLAC) | Yes |
| OGG Vorbis | **METADATA_BLOCK_PICTURE** Base64-encoded Vorbis comment | Yes |
| Opus | **METADATA_BLOCK_PICTURE** Base64-encoded Vorbis comment | Yes |
| AAC/M4A | **`covr`** atom (binary) inside MP4 container | Yes |
| WAV | **APIC** frame inside ID3v2 chunk | Yes (in ID3v2) |
| AIFF | **APIC** frame inside ID3v2 `ID3 ` chunk | Yes |
| WMA/ASF | **WM/Picture** attribute (binary) | Yes |
| WavPack WV | **APEv2** item with binary picture data | Yes |
| APE | **APEv2** item with binary picture data | Yes |
| MusePack MPC | **APEv2** item | Yes |

### 7.2 FLAC Cover Art: METADATA_BLOCK_PICTURE vs COVERART

DBpoweramp writes **METADATA_BLOCK_PICTURE** as the standard FLAC picture block. This is the current recommended format per the Xiph.org Vorbis Comment specification and RFC 9639 (FLAC format spec, 2024). ([XiphWiki](https://wiki.xiph.org/VorbisComment), [RFC 9639](https://www.rfc-editor.org/rfc/rfc9639.txt))

The legacy `COVERART` Vorbis comment (raw JPEG bytes, no metadata) and `COVERARTMD5` are not written by DBpoweramp.

The FLAC picture block structure (per RFC 9639) includes:
- Picture Type (32-bit big-endian, values 0–20)
- MIME type length + MIME type string (UTF-8)
- Description length + description string (UTF-8)
- Width, height, color depth, indexed color count (all 32-bit big-endian)
- Picture data length + binary image data

### 7.3 User Control Over Cover Art

The **ID Tag Processing DSP effect** provides:
- **Import** album art from a file (`Folder.jpg`, `cover.jpg`, or explicit path)
- **Export** album art to a file
- **Maximum art size** limiting by pixel dimensions or byte size
- **Force embedded art to JPG** — converts PNG covers to JPEG for players that don't support PNG

Album art must be explicitly enabled in CD Ripper options: *"scroll down to 'write ID tags' section and tick the box next to 'Album Art'."* ([DBpoweramp Forum](https://forum.dbpoweramp.com/forum/other-topics/how-do-i/28536-how-to-write-id3-tags-in-flac-files-with-flac-tag))

---

## 8. ReplayGain Writing Strategy

### 8.1 Per-Format ReplayGain Storage

| Format | ReplayGain Tag Names | DBpoweramp Behavior |
|---|---|---|
| **MP3** | `REPLAYGAIN_TRACK_GAIN`, `REPLAYGAIN_TRACK_PEAK`, `REPLAYGAIN_ALBUM_GAIN`, `REPLAYGAIN_ALBUM_PEAK` | Written in ID3v2.3 frames by default |
| **FLAC** | `REPLAYGAIN_TRACK_GAIN`, `REPLAYGAIN_TRACK_PEAK`, `REPLAYGAIN_ALBUM_GAIN`, `REPLAYGAIN_ALBUM_PEAK` | Written as Vorbis Comments |
| **OGG Vorbis** | `REPLAYGAIN_TRACK_GAIN`, etc. | Written as Vorbis Comments |
| **Opus** | `REPLAYGAIN_TRACK_GAIN`, etc. | Written as Vorbis Comments |
| **AAC/M4A** | `REPLAYGAIN_*` Vorbis-style tags, plus `iTunNORM` (iTunes SoundCheck) | Both written; `iTunNORM` for iTunes compatibility |
| **WavPack** | `REPLAYGAIN_*` in APEv2 | Written as APEv2 items |
| **WAV** | Requires ID3v2 chunk for ReplayGain storage | DBpoweramp auto-adds ID3v2 chunk if not present |
| **WMA/ASF** | `REPLAYGAIN_*` attributes | Written as ASF attributes |

### 8.2 Calculation and Application Methods

DBpoweramp provides two mechanisms:

1. **ReplayGain DSP Effect** (during conversion): Calculates and writes ReplayGain values as part of the encoding pipeline. The audio data is untouched — only metadata tags are written.

2. **ReplayGain Utility Codec** (batch on existing files): Right-click files → Convert To → `[Replay Gain]`. This scans all selected tracks and writes ReplayGain tags in a single batch operation.

### 8.3 Configuration Options (per ReplayGain DSP)

- **Mode**: Track Gain or Album Gain (album gain takes all tracks' loudness into account so quiet tracks remain relatively quiet within the album context).
- **Calculation Method**: ReplayGain 2.0 (default) or **EBU R128** (recommended, used by streaming platforms).
- **Reference Loudness**: Default −18 LUFS (EBU R128); configurable.
- **Clip Prevention**: Enabled by default; adds 1dB headroom to prevent clipping from volume normalization.
- **True Peak**: For files that will be oversampled on playback; calculates inter-sample peak.
- **Album definition**: By album ID tag, or by grouping files in the same folder.

### 8.4 iTunes SoundCheck (iTunNORM)

For AAC/M4A files, DBpoweramp can write the `iTunNORM` tag alongside ReplayGain tags. The `iTunNORM` tag contains pre-computed gain and peak values that iTunes and Apple devices use for SoundCheck volume normalization (separate from the standard ReplayGain mechanism).

---

## 9. User-Facing Behavior Summary

### 9.1 Key Takeaways for Users

1. **DBpoweramp always writes format-native tags**: FLAC gets Vorbis Comments, not ID3v2. MP3 gets ID3v2 by default, not Vorbis Comments. M4A gets iTunes atoms. WavPack and APE get APEv2. Converting between formats means the output file will have its format's tag system only.

2. **Default ID3v2 version is 2.3**: This is the most compatible version across all devices. v2.4 is available in settings but carries compatibility risks. If you need v2.4, configure it in `dBpoweramp Configuration → Codecs → Advanced Options` (run as administrator).

3. **Tag cleanup is total, not incremental**: The ID Tag Update utility removes all existing tags and rewrites using the current global settings. There is no in-place editing. This means if you run ID Tag Update on a file with APEv2 + ID3v2.4 tags, both are stripped and the configured tag system is written fresh.

4. **Atomic writes prevent partial/corrupt files**: CD Ripper writes to `.tmp` files and only renames to the final extension after the entire album is encoded and tagged. Conversion operations write to a temp file first.

5. **FLAC→MP3 and FLAC→AAC conversions strip Vorbis Comments**: The Vorbis Comment field names are mapped to their ID3v2/MP4 equivalents. Custom field names that don't map to standard ID3v2 frames are generally lost unless they can be represented as TXXX frames.

6. **Padding is automatic for FLAC and M4A**: DBpoweramp automatically reserves padding in FLAC and M4A files during encoding so that future tag edits don't require rewriting the entire audio stream.

7. **Cover art is universally supported across all formats**: DBpoweramp writes it in the format-appropriate manner (APIC for MP3/WAV/AIFF, METADATA_BLOCK_PICTURE for FLAC/OGG/Opus, covr atom for M4A, APEv2 binary items for WavPack/APE).

8. **ReplayGain is supported for all major formats**: Written as format-native tags with both ReplayGain 2.0 and EBU R128 calculation options.

### 9.2 Would a User Notice Any Difference from DBpoweramp?

| Scenario | User Impact |
|---|---|
| **Default ID3v2.3 vs v2.4** | Users with Windows Media Player, older car stereos, or older hardware DAPs: v2.3 tags display correctly. v2.4 tags may show blank fields or be silently ignored by these players. |
| **FLAC→MP3 conversion** | Vorbis Comment fields not supported in ID3v2 (e.g., `RELEASESTATUS`, `SCRIPT`, `RELEASECOUNTRY`, custom fields) are lost unless mapped to equivalent TXXX frames. Users relying on extensive MusicBrainz fields should verify after conversion. |
| **FLAC→AAC conversion** | Same as above. iTunes atoms support a different field name space. Some custom/niche fields may not transfer. |
| **ID Tag Update on MP3 with v2.4** | DBpoweramp cannot write to a file with v2.4 tags in-place. Users must run ID Tag Update, which strips all tags and rewrites using the current v2 version setting. |
| **Cover art format** | PNG embedded in FLAC stays as METADATA_BLOCK_PICTURE (binary PNG). For MP3/WAV/AIFF, embedded art is always APIC (JPEG or PNG). For M4A, it's the covr atom. No format converts cover art to a different image format automatically. |
| **Multi-value artist fields** | DBpoweramp's UI displays multi-value fields with `; ` as separator. Internally, it writes them per format spec: separate Vorbis Comment entries for FLAC/OGG, separate `TPE1` frames for ID3v2.4, or `/`-separated single fields for ID3v2.3. Players that don't support multi-value will see concatenated values. |
| **ReplayGain on WAV** | If a WAV has only RIFF INFO tags, DBpoweramp auto-adds an ID3v2 chunk to store ReplayGain. Some WAV editors may not recognize the ID3v2 chunk in WAV files. |
| **APE/Monkey's Audio → any format** | APEv2 tags are stripped and replaced with the target format's tag system. No APEv2 data persists into FLAC, MP3, or M4A output. |

### 9.3 Configuration Checklist for Optimal Results

| Setting | Recommended | Location |
|---|---|---|
| MP3 Tag Creation | `ID3v2 & ID3v1` (full compatibility) | Config → Codecs → Advanced → mp3 ID Tagging |
| MP3 ID3v2 Version | `2.3` (maximum compatibility) | Config → Codecs → Advanced → mp3 ID Tagging |
| MP3 ID3v2 Text Encoding | `UTF-16` (Unicode support) | Config → Codecs → Advanced → mp3 ID Tagging |
| MP3 ID3v1 Version | `1.1` (track number support) | Config → Codecs → Advanced → mp3 ID Tagging |
| FLAC Tag Creation | `Vorbis Comments` (default, do not change) | Config → Codecs → Advanced → FLAC ID Tagging |
| FLAC Cover Art | `METADATA_BLOCK_PICTURE` (automatic, do not change) | Config → Ripper → Options |
| M4A Cover Art | `covr` atom (automatic) | Config → Ripper → Options |
| Album Art Embedding | **Enabled** (not just external JPG) | Config → Ripper → Options → ID Tags & Metadata |
| ReplayGain | `EBU R128` (streaming-standard calculation) | Config → DSP Effects → ReplayGain |
| FLAC Padding | Automatic (do not change) | Config → Codecs → Advanced → FLAC ID Tagging |

---

## Sources

1. [dBpoweramp MP3 (LAME) Help](https://www.dbpoweramp.com/help/dmc/mp3lame.html) — Tag creation options, ID3v2 default
2. [dBpoweramp FLAC Help](https://www.dbpoweramp.com/help/Codec/Flac/help) — Vorbis Comments, padding, tag mapping
3. [dBpoweramp FLAC Help (OSX)](https://dbpoweramp.com/Help/dMCOSX/flac) — FLAC tag details
4. [dBpoweramp MP4/M4A Help](https://dbpoweramp.com/Help/Codec/mp4/help) — iTunes atoms, covr, tag padding
5. [dBpoweramp WMA Help](https://www.dbpoweramp.com/help/dMC/wma.html) — ASF tagging
6. [dBpoweramp WAV Help](https://dbpoweramp.com/Help/Codec/WAV/help) — RIFF INFO + ID3v2 dual tagging
7. [dBpoweramp AIFF Help](https://www.dbpoweramp.com/Help/Codec/Aiff/help) — iTunes ID3v2 in AIFF
8. [dBpoweramp Opus Help](https://www.dbpoweramp.com/help/dmc/opus.html) — Vorbis Comments in Opus
9. [dBpoweramp OGG Help](https://www.dbpoweramp.com/help/dmc/ogg.html) — Vorbis Comments in OGG
10. [dBpoweramp WavPack Help](https://www.dbpoweramp.com/help/dmc/wavpack.html) — APEv2 tagging
11. [dBpoweramp Monkeys Audio Help](https://www.dbpoweramp.com/help/dmc/monkeysaudio.html) — APEv2 native tagging
12. [dBpoweramp ReplayGain Help](https://dbpoweramp.com/Help/dMC/replaygain.html) — ReplayGain writing options
13. [dBpoweramp DSP Help](https://dbpoweramp.com/Help/dMC/dsp) — ID Tag Processing, ReplayGain DSP
14. [dBpoweramp ID Tag Update Help](https://dbpoweramp.com/Help/dMC/idtagupdate.html) — Total tag replacement behavior
15. [dBpoweramp Configuration Help](https://dbpoweramp.com/Help/dMC/dMCConfig) — Global settings reference
16. [dBpoweramp R2023 Release Notes](https://forum.dbpoweramp.com/forum/read-only/news-updates-read-only/43480-dbpoweramp-r202x-release) — TAK, TTA, MusePack ReplayGain additions
17. [DBpoweramp Forum: ID3v2 tags?](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/19585-id3v2-tags) — ID3v2.3 default, admin config requirement
18. [DBpoweramp Forum: Replaygain tags missing](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/19126-replaygain-tags-missing) — v2.3 default, v2.4 source file limitation
19. [DBpoweramp Forum: v2.3 to v2.4 batch conversion](https://forum.dbpoweramp.com/forum/dbpoweramp/batch-converter/37605-how-to-batch-convert-mp3s-from-id3v2-3-to-id3v2-4) — ID Tag Update, per-user config
20. [DBpoweramp Forum: ID3v1 and ID3v2 tags](https://forum.dbpoweramp.com/forum/other-topics/how-do-i/6882-id3v1-and-id3v2-tags) — Tag Creation options
21. [DBpoweramp Forum: APEv2 + ID3v2 simultaneous](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/27227-mp3-with-id3v1-id3v2-3-and-apev2) — No simultaneous ID3v2+APEv2
22. [DBpoweramp Forum: WAV ID3v2 tags](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/327194-id3-tags-in-wav) — Dual WAV tagging (LIST + ID3 chunk)
23. [DBpoweramp Forum: Ripping .tmp files](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/337184-flac-selected-but-rip-is-tmp) — Atomic .tmp write behavior
24. [DBpoweramp Forum: Temp file location](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/335849-can-i-know-where-does-dbpoweramp-ripper-store-temp-files) — Temp folder strategy
25. [DBpoweramp Forum: FLAC album art in Vorbis](https://forum.dbpoweramp.com/forum/other-topics/how-do-i/28536-how-to-write-id3-tags-in-flac-files-with-flac-tag) — FLAC artwork embedding
26. [AudioUtils: ID3 Tags Explained](https://audioutils.com/blog/id3-tags-explained) — ID3v2.3 vs v2.4 compatibility analysis
27. [Hydrogenaudio Forum: v2.3 and v2.4 coexistence](https://hydrogenaud.io/index.php/topic,74407.0.html) — Device compatibility with v2.4
28. [GetMusicBee Forum: ID3v2.3 vs v2.4](https://getmusicbee.com/forum/index.php?topic=39928.0) — MusicBee v2.4 handling
29. [Mp3tag Forum: ID3v1 stripping](https://community.mp3tag.de/t/id3v1-tags-utterly-obolete-now-can-mp3tag-remove/7580) — Tag stripping mechanics
30. [Mp3tag MPEG Options](https://docs.mp3tag.de/customization/options/tags/mpeg/) — Read/write/remove tag configuration
31. [ID3.org: ID3v2.3.0 Specification](https://id3.org/id3v2.3.0) — Padding, frame ordering, tag structure
32. [ID3.org: ID3v2.4.0 Structure](https://id3.org/id3v2.4.0-structure) — Synchsafe integers, padding rules
33. [RFC 9639: FLAC Format](https://www.rfc-editor.org/rfc/rfc9639.txt) — METADATA_BLOCK_PICTURE, Vorbis Comments in FLAC
34. [Xiph.org: Vorbis Comment](https://wiki.xiph.org/VorbisComment) — METADATA_BLOCK_PICTURE specification
35. [Xiph.org: Vorbis Comment docs](https://xiph.org/vorbis/doc/v-comment.html) — Vorbis comment structure
36. [TagLib Issue #922](https://github.com/taglib/taglib/issues/922) — AIFF ID3v2.3 vs v2.4 in DBpoweramp
37. [Hydrogenaudio Wiki: WavPack](https://wiki.hydrogenaudio.org/index.php?title=WavPack) — APEv2 tagging, ReplayGain
38. [DBpoweramp Version History](https://versions.dbpoweramp.com/?appid=3) — Feature timeline R14–R2024
39. [Mp3tag Tag Mapping Table](https://docs.mp3tag.de/mapping-table/) — Cross-format field name mappings
40. [Stack Overflow: FFmpeg atomic metadata editing](https://stackoverflow.com/questions/11474532/how-to-change-metadata-with-ffmpeg-avconv-without-creating-a-new-file) — Temp file + rename pattern for reference
