# RETRO - White Paper

*A low‑power, deterministic, neo‑retro video processor that blends tile/sprite PPUs with PSX/N64‑era features, optimized for on‑die SRAM and memory‑mapped simplicity.*

---

## 1. Purpose & Vision

RETRO delivers classic console immediacy (scanline compositing, tiles/sprites, raster tricks) with modern embedded pragmatism (memory‑mapped control, a tiny RISC‑V scene controller, low power). It targets hobbyist TFTs and embedded products at **30–60 FPS**, emphasizing **on‑die SRAM used purposefully** and optional external PSRAM for larger scenes.

**Core principles**

*   **Deterministic scanline generation** with a small line buffer (no mandatory full‑frame RMW passes).
*   **Memory‑mapped everything**: data‑as‑control; write memory, hardware reacts.
*   **Mode‑aware flexibility**: tile‑dominant, poly‑dominant, and mixed profiles.
*   **On‑die SRAM optimized for role** via a *SRAM View Table (SVT)*.
*   **Novel compositing**: per‑tile pipeline selection (PMID map) and DMA‑tagged overrides.
*   **Optional primitive shader‑like features**: low‑power vertex‑like transforms and programmable pixel operations that extend CMU functionality.

---

## 2. High‑Level Requirements

*   **Output**: Parallel RGB **DPI** (RGB666 primary, RGB888 optional); 30–60 Hz; supports 320×240 (baseline), 480×272 (extended), optional 800×480 via PSRAM.
*   **Framebuffer (on‑die)**: **RGB6666** (6:6:6 + 6‑bit Aux) **double‑buffered at 320×240** (~600 KB). Aux channel = alpha/depth/special.
*   **Addressing model**: framebuffer supports **3‑byte word addressing** (24‑bit pixel words) *and* packed/unpacked bridges for DMA. Internally, banks are power‑of‑two aligned.
*   **Host I/O**: Memory‑mapped bus via SPI/QSPI/AHB‑like bridge; optional **QSPI/OPI PSRAM** for large assets/FB planes.
*   **RISC‑V scene controller**: RV32IMC (80–120 MHz) with light SIMD and memory‑mapped accelerators (blit/fill/edge/palette/HSL/vertex).

**Risks & mitigations:**

*   PSRAM latency could starve pipelines → **require sequential bursts and prefetch scheduling**.
*   On‑die FB writeback vs scanout hazards → **strict double buffering enforced**.
*   Pixel pipe overload → **cap sources per PMID (≤3) and limit per‑pixel recipe complexity to 3–4 cycles**.

---

## 3. Mode Hierarchy

**High‑level modes** (select at VBlank; sub‑modes tuned recipes):

*   **A) Tile‑Dominant**: Multi‑BG tilemaps, sprites, affine (Mode‑7‑like), palette math.
*   **B) Poly‑Dominant**: PSX‑style quads w/ ordering tables, optional perspective tris, line‑Z or FB‑Z.
*   **C) Mixed**: Hybrid 2D+3D, post‑FX, selective writeback and FB sampling.

Per‑tile **PMID map** (16×16 default granularity; 8×8 optional) selects sources (BG/SPR/TRI/FB), a combiner recipe, and feature flags (Z test, bilinear, HSL enable, HUD‑bypass fog, shader enable, etc.).

**Risk:** Mixed Z policies could conflict → **band‑level policy: one Z type per band, tiles may disable but not change Z type.**

---

## 4. Memory‑Mapped Architecture

Everything appears as **logical regions** within a single address space; writing data triggers side‑effects (cache invalidates, prefetch, wakes). Control is structured as small **config blocks** (not scattered bitfield registers).

### 4.1 SRAM View Table (SVT)

A compact table the firmware writes at VBlank to bind **physical SRAM banks** to **logical regions** and their **ports/policies**. Enables role‑optimized SRAM without hard partitioning the design.

**Logical region types (examples)**

