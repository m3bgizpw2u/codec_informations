# AccurateRip Verification Pipeline
*Generated: 2026-06-04 | Sources: 9 | Confidence: High*

## Executive Summary

AccurateRip is an online database that verifies audio CD rips by comparing checksums against submissions from other users worldwide. The verification algorithm uses a linear summation CRC variant (not a true CRC) over specific sector ranges, with the first and last 5 sectors of audio data excluded from calculation. DBpoweramp integrates AccurateRip checking directly into its secure ripping workflow, using confidence numbers to indicate verification strength. A single match (confidence ≥1) is sufficient for reliable verification when the user hasn't previously submitted rips for that disc.

## 1. AccurateRip Overview

### 1.1 What is AccurateRip?
AccurateRip is an online database and verification system created by the illustrata consortium. It allows CD ripping software to:
- Verify rips against a database of known-correct checksums
- Detect ripped CDs that may have errors or manufacturing defects
- Provide confidence metrics based on community submissions

### 1.2 How AccurateRip Works
1. User rips a CD
2. Software calculates checksums for each track
3. Checksums are submitted to AccurateRip database
4. Other users who ripped the same CD submit their checksums
5. Comparison determines if the rip is "Accurate" or "Inaccurate"

### 1.3 Key Limitation: No Public API
AccurateRip does not provide a public API for general queries. Software must implement the proprietary query protocol. Submission is done through participating software like dBpoweramp ([dBpoweramp Forum](https://forum.dbpoweramp.com/forum/other-topics/general/36120-re-submitting-accuraterip-data-help)).

## 2. AccurateRip Checksum Algorithm

### 2.1 Algorithm Overview
Despite being called "CRC," AccurateRip checksums are **linear summations**, not cyclic redundancy checks. The algorithm:

1. Iterate through each 32-bit DWORD of audio data
2. Multiply each DWORD by its sequential position index
3. Sum all results (modular or extended arithmetic)
4. Skip first 5 sectors of first track
5. Skip last 5 sectors of last track

### 2.2 V1 Algorithm (Original)
```
AR_CRC = 0
MulBy = 1
for each DWORD in track_data:
    if (track_number == 1):
        skip_first_5_sectors
    if (track_number == last_track):
        skip_last_5_sectors
    AR_CRC += DWORD * MulBy
    MulBy++
```

### 2.3 V2 Algorithm (Improved)
V2 improves accuracy using 64-bit intermediate arithmetic to avoid overflow issues:
```
AC_CRCNEW = 0
MulBy = 1
for each DWORD in track_data:
    if in_valid_range:
        CalcCRCNEW = DWORD * MulBy  (64-bit)
        AC_CRCNEW += LOW_32_BITS(CalcCRCNEW)
        AC_CRCNEW += HIGH_32_BITS(CalcCRCNEW)
    MulBy++
```

### 2.4 Sector Range Calculation
- Each sector = 2352 bytes
- 5 sectors = 2940 samples (first track)
- 5 sectors = 2940 samples (last track)
- Maximum drive offset in database (as of 2011): 1776 samples

### 2.5 C Code Reference
```c
#define SectorBytes 2352
DWORD *pAudioData = /* track audio data */;
int DataSize = /* size of data */;
int TrackNumber = /* 1-based track number */;
int AudioTrackCount = /* total tracks on CD */;

DWORD AR_CRC = 0, MulBy = 1;
DWORD AR_CRCPosCheckFrom = 0;
DWORD AR_CRCPosCheckTo = DataSize / sizeof(DWORD);

if (TrackNumber == 1)
    AR_CRCPosCheckFrom += ((SectorBytes * 5) / sizeof(DWORD));
if (TrackNumber == AudioTrackCount)
    AR_CRCPosCheckTo -= ((SectorBytes * 5) / sizeof(DWORD));

for (int i = 0; i < DataSize / sizeof(DWORD); i++) {
    if (MulBy >= AR_CRCPosCheckFrom && MulBy <= AR_CRCPosCheckTo)
        AR_CRC += AR_CRCPosMulti * pAudioData[i];
    MulBy++;
}
```
([AccurateRip CRC Calculation - dBpoweramp Forum](https://forum.dbpoweramp.com/forum/other-topics/developers-corner/20117-accuraterip-crc-calculation))

## 3. AccurateRip Disc ID Computation

### 3.1 Disc ID from TOC
The AccurateRip disc ID is computed from the Table of Contents:
- Track offsets (where each track starts in sectors)
- Total number of tracks
- Lead-out position

### 3.2 Offset Finding CRC
A special offset-finding CRC is calculated 450 frames into the disc:
- Uses single frame data (2352 bytes)
- Calculated with drive-offset adjustment
- Used to find matching pressings at different offsets

### 3.3 Offset Handling
- Drive offset: Hardware-specific reading offset
- Pressing offset: Manufacturing variation in disc position
- Maximum checkable offset: ±5 frames (11760 samples)
- Offset beyond range cannot be cross-verified

## 4. Database Query and Submission

### 4.1 Query Process
1. Compute disc ID from TOC
2. Send disc ID to AccurateRip server
3. Receive list of pressings and their checksums
4. Compare local checksums against all pressings
5. Report matches/no matches

### 4.2 Submission Process
1. Complete ripping with checksums
2. Accumulate results locally
3. Submit via dBpoweramp: CD Ripper Options → AccurateRip → Send Results
4. Reminder every 30 days
5. Results take ~30 days to appear in database

### 4.3 Submission Details
- Multiple rips can be batched and submitted together
- Submissions include all tracks, offsets, and checksums
- Confidence numbers reflect other users' matching submissions

## 5. Confidence and What It Means

### 5.1 Confidence Number Definition
The confidence number represents **how many other users have submitted the exact same bit-perfect rip for that specific pressing of the CD**. Higher numbers indicate stronger verification.

### 5.2 Confidence Levels
| Confidence | Interpretation |
|------------|----------------|
| 0 | No match found (new CD, damaged, or wrong pressing) |
| 1 | Matches 1 other user's submission |
| 2+ | Matches multiple submissions (strong verification) |
| 10+ | Very common disc with many submissions |

### 5.3 Per-Track Confidence
Confidence can vary per track:
- Six tracks may have confidence 41
- Two others may have confidence 40
- One may have confidence 39

This is normal due to:
- Users who skip or don't finish rips
- Different pressings matching different subsets
- Technical variations in submissions

### 5.4 What "Confidence of 1" Means
- Sufficient for verification (for non-submitters)
- Users who have submitted may want confidence ≥2
- Single match still means rip is verified as bit-exact

## 6. Multiple Pressings and No Match Scenarios

### 6.1 Multiple Pressings
Same album can have multiple pressings with:
- Different manufacturing plants
- Different release dates
- Different mastering versions
- Identical TOC but different audio data

### 6.2 No Match Scenarios
When AccurateRip returns no match:

1. **CD not in database**: New release, rare disc, or never submitted
2. **Different pressing**: Disc has offset >5 frames from database entry
3. **Damaged disc**: Errors prevent accurate checksum calculation
4. **Extraction errors**: User's rip contains errors

### 6.3 Disc Identification
Pressings differ in audio stream offset (left or right):
- TOC identification is identical
- Audio comparison only possible after rip
- DBpoweramp doesn't check beyond ±5 frames

### 6.4 Different Masterings
Sometimes different masterings share the same TOC:
- Identical disc structure
- Different audio content
- Requires manual disambiguation

## 7. Secure vs Burst Mode Accuracy

### 7.1 Burst Mode Accuracy
In burst mode:
- Single read, no re-reading
- Checksum calculated from one pass
- If matches AccurateRip: verified
- If doesn't match: unknown (could be rip error or pressing difference)

### 7.2 Secure Mode Accuracy
In secure mode:
- Multiple reads with comparison
- Re-reads problematic sectors
- C2 pointer verification (if supported)
- Higher confidence in successful verification

### 7.3 Accuracy Comparison
| Mode | Speed | Reliability | Error Recovery |
|------|-------|-------------|----------------|
| Burst | Fastest | Depends on disc | None |
| Secure | Medium | High | Partial |
| Ultra Secure | Slowest | Highest | Complete |

### 7.4 DBpoweramp's Approach
DBpoweramp's secure mode effectively combines burst and secure:
1. First pass is burst-speed
2. If AccurateRip matches: stop immediately
3. If no match: proceed to secure re-reads
4. Automated optimization without user intervention

## 8. Offset Handling

### 8.1 Drive Offset
Drives read audio data with a hardware-specific offset:
- Some read earlier, some later
- Offset can be positive or negative
- Modern drives typically have offsets <1000 samples

### 8.2 Offset Correction
- dBpoweramp stores drive offset value
- Reads are adjusted by adding/subtracting samples
- Adjusted data used for checksum calculation
- Ensures consistency with database entries

### 8.3 Pressing Offset
- Manufacturing variation in disc production
- Can differ by ±5 frames maximum for verification
- Beyond ±5 frames: cannot be verified against any pressing

## 9. Edge Cases

### 9.1 CDs with Damaged Lead-in/Lead-out
- May cause incorrect TOC reading
- Leads to incorrect disc ID
- No database match possible

### 9.2 Re-recordable CDs (CD-RW)
- Often not in AccurateRip database
- May have different reflectivity affecting offset
- Verification not available

### 9.3 DualDisc Hybrids
- HD-DVD or DVD side may have different audio
- CD side typically standard
- Requires correct side selection

### 9.4 SACD Hybrids
- DSD layer not readable as CD-DA
- CD layer ripable but may have different offsets
- High-resolution layer requires different extraction

### 9.5 BD-A (Blu-ray Audio)
- Not CD-DA format
- Not eligible for AccurateRip
- Different ripping approach required

### 9.6 Partial Rips
- Only some tracks ripped
- Cannot fully verify disc ID
- Partial AccurateRip check possible

### 9.7 Self-Submitted Rips
- If user has submitted rip results before
- May match own submission
- Recommendation: confidence ≥2 for self-verified

## 10. Would a User Notice a Difference?

### From DBpoweramp to Manual Verification

**No practical difference** in the ripped audio:
- Both produce identical audio files
- AccurateRip is a verification tool, not a ripper
- Any difference indicates an error in one method

### From DBpoweramp to Other Rippers

| Aspect | DBpoweramp | Other Rippers |
|--------|-----------|---------------|
| **Integration** | Seamless in workflow | May require separate tool |
| **Auto-Optimize** | Combines burst/secure | Manual selection |
| **Confidence Display** | Real-time per track | Varies |
| **Submission** | Built-in | May require manual |

### Confidence Level Impact
- **Confidence ≥1**: Safe to assume accurate
- **No match**: Manual verification or re-rip needed
- **Low confidence on rare disc**: May be normal, not error

## Sources

1. [AccurateRip - Hydrogenaudio Knowledgebase](https://wiki.hydrogenaudio.org/index.php?title=AccurateRip)
2. [AccurateRip CRC Calculation - dBpoweramp Forum](https://forum.dbpoweramp.com/forum/other-topics/developers-corner/20117-accuraterip-crc-calculation)
3. [AccurateRip Checksum - leo-bogert/accuraterip-checksum](https://github.com/leo-bogert/accuraterip-checksum)
4. [libarcstk - AccurateRip Toolkit](https://github.com/crf8472/libarcstk)
5. [Submitting AccurateRip Data - dBpoweramp Forum](https://forum.dbpoweramp.com/forum/other-topics/general/36120-re-submitting-accuraterip-data-help)
6. [Query AccurateRip Database - dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/25269-query-accuraterip-database)
7. [Not in AccurateRip - dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/31701-what-does-not-in-accuraterip-mean)
8. [Secure Rip with No Confidence - dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/28022-how-can-i-get-a-secure-rip-of-a-cd-in-the-ar-database-with-no-confidence)
9. [Generating AccurateRip Checksums - Hydrogenaudio](https://hydrogenaudio.org/index.php/topic,97603.0.html)
