# 53 — Lossless-to-Lossy Pipeline

> **Scope:** How dBpoweramp handles conversions from lossless formats (FLAC, WAV, AIFF, ALAC, WavPack) to lossy formats (MP3, AAC, Opus, Vorbis, WMA). Focus on metadata handling, bitrate selection, transparency, and gapless playback. Primary sources: Hydrogenaudio Knowledgebase[^1], dBpoweramp documentation[^2], and community discussions[^3].

---

## 1. Overview: Lossless-to-Lossy as First-Generation Encoding

Lossless-to-lossy conversion is the standard **archival → distribution** workflow. A lossless master is transcoded to a lossy format for playback on space-constrained or compatibility-focused devices. This is the cleanest conversion path — the lossy output is a **first-generation encode** from pristine source material.

> **Key property:** Lossless → lossy introduces loss for the first time. The output is constrained by the lossy encoder's quality settings, not by any prior encoding. This is fundamentally different from lossy → lossy (Document 55) or lossy → lossless (Document 54).

---

## 2. Does dBpoweramp Warn That Output Is "Lossy Audio in Lossless Container"?

### What This Means
Some lossy formats use a **lossless container** (e.g., MP3 in an M4A container, Opus in an Ogg container). The container is lossless; the audio codec inside is lossy. A user might mistakenly believe that because the file is a `.m4a`, the audio is lossless.

### dBpoweramp's Behavior
- **No explicit warning** that the output contains lossy audio in a lossless container.
- The user must understand that:
  - `FLAC` → `MP3`: The MP3 file contains lossy audio.
  - `FLAC` → `M4A` (AAC): The M4A file contains lossy audio (unless `Apple Lossless` is selected as the encoder).
  - `FLAC` → `Opus`: The Opus file contains lossy audio.
- The **file extension** is the primary indicator: `.mp3`, `.m4a` (with AAC), `.ogg`, `.opus`.
- The **encoder name** in the tag (e.g., `LAME 3.100`, `qaac 2.71`) confirms the audio is lossy.

### Recommendation for Our Implementation
- Display a clear summary before conversion: `"Encoding to MP3 (lossy). Original FLAC audio will be re-encoded."`
- The file extension and encoder name already communicate this; a redundant warning may be unnecessary.

---

## 3. Does dBpoweramp Embed Metadata Indicating Source Was Lossless?

**No.** dBpoweramp does not embed any metadata tag that indicates the source was a lossless file.

### What Could Be Written (Optional)
- **ReplayGain origin:** `REPLAYGAIN_ORIGIN = "lossless-source"` — but no standard tag exists for this.
- **Source format tag:** Some encoders/tags support an `ORIGINAL` field:
  - `ORIGINAL_FORMAT = "FLAC"` in some tag schemas.
  - MP3 `TSO` (original filename) — not standardized.
- **User convention:** Some users rename files as `source_album_track_##_lossless.flac` before transcoding. This is a naming convention, not a tag.

### What dBpoweramp Actually Writes
- The encoder tag (`TSSE` in ID3, `Encoder` in Vorbis, `©too` in MP4) reflects the **new encoder** (e.g., `LAME 3.100`).
- The **original encoder tag** from the lossless source (e.g., `reference libFLAC 1.4.0`) is **dropped**.

### Handling the Encoder Tag
- **Recommended approach (dBpoweramp likely follows this):** Write the **new encoder** name/version.
- This is the most technically accurate choice — the audio was re-encoded by the new encoder.
- It does not claim the source was lossless, but it accurately describes what created the output file.

---

## 4. What Does dBpoweramp Do with Original Encoder Tags?

### Scenario: FLAC (libFLAC) → MP3 (LAME)

**Source tags:**
```
Encoder: reference libFLAC 1.4.0 20220730
Encoder Settings: -8
REPLAYGAIN_TRACK_GAIN: -0.80 dB
REPLAYGAIN_TRACK_PEAK: 0.989258
```

**Target tags (MP3):**
```
TSSE: LAME 3.100
REPLAYGAIN_TRACK_GAIN: -0.80 dB
REPLAYGAIN_TRACK_PEAK: 0.989258
```

