# Main Weighted Pipeline

`main_weighted.py` is a top-level script. It executes immediately when run and
does not define a `main()` function. The pipeline below describes the actual
runtime order in the file.

## 1. Imports and GUI Setup

The script imports helper functions from:

- `GUI_functions.py` for selecting/reusing the spray origin and manual
  validation drawing.
- `functions_videos.py` for `.cine` loading.
- `videoProcessingFunctions.py` for rotation, cropping, CLAHE, masks, and nozzle
  timing helpers.
- `opticalFlow.py` for Farneback optical-flow magnitude.
- `clustering.py` for mask cleanup and outline overlays.
- `geometry.py` for penetration, cone angle, and intensity metrics.

It creates and hides a Tkinter root window, then opens a file picker:

```python
all_files = filedialog.askopenfilenames(title="Select one or more files")
```

Each selected file is processed independently in a loop.

## 2. Load `.cine` Video

`load_cine_video(file)` reads Phantom `.cine` data using `pycine.file.read_header`.
It extracts width, height, and frame offsets, allocates a NumPy array with shape:

```text
(frame_count, height, width)
```

and reads frames in parallel with `ThreadPoolExecutor`. The loader returns
`uint16` video data for typical `.cine` inputs.

## 3. Convert Frames to 8-bit

The main script converts every frame to `uint8`:

- integer data is divided by `16`;
- floating data is assumed to be in `[0, 1]` and multiplied by `255`;
- boolean data is mapped to `0` or `255`.

After the loop, the entire video is cast to `np.uint8`.

## 4. Rotate and Find First Active Frame

The video is rotated by:

```python
rotation_angle = 60
```

using `vpf.createRotatedVideo(video, rotation_angle)`.

Then `vpf.findFirstFrame(rotated_video, threshold=10)` returns the first frame
whose mean intensity is greater than `10`. If no frame passes that threshold, it
returns frame `0`.

## 5. Select or Reuse Spray Origin

`set_spray_origin(file, rotated_video, firstFrameNumber, nframes, height)` reads
`spray_origins.json`.

If the exact file path is already present, the saved `(x, y)` origin is reused.
Otherwise, the script opens an OpenCV window titled:

```text
Set Spray Origin - Click on the nozzle
```

The user clicks the nozzle. Left and right arrow keys move between frames. If the
window is quit before a point is selected, the fallback origin is:

```python
(1, height // 2)
```

The selected origin is saved back to `spray_origins.json`.

## 6. Crop a Horizontal Strip

The current script always enters this block:

```python
if True:
    video_strip = vpf.createVideoStrip(rotated_video, spray_origin, strip_half_height=200)
```

`createVideoStrip` extracts a symmetric horizontal band around the origin row.
The effective half-height may be smaller than `200` near image edges so the crop
stays inside the image.

After cropping, the spray origin is changed to:

```python
spray_origin = (spray_origin[0], height // 2)
```

This keeps the original x-coordinate and recenters the origin vertically in the
cropped strip.

## 7. Preserve Raw Strip and Apply CLAHE

The script makes a copy:

```python
video_strip2 = video_strip.copy()
```

`video_strip2` is used later for manual validation display.

Then `vpf.applyCLAHE(video_strip)` applies contrast-limited adaptive histogram
equalization to every frame in place.

## 8. Create Background Mask

`vpf.createBackgroundMask(first_frame, threshold=20)` creates a binary mask from
the first active frame:

```text
first_frame > 20 => 255
otherwise        => 0
```

The script displays this mask in an OpenCV window and waits for a key press. In
later thresholding, pixels where this background mask is `0` are excluded.

## 9. Compute Optical-flow Magnitude

The pipeline has two control switches:

```python
use_intensity_only = False
use_cumulative_as_mask = True
```

With the current defaults, the script initially computes optical flow but later
sets `w_magnitude = 0` and uses the cumulative motion mask to restrict the
intensity score.

When optical flow is active, the code calls:

```python
of.runOpticalFlowCalculationWeighted(firstFrameNumber, video_strip, method="Farneback")
```

That function computes dense Farneback flow between consecutive frames and stores
only the magnitude array. Work is split across threads. Frames before
`firstFrameNumber` remain zero.

If `use_intensity_only` is set to `True`, the code skips real optical flow and
uses an all-ones magnitude array.

## 10. Build Static Priors

The script prepares a cone prior based on the spray origin:

- `cone_angle_deg = 20`
- `falloff_angle = 30`
- `min_cone_length = 100`
- cone pixels ahead of the origin and inside `20` degrees receive weight `1.0`;
- pixels in the next `30` degrees fade linearly to `0`.

The cone is recomputed per frame by trimming the full cone to the previous
frame's penetration plus `50` pixels, with a minimum length.

The script also loads `mask.png` as a freehand mask. If missing, it prints a
warning and uses an empty mask. During score combination, an empty freehand mask
is treated as all ones so it does not suppress the result.

## 11. Initialize Metric Arrays

The script allocates per-frame arrays for:

- `combined_masks`: threshold masks saved only after nozzle opening is detected.
- `final_cluster_masks`: cleaned final masks.
- `intensity_scores`, `mag_scores`, `cumulative_masks`, `cone_masks`: diagnostic
  arrays.
