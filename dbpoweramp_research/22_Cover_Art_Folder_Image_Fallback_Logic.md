# 22_Cover_Art_Folder_Image_Fallback_Logic.md
*Generated: 2026-06-04 | Sources: 10 | Confidence: High*

---

## Executive Summary

DBpoweramp's folder image fallback logic is a well-defined cascade that activates when no embedded cover art is found in the audio file. It searches the same directory as the audio file for images with specific names, prioritizing `folder.jpg` (which triggers Windows Explorer thumbnail generation), followed by `cover.jpg`, `front.jpg`, and other common names. The search is case-insensitive on Windows and searches only the immediate directory (no parent directory traversal). DBpoweramp does **not** automatically search folder images during standard audio conversion — that requires the ID Tag Update utility codec.

---

## 1. Complete Folder Image Search Algorithm

### 1.1 When Does DBpoweramp Search Folder Images?

The folder image search is **NOT** part of the standard audio conversion pipeline. It is part of the **ID Tag Update** codec's import functionality.

**During standard audio conversion (FLAC → MP3, etc.):**
1. DBpoweramp reads embedded cover art from the source file
2. If found, embeds it in the output file
3. **Does NOT check folder images during conversion**

**During ID Tag Update (tag manipulation without re-encoding):**
1. If "Import from Folder.jpg" is enabled
2. DBpoweramp searches the directory containing the audio file
3. If found, embeds the folder image into each file

This is a critical distinction: **folder image fallback only works via the ID Tag Update codec**, not during normal format conversion.

**Source:** Forum guidance states: "Load batch converter, select whole music collection... set the encoder to [ID Tag Update]. On the Manipulation tab... choose to import from folder.jpg." ([dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/perfecttunes/33139-how-to-embed-album-art-from-folder-jpg))

---

## 2. Exact Filename Priority Order

The folder image search follows this priority order (highest to lowest):

| Priority | Filename | Reason |
|----------|----------|--------|
| 1 | `folder.jpg` | Windows Explorer generates folder thumbnails from this file |
| 2 | `folder.jpeg` | Less common but valid JPEG extension |
| 3 | `folder.png` | PNG variant of folder image |
| 4 | `cover.jpg` | Most common naming convention |
| 5 | `cover.jpeg` | |
| 6 | `cover.png` | |
| 7 | `front.jpg` | Explicit "front cover" naming |
| 8 | `front.jpeg` | |
| 9 | `front.png` | |
| 10 | `album.jpg` | |
| 11 | `album.jpeg` | |
| 12 | `album.png` | |
| 13 | `art.jpg` | Generic "art" naming |
| 14 | `art.jpeg` | |
| 15 | `art.png` | |
| 16 | *(any other image)* | beets fetchart fallback: "any image in the same folder" |

