# OGG Container Format (OGA) — Deep Technical Reference

> **Category:** Container Format
> **File Extensions:** .ogg, .oga, .ogv, .ogx, .spx
> **MIME Types:** audio/ogg, video/ogg, application/ogg (RFC 5334)
> **Standardization Body:** Xiph.Org Foundation / IETF
> **Specification Documents:** RFC 3533 (Ogg Encapsulation Format Version 0), RFC 5334 (Ogg Media Types), RFC 7845 (Ogg Opus)
> **Patent Status:** Patent-free, open specification
> **License:** Public domain for specification; implementations under BSD/LGPL

---

## 1. HISTORICAL CONTEXT AND ORIGIN

### 1.1 Design Philosophy and Motivations

The OGG container format emerged from the Xiph.Org Foundation's broader project to create a complete, patent-free multimedia codec ecosystem. The project was initiated in the early 1990s by Chris Montgomery, with the goal of developing open-source audio and video codecs that could compete with proprietary formats without licensing encumbrances. OGG itself was conceived as the transport layer for these codecs, providing the framing, synchronization, and multiplexing infrastructure that individual codecs like Vorbis and Theora lacked by design.

The core motivation for OGG was the recognition that most existing container formats—MPEG-4 Part 14 (MP4), AVI, ASF—were either tightly coupled to specific codec ecosystems, burdened by patent claims, or designed primarily for file-based storage rather than streaming. Xiph.Org's architects wanted a format that could handle both live streaming and file storage with equal elegance, support arbitrary codec types without modification, and provide robust error recovery without the complexity of indexed containers like AVI or the overhead of MPEG-2 Transport Stream.

OGG's design philosophy centers on radical simplicity. The entire container specification can be understood by studying a single data structure: the OGG page. There are no optional fields, no alternate encodings, no codec-specific extensions grafted onto the container layer. Every OGG stream is constructed from pages that are byte-identical in structure regardless of whether they carry Vorbis audio, Theora video, Speex speech, Opus interactive audio, or any other codec. This uniformity makes OGG implementations extremely robust and easy to debug.

### 1.2 Standardization History

The OGG format was initially documented through Xiph.Org's open-source reference implementation, libogg, rather than through a formal standards body. As OGG grew in adoption and interoperability concerns arose, the IETF undertook formal documentation. RFC 3533, authored by Silvia Pfeiffer and published in May 2003, formalized the OGG Encapsulation Format Version 0 as an informational RFC. This RFC established the fundamental page structure, the capture pattern, the lacing mechanism, and the logical-to-physical bitstream architecture that defines OGG.

RFC 5334, published in September 2008 by Ivo Goncalves, Pfeiffer, and Montgomery, updated the specification by defining proper IANA media types for OGG-encapsulated content. It introduced three distinct media types—`application/ogg` for generic or multiplexed streams, `video/ogg` for streams with visual content, and `audio/ogg` for audio-primary streams—along with the `.oga`, `.ogv`, `.ogx`, and `.spx` file extensions. RFC 5334 also introduced the `codecs` parameter for MIME type declarations, enabling content negotiation for specific codec payloads within OGG containers.

RFC 7845, published in April 2016 by Timothy Terriberry, Richard Lee, and Ryan Giles, defines the specific OGG encapsulation requirements for the Opus audio codec. This RFC normalizes the handling of Opus headers (OpusHead, OpusTags), specifies the granule position encoding rules for Opus's 48 kHz sampling basis, and defines pre-roll and seeking behavior unique to Opus. RFC 7845 serves as an update to RFC 5334's guidelines for codec-specific OGG mappings.

### 1.3 Ecosystem and Codec Mappings

OGG was designed as a codec-agnostic transport, and the Xiph.Org ecosystem has produced numerous codec mappings over the years. The primary audio codecs that use OGG as their container include Vorbis (the original lossy audio codec, identified by the magic bytes `01vorbis` on the BOS page), Opus (RFC 7845, magic bytes `OpusHead`), Speex (speech codec, magic bytes `Speex   `), and FLAC (free lossless audio codec, magic bytes `\x7fFLAC`). On the video side, Theora (video codec derived from VP3, magic bytes `\x80theora`) and Dirac (wavelet video codec, magic bytes `BBCD\0`) are the primary mappings.

Beyond Xiph.Org's own codecs, the OGG container has been used to encapsulate OggMIDI (MIDI data), Kate (timed text and subtitle overlay), CMML (Continuous Media Markup Language), and various other data types. The format's extensibility means that any codec that can represent its data as a sequence of byte-oriented packets can be encapsulated in OGG without modification to the container specification.

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Core Design Principles

OGG is built on a hierarchy of three conceptual layers: packets, pages, and bitstreams. A **packet** is a logical unit of codec data—a single compressed audio frame, a video frame, or any other meaningful data entity produced by an encoder. OGG imposes no structure on packet contents; packets are opaque byte sequences from the container's perspective. A **page** is OGG's fundamental physical unit—a variable-length chunk of data that wraps one or more packet segments along with a 27-byte header containing framing, timing, and integrity information. A **bitstream** is an ordered sequence of pages belonging to a single logical stream.

This three-layer model is intentionally minimal. Other containers like MP4 use atoms or boxes that can nest, carry internal version numbers, and carry optional metadata. OGG pages are flat, fixed-format structures with no optional fields. Every byte in an OGG page header has a defined meaning and location. This flatness is a deliberate trade-off: it sacrifices flexibility for implementation simplicity and robustness. Parsing an OGG stream requires understanding exactly one data structure.

### 2.2 Physical vs. Logical Bitstreams

A **logical bitstream** is a contiguous sequence of pages belonging to a single codec stream, identified by a unique serial number. An elementary (single-stream) OGG file contains exactly one logical bitstream. A **physical bitstream** is the complete file or stream on disk or wire, which may contain multiple interleaved logical bitstreams. The physical bitstream is what applications actually read and write; the logical bitstreams are reconstructed from it by demultiplexing pages based on their serial numbers.

The relationship between physical and logical bitstreams is central to OGG's multiplexing architecture. When multiple logical bitstreams (for example, a Vorbis audio track and a Theora video track) are combined into a single physical bitstream, their pages are interleaved at the page level. Each individual page still belongs to exactly one logical bitstream, but the physical stream alternates between serial numbers. Demultiplexing is the process of reading the physical stream sequentially, examining each page's serial number, and routing the page to the appropriate logical bitstream decoder.

### 2.3 Streaming vs. File-Based Access

OGG was architected first and foremost for streaming. The format's single-pass design means that an encoder can produce an OGG stream and a decoder can consume it simultaneously, with no need to know the total duration or the complete file structure in advance. This property makes OGG ideal for live broadcasting, VoIP applications, and any scenario where data is generated and consumed in real time.

For file-based access, OGG supports seeking through a combination of coarse-grain and fine-grain mechanisms. The capture pattern (`OggS`) at the start of every page header enables random-access resynchronization from any position in the stream. Granule position values in page headers provide timing information that supports timestamp-based seeking. The format does not mandate an on-disk index structure (unlike MP4's `moov` atom or AVI's `idx1` chunk), but the OGG Skeleton bitstream (a special logical bitstream that can be multiplexed alongside content bitstreams) provides an explicit index for fast seeking.

### 2.4 Granule Position: OGG's Timing Model

Unlike containers that embed explicit millisecond timestamps in their packet headers, OGG uses a **granule position** field in every page header. The granule position is a 64-bit integer whose meaning is entirely codec-defined. OGG itself assigns no meaning to this value—it is interpreted by the encapsulated codec to determine timing, duration, and seek positions.

For audio codecs, the granule position typically represents a sample count. For Vorbis, the granule position encodes the number of PCM samples encoded up to and including the last completed audio packet on that page. For Opus, the granule position encodes PCM samples at 48 kHz (see Section 14 for detailed Opus granule position semantics). For video codecs like Theora, the granule position encodes a frame count or a combination of frame count and field/frame timing information.

The critical property of granule positions is that they are monotonically increasing within a logical bitstream and can be mapped to absolute time through codec-specific conversion. OGG does not specify how this mapping occurs; it is defined by each codec's mapping specification. This design separates container concerns from codec concerns cleanly: OGG handles page ordering and packet boundaries, while the codec handles time representation.

---

## 3. OGG PAGE STRUCTURE — BYTE-LEVEL LAYOUT

### 3.1 Page Anatomy Overview

Every OGG page consists of two parts: a fixed 27-byte page header followed by a variable-length page body. The page body contains a segment table (one byte per segment) and the raw segment data. The maximum page size is 65,535 bytes (0xFFFF), which arises from the constraint that a page can contain at most 255 segments, each with a maximum lacing value of 255 bytes.

```
┌─────────────────────────────────────────────────────────┐
│                   PAGE HEADER (27 bytes)                │
├──────────┬───────────┬──────────────────────────────────┤
│ Capture  │ Version   │ Header                           │
│ Pattern  │  (1 B)    │ Type  (1 B)                     │
│ (4 B)    │           │                                  │
├──────────┴───────────┴──────────────────────────────────┤
│ Granule Position (8 B, little-endian)                   │
├─────────────────────────────────────────────────────────┤
│ Serial Number (4 B, little-endian)                      │
├─────────────────────────────────────────────────────────┤
│ Page Sequence Number (4 B, little-endian)               │
├─────────────────────────────────────────────────────────┤
│ CRC Checksum (4 B, little-endian)                       │
├─────────────────────────────────────────────────────────┤
│ Page Segments (1 B) │ Segment Table (N bytes)            │
└─────────────────────┴──────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│                   PAGE BODY (variable)                  │
│  Segment Data (sum of lacing values bytes)              │
└─────────────────────────────────────────────────────────┘
```

### 3.2 Capture Pattern (Offset 0–3)

Bytes 0 through 3 contain the **capture pattern**, the magic bytes that identify the start of an OGG page. The pattern is the ASCII string `OggS` encoded as four bytes:

| Offset | Size | Value | Description |
|--------|------|-------|-------------|
| 0 | 1 | 0x4F | Character 'O' |
| 1 | 1 | 0x67 | Character 'g' |
| 2 | 1 | 0x67 | Character 'g' |
| 3 | 1 | 0x53 | Character 'S' |

