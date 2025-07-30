# Design Conflicts & Open Questions

This document tracks the key architectural decisions, open questions, and potential design conflicts that need to be resolved during Phase 1 of the project. Each item will be analyzed, and the final decision will be documented here.

---

### 1. PMID Granularity

*   **Question**: What should the granularity of the Pipeline Mode ID (PMID) map be?
*   **Options**:
    *   `16x16` tiles (Default proposal)
    *   `8x8` tiles (Optional)
*   **Analysis Needed**:
    *   Memory cost of each option.
    *   Flexibility vs. overhead. A finer granularity (8x8) offers more creative control but increases map size and cache pressure.
    *   Impact on the RISC-V scene controller's ability to update the map in real-time.
*   **Decision**: TBD

---

### 2. Internal Framebuffer Layout

*   **Question**: Should the internal framebuffer layout be fixed or configurable?
*   **Options**:
    *   Commit to the **hybrid model**: channel-separated internally (R, G, B, Aux in different banks) and packed 24-bit for DMA.
    *   Allow a build-time selection between packed and channel-separated.
*   **Analysis Needed**:
    *   Complexity of supporting multiple layouts in the hardware design.
    *   Performance benefits of the hybrid model for CMU and pixel operations.
    *   Is there a strong use case for a purely packed internal layout?
*   **Decision**: TBD (Leaning towards committing to the hybrid model as per the white paper recommendation).

---

### 3. Perspective Correction Path

*   **Question**: Should the perspective-correct triangle rendering path always be available?
*   **Options**:
    *   Always available in the hardware.
    *   Gated by a performance/feature bit, allowing it to be disabled to save power or area in a "lite" version of the core.
*   **Analysis Needed**:
    *   Area and power cost of the perspective-correct division hardware.
    *   Impact on clock speed.
    *   Can this be effectively clock-gated to mitigate power consumption when not in use?
*   **Decision**: TBD

---

### 4. Bilinear Filtering

*   **Question**: Where should bilinear filtering be supported?
*   **Options**:
    *   Only for sampling from the framebuffer (for post-processing effects).
    *   Also for sampling from standard textures in the triangle/quad engine.
*   **Analysis Needed**:
    *   Hardware cost. Bilinear texture filtering requires more complex texture fetch units and potentially a larger texture cache.
    *   Performance impact. It can add latency to the pixel pipeline.
    *   Is this feature essential for the "neo-retro" aesthetic, or is nearest-neighbor sufficient? (PSX was nearest-neighbor, N64 had filtering).
*   **Decision**: TBD

---

### 5. Baseline Panel Resolution

*   **Question**: What should be the primary target resolution for the reference design and developer kit?
*   **Options**:
    *   `320x240`
    *   `480x272`
*   **Analysis Needed**:
    *   Availability and cost of LCD panels at these resolutions.
    *   Impact on on-die SRAM size and cost. 320x240 double-buffered is feasible, while 480x272 might require single-buffering or PSRAM.
    *   Community preference (320x240 is a classic retro resolution).
*   **Decision**: TBD

---

### 6. Shader Stage Resource Limits

*   **Question**: What are the precise resource limits for the "Shader-Lite" programmable units?
*   **Items to Define**:
    *   **VME (Vertex Micro-Engine)**: Finalize the instruction set. Confirm the program length (≤32 ops) and state (registers, constants).
    *   **PPM (Parametric Pixel Math)**: Finalize the instruction set. Confirm the program length (≤4 ops) and number of recipes (e.g., 8-16).
*   **Analysis Needed**:
    *   Perform a "paper-based" implementation of common demo scene effects (e.g., plasma, rotozoomer, simple lighting) to see if the proposed limits are sufficient.
    *   Estimate the hardware area and timing impact for the proposed instruction sets.
*   **Decision**: TBD
