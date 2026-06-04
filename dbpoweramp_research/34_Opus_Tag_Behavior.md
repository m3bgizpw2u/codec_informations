# DBpoweramp Music Converter — Opus Tag Behavior
*Generated: 2026-06-04 | Format: OGG Opus | Confidence: High*

---

## 1. Executive Summary

Opus files in OGG encapsulation use Vorbis Comments as their metadata base, making tag reading and writing structurally identical to OGG Vorbis. The critical differentiator is the **R128 gain system**: Opus uses `R128_TRACK_GAIN` and `R128_ALBUM_GAIN` stored as **Q7.8 fixed-point integers** (1/256 dB precision), NOT the ReplayGain float format used by Vorbis/FLAC/MP3. This is because Opus normalizes against the EBU R128 broadcast standard (-23 LUFS for music), which is fundamentally different from ReplayGain's -18 dB reference. DBpoweramp must correctly convert ReplayGain → R128 when tagging Opus output, or write R128 values directly. The OpusTags structure also includes an OpusHead identification header and an OpusTags comment header, both required for valid Opus files.

---

## 2. Format Overview

An OGG Opus file contains two mandatory header packets:

| Packet | Name | Description |
|--------|------|-------------|
| 1 | `OpusHead` | Codec identification, sample rate (48000), channel mapping, pre-skip, input gain |
| 2 | `OpusTags` | Vendor string + Vorbis Comment field list |

The `OpusTags` packet is a standard Vorbis Comment packet (vendor string + field list) plus two Opus-specific fields:

```
OpusTags:
  1. Vendor string length + vendor string (same as Vorbis)
  2. User comment list length
  3. [Vendor string comment fields, optional]
  4. R128_TRACK_GAIN=<integer>
  5. R128_ALBUM_GAIN=<integer>
```

