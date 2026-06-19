# Detector & Geometry Diagnostic Tools

Programs for assessing detector geometry quality and spot prediction accuracy.

---

## cctbx.xfel.detector_residuals

**Source:** `xfel/command_line/detector_residuals.py`  
**Purpose:** Compute statistical measurements of consistency between observed and predicted reflection positions across the detector. The main tool for checking detector geometry quality after refinement.

### Input
```bash
cctbx.xfel.detector_residuals [options] experiment.expt reflections.refl
```
Typically run on combined/refined experiments from an ensemble refinement step.

### Parameters

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `dot_size` | 10 | int | Size of dots in detector panel plots |
| `panel_numbers` | True | bool | Show panel numbers on each panel plot |
| `verbose` | True | bool | Print statistics to stdout |
| `repredict_input_reflections` | True | bool | Use input models to repredict reflection positions before plotting |
| `residuals.plot_max` | 0.3 | float | Maximum residual (mm) shown in detector colormap |
| `residuals.histogram_max` | None | float | Maximum x value for histogram computation |
| `residuals.histogram_xmax` | None | float | Maximum x value shown in histogram |
| `residuals.histogram_ymax` | None | float | Maximum y value shown in histogram |
| `residuals.exclude_outliers_from_refinement` | True | bool | Exclude outlier-flagged reflections from statistics |
| `residuals.i_sigi_cutoff` | None | float | Minimum I/σI for RMSD plots |
| `residuals.recompute_outliers` | False | bool | Use sauter_poon to recompute and remove outliers |
| `residuals.delta_scalar` | 50 | float | Scale factor for deltaXY offset vectors in plots |
| `residuals.mcd_filter.enable` | False | bool | Apply 3-feature MCD (Minimum Covariance Determinant) filter on DeltaPsi plot |
| `residuals.mcd_filter.mahalanobis_distance` | 7 | float | MCD filter cutoff (σ units) |
| `residuals.mcd_filter.keep` | inliers | choice: inliers/outliers | Keep inliers or outliers of MCD filter |
| `residuals.print_correlations` | True | bool | Print per-panel-group radial/transverse vs delta_psi correlations |
| `repredict.enable` | False | bool | Repredict using bandpass/mosaicity approximations |
| `repredict.mode` | gaussian_mosaicity_and_bandpass | choice | Lineshape for reprediction |
| `repredict.refine_mode` | per_experiment | choice: per_experiment/all/None | Mosaicity refinement scope for reprediction |

### Outputs
- Per-panel residual plots (PDF)
- Histograms of position residuals
- DeltaPsi plots (spot offset in reciprocal space)
- Printed RMSD statistics

---

## cctbx.xfel.detector_congruence

**Source:** `xfel/command_line/cspad_detector_congruence2.py`  
**Purpose:** Assess statistical congruence across detector panels — check whether the panel-to-panel geometry is self-consistent. Useful for multi-panel detectors like CSPAD.

### Input
```bash
cctbx.xfel.detector_congruence experiment.expt reflections.refl
```

### Key parameters (phil scope similar to detector_residuals)

| Parameter | Default | Notes |
|---|---|---|
| `reflections_per_panel` | 10 | Minimum reflections per panel to include it in analysis |
| `residuals.plot_max` | 0.5 | Max residual (mm) in plots |
| `steps` | None | Which analysis steps to run |

---

## cctbx.xfel.detector_shifts

**Source:** `xfel/command_line/cspad_detector_shifts.py`  
**Purpose:** Compute the shifts between two detector geometries (e.g. before and after refinement). Shows per-panel translation and rotation.

### Input
```bash
cctbx.xfel.detector_shifts before.expt after.expt
```

---

## cctbx.xfel.experiment_residuals

**Source:** `xfel/command_line/experiment_residuals.py`  
**Purpose:** Visualize prediction offsets for a **single shot** experiment. Shows observed vs predicted positions as arrows overlaid on the detector, useful for debugging individual images.

### Input
```bash
cctbx.xfel.experiment_residuals refined.expt indexed.refl
```

### Parameters

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `lscale` | 25 | float | Scale the offset vectors by this amount |
| `lcolor` | `#777777` | str | Color of offset vector arrows |
| `scatt_cmap` | bwr | str | Colormap for scatter points |
| `clim` | None | floats(size=2) | Colormap limits, e.g. `[-0.01, 0.01]` |
| `mark_scale` | 15 | int | Marker size for scatter plot |
| `exper_id` | 0 | int | Experiment index if file contains multiple shots |
| `title_tag` | None | str | Additional tag for plot title |
| `plot_rad_trans` | False | bool | Also show histogram of radial and transverse offsets |
| `output.file` | None | str | Optional HDF5 output file for saving results |
| `output.overwrite` | False | bool | Overwrite existing output file |
| `output.only` | False | bool | Write output only, don't display plots |

---

## cctbx.xfel.jungfrau_metrics

**Source:** `xfel/command_line/jungfrau_metrics.py`  
**Purpose:** Analyze the agreement between observed and predicted Bragg spots specifically on the **Jungfrau 16M** detector (8 panels × 32 sensors). Assesses metrology quality and spot prediction accuracy using RANSAC fitting.

### Input
```bash
cctbx.xfel.jungfrau_metrics input.refl_glob="run_*.refl" input.expt_glob="run_*.expt"
```

### Parameters

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `input.refl_glob` | None | str | Glob pattern for reflection files |
| `input.expt_glob` | None | str | Glob pattern for experiment files |
| `input.predictions_glob` | None | str | Separate glob for predictions (alternative to refl_glob) |
| `input.observations_glob` | None | str | Separate glob for observations (alternative to refl_glob) |
| `panel_numbers` | True | bool | Show panel numbers on plots |
| `verbose` | False | bool | Print extra statistics |
| `dispatch.by_run` | True | bool | Print per-run metric analysis |
| `dispatch.by_panel_mcd_filter` | True | bool | Repeat sensor analysis with panel-level MCD filter |
| `residuals.mcd_filter.enable` | False | bool | Apply MCD filter |

### Output
- Per-run RMSD analysis
- Per-sensor RANSAC-fit translation vectors
- Overall detector drift assessment
