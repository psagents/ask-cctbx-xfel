# Job Submission and Workflow Tools

---

## cctbx.xfel.submit_job

**Source:** `xfel/command_line/submit_job.py`  
**Note:** Formerly `cctbx.xfel.mpi_submit` (now raises Sorry — use `submit_job`).  
**Purpose:** Submit a cctbx.xfel processing job to a cluster. Creates a structured trial directory, copies and updates phil/config files, then submits via the requested queuing system.

### What it does
1. Creates output directory structure: `<output_dir>/r<run>/trial_NNN/`
2. Copies and path-updates psana config files and phil files
3. Writes a `submit.sh` with the final submit command
4. Submits unless `dry_run=True`

### Usage
```bash
cctbx.xfel.submit_job input.experiment=cxic0415 input.run_num=25 \
  input.address=CxiDs2.0:Cspad.0 output.output_dir=/results \
  mp.nproc=100 mp.queue=psanaq
# or with a phil file:
cctbx.xfel.submit_job submit.phil
```

### Parameters — input scope

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `input.experiment` | None | str | LCLS experiment name (mandatory for XTC processing) |
| `input.run_num` | None | str | Run number to process |
| `input.trial` | None | int | Trial number; auto-determined from output directory if None |
| `input.rungroup` | None | int | Optional rungroup identifier appended as `_rgXXX` to trial path |
| `input.dispatcher` | `cctbx.xfel.xtc_process` | str | Which processing program to run. Use `cctbx.xfel.process` for non-XTC data or `dials.stills_process` for generic stills |
| `input.target` | None | path | Optional path to a phil file with additional processing parameters |
| `input.locator` | None | str | Locator file for `cctbx.xfel.process` to find XTC streams |
| `input.cfg` | None | str | Path to psana config file |
| `input.task` | None | int | Task number for post-processing jobs (uses existing trial folder) |
| `dry_run` | False | bool | Create trial directory and show command but do not submit |

### Parameters — output scope

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `output.output_dir` | `.` | str | Base directory for all output |
| `output.add_output_dir_option` | True | bool | Append `output.output_dir` to submitted command |
| `output.split_logs` | True | bool | Split stdout/stderr into per-process files |

### Parameters — mp scope (multiprocessing)

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `mp.method` | lsf | choice: local/lsf/sge/pbs/slurm/shifter/htcondor/custom | Queuing system. Use `slurm` on S3DF |
| `mp.use_mpi` | True | bool | Use MPI multiprocessing |
| `mp.mpi_command` | mpirun | str | MPI launch command (can include extra flags) |
| `mp.nproc` | 1 | int | Total number of processes |
| `mp.nnodes` | 1 | int | Number of nodes to request |
| `mp.nproc_per_node` | 1 | int | Processes per node |
| `mp.nnodes_index` | None | int | Nodes for indexing/integration step (slurm/shifter only) |
| `mp.nnodes_tder` | None | int | Nodes for ensemble refinement step (slurm/shifter only) |
| `mp.nnodes_scale` | None | int | Nodes for scaling step (slurm/shifter only) |
| `mp.nnodes_merge` | None | int | Nodes for merging step (slurm/shifter only) |
| `mp.queue` | None | str | Queue name to submit to |
| `mp.memory` | None | int | Memory to allocate in MB |
| `mp.wall_time` | None | int | Wall time limit in minutes |
| `mp.max_queued` | None | int | Max number of jobs queued or running at once |
| `mp.env_script` | None | str (multiple) | Scripts to source to set up the environment (needed on SLURM) |
| `mp.extra_options` | None | str (multiple) | Additional options for the queue submission command |
| `mp.extra_args` | None | str (multiple) | Additional arguments appended to the processing command |

### S3DF/SLURM example
```phil
input.experiment = cxic0415
input.run_num = 25
input.dispatcher = cctbx.xfel.process
input.target = stills_process.phil
output.output_dir = /sdf/data/lcls/ds/cxi/cxic0415/scratch/user/results
mp.method = slurm
mp.nproc = 200
mp.nproc_per_node = 64
mp.nnodes = 4
mp.queue = milano
mp.wall_time = 60
mp.env_script = /path/to/setup.sh
```

