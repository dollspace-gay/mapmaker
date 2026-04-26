---
title: "Realistic Planet Generator"
tags: ["design-doc"]
sources: []
contributors: ["iWst"]
created: 2026-04-26
updated: 2026-04-26
---

# Feature: Realistic Planet Generator

## Summary

A browser-based desktop-class application that generates richly realistic worlds end-to-end — from plate tectonics through climate to full procedural civilizations — by simulating the underlying physics rather than drawing maps from aesthetic priors. Target hardware is a workstation-class GPU (NVIDIA RTX 3090 / 24 GB VRAM / 64 GB system RAM); the design embraces "the expensive route" wherever a costlier model produces a more geologically or planetologically defensible result. Output is an interactive 3D globe plus exportable 2D maps and worldbuilding artifacts (cultures, religions, polities, conflicts). Source spec: `design.md` (v1, April 2026); this design doc supersedes it on points where they disagree.

## Requirements

- **REQ-1 (GPU-native simulation).** All per-cell simulation state lives in `GPUBuffer` objects; the inner simulation loop performs no readback to JavaScript. CPU code only orchestrates dispatches, handles graph-topological events (plate split/merge), validates constraints, and marshals exports. (`design.md` §2, §4.2.)
- **REQ-2 (Geological causality).** Every visible feature on the final map — mountains, deserts, river networks, biomes, settlement clusters, cultural distributions — is downstream of simulated physics or causally-grounded procedural rules rather than aesthetic placement. (`design.md` §2.)
- **REQ-3 (Speed budget on workstation hardware).** Target hardware: NVIDIA RTX 3090 (or comparable: RTX 4080+, RX 7900 XTX) with 24 GB VRAM and 64 GB system RAM. At default mesh level 8 (~655 k cells) with full simplified GCM enabled: cold full simulation ≤ 90 s; warm reroll (same parameters, new seed) ≤ 30 s; non-topological parameter tweak ≤ 60 s. At ultra mesh level 9 (~2.6 M cells): cold sim ≤ 5 min; warm reroll ≤ 2 min. Regional refinement at level 10 (~10.5 M cells, clipped subset only) ≤ 5 min. (`design.md` §2, §8 — budgets revised for 3090 baseline.)
- **REQ-4 (Hard physical invariants).** The simulation enforces, every run, all of: rivers merge going downstream *except* at cells flagged `is_delta`, `is_explicit_bifurcation` (user-painted), or `is_natural_bifurcation` (model-detected low-gradient cross-divide flow per REQ-19); total plate divergence rate equals total convergence rate within numerical tolerance; drainage basins partition the surface (every land cell drains to exactly one *primary* outlet — bifurcations have a primary plus secondary outlet); crust age monotone non-decreasing from spreading center to subduction zone; hotspot tracks aligned with plate motion direction. Violation is a bug, not a tradeoff. (`design.md` §2; resolves Q6.)
- **REQ-5 (Mesh — high-resolution defaults).** Subdivided icosahedron with selectable level: 6 (~41 k, fast preview), 7 (~164 k, "draft quality"), **8 (~655 k cells, ~31 km/cell — default global)**, **9 (~2.6 M cells, ~16 km/cell — "ultra" global option)**, **10 (~10.5 M cells, ~8 km/cell — regional refinement only, clipped subset)**. Twelve pentagonal cells per level handled correctly via fixed-size neighbor table with `INVALID = 0xFFFFFFFF` sentinel. WebGPU device must be requested with `maxStorageBufferBindingSize` near hardware max (~2 GiB on RTX 3090) to accommodate level 10 buffers. (`design.md` §4.1; resolves Q1 — level 8 default, level 10 regional refinement, vs original level 6 / level 8 split.)
- **REQ-6 (Pipeline order).** Stages execute sequentially with documented per-stage GPU-buffer outputs: **Tectonics → Erosion (coupled with Hydrology iterations) → Hydrology (with natural-bifurcation detection) → Climate (full simplified GCM, see REQ-18) → Biomes → Regional Refinement (spherical, see REQ-16) → Civilization (settlements + trade + names + cultures + religions + polities + conflicts, see REQ-17)**. Each stage emits an inspectable snapshot the next consumes. Partial reruns skip stages whose inputs are unchanged. (`design.md` §5.)
- **REQ-7 (Layer 1 parameters).** All twenty-something global scalar parameters across Astronomical, Tectonic, Hydrosphere, and Biosphere groups are user-adjustable via sliders / numeric inputs / toggles. Seven named presets ship: Earth-like, Mars-like, Ocean world, Young volcanic, Old worn world, Pangaea, Archipelago. (`design.md` §6.1.)
- **REQ-8 (Layer 2 constraints).** Six paint-brush constraint types are supported: plate, boundary (convergent/divergent/transform), feature pin (mountain range, volcanic arc, rift, hotspot, ancient orogen), climate pin, coastline pin, settlement pin (Layer 2.5). Click-drag paints, right-click erases, esc cancels; curve constraints place control points and double-click to finish. (`design.md` §6.2, §7.5.)
- **REQ-9 (Constraint validation).** Three layers: (a) hard validation at input time, surfaced as red/yellow overlays plus a panel listing each issue with location and ≥1 suggested fix; (b) soft constraints folded into an equilibration objective during simulation; (c) actionable resolution suggestions when validation fails (e.g. "adjust Euler pole to X" / "shorten hotspot activation history"). Validation runs continuously during paint, debounced to ~200 ms. (`design.md` §6.3, §7.5.)
- **REQ-10 (WebGPU only in v1).** No WebGL2 fallback ships in v1. When `navigator.gpu` is unavailable or `requestAdapter()` fails, display a clear "WebGPU required" message naming supported browsers (Chrome / Edge ≥ 113, Firefox Nightly, Safari Tech Preview). Reconsider WebGL2 fallback after 12 months of coverage data. (`design.md` §3.)
- **REQ-11 (Worker-isolated simulation).** The simulation pipeline runs in a single Web Worker that exclusively owns the WebGPU device and all GPU buffers. The main thread holds no `GPUBuffer` references. Communication uses Comlink for control RPC; state snapshots transfer to the main thread via `SharedArrayBuffer` for buffers large enough to matter (heightmap, plate map, biome map, climate fields). At level 9/10, snapshot reads chunk over multiple frames to avoid stalling rendering. (`design.md` §3, §7.4.)
- **REQ-12 (Persistence).** Saved planets, simulation snapshots, user presets, and full civilization state (settlements, polities, languages, religions, conflict log) persist in IndexedDB via the `idb` library. Save → reload → load roundtrips Layer 1 + Layer 2 + simulation snapshot + civilization state identically. Snapshot compression (per-cell quantization where lossless; e.g. f32→f16 for bounded fields) used for level 9+ where uncompressed snapshots would exceed practical IndexedDB blob sizes. (`design.md` §3, §10.)
- **REQ-13 (Render performance).** Globe rendering uses a single instanced draw call with cell colors sourced from the active layer buffer; target 60 fps regardless of simulation state, even at level 9 (~2.6 M cells). Sim progress updates flow to UI without blocking input. (`design.md` §7.6.)
- **REQ-14 (Exports — geographic only in v1).** Export pipeline produces valid, openable files for: PNG (equirectangular projection, with optional Mercator / Robinson / orthographic alternates), SVG, GeoJSON / TopoJSON. **Foundry VTT and Roll20 exports are deferred to post-v1** at user direction (not permanently excluded — see Out of Scope). (`design.md` §10, §14; resolves Q5.)
- **REQ-15 (Plate topology events on CPU — enabled by default).** Plate split, plate merge, and rift propagation are graph-topological events that run CPU-side. Defaults: enabled, with thresholds tuned to produce **3–5 topology events per 500 Myr default sim** (matching observed Earth Wilson-cycle frequency). A "freeze plate topology" toggle disables splits/merges for users who want determinism across rerolls. CPU-side mutation completes within one frame (≤ 16.7 ms) at level 8. (`design.md` §4.3; resolves Q2 — defaults enabled, target frequency specified.)
- **REQ-16 (Regional refinement — spherical mesh).** Stage 6 builds a high-resolution local mesh by **subdividing the global icosahedron to level 10 within the selected region (sphere-clipped subset)**, not by projecting to a flat plane. Erosion / hydrology / climate kernels share a single mesh-type abstraction so they run unchanged on the regional clipped subset. The global flow network is a hard constraint on the local resampling — rivers that exist at global resolution must persist at regional resolution and discharge to the same outlets. (`design.md` §5 Stage 6; resolves Q7 — spherical, not planar.)
- **REQ-17 (Full civilization layer).** Stage 7 generates the complete worldbuilding output, with each sub-stage producing a documented snapshot the next can build on:
  - **17a. Settlements.** Weighted Poisson-disk on a score field (river/confluence, sheltered harbor, defensibility, arable proximity, trade chokepoint, climate harshness). Hierarchy: capitals, cities, towns, villages.
  - **17b. Trade routes.** Dijkstra on the cell graph with terrain-derived edge weights; rivers and coastal seaways preferred, mountains and deserts avoided.
  - **17c. Languages & names.** 4–8 language families with **proper morphological grammar** (phoneme inventory + syllable templates + morphological rules + per-family sound-change history); names within a region share inventory and rule-set, names across regions diverge measurably (resolves Q4).
  - **17d. Cultures.** Per-region cultural traits derived from language family + climate + economic base (agrarian / pastoral / maritime / forager) + dominant biome. Documented as a feature vector that downstream stages consume.
  - **17e. Religions.** Pantheon generation with deity templates (sky, earth, sea, harvest, war, trickster, death, etc.); ritual practices, symbolic systems, holy sites tied to physical features (sacred mountains, rivers, springs). Religion families map onto culture clusters with optional cross-cultural syncretism along trade routes.
  - **17f. Polities.** State formation by clustering settlements weighted by cultural similarity, river/road connectivity, and terrain-bounded contiguity. Borders prefer terrain features (rivers, ridges, coastlines). Multi-tier (empires / kingdoms / city-states / chiefdoms / tribes).
  - **17g. Conflicts & history.** Procedural conflict log generated from: resource scarcity (climate-driven famines), cultural friction (border tensions between dissimilar cultures), succession disputes, religious schisms. Recorded as a structured timeline of events with locations, polities involved, and outcomes.

  (`design.md` §5 Stage 7 + scope expansion per user direction; resolves Q8.)
