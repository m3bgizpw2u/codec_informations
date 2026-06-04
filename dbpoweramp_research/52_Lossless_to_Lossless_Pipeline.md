# 52 — Lossless-to-Lossless Pipeline

> **Scope:** How dBpoweramp handles conversions between lossless formats (FLAC, WAV, AIFF, ALAC, WavPack, TAK). Focus on bit-exactness verification, sample format changes (bit depth / sample rate), dithering, seek table preservation, and user-facing warnings. Primary sources: AudioUtils analysis[^1], Hydrogenaudio forum[^2][^3], FFmpeg documentation[^4], and FLAC format specification[^5].

---

## 1. Overview: Lossless-to-Lossless Conversion

Lossless-to-lossless conversion is the safest class of audio conversion. Formats like FLAC, WAV, AIFF, ALAC, WavPack, and TAK are mathematically lossless — they decode to **bit-identical PCM samples** regardless of the container or codec. A round-trip (FLAC → WAV → FLAC) produces a byte-identical result to the original.

> **Fundamental property:** Lossless means the output of the decoder is always identical to the input to the encoder. There is no information loss in the compression/decompression cycle. Converting between lossless formats changes the **container** and **compression algorithm**, not the PCM data itself.

---

## 2. Should Output Be Bit-Exact PCM Representation of Input?

**Yes, unconditionally.** Lossless-to-lossless conversion must produce bit-exact PCM output. The decoded samples at the output must be **numerically identical** to the decoded samples at the input.

### What "Bit-Exact" Means
- Each sample value must be identical: `output_sample[n][ch] == input_sample[n][ch]` for all `n` (sample index) and all `ch` (channel).
- No rounding, no truncation, no dithering, no gain changes.
- The only permitted operations are:
  - Container format conversion (changing the file wrapper)
  - Compression algorithm change (changing the codec within the container)
  - Metadata tag changes (non-audio data)

### Verification
Bit-exactness is verifiable by computing an **MD5/SHA256 hash of the decoded PCM stream** (ignoring metadata and container differences).

```bash
# Method 1: FFmpeg hash muxer (must specify codec to avoid 16-bit normalization)
ffmpeg -i input.flac -c:a copy -f streamhash -hash md5 -
ffmpeg -i output.wav -c:a copy -f streamhash -hash md5 -
# Match = bit-exact

# Method 2: FLAC metadata (FLAC embeds MD5 of uncompressed audio)
metaflac --show-md5sum input.flac
metaflac --show-md5sum output.flac
# Match = bit-exact (FLAC only; WAV has no embedded PCM hash)

# Method 3: Decode both to raw PCM and hash
ffmpeg -i input.flac -f s32le -acodec pcm_s32le - -y | md5sum
ffmpeg -i output.wav -f s32le -acodec pcm_s32le - -y | md5sum
```

> **Critical:** The FFmpeg hash muxer defaults to converting audio to signed 16-bit PCM before hashing. This means files with different bit depths produce different hashes even if the samples would be identical after zero-padding. Always specify the native bit depth with `-f s32le` or use `-c:a copy -f streamhash`.[^4]

### dBpoweramp's Verification
- dBpoweramp does **not** appear to have an automated bit-exact verification step for lossless-to-lossless conversions.
- However, **FLAC encodes by default with `--verify`**, which decodes the output in parallel and compares against the original PCM.
- FLAC's `STREAMINFO` block embeds an MD5 signature of the original uncompressed audio. When verifying (`flac -V` or `flac --test`), FLAC re-decodes and compares.

---

## 3. Does dBpoweramp Verify Bit-Exactness After Lossless→Lossless?

**No explicit verification** is documented. dBpoweramp:
- Trusts that the decoder + encoder pipeline is bit-exact for lossless sources.
- Does not display a "bit-exactness verified" message.
- Does not write a verification result to any log.

This is reasonable because:
1. Lossless codecs are mathematically proven to be bit-exact.
2. Verification adds CPU overhead (a second decode pass).
3. Any non-bit-exactness would indicate a bug in the decoder or encoder — not an expected condition.

**However**, if DSP effects are applied (Phase 4), the output is no longer bit-exact by definition — the audio has been processed. In this case, dBpoweramp does not claim bit-exactness.

---

## 4. What Happens When Sample Format Changes (24-bit FLAC → 16-bit WAV)?

**This is NOT a lossless operation.** Reducing bit depth permanently destroys information. This is the most critical edge case in the "lossless-to-lossless" category.

