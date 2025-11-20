# A* Visualization

This document gives a quick overview of the repository structure and explains how to run the Vue front-end.

## Folder structure

- `Algorithm/`: Contains documentation and raw code stubs for routing algorithms. Inside you will find the `Astar/` and `Dijkstra/` folders (each currently holding a sample `main.py`) plus an `Algorithm.txt` note that outlines the algorithm requirements in Vietnamese. Use this area to prototype, benchmark, or tweak algorithm variants before wiring them into the UI.
- `app/`: Vue 3 + Vite application used to visualize the algorithms. It includes the source code in `src/`, Vite/Tailwind configuration, static assets under `public/`, and components for rendering maps (Leaflet), managing state (Pinia), and handling routing.

## Set up and run the Vue app

1. **Install system prerequisites**
   - Node.js `^20.19.0` or `>=22.12.0` (see the `engines` field in `app/package.json`).
   - npm (bundled with Node) to execute the Vite scripts.

2. **Install dependencies**

   ```bash
   cd app
   npm install
   ```

3. **Start the development server**

   ```bash
   npm run dev
   ```

   Vite prints a URL (default `http://localhost:5173`). Open it in your browser to view the visualization with hot-reload feedback.

4. **Build & preview production output**

   ```bash
   npm run build      # produces the production bundle in dist/
   npm run preview    # serves the freshly built bundle
   ```

5. **Run type checking (optional)**

   ```bash
   npm run type-check
   ```

After this setup, continue refining algorithms in `Algorithm/` and mirror the logic inside `app/src/` to present them visually.

## Update the bundled OSM map

The Vue app bundles `app/map.osm` (imported via `?raw`) so every grid/walkable check runs against that file. To swap in your own map:

1. Go to [https://www.openstreetmap.org/export](https://www.openstreetmap.org/export), select the bounding box (drag the rectangle or submit coordinates) and hit **Export** to download the `.osm` file.
2. Rename the exported file to `map.osm` and replace `app/map.osm` in this repo.
3. Restart `npm run dev` (or rebuild with `npm run build`).

Whenever you replace `map.osm`, the walkable dataset and overlay follow that file exactly. Share these instructions with others so they can drop in their own `.osm` export and see their custom map rendered.
