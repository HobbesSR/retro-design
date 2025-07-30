# RETRO: A Comprehensive Analysis

## Executive Summary

The Neo-Retro Graphics Processing System (RETRO) represents an innovative approach to graphics processing that combines classic tile/sprite-based rendering techniques with modern 3D capabilities reminiscent of the PlayStation (PSX) and Nintendo 64 (N64) era. The system is designed with a focus on deterministic performance, low power consumption, and efficient use of on-die SRAM. RETRO employs a memory-mapped architecture that simplifies control while providing flexibility through multiple rendering modes and a novel pipeline selection mechanism.

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

This report analyzes the architecture, design choices, and potential applications of the RETRO, highlighting how it bridges the gap between retro graphics systems and modern embedded graphics requirements.

## 1. Introduction and Background

### 1.1 Context of Retro Graphics Systems

Retro gaming consoles of the 1980s and 1990s employed specialized graphics hardware known as Picture Processing Units (PPUs) that were optimized for tile and sprite-based rendering. These systems, exemplified by the Nintendo Entertainment System (NES) and Super Nintendo Entertainment System (SNES), utilized fixed hardware pipelines to compose images scanline by scanline, with dedicated memory for tile patterns, sprite attributes, and color palettes.

The transition to 3D graphics in the mid-1990s, led by systems like the PlayStation (PSX) and Nintendo 64 (N64), introduced polygon-based rendering while still maintaining some of the scanline-based processing approaches. These systems featured dedicated hardware for transforming and rasterizing polygons, with varying approaches to texture mapping and shading.

### 1.2 The Neo-Retro Approach

The RETRO represents a "neo-retro" approach that combines the best aspects of these classic systems with modern design principles:

*   **Classic Console Immediacy**: Maintains the scanline-by-scanline rendering approach that allows for deterministic timing and "raster tricks" (mid-frame modifications)
*   **Modern Embedded Pragmatism**: Incorporates memory-mapped control, efficient SRAM usage, and low-power design suitable for embedded applications
*   **Hybrid Rendering Capabilities**: Supports both tile/sprite-based 2D and polygon-based 3D rendering with flexible mixing

This approach is particularly well-suited for hobbyist TFT displays and embedded products that require graphics capabilities but have constrained power and memory resources.

## 2. Core Architecture and Design Philosophy

### 2.1 Fundamental Design Principles

The RETRO is built on several core principles that define its architecture:

#### 2.1.1 Deterministic Scanline Generation

Unlike modern GPUs that typically require multiple passes over a full framebuffer, RETRO processes graphics scanline by scanline with minimal buffering. This approach:

*   Reduces memory bandwidth requirements
*   Enables precise timing control
*   Allows for mid-frame modifications (raster effects)
*   Simplifies synchronization with display hardware

#### 2.1.2 Memory-Mapped Architecture

All control and data interfaces in RETRO are memory-mapped, meaning:

*   Hardware functionality is controlled by writing to specific memory addresses
*   Data and control are unified, simplifying the programming model
*   Changes to hardware state are triggered by memory writes
*   No complex register programming sequences are required

#### 2.1.3 Mode-Aware Flexibility

The system supports three primary rendering modes:

*   **Tile-Dominant**: Optimized for 2D graphics with multiple background layers, sprites, and affine transformations
*   **Polygon-Dominant**: Focused on 3D rendering with PSX-style quads or perspective-correct triangles
*   **Mixed**: Combines 2D and 3D elements with selective post-processing effects

This flexibility allows developers to choose the most appropriate rendering approach for their specific application.

#### 2.1.4 On-Die SRAM Optimization

RETRO employs a novel SRAM View Table (SVT) that dynamically maps physical SRAM banks to logical functions, allowing:

*   Efficient use of limited on-die SRAM
*   Role-specific optimization of memory usage
*   Flexible reconfiguration between rendering modes
*   Avoidance of hard partitioning that would limit adaptability

#### 2.1.5 Novel Compositing System

The system introduces a per-tile pipeline selection mechanism (PMID) that:

*   Allows different rendering pipelines to be applied to different screen regions
*   Supports up to 3 sources per pixel with configurable combining operations
*   Enables selective application of effects like HSL color transformation
*   Integrates with optional shader-like programmable stages

