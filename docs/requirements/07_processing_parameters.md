# 07 – Processing Parameter Catalogue

This catalogue is the basis for the **parameter schema** (FR-CFG-01), which generates UI forms, CLI
validation and documentation. Values follow the defaults of the underlying tools as far as known;
**all defaults are `[VERIFY]` against the pinned tool versions in spikes S-1…S-3.**

Columns: **Level** = *B*asic (always shown), *A*dvanced (expert mode). **Stage** = pipeline stage (see [02](02_domain_and_method_analysis.md) §2).

## 1. Project & data selection

| Parameter | Level | Default | Range / options | Description |
|-----------|:-----:|---------|-----------------|-------------|
| `aoi` | B | – | Polygon | Area of interest |
| `aoi_buffer_m` | A | 1000 | 0–10000 | Buffer around the AOI |
| `max_aoi_km2` (global setting) | A | 100 warn / 250 block | 1–10000 | FR-AOI-05 |
| `start_date`, `end_date` | B | – | ≥ 2014-10-03; span ≤ 5 years (warning above) | Time range |
| `platform` | – | S1 | S1 | Sensor (fixed in v1) |
| `relative_orbit` | B | chosen from search | 1–175 | Track |
| `flight_direction` | B | from track | ASC/DESC | |
| `polarisation` | A | VV | VV, VH, HH, HV | VV recommended for PS |
| `exclude_dates` | A | [] | list of dates | Manually excluded acquisitions |
| `exclude_months` | A | [] | 1–12 | Rule-based exclusion (e.g. snow) |
| `min_acquisitions` | A | 20 | 2–1000 | Warning threshold (FR-DIS-04) |
| `max_gap_days` | A | 48 | 1–1000 | Largest gap between usable dates before a warning (FR-DIS-04) |

## 2. Acquisition

| Parameter | Level | Default | Range / options | Description |
|-----------|:-----:|---------|-----------------|-------------|
| `download.mode` | A | burst | burst, scene | Burst extraction vs. full SLC |
| `download.concurrency` | A | 4 | 1–16 | Parallel downloads |
| `download.retries` | A | 5 | 0–20 | With exponential back-off |
| `orbit.type` | A | precise | precise, restituted-fallback | |
| `dem.source` | A | COP30 | COP30, SRTM1, custom | |
| `dem.custom_path` | A | – | GeoTIFF | + vertical datum |

## 3. Preprocessing (ISCE2 topsStack)

| Parameter | Level | Default | Range / options | Description |
|-----------|:-----:|---------|-----------------|-------------|
| `reference_date` | B | auto | date in stack | Single reference ("super master") |
| `coregistration` | A | NESD | geometry, NESD | NESD = geometry + ESD (recommended for TOPS; needs ≥ 2 bursts in azimuth, FR-PRE-10) |
| `isce_workflow` | – | slc | slc | Fixed: ISCE2 is used up to the merged coregistered SLCs only; SBAS interferograms are formed from the cropped stack (FR-PRE-09) |
| `esd_coherence_threshold` | A | 0.85 | 0.5–0.99 | Coherence threshold for ESD |
| `num_overlap_connections` | A | 3 | 1–10 | Pairs used for ESD overlap estimation |
| `swaths` | A | auto from AOI | 1,2,3 | Sub-swaths to process |
| `range_looks` / `azimuth_looks` | A | 1 / 1 (PS) | 1–20 / 1–5 | Full resolution for PS; multilook only for previews |
| `keep_intermediates` | A | false | bool | FR-PRE-08 |
| `retention` | A | keep-stack | keep-raw, keep-stack, results-only | FR-ACQ-09 |
| `num_processes` | A | from resource limits | 1–N | Parallelism |

## 4. Engine "PSDS" (Dolphin)

MainSAR's names (block `dolphin` in `project.yaml`, schema `schemas/dolphin-parameters.schema.json`), mapped to
Dolphin 0.42 by the adapter (increment 5; defaults verified in spike S-2).

