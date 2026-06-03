# XMP (Extensible Metadata Platform) in Audio Files — Deep Technical Reference
> **Category:** Metadata
> **File Extensions:** `.wav`, `.mp3`, `.mp4`, `.m4a`, `.flac`, `.ogg`, `.pdf`, `.jpg`, `.png`, `.psd`, `.ai`
> **MIME Types:** Various (format-dependent)
> **Standardization Body:** ISO/IEC, Adobe Systems
> **Primary Specification:** ISO 16684-1:2019, Adobe XMP Specification Part 1-3
> **Patent Status:** Open — XMP is an open standard published by Adobe
> **License:** Open (BSD-style license for SDK)
> **Current Version:** ISO 16684-1:2019 (XMP Part 1), ISO 16684-2:2014 (Part 2 - RELAX NG)
> **Active Development:** Yes — ISO TC 171/SC 2 WG 12 maintains the standard

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Adobe Systems Incorporated
- **Year Created:** 2001 (released with Photoshop 7)
- **Original Purpose:** Create a universal metadata container that could embed standardized metadata across diverse file formats, replacing fragmented proprietary systems.
- **Problem with Predecessors:** Before XMP, metadata was stored in incompatible formats: EXIF for cameras, IPTC for news photos, ID3 for audio, Vorbis comments for OGG, iTunes atoms for MP4. XMP provides a unified RDF/XML-based container.

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| XMP 1.0 | 2001 | Initial release with Photoshop 7 |
| XMP 1.1 | 2002 | Added Dublin Core namespace |
| XMP 2004 | 2004 | RDF serialization improvements |
| ISO 16684-1:2012 | 2012 | ISO standardization of Part 1 |
| ISO 16684-1:2019 | 2019 | Updated specification |
| ISO 16684-2:2014 | 2014 | Part 2: Schema descriptions (RELAX NG) |
| ISO 16684-3 | 2020 | Part 3: Storage in files |
| JSON-LD Serialization | 2024 | New lightweight serialization (ISO 79384) |

### 1.3 Current Adoption
- **Primary use cases today:** Digital asset management, photography workflows, document management, cross-application metadata portability
- **Platforms with native support:** Adobe Creative Suite, Microsoft Office, many digital asset management (DAM) systems
- **Major software using this format:** Photoshop, Lightroom, InDesign, Acrobat, Bridge, Final Cut Pro, Premiere
- **Hardware support:** Limited in dedicated audio hardware; primarily software-based
- **Status:** Dominant in image/video workflows; adoption in audio is growing but non-uniform

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 XMP Data Model

XMP is built on three foundational concepts:

```
1. Data Model (XMP Core)
   ├── Property: name-value pair with type
   ├── Structure: nested property containers
   └── Array: ordered/unordered collection of values

2. Serialization (RDF/XML)
   └── XML representation of the data model

3. Namespaces
   └── URI-prefixed vocabulary definitions
```

### 2.2 XMP Namespaces

Namespaces prevent property name collisions by qualifying properties with a unique URI:

| Namespace URI | Prefix | Description |
|---------------|--------|-------------|
| http://purl.org/dc/elements/1.1/ | dc | Dublin Core Metadata Element Set |
| http://ns.adobe.com/xap/1.0/ | xap | XMP Basic |
| http://ns.adobe.com/xap/1.0/mm/ | xmpMM | XMP Media Management |
| http://ns.adobe.com/xap/1.0/rights/ | xmpRights | Rights management |
| http://ns.adobe.com/photoshop/1.0/ | photoshop | Photoshop-specific |
| http://ns.adobe.com/exif/1.0/ | exif | EXIF data |
| http://ns.adobe.com/pdf/1.3/ | pdf | PDF properties |
| http://www.adobe.com/pdfx/1.3/ | pdfx | PDF/X extension |
| http://ns.adobe.com/voice/1.0/ | voice | Voice annotation [NEEDS VERIFICATION] |

### 2.3 Dublin Core Namespace (dc)

The Dublin Core namespace is the most commonly used metadata vocabulary:

| Property | Description | Example |
|----------|-------------|---------|
| dc:title | Title of the resource | "Symphony No. 5" |
| dc:creator | Creator/author | ["Beethoven", "Vienna Philharmonic"] |
| dc:description | Description | "Live recording from 2024" |
| dc:subject | Subject/keywords | ["classical", "orchestra"] |
| dc:publisher | Publisher | "Classical Records" |
| dc:contributor | Contributors | ["Conductor: Karajan"] |
| dc:date | Date | "2024-03-15" |
| dc:type | Type | "Sound" (Dublin Core Type) |
| dc:format | Format | "audio/wav" |
| dc:identifier | Identifier | "ISRC:..." |
| dc:source | Source | "Original master tape" |
| dc:language | Language | "en" (ISO 639) |
| dc:relation | Related resources | "Part of: Complete Works" |
| dc:coverage | Coverage | "Vienna, Austria" |
| dc:rights | Rights | "© 2024 Classical Records" |

