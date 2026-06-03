# Metadata: Vorbis Comments — Deep Technical Reference

> **Category:** Metadata / Tagging System
> **File Extensions:** .ogg, .flac, .opus, .oga, .webm (via WebM container)
> **MIME Types:** N/A (metadata system shared across multiple formats)
> **Standardization Body:** Xiph.Org Foundation (informal standard)
> **Specification Document:** https://xiph.org/vorbis/doc/v-comment.html, Ogg Vorbis I spec (RFC 5215)
> **Patent Status:** Patent-free
> **License:** BSD-style (libvorbis); public domain-equivalent for spec

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation and Design Philosophy

Vorbis Comments (commonly known as "Vorbis tags" or "Ogg tags") emerged from the Xiph.Org Foundation's development of the Ogg Vorbis audio codec in the late 1990s. The specification was created by Chris Montgomery (Xiphmont) and the Xiph.Org team as part of the Ogg project, which aimed to create a free, open-source multimedia container and codec infrastructure free from patent encumbrances.

The design philosophy behind Vorbis Comments was fundamentally different from the ID3 tagging approach used in MP3 files. Where ID3 was bolted onto MP3 as an afterthought (first as a 128-byte footer, then as a prepend), Vorbis Comments were designed from the beginning as an integral part of the Vorbis bitstream specification. The comment header is defined as the third logical packet in an Ogg Vorbis stream, immediately following the identification header and setup header. This tight integration meant that metadata was a first-class citizen, not an afterthought.

The Vorbis Comment specification was formalized in RFC 5215, "Ogg Vorbis Media Type Registration," which was published as an informational RFC by IANA in September 2007. This RFC codified the Vorbis Comment structure as the canonical metadata format for Vorbis audio streams.

### 1.2 Cross-Format Adoption

What makes Vorbis Comments particularly significant in the audio codec ecosystem is their adoption beyond Ogg Vorbis. The Xiph.Org team designed the comment format to be codec-agnostic, and it was subsequently adopted as the metadata format for several other important audio formats:

**FLAC (Free Lossless Audio Codec):** FLAC adopted Vorbis Comments as its native tagging system starting with FLAC 1.1.0 (released 2004). Prior to this, FLAC had no official tagging mechanism. The Vorbis Comment format was chosen over ID3v2 because it was cleaner, more extensible, and aligned with FLAC's philosophy of an open, patent-free format. FLAC files can also embed cover art using a specially defined METADATA_BLOCK_PICTURE structure.

**Opus Audio Codec:** The Opus codec (RFC 6716), developed jointly by Xiph.Org, Microsoft, and Google, uses Vorbis Comments for metadata in Ogg Opus containers. The Opus specification explicitly references the Vorbis Comment format for tagging.

**WebM Container:** WebM (based on Matroska) supports Vorbis Comments for audio tracks. While WebM is primarily a video container, audio-only WebM files can use Vorbis Comments for metadata.

**Speex:** The Speex speech codec used Vorbis Comments in its Ogg container format.

This cross-format adoption means that understanding Vorbis Comments is essential for anyone working with open-source audio codecs, as it serves as a lingua franca for metadata across the Xiph.Org ecosystem and beyond.

### 1.3 Design Principles

The Vorbis Comment design was guided by several key principles:

**Simplicity:** The format uses a straightforward binary layout with length-prefixed strings. There are no complex nested structures, no variable-length integer encodings, and no framing ambiguities.

**Extensibility:** Any field name not defined in the specification is a valid user-defined field. There is no approval process needed for new field names.

**UTF-8 everywhere:** All strings in Vorbis Comments are encoded in UTF-8. There is no encoding negotiation, no BOM, and no ambiguity about byte ordering.

**No padding:** Vorbis Comments contain no padding bytes. The length of each field and the total comment header size are computed from the actual data, eliminating ambiguity about what is data versus padding.

**Vendor string identification:** The comment header begins with a vendor string that identifies the encoder software, enabling consumers of the metadata to understand which tool wrote the tags.

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Position in Container Hierarchy

In an Ogg Vorbis stream, the bitstream structure consists of three mandatory packets at the beginning:

```
[Ogg Page 1]  Identification Header (vorbis)
[Ogg Page 2]  Comment Header (vorbis-comment)
[Ogg Page 3]  Setup Header (vorbis)
[Ogg Page N]  Audio Data Pages (vorbis)
```

The Comment Header is the second logical packet and must appear on the second Ogg page. This ordering is enforced by the Ogg chaining rules: the identification header initializes the decoder, the comment header provides metadata, and the setup header provides the codec setup data. Audio data cannot appear until all three headers have been received.

In FLAC, the Vorbis Comment block appears as metadata block type 4 in the FLAC metadata header chain, immediately after the streaminfo block (type 0). FLAC reserves metadata block type 4 specifically for Vorbis Comments.

In Ogg Opus, the comment header appears as the second packet in the Ogg stream, following the OpusHead identification packet.

### 2.2 Two-Tier Structure

The Vorbis Comment format has a deliberately simple two-tier structure:

**Tier 1 — Comment Header:** A single block containing the vendor string and the count of comment fields. This is written once per stream.

**Tier 2 — User Comment List:** A flat list of `KEY=VALUE` strings. Each string is an independent, self-contained field with no hierarchy or grouping.

This flat list approach is both a strength and a limitation. It is a strength because it is extremely simple to parse and write: each field is a length-prefixed string, and the only structural information needed is the vendor length, vendor string, and comment count. It is a limitation because there is no native support for grouped or hierarchical metadata, multi-value fields require conventions (like `ARTIST=` appearing multiple times), and there is no mechanism for field ordering preservation across edits.

### 2.3 Codec-Specific Extensions

While the core Vorbis Comment specification is shared, individual codecs add specific extensions:

**FLAC:** FLAC adds the METADATA_BLOCK_PICTURE structure (block type 6) for embedded cover art. This is not part of the Vorbis Comment specification but is typically used alongside Vorbis Comments in FLAC files.

**Opus:** Opus adds the R128_TRACK_GAIN and R128_TRACK_PEAK fields following EBU R128 methodology, in addition to standard ReplayGain fields.

**WebM (Matroska):** WebM uses a Matroska-derived container that maps Vorbis Comment fields to Matroska tag elements. The mapping is generally one-to-one, but Matroska's more complex tag structure allows for additional features like per-language tags.

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 Comment Header Structure

The complete comment header follows this byte-level layout:

```
[Vendor Length:  4 bytes, little-endian uint32]
[Vendor String:  N bytes, UTF-8 encoded text]
[Comment Count:  4 bytes, little-endian uint32]
[Comment 1:      4 bytes (length) + M bytes (UTF-8 string "KEY=VALUE")]
[Comment 2:      4 bytes (length) + M bytes (UTF-8 string)]
...
[Comment N:      4 bytes (length) + M bytes (UTF-8 string)]
```

All multi-byte integers in Vorbis Comments use **little-endian byte order**. This is consistent with the Vorbis codec's use of little-endian encoding throughout. The 4-byte length field preceding each comment string encodes the length of the UTF-8 string in bytes, not the number of characters.

### 3.2 Complete Byte Layout Diagram

For a comment header with vendor string "Xiph.Org libVorbis I 20030909" and two comment fields:

```
Offset  Size  Field                    Value (hex / ASCII)
------  ----  -----                    --------------------
0x0000  4     Vendor Length (LE)       24 00 00 00  (decimal 36)
0x0004  36    Vendor String            "Xiph.Org libVorbis I 20030909"
0x0028  4     Comment Count (LE)       02 00 00 00  (decimal 2)
0x002C  4     Field 1 Length           0B 00 00 00  (decimal 11)
0x0030  11    Field 1 String           "TITLE=Hello"
0x003B  4     Field 2 Length           0F 00 00 00  (decimal 15)
0x003F  15    Field 2 String           "ARTIST=World"
```

### 3.3 Field String Format

Each comment field is a UTF-8 encoded string containing exactly one `=` character that separates the field name from the field value:

```
FIELD_NAME = FIELD_VALUE
```

**Field name rules:**
- Case-insensitive for comparison (the specification states implementations should treat them as case-insensitive, though storage preserves the original case)
- ASCII letters, digits, and underscores only (pattern: `[A-Za-z0-9_]+`)
- No leading or trailing whitespace around the field name
- No leading or trailing whitespace around the `=` character
- Field names starting with `=` or consisting only of whitespace are invalid
- Field names longer than 255 bytes are considered invalid

**Field value rules:**
- May contain any valid UTF-8 sequence
- May be empty (valid: `FIELD=` — empty value)
- May contain literal `=` characters (the first `=` is the delimiter; subsequent `=` characters are part of the value)
- May contain newlines, tabs, and other whitespace characters
- Leading and trailing whitespace in the value is preserved (no automatic trimming)

### 3.4 Complete Field Name Registry

The following field names are defined in the Vorbis Comment specification:

