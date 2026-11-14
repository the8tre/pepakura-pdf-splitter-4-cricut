# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the app

No build step. Serve `index.html` with any static file server from the project root:

```bash
python3 -m http.server 8744
# then open http://localhost:8744
```

To verify JS syntax without running a browser:

```bash
python3 -c "
import re
content = open('index.html').read()
scripts = re.findall(r'<script(?:\s[^>]*)?>(.*?)</script>', content, re.DOTALL)
open('/tmp/chk.js','w').write(scripts[-1])
" && node --check /tmp/chk.js
```

## Architecture

Single self-contained file: `index.html` (HTML + CSS + JS, ~730 lines). No dependencies, no build tooling. `pdfjs-dist` 6.x is loaded from CDN at runtime as an ESM module and exposed on `window.pdfjsLib`.

### Input

PDFs exported from **Pepakura Designer** via File → Print to PDF… with "Print lines clearly (Vector print)" selected. Each page contains a flat bag of 2-point stroked segments — one `moveTo + lineTo` per edge, no topology or piece grouping.

### Processing pipeline

```
processPDF(file)
  → pdfToSegments(doc, pageNum)      # walk pdfjs OPS, classify + Y-flip + pt→mm
  → segmentsToSVG(segs)              # deduplicate, stitch folds, emit class-tagged SVG
  → split(filename, svgText)         # merge, size, preview, download
```

#### pdfToSegments
Walks `page.getOperatorList()`. Key facts:
- Coordinates are in **`args[1]`** (mini-ops array, objects `{0:op,1:x,2:y,...}`). **Do not use `args[2]`** (Float32Array) — it holds a different representation and breaks vertex sharing.
- `OPS.setDash` precedes every path; track the last value. Empty `[]` → **cut**, any non-empty → **fold**.
- Apply Y-flip (`H - y`) and `PT_TO_MM = 25.4/72` to convert points → mm.

#### segmentsToSVG
- **Cut**: raw deduped segments emitted as individual `M x y L x y` subpaths, `class="outer"`, stroked unfilled. Stitching into filled polygons was tried and fails (flat bag of segments with no piece grouping produces holes and cross-net diagonals).
- **Fold**: stitched via `stitchSegments()` (angle-based junction walk) into polylines, then split back to per-edge `M L` subpaths via `polylineToEdges()` — the deboss merger regex only captures one `L` per `M`, so continuous polylines would lose edges.
- **Fold wins over coincident cut**: edges drawn as both solid and dashed are dropped from cut (0.5mm match grid).

#### split(filename, svgText)
Receives a synthesized SVG with `path.outer` (cut) and `path.convex` (fold). Grabs `d` attrs by `querySelector`, calls `mergeCutNodes` / `mergeDebossNodes`, builds cut/deboss/cricut SVGs, wires previews and download.

### Cricut SVG sizing (critical)

Cricut Design Space ignores `width`/`height` and fits to the bounding box. **Do not use full-page viewBox.**

- `viewBox` = tight path bounding box (`minX minY w h`)
- `width`/`height` in **inches**: `(w/25.4).toFixed(4) + 'in'`

### UI

- Drop-zone → page selector (shown only for multi-page PDFs) → two-column results
- Left: ORIGINAL preview (the synthesized SVG with both layers styled)
- Right: CRICUT preview with **Both / Cut / Deboss** radio toggle, Download button
- State: `_pdfState` holds the loaded pdfjs doc; `_previewSVGs` holds `{cut, deboss, both}` SVG strings for the toggle

### Key functions

| Function | Purpose |
|---|---|
| `pdfToSegments(doc, pageNum)` | Extract cut/fold segments from pdfjs operator list |
| `stitchSegments(segs, epsilon)` | Chain 2-point edges into polylines via angle-based endpoint graph walk |
| `segmentsToSVG(segs)` | Deduplicate, stitch folds, emit class-tagged SVG for `split()` |
| `mergeCutNodes(cutD)` | Wrap cut d-string in stroked path, compute bbox |
| `mergeDebossNodes(foldD)` | Deduplicate M…L segments, merge into one fold path |
| `buildCricutSVG(…)` | Combine cut + fold into one SVG with tight union bbox |
| `pathBBox(dAttr)` | Compute bounding box from path d-attribute |
| `renderSelectedLayer()` | Swap cricut preview between cut/deboss/both based on radio state |
