# Drought indices for İznik and Uluabat lakes

Python code for (1) SPI / SPEI from CHIRPS and GLEAM monthly data and (2) a
Standardized Lake Level Index (SLLI) built from gauge, altimetry and DAHITI
levels, cross-checked against standardized NDWI.

```
spi_spei.py     SPI (gamma) and SPEI (log-logistic, PWM), 1-24 month scales
slli_ndwi.py    SLLI, robustness analyses, NDWI cross-check
requirements.txt
```

## Install and run

```bash
pip install -r requirements.txt

python spi_spei.py  --data-dir data --output-dir results --lakes Iznik Uluabat
python slli_ndwi.py --lake Iznik   --data-dir data --output-dir results
python slli_ndwi.py --lake Uluabat --data-dir data --output-dir results
```
Run `python <script> --help` for all options (file-name templates, calibration
period, thresholds, etc.). No machine-specific paths are hard-coded.

### Expected inputs (`data/`)
| Script | File | Columns |
|---|---|---|
| spi_spei | `{Lake}_Monthly_Precip_1981_2024.csv` | `Date`, `Precipitation_mm` |
| spi_spei | `{Lake}_Monthly_PET_1981_2024.csv` | `Date`, `Potential_Evaporation_mm` |
| slli_ndwi | `Observed_{Lake}_Lake.xlsx`, `Satellite_{Lake}.xlsx`, `Dahiti_{Lake}_MA.xlsx` | `Date` + one column whose name contains the lake name |
| slli_ndwi | `{Lake}_NDWI_Stats.xlsx` (optional) | `image_date`, `ndwi_mean` (or `ndwi_median`) |

Raw data are **not** included. Check the licence/terms of CHIRPS, GLEAM, DAHITI
and the gauge provider before committing any data to a public repository.

## Methods in brief
* **SPI** - gamma distribution per calendar month, shape/scale by the Thom (1958)
  ML approximation, mixed model for zero totals (WMO-No. 1090, 2012).
  KS goodness of fit is reported with a naive and a Monte-Carlo (Lilliefors-type) p-value.
* **SPEI** - three-parameter log-logistic fitted by probability-weighted moments
  (Vicente-Serrano et al., 2010). If the log-logistic cannot be fitted (typical for
  near-normal long-scale sums) a normal distribution is used and flagged in
  `*_SPEI_loglogistic_fit_parameters.csv`. Values hitting the probability clip
  (|SPEI| ~ 4.75) are counted and logged.
* **SLLI** - per-calendar-month empirical CDF -> inverse normal (Hazen plotting
  position). Gamma, lognormal, Weibull and normal variants are provided as sensitivity checks.
* **Lake-level merge** - satellite and DAHITI are mapped to the gauge datum on the
  overlap period (`--bias-method ols|mean_std`), then merged: observed > inverse-variance
  blend > single source. `Best_Source` records provenance of every month.
* **NDWI check** - lag correlations in both directions; p-values use an effective
  sample size to account for serial correlation.

## Known limitations (please read before citing results)
1. **Rank-based indices are standard-normal by construction.** About 16 % of months
   fall below -1 regardless of how dry the lake actually is, so event counts compare
   *relative* anomalies, not absolute water stress. See `Threshold_Sensitivity`.
2. **Sample size caps the index.** With *n* values per calendar month the lowest
   attainable SLLI is Φ⁻¹(0.5/n) (n = 32 -> -2.15; n = 10 -> -1.64). "Extreme drought"
   (<= -2) is only reachable if n >= 22.
3. **Non-stationarity.** Trends or regulation in lake level are absorbed into the index.
   `Split_Sample_Test` calibrates on one half of the record and evaluates on the other;
   a held-out mean far from 0 signals non-stationarity.
4. **OLS bias correction shrinks variance** (regression to the mean), which can distort
   the merged series where it switches source. Compare with `--bias-method mean_std`.
5. **Lag selection** (`Best_lag`) searches 2*max_lag+1 correlations; its p-value is not
   adjusted for that search.
6. SPI/SPEI are calibrated on the full record by default; with a warming PET trend,
   SPEI depends on the calibration window (`--calib-start/--calib-end`).

## References
* McKee, Doesken & Kleist (1993), 8th Conf. on Applied Climatology.
* Thom (1958), Monthly Weather Review 86.
* WMO (2012), Standardized Precipitation Index User Guide, WMO-No. 1090.
* Vicente-Serrano, Beguería & López-Moreno (2010), J. Climate 23, 1696-1718.
* Bretherton et al. (1999), J. Climate 12, 1990-2009 (effective degrees of freedom).
* Lilliefors (1967), J. Amer. Statist. Assoc. 62, 399-402.
