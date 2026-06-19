# Analysis, Statistics and Post-Processing Tools

---

## cctbx.xfel.weather

**Source:** `xfel/command_line/weather.py`, `xfel/util/weather.py`  
**Purpose:** Visualize MPI processing performance — shows wall time vs MPI rank number with each data point representing a frame processed by `dials.stills_process`. Read from the `debug/` directory created during processing.

### Usage
```bash
cctbx.xfel.weather input_path=r0025/003/output/debug
```

### Parameters

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `input_path` | `.` | str | Path to the `debug/` directory from a processing run |
| `num_nodes` | 1 | int | Number of nodes used in the processing job |
| `num_cores_per_node` | 72 | int | Cores per node (default is Cori KNL) |
| `wall_time` | None | int | Total wall time (seconds) for the job |
| `plot_title` | Computational weather plot | str | Title of the plot |
| `show_plot` | True | bool | Display the plot on screen |
| `pickle_plot` | False | bool | Save the matplotlib session to a pickle file |
| `pickle_filename` | fig_object.pickle | str | Name for the pickled matplotlib session |

---

## cctbx.xfel.drift

**Source:** `xfel/command_line/drift.py`, `xfel/util/drift.py`  
**Purpose:** Visualize spatial drift of the detector and unit cell parameters as a function of experimental progress (run number / chunk number). Tracks detector origin and unit cell lengths over time. Can read from processed results directories or from a pickle cache.

### Usage
```bash
# Basic: read all batch* directories, exclude batch5
cctbx.xfel.drift scrap.input.glob=batch* scrap.input.exclude=batch5

# With caching
cctbx.xfel.drift scrap.input.glob=batch* scrap.cache.action=write scrap.cache.glob=name.pkl

# From cached pickle
cctbx.xfel.drift scrap.cache.action=read scrap.cache.glob="*.pkl" plot.save=True
```

### Parameters (scrap scope — data collection)

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `scrap.input.glob` | None | str (multiple) | Glob pattern matching processed result directories |
| `scrap.input.exclude` | None | str (multiple) | Directories to exclude |
| `scrap.input.kind` | batch_directory | choice | Type of input: batch_directory or tder_task_directory |
| `scrap.cache.action` | none | choice: none/read/write | Whether to read or write a pickle cache |
| `scrap.cache.glob` | None | str | Glob or path for the cache file |

### Parameters (plot scope)

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `plot.save` | False | bool | Save plot to file instead of displaying |
| `plot.format` | png | str | Output format (png, pdf, etc.) |

---

## cctbx.xfel.filter_experiments_by_rmsd

**Source:** `xfel/command_line/filter_experiments_by_rmsd.py`  
**Purpose:** Filter a multi-experiment set by per-image RMSD using Tukey's rule. For each experiment, compute the RMSD of (predicted − observed) positions, then reject outlier images whose RMSD is more than `iqr_multiplier × IQR` above the third quartile. The standard quality gate after indexing.

### Usage
```bash
cctbx.xfel.filter_experiments_by_rmsd combined.expt combined.refl \
  output.filtered_experiments=good.expt output.filtered_reflections=good.refl
```

### Parameters

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `iqr_multiplier` | 1.5 | float | IQR multiplier for Tukey's rule. 1.5 = standard Tukey; decrease to be more aggressive |
| `show_plots` | False | bool | Show diagnostic plots |
| `max_delta` | None | float | Before Tukey filtering, remove all reflections with obs–pred > max_delta mm |
| `detector` | None | int | Only filter experiments matching this detector index |
| `delta_psi_filter` | None | float | After RMSD filter, also filter by delta_psi angle (degrees) |
| `output.filtered_experiments` | filtered.expt | str | Output file for kept experiments |
| `output.filtered_reflections` | filtered.refl | str | Output file for kept reflections |

---

## cctbx.xfel.recompute_mosaicity

**Source:** `xfel/command_line/recompute_mosaicity.py`  
**Purpose:** Recompute the Nave mosaic parameters (domain size and half-angle) for a set of already-indexed/integrated experiments and apply outlier rejection. Useful after ensemble refinement to update mosaic parameters with the improved geometry.

### Usage
```bash
cctbx.xfel.recompute_mosaicity refined.expt refined.refl
```

### Parameters

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `plot_changes` | False | bool | Plot the change in mosaic parameters before/after |
| `output.experiments` | refined.expt | str | Output experiments file |
| `output.reflections` | refined.refl | str | Output reflections file |

---

## cctbx.xfel.trumpet_plots

