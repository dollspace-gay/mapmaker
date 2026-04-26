---
title: "Realistic Planet Generator"
tags: ["design-doc"]
sources: []
contributors: ["iWst"]
created: 2026-04-26
updated: 2026-04-26
---


## Design Specification

### Summary

A browser-based single-page application that generates realistic D&D-usable worlds by simulating planetary geology — tectonics, erosion, hydrology, climate, biomes, settlements — with WebGPU compute shaders, then producing both an interactive 3D globe and exportable 2D maps. Existing fantasy map generators draw maps from aesthetic priors; this tool simulates the geological causality directly so the pretty stuff falls out of correctly modeled physics. Source spec: `design.md` (v1, April 2026).

### Requirements

- **REQ-1 (GPU-native simulation).** All per-cell simulation state lives in `GPUBuffer` objects; the inner simulation loop performs no readback to JavaScript. CPU code only orchestrates dispatches, handles graph-topological events (plate split/merge), validates constraints, and marshals exports. (`design.md` §2, §4.2.)
- **REQ-2 (Geological causality).** Every visible feature on the final map — mountains, deserts, river networks, biomes — is downstream of simulated physics rather than aesthetic placement. (`design.md` §2.)
- **REQ-3 (Speed budget).** On a mid-range laptop GPU (Apple M1 / NVIDIA GTX 1660 baseline) at default mesh level 6 (~41 k cells): cold full simulation under 60 s; warm reroll (same parameters, new seed) under 2 s; parameter tweak that does not change topology under 5 s. (`design.md` §2, §8.)
- **REQ-4 (Hard physical invariants).** The simulation enforces, every run, all of: rivers merge going downstream (delta and explicit-bifurcation are the only exceptions); total plate divergence rate equals total convergence rate within numerical tolerance; drainage basins partition the surface (every land cell drains to exactly one outlet); crust age monotone non-decreasing from spreading center to subduction zone; hotspot tracks aligned with plate motion direction. Violation of any is treated as a bug, not a tradeoff. (`design.md` §2.)
- **REQ-5 (Mesh).** Subdivided icosahedron with selectable level: 5 (~10 k cells, fast preview), 6 (~41 k, default), 7 (~164 k, "high quality"), 8 (~655 k, regional-refinement only — never global). Twelve pentagonal cells per level handled correctly via fixed-size neighbor table with `INVALID = 0xFFFFFFFF` sentinel. (`design.md` §4.1.)
- **REQ-6 (Pipeline order).** Stages execute sequentially with documented per-stage GPU-buffer outputs: Tectonics → Erosion (coupled with Hydrology iterations) → Hydrology → Climate → Biomes → Regional Refinement → Settlements. Each stage emits an inspectable snapshot the next consumes. Partial reruns skip stages whose inputs are unchanged. (`design.md` §5.)
- **REQ-7 (Layer 1 parameters).** All twenty-something global scalar parameters across Astronomical, Tectonic, Hydrosphere, and Biosphere groups are user-adjustable via sliders / numeric inputs / toggles. Seven named presets ship: Earth-like, Mars-like, Ocean world, Young volcanic, Old worn world, Pangaea, Archipelago. (`design.md` §6.1.)
- **REQ-8 (Layer 2 constraints).** Six paint-brush constraint types are supported: plate, boundary (convergent/divergent/transform), feature pin (mountain range, volcanic arc, rift, hotspot, ancient orogen), climate pin, coastline pin, settlement pin (Layer 2.5). Click-drag paints, right-click erases, esc cancels; curve constraints place control points and double-click to finish. (`design.md` §6.2, §7.5.)
- **REQ-9 (Constraint validation).** Three layers: (a) hard validation at input time, surfaced as red/yellow overlays plus a panel listing each issue with location and ≥1 suggested fix; (b) soft constraints folded into an equilibration objective during simulation; (c) actionable resolution suggestions when validation fails (e.g. "adjust Euler pole to X" / "shorten hotspot activation history"). Validation runs continuously during paint, debounced to ~200 ms. (`design.md` §6.3, §7.5.)
- **REQ-10 (WebGPU only in v1).** No WebGL2 fallback ships in v1. When `navigator.gpu` is unavailable or `requestAdapter()` fails, display a clear "WebGPU required" message naming supported browsers (Chrome / Edge ≥ 113, Firefox Nightly, Safari Tech Preview). Reconsider WebGL2 fallback after 12 months of coverage data. (`design.md` §3.)
- **REQ-11 (Worker-isolated simulation).** The simulation pipeline runs in a single Web Worker that exclusively owns the WebGPU device and all GPU buffers. The main thread holds no `GPUBuffer` references. Communication uses Comlink for control RPC; state snapshots transfer to the main thread via `SharedArrayBuffer` for buffers large enough to matter (heightmap, plate map, biome map). (`design.md` §3, §7.4.)
- **REQ-12 (Persistence).** Saved planets, simulation snapshots, and user presets persist in IndexedDB via the `idb` library. Save → reload → load roundtrips Layer 1 + Layer 2 + simulation snapshot identically. (`design.md` §3, §10.)
- **REQ-13 (Render performance).** Globe rendering uses a single instanced draw call with cell colors sourced from the active layer buffer; target 60 fps regardless of simulation state, even at level 7 (~164 k cells). Sim progress updates flow to UI without blocking input. (`design.md` §7.6.)
- **REQ-14 (Exports).** Export pipeline produces valid, openable files for: PNG (equirectangular projection), SVG, GeoJSON / TopoJSON, Foundry VTT scene, Roll20 map. (`design.md` §10, §14.)
- **REQ-15 (Plate topology events on CPU).** Plate split, plate merge, and rift propagation are graph-topological events that run CPU-side. They are infrequent (at most every few simulation Myr) so the brief readback / mutate / upload is acceptable. A "freeze plate topology" toggle disables splits/merges for users who want stable plate counts across rerolls (resolves Q2). (`design.md` §4.3.)
- **REQ-16 (Regional refinement preserves global drainage).** Stage 6 builds a high-resolution local mesh (~1–5 km/cell) initialized from global state, with high-frequency detail added via constraint-respecting noise. The global flow network is a hard constraint on the local resampling — rivers that exist at global resolution must persist at regional resolution and discharge to the same outlets. (`design.md` §5 Stage 6.)
- **REQ-17 (Settlements & civilization).** Stage 7 places settlements via weighted Poisson-disk sampling on a score field (river/confluence, sheltered harbor, defensibility, arable proximity, trade chokepoint, climate harshness). Trade routes via Dijkstra on the cell graph with terrain-derived edge weights. Names generated with per-region linguistic drift (4–8 language families, Markov phoneme chains for v1). (`design.md` §5 Stage 7.)

