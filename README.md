# MainSAR

MainSAR measures ground deformation with multi-temporal InSAR (Persistent Scatterer / Distributed Scatterer)
from Sentinel-1 data. You define an area of interest and a time range, choose a preset or tune the
parameters, and MainSAR downloads the required data, processes the stack and delivers geocoded velocity maps
and displacement time series. You can view them in the built-in viewer, in Google Earth (KMZ) or in any GIS.

> **Status:** requirements baseline approved (v1.0). Technical spikes in progress; no usable release yet.

## Documentation

- [Requirements baseline](docs/requirements/README.md): vision, method analysis, requirements,
  architecture, installation, verification plan, roadmap and decision log.
- [Spikes](spikes/README.md): time-boxed technical experiments that de-risk the implementation.

## Planned repository layout

| Path | Content | Language |
|------|---------|----------|
| `apps/server` | HTTP API, job orchestration, CLI | TypeScript |
| `apps/web` | Browser UI and viewer | TypeScript |
| `workers` | Processing workers (ISCE2, Dolphin, StaMPS adapters) | Python |
| `schemas` | JSON Schemas: single source of truth for config, parameters, worker contract, results | JSON |
| `installer` | `install.sh` and environment lockfiles | Bash, pixi |
| `docs` | Requirements, ADRs, user and developer guides | Markdown |
| `spikes` | Technical experiments (not product code) | mixed |

## Licence

MainSAR is free software under the [GNU General Public License v3.0](LICENSE).
It builds on third-party tools with their own licences (ISCE2, Dolphin, StaMPS, SNAPHU, triangle, …).
MATLAB is proprietary and has to be licensed separately by the user.

Contains modified Copernicus Sentinel data processed by MainSAR users.
