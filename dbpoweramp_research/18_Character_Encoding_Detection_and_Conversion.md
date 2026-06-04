# 18_Character_Encoding_Detection_and_Conversion.md

> **Research Area:** DBpoweramp Music Converter — Character Encoding Detection and Conversion
> **Covers:** ID3v2 encoding byte handling, ID3v1 encoding assumptions, UTF-16 BOM, UTF-8 heuristics, Unicode normalization
> **Confidence Level:** Medium-High (standards-based + cross-referenced TagLib/Mutagen/DBpoweramp behavior)
> **Cross-references:** File 15 (Preservation Rules), File 16 (Conflict Resolution)

---

## 1. Overview

Character encoding is the most common source of metadata corruption in audio conversion. Files created by different tools in different eras use different encodings, and DBpoweramp must navigate this landscape while producing consistent, interoperable output. This document covers encoding detection, conversion decisions, BOM handling, and Unicode normalization.

---

## 2. The Encoding Problem in Audio Metadata

### 2.1 Why Encoding Matters

Audio metadata files contain text in multiple overlapping standards:

| Standard | Encoding Options | Problem |
|---|---|---|
| ID3v1 | No encoding byte (always assumed ISO-8859-1) | Windows-1252, Shift-JIS, GB18030 often used |
| ID3v2 | $00 ISO-8859-1, $01 UTF-16 BOM, $02 UTF-16BE, $03 UTF-8 | Mismatched bytes vs. declared encoding |
| Vorbis Comment | Assumed UTF-8 | Latin-1 bytes written as UTF-8 |
| MP4 | Assumed UTF-8 for text atoms | Non-UTF-8 bytes may corrupt |

**Root cause:** The 1990s-era ID3 standard was designed for Western European text. When software wrote non-ISO-8859-1 bytes but declared ISO-8859-1, the mismatch produces Mojibake (garbled characters).

### 2.2 DBpoweramp's Encoding Philosophy

DBpoweramp's official stance:
- **Writes:** UTF-8 or UTF-16 (configurable) for all modern formats
- **Reads:** Attempts to handle legacy encodings, but with limitations
- **UTF-8 BOM:** DBpoweramp does NOT add BOM to UTF-8 (correct per standard)

