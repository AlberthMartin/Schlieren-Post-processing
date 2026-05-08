# Tuning and Known Caveats

## Main Parameters in `main_weighted.py`

These values control most of the current behavior.

### Rotation and Crop

```python
rotation_angle = 60
strip_half_height = 200
```

The processing assumes that after rotation the spray points mostly to the right
from the origin. The cropped strip is centered vertically on the selected origin.

### First Active Frame

```python
firstFrameNumber = vpf.findFirstFrame(rotated_video, threshold=10)
```

This uses whole-frame mean intensity, not motion or local nozzle activity.

### Mode Switches

```python
use_intensity_only = False
use_cumulative_as_mask = True
```

Current defaults mean:

- optical flow is still computed;
- high-motion pixels build a cumulative mask;
- intensity scoring is restricted to that cumulative mask;
- the direct optical-flow component is removed from the weighted product.

### Score Weights

Nominal values:

```python
w_intensity = 0.4
w_magnitude = 0.8
w_freehand = 0.1
w_cone = 0.6
intensity_gamma = 3.0
```

With `use_cumulative_as_mask = True`, these become:

```python
w_magnitude = 0
w_intensity = 2.0
```

Weights are normalized internally before being used as exponents in the weighted
geometric product.

### Optical Flow

```python
method = "Farneback"
mag_clip = 0.4
```

`runOpticalFlowCalculationWeighted` currently supports only Farneback in the
weighted path. Motion values at or above `0.4` are treated as maximum motion for
the normalized score.

### Cone Prior

```python
cone_angle_deg = 20
falloff_angle = 30
min_cone_length = 100
```

The cone prior is forward-only from the spray origin. The main cone gets full
weight, and the falloff band decreases linearly to zero.

Per frame, the cone is trimmed using previous penetration:

```python
cone_length = max(penetration[idx - 1] + 50, min_cone_length)
```

### Opening Detection

```python
roi_radius = 100
motion threshold = 0.5
```

The script starts saving masks and calculating geometry only after sustained
motion is detected near the spray origin.

### Mask Cleanup

Current default cleanup path:

```python
fill_holes_in_mask(threshold_mask)
keep_largest_blob(final_mask, horizontal_threshold=50, spray_origin=spray_origin)
```

Alternate clustering path when cumulative/intensity-only modes are disabled:

```python
create_cluster_mask(threshold_mask, cluster_distance=20, alpha=30)
```

## How to Tune Common Problems

### Spray is cropped out

Check:

- `rotation_angle`
- selected spray origin
- `strip_half_height`

After cropping, the origin y-coordinate is reset to the strip center. If the
click was inaccurate before cropping, the strip may be centered on the wrong
region.

### Chamber walls or corners enter the mask

Check:

- `createBackgroundMask(first_frame, threshold=20)`
- `background_mask` display window
- `cone_angle_deg` and `falloff_angle`
- `mask.png`

The background mask is a hard exclusion mask. If useful spray pixels are black
in that display, they cannot be recovered later.

### Spray is under-detected

Try:

- reduce `intensity_gamma`;
- increase `cone_angle_deg`;
- reduce `mag_clip`;
- disable `use_cumulative_as_mask` to allow direct optical-flow contribution;
- create or update `mask.png` so the freehand prior covers the true region.

### Spray is over-detected

Try:

- increase `intensity_gamma`;
- narrow `cone_angle_deg` or reduce `falloff_angle`;
- increase `mag_clip`;
- lower `horizontal_threshold` in `keep_largest_blob`;
- improve `mask.png` or remove it if it is too broad.

### Opening time triggers too early

`calculate_opening_point` currently checks for any sufficiently sustained motion
inside a 100-pixel circular ROI. Noise near the origin can trigger it. Tuning
options include reducing the ROI radius, increasing the motion threshold, or
requiring more pixels to pass the threshold.

## Current Caveats

- The pipeline is interactive and blocks on GUI windows.
- It is not configured from command-line arguments; most settings are hard-coded
  in `main_weighted.py`.
- `requirements.txt` is UTF-16 little-endian text. Some tooling may expect a
  UTF-8 requirements file.
- Output metrics are in pixels and frames. There is no physical calibration.
- `spray_origins.json` uses exact file paths as keys, so saved origins are not
  portable across machines or moved folders without editing the JSON.
- `mask.png` is global for the working directory, not per-video.
- Validation PNGs are written to the repository root, while CSV/MP4 outputs are
  written under `Results/`.
- If the diagnostic display loop is quit early, the overlay video is only written
  through the displayed frames.
- `geometry.calculate_boundary` has an empty-boundary return path with fewer
  values than the normal path. In normal operation, `main_weighted.py` avoids
  calling it until opening is detected, but an empty post-opening mask could hit
  that mismatch.
- `data_capture.py`, `histogram.py`, `extrapolation.py`, and much of
  `videoProcessingFunctions.py` contain useful experimental or alternative
  helpers that are not part of the current default main path.