- **REQ-18 (Climate — simplified GCM with seasons and monsoons).** Climate stage replaces the latitudinal-only wind model with a **two-layer atmospheric simplified GCM** with a 4-season cycle. Per-cell prognostic variables: temperature, moisture, momentum (u, v) per layer. Forcing: seasonal solar insolation (function of latitude × axial tilt × eccentricity), surface heating (continental cells heat/cool faster than oceans → drives monsoons), Coriolis (function of rotation rate parameter), orographic lifting per `precipitation.wgsl`. ITCZ migrates seasonally; Hadley/Ferrel/Polar cells emerge with realistic seasonal shifts; monsoonal continental-heating-driven seasonal wind reversal emerges from the dynamics. Run 50–100 sub-steps per season, then average to mean-annual climate fields. (`design.md` §5 Stage 4; resolves Q3 — full GCM, not latitudinal.)
- **REQ-19 (Natural river bifurcations — Casiquiare-style).** Hydrology stage models naturally-occurring bifurcations at low-gradient cells where flow can plausibly route to multiple downstream basins. Detection criterion: cell with downstream gradient below threshold AND ≥2 candidate downstream neighbors with comparable elevation differences AND those neighbors lie in distinct drainage basins at the global topology. Such cells are flagged `is_natural_bifurcation` and their flow accumulation splits between the two outlets in proportion to gradient. AC-4 invariant test treats flagged cells correctly. (Resolves Q6 — natural bifurcations modeled, not just user-marked.)