The capture pattern serves as a synchronization marker. When parsing an OGG stream, an implementation scans for the byte sequence `0x4F 0x67 0x67 0x53`. Upon finding this pattern, it can attempt to validate the page by computing and checking the CRC-32 checksum (see Section 3.8). If the CRC fails, the pattern match was a false positive and parsing continues. This design enables OGG to recover from corrupted or malformed streams by simply scanning forward for the next capture pattern.

The capture pattern is case-sensitive; `oggS` or `OGGS` are not valid OGG page markers. The choice of `OggS` (capital S) rather than all-lowercase was deliberate—it creates a byte pattern with good autocorrelation properties for detection algorithms.

### 3.3 Stream Structure Version (Offset 4)

Byte 4 contains the **stream structure version**, a single unsigned byte that identifies the version of the OGG framing specification used by the page. The current and only defined version is 0. Implementations encountering a page with a non-zero version should treat the stream as incompatible:

```python
OGG_VERSION = 0x00

def read_version(data: bytes) -> int:
    return data[4]

def validate_version(version: int) -> None:
    if version != OGG_VERSION:
        raise OggFormatError(
            f"Unsupported OGG version: {version}. "
            f"Only version 0 is defined."
        )
```

This field provides forward compatibility. If a future version of the OGG specification is defined, pages using the new format will have a different version byte. Version 0 decoders can detect the version mismatch and handle the stream appropriately.

### 3.4 Header Type Flag (Offset 5)

Byte 5 contains the **header type flag**, a bitmask that indicates the page's role within its logical bitstream. Each bit has an independent meaning:

| Bit | Mask | Name | Meaning when set |
|-----|------|------|-----------------|
| 0 | 0x01 | Continued packet | Page contains a continuation of a packet from the previous page |
| 1 | 0x02 | BOS (Beginning of Stream) | This is the first page of a logical bitstream |
| 2 | 0x04 | EOS (End of Stream) | This is the last page of a logical bitstream |
| 3–7 | — | Reserved | Must be zero in version 0 |

The BOS and EOS flags are mutually exclusive within a single page. A page cannot be simultaneously the first and last page of a stream (though a single-page stream would have both flags set on that page). The continued-packet flag (0x01) can be combined with either BOS or EOS.

```python
HEADER_TYPE_CONTINUED = 0x01
HEADER_TYPE_BOS      = 0x02
HEADER_TYPE_EOS      = 0x04

def read_header_type(data: bytes) -> int:
    return data[5]

def is_bos_page(header_type: int) -> bool:
    return (header_type & HEADER_TYPE_BOS) != 0

def is_eos_page(header_type: int) -> bool:
    return (header_type & HEADER_TYPE_EOS) != 0

def is_continued(header_type: int) -> bool:
    return (header_type & HEADER_TYPE_CONTINUED) != 0
```

### 3.5 Granule Position (Offset 6–13)

Bytes 6 through 13 form an 8-byte (64-bit) little-endian signed integer representing the **granule position**. The field is encoded in two's complement, which means a value of -1 (all bits set to 1: `0xFF 0xFF 0xFF 0xFF 0xFF 0xFF 0xFF 0xFF`) is a special sentinel indicating that the page contains no completed packets. This sentinel is used for pages that are entirely spanned by a single packet continuing from a previous page.

```python
import struct

GRANULE_POS_NO_PACKET_COMPLETE = -1  # All 64 bits set to 1

def read_granule_position(data: bytes) -> int:
    # 8-byte little-endian signed integer
    return struct.unpack('<q', data[6:14])[0]

def is_no_packet_complete_marker(granpos: int) -> bool:
    return granpos == GRANULE_POS_NO_PACKET_COMPLETE
```

The granule position's meaning is entirely codec-dependent. Implementations must consult the specific codec mapping (Vorbis, Opus, Theora, etc.) to interpret this value correctly. The container layer treats it as an opaque 64-bit integer that is compared for ordering purposes and passed through to the codec layer.

### 3.6 Serial Number (Offset 14–17)

Bytes 14 through 17 contain the **bitstream serial number**, a 4-byte (32-bit) unsigned little-endian integer that uniquely identifies the logical bitstream to which this page belongs. The serial number is randomly generated by the encoder when the logical bitstream is created and must remain constant throughout the stream's lifetime. Within a physical bitstream containing multiple logical bitstreams, every page belonging to the same logical stream carries the same serial number.

```python
def read_serial_number(data: bytes) -> int:
    return struct.unpack('<I', data[14:18])[0]
```

Serial numbers must be unique within a physical bitstream. When multiplexing multiple logical bitstreams, the muxer must ensure that no two streams share the same serial number. A common strategy is to use a cryptographically random 32-bit value. When demultiplexing, an implementation maintains a dictionary mapping serial numbers to logical bitstream decoders, routing each page to the appropriate decoder based on its serial number.

### 3.7 Page Sequence Number (Offset 18–21)

Bytes 18 through 21 contain the **page sequence number**, a 4-byte (32-bit) unsigned little-endian integer. This number increments by one for each page within a single logical bitstream. The sequence number enables detection of missing pages and verification of page ordering:

```python
def read_page_sequence(data: bytes) -> int:
    return struct.unpack('<I', data[18:22])[0]

def check_sequence_continuity(expected: int, actual: int) -> None:
    if actual != expected:
        raise OggSyncError(
            f"Page sequence discontinuity: expected {expected}, "
            f"got {actual}. Possible missing page."
        )
```

The page sequence number is scoped to each logical bitstream independently. That is, two different logical bitstreams each maintain their own sequence number counter, both starting from 0 on their respective BOS pages. The sequence number is the decoder's primary tool for detecting gaps in the page stream—gaps that might indicate packet loss in a streaming context or corruption in a stored file.

### 3.8 CRC Checksum (Offset 22–25)

Bytes 22 through 25 contain the **CRC checksum**, a 4-byte (32-bit) little-endian CRC-32 value computed over the entire page including the header with the checksum field itself zeroed out. The CRC polynomial is the standard IEEE polynomial used in many protocols:

```python
import zlib

CRC_POLYNOMIAL = 0x04C11DB7  # IEEE 802.3 polynomial

def compute_page_crc(page_data: bytes, checksum_offset: int = 22) -> int:
    """
    Compute the CRC-32 checksum for an OGG page.
    The checksum field itself is treated as 0x00000000 during computation.
    """
    # Zero out the checksum field
    working_data = (
        page_data[:checksum_offset] +
        b'\x00\x00\x00\x00' +
        page_data[checksum_offset + 4:]
    )
    return zlib.crc32(working_data) & 0xFFFFFFFF

def verify_page_crc(page_data: bytes) -> bool:
    """
    Verify the CRC checksum of an OGG page.
    Returns True if the page passes integrity check.
    """
    stored_crc = struct.unpack('<I', page_data[22:26])[0]
    computed_crc = compute_page_crc(page_data)
    return stored_crc == computed_crc
```

The CRC covers the entire page, including the header and all segment data, but with the checksum bytes themselves replaced by zeros. This approach ensures that the checksum protects the entire page content while the checksum field itself can be populated after computing the CRC over the zeroed placeholder. The CRC-32 provides a reasonable guarantee against random data corruption, though it is not a cryptographic hash and cannot protect against deliberate tampering.

### 3.9 Page Segments and Segment Table (Offset 26+)

Byte 26 contains the **page segments** field, an unsigned integer (range 1–255) indicating how many segment entries are present in the segment table. Bytes 27 through (26 + page_segments) constitute the **segment table**, a sequence of lacing values where each byte indicates the length of one packet segment.

The segment table immediately precedes the page's segment data in the byte stream. The total page header size is `27 + page_segments` bytes. The total page body size is the sum of all lacing values. The total page size is `27 + page_segments + sum(lacing_values)`.

```python
def read_page_segments(data: bytes) -> int:
    return data[26]

def read_segment_table(data: bytes, num_segments: int) -> list[int]:
    offset = 27
    return list(data[offset:offset + num_segments])

def calculate_page_header_size(num_segments: int) -> int:
    return 27 + num_segments

def calculate_page_body_size(segment_table: list[int]) -> int:
    return sum(segment_table)
```

---

## 4. PACKET SEGMENTATION AND LACING

### 4.1 Segmentation Model

OGG divides codec packets into **segments**—chunks of up to 255 bytes each. This segmentation serves two purposes: it enables OGG pages to have variable length (since a page contains whole segments, and the number of segments per page is variable), and it provides a mechanism for packet boundary recovery without examining the encoded data itself.

The segmentation algorithm is deterministic and lossless. Given a codec packet of arbitrary length N bytes, the encoder produces floor(N / 255) segments of exactly 255 bytes, followed by one final segment of N mod 255 bytes (which may be zero, indicating a 255-byte packet terminated by a lacing value of 0). This guarantees that the original packet can be perfectly reconstructed from its segments.

```python
def segment_packet(packet: bytes) -> list[bytes]:
    """
    Segment a codec packet into OGG-compliant segments.
    Each segment is at most 255 bytes.
    Returns a list of segment byte strings.
    """
    segments = []
    offset = 0
    while offset < len(packet):
        remaining = len(packet) - offset
        segment_size = min(remaining, 255)
        segments.append(packet[offset:offset + segment_size])
        offset += segment_size
    return segments

def packet_size_to_lacing_values(packet_size: int) -> list[int]:
    """
    Given a packet size in bytes, compute the lacing values
    that would represent it. This is the inverse of the
    lacing values -> packet reconstruction.
    """
    lacing_values = []
    remaining = packet_size
    while remaining > 255:
        lacing_values.append(255)
        remaining -= 255
    lacing_values.append(remaining)
    return lacing_values
```

### 4.2 Lacing Values

A **lacing value** is a single byte in the segment table that indicates the size of the corresponding segment. The lacing value also carries packet boundary information through its relationship to 255:

- A lacing value of **255** means the segment is exactly 255 bytes and the packet **continues** on the next segment (or the next page if this is the last segment on the current page).
- A lacing value **less than 255** means the segment is that many bytes and the packet **ends** after this segment.
- A lacing value of **0** indicates an empty segment. An empty segment at the end of a packet is the only way to represent a packet whose size is an exact multiple of 255 bytes (e.g., a 510-byte packet becomes segments of 255 + 255, the second terminated by a lacing value of 0).
- A lacing value of **0** at the start of a page (with no preceding segment continuing from the previous page) indicates a zero-length packet—a legal but unusual occurrence.

