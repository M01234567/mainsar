# 04 – Functional Requirements

Priorities: **M**ust / **S**hould / **C**ould / **W**on't (see [README](README.md)).
"The system" = the MainSAR application (UI, API, workers, CLI).
Use-case references point to [03](03_stakeholders_and_use_cases.md).

---

## 1. Project management (FR-PRJ) – UC-01, UC-07

| ID | Requirement | Prio | Acceptance criteria |
|----|-------------|:----:|---------------------|
| FR-PRJ-01 | The system shall let users create, open, rename, clone and delete projects. | M | All five operations work from UI and CLI; deleting asks for confirmation and states which data will be removed (project outputs vs. shared cache). |
| FR-PRJ-02 | A project shall be **fully described by one human-readable configuration file** (YAML) holding AOI, time range, track selection, engine, all parameters and the software version. | M | Exported YAML re-imported on another installation reproduces the same run (see NFR-REP-01). The file validates against a published JSON Schema. |
| FR-PRJ-03 | The system shall keep a per-project directory layout with separate areas for inputs (links to the shared cache), intermediate products, results and logs. | M | Layout documented in [06](06_data_and_interface_requirements.md) §5; no files are written outside the project and cache directories. |
| FR-PRJ-04 | The system shall keep a **run history** per project (parameters, start/end time, software versions, status, outputs) and let users compare two runs. | M | The history lists all runs; a diff of two runs shows changed parameters and QA metrics side by side. |
| FR-PRJ-06 | **Easy reprocessing** (Q-15): from any finished run the user can start a new run with (a) changed parameters, (b) another engine, (c) a changed reference point, or (d) an extended/shortened time range. Only affected stages are recomputed (FR-CFG-05), and cached/retained data is reused. | M | Changing only analysis parameters reprocesses without any download or coregistration; extending the range by N dates downloads and coregisters only those N dates (with FR-PRE-07). |
| FR-PRJ-05 | The system shall let users clean intermediate products of a finished project to free disk space, keeping results and configuration. | S | After cleaning, results stay viewable/exportable and the freed space is reported. |

## 2. Area of interest (FR-AOI) – UC-01

| ID | Requirement | Prio | Acceptance criteria |
|----|-------------|:----:|---------------------|
| FR-AOI-01 | Users shall be able to **draw** a rectangle or polygon on an interactive map (with OSM and satellite base map). | M | Drawing, editing vertices, and deleting work; coordinates are shown. |
| FR-AOI-02 | Users shall be able to **import** an AOI from GeoJSON, KML/KMZ, ESRI Shapefile (zip) and GeoPackage. | M | Each format with a test file imports correctly; non-WGS84 CRS is reprojected to EPSG:4326. |
| FR-AOI-03 | Users shall be able to enter a bounding box (min/max lon/lat) or a point + radius. | M | Values validated (range, order). |
| FR-AOI-04 | The system shall validate the AOI geometry (valid, non-self-intersecting, not empty) and repair trivially invalid geometries (e.g. `make_valid`) with a warning. | M | Invalid test geometries are repaired or rejected with a clear message. |
| FR-AOI-05 | The system shall enforce a configurable **maximum AOI area** and warn when the AOI crosses burst or sub-swath boundaries, since that raises cost. | M | Default limit 100 km² (warning), hard limit configurable (default 250 km²); the warning shows the number of bursts affected. |
| FR-AOI-06 | The system shall process a **buffer** around the AOI (default 1 km, configurable). The minimum buffer is derived from the processing windows (e.g. Dolphin phase-linking half window, filter windows) so that the AOI itself has no edge no-data. (Spike S-2: ~5 px no-data border.) | M | Buffer is applied and shown on the map; no edge no-data inside the AOI in DS-CI results. |
| FR-AOI-07 | The system shall let the user place an optional **reference area/point** (stable ground) on the map. | S | Stored in the project and used by FR-ANA-06. |

## 3. Data discovery (FR-DIS) – UC-02