### Acceptance Criteria

- [ ] **AC-1** (REQ-1): The inner simulation loop calls no buffer readback / `mapAsync` and the main thread holds no per-cell typed arrays except explicit snapshot reads at stage boundaries — verified by code review and a worker-side telemetry counter that asserts zero readbacks during a hot run.
- [ ] **AC-2** (REQ-2): Causality trace test — for any cell on the final map, an automated query can identify the upstream physical cause: mountain cells trace to a convergent boundary; desert cells trace to a windward elevation barrier or a 30°-band subtropical-high; biome cells are a deterministic function of (T, P) lookup.
- [ ] **AC-3** (REQ-3): Benchmark suite (10 runs each, p99) on target hardware reports: cold full sim ≤ 60 s; warm reroll ≤ 2 s; non-topological param tweak ≤ 5 s. Numbers logged to a perf-budget CI artifact.
- [ ] **AC-4** (REQ-4): Invariant test suite, run at the end of every full sim, must pass all of:
- [ ] River bifurcation count equals zero across all land cells, excluding those flagged `is_delta` or `is_explicit_bifurcation`.
- [ ] `|Σ divergence − Σ convergence| / max(Σ divergence, Σ convergence) < 1e-3`.
- [ ] Every land cell has exactly one downstream successor; basin partition coverage = 100 % of land cells.
- [ ] Crust age strictly non-decreasing along all flow lines from spreading center to subduction zone.
- [ ] Hotspot-track-vector vs plate-motion-vector angle within 15° at every track cell.
- [ ] **AC-5** (REQ-5): Mesh generator produces 10 242 / 40 962 / 163 842 / 655 362 cells at levels 5/6/7/8. Per-cell neighbor count test confirms exactly 12 pentagonal cells at every level. Sentinel `INVALID` slot handling verified by traversal test.
- [ ] **AC-6** (REQ-6): Each pipeline stage emits a snapshot consumable by the next via documented buffer contracts (`tectonic_state`, `erosion_state`, `hydrology_state`, `climate_state`, `biome_state`). Partial-rerun cache hits are observable in worker logs and skip dispatching unchanged stages.
- [ ] **AC-7** (REQ-7): Every Layer 1 parameter listed in `design.md` §6.1 surfaces in the UI with a control of appropriate type. All seven presets selectable; each produces a planet visibly distinct from defaults (verified by a perceptual hash diff threshold).
- [ ] **AC-8** (REQ-8): All six brush types paint constraint data that persists across page reloads; brush radius slider, click-drag, right-click erase, and esc-cancel are all functional in an end-to-end test.
- [ ] **AC-9** (REQ-9): A test fixture of intentionally inconsistent constraints (hotspot track misaligned with Euler pole; coastline crossing transform boundary; climate pin inconsistent with computed T/P) produces, for each, a red overlay, a panel entry with location, and ≥1 suggested fix. Soft-constraint relaxation reduces objective monotonically across iterations.
- [ ] **AC-10** (REQ-10): When `navigator.gpu` is undefined or `requestAdapter()` returns null, the app displays a "WebGPU required" message naming supported browsers; no white screen, no thrown error reaches the user.
- [ ] **AC-11** (REQ-11): Worker init creates the `GPUDevice` only inside the worker; main-thread heap inspection finds no `GPUBuffer` references. Comlink-exposed `SimWorker` interface matches `design.md` §7.4 (`init`, `setParameters`, `setConstraints`, `runFullSim`, `rerollFromSeed`, `getSnapshot`, `refineRegion`, `exportData`). Snapshots > 1 MB transfer via `SharedArrayBuffer` (verified by transferable-list test).
- [ ] **AC-12** (REQ-12): Save planet → reload page → load planet roundtrip is bit-identical for elevation, plates, biomes, constraints, and Layer 1 parameters. Preset save/load roundtrips Layer 1 + Layer 2.
- [ ] **AC-13** (REQ-13): Render frame time ≤ 16.7 ms (60 fps p95) during a running sim at level 6 on target hardware. Sim progress events arrive at ≥ 1 Hz; UI button presses register within 100 ms during sim.
- [ ] **AC-14** (REQ-14): Each export format produces a file validated by its target tool: PNG opens in image viewers; SVG opens in browser without parse errors; GeoJSON validates against the geojson.org schema; Foundry scene imports cleanly into a current-version Foundry world; Roll20 map imports via the marketplace tool. (Foundry schema verified at implementation time per Q5.)
- [ ] **AC-15** (REQ-15): Plate split fires when sustained extensional stress exceeds the documented threshold along a connected curve through a continental plate; merge fires after ≥ 50 Myr continuous collision with no relative motion. CPU-side topology mutation completes within one frame (≤ 16.7 ms) at level 6. "Freeze plate topology" toggle suppresses both events.
- [ ] **AC-16** (REQ-16): For a planet at level 6 with `R` rivers discharging at `R` distinct ocean cells, the regional refinement of any chosen region preserves all enclosed rivers and routes them to the same coastline cells. Macro structure unchanged is verified by topological comparison; high-frequency tributaries appear only as additions.
- [ ] **AC-17** (REQ-17): Settlement placement test: capitals/cities/towns counts roughly match Poisson-disk parameter; trade routes prefer rivers and avoid mountains (verified by edge-weight aggregation along generated paths); within-region settlement names share phoneme inventory above a threshold cosine similarity, cross-region pairs below.
- [ ] **AC-18** (Definition of Done §6): A geologist (or geomorphologist) reviewer, shown three sample planets generated from distinct Layer 1 + Layer 2 inputs, identifies all three as plausible without prompting. Recorded as a sign-off artifact, not a CI test.

