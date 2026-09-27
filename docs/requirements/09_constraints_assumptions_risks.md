# 09 – Constraints, Assumptions & Risks

## 1. Constraints

| ID | Constraint | Source |
|----|------------|--------|
| C-01 | Only freely available input data (Copernicus Sentinel-1, Copernicus DEM, ERA5, EGMS). | Cost; data policy |
| C-02 | Development machine (4 threads, 8 GB RAM, ~65 GB free) is for development and DS-CI tests only; all real processing runs on **one dedicated production machine**: Ryzen 7 3700X (8 C/16 T), 32 GB RAM, 1 TB SSD free, RTX 3080 Ti. | Q-05, Q-19 |
| C-03 | MATLAB uses an **individual, machine-bound licence**, so it runs natively on the host and not in containers. MATLAB is optional; the user supplies the licence. | Q-07 |
| C-04 | Linux as the processing platform (ISCE2, SNAPHU, StaMPS C++ tools). | Tooling |
| C-05 | Accounts needed: NASA Earthdata (downloads), optionally CDSE, CDS (ERA5), EU Login (EGMS). | Data providers |
| C-06 | Sentinel-1 data availability: 6-day revisit with S1A + S1B until Dec 2021; 12-day revisit (S1A only) until S1C became operational in 2025; 6-day again afterwards. Some regions only have one orbit direction. | Mission |
| C-07 | The project is public under GPL-3.0; dependency licences (GPL-3, Apache-2.0, BSD-3, triangle/SNAPHU terms) must be compatible and documented. | Q-11, Q-18 |
| C-08 | Application code in TypeScript; Python only inside processing workers. | Q-17 |
| C-09 | One user at a time; no authentication; localhost binding. | Q-05b |

## 2. Assumptions

| ID | Assumption | Impact if wrong | Check |
|----|------------|-----------------|-------|
| A-01 | AOIs are ≤ 100 km² in Europe, covered by 1–4 bursts per track, with ≤ 5 years of data. | Hardware and defaults need revisiting | Confirmed (Q-04) |
| A-02 | ASF burst extraction keeps working at usable throughput for ~100–300 bursts per project. | Fallback to full scenes: ~10–20× more download and disk | S-1 |
| A-03 | burst2safe output is accepted by ISCE2 topsStack without patches. | Own conversion or use of ISCE3/COMPASS needed | S-1 |
| A-04 | Dolphin can process ISCE2 topsStack coregistered SLCs (radar geometry) with our own geocoding afterwards. | Switch preprocessing to COMPASS for the PSDS engine (two preprocessors) | S-2 |
| A-05 | StaMPS 4.1b runs on MATLAB R2026a with at most minor patches. | Extra effort for porting/patching or Octave-only | S-3 |
| A-06 | Users have basic GIS knowledge and interpret LOS displacement correctly with in-app help. | More UX and training effort | Q-02 |
| A-07 | One user at a time is enough for the foreseeable future. | Auth, per-user isolation and concurrent scheduling would need to be added | Confirmed (Q-05b) |
| A-09 | pixi/conda-forge provide ISCE2, Dolphin, SNAPHU and Node.js for linux-64 in mutually compatible versions. | Build some components from source in the installer | **Confirmed in S-6** (one lockfile, clean Ubuntu 24.04) |
| A-08 | An EGMS-covered reference area (and ideally GNSS stations) is available for validation. | Validation based on engine cross-comparison and literature sites only | Q-10 |

## 3. Risk register

Probability (P) and impact (I): 1 = low … 3 = high. Score = P × I.

