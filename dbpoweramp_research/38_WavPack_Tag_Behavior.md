# DBpoweramp Music Converter — WavPack Tag Behavior
*Generated: 2026-06-04 | Format: WV (WavPack) | Confidence: High*

---

## 1. Executive Summary

WavPack (.wv) files store all metadata in **APEv2 tags** appended to the end of the file, following the same structure as standalone APE tags. WavPack does NOT use a native WavPack-specific tag format — APEv2 is the canonical metadata system. ID3v2 tags may also be present in WavPack files (imported from DSF or other sources), but WavPack treats these as "trailing wrappers" and imports applicable items into the APEv2 tag on read. Cover art is stored as a binary APEv2 item with the key "Cover Art (Front)". ReplayGain is managed via dedicated APEv2 fields. DBpoweramp's WavPack tag handling is thus identical to its APEv2 handling, with the addition of WavPack-specific custom fields (WAVPACK_VERSION, ENCODER, SETTINGS).

---

## 2. Format Overview

A WavPack file structure:

```
[WavPack audio data] [APEv2 tag footer] [APEv2 tag items] [APEv2 tag header]
```

WavPack stores metadata in APEv2 at the end of the file. The APEv2 tag consists of:
- **Tag Header** (32 bytes) — optional at end, required at start
- **Tag Items** — key-value pairs sorted by ascending size
- **Tag Footer** (32 bytes) — required at end, mirrors header with flag difference

The WavPack correction file (.wvc) may accompany the .wv file and also contains APEv2 tags.

