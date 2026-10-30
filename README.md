# Denman merged bed ensemble

This repository contains the regional and local bed-elevation results from Lösing et al. (2026), Gravity topography modeling of the Denman Glacier region using a geostatistical approach, published in The Cryosphere (https://doi.org/10.5194/tc-20-5629-2026). It also includes code to merge the two ensembles and produce a combined best-fit bed.

The gravity inversion to produce the results is based on the open-source code developed by Field et al. (2026, https://doi.org/10.5194/gmd-19-1749-2026), available on GitHub and archived on Zenodo (https://doi.org/10.5281/zenodo.14719548). 

Gravity and associated GNSS data collected during the Denman Terrestrial Campaign are available through the IMAS Data Portal (Lösing and Aitken, 2026, https://doi.org/10.25959/RM9S-KT42).

## Inputs

| Path | Contents |
| --- | --- |
| `data/Bedmachine/BedMachineAntarctica-v3.nc` | Reference bed, surface, thickness, masks, and ancillary fields |
| `results/regional/` | 103 regional inversion realizations |
| `results/local/` | 50 local inversion realizations |

Both result folders contain:

- `beds.npy`: bed realizations, shaped `(member, y, x)`.
- `std_beds.npy`: saved bed standard deviations.
- `rmse.npy`: saved fit diagnostics.
- `result_grav.npy`: predicted gravity.
- `target_filt.npy`: filtered gravity targets.

Additional notebook inputs are the reference datasets `ds_run_1k` and `ds_run_2k` (coordinates, bed, inversion mask).
Inversion coordinates and elevations are in metres. The projected CRS is EPSG:3031.

## Processing

1. Calculate radar-reference and gravity RMSE and select `best_index_regional` and `best_index_local`.
2. Cubically interpolate every regional bed onto the local grid.
3. Pair the 103 regional beds with the 50 local beds using reproducible shuffled assignments. Each local member is used two or three times; `local_indices` records the assignments.
4. Insert each assigned local bed into its regional bed within the local inversion mask, shrunk inward by 15 km to the ground-based gravity transect location.
5. Separately combine the independently selected best regional and local beds.
6. Linearly resample the ensemble and best composite onto the finer BedMachine grid.
7. Smooth the insertion seam using NaN-aware Gaussian smoothing (`sigma=3` cells). Smoothing weights equal one in a three-cell outer boundary band and taper to zero over 10 cells from the band, affecting both sides of the seam. Missing cells remain missing.

Regional and local beds are paired randomly, assuming any pair can be combined. The best composite combines the individually best-fitting regional and local beds and may not be part of the paired ensemble. Its gravity fit has not been checked again after merging and smoothing.

## Outputs

Key in-memory arrays:

| Object | Contents |
| --- | --- |
| `merged_beds` | Paired ensemble on the local grid |
| `merged_bed_best` | Separate best composite on the local grid |
| `merged_beds_hr` | Ensemble on the finer grid |
| `merged_bed_best_hr` | Best composite on the finer grid |
| `blended_beds` | Smoothed ensemble |
| `blended_bed_best` | Separately smoothed best composite |

The final export is **`Results/Merged/blended_beds.nc`**:

| Variable / coordinate | Meaning |
| --- | --- |
| `bed_elevation(ensemble_member, y, x)` | All 103 smoothed merged beds, in metres |
| `best_bed_elevation(y, x)` | Separate best-fit composite, in metres |
| `regional_index`, `local_index` | Source indices for each ensemble member |
| `x`, `y` | Projected coordinates in kilometres |
| `lon`, `lat` | Geographic coordinates in degrees |

The best composite stores `selected_regional_index` and `selected_local_index` as attributes.
