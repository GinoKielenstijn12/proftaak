# Plan: Photoshop-style image editor (local demo)

Goal: a working, locally-run image editor that demonstrates the core of what
makes Photoshop Photoshop — **layers, selection, brush tools, adjustments and
non-destructive history** — in a scope that a small team can finish and demo.

This is a *demo*, not a full clone. Everything below is ordered so that the
app is demoable after every phase; later phases are optional extras.

---

## 1. Tech stack

| Concern | Choice | Why |
|---|---|---|
| Language | **TypeScript** | Type safety for the document/layer model, good tooling |
| UI | **React + Vite** | Fast dev server, simple local run (`npm run dev`) |
| Rendering | **HTML5 Canvas 2D** (one offscreen canvas per layer) | Built into every browser, no install, enough for a demo |
| Heavy pixel ops | Plain `ImageData` loops, moved to a **Web Worker** when slow | Keeps UI responsive for filters like blur |
| State | **Zustand** (small store) | Simple global state for tools, layers, history |
| Local "desktop" app (optional) | **Tauri** or **Electron** wrapper | Native window + real file open/save dialogs |
| Tests | **Vitest** (logic) + **Playwright** (smoke test of UI) | Pixel/blend math is easy to unit-test |

Runs fully offline on `localhost` — no server, no accounts, no cloud.

Alternative if the team prefers Python: **PySide6 (Qt) + NumPy + Pillow**.
Same architecture applies; replace Canvas with `QImage`/NumPy arrays.

---

## 2. Architecture

```
src/
  core/                 # framework-free, unit-tested
    document.ts         # Document { width, height, layers[], activeLayerId, selection }
    layer.ts            # Layer { id, name, canvas, visible, opacity, blendMode, locked }
    compositor.ts       # flattens visible layers -> display canvas (blend modes, opacity)
    selection.ts        # selection mask (Uint8 alpha mask) + marching ants path
    history.ts          # undo/redo stack of commands (snapshot per changed layer region)
    blend.ts            # normal, multiply, screen, overlay, darken, lighten
    filters/            # brightness/contrast, hue/sat, invert, grayscale, blur, sharpen
    io.ts               # open PNG/JPG, export PNG/JPG, save/load project (.pdemo = zip/json)
  tools/                # each tool implements a common interface
    Tool.ts             # onPointerDown/Move/Up, cursor, options panel
    MoveTool.ts  BrushTool.ts  EraserTool.ts  RectSelectTool.ts
    LassoTool.ts MagicWandTool.ts FillTool.ts EyedropperTool.ts
    TextTool.ts  CropTool.ts  ZoomTool.ts  HandTool.ts
  ui/
    App.tsx             # layout: menu bar, toolbar (left), canvas (center), panels (right)
    CanvasView.tsx      # viewport: zoom/pan, pointer -> document coordinates, overlays
    Toolbar.tsx  OptionsBar.tsx  LayersPanel.tsx  HistoryPanel.tsx
    ColorPicker.tsx  AdjustmentsDialog.tsx  MenuBar.tsx
  workers/filters.worker.ts
```

Key design rules:
- **Core is UI-independent.** Tools mutate the document through commands;
  React only renders state. This keeps undo/redo and tests simple.
- **Every edit is a Command** (`do()` / `undo()`), pushed to `history.ts`.
  Store only the dirty rectangle of the affected layer, not the whole image.
- **Compositor re-renders only on change**, using `requestAnimationFrame`.
- **Viewport transform** (zoom + pan) is separate from document pixels.

---

## 3. Feature scope

### Must have (the demo)
- New document / open image (PNG, JPG) / drag-and-drop
- Export to PNG / JPG
- Zoom (wheel, Ctrl +/-), pan (space + drag), fit to screen
- Layers panel: add, delete, duplicate, reorder (drag), rename, visibility,
  opacity, blend mode, merge down, flatten
- Tools: Move, Brush (size, hardness, opacity), Eraser, Rectangle select,
  Eyedropper, Paint bucket, Zoom, Hand
