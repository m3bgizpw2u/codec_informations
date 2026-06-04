# 71_FFmpeg_Tag_Mapping_Implementation.md

## FFmpeg Tag Mapping Implementation Guide

> **Purpose:** Research how to replicate DBpoweramp's tag behavior using FFmpeg. For each major conversion, document FFmpeg's native capabilities, limitations, and post-processing requirements.

---

## SECTION 1: FFMPEG METADATA HANDLING OVERVIEW

### 1.1 How FFmpeg Handles Metadata

FFmpeg has a dual-layer metadata system:

1. **Input demuxer extraction**: FFmpeg demuxers extract metadata from container formats into a generic key-value format
2. **Output muxer writing**: FFmpeg muxers write metadata using format-specific writers

**Critical limitation:** FFmpeg's metadata conversion between formats is **imperfect**. It uses a mapping table for common fields, but:
- Non-mapped fields are often dropped
- Multi-value fields are not handled correctly
- Cover art handling is inconsistent across formats
- Opus R128 tags are not automatically converted from ReplayGain

### 1.2 FFmpeg Metadata Command-Line Flags

| Flag | Purpose | DBpoweramp Equivalent |
|------|---------|----------------------|
| `-metadata title="..."` | Set a single metadata field | Tag editor field write |
| `-map_metadata 0` | Copy all metadata from input | "Preserve all tags" |
| `-map_metadata 0:g:0` | Copy global metadata only | Per-file tags only |
| `-map_metadata -1` | Clear all metadata | "Remove all tags" |
| `-id3v2_version 3\|4` | Set ID3v2 version for MP3 | ID3v2.3 vs v2.4 setting |
| `-write_id3v1 0\|1` | Write ID3v1 tag | ID3v1 presence |
| `-id3v2_padding_size` | Padding in ID3v2 tag | Not critical |

### 1.3 FFmpeg's Metadata Mapping Table

FFmpeg uses format-specific mapping tables. Key mappings for MP3 muxer:

| FFmpeg Key | ID3v2 Frame | Notes |
|------------|-------------|-------|
| title | TIT2 | |
| artist | TPE1 | |
| album | TALB | |
| album_artist | TPE2 | |
| composer | TCOM | |
| genre | TCON | |
| date | TDRC | ID3v2.4 only |
| track | TRCK | |
| disc | TPOS | |
| comment | COMM | |
| copyright | TCOP | |
| encoded_by | TENC | |
| language | TLAN | |
| publisher | TPUB | |
| encoder | TSSE | |
| lyrics | USLT | |
| performer | TPE3 | |
| compilation | TCMP | |
| album_sort | TSOA | ID3v2.4 only |
| artist_sort | TSOP | ID3v2.4 only |
| title_sort | TSOT | ID3v2.4 only |
| **Any other key** | TXXX (description=key, text=value) | Custom frame |

**Source:** FFmpeg `libavformat/mp3enc.c`, MultimediaWiki FFmpeg Metadata

---

## SECTION 2: COMPLETE FFMPEG COMMANDS BY CONVERSION TYPE

### 2.1 FLAC → MP3

**Native FFmpeg (broken):**
```bash
# WRONG - loses Artist and Track Number
ffmpeg -i input.flac -codec:a liblame -b:a 320k output.mp3
```

**Working FFmpeg:**
```bash
# Option 1: Use -map_metadata 0:g:0 (global metadata from input)
ffmpeg -i input.flac \
  -map_metadata 0:g:0 \
  -id3v2_version 3 \
  -write_id3v1 1 \
  -codec:a liblame -b:a 320k \
  output.mp3

# Option 2: Explicit metadata mapping
ffmpeg -i input.flac \
  -metadata title="" \
  -metadata artist="" \
  -metadata album="" \
  -map_metadata 0:g:0 \
  -id3v2_version 3 \
  -codec:a liblame -b:a 320k \
  output.mp3
```

**With Cover Art (critical):**
```bash
# Extract cover art and embed
ffmpeg -i input.flac \
  -i input.flac \
  -map 0:a \
  -map 1:v \
  -map_metadata 0:g:0 \
  -id3v2_version 3 \
  -attach input.jpg \
  -metadata:s:t mimetype=image/jpeg \
  -codec:a liblame -b:a 320k \
  output.mp3

# Better: Extract cover art to file, then use
ffmpeg -i input.flac -an -vcodec copy input_cover.jpg
ffmpeg -i input.flac -i input_cover.jpg \
  -map 0:a -map 1:image \
  -map_metadata 0:g:0 \
  -id3v2_version 3 \
  -id3v2_ac pic \
  -codec:a liblame -b:a 320k \
  output.mp3
```

