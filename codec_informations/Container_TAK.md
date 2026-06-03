# TAK (Tom's Lossless Audio Kompressor) Container — Deep Technical Reference

> **Category:** Container
> **File Extensions:** `.tak`
> **MIME Types:** `audio/x-tak`, `audio/tak`
> **Standardization Body:** None — proprietary but documented by Thomas Becker
> **Primary Specification:** Thomas Becker's documentation (thbeck.de/Tak/Tak.html)
> **Patent Status:** Proprietary — no patents known, but format is closed
> **License:** Freeware — free to use, source not available
> **Current Version:** 2.2.0 (released ~2015)
> **Active Development:** No — development discontinued, last update ~2015

---

## 1. HISTORICAL CONTEXT & ORIGIN

### 1.1 Creation & Motivation
- **Creator(s):** Thomas Becker (individual developer)
- **Year Created:** 2009 (beta); 2011 (v1.0); 2.2.0 released ~2015
- **Original Purpose:** Create a high-performance lossless audio codec that offers excellent compression ratios and very fast encoding/decoding speeds, suitable for audiophile archival use
- **Problem with Predecessors:** FLAC was too slow for some use cases; other lossless formats lacked either speed or compression; TAK aimed to combine the best of both

### 1.2 Version History
| Version | Year | Key Changes |
|---------|------|-------------|
| YALAC (prototype) | ~2009 | Early working title |
| TAK 1.0 | 2011 | Initial public release, basic format |
| TAK 1.1.0 | 2012 | Added MD5 checksum metadata, last frame info |
| TAK 1.1.1 | 2013 | Obsoleted old seek table metadata type |
| TAK 2.0 | 2014 | New codec type (Integer 24-bit 2.0), improved compression |
| TAK 2.1 Beta | 2015 | Added LossyWav codec type |
| TAK 2.2 | 2015 | Added multichannel support (Integer 24-bit MC) |

### 1.3 Current Adoption
- **Primary use cases today:** Audiophile archival, music collection storage, where compression ratio matters more than universal compatibility
- **Platforms with native support:** Windows (TAK application), Linux (via Wine or TAK tools), macOS (via Wine)
- **Major services using this format:** None — niche audiophile format
- **Hardware support:** None — software-only
- **Status:** Niche — discontinued but stable and usable; FFmpeg lacks native TAK support

---

## 2. TECHNICAL ARCHITECTURE OVERVIEW

### 2.1 Codec Family & Algorithm Class
- **Codec type:** Lossless waveform compression
- **Core algorithm:** Adaptive linear prediction (similar to FLAC) combined with entropy coding
- **Loss mechanism:** Lossless only — no psychoacoustic model
- **Frame-based vs sample-based:** Frame-based — variable number of samples per frame
- **Fixed vs variable frame size:** Variable frame sizes (94–250 ms range)

### 2.2 High-Level Encoding Flow (Block Diagram)
```
Input PCM Samples (up to 8 channels)
      │
      ▼
[Frame Splitting: Partition into independent frames]
      │
      ▼
[Linear Prediction: Multi-order adaptive FIR filter]
      │
      ▼
[Entropy Coding: Custom range coder with improved compression]
      │
      ▼
[Frame Size Optimization: Balance compression vs frame count]
      │
      ▼
[CRC: 24-bit CRC per frame for integrity]
      │
      ▼
[Seek Table: Index positions every 1 second]
      │
▼
Output Encoded Stream
```

### 2.3 Key Design Parameters
| Parameter | Value | Notes |
|-----------|-------|-------|
| Algorithmic delay | 94–250 ms per frame | Configurable |
| Frame size | Variable (94–250 ms default) | User-configurable |
| Max channels | 8 (TAK 2.2+) | Monophonic and multichannel |
| Max bit depth | 24-bit | Internal precision varies |
| Max sample rate | 192,000 Hz | Supported |
| Bitrate range | ~400–1400 kbps | Depends on content and preset |
| Complexity | O(n) with high efficiency | Optimized for both speed and compression |

---

## 3. BINARY FORMAT SPECIFICATION

### 3.1 File Signature / Magic Bytes
```
Offset  Bytes   Hex Value        ASCII     Meaning
------  ------  ---------------  --------  -------------------
0x0000  4       74 42 61 4B     'tBaK'   TAK file signature
```

The magic bytes `tBaK` (lower case) identify this as a TAK file.

