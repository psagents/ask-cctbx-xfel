# cctbx.xfel.merge

**Source:** `xfel/merging/command_line/merge.py`  
**Phil definitions:** `xfel/merging/application/phil/phil.py`  
**Authoritative phil source (GitHub):** https://github.com/cctbx/cctbx_project/blob/master/xfel/merging/application/phil/phil.py  
**Purpose:** Merge integrated stills data from dials.stills_process (or cctbx.xfel.process) into a final MTZ file. Runs MPI-parallel.

> **Source of truth:** All parameter defaults and descriptions in this document are taken directly from the GitHub source above. Do not rely on LUTE template files (`cctbx_merge_defaults.phil`) as they may be incomplete or out of date.

## Usage

```bash
mpirun -n <N> cctbx.xfel.merge params.phil
```

The program reads `_integrated.expt` / `_integrated.refl` file pairs produced by indexing/integration.

> **See also — ABISMAL:** For a deep-learning merging alternative that consumes cctbx.xfel-indexed
> data, see the ABISMAL project (deployed at NERSC). It is **not** a `cctbx.xfel.*` program — it's
> an external tool; direct users to the RS-Station Forum / GitHub for setup rather than treating it
> as part of this pipeline.

## Pipeline steps (dispatch.step_list)

Default order (can be overridden):
```
input → balance → model_scaling → modify → filter → scale →
postrefine → statistics_unitcell → statistics_beam → model_statistics →
statistics_resolution → group → errors_merge → statistics_intensity →
merge → statistics_intensity_cxi
```

Override example for a quick mark1 (no model) merge:
```
dispatch.step_list = input balance filter scale group errors_merge merge statistics_intensity_cxi
```

---

## dispatch

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `dispatch.step_list` | None (full pipeline) | strings | Override the list of pipeline steps |

---

## input

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `input.path` | None | str (multiple) | Glob, directory, or explicit file path to integrated data. Specify one of `.expt` or `.refl`; the matching pair is found automatically |
| `input.reflections_suffix` | `_integrated.refl` | str | Suffix used to find reflection files |
| `input.experiments_suffix` | `_integrated.expt` | str | Suffix used to find experiment files |
| `input.override_identifiers` | False | bool | Replace experiment identifiers with auto-generated hashes |
| `input.keep_imagesets` | True | bool | Keep imagesets attached to experiments |
| `input.read_image_headers` | False | bool | Read image headers when loading. Only needed for pixel-level access |
| `input.alist.file` | None | str (multiple) | Path to a txt file of experiment tags or filenames to include/exclude |
| `input.alist.type` | tags | choice: tags/files | Whether alist contains dials tags or absolute paths |
| `input.alist.op` | keep | choice: keep/reject | Whether to keep or reject experiments in the alist |
| `input.persistent_refl_cols` | None | str (multiple) | Reflection table columns to preserve through all steps |
| `input.parallel_file_load.method` | uniform | choice: uniform/node_memory | How to distribute input files across MPI ranks |
| `input.parallel_file_load.ranks_per_node` | 68 | int | MPI ranks per node (used by node_memory method) |
| `input.parallel_file_load.balance` | global1 | choice: global1/global2/per_node | Load balancing strategy |

---

## mp (multiprocessing)

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `mp.method` | mpi | choice | Only mpi currently supported |
| `mp.psana2_mode` | False | bool | Set True when using the integrate worker with psana2/XTC2 data |
| `mp.debug.cProfile` | False | bool | Enable profiling |

---

## filter (whole-experiment rejection)