**Complete DBpoweramp-equivalent FLAC→MP3:**
```bash
#!/bin/bash
# flac_to_mp3.sh - DBpoweramp-equivalent FLAC to MP3

INPUT="$1"
OUTPUT="$2"
QUALITY="${3:-320}"

# Extract metadata
TITLE=$(metaflac --show-tag=TITLE "$INPUT" | sed 's/TITLE=//')
ARTIST=$(metaflac --show-tag=ARTIST "$INPUT" | sed 's/ARTIST=//')
ALBUM=$(metaflac --show-tag=ALBUM "$INPUT" | sed 's/ALBUM=//')
ALBUMARTIST=$(metaflac --show-tag=ALBUMARTIST "$INPUT" | sed 's/ALBUMARTIST=//')
GENRE=$(metaflac --show-tag=GENRE "$INPUT" | sed 's/GENRE=//')
DATE=$(metaflac --show-tag=DATE "$INPUT" | sed 's/DATE=//')
TRACKNUMBER=$(metaflac --show-tag=TRACKNUMBER "$INPUT" | sed 's/TRACKNUMBER=//')
TRACKTOTAL=$(metaflac --show-tag=TRACKTOTAL "$INPUT" | sed 's/TRACKTOTAL=//')
DISCNUMBER=$(metaflac --show-tag=DISCNUMBER "$INPUT" | sed 's/DISCNUMBER=//')
DISCTOTAL=$(metaflac --show-tag=DISCTOTAL "$INPUT" | sed 's/DISCTOTAL=//')
COMMENT=$(metaflac --show-tag=COMMENT "$INPUT" | sed 's/COMMENT=//')

# Extract cover art
metaflac --export-picture-to=/tmp/cover.jpg "$INPUT" 2>/dev/null

# Build FFmpeg command
FFMPEG_CMD="ffmpeg -y -i '$INPUT'"

# Add cover art if exists
if [ -f /tmp/cover.jpg ]; then
    FFMPEG_CMD="$FFMPEG_CMD -i /tmp/cover.jpg"
fi

# Add metadata
FFMPEG_CMD="$FFMPEG_CMD -metadata title='$TITLE'"
FFMPEG_CMD="$FFMPEG_CMD -metadata artist='$ARTIST'"
FFMPEG_CMD="$FFMPEG_CMD -metadata album='$ALBUM'"
FFMPEG_CMD="$FFMPEG_CMD -metadata album_artist='$ALBUMARTIST'"
FFMPEG_CMD="$FFMPEG_CMD -metadata genre='$GENRE'"
FFMPEG_CMD="$FFMPEG_CMD -metadata date='$DATE'"

# Track number
if [ -n "$TRACKNUMBER" ]; then
    if [ -n "$TRACKTOTAL" ]; then
        FFMPEG_CMD="$FFMPEG_CMD -metadata track='${TRACKNUMBER}/${TRACKTOTAL}'"
    else
        FFMPEG_CMD="$FFMPEG_CMD -metadata track='$TRACKNUMBER'"
    fi
fi

# Disc number
if [ -n "$DISCNUMBER" ]; then
    if [ -n "$DISCTOTAL" ]; then
        FFMPEG_CMD="$FFMPEG_CMD -metadata disc='${DISCNUMBER}/${DISCTOTAL}'"
    else
        FFMPEG_CMD="$FFMPEG_CMD -metadata disc='$DISCNUMBER'"
    fi
fi

FFMPEG_CMD="$FFMPEG_CMD -metadata comment='$COMMENT'"
FFMPEG_CMD="$FFMPEG_CMD -metadata encoder='FFmpeg'"

# Cover art attachment
if [ -f /tmp/cover.jpg ]; then
    FFMPEG_CMD="$FFMPEG_CMD -attach /tmp/cover.jpg"
    FFMPEG_CMD="$FFMPEG_CMD -metadata:s:t mimetype=image/jpeg"
fi

FFMPEG_CMD="$FFMPEG_CMD -id3v2_version 3"
FFMPEG_CMD="$FFMPEG_CMD -write_id3v1 1"
FFMPEG_CMD="$FFMPEG_CMD -codec:a liblame -b:a ${QUALITY}k"
FFMPEG_CMD="$FFMPEG_CMD '$OUTPUT'"

# Execute
eval $FFMPEG_CMD

# Cleanup
rm -f /tmp/cover.jpg
```

### 2.2 FLAC → AAC/M4A

```bash
# Native FFmpeg with metadata
ffmpeg -i input.flac \
  -map_metadata 0:g:0 \
  -codec:a aac -b:a 256k \
  -movflags +faststart \
  output.m4a

# Explicit metadata for better control
ffmpeg -i input.flac \
  -metadata title="" \
  -metadata artist="" \
  -metadata album="" \
  -map_metadata 0:g:0 \
  -codec:a aac -b:a 256k \
  -movflags +faststart \
  output.m4a

# With cover art extraction
ffmpeg -i input.flac -an -vcodec copy /tmp/cover.png
ffmpeg -i input.flac -i /tmp/cover.png \
  -map 0:a -map 1:image \
  -map_metadata 0:g:0 \
  -codec:a aac -b:a 256k \
  -movflags +faststart \
  output.m4a
rm /tmp/cover.png
```

### 2.3 FLAC → OGG Vorbis

```bash
# OGG Vorbis metadata is preserved natively
ffmpeg -i input.flac \
  -map_metadata 0 \
  -codec:a libvorbis -q:a 6 \
  output.ogg

# With explicit metadata
ffmpeg -i input.flac \
  -map_metadata 0:g:0 \
  -codec:a libvorbis -q:a 6 \
  output.ogg
```

**Note:** FFmpeg's OGG muxer handles Vorbis comments well. The main issue is cover art - FFmpeg doesn't write METADATA_BLOCK_PICTURE correctly. Use `ffmpeg2theora` or `oggenc2` with `--artist`, `--album`, etc. for better results.

