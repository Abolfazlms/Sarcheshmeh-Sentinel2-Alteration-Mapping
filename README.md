# Reproducibility of Sentinel-2 Alteration Mapping — Sarcheshmeh, Iran

This repository contains the Python/Jupyter implementation for:

**“Reproducibility of Self-Supervised and Classical Clustering for Multi-Temporal Sentinel-2 Alteration Mapping in the Sarcheshmeh Porphyry Copper District, Iran.”**

The workflow compares classical clustering baselines and a masked-feature autoencoder representation for multi-temporal Sentinel-2 alteration screening. It emphasizes reproducibility diagnostics and explicitly treats candidate zones as **unverified screening outputs**, not confirmed mineralisation targets.

## Repository contents

```text
.
├── sarcheshmeh_full_study_publication_ready.ipynb   # Main end-to-end notebook
├── README.md
└── requirements.txt
```

After a successful run, the notebook creates:

```text
results/
├── figures/       # Diagnostic and comparison figures
├── tables/        # CSV tables (metrics, cluster signatures, optional validation, zones)
├── rasters/       # GeoTIFF outputs when rasterio/georeferencing are available;
│                  # NumPy arrays may be written as a fallback
├── vectors/       # Vector outputs when optional geospatial dependencies are available
├── report.html
├── report_summary.md
├── run_metadata.json
└── run_log.txt

results_package.zip
```

The exact set of output files depends on optional inputs and the availability of geospatial packages.

## Study area and temporal design

The default area of interest is the Sarcheshmeh district, Kerman Province, Iran.

- Bounding box (longitude/latitude): `55.75, 29.85, 56.00, 30.05`
- Output CRS: UTM zone 40N (`EPSG:32640`)
- Target grid resolution: 20 m
- Period A (model fitting): 2022-02-01 to 2024-10-31
- Period B (independent temporal application): 2024-11-01 to 2026-10-31

The notebook queries the public Earth Search STAC API at run time. It searches Sentinel-2 Level-2A scenes with a scene-level cloud-cover threshold and creates median composites. The Scene Classification Layer (SCL) retains classes 4 (vegetation) and 5 (bare soil). Pixels with fewer than the configured minimum number of valid observations are excluded.

A Copernicus DEM GLO-30 product is used to derive terrain slope. The same slope layer is appended to the spectral feature vector for the `SSL_AE_DEM` variant.

**Important:** the catalogue is queried at run time rather than using a permanently frozen scene manifest. Catalogue updates, asset availability, and the exact scene-selection stride can therefore affect the scenes selected and the resulting composites. The notebook log records the number of scenes found and used in each period. For a fully frozen reproduction, archive the selected STAC item IDs and the exact environment/package versions alongside the outputs.

## Methods implemented

The notebook constructs six reflectance features:

- Blue (B2)
- Green (B3)
- Red (B4)
- Narrow NIR (B8A / `nir08`)
- SWIR1 (B11 / `swir16`)
- SWIR2 (B12 / `swir22`)

It also computes three band ratios:

- Clay proxy: `B11 / B12`
- Ferric-iron proxy: `Red / Blue`
- Ferrous/hydroxyl proxy: `B11 / B8A`

Reflectances are clipped before ratio calculation. Terrain slope is included as an additional feature for the DEM-assisted variant.

Five clustering strategies are implemented with `MiniBatchKMeans` (`k=8`):

1. `KMeans_bands` — six reflectance bands.
2. `KMeans_ratios` — three band ratios.
3. `PCA3` — PCA representation of the nine spectral features, followed by k-means.
4. `SSL_AE` — an 8-dimensional representation learned by a multilayer perceptron autoencoder with feature masking, followed by k-means.
5. `SSL_AE_DEM` — the same masked-feature autoencoder with terrain slope appended.

The autoencoder masks a fraction of input **features** and computes reconstruction loss on the masked features. It is a tabular masked-feature autoencoder; it is not a patch-based image masked autoencoder.

Each method is fitted on Period A and applied to Periods A and B without refitting. Five random seeds are configured by default (`0`–`4`).

## Reproducibility metrics

The notebook reports:

- **Temporal consistency:** fraction of common valid pixels retaining the same cluster index when a Period-A-fitted model is applied to Period B.
- **Seed stability:** mean pairwise Adjusted Rand Index (ARI) between seed-specific partitions.
- **Spatial coherence:** fraction of valid four-neighbour pixel pairs sharing the same cluster label.
- **Separability:** silhouette coefficient and Davies–Bouldin index in a common standardized feature space.

These are label-free diagnostics. They measure consistency and partition properties; they do **not** establish geological correctness.

## Optional reference-based validation

To run supervised spatial-block validation, provide an independent polygon dataset (for example, mapped alteration or mineralisation reference polygons) and set:

```python
CFG["ref_path"] = "/path/to/reference_polygons.geojson"
```

The notebook rasterizes the reference polygons and compares feature sets using a Random Forest classifier under both random cross-validation and spatial GroupKFold validation. The default spatial block size is 2 km (`CFG["block_km"]`).

The reference polygons must be independent of the model-fitting process and should be documented in the resulting study. If no reference file is supplied, supervised validation is skipped; do not report supervised accuracy metrics.

## Mining-disturbance mask

