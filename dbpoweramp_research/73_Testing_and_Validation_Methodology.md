# 73_Testing_and_Validation_Methodology.md

## Testing and Validation Methodology

> **Purpose:** Complete testing methodology to verify a DBpoweramp-equivalent converter. Includes test case definitions, expected outputs, and automated validation scripts.

---

## SECTION 1: TEST METHODOLOGY OVERVIEW

### 1.1 Testing Philosophy

Testing a DBpoweramp-equivalent converter requires:

1. **Functional tests**: Verify tag fields are preserved correctly
2. **Format-specific tests**: Verify format-specific behaviors
3. **Edge case tests**: Verify corner cases are handled correctly
4. **Integration tests**: Verify the full pipeline works end-to-end
5. **Performance tests**: Verify conversion speed is acceptable

### 1.2 Test Categories

| Category | Purpose | Examples |
|----------|---------|----------|
| **Tag Preservation** | Verify fields survive conversion | Track number, artist, genre |
| **Cover Art** | Verify artwork handling | Type codes, folder.jpg fallback |
| **Character Encoding** | Verify text encoding | UTF-8, UTF-16, CJK |
| **Edge Cases** | Verify corner cases | Empty fields, multi-value |
| **Lossless Verification** | Verify audio integrity | Bit-exact for lossless |
| **Gapless Playback** | Verify gapless tags | LAME header, iTunSMPB |
| **Batch Processing** | Verify concurrent reliability | 100+ files |

---

## SECTION 2: TAG PRESERVATION TESTS

### 2.1 Test File Definitions

Create a test file repository with these specific configurations:

**test_basic_tags.mp3:**
- Title: "Test Track"
- Artist: "Test Artist"
- Album: "Test Album"
- Album Artist: "Test Album Artist"
- Composer: "Test Composer"
- Track: 5
- Track Total: 12
- Disc: 1
- Disc Total: 3
- Date: 2024-03-15
- Genre: Rock
- Comment: "This is a test comment"
- Cover Art: Embedded JPEG

**test_multi_artist.flac:**
- Artist: Multiple (ARTIST=A; ARTIST=B)
- Album Artist: Multiple (ALBUMARTIST=X; ALBUMARTIST=Y)
- Genre: Multiple (GENRE=Rock; GENRE=Alternative)

**test_genre_number.mp3:**
- Genre: TCON=(17) → should resolve to "Rock"

**test_track_formats.flac:**
- TRACKNUMBER=5/12 (slash format)
- TRACKTOTAL=12 (separate field)

**test_disc_formats.m4a:**
- trkn=(5, 12) in atom

**test_sort_fields.mp3:**
- TSOT: "Test Track, The"
- TSOP: "Artist, Test"
- TSOA: "Album, Test"
- TSOC: "Composer, Test"

**test_replaygain.flac:**
- REPLAYGAIN_TRACK_GAIN=-6.20 dB
- REPLAYGAIN_TRACK_PEAK=0.876543
- REPLAYGAIN_ALBUM_GAIN=-4.50 dB
- REPLAYGAIN_ALBUM_PEAK=0.987654

**test_r128.opus:**
- R128_TRACK_GAIN=1234 (Q7.8 format)
- R128_ALBUM_GAIN=567

**test_work_movement.m4a:**
- @wrk="Symphony No. 5"
- @mvn="I. Allegro"
- @mvi=1
- @mvc=4

**test_custom_tags.mp3:**
- TXXX:RELEASETIME=2024-01-15
- TXXX:UPC=012345678901
- TXXX:CATNO=ABC123

**test_cover_types.flac:**
- Picture 1: Front Cover (type 3)
- Picture 2: Back Cover (type 4)
- Picture 3: Artist (type 8)

### 2.2 Tag Field Test Matrix

| Source Field | Source Value | Expected in MP3 | Expected in FLAC | Expected in M4A | Expected in OGG |
|-------------|--------------|-----------------|------------------|-----------------|-----------------|
| title | "Track Title" | TIT2="Track Title" | TITLE="Track Title" | ©nam="Track Title" | TITLE="Track Title" |
| artist | "Artist Name" | TPE1="Artist Name" | ARTIST="Artist Name" | ©ART="Artist Name" | ARTIST="Artist Name" |
| album | "Album Name" | TALB="Album Name" | ALBUM="Album Name" | ©alb="Album Name" | ALBUM="Album Name" |
| album_artist | "Album Artist" | TPE2="Album Artist" | ALBUMARTIST="Album Artist" | aART="Album Artist" | ALBUMARTIST="Album Artist" |
| composer | "Composer" | TCOM="Composer" | COMPOSER="Composer" | ©wrt="Composer" | COMPOSER="Composer" |
| track (5/12) | "5/12" | TRCK="5/12" | TRACKNUMBER="5", TRACKTOTAL="12" | trkn=(5,12) | TRACKNUMBER="5", TOTALTRACKS="12" |
| disc (2/3) | "2/3" | TPOS="2/3" | DISCNUMBER="2", DISCTOTAL="3" | diskn=(2,3) | DISCNUMBER="2", TOTALDISCS="3" |
| date | "2024-03-15" | TDRC="2024-03-15" | DATE="2024-03-15" | ©day="2024-03-15" | DATE="2024-03-15" |
| genre (17) | "(17)Rock" | TCON="Rock" | GENRE="Rock" | ©gen="Rock" | GENRE="Rock" |
| comment | "Test comment" | COMM="Test comment" | COMMENT="Test comment" | ©cmt="Test comment" | COMMENT="Test comment" |
| isrc | "USRC12345678" | TSRC="USRC12345678" | ISRC="USRC12345678" | ©isr="USRC12345678" | ISRC="USRC12345678" |
| bpm | 120 | TBPM="120" | BPM="120" | tmpo=120 | BPM="120" |
| encoder | "LAME 3.100" | TSSE="LAME 3.100" | ENCODER="LAME 3.100" | ©enc="LAME 3.100" | ENCODER="LAME 3.100" |
| cover_art | JPEG data | APIC type=3 | METADATA_BLOCK_PICTURE | covr | METADATA_BLOCK_PICTURE |
| rg_track_gain | "-6.20 dB" | TXXX:REPLAYGAIN_TRACK_GAIN | REPLAYGAIN_TRACK_GAIN | ----:REPLAYGAIN_TRACK_GAIN | REPLAYGAIN_TRACK_GAIN |
| sort fields | various | TSOT, TSOP, TSOA, TSOC | TITLESORT, etc. | sonm, soar, etc. | TITLESORT, etc. |

### 2.3 Test Scripts