### 2.4 FLAC → Opus

```bash
# Basic conversion
ffmpeg -i input.flac \
  -map_metadata 0:g:0 \
  -codec:a libopus -b:a 128k \
  output.opus

# With metadata (FFmpeg Opus muxer handles Vorbis comments)
```

**CRITICAL - ReplayGain to R128:** FFmpeg does NOT convert ReplayGain to R128 automatically. You must handle this with a post-processing step:

```bash
#!/bin/bash
# flac_to_opus.sh - FLAC to Opus with R128 conversion

INPUT="$1"
OUTPUT="$2"

# Get ReplayGain values from FLAC
RG_TRACK_GAIN=$(metaflac --show-tag=REPLAYGAIN_TRACK_GAIN "$INPUT" 2>/dev/null | sed 's/REPLAYGAIN_TRACK_GAIN=//')
RG_TRACK_PEAK=$(metaflac --show-tag=REPLAYGAIN_TRACK_PEAK "$INPUT" 2>/dev/null | sed 's/REPLAYGAIN_TRACK_PEAK=//')

# Convert ReplayGain dB to R128 Q7.8
r128_track_gain=0
if [ -n "$RG_TRACK_GAIN" ]; then
    # Extract dB value: "-6.20 dB" -> -6.20
    db_val=$(echo "$RG_TRACK_GAIN" | sed 's/ dB$//' | sed 's/ //g')
    # Convert: R128_val = round((db_val + 5.0) * 256)
    r128_track_gain=$(python3 -c "print(int(round(($db_val + 5.0) * 256)))")
fi

# Convert FLAC to Opus
ffmpeg -i "$INPUT" \
  -map_metadata 0:g:0 \
  -codec:a libopus -b:a 128k \
  "$OUTPUT"

# Add R128 tags using opusenc or python
opuscomment "$OUTPUT" -a "R128_TRACK_GAIN=$r128_track_gain"
```

### 2.5 FLAC → WAV

```bash
# WAV metadata via FFmpeg
ffmpeg -i input.flac \
  -map_metadata 0:g:0 \
  -codec:a pcm_s16le \
  output.wav

# Note: WAV only supports INFO chunk fields
# FFmpeg maps to INFO chunk, but not all fields are supported
```

**FFmpeg WAV INFO mapping:**
| FFmpeg Key | INFO Chunk | Supported |
|------------|------------|-----------|
| title | INAM | Yes |
| artist | IART | Yes |
| album | IPRD | Yes |
| genre | IGNR | Yes |
| date | ICRD | Yes |
| track | ITRK | Yes |
| comment | ICMT | Yes |
| copyright | ICOP | Yes |
| **album_artist** | **N/A** | **No** |
| **cover_art** | **N/A** | **No** |

### 2.6 MP3 → FLAC

```bash
# MP3 to FLAC
ffmpeg -i input.mp3 \
  -map_metadata 0:g:0 \
  -codec:a flac \
  output.flac

# Note: FFmpeg handles this conversion reasonably well
```

### 2.7 OGG → MP3

```bash
# OGG Vorbis to MP3
ffmpeg -i input.ogg \
  -map_metadata 0:s:0 \
  -id3v2_version 3 \
  -write_id3v1 1 \
  -codec:a liblame -b:a 320k \
  output.mp3

# Critical: Must use -map_metadata 0:s:0 for OGG
# -map_metadata 0 alone does NOT work for OGG
```

**Source:** Unix StackExchange, FFmpeg mailing list archives

### 2.8 M4A → MP3

```bash
# AAC/M4A to MP3
ffmpeg -i input.m4a \
  -map_metadata 0:g:0 \
  -id3v2_version 3 \
  -write_id3v1 1 \
  -codec:a liblame -b:a 320k \
  output.mp3
```

### 2.9 MP3 → OGG

```bash
# MP3 to OGG Vorbis
ffmpeg -i input.mp3 \
  -map_metadata 0:g:0 \
  -codec:a libvorbis -q:a 6 \
  output.ogg
```

### 2.10 WAV → MP3

```bash
# WAV to MP3
ffmpeg -i input.wav \
  -map_metadata 0:g:0 \
  -id3v2_version 3 \
  -write_id3v1 1 \
  -codec:a liblame -b:a 320k \
  output.mp3
```

---

## SECTION 3: FFMPEG METADATA LIMITATIONS

### 3.1 Known Issues by Conversion Direction

| Conversion | Issue | Workaround |
|-----------|-------|-----------|
| FLAC→MP3 | Artist, Track Number often lost with native `-map_metadata 0` | Use `-map_metadata 0:g:0` |
| OGG→MP3 | `-map_metadata 0` doesn't work at all | Use `-map_metadata 0:s:0` |
| FLAC→MP3 | Multi-artist becomes `;` not `; ` | Post-process with `mid3v2` |
| FLAC→OGG | Cover art (METADATA_BLOCK_PICTURE) lost | Extract art, re-add manually |
| FLAC→Opus | R128 not converted from ReplayGain | Post-process with `opuscomment` |
| MP3→WAV | Cover art lost, limited fields | Use `wavpack` or manual |
| Any→WAV | ALBUMARTIST not supported | Only in INFO chunk |
| Any→MP3 | Sort fields (TSOA, TSOP) require v2.4 | Set `-id3v2_version 4` |

