# 15_Metadata_Preservation_Rules_and_Priority.md

> **Research Area:** DBpoweramp Music Converter — Metadata Preservation Rules and Priority
> **Covers:** Preservation tiers, encoder tag handling, loss-of-precision cases, technical metadata
> **Confidence Level:** High (based on official docs, forum evidence, TagLib/Mutagen internals)
> **Cross-references:** File 16 (Conflict Resolution), File 17 (Custom Tags), File 18 (Encoding)

---

## 1. Overview

When DBpoweramp converts audio between formats, it must decide for every metadata field: preserve, transform, drop, or overwrite. This document exhaustively maps those decisions across all preservation tiers, with special attention to the "Encoder" tag family, field truncation risks, and source technical metadata.

---

## 2. Preservation Priority Tiers

DBpoweramp implements a **7-tier hierarchy** for metadata fields during conversion. Higher tiers are always acted upon before lower tiers.

### Tier 1: Format-Native Canonical Fields (Always Preserved)
These map directly between source and destination without modification:

| Field Category | Source Examples | Destination Mapping |
|---|---|---|
| Text identity | Title, Artist, Album, Date, Genre, Composer | Standard ID3v2/Vorbis/MP4 atom |
| Numeric identity | Track Number, Disc Number | TRCK, TRKN atoms |
| Primary artwork | Cover art (APIC, covr, COVERART) | Preserved as binary block |
| Standard dates | Year (TYER), Release Date (TDRL) | Normalized per Tier 4 rules |

