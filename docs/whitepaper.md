# RETRO: A Neo-Retro Graphics Processing System - White Paper

## Executive Summary

The Real-time Embedded Tile & Raster Orchestrator (RETRO) represents an innovative approach to graphics processing that combines classic tile/sprite-based rendering techniques with modern 3D capabilities reminiscent of the PlayStation (PSX) and Nintendo 64 (N64) era. The system is designed with a focus on deterministic performance, low power consumption, and efficient use of on-die SRAM. RETRO employs a memory-mapped architecture that simplifies control while providing flexibility through multiple rendering modes and a novel pipeline selection mechanism.

Key features include:

*   Deterministic scanline-based rendering with no mandatory full-frame read-modify-write operations
*   Memory-mapped control interface that simplifies development
*   Multiple rendering modes (tile-dominant, polygon-dominant, and mixed)
*   Efficient on-die SRAM utilization through a dynamic SRAM View Table (SVT)
*   Novel pipeline selection through a per-tile PMID (Pipeline Mode ID) system
*   RGB6666 framebuffer with 6-bit auxiliary channel for alpha/depth/special data
*   Built-in RISC-V scene controller with specialized accelerators
*   Hardware-accelerated color transformations through a Color Math Unit (CMU)
*   Optional shader-like programmable capabilities for enhanced flexibility

This paper analyzes the architecture, design choices, and potential applications of RETRO, highlighting how it bridges the gap between retro graphics systems and modern embedded graphics requirements.

---

## 1. Introduction and Background

### 1.1 Context of Retro Graphics Systems

Retro gaming consoles of the 1980s and 1990s employed specialized graphics hardware known as Picture Processing Units (PPUs) that were optimized for tile and sprite-based rendering. These systems, exemplified by the Nintendo Entertainment System (NES) and Super Nintendo Entertainment System (SNES), utilized fixed hardware pipelines to compose images scanline by scanline, with dedicated memory for tile patterns, sprite attributes, and color palettes.

The transition to 3D graphics in the mid-1990s, led by systems like the PlayStation (PSX) and Nintendo 64 (N64), introduced polygon-based rendering while still maintaining some of the scanline-based processing approaches. These systems featured dedicated hardware for transforming and rasterizing polygons, with varying approaches to texture mapping and shading.

### 1.2 The Neo-Retro Approach

RETRO represents a "neo-retro" approach that combines the best aspects of these classic systems with modern design principles:

*   **Classic Console Immediacy**: Maintains the scanline-by-scanline rendering approach that allows for deterministic timing and "raster tricks" (mid-frame modifications).
*   **Modern Embedded Pragmatism**: Incorporates memory-mapped control, efficient SRAM usage, and low-power design suitable for embedded applications.
*   **Hybrid Rendering Capabilities**: Supports both tile/sprite-based 2D and polygon-based 3D rendering with flexible mixing.

This approach is particularly well-suited for hobbyist TFT displays and embedded products that require graphics capabilities but have constrained power and memory resources.

---

## 2. Core Architecture and Design Philosophy

RETRO is built on several core principles that define its architecture.

### 2.1 Deterministic Scanline Generation

Unlike modern GPUs that typically require multiple passes over a full framebuffer, RETRO processes graphics scanline by scanline with minimal buffering. This approach reduces memory bandwidth requirements, enables precise timing control, allows for mid-frame modifications (raster effects), and simplifies synchronization with display hardware.

### 2.2 Memory-Mapped Architecture

All control and data interfaces in RETRO are memory-mapped. Hardware functionality is controlled by writing to specific memory addresses, unifying data and control to simplify the programming model.

### 2.3 Mode-Aware Flexibility

The system supports three primary rendering modes:

*   **Tile-Dominant**: Optimized for 2D graphics with multiple background layers, sprites, and affine transformations.
*   **Polygon-Dominant**: Focused on 3D rendering with PSX-style quads or perspective-correct triangles.
*   **Mixed**: Combines 2D and 3D elements with selective post-processing effects.

### 2.4 On-Die SRAM Optimization

RETRO employs a novel SRAM View Table (SVT) that dynamically maps physical SRAM banks to logical functions, allowing for efficient use of limited on-die SRAM and flexible reconfiguration between rendering modes.

### 2.5 Novel Compositing System

The system introduces a per-tile pipeline selection mechanism (PMID) that allows different rendering pipelines to be applied to different screen regions, supporting up to 3 sources per pixel with configurable combining operations and selective application of effects.

---

## 3. Memory Architecture and Organization

### 3.1 SRAM View Table (SVT)

The SRAM View Table is a key innovation in the RETRO architecture. It provides a flexible mechanism for mapping physical SRAM banks to logical functions, allowing the system to adapt to different rendering modes and workloads without requiring fixed memory partitioning. The SVT is configured by firmware at VBlank.

### 3.2 Logical Memory Regions

RETRO defines several logical memory regions that can be mapped to physical SRAM banks:

*   Tile Pattern Cache
*   Tilemap/ModeMap
*   Sprite Attributes/Pattern Rows
*   Ordering Tables/Edge & Span Buffers
*   Line-Z/Coverage
*   Combiner & HSL/Shader Parameter Tables/Palettes
*   Framebuffer
*   DMA rings & descriptors

