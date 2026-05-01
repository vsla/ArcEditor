# ArcEditor — Improvement Roadmap

> Target repo: [pascalorg/editor](https://github.com/pascalorg/editor)
> Stack: Next.js 16 · React Three Fiber · Three.js WebGPU · Zustand · Zod · Turborepo
>
> Goal: elevate ArcEditor to a professional-grade architectural tool — free, open-source, AI-powered.

---

## Epic 1 — Camera, Navigation & Viewport

The viewport is the core interaction surface. Every second spent fighting the camera is a second not designing.

### Ticket 1.1 — Orbit Around Cursor

**Problem**
Camera always orbits around the scene center. SketchUp orbits around the point under the cursor, making it effortless to inspect any part of a model without repositioning.

**Acceptance Criteria**
- Middle-mouse drag orbits around the 3D point under the cursor at the moment of press
- If cursor misses geometry, falls back to current behavior
- Double-click on any object focuses and orbits around it

**Technical Approach**
- `packages/editor/src/components/editor/custom-camera-controls.tsx`
- On middle `pointerdown`: raycast against scene → `controls.setOrbitPoint(x, y, z)` (Drei CameraControls supports natively)
- Double-click: emit event via bus → camera interpolates with `setLookAt` to object bounding center

**PR Label:** `feat/camera` | **Package:** `editor`

---

### Ticket 1.2 — Unrestricted Zoom

**Problem**
Zoom locks up far from geometry. Impossible to inspect fine details. SketchUp zooms all the way into surfaces.

**Acceptance Criteria**
- Zoom in goes up to ~1cm from the nearest surface
- Zoom out covers the entire scene without locking
- No distance snapping or hard stops

**Technical Approach**
- `custom-camera-controls.tsx` — tune `minDistance`, `maxDistance`, `dollySpeed`
- Enable `infinityDolly` (native Drei CameraControls option)
- Perspective-correct zoom: dolly speed scales with distance to target

**PR Label:** `feat/camera` | **Package:** `editor`

---

### Ticket 1.3 — Standard Views (Top, Front, Side, Isometric)

**Problem**
No quick way to jump to orthographic standard views. Architects constantly switch between plan, elevation, and isometric to verify dimensions and proportions.

**Acceptance Criteria**
- Keyboard shortcuts: `Numpad 7` top, `Numpad 1` front, `Numpad 3` right, `Numpad 5` toggle ortho/perspective
- Toolbar buttons for standard views
- Smooth animated transition to each view
- Orthographic projection activates automatically for plan/elevation views

**Technical Approach**
- `custom-camera-controls.tsx` — `rotateTo()` + `setLookAt()` presets per view
- Update `useViewer` store with `activeView` state
- Add view buttons to existing toolbar

**PR Label:** `feat/camera` | **Package:** `editor`

---

### Ticket 1.4 — First-Person Walk Mode

**Problem**
No way to walk through a model at eye level. Critical for validating spatial experience — how a room actually feels from human height.

**Acceptance Criteria**
- Toggle with `F` or button in toolbar
- WASD + mouse look (pointer lock)
- Configurable walk speed (scroll wheel)
- Collision with walls optional (toggle)
- Exit with `Escape`

**Technical Approach**
- `custom-camera-controls.tsx` — swap to PointerLockControls when mode is `walk`
- `useViewer` store: `cameraMode: 'orbit' | 'walk'`
- Collision: raycast downward for floor detection, forward for wall avoidance

**PR Label:** `feat/camera` | **Package:** `editor`

---

### Ticket 1.5 — Saved Camera Bookmarks

**Problem**
No way to save and recall specific viewpoints. Architects need to revisit the same angles repeatedly (entry view, kitchen view, bedroom view).

**Acceptance Criteria**
- `Ctrl+Shift+[1-9]` saves current camera position
- `Shift+[1-9]` recalls saved view with smooth transition
- Saved views listed in a sidebar panel
- Views persist across sessions (saved to scene data)

**Technical Approach**
- Extend `camera` schema in `packages/core/src/schema/camera.ts` with named bookmarks array
- `useScene` store saves/restores via existing IndexedDB persistence

**PR Label:** `feat/camera` | **Package:** `editor` + `core`

---

## Epic 2 — Smart Snap & Inference Engine

SketchUp's snap system is its most-loved feature. Magnetic snapping to geometry turns rough placement into precise modeling.

### Ticket 2.1 — Vertex, Edge & Face Snap

**Problem**
Only grid snapping exists. No way to precisely align objects to corners, edges, or face centers of other geometry.

**Acceptance Criteria**
- Object drag magnetizes to: vertices, edge midpoints, face centers of other objects
- Snap priority: vertex > edge midpoint > face center > grid
- Visual indicator shows active snap type (colored dot or icon)
- Snap threshold configurable in settings (default ~0.15m)
- Works in both 2D floorplan and 3D view

**Technical Approach**
- `packages/editor/src/components/tools/item/placement-math.ts` — new `findNearestSnapPoint(worldPoint, registry, threshold)` using BVH raycasting (`three-mesh-bvh` already present)
- New `packages/editor/src/components/editor/snap-indicator.tsx` — small `<mesh>` at snap position with type-based color
- `packages/editor/src/components/tools/item/use-placement-coordinator.tsx` — integrate before applying final position

**PR Label:** `feat/smart-snap` | **Package:** `editor`

---

### Ticket 2.2 — Alignment Inference Lines

**Problem**
No visual feedback when objects are aligned with each other. SketchUp shows colored guide lines (red = X axis, green = Y axis, blue = Z axis, magenta = aligned with another object).

**Acceptance Criteria**
- During drag, guide lines appear when item aligns in X, Y or Z with another object
- Lines disappear when moving out of alignment
- Works in 2D and 3D
- Different colors per axis (standard: red/green/blue)

**Technical Approach**
- New `packages/editor/src/components/editor/alignment-guides.tsx` — `<Line>` from Drei
- Compute alignments in `use-placement-coordinator.tsx` comparing bounding boxes via `sceneRegistry`

**PR Label:** `feat/smart-snap` | **Package:** `editor`

---

### Ticket 2.3 — Angle Snap & Axis Lock

**Problem**
No way to constrain movement or rotation to specific angles or axes. Results in objects placed slightly off-axis.

**Acceptance Criteria**
- Hold `Shift` during drag: locks to 45° angle increments
- Hold `X`, `Y`, `Z` during drag: locks movement to that world axis
- Hold `Shift+X/Y/Z`: locks to plane perpendicular to that axis
- Visual axis indicator shown during lock

**Technical Approach**
- `use-placement-coordinator.tsx` — keyboard modifier detection during drag
- Apply axis constraint as vector projection before position update
- Reuse existing angle snap logic from `wall-drafting.ts`

**PR Label:** `feat/smart-snap` | **Package:** `editor`

---

### Ticket 2.4 — Construction Guide Lines

**Problem**
No way to draw reference lines for precise placement. SketchUp's Tape Measure tool creates infinite guide lines that persist as placement aids.

**Acceptance Criteria**
- `G` key activates Guide tool
- Click on edge/face → drag → places infinite guide line at offset distance
- Guides snap like regular geometry
- Guides visible but non-printing (dashed, different layer)
- Delete all guides with single command (`Edit → Delete Guides`)

**Technical Approach**
- New `guide-line-node` schema in `packages/core/src/schema/`
- New renderer `packages/viewer/src/components/renderers/guide-line-renderer.tsx`
- New tool `packages/editor/src/components/tools/guide/guide-tool.tsx`

**PR Label:** `feat/smart-snap` | **Package:** `editor` + `core` + `viewer`

---

## Epic 3 — Dimensions, Scale & Measurement

Precision is the difference between a sketch and a technical drawing.

### Ticket 3.1 — Item Dimension Panel

**Problem**
No way to see or edit the real-world dimensions (W/H/D) of a placed item. Users have no feedback on actual object size.

**Acceptance Criteria**
- Side panel shows width, height, depth of selected item in m or ft
- Editing a value rescales the object while keeping its center fixed
- "Lock aspect ratio" toggle available
- Uses existing `MetricControl` component (drag-to-adjust, scroll wheel, keyboard)
- Works with all placed items regardless of original model scale

**Technical Approach**
- Read bounding box via `sceneRegistry.getObject(id).geometry.boundingBox`
- Compute `scale.x = targetW / originalW`
- Add optional `dimensions` field to `packages/core/src/schema/item.ts` to store original reference size
- Add panel to existing item property sidebar

**PR Label:** `feat/dimensions` | **Package:** `editor` + `core`

---

### Ticket 3.2 — Type Dimension During Placement

**Problem**
No way to type an exact value during a drag operation. SketchUp: drag to set direction, type a number, press Enter — object snaps to exact dimension.

**Acceptance Criteria**
- During any drag, typing a number opens a floating input at cursor position
- `Enter` confirms and applies typed dimension
- `Escape` dismisses and returns to drag
- Supports unit suffixes: `3.5m`, `3500mm`, `11'6"`
- Works for: item placement, wall drawing, object move

**Technical Approach**
- `use-placement-coordinator.tsx` — capture `keydown` digits during active drag
- Render `<Html>` from Drei at cursor with number input
- Apply as relative scale / absolute position before confirming

**PR Label:** `feat/dimensions` | **Package:** `editor`

---

### Ticket 3.3 — Free Measure Tool

**Problem**
Only automatic wall measurements exist. No way to measure arbitrary distances between points, objects, or calculate room area.

**Acceptance Criteria**
- "Measure" tool in toolbar, hotkey `M`
- Click point A → click point B → real-time distance display
- 3D label with line between points, metric and imperial
- Option to pin measurement to scene (persists as node, can be deleted)
- Pinned measurements appear in node tree
- Area mode: click 3+ points, displays polygon area

**Technical Approach**
- New `packages/editor/src/components/tools/measure/measure-tool.tsx`
- New `measure-node` schema in `packages/core/src/schema/`
- New renderer `packages/viewer/src/components/renderers/measure-renderer.tsx`
- Reuse rendering logic from `wall-measurement-label.tsx`

**PR Label:** `feat/measure-tool` | **Package:** `editor` + `core` + `viewer`

---

### Ticket 3.4 — Angle Measurement

**Problem**
No way to measure angles between surfaces or edges. Critical for verifying roof pitch, stair angles, and non-orthogonal walls.

**Acceptance Criteria**
- Angle mode in Measure Tool (toggle)
- Click vertex → click edge A → click edge B → displays angle
- Shows both the angle and its complement (e.g. 30° / 150°)
- Can be pinned to scene like linear measurements

**Technical Approach**
- Extend `measure-tool.tsx` with angle mode state machine
- Compute angle via `Math.acos(dot(a, b))` between vectors
- Render arc indicator with label

**PR Label:** `feat/measure-tool` | **Package:** `editor`

---

## Epic 4 — Visual Quality & Rendering

First impressions matter. A beautiful render validates design decisions and impresses clients.

### Ticket 4.1 — Ambient Occlusion (SSAO)

**Problem**
Scene looks flat. Without AO, objects lose depth and spatial relationship. Corners, under-furniture shadows, and wall junctions all read identically.

**Acceptance Criteria**
- AO visible in corners, under furniture, between walls
- Doesn't significantly impact performance (60fps on mid-range GPU, ~50 objects)
- Toggle in settings to disable on low-end machines
- Intensity and radius configurable

**Technical Approach**
- Three.js WebGPU r184 has native `ao-pass` — enable in viewer renderer
- Add `aoEnabled`, `aoIntensity` to `useViewer` store
- Toggle UI in Settings panel

**PR Label:** `feat/visual-quality` | **Package:** `viewer`

---

### Ticket 4.2 — Soft Shadows

**Problem**
Current shadows are either absent or pixelated hard-edge shadows. Soft shadows dramatically improve spatial reading of a scene.

**Acceptance Criteria**
- Soft shadow penumbra on all shadow-casting objects
- Configurable shadow quality (low/medium/high)
- Shadows cast by sunlight direction (configurable angle)
- Toggle in settings

**Technical Approach**
- Switch renderer to `PCFSoftShadowMap` or `VSMShadowMap`
- Add `shadowsEnabled`, `shadowQuality` to `useViewer` store
- Sun direction configurable via azimuth/elevation sliders in Settings

**PR Label:** `feat/visual-quality` | **Package:** `viewer`

---

### Ticket 4.3 — HDRI Environment Lighting

**Problem**
Flat directional lighting makes glass, metal, and polished materials look dull. HDRI brings realistic reflections and color temperature to the scene.

**Acceptance Criteria**
- Environment map applied to scene
- Glass and polished surfaces reflect environment
- At least 3 presets: Exterior Day, Interior Studio, Neutral White
- UI to select preset in Settings panel
- Optional: custom HDRI upload

**Technical Approach**
- `<Environment>` from Drei with presets `"apartment"`, `"city"`, `"dawn"`
- Add `environmentPreset` to `useViewer` store
- UI in existing Settings panel

**PR Label:** `feat/visual-quality` | **Package:** `viewer` + `editor`

---

### Ticket 4.4 — Material & Texture Library UI

**Problem**
80+ material presets exist in `material-library.ts` but the UI for applying them is incomplete. Users can't browse, preview, or apply textures visually.

**Acceptance Criteria**
- Material picker shows visual grid of texture thumbnails
- Filter by category: wood, stone, tile, wallpaper, metal, glass
- Drag material onto surface to apply (or click to apply to selected)
- Custom color override per material
- Roughness and scale sliders per applied material

**Technical Approach**
- Expand `packages/editor/src/components/ui/controls/material-picker.tsx`
- Thumbnail generation from existing texture maps in `material-library.ts`
- Apply via `updateNode` on selected surface node

**PR Label:** `feat/visual-quality` | **Package:** `editor`

---

### Ticket 4.5 — Selection Outline Effect

**Problem**
Selected objects use a color change to indicate selection, which can be hard to read depending on material. SketchUp uses a blue outline overlay.

**Acceptance Criteria**
- Selected objects render with a clear outline (configurable color, default blue)
- Hovered objects show a lighter outline
- Outline works on all object types
- No performance regression

**Technical Approach**
- `@react-three/postprocessing` Outline effect (already in ecosystem)
- Pass selected object refs to `<Outline>` in viewer renderer
- Integrate with existing selection system via `sceneRegistry`

**PR Label:** `feat/visual-quality` | **Package:** `viewer`

---

## Epic 5 — Drawing & Direct Modeling

SketchUp's power comes from direct surface manipulation. These tools close the biggest modeling gap.

### Ticket 5.1 — Push/Pull Faces

**Problem**
No way to extrude or inset faces directly. Push/Pull is SketchUp's most iconic feature — select a face, push or pull to add/remove volume.

**Acceptance Criteria**
- Hover face → handle appears → drag to extrude or inset
- Works on slab surfaces, wall faces, roof panels
- Negative drag removes material (cut operation via existing CSG)
- Supports typed input during drag (Ticket 3.2)

**Technical Approach**
- New `packages/editor/src/components/tools/pushpull/push-pull-tool.tsx`
- Detect hovered face via BVH raycasting
- Extrude: generate new geometry by offsetting face along normal
- CSG remove: use existing `three-bvh-csg` for subtract operations

**PR Label:** `feat/direct-modeling` | **Package:** `editor`

---

### Ticket 5.2 — Edge Offset Tool

**Problem**
No way to offset a wall line or edge inward/outward at a specific distance. Essential for creating wall thickness variations, countertops, and structural offsets.

**Acceptance Criteria**
- Select wall or slab edge → Offset tool → drag to set distance
- Preview shows new edge position with distance label
- Typed input supported
- Applies to polygonal shapes (slabs, zones)

**Technical Approach**
- New `packages/editor/src/components/tools/offset/offset-tool.tsx`
- 2D offset math using existing `polygon-clipping` library
- Integrate with slab/ceiling boundary editors

**PR Label:** `feat/direct-modeling` | **Package:** `editor`

---

## Epic 6 — Export & Import

Interoperability is critical for professional workflows. Architects share files across tools constantly.

### Ticket 6.1 — Export to SketchUp via Collada (.dae)

**Problem**
No way to export to SketchUp. Collada is the standard intermediate format accepted natively by SketchUp (File → Import → Collada).

**Acceptance Criteria**
- "Export for SketchUp (.dae)" button in export menu
- Full geometry exported: walls, slabs, roofs, stairs, items
- Basic materials and colors preserved
- File opens cleanly in SketchUp 2021+

**Technical Approach**
- Add `ColladaExporter` (Three.js contrib) to `packages/viewer/src/systems/export/export-system.tsx`
- Same pattern as GLB: strip `EDITOR_LAYER` objects before export
- Add `format: 'dae'` option to export manager UI

**PR Label:** `feat/export` | **Package:** `viewer` + `editor`

---

### Ticket 6.2 — Import External GLB/GLTF Models

**Problem**
No way to import external 3D models (furniture, fixtures, equipment). Severely limits the item catalog beyond built-in assets.

**Acceptance Criteria**
- "Import 3D Model" button in UI (accepts `.glb`, `.gltf`)
- Imported model becomes a standard scene item (movable, deletable, scalable)
- Bounding box computed automatically
- Appears in node tree like any other item
- File stored as blob URL (no server upload required)

**Technical Approach**
- `GLTFLoader` already available via Three.js/R3F
- `<input type="file">` → FileReader → blob URL
- Create `item` node with `modelUrl: blobUrl` in scene store
- Persist blob to IndexedDB via existing `idb-keyval`

**PR Label:** `feat/import` | **Package:** `editor` + `core`

---

### Ticket 6.3 — IFC Export (BIM Standard)

**Problem**
No BIM interoperability. IFC (Industry Foundation Classes) is the universal open standard for architectural data exchange — required by many professional workflows, engineers, and contractors.

**Acceptance Criteria**
- "Export IFC" option in export menu
- Walls, slabs, doors, windows, stairs exported as correct IFC entity types
- Level/storey hierarchy preserved
- Compatible with common BIM viewers (BIMvision, Navisworks)

**Technical Approach**
- Integrate `web-ifc` or `ifc.js` library (open-source)
- Map ArcEditor node types to IFC entities: wall→`IfcWall`, slab→`IfcSlab`, door→`IfcDoor`, etc.
- New `packages/viewer/src/systems/export/ifc-export.ts`

**PR Label:** `feat/export` | **Package:** `viewer`

---

### Ticket 6.4 — PDF Floor Plan Export

**Problem**
No way to export a clean floor plan drawing. Architects need to share 2D floor plans as PDFs for clients, contractors, and permits.

**Acceptance Criteria**
- "Export Floor Plan (PDF)" in export menu
- Clean 2D linework at selected level
- Walls, doors, windows, dimensions rendered at architectural scale
- Scale bar and north arrow included
- Paper size configurable (A4, A3, Letter)

**Technical Approach**
- Use existing 2D floorplan renderer (`packages/editor/src/components/editor-2d/`)
- Capture SVG output → convert to PDF via `jspdf` + `svg2pdf.js`
- Scale calibration from real-world dimensions

**PR Label:** `feat/export` | **Package:** `editor`

---

## Epic 7 — AI Features

ArcEditor is free and AI-powered. These features make it genuinely intelligent.

### Ticket 7.1 — AI Room Layout Suggestions

**Problem**
Starting from a blank canvas is hard. Users with a room boundary should get instant furniture layout suggestions.

**Acceptance Criteria**
- Select a zone/room → "Suggest Layout" button appears
- AI returns 2-3 furniture arrangement options
- User can preview each and accept with one click
- Items placed at correct scale for room dimensions

**Technical Approach**
- Compute room polygon and area from zone node
- Send to MCP tool (existing `packages/mcp/`) with room context
- Claude returns item IDs + positions → apply via `createNode` batch
- Use existing `agentation` package already in dependencies

**PR Label:** `feat/ai` | **Package:** `mcp` + `editor`

---

### Ticket 7.2 — Natural Language to Scene

**Problem**
Complex scene building requires knowing all tools. Natural language commands lower the barrier: "add a kitchen island 2m wide against the north wall."

**Acceptance Criteria**
- Command palette (`Cmd+K`) accepts natural language
- Examples: "add 3 chairs around the table", "rotate the sofa 90 degrees", "duplicate this room on floor 2"
- AI interprets intent and executes scene operations
- Undo-able as a single operation

**Technical Approach**
- Extend existing command palette with AI mode
- MCP tools provide scene read/write primitives to Claude
- Claude maps natural language → MCP tool calls → scene updates

**PR Label:** `feat/ai` | **Package:** `mcp` + `editor`

---

## Implementation Order

| # | Ticket | PR Label | Impact | Complexity |
|---|--------|----------|--------|------------|
| 1 | Orbit Around Cursor | `feat/camera` | High | Low |
| 2 | Unrestricted Zoom | `feat/camera` | High | Low |
| 3 | Standard Views | `feat/camera` | High | Low |
| 4 | Selection Outline Effect | `feat/visual-quality` | High | Low |
| 5 | SSAO | `feat/visual-quality` | High | Low |
| 6 | Soft Shadows | `feat/visual-quality` | High | Low |
| 7 | Vertex/Edge/Face Snap | `feat/smart-snap` | High | Medium |
| 8 | Alignment Inference Lines | `feat/smart-snap` | Medium | Medium |
| 9 | Angle Snap & Axis Lock | `feat/smart-snap` | Medium | Low |
| 10 | Item Dimension Panel | `feat/dimensions` | Medium | Medium |
| 11 | Type Dimension During Placement | `feat/dimensions` | Medium | Medium |
| 12 | Free Measure Tool | `feat/measure-tool` | Medium | Medium |
| 13 | Angle Measurement | `feat/measure-tool` | Medium | Low |
| 14 | HDRI Environment Lighting | `feat/visual-quality` | Medium | Low |
| 15 | Material Library UI | `feat/visual-quality` | Medium | Medium |
| 16 | Export Collada / SketchUp | `feat/export` | Medium | Low |
| 17 | Import GLB External | `feat/import` | High | Medium |
| 18 | Push/Pull Faces | `feat/direct-modeling` | High | High |
| 19 | Edge Offset Tool | `feat/direct-modeling` | Medium | Medium |
| 20 | First-Person Walk Mode | `feat/camera` | Medium | Medium |
| 21 | Construction Guide Lines | `feat/smart-snap` | Medium | Medium |
| 22 | Saved Camera Bookmarks | `feat/camera` | Low | Low |
| 23 | IFC Export | `feat/export` | Medium | High |
| 24 | PDF Floor Plan Export | `feat/export` | Medium | High |
| 25 | AI Room Layout Suggestions | `feat/ai` | High | Medium |
| 26 | Natural Language to Scene | `feat/ai` | High | High |

---

## Contribution Flow

```
main (local fork)
  └── feat/camera          → PR #1 → pascalorg/editor main
  └── feat/smart-snap      → PR #2 → pascalorg/editor main
  └── feat/dimensions      → PR #3 → pascalorg/editor main
  └── feat/measure-tool    → PR #4 → pascalorg/editor main
  └── feat/visual-quality  → PR #5 → pascalorg/editor main
  └── feat/direct-modeling → PR #6 → pascalorg/editor main
  └── feat/export          → PR #7 → pascalorg/editor main
  └── feat/import          → PR #8 → pascalorg/editor main
  └── feat/ai              → PR #9 → pascalorg/editor main
```

Each ticket = 1 branch = 1 isolated PR.
Tickets within the same epic and same PR label can be bundled if small (e.g., tickets 1+2+3 → single `feat/camera` PR).
