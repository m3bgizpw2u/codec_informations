# Field Mapping: Encoder and Encoder Settings Tags

## 1. Encoder Tag Overview

### 1.1 Purpose

The encoder tag identifies:
- Software/hardware used to encode the file
- Version of the encoder
- Encoding parameters

### 1.2 Why Encoder Tags Matter

**Use cases:**
- Quality auditing (verify lossy encoding settings)
- Identifying transcodes (file was re-encoded from source)
- Legal compliance (understanding audio origin)

### 1.3 The Core Question

**When converting, what should DBpoweramp write as the encoder tag?**

| Option | Description | Impact |
|--------|-------------|--------|
| A | Overwrite with "dBpoweramp Music Converter" | Loss of original info |
| B | Preserve original encoder | May be inaccurate for new file |
| C | Write actual new encoder ("LAME 3.100") | Most accurate |
| D | Chain ("LAME 3.100 converted by dBpoweramp") | Verbose but informative |

---

## 2. Format-Specific Encoder Tags

### 2.1 Complete Mapping

| Format | Tag | Standard | Notes |
|--------|-----|----------|-------|
| ID3v2.3 | `TENC` | Encoded by | Encoder used |
| ID3v2.3 | `TSSE` | Software/Hardware | Encoder settings |
| ID3v2.4 | `TENC` | Encoded by | Encoder used |
| ID3v2.4 | `TSSS` | Encoder settings | ID3v2.4 only |
| Vorbis | `ENCODER` | — | Encoder name |
| Vorbis | `ENCODERSETTINGS` | — | Encoder settings |
| APEv2 | `Encoder` | — | Encoder name |
| APEv2 | `Encoder Settings` | — | Encoder settings |
| MP4 | `©too` | — | Encoder software |
| RIFF | `ISFT` | Software | Encoder name |
| WMA | `WM/EncodedBy` | — | Encoder name |
| WMA | `WM/EncodingSettings` | — | Encoder settings |

