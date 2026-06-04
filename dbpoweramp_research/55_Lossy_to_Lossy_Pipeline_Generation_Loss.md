# 55 — Lossy-to-Lossy Pipeline: Generation Loss

> **Scope:** How dBpoweramp handles conversions from lossy formats to lossy formats (MP3 → AAC, MP3 → MP3, AAC → MP3, etc.). Focus on generation loss warnings, same-codec transcode behavior, bitrate checking, encoder chain metadata, and safety options. Primary sources: Hydrogenaudio Knowledgebase[^1], Hydrogenaudio listening tests[^2], dBpoweramp documentation[^3], foobar2000 behavior[^4], and beets documentation[^5].

---

## 1. Overview: Why Lossy→Lossy Is the Worst Conversion

Lossy-to-lossy conversion is the **most damaging** class of audio conversion. Unlike lossy→lossless (which is wasteful but harmless to audio quality) or lossless→lossy (which introduces loss only once), lossy→lossy compounds the artifacts of the first generation and introduces new artifacts from the second.

> **The compounding effect:** A lossy encoder discards information based on its psychoacoustic model. The second encoder — operating on the already-degraded output of the first — cannot recover the discarded information. It must re-apply its own psychoacoustic model to data that no longer accurately represents the original signal. Quantization noise from the first generation is amplified and compounded by the second.

---

## 2. Does dBpoweramp Warn About Generation Loss When Transcoding Lossy→Lossy?

### dBpoweramp Behavior
- **No interactive warning dialog** for lossy→lossy conversions.
- **Documentation states:** *"Lossy (mp3, aac) conversion to other formats is not recommended, when compressing audio quality is lost forever."*[^3]
- The help file provides the information; the UI does not block or prompt.

### Comparison with Other Tools
| Tool | Warning? | Type |
|---|---|---|
| **foobar2000** | Yes | Dialog: "Transcode warning — lossy to lossy" |
| **dBpoweramp** | No (docs only) | None |
| **FFmpeg** | No | None |
| **beets** | No (opt-in skip) | `never_convert_lossy_files` skips lossy sources |
| **fre:ac** | Yes | Losy→lossless warning (also applies) |
| **AudioUtils** | No | Categorization only |

### foobar2000's Warning
foobar2000 displays the **same warning** for all lossy-to-lossy cases:[^4]
- MP3 → MP3: Warn
- MP3 → AAC: Warn
- AAC → MP3: Warn
- OGG → MP3: Warn
- The warning covers any transcode from a lossy source, not just cross-codec.
- The warning can be disabled in **Preferences → Advanced → Tools → Converter → Show transcoding warnings**.

---

## 3. Same-Codec Transcode (MP3 128 → MP3 320): Does It Warn?

### The "Upscaling" Misconception
Users sometimes believe that transcoding a **128 kbps MP3** to a **320 kbps MP3** will "recover quality" or produce a "higher quality" file. **This is incorrect.**

### Why 128 → 320 kbps MP3 Is Still Worse Than Original
1. The 128 kbps MP3 already discarded ~90% of the original PCM information.
2. The 320 kbps encoder receives the **degraded PCM** from the 128 kbps decoder.
3. The 320 kbps encoder re-quantizes this degraded PCM — it cannot restore the lost information.
4. The result is a **320 kbps file with 128 kbps quality**. The file is larger; the audio is no better.

### Hydrogenaudio Test Data
ABX test results from Hydrogenaudio community:[^2]
- Same-codec transcoding (LAME → LAME): **"Not one test I couldn't ABX in several seconds."**
- Every tested re-encoding was audibly distinguishable from the original — even at high bitrates.
- Cross-codec re-encoding (OGG 96/128 → MP3, MP3 128 → WMA 128): **"unacceptable deterioration"**
- AAC at 96 kbps: **"better than MP3 and WMA, but still had objectionable artifacts"**

### Recommendation for Our Implementation
- **Display an explicit warning** for same-codec transcoding, including upscaling scenarios.
- Message: `"Converting MP3 to MP3 will not improve audio quality. The output will be re-encoded and may sound worse. 128 → 320 kbps does not restore lost quality."`
- Allow the user to proceed or cancel.

