# 51 — Batch Conversion Architecture and Threading

> **Scope:** How dBpoweramp handles multiple files in a batch conversion: threading model, queue management, concurrency limits, memory management, error handling, and batch-level operations. Primary sources: dBpoweramp official documentation[^1][^2], community forum discussions[^3][^4][^5], and comparison with fre:ac[^6].

---

## 1. Overview: How dBpoweramp Handles Multiple Files

dBpoweramp Batch Converter dispatches each file to a **separate `CoreConverter.exe` process**. Each process runs the full single-file pipeline (Phases 0–9 from Document 50) independently. The Batch Converter maintains a queue of files and manages the lifecycle of each `CoreConverter.exe` instance.

This is **process-level parallelism**, not thread-level. Each worker is an isolated Windows process with its own memory space.

> **Key design choice:** Parallelism is achieved by running **multiple independent conversion processes simultaneously**, not by parallelizing within a single conversion. The codec itself does not parallelize across CPU cores within a single file.

---

## 2. Number of Worker Threads (Default and Configurable)

### Default Behavior
- dBpoweramp uses **automatic core detection**: it queries the system and uses the maximum number of available CPU cores by default.
- However, for **high disk I/O formats** (e.g., WAV), dBpoweramp **defaults to 1 core** because disk becomes the bottleneck — not CPU. Spinning disks cannot serve multiple concurrent read streams efficiently.

### Maximum Concurrency
- **Hard cap: 16 concurrent tracks** (16 `CoreConverter.exe` processes maximum).[^4]
- Source: "16 cores are the maximum dBpoweramp will process currently."

### User Configurability
Two mechanisms to control concurrency:

**Method 1 — Per-Session (Options dialog):**
- `Options >> Encoding` → select number of cores to use.
- Affects only the current queue session.
- Setting this to 6 cores on a 6-core machine achieves near-full CPU utilization.

**Method 2 — Permanent (DSP Effect):**
- Add the **Multi-CPU Force DSP** effect to the DSP chain.
- Configure the number of cores (1–16) in the DSP settings.
- This forces a specific core count for every conversion, regardless of session.
- The DSP applies the `SetThreadAffinityMask()` API to pin encoder threads.

**Method 3 — Per-Processor Scheduling:**
- `Options >> Encoding >> Do Not Ally to Processor` — allows encoder threads to jump between assigned CPUs, useful when the number of selected cores is less than the total available.
- Can improve throughput when background processes are using some cores.

> **Important performance note:** The **Multi-CPU Force DSP is significantly slower** than manually setting `Options >> Encoding → Using X cores`. Benchmarks show the DSP caps at ~80x encoding speed; the manual option achieves ~250x on the same hardware.[^5]

---

## 3. Queue Management

### Sequential vs. Concurrent
- The queue is **concurrent**, not sequential.
- Files are dispatched to `CoreConverter.exe` instances as cores become available.
- The Batch Converter maintains a running list: completed, current, queued, errored.

### Queue Lifecycle
```
[Add files to queue]
       ↓
[Batch Converter assigns files to available cores]
       ↓
[CoreConverter.exe #1 → Phases 0-9 for file 1]
[CoreConverter.exe #2 → Phases 0-9 for file 2]
[CoreConverter.exe #3 → Phases 0-9 for file 3]
       ...
       ↓
[On completion: next file from queue assigned to free core]
       ↓
[All files processed → Batch summary displayed]
```

### Source Code Reference (Plugin Architecture)
From the official DSP architecture spec:[^1]

```
Loop For All Files:
    dMC - unique batch ID created
    Encoders Created → Encoder Freed
    -- For Each File (CoreConverter) --
        CoreConverter Instance Called
            Decoder Loaded, Object Created
            DSP Loaded, Object Created
            Encoder Loaded, Object Created
            [Encode Loop: Decode → DSP → Encode]
            Encoder EndConversion
            DSP AfterConversion
            [Write Tags]
    All DSPs Freed
```

### Priority
- dBpoweramp does not appear to support priority queuing or reordering.
- The queue order is the order in which files were added.

---

## 4. File Locking Strategy During Batch Operations

### Source Files
- **Source files are opened read-only** throughout the batch.[^7]
- `CoreConverter.exe` opens the source with read-only sharing (`FILE_SHARE_READ` on Windows).
- **The converter cannot corrupt source files** even if a crash occurs mid-conversion.
- This is a hard guarantee from the dBpoweramp team.