**Source:** [Kid3 Handbook](https://kid3.sourceforge.io/kid3_en.html), [Microsoft WMA docs](https://learn.microsoft.com/en-us/windows/win32/wmformat/id3-tag-support)

### 2.2 ID3v2 TENC vs TSSE

**TENC (Encoded by):**
```
TENC = "LAME 3.100"
TENC = "dBpoweramp Music Converter"
TENC = "iTunes 12.12"
```

**TSSE (Software/Hardware and settings):**
```
TSSE = "LAME3.100"
TSSE = "CBR 320 kbps"
TSSE = "-V 0 --vbr-new"
```

**Note:** TSSE in ID3v2.3 is technically "Software/Hardware and settings", but commonly used for encoder identification.

### 2.3 ID3v2.4 TSSS

**Frame:** `TSSS` (Encoder Settings)

**Added in ID3v2.4:**
> "The 'Encoder settings' frame contains the settings used for encoding the audio file."

**Format:** Freeform text

---

## 3. Vorbis ENCODER and ENCODERSETTINGS

### 3.1 ENCODER

**Tag:** `ENCODER`
**Purpose:** Name of encoder software

**Examples:**
```
ENCODER = "LAME 3.100"
ENCODER = "FLAC 1.3.4"
ENCODER = "opusenc 0.19"
```

### 3.2 ENCODERSETTINGS

**Tag:** `ENCODERSETTINGS`
**Purpose:** Encoding parameters

**Examples:**
```
ENCODERSETTINGS = "-V 0"
ENCODERSETTINGS = "CBR 320"
ENCODERSETTINGS = "--quality 8"
```

### 3.3 MP4 ©too

**Atom:** `©too` (copyright note repurposed)
**Purpose:** Encoder software

**Examples:**
```
©too = "LAME 3.100"
©too = "iTunes 12.12"
```

### 3.4 Freeform MP4

**For encoder settings:**
```
----:com.apple.iTunes:ENCODERSETTINGS = "-V 0"
```

---

## 4. DBpoweramp Behavior (Research Findings)

### 4.1 What We Found

**From dBpoweramp forum:**
> "On conversion, dBpoweramp automatically creates the encodedby, encoder and encoder settings tags (if enabled in the settings)"

**Source:** [dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/41137-music-converter-add-id-tag-encoder-encoder-settings)

### 4.2 Configuration Options

**dBpoweramp Control Center → Music Converter → Configuration → Advanced Settings:**
- Option to disable "Update 'Source', 'Encoder', 'Encoded By' & 'Encoder Settings' ID tags"

**Source:** [dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/batch-converter/33132-flac-aiff-mp3-how-can-i-disable-the-writing-of-the-encoder-encoder-fields)

### 4.3 ID Tag Processing DSP

**Available action:** "Additions" → "Encoder"

**Effect:** Add encoder tag from DSP configuration

### 4.4 ID Tag Update Utility Codec

**Available action:** "Deletions" tab

**Use:** Remove encoder tags during conversion

---

## 5. Original Encoder Preservation

### 5.1 The Problem

**When converting:**
```
Source: FLAC file
Encoder: "FLAC 1.3.4"

Conversion: FLAC → MP3
Encoder: "LAME 3.100"
```

**Question:** Should the original FLAC encoder be preserved?

### 5.2 Approaches

**Option A: Overwrite (DBpoweramp default?)**
```
ENCODER = "dBpoweramp Music Converter"  # Written
ENCODER:original = "FLAC 1.3.4"  # Preserved?
```

**Option B: Chain**
```
ENCODER = "LAME 3.100 (converted from FLAC 1.3.4 by dBpoweramp)"
```

**Option C: Separate tags**
```
ENCODER = "LAME 3.100"
ENCODER:original = "FLAC 1.3.4"
```

### 5.3 Common Conventions

**FFmpeg:**
```
encoder = Lavf59.27.100
```

**LAME:**
```
encoder = "LAME3.100"
TSSE = "LAME3.100"
```

**iTunes:**
```
©too = "iTunes 12.12"
```

---

## 6. Encoder Settings Format

### 6.1 LAME Settings

**CBR:**
```
TSSE = "LAME3.100"
TSSE = "CBR 192"
TSSE = "CBR 320"
```

**VBR:**
```
TSSE = "LAME3.100"
TSSE = "-V 0"
TSSE = "-V 2 --vbr-new"
```

**ABR:**
```
TSSE = "LAME3.100"
TSSE = "-b 192"
```

### 6.2 FHG/AAC Settings

**Nero:**
```
TSSE = "Nero AAC 1.5.4.0"
```

**FDK:**
```
TSSE = "Lavc57.107.100"
```

### 6.3 ReplayGain Utility

**From dBpoweramp R4:**
> "EBU R128 used by default"
> "EBU R128 defaults to -18 LUFS"

**Source:** [Hydrogenaudio forum](https://hydrogenaudio.org/index.php/topic,112452.0.html)

---

## 7. Edge Cases

### 7.1 Multiple Encoder Tags

**Scenario:** File has been re-encoded multiple times

**Source tags:**
```
TENC = "Original Encoder"
TENC:1 = "Second Encoder"
TENC:2 = "Third Encoder"
```

**Issue:** Some formats don't support multiple encoder tags

### 7.2 Encoder vs Decoder

**Distinction:**
- **Encoder:** Used to CREATE the file
- **Decoder:** Used to PLAY/RENDER the file

**Note:** TSSE sometimes misused for decoder info

### 7.3 LAME Header vs Tags

**LAME stores info in:**
1. **Xing/LAME header** (in audio data)
2. **ID3 tags** (TENC, TSSE, etc.)

**Problem:** LAME header shows:
- Encoder version
- VBR info
- ReplayGain values (sometimes)

**Conflicts with tag info possible.**

### 7.4 Empty Encoder Tags

**Scenario:** Encoded by hand without tagging

**Handling:** Don't write encoder tag if unknown

---

## 8. Implementation

### 8.1 Python: Reading Encoder Tags

```python
def read_encoder_tags(tags: dict, format: str) -> dict:
    """Read encoder-related tags from format-specific tags."""
    result = {
        'encoder': None,
        'encoder_settings': None,
    }
    
    if format == 'mp3':
        result['encoder'] = tags.get('TENC', [None])[0]
        result['encoder_settings'] = tags.get('TSSE', [None])[0]
    elif format == 'flac':
        result['encoder'] = tags.get('ENCODER', [None])[0]
        result['encoder_settings'] = tags.get('ENCODERSETTINGS', [None])[0]
    elif format == 'mp4':
        result['encoder'] = tags.get('©too', [None])[0]
        # Check freeform for settings
        for key in tags:
            if 'ENCODERSETTINGS' in key:
                result['encoder_settings'] = tags[key][0]
                break
    
    return result
```

### 8.2 Python: Writing Encoder Tags

```python
def write_encoder_tags(tags: dict, format: str, encoder: str,
                      settings: str = None) -> dict:
    """Write encoder-related tags for format."""
    
    if format == 'mp3':
        tags['TENC'] = [encoder]
        if settings:
            tags['TSSE'] = [settings]
    elif format == 'flac':
        tags['ENCODER'] = [encoder]
        if settings:
            tags['ENCODERSETTINGS'] = [settings]
    elif format == 'mp4':
        tags['©too'] = [encoder]
        if settings:
            tags['----:com.apple.iTunes:ENCODERSETTINGS'] = [settings]
    
    return tags
```

### 8.3 DBpoweramp Configuration Check

```python
def get_dbpoweramp_encoder_behavior(config: dict) -> dict:
    """
    Determine DBpoweramp encoder tag behavior based on config.
    
    Returns dict with:
    - overwrite_encoder: bool
    - preserve_original: bool
    - chain_encoder: bool
    """
    # Check "Update 'Source', 'Encoder'..." setting
    update_encoder = config.get('update_encoder_on_conversion', True)
    
    if not update_encoder:
        return {
            'overwrite_encoder': False,
            'preserve_original': True,
            'chain_encoder': False,
        }
    
    # Default DBpoweramp behavior
    return {
        'overwrite_encoder': True,
        'preserve_original': False,
        'chain_encoder': False,
    }
```

---

## 9. User Impact Assessment

| Scenario | User Impact | Severity |
|----------|-------------|----------|
| Original encoder lost | Can't verify source quality | Medium |
| Wrong encoder written | File appears mislabeled | Medium |
| Encoder settings lost | Can't verify encoding params | Medium |
| Chain tag too long | May truncate in some players | Low |

---

## 10. Validation Checklist

- [ ] Read TENC from ID3v2
- [ ] Read TSSE from ID3v2
- [ ] Read ENCODER from Vorbis
- [ ] Read ©too from MP4
- [ ] Write TENC to ID3v2
- [ ] Write ENCODER to Vorbis
- [ ] Write ©too to MP4
- [ ] Preserve original encoder when configured
- [ ] Write actual encoder on conversion
- [ ] Handle missing encoder gracefully

---

## 11. Source Citations

| # | Source | Key Information |
|---|--------|-----------------|
| 1 | [ID3v2.4 spec](https://id3.org/id3v2.4.0-frames) | TENC, TSSE, TSSS specifications |
| 2 | [Kid3 Handbook](https://kid3.sourceforge.io/kid3_en.html) | Unified encoder mapping |
| 3 | [Microsoft WMA docs](https://learn.microsoft.com/en-us/windows/win32/wmformat/id3-tag-support) | WM/EncodedBy, WM/EncodingSettings |
| 4 | [Mutagen MP4 docs](https://mutagen.readthedocs.io/en/latest/api/mp4.html) | ©too atom |
| 5 | [dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/41137) | Encoder tag behavior |
| 6 | [Hydrogenaudio forum](https://hydrogenaudio.org/index.php/topic,112452.0.html) | R128 default settings |
