# urban-network-tools

QGIS tools for street network creation and urban morphology analysis.

## Focus
- **First priority: a clean, validated walkable network that any analysis can use.** The network is a product in its own right.
- **Analyses are open, not decided yet.** PST angular choice is used only to check that the network works for analysis.
- **Validation is central:** Level 1 checks after every step; next after the first working pipeline comes comparison with an official reference network (OSM vs NVDB), then the hand-checked test areas.
- Phase 2 (wrapping depthmapX/cityseer, comparing engines) is on hold.

## Read first
The docs in `docs/` are the source of truth for decisions. Read the relevant one before starting work.
- `docs/00-brief.md`: goals, focus, scope, licences, setup
- `docs/01-conventions.md`: data locations, CRS, core/buffer, test areas, engines
- `docs/03-phase1-pipeline.md`: the approved Phase 1 pipeline plan and progress log

Where the docs say `claude/<file>`, the file is `docs/<file>` here. `02-tool-inventory.md` is not in the repo yet.

**Master copies:** the brief and conventions are mastered in the user's Claude Project. Don't edit them here; replace them when the user provides a newer version. `docs/03-phase1-pipeline.md` is mastered here in the repo; the planning thread reads it on GitHub.

## Working with the user
- The user is an architect and new to the terminal, Python environments and git. Explain each step simply.
- Ask before installing software or changing files.
- **Never commit or push without asking first.**
- Record progress and new decisions in `docs/03-phase1-pipeline.md`, then commit and push it (after asking) so the planning thread can read it on GitHub.
- Build one pipeline step at a time and check with the user after each.

## Key rules
- **Data:** large data (downloads, runs, outputs) lives in `/Users/njasko/Documents/QGIS_Claude`, never in the repo. Only small test-area data goes in the repo. No spaces in file or folder names.
- **Runs** go to `/Users/njasko/Documents/QGIS_Claude/runs/<area>_<YYYYMMDD-HHMM>/`.
- **CRS:** each run uses the study area's local projected CRS in metres (Gothenburg: EPSG:3007). Reproject all inputs in the first step.
- **Core + buffer:** results are reported for the core only. Network-testing patches use a 500 m buffer.
- **Network:** walkable only (no motorways or ramps; footways and cycle paths included), one centre line per street, unlinks at bridges and tunnels.
- **Network representation stays as decided**, whatever analyses come later; thresholds (e.g. parallel-path distance) are logged parameters.
- **Output:** PST-ready GeoPackage (spec in `docs/03-phase1-pipeline.md`). The user runs PST in QGIS 3.42.1 as a check of the network.
- **Data sources:** open Europe-wide data by default (OSM). Nothing may depend on Swedish data; NVDB and other local data are optional references for validation.
- **Licences:** code GPL-3.0; OSM-derived data ODbL ("© OpenStreetMap contributors").

## Environment
- Python environment managed with **pixi** (conda-forge). `pixi.toml` and `pixi.lock` are committed; the `.pixi/` folder is not.
- Scripts run from Terminal through pixi, not inside QGIS's Python.
