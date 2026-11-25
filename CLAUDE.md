# urban-network-tools

QGIS tools for street network creation and urban morphology analysis. Space syntax first (main measure: angular choice).

## Read first
The docs in `docs/` are the source of truth for decisions. Read the relevant one before starting work.
- `docs/00-brief.md`: goals, scope, licences, setup
- `docs/01-conventions.md`: data locations, CRS, core/buffer, test areas, engines
- `docs/03-phase1-pipeline.md`: the approved Phase 1 pipeline plan and progress log

The docs come from the user's Claude Project. Where they say `claude/<file>`, the file is `docs/<file>` here. `02-tool-inventory.md` is not in the repo yet.

## Working with the user
- The user is an architect and new to the terminal, Python environments and git. Explain each step simply.
- Ask before installing software or changing files.
- **Never commit or push without asking first.**
- Record progress and new decisions in `docs/03-phase1-pipeline.md`.
- Build one pipeline step at a time and check with the user after each.

## Key rules
- **Data:** large data (downloads, runs, outputs) lives in `/Users/njasko/Documents/QGIS_Claude`, never in the repo. Only small test-area data goes in the repo. No spaces in file or folder names.
- **Runs** go to `/Users/njasko/Documents/QGIS_Claude/runs/<area>_<YYYYMMDD-HHMM>/`.
- **CRS:** each run uses the study area's local projected CRS in metres (Gothenburg: EPSG:3007). Reproject all inputs in the first step.
- **Core + buffer:** results are reported for the core only. Network-testing patches use a 500 m buffer.
- **Network:** walkable only (no motorways or ramps; footways and cycle paths included), one centre line per street, unlinks at bridges and tunnels.
- **Output:** PST-ready GeoPackage (spec in `docs/03-phase1-pipeline.md`). The user runs PST in QGIS 3.42.1.
- **Data sources:** open Europe-wide data by default (OSM). Nothing may depend on Swedish data.
- **Licences:** code GPL-3.0; OSM-derived data ODbL ("© OpenStreetMap contributors").

## Environment
- Python environment managed with **pixi** (conda-forge). `pixi.toml` and `pixi.lock` are committed; the `.pixi/` folder is not.
- Scripts run from Terminal through pixi, not inside QGIS's Python.
