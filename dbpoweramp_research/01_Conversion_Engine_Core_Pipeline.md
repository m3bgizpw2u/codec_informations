# Conversion Engine Core Pipeline — DBpoweramp Behavior Research
> **Research Category:** Pipeline
> **DBpoweramp Versions Studied:** R14 through R2026-04 (current)
> **Confidence Level:** High — documented (official Spoon developer forum posts, SDK docs) + spec inference
> **Primary Sources:** [Music Converter DSP Architecture - dBpoweramp Forum](https://forum.dbpoweramp.com/forum/other-topics/developers-corner/22043-music-converter-dsp-architecture) | [dBpoweramp Developer DSP Effects](https://www.dbpoweramp.com/developer-dmc-dsp-effects) | [dBpoweramp Developer Compression](https://www.dbpoweramp.com/developer-compression) | [dBpoweramp Batch Multi-Encoder](https://dbpoweramp.com/Help/dMC/multiencoder) | [dBpoweramp Scripting COM](https://dbpoweramp.com/developer-scripting-dmc) | [dBpoweramp Release Notes](https://versions.dbpoweramp.com/?appid=2)
> **Open-Source Reference:** BoCA (fre:ac) | FFmpeg | beets

---

## 1. TOPIC OVERVIEW & PURPOSE

### 1.1 What This Component Does

The DBpoweramp Conversion Engine is the per-file pipeline that runs inside each CoreConverter.exe process. It coordinates: (1) decoding compressed audio to PCM, (2) passing PCM through an ordered chain of DSP effects, (3) encoding PCM to the target format, and (4) managing tag data through a shared state structure (`STdBEncoderFluid`). The engine is responsible for memory management, progress reporting, error propagation, and atomic output file finalization.

### 1.2 Why This Matters for Re-implementation

Getting the pipeline stage ordering wrong causes silent corruption:
- Writing tags before encoding finishes → incomplete or missing tags
- Not calling `DSP.EndConversion()` before `Encoder.EndConversion()` → ReplayGain values never written
- Calling `DSP.AfterConversion()` with the `.tmp` filename instead of the final filename → wrong file gets post-processing
- Not respecting live vs. non-live DSP distinction → format-changing DSPs (resample, channel remix) produce wrong output
- Forgetting `ShouldWriteTags` negotiation → duplicate tag writes or missing tags

### 1.3 DBpoweramp's Design Philosophy for the Pipeline

Spoon designed the pipeline as a strict sequence with well-defined state transitions. The `STdBEncoderFluid` structure is the single source of truth for all mutable state (format, tags, filenames) that flows between stages. DSP plugins can modify the fluid at specific points: `BeginConversion`, `EndConversion`, and `AfterConversion`. This allows DSP effects to be genuinely stateful across the entire conversion — they can accumulate data, defer decisions, and modify tags — without the decoder or encoder needing to know anything about DSP state.

---

## 2. OBSERVED / DOCUMENTED BEHAVIOR

### 2.1 The Complete Master Pipeline — Stage by Stage

```
╔═══════════════════════════════════════════════════════════════════════════════╗
║              DBpoweramp CoreConverter.exe — Complete Pipeline                 ║
╠═══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  STAGE 0: PROCESS STARTUP                                                    ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║  CoreConverter.exe launched by host with CLI args:                            ║
║    -settings="encoder_name" -dspeffects="effect1;effect2;..."              ║
║    -input="source_path" -output="dest_path"                                 ║
║    -dspeffectsargs="encoded_effect_config"                                  ║
║                                                                              ║
║  STAGE 1: PLUGIN DISCOVERY + INSTANTIATION                                   ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║  1.1  Discover decoder DLL based on source file extension                    ║
║  1.2  Discover encoder DLL from -settings argument                           ║
║  1.3  Discover DSP effect DLLs from -dspeffects argument                     ║
║  1.4  LoadLibrary() each DLL                                               ║
║  1.5  Call decoder.Create() → decoder instance                               ║
║  1.6  Call encoder.Create() → encoder instance                               ║
║  1.7  For each DSP: DSP_Create(sample_rate, channels, bits) → DSP instance   ║
║                                                                              ║
║  STAGE 2: BEFORE OPEN COMMUNICATION                                          ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║  2.1  Encoder asks Decoder: NeedHQAudio() — does encoder need high quality? ║
║  2.2  Decoder.Open(InFileName) opens source file, reads audio properties   ║
║  2.3  Decoder reads ID Tags from source file → populates STdBEncoderFluid    ║
║  2.4  Decoder creates EncoderFluid structure with current format settings     ║
║       NOTE: AudioInfo items (FileSize, Lengthmsec) may be 0 for CD input    ║
║                                                                              ║
║  STAGE 3: BEFORE ENCODING COMMUNICATION                                      ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║  3.1  NON-LIVE DSPs only:                                                  ║
║        Decoder decodes ENTIRE source → raw PCM temp file                    ║
║        For each non-live DSP: DSP.PassNonLive(temp_pcm_path, fluid)        ║
║        (Non-live DSPs modify the temp file or produce a new one)            ║
║                                                                              ║
║  3.2  LIVE DSPs: DSP.BeginConversion(EncoderFluid) — setup live DSPs       ║
║                                                                              ║
║  3.3  Encoder.BeginConversion(EncoderFluid)                                  ║
║        Encoder reads: InFileName, OutFileName (.tmp.xxx), WaveFormat,       ║
║        CompressionType, CompressionCLI, IDTags, AudioProps                  ║
║        Encoder opens output file, writes container headers                  ║
║        IMPORTANT: If DSP changes WaveFormat → Encoder MUST be notified        ║
║        If DSP changes ID Tags → Encoder sees updated tags at EndConversion   ║
║                                                                              ║
║  STAGE 4: ENCODE LOOP (repeats until all audio consumed)                    ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║  4.1  Decoder.DecodeBlock() → STdBAudioBlock {pData, Bytes, Blank0}       ║
║        pData: pointer to PCM samples (owned by decoder)                     ║
║        Bytes: size of block in bytes                                        ║
║        Blank0: TRUE for last block, FALSE otherwise                        ║
║                                                                              ║
║  4.2  For each LIVE DSP in order:                                          ║
║        DSP.PassAudioBlock(block) → modified block (or new block)             ║
║        Block ownership: decoder's block → DSP returns new block              ║
║        NOTE: Non-live DSPs are SKIPPED here — already processed             ║
║                                                                              ║
║  4.3  Encoder.EncodeBlock(modified_block)                                   ║
║        Encoder writes compressed audio to output file                        ║
║                                                                              ║
║  4.4  Send progress message to host:                                        ║
║        "[filename] [percent] [encoding_speed]"                               ║
║                                                                              ║
║  LOOP until Blank0 = TRUE (last block received)                             ║
║                                                                              ║
║  STAGE 5: END ENCODING COMMUNICATION                                        ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║  5.1  Decoder.Close(EncoderFluid)                                          ║
║        Decoder releases resources; decoder object STILL EXISTS after Close    ║
║        Decoder may update EncoderFluid (e.g., final audio properties)         ║
║                                                                              ║
║  5.2  For each LIVE DSP: DSP.EndConversion(EncoderFluid)                    ║
║        LIVE DSPs finalize; may write to EncoderFluid.IDTags (e.g., RG)      ║
║        NOTE: Non-live DSPs do NOT receive EndConversion                    ║
║                                                                              ║
║  5.3  Encoder.EndConversion(EncoderFluid, &ShouldWriteTags)                 ║
║        Encoder writes container footer / seek index / final frames            ║
║        Encoder sets ShouldWriteTags:                                         ║
║          TRUE = encoder handles its own tag writing                         ║
║          FALSE = host/DSP will handle tag writing                           ║
║        Encoder object STILL EXISTS after EndConversion                       ║
║                                                                              ║
║  STAGE 6: AFTER CONVERSION COMMUNICATION                                     ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║  6.1  For BOTH live AND non-live DSPs:                                      ║
║        DSP.AfterConversion(EncoderFluid)                                     ║
║        NOW receives FINAL filename (not .tmp)                                ║
║        Called TWICE if EncoderFluid has 2 filenames (split output)           ║
║        On error: receives .tmp filename (not final)                         ║
║        Uses: ReplayGain post-encoding scan, external program hooks, etc.     ║
║                                                                              ║
║  6.2  If ShouldWriteTags = TRUE:                                            ║
║        Encoder has already written tags to output file                      ║
║        If ShouldWriteTags = FALSE:                                          ║
║        Host process writes tags separately (tag-only pass)                   ║
║                                                                              ║
║  STAGE 7: CLEANUP                                                           ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║  7.1  DELETE Encoder object → Encoder DLL freed                             ║
║  7.2  DELETE all DSP objects → DSP DLLs freed                               ║
║  7.3  DELETE Decoder object → Decoder DLL freed                             ║
║                                                                              ║
║  STAGE 8: FILE FINALIZATION                                                  ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║  8.1  If ConversionError = TRUE:                                            ║
║        DELETE output_path.tmp  (remove incomplete file)                     ║
║        Exit with non-zero code                                               ║
║                                                                              ║
║  8.2  MoveFileEx(output_path.tmp, output_path, MOVEFILE_REPLACE_EXISTING)   ║
║        (Atomic rename within same filesystem)                                ║
║                                                                              ║
║  8.3  Exit with code 0 (success)                                           ║
║                                                                              ║
╚═══════════════════════════════════════════════════════════════════════════════╝
```

### 2.2 Behavior Evidence Sources

| Behavior Claim | Source | Confidence | Quote / Reference |
|---|---|---|---|
| Complete DSP pipeline sequence | [dBpoweramp Forum: DSP Architecture](https://forum.dbpoweramp.com/forum/other-topics/developers-corner/22043-music-converter-dsp-architecture) | High | Official Spoon specification — all function calls and stage ordering |
| ID tags read by decoder, passed via fluid | [dBpoweramp Developer Compression](https://www.dbpoweramp.com/developer-compression) | High | "how Tag information is set, it is not handled in any way by compression DLLs, the input DLL is used to set / get Tag info" |
| AfterConversion called with final filename | [dBpoweramp Forum: DSP Architecture](https://forum.dbpoweramp.com/forum/other-topics/developers-corner/22043-music-converter-dsp-architecture) | High | "DSP.AfterConversion (Passed EncoderFluid) Note: LIVE + NON-LIVE NOW PASSED FINAL FILENAME (can be called 2x if 2 filenames)" |
| Memory encode support | [dBpoweramp Forum: DSP Architecture](https://forum.dbpoweramp.com/forum/other-topics/developers-corner/22043-music-converter-dsp-architecture) | High | "Special case memory encode CAN be: [memory]" |
| Multi-encoder spawns multiple processes | [dBpoweramp Multi-Encoder Help](https://dbpoweramp.com/Help/dMC/multiencoder) | High | "Multicore allows Multi Encoder to use multiple cores at the same time" |
| Progress reporting via MsgTo | [dBpoweramp Forum: DSP Architecture](https://forum.dbpoweramp.com/forum/other-topics/developers-corner/22043-music-converter-dsp-architecture) | High | "MsgTo: DMC with the % done, from CoreEncoder.exe to host" |
| ShouldWriteTags flag | [dBpoweramp Forum: DSP Architecture](https://forum.dbpoweramp.com/forum/other-topics/developers-corner/22043-music-converter-dsp-architecture) | High | "Encoder.EndConversion(EncoderFluid, &ShouldWriteTags)" |

### 2.3 Configuration Options (User-Facing)

| UI Option | Location in UI | Effect on Pipeline | Default |
|---|---|---|---|
| Encoder selection | Convert To dropdown | Determines which encoder plugin is loaded | FLAC (Lossless) |
| Compression quality | Encoder settings page | Passed as CLI string to encoder | VBR Quality 5 |
| DSP effects list | DSP tab (top to bottom order) | Ordered DSP chain loaded before BeginConversion | Empty |
| Bit Depth DSP | DSP effects list | Converts between integer and floating point at specified points | None |
| Resample DSP | DSP effects list | Sample rate conversion during encode loop | None |
| ReplayGain | DSP effects list | Calculates or applies loudness normalization | None |
| Output naming template | Output To | Used by host to compute final output path | `[source path]` |
| CPU core count | Options → Encoding | Limits concurrent CoreConverter.exe processes | Auto (max) |
| Temp folder | Control Center → Advanced | Where non-live DSP temp files are stored | System %TEMP% |

### 2.4 Codec Plugin Loading Mechanism

```pseudocode
FUNCTION LoadCodecPlugin(plugin_type, plugin_name):
    // plugin_type: "encoder" | "decoder"
    // plugin_name: user-visible name from "Convert To" menu
    
    folder = GetPluginFolder(plugin_type)  
    // Windows: "C:\Program Files\Illustrate\dBpoweramp\encoder\"
    // macOS:   "/Applications/dBpoweramp/Contents/encoder/"
    
    dll_path = FindDLLByName(folder, plugin_name)
    // Iterates DLLs in folder, compares strip_extension(filename) == plugin_name
    // "mp3 (Lame).dll" → "mp3 (Lame)"
    
    IF dll_path IS NULL:
        THROW "Codec not found: " + plugin_name
    
    hModule = LoadLibrary(dll_path)
    
    // Verify required exports exist
    ASSERT GetProcAddress(hModule, "Create") IS NOT NULL
    ASSERT GetProcAddress(hModule, "Info") IS NOT NULL
    
    // Optional exports
    show_config_fn = GetProcAddress(hModule, "ShowConfig")  // May be NULL
    get_error_fn   = GetProcAddress(hModule, "GetLastError") // May be NULL
    
    RETURN { hModule, create_fn, show_config_fn, get_error_fn }
```

### 2.5 Behavior Differences by Version

| Version | Change | Impact on Pipeline |
|---|---|---|
| Pre-R14 | 32-bit, single-core | All stages in single process, single-threaded |
| R14 | Up to 16 simultaneous CPUs | Process pool for concurrent conversions |
| R17 | 64-bit, 20% faster multi-core | Better memory management per process |
| R17.7 | Fixed floating point corruption through DSP | "floating point put through certain DSP effects would corrupt the audio" |
| R2020 | 64-core support, R16-style Multi-Encoder integration | Better CPU allocation for multi-encoder |
| R2023-12 | New Music Converter modernized start-encoder loop | "fixed race condition if NoConvert is set" |
| R2025-11 | 64-bit ONLY, ARDFTSRC resampler | Removed 32-bit compatibility overhead |
| R2025-12-25 | "CoreConverter: Don't delete overwritten file" | Atomic rename safety improvement |
| R2026-04 | Multi-CPU defaults to 4 for high disk usage formats | Better I/O scaling for WAV/FLAC |

---

## 3. INTERNAL LOGIC (INFERRED / REVERSE-ENGINEERED)

### 3.1 Conversion Coordinator — Queue and Threading

```pseudocode
STRUCT ConversionJob:
    source_path:     string
    output_path:     string
    encoder_name:    string
    dsp_list:        string[]       // Ordered DSP effect names
    dsp_args:        string         // Encoded DSP configuration
    compression_cli:  string         // Encoder CLI settings
    status:          enum { pending, running, completed, error }
    error_message:   string
    progress_pct:    float

STRUCT ConversionCoordinator:
    queue:            ConversionJob[]
    max_concurrent:   int            // User-selected CPU count
    active_processes: int            // Current running count
    
    semaphore:        Semaphore(max_concurrent)
    
    FUNCTION AddJob(job: ConversionJob):
        queue.APPEND(job)
    
    FUNCTION StartBatch():
        // Pre-read metadata for all files (for filename template resolution)
        FOR EACH job IN queue:
            job.tags = ReadIDTags(job.source_path)   // Fast tag read
            job.filename_template = ResolveTemplate(job.output_path, job.tags)
        
        // Start conversions with concurrency limit
        FOR EACH job IN queue:
            semaphore.ACQUIRE()
            THREAD_POOL.SUBMIT(() => {
                TRY:
                    RunCoreConverter(job)
                    job.status = completed
                CATCH ex:
                    job.status = error
                    job.error_message = ex.message
                FINALLY:
                    semaphore.RELEASE()
                    OnJobComplete(job)
            })
        
        THREAD_POOL.WAIT_ALL()
    
    FUNCTION RunCoreConverter(job: ConversionJob):
        // Build CoreConverter CLI
        cli = BuildCLI(job)
        
        // Start process
        proc = CreateProcess("CoreConverter.exe", cli)
        
        // Hook progress stream
        pipe = AttachOutputPipe(proc)
        WHILE line = pipe.READ_LINE():
            ParseProgressLine(line, job)  // Updates job.progress_pct
            UpdateAggregatedProgress()
        
        // Wait for completion
        exit_code = proc.WAIT()
        
        IF exit_code != 0:
            THROW ConversionError(job, ReadErrorLog())
```

### 3.2 Memory Management During Conversion

```pseudocode
// BLOCK-BASED STREAMING (not fully-loaded)
//
// Audio data is NEVER fully loaded into memory.
// Instead, DecodeBlock() returns fixed-size PCM blocks.
//
// Block sizing strategy:
FUNCTION GetOptimalBlockSize(decoder, encoder, dsp_chain):
    // Pick the largest power-of-2 buffer that works for all stages
    max_decoder_block = decoder.GetPreferredBlockSize()   // Format-dependent
    max_encoder_block = encoder.GetPreferredBlockSize()   // Format-dependent
    max_dsp_block     = MAX(dsp_chain.Map(d => d.GetPreferredBlockSize()))
    
    // Typically 4096–65536 samples per block
    // Choose the minimum of all three to avoid buffer copies
    RETURN MIN(max_decoder_block, max_encoder_block, max_dsp_block)

// Block ownership semantics:
FUNCTION EncodeLoop():
    WHILE TRUE:
        block = decoder.DecodeBlock()
        
        // DSP may return a NEW block with different pointer
        FOR EACH dsp IN live_dsp_chain:
            block = dsp.PassAudioBlock(block)
        
        // Encoder receives current block pointer
        encoder.EncodeBlock(block)
        
        // IMPORTANT: Decoder owns block memory until next DecodeBlock call
        // After next DecodeBlock(), previous block memory is invalidated
        
        IF block.IsLast():
            BREAK

// Non-live DSP temp file strategy:
FUNCTION HandleNonLiveDSPs():
    IF has_nonlive_dsp AND NOT has_live_dsp:
        // Simple case: decode to temp, DSP process, encode from temp
        temp_path = GetTempFile()
        DecodeEntireFile(decoder, temp_path)       // Full decode to disk
        FOR EACH dsp IN nonlive_dsp_chain:
            temp_path = dsp.PassNonLive(temp_path) // May produce new temp
        encoder.EncodeFromFile(temp_path)
        DELETE temp_path
    
    ELIF has_nonlive_dsp AND has_live_dsp:
        // Mixed: decode to temp, non-live DSPs, live encode loop
        temp_path = GetTempFile()
        DecodeEntireFile(decoder, temp_path)
        FOR EACH dsp IN nonlive_dsp_chain:
            temp_path = dsp.PassNonLive(temp_path)
        // Now open temp as input for encode loop
        temp_decoder = CreateDecoderForFile(temp_path)
        EncodeLoop(temp_decoder)  // Normal loop using temp decoder
        DELETE temp_path
```

### 3.3 Audio Format Negotiation Between Stages

```pseudocode
// FORMAT NEGOTIATION — Happens in EncoderFluid at BeginConversion
//
// The decoder's Open() populates EncoderFluid with the SOURCE format.
// The encoder's BeginConversion() receives this and may negotiate changes.

FUNCTION BeforeEncodingNegotiation():
    
    // Decoder reports source format
    source_fmt = decoder.Open(source_path)
    fluid.pWaveFormat = {
        format_tag:    WAVE_FORMAT_PCM or WAVE_FORMAT_IEEE_FLOAT,
        channels:      source_fmt.channels,
        samples_per_sec: source_fmt.sample_rate,
        bits_per_sample: source_fmt.bits_per_sample,
        block_align:   channels * bits_per_sample / 8,
        bytes_per_sec: channels * samples_per_sec * bits_per_sample / 8
    }
    
    // Encoder queries decoder: what format do you support?
    // (e.g., encoder might prefer float input for better quality)
    IF encoder.NeedHQAudio():
        // Encoder needs specific format
        fluid.pWaveFormat = encoder.GetRequiredFormat()
    
    // DSP chain may modify format
    FOR EACH dsp IN dsp_chain:
        IF dsp.RequiresFormatChange():
            new_fmt = dsp.GetRequiredFormat()  // e.g., Resample DSP
            // CRITICAL: Encoder must be notified of format change
            // If format changes AFTER BeginConversion, this is non-live only
            fluid.pWaveFormat = new_fmt
    
    // Final format set in EncoderFluid before Encoder.BeginConversion()
    encoder.BeginConversion(fluid)
```

### 3.4 Configuration System

| Storage Location | What Is Stored | Format |
|---|---|---|
| Windows Registry | Codec registration, shell integration | HKLM/HKCU `Software\Illustrate\dBpoweramp` |
| Per-user config files | Encoder preferences, naming templates | `%APPDATA%\dBpoweramp\` |
| Per-installation | Installed codecs, license key | `C:\Program Files\Illustrate\dBpoweramp\` |
| Per-file metadata | Source tags, AccurateRip cache | `%LOCALAPPDATA%\dBpoweramp\Cache\` |
| Temp files | Non-live DSP PCM data | `%TEMP%\dBpoweramp\` |

Key registry values (Windows):
- `InstalledCodecs`: List of registered codec DLLs
- `RegistrationName`, `RegistrationCode`: License key
- `ShellIntegration`: Whether context menu is registered
- `TempFolder`: Override for temp file location

### 3.5 Registration / License Checking Mechanism

```pseudocode
// License checking happens at startup of dBpoweramp.exe (not CoreConverter.exe)
//
// CoreConverter.exe processes are spawned by the host without re-checking license.
// License is enforced by:
//   1. Host process validates license before spawning converters
//   2. DSP effects require Power Pack / Reference license
//   3. COM scripting requires Reference license per PC + concurrent object
//   4. Secure ripping (CD Ripper) requires Reference

FUNCTION CheckLicense():
    IF trial_mode:
        days_remaining = ReadTrialExpiryDate()
        IF days_remaining <= 0:
            ShowPurchaseDialog()
            EXIT
        ELSE IF days_remaining <= 7:
            ShowTrialExpiringWarning(days_remaining)
    
    IF registered:
        stored_key = ReadRegistry("RegistrationCode")
        IF NOT ValidateRSAKey(stored_key):
            ShowInvalidKeyDialog()
            EXIT
    
    // No license check inside CoreConverter.exe — trust the host
    RETURN license_valid
```

### 3.6 Edge Case Handling

| Edge Case | DBpoweramp Behavior |
|---|---|
| Encoder changes output format mid-encode | Non-live path required; live encoders cannot handle format changes mid-stream |
| DSP changes sample rate during conversion | Must be non-live; live format changes are not supported |
| Source file deleted during batch | Current CoreConverter reports error; others continue |
| Output path on network share | Supports UNC paths; slower due to network I/O; CoreConverter spawns fail gracefully |
| DSP modifies IDTags after BeginConversion | Handled via EncoderFluid.IDTags pointer update before EndConversion |
| Encoder plugin crashes mid-encode | Process dies; host detects non-zero exit; .tmp file deleted |
| Multiple output files from single source (split) | EncoderFluid contains two filenames; AfterConversion called twice |
| CD input (stdin, no file) | Source filename = "-" (stdin); special case in decoder.Open() |

---

## 4. OPEN-SOURCE EQUIVALENT IMPLEMENTATION

### 4.1 FFmpeg Pipeline Equivalent

```bash
# FFmpeg equivalent of DBpoweramp's pipeline stages:

# Stage 2 (Before Open): No equivalent — FFmpeg doesn't separate read/open
# Stage 3 (Before Encode): No equivalent pre-communication
# Stage 4 (Encode Loop): Equivalent to FFmpeg's encode loop
# Stage 5 (End Encode): FFmpeg's flush_encoder
# Stage 6 (After Conversion): FFmpeg has no equivalent hook

# Equivalent pipeline for FLAC → MP3 with ReplayGain:
INPUT="source.flac"
OUTPUT="output.mp3"

# Decode + DSP (resample to 44100) + Encode:
ffmpeg -y -i "$INPUT" \
  -af "aresample=44100:res_type=soxr" \
  -c:a pcm_f32le -f wav pipe:1 | \
  lame -V 0 - "$OUTPUT"

# Tag writing (separate pass, equivalent to IDTags in EncoderFluid):
mid3v2 "$OUTPUT" \
  --artist="Artist Name" \
  --album="Album Name" \
  --title="Track Title" \
  --track="1/12" \
  --genre="Rock"

# ReplayGain calculation (equivalent to ReplayGain DSP AfterConversion):
mp3gain -r -c -s i "$OUTPUT"
```

### 4.2 BoCA Pipeline Comparison

| DBpoweramp Stage | BoCA Equivalent | Notes |
|---|---|---|
| Decoder.Open | `DecoderComponent::Initialize()` | Reads tags into component state |
| DSP.BeginConversion | `DSPComponent::StartProcessing()` | Live DSPs |
| DSP.PassNonLive | `DSPComponent::ProcessBuffer()` on temp file | Non-live DSPs |
| Encoder.BeginConversion | `EncoderComponent::Start()` | Opens output container |
| Decode/Encode Loop | `Stream::Flush()` / write cycle | Block-based |
| DSP.EndConversion | `DSPComponent::Finish()` | Finalize live DSP state |
| Encoder.EndConversion | `EncoderComponent::Finish()` | Write container footer |
| DSP.AfterConversion | No equivalent | DBpoweramp-unique |
| ShouldWriteTags | `TaggerComponent` vs encoder tag write | Tag writing separation |

---

## 5. DETAILED IMPLEMENTATION SPECIFICATION

### 5.1 Algorithm — Complete Conversion Sequence

**Step 1: CLI Argument Parsing**
```
Input:    CoreConverter.exe command line string
Process:  Parse key=value pairs from CLI:
           -settings      → encoder_name
           -dspeffects    → semicolon-separated DSP names
           -dspeffectsargs→ encoded DSP configuration (Base64 or similar)
           -input         → source_path
           -output        → dest_path
           -compresscli   → encoder CLI options
Output:   ParsedConversionConfig struct
Error if: Required argument missing → exit with error code
```

**Step 2: Plugin Instantiation**
```
Input:    ConversionConfig with encoder_name, dsp_list, source_extension
Process:
  1. decoder = CreateDecoder(source_extension)   // Auto-detected from extension
  2. encoder = CreateEncoder(encoder_name)
  3. FOR EACH dsp_name IN dsp_list:
       dsp = CreateDSP(dsp_name)
       dsp_chain.APPEND(dsp)
  4. Separate live vs non-live DSPs:
       live_dsp = [dsp IN dsp_chain WHERE dsp.IsLive()]
       nonlive_dsp = [dsp IN dsp_chain WHERE NOT dsp.IsLive()]
Output:   Instantiated pipeline components
Error if: Plugin DLL not found → exit with error code
```

**Step 3: Decoder Open and Tag Read**
```
Input:    source_path, decoder instance
Process:
  1. decoder.Open(source_path)
     → Fills STdBAudioInfo (duration, bitrate, sample rate, channels)
     → Populates STdBEncoderFluid.IDTags with raw encoded tag bytes
  2. Create EncoderFluid with:
     - InFileName = source_path
     - OutFileName = dest_path + ".tmp"  // temp until AfterConversion
     - WaveFormat = source audio format (may change)
     - CompressionType = encoder_name
     - CompressionCLI = encoder_settings
     - IDTags = raw bytes from decoder
Output:   STdBEncoderFluid ready for BeginConversion
Error if: File unreadable → ConversionError = TRUE, exit
```

**Step 4: Non-Live DSP Processing**
```
Input:    decoder, non_live_dsp_chain, EncoderFluid
Process:
  1. IF non_live_dsp_chain is NOT empty:
       a. temp_pcm_path = CreateTempFile(".pcm")
       b. DecodeEntireFile(decoder, temp_pcm_path)
          // Writes raw PCM to disk, all samples
       c. FOR EACH dsp IN non_live_dsp_chain (in order):
              temp_pcm_path = dsp.PassNonLive(temp_pcm_path, EncoderFluid)
              // dsp may write to a NEW temp file
       d. temp_decoder = CreateDecoderForFile(temp_pcm_path)
          // Create a decoder that reads from the processed temp file
       e. Replace decoder reference with temp_decoder for encode loop
  2. ELSE:
       // Normal decode-in-stream mode
Output:   Updated decoder (either original or temp-based)
```

**Step 5: Begin Conversion**
```
Input:    encoder, EncoderFluid, live_dsp_chain
Process:
  1. FOR EACH dsp IN live_dsp_chain:
       dsp.BeginConversion(EncoderFluid)
       // LIVE DSPs receive initial format, may modify EncoderFluid.WaveFormat
  2. encoder.BeginConversion(EncoderFluid)
     → Encoder reads all fluid fields
     → Opens dest_path + ".tmp" for writing
     → Writes container headers (WAV/FLAC headers, MP3 frame headers, etc.)
     → Encoder may duplicate IDTags pointer if it needs to modify them
Output:   Output file opened and headers written
Error if: Cannot write to output path → ConversionError = TRUE
```

**Step 6: Encode Loop**
```
Input:    decoder, live_dsp_chain, encoder, EncoderFluid
Process:
  1. percent_done = 0
  2. WHILE TRUE:
       a. block = decoder.DecodeBlock()
          // Returns STdBAudioBlock: { pData: pointer, Bytes: size, Blank0: bool }
          // block.pData is owned by decoder; invalid after next DecodeBlock() call
       
       b. IF block.pData was reallocated by decoder (format change):
              // CRITICAL: Must update all downstream DSPs with new buffer
              // This is why block ownership is explicit
       
       c. FOR EACH dsp IN live_dsp_chain (in order):
              block = dsp.PassAudioBlock(block)
              // DSP returns new block pointer; may differ from input
       
       d. encoder.EncodeBlock(block)
          // Encoder writes compressed data to output file
       
       e. percent_done = ComputePercent(decoder.Position, decoder.TotalSamples)
       f. SendProgressMsg("[filename] [percent_done]x[speed]")
       
       g. IF block.Blank0 == TRUE:
              BREAK  // Last block received
  3. RETURN
Output:   Encoded audio written to .tmp file; progress reported
Error if: Decoder error → ConversionError = TRUE; encoder closes cleanly
```

**Step 7: End Conversion**
```
Input:    decoder, live_dsp_chain, encoder, EncoderFluid
Process:
  1. decoder.Close(EncoderFluid)
     // Decoder releases file handles; may update fluid fields
     // Decoder object still valid after Close()
  
  2. FOR EACH dsp IN live_dsp_chain:
       dsp.EndConversion(EncoderFluid)
       // DSPs finalize state; may modify EncoderFluid.IDTags (e.g., add ReplayGain)
       // Non-live DSPs do NOT receive EndConversion
  
  3. should_write_tags = TRUE
     encoder.EndConversion(EncoderFluid, &should_write_tags)
     // Encoder:
     //   - Writes final encoded frames (flush encoder)
     //   - Writes seek table / index (FLAC, WMA)
     //   - Writes container footer
     //   - Sets should_write_tags:
     //       TRUE  = encoder already wrote ID tags
     //       FALSE = host should write tags separately
     // Encoder object still valid after EndConversion
Output:   Encoded file complete (including tags if should_write_tags = TRUE)
```

**Step 8: After Conversion**
```
Input:    live_dsp_chain, nonlive_dsp_chain, EncoderFluid, final_filename
Process:
  1. // Called for BOTH live and non-live DSPs
  2. FOR EACH dsp IN dsp_chain (live + non-live):
       dsp.AfterConversion(EncoderFluid)
       // NOW receives final (non-tmp) filename in EncoderFluid.OutFileName
       // Used for:
       //   - ReplayGain post-encoding scan (reads the just-encoded file)
       //   - Playlist writer (reads final filenames)
       //   - External program hooks (Run External DSP)
       //   - Copying folder.jpg to destination
       //   - CRC/MD5 calculation on final file
  3. // Called TWICE if EncoderFluid has 2 output filenames (split)
  4. // On error: called with .tmp filename instead of final
Output:   Post-processing complete
```

**Step 9: Cleanup and Finalization**
```
Input:    encoder, dsp_chain, decoder, EncoderFluid, ConversionError
Process:
  1. // Delete objects (reverse order of creation)
  2. DELETE encoder      → Encoder DLL freed via FreeLibrary()
  3. FOR EACH dsp IN dsp_chain (reverse order):
       DELETE dsp        → DSP DLL freed via FreeLibrary()
  4. DELETE decoder      → Decoder DLL freed via FreeLibrary()
  
  5. IF ConversionError == TRUE:
       a. DELETE dest_path + ".tmp"   // Remove incomplete output
       b. Log error to error file
       c. Exit process with non-zero code
  
  6. ELSE:
       a. // Atomic rename on same filesystem
       b. success = MoveFileEx(dest_path + ".tmp", dest_path, 
                               MOVEFILE_REPLACE_EXISTING | MOVEFILE_WRITE_THROUGH)
       c. IF NOT success:
              Log error: "Cannot rename .tmp to final file"
              DELETE dest_path + ".tmp"
              Exit with error code
       d. Exit process with code 0 (success)
```

---

## 6. INTEGRATION INTO CONVERSION PIPELINE

### 6.1 Full Pipeline Diagram

```
[Source File (.flac)]
     │
     │  CoreConverter.exe spawns
     ▼
[Decoder DLL]  ──── Decoder.Open() ────▶ [STdBEncoderFluid.IDTags = raw tags]
     │                                              │
     │  DecodeBlock() loop                         │ WaveFormat, AudioInfo
     ▼                                              ▼
[STdBAudioBlock] ──▶ [Live DSP Chain] ──▶ [Encoder DLL]
     │               PassAudioBlock()             EncodeBlock()
     │
     │ Non-live:                                          │
     │   Decode to ──▶ [Temp PCM file] ──▶ [Non-live DSPs]
     │   .pcm                                               │
     │                                                      ▼
     │                                              [Output .tmp file]
     │                                                      │
     │  EndConversion ─────────────────────────────────┘
     ▼                                                      │
[Decoder.Close()]                                            │
     │                                                      │
[DSP.EndConversion (live)] ──▶ [STdBEncoderFluid.IDTags updated (ReplayGain)]
     │                                                      │
[Encoder.EndConversion]                                       │
     │                                                      │
[DSP.AfterConversion] ◀─────────────────────────────────────
     │           (final filename now known)
     │
     ▼
[DELETE .tmp] or [MoveFileEx .tmp → final]
```

### 6.2 Tag Data Flow Detail

```
[1] Decoder.Open():
    Reads tags from source → STdBEncoderFluid.IDTags (raw bytes)
    Decoder does NOT parse tags; raw bytes are passed through

[2] Encoder.BeginConversion():
    Encoder reads IDTags from fluid
    If encoder handles its own tagging (MP3, FLAC): writes tags immediately or at EndConversion
    If encoder delegates: should_write_tags = FALSE

[3] DSP.EndConversion():
    DSP may ADD new tags to fluid.IDTags (e.g., ReplayGain DSP appends RG frames)
    DSP may MODIFY existing tags (e.g., ID Tag Processing DSP)
    DSP receives fluid pointer, modifies in place

[4] Encoder.EndConversion(ShouldWriteTags):
    If should_write_tags = TRUE: encoder writes final tags
    If should_write_tags = FALSE: host writes tags

[5] DSP.AfterConversion():
    Can write tags directly to final file (if encoder didn't)
    Can read from final file (ReplayGain post-scan)
    Receives FINAL filename (not .tmp)
```

---

## 7. DBPOWERAMP vs COMPETITORS COMPARISON

| Feature Aspect | DBpoweramp | FFmpeg | fre:ac (BoCA) | beets | Cuetools |
|---|---|---|---|---|---|
| Block-based streaming | Yes (DecodeBlock) | Yes (libavcodec) | Yes (DecoderComponent) | N/A | Partial |
| Non-live DSP support | Yes (temp file) | No | Partial (via temp file) | No | No |
| AfterConversion hook | Yes (post-rename) | No | No | No | No |
| ShouldWriteTags negotiation | Yes (encoder flag) | Implicit (format) | Implicit | N/A | N/A |
| EncoderFluid shared state | Yes (all stages) | No (separate tools) | Component config | N/A | N/A |
| Atomic rename from .tmp | Yes | No (direct write) | Yes (atomic) | N/A | N/A |
| Progress reporting mechanism | IPC to host | stdout pipe | IJobConsumer | N/A | N/A |
| Format negotiation pre-encode | Yes (NeedHQAudio) | No | Partial | N/A | N/A |
| Gapless info handling | Transparent pass-through | Manual with -write_xing | Via tag reader | N/A | Via libcue |
| Temp file management | System temp + custom | N/A | BoCA temp dir | N/A | N/A |

**DBpoweramp Advantage:** The `AfterConversion` hook with the final (non-tmp) filename is unique and critical for ReplayGain post-scan. FFmpeg requires a separate pass for ReplayGain because it lacks this hook. DBpoweramp can calculate ReplayGain on the final encoded file in a single conversion pass.

**DBpoweramp Limitation:** The pipeline is Windows-centric (CLI args, `MoveFileEx`, registry config). Cross-platform implementation requires reimplementing all platform-specific calls.

---

## 8. KNOWN BUGS, QUIRKS & COMMUNITY REPORTS

| Issue | DBpoweramp Version | Description | Workaround | Status |
|---|---|---|---|---|
| Race condition in start-encoder loop | R2025-12-25 | Fixed: "modernized start-encoder loop, fixed race condition if NoConvert is set" | Update | Fixed |
| CoreConverter deleting wrong file on overwrite | R2025-12-25 | Fixed: "Don't delete overwritten file, let MoveFile() replace it" | Update | Fixed |
| Floating point corruption through DSP | R17.7 (R2022) | "floating point put through certain DSP effects would corrupt the audio" | Update | Fixed |
| VST DSP adding extra samples | R2024-09-30 | "VST could add extra samples to end" | Update | Fixed |
| Gapless info handling for corrupted iTunSMPB | R2026-01-31 | "Discard malformed iTunSMPB data, play whole file even if gapless info tells to play small part of it" | Update | Fixed |
| Resampler SSRC off-by-one output sample count | R2025-11-12 | "ARDFTSRC removed output sample count +1" (SSRC had this bug) | Update | Fixed |
| Multi-CPU DSP settings icon overlay | R2026-01-31 | Cosmetic bug in settings page | None needed | Fixed |
| Write Metadata File DSP with floating point | R2023-11 | Sanitization issue | Update | Fixed |
| Corrupted FLAC with large art causing encode failure | R2025-12-25 | Fixed: "Fixed failure to encode FLAC if source file has large attached pictures" | Update | Fixed |

---

## 9. IMPLEMENTATION CHECKLIST

For a developer replicating this pipeline:

**Research Verification**
- [x] Complete stage sequence verified against official Spoon DSP Architecture specification
- [x] Live vs. non-live DSP split confirmed from official documentation
- [x] AfterConversion called with final filename confirmed
- [x] ShouldWriteTags flag behavior confirmed
- [x] IDTags passed through EncoderFluid confirmed (input DLL handles tagging)

**Implementation Requirements**
- [ ] Implement DecodeBlock() → PassAudioBlock() → EncodeBlock() loop as the core
- [ ] Separate live and non-live DSP chains; handle non-live by full decode to temp file first
- [ ] Pass shared state structure through all stages (equivalent to STdBEncoderFluid)
- [ ] Call DSP.EndConversion() BEFORE Encoder.EndConversion() (order matters)
- [ ] Call DSP.AfterConversion() AFTER Encoder.EndConversion() — now with final filename
- [ ] Implement ShouldWriteTags: if encoder writes tags, skip separate tag write
- [ ] DecodeBlock ownership: decoder's block is invalid after next DecodeBlock() call
- [ ] Atomic rename from .tmp → final on success; delete .tmp on error
- [ ] Progress reporting via IPC (pipe or messages) to host aggregator
- [ ] Handle mixed live/non-live DSP chains correctly
- [ ] Handle multiple output filenames (split output) — call AfterConversion twice
- [ ] On error, AfterConversion receives .tmp filename, not final

**Validation**
- [ ] Convert a file with ReplayGain DSP — verify ReplayGain tags are written to final file (not .tmp)
- [ ] Convert with Format Change DSP (resample) — verify output has new sample rate
- [ ] Convert with ID Tag Processing DSP — verify tag modifications appear in final output
- [ ] Error during conversion — verify .tmp file is deleted, source preserved
- [ ] Multi-encoder — verify each encoder sub-process gets its own output path
- [ ] Progress reporting — verify percentage increases from 0% to 100%
- [ ] Batch of 10 files on 4-core — verify 4 concurrent processes, then queue management

---

> **"If I implement exactly what this document describes, will a user converting files with my tool notice any difference from DBpoweramp in terms of tag content, tag completeness, or metadata preservation?"**

**Answer: Largely NO for tag content and completeness.** The pipeline produces identical audio and tag output regardless of the underlying process model. However, users WILL notice:
1. **ReplayGain precision** — DBpoweramp's AfterConversion hook lets it scan the final encoded file. A tool that can't scan the final file will have slightly different ReplayGain values (because lossy encoding changes the PCM slightly, and scanning the source vs. scanning the output gives different results).
2. **No PerfectTUNES integration** — After-conversion album art fixes, duplicate detection, and AccurateRip verification for existing files are not available.
3. **Encoding speed** — A proper multi-process implementation matches DBpoweramp's parallel scaling with CPU cores. A single-process implementation will be significantly slower on multi-core machines.
4. **Gapless conversion** — The pipeline transparently handles gapless playback info (encoder delay/padding). A tool that doesn't read/write these tags will produce files with small gaps between tracks on gapless-compatible players.