*   **Tile Pattern Cache** (read‑mostly; row‑aligned; prefetch‑friendly)
*   **Tilemap / ModeMap** (tile entries + per‑tile PMID)
*   **Sprite Attributes / Pattern Rows**
*   **Ordering Tables (OT) / Edge & Span Buffers**
*   **Line‑Z / Coverage**
*   **Combiner & HSL/Shader Param Tables / Palettes**
*   **Framebuffer(s) RGB6666** (scanout + selective writeback)
*   **DMA rings & descriptors**

**Risk:** SVT rebinds could cause conflicts → **limit rebinds to VBlank and provide build‑time SVT conflict analyzer.**

### 4.2 Porting strategy (Area/Power conscious)

*   **Dual‑port where concurrency is real**: Tile Pattern Cache, Sprite Attrs, On‑die FB (scanout vs writeback).
*   **Single‑port elsewhere** with phase separation (prefetch vs shade): OT, Edge, Line‑Z, LUTs, DMA rings.

### 4.3 Multi‑purpose bank pairings (typical)

*   **Tilemap/ModeMap ↔ Ordering Tables** (2D vs PSX‑like 3D)
*   **Line‑Z ↔ Line scratch / Sprite overflow** (3D vs 2D)
*   **Combiner/HSL/Shader tables ↔ Palette RAM**

---

## 5. Framebuffer & Addressing Choices

### 5.1 RGB6666 on‑die FB

*   **6:6:6 RGB + 6 Aux**; Aux used per‑mode for alpha, coarse depth, or pipeline tags.
*   **Double‑buffered 320×240** on‑die; 480×272 single‑buffer is plausible; **800×480** via PSRAM.

**Mitigation:** Strict double buffering; writeback never targets the active scanout buffer.

### 5.2 24‑bit word addressing (3‑byte) — implications

*   Convenient for **packed DMA** and external RGB888 interfacing; awkward for raw byte math.
*   We hide misalignment by using **wide internal buses** and line‑oriented access; firmware rarely touches pixels randomly.

### 5.3 Alternative: channel‑separated internal layout (hybrid)

*   Internally store R, G, B, Aux in aligned banks (1B/chan), reconstruct at combiner/scanout.
*   Externally **expose packed 24‑bit** path for DMA transfers.
*   Benefits: clean addressing, cheap per‑channel ops (LUT/HSL/shader), easy upgrades for Aux (e.g., 8‑bit alpha or 16‑bit Z plane when needed).

**Recommendation:** Use the **hybrid**—internally channel‑separate for aligned math and block ops; pack/unpack at DMA boundary to keep compatibility and compact storage.

---

## 6. Pipelines (Fixed‑Function, Scanline‑Driven)

### 6.1 Tile Engine (BG layers)

*   Tile sizes: **4/8/16/32/64** (square, power‑of‑two).
*   Features: per‑layer affine (Mode‑7‑like), line/column scroll, mosaic, palette select, **per‑tile transforms** (H‑flip, V‑flip, 90/180/270° rotations via address swizzles).
*   **Tile entry schema** (select via map schema register):
    *   **16‑bit compact**: tile\_id (hi/lo), rot(2), v/h flip, palette/priority.
    *   **32‑bit extended**: tile\_id(16), palette/bank(4), rot(2), v/h flip, **per‑tile PMID override**, misc flags.

### 6.2 Sprite Engine

*   96–128 sprites, up to 64×64, per‑sprite affine, palette or direct color, mesh or true alpha.
*   Per‑scanline active list; row caches; priority & window masks.

**Mitigation:** Cap active sprites per line (e.g., ≤128) to ensure RISC‑V and prefetchers meet timing.

### 6.3 Tri/Quad Engine

*   **PSX‑style quads** (affine map, Gouraud) with **ordering tables** (no Z) *or* perspective‑correct triangles with **line‑Z** (optional FB‑Z plane).
*   Texture sampling: nearest; optional bilinear for FB sampling (gated by PMID).

**Risk:** OT and Z could conflict if mixed per tile → **enforce mutual exclusivity per PMID.**

### 6.4 Compositor (Combiner)

