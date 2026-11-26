# MakeLOD — Plugin Documentation
**Version 1.4 · Unreal Engine 5.2 – 5.8 · Editor Only**
*by kims ferdy · 2026*

---

## Table of Contents

1. [What is MakeLOD?](#1-what-is-makelod)
2. [Installation](#2-installation)
3. [Three Ways to Work](#3-three-ways-to-work)
4. [Interface Overview](#4-interface-overview)
5. [LOD Methods](#5-lod-methods)
6. [LOD Presets](#6-lod-presets)
7. [LOD Levels Table](#7-lod-levels-table)
8. [Options](#8-options)
9. [Mesh List](#9-mesh-list)
10. [Generating LODs](#10-generating-lods)
11. [Right-Click Shortcuts](#11-right-click-shortcuts)
12. [The MakeLOD Window](#12-the-makelod-window)
13. [Profiles](#13-profiles)
14. [Warnings and LOD Health](#14-warnings-and-lod-health)
15. [Technical Notes](#15-technical-notes)
16. [FAQ](#16-faq)
17. [Changelog](#17-changelog)

---

## 1. What is MakeLOD?

MakeLOD is a LOD generator for Static Meshes. You can batch an entire project's worth of meshes in one click, or open a single mesh in its own window and tune every LOD with a live preview before anything touches the asset.

**Key features:**

- Four reduction methods: Auto Redux, Seamless Redux, KDux, Modeling Tools (GeometryScript)
- **New in 1.4: the MakeLOD Window.** Edit one mesh's LODs with a live preview, see every LOD side by side in the Lineup view, drag a distance slider to watch the LODs switch like in game, then Apply (one Ctrl+Z undoes it)
- **New in 1.4: Profiles.** Save your LOD setup as an asset and apply it to one mesh or a hundred
- **New in 1.4: Warnings.** Nanite on, LOD Group overrides, LODs that save almost nothing, screen sizes in the wrong order
- Six one-click LOD presets that scale mathematically to any number of levels
- Triangle budget freedom: go as low as 0.1% of LOD0 (down to 4 triangles minimum)
- Batch processing with an async queue, so the editor stays responsive during generation
- Right-click shortcuts on any Static Mesh asset or scene actor
- Border-locked reduction (Seamless Redux) for tiling/modular meshes with zero seams
- KDux: a custom in-house QEM simplifier with hard-angle, UV-seam and material-boundary protection
- Modeling Tools integration: routes through Epic's GeometryScript simplifier for multi-threaded, engine-maintained reduction

---

## 2. Installation

1. Right-click your `.uproject` file → **Generate Visual Studio project files**.
2. Open the project. UE will prompt to compile the plugin. Click **Yes**.
3. After compilation, go to **Edit → Plugins**, search for `MakeLOD`, and make sure it is enabled.

> **Requirements:** Unreal Engine 5.2 or later. The plugin is editor-only and adds zero runtime overhead to your shipped game. The **Modeling Tools** method additionally requires the `GeometryScripting` plugin to be enabled. `MakeLOD.uplugin` declares this dependency, so you don't need to do anything.

---

## 3. Three Ways to Work

Pick whichever fits the job. All three write LODs the same way, so they give the same result for the same settings.

| Way | Open it from | Best for |
|---|---|---|
| **Right-click, direct** | Right-click meshes or actors → **Generate LODs (MakeLOD)** | "Just give it LODs." No window, it applies right away with the panel's current settings |
| **Batch panel** | `Window → MakeLOD Panel` | Many meshes at once, a whole folder or the whole project |
| **MakeLOD Window** | Right-click a mesh → **Open in MakeLOD**, or `Tools → MakeLOD Window` | One mesh where you want to own every LOD: preview, compare, then Apply |

Both tabs dock like any other UE tab. Drag them next to your Content Browser, Details panel, or anywhere else.

---

## 4. Interface Overview
Images here: https://www.fab.com/listings/f283df66-49f4-491c-bc1d-c2e45c8e7ff6

---

## 5. LOD Methods

Select a method before generating. The method determines *how* each LOD's geometry is reduced.

### Auto Redux
Standard quadric-error-metric reduction using UE's built-in reducer. Collapses vertices based on geometric importance. Fast and reliable for most meshes. May show minor seams at mesh boundaries when meshes tile.

**Best for:** Props, furniture, vehicles, characters, anything that doesn't tile.

### Seamless Redux
Same quadric reduction as Auto Redux, but border edge loops are split into a protected polygon group before reduction. UE's reducer never collapses across polygon group boundaries, so the open edges of the mesh are always preserved at full resolution.

**Best for:** Modular kits, tiling walls, floors, terrain chunks, any mesh that shares edges with a neighbour.

> Enable **Lock Border Vertices** in Options (on by default) for full protection.

### KDux
An in-house custom QEM simplifier (`FKDuxSimplify.h`). Uses quadric error metrics with explicit protection layers for hard edges, UV seams, material boundaries and high-curvature areas. Fully self-contained, no engine plugin dependency.

Each LOD level has its own KDux settings:

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
| **Algorithm** | Attribute Aware | **Standard QEM**: fastest. **Volume Preserving**: adds a volume correction pass, slower. **Attribute Aware**: also minimises UV and normal distortion; best quality, UE's own default. |
| **Allow Seam Collapse** | On | Allow edges that straddle a UV or material seam to be collapsed. Disable for stricter seam protection (same intent as KDux's UV Seam protection). |
| **Allow Seam Smoothing** | On | Allow seam vertices to be repositioned toward a better fit during collapse. |
| **Allow Seam Splits** | On | Allow seam edges to be split during simplification. Disable for the strictest seam protection. |
| **Preserve Vertex Positions** | Off | Skip quadric-optimal vertex repositioning, so collapsed vertices snap to an existing position. Higher shape fidelity; needs more triangles to reach equivalent quality. |

**Best for:** Any mesh where you want maximum quality with minimal setup. Pairs especially well with large batches since it's multi-threaded.

---

## 6. LOD Presets

Presets fill your LOD level rows with mathematically calculated values. They **never change the number of rows**, they only rewrite the values inside existing rows:

- Set up 6 LOD rows → click a preset → all 6 rows get the preset formula applied
- Add or remove rows → click **Fill Levels** → values update to fit the new row count

The same six presets are in the batch panel (buttons) and in the MakeLOD Window (**Preset ▾** menu in the top bar).

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
3. Click **↓ Fill Levels**. All rows recalculate using the formula spread across the current level count
4. Tweak individual rows if needed

---

## 7. LOD Levels Table

Each row represents one generated LOD (LOD1, LOD2, ...). LOD0 is always the original, untouched mesh.

| Column | Description |
|---|---|
| **#** | LOD index (LOD1, LOD2, ...) |
| **Screen Size** | UE screen-coverage threshold. When the mesh covers less than this fraction of the screen, UE switches to this LOD. Range 0.0–1.0. |
| **Tri%** | Triangle budget as a fraction of LOD0. 0.5 = 50% of original. **Minimum is 0.001** (0.1%): a 100 000 tri mesh at 0.001 produces ~100 tris. The hard floor is 4 triangles regardless of percentage. |
| **MaxDev** | Maximum allowed surface deviation in world units. 0 = preserve silhouette exactly. Used by Auto Redux and Seamless Redux. |
| **Est. Tris** | Live estimate of triangle count at this level based on the first selected mesh. |

### Adding / Removing Levels
- **+ Add LOD Level**: adds one row. Tri% is automatically set to half the last row's value, down to the 0.001 floor. Maximum 7 levels (UE hard limit: LOD0–LOD7).
- **- Remove Last**: removes the last row. Minimum 1 level.

---

## 8. Options

| Option | Default | Description |
|---|---|---|
| **Preserve UVs** | On | Keep all UV channels from LOD0 on every generated LOD. |
| **Preserve Materials** | On | Keep material slot assignments and polygon group layout. |
| **Preserve Vertex Colors** | On | Keep vertex color data on every generated LOD. |
| **Lock Border Vertices** | On | Seamless Redux only. Pins open edge loops so they are never collapsed. |
| **Recompute Normals** | Off | Recompute normals after reduction. Leave off to keep baked normals from LOD0. |
| **Recompute Tangents** | On | Recompute tangents after reduction. Recommended on. |

---

## 9. Mesh List

The mesh list in the batch panel shows which static meshes will be processed when you click Generate.

**Populating the list:**
- Select Static Mesh assets in the **Content Browser**
- Or select **StaticMeshActors** in the **Level Viewport**. The plugin resolves them to their source mesh
- Click **↺ Refresh from Selection** to update the list

Each row shows the mesh name, LOD0 triangle count, and how many LOD levels it currently has. The **Open** button on a row sends that mesh to the MakeLOD Window.

---

## 10. Generating LODs

### Generate LODs for Selected Meshes
Processes only the meshes in the current Mesh List. Fast and targeted, good for iteration.

### Generate LODs for ALL Meshes in Project
Finds every Static Mesh asset in the entire project via the Asset Registry and queues them all. The editor stays responsive: meshes are processed one per tick via an `FTSTicker` queue.

> Large projects may take several minutes. Watch the Output Log for progress:
> `[MakeLOD] Processing mesh 12 / 247: SM_Rock_A`

**What happens after generation:**
- LODs are written directly back into the same Static Mesh asset. No new assets are created
- All actors in the scene using that mesh update automatically
- Nanite is disabled on processed meshes (Nanite ignores manual LODs, see [Warnings](#14-warnings-and-lod-health))
- The package is marked dirty. Save normally with Ctrl+S or Save All

---

## 11. Right-Click Shortcuts

**In the Content Browser**, right-click any Static Mesh asset:
- **Open in MakeLOD**: opens the mesh in the [MakeLOD Window](#12-the-makelod-window). Nothing changes until you Apply.
- **Generate LODs (MakeLOD)**: direct impact. Generates LODs for every selected mesh right away, no window.

**In the Level Viewport**, select any StaticMeshActor → right-click → **Generate LODs (MakeLOD)**.

**Generate LODs** uses the batch panel's current settings (method, rows, options, or a loaded profile) if the panel is open, or the Auto Redux defaults if it isn't. So for direct impact with your own setup: open the panel once, pick a preset or a profile, then right-click away.

---

## 12. The MakeLOD Window

The MakeLOD Window gives one mesh its own dockable tab where you own every LOD. The layout follows the Static Mesh editor, so there's nothing new to learn.

**Opening it:**
- Content Browser → right-click a Static Mesh → **Open in MakeLOD**
- The **Open** button on any row of the batch panel's mesh list
- `Tools → MakeLOD Window` opens it empty. Pick a mesh with the picker at the top

Switching to another mesh, or closing the tab, with unapplied edits asks first.

### Layout

| Area | What's there |
|---|---|
| **Top bar** | Mesh picker · **Profile**: Preset ▾, profile asset picker, Save profile... · status line · **Live**, **Revert**, **Apply** |
| **Left: preview** | The stock viewport toolbar (view mode Lit / Unlit / Wireframe..., Show flags, camera) plus a **LOD Auto** menu. The top-left readout shows LOD, Current Screen Size, Triangles, Vertices and Sections |
| **Under the preview** | **Single / Lineup** toggle and the **Distance** slider |
| **Right: LOD Details** | Warnings, then one row per LOD (%, screen size, built tris and verts), **+ Add LOD / Remove last**, the selected LOD's settings, and the mesh-wide method and options |

When a mesh opens, its rows are read back from its current LODs. A mesh with only LOD0 starts with the default ladder.

### Live preview, Apply and Revert
- Edits work on a **temporary copy** of the mesh. Your asset is never touched until **Apply**.
- The preview rebuilds about 0.3 s after you stop editing (and after you let go of the mouse). Only the LOD you edited is reduced again.
- **Apply** writes every LOD to the real mesh as **one undo step**, so Ctrl+Z reverts the whole thing.
- **Revert** throws the edits away and re-reads the rows from the mesh.
- **Live** turns the preview on or off. It's on by default, because static meshes build fast.

### LOD Auto
The **LOD Auto** menu in the viewport toolbar works like the Static Mesh editor's. **Auto** lets the engine pick the LOD by screen size, using the same call the renderer uses in game. **LOD 0..N** locks one LOD.

### Lineup view
**Lineup** shows every LOD side by side in one row, LOD0 on the left. A label floats above each one: `LOD 2` and `3,012 tris · 25% · 0.080` (triangles, % of LOD0, screen size).
- Click a mesh or its label to select that LOD. The selected label is highlighted.
- The camera frames the whole row when you enter Lineup or the LOD count changes.
- Edits update live in the row.

> **Tip:** switch the viewport to **Wireframe** in Lineup. It's the fastest way to read how the density drops from one LOD to the next.

### Distance slider
Drag it to move the camera away from the mesh. It works in Single and Lineup. The readout shows the distance in meters, the live screen size, and which LOD Auto picks there. Mouse-wheel zoom moves the slider too.

In Lineup, the LOD the engine would use at that distance gets a green **IN USE AT THIS DISTANCE** tag.

**Use view** (next to the selected LOD's screen size) sets that LOD's screen size to the current view's screen size. Frame the mesh the way it should look when the LOD switches, then click it.

### Per-LOD settings
Select a row to edit just that LOD: Tri%, screen size, MaxDev, and the KDux or Modeling Tools settings. The method and the options (Preserve UVs, etc.) apply to the whole mesh and sit in the **Mesh (all LODs)** section.

---

## 13. Profiles

A **MakeLOD Profile** is an asset that stores a full LOD setup: the method, every LOD row, and the options. Tune it once and reuse it everywhere.

**Saving:**
- MakeLOD Window → **Save profile...**
- Batch panel → **Profile** row → **Save...**

Pick a folder and name (default `MLP_NewProfile`), then save the asset as usual (Ctrl+S).

**Using:**
- **Window:** pick a profile in the top bar. It replaces the method, rows and options, and the preview rebuilds.
- **Batch panel:** pick a profile in the **Profile** row, select your meshes, then **Generate**. Ten meshes, a hundred, they all get the same setup. With the panel open, the right-click **Generate LODs** shortcut uses it too.

You can also open a profile asset and edit it directly. Triangles are shown as **Triangles Kept (0-1)**.

> Profiles are editor-only assets, they are never cooked into your game build. A profile with more than 7 rows (hand-edited) is clamped to 7 when loaded.

---

## 14. Warnings and LOD Health

The MakeLOD Window shows warnings at the top of LOD Details, and a red line under any LOD row that looks wrong.

### Nanite is on (orange)
Nanite ignores manual LODs wherever it runs. Only mobile and Nanite-off platforms use them. MakeLOD turns Nanite off when it writes LODs, so the LODs are used everywhere, and the status line says "Nanite turned off" after Apply. **Skip the mesh if it should stay Nanite on PC and consoles.** The batch panel and the right-click shortcut turn Nanite off too.

### LOD Group is set (yellow)
A LOD Group can silently override what you set:
- On every build, the group scales each LOD's reduction, so your rows don't come out exactly as typed.
- Picking a group again in the Static Mesh editor resets the LOD rows and turns Auto screen size back on.

The warning has a **Set LOD Group to None on Apply** checkbox, on by default. The preview follows the checkbox, so what you see is what Apply writes. Clearing only drops the group name. It doesn't touch your LODs.

### Row health (red)
- **"Keeps X% of LODn's triangles..."**: this LOD saves less than 10% over the one before it. A LOD swap that saves almost nothing still costs a swap. Lower its Tri% or remove it.
- **"Screen size isn't below LODn's..."**: screen sizes must go down from LOD to LOD. Otherwise the previous LOD never shows.

When the preview is up to date, the check uses the real built triangle counts. While an edit is still building, it uses the row's Tri%.

---

## 15. Technical Notes

### Reduction Backends
- **UE built-in** (`IMeshReductionInterface` / quadric reducer): used by Auto Redux and Seamless Redux. Always available, no extra dependencies.
- **GeometryScript** (`UGeometryScriptLibrary_MeshSimplifyFunctions`): used by Modeling Tools. Requires the `GeometryScripting` UE plugin (declared automatically). Internally converts `FMeshDescription` ↔ `UDynamicMesh` for the simplification pass.
- **KDux** (`FKDuxSimplify.h`): fully self-contained, in-house QEM. No engine or plugin dependency.

### One Write Path
`FLODGeneratorTool` is the only code that writes LODs. `BuildLevelSource` builds one LOD's source mesh and `WriteLODs` writes them all with a single `Build()`. The batch panel, the right-click shortcuts and the MakeLOD Window all go through it, so the same settings always give the same result.

### Triangle Floor
All backends enforce a minimum of 4 triangles regardless of `PercentTriangles`. This is the lowest count that can form a valid closed surface. The `PercentTriangles` field accepts values down to `0.001` (0.1%).

### Async Processing
`GenerateLODsAsync()` uses `FTSTicker` to schedule one mesh per game-thread tick. `UStaticMesh::Build()` must run on the game thread, and this keeps the editor interactive during large batches. The window's preview rebuilds also run off the core ticker.

### Live Preview Copy
The window's preview is a `DuplicateObject` of the mesh in the transient package. It is never saved and never enters the undo history. Each LOD's reduced source is cached, so editing one LOD only re-runs that LOD's reduction (the engine's `Build()` still rebuilds the render data of all LODs).

### SeamlessRedux — How Border Locking Works
Before reduction, polygons that touch any open-edge (naked-edge) vertex are reclassified into a dedicated `MakeLOD_Border` polygon group. UE's reducer never collapses vertices across polygon group boundaries, which freezes the border loops at full resolution while the interior is freely reduced.

### KDux — Double-Reduction Prevention
KDux and Modeling Tools deliver a fully pre-simplified `FMeshDescription`. The engine's built-in reducer is bypassed for these methods by setting `PercentTriangles = 1.0` and `MaxNumOfTriangles = MAX_int32` on the source model's `ReductionSettings`. Otherwise the engine would apply a second reduction pass on top of the custom one.

### Version Guards
Compile-time version guards cover:
- `SetAutoComputeLODScreenSize` (UE 5.7+)
- `GetNaniteSettings` / `SetNaniteSettings` (UE 5.7+)
- `GetLODGroup` / `SetLODGroup_Unsafe` (UE 5.7+)

1.4 is compile-checked on UE 5.2, 5.3, 5.4, 5.5, 5.6, 5.7 and 5.8.

---

## 16. FAQ

**Q: Will this delete my existing LODs?**
A: Yes. MakeLOD strips all existing LODs above LOD0 before regenerating. LOD0 (the original mesh) is always preserved. In the MakeLOD Window the mesh's current LODs are read back as editable rows first, and nothing changes until you Apply.

**Q: Can I use this on Nanite meshes?**
A: MakeLOD turns Nanite off on meshes it processes, because Nanite ignores manual LODs. The MakeLOD Window warns you before you Apply. Don't run MakeLOD on assets you want to keep as Nanite on PC and consoles.

**Q: I want to try settings without wrecking my mesh.**
A: Use **Open in MakeLOD**. The window previews on a temporary copy, and Apply is a single undo step.

**Q: I don't want a window, just LODs.**
A: Right-click → **Generate LODs (MakeLOD)**. It applies straight away with the batch panel's current settings (or Auto Redux defaults if the panel is closed).

**Q: How do I use the same LOD setup on lots of meshes?**
A: Save a **Profile** (window or panel), load it in the batch panel, select your meshes and Generate.

**Q: How low can Tri% go?**
A: Down to 0.001 (0.1%). The absolute floor is 4 triangles regardless of percentage. On a 100 000 tri mesh at 0.001 you get roughly 100 tris. On a 500 tri mesh at 0.001 you hit the 4-tri floor.

**Q: My tiling meshes have seams between LOD levels. What do I do?**
A: Switch to **Seamless Redux** with **Lock Border Vertices** enabled. This guarantees the outer edge loops of every mesh are identical across all LOD levels.

**Q: Which method gives the best quality?**
A: It depends on the mesh. **Modeling Tools** with Attribute Aware is a great default, multi-threaded and low on UV and normal distortion. **KDux** gives finer per-level control, useful for hero props. Open the mesh in the MakeLOD Window, switch methods and compare them in the Lineup view.

**Q: What's the difference between KDux and Modeling Tools?**
A: Both are QEM-based but they're independent implementations. KDux is ours: fully self-contained, no plugin dependency, with explicit per-LOD controls for hard-angle threshold and aggressiveness. Modeling Tools routes through Epic's GeometryScript, is multi-threaded on large meshes, and gets maintenance from Epic with each engine update. Neither is strictly better, they complement each other.

**Q: My LOD Group keeps changing my LODs.**
A: Leave **Set LOD Group to None on Apply** ticked in the MakeLOD Window. See [LOD Group is set](#lod-group-is-set-yellow).

**Q: "Generate LODs for ALL" is taking very long. Can I cancel?**
A: Not mid-batch in the current version. Restart the editor to cancel.

**Q: Does this work in Blueprints or at runtime?**
A: No. MakeLOD is editor-only with zero runtime footprint.

**Q: Can I run this from a commandlet / CI pipeline?**
A: `FLODGeneratorTool::GenerateLODs()` (synchronous) can be called from a commandlet. Wire it up in your own commandlet class and invoke it with `UnrealEditor-Cmd.exe MyProject.uproject -run=YourCommandlet`.

---

## 17. Changelog

### v1.4
- **MakeLOD Window**: a dockable editor for one mesh's LODs, laid out like the Static Mesh editor
  - Live preview on a temporary copy, Apply as one undo step, Revert
  - **Lineup** view: every LOD side by side with labels (tris, % of LOD0, screen size), click to select
  - **Distance** slider with live screen size, the LOD Auto picks, an "IN USE AT THIS DISTANCE" tag, and **Use view**
  - Per-LOD editing, + Add LOD / Remove last, LOD Auto menu, Persona-style stats readout
  - Opens from Content Browser right-click **Open in MakeLOD**, the panel's **Open** button, or `Tools → MakeLOD Window`
- **Profiles**: `MakeLOD Profile` assets store the method, rows and options. Save and load from the window and the batch panel
- Presets are shared between the panel and the window (**Preset ▾** menu in the window)
- **Warnings**: Nanite on, LOD Group set (with "Set LOD Group to None on Apply"), per-LOD row health (under 10% savings, screen size order)
- Right-click **Generate LODs** now really uses the batch panel's current settings when the panel is open (1.3 always used the Auto Redux defaults)
- Minimum engine lowered to UE 5.2; compile-checked on 5.2 through 5.8

### v1.3
- Triangle floor lowered from 5% to **0.1% (0.001)** across all methods and presets. The last LOD can now go as low as 4 triangles
- `PercentTriangles` spinbox updated: minimum `0.001`, delta `0.001`, 3 decimal places displayed
- All preset curve floors updated to match, no more invisible 5% clamp on deep LOD levels
- `OnAddLODLevel` auto-step floor updated to `0.001`

### v1.2
- **Modeling Tools method added**: routes through Epic's GeometryScript (`ApplySimplifyToTriangleCount`). Requires the `GeometryScripting` UE plugin (declared automatically as a dependency in `.uplugin`)
- **KDux method**: custom in-house QEM simplifier with per-level hard-angle, UV-seam, material-boundary and aggressiveness controls
- **Custom % method removed**, redundant with Auto Redux
- Per-level Modeling Tools settings panel: algorithm selector (Standard QEM / Volume Preserving / Attribute Aware), seam collapse, seam smoothing, seam splits, preserve vertex positions
- Method buttons now full-width for KDux and Modeling Tools
- Double-reduction guard added: KDux and Modeling Tools pre-simplified meshes now correctly bypass the engine's built-in reducer

### v1.1
- Initial public release
- Auto Redux, Seamless Redux, KDux, Custom % methods
- Six presets, async batch queue, Content Browser and actor right-click shortcuts

---

*MakeLOD v1.4 — © kims ferdy 2026 — All Rights Reserved*
