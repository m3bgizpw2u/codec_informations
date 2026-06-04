# 50 — Single File Conversion Pipeline Sequence
## The Master Pipeline: Complete 9-Phase Sequence

> **Scope:** This document defines the canonical single-file conversion pipeline used by dBpoweramp Music Converter. It is the master pipeline from which all other pipelines (batch, multi-encoder, lossy-to-lossless, etc.) are derived. Every conversion — single file or batch — executes this sequence. The developer reference is derived from dBpoweramp's published plugin DSP architecture[^1], official help documentation[^2], and community behavioral analysis[^3].

---

## PHASE 0: Validation

**Purpose:** Ensure the conversion can proceed before any resources are consumed.

### 0.1 — File Existence Check
- Verify the source file exists on disk.
- Verify the file is readable (not locked by another process).
- **dBpoweramp behavior:** Source files are opened read-only[^4]; the converter cannot corrupt source files even in batch mode.

### 0.2 — Format Detection by Magic Bytes
- Read the first 4–16 bytes of the file.
- Match against known magic byte signatures:
  - `FLAC`: `fLaC` (0x66 0x4C 0x61 0x43)
  - `MP3`: `0xFF 0xFB` or `0xFF 0xFA` (MPEG sync word)
  - `AAC/M4A`: `ftyp` atom (MPEG-4)
  - `WAV`: `RIFF....WAVE` chunk
  - `OGG`: `OggS`
  - `Opus`: `OpusHead` in Ogg container
  - `ALAC`: `ftyp` atom with `alac` brand
- **Do not trust file extension.** Users rename files incorrectly. Always detect by magic bytes.
- If magic bytes are unrecognized, attempt decoder-based probing as a fallback.

### 0.3 — Supported Format Check
- Verify the detected format has a registered decoder.
- Check whether the target encoder supports the detected input format.
- If no decoder exists: abort with error `"No decoder available for format: XYZ"`.

### 0.4 — Output Path Writable
- Resolve the destination path from the naming template (`[artist]`, `[album]`, etc.).
- Verify the destination directory exists; create it if absent.
- Verify the destination directory is writable.
- Verify no file already exists at the destination (or that overwrite policy allows it).

### 0.5 — Disk Space Check
- Estimate output file size from source metadata (duration × bitrate × channels / compression ratio).
- Compare against available space on the destination volume.
- If insufficient space: abort with error before starting.
- **dBpoweramp behavior:** For CD ripping, checks that output volume has space before beginning.

### 0.6 — Collision Policy
- **Skip:** Do not overwrite; log as skipped.
- **Overwrite:** Delete existing destination before writing.
- **Rename:** Append `_1`, `_2`, etc. to avoid collision.
- **Prompt:** Halt and ask the user (not applicable in batch CLI mode).
- **dBpoweramp default:** Skip if destination exists; user can configure overwrite behavior.

---

## PHASE 1: Tag Read

**Purpose:** Extract all metadata from the source file into a canonical internal representation.

### 1.1 — Open File Handle
- Open source file with read-only access.
- Do not modify the source file under any circumstances.

### 1.2 — Detect All Tag Systems Present
- Scan for all tag systems the format supports:
  - **MP3:** ID3v2.4, ID3v2.3, ID3v1, APEv2
  - **FLAC:** Vorbis Comments (primary), ID3v2 (in wrapper)
  - **M4A/AAC:** iTunes atoms (metadata atoms)
  - **WAV:** RIFF INFO, ID3v2-in-WAV, BWF metadata
  - **OGG:** Vorbis Comments
  - **Opus:** Vorbis Comments with Opus-specific fields
  - **APE:** APEv2 (native), ID3v1
  - **WMA:** ASF header metadata

### 1.3 — Priority Rules When Multiple Tag Systems Exist
- **dBpoweramp read priority (MP3):** ID3v2.4 → ID3v2.3 → APEv2 → ID3v1
- **FLAC:** Vorbis Comments only (ignore any ID3v2 wrapper)
- **M4A:** iTunes atoms only
- **WAV:** BWF `LIST` chunk → RIFF INFO → ID3v2-in-WAV
- When fields appear in multiple tag systems: prefer the richer/earlier source.

