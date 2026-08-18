# Patch notes (unofficial, applied outside the original repo)

This is a patched copy of `eshasadia/CORE@main`, fixing bugs found via static
analysis (`ast.parse`, `pyflakes`) plus manual review, and making
`notebooks/1-WSI_Registration.ipynb` and `notebooks/2-WSI_Registration.ipynb`
runnable standalone in Google Colab. Nothing else in the repo was modified —
all `.py` files outside `core/utils/util.py` were syntax-clean already.

## `core/utils/util.py`

1. **`create_nonrigid_mha`** called `util.combine_deformation(...)` — a
   self-referential lookup of the module from inside itself, which raised
   `NameError: name 'util' is not defined`. Changed to call
   `combine_deformation(...)` directly (it's defined later in the same file).

2. **`create_deformation_field`** required a 5th positional parameter named
   `util` that was never used inside the function body, and that neither
   notebook ever passed at its call sites — every call raised
   `TypeError: missing required positional argument: 'util'`. Removed the
   dead parameter (verified: nothing in the repo relied on the old 6-arg
   signature).

3. **`create_pyramid`** called `resample_tensor_to_size(...)` and
   `gaussian_smoothing(...)`, neither of which was defined or imported in
   this file. `gaussian_smoothing` exists in `core/registration/nonrigid.py`;
   `resample_tensor_to_size` did not exist anywhere in the repo at all — the
   nearest equivalent is `scale_tensor_to_dimensions` in the same file
   (used by the working, non-broken `create_multiscale_representation`,
   which does the same job). Imported both and aliased
   `scale_tensor_to_dimensions` as `resample_tensor_to_size` so the call
   site didn't need to change; adjusted the keyword argument name
   (`mode=` -> `interpolation_method=`) to match the real function's
   signature.

4. **Duplicate `compute_center_of_mass`** — two functions shared this name:
   one operating on a binary mask (`cv2.moments`, ~line 291) and one on a
   point-set array (`np.mean`, ~line 775, defined later so it silently
   shadowed the first). Renamed the point-set version to
   `compute_points_center_of_mass`. (Nothing in the current pipeline called
   either via `util.compute_center_of_mass` externally, but this was a
   landmine for anyone who did.)

5. Removed an unused `from tiatoolbox.utils.metrics import dice` import
   that was silently shadowed by a same-named function defined later in
   the file — kept the local `dice()` definition since that's what
   actually runs.

All four changed functions (`combine_deformation`, `create_deformation_field`,
`create_pyramid`, `create_nonrigid_mha`) were smoke-tested directly with
synthetic NumPy/Torch arrays after patching and now run without error.

## `notebooks/1-WSI_Registration.ipynb`

- Fixed a cell that would not even parse: `if slide=='mif'` was missing its
  colon, `slide` was never defined anywhere, and the `else` branch referenced
  `taget` (typo for `target`, which was itself undefined). Replaced with a
  working `if/else` and an explicit `slide = 'brightfield'` default.
- Replaced the fragile relative `sys.path` cell with a Colab-standalone setup
  cell (see below).
- Added a warning above the `tiatoolbox visualize` / `bokeh serve` cell
  explaining it needs a tunnel (e.g. `pyngrok`) to work in hosted Colab.

## `notebooks/2-WSI_Registration.ipynb`

- Fixed the final displacement-field cell, which referenced `w_x`/`w_y` —
  never defined anywhere in the notebook. Replaced with `r_x`/`r_y`, the
  rigid displacement field actually computed earlier in the same section.
- Replaced hardcoded personal absolute paths
  (`/home/u5552013/Nextcloud/HYRECO/...`) for the nuclei CSVs with
  Colab-friendly placeholders driven by the new setup cell's
  `FIXED_POINTS_PATH`/`MOVING_POINTS_PATH` variables.
- Replaced the `sys.path` cell with the same Colab-standalone setup cell.
- Added warnings above both `tiatoolbox visualize` / `bokeh serve` cells.

## The Colab-standalone setup cell (both notebooks)

Replaces the old two-line "add project root to PYTHONPATH" cell. It:
- Detects whether it's running in Colab (`"google.colab" in sys.modules`).
- If so, clones the repo into `/content/CORE` (if not already present) and
  `cd`s into it.
- Installs **one consistent set of pinned dependency versions** — the
  original repo ships two files that actively disagree with each other
  (`environment.yml` pins `SimpleITK==2.4.0` and `tiatoolbox=1.6`;
  `requirements.txt` requires `SimpleITK<2.4` and `tiatoolbox==1.4.1`, and
  `torch==2.1.2` vs `pytorch=2.5`). This cell does not `pip install -r`
  either file; it installs its own compatible set instead.
- Sets `sys.path` to the repo root either way (Colab or local Jupyter).
- Sets `VISION_AGENT_API_KEY` from a Colab Secret if present, otherwise
  prompts interactively (only needed for the prompt-based tissue-masking
  path — UNet-based masking doesn't require it).
- Exposes `SOURCE_WSI_PATH`, `TARGET_WSI_PATH`, `FIXED_POINTS_PATH`,
  `MOVING_POINTS_PATH` as plain variables to edit directly in the notebook,
  since editing `core/config.py` on every Colab session isn't persistent.

## Known remaining limitations (not fixed, by design)

- `environment.yml` and `requirements.txt` still disagree with each other if
  you try to install from them directly — use the notebooks' setup cell, or
  pick one file and edit it, if you need a local conda/venv install.
- The `tiatoolbox visualize` / `bokeh serve` interactive viewer cells still
  won't work out-of-the-box in hosted Colab (needs port tunneling).
- Full end-to-end runs weren't tested against real whole-slide images (none
  were available); the fixes were verified by direct unit-level testing of
  the changed functions with synthetic data, plus static analysis
  (`ast.parse`/`pyflakes`) across the whole repo confirming no remaining
  undefined-name or syntax errors.
