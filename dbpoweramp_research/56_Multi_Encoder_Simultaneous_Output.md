# 56 — Multi-Encoder Simultaneous Output

> **Scope:** How dBpoweramp handles encoding to multiple formats simultaneously in a single conversion pass. Focus on the Multi Encoder feature, decode-once/encode-multiple architecture, per-encoder settings, file naming, and performance implications. Primary sources: dBpoweramp official documentation[^1][^2], dBpoweramp forum discussions[^3], and fre:ac meh! multi-encoder hub documentation[^4].

---

## 1. Overview: Multi-Encoder Architecture

dBpoweramp's **Multi Encoder** feature allows encoding a single source file to **two or more output formats simultaneously** in a single conversion pass. For example, a user can rip a CD to both FLAC (archival) and MP3 (portable) in one operation — without having to run two separate conversions.

This is fundamentally different from running two separate conversions (which would decode the source twice). Multi Encoder **decodes once** and **encodes multiple times**.

### The "One Decode, Multiple Encode" Principle
```
[Source File]
      ↓
[Decode to PCM — ONCE]
      ↓
      ├──────────────────┬──────────────────┐
      ↓                  ↓                  ↓
[Encoder 1: FLAC]  [Encoder 2: MP3]  [Encoder 3: AAC]
      ↓                  ↓                  ↓
[output1.flac]     [output2.mp3]     [output3.m4a]
```

This architecture is efficient because:
- **Disk I/O is the bottleneck for decoding**, not CPU.
- Decoding is done once, amortizing the I/O cost across all outputs.
- Encoding is CPU-bound and parallelizable.

---

## 2. How Does dBpoweramp Handle Multiple Encoders?

### From the Official Documentation[^1]
- Set the encoder to **Multi Encoder**.
- Click **Add Encoder** to add as many codecs as needed.
- Each encoder has its own **Output To** (destination path) and **naming configuration**.
- The `Multicore` option allows Multi Encoder to use multiple CPU cores at the same time.
- **Source file must not be overwritten** — leads to unpredictable results.

### Per-Encoder Settings
Each encoder in the Multi Encoder can be configured **independently**:

| Setting | Per-Encoder? | Notes |
|---|---|---|
| **Output format** | Yes | FLAC, MP3, AAC, Opus, etc. |
| **Encoder settings** | Yes | Bitrate, quality mode, VBR/CBR, etc. |
| **Output path** | Yes | Can be different folders or same folder |
| **Naming template** | Yes | Different patterns per encoder |
| **DSP effects** | Yes | Per-encoder DSP chain |
| **Tag actions** | Yes | Per-encoder tag handling |

### Example Configuration
| Encoder | Format | Bitrate | Output Folder |
|---|---|---|---|
| Encoder 1 | FLAC | Level 5 | `/archive/flac/` |
| Encoder 2 | MP3 | 320 kbps CBR | `/portable/mp3/` |
| Encoder 3 | AAC | 256 kbps | `/portable/aac/` |

### Dynamic Naming Per Encoder
- Each encoder's naming template can be different.
- Common use case: use the same template structure but different output folders.
- Example: `[albumartist]/[album]/[track] - [title]` → all three encoders use the same template but write to their own folders.

---

## 3. Does It Decode Once and Encode Multiple Times?

**Yes.** This is the defining architectural advantage of Multi Encoder.

### Phase-by-Phase Breakdown

**Phase 3 (Audio Decode):** Performed once.
- The source file is decoded to PCM (32-bit float planar).
- The PCM buffer is held in memory.

**Phase 5 (Audio Encode):** Performed once per encoder.
- Encoder 1: Receives PCM → produces FLAC output.
- Encoder 2: Receives PCM → produces MP3 output.
- Encoder 3: Receives PCM → produces AAC output.

**Phase 6 (Tag Write):** Performed once per encoder.
- Each encoder's tag writer maps CanonicalTag to the target format.
- Cover art may be resized differently per encoder (if configured).

### Memory Efficiency
- **PCM buffer:** ~10 MB per minute of audio at 44.1kHz/stereo/16-bit.
- For a 5-minute track: ~50 MB PCM buffer.
- This 50 MB is shared across all encoders.
- **Memory usage with 3 encoders:** ~50 MB (PCM) + ~5 MB (encoder state per encoder) = ~65 MB total.
- **Without Multi Encoder:** 3 separate conversions = 3 × ~55 MB = ~165 MB total.

### CPU Efficiency
- Encoding is CPU-bound.
- If **Multicore** is enabled, encoders run in parallel on multiple CPU cores.
- Encoding time for a 3-output Multi Encoder = max(encode_time_per_encoder), not sum.

---

## 4. Per-Encoder Settings: Can Each Output Have Different Encoding Parameters?

**Yes.** This is one of the key strengths of Multi Encoder.