### Destination Files
- Each output file maps to a unique destination path (generated from the naming template).
- **Concurrent writes to the same destination file** are not a concern because each file in the batch maps to one output filename.
- The **Collision Policy** (Phase 0.6) handles the case where two source files map to the same destination filename (e.g., two files with identical tags).

### Post-Conversion Source Deletion
- The `Delete Source File` DSP effect is the **only** mechanism to remove source files — and it executes **only after each successful conversion** in the batch.
- A file is never deleted before its conversion completes and is verified.

---

## 5. UI Thread Progress Events

### How Progress Is Communicated
- Each `CoreConverter.exe` instance sends **progress messages** back to the Batch Converter process.
- The communication mechanism is IPC (inter-process communication) — likely Windows messages or a named pipe.
- Progress includes:
  - Current phase (decoding, DSP, encoding, tagging)
  - Percentage complete (0–100%)
  - Estimated time remaining
  - Current file being processed (index/total)

### What the UI Displays
- **Per-file progress bar:** Shows the progress of the currently converting file.
- **Overall batch progress bar:** Shows `N of M files complete`.
- **Per-file status icons:** Pending, converting (spinning), completed (checkmark), errored (X).
- **Estimated total time remaining:** Based on average time per completed file.

### Thread Safety
- The Batch Converter UI thread is separate from the `CoreConverter.exe` worker processes.
- Progress messages are queued and processed by the UI thread without blocking conversion.

---

## 6. Batch-Level ReplayGain (Album Mode)

### dBpoweramp ReplayGain Modes
dBpoweramp's ReplayGain DSP supports two modes:

**Mode 1 — Scan Then Apply (embedded tags):**
1. Phase 7 (ReplayGain Scan) runs after encoding each file.
2. Computes per-track RG values and embeds them as tags.
3. If **album mode** is enabled:
   - After all tracks are encoded, the Batch Converter re-scans each output file.
   - Computes album-level RG values from the aggregate loudness.
   - Updates all track files with album gain/peak tags.

**Mode 2 — Apply During Encoding (real-time normalization):**
1. The DSP computes RG values on the fly during Phase 4 (DSP Chain).
2. Applies gain to the PCM buffer before encoding.
3. Tags are written with the applied gain values.

### Album Mode Implementation
Album mode requires a **two-pass approach** across the batch:
- **Pass 1:** Encode all tracks; collect per-track integrated loudness values.
- **Pass 2:** Calculate album-level gain; update all track files with the album gain tag.
- This is an additional step after Phase 9 (Finalization) of each individual file.

### fre:ac Album Mode
fre:ac also supports ReplayGain with album mode, computing album gain from all tracks in a single encoding pass when tracks are added together.

---

## 7. Error Handling in Batch (Continue vs. Abort)

### Default Behavior: Continue on Error
- **Per-file errors do NOT abort the batch.** The queue continues with the next file.
- Each `CoreConverter.exe` instance handles its own errors independently.

### Per-File Error Handling
- When an error occurs during conversion (e.g., decode error, encoder failure):
  1. The `CoreConverter.exe` process logs the error.
  2. If the **`Delete Destination File on Error`** DSP effect is enabled: the temp/destination file is deleted.
  3. The file is marked as **errored** in the batch queue.
  4. The batch continues with the next file.

### Codec-Level Error Tolerance
- **FLAC:** Can be configured to "Continue Decoding" (instead of "Stop Decoding on error") via Advanced Codec Options.
- This prevents a single corrupt frame from failing the entire file.

### Conditional Encoding
- The **`Conditional Encoding` DSP** can skip or 1:1-copy files matching certain criteria:
  - Bitrate threshold (skip lossy files above a certain bitrate)
  - Bit depth threshold
  - Sample rate threshold
  - File extension

### Batch Log Files
- dBpoweramp **does not generate batch log files** by default.[^8]
- The developer (Spoon) confirmed: "No log files for converter."
- The only record of batch results is the UI summary at the end.
- **Users must manually review** the error icons in the batch queue.

### Aborting a Batch
- Clicking "Cancel" stops all active `CoreConverter.exe` processes.
- Any in-progress files produce partial temp files that are cleaned up.
- Completed files are NOT rolled back.

---

## 8. Batch Statistics

### What dBpoweramp Reports
At the end of a batch, the UI displays:
- **Total files processed**
- **Successful conversions**
- **Failed conversions** (with error reason on hover/click)
- **Skipped files** (e.g., destination already exists with skip policy)

### No Persistent Log File
- There is **no batch log file** written to disk by default.
- Users who need audit trails must use the command-line interface with output redirection:
  ```bash
  dMC -convert "source_folder" -output "dest_folder" > conversion_log.txt 2>&1
  ```

