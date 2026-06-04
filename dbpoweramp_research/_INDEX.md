# DBpoweramp Research — Index

## About This Research

This directory contains exhaustive behavioral research on **DBpoweramp Music Converter** (by Illustrate Ltd., spoon.net), reverse-engineered from official documentation, Spoon developer forum posts, community observations, and comparison with open-source equivalents.

> **Research is organized into categories by topic.** Each file is self-contained and cross-references other files where relevant.

---

## Architecture Research (Files 00–09)

| File | Topic | Lines |
|---|---|---|
| `00_Architecture_Overview_and_Component_Map.md` | Full component diagram, process/thread architecture, plugin discovery, PerfectTUNES, CD Ripper, STdBEncoderFluid | 713 |
| `01_Conversion_Engine_Core_Pipeline.md` | Complete stage-by-stage pipeline, conversion coordinator, memory management, format negotiation, config system, license checking | 842 |
| `02_Codec_Plugin_Architecture_DSP_Framework.md` | Encoder/decoder/DSP plugin SDK, live vs non-live DSP, ShowConfigBit embedding, CLI encoder wrapper, all 40+ DSP effects | 915 |
| `03_Internal_Audio_Representation_PCM_Buffer.md` | 32-bit/64-bit float internal format, TPDF dither, ARDFTSRC resampler, channel layout, gapless playback, buffer management | 1021 |

## Metadata Core Systems (Files 10–19)

| File | Topic | Lines |
|---|---|---|
| `10_Internal_Tag_Model_Canonical_Representation.md` | CanonicalTag structure, multi-value fields, field normalization rules | 656 |
| `11_Tag_Reading_Pipeline_All_Formats.md` | Multi-tag priority matrix, merge vs first-wins, graceful degradation | 601 |
| `12_Tag_Writing_Pipeline_All_Formats.md` | Per-format tag systems, atomic writes, tag cleanup | 438 |
| `13_Tag_Normalization_and_Field_Standardization.md` | Field normalization rules across formats | 496 |
| `14_Tag_Mapping_Cross_Format_Master_Table.md` | Complete cross-format tag field mapping (master reference) | 1032 |
| `15_Metadata_Preservation_Rules_and_Priority.md` | Preservation tiers, encoder tag handling | 350 |
| `16_Metadata_Conflict_Resolution_Logic.md` | Conflict resolution across tag systems | 351 |
| `17_Custom_and_Unknown_Tag_Field_Handling.md` | Custom/unknown field preservation | 412 |
| `18_Character_Encoding_Detection_and_Conversion.md` | UTF-8/UTF-16/ISO-8859-1 handling, NFC normalization | 498 |

## Cover Art Pipeline (Files 20–29)

| File | Topic | Lines |
|---|---|---|
| `20_Cover_Art_Read_Pipeline.md` | Cover art read priority cascade | 239 |
| `21_Cover_Art_Write_Pipeline.md` | Embedding behavior, type tags | 333 |
| `22_Cover_Art_Folder_Image_Fallback_Logic.md` | folder.jpg and folder image fallback | 335 |
| `23_Cover_Art_Resize_and_Format_Conversion.md` | Resize algorithm, format conversion | 403 |
| `24_Multi_Image_Type_Handling_Front_Back_Artist.md` | Multi-image type handling | 482 |

## Format-Specific Tag Behavior (Files 30–39)

| File | Topic | Lines |
|---|---|---|
| `30_MP3_ID3_Tag_Behavior_Deep_Dive.md` | ID3v2.3/ID3v2.4, APEv2 in MP3, unsynchronization | 207 |
| `31_FLAC_Vorbis_Comment_Tag_Behavior.md` | METADATA_BLOCK_PICTURE, vendor string | 208 |
| `32_AAC_MP4_iTunes_Atom_Tag_Behavior.md` | iTunes atoms, work/movement, gapless tags | 231 |
| `33_OGG_Vorbis_Tag_Behavior.md` | Vorbis Comment quirks, R128 tags | 206 |
| `34_Opus_Tag_Behavior.md` | Opus R128 tags, CELT/Silk mapping | 196 |
| `35_WAV_AIFF_RIFF_Tag_Behavior.md` | RIFF INFO, BWF, ID3v2-in-WAV, AIFF chunks | 216 |
| `36_WMA_ASF_Tag_Behavior.md` | ASF/WMA attributes, DRM | 214 |
| `37_APE_Monkeys_Audio_Tag_Behavior.md` | APEv2 in APE/MAC files | 216 |
| `38_WavPack_Tag_Behavior.md` | WavPack hybrid, APEv2 tags | 194 |
| `39_ALAC_Apple_Lossless_Tag_Behavior.md` | ALAC in M4A container | 206 |

## Field-Level Mapping Deep Dives (Files 40–49)