### 3.2 File-Level Header Layout
```
Full file structure:
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------
0x0000   4B      File Signature          char[4]     'tBaK'
0x0004   N       Metadata Objects         variable    Metadata objects (type + length + data)
           ...      until metadata end
           ...
0xXXXX   N       Audio Data              variable    Compressed audio frames
```

### 3.3 Metadata Object Structure
```
Each metadata object has:
Offset   Size    Field Name              Type        Description
-------  ------  ----------------------  ----------  ---------------------------
0x0000   4B      Object Size            uint32      Size including this header
0x0004   1B      Object Type           uint8       Metadata type (see table)
0x0005   N       Object Data            variable    Type-specific data
```

#### Metadata Object Types
| Type ID | Name | Description | Introduced |
|---------|------|-------------|------------|
| 0x00 | End of Metadata | Marks end of metadata section | v1.0 |
| 0x01 | Stream Info | Audio format, channel count, sample rate, etc. | v1.0 |
| 0x02 | Seek Table (obsolete) | Frame offsets and sample positions | v1.0 |
| 0x03 | Original File Data | Filename and path information | v1.0 |
| 0x04 | Encoder Info | Encoder version, profile/preset | v1.0 |
| 0x05 | Padding | Reserved space for future use | v1.0.3 |
| 0x06 | MD5 Checksum | Full audio data MD5 hash | v1.1.1 |
| 0x07 | Last Frame Info | Offset and sample position of last frame | v1.1.1 |

### 3.4 Stream Info Object (Type 0x01) — Complete Layout
```
Bit Layout (bitstream format, not byte-aligned):
Offset   Bits    Field Name              Description
-------  -----   --------------------  ---------------------------
0        6       Codec Type             0=Integer 24-bit (v1.0), 1=Experimental
                                           2=Integer 24-bit (v2.0), 3=LossyWav
                                           4=Integer 24-bit MC (v2.2)
1        4       Encoder Profile        0=Turbo, 1=Fast, 2=Normal
                                           3=High, 4=Extra, 5=Insane
5        4       Frame Duration        Frame length in ms (actual value varies)
9        35      Sample Count          Total samples per channel
44       4       Frame Size            Frame size code
48       3       Data Type             0=PCM, others=reserved
51       18      Sample Rate           (value + 6000) Hz
69       5       Bits Per Sample       (value + 8) bits
74       4       Audio Channels        (value + 1) channels
78       1       Extension Present     0=no, 1=yes
         if extension present:
79       5       Valid Bits Per Sample
84       1       Speaker Assignment Present
85       6×N     Speaker Assignments   For each channel

Bits 0-3: Codec type (6 bits total)
Bits 4-7: Encoder profile (4 bits)
Bits 8-11: Frame duration (4 bits)
Bits 12-46: Sample count (35 bits)
Bits 47-50: Frame size (4 bits)
Bits 51-53: Data type (3 bits)
Bits 54-71: Sample rate (18 bits) — actual = value + 6000
Bits 72-76: Bits per sample (5 bits) — actual = value + 8
Bits 77-80: Audio channels (4 bits) — actual = value + 1
Bit 81: Extension present flag
```

### 3.5 Encoder Info Object (Type 0x04)
```
Offset   Bits    Field Name              Description
-------  -----   --------------------  ---------------------------
0        8       Version Major          Major version number
8        8       Version Minor          Minor version number
16       16      Version Revision       Revision/build number
32       8       Encoder Application    0=TAK, 1=other
40       8       Operating System       0=Windows, 1=Linux, 2=macOS, etc.
```

### 3.6 Audio Data — Frame Structure
```
Each audio frame:
Offset   Bits    Field Name              Description
-------  -----   --------------------  ---------------------------
0        8       Sync Code              Frame synchronization marker
8        40      Last Frame Position    Byte offset from file start (optional)
48       35      Frame Sample Position  Sample index for this frame (optional)
         N       Frame Data             Entropy-coded audio data
         24      Frame CRC              24-bit CRC for integrity
```

### 3.7 Sample Format Support
| Bit Depth | Type | Supported | Notes |
|-----------|------|-----------|-------|
| 8-bit | Unsigned integer | No | TAK minimum is 16-bit |
| 16-bit | Signed integer | Yes | Primary format |
| 20-bit | Signed integer | Yes | Supported internally |
| 24-bit | Signed integer | Yes | Full precision |
| 32-bit | Signed integer | No | Not supported |
| 32-bit | IEEE float | No | Not supported |

