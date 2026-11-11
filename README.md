# Permafrost–Wildfire KBDI and CMIP6 Analysis

Python programs for frozen-ground classification, daily Keetch–Byram Drought Index (KBDI), CMIP6 scenario processing, multi-model ensemble statistics, wildfire trends and active-layer analysis.

## Modules

| Module | Input | Output |
|---|---|---|
| `src/permafrost/Permafrost_distribution_data.py` | Annual `FI`, `TI` and plateau mask | Frozen-ground classes (NetCDF) |
| `src/permafrost/frost_number_classification.py` | Annual `FI`, `TI` and plateau mask | Frozen-ground classes (NetCDF) |
| `src/kbdi/kbdi_core.py` | Daily maximum temperature, precipitation, annual precipitation climatology | Daily KBDI |
| `src/kbdi/run_cmip6_kbdi.py` | Historical and SSP `tasmax` / `pr` (NetCDF) | Annual and optional daily KBDI per GCM |
| `src/kbdi/ensemble.py` | Individual GCM KBDI outputs | Ensemble mean, inter-model SD and model counts |
| `src/statistics/wildfire_all_latitudes_mk_sen.py` | Annual normalized burned area (`.xlsx`) | Mann–Kendall and Sen slope statistics |
| `src/statistics/analyze_ALT_significance.py` | Annual active-layer observations (`.xlsx`) | OLS trends, Pearson `r`, t-test statistics |
| `src/statistics/methane_fire_statistics.py` | Daily methane, temperature, humidity and fire detections (`.xlsx`) | Negative-binomial regression and temporal-shift test |

## Climate model configuration

`src/cmip6_config.py` specifies 25 CMIP6 models and four scenarios: `ssp126`, `ssp245`, `ssp370`, `ssp585`.

The frozen-ground model uses **annual** freezing index (`FI`) and thawing index (`TI`). KBDI uses **daily** `tasmax` and `pr`. The two calculation modules are independent.

## Installation

Python 3.10+:

```bash
python -m pip install -r requirements.txt
```

## KBDI inputs

Expected NetCDF layout (one complete calendar year per file):

```text
cmip6/
  historical/
    tasmax_ACCESS-CM2_1950.nc
    pr_ACCESS-CM2_1950.nc
  ssp126/
    tasmax_ACCESS-CM2_2015.nc
    pr_ACCESS-CM2_2015.nc
  ssp245/
  ssp370/
  ssp585/
```

`tasmax` input units: kelvin or degrees Celsius; `pr` input units: kg m⁻² s⁻¹ or mm/day. The program checks variable units and daily time coordinates. Input paths can be configured with `--file-template`.

KBDI uses the temperature-dependent drying equation and 0.2-inch rainfall interception. The CLI supports rainfall interception per consecutive wet period (`wet_spell`) or per day (`daily`). KBDI is initialized at the configured historical start year and then carried forward daily, including the transition from 2014 to each SSP.

The long-term annual precipitation parameter is computed from each model's historical baseline (default 1981–2014). Warm-season output defaults to May–September.

### Individual CMIP6 model and scenario calculations

```bash
python src/kbdi/run_cmip6_kbdi.py \
  --input-root /data/cmip6 \
  --output-root /data/kbdi_by_gcm \
  --historical-start 1950 --historical-end 2014 \
  --baseline-start 1981 --baseline-end 2014 \
  --start-year 2015 --end-year 2099 \
  --rainfall-mode wet_spell
```

### Multi-model ensemble

```bash
python src/kbdi/ensemble.py \
  --model-output-root /data/kbdi_by_gcm \
  --output-root /data/kbdi_mme \
  --start-year 2015 --end-year 2099 \
  --min-models 25
```

The ensemble mean is calculated from KBDI values computed separately for each GCM, with equal model weights.

## Frozen-ground classification

```bash
python src/permafrost/frost_number_classification.py \
  --input-root /data/frozen_indices \
  --mask /data/frozen_indices/GTOPOP_v4.nc \
  --output-root /data/frozen_classes \
  --start-year 1950 --end-year 2099
```

The Frost Number is `sqrt(FI) / (sqrt(FI) + sqrt(TI))`. The plateau class thresholds are 0.55 and 0.01; outside the plateau they are 0.53, 0.10 and 0.01. Frozen-ground classes are encoded as 0 (unfrozen), 1 (intermittent), 2 (seasonal) and 3 (permafrost).

## Statistical analysis

### Burned area: Mann–Kendall and Sen's slope

```bash
python src/statistics/wildfire_all_latitudes_mk_sen.py \
  --input /data/Wildfire_data.xlsx --output-dir results/wildfire
```

Input normalized values are multiplied by 100 before trend analysis; slopes refer to the scaled normalized index per year.

### Active-layer thickness: Pearson correlation and OLS

```bash
python src/statistics/analyze_ALT_significance.py \
  --input /data/Thawing_Depths.xlsx --output-dir results/alt
```

Optional map annotation: add `--figure /data/Fig2.png`.

### Methane and fire counts: negative-binomial regression

```bash
python src/statistics/methane_fire_statistics.py \
  --input /data/monitoring.xlsx --output-dir results/methane \
  --permutations 1000 --seed 20261109
```

## Tests

```bash
python -m pip install -r requirements-ci.txt
PYTHONPATH=src pytest -q
```

```bash
bash examples/run_demo.sh
```

## Sources

- Keetch, J. J. & Byram, G. M. (1968). *A Drought Index for Forest Fire Control*. USDA Forest Service Research Paper SE-38.
- NEX-GDDP-CMIP6: https://doi.org/10.7917/OFSG3345
- Frozen-ground distribution data: https://doi.org/10.11888/Cryos.tpdc.300901
- Circumpolar Active Layer Monitoring (CALM): https://www2.gwu.edu/~calm/
