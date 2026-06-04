# 57 — On-the-Fly Pipe Encoding Architecture

> **Scope:** How dBpoweramp handles streaming/pipe mode encoding (decode→encode without intermediate temp files), when temp files are used vs. pipes, memory usage, error handling, CD ripping integration, and audio data flow. Primary sources: dBpoweramp CLI documentation[^1], dBpoweramp forum discussions[^2], FFmpeg pipe documentation[^3], cdparanoia documentation[^4], fre:ac documentation[^5], and PipeWire architecture docs[^6].

---

## 1. Overview: Pipe Mode vs. Temp File Architecture

There are two fundamentally different approaches to the decode→encode pipeline:

### Temp File Architecture
```
Source File → Decode to PCM → Write PCM to temp file → Read PCM from temp file → Encode to output
```
- The PCM is written to disk between decode and encode stages.
- Requires disk space for the PCM dump.
- Provides seekability: if the encoder needs to seek (e.g., to update a header), it can read the temp file.

### Pipe Architecture (Streaming)
```
Source File → Decode to PCM → Feed PCM directly to encoder → Output file
```
- No intermediate temp file; PCM flows directly from decoder to encoder in memory.
- Requires less disk space.
- **No seeking:** If the encoder needs to seek backward (e.g., to update a header with final values), this is impossible in pure pipe mode.

### dBpoweramp's Approach
- **dBpoweramp uses temp files** throughout the pipeline.
- Output is always written to a temp file, then atomically renamed to the final destination.
- **dBpoweramp does NOT support pipe/streaming output.**[^2]

---

## 2. Does dBpoweramp Support Streaming/Pipe Mode?

### Critical Finding: No

Direct quote from dBpoweramp developers:[^2]

> *"Default ffmpeg lines use %s for source file and transcode to stdout (dBpoweramp can't do this)."*

### Why dBpoweramp Cannot Stream to stdout
- **File header requirements:** Many formats (M4A, MP4, FLAC) require writing accurate header information (sample rate, channel count, total sample count) **before** or **at the beginning** of the audio data. This requires knowing the total sample count before encoding begins, or seeking backward after encoding completes.
- **Metadata writing:** dBpoweramp's tag writing (Phase 6) often requires seeking to the beginning of the file after audio encoding to write tags before the audio data.
- **Verification:** Phase 8 (Verification) requires reading the output file to verify duration and checksums.
- **Temp file approach:** Writing to a temp file allows all of these operations without requiring a seekable output stream.

### When dBpoweramp Uses Temp Files
- **Always.** dBpoweramp writes to a `.tmp` file in the same directory as the destination.
- The temp file is atomically renamed to the final filename on success.
- The temp file is deleted on failure.

### fre:ac's "Encode on the Fly"
fre:ac supports **"Encode on the Fly"** — an option that streams audio data through the encoder without creating intermediate WAV files on disk.[^5]
- This is a **disk-space optimization**, not a stdio pipe architecture.
- fre:ac still uses a temporary internal buffer; it just avoids writing a WAV file to disk between stages.
- fre:ac does **not** expose a stdout interface for external tool chaining.

---

## 3. When Does It Use Temp Files vs. Pipes?

### Temp File Usage Decision Tree
```
Does the output format require:
  - Accurate total sample count in the header?  → YES → Must use temp file
  - Seeking to update header after encoding?    → YES → Must use temp file
  - Post-encoding verification?                 → YES → Must use temp file
  - Tag writing after audio data?               → YES → Must use temp file
         ↓
If ALL answers are NO → Pipe mode is possible.
If ANY answer is YES → Temp file is required.
```

### Formats That Cannot Use Pipe Mode
| Format | Reason | Temp File Required? |
|---|---|---|
| **MP4/M4A (AAC, ALAC)** | `moov` atom size must be accurate; often placed at end | Yes |
| **FLAC** | `STREAMINFO` has total sample count; MD5 signature | Yes |
| **WAV** | `data` chunk size in header | Yes (unless using WAVEXT/RIFF64) |
| **AIFF** | `SSND` chunk size in header | Yes |

### Formats That Could Use Pipe Mode
| Format | Reason | Pipe Mode Possible? |
|---|---|---|
| **Raw PCM** | No container; just audio frames | Yes |
| **OGG Vorbis** | Ogg container supports streaming; page order doesn't require pre-knowledge | Yes |
| **Opus (in Ogg)** | Ogg container supports streaming | Yes |
| **MP3** | Bit reservoir complicates pipe mode; LAME can output to stdout | Yes, but complex |
| **Null/Testing** | No output needed | N/A |

### dBpoweramp's Universal Temp File
- dBpoweramp **always uses temp files** regardless of format.
- This ensures consistent behavior across all formats.
- The temp file is deleted after atomic rename, so the user never sees it.