**Source:** `xfel/command_line/trumpet_plots.py`  
**Purpose:** Generate trumpet plots from DIALS output — one per experiment. Trumpet plots show the prediction offset as a function of resolution (Sauter 2014), useful for assessing mosaicity and partiality.

Reference: Sauter 2014, https://doi.org/10.1107/S1399004714024134

### Usage
```bash
cctbx.xfel.trumpet_plots experiment.expt reflections.refl
```

### Parameters

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `max_plots` | 10 | int | Maximum number of plots to generate |
| `repredict_input_reflections` | True | bool | Repredict positions using input models before plotting |

---

## cctbx.xfel.show_spot_separation

**Source:** `xfel/command_line/show_spot_separation.py`, `xfel/util/show_spot_separation.py`  
**Purpose:** Compute and display statistics on the angular separation between strong spots, as a diagnostic for whether images are real crystalline hits vs noise.

### Usage
```bash
cctbx.xfel.show_spot_separation strong.refl
```

---

## cctbx.xfel.frame_extractor

**Source:** `xfel/command_line/frame_extractor.py`  
**Purpose:** Convert DIALS `.expt` + `.refl` files (from `dials.stills_process`) into the legacy cctbx.xfel pickle format (`.pickle`). Needed when downstream tools (e.g. old-style `cxi.merge`) expect pickle input.

### Usage
```bash
cctbx.xfel.frame_extractor \
  input.experiments=integrated.expt input.reflections=integrated.refl \
  output.dirname=/output/dir
```

### Parameters

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `input.experiments` | None | path | Path to experiments `.expt` file (or glob) |
| `input.reflections` | None | path | Path to reflection table `.refl` file (or glob) |
| `output.filename` | None | str | Explicit output pickle filename |
| `output.dirname` | None | path | Output directory for pickle files |

---

## cctbx.xfel.h5_average

**Source:** `xfel/command_line/h5_average.py`  
**Purpose:** Compute the average, maximum projection, or standard deviation image from a multi-image HDF5 file. Useful for generating background/dark reference images.

### Usage
```bash
cctbx.xfel.h5_average run25792.h5
```

### Parameters

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `average` | True | bool | Generate average image |
| `max` | True | bool | Generate maximum projection |
| `stddev` | True | bool | Generate standard deviation image |

---

## cctbx.xfel.mpi_integrate

**Source:** `xfel/merging/command_line/mpi_integrate.py`  
**Purpose:** Re-integrate already-indexed experiments in parallel, without re-running spotfinding or indexing. Useful when you want to re-integrate with updated geometry (after ensemble refinement) or different integration parameters.

### Usage
```bash
mpirun -n <N> cctbx.xfel.mpi_integrate input.path=/indexed/results scaling.model=ref.pdb merging.d_min=2.5
```

This program uses the same `input`, `mp`, and `output` phil as `cctbx.xfel.merge`, plus an integration-specific scope.

### Additional parameters (integration scope)

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `integration.debug.delete_shoeboxes` | True | bool | Delete shoeboxes to save memory |
| `integration.integration_only_overrides.trusted_range` | None | floats(size=2) | Override panel trusted range (underload, saturation) during integration |
| `integration.recruitment.expand_nave_parameters` | False | bool | Expand Nave trumpet envelope to capture more signal after time-varying refinement |
| `integration.recruitment.expansion_factor` | 2.0 | float | Expansion factor for the Nave trumpet. Note: stills_process already encodes 1.4x |
| `integration.recruitment.significance_cutoff` | 2.0 | float | I/σ cutoff when re-determining Nave parameters after expansion |
| `dispatch.coset` | False | bool | Integrate a coset sublattice as negative control |

---

## cctbx.xfel.upload_mtz

**Source:** `xfel/command_line/upload_mtz.py`  
**Purpose:** Upload a merged `.mtz` file and its log to a shared Google Drive folder via a service account credential. Used for sharing results.

### Parameters

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `drive.credential_file` | None | path | Google Cloud service account credential JSON file |
| `drive.shared_folder_id` | None | str | ID of the destination Google Drive folder |
| `input.mtz_file` | None | path | MTZ file to upload |
| `input.log_file` | None | path | Log file to upload (guessed from MTZ name if None) |
| `input.other_files` | None | path (multiple) | Additional files to upload |
| `input.dataset_root` | None | path | Path to folder with dataset versions |
| `input.version` | None | int | Dataset version number (guessed from MTZ name if None) |
| `input.guess_root_and_version` | True | bool | Automatically infer root and version from MTZ filename |
| `input.disregard_mtz` | False | bool | Skip uploading the MTZ (upload log only) |