---

## 3. XMP PACKET STRUCTURE

### 3.1 XMP Packet Wrapper

An XMP packet is a self-contained XML block that can be embedded in files:

```xml
<?xpacket begin="" id="W5M0MpCehiHzreSzNTczkc9d"?>
<x:xmpmeta xmlns:x="adobe:ns:meta/">
  <rdf:RDF xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#">
    <rdf:Description rdf:about=""
          xmlns:dc="http://purl.org/dc/elements/1.1/"
          xmlns:xmp="http://ns.adobe.com/xap/1.0/">
      <dc:title>
        <rdf:Alt>
          <rdf:li xml:lang="x-default">Track Title</rdf:li>
        </rdf:Alt>
      </dc:title>
      <dc:creator>
        <rdf:Seq>
          <rdf:li>Artist Name</rdf:li>
        </rdf:Seq>
      </dc:creator>
    </rdf:Description>
  </rdf:RDF>
</x:xmpmeta>
<?xpacket end="w"?>
```

### 3.2 Packet Header/Trailer

| Element | Purpose | Notes |
|---------|---------|-------|
| `<?xpacket begin="" id="..."?>` | Packet start marker | Unique ID for the packet |
| `<?xpacket end="w"?>` | Packet end marker | "w" = writable, "r" = read-only |

### 3.3 Complete XMP Packet Example for Audio

```xml
<?xpacket begin="" id="W5M0MpCehiHzreSzNTczkc9d"?>
<x:xmpmeta x:xmptk="Adobe XMP Core 9.0-c000 79.171919, 2023/08/15-00:00:00 "
           xmlns:x="adobe:ns:meta/">
  <rdf:RDF xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#"
           xmlns:dc="http://purl.org/dc/elements/1.1/"
           xmlns:xmp="http://ns.adobe.com/xap/1.0/"
           xmlns:xmpMM="http://ns.adobe.com/xap/1.0/mm/"
           xmlns:ie="http://ns.adobe.com/iel/1.0/"
           xmlns:ebucore="http://www.ebu.ch/metadata/ebucore/">
    <rdf:Description rdf:about=""
         dc:title="Symphony No. 5 in C minor"
         dc:creator="Ludwig van Beethoven"
         dc:description="Live recording from Vienna Konzerthaus"
         dc:subject="classical;symphony;orchestra"
         dc:date="2024-03-15"
         dc:rights="© 2024 Classical Records Ltd."
         dc:format="audio/wav"
         xmp:CreatorTool="Adobe Audition 24.0"
         xmpMM:DocumentID="xmp.did:12345678-abcd-efgh-ijkl-012345678901">
      <dc:creator>
        <rdf:Seq>
          <rdf:li>Beethoven</rdf:li>
        </rdf:Seq>
      </dc:creator>
      <dc:subject>
        <rdf:Bag>
          <rdf:li>classical</rdf:li>
          <rdf:li>symphony</rdf:li>
          <rdf:li>orchestra</rdf:li>
        </rdf:Bag>
      </dc:subject>
      <dc:relation>
        <rdf:Bag>
          <rdf:li rdf:resource="xmp.did:parent-document-id"/>
        </rdf:Bag>
      </dc:relation>
    </rdf:Description>
  </rdf:RDF>
</x:xmpmeta>
<?xpacket end="w"?>
```

---

## 4. XMP IN AUDIO FILE FORMATS

### 4.1 Audio Format XMP Storage Overview

| Format | XMP Storage Method | Standard? | Notes |
|--------|-------------------|-----------|-------|
| MP3 | ID3v2 PRIV frame | Yes | Owner ID "com.adobe.xmp" |
| MP4/M4A | uuid atom | No | Non-standard extension |
| WAV | LIST INFO 'XMP ' | No | Non-standard extension |
| FLAC | Vorbis Comment | Yes | XMP field or metadata block |
| OGG Vorbis | Vorbis Comment | Yes | XMP field |
| AIFF | ID3v2 at start | Yes | As with MP3 |
| PDF | XMP stream | Yes | Standard PDF feature |

### 4.2 XMP in MP3 Files (ID3v2)

XMP is stored in MP3 files using an ID3v2.4 PRIV frame with owner identifier `com.adobe.xmp`.

**ID3v2 PRIV Frame Structure:**
```
Offset  Bytes   Field Name              Type        Description
------  ------  ------------------     ----------  ----------------------------------
0x00    4       Frame ID                char[4]     "PRIV"
0x04    4       Size                    uint32 BE   Frame size (synchsafe in v2.4)
0x08    2       Flags                   uint16 BE   Frame flags
0x0A    n       Owner Identifier        string      Null-terminated ASCII
0x??    m       XMP Packet              binary      Full XMP packet data
```

