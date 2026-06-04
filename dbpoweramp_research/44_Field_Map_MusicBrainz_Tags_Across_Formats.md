# Field Mapping: MusicBrainz Tags Across Formats

## 1. Overview: Why MusicBrainz Tags Are Complex

### 1.1 The Problem

MusicBrainz identifiers (MBIDs) must be stored differently in each format because:
- Each format has different tag name restrictions
- ID3 uses 4-character frame IDs
- Vorbis uses freeform text fields
- MP4 uses atoms with namespace restrictions

**Source:** [MusicBrainz Forum](https://community.metabrainz.org/t/converting-transcoding-and-keeping-mapping/686122)

### 1.2 Format-Specific Tag Names

| Internal Name | ID3v2 | Vorbis | MP4 | WMA |
|---------------|-------|--------|-----|-----|
| Track ID | `UFID:http://musicbrainz.org` | `MUSICBRAINZ_TRACKID` | `----:com.apple.iTunes:MusicBrainz Track Id` | `MusicBrainz/Id` |
| Album ID | `TXXX:MusicBrainz Album Id` | `MUSICBRAINZ_ALBUMID` | `----:com.apple.iTunes:MusicBrainz Album Id` | `MusicBrainz/Album Id` |
| Artist ID | `TXXX:MusicBrainz Artist Id` | `MUSICBRAINZ_ARTISTID` | `----:com.apple.iTunes:MusicBrainz Artist Id` | `MusicBrainz/Artist Id` |
| Release Group ID | `TXXX:MusicBrainz Release Group Id` | `MUSICBRAINZ_RELEASEGROUPID` | `----:com.apple.iTunes:MusicBrainz Release Group Id` | `MusicBrainz/Release Group Id` |
| Disc ID | `TXXX:MusicBrainz Disc Id` | `MUSICBRAINZ_DISCID` | `----:com.apple.iTunes:MusicBrainz Disc Id` | `MusicBrainz/Disc Id` |
| Release Track ID | `TXXX:MusicBrainz Release Track Id` | `MUSICBRAINZ_RELEASETRACKID` | `----:com.apple.iTunes:MusicBrainz Release Track Id` | `MusicBrainz/Release Track Id` |
| Album Artist ID | `TXXX:MusicBrainz Album Artist Id` | `MUSICBRAINZ_ALBUMARTISTID` | `----:com.apple.iTunes:MusicBrainz Album Artist Id` | `MusicBrainz/Album Artist Id` |
| TRM ID | `TXXX:MusicBrainz TRM Id` | `MUSICBRAINZ_TRMID` | (obsolete) | (obsolete) |

**Source:** [Picard Tag Mapping](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html)

---

## 2. Track ID (Recording ID)

### 2.1 ID3v2: UFID Frame

**Format:** `UFID:http://musicbrainz.org` frame
- Owner identifier: `http://musicbrainz.org`
- Identifier: 16 bytes binary UUID (or 36-char string representation)

**Example:**
```
UFID:http://musicbrainz.org = "12345678-1234-1234-1234-123456789abc"
```

**Source:** [MusicBrainz spec](https://web.archive.org/web/20110120221258/musicbrainz.org/docs/specs/metadata_tags.html)

### 2.2 Vorbis/FLAC

**Format:** `MUSICBRAINZ_TRACKID` text field
- Value: UUID string
- Multiple values possible (multiple recordings)

**Example:**
```
MUSICBRAINZ_TRACKID=12345678-1234-1234-1234-123456789abc
```

### 2.3 MP4

**Format:** `----:com.apple.iTunes:MusicBrainz Track Id` freeform
- Namespace: `com.apple.iTunes`
- Value: UUID string

**Example:**
```
----:com.apple.iTunes:MusicBrainz Track Id=12345678-1234-1234-1234-123456789abc
```

### 2.4 Conversion: Track ID

| Source Format | Destination Format | Required Conversion |
|---------------|-------------------|---------------------|
| ID3 UFID | Vorbis | Extract UUID → `MUSICBRAINZ_TRACKID` |
| Vorbis | ID3 UFID | `MUSICBRAINZ_TRACKID` → UFID with owner `http://musicbrainz.org` |
| Vorbis | MP4 | `MUSICBRAINZ_TRACKID` → freeform with namespace |
| MP4 | Vorbis | Strip namespace prefix |

---

## 3. Album ID, Artist ID, Release Group ID

### 3.1 ID3v2: TXXX Frame

**Format:** `TXXX:MusicBrainz Album Id`
- TXXX frame with description: `MusicBrainz Album Id`
- Value: UUID string

**Example:**
```
TXXX:MusicBrainz Album Id=12345678-1234-1234-1234-123456789abc
```

### 3.2 Vorbis/FLAC

**Format:** `MUSICBRAINZ_ALBUMID`, `MUSICBRAINZ_ARTISTID`, `MUSICBRAINZ_RELEASEGROUPID`

**Example:**
```
MUSICBRAINZ_ALBUMID=12345678-1234-1234-1234-123456789abc
MUSICBRAINZ_ARTISTID=12345678-1234-1234-1234-123456789abc
MUSICBRAINZ_RELEASEGROUPID=12345678-1234-1234-1234-123456789abc
```

### 3.3 MP4

**Format:** `----:com.apple.iTunes:MusicBrainz <Type> Id`

**Example:**
```
----:com.apple.iTunes:MusicBrainz Album Id=12345678-1234-1234-1234-123456789abc
----:com.apple.iTunes:MusicBrainz Artist Id=12345678-1234-1234-1234-123456789abc
```

### 3.4 WMA/ASF

**Format:** `MusicBrainz/Album Id`, `MusicBrainz/Artist Id`, etc.

**Example:**
```
MusicBrainz/Album Id=12345678-1234-1234-1234-123456789abc
```

---

## 4. Disc ID and Release Track ID

### 4.1 Disc ID

**Purpose:** Identifies the specific disc in a release (accounts for pressed SID codes)

**Vorbis format:**
```
MUSICBRAINZ_DISCID=tDiscid_value_here
```

**ID3v2 format:**
```
TXXX:MusicBrainz Disc Id=tDiscid_value_here
```

### 4.2 Release Track ID

**Purpose:** Identifies the specific track as it appears on a specific release (accounts for different trackcount, disc number)

**Vorbis format:**
```
MUSICBRAINZ_RELEASETRACKID=12345678-1234-1234-1234-123456789abc
```

**Source:** [Picard Tag Mapping](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html)

---

## 5. Cross-Format Conversion

### 5.1 Python Implementation

```python
import re
import uuid

class MusicBrainzTagConverter:
    """Convert MusicBrainz tags between formats."""
    
    # Mapping table: internal_name -> format_specific_names
    MBID_MAPPING = {
        'musicbrainz_trackid': {
            'id3_ufid_owner': 'http://musicbrainz.org',
            'id3': None,  # Uses UFID frame
            'vorbis': 'MUSICBRAINZ_TRACKID',
            'mp4': '----:com.apple.iTunes:MusicBrainz Track Id',
            'wma': 'MusicBrainz/Id',
        },
        'musicbrainz_albumid': {
            'id3': 'TXXX:MusicBrainz Album Id',
            'vorbis': 'MUSICBRAINZ_ALBUMID',
            'mp4': '----:com.apple.iTunes:MusicBrainz Album Id',
            'wma': 'MusicBrainz/Album Id',
        },
        'musicbrainz_artistid': {
            'id3': 'TXXX:MusicBrainz Artist Id',
            'vorbis': 'MUSICBRAINZ_ARTISTID',
            'mp4': '----:com.apple.iTunes:MusicBrainz Artist Id',
            'wma': 'MusicBrainz/Artist Id',
        },
        'musicbrainz_releasetrackid': {
            'id3': 'TXXX:MusicBrainz Release Track Id',
            'vorbis': 'MUSICBRAINZ_RELEASETRACKID',
            'mp4': '----:com.apple.iTunes:MusicBrainz Release Track Id',
            'wma': 'MusicBrainz/Release Track Id',
        },
    }
    
    @staticmethod
    def validate_mbid(value: str) -> bool:
        """Validate UUID format."""
        try:
            uuid.UUID(value)
            return True
        except (ValueError, AttributeError):
            return False
    
    @staticmethod
    def normalize_mbid(value: str) -> str:
        """Normalize MBID to lowercase with hyphens."""
        try:
            return str(uuid.UUID(value)).lower()
        except (ValueError, AttributeError):
            return value.lower()
    
    def read_from_id3(self, tag_data: dict) -> dict:
        """Read MusicBrainz tags from ID3 tag dictionary."""
        result = {}
        
        # UFID frame
        if 'UFID' in tag_data:
            for frame in tag_data['UFID']:
                if hasattr(frame, 'owner') and frame.owner == 'http://musicbrainz.org':
                    # UFID contains binary UUID
                    if hasattr(frame, 'data'):
                        mbid = frame.data.decode('ascii', errors='replace').rstrip('\x00')
                    else:
                        mbid = str(frame).rstrip('\x00')
                    result['musicbrainz_trackid'] = self.normalize_mbid(mbid)
        
        # TXXX frames
        for key, mbid_key in [
            ('MusicBrainz Album Id', 'musicbrainz_albumid'),
            ('MusicBrainz Artist Id', 'musicbrainz_artistid'),
            ('MusicBrainz Release Group Id', 'musicbrainz_releasegroupid'),
            ('MusicBrainz Release Track Id', 'musicbrainz_releasetrackid'),
        ]:
            if key in tag_data:
                value = tag_data[key]
                if isinstance(value, list):
                    value = value[0]
                if self.validate_mbid(value):
                    result[mbid_key] = self.normalize_mbid(value)
        
        return result
    
    def write_to_vorbis(self, mbid_data: dict) -> dict:
        """Write MusicBrainz tags to Vorbis comment dictionary."""
        tags = {}
        
        for internal_key, tag_key in [
            ('musicbrainz_trackid', 'MUSICBRAINZ_TRACKID'),
            ('musicbrainz_albumid', 'MUSICBRAINZ_ALBUMID'),
            ('musicbrainz_artistid', 'MUSICBRAINZ_ARTISTID'),
            ('musicbrainz_releasetrackid', 'MUSICBRAINZ_RELEASETRACKID'),
        ]:
            if internal_key in mbid_data:
                tags[tag_key] = mbid_data[internal_key]
        
        return tags
    
    def write_to_mp4(self, mbid_data: dict) -> dict:
        """Write MusicBrainz tags to MP4 freeform dictionary."""
        tags = {}
        
        for internal_key, tag_key in [
            ('musicbrainz_trackid', '----:com.apple.iTunes:MusicBrainz Track Id'),
            ('musicbrainz_albumid', '----:com.apple.iTunes:MusicBrainz Album Id'),
            ('musicbrainz_artistid', '----:com.apple.iTunes:MusicBrainz Artist Id'),
            ('musicbrainz_releasetrackid', '----:com.apple.iTunes:MusicBrainz Release Track Id'),
        ]:
            if internal_key in mbid_data:
                tags[tag_key] = mbid_data[internal_key]
        
        return tags
```

---

## 6. Picard's Tag Normalization Approach

### 6.1 Internal Tag Names

Picard uses internal names that are independent of format:
- `musicbrainz_recordingid` (track ID)
- `musicbrainz_albumid`
- `musicbrainz_artistid`
- `musicbrainz_releasegroupid`
- `musicbrainz_discid`
- `musicbrainz_releasetrackid`
- `musicbrainz_albumartistid`

### 6.2 Picard's Script for Tag Standardization

**Problem:** When converting formats, tools may write format-specific names that Picard doesn't recognize.

**Solution:** Use Picard's scripting to normalize:

```
$if($not(%musicbrainz_albumid%),$set(musicbrainz_albumid,%MUSICBRAINZ_ALBUMID%)$delete(MUSICBRAINZ_ALBUMID))
$if($not(%musicbrainz_trackid%),$set(musicbrainz_trackid,%MUSICBRAINZ_RELEASETRACKID%)$delete(MUSICBRAINZ_RELEASETRACKID))
```

**Source:** [MusicBrainz Forum](https://community.metabrainz.org/t/converting-transcoding-and-keeping-mapping/686122)

### 6.3 Case Sensitivity

**Critical issue:** Some tools write lowercase:

| Canonical | Lowercase (broken) |
|-----------|-------------------|
| `MUSICBRAINZ_TRACKID` | `musicbrainz_trackid` |
| `MUSICBRAINZ_ALBUMID` | `musicbrainz_albumid` |

**Most tools are case-insensitive**, but Picard may have issues.

**Recommendation:** Always use uppercase for Vorbis/FLAC.

---

## 7. Edge Cases

### 7.1 Multiple Track IDs

**Scenario:** A recording appears on multiple releases

**ID3v2:** Multiple UFID frames with same owner
**Vorbis:** Multiple `MUSICBRAINZ_TRACKID` values

**Recommendation:** Keep all values; they represent valid alternatives.

### 7.2 Release Track ID vs Recording ID

**Recording ID (Track ID):** Identifies the specific musical work/performance
- Same across all releases
- `MUSICBRAINZ_TRACKID` / `UFID`

**Release Track ID:** Identifies the track on a specific release
- Different for each release
- `MUSICBRAINZ_RELEASETRACKID` / `TXXX:MusicBrainz Release Track Id`

**Important:** When ripping from a compilation, Release Track ID is critical for identifying the specific appearance.

### 7.3 Missing MBIDs After Conversion

**Common issue:** FFmpeg doesn't preserve MBIDs

> "FFmpeg doesn't [preserve MBIDs] but I just found out that fre:ac (freaccmd) does this correctly."

**Source:** [MusicBrainz Forum](https://community.metabrainz.org/t/converting-transcoding-and-keeping-mapping/686122)

**Solution:** Use ID Tag Processing DSP or Picard to restore MBIDs after conversion.

### 7.4 Format-Specific Limitations

**WMA/WAV:** Limited custom tag support
- May not support all MBID fields
- Consider keeping a parallel metadata file

**Opus:** Uses Ogg container
- Similar limitations to Vorbis
- Same MBID tag names work

---

## 8. User Impact Assessment

| Scenario | User Impact | Severity |
|----------|-------------|----------|
| Track ID lost on conversion | No MusicBrainz lookup | High |
| Album ID lost on conversion | No album lookup | High |
| Release Track ID lost | Incorrect release matching | Medium |
| MBIDs written with wrong case | Picard can't read | High |
| UFID not converted to TXXX | Track ID lost | High |

---

## 9. Validation Checklist

- [ ] Validate MBID format (UUID with hyphens)
- [ ] Normalize MBIDs to lowercase
- [ ] Handle UFID owner identifier correctly
- [ ] Convert TXXX → Vorbis correctly
- [ ] Convert Vorbis → MP4 freeform correctly
- [ ] Preserve multiple MBID values
- [ ] Handle missing fields gracefully
- [ ] Distinguish Release Track ID from Recording ID

---

## 10. Source Citations

| # | Source | Key Information |
|---|--------|-----------------|
| 1 | [Picard Tag Mapping](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html) | Complete MBID mapping |
| 2 | [MusicBrainz Forum](https://community.metabrainz.org/t/converting-transcoding-and-keeping-mapping/686122) | Transcoding and MBID preservation |
| 3 | [MusicBrainz spec](https://web.archive.org/web/20110120221258/musicbrainz.org/docs/specs/metadata_tags.html) | Original MBID format specification |
| 4 | [Picard Scripting docs](https://picard-docs.musicbrainz.org/en/latest/extending/scripting.html) | Tag normalization scripts |
| 5 | [RFC 4122 UUID](https://datatracker.ietf.org/doc/html/rfc4122) | UUID format specification |