**Source:** [DBpoweramp Forum — UTF-8 with BOM](https://forum.dbpoweramp.com/forum/dbpoweramp/batch-converter/42002-utf-8-with-bom-for-id3-and-vorbis-tags) — "UTF-8 does not have BOM as it is stored as encoded 8 bit, not 16 bit (which is UTF-16)."

---

## 3. ID3v2 Frame Encoding Byte Handling

### 3.1 The Four Encoding Bytes

| Byte | Name | DBpoweramp Writes | DBpoweramp Reads |
|---|---|---|---|
| $00 | ISO-8859-1 (Latin-1) | ASCII-only text | Assumes Latin-1; no heuristic detection |
| $01 | UTF-16 with BOM | Yes (with BOM) | Reads BOM; defaults to little-endian if missing |
| $02 | UTF-16BE without BOM | Rarely | Assumes big-endian if no BOM |
| $03 | UTF-8 | Yes (default for v2.4) | Validates UTF-8; may fall back to Latin-1 |

**Source:** [ID3v2.4 spec](https://id3.org/id3v2.4.0-structure)

### 3.2 When Declared Encoding ≠ Actual Bytes

This is the most common source of encoding errors. Example:

```
Frame header:  TIT2 $03 (UTF-8 declared)
Frame bytes:    C3 A4 C3 B6 C3 BC        // "äöü" in UTF-8
               Valid UTF-8 → displays correctly
               
Frame header:  TIT2 $00 (ISO-8859-1 declared)
Frame bytes:    E4 F6 FC                  // "äöü" in Windows-1252
               Declared as Latin-1, actual Windows-1252
               → Shows "Ã¤Ã¶Ã¼" in UTF-8 context (Mojibake)
```

### 3.3 DBpoweramp's $00 (ISO-8859-1) Handling

**DBpoweramp reads $00 frames as:**
1. Valid UTF-8: decoded as UTF-8
2. Valid ISO-8859-1: decoded as ISO-8859-1 → converted to output encoding
3. Invalid for both: **no recovery** — bytes passed through as-is or replaced with ?

**TagLib (DBpoweramp's underlying library) behavior for $00:**
- Default: ISO-8859-1 (Latin-1) → UTF-16 internal
- **No automatic charset detection** — the declared encoding is trusted
- For non-standard encodings: must subclass `Latin1StringHandler`

**Source:** [TagLib — ID3v2::Latin1StringHandler](https://taglib.org/api/classTagLib_1_1ID3v2_1_1Latin1StringHandler.html) — "An abstraction for the ISO-8859-1 string to data encoding in ID3v2 tags... ID3v2 tag can store strings in ISO-8859-1 (Latin1), and TagLib only supports genuine ISO-8859-1 by default."

### 3.4 Fixing Encoding Mismatches with ID Tag Update

DBpoweramp's ID Tag Update utility can **rewrite** ID3v2 tags in a target encoding:

```
ID Tag Update → writes tags with the encoding set in:
  dBpoweramp Configuration → Codecs → Advanced → ID3v2 Text Encoding
Options:
  - Unicode (UTF-16)
  - UTF-8
  - ISO-8859-1
```

**How it works:**
1. Reads tags (using whatever encoding the bytes happen to be in)
2. Re-encodes text with the target encoding
3. Writes new frames with correct encoding byte

**Source:** [DBpoweramp ID Tag Update Help](https://dbpoweramp.com/Help/dMC/idtagupdate.html) — "convert to [ID Tag Update] as the encoder, it will read the tags, remove them and rewrite as the options set in Control Center."

---

## 4. Windows-1252 Fallback Behavior

### 4.1 The Windows-1252 vs. ISO-8859-1 Overlap

Windows-1252 (CP1252) is a **superset** of ISO-8859-1. The difference: bytes 0x80–0x9F are control characters in ISO-8859-1 but printable characters in Windows-1252.

```
Byte    ISO-8859-1      Windows-1252
0x80    (control)        € (Euro sign)
0x91    (control)        ' (left single quote)
0x92    (control)        ' (right single quote)
0x93    (control)        " (left double quote)
0x94    (control)        " (right double quote)
0x95    (control)        • (bullet)
0x96    (control)        – (en dash)
0x97    (control)        — (em dash)
0x99    (control)        ™ (trademark)
```

### 4.2 Detection Heuristic (for repair tools)

Tools like fix-music-tags use this two-stage check:

1. **≥20% Latin-1 high-range check:** Count bytes in range 0x80–0xFF. If ≥20%, proceed.
2. **Round-trip test:** Decode with candidate encoding (CP1252, GB18030, etc.) and re-encode. If the round-trip produces the original bytes with no replacement characters (U+FFFD), the encoding is likely correct.

**Source:** [fix-music-tags — GitHub](https://github.com/mentax007/fix-music-tags)

### 4.3 DBpoweramp's Approach

DBpoweramp does **NOT** perform automatic encoding detection. Instead:
- **Writing:** Uses configured encoding (UTF-8 or UTF-16) — no ambiguity
- **Reading:** Trusts the declared encoding byte; no heuristic detection
- **Repair:** ID Tag Update can rewrite with target encoding, but assumes input is already valid

**User workaround:** If reading garbled text, use ID Tag Update to rewrite in UTF-8, which will fix most cases (UTF-8 is unambiguous).

---

## 5. ID3v1 Encoding Detection

### 5.1 ID3v1 Has No Encoding Byte

ID3v1 was designed in 1996 for ISO-8859-1. There is **no encoding indicator** in the 128-byte tag.

**Practical reality:** ID3v1 tags from Windows software are usually Windows-1252. ID3v1 tags from Japanese/Chinese software may be Shift-JIS, EUC-JP, or GB18030. ID3v1 tags from European software may be ISO-8859-1 or Windows-1252.

### 5.2 DBpoweramp's ID3v1 Reading Behavior

**TagLib (DBpoweramp's library):** Only supports ISO-8859-1 by default.

```cpp
// TagLib ID3v1::StringHandler default
TagLib::String parse(const TagLib::ByteVector &data) const override {
    // Assumes ISO-8859-1; no detection
    return TagLib::String(data, TagLib::String::Latin1);
}
```

**Source:** [TagLib ID3v1::StringHandler](https://taglib.org/api/classTagLib_1_1ID3v1_1_1StringHandler.html)

**DBpoweramp reads ID3v1 as:**
1. Raw bytes decoded as ISO-8859-1
2. Converted to target output encoding
3. Characters not in ISO-8859-1 (bytes 0x80+) are converted via Latin-1 code point (U+0080–U+00FF)
4. **Result:** Windows-1252 characters appear as wrong Latin-1 characters

### 5.3 User Setting for ID3v1 Encoding

**DBpoweramp does not expose an ID3v1 encoding setting** in the standard UI. Workarounds:

1. **Disable ID3v1 writing** (to avoid corruption): Configuration → Codecs → MP3 → Advanced → ID3v1 Version: "None"
2. **Use ID Tag Update** to strip ID3v1 from existing files
3. **Third-party repair:** Use fix-music-tags or mid3v2 to detect and convert ID3v1 encoding before DBpoweramp processing

**Source:** [DBpoweramp Forum — ID3 tags in WAV](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/30115-id-tags-in-mp3-files-truncated-at-30-characters)

### 5.4 getID3 Encoding Detection (Reference Implementation)

The getID3 library implements a detection heuristic for ID3v1:

```php
// ID3v1 encoding detection: check for non-Latin-1 high bytes
if (preg_match('#^[\x00-\x40\xA8\xB8\x80-\xFF]+$#', $value)) {
    // Byte pattern suggests non-ISO-8859-1 encoding
    foreach (array('Windows-1251', 'KOI8-R') as $id3v1_bad_encoding) {
        if (function_exists('mb_convert_encoding') &&
            @mb_convert_encoding($value, $id3v1_bad_encoding, $id3v1_bad_encoding) === $value) {
            $ID3v1encoding = $id3v1_bad_encoding;
            break;
        }
    }
}
```

**Source:** [ILIAS getid3_id3v1](https://ildoc.hrz.uni-giessen.de/ildoc/release_5-3/html/d3/d1a/classgetid3__id3v1.html)

---

## 6. UTF-16 BOM Handling

### 6.1 BOM Types and Their Meanings

| BOM Bytes | Encoding | Endianness |
|---|---|---|
| `FE FF` | UTF-16BE | Big-endian |
| `FF FE` | UTF-16LE | Little-endian |
| None | UTF-16BE (per Unicode spec) | Default assumption |

### 6.2 DBpoweramp's UTF-16 BOM Behavior

**Writing:** DBpoweramp always writes a **little-endian BOM** (`FF FE`) when using UTF-16 encoding. This is the Windows standard and maximizes compatibility with Windows Media Player and other PC software.

**Source:** FFmpeg (used internally by many converters) writes `FE FF` BOM (big-endian). DBpoweramp's TagLib-based approach uses little-endian.

### 6.3 Reading UTF-16 Without BOM

When a UTF-16 frame (encoding $01 or $02) has no BOM:

| Scenario | TagLib Behavior | DBpoweramp Result |
|---|---|---|
| Encoding $01, no BOM | DefaultUTF16WithBOMByteOrder = LittleEndian | Reads as UTF-16LE |
| Encoding $02 (UTF-16BE) | No BOM expected; reads as-is | Reads as big-endian |

**Source:** [dhowden/tag — id3v2frames.go](https://github.com/dhowden/tag/blob/master/id3v2frames.go) — "DefaultUTF16WithBOMByteOrder is the byte order used when the 'UTF16 with BOM' encoding is specified without a corresponding BOM in the data."

### 6.4 FFmpeg's UTF-16 Encoding Decision

FFmpeg (used by some DBpoweramp codec paths) uses a smarter approach:

```c
// FFmpeg id3v2enc.c
// check if the strings are ASCII-only and use UTF16 only if they're not
if (enc == ID3v2_ENCODING_UTF16BOM && string_is_ascii(str1) &&
    (!str2 || string_is_ascii(str2)))
    enc = ID3v2_ENCODING_ISO8859;

// If writing UTF-16, always add BOM
avio_wl16(pb, 0xFEFF); /* BOM */
```

**This means:** FFmpeg only uses UTF-16 when the text contains non-ASCII characters. Pure ASCII is written as ISO-8859-1. This optimizes for file size and compatibility.

### 6.5 UTF-16 in ID3v2.3 vs. ID3v2.4

| Version | $01 (UTF-16 BOM) | $02 (UTF-16BE) |
|---|---|---|
| ID3v2.3 | Valid (UCS-2 with BOM, treated as UTF-16) | Invalid (not supported) |
| ID3v2.4 | Valid (UTF-16 with BOM) | Valid (UTF-16BE without BOM) |

DBpoweramp defaults to **ID3v2.4** with UTF-8 encoding, avoiding most UTF-16 complexity.

---

## 7. UTF-16BE (No BOM) Handling

### 7.1 The Rarity of UTF-16BE

UTF-16BE (encoding byte $02 in ID3v2.4) without BOM is **extremely rare** in practice. Most software either:
- Uses UTF-16 with BOM ($01)
- Uses UTF-8 ($03)

### 7.2 DBpoweramp's $02 Behavior

When reading a $02 frame:
- TagLib interprets bytes as UTF-16BE
- Each pair of bytes is decoded as a big-endian 16-bit code unit
- No byte-order detection needed (explicit big-endian)

When writing:
- DBpoweramp does NOT use $02 encoding by default
- All UTF-16 is written with BOM ($01) and little-endian byte order

---

## 8. Unicode Normalization Forms

### 8.1 The Four Normalization Forms

| Form | Description | Use Case |
|---|---|---|
| **NFC** (Composed) | Characters as single code points (é = U+00E9) | Display, file exchange |
| **NFD** (Decomposed) | Base + combining characters (é = e + U+0301) | Internal processing, macOS HFS+ |
| **NFKC** | Compatibility decomposed + recomposed |Identifiers, security |
| **NFKD** | Compatibility decomposed | Stripping formatting |

### 8.2 Why Normalization Matters for Audio Tags

```
Example: "résumé"

NFC:  U+0072 U+00E9 U+0073 U+0075 U+006D U+00E9  (6 code points)
NFD:  U+0072 U+0065 U+0301 U+0073 U+0075 U+006D U+0065 U+0301  (8 code points)

Byte representation in UTF-8:
NFC:  72 C3 A9 73 75 6D C3 A9  (8 bytes)
NFD:  72 65 CC 81 73 75 6D 65 CC 81  (10 bytes)
```

**Impact:**
- **String comparison:** NFC "résumé" ≠ NFD "résumé" (different byte sequences)
- **File comparison:** Same visual name but different bytes → different files on some systems
- **Database lookups:** Without normalization, "résumé" may not match "résumé"

### 8.3 TagLib's Internal String Storage

TagLib stores strings internally as **UTF-16** (without BOM, CPU byte order). This means:
- Input encoding (Latin-1, UTF-8, UTF-16) is decoded to Unicode code points
- Stored as UTF-16 internally
- Output encoding is selected at write time

**Unicode normalization is NOT performed by TagLib.** A string that arrives as NFD stays NFD internally.

### 8.4 MusicBrainz Picard's Normalization Approach

Picard uses normalization **only for filenames**, not for tags:

```python
# picard/util/filenaming.py
if mac_compat:
    path = unicodedata.normalize("NFD", path)  # For HFS+ filesystem
```

For tags, Picard preserves the original normalization form from the MusicBrainz database.

**Source:** [Picard filenaming.py](https://github.com/metabrainz/picard/blob/master/picard/util/filenaming.py)

### 8.5 DBpoweramp's Normalization Behavior

**DBpoweramp does NOT perform Unicode normalization** on tag text. This means:
- NFC source → NFC output (no change)
- NFD source → NFD output (no change)
- Mixed: may produce inconsistent comparisons between files

**Impact on user:** Files with NFD-encoded metadata may not compare equal to files with NFC-encoded metadata from a different source. This can cause:
- Duplicate detection failures
- Library browsing inconsistencies
- Search/filter mismatches

### 8.6 macOS HFS+ and APFS Normalization

**HFS+:** Always uses **NFD** for filenames. Picard normalizes to NFD for macOS compatibility.

**APFS:** Uses **NFC** by default, but is normalization-insensitive for comparison.

**DBpoweramp on macOS:** If writing to HFS+ volumes, filenames with NFC characters may appear differently than intended. No automatic NFD normalization is performed.

---

## 9. Why Encoding Matters for String Comparison and Filesystem Compatibility

### 9.1 Tag Comparison Without Normalization

```
File A:  ARTIST = "Björk"  (NFC: composed)
File B:  ARTIST = "Björk"  (NFD: decomposed + combining acute)
File A bytes (UTF-8): 42 6A C3 B6 72 6B  (C3 B6 = U+00F6)
File B bytes (UTF-8): 42 6A 62 CC 81 72 6B  (62 = o, CC 81 = combining acute)

These compare as DIFFERENT strings even though they display identically.
```

### 9.2 Filesystem Compatibility

| Filesystem | Encoding | Normalization | Tag Encoding Implication |
|---|---|---|---|
| NTFS | UTF-16LE | None | NFC preferred; NFD works |
| HFS+ | UTF-16 (decomposed) | NFD | Must NFD for filenames |
| APFS | UTF-8 (composed) | NFC preferred | NFC preferred |
| ext4 | UTF-8 | Varies | NFC preferred |
| VFAT/FAT32 | UTF-8 (compressed) | None | ASCII-only for reliability |

### 9.3 The "Asian Characters === ???" Problem

**The Issue:** Japanese/Chinese characters appear as "???" after DBpoweramp conversion.

**Root Cause:** ID3v2 encoding byte set to ISO-8859-1 ($00), but actual bytes are Shift-JIS or another CJK encoding.

**Solution in DBpoweramp:**
```
dBpoweramp Configuration → Codecs → Advanced → ID3v2 Text Encoding:
  Set to "Unicode" (UTF-16)
```

**Source:** [DBpoweramp Forum — Asian characters](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/14515-how-do-i-preserve-asian-characters-in-id3-tags) — "dBpoweramp Configuration >> Codecs >> Advanced and set the ID3v2 option to Unicode."

---

## 10. Edge Cases

### Edge Case 1: ID3v2.3 + UTF-8

ID3v2.3 predates the UTF-8 encoding byte ($03 was added in ID3v2.4). Some software writes UTF-8 bytes with the $01 (UTF-16 BOM) encoding byte set, which is **incorrect**. TagLib may:
- Try to decode UTF-8 bytes as UTF-16 → garbled output
- Or fall back to Latin-1 → different garbled output

### Edge Case 2: UTF-8 BOM (EF BB BF)

The UTF-8 BOM (`EF BB BF`) is sometimes prepended to text fields. This is non-standard but tolerated. DBpoweramp:
- Reads: BOM bytes may appear as garbage characters in the string
- Writes: Does NOT write UTF-8 BOM (correct per standard)

### Edge Case 3: Null Byte in String

Vorbis spec: strings are length-prefixed, not null-terminated. A literal null byte (U+0000) in the middle of a string is valid Vorbis but may truncate in ID3v2.

### Edge Case 4: GB18030 Encoding

GB18030 is a Chinese standard that encodes characters as 1, 2, or 4 bytes. An ID3v1 tag written by Chinese software may use GB18030. Without detection:
- Bytes decoded as Latin-1 → garbage
- The garbage may include valid UTF-8 byte sequences → double-decoded garbage

### Edge Case 5: Latin-1 Surrogate Pairs

Latin-1 (ISO-8859-1) only covers U+0000–U+00FF. Any character outside this range in a $00 (Latin-1) frame is **impossible** by spec but happens due to mislabeling. These bytes cannot be round-tripped through Latin-1.

---

## 11. Code Examples

### 11.1 Mutagen ID3v2 Encoding Handling (Reference)

```python
# mutagen/id3.py — simplified
def ParseFrameText(self, data):
    encoding = data[0]  # The encoding byte
    if encoding == 0x00:
        return data[1:].decode('latin-1')
    elif encoding == 0x01:
        # UTF-16 with BOM — check BOM bytes
        if data[1:3] == b'\xff\xfe':
            return data[3:].decode('utf-16-le')
        elif data[1:3] == b'\xfe\xff':
            return data[3:].decode('utf-16-be')
        else:
            return data[1:].decode('utf-16-le')  # Assume LE if no BOM
    elif encoding == 0x02:
        return data[1:].decode('utf-16-be')  # No BOM in $02
    elif encoding == 0x03:
        return data[1:].decode('utf-8')
```

### 11.2 VLC's TagLib Charset Detection (Reference)

```cpp
// VLC patch for TagLib Latin-1 handler
// When ISO-8859-1 decoding produces invalid Latin-1 characters,
// try charset detection via uchardet
static char *TryDetectCharset(const char *str) {
    uchardet_t ud = uchardet_new();
    uchardet_handle_data(ud, str, strlen(str));
    uchardet_data_end(ud);
    const char *psz_charset = uchardet_get_charset(ud);
    return strdup(psz_charset);  // Returns e.g., "WINDOWS-1252"
}
```

**Source:** [VLC-devel mailing list](https://mailman.videolan.org/pipermail/vlc-devel/2020-October/139562.html)

### 11.3 Picard Encoding Detection (Reference)

```python
# picard/util/__init__.py
ENCODING_BOMS = {
    b'\xff\xfe\x00\x00': 'utf-32-le',
    b'\x00\x00\xfe\xff': 'utf-32-be',
    b'\xef\xbb\xbf': 'utf-8-sig',
    b'\xff\xfe': 'utf-16-le',
    b'\xfe\xff': 'utf-16-be',
}

def detect_file_encoding(path):
    # Check BOM first
    # Then statistical detection (uchardet if available)
    # Fallback: utf-8
```

**Source:** [Picard util/__init__.py](https://github.com/metabrainz/picard/blob/master/picard/util/__init__.py)

---

## Summary: Would a User Notice Any Difference?

| Encoding Scenario | User Impact |
|---|---|
| Files with Windows-1252 in ID3v1 | Appear garbled; no auto-detection |
| Asian characters in ID3v1 | Show as "???" unless converted to UTF-16/UTF-8 |
| UTF-8 BOM in frames | Garbage characters at start of field |
| UTF-16 without BOM | May be read as wrong endianness by some players |
| NFD vs NFC normalization | String comparison fails; duplicate detection misses |
| GB18030 ID3v1 | Complete garbling; no detection |

**Bottom Line:** DBpoweramp produces clean, standard-compliant output (UTF-8 or UTF-16). The encoding problems users encounter come from **input files** with mismatched encodings. DBpoweramp does NOT detect or auto-fix input encoding — users must use ID Tag Update (with UTF-8 target) to repair malformed tags, or disable ID3v1 writing to avoid propagating legacy encoding errors.