| Field Name | Description | Multi-instance |
|---|---|---|
| TITLE | Track title | No |
| VERSION | Track version (e.g., "Remastered", "Radio Edit") | No |
| ALBUM | Album name | No |
| ALBUMARTIST | Album artist / collective name | Yes |
| ARTIST | Primary artist name | Yes |
| COMPOSER | Music composer | Yes |
| PERFORMER | Performer credit (distinct from artist) | Yes |
| DATE | Recording date (YYYY or YYYY-MM-DD) | No |
| ORIGINALDATE | Original recording date | No |
| ORIGINALALBUM | Original album name | No |
| ORIGINALARTIST | Original artist for covers/remakes | Yes |
| LABEL | Record label | No |
| CATALOGNUMBER | Label catalog number | Yes |
| BARCODE | UPC/EAN barcode | No |
| ISRC | International Standard Recording Code | Yes |
| PUBLISHER | Publisher name (synonym for LABEL) | No |
| COPYRIGHT | Copyright notice | No |
| LICENSE | License URI | No |
| LOCATION | Recording location | No |
| ORGANIZATION | Organization name | No |
| GENRE | Genre classification | Yes |
| DESCRIPTION | General description | No |
| TRACKNUMBER | Track number within the album | No |
| TRACKTOTAL | Total number of tracks in the album | No |
| DISCNUMBER | Disc number within a multi-disc set | No |
| DISCTOTAL | Total number of discs | No |
| COMMENT | General comment | Yes |
| COMMENTURL | URL for comments/discussion | No |
| BUYURL | URL to purchase this track | No |
| ARTISTURL | URL for the artist | No |
| LABELURL | URL for the record label | No |
| LICENSEURL | URL for the license | No |
| BPM | Beats per minute (integer or decimal) | No |
| KEY | Musical key (e.g., "C#m") | No |
| DIVINE | Disc ID (MusicBrainz) | No |
| MUSICBRAINZ_TRACKID | MusicBrainz recording ID (UUID) | Yes |
| MUSICBRAINZ_ALBUMID | MusicBrainz release ID (UUID) | No |
| MUSICBRAINZ_ARTISTID | MusicBrainz artist ID (UUID) | Yes |
| MUSICBRAINZ_ALBUMARTISTID | MusicBrainz album artist ID (UUID) | Yes |
| MUSICBRAINZ_RELEASEGROUPID | MusicBrainz release group ID (UUID) | No |
| MUSICBRAINZ_WORKID | MusicBrainz work ID (UUID) | No |
| MUSICBRAINZ_TRACKID | MusicBrainz track ID (UUID) | Yes |
| REPLAYGAIN_TRACK_GAIN | ReplayGain track gain adjustment in dB | No |
| REPLAYGAIN_TRACK_PEAK | ReplayGain track peak amplitude | No |
| REPLAYGAIN_ALBUM_GAIN | ReplayGain album gain adjustment in dB | No |
| REPLAYGAIN_ALBUM_PEAK | ReplayGain album peak amplitude | No |

### 3.5 Reserved and Deprecated Fields

Several field names have been used historically but are deprecated or reserved:

| Field Name | Status | Replacement |
|---|---|---|
| COVERART | Deprecated (FLAC-specific) | METADATA_BLOCK_PICTURE |
| COVERARTMIME | Deprecated (FLAC-specific) | Part of METADATA_BLOCK_PICTURE |
| ENCODER | Synonym for vendor string, not a comment field | N/A |
| METADATA_BLOCK_PICTURE | FLAC-specific cover art block | N/A (FLAC-specific) |
| R128_TRACK_GAIN | Opus-specific EBU R128 gain | No |
| R128_TRACK_PEAK | Opus-specific EBU R128 peak | No |

### 3.6 Literal Value Escaping

Values in Vorbis Comments may contain literal characters that need special handling. The escaping rules are:

**The `=` character:** The first `=` in a field string is the delimiter. Any `=` characters after the first are part of the value. A field like `URL=http://example.com/?a=b&c=d` has value `http://example.com/?a=b&c=d`.

**Leading and trailing whitespace in values:** Leading and trailing whitespace in the field value is preserved verbatim. There is no escape mechanism for whitespace.

**Newlines and carriage returns:** A field value may contain literal newline (`0x0A`) or carriage return (`0x0D`) characters. There is no escape mechanism. A single field can span multiple lines if the value contains newline characters.

**Backslashes:** No escape sequence is defined. A literal backslash is simply a backslash character.

**Literal null bytes:** Null bytes (`0x00`) are technically valid in UTF-8 only as part of multi-byte sequences, not as standalone bytes. A literal null byte in a field value would be an invalid UTF-8 sequence and should be rejected by a strict parser.

### 3.7 Bitstream Map — Complete Annotated Layout

```
Byte offset  Field                      Type           Notes
------------  -----                     ----           -----
[0x0000]     Vendor Length             uint32 LE      Bytes 0-3, little-endian
[0x0004]     Vendor String             UTF-8 text     Length = vendor_length bytes
[0x0004+N]   Comment Count             uint32 LE      Bytes 0-3, little-endian
[0x0008+N]   Comment[0] Length         uint32 LE      Length of field string in bytes
[0x000C+N]   Comment[0] String          UTF-8 text     "KEY=VALUE", length from previous
[0x000C+N+M] Comment[1] Length          uint32 LE
[0x0010+N+M] Comment[1] String          UTF-8 text
...          (repeated)
```

Where:
- `N` = vendor_length bytes
- `M` = length of each comment field string in bytes
- Each comment field string is a single `KEY=VALUE` pair encoded as UTF-8

---

## 4. ENCODING / WRITING ALGORITHM

### 4.1 Writing Algorithm Overview

Writing a Vorbis Comment block involves the following sequential steps:

1. Encode the vendor string as UTF-8 and compute its byte length
2. Encode each field as `KEY=VALUE` UTF-8 strings
3. Pack the vendor length as a little-endian uint32
4. Append the vendor string bytes
5. Pack the comment count as a little-endian uint32
6. For each comment field: pack its byte length as a little-endian uint32, then append the UTF-8 string bytes
7. Compute the total comment header size
8. Wrap the comment header in the appropriate container packet

### 4.2 Complete Python Implementation

```python
import struct

def write_vorbis_comment(vendor_string: str, fields: list[tuple[str, str]]) -> bytes:
    """
    Serialize a Vorbis Comment block.
    
    Args:
        vendor_string: Encoder identification string (e.g., "Xiph.Org libVorbis I 20030909")
        fields: List of (field_name, field_value) tuples. Field names are
                treated as case-insensitive by the spec.
    
    Returns:
        Complete comment header bytes suitable for embedding in an Ogg/FLAC container.
    """
    result = bytearray()
    
    # Step 1: Vendor length + vendor string
    vendor_bytes = vendor_string.encode('utf-8')
    result.extend(struct.pack('<I', len(vendor_bytes)))  # little-endian uint32
    result.extend(vendor_bytes)
    
    # Step 2: Comment count
    result.extend(struct.pack('<I', len(fields)))
    
    # Step 3: Each comment field
    for field_name, field_value in fields:
        field_string = f"{field_name}={field_value}"
        field_bytes = field_string.encode('utf-8')
        result.extend(struct.pack('<I', len(field_bytes)))  # length prefix
        result.extend(field_bytes)
    
    return bytes(result)
```

### 4.3 Field Name Canonicalization

The Vorbis Comment specification states that field names are **case-insensitive** for comparison purposes. However, implementations differ in how they handle case when storing and retrieving field names:

**Recommended practice:** Preserve the original case of field names as written by the encoder. When looking up a field by name, perform a case-insensitive comparison. For example, a lookup for `TITLE` should find both `TITLE=Hello` and `Title=Hello`.

**Common implementation:** Many implementations convert field names to uppercase on write (e.g., `Title=Hello` becomes `TITLE=Hello`). This provides a canonical form. FFmpeg, for example, normalizes field names to uppercase when writing Vorbis Comments.

**Field name validation:** A robust writer should reject field names containing characters outside `[A-Za-z0-9_]` or longer than 255 bytes. Empty field names (just `=`) should be rejected.

### 4.4 Value Encoding

All field values are encoded as UTF-8. No other encodings are supported. If the source metadata is in a different encoding (e.g., Latin-1, Windows-1252), it must be transcoded to UTF-8 before writing.

**Handling encoding errors:** If a value contains invalid UTF-8 byte sequences, the writer has three options:

1. Reject the value and report an error
2. Replace invalid bytes with the Unicode replacement character (U+FFFD, encoded as `0xEF 0xBF 0xBD`)
3. Transcode from the source encoding (if known) to UTF-8

Option 2 is the most common in practice because it allows the write operation to succeed even with malformed input.

**Empty values:** The field `FIELD=` (with no value after the `=`) is valid. The empty string is a valid UTF-8 string of length zero.

**Values containing newlines:** If a value contains literal newline characters, it is encoded as-is. The field string will contain the `\n` byte (`0x0A`). This is valid Vorbis Comment data. However, many tag editors and players do not handle multi-line values gracefully, so this should be used sparingly.

### 4.5 Container-Specific Wrapping

The Vorbis Comment bytes produced by the algorithm above must be wrapped in the appropriate container packet:

**Ogg Vorbis:** The comment header bytes are placed in an Ogg packet with packet type `3` (comment) and the packet is placed on the second Ogg page of the stream.

**FLAC:** The comment header bytes are placed in a FLAC metadata block with block type `4`. The FLAC encoder adds a 4-byte block header (1 byte type + 3 bytes length) to produce the complete METADATA_BLOCK_DATA.

**Ogg Opus:** Similar to Ogg Vorbis; the comment header follows the OpusHead packet as the second packet in the stream.

### 4.6 Multi-Value Fields

