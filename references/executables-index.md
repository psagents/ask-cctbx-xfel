# cctbx.xfel.* Executables — Master Index

All executables installed at:
`/sdf/group/lcls/ds/tools/cctbx/psana2/build/bin/cctbx.xfel.*`

Source code at:
`/sdf/group/lcls/ds/tools/cctbx/psana2/modules/cctbx_project/xfel/`

---

## Executables by category

### Core SFX Processing

| Executable | Source file | Description |
|---|---|---|
| `cctbx.xfel.process` | `xfel/command_line/xfel_process.py` | Process pre-formatted stills (CBF, HDF5) with DIALS + database logging. Inherits full `dials.stills_process` phil. Modern choice for non-XTC data. |
| `cctbx.xfel.xtc_process` | `xfel/command_line/xtc_process.py` | Process raw LCLS XTC stream data. Reads via psana, converts to CBF on-the-fly, then runs DIALS. Legacy but still used for CSPAD/CBF workflows. |
| `cctbx.xfel.merge` | `xfel/merging/command_line/merge.py` | Merge indexed/integrated stills to MTZ. MPI-parallel. The central merging engine. Full phil in `xfel/merging/application/phil/phil.py`. |
| `cctbx.xfel.mpi_integrate` | `xfel/merging/command_line/mpi_integrate.py` | Re-integrate indexed experiments in parallel with updated geometry. Skips spotfinding/indexing. |

### Job Submission and Workflow

| Executable | Source file | Description |
|---|---|---|
| `cctbx.xfel.submit_job` | `xfel/command_line/submit_job.py` | Submit a cctbx.xfel job to a cluster (SLURM, LSF, PBS, SGE). Creates trial directory structure, copies phil files, submits. |
| `cctbx.xfel.time_varying_refinement` | `xfel/command_line/time_varying_refinement.py` | Ensemble detector refinement: divides results into time-ordered chunks and runs `dials.combine_experiments` + `dials.refine` per chunk. Corrects time-varying detector drift. |
| `cctbx.xfel.xtc_dump` | `xfel/command_line/xtc_dump.py` | Dump raw XTC events to CBF or pickle without DIALS processing. |
| `cctbx.xfel.mpi_submit` | (deprecated) | **Renamed to `cctbx.xfel.submit_job`.** Raises Sorry if called. |
| `cctbx.xfel.stripe_experiment` | (deprecated) | **Renamed to `cctbx.xfel.time_varying_refinement`.** Raises Sorry if called. |

### Detector Geometry Diagnostics

| Executable | Source file | Description |
|---|---|---|
| `cctbx.xfel.detector_residuals` | `xfel/command_line/detector_residuals.py` | Statistical analysis of obs–predicted residuals across all detector panels. Main geometry quality tool. Produces plots. |
| `cctbx.xfel.detector_congruence` | `xfel/command_line/cspad_detector_congruence2.py` | Check panel-to-panel self-consistency of a multi-panel detector. |
| `cctbx.xfel.detector_shifts` | `xfel/command_line/cspad_detector_shifts.py` | Show per-panel translation/rotation shifts between two detector geometries. |
| `cctbx.xfel.experiment_residuals` | `xfel/command_line/experiment_residuals.py` | Visualize prediction offsets for a single shot as arrows overlaid on detector image. |
| `cctbx.xfel.jungfrau_metrics` | `xfel/command_line/jungfrau_metrics.py` | Jungfrau 16M specific: per-run and per-sensor RANSAC metrics for spot prediction quality. |

### Analysis and Post-Processing