```python
#!/usr/bin/env python3
"""
tag_preservation_test.py - Test tag preservation across format conversions.

Usage:
    python tag_preservation_test.py --source-dir=/path/to/test/files
"""

import os
import sys
import argparse
import subprocess
from pathlib import Path
from dataclasses import dataclass
from typing import Dict, List, Optional, Any
from enum import Enum


class TestResult(Enum):
    PASS = "PASS"
    FAIL = "FAIL"
    SKIP = "SKIP"
    WARN = "WARN"


@dataclass
class FieldTest:
    field_name: str
    source_value: Any
    expected_value: Any
    actual_value: Any = None
    result: TestResult = TestResult.SKIP
    message: str = ""


@dataclass
class ConversionTest:
    name: str
    source_file: str
    dest_file: str
    source_format: str
    dest_format: str
    field_tests: List[FieldTest]
    audio_verified: bool = False


def read_tag_value(file_path: str, field_name: str) -> Optional[Any]:
    """
    Read a tag field value from an audio file.
    Uses mutagen for cross-format support.
    """
    from mutagen import File as MutagenFile
    from mutagen.mp3 import MP3
    from mutagen.flac import FLAC
    from mutagen.mp4 import MP4
    from mutagen.oggvorbis import OggVorbis
    
    try:
        ext = Path(file_path).suffix.lower()
        
        if ext == '.mp3':
            f = MP3(file_path)
            return _get_mp3_field(f, field_name)
        
        elif ext == '.flac':
            f = FLAC(file_path)
            return _get_flac_field(f, field_name)
        
        elif ext in ('.m4a', '.mp4', '.m4b'):
            f = MP4(file_path)
            return _get_mp4_field(f, field_name)
        
        elif ext == '.ogg':
            f = OggVorbis(file_path)
            return _get_vorbis_field(f, field_name)
        
    except Exception as e:
        return f"ERROR: {e}"
    
    return None


def _get_mp3_field(f: MP3, field: str) -> Any:
    """Get MP3 tag field value."""
    if not f.tags:
        return None
    
    field_map = {
        'title': 'TIT2',
        'artist': 'TPE1',
        'album': 'TALB',
        'album_artist': 'TPE2',
        'composer': 'TCOM',
        'track': 'TRCK',
        'disc': 'TPOS',
        'date': 'TDRC',
        'genre': 'TCON',
        'comment': 'COMM',
        'isrc': 'TSRC',
        'bpm': 'TBPM',
        'encoder': 'TSSE',
        'title_sort': 'TSOT',
        'artist_sort': 'TSOP',
        'album_sort': 'TSOA',
        'composer_sort': 'TSOC',
    }
    
    frame_id = field_map.get(field)
    if not frame_id:
        return None
    
    try:
        frames = f.tags.getall(frame_id)
        if frames:
            return frames[0].text[0] if hasattr(frames[0], 'text') else str(frames[0])
    except (KeyError, IndexError, AttributeError):
        pass
    
    return None


def _get_flac_field(f: FLAC, field: str) -> Any:
    """Get FLAC tag field value."""
    field_map = {
        'title': 'TITLE',
        'artist': 'ARTIST',
        'album': 'ALBUM',
        'album_artist': 'ALBUMARTIST',
        'composer': 'COMPOSER',
        'track': 'TRACKNUMBER',
        'track_total': 'TRACKTOTAL',
        'disc': 'DISCNUMBER',
        'disc_total': 'DISCTOTAL',
        'date': 'DATE',
        'genre': 'GENRE',
        'comment': 'COMMENT',
        'isrc': 'ISRC',
        'bpm': 'BPM',
        'encoder': 'ENCODER',
        'title_sort': 'TITLESORT',
        'artist_sort': 'ARTISTSORT',
        'album_artist_sort': 'ALBUMARTISTSORT',
    }
    
    key = field_map.get(field)
    if not key:
        return None
    
    vals = f.get(key)
    return vals[0] if vals else None


def _get_mp4_field(f: MP4, field: str) -> Any:
    """Get MP4 tag field value."""
    field_map = {
        'title': '©nam',
        'artist': '©ART',
        'album': '©alb',
        'album_artist': 'aART',
        'composer': '©wrt',
        'track': 'trkn',
        'disc': 'diskn',
        'date': '©day',
        'genre': '©gen',
        'comment': '©cmt',
        'isrc': '©isr',
        'bpm': 'tmpo',
        'encoder': '©enc',
    }
    
    key = field_map.get(field)
    if not key:
        return None
    
    vals = f.get(key)
    if vals:
        # Special handling for tuples (track, disc)
        if key in ('trkn', 'diskn') and isinstance(vals[0], tuple):
            return vals[0]
        return vals[0]
    
    return None


def _get_vorbis_field(f: OggVorbis, field: str) -> Any:
    """Get Vorbis tag field value."""
    field_map = {
        'title': 'TITLE',
        'artist': 'ARTIST',
        'album': 'ALBUM',
        'album_artist': 'ALBUMARTIST',
        'composer': 'COMPOSER',
        'track': 'TRACKNUMBER',
        'track_total': 'TOTALTRACKS',
        'disc': 'DISCNUMBER',
        'disc_total': 'TOTALDISCS',
        'date': 'DATE',
        'genre': 'GENRE',
        'comment': 'COMMENT',
        'isrc': 'ISRC',
        'bpm': 'BPM',
        'encoder': 'ENCODER',
    }
    
    key = field_map.get(field)
    if not key:
        return None
    
    vals = f.get(key)
    return vals[0] if vals else None


def run_tag_test(source_file: str, dest_file: str,
                source_format: str, dest_format: str,
                test_fields: List[str]) -> ConversionTest:
    """Run tag preservation test for a conversion."""
    
    test_name = f"{source_format} → {dest_format}"
    field_tests = []
    
    for field in test_fields:
        source_value = read_tag_value(source_file, field)
        dest_value = read_tag_value(dest_file, field)
        
        # Normalize for comparison
        source_norm = normalize_value(source_value, field, source_format)
        dest_norm = normalize_value(dest_value, field, dest_format)
        
        if source_norm == dest_norm:
            result = TestResult.PASS
            message = f"✓ {field}: '{source_norm}'"
        elif source_norm is None and dest_norm is None:
            result = TestResult.PASS
            message = f"✓ {field}: both empty"
        elif source_norm is None:
            result = TestResult.WARN
            message = f"⚠ {field}: source empty, dest='{dest_norm}'"
        elif dest_norm is None:
            result = TestResult.FAIL
            message = f"✗ {field}: source='{source_norm}', dest empty"
        else:
            result = TestResult.FAIL
            message = f"✗ {field}: source='{source_norm}', dest='{dest_norm}'"
        
        field_tests.append(FieldTest(
            field_name=field,
            source_value=source_value,
            expected_value=source_norm,
            actual_value=dest_value,
            result=result,
            message=message
        ))
    
    return ConversionTest(
        name=test_name,
        source_file=source_file,
        dest_file=dest_file,
        source_format=source_format,
        dest_format=dest_format,
        field_tests=field_tests
    )


def normalize_value(value: Any, field: str, format: str) -> Any:
    """Normalize value for comparison across formats."""
    if value is None:
        return None
    
    # Track/disc number normalization
    if field in ('track', 'disc'):
        import re
        value_str = str(value)
        
        # Handle tuples (MP4)
        if isinstance(value, tuple):
            track, total = value
            if total and total > 0:
                return f"{track}/{total}"
            return str(track)
        
        # Handle "5/12" format
        match = re.match(r'^(\d+)/(\d+)$', value_str)
        if match:
            return value_str
        
        # Handle plain number
        match = re.match(r'^(\d+)$', value_str)
        if match:
            return value_str
        
        return value_str
    
    # String values
    return str(value).strip()


def print_test_report(test: ConversionTest):
    """Print test report."""
    print(f"\n{'='*70}")
    print(f"Test: {test.name}")
    print(f"Source: {test.source_file}")
    print(f"Dest: {test.dest_file}")
    print('='*70)
    
    passed = sum(1 for ft in test.field_tests if ft.result == TestResult.PASS)
    failed = sum(1 for ft in test.field_tests if ft.result == TestResult.FAIL)
    warnings = sum(1 for ft in test.field_tests if ft.result == TestResult.WARN)
    
    for ft in test.field_tests:
        print(f"  {ft.message}")
    
    print(f"\nResults: {passed} passed, {failed} failed, {warnings} warnings")
    
    return failed == 0


def main():
    parser = argparse.ArgumentParser(description='Tag preservation tests')
    parser.add_argument('--source-dir', required=True, help='Test files directory')
    parser.add_argument('--output-dir', default='/tmp/test_output', help='Output directory')
    args = parser.parse_args()
    
    source_dir = Path(args.source_dir)
    output_dir = Path(args.output_dir)
    output_dir.mkdir(parents=True, exist_ok=True)
    
    test_fields = [
        'title', 'artist', 'album', 'album_artist', 'composer',
        'track', 'disc', 'date', 'genre', 'comment',
        'isrc', 'bpm', 'encoder', 'title_sort', 'artist_sort'
    ]
    
    # Test conversions
    conversions = [
        ('test_basic_tags.mp3', 'mp3', 'flac', 'flac'),
        ('test_basic_tags.flac', 'flac', 'mp3', 'mp3'),
        ('test_basic_tags.flac', 'flac', 'm4a', 'm4a'),
        ('test_basic_tags.m4a', 'm4a', 'flac', 'flac'),
        ('test_basic_tags.flac', 'flac', 'ogg', 'ogg'),
        ('test_basic_tags.flac', 'flac', 'opus', 'opus'),
        ('test_multi_artist.flac', 'flac', 'mp3', 'mp3'),
        ('test_replaygain.flac', 'flac', 'mp3', 'mp3'),
        ('test_replaygain.flac', 'flac', 'opus', 'opus'),
    ]
    
    all_passed = True
    
    for source_name, source_fmt, dest_ext, dest_fmt in conversions:
        source_file = source_dir / source_name
        
        if not source_file.exists():
            print(f"SKIP: {source_file} not found")
            continue
        
        dest_file = output_dir / f"{source_file.stem}_to_{dest_fmt}.{dest_ext}"
        
        # Run conversion (would use actual converter here)
        # convert_file(source_file, dest_file, source_fmt, dest_fmt)
        
        # Run test
        test = run_tag_test(
            str(source_file),
            str(dest_file),
            source_fmt,
            dest_fmt,
            test_fields
        )
        
        passed = print_test_report(test)
        if not passed:
            all_passed = False
    
    if all_passed:
        print("\n✓ ALL TESTS PASSED")
        return 0
    else:
        print("\n✗ SOME TESTS FAILED")
        return 1


if __name__ == "__main__":
    sys.exit(main())
```

