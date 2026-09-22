# InSAR — Multi-Site Global Land Subsidence Monitoring

Satellite radar (InSAR) time-series analysis to detect and map surface deformation — primarily land subsidence — across five agriculturally and hydrologically significant sites on four continents, using Sentinel-1 SAR data processed through **ASF HyP3** and **MintPy**.

## Overview

Land subsidence, often driven by groundwater over-extraction, hydrocarbon withdrawal, or sediment compaction, is a slow-onset hazard that's hard to see on the ground but shows up clearly in satellite radar time series. This project builds a Small Baseline Subset (SBAS) InSAR pipeline and applies it to five sites, then overlays the resulting deformation maps on land-cover data to give the velocity signal geographic and agricultural context.

| # | Site | Region (context) | Land-cover reference layer |
|---|------|-------------------|------------------------------|
| 1 | **Arkansas, USA** | Mississippi Delta agricultural belt | USDA Cropland Data Layer (CDL), 2016 |
| 2 | **Italy** | Po Valley coastal plain, near the Adriatic | Corine Land Cover (CLC) 2018 |
| 3 | **Gujarat, India** | Kutch region arid agricultural belt | Regional land-cover base map |
| 4 | **Australia** | Inland South Australia / Murray–Darling agricultural belt | Regional land-cover base map |
| 5 | **Assam, India** | Brahmaputra River floodplain | Regional land-cover base map |

> Site descriptions above are inferred from the basemap imagery and file naming in `Results/` — worth a quick double-check/edit if any region is off.

## Methodology

1. **SAR acquisition & interferogram generation** — Sentinel-1 SLC pairs processed on-demand through **ASF HyP3**, producing per-pair unwrapped phase, spatial coherence, DEM, incidence-angle (look vector), and water-mask GeoTIFFs.
2. **Time-series inversion (MintPy `smallbaselineApp.py`)** — driven by `hyp3_config.txt` (HyP3 loader paths) and `smallbaselineApp.cfg` (full SBAS configuration):
   - Network built from temporal/perpendicular baseline thresholds and coherence-based edge selection
   - Reference acquisition date auto-selected per site (example run reference: `2024-01-03`)
   - Unwrapping-error correction and minimum-norm velocity inversion
   - Tropospheric delay correction (PyAPS + ERA5 reanalysis) and orbital ramp removal
3. **Quality control** —
   - `coherenceSpatialAvg.txt`: mean spatial coherence, temporal baseline, and perpendicular baseline per interferogram pair
   - `rms_timeseriesResidual_ramp.txt`: per-date RMS of the de-ramped residual time series, used to flag noisy acquisitions
4. **Deformation product** — mean line-of-sight (LOS) velocity exported to GeoTIFF (`subsidence_velocity.tif` / `<site>_subsidence.tif`)
5. **Contextualization** — velocity rasters overlaid on land-cover products in QGIS (USDA CDL for Arkansas, Corine Land Cover for Italy, etc.) to relate deformation hot spots to cropland, water bodies, and built-up areas

## Repository Structure

```
InSAR/
├── hyp3_config.txt                   # HyP3-specific MintPy loader paths
├── smallbaselineApp.cfg              # Full MintPy SBAS processing configuration
├── reference_date.txt                # Reference SAR acquisition date for the network
├── coherenceSpatialAvg.txt           # Per-interferogram mean coherence, Btemp, Bperp
├── rms_timeseriesResidual_ramp.txt   # Per-date RMS of de-ramped residual time series (QC)
├── subsidence_velocity.tif.aux.xml   # Raster stats — Arkansas velocity map
├── gujarat_subsidence.tif.aux.xml    # Raster stats — Gujarat velocity map
├── italy_subsidence.tif.aux.xml      # Raster stats — Italy velocity map
├── australia_subsidence.tif.aux.xml  # Raster stats — Australia velocity map
├── assam_subsidence.tif.aux.xml      # Raster stats — Assam velocity map
└── Results/
    ├── Site_1_Arkansas/     # Velocity map + USDA CDL overlay, screenshot
    ├── Site_2_Italy/        # Velocity map + Corine Land Cover overlay, screenshots
    ├── Site_3_Gujarat/      # Velocity map overlay, screenshot
    ├── Site_4_Australia/    # Velocity map overlay, screenshot
    └── Site_5_Assam/        # Velocity map overlay, screenshot
```