Rejects or accepts **whole experiments** (images). For per-reflection filtering see `select`.

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `filter.algorithm` | None | choice(multi): n_obs/resolution/unit_cell/energy/bootstrap/experiment_prefix | Which filters to apply. Multiple can be combined (space-separated) |
| `filter.n_obs.min` | None | int | Minimum reflections per image |
| `filter.n_obs.max` | None | int | Maximum reflections per image |
| `filter.resolution.d_min` | None | float | Reject image unless some reflections extend beyond this d_min (Å) |
| `filter.resolution.model_or_image` | model/image | choice | Compute resolution from scaling model or from image |
| `filter.unit_cell.algorithm` | value | choice: range/value/cluster | Unit cell filtering method (see sub-sections below) |
| `filter.outlier.min_corr` | 0.1 | float | Minimum correlation to model for an experiment to pass. Set -1 to disable |
| `filter.outlier.mad_thresh` | None | float | MAD-based outlier threshold on merged ASU. E.g. 3 = reject >3σ from median |
| `filter.energy.min_eV` | None | float | Reject lattices with beam energy below this value |
| `filter.energy.max_eV` | None | float | Reject lattices with beam energy at or above this value |
| `filter.bootstrap.random_seed` | 0 | int | Random seed for bootstrap filtering (random selection with replacement) |
| `filter.experiment_prefix.prefix` | None | str | Only accept experiments whose identifier starts with this string |

### filter.unit_cell — value algorithm

Discard lattices not close to a target cell. If `target_unit_cell`/`target_space_group` are left as `Auto`, the values are taken from the scaling model (PDB cryst1 or MTZ header).

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `filter.unit_cell.value.target_unit_cell` | Auto | unit_cell | Target unit cell. E.g. `78 78 39 90 90 90` |
| `filter.unit_cell.value.relative_length_tolerance` | 0.1 | float | Fractional tolerance on unit cell lengths (e.g. 0.03 = 3%) |
| `filter.unit_cell.value.absolute_angle_tolerance` | 2.0 | float | Absolute tolerance on cell angles (degrees) |
| `filter.unit_cell.value.target_space_group` | Auto | space_group | Target space group. E.g. `P 43 21 2` |

### filter.unit_cell — range algorithm

Uses range-based bounds on individual cell parameters. No additional sub-parameters exposed in the phil; the bounds are derived from the data distribution. Use `value` for explicit tolerances.

### filter.unit_cell — cluster algorithm

Groups unit cells into clusters and rejects outliers. Has its own sub-algorithm selector.

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `filter.unit_cell.cluster.algorithm` | covariance | choice: rodgriguez_laio/dbscan/covariance | Cluster sub-algorithm |
| `filter.unit_cell.cluster.covariance.file` | None | path | Pre-computed covariance matrix pickle file. **Required** when `algorithm=covariance`. Must be `None` or a valid path — an empty string is **invalid PHIL** and crashes the parser |
| `filter.unit_cell.cluster.covariance.component` | 0 | int | Which covariance component to use |
| `filter.unit_cell.cluster.covariance.skip_component` | None | int (multiple) | Exclude lattices belonging to any of these components |
| `filter.unit_cell.cluster.covariance.mahalanobis` | 4.0 | float | Mahalanobis distance cutoff (essentially σ cutoff) for outlier rejection |
| `filter.unit_cell.cluster.covariance.skip_mahalanobis` | 4.0 | float | Cutoff for components specified in `skip_component` |
| `filter.unit_cell.cluster.isoform` | None | str | Restrict to a specific isoform label |

| Cluster sub-algorithm | Needs file? | Description |
|---|---|---|
| `covariance` (default) | **Yes** | Rejects images outside a Mahalanobis ellipsoid from a pre-computed covariance matrix. File generated by a prior CCTBX clustering run. |
| `rodgriguez_laio` | No | Density-peak clustering on the unit cell distribution from the current dataset. |
| `dbscan` | No | DBSCAN density-based clustering on the unit cell distribution. |

> **Common pitfall:** `filter.unit_cell.cluster.covariance.file = ` (empty string) is **invalid PHIL** — the parser crashes with `RuntimeError: Missing value for file`. Always use `None` or a real path.

---