### 3.2 FFmpeg Metadata Encoding Issues

**UTF-8:**
```bash
# FFmpeg handles UTF-8 metadata reasonably
ffmpeg -i input.flac -metadata title="Café" output.mp3
```

**UTF-16 (ID3v2.3 default):**
```bash
# ID3v2.3 uses UTF-16 for non-ASCII by default
# FFmpeg's ID3v2.3 encoder handles this
ffmpeg -i input.flac -id3v2_version 3 -metadata title="日本語" output.mp3

# For explicit UTF-16
ffmpeg -i input.flac -id3v2_version 3 -metadata encoding=utf-16 -metadata title="日本語" output.mp3
```

**Character encoding detection:**
```bash
# FFmpeg doesn't always detect encoding correctly
# Use ffprobe to inspect
ffprobe -show_entries stream_tags=title,artist,album -of default=noprint_wrappers=1 input.mp3

# Check encoding
file input.mp3
# May show: "Non-ISO extended-ASCII text" for ID3v1
# Should show: "UTF-8 Unicode text" for properly tagged files
```

### 3.3 FFmpeg's -id3v2_version Option

```bash
# ID3v2.3 (DBpoweramp default)
ffmpeg -i input.flac -id3v2_version 3 output.mp3
# Uses: TYER, TDAT, TORY, IPLS, TIPL frames

# ID3v2.4 (more modern)
ffmpeg -i input.flac -id3v2_version 4 output.mp3
# Uses: TDRC, TDOR, IPLS replaced by TIPL/TMCL
```

**DBpoweramp default:** ID3v2.3 (`-id3v2_version 3`)

### 3.4 FFmpeg's -write_id3v1 Option

```bash
# Write ID3v1 tag alongside ID3v2
ffmpeg -i input.flac -write_id3v1 1 -id3v2_version 3 output.mp3

# ID3v1 only
ffmpeg -i input.flac -write_id3v1 1 -id3v2_version 0 output.mp3

# No ID3v1
ffmpeg -i input.flac -write_idv1 0 output.mp3
```

### 3.5 FFmpeg Atomic Write Strategy

FFmpeg doesn't have atomic write built-in. Implement manually:

```bash
#!/bin/bash
# atomic_ffmpeg_convert.sh

INPUT="$1"
OUTPUT="$2"
shift 2
FFMPEG_ARGS="$@"

# Write to temp file
TMP_OUTPUT="${OUTPUT}.tmp.$$"

# Run FFmpeg
ffmpeg -y -i "$INPUT" $FFMPEG_ARGS "$TMP_OUTPUT"

# Verify output
if [ -f "$TMP_OUTPUT" ] && [ -s "$TMP_OUTPUT" ]; then
    # Atomically rename
    mv "$TMP_OUTPUT" "$OUTPUT"
else
    echo "Error: FFmpeg conversion failed"
    rm -f "$TMP_OUTPUT"
    exit 1
fi
```

---

## SECTION 4: POST-PROCESSING WITH FFprobe + External Tools

When FFmpeg can't handle something natively, use post-processing:

### 4.1 ReplayGain to R128 (Opus)