**Example:**
```
Frame ID:   "PRIV"
Size:       0x00010000 (65536 bytes)
Flags:      0x0000
Owner:      "com.adobe.xmp\0"
Data:       [Full XMP packet]
```

### 4.3 XMP in MP4/M4A Files (uuid atom)

XMP in MP4 files is stored as a `uuid` atom containing the XMP packet. This is a non-standard but documented extension used by Adobe products.

**uuid Atom Structure:**
```
Offset  Bytes   Field Name              Type        Description
------  ------  ------------------     ----------  ----------------------------------
0x00    4       Atom Size               uint32 BE   Atom size including header
0x04    16      UUID Type               binary      "http://ns.adobe.com/xap/1.0/"
0x14    n       XMP Packet              binary      Full XMP packet data
```

**UUID Value:**
```
UUID = { 0xBE, 0xBA, 0x7B, 0x1D, 0x34, 0x23, 0x41, 0x7D,
         0xAC, 0xCF, 0x6E, 0x4F, 0xE4, 0x0F, 0x00, 0x00 }
```

### 4.4 XMP in WAV Files (LIST INFO extension)

XMP in WAV files is stored as a subchunk within the `LIST INFO` container. This is a non-standard extension.

**WAV XMP Subchunk Structure:**
```
Offset  Bytes   Field Name              Type        Description
------  ------  ------------------     ----------  ----------------------------------
0x00    4       Subchunk ID             char[4]     "XMP " (space after)
0x04    4       Subchunk Size           uint32 LE   Size of XMP packet
0x08    n       XMP Packet              binary      Full XMP packet data
```

**Important:** The subchunk ID includes a trailing space: `"XMP "` (0x58 0x4D 0x50 0x20).

### 4.5 XMP in FLAC Files

XMP in FLAC can be stored via the Vorbis Comment field:

```
Field Name: XMP
Field Value: [Full XMP packet as single-line string]
```

Alternatively, XMP can be embedded as a METADATA_BLOCK_PICTURE using the XMP picture type.

### 4.6 XMP in OGG Vorbis Files

Similar to FLAC, XMP is stored as a Vorbis Comment field:

```bash
# Reading XMP from FLAC/OGG Vorbis
exiftool -XMP output.flac

# Writing XMP
exiftool -XMP="<x:xmpmeta>..." output.flac
```

---

## 5. RDF SERIALIZATION — DEEP DETAIL

### 5.1 RDF Basics for XMP

XMP uses a subset of RDF/XML for serialization. The key constructs are:

**Simple Properties:**
```xml
<dc:title>Track Title</dc:title>
```

**Localized Properties (with language):
```xml
<dc:title>
  <rdf:Alt>
    <rdf:li xml:lang="x-default">Default Title</rdf:li>
    <rdf:li xml:lang="de">Standard Titel</rdf:li>
    <rdf:li xml:lang="fr">Titre par défaut</rdf:li>
  </rdf:Alt>
</dc:title>
```

**Ordered Arrays (Seq):**
```xml
<dc:creator>
  <rdf:Seq>
    <rdf:li>First Creator</rdf:li>
    <rdf:li>Second Creator</rdf:li>
  </rdf:Seq>
</dc:creator>
```

**Unordered Arrays (Bag):**
```xml
<dc:subject>
  <rdf:Bag>
    <rdf:li>keyword1</rdf:li>
    <rdf:li>keyword2</rdf:li>
  </rdf:Bag>
</dc:subject>
```

**Structures (nested properties):**
```xml
<photoshop:Headline>
  <rdf:Description>
    <photoshop:City>Berlin</photoshop:City>
    <photoshop:Country>Germany</photoshop:Country>
  </rdf:Description>
</photoshop:Headline>
```

### 5.2 Complete RDF/XML Structure

```xml
<?xml version="1.0" encoding="UTF-8"?>
<rdf:RDF xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#"
         xmlns:dc="http://purl.org/dc/elements/1.1/"
         xmlns:xmp="http://ns.adobe.com/xap/1.0/"
         xmlns:xmpMM="http://ns.adobe.com/xap/1.0/mm/">
  <rdf:Description rdf:about=""
        dc:title="Audio Title"
        dc:creator="Creator Name"
        xmp:CreatorTool="Audio Editor">
    <dc:subject>
      <rdf:Bag>
        <rdf:li>Tag1</rdf:li>
        <rdf:li>Tag2</rdf:li>
      </rdf:Bag>
    </dc:subject>
  </rdf:Description>
</rdf:RDF>
```

---

## 6. XMP CORE PROPERTIES (XAP NAMESPACE)

### 6.1 xap: Core Properties