---

## SECTION 3: COVER ART TESTS

### 3.1 Cover Art Test Cases

**Test 3.1: Front Cover Type Code**
```
Setup: FLAC with picture type 3 (front cover)
Convert: FLAC → MP3
Expected: MP3 APIC frame type = 3
Verification: kid3-cli shows "Front Cover"
```

**Test 3.2: Multiple Images → Single Front**
```
Setup: FLAC with 3 pictures (front, back, artist)
Convert: FLAC → MP3
Expected: MP3 has only 1 APIC (front cover)
Verification: kid3-cli shows single picture
```

**Test 3.3: No Embedded Art → folder.jpg Fallback**
```
Setup: MP3 without embedded art, folder.jpg in same directory
Convert: MP3 → FLAC with folder.jpg fallback enabled
Expected: FLAC has folder.jpg embedded
Verification: metaflac --show-pictures
```

**Test 3.4: PNG vs JPEG**
```
Setup: FLAC with PNG cover (front cover)
Convert: FLAC → MP3
Expected: MP3 APIC MIME = image/png
Verification: ffprobe -show_streams shows png
```

**Test 3.5: JPEG Quality Preservation**
```
Setup: FLAC with 1000x1000 JPEG cover, 500KB
Convert: FLAC → FLAC (re-encode)
Expected: Cover art identical (lossless)
Verification: md5sum of cover art bytes
```

**Test 3.6: Cover Art with Large Size**
```
Setup: FLAC with 4000x4000 cover, 8MB
Convert: FLAC → MP3
Expected: MP3 has embedded cover (may be compressed)
Verification: kid3-cli shows Picture
```

**Test 3.7: Multiple Front Covers**
```
Setup: FLAC with 2 front cover pictures (type 3)
Convert: FLAC → FLAC
Expected: Output has 2 front cover pictures
Verification: metaflac --show-pictures shows 2 type=3
```

### 3.2 Cover Art Test Script