---

## 4. Does dBpoweramp Check Bitrate (Warn When Downsampling)?

### dBpoweramp's Conditional Encoding DSP
- **Yes**, dBpoweramp can **warn or skip** based on bitrate thresholds.
- Configuration: Set conditions like "If source bitrate > target bitrate, skip" or "If source bitrate ≥ 320 kbps, skip."
- This prevents the **"I'm re-encoding my 320 kbps MP3s to 128 kbps for my MP3 player"** scenario, where the user might not realize the quality degradation.

### Bitrate Change Scenarios
| Scenario | Quality Effect | Warning Needed? |
|---|---|---|
| MP3 128 → MP3 320 | No improvement | Yes |
| MP3 320 → MP3 128 | Degradation | Yes |
| MP3 128 → AAC 256 | Cross-codec; different artifacts | Yes |
| MP3 320 → AAC 320 | Cross-codec; slightly different quality | Yes |
| MP3 128 → Opus 128 | Different codec; better at same bitrate | Optional |

### Downsample vs. Upsample Warnings
- **Downsample** (higher bitrate → lower bitrate): Clear quality loss. Warn.
- **Upsample** (lower bitrate → higher bitrate): No quality gain. Warn.
- **Same bitrate, cross-codec:** Different artifacts; quality may be similar but not identical. Inform.

---

## 5. Cross-Codec Lossy→Lossy (MP3 → AAC): Generation Loss Warning?

### Cross-Codec Compounding
Cross-codec lossy→lossy is **worse than same-codec** because each codec has a different psychoacoustic model:[^1]

> *"X already removes some information it considers unimportant, but which in fact is important for Y. The result is Y's encoding will be greatly maimed."*

### Example: MP3 → AAC
1. **MP3 (LAME) encoding:** Discards frequency bands based on LAME's psychoacoustic model. Some information deemed "masked" by LAME is actually relevant to AAC's model.
2. **AAC decoding:** Reconstructs PCM from MP3's quantized coefficients. The reconstruction errors are different from the original.
3. **AAC encoding:** Applies its own psychoacoustic model to already-degraded PCM. New quantization errors compound the existing artifacts.
4. **Result:** The AAC file has artifacts from **both** LAME's encoding **and** AAC's encoding, layered on top of each other.

### Hydrogenaudio Test
MP3 at 128 kbps → OGG Vorbis at 128 kbps: *"audibly different from source mp3, but still acceptable."*
MP3 and WMA at 128 kbps: *"unacceptable deterioration."*
AAC at 96 kbps from MP3 source: Better than MP3/WMA but still objectionable artifacts.

### Cross-Codec Conversion Tables
| Source → Target | Quality Direction | Notes |
|---|---|---|
| MP3 → AAC | Slightly worse | AAC's model applied to MP3's artifacts |
| AAC → MP3 | Worse | LAME's model on AAC's artifacts |
| MP3 → Opus | Similar or slightly worse | Opus is efficient but cannot recover MP3's lost data |
| AAC → Opus | Similar | Both modern codecs; artifacts may be comparable |
| Vorbis → MP3 | Worse | LAME's model on Vorbis's artifacts |
| MP3 → Vorbis | Similar | Vorbis is slightly more efficient but can't recover |

### Recommendation
- **Warn on all cross-codec lossy→lossy conversions.**
- Message: `"Cross-codec lossy to lossy conversion (MP3 → AAC) compounds artifacts from both codecs. Quality will be lower than either codec at the same bitrate."`

---

## 6. Encoder Chain in Tag: Shows Full Conversion History?

### What Conversion History Should Be Recorded
- Original encoder (from source file)
- Intermediate encoders (if multiple conversions have occurred)
- Final encoder (what created the output file)

### dBpoweramp's Behavior
- **Writes only the new encoder** as the `Encoder` / `TSSE` / `©too` tag.
- **Does not preserve** the source file's encoder tag.
- **Does not write** a conversion history chain.