## Acceptance Criteria

- [ ] **AC-1** (REQ-1): The inner simulation loop calls no buffer readback / `mapAsync` and the main thread holds no per-cell typed arrays except explicit snapshot reads at stage boundaries — verified by code review and a worker-side telemetry counter that asserts zero readbacks during a hot run.
- [ ] **AC-2** (REQ-2): Causality trace test — for any cell on the final map, an automated query can identify the upstream physical or procedural cause: mountain cells trace to a convergent boundary; desert cells trace to a windward elevation barrier or a 30°-band subtropical-high; biome cells are a deterministic function of (T, P) lookup; settlement cells trace to score-field components; cultural attributes trace to language family + climate + economic base.
- [ ] **AC-3** (REQ-3): Benchmark suite (10 runs each, p99) on RTX 3090 reports: at level 8 default — cold full sim ≤ 90 s, warm reroll ≤ 30 s, non-topological param tweak ≤ 60 s; at level 9 ultra — cold sim ≤ 5 min, warm reroll ≤ 2 min; regional refinement at level 10 ≤ 5 min. Numbers logged to a perf-budget CI artifact.
- [ ] **AC-4** (REQ-4): Invariant test suite, run at the end of every full sim, must pass all of:
  - Branching count = 0 across all land cells, except those flagged `is_delta`, `is_explicit_bifurcation`, or `is_natural_bifurcation`.
  - `|Σ divergence − Σ convergence| / max(Σ divergence, Σ convergence) < 1e-3`.
  - Every land cell has exactly one *primary* downstream successor; flagged-bifurcation cells additionally record one secondary outlet with proportional flow split.
  - Crust age strictly non-decreasing along all flow lines from spreading center to subduction zone.
  - Hotspot-track-vector vs plate-motion-vector angle within 15° at every track cell.