**Source:** [WavPack Documentation](https://www.wavpack.com/wavpack_doc.html), [GitHub — WavPack wavpack_doc.html](https://github.com/dbry/WavPack/blob/master/doc/wavpack_doc.html)

---

## 3. Tag Reading Behavior

### APEv2 as Primary Tag Format

WavPack uses APEv2 as its **only native metadata system**. All tag fields are APEv2 items. TagLib and other WavPack-aware libraries read APEv2 tags from .wv files.

### ID3v2 Handling

Per the WavPack documentation:

> "WavPack considers this a trailing 'wrapper' and stores it in the WavPack file as such so that the original file can be restored verbatim. However, stored this way it is not easily accessible for reading (and it is certainly not writable). If a trailing ID3v2.3 or ID3v2.4 tag is present in the DSF file (or any other file type) to be scanned, all applicable items are imported into the APEv2 tag."

DBpoweramp's WavPack tag reader:
1. Reads APEv2 tags first (primary).
2. Imports applicable items from any trailing ID3v2 tag into the APEv2 tag.
3. On write, only writes APEv2 (not ID3v2 back to the file).
4. Can use `--import-id3` flag to explicitly import ID3v2 items.

### Cover Art Embedding

Cover art in WavPack uses the standard APEv2 binary item:

```
Key: "Cover Art (Front)"
Flags: 0x00000002 (binary type)
Value: Raw JPEG or PNG image bytes
```

Multiple cover art images use different keys:
- `Cover Art (Front)` — Front cover
- `Cover Art (Back)` — Back cover
- `Cover Art (Leaflet)` — Inlay/leaflet
- `Cover Art (Media)` — Label/medium

**Source:** [WavPack Manpage — wavpack](https://manpages.debian.org/bookworm/wavpack/wavpack.1.en.html)

### WavPack Custom Tags

WavPack stores encoder-specific metadata in dedicated APEv2 fields:

| Key | Value Type | Description | Example |
|-----|-----------|-------------|---------|
| `WAVPACK_VERSION` | Text | WavPack encoder version | `5.5.0` |
| `Encoder` | Text | Encoder software | `WavPack 5.5.0` |
| `Settings` | Text | Encoding settings | `-hhx3` |
| `Cuesheet` | Text | Embedded cue sheet | (cue sheet text) |
| `Log` | Text | Encoding log | (log text) |

The `WvGain` utility manages ReplayGain tags:
- `REPLAYGAIN_TRACK_GAIN=-6.20 dB`
- `REPLAYGAIN_TRACK_PEAK=0.99996948`
- `REPLAYGAIN_ALBUM_GAIN=-4.50 dB`
- `REPLAYGAIN_ALBUM_PEAK=0.87654321`

**Source:** [WavPack Documentation](https://www.wavpack.com/wavpack_doc.html), [Debian Manpage — wavpack](https://manpages.debian.org/bookworm/wavpack/wavpack.1.en.html)

---

## 4. Tag Writing Behavior

### Writing Cover Art

Using `wavpack` command-line:

```bash
wavpack --write-binary-tag "Cover Art (Front)=@cover.jpg" input.wv
```

The `@` prefix indicates the value is a file path to be read as binary data.

**Size limit**: WavPack allows up to **1 MB** of tag data per item by default. For larger cover art (>1 MB), use:

```bash
wavpack --allow-huge-tags --write-binary-tag "Cover Art (Front)=@cover.jpg" input.wv
```

**DBpoweramp behavior**: Writes cover art using standard APEv2 binary items with the "Cover Art (Front)" key.

### Writing Text Tags

```bash
wavpack -w "Artist=Dizzy Gillespie" -w "Album=Live at the Village Vanguard" input.wv
```

Multiple `-w` options add or update items.

### Tag Size Limits

| Limit | Default | With --allow-huge-tags |
|-------|---------|----------------------|
| Per item | 1 MB | Unlimited |
| Total tag size | Limited by item count | Unlimited |

The 1 MB limit was implemented for portable device compatibility. Desktop players handle larger tags fine.

### WvTag Utility

The `wvtage` utility provides advanced tag manipulation:
- List tags: `wvtag --list file.wv`
- Extract cover art: `wvtag --extract-front-cover output.jpg file.wv`
- Delete tags: `wvtag --delete file.wv`
- Import from ID3v1: `wvtag --import-id3 file.wv`

---

## 5. Normalization Rules

| Rule | Behavior |
|------|----------|
| **Tag format** | APEv2 v2.0 with footer |
| **Key case** | Typically UPPERCASE (e.g., `ARTIST`, `ALBUM`) |
| **Cover art key** | `Cover Art (Front)` — exact spelling, space after "Cover" |
| **ReplayGain format** | `-6.20 dB` (signed float, space before "dB") |
| **ID3v2 import** | Automatic on read; stripped on write |
| **Item sorting** | Ascending by size (recommended) |

---

## 6. Edge Cases

1. **Cover art > 1 MB**: High-resolution cover art (4K, 8K) can exceed the default 1 MB limit. `wvtag` and `wavpack` with `--allow-huge-tags` handle these. DBpoweramp may silently truncate or fail with very large cover art.

2. **ID3v2 import removing original ID3v2**: When `wvtag --import-id3` imports ID3v2 items into APEv2 and then deletes the original ID3v2 tag, the original file cannot be restored verbatim. Use with caution for archival.

3. **Duplicate "Cover Art (Front)" entries**: If multiple cover art items use the same key, only one should exist per APEv2 spec. Some tools write multiple entries; DBpoweramp may deduplicate or replace.

4. **Cue sheets as text vs binary**: Embedded cue sheets can be stored as text items (UTF-8) or referenced externally. The standard key is `Cuesheet`. Large cue sheets may exceed item limits.

5. **Correction file (.wvc) tags**: The .wvc correction file may contain its own APEv2 tags, independent of the .wv file's tags. Tag reading tools must handle both.

---

## 7. DBpoweramp-Specific Behavior

- **Tag writing**: Uses APEv2 v2.0 for WavPack output.
- **Cover art**: Written as `Cover Art (Front)` binary APEv2 item.
- **ReplayGain**: Not written natively. Use `WvGain` utility.
- **ID3v1/ID3v2 import**: May import ID3v1 on read if APEv2 is absent. ID3v2 imported if `--import-id3` is configured.
- **WavPack custom fields**: `WAVPACK_VERSION`, `ENCODER`, `SETTINGS` preserved or stripped depending on configuration.
- **Tag size**: Respects default 1 MB limit; `--allow-huge-tags` available for large cover art.
- **Lossless tag updates**: Re-tagging does not affect audio data (WavPack's MD5 signature is verified).

---

## 8. Verification Checklist

- [ ] Tag is APEv2 v2.0 with footer present
- [ ] Cover art key is `Cover Art (Front)` (exact spelling)
- [ ] Cover art value is raw JPEG/PNG binary (not base64)
- [ ] No duplicate keys (case-insensitive)
- [ ] `REPLAYGAIN_TRACK_GAIN` format is `-6.20 dB`
- [ ] `ENCODER` field identifies the encoder (e.g., "dBpoweramp")
- [ ] No ID3v2 remnants if configured to strip (check end of file before audio end)
- [ ] Cue sheet present if originally embedded (`Cuesheet` key)

---

## 9. Sources

1. [WavPack Documentation](https://www.wavpack.com/wavpack_doc.html)
2. [GitHub — WavPack wavpack_doc.html](https://github.com/dbry/WavPack/blob/master/doc/wavpack_doc.html)
3. [Debian Manpage — wavpack](https://manpages.debian.org/bookworm/wavpack/wavpack.1.en.html)
4. [Debian Manpage — wvtag](https://manpages.debian.org/unstable/wavpack/wvtag.1.en.html)
5. [manpagez — WavPack man page](http://www.manpagez.com/man/1/wavpack/)
