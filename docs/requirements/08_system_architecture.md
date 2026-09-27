# 08 – System Architecture (proposal)

This is the **high-level** architecture derived from the requirements. It is a proposal for review:
detailed design (module structure, API specification) follows after baseline approval. Decisions are
recorded as ADRs (§6). Alternatives are listed so the rationale can be checked.

## 1. Architectural drivers

| Driver | Source | Consequence |
|--------|--------|-------------|
| Heavy, long-running, failure-prone processing (hours) | FR-JOB-*, NFR-REL-* | Separate worker processes, checkpointed stages, UI decoupled from processing |
| InSAR tooling exists only in Python/C/Fortran/MATLAB | 02 §3 | Python is **confined to processing workers** behind a language-neutral contract |
| Team preference: TypeScript over Python for application code | Q-06, Q-17 | Backend, CLI and frontend in TypeScript; shared types |
| Single production machine, one user at a time | Q-05, Q-05b | No auth, no distributed queue; local SQLite; process-level scheduling |
| MATLAB with an individual, machine-bound licence | Q-07 | MATLAB runs **natively on the host**, so processing is not containerised (ADR-006) |
| One installer must provide *all* dependencies | PO requirement | Lockfile-pinned native environment + install script + self-test (FR-INS-*) |
| Several processing engines, future sensors | FR-ANA-01, NFR-MAINT-03 | Plugin interfaces, standardised result schema |
| Reproducibility | NFR-REP-* | Config file as single source of truth, input hashing, lockfiles |
| Large result sets (up to ~1 M points) in the browser | NFR-PERF-04 | WebGL rendering, binary (Arrow/Parquet) data transfer |

## 2. Component view

```mermaid
flowchart TB
    subgraph Browser
        UI[Web UI<br/>TypeScript · React · Vite<br/>MapLibre GL · deck.gl · Plotly]
    end

    subgraph Host["Production host (Linux, single user)"]
        subgraph App["Application (TypeScript / Node.js)"]
            API[HTTP API + SSE<br/>Fastify]
            CORE[Core<br/>projects · config · stage orchestrator<br/>input hashing · estimator · scheduler]
            CLI[CLI]
            DB[(SQLite state DB)]
        end

        subgraph Workers["Processing workers (Python, pinned env)"]
            WU[Utility commands<br/>AOI import · search · estimate]
            WS[Stage commands<br/>acquire · coregister · engine · post · export]
        end

        subgraph Tools["Native tools"]
            ISCE[ISCE2 topsStack]
            DOL[Dolphin]
            MPY[MintPy]
            SNA[SNAPHU]
            STA[StaMPS C++ tools + triangle]
            MAT[MATLAB (host, licensed)]
        end

        FS[(File storage<br/>cache + projects)]
    end

    EXT[(ASF · orbits · DEM · ERA5 · EGMS)]

    UI <-->|REST · SSE · static files| API
    API --> CORE
    CLI --> CORE
    CORE <--> DB
    CORE -->|spawn · JSON contract| WU
    CORE -->|spawn · JSON contract| WS
    WS --> ISCE & DOL & MPY & SNA & STA & MAT
    WS <--> FS
    WU <--> EXT
    WS <--> EXT
    API -->|range requests: COG / Parquet| FS
```

### 2.1 Responsibilities

| Component | Language | Responsibility |
|-----------|----------|----------------|
| **Core** | TypeScript | Domain model, project config, stage graph, input hashing / invalidation, job scheduling, process supervision (spawn, progress, cancel, resource limits), resource estimator, state DB. No HTTP code. |
| **API** | TypeScript | Thin HTTP layer (Fastify): projects, search, jobs, results; SSE for progress/logs; serves the UI bundle and result files (HTTP range requests). OpenAPI spec generated from schemas. Binds to `127.0.0.1` by default. |
| **CLI** | TypeScript | Same core as the API (FR-CLI-03). |
| **Web UI** | TypeScript | Map-centric SPA: project wizard, discovery, schema-driven parameter forms, job monitor, result viewer, comparison. |
| **Workers** | Python | Stateless command-line programs, one per stage or utility, wrapping ISCE2, Dolphin, StaMPS, asf_search, burst2safe, GDAL/rasterio. **No orchestration logic, no state beyond their output directory.** |
| **Native tools** | C/C++/Fortran/MATLAB | Invoked only by workers. |

### 2.2 Worker contract (language boundary)

