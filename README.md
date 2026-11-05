# motion-volume-3d
Converts video motion to 3D space where time is the third dimension, with background suppression  

Turn a video into an interactive 3D **space-time sculpture** of everything that moves in it.

Each frame is background-subtracted (OpenCV MOG2 / KNN), the foreground masks are stacked along time into a `(T, H, W)` volume, and **scikit-image Marching Cubes** extracts the iso-surface. Each moving object becomes a tube through time, coloured with its real colours from the video, and is published as a lightweight interactive **Plotly** page next to the looping source clip.

```
video ──► background subtraction ──► mask volume (T,H,W) ──► reduce resolution ──► Marching Cubes
                                                                                      │
                     side-by-side HTML (video | 3D) ◄── Plotly Mesh3d ◄── vertex colours from frames
```

## Features

- **In-memory background subtraction**: MOG2 or KNN, optional Gaussian blur and horizontal flip, no intermediate video files.
- **Compact meshes**: the mask volume is area-averaged in H×W *before* Marching Cubes, tiny speckle blobs are removed, and the reduction factor increases automatically until the mesh fits a face budget.
- **True-colour surfaces**: every vertex is coloured from the original frame at its `(t, y, x)` position, sampled slightly *inside* the object along the surface normal to avoid picking up road or background pixels.
- **Side-by-side viewer**: one self-contained HTML page with the looping clip on the left and the 3D mesh on the right, with a frame counter and a **time-cursor plane** that follows the video.
- **Static snapshot**: a three-panel PNG (original frame | foreground mask | 3D projection) for reports and READMEs.

## Repository layout

```
.
├── code/
│   └── BackgroundFlow_Skimage_To_Plotly.ipynb
├── html/                     # generated
│   ├── background_3d.html               # standalone Plotly 3D view
│   ├── background_side_by_side.html     # video | 3D page
│   └── background_clip.mp4             # processed clip (H.264)
└── images/                   # generated
    ├── background_3d.png                # optional, requires kaleido
    └── background_combined.png          # 3-panel snapshot
```

The notebook lives in `code/` and writes its outputs to `../html` and `../images`.

## Installation

```bash
pip install numpy scipy scikit-image opencv-contrib-python plotly matplotlib kaleido imageio-ffmpeg
```

- `imageio-ffmpeg` provides an ffmpeg binary for H.264 encoding, so you don't need it if `ffmpeg` is already on your `PATH`.
- `kaleido` is only needed for the static `background_3d.png`.

## Usage

1. Set `VIDEOFILE` in the configuration cell to your video.
2. Run all cells.
3. Open `html/background_side_by_side.html` in a browser.

## Configuration

All parameters are in the first code cell.

| Parameter | Default | Description |
|---|---|---|
| `VIDEOFILE` | — | Input video path |
| `ALGO` | `'MOG2'` | Background subtractor: `'MOG2'` or `'KNN'` |
| `MAX_FRAMES` | `80` | Number of frames to process (`None` = whole video) |
| `BLUR_K` | `5` | Gaussian blur kernel on the mask (odd, `0` = off) |
| `FLIP_H` | `True` | Flip frames horizontally |
| `ISO_LEVEL` | `192` | Marching Cubes threshold on the 0–255 mask |
| `STEP_SIZE` | `1` | Marching Cubes step size |
| `DOWNSCALE` | `4` | Initial H×W reduction factor before Marching Cubes |
| `MAX_FACES` | `400_000` | Face budget; the factor grows until the mesh fits |
| `MIN_BLOB_VOX` | `8` | Remove connected blobs smaller than this (voxels, after downscale) |
| `COLOR_MODE` | `'video'` | `'video'` = colours from the frames, `'frame'` = colourscale by frame number |
| `COLOR_DEPTH` | `1.0` | How far inside the surface to sample colours (downscaled voxels) |

The side-by-side page has its own options at the top of section 4b:

| Parameter | Default | Description |
|---|---|---|
| `CLIP_WIDTH` | `960` | Width of the embedded clip (px) |
| `EMBED_VIDEO` | `True` | Embed the clip as base64 (single portable file), or reference `background_clip.mp4` |
| `SYNC_CURSOR` | `True` | Show the time-cursor plane in the 3D view |

## Design notes

### Why reduce resolution before meshing rather than decimating afterwards

At full resolution, 80 frames of 1080×1920 produced **10.4 M vertices / 20.7 M faces**, which made an HTML file of several hundred MB. Two remedies were benchmarked on a synthetic volume of the same shape:

| Approach | Faces | Time | Mean / p95 error (px) |
|---|---|---|---|
| Full-resolution Marching Cubes | 9.0 M | 6.7 s | reference |
| Full-res, then vertex-clustering decimation (cell 4×4 px) | 783 k | 6.7 s + decimation | 1.29 / 2.12 |
| Area-averaged ×4, then Marching Cubes | 809 k | 0.6 s | 1.53 / 2.47 |
| Full-res, then decimation (cell 2×8×8) | 198 k | 6.7 s + decimation | 2.57 / 4.30 |
| Area-averaged ×8, then Marching Cubes | 282 k | 0.2 s | 2.90 / 4.95 |

- **Same accuracy:** both approaches give similar error for a similar number of faces.
- **Faster and lighter:** reducing first is 10–30× faster and never builds the multi-GB intermediate mesh.
- **Cleaner mesh:** it produces a clean, manifold mesh, whereas vertex clustering leaves folded triangles that shade badly.
- **Nothing visible is lost:** with an equal aspect ratio, the 1920 px width is displayed at the same scale as 80 frames, so sub-4-px detail is invisible anyway.

`marching_cubes(..., spacing=(1, f, f))` keeps the coordinates in original pixel units.

### Smaller HTML

- Arrays are cast to `float32` / `int32`, which plotly ≥ 6 serialises as base64 typed arrays.
- plotly.js is loaded from the CDN (`include_plotlyjs='cdn'`), saving about 4.6 MB per file.
- Vertex colours are `#rrggbb` strings, about 10 bytes per vertex.

### Colour sampling

The iso-surface lies on the mask boundary, where pixels are half object, half background. For each vertex the code:

1. steps `COLOR_DEPTH` voxels along the normal in both directions;
2. reads the reduced mask at both points;
3. samples the colour on the side with the higher mask value, i.e. the foreground.

This works whichever way the normals point. Frames are area-averaged by the same factor as the mask, so each vertex takes the mean colour of the cell it represents. Sampling uses trilinear interpolation (`scipy.ndimage.map_coordinates`).

## Tips

- **Smaller files:** lower `MAX_FACES`. For more detail, raise it or set `DOWNSCALE` lower.
- **Speckle:** raise `MIN_BLOB_VOX` or `BLUR_K`.
- **Shadows:** MOG2 marks shadows as 127. Keep `ISO_LEVEL` above 127 to exclude them from the surface.
- **Dark patches on the surface:** increase `COLOR_DEPTH` to 1.5–2.
- **Offline viewing:** the pages load plotly.js from the CDN. For offline use, change `include_plotlyjs='cdn'` to `True` (adds about 4.6 MB).
- **Sluggish rotation in the side-by-side page:** set `SYNC_CURSOR = False`.

## License

Add your license of choice (e.g. MIT) in a `LICENSE` file.
