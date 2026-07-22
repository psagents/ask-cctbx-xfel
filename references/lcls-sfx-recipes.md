# LCLS SFX Recipes and Good Defaults

Practical guidance for processing serial femtosecond crystallography data at LCLS using the cctbx.xfel pipeline.

---

## Typical full pipeline overview

```
Raw XTC data
    │
    ▼
cctbx.xfel.xtc_process  (or cctbx.xfel.process for pre-converted data)
    │  spotfinding + indexing + integration (via dials.stills_process)
    │  → r0025/003/output/*_integrated.expt, *_integrated.refl
    │
    ▼
cctbx.xfel.filter_experiments_by_rmsd  (optional quality gate)
    │  → filtered.expt, filtered.refl
    │
    ▼
cctbx.xfel.time_varying_refinement  (ensemble detector refinement)
    │  groups by rungroup, splits into chunks, runs dials.refine per chunk
    │  → tder/task_NNN/refined.expt
    │
    ▼
cctbx.xfel.mpi_integrate  (re-integrate with refined geometry)
    │  → re_integrated.expt, re_integrated.refl
    │
    ▼
cctbx.xfel.merge  (MPI merge → MTZ)
    │  → iobs_all.mtz, iobs_odd.mtz, iobs_even.mtz
    │
    ▼
cctbx.xfel.upload_mtz  (optional: share results)
```

---

## Detector-specific defaults

### ePix10k (most common at CXI, MFX, MEC)

The ePix10k detector's gain is handled internally by the detector model; do **not** set a manual gain in the DIALS spotfinding scope.

```phil
# xtc_process
input.address = MfxEndstation.0:Epix10ka2M.0   # or CxiDs1.0:Epix10ka2M.0
format.file_format = cbf
format.cbf.mode = cspad                         # ePix uses the CSPAD-style CBF path
format.cbf.detz_offset = <measured value>       # check run metadata
format.cbf.invalid_pixel_mask = mask.pickle     # use cxi.make_dials_mask output

# spotfinding — in dials.stills_process scope (ask DIALS skill for full list)
spotfinder.threshold.dispersion.sigma_strong = 3.0
spotfinder.threshold.dispersion.sigma_background = 6.0
spotfinder.filter.min_spot_size = 2
```

### Jungfrau 16M (CXI hutch, hard X-ray)

```phil
input.address = CxiDs1.0:Jungfrau.0
format.cbf.mode = cspad
format.cbf.detz_offset = <value>
format.cbf.cspad.gain_mask_value = None   # Jungfrau auto-gain; do not set manually
```

For geometry quality checks after processing, use `cctbx.xfel.jungfrau_metrics`.

### CSPAD (older experiments, still in use)

```phil
input.address = CxiDs2.0:Cspad.0
format.cbf.mode = cspad
format.cbf.detz_offset = 585              # typical CXI value; verify from run
format.cbf.cspad.common_mode.algorithm = default
format.cbf.cspad.gain_mask_value = 6.85  # only if not using per-pixel gain
```

### Rayonix (XPP, MFX)

```phil
format.cbf.mode = rayonix
format.cbf.rayonix.bin_size = 2          # use 1 for full resolution
format.cbf.detz_offset = <detector distance>
```

---

## Hitfinding tuning

The hitfinder threshold controls what fraction of events are processed by DIALS. Too low → wasted computation on noise; too high → miss weak hits.

```phil
dispatch.hit_finder.enable = True
dispatch.hit_finder.minimum_number_of_reflections = 16   # start here
# Lower to 8–10 for low-concentration samples
# Raise to 25–50 for very high signal
dispatch.hit_finder.maximum_number_of_reflections = None  # set if ice rings cause false positives
```

### Quick hit-rate survey

Process a small fraction first to estimate hit rate before committing full resources:
```phil
dispatch.process_percent = 10   # process 10% of events
dispatch.find_spots = True
dispatch.index = False
dispatch.integrate = False
```

---

## Output file management (composite mode)

For large runs (>10k events per rank), always use composite output to avoid filesystem inode exhaustion:

```phil
output.composite_output = True
mp.composite_stride = 64    # match to nproc_per_node to get 1 file per node
```

---

## cctbx.xfel.merge — recommended starting parameters

### First-pass merge (mark0, with PDB reference)

```phil
input.path = /results/r0025/003/output/
scaling.model = reference.pdb
merging.d_min = 2.5          # adjust to your data
merging.d_max = 50.0
merging.error.model = ev11
filter.outlier.min_corr = 0.1
filter.n_obs.min = 10
output.output_dir = merge_v01/
output.prefix = myexpt_v01
mp.psana2_mode = False       # True only for XTC2/psana2 integrated data
```

### Mark1 merge (no reference, unit cell only)

Use when you have no reference model or want unbiased data:

```phil
input.path = /results/r0025/003/output/
scaling.algorithm = mark1
scaling.unit_cell = 79.3 79.3 37.6 90 90 90
scaling.space_group = P43212
merging.d_min = 2.5
merging.error.model = ev11
filter.n_obs.min = 10
output.output_dir = merge_mark1/
```

