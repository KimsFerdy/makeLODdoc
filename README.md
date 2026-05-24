# MakeLOD — Plugin Documentation
**Version 1.0 · Unreal Engine 5.2 – 5.7 · Editor Only**
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

---

## 1. What is MakeLOD?

MakeLOD is a batch LOD generator for Static Meshes. It replaces Unreal Engine's built-in per-mesh LOD workflow with a single panel that can process an entire project's worth of meshes in one click, with fine-grained control over how each LOD level is built.

**Key features:**

- Four reduction methods — Auto Redux, Seamless Redux, Quad Based, Custom
- Six one-click LOD presets that scale mathematically to any number of levels
- Batch processing with async queue — the editor stays responsive during generation
- Right-click shortcut on any Static Mesh asset or scene actor
- Border-locked reduction (SeamlessRedux) for tiling/modular meshes with zero seams
- Terrain-style quad-grid LODs (QuadBased) with bilinear height sampling and exact border snapping
- No external dependencies — uses only UE's built-in mesh reduction interface

---

## 2. Installation

1. Copy the `MakeLOD` folder into your project's `Plugins/` directory or Download it from [FAB](https://www.fab.com/listings/82c66f87-985e-4d05-96b2-799754196368https://www.fab.com/portal/listings/82c66f87-985e-4d05-96b2-799754196368).
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

> **Requirements:** Unreal Engine 5.2 or later. The plugin is editor-only and adds zero runtime overhead to your shipped game.

---

## 3. Opening the Panel

**From the menu bar:**
`Window → MakeLOD Panel`

The panel docks like any other UE tab — you can drag it next to your Content Browser, Details panel, or anywhere else.

---

## 4. Interface Overview

```
┌─ LOD Method ────────────────────────────────────────────┐
│  [Auto Redux] [Seamless Redux] [Quad Based] [Custom]     │
├─ LOD Presets ───────────────────────────────────────────┤
│  [2x]  [Grad]  [D2]                                     │
│  [D3]  [Perf]  [Qual]                                   │
│  [↓ Fill Levels]                                        │
├─ LOD Levels ────────────────────────────────────────────┤
│  #   ScreenSize  Tri%   QuadMerge  MaxDev   Est.Tris    │
│  1   [0.95]      [0.75] [2]        [0]      ~18000      │
│  2   [0.60]      [0.50] [4]        [0]      ~12000      │
│  3   [0.25]      [0.25] [8]        [0]      ~6000       │
│  4   [0.05]      [0.10] [16]       [0]      ~2400       │
│  [+ Add LOD Level]           [- Remove Last]            │
├─ Options ───────────────────────────────────────────────┤
│  [x] Preserve UVs            [x] Preserve Materials     │
│  [x] Preserve Vertex Colors  [x] Lock Border Verts      │
│  [ ] Recompute Normals       [x] Recompute Tangents     │
│  [x] Follow Mesh Boundaries (Quad Based)                │
├─ Selected Meshes ───────────────────────────────────────┤
│  [↺ Refresh from Selection]                             │
│  SM_Wall    LOD0: 4800 tris  |  Current LODs: 1         │
│  SM_Floor   LOD0: 2048 tris  |  Current LODs: 3         │
├─────────────────────────────────────────────────────────┤
│  [▶ Generate LODs for Selected Meshes]                  │
│  [▶▶ Generate LODs for ALL Meshes in Project]           │
└─────────────────────────────────────────────────────────┘
```

---

## 5. LOD Methods

Select a method before generating. The method determines *how* each LOD's geometry is built.

### Auto Redux
Standard quadric-error-metric mesh reduction using UE's built-in reducer. Collapses vertices based on geometric importance. Fast and reliable for most meshes. May show minor seams at mesh boundaries when meshes tile.

**Best for:** Props, furniture, vehicles, characters, anything that doesn't tile.

### Seamless Redux
Same quadric reduction as Auto Redux, but border edge loops are split into a protected polygon group before reduction. UE's reducer never collapses across polygon group boundaries, so the open edges of the mesh are always preserved at full resolution.

**Best for:** Modular kits, tiling walls, floors, terrain chunks, any mesh that shares edges with a neighbour.

> Enable **Lock Border Vertices** in Options (on by default) for full protection.

