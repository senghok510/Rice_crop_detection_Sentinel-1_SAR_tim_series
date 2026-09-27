# CarbonFarm Rice Crop Classification — Work Summary

## Project objective

The notebook predicts whether a labelled geographical point contains rice. Latitude and longitude identify where Sentinel-1 satellite data should be retrieved; coordinates are not used directly as model predictors.

The operational workflow is:

```text
Latitude and longitude
        ↓
Retrieve Sentinel-1 data around the location
        ↓
Calculate spatial and seasonal SAR features
        ↓
Apply the fitted classification model
        ↓
Rice / Non Rice prediction
```

## Python coding tasks completed

### Sentinel-1 data retrieval

The original `get_sentinel_data` implementation was reviewed and rewritten. The completed implementation:

- validates latitude, longitude, time range, resolution, height, and width;
- tolerates the two malformed coordinate strings in the supplied CSV;
- creates the bounding box in the point's local UTM coordinate system, where dimensions are measured in metres;
- requests an exact `h × w` output window;
- retrieves VH, VV, and `dataMask` with their correct units;
- uses gamma0 terrain correction and Copernicus DEM orthorectification;
- converts no-data pixels to `NaN`;
- consistently returns an array of shape `(h, w, 2)` in VH/VV order; and
- avoids saving OAuth credentials in the notebook.

The implementation was checked against the official Sentinel Hub Process API, Sentinel-1 GRD, Evalscript, and data-mask documentation.

### `CropObservation` class

The `CropObservation` class now represents one labelled point and contains:

- latitude and longitude;
- the Rice/Non-Rice label;
- 9×9 VH and VV arrays;
- constructors for a coordinate pair or a CSV row;
- validation of coordinates, labels, array shapes, and window sizes;
- extraction of centred 1×1, 3×3, 5×5, 7×7, or 9×9 subwindows;
- spatial summary features for modelling;
- RVI and VH/VV ratio features; and
- a Folium map visualization method.

### Task 2.b map visualization

`CropObservation.to_map()` returns an interactive Folium map centred on the observation. Its marker contains:

- the land-class label in the tooltip;
- the class and coordinates in the popup; and
- a configurable initial zoom level.

The notebook includes and executes an example using a real 9×9 Sentinel-1 observation. The interactive output is embedded beneath the Task 2.b heading.

### Unit tests

Five unit tests cover:

- valid and malformed coordinate parsing;
- coordinate-range validation;
- exact metric bounding-box and output dimensions;
- data-mask application;
- centred band extraction;
- model-feature generation; and
- Folium map content.

All five tests pass.

## Satellite time-series dataset

The final model uses a full-year Sentinel-1 time series rather than one pixel from one date.

For each location, the workflow calculates statistics over a centred 9×9 window for:

- VH;
- VV;
- VH/VV ratio; and
- dual-polarization Radar Vegetation Index (RVI).

The year 2020 is divided into 25 approximately 15-day periods. For each feature and period, the cache contains:

- mean;
- standard deviation;
- 25th percentile;
- median;
- 75th percentile;
- valid-pixel count; and
- no-data-pixel count.

The cached dataset is stored in `sentinel1_timeseries_2020.csv.gz`.

Its structure is:

```text
600 observations × 25 periods × 4 feature families = 60,000 rows
```

The cache audit reports:

| Item | Value |
|---|---:|
| Rows | 60,000 |
| Observations | 600 |
| Spatial sites | 7 |
| Time periods | 25 |
| Rows with valid median values | 96% |

The remaining 4% correspond to location-period combinations without valid matching satellite pixels. Missing periods are excluded from the annual summaries, and the number of valid periods is retained as a model feature.

To reduce API calls, the extraction downloads one raster per spatial site and period, then extracts every point's 9×9 window locally. This requires at most 175 site-period requests instead of thousands of independent point-date requests.

## Spatial leakage audit

The 600 observations belong to seven geographically separated sampling sites:

- two rice sites; and
- five non-rice sites.

Because the CSV does not provide site identifiers, the sites are reconstructed from coordinates alone. The WGS84 longitude/latitude values are projected into the dataset's local metric coordinate system, **UTM zone 48N (EPSG:32648)**, before applying DBSCAN. This avoids treating angular degrees as though they were uniform physical distances.

DBSCAN uses a 3 km neighbourhood and a minimum of 20 observations. A sensitivity audit varies the neighbourhood radius from 0.25 km to 20 km. The same seven clusters, with no noise observations, remain stable from **1.25 km through 13.75 km**. The selected 3 km value is therefore inside a broad stable interval rather than being a narrowly tuned threshold. Class labels are not used to construct these groups.

