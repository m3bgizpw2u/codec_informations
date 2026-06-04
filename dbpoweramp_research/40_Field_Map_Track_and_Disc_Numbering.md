# Field Mapping: Track and Disc Numbering Across Formats

## 1. Format-Specific Storage Mechanisms

### 1.1 ID3v2 (MP3)

**TRCK Frame (Track Number/Position in Set)**
- **Format:** Numeric string, optionally extended with "/" separator
- **Values:** `"5"` or `"5/12"` (track/total)
- **Source:** [ID3v2.4 spec](https://id3.org/id3v2.4.0-frames) — "The 'Track number/Position in set' frame is a numeric string containing the order number of the audio-file on its original recording. This MAY be extended with a '/' character and a numeric string containing the total number of tracks/elements on the original recording. E.g., '4/9'."
- **Leading zeros:** Permitted but variable; "04" and "4" both valid

**TPOS Frame (Part of a Set)**
- **Format:** Same as TRCK — numeric string with optional "/" separator
- **Values:** `"1"` or `"1/2"` (disc/total)
- **Source:** [ID3v2.4 spec](https://id3.org/id3v2.4.0-frames) — "The 'Part of a set' frame is a numeric string that describes which part of a set the audio came from. This frame is used if the source described in the 'TALB' frame is divided into several mediums, e.g. a double CD."

**ID3v2.3 vs ID3v2.4 Difference:**
- ID3v2.3 also uses TRCK and TPOS with identical semantics
- TYER (year only) vs TDRC (full date) applies to date fields, not track/disc

### 1.2 Vorbis Comment / FLAC

**TRACKNUMBER Field**
- **Format:** Single integer string
- **Values:** `"5"`, `"12"`, `"01"` (leading zeros common but optional)
- **Source:** [Xiph VorbisComment spec](https://wiki.xiph.org/VorbisComment)

**TOTALTRACKS / TRACKTOTAL Field**
- **Format:** Single integer string (total count only)
- **Values:** `"12"`, `"10"`
- **Source:** Both `TOTALTRACKS` and `TRACKTOTAL` are used interchangeably; tools should read both
- **Source:** [Hydrogenaudio Tag Mapping](https://wiki.hydrogenaudio.org/index.php?title=Tag_Mapping)

**DISCNUMBER / DISCTOTAL / TOTALDISCS**
- **Format:** Single integer strings
- **Values:** `"1"` / `"2"` or `"1"` / `"2"` (disc number / total discs)
- **Source:** [Picard Tag Mapping](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html)

### 1.3 MP4 / M4A / iTunes

**trkn Atom (Track Number)**
- **Format:** 6 bytes binary structure `[reserved, reserved, track_uint16, total_uint16, reserved, reserved]`
- **Values:** `[(track, total)]` tuple in mutagen, e.g., `[(5, 12)]`
- **Source:** [Mutagen MP4 docs](https://mutagen.readthedocs.io/en/latest/api/mp4.html) — "Tuples of ints (multiple values per key are supported): 'trkn' – track number, total tracks"
- **Constraints:** Both track and total are 16-bit unsigned integers (0-65535)

**disk Atom (Disc Number)**
- **Format:** Same 6-byte binary structure as trkn
- **Values:** `[(1, 2)]` for disc 1 of 2
- **Source:** [Mutagen MP4 docs](https://mutagen.readthedocs.io/en/latest/api/mp4.html) — "'disk' – disc number, total discs"

### 1.4 APEv2 (Monkey's Audio, WavPack, MusePack)

**Track Field**
- **Format:** Either single integer `"5"` or slash-separated `"5/12"`
- **Behavior:** Some tools write `"5/12"`, others write separate Track and Track复兴 fields
- **Source:** [Mutagen APEv2 spec](https://mutagen-specs.readthedocs.io/en/latest/apev2/apev2.html) — values are UTF-8 strings
- **Source:** [2ManyRobots/Yate](https://2manyrobots.com/YateResources/InAppHelp/PrefsAudioFilesAPE.html) — "When the Write the Track and Track Count fields to the Track mapping option is set, Yate will Write the low level Track mapping as track number{/track count}"

**Disc / DiscNumber Field**
- **Format:** Plain integer or slash-separated `"1/2"`
- **Source:** [Picard Tag Mapping](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html)

### 1.5 RIFF INFO / WAV

**ITRK Field (Track Number)**
- **Format:** Numeric string
- **Values:** `"5"` or `"5/12"` (some implementations support slash separator)
- **Source:** [Kid3 Handbook](https://kid3.sourceforge.io/kid3_en.html) — "ITRK" maps to "Track Number" in unified view

**IPRT Field (Part of Track Total)**
- **Format:** Numeric string with optional "/" separator
- **Source:** [Hydrogenaudio Tag Mapping](https://wiki.hydrogenaudio.org/index.php?title=Tag_Mapping)

### 1.6 ASF / WMA

**WM/TrackNumber Field**
- **Format:** String integer `"5"` or `"5/12"` format
- **Critical issue:** Older WMA implementations used **zero-indexed** track numbers
- **Source:** [ReNamer forum](https://www.den4b.com/forum/viewtopic.php?id=2069) — "Converted WMA_TrackNo meta tag from 0-based to 1-based"
- **Source:** [Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/wmformat/id3-tag-support) — "WM/TrackNumber" maps to ID3 TRK/TRCK

**WM/PartOfSet Field**
- **Format:** String `"1/2"` (disc/total)
- **Source:** [Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/wmformat/id3-tag-support) — "WM/PartOfSet" maps to ID3 TPA/TPOS

---

## 2. Cross-Format Mapping Directions

### 2.1 ID3 "5/12" → Vorbis

**Source (ID3v2):**
```
TRCK = "5/12"
```

**Destination (Vorbis/FLAC):**
```
TRACKNUMBER=5
TOTALTRACKS=12
```
**OR:**
```
TRACKNUMBER=5
TRACKTOTAL=12
```

**Canonical behavior:**
- Split on "/" separator
- First part → TRACKNUMBER
- Second part → TOTALTRACKS (or TRACKTOTAL)
- Leading zeros stripped: "05" → "5"

**Edge cases:**
- `"5/"` (trailing slash) → TRACKNUMBER=5, no TOTALTRACKS
- `"/12"` (leading slash) → TRACKNUMBER=0? (invalid, reject)
- `"5/12/3"` (extra slashes) → parse first two parts only

### 2.2 Vorbis TRACKNUMBER=5 + TOTALTRACKS=12 → ID3

**Source (Vorbis):**
```
TRACKNUMBER=5
TOTALTRACKS=12
```

**Destination (ID3v2):**
```
TRCK = "5/12"
```

**Canonical behavior:**
- Combine with "/" separator
- No leading zeros: "5/12" not "05/12"

**Edge cases:**
- Only TRACKNUMBER present → TRCK="5" (no slash)
- Only TOTALTRACKS present → ignore (can't construct valid TRCK)
- TRACKNUMBER="" (empty) → skip field

### 2.3 MP4 trkn bytes → Vorbis

**Source (MP4 atom):**
```
trkn = [(5, 12)]  # binary: [0, 0, 0x00, 0x05, 0x00, 0x0C, 0, 0]
```

**Destination (Vorbis/FLAC):**
```
TRACKNUMBER=5
TOTALTRACKS=12
```

**Parsing:**
- Read uint16 at bytes 2-3: track number
- Read uint16 at bytes 4-5: total tracks
- Both are big-endian unsigned 16-bit integers

**Edge cases:**
- total=0 → no TOTALTRACKS written
- total=65535 → max valid value
- Malformed (less than 6 bytes) → partial read or error

### 2.4 APEv2 "5/12" → ID3

**Source (APEv2):**
```
Track = "5/12"
```

**Destination (ID3v2):**
```
TRCK = "5/12"
```

**Behavior:**
- APEv2 typically preserves the slash-separated format
- Some APEv2 tools may store as separate "Track" and "Track复兴" fields

### 2.5 Disc Number Mapping (Same Patterns)

| Source Format | Source Value | → ID3 | → Vorbis | → MP4 |
|---------------|--------------|-------|----------|-------|
| ID3 TPOS | "2/3" | TPOS="2/3" | DISCNUMBER=2, TOTALDISCS=3 | disk=[(2,3)] |
| Vorbis | DISCNUMBER=2, TOTALDISCS=3 | TPOS="2/3" | DISCNUMBER=2, TOTALDISCS=3 | disk=[(2,3)] |
| MP4 | disk=[(2,3)] | TPOS="2/3" | DISCNUMBER=2, TOTALDISCS=3 | disk=[(2,3)] |
| APEv2 | Disc="2/3" | TPOS="2/3" | DISCNUMBER=2, TOTALDISCS=3 | disk=[(2,3)] |

---

## 3. Leading Zeros Handling

### 3.1 Read Behavior

**ID3v2:**
- TRCK="05" and TRCK="5" are both valid
- Most readers treat them as equivalent integers
- kid3 preserves the string representation as written

**Vorbis:**
- TRACKNUMBER="05" and TRACKNUMBER="5" both valid
- No canonical enforcement

**MP4:**
- Binary format — no string representation to preserve zeros

### 3.2 Write Behavior

**Question: Does DBpoweramp write "05" or "5"?**

Based on industry practice:
- **Most tools write "5"** (canonical integer form)
- **iTunes sometimes writes "05"** for display purposes
- **DBpoweramp behavior:** Unknown from public documentation; assume canonical "5" form

**Recommendation:**
- Read both "05" and "5" as equivalent
- Write canonical "5" form (no leading zeros)

### 3.3 Code Example: Normalizing Track Numbers

```python
def normalize_track_number(trck_value: str) -> tuple[str, str | None]:
    """
    Parse ID3 TRCK "5/12" format into track and total.
    Returns (track_str, total_str_or_none).
    """
    if '/' in trck_value:
        parts = trck_value.split('/', 1)
        track = str(int(parts[0]))  # Strip leading zeros
        total = str(int(parts[1]))  # Strip leading zeros
        return track, total
    else:
        return str(int(trck_value)), None

def format_trck_for_id3(track: int, total: int | None) -> str:
    """Format track/total for ID3 TRCK frame."""
    if total is not None:
        return f"{track}/{total}"
    return str(track)
```

---

## 4. Multi-Disc Sets

### 4.1 Storage Variations

**Format 1: Combined disc+track**
```
TRCK = "1/12"   # Track 1 of 12
TPOS = "2/3"    # Disc 2 of 3
```

**Format 2: Prefixed track numbers (rare)**
```
TRACKNUMBER = "CD2:5"  # Non-standard, used by some tools
```

**Format 3: Multiple fields**
```
DISCNUMBER = 2
TRACKNUMBER = 5
TOTALTRACKS = 12
TOTALDISCS = 3
```

### 4.2 Edge Cases for Multi-Disc

**Edge Case 1: Disc-only file (no tracks)**
- Single file spanning entire disc
- TPOS="1/2", no TRCK
- Source: [Hydrogenaudio](https://wiki.hydrogenaudio.org/index.php?title=Tag_Mapping)

**Edge Case 2: Hidden track zero**
- Some classical releases number tracks per disc, not per album
- TRACKNUMBER might start at 0 on disc 2
- Source: [MusicBrainz documentation](https://community.metabrainz.org/t/converting-transcoding-and-keeping-mapping/686122)

**Edge Case 3: Sampler with multiple discs**
- Compilation spanning multiple discs
- TCMP=1 + TPOS="2/3"
- Track numbering restarts on each disc

**Edge Case 4: Box set with separate albums**
- Each disc is a separate album
- No TPOS field
- Source: [Kid3 Handbook](https://kid3.sourceforge.io/kid3_en.html)

**Edge Case 5: Digitally released as single album**
- Originally separate EPs, later combined
- Total track count reflects combined release
- Total disc count reflects physical media

---

## 5. Edge Cases and Corner Cases

### 5.1 Invalid Values

| Input | Handling | Recommendation |
|-------|----------|----------------|
| TRCK="0" | Valid (track 0 unusual but possible) | Warn, but accept |
| TRCK="-1" | Invalid negative | Reject, skip field |
| TRCK="abc" | Invalid non-numeric | Reject, skip field |
| TRCK="5/0" | Total cannot be zero | Write as "5" (no slash) |
| TRCK="5/99999" | Exceeds practical range | Accept, truncate if needed |

### 5.2 Overflow and Bounds

| Format | Track Max | Total Max | Notes |
|--------|-----------|-----------|-------|
| ID3v2 | 65535 | 65535 | Numeric string, theoretically unlimited |
| MP4 | 65535 | 65535 | uint16 limit |
| Vorbis | Unlimited | Unlimited | String-based |
| APEv2 | Unlimited | Unlimited | String-based |

### 5.3 Unicode and Encoding

- All numeric track/disc values are ASCII digits (0-9)
- No localization or Unicode digit variations
- Encoding: UTF-8 for Vorbis/APEv2, specified encoding for ID3v2

### 5.4 Tag Ordering

- TRCK and TPOS position in tag header is not significant
- Some players rely on field presence, not order

### 5.5 Multiple Values

- **MP4 trkn/disk:** Supports list of tuples for multi-value (rarely used)
- **ID3v2:** Single TRCK frame per tag; multiple = separate frames
- **Vorbis:** Multiple TRACKNUMBER values possible but ambiguous

---

## 6. DBpoweramp Specific Behavior

### 6.1 What We Know

Based on forum research:

**DBpoweramp Configuration:**
- Has settings to control whether track/total are written
- Can disable "Update 'Source', 'Encoder', 'Encoded By' & 'Encoder Settings' ID tags"
- Source: [dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/batch-converter/33132-flac-aiff-mp3-how-can-i-disable-the-writing-of-the-encoder-encoder-fields)

**Track/Disc Handling:**
- Standard CD ripping writes TRCK and TPOS from CDDB/CDтекст data
- On conversion, reads source tags and writes destination tags

### 6.2 What We Don't Know

- Does DBpoweramp write TRCK="5" or TRCK="5/12"?
- Does DBpoweramp read both TOTALTRACKS and TRACKTOTAL?
- How does DBpoweramp handle MP4 trkn parsing?

### 6.3 User Impact Assessment

**Would a user notice any difference?**

| Scenario | User Impact | Severity |
|----------|-------------|----------|
| DBpoweramp writes "5" instead of "5/12" | Some players show "5" only | Low |
| DBpoweramp doesn't write TOTALTRACKS | Track total lost in player | Medium |
| MP4 trkn parsing incorrect | Disc number wrong | High |
| Leading zeros stripped | Visual difference only | Low |

---

## 7. Code Examples for Cross-Format Conversion

### 7.1 Python Implementation (Using Mutagen)

```python
from mutagen.mp4 import MP4
from mutagen.flac import FLAC
from mutagen.mp3 import MP3
from mutagen.oggvorbis import OggVorbis

def parse_trck(value: str) -> tuple[int, int | None]:
    """Parse ID3v2 TRCK value into track and total."""
    if '/' in value:
        parts = value.split('/', 1)
        return int(parts[0]), int(parts[1])
    return int(value), None

def format_vorbis_track(track: int, total: int | None) -> dict:
    """Format track/total for Vorbis comment."""
    result = {'TRACKNUMBER': str(track)}
    if total is not None:
        result['TOTALTRACKS'] = str(total)
    return result

def format_mp4_trkn(track: int, total: int | None) -> list:
    """Format track/total for MP4 trkn atom."""
    return [(track, total or 0)]

def convert_track_tags(source_file: str, source_format: str, 
                       dest_file: str, dest_format: str) -> None:
    """Convert track/disc tags between formats."""
    
    # Read source tags
    if source_format == 'mp3':
        audio = MP3(source_file)
        trck = str(audio['TRCK'][0]) if 'TRCK' in audio else None
        tpos = str(audio['TPOS'][0]) if 'TPOS' in audio else None
    elif source_format == 'flac':
        audio = FLAC(source_file)
        trck = audio['TRACKNUMBER'][0] if 'TRACKNUMBER' in audio else None
        total = audio.get('TOTALTRACKS', audio.get('TRACKTOTAL', [None]))[0]
        tpos_disc = audio['DISCNUMBER'][0] if 'DISCNUMBER' in audio else None
        tpos_total = audio.get('TOTALDISCS', audio.get('DISCTOTAL', [None]))[0]
    elif source_format == 'm4a':
        audio = MP4(source_file)
        trkn = audio['trkn'][0] if 'trkn' in audio else None
        trck = f"{trkn[0]}/{trkn[1]}" if trkn else None
        disk = audio['disk'][0] if 'disk' in audio else None
        tpos = f"{disk[0]}/{disk[1]}" if disk else None
    
    # Parse values
    track_num, track_total = parse_trck(trck) if trck else (None, None)
    disc_num, disc_total = parse_trck(tpos) if tpos else (None, None)
    
    # Write destination tags
    if dest_format == 'mp3':
        dest = MP3(dest_file)
        if track_num is not None:
            dest['TRCK'] = format_trck_for_id3(track_num, track_total)
        if disc_num is not None:
            dest['TPOS'] = format_trck_for_id3(disc_num, disc_total)
    elif dest_format == 'flac':
        dest = FLAC(dest_file)
        if track_num is not None:
            dest['TRACKNUMBER'] = [str(track_num)]
            if track_total:
                dest['TOTALTRACKS'] = [str(track_total)]
        if disc_num is not None:
            dest['DISCNUMBER'] = [str(disc_num)]
            if disc_total:
                dest['TOTALDISCS'] = [str(disc_total)]
    elif dest_format == 'm4a':
        dest = MP4(dest_file)
        if track_num is not None:
            dest['trkn'] = format_mp4_trkn(track_num, track_total)
        if disc_num is not None:
            dest['disk'] = format_mp4_trkn(disc_num, disc_total)
```

---

## 8. Validation Checklist

- [ ] Parse TRCK "5/12" correctly into track=5, total=12
- [ ] Write TRCK "5/12" from track=5, total=12
- [ ] Handle TRCK "5" (no total) correctly
- [ ] Parse MP4 trkn [(5, 12)] correctly
- [ ] Write MP4 trkn correctly as binary atom
- [ ] Read both Vorbis TOTALTRACKS and TRACKTOTAL
- [ ] Write Vorbis TOTALTRACKS (canonical) not TRACKTOTAL
- [ ] Handle leading zeros: "05" → "5"
- [ ] Handle disc number TPOS same as TRCK
- [ ] Validate range: 0-65535 for MP4
- [ ] Handle missing fields gracefully
- [ ] Preserve empty vs missing distinction

---

## 9. Source Citations

| # | Source | Key Information |
|---|--------|-----------------|
| 1 | [ID3v2.4 spec](https://id3.org/id3v2.4.0-frames) | TRCK and TPOS format specification |
| 2 | [Picard Tag Mapping](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html) | Cross-format field names |
| 3 | [Hydrogenaudio Tag Mapping](https://wiki.hydrogenaudio.org/index.php?title=Tag_Mapping) | Comprehensive format matrix |
| 4 | [Mutagen MP4 docs](https://mutagen.readthedocs.io/en/latest/api/mp4.html) | MP4 trkn atom structure |
| 5 | [Mutagen APEv2 spec](https://mutagen-specs.readthedocs.io/en/latest/apev2/apev2.html) | APEv2 format details |
| 6 | [Microsoft WMA docs](https://learn.microsoft.com/en-us/windows/win32/wmformat/id3-tag-support) | WMA/ASF tag mapping |
| 7 | [Kid3 Handbook](https://kid3.sourceforge.io/kid3_en.html) | Unified field mapping table |
| 8 | [ReNamer forum](https://www.den4b.com/forum/viewtopic.php?id=2069) | WMA zero-based track number issue |
| 9 | [dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/batch-converter/33132) | DBpoweramp configuration options |
| 10 | [Xiph VorbisComment](https://wiki.xiph.org/VorbisComment) | Vorbis comment format |