---

## 4. Memory Usage for Pipe Mode (Buffer Sizes)

### Theoretical Minimum Memory
For a pipe-mode conversion:
- **Input buffer:** Decoder output buffer (e.g., 4096–65536 samples).
- **DSP buffer:** Processing buffer (same size as input).
- **Output buffer:** Encoder input buffer (same size).
- **Total:** ~3 × buffer_size of audio samples.

### Practical Buffer Sizes
| Buffer Size | CD Quality (44.1kHz/2ch/16-bit) | Latency |
|---|---|---|
| 256 frames | 5.8 ms | Ultra-low latency |
| 512 frames | 11.6 ms | Low latency |
| 1024 frames | 23.2 ms | Standard latency |
| 4096 frames | 92.9 ms | High latency |
| 8192 frames | 185.8 ms | Very high latency |

### Memory Calculation
```
Memory per buffer = frames × channels × bytes_per_sample

At 44.1kHz/2ch/16-bit:
  1024 frames: 1024 × 2 × 2 = 4,096 bytes ≈ 4 KB
  65536 frames: 65536 × 2 × 2 = 262,144 bytes ≈ 256 KB

At 96kHz/6ch/24-bit:
  1024 frames: 1024 × 6 × 3 = 18,432 bytes ≈ 18 KB
  65536 frames: 65536 × 6 × 3 = 1,179,648 bytes ≈ 1.1 MB
```

### OS Pipe Buffer Sizes
- **Linux:** Default pipe buffer = 65536 bytes (64 KB). Can be increased with `fcntl(F_SETPIPE_SZ)`.
- **Windows:** Anonymous pipe buffer = 64 KB (configurable).
- **Named pipe (FIFO):** Similar to anonymous pipe.

### FFmpeg Pipe Mode Memory
```bash
# FFmpeg pipe mode
ffmpeg -i input.flac -f s16le -acodec pcm_s16le pipe:1 | \
  lame -b 320 - -o output.mp3
```
- FFmpeg writes to stdout; `lame` reads from stdin.
- The OS pipe buffer (64 KB) manages flow control.
- Memory usage: FFmpeg decoder buffer + lame encoder buffer + OS pipe buffer ≈ a few MB.

---

## 5. Error Handling Mid-Stream in Pipe Mode

### Challenges
1. **No rollback:** In pipe mode, once data has been written to stdout and consumed by the encoder, it cannot be recovered.
2. **No partial output:** If the encoder fails mid-stream, the output file is incomplete.
3. **Flow control:** If the encoder is slower than the decoder, the pipe buffer fills; the decoder must pause.

### Error Scenarios

#### Scenario A: Decode Error Mid-Stream
- **Source file is corrupted.**
- Decoder encounters an error frame.
- **In temp file mode:** Decoder stops; temp file is deleted; error is logged.
- **In pipe mode:** Decoder stops writing to pipe; encoder receives partial data; output is corrupt.
- **Mitigation:** Decoder verification (FLAC `--verify`); skip corrupt frames if `continue_on_error` is set.

#### Scenario B: Encoder Error Mid-Stream
- **Encoder encounters an issue** (e.g., bit reservoir overflow in MP3).
- Encoder stops consuming from the pipe.
- Pipe buffer fills; decoder blocks on write; deadlock possible.
- **Mitigation:** Use non-blocking writes; detect encoder failure; drain the pipe.

#### Scenario C: Disk Full on Output
- **In temp file mode:** Write fails; temp file is deleted; error is logged.
- **In pipe mode:** Output stream write fails (e.g., network disconnection); pipe breaks; decoder detects broken pipe.
- **Mitigation:** Check disk space before starting; use buffered writes with error detection.

#### Scenario D: User Cancellation Mid-Stream
- **User presses Cancel.**
- Decoder stops producing data.
- Encoder drains remaining data; completes partial output.
- **Result:** Incomplete output file.
- **Mitigation:** Delete the partial output file on cancellation.

### dBpoweramp's Error Handling (Temp File Mode)
- Temp file is deleted on any error.
- Error is logged with details.
- Batch continues with next file.
- Output is never left in a corrupt/incomplete state.

---

## 6. CD ripping on-the-Fly: cdparanoia / cdda2wav Integration

### cdparanoia Native Pipe Mode
cdparanoia supports **native pipe output** by using `-` as the output file argument:[^4]

```bash
# Basic pipe to encoder
cdparanoia -r 1 - | flac - -o track01.flac

# With verification
cdparanoia -X 1 - | lame -b 320 - track01.mp3

# Batch with track splitting
cdparanoia -B 1-12 -
```

### cdparanoia Output Formats
| Flag | Format | Notes |
|---|---|---|
| `-w` (default) | WAV | LSB-first byte order |
| `-a` | AIFF | MSB-first byte order |
| `-r` | Raw PCM LE | 16-bit little-endian |
| `-R` | Raw PCM BE | 16-bit big-endian |
| `-p` | Raw PCM | Host byte order |

