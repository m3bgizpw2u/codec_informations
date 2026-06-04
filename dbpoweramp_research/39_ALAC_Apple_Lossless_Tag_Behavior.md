# DBpoweramp Music Converter — ALAC Apple Lossless Tag Behavior
*Generated: 2026-06-04 | Format: ALAC, M4A | Confidence: High*

---

## 1. Executive Summary

ALAC (Apple Lossless Audio Codec) files are stored inside the MPEG-4 container (.m4a extension), meaning they use the **exact same iTunes-style MP4 atom metadata system** as AAC files. There is no ALAC-specific tag format — the fourcc `'alac'` identifies the audio codec, while all metadata lives in the standard `moov.udta.meta.ilst` atom hierarchy. DBpoweramp's ALAC tag handling is therefore identical to its AAC/MP4 handling: `©nam`, `©ART`, `©alb`, `trkn`, `covr`, freeform `----:com.apple.iTunes:*` atoms, and `iTunSMPB` for gapless playback. ALAC-specific behaviors include the magic cookie in the `alac` atom, potential 24-bit sample size metadata quirks, and Apple's channel layout atoms.

---

## 2. Format Overview

ALAC in M4A is structurally identical to AAC in M4A:

```
moov (movie container)
  └─ trak (track container)
      └─ mdia (media container)
          └─ minf (media information)
              └─ stbl (sample table)
                  └─ stsd (sample description)
                      └─ alac (ALAC audio specific config)
  └─ udta (user data)
      └─ meta (metadata container, ilst follows)
          └─ hdlr (handler: 'mdta')
          └─ ilst (iTunes metadata list)
              ├─ ©nam (title)
              ├─ ©ART (artist)
              ├─ ©alb (album)
              ├─ trkn (track number, binary)
              ├─ covr (cover art, binary)
              └─ ... (all standard iTunes atoms)
```

The `alac` atom within `stsd` stores:
1. An ISO full box header (size + 'alac' type + version + flags)
2. The ALAC-specific configuration data (magic cookie)
3. The decoder initialization parameters