- Foreground/background color with color picker
- Undo/redo (Ctrl+Z / Ctrl+Shift+Z) + History panel
- Adjustments: Brightness/Contrast, Hue/Saturation, Invert, Grayscale
- Keyboard shortcuts matching Photoshop (B, E, V, M, I, G, Z, H, X, D, [ ])

### Should have
- Lasso + Magic wand selection, selection add/subtract, invert, feather
- Filters: Gaussian blur, Sharpen (in a Web Worker)
- Crop tool, Image > Resize / Canvas size, rotate/flip
- Text tool (rasterized on commit)
- Save/load project file that keeps layers (`.pdemo`)

### Could have (stretch)
- Layer masks, adjustment layers (non-destructive)
- Transform (scale/rotate) with handles
- Clone stamp / healing (simple version)
- Shapes (rectangle, ellipse), gradient tool
- Pressure-sensitive brush (Pointer Events `pressure`)
- WebGL compositor for large images
- Desktop packaging via Tauri

### Out of scope
PSD import/export, CMYK, 16/32-bit color, smart objects, plugins, AI features.

---

## 4. Phased roadmap

Each phase ends with something you can show.

| Phase | Deliverable | Est. |
|---|---|---|
| **0. Setup** | Vite + React + TS repo, lint/format, Vitest, CI, empty layout (menu, toolbar, canvas, panels) | 1–2 days |
| **1. Canvas & viewport** | Open image, display it, zoom/pan, export PNG | 2–3 days |
| **2. Layers** | Document/layer model, compositor with opacity + blend modes, Layers panel | 3–4 days |
| **3. Painting** | Brush, eraser, color picker, eyedropper, fill; smooth strokes (interpolated stamps) | 3–4 days |
| **4. History** | Command pattern, undo/redo, History panel | 2 days |
| **5. Selection** | Rect select, marching ants, painting clipped to selection, copy/paste/delete | 3–4 days |
| **6. Adjustments & filters** | Dialogs with live preview; blur/sharpen in a worker | 3 days |
| **7. Polish for demo** | Shortcuts, cursors, tooltips, dark theme, sample images, project save/load | 2–3 days |
| **8. Stretch** | Pick from "Could have" as time allows | rest |

Total for the must-have demo: roughly **3–4 weeks** for 1–2 people.

---

## 5. Technical notes / risks

- **Brush quality:** draw by stamping a soft circle every `size * 0.25` px
  along the pointer path; interpolate between pointer events so fast strokes
  don't leave gaps. Draw the stroke on a temporary layer, then commit with the
  brush opacity so overlapping stamps don't build up.
- **Performance:** keep images ≤ ~4000×4000 for the demo. Only recomposite
  the dirty rect. Move per-pixel filters into a Web Worker with transferable
  `ImageData` buffers.
- **Memory for undo:** store dirty-rect snapshots and cap history (e.g. 50 steps).
- **Blend modes:** Canvas 2D `globalCompositeOperation` already supports
  multiply, screen, overlay, darken, lighten etc. — use those instead of
  writing them by hand.
- **Coordinates:** one helper converts screen → document coordinates using
  the viewport transform; every tool uses it.
- **Selection:** represent as an 8-bit mask the size of the document; use it
  as a clip when painting and as alpha when copying.

---

## 6. Demo script (≈5 min)

1. Open a photo (drag and drop).
2. Add a new layer, paint on it with the brush, change blend mode to Multiply,
   lower opacity — show the result live.
3. Select a region with the rectangle / magic wand, apply Hue/Saturation only
   to that region.
4. Apply a Gaussian blur to the background layer.
5. Undo a few steps via the History panel, redo.
6. Add text, reorder layers, toggle visibility.
7. Export as PNG and open it in the OS image viewer.

---

## 7. How to run (once phase 0 is done)

```bash
npm install
npm run dev      # opens http://localhost:5173
npm test         # unit tests
npm run build    # static build in dist/ — open locally or wrap with Tauri
```

---

## 8. Definition of done (demo)

- All "Must have" features work in Chrome/Edge/Firefox on a laptop.
- No crash or visible lag on a 3000×2000 photo during the demo script.
- Core modules (compositor, blend, history, selection, filters) have unit tests.
- README explains how to install and run locally.