Each worker call is a separate OS process:

```
mainsar-worker <command> --request request.json --workdir <dir>
```

- **Input** `request.json`: stage name, effective parameters, input artefact paths, resource budget
  (threads, RAM, scratch dir). Validated against the shared JSON Schema.
- **Progress/log output**: JSON Lines on stdout (`{"type":"progress","stage":"coregister","done":37,"total":120}`,
  `{"type":"log","level":"info",...}`, `{"type":"metric",...}`). stderr is captured into the stage log.
- **Result** `result.json` in the work directory: produced artefacts (paths + checksums), QA metrics,
  tool versions. Written **last**, atomically.
- **Exit codes**: `0` success, `2` invalid request, `3` external service error (retryable), `4` resource limit,
  `5` tool failure, `130` cancelled.
- **Cancellation**: SIGTERM to the process group; workers clean up child processes (MATLAB, SNAPHU) within a timeout,
  then SIGKILL.

This contract is the only coupling between the TypeScript application and Python. It can be tested from both
sides with fixture files.

### 2.3 Single source of truth for schemas

Project configuration, processing parameters, the worker contract and the result schema are defined as
**JSON Schema files** in `schemas/`. From these we generate:
- TypeScript types and runtime validators (backend, CLI, UI forms),
- Python models (pydantic) for the workers,
- parameter documentation (07 catalogue, help texts).

CI fails if generated code is out of date.

## 3. Pipeline & stages

```mermaid
flowchart LR
    S1[discover] --> S2[acquire<br/>bursts · orbits · DEM]
    S2 --> S3[prepare<br/>burst2safe · DEM ellipsoid]
    S3 --> S4[coregister<br/>topsStack + ESD]
    S4 --> S5[crop & geometry]
    S5 --> S6[engine.prepare]
    S6 --> S7[engine.run<br/>select · link · unwrap · invert]
    S7 --> S8[engine.collect<br/>→ result schema]
    S8 --> S9[post-process<br/>reference · tropo · model · geocode]
    S9 --> S10[export & report]
```

**Stage contract** (all stages):
- **Deterministic inputs**: upstream artefact checksums + parameters relevant to this stage + worker/tool versions → `input_hash`.
- **Idempotent**: re-running with the same `input_hash` is a no-op; outputs written to a temp dir and
  atomically renamed; `result.json` written last.
- **Resumable at sub-stage level** where cheap (per-date download, per-date coregistration, per StaMPS step).
- **Resource declaration**: each stage declares expected CPU/RAM/disk as a function of stack size, used by the
  estimator (FR-DIS-03) and the scheduler.
- **Cancellable** via the worker contract.

## 4. Deployment

Single Linux host, native installation, single user (ADR-006, ADR-008):

```mermaid
flowchart LR
    B[Browser on the host,<br/>or via SSH tunnel] -->|127.0.0.1:8080| N
    subgraph Host["Production host"]
        N[Node.js app<br/>systemd user service]
        P[Python worker env<br/>pixi, lockfile-pinned]
        M[MATLAB (optional, host-installed, licensed)]
        V[(data root: cache + projects)]
        N --> P
        P --> M
        N --- V
        P --- V
    end
```

- Remote use is through an **SSH port forward** (`ssh -L 8080:localhost:8080 host`). The app itself exposes no network service.
- A **container image without MATLAB** is also provided for development and CI (integration tests with
  the Dolphin engine). It is not the production path.

Installation and operation are specified in [14](14_installation_and_operations.md).

## 5. Technology stack (proposal)

