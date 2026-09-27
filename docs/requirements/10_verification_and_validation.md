# 10 – Verification & Validation Plan

**Verification**: did we build the system right (requirements met, code correct)?
**Validation**: did we build the right system (are the displacement results *true*)?

## 1. Test levels

| Level | Scope | Data | When | Tools |
|-------|-------|------|------|-------|
| Unit | TS core (config, hashing, orchestrator, estimator) and Python workers (geometry, statistics, exporters) | In-memory, synthetic | Every commit (CI) | Vitest; pytest + hypothesis |
| Contract | Worker contract (TS core ↔ fake workers; workers ↔ request fixtures); plugin interfaces against recorded API responses | Fixtures | Every commit | Vitest, pytest |
| Installation | Installer on a fresh Ubuntu 24.04 VM, `mainsar doctor --full` | DS-CI subset | Nightly / per release | CI VM |
| UI end-to-end | Browser workflows against a backend with fake workers | Fixtures | Every commit | Playwright |
| Integration | Real tools on a **tiny real dataset** (DS-CI) | DS-CI | Nightly / pre-merge for pipeline changes | pytest (pixi env) + Vitest |
| End-to-end | Full pipeline incl. download on the reference scenario | DS-REF | Before each release; manual trigger | CLI + scripted checks |
| Scientific validation | Result accuracy vs. independent references | DS-REF + EGMS + GNSS | Per release and per engine change | Validation notebooks + FR-VAL tools |
| Performance | Runtime, peak RAM, disk on HW-PROD (and DS-CI on HW-DEV) | DS-CI, DS-SMALL, DS-REF | Per release | Stage metrics from the state DB |
| Robustness | Kill/resume, network failures, disk full, corrupt input | DS-CI | Nightly | Fault injection |
| Usability | Persona-P1 walkthrough | DS-SMALL | Before v1.0 | Moderated test with ≥ 2 users |

## 2. Test datasets

| ID | Description | Size | Purpose |
|----|-------------|------|---------|
| DS-SYN | **Synthetic stacks** with known deformation (linear, seasonal, step), known DEM error, simulated atmosphere (fractal), noise, PS/DS mix | MB | Ground truth for algorithm verification (NFR-ACC-05), engine adapters, post-processing |
| DS-CI | Tiny real stack: 1 burst cropped to ~2 × 2 km, 15–20 dates, stored in a test data repository/object store | < 2 GB | Integration tests, kill/resume tests |
| DS-SMALL | 5 × 5 km, ~60 dates, 1 track | ~20–40 GB peak | Fast realistic runs on HW-PROD, usability tests |
| DS-REF | **Reference scenario** (§4) | ~100–200 GB peak | End-to-end, performance, scientific validation |

## 3. Scientific validation approach

1. **Synthetic ground truth** (DS-SYN): engines must recover the injected signals within tolerances
   (e.g. velocity error < 0.5 mm/yr at SNR typical of PS).
2. **EGMS comparison** (for European AOIs): compare with EGMS L2b *Calibrated* for the same track
   (or L3 Ortho after decomposition). Common reference area, overlapping time period, nearest-neighbour
   matching (≤ 20 m) or 100 m grid aggregation. Metrics: bias, RMSE, Pearson r, share of points within
   ±2 mm/yr. Target: NFR-ACC-01.
3. **GNSS comparison**: stations inside the AOI, 3D GNSS velocities projected onto the LOS vector,
   compared with the mean of PS/DS within 200 m of the station. Target: NFR-ACC-01.
4. **Cross-engine / cross-technique comparison**: Dolphin vs. StaMPS on common points; PSInSAR vs. SBAS aggregated
   to the SBAS grid. Target: NFR-ACC-02.
5. **Literature site**: reproduce a published deformation signal (e.g. known mining subsidence bowl)
   qualitatively (location, extent, order of magnitude).
6. **Sanity checks** (automatic, every run): stable-area velocity distribution centred at 0, mean closure
   phase / unwrapping error rate, no burst-boundary discontinuities, sign check (known subsidence → negative).

## 4. Reference scenarios (DS-REF, DS-STRESS): confirmed (Q-10)

### 4.1 Selection criteria

1. In Europe with **EGMS** coverage (L2a/L2b/L3) for independent comparison.
2. **≤ 100 km²** AOI that matches the typical study size.
3. A **known, published deformation signal**, so a wrong sign, magnitude or location is obvious.
4. **GNSS stations** with public time series inside or near the AOI.
5. A mix of land cover (urban + agricultural/natural) to exercise both PS and DS.
6. Covered by ascending **and** descending tracks, with S1B data in the archive period.

### 4.2 Candidate sites

Facts below are from memory of the literature and are checked in spike S-1 (`[VERIFY]`: EGMS tiles, GNSS
station IDs, burst coverage, current rates) before the final choice.

