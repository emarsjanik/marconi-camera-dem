# Marconi Beach camera DEMs (CACO05)

Intertidal digital elevation models (DEMs) of Marconi Beach, Cape Cod, made from two shore-based cameras.

## The idea
- Two cameras on the bluff, both looking north-east over the beach (EO azimuths 54.5° for c1 and 17.5° for c2; c1 covers the stretch south of the seam at northing 4638415), take a timex photo every 30 minutes. A timex is a time-exposure average that blurs the waves into a clean water edge.
- Every waterline is a contour: where the water meets the sand, the beach elevation equals the water level at that moment plus the wave setup.
- As the tide rises and falls, the waterline sweeps the beach. Hundreds of waterlines at different water levels, gridded at 2 m, give an intertidal DEM.
- Water level comes from GNSS-R (the live station), an ADCP (Jan–Mar 2025) or the Chatham tide gauge transferred to Marconi (older dates).
- Wave setup is C·√(Hs·L0) with C = 0.037, fitted to the 29 Sep 2026 RTK survey.

## Pipeline
| Step | Script | What it does |
|---|---|---|
| Detect | `waterline_detector_v5.py`, `detect_original_view.py` | Finds the water/sand edge in each photo, inside a search band |
| Elevate | `extract_elevation_contours.py`, `setup_model.py`, `timex_timing.py` | Water level + setup for each line (setup formula in one place; opt-in Stockdon slope-aware setup; `TIMING_MODE` switch for mid-exposure timing, see VALIDATION 2.7) |
| Georectify | `georectify.py`, `view_reproject.py` | Pixel to UTM (EPSG:6348) using the calibration in `calibration/` |
| Quality control | `waterline_consistency.py`, `waterline_track.py` | Drops lines out of tidal order, whole bad days (e.g. snow) and traces running out over the sea |
| DEM | `dem_from_contours.py`, `dem_loess.py` | 2 m cells: median of ≥3 crossings; spread >0.5 m blanked; waves >1.5 m left out; day-consistency test; `_sigma.asc` standard error per cell; tide-stage/Hs residual and c1−c2 offset diagnostics; opt-in loess surface with prediction error (Plant et al. 2002) |
| Output | `asc_to_geotiff.py`, `dem_map.py`, `dem_figure.py`, `daily_elevation_map.py` | GeoTIFFs (`dem`, `spread`, `count`), a plain north-up map, a diagnostic page, lines on the photos |
| Compare | `survey_compare.py`, `compare_dem_survey.py`, `compare_rtk.py` | DEM minus survey statistics, by camera, elevation band and day, with block-bootstrap intervals and flood/ebb, Hs and range strata |
| Uncertainty and stability | `georectify_uncertainty.py`, `pointing_daily.py`, `recalibrate_timing.py`, `evaluate_setup_model.py` | Leave-one-out GCP residuals and per-pixel geometric σ; daily pointing log and opt-in per-day EO correction; NUC timing recalibration; leave-one-survey-out setup evaluation |

## Two ways it runs
- **Live station:** `waterline_timex_cron.sh` runs three times a day. On the station it also calls `owg.sh` (the optical wave gauge, not included here), which writes `archive/waves_marconi.csv` hourly with `marconi_waves.py`. Without that file the cron falls back to the raw NDBC buoy, a different wave currency from the one C was fitted in, and the DEM log now says so with a WARNING. Run `marconi_waves.py` hourly yourself if you run the cron from this repository. It builds the whole-archive DEM, the 7-day DEM, the week-to-week change map and the pictures for the daily email. `gnssr_qc.py` screens the GNSS-R water level against the Chatham gauge; `station_status.py` writes the health line.
- **Survey-date products:** `survey_products.py --date YYYY-MM-DD` builds one documented product per survey date:
  - forcing (`historical_forcing.py`);
  - pointing check (`pointing_check.py`, `horizon_check.py`, `estimate_eo_rotation.py`);
  - detection, filter, DEM, maps and comparison;
  - a README, `provenance.json` and an independence label (INDEPENDENT / PARTLY-CIRCULAR / CIRCULAR);
  - `--share` packs a date into one archive.
- **Blind dates:** the survey is held by someone else; no survey file is ever opened.
- `gcpfree_eo.py`: the pre-registered, GCP-free calibration procedure (DEM B).

## Validation
`docs/VALIDATION.md` lists every claim with the command that reproduces it. Tests (plain `python3`, no pytest needed):
every `test_*.py` in the root, and `python3 waterline_consistency.py --self-test`. `georectify.py` and `view_reproject.py` are sha256-pinned for the GCP-free check (`GCPFREE_PINS`).

## Requirements
Python ≥ 3.8, numpy, OpenCV, matplotlib, pandas. Photos, water-level records and surveys are not included.