Raw Sentinel-1 products, intermediate MintPy HDF5 stacks (`ifgramStack.h5`, `timeseries*.h5`, etc.), and ERA5 tropospheric model files are intentionally excluded (see `.gitignore`) to keep the repository lightweight — only configs, QC metrics, final velocity rasters, and result visuals are version-controlled.

## Results — LOS Velocity Summary

Statistics pulled from each site's raster metadata (`*.tif.aux.xml`). Values are **line-of-sight (LOS)** velocities, not vertical-only, and follow MintPy's standard convention (negative = motion away from the satellite, i.e. typically subsidence).

| Site | Min (m/yr) | Max (m/yr) | Mean (m/yr) | Std Dev |
|------|-----------:|-----------:|------------:|--------:|
| Arkansas, USA | -1.44 | 0.72 | -0.16 | 0.22 |
| Italy | -2.16 | 2.71 | -0.09 | 0.26 |
| Gujarat, India | -1.68 | 0.80 | -0.14 | 0.20 |
| Australia | -0.37 | 0.43 | 0.03 | 0.07 |
| Assam, India | -1.10 | 0.54 | -0.02 | 0.23 |

Italy and Gujarat show the widest spread and the most negative means — consistent with locally concentrated subsidence hot spots — while Australia's range is comparatively tight. These are short-network, single-track estimates and should be read as indicative rather than validated ground-truth rates.

## Tools & Tech Stack

- **SAR data & processing:** Sentinel-1 (Copernicus), ASF DAAC / HyP3 On-Demand InSAR
- **Time-series InSAR:** MintPy (Miami INsar Time-series software in PYthon)
- **Atmospheric correction:** PyAPS + ERA5 reanalysis
- **GIS & visualization:** QGIS, GDAL
- **Land-cover reference data:** USDA Cropland Data Layer (CDL), Copernicus Corine Land Cover (CLC)

## Reproducing a Site Run

1. Request Sentinel-1 InSAR pairs for the AOI and date range via [ASF HyP3](https://hyp3-docs.asf.alaska.edu/) (or ASF Vertex) — download unwrapped phase, coherence, DEM, look-vector, and water-mask GeoTIFFs into `data/<site>/`.
2. Install MintPy: `conda install -c conda-forge mintpy`
3. Run the SBAS pipeline:
   ```bash
   smallbaselineApp.py smallbaselineApp.cfg --dir ./mintpy_<site>
   ```
   using the load paths defined in `hyp3_config.txt`.
4. Check QC outputs (`coherenceSpatialAvg.txt`, `rms_timeseriesResidual_ramp.txt`); drop noisy dates/pairs from the network config and re-run if needed.
5. Export the final velocity field to GeoTIFF:
   ```bash
   save_gdal.py velocity.h5
   ```
6. Overlay the velocity raster on a land-cover layer in QGIS and export a figure for the site.

## Future Work

- Quantitatively correlate deformation trends with land-cover class and, where available, groundwater extraction records
- Extend the SAR time series per site for higher-confidence long-term velocity estimates
- Consolidate the per-site workflow into a single automated batch pipeline
- Package results into an interactive dashboard (e.g. Streamlit + Leaflet) for cross-site comparison

## Author

**Kirtan Patel (Santoki)** — B.Tech CSE, CSPIT, CHARUSAT
GitHub: [@KirtanPatel2811](https://github.com/KirtanPatel2811)