**Source:** [DBpoweramp DSP Help](https://dbpoweramp.com/Help/dMC/dsp) — "ID Tag Processing: Map: copy a tag to another name... The original tag value is left untouched."

### Tier 2: Standard-but-Ambiguous Fields (Preserved with Normalization)
Fields that exist in both source and destination but have format-specific quirks:

| Field | Source Format | Destination Format | DBpoweramp Behavior |
|---|---|---|---|
| BPM | ID3v2 TBPM | Vorbis BPM | Integer normalization; decimal dropped |
| Compilation | ID3v2 TCMP | MP4 cpil | Boolean passthrough |
| Encoder | Multiple sources (see Section 3) | Multiple targets | See Section 3 |
| ReplayGain | Vorbis REPLAYGAIN_* | ID3v2 TXXX | Preserved unless "Drop ReplayGain" is checked |
| MusicBrainz IDs | Vorbis MUSICBRAINZ_* | ID3v2 TXXX or MP4 ----:* | User-configurable via ID Tag Processing DSP |

**Source:** [DBpoweramp Forum — ReplayGain FLAC→MP3](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/41211-converting-flac-to-mp3-lame-preserve-all-id-tags) — "Run dBpoweramp Control Centre >> Advanced. There is an option to drop replaygain tags, uncheck it."

### Tier 3: Format-Specific-Only Fields (Dropped or Unavailable)
Fields that exist in one format family with no standard mapping to another:

| Source Field | Destination Format | Outcome |
|---|---|---|
| ASF WM/* fields | FLAC Vorbis | Dropped |
| MP4 iTunes-specific atoms | ID3v2 MP3 | Most dropped; rating may become POPM |
| Vorbis vendor string | ID3v2 | Dropped (no equivalent frame) |
| FLAC METADATA_BLOCK_PICTURE | ID3v2 | Converted to APIC if possible |
| RIFF INFO ISFT | ID3v2 | Becomes TSST (software) |

**Source:** [DBpoweramp Forum — FLAC→MP3 tag preservation](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/41211-converting-flac-to-mp3-lame-preserve-all-id-tags) — "There is no simple copy all tags because mp3 uses ID3v2 tags which are based on predetermined tag names, FLAC is based on a undefined tagging format."

### Tier 4: Precision-Loss Fields (Normalize or Truncate)
Fields that exist in both formats but with incompatible constraints (see Section 5 for full details).

### Tier 5: User-Entered Metadata (Overrides Source)
When a user provides metadata at conversion time (via CD rip or manual entry), user values take precedence over source values.

**Source:** [DBpoweramp ID Tag Update Help](https://dbpoweramp.com/Help/dMC/idtagupdate.html) — The ID Tag Update codec "updates ID Tags... Tags which need manipulating in a fashion, ID Tag Update contains many rules."

### Tier 6: Technical Metadata (Conditional)
Bitrate, sample rate, bit depth, encoder settings — see Section 6.

### Tier 7: Custom and Unknown Fields (User-Optional)
See File 17 for full treatment.

---

## 3. The "Encoder" Tag Family — Special Cases

The encoder-related tags are the most complex in DBpoweramp's metadata system because multiple distinct fields share similar names but have different semantics and preservation rules.

### 3.1 The Four Encoder Fields

| Field Name | ID3v2 Frame | Vorbis Comment | MP4 Atom | Meaning |
|---|---|---|---|---|
| Encoder | TENR | ENCODER | ©too | Software/hardware used to create the file |
| Encoded By | TENC | ENCODEDBY | — | Person/organization that encoded the file |
| Encoder Settings | TSSE | ENCODERSETTINGS | — | CLI parameters used (e.g., "LAME --preset insane") |
| Source | — | SOURCE | — | Origin of the source material |

### 3.2 "Encoder" Tag (TENR / ENCODER) — Three Competing Behaviors

The critical question: when converting FLAC→MP3, does DBpoweramp preserve the original encoder tag ("reference libFLAC 1.4.0")?

**Answer: It depends on configuration settings.**

DBpoweramp has a **per-conversion option** in dBpoweramp Configuration:

> "Update 'Source', 'Encoder', 'Encoded By' & 'Encoder Settings' ID tags in dBpoweramp Configuration, Music Converter, When Converting section"

**Source:** [DBpoweramp Forum — FLAC→AIFF/MP3 encoder fields](https://forum.dbpoweramp.com/forum/dbpoweramp/batch-converter/33132-flac-aiff-mp3-how-can-i-disable-the-writing-of-encoder-encoder-fields)

**Three behaviors (user-selectable):**

| Behavior | Setting State | Result |
|---|---|---|
| **Overwrite** | "Update encoder tags" enabled (default) | Original encoder replaced with new encoder info |
| **Preserve** | "Update encoder tags" disabled | Original encoder tag preserved as-is |
| **Chain** | Not natively supported | Must be done manually via ID Tag Processing DSP → Additions |

**Practical chain example via ID Tag Processing DSP:**

If the user wants to chain (original → new), they would:
1. Use ID Tag Processing DSP → Additions: add a custom field like `ORIGINAL_ENCODER = {source encoder}`
2. Let DBpoweramp write the new encoder as `TENR`
3. Result: both original and new encoder info preserved

**Default behavior:** DBpoweramp **overwrites** the Encoder tag with the new encoder's identity (e.g., "dBpoweramp Music Converter" or "LAME 3.100"). It does NOT chain by default.

### 3.3 "Encoder Settings" Tag (TSSE / ENCODERSETTINGS)

This tag stores the actual encoding parameters (bitrate, mode, etc.).

**DBpoweramp default:** Writes the current conversion parameters (e.g., `--preset standard` for LAME) to TSSE.

**To preserve original settings:** Disable "Update encoder tags" in Configuration. However, note that original ENCODERSETTINGS values may not be meaningful for the new encoder anyway (different codecs = different parameter semantics).

**Source:** [DBpoweramp Forum — encoder field disable](https://forum.dbpoweramp.com/forum/dbpoweramp/batch-converter/33132-flac-aiff-mp3-how-can-i-disable-the-writing-of-encoder-encoder-fields) — "To remove these tags on conversion use either: ID Tag Update utility codec >> Deletions tab OR Add DSP Effect ID Tag Processing >> Deletions tab."

### 3.4 "Encoded By" (TENC / ENCODEDBY)

Typically set to "dBpoweramp Music Converter" on rip/conversion. Controlled by the same "Update encoder tags" setting.

### 3.5 "Source" Field

Available in Vorbis comments (FLAC/OGG) but has **no direct ID3v2 equivalent**. When converting to MP3:
- If "Update encoder tags" is enabled: Source field is dropped
- If disabled: Source field is dropped (no mapping exists)
- Workaround: Use ID Tag Processing DSP → Map to store Source in a TXXX frame

---

## 4. What Is Never Dropped

DBpoweramp will **never** drop these fields during a conversion where the destination format supports an equivalent:

- **Title** (TIT2 / TITLE / ©nam)
- **Artist** (TPE1 / ARTIST / ©ART)
- **Album** (TALB / ALBUM / ©alb)
- **Track Number** (TRCK / TRACKNUMBER / trkn)
- **Disc Number** (TPOS / DISCNUMBER / disk)
- **Genre** (TCON / GENRE / gnre)
- **Year** (TYER / DATE / ©day)
- **Cover Art** (APIC / COVERART / covr) — unless explicitly removed
- **ReplayGain fields** — unless the "Drop ReplayGain" advanced option is checked

---

## 5. Loss-of-Precision Cases

### 5.1 ID3v1 30-Character Limit

**The Problem:** ID3v1 (v1 and v1.1) strictly limits title, artist, album, and comment fields to 30 bytes each. Any content beyond 30 bytes is silently truncated.

**DBpoweramp Behavior:** DBpoweramp writes **both** ID3v1 and ID3v2 tags by default. The ID3v1 tag will be truncated to 30 characters; the ID3v2 tag retains the full value.

**Source:** [DBpoweramp Forum — ID3v1 30-char truncation](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/30115-id-tags-in-mp3-files-truncated-at-30-characters) — "Tag from filename will not write a 2.4 tag if there is already a 1.1 tag, only this will be written."

**Edge Case — Character Encoding Multiplier:**
If a title contains multi-byte UTF-8 characters (e.g., Japanese: "約束"), each kanji is 3 UTF-8 bytes. The ID3v1 limit is **30 bytes**, not 30 characters. So "約束の土地" (15 characters) would be truncated mid-character because its UTF-8 representation exceeds 30 bytes.

**Mitigation:** Disable ID3v1 writing in DBpoweramp Configuration → Audio Codecs → MP3 → Advanced → ID3v1 Version: "None"

### 5.2 MP4 Track/Total Overflow Beyond 65535

**The Problem:** The MP4 `trkn` atom stores track number and total as two separate 16-bit unsigned integers (uint16). The maximum value is 65,535.

**Source:** [Android mp4v2 — src/mp4meta.cpp](https://android.googlesource.com/platform/external/mp4v2/+/master/src/mp4meta.cpp) — "16 bit: tracknumber / 16 bit: total tracks on album"

**Edge Case:** Multi-disc box sets with more than 65,535 total tracks (theoretical but possible in compilation scenarios). Values exceeding 65535 would wrap around (overflow) or be silently capped.

**DBpoweramp Behavior:** Unknown if DBpoweramp caps at 65535 or wraps. No explicit forum evidence found. **Recommendation:** Store high track counts in a TXXX frame (e.g., `TXXX:TRACKTOTAL=70000`) as a parallel reference.

### 5.3 Vorbis DATE → ID3v2.3 TYER Truncation

**The Problem:** ID3v2.3 has a dedicated year field (TYER) that can only hold 4 digits (YYYY). Vorbis DATE can hold full ISO 8601 timestamps (e.g., `2024-03-15T14:30:00Z`).

**DBpoweramp Behavior:** Extracts the year component (first 4 digits) and writes to TYER. The full timestamp is **not** stored in ID3v2.3.

**Source:** ID3v2.3 spec — TYER is defined as a 4-digit year.

**Edge Cases:**

| Vorbis DATE Value | ID3v2.3 TYER Output | Notes |
|---|---|---|
| `2024` | `2024` | Simple year |
| `2024-03-15` | `2024` | Date extracted correctly |
| `2024-03-15T14:30:00Z` | `2024` | Time component stripped |
| `circa 2024` | `circa` (invalid) | Non-numeric year may cause issues |
| `2024/2025` | `2024` | First year extracted |

**Upgrade Path:** ID3v2.4 replaced TYER, TDAT, and TIME with the flexible TDRC frame, which can hold full timestamps. Using ID3v2.4 (DBpoweramp default) avoids this truncation.

### 5.4 Bitrate/Sample Rate from Source vs. Destination

When converting from a lossless source to a lossy destination, the source's lossless bitrate is different from the output's lossy bitrate. DBpoweramp writes the **output (destination) technical metadata**, not the source's.

---

## 6. Source Technical Metadata Handling

### 6.1 Bitrate

| Conversion Direction | Written Tag |
|---|---|
| Lossless → Lossy | Output bitrate (newly encoded) |
| Lossy → Lossless | Output bitrate (from decoded/re-encoded) |
| Lossless → Lossless | Output bitrate (re-encoded) |
| Same format transcode | Output bitrate |

**Note:** Some workflows preserve original bitrate in a custom TXXX field (`TXXX:ORIGINAL_BITRATE`) via ID Tag Processing DSP.

### 6.2 Sample Rate / Bit Depth

Written to the audio codec's internal headers (not as metadata tags). Preserved through conversion if the codec supports it.

### 6.3 Encoder Info

Controlled by the "Update encoder tags" setting (see Section 3). Default: overwrite with new encoder info.

### 6.4 Technical Metadata as Custom Tags

To preserve source technical metadata as tags:

```bash
# Via ID Tag Processing DSP, add custom mappings:
# Source: BITRATE → TXXX:ORIGINAL_BITRATE
# Source: SAMPLE_RATE → TXXX:ORIGINAL_SAMPLERATE
# Source: BIT_DEPTH → TXXX:ORIGINAL_BITDEPTH
```

---

## 7. Field Preservation Checklist

### Always Preserved (Standard Fields)
- [ ] Title
- [ ] Artist
- [ ] Album
- [ ] Track Number / Total
- [ ] Disc Number / Total
- [ ] Genre
- [ ] Cover Art (binary, not converted to text)
- [ ] ReplayGain (if option enabled)

### Preserved with Normalization
- [ ] Year (→ 4-digit TYER for ID3v2.3)
- [ ] Date (→ year component for ID3v2.3)
- [ ] BPM (decimal → integer)
- [ ] Encoder (if option disabled; overwritten if enabled)
- [ ] MusicBrainz IDs (via TXXX or ----:*)

### Dropped by Default
- [ ] Vorbis vendor string
- [ ] SOURCE field (no ID3v2 equivalent)
- [ ] ASF/WMA-specific fields (→ FLAC)
- [ ] FLAC METADATA_BLOCK_PICTURE internal metadata

### User-Configurable (DSP/Options)
- [ ] Encoder Settings (TSSE/ENCODERSETTINGS)
- [ ] Encoded By (TENC/ENCODEDBY)
- [ ] ReplayGain (Advanced option to drop)
- [ ] Custom/unknown fields (ID Tag Processing rules)

---

## 8. Edge Cases

### Edge Case 1: FLAC with no Vorbis Comments
Converting a raw PCM WAV with no metadata to FLAC: DBpoweramp creates an empty Vorbis comment block with only a vendor string. Converting back to WAV: the vendor string is lost (no RIFF INFO equivalent).

### Edge Case 2: Multi-Disc Album with Track > 65535
Theoretical: A compilation with 70,000 tracks stored in MP4 would overflow the `trkn` uint16. DBpoweramp would likely cap or wrap. Mitigation: use `TXXX:TRACKTOTAL` as a parallel counter.

### Edge Case 3: Encoded-By Chain in Multi-Generation Transcode
If A (original) → B (1st transcode with "preserve encoder") → C (2nd transcode), the encoder chain grows: A's encoder + B's encoder + C's encoder. Each transcode may append rather than overwrite if the user has configured chaining manually.

### Edge Case 4: ID3v1 Truncation with Non-ASCII Characters
A title like "Mötley Crüe" in UTF-8 is 13 bytes. In Latin-1 (ISO-8859-1) it's 11 bytes. The ID3v1 limit of 30 bytes may truncate differently depending on how the bytes are counted. DBpoweramp's ID3v1 output uses the OS/system locale for encoding assumptions.

### Edge Case 5: ReplayGain Tag Version Mismatch
ReplayGain has two versions: v1 (REPLAYGAIN_TRACK_GAIN) and v2 (REPLAYGAIN2_TRACK_GAIN). Some taggers write both. DBpoweramp reads both but may only write one depending on format. Converting between versions can lose the alternate.

---

## 9. Code Examples and References

### 9.1 Preserving All Tags with ID Tag Processing DSP (FLAC → MP3)

```
DSP Effect Chain Order:
1. ID Tag Processing
   - Map: VORBIS_FIELD → ID3V2_FIELD (custom mappings for non-standard tags)
   - Deletions: (none)
   - Additions: Add ENCODEDBY = "dBpoweramp Music Converter"
2. [Encoder codec]

Example custom mapping for ReplayGain:
Map: REPLAYGAIN_TRACK_GAIN → TXXX:REPLAYGAIN_TRACK_GAIN
Map: REPLAYGAIN_ALBUM_GAIN → TXXX:REPLAYGAIN_ALBUM_GAIN
```

**Source:** [DBpoweramp DSP Help](https://dbpoweramp.com/Help/dMC/dsp)

### 9.2 TagLib Encoder String Handling (Reference Implementation)

```cpp
// TagLib default: ISO-8859-1 for ID3v1
// Override via ID3v1::Tag::setStringHandler() for custom encoding
class CustomStringHandler : public TagLib::ID3v1::StringHandler {
    TagLib::String parse(const TagLib::ByteVector &data) const override {
        // Attempt encoding detection, fallback to ISO-8859-1
        return TagLib::String(data, TagLib::String::Latin1);
    }
};
```

**Source:** [TagLib ID3v1::StringHandler](https://taglib.org/api/classTagLib_1_1ID3v1_1_1StringHandler.html)

### 9.3 FFmpeg ID3v2 Text Encoding Decision (Reference)

```c
// FFmpeg id3v2enc.c — determines encoding per frame
// Uses UTF-16 BOM for non-ASCII, ISO-8859-1 for ASCII-only
if (enc == ID3v2_ENCODING_UTF16BOM && string_is_ascii(str1))
    enc = ID3v2_ENCODING_ISO8859;
```

**Source:** [FFmpeg libavformat/id3v2enc.c](https://ffmpeg.org/doxygen/trunk/id3v2enc_8c_source.html)

---

## Summary: Would a User Notice Any Difference?

| Scenario | User Impact |
|---|---|
| FLAC→MP3 with "preserve encoder" disabled | Original encoder tag overwritten; user may lose "reference libFLAC 1.4.0" |
| Long titles in ID3v1 | ID3v1 shows 30-char truncation; ID3v2 shows full title |
| Multi-disc box set in MP4 | Track counts >65535 may overflow or cap |
| DATE with timestamp → ID3v2.3 | Time component lost (only year kept) |
| ReplayGain preservation | Requires unchecking "Drop ReplayGain" in Advanced settings |
| Custom tags (MUSICBRAINZ_*, REPLAYGAIN_*) | Require ID Tag Processing DSP mapping; not automatic |

**Bottom Line:** Standard fields (title, artist, album, track) are well-preserved. The main sources of surprise are: (1) encoder tag overwrite default, (2) ID3v1 truncation, (3) DATE→TYER loss, and (4) custom/unknown tags requiring manual DSP configuration.