| ID | Requirement | Prio | Acceptance criteria |
|----|-------------|:----:|---------------------|
| FR-DIS-01 | The system shall search the Sentinel-1 IW SLC archive (burst level where available) for AOI + time range + polarisation. | M | Results match a manual ASF Vertex search for three test AOIs. |
| FR-DIS-02 | Results shall be grouped by **relative orbit + flight direction**, showing: number of acquisitions, first/last date, timeline with gaps, AOI coverage %, number of bursts per acquisition, satellites (S1A/B/C/D). | M | All fields shown for each track; coverage computed from burst footprints. |
| FR-DIS-03 | The system shall show a **resource estimate** per track before processing: download volume, peak disk usage, RAM need, rough processing time on the configured hardware. | M | Estimate within ±30 % of measured values for the reference scenario (after calibration in spike S-4). |
| FR-DIS-04 | The system shall **warn about unsuitable stacks**: fewer than N acquisitions (default 20 for PS `[TBC]`), gaps longer than X days, incomplete AOI coverage, mixed burst IDs (changed acquisition geometry). | M | Each condition triggers a specific warning in tests with synthetic search results. |
| FR-DIS-05 | Users shall be able to exclude single acquisitions (manually, or with a rule such as "exclude Dec–Feb" to avoid snow). | S | Excluded dates are not downloaded or processed and are recorded in the configuration. |
| FR-DIS-06 | The system shall show the **perpendicular/temporal baseline plot** of the selected stack and propose a reference (super-master) date that minimises total baseline decorrelation. | S | The plot is shown; the proposed reference date matches the StaMPS/ISCE heuristic (min. sum of normalised baselines). |
| FR-DIS-08 | The system shall treat **all Sentinel-1 units (S1A, S1B, S1C, S1D)** as one constellation: acquisitions of all units with the same relative orbit and burst IDs go into one stack; per-date satellite is recorded and shown; S1B archive data (2016–2021) is included automatically when the time range covers it. | M | A 2019–2023 search returns S1A and S1B dates in one track stack; S1B-only dates are coregistered and processed like S1A dates. |
| FR-DIS-07 | The system shall let users process **several tracks** (e.g. ascending + descending) as separate stacks in one project. | S | Two tracks run as independent sub-jobs; results listed per track. |

## 4. Data acquisition (FR-ACQ) – stages 3

| ID | Requirement | Prio | Acceptance criteria |
|----|-------------|:----:|---------------------|
| FR-ACQ-01 | The system shall download **only the SLC bursts intersecting the (buffered) AOI** where burst products are available, with fallback to full SLC scenes. | M | Downloaded volume per acquisition ≈ number of required bursts × burst size (NFR-EFF-01). |
| FR-ACQ-02 | The system shall download **precise orbit files** (POEORB; RESORB fallback for recent acquisitions, with a warning and a later re-run option). | M | Every acquisition has a matching orbit file; orbit type is recorded in the metadata. |
| FR-ACQ-03 | The system shall download and mosaic the **DEM** (default Copernicus GLO-30; alternatives SRTM 1″, user-supplied GeoTIFF) for the buffered AOI, converted to the reference required by the preprocessor (WGS84 ellipsoid heights). | M | DEM covers the AOI plus margin; geoid→ellipsoid conversion verified against a known point. |
| FR-ACQ-04 | Downloads shall be **parallel** (configurable concurrency), **resumable**, written atomically (temp file + rename), **integrity-verified** (size/checksum where available, otherwise a full read of the raster) and retried with exponential back-off. Third-party download tools are not trusted to do this (spike S-1: burst2stack reused truncated files). | M | Killing the process during download and restarting continues without re-downloading completed files; truncated or corrupted files are detected and re-downloaded. |
| FR-ACQ-05 | The system shall keep a **shared, content-addressed cache** of downloaded data so several projects reuse the same bursts/orbits/DEM tiles. | M | A second project over the same area downloads nothing that is already cached. |
| FR-ACQ-06 | The system shall be able to repackage burst products into the format the preprocessor needs (e.g. SAFE via `burst2safe` `[VERIFY S-1]`). | M | ISCE2 topsStack accepts the repackaged data. |
| FR-ACQ-07 | Users shall be able to supply locally stored SLC zips instead of downloading. | S | A directory of SAFE zips is recognised and indexed. |
| FR-ACQ-09 | **Retention policy** (1 TB disk, Q-19): by default raw bursts are evicted after successful coregistration, while the cropped coregistered stack and geometry of each project are **retained** so reprocessing (FR-PRJ-06) needs no re-download. Policy configurable per project (keep raw / keep stack / results only). | M | With the default policy, reprocessing with new analysis parameters needs no network access; disk usage per finished project ≈ cropped stack + results. |
| FR-ACQ-08 | The system shall have a configurable cache size limit with LRU eviction of unused data. | C | Eviction never removes data used by a running job. |

