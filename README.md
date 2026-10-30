# Vivify Quest Port

A Quest/Android port of [Vivify](https://github.com/vivify-mod/Vivify) — a Beat Saber mod that lets custom shaders run via Unity asset bundles.

This port brings Vivify's custom shader system to Beat Saber on Meta Quest (Android), with an on-device PC→Quest bundle converter and DirectX→GLSL shader translator.

## Credits

- **Original Vivify** by [Rayla [QuestPort]([https://github.com/GRFN](https://github.com/trail-b1az3r)) / [vivify-mod](https://github.com/vivify-mod/Vivify)
- **Quest Port** by [GRFN (donoigabriel6-a11y)](https://github.com/donoigabriel6-a11y)
- **BSAudioLink** by [Aeroluna](https://github.com/Aeroluna/BSAudioLink) (MIT)
- **AudioLink** by [llealloo](https://github.com/llealloo/vrc-udon-audio-link) (MIT)
- **BeatSaber Hook**, **BSML**, **SongCore**, **Custom JSON Data**, **Tracks** — the modding ecosystem this builds on

See [AUTHORS](AUTHORS) for a full contributor list.

## Features

| Feature | Status | Notes |
|---------|--------|-------|
| **Custom Shader Support** | ✅ Full | Load Vivify bundles (`.vivify`) with custom fragment/vertex shaders |
| **On-Device PC→Quest Conversion** | ✅ Full | Converts PC bundles to Quest format on the headset; no PC required |
| **DirectX→GLSL Shader Translation** | ✅ Full | Translates HLSL/DXBC to GLSL ES 3.0 via [VivifyDxbc](src/VivifyDxbc.cpp) |
| **AudioLink** | ✅ Working | Beat-reactive audio visualization via BSAudioLink 3.1.2 port |
| **Custom Note/Saber/Debris Visuals** | ✅ Full | Full prefab replacement system with track assignment |
| **Post-Processing Effects** | ✅ Full | Screen-space effects, blits, camera depth, render textures |
| **Geometry Shader Effects** | ⚠️ Experimental | Toggle in settings; may not work on all GPUs |
| **Multiplayer Support** | ✅ Optional | Visuals can be disabled in multiplayer (default: on) |
| **Performance Options** | ✅ Full | Effect resolution scaling, far culling, offscreen culling, crash guard |

### AudioLink Details

AudioLink provides a 128×64 `_AudioTexture` that shaders can sample for:
- Frequency bands (beat-reactive)
- Waveform data
- Chronotensity (onset detection)
- Theme colors (environment color 0/1 + boost variants)

The Quest port uses a CustomRenderTexture driven by the AudioLink analysis shader (converted from PC). It captures game audio from `AudioTimeSyncController` (gameplay) or `SongPreviewPlayer` (menu).

**Known limitation**: The "42-flux" AudioLink shader variant does not work on Quest due to GPU instruction limits.

## Installation

1. Download the latest `.qmod` from [Releases](https://github.com/donoigabriel6-a11y/Vivify-Quest-Port/releases)
2. Go to [MBF Quest](https://mbf.bsquest.xyz/) and connect your Quest
3. Click "Add a Mod" and select the `.qmod` file
4. Launch Beat Saber — Vivify will appear in the Mod Settings menu

## Configuration

Open **Mod Settings → Vivify** to configure:

### Performance
- **Effect Resolution %** — Render post-process effects below eye resolution (25–100%, default 100%)
- **Offscreen Culling** — Skip animator updates for off-screen renderers (default: on)
- **Far Culling Distance** — Stop rendering beyond N metres (0 = off, default: 0)
- **Disable All Blits** — Turn off all post-processing blits (troubleshooting)
- **Disable Beat 0 Filmgrain Blit** — Skip the first-frame filmgrain blit (troubleshooting)
- **Disable CreateCamera Depth** — Disable custom camera depth textures (troubleshooting)
- **Crash Guard** — Fall back to grey stand-in after 2 consecutive load freezes (default: on)

### Shaders & Conversion
- **Convert PC Bundles On Device** — Auto-convert PC bundles when no Quest bundle exists (default: on)
- **Translate Shaders On Conversion** — Cross-compile DXBC→GLSL instead of archive-only retarget (default: on)
- **Stand-In Shading** — Use generic shader when a bundle's shader can't run (default: on)
- **Scene Depth For Map Shaders** — Provide `_SceneDepth` texture to map shaders (default: on)
- **Map Realtime Shadows** — Enable realtime shadows for converted maps (default: off)
- **Draw Geometry-Shader Effects** — Experimental: render geometry stage effects (default: off)
- **Prepare Shaders Before Playing** — Warm up shaders in menu (default: off)
- **Replay Render Mode** — Render converted maps in replays (default: off)

### Features
- **AudioLink** — Enable beat-reactive audio visualization (default: on)
  - **Gain** — Overall audio sensitivity (0.0–4.0, default 1.0)
  - **Bass** — Low-frequency boost (0.0–3.0, default 1.0)
  - **Treble** — High-frequency boost (0.0–3.0, default 1.0)
- **Submit Scores On Vivify Maps** — Allow leaderboard submission (default: on)
- **Disable Custom Note Visuals** — Use game's default notes (default: off)
- **Disable Vivify Visuals In Multiplayer** — Disable custom visuals in multiplayer (default: on)
- **Disable VR Center Adjust Handling** — Disable VR center adjustment fixes (default: off)

### Conversion Tools
- **Convert All PC Bundles Now** — Batch convert installed maps
- **Force Reconvert All** — Ignore cache and reconvert everything
- **Reset Vivify** — Delete all converted bundles, downloaded PC bundles, and reset settings (double-press within 5s)
- **Redownload All Vivify Maps** — Fresh BeatSaver downloads for all Vivify maps (double-press within 5s)

## For Map Makers

### Bundle Requirements

Vivify on Quest loads bundles named:
- `bundleAndroid2021.vivify` (preferred)
- `bundleAndroid.vivify` (legacy)

PC bundles (`bundle.vivify`) are auto-converted on device if **Convert PC Bundles On Device** is enabled.

### Shader Translation

The converter translates a subset of HLSL/DXBC to GLSL ES 3.0:
- ✅ Basic arithmetic, swizzles, texture sampling
- ✅ `tex2D`, `tex2Dlod`, `texCUBE`, `texCUBElod`
- ✅ `discard`, `clip`, `ddx`/`ddy`/`fwidth`
- ✅ Uniform buffers, constant buffers
- ⚠️ Geometry shaders — experimental, limited support
- ❌ Compute shaders, tessellation, ray tracing
- ❌ Advanced DX11+ features (bindless, mesh shaders, etc.)

Shaders using unsupported features fall back to **Stand-In Shading** (if enabled) or render as grey.

### AudioLink in Shaders

To use AudioLink data, declare in your shader:

```glsl
uniform sampler2D _AudioTexture;
// Sample UV: x = time (0–1 across 128px), y = feature row (0–63)
```

Rows in `_AudioTexture` (y coordinate / 64):
| Row Range | Data |
|-----------|------|
| 0–7 | Waveform (L/R interleaved) |
| 8–23 | Frequency bands (16 bands × L/R) |
| 24–31 | Chronotensity (onset detection) |
| 32–39 | Raw spectrum |
| 40–47 | Integrated spectrum |
| 48–55 | Theme color 0 / 1 / 0boost / 1boost |
| 56–63 | Advanced time / player data / version |

See [AudioLink 3.1.2 shader](https://github.com/Aeroluna/BSAudioLink) for full sampling logic.

## Building from Source

This project builds via **GitHub Actions only**. Local builds are not supported.

### Prerequisites (for reference)
- Android NDK r25+
- CMake 3.22+
- Python 3.8+ (for shader converter)
- QuestBeatSaber modding toolchain (beatsaber-hook, bsml, songcore, etc.)

### Build Process (GitHub Actions)

The workflow:
1. Sets up Android NDK and dependencies
2. Runs `tools/audiolink/build_bundle.py` to convert AudioLink PC bundle → Quest bundle
3. Compiles `libVivify.so` (arm64-v8a) via CMake
4. Packages `.qmod` with manifest, bundles, and resources

To trigger a build: push to `main` or `GRFN's-Vivify` branch, or run the workflow manually.

## Troubleshooting

### Logs

Session log: `/sdcard/ModData/com.beatgames.beatsaber/Mods/Vivify/VivifySession.txt`  
Global log (all sessions): `/sdcard/ModData/com.beatgames.beatsaber/Mods/Vivify/vivify_global.txt`

Enable **Debug Logging** in settings for verbose output.

### Common Issues

| Symptom | Likely Cause | Fix |
|---------|--------------|-----|
| Map plays but custom visuals missing | Shader unsupported on GPU | Enable **Stand-In Shading**; check log for "device rejected" |
| AudioLink not reacting | Audio source not found | Check log for "AudioLink: running"; ensure AudioLink enabled |
| Converted map grey/white | Shader translation failed | Disable **Translate Shaders On Conversion** for archive-only fallback |
| Game freezes on map load | Shader compilation stall | Enable **Crash Guard**; check log for "frame stall" |
| Notes invisible in multiplayer | **Disable Vivify Visuals In Multiplayer** on | Toggle off if you want custom visuals in multiplayer |

### Reporting Bugs

1. Check [Issues](https://github.com/donoigabriel6-a11y/Vivify-Quest-Port/issues) first
2. Include `VivifySession.txt` and `vivify_global.txt` (redact personal info)
3. Specify: Quest firmware version, Beat Saber version, Vivify version
4. List other installed mods

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      Vivify Runtime                         │
├──────────────┬──────────────┬──────────────┬────────────────┤
│  Assets      │  Components  │  AudioLink   │  Core Systems  │
│  (bundles,   │  (behaviours,│  (GPU audio  │  (events,      │
│   shaders,   │   cameras,   │   analysis,  │   prefabs,     │
│   textures)  │   trails)    │   texture)   │   animations)  │
└──────┬───────┴──────┬───────┴──────┬───────┴───────┬────────┘
       │              │              │               │
       ▼              ▼              ▼               ▼
┌─────────────────────────────────────────────────────────────┐
│              Unity / IL2CPP / Beat Saber                    │
└─────────────────────────────────────────────────────────────┘
```

Key modules:
- **VivifyRuntime** — Main lifecycle, bundle loading, event handling
- **VivifyAudioLink** — GPU audio analysis, `_AudioTexture` publishing
- **VivifyComponents** — MonoBehaviours for cameras, trails, culling
- **VivifyDxbc** — DXBC→GLSL shader translation
- **VivifyBundleConvert** — PC→Quest archive retargeting

## License

This project is licensed under the same terms as the upstream Vivify project. See individual file headers for details.

- BSAudioLink / AudioLink shader: MIT
- beatsaber-hook, BSML, SongCore, etc.: their respective licenses

---

*This is an unofficial port. Not affiliated with Beat Games, Meta, or the official Vivify maintainers.*