```python
LACING_CONTINUATION = 255
LACING_PACKET_END   = 0   # Packet ends with empty segment

def is_packet_continuation(lacing: int) -> bool:
    return lacing == LACING_CONTINUATION

def is_packet_end(lacing: int) -> bool:
    return lacing < LACING_CONTINUATION

def reconstruct_packets_from_page(
    segment_table: list[int],
    page_data: bytes,
    packet_continuation: bool
) -> list[bytes]:
    """
    Reconstruct codec packets from a page's segment table and data.
    
    Args:
        segment_table: List of lacing values from the page header
        page_data: The page body containing segment data
        packet_continuation: True if this page has the continuation flag set
    
    Returns:
        List of reconstructed packets (may include a partial first packet)
    """
    packets = []
    current_packet = b''
    offset = 0
    
    for i, lacing in enumerate(segment_table):
        segment = page_data[offset:offset + lacing]
        offset += lacing
        
        if i == 0 and packet_continuation:
            # First segment continues from previous page
            current_packet += segment
        else:
            # Start or restart a new packet
            if current_packet:
                packets.append(current_packet)
            current_packet = segment
        
        # If lacing < 255, the packet ends here
        if lacing < LACING_CONTINUATION:
            if current_packet:
                packets.append(current_packet)
            current_packet = b''
    
    # If there's leftover data and the page does NOT end with
    # a continuation lacing value (255), it must be an incomplete
    # page that needs the next page to complete the packet
    return packets
```

### 4.3 Page Boundaries and Packet Spanning

A page may contain any number of complete packets, one or more partial packets (packets that continue from a previous page or continue to a subsequent page), or a combination. The constraints are:

1. A page may contain a **continuation segment** only if the page has the `continued_packet` (0x01) flag set.
2. A page may contain a **continuation segment** only as the **first segment** on the page (the segment that continues a packet begun on the previous page).
3. The **last lacing value on a page** determines whether the final packet on the page is complete or continues: a value of 255 means the packet continues onto the next page; a value less than 255 means the packet ends on this page.

This design allows a codec packet of any size to span an arbitrary number of pages. The extreme case is a single 65,025-byte packet (the maximum page body size), which would fill an entire page exactly with 255 segments of 255 bytes each plus one zero lacing value. More realistically, large video keyframes often span multiple pages in Theora streams.

### 4.4 Page Size Constraints

The maximum page size is 65,535 bytes. This arises from the page structure: the page header is at most 282 bytes (27 fixed bytes + 255 segment table entries), and the page body is at most 65,253 bytes (255 segments × 255 bytes). In practice, most OGG implementations target page sizes in the 4–8 KB range for efficiency.

OGG does not specify a minimum page size. A page could contain zero segments (empty page with only a header) if the page_segments field is set to 0. Such pages are unusual but legal; they would be BOS/EOS nil pages used for signaling. The minimum practical page contains at least one segment with a lacing value greater than 0.

---

## 5. LOGICAL BITSTREAM STRUCTURE AND PAGE ORDERING

### 5.1 Beginning of Stream (BOS) Pages

Every logical bitstream begins with a **BOS page** (header type flag 0x02). The BOS page has three mandatory properties:

1. It is the **only page** in the logical bitstream that may carry the BOS flag.
2. It contains the **codec identification packet**—the first packet of the bitstream, which must include sufficient information to identify the exact codec type and determine the stream's timebase.
3. The identification packet **must fit entirely within the BOS page**; it cannot span multiple pages.

The codec identification packet's content is codec-specific and defined by each codec's OGG mapping. For Vorbis, the identification packet begins with the 7-byte header `01vorbis` (packet type 1 followed by the string "vorbis"). For Opus, the identification packet begins with the 8-byte header `OpusHead` (see Section 14). For Speex, it begins with `Speex   ` (8 bytes, space-padded). The common property is that the identification header is always the first packet and always fits in the BOS page.

### 5.2 Header Packets

Following the BOS page, a logical bitstream contains **header packets** that carry codec setup and metadata information. The number and format of header packets is codec-specific:

- **Vorbis** requires exactly three header packets: identification header, comment header, and setup header. These three header packets are placed on their own pages (the identification header on the BOS page, the comment header on the next page or pages, and the setup header on the following page). After the setup header page, all subsequent pages contain audio data packets.
- **Opus** requires exactly two header packets: the identification header (OpusHead) and the comment header (OpusTags). Unlike Vorbis, Opus does not have a separate setup header; codec setup information is embedded directly in the Opus audio packets.
- **Theora** has an identification header, comment header, and setup header similar to Vorbis, plus additional tables specific to video decoding.
- **FLAC** in OGG uses its own framing structure, with the STREAMINFO block on the BOS page followed by metadata blocks on subsequent pages.

The critical constraint is that **all header packets must appear before any data packets** within a logical bitstream, and header packets must be on dedicated pages that do not intermix with data. The final header page must be flushed (i.e., end with a non-continuation lacing value) so that data begins on a fresh page.

### 5.3 End of Stream (EOS) Pages

A logical bitstream ends with an **EOS page** (header type flag 0x04). The EOS page may be a regular data page that happens to be the last, or it may be a **nil page**—a page containing only a header with no segment data (page_segments = 0). Nil EOS pages are useful when the encoder needs to signal end-of-stream at a specific timing point without having any remaining data to flush.

```python
HEADER_TYPE_EOS = 0x04

def is_nil_eos_page(page_data: bytes) -> bool:
    """Check if an EOS page is a nil (empty) page."""
    header_type = page_data[5]
    num_segments = page_data[26]
    return (
        (header_type & HEADER_TYPE_EOS) != 0 and
        num_segments == 0
    )
```

For multiplexed streams, EOS pages of different logical bitstreams need not be contiguous. One logical bitstream may end long before others in the same physical stream. This is the natural behavior when an audio track is shorter than the video track it accompanies—the audio stream's EOS page appears earlier in the physical bitstream, but video pages continue to be muxed afterward.

### 5.4 Page Ordering Requirements

Within a single logical bitstream, pages must appear in **strictly ascending order** by page sequence number and granule position. This is an absolute requirement. OGG does not support out-of-order pages within a logical stream; doing so would break timestamp continuity and seeking.

For multiplexed physical bitstreams, the ordering constraint applies only at the page level within each logical stream. The physical bitstream interleaves pages from multiple logical streams, but each logical stream's pages appear in their own sequential order. Specifically:

- Pages from different logical streams may be interleaved arbitrarily.
- Pages from the same logical stream must appear in ascending sequence order.
- The granule positions within a logical stream must be non-decreasing (some codecs may use equal granule positions for multiple pages, but positions cannot decrease).

---

## 6. PHYSICAL BITSTREAM: MULTIPLEXING, GROUPING, AND CHAINING

### 6.1 Multiplexing (Grouping) Model

OGG's multiplexing operates at the page level. To multiplex N logical bitstreams into a single physical bitstream, the muxer interleaves complete pages from each logical stream. The pages themselves are not modified during multiplexing—a page written by the Vorbis encoder appears byte-for-byte identical in the multiplexed output, merely repositioned within the byte stream.

The key principle is that multiplexing does not alter packet data. The OGG page structure, including the capture pattern, header type, serial number, and CRC, is generated by the OGG framing layer and written along with the packet data. The codec layer is unaware of multiplexing; it simply produces pages with its own serial number. The muxer's job is to select which page to write next from each active logical stream.

```python
class OggMultiplexer:
    """
    Multiplexes multiple Ogg logical bitstreams into a single
    physical bitstream by interleaving pages.
    """
    def __init__(self):
        self.streams: dict[int, OggLogicalStream] = {}
    
    def add_stream(self, serial: int, stream: OggLogicalStream) -> None:
        self.streams[serial] = stream
    
    def mux_next_page(self) -> bytes | None:
        """
        Select the next page to write based on granule position.
        Returns None when all streams are exhausted.
        """
        active_streams = {
            serial: stream
            for serial, stream in self.streams.items()
            if not stream.is_exhausted
        }
        if not active_streams:
            return None
        
        # Select stream with earliest granule position
        selected_serial = min(
            active_streams,
            key=lambda s: active_streams[s].current_granule_position()
        )
        return active_streams[selected_serial].read_next_page()
```

### 6.2 Multiplexing Constraints for Grouped Streams

RFC 3533 specifies strict ordering requirements for grouped (multiplexed) streams:

1. **All BOS pages must appear first**, before any other page types from any logical stream. The BOS pages of all logical streams in a group must appear in consecutive sequence (with respect to each other), with no data pages or auxiliary header pages intervening.
2. **All header pages** (codec headers, comment headers, setup headers) must appear **after all BOS pages** and **before all data pages**.
3. **Data pages** appear after headers and are interleaved based on granule position ordering.
4. **EOS pages** may appear at any time after their logical stream's data is complete; they need not be contiguous with other EOS pages.

This ordering ensures that a demuxer can always determine the codec type of each logical stream before encountering any encoded data, enabling selective decoding (e.g., ignoring a video track and decoding only audio).

### 6.3 Chaining

**Chaining** is the concatenation of complete OGG physical bitstreams. Unlike grouping, which interleaves pages from concurrent logical streams, chaining places entire physical bitstreams end-to-end. The EOS page of one physical bitstream is immediately followed by the BOS page of the next physical bitstream in the chain.

The key constraint for chaining is that **each logical bitstream within a chained physical stream must have a unique serial number across the entire chain**. A muxer cannot reuse a serial number from the first chained segment in a subsequent segment, because demuxers may maintain stream state keyed by serial number. Violating this constraint would cause the demuxer to confuse pages from different logical streams.

```python
def validate_chain_serial_uniqueness(chain: list[OggPhysicalBitstream]) -> None:
    """
    Verify that all logical bitstreams across all chained segments
    have unique serial numbers.
    """
    seen_serials: set[int] = set()
    for segment in chain:
        for page in segment.pages():
            serial = page.serial_number
            if serial in seen_serials:
                raise OggChainingError(
                    f"Duplicate serial number {serial} in chain. "
                    f"Each logical stream must have a unique serial "
                    f"across all chained segments."
                )
            seen_serials.add(serial)
```

Chaining is commonly used for streaming scenarios where discrete programs or segments are broadcast in sequence—a radio station might chain multiple program segments into a single logical broadcast stream without requiring a unified container header.

### 6.4 Combined Grouping and Chaining