```python
#!/usr/bin/env python3
"""
Convert ReplayGain tags to Opus R128 format.

Opus uses R128_TRACK_GAIN with Q7.8 fixed-point integers.
ReplayGain uses dB strings like "-6.20 dB".

Conversion: R128_val = round((RG_dB + 5.0) * 256)
"""

import sys
import re
import subprocess


def parse_replaygain_db(value: str) -> float:
    """Parse ReplayGain string like '-6.20 dB' to float."""
    if not value:
        return 0.0
    match = re.match(r'([+-]?\d+\.?\d*)\s*dB', str(value), re.IGNORECASE)
    if match:
        return float(match.group(1))
    try:
        return float(value)
    except (ValueError, TypeError):
        return 0.0


def replaygain_to_r128(gain_db: float) -> int:
    """Convert ReplayGain dB to Opus R128 Q7.8 format."""
    return round((gain_db + 5.0) * 256)


def get_replaygain_from_flac(flac_path: str) -> tuple:
    """Extract ReplayGain values from FLAC file."""
    try:
        result = subprocess.run(
            ['metaflac', '--show-tag=REPLAYGAIN_TRACK_GAIN', flac_path],
            capture_output=True, text=True
        )
        track_gain = parse_replaygain_db(result.stdout.strip())
        
        result = subprocess.run(
            ['metaflac', '--show-tag=REPLAYGAIN_TRACK_PEAK', flac_path],
            capture_output=True, text=True
        )
        track_peak = 0.0
        try:
            track_peak = float(result.stdout.strip())
        except ValueError:
            pass
        
        return track_gain, track_peak
    except Exception as e:
        print(f"Error reading FLAC: {e}", file=sys.stderr)
        return 0.0, 0.0


def add_r128_tags(opus_path: str, track_gain: float, track_peak: float = 0.0):
    """Add R128 tags to Opus file using opuscomment."""
    r128_val = replaygain_to_r128(track_gain)
    
    try:
        # Remove existing R128 tags
        subprocess.run(
            ['opuscomment', '--remove', '--raw', 'R128_TRACK_GAIN', opus_path],
            capture_output=True
        )
        
        # Add new R128 tag
        subprocess.run(
            ['opuscomment', '--raw', '--add', f'R128_TRACK_GAIN={r128_val}', opus_path],
            capture_output=True, check=True
        )
        
        print(f"Added R128_TRACK_GAIN={r128_val} (from ReplayGain {track_gain:.2f} dB)")
        
        # Also add ReplayGain for compatibility
        if track_gain != 0.0:
            subprocess.run(
                ['opuscomment', '--remove', '--raw', 'REPLAYGAIN_TRACK_GAIN', opus_path],
                capture_output=True
            )
            subprocess.run(
                ['opuscomment', '--raw', '--add', f'REPLAYGAIN_TRACK_GAIN={track_gain:.2f} dB', opus_path],
                capture_output=True, check=True
            )
        
        if track_peak > 0:
            subprocess.run(
                ['opuscomment', '--remove', '--raw', 'REPLAYGAIN_TRACK_PEAK', opus_path],
                capture_output=True
            )
            subprocess.run(
                ['opuscomment', '--raw', '--add', f'REPLAYGAIN_TRACK_PEAK={track_peak}', opus_path],
                capture_output=True, check=True
            )
        
    except Exception as e:
        print(f"Error updating Opus tags: {e}", file=sys.stderr)


if __name__ == "__main__":
    if len(sys.argv) < 3:
        print(f"Usage: {sys.argv[0]} <flac_path> <opus_path>")
        sys.exit(1)
    
    flac_path = sys.argv[1]
    opus_path = sys.argv[2]
    
    track_gain, track_peak = get_replaygain_from_flac(flac_path)
    
    if track_gain != 0.0:
        add_r128_tags(opus_path, track_gain, track_peak)
    else:
        print("No ReplayGain found, skipping R128 conversion")
```

### 4.2 Cover Art Extraction and Re-embedding

```python
#!/usr/bin/env python3
"""
Extract and re-embed cover art using FFmpeg + Python.
"""

import subprocess
import tempfile
import os


def extract_cover_art(input_path: str, output_path: str = None) -> str:
    """Extract cover art from audio file to image file."""
    if output_path is None:
        fd, output_path = tempfile.mkstemp(suffix='.jpg')
        os.close(fd)
    
    # Try FLAC
    result = subprocess.run(
        ['metaflac', '--export-picture-to=' + output_path, input_path],
        capture_output=True
    )
    
    if result.returncode != 0:
        # Try FFmpeg (extract first video stream = cover art)
        result = subprocess.run(
            ['ffmpeg', '-y', '-i', input_path, '-an', '-vcodec', 'copy', output_path],
            capture_output=True
        )
        
        if result.returncode != 0:
            os.remove(output_path)
            return None
    
    return output_path


def embed_cover_art(image_path: str, audio_path: str, output_path: str, 
                   format: str = 'mp3'):
    """Embed cover art image into audio file."""
    
    if format == 'mp3':
        # Use FFmpeg with ID3v2
        cmd = [
            'ffmpeg', '-y',
            '-i', audio_path,
            '-i', image_path,
            '-map', '0:a',
            '-map', '1:image',
            '-id3v2_version', '3',
            '-write_id3v1', '1',
            '-codec:a', 'copy',
            '-metadata:s:v', 'mimetype=image/jpeg',
            output_path
        ]
    elif format == 'm4a':
        cmd = [
            'ffmpeg', '-y',
            '-i', audio_path,
            '-i', image_path,
            '-map', '0:a',
            '-map', '1:image',
            '-codec:a', 'copy',
            '-disposition:v', 'attached_pic',
            output_path
        ]
    elif format == 'flac':
        cmd = [
            'ffmpeg', '-y',
            '-i', audio_path,
            '-i', image_path,
            '-map', '0:a',
            '-map', '1:image',
            '-codec:a', 'copy',
            output_path
        ]
    elif format == 'ogg':
        # OGG/Vorbis requires special handling
        # FFmpeg doesn't write METADATA_BLOCK_PICTURE correctly
        print("Warning: OGG cover art requires oggfwd or similar tool")
        return False
    else:
        print(f"Unsupported format: {format}")
        return False
    
    result = subprocess.run(cmd, capture_output=True)
    return result.returncode == 0


def convert_with_cover_art(input_path: str, output_path: str,
                          format: str, encoder: str = None,
                          bitrate: str = None):
    """
    Complete conversion pipeline with cover art handling.
    """
    # Extract cover art
    cover_path = extract_cover_art(input_path)
    
    # Build FFmpeg command
    cmd = ['ffmpeg', '-y', '-i', input_path]
    
    if cover_path and os.path.exists(cover_path):
        cmd.extend(['-i', cover_path])
    
    # Map streams
    cmd.extend(['-map', '0:a'])
    if cover_path and os.path.exists(cover_path):
        cmd.extend(['-map', '1:image'])
    
    # Encoding
    if encoder:
        if format == 'mp3':
            cmd.extend(['-codec:a', encoder])
            if bitrate:
                cmd.extend(['-b:a', bitrate])
        elif format == 'm4a':
            cmd.extend(['-codec:a', encoder])
            if bitrate:
                cmd.extend(['-b:a', bitrate])
        elif format == 'ogg':
            cmd.extend(['-codec:a', encoder])
            if bitrate:
                cmd.extend(['-b:a', bitrate])
        elif format == 'opus':
            cmd.extend(['-codec:a', encoder])
            if bitrate:
                cmd.extend(['-b:a', bitrate])
        elif format == 'flac':
            cmd.extend(['-codec:a', 'flac'])
        else:
            cmd.extend(['-codec:a', 'copy'])
    
    # Metadata
    cmd.extend(['-map_metadata', '0:g:0'])
    
    if format == 'mp3':
        cmd.extend(['-id3v2_version', '3'])
        cmd.extend(['-write_id3v1', '1'])
        if cover_path:
            cmd.extend(['-metadata:s:v', 'mimetype=image/jpeg'])
    elif format == 'm4a':
        cmd.extend(['-movflags', '+faststart'])
        if cover_path:
            cmd.extend(['-disposition:v', 'attached_pic'])
    elif format == 'flac':
        pass  # FLAC handles this natively
    
    cmd.append(output_path)
    
    result = subprocess.run(cmd, capture_output=True, text=True)
    
    # Cleanup
    if cover_path and os.path.exists(cover_path):
        os.remove(cover_path)
    
    if result.returncode != 0:
        print(f"FFmpeg error: {result.stderr}")
        return False
    
    return True
```