| Property | Type | Description | Example |
|----------|------|-------------|---------|
| xmp:CreatorTool | Text | Tool that created the file | "Adobe Audition 24.0" |
| xmp:CreateDate | Date | Creation date/time | "2024-03-15T14:30:00Z" |
| xmp:ModifyDate | Date | Last modification date | "2024-03-16T10:00:00Z" |
| xmp:MetadataDate | Date | Last metadata modification | "2024-03-16T10:00:00Z" |
| xmp:CreatorTool | Text | Application name | "FFmpeg" |

### 6.2 xmpMM: Media Management Properties

| Property | Type | Description | Example |
|----------|------|-------------|---------|
| xmpMM:DocumentID | URI | Unique document identifier | "xmp.did:123456..." |
| xmpMM:OriginalDocumentID | URI | Original document reference | "uuid:original-uuid" |
| xmpMM:InstanceID | URI | Specific instance identifier | "xmp.iid:instance-id" |
| xmpMM:DerivedFrom | URI | Source document reference | "xmp.did:source-id" |
| xmpMM:History | Bag of structured text | Editing history | See 6.3 |

### 6.3 xmpMM:History — Edit History Tracking

The `xmpMM:History` property records the editing history for tamper detection:

```xml
<xmpMM:History>
  <rdf:Seq>
    <rdf:li rdf:parseType="Resource">
      <stEvt:action>created</stEvt:action>
      <stEvt:instanceID>xmp.iid:123456</stEvt:instanceID>
      <stEvt:when>2024-03-15T14:30:00Z</stEvt:when>
      <stEvt:softwareAgent>Adobe Audition 24.0</stEvt:softwareAgent>
    </rdf:li>
    <rdf:li rdf:parseType="Resource">
      <stEvt:action>saved</stEvt:action>
      <stEvt:instanceID>xmp.iid:789012</stEvt:instanceID>
      <stEvt:when>2024-03-16T10:00:00Z</stEvt:when>
      <stEvt:softwareAgent>Adobe Audition 24.0</stEvt:softwareAgent>
      <stEvt:changed>/metadata</stEvt:changed>
    </rdf:li>
  </rdf:Seq>
</xmpMM:History>
```

---

## 7. AUDIO-SPECIFIC XMP PROPERTIES

### 7.1 Custom Audio Schemas

For audio-specific metadata, custom namespaces are used. The EBU has defined an audio extension:

**EBU Core (ebucore) Namespace:**
```
xmlns:ebucore="http://www.ebu.ch/metadata/ebucore/"
```

### 7.2 EBU Core Audio Properties

| Property | Description | Example |
|----------|-------------|---------|
| ebucore:title | Audio title | "Program Title" |
| ebucore:creator | Creator/artist | "Orchestra Name" |
| ebucore:description | Description | "Concert recording" |
| ebucore:date | Date | "2024-03-15" |
| ebucore:duration | Duration | "PT1H30M0S" (ISO 8601) |
| ebucore:genre | Genre | "Classical" |
| ebucore:identifier | Identifier | "ISRC:..." |

### 7.3 LOM (Learning Object Metadata) for Audio

IEEE 1484.12.1 defines educational metadata that can be used in XMP:

```xml
<lom xmlns="http://ltsc.ieee.org/xsd/LOM">
  <general>
    <title>
      <string>Audio Lecture Title</string>
    </title>
    <description>
      <string>Educational audio content description</string>
    </description>
  </general>
  <technical>
    <duration>
      <duration>PT0H45M0S</duration>
    </duration>
  </technical>
</lom>
```

### 7.4 Complete Audio XMP Example