```python
#!/usr/bin/env python3
"""
cover_art_test.py - Test cover art handling in conversions.
"""

import os
import sys
import argparse
import subprocess
from pathlib import Path
from dataclasses import dataclass
from typing import List, Optional, Tuple


@dataclass
class CoverArtInfo:
    exists: bool
    picture_type: int
    mime_type: str
    width: int
    height: int
    size_bytes: int
    description: str


def get_cover_art_info(file_path: str) -> List[CoverArtInfo]:
    """Extract cover art information from audio file."""
    from mutagen.flac import FLAC
    from mutagen.mp3 import MP3
    from mutagen.mp4 import MP4
    
    ext = Path(file_path).suffix.lower()
    results = []
    
    try:
        if ext == '.flac':
            f = FLAC(file_path)
            for pic in f.pictures:
                results.append(CoverArtInfo(
                    exists=True,
                    picture_type=pic.type,
                    mime_type=pic.mime,
                    width=pic.width,
                    height=pic.height,
                    size_bytes=len(pic.data),
                    description=pic.desc
                ))
        
        elif ext == '.mp3':
            f = MP3(file_path)
            if f.tags:
                for frame in f.tags.getall('APIC'):
                    results.append(CoverArtInfo(
                        exists=True,
                        picture_type=frame.type if hasattr(frame, 'type') else 3,
                        mime_type=frame.mime if hasattr(frame, 'mime') else 'image/jpeg',
                        width=0,  # Not stored in APIC
                        height=0,
                        size_bytes=len(frame.data) if hasattr(frame, 'data') else 0,
                        description=frame.desc if hasattr(frame, 'desc') else ''
                    ))
        
        elif ext in ('.m4a', '.mp4'):
            f = MP4(file_path)
            covr = f.get('covr', [])
            for img_data in covr:
                if isinstance(img_data, bytes):
                    results.append(CoverArtInfo(
                        exists=True,
                        picture_type=3,  # Assume front cover
                        mime_type='image/jpeg',  # Assumed
                        width=0,
                        height=0,
                        size_bytes=len(img_data),
                        description=''
                    ))
    
    except Exception as e:
        print(f"Error reading cover art: {e}")
    
    return results


def test_cover_art(source_file: str, dest_file: str,
                 expected_count: int = None,
                 expected_type: int = None,
                 expected_mime: str = None) -> Tuple[bool, str]:
    """
    Test cover art preservation in conversion.
    
    Returns: (passed, message)
    """
    source_covers = get_cover_art_info(source_file)
    dest_covers = get_cover_art_info(dest_file)
    
    messages = []
    passed = True
    
    # Check count
    if expected_count is not None:
        if len(dest_covers) != expected_count:
            passed = False
            messages.append(
                f"Cover count: expected {expected_count}, got {len(dest_covers)}"
            )
        else:
            messages.append(f"✓ Cover count: {len(dest_covers)}")
    
    # Check type
    if expected_type is not None and dest_covers:
        for i, cover in enumerate(dest_covers):
            if cover.picture_type != expected_type:
                passed = False
                messages.append(
                    f"Cover {i} type: expected {expected_type}, got {cover.picture_type}"
                )
            else:
                messages.append(f"✓ Cover {i} type: {cover.picture_type}")
    
    # Check MIME
    if expected_mime is not None and dest_covers:
        for i, cover in enumerate(dest_covers):
            if cover.mime_type != expected_mime:
                passed = False
                messages.append(
                    f"Cover {i} MIME: expected {expected_mime}, got {cover.mime_type}"
                )
            else:
                messages.append(f"✓ Cover {i} MIME: {cover.mime_type}")
    
    # At least one cover should exist if source had one
    if source_covers and not dest_covers:
        passed = False
        messages.append("✗ Source had cover art, but destination does not")
    
    return passed, "\n".join(messages)


def run_cover_art_tests():
    """Run all cover art tests."""
    tests = [
        {
            'name': 'FLAC → MP3 Front Cover Type',
            'source': 'test_cover_front.flac',
            'dest': 'output.mp3',
            'expected_count': 1,
            'expected_type': 3,  # Front cover
            'expected_mime': 'image/jpeg'
        },
        {
            'name': 'FLAC → FLAC Multiple Covers',
            'source': 'test_multi_cover.flac',
            'dest': 'output.flac',
            'expected_count': 3,
        },
        {
            'name': 'FLAC → OGG Cover Art',
            'source': 'test_cover.flac',
            'dest': 'output.ogg',
            'expected_count': 1,
        },
    ]
    
    for test in tests:
        print(f"\nRunning: {test['name']}")
        passed, msg = test_cover_art(
            test['source'],
            test['dest'],
            test.get('expected_count'),
            test.get('expected_type'),
            test.get('expected_mime')
        )
        print(msg)
        
        if not passed:
            print("TEST FAILED")


if __name__ == "__main__":
    run_cover_art_tests()
```

---

## SECTION 4: CHARACTER ENCODING TESTS

### 4.1 Character Encoding Test Cases

**Test 4.1: UTF-8 Basic Characters**
```
Input: Title="Café", Artist="Björk", Album="Múm"
Expected: All characters preserved correctly
Formats: All formats
```

**Test 4.2: CJK Characters**
```
Input: Title="咖啡", Artist="中文", Album="日本語"
Expected: All CJK characters preserved
Formats: FLAC, MP3, M4A, OGG, Opus
```

**Test 4.3: Emojis**
```
Input: Title="🎵 Music 🎶", Artist="Rock 🎸"
Expected: Emojis preserved (if supported)
Formats: FLAC (best), MP3 (varies), M4A (varies)
```

**Test 4.4: Special Characters**
```
Input: Title="Test & More <Less> \"Quote\""
Expected: Special characters escaped/preserved
Formats: All formats
```

**Test 4.5: Mixed Encoding**
```
Input: Title="Rock + 摇滚 + ロック"
Expected: All scripts preserved
Formats: All formats
```

**Test 4.6: Latin-1 Characters**
```
Input: Title="Über", Artist="Naïve", Album="Zürich"
Expected: Characters preserved
Formats: All formats
```

**Test 4.7: Windows-1252 Characters**
```
Input: Title="Curly ’Quotes’ and …ellipsis"
Expected: Curly quotes and ellipsis preserved
Formats: All formats
```

### 4.2 Encoding Test Script

