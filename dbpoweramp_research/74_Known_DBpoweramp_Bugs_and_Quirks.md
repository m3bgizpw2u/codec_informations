# 74_Known_DBpoweramp_Bugs_and_Quirks.md

## Known DBpoweramp Bugs and Quirks

> **Purpose:** Research and document all known DBpoweramp issues including community-reported bugs, version-specific quirks, format-specific problems, and workarounds.

---

## SECTION 1: COMMUNITY-REPORTED BUGS

### 1.1 Hydrogenaudio Forums

**Source:** https://hydrogenaudio.org

#### Bug: FLAC files tagged with ID3v2 instead of Vorbis Comments

**Thread:** "Tags made by dbpoweramp doesn't show in Mp3tag or MediaMonkey"
**URL:** https://hydrogenaudio.org/index.php/topic,68944.0.html
**Severity:** Medium
**Status:** Resolved (user configuration issue)

**Description:**
DBpoweramp was configured to write ID3v2 tags to FLAC files instead of Vorbis Comments. This caused compatibility issues with players that don't recognize ID3v2 in FLAC files.

**Symptoms:**
- Mp3tag and MediaMonkey couldn't read tags
- Foobar2000 and Squeezecenter read tags correctly
- Hovering over files in Explorer showed "ID3v2.3(ANSI)"

**Root Cause:**
User had configured FLAC codec to use ID3 tagging instead of Vorbis Comments.

**Workaround:**
1. Open dBpoweramp Control Center
2. Go to Audio Codecs → FLAC → Advanced Options
3. Set "Tag Creation" to "Vorbis Comments"
4. To fix existing files: Install [ID Tag Update] utility codec
5. Batch convert all FLAC files using [Update ID Tag] with new settings

**Prevention:**
Check tag creation settings before ripping any CDs.

---

#### Bug: Album art causing tag visibility issues in Windows Explorer

**Thread:** "ALAC to FLAC conversion - No tags in Windows explorer"
**URL:** https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/331261-alac-to-flac-conversion-no-tags-in-windows-explorer
**Severity:** Medium
**Status:** Resolved (R2022-08-09+)

**Description:**
After converting ALAC (M4A) files to FLAC, Windows Explorer wouldn't display tags in folder view even though:
- Tags were visible in dBpoweramp popup info
- Tags were visible in file Properties → Audio Properties and ID-Tag tabs
- Tags were visible in Explorer hover popup

**Symptoms:**
- Windows Explorer shows empty tag columns
- dBpoweramp popup and Properties dialog show tags correctly
- Windows Media Player couldn't play the files

**Root Cause:**
Large album art embedded in the ALAC file caused corruption of the FLAC header when converted. Specifically, the album art block size exceeded expected values.

**Workaround:**
1. Delete album art from source file before conversion
2. Or convert again after deleting art
3. Or re-convert without art, then add art back

**Fix:**
This was addressed in R2022-08-09 with improved album art handling.

---

### 1.2 DBpoweramp Official Forums

**Source:** https://forum.dbpoweramp.com

#### Bug: GetPopupInfo crash (Explorer integration)

**Threads:**
1. "Crash: dBpoweramp GetPopupInfo has stopped working" (August 2024)
   - URL: https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/326906-crash-dbpoweramp-getpopupinfo-has-stopped-working
   
2. "Error: GetPopupInfo has stopped working" (January 2026)
   - URL: https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/337596-error-getpopupinfo-has-stopped-working

**Severity:** High (Explorer crashes)
**Status:** Partially resolved

**Description:**
dBpoweramp's shell extension (GetPopupInfo) caused random Explorer.exe crashes on Windows 11. The crash module was Wavpack.dll in some cases.

**Symptoms:**
- Random Explorer crashes when hovering over audio files
- "GetPopupInfo has stopped working" popup appears
- Crash dumps show Wavpack.dll as the faulting module
- Issue sometimes tied to specific WavPack files

**Root Cause:**
Malformed WavPack files (with 4GB+ RIFF chunks) triggered a crash in the Wavpack decoder used by the shell extension.

**Workarounds:**
1. Disable popup information tips:
   - dBpoweramp Configuration → Advanced Settings
   - Uncheck "Popup information tips"
   
2. Disable thumbnail and property handlers:
   - dBpoweramp Configuration → Advanced Settings
   - Uncheck "Thumbnail and property handler"
   
