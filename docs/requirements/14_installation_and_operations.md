# 14 – Installation, Hardware & Operations

The product owner requires that **installing the application installs everything it needs**
(SNAPHU, ISCE2, Dolphin, StaMPS, Python and Node.js runtimes, …). MATLAB is the one exception because it
needs a licence. It is an **optional prerequisite** that the installer detects, validates, and can help install.

## 1. Recommended production hardware

Sizing basis ([01](01_vision_and_scope.md) §4): AOI ≤ 100 km² in Europe, ≤ 5 years, one user at a time,
usually one track per run, sometimes ascending + descending.

Worst case per track: a 100 km² AOI on burst/sub-swath boundaries → up to ~4 bursts per date. 5 years at a 6/12-day
revisit → ~200–300 dates. Raw burst download ≈ 300 × 4 × ~150 MB ≈ **~180 GB per track** `[VERIFY burst size S-1]`.
Cropped coregistered stack ≈ 2–3 Mpx × 300 dates × 8 B ≈ **~5–10 GB**. Peak working space during coregistration is
dominated by uncropped burst products (hence per-date cropping and cleanup, FR-PRE-02/08).

| Component | Minimum | Recommended | Reason |
|-----------|---------|-------------|--------|
| CPU | 8 cores / 16 threads, x86-64 with AVX2 | **16–24 cores**, high single-core clock | Coregistration and downloads parallelise over dates; StaMPS/MATLAB steps are largely single-threaded |
| RAM | 32 GB | **64 GB** (128 GB if two heavy jobs should overlap) | ISCE2 per-date processes ~2–4 GB each × parallel processes; Dolphin/StaMPS blocks |
| Working storage | 1 TB NVMe SSD | **2 TB NVMe SSD** | Coregistration I/O is heavy random access |
| Cache/archive storage | – (on working disk) | **4–8 TB** HDD/NAS or second SSD | Burst cache reused across projects (asc + desc × several projects) |
| GPU | none | Optional NVIDIA, ≥ 12 GB VRAM, CUDA | Dolphin can accelerate phase linking on GPU `[VERIFY S-2]`; not required |
| Network | 100 Mbit/s | **≥ 500 Mbit/s** | ~180 GB per track and 5 years |
| OS | Linux x86-64 with glibc ≥ 2.28 | **Ubuntu 24.04 LTS** (reference platform) | Installer is tested on it; Manjaro/Arch also supported for development |

Final values are confirmed with measurements in spike S-4.

### 1.2 Assessment of the actual production machine (Q-19)

| Component | Actual | vs. recommended | Consequence |
|-----------|--------|-----------------|-------------|
| CPU | Ryzen 7 3700X, 8 C / 16 T, AVX2 | = minimum | Fine. Coregistration runs ~6–8 dates in parallel; runtimes are longer than on 16+ cores (NFR-PERF-01 relaxed to ≤ 12 h `[TBC S-4]`). |
| RAM | 32 GB | = minimum (recommended 64 GB) | Enough for one job at a time with memory-bounded settings. The resource-derived parameters (07 §8) must keep the RAM budget at ~24 GB. **Upgrade to 64 GB recommended** (inexpensive DDR4 on AM4) if StaMPS runs on long stacks or two jobs should overlap. |
| Storage | 1 TB SSD free | = minimum (recommended 2 TB + 4–8 TB cache) | **The main constraint.** One track × 5 years ≈ 180 GB raw bursts; asc + desc ≈ 360 GB. The cache policy (FR-ACQ-09) evicts raw bursts after coregistration by default and keeps the small cropped stacks, so reprocessing stays cheap. An additional 2–4 TB HDD for the cache is recommended once several studies accumulate. |
| GPU | RTX 3080 Ti, 12 GB VRAM, CUDA | = recommended | Dolphin's GPU acceleration is enabled by default when detected `[VERIFY S-2]`. The installer must set up a matching CUDA runtime (via pixi) and needs the NVIDIA driver on the host (prerequisite). |

**Verdict:** suitable for MainSAR as specified (AOI ≤ 100 km², ≤ 5 years, one user). Recommended upgrades, in order
of benefit: (1) extra HDD/SSD for the cache, (2) 64 GB RAM.

## 2. Software prerequisites (must exist *before* installation)

| Prerequisite | Needed for | Notes |
|--------------|------------|-------|
| Linux x86-64, `bash`, `curl`, `tar`, `sha256sum`, ~15 GB free for the environment | Installer | Checked by the installer. No git, compilers, csh or root needed (spike S-6). |
| Internet access | Installer, data download | Proxy supported |
| NVIDIA driver (optional) | GPU acceleration | Only the host driver; the CUDA runtime comes with the pinned environment. The installer checks the driver version. |
| NASA Earthdata account | Data download | Entered during setup (FR-SET-01) |
| **MATLAB (optional)** + valid licence for the host | StaMPS engine | Individual/machine-bound licence → MATLAB installed natively on the host. Required toolboxes `[VERIFY S-3]`. **Without MATLAB, only the Dolphin engine is available** in the first implementation (Octave fallback deferred, Q-20). |
| Optional: CDS (ERA5) account, EU Login (EGMS) | Tropospheric correction, validation | Can be added later |

No root access is needed except for optional system packages (e.g. for PDF rendering) and systemd user lingering.

## 3. What the installer installs

All pinned by one lockfile (`pixi.lock`), installed in user space under the installation directory (MATLAB excluded):

