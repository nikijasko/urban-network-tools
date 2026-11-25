# Project Brief: QGIS tools for urban morphology

Last updated: 2026-11-25

## Current status
*Updated by the hub thread whenever a step finishes or the plan changes.*

- **Done:** project setup (this brief, conventions, GitHub repo); tool research → `claude/02-tool-inventory.md` (final); Phase 1 design, steps 1–3 → `claude/03-phase1-pipeline.md` (plan approved 2026-11-25).
- **Now:** Phase 1, step 4: set up Claude Code on my Mac and write/test the pipeline scripts, starting with the Olskroken test patch.
- **Next:** add the GPL-3.0 licence file to the repo before the first script; run the pipeline on Olskroken and review the output in QGIS; then reference comparison (OSM vs NVDB) and the hand-checked test areas; then city scale.
- **On hold:** Phase 2 (wrapping depthmapX and cityseer, comparing engines) until I know which analyses I want.
- **Open questions:** see the lists at the end of `claude/02-tool-inventory.md` and the risks in `claude/03-phase1-pipeline.md`.

## Goal
Build a set of tools that automate street network creation and urban morphology analyses in QGIS, so the same work can be repeated reliably across sites and cities.

## Output
- **Now:** a page on my website (https://nikijasko.github.io) with scripts, explanations and worked examples. **The first page is the network pipeline itself**; it doesn't wait for analyses.
- **Later (maybe):** package the tools as a QGIS plugin.

## Focus
- **First priority: getting the network right.** A clean, validated walkable network that any analysis can use. This is a product in its own right.
- **Analyses: open, to be decided later.** Angular choice in PST is used for now to check that the network works for analysis, not as a commitment to which analyses the project will do.
- **Later:** analyses (space syntax and others), densities and functions (land use, mix, attractors).
- **Parked:** building-level analyses. To be decided later.

## Network representation
- **Walkable network only.** No motorways or motorway ramps; footways and cycle paths are included. Details of the filter are in `claude/03-phase1-pipeline.md`.
- **Centre lines** (one line per street, dual carriageways merged), the space syntax standard. This is what the pipeline builds. A street's line also stands for its sidewalks and cycle tracks; paths that are separate spaces (park paths, tracks behind a green strip) stay as their own lines.
- **Unlinks are required** (bridges and tunnels where lines cross but don't connect).
- These representation choices **stay as decided**, independent of which analyses come later. Thresholds already in the plan (e.g. the parallel-path distance) remain logged parameters.
- Parked idea: an automated hybrid pedestrian network (two sidewalk lines where a road is hard to cross, one line otherwise), based on the method used in my thesis. Not planned; revisit only if I ask.

## Geographic scope
- **Mainly Europe** for now. The tools must not depend on Swedish data.
- **Default data: open sources available across Europe** (OpenStreetMap, plus other Europe-wide or global open datasets where useful).
- **Local official data** (e.g. NVDB, Lantmäteriet, city base maps) is an **optional add-on per country**, mainly used as a reference for validation.
- **Gothenburg is the test case**, chosen because good reference data exists there. It is not a limit on the project. Test areas are defined in `claude/01-conventions.md`.

## Requirements
- Must work at **city scale**.
- **Validation is central**, since network quality is the focus: Level 1 checks after every step, the hand-checked test areas, and **comparison with an official reference network (OSM vs NVDB) right after the first working pipeline** (moved forward from Phase 2).

## Approach
- Use existing tools where they are good and fit. Build only what's missing (see `claude/02-tool-inventory.md`).
- The main value is expected to be in the **glue**: connecting data preparation, network cleaning, validation and output into one repeatable workflow.

## Where things live
- **Code:** a separate, public GitHub repo just for the tools, named **`urban-network-tools`** (scripts, test data, validation, later the plugin). Rule of thumb: if it runs, it goes here.
- **Website repo:** only what appears on the site (pages, explanations, images). Kept light; links to the tools repo. Rule of thumb: if it explains or shows, it goes here.
- **Decisions and explanations:** the docs in this Claude Project are the master copies of the Brief and Conventions. The repo's `docs/` folder holds copies for Claude Code, and is the master for the Phase 1 pipeline doc once building starts. The hub reads the repo to stay in sync.

## Licences
- **Code: GPL-3.0.** Keeps improved versions open, fits QGIS plugins, combines cleanly with AGPL tools (cityseer, sDNA). Licence file to be added to the repo before the first script.
- **Data derived from OSM or Overture: ODbL.** Needs attribution ("© OpenStreetMap contributors") and derived data shared under the same terms. Applies to sample data in the repo and maps on the website.

## My setup
- **QGIS 3.42.1** (with PST 3.3.2) on macOS. I'm staying on this version for development; no extra install for now.
- I currently download OSM data and edit the network by hand before running PST. The pipeline should replace the manual editing.
- Building happens in **Claude Code** on my Mac; planning and decisions in the Claude Project.

## How we work in Claude
- This Project holds the shared docs. Every thread reads them before starting.
- One coordination thread (hub) for general questions and small decisions; separate threads for each workstream.
- Decisions get written into the docs so other threads can see them.