OGG supports the combination of grouping and chaining, where a physical bitstream consists of multiple chained segments, each containing multiple grouped logical streams. Each chained segment must individually satisfy all the grouping constraints: within a segment, all BOS pages appear first, headers follow, and then data. When unchained, each resulting physical bitstream must be independently valid.

---

## 7. BINARY FORMAT SPECIFICATION

### 7.1 Complete Page Header Layout

The following table provides the definitive byte-level specification for the OGG page header, referencing RFC 3533's canonical diagram:

| Offset | Size | Field Name | Data Type | Endianness | Description |
|--------|------|-----------|-----------|------------|-------------|
| 0 | 4 | `capture_pattern` | char[4] | — | Fixed: `OggS` (0x4F 0x67 0x67 0x53) |
| 4 | 1 | `stream_structure_version` | uint8 | — | Fixed: 0x00 |
| 5 | 1 | `header_type_flag` | uint8 | — | Bitmask: 0x01=continued, 0x02=BOS, 0x04=EOS |
| 6 | 8 | `granule_position` | int64 | Little | Codec-specific timing value |
| 14 | 4 | `bitstream_serial_number` | uint32 | Little | Logical stream identifier |
| 18 | 4 | `page_sequence_no` | uint32 | Little | Page count within logical stream |
| 22 | 4 | `CRC_checksum` | uint32 | Little | CRC-32 (IEEE) of page with zeroed checksum |
| 26 | 1 | `page_segments` | uint8 | — | Number of segment table entries (1–255) |
| 27 | N | `segment_table` | uint8[N] | — | Array of lacing values, N = page_segments |
| 27+N | M | `segment_data` | uint8[M] | — | Raw packet segments, M = sum(segment_table) |

**Total header size:** `27 + page_segments` bytes
**Total page size:** `27 + page_segments + sum(segment_table)` bytes
**Maximum total page size:** `27 + 255 + (255 × 255) = 65,535` bytes

### 7.2 Multi-Byte Integer Encoding

All multi-byte integer fields in OGG use **little-endian** byte ordering (least significant byte first). This applies to:
- Granule position (8 bytes, signed)
- Serial number (4 bytes, unsigned)
- Page sequence number (4 bytes, unsigned)
- CRC checksum (4 bytes, unsigned)

The bit-order within each byte follows the native platform representation (LSb first within each byte). This is the standard x86/x64 little-endian convention and is why the CRC polynomial 0x04C11DB7 corresponds to the IEEE 802.3 CRC used in Ethernet and many other protocols.

### 7.3 Worked Example: Parsing a Minimal BOS Page

Consider a minimal OGG Vorbis BOS page containing only the 30-byte identification header packet:

```
Offset 00: 4F 67 67 53  -- capture_pattern "OggS"
Offset 04: 00            -- version 0
Offset 05: 02            -- header_type: BOS (0x02)
Offset 06: 00 00 00 00 00 00 00 00  -- granule_position: 0
Offset 14: 1A 2B 3C 4D  -- serial_number: 0x4D3C2B1A (little-endian)
Offset 18: 00 00 00 00  -- page_sequence: 0
Offset 22: XX XX XX XX  -- CRC checksum (computed)
Offset 26: 01            -- page_segments: 1 (one lacing entry)
Offset 27: 1E            -- segment_table[0]: 30 (0x1E bytes)
Offset 28: 01 76 6F 72 ... -- 30 bytes of Vorbis identification header
```

The page header size is `27 + 1 = 28` bytes. The page body contains `30` bytes (one segment of 30 bytes, since 30 < 255, the lacing value is 30, not a continuation). The total page size is `28 + 30 = 58` bytes.

### 7.4 Worked Example: Multi-Segment Packet Spanning Pages

Consider a 512-byte codec packet being wrapped in OGG pages with a target page body size of ~200 bytes:

**Packet segmentation:**
- Segment 1: 200 bytes (lacing = 200)
- Segment 2: 200 bytes (lacing = 200, packet continues)
- Segment 3: 112 bytes (lacing = 112, packet ends)

**Page 1 (first part of packet):**
```
header_type: 0x02 (BOS on first page)
granule_position: 0 (header page)
segment_table: [200, 200]  # Both < 255, so Page 1 contains two
                            # complete segments of the first packet
page_body: 200 + 200 = 400 bytes
```

**Page 2 (second part of packet):**
```
header_type: 0x00 (no flags)
granule_position: 0xFFFFFFFFFFFFFFFF (no packet completes on this page)
segment_table: [112]  # Single segment, < 255, packet ends
page_body: 112 bytes
```

Wait—granule_position on Page 1 is 0 because the BOS page has no completed audio packets. The data page containing the start of the audio data would have its own granule position reflecting the audio timing.

### 7.5 CRC Computation Algorithm

The CRC-32 computation for OGG pages uses the standard IEEE 802.3 CRC-32 polynomial (0x04C11DB7), implemented identically to the CRC-32 used in Ethernet, PNG, and gzip. The key step is that the checksum bytes (offsets 22–25) are treated as zero during computation:

```python
import struct
import binascii

def crc32_page(page_data: bytes) -> int:
    """
    Compute the CRC-32 checksum for an OGG page.
    The CRC is computed over the entire page with the
    4-byte checksum field treated as zeros.
    
    Polynomial: 0x04C11DB7 (IEEE 802.3)
    Initial value: 0x00000000
    Reflected input: No
    Reflected output: No
    """
    # Zero out the checksum field (offset 22, 4 bytes)
    data_for_crc = bytearray(page_data)
    data_for_crc[22] = 0
    data_for_crc[23] = 0
    data_for_crc[24] = 0
    data_for_crc[25] = 0
    
    return binascii.crc32(bytes(data_for_crc)) & 0xFFFFFFFF

def verify_ogg_page(page_bytes: bytes) -> bool:
    """
    Verify an OGG page by checking:
    1. Capture pattern is 'OggS'
    2. Version is 0
    3. CRC-32 matches
    """
    # Check capture pattern
    if page_bytes[0:4] != b'OggS':
        return False
    
    # Check version
    if page_bytes[4] != 0:
        return False
    
    # Check CRC
    stored_crc = struct.unpack('<I', page_bytes[22:26])[0]
    computed_crc = crc32_page(page_bytes)
    
    return stored_crc == computed_crc
```

---

## 8. VORBIS COMMENT AND SETUP HEADERS IN OGG

### 8.1 Vorbis Identification Header

When Vorbis is encapsulated in OGG, the identification header occupies the BOS page and begins with a 7-byte codec identification signature:

```python
VORBIS_ID_HEADER_SIGNATURE = b'\x01vorbis'  # packet_type=1, "vorbis"

def parse_vorbis_id_header(packet_data: bytes) -> dict:
    """
    Parse a Vorbis identification header.
    Packet structure (little-endian):
      1 byte  : packet_type (must be 1)
      6 bytes : 'vorbis' string
      4 bytes : vorbis_version (always 0 for Vorbis I)
      1 byte  : channels (1-8 typical)
      4 bytes : sample_rate (Hz)
      4 bytes : bitrate_upper (bps, 0 if unknown)
      4 bytes : bitrate_nominal (bps, 0 if unknown)
      4 bytes : bitrate_lower (bps, 0 if unknown)
      1 byte  : block_size_0 (samples, power of 2)
      1 byte  : block_size_1 (samples, power of 2)
      1 byte  : framing_flag (must be 1)
    """
    import struct
    
    if packet_data[0:7] != VORBIS_ID_HEADER_SIGNATURE:
        raise ValueError("Not a Vorbis identification header")
    
    offset = 7
    vorbis_version   = struct.unpack('<I', packet_data[offset:offset+4])[0]; offset += 4
    channels        = packet_data[offset]; offset += 1
    sample_rate     = struct.unpack('<I', packet_data[offset:offset+4])[0]; offset += 4
    bitrate_upper   = struct.unpack('<i', packet_data[offset:offset+4])[0]; offset += 4
    bitrate_nominal = struct.unpack('<i', packet_data[offset:offset+4])[0]; offset += 4
    bitrate_lower   = struct.unpack('<i', packet_data[offset:offset+4])[0]; offset += 4
    blocksize_0     = 1 << packet_data[offset]; offset += 1
    blocksize_1     = 1 << packet_data[offset]; offset += 1
    framing_flag    = packet_data[offset]
    
    return {
        'vorbis_version': vorbis_version,
        'channels': channels,
        'sample_rate': sample_rate,
        'bitrate_upper': bitrate_upper,
        'bitrate_nominal': bitrate_nominal,
        'bitrate_lower': bitrate_lower,
        'blocksize_0': blocksize_0,
        'blocksize_1': blocksize_1,
        'framing_flag': framing_flag,
    }
```

### 8.2 Vorbis Comment Header

The Vorbis comment header uses the **Vorbis comment** format (also adopted by Opus, Theora, and Speex), a simple length-prefixed string container. The comment header begins with the 7-byte signature `03vorbis` and contains a vendor string followed by a list of user comment fields:

```python
VORBIS_COMMENT_HEADER_SIGNATURE = b'\x03vorbis'

def parse_vorbis_comment_header(packet_data: bytes) -> dict:
    """
    Parse a Vorbis comment header (also used by Opus, Theora, Speex).
    
    Structure:
      1 byte   : packet_type (must be 3)
      6 bytes  : 'vorbis' string
      4 bytes  : vendor_string_length (little-endian)
      N bytes  : vendor_string (UTF-8)
      4 bytes  : user_comment_list_length (little-endian)
      For each comment:
        4 bytes : comment_length (little-endian)
        M bytes : comment_string (UTF-8, "KEY=value" format)
      1 bit (at end of last comment) : framing_bit (must be 1)
    """
    import struct
    
    if packet_data[0:7] != VORBIS_COMMENT_HEADER_SIGNATURE:
        raise ValueError("Not a Vorbis comment header")
    
    offset = 7
    
    # Vendor string
    vendor_length = struct.unpack('<I', packet_data[offset:offset+4])[0]; offset += 4
    vendor_string = packet_data[offset:offset + vendor_length].decode('utf-8'); offset += vendor_length
    
    # User comment list
    comment_count = struct.unpack('<I', packet_data[offset:offset+4])[0]; offset += 4
    
    comments = []
    for _ in range(comment_count):
        comment_length = struct.unpack('<I', packet_data[offset:offset+4])[0]; offset += 4
        comment = packet_data[offset:offset + comment_length].decode('utf-8', errors='replace')
        comments.append(comment)
        offset += comment_length
    
    return {
        'vendor_string': vendor_string,
        'comments': comments,
    }
```