```xml
<?xpacket begin="" id="W5M0MpCehiHzreSzNTczkc9d"?>
<x:xmpmeta xmlns:x="adobe:ns:meta/"
           xmlns:dc="http://purl.org/dc/elements/1.1/"
           xmlns:xmp="http://ns.adobe.com/xap/1.0/"
           xmlns:xmpMM="http://ns.adobe.com/xap/1.0/mm/"
           xmlns:ebucore="http://www.ebu.ch/metadata/ebucore/">
  <rdf:RDF xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#">
    <rdf:Description rdf:about="">
      <dc:title>
        <rdf:Alt>
          <rdf:li xml:lang="x-default">Symphony No. 5 in C minor, Op. 67</rdf:li>
        </rdf:Alt>
      </dc:title>
      <dc:creator>
        <rdf:Seq>
          <rdf:li>Ludwig van Beethoven</rdf:li>
        </rdf:Seq>
      </dc:creator>
      <dc:description>
        <rdf:Alt>
          <rdf:li xml:lang="x-default">Live recording from Vienna Konzerthaus, March 2024</rdf:li>
        </rdf:Alt>
      </dc:description>
      <dc:subject>
        <rdf:Bag>
          <rdf:li>classical</rdf:li>
          <rdf:li>symphony</rdf:li>
          <rdf:li>orchestra</rdf:li>
          <rdf:li>beethoven</rdf:li>
        </rdf:Bag>
      </dc:subject>
      <dc:date>2024-03-15</dc:date>
      <dc:rights>© 2024 Classical Records Ltd. All Rights Reserved.</dc:rights>
      <dc:format>audio/wav</dc:format>
      <dc:identifier>ISRC:DEA620400001</dc:identifier>
      <xmp:CreatorTool>Adobe Audition 24.0</xmp:CreatorTool>
      <xmpMM:DocumentID>xmp.did:3a4b5c6d-7e8f-9012-3456-7890abcdef12</xmpMM:DocumentID>
      <xmpMM:InstanceID>xmp.iid:3a4b5c6d-7e8f-9012-3456-7890abcdef12</xmpMM:InstanceID>
      <xmpMM:History>
        <rdf:Seq>
          <rdf:li rdf:parseType="Resource">
            <stEvt:action>created</stEvt:action>
            <stEvt:when>2024-03-15T14:30:00Z</stEvt:when>
            <stEvt:softwareAgent>Adobe Audition 24.0</stEvt:softwareAgent>
          </rdf:li>
        </rdf:Seq>
      </xmpMM:History>
      <ebucore:duration>PTS1H7M28S</ebucore:duration>
    </rdf:Description>
  </rdf:RDF>
</x:xmpmeta>
<?xpacket end="w"?>
```

---

## 8. XMP STORAGE IN SPECIFIC FORMATS

### 8.1 XMP in JPEG Files

XMP is stored in JPEG files using the APP1 marker segment with a specific header:

```
Offset  Bytes   Field Name              Type        Description
------  ------  ------------------     ----------  ----------------------------------
0x00    2       APP1 Marker             uint16 BE   0xFFE1
0x02    2       APP1 Length             uint16 BE   Segment length including header
0x04    29      XMP Header              string      "http://ns.adobe.com/xap/1.0/"
0x21    n       XMP Packet              binary      Full XMP packet (without xpacket wrapper)
```

**Maximum XMP in JPEG:** Typically limited by the JPEG segment size (65535 bytes maximum for APP1).

### 8.2 XMP in PDF Files

XMP is stored in PDF files as a metadata stream object:

```
/Type /Metadata
/Subtype /XML
/Length [stream length]
stream
[Full XMP packet]
endstream
```

### 8.3 XMP in TIFF Files

XMP is stored in TIFF as an IFD entry with tag 700 (XMP):

```
Tag ID:    700 (0x02BC)
Type:      BYTE or ASCII
Count:     Length of XMP packet
Value:     Offset to XMP packet data
```

### 8.4 XMP in PNG Files

XMP is stored in PNG as a text chunk with keyword "XML:com.adobe.xmp":

```
Chunk Type: tEXt
Keyword:    "XML:com.adobe.xmp"
Text:       [Full XMP packet]
```

### 8.5 XMP in PostScript/PDF

PostScript files can embed XMP in a stream dictionary:
```postscript
[ /Metadata << /Type /Metadata /Subtype /XML /Length 1234 >> stream
  (XMP packet) ...
>>
```

---

## 9. FFMPEG AND XMP

### 9.1 FFmpeg XMP Support Limitations

**Critical Note:** FFmpeg and ffprobe have **very limited XMP support**:

```bash
# FFmpeg does NOT natively read XMP
ffprobe -v quiet -show_format input.mp3 | grep XMP   # Will not work

# FFmpeg does NOT write XMP natively
ffmpeg -i input.wav -metadata xmp="..." output.mp3  # Not supported
```

### 9.2 What FFmpeg Can Do

| Operation | Support | Notes |
|-----------|--------|-------|
| Read standard tags | Yes | Via -show_format |
| Write standard tags | Yes | Via -metadata |
| Read XMP | No | Requires exiftool |
| Write XMP | No | Requires exiftool |
| Preserve XMP on copy | Partial | -c copy may preserve |

### 9.3 FFmpeg XMP Workflow

Since FFmpeg doesn't natively support XMP, use exiftool for XMP operations:

```bash
# Step 1: Use FFmpeg for audio conversion
ffmpeg -i input.wav -c:a pcm_s24le output.wav

# Step 2: Use exiftool to copy XMP metadata
exiftool -TagsFromFile source.wav -XMP output.wav

# Step 3: Verify XMP
exiftool -XMP output.wav
```

### 9.4 Preserving XMP Through Conversion

```bash
# Convert audio while preserving XMP
ffmpeg -i input.wav -c:a flac output.flac

# Copy XMP from original
exiftool -TagsFromFile input.wav -XMP output.flac

# Or copy all metadata including XMP
exiftool -TagsFromFile input.wav -all:all output.flac
```

---