3. Uninstall shell extension components:
   - Uncheck all "Shell" options in configuration

**Fix:**
R2026-01-31 introduced a workaround for corrupted WavPack files.

---

#### Bug: WavPack corrupted file handling (4GB RIFF chunks)

**Severity:** High
**Status:** Resolved (R2026-01-31)

**Description:**
WavPack files with corrupted RIFF chunks (specifically chunks larger than 4GB) caused:
- Explorer crashes
- Conversion failures
- Shell extension instability

**Symptoms:**
- Explorer crashes when accessing WavPack files
- "Riff chunk set at 4,294,967,287 bytes" - obviously invalid
- dBpoweramp couldn't convert the files

**Workaround:**
1. Beta Wavpack.dll available at: https://dbpoweramp.com/beta/Wavpack.dll
2. Download and copy to C:\Program Files\dBpoweramp\decoder
3. Full Windows restart required

**Fix:**
R2026-01-31 included Wavpack 5.9.0 with improved corrupted file handling.

---

#### Bug: CD Ripper metadata lookup failures with proxy settings

**Severity:** Medium
**Status:** Resolved (R2022-08-09)

**Description:**
CD Ripper couldn't retrieve metadata when proxy settings were configured on Windows.

**Symptoms:**
- Metadata lookup fails silently
- No error message shown
- Manual lookup works

**Fix:**
R2022-08-09 fixed proxy settings handling for CD Ripper metadata.

---

#### Bug: Ogg Vorbis duplicate Album Artist tags

**Severity:** Low
**Status:** Resolved (R2022-08-09)

**Description:**
Ogg Vorbis and Opus files could contain duplicate "Album Artist" tags that couldn't be edited or deleted, leading to accumulation of duplicate tags.

**Symptoms:**
- Multiple identical ALBUMARTIST tags in file
- Editing in tag editor doesn't remove duplicates
- Some players show wrong artist

**Workaround:**
Use [ID Tag Update] utility codec to rewrite tags cleanly.

**Fix:**
R2022-08-09 fixed reading and writing to remove duplicates.

---

#### Bug: m4a tagging could fail on some systems

**Severity:** Medium
**Status:** Resolved (R17.6)

**Description:**
On some systems, m4a tag writing would fail to write title and other tags.

**Symptoms:**
- Title tag missing after conversion to m4a
- Other tags also missing
- Error not always reported

**Fix:**
R17.6 improved m4a tag writer reliability.

---

### 1.3 SoundExchange / Professional Forums

**Source:** Various professional audio forums

#### Issue: WAV file tag limitations

**Severity:** Low
**Status:** Known limitation

**Description:**
WAV files support very limited metadata through the INFO chunk. DBpoweramp supports both INFO and ID3v2-in-WAV, but some players only read one or the other.

**Limitations:**
- INFO chunk only supports: IPRD, INAM, IART, IGNR, ICRD, ITRK, ICMT, ISFT, IKEY, IMED, ICOP, IRTD, IWEB, IASPI
- ALBUMARTIST not natively supported in INFO
- Cover art only in ID3v2 chunk (non-standard)
- ReplayGain not natively supported

**Workarounds:**
1. Configure WAV tagging to use both LIST & ID3 in Control Center
2. Accept that some players won't read all tags
3. Convert to FLAC if full tag support needed

---

## SECTION 2: VERSION-SPECIFIC QUIRKS

### 2.1 Version History (Relevant Changes)

#### R21 (2025-2026)

| Version | Date | Key Changes | Bug Fixes |
|---------|------|-------------|-----------|
| R2026-04-03 | Apr 2026 | Opus 1.6.1, Multi-CPU defaults | Wavpack corrupted file crash |
| R2026-01-31 | Jan 2026 | Opus 1.6.1, M4A tagging tech update | "Error 380" bug, FLAC version |
| R2025-12-25 | Dec 2025 | New icons, Ogg faster, Opus 1.6 | Large FLAC art, CoreConverter race |
| R2025-11-12 | Nov 2025 | ARDFTSRC resampler, 64-bit only | Pre-emphasis, IEEE_FLOAT |
| R2025-07-07 | Jul 2025 | Darkmode, compilation detection | Multi-artist display |

#### R20 (2024)

