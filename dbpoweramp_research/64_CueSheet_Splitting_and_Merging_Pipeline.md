# CueSheet Splitting and Merging Pipeline
*Generated: 2026-06-04 | Sources: 8 | Confidence: High*

## Executive Summary

CUE sheets are text-based files that describe how to split a single audio file (image) into individual tracks, preserving album structure including pregaps, indexes, and metadata. DBpoweramp supports CUE sheet splitting through its converter interface, handling various CUE sheet formats including gaps appended, gaps prepended, gaps left out, and HTOA (Hidden Track One Audio). Frame-accurate splitting is achieved using MM:SS:FF format where each frame is 1/75th of a second, allowing precise positioning within audio files.

## 1. CUE Sheet Format Overview

### 1.1 What is a CUE Sheet?
A CUE sheet (from CDRWIN/CUETools terminology) is a plain-text file containing:
- Reference to source audio file(s)
- Track definitions with numbers and types
- Index points for track boundaries
- Optional metadata (titles, performers, ISRCs)

### 1.2 Standard CUE Sheet Commands
| Command | Description | Scope |
|---------|-------------|-------|
| `FILE` | Source audio filename and format | Global |
| `TRACK` | Track number and mode (AUDIO/DATA) | Track |
| `TITLE` | Album or track title | Global/Track |
| `PERFORMER` | Album or track artist | Global/Track |
| `SONGWRITER` | Composer | Global/Track |
| `INDEX` | Position within FILE (MM:SS:FF) | Track |
| `PREGAP` | Length of pre-gap silence | Track |
| `POSTGAP` | Length of post-gap silence | Track |
| `FLAGS` | Track flags (SCMS, 4CH, etc.) | Track |
| `ISRC` | International Standard Recording Code | Track |
| `REM` | Comments/notes | Global/Track |

