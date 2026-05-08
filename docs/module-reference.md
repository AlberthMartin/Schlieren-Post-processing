# Module Reference

This page summarizes the role of each non-legacy file in the repository.

## `main_weighted.py`

Current main entry point. It orchestrates the full interactive processing
pipeline:

1. choose input `.cine` files;
2. load video frames;
3. convert to 8-bit;
4. rotate video;
5. find first active frame;
6. select or reuse spray origin;
7. crop a strip around the spray;
8. apply CLAHE;
9. build manual exclusion, cone, intensity, and motion masks;
10. combine scoring masks into a weighted spray score;
11. threshold and clean the score into a final binary mask;
12. calculate penetration, cone angles, area, intensity, and timing metrics;
13. display diagnostics;
14. save overlay video, validation images, plots, and CSV.

Important hard-coded defaults live here, not in a config file.

## `functions_videos.py`

Video loading and simple playback utilities.

- `load_cine_video(cine_file_path)`: reads a Phantom `.cine` file with `pycine`,
  then loads frames in parallel into a `(frames, height, width)` `uint16` array.
- `read_frame(...)`: seeks to a frame offset and reads a `uint16` frame.
- `play_video_cv2(video, gain=1)`: displays a video with OpenCV after converting
  common data types to displayable 8-bit.
- `get_subfolder_names(parent_folder)`: small filesystem helper.

## `videoProcessingFunctions.py`

Frame and mask processing helpers.

Functions used directly by `main_weighted.py`:

- `createRotatedVideo(video, angle)`: rotates every frame with OpenCV affine
  warping.
- `findFirstFrame(video, threshold=10)`: returns the first frame whose mean
  intensity exceeds the threshold.
- `createVideoStrip(video, spray_origin, strip_half_height=200)`: crops a
  symmetric horizontal band around the origin row.
- `applyCLAHE(video, clipLimit=2.0, tileGridSize=(8,8))`: applies CLAHE to every
  frame in place.
- `calculate_opening_point(cone_mask_f, mag)`: detects sustained motion near the
  origin.
- `calculate_closing_point(close_point_distance, penetration, intensity_values,
  spray_area)`: estimates a closing frame from spray area and intensity
  heuristics.

Additional experimental helpers include thresholding, filtering, background
subtraction, SVD filtering, Chan-Vese segmentation, and temporal median
filtering.

## `opticalFlow.py`

Dense optical-flow calculation.

- `opticalFlowFarnebackCalculation(prev_frame, frame)`: wraps
  `cv2.calcOpticalFlowFarneback`.
- `runOpticalFlowCalculationWeighted(firstFrameNumber, video, method="Farneback")`:
  computes a per-frame flow magnitude array. This is what `main_weighted.py`
  uses.
- `runOpticalFlowCalculation(...)`: older visualization-oriented path that
  thresholds flow magnitude and returns masks/overlays.
- `opticalFlowDeepFlowCalculation(...)`: DeepFlow helper. The main weighted path
  currently supports only Farneback.

## `clustering.py`

Binary mask cleanup and shape construction.

- `create_cluster_mask(mask, cluster_distance=30, alpha=30)`: extracts contour
  points, clusters them with DBSCAN, builds an alpha/concave hull for each
  cluster, and fills the largest cluster.
- `fill_holes_in_mask(mask)`: flood-fills from boundaries and restores the
  original mask to fill internal holes.
- `keep_largest_blob(mask, horizontal_threshold=50, spray_origin=None)`: keeps
  the largest connected component plus nearby components that are horizontally
  close.
- `overlay_cluster_outline(frame, cluster_mask)`: draws mask outlines over a
  frame.
- `convex_hull_mask(mask)`, `alpha_shape(...)`, and `fast_alpha_shape(...)` are
  support/alternative shape helpers.

## `geometry.py`

Post-mask measurement logic.

- `calculate_boundary(boundary, nozzle_x, nozzle_y, angle_d, ax=None)`: converts
  boundary points into penetration, cone angle, regularized cone angle, closest
  boundary distance, and side-line angles.
- `calculate_video_intensity(video_strip, combined_masks)`: computes mean
  intensity inside the accepted mask for each frame, smooths it, and returns its
  derivative.

The code expects boundary points in `(row, col)` / `(y, x)` order, while OpenCV
contours are `(x, y)`. `main_weighted.py` performs that conversion before calling
`calculate_boundary`.

## `GUI_functions.py`

OpenCV/Tkinter interaction helpers.

- `set_spray_origin(...)`: reuses or collects the nozzle/spray origin and saves
  it in `spray_origins.json`.
- `get_freehand_exclusion_mask(video_strip, mask_path="mask.png")`: asks whether
  to reuse, edit, create, or skip the saved exclusion mask.
- `draw_freehand_mask(video_strip, ...)`: lets a user draw `mask.png` as a
  hard exclusion mask.
- `draw_and_compare_mask_frames(...)`: lets a user draw a validation mask on one
  frame and calculates Dice, precision, recall, and pixel accuracy.

## `histogram.py`

Histogram and frequency-domain visualization utilities. These are imported by
`main_weighted.py` but not called in the current default path.

Useful functions include:

- `compute_frame_histogram`
- `plot_histogram_change_heatmap`
- `analyze_histogram_statistics`
- `draw_single_frame_histogram`
- `plot_frame_histogram`
- `plot_fft_frequency_image`
- `render_histogram_to_array`
- `display_histogram_animation`

## `data_capture.py`

Alternative boundary-analysis implementation. It contains:

- `analyze_boundary(mask, angle_d, nozzle_point)`
- `regression_cone_angle(x_forward, y_forward)`

`main_weighted.py` imports this file with `from data_capture import *`, but the
current pipeline uses `geometry.calculate_boundary` instead.

## `extrapolation.py`

Experimental cone backfill/extrapolation utilities for extending detected masks
toward the origin. Not used in the current main pipeline.

## `displayGraph.py`

Standalone CSV plotting helper. It defines `main(csv_path, out_path=None)` for
plotting metrics from an output CSV.

## `mask.png`

Freehand binary exclusion mask. White pixels are blocked from
`main_weighted.py` scoring and final masks.

## `spray_origins.json`

Saved nozzle/origin coordinates keyed by exact input file path.

## `Legacy/`

Older scripts and experiments retained for reference. The main weighted pipeline
does not depend on them except for this import in `main_weighted.py`:

```python
from Legacy.std_functions3 import *
```

The imported names are not used by the current default code path.
