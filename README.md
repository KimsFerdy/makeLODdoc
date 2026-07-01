# MakeLOD — Plugin Documentation
**Version 1.3 · Unreal Engine 5.3 – 5.8 · Editor Only**
*by kims ferdy · 2026*

---

## Table of Contents

1. [What is MakeLOD?](#1-what-is-makelod)
2. [Installation](#2-installation)
3. [Opening the Panel](#3-opening-the-panel)
4. [Interface Overview](#4-interface-overview)
5. [LOD Methods](#5-lod-methods)
6. [LOD Presets](#6-lod-presets)
7. [LOD Levels Table](#7-lod-levels-table)
8. [Options](#8-options)
9. [Mesh List](#9-mesh-list)
10. [Generating LODs](#10-generating-lods)
11. [Right-Click Shortcuts](#11-right-click-shortcuts)
12. [Technical Notes](#12-technical-notes)
13. [FAQ](#13-faq)
14. [Changelog](#14-changelog)

---

## 1. What is MakeLOD?

MakeLOD is a batch LOD generator for Static Meshes. It replaces Unreal Engine's built-in per-mesh LOD workflow with a single panel that can process an entire project's worth of meshes in one click, with fine-grained control over how each LOD level is built.

**Key features:**

- Four reduction methods — Auto Redux, Seamless Redux, KDux, Modeling Tools (GeometryScript)
- Six one-click LOD presets that scale mathematically to any number of levels
- Triangle budget freedom — go as low as 0.1% of LOD0 (down to 4 triangles minimum)
- Batch processing with async queue — the editor stays responsive during generation
- Right-click shortcut on any Static Mesh asset or scene actor
- Border-locked reduction (SeamlessRedux) for tiling/modular meshes with zero seams
- KDux — a custom in-house QEM simplifier with hard-angle, UV-seam, and material-boundary protection
- Modeling Tools integration — routes through Epic's GeometryScript simplifier for multi-threaded, engine-maintained reduction

---

## 2. Installation

1. Copy the `MakeLOD` folder into your project's `Plugins/` directory or download it from [FAB](https://www.fab.com/listings/82c66f87-985e-4d05-96b2-799754196368).
   ```
   MyProject/
   └── Plugins/
       └── MakeLOD/
           ├── MakeLOD.uplugin
           └── Source/
   ```
2. Right-click your `.uproject` file → **Generate Visual Studio project files**.
3. Open the project — UE will prompt to compile the plugin. Click **Yes**.
4. After compilation, go to **Edit → Plugins**, search for `MakeLOD`, and make sure it is enabled.

> **Requirements:** Unreal Engine 5.3 or later. The plugin is editor-only and adds zero runtime overhead to your shipped game. The **Modeling Tools** method additionally requires the `GeometryScripting` plugin to be enabled (the `MakeLOD.uplugin` declares this dependency automatically — no manual step needed).

---

## 3. Opening the Panel

**From the menu bar:**
`Window → MakeLOD Panel`

The panel docks like any other UE tab — drag it next to your Content Browser, Details panel, or anywhere else.

---

## 4. Interface Overview

```
┌─ LOD Method ────────────────────────────────────────────────────┐
│  [Auto Redux]  [Seamless Redux]                                  │
│  [KDux]                                                          │
│  [Modeling Tools (GeometryScript)]                               │
├─ KDux Settings (visible when KDux selected) ────────────────────┤
│  LOD1: Hard Angle [30°]  UV Seams [x]  Mat Bounds [x]           │
│        Aggressiveness [0.5]                                      │
│  LOD2: ...                                                       │
├─ Modeling Tools Settings (visible when Modeling Tools selected) ─┤
│  LOD1: Algorithm [Attribute Aware ▾]                             │
│        [x] Allow Seam Collapse  [x] Preserve Vertex Positions    │
│        [x] Allow Seam Smoothing [x] Allow Seam Splits            │
│  LOD2: ...                                                       │
├─ LOD Presets ───────────────────────────────────────────────────┤
│  [2x]  [Grad]  [D2]                                              │
│  [D3]  [Perf]  [Qual]                                            │
│  [↓ Fill Levels]                                                 │
├─ LOD Levels ────────────────────────────────────────────────────┤
│  #   ScreenSize  Tri%    MaxDev   Est.Tris                       │
│  1   [0.50]      [0.500] [0]      ~18 000                        │
│  2   [0.20]      [0.250] [0]      ~9 000                         │
│  3   [0.08]      [0.125] [0]      ~4 500                         │
│  4   [0.02]      [0.001] [0]      ~36                            │
│  [+ Add LOD Level]               [- Remove Last]                 │
├─ Options ───────────────────────────────────────────────────────┤
│  [x] Preserve UVs            [x] Preserve Materials              │
│  [x] Preserve Vertex Colors  [x] Lock Border Verts               │
│  [ ] Recompute Normals       [x] Recompute Tangents              │
├─ Selected Meshes ───────────────────────────────────────────────┤
│  [↺ Refresh from Selection]                                      │
│  SM_Wall    LOD0: 4 800 tris  |  Current LODs: 1                 │
│  SM_Floor   LOD0: 2 048 tris  |  Current LODs: 3                 │
├─────────────────────────────────────────────────────────────────┤
│  [▶ Generate LODs for Selected Meshes]                           │
│  [▶▶ Generate LODs for ALL Meshes in Project]                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. LOD Methods

Select a method before generating. The method determines *how* each LOD's geometry is reduced.

### Auto Redux
Standard quadric-error-metric reduction using UE's built-in reducer. Collapses vertices based on geometric importance. Fast and reliable for most meshes. May show minor seams at mesh boundaries when meshes tile.

**Best for:** Props, furniture, vehicles, characters — anything that doesn't tile.

### Seamless Redux
Same quadric reduction as Auto Redux, but border edge loops are split into a protected polygon group before reduction. UE's reducer never collapses across polygon group boundaries, so the open edges of the mesh are always preserved at full resolution.

**Best for:** Modular kits, tiling walls, floors, terrain chunks — any mesh that shares edges with a neighbour.

> Enable **Lock Border Vertices** in Options (on by default) for full protection.

### KDux
An in-house custom QEM simplifier (`FKDuxSimplify.h`). Uses quadric error metrics with explicit protection layers for hard edges, UV seams, material boundaries, and high-curvature areas. Fully self-contained — no engine plugin dependency.

Each LOD level has its own KDux settings exposed in the **KDux Settings** panel section:

| Setting | Description |
|---|---|
| **Hard Angle Threshold** | Edges whose adjacent faces form an angle above this value are classified as hard edges and protected from collapse with a high-penalty quadric. |
| **Protect UV Seams** | Vertices that straddle a UV seam receive a large penalty quadric, strongly discouraging collapse across the seam. |
| **Protect Material Boundaries** | Same protection applied at polygon group / material slot boundaries. |
| **Aggressiveness Balance** | 0.0 = prioritise curvature preservation. 1.0 = pure geometric error, more aggressive reduction. |

**Best for:** Props and hero assets where you want explicit per-LOD control over how features are preserved.

### Modeling Tools (GeometryScript)
Routes simplification through Epic's **GeometryScripting** plugin (`UGeometryScriptLibrary_MeshSimplifyFunctions::ApplySimplifyToTriangleCount`). Converts the mesh to `UDynamicMesh`, runs GeometryScript's simplifier, then converts back. Multi-threaded on large meshes, engine-maintained, and consistently produces high-quality results.

Each LOD level has its own Modeling Tools settings:

| Setting | Default | Description |
|---|---|---|
| **Algorithm** | Attribute Aware | **Standard QEM** — fastest. **Volume Preserving** — adds a volume correction pass, slower. **Attribute Aware** — also minimises UV and normal distortion; best quality, UE's own default. |
| **Allow Seam Collapse** | On | Allow edges that straddle a UV or material seam to be collapsed. Disable for stricter seam protection (equivalent intent to KDux's UV Seam protection). |
| **Allow Seam Smoothing** | On | Allow seam vertices to be repositioned toward a better fit during collapse. |
| **Allow Seam Splits** | On | Allow seam edges to be split during simplification. Disable for the strictest seam protection. |
| **Preserve Vertex Positions** | Off | Skip quadric-optimal vertex repositioning — collapsed vertices snap to an existing position. Higher shape fidelity; needs more triangles to reach equivalent quality. |

> **Editor-only dependency:** Modeling Tools requires the `GeometryScripting` UE plugin. This is declared automatically in `MakeLOD.uplugin` — no manual setup needed.

**Best for:** Any mesh where you want maximum quality with minimal setup. Pairs especially well with large batches since it's multi-threaded.

---

## 6. LOD Presets

Presets fill your LOD level rows with mathematically calculated values. They **never change the number of rows** — they only rewrite the values inside existing rows. This means:

- Set up 6 LOD rows → click a preset → all 6 rows get the preset formula applied
- Add or remove rows → click **Fill Levels** → values update to fit the new row count

### The 6 Presets

| Button | Formula | Best For |
|---|---|---|
| **2x** | Exponential halving. Each level = previous × 0.5 | General purpose, all mesh types |
| **Grad** | Even gradient. Divides the full range into N+1 equal slots | Perfectly predictable, evenly spaced LOD pop |
| **D2** | Linear decrement, Tri% steps 0.80 → 0.10 | Gentle reduction, quality-focused |
| **D3** | Steeper linear decrement, Tri% steps 0.70 → 0.001 | Aggressive reduction, performance-focused |
| **Perf** | LOD transitions packed tight at 0.90–0.12 screen size | Open-world, large scenes, distant objects |
| **Qual** | LOD0 stays visible to screen 0.60, gentle Tri% drop 0.85 → 0.20 | Hero props, cinematics, close-up assets |

### Fill Levels Button

After selecting a preset, **Fill Levels** re-applies the active preset formula to however many rows currently exist. Recommended workflow:

1. Add/remove LOD rows to get the count you want
2. Click a preset button
3. Click **↓ Fill Levels** — all rows recalculate using the formula spread across the current level count
4. Tweak individual rows if needed

---

## 7. LOD Levels Table

Each row represents one generated LOD (LOD1, LOD2, ...). LOD0 is always the original, untouched mesh.

| Column | Description |
|---|---|
| **#** | LOD index (LOD1, LOD2, ...) |
| **Screen Size** | UE screen-coverage threshold. When the mesh covers less than this fraction of the screen, UE switches to this LOD. Range 0.0–1.0. |
| **Tri%** | Triangle budget as a fraction of LOD0. 0.5 = 50% of original. **Minimum is 0.001** (0.1%) — a 100 000 tri mesh at 0.001 produces ~100 tris. The hard floor is 4 triangles regardless of percentage. |
| **MaxDev** | Maximum allowed surface deviation in world units. 0 = preserve silhouette exactly. Used by Auto Redux and Seamless Redux. |
| **Est. Tris** | Live estimate of triangle count at this level based on the first selected mesh. |

### Adding / Removing Levels
- **+ Add LOD Level** — adds one row. Tri% is automatically set to half the last row's value, down to the 0.001 floor. Maximum 7 levels (UE hard limit: LOD0–LOD7).
- **- Remove Last** — removes the last row. Minimum 1 level.

---

## 8. Options

| Option | Default | Description |
|---|---|---|
| **Preserve UVs** | On | Keep all UV channels from LOD0 on every generated LOD. |
| **Preserve Materials** | On | Keep material slot assignments and polygon group layout. |
| **Preserve Vertex Colors** | On | Keep vertex color data on every generated LOD. |
| **Lock Border Vertices** | On | SeamlessRedux only. Pins open edge loops so they are never collapsed. |
| **Recompute Normals** | Off | Recompute normals after reduction. Leave off to keep baked normals from LOD0. |
| **Recompute Tangents** | On | Recompute tangents after reduction. Recommended on. |

---

## 9. Mesh List

The mesh list shows which static meshes will be processed when you click Generate.

**Populating the list:**
- Select Static Mesh assets in the **Content Browser**
- Or select **StaticMeshActors** in the **Level Viewport** — the plugin resolves to their source mesh
- Click **↺ Refresh from Selection** to update the list

Each row shows the mesh name, LOD0 triangle count, and how many LOD levels it currently has.

---

## 10. Generating LODs

### Generate LODs for Selected Meshes
Processes only the meshes in the current Mesh List. Fast and targeted — good for iteration.

### Generate LODs for ALL Meshes in Project
Finds every Static Mesh asset in the entire project via the Asset Registry and queues them all. The editor stays responsive — meshes are processed one per tick via a `FTSTicker` queue.

> Large projects may take several minutes. Watch the Output Log for progress:
> `[MakeLOD] Processing mesh 12 / 247: SM_Rock_A`

**What happens after generation:**
- LODs are written directly back into the same Static Mesh asset — no new assets created
- All actors in the scene using that mesh update automatically
- Nanite is disabled on processed meshes (Nanite and manual LODs are mutually exclusive in UE5)
- The package is marked dirty — save normally with Ctrl+S or Save All

---

## 11. Right-Click Shortcuts

Right-click shortcuts use the current panel settings, or Auto Redux defaults if the panel was never opened.

**In the Content Browser:**
Right-click any Static Mesh asset → **Generate LODs (MakeLOD)**

**In the Level Viewport:**
Select any StaticMeshActor → right-click → **Generate LODs (MakeLOD)**

---

## 12. Technical Notes

### Reduction Backends
MakeLOD supports two reduction backends:

- **UE built-in** (`IMeshReductionInterface` / Chaos quadric reducer) — used by Auto Redux and Seamless Redux. Always available, no extra dependencies.
- **GeometryScript** (`UGeometryScriptLibrary_MeshSimplifyFunctions`) — used by Modeling Tools. Requires the `GeometryScripting` UE plugin (declared automatically as a plugin dependency). Internally converts `FMeshDescription` ↔ `UDynamicMesh` for the simplification pass.
- **KDux** (`FKDuxSimplify.h`) — fully self-contained, in-house QEM. No engine or plugin dependency. `FDynamicMesh3`-free.

### Triangle Floor
All three backends enforce a minimum of 4 triangles regardless of `PercentTriangles`. This is the lowest count that can form a valid closed surface. The `PercentTriangles` field accepts values down to `0.001` (0.1%).

### Async Processing
`GenerateLODsAsync()` uses `FTSTicker` to schedule one mesh per game-thread tick. `UStaticMesh::Build()` must run on the game thread — this approach keeps the editor interactive during large batches.

### SeamlessRedux — How Border Locking Works
Before reduction, polygons that touch any open-edge (naked-edge) vertex are reclassified into a dedicated `MakeLOD_Border` polygon group. UE's reducer never collapses vertices across polygon group boundaries, which freezes the border loops at full resolution while the interior is freely reduced.

### KDux — Double-Reduction Prevention
KDux and Modeling Tools deliver a fully pre-simplified `FMeshDescription` to `AppendLOD`. The engine's built-in reducer is bypassed for these methods by setting `PercentTriangles = 1.0` and `MaxNumOfTriangles = MAX_int32` on the source model's `ReductionSettings` — otherwise the engine would apply a second reduction pass on top of the custom one.

### Version Guards
The plugin includes compile-time version guards covering:
- `SetAutoComputeLODScreenSize` (UE 5.5+)
- `SetNaniteSettings` (UE 5.6+)

---

## 13. FAQ

**Q: Will this delete my existing LODs?**
A: Yes. MakeLOD strips all existing LODs above LOD0 before regenerating. LOD0 (the original mesh) is always preserved.

**Q: Can I use this on Nanite meshes?**
A: MakeLOD disables Nanite on meshes it processes — Nanite and manual LODs are mutually exclusive in UE5. Do not run MakeLOD on assets you intend to keep as Nanite.

**Q: How low can Tri% go?**
A: Down to 0.001 (0.1%). The absolute floor is 4 triangles regardless of percentage — you can't have a valid mesh with fewer. On a 100 000 tri mesh at 0.001 you get roughly 100 tris; on a 500 tri mesh at 0.001 you hit the 4-tri floor.

**Q: My tiling meshes have seams between LOD levels. What do I do?**
A: Switch to **Seamless Redux** with **Lock Border Vertices** enabled. This guarantees the outer edge loops of every mesh are identical across all LOD levels.

**Q: Which method gives the best quality?**
A: It depends on the mesh. **Modeling Tools** with Attribute Aware is a great default — it's multi-threaded and minimises UV and normal distortion. **KDux** gives you finer per-level control, especially useful for hero props where you want to dial in exactly how features are preserved. Try both and compare — the Est. Tris counter updates live.

**Q: What's the difference between KDux and Modeling Tools?**
A: Both are QEM-based but they're independent implementations. KDux is ours — fully self-contained, no plugin dependency, with explicit controls for hard-angle threshold and aggressiveness per LOD level. Modeling Tools routes through Epic's GeometryScript, is multi-threaded on large meshes, and gets maintenance from Epic with each engine update. Neither is strictly better — they complement each other.

**Q: "Generate LODs for ALL" is taking very long. Can I cancel?**
A: Not mid-batch in the current version. Restart the editor to cancel. A cancel button is planned for a future update.

**Q: Does this work in Blueprints or at runtime?**
A: No. MakeLOD is editor-only with zero runtime footprint. All code is inside `#if WITH_EDITOR` guards.

**Q: Can I run this from a commandlet / CI pipeline?**
A: `FLODGeneratorTool::GenerateLODs()` (synchronous) can be called from a commandlet. Wire it up in your own commandlet class and invoke it with `UnrealEditor-Cmd.exe MyProject.uproject -run=YourCommandlet`.

---

## 14. Changelog

### v1.3
- Triangle floor lowered from 5% to **0.1% (0.001)** across all methods and presets — the last LOD can now go as low as 4 triangles
- `PercentTriangles` spinbox updated: minimum `0.001`, delta `0.001`, 3 decimal places displayed
- All preset curve floors updated to match — no more invisible 5% clamp on deep LOD levels
- `OnAddLODLevel` auto-step floor updated to `0.001`
- Version bump to 1.3

### v1.2
- **Modeling Tools method added** — routes through Epic's GeometryScript (`ApplySimplifyToTriangleCount`). Requires `GeometryScripting` UE plugin (declared automatically as a dependency in `.uplugin`)
- **KDux method** — custom in-house QEM simplifier with per-level hard-angle, UV-seam, material-boundary, and aggressiveness controls
- **Custom % method removed** — redundant with Auto Redux
- `MakeLOD.uplugin` updated: `GeometryScripting` plugin dependency added, `MeshConversion` module added to `Build.cs`
- Per-level Modeling Tools settings panel: algorithm selector (Standard QEM / Volume Preserving / Attribute Aware), seam collapse, seam smoothing, seam splits, preserve vertex positions
- UI layout updated: method buttons now full-width for KDux and Modeling Tools
- Double-reduction guard added in `AppendLOD` — KDux and Modeling Tools pre-simplified meshes now correctly bypass the engine's built-in reducer

### v1.1
- Initial public release
- Auto Redux, Seamless Redux, KDux, Custom % methods
- Six presets, async batch queue, content browser and actor right-click shortcuts

---

*MakeLOD v1.3 — © kims ferdy 2026 — All Rights Reserved*