### 1.3 Supported File Types
CUE sheets typically reference:
- WAV (uncompressed PCM)
- FLAC (lossless)
- APE (Monkey's Audio)
- WV (WavPack)
- AIFF (Apple Interchange File Format)
- NRG (Nero image format)
- ISO (disc image)

## 2. Frame-Accurate Splitting

### 2.1 INDEX Format
CUE sheet INDEX values use MM:SS:FF format:
- **MM**: Minutes (0-99)
- **SS**: Seconds (0-59)
- **FF**: Frames (0-74)
- 75 frames per second
- Each frame = 1/75th second ≈ 13.33ms

### 2.2 Precision Calculation
For CD-quality audio (44.1 kHz, 16-bit stereo):
- 1 frame = 1/75 second
- Samples per frame = 44100 / 75 = 588 samples
- Bytes per frame = 588 × 2 channels × 2 bytes = 2352 bytes

### 2.3 Byte Offset Calculation
```
Byte Offset = (MM × 60 × 60 + SS × 60 + FF) × 75 × channels × bytes_per_sample
```

For 44.1kHz 16-bit stereo:
```
Offset = (MM*60*60 + SS*60 + FF) * 2352 bytes
```

### 2.4 High-Precision Splitting
While CUE sheets specify frame accuracy (1/75s), true sample-accurate splitting requires:
1. Parse INDEX to frame position
2. Calculate byte offset
3. Seek to byte position in source
4. Read from sample boundary
5. Handle formats with different sample rates

## 3. Pre-gap Handling

### 3.1 INDEX 00 vs INDEX 01
In CUE sheet terminology:
- **INDEX 00**: Pre-gap start (silence or hidden content before track)
- **INDEX 01**: Track data start (what's stored in TOC)

### 3.2 Pre-gap Processing Options

#### Option A: Append to Previous Track
- Pre-gap added to END of previous track
- Common for gapless playback
- Preserves album continuity

#### Option B: Prepend to Next Track
- Pre-gap added to START of next track
- Creates silence at end of previous track
- Standard CD-DA behavior

#### Option C: Discard (Leave Out)
- Pre-gap removed entirely
- Replaced with silence tag
- File sizes reduced

#### Option D: Preserve as Separate File
- Pre-gap extracted as separate audio file
- May be named "track00" or similar
- Required for HTOA preservation

### 3.3 Standard Pre-gap Handling
Most ripper defaults:
1. Extract audio with gaps appended
2. INDEX 00 silence included in previous track
3. INDEX 01 marks true track start
4. Gapless playback preserved

## 4. Hidden Track One Audio (HTOA)

### 4.1 What is HTOA?
Hidden Track One Audio refers to audio content stored in the pregap of Track 1 that contains actual music/speech rather than silence. Examples:
- Hidden tracks on albums (e.g., Pink Floyd "The Dark Side of the Moon")
- Bonus content in track 1 pregap
- Alternate versions or intros

### 4.2 Detection
HTOA is detected when:
- INDEX 00 has non-silent audio
- Content length > 5 seconds (configurable)
- Audio differs significantly from silence

### 4.3 HTOA Handling Modes

#### Gaps Appended + HTOA (Recommended)
- Pregaps appended to previous track
- HTOA preserved as separate file (track 00)
- Most complete preservation

#### Gaps Appended (No HTOA)
- Pregaps appended to previous track
- HTOA discarded
- Silence tag written instead

#### Gaps Prepended
- Pregaps prepended to next track
- No HTOA preservation

#### Gaps Left Out
- Pregaps discarded
- Silence tag written

### 4.4 HTOA Extraction Requirements
- Drive capable of reading pregap
- Software configured for index extraction
- Sufficient drive cache invalidation
- Not all drives/software support this

## 5. Multi-File CUE Sheets

### 5.1 One CUE to Multiple Files
Some CUE sheets reference multiple audio files:
```
FILE "disc1.wav" WAVE
  TRACK 01 AUDIO
    INDEX 01 00:00:00
  TRACK 02 AUDIO
    INDEX 01 02:30:00
FILE "disc2.wav" WAVE
  TRACK 03 AUDIO
    INDEX 01 00:00:00
```

Used for:
- Multi-disc albums
- Discs split for size limits
- DDP image files

### 5.2 Processing Multi-File CUE
1. Parse FILE commands to identify source files
2. Track which indices belong to which file
3. Process each file independently
4. Handle cross-file track numbering
5. Apply appropriate metadata

### 5.3 Embedded CUE Sheets
Some audio formats (FLAC, WAVPack) support embedded CUE:
- CUE data stored in file metadata
- No external sheet required
- Must be preserved during conversion

## 6. Metadata Source Priority

### 6.1 Priority Order
When splitting with CUE, metadata sources checked in order:
1. **CUE FILE/TITLE/PERFORMER**: Global album metadata
2. **CUE TRACK TITLE/PERFORMER**: Track-level metadata
3. **Embedded File Tags**: If source has existing tags
4. **Filename Parsing**: Fallback to filename-based guess
5. **User Override**: Manual specification

### 6.2 Metadata Extraction
For each output track:
1. Extract from TRACK level:
   - TITLE
   - PERFORMER
   - SONGWRITER
   - ISRC
   - FLAGS
2. Fall back to FILE level for album:
   - TITLE (album name)
   - PERFORMER (album artist)

### 6.3 Missing Metadata
If CUE sheet lacks metadata:
- Use embedded tags from source image
- Parse from filename if possible
- Leave blank (user can add later)

## 7. Gapless Album Silence Handling

### 7.1 What is Gapless?
Gapless playback means no silence between tracks:
- Album plays continuously
- Live recordings
- Classical works
- Concept albums

### 7.2 Requirements for Gapless
1. Pregaps included in audio data
2. Index points accurate
3. No silence added during split
4. Proper output format handling

### 7.3 Silence vs. Non-Silence
CUE splitting must distinguish:
- **True silence**: Near-zero samples, can be trimmed
- **Program silence**: Low-level audio, must be preserved
- **HTOA**: Non-silence in pregap, must be preserved

### 7.4 Format Considerations
| Format | Gapless Support |
|--------|----------------|
| FLAC | Full (preserves pregap) |
| WAV | Full (no metadata) |
| ALAC | Full (preserves pregap) |
| MP3 | Limited (gapless not native) |
| AAC | Limited (gapless not native) |

## 8. Output Options

### 8.1 One File Per Track
Split into individual track files:
- `[track] - [title].flac`
- Individual tag per file
- Standard media player compatibility

### 8.2 Keep Album as Image
Keep as single file with cue sheet:
- `[album].flac` + `[album].cue`
- Preserves original structure
- Best for archival

### 8.3 Re-tag After Split
Post-split processing:
1. Split into tracks
2. Apply metadata from CUE
3. Update filenames
4. Write to output folder

## 9. Edge Cases

### 9.1 Corrupt CUE Sheet
- Invalid INDEX format
- Missing FILE reference
- Invalid track numbers
- Recovery may be partial

### 9.2 Encoding Issues
- Non-ASCII characters in titles
- UTF-8 vs Latin-1 encoding
- Japanese/Cyrillic/Chinese characters
- Requires proper charset detection

### 9.3 Sample Rate Mismatch
- CUE assumes 44.1kHz
- Source may be different (48kHz, 96kHz)
- Offset calculation must account for sample rate
- Output resampling may be needed

### 9.4 Single WAV with Multiple INDEX
EAC "Single WAV" mode:
- All indices extracted to separate files
- Both INDEX 00 and INDEX 01 preserved
- True pregap handling

### 9.5 DDP (Disc Description Protocol)
- DVD-audio and some CD images
- Different from standard WAV
- May require special handling
- CUETools supports DDP

### 9.6 CD-ROM Data Tracks
- Mixed-mode CDs
- Data + audio on same disc
- Data tracks must be skipped
- Only AUDIO tracks extracted

### 9.7 Session CDs
- Multiple sessions on one disc
- Audio in different sessions
- Session boundary handling
- May confuse standard rippers

### 9.8 Vinyl Rip Simulations
- Pre-gap may contain noise
- INDEX 00 different from silence
- Manual review needed
- Some vinyl rips have intentional pregap

## 10. Would a User Notice a Difference?

### From DBpoweramp CUE Split vs. Other Tools

| Aspect | DBpoweramp | Other Tools |
|--------|-----------|-------------|
| **Accuracy** | Frame-accurate | May vary |
| **HTOA** | Supported | Often missing |
| **Format Support** | Extensive | Limited |
| **Metadata** | CUE → Tags | Varies |

### Audio Quality
**No difference in audio quality**:
- Same source audio extracted
- Lossless split = lossless output
- Any difference = tool error

### Album Integrity
| Handling | User Experience |
|----------|-----------------|
| Gaps appended | Gapless playback |
| Gaps discarded | Silent gaps between tracks |
| HTOA preserved | Hidden content accessible |
| HTOA discarded | Hidden content lost |

## Sources

1. [CUE Sheet - Hydrogenaudio Knowledgebase](https://wiki.hydrogenaudio.org/index.php?title=Cue_sheet)
2. [GNU ccd2cue: INDEX Command](https://www.gnu.org/software/ccd2cue/manual/html_node/INDEX-_0028CUE-Command_0029.html)
3. [Cue sheets - Kodi Wiki](https://kodi.wiki/view/Cue_sheets)
4. [TSM - Cue Sheet Syntax](https://totalsonic.net/cuesheetsyntax.html)
5. [CUETools Advanced Settings](http://cue.tools/wiki/CUETools_Advanced_Settings:_CUETools)
6. [Pregap - CUETools](https://cue.tools/wiki/Pregap)
7. [EAC Gap Settings - Hydrogenaudio](https://wiki.hydrogenaudio.org/index.php?title=EAC_Gap_Settings)
8. [CUERipper Options](http://www.cuetools.net/wiki/CUERipper_Options)
