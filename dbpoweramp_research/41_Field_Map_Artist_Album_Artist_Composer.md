# Field Mapping: Artist, Album Artist, and Composer Across Formats

## 1. Format-Specific Storage Mechanisms

### 1.1 ID3v2 (MP3)

**TPE1 Frame (Lead Performer/Soloist/Principal Performer)**
- **Standard meaning:** Primary artist / lead performer
- **Source:** [ID3v2.4 spec](https://id3.org/id3v2.4.0-frames)

**TPE2 Frame (Band/Orchestra/Accompaniment)**
- **Standard meaning:** Album artist / band
- **iTunes usage:** iTunes uses TPE2 for "Album Artist"
- **Source:** [Joel Verhagen blog](https://www.joelverhagen.com/blog/2010/12/how-itunes-uses-id3-tags)

**TPE3 Frame (Conductor/Performer Refinement)**
- **Standard meaning:** Conductor
- **Source:** [Microsoft WMA docs](https://learn.microsoft.com/en-us/windows/win32/wmformat/id3-tag-support)

**TPE4 Frame (Interpreted, Remixed, or Otherwise Modified By)**
- **Standard meaning:** Remixer, arranger
- **Source:** [ID3v2.4 spec](https://id3.org/id3v2.4.0-frames)

**TCOM Frame (Composer)**
- **Format:** Text string
- **Source:** [ID3v2.4 spec](https://id3.org/id3v2.4.0-frames)

**TXXX:MULTI-ARTIST (Custom)**
- **Purpose:** Multi-value artist list
- **Format:** Artists separated by delimiter
- **Source:** [Picard Tag Mapping](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html)

### 1.2 Vorbis Comment / FLAC

**ARTIST Field**
- **Format:** Single or multiple values
- **Source:** [Xiph VorbisComment spec](https://wiki.xiph.org/VorbisComment)

**ALBUMARTIST Field**
- **Format:** Single or multiple values
- **Source:** [Picard Tag Mapping](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html)
- **Aliases:** `ALBUMARTIST`, `ALBUM ARTIST`, `ALBUM_ARTIST`

**COMPOSER Field**
- **Format:** Single or multiple values
- **Source:** [Picard Tag Mapping](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html)

**ARTISTS Field**
- **Purpose:** Multi-value artist list (Picard extension)
- **Format:** Semicolon-separated or multiple values
- **Source:** [Picard Tag Mapping](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html)

**ALBUMARTISTS Field**
- **Purpose:** Multi-value album artist list
- **Source:** [Picard Tag Mapping](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html)

### 1.3 MP4 / M4A / iTunes

**©ART Atom (Artist)**
- **Format:** Text
- **Source:** [Mutagen MP4 docs](https://mutagen.readthedocs.io/en/latest/api/mp4.html)

**aART Atom (Album Artist)**
- **Format:** Text
- **Source:** [Mutagen MP4 docs](https://mutagen.readthedocs.io/en/latest/api/mp4.html)

**©wrt Atom (Composer/Writer)**
- **Format:** Text
- **Source:** [Mutagen MP4 docs](https://mutagen.readthedocs.io/en/latest/api/mp4.html)

### 1.4 RIFF INFO / WAV

| Field | RIFF INFO | Meaning |
|-------|-----------|---------|
| IART | Artist | Lead performer |
| IPRO | Album artist | Band/orchestra |
| ICMT | Comment | Additional info |
| ICRD | Date | Release date |
| IGNR | Genre | Content type |
| INAM | Title | Track name |
| IPRD | Album | Album name |

### 1.5 ASF / WMA

| Field | WMA/ASF | Meaning |
|-------|---------|---------|
| Author | Artist | Lead performer |
| WM/AlbumArtist | Album artist | Band/orchestra |
| WM/Composer | Composer | Composer |

---

## 2. Artist vs Album Artist Distinction

### 2.1 Conceptual Model

**Artist (Lead Performer):**
- The primary performing entity for THIS track
- May differ from album artist on:
  - Compilation albums
  - Guest appearances
  - Various artist releases
- Stored in: TPE1 (ID3), ARTIST (Vorbis), ©ART (MP4)

**Album Artist:**
- The artist credited at the album level
- Usually consistent across all tracks
- Used for:
  - Album-based browsing
  - Sorting
  - Display on album view
- Stored in: TPE2 (ID3), ALBUMARTIST (Vorbis), aART (MP4)

### 2.2 Cross-Format Mapping

| Concept | ID3v2 | Vorbis | MP4 | WMA |
|---------|-------|--------|-----|-----|
| Artist | TPE1 | ARTIST | ©ART | Author |
| Album Artist | TPE2 | ALBUMARTIST | aART | WM/AlbumArtist |
| Composer | TCOM | COMPOSER | ©wrt | WM/Composer |
| Conductor | TPE3 | CONDUCTOR | ---- | WM/Conductor |

### 2.3 Fallback Behavior

**When ALBUMARTIST is missing:**
1. Check TPE2 (ID3) / ALBUMARTIST (Vorbis) / aART (MP4)
2. Fall back to Artist / TPE1 / ARTIST / ©ART
3. Display uses fallback; sorting may be affected

**DBpoweramp behavior (unknown):**
- Does DBpoweramp preserve TPE2 as-is during conversion?
- Does DBpoweramp create TPE2 from ARTIST if missing?
- Source: [dBpoweramp Forum](https://forum.dbpoweramp.com/) — no explicit documentation found

---

## 3. Compilation Albums

### 3.1 Detection Methods

**Compilation detection approaches:**

1. **Album Artist field:**
   - Empty or "Various Artists"
   - Multiple different artists per album

2. **Compilation flag:**
   - TCMP=1 (ID3)
   - cpil=true (MP4)
   - COMPILATION=1 (Vorbis)
   - WM/IsCompilation=true (WMA)

3. **iTunes-specific:**
   - TCMP frame (iTunes extension)
   - Source: [Hydrogenaudio Forum](https://hydrogenaudio.org/index.php/topic,30752.0.html)

### 3.2 Handling in Conversion

**Source: ID3 (compilation track):**
```
TPE1 = "Track Artist"
TPE2 = "Various Artists"
TCMP = 1
```

**Destination: Vorbis (compilation):**
```
ARTIST = "Track Artist"
ALBUMARTIST = "Various Artists"
COMPILATION = 1
```

**Destination: MP4 (compilation):**
```
©ART = "Track Artist"
aART = "Various Artists"
cpil = true
```

### 3.3 Edge Cases

**Edge Case 1: Self-titled compilation**
- Soundtrack with one main artist
- TCMP may still be set
- Artist ≠ Album Artist but not "Various Artists"

**Edge Case 2: Multi-artist single**
- Single track with multiple artists
- ARTIST="Artist 1; Artist 2" or "Artist 1 feat. Artist 2"
- TCMP not set (not a compilation album)

**Edge Case 3: Classical compilations**
- Multiple performers on same album
- Various conductors, orchestras
- Album Artist may be composer or label

---

## 4. Featured Artist Handling

### 4.1 Storage Formats

**Format 1: "feat." in title (common)**
```
Title = "Song Name feat. Guest Artist"
Artist = "Main Artist"
```

**Format 2: Semicolon separator (Picard standard)**
```
Title = "Song Name"
ARTIST = "Main Artist"
Artist = "Main Artist; Guest Artist"
```

**Format 3: Separate roles (advanced)**
```
TPE1 = "Main Artist"
TPE4 = "Guest Artist"
```

### 4.2 Artist Separator Patterns

| Pattern | Example | Common In |
|---------|---------|----------|
| `feat.` | "Artist feat. Guest" | Radio, streaming |
| `ft.` | "Artist ft. Guest" | Radio, streaming |
| `featuring` | "Artist featuring Guest" | iTunes |
| `with` | "Artist with Guest" | Classical, jazz |
| `;` | "Artist; Guest" | Picard, beets |
| ` / ` | "Artist / Guest" | Classical |

### 4.3 Splitting and Joining

**Splitting on conversion:**
```
Source: ARTIST = "Artist 1; Artist 2"
↓
MP3: TPE1 = "Artist 1" (first value)
MP3: TPE4 = "Artist 2" (additional performer)
OR
MP3: TPE1 = "Artist 1" (joined)
```

**Joining on conversion:**
```
Source: TPE1 = "Artist 1"
Source: TPE4 = "Artist 2"
↓
Vorbis: ARTIST = "Artist 1; Artist 2"
```

**Source:** [Picard Tag Mapping](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html) — ARTISTS and ALBUMARTISTS fields

---

## 5. Composer and Conductor

### 5.1 Composer Field Mapping

| Format | Tag | Notes |
|--------|-----|-------|
| ID3v2 | TCOM | Composer |
| Vorbis | COMPOSER | Standard field |
| MP4 | ©wrt | iTunes "writer" atom |
| WMA | WM/Composer | Maps to TCOM |
| RIFF | (none) | No native composer |

### 5.2 Conductor Field Mapping

| Format | Tag | Notes |
|--------|-----|-------|
| ID3v2 | TPE3 | Conductor/Performer refinement |
| Vorbis | CONDUCTOR | Standard field |
| MP4 | ----:com.apple.iTunes:CONDUCTOR | Custom freeform |
| WMA | WM/Conductor | Maps to TPE3 |
| RIFF | (none) | No native conductor |

### 5.3 Code Example: Parsing Multi-Value Artist Fields

```python
import unicodedata
from typing import Optional

ARTIST_SEPARATORS = [';', ' / ', '/', 'feat.', 'ft.', 'featuring', ' with ', ', ']

def split_artists(artist_string: str) -> list[str]:
    """
    Split artist string into individual artists.
    Handles multiple separator conventions.
    """
    artists = [artist_string]
    
    for sep in ARTIST_SEPARATORS:
        result = []
        for artist in artists:
            parts = artist.split(sep)
            result.extend([p.strip() for p in parts])
        if len(result) > len(artists):
            artists = result
    
    # Filter empty and normalize
    artists = [a.strip() for a in artists if a.strip()]
    
    # Normalize Unicode
    artists = [unicodedata.normalize('NFC', a) for a in artists]
    
    return artists

def join_artists(artists: list[str], format: str = 'vorbis') -> str:
    """
    Join artists into single string based on format convention.
    """
    if format == 'vorbis':
        return '; '.join(artists)
    elif format == 'id3':
        return '; '.join(artists)  # Or ' / ' for classical
    elif format == 'title':
        return ' feat. '.join(artists)
    else:
        return '; '.join(artists)
```

---

## 6. Sort Fields

### 6.1 ID3v2 Sort Tags

| Frame | Field | Example |
|-------|-------|---------|
| TSOP | Performer sort order | "Beatles, The" → "Beatles, The" |
| TSOA | Album sort order | "Beatles, The" → "Beatles, The" |
| TSOT | Title sort order | "The Song" → "Song, The" |
| TSOC | Composer sort order | "Bach, J.S." → "Bach, J.S." |
| TSOD | Disc subtitle sort | |

**Source:** [Microsoft WMA docs](https://learn.microsoft.com/en-us/windows/win32/wmformat/id3-tag-support) — WM/AlbumSortOrder, WM/ArtistSortOrder, WM/TitleSortOrder

### 6.2 Vorbis/FLAC Sort Fields

| Field | Example |
|-------|---------|
| ALBUMSORT | Album sort (album artist) |
| ARTISTSORT | Performer sort |
| TITLESORT | Title sort |
| COMPOSERSORT | Composer sort |

### 6.3 MP4 Sort Fields

| Atom | Meaning |
|------|---------|
| soaa | Album artist sort |
| soal | Album sort |
| sonm | Title sort |
| soco | Composer sort |
| sopn | Performer sort |

### 6.4 Canonicalization Examples

| Display Name | Sort Name | Convention |
|--------------|-----------|------------|
| "The Beatles" | "Beatles, The" | Leading article moved |
| "A Tribe Called Quest" | "Tribe Called Quest, A" | Leading article moved |
| "Los Tigres del Norte" | "Tigres del Norte, Los" | Leading article moved |
| "St. Vincent" | "Saint Vincent" | Abbreviation expanded |

---

## 7. Edge Cases

### 7.1 Single vs Multi-Value

**Single artist (most common):**
```
ARTIST = "Taylor Swift"
```

**Multi-value (various artists):**
```
ARTIST = "Artist One"
ARTIST = "Artist Two"
ARTIST = "Artist Three"
```
OR
```
ARTISTS = "Artist One; Artist Two; Artist Three"
```

### 7.2 Artist with Role

**Format 1: Role in parentheses**
```
COMPOSER = "John Williams (orchestrator)"
```

**Format 2: Separate fields (Picard)**
```
COMPOSER = "John Williams"
ARRANGER = "John Powell"
```

### 7.3 Unicode and Normalization

**Issue:** Same artist may be encoded differently:
- NFC: "Björk" (composed)
- NFD: "Bjo\u0308rk" (decomposed)

**Solution:** Normalize to NFC before comparison and storage.
```python
import unicodedata
artist = unicodedata.normalize('NFC', artist)
```

### 7.4 Empty Fields

| Scenario | Behavior |
|----------|----------|
| Empty ARTIST | Display "Unknown Artist" |
| Empty ALBUMARTIST | Fall back to ARTIST |
| Empty COMPOSER | Leave unset |
| Empty TPE2 in ID3 | Fall back to TPE1 |

### 7.5 Artist Name Variations

| Variation | Example |
|-----------|---------|
| With punctuation | "AC/DC" vs "ACDC" |
| With special chars | "Mötley Crüe" |
| Transliteration | "Kylie Minogue" |
| Legal name | "Snoop Dogg" vs "Calvin Broadus" |

---

## 8. DBpoweramp Specific Behavior

### 8.1 What We Know

Based on public documentation:

**Tag preservation:**
- DBpoweramp preserves standard ID3v2 tags during conversion
- Can configure to strip or update specific tags
- Source: [dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/41137-music-converter-add-id-tag-encoder-encoder-settings)

**Multi-artist handling:**
- Uses standard tag fields (TPE1, TPE2)
- Supports DSP effects for tag manipulation
- Source: [dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/batch-converter/33132)

### 8.2 What We Don't Know

- Does DBpoweramp read ARTISTS (Picard plural) field?
- How does DBpoweramp handle semicolon-separated artists?
- Does DBpoweramp normalize Unicode artist names?
- Does DBpoweramp preserve sort fields?

### 8.3 User Impact Assessment

| Scenario | User Impact | Severity |
|----------|-------------|----------|
| Sort fields not preserved | Wrong alphabetical ordering | Medium |
| Multi-value artists collapsed | Missing featured artists | High |
| ARTIST/ALBUMARTIST confused | Wrong album grouping | High |
| Unicode not normalized | Duplicates in library | Low |

---

## 9. Code Examples

### 9.1 Python: Artist Field Normalization

```python
from typing import Optional
import unicodedata

def normalize_artist_name(name: str) -> str:
    """
    Normalize artist name for consistent storage.
    """
    # Trim whitespace
    name = name.strip()
    
    # Normalize Unicode (NFC)
    name = unicodedata.normalize('NFC', name)
    
    # Collapse multiple spaces
    import re
    name = re.sub(r'\s+', ' ', name)
    
    return name

def extract_lead_artist(artist_string: str) -> str:
    """
    Extract primary artist from multi-value or separated string.
    """
    # Handle semicolon separation
    if ';' in artist_string:
        return artist_string.split(';')[0].strip()
    
    # Handle "feat." pattern
    import re
    match = re.match(r'^(.+?)\s+(?:feat\.?|ft\.?|featuring)\s+', artist_string, re.IGNORECASE)
    if match:
        return match.group(1).strip()
    
    return artist_string.strip()

def merge_artist_fields(tpe1: Optional[str], tpe4: Optional[str] = None) -> str:
    """
    Merge TPE1 and TPE4 into single artist string.
    """
    if tpe4:
        return f"{tpe1}; {tpe4}"
    return tpe1 or ""
```

### 9.2 Python: Album Artist Fallback

```python
def get_album_artist(
    albumartist: Optional[str],
    artist: Optional[str]
) -> Optional[str]:
    """
    Get album artist with proper fallback chain.
    """
    # Priority order:
    # 1. Explicit album artist
    # 2. Artist (fallback)
    # 3. "Various Artists" for compilations
    # 4. None
    
    if albumartist and albumartist.strip():
        return albumartist.strip()
    
    if artist and artist.strip():
        return artist.strip()
    
    return None

def is_various_artists(albumartist: Optional[str]) -> bool:
    """
    Check if album is various artists based on album artist field.
    """
    if not albumartist:
        return True
    
    va_patterns = [
        'various artists',
        'various',
        'va',
        'multiple artists',
        'different artists',
    ]
    
    return albumartist.lower().strip() in va_patterns
```

---

## 10. Validation Checklist

- [ ] Artist field maps correctly to TPE1 / ARTIST / ©ART
- [ ] Album Artist field maps to TPE2 / ALBUMARTIST / aART
- [ ] Composer field maps to TCOM / COMPOSER / ©wrt
- [ ] Multi-value artists handled correctly
- [ ] Featured artist patterns detected and preserved
- [ ] Sort fields preserved across conversions
- [ ] Fallback chain: ALBUMARTIST → ARTIST
- [ ] Unicode normalized to NFC
- [ ] Empty vs missing distinction preserved
- [ ] Leading/trailing whitespace trimmed

---

## 11. Source Citations

| # | Source | Key Information |
|---|--------|-----------------|
| 1 | [ID3v2.4 spec](https://id3.org/id3v2.4.0-frames) | TPE1-4, TCOM frame specifications |
| 2 | [Picard Tag Mapping](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html) | ARTISTS, ALBUMARTISTS plural fields |
| 3 | [Mutagen MP4 docs](https://mutagen.readthedocs.io/en/latest/api/mp4.html) | ©ART, aART, ©wrt atoms |
| 4 | [Microsoft WMA docs](https://learn.microsoft.com/en-us/windows/win32/wmformat/id3-tag-support) | WM/AlbumArtist, WM/Composer |
| 5 | [Joel Verhagen blog](https://www.joelverhagen.com/blog/2010/12/how-itunes-uses-id3-tags) | iTunes ID3 tag usage including TCP |
| 6 | [Hydrogenaudio Forum](https://hydrogenaudio.org/index.php/topic,30752.0.html) | TCMP compilation flag |
| 7 | [Kid3 Handbook](https://kid3.sourceforge.io/kid3_en.html) | Unified field mapping table |
| 8 | [Xiph VorbisComment](https://wiki.xiph.org/VorbisComment) | Vorbis comment specification |
| 9 | [dBpoweramp Forum](https://forum.dbpoweramp.com/) | DBpoweramp tag handling behavior |
