# Field Mapping: BPM, Key, and Rating Tags Across Formats

## 1. BPM (Beats Per Minute)

### 1.1 Format Comparison

| Format | Tag | Value Type | Range | Notes |
|--------|-----|------------|-------|-------|
| ID3v2 | `TBPM` | String (integer or decimal) | 0-999 | May include decimal |
| Vorbis | `BPM` | String (integer) | 0-999 | Integer |
| APEv2 | `BPM` | String (integer) | 0-999 | Integer |
| MP4 | `tmpo` | Integer | 0-255 | Limited range |
| WMA | `WM/BeatsPerMinute` | Integer | 0-999 | Integer |

**Source:** [Picard Tag Mapping](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html)

### 1.2 ID3v2 TBPM

**Format:** Numeric string
```
TBPM = "120"        # Integer
TBPM = "120.5"      # May include decimal
```

**Spec note:** "The BPM is an integer showing the number of beats per minute"

**Common practice:** Some tools store decimal BPM for tempo-synced tracks.

### 1.3 MP4 tmpo

**Format:** 16-bit unsigned integer
**Range:** 0-255

**Issue:** Cannot store BPM > 255

**Example:**
```python
from mutagen.mp4 import MP4

# Read
bpm = mp4_file['tmpo'][0]  # Integer 0-255

# Write
mp4_file['tmpo'] = [120]
```

