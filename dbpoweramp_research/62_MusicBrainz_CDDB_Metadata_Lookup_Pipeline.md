# MusicBrainz CDDB Metadata Lookup Pipeline
*Generated: 2026-06-04 | Sources: 8 | Confidence: High*

## Executive Summary

DBpoweramp implements a multi-tier metadata lookup system that queries multiple providers (MusicBrainz, Discogs, GD3, freedb) to identify audio CDs and retrieve album/track metadata. The lookup sequence begins with AccurateRip disc ID, progresses through MusicBrainz, then freedb/CDDB, with Discogs bridging via an internal database. PerfectMeta cross-checks results from all enabled providers to correct errors. The MusicBrainz disc ID is computed using SHA-1 hashing of TOC data followed by Base64 encoding, while freedb uses a different 8-digit hex algorithm.

## 1. Metadata Lookup Sequence

### 1.1 Default Lookup Order
DBpoweramp queries metadata providers in this sequence:

1. **AccurateRip Disc ID**: Quick identification via existing rip database
2. **MusicBrainz**: Primary open-source metadata source
3. **freedb/CDDB**: Legacy database (discontinued March 2020)
4. **Discogs**: Commercial provider with extensive catalog
5. **GD3**: Commercial Japanese metadata provider
6. **Manual Entry**: User-defined or copied URLs

### 1.2 PerfectMeta / Intelligent Meta
When PerfectMeta is enabled:
- All enabled providers are queried simultaneously
- Results are cross-checked for errors
- Discrepancies are flagged for review
- Best elements from each source can be selected

### 1.3 Provider Selection
In CD Ripper Options → Active Providers:
- Multiple providers can be enabled simultaneously
- More providers = better PerfectMeta results
- freedb and Discogs: free
- GD3: commercial, per-disc lookup fees apply

### 1.4 Forcing Single Provider
To use only MusicBrainz:
1. Uncheck all other providers in Active Providers
2. If Batch Ripper is used, check its own metadata menu
3. If MusicBrainz fails to retrieve, the lookup fails (no fallback)

## 2. MusicBrainz Disc ID Calculation

### 2.1 TOC Data Required
The MusicBrainz disc ID requires:
- First track number (1 byte)
- Last track number (1 byte)
- Lead-out track offset (4 bytes)
- Frame offsets for up to 99 tracks (4 bytes each)

### 2.2 SHA-1 Hashing
The TOC data is converted to uppercase hex ASCII and hashed:
```
First Track:     sprintf(temp, "%02X", pCDInfo->FirstTrack);
Last Track:      sprintf(temp, "%02X", pCDInfo->LastTrack);
Lead-out Offset: sprintf(temp, "%08X", pCDInfo->FrameOffset[0]);
Track Offsets:   sprintf(temp, "%08X", pCDInfo->FrameOffset[i]);
```

### 2.3 Base64 Encoding
The 20-byte SHA-1 hash is Base64-encoded to create a 28-character disc ID.

### 2.4 Example Disc ID
Disc with 6 audio tracks: `49HHV7Eb8UKF3aQiNmu1GR8vKTY-`

### 2.5 Comparison with freedb Disc ID
| Aspect | MusicBrainz | freedb/CDDB |
|--------|-------------|-------------|
| **Algorithm** | SHA-1 → Base64 | Hex sum |
| **Length** | 28 characters | 8 hex digits |
| **Lead-out** | Included | Not included |
| **Conversion** | MB → freedb possible | MB → freedb not reliable |

## 3. MusicBrainz API Interaction

### 3.1 API Query
DBpoweramp queries the MusicBrainz web service:
- Endpoint: `musicbrainz.org/ws/2/disc/`
- Input: Disc ID
- Output: Release metadata including tracks, artists, dates

### 3.2 Submission URL
For releases not in MusicBrainz:
- Submission URL generated from disc ID
- Allows user to add release via web browser
- Supports Picard integration for batch submission

### 3.3 libdiscid Library
Many applications use libdiscid for cross-platform disc ID:
- Reads CD TOC via platform-specific methods
- Generates both MusicBrainz and freedb disc IDs
- Available for Windows, macOS, Linux

## 4. Handling Multiple Releases

### 4.1 When Multiple Releases Match
When MusicBrainz returns multiple releases for one disc ID:
1. All matching releases displayed
2. User can select appropriate one
3. Or: highest-ranked release auto-selected

### 4.2 Release Selection Logic
DBpoweramp auto-selects the release with highest ranking:
- Based on number of votes/confirmations
- User can override manually
- Box set identification possible

### 4.3 Batch Ripper Behavior
In Batch Ripper with multiple matches:
- The highest-ranking release is automatically used
- No manual intervention by default
- User can configure for manual review

### 4.4 Multi-Disc Sets
For box sets and multi-disc releases:
- Each disc has unique TOC
- Each disc has unique disc ID
- Disc number must be tracked separately

## 5. User Selection Interface

### 5.1 PerfectMeta Review Page
Accessible via:
- ALT+M keyboard shortcut
- Clicking Metadata Icon on toolbar
- Appears when metadata needs review

### 5.2 Selection Options
From the PerfectMeta review page:
- **Single provider**: Click provider button to use all data from that source
- **Single item**: Click individual track/album field to use specific data
- **Album art**: Click to add/replace artwork

### 5.3 Provider-Specific Data
| Provider | Strengths |
|----------|-----------|
| MusicBrainz | Open source, accurate, comprehensive |
| Discogs | Extensive catalog, cover art |
| GD3 | Japanese music specialization |
| freedb | Legacy support, basic metadata |

## 6. Field Mapping and Sources