### 4.3 Multi-Artist Handling with mid3v2

FFmpeg joins multiple artists with `;` but DBpoweramp uses `; `. Fix with `mid3v2`:

```bash
#!/bin/bash
# fix_multi_artist.sh - Fix multi-artist separator

MP3_FILE="$1"

# Read current artist
CURRENT_ARTIST=$(mid3v2 --artist "$MP3_FILE" 2>/dev/null)

# Replace ; with ; 
if [[ "$CURRENT_ARTIST" == *";"* ]]; then
    NEW_ARTIST=$(echo "$CURRENT_ARTIST" | sed 's/;/; /g')
    mid3v2 --artist "$NEW_ARTIST" "$MP3_FILE"
    echo "Fixed: $CURRENT_ARTIST -> $NEW_ARTIST"
fi
```

```python
#!/usr/bin/env python3
"""Python version of multi-artist fix."""
from mutagen.mp3 import MP3
from mutagen.id3 import TPE1


def fix_multi_artist_separator(mp3_path: str):
    """Replace ; with ;  in artist field."""
    mp3 = MP3(mp3_path)
    
    if not mp3.tags:
        return
    
    try:
        frames = mp3.tags.getall('TPE1')
        if frames:
            current = str(frames[0])
            if ';' in current and '; ' not in current:
                new_artist = current.replace(';', '; ')
                frames[0].text = [new_artist]
                mp3.tags.add(TPE1(encoding=3, text=new_artist))
                mp3.save()
                print(f"Fixed: {current} -> {new_artist}")
    except Exception as e:
        print(f"Error: {e}")
```

### 4.4 Genre Number Resolution

FFmpeg doesn't resolve genre numbers. Use `mid3v2` or `eyeD3`:

```bash
#!/bin/bash
# fix_genre.sh - Resolve ID3v1 genre numbers

MP3_FILE="$1"

# Get current genre
CURRENT_GENRE=$(mid3v2 --genre "$MP3_FILE" 2>/dev/null)

# Check if it's a number
if [[ "$CURRENT_GENRE" =~ ^[0-9]+$ ]]; then
    # mid3v2 can set by number, but we want the string
    # Use eyeD3 for number resolution
    GENRE_STRING=$(eyeD3 --plugin genre-to-string -- "$MP3_FILE" 2>/dev/null)
    if [ -n "$GENRE_STRING" ]; then
        mid3v2 --genre "$GENRE_STRING" "$MP3_FILE"
        echo "Resolved genre: $CURRENT_GENRE -> $GENRE_STRING"
    fi
fi
```

---

## SECTION 5: COMPLETE FFMPEG COMMANDS REFERENCE TABLE

