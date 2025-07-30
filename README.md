# RETRO - Real-time Embedded Tile & Raster Orchestrator

**Design lab for the RETRO chip (Real-time Embedded Tile & Raster Orchestrator) – open hardware docs, studies, and prototypes.**

[![License: CERN-OHL-S v2.0](https://img.shields.io/badge/License-CERN--OHL--S%20v2.0-blue.svg)](https://cern.ch/cern-ohl)

---

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

---

## ⚠️ Work in Progress

This repository is in the early stages of design and experimentation. The architecture, documentation, and code are subject to radical change. Nothing here is final.

## About The Project

RETRO is an open, copyleft hardware design for a neo-retro graphics processor. It blends the deterministic, scanline-based rendering of classic 2D consoles with features from the PSX/N64 era, optimized for low-power embedded systems, FPGAs, and homebrew projects.

The goal is to create a fun, accessible, and powerful graphics core that can serve as the heart of custom-built consoles and creative hardware projects for the demoscene and hobbyist communities.

This repository contains the collaborative design files, including:
*   **White Paper:** The high-level vision and architecture.
*   **Requirements:** Detailed technical specifications.
*   **Roadmap:** The plan for development and release.
*   **Feasibility Studies:** Analysis of design choices and conflicts.

## Getting Started

To understand the vision and architecture of RETRO, please start by reading the documents in the `/docs` directory:

1.  **[White Paper](./docs/whitepaper.md)**
2.  **[Requirements](./docs/requirements.md)**
3.  **[Roadmap](./docs/roadmap.md)**
4.  **[Comprehensive Analysis](./docs/analysis.md)** - For a deep dive into the design context and comparisons.

## License

This project is licensed under the **CERN Open Hardware License v2 - Strongly Reciprocal (CERN-OHL-S)**. See the [LICENSE](./LICENSE) file for details.