### 6.1 Standard Fields from Each Source
| Field | MusicBrainz | Discogs | freedb |
|-------|-------------|---------|--------|
| Artist | ✓ | ✓ | ✓ |
| Album Title | ✓ | ✓ | ✓ |
| Track Titles | ✓ | ✓ | ✓ |
| Release Date | ✓ | ✓ | Partial |
| Genre | ✓ | ✓ | ✓ |
| Track Numbers | ✓ | ✓ | ✓ |
| Album Artist | ✓ | ✓ | Limited |
| Barcode | ✓ | ✓ | - |
| Catalog Number | ✓ | ✓ | - |
| ISRC | ✓ | - | - |

### 6.2 DBpoweramp Tag Mapping
DBpoweramp maps metadata to standard tags:
- `[album artist]`: Album-level artist
- `[artist]`: Track-level artist
- `[composer]`: If available from source
- `[year]`: Release year (may differ from original)
- `[albumyear]`: Original release year (not always available)

### 6.3 Album Year Limitation
From dBpoweramp forum response:
> "I think we do not read this tag from musicbrainz."

The `[albumyear]` tag (original release year) is not currently supported by dBpoweramp, even though MusicBrainz provides it.

## 7. Releases with Multiple Disc IDs

### 7.1 Why Multiple Disc IDs Exist
Same physical disc can generate different IDs due to:
- Different pressings (manufacturing variations)
- CD-ROM vs CD-DA versions
- Enhanced CDs with bonus content
- Region variations

### 7.2 Handling in DBpoweramp
1. Each TOC generates unique disc ID
2. Each ID can map to different MusicBrainz release
3. User must select correct release manually
4. Auto-selection uses highest-confidence match

### 7.3 Box Set Identification
When ripping box sets:
- Each disc in box has unique TOC
- Each disc gets individual disc ID
- Batch Ripper can detect and group
- Disc Type option for verification

## 8. Discogs TOC Linking

### 8.1 Discogs Limitation
Discogs does not natively link CD TOCs to releases:
- No programmatic TOC → release lookup
- Requires manual search or internal database

### 8.2 DBpoweramp's Internal Database
DBpoweramp maintains internal database for Discogs:
- Maps known TOCs to Discogs releases
- Allows Discogs lookup without TOC
- Updated with new submissions

### 8.3 Manual Discogs Lookup
Recent versions allow manual Discogs URL input:
- Copy Discogs release URL
- Paste into dBpoweramp
- Uses release data from URL

## 9. freedb/CDDB Status

### 9.1 Service Discontinuation
freedb officially shut down March 31, 2020:
- No new submissions accepted
- Existing data still accessible
- DBpoweramp maintains local cached data

### 9.2 freedb Disc ID Algorithm
The freedb disc ID is computed differently:
```
n = sum of (cddb_sum(seconds/60) + seconds%60) for each track
disc_id = ((n % 0xFF) << 24) | (total_seconds << 8) | num_tracks
```

### 9.3 Impact on DBpoweramp
- freedb lookups may fail for new discs
- Old freedb data still usable
- MusicBrainz is primary fallback

## 10. Edge Cases

### 10.1 Self-Recorded CDs
- No metadata in any database
- CD-TEXT only option
- Manual metadata entry required

### 10.2 Pirated/Ripped CDs
- May have incorrect metadata
- CD-TEXT may be wrong or missing
- Manual verification needed

### 10.3 Damaged CD-TEXT
- Encoding issues possible
- Japanese/Cyrillic/Chinese characters
- Fallback to online lookup

### 10.4 Compilation Albums
- Multiple artists on one disc
- Various Artists grouping
- Track-level artist handling

### 10.5 Classical Music
- Conductor/ensemble naming
- Work/movement structure
- Multiple catalog numbers

### 10.6 Jazz and World Music
- Various spellings of names
- Romanization differences
- Multiple instrument credits

### 10.7 Hidden Tracks / Enhanced CDs
- Extra content not in metadata
- Data tracks mixed with audio
- Pre-gap audio (HTOA)

## 11. Would a User Notice a Difference?

### From DBpoweramp to Manual Tagging

**Significant time savings**:
- Automatic lookup vs. manual entry
- PerfectMeta cross-validation
- Batch processing capability

**Data quality**:
- Expert-curated metadata
- Consistent formatting
- Cross-reference validation

### From DBpoweramp to Other Rippers

| Aspect | DBpoweramp | Other Rippers |
|--------|-----------|---------------|
| **Providers** | Multiple integrated | Usually single |
| **PerfectMeta** | Cross-validation | Single source |
| **Discogs** | Native support | May not support |
| **Auto-select** | Highest ranking | First match |

### Confidence Difference
- **High confidence**: Multiple providers agree
- **Low confidence**: Manual review recommended
- **No match**: User must enter manually

## Sources

1. [MusicBrainz Disc ID Calculation](https://musicbrainz.org/doc/Disc_ID_Calculation)
2. [libdiscid - MusicBrainz](https://musicbrainz.org/doc/libdiscid/)
3. [FreeDB - MusicBrainz](https://musicbrainz.org/doc/FreeDB)
4. [CDDB/FREEDB DISCID Algorithm](https://courses.cs.duke.edu/fall04/cps006g/class/isis/freedb.pdf)
5. [dBpoweramp CD Ripper Setup Guide](https://doc.dbpoweramp.com/dmc/en/cd-ripper-setup-guide.htm)
6. [dBpoweramp CD Ripper: Advanced Options](https://dbpoweramp.com/Help/dMC/CDadvanced)
7. [Metadata: Discogs, freedb - dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/39215-metadata-discogs-freedb)
8. [Forcing only MusicBrainz - dBpoweramp Forum](https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/39482-forcing-only-musicbrainz-and-other-metadata-questions)