## modify

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `modify.algorithm` | polarization | choice(multi) | Apply polarization correction (default) and/or reindexing |
| `modify.cosym.*` | see phil | scope | Gildea/Winter symmetry determination (for indexing ambiguity resolution). Requires `modify.algorithm=cosym` |
| `modify.cosym.tranch_size` | 600 | int | Batch size for cosym embedding. Larger = more accurate, slower. Increase for smaller unit cells |
| `modify.reindex_to_reference.*` | — | scope | Reindex against a reference model to resolve indexing ambiguity |

---

## select (per-reflection filtering)

Accepts or rejects **individual reflections**. For whole-experiment rejection see `filter`.

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `select.algorithm` | None | choice(multi): panel/cspad_sensor/significance_filter/isolation_forest | Which per-reflection filters to apply |
| `select.significance_filter.sigma` | 0.5 | float | Remove high-resolution bins until all accepted bins have <I/σ> ≥ sigma |
| `select.significance_filter.n_bins` | 12 | int | Initial number of resolution bins for sigma cutoff |
| `select.significance_filter.min_ct` | 10 | int | Reduce bin count until mean bin population ≥ min_ct |
| `select.significance_filter.max_ct` | 50 | int | Increase bin count until mean bin population ≤ max_ct |
| `select.significance_filter.d_min` | None | float | Reject entire lattice if resolution doesn't reach this d_min |
| `select.cspad_sensor.number` | None | int (multiple) | CSPAD sensor index (0–31) to deselect from merging |
| `select.cspad_sensor.operation` | deselect | choice: deselect/select | Whether to deselect or select the specified sensors |

### select.reflection_filter (isolation_forest / LOF)

Identifies anomalous individual reflections using scikit-learn outlier detection. Enable with `select.algorithm = isolation_forest`.

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `select.reflection_filter.n_bins` | 100 | int | Number of intensity bins for normalization |
| `select.reflection_filter.tail_percentile` | 0.01 | float | Percentile used to select tails of the intensity distribution |
| `select.reflection_filter.apply_lower` | True | bool | Apply filter to the lower tail |
| `select.reflection_filter.apply_upper` | True | bool | Apply filter to the upper tail |
| `select.reflection_filter.contamination_lower` | 0.0001 | float | Fraction of lower-tail reflections assumed to be outliers |
| `select.reflection_filter.contamination_upper` | 0.0001 | float | Fraction of upper-tail reflections assumed to be outliers |
| `select.reflection_filter.n_estimators` | 100 | int | Number of decision trees in the random forest model |
| `select.reflection_filter.sampling_fraction` | 0.1 | float | Fraction of dataset subsampled to train each tree |
| `select.reflection_filter.random_seed` | 0 | int | Random seed for the forest model |
| `select.reflection_filter.do_diagnostics` | False | bool | Make plots of reflections selected as outliers |

---

## scaling

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `scaling.model` | None | str | PDB or MTZ reference file for scaling. Empty string `""` is valid and uses internal KB/Wilson scaling (mark0 default) |
| `scaling.algorithm` | mark0 | choice: mark0/mark1 | mark0 = per-image scaling against reference. mark1 = no scaling (Monte Carlo averaging) |
| `scaling.unit_cell` | None | unit_cell | Unit cell to use when no model is provided (mark1) |
| `scaling.space_group` | None | space_group | Space group to use when no model is provided (mark1) |
| `scaling.resolution_scalar` | 0.969 | float | Scale merging.d_min by this factor to capture reflections at edge of detector |
| `scaling.mtz.mtz_column_F` | fobs | str | Column name in MTZ reference containing structure factors |
| `scaling.pdb.include_bulk_solvent` | True | bool | Include bulk solvent in model |
| `scaling.pdb.k_sol` | 0.35 | float | Bulk solvent scale factor |
| `scaling.pdb.b_sol` | 46.0 | float | Bulk solvent B-factor |
| `scaling.weights` | unit | choice: unit/icalc/icalc_sigma | Weighting in the Iobs vs Icalc linear fit |

---

