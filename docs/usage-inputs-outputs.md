# Inputs, Interaction, and Outputs

## Required Runtime Environment

The main pipeline is interactive. It needs:

- Python dependencies from `requirements.txt`.
- GUI support for Tkinter file dialogs.
- GUI support for OpenCV windows.
- `.cine` input videos readable by `pycine`.

Run:

```bash
python main_weighted.py
```

Then select one or more `.cine` files in the file picker.

## Input Files

### `.cine` Videos

Selected videos are loaded by `functions_videos.load_cine_video`. The loader
uses the frame offsets from the `.cine` header and returns a NumPy array in
`(frames, height, width)` order.

### `spray_origins.json`

This file stores per-video nozzle/spray-origin coordinates. Keys are exact file
paths as passed by the file picker. Values are two-number arrays:

```json
{
  "C:/SchlierenVids/NewH2/T85_Schlieren_2.cine": [36, 390]
}
```

Coordinates are in image order:

```text
(x, y)
```

If a selected file path is not present, the script asks the user to click the
origin and then writes the result to this file.

Because the key is the full path, moving videos to a new directory or selecting
the same video via a different path will make the script ask for the origin
again.

### `mask.png`

`main_weighted.py` tries to load `mask.png` from the current working directory.
It is treated as a single-channel binary freehand mask and resized to the current
strip dimensions if necessary.

If `mask.png` is missing, the pipeline continues. An empty freehand mask is
treated as neutral during combination.

`GUI_functions.draw_freehand_mask(video_strip)` can create `mask.png`, but the
current main script does not call that helper automatically.

## Runtime Windows

### File Picker

Tkinter asks for one or more input files.

### Spray Origin Window

Only shown when the video path is not already in `spray_origins.json`.

Controls:

- left click: select nozzle/origin
- left arrow: previous frame
- right arrow: next frame
- `q`: quit selection and use fallback origin

### Background Mask Window

Shows the threshold-derived background mask and waits for any key.

### Diagnostic Results Window

Shows a 3x3 grid for each frame.

Controls:

- `q`: quit display loop
- `p`: pause

Quitting this display loop early also stops writing the overlay video at that
frame, because the display loop and video writer are the same loop.

### Validation Window

Shows a representative frame with the predicted outline. The user can draw a
ground-truth mask for comparison.

Controls:

- left drag: draw ground-truth mask
- left arrow: previous frame
- right arrow: next frame
- `r`: reset current frame's ground-truth mask
- `q`: finish validation

## Output Directory

For each selected input file, the script creates `Results/` in the repository
root if needed.

The main outputs are:

```text
Results/<input_basename>_spray_metrics.csv
Results/<input_basename>_overlay.mp4
```

Validation outputs are written to the repository root:

```text
<input_basename>_gt_mask_frame_<frame>.png
<input_basename>_comparison_frame_<frame>.png
```

## CSV Columns

The exported CSV has one row per frame and these columns:

- `Frame`
- `Penetration (pixels)`
- `Cone Angle (degrees)`
- `Regularized Cone Angle (degrees)`
- `Close Point Distance (pixels)`
- `Spray Area (pixels^2)`
- `Mean Intensity`
- `Intensity Derivative`
- `Spray Volume (cubic pixels)`
- `Nozzle Opening Time (frames)`
- `Nozzle Closing Time (frames)`

`Nozzle Opening Time` and `Nozzle Closing Time` are repeated on every row as
video-level values.

## Output Units

The script currently reports image-space units:

- distances are pixels;
- areas are pixel counts;
- angles are degrees;
- frame timing is frame index, not seconds;
- volume is a rough cubic-pixel estimate.

No calibration from pixels to physical units or frames to time is applied in the
current pipeline.