The Vorbis comment format uses the Vorbis I specification's string encoding rules: strings are UTF-8 encoded, case-insensitive key names (converted to uppercase for comparison per spec), and `=` as the key-value separator. Common tags include `ARTIST`, `TITLE`, `ALBUM`, `DATE`, `TRACKNUMBER`, `GENRE`, and `COMMENT`.

### 8.3 Vorbis Setup Header

The Vorbis setup header is the most complex of the three required headers. It contains the complete codec configuration needed for decoding: codebook definitions (Huffman trees and VQ lookup tables), floor function configurations, residue configurations, mapping configurations, and mode configurations. The setup header begins with the signature `05vorbis`:

```python
VORBIS_SETUP_HEADER_SIGNATURE = b'\x05vorbis'

def parse_vorbis_setup_header(packet_data: bytes) -> dict:
    """
    Parse the Vorbis setup header to extract codec configuration.
    
    The setup header contains, in order:
      - Codebooks (Huffman trees + VQ lookup tables)
      - Time-domain transform placeholders (Vorbis I: always type 0)
      - Floor configurations (type 0 and type 1)
      - Residue configurations (type 0, 1, 2)
      - Mapping configurations (Vorbis I: type 0 only)
      - Mode configurations
    
    The header is decoded bit-by-bit using the Vorbis bitpacking rules.
    The header ends with a framing bit that must be 1.
    """
    if packet_data[0:7] != VORBIS_SETUP_HEADER_SIGNATURE:
        raise ValueError("Not a Vorbis setup header")
    
    # Full decoding requires implementing the Vorbis bitpacking reader
    # and the full header decode specification from the Vorbis I spec.
    # This is a structural outline; actual implementation requires
    # a complete Vorbis bit reader per Section 4.2 of Vorbis I spec.
    raise NotImplementedError(
        "Full setup header parsing requires a complete Vorbis "
        "bitreader implementation per the Vorbis I specification."
    )
```

---

## 9. THEORA VIDEO SYNCHRONIZATION IN OGG

### 9.1 Theora Identification Header

Theora, Xiph.Org's video codec, follows the Vorbis pattern of three header packets for stream identification and setup. The Theora identification header appears on the BOS page and begins with an 8-byte signature:

```python
THEORA_ID_HEADER_SIGNATURE = b'\x80theora'  # 0x80 followed by "theora"

def parse_theora_id_header(packet_data: bytes) -> dict:
    """
    Parse a Theora identification header.
    
    Structure:
      1 byte  : 0x80
      6 bytes: 'theora' string
      3 bytes: vmaj, vmin, vrev (version)
      4 bytes: fmbw (frame width in microblocks)
      3 bytes: fmbh (frame height in microblocks)
      4 bytes: pb (picture base)
      8 bytes: fps_numerator (frames per second numerator)
      8 bytes: fps_denominator
      4 bytes: par_numerator
      4 bytes: par_denominator
      1 byte:  cs (color space)
      1 byte:  ns (nominal bitrate in kbps)
      4 bytes: quality (0-63)
      4 bytes: kfgshift (keyframe foreground shift)
    """
    if packet_data[0:7] != THEORA_ID_HEADER_SIGNATURE:
        raise ValueError("Not a Theora identification header")
    
    import struct
    offset = 7
    vmaj  = packet_data[offset]; offset += 1
    vmin  = packet_data[offset]; offset += 1
    vrev  = packet_data[offset]; offset += 1
    
    # Frame dimensions in pixels
    fmbw  = struct.unpack('<I', b'\x00' + packet_data[offset:offset+3])[0]; offset += 3
    fmbh  = struct.unpack('<I', b'\x00' + packet_data[offset:offset+3])[0]; offset += 3
    
    # Keyframe frequency shift (2^kfgshift)
    kfgshift = struct.unpack('<I', packet_data[offset:offset+4])[0]; offset += 4
    
    return {
        'version': f'{vmaj}.{vmin}.{vrev}',
        'frame_width_mb': fmbw,
        'frame_height_mb': fmbh,
        'frame_width': fmbw * 16,
        'frame_height': fmbh * 16,
        'keyframe_frequency_shift': 1 << kfgshift,
    }
```

### 9.2 Theora Granule Position and Video Synchronization

Theora's granule position encoding is more complex than audio codecs because video has both frame-level timing and the concept of keyframes (INTRA frames) vs. predicted frames. Theora's granule position encodes two pieces of information: the frame count of the last frame decoded before this page, and optionally, information about the keyframe state.

The Theora granule position uses the upper 32 bits for the frame count and the lower 32 bits for the keyframe offset. The codec uses this encoding to support random access: when seeking to a specific time, the decoder must locate the nearest keyframe, decode forward to the target frame, and discard intermediate output.

```python
def decode_theora_granule_position(granpos: int, kfgshift: int) -> dict:
    """
    Decode Theora's packed granule position into frame number and
    keyframe number.
    
    Theora packs: upper 32 bits = iframe + pframe (total frames decoded)
                  lower 32 bits = iframe (keyframe number)
    Where iframe = (granpos >> kfgshift)
          pframe = granpos & ((1 << kfgshift) - 1)
    """
    iframe = granpos >> kfgshift
    pframe = granpos & ((1 << kfgshift) - 1)
    frame_count = iframe + pframe
    
    return {
        'iframe': iframe,
        'pframe': pframe,
        'frame_count': frame_count,
        'is_keyframe': pframe == 0,
    }
```

---

## 10. PAGE GROUPING AND TIMING RECONSTRUCTION

### 10.1 Grouping Model

OGG's page grouping mechanism enables the container to signal logical groupings of related pages—typically audio and video frames that should be decoded and presented together. Unlike MP4's explicit composition offset system or MKV's block-group structure, OGG's grouping is implicit, relying on granule position values to establish temporal relationships between pages from different logical streams.

For a multiplexed OGG audio-video stream, the demuxer reconstructs synchronization by reading the granule positions of completed audio and video pages and aligning them on the time axis. The muxer's job is to interleave pages such that audio and video pages with nearby presentation times are placed close together in the physical stream, minimizing the buffering requirements at the decoder.

### 10.2 Timestamp Derivation

Timestamp derivation from granule positions is codec-specific:

**Vorbis:** The granule position represents the total number of PCM audio samples encoded up to and including the last completed audio packet on that page. Converting to time:

```python
def vorbis_granule_to_time(granpos: int, sample_rate: int) -> float:
    """
    Convert a Vorbis granule position to a presentation timestamp in seconds.
    """
    return granpos / sample_rate

def time_to_vorbis_granule(timestamp: float, sample_rate: int) -> int:
    """
    Convert a timestamp in seconds to a Vorbis granule position.
    """
    return int(timestamp * sample_rate)
```

**Opus:** The granule position is in units of 48 kHz PCM samples (see Section 14). Converting to time:

```python
def opus_granule_to_time(granpos: int) -> float:
    """
    Convert an Opus granule position to a presentation timestamp in seconds.
    Opus granule positions are in units of 1/48000 second.
    """
    return granpos / 48000.0
```

**Theora:** The granule position encodes frame counts (see Section 9.2):

```python
def theora_granule_to_frame(granpos: int, kfgshift: int) -> int:
    """
    Convert a Theora granule position to a frame number.
    """
    iframe = granpos >> kfgshift
    pframe = granpos & ((1 << kfgshift) - 1)
    return iframe + pframe

def theora_frame_to_time(frame: int, fps_numerator: int, fps_denominator: int) -> float:
    """
    Convert a Theora frame number to a presentation timestamp in seconds.
    """
    return frame * fps_denominator / fps_numerator
```

### 10.3 Seeking Strategy

OGG supports seeking through an **interpolated bisection search** that requires no index structure. The algorithm:

1. **Establish bounds:** Start with `lo = 0` (beginning of stream) and `hi = file_size`.
2. **Binary search:** Probe a position roughly midway between `lo` and `hi`.
3. **Capture and decode:** Read forward from the probe position to find the next OggS capture pattern, read that page's granule position, and decode enough data to establish its timestamp.
4. **Narrow bounds:** If the timestamp is before the seek target, move `lo` to the current probe. If after, move `hi`.
5. **Repeat:** Continue bisecting until the interval is small enough (typically within one page).
6. **Verify:** Once a page near the target timestamp is found, backtrack to the nearest keyframe (for video) or the nearest audio packet boundary, then decode forward to the exact target.

This algorithm guarantees O(log n) seeks and requires only the ability to scan forward from any position and read granule positions. The OGG Skeleton bitstream can accelerate this process by providing a precomputed index.

---

## 11. ERROR DETECTION, RESYNCHRONIZATION, AND RECOVERY

### 11.1 CRC-Based Corruption Detection

The CRC-32 checksum provides per-page integrity verification. When a demuxer reads a page, it computes the CRC over the page data (with the checksum field zeroed) and compares it to the stored value. A mismatch indicates that the page was corrupted in storage or transit.

CRC failures trigger different recovery behaviors depending on the page type:

- **Header pages (BOS, initial header packets):** A CRC failure on a BOS page or any required header page is **fatal**. The logical bitstream cannot be identified or initialized without valid headers. The demuxer should treat the stream as corrupted and halt decoding of that logical stream.
- **Data pages:** A CRC failure on a data page is a **non-fatal corruption** event. The demuxer should discard the corrupted page and continue reading. If the page was carrying a critical audio packet, the decoder may produce artifacts or use packet loss concealment if available.
- **Continued-packet pages:** If a page with the continuation flag has a CRC failure, the demuxer should mark the packet as corrupted and ensure the decoder does not attempt to use partial data from the corrupted page.

### 11.2 Capture Pattern Resynchronization

When byte-level parsing loses synchronization (e.g., after seeking to an arbitrary file offset, or after encountering a corrupted page), the demuxer resynchronizes by scanning forward for the next `OggS` capture pattern:

