# Field Mapping: Genre, Year, and Date Formats Across Formats

## 1. Genre Field Mapping

### 1.1 ID3v1 Genre Table (0-191)

**ID3v1 genres are numbered 0-80 in the original spec, extended to 191.**

| # | Genre | # | Genre | # | Genre |
|---|-------|---|-------|---|-------|
| 0 | Blues | 27 | Ambient | 54 | Eurodance |
| 1 | Classic Rock | 28 | Trip-Hop | 55 | Dream |
| 2 | Country | 29 | Vocal | 56 | Southern Rock |
| 3 | Dance | 30 | Jazz+Funk | 57 | Comedy |
| 4 | Disco | 31 | Fusion | 58 | Gangsta |
| 5 | Funk | 32 | Trance | 59 | Top 40 |
| 6 | Grunge | 33 | Classical | 60 | Christian Rap |
| 7 | Hip-Hop | 34 | Instrumental | 61 | Pop/Funk |
| 8 | Jazz | 35 | Acid | 62 | Jungle |
| 9 | Metal | 36 | House | 63 | Native American |
| 10 | New Age | 37 | Game | 64 | Cabaret |
| 11 | Oldies | 38 | Sound Clip | 65 | New Wave |
| 12 | Other | 39 | Gospel | 66 | Psychedelic |
| 13 | Pop | 40 | Noise | 67 | Rave |
| 14 | R&B | 41 | AlternRock | 68 | Showtunes |
| 15 | Rap | 42 | Bass | 69 | Trailer |
| 16 | Reggae | 43 | Soul | 70 | Lo-Fi |
| 17 | Rock | 44 | Punk | 71 | Tribal |
| 18 | Techno | 45 | Space | 72 | Acid Punk |
| 19 | Industrial | 46 | Meditative | 73 | Acid Jazz |
| 20 | Alternative | 47 | Instrumental Pop | 74 | Polka |
| 21 | Ska | 48 | Instrumental Rock | 75 | Retro |
| 22 | Death Metal | 49 | Ethnic | 76 | Musical |
| 23 | Pranks | 50 | Gothic | 77 | Rock & Roll |
| 24 | Soundtrack | 51 | Darkwave | 78 | Hard Rock |
| 25 | Euro-Techno | 52 | Techno-Industrial | 79 | Folk |
| 26 | Synthpop | 53 | Electronic | 80-191 | (Extended) |

**Extended genres (80-191):** Rev, J-Rock, Sex, Macho, Narration, Synthpunk, Country-Rock, Polka, Satire, Throat Singer, Deloused, G-Funk, Drill, Barbershop

