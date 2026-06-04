# DBpoweramp Music Converter — WMA/ASF Tag Behavior
*Generated: 2026-06-04 | Format: WMA, ASF | Confidence: High*

---

## 1. Executive Summary

WMA files use the ASF (Advanced Systems Format) container with metadata stored as **attributes** — name-value pairs with associated data types. DBpoweramp's WMA tag handling works with the Windows Media Format SDK attribute set, which uses the `WM/` prefix convention for standardized fields (e.g., `WM/AlbumTitle`, `WM/TrackNumber`) and allows custom attributes. Cover art is stored as `WM/Picture` (binary ASF picture structure), not as a separate atom or comment field. The ASF format natively supports multiple values for the same attribute, and DRM-protected files embed rights management metadata that DBpoweramp cannot strip or modify.

---

## 2. Format Overview

ASF is a Microsoft-developed container format used for WMA, WMV, and other Windows Media content. The header section contains:

```
ASF Header:
  Header Object
  ├─ ASF_File_Properties_Object
  ├─ ASF_Stream_Properties_Object
  ├─ ASF_Content_Description_Object (title, author, copyright, description, rating)
  ├─ ASF_Extended_Content_Description_Object (WM/* attributes)
  ├─ ASF_Content_Distributor_Descriptor_Object
  └─ Stream-specific objects
```

The **ASF_Extended_Content_Description_Object** holds all `WM/` namespaced attributes and custom fields.