#### Supported Sample Rates
| Rate (Hz) | Common Name | Supported | Notes |
|-----------|-------------|-----------|-------|
| 8000 | Telephone | Yes | |
| 11025 | — | Yes | |
| 16000 | Wideband voice | Yes | |
| 22050 | — | Yes | |
| 32000 | Broadcast | Yes | |
| 44100 | CD audio | Yes | Most common |
| 48000 | Professional | Yes | |
| 88200 | 2× CD | Yes | |
| 96000 | High-res | Yes | |
| 176400 | 4× CD | Yes | |
| 192000 | High-res max | Yes | |

---

## 4. ENCODING ALGORITHM — DEEP DETAIL

### 4.1 Pre-Processing Stage
- **DC offset removal:** Not explicitly performed
- **Channel separation:** Channels are processed together in multichannel mode
- **Sample format:** All internal processing at 24-bit or higher precision

### 4.2 Analysis / Transform Stage

#### Transform Type: Time-Domain Linear Prediction
```
TAK uses multi-order adaptive linear prediction in the time domain.
No frequency-domain transform is used.

Parameters:
  Max prediction order:  32 (configurable per preset)
  Window:                Frame-based (94–250 ms)
  Algorithm:             Adaptive filtering with efficient update
```

### 4.3 No Psychoacoustic Model
TAK is a lossless codec. There is no quality setting — the output is always bit-exact.

### 4.4 Entropy Coding — Custom Range Coder
```
Method: Custom range coder (arithmetic coding variant)

The range coder provides near-optimal compression for the residual signal.
Key properties:
  - Adaptive probability modeling
  - Efficient bit packing
  - Fast encoding and decoding
  - Better compression than Rice coding used by FLAC
```

### 4.5 Encoder Settings / Presets

#### For Lossless Codec
| Preset | Encoding Speed | Decode Speed | Compression | Notes |
|--------|---------------|--------------|-------------|-------|
| Turbo | ~10× realtime | ~10× realtime | Good | Fastest |
| Fast | ~5× realtime | ~8× realtime | Better | Good balance |
| Normal | ~2× realtime | ~5× realtime | Best | Default |
| High | ~1× realtime | ~4× realtime | Better | Slower |
| Extra | ~0.5× realtime | ~3× realtime | Best | Very slow |
| Insane | ~0.1× realtime | ~3× realtime | Maximum | Extremely slow |

---

## 5. DECODING ALGORITHM — DEEP DETAIL

### 5.1 Stream Synchronization & Seeking

#### Sync Strategy
```
1. Read first 4 bytes: 'tBaK' signature
2. Parse metadata objects until End-of-Metadata (type 0x00)
3. Extract Stream Info: channels, sample rate, total samples
4. Parse audio frames sequentially:
     a. Read sync code
     b. Read optional frame position fields
     c. Decode frame data using range decoder
     d. Verify frame CRC
5. Seek using seek table (if present) or frame synchronization
```

#### Seeking
```
TAK supports efficient seeking via:
1. Frame sync codes: Each frame begins with a unique sync pattern
2. Optional seek table: Frame offsets stored as metadata
3. Optional frame position fields: Embedded in each frame header

Seek to time T:
1. Calculate target frame: frame ≈ T × sample_rate / frame_size_samples
2. Use seek table or scan for sync code
3. Decode from nearest keyframe position
```

### 5.2 Core Decode Pipeline
```
1. Read 'tBaK' signature and validate

2. Parse metadata objects:
     ├── Stream Info: extract audio format
     ├── Encoder Info: version, preset
     ├── MD5 Checksum: for verification
     └── End-of-Metadata: stop parsing metadata

3. For each frame:
     ├── Read frame sync code
     ├── Read frame header (position, size)
     ├── Decode entropy-coded data (range decoder)
     ├── Reconstruct prediction residuals
     ├── Apply linear prediction
     └── Verify frame CRC (24-bit)

4. Verify MD5 checksum if present

5. Format output as PCM (16-bit or 24-bit signed integer)
```

### 5.3 Error Concealment
- **Corrupt frame detection:** CRC-24 check on each frame
- **Concealment method:** None — decode stops on error
- **Maximum consecutive errors before silence:** 1

---

## 6. CONTAINER / WRAPPER INTEGRATION

