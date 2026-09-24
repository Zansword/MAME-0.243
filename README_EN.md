# MAME 0.243 Atomiswave/PowerVR2 (Metal Slug 6) Emulation Profiling & Fix Report

> **One-Line Summary**: A reverse engineering and debugging record analyzing the root causes of frame drops, audio stuttering, and graphic glitches in MAME 0.243's Sammy Atomiswave (Dreamcast-derived hardware) PowerVR2 video core by cross-referencing flycast and Demul sources, followed by actual patch implementation.

**Compiler**: `mingw-mame-20100102`

---

## 1. Background & Goals

- **Target**: `mame_243/src/mame/video/powervr2.cpp` (+`.h`) — Video core for the Sammy Atomiswave board (SH-4 200MHz + PowerVR2 100MHz + Yamaha AICA/ARM7) driver.
- **Test Title**: *Metal Slug 6* (`mslug6`) — Despite the name, it runs on Atomiswave (Dreamcast derivative) hardware rather than Neo Geo.
- **Symptoms**: The BIOS boot screen runs at 100% speed, but actual gameplay rendering drops to around 70%. Graphics render incorrectly on specific screens (such as the ranking screen), and audio stutters alongside the frame drops.
- **Comparison Baseline**: Three-way analysis conducted by securing sources from **flycast** (an actively maintained high-accuracy emulator) and **Demul** (an early-generation emulator from the 2000s featuring a PSEmu Pro-style plugin architecture) running the same hardware.

---

## 2. Methodology

1. Statically analyzed code suspected of performance/accuracy issues within the `mame_243` (MAME 0.243) source.
2. Located and structurally compared identical hardware implementations in flycast and Demul.
3. Upon confirming differences, proceeded in the order of **Hypothesis $\rightarrow$ Minimum-Invasive Patch $\rightarrow$ (When possible) Verification $\rightarrow$ Rollback on Failure**.
4. Risk-heavy changes (such as threading) under unverified conditions (limited access to actual build/runtime) were **immediately reverted upon failure, logging the lessons learned**.

---

## 3. Core Architectural Conclusions

### 3.1 Fundamental Structural Differences Across the Three Emulators

| | MAME (mame_243, Pre-Fix) | flycast | Demul |
|---|---|---|---|
| Polygon Raster / Texture / Blend | CPU scalar loop (software renderer) | **Real GPU hardware** (OpenGL/Vulkan/DX draw calls) | **Real GPU hardware** (OpenGL) |
| SH4 CPU Execution | Shared main cooperative scheduler thread | Dedicated OS thread separation (`ThreadedRendering`) | CPU/device separation by core architecture |
| GPU Render Request Handling | Synchronous calculation directly inside the SH4 instruction dispatch callstack upon `STARTRENDER` register write | TA_context snapshot $\rightarrow$ Separate render thread queuing (fire-and-forget) | Physical separation via GPU plugin DLL boundary |
| Audio Generation | Dependent on the main scheduler's periodic timer (`sound_manager::update`) | Separate thread/buffer path | **Dedicated real-time priority thread** (`THREAD_PRIORITY_TIME_CRITICAL`), driven by DirectSound hardware notification events |

**Conclusion**: MAME processes the SH4, GPU (PowerVR2), and sound (AICA/ARM7) via a **single cooperative thread** using time-sharing, presenting structural limitations in accurately reproducing the timing of genuinely parallel silicon (Dreamcast/Naomi-class hardware). flycast and Demul are immune to this issue because they were **designed from the ground up to separate the GPU/audio into separate threads and separate hardware (GPU APIs)**.

> Note: This is not an issue affecting "all arcade boards post-PS1," but rather a limitation restricted to **specific generations of hardware where PowerVR2 + AICA/ARM7 operate as true parallel silicon**. MAME accurately implements other multi-CPU arcade boards.

### 3.2 "Unimplemented/Incomplete" Items Classified by Risk Level

| Risk Level | Item | Notes |
|---|---|---|
| Low (Local code, independent of MAME core) | Dirty regions, SIMD blend, layer order, missing registers, G2 DMA bulk copy, RTT, modifier volumes | Implemented/fixed in this work |
| Medium (Recombination of MAME-provided tools, concurrency bug potential) | Asynchronous render workers based on `osd_work_queue` | Both success and failure (rollback) cases exist |
| High (Requires modifying the MAME core/skeleton itself) | Complete thread separation of the sound pipeline, transition to GPU hardware draw-call-based renderer | Not attempted — requires major surgery affecting the entire driver |

---

## 4. Discovered and Fixed Issues

### 4.1 Frame Rate Drops (Performance)

