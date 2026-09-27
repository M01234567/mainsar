# 05 – Non-Functional Requirements

Targets marked `[TBC]` are preliminary. They get calibrated in spike S-4 ([11](11_roadmap.md)) and
confirmed before baseline v1.0.

## 0. Reference hardware profiles

Performance targets need defined hardware. Three profiles:

| Profile | CPU | RAM | Free disk | GPU | Purpose |
|---------|-----|-----|-----------|-----|---------|
| **HW-DEV** "Laptop" (current dev machine) | 4 threads (i5-6200U) | 8 GB | 65 GB | none | Development, unit/integration tests with DS-CI only |
| **HW-PROD** "Production machine" (actual, Q-19) | AMD Ryzen 7 3700X, 8 cores / 16 threads | 32 GB | 1 TB SSD free | NVIDIA RTX 3080 Ti (12 GB, CUDA) | All real processing: AOI ≤ 100 km², ≤ 5 years, one user at a time |

HW-PROD is assessed against the recommended spec in [14](14_installation_and_operations.md) §1.2. Performance targets
are confirmed on this machine in spike S-4.

## 1. Performance (NFR-PERF)

| ID | Requirement | Target |
|----|-------------|--------|
| NFR-PERF-01 | End-to-end processing time of the **reference scenario** ([10](10_verification_and_validation.md) §4: 100 km², ~5 years, one track), excluding download, on HW-PROD. | ≤ 12 h `[TBC S-4]` (8 cores, 32 GB) |
| NFR-PERF-02 | The integration dataset DS-CI runs end-to-end on HW-DEV (development feedback loop). | ≤ 30 min, no out-of-memory `[TBC S-4]` |
| NFR-PERF-03 | Data discovery (search + estimate) for one AOI and 3 years. | ≤ 30 s |
| NFR-PERF-04 | Result viewer: initial display of up to 1 million points; pan/zoom and filter responsive. | first render ≤ 3 s; interactions ≤ 200 ms (vector tiles / WebGL) |
| NFR-PERF-05 | Time series plot after click. | ≤ 1 s |
| NFR-PERF-06 | Parallel scaling: stages that parallelise over dates or blocks should use the configured cores effectively. | ≥ 60 % parallel efficiency up to 8 cores for coregistration and phase linking |
| NFR-PERF-07 | Incremental update (FR-ANA-09) with 5 new dates. | ≤ 30 % of full re-run time |

## 2. Efficiency & resource usage (NFR-EFF, NFR-RES)

| ID | Requirement | Target |
|----|-------------|--------|
| NFR-EFF-01 | Download volume per acquisition limited to the bursts needed for the buffered AOI. | ≤ (number of intersecting bursts × burst size) + orbit file; no full scenes when burst access works |
| NFR-EFF-02 | No redundant downloads across projects (shared cache). | 0 bytes re-downloaded for cached items |
| NFR-EFF-03 | Intermediate products cropped to the AOI and cleaned up after use (unless "keep intermediates"). | Peak disk ≤ estimate + 20 %; final project footprint ≤ 25 % of peak `[TBC]` |
| NFR-EFF-04 | Stages that don't depend on each other's parameters are not recomputed (FR-CFG-05). | Verified by stage-invalidation tests |
| NFR-RES-01 | **RAM ceiling**: no stage exceeds the configured memory limit; engines are configured for block/patch processing accordingly. | Peak RSS ≤ configured limit (default: 75 % of physical RAM); verified on HW-DEV with DS-CI and a limit of 4 GB |
| NFR-RES-02 | **Disk guard**: the job pauses before free disk space drops below a configurable minimum (default 5 GB). | Tested by filling a test volume |
| NFR-RES-03 | **CPU limit** honoured by all child processes (thread env vars: `OMP_NUM_THREADS`, `MKL_NUM_THREADS`, `NUMBA_NUM_THREADS`, MATLAB `maxNumCompThreads`, …). | Measured CPU usage ≤ configured cores |
| NFR-RES-04 | The UI and API stay responsive while a heavy job runs (processing in separate processes, lower OS priority). | UI interactions ≤ 500 ms during processing on HW-PROD |

