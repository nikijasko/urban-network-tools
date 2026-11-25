# Phase 1: network pipeline

Workstream thread: Phase 1 network pipeline. Last updated: 2026-11-25.
Status: **Steps 1–3 done; plan approved 2026-11-25. Step 4: Claude Code setup done (2026-11-25).** Next: build pipeline step 1 (download) for Olskroken in Claude Code, one step at a time.

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

## Step 4: Claude Code setup – DONE (2026-11-25)
Plan approved. No code written in the Project thread. Claude Code setup session 1 results below; next: implement steps 1–10 for Olskroken.

### Session 1: setup (Claude Code, 2026-11-25)
- **Machine:** Intel Mac (Core i7, `osx-64`), macOS 26.6.2, git 2.39.5 (Xcode command line tools). Disk ~95 % full (26 GB free): keep an eye on space for city-scale runs.
- **Repo** cloned to `/Users/njasko/Documents/urban-network-tools`. Added `docs/` (00, 01, 03), `CLAUDE.md` (key rules + pointers to docs), `LICENSE` (GPL-3.0 text from gnu.org), `.gitignore` (excludes `.pixi/`, `data/`, `runs/`, `output(s)/`, `cache/`, OSM extracts, GeoPackage temp files, Python cache, `.DS_Store`). Small `.gpkg` test files are not excluded.
- **Doc paths:** in the repo the Project docs live in `docs/`, not `claude/`. `02-tool-inventory.md` is not in the repo yet.
- **GitHub access:** SSH key `~/.ssh/id_ed25519_github` (GitHub only, no passphrase), via `~/.ssh/config`; remote is `git@github.com:nikijasko/urban-network-tools.git`. Commits use name "Niki Jasko" and the GitHub noreply email (personal email kept private).
- **Environment:** pixi 0.85.0 (in `~/.pixi`). `pixi.toml` + `pixi.lock` in the repo, conda-forge only, platforms `osx-64` and `osx-arm64`. Packages: python 3.12, osmnx, neatnet, momepy, geopandas, shapely, pyyaml, pyogrio (GeoPackage I/O). Environment ~820 MB in `.pixi/` (not committed).
- **Resolved versions (all import OK, GeoPackage round trip in EPSG:3007 OK):** Python 3.12.16, osmnx 2.2.0, neatnet 0.1.6, momepy 1.0.0, geopandas 1.2.0, shapely 2.2.0, pyyaml 6.0.3, pyogrio 0.13.0, networkx 3.7, pandas 3.1.1, numpy 2.5.5, pyproj 3.8.0, libpysal 4.15.1, scipy 1.18.1, GDAL 3.13.3, PROJ 9.9.0. Note: momepy is 1.0 (the plan was researched without a fixed momepy version); check its API when writing steps 6–7.
- **Open:** licence field in `pixi.toml` set to `GPL-3.0-or-later` (the brief says "GPL-3.0"; "only" vs "or later" to confirm). The pixi installer sends an anonymous install ping; telemetry can be switched off with `PIXI_NO_TELEMETRY=1`.
- **Update from planning thread (2026-11-25):** brief replaced with the newer version (16:31). Focus is now **a clean, validated walkable network that any analysis can use**; analyses are open, and PST angular choice in this plan is only a check that the network works. OSM vs NVDB reference comparison comes right after the first working pipeline. Phase 2 is on hold. This doc is now mastered in the repo: progress recorded here is committed and pushed (after asking) so the planning thread can read it on GitHub. `CLAUDE.md` updated to match.
- **Next session:** pipeline step 1 (download) for Olskroken: area config YAML, core polygon `olskroken_core.gpkg`, custom walkable filter (verify whether OSMnx "walk" excludes `highway=cycleway`).
