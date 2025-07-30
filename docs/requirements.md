# RETRO - Technical Requirements

This document outlines the core technical requirements for the RETRO graphics processor. It is a distillation of the specifications found in the main [White Paper](./whitepaper.md).

---

## 1. Core System

*   **Host Interface**: Memory-mapped bus accessible via SPI/QSPI or an AHB-like bridge.
*   **Scene Controller**: RISC-V RV32IMC core (target 80–120 MHz) with light SIMD extensions.
*   **Memory Model**: All hardware features are controlled via memory-mapped logical regions. Control is handled through configuration blocks, not scattered registers.

---

## 2. Video Output

*   **Interface**: Parallel RGB DPI.
*   **Color Depth**: RGB666 (primary), RGB888 (optional).
*   **Resolutions & Refresh Rate**:
    *   320×240 @ 30-60 Hz (baseline, on-die FB)
    *   480×272 @ 30-60 Hz (extended, on-die FB)
    *   800×480 (requires external PSRAM)
*   **Timing**: Deterministic scanline generation with a small line buffer.

---

## 3. Framebuffer

*   **Format**: **RGB6666** (6 bits each for Red, Green, Blue + 6-bit Auxiliary channel).
*   **Aux Channel Use**: Mode-dependent (Alpha, Coarse Depth, Stencil, etc.).
*   **On-Die Buffering**: Double-buffered at 320×240 resolution (~600 KB total).
*   **Addressing**:
    *   Internal: Channel-separated (R, G, B, Aux in separate banks) for aligned math.
    *   External DMA: Exposed as packed 24-bit (3-byte) words.
*   **Write Safety**: Strict double buffering is enforced. Writeback never targets the active scanout buffer.

---

## 4. Memory System

*   **SRAM View Table (SVT)**: A memory-mapped table to bind physical SRAM banks to logical roles (e.g., tile cache, Z-buffer, ordering tables) at VBlank.
*   **External RAM**: Optional support for QSPI/OPI PSRAM for large assets or framebuffers.
*   **Porting**:
    *   **Dual-port**: For high-contention regions (Tile Cache, Sprite Attributes, Framebuffer).
    *   **Single-port**: For phase-separated regions (Ordering Tables, Line-Z, LUTs).

---

## 5. Rendering Pipelines

### 5.1 General

*   **Compositing**: Per-tile pipeline selection via a **PMID (Pipeline Mode ID) map**.
*   **Pixel Budget**: Max 3-4 cycles per pixel.
*   **Source Limit**: Max 3 sources per PMID recipe (e.g., BG + Sprite + FB Sample).

### 5.2 Tile Engine

*   **Sizes**: 4x4, 8x8, 16x16, 32x32, 64x64 pixels.
*   **Transforms**: H/V Flip, 90/180/270° rotation (via address swizzles).
*   **Features**: Per-layer affine (Mode-7 like), line/column scroll, mosaic.

### 5.3 Sprite Engine

*   **Capacity**: 96-128 sprites total, capped per scanline (e.g., 16-32 active).
*   **Size**: Up to 64x64 pixels.
*   **Features**: Per-sprite affine transforms, palette or direct color.

### 5.4 Triangle/Quad Engine

*   **Modes** (mutually exclusive per PMID):
    *   **PSX-style**: Affine textured quads with ordering tables (painter's algorithm).
    *   **Perspective**: Perspective-correct triangles with a line-based Z-buffer.
*   **Texturing**: Nearest neighbor sampling. Optional bilinear for framebuffer sampling.

---

## 6. Shader-Lite & Color Math Units

### 6.1 Color Math Unit (CMU)

*   **Function**: Hardware-accelerated HSL/YUV color adjustments.
*   **Path**: RGB→YUV → Hue Rotate, Saturation, Lightness → RGB → Dither.
*   **Control**: Parameters selected per-pixel via PMID from an **HSL Param Table**.

### 6.2 Vertex Micro-Engine (VME)

*   **Purpose**: Per-vertex transforms for 2D/2.5D effects.
*   **Program Constraints**: ≤ 32 operations, no loops.
*   **Instruction Set**: Fixed-point math (`ADD`, `MUL`, `MAD`, `DOT2/3`, `MUL_MAT2x3`).
*   **Output**: Screen-space coordinates {x,y,z,w} and up to two varyings (`v0`, `v1`).

### 6.3 Varying Interpolator (VI)

*   **Function**: Interpolates up to two varyings from the VME.
*   **Modes**: Affine or perspective-correct.

### 6.4 Parametric Pixel Math (PPM)

*   **Purpose**: Complements the CMU with generic per-pixel arithmetic.
*   **Program Constraints**: ≤ 3-5 operations per recipe.
*   **Instruction Set**: `MUL`, `ADD`, `MAD`, `SAT`, `THRESH`, `1D LUT`.

---

## 7. Block Operation Engines (DMA)

*   **Operations**: `FILL`, `BLIT`, `LUT_APPLY`, `VECTOR_ADD`, `COMPARE_MASKED_WRITE`, `HSL_BLT`, `SHADER_BLT`.
*   **Procedural Fill**: `TILE_RNG_FILL` for procedural generation of tilemaps.
*   **DMA Override**: DMA descriptors can carry a PMID to override the default pipeline for a specific block.