## 3. Accuracy & scientific quality (NFR-ACC)

| ID | Requirement | Target |
|----|-------------|--------|
| NFR-ACC-01 | Mean LOS velocity agrees with an independent reference (EGMS calibrated product or GNSS projected to LOS) over stable and moving areas, after common referencing and over the overlapping period. | Bias ≤ 1 mm/yr, RMSE ≤ 2 mm/yr (points with coherence ≥ threshold) `[TBC after S-2/S-3; realistic values depend on stack length and area]` |
| NFR-ACC-02 | Cross-engine consistency (PSDS vs. StaMPS-PS on common points; PSInSAR vs. SBAS aggregated to the SBAS grid). | RMSE ≤ 1.5 mm/yr `[TBC]` |
| NFR-ACC-03 | Every point carries a quality measure (temporal/ensemble coherence, velocity std. dev.); results below the configured quality threshold are not exported by default. | – |
| NFR-ACC-04 | The **sign convention**, units (mm, mm/yr), reference point and reference epoch are documented in every output. | – |
| NFR-ACC-05 | Numerical routines implemented in-house (e.g. time series fit, decomposition, comparison statistics) are verified against synthetic data with known ground truth. | Recovery error < 0.1 mm/yr on noise-free data |

## 4. Reliability & robustness (NFR-REL)

| ID | Requirement | Target |
|----|-------------|--------|
| NFR-REL-01 | Resume after interruption (power loss, kill) without data corruption (atomic writes: temp file + rename; stage markers written last). | 100 % of kill tests pass |
| NFR-REL-02 | External service outages (ASF, orbit service, CDS) produce clear, retryable errors, never silent partial results. | – |
| NFR-REL-03 | Input data validation (corrupt zips, missing orbits, inconsistent burst IDs) before expensive processing. | Detected in pre-flight or acquisition stage |
| NFR-REL-04 | Failures of third-party tools (non-zero exit, MATLAB errors) are detected and reported with the relevant log excerpt. | – |

## 5. Reproducibility (NFR-REP)

| ID | Requirement | Target |
|----|-------------|--------|
| NFR-REP-01 | Same config + same software version + same input data → bitwise-identical or numerically equivalent results (differences only from floating-point non-determinism in parallel code). | Velocity differences < 0.01 mm/yr |
| NFR-REP-02 | Software environment pinned (one pixi lockfile for all non-MATLAB dependencies; package-lock for the TS app; MATLAB release recorded). | – |
| NFR-REP-03 | Every result records the exact input data IDs (burst/scene IDs, orbit file names, DEM source + version). | – |
| NFR-REP-04 | Random processes (e.g. StaMPS random-phase density) use recorded seeds. | – |

## 6. Usability (NFR-USE)