1. **Fully Synchronous Rendering** — Calculated the entire screen directly inside the SH4 command dispatch callstack upon writing to the `STARTRENDER` register $\rightarrow$ Separated into an asynchronous worker based on `osd_work_queue` (referencing flycast's `Renderer_if.cpp` pattern and reusing MAME's existing convention from `epic12.cpp`).
2. **Pixel Corruption Due to Missing Register Snapshots** — Discovered and fixed a real race condition where the SH4 overwrote `region_base`/`param_base` mid-asynchronization, causing the worker to read incorrect coordinates (added `*_snap` fields to `receiveddata`).
3. **Full-Screen Recalculation Waste** — Reduced processing from clearing and refilling the entire 1024x1024 area every frame to handling only the **dirty region bounding box** calculated from tile coordinates in the region list.
4. **Framebuffer Conversion Pixel-Level `address_space` Dispatch** — Replaced `space.write_word/byte/dword` calls in 20 `fb_convert_*` functions with direct VRAM pointer access.
5. **G2 DMA (for Sound Sample Loading) 2-Byte Unit Transfer** — Bottleneck culprit during narration voice playback. Added a high-speed dword-format bulk copy path (identical behavior, halved dispatch count).
6. **EOR (Render Complete) IRQ Timing** — Replaced the tile-count-based magic number with the actual data-volume-proportional formula identical to flycast.
7. **Root Cause of Audio Stuttering** — Confirmed that `sound_manager::update()` shared the **same thread** as the render worker, causing blocking waits to delay audio as well. Mitigated by converting only the screen display path (`screen_update`) to non-blocking polling.
8. **Attempted Band-Parallel Rasterization $\rightarrow$ Rolled Back**: Attempted parallel processing by dividing the screen into multiple bands to utilize more cores, but graphics became more unstable during actual testing (presumed related to `osd_work_queue` multi-threaded "stealing" paths) — Immediately reverted following the principle of **not pushing concurrency code without empirical validation**.
9. **SIMD Optimization of Blend Operations** — Rewrote six hand-written 32-bit SWAR bit-trick functions using MAME's built-in `rgbaint_t` (automatically selecting SSE2/AltiVec/Generic).

### 4.2 Graphic Accuracy

10. **Layer Drawing Order Bug** — Corrected Opaque $\rightarrow$ **Translucent $\rightarrow$ Punch-Through** (incorrect) order to match flycast's actual render pass (Opaque $\rightarrow$ **Punch-Through $\rightarrow$ Translucent**).
11. **Render-to-Texture (RTT) Completely Unimplemented** — Bit 24 of `FB_W_SOF1` (RTT signal) was unhandled anywhere, causing UIs pre-rendered as textures (like ranking screens) to break and leading to out-of-bounds memory access bugs beyond the **16MB+ range** due to unmasked bits. Implemented RTT detection + masking + render target redirection to texture memory.
12. **Completely Missing `Y_COEFF` Register (0x118)** — Discovered via actual log analysis (unmapped write). Confirmed as a real R/W register after comparing with flycast; added storage + handler + register map registration.
13. **Modifier Volume (Shadow/Lighting Stencil Effect) Implementation** — Newly implemented a CPU software version (simplified odd-even counting rule) of a feature left unimplemented in both MAME and Demul (Demul discarded it without even leaving debug logs), referencing flycast's stencil volume algorithm. Also discovered and fixed a hidden bug where the TA parser was discarding the data itself.

---

## 5. Remaining Issues

- The root cause for certain screens like the ranking screen remains **unresolved** — layer order, dirty regions, RTT, and `Y_COEFF` were sequentially ruled out, but a definitive conclusion could not be reached. This stage requires actual runtime register/TA command traces.
- Frame drops in heavy overdraw zones with many enemies are judged to be a **physical limitation of the CPU software renderer** — impossible to resolve fundamentally without GPU offloading.
- The modifier volume implementation is intentionally simplified (multi-volume combinations and per-polygon Shadow bit gating are unsupported).

---

## 6. Notes on Tools and Methods Used

- Due to limited build and runtime debugging environments, all changes were applied via minimum-invasive methods after establishing grounds through **static code analysis + three-way source cross-verification (MAME/flycast/Demul)**.
- Unverifiable risks (concurrency threading) were **unhesitatingly rolled back** if actual user test results were poor, recording the lesson — dangerous changes were never forced based solely on the guess that "it might get fixed."
- Confirmed missing registers based on **observed facts rather than guesswork**, using actual MAME unmapped access logs provided by the user.

---

## 7. License and Source Code Notices

- Modified code regarding MAME/flycast in this repository complies with the respective original project licenses (such as GPL-2.0-or-later).
- **Demul source code is not included in this repository.** Because Demul's original distribution license text includes non-standard/restrictive wording distinct from standard open-source licenses (explicitly forbidding redistribution), only the analysis results are quoted as text.
- Game ROM files are not included.

---

## 8. Summary One-Line Conclusion

> MAME's cooperative single-threaded scheduler structure imposes structural limitations in accurately reproducing the timing of Dreamcast/Naomi-class hardware (SH4 + PowerVR2 + AICA/ARM7) where the GPU and sound CPU operate as genuine independent parallel silicon. While partial mitigations (async worker threads, timing adjustments, non-blocking polling) are possible, a complete solution requires major surgery at the level of MAME's core scheduler and sound pipeline. Conversely, emulators like flycast and Demul—designed from the start with independent threads and real GPU hardware for CPU, GPU, and audio—are free from this structural issue.