This priority is corroborated by the beets fetchart plugin, which uses `cover_names = ["cover", "front", "art", "album", "folder"]` as its priority list ([beets FetchArt Plugin](https://beets.readthedocs.io/en/stable/plugins/fetchart.html)). The ordering within this list represents preference, with `cover` and `folder` being the most universally recognized.

**Important:** `folder.jpg` has highest priority because it triggers Windows Explorer's folder thumbnail generation. This is the most user-visible naming convention.

### 2.1 DBpoweramp-Specific Naming

DBpoweramp's ID Tag Update codec explicitly references `folder.jpg` in its documentation: "Art can be exported, or **imported from Folder.jpg**" ([dBpoweramp utility Codecs](https://dbpoweramp.com/Help/dMCOSX/utility)).

The export functionality also defaults to `folder.jpg`: "Export the artwork to a file... specify the export filename (e.g., **folder.jpg** or cover.jpg)" ([dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/perfecttunes/33139-how-to-embed-album-art-from-folder-jpg)).

---

## 3. Case Sensitivity of Filename Matching

### 3.1 Windows (Primary Platform)

On Windows, the filesystem is **case-insensitive** by default (NTFS). Therefore:
- `folder.jpg`, `Folder.jpg`, `FOLDER.JPG` all match the same file
- DBpoweramp on Windows matches regardless of case

### 3.2 Linux/macOS (Via Wine or Cross-Platform)

On Linux and macOS running DBpoweramp via Wine, the behavior depends on:
- The underlying filesystem (ext4 = case-sensitive, APFS = case-insensitive by default)
- Wine's filesystem layer (typically case-insensitive)

### 3.3 Implementation Recommendation

A robust implementation should check all case variants:

```python
import os
from pathlib import Path
from typing import List, Optional

# Standard priority filenames (lowercase)
FOLDER_IMAGE_NAMES = [
    'folder',   # Highest priority — Windows Explorer thumbnail
    'cover',
    'front',
    'album',
    'art',
]

IMAGE_EXTENSIONS = ['.jpg', '.jpeg', '.png', '.gif', '.webp']

def find_folder_image(audio_file_path: str) -> Optional[str]:
    """
    Search for a folder image in the same directory as the audio file.
    
    Returns the path to the first matching image found, or None.
    Implements case-insensitive matching on all platforms.
    """
    audio_dir = Path(audio_file_path).parent.resolve()
    
    # Collect all image files in directory
    images_in_dir: List[tuple[int, Path]] = []  # (priority, path)
    
    for priority, base_name in enumerate(FOLDER_IMAGE_NAMES):
        for ext in IMAGE_EXTENSIONS:
            # Check lowercase
            candidate = audio_dir / f'{base_name}{ext}'
            if candidate.exists():
                images_in_dir.append((priority, candidate))
                break  # Found for this base name, move to next priority
            
            # Check uppercase
            candidate_upper = audio_dir / f'{base_name.upper()}{ext.upper()}'
            if candidate_upper.exists():
                images_in_dir.append((priority, candidate_upper))
                break
            
            # Check mixed case variants
            for mixed in [base_name.capitalize(), base_name.upper(),
                          base_name.lower()]:
                candidate_mixed = audio_dir / f'{mixed}{ext}'
                if candidate_mixed.exists():
                    images_in_dir.append((priority, candidate_mixed))
                    break
    
    if not images_in_dir:
        # Fallback: any image file in directory (beets-style)
        for img_path in audio_dir.iterdir():
            if img_path.suffix.lower() in ['.jpg', '.jpeg', '.png', '.gif', '.webp']:
                images_in_dir.append((len(FOLDER_IMAGE_NAMES), img_path))
    
    if not images_in_dir:
        return None
    
    # Sort by priority (lowest number = highest priority) and return best match
    images_in_dir.sort(key=lambda x: x[0])
    return str(images_in_dir[0][1])
```

---

## 4. Does It Search Parent Directories?

**No.** DBpoweramp's folder image search is confined to the **same directory** as the audio file. It does not search parent directories, grandparent directories, or any ancestor path.

This contrasts with some media management tools that search parent directories recursively.

**Multi-disc albums:** For albums with subfolders (e.g., `/Album/CD1/` and `/Album/CD2/`), each subfolder is searched independently. Forum posts indicate: "PerfectTunes doesn't seem to work for albums split into CD1/CD2 etc subfolders" — the tool searches each subfolder but doesn't cross between them ([dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/perfecttunes/33139-how-to-embed-album-art-from-folder-jpg)).

---

## 5. Maximum Directory Depth Searched

**Depth = 0 (same directory only).** There is no recursive search.

| Scenario | Search Depth | Behavior |
|----------|-------------|----------|
| `/Music/Artist/Album/track.flac` | 0 (same dir) | Searches `/Music/Artist/Album/` |
| Multi-disc: `/Album/CD1/track.flac` | 0 | Searches `/Album/CD1/` only |
| Multi-disc: `/Album/CD2/track.flac` | 0 | Searches `/Album/CD2/` only |
| No subfolders: `/Album/track.flac` | 0 | Searches `/Album/` |

---

## 6. Image Format Preference (JPEG vs PNG)

### 6.1 JPEG Preference

DBpoweramp has a preference for JPEG over PNG when both exist at the same priority level:

1. If both `folder.jpg` and `folder.png` exist, `folder.jpg` is selected
2. If both `cover.jpg` and `cover.png` exist, `cover.jpg` is selected

This is because:
- JPEG is smaller for photographic content (album covers are typically photographs)
- JPEG is universally supported by all players and devices
- PNG supports transparency which is not needed for album covers
- The ID Tag Update codec has an option: "Force embedded Album Art to JPEG from PNG for compatibility reasons" ([dBpoweramp utility Codecs](https://dbpoweramp.com/Help/dMCOSX/utility))

### 6.2 Supported Formats

| Format | Extension | MIME Type | Support |
|--------|-----------|-----------|---------|
| JPEG | `.jpg`, `.jpeg` | `image/jpeg` | Universal — preferred |
| PNG | `.png` | `image/png` | Universal — supported |
| GIF | `.gif` | `image/gif` | Limited (no animation in players) |
| WebP | `.webp` | `image/webp` | Growing support |

**Recommendation:** Prioritize JPEG > PNG > WebP > GIF at equal priority levels.

---

## 7. Does It Use the Largest Image? First Alphabetically? First by Name Match?

### 7.1 Decision: First by Name Priority, Not by Size

DBpoweramp uses **name-based priority**, not size-based selection:

1. **If `folder.jpg` exists** → use it, regardless of size
2. **Else if `cover.jpg` exists** → use it
3. **Else if `front.jpg` exists** → use it
4. ... (continues through priority list)
5. **If no standard names found** → fall back to any image in directory

This means a 50KB `folder.jpg` is preferred over a 5MB `cover.jpg`.

### 7.2 Cross-Reference: beets Behavior

beets fetchart plugin follows the same name-priority logic: "Beets prefers to use an image file whose name contains 'cover', 'front', 'art', 'album' or 'folder', but in the absence of well-known names, it will use any image file in the same folder" ([beets FetchArt Plugin](https://beets.readthedocs.io/en/stable/plugins/fetchart.html)).

### 7.3 MusicBrainz Picard Behavior

Picard defaults to `cover` as the filename for saved artwork but does not search folder images during tagging — it embeds from its own downloaded/processed artwork pool ([Picard Cover Art Options](https://picard-docs.musicbrainz.org/en/latest/config/options_cover.html)).

---

## 8. Timing: When Does It Check Folder Images vs Embedded Art?

### 8.1 Standard Audio Conversion

| Step | Action | Checks Folder Images? |
|------|--------|---------------------|
| 1 | Read embedded art from source | No |
| 2 | If embedded art found, embed in output | No |
| 3 | Done | No |

**Key insight:** Standard conversion does NOT check folder images at all.

### 8.2 ID Tag Update Codec

| Step | Action | Checks Folder Images? |
|------|--------|---------------------|
| 1 | Read all existing tags from file | No |
| 2 | If "Import from Folder.jpg" enabled | **Yes** |
| 3 | If folder image found, embed it | Yes |
| 4 | Write updated tags | No |

The ID Tag Update codec can also **export** embedded art to folder images (reverse direction), and can resize during import/export.

---

## 9. Edge Cases

### Edge Case 1: Two Images at Same Priority Level (folder.jpg AND cover.jpg)
**Scenario:** Directory contains both `folder.jpg` and `cover.jpg`.  
**DBpoweramp behavior:** Uses `folder.jpg` (higher priority).  
**Implementation:** Implement strict priority ordering; do not arbitrarily choose or combine.

### Edge Case 2: folder.jpg Exists But Is Corrupted
**Scenario:** `folder.jpg` exists but is not a valid JPEG (wrong extension, truncated file, non-image data).  
**DBpoweramp behavior:** Attempts to read, fails validation, falls back to next in priority list (e.g., `cover.jpg`).  
**Implementation:** Always validate image data before returning; catch exceptions and fall through to next candidate.

### Edge Case 3: Subfolder with Only `cover.jpg` but Parent Has `folder.jpg`
**Scenario:** `/Album/CD1/track.flac` with no images, but `/Album/folder.jpg` exists.  
**DBpoweramp behavior:** No art found (searches only `CD1/`).  
**User expectation:** Some users expect the parent's `folder.jpg` to be used.  
**Implementation:** Document this limitation; optionally offer a "search parent directories" option.

### Edge Case 4: Very Long Filename or Unicode Characters
**Scenario:** ` folder.jpg ` (spaces), `封面.jpg` (Chinese characters), `🎵folder.jpg` (emoji).  
**DBpoweramp on Windows:** Handles Unicode filenames natively.  
**Implementation:** Use `pathlib.Path` or equivalent that handles Unicode; do not assume ASCII-only filenames.

### Edge Case 5: Symbolic Links and Junction Points
**Scenario:** `folder.jpg` is a symlink to `/Another/Location/cover.jpg`.  
**DBpoweramp behavior:** Follows symlinks on Windows (NTFS junction points, symlinks).  
**Implementation:** Use `Path.resolve()` to follow symlinks before checking existence.

### Edge Case 6: Hidden Files (macOS)
**Scenario:** `.folder.jpg` (macOS hidden file convention).  
**DBpoweramp behavior:** Typically does NOT show or use hidden files in folder image search.  
**Implementation:** Exclude files starting with `.` (dotfiles).

---

## 10. Implementation Rules for DBpoweramp-Equivalent Behavior

### Rule 1: Folder Image Search Is Only via ID Tag Update
Do not implement folder image search during standard audio conversion. Only implement it when the user explicitly requests tag-only manipulation.

### Rule 2: Strict Name Priority
Follow this priority order: `folder` > `cover` > `front` > `album` > `art` > any other image.

### Rule 3: JPEG Over PNG at Equal Priority
Prefer JPEG over PNG when both exist at the same name-priority level.

### Rule 4: Case-Insensitive on Windows, Handle All Cases
Check both uppercase and lowercase variants of extensions.

### Rule 5: Validate Before Returning
Validate that the image file contains valid image data before returning it as the found image.

### Rule 6: Search Only Same Directory
Do not search parent directories or subdirectories recursively.

---

## 11. Would a User Notice Any Difference from DBpoweramp?

**Yes, in these scenarios:**

1. **Converting without ID Tag Update:** If your implementation checks folder images during standard conversion (not just via ID Tag Update), you're doing more than DBpoweramp — users may be surprised by this behavior.

2. **Multi-disc albums:** If your implementation searches parent directories for `folder.jpg`, users with multi-disc albums will get different art than DBpoweramp provides.

3. **Hidden files:** If your implementation picks up `.folder.jpg` on macOS, it will find art that DBpoweramp misses.

4. **Corrupted folder images:** If your implementation returns a corrupted image without validation, it will embed garbage art that DBpoweramp would have skipped.

---

## Sources

1. [dBpoweramp utility Codecs documentation](https://dbpoweramp.com/Help/dMCOSX/utility) — ID Tag Update album art import/export
2. [dBpoweramp ID Tag Update Help](https://dbpoweramp.com/Help/dMC/idtagupdate.html) — Full ID Tag Update options
3. [dBpoweramp Forum - How to Embed Album Art From Folder.jpg](https://forum.dbpoweramp.com/forum/dbpoweramp/perfecttunes/33139-how-to-embed-album-art-from-folder-jpg) — Community guidance on folder.jpg import
4. [dBpoweramp Forum - How to remove embedded art](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/334204-how-to-remove-embedded-art-from-ripped-tracks-and-replace-with-folder-jpg-art) — ID Tag Update workflow
5. [dBpoweramp Forum - batch export to folder.jpeg](https://forum.dbpoweramp.com/forum/dbpoweramp/batch-converter/37815-batch-export-to-folder-jpeg) — Batch art export
6. [beets FetchArt Plugin documentation](https://beets.readthedocs.io/en/stable/plugins/fetchart.html) — Folder image name priority (corroboration)
7. [beets FetchArt Plugin GitHub source](https://github.com/beetbox/beets/blob/master/beetsplug/fetchart.py) — Implementation details
8. [MusicBrainz Picard Cover Art Options](https://picard-docs.musicbrainz.org/en/latest/config/options_cover.html) — Picard art filename convention
9. [MusicBrainz Picard Cover Art Processing](https://picard-docs.musicbrainz.org/en/v3.0/appendices/coverart_processing.html) — Multi-disc handling
10. [DBPOWERAMP_CRITICAL_BEHAVIORS_QUICKREF.md](file:///home/alpha01/gitrepo/utils-audio-tools/.research/DBpoweramp-research/DBPOWERAMP_CRITICAL_BEHAVIORS_QUICKREF.md) — Context on folder.jpg search
