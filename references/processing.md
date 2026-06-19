# cctbx.xfel.process and cctbx.xfel.xtc_process

## Overview

These are the two LCLS-specific wrappers around `dials.stills_process` that add:
- LCLS experiment/run/detector routing and psana data reading
- Database logging (`experiment_tag` parameter)
- Radial average calculation per event
- XTC stream handling

**For spotfinding, indexing, integration, and refinement parameters** (the core processing parameters), refer to the **ask-dials** skill — these programs inherit the full `dials.stills_process` phil scope unchanged.

This document covers only the LCLS-specific additions.

---

## cctbx.xfel.process

**Source:** `xfel/command_line/xfel_process.py`  
**Dispatcher:** `cctbx.xfel.process`  
**Purpose:** Process pre-formatted still images (CBF, HDF5, etc.) with DIALS, adding database logging. This is the modern replacement for `cctbx.xfel.xtc_process` when data has already been converted from XTC streams, or when processing SACLA/SwissFEL data.

### Additional parameters (beyond dials.stills_process)

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `input.run_num` | None | str | Run name/number for database logging |
| `input.trial` | None | int | Trial number for this run |
| `input.rungroup` | None | int | Rungroup identifier for organizing similar-parameter runs |
| `radial_average.enable` | False | bool | Compute radial average per image |
| `radial_average.two_theta_low` | None | float | Log radial average at this 2θ (degrees) in database |
| `radial_average.two_theta_high` | None | float | Log radial average at this 2θ (degrees) in database |

### Usage

```bash
mpirun -n <N> cctbx.xfel.process params.phil input_files.cbf
# or via file list:
mpirun -n <N> cctbx.xfel.process params.phil filelist.txt
```

---

## cctbx.xfel.xtc_process

**Source:** `xfel/command_line/xtc_process.py`  
**Dispatcher:** `cctbx.xfel.xtc_process`  
**Purpose:** Process raw LCLS XTC stream data directly. Reads from psana, converts to CBF on-the-fly, then runs the DIALS processing pipeline. The legacy equivalent is `cxi.xtc_process`.

> **Note:** For modern LCLS-II / psana2 data (XTC2), use `mp.psana2_mode = True` in `cctbx.xfel.merge` and ensure `PSANA2_VERSION` is set. The per-event processing path is handled differently under psana2.

### LCLS-specific parameters

#### dispatch scope

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `dispatch.max_events` | None | int | Process only this many events (useful for testing) |
| `dispatch.process_percent` | None | int (1–100) | Process only this fraction of events (for quick surveys) |
| `dispatch.estimate_gain_only` | False | bool | Print gain estimates per event and exit |
| `dispatch.find_spots` | True | bool | Run spotfinding (disable for raw dump only) |
| `dispatch.datasource` | None | str | Override psana datasource string, e.g. `exp=mfxo1916:run=20:xtc` |
| `dispatch.hit_finder.enable` | True | bool | Enable hitfinding; images below min reflections are skipped |
| `dispatch.hit_finder.minimum_number_of_reflections` | 16 | int | Minimum strong reflections to accept an image as a hit |
| `dispatch.hit_finder.maximum_number_of_reflections` | None | int | Reject images with more reflections than this |
| `dispatch.index` | True | bool | Run indexing |
| `dispatch.refine` | False | bool | Run per-image refinement after indexing |
| `dispatch.integrate` | True | bool | Run integration |
| `dispatch.dump_strong` | False | bool | Save strongly-diffracting images to CBF |
| `dispatch.dump_indexed` | True | bool | Save indexed images to CBF |
| `dispatch.dump_all` | False | bool | Save all images to CBF |

#### input scope

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `input.experiment` | None | str | **Required.** LCLS experiment name, e.g. `cxic0415` |
| `input.run_num` | None | int | **Required.** Run number to process |
| `input.address` | None | str | **Required (CBF mode).** Detector address, e.g. `CxiDs2.0:Cspad.0` or alias `Ds1CsPad` |
| `input.cfg` | None | str | Path to psana config file (needed for pickle format only) |
| `input.use_ffb` | False | bool | Read from LCLS fast-feedback buffer (active users only) |
| `input.xtc_dir` | None | str | Non-standard XTC data directory |
| `input.calib_dir` | None | str | Non-standard calibration directory |
| `input.trial` | None | int | Trial number for database logging |
| `input.rungroup` | None | int | Rungroup for organizing runs with similar parameters |
| `input.override_spotfinding_trusted_max` | None | int | Override saturation value during spotfinding |
| `input.override_spotfinding_trusted_min` | None | int | Override minimum pixel value during spotfinding |
| `input.override_integration_trusted_max` | None | int | Override saturation value during integration |
| `input.override_integration_trusted_min` | None | int | Override minimum pixel value during integration |
| `input.ignore_gain_mismatch` | False | bool | Suppress gain mismatch check (use with caution) |

