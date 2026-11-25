# Conventions

Last updated: 2026-11-25

Rules every script, run and thread follows. Add to this doc when a new convention is decided.

## Where data lives
- **Large working data** (downloads, city-scale runs, outputs) lives in a local folder on my Mac: `/Users/njasko/Documents/QGIS_Claude`. Not in GitHub, not in a cloud-synced folder (sync can corrupt GeoPackages and stall scripts).
- **Only small test-area data** goes into the GitHub repo.
- Avoid spaces in folder and file names (command-line tools handle them badly).

## Coordinate system
- **Each run uses the local projected coordinate system of the study area.** Example: Gothenburg → EPSG:3007 (SWEREF 99 12 00). Other places use their own local zone or projection.
- **All inputs are reprojected to that system in the first step** of a run (e.g. NVDB and Lantmäteriet data come in EPSG:3006; European layers often in EPSG:3035).
- **Within one run, all layers share the same coordinate system.** Validation (Level 1) checks this before any calculation.

## Study area: core and buffer (edge effect)
- **Every run has a core area and a buffer around it.** The network in the buffer is included in the analysis so routes can pass through it, but results are **reported only for the core**.
- **Buffer width:** at least as wide as the largest analysis radius used. For whole-network measures without a radius limit, the buffer covers the connected metropolitan network.
- **Exception – test patches for network testing:** when a test patch is used only to build and check the network (no analysis results interpreted), the buffer is **500 m**. Analysis runs on test patches use the full rule above.
- **Validation:** the core is checked in full; the buffer gets Level 1 checks only.

## Test areas (Gothenburg test case)
- **Hand-checked patches** (gold standard, each roughly 600–800 m square, analysed with their own buffer):
  1. **Inom Vallgraven:** dense historic grid, pedestrian streets, canal bridges.
  2. **Olskroken incl. Olskroksmotet:** motorway ramps, dual carriageways, railway, tunnels and bridges (hardest case for cleaning). Core drawn 2026-11-25 as an approximate 1200 × 1200 m square (EPSG:3007: x 149200–150400, y 6398950–6400150), larger than the others on purpose; see claude/03-phase1-pipeline.md.
  3. **Backa:** modernist suburb with separated pedestrian/cycle paths and underpasses (hardest case for path completeness).
  - Possible later addition: Johanneberg (campus paths, semi-private areas, stairs).
- **Large area (city scale):** core = **Gothenburg municipality**; buffer = surrounding network in neighbouring municipalities, **including Mölndal** and Partille.

## QGIS version
- **Development and testing happen on QGIS 3.42.1** (my installed version, with PST), on macOS.
- Published tools aim to work on **QGIS 3.44 LTR and 4.x** as well; tools from the inventory are still checked for both. Before publishing a tool, note which versions it has actually been tested on.
- If a tool we rely on (e.g. PST) doesn't run on QGIS 4.x, that step stays on 3.x until the tool supports 4.x.

## Analysis engines
- **PST** is the baseline space syntax tool.
- **depthmapX is the reference engine** for checking results (most widely used in space syntax research, so results are comparable with published studies). Record the depthmapX version with every run, since 0.9.1 changed some calculations.
- **cityseer is the second engine**, for city-scale runs and later for densities and functions. Its measures and radius types differ from classic space syntax, so results are compared with care, not expected to match one-to-one.
