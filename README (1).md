# Freight Rate Prediction Challenge

A reproducible tabular-regression workflow for the Spotter freight-rate assessment. The implementation uses the supplied `score.py` contract and creates the required 12,000-row validation submission and 31-day December scenario chart input.

> The assessment PDF was not present in the supplied uploads. This implementation follows the README and `score.py` text supplied with the task. Confirm the official metric and data-sharing rules against the PDF before submission.

## Final approach

- Development data: 48,000 labeled rows from January 1 through October 31, 2025.
- Unlabeled validation data: 12,000 rows from November 1 through December 31, 2025 (5,836 November and 6,164 December rows).
- Primary validation: chronological holdout on October 2025 (43,147 train and 4,853 holdout rows; train January-September).
- Secondary robustness check: chronological holdout on September 2025 (train January-August).
- Final predictor: equal-weight blend of (1) Ridge regression trained on posted rate per mile and (2) CatBoost trained on `log1p(posted_rate)`.
- The target (`posted_rate`), load ID, and placeholder prediction column are explicitly excluded from model features.
- The training data contains 300 missing weights (0.63%), 374 missing market indices (0.78%), 292 negative weights and 1,191 weights at the 47,500 upper cap. Negative weights are treated as missing and flagged; capped values are retained and flagged. Missing numerical values are imputed by the Ridge preprocessing pipeline; CatBoost handles numerical missingness natively.
- Unknown pickup/delivery labels are supported. Geographic features are derived from coordinates supplied in the validation file.

## Validation results

Primary October holdout, 4,853 rows:

| Model | MAE ($) | RMSE ($) | R2 |
|---|---:|---:|---:|
| Mean baseline | 1,179.80 | 1,528.57 | -0.000 |
| Ridge regression | 141.73 | 649.65 | 0.819 |
| LightGBM | 150.88 | 666.23 | 0.810 |
| CatBoost (raw target) | 125.68 | 649.05 | 0.820 |
| Ridge (per-mile target) | 119.19 | 645.11 | 0.822 |
| CatBoost (log target) | 109.38 | 646.23 | 0.821 |
| **Final 50/50 ensemble** | **108.13** | **644.53** | **0.822** |

Secondary September holdout: ensemble MAE $110.82, RMSE $619.46, R2 0.835.

These are time-based holdout metrics. The 12,000-row validation.csv file is unlabeled, so its true error cannot be calculated locally.

## Project layout

```text
freight-rate-prediction/
├── data/                         # local input data; CSV files ignored by Git
├── models/                       # fitted Ridge and CatBoost artifacts
├── notebooks/
│   └── freight_rate_analysis.ipynb
├── outputs/
│   ├── charts/                   # EDA and feature importance charts
│   ├── metrics/                  # model comparison and run checks
│   ├── validation_predictions.csv
│   └── december_predictions.csv
├── reports/
│   └── freight_rate_assessment_report.pdf
├── src/
│   ├── preprocessing.py
│   ├── train.py
│   ├── predict.py
│   ├── evaluation.py
│   └── pipeline.py
├── score.py
├── requirements.txt
├── requirements-scorer.txt
├── validation_predictions.csv
├── december_predictions.csv
└── README.md
```

## Reproduce locally

1. Create a virtual environment and install dependencies:

   ```bash
   python -m venv .venv
   # Windows PowerShell: .venv\\Scripts\\Activate.ps1
   # macOS/Linux: source .venv/bin/activate
   python -m pip install -r requirements.txt
   ```

2. Place the four provided CSV files in `data/` with the exact names documented in `data/README.md`.

3. Run the workflow from the repository root:

   ```bash
   python -m src.pipeline \
     --train data/train_test.csv \
     --validation data/validation.csv \
     --template data/validation_predictions_template.csv \
     --december data/december_chart_inputs.csv \
     --project-dir .
   ```

   On Windows PowerShell, the same command can be entered on one line.

4. Run the provided scorer:

   ```bash
   python score.py --predictions validation_predictions.csv --december-predictions december_predictions.csv
   ```

5. The official scorer validates the two outputs and writes `scorer_results/candidate_december.png`. Copy that chart to `outputs/charts/` if regenerating the repository from scratch.

## December scenario assumptions

The supplied December scenario contains only seven columns and omits coordinates, `market_index`, and `quote_signal`. To create model-ready features without inventing row-level observations:

- Pickup and delivery coordinates are recovered from the city coordinate map in the labeled development data.
- `market_index` uses the median from the latest 30 days in the development data.
- `quote_signal` uses the median observed for the exact Lexington-Fort Wayne / Dry Van lane, because there are at least five such observations.

The actual defaults used are written to `outputs/metrics/december_scenario_imputation.json`. These scenario imputations affect only the 31-day chart; November-December validation predictions use their supplied feature values.

## Important submission note

Do not publish the raw challenge CSV files unless the assessment explicitly permits it. The Git ignore configuration excludes `data/*.csv`. The generated validation predictions are included because they are a required assessment deliverable.