- `penetration`, `cone_angle`, `cone_angle_reg`, `close_point_distance`.
- fitted side-line diagnostic angles.
- `spray_area` and `spray_volume`.

It also initializes a 100-pixel-radius circular ROI around the spray origin for
detecting nozzle opening from motion.

## 12. Per-frame Scoring Loop

For every frame, the script calculates four normalized components.

### Intensity Score

The frame is converted to float. The script computes the 1st and 99th
percentiles, normalizes the image to `[0, 1]`, inverts it so darker pixels score
higher, and applies:

```python
intensity_gamma = 3.0
```

With `use_cumulative_as_mask = True`, only pixels already in the cumulative
motion mask may keep a nonzero intensity score.

### Motion Score

The optical-flow magnitude is clipped at:

```python
mag_clip = 0.4
```

and normalized to `[0, 1]`. Pixels above `0.99` in normalized magnitude are added
to the cumulative mask.

### Nozzle Opening Detection

Starting at `firstFrameNumber`, `vpf.calculate_opening_point(circle_mask, mag)`
checks whether any pixels inside the circular ROI have enough sustained motion.
The first frame that passes latches:

```python
write_masks_started = True
nozzle_opening_time = idx
```

Before this point, `combined_masks[idx]` is forced to zero and geometry metrics
are set to zero.

### Weighted Combination

The nominal weights are:

```python
w_intensity = 0.4
w_magnitude = 0.8
w_freehand = 0.1
w_cone = 0.6
```

With the current default `use_cumulative_as_mask = True`, the script changes
them to:

```python
w_magnitude = 0
w_intensity = 2.0
```

Weights are normalized by their sum. The components are combined as a weighted
geometric product:

```python
combined_score =
    (intensity + eps) ** norm_intensity *
    (motion + eps) ** norm_magnitude *
    (freehand + eps) ** norm_freehand *
    (cone + eps) ** norm_cone
```

The result is divided by its frame maximum when possible.

## 13. Threshold and Clean the Mask

With `use_cumulative_as_mask = True` or `use_intensity_only = True`, the script
uses Otsu thresholding on the combined score. Otherwise, it thresholds at
`0.8 * peak`.

Pixels outside the background mask are removed:

```python
threshold_mask[background_mask == 0] = 0
```

For the current cumulative-mask path, cleanup is:

1. `fill_holes_in_mask(threshold_mask)`
2. `keep_largest_blob(..., horizontal_threshold=50, spray_origin=spray_origin)`

If cumulative-mask and intensity-only modes are disabled, the alternate cleanup
path uses `create_cluster_mask(threshold_mask, cluster_distance=20, alpha=30)`.

## 14. Boundary Geometry

The final mask is converted to OpenCV contours. Contours are converted from
OpenCV `(x, y)` format to `(row, col)` / `(y, x)` arrays for
`geometry.calculate_boundary`.

Once nozzle opening has been detected, `calculate_boundary` returns:

- penetration: maximum distance from the nozzle to any boundary point.
- cone angle: included angle between centroid vectors for upper/lower boundary
  points.
- regularized cone angle: angle between fitted upper/lower boundary lines.
- boundary points.
- close point distance: minimum boundary distance from the nozzle.
- side-line angles for overlay diagnostics.

Cone angle calculations use boundary points whose distance from the nozzle is
between `10%` and `60%` of the penetration distance.

## 15. Area, Volume, and Intensity Metrics

For each frame:

- `spray_area` is the count of nonzero pixels in the final mask.
- `spray_volume` is a rough estimate:

```python
(pi / 4) * (spray_area ** 2) * penetration
```

After the frame loop, `calculate_video_intensity(video_strip, combined_masks)`
computes mean intensity inside the saved combined masks, smooths it with a
5-frame moving average, and takes the frame-to-frame derivative.

`vpf.calculate_closing_point(...)` estimates nozzle closing from spray area and
intensity heuristics. The area-derived candidate has the highest weight.

## 16. Display and Save Overlay Video

The script creates:

```text
Results/<input_basename>_overlay.mp4
```

For every frame, it displays a 3x3 diagnostic grid containing:

- original processed frame
- combined weighted mask
- clustered/final mask
- intensity score
- optical-flow magnitude
- cumulative mask
- cone mask
- freehand mask
- final overlay

The overlay video itself contains the processed strip with the final mask
outline, spray origin, penetration line, penetration-tip marker, and cone-angle
diagnostic side lines.

OpenCV controls:

- `q`: quit display loop
- `p`: pause until another key press

## 17. Manual Validation

After the overlay loop, the script opens a validation window on:

```python
compare_frame_idx = min(200, nframes - 1)
```

The user draws a ground-truth mask. The helper computes:

- selected frame index
- intersection
- ground-truth area
- predicted area
- Dice score
- precision
- recall
- pixel accuracy

It writes validation images to the current working directory, not `Results/`:

```text
<input_basename>_gt_mask_frame_<frame>.png
<input_basename>_comparison_frame_<frame>.png
```

## 18. Plot and Export CSV

The script plots a 2x4 Matplotlib figure of:

- penetration
- cone angle
- regularized cone angle
- spray volume estimate
- close point distance
- spray area
- mean intensity
- intensity derivative

Finally, it writes:

```text
Results/<input_basename>_spray_metrics.csv
```

with one row per frame.

