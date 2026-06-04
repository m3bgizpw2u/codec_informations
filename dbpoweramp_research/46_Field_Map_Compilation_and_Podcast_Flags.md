# Field Mapping: Compilation and Podcast Flags Across Formats

## 1. Compilation Flag Overview

### 1.1 Why Compilation Matters

Compilation albums require special handling:
- Multiple artists on one album
- Track should appear under each artist AND under "Various Artists"
- Affects library grouping and display
- Used by iTunes, MediaMonkey, MusicBee, etc.

### 1.2 Format-Specific Tags

| Format | Tag | Value | Notes |
|--------|-----|-------|-------|
| ID3v2 | `TCMP` | `1` | iTunes extension, not official ID3 |
| Vorbis | `COMPILATION` | `1` | Standard Vorbis field |
| APEv2 | `COMPILATION` | `1` | Standard APE field |
| MP4 | `cpil` | `true` (boolean) | Binary atom |
| ASF/WMA | `WM/IsCompilation` | `1` | Integer value |
| RIFF | (none) | — | No native support |

**Source:** [Picard Tag Mapping](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html)

---

## 2. ID3v2 TCMP Frame

### 2.1 iTunes Extension

**TCMP is NOT part of the official ID3v2 specification.**

**History:**
- Introduced by iTunes
- Adopted by other taggers
- Recognized by most music players

**Format:**
```
TCMP = 1  (present = compilation)
TCMP = 0  (or absent = not compilation)
```