The notebook first uses `CFG["mine_mask_path"]` if a user-provided polygon file is specified. Otherwise, it attempts a best-effort OpenStreetMap Overpass query for quarry, mine, tailings and industrial polygons.

If the OSM request fails or returns no polygons, the workflow continues with an empty mask. **Check the run log and the reported masked fraction.** An empty mask means mine-affected pixels may remain in the analysis and should be discussed as a limitation.

## Requirements

- Python 3.10 or later is recommended.
- PyTorch is required for the masked-feature autoencoder.
- A working internet connection is required to query the STAC catalogue and retrieve imagery/DEM assets.
- RAM requirements depend on the area, number of scenes, and chunking; a cloud runtime with sufficient memory is recommended.

Install the packages with:

```bash
pip install -r requirements.txt
```

For GPU acceleration, install a PyTorch build compatible with your CUDA driver using the official PyTorch installation selector. The notebook automatically selects CUDA when available and otherwise uses CPU.

### Important autoencoder check

The notebook contains a PCA fallback if PyTorch cannot be imported. In that case, the methods named `SSL_AE` and `SSL_AE_DEM` are **not actual autoencoder experiments**. After running the notebook, inspect:

```text
results/run_metadata.json
```

and verify that:

```json
"ae_available": true
```

If it is `false`, install PyTorch and rerun the notebook before describing the results as self-supervised autoencoder results.

## Run the notebook

### Option A — Google Colab

1. Upload `sarcheshmeh_full_study_publication_ready.ipynb` to Google Colab.
2. Select a runtime with sufficient RAM; enable a GPU if available.
3. Run all cells from top to bottom.
4. Wait for the data retrieval, model fitting, evaluation, and export steps to finish.
5. Download `results_package.zip` when prompted, or retrieve it from the notebook session.

### Option B — Local Jupyter

1. Create and activate a virtual environment.
2. Install dependencies with `pip install -r requirements.txt`.
3. Start Jupyter Lab or Notebook.
4. Open the notebook and run all cells.
5. Inspect the generated `results/` directory and `results_package.zip`.

## Main configuration

The main settings are in the `CFG` dictionary near the beginning of the notebook:

| Setting | Meaning | Default |
|---|---|---|
| `bbox` | Study-area bounding box | `(55.75, 29.85, 56.00, 30.05)` |
| `epsg` | Output CRS | `32640` |
| `res` | Target resolution (m) | `20` |
| `period_A` | Fitting period | `2022-02-01` to `2024-10-31` |
| `period_B` | Temporal test period | `2024-11-01` to `2026-10-31` |
| `cloud_lt` | Scene cloud-cover threshold (%) | `15` |
| `max_scenes` | Scene-selection stride control | `30` |
| `k` | Number of clusters | `8` |
| `seeds` | Random seeds | `[0, 1, 2, 3, 4]` |
| `latent` | Autoencoder latent dimension | `8` |
| `epochs` | Autoencoder training epochs | `15` |
| `n_train` | Pixels sampled for fitting | `300000` |
| `primary` | Primary method for selected maps | `SSL_AE` |
| `ref_path` | Optional reference polygons | `None` |
| `mine_mask_path` | Optional mine-disturbance polygons | `None` |
| `block_km` | Spatial validation block size (km) | `2.0` |
| `min_target_ha` | Minimum connected candidate area (ha) | `5` |
| `top_quantile` | Alteration-index quantile threshold | `0.90` |
| `conf_min` | Minimum cluster agreement | `0.80` |

## Interpretation and limitations

- Band ratios can become unstable when denominator reflectance is very small. Inspect reflectance and ratio distributions before interpreting alteration indices.
- The ferric ratio (`Red / Blue`) is particularly sensitive to low blue reflectance.
- Sentinel-2 has two SWIR bands; these data alone do not uniquely identify individual alteration minerals.
- Clustering labels are statistical groups and do not inherently correspond to geological alteration zones.
- High temporal consistency or spatial coherence does not imply geological accuracy.
- The optional OSM mask is not guaranteed to be complete.
- Candidate zones require independent geological references, field spectroscopy, and/or laboratory confirmation before being treated as exploration targets.

## Data sources

- **Sentinel-2 Level-2A:** accessed through the public Earth Search STAC API: https://earth-search.aws.element84.com/v1
- **Copernicus DEM GLO-30:** accessed through the Earth Search STAC catalogue.
- **OpenStreetMap:** optionally queried for mining-disturbance polygons through the Overpass API.

The notebook retrieves the remote data when executed. This repository does not redistribute the satellite imagery or DEM.

## Citation

If you use this code, please cite the associated paper:

> Abolfazl Mastaalizadeh and Fatemeh Sheybani, “Reproducibility of Self-Supervised and Classical Clustering for Multi-Temporal Sentinel-2 Alteration Mapping in the Sarcheshmeh Porphyry Copper District, Iran,” [publication details to be added].

When the paper is accepted/published, replace the placeholder with the final venue, year, pages, and DOI. If the repository is archived on Zenodo, also cite the version-specific Zenodo DOI.

## License

No software license is specified in this repository yet. Add a `LICENSE` file before describing the code as open-source or granting reuse rights. Choose a license only after confirming that all included code and assets can be distributed under it.
