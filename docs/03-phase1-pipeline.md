# Phase 1: network pipeline

Workstream thread: Phase 1 network pipeline. Last updated: 2026-11-25.
Status: **Steps 1–3 done; plan approved 2026-11-25. Step 4: stopped here.** Next: hub writes the Claude Code setup prompt; scripts are written and tested there, Olskroken first.

---

## Step 1: PST input requirements – DONE

Source: PST documentation v3.2.5 (2023-11-28); v3.3.1 notes (2024-11). User runs PST 3.3.2.

- Angular Choice runs **only on a segment map**, built by PST's **Create Segment Map** (Road Centre Line mode; snap, remove duplicates, remove zero-length, min tail, merge collinear; copies **one** column).
- **Level junctions = endpoints touch. Bridges/tunnels = lines cross without a node.** PST can auto-generate unlinks from such crossings (check by hand).
- **Unlinks:** point layer; each point removes the **closest** intersection. No tolerance documented → points exactly on crossings.
- Docs list Shapefile/TAB only; **GeoPackage works** on user's install.

### PST-ready output spec
| Item | Rule |
|---|---|
| File | GeoPackage per run: layers `network`, `unlinks`, `errors`, `core`, `buffer` |
| CRS | local projected CRS in metres (Gothenburg EPSG:3007) |
| Geometry | single-part LineString, 2D, no empty/invalid/zero-length |
| Topology | junction to junction; level junctions share exact endpoints; bridges/tunnels cross with no node |
| Unlinks | one point exactly on each grade-separated crossing |
| ID | one unique integer field `seg_id` (passed through Create Segment Map) |
| Other fields | for validation only: OSM way IDs, highway, bridge/tunnel/layer, source group |

### Decisions
- GeoPackage end to end.
- **Walkable network only:** no motorways/ramps; footways and cycle paths included.
- Angular Choice: **walking distance 1000, 2000, 5000 m.**
- **Hand-off:** pipeline → clean centre lines + unlinks + report + run log. User runs PST Create Segment Map (defaults, values recorded at first run) and Angular Choice.
- **Test patches for network testing: buffer 500 m** (added to conventions). Angular Choice results on these runs carry edge effects and are not interpreted.

## Step 2: how scripts run – DONE

- **Option C:** separate Python environment does download, cleaning, unlinks, validation and run log (run from Terminal); QGIS for viewing, hand-checking and PST.
- **Environment tool: pixi** (conda-forge; `pixi.toml` + `pixi.lock` in the repo record exact versions → run log). Set up in Claude Code.
- Reasons: QGIS 3.42–3.44 on macOS bundle Shapely 2.0.6, libpysal (neatnet dependency) needs ≥ 2.1.2 → clash; packages in QGIS's Python are lost on QGIS updates; neatnet recommends conda-forge.
- Level 1 checks run in Python (no dependency on `qgis_process` on the 3.42 macOS build). QGIS Check Geometry = optional manual cross-check.

---

## Step 3: pipeline plan – APPROVED (2026-11-25)

### Key findings shaping the plan (neatnet 0.1.6 docs + paper, 2025)
- neatify: input **must be projected, in metres**; **non-street paths such as sidewalks should be filtered out first**; other path types untested.
- neatnet does **not** handle bridges/tunnels explicitly; grade-separated interchanges may have their topology altered. It nodes lines that *end* on another line, not crossings.
- In raw OSM, level junctions share a node; crossings **without** a shared node are grade separations (or mapping errors) → read grade separations from raw OSM **before** cleaning, re-apply after.
- OSMnx's built-in "walk" filter is believed to exclude `highway=cycleway` (verify in coding). Many Swedish gång- och cykelvägar are `highway=cycleway` + `foot=designated` → use our own filter.
- neatnet parameters at defaults for the first run, all logged.

### Decisions in the plan
- **Parallel sidewalks/cycle tracks: tags + geometric test.** Drop ways tagged as sidewalk/crossing, plus paths running parallel within ~15 m of a street for most of their length. **Result: one centre line per street**; the street line stands for its sidewalks and cycle tracks. Paths further away (park paths, tracks behind a green strip or embankment) stay as their own lines – they are separate spaces. Threshold is a logged parameter; dropped lines exported for review.
- **Service roads:** drop driveways, parking aisles, drive-throughs; keep plain `service`.
- **Olskroken core:** approximate 1200 × 1200 m square drawn by Claude (user: "bigger rather than smaller"), EPSG:3007, x 149200–150400, y 6398950–6400150. Covers Olskroksmotet (E6/E20), the Olskroken district (centroid ≈ 150100, 6399630) and part of the rail yard. Placement based on Gothenburg City Museum street coordinates and Wikipedia; check visually in QGIS and adjust if needed. File: `olskroken_core.gpkg` (layer `core`) → goes into the repo as test data.