### What Is Preserved
- **ReplayGain tags:** These are format-independent; they are converted to the target format's tag conventions.
- **All other metadata:** Title, artist, album, track, disc, date, genre, ISRC, etc.
- **Cover art:** Preserved and converted to the target format's cover art frame.

### What Is Dropped
- **Original encoder tag:** Replaced with the new encoder.
- **Original encoder settings:** Discarded (not relevant to the new encode).
- **Original ReplayGain origin indicator:** Not written (no standard field).

### Multi-Value Fields
- If the source has multiple artists in Vorbis Comments:
  ```vorbis
  ARTIST=John Lennon
  ARTIST=Paul McCartney
  ```
- Converted to ID3v2 `TPE1`: `John Lennon / Paul McCartney` (joined with ` / ` delimiter).
- This is the standard behavior across dBpoweramp, foobar2000, and beets.

---

## 5. Bitrate Selection Defaults and Quality Modes

### Common Default Bitrates
| Format | Default | Transparent Quality | Notes |
|---|---|---|---|
| **MP3** | 320 kbps (CBR) or V0 (VBR) | ~V2 (190–210 kbps VBR) | LAME recommended; `-V 2` for transparency |
| **AAC** | 256 kbps | ~192 kbps | QAAC recommended; HE-AAC for lower bitrates |
| **Opus** | 128 kbps (VBR) | ~96 kbps | Transparent at 128 kbps for most music |
| **Vorbis** | q8 (~256 kbps VBR) | ~200 kbps VBR | aoTuV b6.03 recommended |
| **WMA** | 192 kbps | ~128 kbps | Windows Media Player encoder |

### Quality Modes
| Mode | Description | Use Case |
|---|---|---|
| **CBR (Constant Bitrate)** | Fixed bitrate throughout | Streaming, compatibility, predictable file size |
| **VBR (Variable Bitrate)** | Bitrate varies with audio complexity | Best quality-per-filesize ratio |
| **ABR (Average Bitrate)** | Average bitrate with variance | Legacy compatibility |
| **CRF (Constrained Rate Factor)** | Quality target with max bitrate cap | Opus default |

### dBpoweramp Defaults
- **MP3:** LAME encoder; default quality is **V2 (≈190–210 kbps VBR)** — transparent for most listeners.
- **AAC:** QAAC encoder (via CoreAudio); default is **Apple AudioVariable Bitrate** (high quality).
- **Opus:** opusenc; default is **128 kbps VBR**.
- **dBpoweramp help file:** Recommends **VBR** modes for archival-to-distribution workflows.

### Transparency Threshold
The **transparency threshold** is the bitrate at which a lossy encoder produces output indistinguishable from the lossless source in double-blind ABX testing.

| Format | Transparency Threshold | Source |
|---|---|---|
| MP3 (LAME) | ~190–210 kbps VBR (V2) | Hydrogenaudio listening tests |
| AAC (QAAC) | ~192 kbps | Hydrogenaudio listening tests |
| Opus | ~96 kbps | Opus listening tests |
| Vorbis (aoTuV) | ~200 kbps VBR | Hydrogenaudio listening tests |

> **Key insight:** For a first-generation encode from lossless, the transparent bitrate is the **only** quality ceiling. There is no benefit to encoding at 320 kbps if 192 kbps is already transparent. The user's bitrate choice should target transparency, not maximum bitrate.

---

## 6. Lossless→Lossy as Transparent (Same Listening Experience at High Bitrate)

### What "Transparent" Means
- In a **properly conducted double-blind ABX test**, a listener cannot reliably identify which of two samples is the lossy encode and which is the lossless original.
- Transparency is **format-dependent** and **source-dependent**:
  - Simple acoustic music (solo piano, acoustic guitar) may be transparent at lower bitrates.
  - Complex acoustic music (orchestra, dense rock/pop) requires higher bitrates.
  - **Transparency ≠ identical waveforms.** The lossy encoder introduces inaudible artifacts; the PCM samples are different, but human hearing cannot distinguish them.

### dBpoweramp's Approach
- dBpoweramp recommends **VBR modes** (V2 for MP3, Apple AudioVBR for AAC) as the default.
- This automatically allocates more bits to complex passages and fewer to simple passages.
- VBR at transparent quality typically produces **smaller files than CBR at the same perceived quality**.

