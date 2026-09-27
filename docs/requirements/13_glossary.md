# 13 – Glossary

| Term | Definition |
|------|------------|
| **AOI** | Area of interest: the region to analyse. |
| **APS** | Atmospheric phase screen: the atmospheric contribution to interferometric phase. |
| **Amplitude dispersion (D_A)** | σ_A / μ_A of a pixel's amplitude over time; a proxy for phase stability used to pick PS candidates. |
| **ASF DAAC** | Alaska Satellite Facility Distributed Active Archive Center; hosts the Sentinel-1 archive and burst service. |
| **Ascending / descending** | Satellite flight direction (south→north / north→south); gives different look directions. |
| **Baseline (perpendicular, B⊥)** | Distance between orbit positions perpendicular to the look direction; drives topographic phase sensitivity and decorrelation. |
| **Worker** | A stateless Python command-line program that runs one processing stage for the TypeScript core. |
| **Burst** | Azimuth segment of a TOPS sub-swath (~20 km); the smallest independently processable unit of an IW SLC. |
| **Burst ID** | Stable, global identifier of a burst footprint (ESA, since IPF 3.40). |
| **CDSE** | Copernicus Data Space Ecosystem: ESA's data access platform (replaced SciHub in 2023). |
| **CLAP** | Combined Low-pass and Adaptive Phase filter used by StaMPS. |
| **COG** | Cloud Optimized GeoTIFF (proper name, original spelling): tiled GeoTIFF with overviews, efficient for partial reads. |
| **Coherence** | Measure (0–1) of phase similarity; *temporal coherence* measures how well a point's phase history fits the estimated model. |
| **Coregistration** | Resampling all SLCs onto the grid of a reference acquisition, with sub-pixel accuracy. |
| **DEM** | Digital elevation model (here Copernicus GLO-30). |
| **DEM error** | Difference between the true scatterer height and the DEM height; causes baseline-dependent phase. |
| **Dolphin** | Open-source PS/DS phase-linking and time-series software from JPL (OPERA project). |
| **DS** | Distributed scatterer: a group of pixels with similar statistics and moderate coherence (fields, bare soil). |
| **EGMS** | European Ground Motion Service (Copernicus): Europe-wide Sentinel-1 PS/DS products. |
| **ESD** | Enhanced Spectral Diversity: burst-overlap-based azimuth coregistration refinement for TOPS data. |
| **Engine** | A pluggable implementation of the multi-temporal analysis stages (e.g. Dolphin, StaMPS). |
| **ERA5** | ECMWF global atmospheric reanalysis; used to model tropospheric delay. |
| **GACOS** | Generic Atmospheric Correction Online Service for InSAR. |
| **GNSS** | Global Navigation Satellite Systems (GPS etc.); independent point displacement measurements. |
| **Interferogram** | Pixel-wise phase difference of two coregistered SLCs. |
| **ISCE2 / topsStack** | InSAR Scientific Computing Environment v2; topsStack = its Sentinel-1 stack processor. |
| **IW** | Interferometric Wide swath mode of Sentinel-1 (TOPS acquisition). |
| **LOS** | Line of sight: the satellite–ground direction; InSAR measures displacement projected onto it. |
| **Ministack** | Subset of dates processed together in Dolphin's sequential phase-linking estimator. |
| **MintPy** | Python toolbox for InSAR time-series analysis (mainly SBAS). |
| **MT-InSAR** | Multi-temporal InSAR: PS, SBAS, DS methods using image stacks. |
| **Phase linking** | Estimating a consistent phase series from all interferometric combinations of a pixel neighbourhood (EMI/EVD). |
| **Phase unwrapping** | Recovering absolute phase from phase known modulo 2π. |
| **POEORB / RESORB** | Sentinel-1 precise (~20 days latency) / restituted (~3 h) orbit files. |
| **pixi** | Package/environment manager (conda-forge + PyPI) with lockfiles; used by the installer. |
| **PS** | Persistent scatterer: a pixel dominated by one stable scatterer over time. |
| **Reference point** | Point/area assumed stable; all displacements are relative to it. |
| **Relative orbit (track)** | Sentinel-1 orbit number within the repeat cycle (1–175); fixes the viewing geometry. |
| **SAFE** | Standard Archive Format for Europe: Sentinel product packaging. |
| **SBAS** | Small Baseline Subset: time-series method using short-baseline interferograms. |
| **SCLA** | Spatially correlated look-angle error (StaMPS term; DEM error + orbit residuals). |
| **SHP** | Statistically homogeneous pixels: neighbours sharing the same amplitude distribution (for DS). |
| **SLC** | Single Look Complex: SAR image with amplitude and phase at full resolution. |
| **SSE** | Server-Sent Events: one-way push from server to browser (progress, logs). |
| **SNAPHU** | Statistical-cost Network-flow Algorithm for Phase Unwrapping. |
| **StaMPS** | Stanford Method for Persistent Scatterers: MATLAB-based PS/SBAS software (Hooper et al.). |
| **TOPS** | Terrain Observation with Progressive Scans: Sentinel-1 IW acquisition mode. |
| **Temporal coherence** | Quality measure of a point's phase time series relative to the fitted model/phase-linked solution. |
