# cctbx.xfel GUI workflow

Most LCLS SFX users drive `cctbx.xfel` through its GUI rather than calling the executables in
this skill directly by hand. The GUI is a front end that generates and submits the same phil
files documented elsewhere in this skill — this reference covers the GUI-specific concepts
(Trial/Dataset/Task, `settings.phil`, remote access) that don't show up in any phil scope.

Content here is adapted from the community handbook at https://lcls-mlcv.github.io/comp-xtal/,
which has more detail on GUI mechanics, interactive geometry/mask tools, and live debugging than
is reproduced here.

**Out of scope for this file** (see the handbook directly): raw MySQL database debugging,
`dials.image_viewer` mask-drawing and unit-cell-fitting mechanics, and step-by-step
geometry-refinement-from-powder procedures. Mask creation itself (`dials.generate_mask`) is
covered by the **ask-dials** skill.

---

## Remote access & environment

Recommended connection methods, in order of preference for S3DF:

| Method | Notes |
|---|---|
| **OnDemand** (S3DF) | Least lag, but no copy/paste. https://s3df.slac.stanford.edu/ondemand → Interactive Apps → S3DF. |
| **ThinLinc** (NERSC) | See https://docs.nersc.gov/connect/thinlinc/ |
| **FastX v4 (StarNet)** | Launch a new Terminal session, then SSH into psana from within it |
| **NoMachine** | Create a virtual desktop, open a terminal within it |

For direct SSH with graphics forwarding:
```bash
ssh -Y <username>@s3dflogin.slac.stanford.edu
ssh -Y psana   # additional hop, depending on SSH config
```

**Sourcing cctbx** — do this manually each session rather than in `.bashrc` (auto-sourcing on
login nodes is discouraged):

```bash
# S3DF, psana2
source /sdf/group/lcls/ds/tools/cctbx/psana2_setup.sh
# S3DF, psana1 (legacy)
source /sdf/group/lcls/ds/tools/cctbx/psana1_setup.sh
# NERSC
source /global/common/software/cctbx/alcc-recipes/cctbx/activate.sh
```

If a setup script is missing, a minimal S3DF equivalent:
```bash
export SIT_DATA=/sdf/group/lcls/ds/ana/data
export SIT_PSDM=/sdf/data/lcls/ds/
export SIT_ROOT=/sdf/data/lcls/ds
export SLURM_PRIORITY_RESERVATION=lcls.onshift
umask 002
source /sdf/group/lcls/ds/tools/cctbx/build/conda_setpaths.sh
```

---

## Initial GUI configuration

### settings.phil

The GUI persists its configuration at:
```
/sdf/home/<username>/.cctbx.xfel/settings.phil
```

This file pre-populates GUI fields and is updated as you edit the interface. It's created
automatically on first GUI launch if the directory doesn't exist. **Back it up per experiment**
before switching experiments:
```bash
cp /sdf/home/<username>/.cctbx.xfel/settings.phil /sdf/home/<username>/.cctbx.xfel/<experiment>.phil
```

> **Gotcha:** output folder paths referenced in the config must already exist on disk — the GUI
> will not create them, and may hang or crash if they're missing.

### Launching

```bash
cctbx.xfel
```
Startup takes roughly a minute. Once loaded, fill in (or verify, if pre-populated from
`settings.phil`):

- **Experiment Tag** (`experiment_tag`)
- **Output** (`output_folder`)
- **Facility** (`facility.name`)
- **Experiment** (`facility.lcls.experiment`)

Convention is for the experiment tag and processing folder to match, but this isn't enforced —
for unofficial/personal processing, use a personal subfolder, e.g.
`/sdf/data/lcls/ds/mfx/mfx100904224/results/<username>/results`.

### Database credentials

Match the GUI's **db** section to `settings.phil`. Name and user are usually the experiment name;
password is often `lcls`. If the DB window's buttons disappear, exit with **Return** or
**Escape**.

### Advanced settings (mp section)

Corresponds to the `mp` scope also used by `cctbx.xfel.submit_job` (see
[job-submission.md](job-submission.md)):

- **Environment setup script** — must match whichever script you sourced above
- **Extra submission arguments** — raw Slurm args, as a string
- Processor counts: Milano nodes have 128 cores but WEKA storage uses 8 — use at most ~120
  processors per node