**Source:** [ID3v2.3 spec - Appendix A](https://id3.org/id3v2.3.0), [ID3v2.4 spec - Appendix A](https://id3.org/id3v2.4.0-frames)

### 1.2 ID3v2 TCON Format Variations

**ID3v2 TCON can store genre in multiple formats:**

**Format 1: Numeric reference**
```
TCON = "17"         # ID3v1 genre number
```

**Format 2: Numeric reference with parentheses**
```
TCON = "(17)"       # ID3v1 genre number in parentheses
TCON = "(17)(39)"   # Multiple genres
```

**Format 3: Freeform string**
```
TCON = "Rock"      # Direct genre name
TCON = "Electronic" # Direct genre name
```

**Format 4: Refinement (number + text)**
```
TCON = "(17)Rock"        # ID3v1 number + custom
TCON = "(4)Eurodisco"    # Dance + refinement
```

**Format 5: RX and CR (Remix/Cover)**
```
TCON = "(RX)"      # Remix
TCON = "(CR)"      # Cover
```

**Format 6: Escaped parenthesis**
```
TCON = "(17)((House)"   # "17 (House)"
```

**Source:** [ID3.org TCON spec](https://id3.org/id3v2.4.0-frames) — "Example: '21' $00 'Eurodisco' $00"

### 1.3 Vorbis/FLAC Genre

**Vorbis GENRE field:**
```
GENRE = "Rock"
GENRE = "Electronic"
GENRE = "Soundtrack"
```

**No numeric reference system — always freeform strings.**

**Multiple genres:**
```
GENRE = "Rock"
GENRE = "Alternative"
```
OR
```
GENRE = "Rock; Alternative"
```

### 1.4 MP4 Genre

**©gen atom:**
```
©gen = "Rock"
©gen = "Alternative"
```

**gnre atom (iTunes numeric):**
```
gnre = [(17)]  # Numeric, rarely used
```

**Source:** [Mutagen MP4 docs](https://mutagen.readthedocs.io/en/latest/api/mp4.html) — "'gnre' – not supported. Use '\\xa9gen' instead"

### 1.5 Genre Mapping Table

| Concept | ID3v2 | Vorbis | MP4 | WMA |
|---------|-------|--------|-----|-----|
| Genre | TCON | GENRE | ©gen | WM/Genre |
| iTunes Genre | TCON=(N) | (N/A) | gnre | (N/A) |

---

## 2. Date Field Formats

### 2.1 ID3v2 Date Frames

**ID3v2.3 TYER (Year):**
```
TYER = "2024"      # Year only, 4 digits
```

**ID3v2.4 TDRC (Recording Time):**
```
TDRC = "2024"      # Year only
TDRC = "2024-03"   # Year-month
TDRC = "2024-03-15" # ISO date
```

**Additional date frames in ID3v2.4:**
| Frame | Purpose | Format |
|-------|---------|--------|
| TDRC | Recording time | ISO 8601 |
| TDOR | Original release time | ISO 8601 |
| TDRL | Release time | ISO 8601 |
| TDTG | Tagging time | ISO 8601 |

**Source:** [ID3v2.4 spec](https://id3.org/id3v2.4.0-frames) — TDRC specification

### 2.2 Vorbis DATE

**Vorbis DATE field:**
```
DATE = "2024"           # Year
DATE = "2024-03"        # Year-month
DATE = "2024-03-15"     # ISO date
DATE = "2024-03-15T10:30:00Z"  # Full ISO 8601
```

**MusicBrainz convention:**
```
DATE = "2024-03-15"     # Full ISO date
```

**iTunes/r28 convention:**
```
DATE = "2024"           # Year only
```

### 2.3 MP4 ©day

**MP4 ©day atom:**
```
©day = "2024"           # Year only
©day = "2024-03-15T00:00:00Z"  # Full ISO 8601 with time
```

**Source:** [Mutagen MP4 docs](https://mutagen.readthedocs.io/en/latest/api/mp4.html) — date handling

### 2.4 Cross-Format Date Mapping

| Source | ID3v2.3 | ID3v2.4 | Vorbis | MP4 |
|--------|----------|---------|--------|-----|
| "2024" | TYER="2024" | TDRC="2024" | DATE="2024" | ©day="2024" |
| "2024-03" | TYER="2024" | TDRC="2024-03" | DATE="2024-03" | ©day="2024-03" |
| "2024-03-15" | TYER="2024" | TDRC="2024-03-15" | DATE="2024-03-15" | ©day="2024-03-15T00:00:00Z" |

### 2.5 Date Extraction Behavior

**Question: What does DBpoweramp display when given "2024-03-15"?**

Based on standard behavior:
- Display year: "2024" (extracted from date)
- Display full date: "March 15, 2024"
- ID3v2.3: Only 4-digit year stored → "2024"
- ID3v2.4/Vorbis: Full date preserved

**Recommendation:**
- Always write ID3v2.4 TDRC for full date preservation
- Write separate DATE field in Vorbis
- For MP4, write ©day with full ISO 8601

---

## 3. Year Extraction Rules

### 3.1 Extraction Patterns

| Input Format | Extracted Year | Notes |
|--------------|----------------|-------|
| "2024" | 2024 | Direct parse |
| "2024-03" | 2024 | Take year part |
| "2024-03-15" | 2024 | Take year part |
| "2024-03-15T10:30:00Z" | 2024 | Strip time |
| "©2024" | 2024 | Strip copyright symbol |
| "(P)2024" | 2024 | Strip phonogram symbol |
| "2024.03.15" | 2024 | Alternate separators |

### 3.2 Validation Rules

```python
import re
from datetime import datetime

def extract_year(date_string: str) -> int | None:
    """
    Extract 4-digit year from various date formats.
    """
    if not date_string:
        return None
    
    # Clean input
    date_string = date_string.strip()
    
    # Try ISO format first
    patterns = [
        r'^(\d{4})',           # 2024, 2024-03-15
        r'^(\d{4})-\d{2}-\d{2}',  # Match year in ISO
        r'^©?\(P\)?(\d{4})',   # ©2024 or (P)2024
    ]
    
    for pattern in patterns:
        match = re.match(pattern, date_string)
        if match:
            year = int(match.group(1))
            if 1900 <= year <= 2100:  # Reasonable range
                return year
    
    return None
```

### 3.3 Edge Cases

**Edge Case 1: Year with parentheses**
```
DATE = "(2024)"
```
→ Extract: 2024

**Edge Case 2: Range of years**
```
DATE = "2020-2024"
```
→ Extraction: 2020 (first year) or ambiguous

**Edge Case 3: Approximate year**
```
DATE = "circa 1985"
```
→ No numeric extraction possible

**Edge Case 4: Retroactive release date**
```
TDRC = "2024"      # Recording date
TDOR = "1975"      # Original release
```
→ Two different dates for different purposes

---

## 4. Cross-Format Date Conversion

### 4.1 Conversion Rules

```python
from datetime import datetime
from typing import Optional

def convert_date_to_id3v24(date_str: str) -> str:
    """
    Convert various date formats to ID3v2.4 TDRC format.
    """
    if not date_str:
        return ""
    
    date_str = date_str.strip()
    
    # Already ISO format
    if re.match(r'^\d{4}(-\d{2})?(-\d{2})?$', date_str):
        return date_str
    
    # Try to parse and reformat
    for fmt in ['%Y-%m-%d', '%Y-%m', '%Y', '%Y.%m.%d']:
        try:
            dt = datetime.strptime(date_str, fmt)
            # Return minimal sufficient format
            if dt.month == 1 and dt.day == 1:
                return dt.strftime('%Y')
            if dt.day == 1:
                return dt.strftime('%Y-%m')
            return dt.strftime('%Y-%m-%d')
        except ValueError:
            continue
    
    # Return as-is if can't parse
    return date_str

def convert_date_to_vorbis(date_str: str) -> str:
    """
    Convert various date formats to Vorbis DATE format.
    MusicBrainz convention: ISO date.
    """
    return convert_date_to_id3v24(date_str)

def convert_date_to_mp4(date_str: str) -> str:
    """
    Convert various date formats to MP4 ©day format.
    iTunes convention: Full ISO 8601 with time.
    """
    if not date_str:
        return ""
    
    date_str = date_str.strip()
    
    # Already full ISO with time
    if 'T' in date_str:
        return date_str
    
    # Add time component for MP4
    iso_base = convert_date_to_id3v24(date_str)
    if iso_base:
        return f"{iso_base}T00:00:00Z"
    
    return date_str
```

### 4.2 Year-Only Extraction

```python
def get_year_for_id3v23(date_str: str) -> str:
    """
    Extract year for ID3v2.3 TYER frame.
    Only supports 4-digit year.
    """
    year = extract_year(date_str)
    if year:
        return str(year)
    return ""
```

---

## 5. ID3v2.3 vs ID3v2.4 Date Handling

### 5.1 Version Differences

| Aspect | ID3v2.3 | ID3v2.4 |
|--------|---------|----------|
| Year frame | TYER (4 digits) | TDRC (full date) |
| Date precision | Year only | Year to second |
| TDAT frame | Day/month (deprecated) | Replaced by TDRC |
| TIME frame | Hour/minute (deprecated) | Replaced by TDRC |

### 5.2 Conversion Considerations

**When converting from ID3v2.4 to ID3v2.3:**
- TDRC="2024-03-15" → TYER="2024"
- Loss of precision (month and day lost)

**When converting from ID3v2.3 to ID3v2.4:**
- TYER="2024" → TDRC="2024"
- No additional precision available

**Recommendation:**
- Use ID3v2.4 TDRC when possible
- Preserve original date frames during conversion

---

## 6. Edge Cases and Corner Cases

### 6.1 Genre Edge Cases

**Edge Case 1: Multiple genres**
```
TCON = "(17)(39)"    # Rock + Gospel
```
→ Convert to Vorbis: GENRE="Rock"; GENRE="Gospel"

**Edge Case 2: Numeric with refinement**
```
TCON = "(4)Eurodisco"
```
→ Convert to Vorbis: GENRE="Eurodisco" (refinement lost)

**Edge Case 3: Unknown numeric genre**
```
TCON = "(150)"       # Non-standard genre number
```
→ Keep as string: "(150)" or extract "150"

**Edge Case 4: RX/CR special genres**
```
TCON = "(RX)"        # Remix
TCON = "(CR)"        # Cover
```
→ Keep as-is in all formats

### 6.2 Date Edge Cases

**Edge Case 1: Date before 1900**
```
DATE = "1842"
```
→ Rare but valid for classical music

**Edge Case 2: Future date**
```
DATE = "2050"
```
→ Possible for pre-release tracks

**Edge Case 3: Partial date in MP4**
```
©day = "2024-01"     # Only month, no day
```
→ Valid ISO 8601 partial date

**Edge Case 4: Multiple dates**
```
TDRC = "2024"        # Recording
TDOR = "1975"        # Original release
TDRL = "2024-03-15"  # This release
```
→ All three should be preserved

---

## 7. DBpoweramp Specific Behavior

### 7.1 What We Know

**Tag preservation during conversion:**
- DBpoweramp preserves standard tags
- Can strip specific tags via DSP effects
- Source: [dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/41137-music-converter-add-id-tag-encoder-encoder-settings)

**Unknown:**
- Does DBpoweramp use TYER or TDRC?
- Does DBpoweramp write full ISO dates to Vorbis?
- Does DBpoweramp convert numeric genres to names?

### 7.2 User Impact Assessment

| Scenario | User Impact | Severity |
|----------|-------------|----------|
| TYER only (ID3v2.3) | Year loses month/day | Low |
| Numeric genre not expanded | "17" instead of "Rock" | Medium |
| DATE="2024-03-15" → MP4 loses time | Minor inconsistency | Low |
| Original release date lost | Historical info missing | Medium |

---

## 8. Code Examples

### 8.1 Genre Conversion

```python
from typing import Optional
import re

# ID3v1 genre table (subset)
ID3V1_GENRES = {
    0: "Blues", 1: "Classic Rock", 2: "Country", 3: "Dance",
    4: "Disco", 5: "Funk", 6: "Grunge", 7: "Hip-Hop",
    8: "Jazz", 9: "Metal", 10: "New Age", 11: "Oldies",
    12: "Other", 13: "Pop", 14: "R&B", 15: "Rap",
    16: "Reggae", 17: "Rock", 18: "Techno", 19: "Industrial",
    20: "Alternative", 21: "Ska", 22: "Death Metal", 23: "Pranks",
    24: "Soundtrack", 25: "Euro-Techno", 26: "Synthpop", 27: "Ambient",
    28: "Trip-Hop", 29: "Vocal", 30: "Jazz+Funk", 31: "Fusion",
    32: "Trance", 33: "Classical", 34: "Instrumental", 35: "Acid",
    36: "House", 37: "Game", 38: "Sound Clip", 39: "Gospel",
    40: "Noise", 41: "AlternRock", 42: "Bass", 43: "Soul",
    44: "Punk", 45: "Space", 46: "Meditative", 47: "Instrumental Pop",
    48: "Instrumental Rock", 49: "Ethnic", 50: "Gothic",
    51: "Darkwave", 52: "Techno-Industrial", 53: "Electronic",
    54: "Eurodance", 55: "Dream", 56: "Southern Rock",
    57: "Comedy", 58: "Gangsta", 59: "Top 40",
    60: "Christian Rap", 61: "Pop/Funk", 62: "Jungle",
    63: "Native American", 64: "Cabaret", 65: "New Wave",
    66: "Psychedelic", 67: "Rave", 68: "Showtunes",
    69: "Lo-Fi", 70: "Tribal", 71: "Acid Punk",
    72: "Acid Jazz", 73: "Polka", 74: "Retro",
    75: "Musical", 76: "Rock & Roll", 77: "Hard Rock", 78: "Folk",
}

def parse_tcon(tcon: str) -> list[str]:
    """
    Parse ID3v2 TCON field into list of genre names.
    """
    if not tcon:
        return []
    
    genres = []
    tcon = tcon.strip()
    
    # Check for numeric references
    numeric_pattern = r'\((\d+)\)'
    matches = re.findall(numeric_pattern, tcon)
    
    if matches:
        for num_str in matches:
            num = int(num_str)
            if num in ID3V1_GENRES:
                genres.append(ID3V1_GENRES[num])
            else:
                genres.append(num_str)  # Keep unknown number
        
        # Check for refinement after numbers
        refinement = re.sub(numeric_pattern, '', tcon).strip()
        if refinement:
            genres.append(refinement)
    else:
        genres.append(tcon)
    
    return genres

def format_genre_for_vorbis(genres: list[str]) -> dict:
    """
    Format genre list for Vorbis comment.
    """
    return {'GENRE': genres}

def format_genre_for_mp4(genres: list[str]) -> str:
    """
    Format genre for MP4 ©gen atom.
    Takes first genre only.
    """
    if genres:
        return genres[0]
    return ""
```

### 8.2 Date Validation

```python
from datetime import datetime
from typing import Optional

def validate_and_normalize_date(date_str: str) -> Optional[str]:
    """
    Validate date string and return normalized ISO format.
    """
    if not date_str:
        return None
    
    date_str = date_str.strip()
    
    # Patterns in order of specificity
    patterns = [
        ('%Y-%m-%dT%H:%M:%SZ', None),  # ISO 8601 with time
        ('%Y-%m-%dT%H:%M:%S', None),
        ('%Y-%m-%d', None),
        ('%Y-%m', None),
        ('%Y', None),
        ('%Y.%m.%d', None),
        ('%d.%m.%Y', None),  # European format
    ]
    
    for fmt, _ in patterns:
        try:
            dt = datetime.strptime(date_str, fmt)
            # Return canonical ISO format
            if dt.month == 1 and dt.day == 1 and dt.hour == 0:
                return dt.strftime('%Y')
            if dt.day == 1 and dt.hour == 0:
                return dt.strftime('%Y-%m')
            return dt.strftime('%Y-%m-%d')
        except ValueError:
            continue
    
    # Can't parse, return original
    return date_str if date_str else None
```

---

## 9. Validation Checklist

- [ ] Parse numeric genre "(17)" → "Rock"
- [ ] Parse multiple genres "(17)(39)" → ["Rock", "Gospel"]
- [ ] Parse refinement "(4)Eurodisco" → ["Eurodisco"]
- [ ] Convert ID3 numeric to Vorbis freeform
- [ ] Convert Vorbis to ID3v2 numeric when possible
- [ ] Extract year from "2024-03-15" → "2024"
- [ ] Preserve month from "2024-03" → TDRC="2024-03"
- [ ] Convert MP4 ©day to ID3v2 TDRC correctly
- [ ] Handle RX/CR special genres
- [ ] Handle multiple date fields (TDRC, TDOR, TDRL)
- [ ] Validate year range (1900-2100)

---

## 10. Source Citations

| # | Source | Key Information |
|---|--------|-----------------|
| 1 | [ID3v2.3 spec](https://id3.org/id3v2.3.0) | TYER, TCON, genre table |
| 2 | [ID3v2.4 spec](https://id3.org/id3v2.4.0-frames) | TDRC, TDOR, TDRL, TDTG |
| 3 | [Mutagen MP4 docs](https://mutagen.readthedocs.io/en/latest/api/mp4.html) | ©day, gnre atoms |
| 4 | [Kid3 Handbook](https://kid3.sourceforge.io/kid3_en.html) | Unified field mapping |
| 5 | [Picard Tag Mapping](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html) | DATE field convention |
| 6 | [Hydrogenaudio](https://wiki.hydrogenaudio.org/index.php?title=Tag_Mapping) | Genre mapping across formats |
| 7 | [dBpoweramp Forum](https://forum.dbpoweramp.com/) | DBpoweramp tag behavior |