| Version | Date | Key Changes | Bug Fixes |
|---------|------|-------------|-----------|
| R2024-05-30 | May 2024 | AccurateRip metadata (replaces freedb) | Cross-press AR calculation |
| R2024-04-01 | Apr 2024 | Opus 1.5.1, VST fixes | Conditional encode |
| R2024-02-01 | Feb 2024 | Shell menu improvements | Vorbis tagging, folder.jpg |

#### R19 (2023)

| Version | Date | Key Changes | Bug Fixes |
|---------|------|-------------|-----------|
| R2023-12-22 | Dec 2023 | More decoders, DSD support | Various decoding issues |
| R2023-10-10 | Oct 2023 | LAME 3.101.b3, ReplayGain true peak | Scanner fixes |
| R2023-06-26 | Jun 2023 | FLAC 1.4.3 | Pre-Windows 10 compatibility |

#### R17 (2020-2022)

| Version | Date | Key Changes | Bug Fixes |
|---------|------|-------------|-----------|
| R17.7 | Mar 2022 | — | m4a decoder silence, DSP float |
| R17.6 | Oct 2021 | R128 adaptive normalize | m4a tagging, Wave ID3 |
| R17.0 | Apr 2020 | Multi-core encoding, Quick Convert | Various |

#### R16 (2016-2019)

| Version | Date | Key Changes | Bug Fixes |
|---------|------|-------------|-----------|
| R16.6 | Jan 2019 | Drop encoding tool tag | — |
| R16.5 | Sep 2018 | High DPI, FDK update | Various shell |
| R16.0 | Jun 2016 | Complete UI redesign | Various |

#### R15 (2014-2015)

| Version | Date | Key Changes | Bug Fixes |
|---------|------|-------------|-----------|
| R15.3 | May 2015 | — | Naming, genre |
| R15.2 | Feb 2015 | 64-bit, Musepack/Opus/Ogg | AIFF corruption, m4a decoder |
| R15.0 | Feb 2014 | 64-bit version | Various |

#### R14 (2011-2013)

| Version | Date | Key Changes | Bug Fixes |
|---------|------|-------------|-----------|
| R14.4 | Feb 2013 | Multi-item tags | FLAC picture, m4a tagging |
| R14.3 | Sep 2012 | MP3 v2.4, Sort tags | Various |
| R14.0 | Apr 2011 | AccurateRip v2 | Various |

### 2.2 Behavioral Changes Between Versions

#### Track/Disc Number Handling Changes

**R14.3 (September 2012):**
- Added support for reading/writing TRACKTOTAL and DISCTOTAL tags for FLAC
- Previously relied solely on parsing "5/12" format from TRACKNUMBER

**R17.2 (September 2020):**
- FLAC and M4A can now store disc and track count only without corresponding number
- Fixed issue where having only count but no number was rejected

#### Sort Tag Behavior

**R14.3 (September 2012):**
- Sort tags now tied to unsort tags
- If metadata provider provides a sort value, the corresponding tag is also used
- Previously sort tags were set independently

**R16.5 (September 2018):**
- Album Sort is no longer split as multi-item tag if contains `'; '`

#### Multi-Artist Handling

**R17.0 (April 2020):**
- Multi-artist handling improved
- [Multi-Encoder] works correctly with ID Processing DSP

**R2025-07-21 (July 2025):**
- Fixed: ID Tag editing, multiple artists / etc would not show `'; '` between when editing

#### Genre Number Resolution

**R14.3 (September 2012):**
- MP3 genre written as number `(17)` with freeform text `(17)Rock` no longer created
- Always writes freeform genre strings
- Can retroactively fix existing files using [ID Tag Update]

#### CD Ripper Metadata

**R2024-05-30:**
- ID Tags from providers: unicode `\u2010` (hyphen) replaced with `-`
- "weird unicode spaces" standardized to regular space

**R2025-06-05:**
- Added multiline editboxes for comment/lyrics fields in tag editor

### 2.3 What Remains Broken

Based on current known issues (as of R2026-04-03):

1. **WavPack shell extension crashes**: Still requires disabling popup info on some systems with corrupted WV files

2. **FLAC → WAV tag visibility**: Windows Explorer may not show all tags when converting from FLAC to WAV due to WAV format limitations

3. **WAV ALBUMARTIST**: No native WAV INFO support for ALBUMARTIST; requires ID3v2 chunk