| File | Topic | Lines |
|---|---|---|
| `40_Field_Map_Track_and_Disc_Numbering.md` | Complete track/disc number cross-format mapping | 484 |
| `41_Field_Map_Artist_Album_Artist_Composer.md` | Multi-artist, sort tags, classical handling | 589 |
| `42_Field_Map_Genre_Year_Date_Formats.md` | Genre number resolution, date format normalization | 629 |
| `43_Field_Map_ReplayGain_Across_Formats.md` | ReplayGain/R128 field names, value formats | 531 |
| `44_Field_Map_MusicBrainz_Tags_Across_Formats.md` | MBIDs preservation across formats | 394 |
| `45_Field_Map_Lyrics_Comment_Description.md` | Lyrics, comment, description field handling | 470 |
| `46_Field_Map_Compilation_and_Podcast_Flags.md` | TCMP/cpil/podcast flags | 364 |
| `47_Field_Map_BPM_Key_Rating_Tags.md` | TBPM/TKEY/POPM across formats | 421 |
| `48_Field_Map_ISRC_Copyright_Publisher.md` | ISRC, copyright, publisher fields | 379 |
| `49_Field_Map_Encoder_EncoderSettings_Tags.md` | TSSE/ENCODER/ENCODERSETTINGS across formats | 421 |

## Conversion & Processing Pipelines (Files 50–59)

| File | Topic | Lines |
|---|---|---|
| `50_Single_File_Conversion_Pipeline_Sequence.md` | Complete single-file pipeline (Phase 0–9) | 500 |
| `51_Batch_Conversion_Architecture_and_Threading.md` | Queue management, CPU core allocation | 364 |
| `52_Lossless_to_Lossless_Pipeline.md` | Bit-exact verification, FLAC/ALAC re-encode | 288 |
| `53_Lossless_to_Lossy_Pipeline.md` | Transcoding, quality selection | 284 |
| `54_Lossy_to_Lossless_Pipeline_Transparency.md` | Generation loss detection, lossy→lossy | 284 |
| `55_Lossy_to_Lossy_Pipeline_Generation_Loss.md` | Same-codec transcode, quality degradation | 278 |

## Advanced Features (Files 60–69)

| File | Topic | Lines |
|---|---|---|
| `60_CD_Ripping_Pipeline_and_Architecture.md` | CD ripper architecture, drive control | 238 |
| `61_AccurateRip_Verification_Pipeline.md` | AccurateRip v2, confidence scoring | 312 |
| `62_MusicBrainz_CDDB_Metadata_Lookup_Pipeline.md` | Metadata lookup, PerfectMeta | 305 |
| `63_ReplayGain_Scanning_Pipeline_EBU_R128.md` | R128 algorithm, album vs track mode | 373 |
| `64_CueSheet_Splitting_and_Merging_Pipeline.md` | CUE parsing, frame-accurate splitting | 348 |
| `65_DSP_Effects_Chain_Order_and_Behavior.md` | Complete DSP chain ordering guide | 355 |
| `66_File_Naming_Template_Engine.md` | Template syntax, conditional expressions | 361 |
| `67_Folder_Organization_and_Output_Path_Logic.md` | Output path resolution, folder creation | 323 |
| `68_Error_Handling_Recovery_and_Logging.md` | Error codes, recovery, logging | 344 |
| `69_Audio_Quality_Verification_Post_Encode.md` | Post-encode verification, CRC | 381 |

## Implementation Reference (Files 70–79)

| File | Topic | Lines |
|---|---|---|
| `70_Rebuilding_Tag_Preservation_Implementation_Guide.md` | **Actionable implementation guide** — complete Python/FFmpeg code for full tag preservation | 3888 |
| `71_FFmpeg_Tag_Mapping_Implementation.md` | FFmpeg CLI equivalents for all DBpoweramp tag behaviors | 1154 |

---

## Key Research Findings Summary

### Architecture (Files 00–03)
- **Multi-process**: Each CoreConverter.exe uses exactly 1 CPU core; parallelism via N simultaneous processes
- **STdBEncoderFluid**: Shared state structure carries format, tags, filenames through all pipeline stages
- **Live vs non-live DSP**: Live = per-block in encode loop; non-live = full decode to temp file
- **AfterConversion hook**: Unique feature — called with FINAL filename (not .tmp) for post-encode tag writing
- **32-bit float internal PCM**: Primary format since R14; 64-bit float added R2025-11
- **ARDFTSRC**: New resampler (R2025-11) supports arbitrary sample rates including non-standard values

### Metadata (Files 10–18)
- **Track numbering**: Parse "N/M" → separate fields; recombine per target format
- **ReplayGain**: DBpoweramp writes R128 for Opus, ReplayGain for others; reads all variants
- **Cover art**: Always type 3 (front cover); METADATA_BLOCK_PICTURE for FLAC
- **ID3v2.3 default**: Most compatible; v2.4 optional

### Conversion (Files 50–55)
- **Atomic writes**: Always write to .tmp → MoveFileEx() to final on success
- **Gapless**: Transparent pass-through of encoder delay/padding
- **Lossless→lossless**: Re-encode preserves decoded PCM; not bit-exact at bitstream level

---

## Research Methodology

- **Official sources**: spoon.net documentation, dBpoweramp Developer SDK pages, Spoon's official forum posts
- **Community reverse-engineering**: Hydrogenaudio forums, dBpoweramp forums, Codec Central
- **Behavioral comparison**: fre:ac (BoCA), MusicBrainz Picard, beets, Mp3tag, TagLib
- **Spec inference**: Xiph.org FLAC/Vorbis specs, ID3.org, MP4 spec, LAME documentation
- **Version analysis**: 10+ years of release notes (R14 through R2026-04)

---

*Research directory: `/home/alpha01/gitrepo/utils-audio-tools/dbpoweramp_research/`*
*Total files: 71 research documents | ~27,000 lines*