Every site contains only one class. Consequently, a random row split places neighbouring observations from the same sites in both training and testing data. Such a split measures interpolation within known sites and can overestimate performance at a new location.

Coordinates and site IDs are therefore excluded from the model predictors. Site IDs are used only to construct validation folds.

## Feature engineering

Initial experiments using features aligned to exact calendar dates transferred poorly between the two rice sites because their planting calendars differed.

The final representation uses phase-invariant annual phenology descriptors. For every SAR feature and spatial statistic, it calculates:

- annual mean and standard deviation;
- minimum, maximum, and range;
- 10th, 25th, 50th, 75th, and 90th percentiles;
- interquartile range;
- mean absolute change;
- maximum rise;
- maximum fall;
- variability of consecutive changes; and
- number of valid periods.

This produces 240 model features for each observation. These describe seasonal behaviour without requiring the rice season to begin on the same calendar date at every site.

## Model training and validation

Four models were compared:

- constant non-rice baseline;
- regularized logistic regression;
- Extra Trees; and
- RBF-kernel support vector machine.

All preprocessing is contained inside each model pipeline. Imputation and scaling are therefore fitted only on the training sites in each fold.

The primary evaluation uses leave-one-site-out cross-validation. Each fold trains on six sites and predicts the seventh, ensuring that no observation from the validation site appears in training.

### Model comparison

| Model | Accuracy | Macro F1 | Rice F1 | ROC AUC |
|---|---:|---:|---:|---:|
| RBF SVM | 0.988 | 0.988 | 0.988 | 0.989 |
| Logistic regression | 0.950 | 0.950 | 0.952 | 0.992 |
| Extra Trees | 0.925 | 0.925 | 0.930 | 0.997 |
| Constant non-rice | 0.500 | 0.333 | 0.000 | 0.500 |

The RBF SVM was selected because it produced the best classification performance at the operational threshold. It is appropriate for this relatively small, high-dimensional dataset and can represent nonlinear relationships among seasonal SAR descriptors.

### Final confusion matrix

The pooled leave-one-site-out confusion matrix is:

| Actual \ Predicted | Non Rice | Rice |
|---|---:|---:|
| Non Rice | 300 | 0 |
| Rice | 7 | 293 |

This corresponds to:

- 300 true negatives;
- 293 true positives;
- 0 false positives;
- 7 false negatives; and
- 98.83% overall accuracy.

All errors were rice observations predicted as non-rice. All five held-out non-rice sites were classified correctly. The more difficult of the two held-out rice sites achieved approximately 95.3% accuracy.

### Random split comparison

The same RBF SVM achieved 100% accuracy and macro F1 with a random 70/30 row split. This result is considered optimistic because both partitions contain observations from all seven sites. The 98.8% leave-one-site-out result is the primary estimate.

## Correction to the original evaluation

The original in-sample report used the arguments in reverse order:

```python
classification_report(insample_predictions, y_train)
```

The correct call is:

```python
classification_report(y_train, insample_predictions)
```

Reversing these arguments does not change accuracy, but it changes the interpretation of precision, recall, and support.

The original logistic regression also had only two noisy, single-date predictors: VH and VV from one pixel. Its low training accuracy primarily indicated insufficient predictive information. The improved logistic model performs well because it receives spatially aggregated, full-season phenology features—not simply because a different classifier implementation was used.

## Final artifacts

- `Starting Notebook - Sentinel Hub.ipynb`: completed and executed notebook.
- `sentinel1_timeseries_2020.csv.gz`: reusable Sentinel-1 time-series feature cache.
- `PROJECT_SUMMARY.md`: this project summary.

The notebook contains embedded model tables, seasonal visualizations, a confusion matrix, an ROC curve, per-site diagnostics, and the Task 2.b interactive map.

## Validation completed

- All five Python unit tests pass.
- The model section executes successfully.
- The Task 2.b map was tested with a live Sentinel Hub request.
- The notebook conforms to the Jupyter notebook schema.
- The notebook contains no error outputs.
- No OAuth client ID or secret remains embedded in the notebook.
- The feature cache contains 60,000 rows covering all 600 observations, seven sites, 25 periods, and four SAR feature families.

## Limitations and recommended next steps

- The dataset contains only two independent rice sites. The results demonstrate transfer between the sampled sites but not universal performance across Vietnam.
- Four candidate models were selected using the same seven spatial folds. Because there are too few rice sites for nested spatial validation, the reported selected-model performance can retain modest model-selection optimism.
- The model is based on 2020 observations. Predictions for another year require Sentinel-1 features from the relevant growing season.
- A production evaluation should include more rice and non-rice regions, multiple years, different acquisition geometries, and completely untouched geographic test regions.
- Probability calibration and the operational decision threshold should be validated before deployment.