```python
def find_next_ogg_page(file_handle, start_offset: int) -> int | None:
    """
    Scan forward from start_offset looking for the next 'OggS' capture pattern.
    Returns the file offset of the next page, or None if not found.
    
    In practice, implementations use optimized byte scanning:
    - Scan for 0x4F (the first byte of 'OggS')
    - Verify the next three bytes are 0x67, 0x67, 0x53
    - Optionally verify the CRC of the candidate page
    """
    file_handle.seek(start_offset)
    
    window_size = 65536  # Scan in 64KB windows
    while True:
        chunk = file_handle.read(window_size)
        if not chunk:
            return None  # End of file
        
        # Scan for 'O' (0x4F) and verify "OggS"
        for i, byte in enumerate(chunk):
            if byte == 0x4F:
                if chunk[i:i+4] == b'OggS':
                    return start_offset + i
        
        # Move start_offset to just before where we scanned to avoid
        # missing a pattern at the window boundary
        start_offset += len(chunk) - 3
    
    return None
```

This resynchronization mechanism is the foundation of OGG's error recovery. Because every page starts with the same 4-byte capture pattern, a demuxer can always resynchronize by scanning forward. The maximum resynchronization cost is bounded by the maximum page size: a corrupted page can be at most 65,535 bytes, so scanning 128 KB guarantees finding the next valid page (worst case: one full corrupted page followed by the next valid page).

### 11.3 Handling Missing Pages

Missing pages (detected via non-consecutive page sequence numbers) trigger the following recovery logic:

1. For audio codecs: if Opus PLC is available, insert silence packets to fill the gap. Otherwise, mark the gap as corrupted audio.
2. For video codecs: seek backward to the nearest keyframe, decode forward to the target frame, and mark any missing frames as dropped.
3. For multiplexed streams: continue demuxing other logical streams; the missing page only affects the stream with the non-consecutive sequence number.

### 11.4 Truncated and Premature Streams

OGG streams may be truncated due to incomplete file downloads, interrupted streaming, or recording failures. A demuxer should handle truncated streams gracefully:

- If the final page is missing the EOS flag, the demuxer should not treat this as an error condition; the stream simply ended without a proper EOS marker.
- If a packet is continued across the last page and the continuation has no subsequent page, the partial packet should be discarded.
- If a header packet is truncated, the logical stream should be marked as undecodable.

---

## 12. SKELETON METADATA BITSTREAM

### 12.1 Purpose and Structure

The **Ogg Skeleton** is a special logical bitstream that can be multiplexed alongside content bitstreams to provide universal metadata about the physical OGG stream. Skeleton is defined in the document "The Ogg Skeleton Metadata Bitstream" by Pfeiffer and Parker and is recommended by RFC 5334 for content served under all three OGG media types.

The Skeleton bitstream is identified by the magic bytes `fishead\0` on its BOS page. It provides:

1. **Presentation time offset:** The relationship between granule positions and wall-clock time for each logical stream.
2. **Content description:** MIME types, codec names, and technical parameters for each logical stream.
3. **Index:** Optional but recommended, containing seek table entries for fast random access.
4. **UUID identification:** Unique identifier for the content.

### 12.2 Skeleton BOS Page

The Skeleton BOS page contains the `fishead` (field: "fishead\0") identification header:

```python
SKELETON_BOS_SIGNATURE = b'fishead\0'

def parse_skeleton_id_header(packet_data: bytes) -> dict:
    """
    Parse Skeleton identification header.
    
    Structure:
      8 bytes : 'fishead\\0'
      2 bytes : major_version
      2 bytes : minor_version
      4 bytes : presentation_time denominator (ns)
      8 bytes : presentation_time numerator (time base)
      8 bytes : base_date (UTC nanoseconds since epoch)
      4 bytes : num logical streams
    """
    import struct
    
    if packet_data[0:8] != SKELETON_BOS_SIGNATURE:
        raise ValueError("Not a Skeleton identification header")
    
    offset = 8
    vmajor = struct.unpack('<H', packet_data[offset:offset+2])[0]; offset += 2
    vminor = struct.unpack('<H', packet_data[offset:offset+2])[0]; offset += 2
    
    return {
        'version': f'{vmajor}.{vminor}',
        'presentation_denom': struct.unpack('<I', packet_data[offset:offset+4])[0]; offset += 4,
        'presentation_num': struct.unpack('<Q', packet_data[offset:offset+8])[0]; offset += 8,
        'base_date': struct.unpack('<Q', packet_data[offset:offset+8])[0]; offset += 8,
        'num_streams': struct.unpack('<I', packet_data[offset:offset+4])[0]; offset += 4,
    }
```

### 12.3 Skeleton Index

The Skeleton index is a separate logical bitstream (serial number distinct from content streams) that contains seek table entries mapping timestamps to page offsets. For each indexed page, the index entry stores:

- The page's serial number
- The page's granule position
- The page's byte offset within the physical bitstream

```python
SKELETON_INDEX_PACKET_TYPE = 3

def parse_skeleton_index(packet_data: bytes) -> list[dict]:
    """
    Parse Skeleton index packet.
    Returns a list of index entries:
      {
          'serial': int,
          'granule': int,
          'offset': int,
      }
    """
    import struct
    entries = []
    offset = 0
    
    while offset < len(packet_data) - 16:
        serial  = struct.unpack('<I', packet_data[offset:offset+4])[0]; offset += 4
        granule = struct.unpack('<Q', packet_data[offset:offset+8])[0]; offset += 8
        offset_file = struct.unpack('<Q', packet_data[offset:offset+8])[0]; offset += 8
        entries.append({
            'serial': serial,
            'granule': granule,
            'offset': offset_file,
        })
    
    return entries
```

---

## 13. VORBIS IN OGG: DETAILED SPECIFICATION

### 13.1 Packet Organization

Ogg Vorbis organizes its three required header packets as follows:

1. **Identification header** (BOS page, alone): Contains the codec signature `01vorbis`, Vorbis version (always 0 for Vorbis I), audio channel count, sample rate, and bitrate hints.
2. **Comment header** (second page or pages): Contains vendor string and user metadata in Vorbis comment format.
3. **Setup header** (third page): Contains all codec setup data including codebooks, floor configurations, residue configurations, and mode configurations. This header must be fully received before decoding can begin.
4. **Audio data packets** (all subsequent pages): Compressed audio frames.

The critical constraint is that **all three headers must be received and processed before any audio data is decoded**. The setup header, in particular, contains the Huffman codebooks needed for entropy decoding of audio packets; without it, the audio data is indecipherable.

### 13.2 Vorbis Audio Packet Framing

Vorbis audio packets are distinguished from header packets by their first bit. Audio packets have a first bit of `0`, while header packets have a first bit of `1` (specifically, the packet type bytes 1, 3, and 5 all have their MSB set to 1). This bit-level framing enables decoders to identify audio packets without requiring page boundary information:

```python
def is_vorbis_audio_packet(packet_data: bytes) -> bool:
    """
    A Vorbis audio packet has its first bit set to 0.
    Header packets (type 1, 3, 5) have their first bit set to 1.
    """
    return (packet_data[0] & 0x01) == 0

def vorbis_packet_type(packet_data: bytes) -> int:
    """
    Returns the Vorbis packet type:
      1 = identification header
      3 = comment header
      5 = setup header
      even values (including 0) = audio packet
    """
    return packet_data[0] & 0x0F
```

### 13.3 Vorbis Granule Position Semantics

For Vorbis, the granule position encodes the **total number of PCM samples per channel** encoded up to and including the last completed audio packet on the page. The encoding is straightforward: the granule position equals the sum of the block sizes (in samples) of all audio packets that end on that page.

Vorbis supports two block sizes (short and long windows, configured in the identification header). The block size used for each packet is signaled within the packet itself via the mode selection. The granule position for a page is:

```python
def vorbis_page_audio_duration(page_granpos: int, prev_granpos: int) -> int:
    """
    Calculate the number of audio samples represented by a Vorbis page.
    The result is page_granpos - prev_granpos, where prev_granpos
    is the granule position of the previous page.
    """
    return page_granpos - prev_granpos
```

A special case: if a page contains only a partial Vorbis packet (the packet continues to the next page), the granule position is set to -1 (0xFFFFFFFFFFFFFFFF) to indicate that no samples are completed on this page.

---

## 14. OPUS IN OGG: DETAILED SPECIFICATION (RFC 7845)

### 14.1 Opus Identification Header (OpusHead)

RFC 7845 specifies that the Opus identification header begins with the 8-byte magic signature `OpusHead` and is placed alone on the BOS page:

```python
OPUS_HEAD_SIGNATURE = b'OpusHead'

def parse_opus_head(packet_data: bytes) -> dict:
    """
    Parse Opus identification header per RFC 7845.
    
    Structure (all fields little-endian):
      8 bytes : magic signature 'OpusHead'
      1 byte  : version (must be 1)
      1 byte  : channel count 'c' (1-255, MUST NOT be 0)
      2 bytes : pre-skip (unsigned, samples to discard from decoder output)
      4 bytes : input sample rate (unsigned, informational)
      2 bytes : output gain (signed, Q7.8 format, dB)
      1 byte  : channel mapping family (0=mono/stereo, 1=vorbis order, >1=multichannel)
      -- channel mapping table (present if family > 0):
        1 byte  : stream count 'N'
        1 byte  : coupled count 'M'
        N bytes : channel mapping
    """
    import struct
    
    if not packet_data.startswith(OPUS_HEAD_SIGNATURE):
        raise ValueError("Not an OpusHead identification header")
    
    if packet_data[8] != 1:
        raise ValueError(f"Unsupported Opus version: {packet_data[8]}")
    
    offset = 9
    channels = packet_data[offset]; offset += 1
    
    pre_skip = struct.unpack('<H', packet_data[offset:offset+2])[0]; offset += 2
    input_sample_rate = struct.unpack('<I', packet_data[offset:offset+4])[0]; offset += 4
    output_gain = struct.unpack('<h', packet_data[offset:offset+2])[0]; offset += 2
    
    channel_mapping_family = packet_data[offset]; offset += 1
    
    result = {
        'version': packet_data[8],
        'channels': channels,
        'pre_skip': pre_skip,
        'input_sample_rate': input_sample_rate,
        'output_gain_db': output_gain / 256.0,  # Q7.8 format
        'channel_mapping_family': channel_mapping_family,
    }
    
    if channel_mapping_family > 0:
        stream_count = packet_data[offset]; offset += 1
        coupled_count = packet_data[offset]; offset += 1
        channel_mapping = list(packet_data[offset:offset + channels])
        result['stream_count'] = stream_count
        result['coupled_count'] = coupled_count
        result['channel_mapping'] = channel_mapping
    
    return result
```

### 14.2 Opus Comment Header (OpusTags)