### 2.2 Technical Specifications

The RETRO targets the following specifications:

*   **Output**: Parallel RGB DPI interface (RGB666 primary, RGB888 optional)
*   **Resolution**: 320×240 (baseline), 480×272 (extended), optional 800×480 via external PSRAM
*   **Framebuffer**: RGB6666 (6:6:6 + 6-bit Aux) double-buffered at 320×240 (~600 KB)
*   **Addressing**: 3-byte word addressing (24-bit pixel words) with support for packed/unpacked DMA
*   **Host Interface**: Memory-mapped bus via SPI/QSPI/AHB-like bridge
*   **External Memory**: Optional QSPI/OPI PSRAM for large assets and extended framebuffer planes
*   **Processing Core**: RISC-V RV32IMC (80–120 MHz) with light SIMD and memory-mapped accelerators
*   **Performance Target**: 30–60 FPS depending on scene complexity

## 3. Memory Architecture and Organization

### 3.1 SRAM View Table (SVT)

The SRAM View Table is a key innovation in the RETRO architecture. It provides a flexible mechanism for mapping physical SRAM banks to logical functions, allowing the system to adapt to different rendering modes and workloads without requiring fixed memory partitioning.

The SVT is configured by firmware at VBlank and defines:

*   Which physical SRAM banks are assigned to which logical regions
*   Access policies and port assignments for each region
*   Prefetch and caching behaviors

This approach enables efficient use of on-die SRAM, which is typically a limited resource in embedded systems.

### 3.2 Logical Memory Regions

The RETRO defines several logical memory regions that can be mapped to physical SRAM banks:

*   Tile Pattern Cache: Stores tile graphics data, optimized for row-aligned access and prefetching
*   Tilemap/ModeMap: Contains tile entries and per-tile PMID selections
*   Sprite Attributes/Pattern Rows: Stores sprite properties and graphics data
*   Ordering Tables/Edge & Span Buffers: Used for 3D rendering and polygon sorting
*   Line-Z/Coverage: Provides depth information for 3D rendering
*   Combiner & HSL/Shader Parameter Tables/Palettes: Stores configuration for pixel processing
*   Framebuffer: RGB6666 format for display output and selective writeback
*   DMA rings & descriptors: Controls data movement between memory regions

### 3.3 Framebuffer Design

The RETRO employs an RGB6666 framebuffer format, providing:

*   6 bits each for red, green, and blue channels (18 bits total for color)
*   An additional 6-bit auxiliary channel that can be used for:
    *   Alpha transparency
    *   Coarse depth information
    *   Special effect flags
    *   Pipeline tagging

The system supports a hybrid approach to framebuffer organization:

*   Internally, channels can be separated into aligned banks for efficient per-channel operations
*   Externally, a packed 24-bit interface is provided for DMA transfers

This approach combines the benefits of clean addressing and channel-specific operations with compact storage and compatibility with external interfaces. Double buffering is strictly enforced to prevent read/write hazards between scanout and rendering operations.

## 4. Rendering Pipelines and Processing

### 4.1 Mode Hierarchy and PMID System

The RETRO organizes rendering into a hierarchy of modes and sub-modes:

*   **High-level modes** (selected at VBlank):
    *   Tile-Dominant: Optimized for 2D graphics with multiple background layers and sprites
    *   Polygon-Dominant: Focused on 3D rendering with PSX-style quads or triangles
    *   Mixed: Combines 2D and 3D elements with selective post-processing
*   **Per-tile PMID map**:
    *   Divides the screen into 16×16 (or optionally 8×8) tiles
    *   Each tile has a Pipeline Mode ID (PMID) that selects:
        *   Input sources (BG/SPR/TRI/FB)
        *   Combiner recipe for pixel blending
        *   Feature flags (Z-test, bilinear filtering, HSL enable, etc.)
        *   Shader enable flags

This approach allows different rendering techniques to be applied to different screen regions, providing flexibility while maintaining deterministic performance.

### 4.2 Fixed-Function Pipelines

The RETRO includes several fixed-function pipelines that operate on a scanline basis:

#### 4.2.1 Tile Engine (Background Layers)