Some fields conceptually have multiple values (e.g., multiple artists, multiple composers). Vorbis Comments handle this by allowing the same field name to appear multiple times in the comment list:

```
ARTIST=John Doe
ARTIST=Jane Smith
COMPOSER=Bach
COMPOSER=Mozart
```

When reading, all instances of the field should be returned. When writing, the order of multiple values for the same field is generally preserved (the comment list is ordered).

Alternatively, some implementations use a separator convention within a single value:

```
ARTIST=John Doe / Jane Smith
```

The `/` separator is a common convention but is not mandated by the specification. The multi-instance approach (repeated field names) is more robust because it avoids ambiguity with the `/` character appearing in artist names.

### 4.7 Leading Zeros in Track Numbers

Track numbers should use zero-padded strings for consistent sorting. The convention is:

- Single-digit tracks: `01`, `02`, ..., `09`
- Double-digit tracks: `10`, `11`, ..., `99`
- Triple-digit tracks: `100`, `101`, etc.

A two-digit minimum is standard (e.g., `01/12`). The `TRACKTOTAL` or `TOTALTRACKS` field indicates the total count. Some encoders use `TOTALTRACKS` as an alias.

```python
def format_track_number(track: int, total: int | None = None) -> str:
    """Format a track number with leading zeros for sorting."""
    if total is not None:
        width = len(str(total))
        return f"{track:0{width}d}/{total:0{width}d}"
    else:
        return f"{track:02d}"
```

Example outputs: `format_track_number(5, 12)` → `"05/12"`, `format_track_number(5)` → `"05"`.

---

## 5. DECODING / READING ALGORITHM

### 5.1 Reading Algorithm Overview

Reading a Vorbis Comment block follows this procedure:

1. Read the first 4 bytes as a little-endian uint32: vendor length
2. Read exactly that many bytes: vendor string
3. Read the next 4 bytes as a little-endian uint32: comment count
4. For i in range(comment_count):
   - Read 4 bytes as little-endian uint32: field byte length
   - Read exactly that many bytes: field string
   - Parse the field string by splitting on the first `=` character
5. Return the vendor string and a list of (field_name, field_value) tuples

### 5.2 Complete Python Implementation

```python
import struct

def read_vorbis_comment(data: bytes) -> tuple[str, list[tuple[str, str]]]:
    """
    Deserialize a Vorbis Comment block.
    
    Args:
        data: Raw comment header bytes (starting from vendor length).
    
    Returns:
        Tuple of (vendor_string, fields) where fields is a list of
        (field_name, field_value) tuples in the order they appear.
    
    Raises:
        ValueError: If the data is truncated or malformed.
    """
    offset = 0
    
    # Step 1: Vendor length
    if offset + 4 > len(data):
        raise ValueError("Truncated data: cannot read vendor length")
    vendor_length = struct.unpack_from('<I', data, offset)[0]
    offset += 4
    
    # Step 2: Vendor string
    if offset + vendor_length > len(data):
        raise ValueError("Truncated data: vendor string extends past end")
    vendor_string = data[offset:offset + vendor_length].decode('utf-8')
    offset += vendor_length
    
    # Step 3: Comment count
    if offset + 4 > len(data):
        raise ValueError("Truncated data: cannot read comment count")
    comment_count = struct.unpack_from('<I', data, offset)[0]
    offset += 4
    
    # Step 4: Comment fields
    fields = []
    for _ in range(comment_count):
        if offset + 4 > len(data):
            raise ValueError("Truncated data: cannot read field length")
        field_length = struct.unpack_from('<I', data, offset)[0]
        offset += 4
        
        if offset + field_length > len(data):
            raise ValueError("Truncated data: field string extends past end")
        field_bytes = data[offset:offset + field_length]
        offset += field_length
        
        try:
            field_str = field_bytes.decode('utf-8')
        except UnicodeDecodeError:
            # Replace invalid UTF-8 with replacement characters
            field_str = field_bytes.decode('utf-8', errors='replace')
        
        # Split on first '=' only
        if '=' in field_str:
            eq_pos = field_str.index('=')
            field_name = field_str[:eq_pos]
            field_value = field_str[eq_pos + 1:]
        else:
            # Malformed: no '=' present
            field_name = field_str
            field_value = ''
        
        fields.append((field_name, field_value))
    
    return vendor_string, fields
```

### 5.3 Field Lookup Helper

```python
from collections import defaultdict

def build_field_index(fields: list[tuple[str, str]]) -> dict[str, list[str]]:
    """
    Build an index of all field values by field name (case-insensitive).
    
    Returns a dict mapping uppercase field names to lists of values.
    Multiple instances of the same field name are preserved.
    """
    index = defaultdict(list)
    for name, value in fields:
        index[name.upper()].append(value)
    return dict(index)


def get_field(fields: list[tuple[str, str]], key: str) -> str | None:
    """Get the first value of a field, case-insensitive."""
    key_upper = key.upper()
    for name, value in fields:
        if name.upper() == key_upper:
            return value
    return None


def get_all_fields(fields: list[tuple[str, str]], key: str) -> list[str]:
    """Get all values of a field (for multi-value fields), case-insensitive."""
    key_upper = key.upper()
    return [value for name, value in fields if name.upper() == key_upper]
```

### 5.4 UTF-8 Validation

A strict Vorbis Comment reader should validate UTF-8 encoding. Invalid UTF-8 byte sequences can arise from:

- Tag editors that incorrectly assumed a different source encoding
- Corrupted file data
- Binary data accidentally written into comment fields

```python
def is_valid_utf8(data: bytes) -> bool:
    """Check if a byte sequence is valid UTF-8."""
    try:
        data.decode('utf-8')
        return True
    except UnicodeDecodeError:
        return False


def validate_field_string(data: bytes) -> tuple[bool, str | None]:
    """
    Validate a field string for conformance.
    
    Returns (is_valid, error_message).
    """
    # Check for valid UTF-8
    try:
        field_str = data.decode('utf-8')
    except UnicodeDecodeError as e:
        return False, f"Invalid UTF-8 at byte {e.start}"
    
    # Check for '=' presence
    if '=' not in field_str:
        return False, "Field has no '=' delimiter"
    
    # Check field name characters
    eq_pos = field_str.index('=')
    field_name = field_str[:eq_pos]
    
    if len(field_name) > 255:
        return False, f"Field name exceeds 255 bytes ({len(field_name)})"
    
    import re
    if not re.match(r'^[A-Za-z0-9_]*$', field_name):
        return False, f"Field name contains invalid characters: '{field_name}'"
    
    if not field_name:
        return False, "Field name is empty"
    
    return True, None
```

### 5.5 Container-Level Reading

**Ogg Vorbis:** The Ogg demuxer extracts the comment header packet from page 2 of the stream. The packet type byte (first byte of the Ogg packet) is `3` for comment headers.

**FLAC:** The FLAC metadata reader finds the first metadata block with type `4` and reads its payload as the Vorbis Comment block.

**Ogg Opus:** Similar to Ogg Vorbis; the comment header packet follows the OpusHead packet as the second Ogg packet in the stream.

### 5.6 Error Handling

Robust readers should handle the following error conditions:

**Truncated vendor length:** If fewer than 4 bytes are available at the start of the comment header, the data is corrupt or truncated.

**Vendor length too large:** If vendor_length exceeds the remaining data bytes, the data is corrupt.

**Truncated vendor string:** If vendor_length bytes cannot be read, the data is corrupt.

**Comment count exceeds reasonable limit:** If comment_count is extremely large (e.g., > 1,000,000), the data is likely corrupt. A reasonable upper bound is ~65,536 comments.

**Truncated comment field:** If a field_length exceeds the remaining data, the data is corrupt.

**Invalid UTF-8 in field string:** Replace invalid bytes with U+FFFD or treat the field as empty.

**Missing '=' delimiter:** Treat the entire field string as the field name with an empty value, or skip the field with a warning.

---

## 6. METADATA ARCHITECTURE

### 6.1 Complete Supported Tag Fields

| Field Name | Category | Multi-value | Format Notes |
|---|---|---|---|
| TITLE | Identification | No | Primary track title |
| VERSION | Identification | No | Track version or remix info |
| ALBUM | Identification | No | Album name |
| ALBUMARTIST | Identification | Yes | Album-level artist/collective |
| ARTIST | Identification | Yes | Primary performer(s) |
| COMPOSER | Attribution | Yes | Music composer |
| PERFORMER | Attribution | Yes | Performer credit (distinct from ARTIST) |
| ARRANGER | Attribution | Yes | Arrangement credit |
| LYRICIST | Attribution | Yes | Lyrics writer |
| DATE | Temporal | No | Recording date (YYYY or YYYY-MM-DD) |
| ORIGINALDATE | Temporal | No | Original recording date |
| ORIGINALALBUM | Identification | No | Original album name |
| ORIGINALARTIST | Attribution | Yes | Original artist for covers |
| LABEL | Publisher | No | Record label name |
| CATALOGNUMBER | Publisher | Yes | Label catalog number |
| BARCODE | Publisher | No | UPC/EAN barcode |
| ISRC | Publisher | Yes | International Standard Recording Code |
| PUBLISHER | Publisher | No | Synonym for LABEL |
| UPC | Publisher | No | Universal Product Code (alias for BARCODE) |
| EAN | Publisher | No | European Article Number (alias for BARCODE) |
| COPYRIGHT | Legal | No | Copyright notice |
| LICENSE | Legal | No | License URI |
| LOCATION | Context | No | Recording location |
| ORGANIZATION | Context | No | Organization name |
| GENRE | Classification | Yes | Genre classification |
| DESCRIPTION | Content | No | General track description |
| COMMENT | Content | Yes | Comment text |
| TRACKNUMBER | Ordering | No | Track number, zero-padded (e.g., "05/12") |
| TRACKTOTAL | Ordering | No | Total number of tracks |
| TOTALTRACKS | Ordering | No | Alias for TRACKTOTAL |
| DISCNUMBER | Ordering | No | Disc number (e.g., "1/2") |
| DISCTOTAL | Ordering | No | Total number of discs |
| TOTALDISCS | Ordering | No | Alias for DISCTOTAL |
| BPM | Technical | No | Beats per minute |
| KEY | Technical | No | Musical key (e.g., "C#m") |
| ENCODEDBY | Technical | No | Encoder software (write only, not part of spec) |
| ENCODER | Technical | No | Alias for ENCODEDBY |
| COMMENTURL | Reference | No | URL for comments |
| BUYURL | Reference | No | Purchase URL |
| ARTISTURL | Reference | No | Artist website URL |
| LABELURL | Reference | No | Label website URL |
| LICENSEURL | Reference | No | License information URL |