4. **Opus cover art**: Cover art requires METADATA_BLOCK_PICTURE in Opus container, not all tools handle this correctly

5. **CJK character encoding**: Some older players may not display CJK characters correctly in certain formats

---

## SECTION 3: FORMAT-SPECIFIC QUIRKS

### 3.1 MP3/ID3 Quirks

#### ID3v2 Unsynchronization

**Issue:**
ID3v2.3 frames containing `0xFF 0xE0` byte sequence (which looks like MP3 sync word) must be "unsynchronized" — `0x00` is inserted after `0xFF`.

**Impact:**
- Some players crash or play wrong audio if unsynchronization not done
- Files may be unplayable on some devices

**DBpoweramp behavior:**
- Automatically handles unsynchronization for ID3v2.3
- ID3v2.4 has different handling

**R2025-02-07:**
- "ID3 do not toggle ID3v2.4 unsynchronization on frames not being altered" — fix for regression

---

#### Multi-Value Artist in MP3

**Issue:**
ID3v2.2 (ID3v1 era) had `TP1` frame for artist. ID3v2.3+ has `TPE1`. Multiple artists require either:
1. Multiple TPE1 frames (technically invalid but supported)
2. Semicolon-separated single frame (common)
3. Null-separated values in v2.4 (rarely used)

**DBpoweramp behavior:**
- Uses `"; "` separator for multiple artists
- R2025-07-21 fix: Multi-artist now displays `'; '` between values in tag editor

---

#### LAME Encoder Tag Compatibility

**Issue:**
LAME writes a special tag at the end of MP3 files containing:
- Encoder version
- Encoder delay/padding
- ReplayGain info
- Useful for gapless playback

**DBpoweramp behavior:**
- R16.6 (January 2019): "Drops 'encoding tool' tag on converting"
- This means the LAME tag info is removed during conversion, losing:
  - Original LAME version info
  - Some ReplayGain info (stored in LAME tag)
  - Gapless info

**Impact:**
- Gapless playback may not work correctly for re-encoded files
- Some players rely on LAME tag for ReplayGain

---

### 3.2 FLAC Metadata Quirks

#### METADATA_BLOCK_PICTURE Format

**Issue:**
FLAC embedded cover art uses METADATA_BLOCK_PICTURE, which is a binary block, not raw image data.

**Format:**
```
- picture_type: uint32 (big-endian)
- mime_length: uint32
- mime_type: string (mime_length bytes)
- description_length: uint32
- description: string (description_length bytes)
- width: uint32
- height: uint32
- color_depth: uint32
- color_count: uint32
- data_length: uint32
- data: image bytes (data_length bytes)
```

**Common mistake:**
Some tools write raw JPEG bytes as the `data` without the proper block header. This creates an invalid FLAC file that some players reject.

**DBpoweramp behavior:**
- Correctly implements METADATA_BLOCK_PICTURE
- Sets picture_type=3 for front cover

---

#### FLAC ID3 vs Vorbis Comments

**Issue:**
FLAC files can have both ID3v2 tags (at beginning) and Vorbis Comments (at end). Most players expect Vorbis Comments.