*   Inputs: BG, SPR, TRI, FB‑A/B sample, CONST/FOG, **CMU (HSL/YUV) stage**, **Shader Stage**.
*   Recipes: 8–16 small opcode pipelines (add/sub/lerp/min/max/XOR + alpha factors + shader stage outputs).
*   **Z‑test**: line‑Z or FB‑Z if enabled. Ordered painter’s mode when Z off.

**Mitigation:** Limit max sources per PMID (e.g., ≤3) and gate HSL and shader stage by PMID to meet pixel budget.

### 6.5 Shader Stage (Primitive Shaders)

*   Optional programmable stage that executes **small vertex‑like transforms** or **pixel‑level arithmetic** on attributes (position, color, depth).
*   Reuses the same infrastructure as CMU, parameterized per PMID.
*   Examples:
    *   Vertex‑like: apply translation/scale/rotation to coordinates before rasterization.
    *   Pixel‑like: apply conditional blending, color modulation, or additional math beyond CMU.
*   Parameters stored in a **Shader Param Table**; operations are fixed‑cycle and heavily constrained to avoid stalls.

**Mitigation:** Restrict shader ops to a small, deterministic instruction set (e.g., ≤4 ops per pixel/vertex) and clock‑gate when disabled.

---

## 7. Color Math Unit (CMU): HSL‑style Hardware

**Live per‑pixel** and **block‑op** HSL editing without general shaders.

*   **Live path (combiner stage)**: RGB→YUV → hue‑rotate (2×2), saturation scale, lightness gain/offset → RGB → dither.
    *   Parameters indexed by PMID → **HSL Param Table** (e.g., 16 rows; `{θ, s, g, o, flags}` in fixed‑point).
    *   Optional gamma LUTs for linearization when applying lightness.
*   **Block‑op path**: apply HSL over FB regions, tile atlases, or palettes.
    *   Modes: **YUV‑fast** (same as live) or **Exact HSL** (piecewise max/min; slower, offline only).

**Fixed‑point**: cos/sin LUT (Q1.8), YUV Q formats; 1 px/cycle when enabled; fully clock‑gated when not in use.

**Mitigation:** Do CMU math at ≥9‑bit internal and ordered‑dither back to RGB666 to avoid banding.

---

## 8. Shader‑Lite Programmability (Microcode Units)

Add **primitive, low‑power “shader‑like” features** that extend flexibility without turning RETRO into a general shader GPU. Three tiny, fixed‑budget units cover the useful space: a **Vertex Micro‑Engine (VME)**, a **Varying Interpolator (VI)**, and **Parametric Pixel Math (PPM)** that integrates with the combiner/CMU.

### 8.1 Vertex Micro‑Engine (VME)

A small, deterministic micro‑engine that runs **per‑vertex** programs to prepare rasterization parameters.

*   **Purpose**: apply **2D/2.5D transforms**, generate varyings (e.g., UVs, color), and set per‑primitive constants (fog factor, combiner constants), without waking the host CPU.
*   **Instruction set (fixed‑point, no loops)**: `ADD`, `MUL`, `MAD`, `SHL/SHR`, `CLAMP`, `DOT2/3`, `MUL_MAT2x3`, `RECIP_APPROX` (optional), `PACK`. All Q formats are implementation‑fixed (e.g., 16.16 or 12.4 depending on target).
*   **State**: 16 scalar registers, 16 constants, program length ≤ **32 ops**. Program memory ≤ **1 KB**; selected by **sub‑mode** or draw‑call header.
*   **Outputs**: screen‑space `{x,y}`, **depth `z`**, up to **two varyings** `v0,v1` (e.g., UV or color), and optional `w` for perspective.
*   **Interface**: VME writes **edge/plane coefficients** and **varying seeds** into the **Edge & Span Buffers** (logical region). The **Tri/Quad Engine** consumes them.

**Examples**

*   2D affine sprite batches (matrix apply + color scale per sprite).
*   Billboards: face camera by building a 2×2 basis and offsetting vertex quad.
*   Simple lighting: `color = dot(n, l) * base` for flat‑lit polys.