**Source:** [Mutagen MP4 docs](https://mutagen.readthedocs.io/en/latest/api/mp4.html)

---

## 2. Musical Key

### 2.1 Format Comparison

| Format | Tag | Example Values | Notes |
|--------|-----|----------------|-------|
| ID3v2 | `TKEY` | "C", "Am", "Cmaj", "F#m" | 2-3 characters |
| Vorbis | `INITIALKEY` | "C", "Am", "F# minor" | Freeform text |
| Vorbis | `KEY` | Alternative | Shorter form |
| WMA | `WM/InitialKey` | "Am" | Same as ID3 |

**Source:** [Kid3 Handbook](https://kid3.sourceforge.io/kid3_en.html)

### 2.2 Key Notation

**Standard notation:**
- Major: C, C#, Db, D, D#, Eb, E, F, F#, Gb, G, G#, Ab, A, A#, Bb, B
- Minor: Cm, C#m, Dbm, Dm, D#m, Ebm, Em, Fm, F#m, Gbm, Gm, G#m, Abm, Am, A#m, Bbm, Bm

**Alternative notation:**
- Camelot: "8B", "11A"
- Open Key: "1o", "12d"

### 2.3 Key Extraction

**Automatic BPM and Key detection:**
- Beat tracking algorithms
- Chroma feature analysis
- Key detection from spectral content

**Tools:**
- Mixed In Key (commercial)
- Captain (free)
- Sonic Visualiser (free)

---

## 3. Rating Tags

### 3.1 ID3v2 POPM (Popularimeter)

**Frame structure:**
```
Frame ID: POPM
Email: user@example.com
Rating: 1-255
Counter: (optional) play count
```

**Rating scale:**
- 0 = Unknown/not rated
- 1 = Lowest
- 255 = Highest

**Common mappings:**
| Stars | POPM Value | Notes |
|-------|------------|-------|
| 1 | 1-63 | Lowest |
| 2 | 64-127 | |
| 3 | 128-191 | Middle |
| 4 | 192-223 | |
| 5 | 224-255 | Highest |

**Source:** [ID3v2 POPM spec](https://id3.org/id3v2.4.0-frames)

### 3.2 POPM Variations

**Email field uses:**
- Blank (Windows Media Player)
- "Windows Media Player 9 Series"
- "AMG All Music Guide"
- Custom identifier

**Multiple POPM frames:**
```
POPM:user1@email.com=196
POPM:user2@email.com=128
```

### 3.3 Vorbis RATING

**Tag:** `RATING` or `RATING[:@email]`

**Scale:** 0-100 (most implementations)

**Example:**
```
RATING=80
RATING:user@email.com=75
```

### 3.4 MP4 Rating

**Freeform:** `----:com.apple.iTunes:RATING`

**Scale:** 0-100

**Example:**
```
----:com.apple.iTunes:RATING=80
```

### 3.5 WMA Rating

**Tag:** `WM/SharedUserRating`

**Scale:** 0-99 (Windows Media Player)

**Source:** [bliss blog](https://www.blisshq.com/music-library-management-blog/2018/10/11/using-mp3-rating-tags/)

---

## 4. Rating Scale Conversion

### 4.1 Scale Mapping Table

| Stars | POPM (1-255) | Vorbis (0-100) | WMA (0-99) | Percentage |
|-------|--------------|----------------|------------|------------|
| 0 | 0 | 0 | 0 | 0% |
| 1 | 1-51 | 1-20 | 1-19 | 1-20% |
| 2 | 52-102 | 21-40 | 20-39 | 21-40% |
| 3 | 103-153 | 41-60 | 40-59 | 41-60% |
| 4 | 154-204 | 61-80 | 60-79 | 61-80% |
| 5 | 205-255 | 81-100 | 80-99 | 81-100% |

### 4.2 Conversion Formulas

```python
def stars_to_popm(stars: int) -> int:
    """Convert star rating (0-5) to POPM value (0-255)."""
    if stars == 0:
        return 0
    # Map 1-5 stars to 1-255
    return int((stars / 5.0) * 254) + 1

def popm_to_stars(popm: int) -> int:
    """Convert POPM value (0-255) to star rating (0-5)."""
    if popm == 0:
        return 0
    return int((popm / 255.0) * 5.0 + 0.5)

def stars_to_vorbis(stars: int) -> int:
    """Convert star rating (0-5) to Vorbis rating (0-100)."""
    if stars == 0:
        return 0
    return int((stars / 5.0) * 100)

def popm_to_vorbis(popm: int) -> int:
    """Convert POPM value (0-255) to Vorbis rating (0-100)."""
    if popm == 0:
        return 0
    return int((popm / 255.0) * 100)
```

---

## 5. APEv2 Rating

### 5.1 Tag Name

**Tag:** `RATING`

**Format:** String value

**Scale:** Various (0-100, 0-255, or 1-5)

**Source:** [Symfonium support forum](https://support.symfonium.app/t/rating-ape-tag-not-displayed/4419)

### 5.2 Implementation Note

**TagLib mapping:**
> "For mp3 it is a bug on the parser on my side with your files where it think there's id3 tags and so skip the ape, it's already fixed."

---

## 6. Cross-Format Rating Conversion

### 6.1 Python Implementation

```python
class RatingConverter:
    """Convert rating values between formats."""
    
    def __init__(self):
        self.scale_map = {
            'popm': (0, 255),      # 1-255 (0=unrated)
            'vorbis': (0, 100),    # 0-100
            'wma': (0, 99),        # 0-99
            'stars': (0, 5),       # 0-5
        }
    
    def normalize(self, value: int, from_scale: str) -> float:
        """Normalize rating to 0-1 float."""
        min_val, max_val = self.scale_map[from_scale]
        return (value - min_val) / (max_val - min_val)
    
    def convert(self, value: int, from_scale: str, to_scale: str) -> int:
        """Convert rating between scales."""
        if value == 0 and from_scale == 'popm':
            return 0  # 0 means unrated in POPM
        
        # Normalize to 0-1
        normalized = self.normalize(value, from_scale)
        
        # Convert to target scale
        min_val, max_val = self.scale_map[to_scale]
        result = int(normalized * (max_val - min_val) + min_val)
        
        # Clamp to valid range
        return max(min_val, min(max_val, result))
    
    def write_popm(self, rating: int, email: str = '') -> dict:
        """Create POPM frame data."""
        return {
            'POPM': {'email': email, 'rating': rating}
        }
    
    def write_vorbis_rating(self, rating: int) -> dict:
        """Create Vorbis rating tag."""
        return {'RATING': [str(rating)]}
    
    def write_mp4_rating(self, rating: int) -> dict:
        """Create MP4 rating tag."""
        return {'----:com.apple.iTunes:RATING': [str(rating)]}
```

---

## 7. Edge Cases

### 7.1 Rating of Zero

**POPM:** 0 means "unknown/not rated"
**Vorbis:** 0 may mean unrated or 0%
**WMA:** 0 means unrated

**Conversion:** Always preserve 0 as unrated

### 7.2 Multiple Rating Sources

**Scenario:** File has both POPM and Vorbis RATING

**Resolution:** Use most recent or most reliable source
- POPM: More widely supported
- RATING: Simpler format

### 7.3 BPM Over Range

**MP4 tmpo limit:** 0-255

**Workaround for high BPM:**
- Store in custom tag: `----:com.apple.iTunes:BPM`
- Convert to seconds: 60000/BPM for display

### 7.4 Decimal BPM

**ID3v2 TBPM:** May contain decimals

**Conversion:**
```
TBPM = "120.5"
↓
Vorbis BPM = "121" (rounded)
```

---

## 8. DBpoweramp Behavior

### 8.1 What We Know

- DBpoweramp preserves standard tags
- DSP effects available for tag manipulation
- Source: [dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/41137)

### 8.2 Unknowns

- Does DBpoweramp preserve POPM rating?
- Does DBpoweramp handle BPM decimal values?
- Does DBpoweramp preserve multiple rating sources?

### 8.3 User Impact

| Scenario | User Impact | Severity |
|----------|-------------|----------|
| POPM lost on conversion | Rating lost | Medium |
| Rating scale mismatch | Different star display | Medium |
| BPM > 255 in MP4 | Truncated to 255 | Low |
| Decimal BPM lost | Slight tempo inaccuracy | Low |

---

## 9. Code Examples

### 9.1 Reading Ratings

```python
def read_rating_from_mp3(mp3_file) -> tuple[int, str]:
    """
    Read rating from MP3 POPM frame.
    Returns (rating_0_255, email).
    """
    if 'POPM' in mp3_file:
        popm = mp3_file['POPM']
        if isinstance(popm, list):
            popm = popm[0]
        if hasattr(popm, 'rating'):
            return popm.rating, getattr(popm, 'email', '')
        elif isinstance(popm, dict):
            return popm.get('rating', 0), popm.get('email', '')
    return 0, ''

def read_bpm_from_mp4(mp4_file) -> int:
    """Read BPM from MP4 tmpo atom."""
    if 'tmpo' in mp4_file:
        return int(mp4_file['tmpo'][0])
    return 0
```

### 9.2 Writing Ratings

```python
from mutagen.id3 import POPM
from mutagen.mp4 import MP4

def write_rating_mp3(mp3_file, rating: int, email: str = '') -> None:
    """Write POPM rating to MP3."""
    mp3_file['POPM'] = POPM(
        encoding=3,  # UTF-8
        email=email,
        rating=rating,
        count=0
    )

def write_bpm_mp4(mp4_file, bpm: int) -> None:
    """Write BPM to MP4 tmpo atom (max 255)."""
    # Clamp to valid range
    bpm = max(0, min(255, int(bpm)))
    mp4_file['tmpo'] = [bpm]
```

---

## 10. Validation Checklist

- [ ] Read TBPM from ID3v2 (supports decimal)
- [ ] Write TBPM to ID3v2
- [ ] Read BPM from Vorbis
- [ ] Write BPM to Vorbis
- [ ] Read tmpo from MP4 (0-255 range)
- [ ] Write tmpo to MP4 (clamp to 0-255)
- [ ] Read TKEY from ID3v2
- [ ] Write INITIALKEY to Vorbis
- [ ] Read POPM rating (1-255 scale)
- [ ] Write POPM rating
- [ ] Convert rating between scales correctly
- [ ] Handle 0 as unrated in POPM
- [ ] Preserve email identifier in POPM

---

## 11. Source Citations

| # | Source | Key Information |
|---|--------|-----------------|
| 1 | [ID3v2.4 spec](https://id3.org/id3v2.4.0-frames) | TBPM, POPM specifications |
| 2 | [Picard Tag Mapping](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html) | BPM, rating mapping |
| 3 | [Mutagen MP4 docs](https://mutagen.readthedocs.io/en/latest/api/mp4.html) | tmpo atom |
| 4 | [Kid3 Handbook](https://kid3.sourceforge.io/kid3_en.html) | Key field mapping |
| 5 | [bliss blog](https://www.blisshq.com/music-library-management-blog/2018/10/11/using-mp3-rating-tags/) | Rating tag comparison |
| 6 | [Beaglebuddy POPM](http://www.beaglebuddy.com/content/pages/javadocs/com/beaglebuddy/id3/v23/frame_body/ID3v23FrameBodyPopularimeter.html) | POPM details |
