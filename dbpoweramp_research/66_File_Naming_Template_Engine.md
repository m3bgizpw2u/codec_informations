# File Naming Template Engine
*Generated: 2026-06-04 | Sources: 8 | Confidence: High*

## Executive Summary

DBpoweramp's Dynamic Naming feature uses a powerful template engine that constructs output file paths and names from metadata tags using square bracket syntax. The template system supports standard tags, zero-padding for track numbers, conditional expressions (IFCOMP, IFVALUE, IFMULTI), character transformations (UPPER, LOWER), string manipulation (RIGHT, GRAB, MAXLENGTH), and path trimming. Files are organized into folder hierarchies with illegal characters automatically replaced. The Reference version enables programmable naming with advanced logic.

## 1. Template Syntax Overview

### 1.1 Basic Syntax
Templates use `[TAG]` syntax to insert metadata values:
```
[artist]\[album]\[track] - [title]
```

This produces: `Artist Name/Album Title/01 - Track Title.ext`

### 1.2 Standard Tags
| Tag | Description |
|-----|-------------|
| `[artist]` | Track artist |
| `[album]` | Album title |
| `[album artist]` | Album artist (if different) |
| `[title]` | Track title |
| `[track]` | Track number |
| `[disc]` | Disc number |
| `[year]` | Release year |
| `[genre]` | Genre |
| `[composer]` | Composer |
| `[comment]` | File comment |
| `[origfilename]` | Original source filename |
| `[origpath]` | Full source path |
| `[origdrive]` | Source drive letter |

### 1.3 Numeric Formatting
| Tag | Format | Example |
|-----|--------|---------|
| `[track]` | Single digit | `1, 2, 3...` |
| `[track,2]` | Zero-padded (2 digits) | `01, 02, 03...` |
| `[track,3]` | Zero-padded (3 digits) | `001, 002, 003...` |
| `[disc]` | Single digit | `1, 2, 3...` |
| `[disc,2]` | Zero-padded | `01, 02, 03...` |

## 2. Conditional Expressions

### 2.1 IFCOMP - Compilation Check
Includes content if album is marked as compilation:
```
[IFCOMP]string[]
```
Example:
```
[IFCOMP]Various Artists[]
```
### 2.2 IF!COMP - Not Compilation
Includes content if NOT a compilation:
```
[IF!COMP]string[]
```
Example:
```
[IF!COMP][artist][]
```

### 2.3 IFMULTI - Multi-Disc Check
Includes content if part of multi-disc set:
```
[IFMULTI]string[]
```
Example:
```
[IFMULTI] Disc [disc][]
```
Produces: ` Disc 1`, ` Disc 2`, etc. only for multi-disc albums.

### 2.4 IF!MULTI - Not Multi-Disc
Includes content if NOT part of multi-disc:
```
[IF!MULTI]string[]
```

### 2.5 IFVALUE - Tag Value Check
Includes content if tag has a value:
```
[IFVALUE]tagname,value_if_exists,value_if_empty[]
```
Example:
```
[IFVALUE]album artist,[album artist],[artist][]
```
This uses album artist if present, otherwise falls back to artist.

## 3. String Manipulation

### 3.1 UPPER - Uppercase
Converts string to uppercase:
```
[UPPER]string[]
```
Example: `[UPPER][artist][]` → `ARTIST NAME`

### 3.2 LOWER - Lowercase
Converts string to lowercase:
```
[LOWER]string[]
```

### 3.3 RIGHT - Right Substring
Extracts last N characters:
```
[RIGHT]count,string[]
```
Example: `[RIGHT]3,abcdefg[]` → `efg`

### 3.4 GRAB - Extract Substring
Extracts portion of string:
```
[GRAB]from,to,string[]
```
- `from`: Starting position (0-indexed)
- `to`: Ending position
- If `to` omitted: extracts to end

### 3.5 MAXLENGTH - Truncate
Limits string length:
```
[MAXLENGTH]max_length,string[]
```
Example: `[MAXLENGTH]50,[title][]` → Title truncated to 50 chars

## 4. Multi-Artist Handling

### 4.1 Standard Multi-Artist
Only first artist in multi-artist field is used by default:
- `[album artist]`: First album artist
- `[artist]`: First track artist

### 4.2 Multi-Tag for All Artists
Use `[multitag]` to get all artists:
```
[multitag]album artist[]
```
Produces: `Artist1; Artist2; Artist3`

### 4.3 Custom Multi-Artist Separator
Default separator is semicolon:
- Configurable in PerfectMeta options
- Used for compilations and collaborations

## 5. Path and Folder Templates

### 5.1 Folder Separator
Use backslash `\` to create folders:
```
[artist]\[album]\[track] - [title]
```
Result: `Artist/Album/01 - Title.ext`

### 5.2 Nested Conditionals
Conditionals can be nested for complex logic:
```
[IFCOMP]
  [IFVALUE]album artist,[album artist],Various Artists[]
  \[album]
  [IFMULTI] Disc [disc][]
  \[track] - [title]
[]
[IF!COMP]
  [IFVALUE]album artist,[album artist],[artist][]
  \[album]
  [IFMULTI] Disc [disc][]
  \[track] - [title]
[]
```

### 5.3 Output to Single Folder
To put all files in one folder:
```
My Music\[origfilename]
```

## 6. Illegal Character Handling

### 6.1 Windows Filename Restrictions
Windows forbids:
```
\ / : * ? " < > |
```

### 6.2 DBpoweramp's Replacement
DBpoweramp automatically replaces illegal characters:
- `/` → `_` (or configurable)
- `:` → `_`
- `*` → `_`
- `?` → `_`
- `"` → `_`
- `<` → `_`
- `>` → `_`
- `|` → `_`

