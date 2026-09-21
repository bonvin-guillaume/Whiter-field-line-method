# Auroral height from stereo all-sky images

`whiter_fieldline_method.ipynb` estimates the **peak emission height of aurora** from a pair of all-sky cameras (ASC) at Longyearbyen (LYR) and Ny-Ålesund (NYA). It follows the magnetic **field-line method** of Whiter et al. (2013): the same field line is projected into both images, brightness is sampled along altitude, and lines that have a similar shape at both sites are kept.

The star calibration for the ASC is done with [WISC - Widefield Star Calibrator](https://juha.no/aida/) and this notebook uses **WISC StarCal** HDF5 files for the camera pointing (azimuth/elevation grids in raw pixel coordinates).
## Method
<img width="2517" height="880" alt="Coordinate systems used - visual selection" src="https://github.com/user-attachments/assets/9bb638d3-fd4a-4f53-904f-4f66ed3d49e6" />

1. Load an image pair and the matching StarCal files.
2. Over a geographic box, start field lines at a reference altitude and trace them with IGRF via [PyGeopack](https://github.com/mattkjames7/PyGeopack).
3. Convert each `(lat, lon, alt)` sample to azimuth/elevation from each station, then to image pixels using the StarCal grids.
4. Sample image brightness along the line to get a brightness–altitude profile at LYR and NYA.
5. Keep only field lines that pass the stereo tests (similar peak altitudes, high correlation, similar intensity-weighted centroids, well-defined peaks).
6. Report peak altitudes (mean ± std / SEM) and save overlay and profile figures.


## Requirements

Python 3 with the packages in `requirements.txt`:

```
numpy
scipy
h5py
matplotlib
pillow
PyGeopack
```

Install with:

```bash
pip install -r requirements.txt
```

PyGeopack needs geomagnetic model files. On first use it typically downloads them; if that fails, set `GEOPACK_PATH` to a writable directory.

Run the notebook from the repository root so that `Data/` and `out/` resolve correctly.

## Input data

Place images and calibrations under `Data/`. Each `Station` needs:

| Input | Role |
| --- | --- |
| All-sky image (PNG/JPG) | LYR or NYA frame; RGB is converted to greyscale |
| StarCal HDF5 | `azimuth_deg`, `elevation_deg` corner grids of shape `(height+1, width+1)`, plus site attributes |

StarCal attributes used: `site_lat_deg`, `site_lon_deg`, `site_alt_m`, `image_width`, `image_height`. Image size must match the calibration. `flip_image_x` / `flip_image_y` are not supported.

The cell that constructs `lyr` and `nya` is where you select the event. Commented examples include BACC 2020-01-03, Sony 2020-01-03, and Sony 2021-12-15. The active setup is:

- Event time: `2020-01-03 08:57:50 UTC`
- Calibrations: `StarCal_BACC_LYR_2020.h5`, `StarCal_BACC_NYA_2020.h5`
- Images: currently both stations point at `RGB_product_BACC_NYA_20200103_085750.png` — swap in the matching LYR/NYA frames before a real run

Also set `event_dt` to the same instant as the images. That timestamp is used for the magnetic field model and for output file names.

## How to run

Open `whiter_fieldline_method.ipynb` and run all cells in order.

The tracing cell defines a geographic box and the altitude window. Current values:

```python
region = [
    {'lat1': 77.2, 'lat2': 78.4, 'lon1': 11.0, 'lon2': 16.6, 'label': 'R1'},
]

trace_and_filter_field_lines(
    lat_range, lon_range,
    alt_fl=165, step=0.1,
    upper=85, lower=85, interval=2,
    peak=10, corr_thr=0.7, edge=0, frac=0.9, width_limit=999, centroid_thr=20,
)
```

That traces field lines on a `0.1°` lat/lon grid, from **80 to 250 km** (`alt_fl ± 85 km`) every **2 km**.

Optional: after tracing, subset `selected_*` arrays if you only want to plot some of the accepted lines.

## Field-line acceptance tests

A line is kept when all of the following hold (see the markdown cell above the tracing call for the intended “strict” set):

| Parameter | Meaning |
| --- | --- |
| `peak` | Peak altitudes at LYR and NYA differ by less than this (km) |
| `corr_thr` | Pearson correlation of the two brightness profiles |
| `edge` | Reject peaks within this many km of the top or bottom of the altitude window |
| `centroid_thr` | Intensity-weighted altitude centroids differ by less than this (km) |
| `frac`, `width_limit` | Peak width at `frac` of maximum (e.g. 90%) must be ≤ `width_limit` km |
| peak shape | Peak brightness must stand out relative to the mean above/below the peak |

Looser numbers (`peak=10`, `corr_thr=0.7`, `edge=0`, `width_limit=999`) accept more lines; tighter numbers (`peak=5`, `corr_thr=0.8`, `edge=10`, `width_limit=40`) match the notebook’s documented example.

## Outputs

Figures go to `out/`:

## Notebook layout

| Section | What it does |
| --- | --- |
| Map `(lat, lon, alt)` to pixels | `Station` class: load StarCal + image, inverse az/el → pixel interpolator |
| Field-line tracing functions | GEO/ECEF geometry, PyGeopack traces, pixel mapping, brightness sampling, selection metrics |
| Trace fieldlines | Geographic box, `trace_and_filter_field_lines(...)` |
| Plot | Image overlays, single-station profiles, combined normalised profile |