## 5. Stack preprocessing (FR-PRE) – stage 4

| ID | Requirement | Prio | Acceptance criteria |
|----|-------------|:----:|---------------------|
| FR-PRE-01 | The system shall build a **single-reference coregistered SLC stack** with geometric coregistration and **ESD** azimuth refinement (ISCE2 topsStack, workflow `slc`). | M | ESD residual azimuth offset stats are logged; interferograms show no burst-edge phase jumps (visual QA + metric: phase discontinuity at burst overlaps < threshold). |
| FR-PRE-02 | Processing shall be **restricted to the bursts/sub-swaths covering the AOI**, and coregistration shall **stream per date**: after a date is resampled it is cropped to the buffered AOI and its full-burst products and offsets are deleted immediately. (Spike S-1: without this, intermediates were ~1,500× larger than the cropped result.) | M | Peak disk during coregistration ≤ raw burst cache + reference products + (parallel dates × per-date full-burst size) + 10 %; verified on DS-CI and DS-REF. |
| FR-PRE-03 | The reference date shall be selectable (auto per FR-DIS-06 or manual). | M | The chosen date is used and recorded. |
| FR-PRE-04 | The system shall produce the **geometry products** needed downstream: lat/lon/height per pixel, incidence and heading angles, perpendicular baselines per date, shadow/layover mask. | M | All products exist and are consistent (lat/lon within AOI bounds; incidence angle within 29–46°). |
| FR-PRE-05 | The system shall produce **amplitude dispersion (D_A)** and mean amplitude images as QA and PS-candidate input. | M | Rasters exist and are viewable. |
| FR-PRE-06 | Coregistration shall run in parallel over dates (configurable number of processes) within the memory limit. | M | CPU usage scales with the configured process count; peak RAM ≤ configured limit. |
| FR-PRE-07 | Adding new acquisitions to an existing stack shall coregister **only the new dates** to the existing reference. | M | Extending by 5 dates processes 5 dates, not the whole stack. |
| FR-PRE-08 | Intermediate full-burst products shall be deleted once cropped products are verified (configurable "keep intermediates" flag). | S | With the flag off, only cropped products remain after the stage. |
| FR-PRE-09 | For SBAS, the system shall generate a **small-baseline interferogram network from the AOI-cropped coregistered stack** (not from full bursts): pair selection (sequential *n* nearest, optional temporal/perpendicular baseline thresholds), multilooking, filtering (Goldstein), coherence, SNAPHU unwrapping. (Spike S-1: ISCE2's interferogram workflow needs ~0.55 GB per pair for full-burst interferograms.) | M | Network, coherence and unwrapped interferograms are produced for DS-CI; disk use per pair ≤ 2 × cropped SLC size; the network plot is shown (FR-VIS-07). |
| FR-PRE-11 | External tool runs (e.g. ISCE2 run files) shall be executed command by command by the worker, with every exit code checked; generated shell scripts that background jobs (`&`/`wait`) are not executed as-is. (Spike S-1.) | M | A deliberately failing command in a run file fails the stage. |
| FR-PRE-10 | Burst selection shall guarantee **ESD-capable coverage**: if the AOI lies within a single burst, the adjacent burst in azimuth is added (ESD needs burst overlaps); if that is impossible, coregistration falls back to geometry-only with a warning. (Finding from spike S-1.) | M | Single-burst AOI → two bursts downloaded; the fallback is logged and shown in the report. |

## 6. Multi-temporal analysis (FR-ANA) – stages 5–8

| ID | Requirement | Prio | Acceptance criteria |
|----|-------------|:----:|---------------------|
| FR-ANA-01 | The system shall provide an **engine interface** so analysis engines are interchangeable (input: coregistered stack + geometry; output: standardised result dataset, see [06](06_data_and_interface_requirements.md) §4). | M | Two engines (Phase 3) produce outputs in the same schema; post-processing code has no engine-specific branches. |
| FR-ANA-02 | **Engine "PSDS" (Dolphin)**: PS selection via amplitude dispersion, DS via SHP + phase linking, interferogram network, unwrapping, time-series inversion. Default engine (Q-01). | M | Reference scenario processes end-to-end; outputs pass V&V ([10](10_verification_and_validation.md)). |
| FR-ANA-03 | **Engine "StaMPS-PS"**: StaMPS PS steps 1–8 including export from ISCE2 (`make_single_reference_stack_isce`, `mt_prep_isce`), patch handling, merge, 3D unwrapping, SCLA/DEM error, atmospheric filtering; running on **MATLAB** (Octave support deferred, FR-INS-13). Second engine (Q-01). | S | Reference scenario processes end-to-end with MATLAB R2026a. |
| FR-ANA-04 | The system shall estimate and remove the **DEM error** (spatially uncorrelated look-angle error) per point. | M | DEM error layer exported; synthetic test with injected DEM error recovers it within tolerance. |
| FR-ANA-05 | The system shall compute per point: mean LOS velocity, velocity standard deviation, displacement time series, temporal coherence (or ensemble coherence), residual height, and point type (PS/DS). | M | All attributes present in the result schema. |
| FR-ANA-06 | The system shall reference all results to a **reference point/area** (user-defined per FR-AOI-07, or automatic: highest coherence + lowest velocity variance near the AOI centre or in a user-drawn stable zone). | M | Mean velocity inside the reference area = 0 ± 0.1 mm/yr; the reference is shown in outputs. |
| FR-ANA-07 | The system shall offer **atmospheric correction**: (a) spatio-temporal filtering (engine built-in), (b) ERA5 via PyAPS (optional, needs CDS credentials), (c) GACOS (optional, user-supplied files). | M (a), S (b), C (c) | With (b) on, the tropospheric delay rasters are stored and the reduction in residual RMS is reported. |
| FR-ANA-08 | The system shall be able to decompose **ascending + descending** LOS velocities into **vertical and east-west** components on a common grid. | S | Result checked against EGMS Ortho within NFR-ACC-01 tolerances. |
| FR-ANA-09 | The PSDS engine shall support **incremental updates** (new acquisitions processed with compressed history, without re-running the whole stack). | S | Adding acquisitions to the reference scenario takes < 30 % of a full re-run and gives velocities within 1 mm/yr of a full re-run. |
| FR-ANA-10 | The system shall support fitting a **deformation model** per point for reporting: linear (default), linear + seasonal (annual sinusoid), optionally step at a given date. | S | Model parameters and residuals exported. |
| FR-ANA-11 | The system shall let users exclude interferograms/dates from the inversion after the first run (e.g. outliers) and re-run the affected stages. | S | Excluded dates recorded and shown. |
| FR-ANA-12 | **Engine "SBAS" (MintPy)**: small-baseline time-series analysis of the FR-PRE-09 network: network modification (coherence-based pair exclusion), unwrapping-error correction, network inversion (weighted), temporal-coherence masking, DEM error, optional ERA5 and deramping; results converted into the standard result schema (pixels as `SBAS` points + rasters). | M | Reference scenario processes end-to-end; velocities compared with EGMS and the PSInSAR engine (NFR-ACC-01/02). |
| FR-ANA-13 | StaMPS small-baseline and merged PS+SBAS modes as options of the StaMPS engine (MATLAB). | C | – |
| FR-ANA-14 | **Open-source processing**: every technique (PSInSAR, SBAS) shall be available using only open-source software; proprietary components (MATLAB) are optional alternatives. | M | A fresh installation without MATLAB runs the reference scenario with both techniques. |

## 7. Geocoding (FR-GEO) – stage 9

| ID | Requirement | Prio | Acceptance criteria |
|----|-------------|:----:|---------------------|
| FR-GEO-01 | Point results shall be geocoded with **precise per-point coordinates** (from lat/lon lookup tables, corrected for the estimated DEM error / height). | M | Known corner reflector or distinctive building PS within 1 resolution cell (~5 × 20 m) of the true position `[TBC S-2/S-3]`. |
| FR-GEO-02 | Raster results shall be resampled onto a regular geographic grid (EPSG:4326) or a user-chosen projected CRS (e.g. UTM, national grids like EPSG:25832). | M | GeoTIFFs open correctly in QGIS with the correct CRS. |
| FR-GEO-03 | The system shall export the **LOS unit vector** (east, north, up components) per point/pixel. | M | Components present and have unit norm. |

## 8. Visualisation (FR-VIS) – UC-05

| ID | Requirement | Prio | Acceptance criteria |
|----|-------------|:----:|---------------------|
| FR-VIS-01 | Interactive map with points coloured by mean velocity (diverging colour scale, symmetric range, user-adjustable), legend with units (mm/yr) and sign convention. | M | Map with 1 million points stays responsive (NFR-PERF-04). |
| FR-VIS-02 | Clicking a point shows its **time series** (displacement in mm vs. date, error bars if available, fitted model) and attributes. | M | Plot shown < 1 s after click. |
| FR-VIS-03 | Filters: temporal coherence threshold, velocity range, point type (PS/DS), spatial selection. | M | Filters update the map live. |
| FR-VIS-04 | Selection of several points / polygon → mean time series and statistics. | S | |
| FR-VIS-05 | QA layers: mean amplitude, D_A, coherence, point density, velocity std. dev., reference point, AOI. | M | |
| FR-VIS-06 | Profile tool: velocity along a user-drawn line. | C | |
| FR-VIS-07 | Stack overview: acquisition timeline, baseline plot, per-date quality (mean coherence, excluded dates). | S | |
| FR-VIS-08 | Displacement animation over time (time slider). | C | |

## 9. Export & reporting (FR-EXP) – UC-06

| ID | Requirement | Prio | Acceptance criteria |
|----|-------------|:----:|---------------------|
| FR-EXP-01 | Point results as **GeoPackage** (default) with one row per point: coordinates, height, velocity, std. dev., coherence, type, LOS vector, and time series (as columns `D_YYYYMMDD`, as EGMS does). | M | Opens in QGIS; attribute schema matches [06](06_data_and_interface_requirements.md) §4.2. |
| FR-EXP-02 | **CSV** export with the same content (EGMS-like layout). | M | |
| FR-EXP-03 | **GeoTIFF** rasters as COG (Cloud Optimized GeoTIFF): velocity, std. dev., coherence, DEM error, per-date displacement (multi-band). | M | Valid COG (checked with `rio cogeo validate`). |
| FR-EXP-04 | **KMZ for Google Earth** (Q-09): (a) points coloured by velocity with a colour-bar screen overlay and a legend (units, sign convention, reference point); (b) click balloon with attributes and a **time-series chart**; (c) level-of-detail (KML *Regions*/tiling) or aggregation so large point sets stay usable in Google Earth; (d) optional velocity raster as a ground overlay. | M | Opens in Google Earth Pro and Google Earth Web; 500 k points stay navigable; balloons show the chart `[VERIFY chart approach in S-5]`. |
| FR-EXP-05 | ESRI Shapefile export (for legacy GIS; time series truncated to field limits with a warning). | C | |
| FR-EXP-06 | Every export shall carry **metadata**: project config, software and engine versions, data IDs, reference point, sign convention, units, CRS, processing date. | M | Metadata embedded (GeoTIFF tags, GPKG metadata table) and as a sidecar JSON. |
| FR-EXP-08 | **Self-contained web viewer export** (Q-09): a folder/zip with a static HTML page (MainSAR's own viewer: velocity map, click-to-plot time series, filters, legend) plus the result data. Opens in any modern browser **without MainSAR installed**, e.g. to share with supervisors or attach to a thesis. | S | Opens offline from the file system or a static web host; reference scenario loads in ≤ 5 s. |
| FR-EXP-07 | Automatic **processing report** (HTML, optionally PDF): AOI map, stack summary, baseline plot, parameters, QA statistics, velocity map, sample time series, known limitations, data attributions. | S | Report produced for the reference scenario and readable standalone. |

## 10. Validation support (FR-VAL) – UC-08

| ID | Requirement | Prio | Acceptance criteria |
|----|-------------|:----:|---------------------|
| FR-VAL-01 | Import of **EGMS** products (CSV/GPKG download for the AOI) as a reference layer. | S | EGMS L2a/L2b/L3 files for the AOI import and display. |
| FR-VAL-02 | Import of **GNSS** time series (CSV, NGL tenv3) and projection onto the LOS. | S | |
| FR-VAL-03 | **Comparison** of two results (engine vs. engine, PSInSAR vs. SBAS, engine vs. EGMS/GNSS): spatial matching (nearest neighbour within radius / common grid), bias, RMSE, correlation, scatter plot, difference map. | S | Statistics reproduce a hand-computed test case. |
| FR-VAL-04 | Time-period alignment and re-referencing of the compared datasets to a common reference area. | S | |

## 11. Configuration (FR-CFG) – UC-03

| ID | Requirement | Prio | Acceptance criteria |
|----|-------------|:----:|---------------------|
| FR-CFG-01 | All parameters shall have documented defaults, units, valid ranges and help text ([07](07_processing_parameters.md)); the UI and CLI take them from **one parameter schema** (single source of truth). | M | UI forms are generated from the schema; no parameter is defined twice. |
| FR-CFG-02 | **Presets**: *Urban/Infrastructure*, *Rural/Natural terrain*, *Fast preview* (lower resolution, fewer dates). | M | Presets load; changes relative to a preset are highlighted. |
| FR-CFG-03 | **Expert mode** showing all engine parameters, including direct editing of engine-native parameter files. | S | |
| FR-CFG-04 | Validation of parameter combinations with understandable error messages before a job starts. | M | Invalid combinations (e.g. reference date outside range) are rejected pre-flight. |
| FR-CFG-05 | Changing a parameter shall mark **only the dependent stages** as outdated. | M | Changing the coherence threshold re-runs only analysis/post-processing stages, not download or coregistration. |
| FR-CFG-06 | Import/export of parameter sets (YAML) independent of projects. | S | |
| FR-CFG-07 | Global settings: data/cache directories, resource limits, credentials, UI language. | M | |

## 12. Job execution (FR-JOB) – UC-04, UC-07

| ID | Requirement | Prio | Acceptance criteria |
|----|-------------|:----:|---------------------|
| FR-JOB-01 | Processing runs as **background jobs** independent of the UI session (closing the browser does not stop the job). | M | |
| FR-JOB-02 | Progress reporting per stage and overall (%), ETA, current step, live log stream. | M | Updates at least every 5 s while a stage is running. |
| FR-JOB-03 | **Checkpoint/resume**: every stage is idempotent and records completion; a failed or interrupted job resumes from the first incomplete stage. | M | Kill test at a random time in each stage → resume → same result as an uninterrupted run. |
| FR-JOB-04 | **Cancel** a running job cleanly (child processes, e.g. MATLAB, SNAPHU, terminated; partial outputs of the running stage discarded). | M | No orphan processes after cancel. |
| FR-JOB-05 | **Re-run from stage X** on user request. | M | |
| FR-JOB-06 | Job queue with configurable concurrency (default: 1 heavy job at a time; light jobs such as searches and exports run in parallel). | M | Two queued heavy jobs run one after another with the default setting. |
| FR-JOB-07 | **Pre-flight checks** before start: credentials valid, free disk ≥ estimate + margin, required tools available, parameters valid. | M | Each failing check blocks the start with a specific message. |
| FR-JOB-08 | Notification on completion/failure (UI; optional e-mail/webhook). | S (UI), C (e-mail) | |
| FR-JOB-09 | ~~Remote execution on other machines~~ – *withdrawn*: one production machine (Q-05). | W | – |

## 13. Command line (FR-CLI) – UC-09

| ID | Requirement | Prio | Acceptance criteria |
|----|-------------|:----:|---------------------|
| FR-CLI-01 | CLI with at least: `init`, `search`, `estimate`, `run [--from-stage] [--to-stage]`, `status`, `resume`, `cancel`, `export`, `clean`, `validate-config`. | M | `--help` for all commands; tested in CI. |
| FR-CLI-02 | Machine-readable output (`--json`) and meaningful exit codes. | M | |
| FR-CLI-03 | CLI and UI use the same core library (no duplicated logic). | M | Architecture review. |
| FR-CLI-04 | Worker commands can be run standalone with a request file (for debugging and scripting); result files (GeoParquet/COG/NetCDF) are documented so they can be analysed in Python/R/MATLAB notebooks. | S | Documented example runs. |

## 14. Settings & credentials (FR-SET) – UC-10

| ID | Requirement | Prio | Acceptance criteria |
|----|-------------|:----:|---------------------|
| FR-SET-01 | Credential management for NASA Earthdata (ASF), Copernicus Data Space (orbits, if needed) and CDS (ERA5), with a **"test connection"** button. | M | Invalid credentials are reported clearly. |
| FR-SET-02 | Credentials stored securely (OS keyring, or encrypted file / environment variables for headless use), never in project files or logs. | M | Grep over logs/config of a test run finds no secrets. |
| FR-SET-03 | Resource limits: max CPU cores, max RAM per job, disk quota / minimum free space. | M | Limits enforced (NFR-RES-01…03). |
| FR-SET-04 | Proxy configuration for downloads. | C | |
| FR-SET-05 | Health/status page: tool versions (ISCE2, engine, SNAPHU, MATLAB/Octave), connectivity, disk space. | S | |

## 15. Logging & QA (FR-LOG)

| ID | Requirement | Prio | Acceptance criteria |
|----|-------------|:----:|---------------------|
| FR-LOG-01 | Structured logs (JSON lines) per job and stage, with levels; third-party tool output captured into the stage log. | M | |
| FR-LOG-02 | Automatic **QA metrics** per stage (e.g. number of PS/DS, mean coherence per date, ESD residuals, unwrapping closure errors, residual RMS), with **plausibility warnings** (e.g. fewer than N points/km², unwrapping errors > X %). | M | QA metrics appear in the report and UI. |
| FR-LOG-03 | Downloadable "debug bundle" (logs, config, versions, QA metrics; no credentials, no large data). | S | |

---

## 16. Installation (FR-INS)

Installation requirements FR-INS-01…12 are specified in [14](14_installation_and_operations.md) §4.

## 17. Traceability: use cases → requirements

| Use case | Requirements |
|----------|--------------|
| UC-01 | FR-PRJ-01…03, FR-AOI-01…07 |
| UC-02 | FR-DIS-01…07 |
| UC-03 | FR-CFG-01…07 |
| UC-04 | FR-JOB-01…09, FR-ACQ-*, FR-PRE-*, FR-ANA-*, FR-GEO-* |
| UC-05 | FR-VIS-01…08 |
| UC-06 | FR-EXP-01…08 |
| UC-07 | FR-PRJ-04, FR-PRJ-06, FR-ACQ-09, FR-CFG-05, FR-JOB-05, FR-PRE-07, FR-ANA-09, FR-ANA-11 |
| UC-08 | FR-VAL-01…04 |
| UC-09 | FR-CLI-01…04 |
| UC-10 | FR-SET-01…05, FR-INS-01…12 |

Goals → requirements: G-1 → FR-JOB-*, FR-ACQ-*; G-2 → FR-ACQ-01, FR-PRE-02, FR-ANA-09, NFR-EFF-*;
G-3 → FR-ANA-*, FR-VAL-*, NFR-ACC-*; G-4 → FR-PRJ-02, NFR-REP-*; G-5 → FR-CFG-*, FR-VIS-*, NFR-USE-*;
G-6 → FR-JOB-03, NFR-REL-*; G-7 → FR-ANA-01, NFR-MAINT-*.
