# Node Pathway Visualizer

A hybrid 3D fractal tree visualizer built as a single HTML file using Three.js. Procedurally generates tree/branch structures using multiple algorithms, with full customization via lil-gui.

![Demo](demo.gif)

## Features

- **3 Generation Algorithms** — Recursive subdivision, L-system (botanical), and space colonization
- **Collision/Spacing System** — Fidenza-style ribbon avoidance prevents branch overlap with configurable minimum spacing and deflection
- **Canopy Shaping** — Bounding envelopes (sphere, ellipsoid, cone, cylinder) control the overall tree silhouette with soft falloff
- **High-Res Export** — Export PNG at 2x/4x/8x resolution for print, with optional transparent background
- **Full Customization** — Node shape/size/color, edge thickness/curvature, lighting, bloom, fog
- **Interaction** — Orbit/zoom/pan camera, hover tooltips, click to highlight connections
- **JSON Import/Export** — Save and restore configurations and graph data

## Usage

Open `index.html` in any modern browser. No build step, no dependencies to install.

```
open index.html
```

## Controls

| Panel | What it does |
|---|---|
| **Tree Structure** | Algorithm, depth, branching angle, length ratio, children count, jitter |
| **Spacing** | Collision avoidance toggle, minimum spacing, deflection strength |
| **Canopy** | Bounding shape, dimensions, softness, wireframe preview |
| **Nodes** | Shape, size, color, opacity, glow, color-by-depth gradient |
| **Pathways** | Edge thickness, color, curvature, opacity |
| **Environment** | Background, lighting, fog, bloom |
| **Import / Export** | JSON save/load, hi-res PNG export |

## Tech Stack

- [Three.js](https://threejs.org/) r170 — WebGL rendering
- [lil-gui](https://lil-gui.georgealways.com/) — GUI controls
- ES modules via CDN — zero build step