- [ ] **AC-5** (REQ-5): Mesh generator produces correct cell counts at levels 6–10 (40 962 / 163 842 / 655 362 / 2 621 442 / 10 485 762). Per-cell neighbor count test confirms exactly 12 pentagonal cells at every level. Sentinel `INVALID` slot handling verified by traversal test. WebGPU device acquired with `maxStorageBufferBindingSize` ≥ 2 GiB on RTX 3090; level 10 buffer allocation succeeds.
- [ ] **AC-6** (REQ-6): Each pipeline stage emits a snapshot consumable by the next via documented buffer contracts (`tectonic_state`, `erosion_state`, `hydrology_state`, `climate_state`, `biome_state`, `civilization_state`). Partial-rerun cache hits are observable in worker logs and skip dispatching unchanged stages.
- [ ] **AC-7** (REQ-7): Every Layer 1 parameter listed in `design.md` §6.1 surfaces in the UI with a control of appropriate type. All seven presets selectable; each produces a planet visibly distinct from defaults (verified by a perceptual hash diff threshold).
- [ ] **AC-8** (REQ-8): All six brush types paint constraint data that persists across page reloads; brush radius slider, click-drag, right-click erase, and esc-cancel are all functional in an end-to-end test.
- [ ] **AC-9** (REQ-9): A test fixture of intentionally inconsistent constraints (hotspot track misaligned with Euler pole; coastline crossing transform boundary; climate pin inconsistent with computed T/P) produces, for each, a red overlay, a panel entry with location, and ≥1 suggested fix. Soft-constraint relaxation reduces objective monotonically across iterations.
- [ ] **AC-10** (REQ-10): When `navigator.gpu` is undefined or `requestAdapter()` returns null, the app displays a "WebGPU required" message naming supported browsers; no white screen, no thrown error reaches the user.
- [ ] **AC-11** (REQ-11): Worker init creates the `GPUDevice` only inside the worker; main-thread heap inspection finds no `GPUBuffer` references. Comlink-exposed `SimWorker` interface matches `design.md` §7.4. Snapshots > 1 MB transfer via `SharedArrayBuffer` (verified by transferable-list test). At level 9 the snapshot read chunks over ≥ 4 frames without dropping render fps below 60.
- [ ] **AC-12** (REQ-12): Save planet → reload page → load planet roundtrip is bit-identical for elevation, plates, biomes, constraints, Layer 1 parameters, and full civilization state (settlements, polities, languages, religions, conflict log). Preset save/load roundtrips Layer 1 + Layer 2.
- [ ] **AC-13** (REQ-13): Render frame time ≤ 16.7 ms (60 fps p95) during a running sim at level 8 on RTX 3090. Sim progress events arrive at ≥ 1 Hz; UI button presses register within 100 ms during sim.
- [ ] **AC-14** (REQ-14): Each in-scope export format produces a file validated by its target tool: PNG opens in image viewers; SVG opens in browser without parse errors; GeoJSON validates against the geojson.org schema. Foundry VTT and Roll20 exports return a "deferred — post-v1" message in the export panel and do not appear in the v1 release notes.
- [ ] **AC-15** (REQ-15): Plate split fires when sustained extensional stress exceeds the documented threshold along a connected curve through a continental plate; merge fires after ≥ 50 Myr continuous collision with no relative motion. Default-tuned thresholds produce 3–5 topology events per 500 Myr sim across 100 random seeds (mean 4, σ ≤ 1). CPU-side topology mutation completes within one frame at level 8. "Freeze plate topology" toggle suppresses both events.
- [ ] **AC-16** (REQ-16): Regional refinement uses the same mesh-type abstraction as global; erosion / hydrology / climate kernels run unchanged on the level-10 sphere-clipped subset. For a planet at level 8 with `R` rivers discharging at `R` distinct ocean cells, the regional refinement of any chosen region preserves all enclosed rivers and routes them to the same coastline cells. Macro structure unchanged is verified by topological comparison; high-frequency tributaries appear only as additions.
- [ ] **AC-17** (REQ-17): Full civilization layer test:
  - **17a:** Capitals/cities/towns/villages counts roughly match Poisson-disk parameters at each tier; placements respect score field (verified by sample correlation with ranked score).
  - **17b:** Trade routes prefer rivers / coasts and avoid mountains / deserts (verified by edge-weight aggregation along generated paths).
  - **17c:** Within-region settlement names share phoneme inventory above a cosine-similarity threshold; cross-region pairs below. Sound-change history per language family produces visibly diverged daughter forms when language family fissions.
  - **17d:** Cultural feature vectors cluster by language family + climate; clusters visible on a 2D projection.
  - **17e:** Generated religions have non-empty pantheons with named deities and ritual practices; holy-site cells correlate with notable physical features (peaks, river headwaters, springs).
  - **17f:** Polity borders prefer terrain features (≥ 60 % of border length follows river / ridge / coast cells).
  - **17g:** Conflict log non-empty for any default-parameter run with ≥ 4 polities; events have causes traceable to resource scarcity, cultural friction, succession, or religious schism.