**Source:** [RFC 7845 — Ogg Encapsulation for the Opus Audio Codec](https://datatracker.ietf.org/doc/html/rfc7845), [XiphWiki — OggOpus](https://wiki.xiph.org/OggOpus)

---

## 3. Tag Reading Behavior

### R128_TRACK_GAIN and R128_ALBUM_GAIN Format

Per RFC 7845 Section 5.2:

```
R128_TRACK_GAIN=-573
R128_ALBUM_GAIN=111
```

- **Q7.8 fixed-point integer**: The value is a 16-bit signed integer (range: -32768 to +32767) representing gain in 1/256 dB units.
- To convert to dB: `gain_dB = value / 256.0`
- Example: `-573 / 256 = -2.238 dB`

| R128 Tag Value | dB Value | ReplayGain Equivalent (approx.) |
|----------------|----------|--------------------------------|
| -512 | -2.00 dB | -2.00 dB (same reference) |
| -256 | -1.00 dB | -1.00 dB (same reference) |
| 0 | 0.00 dB | +18 dB relative |
| +512 | +2.00 dB | +20 dB relative |

**Critical**: The R128 reference is **-23 LUFS** (EBU R128), not -18 dB (ReplayGain). Converting between systems requires adding +5 dB to ReplayGain values before scaling.

**Formula: ReplayGain → R128 conversion:**
```
R128_value = round(ReplayGain_dB * 256)
```
Example: ReplayGain -6.20 dB → R128 value = round(-6.20 * 256) = -1587

**Source:** [RFC 7845 — Section 5.2 Tag Definitions](https://datatracker.ietf.org/doc/html/rfc7845#section-5.2)

### R128_REFERENCE_LOUDNESS (Optional)

```
R128_REFERENCE_LOUDNESS=-23 LU
```

Documents the loudness of the source material used for R128 gain calculation. Default is -23 LU per EBU R128. Some taggers write this field.

### OpusHead Output Gain

The `OpusHead` header contains an `output_gain` field (Q7.8, same format as R128 tags). This gain is applied by the decoder **before** any R128 tag processing:

> "If a player chooses to apply any volume adjustment or gain modification, such as the R128_TRACK_GAIN... the adjustment MUST be applied **in addition to** the 'output gain' value [from OpusHead]."

**Source:** [XiphWiki — OggOpus](https://wiki.xiph.org/OggOpus)

### Forbidden: ReplayGain Tags in Opus

RFC 7845 explicitly states:

> "To avoid confusion with multiple normalization schemes, an OpusTags packet SHOULD NOT contain any of the REPLAYGAIN_TRACK_GAIN, REPLAYGAIN_TRACK_PEAK, REPLAYGAIN_ALBUM_GAIN, or REPLAYGAIN_ALBUM_PEAK fields."

If ReplayGain tags are present, DBpoweramp should convert them to R128 format or strip them.

---

## 4. Tag Writing Behavior

### R128 Tag Value Constraints (per RFC 7845)

- Value MUST be a signed 16-bit integer (-32768 to +32767).
- Represented as ASCII decimal with no whitespace.
- Leading `+` or `-` sign allowed.
- Maximum 6 characters.
- Only one `R128_TRACK_GAIN` tag allowed.
- Only one `R128_ALBUM_GAIN` tag allowed.

**Examples of valid values:**
- `R128_TRACK_GAIN=-573`
- `R128_TRACK_GAIN=+128`
- `R128_TRACK_GAIN=0`

**Examples of invalid values:**
- `R128_TRACK_GAIN=-6.20` (decimal not allowed — must be integer)
- `R128_TRACK_GAIN=-6.20 dB` (unit string not allowed)
- `R128_TRACK_GAIN=-157` (valid but less precise)

### Vendor String

Same format as Vorbis Comment vendor string. Common values:
- `opusenc from Opus 1.3` — libopusenc encoder
- `dBpoweramp Release 15.1` — DBpoweramp encoder
- `Lavf60.x.x` — FFmpeg encoder

### Cover Art

Cover art in Opus uses `METADATA_BLOCK_PICTURE` (same as OGG Vorbis). The field value is base64-encoded of the complete picture structure.

### Fields That Should NOT Be Written

Per RFC 7845, Opus files should NOT contain:
- `REPLAYGAIN_TRACK_GAIN`
- `REPLAYGAIN_TRACK_PEAK`
- `REPLAYGAIN_ALBUM_GAIN`
- `REPLAYGAIN_ALBUM_PEAK`

These must be converted to R128 equivalents or removed.

---

## 5. Normalization Rules

| Rule | Behavior |
|------|----------|
| **R128 gain format** | Integer only, range -32768 to +32767, 6 chars max |
| **ReplayGain conversion** | ReplayGain_dB × 256 = R128 integer value |
| **Reference standard** | EBU R128 (-23 LUFS), not ReplayGain (-18 dB) |
| **Field name case** | UPPERCASE (`R128_TRACK_GAIN`) per RFC spec |
| **Multiple R128 fields** | Forbidden — only one of each allowed |
| **OpusHead output gain** | Applied in addition to tag-based R128 gain |

---

## 6. Edge Cases

1. **R128 value precision loss**: ReplayGain uses 2 decimal places ("-6.20 dB"). R128 uses integer steps of 1/256 dB (~0.0039 dB). Converting -6.20 dB gives -1587 (exactly -6.1992 dB). The small error is generally imperceptible, but precision stacking across albums should be noted.

2. **OpusHead output gain conflict**: If an OpusHead already has an output gain (e.g., from encoding with `--gain 0`), the R128 tag value must be relative to that header gain. Mismatches cause incorrect loudness normalization.

3. **Players that ignore R128**: Not all Opus players apply R128 gain automatically. Some only apply OpusHead output gain. DBpoweramp should write the gain to the OpusHead `output_gain` field and set `R128_TRACK_GAIN=0` as recommended by RFC 7845 when writing a single preferred gain value.

4. **Negative R128 values and dithering**: When applying R128 gain with negative values (attenuation), some players may introduce quantization noise at low bit depths. This is a player implementation issue, not a tagging issue.

5. **Album gain calculation**: `R128_ALBUM_GAIN` should be the gain to normalize the entire album, not the average of track gains. Album gain = loudness of album relative to -23 LUFS. Track gain = loudness of individual track relative to -23 LUFS.

---

## 7. DBpoweramp-Specific Behavior

- **Tag writing**: DBpoweramp uses libopusenc or FFmpeg's libopus for Opus encoding. Metadata uses OpusTags (Vorbis Comment base).
- **R128 gain**: DBpoweramp does NOT natively calculate or write R128 gain. Use a dedicated tool (e.g., `opusgain`, `eb128normalize`) to scan and apply R128 tags.
- **ReplayGain → R128 conversion**: DBpoweramp does NOT automatically convert ReplayGain to R128. Files tagged with ReplayGain (from FLAC/OGG Vorbis source) will retain those tags in Opus output unless explicitly handled.
- **Cover art**: Written as `METADATA_BLOCK_PICTURE` in OpusTags.
- **Vendor string**: Set to the encoding software identity.
- **OpusHead output gain**: Can be set via encoder options; defaults to 0.

---

## 8. Verification Checklist

- [ ] `R128_TRACK_GAIN` present as integer (no decimal, no "dB" suffix)
- [ ] `R128_TRACK_GAIN` range is -32768 to +32767
- [ ] `R128_ALBUM_GAIN` present as integer (if album gain applied)
- [ ] No `REPLAYGAIN_*` fields present (should be converted or absent)
- [ ] Only one `R128_TRACK_GAIN` and one `R128_ALBUM_GAIN` tag
- [ ] Vendor string identifies encoder
- [ ] Cover art present as `METADATA_BLOCK_PICTURE`
- [ ] Field names uppercase

---

## 9. Sources

1. [RFC 7845 — Ogg Encapsulation for the Opus Audio Codec](https://datatracker.ietf.org/doc/html/rfc7845)
2. [RFC 7845 — Section 5.2 Tag Definitions](https://datatracker.ietf.org/doc/html/rfc7845#section-5.2)
3. [XiphWiki — OggOpus](https://wiki.xiph.org/OggOpus)
4. [IETF Archive — Draft ietf codec oggopus 14](https://www.ietf.org/archive/id/draft-ietf-codec-oggopus-14.txt)
5. [Opus-Codec.org — opusfile API Header Information](https://opus-codec.org/docs/opusfile_api-0.7/group__header__info.html)
6. [XiphWiki — OggOpus (Full Content)](https://wiki.xiph.org/OggOpus)