The Opus comment header uses the Vorbis comment format with the magic signature `OpusTags`:

```python
OPUS_TAGS_SIGNATURE = b'OpusTags'

def parse_opus_tags(packet_data: bytes) -> dict:
    """
    Parse OpusTags comment header per RFC 7845.
    Uses the same format as Vorbis comment header.
    
    Structure:
      8 bytes : 'OpusTags' signature
      4 bytes : vendor_string_length (little-endian)
      N bytes : vendor_string (UTF-8)
      4 bytes : user_comment_count (little-endian)
      For each comment:
        4 bytes : comment_length (little-endian)
        M bytes : comment_string (UTF-8, "KEY=value" format)
    """
    import struct
    
    if not packet_data.startswith(OPUS_TAGS_SIGNATURE):
        raise ValueError("Not an OpusTags comment header")
    
    offset = 8
    
    vendor_length = struct.unpack('<I', packet_data[offset:offset+4])[0]; offset += 4
    vendor_string = packet_data[offset:offset + vendor_length].decode('utf-8', errors='replace'); offset += vendor_length
    
    comment_count = struct.unpack('<I', packet_data[offset:offset+4])[0]; offset += 4
    
    comments = []
    for _ in range(comment_count):
        comment_length = struct.unpack('<I', packet_data[offset:offset+4])[0]; offset += 4
        comment = packet_data[offset:offset + comment_length].decode('utf-8', errors='replace')
        comments.append(comment)
        offset += comment_length
    
    return {
        'vendor_string': vendor_string,
        'comments': comments,
    }
```

### 14.3 Opus Granule Position Encoding

RFC 7845 defines precise rules for Opus granule position encoding:

1. **BOS and comment header pages:** Granule position MUST be 0.
2. **Audio data pages:** Granule position encodes the total number of PCM samples (at 48 kHz, per channel) that the decoder will output after completing all packets that finish on this page.
3. **Pages where no packet completes:** Granule position is -1 (all 64 bits set to 1).
4. **Pre-skip handling:** The first audio data page's granule position accounts for the pre-skip value from OpusHead. The pre-skip specifies how many samples the decoder should discard from the beginning of its output to account for encoder lookahead and alignment padding.

```python
OPUS_SAMPLE_RATE = 48000  # Opus granule positions are in 48 kHz units

def opus_granule_to_samples(granpos: int) -> int:
    """
    Convert an Opus granule position to PCM sample count.
    Returns the number of PCM samples that will be output
    by the Opus decoder after processing all packets that
    complete on this page.
    """
    return granpos  # Already in 48 kHz samples

def opus_granule_to_time(granpos: int) -> float:
    """
    Convert an Opus granule position to a timestamp in seconds.
    """
    return granpos / OPUS_SAMPLE_RATE

def opus_pre_roll_samples(pre_skip: int, pre_roll_ms: int = 80) -> int:
    """
    Opus requires pre-roll samples before the actual audio.
    The pre_skip value specifies discard samples for decoder state.
    Default pre-roll is 80 ms (3840 samples at 48 kHz).
    """
    return pre_skip + (pre_roll_ms * OPUS_SAMPLE_RATE // 1000)
```

### 14.4 Opus Audio Packet Organization

Each Opus audio packet contains one or more Opus frames (individual encoded audio segments). The packet format uses self-delimiting framing for all but the last frame in a packet, enabling the decoder to determine each frame's boundaries without external context:

- **Single-frame packets:** The frame length is determined by the total packet size minus the TOC byte.
- **Multi-frame packets:** Each frame begins with a self-delimiting length (vorbis-like encoding), allowing the decoder to parse frames independently.

---

## 15. LIBOGG API REFERENCE AND IMPLEMENTATION NOTES

### 15.1 Core libogg Structures

The reference implementation, libogg, provides the canonical API for OGG encoding and decoding. The key structures are:

```c
// ogg_page: represents a single OGG page
typedef struct {
    unsigned char *header;      // page header (27 + nsegments bytes)
    long header_len;             // header length
    unsigned char *body;          // page body (segment data)
    long body_len;               // body length
} ogg_page;

// ogg_stream_state: maintains state for one logical bitstream
typedef struct {
    unsigned char *body_data;    // bytes from packet bodies
    long body_fill;              // bytes used in body_data
    long body_storage;           // bytes allocated in body_data
    
    unsigned char header[27];    // workspace for building page headers
    long header_fill;            // bytes used in header workspace
    
    unsigned char lacing_vals[255];  // lacing values for current page
    long lacing_fill;            // number of lacing entries
    long packet_granulepos;       // granule position for current packet
    
    long sequence;                // page sequence counter
    long serialno;               // stream serial number
    
    int e_o_s;                   // end-of-stream flag
    int b_o_s;                   // beginning-of-stream flag
    int filling;                 // building page vs. flushing
} ogg_stream_state;

// ogg_sync_state: manages the physical bitstream read buffer
typedef struct {
    unsigned char *data;         // sync read buffer
    int storage;                  // allocated buffer size
    int returned;                // bytes returned to caller
    int unsynced;                // never synced (streaming)
    int headerbytes;             // header bytes scanned
    int bodybytes;               // body bytes scanned
} ogg_sync_state;
```

### 15.2 Encoder API Functions

The encoder (packet-in, page-out) API:

```c
// Initialize a stream with a given serial number
int ogg_stream_init(ogg_stream_state *os, int serialno);

// Clear and free a stream
int ogg_stream_destroy(ogg_stream_state *os);

// Submit a packet to the encoding pipeline
// Returns 0 on success, -1 on internal error
int ogg_stream_packetin(ogg_stream_state *os, ogg_packet *op);

// Flush the stream to output a page
// Returns: -1 = needs more data (page not complete)
//           0 = page flushed to output
//           EOF = BOS/EOS flushed and stream is empty
int ogg_stream_pageout(ogg_stream_state *os, ogg_page *og);

// Force-flush the current page (used for BOS, EOS, and header pages)
// Always outputs a page if there's any data
int ogg_stream_flush(ogg_stream_state *os, ogg_page *og);

// Write a page to the output (updates CRC, writes to file)
int ogg_write_page(ogg_page *og, FILE *out);
```

### 15.3 Decoder API Functions

The decoder (page-in, packet-out) API:

```c
// Initialize the sync layer for reading from a byte source
int ogg_sync_init(ogg_sync_state *oy);
int ogg_sync_destroy(ogg_sync_state *oy);

// Fill the read buffer from a file descriptor
// Returns: bytes added, 0 = EOF, -1 = error
long ogg_sync_read(ogg_sync_state *oy, FILE *in, long bytes_to_read);

// Point to the next available page in the buffer
// Returns: pointer to page, or NULL if no page ready
ogg_page *ogg_sync_pageout(ogg_sync_state *oy, ogg_page *og);

// Submit a page to a logical stream
// Returns: 0 = success, -1 = CRC mismatch, 0 = hole in stream
int ogg_stream_pagein(ogg_stream_state *os, ogg_page *og);

// Get the next packet from a logical stream
// Returns: 1 = packet available, 0 = need more pages, -1 = EOS
int ogg_stream_packetout(ogg_stream_state *os, ogg_packet *op);
```

### 15.4 Implementation Considerations

When implementing OGG handling from scratch, several subtle details require attention:

**CRC computation order:** The CRC must be computed over the entire page including the header, but with the 4-byte checksum field replaced by zeros. The checksum itself is then written into those bytes. This means the checksum cannot be computed incrementally—it must cover the complete page.

**Page flushing semantics:** When encoding, the muxer must decide when to flush a page. Flushing too eagerly (after every packet) produces many small pages with high overhead. Flushing too reluctantly (holding data for too long) increases latency and the resynchronization cost after errors. The typical strategy is to flush when the accumulated page body reaches a target size (e.g., 4–8 KB) or when a header packet must be flushed.

**Lacing value accumulation:** The encoder accumulates lacing values as packets are submitted. When the sum of accumulated lacing values would exceed the maximum page body size (65,253 bytes), the encoder must flush the current page and start a new one, even if the current packet is not complete.

**BOS page constraints:** The BOS page must contain the codec identification header entirely within a single page. This is always satisfied because codec identification headers are small (typically under 100 bytes). However, the muxer must explicitly flush the page after writing the BOS header to ensure the identification data is on its own page with the BOS flag set.

**EOS page with no data:** An EOS page may have zero segments (page_segments = 0). In this case, the page header is written with the EOS flag and the CRC is computed over just the 27-byte header. This nil EOS page can be used to signal end-of-stream at a specific timing point without requiring additional data.

---

## 16. FFmpeg OGG DEMUXER AND MUXER IMPLEMENTATION

### 16.1 FFmpeg OGG Demuxer Architecture

FFmpeg's libavformat provides a full OGG demuxer (`libavformat/oggdec.c`) that handles all major codec mappings. The demuxer implements the generic OGG page parsing and delegates codec-specific header parsing and packet decoding to the respective codec parsers.

The demuxer's page parsing flow:

```python
# Pseudocode for the FFmpeg OGG demuxer page processing

def process_ogg_page(ctx, page: OggPage) -> None:
    serial = page.serial_number
    
    if serial not in ctx.streams:
        # New logical stream discovered
        stream = ctx.create_stream(serial)
        
        # BOS page: identify codec
        if page.is_bos:
            codec_id = identify_codec(page.first_packet)
            stream.codec_context = ctx.init_codec(codec_id)
            parse_codec_headers(stream, page)
        else:
            # BOS page missing—stream was truncated before headers
            ctx.remove_stream(serial)
            return
    
    stream = ctx.streams[serial]
    
    # Check page sequence continuity
    if page.sequence != stream.next_sequence:
        ctx.log_warning(
            f"Stream {serial}: missing pages "
            f"{stream.next_sequence}..{page.sequence - 1}"
        )
    
    # Submit page to codec parser
    ctx.codec_parser.parse_page(stream, page)
    
    # Extract packets and add to output queue
    while True:
        packet = ctx.codec_parser.get_packet(stream)
        if packet is None:
            break
        ctx.output_packets.append(packet)
```

### 16.2 FFmpeg OGG Muxer Architecture

The FFmpeg OGG muxer implements the encoding pipeline (packet-in, page-out). For each output stream:

```python
def mux_ogg_stream(ctx, stream: AVStream, pkt: AVPacket) -> None:
    # Wrap the codec packet in an OGG page
    ogg_packet = OggPacket(
        data=pkt.data,
        bytes=pkt.size,
        b_o_s=stream.is_first_packet,
        e_o_s=stream.is_last_packet,
        granulepos=encode_granule_position(stream, pkt),
    )
    
    # Submit to OGG stream encoder
    ogg_stream = ctx.ogg_streams[stream.index]
    ret = libogg.ogg_stream_packetin(ogg_stream, ogg_packet)
    
    # Flush pages as needed
    while True:
        page = OggPage()
        ret = libogg.ogg_stream_pageout(ogg_stream, page)
        if ret == 0:
            break  # No page ready
        
        # Update CRC and write page
        page.crc = compute_ogg_crc(page)
        ctx.write_page(page)
        
        if ret == EOF:
            break  # Stream complete
```

### 16.3 Opus-Specific FFmpeg Handling

FFmpeg's OGG demuxer handles Opus with specific awareness of RFC 7845's requirements:

```python
def ff_opus_parse_id_header(ctx, stream: OggStream, page: OggPage) -> int:
    """
    FFmpeg's Opus-specific BOS page handler.
    Extracts OpusHead fields and initializes the Opus codec context.
    """
    header = parse_opus_head(page.packets[0].data)
    
    ctx.sample_rate = 48000  # Always 48 kHz for Opus
    ctx.channels = header['channels']
    ctx.pre_skip = header['pre_skip']
    
    # Opus uses 48 kHz internally; resampling is done by libswresample
    # if a different output sample rate is requested
    return 0
```

---

## 17. IMPLEMENTATION CHECKLIST

### 17.1 Parser/Demuxer Checklist

Before declaring an OGG demuxer complete, verify each of the following:

- [ ] **Capture pattern detection:** Correctly identifies `0x4F 0x67 0x67 0x53` at the start of every page
- [ ] **Version validation:** Rejects or handles non-zero stream structure version
- [ ] **Header type parsing:** Correctly identifies BOS (0x02), EOS (0x04), and continued (0x01) flags in isolation and combination
- [ ] **Little-endian decoding:** All multi-byte fields are read in little-endian byte order
- [ ] **Granule position handling:** Correctly interprets 64-bit signed granule positions, including the -1 sentinel for incomplete packets
- [ ] **Serial number scoping:** Maintains independent page sequence counters per serial number
- [ ] **Sequence number continuity:** Detects and logs missing pages based on non-consecutive page sequence numbers
- [ ] **CRC-32 verification:** Computes CRC-32 over the complete page with zeroed checksum field
- [ ] **Segment table parsing:** Correctly reads lacing values and reconstructs packet boundaries
- [ ] **Packet reconstruction:** Correctly handles partial packets (continued from previous page) and spanning packets (continuing to next page)
- [ ] **BOS page handling:** Validates that the first page of each logical stream has the BOS flag and contains a valid codec identification header
- [ ] **Header sequencing:** Verifies that all header packets precede data packets within each logical stream
- [ ] **EOS page handling:** Correctly handles EOS flags and nil (empty) EOS pages
- [ ] **Codec identification:** Recognizes magic bytes for Vorbis (`01vorbis`), Opus (`OpusHead`), Speex (`Speex   `), Theora (`\x80theora`), FLAC (`\x7fFLAC`), and Skeleton (`fishead\0`)
- [ ] **Vorbis header parsing:** Correctly parses identification header (version, channels, sample rate), comment header (vendor string, tag list), and setup header structure
- [ ] **Opus header parsing:** Correctly parses OpusHead (channels, pre-skip, output gain, channel mapping) and OpusTags (vendor string, tag list)
- [ ] **Multiplexing:** Correctly demultiplexes interleaved logical streams based on serial numbers
- [ ] **Resynchronization:** Recovers from corrupted pages by scanning forward for the next capture pattern
- [ ] **Truncated stream handling:** Gracefully handles streams without EOS markers and incomplete final packets
- [ ] **Timestamp derivation:** Correctly maps granule positions to timestamps for each codec type (Vorbis: sample_rate; Opus: 48000 Hz; Theora: frame rate from identification header)

### 17.2 Muxer/Encoder Checklist

- [ ] **BOS page generation:** First page of each logical stream has the BOS flag set and contains the complete codec identification header
- [ ] **Header page flushing:** All header pages are flushed before any data pages (header packets end their pages with lacing values < 255)
- [ ] **Serial number uniqueness:** Each logical stream has a unique serial number within the physical bitstream
- [ ] **Packet segmentation:** Packets are correctly segmented into chunks of ≤255 bytes with appropriate lacing values
- [ ] **Page body size:** No page body exceeds 65,253 bytes (255 × 255)
- [ ] **CRC computation:** CRC-32 is computed over the complete page with zeroed checksum field and written in little-endian byte order
- [ ] **Granule position assignment:** Each data page's granule position correctly reflects the timing of completed packets
- [ ] **EOS page generation:** Final page of each logical stream has the EOS flag set; nil EOS pages are used when no data remains to flush
- [ ] **Multiplexing correctness:** Pages from multiple logical streams are interleaved in ascending granule-position order, not round-robin
- [ ] **Continued packet handling:** Pages with continued packets have the continuation flag set and the continued segment appears first
- [ ] **Opus-specific:** OpusHead on BOS page has version=1, channels>0, pre-skip ≥ 3840; OpusTags follows immediately; granule positions are in 48 kHz units
- [ ] **Vorbis-specific:** Three headers (identification, comment, setup) precede all audio data; setup header is fully contained before audio begins
- [ ] **Streaming compatibility:** Muxer can operate in single-pass mode without seeking backward; all decisions are made online

### 17.3 Metadata Handling Checklist

- [ ] **Vorbis comment encoding:** All strings are UTF-8 encoded; key names are compared case-insensitively per spec; vendor string is preserved
- [ ] **Opus tag encoding:** OpusTags uses identical format to Vorbis comments; vendor string and user tags are preserved byte-for-byte
- [ ] **Tag round-trip:** Metadata written by the encoder is bit-identical to metadata read by the decoder for Vorbis/Opus comment headers
- [ ] **Unknown tags:** Tags not recognized by the implementation are preserved as-is in the output (pass-through behavior)
- [ ] **Cover art:** Cover art embedded in comment headers (as MIME-tagged data) is preserved; not converted to separate picture atoms
- [ ] **ReplayGain:** ReplayGain tags in Vorbis comments (REPLAYGAIN_TRACK_GAIN, REPLAYGAIN_ALBUM_GAIN, etc.) are recognized and preserved

### 17.4 Seeking and Random Access Checklist

- [ ] **Coarse seeking:** Scan to approximate target position using byte offset estimation
- [ ] **Capture pattern resynchronization:** Find the nearest `OggS` pattern after any seek
- [ ] **Granule position decode:** Extract and decode granule position from the resynchronized page header
- [ ] **Fine seeking:** Locate the exact target frame by decoding forward or backward from the coarse position
- [ ] **Keyframe alignment:** For video (Theora), seek to the nearest keyframe before decoding forward to the target frame
- [ ] **Pre-roll handling:** For Opus, account for pre-roll samples (default 80 ms) when seeking
- [ ] **Skeleton index usage:** If a Skeleton index is present, use it for O(log n) seeking; fall back to bisection search otherwise
- [ ] **Multi-stream sync:** When seeking in a multiplexed AV stream, locate keyframes in both audio and video streams and decode both forward to the synchronization point

### 17.5 Conformance and Interoperability Checklist

- [ ] **RFC 3533 compliance:** All page structure, byte ordering, and multiplexing rules follow RFC 3533 exactly
- [ ] **RFC 5334 compliance:** MIME types, file extensions, and codec parameter naming follow RFC 5334
- [ ] **RFC 7845 compliance (Opus):** Opus in OGG follows RFC 7845 for OpusHead, OpusTags, and granule position encoding
- [ ] **libogg interoperability:** Files produced by the implementation can be decoded by libogg (and vice versa)
- [ ] **FFmpeg interoperability:** Files produced by the implementation can be decoded by FFmpeg's libavformat OGG demuxer
- [ ] **Reference player compatibility:** Files are playable in at least one reference player (e.g., VLC, Firefox OGG support, libvorbis/libopus command-line decoders)
- [ ] **Corruption resilience:** Demuxer correctly recovers from CRC errors, missing pages, truncated streams, and false-positive capture patterns
- [ ] **Edge cases:** Correctly handles single-page streams, nil BOS/EOS pages, zero-length packets, and packets spanning many pages

---

## 18. REFERENCES AND SPECIFICATIONS

### 18.1 Primary Specifications

| Document | Title | Year | Notes |
|----------|-------|------|-------|
| RFC 3533 | The Ogg Encapsulation Format Version 0 | 2003 | Core OGG framing specification |
| RFC 5334 | Ogg Media Types | 2008 | MIME types, file extensions, IANA registration |
| RFC 7845 | Ogg Encapsulation for the Opus Audio Codec | 2016 | Opus-specific OGG mapping |
| Vorbis I Spec | Vorbis I Specification | 2004 | Vorbis codec bitstream format |
| RFC 6716 | Definition of the Opus Audio Codec | 2012 | Opus codec core specification |
| Ogg Skeleton | The Ogg Skeleton Metadata Bitstream | 2007 | Skeleton format specification |

### 18.2 Reference Implementations

- **libogg:** The reference OGG framing library (BSD license). Provides `ogg_sync_state`, `ogg_stream_state`, `ogg_page`, and `ogg_packet` structures with encoder and decoder APIs. Available at `https://xiph.org/ogg/`
- **libvorbis:** Reference Vorbis encoder/decoder with OGG muxing/demuxing
- **libopus:** Reference Opus encoder/decoder with OGG support
- **libtheora:** Reference Theora encoder/decoder with OGG muxing/demuxing
- **FFmpeg libavformat:** Full OGG demuxer and muxer via `libavformat/oggdec.c` and `libavformat/oggenc.c`

### 18.3 Xiph.Org Resources

- Main OGG documentation: `https://xiph.org/ogg/`
- OGG framing specification: `https://xiph.org/ogg/doc/framing.html`
- OGG stream overview: `https://xiph.org/ogg/doc/oggstream.html`
- Vorbis documentation: `https://xiph.org/vorbis/`
- Opus documentation: `https://opus-codec.org/`
- Codec identifier table: `https://wiki.xiph.org/MIMETypesCodecs`