| Layer | Choice | Alternatives considered | Reason |
|-------|--------|-------------------------|--------|
| Application language | **TypeScript** (strict), Node.js LTS | Go, Kotlin, Python | Team preference; types shared between backend and frontend (Q-17) |
| HTTP framework | **Fastify** + SSE | Express, NestJS, Hono | Fast, schema-first (JSON Schema native), low overhead |
| Schemas / validation | **JSON Schema** as source; Ajv (TS), pydantic (Py) generated | Zod-first, TypeBox-first | Language-neutral source of truth (§2.3) |
| State DB | **SQLite** (better-sqlite3) + Kysely or Drizzle (typed SQL) | PostgreSQL | Single user, zero ops, transactional |
| Job scheduling | In-process scheduler in core + child processes | BullMQ/Redis, Temporal | One machine, one user: no broker needed |
| CLI | commander / oclif | – | Same core as the API |
| Frontend | **React + TypeScript + Vite**, MapLibre GL, **deck.gl** (WebGL points), Plotly.js, react-jsonschema-form | Vue/Svelte; OpenLayers/Leaflet | deck.gl renders ≥ 1 M points; form generation from JSON Schema |
| Result data in browser | Points: GeoParquet/Arrow via loaders.gl or parquet-wasm; rasters: COG read client-side (geotiff.js / deck.gl) | Tile server (TiTiler) | AOIs ≤ 100 km² make client-side rendering feasible; no extra Python server |
| Worker language | **Python 3.12** (pinned) | – | Only language with the needed InSAR libraries |
| Worker libs | ISCE2, Dolphin, asf_search, burst2safe, sentineleof, dem-stitcher, rasterio, GDAL, GeoPandas, Shapely 2, pyproj, xarray, PyAPS `[VERIFY versions S-1/S-2/S-6]` | – | Maintained, standard |
| Engines | PSInSAR: Dolphin (default), StaMPS 4.1b + our patches; SBAS: MintPy | MiaplPy, SARvey | ADR-004 |
| Unwrapping | SNAPHU, spurt; StaMPS-3D (needs `triangle`) | – | |
| Environment | **pixi** (conda-forge + PyPI, lockfile) for Python, native tools and Node.js | conda-lock + micromamba; Docker | One lockfile for all non-MATLAB dependencies, reproducible, user-space install (ADR-006) |
| Reports | HTML templates (TS) → optional PDF (headless Chromium) | WeasyPrint | Stays in the TS stack |
| Quality | TS: Vitest, ESLint, Prettier, `tsc --strict`; Py: pytest, ruff, mypy; Playwright for UI E2E | – | NFR-MAINT-* |

## 6. Architecture decision records (summary)

| ADR | Decision | Status |
|-----|----------|--------|
| ADR-001 | **TypeScript** for application code (backend, CLI, frontend). **Python only in stateless processing workers** behind a JSON/CLI contract. | Accepted (Q-17) |
| ADR-002 | Browser-based UI served by the local backend. | Accepted (Q-06) |
| ADR-003 | ISCE2 topsStack with burst-based input (ASF bursts → burst2safe) as the single preprocessor for all engines. Fallback: full SLC scenes. | Proposed `[S-1]` |
| ADR-004 | Pluggable analysis engines with a standardised result schema. **PSInSAR: Dolphin (default) and StaMPS-PS; SBAS: MintPy.** All mandatory engines open source. | Accepted (Q-01, Q-21) |
| ADR-005 | Own stage orchestrator in the TS core with input hashing + SQLite state; no message broker. | Proposed |
| ADR-006 | **Native installation** with a lockfile-pinned pixi environment. MATLAB stays a host installation (machine-bound licence). Container image only for dev/CI. | Proposed `[S-6]` |
| ADR-007 | Internal results as GeoParquet (points) + COG (rasters) + Zarr/NetCDF (cube); user exports derived from them. | Proposed |
| ADR-008 | Single-user mode: no authentication; server binds to `127.0.0.1`; remote access via SSH tunnel. | Accepted (Q-05b) |
| ADR-009 | JSON Schema files as the single source of truth for config, parameters, worker contract and results; code generation for TS and Python. | Proposed |

Full ADR texts (context, decision, consequences) go under `docs/adr/` once approved.

## 7. Cross-cutting concerns

- **Resource governance**: the core passes thread and memory budgets to each worker. Workers set
  `OMP_NUM_THREADS`, `MKL_NUM_THREADS`, `NUMBA_NUM_THREADS`, MATLAB `maxNumCompThreads`, Dolphin block size,
  StaMPS patch grid and SNAPHU tiles. Optionally, the core runs workers in a systemd scope / cgroup with a memory limit.
- **Observability**: structured logs (JSON Lines) per job/stage; per-stage metrics (duration, peak RSS, disk delta)
  stored in SQLite. They feed the estimator calibration.
- **Error handling**: worker exit codes map to typed errors in the core (`ExternalServiceError`, `ToolError`,
  `ResourceLimitError`, `ValidationError`, …) and then to user-facing messages (NFR-USE-03).
- **Configuration layering**: tool defaults → preset → project config → run overrides; the effective config is
  stored with every run.
- **Testing seams**: the worker contract lets the TS core be tested with fake workers, and workers be tested
  without the TS core. External services are recorded as fixtures.
