# Lesson 5 — Build, evaluate, and save a model

## Goal

Compare an honest baseline with an interpretable regression, using future dates
only for evaluation. Understand the result even if regression loses.

## 1. Learn the quantities with tiny examples

For actual units `[10, 20]` and predictions `[12, 14]`:

```text
Absolute errors: 2 and 6
MAE = (2+6)/2 = 4 units
WAPE = (2+6)/(10+20) = 0.2667 = 26.67%
```

MAE has units. WAPE is a fraction unless displayed as a percentage. Averaging
product WAPEs is not the same as calculating overall WAPE from all observations.

**Baseline:** predict each product's mean units from earlier fitting data.
Never calculate that mean from the holdout.

**Regression:**

```text
log(units) = intercept + beta*log(price)
             + product effects + weekday effects + time trend + error
```

`log` is the natural logarithm. Both units and prices must be positive.
Product and weekday effects allow different intercepts for categories.
The pooled beta assumes the same price elasticity across these six products.
For small changes, beta=-1.5 means 1% higher price is associated with about 1.5%
lower units. For a finite change, the multiplier is `(new_price/old_price)^beta`.

Why logs? A multiplicative relationship becomes additive. This is a modeling
assumption, not a discovery that all real demand has constant elasticity.

**Paper exercise:** use `units = 10 × price^(-1.5)` with price 2, then 2.20.
Calculate both demand values and revenues with a calculator. Confirm that the
demand ratio is `1.10^(-1.5)` and the revenue ratio is `1.10^(-0.5)`. Changing
the intercept changes scale, but it does not change those ratios.

## 2. Agree on evaluation before looking at its result

Use shared distinct observed dates: first 60% train, next 20% validation, last
20% holdout. Each date belongs to exactly one partition for all products.

Choose between two price measures using **validation MAE**. Compare the chosen
regression with the baseline. Refit on train+validation, then evaluate on the
holdout. Do not change the model because you dislike the holdout score.

Our earlier full-window exploration and product selection mean this holdout is
still exploratory. It is not a pristine confirmatory experiment.

## 3. Write the first half of your training script

Create **`learning/train.py`**:

```python
import json
import uuid
import numpy as np
import pandas as pd
import statsmodels.formula.api as smf
from psycopg.types.json import Jsonb
from db import WORK, connect, fetch

FORMULA = "log_units ~ log_price + C(stock_code) + C(weekday) + trend_years"


def features(data, price):
    result = data.copy()
    result["log_units"] = np.log(result.units_sold.astype(float))
    result["log_price"] = np.log(result[price].astype(float))
    result["trend_years"] = result.time_index / 365
    return result


def fit(data, price):
    transformed = features(data, price)
    model = smf.ols(FORMULA, data=transformed).fit()
    smear = float(np.exp(model.resid).mean())
    return model, smear


def predict(model, smear, data, price):
    return np.exp(model.predict(features(data, price))) * smear


def score(actual, predicted):
    errors = np.abs(np.asarray(actual) - np.asarray(predicted))
    return {"mae": float(errors.mean()),
            "wape": float(errors.sum() / np.asarray(actual).sum())}


data = fetch("SELECT * FROM modeling_daily_sales ORDER BY sale_date, stock_code")
data["sale_date"] = pd.to_datetime(data.sale_date)
for column in ["units_sold", "weighted_unit_price", "median_line_price"]:
    data[column] = data[column].astype(float)
dates = np.sort(data.sale_date.unique())
validation_start = dates[int(0.6 * len(dates))]
test_start = dates[int(0.8 * len(dates))]
train = data[data.sale_date < validation_start]
validation = data[(data.sale_date >= validation_start) & (data.sale_date < test_start)]
development = data[data.sale_date < test_start]
test = data[data.sale_date >= test_start]
assert train.sale_date.max() < validation.sale_date.min()
assert validation.sale_date.max() < test.sale_date.min()
for name, frame in [("train", train), ("validation", validation), ("test", test)]:
    print(name, len(frame), frame.sale_date.min(), frame.sale_date.max())

validation_scores = {}
means = train.groupby("stock_code").units_sold.mean()
validation_scores["baseline"] = score(validation.units_sold, validation.stock_code.map(means))
for price in ["weighted_unit_price", "median_line_price"]:
    model, smear = fit(train, price)
    validation_scores[price] = score(validation.units_sold, predict(model, smear, validation, price))
chosen_price = min(["weighted_unit_price", "median_line_price"],
                   key=lambda name: validation_scores[name]["mae"])
winner = min(validation_scores, key=lambda name: validation_scores[name]["mae"])
print("Validation:", validation_scores)
print("Chosen regression price:", chosen_price, "Overall winner:", winner)
```

Run the partial script:

```sh
.venv/bin/python learning/train.py
```

Expected splits: train 1,027 rows through 2011-07-13; validation 347 rows from
2011-07-14 through 2011-09-21; test 355 rows from 2011-09-22 through 2011-11-30.
The baseline should win validation; weighted price should beat median price
among the two regressions.

`map(means)` looks up the fitting-period mean for each product. `C(...)` tells
the formula library to treat a value as a category. `.copy()` prevents feature
creation from unexpectedly changing another DataFrame.

## 4. Append final fitting, evaluation, and uncertainty

Append to the **same file**:

```python
model, smear = fit(development, chosen_price)
transformed = features(development, chosen_price)
robust = model.get_robustcov_results(
    cov_type="hac-groupsum",
    time=development.time_index.to_numpy(dtype=int),
    maxlags=7, use_correction="hac", df_correction=False, use_t=False,
)
beta_index = model.model.exog_names.index("log_price")
low, high = robust.conf_int()[beta_index]
elasticity = {"estimate": float(model.params["log_price"]),
              "low": float(low), "high": float(high)}
assert np.isfinite(robust.conf_int()).all()

predictions = test[["stock_code", "sale_date", "units_sold"]].copy()
development_means = development.groupby("stock_code").units_sold.mean()
predictions["baseline"] = test.stock_code.map(development_means)
predictions["regression"] = predict(model, smear, test, chosen_price)
rows = []
for name in ["baseline", "regression"]:
    rows.append({"product": "all", "model": name,
                 **score(predictions.units_sold, predictions[name])})
    for code, group in predictions.groupby("stock_code"):
        rows.append({"product": code, "model": name,
                     **score(group.units_sold, group[name])})
evaluation = pd.DataFrame(rows)
print(evaluation)
print("Elasticity:", elasticity, "Smearing:", smear)
predictions.to_csv(WORK / "reports/predictions.csv", index=False)
evaluation.to_csv(WORK / "reports/metrics.csv", index=False)

diagnostics = development[["stock_code", "sale_date", "units_sold"]].copy()
diagnostics["log_residual"] = model.resid
diagnostics["log_fitted"] = model.fittedvalues
diagnostics["cooks_distance"] = model.get_influence().cooks_distance[0]
diagnostics.to_csv(WORK / "reports/diagnostics.csv", index=False)

for price in ["weighted_unit_price", "median_line_price"]:
    alternative, _ = fit(development, price)
    print("Price sensitivity:", price, alternative.params["log_price"])
thresholds = development.groupby("stock_code").units_sold.quantile(0.99)
trimmed = development[development.units_sold <= development.stock_code.map(thresholds)]
alternative, _ = fit(trimmed, chosen_price)
print("Without top 1% daily-unit observations:", alternative.params["log_price"])
```

Expected overall holdout MAE is about **103.29 units**, WAPE about **0.8068** for
both methods. The baseline is slightly better overall before rounding. Elasticity
is about **-1.85**, interval **[-2.22, -1.48]**. Median-price sensitivity is about
-1.20; trimming large development product-days gives about -1.75.

### Why the smearing factor?

Exponentiating an expected log outcome is not generally the same as predicting
the average outcome. Try residuals -1 and +1: their mean is zero, but the mean
of their exponentials is greater than one. The mean of `exp(fitting residual)`
is a simple retransformation correction. It is estimated only on the fitting
partition. A single global correction may still be inaccurate if variance
differs by product or price.

### What does the interval mean?

Observations from the same day can share demand shocks, and nearby days can be
related. Date-aggregated score HAC allows same-day cross-product dependence and
up to seven calendar-day lags. This is an advanced covariance estimator: you do
not need to derive it yet, but you must explain its purpose and assumptions.
It uses asymptotic normal critical values and does not address omitted confounders,
longer dependence, or choosing the specification from data. It is not a causal
interval. Never read it as a guarantee that price changes will have this effect.

### Inspect the diagnostics

Plot `log_fitted` against `log_residual`. Plot the mean residual by date.
Inspect the five largest Cook's distances and their transaction lines. Do not
delete them just because they make the model worse. Explain whether bulk orders,
calendar variation, or customer mix could contribute.

The trimming threshold above is learned from development data only; holdout
rows remain intact. Sensitivity checks are not a contest to find the most
attractive coefficient.

## 5. Append artifact saving and database provenance

Append to `learning/train.py`:

```python
run_id = str(uuid.uuid4())
directory = WORK / "artifacts" / run_id
directory.mkdir(parents=True)
model.save(str(directory / "regression.pickle"))
metadata = {
    "run_id": run_id, "price": chosen_price, "winner": winner,
    "formula": FORMULA, "smear": smear, "elasticity": elasticity,
    "training_start": str(development.sale_date.min().date()),
    "training_end": str(development.sale_date.max().date()),
    "validation_start": str(pd.Timestamp(validation_start).date()),
    "test_start": str(pd.Timestamp(test_start).date()),
    "weekdays": sorted(map(int, development.weekday.unique())),
    "price_support": development.groupby("stock_code")[chosen_price].agg(["min", "max"]).to_dict("index"),
}
(directory / "metadata.json").write_text(json.dumps(metadata, indent=2))
with connect() as connection:
    connection.execute("""
        CREATE TABLE IF NOT EXISTS learning_model_runs (
            run_id UUID PRIMARY KEY,
            created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
            metadata JSONB NOT NULL,
            metrics JSONB NOT NULL,
            model_directory TEXT NOT NULL
        )
    """)
    connection.execute("""
        INSERT INTO learning_model_runs (run_id, metadata, metrics, model_directory)
        VALUES (%s, %s, %s, %s)
    """, (run_id, Jsonb(metadata), Jsonb(evaluation.to_dict("records")), str(directory.relative_to(WORK))))
print("Saved run", run_id)
```

Run the completed script. Each invocation gets a new UUID and preserves a model.
The pickle is your own local fitted-model artifact; later lessons reload that
file. As an extension, save portable JSON coefficients and a data fingerprint;
compare that design with this simpler local artifact approach.

## Checkpoint

Write **`learning/reports/model_evaluation.md`**. Include split dates, baseline
definition, metrics, selected model, coefficient/interval, sensitivity, and
three limitations. State plainly whether regression improved prediction.

Explain: why not split randomly? Why is the baseline necessary? Why is price
still not an advance-known offered-price variable? Why can a negative coefficient
be precise and still fail to identify a causal effect?