#### format scope

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `format.file_format` | cbf | choice: cbf/pickle | Output format: CBF (standard) or image pickle (legacy) |
| `format.cbf.detz_offset` | None | float | **Required (CBF mode).** Distance from back of detector rail to sample (CXI) or detector distance (XPP/MFX), in mm |
| `format.cbf.override_energy` | None | float | Use this energy (eV) for all events instead of reading from XTC |
| `format.cbf.override_distance` | None | float | Use this distance (mm) for all events |
| `format.cbf.invalid_pixel_mask` | None | str | Path to DIALS-format invalid pixel mask |
| `format.cbf.mode` | cspad | choice: cspad/rayonix | Detector type |
| `format.cbf.cspad.gain_mask_value` | None | float | Multiply low-gain pixels by this value using the psana gain mask |
| `format.cbf.cspad.per_pixel_gain` | False | bool | Apply per-pixel gain from calib folder |
| `format.cbf.cspad.additional_gain_factor` | None | float | Additional gain correction applied after all others |
| `format.cbf.cspad.common_mode.algorithm` | None | choice: default/custom | Common mode correction. None = only dark pedestal subtraction |
| `format.cbf.cspad.common_mode.custom_parameterization` | None | ints | Parameters for custom common mode algorithm |
| `format.cbf.rayonix.bin_size` | 2 | int | Rayonix detector binning mode |
| `format.cbf.rayonix.override_beam_x` | None | float | Override beam center X (pixels) |
| `format.cbf.rayonix.override_beam_y` | None | float | Override beam center Y (pixels) |

#### output scope

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `output.output_dir` | `.` | str | **Required.** Directory for output files |
| `output.composite_output` | True | bool | Save one expt/refl per MPI rank (True) vs one per image (False). True is strongly preferred for large datasets |
| `output.delete_integration_shoeboxes` | True | bool | Delete shoeboxes after integration to save memory |
| `output.logging_dir` | None | str | Directory for per-rank log files. Set to `DevNull` to suppress all output |
| `output.experiments_filename` | `%s.expt` | str | Template for output experiment file names |
| `output.integrated_filename` | `%s_integrated.refl` | str | Template for integrated reflections |
| `output.integrated_experiments_filename` | `%s_integrated.expt` | str | Template for integrated experiments |

#### mp scope (MPI)

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `mp.method` | mpi | choice: mpi/sge | Multiprocessing method |
| `mp.mpi.method` | client_server | choice: client_server/striping | client_server: one rank serves timestamps; striping: each rank takes every Nth event |
| `mp.composite_stride` | None | int | Aggregate results from this many ranks. Set to nproc_per_node to reduce output files |

#### debug scope

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `debug.skip_processed_events` | False | bool | Skip events already successfully processed |
| `debug.skip_unprocessed_events` | False | bool | Skip events not yet processed |
| `debug.skip_bad_events` | False | bool | Skip events that caused unhandled exceptions |
| `debug.event_timestamp` | None | str (multiple) | Process only events with these timestamps |

#### joint_reintegration scope

| Parameter | Default | Type | Notes |
|---|---|---|---|
| `joint_reintegration.enable` | False | bool | After processing, do joint refinement and re-integration using combined detector model |
| `joint_reintegration.minimum_results` | 30 | int | Minimum indexed images needed to trigger joint reintegration |

---

## Minimum working example (xtc_process)

```phil
input.experiment = cxic0415
input.run_num = 25
input.address = CxiDs2.0:Cspad.0
format.file_format = cbf
format.cbf.detz_offset = 585
output.output_dir = /path/to/output
dispatch.hit_finder.minimum_number_of_reflections = 10
```

Run as:
```bash
mpirun -n 100 cctbx.xfel.xtc_process params.phil
```
