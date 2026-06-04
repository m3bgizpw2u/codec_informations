# Audio Quality Verification Post-Encode
*Generated: 2026-06-04 | Sources: 8 | Confidence: High*

## Executive Summary

DBpoweramp provides multiple post-encoding verification methods including bit-exact comparison for lossless formats, duration verification, sample rate/bit depth checks, peak level detection, and bitrate verification. The Test Conversion feature scans files for corruption by performing a decode pass without creating output. PerfectTUNES extends verification with AccurateRip integration, De-Dup (duplicate detection via audio fingerprinting), and enhanced ReplayGain. Verification results are displayed in the UI and logged, with MD5 checksums used for bit-exact comparison.

## 1. Post-Encoding Verification Overview

### 1.1 Why Verify?
Post-encoding verification ensures:
- Audio data not corrupted during conversion
- Format parameters match specifications
- Bit-exact preservation for lossless formats
- No clipping or distortion introduced

### 1.2 Verification Timing
Verification can occur at multiple stages:
1. **During encoding**: Internal consistency checks
2. **Immediately after**: Quick validation
3. **Later**: Periodic library verification
4. **Pre-Delete**: Before removing source

### 1.3 DBpoweramp Verification Features
| Feature | Purpose | Lossless | Lossy |
|---------|---------|----------|-------|
| Test Conversion | Corruption check | ✓ | ✓ |
| AccurateRip | Community verification | ✓ | - |
| Duration check | Length validation | ✓ | ✓ |
| Parameter check | SR/bit depth | ✓ | ✓ |
| Peak detection | Clipping check | ✓ | ✓ |

## 2. Test Conversion

### 2.1 What is Test Conversion?
Test Conversion performs a complete decode-encode cycle without writing output:
1. Read source file
2. Decode to PCM
3. Encode to target format
4. Decode output
5. Compare against original decode
6. Report any mismatches

### 2.2 Running Test Conversion
In Batch Converter:
1. Select files to test
2. Choose encoder: "Test Conversion"
3. Select same encoder as planned conversion
4. Click Convert
5. Errors are reported; no files written

### 2.3 Test Conversion Results
| Result | Meaning |
|--------|---------|
| No errors | File passes verification |
| `md5 did not match` | File is corrupt |
| `Error opening file` | File unreadable |
| `Decoding error` | File damaged |

### 2.4 When to Use
Test Conversion is ideal for:
- Verifying existing library integrity
- Before batch conversion (quality check)
- After file operations (copy, move)
- Periodic library maintenance

## 3. Bit-Exact Comparison (Lossless)

### 3.1 MD5 Checksum Verification
For lossless formats, DBpoweramp compares MD5 checksums:
- **Source MD5**: Calculated from decoded source
- **Output MD5**: Calculated from decoded output
- **Match**: Bit-exact preservation confirmed
- **Mismatch**: Corruption occurred

### 3.2 What MD5 Verifies
MD5 comparison confirms:
- All samples identical
- No data corruption
- Proper format conversion
- Tag writing didn't corrupt audio

### 3.3 Limitations of MD5
MD5 cannot detect:
- Bit-exactness of lossy formats (expected differences)
- Perceptual quality
- Metadata quality

### 3.4 FLAC Verification
FLAC files include internal checksums:
- Frame checksums (CRC)
- Stream checksum
- dBpoweramp verifies these during decode

## 4. Duration Verification

### 4.1 Duration Check
Compare source and output durations:
- Same length expected for lossless
- Small differences for lossy (padding)
- Large differences indicate error

### 4.2 Acceptable Duration Differences
| Format | Expected Difference |
|--------|-------------------|
| FLAC | Exact match |
| WAV | Exact match |
| ALAC | Exact match |
| MP3 | ±0-50ms (padding) |
| AAC | ±0-50ms (padding) |

### 4.3 Duration Mismatch Causes
| Cause | Implication |
|-------|-------------|
| Source trimmed | Sample lost |
| Output padded | Extraneous data |
| Decode error | Corruption |
| Format limit | Normal (MP3 padding) |

## 5. Sample Rate and Bit Depth Verification

