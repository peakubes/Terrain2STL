# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Terrain2STL is a web service that converts SRTM HGT elevation data files into downloadable STL files for 3D printing. It has two main components:
1. A C program (`celevstl`) that reads HGT files and writes binary STL geometry
2. A Node.js/Express server (`terrainServer.js`) serving a Leaflet-based map UI

## Commands

### Build the C STL generator
```sh
make
```
This compiles `src/elevstl.c`, `src/STLWriter.c`, `src/elevation.c` into `./celevstl`.

### Test the C binary directly
```sh
./celevstl <lat> <lng> <width> <height> <vScale> <rotation_deg> <waterDrop_mm> <baseHeight_mm> <stepSize> <output.stl>
# Example (Rockport harbor):
./celevstl 44.1928 -69.0851 40 40 1.7 0 1 3 1 test.stl
```

### Install Node.js dependencies
```sh
npm install
```

### Run the web server
```sh
node terrainServer.js          # with static file serving (default, port 8080)
NOSTATIC=true node terrainServer.js  # without static serving (for use behind NGINX)
PORT=3000 node terrainServer.js      # on a custom port
```

The server must have `stls/` and `logs/` directories present before starting (it writes generated files there).

## Architecture

### Data flow
1. Browser UI (`terrain2stl.html` + `terrain2stl.js`) sends a POST to `/gen` with model parameters
2. `terrainServer.js` queues the request (concurrency: 2, timeout: 20s) and runs `./celevstl` via `child_process.exec`
3. `celevstl` reads `.hgt` files from `./hgt_files/`, generates a `.stl`, which is then zipped
4. Server responds with the file number; browser constructs the download URL (`stls/terrain-<N>.zip`)

### C program (`src/`)
- `elevstl.c` — entry point; parses CLI args, iterates elevation lines, builds watertight STL geometry (top surface strips, four walls, flat bottom)
- `elevation.c` / `elevation.h` — reads and interpolates HGT tile data, handles tile-crossing and void filling
- `STLWriter.c` / `STLWriter.h` — low-level binary STL output helpers

### Web frontend
- **Map**: Leaflet with a MapLibre GL layer using OpenFreeMap's "liberty" style, plus a GeoRaster COG hillshade overlay from a Linode object store
- **Selection box**: A draggable Leaflet polygon with rotation support (via `Path.Drag.js`)
- **URL params**: `ingestURLParams()` reads form state from query string; `createShareableURL()` generates shareable links
- The form POST goes to `/gen` (not `/`); `terrain2stl.js` uses XHR to submit and handles the download button response

### HGT data
- Place `.hgt` SRTM files in `./hgt_files/`. The full dataset is ~40GB; only sample files are included.
- `hgt_files/` can be a symlink to an external directory.

### Configuration (`config.js`)
Controls request and parameter logging; log files go to `logs/requests.log` and `logs/params.log`.