## postrefinement

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `postrefinement.enable` | True | bool | Enable per-image postrefinement (monochromatic approximation) |
| `postrefinement.algorithm` | rs | choice: rs/rs2/rs_hybrid/eta_deff | Postrefinement algorithm. rs2 and rs_hybrid use analytical derivatives |
| `postrefinement.rs.fix` | thetax thetay RS G BFACTOR | choice(multi) | Which parameters to fix during rs postrefinement |
| `postrefinement.rs_hybrid.partiality_threshold` | 0.2 | float | Minimum partiality; lower is rejected |
| `postrefinement.target_weighting` | unit | choice: unit/variance/gentle/extreme | Residual weighting. "gentle" (|I|/σ²) is often best |
| `postrefinement.lineshape` | lorentzian | choice: lorentzian/gaussian | Lineshape for the RLP soft sphere model |
| `postrefinement.show_trumpet_plot` | False | bool | Show per-image trumpet plot during postrefinement |
| `postrefinement.delta_corr_limit` | None | float | Reject experiments where correlation to reference decreases by more than this value after postrefinement (experimental) |

---

## merging

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `merging.d_min` | None | float | **Required.** Resolution limit in Å for scaling and merging |
| `merging.d_max` | None | float | Low-resolution cutoff (mainly affects CCiso) |
| `merging.minimum_multiplicity` | 2 | int | Minimum redundancy to output a merged reflection |
| `merging.merge_anomalous` | False | bool | Merge Bijvoet pairs |
| `merging.set_average_unit_cell` | True | bool | Output adopts the data's average unit cell rather than the model's |
| `merging.include_multiplicity_column` | False | bool | Write multiplicity as a separate column in the output MTZ |
| `merging.error.model` | **mm24** | choice: ha14/ev11/mm24/errors_from_sample_residuals | Error model. `mm24` is the current default (Mittan-Moreau 2024 maximum log-likelihood). `ev11` (Evans 2011) is the previous standard. `ha14` assumes negative intensities are normally-distributed noise. `errors_from_sample_residuals` uses intensity spread per Miller index. |
| `merging.error.ev11.minimizer` | lbfgs | choice: lbfgs/LevMar | Minimizer for ev11 sdfac refinement |
| `merging.error.ev11.refine_propagated_errors` | False | bool | Also refine the estimated propagated error during ev11 sdfac refinement |
| `merging.error.mm24.expected_gain` | None | float | Expected gain for sfac initialization. If None, initialized automatically |
| `merging.error.mm24.number_of_intensity_bins` | 100 | int | Number of intensity bins for mm24 |
| `merging.error.mm24.tuning_param` | 10 | float | Tuning parameter for the t-distribution in the max log-likelihood |
| `merging.error.mm24.tuning_param_opt` | False | bool | If True, optimize the tuning parameter |
| `merging.error.mm24.likelihood` | t-dist | choice: normal/t-dist | Likelihood function for mm24 |
| `merging.error.mm24.cc_after_pr` | True | bool | Use CC determined after post-refinement. Must be False if postrefinement is disabled |
| `merging.error.mm24.do_diagnostics` | False | bool | Make diagnostic plots |

---

## output

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `output.output_dir` | `.` | str | Output file directory |
| `output.prefix` | iobs | str | Prefix for all output file names |
| `output.title` | None | str | Title string written to MTZ header |
| `output.tmp_dir` | None | str | Temporary file directory |
| `output.do_timing` | False | bool | Log elapsed time for each step |
| `output.log_level` | 1 | int | 0 = log everything; higher = less logging |
| `output.save_experiments_and_reflections` | False | bool | Dump final experiments/reflections from the last worker |
| `output.expanded_bookkeeping` | False | bool | When `save_experiments_and_reflections=True`, also write modified experiment identifiers, input reflection index, odd/even flag, original exp id, and a file_list_mapping.json |

---