### 5.1 Parameter Checks
Verify output parameters match specification:
- **Sample rate**: Expected vs actual
- **Bit depth**: Expected vs actual
- **Channels**: Expected vs actual
- **Codec**: Expected vs actual

### 5.2 DBpoweramp Verification
During encoding:
- Target parameters specified
- Output header checked
- Mismatch reported as error

### 5.3 Verification Command
Using MediaInfo or similar:
```
MediaInfo "file.flac" | grep -E "Sampling rate|Bit depth|Channels"
```

### 5.4 Common Mismatches
| Expected | Got | Cause |
|----------|-----|-------|
| 44100 Hz | 48000 Hz | SRC misconfigured |
| 16-bit | 24-bit | Dither/noise |
| Stereo | Mono | Downmix enabled |

## 6. Peak Level Detection (Clipping Warning)

### 6.1 Clipping Detection
Peak level analysis detects:
- Samples at or near 0 dBFS (±1.0)
- Intersample peaks (for lossy)
- Potential distortion in playback

### 6.2 DSP Effect Integration
Peak detection during conversion:
- Applied as DSP effect
- Checks output after processing
- Reports if clipping detected

### 6.3 True Peak Detection
For lossy formats:
- Oversampled peak detection
- Catches inter-sample peaks
- More accurate than sample peak

### 6.4 Clipping Thresholds
| Threshold | Action |
|-----------|--------|
| No peaks > 0.99 | Safe |
| Peaks at 0.99-1.0 | Warning |
| Peaks > 1.0 | Clipping |

## 7. Bitrate Verification

### 7.1 Actual vs Target Bitrate
Compare expected vs actual:
- Lossless: Variable, but in expected range
- Lossy CBR: Should match exactly
- Lossy VBR: Average in expected range

### 7.2 Bitrate Monitoring
During encoding:
- Real-time bitrate displayed
- Significant deviation flagged
- Encoder settings can be adjusted

### 7.3 Bitrate Ranges by Format
| Format | Expected Range |
|--------|---------------|
| FLAC | 700-1200 kbps |
| ALAC | 700-1200 kbps |
| MP3 320 | 320 kbps |
| AAC 256 | 220-280 kbps |
| Opus 128 | 100-160 kbps |

### 7.4 Abnormal Bitrate Causes
| Symptom | Cause |
|---------|-------|
| MP3 at 128 instead of 320 | Setting wrong |
| FLAC at 300 kbps | Source is not lossless |
| VBR outside range | Source issue |

## 8. PerfectTUNES Verification Suite

### 8.1 PerfectTUNES Overview
PerfectTUNES is a separate utility from dBpoweramp providing:
- AccurateRip verification
- De-Dup (duplicate detection)
- ID Tag editing
- Album art management
- ReplayGain calculation

### 8.2 PerfectTUNES AccurateRip
AccurateRip verification for existing files:
- Compare against online database
- Verify previously ripped discs
- Perfect for non-dBpoweramp rips
- Detects corruption from scratches/disc rot

### 8.3 PerfectTUNES Features
| Feature | Purpose |
|---------|---------|
| **AccurateRip** | Community verification |
| **De-Dup** | Duplicate detection |
| **ID Tag** | Metadata editing |
| **Artwork** | Cover management |
| **ReplayGain** | Loudness calculation |

### 8.4 De-Dup (Duplicate Detection)
PerfectTUNES De-Dup analyzes audio content:
- **Certain**: 100% identical audio
- **Possible**: Similar but not identical
- Audio fingerprinting (not just metadata)
- Cover versions detected as different

## 9. AccurateRip Integration

### 9.1 What AccurateRip Verifies
AccurateRip checks:
- Your rip matches other users' rips
- Disc ID computed from file metadata
- Track offsets verified
- Checksum comparison

### 9.2 PerfectTUNES AccurateRip Requirements
| Requirement | Details |
|-------------|---------|
| Format | Lossless only (FLAC, ALAC) |
| Complete | All tracks present |
| Tagged | AccurateRip tags present |
| Database | Disc in AccurateRip DB |

