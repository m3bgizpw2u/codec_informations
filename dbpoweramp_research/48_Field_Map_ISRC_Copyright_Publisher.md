# Field Mapping: ISRC, Copyright, and Publisher Across Formats

## 1. ISRC (International Standard Recording Code)

### 1.1 Overview

**Purpose:** Unique identifier for sound recordings
**Format:** 12 characters: `CC-XXX-YY-NNNNN`
- CC: Country code (2 letters)
- XXX: Registrant code (3 alphanumeric)
- YY: Year (2 digits)
- NNNNN: Recording number (5 digits)

### 1.2 Format Mapping

| Format | Tag | Example |
|--------|-----|---------|
| ID3v2 | `TSRC` | `USRC1-21-12345` |
| Vorbis | `ISRC` | `USRC1-21-12345` |
| APEv2 | `ISRC` | `USRC1-21-12345` |
| MP4 | `----:com.apple.iTunes:ISRC` | `USRC1-21-12345` |
| WMA | `WM/ISRC` | `USRC1-21-12345` |
| RIFF | (none) | No native support |

**Source:** [ID3v2.4 spec](https://id3.org/id3v2.4.0-frames)

### 1.3 ID3v2 TSRC

**Specification:** "The 'ISRC' frame should contain the International Standard Recording Code [ISRC] (12 characters)."

**Format:** Plain 12-character string

**Validation:**
```python
import re

def validate_isrc(isrc: str) -> bool:
    """Validate ISRC format."""
    # ISRC format: CC-XXX-YY-NNNNN
    pattern = r'^[A-Z]{2}[A-Z0-9]{3}-\d{2}-\d{5}$'
    return bool(re.match(pattern, isrc.upper()))
```

---

## 2. Copyright

### 2.1 Format Comparison

| Format | Tag | Format Requirement | Example |
|--------|-----|-------------------|---------|
| ID3v2 | `TCOP` | Must start with year | `(C) 2024 Artist Name` |
| Vorbis | `COPYRIGHT` | Freeform | `(C) 2024 Artist Name` |
| MP4 | `cprt` | Freeform | `(C) 2024 Artist Name` |
| APEv2 | `Copyright` | Freeform | `(C) 2024 Artist Name` |
| WMA | `Copyright` | URL to terms | `http://example.com/copyright` |
| RIFF | `ICOP` | Freeform | `(C) 2024` |

**Source:** [ID3v2.4 spec](https://id3.org/id3v2.4.0-frames)

### 2.2 ID3v2 TCOP Specification

> "The 'Copyright message' frame, in which the string must begin with a year and a space character (making five characters), is intended for the copyright holder of the original sound, not the audio file itself."

**Correct format:**
```
TCOP = "2024 Artist Name Inc."
TCOP = "(C) 2024 Artist Name"  # Common but non-standard
```

**Note:** W3C recommends `(C)` character, not `(c)`

### 2.3 Copyright vs Legal Information

**TCOP:** Copyright holder (intended use)
**WCOP:** URL to copyright/legal terms

**Distinction:**
- TCOP: "2024 Sony Music Entertainment"
- WCOP: "https://www.sonymusic.com/copyright"

---

## 3. Publisher / Label

### 3.1 Format Comparison

| Format | Tag | Aliases | Notes |
|--------|-----|---------|-------|
| ID3v2 | `TPUB` | — | Record label |
| Vorbis | `LABEL` | `ORGANIZATION` | Label name |
| Vorbis | `PUBLISHER` | (not standard) | Label name |
| APEv2 | `Label` | — | Label name |
| MP4 | `----:com.apple.iTunes:LABEL` | (custom) | Freeform |
| WMA | `WM/Publisher` | — | Label name |

**Source:** [MusicBrainz Picard Tag Mapping](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html)

### 3.2 Vorbis ORGANIZATION

**Convention:** `ORGANIZATION` = Record label name

**Note:** Originally meant for encoding organization, repurposed for label

### 3.3 Mapping Between Formats

```python
def map_publisher_to_format(publisher: str, format: str) -> dict:
    """Map publisher/label to format-specific tag."""
    if format == 'mp3':
        return {'TPUB': [publisher]}
    elif format in ('flac', 'ogg'):
        return {'LABEL': [publisher]}  # Preferred
    elif format == 'mp4':
        return {'----:com.apple.iTunes:LABEL': [publisher]}
    elif format == 'wma':
        return {'WM/Publisher': [publisher]}
    return {}
```

---

## 4. Catalog Number

### 4.1 Format Comparison

| Format | Tag | Example |
|--------|-----|---------|
| ID3v2 | `TXXX:CATALOGNUMBER` | `CAT-1234` |
| Vorbis | `CATALOGNUMBER` | `CAT-1234` |
| APEv2 | `CatalogNumber` | `CAT-1234` |
| MP4 | `----:com.apple.iTunes:CATALOGNUMBER` | `CAT-1234` |
| WMA | `WM/CatalogNo` | `CAT-1234` |

**Source:** [MusicBrainz Picard Tag Mapping](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html)

### 4.2 Multiple Catalog Numbers

**Some releases have multiple:**
```
CATALOGNUMBER = "CAT-1234"
CATALOGNUMBER = "LP-5678"
```

**Label-specific:**
```
CATALOGNUMBER = "EMI-1234"
CATALOGNUMBER = "COLUMBIA-5678"
```

### 4.3 Picard Standardization

**Internal name:** `catalognumber`

**Note:** Use `DISCOGS_CATALOGNUMBER` or `DISCOGS_LABEL` for Discogs-specific numbers

---

## 5. UPC / Barcode

### 5.1 UPC (Universal Product Code)

| Format | Tag | Example |
|--------|-----|---------|
| ID3v2 | `TXXX:UPC` | `012345678905` |
| Vorbis | `UPC` | `012345678905` |
| APEv2 | `UPC` | `012345678905` |
| MP4 | `----:com.apple.iTunes:UPC` | `012345678905` |
| WMA | `WM/Barcode` | `012345678905` |

**Source:** [MusicBrainz Picard Tag Mapping](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html)

### 5.2 EAN-13 Barcode

**Format:** 13 digits
**UPC-A:** 12 digits (US)
**EAN-13:** 13 digits (International)

**Validation:**
```python
def validate_barcode(barcode: str) -> bool:
    """Validate UPC/EAN barcode."""
    # Remove spaces and dashes
    cleaned = re.sub(r'[\s-]', '', barcode)
    
    # Check length: 8, 12, 13, or 14 digits
    if len(cleaned) not in (8, 12, 13, 14):
        return False
    
    # Check all digits
    if not cleaned.isdigit():
        return False
    
    return True
```

---

## 6. ASIN (Amazon Standard Identification Number)

### 6.1 Format

**Tag:** `TXXX:ASIN` (ID3), `ASIN` (Vorbis)

**Format:** 10 alphanumeric characters

**Example:** `B0000000XX`

**Source:** [MusicBrainz Picard Tag Mapping](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html)

### 6.2 Mapping Table

| Format | Tag | Notes |
|--------|-----|-------|
| ID3v2 | `TXXX:ASIN` | User-defined frame |
| Vorbis | `ASIN` | Standard field |
| APEv2 | `ASIN` | Standard field |
| MP4 | `----:com.apple.iTunes:ASIN` | Freeform |
| WMA | `ASIN` | Standard field |

---

## 7. Cross-Format Conversion

### 7.1 Conversion Matrix

```python
class CommercialTagConverter:
    """Convert commercial identifier tags between formats."""
    
    TAG_MAPPING = {
        'isrc': {
            'mp3': 'TSRC',
            'flac': 'ISRC',
            'mp4': '----:com.apple.iTunes:ISRC',
            'wma': 'WM/ISRC',
        },
        'copyright': {
            'mp3': 'TCOP',
            'flac': 'COPYRIGHT',
            'mp4': 'cprt',
            'wma': 'Copyright',
        },
        'label': {
            'mp3': 'TPUB',
            'flac': 'LABEL',
            'mp4': '----:com.apple.iTunes:LABEL',
            'wma': 'WM/Publisher',
        },
        'catalognumber': {
            'mp3': 'TXXX:CATALOGNUMBER',
            'flac': 'CATALOGNUMBER',
            'mp4': '----:com.apple.iTunes:CATALOGNUMBER',
            'wma': 'WM/CatalogNo',
        },
        'upc': {
            'mp3': 'TXXX:UPC',
            'flac': 'UPC',
            'mp4': '----:com.apple.iTunes:UPC',
            'wma': 'WM/Barcode',
        },
        'asin': {
            'mp3': 'TXXX:ASIN',
            'flac': 'ASIN',
            'mp4': '----:com.apple.iTunes:ASIN',
            'wma': 'ASIN',
        },
    }
    
    def convert(self, tag_name: str, value: str, from_format: str, 
                to_format: str) -> dict:
        """Convert a commercial tag to target format."""
        tag_key = self.TAG_MAPPING.get(tag_name, {})
        target_tag = tag_key.get(to_format)
        
        if not target_tag:
            return {}
        
        return {target_tag: [value]}
```

---

## 8. Edge Cases

### 8.1 Multiple Catalog Numbers

**Scenario:** Release from two labels

**Source Vorbis:**
```
CATALOGNUMBER=SN-001
CATALOGNUMBER=LP-002
```

**Destination ID3:**
```
TXXX:CATALOGNUMBER=SN-001
TXXX:CATALOGNUMBER=LP-002
```

**Note:** Multiple TXXX frames with same description

### 8.2 Copyright Year Mismatch

**Scenario:** File recorded in 2023, released in 2024

**TCOP:** Should show release year "2024"
**TDOR:** Original recording year "2023"

### 8.3 Label vs Publisher

**Distinction:**
- **Label:** Record label that released the album
- **Publisher:** Music publishing company (separate from label)

**Recommendation:** Use separate tags for each if available

### 8.4 Barcode Variations

**UPC-A:** 12 digits (US)
**EAN-13:** 13 digits (International)
**GTIN-14:** 14 digits (Shipping)

**Conversion:** Add leading zero to convert UPC-A to EAN-13

---

## 9. DBpoweramp Behavior

### 9.1 What We Know

- DBpoweramp preserves standard tags during conversion
- DSP effects can manipulate tags
- Source: [dBpoweramp Forum](https://forum.dbpoweramp.com/)

### 9.2 Unknowns

- Does DBpoweramp preserve TXXX custom frames like CATALOGNUMBER?
- Does DBpoweramp handle multiple catalog numbers?
- Does DBpoweramp validate ISRC format?

### 9.3 User Impact

| Scenario | User Impact | Severity |
|----------|-------------|----------|
| ISRC lost | No recording lookup | High |
| Catalog number lost | No release lookup | Medium |
| UPC lost | No product lookup | Medium |
| ASIN lost | No Amazon link | Low |
| Copyright lost | Legal info missing | Medium |

---

## 10. Validation Checklist

- [ ] Validate ISRC format (12 chars, CC-XXX-YY-NNNNN)
- [ ] Parse TCOP copyright year
- [ ] Write TCOP with year prefix
- [ ] Read catalog number from TXXX:CATALOGNUMBER
- [ ] Write catalog number to TXXX:CATALOGNUMBER
- [ ] Read UPC from TXXX:UPC
- [ ] Write UPC to TXXX:UPC
- [ ] Handle multiple catalog numbers
- [ ] Preserve ASIN across formats
- [ ] Map ORGANIZATION to TPUB when needed

---

## 11. Source Citations

| # | Source | Key Information |
|---|--------|-----------------|
| 1 | [ID3v2.4 spec](https://id3.org/id3v2.4.0-frames) | TSRC, TCOP specifications |
| 2 | [Picard Tag Mapping](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html) | Complete commercial tag mapping |
| 3 | [Music-metadata docs](https://borewit.github.io/music-metadata/doc/common_metadata.html) | Common metadata fields |
| 4 | [Mutagen specs ID3v2](http://mutagen-specs.readthedocs.io/en/latest/id3/id3v2.4.0-frames.html) | TXXX frame details |
| 5 | [Mp3tag mapping](https://docs.mp3tag.de/mapping-table/) | Tag field mapping table |
| 6 | [ISO 3901 ISRC](https://www.iso.org/standard/8515.html) | ISRC standard |