**Source:** [macosforge/alac — ALACMagicCookieDescription.txt](https://github.com/macosforge/alac/blob/master/ALACMagicCookieDescription.txt), [KaijuConverter — M4A Format Guide](https://kaijuconverter.com/guides/m4a-mpeg4-audio-format)

---

## 3. Tag Reading Behavior

### iTunes Metadata Atoms

ALAC files use the complete iTunes metadata atom set (identical to AAC). See document **32_AAC_MP4_iTunes_Atom_Tag_Behavior.md** for the full atom reference table.

### ALAC-Specific Metadata

#### alac Atom (Codec Configuration)

The `alac` atom contains the ALAC magic cookie with decoder configuration:

| Field | Description |
|-------|-------------|
| `sampleRate` | Audio sample rate (e.g., 44100, 48000, 96000) |
| `bitDepth` | Source PCM bit depth (e.g., 16, 24) |
| `numChannels` | Number of audio channels |
| `magicCookie` | Decoder initialization data (24 or 48 bytes) |

**Source:** [macosforge/alac — ALACMagicCookieDescription.txt](https://github.com/macosforge/alac/blob/master/ALACMagicCookieDescription.txt)

#### Channel Layout Atom

ALAC files with multi-channel audio (5.1, 7.1) may contain a `chan` atom within `stsd` specifying the speaker layout per ISO/IEC 14496-12:

| Channel Count | Layout |
|--------------|--------|
| 1 | Mono (center front) |
| 2 | Stereo (left, right front) |
| 6 | 5.1 surround |
| 8 | 7.1 surround |

DBpoweramp preserves channel layout atoms during lossless transcoding.

#### Freeform Atoms (----:com.apple.iTunes:*)

ALAC files support the same freeform atoms as AAC:

| Atom | Purpose |
|------|---------|
| `----:com.apple.iTunes:REPLAYGAIN_TRACK_GAIN` | Track loudness normalization |
| `----:com.apple.iTunes:REPLAYGAIN_TRACK_PEAK` | Track peak amplitude |
| `----:com.apple.iTunes:iTunSMPB` | Gapless playback delay/padding |
| `----:com.apple.iTunes:ENCODER` | Encoder identity |

---

## 4. Tag Writing Behavior

### iTunSMPB for Gapless Playback

ALAC (like AAC) stores gapless playback parameters in the `iTunSMPB` freeform atom:

```
iTunSMPB=DDDD PPPP [samples]
- DDDD: Delay samples (leading silence before audio)
- PPPP: Padding samples (trailing silence after audio)
```

**Example**: `iTunSMPB=0000 1A20 0000 1B78`
- Delay: 0x0000 = 0 samples
- Padding: 0x1A20 = 6688 samples

This information is **preserved during lossless ALAC-to-ALAC transcoding** and **stripped during lossy conversion**.

**Source:** [KaijuConverter — M4A Format Guide](https://kaijuconverter.com/guides/m4a-mpeg4-audio-format)

### 24-bit ALAC Metadata Quirk

There is a known issue where some ALAC encoders (notably Max, an older Mac encoder) fail to properly set the bit depth in the ALAC `alac` atom's magic cookie. This causes some players (notably Squeezebox Server/LMS) to report 16-bit instead of 24-bit even when the audio is actually 24-bit.

DBpoweramp, when re-encoding ALAC, correctly sets the bit depth in the `alac` atom. However, simply copying tags from a source with this bug may not fix the reported bit depth.

### alac FourCC Identification

The audio codec is identified by the `alac` fourcc in the `stsd` (Sample Description) box:

```
stsd (Sample Description)
  └─ alac (ALAC Specific Box)
      ├─ Box header (size + 'alac')
      ├─ Version/flags (4 bytes, usually 0)
      ├─ ALAC configuration (36 bytes: 12 header + 24 magic cookie)
      └─ Magic cookie (decoder parameters)
```

Some older encoders may use different magic cookie sizes. DBpoweramp handles both 24-byte and 48-byte magic cookies.

### Cover Art

Cover art in ALAC uses the same `covr` atom as AAC:
- Binary JPEG/PNG/BMP data
- Multiple `data` children allowed (one per image)
- MIME type indicated by `mdia.minf.hdlr` class in each `data` child

---

## 5. Normalization Rules

| Rule | Behavior |
|------|----------|
| **Tag format** | iTunes MP4 atoms in `moov.udta.meta.ilst` |
| **Cover art** | `covr` binary atom (JPEG/PNG) |
| **Gapless** | `iTunSMPB` preserved during lossless, stripped during lossy |
| **ReplayGain** | `----:com.apple.iTunes:REPLAYGAIN_*` freeform atoms |
| **Bit depth** | Set in `alac` atom; must match actual audio |
| **Sample rate** | Set in `alac` atom; preserved |
| **Channel layout** | `chan` atom preserved for multi-channel |
| **Text encoding** | UTF-8 for all text atoms |

---

## 6. Edge Cases

1. **Magic cookie version mismatch**: Different ALAC encoders produce different magic cookie formats (24-byte vs 48-byte). DBpoweramp handles both during decode/encode. However, the 48-byte format (introduced by Apple in iTunes 7) includes additional error correction data that the 24-byte format lacks.

2. **Bit depth in `alac` atom vs actual audio**: If the `alac` atom claims 16-bit but the audio data is 24-bit, some players display the wrong bit depth. Re-encoding with DBpoweramp corrects this metadata issue.

3. **iTunSMPB with zero values**: A track with no gapless delay/padding may have `iTunSMPB=0000 0000 0000 0000`. This is valid and should be preserved for gapless albums.

4. **Multiple `covr` entries with different formats**: A file may have both a JPEG and PNG version of the same cover art as separate `data` children. Some players display the first one; others may display both or prompt for selection.

5. **`----:com.apple.iTunes:ORIGINALDATE`**: Some rippers write `----:com.apple.iTunes:ORIGINALDATE` as a freeform atom for the original release date, distinct from `©day` (recording date). This is non-standard but used by MusicBrainz Picard.

---

## 7. DBpoweramp-Specific Behavior

- **Encoding**: DBpoweramp encodes ALAC using the standard 48-byte magic cookie format compatible with iTunes 7+.
- **Tag writing**: Uses mp4v2/TagLib for MP4 atom I/O. Standard iTunes-compatible atoms.
- **Cover art**: Written as `covr` binary atom.
- **Gapless**: `iTunSMPB` preserved during ALAC→ALAC lossless transcoding.
- **ReplayGain**: Not written natively. External tools or manual tagging required.
- **Bit depth**: Correctly set in `alac` atom based on source audio.
- **Sample rate**: Preserved from source.
- **Sort atoms**: `soal`, `soar`, `sonm`, `sosn` preserved if present.

---

## 8. Verification Checklist

- [ ] `©nam`, `©ART`, `©alb`, `©day` display correctly (UTF-8)
- [ ] `trkn` shows as "3/12" format
- [ ] `covr` shows as `[JPEG, N bytes]` or `[PNG, N bytes]`
- [ ] `alac` atom fourcc identifies ALAC codec
- [ ] `alac` magic cookie size is 24 or 48 bytes
- [ ] Bit depth in `alac` atom matches actual audio (24-bit vs 16-bit)
- [ ] `iTunSMPB` present and values correct for gapless albums
- [ ] `----:com.apple.iTunes:*` freeform atoms preserved if present
- [ ] No `TagLib: MP4: Ignoring duplicate atom` warnings
- [ ] Channel layout (`chan` atom) preserved for multi-channel files

---

## 9. Sources

1. [macosforge/alac — ALACMagicCookieDescription.txt](https://github.com/macosforge/alac/blob/master/ALACMagicCookieDescription.txt)
2. [KaijuConverter — M4A Format Guide: MPEG-4 Audio, AAC vs ALAC](https://kaijuconverter.com/guides/m4a-mpeg4-audio-format)
3. [AudioUtils — FLAC vs. ALAC: Lossless Audio Format Comparison](https://audioutils.com/blog/flac-vs-alac)
4. [Apple Lossless — applelossless.com](http://www.applelossless.com/)
5. [Squeezebox Forum — 24-bit ALAC files being truncated to 16-bit](https://squeezecenter.slimdevices.com/narkive/0sZ5UfYQ/24-bit-alac-files-being-truncated-to-16-bit)
6. [libmp4v2 Wiki — iTunesMetadata](https://github.com/sergiomb2/libmp4v2/wiki/iTunesMetadata)