| Executable | Source file | Description |
|---|---|---|
| `cctbx.xfel.weather` | `xfel/command_line/weather.py` | Plot MPI processing performance (wall time vs rank) from debug files. |
| `cctbx.xfel.drift` | `xfel/command_line/drift.py` | Visualize detector origin and unit cell drift over time across run/chunk batches. |
| `cctbx.xfel.filter_experiments_by_rmsd` | `xfel/command_line/filter_experiments_by_rmsd.py` | Filter multi-experiment set by per-image RMSD using Tukey's rule. Standard quality gate. |
| `cctbx.xfel.recompute_mosaicity` | `xfel/command_line/recompute_mosaicity.py` | Recompute Nave mosaic parameters after ensemble refinement. |
| `cctbx.xfel.trumpet_plots` | `xfel/command_line/trumpet_plots.py` | Generate trumpet plots (partiality vs resolution) per experiment. See Sauter 2014. |
| `cctbx.xfel.show_spot_separation` | `xfel/command_line/show_spot_separation.py` | Statistics on angular separation between strong spots (hit quality diagnostic). |
| `cctbx.xfel.frame_extractor` | `xfel/command_line/frame_extractor.py` | Convert DIALS `.expt`+`.refl` to legacy cctbx.xfel `.pickle` format. |
| `cctbx.xfel.h5_average` | `xfel/command_line/h5_average.py` | Average/max-project/stddev a multi-image HDF5 file. |
| `cctbx.xfel.upload_mtz` | `xfel/command_line/upload_mtz.py` | Upload merged MTZ and log to Google Drive. |

### Plot Utilities (output-only)

| Executable | Source file | Description |
|---|---|---|
| `cctbx.xfel.plot_reflection_stats` | `xfel/command_line/` | Plot reflection statistics from processing output |
| `cctbx.xfel.plot_run_stats_from_experiments` | `xfel/command_line/` | Plot per-run statistics from experiment files |
| `cctbx.xfel.plot_run_stats_from_stats_pickle` | `xfel/command_line/` | Plot run stats from a stats pickle |
| `cctbx.xfel.plot_spotfinder_stats_from_stats_pickle` | `xfel/command_line/` | Plot spotfinder stats from a stats pickle |
| `cctbx.xfel.plot_uc_cloud_from_experiments` | `xfel/command_line/` | Plot unit cell parameter cloud from experiment files |
| `cctbx.xfel.powder_from_spots` | `xfel/command_line/` | Generate a powder pattern from strong spots |

### Specialist / Less Common

| Executable | Source file | Description |
|---|---|---|
| `cctbx.xfel.small_cell_process` | `xfel/command_line/` | Process small-unit-cell crystals with specialized indexing |
| `cctbx.xfel.small_cell.cake_plot` | `xfel/command_line/` | Cake (azimuthal) plot for small cell analysis |
| `cctbx.xfel.candidate_cells` | `xfel/command_line/` | Generate candidate unit cells from powder rings |
| `cctbx.xfel.preference` | `xfel/command_line/preference.py` | Preference model utility |
| `cctbx.xfel.ui_server` | `xfel/command_line/` | cctbx.xfel GUI server |
| `cctbx.xfel` | (entry point) | Base entry point |

---

## Renamed/deprecated executables

| Old name | Current name |
|---|---|
| `cctbx.xfel.mpi_submit` | `cctbx.xfel.submit_job` |
| `cctbx.xfel.stripe_experiment` | `cctbx.xfel.time_varying_refinement` |

---

## Reference navigation

| Topic | Reference file |
|---|---|
| `cctbx.xfel.merge` full parameter reference | [merge.md](merge.md) |
| `cctbx.xfel.process` + `cctbx.xfel.xtc_process` LCLS-specific parameters | [processing.md](processing.md) |
| Detector geometry diagnostics (detector_residuals, congruence, shifts, jungfrau) | [detector-tools.md](detector-tools.md) |
| Job submission + workflow (submit_job, time_varying_refinement, xtc_dump) | [job-submission.md](job-submission.md) |
| Analysis tools (weather, drift, filter_experiments_by_rmsd, etc.) | [analysis-tools.md](analysis-tools.md) |
| LCLS SFX good defaults and recipes | [lcls-sfx-recipes.md](lcls-sfx-recipes.md) |