### 6.2 MusicBrainz Fields

MusicBrainz-specific fields used for discography integration:

| Field Name | Description | Format |
|---|---|---|
| MUSICBRAINZ_TRACKID | Recording identifier | UUID string |
| MUSICBRAINZ_ALBUMID | Release identifier | UUID string |
| MUSICBRAINZ_ALBUMARTISTID | Release artist identifier | UUID string |
| MUSICBRAINZ_ARTISTID | Artist identifier | UUID string |
| MUSICBRAINZ_RELEASEGROUPID | Release group identifier | UUID string |
| MUSICBRAINZ_WORKID | Work identifier | UUID string |
| MUSICBRAINZ_RELEASETRACKID | Release track identifier | UUID string |
| MUSICBRAINZ_DISCID | CD table of contents identifier | Base64-encoded TOC string |

### 6.3 Multi-Value Field Behavior

The following fields commonly have multiple values and should be stored as repeated field names:

```python
MULTI_VALUE_FIELDS = {
    'ARTIST',
    'ALBUMARTIST',
    'COMPOSER',
    'PERFORMER',
    'ARRANGER',
    'LYRICIST',
    'ORIGINALARTIST',
    'GENRE',
    'COMMENT',
    'CATALOGNUMBER',
    'ISRC',
    'MUSICBRAINZ_ARTISTID',
    'MUSICBRAINZ_TRACKID',
}
```

When writing, the order of values for multi-value fields should be preserved. When displaying, all values should be shown (e.g., "John Doe, Jane Smith" or a multi-line list).

### 6.4 Date Field Conventions

Date fields in Vorbis Comments do not have a strictly defined format, but conventions exist:

| Field | Recommended Format | Examples |
|---|---|---|
| DATE | ISO 8601 date | `2024`, `2024-03`, `2024-03-15` |
| ORIGINALDATE | ISO 8601 date | `1972`, `1972-06`, `1972-06-15` |

The year is the minimum useful granularity. Month and day provide more precision when available. The format should be consistent within the same field (mixing `2024` with `2024-03-15` in the same file is ambiguous).

The DATE field represents the recording date. It is distinct from ORIGINALDATE (the date of the original recording for reissues or covers).

---

## 7. VENDOR STRING SPECIFICATION

### 7.1 Purpose and Meaning

The vendor string identifies the software that wrote the Vorbis Comment metadata. It appears at the beginning of every Vorbis Comment block as a UTF-8 encoded string prefixed by its byte length. The vendor string is read-only in most implementations — it is set by the encoding software and is not meant to be manually edited.

The vendor string is primarily useful for:

- **Debugging metadata issues:** If tags are being written incorrectly, the vendor string identifies which tool is responsible
- **Compatibility workarounds:** Players can detect known problematic encoders and apply workarounds
- **Statistics:** Users can determine which encoders are most common in their library

### 7.2 Common Vendor Strings

| Vendor String | Encoder | Notes |
|---|---|---|
| `Xiph.Org libVorbis I 20030909` | libvorbis 1.0 | Classic Ogg Vorbis encoder |
| `Xiph.Org libVorbis I` | libvorbis | Generic libvorbis identifier |
| `Xiph.Org libFLAC 1.2.1` | FLAC 1.2.1 encoder | FLAC metadata writer |
| `Lavf60.xx.xx` | FFmpeg libavformat | FFmpeg muxer (replaced by Lavf) |
| `Lavf58.xx.xx` | FFmpeg libavformat | Older FFmpeg versions |
| `Lavf57.83.100` | FFmpeg libavformat | Example of format-specific vendor |
| `reference libFLAC 1.3.x` | FLAC reference encoder | Modern FLAC |
| `reference libFLAC 1.2.1 20070707` | FLAC 1.2.1 reference encoder | With build date |
| `opusENC 1.3` | opus-tools encoder | opusenc from opus-tools |
| `libopusenc 0.1` | libopusenc library | Used by various tools |
| `iTunes v12.x.x` | Apple iTunes | iTunes encodes to AAC, not Vorbis, but may write metadata |
| `dBpoweramp` | dBpoweramp | Audio conversion tool |
| `soundconverter` | SoundConverter | GNOME audio converter |
| `EasyMedia HI` | Easy Media | Easy Media audio converter |
| `Koepi` | Koepi tools | Various Xiph.Org tools on Windows |
| `vorbis-tools 1.4.0` | vorbis-tools | Command-line Ogg tools |
| `oggenc 2.87` | oggenc | Ogg Vorbis encoder from vorbis-tools |

### 7.3 FFmpeg Vendor String Evolution

FFmpeg's Vorbis encoder has historically written a vendor string indicating the libavformat version:

**Older FFmpeg versions (pre-2019):** Used `Lavf58.xx.xx` or similar, where `58` represents the libavformat major version number.

**Current FFmpeg versions:** FFmpeg's Vorbis encoder writes `LavfXX.XX.XX` where `XX.XX.XX` is the libavformat version string. For example, FFmpeg 6.0 would write `Lavf60.18.102`.

**FLAC muxer:** FFmpeg's FLAC muxer writes a vendor string like `LavfXX.XX.XX` for the Vorbis Comment block it creates.

**Opus muxer:** FFmpeg's Ogg Opus muxer also writes a Vorbis Comment block with the same vendor string format.

### 7.4 Vendor String Format Conventions

Vendor strings typically follow this pattern:

```
[Library Name] [Version] [Build Info]
```

- **Library name:** Identifies the software library or application (e.g., `Xiph.Org libVorbis`, `reference libFLAC`, `Lavf`)
- **Version:** Version number of the software (e.g., `1.2.1`, `60.18.102`)
- **Build info:** Optional build-specific information (e.g., `20070707` build date, `20030909` for libvorbis)

The vendor string must be valid UTF-8. It should be a human-readable ASCII string in most cases, though UTF-8 can accommodate non-ASCII characters if needed.

### 7.5 Reading Vendor String

```python
def get_vendor(data: bytes) -> str:
    """
    Extract the vendor string from a Vorbis Comment block.
    
    Args:
        data: Raw comment header bytes.
    
    Returns:
        The vendor string, decoded as UTF-8.
    """
    if len(data) < 4:
        raise ValueError("Data too short to contain vendor length")
    
    vendor_length = struct.unpack_from('<I', data, 0)[0]
    vendor_bytes = data[4:4 + vendor_length]
    return vendor_bytes.decode('utf-8', errors='replace')
```

---

## 8. REPLAYGAIN TAGS IN VORBIS COMMENTS

### 8.1 ReplayGain Tag Overview

ReplayGain tags in Vorbis Comments store volume normalization data. They are stored as ordinary text fields following the standard `KEY=VALUE` format. The field names are case-insensitive per the Vorbis Comment specification, but convention writes them in the canonical form shown below.

### 8.2 ReplayGain Field Specifications

**REPLAYGAIN_TRACK_GAIN**

- **Purpose:** The gain adjustment in dB to normalize the track to the reference loudness level
- **Format:** Signed decimal number with up to 2 decimal places, followed by " dB"
- **Example values:** `+3.20 dB`, `-1.50 dB`, `+0.00 dB`, `-6.00 dB`
- **Positive value:** Track is quieter than reference; positive gain raises it
- **Negative value:** Track is louder than reference; negative gain attenuates it

**REPLAYGAIN_TRACK_PEAK**

- **Purpose:** The peak sample amplitude in the track, used to prevent clipping after gain application
- **Format:** Unsigned decimal number with 6 decimal places (0.000000 to 1.000000)
- **Example values:** `0.987654`, `0.500000`, `1.000000`
- **Values above 1.0:** Indicate inter-sample peak (with oversampling) exceeded sample peak
- **Note:** Stored as linear amplitude ratio, not dB

**REPLAYGAIN_ALBUM_GAIN**

- **Purpose:** Gain adjustment for album-level normalization (all tracks normalized together)
- **Format:** Same as REPLAYGAIN_TRACK_GAIN
- **Note:** Computed across all tracks in album as a single program

**REPLAYGAIN_ALBUM_PEAK**