## 10. EXIFTOOL AND XMP

### 10.1 exiftool XMP Reading

```bash
# Read all XMP properties
exiftool -XMP input.mp3

# Read specific namespace
exiftool -XMP:dc:title input.mp3
exiftool -XMP:dc:creator input.mp3
exiftool -XMP:xmpMM:History input.mp3

# Read as JSON
exiftool -XMP -json input.mp3

# Read from specific format
exiftool -XMP input.wav
exiftool -XMP input.flac
exiftool -XMP input.m4a
exiftool -XMP input.aiff
```

### 10.2 exiftool XMP Writing

```bash
# Write XMP property
exiftool -XMP:dc:title="New Title" input.mp3

# Write multiple XMP properties
exiftool -XMP:dc:title="Title" \
         -XMP:dc:creator="Creator" \
         -XMP:dc:description="Description" \
         input.mp3

# Write full XMP packet from file
exiftool -XMP<metadata.xmp input.mp3

# Write XMP from another file
exiftool -TagsFromFile source.mp3 -XMP target.mp3
```

### 10.3 exiftool XMP Structure Writing

```bash
# Write array properties
exiftool -XMP:dc:subject+="new keyword" input.mp3

# Write localized property
exiftool -XMP:dc:title="Title" -XMP:dc:title:de="Titel" input.mp3

# Write structured property
exiftool -XMP:xmpMM:History='<rdf:Seq>...</rdf:Seq>' input.mp3
```

### 10.4 exiftool XMP Extraction

```bash
# Extract full XMP packet to file
exiftool -XMP -b input.mp3 > metadata.xmp

# Extract and view
exiftool -XMP -b input.mp3 | head -50

# Extract XMP from specific format
exiftool -XMP -w %f.xmp input.mp3  # Create sidecar file
```

---

## 11. TOOL REFERENCE

### 11.1 Tool Support Matrix

| Tool | XMP Read | XMP Write | Format Support |
|------|----------|-----------|----------------|
| exiftool | Full | Full | MP3, MP4, FLAC, WAV, AIFF, PDF, JPEG, PNG |
| FFmpeg | No | No | No native support |
| mediainfo | Limited | No | MP4, some others |
| kid3 | Partial | Partial | Via plugins |
| Adobe products | Full | Full | Native |
| libxmp | Yes [NEEDS VERIFICATION] | Yes [NEEDS VERIFICATION] | Various |

### 11.2 exiftool XMP Tags Reference

| Tag | Namespace | Description |
|-----|-----------|-------------|
| XMP | All | Full XMP packet |
| dc:title | Dublin Core | Title |
| dc:creator | Dublin Core | Creator/author |
| dc:description | Dublin Core | Description |
| dc:subject | Dublin Core | Keywords/subject |
| dc:rights | Dublin Core | Rights |
| dc:date | Dublin Core | Date |
| xmp:CreatorTool | XMP Basic | Creating application |
| xmpMM:DocumentID | XMP Media Management | Document ID |
| xmpMM:History | XMP Media Management | Edit history |
| photoshop:DateCreated | Photoshop | Creation date |

### 11.3 Limitations in Audio Files

| Format | XMP Support | Notes |
|--------|-------------|-------|
| MP3 | Good | Via ID3v2 PRIV frame |
| MP4/M4A | Good | Via uuid atom (Adobe convention) |
| WAV | Limited | Non-standard LIST INFO XMP |
| FLAC | Good | Via Vorbis Comment |
| OGG | Good | Via Vorbis Comment |
| AIFF | Good | Via ID3v2 at start |
| Opus | Limited | Not standardized |

---

## 12. XMP TAMPERING DETECTION

### 12.1 xmpMM:History Analysis

The `xmpMM:History` property records all editing operations:

```xml
<xmpMM:History>
  <rdf:Seq>
    <rdf:li rdf:parseType="Resource">
      <stEvt:action>created</stEvt:action>
      <stEvt:instanceID>xmp.iid:original-id</stEvt:instanceID>
      <stEvt:when>2024-03-15T10:00:00Z</stEvt:when>
      <stEvt:softwareAgent>Source App</stEvt:softwareAgent>
    </rdf:li>
    <rdf:li rdf:parseType="Resource">
      <stEvt:action>converted</stEvt:action>
      <stEvt:when>2024-03-16T14:00:00Z</stEvt:when>
      <stEvt:softwareAgent>FFmpeg</stEvt:softwareAgent>
      <stEvt:parameters>From=flac To=wav</stEvt:parameters>
    </rdf:li>
  </rdf:Seq>
</xmpMM:History>
```

### 12.2 Tampering Indicators