### User Expectations Management
- Users upgrading from MP3 128 kbps to FLAC may be surprised that re-encoding their FLAC to MP3 320 kbps "sounds the same" as their old 128 kbps files.
- **Education:** The bottleneck is the first generation of lossy encoding. A 320 kbps MP3 re-encoded from FLAC is a 320 kbps MP3 — no better and no worse than any other 320 kbps MP3 of the same source.
- **dBpoweramp help file:** "Lossless to lossy re-encodes the audio. The quality is determined by the encoder settings chosen."

---

## 7. How Gapless Info Is Handled (MP3, AAC, Opus)

### The Gapless Problem
Lossy encoders introduce **encoder delay** (samples before the first audio frame) and **encoder padding** (samples after the last audio frame). Without compensation, there is a gap between tracks on gapless-capable players.

### dBpoweramp's Gapless Handling

**MP3 (LAME):**
- LAME encodes with a delay of 576 samples and padding of 1152 samples.
- This information is written to the **LAME Info tag** (`LAME` header) as `encoder_delay` and `encoder_padding` in samples.
- dBpoweramp reads these values during **Phase 2 (Audio Analysis)**.
- During **Phase 3 (Audio Decode)**, the delay samples are skipped at the start, and the padding samples are truncated at the end.
- The **output MP3** is re-encoded with its own delay/padding values.
- Gapless playback is maintained because the delay/padding chain is compensated for at each stage.