### Run setup
- **Area config** per study area (YAML in repo): core polygon, buffer width, CRS, OSM filter version.
- **Download area = core + buffer + 200 m margin**, so cleaning artefacts at the edge fall outside what we keep; clipped back to core + buffer at the end.
- **Run folder:** `/Users/njasko/Documents/QGIS_Claude/runs/<area>_<YYYYMMDD-HHMM>/` with one GeoPackage per step (`01_download.gpkg` …), final `network.gpkg`, `report.md`, `runlog.json`.
- **One command:** `pixi run pipeline --area olskroken`; each step can also run alone.

### Steps (each ends with Level 1 checks written to `report.md` + `errors` layer)
| # | Step | What happens | Tools | Checks after |
|---|---|---|---|---|
| 1 | Download | OSM ways in download area, custom walkable filter, tags (highway, footway, cycleway, foot, access, service, bridge, tunnel, layer, sidewalk, area); record OSM data timestamp | OSMnx (unsimplified graph → lines) | not empty, CRS, counts per highway type |
| 2 | Reproject | to EPSG:3007 (from config) | GeoPandas | CRS = run CRS, invalid/empty geometry |
| 3 | Grade separations (raw) | crossings without shared node in raw OSM, tagged with bridge/tunnel/layer evidence → `raw_unlinks` | Shapely/GeoPandas | untagged crossings listed (possible OSM errors) |
| 4 | Split into groups | **A streets** (walkable vehicle streets incl. pedestrian streets, plain service) · **B separate paths** (footway, cycleway, path, steps, bridleway not along a street) · **C dropped** (sidewalks, crossings, parallel tracks, driveways/parking aisles/drive-throughs, motorways/ramps, foot=no, private) | rules + parallel test | counts/length per group; C exported |
| 5 | Clean streets | neatify group A | neatnet | FaceArtifacts leftovers, degree-2 nodes, duplicates, tiny segments |
| 6 | Re-attach paths | add group B; extend dangles to nearby lines (gap tolerance ~3 m, logged); node level crossings; skip crossings near `raw_unlinks` | neatnet `extend_lines`/`close_gaps`, momepy | gaps, unconnected parts |
| 7 | Final topology | remove degree-2 nodes, duplicates, zero-length; single-part 2D | momepy, neatnet `fix_topology` | full Level 1 list |
| 8 | Unlinks | point at every unnoded crossing; evidence from step 3; no evidence → `errors` | Shapely | every unnoded crossing has an unlink or an error |
| 9 | Clip + output | clip to core + buffer, `seg_id`, write `network.gpkg` | GeoPandas | unique IDs, fields, CRS |
| 10 | Report + run log | date, OSM timestamp, area, filter, parameters, package versions (pixi.lock), counts, durations | own code | — |

**Level 1 street-network checks:** invalid/empty geometry · duplicates · partial overlaps · gaps · crossings without node (→ unlinks) · unconnected parts (count + length share of largest) · tiny segments · leftover dual carriageways/roundabouts (FaceArtifacts) · degree-2 nodes · attributes · CRS.

### Manual steps (user, in QGIS)
1. Open `network.gpkg`; review `errors` and dropped group C.
2. PST Create Segment Map (Road Centre Line, defaults, unlinks layer, copy `seg_id`) → Angular Choice 1k/2k/5k walking distance.
3. Optional: PST's auto-generated unlinks vs ours.
4. Later (gap 3): hand-corrected copy of the Olskroken core = gold standard; automatic comparison every run.

### Main risks
- **Step 6** (re-attaching paths after neatnet moved street lines) – hardest part; Olskroken will show.
- Parallel-path test can drop real paths or keep duplicates → group C exported for review; threshold tunable.
- Pedestrian squares mapped as areas are skipped → gaps on squares (matters for Inom Vallgraven later).

## Step 4: STOP
Plan approved. No code written here. Next: Claude Code setup (prompt from the hub), then implement steps 1–10 for Olskroken.