| Component | Source | Notes |
|-----------|--------|-------|
| Node.js LTS runtime + app bundle | conda-forge / release artefact | Runs backend + serves UI |
| Python 3.12 + worker package | conda-forge + PyPI | `mainsar-worker` |
| ISCE2 (topsStack) | conda-forge (`isce2` 2.6.5 verified in S-1) | The installer must add `share/isce2/topsStack` and `site-packages/isce/applications` to `PATH` and `share/isce2` to `PYTHONPATH`; the package does not do it (S-1). |
| Dolphin | conda-forge / PyPI | |
| MintPy | conda-forge | SBAS engine |
| SNAPHU (2.x) | conda-forge `snaphu` (snaphu-py, bundles SNAPHU 2.0.7) | The installer links the bundled executable into its `bin/` (StaMPS calls `snaphu` by name); never relies on a system copy (S-6) |
| spurt (optional unwrapper) | PyPI | |
| asf_search, burst2safe, sentineleof, dem-stitcher | PyPI / conda-forge | |
| GDAL, PROJ, rasterio, GeoPandas, Shapely, xarray, PyAPS | conda-forge | |
| **StaMPS 4.1b** | Upstream commit `c159eb8` + **our patches** (in this GPL-3 repo, e.g. the byte-swap fix from S-3) | C++ tools compiled with pinned compilers and `-Werror=return-type`; MATLAB scripts registered on the MATLAB path. Also needs `csh`/`tcsh` and `gawk` (StaMPS scripts are csh). |
| **triangle** | conda-forge | Needed by StaMPS 3D unwrapping (note its non-commercial licence terms) |
| tcsh, gawk, patch, C++ compiler, make | conda-forge | StaMPS scripts and build; the installer links `tcsh` as `csh` and rewrites StaMPS shebangs (S-6) |
| GNU Octave + required packages | conda-forge | **Deferred** (Q-20): later fallback runtime for StaMPS; not part of the first implementation |
| Copernicus burst ID map | Download (checksum-pinned) | |

## 4. Installation requirements

| ID | Requirement | Prio | Acceptance criteria |
|----|-------------|:----:|---------------------|
| FR-INS-01 | **One-command installation** (`./install.sh` or `curl … \| bash` with checksum) that installs all components in §3 from a lockfile. | M | Fresh Ubuntu 24.04 VM → working installation without manual steps other than entering credentials. |
| FR-INS-02 | **Prerequisite check** before installing: OS/arch, glibc, CPU features (AVX2), RAM, free disk, internet access, write permissions. Shows a clear report and stops on blockers. | M | Each failing check produces a specific message. |
| FR-INS-03 | **MATLAB integration (optional)**: detect installed MATLAB versions (standard paths, `PATH`, user-supplied path), check that it starts headless (`matlab -batch`), check that the licence works and required toolboxes are present, and register it for the StaMPS engine. | M | Detection works for a MATLAB in `/usr/local/MATLAB`, `/opt/MATLAB` and a custom path; missing licence reported clearly. |
| FR-INS-04 | **MATLAB installation assistance (optional)**: offer to install MATLAB via MathWorks' `mpm` for users who hold a licence, followed by the MathWorks activation step (interactive, by the user). | S | Documented flow; the installer never handles licence files or credentials itself. |
| FR-INS-05 | Without MATLAB, the installer completes successfully, disables the StaMPS engine and states clearly which engines/features are available. | M | Capability summary shown at the end of installation and in the health page (FR-SET-05). |
| FR-INS-13 | Octave as an alternative StaMPS runtime (installed and configured by the installer). | C | Deferred (Q-20); acceptable fallback, not in the first implementation. |
| FR-INS-06 | **Post-install self-test**: runs a short end-to-end check on a bundled tiny dataset (DS-CI subset) through each available engine and reports pass/fail per component. | M | `mainsar doctor --full` passes on a fresh installation. |
| FR-INS-07 | **Service setup**: register the app as a systemd *user* service (auto-start optional), with start/stop/status commands. | S | `systemctl --user status mainsar` works. |
| FR-INS-08 | **Upgrade** in place: new lockfile/app version installed side by side, projects and cache migrated (DB schema migrations), rollback to the previous version possible. | S | Upgrade + rollback tested in CI on a VM. |
| FR-INS-09 | **Uninstall** removes the application and environment; data root (projects, cache) is kept unless explicitly requested. | M | |
| FR-INS-10 | **Offline/air-gapped install** from a prepared bundle. | W | Not needed (no requirement). |
| FR-INS-11 | Installation logs are written to a file and contain all component versions. | M | |
| FR-INS-12 | Developer setup: the same lockfile plus dev tools (`pixi run dev`) and a CI container image without MATLAB. | M | New developer runs tests within 30 min of cloning. |

## 5. Operations

| Topic | Requirement |
|-------|-------------|
| Start/stop | `mainsar start|stop|status` and the systemd user service. |
| Access | UI at `http://127.0.0.1:8080`; remote access through an SSH tunnel (ADR-008). |
| Configuration | `~/.config/mainsar/config.yaml` (data root, resource limits, MATLAB path); secrets in the OS keyring or a `0600` file. |
| Health | `mainsar doctor` checks tools, versions, credentials, disk; the same data on the UI health page. |
| Backup | Back up project directories (config, SQLite, results). The cache can be re-downloaded and needs no backup. |
| Disk management | Cache size limit + LRU eviction (FR-ACQ-08); "clean intermediates" per project (FR-PRJ-05). |
| Logs | Per-job logs in the project; application log with rotation. |
