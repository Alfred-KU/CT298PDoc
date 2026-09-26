# Batch Mode Guide — `batch_report.py`

This guide explains how to run PDA3 analysis outside the web dashboard, in a
scriptable/headless "batch mode", and how to read the PDF report it produces.

`batch_report.py` reuses the exact same analysis pipeline as the Flask API
(`analyze_frame` in [analysis.py](analysis.py)), so results match what you would
see in the dashboard for the same input file — no server, browser, or database
is required.

## 1. Setup

```powershell
cd backend
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

`requirements.txt` includes `matplotlib`, which `batch_report.py` uses to render
charts, tables, and the PDF itself.

## 2. Running the batch report

Basic usage — analyze a CSV and write a PDF next to it:

```powershell
cd backend
python batch_report.py --input "..\source_added\0005.HK260101.csv" --symbol 0005.HK
```

This writes `0005.HK-report.pdf` into `source_added\` (same folder as the input
file), because no `--output` was given.

### Command-line options

| Option | Short | Required | Description |
|---|---|---|---|
| `--input` | `-i` | Yes | Path to a CSV or Excel file with `Date` and `Close` (or `Adj Close`) columns. |
| `--symbol` | `-s` | No | Label shown in the report title and tables. Defaults to the input filename (e.g. `0005.HK260101.csv` → `0005.HK260101`). |
| `--horizon` | `-n` | No | Number of future trading days to forecast. Must be between 5 and 90 (default `30`). |
| `--output` | `-o` | No | Destination PDF path. Defaults to `<symbol>-report.pdf` next to the input file. |

### Example: custom horizon and output path

```powershell
python batch_report.py `
  --input "..\source_added\0700.HK260101.csv" `
  --symbol 0700.HK `
  --horizon 60 `
  --output "..\reports\0700-60day.pdf"
```

### Batch-processing every file in a folder

To run the report for every CSV in `source_added/` in one go:

```powershell
cd backend
Get-ChildItem "..\source_added\*.csv" | ForEach-Object {
    python batch_report.py --input $_.FullName --output "..\reports\$($_.BaseName)-report.pdf"
}
```

Input files need at least 80 valid price rows; the script prints an error and
returns a non-zero exit code (without crashing) if a file is invalid, too
short, or missing required columns — useful for scripted/CI batch runs where
you want to skip bad files and continue.

## 3. What's in the PDF report

Each report has 5 pages, generated from the same payload structure returned by
the `/api/data/analyze` and `/api/data/upload` endpoints:

1. **Cover page** — symbol, source filename, generation timestamp, number of
   observations used, and the latest technical indicators:
   - `last_close`, `change_30d` (%), `sma_20`, `sma_50`, `volatility` (annualized, %).
   - The standard disclaimer that outputs are educational, not investment advice.
2. **Historical price + forecast chart** — the black line is actual historical
   close prices; the dashed colored lines are each model's forward forecast
   (Linear Regression, Random Forest, ARIMA, LSTM) for the requested horizon.
3. **Model validation metrics table** — MAE, RMSE, and MAPE (%) for each model,
   computed on a chronological walk-forward holdout (not the future forecast).
   Lower values mean the model tracked the holdout period more closely. A
   `Notes` column shows an error message if a model failed to fit (e.g. LSTM
   without TensorFlow installed).
4. **Forecast table** — up to 30 rows of per-day forecast values for every
   model, useful for pulling exact numbers instead of reading them off the chart.
5. **Correlation matrix heatmap** — pairwise correlation between close price,
   daily returns, and the 20/50-day moving averages. Values near `1`/`-1`
   indicate strong positive/negative relationships; values near `0` indicate
   little linear relationship.

## 4. Interpreting the output

- **Compare models using page 3, not page 2.** The chart on page 2 shows each
  model's forecast, but only page 3's MAE/RMSE/MAPE tell you which model was
  historically more accurate on held-out data.
- **Lower MAPE (%) is generally the easiest metric to compare across models**
  because it's scale-independent (a % error rather than a price error).
- **Diverging model forecasts on page 2 signal higher uncertainty.** If Linear
  Regression, Random Forest, ARIMA, and LSTM diverge sharply, treat the
  forecast horizon as less reliable; if they broadly agree, there's more
  consensus (though still no guarantee of future accuracy).
- **`volatility` on the cover page** is the annualized standard deviation of
  daily returns — higher values indicate a historically more volatile stock.
- Every report repeats the disclaimer: forecasts are educational model
  outputs, not investment advice, and historical accuracy does not guarantee
  future performance.

## 5. Exit codes (for scripting/CI)

- `0` — report generated successfully.
- `1` — input file not found, invalid data (e.g. missing `Close` column), or
  fewer than 80 valid price rows. The error message is printed to stderr.