### 1.4 — Read into CanonicalTag
- Normalize all fields into a canonical internal representation:
  - `title`, `artist`, `album`, `album_artist`, `composer`, `genre`
  - `date` (parse year/month/day components from varied formats)
  - `track_number`, `track_total`, `disc_number`, `disc_total`
  - `comment`, `bpm`, `conductor`, `lyricist`, `remixer`
  - `isrc`, `catalog_number`, `barcode`, `label`, `publisher`
  - `compilation` (boolean), `album_gain`, `album_peak`, `track_gain`, `track_peak`
  - Custom/freeform fields preserved as-is
- **Track number normalization:**
  - Source stores `TRCK = "5/12"` (Vorbis) → parse to `(5, 12)`
  - Source stores `trkn` atom (MP4 uint16 pair) → parse to `(5, 12)`
  - Source stores `Track = "5/12"` (APEv2) → split on `/` → `(5, 12)`
  - **Always write the recombined form correctly per target format.**

### 1.5 — Read Cover Art
- Extract the primary cover image.
- **ID3v2 APIC picture type:** Read the `picture_type` byte. If absent, default to type 3 (Front Cover).[^5]
- **FLAC METADATA_BLOCK_PICTURE:** Decode the binary block correctly (not raw JPEG bytes in a Vorbis field).[^6]
- **MP4:** Read the `covr` atom as binary image data.
- If multiple images exist: prefer Front Cover (type 3) > Back Cover (type 4) > Other (type 0).
- Resize large images (e.g., > 3000px) to a reasonable maximum to avoid bloat, unless the user explicitly requests full-resolution embedding.

### 1.6 — Close Handle
- Release the source file handle before proceeding to Phase 2.
- The source file must not remain locked during the conversion.

---

## PHASE 2: Audio Analysis

**Purpose:** Read technical stream metadata without decoding the full audio.

### 2.1 — Stream Metadata
- Read header/metadata blocks:
  - `sample_rate`: samples per second (Hz)
  - `bit_depth`: bits per sample (16, 24, 32, float)
  - `channels`: number of audio channels
  - `duration`: total length in samples (if available) or milliseconds
  - `total_samples`: used for gapless playback calculations

### 2.2 — Codec-Specific Information
- **FLAC:** Read `STREAMINFO` block; extract `min_block_size`, `max_block_size`, `sample_rate`, `channels`, `bits_per_sample`, `total_samples`, `MD5 signature`, seek table presence.
- **MP3:** Read `Xing` or `Info` header; extract frame count, quality indicator, encoder delay/padding samples.
- **AAC/M4A:** Read `mdhd` atom; extract sample count, timescale, encoder delay/padding.
- **Opus:** Read `OpusHead` packet; extract pre-skip samples, input sample rate.
- **WAV:** Read `fmt ` chunk; extract audio format, channels, sample rate, byte rate, block align.
- **ALAC:** Read `alac` atom; extract frame size, compatibility info.

### 2.3 — Gapless Info Extraction
- For formats that use encoder delay/padding:
  - **MP3 (LAME):** Read `LAME` tag; extract `encoder_delay` and `encoder_padding` in samples.
  - **M4A/AAC:** Read `iTunSMPB` tag; extract `begin`, `end`, `orig` samples.
  - **Opus:** Read `OpusHead`; extract `pre-skip` value.
- Store these values to pass through to the output encoder.

### 2.4 — Seek Table Presence
- Check whether the source has a seek table (FLAC SEEKTABLE block).
- Note: seek tables do not survive format conversion to formats that lack native seek table support (WAV, ALAC, M4A).

---

## PHASE 3: Audio Decode

**Purpose:** Decode the source audio to a standard internal representation.

### 3.1 — Open for Decoding
- Re-open the source file (if closed after Phase 1) or maintain the existing handle.
- Initialize the format-specific decoder.