### 6.1 Native Containerization
- **Native container:** TAK — standalone file format
- **Overhead:** Minimal — metadata objects + per-frame CRC
- **Seeking in native container:** Yes — via sync codes and optional seek table
- **Multiple streams in native container:** No — single audio stream only

### 6.2 Codec-to-Container Compatibility
| Container | Can Store This Codec | Seeking | Metadata | Notes |
|-----------|---------------------|---------|----------|-------|
| TAK (.tak) | TAK audio | Yes | Native APEv2, ID3v2 | Native format |
| Matroska/MKA | No | — | — | No support |
| MP4/M4A | No | — | — | No support |
| WAV | No | — | — | Raw PCM only |

---

## 7. METADATA ARCHITECTURE

### 7.1 Native Tag System
- **Default tag system:** APEv2 tags (at end of file) and ID3v2 tags (at beginning)
- **Tag block location:** End of file (APEv2) or beginning (ID3v2)
- **Native TAK metadata:** Encoder info, MD5 checksum embedded in bitstream

### 7.2 Standard Tag Fields — TAK with APEv2
| Tag Field | Internal Key (TAK/APEv2) | Max Length | Character Encoding | Multi-value | Notes |
|-----------|--------------------------|------------|-------------------|-------------|-------|
| Title | Title | variable | UTF-8 | No | |
| Artist | Artist | variable | UTF-8 | No | |
| Album | Album | variable | UTF-8 | No | |
| Album Artist | Album Artist | variable | UTF-8 | No | |
| Composer | Composer | variable | UTF-8 | No | |
| Genre | Genre | variable | UTF-8 | No | |
| Year / Date | Year | 4 bytes | ASCII | No | YYYY format |
| Track Number | Track | variable | UTF-8 | No | Format "N" or "N/Total" |
| Disc Number | Disc | variable | UTF-8 | No | Format "N" or "N/Total" |
| Comment | Comment | variable | UTF-8 | No | |
| Cover Art | Cover Art (front) | variable | Binary | No | APEv2 item |
| ReplayGain Track Gain | REPLAYGAIN_TRACK_GAIN | variable | ASCII | No | "-6.20 dB" |
| ReplayGain Track Peak | REPLAYGAIN_TRACK_PEAK | variable | ASCII | No | "0.998459" |
| ReplayGain Album Gain | REPLAYGAIN_ALBUM_GAIN | variable | ASCII | No | |
| ReplayGain Album Peak | REPLAYGAIN_ALBUM_PEAK | variable | ASCII | No | |

### 7.3 Cover Art Storage in TAK
```
Cover art stored as APEv2 binary item:
  Key: "Cover Art (front)" or custom key
  Flags: 0x01 (binary data)
  Value: Binary image data (JPEG or PNG)
```

### 7.4 Metadata Compatibility Matrix
| Tag System | Read | Write | Round-trip Lossless | Priority |
|------------|------|-------|---------------------|----------|
| Native TAK metadata | ✓ | ✓ | ✓ | Highest |
| APEv2 | ✓ | ✓ | ✓ | High |
| ID3v2 (beginning) | ✓ | ✓ | ✓ | Medium |
| ID3v1 (end) | ✓ | ✓ | ✓ | Low |

---

## 8. FFMPEG IMPLEMENTATION REFERENCE

### 8.1 FFmpeg Identifiers
```
Codec Name (CLI):   tak           # NOT available in most builds
AV_CODEC_ID:        AV_CODEC_ID_TAK  # Defined but decoder may not be compiled
Format Name (CLI):  tak           # May not be available
Encoder(s):         None          # FFmpeg does NOT support TAK encoding
Decoder(s):         tak           # Limited support — may require FFmpeg build
Muxer(s):          tak           # Not available
Demuxer(s):        tak           # Limited support
```

### 8.2 FFmpeg Support Status
```
IMPORTANT: FFmpeg does NOT have native TAK support in most builds.
TAK demuxer and decoder require FFmpeg to be compiled with TAK support enabled.

To check if TAK is supported:
  ffmpeg -decoders | grep tak
  ffmpeg -demuxers | grep tak

If not available, use official TAK tools for encoding/decoding.
```

### 8.3 TAK Official Tools (Recommended)

#### TAK Encoder (Command Line)
```bash
# Encode WAV to TAK (using official TAK tools)
takc -e -p4 input.wav output.tak
#   -e: encode mode
#   -p4: preset 4 (High quality)
#   Presets: -p0=Turbo, -p1=Fast, -p2=Normal, -p3=High, -p4=Extra, -p5=Insane

# Encode with custom frame size
takc -e -p3 -fs200ms input.wav output.tak
#   -fs: frame size in milliseconds

# Verify after encoding
takc -v output.tak
```