- **Purpose:** Peak amplitude across the entire album
- **Format:** Same as REPLAYGAIN_TRACK_PEAK

### 8.3 ReplayGain Value Format Details

The ReplayGain value format has specific requirements that differ from general Vorbis Comment values:

**The " dB" suffix:** The gain value must end with a space followed by "dB" (exactly: space + lowercase 'd' + uppercase 'B'). There is no space between the number and the "dB". Valid: `+3.20 dB`. Invalid: `+3.20`, `+3.20  dB`, `+3.20DB`.

**Decimal precision:** The value should have at most 2 decimal places. However, implementations may accept more and truncate. A value of `+3.2 dB` is equivalent to `+3.20 dB`.

**Sign character:** A leading `+` or `-` is required. A gain of exactly zero should be written as `+0.00 dB` (not `0.00 dB`).

**Decimal separator:** The period (`.`) is the only valid decimal separator. Commas are not supported.

### 8.4 Example ReplayGain Fields in Binary

For a FLAC file with ReplayGain data:

```
# REPLAYGAIN_TRACK_GAIN=-6.23 dB
# REPLAYGAIN_TRACK_PEAK=0.987654
# REPLAYGAIN_ALBUM_GAIN=-4.50 dB
# REPLAYGAIN_ALBUM_PEAK=1.000000

# Byte layout (length prefix + string for each field):
# Field 1: "REPLAYGAIN_TRACK_GAIN=-6.23 dB"
#   19 00 00 00  "REPLAYGAIN_TRACK_GAIN=-6.23 dB"
# Field 2: "REPLAYGAIN_TRACK_PEAK=0.987654"
#   1F 00 00 00  "REPLAYGAIN_TRACK_PEAK=0.987654"
# Field 3: "REPLAYGAIN_ALBUM_GAIN=-4.50 dB"
#   1C 00 00 00  "REPLAYGAIN_ALBUM_GAIN=-4.50 dB"
# Field 4: "REPLAYGAIN_ALBUM_PEAK=1.000000"
#   1C 00 00 00  "REPLAYGAIN_ALBUM_PEAK=1.000000"
```

### 8.5 ReplayGain Parsing

```python
import re

def parse_replaygain_gain(value: str) -> float | None:
    """
    Parse a ReplayGain gain value string.
    
    Valid formats: "+3.20 dB", "-1.50 dB", "+0.00 dB"
    
    Returns:
        Gain value in dB as a float, or None if invalid.
    """
    match = re.match(r'^([+-]?\d+\.?\d*)\s+dB$', value.strip())
    if not match:
        return None
    return float(match.group(1))


def parse_replaygain_peak(value: str) -> float | None:
    """
    Parse a ReplayGain peak value string.
    
    Valid formats: "0.987654", "1.000000", "0.500000"
    
    Returns:
        Peak as a linear amplitude ratio (0.0 to ~1.0+), or None if invalid.
    """
    match = re.match(r'^(\d+\.\d+)$', value.strip())
    if not match:
        return None
    return float(match.group(1))


def write_replaygain_gain(gain_db: float) -> str:
    """
    Format a ReplayGain gain value for writing.
    
    Args:
        gain_db: Gain value in dB.
    
    Returns:
        Formatted string like "+3.20 dB".
    """
    return f"{gain_db:+6.2f} dB"


def write_replaygain_peak(peak: float) -> str:
    """
    Format a ReplayGain peak value for writing.
    
    Args:
        peak: Peak as linear amplitude ratio.
    
    Returns:
        Formatted string like "0.987654".
    """
    return f"{peak:.6f}"
```

### 8.6 ReplayGain 2.0 Extensions

ReplayGain 2.0 introduced additional fields, though these are less widely supported:

| Field Name | Description | Format |
|---|---|---|
| REPLAYGAIN_TRACK_MINMAX | Minimum and maximum sample values | `min max` (two decimals) |
| REPLAYGAIN_ALBUM_MINMAX | Album minimum and maximum sample values | `min max` |

Example: `REPLAYGAIN_TRACK_MINMAX=0.000000 0.987654`

### 8.7 EBU R128 Tags (Opus-specific)

Opus files stored in Ogg containers may use EBU R128 tags instead of ReplayGain tags:

| Field Name | Description | Format |
|---|---|---|
| R128_TRACK_GAIN | EBU R128 track gain | Signed decimal dB (e.g., `-6.23`) |
| R128_TRACK_PEAK | EBU R128 track peak | Decimal (e.g., `0.987654`) |

Note: R128 gain values do **not** include the " dB" suffix — they are plain decimal numbers.

```python
def parse_r128_gain(value: str) -> float | None:
    """Parse an R128_TRACK_GAIN value (plain decimal, no 'dB' suffix)."""
    try:
        return float(value.strip())
    except ValueError:
        return None
```

---

## 9. STRING ENCODING RULES

### 9.1 UTF-8 Mandate

Vorbis Comments are defined to use **UTF-8 encoding exclusively**. This is one of the most important and cleanest aspects of the Vorbis Comment design. There is no encoding negotiation, no BOM, no endianness ambiguity, and no fallback to legacy encodings.

The UTF-8 mandate applies to:
- The vendor string
- All field names (which are ASCII by definition, but must be valid UTF-8)
- All field values

### 9.2 UTF-8 Validation

A strict Vorbis Comment reader must validate UTF-8 encoding. The following byte sequences are valid UTF-8:

| Code Point Range | UTF-8 Encoding |
|---|---|
| U+0000–U+007F | 0xxxxxxx (1 byte) |
| U+0080–U+07FF | 110xxxxx 10xxxxxx (2 bytes) |
| U+0800–U+FFFF | 1110xxxx 10xxxxxx 10xxxxxx (3 bytes) |
| U+10000–U+10FFFF | 11110xxx 10xxxxxx 10xxxxxx 10xxxxxx (4 bytes) |

Any byte sequence that does not conform to these patterns is invalid UTF-8.

**Common invalid patterns:**
- A continuation byte (`10xxxxxx`) not preceded by an appropriate start byte
- A start byte with too few continuation bytes
- Overlong encodings (e.g., encoding U+002F as `0xC0 0xAF` instead of `0x2F`)
- A byte with value `0xC0` or `0xC1` (these can only start overlong encodings)
- A byte with value `0xFE` or `0xFF` (these are never valid UTF-8)

### 9.3 UTF-8 Replacement Strategy

When encountering invalid UTF-8, a permissive reader should:

1. Attempt to decode with error handling enabled
2. Replace invalid sequences with the Unicode Replacement Character (U+FFFD, encoded as `0xEF 0xBF 0xBD`)
3. Continue parsing rather than aborting

```python
def safe_utf8_decode(data: bytes) -> str:
    """
    Decode bytes as UTF-8, replacing invalid sequences with U+FFFD.
    """
    return data.decode('utf-8', errors='replace')
```

### 9.4 Field Name Encoding

Field names in Vorbis Comments are constrained to ASCII characters `[A-Za-z0-9_]`. While they are technically encoded as UTF-8, ASCII is a subset of UTF-8, so this constraint is automatically satisfied.

**Field name comparison:** Field names are compared case-insensitively. This means `ARTIST`, `artist`, `Artist`, and `ArTiSt` are all treated as the same field name. However, the original case should be preserved when writing (many tools normalize to uppercase).

### 9.5 Value Encoding Examples

**ASCII-only value:**
```
ARTIST=The Beatles
# Bytes: 41 52 54 49 53 54 3D 54 68 65 20 42 65 61 74 6C 65 73
```

**UTF-8 with non-ASCII characters:**
```
TITLE=Mötley Crüe
# Bytes: 54 49 54 4C 45 3D 4D C3 B6 74 6C 65 79 20 43 72 C3 BC 65
# Note: ö = U+00F6 = C3 B6 in UTF-8
#       ü = U+00FC = C3 BC in UTF-8
```

**Cyrillic characters:**
```
ARTIST=Ария
# Bytes: 41 52 54 49 53 54 3D D0 90 D1 80 D0 B8 D1 8F
# Note: А = U+0410 = D0 90 in UTF-8
#      рия = D1 80 D0 B8 D1 8F in UTF-8
```

**Emoji characters (valid UTF-8, but display may vary):**
```
TITLE=Rock 🎸
# Bytes: 54 49 54 4C 45 3D 52 6F 63 6B 20 F0 9F 8E A8
# Note: 🎸 = U+1F3B8 = F0 9F 8E A8 in UTF-8 (4 bytes)
```

---

## 10. FIELD NAME CONVENTIONS

### 10.1 Case Insensitivity

The Vorbis Comment specification explicitly states that field names are **case-insensitive for comparison**. When looking up a field, implementations should compare field names in a case-insensitive manner.

However, the **original case should be preserved** when reading and writing. A field read as `Title=Hello` should be written back as `Title=Hello`, not normalized to `TITLE=Hello` (unless the writing tool explicitly normalizes case).

### 10.2 Canonical Case

While not mandated by the specification, the following canonical case conventions are widely followed:

| Field Name | Canonical Case | Notes |
|---|---|---|
| TITLE | TITLE | All uppercase |
| ARTIST | ARTIST | All uppercase |
| ALBUM | ALBUM | All uppercase |
| ALBUMARTIST | ALBUMARTIST | CamelCase |
| TRACKNUMBER | TRACKNUMBER | All uppercase |
| TRACKTOTAL | TRACKTOTAL | All uppercase |
| DATE | DATE | All uppercase |
| GENRE | GENRE | All uppercase |
| COMMENT | COMMENT | All uppercase |
| REPLAYGAIN_* | REPLAYGAIN_* | Uppercase with underscore |
| MUSICBRAINZ_* | MUSICBRAINZ_* | Uppercase with underscore |

FFmpeg normalizes all field names to uppercase when writing Vorbis Comments. Many other tools follow this convention.

### 10.3 Leading and Trailing Whitespace

Field names must not contain leading or trailing whitespace. A field name with a leading space (` TITLE=value`) or trailing space (`TITLE =value`) is technically malformed, though permissive parsers may strip the whitespace.

Field values may contain leading and trailing whitespace. This whitespace is preserved and should be included when reading. If the intent is to have a clean value, the whitespace should be stripped explicitly by the application.

### 10.4 Field Name Length

The Vorbis Comment specification recommends that field names not exceed **255 bytes** in length. This is a soft recommendation (not a hard constraint), but field names longer than 255 bytes may be rejected by some implementations.

The maximum useful field name length is much shorter in practice. Most field names are under 30 characters. Even complex MusicBrainz field names like `MUSICBRAINZ_RELEASEGROUPID` are only 28 characters.

### 10.5 Custom/User-Defined Fields

Vorbis Comments support arbitrary user-defined field names. Any field name not in the standard registry is a valid user-defined field. The convention for user-defined fields is to prefix them with `X-` or use a reverse-domain namespace:

**X- prefix convention (informal):**
```
X-ORIGINALALBUM=Original Album Name
X-CUSTOMTAG=Custom Value
X-MYTOOL-SOMETHING=Tool-specific metadata
```

**Reverse-domain convention (formal):**
```
ORG.MYDOMAIN.ALBUMTYPE=Live Recording
COM.MYDOMAIN.CUSTOMFIELD=Custom Value
```

The `X-` prefix convention is widely used and recognized as a marker for unofficial extensions. The reverse-domain convention provides namespace isolation for organizational use.

---

## 11. NORMALIZATION AND ALBUM CONVENTIONS

### 11.1 Track Number Format

Track numbers in Vorbis Comments follow specific formatting conventions for consistent sorting and display:

**Standard format:** `NN` or `NN/MM`
- `NN` is the track number, zero-padded to a consistent width
- `/MM` is the total number of tracks (optional but recommended)

**Width:** The padding width should match the total track count. For an album with 12 tracks, track numbers are `01`, `02`, ..., `12`. For an album with 105 tracks, they are `001`, `002`, ..., `105`.

**TRACKTOTAL vs TOTALTRACKS:** Both field names are used in practice. TRACKTOTAL is more common in the Vorbis/FLAC ecosystem. TOTALTRACKS is an alias that some tools use.

```python
# Good:
TRACKNUMBER=05/12
TRACKNUMBER=1/12
TRACKNUMBER=001/105

# Ambiguous (width unclear):
TRACKNUMBER=5/12

# Non-standard (missing total):
TRACKNUMBER=05
```

### 11.2 Disc Number Format

Disc numbers follow the same convention as track numbers:

**Standard format:** `NN` or `NN/MM`
- `NN` is the disc number
- `/MM` is the total number of discs

```python
# Good:
DISCNUMBER=1/2
DISCNUMBER=02/03

# Non-standard:
DISCNUMBER=Disc 1
```

### 11.3 Genre Conventions

