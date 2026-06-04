# Field Mapping: Lyrics, Comments, and Description Across Formats

## 1. Lyrics Field Mapping

### 1.1 Format Comparison

| Format | Tag Name | Type | Language Support | Synchronized |
|--------|----------|------|------------------|--------------|
| ID3v2 | `USLT` | Unsynchronized lyrics | Yes (3-char code) | No |
| ID3v2 | `SYLT` | Synchronized lyrics | Yes | Yes (timestamps) |
| Vorbis | `LYRICS` | Unsynchronized lyrics | No | No (LRC in text) |
| MP4 | `©lyr` | Unsynchronized lyrics | No | No |
| WMA | `WM/Lyrics` | Unsynchronized lyrics | No | No |
| APEv2 | `LYRICS` | Unsynchronized lyrics | No | No |

**Source:** [Zeugma440/atldotnet wiki](https://github.com/Zeugma440/atldotnet/wiki/Focus-on-lyrics-metadata)

### 1.2 ID3v2 USLT Frame

**Structure:**
```
Frame ID: USLT
Encoding: $xx (text encoding byte)
Language: $xx $xx $xx (ISO 639-2)
Descriptor: <text string> $00
Lyrics: <full text>
```

**Example:**
```
USLT (English, no descriptor):
Encoding: UTF-8
Language: eng
Descriptor: (empty)
Lyrics: [Full lyrics text here]
```

**Multiple lyrics frames:**
- One per language
- Descriptor distinguishes content
- Source: [ID3v2 spec](https://id3.org/id3v2.4.0-frames)

### 1.3 ID3v2 SYLT Frame

**Purpose:** Synchronized lyrics (karaoke)

**Structure:**
```
Frame ID: SYLT
Encoding: $xx
Language: $xx $xx $xx
Time stamp format: $02 (milliseconds)
Content type: $01 (lyrics)
Descriptor: <text string> $00
Lyrics: timestamp + text pairs
```

**Time stamp format:**
- Format $02: Milliseconds since beginning
- Format $01: MPEG frames

**Example SYLT content:**
```
[00:15.000]First line of lyrics
[00:22.500]Second line here
[00:30.000]And the third line
```

**Source:** [ID3v2.2.00 spec](https://web.archive.org/web/20161124001116/http:/id3.org/id3v2-00)

---

## 2. LRC Format (Text-Based Synchronized Lyrics)

### 2.1 LRC Standard Format

**Format:** `[mm:ss.xx]Lyrics text`

**Examples:**
```
[00:15.00]First line of the song
[00:22.50]This is the second line
[01:30.00]Chorus begins here
```

**Extensions:**
- `A2` extension: Word-level timestamps
- `Karaoke` tag format

### 2.2 Embedding LRC in USLT

**Method:** Paste LRC text directly into USLT frame

**Pros:**
- Works with existing tools
- Portable

**Cons:**
- Not native SYLT format
- Players may not parse correctly

**Source:** [mp3apps.net](https://mp3apps.net/how-to-add-lyrics-to-mp3-files-the-right-way/)

---

## 3. Comment Field Mapping

### 3.1 ID3v2 COMM Frame

**Structure:**
```
Frame ID: COMM
Encoding: $xx
Language: $xx $xx $xx
Descriptor: <text string> $00
Text: <full comment text>
```

**Descriptor uses:**
- Language-specific descriptions
- " eng Description text"
- Empty descriptor for general comment

**Multiple COMM frames:**
- One per description/language
- Descriptor distinguishes content

**Example:**
```
COMM (Description: "eng", text: "Comment text here")
```

**Source:** [ID3v2.4 spec](https://id3.org/id3v2.4.0-frames)

### 3.2 Vorbis COMMENT

**Format:** `COMMENT=Comment text`

**Multiple values:**
```
COMMENT=First comment
COMMENT=Second comment
```
OR
```
COMMENT=First comment; Second comment
```

**Convention:** Many tools use first value only.

### 3.3 Cross-Format Comment Mapping

| Format | Tag | Notes |
|--------|-----|-------|
| ID3v2 | `COMM:Description` | Descriptor separates comments |
| Vorbis | `COMMENT` | Single value or semicolon list |
| MP4 | `©cmt` | Single value |
| WMA | `Description` | Single value |

**Source:** [Kid3 Handbook](https://kid3.sourceforge.io/kid3_en.html)

---

## 4. Description vs Content Distinction

### 4.1 ID3v2

**COMM frame has explicit descriptor:**
```
COMM:eng,iTunes
Text: "Recorded at Abbey Road Studios"
```

**Common descriptors:**
- `eng` — English (general)
- `iTunes` — iTunes specific
- `MusicMatch` — MusicMatch
- `LYRICSBEGIN`/`LYRICSEND` — Lyrics markers

### 4.2 Vorbis/FLAC

**Description is part of value:**
```
COMMENT=This is the description:This is the content
```

**Common patterns:**
```
COMMENT= transcoder:made from FLAC
COMMENT= rip date:2024-01-15
```

### 4.3 Conversion Considerations

**When converting COMM to Vorbis:**
```
COMM:eng,Album="Dark Side of the Moon"
↓
COMMENT=Album: Dark Side of the Moon
```

**When converting Vorbis to COMM:**
```
COMMENT=Album: Dark Side of the Moon
↓
COMM:eng,Album=Dark Side of the Moon
```

---

## 5. Synchronized Lyrics Preservation

### 5.1 DBpoweramp Behavior (Unknown)

**Questions:**
- Does DBpoweramp preserve SYLT frames?
- Does DBpoweramp convert SYLT to LRC text?
- Does DBpoweramp preserve LRC text in USLT?

### 5.2 Conversion Matrix

| Source | Destination | Preserved? | Method |
|--------|-------------|------------|--------|
| SYLT | SYLT | Yes | Binary copy |
| SYLT | USLT+LRC | Partial | Extract and reformat |
| SYLT | Vorbis | No | Loss |
| LRC in USLT | SYLT | No | Loss (re-conversion imperfect) |
| LRC in USLT | LRC in USLT | Yes | Text preservation |
| LRC sidecar file | Any | Manual | Must copy separately |

### 5.3 Edge Cases

**Edge Case 1: Multiple languages**
```
USLT:eng=English lyrics here
USLT:spa=Letras en español aquí
```
→ Each language in separate frame

**Edge Case 2: Mixed synchronized and unsynchronized**
- Some files have both USLT and SYLT
- Converters should preserve both

**Edge Case 3: LRC with word-level timing**
```
[00:15.00]First [00:15.50]word [00:16.00]level
```
→ A2 extension, may not convert cleanly

---

## 6. Comment Truncation

### 6.1 Format-Specific Limits

| Format | Limit | Notes |
|--------|-------|-------|
| ID3v2 | 64 KB per frame | Theoretical |
| Vorbis | No strict limit | Practical memory limits |
| MP4 | ~2 GB | Unlimited |
| WMA | Limited | Format-specific |

### 6.2 Handling Long Comments

**Vorbis approach:** Split into multiple COMMENT fields

**ID3v2 approach:** Multiple COMM frames with different descriptors

**MP4 approach:** Single large ©cmt value

### 6.3 Python Example

```python
def split_comment_for_vorbis(comment: str, max_length: int = 8192) -> list:
    """Split long comments into multiple values."""
    if len(comment) <= max_length:
        return [comment]
    
    # Split by sentences or newlines
    parts = []
    current = ""
    
    for line in comment.split('\n'):
        if len(current) + len(line) + 1 <= max_length:
            current = current + "\n" + line if current else line
        else:
            if current:
                parts.append(current)
            current = line
    
    if current:
        parts.append(current)
    
    return parts

def merge_vorbis_comments(comments: list) -> str:
    """Merge multiple comment values."""
    return "\n".join(comments)
```

---

## 7. Edge Cases

### 7.1 Null Bytes in Lyrics

**Issue:** Some encoders insert null bytes

**Detection:**
```python
def clean_lyrics(text: str) -> str:
    """Remove null bytes and normalize line endings."""
    # Remove null bytes
    text = text.replace('\x00', '')
    # Normalize line endings
    text = text.replace('\r\n', '\n').replace('\r', '\n')
    return text.strip()
```

### 7.2 Encoding Issues

**Common problem:** UTF-8 in Latin-1 frame

**Solution:** Try multiple encodings
```python
def decode_text(data: bytes, encoding: str = 'utf-8') -> str:
    """Try to decode text with fallback."""
    for enc in ['utf-8', 'latin-1', 'utf-16']:
        try:
            return data.decode(enc).rstrip('\x00')
        except (UnicodeDecodeError, LookupError):
            continue
    return data.decode('utf-8', errors='replace')
```

### 7.3 Empty Descriptors

**ID3 COMM:** Empty descriptor = general comment
**Vorbis COMMENT:** Empty value = no comment

**Conversion:**
```
Empty descriptor COMM → Empty COMMENT value
```

### 7.4 Special Characters

**LRC format:** `[00:00.00]` bracket syntax
**USLT:** May contain timestamps, escape them

---

## 8. Code Examples

### 8.1 Reading Lyrics

```python
def read_lyrics_from_mp3(mp3_file) -> dict:
    """
    Read lyrics from MP3 ID3v2 tags.
    Returns dict with language as key.
    """
    lyrics = {}
    
    # Unsynchronized lyrics
    if 'USLT:eng' in mp3_file:
        lyrics['unsynced'] = {
            'language': 'eng',
            'text': str(mp3_file['USLT:eng'])
        }
    
    # Synchronized lyrics
    if 'SYLT:eng' in mp3_file:
        lyrics['synced'] = {
            'language': 'eng',
            'format': 'syllable' if mp3_file['SYLT:eng'].timestamp_format == 2 else 'frame',
            'text': format_sylt_as_lrc(mp3_file['SYLT:eng'])
        }
    
    return lyrics

def format_sylt_as_lrc(sylt_frame) -> str:
    """Convert SYLT frame to LRC format."""
    lines = []
    for timestamp, text in sylt_frame.text:
        # Format: [mm:ss.xx]
        minutes = timestamp // 60000
        seconds = (timestamp % 60000) // 1000
        centiseconds = (timestamp % 1000) // 10
        line = f"[{minutes:02d}:{seconds:02d}.{centiseconds:02d}]{text}"
        lines.append(line)
    return "\n".join(lines)
```

### 8.2 Reading Comments

```python
def read_comments_from_mp3(mp3_file) -> list:
    """
    Read all comments from MP3 ID3v2 COMM frames.
    """
    comments = []
    
    for key, value in mp3_file.items():
        if key.startswith('COMM'):
            # Parse descriptor from key
            if ':' in key:
                _, descriptor = key.split(':', 1)
            else:
                descriptor = ''
            
            comments.append({
                'descriptor': descriptor,
                'text': str(value)
            })
    
    return comments
```

---

## 9. DBpoweramp Behavior

### 9.1 What We Know

- DBpoweramp preserves standard ID3v2 tags during conversion
- DSP effects available for tag manipulation
- Source: [dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/41137-music-converter-add-id-tag-encoder-encoder-settings)

### 9.2 Questions

- Does DBpoweramp preserve SYLT (synchronized lyrics)?
- Does DBpoweramp handle multi-language USLT?
- Does DBpoweramp truncate long comments?

### 9.3 User Impact

| Scenario | User Impact | Severity |
|----------|-------------|----------|
| Synchronized lyrics lost | Karaoke broken | High |
| Multi-language lyrics lost | Missing translations | Medium |
| Long comment truncated | Partial info | Low |
| Encoding corruption | Garbled text | High |

---

## 10. Validation Checklist

- [ ] Read USLT from ID3v2 with language and descriptor
- [ ] Read SYLT and convert to LRC format
- [ ] Write USLT with correct encoding and language
- [ ] Preserve LRC format in USLT
- [ ] Read all COMM frames with descriptors
- [ ] Handle multiple comment values in Vorbis
- [ ] Clean null bytes from text
- [ ] Handle encoding errors gracefully
- [ ] Preserve line endings

---

## 11. Source Citations

| # | Source | Key Information |
|---|--------|-----------------|
| 1 | [ID3v2 USLT spec](https://id3.org/id3v2.4.0-frames) | USLT frame structure |
| 2 | [ID3v2.2.00 spec](https://web.archive.org/web/20161124001116/http:/id3.org/id3v2-00) | SYLT frame structure |
| 3 | [atldotnet wiki](https://github.com/Zeugma440/atldotnet/wiki/Focus-on-lyrics-metadata) | Lyrics format comparison |
| 4 | [mp3apps.net](https://mp3apps.net/how-to-add-lyrics-to-mp3-files-the-right-way/) | LRC format embedding |
| 5 | [Kid3 Handbook](https://kid3.sourceforge.io/kid3_en.html) | Comment field mapping |
| 6 | [cweiske lyrics](https://cweiske.de/tagebuch/embedded-lyrics.htm) | Lyrics in Vorbis |