| Source | Target | Command | Notes |
|--------|--------|---------|-------|
| FLAC | MP3 | `ffmpeg -i in.flac -map_metadata 0:g:0 -id3v2_version 3 -write_id3v1 1 -codec:a liblame -b:a 320k out.mp3` | Use `-map_metadata 0:g:0` |
| FLAC | M4A | `ffmpeg -i in.flac -map_metadata 0:g:0 -codec:a aac -b:a 256k -movflags +faststart out.m4a` | |
| FLAC | OGG | `ffmpeg -i in.flac -map_metadata 0 -codec:a libvorbis -q:a 6 out.ogg` | Cover art via post-processing |
| FLAC | Opus | `ffmpeg -i in.flac -map_metadata 0:g:0 -codec:a libopus -b:a 128k out.opus` | Post-process for R128 |
| FLAC | WAV | `ffmpeg -i in.flac -map_metadata 0:g:0 -codec:a pcm_s16le out.wav` | Limited fields |
| MP3 | FLAC | `ffmpeg -i in.mp3 -map_metadata 0:g:0 -codec:a flac out.flac` | |
| MP3 | OGG | `ffmpeg -i in.mp3 -map_metadata 0:s:0 -codec:a libvorbis -q:a 6 out.ogg` | **Must use 0:s:0** |
| MP3 | M4A | `ffmpeg -i in.mp3 -map_metadata 0:g:0 -codec:a aac -b:a 256k -movflags +faststart out.m4a` | |
| MP3 | Opus | `ffmpeg -i in.mp3 -map_metadata 0:g:0 -codec:a libopus -b:a 128k out.opus` | |
| OGG | MP3 | `ffmpeg -i in.ogg -map_metadata 0:s:0 -id3v2_version 3 -write_id3v1 1 -codec:a liblame -b:a 320k out.mp3` | **Must use 0:s:0** |
| OGG | FLAC | `ffmpeg -i in.ogg -map_metadata 0 -codec:a flac out.flac` | |
| OGG | M4A | `ffmpeg -i in.ogg -map_metadata 0:s:0 -codec:a aac -b:a 256k out.m4a` | |
| M4A | MP3 | `ffmpeg -i in.m4a -map_metadata 0:g:0 -id3v2_version 3 -write_id3v1 1 -codec:a liblame -b:a 320k out.mp3` | |
| M4A | FLAC | `ffmpeg -i in.m4a -map_metadata 0:g:0 -codec:a flac out.flac` | |
| M4A | OGG | `ffmpeg -i in.m4a -map_metadata 0:g:0 -codec:a libvorbis -q:a 6 out.ogg` | |
| WAV | MP3 | `ffmpeg -i in.wav -map_metadata 0:g:0 -id3v2_version 3 -write_id3v1 1 -codec:a liblame -b:a 320k out.mp3` | |
| WAV | FLAC | `ffmpeg -i in.wav -map_metadata 0:g:0 -codec:a flac out.flac` | |
| AIFF | MP3 | `ffmpeg -i in.aiff -map_metadata 0:g:0 -id3v2_version 3 -write_id3v1 1 -codec:a liblame -b:a 320k out.mp3` | |

---

## SECTION 6: METADATA ENCODING ISSUES

### 6.1 UTF-8 Handling

```bash
# Check encoding with ffprobe
ffprobe -show_entries stream_tags=title -of json input.mp3

# Force UTF-8 metadata
ffmpeg -i input.mp3 -metadata title="Test — Title" -metadata encoding=UTF-8 output.mp3
```

### 6.2 UTF-16 Handling (ID3v2.3)

```bash
# ID3v2.3 uses UTF-16 for non-ASCII characters
# FFmpeg handles this automatically when using -id3v2_version 3

# Verify encoding
mid3v2 -v input.mp3 | head -20
```

### 6.3 ISO-8859-1 Legacy Files

```bash
# Convert legacy ISO-8859-1 tags to UTF-8
mid3v2 --encoding=UTF-8 input.mp3

# Or with eyeD3
eyeD3 --encoding=utf8 input.mp3
```

---

## SECTION 7: ATOMIC WRITE WITH FFMPEG

### 7.1 Safe Conversion Script

