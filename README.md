# Figma Design Extractor

A Figma plugin, written in TypeScript on the Figma Plugin API, that turns a selected frame into a design reference package that code agents and developers can follow: layout, raw design tokens, content, images and icons, plus an `AGENTS.md` build guide.

I built it for my own design-to-code workflow and use it every day.

> **Source is private.** This repository documents the problem, the architecture and the technical decisions. The code is available for technical review on request.

<img width="962" height="523" alt="image" src="https://github.com/user-attachments/assets/7380f63f-367d-4b4a-829f-7a0d7255d6ae" />
<img width="956" height="588" alt="image" src="https://github.com/user-attachments/assets/d405b33a-21c9-4624-9c5a-e3018c276227" />

## The problem

Building a page from a Figma file used to mean copying values by hand: colors, font sizes, spacing, radii, shadows, image crops. Every hand-copied value is a chance for the design and the theme to drift apart. AI coding agents make this worse when they don't have exact values: they estimate from a screenshot and invent the rest.

The goal: **give the developer, or the agent, every value already resolved, so nothing has to be guessed.**

## What it does

Select one frame (for example, a home page) and run the plugin. It:

1. **Runs a pre-flight check**: counts nodes and flags text with missing fonts, because a missing font silently breaks the reference screenshot.
2. **Detects sections automatically** from the layer structure, in visual order (top to bottom).
3. **Extracts each section**: layout, styles, text content and the image and vector assets as real files.
4. **Aggregates raw design tokens** for the whole frame: colors, typography, spacing, radii and effects, ranked by how often they are used.
5. **Writes an `AGENTS.md`** filled in with the data from this export, telling any code agent how to rebuild the layout faithfully in any stack.
6. **Packs everything into a ZIP**: one folder per section, plus `global/`.

## Output

```
home-design-reference.zip
├── AGENTS.md      build guide, generated from this export's data
├── global/        frame-wide tokens, components, section index, full screenshot
└── <section>/     layout, content, assets and a screenshot for each section
```

## How it works

```
Figma main thread (code.ts)                 UI iframe (ui.html)
───────────────────────────                 ───────────────────
pre-flight -> section detection
-> recursive extraction per section   --->  receives each file as a message
-> token aggregation                        (JSON text or binary PNG/SVG)
-> AGENTS.md generation               --->  builds the ZIP with JSZip
                                            -> download
```

- **Two contexts, one message channel.** The plugin's main thread has access to the document but no DOM. The UI iframe has a DOM but no document access. The main thread streams every file to the UI through `postMessage`, and the UI assembles the ZIP.
- **No network.** `manifest.json` declares `networkAccess: none`. The build step bundles JSZip into `ui.html`, so the plugin works offline, and no design data ever leaves the machine.
- **Build:** esbuild bundles `src/*.ts` into `code.js` and injects JSZip into the UI template. `tsc --noEmit` runs in strict mode for type checking.

## Technical decisions and trade-offs

- **Values are resolved per node, never read from variables or styles.** Imported `.fig` files often arrive with broken library links. Reading resolved values survives that. Trade-off: the export does not reproduce the designer's token names.
- **Tokens get neutral names (`color-01`, `space-03`).** The plugin doesn't guess semantics like "brand" or "surface". Naming belongs to the project's own conventions. Tokens are sorted by usage, so the most important values come first.
- **Likely designer slips are flagged, not fixed.** Values that look like inconsistencies in the scale are marked for review. The plugin never rewrites the design. The final decision stays with a human.
- **Every heuristic is labeled.** Asset classification, gradient angle, crop focal point and content hints all carry `"inferred": true`, and the raw value always ships alongside them. When the two conflict, the raw data wins.
- **Assets are classified, not exported blindly.** Each visual element is classified as a raster image, a vector, or something CSS can reproduce, so the agent references real files and never redraws an asset or uses a placeholder.
- **The CSS from Figma is used only as a cross-check.** `getCSSAsync()` output is stored per node, within a budget so large frames stay fast. Differences between it and the raw data point to special cases such as rotation, crop or absolute positioning.
- **Performance hints at the source.** The largest image in the first section is flagged as the LCP candidate, so it can be loaded with high priority. Decorative assets are flagged so they get an empty `alt` or become a CSS background.

## Limitations

- Runs as a development plugin in Figma Desktop only, on files with edit permission. It is not published to the Figma Community.
- Exports one breakpoint, the selected frame (usually desktop). Responsive behavior is derived from constraints and the auto-layout structure.
- Section detection and asset classification are heuristics. Unusual layer structures can fall back to single-block mode or misclassify an asset, and the export flags when that happens.
- Components are recorded shallowly (name, size, variants). Main components are not traversed again.

## Next steps

- Multiple frames in one export (desktop + mobile), to map breakpoints explicitly
- Optional semantic naming pass driven by a project config file

## Stack

TypeScript (strict) · Figma Plugin API · esbuild · JSZip