### What Is Technically Correct
The encoder tag should reflect what **created the file** — not the history of all prior encoders.

### What Could Be Written (Optional Custom Fields)
```
ENCODER: LAME 3.100
ORIGINAL_ENCODER: LAME 3.99r
CONVERSION_HISTORY: MP3(LAME 3.99r) → MP3(LAME 3.100)
```

However, **no standard tag** exists for this conversion history. Implementations vary.

### Hydrogenaudio Community Practice
> *"If you must do lossy→lossy, you should indicate its lossy origins in the file name, if not also in tags, so that you (and anyone else using the file) will know at a glance that it's not a first-generation lossy encode."*[^1]

---

## 7. Safety Options to Prevent Unintended Lossy→Lossy

### dBpoweramp's Conditional Encoding DSP
- **Skip:** Do not convert files matching the condition.
- **Copy:** Copy the file as-is (1:1) instead of converting.
- **Conditions available:**
  - Bitrate (≥, ≤, >, <, =)
  - Extension
  - Sample rate
  - Bit depth
  - Codec type (lossy/lossless detection)

### Practical Configurations
| Safety Goal | Configuration |
|---|---|
| Prevent all lossy→lossy | Skip if codec is lossy AND target is lossy |
| Prevent MP3→MP3 | Skip if extension is `.mp3` AND target is `.mp3` |
| Prevent downsample | Skip if source bitrate > target bitrate |
| Prevent upsampling | Skip if source bitrate < target bitrate AND source bitrate is maximum for that codec |
| Allow archival | Allow only if source is lossless OR target is lossless |

### beets Convert Plugin Safety
- **`never_convert_lossy_files: yes`:** Skips all conversions from lossy sources (copies as-is).
- **`--force` / `-F`:** Overrides all safety flags to force conversion.
- **`auto_keep` (v2.6.0+):** Respects `never_convert_lossy_files` during import auto-conversion.

### FFmpeg Safety
- **No built-in safety.** User must know to use `-c:a copy` to avoid re-encoding.
- **`-c:a copy`:** Copies the audio stream without decode/re-encode — lossless, fast.
- **Warning:** FFmpeg will happily re-encode any input to any output. The user must explicitly choose not to.

---

## 8. Edge Cases

### Edge Case 1: Re-encoding a Transcoded File (3rd Generation)
- **Chain:** FLAC → MP3 128 (gen 1) → AAC 256 (gen 2) → Opus 128 (gen 3)
- **Each generation compounds artifacts.** Third-generation audio may have audible degradation compared to first-generation.
- **No special detection** beyond standard lossy→lossy warning.
- **Recommendation:** Track the generation count in a custom tag field.

### Edge Case 2: MP3 with Low Bitrate (64 kbps) → AAC at 320 kbps
- **Source:** 64 kbps MP3 (severely degraded).
- **Target:** AAC at 320 kbps.
- **Result:** AAC file is 320 kbps but sounds like 64 kbps.
- **The higher bitrate is misleading.** The quality ceiling is the 64 kbps source.
- **Warning should include:** The source bitrate and quality ceiling information.

### Edge Case 3: Converting From a Podcast / Audiobook MP3
- **Source:** Podcast at 64 kbps mono, spoken word.
- **Target:** MP3 at 192 kbps stereo.
- **Reality:** Re-encoding spoken word is less problematic than re-encoding complex music.
- **Spoken word** at 64 kbps is often acceptable; re-encoding at higher bitrate produces a slightly larger file with negligible quality improvement.
- **No special handling needed** beyond the standard warning.

### Edge Case 4: Streaming Server Transcoding (SABnzbd + SqueezeCenter)
- **Scenario:** Downloads arrive as NZB → processed by SABnzbd → served by LMS/SqueezeServer.
- **LMS transcodes on the fly:** FLAC → MP3 320 for Sonos players.
- **User-initiated conversion:** dBpoweramp batch converts all MP3s to Opus for personal library.
- **Result:** Two generations of lossy encoding (original + dBpoweramp) if the MP3 sources were already lossy.
- **Mitigation:** Use Conditional Encoding to skip lossy sources.