| ID | Requirement | Target |
|----|-------------|--------|
| NFR-USE-01 | A user of persona P1 completes the reference workflow (create → run → export) using only the in-app help and user guide. | Usability test with ≥ 2 users `[TBC]` |
| NFR-USE-02 | Each parameter has help text (what it does, typical values, effect of raising/lowering it). | 100 % of UI parameters |
| NFR-USE-03 | Error messages state *what* went wrong, *why* (if known), and *what the user can do*. | Reviewed for all known error types |
| NFR-USE-04 | **British English** for UI texts, documentation, reports, log messages and code comments (e.g. *colour*, *analyse*, *licence*/*license* (verb), *metre*). Proper names and external APIs keep their original spelling (e.g. *Cloud Optimized GeoTIFF*, deck.gl `getFillColor`). UI strings externalised for later translation. | Spell-check (en-GB dictionary) in CI for docs and UI strings |
| NFR-USE-05 | Works in current Chrome/Firefox/Edge; minimum screen width 1280 px (no mobile support needed). | – |
| NFR-USE-06 | Accessibility: colour scales colour-blind safe (e.g. *RdBu*, *vik*), keyboard operable forms. | – |

## 7. Maintainability & code quality (NFR-MAINT)

| ID | Requirement | Target |
|----|-------------|--------|
| NFR-MAINT-01 | TypeScript in `strict` mode with ESLint + Prettier for application code; Python workers pinned (3.12), fully type-hinted, `mypy --strict`, `ruff` lint + format. | CI enforced |
| NFR-MAINT-02 | Automated tests: unit, integration (small synthetic/real stacks), end-to-end (reference scenario, nightly/manual). Coverage ≥ 80 % for the TS core and the Python worker package; UI end-to-end tests with Playwright. | CI enforced |
| NFR-MAINT-03 | Modularity: sensors, data sources, preprocessors, engines, unwrappers and exporters are **plugins** behind documented interfaces. | Adding an exporter needs no change in the core |
| NFR-MAINT-04 | Documentation: TSDoc (TypeScript) and NumPy-style docstrings (Python) for public APIs, architecture docs, user guide, developer guide, ADRs. | – |
| NFR-MAINT-05 | Third-party tools wrapped behind adapters; versions checked at start-up. | – |
| NFR-MAINT-06 | Semantic versioning; changelog. | – |

## 8. Portability & deployment (NFR-PORT)

| ID | Requirement | Target |
|----|-------------|--------|
| NFR-PORT-01 | Linux x86-64 is the only processing platform. Reference: Ubuntu 24.04 LTS (production); Manjaro/Arch (development). | – |
| NFR-PORT-02 | **Native installation of all dependencies from one lockfile** (pixi, conda-forge + PyPI) via a single installer, see [14](14_installation_and_operations.md). A container image without MATLAB is provided for development/CI only. | Fresh Ubuntu 24.04 VM → working install with one command (FR-INS-01) |
| NFR-PORT-03 | Other operating systems access the application only through the browser (SSH tunnel to the production machine). | – |
| NFR-PORT-04 | MATLAB is only needed for the StaMPS engine on MATLAB; everything else runs without it. | – |

## 9. Security & privacy (NFR-SEC)

| ID | Requirement | Target |
|----|-------------|--------|
| NFR-SEC-01 | Secrets never written to logs, configs, exports or debug bundles. | Automated test (secret scanning) |
| NFR-SEC-02 | Single-user mode: the HTTP server binds to `127.0.0.1` only; remote access via SSH tunnel (ADR-008). Binding to other interfaces needs an explicit setting and shows a warning, since there is no authentication. | Port scan from another host finds no open port |
| NFR-SEC-03 | Protection against cross-site requests to the local API (Origin/Host header checks, CSRF token), since a browser on the same host could be tricked into calling it. | Requests with foreign Origin rejected |
| NFR-SEC-04 | Uploaded files (AOI) validated and size-limited; no execution of user-provided code; safe path handling (no path traversal). | – |
| NFR-SEC-05 | Dependencies scanned for known vulnerabilities (e.g. `pip-audit`) in CI. | – |

## 10. Legal & compliance (NFR-LEG)

| ID | Requirement | Target |
|----|-------------|--------|
| NFR-LEG-01 | The project is published under **GPL-3.0**. All dependency licences are documented and checked for compatibility (incl. `triangle`'s and SNAPHU's own terms). MATLAB is never redistributed. | Licence report in the repo (generated in CI) |
| NFR-LEG-02 | Data attribution in exports and reports ("Contains modified Copernicus Sentinel data [year]", DEM attribution, EGMS attribution). | – |
| NFR-LEG-03 | Respect data provider terms of use and rate limits (concurrency limits, user agent). | – |
| NFR-LEG-04 | All mandatory components are open source (FR-ANA-14). A proprietary dependency may only be an optional alternative and must be clearly marked as such in the UI and docs. | Licence report shows no mandatory proprietary component |