### 9.3 AccurateRip Settings
| Option | Description |
|--------|-------------|
| Album Matching | Group by folder |
| Use AR Result Tag | Use cached result |
| Delete Corrupted | Move to Recycle Bin |
| Confidence 2+ | Stricter matching |

### 9.4 Verification Results
| Result | Meaning |
|--------|---------|
| Accurately Ripped | Error-free (confidence N) |
| Cannot Check | Format not supported |
| Errors Found | Corruption detected |
| Not in DB | Unknown disc |

## 10. Verification Results Display and Logging

### 10.1 UI Display
During verification:
- Progress indicator
- Current file being checked
- Result per file (pass/fail)
- Summary at completion

### 10.2 Log Output
Test Conversion log entries:
```
Information converting to Test Conversion, 'Z:\album\track01.flac'
  to 'Z:\album\track01.IGNORE'
  1 errors / information messages whilst decoding:
  md5 did not match decoded data, file is corrupt.
```

### 10.3 Error Reporting
Errors reported with:
- Full file path
- Specific error type
- Technical details
- Recovery suggestions

### 10.4 Log File Location
| Context | Location |
|---------|----------|
| Conversion | On-screen / copyable |
| Secure Rip | Music Folder (text file) |
| PerfectTUNES | PerfectTUNES interface |

## 11. Edge Cases

### 11.1 Partial File Verification
File partially verified:
- Complete portion checked
- Incomplete portion reported
- Re-conversion required

### 11.2 Multi-Disc Verification
Album across multiple discs:
- Each disc verified separately
- Cross-disc comparison not available
- Individual AccurateRip checks

### 11.3 Network File Verification
Files on network drives:
- Latency affects timing
- Connection interruption handling
- Recommend local copy for critical

### 11.4 Modified Files After Encoding
Post-encoding changes:
- Re-verification needed
- MD5 won't match original
- Manual verification required

### 11.5 Mismatch with Source Removed
If source deleted:
- Cannot re-verify
- Trust output file only
- Recommend keeping sources

### 11.6 Format-Specific Quirks
| Format | Quirk |
|--------|-------|
| MP3 VBR | Padding affects MD5 |
| AAC | Encrypted files cannot verify |
| DSD | Requires PCM conversion first |
| M4A | Bitrate varies significantly |

## 12. Would a User Notice a Difference?

### From DBpoweramp vs No Verification

| Scenario | Without Verification | With Verification |
|----------|---------------------|-------------------|
| Corrupt file | Silent playback issues | Error reported |
| Bitrate mismatch | Wrong quality | Detected |
| Clipping | Distorted audio | Warning shown |
| Duration mismatch | Missing audio | Error reported |

### From PerfectTUNES vs Core DBpoweramp

| Feature | DBpoweramp | PerfectTUNES |
|---------|-----------|--------------|
| AccurateRip | During rip only | Any existing file |
| Duplicate detection | None | Audio fingerprinting |
| Tag editing | Basic | Advanced |
| Batch ReplayGain | Limited | Full library |

### From Community Verification vs Internal

| Aspect | Internal MD5 | AccurateRip |
|--------|-------------|-------------|
| Detects corruption | Yes | Yes |
| Community verification | No | Yes |
| Works offline | Yes | No |
| Format limitation | Any | Lossless only |

## Sources

1. [PerfectTUNES](https://www.dbpoweramp.com/perfecttunes)
2. [PerfectTUNES AccurateRip Help](https://www.dbpoweramp.com/Help/perfecttunes/accuraterip)
3. [Verify Ripped Album - dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/27680-how-do-i-verify-a-ripped-album-s-step-by-step-guide)
4. [PerfectTUNES De-Dup](https://www.dbpoweramp.com/help/PerfectTUNESOSX/dedup)
5. [Test Conversion - dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/36837-test-conversion-to-check-integrity-of-flac-files-where-is-the-log-file)
6. [FLAC Verification - dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/36806-flac-audio-file-passed-verification-what-about-when-it-doesn-t-pass)
7. [Conversion Errors - dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/36701-conversion-errors)
8. [AccurateRip - Hydrogenaudio Knowledgebase](https://wiki.hydrogenaudio.org/index.php?title=AccurateRip)