### Architecture

The application is a single-page browser app with no backend. The full design specification lives at `design.md` (root of repo); this section summarizes the architecture as the planned implementation will realize it.

### Layered structure

Top to bottom, per `design.md` §3:

| Layer | Implementation | Role |
|---|---|---|
| UI | React + TypeScript | Sliders, brush UI, exports |
| 3D rendering | three.js via `@react-three/fiber` | Globe, regional view, gizmos |
| State | Zustand | Layer 1 params, Layer 2 constraints, undo, view state |
| Sim orchestration | Web Worker (TS) | Owns the WebGPU device; sequences pipeline stages |
| Compute | WGSL compute shaders via WebGPU | Per-cell parallel kernels for all hot loops |
| Persistence | IndexedDB via `idb` | Saved planets, snapshots, presets |

The simulation runs entirely in the worker. The main thread holds no `GPUBuffer` references — only the latest snapshot read for rendering. Comlink-exposed methods on the worker expose the contract documented in `design.md` §7.4.

### Planned source tree

The implementation will follow the structure in `design.md` §11. Key directories that anchor the pipeline:

- **`src/sim/worker.ts`** — Web Worker entry, owns the `GPUDevice` and all buffers.
- **`src/sim/pipeline.ts`** — orchestrates stages and caches per-stage outputs for partial reruns.
- **`src/sim/mesh/icosahedron.ts`** — subdivision generator (levels 5–8); emits positions, neighbors, areas, latitudes as static GPU buffers.
- **`src/sim/mesh/voronoi.ts`** — spherical Voronoi tessellation (uses `d3-geo-voronoi`) for plate seeding and cell-area computation.
- **`src/sim/tectonics/`** — `plates.ts` (init, Euler poles), `topology.ts` (CPU split/merge events per REQ-15), `kernels.ts` (loads `tectonic_step.wgsl`).
- **`src/sim/erosion/`**, **`src/sim/hydrology/`**, **`src/sim/climate/`**, **`src/sim/biomes/`**, **`src/sim/refinement/`**, **`src/sim/settlements/`** — one directory per pipeline stage.
- **`src/shaders/*.wgsl`** — `tectonic_step.wgsl`, `erosion_step.wgsl`, `flow_direction.wgsl`, `pit_fill.wgsl`, `precipitation.wgsl`, `temperature.wgsl`, `biome_classify.wgsl`, `globe_fragment.wgsl`. Reference WGSL kernels are sketched in `design.md` §5.
- **`src/webgpu/`** — `device.ts` (init + `WebGPU required` fallback message per REQ-10), `buffers.ts`, `pipelines.ts`, `dispatch.ts`. Uses `webgpu-utils` (greggman) for boilerplate cleanup without hiding the API.
- **`src/store/appStore.ts`** — Zustand store with the slices documented in `design.md` §7.3 (params, constraints, simStatus, validationIssues, view, currentSnapshot, actions).
- **`src/constraints/{types,validate,suggest}.ts`** — typed constraints, hard validation, resolution suggestions (REQ-9).
- **`src/export/{png,svg,geojson,foundry,roll20}.ts`** — one file per export format (REQ-14).
- **`src/persistence/idb.ts`** — IndexedDB wrapper around saved planets, snapshots, presets (REQ-12).

