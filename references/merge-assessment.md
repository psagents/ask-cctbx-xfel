# Assessing a cctbx.xfel merge and choosing d_min

How to read the statistics in a `*_main.log` from `cctbx.xfel.merge`, and when to
move the resolution cutoff. The numbers are read by `ask-lute/cctbx/parse_merge_log.py`
(field names below are its JSON keys); this file is the judgment.

Sources: the LCLS comp-xtal handbook (https://lcls-mlcv.github.io/comp-xtal/,
"look at obs_multi > 10, CC1/2 > 0.3, and other metrics to decide resolution and if
any pathology"), and the d_min rule `psagents/cctbx-sfx-workflow` ran on mfx101624926
(`dmin_advisor.py`, completeness ≥ 99 %). These are **starting points**: the
experiment team owns the final cutoff.

## Where each number comes from

| Key | Log table | Meaning |
|---|---|---|
| `shells[].completeness_pct` | Intensity Statistics (all accepted) | measured unique reflections / possible in that shell |
| `shells[].obs_multi` | same, `<obs multi>` | mean observations per *observed* reflection |
| `shells[].asu_multi` | same, `<asu multi>` | mean observations per *possible* reflection (≤ obs_multi) |
| `shells[].i_over_sigma` | same, merged `<I/sig(I)>` | signal in the merged data |
| `shells[].cc_half_pct` | Table of Scaling Results, `CC int` | CC½: odd/even half-dataset correlation |
| `shells[].r_split_pct` | same, `R split` | half-dataset disagreement; rises at high resolution |
| `image_shells[].accepted_pct` | Image Statistics | % of lattices with reflections out to that shell |
| `overall.cc_half_pct` | `C.C. int is X% on N indices` | CC½ over the whole resolution range |

## Choosing d_min

The cutoff is the **highest-resolution shell that still carries signal**, judged per
shell — never from the overall row.

A shell supports the cutoff when **all** of:

- `cc_half_pct > 30`
- `obs_multi >= 10`
- `completeness_pct >= 99`

Recommend tightening d_min to the `d_min` of the highest-resolution shell meeting
all three, **only if it is below the current d_min**. Otherwise:

- the outermost shells fail but inner ones pass → the current cutoff is too
  ambitious *for the data so far*; more runs usually fix multiplicity and
  completeness before CC½. Report it, do not loosen d_min unprompted.
- no shell passes → not enough data to judge a cutoff yet. Say so.

A recommendation is advisory. The operator approves a d_min change.

## Rules that outrank a nice number

- **Compare two merges only at equal d_min.** Overall completeness, multiplicity and
  CC½ change with the cutoff on identical data: tightening it "improves" them. That is
  a re-cut, not better data.
- **Overall CC½ is dominated by the strong low-resolution shells.** A high overall
  value says nothing about the outer shell; read `shells`.
- **Completeness well below 100 % in the *low*-resolution shells** is a beamstop /
  mask / saturation question, not a cutoff question.
- **`image_shells[0].accepted_pct` well below 100 %** means lattices are being
  rejected outright (unit-cell filter, significance filter) before resolution
  matters — look at filtering before resolution.
- **`missing_sections` non-empty** on a non-empty log means the parser could not
  find a table. That is a log-format change or an unfinished merge, not bad data.