| Parameter | Level | Default | Range / options | Description |
|-----------|:-----:|---------|-----------------|-------------|
| `ps.amp_dispersion_threshold` | B | 0.25 | 0.1–0.5 | D_A threshold for PS candidates; lower = fewer, more reliable PS |
| `phase_linking.half_window` | A | x=11, y=5 | 1–50 | SHP search window (range/azimuth half-size), ~matched to 2.3×14 m pixels |
| `phase_linking.ministack_size` | A | 15 | 5–50 | Dates per ministack (sequential estimator; memory vs. accuracy) |
| `phase_linking.shp_method` | A | glrt | glrt, ks, rect | SHP test |
| `phase_linking.shp_alpha` | A | 0.005 | 0.001–0.1 | Significance level of the SHP test |
| `phase_linking.use_evd` | A | false | bool | EVD instead of EMI |
| `network_bandwidth` | A | 3 | 1–10 | Unwrapping network: each date paired with its N following dates (nearest-N; 1 = sequential). Nearest-3 as verified in S-2: the redundancy lets the L1 inversion reject unwrapping errors (OQ-1) |
| `unwrap_method` | B | snaphu | snaphu (spurt, phass, icu once their packages are in the environment) | Phase unwrapping algorithm |
| `unwrap.ntiles` | A | auto | 1×1 … 8×8 | Tiling for SNAPHU (memory) |
| `unwrap.cost` | A | smooth | smooth, defo | SNAPHU statistical cost mode |
| `timeseries.method` | A | L1 | L1, L2 | Network inversion norm |
| `output.coherence_threshold` | B | 0.7 | 0.3–0.99 | Minimum temporal coherence for exported points |
| `worker.block_shape` | A | auto | px | Block size for memory control, derived from the memory budget: 1 GB base + block² × N² × 8 B × 50 (N = ministack + compressed SLCs); calibrated on S-2, recalibrated in S-4 |
| `worker.threads_per_worker` | A | auto | 1–N | |
| `worker.gpu_enabled` | A | auto (on if a CUDA GPU is detected) | auto, true, false | GPU acceleration for Dolphin (RTX 3080 Ti on HW-PROD) |

## 5. Engine "StaMPS-PS"

Names are the original StaMPS parameter names (`setparm`). Defaults per StaMPS 4.1b `ps_parms_default` `[VERIFY S-3]`.

### 5.1 Export / preparation (`mt_prep_isce`)

| Parameter | Level | Default | Range | Description |
|-----------|:-----:|---------|-------|-------------|
| `da_thresh` | B | 0.4 | 0.3–0.6 | Initial D_A threshold for PS candidates (loose; phase-based selection follows) |
| `range_patches` / `azimuth_patches` | A | auto (from RAM) | 1–20 | Patch grid (memory control) |
| `range_overlap` / `azimuth_overlap` | A | 50 / 50 | 0–200 px | Patch overlap |

### 5.2 Processing parameters

| Parameter | Level | Default | Range | Step | Description |
|-----------|:-----:|---------|-------|:----:|-------------|
| `max_topo_err` | B | 20 | 5–100 m | 2 | Max. uncorrelated DEM error; larger = more candidates, slower |
| `filter_grid_size` | A | 50 | 20–200 m | 2 | Grid for spatial phase filtering |
| `filter_weighting` | A | P-square | P-square, SNR | 2 | |
| `clap_win` | A | 32 | 8–64 | 2 | CLAP filter window |
| `clap_low_pass_wavelength` | A | 800 | 200–2000 m | 2 | |
| `clap_alpha` / `clap_beta` | A | 1 / 0.3 | – | 2 | CLAP filter weights |
| `gamma_change_convergence` | A | 0.005 | 0.001–0.05 | 2 | Convergence criterion |
| `gamma_max_iterations` | A | 3 | 1–10 | 2 | |
| `select_method` | A | DENSITY | DENSITY, PERCENT | 3 | |
| `density_rand` | A | 20 | 1–100 | 3 | Max. random-phase PS density per km² |
| `weed_standard_dev` | B | 1.0 | 0.5–2 rad | 4 | Weeding threshold (noise) |
| `weed_time_win` | A | 730 | days | 4 | |
| `weed_max_noise` | A | inf | rad | 4 | |
| `weed_zero_elevation` | A | n | y/n | 4 | Drop points at zero height (sea) |
| `weed_neighbours` | A | n | y/n | 4 | |
| `merge_resample_size` | A | 0 | 0–500 m | 5 | 0 = no resampling (keep full PS resolution) |
| `unwrap_method` | A | 3D | 3D, 3D_QUICK | 6 | |
| `unwrap_grid_size` | A | 200 | 50–500 m | 6 | |
| `unwrap_time_win` | A | 730 | days | 6 | |
| `unwrap_gold_n_win` | A | 32 | 8–64 | 6 | Goldstein filter window |
| `unwrap_gold_alpha` | A | 0.8 | 0–1 | 6 | |
| `unwrap_prefilter_flag` | A | y | y/n | 6 | |
| `scla_deramp` | A | n | y/n | 7 | Remove orbital ramps |
| `scn_wavelength` | A | 100 | 50–1000 m | 8 | Spatially correlated noise (APS) filter wavelength |
| `scn_time_win` | A | 365 | days | 8 | APS temporal filter window |
| `ref_centre_lonlat` / `ref_radius` | B | from reference area | – | 6+ | Reference point/area |
| `subtr_tropo` / `tropo_method` | A | n / a_l | – | 7/8 | TRAIN-based tropospheric correction (optional) |

### 5.3 Runtime

| Parameter | Level | Default | Options | Description |
|-----------|:-----:|---------|---------|-------------|
| `stamps.runtime` | A | matlab | matlab (octave later, FR-INS-13) | |
| `stamps.max_threads` | A | from resource limits | 1–N | MATLAB `maxNumCompThreads` |
| `stamps.random_seed` | A | 0 | int | NFR-REP-04 |