| Indicator | Meaning | Investigation |
|-----------|---------|---------------|
| Missing xmpMM:History | Stripped or old software | Verify with original |
| Gap in timestamps | Missing edits | Review timeline |
| Modified instanceID | Re-saved from copy | Compare IDs |
| Unknown softwareAgent | Third-party modification | Trace provenance |
| Deleted entries | Metadata editing | Compare with backup |

### 12.3 Provenance Tracking

```bash
# Extract full history
exiftool -XMP:xmpMM:History -XMP:xmpMM:DerivedFrom input.mp3

# Compare two files
exiftool -XMP source.mp3 > source_xmp.txt
exiftool -XMP target.mp3 > target_xmp.txt
diff source_xmp.txt target_xmp.txt
```

---

## 13. CROSS-APPLICATION PORTABILITY

### 13.1 XMP Promise

XMP's primary design goal is metadata portability across applications:

```
Source App          Target App
   │                    │
   │  XMP packet        │
   ├────────────────────►│
   │                    │ Parse XMP
   │                    │ Store in native format
   │                    │ Generate new XMP
   │◄───────────────────┤
   │  Updated XMP       │
   │                    │
```

### 13.2 Limitations in Audio

| Limitation | Description | Mitigation |
|------------|-------------|------------|
| Non-standard audio storage | WAV, OGG use non-standard XMP | Use standardized formats |
| Tool support varies | Not all audio tools support XMP | Use exiftool as fallback |
| XMP size | Large packets increase file size | Optimize packet size |
| Encoding | UTF-8 required | Ensure proper encoding |

### 13.3 Best Practices for Audio XMP

1. **Use standardized containers** where possible (MP3 with ID3v2 PRIV)
2. **Prefer EXIFTool** for XMP operations in audio
3. **Include Dublin Core** for basic metadata
4. **Add xmpMM:History** for provenance tracking
5. **Test across tools** before relying on portability

---

## 14. BYTE-LEVEL EXAMPLES

### 14.1 ID3v2 PRIV Frame with XMP (MP3)

```
Offset   Hex                                         ASCII        Description
-------  ------------------------------------------  ----------   ------------------------
0x0000   50 52 49 56                                 PRIV         Frame ID
0x0004   00 01 23 45                                 83977        Frame size (synchsafe)
0x0008   00 00                                       Flags
0x000A   63 6F 6D 2E 61 64 6F 62 65 2E 78 6D 70 00  com.adobe.xmp. Owner ID
0x0016   3C 3F 78 70 61 63 6B 65 74 20 62 65 67...  <?xpacket...  XMP packet
```

### 14.2 MP4 uuid Atom with XMP

```
Offset   Bytes   Field Name              Type        Description
-------  ------  ------------------     ----------  ----------------------------------
0x00     4       Atom Size               uint32 BE   28 + XMP length
0x04     16      UUID                    binary      {BEBA7B1D-3423-417D-ACF6-4FE40F000000}
0x14     n       XMP Packet              binary      Full XMP packet
```

### 14.3 WAV LIST INFO XMP Subchunk

```
Offset   Hex                                         ASCII        Description
-------  ------------------------------------------  ----------   ------------------------
0x0000   4C 49 53 54                                 LIST         LIST chunk
0x0004   XX XX XX XX                                 ...          Size
0x0008   49 4E 46 4F                                 INFO         List type
0x000C   58 4D 50 20                                 XMP          Subchunk ID (XMP )
0x0010   XX XX XX XX                                 ...          Subchunk size
0x0014   3C 3F 78 70 61 63 6B 65 74...               <?xpacket...  XMP packet
```

---

## 15. REFERENCE SPECIFICATIONS

### 15.1 Primary Standards

| Standard | Description | URL |
|----------|-------------|-----|
| ISO 16684-1:2019 | XMP Part 1: Data model, serialization, core properties | https://www.iso.org/standard/75163.html |
| ISO 16684-2:2014 | XMP Part 2: RELAX NG schema descriptions | https://www.iso.org/standard/57211.html |
| ISO 16684-3:2020 | XMP Part 3: Storage in files | https://www.iso.org/standard/79383.html |
| ISO 79384 | JSON-LD serialization of XMP | https://www.iso.org/standard/79384.html |

### 15.2 Adobe Documentation

| Document | Description | URL |
|----------|-------------|-----|
| XMP Specification Part 1 | Main XMP documentation | https://developer.adobe.com/xmp/docs/xmp-specification/ |
| XMP Part 2 | Additional properties | https://developer.adobe.com/xmp/docs/xmp-specification/ |
| XMP Part 3 | Storage in files | https://developer.adobe.com/xmp/docs/xmp-specification/ |

### 15.3 Dublin Core

| Document | Description | URL |
|----------|-------------|-----|
| DCMI Terms | Dublin Core Metadata Initiative | https://www.dublincore.org/specifications/dublin-core/dcmi-terms/ |
| DC Elements 1.1 | Original Dublin Core elements | http://purl.org/dc/elements/1.1/ |

