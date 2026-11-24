# OptiLuman

**A rendering engine for Minecraft built on Vulkan: hardware ray tracing, modern upscalers (when supported by hardware), and a custom shader pack format.**

> ⚠️ **Alpha 0.1.0.** Early build — expect bugs and keep backups of your worlds.

---

## About

OptiLuman is a custom rendering engine for Minecraft built on the **Vulkan API**. It replaces the default rendering pipeline, supports hardware ray tracing on compatible GPUs, and lets authors build their own shader packs.

📖 **Shader Pack Format Specification:** [docs/shaderpack-format.md](docs/shaderpack-format.md)

## Features

- **Hardware Ray Tracing** on supported GPUs (NVIDIA RTX, AMD RX 6000+, Intel Arc).
- **Standard Rasterization** fallback on systems without hardware ray tracing.
- **Upscalers** when supported by the user's hardware: DLSS (including Ray Reconstruction), FSR, XeSS, and TAA.
- **Partial Physics**: rigid-body physics for dropped items (items fall, roll, and settle) and solid collisions for mobs and players.
- **Custom Shader Packs**: a dedicated pack format (`optiluman.pack.json`) supporting custom passes, render targets, options, option screens, presets, and translations, with parallel shader compilation and disk caching.

## Requirements

| | |
|---|---|
| **Minecraft** | **26.2** (26.3 planned) |
| **Loader** | Fabric Loader 0.20.1+ |
| **Dependencies** | Fabric API |
| **Java** | 25 |
| **GPU** | Vulkan; hardware ray tracing requires an RT-capable GPU (NVIDIA RTX, AMD RX 6000+, Intel Arc). |

**Not compatible with Sodium or Iris**: OptiLuman directly replaces the renderer they modify.

## Installation

1. Install Fabric Loader and Fabric API for Minecraft 26.2.
2. Drop the OptiLuman `.jar` into your `mods` folder.
3. Place an OptiLuman shader pack into `.minecraft/shaderpacks/`.
4. Press **I** in-game to open the OptiLuman menu and select a pack. Upscalers can be configured in **Video Settings**.

The UI and example pack are translated into English, Russian, Kazakh, Japanese, Chinese, and Korean.

## Known Issues (Alpha)

- The game may crash when closing.
- Without ray tracing, held light sources may shine through walls.
- Parallax is not applied to mobs or inside reflections.

## Community & Feedback

- **Discord:** *(insert your Discord invite link here)*
- **Issues:** [GitHub Issues](../../issues)
- **Downloads:** *(insert your Modrinth / Releases link here)*

Bug reports and screenshots are welcome.