### Encoder-Specific Parameters
| Encoder | Parameters |
|---|---|
| **FLAC** | Compression level (0–12), verify mode |
| **MP3 (LAME)** | Bitrate (CBR/VBR/ABR), quality (V0–V9), LAME tag settings |
| **AAC (QAAC)** | Bitrate, HE-AAC/HE-AACv2, VBR/CBR, Apple AudioVBR |
| **Opus** | Bitrate (VBR/CBR), application (audio/music/voice), complexity |
| **Vorbis** | Quality level (q-1 to q10), managed VBR |
| **WMA** | Bitrate, quality mode |

### DSP Effects Per Encoder
- **Encoder 1 (FLAC):** Apply ReplayGain scan only (no DSP effects).
- **Encoder 2 (MP3):** Apply ReplayGain scan + volume normalization DSP.
- **Encoder 3 (AAC):** Apply volume normalization DSP only.
- Each encoder's DSP chain is **configured independently**.

### Tag Actions Per Encoder
- **Encoder 1 (FLAC):** Write all tags including ReplayGain.
- **Encoder 2 (MP3):** Write all tags except ReplayGain (MP3 player doesn't support it).
- **Encoder 3 (AAC):** Strip all custom tags; write only standard tags.

---

## 5. How Are Multiple Output Files Named?

### dBpoweramp's Dynamic Naming
- dBpoweramp uses a **programmable naming template** system.
- Placeholders are replaced with ID Tag values:
  - `[artist]` → artist name from tag
  - `[album]` → album name
  - `[track]` → track number (padded)
  - `[title]` → track title
  - `[albumartist]` → album artist
  - `[year]` → year
  - `[genre]` → genre
  - `[bitrate]` → output bitrate
  - `[origdrive]` → source drive letter (for Music Converter only, not CD Ripper)
  - `[number of bits]` → bit depth
  - `[sampling rate]` → sample rate
  - `[kbps]` → kilobits per second

### Common Naming Patterns
```
# Archive + Portable split
/archive/[albumartist]/[album]/[track] - [title].flac
/portable/[albumartist]/[album]/[track] - [title].mp3

# All in same folder with format identifier
[albumartist] - [album]/[track] - [title] (FLAC).flac
[albumartist] - [album]/[track] - [title] (MP3 320).mp3
[albumartist] - [album]/[track] - [title] (AAC 256).m4a

# Per-format subfolder
FLAC/[albumartist] - [album]/[track] - [title].flac
MP3/[albumartist] - [album]/[track] - [title].mp3
AAC/[albumartist] - [album]/[track] - [title].m4a
```

### The `<<filetype>>` Placeholder (fre:ac)
fre:ac's meh! component supports `<<filetype>>` placeholder for automatic folder separation:[^4]
```
<<filetype>>\\<<albumartist>> - <<album>>\\<<track>> - <<artist>> - <<title>>
```
This creates `FLAC/` and `MP3/` top-level folders automatically.

### Collision Prevention
- If two encoders write to the same path: **the second encoder fails**.
- dBpoweramp's UI prevents configuring two encoders with identical output paths.
- The user must ensure each encoder has a unique destination path.

---

## 6. Performance Implications

### Encoding Speed
- With **Multicore disabled:** Encoders run sequentially. Total time = sum of each encoder's encode time.
- With **Multicore enabled:** Encoders run in parallel on separate CPU cores. Total time = max of each encoder's encode time.
- For example: FLAC encoding (CPU-light) + MP3 320 kbps (CPU-heavy) + AAC 256 kbps (CPU-heavy):
  - Sequential: 10s + 30s + 25s = 65s
  - Parallel (3 cores): max(10s, 30s, 25s) = 30s

### CPU Utilization
- **Multicore ON:** Near-full CPU utilization across all configured cores.
- **Multicore OFF:** One CPU core at high utilization; others idle.
- **Optimal setting:** Enable Multicore when encoding to multiple formats simultaneously.

### Disk I/O
- **Disk read (decode):** Performed once. Single sequential read of source file.
- **Disk write (encode):** Performed once per encoder. Parallel writes to different destinations.
- **Total disk throughput:** Sum of read + all write throughputs. Disk I/O is typically not the bottleneck for this workload.

### Memory Usage
- PCM buffer shared across all encoders (one copy in memory).
- Each encoder maintains its own encoder state (~1–5 MB per encoder).
- Total memory ≈ PCM buffer + (N × encoder state).

### Comparison: Multi Encoder vs. Sequential Conversions
| Metric | Multi Encoder (3 formats) | Sequential (3 passes) |
|---|---|---|
| Decode operations | 1 | 3 |
| Total disk read | 1× source | 3× source |
| CPU time | ~max(enc1, enc2, enc3) | sum(enc1, enc2, enc3) |
| Wall clock time (Multicore ON) | ~max encode time | sum encode time |
| Wall clock time (Multicore OFF) | sum encode time | sum encode time |
| Memory | 1× PCM + N× state | 1× PCM + 1× state |

**Conclusion:** Multi Encoder is more efficient than sequential conversions primarily because it **eliminates redundant decoding**. For large libraries, this is a significant time savings.

---

## 7. Edge Cases

### Edge Case 1: One Encoder Fails Mid-Conversion
- **Scenario:** Encoder 2 (MP3) fails due to a bitrate compatibility issue.
- **Encoders 1 and 3:** Continue and complete successfully.
- **Result:** Two output files created; one failed.
- **Batch summary:** Reports partial success.
- **No rollback** of successful outputs.

### Edge Case 2: Insufficient Disk Space for One Encoder
- **Scenario:** Encoders 1 and 2 write to a volume with 500 MB free. Encoder 3 needs 600 MB.
- **Encoders 1 and 2:** Write successfully.
- **Encoder 3:** Fails with disk full error.
- **Result:** Two output files; one failed due to disk space.

### Edge Case 3: Very Large Cover Art
- **Source:** FLAC with 10 MB embedded cover art.
- **Encoder 1 (FLAC):** Cover art copied as-is (10 MB).
- **Encoder 2 (MP3):** Cover art resized to 500 KB JPEG.
- **Encoder 3 (AAC):** Cover art resized to 200 KB JPEG.
- **Total disk write:** 10 MB + 0.5 MB + 0.2 MB = 10.7 MB (vs. 30 MB if each encoder kept full-resolution art).

### Edge Case 4: Different DSP Chains Per Encoder
- **Scenario:** Encoder 1 (FLAC) has no DSP; Encoder 2 (MP3) has ReplayGain applied.
- **Phase 4 (DSP):** DSP is applied once to the PCM buffer.
- **Result:** The same processed PCM is fed to both encoders.
- **BUT:** If Encoder 2 requires different DSP (e.g., MP3 needs normalization but FLAC doesn't), the DSP must be encoder-specific.
- **dBpoweramp behavior:** Each encoder has its own DSP chain. If Encoder 2 has DSP and Encoder 1 doesn't, the **same PCM** is fed to both — but Encoder 2's DSP settings are applied before Encoder 2's encoding. Encoder 1 gets the unprocessed PCM.

### Edge Case 5: Per-Encoder ReplayGain
- **Scenario:** Encoder 1 (FLAC) embeds ReplayGain tags; Encoder 2 (MP3) does not.
- **Phase 7:** ReplayGain scan is performed per-encoder if enabled.
- **Result:** FLAC file has ReplayGain tags; MP3 file does not.

### Edge Case 6: Batch Multi-Encoder
- **Scenario:** 100 files in batch, each converted to FLAC + MP3 simultaneously.
- **Queue:** Each file runs through Multi Encoder (2 outputs per file).
- **Concurrency:** 16 concurrent `CoreConverter.exe` processes × 2 encoders each = up to 32 encoder instances.
- **Resource management:** With Multicore enabled per encoder, CPU core allocation becomes complex. dBpoweramp manages this internally.

### Edge Case 7: Same Output Folder for Multiple Encoders
- **Scenario:** User configures both FLAC and ALAC encoders to write to `/music/`.
- **Naming templates are the same.**
- **Result:** Both encoders try to write to the same filename. Collision.
- **Prevention:** dBpoweramp prevents this at configuration time by requiring unique destination paths.

### Edge Case 8: Encoder-Specific Format Limitations
- **Scenario:** Encoder 1 (WMA) requires Windows Media Player to be installed.
- **Result:** If WMP is not installed, WMA encoder fails; other encoders continue.
- **Error message:** "Encoder not available. Install Windows Media Player."

### Edge Case 9: Changing Filename Placeholder Mid-Conversion
- **Scenario:** The `[bitrate]` placeholder resolves to `320` for MP3 and `1411` for FLAC.
- **Each encoder:** Uses its own resolved name.
- **Result:** No collision; different filenames.

### Edge Case 10: Network Path as Output Destination
- **Scenario:** Encoders 1, 2, 3 all write to a network share (SMB/NFS).
- **Result:** Network throughput is shared across all three encoders.
- **Bottleneck:** Network bandwidth may limit parallel write performance.
- **Mitigation:** Use local disk as intermediate; move files afterward.

---

## Would a User Notice Any Difference from dBpoweramp?

**Likely no** for the core Multi Encoder functionality. dBpoweramp's implementation is well-documented and the architecture is clear.

**Possible differences:**
1. **Per-encoder DSP chains:** Our implementation should support independent DSP configuration per encoder.
2. **Naming template features:** Our placeholder system should support all the placeholders dBpoweramp supports (especially `[kbps]`, `[number of bits]`, `[sampling rate]`).
3. **Multicore parallelization:** The `Multicore` option is a significant performance feature. Our implementation should support parallel encoding when multiple cores are available.
4. **`<<filetype>>` equivalent:** fre:ac's `<<filetype>>` placeholder for automatic format folders is a useful feature we should include.

---

## References

[^1]: dBpoweramp Music Converter Help — Multi Encoder. https://dbpoweramp.com/Help/dMC/dMC
[^2]: dBpoweramp Help — DSP effects. https://dbpoweramp.com/Help/dMC/dsp
[^3]: dBpoweramp Forum — Multi Encoder discussions. https://forum.dbpoweramp.com/
[^4]: fre:ac Developer Blog — meh! multi encoder hub. https://github.com/enzo1982/freac/blob/master/i18n/freac/freac.xml