### fre:ac Comparison
- fre:ac displays a conversion log with warnings and errors after batch completion.
- fre:ac shows: "Conversion finished with X warnings" / "Conversion finished with X errors."

---

## 9. Memory Management Across Concurrent Conversions

### Per-Process Memory Isolation
- Each `CoreConverter.exe` instance runs in its own process with private memory space.
- Memory usage **scales linearly** with the number of concurrent conversions.
- Windows process isolation prevents memory corruption between workers.

### Memory Usage Per Conversion
- **Decode buffer:** `~10 MB per minute of audio` at 44.1kHz/stereo/16-bit.
- At 24-bit/96kHz/5.1: ~85 MB per minute of audio.
- A 5-minute track at CD quality: ~50 MB of PCM buffer during Phase 3.
- DSP chain adds minimal memory overhead (filter state buffers).
- Encoder output buffer: proportional to bitrate.

### Concurrent Memory Budget
- On a system with 16 GB RAM running 16 concurrent conversions:
  - At 50 MB per conversion: ~800 MB total.
  - Headroom remains for OS, GUI, and other applications.
- On a system with 4 GB RAM running 4 concurrent conversions:
  - Memory pressure could occur with high-resolution audio.
  - The OS swaps; disk becomes the bottleneck.

### Bottleneck Analysis
- **Lossless-to-lossless conversions:** Disk I/O is typically the bottleneck, not CPU or memory.
- **Lossy encoding (MP3, AAC, Opus):** CPU is the bottleneck.
- **High-resolution audio (96kHz+):** Memory bandwidth and CPU cache efficiency matter.

### Comparison with fre:ac SuperFast
fre:ac takes a different approach — **within-file parallelism**: splitting a single audio stream into chunks and encoding them with multiple codec instances simultaneously.[^6]
- Advantage: Near-linear speedup for a single file on multi-core.
- Disadvantage: Only works for codecs that support chunk-based encoding (AAC, Opus, LAME). Does NOT work for FLAC, Vorbis, or WMA.
- dBpoweramp's approach (file-level parallelism) works for all codecs.

---

## 10. Performance Benchmarks

| Test Case | Setup | Time | Speed | Notes |
|---|---|---|---|---|
| 26 APE → WAV, 1 thread | Quad-core HT | 10m 16s | 1x | Baseline |
| 26 APE → WAV, 4 threads | Quad-core HT | 5m 7s | ~2x | ~50% faster |
| 102 FLAC → MP3 V0, Multi-CPU DSP (6 cores) | 6-core | 4m 31s | 75–80x | DSP overhead |
| 102 FLAC → MP3 V0, Options → 6 cores | 6-core | **1m 25s** | 200–250x | Optimal |
| 236 tracks (5GB), dBpoweramp | QAAC encoder | **1:35** | — | Near full CPU |
| 236 tracks (5GB), Media Center | QAAC encoder | 2:40 | — | Half-idle, spiky |

Sources: dBpoweramp Forum Multi-Core discussions[^3][^5]

> **Key finding:** The `Multi-CPU Force DSP` is 2–3x slower than the `Options → Encoding → Using X cores` setting. Always prefer the Options setting over the DSP for performance.

---

## 11. Edge Cases

### Edge Case 1: One File Fails in a Batch of 1000
- **Behavior:** The remaining 999 files continue processing. The failed file is marked with an error icon.
- **dBpoweramp:** Continue-on-error by default.
- **Impact:** None on other files.

### Edge Case 2: Disk Fills Up Mid-Batch
- **Behavior:** The current `CoreConverter.exe` process encounters a write error when the disk is full.
- **Phase 5:** Encoder fails to write; temp file is deleted.
- **Remaining files:** Continue if they write to a different volume; abort if all destinations are on the same full volume.
- **dBpoweramp:** Does not pre-check disk space per file; only at Phase 0 per individual file.

### Edge Case 3: Memory-Size Source File (24-hour 96kHz/5.1 FLAC)
- **Phase 3:** Decoder buffers the entire file (~250 GB PCM).
- **Memory exhaustion:** System OOM killer terminates `CoreConverter.exe`.
- **Behavior:** File marked as errored. Temp file is deleted.
- **Mitigation:** Chunked decode/encode processing is needed for very long files.

### Edge Case 4: User Removes Files from Queue Mid-Batch
- **Behavior:** Files currently converting continue to completion. Files not yet started are removed from the queue.
- **No rollback:** Already-completed files are not affected.