### 3.2 — Initialize Decoder
- Instantiate the decoder for the detected source format.
- Configure decoder options from the codec's advanced settings:
  - **FLAC:** `verify` mode for integrity checking; `continue_on_error` for damaged sources.
  - **MP3:** Verify frame CRCs if available.
  - **CD (cdparanoia):** Set paranoia level (0=no correction, 1=overlap, 2=full paranoia).

### 3.3 — Apply Gapless Settings
- Before decoding, skip the encoder delay samples at the start of the audio.
- After decoding, truncate the encoder padding samples at the end.
- **MP3 (LAME/Xing):** Skip first `delay` samples; decode `total_frames × frame_size − delay − padding` samples.
- **Opus:** Skip `pre-skip` samples from the decoded output.
- This ensures gapless playback when the track is encoded to a new format.

### 3.4 — Decode to PCM
- Decode the entire audio stream (or stream in chunks for very large files) into raw PCM frames.
- **Internal standard format:** 32-bit float, planar (separate channel arrays), interleaved on write.
- Handle signed/unsigned, little/big-endian, and integer/float conversions from source.
- Normalize sample values to the [-1.0, 1.0] floating-point range.

### 3.5 — Validate PCM Levels
- After decoding: scan the PCM buffer for clipping (samples at or beyond ±1.0).
- If clipping detected: optionally apply a small gain reduction (e.g., -0.5 dB) or warn the user.
- Report peak level for DSP chain reference.

---

## PHASE 4: DSP Chain

**Purpose:** Apply any user-requested audio processing to the decoded PCM before encoding.

> **Note:** This phase is optional. If no DSP effects are enabled, Phase 4 is a no-op (pass-through).

### 4.1 — Sample Rate Conversion (SRC)
- If the source sample rate differs from the target sample rate:
  - Select an appropriate resampling algorithm.
  - **Best quality:** SoX's VHQ algorithm, Secret Rabbit Code (libsamplerate), or ffmpeg's `soxr` resampler.
  - **Minimum:** Linear interpolation (avoids pre-ringing but introduces some HF rolloff).
  - **dBpoweramp:** Uses its own DSP engine with configurable quality levels.
- Resampling must not introduce pre-ringing artifacts at the start of audio.
- Store the new sample rate in the stream metadata.

### 4.2 — Bit Depth Processing
- If reducing bit depth (e.g., 24-bit → 16-bit):
  - **DITHER:** Add low-level triangular noise (TPDF or shaped) before truncation. This is mandatory for professional quality at 16-bit output.[^7]
  - **Do NOT simply truncate** — this produces harmonic distortion on low-level signals.
  - **Noise shaping** (optional): Use Shibata, UV22, or similar algorithm to push quantization noise into less audible frequency ranges.
  - **Dither only once:** If converting 24-bit → 32-bit float → 16-bit, apply dither only at the final truncation step.
- If increasing bit depth (e.g., 16-bit → 24-bit):
  - Zero-pad the lower bits (no dither needed).
- If no bit depth change: pass through unchanged.

### 4.3 — Channel Up-mix / Down-mix
- If source has more channels than target (e.g., 5.1 → stereo):
  - Apply a proper down-mix matrix (Dolby Pro Logic, ITU-R BS.775, etc.).
  - Apply appropriate gain adjustments for rear channels.
  - Apply bass management if configured.
- If source has fewer channels than target (e.g., stereo → 5.1):
  - Apply an up-mix algorithm (duplicate, ambient extraction, or matrix-based).
  - Flag as processed audio (not original).

### 4.4 — Volume Normalization
- **ReplayGain 2.0 (EBU R128):** Scan the PCM buffer; compute integrated loudness (LUFS), true peak, loudness range (LRA); apply gain to reach target level (-18 LUFS for EBU R128 or -18 dBTP for track normalization).[^8]
- **Peak normalization:** Scale to a target peak level (e.g., -0.1 dBFS).
- **dBpoweramp:** Supports ReplayGain via the ReplayGain DSP effect. Also supports EBU R128 mode.

### 4.5 — Equalizer
- Apply parametric EQ curves if configured.
- **IIR filters:** For gentle EQ; low computational cost.
- **FIR filters:** For linear-phase EQ; better for critical listening.