| ID | Risk | P | I | Score | Mitigation | Owner |
|----|------|:-:|:-:|:-----:|-----------|-------|
| R-01 | **Production machine at the minimum spec** (32 GB RAM, 1 TB disk): out-of-memory or disk-full on 5-year stacks, especially asc + desc. | 2 | 2 | 4 | Memory-bounded engine settings derived from a 24 GB budget; disk guard (NFR-RES-02); retention policy (FR-ACQ-09); pre-flight disk estimate; recommended upgrades in [14](14_installation_and_operations.md) §1.2. | PO |
| R-02 | **StaMPS incompatibility** with current compilers/MATLAB (unmaintained code base). *Materialised in S-3*: undefined behaviour in the C tools produced silent garbage with GCC 16 (patched); MATLAB R2026a crashes on the Manjaro dev laptop (unsupported platform). | 3 | 2 | 6 | Maintained patch set, build with `-Werror=return-type`, verified exporter + verification scripts; MATLAB validation in the installer; StaMPS stays the second engine (ADR-004); production on a MathWorks-supported distribution. | Dev |
| R-03 | **External API changes / outages** (ASF, CDSE, orbit hosting has moved before, e.g. SciHub shutdown 2023). | 2 | 3 | 6 | Adapter layer (DR-12), multiple sources for orbits/SLCs, schema checks (DR-13), cache. | Dev |
| R-04 | **Dependency hell** (ISCE2 + GDAL + Dolphin + Python versions conflict in one environment). | 3 | 2 | 6 | Spike S-6; pixi lockfile; separate pixi environments per tool group if needed (e.g. `isce`, `dolphin`); installer tested on a fresh VM in CI. | Dev |
| R-05 | **Incorrect results that look plausible** (unwrapping errors, wrong sign, wrong reference). | 2 | 3 | 6 | V&V plan with independent references; cross-engine comparison; QA metrics and plausibility warnings; explicit conventions. | Dev/PO |
| R-06 | **Scope creep** (many engines, sensors, features). | 3 | 2 | 6 | Strict MoSCoW; MVP definition ([11](11_roadmap.md)); change control on the baseline. | PO |
| R-07 | **Licence issues** in the public repo (e.g. bundling `triangle` or SNAPHU sources with non-GPL terms). | 1 | 2 | 2 | The installer downloads such components from upstream instead of vendoring them; licence report in CI (NFR-LEG-01). | Dev |
| R-08 | **Low PS density** in the user's typical AOIs (rural or vegetated). | 2 | 2 | 4 | PS/DS engine (Dolphin); preset tuning; clear expectation-setting in the report. | Dev |
| R-09 | **Performance targets unrealistic** without GPU/large machines. | 2 | 2 | 4 | Calibrate in S-4 before fixing NFR-PERF targets. | Dev |
| R-10 | **Burst ID inconsistencies** (older data before IPF 3.40, orbit/geometry changes). | 2 | 2 | 4 | Footprint-based matching as fallback; pre-flight check FR-DIS-04. | Dev |
| R-11 | **Download volume/time** high over slow connections even with bursts. | 2 | 2 | 4 | Burst mode, concurrency, cache, resumable downloads; option to run the acquisition stage on a remote machine. | Dev |
| R-12 | **Key-person / knowledge risk** (InSAR expertise concentrated in few people). | 2 | 2 | 4 | Domain doc (02), ADRs, parameter catalogue, code documentation, pair reviews. | PO |
| R-13 | **TS/Python boundary** drifts (schema mismatch, duplicated logic). | 2 | 2 | 4 | JSON Schema as single source + code generation (ADR-009); contract tests on both sides; no business logic duplicated across languages. | Dev |
| R-14 | **MATLAB licence or version problems** on the production machine (activation, host-ID change after hardware changes, StaMPS incompatibility with newer releases). | 2 | 2 | 4 | Installer validation (FR-INS-03); Dolphin default engine does not need MATLAB; Octave fallback later (FR-INS-13). | Admin |
| R-16 | **Scope increase** from a third engine (SBAS) delays v1.0. | 2 | 2 | 4 | MintPy reads ISCE2 topsStack natively; SBAS reuses the shared preprocessing and post-processing; spike S-7 before Phase 3. | PO |
| R-17 | **ISCE2 caps the shared environment at Python 3.11 / NumPy 1.26**; future Dolphin/MintPy releases may require newer versions. | 2 | 2 | 4 | Pin engine versions in the lockfile; if needed, move ISCE2 into its own pixi environment (the worker contract already isolates the stages). | Dev |
