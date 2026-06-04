# Architecture Overview and Component Map — DBpoweramp Behavior Research
> **Research Category:** Architecture
> **DBpoweramp Versions Studied:** R14 through R2026-04 (current)
> **Confidence Level:** High — documented (official Spoon developer forum posts, SDK docs) + spec inference
> **Primary Sources:** [Music Converter DSP Architecture - dBpoweramp Forum](https://forum.dbpoweramp.com/forum/other-topics/developers-corner/22043-music-converter-dsp-architecture) | [dBpoweramp Developer Compression SDK](https://www.dbpoweramp.com/developer-compression) | [dBpoweramp DSP Effects Help](https://dbpoweramp.com/Help/dMC/dsp) | [dBpoweramp Developer DSP Effects](https://www.dbpoweramp.com/developer-dmc-dsp-effects) | [Illustrate Versions Changelog](https://versions.dbpoweramp.com/?appid=2) | [dBpoweramp CD Ripper](https://www.dbpoweramp.com/cd-ripper) | [dBpoweramp Batch Multi-Encoder](https://dbpoweramp.com/Help/dMC/multiencoder)
> **Open-Source Reference:** BoCA (fre:ac backend) | TagLib | beets

---

## 1. TOPIC OVERVIEW & PURPOSE

### 1.1 What This Component Does

DBpoweramp Music Converter (dMC) is a commercial Windows/macOS audio conversion suite with a deliberately segmented, multi-process architecture. It is not a monolithic application — the conversion engine runs as isolated processes, codec and DSP functionality live in plugin DLLs, and the UI host manages the queue and progress. Understanding this architecture is prerequisite to implementing any DBpoweramp-equivalent behavior.

### 1.2 Why This Matters for Re-implementation

A naive re-implementation that treats conversion as a single-threaded, single-component operation will:
- Fail to replicate DBpoweramp's parallel batch throughput (one CoreConverter.exe per CPU core)
- Miss the DSP architecture's live vs. non-live split (which affects when file length must be known)
- Misunderstand how encoder settings are finalized (EncoderFluid's state at `EndConversion` differs from `BeginConversion`)
- Drop tag-writing decisions that happen in `AfterConversion` (after the final filename is known)

### 1.3 DBpoweramp's Design Philosophy for This Architecture

Spoon (the developer) has stated in forum posts that audio encoders "expect data to be encoded start to finish on one core," which is why each CoreConverter instance is single-threaded. Parallelism is achieved at the **process level** — one CoreConverter process per file per CPU core — rather than within a single process. This avoids thread-safety requirements in codec plugins and simplifies the plugin SDK.

---

## 2. OBSERVED / DOCUMENTED BEHAVIOR

### 2.1 Full Component Dependency Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────┐
│                        DBpoweramp Music Converter — Architecture                      │
│                                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────┐    │
│  │                          HOST PROCESS (dBpoweramp.exe)                        │    │
│  │                                                                              │    │
│  │   ┌──────────────────┐  ┌─────────────────┐  ┌────────────────────────┐  │    │
│  │   │   UI / Queue     │  │  DSP Effect     │  │  Registration &       │  │    │
│  │   │   Manager        │  │  Selector       │  │  License Manager      │  │    │
│  │   │  (File list,     │  │ (Effect order   │  │  (COM scripting,      │  │    │
│  │   │   progress bar,  │  │  UI, on/off)    │  │   license checks)     │  │    │
│  │   │   output naming) │  │                 │  │                       │  │    │
│  │   └────────┬─────────┘  └────────┬────────┘  └───────────┬────────────┘  │    │
│  │            │                    │                      │                 │    │
│  │            │  Spawns             │ Passes Effect       │                 │    │
│  │            ▼  CoreConverter.exe  │ Selection List      │                 │    │
│  └────────────┼─────────────────────┼─────────────────────┼─────────────────┘    │
│               │                     │                     │                      │
│               │ [Thread Pool: 1 to N concurrent processes] │                     │
│               │ N = user-selected CPU core count           │                     │
│               ▼                                             ▼                      │
│  ┌───────────────────────────────────────────────────────────────────────────┐   │
│  │           CoreConverter.exe Process (per-file, per-core)                   │   │
│  │                                                                           │   │
│  │  ┌──────────────────────────────────────────────────────────────────────┐  │   │
│  │  │                     Pipeline Coordinator (single thread)               │  │   │
│  │  │                                                                       │  │   │
│  │  │  ┌──────────────┐    ┌──────────────────────┐    ┌───────────────┐  │  │   │
│  │  │  │  Input Codec │    │     DSP Chain         │    │  Output Codec │  │  │   │
│  │  │  │  Plugin DLL  │───▶│  (0..N live + 0..M   │───▶│  Plugin DLL   │  │  │   │
│  │  │  │  (Decoder)   │    │   non-live effects)   │    │  (Encoder)    │  │  │   │
│  │  │  │              │◀───│  PassAudioBlock() +  │◀───│               │  │  │   │
│  │  │  │  DecodeBlock │    │  PassNonLive()       │    │  EncodeBlock  │  │  │   │
│  │  │  └──────────────┘    └──────────────────────┘    └───────────────┘  │  │   │
│  │  │         │                        │                       │             │  │   │
│  │  │         ▼                        ▼                       ▼             │  │   │
│  │  │  ┌────────────────────────────────────────────────────────────────┐ │  │   │
│  │  │  │              STdBEncoderFluid (shared state structure)          │ │  │   │
│  │  │  │  wchar_t* InFileName    wchar_t* OutFileName                   │ │  │   │
│  │  │  │  WAVEFORMATEX* pWaveFormat   STdBAudioInfo* pAudioInfo        │ │  │   │
│  │  │  │  void* IDTags (encoded tag bytes)  DWORD IDTagByteSize          │ │  │   │
│  │  │  │  int ConversionError                                        │ │  │   │
│  │  │  └────────────────────────────────────────────────────────────────┘ │  │   │
│  │  └──────────────────────────────────────────────────────────────────────┘  │   │
│  │                          │                                                   │   │
│  │  Writes .tmp file  ────▶ │ ────▶ Atomic rename to final filename             │   │
│  └───────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │                     PLUGIN DISCOVERY / LOADING LAYER                          │   │
│  │                                                                               │   │
│  │  Plugin Folder (Windows):                                                      │   │
│  │  C:\Program Files\Illustrate\dBpoweramp\encoder\    ← Output codecs (.dll)    │   │
│  │  C:\Program Files\Illustrate\dBpoweramp\decoder\   ← Input decoders (.dll)   │   │
│  │  C:\Program Files\Illustrate\dBpoweramp\DSPEffects\ ← DSP effect DLLs        │   │
│  │                                                                               │   │
│  │  Plugin Naming Convention:                                                     │   │
│  │    "FLAC (Lossless).dll"  →  appears in Convert To list as "FLAC (Lossless)"  │   │
│  │    "mp3 (Lame).dll"       →  appears as "mp3 (Lame)"                         │   │
│  │    "Windows Media Audio 10.DLL" → "Windows Media Audio 10"                    │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │                     PERFECTTUNES (Separate Process / Application)               │   │
│  │  Not part of conversion pipeline. Separate executable.                          │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────┐  ┌───────────────────┐   │   │
│  │  │  Album Art   │  │  ID Tags     │  │  De-Dup  │  │  AccurateRip      │   │   │
│  │  │  Manager     │  │  Editor      │  │  Finder  │  │  Verifier         │   │   │
│  │  └──────────────┘  └──────────────┘  └──────────┘  └───────────────────┘   │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                      │
│  ┌──────────────────────────────────────────────────────────────────────────────┐   │
│  │                     CD RIPPER (Separate Process)                                │   │
│  │  dBpowerampCDRipper.exe — AccurateRip integration, drive control, secure rip   │   │
│  │  Uses CoreConverter.exe for encoding ripped audio                              │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Behavior Evidence Sources

| Behavior Claim | Source | Confidence | Quote / Reference |
|---|---|---|---|
| Multi-process per-core architecture | [dBpoweramp Forum: Multi-core batch](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/34814-how-can-i-get-batch-file-conversion-to-use-multiple-cores) | High | "Each core converter would only run with 1 core, regardless of r version. It is not possible, the encoders expect data to be encoded start to finish on one core." |
| DSP Architecture — live vs non-live | [dBpoweramp Forum: DSP Architecture](https://forum.dbpoweramp.com/forum/other-topics/developers-corner/22043-music-converter-dsp-architecture) | High | Official Spoon specification of the full DSP pipeline sequence |
| Plugin DLL discovery by name | [dBpoweramp Developer Compression](https://www.dbpoweramp.com/developer-compression) | High | "The name given to your compression DLL (i.e. Example Converter.dll) is the one displayed by dMC's Convert To list." |
| PerfectTUNES is separate app | [dBpoweramp PerfectTUNES Help](https://dbpoweramp.com/Help/PerfectTUNES/) | High | Lists Album Art, ID Tags, De-Dup, AccurateRip, Replaygain as separate programs |
| CoreConverter.exe per file | [dBpoweramp Forum: Multi-core](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/34814-how-can-i-get-batch-file-conversion-to-use-multiple-cores) | High | "Batch Converter works by calling 4 independent converters, that is how Batch Converter works." |
| 64-bit only since R2025-11 | [Illustrate Versions R2025-11-12](https://versions.dbpoweramp.com/?appid=2) | High | "Goodbye 32 bit, we have retired 32 bit programs and now only supply 64 bit versions" |
| Multi-CPU Force DSP | [dBpoweramp DSP Help](https://dbpoweramp.com/Help/dMC/dsp) | High | "By default Music Converter / CD Ripper will encode using the maximum number of CPUs available" |

### 2.3 Configuration Options (User-Facing)

| UI Option | Location in UI | Effect | Default Value |
|---|---|---|---|
| Number of CPU cores for encoding | Convert page → Options → Encoding | Sets how many concurrent CoreConverter.exe processes | Auto (max available) |
| Encoding Priority | Options → Encoding | Windows process priority: Idle / Normal / Above Normal / High | Normal |
| DSP Effect Chain | DSP Effects tab | Ordered list of DSP effects to apply | Empty |
| Multi-CPU Force DSP | DSP Effects list | Force specific CPU core for this conversion | Auto |
| Temp folder location | Control Center → Advanced | Where non-live DSP temp files are written | System temp (%TEMP%) |
| Multi-Encoder | Separate encoder selector | Encode to multiple formats simultaneously | Off |
| Power Pack / Reference required for DSP | License | Enables DSP effects | Free trial / Purchase |

### 2.4 Behavior Differences by Version

| DBpoweramp Version | Behavior | Changed From |
|---|---|---|
| Pre-R14 | 32-bit build, single-core per encoder | n/a |
| R14 (2011) | AccurateRip v2, up to 16 simultaneous CPUs | Multi-core for CD Ripper |
| R16 (2016) | 64-bit available, 64-core support added | Internal memory pool per encoder |
| R17 (2020) | 20% faster multi-core, 64-core support | Rewritten multi-core management |
| R2023-12 | RIFF64 support, 32-bit PCM FLAC support | New codec additions |
| R2025-11 | 64-bit ONLY (32-bit retired), ARDFTSRC resampler, 64-bit float DSP | Major architectural cleanup |
| R2026-04 | Multi-CPU defaults to 4 encoders for high disk-usage formats | Changed default CPU count |

---

## 3. INTERNAL LOGIC (INFERRED / REVERSE-ENGINEERED)

### 3.1 Process/Thread Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                    HOST PROCESS (dBpoweramp.exe)                  │
│                                                                   │
│  UI Thread          │ Main worker thread (queue + progress)       │
│  (window messages)  │ (manages N CoreConverter processes)          │
│                     │                                            │
│  - File list       ─┼─▶ Queue Manager (populates file list)        │
│  - Progress bar    ─┼─▶ Progress Aggregator (sum of all %)        │
│  - DSP selection   ─┼─▶ DSP Config Serializer                    │
│                     │                                            │
│  Thread Pool (spawns N concurrent processes):                     │
│    for each CPU core available (up to user max):                 │
│      StartProcess("CoreConverter.exe", [args])                    │
│      Attach to IProgressSink for that process                    │
└──────────────────────────────────────────────────────────────────┘
         │
         │  N × IPC (WM_COPYDATA / named pipe / COM events)
         ▼
┌──────────────────────────────────────────────────────────────────┐
│         CoreConverter.exe Process (single thread per file)         │
│                                                                   │
│  Message Loop  ←── Progress events ──── To Host                   │
│                                                                   │
│  Single Thread executes the full pipeline:                        │
│    Decoder.Open() → [DSP setup] → Encoder.BeginConversion()      │
│    → Decode/DSP/Encode loop (single CPU core)                    │
│    → Encoder.EndConversion() → DSP.AfterConversion()              │
│    → Close() → Exit                                             │
│                                                                   │
│  WHY SINGLE-THREADED PER PROCESS:                                 │
│    Audio encoders (LAME, FLAC, AAC) are not thread-safe.         │
│    Parallelism at process level avoids thread-safety in codecs.   │
└──────────────────────────────────────────────────────────────────┘
```

**Key architectural constraint:** Each CoreConverter.exe process uses exactly 1 CPU core. Parallelism comes from running multiple CoreConverter.exe processes simultaneously (one per file, subject to max-core limit). A user with 8 CPU cores can convert 8 files in parallel.

### 3.2 Plugin Loading Mechanism

```pseudocode
FUNCTION LoadCodecPlugins():
    encoder_folder = "C:\\Program Files\\Illustrate\\dBpoweramp\\encoder\\"
    decoder_folder = "C:\\Program Files\\Illustrate\\dBpoweramp\\decoder\\"
    dsp_folder     = "C:\\Program Files\\Illustrate\\dBpoweramp\\DSPEffects\\"

    FOR EACH dll IN enumerate_files(encoder_folder, "*.dll"):
        name = strip_extension(dll.filename)   // "mp3 (Lame).dll" → "mp3 (Lame)"
        LoadLibrary(dll.full_path)
        plugin = {
            name: name,
            path: dll.full_path,
            type: "encoder",
            create_fn: GetProcAddress(hModule, "Create")
        }
        registerPlugin(plugin)

    FOR EACH dll IN enumerate_files(decoder_folder, "*.dll"):
        // Same pattern for decoders
        registerPlugin(plugin)

    FOR EACH dll IN enumerate_files(dsp_folder, "*.dll"):
        name = strip_extension(dll.filename)
        LoadLibrary(dll.full_path)
        plugin = {
            name: name,
            path: dll.full_path,
            type: "dsp",
            create_fn: GetProcAddress(hModule, "DSP_Create")
        }
        registerPlugin(plugin)
```

**Plugin discovery by name:** The DLL filename itself becomes the user-visible name in the Convert To list. No registry of plugin names is needed — the filename IS the identifier.

### 3.3 Plugin Interface Specification

**Compression/Encoder Plugin (.dll in `encoder/`):**

| Exported Function | Signature | Purpose |
|---|---|---|
| `Create` | `void* Create()` | Returns plugin instance pointer |
| `Info` | `void Info(char* buf, int buf_size)` | Returns format info string |
| `ShowConfig` | `void ShowConfig(HWND parent)` | Display config dialog |
| `GetLastError` | `char* GetLastError()` | Error message |

**Decoder/Input Plugin (.dll in `decoder/`):**

| Exported Function | Signature | Purpose |
|---|---|---|
| `Create` | `void* Create()` | Returns decoder instance |
| `GetFileTypes` | `char* GetFileTypes()` | Returns supported extensions |
| `ShowConfig` | `void ShowConfig(HWND parent)` | Config dialog |
| `GetLastError` | `char* GetLastError()` | Error message |

**DSP Plugin (.dll in `DSPEffects/`):**

| Exported Function | Signature | Purpose |
|---|---|---|
| `DSP_Create` | `void* DSP_Create(int sample_rate, int channels, int bits_per_sample)` | Create DSP instance |
| `DSP_Init` | `int DSP_Init(void* instance, void* encoder_fluid)` | Initialize with encoder settings |
| `DSP_EndConversion` | `void DSP_EndConversion(void* instance, void* encoder_fluid)` | Finalize before encoder end |
| `DSP_AfterConversion` | `void DSP_AfterConversion(void* instance, void* encoder_fluid, wchar_t* final_filename)` | Post-encoding hook |
| `DSP_Free` | `void DSP_Free(void* instance)` | Release instance |

### 3.4 STdBEncoderFluid — The Central Shared Data Structure

This is the critical shared structure passed through all pipeline stages:

```pseudocode
STRUCT STdBEncoderFluid:
    // FILE IDENTIFIERS
    wchar_t* InFileName      // Source filename with extension, or "-" for stdin
    wchar_t* OutFileName     // Output filename (.tmp.xxx during processing)
                             // After AfterConversion: updated to final filename

    // AUDIO FORMAT
    WAVEFORMATEX* pWaveFormat  // PCM format: sample_rate, channels, bits_per_sample

    // ENCODER SETTINGS
    char* CompressionType    // e.g., "mp3 (Lame)"
    char* CompressionCLI     // e.g., "-b 320 -q 0"
    char* StreamDetails      // Codec-specific settings string

    // AUDIO METADATA
    STdBAudioInfo* pAudioInfo // { file_size, length_msec, ... }
    void* IDTags             // Pointer to encoded ID tag bytes
    DWORD IDTagByteSize      // Size of ID tag data
    void* AudioProps         // Additional audio properties
    DWORD AudioPropsByteSize

    // ERROR STATE
    int ConversionError      // READ-ONLY from plugin perspective
                             // Set to true if any plugin detected an error
```

### 3.5 Tag Data Flow

```
┌────────────────────────────────────────────────────────────────────────────┐
│                        TAG DATA FLOW THROUGH PIPELINE                       │
│                                                                             │
│  [1] Decoder.Open(InFileName)                                              │
│      │  Decoder reads IDTags from source file → IDTags pointer             │
│      │  Decoder sets AudioProps (duration, bitrate, etc.)                   │
│      ▼                                                                     │
│  [2] STdBEncoderFluid.IDTags is populated by decoder                        │
│      │  (raw encoded tag bytes, not parsed)                               │
│      ▼                                                                     │
│  [3] Encoder.BeginConversion(EncoderFluid)                                 │
│      │  Encoder reads IDTags from fluid                                    │
│      │  Encoder may duplicate IDTags and modify (DSP can change tags)      │
│      ▼                                                                     │
│  [4] Encode Loop: Decoder → DSP → Encoder                                  │
│      │  Audio blocks flow; tags are held in fluid                          │
│      ▼                                                                     │
│  [5] DSP.EndConversion(EncoderFluid)                                       │
│      │  DSP can modify IDTags in fluid (e.g., ReplayGain writes here)     │
│      ▼                                                                     │
│  [6] Encoder.EndConversion(EncoderFluid, &ShouldWriteTags)                 │
│      │  Encoder finalizes output file                                      │
│      │  ShouldWriteTags: boolean — does encoder write its own tags?        │
│      ▼                                                                     │
│  [7] DSP.AfterConversion(EncoderFluid)                                     │
│      │  Called with FINAL filename (not .tmp)                              │
│      │  Called twice if EncoderFluid has 2 filenames (split output)       │
│      │  Used for: ReplayGain post-scan, external file operations          │
│      ▼                                                                     │
│  [8] .tmp file is atomically renamed to final OutFileName                  │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### 3.6 UI Progress Reporting Mechanism

```
Host (dBpoweramp.exe)                    CoreConverter.exe (per file)
      │                                          │
      │──── Start conversion ────────────────────▶│
      │     (CLI args + DSP list)                │
      │                                          │
      │◀──── Progress messages ───────────────────│
      │     (WM_COPYDATA / pipe:                 │
      │      "[filename] [percent] [speed]")      │
      │                                          │
      │◀──── Completion code ─────────────────────│
      │     (0=success, non-zero=error)          │
```

Progress is percentage + encoding speed (e.g., "12x real-time"). Host aggregates N concurrent progress streams into a single progress bar.

### 3.7 File Locking Strategy During Batch Operations

| Strategy | Implementation |
|---|---|
| Source file locking | CoreConverter.exe opens source file with `FILE_SHARE_READ`, never `FILE_SHARE_WRITE`. Safe for concurrent reads. |
| Destination file | Written to `output_path.tmp` first. On success, `MoveFileEx()` atomic rename. No locking needed. |
| Collision handling | R2025-12: "CoreConverter: Don't delete overwritten file, let MoveFile() replace it; prevent file being erased if MoveFile() fails" |
| Temp file isolation | Each CoreConverter.exe has isolated temp file scope. No temp file sharing. |
| Multi-encoder isolation | Each encoder sub-process gets its own output path. Source file is read-only shared. |

### 3.8 Edge Case Handling

| Edge Case | DBpoweramp Behavior |
|---|---|
| N files, M cores, N > M | Queue system: M files start simultaneously, remaining queued. As each completes, next from queue starts. |
| Output folder same as source | R2025-12 fix: CoreConverter no longer deletes file before MoveFile(), preventing source overwrite race |
| DSP changes output format (e.g., channel remix) | Non-live path: full decode → DSP → re-encode. DSP signals format change in `EncoderFluid.pWaveFormat` |
| Encoder delay/padding (gapless) | Decoder reads from source (including gapless info), encoder writes gapless info to output. Pipeline is transparent for gapless data. |
| Corrupt source file | `ConversionError = true` in fluid. Error logged. `.tmp` deleted. Next file continues. |
| Disk full during write | `ConversionError = true`. More verbose error message since R2024-09. `.tmp` deleted. |

---

## 4. OPEN-SOURCE EQUIVALENT IMPLEMENTATION

### 4.1 BoCA — fre:ac's Component Architecture

BoCA (fre:ac's backend) is the most structurally similar open-source project. Both use a component-based plugin architecture with decoders, encoders, DSP, and tagger components. Key parallels:

```cpp
// BoCA equivalents to DBpoweramp concepts:
Registry& boca = Registry::Get();                          // Plugin discovery registry
DecoderComponent* decoder = boca.CreateDecoderForStream();  // Decoder plugin creation
EncoderComponent* encoder = boca.CreateComponentByID();     // Encoder by ID
DSPComponent* dsp = boca.CreateDSPComponent();              // DSP effect
Track trackInfo = ...;                                      // Equivalent to STdBAudioInfo
Format audioFormat = decoder->GetWaveFormat();              // WAVEFORMATEX equivalent
stream->SetFilter(decoder);                                 // Pipeline assembly
LockComponent(decoder);  // Thread safety for non-thread-safe codecs
```

| DBpoweramp Concept | BoCA Equivalent | Notes |
|---|---|---|
| CoreConverter.exe process | fre:ac engine thread | Single-threaded per conversion |
| STdBEncoderFluid | `Track` + `Format` + component state | Shared across decode/DSP/encode |
| Plugin DLL | `.so` / `.dylib` component | Registered with BoCA registry |
| DSP live/non-live | `IsLive()` method per DSP | Same design pattern |
| Encoder settings in fluid | Component configuration | Passed to encoder at BeginConversion |
| Progress callback | IJobConsumer interface | Aggregates from all conversion threads |

### 4.2 FFmpeg as Pipeline Backbone

```bash
# DBpoweramp-equivalent conversion using FFmpeg (simplified model):
# The DBpoweramp pipeline: Decoder → DSP → Encoder
# maps to FFmpeg as:

# Decode + DSP (volume normalize example):
ffmpeg -i input.flac \
  -af "volume=replaygain=track" \
  -c:a pcm_f32le            # Internal: 32-bit float PCM
  -f wav -                  # Pipe to encoder

# Encode:
ffmpeg -i -                # Read from pipe (32-bit float WAV)
  -c:a libmp3lame -b:a 320k \
  -id3v2_version 3         # ID3v2.3 default
  output.mp3

# Tag writing (separate pass with FFmpeg metadata):
ffmpeg -i output.mp3 \
  -metadata title="Track Title" \
  -metadata artist="Artist Name" \
  -c:a copy \
  output_tagged.mp3
```

**Key difference:** FFmpeg does not have an equivalent to the `STdBEncoderFluid` mechanism — tag data must be carried separately through the pipeline, and tag-writing must happen as a separate stage (FFmpeg's `-metadata` on encode pass only works for certain formats).

### 4.3 Libraries That Implement This Architecture

| Library | Language | License | Relevant Component | Notes |
|---|---|---|---|---|
| BoCA | C/C++ | GPLv2 | `DecoderComponent`, `EncoderComponent`, `DSPComponent`, `TaggerComponent` | Most structurally similar to DBpoweramp |
| TagLib | C++ | LGPL | `TagLib::FileRef`, `TagLib::Tag` | Tag reading/writing only |
| FFmpeg/libavcodec | C | LGPL/GPL | Decoder/encoder pipeline | Reference codec implementation |
| beets | Python | MIT | `beets.mediafile.MediaFile` | Tag mapping reference |
| SoX | C | GPLv2 | `sox_format_t`, DSP effects chain | DSP processing reference |

---

## 5. DETAILED IMPLEMENTATION SPECIFICATION

### 5.1 Component Discovery and Registration

**Step 1: Plugin Directory Scanning**
```
Input:   Plugin folder path (e.g., "C:\...\encoder\")
Process: 
  1. Create directory iterator for "*.dll" files
  2. For each DLL:
     a. LoadLibrary() with DONT_RESOLVE_DLL_REFERENCES (just check loadable)
     b. GetProcAddress(hModule, "Create") — verify export exists
     c. If Create exists: GetProcAddress(hModule, "Info") for display name
     d. Store { path, name, type: encoder|decoder|dsp }
  3. Filter by type — subfolders determine type, not internal detection
Output:  Sorted list of available plugins for Convert To menu
Error if: DLL cannot be loaded → skip, log warning, continue
```

**Step 2: Plugin Instantiation Per Conversion**
```
Input:   Selected encoder name, DSP effect names list
Process:
  1. LoadLibrary(selected_encoder.dll)
  2. Create encoder instance: encoder.Create()
  3. For each DSP in order:
     a. LoadLibrary(dsp_name.dll)
     b. Create DSP instance: DSP_Create(sample_rate, channels, bits)
     c. Store DSP instance in ordered list
  4. LoadLibrary(decoder_for_source.dll) — determined by source extension
  5. Create decoder instance: decoder.Create()
Output:  Fully instantiated pipeline ready for conversion
Error if: Missing decoder for source format → abort with "unsupported format"
```

### 5.2 Conversion Loop — Complete Sequence

```pseudocode
FUNCTION ConvertFile(source_path, output_path, encoder_name, dsp_list, options):
  
  // === PHASE 0: VALIDATION ===
  ASSERT file_exists(source_path)
  ASSERT decoder_exists_for_extension(get_extension(source_path))
  ASSERT encoder_exists(encoder_name)
  
  // === PHASE 1: PLUGIN INSTANTIATION ===
  decoder = CreateDecoder(get_extension(source_path))
  encoder = CreateEncoder(encoder_name, options.compression_cli)
  
  dsp_chain = []
  FOR EACH dsp_name IN dsp_list:
    dsp_instance = CreateDSP(dsp_name, options.sample_rate, options.channels, options.bits)
    dsp_chain.APPEND(dsp_instance)
  
  // === PHASE 2: FLUID CREATION ===
  fluid = NEW STdBEncoderFluid()
  fluid.InFileName    = source_path
  fluid.OutFileName   = output_path + ".tmp"  // temp until AfterConversion
  fluid.CompressionType = encoder_name
  fluid.CompressionCLI  = options.compression_cli
  fluid.IDTags          = NULL
  fluid.IDTagByteSize   = 0
  fluid.ConversionError = FALSE
  
  // === PHASE 3: DECODER OPEN ===
  audio_info = decoder.Open(source_path, fluid.IDTags_ptr, fluid.AudioProps_ptr)
  fluid.pWaveFormat  = audio_info.waveformat   // Source PCM format
  fluid.pAudioInfo   = audio_info             // Duration, bitrate, etc.
  
  // === PHASE 4: DSP SETUP ===
  FOR EACH dsp IN dsp_chain:
    IF dsp.IsNonLive() THEN
      // Will process from temp file later
      dsp.MarkNonLive()
    ELSE
      dsp.BeginConversion(fluid)  // LIVE DSPs only
    END IF
  
  // === PHASE 5: ENCODER SETUP ===
  // Non-live DSPs: decode entire source to temp PCM file first
  FOR EACH dsp IN dsp_chain WHERE dsp.IsNonLive():
    DecodeEntireFileToTemp(decoder, temp_pcm_path)
    dsp.PassNonLive(temp_pcm_path, fluid)  // Process temp file
  END FOR
  
  // Now start the encoder
  encoder.BeginConversion(fluid)
  
  // === PHASE 6: ENCODE LOOP ===
  WHILE (block = decoder.DecodeBlock()) IS NOT NULL:
    FOR EACH dsp IN dsp_chain WHERE dsp.IsLive():
      block = dsp.PassAudioBlock(block)  // Modify PCM block
    END FOR
    encoder.EncodeBlock(block)
    ReportProgress(fluid.pAudioInfo.percent_complete)
  END WHILE
  
  decoder.Close(fluid)  // Note: decoder object still exists after Close
  
  // === PHASE 7: END CONVERSION ===
  FOR EACH dsp IN dsp_chain WHERE dsp.IsLive():
    dsp.EndConversion(fluid)  // LIVE DSPs only
  END FOR
  
  should_write_tags = TRUE
  encoder.EndConversion(fluid, &should_write_tags)
  
  // === PHASE 8: AFTER CONVERSION ===
  FOR EACH dsp IN dsp_chain:  // BOTH live and non-live
    dsp.AfterConversion(fluid)  // NOW receives final (non-tmp) filename
  END FOR
  
  // === PHASE 9: CLEANUP ===
  DELETE encoder
  DELETE ALL dsp instances
  DELETE decoder
  
  // === PHASE 10: ATOMIC RENAME ===
  IF fluid.ConversionError THEN
    DELETE fluid.OutFileName  // Remove .tmp file
    RETURN ERROR
  ELSE
    MoveFileEx(fluid.OutFileName, output_path, MOVEFILE_REPLACE_EXISTING)
    RETURN SUCCESS
  END IF
```

### 5.3 Batch Conversion Architecture

```pseudocode
FUNCTION BatchConvert(file_list, output_format, options):
  
  max_concurrent = GetUserSelectedCoreCount()  // Default: all available cores
  
  // Special case: high disk I/O formats (WAV) use fewer cores
  IF output_format.uses_wave_pcm THEN
    max_concurrent = MIN(max_concurrent, 4)  // Default 4 since R2026-04
  END IF
  
  semaphore = NewSemaphore(max_concurrent)
  results = []
  
  // Spawn all files with concurrency limit
  FOR EACH source_file IN file_list:
    semaphore.ACQUIRE()  // Block if at max concurrent
    THREAD_POOL.SUBMIT(() => {
      TRY:
        result = ConvertFile(source_file, output_path, ...)
        results.APPEND(result)
      FINALLY:
        semaphore.RELEASE()
    })
  END FOR
  
  THREAD_POOL.WAIT_ALL()  // Block until all complete
  RETURN AggregateResults(results)
```

---

## 6. INTEGRATION INTO CONVERSION PIPELINE

### 6.1 Pipeline Position

```
[Source File]
     │
     ▼
[HOST: Queue + Core Spawn]     ← Architecture layer (multi-process coordination)
     │
     │  CoreConverter.exe process created per file
     ▼
[Decoder DLL — Open + DecodeBlock]     ← STdBEncoderFluid.IDTags populated
     │
     ▼
[DSP Chain — PassAudioBlock + EndConversion + AfterConversion]     ← Tag modifications
     │
     ▼
[Encoder DLL — BeginConversion + EncodeBlock + EndConversion]     ← Output file written
     │
     ▼
[HOST: .tmp → Atomic Rename]     ← Finalization
     │
     ▼
[Output File]
```

### 6.2 Data Passed Between Stages

| Stage Transition | Data Structure | Key Fields |
|---|---|---|
| Host → Decoder | `STdBEncoderFluid` | `InFileName` |
| Decoder → DSP | `STdBAudioBlock` | `pData` (PCM samples), `Bytes`, `Blank0` (is last block) |
| DSP → Encoder | `STdBAudioBlock` | Modified PCM after DSP effects |
| Encoder → Host | `STdBEncoderFluid` | Updated `OutFileName`, `ConversionError` |
| AfterConversion | `STdBEncoderFluid` | Final filename (non-tmp) |

---

## 7. DBPOWERAMP vs COMPETITORS COMPARISON

| Feature Aspect | DBpoweramp | fre:ac | beets | FFmpeg CLI | Mp3tag |
|---|---|---|---|---|---|
| Multi-process parallelism | Yes (N CoreConverter processes) | Yes (thread pool) | No (single file) | No | N/A |
| Per-core isolation | Yes (each file = 1 process) | Thread-based | N/A | N/A | N/A |
| Plugin discovery | Filename-as-name in folders | Component registry | N/A | N/A | N/A |
| DSP live/non-live split | Yes (both supported) | Partial | N/A | No (one-pass only) | N/A |
| STdBEncoderFluid shared state | Yes (all stages access) | Via component config | N/A | No | N/A |
| AfterConversion hook | Yes (post-rename) | AfterEncode hook | N/A | No | N/A |
| Atomic rename on completion | Yes (.tmp → final) | Yes | N/A | No (direct write) | N/A |
| COM scripting API | Yes (dMCScripting) | No | Python scripting | No | No |
| PerfectTUNES integration | Separate app | None | Plugins | N/A | N/A |
| 64-bit only | Yes (R2025+) | Yes | Yes | N/A | Yes |
| Max CPU cores | 64+ | 64+ | 1 | 1 | N/A |

**DBpoweramp Advantage:** Process-level isolation eliminates all thread-safety requirements from codec plugins. Every codec plugin is inherently thread-safe because each conversion runs in its own process address space.

**DBpoweramp Limitation:** Process spawning overhead makes it unsuitable for very small files (startup cost per file). Not suitable for single-file rapid conversion workflows where latency matters.

---

## 8. KNOWN BUGS, QUIRKS & COMMUNITY REPORTS

| Issue | DBpoweramp Version | Description | Workaround | Status |
|---|---|---|---|---|
| Race condition in start-encoder loop | R2025-12-25 | Fixed: "Music Converter modernized start-encoder loop, fixed race condition if NoConvert is set" | Update to R2025-12-25+ | Fixed |
| CoreConverter deleting wrong file | R2025-12-25 | Fixed: "Don't delete overwritten file, let MoveFile() replace it; prevent file being erased if MoveFile() fails" | Update to R2025-12-25+ | Fixed |
| Floating point PCM corruption through DSP | R17.7 (R2022) | "floating point put through certain DSP effects would corrupt the audio" | Update past R17.7 | Fixed |
| VST DSP adding extra samples to end | R2024-09-30 | "VST could add extra samples to end" | Update past R2024-09-30 | Fixed |
| Multi-CPU DSP settings icon overlay | R2026-01-31 | "Multi-CPU DSP when select line on settings page the icon would be overlaid with wrong icon" | Cosmetic only | Fixed |
| Error 380 on Windows 10/11 updates | R2026-01-31 | Fix for "error 380" bug introduced by Windows updates | Update to R2026-01-31+ | Fixed |
| Write metadata file DSP with floating point | R2023-11 | "Write Metadata File DSP effect would sanitize output" | Update past R2023-11 | Fixed |

**Community reports from Hydrogenaudio, spoon.net forums:**
- [Multi-core batch thread](https://forum.dbpoweramp.com/forum/dbpoweramp/music-converter/34814-how-can-i-get-batch-file-conversion-to-use-multiple-cores): Spoon confirms each CoreConverter uses exactly 1 core; parallelism is at the process level.
- [DSP Architecture thread](https://forum.dbpoweramp.com/forum/other-topics/developers-corner/22043-music-converter-dsp-architecture): Official Spoon post with complete DSP pipeline specification including all function calls.

---

## 9. IMPLEMENTATION CHECKLIST

For a developer replicating this architecture:

**Research Verification**
- [x] Multi-process architecture confirmed by multiple sources (forum posts + SDK docs)
- [x] Per-core isolation confirmed (each CoreConverter.exe = 1 process = 1 core)
- [x] STdBEncoderFluid structure inferred from official DSP Architecture specification
- [x] Plugin discovery by filename confirmed from compression SDK documentation
- [x] PerfectTUNES confirmed as separate application, not part of conversion pipeline

**Implementation Requirements**
- [ ] Implement process-level parallelism (not thread-level) — spawn N processes for N cores
- [ ] Each process runs single-threaded conversion pipeline
- [ ] Implement live vs. non-live DSP split (non-live needs full decode to temp file first)
- [ ] Pass `STdBEncoderFluid`-equivalent structure through all pipeline stages
- [ ] Implement AfterConversion hook AFTER atomic rename is known
- [ ] Tag data flows through fluid structure, not as separate pass
- [ ] Atomic rename from .tmp → final path on success; delete .tmp on error
- [ ] Discover plugins by enumerating DLL filenames in known folders
- [ ] Plugin DLL filename becomes user-visible "Convert To" menu name

**Validation**
- [ ] Batch conversion with 8 files on 8-core machine produces 8 concurrent CoreConverter.exe processes
- [ ] Conversion speed on N cores ≈ N × single-core speed (subject to disk I/O)
- [ ] High disk-usage format (WAV) limits concurrency to prevent disk thrashing
- [ ] Error in one conversion does not affect others (process isolation)
- [ ] DSP AfterConversion receives final (non-tmp) filename

---

> **"If I implement exactly what this document describes, will a user converting files with my tool notice any difference from DBpoweramp in terms of tag content, tag completeness, or metadata preservation?"**

**Answer: Architecture-level differences — users will NOT notice tag/metadata differences**, because the architecture (multi-process vs. single-process) does not affect the logical conversion pipeline — decode → DSP → encode → tag-write produces identical output regardless of whether the pipeline runs as threads or processes. However, users WILL notice:
1. **Batch throughput scaling** — A proper multi-process implementation matches DBpoweramp's near-linear scaling with CPU cores. A naive single-threaded implementation will be N× slower on N-core machines.
2. **Error isolation** — Process-level isolation means one corrupt file doesn't crash the whole batch. A single-process implementation may crash the entire batch.
3. **PerfectTUNES is not available** — This is a separate product. Any re-implementation cannot claim "DBpoweramp equivalence" without also noting the absence of PerfectTUNES features (De-Dup, Album Art Manager, AccurateRip verification for existing files).