- [ ] **AC-18** (REQ-18): GCM produces the four canonical realism markers without manual tuning per scenario:
  - **ITCZ migration** between 10°N (boreal summer) and 10°S (austral summer).
  - **Monsoon** continental-heating-driven seasonal wind reversal over a continental block ≥ 1000 km wide between 10–30° latitude.
  - **Subtropical highs** (descending air → desert biomes) at 25–35° on continental west sides.
  - **Mid-latitude westerlies** with eastward-tracking storm activity.
  Verified by inspection on the Earth-like preset with default parameters.
- [ ] **AC-19** (REQ-19): On a planet with at least one low-gradient continental divide between two basins, hydrology stage detects ≥ 1 natural bifurcation; flagged cells satisfy: gradient < threshold, ≥ 2 candidate downstreams in distinct global basins, comparable elevation differences. Flow accumulation splits proportionally; AC-4 invariant test treats flagged cells correctly.
- [ ] **AC-20** (Definition of Done): A geologist (and ideally a hydrologist + climatologist + linguist + worldbuilder) reviewer, shown three sample planets generated from distinct Layer 1 + Layer 2 inputs, identifies all three as plausible without prompting. Recorded as a sign-off artifact, not a CI test.

## Architecture

The application is a single-page browser app with no backend, targeting workstation-class hardware. The full design specification lives at `design.md` (root of repo); this design doc supersedes it on the points where they disagree (mesh defaults, climate model, regional refinement, civilization scope, hardware target, export scope).

### Layered structure

Top to bottom, per `design.md` §3:

| Layer | Implementation | Role |
|---|---|---|
| UI | React + TypeScript | Sliders, brush UI, exports, civilization browser |
| 3D rendering | three.js via `@react-three/fiber` | Globe, regional view, gizmos |
| State | Zustand | Layer 1 params, Layer 2 constraints, undo, view state, civilization browser state |
| Sim orchestration | Web Worker (TS) | Owns the WebGPU device; sequences pipeline stages |
| Compute | WGSL compute shaders via WebGPU | Per-cell parallel kernels for all hot loops |
| Persistence | IndexedDB via `idb` | Saved planets, snapshots, presets, civilization state |

The simulation runs entirely in the worker. The main thread holds no `GPUBuffer` references — only the latest snapshot read for rendering. Comlink-exposed methods on the worker expose the contract documented in `design.md` §7.4.

### Planned source tree

The implementation extends the structure in `design.md` §11 with new directories for the expanded scope:

- **`src/sim/worker.ts`** — Web Worker entry, owns the `GPUDevice` and all buffers.
- **`src/sim/pipeline.ts`** — orchestrates stages and caches per-stage outputs for partial reruns.
- **`src/sim/mesh/icosahedron.ts`** — subdivision generator (levels 6–10); emits positions, neighbors, areas, latitudes as static GPU buffers. Includes a sphere-clipping helper for regional refinement (REQ-16).
- **`src/sim/mesh/voronoi.ts`** — spherical Voronoi tessellation (uses `d3-geo-voronoi`) for plate seeding and cell-area computation.
- **`src/sim/tectonics/`** — `plates.ts` (init, Euler poles), `topology.ts` (CPU split/merge events per REQ-15), `kernels.ts` (loads `tectonic_step.wgsl`).
- **`src/sim/erosion/`**, **`src/sim/hydrology/`** — fluvial / hillslope erosion, flow direction, pit fill, **`bifurcation.ts`** (natural bifurcation detection per REQ-19).
- **`src/sim/climate/`** — full GCM module (REQ-18): `gcm.ts` (atmospheric integration), `seasonal.ts` (4-season forcing), `coriolis.ts`, `itcz.ts`, `orographic.ts`, `ocean_currents.ts`. Loads `gcm_step.wgsl`, `precipitation.wgsl`, `temperature.wgsl`.
- **`src/sim/biomes/`** — Whittaker classifier, soil fertility derived attributes.
- **`src/sim/refinement/`** — regional refinement on level-10 sphere-clipped subset; reuses erosion / hydrology / climate kernels.
- **`src/sim/civilization/`** — expanded scope per REQ-17:
  - **`settlements.ts`** (17a)
  - **`trade.ts`** (17b)
  - **`language/`** — `grammar.ts`, `phonology.ts`, `morphology.ts`, `sound-change.ts`, `naming.ts` (17c, morphological grammar engine)
  - **`cultures.ts`** (17d)
  - **`religions.ts`** (17e — pantheon generation, ritual templates, holy sites)
  - **`polities.ts`** (17f — state formation, border generation)
  - **`conflicts.ts`** (17g — procedural conflict log)
  - **`history.ts`** — unified timeline of polity events, conflicts, syncretic religious changes, cultural drift
- **`src/shaders/*.wgsl`** — `tectonic_step.wgsl`, `erosion_step.wgsl`, `flow_direction.wgsl`, `pit_fill.wgsl`, `bifurcation_detect.wgsl`, **`gcm_step.wgsl`** (atmospheric advection per layer per season), `precipitation.wgsl`, `temperature.wgsl`, **`ocean_currents.wgsl`** (gyre dynamics with continental boundary deflection), `biome_classify.wgsl`, `globe_fragment.wgsl`. Reference WGSL kernels are sketched in `design.md` §5.
- **`src/webgpu/`** — `device.ts` (init + `WebGPU required` fallback message per REQ-10; requests `maxStorageBufferBindingSize` near hardware max for level 10 support per REQ-5), `buffers.ts`, `pipelines.ts`, `dispatch.ts`. Uses `webgpu-utils` (greggman) for boilerplate cleanup without hiding the API.
- **`src/store/appStore.ts`** — Zustand store with the slices documented in `design.md` §7.3, plus `civilizationState` and `historyView` slices for the worldbuilding browser.
- **`src/constraints/{types,validate,suggest}.ts`** — typed constraints, hard validation, resolution suggestions (REQ-9).
- **`src/export/{png,svg,geojson}.ts`** — one file per in-scope export format (REQ-14). `foundry.ts` and `roll20.ts` are *not* present in v1; deferred per REQ-14.
- **`src/persistence/idb.ts`** — IndexedDB wrapper around saved planets, snapshots, presets, civilization state (REQ-12). Includes per-cell quantization helpers for level 9 snapshot compression.

### Key data structures

GPU storage buffers (`design.md` §4.2) are allocated once at sim init, mutated in place by kernels. Memory at level 8 (~655 k cells, ~150 bytes/cell): ~100 MB GPU state. Level 9: ~390 MB. Level 10 (regional, clipped subset): up to ~1.5 GB — fits within RTX 3090's 24 GB and its 2 GiB max storage buffer binding size when requested.

Static buffers (`positions`, `neighbors`, `cellArea`, `latitude`) are immutable post-mesh-init. Per-stage state buffers as documented in `design.md` §4.2 plus new buffers for the expanded scope:

- **GCM state (REQ-18):** per-cell, per-layer, per-season `temperature`, `moisture`, `windU`, `windV`. Total: 4 fields × 2 layers × 4 seasons = 32 fields per cell at level 8 (~100 MB).
- **Hydrology bifurcation (REQ-19):** `secondaryFlowDirection : array<u32>`, `bifurcationSplitRatio : array<f32>`, `isNaturalBifurcation : array<u32>`.
- **Civilization (REQ-17):** sparse arrays keyed by entity ID — `settlements[]`, `polities[]`, `languages[]`, `religions[]`, `conflicts[]`. Cell-side lookup via `cellSettlementID : array<u32>`, `cellPolityID : array<u32>`, `cellLanguageID : array<u32>`, `cellCultureID : array<u32>`.

The pentagonal-cell sentinel (`INVALID = 0xFFFFFFFF`) lives in the 6th neighbor slot of the 12 cells at original icosahedron vertices. All neighbor-iterating kernels must check for it (REQ-5, AC-5).

### Worker boundary contract

The `SimWorker` interface (`design.md` §7.4) is the only surface the main thread sees. Snapshot reads return typed arrays (`Float32Array` / `Uint32Array`) backed by `SharedArrayBuffer` for large buffers (REQ-11). At level 9+, snapshot reads chunk over multiple frames (AC-11). The worker owns the WebGPU device for its lifetime; if the device is lost (`device.lost`), the worker re-initializes and emits a recovery event.

### Error and invariant handling

- WebGPU unavailable → cold-launch detection branch (REQ-10) renders the static fallback page.
- WebGPU device limits insufficient for level 10 → fall back to level 9 with a UI notice, do not crash.
- Constraint validation surfaces errors interactively, never silently (REQ-9).
- Invariant violations during simulation are treated as bugs: AC-4 fails the build. Numerical-tolerance windows (`< 1e-3` for divergence-convergence balance) are documented; tightening them is a perf optimization, not a correctness change.
- Plate-split timestep rejection: if the divergence-convergence balance check exceeds tolerance during a tectonic step, the timestep is rejected and retried with smaller `dt` (per `design.md` §5 Stage 1 invariants).
- GCM convergence failure: if a season-step fails to converge within 100 sub-steps, log a warning and use the partially-converged state; do not abort the run.

### Tech stack pin (April 2026)

Per `design.md` §10: Vite, TypeScript ≥ 5.4, React ≥ 18, three.js r170+, Zustand, Comlink, `idb`, `d3-geo-voronoi`, `gl-matrix`, `webgpu-utils`. Vitest + Playwright for unit + e2e. Avoid: gpu.js, tensorflow.js, Redux, Lodash. Add for civilization layer: a small typed-graph helper (custom or `graphology`) for polity / trade / language-family graphs.

### Build milestones

`design.md` §9 defines M0–M8. With the expanded scope, milestones extend:

- M0–M5 unchanged (scaffolding through core sim + Layer 1 UI + basic exports).
- M6 (Layer 2 constraints) unchanged.
- M7 (Regional refinement) revised to spherical mesh per REQ-16 (resolves Q7).
- M8 (Civilization base layer) — settlements, trade, names with morphological grammar (REQ-17a–c).
- **M9 (Climate GCM)** — replaces the latitudinal-only Stage 4 with full GCM per REQ-18. Can land before M8 if compute budget allows, since civilization depends on climate.
- **M10 (Cultures + religions + polities + conflicts)** — REQ-17d–g.
- **M11 (Worldbuilding browser UI)** — surfaces the civilization timeline, polity browser, religion/culture inspectors.
- **M12 (Performance pass for level 9)** — optimization to bring ultra-quality cold sim into the AC-3 budget.

VTT exports (Foundry, Roll20) were originally M8 but are now deferred to a post-v1 milestone (no number assigned).

## Open Questions — Resolutions

All eight open questions from the original draft are resolved per user direction. Recorded here as decisions, not as open items.

### Q1: Default mesh level → **Level 8 default global / Level 10 regional refinement.**
With RTX 3090 / 24 GB VRAM as target, level 8 (655 k cells, ~31 km/cell) is feasible globally and gives cell sizes that resolve real-world plate-boundary thicknesses and major mountain ranges. Level 9 (~16 km/cell) is available as an "ultra" toggle. Level 10 (~8 km/cell) is the regional refinement target (clipped subset only, never global). REQ-3, REQ-5, AC-3, AC-5 updated accordingly.