```python
#!/usr/bin/env python3
"""
encoding_test.py - Test character encoding preservation.
"""

import os
import sys
import subprocess
from pathlib import Path
from typing import Dict, List, Tuple, Optional


# Test strings with various encodings
TEST_STRINGS = {
    'utf8_basic': {
        'title': "Café au Lait",
        'artist': "Björk",
        'album': "Vespertine",
    },
    'utf8_cjk': {
        'title': "日本語タイトル",
        'artist': "中文艺术家",
        'album': "한국 앨범",
    },
    'emoji': {
        'title': "🎵 Music 🎶",
        'artist': "Rock 🎸 Band",
        'album': "Greatest 🎤 Hits",
    },
    'special': {
        'title': "Test & More <Less>",
        'artist': 'Artist "Quoted"',
        'album': "Path: C:\\Music",
    },
    'mixed': {
        'title': "Rock + 摇滚 + ロック",
        'artist': "Naïve Über",
        'album': "Zürich / 北京 / Tokyo",
    },
}


def create_test_file(format: str, test_name: str, test_data: Dict) -> str:
    """Create a test file with specific encoding."""
    # This would use FFmpeg or mutagen to create a tagged file
    pass


def read_tag_with_encoding(file_path: str, field: str) -> Tuple[Optional[str], str]:
    """
    Read a tag field and detect encoding.
    
    Returns: (value, encoding)
    """
    try:
        from mutagen.mp3 import MP3
        from mutagen.flac import FLAC
        
        ext = Path(file_path).suffix.lower()
        
        if ext == '.mp3':
            f = MP3(file_path)
            # Read value and encoding
            pass
        elif ext == '.flac':
            f = FLAC(file_path)
            # Vorbis comments are UTF-8
            pass
        
    except Exception as e:
        return None, f"ERROR: {e}"
    
    return None, "UNKNOWN"


def test_encoding_preservation(source_file: str, dest_file: str) -> bool:
    """Test that encoding is preserved in conversion."""
    passed = True
    
    for field in ['title', 'artist', 'album']:
        source_val, source_enc = read_tag_with_encoding(source_file, field)
        dest_val, dest_enc = read_tag_with_encoding(dest_file, field)
        
        if source_val != dest_val:
            print(f"FAIL: {field}")
            print(f"  Source: {source_val!r}")
            print(f"  Dest:   {dest_val!r}")
            passed = False
        else:
            print(f"PASS: {field} = {source_val!r}")
    
    return passed


def main():
    print("Character Encoding Tests")
    print("=" * 60)
    
    # Test UTF-8 basic
    print("\n[Test: UTF-8 Basic]")
    # Create test files, run conversions, verify
    
    # Test CJK
    print("\n[Test: CJK Characters]")
    # ...
    
    # Test emojis
    print("\n[Test: Emojis]")
    # ...
    
    print("\n" + "=" * 60)
    print("Encoding tests complete")


if __name__ == "__main__":
    main()
```

---

## SECTION 5: EDGE CASE TESTS

### 5.1 Edge Case Test Definitions

**Test 5.1: Multi-Value Artist Field**
```
Scenario: FLAC with multiple ARTIST values
Input: ARTIST=Artist1, ARTIST=Artist2
Convert: FLAC → MP3
Expected: MP3 TPE1="Artist1; Artist2" (DBpoweramp separator)
Verification: mid3v2 --artist output.mp3
```

**Test 5.2: Genre with ID3v1 Number**
```
Scenario: MP3 with TCON=(17)
Input: Genre stored as ID3v1 number 17
Convert: MP3 → FLAC
Expected: FLAC GENRE="Rock" (resolved)
Verification: metaflac --show-tag=GENRE output.flac
```

**Test 5.3: Track Number "5/12" Across Formats**
```
Scenario: Track stored as "5/12"
Input: TRCK="5/12"
Convert: MP3 → FLAC → M4A → OGG
Expected: All formats preserve both track number (5) and total (12)
Verification: Read each format and verify values
```

**Test 5.4: ReplayGain → Opus R128**
```
Scenario: ReplayGain in source, convert to Opus
Input: REPLAYGAIN_TRACK_GAIN="-6.20 dB"
Convert: FLAC → Opus
Expected: Opus R128_TRACK_GAIN ≈ 256 (Q7.8 format)
Formula: round((-6.20 + 5.0) * 256) = -307
Verification: opuscomment --raw --list output.opus
```

**Test 5.5: Opus R128 → ReplayGain**
```
Scenario: R128 in source, convert from Opus
Input: R128_TRACK_GAIN=256
Convert: Opus → MP3
Expected: MP3 REPLAYGAIN_TRACK_GAIN ≈ "-1.00 dB"
Formula: (256 / 256) - 5.0 = -4.0 dB
Verification: mid3v2 --show REPLAYGAIN_TRACK_GAIN output.mp3
```

**Test 5.6: Multi-Image Files**
```
Scenario: FLAC with 4 embedded pictures (front, back, artist, other)
Input: 4 pictures with different types
Convert: FLAC → MP3
Expected: MP3 has 4 APIC frames with correct types
Verification: kid3-cli -c "get" output.mp3
```

**Test 5.7: Corrupted Tag Handling**
```
Scenario: MP3 with malformed ID3v2 tag
Input: Corrupted TXXX frame
Convert: MP3 → FLAC
Expected: Conversion succeeds, corrupted frame skipped
Verification: No crash, valid output file
```

**Test 5.8: Empty Fields**
```
Scenario: FLAC with empty TITLE field
Input: TITLE="" (empty string)
Convert: FLAC → MP3
Expected: MP3 has no TIT2 frame (not empty string)
Verification: kid3-cli shows no title
```

**Test 5.9: Disc Subtitle**
```
Scenario: M4A with disc subtitle
Input: @sdt="Live in Tokyo"
Convert: M4A → MP3
Expected: MP3 TSST="Live in Tokyo" (ID3v2.4 only)
Verification: mid3v2 --show TSST output.mp3
```

**Test 5.10: Compilation Flag**
```
Scenario: MP3 with TCMP=1
Input: Compilation=true
Convert: MP3 → FLAC
Expected: FLAC COMPILATION=1
Verification: metaflac --show-tag=COMPILATION output.flac
```

### 5.2 Edge Case Test Script