### Quad Based
A terrain-style grid resampler. Detects the source mesh's vertex grid, builds a coarser `DstSectX × DstSectY` grid using the **QuadMerge** value, and samples heights bilinearly. Border vertices are always sampled at their exact source position — never interpolated — so adjacent chunks always share identical border vertices and seams are impossible.

**Best for:** Heightfield terrain chunks, regular grid meshes, voxel surfaces.

> Use **QuadMerge** values that are powers of 2 (2, 4, 8, 16) for perfectly regular grids.
> Enable **Follow Mesh Boundaries** in Options (on by default) to pin borders.

### Custom
You control the exact `PercentTriangles` for each LOD level manually. The UE built-in reducer applies your percentages directly with no automatic scaling.

**Best for:** Hero assets where you want precise per-LOD triangle budgets.

---

## 6. LOD Presets

Presets fill your LOD level rows with mathematically calculated values. They **never change the number of rows** — they only rewrite the values inside existing rows. This means:

- Set up 6 LOD rows → click a preset → all 6 rows get the preset formula
- Add or remove rows → click the same preset again (or **Fill Levels**) → values update

### The 6 Presets

| Button | Formula | Best For |
|---|---|---|
| **2x** | Exponential halving. Each level = previous × 0.5 | General purpose, all mesh types |
| **Grad** | Even gradient. Divides the full range into N+1 equal slots. LOD1 = N/(N+1), LOD2 = (N-1)/(N+1), ... | When you want perfectly predictable, evenly spaced LOD pop |
| **D2** | Linear decrement. Tri% steps from 0.80 → 0.10 across N levels | Gentle reduction, quality-focused |
| **D3** | Steeper linear decrement. Tri% steps from 0.70 → 0.05 | Aggressive reduction, performance-focused |
| **Perf** | Performance first. LOD0 ends at screen 0.90 — barely any time at full quality. All transitions packed 0.90→0.12 | Open-world, large scenes, distant objects |
| **Qual** | Quality first. LOD0 stays visible until screen 0.60. Gentle Tri% drop 0.85→0.20 | Hero props, cinematics, close-up assets |

### Fill Levels Button

After selecting a preset, **Fill Levels** re-applies the active preset formula to however many rows currently exist. Use this workflow:

1. Add/remove LOD rows until you have the count you want (e.g. 6)
2. Click a preset button (e.g. **Grad**)
3. Click **↓ Fill Levels** — all 6 rows recalculate using the Grad formula spread across 6 levels
4. Tweak individual rows if needed

---

## 7. LOD Levels Table

Each row represents one generated LOD (LOD1, LOD2, ...). LOD0 is always the original, untouched mesh.

| Column | Description |
|---|---|
| **#** | LOD index (LOD1, LOD2, ...) |
| **Screen Size** | UE screen-coverage threshold. When the mesh covers less than this fraction of the screen, UE switches to this LOD. Range 0.0–1.0. |
| **Tri%** | Triangle budget as a fraction of LOD0. 0.5 = keep 50% of original triangles. Range 0.01–1.0. |
| **QuadMerge** | Quad Based only. Merge N×N source quads into one LOD quad. 2 = 1/4 resolution, 4 = 1/16, etc. |
| **MaxDev** | Maximum allowed surface deviation in world units. 0 = preserve silhouette exactly. |
| **Est. Tris** | Live estimate of triangle count at this level based on the first selected mesh. |

### Adding / Removing Levels
- **+ Add LOD Level** — adds one row. Values auto-calculated from the last row (halved). Maximum 7 levels (UE hard limit: LOD0–LOD7).
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
| **Recompute Tangents** | On | Recompute tangents after reduction. Recommended to leave on. |
| **Follow Mesh Boundaries** | On | Quad Based only. Pins boundary vertices so silhouette, UV seams, and hard edges are preserved. Equivalent to Blender's Decimate "Follow Mesh Boundaries". |

---

## 9. Mesh List

The mesh list shows which static meshes will be processed when you click Generate.

**Populating the list:**
- Select Static Mesh assets in the **Content Browser**
- Or select **StaticMeshActors** in the **Level Viewport** (the plugin resolves to their source mesh)
- Click **↺ Refresh from Selection** to update the list