---

## cctbx.xfel.time_varying_refinement

**Source:** `xfel/command_line/time_varying_refinement.py`  
**Note:** Formerly `cctbx.xfel.stripe_experiment` (now raises Sorry — use `time_varying_refinement`).  
**Purpose:** Group indexed/integrated results by run and rungroup, split into time-ordered chunks (or stripes), and run `dials.combine_experiments` + `dials.refine` on each chunk. This is the **ensemble detector refinement** step — the key tool for correcting time-varying detector geometry.

### Concept
After indexing, the detector geometry may drift over time (thermal expansion, beam drift). This program divides the processed images into time chunks, refines the detector for each chunk, then produces refined geometry files ready for re-integration.

### Usage
```bash
cctbx.xfel.time_varying_refinement \
  time_varying_refinement.results_dir=/results \
  time_varying_refinement.trial=0 \
  time_varying_refinement.chunk_size=1000
```

### Parameters — time_varying_refinement scope

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `time_varying_refinement.results_dir` | None | path | cctbx.xfel results directory containing run subdirs (`r0001/`, etc.) |
| `time_varying_refinement.trial` | None | int | Trial identifier for cctbx.xfel processing output |
| `time_varying_refinement.rungroup` | None | int (multiple) | Selected rungroups; None = all |
| `time_varying_refinement.run` | None | str (multiple) | Selected runs; None = all |
| `time_varying_refinement.chunk_size` | 1000 | float | Maximum images per chunk/stripe |
| `time_varying_refinement.stripe` | False | bool | Use evenly-spaced stripes instead of contiguous chunks |
| `time_varying_refinement.respect_rungroup_barriers` | True | bool | Keep rungroups separate (default). Set False to allow multi-rungroup detector models |
| `time_varying_refinement.dry_run` | False | bool | Set up but don't execute jobs |
| `time_varying_refinement.pre_split` | False | bool | Load data and chunk before submitting (by lattice count). If False, chunk by file count |
| `time_varying_refinement.output_folder` | None | path | Output path; defaults to current directory |

### Parameters — combine_experiments scope

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `combine_experiments.keep_integrated` | False | bool | Use integrated.refl files (True) vs indexed.refl files (False) for input to combine |
| `combine_experiments.clustering.dendrogram` | False | bool | Interactive dendrogram mode for clustering (forces single-process) |

---

## cctbx.xfel.xtc_dump

**Source:** `xfel/command_line/xtc_dump.py`  
**Purpose:** Dump raw LCLS XTC stream events to CBF or pickle files without running the DIALS processing pipeline. Useful for inspecting raw data or generating test images.

### Input
```bash
cctbx.xfel.xtc_dump input.experiment=cxic0415 input.run_num=25 \
  input.address=CxiDs2.0:Cspad.0 format.cbf.detz_offset=585
```

### Parameters

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `dispatch.max_events` | None | int | Dump only this many events |
| `dispatch.selected_events` | False | bool | Only dump events listed in `input.timestamp` |
| `input.experiment` | None | str | LCLS experiment name |
| `input.run_num` | None | int | Run number |
| `input.address` | None | str | Detector address |
| `input.timestamp` | None | str (multiple) | Human-readable event timestamps to dump |
| `input.xtc_dir` | None | str | Non-standard XTC data directory |
| `input.calib_dir` | None | str | Non-standard calibration directory |
| `format.file_format` | cbf | choice: cbf/pickle | Output format |
| `format.cbf.detz_offset` | None | float | Detector distance offset (mm), required for CBF |
| `format.cbf.override_energy` | None | float | Override beam energy (eV) |
| `format.cbf.mode` | cspad | choice: cspad/rayonix | Detector type |
| `format.cbf.cspad.gain_mask_value` | None | float | Low-gain pixel correction factor |