### Key data structures

GPU storage buffers (`design.md` §4.2) are allocated once at sim init, mutated in place by kernels. At level 6 (~41 k cells), total GPU state is approximately 6 MB; at level 7, ~24 MB. Static buffers (`positions`, `neighbors`, `cellArea`, `latitude`) are immutable post-mesh-init. Tectonic state (`plateID`, `crustType`, `crustAge`, `crustThickness`, `elevation`, `upliftRate`) is mutated each tectonic step. Erosion, hydrology, climate, biome states each have their own buffer set. Plate data (length P plates, typically 8–15) is a small CPU-mirrored buffer for topology events.

The pentagonal-cell sentinel (`INVALID = 0xFFFFFFFF`) lives in the 6th neighbor slot of the 12 cells at original icosahedron vertices. All neighbor-iterating kernels must check for it (see REQ-5, AC-5).

### Worker boundary contract

The `SimWorker` interface (`design.md` §7.4) is the only surface the main thread sees. Snapshot reads return typed arrays (`Float32Array` / `Uint32Array`) backed by `SharedArrayBuffer` for large buffers (REQ-11). The worker owns the WebGPU device for its lifetime; if the device is lost (`device.lost`), the worker re-initializes and emits a recovery event.

### Error and invariant handling

- WebGPU unavailable → cold-launch detection branch (REQ-10) renders the static fallback page.
- Constraint validation surfaces errors interactively, never silently (REQ-9).
- Invariant violations during simulation are treated as bugs: AC-4 fails the build. Numerical-tolerance windows (`< 1e-3` for divergence-convergence balance) are documented; tightening them is a perf optimization, not a correctness change.
- Plate-split timestep rejection: if the divergence-convergence balance check exceeds tolerance during a tectonic step, the timestep is rejected and retried with smaller `dt` (per `design.md` §5 Stage 1 invariants).