**M4A/AAC (QAAC/CoreAudio):**
- Apple's AAC encoder introduces encoder delay and padding.
- This is stored in the **`iTunSMPB` tag** (or the `mdia` atom's `stbl` for gapless in iTunes).
- Format: `begin samples | end samples | orig sample count | begin | end` — all in samples.
- dBpoweramp reads `iTunSMPB` on decode; writes a new `iTunSMPB` on encode.

**Opus:**
- Opus has a **pre-skip** value (typically 3120 samples at 48kHz) in the `OpusHead` header.
- The pre-skip compensates for the encoder's look-ahead.
- dBpoweramp reads `OpusHead` pre-skip on decode; writes a new `OpusHead` on encode.
- Opus gapless is handled natively by the Opus spec.

**FLAC / WAV / ALAC (Lossless):**
- **No encoder delay or padding.** Lossless formats have natural gapless playback.
- No gapless metadata needed.

### Gapless Playback Verification
- Players that support gapless (iTunes, foobar2000, VLC, Spotify, Navidrome, Roon) read the delay/padding metadata and compensate.
- **Ungapless players** (some older hardware, some streaming clients) do not compensate — a small gap (~21–46 ms) may be audible between tracks.

---

## 8. Edge Cases

### Edge Case 1: Very Short Tracks (e.g., 2-Second Sound Effects)
- **Problem:** Encoder delay/padding (576 + 1152 = 1728 samples at 44.1kHz ≈ 39 ms) is significant relative to a 2-second (2000 ms) track.
- **Result:** The actual audio content is displaced by the encoder's framing.
- **Mitigation:** Most encoders handle this correctly by reporting the delay/padding in metadata; the player compensates.
- **Some players** may still produce a gap if they don't read the gapless metadata.

### Edge Case 2: Source Sample Rate Not Supported by Encoder
- **Example:** 88.2kHz FLAC → MP3 (LAME). LAME supports sample rates of 16/22.05/24/32/44.1/48 kHz.
- **Behavior:** LAME internally upsamples 88.2kHz → 176.4kHz or downsamples → 44.1kHz.
- **FFmpeg/LAME:** Automatically resample to a supported rate.
- **dBpoweramp:** Likely uses its DSP chain (Phase 4) to resample to a standard rate before encoding.

### Edge Case 3: Multi-Channel (5.1) Source → Stereo Output
- **Phase 4:** Channel down-mix is applied.
- **Bitrate:** 5.1 audio at 320 kbps is actually 320 kbps for 6 channels ≈ 53 kbps per channel.
- **Stereo output at 320 kbps:** 160 kbps per channel — higher effective quality per channel.
- **Recommendation:** Inform the user that channel count affects effective quality.

### Edge Case 4: Source ReplayGain Tags
- **Source has ReplayGain tags** (e.g., `-6.20 dB` track gain).
- **Behavior:** ReplayGain tags are preserved and converted to the target format's conventions.
- **Do NOT apply the gain** during encoding unless the user explicitly enables the ReplayGain DSP.
- **If applying ReplayGain during encoding:** The gain is applied to the PCM (Phase 4); the resulting MP3 is pre-normalized. ReplayGain tags are updated to reflect the applied gain.

### Edge Case 5: Source Cover Art Type
- **Source FLAC has cover art** with picture type 3 (Front Cover) via `METADATA_BLOCK_PICTURE`.
- **Target MP3:** Written as `APIC` frame with `picture_type = 3` (Front Cover).
- **This is correct** — dBpoweramp preserves the picture type.
- See Section 3 of the Quick Reference.

### Edge Case 6: Source Has Multiple Cover Art Images
- **Source has Front Cover (type 3) + Back Cover (type 4).**
- **Target:** Only Front Cover is written (type 3).
- **Back Cover:** Dropped (MP3 doesn't commonly support multiple images; dBpoweramp writes only the primary).
- **Alternative:** Write both images with their respective picture types if the encoder supports it.

### Edge Case 7: Bitrate Exceeds Source's Lossless Bit Depth
- **Example:** 16-bit/44.1kHz FLAC (~1411 kbps) → MP3 128 kbps.
- **Result:** The MP3 is 128 kbps — clearly lossy.
- **Example:** 24-bit/192kHz FLAC (~8291 kbps) → MP3 320 kbps.
- **Result:** The MP3 is still 320 kbps — the excess source information is discarded by the encoder.
- **No special handling needed** — the encoder naturally discards information beyond its bitrate budget.

### Edge Case 8: Lossless→Opus at 8 kHz (Voice)
- **Source:** FLAC at 44.1kHz/stereo.
- **Target:** Opus at 8 kbps mono.
- **Result:** Extreme downsampling and downscaling: 44.1kHz → resampled to 8kHz (pre-filter), stereo → mono, extreme compression.
- **Not recommended** for music — only suitable for voice recordings.
- **dBpoweramp:** Allows any encoder with any source; does not block extreme conversions.

### Edge Case 9: Encoding to the Same Format (Lossless FLAC → Lossy MP3)
- **Source:** FLAC.
- **Target:** MP3.
- **Phase 0:** Format check passes (different formats).
- **Phase 1–4:** Tags read, audio decoded, DSP applied.
- **Phase 5:** Encoded to MP3.
- **This is the expected lossless→lossy workflow.**

### Edge Case 10: Converting a DSD/SACD Source to Lossy
- **Source:** DSD64 (2.8 MHz) in DFF or DSF container.
- **Phase 2:** Decoder detects DSD format.
- **Phase 3:** DSD is decoded to PCM (DSD-to-PCM conversion at 32-bit/352.8kHz or 384kHz).
- **Phase 4:** DSP chain operates on the PCM.
- **Phase 5:** Encoded to lossy format.
- **Result:** The lossy output is a first-generation encode from DSD. The DSD → PCM conversion is a separate quality step.
- **dBpoweramp:** Supports DSD decoding via the DSD decoder codec.

---

## Would a User Notice Any Difference from dBpoweramp?

**Likely no** for the core encoding path. Key alignment points:

1. **Gapless info handling** — dBpoweramp correctly reads and writes delay/padding metadata. Our implementation must do the same.
2. **VBR quality defaults** — dBpoweramp uses sensible VBR defaults (V2 for MP3). Our defaults should match.
3. **Tag preservation** — all tags preserved, encoder tag updated to new encoder.
4. **Cover art picture type** — preserved as type 3.

**Possible differences:**
- **Encoder quality differences** — different encoder versions (LAME 3.100 vs. LAME 3.99) may produce slightly different output at the same bitrate. This is typically imperceptible.
- **Multi-channel down-mix matrix** — dBpoweramp's stereo down-mix may differ from ours in channel weighting.

---

## References

[^1]: Hydrogenaudio Knowledgebase — "Transcoding." https://wiki.hydrogenaudio.org/index.php?title=Transcoding
[^2]: dBpoweramp Music Converter Help. https://dbpoweramp.com/Help/dMC/dMC
[^3]: dBpoweramp Community Forum — Lossless and Lossy discussion. https://forum.dbpoweramp.com/forum/dbpoweramp/cd-ripper/35133-lossless-and-lossy