### cdda2wav Pipe Mode
```bash
cdda2wav -via - /path/to/output.wav
```
- `-paranoia`: Use libparanoia for superior quality.
- `readahead=`: Sets read-ahead buffer size (150–300 sectors default in paranoia mode).

### Error Handling in Paranoia Mode
- libparanoia performs **overlap checking** to verify data integrity.
- Scratch reconstruction for damaged CDs.
- Errors logged to stderr or a log file (`-L --logfile`).

### fre:ac CD Ripping Architecture[^5]
- **Linux/FreeBSD/OpenBSD:** Uses `cdparanoia` backend.
- **macOS/Solaris:** Uses `cdio` backend.
- **Windows:** Uses `CDRip` backend.
- fre:ac does **not** expose a pipe interface for external encoders.
- fre:ac handles CD ripping internally and passes audio data to its internal encoders.

### dBpoweramp CD Ripping
- dBpoweramp uses its own CD ripper (not cdparanoia).
- The ripped audio data is passed internally to `CoreConverter.exe` for encoding.
- No external pipe interface is exposed.
- **dBpoweramp does NOT support piping CD audio to external tools.**

### Pipe Chain Examples
```bash
# CD → FLAC (one liner)
cdparanoia -X 1 - | flac - -o track01.flac

# CD → MP3 320 kbps
cdparanoia 1 - | lame -b 320 - track01.mp3

# CD → Ogg Vorbis q8
cdparanoia 1 - | oggenc -q 8 -o track01.ogg

# CD → Opus 128 kbps
cdparanoia 1 - | opusenc --bitrate 128 - track01.opus

# Batch with automatic naming
cdparanoia -B --stderr -Z -w --prefix="track" - | \
  parallel -j 4 flac - -o {.}.flac
```

### Latency in CD Pipe Mode
- cdparanoia reads at ~10x realtime from a healthy CD.
- The pipe delivers data to the encoder at the encoder's consumption rate.
- Overall throughput is limited by the encoder's speed.
- For a CD rip → FLAC: ripping is fast; encoding is the bottleneck.

---

## 7. Audio Data Flow in Pipe Mode

### Pure Pipe Mode (FFmpeg-centric)
```
[CD Drive / Source File]
        ↓
[cdparanoia / Decoder]
        ↓ (pipe:1)
[Optional DSP / Filter]
        ↓ (pipe)
[Encoder (LAME/opusenc/flac)]
        ↓ (pipe:1)
[Output File / stdout]
```

### FFmpeg Filter Graph in Pipe Mode
```bash
ffmpeg -i input.flac \
  -af "volume=0.8,highpass=f=200,lowpass=f=16000" \
  -f s16le pipe:1 | \
  opusenc --bitrate 128 - output.opus
```

### dBpoweramp's Internal Data Flow
```
[Source File]
        ↓
[Decoder (Phases 2-3)]
        ↓ (internal PCM buffer)
[DSP Chain (Phase 4)]
        ↓ (internal PCM buffer)
[Encoder (Phase 5)]
        ↓ (temp file)
[Tag Writer (Phase 6)]
        ↓ (temp file)
[Verification (Phase 8)]
        ↓
[Atomic Rename (Phase 9)]
        ↓
[Output File]
```

### Key Differences from Pipe Mode
| Aspect | Pipe Mode | dBpoweramp Temp File |
|---|---|---|
| **Seeking** | Not possible mid-stream | Fully supported |
| **Header updates** | Not possible | Supported (rewind temp file) |
| **Tag writing** | Must know tags before encoding | Can write after encoding |
| **Verification** | Not possible | Fully supported |
| **Error recovery** | Partial | Complete (delete temp) |
| **Disk usage** | Minimal | Requires free space |
| **Memory usage** | Low | Higher (full PCM in memory) |

---

## 8. Edge Cases

### Edge Case 1: Container Formats with Post-Header Placement
- **MP4/M4A:** The `moov` atom contains all metadata and sample tables. In standard MP4, it is placed at the **end** of the file.
- **Faststart:** A special MP4 optimization that moves the `moov` atom to the **beginning** for streaming.
- **Pipe mode:** Without `faststart`, the `moov` is at the end — streaming playback is impossible.
- **dBpoweramp:** Likely places `moov` at the beginning (faststart equivalent) for better compatibility.
- **FFmpeg solution:** Use `movflag +faststart` after encoding: `ffmpeg -i input -c:a copy -movflags +faststart output.mp4`

