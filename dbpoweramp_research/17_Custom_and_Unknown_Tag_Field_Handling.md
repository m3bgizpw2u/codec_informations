# 17_Custom_and_Unknown_Tag_Field_Handling.md

> **Research Area:** DBpoweramp Music Converter — Custom and Unknown Tag Field Handling
> **Covers:** Unknown field preservation, TXXX frames, APEv2 custom items, Vorbis custom fields, user options, name normalization
> **Confidence Level:** Medium (primarily inferred from TagLib/Mutagen behavior + DBpoweramp DSP documentation)
> **Cross-references:** File 15 (Preservation Rules), File 16 (Conflict Resolution)

---

## 1. Overview

Custom and unknown tag fields are metadata entries whose names are not part of any standardized tag mapping. Examples include MusicBrainz-specific IDs (MUSICBRAINZ_ALBUMID), ReplayGain values (REPLAYGAIN_TRACK_GAIN), and user-defined fields like "Mood" or "Catalog #". This document covers how DBpoweramp handles these during conversion.

---

## 2. Preservation of Unknown/Custom Tags

### 2.1 The Core Problem: Schema-Based vs. Free-Form Tagging

| Format Family | Schema Type | Custom Field Mechanism | DBpoweramp Support |
|---|---|---|---|
| ID3v2 (MP3) | Schema-based + extension | TXXX frames (user-defined text) | Full support |
| APEv2 (MP3/APE/MPC) | Free-form | Any name-value pair | Full support |
| Vorbis (FLAC/OGG) | Free-form | Any key-value pair | Full support |
| MP4 | Semi-structured | ----:* namespaced atoms | Partial — some unsupported |
| RIFF INFO (WAV) | Fixed set | No custom fields | None |

**Key Insight:** Formats with free-form tagging (Vorbis, APEv2) can preserve arbitrary field names. Formats with schema-based tagging (ID3v2) require custom fields to be stored in designated extension frames (TXXX).

**Source:** [DBpoweramp Forum — FLAC→MP3 tag preservation](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/41211-converting-flac-to-mp3-lame-preserve-all-id-tags) — "There is no simple copy all tags because mp3 uses ID3v2 tags which are based on predetermined tag names, FLAC is based on a undefined tagging format."

### 2.2 DBpoweramp's Default Custom Tag Behavior

**Without any DSP configuration:**
- **Standard fields** (Title, Artist, Album, etc.): Mapped automatically
- **Recognized non-standard fields**: May or may not be preserved depending on codec support
- **Completely unknown fields**: **Likely dropped** during cross-format conversion

**The "ID Tag Processing" DSP is the primary mechanism for custom tag preservation.** Without it configured, DBpoweramp does not guarantee unknown field preservation.

### 2.3 Fields That Are Preserved Without DSP Configuration

Based on forum evidence and codec documentation, these fields survive FLAC→MP3 conversion without explicit DSP rules:

| Field | FLAC/Vorbis Name | MP3/ID3v2 Name |
|---|---|---|
| ReplayGain Track Gain | REPLAYGAIN_TRACK_GAIN | TXXX:REPLAYGAIN_TRACK_GAIN |
| ReplayGain Album Gain | REPLAYGAIN_ALBUM_GAIN | TXXX:REPLAYGAIN_ALBUM_GAIN |
| ReplayGain Track Peak | REPLAYGAIN_TRACK_PEAK | TXXX:REPLAYGAIN_TRACK_PEAK |
| ReplayGain Album Peak | REPLAYGAIN_ALBUM_PEAK | TXXX:REPLAYGAIN_ALBUM_PEAK |
| AcoustID Fingerprint | ACOUSTID_FINGERPRINT | TXXX:Acoustid Fingerprint |
| AcoustID ID | ACOUSTID_ID | TXXX:Acoustid Id |

**Source:** [DBpoweramp Forum — ReplayGain preservation](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/41211-converting-flac-to-mp3-lame-preserve-all-id-tags) — "About replaygain tags, run dBpoweramp Control Centre >> Advanced. There is an option to drop replaygain tags, uncheck it."

### 2.4 Fields That Are Likely Dropped

These fields have no standard mapping and no automatic preservation:

- SOURCE (Vorbis) — no ID3v2 equivalent
- ORIGINALDATE, ORIGINALYEAR (Vorbis) — no standard mapping
- LABEL, LABELID (Vorbis) — no standard mapping
- BARCODE, CATALOGNUMBER — mapped by some taggers but not automatic in DBpoweramp
- Custom user fields (e.g., "MY_CUSTOM_FIELD")

**Mitigation:** Use ID Tag Processing DSP → Map to explicitly map these fields to TXXX frames before conversion.

---

## 3. TXXX Frames in ID3v2 — Unknown TXXX Handling

### 3.1 TXXX Frame Structure

```
TXXX frame body:
  Text encoding     (1 byte: $00-ISO-8859-1, $01-UTF-16, $02-UTF-16BE, $03-UTF-8)
  Description       (variable: encoding-dependent string, null-terminated)
  Value             (variable: the actual custom field value)
```

### 3.2 How DBpoweramp Reads TXXX Frames

DBpoweramp uses **TagLib** for ID3v2 tag I/O. TagLib's TXXX handling:

- TXXX frames are read with description as the key
- The value is stored as a string
- **Unknown TXXX descriptions are preserved as-is** (TagLib does not filter by known descriptions)
- Multiple TXXX frames with the same description: **last value wins** (TagLib reads the last occurrence)