## 6. Corrections & post-processing (engine-independent)

| Parameter | Level | Default | Options | Description |
|-----------|:-----:|---------|---------|-------------|
| `reference.mode` | B | auto | auto, point, area | FR-ANA-06 |
| `reference.lonlat` / `reference.radius_m` | B | – | – | |
| `troposphere.method` | B | none | none, era5, gacos | Engine filtering stays on independently |
| `deramp` | A | none | none, linear, quadratic | Residual ramp removal |
| `model.type` | A | linear | linear, linear+seasonal, +step | FR-ANA-10 |
| `model.step_dates` | A | [] | dates | |
| `output.crs` | B | EPSG:4326 | any EPSG | FR-GEO-02 |
| `output.raster_resolution_m` | A | 20 | 5–500 | Raster products grid |
| `output.formats` | B | [gpkg, geotiff, csv, kmz] | gpkg, csv, geotiff, kmz, shp, netcdf | |
| `output.min_coherence` | B | 0.7 (PSDS) / n.a. (StaMPS: weeding) | 0–1 | |

## 6a. Engine "SBAS" (ISCE2 network + MintPy)

Names follow ISCE2 `stackSentinel.py` and MintPy `smallbaselineApp.cfg` `[VERIFY S-7]`.

### 6a.1 Interferogram network (FR-PRE-09)

| Parameter | Level | Default | Range / options | Description |
|-----------|:-----:|---------|-----------------|-------------|
| `num_connections` | B | 3 | 1–10 | Each date is paired with its *n* nearest successors |
| `max_temporal_baseline_days` | A | none | 6–2000 | Optional additional pair restriction |
| `max_perp_baseline_m` | A | none | 10–500 | Optional additional pair restriction |
| `range_looks` / `azimuth_looks` | B | 9 / 3 | 1–40 / 1–10 | ≈ 30 × 40 m ground pixels, suitable for AOIs ≤ 100 km² |
| `filter_strength` | A | 0.5 | 0–1 | Goldstein filter before unwrapping; reduced SBAS noise from 1.8 to 1.2 mm per date on DS-CI (spike S-7) |
| `unwrap_method` | A | snaphu | snaphu, icu | |

### 6a.2 MintPy time series

| Parameter | Level | Default | Range / options | Description |
|-----------|:-----:|---------|-----------------|-------------|
| `mintpy.network.coherenceBased` | A | **no** | yes/no | Drop low-coherence pairs. Off by default: on DS-CI it reduced the network from 51 to 23 pairs (almost a spanning tree), which more than doubled the noise and caused unwrapping errors (spike S-7) |
| `mintpy.network.minCoherence` | A | 0.5 | 0–1 | Only used if `coherenceBased` = yes |
| `mintpy.unwrapError.method` | A | no | no, bridging, phase_closure, bridging+phase_closure | Unwrapping-error correction |
| `mintpy.networkInversion.weightFunc` | A | var | var, fim, coh, no | Pair weighting |
| `mintpy.networkInversion.minTempCoh` | B | 0.7 | 0–1 | Temporal-coherence mask threshold |
| `mintpy.topographicResidual` | A | yes | yes/no | DEM error correction |
| `mintpy.troposphericDelay.method` | B | no | no, pyaps (ERA5), gacos | Mapped from `troposphere.method` |
| `mintpy.deramp` | A | no | no, linear, quadratic | Mapped from `deramp` |
| `mintpy.reference.lalo` | B | from reference area | lat, lon | Mapped from `reference.*` |
| `mintpy.reference.date` | A | first date | date | Set explicitly: MintPy's `auto` moved the reference date to the 2nd date in S-7; MainSAR's convention is the first date (DR 3.2) |

## 7. Presets (FR-CFG-02) – initial proposal

| Preset | Key settings | Use case |
|--------|--------------|----------|
| **Urban / Infrastructure** | PSDS: `amp_dispersion_threshold`=0.25, coherence ≥ 0.75; StaMPS: `da_thresh`=0.4, `weed_standard_dev`=1.0 | Buildings, bridges, railways |
| **Rural / Natural terrain** | PSDS: larger `half_window` (15×7), coherence ≥ 0.5, DS enabled; StaMPS: `da_thresh`=0.45, `density_rand`=20 | Landslides, mining, agriculture margins |
| **Fast preview** | Range/azimuth looks 4×1 for preview only, reduced date set (every 2nd date), coherence ≥ 0.6 | Quick feasibility check before a full run |

## 8. Resource-derived parameters

These are **computed automatically** from the resource limits (FR-SET-03) and the stack size, and are
overridable in expert mode: number of processes (coregistration), Dolphin block size and threads, StaMPS
patch grid, SNAPHU tiles. The derivation rules are designed and calibrated in spike S-4.
