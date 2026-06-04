# Error Handling, Recovery, and Logging
*Generated: 2026-06-04 | Sources: 8 | Confidence: High*

## Executive Summary

DBpoweramp implements comprehensive error handling throughout the conversion and ripping pipeline, with different strategies for source file errors, encoding errors, and tag writing failures. The software provides detailed logging for debugging, supports batch mode error recovery (skip vs abort), and includes command-line interface options for automated error capture. Source file errors during Test Conversion trigger corruption detection via MD5 mismatch, while tag writing failures allow partial file creation with retry mechanisms. The Secure Extraction Log captures sector-level error details for CD ripping.

## 1. Error Categories

### 1.1 Source File Errors
Errors reading the source audio file:
- **File not found**: Source file doesn't exist
- **Permission denied**: File locked or access restricted
- **File corrupted**: Audio data damaged
- **Format not supported**: Codec missing or incompatible
- **File in use**: Another program has file locked

### 1.2 Encoding Errors
Errors during the conversion process:
- **Codec not found**: Required encoder not installed
- **Invalid settings**: Encoding parameters invalid
- **Disk full**: Insufficient space for output
- **Memory allocation**: System resources exhausted
- **Interruption**: User or system canceled

### 1.3 Tag Writing Errors
Errors writing metadata to output file:
- **Permission denied**: Output file locked
- **Format limitation**: Format doesn't support certain tags
- **Encoding error**: Non-ASCII characters in tags
- **File moved**: File moved/deleted during tagging
- **Read-only**: Output file is read-only

### 1.4 Drive/CD Errors
Errors during CD ripping:
- **No disc**: CD tray empty or not recognized
- **Disc unreadable**: Physical damage or dirty disc
- **Drive error**: Hardware failure
- **Timeout**: Drive response timeout

## 2. Error Codes and Meanings

### 2.1 Error Prefix System
DBpoweramp uses prefix-based error categorization:
| Prefix | Category | Example |
|--------|----------|---------|
| `Error opening file` | Source read | Permission, corruption |
| `Error converting to` | Encoding | Codec, settings |
| `Unable to write tags` | Tag write | Lock, format |
| `md5 did not match` | Corruption | Source damaged |

### 2.2 Specific Error Messages
| Message | Meaning | Resolution |
|---------|---------|------------|
| `check no other program has it open` | File locked | Close other program |
| `file is corrupt` | Source damaged | Re-rip or re-download |
| `Codec not installed` | Missing encoder | Install codec |
| `Unable to write tags` | Tag write failed | Retry or skip |

### 2.3 Error Codes
| Code | Description |
|------|-------------|
| `E_OPEN` | Cannot open source file |
| `E_DECODE` | Cannot decode source |
| `E_ENCODE` | Cannot encode output |
| `E_TAG` | Cannot write tags |
| `E_WRITE` | Cannot write file |
| `E_DISK_FULL` | Insufficient disk space |

## 3. Log File Format and Location

### 3.1 Log File Types
| Log Type | Location | Contents |
|----------|----------|----------|
| **Conversion Log** | Shown in UI or saved | Encoding progress, errors |
| **Secure Extraction Log** | Music Folder | CD ripping details |
| **Debug Log** | Configurable | Full debugging info |
| **Error File** | User-specified | Only errors from CLI |

### 3.2 Secure Extraction Log (CD Ripping)
Saved to Music Folder as text file:
```
========================================
dBpoweramp Secure Extraction Log
========================================
Album: Artist - Album Title
Disc ID: XXHHV7Eb8UKF3aQiNmu1GR8vKTY-
Date: 2024-01-15 14:32:00
----------------------------------------
Track 01: Accurate (12) - No errors
Track 02: Accurate (15) - No errors
Track 03: Inaccurate (0) - Errors detected
  - Sector 12543: 3 re-reads required
  - Sector 12544: 7 re-reads required
Track 04: Accurate (14) - No errors
----------------------------------------
```

### 3.3 Debug Logging
In Configuration → Music Converter Debug:
- Enable full debug logging
- Log file created on crash/error
- Contains all operation details
- Useful for support requests

### 3.4 Log File Format
Typical log entries:
```
[Timestamp] [Level] [Component] Message
2024-01-15 14:32:01 INFO  Decoder     Opening: track01.flac
2024-01-15 14:32:02 INFO  DSP         Processing: ReplayGain
2024-01-15 14:32:03 INFO  Encoder     Encoding: FLAC
2024-01-15 14:32:15 INFO  Writer      Tags written successfully
```

## 4. Recovery from Interrupted Batch Conversion

### 4.1 Interruption Causes
Batch conversions can be interrupted by:
- User cancellation
- System shutdown/sleep
- Power failure
- Application crash
- Disk full error

### 4.2 Resume Capabilities
DBpoweramp does not support true resume:
- Completed files are not re-processed
- Interrupted files must be re-converted
- Overwrite detection handles existing files

### 4.3 Overwrite on Resume
When resuming a batch:
- Completed files detected as existing
- Options: Skip, Overwrite, Rename
- Default: Prompt for each file
- Recommendation: Set "Skip if exists"

### 4.4 Recovery Best Practices
1. Use "Skip if exists" for interrupted batches
2. Check log for completed files
3. Verify file integrity after recovery
4. Consider using Test Conversion first

## 5. Disk Full Handling

### 5.1 Detection
DBpoweramp detects disk full conditions:
- Before writing large files
- During encoding
- When writing tags

