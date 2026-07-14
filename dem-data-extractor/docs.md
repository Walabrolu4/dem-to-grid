# extract_dem.py — Technical Documentation

This document explains what `extract_dem.py` does internally, function by function, for anyone maintaining or extending it. For install/usage instructions, see [README.md](README.md).

## Purpose

Takes a DEM GeoTIFF and extracts it into a form an external runtime (e.g. Unity, a game engine, a custom viewer) can load quickly without linking GDAL/rasterio:

- a flat, headerless binary of every pixel's elevation (`dem.bin`)
- a small JSON sidecar (`dem_metadata.json`) with everything needed to interpret that binary correctly (dimensions, byte layout, elevation range, nodata, geographic bounds, CRS, pixel size)
- optionally, a human-viewable grayscale PNG (`dem-preview.png`) and/or a CSV dump, neither of which the runtime is meant to actually consume

## Pipeline overview

`main()` runs these steps in order:

1. **Read** band 1 both raw and masked, plus geospatial metadata (`read_dem`)
2. **Build metadata** — elevation range computed only from valid pixels (`build_metadata`)
3. **Write** the binary (`write_binary`) and metadata (`write_metadata`) — always
4. **Optionally write** a CSV (`write_csv`) and/or a preview PNG (`write_preview`)

## Function reference

### `read_dem(path) -> (raw, masked, xres, yres, bounds, crs, nodata, width, height)`

Opens the GeoTIFF with `rasterio` and reads band 1 **twice**, deliberately:

- `raw = dataset.read(1)` — the plain array, exactly as stored in the source file, including whatever nodata sentinel or `NaN` values it contains. This is what gets written to `dem.bin` — the runtime gets the *real* pixel values, not a masked/altered copy.
- `masked = dataset.read(1, masked=True)`, further passed through `np.ma.masked_invalid` — masks both the raster's declared nodata value and any stray `NaN`/`Inf` (some float DEMs use those instead of a nodata tag). This copy is used **only** for computing `minElevation`/`maxElevation`, so sentinel values like `-9999` don't corrupt the reported range.

The plain (non-masked) `dataset.read(1)` call triggers a spurious `DeprecationWarning` on newer NumPy versions — it originates inside rasterio's own code, not this script, and doesn't affect the returned data, so it's suppressed narrowly around just that call (`warnings.catch_warnings()` + `filterwarnings("ignore", category=DeprecationWarning)`) rather than globally.

Also returns `xres, yres` (pixel size in CRS units, from `dataset.res`), `bounds`, `crs`, `nodata` (the raster's declared nodata value, or `None`), and `width, height`.

### `build_metadata(masked, xres, yres, bounds, crs, nodata, width, height) -> dict`

Raises `ValueError` if every pixel is invalid (`masked.count() == 0`) — there'd be no meaningful elevation range to report. Otherwise computes `minElevation`/`maxElevation` via `np.ma.min`/`np.ma.max` over the masked array (excluding nodata/NaN/Inf), and assembles the full metadata dict:

| Field | Meaning |
|---|---|
| `width`, `height` | Pixel dimensions |
| `minElevation`, `maxElevation` | Computed over valid pixels only |
| `noDataValue` | The raster's declared nodata value, or `null` if none is set (the binary may still contain raw `NaN`s in that case) |
| `bounds` | `{left, right, bottom, top}` in the raster's CRS units |
| `crs` | CRS as a string (e.g. `"EPSG:4326"`), or `null` |
| `epsg` | EPSG code as an integer, or `null` if the CRS has none |
| `pixelSize` | `{x, y}` pixel size in CRS units (`dataset.res`) |
| `dataType` | Always `"float32"` — the binary's element type regardless of the source raster's own dtype (e.g. int16 DEMs are upcast) |
| `byteOrder` | Always `"little"` |
| `pixelOrder` | Documents the binary's layout: row-major, row 0 = the top/north row |

### `write_binary(raw, path)`

```python
raw.astype("<f4").tofile(path)
```

Casts to little-endian float32 (`<f4`) regardless of the source dtype, and writes it with no header — just `width * height * 4` bytes, row-major in the array's existing order (which matches the source raster: row 0 is the top/north row). `numpy.tofile()` writes the array flattened in C order (row-major), which is what `pixelOrder` in the metadata documents.

### `write_metadata(metadata, path)`

Plain `json.dump(metadata, f, indent=2)`.

### `write_csv(raw, path)`

One row per DEM row, comma-separated, raw pixel values (not the masked/statistics view) — same nodata sentinel semantics as `dem.bin`. Meant for prototyping/inspection only; for any DEM of meaningful size this is much larger and slower to parse than the binary.

### `write_preview(masked, min_elevation, max_elevation, path, max_dim)`

Normalizes the **masked** array to `[0, 1]` via min-max stretch (`(masked - min_elevation) / (max_elevation - min_elevation)`, or all-zero if the range is degenerate), then renders it grayscale with nodata pixels transparent (`cmap.set_bad(alpha=0.0)`).

- If `max_dim` is `None`, or the DEM's longer side is already `<= max_dim`, saves directly via `matplotlib.pyplot.imsave` at native resolution.
- If downsampling is needed, builds the full-resolution RGBA array by hand (`cmap(...)`) and resizes it with `PIL.Image.resize(..., Image.BILINEAR)` — `matplotlib.imsave` itself has no resizing option, so this path exists specifically to keep preview file size/dimensions bounded for very large DEMs.

Uses the `Agg` backend (`matplotlib.use("Agg")`) since there's no display under WSL; both `matplotlib` and `Pillow` imports are deferred into this function so they're only required when `--preview` is actually used.

### `validate_args(args)`

Only checks `--preview-max-dim` is a positive integer when given. (There's no analogous check needed for `--csv`/`--preview` themselves since they're boolean flags.)

## CLI reference

| Flag | Required | Description |
|---|---|---|
| `input` | yes | Path to the input DEM `.tif` |
| `--output-dir` | no | Directory to write outputs into (default: current directory; created if missing) |
| `--bin-name` | no | Binary output filename (default `dem.bin`) |
| `--metadata-name` | no | Metadata output filename (default `dem_metadata.json`) |
| `--preview` | no | Also save a grayscale preview PNG |
| `--preview-name` | no | Preview filename (default `dem-preview.png`) |
| `--preview-max-dim` | no | Downsample the preview so its longer side is at most this many pixels |
| `--csv` | no | Also save the raw elevation grid as CSV |
| `--csv-name` | no | CSV filename (default `dem.csv`) |

All validation happens up front in `validate_args` before any raster I/O.

## Error handling

`main()` wraps the whole pipeline in a single `try/except Exception`, printing `Error: <message>` to stderr and exiting with status 1 on any failure (missing file, invalid args, all-nodata DEM, etc.) — there's no partial/silent output on error.

## Known limitations

- `dem.bin` preserves whatever nodata sentinel the source raster uses (or raw `NaN`, if that's how the DEM encodes missing data) rather than normalizing to one convention — a consumer must check `noDataValue` in the metadata (and separately handle `NaN`, if present, since `noDataValue` will be `null` in that case) rather than assuming a single fixed sentinel.
- No CRS reprojection: `bounds`/`pixelSize`/`crs` are reported exactly as the source raster has them; if you need a specific CRS downstream, reproject the DEM before extraction.
- The preview's downsampling path (`--preview-max-dim`) rebuilds a full-resolution RGBA array in memory before resizing, so it doesn't reduce peak memory usage for very large DEMs — only the output file size/dimensions.
