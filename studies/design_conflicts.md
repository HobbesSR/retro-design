# Design Conflicts & Open Questions

This document tracks the key architectural decisions, open questions, and potential design conflicts that need to be resolved during Phase 1 of the project. Each item will be analyzed, and the final decision will be documented here.

---

### 1. PMID Granularity

*   **Question**: What should the granularity of the Pipeline Mode ID (PMID) map be?
*   **Options**:
    *   `16x16` tiles (Default proposal)
    *   `8x8` tiles (Optional)
*   **Analysis**:
    *   **Memory Cost**: The PMID map is a 2D array covering the screen. Assuming 1 byte per PMID entry:
        *   **For a 320x240 screen:**
            *   `16x16` granularity: `(320/16) x (240/16)` = `20 x 15` = **300 bytes**.
            *   `8x8` granularity: `(320/8) x (240/8)` = `40 x 30` = **1200 bytes**.
        *   **For a 480x272 screen:**
            *   `16x16` granularity: `(480/16) x (272/16)` = `30 x 17` = **510 bytes**.
            *   `8x8` granularity: `(480/8) x (272/8)` = `60 x 34` = **2040 bytes**.
        *   The `8x8` option requires **4x** the memory of the `16x16` option, making it significantly more expensive in terms of on-die SRAM.

    *   **Flexibility vs. Overhead**:
        *   `16x16` Granularity: Provides a good balance. It's fine enough for status bars, water effects, or splitting the screen for different rendering styles without excessive memory overhead. It aligns well with common tile sizes (e.g., 16x16 tiles).
        *   `8x8` Granularity: Offers much greater creative control, allowing for smaller, more intricate effects, text boxes, or localized post-processing. However, this comes at the cost of the increased memory footprint and higher processing overhead.

    *   **Performance & Hardware Impact**:
        *   **PMID Fetch**: The hardware needs to fetch the correct PMID for each 16x16 or 8x8 block of pixels. A larger map (`8x8`) may increase cache pressure if the map is cached, or require more bandwidth if fetched directly from SRAM.
        *   **RISC-V Controller**: The CPU is responsible for updating this map. A map that is 4x larger will take 4x longer to clear or modify, consuming valuable CPU cycles during VBlank. For dynamic effects that change the map every frame, this could become a bottleneck.

*   **Recommendation**:
    *   Commit to **`16x16` granularity as the baseline standard** for RETRO. This provides a strong balance of flexibility and resource efficiency, which is core to the project's philosophy. It is sufficient for the majority of targeted use cases (status bars, screen splits, localized effects) without imposing a significant memory or performance tax.
    *   Treat `8x8` granularity as a **potential future extension or a build-time option**, but not a requirement for the reference implementation. This allows the core design to remain lean while leaving the door open for more advanced use cases if a specific project can afford the 4x increase in resource consumption. By not requiring it in the baseline, we simplify the hardware and firmware for the majority of users.
*   **Decision**: **`16x16` baseline**, with `8x8` as a possible non-standard extension.

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