### 4.6 — DC Offset Removal
- Detect and remove DC offset (sustained non-zero mean in the waveform).
- DC offset can cause issues with some encoders and playback systems.

### 4.7 — Validate PCM Levels (Post-DSP)
- Re-scan the processed PCM for clipping after all DSP effects.
- If clipping: apply a brick-wall limiter or gain reduction.

---

## PHASE 5: Audio Encode

**Purpose:** Encode the processed PCM to the target format.

### 5.1 — Initialize Encoder
- Instantiate the encoder for the target format.
- Configure encoder parameters from the codec's settings:
  - **MP3:** bitrate (CBR/VBR/ABR), quality, LAME tag settings
  - **AAC:** bitrate, HE-AAC/HE-AACv2, VBR/CBR
  - **FLAC:** compression level (0–12), verify mode
  - **Opus:** bitrate, VBR/CBR, application (audio/music/voice)
  - **WAV:** PCM format parameters

### 5.2 — Feed PCM Frames
- Write processed PCM frames to the encoder in order.
- The encoder may accumulate frames for encoding in blocks.

### 5.3 — Write to Temp File
- **CRITICAL:** Write encoded output to a temporary file (`.tmp` extension) in the same directory as the destination.
- Do NOT write directly to the final destination path.
- This ensures atomic replacement on completion (Phase 9).

### 5.4 — Flush Encoder
- Signal end-of-input to the encoder.
- The encoder writes any remaining buffered data.

### 5.5 — Finalize
- The encoder completes the bitstream.
- For formats that require a finalization step (FLAC: write MD5 signature; M4A: write size atoms; MP3: finalize Xing header):
  - Rewind or update the temp file with final values.

### 5.6 — Write Seek Index / TOC
- For formats that support seek tables:
  - **FLAC:** Write a SEEKTABLE block (default: one seekpoint per 10 seconds).
  - **MP3:** Xing/Info header written with final frame count.
- For formats without seek tables (WAV, ALAC): no action needed.

---

## PHASE 6: Tag Write

**Purpose:** Write the processed metadata to the encoded output file.

### 6.1 — Map CanonicalTag to Target Format
- Convert the canonical internal tag representation to the target format's tag system.
- Use the correct field names and value formats per target:
  - **ID3v2:** `TIT2` (title), `TPE1` (artist), `TALB` (album), `TDRC` (date, v2.4) / `TYER`+`TDAT` (date, v2.3), `TRCK` (track, format: `"5/12"`), `TPOS` (disc), `TCON` (genre, freeform string, no `(N)` prefix), `COMM` (comment, language: `XXX`), `APIC` (picture, type 3), `TSSE` (encoder)
  - **Vorbis Comment (FLAC/OGG):** `TITLE`, `ARTIST`, `ALBUM`, `DATE`, `TRACKNUMBER`, `TRACKTOTAL`/`TOTALTRACKS`, `DISCNUMBER`, `DISCTOTAL`, `GENRE`, `COMMENT`, `COVERART`, `REPLAYGAIN_TRACK_GAIN`, `REPLAYGAIN_TRACK_PEAK`, `REPLAYGAIN_ALBUM_GAIN`, `REPLAYGAIN_ALBUM_PEAK`
  - **MP4 iTunes atoms:** `©nam`, `©ART`, `©alb`, `©day`, `trkn`, `disk`, `©gen`, `©cmt`, `covr`, `©too`
  - **RIFF INFO (WAV):** `INAM`, `IART`, `IPRD`, `ICRD`, `ITRK`, `IGNR`, `ICMT`, `ISFT`

### 6.2 — Apply Transformations
- **Track number:** Combine `(5, 12)` → write `"5/12"` to ID3, `"5"` + `"12"` to Vorbis `TRACKNUMBER`/`TOTALTRACKS`, uint16 pair to MP4 `trkn`.
- **Genre:** Write freeform string only (no `(N)` prefix for ID3v2.3+).
- **Date:** Write fullest representation available; for ID3v2.3: write `TYER` + `TDAT` + `TIME` if month/day/time available.
- **Encoder tag:** Write the name/version of the new encoder (e.g., `"LAME 3.100"`, `"FLAC 1.4.0"`, `"dBpoweramp Music Converter"`).[^9]
- **ReplayGain tags:** Convert to the correct format per target (see Section 6 in the Quick Reference).[^10]