**Source:** [Microsoft Learn — Metadata (Win32 apps)](https://learn.microsoft.com/en-us/windows/win32/wmformat/metadata)

---

## 3. Tag Reading Behavior

### WM/* Attribute Names

Microsoft's Windows Media Format SDK defines a standardized attribute namespace:

| Attribute | Data Type | Description |
|-----------|-----------|-------------|
| `WM/AlbumTitle` | String | Album name |
| `WM/AlbumArtist` | String | Album artist |
| `WM/Track` | String or DWORD | Track number |
| `WM/TrackNumber` | String or DWORD | Track number (modern) |
| `WM/PartOfSet` | String or DWORD | Disc number |
| `WM/Year` | String or DWORD | Release year |
| `WM/Genre` | String | Genre |
| `WM/Author` | String | Artist (primary) |
| `WM/Title` | String | Track title |
| `WM/Description` | String | Description/comment |
| `WM/Copyright` | String | Copyright notice |
| `WM/Publisher` | String | Publisher/label |
| `WM/EncodingTime` | QWORD | Encoding timestamp |
| `WM/Provider` | String | Content provider |
| `WM/BeatsPerMinute` | DWORD | BPM |
| `WM/InitialKey` | String | Musical key |
| `WM/Mood` | String | Mood |
| `WM/IsCompilation` | DWORD | Compilation flag (1=yes) |
| `WM/MediaClassPrimaryID` | String | Media brainz ID |
| `WM/MediaClassSecondaryID` | String | Secondary ID |
| `WM/ContentDistributor` | String | Distributor |
| `WM/SubTitle` | String | Subtitle |
| `WM/PromotionURL` | String | Promo URL |
| `WM/AlbumCoverURL` | String | Cover art URL |
| `WM/Composer` | String | Composer |
| `WM/Conductor` | String | Conductor |
| `WM/EncodedBy` | String | Encoded by |
| `WM/EncodingSettings` | String | Encoder settings |
| `WM/Lyrics` | String | Lyrics text |
| `WM/Lyrics_Synchronised` | String | Synchronized lyrics |
| `WM/Period` | String | TV period |
| `WM/Period` | String | TV season |
| `WM/Picture` | Binary | Embedded cover art |
| `WM/UniqueFileIdentifier` | String | UID |
| `WM/Writer` | String | Lyricist/writer |
| `WM/ToolName` | String | Encoder name |
| `WM/ToolVersion` | String | Encoder version |
| `WM/Language` | String | Language |
| `WM/ParentalRating` | String | Rating |
| `WM/Producer` | String | Producer |
| `WM/UserWebURL` | String | User URL |
| `WM/AuthorURL` | String | Author URL |

**Source:** [Microsoft Docs — WM/ attributes for music files](https://github.com/MicrosoftDocs/win32/blob/docs/desktop-src/wmformat/attributes-for-music-files.md), [Microsoft Docs — Metadata Properties](https://learn.microsoft.com/en-us/windows/win32/medfound/metadata-properties-for-media-files)

### ID3 Tag Mapping in ASF

ASF files can store ID3 tags using the `ID3/` prefix:

| ASF Attribute | ID3v2 Equivalent | ID3v1 Equivalent |
|---------------|------------------|------------------|
| `ID3/TPE1` | `TPE1` (Lead Artist) | `Artist` |
| `ID3/TIT2` | `TIT2` (Title) | `Title` |
| `ID3/TALB` | `TALB` (Album) | `Album` |
| `ID3/TYER` | `TYER` (Year) | `Year` |
| `ID3/TRCK` | `TRCK` (Track) | — |
| `ID3/TCON` | `TCON` (Genre) | `Genre` |
| `ID3/APIC` | `APIC` (Picture) | — |
| `ID3/TXXX` | `TXXX` (User-defined) | — |
| `ID3/COMM` | `COMM` (Comment) | — |

DBpoweramp reads `ID3/*` attributes and may convert them to `WM/*` equivalents on write.

**Source:** [Microsoft Learn — ID3 Tag Support](https://learn.microsoft.com/en-us/windows/win32/wmformat/id3-tag-support)

### WM/Picture for Cover Art

The `WM/Picture` attribute stores cover art as a binary ASF picture structure:

```
WM/Picture:
  [1 byte] Picture type (1=Front cover, 2=Back cover, etc.)
  [4 bytes] Picture data size (N)
  [N bytes] MIME type string (null-terminated, e.g., "image/jpeg")
  [N bytes] Picture description (null-terminated string)
  [N bytes] Raw picture data (JPEG/PNG/BMP)
```

Supported MIME types: `image/jpeg`, `image/png`, `image/bmp`

---

## 4. Tag Writing Behavior

### Multiple Value Handling

ASF natively supports **multiple values per attribute** for certain data types. Multiple `WM/Author` entries are common for multi-artist tracks. DBpoweramp writes multiple values by creating multiple attribute entries with the same name.

### DRM / Encryption Tags

DRM-protected WMA files contain additional attributes and structures:
- `WM/DRM` — DRM activation flags
- `WM/DRMFlags` — DRM state
- `WM/WMDRMContentDistributor` — Content distributor ID

**DBpoweramp behavior**: DRM cannot be stripped or bypassed. Protected files are either transcoded (if key is available) or rejected with an error message.

### Attribute Data Types

| Type | Description |
|------|-------------|
| `DWORD` | 32-bit unsigned integer (track number, disc, BPM) |
| `QWORD` | 64-bit unsigned integer (timestamps, durations) |
| `STRING` | Null-terminated UTF-16LE string |
| `BINARY` | Byte array (cover art, GUIDs) |
| `BOOL` | 32-bit boolean (0 or 1) |

DBpoweramp selects the appropriate type based on field content. Track numbers may be written as `STRING` ("3") or `DWORD` (3).

### Extended Content Description Object

Custom attributes are stored in the `ASF_Extended_Content_Description_Object`. Any attribute name is valid, though `WM/` namespaced attributes are preferred for compatibility.

---

## 5. Normalization Rules

| Rule | Behavior |
|------|----------|
| **Track number** | Written as `WM/TrackNumber` (DWORD or STRING) |
| **PartOfSet** | Written as `WM/PartOfSet` for disc number |
| **Genre** | Text string; no numeric genre mapping (unlike ID3) |
| **Cover art** | `WM/Picture` binary structure, not base64 |
| **Encoding** | UTF-16LE for all string attributes (native ASF encoding) |
| **Case** | `WM/*` prefix is always uppercase; values are preserved as-is |

---

## 6. Edge Cases

1. **`WM/Year` vs `WM/EncodingTime`**: A file may have both a release year (`WM/Year`) and an encoding timestamp (`WM/EncodingTime`). DBpoweramp reads both and maps them to appropriate fields during conversion.

2. **`WM/Picture` with incorrect MIME type**: If a tool writes `image/jpg` instead of `image/jpeg`, some players may fail to display the image. DBpoweramp writes `image/jpeg` for JPEG content.

3. **ASF header size and streaming**: Large metadata objects can increase the ASF header beyond streaming-friendly sizes. WMA streams typically require small headers for efficient HTTP streaming.

4. **Duplicate attribute names**: Some ASF writers create duplicate `WM/Author` entries. DBpoweramp reads all values and may deduplicate or preserve duplicates on write.

5. **ID3-style attributes with `ID3/` prefix**: Old rippers wrote ID3 frames with the `ID3/` prefix directly into ASF. These are non-standard but readable. DBpoweramp may convert them to `WM/*` equivalents.

---

## 7. DBpoweramp-Specific Behavior

- **Default output**: WMA with standard `WM/*` attributes and `WM/Picture` for cover art.
- **Tag writing**: Uses the ASF metadata system. Strings are UTF-16LE encoded.
- **Cover art**: Written as `WM/Picture` binary structure. Multiple images possible via multiple `WM/Picture` entries.
- **ReplayGain**: Not natively supported in WMA/ASF. Use `wvgain` or player-based normalization.
- **DRM**: Cannot be stripped. Only transcodes if key is available.
- **Extended attributes**: Custom `WM/*` fields preserved if present in source.
- **ID3 tag preservation**: `ID3/*` attributes may be stripped or converted to `WM/*`.

---

## 8. Verification Checklist

- [ ] `WM/Title` shows track title
- [ ] `WM/AlbumTitle` shows album name
- [ ] `WM/TrackNumber` displays correctly (numeric)
- [ ] `WM/Picture` shows cover art (check binary structure validity)
- [ ] `WM/Year` / `WM/EncodingTime` present
- [ ] No DRM remnants if source was DRM-free
- [ ] Multiple `WM/Author` entries preserved for multi-artist tracks
- [ ] UTF-16LE encoding correct (no garbled characters)

---

## 9. Sources

1. [Microsoft Learn — Metadata (Win32 apps)](https://learn.microsoft.com/en-us/windows/win32/wmformat/metadata)
2. [Microsoft Docs — WM/ attributes for music files](https://github.com/MicrosoftDocs/win32/blob/docs/desktop-src/wmformat/attributes-for-music-files.md)
3. [Microsoft Learn — Metadata Properties for Media Files](https://learn.microsoft.com/en-us/windows/win32/medfound/metadata-properties-for-media-files)
4. [Microsoft Learn — ID3 Tag Support (ASF)](https://learn.microsoft.com/en-us/windows/win32/wmformat/id3-tag-support)
5. [GitHub — wmsdkidl.idl attribute definitions](https://github.com/cwi-dis/grins/blob/master/mm/win32/wmsdk/wmfsdk/include/wmsdkidl.idl)