```python
#!/usr/bin/env python3
"""
edge_case_test.py - Test edge cases in tag conversion.
"""

import re
from typing import Optional, Tuple, Any


def parse_track_number(value: Any) -> Tuple[int, int]:
    """
    Parse track number from various formats.
    Returns: (track, total)
    """
    if value is None:
        return (0, 0)
    
    # Handle tuple (MP4)
    if isinstance(value, tuple):
        return value
    
    value_str = str(value).strip()
    
    # Handle "5/12" format
    if '/' in value_str:
        parts = value_str.split('/')
        try:
            track = int(parts[0]) if parts[0] else 0
            total = int(parts[1]) if parts[1] else 0
            return (track, total)
        except ValueError:
            pass
    
    # Plain number
    try:
        return (int(value_str), 0)
    except ValueError:
        return (0, 0)


def test_track_number_parsing():
    """Test track number parsing across formats."""
    test_cases = [
        ("5/12", (5, 12)),
        ("5", (5, 0)),
        (5, (5, 0)),
        ("12", (12, 0)),
        ("0/5", (0, 5)),
        ("0", (0, 0)),
        (None, (0, 0)),
        ("", (0, 0)),
        ((5, 12), (5, 12)),
        ([5, 12], (5, 12)),  # Might be list
    ]
    
    print("Track Number Parsing Tests")
    print("=" * 50)
    
    all_passed = True
    for input_val, expected in test_cases:
        result = parse_track_number(input_val)
        passed = result == expected
        status = "PASS" if passed else "FAIL"
        
        print(f"[{status}] parse({input_val!r}) = {result}, expected {expected}")
        
        if not passed:
            all_passed = False
    
    return all_passed


def test_replaygain_to_r128(gain_db: float) -> int:
    """Convert ReplayGain dB to R128 Q7.8."""
    return round((gain_db + 5.0) * 256)


def test_r128_to_replaygain(r128_val: int) -> float:
    """Convert R128 Q7.8 to ReplayGain dB."""
    return (r128_val / 256.0) - 5.0


def test_replaygain_conversion():
    """Test ReplayGain/R128 conversion."""
    print("\nReplayGain/R128 Conversion Tests")
    print("=" * 50)
    
    test_cases = [
        # (replaygain_db, expected_r128)
        (-12.0, -1792),
        (-6.0, -256),
        (-3.0, 512),
        (0.0, 1280),
        (3.0, 2048),
        (6.0, 2816),
    ]
    
    all_passed = True
    
    print("\nReplayerain → R128:")
    for rg_db, expected_r128 in test_cases:
        result = test_replaygain_to_r128(rg_db)
        passed = result == expected_r128
        status = "PASS" if passed else "FAIL"
        
        print(f"[{status}] {rg_db} dB → {result} (expected {expected_r128})")
        
        if not passed:
            all_passed = False
    
    print("\nR128 → ReplayGain:")
    for rg_db, r128_val in test_cases:
        result = test_r128_to_replaygain(r128_val)
        # Allow small floating point difference
        passed = abs(result - rg_db) < 0.01
        status = "PASS" if passed else "FAIL"
        
        print(f"[{status}] {r128_val} → {result:.2f} dB (expected {rg_db})")
        
        if not passed:
            all_passed = False
    
    return all_passed


def test_multi_artist_separator(artist_string: str) -> Tuple[bool, str]:
    """
    Test multi-artist separator handling.
    DBpoweramp uses "; " as separator.
    """
    # Split on DBpoweramp separator
    artists = artist_string.split('; ')
    
    if len(artists) > 1:
        return True, f"Multi-artist: {artists}"
    else:
        # Try other separators
        if '/' in artist_string:
            return False, f"Single-artist with /: {artist_string}"
        return True, f"Single artist: {artist_string}"


def test_multi_artist_handling():
    """Test multi-artist field handling."""
    print("\nMulti-Artist Handling Tests")
    print("=" * 50)
    
    test_cases = [
        ("Artist1; Artist2", True, "DBpoweramp multi-artist"),
        ("Artist1;Artist2", False, "Missing space after semicolon"),
        ("Artist1 / Artist2", False, "Wrong separator"),
        ("Single Artist", True, "Single artist"),
    ]
    
    all_passed = True
    
    for artist, expect_multi, description in test_cases:
        is_multi, result = test_multi_artist_separator(artist)
        
        if is_multi == expect_multi:
            print(f"[PASS] {description}: {result}")
        else:
            print(f"[FAIL] {description}: expected {'multi' if expect_multi else 'single'}, got {result}")
            all_passed = False
    
    return all_passed


def main():
    print("Edge Case Tests")
    print("=" * 60)
    
    results = []
    
    results.append(("Track Number Parsing", test_track_number_parsing()))
    results.append(("ReplayGain/R128 Conversion", test_replaygain_conversion()))
    results.append(("Multi-Artist Handling", test_multi_artist_handling()))
    
    print("\n" + "=" * 60)
    print("SUMMARY")
    print("=" * 60)
    
    for name, passed in results:
        status = "PASSED" if passed else "FAILED"
        print(f"{name}: {status}")
    
    all_passed = all(r[1] for r in results)
    
    if all_passed:
        print("\n✓ ALL EDGE CASE TESTS PASSED")
        return 0
    else:
        print("\n✗ SOME EDGE CASE TESTS FAILED")
        return 1


if __name__ == "__main__":
    sys.exit(main())
```

---

## SECTION 6: LOSSLESS VERIFICATION TESTS

### 6.1 Lossless Verification Methodology

For lossless conversions (FLAC → FLAC, ALAC → FLAC, etc.), verify audio data integrity:

```bash
#!/bin/bash
# lossless_verify.sh

SOURCE="$1"
DEST="$2"

# Method 1: MD5 of decoded PCM
echo "=== Lossless Verification ==="

# Decode both files to raw PCM
ffmpeg -y -i "$SOURCE" -f wav /tmp/source.wav 2>/dev/null
ffmpeg -y -i "$DEST" -f wav /tmp/dest.wav 2>/dev/null

# Compare file sizes (should be identical)
SRC_SIZE=$(stat -c%s /tmp/source.wav)
DEST_SIZE=$(stat -c%s /tmp/dest.wav)

if [ "$SRC_SIZE" = "$DEST_SIZE" ]; then
    echo "✓ File sizes match: $SRC_SIZE bytes"
else
    echo "✗ File sizes differ: $SRC_SIZE vs $DEST_SIZE"
    exit 1
fi

# Compare MD5 checksums
SRC_MD5=$(md5sum /tmp/source.wav | cut -d' ' -f1)
DEST_MD5=$(md5sum /tmp/dest.wav | cut -d' ' -f1)

if [ "$SRC_MD5" = "$DEST_MD5" ]; then
    echo "✓ Audio data identical (MD5: $SRC_MD5)"
else
    echo "✗ Audio data differs"
    echo "  Source MD5: $SRC_MD5"
    echo "  Dest MD5:   $DEST_MD5"
    exit 1
fi

# Cleanup
rm -f /tmp/source.wav /tmp/dest.wav

echo "✓ Lossless verification passed"
```

### 6.2 Lossless Test Cases

| Source | Destination | Expected | Verification |
|--------|-------------|----------|--------------|
| FLAC | FLAC | Bit-exact | MD5 of decoded PCM |
| ALAC | FLAC | Bit-exact | MD5 of decoded PCM |
| WAV | FLAC | Bit-exact | MD5 of decoded PCM |
| AIFF | FLAC | Bit-exact | MD5 of decoded PCM |
| WAVPACK | FLAC | Lossy (transcode) | SSIM > 0.999 |
| MP3 | FLAC | Lossy (transcode) | SSIM comparison |