### 6.3 — Handle Unmappable Fields
- If a source field has no equivalent in the target format:
  - Drop the field silently (unless it's a custom/user-defined field).
  - Log custom fields that could not be mapped.
- If a target format cannot store a required field: skip it.

### 6.4 — Resize Cover Art
- If the cover art exceeds a maximum size configured by the user:
  - Resize the image to the maximum dimensions (e.g., 3000px longest edge).
  - Re-encode to a reasonable quality (e.g., JPEG at 90% quality, or PNG).
- Strip EXIF metadata from the cover image to reduce file size.

### 6.5 — Write Tags to Temp File
- Open the temp file for writing (or appending after the audio data, depending on format).
- Write all tag fields in the correct byte order and encoding.
- **ID3v2:** Write at the beginning of the file (before audio frames).
- **Vorbis Comment:** Write as the first metadata block (before audio data).
- **MP4:** Write metadata atoms; `moov` atom placement depends on optimization settings.
- **RIFF INFO:** Write after `data` chunk; ID3v2-in-WAV at the beginning.
- **FLAC:** Write metadata blocks before the audio frames; write `METADATA_BLOCK_PICTURE` as a properly formatted binary block.

### 6.6 — Write Cover Art
- Write the cover image as the appropriate frame/atom:
  - **ID3v2:** `APIC` frame with `picture_type = 3` (Front Cover), `mime = "image/jpeg"` or `"image/png"`.
  - **FLAC:** `METADATA_BLOCK_PICTURE` (properly formatted binary block, not raw JPEG bytes in a Vorbis field).
  - **MP4:** `covr` atom with image data (JPEG or PNG).
  - **Vorbis:** `COVERART` (base64-encoded) + `COVERARTMIME` fields, or `METADATA_BLOCK_PICTURE` (preferred, properly formatted).

---

## PHASE 7: ReplayGain Scan

**Purpose:** Compute and embed loudness normalization tags in the output file.

> **Note:** This phase only executes if ReplayGain is enabled in the conversion settings. It may run before or after tag write; dBpoweramp supports both embedded ReplayGain tags and DSP-based real-time normalization.

### 7.1 — Decode Output for Analysis (if not already PCM)
- If ReplayGain scan is requested and the DSP chain did not already produce the PCM:
  - Open the newly encoded file.
  - Decode it back to PCM.

### 7.2 — Run EBU R128 / ReplayGain Algorithm
- **EBU R128 (preferred):**
  - Compute integrated loudness (LUFS) using ITU-R BS.1770-4 / EBU R128.
  - Compute true peak in dBTP.
  - Compute loudness range (LRA) in LU.
- **ReplayGain 1.0 (legacy):**
  - Compute RG gain in dB.
  - Compute peak amplitude as a linear value (0.0–1.0) or dB.

### 7.3 — Compute Gain and Peak
- For **track gain:** target level = -18 LUFS (EBU R128) or -18 dBFS (ReplayGain).
- For **album gain:** compute integrated loudness of all tracks in the album; apply uniform gain.
- Compute the peak value (maximum absolute sample amplitude) in dBFS or linear.

### 7.4 — Update Tags
- Write the computed values to the output file's tags:
  - **FLAC/Vorbis:** `REPLAYGAIN_TRACK_GAIN`, `REPLAYGAIN_TRACK_PEAK`, `REPLAYGAIN_ALBUM_GAIN`, `REPLAYGAIN_ALBUM_PEAK`, `REPLAYGAIN_REFERENCE_LOUDNESS` (= "89 dB" for RG 1.0)
  - **MP3:** TXXX frames with descriptions `REPLAYGAIN_TRACK_GAIN`, `REPLAYGAIN_TRACK_PEAK`, etc.
  - **MP4:** `----:com.apple.iTunes:REPLAYGAIN_TRACK_GAIN`, etc.
- **dBpoweramp:** The ReplayGain DSP effect can apply the gain during encoding or embed it as tags. The "Scan then Apply" option does both.

### 7.5 — Album Mode (Batch ReplayGain)
- For batch conversions with album mode:
  - Compute per-track RG values first.
  - After all tracks are encoded, compute album-level RG values.
  - Update all track files with album gain/peak values.
- This requires re-opening the encoded files and updating their tags.

---

## PHASE 8: Verification

**Purpose:** Confirm the output file is valid before finalization.

### 8.1 — Re-decode Header
- Open the temp output file.
- Read the header/metadata blocks.

### 8.2 — Verify Duration
- Decode the first few frames and the last few frames.
- Verify the duration matches the expected value (within tolerance for lossy formats).
- For lossless formats: duration should be exact.

### 8.3 — Compute MD5 for Lossless
- For lossless-to-lossless conversions:
  - Compute the MD5/SHA256 of the decoded PCM.
  - Compare against the source's embedded MD5 signature (FLAC `STREAMINFO`).
  - If mismatch: flag as potential issue.
- **FLAC:** Verify the embedded MD5 signature by decoding and hashing.
- **dBpoweramp:** Does not appear to have an automated bit-exact verification step, but the FLAC encoder embeds and verifies the MD5 by default.

### 8.4 — Verify Tags
- Re-read the written tags from the temp file.
- Confirm that all expected fields were written correctly.
- Confirm cover art is present and intact.

### 8.5 — Check for Errors
- If any verification step fails:
  - Do NOT rename the temp file to the final destination.
  - Log the error with details.
  - Delete the temp file.
  - Report failure to the caller.

---

## PHASE 9: Finalization

**Purpose:** Atomically commit the output file and clean up.

### 9.1 — Atomic Rename
- **CRITICAL:** Use an atomic rename operation: `rename(temp_path, final_path)`.
- On POSIX: `rename()` is atomic when source and destination are on the same filesystem.
- On Windows: Use `MoveFileEx()` with `MOVEFILE_REPLACE_EXISTING` or use a rename-and-delete approach.
- **Do NOT** copy then delete — this leaves the destination vulnerable to corruption if the copy is interrupted.

### 9.2 — Set File Modification Time (mtime)
- Set the output file's `mtime` to match the source file's `mtime`.
- Optionally set the `atime` (last accessed time) to the current time.
- dBpoweramp preserves the source's modification time on the output by default.

### 9.3 — Log Result
- Log the successful conversion:
  - Source path, destination path, duration, output file size.
  - Encoder name and settings used.
  - DSP effects applied (if any).
  - Elapsed time.

### 9.4 — Update Progress
- Report completion to the UI/batch controller.
- Include any warnings (e.g., clipping detected, fields dropped, cover art resized).

---

## Edge Cases

### Edge Case 1: Source File Disappears Mid-Conversion
- **Phase 0:** File exists and is readable.
- **Phase 3:** Source handle is open; if the file is deleted externally, the decode loop detects an I/O error.
- **Behavior:** Abort conversion; delete temp file; report error. Do not create an output file.

### Edge Case 2: Encoder Produces Larger Output Than Source
- **Phase 5:** This is expected for lossless-to-lossless conversions where the target format has less efficient compression.
- **Behavior:** Proceed. File size difference is informational only.

### Edge Case 3: No Cover Art in Source
- **Phase 1:** Cover art extraction finds nothing.
- **Phase 6:** No cover art is written to the output.
- **Behavior:** Proceed without cover art. Do not add a placeholder.

### Edge Case 4: Source Has Hybrid Lossless (DSD/SACD)
- **Phase 2:** Some SACD rippers output DSD as DFF/DST. These are not natively supported by most decoders.
- **Behavior:** Fail gracefully with an error indicating no decoder is available.

### Edge Case 5: Very Long Source Files (e.g., 8-hour live recordings)
- **Phase 3:** Decoding requires buffering the entire stream.
- **Phase 4:** DSP operations operate on the full PCM buffer.
- **Phase 5:** Encoder processes the full buffer.
- **Behavior:** Memory usage = `duration × sample_rate × channels × 4 bytes (float)`. A 24-hour 96kHz/5.1 FLAC would require ~250 GB — impractical. **Chunked processing** (decode/encode in chunks) is needed for very long files.

### Edge Case 6: Source File Encoding Errors (Corrupt Audio Frames)
- **Phase 3:** Decoder encounters an error frame.
- **Behavior:** If `continue_on_error` is set: skip the corrupt frame and continue. If not set: abort. The `Delete Destination File on Error` DSP effect removes the output file if conversion fails.

### Edge Case 7: User Cancellation Mid-Conversion
- **Phase 3–5:** User cancels the operation.
- **Behavior:** Stop the decode/encode loop immediately. Delete the temp file. Report cancellation. Do NOT leave a partial output file.

### Edge Case 8: Destination Drive Full Mid-Write
- **Phase 5:** Encoder runs out of disk space while writing.
- **Behavior:** Stop encoding. Delete the temp file. Report error `"Insufficient disk space"`. Do not produce a partial output file.

### Edge Case 9: Concurrent Access to Same Output File (Batch Collision)
- **Phase 0:** Two files in a batch resolve to the same destination path.
- **Behavior:** With default collision policy (skip), the second file is skipped. With overwrite policy, the second overwrites the first. The pipeline must detect this at Phase 0, not at Phase 9.

### Edge Case 10: Format Supports No Tags (RAW PCM)
- **Phase 6:** No tag system available in the target format.
- **Behavior:** Skip tag writing. All metadata is lost unless it is encoded in the filename (via naming template).

---

## Would a User Notice Any Difference from dBpoweramp?

**Likely no** — for the core pipeline phases. dBpoweramp's pipeline is well-established and these phases align with documented dBpoweramp behavior. Key areas where an implementation would closely match dBpoweramp:

1. **Tag field mapping** — dBpoweramp is known for comprehensive tag field support across all major formats. Our CanonicalTag approach maps all the fields dBpoweramp handles.
2. **Cover art picture type** — dBpoweramp correctly writes type 3 (Front Cover). Our implementation must do the same.
3. **ReplayGain** — dBpoweramp's ReplayGain DSP supports both EBU R128 and legacy ReplayGain modes. Our Phase 7 implementation covers both.
4. **Atomic rename** — dBpoweramp uses temp files and rename. Our Phase 9 follows this exactly.

**Possible user-noticeable differences:**
- **DSP quality:** dBpoweramp's DSP engine has tuned algorithms; a naive SRC or EQ implementation could sound different.
- **Error recovery:** dBpoweramp's `continue_on_error` and `Delete Destination File on Error` behaviors are nuanced; exact reproduction requires careful attention to decoder options.
- **Tag read priority** when multiple tag systems coexist (e.g., ID3v2 + APEv2 in MP3) — exact priority matching requires reverse-engineering dBpoweramp's specific behavior.

---

## References

[^1]: dBpoweramp DSP Architecture. Spoon (dBpoweramp developer). https://forum.dbpoweramp.com/forum/other-topics/developers-corner/22043-music-converter-dsp-architecture
[^2]: dBpoweramp Music Converter Help. https://dbpoweramp.com/Help/dMC/dMC
[^3]: dBpoweramp Community Forum discussions on conversion pipeline behavior.
[^4]: dBpoweramp Forum — "Original files corrupted after batch convert." https://forum.dbpoweramp.com/forum/dbpoweramp/batch-converter/39962-original-files-corrupted-after-batch-convert
[^5]: dBpoweramp Metadata Behavior Quick Reference. See Section 3: Cover Art Type Code.
[^6]: FLAC format overview. Xiph.org. https://xiph.org/flac/documentation_format_overview.html
[^7]: AudioFanzine — "Much Ado About Dithering." https://en.audiofanzine.com/mastering/editorial/articles/much-ado-about-dithering.html
[^8]: EBU R128 loudness normalization. https://tech.ebu.ch/r128
[^9]: dBpoweramp Metadata Behavior Quick Reference. See Section 8: Encoder Tag.
[^10]: dBpoweramp Metadata Behavior Quick Reference. See Section 6: ReplayGain Tag Names.