Genres in Vorbis Comments are freeform text. There is no enumerated genre list (unlike ID3v1's byte-indexed genre system). Common genre strings include:

- Single genres: `Rock`, `Electronic`, `Classical`, `Jazz`, `Hip-Hop`
- Compound genres: `Electronic / Ambient`, `Rock / Alternative`
- Subgenres: `Black Metal`, `Post-Rock`, `Drum & Bass`

Some tools support MusicBrainz genre taxonomy, which uses a standardized set of genre names with specific capitalization.

### 11.4 Album Artist Convention

The `ALBUMARTIST` field (also called `ALBUM ARTIST`, `ALBUMARTIST`, or `ALBUM_ARTIST`) identifies the artist credited at the album level. This is distinct from `ARTIST`, which identifies the performer of each track.

**Use cases for ALBUMARTIST:**

1. **Compilation albums:** `ARTIST=John Doe` on each track, `ALBUMARTIST=Various Artists`
2. **Album by a group with featured artists:** `ARTIST=Beyoncé feat. Jay-Z`, `ALBUMARTIST=Beyoncé`
3. **Classical music:** `ARTIST=Yo-Yo Ma`, `ALBUMARTIST=Yo-Yo Ma; New York Philharmonic`
4. **Soundtracks:** `ALBUMARTIST=Various Artists`

The convention is to use `ALBUMARTIST` (no space) as the canonical field name, though `ALBUM ARTIST` (with space) is sometimes encountered.

### 11.5 ARTISTSORT and ALBUMARTISTSORT

For sorting purposes, the following sort fields are commonly used:

| Field | Purpose | Example |
|---|---|---|
| ARTISTSORT | Artist name for alphabetical sorting | ARTISTSORT=Doe, John |
| ALBUMARTISTSORT | Album artist for alphabetical sorting | ALBUMARTISTSORT=Various Artists |
| ALBUMSORT | Album name for alphabetical sorting | ALBUMSORT=Greatest Hits (1999) |
| TITLESORT | Track title for alphabetical sorting | TITLESORT=Hello (Radio Edit) |

Sort fields contain the intended sort key, not the display value. They are used by players and libraries that support alphabetical sorting.

---

## 12. FFMPEG IMPLEMENTATION REFERENCE

### 12.1 Reading Vorbis Comments with FFmpeg

FFmpeg and ffprobe read Vorbis Comments automatically for all supported formats (OGG Vorbis, FLAC, Ogg Opus, WebM).

```bash
# Read all metadata from an OGG Vorbis file
ffprobe -v quiet -show_format input.ogg

# Read in JSON format for programmatic parsing
ffprobe -v quiet -print_format json -show_format -show_streams input.flac

# Extract specific tag fields
ffprobe -v quiet -show_entries format_tag=title,artist,album,track -of default=noprint_wrappers=1 input.ogg

# Show all tag fields
ffprobe -v quiet -show_format input.ogg | grep -E "^TAG:|^comment="

# Check for embedded cover art (FLAC)
ffprobe -v quiet -show_entries format_tags -of json input.flac
```

### 12.2 Writing Vorbis Comments with FFmpeg

FFmpeg's `-metadata` option writes Vorbis Comments during encoding. The encoder maps these to the appropriate format.

```bash
# Encode to OGG Vorbis with metadata
ffmpeg -i input.wav -c:a libvorbis -q:a 6 \
  -metadata title="Song Title" \
  -metadata artist="Artist Name" \
  -metadata album="Album Name" \
  -metadata date=2024 \
  -metadata track=4 \
  -metadata genre="Rock" \
  output.ogg

# Encode to FLAC with metadata
ffmpeg -i input.wav -c:a flac -compression_level 8 \
  -metadata title="Song Title" \
  -metadata artist="Artist Name" \
  -metadata album="Album Name" \
  -metadata track=5 \
  output.flac

# Encode to Ogg Opus with metadata
ffmpeg -i input.wav -c:a libopus -b:a 128k \
  -metadata title="Song Title" \
  -metadata artist="Artist Name" \
  output.opus
```

### 12.3 ReplayGain Tags with FFmpeg

FFmpeg does not automatically compute ReplayGain values. However, you can write pre-computed ReplayGain tags:

```bash
# Write ReplayGain track gain and peak
ffmpeg -i input.flac -c:a copy \
  -metadata REPLAYGAIN_TRACK_GAIN="-6.23 dB" \
  -metadata REPLAYGAIN_TRACK_PEAK="0.987654" \
  output.flac

# Write album gain and peak
ffmpeg -i input.flac -c:a copy \
  -metadata REPLAYGAIN_ALBUM_GAIN="-4.50 dB" \
  -metadata REPLAYGAIN_ALBUM_PEAK="1.000000" \
  output.flac
```

For actually computing ReplayGain values, use a dedicated tool like `ffmpeg-normalize` (Python) or `vorbisgain`/`aacgain`.

### 12.4 Metadata Mapping

FFmpeg uses its own internal metadata key names, which map to Vorbis Comment field names. The mapping is generally straightforward:

| FFmpeg Key | Vorbis Comment Field |
|---|---|
| title | TITLE |
| artist | ARTIST |
| album | ALBUM |
| date | DATE |
| track | TRACKNUMBER |
| tracktotal | TRACKTOTAL |
| disc | DISCNUMBER |
| disctotal | DISCTOTAL |
| genre | GENRE |
| comment | COMMENT |
| composer | COMPOSER |
| copyright | COPYRIGHT |
| encodedby | ENCODEDBY |
| encoder | ENCODER |
| performer | PERFORMER |

**Critical note:** FFmpeg may not preserve all custom or MusicBrainz fields during transcoding. When converting between formats that both support Vorbis Comments (e.g., FLAC to OGG), use `-map_metadata 0` to preserve all metadata:

```bash
# Preserve all metadata when converting FLAC → OGG
ffmpeg -i input.flac -c:a libvorbis -q:a 6 -map_metadata 0 output.ogg

# Strip metadata and add specific fields
ffmpeg -i input.flac -c:a libvorbis -q:a 6 -map_metadata -1 \
  -metadata title="New Title" \
  output.ogg
```

### 12.5 Embedded Cover Art

FLAC supports embedded cover art via the METADATA_BLOCK_PICTURE structure (metadata block type 6). FFmpeg can handle this through the `-c:v copy` approach:

```bash
# Extract cover art from FLAC
ffmpeg -i input.flac -c:v copy -map 0:v output.jpg

# Embed cover art when encoding
ffmpeg -i input.wav -i cover.jpg -c:a flac -compression_level 8 \
  -c:v copy -map 1:v -map 0:a \
  output.flac
```

Note: Vorbis Comments themselves do not define a mechanism for embedded cover art. FLAC's METADATA_BLOCK_PICTURE is a FLAC-specific extension. For Ogg Vorbis, cover art is typically stored as a separate Ogg packet (using the `SPX` picture extension) or not at all.

---

## 13. TAG COLLISION AND DUPLICATION HANDLING

### 13.1 Multiple Instances of the Same Field

Vorbis Comments allow the same field name to appear multiple times in the comment list. This is by design and is the standard mechanism for multi-value fields like ARTIST and GENRE.

**Reading behavior:** A reader should return all instances of a field. A writer should preserve all instances from the source.

**Writing behavior:** When writing, if the new value set differs from the old, the writer should replace all instances of the field with the new set. Some tools incorrectly overwrite only the first instance, leaving orphaned duplicates.

### 13.2 Orphaned Field Duplicates

A common issue arises when editing tags: a field like ARTIST may appear 3 times in the original, and the editor wants to replace all instances with 1 new value. If the editor only overwrites the first instance, the result is:

```
ARTIST=New Artist
ARTIST=Old Artist 2
ARTIST=Old Artist 3
```

This is a metadata corruption issue. The fix is to completely rebuild the field list when writing:

```python
def merge_fields(
    existing: list[tuple[str, str]],
    updates: dict[str, str | list[str]],
) -> list[tuple[str, str]]:
    """
    Merge updated fields into existing fields, removing duplicates.
    
    Args:
        existing: List of (field_name, field_value) tuples from the source.
        updates: Dict mapping field names to new values. Single string = replace all
                 instances; list of strings = set to those values.
    
    Returns:
        Updated list of fields with duplicates removed.
    """
    # Build a dict of all fields (uppercase name -> list of values)
    field_index: dict[str, list[str]] = defaultdict(list)
    for name, value in existing:
        field_index[name.upper()].append(value)
    
    # Apply updates
    for field_name, new_value in updates.items():
        key = field_name.upper()
        if isinstance(new_value, list):
            field_index[key] = new_value
        else:
            field_index[key] = [new_value]
    
    # Flatten back to ordered list
    result = []
    for name, value in existing:
        key = name.upper()
        if field_index.get(key):
            result.append((name, field_index[key].pop(0)))
            if not field_index[key]:
                del field_index[key]
    
    # Add any remaining new fields
    for key, values in field_index.items():
        for value in values:
            result.append((key, value))
    
    return result
```

### 13.3 Conflict Resolution

When the same field appears with different values from different sources (e.g., during metadata merging from multiple databases), the resolution strategy depends on the use case:

**Prefer authoritative source:** If merging MusicBrainz data with user-entered data, prefer MusicBrainz for structured fields (artist IDs, release IDs) and user data for display fields (title, artist name).

**Preserve all values:** For multi-value fields, preserve all values rather than arbitrarily discarding some.

**Track provenance:** Some systems store the source of each field value to allow later conflict resolution.

---

## 14. KNOWN ISSUES AND EDGE CASES

### 14.1 Null Bytes in Field Values

Some tag editors have historically written null bytes (`0x00`) in field values, particularly when migrating from other formats. A null byte in a Vorbis Comment value is invalid UTF-8 and technically indicates corrupted data.

**Handling:** Strip null bytes or replace them with a replacement character. Never silently pass invalid UTF-8 to downstream consumers.

### 14.2 Missing Field Name After Edit

A common bug in naive tag editors is writing a field with an empty field name (just `=value`). This is technically invalid per the spec, which requires field names to contain at least one character from `[A-Za-z0-9_]`.

**Handling:** Reject fields with empty or invalid field names. Log a warning when encountering such fields.

### 14.3 Multiple Equals Signs

A field value may contain literal `=` characters. The parsing rule is: split on the **first** `=` only. Everything after the first `=` is the value, including subsequent `=` characters.

```python
# These are all valid:
URL=http://example.com?a=b&c=d
FORMULA=Na=HCO3
EQUATION=x=y=z

# Parsing:
"URL=http://example.com?a=b&c=d" → name="URL", value="http://example.com?a=b&c=d"
"FORMULA=Na=HCO3" → name="FORMULA", value="Na=HCO3"
```

### 14.4 Very Long Field Values

The Vorbis Comment spec does not impose a maximum field value length. However, in practice, very long values (e.g., >1 MB of text) may cause issues with some parsers or players.

**Handling:** Accept arbitrarily long values, but be aware that some tools may truncate or reject them.

### 14.5 Case Mismatch in Field Lookup

A subtle issue arises when a field like `ARTIST=John` is read and then written back by a tool that uses case-insensitive lookup internally but preserves original case. The next read might return `ARTIST=John`. This is correct behavior.

However, if the writing tool uses case-sensitive lookup, it might fail to find the existing `ARTIST` when trying to update it if the field was written as `artist=John`.

**Handling:** Always use case-insensitive comparison for field lookup.

### 14.6 Binary Data in Field Values

Some tools have written binary data (e.g., ID3v2 PRIV frames serialized as text) into Vorbis Comment fields. This produces invalid UTF-8 and garbage display.

**Handling:** Reject or replace invalid UTF-8. Warn the user about corrupted field values.

### 14.7 Vendor String Truncation

Some tools truncate the vendor string to fit a fixed buffer, producing a partial UTF-8 sequence at the end. This is invalid but sometimes encountered.

**Handling:** Validate UTF-8 in the vendor string. If the vendor string ends with an incomplete multi-byte sequence, truncate to the last complete character.

---

## 15. REFERENCE IMPLEMENTATIONS

### 15.1 libvorbis / libogg (C)

The reference implementation of the Vorbis codec includes the comment header reading and writing code. The `vorbis_comment_*` functions in `vorbis/codec.h` and `vorbis/comment.c` are the canonical implementation.

Key functions:
- `vorbis_comment_init()` — create an empty comment state
- `vorbis_comment_add()` — add a field
- `vorbis_comment_add_tag()` — add a field by name and value
- `vorbis_comment_query()` — query a field value
- `vorbis_comment_query_count()` — count instances of a field
- `vorbis_comment_clear()` — free comment state

```c
#include <vorbis/codec.h>

vorbis_comment vc;
vorbis_comment_init(&vc);
vorbis_comment_add_tag(&vc, "TITLE", "Example Track");
vorbis_comment_add_tag(&vc, "ARTIST", "Example Artist");

// Query
char *artist = vorbis_comment_query(&vc, "ARTIST", 0);  // First ARTIST

vorbis_comment_clear(&vc);
```

### 15.2 Mutagen (Python)

The Mutagen library provides a complete Vorbis Comment implementation in Python, used by Quod Libet, Ex Falso, and other applications.

```python
from mutagen.flac import FLAC
from mutagen.oggvorbis import OggVorbis

# Read FLAC tags
audio = FLAC("input.flac")
print(audio["TITLE"])
print(audio["ARTIST"])
print(audio["REPLAYGAIN_TRACK_GAIN"])

# Write tags
audio["TITLE"] = "New Title"
audio["ARTIST"] = ["Artist 1", "Artist 2"]  # Multi-value
audio["REPLAYGAIN_TRACK_GAIN"] = "+3.20 dB"
audio["REPLAYGAIN_TRACK_PEAK"] = "0.987654"
audio.save()

# Read Ogg Vorbis tags
audio = OggVorbis("input.ogg")
print(audio["TITLE"])
```

### 15.3 TagLib (C++)

TagLib provides a high-level interface for reading and writing Vorbis Comments across multiple formats.

```cpp
#include <taglib/taglib.h>
#include <taglib/flacfile.h>
#include <taglib/vorbisfile.h>
#include <taglib/opusfile.h>

// FLAC
TagLib::FLAC::File flacFile("input.flac");
TagLib::Ogg::Vorbis::Tag *tag = flacFile.vorbisComment();
tag->setTitle("New Title");
tag->setArtist("Artist Name");
tag->addField("REPLAYGAIN_TRACK_GAIN", "+3.20 dB");
tag->addField("REPLAYGAIN_TRACK_PEAK", "0.987654");
flacFile.save();

// Ogg Vorbis
TagLib::Ogg::Vorbis::File oggFile("input.ogg");
TagLib::Ogg::Vorbis::Tag *oggTag = oggFile.tag();
oggTag->setTitle("New Title");
oggTag->save();
```

### 15.4 kid3 (Qt/C++)

kid3 is a feature-rich tag editor with a command-line interface (kid3-cli) that fully supports Vorbis Comments. It is particularly useful for batch editing and for verification workflows.

```bash
# Read all tags
kid3-cli -c "get" "input.flac"

# Set fields
kid3-cli -c "set title 'New Title'" "input.flac"
kid3-cli -c "set artist 'Artist Name'" "input.flac"

# Set ReplayGain
kid3-cli -c "set REPLAYGAIN_TRACK_GAIN '+3.20 dB'" "input.flac"
kid3-cli -c "set REPLAYGAIN_TRACK_PEAK '0.987654'" "input.flac"

# Batch set from template
kid3-cli -c "set title %1" -c "set artist %2" "input.flac"
```

### 15.5 FFmpeg (C)

FFmpeg's libavformat implements Vorbis Comment reading and writing through the Ogg and FLAC muxers and demuxers. The code is in `libavformat/oggparsevorbis.c` and `libavformat/flacenc.c`.

```bash
# Read metadata
ffprobe -v quiet -show_format -of json input.ogg

# Write metadata
ffmpeg -i input.wav -c:a libvorbis -q:a 6 \
  -metadata title="Title" \
  -metadata artist="Artist" \
  -metadata REPLAYGAIN_TRACK_GAIN="+3.20 dB" \
  -metadata REPLAYGAIN_TRACK_PEAK="0.987654" \
  output.ogg
```

---

## 16. RELEVANT SPECIFICATIONS AND FURTHER READING

### 16.1 Primary Specifications

**Vorbis Comment Specification:**
- Official page: https://xiph.org/vorbis/doc/Vorbis_I_spec.html
- Comment header section: Section 5 of the Vorbis I specification
- Covers: vendor string, comment field format, field name conventions

**Ogg Vorbis Registration (RFC 5215):**
- https://www.rfc-editor.org/rfc/rfc5215
- IETF informational RFC registering the Ogg Vorbis media type
- Codifies Vorbis Comments as the canonical metadata format

**Ogg Container Specification:**
- https://xiph.org/ogg/doc/
- Defines Ogg page structure, packet ordering, and chaining rules
- Vorbis Comments occupy a specific position in the Ogg stream

**FLAC Format Specification:**
- https://xiph.org/flac/format
- Defines FLAC metadata block types, including Vorbis Comments (type 4)
- Also defines METADATA_BLOCK_PICTURE (type 6) for cover art

**Opus RFC 6716:**
- https://www.rfc-editor.org/rfc/rfc6716
- Section 5 defines the Ogg Opus mapping
- Ogg Opus uses Vorbis Comments as the metadata format

### 16.2 ReplayGain Specifications

**ReplayGain 1.0 Specification:**
- http://wiki.hydrogenaudio.org/index.php?title=ReplayGain_specification
- Original ReplayGain specification from Hydrogenaudio
- Defines 89 dB SPL reference level and measurement algorithm

**ReplayGain 2.0:**
- http://wiki.hydrogenaudio.org/index.php?title=ReplayGain_2.0_specification
- Updated specification with 95 dB SPL reference level
- Defines true peak measurement and additional fields

### 16.3 Community Resources

**Hydrogenaudio Knowledgebase:**
- https://wiki.hydrogenaudio.org/
- Comprehensive information on audio formats and metadata
- Vorbis Comment field registry and usage conventions

**MusicBrainz Tagging Guidelines:**
- https://musicbrainz.org/doc/style/
- Detailed conventions for all metadata fields
- MusicBrainz field name mapping

### 16.4 Library Documentation

**libvorbis documentation:**
- https://xiph.org/vorbis/doc/

**Mutagen documentation:**
- https://mutagen.readthedocs.io/
- Python library for all audio metadata formats

**TagLib API documentation:**
- https://taglib.org/api/
- C++ library for reading and writing audio metadata

---

## 17. IMPLEMENTATION CHECKLIST (for the Converter Developer)

This checklist aligns with the project's kid3-cli metadata verification rule. For each conversion run, verify these points against the source and destination files.

### 17.1 Reading and Validation

- [ ] Vendor string is readable and valid UTF-8
- [ ] All field names contain only `[A-Za-z0-9_]` characters
- [ ] All field names are ≤ 255 bytes
- [ ] All field values are valid UTF-8 (replace invalid sequences if needed)
- [ ] No field string contains fewer than 4 bytes (would indicate corrupted length prefix)
- [ ] Comment count is reasonable (≤ 65,536 fields)
- [ ] Total comment header size is consistent with declared vendor length + comment count + all field lengths
- [ ] Field lookup uses case-insensitive comparison

### 17.2 Standard Field Preservation

- [ ] TITLE preserved exactly (including leading/trailing whitespace in value)
- [ ] ARTIST preserved — all instances if multiple artists
- [ ] ALBUM preserved
- [ ] ALBUMARTIST preserved — all instances if multiple
- [ ] DATE preserved exactly
- [ ] TRACKNUMBER preserved in zero-padded format
- [ ] TRACKTOTAL preserved (or TOTALTRACKS if used)
- [ ] DISCNUMBER preserved
- [ ] DISCTOTAL preserved (or TOTALDISCS if used)
- [ ] GENRE preserved — all instances if multiple genres
- [ ] COMMENT preserved — all instances
- [ ] COMPOSER preserved — all instances
- [ ] PERFORMER preserved — all instances

### 17.3 Extended Field Preservation

- [ ] COPYRIGHT preserved
- [ ] LABEL / PUBLISHER preserved
- [ ] CATALOGNUMBER preserved — all instances
- [ ] BARCODE / UPC preserved
- [ ] ISRC preserved — all instances
- [ ] LOCATION preserved
- [ ] BPM preserved
- [ ] KEY (musical) preserved

### 17.4 ReplayGain Tag Preservation

- [ ] REPLAYGAIN_TRACK_GAIN preserved — format matches `+N.NN dB`
- [ ] REPLAYGAIN_TRACK_PEAK preserved — format matches `0.NNNNNN`
- [ ] REPLAYGAIN_ALBUM_GAIN preserved if present — format matches
- [ ] REPLAYGAIN_ALBUM_PEAK preserved if present — format matches
- [ ] R128_TRACK_GAIN preserved if present (Opus files)
- [ ] R128_TRACK_PEAK preserved if present (Opus files)
- [ ] No ReplayGain value exceeds reasonable bounds (gain: -30 to +30 dB; peak: 0 to 2.0)

### 17.5 MusicBrainz Field Preservation

- [ ] MUSICBRAINZ_TRACKID preserved — UUID format
- [ ] MUSICBRAINZ_ALBUMID preserved — UUID format
- [ ] MUSICBRAINZ_ARTISTID preserved — all instances, UUID format
- [ ] MUSICBRAINZ_ALBUMARTISTID preserved if present
- [ ] MUSICBRAINZ_RELEASEGROUPID preserved if present
- [ ] MUSICBRAINZ_WORKID preserved if present
- [ ] All other MUSICBRAINZ_* fields preserved if present

### 17.6 Sort Field Preservation

- [ ] ARTISTSORT preserved if present
- [ ] ALBUMARTISTSORT preserved if present
- [ ] ALBUMSORT preserved if present
- [ ] TITLESORT preserved if present

### 17.7 Cover Art Preservation (FLAC)

- [ ] METADATA_BLOCK_PICTURE present if source had embedded cover art
- [ ] Cover art MIME type correct (image/jpeg or image/png)
- [ ] Cover art dimensions preserved
- [ ] Cover art binary data identical to source (byte-for-byte)

### 17.8 kid3-cli Verification Commands

```bash
# Read source file tags
kid3-cli -c "get" "/path/to/source/file"

# Read destination file tags
kid3-cli -c "get" "/path/to/destination/file"

# Compare specific fields
kid3-cli -c "get" -c "export -" "/path/to/source/file"
kid3-cli -c "get" -c "export -" "/path/to/destination/file"
```

### 17.9 Verification Checklist (Pass Conditions)

- [ ] No `TagLib: ... Invalid UTF-8` errors in kid3 output
- [ ] No `TagLib: ... Ignoring duplicate atom` warnings (for FLAC/MP4)
- [ ] All standard tags match source exactly (case-sensitive for values)
- [ ] Every custom/unknown field from source appears in destination
- [ ] Numeric fields (track, disc, year) contain the correct numeric values
- [ ] ReplayGain values have correct format (`+N.NN dB`, `0.NNNNNN`)
- [ ] Cover art (METADATA_BLOCK_PICTURE) present if source had it
- [ ] Cover art binary matches source exactly
- [ ] No truncated or orphaned duplicate fields after edit

### 17.10 Common Failure Modes

**FFmpeg metadata stripping:** FFmpeg may not preserve all custom fields when using `-map_metadata 0`. Always verify with kid3-cli after FFmpeg-mediated conversions.

**Case normalization:** Some tools normalize field names to uppercase. This is acceptable but should be consistent across the pipeline.

**ReplayGain format errors:** The ` dB` suffix must be lowercase, with exactly one space before `dB`. Wrong: `dB`, `DB`, ` dB`, `  dB`.

**Track number format drift:** Some tools write `TRACKNUMBER=1` while others write `TRACKNUMBER=01`. For sorting consistency, prefer zero-padded format with total: `TRACKNUMBER=01/12`.

**Multi-value field duplication:** When updating multi-value fields, ensure all old instances are removed before adding new ones. Check for orphaned duplicates.

**Vendor string changes:** The vendor string will change when converting with FFmpeg or other tools. This is expected and acceptable — it indicates the destination was written by the conversion tool.