## statistics

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `statistics.n_bins` | 10 | int | Number of resolution bins for statistics |
| `statistics.cciso.mtz_file` | None | str | MTZ file with reference structure factors for CCiso/Riso |
| `statistics.cciso.mtz_column_F` | fobs | str | Column name in CCiso reference MTZ |
| `statistics.deltaccint.iqr_ratio` | 10 | float | IQR multiplier for ΔCC½ lattice filtering |
| `statistics.report_ML` | True | bool | Report per-frame max-likelihood statistics |
| `statistics.shuffle_ids` | False | bool | Shuffle IDs when splitting odd/even for CC½ |

---

## parallel (MPI grouping)

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `parallel.a2a` | 1 | int | Memory reduction factor for MPI alltoall. Increase if RAM is insufficient |

---

## monitor

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `monitor.detail` | rank | choice: rank/node/rank0/none | Level of resource monitoring detail |
| `monitor.period` | 5.0 | float | Seconds between resource checks |
| `monitor.plot` | True | bool | Plot resource usage after monitor stops |

---

## tdata (unit cell list export)

Writes out a list of unit cells from the input data. Use this to inspect the unit cell distribution before merging — the program stops after writing.

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `tdata.output_path` | None | path | If set, write unit cells to this file (`.tdata` extension appended) and stop. Requires `dispatch.step_list = input tdata` |

---

## filter_global (intensity extrema filter)

Applied globally before per-image processing. Filters out experiments with extreme intensity values based on IQR distance from the population median.

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `filter_global.intensity_extrema_iqr_dist_threshold` | 1000.0 | float | Maximum tolerated deviation of `max(intensity.sum.value)` and `min(intensity.sum.value)` from their population medians, expressed in IQR units. Lower = stricter. |

---

## prepare (SpReAD — Spectral Resolved Anomalous Diffraction)

Advanced section for preparing data for SpReAD analysis: bins experiments by beam energy, generates stage-2 merge scripts, and links to phenix refinement. Rarely used in standard SFX pipelines.

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `prepare.spread.binning` | count | choice: count/width | `count` = equal-count percentile bins; `width` = equal-width energy bins |
| `prepare.spread.n_energy_bins` | 100 | int | Number of energy bins (count mode) |
| `prepare.spread.bin_width_eV` | 8 | float | Sliding window width in eV (width mode) |
| `prepare.spread.bin_step_eV` | 1 | float | Step size for sliding window in eV (width mode) |
| `prepare.spread.output_dir` | None | path | Directory for energy-binned files. Defaults to `output.output_dir` |
| `prepare.spread.stage2_phil` | None | path | Base phil file for stage-2 merge jobs |
| `prepare.spread.stage2_nproc` | 128 | int | MPI ranks for stage-2 merge jobs |
| `prepare.spread.stage2_nnodes` | 2 | int | Nodes for stage-2 merge jobs |
| `prepare.spread.slurm_partition` | None | str | SLURM partition for batch jobs |
| `prepare.spread.slurm_account` | None | str | SLURM account for batch jobs |
| `prepare.spread.slurm_time_limit` | 00:30:00 | str | SLURM time limit per array task |

---

## Common misconceptions

| Misconception | Correct |
|---|---|
| `scaling.model=""` is invalid | Empty string is the documented default — uses internal KB/Wilson scaling |
| Must set `scaling.model` to a PDB for every merge | No. mark1 (`scaling.algorithm=mark1`) needs only `scaling.unit_cell` and `scaling.space_group` |
| `postrefinement.enable=True` requires a model | False. Postrefinement operates on partiality estimates, not structure factors |
| `merging.d_min` is optional | It is **required** and controls the resolution of the output MTZ |

---

## Minimal working example (LCLS SFX, mark0 with PDB reference)

```phil
input.path = /path/to/indexed/results/
scaling.model = reference.pdb
merging.d_min = 2.5
output.output_dir = merge_output/
output.prefix = myexpt
mp.psana2_mode = False
```

## Minimal working example (mark1 / no model)

```phil
input.path = /path/to/indexed/results/
scaling.algorithm = mark1
scaling.unit_cell = 78 78 37 90 90 90
scaling.space_group = P43212
merging.d_min = 2.5
output.output_dir = merge_output/
```