**Risk & Mitigation**

*   *Risk*: VME cannot process all vertices in time for the next band.
    *Mitigation*: cap vertices per band; provide a **vertex FIFO depth** and **back‑pressure counter**; allow RISC‑V to pre‑bake vertices for overflow cases.

### 8.2 Varying Interpolator (VI)

*   **Function**: hardware computes per‑span/per‑pixel interpolation of **up to two varyings** selected by the VME. Modes: **affine** or **perspective‑correct** (uses `w`/`z`).
*   **Setup**: VME emits plane equations; VI converts to **per‑span steps** during the **prefetch phase**.
*   **Delivery**: interpolated `v0,v1` available as inputs to the **Combiner** and/or **CMU**.

**Risk & Mitigation**

*   *Risk*: Allowing >2 varyings explodes area/latency.
    *Mitigation*: **Hard‑limit to two**; if more are needed, encode additional effects via **PMID recipes** or pre‑bake into textures.

### 8.3 Parametric Pixel Math (PPM)

A tiny **per‑pixel arithmetic stage** that complements CMU.

*   **Ops**: `MUL`, `ADD`, `MAD`, `SAT`, `THRESH`, and **1D LUT sample**. Operates on **combiner inputs** and/or `v0,v1` from VI.
*   **Use cases**: distance‑based fog from `z`, toon thresholds, color modulate by v0, alpha from LUT(v1), cheap dissolve masks.
*   **Control**: **PMID recipe** selects **one of N PPM programs** (e.g., N=8) stored in a small table (each 3–5 ops max).
*   **Placement**: **Before** CMU (so HSL can act on PPM‑modified RGB) or **after** (for post‑grade thresholds), selectable per recipe.

**Risk & Mitigation**

*   *Risk*: PPM complexity jeopardizes the **3–4 cycle** pixel pipe budget.
    *Mitigation*: compile‑time **op‑budget per recipe**; hardware rejects configurations that exceed the budget; **≤3 sources** per PMID enforced.

**Notes**

*   The existing **CMU (HSL)** acts as a specialized “pixel shader‑lite” stage; PPM provides the general arithmetic glue without opening a full shader model.
*   All shader‑lite features are **fully clock‑gated** when unused.

---

## 9. DMA‑Aware Compositing & Block Operations

### 9.1 DMA with PMID override

*   DMA descriptors for FB/tilemap writes carry an optional **PMID override** and combiner flags. Blocks (e.g., 16×N) inherit that pipeline behavior without touching the main ModeMap.

### 9.2 Block/Region Operations (memory‑mapped descriptors)

Lightweight engines that walk rectangular regions using wide bank writes/reads:

*   **FILL**: constant color/Z/tile ID/attr.
*   **BLIT**: src→dst copy (same or cross‑bank), masked or unmasked.
*   **LUT**: apply 1D LUT to a region or palette (posterize, gamma tweak).
*   **VECTOR‑ADD**: add Δ to sprite attr blocks (e.g., camera pan).
*   **COMPARE/MASKED‑WRITE**: conditional region ops (e.g., Z<th).
*   **TILE‑RNG‑FILL**: procedural tilemap fill using seeded LFSR/xorshift with optional lookup table, no‑repeat constraint, and transform bit probabilities (H/V/rot). Optional 8×8 weight map biases distribution.
*   **HSL\_BLT**: region HSL edits (YUV‑fast or exact HSL) with gamma flags.
*   **SHADER\_BLT**: execute primitive shader programs on blocks of pixels or vertices using the Shader Stage.

**Mitigation:** Epoch bit system to avoid DMA/ModeMap races (only apply overrides for current epoch).

All exposed as simple **descriptor structs** in memory; engines run concurrently if banks don’t conflict.

---

## 10. Scheduling & Timing (Meeting Deadlines)

*   **Phases per line**: Prefetch (−2…−1 lines), Shade (current), Scanout (current) with small elastic FIFOs.
*   **Miss policy**: Fallback color on cache miss; perf counters record late/miss stats for tuning.
*   **Epoch flip**: Double‑buffered control pages (SVT, combiner/HSL tables, mode config, **PPM programs**) swapped at VBlank or region boundary.