Each row shows the mesh name, LOD0 triangle count, and how many LOD levels it currently has.

---

## 10. Generating LODs

### Generate LODs for Selected Meshes
Processes only the meshes in the current Mesh List. Fast, targeted, good for iteration.

### Generate LODs for ALL Meshes in Project
Finds every Static Mesh asset in the entire project via the Asset Registry and queues them all. The editor stays responsive — meshes are processed one per tick using a ticker queue.

> Large projects with hundreds of meshes may take several minutes. Watch the Output Log for progress: `[MakeLOD] Processing mesh 12 / 247: SM_Rock_A`

**What happens to the mesh after generation:**
- LODs are written directly back into the same Static Mesh asset (no new assets created)
- All actors in the scene using that mesh update automatically — no rebake needed
- The package is marked dirty — save normally with Ctrl+S or Save All

---

## 11. Right-Click Shortcuts

You don't have to open the panel to generate LODs. Right-click shortcuts use the last-used settings (or Auto Redux defaults if the panel was never opened).

**In the Content Browser:**
Right-click any Static Mesh asset → **Generate LODs (MakeLOD)**

**In the Level Viewport:**
Select any StaticMeshActor → right-click → **Generate LODs (MakeLOD)**
The plugin resolves the actor's source mesh and processes it.

---

## 12. Technical Notes

### Reduction Backend
MakeLOD uses Unreal Engine's own `IMeshReductionInterface` (the Chaos quadric reducer). No third-party SDK is required. The reducer is always available regardless of engine installation.

### Async Processing
`GenerateLODsAsync()` uses `FTSTicker` to schedule one mesh per game-thread tick. `Build()` (the expensive UE step) must run on the game thread — this approach keeps the editor interactive while processing large batches.

### SeamlessRedux — How Border Locking Works
Before reduction, polygons that touch any open-edge (naked-edge) vertex are reclassified into a dedicated `MakeLOD_Border` polygon group. UE's reducer never collapses vertices across polygon group boundaries, which effectively freezes the border loops at full resolution while the interior is freely reduced.

### QuadBased — Grid Detection
The function detects the source grid by counting unique rounded X-axis values. `SrcVertsX` = number of unique X positions, `SrcVertsY` = total vertices / SrcVertsX. This works for any regular quad grid; non-grid meshes fall back gracefully to a coarser version of the same topology.

### Version Guards
The plugin includes compile-time version guards for UE 5.3 through 5.7 covering:
- `SetAutoComputeLODScreenSize` (5.5+)
- `SetNaniteSettings` (5.6+)
- Asset Registry class path API (5.1+)

---

## 13. FAQ

**Q: Will this delete my existing LODs?**
A: Yes. MakeLOD strips all existing LODs above LOD0 before regenerating them. LOD0 (the original mesh) is always preserved.

**Q: Can I use this on Nanite meshes?**
A: MakeLOD disables Nanite on meshes it processes, since Nanite and manual LODs are mutually exclusive in UE5. If you need Nanite, do not run MakeLOD on those assets.

**Q: My tiling meshes have seams between LOD levels. What do I do?**
A: Switch to **Seamless Redux** and make sure **Lock Border Vertices** is enabled. This guarantees the outer edge loops of every mesh are identical across all LOD levels.

**Q: QuadBased produces a flat mesh — what happened?**
A: QuadBased expects a regular grid mesh (terrain-style). If your mesh is not a uniform quad grid, QuadBased will not detect the grid correctly. Use Auto Redux or Seamless Redux instead.

**Q: "Generate LODs for ALL" is taking very long. Can I cancel?**
A: Not mid-batch in v1.0. Restart the editor to cancel. A cancel button is planned for a future update.

**Q: Does this work in Blueprints or at runtime?**
A: No. MakeLOD is an editor-only plugin with zero runtime footprint. All code is inside `#if WITH_EDITOR` guards.

**Q: Can I run this as part of a CI pipeline?**
A: `FLODGeneratorTool::GenerateLODs()` (synchronous) can be called from a commandlet. Wire it up in your own commandlet class and invoke it with `UE4Editor-Cmd.exe MyProject.uproject -run=YourCommandlet`.

---

*MakeLOD v1.0 — © kims ferdy 2026 — All Rights Reserved*