| # | Site | Phenomenon | Expected signal `[VERIFY]` | What it tests | Pros | Cons |
|---|------|-----------|----------------------------|---------------|------|------|
| **A** | **Bologna, Italy** (city + surrounding plain, ~10 × 10 km) | Subsidence from groundwater extraction | Up to ~1–2 cm/yr in parts of the city/plain, stable areas nearby | Urban PS density, moderate rates, GNSS comparison, reference-area handling | Classic, well-documented InSAR benchmark; dense PS; permanent GNSS in the city; EGMS coverage | Signal partly slowing in recent years |
| **B** | **Alto Guadalentín basin / Lorca, Spain** | Aquifer overexploitation subsidence | Among the fastest in Europe, several cm/yr up to ~10 cm/yr at the centre of the basin | Unwrapping under large rates, DS in agricultural land, sign/magnitude sanity | Very strong, unambiguous signal; well published | Rapid rates stress unwrapping (intended); agricultural decorrelation |
| **C** | **Rhenish lignite mining area, e.g. around Hambach/Elsdorf, Germany** | Subsidence from mining-related groundwater lowering | mm/yr to cm/yr at pit margins; decorrelation inside the pits | Mixed land cover, strong gradients, German reference (SAPOS GNSS) | Close to the team; relevant to German students; EGMS + SAPOS | Complex signals (pit edges, active mining changes); GNSS data access may need a licence |
| **D** | **Groningen gas field, Netherlands** (subset) | Subsidence from gas extraction (production ended 2023/24) | Slow, broad bowl of mm/yr, declining | Accuracy for small rates, DS density in flat rural land, dense GNSS/levelling | Very well monitored; good for NFR-ACC checks | Bowl larger than 100 km²; a small subset needs a careful reference area |

### 4.3 Decision (confirmed by the product owner)

- **DS-REF (primary acceptance scenario): A – Bologna.** Balanced urban/rural mix, moderate signal, good GNSS and
  EGMS coverage. It matches the typical user study most closely.
- **DS-STRESS (secondary scenario): B – Lorca / Alto Guadalentín.** Stress test for unwrapping and DS processing.
- **DS-CI:** a ~2 × 2 km crop of site A with 15–20 dates.
- C and D stay optional extra validation sites.

### 4.4 Scenario parameters

| Property | Proposal |
|----------|----------|
| Tracks | 1 ascending + 1 descending (descending-only for the MVP) |
| Period | 5 years (e.g. 2021-01-01 → 2025-12-31) → ~200–300 dates depending on revisit (worst case of the sizing) |
| Hardware | HW-PROD |

## 5. Acceptance tests (selection, one per key requirement group)

| ID | Test | Covers | Pass criterion |
|----|------|--------|----------------|
| AT-01 | Create a project by importing the reference AOI (GeoPackage in EPSG:25832) | FR-AOI-02, FR-PRJ-01 | AOI reprojected correctly (vertex error < 1 m) |
| AT-02 | Search reference AOI 2023–2025 | FR-DIS-01…04 | Tracks and counts match ASF Vertex; estimate shown; warnings correct |
| AT-03 | Download with interruption after 50 % | FR-ACQ-04, NFR-REL-01 | Resumes; no complete file re-downloaded; checksums valid |
| AT-04 | Burst-only download volume | FR-ACQ-01, NFR-EFF-01 | Volume ≤ bursts × size + orbits |
| AT-05 | Coregistration quality | FR-PRE-01 | No phase jumps at burst boundaries; ESD residuals logged and within threshold |
| AT-06 | End-to-end run (default engine, default preset) | G-1, FR-JOB-*, FR-ANA-* | Completes without manual intervention; outputs in schema |
| AT-07 | Kill at random time in each stage + resume | FR-JOB-03 | Final result identical to an uninterrupted run (NFR-REP-01 tolerance) |
| AT-08 | Parameter change → partial re-run | FR-CFG-05 | Only dependent stages re-run (verified via stage log) |
| AT-09 | EGMS comparison | NFR-ACC-01 | Within tolerances |
| AT-19 | SBAS run of DS-REF without MATLAB installed | FR-ANA-12, FR-ANA-14, FR-PRE-09 | Completes; velocities within NFR-ACC-01 vs. EGMS and NFR-ACC-02 vs. PSInSAR |
| AT-10 | Viewer with 1 M points | NFR-PERF-04, FR-VIS-01…03 | First render ≤ 3 s, filter ≤ 200 ms |
| AT-11 | Exports open in QGIS, metadata complete | FR-EXP-01…06 | Visual + schema check |
| AT-17 | KMZ of DS-REF in Google Earth Pro/Web | FR-EXP-04 | Colours, legend and balloons with time-series chart correct; navigation usable |
| AT-18 | Reprocess DS-REF with a changed coherence threshold, then extend by 5 dates | FR-PRJ-06, FR-ACQ-09, FR-PRE-07 | First: no download/coregistration; second: only 5 dates downloaded/coregistered |
| AT-12 | Secrets scan over logs/configs/exports | NFR-SEC-01 | No secrets found |
| AT-13 | DS-CI run on HW-DEV with a 4 GB memory limit | NFR-PERF-02, NFR-RES-01 | Completes; peak RAM ≤ limit |
| AT-14 | CLI headless run reproduces UI run | FR-CLI-*, NFR-REP-01 | Identical results |
| AT-15 | Fresh-VM installation with and without MATLAB | FR-INS-01…06, NFR-PORT-02 | Installs with one command; `mainsar doctor --full` passes; capability summary correct |
| AT-16 | Local API rejects cross-origin requests and is not reachable from another host | NFR-SEC-02/03 | Requests rejected; no open port |

## 6. Definition of Done (per feature)

- Requirement ID(s) referenced in the merge request.
- Code typed, linted, documented (docstrings); tests added (unit and, where relevant, integration).
- CI green; coverage of changed code ≥ 80 %.
- User-facing change: help text / user guide updated.
- Scientific change: validation notebook re-run and results attached.