#### TAK Decoder (Command Line)
```bash
# Decode TAK to WAV
takc -d input.tak output.wav

# Decode and pipe to another tool
takc -d input.tak - | ffmpeg -i - output.flac
```

### 8.4 FFmpeg Decoding — Only If Available
```bash
# Only works if FFmpeg was built with TAK support
ffmpeg -i input.tak \
  -c:a pcm_s16le \
  output.wav

# Probe format
ffprobe -v quiet -print_format json -show_streams input.tak
```

### 8.5 Quality / Fidelity Decision Table
| Use Case | Settings | Expected Size | Notes |
|----------|----------|----------------|-------|
| Archival | -p5 (Insane) | 50–65% of WAV | Maximum compression |
| General archival | -p3 (High) | 55–70% of WAV | Good balance |
| Space-saving | -p2 (Normal) | 60–75% of WAV | Default |
| Speed priority | -p1 (Fast) | 65–80% of WAV | Fast encoding |

---

## 9. SEEKING & RANDOM ACCESS

### 9.1 Seek Table / Index Structure
```
TAK Seek Table (obsoleted in v1.1.1, but still parsed for backward compatibility):
  Location:    As metadata object (type 0x02) before audio data
  Entry size:  Variable
  Entry format:
    [0x00-0x07]  frame_position (uint64) — byte offset from file start
    [0x08-0x0F]  frame_sample_position (uint64) — sample number
    [0x10-0x13]  frame_size (uint32) — frame size in bytes

Modern TAK files use embedded frame position fields in each frame header instead.
Each frame header optionally contains:
  - Last frame position (40 bits)
  - Frame sample position (35 bits)
```

### 9.2 Gapless Playback Data
```
TAK stores encoder delay and padding in the stream info.
Gapless playback: Supported via encoder delay information
Encoder delay:    Variable, stored in frame headers
Padding:         Variable, stored in frame headers
```

---

## 10. STREAMING & REAL-TIME CONSIDERATIONS

| Property | Value | Notes |
|----------|-------|-------|
| Streamable (decode before full download) | Yes | Frame sync codes enable random access |
| Algorithmic encoder delay | ~100 ms | One frame delay |
| Live encoding feasible | Yes | Fast presets enable real-time |
| HTTP progressive download | Yes | Seeking works with partial file |
| HTTP Live Streaming (HLS) | No | Not segmented |
| WebRTC / RTP transport | No | Not a typical transport |
| Minimum decode buffer | 1 frame | ~100–250 ms |

---

## 11. MULTI-CHANNEL & SURROUND AUDIO

### 11.1 Supported Channel Layouts
| Channels | Layout Name | Channel Order | FFmpeg Layout Constant | Notes |
|----------|-------------|---------------|----------------------|-------|
| 1 | Mono | C | AV_CHANNEL_LAYOUT_MONO | Supported |
| 2 | Stereo | L, R | AV_CHANNEL_LAYOUT_STEREO | Supported |
| 3 | 2.1 | L, R, LFE | AV_CHANNEL_LAYOUT_2POINT1 | Supported (v2.2) |
| 4 | Quad | FL, FR, RL, RR | AV_CHANNEL_LAYOUT_QUAD | Supported (v2.2) |
| 5 | 5.0 | FL, FR, C, BL, BR | AV_CHANNEL_LAYOUT_5POINT0 | Supported (v2.2) |
| 6 | 5.1 | FL, FR, C, LFE, BL, BR | AV_CHANNEL_LAYOUT_5POINT1 | Supported (v2.2) |
| 7 | 6.1 | FL, FR, C, LFE, BL, BR, BC | AV_CHANNEL_LAYOUT_6POINT1 | Supported (v2.2) |
| 8 | 7.1 | FL, FR, C, LFE, BL, BR, SL, SR | AV_CHANNEL_LAYOUT_7POINT1 | Supported (v2.2) |

---

## 12. HIGH-RESOLUTION AUDIO SUPPORT

| Spec | Value | Notes |
|------|-------|-------|
| Max bit depth | 24-bit | Full internal precision |
| Max sample rate | 192,000 Hz | Supported |
| Float support | No | Integer PCM only |
| DSD support | No | Not applicable |

---

