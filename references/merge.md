# cctbx.xfel.merge

**Source:** `xfel/merging/command_line/merge.py`  
**Phil definitions:** `xfel/merging/application/phil/phil.py`  
**Purpose:** Merge integrated stills data from dials.stills_process (or cctbx.xfel.process) into a final MTZ file. Runs MPI-parallel.

## Usage

```bash
mpirun -n <N> cctbx.xfel.merge params.phil
```

The program reads `_integrated.expt` / `_integrated.refl` file pairs produced by indexing/integration.

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

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `filter.algorithm` | None | choice(multi): n_obs/resolution/unit_cell/energy | Which filters to apply |
| `filter.n_obs.min` | None | int | Minimum reflections per image |
| `filter.n_obs.max` | None | int | Maximum reflections per image |
| `filter.resolution.d_min` | None | float | Reject image unless it has reflections beyond this d_min (Å) |
| `filter.resolution.model_or_image` | model/image | choice | Compute resolution from scaling model or from image |
| `filter.unit_cell.algorithm` | value | choice: range/value/cluster | Unit cell filtering method |
| `filter.unit_cell.value.target_unit_cell` | Auto | unit_cell | Target unit cell; uses scaling model if Auto |
| `filter.unit_cell.value.relative_length_tolerance` | 0.1 | float | Fractional tolerance on unit cell lengths |
| `filter.unit_cell.value.absolute_angle_tolerance` | 2.0 | float | Absolute tolerance on cell angles (degrees) |
| `filter.unit_cell.value.target_space_group` | Auto | space_group | Target space group |
| `filter.outlier.min_corr` | 0.1 | float | Minimum correlation to model for an experiment to pass. Set -1 to disable |
| `filter.outlier.mad_thresh` | None | float | MAD-based outlier threshold. E.g. 3 = reject reflections >3σ from median |
| `filter.energy.min_eV` | None | float | Minimum beam energy (eV) |
| `filter.energy.max_eV` | None | float | Maximum beam energy (eV) |

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

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `select.algorithm` | None | choice(multi): panel/cspad_sensor/significance_filter/isolation_forest | Which per-reflection filters to apply |
| `select.significance_filter.sigma` | 0.5 | float | Remove high-resolution bins until all accepted bins have <I/σ> ≥ sigma |
| `select.significance_filter.n_bins` | 12 | int | Initial number of resolution bins for sigma cutoff |
| `select.significance_filter.d_min` | None | float | Reject entire lattice if resolution doesn't reach this d_min |
| `select.cspad_sensor.number` | None | int (multiple) | CSPAD sensor index (0–31) to deselect from merging |

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
| `merging.error.model` | ev11 | choice: ha14/ev11/mm24/errors_from_sample_residuals | Error model. ev11 (Evans 2011 sdfac refinement) is the standard choice |
| `merging.error.ev11.minimizer` | lbfgs | choice: lbfgs/LevMar | Minimizer for ev11 error refinement |
| `merging.error.mm24.likelihood` | t-dist | choice: normal/t-dist | Likelihood for mm24 (Mittan-Moreau 2024) |

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