### Tech stack pin (April 2026)

Per `design.md` §10: Vite, TypeScript ≥ 5.4, React ≥ 18, three.js r170+, Zustand, Comlink, `idb`, `d3-geo-voronoi`, `gl-matrix`, `webgpu-utils`. Vitest + Playwright for unit + e2e. Avoid: gpu.js, tensorflow.js, Redux, Lodash.

### Build milestones

`design.md` §9 defines M0–M8 as independently shippable milestones (~4–5 months calendar). Each milestone produces a working artifact; v0 ships at M5 with full Layer 1 and exports, v1 at M8 with constraints, regional refinement, and settlements. Milestones map to follow-up issues created off this design doc; see "Out of Scope" for what is explicitly *not* in v1.

### Out of Scope

- **WebGL2 fallback** for browsers without WebGPU. Reconsider in 12 months once coverage data is in. (`design.md` §3.)
- **CPU prototype phase** or any code path that runs the simulation on CPU and is later "ported" to GPU. The data structures are wrong on CPU; from-scratch GPU implementation is cheaper than porting. (`design.md` §2 — REQ-1 implication.)
- **Seasonal climate / monsoons.** v1 ships steady-state mean-annual climate only. (Q3.)
- **Sea level history / paleogeography.** No replay of tectonic history at varying sea levels.
- **Procedural cultures, political boundaries, religions, mythology, pantheon generation.** These are worldbuilding, not geography. (Q8.)
- **Multi-user collaborative editing.** Single-user, single-tab.
- **Mobile / touch UI.** Desktop browser only in v1.
- **Natural river bifurcations** beyond explicit user constraint (Casiquiare-style). v1 supports only painted bifurcations. (Q6.)
- **Proper morphological grammar for naming.** Markov phoneme chains ship in v1; the engine API is extensible for a later grammar plugin. (Q4.)
- **Spherical regional view.** Planar projection only at the regional refinement stage in v1. (Q7.)
- **Backend / cloud sync / shared planet library.** All persistence is local IndexedDB.
- **Live editing of a saved planet's tectonic history.** Edits to Layer 1 / Layer 2 trigger a re-sim from the appropriate stage; there is no in-place mutation of historical state.
- **Custom mesh types other than subdivided icosahedron** (e.g. cube-sphere, hex-on-sphere). Icosahedron only. (`design.md` §4.1.)
- **Browser support outside WebGPU-capable Chromium / Firefox Nightly / Safari Tech Preview.** No Internet Explorer, no legacy-mode shims. (REQ-10.)