### 5.2 Behavior on Disk Full
When disk becomes full:
- Current conversion fails
- Error logged with specific message
- Batch continues with next file
- User notified of failure

### 5.3 Prevention
Preventive measures:
- Check available space before batch
- Set minimum free space threshold
- Use reliable storage
- Monitor during large batches

### 5.4 Recovery
After freeing disk space:
- Re-run batch with "Skip if exists"
- Completed files preserved
- Only failed files re-converted

## 6. Codec Not Found Handling

### 6.1 Missing Codec Detection
When codec is not installed:
- Error message indicates codec name
- Available codecs listed
- Download link provided (if applicable)

### 6.2 Codec Installation
For missing codecs:
1. Download from dBpoweramp website
2. Install while application closed
3. Restart application
4. Retry conversion

### 6.3 Format Fallback
If specific format unavailable:
- Lossy → Different lossy codec
- Lossless → Different lossless codec
- May require manual selection

## 7. Tag Write Failure Handling

### 7.1 Partial File Creation
When tag write fails:
- Audio file is complete and playable
- Tags may be missing or incomplete
- File not corrupted

### 7.2 Retry Mechanism
DBpoweramp retries tag writing:
- Up to 50 retries over ~1 second
- Handles temporary file locks
- Most transient failures recover

### 7.3 Persistent Failure
When retries exhausted:
- Error logged
- File remains playable
- Manual tag addition possible

### 7.4 Error Messages
| Message | Cause | Solution |
|---------|-------|----------|
| `Unable to write tags` | File locked | Close other program |
| `Tag write failed` | Format doesn't support | Use different format |
| `Unicode error` | Invalid characters | Remove special chars |

## 8. Skip vs Abort in Batch Mode

### 8.1 Skip Single File
When error occurs on one file:
- Log error message
- Continue to next file
- Batch completes (partial)

### 8.2 Abort Batch
User-initiated abort:
- Current file completes (if safe)
- No new files started
- Already completed files preserved

### 8.3 Automatic Abort Triggers
CD Ripping Secure Rip Abort options:
- Maximum time per track exceeded
- Maximum time per disc exceeded
- Consecutive error threshold reached

### 8.4 Error Reporting Granularity
Errors report:
- Exact filename and path
- Specific step that failed (open, decode, encode, write, tag)
- Technical details for debugging
- Recovery suggestions when available

## 9. Error Recovery Best Practices

### 9.1 Before Batch
1. Verify source file integrity (Test Conversion)
2. Ensure adequate disk space
3. Check codec availability
4. Close other programs that may lock files

### 9.2 During Batch
1. Monitor for errors
2. Address disk space issues immediately
3. Note any recurring error patterns

### 9.3 After Batch
1. Review error log
2. Identify failed files
3. Re-run failed files
4. Verify output integrity

### 9.4 Validation Workflow
```
1. Test Conversion (scan for corruption)
2. Batch Convert (with logging)
3. Review Log (check for errors)
4. PerfectTUNES (verify AccurateRip)
5. Fix Errors (re-rip/re-convert)
```

## 10. Edge Cases

### 10.1 Corrupted Source with Valid Header
Source file opens but contains errors:
- Decoding fails partway
- Partial MD5 comparison
- Error reported
- Must re-source

### 10.2 Network Interruption
Network drive becomes unavailable:
- Immediate failure
- Retry when reconnected
- Files may be partially written

### 10.3 Unicode Filename Issues
Filenames with unusual Unicode:
- May not display correctly
- Conversion still works
- Tags may need adjustment

### 10.4 Very Large Files
Files exceeding memory:
- Streaming processing
- Slower but completes
- Disk space more critical

### 10.5 Mixed Format Batch
Different source formats in batch:
- Each decoded appropriately
- Individual error handling
- Continue with others

### 10.6 Long-Running Batch
Batch running for hours:
- System may sleep/hibernate
- Power management disabled recommended
- Periodic saves/checkpoints

## 11. Would a User Notice a Difference?

### From DBpoweramp vs Other Tools

| Aspect | DBpoweramp | Other Tools |
|--------|-----------|------------|
| **Error detail** | Sector-level for CD | Basic |
| **Logging** | Comprehensive | Limited |
| **Recovery** | Partial (skip) | May abort all |
| **MD5 verification** | Built-in | May need external |

### From Proper vs No Error Handling

| Scenario | With Error Handling | Without |
|----------|-------------------|---------|
| Corrupt source | Error reported, batch continues | Silent corruption |
| Disk full | Partial batch, recoverable | Data loss |
| Tag failure | Partial tags, retry possible | Silent failure |
| Power loss | Skip mode protects done work | Risk of corruption |

## Sources

1. [dBpoweramp Configuration](https://dbpoweramp.com/Help/dMC/dMCConfig)
2. [dBpoweramp CD Ripper Setup Guide](https://doc.dbpoweramp.com/dmc/en/cd-ripper-setup-guide.htm)
3. [Test Conversion - dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/36837-test-conversion-to-check-integrity-of-flac-files-where-is-the-log-file)
4. [Conversion Errors - dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/36701-conversion-errors)
5. [FLAC Verification - dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/36806-flac-audio-file-passed-verification-what-about-when-it-doesn-t-pass)
6. [dBpoweramp Music Converter Help](https://dbpoweramp.com/Help/dMC/dMC)
7. [dBpoweramp DSP Help](https://dbpoweramp.com/Help/dMC/dsp)
8. [CoreConverter Documentation - dBpoweramp](https://www.dbpoweramp.com/Help/dMC/coreconverter)