```python
#!/usr/bin/env python3
"""
Safe FFmpeg conversion with atomic write and tag preservation.

This script:
1. Converts audio using FFmpeg
2. Preserves all metadata using FFmpeg's -map_metadata
3. Atomically replaces the output file
4. Handles errors gracefully
"""

import subprocess
import shutil
import os
import tempfile
import sys


def safe_ffmpeg_convert(input_path: str,
                        output_path: str,
                        codec: str = None,
                        bitrate: str = None,
                        extra_args: list = None,
                        metadata_format: str = 'mp3'):
    """
    Safely convert audio file with FFmpeg.
    
    Args:
        input_path: Input file path
        output_path: Output file path
        codec: Audio codec (e.g., 'liblame', 'aac', 'libvorbis')
        bitrate: Audio bitrate (e.g., '320k', '256k')
        extra_args: Additional FFmpeg arguments
        metadata_format: Format for metadata handling
    
    Returns:
        (success: bool, error_message: str)
    """
    # Create temp file in same directory as output
    output_dir = os.path.dirname(output_path) or '.'
    fd, temp_path = tempfile.mkstemp(
        suffix='.tmp',
        prefix='.ffmpeg_convert_',
        dir=output_dir
    )
    os.close(fd)
    
    try:
        # Build FFmpeg command
        cmd = ['ffmpeg', '-y', '-i', input_path]
        
        # Cover art (handle separately)
        cover_added = False
        
        # Metadata mapping
        cmd.extend(['-map_metadata', '0:g:0'])
        
        # Format-specific metadata options
        if metadata_format == 'mp3':
            cmd.extend(['-id3v2_version', '3'])
            cmd.extend(['-write_id3v1', '1'])
        elif metadata_format == 'm4a':
            cmd.extend(['-movflags', '+faststart'])
        elif metadata_format == 'ogg':
            pass  # OGG handles metadata natively
        elif metadata_format == 'opus':
            pass  # Opus handles metadata natively
        
        # Audio codec
        if codec:
            if codec == 'copy':
                cmd.extend(['-codec:a', 'copy'])
            else:
                cmd.extend(['-codec:a', codec])
                if bitrate:
                    cmd.extend(['-b:a', bitrate])
        
        # Extra arguments
        if extra_args:
            cmd.extend(extra_args)
        
        # Output
        cmd.append(temp_path)
        
        # Run FFmpeg
        result = subprocess.run(
            cmd,
            capture_output=True,
            text=True,
            timeout=300  # 5 minute timeout
        )
        
        if result.returncode != 0:
            return False, f"FFmpeg error: {result.stderr[:500]}"
        
        # Verify output file
        if not os.path.exists(temp_path) or os.path.getsize(temp_path) == 0:
            return False, "Output file is empty or missing"
        
        # Atomic rename
        os.replace(temp_path, output_path)
        return True, ""
        
    except subprocess.TimeoutExpired:
        return False, "FFmpeg conversion timed out"
    except Exception as e:
        return False, f"Error: {str(e)}"
    finally:
        # Clean up temp file
        if os.path.exists(temp_path):
            try:
                os.remove(temp_path)
            except OSError:
                pass


def convert_with_metadata(input_path: str,
                        output_path: str,
                        source_format: str,
                        target_format: str,
                        encoder: str = None,
                        quality: str = None):
    """
    High-level conversion function with automatic codec selection.
    """
    # Codec mapping
    codecs = {
        'mp3': ('liblame', quality or '320k'),
        'm4a': ('aac', quality or '256k'),
        'ogg': ('libvorbis', quality or 'q6'),
        'opus': ('libopus', quality or '128k'),
        'flac': ('flac', None),
        'wav': ('pcm_s16le', None),
        'alac': ('alac', None),
    }
    
    codec, bitrate = codecs.get(target_format, ('copy', None))
    
    if encoder:
        codec = encoder
    
    extra_args = []
    if target_format == 'opus':
        extra_args = ['-vbr', 'on', '-compression_level', '10']
    
    success, error = safe_ffmpeg_convert(
        input_path=input_path,
        output_path=output_path,
        codec=codec if target_format != 'flac' else 'flac',
        bitrate=bitrate if target_format not in ('flac', 'wav', 'alac') else None,
        extra_args=extra_args,
        metadata_format=target_format
    )
    
    return success, error
```

---

## SECTION 8: EDGE CASES AND SPECIAL HANDLING

### 8.1 ReplayGain to R128 Conversion Table

| ReplayGain (dB) | R128 (Q7.8) | Calculation |
|-----------------|-------------|-------------|
| -12.00 | -1792 | round((-12.0 + 5.0) * 256) = round(-1792) |
| -6.00 | -256 | round((-6.0 + 5.0) * 256) = round(-256) |
| -3.00 | 512 | round((-3.0 + 5.0) * 256) = round(512) |
| 0.00 | 1280 | round((0.0 + 5.0) * 256) = round(1280) |
| +3.00 | 2048 | round((3.0 + 5.0) * 256) = round(2048) |
| +6.00 | 2816 | round((6.0 + 5.0) * 256) = round(2816) |

### 8.2 FFmpeg Metadata Bug Reports

1. **FLAC→MP3 Artist/Track loss**: Reported in 2009, partially fixed
   - Source: https://ffmpeg.org/pipermail/ffmpeg-devel/2009-June/071122.html
   
2. **OGG metadata not mapping**: `-map_metadata 0` doesn't work for OGG
   - Source: https://unix.stackexchange.com/questions/176945

3. **Cover art in FLAC→MP3**: METADATA_BLOCK_PICTURE not converted
   - Requires manual extraction and re-embedding

### 8.3 Cover Art Type Codes for FFmpeg

```bash
# Set picture type for MP3 (ID3v2 APIC)
# Type codes:
# 0 = Other
# 1 = 32x32 file icon
# 2 = Other file icon
# 3 = Front cover (DBpoweramp default)
# 4 = Back cover
# 5 = Leaflet
# 6 = Media
# 7 = Lead artist
# 8 = Artist
# 9 = Conductor
# 10 = Band/Orchestra
# 11 = Composer
# 12 = Lyricist
# 13 = Recording location
# 14 = During recording
# 15 = During performance
# 16 = Movie/video capture
# 17 = Illustration
# 18 = Band/artist logotype
# 19 = Publisher/Studio logotype

# FFmpeg doesn't directly support setting picture type
# Use mutagen or mid3v2 post-processing
```

---

## SECTION 9: SOURCE CITATIONS

| Claim | Source |
|-------|--------|
| FFmpeg MP3 metadata mapping table | MultimediaWiki, FFmpeg libavformat/mp3enc.c |
| `-map_metadata 0:s:0` for OGG | Unix StackExchange #176945 |
| ReplayGain → R128 conversion formula | FFmpeg libavcodec/opus.c, Hydrogenaudio |
| R128 Q7.8 fixed-point format | RFC 7845 (Ogg Opus) |
| FLAC→MP3 Artist/Track loss bug | FFmpeg mailing list, June 2009 |
| Cover art in FLAC via metaflac | metaflac manual |
| Opus METADATA_BLOCK_PICTURE format | Xiph FLAC specification |

---

*Document: 71_FFmpeg_Tag_Mapping_Implementation.md*
*Generated from: FFmpeg metadata handling research*
*Version: 1.0*