*   Supports tile sizes from 4×4 to 64×64 pixels
*   Features include per-layer affine transformations, line/column scrolling, and per-tile transforms
*   Tile entries can include ID, rotation, flipping, palette selection, and PMID overrides

#### 4.2.2 Sprite Engine

*   Handles 96-128 sprites with sizes up to 64×64 pixels
*   Supports per-sprite affine transformations, palette or direct color, and alpha blending
*   Maintains per-scanline active lists with priority and window masking

#### 4.2.3 Tri/Quad Engine

*   Processes PSX-style quads with affine mapping and Gouraud shading
*   Can use ordering tables (painter's algorithm) or line-Z for depth sorting
*   Optionally supports perspective-correct triangles and framebuffer Z-plane

#### 4.2.4 Compositor (Combiner)

*   Combines inputs from multiple sources (BG, SPR, TRI, FB samples, constants)
*   Applies programmable blending recipes (add/subtract/lerp/min/max/XOR)
*   Performs Z-testing when enabled
*   Integrates with the Color Math Unit (CMU) and optional Shader Stage

### 4.3 Color Math Unit (CMU)

The Color Math Unit provides hardware-accelerated color transformations in HSL (Hue, Saturation, Lightness) color space:

*   **Live path**: Transforms RGB to YUV, applies hue rotation, saturation scaling, and lightness adjustments
*   **Block-op path**: Performs HSL transformations on framebuffer regions, tile atlases, or palettes
*   **Modes**: YUV-fast (optimized for performance) or Exact HSL (higher quality, offline only)

This capability enables dynamic color grading, mood setting, and special effects without requiring general-purpose shaders.

### 4.4 Shader-Lite Programmability

The RETRO includes optional "shader-lite" programmable stages that extend flexibility without the complexity of a full shader GPU:

#### 4.4.1 Vertex Micro-Engine (VME)

*   Executes small per-vertex programs (≤32 ops)
*   Applies 2D/2.5D transforms and generates interpolation parameters
*   Supports basic lighting calculations and matrix operations
*   Outputs screen-space coordinates, depth, and up to two varying parameters

#### 4.4.2 Varying Interpolator (VI)

*   Hardware-accelerated interpolation of up to two varying parameters
*   Supports both affine and perspective-correct interpolation
*   Delivers interpolated values to the Combiner and CMU

#### 4.4.3 Parametric Pixel Math (PPM)

*   Small per-pixel arithmetic stage with operations like MUL, ADD, MAD, SAT
*   Operates on combiner inputs and/or interpolated varying parameters
*   Controlled by PMID recipe with a limited op budget (≤4 ops per pixel)
*   Can be placed before or after the CMU in the pipeline

These programmable elements provide flexibility for custom effects while maintaining deterministic performance characteristics.

## 5. Memory Management and DMA Operations

### 5.1 DMA-Aware Compositing

The RETRO incorporates DMA operations directly into the rendering pipeline through:

*   PMID override: DMA descriptors can carry PMID information, allowing blocks to inherit specific pipeline behaviors
*   Epoch bit system: Prevents races between DMA operations and ModeMap updates
*   Memory-mapped descriptors: Simple structs in memory control DMA operations

### 5.2 Block/Region Operations

The system provides several hardware-accelerated block operations:

*   **FILL**: Constant color/Z/tile ID/attribute filling
*   **BLIT**: Source to destination copying with optional masking
*   **LUT**: Lookup table application for color transformation
*   **VECTOR-ADD**: Vector addition for sprite attribute blocks
*   **COMPARE/MASKED-WRITE**: Conditional region operations
*   **TILE-RNG-FILL**: Procedural tilemap generation using seeded random functions
*   **HSL_BLT**: Region HSL color transformations
*   **SHADER_BLT**: Execution of shader programs on blocks of pixels or vertices

These operations run concurrently when bank conflicts don't occur, providing efficient memory manipulation without CPU intervention.

## 6. RISC-V Scene Controller

The RETRO includes an integrated RISC-V processor (RV32IMC) that serves as a scene controller:

### Core Features:

*   32-bit RISC-V with integer, multiply/divide, and compressed instruction extensions
*   Light SIMD capabilities for packed 8/16-bit operations
*   80-120 MHz clock frequency

### Responsibilities:

*   Writing Scanline Recipes for the rendering pipelines
*   Maintaining Ordering Tables, edge spans, and sprite lists
*   Updating the ModeMap for per-tile pipeline selection
*   Programming block operations through memory-mapped descriptors
*   Loading and validating shader programs and parameters

### Acceleration Support:

*   Hardware assist for span/OT building
*   Vertex FIFO for back-pressure control
*   Dedicated accelerators for common operations (blit, fill, edge, palette, HSL)

This integrated controller eliminates the need for the host CPU to handle real-time graphics tasks, reducing system complexity and improving performance.

## 7. Developer Model and Ergonomics

The RETRO is designed with developer ergonomics in mind:

### 7.1 Memory-Mapped Simplicity

The entire system is controlled through memory writes:

*   No complex register programming sequences
*   Data structures directly control hardware behavior
*   Simple descriptor-based programming model

### 7.2 Mode Control

Switching between rendering modes is accomplished by:

*   Writing a Mode Control Block (MCB) that points to SVT and recipe tables
*   Triggering a mode flip at VBlank
*   All configuration is contained in memory structures

### 7.3 Per-Tile Artistry

The PMID system enables fine-grained control over rendering:

*   Different screen regions can use different rendering techniques
*   Per-tile transformations (rotation, mirroring) are supported
*   Color grading can be applied selectively through the CMU
*   Shader effects can be toggled on a per-tile basis

### 7.4 Procedural Content Generation

The system supports procedural content through:

*   RNG-based tile filling with customizable distribution
*   LUT and HSL operations for color transformation
*   Shader programs for procedural texturing and effects

## 8. Comparative Analysis with Historical Systems

### 8.1 Comparison with Classic Tile/Sprite Systems

The RETRO shares several characteristics with classic tile/sprite-based systems like the NES and SNES:

*   **Similarities**:
    *   Tile-based background layers with attribute maps
    *   Sprite system with per-sprite attributes and priorities
    *   Scanline-based rendering with deterministic timing
    *   Palette-based color management options
*   **Enhancements**:
    *   Higher color depth (18-bit vs 4/8-bit)
    *   Larger tile sizes and more flexible transformations
    *   Per-tile pipeline selection through the PMID system
    *   Hardware-accelerated color transformations via the CMU
    *   Optional shader-like programmability

### 8.2 Comparison with PSX/N64 Era Systems

The RETRO also incorporates features from early 3D systems like the PlayStation and Nintendo 64:

*   **Similarities**:
    *   Support for textured polygons with perspective correction
    *   Gouraud shading and affine texture mapping
    *   Z-buffering or ordering table approaches for visibility
    *   Limited vertex transformation capabilities
*   **Enhancements**:
    *   Unified memory architecture with flexible SRAM mapping
    *   Memory-mapped control instead of register programming
    *   Per-tile pipeline selection for mixing 2D and 3D
    *   More sophisticated color processing through the CMU
    *   Deterministic performance characteristics

### 8.3 Modern Embedded Considerations

The RETRO addresses several concerns specific to modern embedded systems:

*   **Power Efficiency**:
    *   Scanline-based processing reduces memory bandwidth
    *   Clock gating for unused components
    *   Deterministic operation allows for precise power management
*   **Memory Efficiency**:
    *   Dynamic SRAM allocation through the SVT
    *   Compact RGB6666 framebuffer format
    *   Tile/sprite caching to reduce memory access
*   **Flexibility**:
    *   Multiple rendering modes for different applications
    *   PMID-based pipeline selection for mixed rendering
    *   Optional shader-like capabilities for custom effects

## 9. Potential Applications and Use Cases

### 9.1 Retro Gaming and Emulation

The RETRO is well-suited for retro gaming applications:

*   Native support for tile/sprite-based 2D games
*   PSX/N64-style 3D rendering capabilities
*   Low latency and deterministic timing
*   Support for classic effects like raster interrupts and mid-frame modifications

### 9.2 Embedded User Interfaces

The system provides efficient graphics for embedded UIs:

*   Memory-mapped control simplifies integration
*   Multiple rendering modes support different UI paradigms
*   Low power consumption for battery-powered devices
*   Flexible compositing for layered interfaces

### 9.3 Educational Platforms

The RETRO could serve as an educational platform:

*   Exposes fundamental graphics concepts in a tangible way
*   Simpler programming model than modern GPUs
*   Deterministic behavior aids in understanding
*   RISC-V integration provides a complete system for learning

### 9.4 IoT and Smart Devices

For IoT and smart devices, the RETRO offers:

*   Efficient graphics capabilities with low power requirements
*   Flexible rendering options for different display types
*   Integrated RISC-V controller reduces system complexity
*   Memory-mapped interface simplifies software development

## 10. Technical Challenges and Mitigations

### 10.1 PSRAM Latency

*   **Challenge**: External PSRAM could introduce latency that starves the rendering pipelines.
*   **Mitigation**:
    *   Require sequential burst access patterns
    *   Implement prefetch scheduling
    *   Use on-die caching for frequently accessed data

### 10.2 Framebuffer Hazards

*   **Challenge**: Conflicts between framebuffer writeback and scanout could cause visual artifacts.
*   **Mitigation**:
    *   Enforce strict double buffering
    *   Never target the active scanout buffer for writeback
    *   Use separate read and write ports for framebuffer access

### 10.3 Pixel Pipeline Overload

*   **Challenge**: Complex pixel processing could exceed the available cycles per pixel.
*   **Mitigation**:
    *   Cap sources per PMID to ≤3
    *   Limit per-pixel recipe complexity to 3-4 cycles
    *   Provide compile-time validation of recipe complexity
    *   Clock-gate unused stages to save power

### 10.4 Z-Policy Conflicts

*   **Challenge**: Different Z-buffer policies could conflict when mixed in the same scene.
*   **Mitigation**:
    *   Enforce band-level policy: one Z type per band
    *   Allow tiles to disable Z testing but not change Z type
    *   Provide clear documentation on Z policy constraints

### 10.5 SVT Rebind Conflicts

*   **Challenge**: Dynamic rebinding of SRAM banks could cause conflicts.
*   **Mitigation**:
    *   Limit rebinds to VBlank periods
    *   Provide build-time SVT conflict analyzer
    *   Implement hardware fault counters for SVT conflicts

## 11. Future Considerations and Open Decisions

Several aspects of the RETRO design remain open for consideration:

*   **11.1 PMID Granularity**: 16×16 default vs. 8×8 optional. Trade-off between flexibility and memory usage.
*   **11.2 Internal Layout**: Hybrid approach (channel-separated internal, 24-bit packed DMA) vs. build-time selection. Impacts addressing efficiency and channel-specific operations.
*   **11.3 Perspective Path**: Always available vs. gated by performance bit. Trade-off between feature availability and hardware complexity.
*   **11.4 Bilinear Filtering**: FB sampling only vs. also for textures. Impacts quality vs. performance balance.
*   **11.5 Panel Baseline**: 320×240 vs. 480×272 as dev-kit default. Determines baseline performance expectations.
*   **11.6 Shader Stage Design**: Instruction set and resource limits. Balance between flexibility and deterministic performance.

## 12. Conclusion

The Neo-Retro Graphics Processing System (RETRO) represents an innovative approach to graphics processing that combines the best aspects of classic tile/sprite systems and early 3D consoles with modern embedded design principles. Its focus on deterministic performance, memory-mapped simplicity, and flexible rendering capabilities makes it well-suited for a range of applications from retro gaming to embedded user interfaces.

Key innovations include the SRAM View Table for dynamic memory allocation, the PMID system for per-tile pipeline selection, the Color Math Unit for hardware-accelerated color transformations, and the optional shader-like programmable stages for enhanced flexibility. These features, combined with the integrated RISC-V scene controller, provide a complete graphics solution that balances performance, power efficiency, and developer ergonomics.

As embedded systems continue to demand more sophisticated graphics capabilities within tight power and memory constraints, the RETRO approach offers a compelling alternative to both fully programmable GPUs and fixed-function display controllers. Its deterministic nature, memory-mapped simplicity, and flexible rendering options address the specific needs of embedded applications while providing a nostalgic nod to the classic systems that pioneered real-time graphics processing.