### With postrefinement (rs2 algorithm)

```phil
input.path = /results/
scaling.model = reference.pdb
merging.d_min = 2.0
postrefinement.enable = True
postrefinement.algorithm = rs2
postrefinement.target_weighting = gentle
merging.error.model = ev11
output.output_dir = merge_postrefine/
```

### Unit cell filtering

Always filter to avoid contaminating the merge with wrong-cell images:

```phil
filter.algorithm = unit_cell
filter.unit_cell.algorithm = value
filter.unit_cell.value.target_unit_cell = 79.3 79.3 37.6 90 90 90
filter.unit_cell.value.relative_length_tolerance = 0.05  # 5% tolerance
filter.unit_cell.value.absolute_angle_tolerance = 1.0
filter.unit_cell.value.target_space_group = P43212
```

### Resolution-adaptive filtering

```phil
filter.algorithm = unit_cell resolution
filter.resolution.d_min = 3.5
filter.resolution.model_or_image = image
```

---

## cctbx.xfel.time_varying_refinement — ensemble refinement setup

Standard LCLS SFX ensemble refinement (TDER):

```phil
time_varying_refinement.results_dir = /results
time_varying_refinement.trial = 3
time_varying_refinement.chunk_size = 1000
time_varying_refinement.stripe = False           # use contiguous chunks (default)
time_varying_refinement.respect_rungroup_barriers = True
combine_experiments.keep_integrated = False      # use indexed.refl for refinement input
mp.method = slurm
mp.nnodes = 1
mp.nproc_per_node = 32
mp.wall_time = 60
```

---

## Photon energy calibration

`xfel.fee_calibration` (note: no `cctbx.` prefix) derives an energy-per-pixel calibration from
FEE spectrometer notch scans — the CLI counterpart to the GUI's "Energy" page. Run it against the
runs from an energy-scan, where a known narrow energy band ("notch") was removed from the beam
for each run:

```bash
xfel.fee_calibration experiment=mfxl1013621 output_phil=fee_calib.phil max_events=1000
```

This writes a phil file with the fitted `spectrum_eV_per_pixel` / `spectrum_ev_offset` values,
which get entered directly into the GUI's run-group "Spectrum calibration" fields (or the
equivalent `format.cbf.filter`-adjacent XTC format params) — see
[gui-workflow.md](gui-workflow.md#run-groups-within-trials).

---

## S3DF Slurm submit pattern

```phil
# submit.phil for S3DF
input.experiment = cxic0415
input.run_num = 25
input.dispatcher = cctbx.xfel.process
input.target = stills_process.phil
output.output_dir = /sdf/data/lcls/ds/cxi/cxic0415/scratch/username/results
mp.method = slurm
mp.nproc = 128
mp.nproc_per_node = 64
mp.nnodes = 2
mp.queue = milano
mp.wall_time = 120
mp.env_script = /sdf/group/lcls/ds/tools/cctbx/psana2/setup.sh
```

Then submit:
```bash
cctbx.xfel.submit_job submit.phil
```

---

## Common failure modes

| Symptom | Likely cause | Fix |
|---|---|---|
| Low hit rate (<1%) | `dispatch.hit_finder.minimum_number_of_reflections` too high | Lower to 8–10 |
| All events pass hitfinding but none index | Incorrect geometry (`detz_offset`) or missing `known_symmetry` | Check `format.cbf.detz_offset` and supply `known_symmetry` to stills_process |
| MPI merge produces empty MTZ | `merging.d_min` too tight, or `input.path` doesn't find `.expt` files | Check `input.reflections_suffix` / `input.experiments_suffix` match your filenames |
| `gain mismatch` error in spotfinding | Setting `spotfinder.threshold.dispersion.gain` when detector model already handles gain | Remove gain from spotfinder; set in one place only |
| `cctbx.xfel.mpi_submit raises Sorry` | Renamed executable | Use `cctbx.xfel.submit_job` instead |
| `cctbx.xfel.stripe_experiment raises Sorry` | Renamed executable | Use `cctbx.xfel.time_varying_refinement` instead |
| Merge fails with MPI alltoall memory error | Too many reflections for available RAM | Increase `parallel.a2a` in merge phil |
| Low CC½ after merge | Images from multiple isoforms mixed | Add unit cell filter (`filter.algorithm = unit_cell`) |
| `xtc_process` errors on startup or finds no images | Wrong `input.address` for the actual detector alias | Cross-check against the `detnames` psana tool output for that experiment/run |
| `xtc_process` processes 0 events, no error | `format.cbf.filter.required_present_codes` doesn't match any EVR code actually present on the run | Verify the run's real event codes before assuming a bug; drop the filter to confirm events flow, then re-add with correct codes |
| GUI shows a run but processing can't find data | Run-group "Extra XTC format parameters" (which populate the `data.loc` file) don't match the DAQ alias, or `mode=psana2_idx` is missing for psana2 data | Check/regenerate `data.loc` for that run group; see [gui-workflow.md](gui-workflow.md#run-groups-within-trials) |