### 15.4 Implementation Resources

| Resource | Description | URL |
|----------|-------------|-----|
| Adobe XMP Toolkit | Open-source XMP library | https://developer.adobe.com/xmp/docs/xmp-toolkit/ |
| exiftool | Universal metadata tool | https://exiftool.org/ |
| XMP SDK | Adobe's reference implementation | Available from Adobe |

---

## 16. IMPLEMENTATION CHECKLIST

### 16.1 Reading XMP from Audio Files

- [ ] Identify file format
- [ ] Locate XMP storage location (format-specific)
- [ ] Parse XMP packet wrapper
- [ ] Parse RDF/XML content
- [ ] Extract Dublin Core properties
- [ ] Extract XMP Basic properties
- [ ] Extract XMP Media Management properties
- [ ] Extract format-specific properties
- [ ] Handle encoding issues
- [ ] Validate XML structure

### 16.2 Writing XMP to Audio Files

- [ ] Create XMP packet structure
- [ ] Add Dublin Core properties
- [ ] Add XMP Basic properties (CreatorTool, etc.)
- [ ] Add xmpMM:History entry
- [ ] Serialize to RDF/XML
- [ ] Wrap in xpacket header/footer
- [ ] Store in format-specific location
- [ ] Update file as needed

### 16.3 Format-Specific Implementation

**For MP3:**
- [ ] Create ID3v2 PRIV frame
- [ ] Set owner to "com.adobe.xmp"
- [ ] Store full XMP packet
- [ ] Update frame size (synchsafe)

**For MP4/M4A:**
- [ ] Create uuid atom
- [ ] Use Adobe UUID: {BEBA7B1D-3423-417D-ACF6-4FE40F000000}
- [ ] Store full XMP packet
- [ ] Place atom in proper location

**For WAV:**
- [ ] Create LIST INFO chunk
- [ ] Add 'XMP ' subchunk
- [ ] Store full XMP packet
- [ ] Update chunk sizes

**For FLAC/OGG:**
- [ ] Add Vorbis Comment field "XMP"
- [ ] Store full XMP packet as single line
- [ ] Escape special characters if needed

### 16.4 XMP Validation

- [ ] Validate XML well-formedness
- [ ] Validate namespace URIs
- [ ] Check property value types
- [ ] Verify required properties
- [ ] Check for duplicate properties
- [ ] Test round-trip (read/write/read)

---

## 17. COMMON PATTERNS

### 17.1 Minimal Audio XMP Packet

```xml
<?xpacket begin="" id="W5M0MpCehiHzreSzNTczkc9d"?>
<x:xmpmeta xmlns:x="adobe:ns:meta/">
  <rdf:RDF xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#"
           xmlns:dc="http://purl.org/dc/elements/1.1/">
    <rdf:Description rdf:about="">
      <dc:title>Track Title</dc:title>
      <dc:creator>Artist Name</dc:creator>
    </rdf:Description>
  </rdf:RDF>
</x:xmpmeta>
<?xpacket end="w"?>
```

### 17.2 Full Audio XMP Packet with History

```xml
<?xpacket begin="" id="W5M0MpCehiHzreSzNTczkc9d"?>
<x:xmpmeta xmlns:x="adobe:ns:meta/"
           xmlns:dc="http://purl.org/dc/elements/1.1/"
           xmlns:xmp="http://ns.adobe.com/xap/1.0/"
           xmlns:xmpMM="http://ns.adobe.com/xap/1.0/mm/"
           xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#">
  <rdf:RDF>
    <rdf:Description rdf:about=""
         dc:title="Album Title"
         dc:creator="Artist"
         dc:date="2024"
         xmp:CreatorTool="ExifTool">
      <dc:subject>
        <rdf:Bag>
          <rdf:li>rock</rdf:li>
          <rdf:li>album</rdf:li>
        </rdf:Bag>
      </dc:subject>
      <xmpMM:History>
        <rdf:Seq>
          <rdf:li rdf:parseType="Resource">
            <stEvt:action>created</stEvt:action>
            <stEvt:when>2024-01-01T00:00:00Z</stEvt:when>
          </rdf:li>
        </rdf:Seq>
      </xmpMM:History>
    </rdf:Description>
  </rdf:RDF>
</x:xmpmeta>
<?xpacket end="w"?>
```

### 17.3 Localized Audio Metadata

```xml
<dc:title>
  <rdf:Alt>
    <rdf:li xml:lang="x-default">Symphony No. 5</rdf:li>
    <rdf:li xml:lang="de">Sinfonie Nr. 5</rdf:li>
    <rdf:li xml:lang="fr">Symphonie No. 5</rdf:li>
    <rdf:li xml:lang="ja">交響曲第5番</rdf:li>
  </rdf:Alt>
</dc:title>
```

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
