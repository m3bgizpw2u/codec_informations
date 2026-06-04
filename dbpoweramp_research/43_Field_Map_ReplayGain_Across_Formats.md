# Field Mapping: ReplayGain Across Formats (CRITICAL FOR AUDIOPHILES)

## 1. Overview: ReplayGain vs R128

### 1.1 Two Competing Standards

**ReplayGain 1.0 / 2.0:**
- Reference loudness: **-18 LUFS** (equivalent to 89 dB SPL)
- Used by: MP3, FLAC, Ogg Vorbis, AAC (MP4)
- Tag names: `REPLAYGAIN_TRACK_GAIN`, `REPLAYGAIN_TRACK_PEAK`, etc.

**EBU R128:**
- Reference loudness: **-23 LUFS** (broadcast standard)
- Used by: Opus, some modern streaming services
- Tag names: `R128_TRACK_GAIN`, `R128_ALBUM_GAIN`

### 1.2 Critical Difference

```
ReplayGain value ≈ R128 value - 5 dB
```

**Example:**
- Track at -23 LUFS
  - ReplayGain: -5.00 dB (relative to -18 LUFS)
  - R128: 0 (relative to -23 LUFS)

**Source:** [Hydrogenaudio ReplayGain spec](https://wiki.hydrogenaudio.org/index.php/ReplayGain_1.0_specification), [RFC 7845](https://tools.ietf.org/html/rfc7845)

---

## 2. ReplayGain Tag Names by Format

### 2.1 Complete Tag Mapping Table

| Format | Track Gain | Album Gain | Track Peak | Album Peak | Reference |
|--------|------------|------------|------------|------------|-----------|
| ID3v2 | `TXXX:REPLAYGAIN_TRACK_GAIN` | `TXXX:REPLAYGAIN_ALBUM_GAIN` | `TXXX:REPLAYGAIN_TRACK_PEAK` | `TXXX:REPLAYGAIN_ALBUM_PEAK` | `TXXX:REPLAYGAIN_REFERENCE_LOUDNESS` |
| Vorbis | `REPLAYGAIN_TRACK_GAIN` | `REPLAYGAIN_ALBUM_GAIN` | `REPLAYGAIN_TRACK_PEAK` | `REPLAYGAIN_ALBUM_PEAK` | `REPLAYGAIN_REFERENCE_LOUDNESS` |
| APEv2 | `REPLAYGAIN_TRACK_GAIN` | `REPLAYGAIN_ALBUM_GAIN` | `REPLAYGAIN_TRACK_PEAK` | `REPLAYGAIN_ALBUM_PEAK` | `REPLAYGAIN_REFERENCE_LOUDNESS` |
| MP4 | `----:com.apple.iTunes:REPLAYGAIN_TRACK_GAIN` | `----:com.apple.iTunes:REPLAYGAIN_ALBUM_GAIN` | `----:com.apple.iTunes:REPLAYGAIN_TRACK_PEAK` | `----:com.apple.iTunes:REPLAYGAIN_ALBUM_PEAK` | (rare) |
| ASF/WMA | (not standard) | (not standard) | (not standard) | (not standard) | (none) |
| **Opus** | `R128_TRACK_GAIN` | `R128_ALBUM_GAIN` | (none) | (none) | `R128_REFERENCE_LOUDNESS` |

**Source:** [Hydrogenaudio ReplayGain specification](https://wiki.hydrogenaudio.org/index.php/ReplayGain_1.0_specification), [Moonbase59/loudgain](https://github.com/Moonbase59/loudgain/)

### 2.2 Alternate Tag Names (Read Compatibility)

**Many tools use lowercase or variant names:**

| Canonical | Variant 1 | Variant 2 |
|-----------|-----------|-----------|
| `REPLAYGAIN_TRACK_GAIN` | `replaygain_track_gain` | `REPLAYGAIN_TRACKGAIN` |
| `REPLAYGAIN_TRACK_PEAK` | `replaygain_track_peak` | `REPLAYGAIN_TRACKPEAK` |

**Source:** [Hydrogenaudio wiki](https://wiki.hydrogenaudio.org/index.php/ReplayGain_2.0_specification) — "While the specification defines keys in uppercase, many implementations use lowercase variants"

---

## 3. Value Format Specifications

### 3.1 Gain Values

**ReplayGain Format (MP3, FLAC, Vorbis):**
```
REPLAYGAIN_TRACK_GAIN = "-6.20 dB"
```
- Prefix: `-` for negative (attenuation), no prefix for positive
- Decimal places: **exactly 2** (a.bb)
- Suffix: **space + "dB"**
- Range: approximately -50 dB to +50 dB

**Source:** [Hydrogenaudio ReplayGain 1.0 spec](https://wiki.hydrogenaudio.org/index.php/ReplayGain_1.0_specification) — Table 3

### 3.2 Peak Values

**Format:**
```
REPLAYGAIN_TRACK_PEAK = "1.218222"
```
- **6 decimal places** (c.dddddd)
- Normalized to 1.0 = maximum sample level
- Range: 0.0 to ~2.0 (can exceed 1.0 before clipping)

**Source:** [Hydrogenaudio ReplayGain 1.0 spec](https://wiki.hydrogenaudio.org/index.php/ReplayGain_1.0_specification)

### 3.3 True Peak (ReplayGain 2.0)

**ReplayGain 2.0 adds True Peak measurement:**
```
REPLAYGAIN_TRACK_PEAK = "1.21822226"
```
- **8 decimal places** for true peak (oversampled)
- Measured using ITU-R BS.1770-4 True Peak algorithm

**Source:** [ReplayGain 2.0 spec](https://wiki.hydrogenaudio.org/index.php?title=ReplayGain_2.0_specification)

---

## 4. Opus R128: Q7.8 Fixed-Point Format (CRITICAL)

### 4.1 Why R128 is Different

Opus uses **R128 tags with a completely different format**:
- **NOT a dB string** like ReplayGain
- **Q7.8 fixed-point integer** — the raw 16-bit value
- Reference: -23 LUFS (not -18 LUFS)

**Source:** [RFC 7845 Ogg Opus spec](https://tools.ietf.org/html/rfc7845)

### 4.2 R128 Tag Names for Opus

| Tag | Purpose | Example Value |
|-----|---------|---------------|
| `R128_TRACK_GAIN` | Track gain adjustment | `-573` |
| `R128_ALBUM_GAIN` | Album gain adjustment | `-512` |
| `R128_REFERENCE_LOUDNESS` | Reference loudness (optional) | `-23 dB` |

**Source:** [RFC 7845 Section 5.2](https://tools.ietf.org/html/rfc7845#page-25)

### 4.3 Q7.8 Fixed-Point Format Explained

**Q7.8 means:**
- 16-bit signed integer
- 7 bits: integer part (including sign)
- 8 bits: fractional part
- Conversion: `gain_dB = integer_value / 256.0`

**Examples:**

| Integer Value | Gain (dB) | Calculation |
|--------------|-----------|-------------|
| `-512` | -2.00 dB | -512 / 256 |
| `-1280` | -5.00 dB | -1280 / 256 |
| `+256` | +1.00 dB | 256 / 256 |

**Source:** [RFC 7845](https://tools.ietf.org/html/rfc7845) — "Q7.8 fixed-point number in dB"

### 4.4 Converting Between ReplayGain and R128

**Formula for ReplayGain → R128:**
```python
def replaygain_to_r128(replaygain_db: float) -> int:
    """
    Convert ReplayGain dB value to R128 Q7.8 integer.
    Formula: R128 = round((RG_value + 5.0) * 256)
    
    The +5 dB offset accounts for the difference between
    -18 LUFS (ReplayGain) and -23 LUFS (R128) reference levels.
    """
    return round((replaygain_db + 5.0) * 256.0)

def r128_to_replaygain(r128_int: int) -> float:
    """
    Convert R128 Q7.8 integer to ReplayGain dB value.
    Formula: RG_dB = (R128_value / 256.0) - 5.0
    """
    return (r128_int / 256.0) - 5.0
```

**Conversion Examples:**

| ReplayGain (dB) | R128 Integer | R128 (dB) |
|-----------------|--------------|-----------|
| -5.00 | -0 | 0.00 |
| -6.00 | -256 | -1.00 |
| -10.00 | -1280 | -5.00 |

### 4.5 R128 Peak Values

**Opus does NOT use peak tags.**

Per RFC 7845 and loudgain documentation:
> "Peak normalization tags should NOT be used with Opus files"

The Opus decoder's true peak is handled differently.

**Source:** [Moonbase59/loudgain](https://github.com/Moonbase59/loudgain/) — "So we don't store any 'Peak' type tags"

---

## 5. Reference Loudness Tags

### 5.1 ReplayGain Reference

```
REPLAYGAIN_REFERENCE_LOUDNESS = "-18 dB"
```
- Optional tag indicating reference level
- Default: -18 dB if not specified
- Some scanners write this explicitly

### 5.2 R128 Reference

```
R128_REFERENCE_LOUDNESS = "-23 dB"
```
- Required for R128 but often omitted
- Default: -23 LUFS if not specified

### 5.3 Loudness Range Tags (ReplayGain 2.0)

```
REPLAYGAIN_ALBUM_RANGE = "2.30 dB"
REPLAYGAIN_TRACK_RANGE = "3.50 dB"
```
- Measured in dB (not LUFS for Range)
- Range = difference between loud and quiet parts

**Source:** [ReplayGain 2.0 spec](https://wiki.hydrogenaudio.org/index.php?title=ReplayGain_2.0_specification)

---

## 6. Cross-Format Conversion

### 6.1 Conversion Matrix

| Source | Destination | Action Required |
|--------|-------------|-----------------|
| FLAC (RG) | MP3 (RG) | Copy tags as-is |
| FLAC (RG) | Opus | Convert to R128: `round((val + 5) * 256)` |
| Opus (R128) | MP3 (RG) | Convert to RG: `val / 256 - 5` |
| MP3 (RG) | FLAC (RG) | Copy tags as-is |
| Vorbis (RG) | Opus | Convert to R128 |
| Opus (R128) | Vorbis | Convert to RG |

### 6.2 Python Implementation

```python
class ReplayGainConverter:
    """Convert ReplayGain values between formats."""
    
    R128_OFFSET = 5.0  # dB difference between -18 and -23 LUFS
    
    def __init__(self, track_gain_db: float | None = None,
                 track_peak: float | None = None,
                 album_gain_db: float | None = None,
                 album_peak: float | None = None):
        self.track_gain_db = track_gain_db
        self.track_peak = track_peak
        self.album_gain_db = album_gain_db
        self.album_peak = album_peak
    
    def format_replaygain_gain(self, gain_db: float | None) -> str | None:
        """Format gain value for ReplayGain tags."""
        if gain_db is None:
            return None
        # Format: "-6.20 dB" (2 decimal places)
        return f"{gain_db:+.2f} dB".replace('+', '') if gain_db < 0 else f"+{gain_db:.2f} dB"
    
    def replaygain_to_r128(self, gain_db: float) -> int:
        """Convert ReplayGain dB to R128 Q7.8 integer."""
        return round((gain_db + self.R128_OFFSET) * 256.0)
    
    def r128_to_replaygain(self, r128_int: int) -> float:
        """Convert R128 Q7.8 integer to ReplayGain dB."""
        return (r128_int / 256.0) - self.R128_OFFSET
    
    def to_vorbis_tags(self) -> dict:
        """Export as Vorbis/FLAC tags."""
        tags = {}
        if self.track_gain_db is not None:
            tags['REPLAYGAIN_TRACK_GAIN'] = self.format_replaygain_gain(self.track_gain_db)
        if self.track_peak is not None:
            tags['REPLAYGAIN_TRACK_PEAK'] = f"{self.track_peak:.6f}"
        if self.album_gain_db is not None:
            tags['REPLAYGAIN_ALBUM_GAIN'] = self.format_replaygain_gain(self.album_gain_db)
        if self.album_peak is not None:
            tags['REPLAYGAIN_ALBUM_PEAK'] = f"{self.album_peak:.6f}"
        return tags
    
    def to_r128_tags(self) -> dict:
        """Export as Opus R128 tags."""
        tags = {}
        if self.track_gain_db is not None:
            tags['R128_TRACK_GAIN'] = str(self.replaygain_to_r128(self.track_gain_db))
        if self.album_gain_db is not None:
            tags['R128_ALBUM_GAIN'] = str(self.replaygain_to_r128(self.album_gain_db))
        # No peak tags for Opus
        return tags
    
    def to_mp4_tags(self) -> dict:
        """Export as MP4 freeform tags."""
        tags = {}
        prefix = '----:com.apple.iTunes:'
        if self.track_gain_db is not None:
            tags[f'{prefix}REPLAYGAIN_TRACK_GAIN'] = self.format_replaygain_gain(self.track_gain_db)
        if self.track_peak is not None:
            tags[f'{prefix}REPLAYGAIN_TRACK_PEAK'] = f"{self.track_peak:.6f}"
        if self.album_gain_db is not None:
            tags[f'{prefix}REPLAYGAIN_ALBUM_GAIN'] = self.format_replaygain_gain(self.album_gain_db)
        if self.album_peak is not None:
            tags[f'{prefix}REPLAYGAIN_ALBUM_PEAK'] = f"{self.album_peak:.6f}"
        return tags
```

### 6.3 R128 Value Ranges

| R128 Integer | Gain (dB) | Notes |
|--------------|-----------|-------|
| -32768 | -127.00 dB | Minimum |
| -5120 | -20.00 dB | Near reference |
| -1280 | -5.00 dB | Common value |
| 0 | 0.00 dB | At -23 LUFS |
| 1280 | +5.00 dB | Boost needed |
| 32767 | +127.00 dB | Maximum |

---

## 7. DBpoweramp R128 Behavior

### 7.1 What We Know

**From forum research:**
> "EBU R128 used by default", "EBU R128 defaults to -18 LUFS"

**Source:** [Hydrogenaudio forum - foobar2000 R128 discussion](https://hydrogenaudio.org/index.php/topic,112452.0.html)

### 7.2 Questions About DBpoweramp

- Does DBpoweramp write R128 tags for Opus files?
- Does DBpoweramp convert ReplayGain to R128 when converting FLAC → Opus?
- Does DBpoweramp preserve the dB suffix in ReplayGain tags?

### 7.3 User Impact Assessment

| Scenario | User Impact | Severity |
|----------|-------------|----------|
| RG→Opus without conversion | Tracks play 5 dB too quiet | **CRITICAL** |
| RG peak ignored in Opus | Clipping possible | **HIGH** |
| Wrong reference loudness | Incorrect normalization | High |
| R128→FLAC without conversion | Tracks play 5 dB too loud | **CRITICAL** |

---

## 8. Edge Cases

### 8.1 Opus Header Gain vs Tags

**Opus has TWO gain mechanisms:**

1. **Header gain field** in OpusHead
   - Applied by decoder before output
   - Format: Q7.8 integer (same as R128 tags)

2. **R128 tag comments**
   - Suggested gain for players
   - Per RFC 7845, players should prefer tag over header

**Critical issue:** Some players use header gain, some use tags, some combine them incorrectly.

**Source:** [Hydrogenaudio - ReplayGain writes bad Opus Header Gain](https://hydrogenaudio.org/index.php/topic,120381.0.html)

### 8.2 Multiple ReplayGain Entries

**Some files have duplicate tags:**
```
REPLAYGAIN_TRACK_GAIN = "-6.20 dB"
REPLAYGAIN_TRACK_GAIN = "-5.50 dB"  # Duplicate
```

**Rule:** Use the first value or the most reasonable one.

### 8.3 Opus Without R128 Tags

**If no R128 tags present:**
- Assume track is at reference level (-23 LUFS)
- No gain adjustment needed
- Header gain may have been set by encoder

### 8.4 Negative Gain Values Without dB Suffix

**Invalid but sometimes seen:**
```
REPLAYGAIN_TRACK_GAIN = "-6.20"
REPLAYGAIN_TRACK_GAIN = "-6.20dB"  # Missing space
```

**Validation:** Reject values not matching `[-+]?\d+\.\d{2} dB`

### 8.5 Peak Values Greater Than 1.0

**Legitimate for pre-clipped material:**
```
REPLAYGAIN_TRACK_PEAK = "1.452123"
```

**Note:** Can exceed 1.0; indicates samples would clip if boosted.

---

## 9. Complete Code Example

### 9.1 Full Parser

```python
import re
from dataclasses import dataclass
from typing import Optional

@dataclass
class ReplayGainData:
    """Parsed ReplayGain/R128 data."""
    track_gain_db: Optional[float] = None
    track_peak: Optional[float] = None
    album_gain_db: Optional[float] = None
    album_peak: Optional[float] = None
    reference_loudness: Optional[float] = None  # -18 or -23 LUFS
    is_r128: bool = False  # True if Opus R128 format

class ReplayGainParser:
    """Parse ReplayGain and R128 tags from various formats."""
    
    RG_GAIN_PATTERN = re.compile(r'^([-+]?\d+\.\d{2})\s*dB$')
    RG_PEAK_PATTERN = re.compile(r'^(\d+\.\d{6,})$')
    R128_INT_PATTERN = re.compile(r'^([-]?\d+)$')
    R128_OFFSET = 5.0  # dB between -18 and -23 LUFS
    
    def parse_gain(self, value: str, is_r128: bool = False) -> Optional[float]:
        """Parse gain value, returning dB."""
        if not value:
            return None
        
        value = value.strip()
        
        if is_r128:
            # R128 format: integer
            match = self.R128_INT_PATTERN.match(value)
            if match:
                return (int(match.group(1)) / 256.0) - self.R128_OFFSET
        else:
            # ReplayGain format: "-6.20 dB"
            match = self.RG_GAIN_PATTERN.match(value)
            if match:
                return float(match.group(1))
        
        return None
    
    def parse_peak(self, value: str) -> Optional[float]:
        """Parse peak value."""
        if not value:
            return None
        
        match = self.RG_PEAK_PATTERN.match(value.strip())
        if match:
            return float(match.group(1))
        
        # Try simple float
        try:
            return float(value)
        except ValueError:
            return None
    
    def parse_r128_gain(self, value: str) -> Optional[float]:
        """Parse R128 integer and convert to ReplayGain dB."""
        try:
            r128_int = int(value.strip())
            return (r128_int / 256.0) - self.R128_OFFSET
        except ValueError:
            return None
    
    def format_r128_gain(self, gain_db: float) -> str:
        """Format ReplayGain dB as R128 integer."""
        r128_int = round((gain_db + self.R128_OFFSET) * 256.0)
        return str(max(-32768, min(32767, r128_int)))
    
    def format_replaygain_gain(self, gain_db: float) -> str:
        """Format gain as ReplayGain dB string."""
        return f"{gain_db:+.2f} dB".replace('+', '') if gain_db < 0 else f"{gain_db:.2f} dB"
```

### 9.2 Format-Specific Readers

```python
def read_flac_replaygain(flac_file) -> ReplayGainData:
    """Read ReplayGain tags from FLAC file."""
    data = ReplayGainData(is_r128=False)
    
    if 'REPLAYGAIN_TRACK_GAIN' in flac_file:
        data.track_gain_db = parse_gain(flac_file['REPLAYGAIN_TRACK_GAIN'][0])
    if 'REPLAYGAIN_TRACK_PEAK' in flac_file:
        data.track_peak = parse_peak(flac_file['REPLAYGAIN_TRACK_PEAK'][0])
    if 'REPLAYGAIN_ALBUM_GAIN' in flac_file:
        data.album_gain_db = parse_gain(flac_file['REPLAYGAIN_ALBUM_GAIN'][0])
    if 'REPLAYGAIN_ALBUM_PEAK' in flac_file:
        data.album_peak = parse_peak(flac_file['REPLAYGAIN_ALBUM_PEAK'][0])
    
    return data

def read_opus_replaygain(opus_file) -> ReplayGainData:
    """Read R128 tags from Opus file."""
    data = ReplayGainData(is_r128=True)
    
    if 'R128_TRACK_GAIN' in opus_file:
        data.track_gain_db = parse_r128_gain(opus_file['R128_TRACK_GAIN'][0])
    if 'R128_ALBUM_GAIN' in opus_file:
        data.album_gain_db = parse_r128_gain(opus_file['R128_ALBUM_GAIN'][0])
    
    return data
```

---

## 10. Validation Checklist

- [ ] Parse ReplayGain "-6.20 dB" correctly
- [ ] Parse ReplayGain "+2.50 dB" (positive) correctly
- [ ] Parse ReplayGain peak "1.218222" correctly
- [ ] Parse R128 integer "-512" correctly
- [ ] Convert ReplayGain → R128: -5.00 dB → "-0"
- [ ] Convert R128 → ReplayGain: -1280 → -10.00 dB
- [ ] Format R128 integer without decimal places
- [ ] Format ReplayGain with exactly 2 decimal places and " dB" suffix
- [ ] Handle missing/empty tags gracefully
- [ ] Detect Opus files and use R128 format
- [ ] No peak tags written for Opus files

---

## 11. Source Citations

| # | Source | Key Information |
|---|--------|-----------------|
| 1 | [ReplayGain 1.0 spec](https://wiki.hydrogenaudio.org/index.php/ReplayGain_1.0_specification) | Original ReplayGain format |
| 2 | [ReplayGain 2.0 spec](https://wiki.hydrogenaudio.org/index.php?title=ReplayGain_2.0_specification) | Revised specification |
| 3 | [RFC 7845 Ogg Opus](https://tools.ietf.org/html/rfc7845) | R128 tags for Opus |
| 4 | [Moonbase59/loudgain](https://github.com/Moonbase59/loudgain/) | R128 and ReplayGain implementation |
| 5 | [Hydrogenaudio Forum](https://hydrogenaudio.org/index.php/topic,112452.0.html) | R128 vs ReplayGain discussion |
| 6 | [Hydrogenaudio Forum](https://hydrogenaudio.org/index.php/topic,120381.0.html) | Opus header gain issues |
| 7 | [Xiph VorbisComment](https://wiki.xiph.org/VorbisComment) | Vorbis ReplayGain tags |
| 8 | [Picard Tag Mapping](https://picard-docs.musicbrainz.org/en/v3.0/appendices/tag_mapping.html) | Cross-format tag mapping |