### 3.3 Framebuffer Design

RETRO employs an RGB6666 framebuffer format (6 bits each for R, G, B, and a 6-bit Aux channel). The system supports a hybrid approach to framebuffer organization: internally, channels can be separated for efficient operations, while externally, a packed 24-bit interface is provided for DMA. Double buffering is strictly enforced.

---

## 4. Rendering Pipelines and Processing

### 4.1 Mode Hierarchy and PMID System

Rendering is organized into a hierarchy of modes. A per-tile PMID map divides the screen into 16×16 (or optionally 8×8) regions, each with a Pipeline Mode ID (PMID) that selects input sources, a combiner recipe, and feature flags.

### 4.2 Fixed-Function Pipelines

RETRO includes several fixed-function pipelines that operate on a scanline basis:

*   **Tile Engine**: Supports various tile sizes and features per-layer affine transformations and per-tile transforms.
*   **Sprite Engine**: Handles 96-128 sprites with per-sprite affine transformations and alpha blending.
*   **Tri/Quad Engine**: Processes PSX-style quads or perspective-correct triangles, using ordering tables or line-Z for depth.
*   **Compositor (Combiner)**: Combines inputs from multiple sources, applies blending recipes, and performs Z-testing.

### 4.3 Color Math Unit (CMU)

The CMU provides hardware-accelerated color transformations in HSL color space, enabling dynamic color grading and special effects without general-purpose shaders.

### 4.4 Shader-Lite Programmability

Optional "shader-lite" programmable stages extend flexibility:

*   **Vertex Micro-Engine (VME)**: Executes small per-vertex programs for 2D/2.5D transforms.
*   **Varying Interpolator (VI)**: Hardware-accelerated interpolation of up to two varying parameters.
*   **Parametric Pixel Math (PPM)**: A small per-pixel arithmetic stage for custom effects.

---

## 5. Memory Management and DMA Operations

### 5.1 DMA-Aware Compositing

DMA operations are integrated into the rendering pipeline through PMID overrides in DMA descriptors and an epoch bit system to prevent race conditions.

### 5.2 Block/Region Operations

Hardware-accelerated block operations (`FILL`, `BLIT`, `LUT`, etc.) provide efficient memory manipulation without CPU intervention.

---

## 6. RISC-V Scene Controller

An integrated RISC-V processor (RV32IMC) serves as a scene controller, managing scanline recipes, ordering tables, sprite lists, and block operations, offloading real-time graphics tasks from the host CPU.

---

## 7. Developer Model and Ergonomics

RETRO is designed for simplicity:

*   **Memory-Mapped Control**: The entire system is controlled through memory writes, with no complex register programming.
*   **Simple Mode Switching**: Modes are changed by writing a Mode Control Block (MCB) at VBlank.
*   **Per-Tile Artistry**: The PMID system enables fine-grained control over rendering and effects on a per-tile basis.
*   **Procedural Content**: Supported through RNG tile fills, LUT/HSL operations, and shader programs.

---

## 8. Comparative Analysis

### 8.1 Comparison with Classic Tile/Sprite Systems (NES/SNES)

RETRO enhances classic PPU concepts with higher color depth, larger tile sizes, flexible per-tile pipeline selection, and hardware-accelerated color math.

### 8.2 Comparison with PSX/N64 Era Systems

RETRO modernizes early 3D concepts with a unified, flexible memory architecture, simplified memory-mapped control, and more sophisticated color processing.

### 8.3 Modern Embedded Considerations

RETRO is designed for modern embedded systems with a focus on power efficiency (via scanline processing and clock gating), memory efficiency (via the SVT), and flexibility.

---

## 9. Potential Applications

*   **Retro Gaming and Emulation**: Native support for 2D and early 3D gaming paradigms.
*   **Embedded User Interfaces**: Efficient, low-power graphics for layered UIs.
*   **Educational Platforms**: A simpler, more transparent model than modern GPUs for learning graphics concepts.
*   **IoT and Smart Devices**: Low-complexity, efficient graphics for smart displays.

---

## 10. Technical Challenges and Mitigations

*   **PSRAM Latency**: Mitigated by burst access, prefetching, and caching.
*   **Framebuffer Hazards**: Mitigated by strict double buffering.
*   **Pixel Pipeline Overload**: Mitigated by limiting sources and recipe complexity.
*   **Z-Policy Conflicts**: Mitigated by enforcing a single Z-policy per scanline band.
*   **SVT Rebind Conflicts**: Mitigated by limiting rebinds to VBlank and providing a conflict analyzer.

---

## 11. Future Considerations and Open Decisions

Key design questions remain open for further study, including final PMID granularity, bilinear filtering support, and shader resource limits. These are tracked in `studies/design_conflicts.md`.

---

## 12. Conclusion

RETRO offers a compelling alternative to both fully programmable GPUs and fixed-function display controllers for embedded systems. Its deterministic nature, memory-mapped simplicity, and flexible rendering options address the specific needs of these applications while providing a powerful and fun platform for creative development.