### 6.3 Additional Restrictions
Configurable in Configuration:
- Leading/trailing spaces removed
- Unicode spaces converted to standard spaces
- Other OS-specific restrictions handled

## 7. Unicode and Special Characters

### 7.1 Unicode Support
DBpoweramp supports Unicode in filenames:
- Full UTF-8 character support
- Japanese, Chinese, Korean characters
- Cyrillic, Greek, Arabic
- Emoji (in supported file systems)

### 7.2 Compatibility
- NTFS: Full Unicode support
- exFAT: Full Unicode support
- FAT32: Limited Unicode
- Network shares: May have restrictions

### 7.3 Platform Differences
| Platform | Unicode Support |
|----------|----------------|
| Windows 10/11 | Full |
| macOS | Full |
| Linux | Full (取决于文件系统) |
| Network shares | Varies |

## 8. MAX_PATH on Windows

### 8.1 What is MAX_PATH?
Windows traditionally limits paths to 260 characters:
- `C:\path\to\file` format
- Includes drive, folders, filename, extension
- Registry setting to increase to ~32,767

### 8.2 DBpoweramp Handling
DBpoweramp can handle longer paths:
- Uses extended path syntax: `\\?\` prefix
- Long path support enabled by default on Windows 10+
- MAXLENGTH can reduce path component lengths

### 8.3 Recommendations
For maximum compatibility:
- Keep paths under 200 characters
- Use MAXLENGTH to truncate long album/titles
- Avoid deeply nested folder structures

## 9. Technical Tag Variables

### 9.1 Audio Properties
| Tag | Description | Example |
|-----|-------------|---------|
| `[bitrate]` | Audio bitrate | `320` |
| `[samplerate]` | Sample rate | `44100` |
| `[bit depth]` | Bit depth | `16` |
| `[channels]` | Channel count | `2` |
| `[codec]` | Codec name | `FLAC` |

### 9.2 Date/Time Tags
| Tag | Description |
|-----|-------------|
| `[year]` | 4-digit year |
| `[month]` | Month (01-12) |
| `[day]` | Day (01-31) |
| `[date]` | Full date (varies by format) |

### 9.3 Source File Tags
| Tag | Description |
|-----|-------------|
| `[origfilename]` | Original filename (no extension) |
| `[origpath]` | Full original path |
| `[origdrive]` | Source drive |
| `[subpath]` | Path after library location |

## 10. Edge Cases

### 10.1 Missing Tags
When a tag is empty:
- Use IFVALUE to provide fallback
- Leave blank (may create empty folder levels)
- Use DEFAULT tag if available

### 10.2 Very Long Values
Long album titles or artist names:
- Use MAXLENGTH to cap
- Truncation may lose meaning
- Consider path length limits

### 10.3 Special Characters in Tags
Characters from CD-TEXT or metadata:
- May contain unusual punctuation
- Automatically cleaned for filenames
- Check resulting filenames

### 10.4 Disc Number Variations
Multi-disc naming:
```
[IFMULTI][disc].[][track][] vs [IFMULTI] - Disc [disc][]
```
First: `1.01`, `2.03`
Second: ` - Disc 1`, ` - Disc 2`

### 10.5 Compilation vs Regular
Distinguish with conditionals:
```
[IFCOMP]Compilations\[album][][IF!COMP][artist]\[album][]
```

### 10.6 Year in Album Title
Some albums include year:
```
[artist] - [album] ([year])
```
Produces: `Artist - Album Title (2023)`

### 10.7 Multi-Value Fields
When multiple values exist:
- First value used by default
- Use [multitag] for all values
- Separator configurable

## 11. Would a User Notice a Difference?

### From Different Template Patterns

| Pattern | User Impact |
|---------|-------------|
| Proper artist/album folders | Easy browsing |
| Track number prefixing | Correct sort order |
| Zero-padding | Sortable numerical order |
| Multi-disc handling | Proper album grouping |

### From No Dynamic Naming
| Aspect | Impact |
|--------|--------|
| File organization | Major (random vs organized) |
| Search capability | Easier finding |
| Media server compatibility | Better indexing |
| Backup/library management | Improved structure |

### From DBpoweramp to Other Tools

| Feature | DBpoweramp | Other Tools |
|---------|-----------|-------------|
| Conditional logic | Full | Limited |
| String manipulation | Extensive | Basic |
| Multi-disc handling | Native | Manual |
| Path length handling | Robust | May fail |

## Sources

1. [dBpoweramp Naming Help](https://www.dbpoweramp.com/help/dmc/Naming)
2. [dBpoweramp Naming - macOS](https://www.dbpoweramp.com/Help/dMIOSX/Naming)
3. [Naming Scheme Tutorial - dBpoweramp Forum](https://forum.dbpoweramp.com/forum/read-only/hints-tips-read-only/14096-naming-scheme-tutorial)
4. [Custom Naming - dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/41110-custom-naming)
5. [Help with File Naming - dBpoweramp Forum](https://forum.dbpoweramp.com/forum/other-topics/test-upcoming-releases/11858-help-with-understanding-file-naming)
6. [TuneFUSION Naming Help](https://www.dbpoweramp.com/Help/TuneFUSION/Naming)
7. [dBpoweramp Configuration](https://dbpoweramp.com/Help/dMC/dMCConfig)
8. [dBpoweramp Multi Encoder Help](https://dbpoweramp.com/Help/dMC/multiencoder)