### Q2: Plate split/merge defaults → **Enabled by default, tuned to 3–5 events per 500 Myr default sim.**
"Freeze plate topology" toggle remains for users who want determinism. Threshold tuning happens during M1 against the target frequency. REQ-15 / AC-15 specify the target.

### Q3: Climate model → **Full simplified GCM with 4-season cycle, monsoons, ITCZ migration.**
2-layer atmospheric model with seasonal solar forcing, Coriolis from rotation rate parameter, surface heating differential between continents and oceans (drives monsoons). 50–100 sub-steps per season, averaged for mean-annual climate output. REQ-18 + AC-18 specify the realism markers (ITCZ migration, monsoons, subtropical highs, mid-latitude westerlies).

### Q4: Naming → **Proper morphological grammar with per-family sound-change history.**
Phoneme inventory + syllable templates + morphological rules (root + affix) + documented sound-change rules (palatalization, assimilation, vowel shifts) per language family. Per-region drift produces visibly diverged daughter forms when languages fission. REQ-17c + AC-17c specify the realism markers.

### Q5: Foundry VTT export format → **Deferred to post-v1.**
Per user direction, Foundry VTT and Roll20 exports are out of v1 scope. PNG / SVG / GeoJSON / TopoJSON ship in v1. VTT exports are a post-v1 milestone (no number assigned) — they will be revisited if user demand justifies.

### Q6: Natural river bifurcations → **Modeled as physics, not just user-marked.**
Hydrology stage detects low-gradient cells where flow can plausibly route to multiple downstream basins (Casiquiare-style). Flagged `is_natural_bifurcation`; flow accumulation splits proportionally between outlets. REQ-19 + AC-19 specify detection criteria. AC-4 invariant test treats both natural and explicit bifurcations correctly.

### Q7: Regional refinement projection → **Spherical mesh.**
Level 10 sphere-clipped subset of the global icosahedron, sharing a single mesh-type abstraction with global so erosion / hydrology / climate kernels run unchanged. No flat projection — D&D-scale regions remain on the sphere. REQ-16 + AC-16 specify the contract.

### Q8: Civilization layer scope → **Full worldbuilding tool — cultures, religions, polities, conflicts all in scope.**
The end goal is a worldbuilding tool, not a geography tool. Settlements + trade + names form the M8 base; cultures + religions + polities + conflicts form M10. REQ-17 covers all seven sub-stages (17a–g), AC-17 specifies measurable realism markers for each.

## Out of Scope

Explicitly excluded from v1 even after the scope expansion. Each is either a deferral (reconsider after v1 ships) or a permanent exclusion.

- **WebGL2 fallback** for browsers without WebGPU. Reconsider in 12 months once coverage data is in. (`design.md` §3.)
- **CPU prototype phase** or any code path that runs the simulation on CPU and is later "ported" to GPU. Data structures are wrong on CPU; from-scratch GPU implementation is cheaper than porting. (`design.md` §2 — REQ-1 implication.)
- **Foundry VTT export.** Deferred to post-v1 per user direction. Architecture leaves room for it (scene-format adapter pattern); not implemented in v1. (Q5.)
- **Roll20 export.** Same deferral as Foundry.
- **Sea level history / paleogeography.** No replay of tectonic history at varying sea levels in v1. The simulation runs to a present-day snapshot and stops; historical-replay UI is post-v1.
- **Multi-user collaborative editing.** Single-user, single-tab.
- **Mobile / touch UI.** Desktop browser only; the input model assumes mouse + keyboard.
- **Live editing of a saved planet's tectonic history.** Edits to Layer 1 / Layer 2 trigger a re-sim from the appropriate stage; no in-place mutation of historical state.
- **Custom mesh types other than subdivided icosahedron** (cube-sphere, hex-on-sphere, adaptive Voronoi). Icosahedron only. (`design.md` §4.1.)
- **Browser support outside WebGPU-capable Chromium / Firefox Nightly / Safari Tech Preview.** No Internet Explorer, no legacy-mode shims. (REQ-10.)
- **Real-time tectonic-history scrub / animation playback.** Snapshots at intermediate Myr are not retained for v1; playback is a post-v1 feature on the timeline UI.
- **Cloud sync / shared planet library / community uploads.** All persistence is local IndexedDB.

The scope expansion in REQ-17 (cultures / religions / polities / conflicts) consumes what `design.md` originally listed as "Post-ship roadmap." That roadmap is now part of v1, M9–M11.
