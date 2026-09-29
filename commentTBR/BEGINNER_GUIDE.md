# PDA3 Beginner Guide: Teaching Notes

This guide is a short tour of the PDA3 codebase for someone who is learning Python, Flask, pandas, and forecasting. The application takes historical closing prices, prepares them, runs four forecasting approaches, and returns a JSON report for the dashboard.

> Forecasts are educational model outputs, not investment advice. A model that was accurate in the past can still be wrong about the future.

## The story of one analysis

1. `load_market_frame` or an upload provides a raw table.
2. `_normalise` makes the table predictable: it keeps a date and a numeric close price.
3. `analyze_frame` coordinates validation, model evaluation, future forecasts, indicators, and correlations.
4. The Flask routes in `app.py` expose that result as JSON.
5. The browser draws charts from the JSON response.

A tiny input looks like this:

```python
import pandas as pd
from analysis import analyze_frame

prices = pd.DataFrame({
    "Date": ["2026-01-02", "2026-01-05"],
    "Close": [100.0, 102.0],
})
# Real analysis needs at least 80 valid observations.
report = analyze_frame(prices, symbol="DEMO")
```

## `backend/analysis.py`

### Data creation and loading

- **`demo_frame(rows=420)`** creates repeatable synthetic business-day prices and volume. The fixed random seed makes tests and demos behave the same way each time. Example: `demo_frame(rows=120)` returns a DataFrame with 120 rows.
- **`load_market_frame(symbol, period)`** downloads historical prices from Yahoo Finance and turns the index into a normal `Date` column. Example: `load_market_frame("AAPL", "6mo")`.

### Cleaning and evaluation helpers

- **`_normalise(frame)`** accepts uploaded or downloaded data, finds `Date`/`Datetime` and `Close`/`Adj Close` columns without caring about capitalization, removes invalid rows, sorts dates, and requires 80 usable observations. Example: `_normalise(raw_frame)` returns `date` and `close` columns.
- **`_metrics(actual, predicted)`** compares known prices with predicted prices and returns MAE, RMSE, and MAPE. Smaller numbers generally mean smaller errors. Example: `_metrics([100, 110], [101, 108])`.
- **`_supervised(values, window=20)`** turns a sequence into learning examples: the previous 20 prices become the input and the next price becomes the target. Example: prices 1-20 predict price 21, then prices 2-21 predict price 22.

### Forecast functions

Each forecast function returns `(model, forecast)`: the fitted model object and a NumPy array of future prices.

- **`_linear_forecast(values, horizon)`** fits one straight line through the entire price history and extends that trend. Example: `_linear_forecast(prices, 5)` predicts the next five points.
- **`_random_forest_forecast(values, horizon)`** learns from recent 20-price windows, then feeds each prediction back into the next window. Example: `_random_forest_forecast(prices, 5)` makes five recursive predictions.
- **`_arima_forecast(values, horizon)`** uses an ARIMA `(2, 1, 2)` time-series model to forecast several future points at once. Example: `_arima_forecast(prices, 5)`.
- **`_lstm_forecast(values, horizon)`** scales prices, trains a small neural network on 20-step sequences, recursively predicts, and converts results back to the original price scale. Example: `_lstm_forecast(prices, 5)`; TensorFlow must be installed.

### Validation and orchestration

- **`_walk_forward(values, builder, horizon=30, minimum=100)`** holds out the later part of the series, trains a chosen forecast function on earlier prices, and scores its predictions against the held-out prices. Example: `_walk_forward(prices, _arima_forecast)`.
- **`analyze_frame(frame, symbol="UPLOADED", horizon=30)`** is the main public function. It calls the helpers and models, limits the horizon to 5-90 days, calculates indicators and correlations, and returns the complete dashboard payload. Example: `analyze_frame(demo_frame(), "DEMO", 10)`.

## The four forecasting models

These are intentionally different lenses, so the dashboard can compare them rather than pretending that one method is always best.

| Model | One-line beginner summary |
| --- | --- |
| Linear Regression | Draws the best straight trend line through the historical prices and extends it. |
| Random Forest | Combines many decision trees that learn how recent 20-price patterns relate to the next price. |
| ARIMA | Uses recent time-series behavior, differencing, and lag relationships to project the next values. |
| LSTM | Uses a neural network designed for sequences to learn patterns across a 20-price lookback window. |

### Reading the scores

- **MAE** is the average absolute size of an error, in price units.
- **RMSE** is similar but penalizes large misses more strongly.
- **MAPE** expresses the average error as a percentage of the actual price.

For example, an MAE of `2.5` means the predictions missed by about 2.5 price units on average over the validation period. These scores compare models on this dataset; they do not prove that a model will predict the future reliably.

## `backend/app.py`

### Database models

- **`User`** represents an account with an id, unique email, and hashed password.
- **`AnalysisResult`** represents a saved report with its symbol, creator, timestamp, and JSON payload.

### Application setup and page routes

- **`create_app(test_config=None)`** builds and configures the Flask application, database, JWT support, and HTTP routes. Passing a test configuration lets tests use a temporary database.
- **`dashboard()`** serves the HTML dashboard for `GET /`.
- **`health()`** returns a small status response for `GET /api/health`. Example response: `{"status": "ok", "service": "pda3-api"}`.

### Authentication routes

- **`register()`** validates an email and six-character minimum password, hashes the password, saves the user, and returns a JWT token for `POST /api/auth/register`.
- **`login()`** checks the submitted email and password against the stored hash and returns a JWT token for `POST /api/auth/login`.

### Analysis and result routes

- **`upload()`** reads a CSV or Excel upload, passes it to `analyze_frame`, saves the report, and returns JSON from `POST /api/data/upload`.
- **`analyze()`** loads live Yahoo Finance data for the requested symbol and period, analyzes it, saves it, and returns JSON from `POST /api/data/analyze`.
- **`results()`** lists the twelve most recent saved reports for `GET /api/results`.
- **`result_detail(result_id)`** finds one saved report and returns its complete JSON payload for `GET /api/results/<id>`.
- **`demo()`** runs the full pipeline using generated data so the dashboard can be tried without a network request. It serves `GET /api/demo`.
- **`export_result(result_id)`** takes the forecast list from a saved report, converts it to CSV in memory, and sends it as a download from `GET /api/results/<id>/export.csv`.
- **`save_result(result)`** is an internal helper that stores a completed report and associates it with the current logged-in user when there is one.

A simple API request can be made with Python like this:

```python
import requests

response = requests.get("http://localhost:5000/api/demo")
report = response.json()
print(report["models"]["ARIMA"]["forecast"][:3])
```

## A useful learning path

1. Run `python backend/app.py` and open `/api/demo` in a browser.
2. Read `demo_frame` and print `demo_frame(5)` to see a DataFrame.
3. Read `_normalise` and try changing a column name in a small DataFrame.
4. Compare the four model summaries and their validation metrics.
5. Trace `analyze_frame` from top to bottom, then follow the `analyze` route in `app.py`.
6. Run the tests with `cd backend` followed by `pytest`.

The two tests in `backend/test_analysis.py` demonstrate the expected shape of a successful report and the rule that very short price series are rejected.
