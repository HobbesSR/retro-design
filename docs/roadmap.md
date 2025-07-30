# RETRO Project Roadmap

This document outlines the high-level roadmap for the RETRO project. The timeline is aspirational and will evolve as the project progresses. The phases are sequential, but some tasks may be parallelized.

---

### Phase 1: Architecture & Feasibility (Current Phase)

*   **Goal**: Solidify the core architecture and make key design decisions.
*   **Key Activities**:
    *   Develop and refine the White Paper and Requirements documents.
    *   Identify and resolve architectural conflicts and open questions (see `studies/design_conflicts.md`).
    *   Create detailed block diagrams for the major hardware units.
    *   Define the memory map and register interfaces.
    *   Establish SystemVerilog coding standards and best practices for the project.
    *   Set up the initial CI/CD pipeline for linting and simulation.

---

### Phase 2: HDL Implementation & Simulation

*   **Goal**: Implement the core RETRO architecture in SystemVerilog and verify its functionality in simulation.
*   **Key Activities**:
    *   Implement core modules:
        *   Memory System (SVT, Bank Controllers)
        *   RISC-V Scene Controller
        *   Tile & Sprite Engines
        *   Triangle/Quad Engine
        *   Compositor & Pixel Pipelines
        *   Shader-Lite Units (VME, VI, PPM)
        *   DMA & Block Operation Engines
    *   Develop comprehensive testbenches for each module.
    *   Create a top-level simulation environment to test the integrated system.
    *   Write basic firmware examples to drive simulations.

---

### Phase 3: FPGA Prototyping & Validation

*   **Goal**: Validate the RETRO design on real hardware using a common FPGA development board.
*   **Key Activities**:
    *   Select a target FPGA board (e.g., a board from the Terasic DE10 family, or a low-cost Lattice ECP5 board).
    *   Synthesize the SystemVerilog design for the target FPGA.
    *   Develop a top-level wrapper to connect RETRO to the FPGA's I/O (DPI video, PSRAM, etc.).
    *   Port firmware examples to run on the FPGA.
    *   Debug and validate hardware features, performance, and timing.
    *   Create simple demos to showcase capabilities (e.g., tile scrolling, sprite rendering, 3D objects).

---

### Phase 4: Reference Hardware & SDK

*   **Goal**: Design and produce a reference hardware board ("RETRO Console") and a basic Software Development Kit (SDK).
*   **Key Activities**:
    *   Design the schematic and PCB for the reference board.
        *   Features: RETRO core (as soft-core on FPGA or future ASIC), video output, controller ports, audio, SD card slot.
    *   Produce a small batch of prototype boards for developers.
    *   Develop a basic SDK with libraries, toolchain, and examples for writing software for RETRO.
    *   Write clear, developer-focused documentation.

---

### Phase 5: Community & Ecosystem Growth

*   **Goal**: Foster a self-sustaining community of developers and creators around the RETRO platform.
*   **Key Activities**:
    *   Publish all design files, source code, and documentation under the CERN-OHL-S license.
    *   Create a website and community forum/chat for support and collaboration.
    *   Promote the project within the homebrew, demoscene, and open-source hardware communities.
    *   Establish a "RETRO Certified" program for derivative hardware that meets compatibility standards.
    *   Organize game jams, demo competitions, and other community events.