- For real-time/on-shift work, add `--reservation=lcls:onshift` for higher scheduling priority

### Populating & tagging runs

Click **Watch for new runs** and hold until it turns yellow — only one GUI instance should watch
at a time. Detected runs appear under the **Run** page.

Three tagging tools:
- **Manage Tags** — create new tags
- **Change Tags on Multiple Runs** — apply tags in bulk
- **Manage Persistent Tags** — auto-tag incoming runs while sample conditions are stable

> **Gotcha:** if the GUI crashes during real-time processing, persistent tagging must be
> reconfigured after relaunch, or untagged runs will pile up.

---

## The Trial → Dataset → Task model

This is the core organizing concept of the GUI and has no direct phil-file analogue — it's worth
understanding even if you never touch the GUI, since it determines the folder layout you'll see
in job output.

### Defining Trials

A Trial holds the **indexing-stage** phil — this is the somewhat confusingly-named part of the
GUI where "Trial" parameters really configure the Indexing task specifically. Entered as raw phil
text. To discover available parameters:
```bash
dials.stills_process -ca 2 -e 10
```
(see the **ask-dials** skill for what spotfinding/indexing parameters actually do). Phil syntax
reference: https://cci.lbl.gov/docs/cctbx/doc_low_phil/

Example trial parameters:
```phil
dispatch {
  hit_finder {
    minimum_number_of_reflections = 16
  }
}
spotfinder {
  filter {
    min_spot_size = 2
    max_spot_size = 100
  }
  threshold {
    algorithm = dispersion
    dispersion {
      gain = 1
      kernel_size = 1 1
      global_threshold = 10
    }
  }
}
indexing {
  stills.reflection_subsampling.enable = True
  known_symmetry {
    space_group = P 1 21 1
    unit_cell = 87.970 52.420 95.650 90 95.609 90
  }
  refinement_protocol {
    d_min_start = 2
  }
  multiple_lattice_search {
    max_lattices = 3
  }
}
```

Always double-check **space group** and **unit cell** match the sample. For real-time speed, add
`dispatch.integrate = False` — but remove it (or run ensemble refinement first) before
scaling/merging, since scaling needs integrated reflections.

### Run groups within Trials

A run group specifies a start/end run number (inclusive) plus per-detector-condition settings:

| Setting | Effect |
|---|---|
| Geometry file (`.expt`) | Becomes `input.reference_geometry` for runs in this group |
| Extra XTC format parameters | Free-text phil that populates the `data.loc` file for psana; e.g. add `mode=psana2_idx` for psana2 data |
| Detector name | Must match exactly — cross-check with the `detnames` psana tool if unsure |
| Binning | If applicable to the detector |
| Spectrum calibration | `spectrum_eV_per_pixel`, `spectrum_ev_offset` — enter directly or import via "Import PHIL" from a saved `xfel.fee_calibration` output (see [lcls-sfx-recipes.md](lcls-sfx-recipes.md#photon-energy-calibration)) |
| Pixel mask | Load a DIALS-format mask file |

For real-time processing, "Auto add run" helps extend the run group live, but remember to set an
end run once the sample/condition changes.

### Defining Datasets

A Dataset selects which tagged runs feed into merging, via:
- **Intersection** — combine independent criteria (e.g. time AND temperature)
- **Union** — combine similar runs (e.g. different delivery speeds, same sample)

Only runs that fall within the trial's run groups are actually processed, even if the tag
selection is broader. **Dataset name becomes the merging output folder name** — use a single
word or underscores only; spaces/special characters can make merging fail. Set the dataset active
(and deactivate previous ones) when reprocessing.

### Defining Tasks

Under a dataset, add tasks via "New Task" and associate each with a trial. The standard chain is
four tasks: **Indexing → Ensemble Refinement → Scaling → Merging**. Local tasks (all but Merging)
are chained per-run — a task won't start on a run until the previous local task finishes for that
run.

- Indexing must always be defined explicitly in the dataset, even when reprocessing existing
  indexing results.
- You can chain Indexing directly to Scaling, skipping Ensemble Refinement — but only if
  `dispatch.integrate` was **not** set to `False` at the Indexing/Trial stage.

**Indexing task** — no extra parameters beyond the trial phil. Outputs: `data.loc`, an `out`
folder (`.expt`/`.refl`), `stdout`, `params_1.phil`, `submit.sh`/`submit_submit.sh`. A single
`idx-{number}_refined.expt` may contain multiple images — check `single_file_indices` for actual
indices. View with:
```bash
dials.image_viewer idx-{number}_*.{expt,refl} load_models=False
```

**Ensemble Refinement task** — corresponds to
[cctbx.xfel.time_varying_refinement](lcls-sfx-recipes.md#cctbxxfeltime_varying_refinement--ensemble-refinement-setup).
Example GUI phil:
```phil
combine_experiments.clustering.use = False
reintegration.integration.debug.output = True
reintegration.integration.debug.separate_files = False
reintegration.integration.mp.nproc = 32
reintegration.integration.summation.detector_gain = 0.46   # adjust per detector, e.g. 1 for Epix10ka2M
```
Outputs land under `combine_experiments_t{trial}` with `final_extracted` and `intermediates`
subfolders.

**Scaling task** — runs [cctbx.xfel.merge](merge.md) with a dispatcher tuned for per-run scaling
rather than the final merge:
```phil
dispatch.step_list = input balance model_scaling modify filter errors_premerge scale postrefine statistics_unitcell statistics_beam model_statistics statistics_resolution
input.parallel_file_load.method = uniform
filter.outlier.min_corr = -1
filter.algorithm = unit_cell
filter.unit_cell.value.relative_length_tolerance = 0.01
select.algorithm = significance_filter
select.significance_filter.sigma = 0.1
select.significance_filter.min_ct = 200
select.significance_filter.max_ct = 300
scaling.model = /path/to/reference.pdb
scaling.resolution_scalar = 0.96
merging.d_min = 1.2
merging.merge_anomalous = True
postrefinement.enable = True
statistics.n_bins = 20
output.save_experiments_and_reflections = True
```

**Merging task** — global (runs once across all runs in the dataset), not per-run. Output folder
matches the dataset name. Example phil:
```phil
dispatch.step_list = input model_scaling statistics_unitcell statistics_beam model_statistics statistics_resolution group errors_merge statistics_intensity merge statistics_intensity_cxi publish
input.parallel_file_load.method = uniform
scaling.model = /path/to/reference.pdb
scaling.resolution_scalar = 0.96
statistics.n_bins = 20
merging.d_min = 1.28
merging.merge_anomalous = True
merging.error.model = mm24
publish.drive.credential_file =
publish.drive.shared_folder_id =
```
For Google Drive publishing setup, see
https://github.com/cctbx/cctbx_project/tree/master/xfel/merging/application/publish (also usable
directly with [cctbx.xfel.upload_mtz](analysis-tools.md)).

Key merge output files (versioned by dataset name):
- `{dataset}_v{version}_main.log` — merging statistics; check `obs_multi > 10`, `CC1/2 > 0.3` as
  starting quality thresholds
- `{dataset}_v{version}_all.mtz` — merged data
- `{dataset}_merging{task_id}_v{version}_params.phil` — parameters used
- `{dataset}_merging{task_id}_v{version}_submit.sh` — submission script

### Output folder layout

Local tasks (Indexing, Ensemble Refinement, Scaling) organize per-run:
```
r0023/002_rg015/          # run 23, trial 2, run group 15
r0023/002_rg015/task009/  # non-indexing local tasks get a task-id subfolder
```
Merging (global) outputs to:
```
{dataset_name}_v{version_number}/
```
New versions appear as more data is merged in over time.

---

## Quick workflow checklist

1. Connect via OnDemand/ThinLinc/FastX/NoMachine
2. `ssh -Y` into S3DF if connecting directly
3. Source the cctbx environment script
4. Copy in experiment-specific `settings.phil` (or back up before editing)
5. Launch `cctbx.xfel`, enter experiment info
6. Configure DB credentials
7. Set options + advanced (`mp`) settings
8. Start processing: "Watch for new runs" or manual tag selection
9. Perform geometry calibration and photon energy calibration
10. Make/load the pixel mask
11. Define a Trial and its run groups
12. Define a Dataset
13. Add and chain Indexing → Ensemble Refinement → Scaling → Merging tasks