### Edge Case 5: Bitrate Hole in Quality
- **Scenario:** Converting MP3 128 → Vorbis q0 (≈450 kbps).
- **Result:** Vorbis file is larger but **not better** than the MP3.
- **Quality ceiling:** The MP3 128 kbps quality is the ceiling.
- **This is counterintuitive** to users who associate higher bitrate with better quality.
- **Recommendation:** Show the quality ceiling explicitly: `"Source is MP3 128 kbps. Output bitrate of 450 kbps does not improve quality. Quality ceiling: MP3 128 kbps."`

### Edge Case 6: Different Channel Configurations
- **Source:** MP3 128 kbps mono (audiobook).
- **Target:** AAC 256 kbps stereo.
- **Result:** AAC is stereo but the source was mono.
- **Channel up-mix:** The mono signal is duplicated to stereo (or processed by an up-mix algorithm).
- **The up-mix does not create stereo information** — it spreads mono content across two channels.
- **Inform the user:** `"Source is mono. Output is stereo but does not contain original stereo information."`

### Edge Case 7: Converting Between Perceptual Codecs
- **Scenario:** MP3 → ATRAC (Sony).
- **ATRAC** has a very different psychoacoustic model from MP3.
- **Result:** Significant quality loss due to codec mismatch.
- **Recommendation:** Block or strongly warn against less-common perceptual codecs.

### Edge Case 8: Hardware Player Compatibility
- **Scenario:** User has an old MP3 player that only plays MP3.
- **Source:** FLAC files in their archive.
- **They want to convert FLAC → MP3** for the player.
- **This is correct usage** of lossy→lossy: one generation from lossless, playable on the device.
- **No warning needed** if the source was lossless.

### Edge Case 9: Re-encoding for "Better" Metadata
- **Scenario:** User has MP3 files with corrupted tags. They want to re-encode to "clean up" the metadata.
- **Reality:** Re-encoding does NOT fix metadata corruption. The tags are read and written separately from the audio.
- **Proper fix:** Use a tag editor (Kid3, mp3tag) to fix the tags without re-encoding.
- **Warning:** Explain that re-encoding will not fix metadata issues.

### Edge Case 10: File Size Inflation Without Quality Gain
- **Scenario:** MP3 128 kbps (5 MB) → Opus 256 kbps (10 MB).
- **User sees:** A file that's 2x larger.
- **User expects:** Better quality.
- **Reality:** No quality improvement; double the file size.
- **The size increase is entirely wasted.**
- **Recommendation:** Show the wasted space estimate: `"This conversion will increase file size by ~5 MB without improving audio quality."`

---

## Would a User Notice Any Difference from dBpoweramp?

**Likely yes** in two areas:

1. **Warning dialog:** dBpoweramp does NOT show interactive warnings for lossy→lossy. Our implementation **should show one** — this is a critical safety improvement.
2. **Quality ceiling communication:** dBpoweramp doesn't explicitly communicate the quality ceiling. Our implementation should show: source bitrate, quality ceiling, expected output bitrate, and wasted space.
3. **Conditional Encoding defaults:** Pre-configuring the Conditional Encoding DSP with sensible defaults (e.g., "warn on MP3→MP3") would make the safety feature discoverable.

**In terms of output quality:** No difference. The re-encoded output is identical regardless of whether a warning was shown. The warning is purely for user education.

---

## References

[^1]: Hydrogenaudio Knowledgebase — "Transcoding." https://wiki.hydrogenaudio.org/index.php?title=Transcoding
[^2]: Hydrogenaudio Forum — "Same codec MP3→MP3 re-encoding tests." https://hydrogenaudio.org/index.php/topic,25671.0.html
[^3]: dBpoweramp Music Converter Help. https://dbpoweramp.com/Help/dMC/dMC
[^4]: Hydrogenaudio Forum — "foobar2000 transcoding warnings." https://hydrogenaud.io/index.php/topic,68513.0.html
[^5]: beets documentation — Convert plugin. https://docs.beets.io/en/stable/plugins/convert.html