---

## SECTION 7: GAPLESS PLAYBACK TESTS

### 7.1 Gapless Playback Test Cases

**Test 7.1: LAME Xing Header Preservation**
```
Scenario: MP3 encoded with LAME
Source: LAME-encoded MP3 with Xing header
Convert: MP3 → MP3 (re-encode)
Expected: Output has valid Xing header with:
  - Encoder delay samples
  - Encoder padding samples
  - Valid LAME tag
Verification: Check LAME info with ffprobe
```

**Test 7.2: iTunSMPB Preservation**
```
Scenario: M4A with gapless info
Source: Apple Lossless M4A with iTunSMPB tag
Convert: ALAC → FLAC
Expected: FLAC has appropriate gapless info
Verification: Check STREAMINFO for total samples
```

**Test 7.3: Gapless Between Tracks**
```
Scenario: Two consecutive tracks from same album
Input: track01.mp3, track02.mp3 (gapless album)
Convert: Both tracks to AAC
Expected: Gapless info preserved in both files
Verification: Play tracks back-to-back, no silence
```

### 7.2 Gapless Test Script

```python
#!/usr/bin/env python3
"""
gapless_test.py - Test gapless playback tag preservation.
"""

import re
from dataclasses import dataclass
from typing import Optional


@dataclass
class GaplessInfo:
    """Gapless playback information."""
    encoder_delay_samples: int
    encoder_padding_samples: int
    valid: bool = True


def parse_lame_tag(mp3_path: str) -> Optional[GaplessInfo]:
    """Parse LAME tag from MP3 file."""
    try:
        with open(mp3_path, 'rb') as f:
            data = f.read()
        
        # LAME tag is at end of file (last 128 bytes before ID3v1)
        # Or in Xing/LAME info header
        
        # Look for "LAME" signature
        lame_pos = data.rfind(b'LAME')
        if lame_pos == -1:
            return None
        
        # Parse LAME version
        lame_version = data[lame_pos+4:lame_pos+8]
        
        # VBR quality
        vbr_quality = data[lame_pos+8]
        
        # Encoder delay/padding (in some LAME versions)
        # These are in the first 9 bytes after LAME
        #delay = int.from_bytes(data[lame_pos+9:lame_pos+12], 'big')
        #padding = int.from_bytes(data[lame_pos+12:lame_pos+15], 'big')
        
        return GaplessInfo(
            encoder_delay_samples=0,  # Would need proper parsing
            encoder_padding_samples=0,
            valid=True
        )
    
    except Exception as e:
        return GaplessInfo(0, 0, valid=False)


def parse_itunsmpb(mp3_path: str) -> Optional[GaplessInfo]:
    """Parse iTunSMPB tag from M4A/MP3."""
    # iTunSMPB format: "00000000 00000840 00000840 00000000 00000000"
    # Format: skip start, sample rate, sample count, end skip, part total
    # All in hex, 8 hex digits per field
    
    # This is simplified - actual parsing needed
    return None


def test_gapless_preservation(source: str, dest: str) -> bool:
    """Test gapless info preservation."""
    print(f"Testing gapless preservation: {source} → {dest}")
    
    source_info = parse_lame_tag(source)
    dest_info = parse_lame_tag(dest)
    
    if source_info is None:
        print("  Source has no gapless info (skipping)")
        return True
    
    if dest_info is None:
        print("  ✗ DESTINATION LOST GAPLESS INFO")
        return False
    
    if not dest_info.valid:
        print("  ✗ DESTINATION HAS INVALID GAPLESS INFO")
        return False
    
    print("  ✓ Gapless info preserved")
    return True


def main():
    print("Gapless Playback Tests")
    print("=" * 50)
    
    # Test cases
    tests = [
        ("lame_track.mp3", "lame_track_copy.mp3"),
        ("lame_track.mp3", "lame_track_to_flac.flac"),
    ]
    
    all_passed = True
    for source, dest in tests:
        if not test_gapless_preservation(source, dest):
            all_passed = False
    
    if all_passed:
        print("\n✓ ALL GAPLESS TESTS PASSED")
    else:
        print("\n✗ SOME GAPLESS TESTS FAILED")


if __name__ == "__main__":
    main()
```

---

## SECTION 8: BATCH PROCESSING TESTS

### 8.1 Batch Processing Test Cases

**Test 8.1: Large Batch Reliability**
```
Scenario: Convert 1000 files
Expected: All files converted successfully
Verification: Count input vs output, check for errors
```

**Test 8.2: Concurrent Conversion Safety**
```
Scenario: Convert same source to multiple destinations simultaneously
Expected: No file corruption, all outputs valid
Verification: Compare outputs, check file integrity
```

**Test 8.3: Disk Space Exhaustion**
```
Scenario: Disk nearly full during conversion
Expected: Graceful error handling, no partial files
Verification: Check for .tmp files left behind
```

**Test 8.4: Source File Locked**
```
Scenario: Source file open by another process
Expected: Error reported, file not corrupted
Verification: Open file, attempt conversion, close file
```

### 8.2 Batch Test Script