### Edge Case 2: FLAC Ogg Wrapper for Pipe Mode
- **FLAC** normally uses native framing (no container).
- **For pipe mode,** FLAC must use the **Ogg wrapper**: `flac --ogg - -o output.oga`
- The Ogg container allows streaming; native FLAC framing does not.
- dBpoweramp: Uses native FLAC framing (not Ogg wrapper) because it writes to a temp file.

### Edge Case 3: MP3 Bit Reservoir
- **Problem:** MP3 uses a bit reservoir — frames can reference data from previous frames.
- **In pipe mode:** If frames are output out of order, the bit reservoir is invalid.
- **SuperFast LAME:** Handles this by unpacking frames, re-packing with reservoir management.[^7]
- **FFmpeg + LAME:** Outputs to a file; seeks work correctly. Pipe mode may corrupt the bit reservoir.
- **dBpoweramp:** Writes to temp file; bit reservoir is handled correctly.

### Edge Case 4: Encoder Failure on First Frame
- **Scenario:** Encoder (e.g., QAAC) fails to initialize on the first PCM frame.
- **Pipe mode:** No output is written; the input pipe is still open.
- **Mitigation:** Detect encoder initialization failure before starting the pipe.
- **dBpoweramp:** Encoder initialization happens before Phase 5 begins.

### Edge Case 5: Partial Output File on Cancellation
- **Scenario:** User cancels mid-conversion.
- **Pipe mode:** Output file is partially written; must be deleted.
- **Temp file mode:** Temp file is partially written; deleted on cancellation.
- **Both approaches:** Leave a partial output if not handled. dBpoweramp cleans up the temp file.

### Edge Case 6: Network Stream as Source
- **Scenario:** Input is a network stream (HTTP, SFTP).
- **Pipe mode:** `ffmpeg -i http://source/audio.flac - | flac - -o output.flac`
- **dBpoweramp:** Does not support network streams as input.

### Edge Case 7: Converting While Downloading
- **Scenario:** Downloading a FLAC file; want to convert it as it's being written.
- **Pipe mode:** `curl -L http://source/audio.flac | ffmpeg -i pipe:0 - | flac - -o output.flac`
- **Temp file mode:** Must wait for download to complete.
- **dBpoweramp:** Cannot do this; requires a complete file on disk.

### Edge Case 8: Encoder Header Updates Required
- **WAV:** `fmt ` chunk and `data ` chunk sizes must be accurate.
- **AIFF:** `SSND` chunk size must be accurate.
- **FLAC:** `STREAMINFO` total sample count and MD5 signature.
- **Pipe mode:** Cannot update headers after encoding without seeking.
- **Workaround:** Buffer entire output; seek and update; write final file. This defeats the purpose of pipe mode.

### Edge Case 9: Multi-Encoder in Pipe Mode
- **Scenario:** Pipe output to multiple encoders simultaneously.
- **Implementation:** `tee` to multiple encoder processes.
- **Pipe buffer:** Each encoder reads from a separate pipe fed by `tee`.
- **Synchronization:** `tee` must handle multiple readers without blocking.
- **dBpoweramp:** Not applicable (no pipe mode).

### Edge Case 10: Testing with /dev/null Output
- **Scenario:** Benchmarking decode speed without encoding overhead.
- **Pipe to /dev/null:** `ffmpeg -i input.flac -f null -`
- **Result:** Decode speed measured without encoding bottleneck.
- **dBpoweramp:** No equivalent; must always produce output.

---

## Would a User Notice Any Difference from dBpoweramp?

**Likely yes** in the following areas:

1. **Pipe mode support:** dBpoweramp does NOT support pipe/streaming output. Our implementation should also NOT attempt pipe mode — dBpoweramp's approach (temp files) is correct for correctness. Pipe mode is only viable for specific formats (OGG, raw PCM) and is rarely needed.
2. **CD ripping:** dBpoweramp has its own CD ripper. If we implement CD ripping, using `cdparanoia` directly with pipe output is the most flexible approach.
3. **Verification:** dBpoweramp's temp file approach enables Phase 8 verification. Our implementation must also support verification, which is impossible in pure pipe mode.

**No practical difference** for typical users who convert files from disk. The temp file approach is transparent and more reliable.

---

## References

[^1]: dBpoweramp CLI Encoder documentation. https://www.dbpoweramp.com/
[^2]: dBpoweramp Forum — CLI encoding and stdout limitations. https://forum.dbpoweramp.com/
[^3]: FFmpeg Documentation — Pipe input/output. https://ffmpeg.org/ffmpeg.html
[^4]: cdparanoia manual — Xiph.org. https://xiph.org/paranoia/
[^5]: fre:ac official site and documentation. https://www.freac.org/
[^6]: PipeWire documentation. https://pipewire.org/
[^7]: fre:ac Developer Blog — SuperFast LAME Technical Details. https://www.freac.org/developer-blog-mainmenu-9/14-freac/287-superfastlame
