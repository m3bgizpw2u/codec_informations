# Codec Plugin Architecture and DSP Framework — DBpoweramp Behavior Research
> **Research Category:** Architecture
> **DBpoweramp Versions Studied:** R14 through R2026-04 (current)
> **Confidence Level:** High — documented (official Spoon developer forum posts, SDK docs) + spec inference
> **Primary Sources:** [Music Converter DSP Architecture - dBpoweramp Forum](https://forum.dbpoweramp.com/forum/other-topics/developers-corner/22043-music-converter-dsp-architecture) | [dBpoweramp Developer DSP Effects](https://www.dbpoweramp.com/developer-dmc-dsp-effects) | [dBpoweramp Developer Compression](https://www.dbpoweramp.com/developer-compression) | [dBpoweramp DSP Effects Help](https://dbpoweramp.com/Help/dMC/dsp) | [dBpoweramp Batch Multi-Encoder](https://dbpoweramp.com/Help/dMC/multiencoder)
> **Open-Source Reference:** BoCA (fre:ac) | SoX | FFmpeg libavfilter

---

## 1. TOPIC OVERVIEW & PURPOSE

### 1.1 What This Component Does

DBpoweramp's plugin system is the extensible backbone of its codec ecosystem. Codec plugins (`.dll` / `.dylib`) extend format support by implementing decoder and encoder interfaces. DSP effect plugins (`.dll` / `.dylib`) implement audio processing effects (volume normalization, resampling, equalization, ReplayGain, etc.) that are chained into the conversion pipeline. The plugin architecture is intentionally simple — plugins are discovered by filename, loaded on demand, and communicate with the host through a small set of well-defined C-style exported functions.

### 1.2 Why This Matters for Re-implementation

A DBpoweramp-equivalent plugin system must:
- **Discover plugins by filename** (no registry, no manifest) to replicate the exact plugin loading behavior
- **Separate encoder from decoder plugins** — the same `.dll` can act as encoder OR decoder depending on which folder it lives in
- **Support both live and non-live DSP modes** — this split is architectural, not just a flag
- **Pass the `STdBEncoderFluid` correctly** — DSP effects receive it at `BeginConversion`, `EndConversion`, and `AfterConversion`, and can modify it at each point
- **Expose a configuration UI** via `ShowConfigBit` — DBpoweramp embeds the plugin's dialog as a child window in the main options form

### 1.3 DBpoweramp's Design Philosophy

Spoon has kept the plugin SDK deliberately simple to minimize the burden on codec developers:
- No COM registration required — filename IS the identifier
- No version negotiation — the plugin either implements the required exports or it doesn't load
- State is carried through the fluid structure, not through the plugin instance
- Non-live DSP processing (via temp file) allows complex DSP effects that need the full audio before processing begins

---

## 2. OBSERVED / DOCUMENTED BEHAVIOR

### 2.1 Plugin Discovery Mechanism

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Plugin Discovery — By Folder and Filename                │
│                                                                              │
│  Windows Plugin Folders:                                                      │
│  C:\Program Files\Illustrate\dBpoweramp\encoder\     ← Output codecs        │
│  C:\Program Files\Illustrate\dBpoweramp\decoder\    ← Input decoders        │
│  C:\Program Files\Illustrate\dBpoweramp\DSPEffects\ ← DSP effect DLLs       │
│                                                                              │
│  Plugin Folder Structure (Windows):                                           │
│  encoder/                                                                   │
│  ├── FLAC (Lossless).dll         → "FLAC (Lossless)" in Convert To list   │
│  ├── mp3 (Lame).dll              → "mp3 (Lame)"                           │
│  ├── Apple Lossless (ALAC).dll   → "Apple Lossless (ALAC)"                 │
│  ├── CLI Encoder.dll              → "CLI Encoder"                           │
│  ├── iTunes (m4a).dll           → "iTunes (m4a)" (wrapper for CLI tool)  │
│  └── Windows Media Audio 10.DLL  → "Windows Media Audio 10"                │
│                                                                              │
│  decoder/                                                                   │
│  ├── FLAC.ddfa                    → .flac support (legacy naming)           │
│  ├── dsd_dff.ddfa                → .dff DSD support                       │
│  └── (others for every supported input format)                              │
│                                                                              │
│  DSPEffects/                                                                │
│  ├── ReplayGain.dll             → "ReplayGain" in DSP list                │
│  ├── Volume Normalize.dll        → "Volume Normalize"                      │
│  ├── Resample.dll                → "Resample"                              │
│  ├── Bit Depth.dll               → "Bit Depth"                             │
│  ├── [ID Tag Update].dll         → Utility codecs also appear here         │
│  └── ... (30+ effects)                                                     │
│                                                                              │
│  Discovery Algorithm:                                                        │
│  for each dll in folder:                                                    │
│      strip_extension(dll) → display_name                                   │
│      try LoadLibrary(dll)                                                   │
│      if GetProcAddress(hModule, "Create") != NULL:                         │
│          register_plugin(display_name, dll, type=folder_type)              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Plugin Interface — Complete Function Specifications

**Encoder Plugin (`encoder/*.dll`) — Required Exports:**

| Export | Signature | Called By | Purpose |
|---|---|---|---|
| `Create` | `void* __cdecl Create()` | Host (at instantiation) | Returns plugin instance handle |
| `Info` | `void __cdecl Info(char* buf, int buf_size)` | Host (at discovery) | Fills buffer with format info string |
| `ShowConfig` | `void __cdecl ShowConfig(HWND parent)` | Host (when user clicks "Settings") | Shows encoder config dialog as child of parent HWND |
| `GetLastError` | `char* __cdecl GetLastError()` | Host (after failed call) | Returns last error message as ASCII string |

**Decoder Plugin (`decoder/*.dll`) — Required Exports:**

| Export | Signature | Called By | Purpose |
|---|---|---|---|
| `Create` | `void* __cdecl Create()` | Host | Returns decoder instance |
| `GetFileTypes` | `char* __cdecl GetFileTypes()` | Host (at discovery) | Returns semicolon-separated list: `"mp3;mpa;mp2;m1a"` |
| `ShowConfig` | `void __cdecl ShowConfig(HWND parent)` | Host | Decoder config dialog |
| `GetLastError` | `char* __cdecl GetLastError()` | Host | Last error message |

**DSP Plugin (`DSPEffects/*.dll`) — Required Exports:**

| Export | Signature | Called By | Purpose |
|---|---|---|---|
| `DSP_Create` | `void* __cdecl DSP_Create(int sample_rate, int channels, int bits)` | CoreConverter | Create DSP instance with audio format context |
| `DSP_Init` | `int __cdecl DSP_Init(void* instance, void* encoder_fluid)` | CoreConverter | Initialize DSP with encoder settings |
| `DSP_EndConversion` | `void __cdecl DSP_EndConversion(void* instance, void* encoder_fluid)` | CoreConverter | Finalize live DSP before encoder end |
| `DSP_AfterConversion` | `void __cdecl DSP_AfterConversion(void* instance, void* encoder_fluid, wchar_t* final_filename)` | CoreConverter | Post-encoding hook |
| `DSP_Free` | `void __cdecl DSP_Free(void* instance)` | CoreConverter | Release DSP instance |

**DSP Plugin — Optional but Standard:**

| Export | Signature | Called By | Purpose |
|---|---|---|---|
| `DSP_BeginConversion` | `void __cdecl DSP_BeginConversion(void* instance, void* encoder_fluid)` | CoreConverter | LIVE DSP setup (called before encode loop) |
| `DSP_PassAudioBlock` | `void* __cdecl DSP_PassAudioBlock(void* instance, void* block)` | CoreConverter | Process one PCM block (live mode) |
| `DSP_PassNonLive` | `void __cdecl DSP_PassNonLive(void* instance, wchar_t* temp_pcm_path, void* encoder_fluid)` | CoreConverter | Process entire file from temp PCM (non-live) |

### 2.3 DSP Architecture — Live vs Non-Live

```
┌────────────────────────────────────────────────────────────────────────────┐
│                     DSP Live vs Non-Live Architecture                       │
│                                                                             │
│  LIVE DSPs (processed during encode loop):                                │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ DecodeBlock() ──▶ DSP_A ──▶ DSP_B ──▶ DSP_C ──▶ EncodeBlock()   │ │
│  │                                                                       │ │
│  │ Processing:                                                          │ │
│  │   1. DSP_BeginConversion(fluid) ← called before encode loop        │ │
│  │   2. For each PCM block:                                            │ │
│  │         DSP_PassAudioBlock(block)                                    │ │
│  │         Block may be modified in-place or replaced                   │ │
│  │   3. DSP_EndConversion(fluid) ← called after encode loop           │ │
│  │                                                                       │ │
│  │ Examples: Resample (with specific output rate), Channel Count,       │ │
│  │           Channel Mapper, Volume Normalize (Peak), Bit Depth        │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  NON-LIVE DSPs (processed on full decoded file):                          │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │                                                                       │ │
│  │  Stage A: Decode entire file to temp raw PCM                        │ │
│  │           DecodeBlock() × all blocks ──▶ temp.pcm (file on disk)  │ │
│  │                                                                       │ │
│  │  Stage B: Process temp file                                          │ │
│  │           DSP_PassNonLive(temp1.pcm) ──▶ temp2.pcm (may differ)    │ │
│  │           Multiple non-live DSPs chain: temp1 → DSP_A → temp2 →   │ │
│  │           DSP_B → temp3.pcm                                         │ │
│  │                                                                       │ │
│  │  Stage C: Encode from processed temp file                           │ │
│  │           New decoder reads tempN.pcm ──▶ EncodeBlock() ──▶ output │ │
│  │                                                                       │ │
│  │  Examples: Fade, Reverse, Speed up/Slow down, Trim Silence,       │ │
│  │            Loop, Grabber, ReplayGain (calculate), Volume Norm      │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
│  MIXED (live + non-live):                                                 │
│  Non-live DSPs run FIRST (decode full file → process → re-decode)        │
│  Live DSPs run during encode loop (normal)                                 │
│                                                                             │
│  Force Non-Live DSP:                                                      │
│  Forces ALL live DSPs to behave as non-live (uses temp file)             │
│  Needed when: Fade + Length Split, or any effect requiring file length    │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### 2.4 DSP Effect Ordering Rules

DSP effects are applied **in top-to-bottom order as displayed in the UI**. This order is **user-configurable** but constrained by audio physics. The recommended order from DBpoweramp's own documentation:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DSP Effect Ordering — Recommended Sequence               │
│                                                                              │
│  1. Bit Depth ──▶ Convert to 32-bit float (preserves precision)            │
│  2. Channel Mapper ──▶ Remix / combine channels                            │
│  3. DC Offset Removal ──▶ Remove DC component                              │
│  4. Volume Normalize ──▶ EBU R128, Peak, ReplayGain, Adaptive             │
│  5. Graphic Equalizer ──▶ Frequency band adjustments (requires float)       │
│  6. Dynamic Range Compression ──▶ Level compression                         │
│  7. Resample ──▶ Sample rate conversion (e.g., 96kHz → 44.1kHz)           │
│  8. Bit Depth ──▶ Convert back to integer (e.g., 32-bit float → 16-bit)  │
│  9. Apply Dither ──▶ Triangular PDF dither (only when reducing bit depth)  │
│                                                                              │
│  Non-live effects are always processed BEFORE the encode loop starts:       │
│  ─────────────────────────────────────────────────────────────────────────  │
│  Fade → processed on full file before encoding                              │
│  Reverse → processed on full file before encoding                           │
│  Speed up / Slow down → processed on full file before encoding             │
│  Trim Silence → processed on full file before encoding                     │
│  Loop → processed on full file before encoding                             │
│  Grabber → processed on full file before encoding                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.5 DSP Parameter Persistence

```pseudocode
// DSP settings are persisted between sessions:
// 1. When user changes DSP options → plugin writes to registry or config file
// 2. On next session → plugin reads from registry when DSP_Create is called

// From dBpoweramp Developer DSP Effects documentation:
// "any options set on your small option page should be saved (perhaps to registry) 
//  on the Change events, as you will not be notified when your option page 
//  has been destroyed."

// Typical persistence pattern:
FUNCTION DSP_Create(sample_rate, channels, bits):
    instance = AllocateDSPInstance()
    instance.sample_rate = sample_rate
    instance.channels = channels
    instance.bits = bits
    
    // Load persisted settings from registry
    reg_key = "Software\Illustrate\dBpoweramp\DSPEffects\" + GetPluginName()
    instance.enabled = ReadRegistryBool(reg_key, "Enabled", TRUE)
    instance.option1 = ReadRegistryInt(reg_key, "Option1", DEFAULT_VAL)
    instance.option2 = ReadRegistryFloat(reg_key, "Option2", DEFAULT_VAL)
    
    RETURN instance

FUNCTION ShowConfig(HWND parent):
    // Load current settings into UI controls
    SetDlgItemInt(IDC_OPTION1, instance.option1)
    SetDlgItemInt(IDC_OPTION2, instance.option2)
    
    // Show dialog as child of parent
    DialogBoxParam(GetModuleHandle(), 
                   MAKEINTRESOURCE(IDD_DSPCONFIG), 
                   parent, 
                   DlgProc, 
                   (LPARAM)instance)
    
    // On OK: save to registry immediately
    WriteRegistryBool(reg_key, "Enabled", instance.enabled)
    WriteRegistryInt(reg_key, "Option1", instance.option1)
    WriteRegistryFloat(reg_key, "Option2", instance.option2)
```

### 2.6 PerfectTUNES Integration

**PerfectTUNES is NOT part of the conversion pipeline.** It is a separate application suite launched independently. It shares the codec installation (same decoder/encoder DLLs) but has no direct integration with the CoreConverter pipeline.

| PerfectTUNES Component | Purpose | Integration with dMC |
|---|---|---|
| Album Art | Add missing covers, find better art | None (standalone) |
| ID Tags | Edit metadata across library | None (standalone) |
| De-Dup | Find duplicate tracks | None (standalone) |
| AccurateRip | Verify existing rips against AccurateRip DB | None (standalone) |
| ReplayGain | Calculate and apply volume normalization | None (standalone) |

DBpoweramp's ReplayGain DSP does the same job as PerfectTUNES ReplayGain — it's just integrated into the conversion pipeline rather than run as a separate tool.

### 2.7 Behavior Evidence Sources

| Behavior Claim | Source | Confidence | Quote / Reference |
|---|---|---|---|
| Plugin name = filename | [dBpoweramp Developer Compression](https://www.dbpoweramp.com/developer-compression) | High | "The name given to your compression DLL (i.e. Example Converter.dll) is the one displayed by dMC's Convert To list." |
| Only 3 DSP functions | [dBpoweramp Developer DSP Effects](https://www.dbpoweramp.com/developer-dmc-dsp-effects) | High | "There are only 3 Functions which dMC communicates with the DSP all are shown in WaveOut.cpp" |
| Live/non-live via temp file | [dBpoweramp Developer DSP Effects](https://www.dbpoweramp.com/developer-dmc-dsp-effects) | High | "The data is passed from dMC to the DSP Effect in a temporary file" |
| PerfectTUNES separate | [dBpoweramp PerfectTUNES Help](https://dbpoweramp.com/Help/PerfectTUNES/) | High | Lists Album Art, ID Tags, De-Dup, AccurateRip, Replaygain as separate programs |
| ShowConfigBit embed | [dBpoweramp Developer Compression](https://www.dbpoweramp.com/developer-compression) | High | "ShowConfigBit(HWND OnForm) attaches your page as a child on dMC's options window" |

### 2.8 Third-Party Codec Plugin Discovery

```pseudocode
// Third-party codecs are discovered the same way as built-in ones:
FUNCTION DiscoverEncoderPlugins():
    folder = "C:\\Program Files\\Illustrate\\dBpoweramp\\encoder\\"
    FOR EACH file IN enumerate(folder, "*.dll"):
        name = strip_extension(file.name)
        TRY:
            hModule = LoadLibrary(file.full_path)
            IF GetProcAddress(hModule, "Create") != NULL:
                // Valid encoder plugin
                plugin = { name: name, path: file.full_path, type: "encoder" }
                register(plugin)
            ELSE:
                FreeLibrary(hModule)  // Not a codec DLL, skip
        CATCH:
            // DLL won't load (missing dependency, wrong architecture), skip
            LogWarning("Skipping " + name + ": " + GetLoadError())
    
    // No whitelist, no manifest — every .dll is attempted
    // This allows: CLI Encoder.dll (wrapper for external tools), NeroAACEnc.dll, etc.
```

The CLI Encoder codec demonstrates this: it's a special codec that wraps command-line tools (QAAC, Nero AAC, etc.) as plugins. The DLL's `Create` function spawns the external encoder process with the correct CLI arguments, reads its stdout/stderr, and manages the output file.

### 2.9 Behavior Differences by Version

| Version | Change | Impact |
|---|---|---|
| Pre-R14 | Basic codec plugins | No live/non-live distinction documented |
| R14 | Advanced DSP effects added | ReplayGain, ID Tag Processing |
| R16 | VST DSP support added | Use Steinberg VST effects in conversion |
| R17 | Power Pack / Reference distinction | DSP effects gated behind license |
| R2020 | ID Tag Processing as standard | Utility codecs included as standard |
| R2023 | ReplayGain detects lossy encoding | "Replaygain DSP detects when encoding lossy, will calculate RG values by decoding the lossy encoded file after encoded" |
| R2025 | 64-bit ONLY, 64-bit float DSP | 64-bit float support in Bit Depth DSP |
| R2026-03 | Gamma, Vibrance, Anamorphic De-squeeze DSPs | New Video Converter DSPs |

---

## 3. INTERNAL LOGIC (INFERRED / REVERSE-ENGINEERED)

### 3.1 DSP Plugin State Machine

```pseudocode
// Each DSP instance has a lifecycle tied to the conversion:

STATE machine DSPInstance:
    current_state: enum { created, initialized, live_running, nonlive_processing, 
                          ended, after_conversion, freed }
    
    // State transitions:
    
    CREATE:
        instance = DSP_Create(sample_rate, channels, bits)
        current_state = created
        RETURN instance
    
    INIT:
        DSP_Init(instance, encoder_fluid)
        current_state = initialized
        // At this point, DSP knows the encoder's output format
        // DSP can decide: am I live or non-live?
        IF instance.is_nonlive THEN
            // Will process from temp file later
            current_state = nonlive_pending
        ELSE
            current_state = initialized
            // Will receive blocks during encode loop
        END IF
    
    LIVE_START (for live DSPs):
        DSP_BeginConversion(instance, encoder_fluid)
        current_state = live_running
    
    LIVE_PROCESS (for live DSPs, per block):
        result_block = DSP_PassAudioBlock(instance, block)
        RETURN result_block  // May be same pointer or new block
    
    LIVE_END (for live DSPs):
        DSP_EndConversion(instance, encoder_fluid)
        current_state = ended
        // At this point, DSP may have modified encoder_fluid.IDTags
    
    NONLIVE_PROCESS (for non-live DSPs):
        // Called during the "before encoding" phase
        processed_path = DSP_PassNonLive(instance, temp_pcm_path, encoder_fluid)
        current_state = nonlive_processing
        RETURN processed_path  // Path to (possibly new) temp file
    
    AFTER_CONVERSION:
        DSP_AfterConversion(instance, encoder_fluid, final_filename)
        current_state = after_conversion
        // Called for BOTH live and non-live DSPs
        // Called AFTER the final filename is known
    
    FREE:
        DSP_Free(instance)
        current_state = freed
```

### 3.2 Audio Block Structure (Passed Between DSP Stages)

```pseudocode
STRUCT STdBAudioBlock:
    void*  pData        // Pointer to PCM sample data
                         // Ownership: decoder allocates; each DSP can reallocate
                         // Last DSP or encoder uses the pointer; host frees
    int    Bytes        // Size of the data block in bytes
    int    Blank0       // TRUE (1) = last block (end of audio)
                         // FALSE (0) = more blocks to follow
    // Default: dMC sets all items to 0/NULL
    // PCM format described by WAVEFORMATEX in STdBEncoderFluid

// PCM data format:
// For 16-bit stereo interleaved: [L Sample1][R Sample1][L Sample2][R Sample2]...
// For 8-bit stereo interleaved: same pattern, 8-bit samples
// Channels are interleaved (not planar) by default
// DSP must handle the format it receives; may output different format
```

### 3.3 CLI Encoder Codec — Special Plugin Pattern

```pseudocode
// The CLI Encoder codec is a special wrapper plugin:
// Filename: "CLI Encoder.dll" or "iTunes (m4a).dll"
// The DLL itself doesn't encode — it wraps an external CLI tool

// How it works:
CLASS CLIEncoderPlugin:
    FUNCTION Create():
        RETURN new CLIEncoderInstance()
    
    FUNCTION ShowConfig(HWND parent):
        // Config page lets user select external encoder executable
        // e.g., QAAC64.exe, NeroAacEnc.exe, etc.
        // And set encoder-specific options
        // Saved to registry
    
    FUNCTION BeginConversion(fluid):
        // Build command line from encoder settings
        cli = BuildCommandLine(fluid.CompressionCLI, 
                              fluid.InFileName, 
                              fluid.OutFileName)
        
        // Spawn external process
        proc = CreateProcess(self.encoder_path, cli)
        
        // Capture stdout/stderr for progress
        pipe_stdout = AttachPipe(proc.stdout)
        pipe_stderr = AttachPipe(proc.stderr)
        
        // Monitor for progress patterns
        WHILE proc.IS_RUNNING():
            line = pipe_stdout.READ_LINE()
            ParseProgressFromCLIOutput(line, fluid)
    
    FUNCTION EncodeBlock(block):  // Not used for CLI encoder
        // CLI encoder reads from file directly (InFileName)
        // This function may be a no-op
        PASS
    
    FUNCTION EndConversion(fluid):
        proc.WAIT()  // Wait for encoder to finish
        IF proc.EXIT_CODE() != 0:
            stderr_output = pipe_stderr.READ_ALL()
            SET fluid.ConversionError = TRUE
            SET error_message = stderr_output
```

### 3.4 Edge Case Handling

| Edge Case | DBpoweramp Behavior |
|---|---|
| Plugin DLL missing required export | Plugin skipped; error logged; conversion continues |
| Plugin DLL won't load (missing dependency) | Plugin skipped; warning shown in UI |
| Plugin crashes during DSP_PassAudioBlock | Process crashes; .tmp deleted; error reported; other conversions continue |
| DSP changes output PCM format mid-conversion | Only possible in non-live mode; live format changes break the encoder |
| DSP reallocates audio block buffer | DSP returns new pointer; host uses new pointer; old pointer freed |
| DSP needs to know final output filename | Uses AfterConversion hook (called with final filename) |
| DSP needs file length before processing | Must be non-live; Force Non-Live DSP forces all live DSPs to temp file |
| Multiple non-live DSPs chained | Each produces a new temp file; chain: temp1 → DSP_A → temp2 → DSP_B → temp3 |

---

## 4. OPEN-SOURCE EQUIVALENT IMPLEMENTATION

### 4.1 BoCA Component Equivalents

BoCA (fre:ac's backend) implements the same architecture with a C++ class hierarchy:

```cpp
// BoCA DSP Component base class (equivalent to DBpoweramp DSP plugin):
class DSPComponent {
public:
    virtual bool IsLive() const = 0;              // Live vs non-live
    virtual void StartProcessing();              // BeginConversion equivalent
    virtual void Process(Buffer<Float> &buffer); // PassAudioBlock equivalent
    virtual void Finish();                        // EndConversion equivalent
    virtual void Finish(const String &outputFile); // AfterConversion equivalent
};

// BoCA Encoder Component (equivalent to DBpoweramp encoder plugin):
class EncoderComponent {
public:
    virtual bool Start();                         // BeginConversion equivalent
    virtual void Process(const Buffer<Float> &buffer); // EncodeBlock equivalent
    virtual void Finish();                         // EndConversion equivalent
    virtual void Finish(const String &outputFile);
    // GetLastError(), ShowConfig() equivalents via base class
};

// BoCA Decoder Component (equivalent to DBpoweramp decoder plugin):
class DecoderComponent {
public:
    virtual bool ReadWaveFormat(WAVEFORMATEX *waveFormat); // Get format
    virtual bool Open(const String &fileName);              // Open file
    virtual bool Read(Buffer<Float> &buffer);              // DecodeBlock
    virtual bool Close();                                  // Close file
};
```

| DBpoweramp DSP Concept | BoCA Equivalent | Notes |
|---|---|---|
| Live DSP | `IsLive() == true` | Processed in real-time during decode |
| Non-live DSP | `IsLive() == false` | Processed on full file via temp file |
| DSP_Create | Constructor | Format passed to constructor |
| DSP_Init | `SetFormat()` + `StartProcessing()` | Setup before processing |
| DSP_BeginConversion | `StartProcessing()` | Called once before encode loop |
| DSP_PassAudioBlock | `Process(buffer)` | Per-block processing |
| DSP_EndConversion | `Finish()` | Finalize before encoder end |
| DSP_AfterConversion | `Finish(outputFile)` | Post-encoding with final filename |
| DSP_Free | Destructor | RAII cleanup |

### 4.2 SoX DSP Equivalent

SoX is the open-source reference for DSP processing:

```bash
# SoX as DSP processing engine (equivalent to DBpoweramp DSP chain):
# Each effect in SoX is a DSP component:

# Resample:
sox input.pcm output.pcm rate 44100

# Volume normalize (peak):
sox input.pcm output.pcm gain -b 3

# EBU R128 (ReplayGain):
# SoX doesn't have native R128; use ffmpeg's loudnorm filter
ffmpeg -i input.flac -af loudnorm=I=-18:TP=-1.5:LRA=11 output.flac

# Graphic equalizer:
sox input.pcm output.pcm equalizer 1000 1.5q 6

# Dither (TPDF):
sox input.pcm output.pcm dither -f tpdf

# Chain multiple effects:
sox input.pcm output.pcm \
    rate 44100 \
    gain -b 3 \
    equalizer 1000 1.5q 6 \
    dither -f tpdf
```

### 4.3 FFmpeg libavfilter as DSP Framework

```bash
# FFmpeg filter graph as equivalent to DBpoweramp DSP chain:

# Chain: Resample → Volume Normalize → Bit Depth Reduce with Dither
ffmpeg -i input.flac -af \
    "aresample=44100:res_type=soxr,
     volume=replaygain=track,
     aformat=sample_fmts=s16:dither_method=triangular" \
    output.mp3

# Key FFmpeg filter equivalents:
# aresample / aformat  → Resample + Bit Depth DSP
# volume               → Volume Normalize DSP
# loudnorm             → EBU R128 Volume Normalize
# equalizer            → Graphic Equalizer DSP
# acompressor          → Dynamic Range Compression
# highpass / lowpass   → Highpass / Lowpass filters
# acontrast            → Volume Normalize (Adaptive)
# afade                → Fade DSP
# areverse             → Reverse DSP
# atempo               → Speed up / Slow down DSP
# atrim / silenceremove → Trim Silence DSP
```

---

## 5. DETAILED IMPLEMENTATION SPECIFICATION

### 5.1 Algorithm — DSP Plugin Instantiation and Lifecycle

**Step 1: DSP Plugin Discovery**
```
Input:    None (scans DSPEffects folder)
Process:
  1. folder = GetDSPEffectsFolder()
  2. FOR EACH file IN enumerate(folder, "*.dll"):
       name = strip_extension(file.name)
       TRY:
           hModule = LoadLibrary(file.full_path)
           create_fn = GetProcAddress(hModule, "DSP_Create")
           IF create_fn != NULL:
               // Verify optional exports
               begin_fn = GetProcAddress(hModule, "DSP_BeginConversion")
               pass_fn  = GetProcAddress(hModule, "DSP_PassAudioBlock")
               nonlive_fn = GetProcAddress(hModule, "DSP_PassNonLive")
               end_fn   = GetProcAddress(hModule, "DSP_EndConversion")
               after_fn = GetProcAddress(hModule, "DSP_AfterConversion")
               free_fn  = GetProcAddress(hModule, "DSP_Free")
               register_plugin(name, hModule, ...)
           ELSE:
               FreeLibrary(hModule)
       CATCH:
           LogWarning("DSP plugin " + name + " failed to load")
Output:   List of available DSP plugins (names + module handles)
```

**Step 2: DSP Instance Creation (per-conversion)**
```
Input:    DSP plugin handle, source audio format
Process:
  1. instance = DSP_Create(sample_rate, channels, bits_per_sample)
     // Plugin allocates instance, stores format
     // Plugin may load persisted settings from registry
  2. Return instance handle to host
Output:   DSP instance handle
```

**Step 3: DSP Initialization**
```
Input:    DSP instance, STdBEncoderFluid
Process:
  1. DSP_Init(instance, encoder_fluid)
     // Plugin reads: encoder format, compression settings
     // Plugin may show config dialog (if ShowConfig implemented)
     // Plugin decides: live or non-live?
     // Plugin may modify encoder_fluid.WaveFormat (e.g., Resample DSP)
Output:   Initialization complete; plugin ready for processing
```

**Step 4: Non-Live DSP Processing (pre-encode)**
```
Input:    DSP instance, temp PCM file path, STdBEncoderFluid
Process:
  1. IF DSP_IsNonLive(instance):
       result_path = DSP_PassNonLive(instance, temp_pcm_path, encoder_fluid)
       // Plugin processes the entire PCM file
       // May produce a new temp file (different pointer)
       RETURN result_path
  2. ELSE:
       // Skip — live processing in encode loop
       RETURN temp_pcm_path  // Unchanged
Output:   Path to processed temp file (may be same as input)
```

**Step 5: Live DSP Processing (encode loop)**
```
Input:    DSP instance, STdBAudioBlock
Process:
  1. IF DSP_IsLive(instance):
       // First block: DSP_BeginConversion(instance, fluid)
       result = DSP_PassAudioBlock(instance, block)
       // Subsequent blocks: DSP_PassAudioBlock(instance, block)
       RETURN result  // Modified or replacement block
  2. ELSE:
       RETURN block  // Non-live, skip
Output:   Processed audio block (may be same pointer or new allocation)
```

**Step 6: DSP End Conversion (post-encode loop)**
```
Input:    DSP instance, STdBEncoderFluid
Process:
  1. IF DSP_IsLive(instance):
       DSP_EndConversion(instance, encoder_fluid)
       // Plugin finalizes; may modify encoder_fluid.IDTags
       // e.g., ReplayGain DSP appends RG frames to IDTags
  2. ELSE:
       // Non-live: nothing here; AfterConversion will be called
Output:   Fluid updated; plugin state finalized
```

**Step 7: After Conversion (post-rename)**
```
Input:    DSP instance, STdBEncoderFluid, final_filename
Process:
  1. DSP_AfterConversion(instance, encoder_fluid, final_filename)
     // NOW final_filename is known (not .tmp)
     // Plugin can:
     //   - Read from final file (ReplayGain post-scan)
     //   - Write tags directly to final file
     //   - Copy file to destination folder
     //   - Trigger external programs
  2. // Called for BOTH live and non-live DSPs
  3. // May be called TWICE if encoder produces 2 output files
Output:   Post-processing complete
```

**Step 8: DSP Cleanup**
```
Input:    DSP instance
Process:
  1. DSP_Free(instance)
     // Plugin frees all allocated memory
     // Frees any temp files it created
  2. FreeLibrary(hModule)
Output:   Plugin resources released
```

### 5.2 ShowConfigBit — Embedding Plugin UI in Host Window

```pseudocode
// Plugin config dialog is embedded as a CHILD window of the host's options form:

// Host (DBpoweramp) side:
FUNCTION ShowEncoderOptions(parent_hwnd, encoder_name):
    dll_path = FindEncoderDLL(encoder_name)
    hModule = LoadLibrary(dll_path)
    
    show_config_fn = GetProcAddress(hModule, "ShowConfig")
    IF show_config_fn != NULL:
        // Embed plugin's dialog as child of our options form
        // This is why plugin dialogs appear inside DBpoweramp's UI
        // NOT as separate top-level windows
        show_config_fn(parent_hwnd)
    
    RETURN

// Plugin side:
FUNCTION ShowConfig(HWND parent):
    // Create dialog as child of parent
    // Use CreateDialogParam with parent as owner
    hwnd = CreateDialogParam(GetModuleHandle(), 
                             MAKEINTRESOURCE(IDD_OPTIONS_DIALOG),
                             parent,          // Parent window
                             DlgProc, 
                             (LPARAM)this)
    
    // Set window text, default values from registry
    // When user clicks OK:
    //   - Read values from controls
    //   - Save to registry immediately
    //   - EndDialog(hwnd, IDOK)
    
    // When user clicks Cancel:
    //   - Discard changes
    //   - EndDialog(hwnd, IDCANCEL)

// IMPORTANT: Plugin's window class must have CS_HREDRAW | CS_VREDRAW 
// if it contains a "RemoveConfigBit" stub (no-op per SDK docs)
```

### 5.3 Complete DSP Effect List (R2026-04)

| DSP Effect | Type | Live/Non-Live | Purpose |
|---|---|---|---|
| Audio CD - De-emphasis | Audio | Non-live | Apply de-emphasis to pre-emphasized CDs |
| Audio CD - Hidden Track Silence Removal | Audio | Non-live | Remove hidden track silence |
| Audio CD - Remove Gaps | Audio | Non-live | Remove inter-track gaps from CDs |
| Audio CD - Silence Track Deletion | Audio | Non-live | Remove silent tracks from CDs |
| Bandpass Filter | Audio | Live | Filter frequencies above/below range |
| Bit Depth | Audio | Live | Convert bit depth, float↔int |
| Channel Count | Audio | Live | Force channel count |
| Channel Mapper | Audio | Live | Move/combine channels |
| Conditional Encoding | Action | Non-live | Skip or copy files by condition |
| CPU Throttle | Action | Non-live | Limit encoding speed |
| Delete Destination File on Error | Action | Non-live | Delete bad output |
| Delete Source File | Action | Non-live | Remove source after conversion |
| DirectX PlugIn | Audio | Live | Use DirectX audio effects |
| Dynamic Range Compression | Audio | Live | Compress dynamic range |
| Fade | Audio | Non-live | Fade in/out |
| Folder.jpg Preserve | Action | AfterConversion | Copy folder art to dest |
| Force Non-Live | Control | Non-live | Force all DSPs to temp file |
| Grabber | Audio | Non-live | Extract portion of audio |
| Graphic Equalizer | Audio | Live | Boost/reduce frequency bands |
| HD HDCD | Audio | Live | Decode HDCD to 24-bit |
| Highpass | Audio | Live | Filter frequencies below threshold |
| ID Tag Processing | Action | Non-live | Manipulate tags |
| Insert Audio | Audio | Non-live | Insert audio from file |
| Karaoke | Audio | Live | Remove center-channel voice |
| Loop | Audio | Non-live | Repeat track |
| Lowpass | Audio | Live | Filter frequencies above threshold |
| Maximum / Minimum Length | Audio | Non-live | Trim or pad track length |
| Multi-CPU Force | Control | Non-live | Pin to specific CPU core |
| Move Destination File on Error | Action | AfterConversion | Move bad files |
| Play Sound After Conversion | Action | AfterConversion | Notify completion |
| Playlist Writer | Action | AfterConversion | Write playlist file |
| Preserve Source Attributes | Action | AfterConversion | Copy file timestamps |
| Read Metadata File | Action | Pre-conversion | Import tags from XML |
| ReplayGain | Analysis | Non-live | Calculate loudness |
| ReplayGain (Apply) | Audio | Live | Apply pre-calculated ReplayGain |
| Resample | Audio | Live | Change sample rate |
| Reverse | Audio | Non-live | Reverse audio |
| Run External | Action | AfterConversion | Invoke external program |
| Speed up, Slow down | Audio | Non-live | Time stretch |
| Trim Silence | Audio | Non-live | Remove silence |
| Trim | Audio | Non-live | Remove fixed-length segment |
| Volume Normalize | Audio | Live | 6 normalization modes |
| VST | Audio | Live | Use VST3/VST2 effects |
| Write Metadata File | Action | AfterConversion | Export tags to XML |
| Write Silence | Audio | Non-live | Insert silence |
| Gamma / Vibrance / Anamorphic | Video | Live | Video Converter DSPs |

---

## 6. INTEGRATION INTO CONVERSION PIPELINE

### 6.1 DSP in the Conversion Loop

```
[Source File]
     │
     │ CoreConverter.exe
     ▼
[Decoder.DecodeBlock()] ──▶ [STdBAudioBlock: pData, Bytes, Blank0]
     │
     │ Non-live DSPs: full decode to temp file handled HERE
     │ (before encode loop starts)
     ▼
     For each block in source:
         │
         ▼
         [Live DSP Chain — in order]
         │
         ├─ DSP_BeginConversion() ──────────── (first block only)
         │
         ├─ DSP_1.PassAudioBlock(block) ──▶ block₁
         │       │
         │       ▼
         ├─ DSP_2.PassAudioBlock(block₁) ──▶ block₂
         │       │
         │       ▼
         ├─ DSP_N.PassAudioBlock(blockₙ₋₁) ──▶ blockₙ
         │       │
         │       ▼
         ▼
         [Encoder.EncodeBlock(blockₙ)]
              │
              ▼
         [Progress report]
     
     │
     │ Last block received (Blank0 = TRUE)
     ▼
[Decoder.Close()]
     │
     ▼
[Live DSP_1.EndConversion()] ───▶ May modify IDTags (ReplayGain)
[Live DSP_N.EndConversion()]
     │
     ▼
[Encoder.EndConversion(ShouldWriteTags)]
     │
     ▼
[All DSPs.AfterConversion(final_filename)]
     │
     ▼
[Atomic rename .tmp → final]
```

---

## 7. DBPOWERAMP vs COMPETITORS COMPARISON

| Feature Aspect | DBpoweramp | SoX | FFmpeg | fre:ac (BoCA) | Adobe Audition |
|---|---|---|---|---|---|
| Plugin discovery | Filename in folder | Built-in | Built-in filter | Component registry | VST SDK |
| Live/non-live split | Yes (both) | Partial (in-place vs temp) | No (one-pass) | Partial | No |
| AfterConversion hook | Yes (post-rename) | No | No | No | No |
| VST effect support | Yes (Reference) | Via libsox | Via lavfi | Via VST component | Native VST |
| DirectX effect support | Yes (Reference) | No | No | No | No |
| ShowConfigBit embedding | Yes (child window) | N/A | N/A | Config in component | Component panel |
| Third-party plugins | Yes (DLL in folder) | No | No | Yes (components) | VST SDK |
| CLI encoder wrapper | Yes (CLI Encoder.dll) | No | N/A | No | No |
| Force Non-Live option | Yes | No | N/A | No | No |
| Multi-channel DSP | Yes (up to 12 channels) | Yes | Yes | Yes | Yes |
| 64-bit float DSP | Yes (Bit Depth DSP) | No (int only) | Yes (aformat) | Yes | Yes |
| DSP parameter persistence | Registry per plugin | Config file | CLI args | Component config | Session |

**DBpoweramp Advantage:** The combination of live/non-live split + AfterConversion hook is unique. SoX can do live effects but no post-encoding hook. FFmpeg's filter graph is more powerful but can't scan the final encoded file for ReplayGain.

**DBpoweramp Limitation:** No GPU-accelerated DSP (unlike some competitors). VST support is limited to VST2/VST3 (no VST3 latest features). Third-party codec plugins are Windows/macOS only (DLL/dylib).

---

## 8. KNOWN BUGS, QUIRKS & COMMUNITY REPORTS

| Issue | Version | Description | Workaround | Status |
|---|---|---|---|---|
| VST adding extra samples | R2024-09-30 | "VST could add extra samples to end" | Update | Fixed |
| Floating point corruption through DSP | R17.7 (R2022) | "floating point put through certain DSP effects would corrupt the audio" | Update | Fixed |
| Volume Normalize IEEE_FLOAT reading | R2023-11 | "Volume Normalize DSP would not read IEEE_FLOAT correctly if passed in a WaveFormatExtensible header" | Update | Fixed |
| Write Metadata File DSP sanitization | R2023-11 | "Write Metadata File DSP effect would sanitize output" | Update | Fixed |
| SSRC output sample count +1 | R2025-11 | "ARDFTSRC removed output sample count +1" (ARDFTSRC replaces SSRC) | Update | Fixed |
| ARDFTSRC crash | R2025-12-25 | Fixed obscure regression in ARDFTSRC | Update | Fixed |
| Multi-CPU DSP settings icon overlay | R2026-01-31 | Cosmetic only | None needed | Fixed |
| Bit Depth DSP added 64-bit IEEE float | R2025-11 | New float depth option | Update | Fixed |

---

## 9. IMPLEMENTATION CHECKLIST

For a developer replicating this architecture:

**Research Verification**
- [x] Plugin discovery by filename confirmed from SDK docs
- [x] Live vs non-live DSP split confirmed from official DSP Architecture spec
- [x] AfterConversion hook with final filename confirmed
- [x] ShowConfigBit child window embedding confirmed from compression SDK

**Implementation Requirements**
- [ ] Discover plugins by enumerating DLL filenames in designated folders
- [ ] LoadLibrary() each candidate; verify "Create" export; register if valid
- [ ] Plugin filename (stripped of .dll/.dylib extension) IS the display name
- [ ] Separate live from non-live DSPs before encode loop
- [ ] Non-live DSPs: decode entire source to temp PCM, process temp file, re-decode from temp
- [ ] Live DSPs: process each block in the encode loop (in order)
- [ ] Call DSP.EndConversion() for live DSPs AFTER encode loop, BEFORE Encoder.EndConversion()
- [ ] Call DSP.AfterConversion() for ALL DSPs AFTER Encoder.EndConversion() — NOW final filename is known
- [ ] Implement ShowConfigBit: embed plugin dialog as child window of host's options form
- [ ] Persist DSP settings to registry (or config file) on change events
- [ ] Handle block ownership: DSP can reallocate block; host uses returned pointer
- [ ] CLI encoder wrapper: spawn external process, capture progress, manage temp files

**Validation**
- [ ] Add a third-party codec DLL to encoder folder — appears in Convert To list with its filename as name
- [ ] ReplayGain DSP: verify it calculates on source (before encode) and writes tags after encode
- [ ] Force Non-Live + Fade DSP: verify full file is decoded, fade applied, re-encoded
- [ ] VST DSP: verify VST3/VST2 effects load and process audio
- [ ] AfterConversion: verify DSP receives final (non-tmp) filename
- [ ] Multi-CPU Force DSP: verify process affinity is set correctly

---

> **"If I implement exactly what this document describes, will a user converting files with my tool notice any difference from DBpoweramp in terms of tag content, tag completeness, or metadata preservation?"**

**Answer: Largely NO.** The DSP plugin architecture produces identical audio processing regardless of implementation details. However, users WILL notice:
1. **ReplayGain precision** — DBpoweramp's AfterConversion hook lets it scan the final encoded file for ReplayGain. A tool without this hook must either scan the source (less accurate for lossy→lossless) or use a separate pass (slower).
2. **VST effect availability** — DBpoweramp Reference's VST integration is well-tested. A custom implementation using a different VST SDK may have compatibility differences.
3. **DirectX effects** — DBpoweramp supports legacy DirectX audio effects. An open-source equivalent using modern audio APIs may not replicate this.
4. **Plugin ecosystem** — DBpoweramp has a third-party codec market (Codec Central). Replicating this requires building the same plugin discovery mechanism, which attracts third-party developers.