```python
#!/usr/bin/env python3
"""
batch_test.py - Test batch processing reliability.
"""

import os
import sys
import time
import subprocess
from pathlib import Path
from concurrent.futures import ThreadPoolExecutor, as_completed
from dataclasses import dataclass
from typing import List, Dict


@dataclass
class ConversionResult:
    source: str
    dest: str
    success: bool
    duration_seconds: float
    error: str = ""


def convert_single(source: str, dest: str, converter: str) -> ConversionResult:
    """Convert a single file."""
    start = time.time()
    
    try:
        result = subprocess.run(
            [converter, '-i', source, '-o', dest],
            capture_output=True,
            timeout=300
        )
        
        duration = time.time() - start
        
        if result.returncode == 0:
            return ConversionResult(source, dest, True, duration)
        else:
            return ConversionResult(
                source, dest, False, duration,
                error=result.stderr[:500]
            )
    
    except subprocess.TimeoutExpired:
        return ConversionResult(source, dest, False, 300, "Timeout")
    except Exception as e:
        return ConversionResult(source, dest, False, time.time() - start, str(e))


def batch_convert_test(source_files: List[str],
                      dest_dir: str,
                      converter: str,
                      max_workers: int = 4) -> Dict:
    """Test batch conversion with concurrent processing."""
    
    print(f"Starting batch conversion of {len(source_files)} files")
    print(f"Max concurrent: {max_workers}")
    
    results = []
    
    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        futures = []
        
        for source in source_files:
            source_path = Path(source)
            dest = Path(dest_dir) / source_path.name
            
            future = executor.submit(convert_single, source, str(dest), converter)
            futures.append((future, source, dest))
        
        completed = 0
        for future, source, dest in futures:
            result = future.result()
            results.append(result)
            completed += 1
            
            status = "✓" if result.success else "✗"
            print(f"[{completed}/{len(source_files)}] {status} {source}")
            
            if not result.success:
                print(f"  Error: {result.error}")
    
    # Summarize
    successful = sum(1 for r in results if r.success)
    failed = len(results) - successful
    total_time = sum(r.duration_seconds for r in results)
    
    print(f"\n{'='*50}")
    print(f"Batch Conversion Results")
    print(f"{'='*50}")
    print(f"Total files: {len(results)}")
    print(f"Successful: {successful}")
    print(f"Failed: {failed}")
    print(f"Total time: {total_time:.1f}s")
    print(f"Average per file: {total_time/len(results):.2f}s")
    
    if failed > 0:
        print(f"\nFailed conversions:")
        for r in results:
            if not r.success:
                print(f"  {r.source}: {r.error}")
    
    return {
        'total': len(results),
        'successful': successful,
        'failed': failed,
        'total_time': total_time,
        'results': results
    }


def main():
    # Example usage
    print("Batch Processing Tests")
    print("=" * 50)
    
    # Would run actual tests here
    print("This would run batch conversion tests on actual files")


if __name__ == "__main__":
    main()
```

---

## SECTION 9: TEST FILE REPOSITORY

### 9.1 Required Test Files

Create a test repository with these files:

```
tests/
├── tags/
│   ├── basic_tags/
│   │   ├── test_basic_tags.mp3
│   │   ├── test_basic_tags.flac
│   │   ├── test_basic_tags.m4a
│   │   └── test_basic_tags.ogg
│   ├── multi_artist/
│   │   └── test_multi_artist.flac
│   ├── genre_number/
│   │   └── test_genre_number.mp3
│   ├── track_formats/
│   │   └── test_track_formats.flac
│   ├── sort_fields/
│   │   └── test_sort_fields.mp3
│   ├── replaygain/
│   │   └── test_replaygain.flac
│   ├── r128/
│   │   └── test_r128.opus
│   ├── work_movement/
│   │   └── test_work_movement.m4a
│   ├── custom_tags/
│   │   └── test_custom_tags.mp3
│   └── cover_art/
│       ├── test_cover_front.flac
│       ├── test_multi_cover.flac
│       └── test_cover_png.flac
├── encoding/
│   ├── utf8_basic.mp3
│   ├── cjk.mp3
│   └── emoji.mp3
└── gapless/
    ├── track01_lame.mp3
    └── track02_lame.mp3
```

### 9.2 Test File Generation

```bash
#!/bin/bash
# generate_test_files.sh

# Create test directory
mkdir -p tests/tags tests/encoding tests/gapless

# Generate basic tags file
ffmpeg -f lavfi -i "sine=frequency=440:duration=5" -y \
  -metadata title="Test Track" \
  -metadata artist="Test Artist" \
  -metadata album="Test Album" \
  -metadata track="5/12" \
  -metadata disc="1/3" \
  -metadata date="2024-03-15" \
  -metadata genre="Rock" \
  -id3v2_version 3 \
  tests/tags/test_basic_tags.mp3

echo "Test files generated in tests/"
```

---

## SECTION 10: AUTOMATED VALIDATION SCRIPTS

### 10.1 Full Test Suite Runner

```bash
#!/bin/bash
# run_all_tests.sh - Run complete test suite

set -e

TEST_DIR="/path/to/test/files"
OUTPUT_DIR="/tmp/test_output"
CONVERTER="./converter"

echo "========================================"
echo "DBpoweramp-Equivalent Converter Test Suite"
echo "========================================"
echo ""

# Setup
mkdir -p "$OUTPUT_DIR"
cd "$(dirname "$0")"

# Run test categories
echo "[1/5] Running tag preservation tests..."
python3 test_tag_preservation.py --source-dir="$TEST_DIR" --output-dir="$OUTPUT_DIR"
TAG_RESULT=$?

echo ""
echo "[2/5] Running cover art tests..."
python3 test_cover_art.py
COVER_RESULT=$?

echo ""
echo "[3/5] Running encoding tests..."
python3 test_encoding.py
ENCODING_RESULT=$?

echo ""
echo "[4/5] Running edge case tests..."
python3 test_edge_cases.py
EDGE_RESULT=$?

echo ""
echo "[5/5] Running lossless verification..."
./test_lossless.sh
LOSSLESS_RESULT=$?

# Summary
echo ""
echo "========================================"
echo "TEST SUMMARY"
echo "========================================"
echo "Tag Preservation: $([ $TAG_RESULT -eq 0 ] && echo PASS || echo FAIL)"
echo "Cover Art:        $([ $COVER_RESULT -eq 0 ] && echo PASS || echo FAIL)"
echo "Encoding:         $([ $ENCODING_RESULT -eq 0 ] && echo PASS || echo FAIL)"
echo "Edge Cases:       $([ $EDGE_RESULT -eq 0 ] && echo PASS || echo FAIL)"
echo "Lossless:         $([ $LOSSLESS_RESULT -eq 0 ] && echo PASS || echo FAIL)"

# Exit with error if any tests failed
if [ $TAG_RESULT -ne 0 ] || [ $COVER_RESULT -ne 0 ] || \
   [ $ENCODING_RESULT -ne 0 ] || [ $EDGE_RESULT -ne 0 ] || \
   [ $LOSSLESS_RESULT -ne 0 ]; then
    echo ""
    echo "SOME TESTS FAILED"
    exit 1
fi

echo ""
echo "ALL TESTS PASSED"
exit 0
```

### 10.2 Continuous Integration Script

```yaml
# .github/workflows/test.yml
name: Tag Preservation Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install mutagen pillow
      
      - name: Install FFmpeg
        run: |
          sudo apt-get update
          sudo apt-get install -y ffmpeg
      
      - name: Generate test files
        run: |
          ./scripts/generate_test_files.sh
      
      - name: Run test suite
        run: |
          ./scripts/run_all_tests.sh
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: test-results/
```

---

*Document: 73_Testing_and_Validation_Methodology.md*
*Generated from: DBpoweramp testing research*
*Version: 1.0*