**Source:** [ID3v2.4 spec — TXXX](https://id3.org/id3v2.4.0-structure) — "There may be more than one 'TXXX' frame in each tag, but only one with the same description."

### 3.3 How DBpoweramp Writes TXXX Frames

When DBpoweramp writes a TXXX frame (e.g., REPLAYGAIN_TRACK_GAIN):
- Encoding: **UTF-8** (ID3v2.4) or **UTF-16 BOM** (ID3v2.3) — configurable in dBpoweramp Configuration
- Description: exact field name (case-sensitive)
- Value: field value

### 3.4 Multi-Value TXXX Frames

**Critical Issue:** ID3v2.3 spec says only one TXXX per description. However, some taggers (foobar2000) write multiple TXXX frames with the same description as separate frames. This is non-compliant.

**DBpoweramp's behavior when reading:**
- TagLib sees multiple TXXX frames with same description
- TagLib returns the **last** value (overwrites earlier ones)

**DBpoweramp's behavior when writing:**
- Writes a single TXXX frame per description
- Multiple values for the same field are joined with "; " (semicolon-space)

**Source:** [Hydrogenaudio — Multi valued TXXX frames in ID3v2.3](https://hydrogenaud.io/index.php/topic,112341.0.html)

---

## 4. APEv2 Custom Item Preservation

### 4.1 APEv2 Item Structure

```
APEv2 item:
  Item flags      (4 bytes: contains type info: text, binary, external)
  Item key        (null-terminated string, ASCII)
  Item value      (length from header; no null terminator)
```

### 4.2 APEv2 Supports Arbitrary Keys

APEv2's free-form nature means **any ASCII key name is valid**. There is no predefined set. This is the most permissive custom-tag format.

### 4.3 DBpoweramp APEv2 Handling

**Reading:**
- DBpoweramp reads APEv2 items for ReplayGain and rating fields
- Unknown APEv2 items: **likely preserved** (APE items are stored as key-value pairs; TagLib reads them generically)
- APEv2 text items are decoded as UTF-8 if valid, otherwise as ISO-8859-1

**Writing:**
- When DBpoweramp writes APEv2 tags (MP3 files with APE enabled), it writes standard fields
- Custom items from source APEv2 are **not automatically preserved** unless specifically mapped
- To preserve custom APEv2 items: use ID Tag Processing DSP → Map

**Source:** [TagLib documentation](https://taglib.org/api/) — APEv2 items support arbitrary keys

### 4.4 APEv2 Binary Items (e.g., cover art in APE)

APEv2 can store binary items (cover art via `Cover Art (Front)` key). DBpoweramp:
- Reads embedded APEv2 cover art
- Converts to ID3v2 APIC when writing to MP3
- May not preserve non-image binary items

---

## 5. Vorbis Comment Custom Field Preservation

### 5.1 Vorbis Comment Key Rules

From the Vorbis spec:

- Keys are **UTF-8** encoded
- Keys are **case-insensitive** for comparison (but case is preserved)
- Keys can be any ASCII string (recommended: alphanumeric + underscore)
- The format reserves certain lowercase keys for standard fields; uppercase keys are explicitly user-defined

**Source:** [Xiph Vorbis Comment spec](https://xiph.org/vorbis/doc/v-comment.html)

### 5.2 Case Sensitivity in Vorbis Comments

**Comparison:** Vorbis keys are compared **case-insensitively**. `ARTIST`, `Artist`, and `artist` refer to the same field.

**Storage:** The original case is preserved in storage. Reading back, all three would be accessible as "artist".

**DBpoweramp behavior:** When converting Vorbis→ID3v2:
- Vorbis keys are lowercased internally for mapping
- Mapped fields are written to ID3v2 in standard case (uppercase for frames)
- Unmapped fields: preserved in TXXX with original-case key as description

### 5.3 Multi-Value Vorbis Fields

Vorbis comments support **multiple values per key** naturally (each instance of the key is a separate value). There is no separator convention — multiple instances are semantically equivalent to multiple values.

**DBpoweramp conversion to ID3v2:**
- Multiple values for standard fields (e.g., multiple ARTIST entries): joined with "; "
- Multiple values for custom fields: joined with "; " or written as multiple TXXX frames (one per value — non-compliant with ID3v2.3 spec)

---

## 6. User Option: Preserve as Custom vs. Drop

### 6.1 DBpoweramp Configuration Options

| Setting | Location | Effect |
|---|---|---|
| Drop ReplayGain | Control Centre → Advanced | When checked: REPLAYGAIN_* tags are dropped |
| Update encoder tags | Configuration → Music Converter | When checked: encoder tags are overwritten |
| ID3v2 text encoding | Configuration → Codecs → Advanced | UTF-8 vs UTF-16 for custom fields |
| Preserve Source Attributes | DSP effects | Copies file attributes, not tag fields |

**No generic "preserve all unknown tags" option exists.** Users must configure ID Tag Processing DSP rules individually.

### 6.2 ID Tag Processing DSP — Custom Tag Mapping

The ID Tag Processing DSP supports:
- **Map:** Copy a field to a new name
- **Deletions:** Remove specific fields
- **Additions:** Add new fields
- **Manipulation:** Apply transformations (case, multiple artist split/join)

**Example: Preserving a custom FLAC field in MP3 output**

```
DSP: ID Tag Processing
  Map: MY_CUSTOM_FIELD → TXXX:MY_CUSTOM_FIELD
```

**Example: Preserving multiple MusicBrainz IDs**

```
Map: MUSICBRAINZ_ALBUMID → TXXX:MusicBrainz Album Id
Map: MUSICBRAINZ_RELEASEGROUPID → TXXX:MusicBrainz Release Group Id
Map: MUSICBRAINZ_RECORDINGID → TXXX:MusicBrainz Recording Id
Map: MUSICBRAINZ_TRACKID → TXXX:MusicBrainz Track Id
```

**Source:** [DBpoweramp Forum — MusicBrainz tag mapping](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/41211-converting-flac-to-mp3-lame-preserve-all-id-tags) — "I found the following information... This helped me to define a tag mapping for the 'ID tag processing' DSP effect."

### 6.3 Automatic MusicBrainz ID Mapping

DBpoweramp has partial built-in support for MusicBrainz IDs. Some versions may automatically map:
- MUSICBRAINZ_ALBUMID → TXXX:MusicBrainz Album Id
- MUSICBRAINZ_TRACKID → TXXX:MusicBrainz Track Id

However, forum evidence suggests this is **not universal across all DBpoweramp versions**. Explicit DSP mapping is the reliable approach.

---

## 7. Custom Field Name Normalization

### 7.1 Case Sensitivity Across Formats

| Format | Key/Field Name Case Sensitivity |
|---|---|---|
| ID3v2 TXXX | Case-insensitive in comparison; stored as-is |
| APEv2 | Case-sensitive for storage; case-insensitive for comparison |
| Vorbis Comment | Case-insensitive for comparison; stored as-is |
| MP4 ----:* | Case-sensitive (namespace is part of the key) |
| RIFF INFO | Fixed set (no custom fields) |

### 7.2 DBpoweramp Normalization Behavior

**Vorbis → ID3v2:**
- Standard field names: converted to uppercase frame IDs (ARTIST → TPE1)
- Custom field names: written as TXXX description with original case preserved
- Example: Vorbis `foo_bar` → TXXX `foo_bar` (case preserved)

**ID3v2 → Vorbis:**
- Frame IDs: converted to lowercase Vorbis keys (TPE1 → artist)
- TXXX description: used as Vorbis key (case preserved)

### 7.3 Cross-Tagger Case Compatibility

| Source Tool | Field Name | Stored As | Other Tool Reads As |
|---|---|---|---|
| Mp3tag | CATALOGNUMBER | TXXX:CATALOGNUMBER | TXXX:CATALOGNUMBER |
| Picard | CATALOGNUMBER | TXXX:CATALOGNUMBER | TXXX:CATALOGNUMBER |
| DBpoweramp | CATALOGNUMBER | TXXX:CATALOGNUMBER | TXXX:CATALOGNUMBER |
| Hand-written | catalogNumber | TXXX:catalogNumber | TXXX:catalogNumber |

**Note:** Picard stores MusicBrainz IDs with `MusicBrainz` prefix (proper capitalization). DBpoweramp maps may need to match exactly.

---

## 8. Multi-Value Custom Fields

### 8.1 Format-Specific Multi-Value Behavior

| Format | Multiple Values for Custom Field |
|---|---|
| Vorbis | Natural: multiple comment lines with same key |
| APEv2 | Natural: multiple items with same key |
| ID3v2.4 | Multiple TXXX frames with same description (compliant) |
| ID3v2.3 | Single TXXX per description; multiple values must be joined with separator |
| MP4 | Multiple ----:* items with same key (within namespace) |

### 8.2 DBpoweramp Multi-Value Handling

**Vorbis → ID3v2.3:**
- Multiple values for custom field → single TXXX with values joined by "; "
- Example: FLAC with `CATALOGNUMBER=ABC123` and `CATALOGNUMBER=XYZ789` → MP3 with `TXXX:CATALOGNUMBER=ABC123; XYZ789`

**ID3v2.4 → Vorbis:**
- Multiple TXXX frames with same description → multiple Vorbis comment lines
- Example: MP3 with two `TXXX:CATALOGNUMBER` frames → FLAC with two `CATALOGNUMBER` entries

**MP4 → ID3v2:**
- Multiple `----:com.apple.iTunes:CATALOGNUMBER` items → single `TXXX:CATALOGNUMBER` with "; " join

### 8.3 Separator Convention

DBpoweramp uses **"; "** (semicolon-space) as the multi-value separator for ID3v2 single-value fields. This is standard practice but not universally adopted — other tools may use null bytes, slashes, or commas.

---

## 9. Edge Cases

### Edge Case 1: TXXX Frame with Empty Description

Some taggers write TXXX frames with empty descriptions. This is technically valid (description is a zero-length string). DBpoweramp's handling:
- TagLib: reads but may not expose empty-description TXXX in the standard field interface
- **Result:** Empty-description TXXX frames may be silently dropped

### Edge Case 2: Binary Data in TXXX Frame

TXXX frames are defined as text frames (text encoding byte applies). Writing binary data (e.g., raw MD5 hash as bytes) to TXXX:
- The bytes are interpreted as text (encoding specified in the frame header)
- Non-text bytes may produce garbled output
- **Better approach:** Use a dedicated binary-supporting format (APIC for images; a separate custom atom for binary data)

### Edge Case 3: Unicode Characters in Vorbis Custom Field Keys

Vorbis spec says keys should be ASCII. However, some tools write UTF-8 keys. DBpoweramp:
- Likely reads them correctly (TagLib handles UTF-8)
- When converting to ID3v2 TXXX: description may be written as-is (UTF-8 compatible)
- **Risk:** Some ID3v2 parsers expect TXXX descriptions to be ASCII or valid in the specified encoding

### Edge Case 4: Picard-Specific Custom Fields

Picard writes many non-standard fields:
- `MUSICBRAINZ_RELEASEATTRIBUTE` (disc selection)
- `SCRIPT` (character encoding)
- `BARCODE` (may map or not)
- `RELEASESTATUS`, `RELEASETYPE`

These are stored as Vorbis comments in FLAC. DBpoweramp's conversion:
- `MUSICBRAINZ_*` fields: many have no automatic mapping → likely dropped
- `BARCODE`: no standard mapping → likely dropped
- Workaround: ID Tag Processing DSP → Map

### Edge Case 5: ReplayGain Version Ambiguity

ReplayGain has two versions:
- **v1:** REPLAYGAIN_TRACK_GAIN, REPLAYGAIN_ALBUM_GAIN, REPLAYGAIN_TRACK_PEAK, REPLAYGAIN_ALBUM_PEAK
- **v2:** REPLAYGAIN2_TRACK_GAIN, REPLAYGAIN2_ALBUM_GAIN, REPLAYGAIN2_TRACK_PEAK, REPLAYGAIN2_ALBUM_PEAK

DBpoweramp:
- Reads both versions
- Writes v1 by default
- During conversion: v2 fields are likely **dropped** unless explicitly mapped

---

## 10. Code Examples

### 10.1 TagLib Reading Custom TXXX Frames

```cpp
// TagLib: reading all TXXX frames
ID3v2::Tag *tag = /* ... */;
const ID3v2::FrameList &frames = tag->frameList();
for (auto it = frames.begin(); it != frames.end(); ++it) {
    if ((*it)->frameID() == "TXXX") {
        ID3v2::UserTextIdentificationFrame *txxx =
            dynamic_cast<ID3v2::UserTextIdentificationFrame*>(*it);
        String desc = txxx->description();  // The key
        String val = txxx->fieldList().front()->text();  // The value
        // Both desc and val are preserved as-is
    }
}
```

### 10.2 Mp3tag TXXX Mapping Table (Reference)

From Mp3tag's official mapping table, key custom fields and their storage:

| Field | ID3v2.4 | MP4 | Vorbis |
|---|---|---|---|
| AcoustID Fingerprint | TXXX:Acoustid Fingerprint | ----:com.apple.iTunes:Acoustid Fingerprint | ACOUSTID_FINGERPRINT |
| MusicBrainz Album Id | TXXX:MusicBrainz Album Id | ----:com.apple.iTunes:MusicBrainz Album Id | MUSICBRAINZ_ALBUMID |
| MusicBrainz Track Id | TXXX:MusicBrainz Track Id | ----:com.apple.iTunes:MusicBrainz Track Id | MUSICBRAINZ_TRACKID |
| ReplayGain Track Gain | TXXX:REPLAYGAIN_TRACK_GAIN | ----:com.apple.iTunes:REPLAYGAIN_TRACK_GAIN | REPLAYGAIN_TRACK_GAIN |

**Source:** [Mp3tag Tag Field Mappings Table](https://docs.mp3tag.de/mapping-table/)

### 10.3 DBpoweramp ID Tag Processing — Mapping Unknown Fields

```
Configuration for preserving unknown/custom FLAC fields in MP3:

DSP: ID Tag Processing
  Map: ORIGINALDATE → TXXX:Original Date
  Map: LABEL → TXXX:Label
  Map: CATALOGNUMBER → TXXX:Catalog Number
  Map: BARCODE → TXXX:Barcode
  Map: SCRIPT → TXXX:Script
  Map: RELEASESTATUS → TXXX:Release Status
  Map: RELEASETYPE → TXXX:Release Type
```

---

## Summary: Would a User Notice Any Difference?

| Custom Field Scenario | User Impact |
|---|---|
| FLAC custom field with no mapping | Dropped during FLAC→MP3 without DSP |
| MusicBrainz IDs | Some preserved, some dropped; DSP mapping needed |
| ReplayGain | Preserved by default; lost if "Drop ReplayGain" checked |
| Case differences in field names | TagLib normalizes; Picard may not recognize all names |
| Multi-value custom fields | Joined with "; " in ID3v2.3; multiple frames in ID3v2.4 |
| TXXX with empty description | Likely silently dropped |
| Picard-specific fields (SCRIPT, etc.) | Dropped unless explicitly mapped |

**Bottom Line:** DBpoweramp preserves standard and semi-standard fields (ReplayGain, AcoustID, basic MusicBrainz IDs) without configuration. Everything else requires explicit ID Tag Processing DSP rules. There is no "preserve all custom fields" switch.