**Source:** [Hydrogenaudio Forum](https://hydrogenaudio.org/index.php/topic,30752.0.html)

### 2.2 Detection in ID3

**Method 1: Frame presence**
```python
def is_compilation_id3(mp3_file) -> bool:
    """Check TCMP frame for compilation flag."""
    return 'TCMP' in mp3_file and str(mp3_file['TCMP'][0]) == '1'
```

**Method 2: Fallback to album artist**
```python
def infer_compilation(album_artist: str) -> bool:
    """Infer compilation from album artist."""
    if not album_artist:
        return True  # Assume compilation if no album artist
    va_patterns = ['various artists', 'various', 'va']
    return album_artist.lower().strip() in va_patterns
```

---

## 3. MP4 cpil Atom

### 3.1 Boolean Atom

**cpil atom type:** `boolean` (1 byte)

**Value:**
```
cpil = 1  (true = compilation)
cpil = 0  (false = not compilation)
```

**Source:** [Mutagen MP4 docs](https://mutagen.readthedocs.io/en/latest/api/mp4.html)

### 3.2 Python Handling

```python
from mutagen.mp4 import MP4

def read_compilation_mp4(mp4_file) -> bool:
    """Read compilation flag from MP4."""
    if 'cpil' in mp4_file:
        return bool(mp4_file['cpil'][0])
    return False

def write_compilation_mp4(mp4_file, is_compilation: bool) -> None:
    """Write compilation flag to MP4."""
    mp4_file['cpil'] = [is_compilation]
```

---

## 4. Vorbis COMPILATION Field

### 4.1 Standard Format

**Tag:** `COMPILATION`
**Value:** `1` (string)

**Multiple values:**
```
COMPILATION=1
```

**Note:** Only one value makes sense (binary flag)

### 4.2 Case Sensitivity

**Tools may use:**
- `COMPILATION`
- `compilation`
- `Compilation`

**Recommendation:** Normalize to uppercase `COMPILATION`

---

## 5. WMA WM/IsCompilation

### 5.1 Integer Value

**Tag:** `WM/IsCompilation`
**Value:** `1` or `0`

**Source:** [Picard Tag Mapping](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html)

---

## 6. Cross-Format Conversion

### 6.1 Conversion Table

| Source | Destination | Conversion |
|--------|-------------|------------|
| ID3 TCMP=1 | Vorbis | `COMPILATION=1` |
| ID3 TCMP=1 | MP4 | `cpil=true` |
| ID3 TCMP=1 | WMA | `WM/IsCompilation=1` |
| Vorbis COMPILATION=1 | ID3 | `TCMP=1` |
| Vorbis COMPILATION=1 | MP4 | `cpil=true` |
| MP4 cpil=true | ID3 | `TCMP=1` |
| MP4 cpil=true | Vorbis | `COMPILATION=1` |

### 6.2 Python Implementation

```python
class CompilationConverter:
    """Convert compilation flags between formats."""
    
    def read_compilation(self, tags: dict, format: str) -> bool:
        """Read compilation flag from format-specific tags."""
        if format == 'mp3':
            return tags.get('TCMP', [''])[0] == '1'
        elif format in ('flac', 'ogg'):
            return tags.get('COMPILATION', [''])[0] == '1'
        elif format == 'mp4':
            return bool(tags.get('cpil', [False])[0])
        elif format == 'wma':
            return tags.get('WM/IsCompilation', [0])[0] == 1
        return False
    
    def write_compilation(self, is_compilation: bool, format: str) -> dict:
        """Write compilation flag for format."""
        tags = {}
        
        if not is_compilation:
            return tags  # Don't write flag if not compilation
        
        if format == 'mp3':
            tags['TCMP'] = ['1']
        elif format in ('flac', 'ogg'):
            tags['COMPILATION'] = ['1']
        elif format == 'mp4':
            tags['cpil'] = [True]
        elif format == 'wma':
            tags['WM/IsCompilation'] = [1]
        
        return tags
```

---

## 7. Podcast Flags

### 7.1 iTunes Podcast Tags

| Tag | Type | Purpose |
|-----|------|---------|
| `pcst` | boolean | Podcast flag |
| `purl` | text | Podcast feed URL |
| `egid` | text | Episode GUID |
| `cat` | text | iTunes category |
| `tves` | integer | Episode number |
| `tvsh` | text | Show name |
| `tvsn` | integer | Season number |
| `tven` | text | Episode ID |

**Source:** [Picard Tag Mapping](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html)

### 7.2 Podcast Tag Mapping

| Internal | MP4 | ID3v2 | Vorbis |
|----------|-----|-------|--------|
| podcast | `pcst` | (none) | (none) |
| podcasturl | `purl` | (none) | (none) |
| podcastguid | `egid` | (none) | (none) |
| series | `tvsh` | (none) | (none) |
| season | `tvsn` | (none) | (none) |
| episode | `tves` | (none) | (none) |

### 7.3 iTunes-Specific MP4 Atoms

| Atom | Type | Description |
|------|------|-------------|
| `stik` | integer | Media type (1-10) |
| `----:com.apple.iTunes:ITUNESADVISORY` | integer | Explicit content |
| `----:com.apple.iTunes:ITUNESCOUNTRY` | text | Country code |
| `----:com.apple.iTunes:ITUNESCOUNTRY` | text | Rating |

**stik values:**
- 0: Movie
- 1: Normal music
- 2: Audiobook
- 5: Short film
- 6: TV show
- 9: Music video
- 10: Interactive booklet
- 14: Ringtone

**Source:** [Joel Verhagen blog](https://www.joelverhagen.com/blog/2010/12/how-itunes-uses-id3-tags)

---

## 8. TV Show Fields (MP4)

### 8.1 TV Show Metadata

| Atom | Type | Example |
|------|------|---------|
| `tvsh` | text | "Breaking Bad" |
| `tvsn` | integer | 5 (season 5) |
| `tves` | integer | 8 (episode 8) |
| `tvnn` | text | "AMC" (network) |
| `tven` | text | "508" (episode ID) |

### 8.2 Content Type (cat)

**Purpose:** iTunes Store category

**Format:** Hierarchical with `/` separator
```
cat = "TV Shows/Drama"
cat = "Podcasts/Society & Culture"
```

### 8.3 Python Handling

```python
from mutagen.mp4 import MP4

def read_tv_show_metadata(mp4_file) -> dict:
    """Read TV show metadata from MP4."""
    return {
        'show': mp4_file.get('tvsh', [None])[0],
        'season': mp4_file.get('tvsn', [None])[0],
        'episode': mp4_file.get('tves', [None])[0],
        'network': mp4_file.get('tvnn', [None])[0],
        'episode_id': mp4_file.get('tven', [None])[0],
    }

def write_tv_show_metadata(mp4_file, show: str, season: int, 
                           episode: int, **kwargs) -> None:
    """Write TV show metadata to MP4."""
    mp4_file['tvsh'] = [show]
    mp4_file['tvsn'] = [season]
    mp4_file['tves'] = [episode]
    
    if 'network' in kwargs:
        mp4_file['tvnn'] = [kwargs['network']]
    if 'episode_id' in kwargs:
        mp4_file['tven'] = [kwargs['episode_id']]
```

---

## 9. Edge Cases

### 9.1 Compilation Without Flag

**Scenario:** Compilation album without TCMP/COMPILATION

**Detection heuristics:**
1. Multiple tracks with different artists
2. Album artist = "Various Artists"
3. Track numbers restart on each disc

### 9.2 Single-Artist Multi-CD

**Scenario:** Artist album with multiple discs

**Not a compilation:**
- Same artist for all tracks
- Album artist set correctly
- No TCMP flag needed

### 9.3 Podcast vs Music Track

**Scenario:** Music video with stik=9

**Issue:** Both podcast and music metadata may coexist

**Rule:** Treat stik as primary discriminator

### 9.4 Missing Podcast Fields

**Scenario:** Track marked as pcst=true but no purl

**Behavior:**
- Most players still recognize as podcast
- purl may be needed for podcast subscription

---

## 10. User Impact Assessment

| Scenario | User Impact | Severity |
|----------|-------------|----------|
| TCMP lost on FLAC→MP3 | iTunes thinks not compilation | **HIGH** |
| COMPILATION not written to MP4 | iTunes grouping wrong | **HIGH** |
| pcst not preserved | Not recognized as podcast | **HIGH** |
| TV show fields lost | No TV show info | Medium |

---

## 11. Validation Checklist

- [ ] Read TCMP=1 from ID3v2
- [ ] Write TCMP=1 to ID3v2
- [ ] Read COMPILATION=1 from Vorbis
- [ ] Write COMPILATION=1 to Vorbis
- [ ] Read cpil=true from MP4
- [ ] Write cpil=true to MP4 (boolean)
- [ ] Read WM/IsCompilation from WMA
- [ ] Handle absence of flag as False
- [ ] Convert TCMP→COMPILATION correctly
- [ ] Convert COMPILATION→cpil correctly

---

## 12. Source Citations

| # | Source | Key Information |
|---|--------|-----------------|
| 1 | [Picard Tag Mapping](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html) | Compilation and podcast mapping |
| 2 | [Hydrogenaudio Forum](https://hydrogenaudio.org/index.php/topic,30752.0.html) | TCMP frame discussion |
| 3 | [Joel Verhagen blog](https://www.joelverhagen.com/blog/2010/12/how-itunes-uses-id3-tags) | iTunes tag usage |
| 4 | [Mutagen MP4 docs](https://mutagen.readthedocs.io/en/latest/api/mp4.html) | cpil and TV show atoms |
| 5 | [Mp3tag mapping](https://docs.mp3tag.de/mapping-table/) | TCMP and pcst mapping |
