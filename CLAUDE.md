# GeoSpatial Studio

A single-page web application for geospatial polygon and point analysis, built on Google Maps. Users load GeoJSON files, draw/edit/move polygons, manage points, run point-in-polygon matching, and export results.

## Project Structure

```
index.html                    # Entire application (HTML + CSS + JS in one file)
.github/workflows/static.yml  # GitHub Pages deployment (triggers on push to main)
```

This is a **zero-build, single-file** application. There is no package manager, bundler, framework, or test suite. All logic lives in `index.html`.

## Tech Stack

- **Google Maps JavaScript API** — map rendering, drawing, geocoding (API key is embedded in the HTML)
- **Turf.js 6.5.0** (CDN) — geospatial calculations (area, point-in-polygon, simplification, centroid)
- **Tailwind CSS** (CDN) — utility-first styling
- **Vanilla JavaScript** — no framework, all state in global variables

## Architecture

### State Management

All application state is held in global variables at the top of the `<script>` block (~line 288):

- `polygons[]` — array of `google.maps.Polygon` objects, each with a `.meta` property storing `{ id, areaSqFt, centerLat, centerLng }`
- `points[]` — array of `google.maps.Marker` objects, each with `.meta` storing `{ id, original: {lat, lng}, moved }`
- `polygonFileLayers{}` — registry of loaded polygon files keyed by `fileId`, tracks per-file polygons and bounds
- Boolean flags: `editingPolygons`, `editingPoints`, `movingPolygons`, `drawingMode`, `showCenterPoints`

### Key Functional Areas

| Area | Functions | Lines |
|------|-----------|-------|
| Area calculation | `calculatePolygonArea()`, `formatAreaDisplay()` | ~318–331 |
| Center markers | `calculateAndShowCenter()`, `removeCenterMarker()`, `toggleCenterPoints()` | ~336–387 |
| Reverse geocoding | `reverseGeocodeWithThrottle()` (throttled + cached) | ~392–409 |
| Point tooltips | `attachPointTooltip()`, sticky tooltip with copy buttons | ~450–498 |
| Polygon tooltips | `attachAreaTooltip()` — shows area on hover | ~503–522 |
| Map init | `initMap()` — sets up map, controls, keyboard listeners, context menu | ~527–767 |
| Drawing | `initDrawingManager()`, `toggleDrawPolygon()` | ~772–807 |
| File loading | `readPointFile()`, `handleMultiplePolygonFiles()`, `createPolygonsFromGeoJSON()` | ~812–937 |
| Layer management | `addPolygonFileControl()` — per-file toggle/zoom/unload UI | ~939–1024 |
| Editing | `togglePolygonEditing()`, `togglePointEditing()`, `togglePolygonMoving()` | ~1115–1182 |
| Polygon dragging | `attachPolygonDragListeners()`, `setupPolygonDragOnMap()` | ~1183–1293 |
| Vertex deletion | `attachPolygonVertexDeleteListener()` — right-click to delete vertex | ~1198–1253 |
| Processing | `processData()` — point-in-polygon matching | ~1298–1387 |
| Export | `exportPolygonCSV()`, `exportPointCSV()`, `exportPolygonWKTCSV()`, `exportPolygonGeoJSON()` | ~1417–1482 |
| Search | `searchLocation()` — address geocoding, click pin to add as point | ~1487–1525 |
| Simplification | Douglas-Peucker (via Turf.js) and Visvalingam-Whyatt (custom impl) | ~1530–1801 |

### UI Layout

- **Horizontal toolbar** (top) — two rows: status/data/tools on row 1, search/process/export on row 2
- **Google Map** — fills remaining viewport
- **Floating results panel** — glass-morphism panel on the right, shows after processing
- **Context menu** — right-click on map to add a point
- **Simplify modal** — modal dialog for polygon simplification settings

## Development Workflow

### Running Locally

Open `index.html` directly in a browser. No server required (though a local server avoids CORS issues with some browser features):

```bash
python3 -m http.server 8000
# or
npx serve .
```

### Deployment

Pushes to `main` auto-deploy to GitHub Pages via the `static.yml` workflow. The workflow uploads the entire repo root as a Pages artifact.

### Making Changes

1. Edit `index.html` directly
2. Refresh browser to test
3. Commit and push to `main` to deploy

## Conventions

- **No build step** — all changes are made directly in `index.html`
- **Global state** — polygon/point state uses global variables, not classes or modules
- **Google Maps patterns** — polygons and markers carry a `.meta` object for app-specific data; use `google.maps.event.addListener()` for map events
- **Turf.js coordinate order** — Turf uses `[lng, lat]`; Google Maps uses `{lat, lng}` — coordinate conversion happens at every boundary
- **Area units** — all areas are in square feet (converted from sq meters via `* 10.7639`)
- **IDs** — `nextPolygonId` and `nextFileLayerId` are monotonically incrementing counters
- **Polygon metadata** — always update `meta.areaSqFt` and center after modifying a polygon's path
- **File input pattern** — hidden `<input type="file">` triggered by a styled button; input value is reset after read to allow re-selecting the same file

## Gotchas

- The Google Maps API key is embedded directly in the HTML (~line 283). It requires the Geometry, Places, and Drawing libraries.
- Turf.js polygon rings must be closed (first point == last point). Many functions manually close the ring before passing to Turf.
- Right-click behavior changes by mode: in edit-polygons mode it deletes vertices; otherwise it shows the context menu to add a point.
- The `suppressAreaTooltip` flag prevents area tooltips during editing/moving to avoid visual clutter.
- Polygon simplification uses two algorithms: Douglas-Peucker (Turf.js built-in) and Visvalingam-Whyatt (custom implementation starting ~line 1533).