**DBpoweramp behavior:**
- Default: Writes Vorbis Comments
- Can be configured to write ID3v2 (but shouldn't be)
- R2022-08-09: Fixed handling of album names starting with `=`

---

#### Duplicate Vorbis Comment Tags

**Issue:**
Some taggers (notably older versions of some tools) write duplicate tags:
```
ARTIST=Artist1
ARTIST=Artist1
ARTIST=Artist1
```

**DBpoweramp behavior:**
- R2022-08-09: Fixed reading duplicate Album Artist tags in Ogg/Opus
- Removes duplicates when reading
- Prevents accumulation of duplicate tags

---

### 3.3 AAC/M4A Quirks

#### iTunes Atom Structure

**Issue:**
MP4/M4A files have a complex nested atom structure. Tags are stored in `moov/udta/meta` atoms with specific hierarchies.

**Key atoms:**
| Atom | Purpose |
|------|---------|
| ©nam | Title |
| ©ART | Artist |
| ©alb | Album |
| aART | Album Artist |
| ©wrt | Composer |
| trkn | Track number (as tuple) |
| diskn | Disc number (as tuple) |
| ©day | Date |
| ©gen | Genre |
| covr | Cover art |
| @wrk | Work |
| @mvn | Movement name |
| @mvi | Movement number |
| @mvc | Movement count |

---

#### iTunes Compilation Flag (cpil)

**Issue:**
The `cpil` atom (compilation) requires special handling:
- Value is boolean (1=true, 0=false)
- Setting to 0 should remove the atom, not write "0"

**R2025-12-25:**
- "dMCShell: Remove compilation=0 when writing tags"

This fixes an issue where writing `cpil=0` incorrectly marked files as compilations.

---

#### Fragmented MP4 Files

**Issue:**
Some M4A files (especially from video editing) have fragmented atom structures that older parsers can't handle.

**R2026-01-31:**
- "M4A/MP4 tagging tech update, proper handling of fragmented MP4"
- "mp4/m4a update, more efficient parsing of very large files"

---

#### m4a grup Tag Renamed

**Issue:**
The `grup` atom (grouping) was causing issues with iTunes compatibility.

**R17.0 (April 2020):**
- "m4a grup tag renamed to @grp as it was causing issues for iTunes"

---

### 3.4 WAV/RIFF Quirks

#### Multiple Tag Systems

WAV files support three different metadata systems:

| System | Location | Field Coverage | Compatibility |
|--------|----------|----------------|---------------|
| INFO chunk | Within RIFF | INAM, IART, IPRD, IGNR, ICRD, ITRK, ICMT, ISFT, etc. | Universal |
| ID3v2 chunk | At file start | Full ID3v2 support | Widely supported |
| LIST-INFO | Alternative INFO | Same as INFO chunk | Limited |

**Limitation:**
- ALBUMARTIST not supported in INFO chunk
- Cover art only in ID3v2 chunk
- ReplayGain only in ID3v2 chunk
- Some players only read one system

**DBpoweramp default:**
- Tag Creation: LIST & ID3 (both systems)
- Writes to both for maximum compatibility

---

#### Corrupted WAV Header Handling

**R2025-11-12:**
- "Better compatibility reading malformed wave formats (where cbsize was set larger than the actual riff block size)"

---

### 3.5 Opus Quirks

#### R128 vs ReplayGain

**Critical issue:**
Opus uses R128 tags, NOT ReplayGain tags. This is a fundamental difference.

**R128 format:**
- Integer value in Q7.8 fixed-point format
- Reference level: -23 LUFS
- Formula: `R128_val = round((ReplayGain_dB + 5.0) * 256)`

**ReplayGain format:**
- String value like "-6.20 dB"
- Reference level: -18 dB over CD peak

**DBpoweramp behavior:**
- When converting TO Opus: converts ReplayGain to R128
- When converting FROM Opus: converts R128 to ReplayGain
- Also writes ReplayGain tags for compatibility with non-R128-aware players

---

#### METADATA_BLOCK_PICTURE in Opus

**Issue:**
Opus stores cover art as base64-encoded METADATA_BLOCK_PICTURE in a Vorbis Comment field.

**Format:**
```
METADATA_BLOCK_PICTURE=base64(METADATA_BLOCK_PICTURE_binary)
```

**Common mistake:**
Some tools write raw image data or incorrectly format the block.

---

#### Opus Encoder Delay/Padding

**Issue:**
Opus files have encoder delay in the first samples and padding in the last samples. This is stored in the container header, not in tags.

**DBpoweramp behavior:**
- Reads delay/padding from source
- Writes delay/padding to output
- Gapless playback works correctly

---

### 3.6 WMA/ASF Quirks

#### Limited Tag Support

**Issue:**
WMA files have limited metadata compared to other formats.

**Supported tags:**
- Title, Author, Description, Copyright
- WM/AlbumTitle, WM/AlbumArtist, WM/Composer
- WM/TrackNumber, WM/PartOfSet
- WM/Year
- WM/Genre
- WM/Picture
- WM/UserWebURL
- WM/MediaClassPrimaryID, WM/MediaClassSecondaryID

**Unsupported:**
- Sort fields
- Work/Movement
- Some custom fields

**DBpoweramp behavior:**
- Maps standard fields correctly
- Custom fields may be lost

---

### 3.7 AIFF Quirks

#### ID3v2 Chunk Location

**Issue:**
AIFF files store ID3v2 tags in a chunk at the beginning of the file (before FORM chunk).

**Format:**
```
[ID3 chunk] → [FORM AIFF chunk] → [audio data]
```

**Compatibility:**
- Most modern players support ID3v2-in-AIFF
- Some older players don't

**R14.4 (February 2013):**
- "AIFF tagging fix (could indicate file was corrupted)"

---

## SECTION 4: TAG SYSTEM QUIRKS

### 4.1 ID3v2 Unsynchronization Issues

**Issue:**
ID3v2.3 frames containing `0xFF` bytes must be escaped to prevent confusion with MP3 sync words.

**Rules:**
- After `0xFF`, insert `0x00` (escape byte)
- Some decoders automatically remove these
- Some don't

**R2025-02-07:**
- "ID3 do not toggle ID3v2.4 unsynchronization on frames not being altered"
- Fixes regression that caused encoding issues

---

### 4.2 UTF-8 BOM Handling

**Issue:**
Some tools write UTF-8 BOM (`EF BB BF`) at the start of text fields, which can cause display issues.

**DBpoweramp behavior:**
- Does not write BOM in text fields
- Handles BOM correctly when reading (ignores it)

---

### 4.3 Multi-Tag System Conflicts

**Issue:**
Some files have multiple tag systems simultaneously:
- MP3: ID3v2 + ID3v1 + APEv2
- FLAC: Vorbis Comments + ID3v2
- WAV: INFO chunk + ID3v2

**Priority question:**
When multiple systems are present, which takes precedence?

**DBpoweramp behavior:**
- Generally prefers the most complete/written tag
- ID Tag Update utility can consolidate tags
- Can remove specific tag systems

---

### 4.4 APEv2 in MP3 Issues

**Issue:**
Early versions of EAC wrote APEv2 tags to MP3 files alongside ID3v1 tags. Modern players may:
- Ignore APEv2
- Read APEv2 instead of ID3
- Get confused by dual-tag files

**R17.0 (April 2020):**
- "Fixed duplication of album art on MP3 files having both ID3v2 and APE tags"

---

### 4.5 Genre Number vs String Issues

**Issue:**
ID3v1 genres are stored as numbers 0-191. ID3v2 can store:
1. Numbers: `(17)`
2. Strings: `Rock`
3. Both: `(17)Rock`

**DBpoweramp behavior:**
- Never writes genre numbers
- Always writes freeform genre strings
- When reading: resolves numbers to strings

**R2025-06-05:**
- "Tag Editor includes multiline editboxes for comment/lyrics fields"
- Genre display/editing improved

---

## SECTION 5: CD RIPPING QUIRKS

### 5.1 AccurateRip Confidence Quirks

**Issue:**
AccurateRip provides confidence levels for rip accuracy:
- Confidence 0-100+
- Higher = more verified
- Some drives/report submissions more accurate

**DBpoweramp behavior:**
- Shows confidence level in rip log
- "Secure" mode requires minimum confidence
- R2024-04-01: Option to "mark track as error 'if not verified by accuraterip'"

---

### 5.2 Drive Offset Database Issues

**Issue:**
Each CD drive has an offset (samples read early/late). Wrong offset causes inaccurate results.

**DBpoweramp behavior:**
- Uses database of known offsets
- "Drive offset" setting in configuration
- R2025-11-12: ARDFTSRC resampler handles offsets correctly

---

### 5.3 Secure Mode vs Certain Drives

**Issue:**
Some drives have issues with certain Secure Ripping features:
- C2 error pointers (not all drives support)
- Read retry behavior varies

**R2024-02-01:**
- Improved compatibility with various drives

**Workaround:**
- Disable "Use C2 error pointers" for problematic drives
- Use secure ripping vs quick rip based on disc quality

---

### 5.4 Metadata Provider Quirks

**Issue:**
Metadata providers may return inconsistent data:
- Different spellings
- Different capitalization
- Different field names
- Missing fields
- Extra fields

**R2025-06-05:**
- "CD Ripper rips to .tmp instead of ._ by default (then renames after ripping album)"
- Prevents partial files from being visible

**R2024-05-30:**
- "ID Tags from providers unicode \u2010 is replaced with -"
- "weird unicode spaces are standardized to ' '"

---

## SECTION 6: WORKAROUNDS REFERENCE

### 6.1 Quick Workarounds

| Issue | Workaround | Command/Action |
|-------|------------|----------------|
| FLAC has ID3 tags instead of Vorbis | Rewrite tags | Use [ID Tag Update] utility |
| Album art causes problems | Remove before convert | Delete art, convert, re-add |
| Explorer crashes | Disable shell | Uncheck popup, thumbnail options |
| Multi-artist not showing | Edit display | Tag editor now shows `'; '` |
| WAV missing ALBUMARTIST | Convert to FLAC | Use FLAC for full tag support |
| Genre shows as number | Fix tags | Use [ID Tag Update] to resolve |
| Corrupted WV files | Disable popup | Uncheck popup info tips |
| ID3v1 present | Upgrade to v2 | Use [ID Tag Update] |

### 6.2 Batch Fix Scripts

#### Fix FLAC ID3 tags → Vorbis Comments

```bash
# In Batch Converter:
# 1. Select all affected FLAC files
# 2. Choose encoder: [ID Tag Update]
# 3. Configure to use Vorbis Comments
# 4. Convert (only tags updated, no re-encoding)
```

#### Remove duplicate tags

```bash
# Use [ID Tag Update] utility codec
# Options: Deletions → [specific tag] → [remove if duplicate]
```

#### Convert all genres from numbers to strings

```bash
# Use [ID Tag Update] utility codec
# Options: Manipulation → Genre → [keep as-is, already resolved]
# Actually: The utility doesn't need special action - 
# DBpoweramp always writes freeform genres
```

#### Fix multi-value artist display

```bash
# R2025-07-21+:
# Just open in Tag Editor, values now show '; ' between items
```

---

## SECTION 7: VERSION REFERENCE

### 7.1 Complete Version History

| Version | Date | Major Changes |
|---------|------|--------------|
| R21.x | 2025-2026 | Current, ARDFTSRC, 64-bit only |
| R20.x | 2024 | AccurateRip metadata, ARDFTSRC |
| R19.x | 2023 | Decoders update, DSD |
| R18.x | 2022 | Modernization, FLAC 1.4 |
| R17.x | 2020-2022 | Multi-core, Quick Convert |
| R16.x | 2016-2019 | UI redesign, 64-bit |
| R15.x | 2014-2015 | 64-bit, Sort tags |
| R14.x | 2011-2013 | AccurateRip v2 |

### 7.2 End-of-Life Versions

| Version | EOL Date | Notes |
|---------|----------|-------|
| R14 and earlier | 2020 | 64-bit required |
| 32-bit versions | Nov 2025 | 64-bit only after R2025-11-12 |

---

## SECTION 8: SOURCE CITATIONS

### 8.1 Forum Threads Referenced

| Thread | URL | Date |
|--------|-----|------|
| Tags not showing in Mp3tag | https://hydrogenaudio.org/index.php/topic,68944.0.html | Various |
| GetPopupInfo crash | https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/326906-crash-dbpoweramp-getpopupinfo-has-stopped-working | Aug 2024 |
| GetPopupInfo crash 2 | https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/337596-error-getpopupinfo-has-stopped-working | Jan 2026 |
| ALAC to FLAC no tags | https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/331261-alac-to-flac-conversion-no-tags-in-windows-explorer | 2022 |
| ID Tag Update Help | https://dbpoweramp.com/Help/dMC/idtagupdate.html | Documentation |

### 8.2 Official Resources

| Resource | URL |
|----------|-----|
| DBpoweramp Website | https://www.dbpoweramp.com/ |
| Release Notes | https://versions.dbpoweramp.com/ |
| Official Forums | https://forum.dbpoweramp.com/ |
| dBpoweramp Reddit | https://www.reddit.com/r/dBpoweramp/ |

### 8.3 Comparison Resources

| Resource | URL |
|----------|-----|
| Hydrogenaudio Comparison | https://hydrogenaudio.org/ |
| fre:ac vs dBpoweramp | https://appmus.com/vs/dbpoweramp-vs-bonkenc |
| Nero Comparison | https://pcai.nero.com/blog/nero-eac-dbpoweramp-best-cd-ripper-with-metadata |

---

*Document: 74_Known_DBpoweramp_Bugs_and_Quirks.md*
*Generated from: Community bug reports and version history*
*Version: 1.0*