### What Changes
- **Bit depth reduction** (24-bit → 16-bit, 24-bit → 32-bit float → 16-bit): The bottom 8 bits are discarded. This is irreversible.
- **Sample rate conversion** (96kHz → 44.1kHz): High frequencies above Nyquist are removed. This is irreversible.

### dBpoweramp's Behavior
- dBpoweramp **allows** bit depth and sample rate changes without blocking the conversion.
- The user must know to use the DSP chain for this.
- **No implicit warning** is shown for bit depth reduction — but the operation is technically valid (you're creating a new, lower-quality version).

### What Should Happen: Dithering for Bit Depth Reduction
When reducing bit depth, **dithering is mandatory** for professional quality:[^6][^7]

| Conversion | Action | Rationale |
|---|---|---|
| 24-bit → 16-bit | **Dither** (TPDF or shaped) | Prevents harmonic quantization distortion |
| 32-bit float → 24-bit | Dither optional | 24-bit noise floor is already inaudible |
| 24-bit → 24-bit (no change) | Pass through | No action needed |
| 16-bit → 24-bit | Zero-pad | No information to add |
| 24-bit → 32-bit float | Convert | No information loss in float |

### Types of Dither
1. **TPDF (Triangular Probability Density Function):** The simplest and most widely accepted. Adds noise spread over ±1 LSB.
2. **Shaped dither (Shibata, UV22):** Noise is shaped to push quantization noise into less audible frequency ranges. Used in professional mastering.
3. **No dither (truncation):** Produces harmonic distortion on low-level signals. Acceptable only for quick previews.

> **The First Law of Dithering:** Do NOT dither a signal more than once. Dithering should happen only when converting to the final output bit depth. If you convert 24-bit → 32-bit float → 16-bit, dither only at the 16-bit truncation step.[^6]

### What Should Happen: Sample Rate Conversion
- High-quality SRC is required (see Phase 4.1 in Document 50).
- Simple linear interpolation is fast but introduces high-frequency rolloff.
- **SoX VHQ**, **Secret Rabbit Code (libsamplerate)**, or **ffmpeg soxr** are appropriate choices.
- **Do NOT** use cheap SRC (e.g., Windows DirectSound resampler) for quality-critical work.

---

## 5. Does dBpoweramp Dither? Noise-Shape? Just Truncate?

**dBpoweramp's documented behavior:** The ReplayGain DSP effect and volume normalization operate on the float PCM buffer. For pure bit-depth reduction (without DSP), dBpoweramp's behavior is **not explicitly documented** as including dithering.

### What Open-Source Tools Do
- **SoX:** Automatically dithers when reducing bit depth (unless `-D` flag is used). This is considered SoX's "fail-safe" behavior.
- **FLAC CLI:** **Cannot dither by design.** FLAC is limited to 24-bit maximum; 32-bit files are "truncated" (no dither) to 24-bit. This is a known limitation.
- **ffmpeg:** Does not dither by default when converting bit depth. Must use `-af aformat=sample_fmts=s16p:dither_method=triangular` to enable dithering.

### Recommendation for Our Implementation
- When the target bit depth is lower than the source:
  - If DSP effects are applied: dither after the final DSP stage, at the target bit depth.
  - If no DSP effects: still dither when reducing to 16-bit (or warn the user that truncation will occur).
  - Use TPDF dithering as the default; offer shaped dither (Shibata) as an optional higher-quality mode.
- When the target bit depth is higher: zero-pad (no dither needed).

---

## 6. ALAC→FLAC: Both Lossless But Different Containers — Is Conversion Always Bit-Exact?

**Yes.** ALAC (Apple Lossless) and FLAC are both lossless codecs. They decode to identical PCM samples. Converting ALAC → FLAC → ALAC produces byte-identical output to the input.

### Verification
- Both ALAC and FLAC embed MD5 checksums of the uncompressed audio (FLAC in `STREAMINFO`, ALAC in the `mdia` chunk).
- Decode both files to raw PCM and compare; the hashes will match.

### What Changes in the Conversion
- **Container:** M4A → FLAC (different file wrapper).
- **Codec:** ALAC frame structure → FLAC frame structure.
- **Metadata:** iTunes atoms → Vorbis Comments.
- **Seek table:** ALAC has no seek table; FLAC's encoder will generate one. This is fine.
- **Padding:** ALAC files may have padding atoms; FLAC's default padding is ~8192 bytes. Both are metadata and don't affect audio.

### Edge Case: DRM-Protected ALAC
- If the M4A file is DRM-protected (iTunes Match / Apple Music), dBpoweramp cannot decode it.
- DRM-free ALAC (purchased from iTunes before DRM was removed) converts normally.

---

## 7. What Codec Info Is Preserved (Seek Tables, Padding)?

### Seek Tables
- **FLAC → WAV:** FLAC seek tables are **lost**. WAV has no native seek table structure.
- **FLAC → ALAC:** FLAC seek tables are **lost**. ALAC/M4A has no native seek table structure.
- **FLAC → AIFF:** FLAC seek tables are **lost**. AIFF has no native seek table structure.
- **FLAC → FLAC:** Seek table is **preserved** (FLAC to FLAC, the encoder can copy the seek table).
- **When re-encoding to a new FLAC:** A new seek table is generated (default: one seekpoint per 10 seconds).

**Practical impact:** Seeking in WAV/ALAC files is slower on large files or network streams. Players fall back to scanning for sync codes. This is not a quality issue — only a performance issue.

### Padding
- **FLAC:** Optional PADDING metadata block. Default: ~8192 bytes of padding to allow in-place tag editing.
- **WAV:** No padding; `data` chunk ends exactly at the last sample.
- **ALAC/M4A:** Padding atoms within the `moov` atom allow in-place tag editing.
- **AIFF:** Optional padding chunks.

**Preservation:** Padding is not audio data and does not affect quality. Padding policy varies by format and is not preserved across format changes.

### Encoder Delay / Padding (Gapless Info)
- **FLAC:** Natural gapless — no padding needed. `STREAMINFO` contains `total_samples`; players seek to sample 0.
- **WAV:** Natural gapless — no special handling needed.
- **ALAC/M4A:** Natural gapless.
- Gapless info is preserved because lossless formats have no encoder delay or padding (unlike MP3/AAC).

---

## 8. User Warnings

### fre:ac Warnings
fre:ac displays a warning when converting from a lossy source to lossless. However, for **true lossless → lossless**, fre:ac does **not** display a warning — because no quality loss occurs.

### dBpoweramp Warnings
- **No warning** for lossless → lossless without DSP processing.
- **No warning** for lossless → lossless with bit-depth/sample-rate reduction (this is a gap — users may not realize they're reducing quality).

### Hydrogenaudio Community Request
The Hydrogenaudio community has requested that converters show a warning when **DSP processing is applied** during lossless-to-lossless conversion:[^8]

> "When you convert from lossless to lossless there should be a warning when you add any kind of processing. Such 'You change the original audio track, are you sure?'"

### Recommended Warnings for Our Implementation
| Scenario | Warning? | Message |
|---|---|---|
| FLAC → WAV (no DSP) | No | Not needed; bit-exact |
| FLAC → WAV (DSP: volume normalization) | Yes | "Processing will modify audio data. Output will not be bit-exact to source." |
| FLAC → WAV (DSP: SRC 96kHz→44.1kHz) | Yes | "Sample rate conversion will remove frequencies above 22.05kHz." |
| FLAC → 16-bit WAV (no DSP) | Yes | "Bit depth reduction from 24-bit to 16-bit will discard information." |
| ALAC → FLAC (no DSP) | No | Not needed; bit-exact |
| FLAC → WAV (no DSP) | No | Not needed; bit-exact |

---

## 9. Edge Cases

### Edge Case 1: FLAC with Embedded CUE Sheet
- **Behavior:** FLAC files often embed a CUE sheet in a `CUESHEET` Vorbis Comment field.
- **FLAC → WAV:** The CUE sheet is preserved as a Vorbis Comment tag in the WAV file (if using ID3v2-in-WAV or BWF).
- **FLAC → ALAC/M4A:** The CUE sheet must be converted to an `stik` atom or equivalent; M4A does not natively support track markers.
- **Recommendation:** Use CUETools for CUE sheet handling across format boundaries.

### Edge Case 2: Multi-Channel FLAC (5.1) → Stereo WAV
- **Phase 4 (DSP):** Channel down-mix is applied (stereo).
- **This is NOT bit-exact** — audio information is lost in the down-mix.
- **No warning** may appear if the user is unaware of the down-mix.
- **Recommendation:** Warn when channel count changes.

### Edge Case 3: FLAC with Negative DC Offset
- **Phase 4:** DC offset removal is a DSP effect.
- **This is NOT bit-exact** — the DC component is modified.
- **Recommendation:** Warn when DC offset removal is enabled for lossless-to-lossless.

### Edge Case 4: Rounded Sample Rates (48kHz → 44.1kHz)
- **SRC:** 48kHz → 44.1kHz is a non-integer ratio (160:147).
- **Every SRC algorithm introduces some artifacts** (pre-ringing, HF rolloff).
- **The result is still high quality** — CD-accurate sample rates from 48kHz masters are standard practice.
- **No warning needed** beyond standard SRC disclosure.

### Edge Case 5: WavPack (WV) → FLAC
- WavPack can operate in **lossless** or **hybrid** mode.
- **Lossless WV → FLAC:** Bit-exact.
- **Hybrid WV (lossy+correction) → FLAC:** The lossy component is extracted and encoded; the correction file is discarded. Result is lossy, not lossless.
- **Detection:** WavPack hybrid mode files can be identified by their `.wv` file size (smaller than a pure lossless file of the same duration).

### Edge Case 6: TAK → FLAC
- TAK (Tom's Lossless Audio Kompressor) is a relatively new lossless codec.
- If a TAK decoder is available, conversion to FLAC is bit-exact.
- **Detection:** TAK files start with the magic bytes `tBaK`.

### Edge Case 7: FLAC with Very Large Embedded Image
- **Phase 1:** Cover art is read (up to tens of MB for high-resolution images).
- **Phase 6:** Cover art is written to output.
- **Bit-exactness:** Cover art is metadata, not audio. Changing the embedded image does not affect PCM data.
- **Recommendation:** Warn if cover art exceeds a size threshold (e.g., >5 MB) and prompt to resize.

### Edge Case 8: Bit-Exactness Across Big-Endian / Little-Endian
- **WAV:** Little-endian.
- **AIFF:** Big-endian.
- **Decoded PCM:** Both convert to host-native byte order.
- **Re-encoded:** The encoder writes in the format's native byte order.
- **Result:** Bit-exact at the PCM level; raw bytes differ due to endianness (but both decode to identical sample values).

### Edge Case 9: Floating-Point Source (32-bit float WAV)
- **Phase 2:** Detected as `bit_depth = 32` (float).
- **Lossless codecs** (FLAC, ALAC) limit to 24-bit integer maximum.
- **Conversion:** 32-bit float → 24-bit integer requires quantization.
- **This is NOT bit-exact** — the float → int conversion involves rounding.
- **dBpoweramp:** Likely rounds to the nearest integer; does not dither for float→int (dithering is for int→int bit-depth reduction).
- **Recommendation:** Warn when converting 32-bit float to 24-bit FLAC.

### Edge Case 10: Corrupt FLAC File (bit-exact but damaged)
- **Phase 3:** Decoder encounters an error frame.
- **FLAC with `verify`:** Detects frame CRC mismatch; aborts.
- **FLAC without `verify`:** Silently outputs garbage for corrupt frames.
- **Phase 8 verification:** Re-decoding and hashing would detect the corruption.
- **Recommendation:** Always enable decoder verification for lossless sources.

---

## Would a User Notice Any Difference from dBpoweramp?

**Mostly no.** dBpoweramp's lossless-to-lossless handling is solid. The main differences would be:

1. **Dithering on bit-depth reduction:** If our implementation truncates instead of dithering, users may notice **harmonic distortion on very quiet passages** (e.g., fade-outs) at 16-bit output. This is a significant quality difference.
2. **DSP processing warnings:** If dBpoweramp shows a warning when applying volume normalization to a lossless file and ours doesn't, users might unknowingly modify their audio.
3. **FLAC MD5 verification:** dBpoweramp relies on FLAC's built-in verification (via `--verify`). Our implementation should do the same.
4. **Floating-point handling:** The 32-bit float → 24-bit integer conversion path is rarely exercised; behavior here may differ.

---

## References

[^1]: AudioUtils — "What is audio transcoding vs converting?" https://audioutils.com/blog/what-is-audio-transcoding-vs-converting
[^2]: Hydrogenaudio Forums — "Lossless to lossless, is it always bit perfect?" (multiple threads)
[^3]: SuperUser — "How to compare two lossless audio files." https://superuser.com/questions/531778/how-to-compare-two-lossless-audio-files
[^4]: FFmpeg Formats Documentation — hash muxer. https://ffmpeg.org/ffmpeg-formats.html#hash-1
[^5]: FLAC format documentation. Xiph.org. https://xiph.org/flac/documentation_format_overview.html
[^6]: AudioFanzine — "Much Ado About Dithering." https://en.audiofanzine.com/mastering/editorial/articles/much-ado-about-dithering.html
[^7]: Fremen — "Audio Dithering Guide." https://fremen.fi/guru/audio-dithering/
[^8]: Hydrogenaudio Forum — "Warning for DSP processing on lossless to lossless." https://hydrogenaudio.org/index.php/topic,117546.0.html
