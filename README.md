# MainSAR

MainSAR measures ground deformation with multi-temporal InSAR (Persistent Scatterer / Distributed Scatterer)
from Sentinel-1 data. You define an area of interest and a time range, choose a preset or tune the
parameters, and MainSAR downloads the required data, processes the stack and delivers geocoded velocity maps
and displacement time series. You can view them in the built-in viewer, in Google Earth (KMZ) or in any GIS.

> **Status:** Phase 1 (headless MVP core) complete in its first version: from an AOI and a time range to
> geocoded PS/DS velocities and time series (GeoPackage, CSV, COG, KMZ) with the Dolphin engine. Phase 2 (browser
> UI) started: projects, in-app AOI map selector, data search, processing monitor and result viewer work.
> Not yet a release: see [docs/implementation/phase1-plan.md](docs/implementation/phase1-plan.md) and the
> [open questions](docs/implementation/open-questions.md).

## Quick start

```bash
installer/install.sh                  # everything except MATLAB, into ~/.local/share/mainsar (no root needed)
export PATH=~/.local/share/mainsar/bin:$PATH
# NASA Earthdata login for downloads: ~/.netrc with "machine urs.earthdata.nasa.gov login … password …"

mainsar serve                         # browser UI at http://127.0.0.1:8765/ (remote: ssh -L 8765:localhost:8765 host)
```

or on the command line:

```bash
mainsar init bologna && cd bologna
mainsar aoi bbox 11.33 44.485 11.355 44.503 project.yaml   # or: mainsar aoi import area.geojson project.yaml
mainsar search project.yaml                                 # candidate tracks with estimates
mainsar search project.yaml --select 95                     # choose a track
mainsar run project.yaml                                    # download, coregister, analyse, export (resumable)
mainsar status project.yaml
```

Results are written to `stages/export/exports/`: `<name>.gpkg`, `<name>.csv`, `<name>.kmz` (Google Earth),
`rasters/*.tif` (COG) and `metadata.json`. Displacements are in mm, velocities in mm/yr, positive towards the
satellite, relative to the reference area and the first date.

## Documentation

- [Requirements baseline](docs/requirements/README.md): vision, method analysis, requirements,
  architecture, installation, verification plan, roadmap and decision log.
- [Spikes](spikes/README.md): time-boxed technical experiments that de-risk the implementation.

## Development

Everything runs inside the pinned pixi environment (Node.js 24 LTS, Python 3.11); no system Node or Python needed.

```bash
~/.pixi/bin/pixi install              # development environment (add `-e full` for all processing tools, ~3.5 GB)
pixi run npm ci                       # Node.js dependencies
pixi run check                        # everything CI runs: generated code, lint, type checks, all tests
pixi run generate                     # after editing schemas/*.json
pixi run npm run build && pixi run node apps/cli/dist/main.js --help
pixi run -e full node apps/cli/dist/main.js serve   # UI + API with all processing tools
pixi run npm run dev -w @mainsar/web                # UI development server (proxies /api to `serve`)
node apps/web/e2e/smoke.mjs --url http://127.0.0.1:8765 --search   # browser smoke test (system Chromium)
```

## Repository layout

| Path | Content | Language |
|------|---------|----------|
| `packages/core` | Core library: config, orchestrator, worker supervisor, state DB | TypeScript |
| `apps/cli` | `mainsar` command-line interface | TypeScript |
| `packages/server` | HTTP API, progress events (SSE), result files, UI hosting (`mainsar serve`) | TypeScript |
| `apps/web` | Browser UI: projects, AOI map selector, search, processing monitor, result viewer | TypeScript/React |
| `installer` | `install.sh` (FR-INS-*), StaMPS patches | Bash |
| `testdata` | Small DS-CI crop for `mainsar doctor --full` | GeoTIFF/JSON |
| `workers` | Processing workers (ISCE2, Dolphin, StaMPS adapters) | Python |
| `schemas` | JSON Schemas: single source of truth for config, parameters, worker contract, results | JSON |
| `scripts` | Code generators (schemas → TypeScript/Python), test-data builder | JS, Python |
| `pixi.toml` / `pixi.lock` | Product environment (development, CI, installation) | pixi |
| `docs` | Requirements, ADRs, user and developer guides | Markdown |
| `spikes` | Technical experiments (not product code) | mixed |

## Licence

MainSAR is free software under the [GNU General Public License v3.0](LICENSE).
It builds on third-party tools with their own licences (ISCE2, Dolphin, StaMPS, SNAPHU, triangle, …).
MATLAB is proprietary and has to be licensed separately by the user.

Contains modified Copernicus Sentinel data processed by MainSAR users.