### Edge Case 5: Two Source Files Map to Same Destination
- **Phase 0.6:** Collision Policy determines behavior (skip/overwrite/rename/prompt).
- **Default:** Skip (do not overwrite).
- **User notice:** Second file appears as "Skipped" in the batch summary.

### Edge Case 6: Codec License Issue Mid-Batch (e.g., MP3 Encoder Not Registered)
- **Behavior:** `CoreConverter.exe` fails to initialize the encoder.
- **Error reported:** "No encoder available" or "Encoder registration required."
- **Batch continues:** Other files using different encoders proceed.

### Edge Case 7: Network Path Becomes Unavailable
- **Behavior:** Source or destination on a network share that drops mid-batch.
- **Current file:** Fails with I/O error.
- **Remaining files:** If the network recovers, subsequent files may succeed; if not, all remaining files fail.
- **No automatic retry.**

### Edge Case 8: Batch with Mixed Source Formats
- **Behavior:** Each `CoreConverter.exe` instance loads the appropriate decoder for its file's format.
- **Decoders are cached** across instances where possible (same DLL loaded by multiple processes).
- **No cross-file dependency:** Each file is fully independent.

### Edge Case 9: Encoding Priority vs. Background Tasks
- **Setting:** `Options >> Encoding >> Encoding Priority`
  - `Idle`: Background priority; other apps get CPU priority.
  - `Normal`: Balanced.
  - `High`: dBpoweramp gets aggressive CPU time.
- **Behavior:** Affects only the `CoreConverter.exe` processes, not the Batch Converter UI.

### Edge Case 10: Batch with Album ReplayGain — One Track Fails
- **Pass 1:** Files 1, 2, 3, 4 encode successfully; file 5 fails.
- **Pass 2 (album mode):** Only files 1–4 are scanned for album gain. File 5 is excluded.
- **Result:** Album gain is computed from the 4 successful files only.
- **dBpoweramp behavior:** Unclear whether it retries file 5 or proceeds with partial album. Likely proceeds with partial album.

---

## Would a User Notice Any Difference from dBpoweramp?

**For batch conversion behavior: Very likely no.** dBpoweramp's batch model is well-understood and the concurrency model is straightforward. Key areas of close alignment:

1. **Process-level parallelism** — our implementation using subprocess/worker pool matches dBpoweramp's `CoreConverter.exe` model.
2. **Per-file error isolation** — a failure in one file does not cascade.
3. **Read-only source access** — source files are never modified.
4. **UI progress reporting** — per-file and batch-level progress.

**Possible user-noticeable differences:**
- **Hard cap of 16 concurrent tracks** — our implementation could theoretically allow more, but diminishing returns and system stability make 16 a reasonable cap.
- **DSP Multi-CPU overhead** — the Multi-CPU Force DSP has significant overhead. An implementation that does not replicate this exact DSP behavior will actually be *faster* — which is an improvement, not a divergence.
- **No batch log file** — dBpoweramp has no persistent log. Our implementation could add one as a feature improvement.
- **fre:ac's within-file SuperFast parallelism** is superior for single large files on multi-core, but irrelevant for batch conversion where file-level parallelism already scales.

---

## References

[^1]: dBpoweramp DSP Architecture. Spoon (dBpoweramp developer). https://forum.dbpoweramp.com/forum/other-topics/developers-corner/22043-music-converter-dsp-architecture
[^2]: dBpoweramp Music Converter Help. https://dbpoweramp.com/Help/dMC/dMC
[^3]: dBpoweramp Forum — "How can I get batch file conversion to use multiple cores?" https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/34814-how-can-i-get-batch-file-conversion-to-use-multiple-cores
[^4]: dBpoweramp Forum — "Maximum Number of Cores/Threads for Batch Conversion." https://forum.dbpoweramp.com/forum/dbpoweramp/batch-converter/34751-maximum-number-of-cores-threads-for-batch-conversion
[^5]: dBpoweramp Forum — "Multi Core During Batch Conversion" / "Option: Encoding → Using x Cores." https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/21725-multi-core-during-batch-conversion
[^6]: fre:ac Developer Blog — "Introducing SuperFast." https://www.freac.org/developer-blog-mainmenu-9/14-freac/257-introducing-superfast-conversions
[^7]: dBpoweramp Forum — "Original files corrupted after batch convert." https://forum.dbpoweramp.com/forum/dbpoweramp/batch-converter/39962-original-files-corrupted-after-batch-convert
[^8]: dBpoweramp Forum — "Batch conversion: Skip files in destination." https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/15609-batch-conversion-skip-files-in-destination