**Mitigation:** Hardware fault counters for SVT conflicts, ModeMap epoch mismatches, OT/Z invalid combinations, and **VME/VI underflow** (so firmware can throttle draw calls).

---

## 11. RISC‑V Scene Controller & Accelerators

*   RV32IMC + light SIMD (packed 8/16‑bit ops) accelerates affine steps, list builds, palette math.
*   Writes **Scanline Recipes** and maintains OT bins, edge spans, sprite lists, ModeMap updates.
*   Programs block‑ops by writing descriptors into **per‑bank command FIFOs** (also memory‑mapped regions).
*   **Feeds shader‑lite**: loads **VME programs**, **PPM tables**, and HSL params; validates recipe op‑budgets.

**Mitigation:** Provide assist accelerators for span/OT building so RISC‑V enqueues rather than computes in real‑time; expose **vertex FIFO depth** to firmware for back‑pressure control.

---

## 12. Developer Model (Ergonomics)

*   **Write memory, not registers**: tilemaps, sprites, OT, HSL/combiner/shader tables, FB—everything is a region.
*   **Switch modes** by writing a small **Mode Control Block (MCB)** that points to SVT + recipe tables; flip at VBlank.
*   **Per‑tile artistry**: rotate/mirror flags; per‑tile PMID overrides; color grading via CMU; shader toggles.
*   **Procedural content**: RNG tile fills, LUT/HSL ops, and Shader Stage programs with one descriptor write.

**Mitigation:** Freeze a canonical memory map and headers/codegen from spec to keep developer UX clean.

---

## 13. Open Decisions / Tunables

*   **PMID granularity**: 16×16 default, 8×8 optional.
*   **Internal layout**: commit to **hybrid** (channel‑sep internal + 24‑bit packed DMA), or allow build‑time selection.
*   **Perspective path**: always available or gated by a perf bit.
*   **Bilinear**: FB sampling only (post‑FX) vs also for textures.
*   **Panel baseline**: 320×240 vs 480×272 as the dev‑kit default.
*   **Shader Stage**: define instruction set and resource limits (≤4 ops/pixel and ≤8 ops/vertex recommended).

---

## 14. Appendix — Example Structs (Sketch)

**PMID Map Entry (1 byte)**
```
bit7    : HSL enable
bit6    : Shader enable
bit5    : Z test enable / FB bilinear (mode‑dep)
bits4:3 : Combiner index (0..3 or 0..15 in extended)
bits2:0 : Pipeline select (BG/SPR/TRI/FB_A/FB_B/mixed)
```

**Shader Param Table Row (example)**
```
uint8 ops[4];       // up to 4 fixed ops (encoded)
int16 params[4];    // params for ops (e.g., scale, offset, threshold)
```

**HSL Param Table Row (8 bytes)**
```
uint8  theta_idx;   // 0..255 ≈ 0..360°
uint8  sat_q1_8;    // 0..2.0×
uint8  gain_q1_8;   // 0..2.0×
int8   offset_q1_8; // −1..+1
uint8  flags;       // gamma on/off, clamp mode, etc.
uint8  reserved[3];
```

**Tile Entry (32‑bit extended, schema‑dependent)**
```
[31:16] tile_id
[15:12] palette/bank
[11:10] rot (0,90,180,270)
[9]     v_flip
[8]     h_flip
[7:4]   per‑tile PMID/combiner/shader override
[3:0]   misc flags (blend, mask, size override)
```

**TILE_RNG_FILL Descriptor (sketch)**
```
struct RNGFill {
  u32 dst_base, stride_tiles;
  u16 width_tiles, height_tiles;
  u32 seed;            // LFSR/xorshift seed
  u32 table_base;      // optional tile ID table
  u16 table_len;       // 0 => uniform over range
  u8  mode;            // bits: algo | no_repeat | mask_write
  u8  prob_rot_flip;   // packed probabilities (rot/flip weights)
};
```