## 13. KNOWN ISSUES, BUGS & EDGE CASES

### 13.1 Interoperability Issues
- **FFmpeg support:** Most FFmpeg builds lack TAK support
- **Cross-platform:** Official tools primarily Windows-based
- **macOS/Linux:** Requires Wine or unofficial ports
- **Tag conflicts:** ID3v2 at beginning + APEv2 at end may conflict

### 13.2 Edge Cases to Handle
- **Corrupt MD5:** Report but continue playback
- **Missing seek table:** Fall back to frame sync scanning
- **Multichannel TAK in stereo player:** May fail or produce incorrect output

---

## 15. CONVERSION GUIDE (DBpoweramp Context)

### 15.1 Converting FROM TAK
| Target | Command | Metadata Preserved | Quality Notes |
|--------|---------|-------------------|---------------|
| → FLAC | TAK decoder + FLAC encoder | Via tag mapping | Lossless |
| → WAV | TAK decoder | No | Lossless decode |
| → MP3 | TAK decoder + LAME | Tags via LAME | Generation loss |
| → AAC | TAK decoder + AAC encoder | Tags via metadata | Re-encode |

### 15.2 Converting TO TAK
| Source | Command | Metadata Preserved | Quality Notes |
|--------|---------|-------------------|---------------|
| FLAC → | TAK encoder | APEv2 → APEv2 | Lossless |
| WAV → | TAK encoder | No | Lossless |
| MP3 → | TAK encoder | No | Generation loss |

### 15.3 Lossless Round-Trip Verification
```bash
# Decode TAK to WAV
takc -d input.tak output.wav

# Compare checksums
ffmpeg -i original.wav -c:a pcm_s16le -f md5 -
ffmpeg -i output.wav -c:a pcm_s16le -f md5 -
# Both must match for lossless

# Verify MD5 from TAK metadata
takc -v input.tak
# Shows MD5 checksum comparison
```

---

## 16. REFERENCE IMPLEMENTATIONS & LIBRARIES

| Library | Language | License | Encoder | Decoder | URL |
|---------|----------|---------|---------|---------|-----|
| TAK official | C++ | Freeware | Reference | Reference | thbeck.de/Tak |
| FFmpeg TAK | C | LGPL | No | Partial | ffmpeg.org |
| foo_input_tak | C++ | Proprietary | Reference | Reference | Winamp plugin |

### Build Instructions
```bash
# FFmpeg TAK support is optional and may not be enabled by default
./configure --enable-decoder=tak --enable-demuxer=tak
make -j$(nproc)
```

---

## 17. RELEVANT SPECIFICATIONS & FURTHER READING

### Primary Specification
- **TAK Format:** http://thbeck.de/Tak/Tak.html
- **Multimedia Wiki:** https://wiki.multimedia.cx/index.php?title=TAK

### Technical Resources
- TAK encoder downloads: thbeck.de/Tak/Download.html
- foo_input_tak plugin: Winamp plugin for TAK playback

---

## 18. IMPLEMENTATION CHECKLIST (Converter Developer)

### Build & Environment
- [ ] Verify FFmpeg has TAK support: `ffmpeg -decoders | grep tak`
- [ ] If not available, use official TAK tools: takc (encoder/decoder)
- [ ] Test on Windows (native) or Linux via Wine

### Encoding Pipeline
- [ ] Use official TAK encoder (takc) for encoding
- [ ] Select appropriate preset (-p0 to -p5)
- [ ] Configure frame size if needed (-fs option)
- [ ] Verify output with MD5 checksum

### Decoding Pipeline
- [ ] Use official TAK decoder (takc) for decoding
- [ ] If FFmpeg available, test with FFmpeg
- [ ] Verify with MD5 checksum if present

### Metadata
- [ ] Read APEv2 tags from TAK file
- [ ] Map APEv2 keys to standard field names
- [ ] Preserve cover art as binary item
- [ ] Write tags back to APEv2 format

### Quality & Verification
- [ ] Verify bit-exact decoding
- [ ] Test with all presets
- [ ] Test with multichannel files

### Edge Cases
- [ ] Handle files without seek table
- [ ] Handle files with obsolete seek table metadata
- [ ] Handle corrupted frames gracefully

---

*File generated for: DBpoweramp-equivalent audio converter knowledge base*
*Depth target: Complete implementation reference*
*[NEEDS VERIFICATION] markers indicate claims requiring additional source confirmation*
