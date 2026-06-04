# DBpoweramp Music Converter — APEv2 Tag Behavior
*Generated: 2026-06-04 | Format: APE (Monkey's Audio) | Confidence: High*

---

## 1. Executive Summary

Monkey's Audio (.ape) files use APEv2 tags as their primary and native metadata system. APEv2 is a binary tag format with a header/footer structure and individual items containing key-value pairs with type flags. The format supports UTF-8 text, binary data (for cover art), and external locators. DBpoweramp's APEv2 handling must respect the structural layout: items are sorted by ascending size, the tag header and footer are 32 bytes each, item keys must be 2–255 printable ASCII characters (0x20–0x7E), and item values can be up to 2^32-1 bytes. Keys are technically case-sensitive but readers are encouraged to treat them case-insensitively. Cover art is stored as a binary item with the key "Cover Art (Front)".

---

## 2. Format Overview

An APEv2 tag appends to the end of an audio file (preferred location) or prepends (discouraged). The structure is:

```
[Audio Data] [APE Tag Footer (32 bytes)] [APE Tag Items] [APE Tag Header (32 bytes)]
```

Or if the tag is at the beginning:
```
[APE Tag Header (32 bytes)] [APE Tag Items] [Audio Data]
```

### APE Tag Header/Footer (32 bytes each)

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 8 | Preamble | `APETAGEX` |
| 8 | 4 | Version | `2000` (v2.0) or `1000` (v1.0) |
| 16 | 4 | Tag Size | Bytes from end of header to end of footer |
| 20 | 4 | Item Count | Number of items |
| 24 | 4 | Flags | Tag flags (header/footer presence, item type) |
| 28 | 4 | Reserved | Must be zero |

**Source:** [Mutagen Specs — APEv2](http://mutagen-specs.readthedocs.io/en/latest/apev2/apev2.html)

### Tag Version Differences

| Version | Footer | Header | Item Flags |
|---------|--------|--------|------------|
| **v2.0** | Required | Optional | Used |
| **v1.0** | Not used | Present at start | Ignored (all zero) |

DBpoweramp writes APEv2 **v2.0** with footer present and header optional.

---

## 3. Tag Reading Behavior

### APE Tag Item Structure

Each item follows this layout:

| Offset | Size | Field | Description |
|--------|------|-------|-------------|
| 0 | 4 | Value Size | Length of value in bytes (32-bit LE) |
| 4 | 4 | Item Flags | 32-bit flags (bits 0-1 = type) |
| 8 | 2–255 | Item Key | ASCII printable characters (null-terminated) |
| — | 1 | Key Terminator | 0x00 |
| — | N | Item Value | Text, binary, or locator |

**Item Flags (bits 0-1 = data type):**

| Value | Type | Description |
|-------|------|-------------|
| 0 | Text | UTF-8 encoded text string |
| 1 | Reserved | — |
| 2 | Binary | Raw binary data (e.g., cover art) |
| 3 | External Locator | URL or file path reference |

Bit 2: Read-only flag (0 = R/W, 1 = read-only)

**Source:** [TagLib — ape-tag-format.txt](https://github.com/taglib/taglib/blob/master/taglib/ape/ape-tag-format.txt), [Hydrogenaudio — APE Tag Item](https://wiki.hydrogenaudio.org/index.php?title=APE_Tag_Item)

### Key Case Sensitivity

Per the APEv2 spec:
- **Keys are case-sensitive** in the format.
- **It is illegal to use keys that differ only in case** (e.g., `Artist` and `ARTIST` cannot both exist).
- **Readers are encouraged to treat keys case-insensitively** for lookup.

Most tools (TagLib, DBpoweramp) normalize keys to a consistent case internally.

### Max Key Length

Keys must be **2 to 255 characters** of printable ASCII (0x20–0x7E). Null (0x00) and equals (0x3D) are excluded from valid key characters.

### Standard APEv2 Keys

| Key | Value Type | Description |
|-----|-----------|-------------|
| `Title` | Text | Track title |
| `Artist` | Text | Artist name |
| `Album` | Text | Album name |
| `Comment` | Text | Comment |
| `Year` | Text | Year (YYYY or YYYY-MM-DD) |
| `Track` | Text or DWORD | Track number |
| `Genre` | Text | Genre |
| `Media` | Text | Media type |
| `Catalog` | Text | Catalog number |
| `Composer` | Text | Composer |
| `Publisher` | Text | Record label |
| `Copyright` | Text | Copyright holder |
| `Cover Art (Front)` | Binary | Front cover image (JPEG/PNG) |
| `Cover Art (Back)` | Binary | Back cover image |
| `CueSheet` | Text | Embedded cue sheet |
| `ReplayGain Track Gain` | Text | e.g., "-6.20 dB" |
| `ReplayGain Track Peak` | Text | e.g., "0.99996948" |
| `ReplayGain Album Gain` | Text | e.g., "-4.50 dB" |
| `ReplayGain Album Peak` | Text | e.g., "0.87654321" |

---

## 4. Tag Writing Behavior

### Item Sorting

APE tag items should be sorted by **ascending size** in the tag. This is a recommendation (not strictly required) to facilitate streaming. Large items (like cover art) come last.

### Value Length Limits

| Limit | Value |
|-------|-------|
| Max key length | 255 bytes |
| Max value length | 4,294,967,295 bytes (2^32-1) — theoretical |
| Recommended practical limit | 1 MB per item, 4 MB total tag |

WavPack's APEv2 implementation caps at 1 MB per item unless `--allow-huge-tags` is used.

### Padding Bytes

APEv2 tags can include padding bytes (arbitrary data) between the last item and the footer. This allows adding new items without rewriting the entire tag. DBpoweramp may or may not preserve padding during re-tagging.

### Forbidden Keys

The following keys are forbidden and may cause parsing issues:
- `ID3`
- `TAG`
- `OggS`
- `MP+`

These are reserved for ID3v1, APEv1, OGG headers, and MP+ tag detection.

### Binary Items for Cover Art

Cover art is stored as a binary item (flags bits 0-1 = 2):

```
Key: "Cover Art (Front)"
Flags: 0x00000002 (binary type)
Value: Raw JPEG or PNG image bytes
```

DBpoweramp writes cover art as binary items with the standard "Cover Art (Front)" key.

---

## 5. Normalization Rules

| Rule | Behavior |
|------|----------|
| **Key case** | Typically UPPERCASE (e.g., `TITLE`, `ARTIST`) |
| **Key character set** | Printable ASCII 0x20–0x7E only |
| **Value encoding** | UTF-8 for text items |
| **Item sorting** | Ascending by size (recommended) |
| **Duplicate keys** | Not allowed — update replaces existing |
| **ReplayGain format** | "-6.20 dB" (with space, signed float) |

---

## 6. Edge Cases

1. **APEv1 vs APEv2 in the same file**: APEv1 uses a header at the file start. APEv2 uses a footer at the file end. A file could theoretically contain both. DBpoweramp reads APEv2 preferentially and may strip APEv1 on write.

2. **Case-sensitive key collision**: A file with both `Artist=John` and `ARTIST=Paul` (different case keys) violates the spec. Some readers handle this gracefully; others may crash or corrupt the tag.

3. **External locator items (type 3)**: Items with URLs or file paths (e.g., `Cover Art=file://album.jpg`) are valid but rarely used. DBpoweramp does not write locator items.

4. **Tag size overflow**: If tag size exceeds the 32-bit size field maximum (4 GB), the tag is invalid. This is not a practical concern for audio files.

5. **`CueSheet` item size**: Large embedded cue sheets can exceed 1 MB. WavPack allows these with `--allow-huge-tags`. DBpoweramp may truncate or reject items exceeding its internal limits.

---

## 7. DBpoweramp-Specific Behavior

- **Native format**: APEv2 is DBpoweramp's primary tag format for Monkey's Audio files.
- **Tag writing**: Writes APEv2 v2.0 with footer. Items sorted by size.
- **Cover art**: Written as binary item with key "Cover Art (Front)".
- **ReplayGain**: Not written natively. External tools required.
- **ID3v1 coexistence**: APEv2 and ID3v1 can coexist. DBpoweramp may write both.
- **Item ordering**: Large items (cover art) written last.
- **Key normalization**: Keys normalized to standard capitalization.

---

## 8. Verification Checklist

- [ ] Tag is APEv2 v2.0 with footer present
- [ ] Items sorted by ascending size
- [ ] No forbidden keys (ID3, TAG, OggS, MP+)
- [ ] Cover art binary item key is "Cover Art (Front)" (exact spelling)
- [ ] No duplicate keys (case-insensitive check)
- [ ] Key characters are printable ASCII (no null bytes in keys)
- [ ] ReplayGain format matches "-6.20 dB" pattern
- [ ] Tag size field in footer matches actual size

---

## 9. Sources

1. [Mutagen Specs — APEv2 Specification](http://mutagen-specs.readthedocs.io/en/latest/apev2/apev2.html)
2. [TagLib — ape-tag-format.txt](https://github.com/taglib/taglib/blob/master/taglib/ape/ape-tag-format.txt)
3. [Hydrogenaudio Knowledgebase — APE Tag Item](https://wiki.hydrogenaudio.org/index.php?title=APE_Tag_Item)
4. [Hydrogenaudio Knowledgebase — APE key](https://wiki.hydrogenaudio.org/index.php?title=APE_key)
