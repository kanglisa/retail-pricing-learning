# Lesson 7 — Build bounded scenarios and your own dashboard

## Goal

Turn a saved model into a transparent exploratory tool without pretending it
has proved an optimal price.

## 1. Derive the optimizer on paper

At fixed product and calendar inputs, constant-elasticity demand has the form
`units = A × price^beta`. Therefore:

```text
revenue = price × units = A × price^(1+beta)
```

If beta < -1, this curve favors lower prices. If beta > -1, it favors higher
prices. At -1, revenue is flat. An optimizer choosing the boundary is a consequence
of this assumption, not proof that the boundary is a business optimum.

We will evaluate only prices inside both the product's development range and
±10% of a reference price. The reference is the latest observed daily price at
or before a chosen historical date. Default to the training cutoff so the time
trend is not extrapolated into a future period.

Because the baseline won validation, we will save `unsupported` results with
no recommended price. A curve can still illustrate the fitted model's behavior.

## 2. Write a reusable scenario module

Create **`learning/scenarios.py`**:

```python
from datetime import date
from decimal import Decimal
import json
import uuid
import numpy as np
import pandas as pd
import statsmodels.api as sm
from psycopg.types.json import Jsonb
from db import WORK, connect, fetch


def load_run():
    runs = fetch("SELECT * FROM learning_model_runs ORDER BY created_at DESC LIMIT 1")
    if runs.empty:
        raise ValueError("Train a model first")
    run = runs.iloc[0]
    directory = WORK / run.model_directory
    metadata = json.loads((directory / "metadata.json").read_text())
    model = sm.load(str(directory / "regression.pickle"))
    return model, metadata, run.metrics


def daily_data():
    data = fetch("SELECT * FROM modeling_daily_sales ORDER BY sale_date, stock_code")
    data["sale_date"] = pd.to_datetime(data.sale_date)
    for column in ["units_sold", "revenue", "weighted_unit_price", "median_line_price"]:
        data[column] = data[column].astype(float)
    return data


def make_scenario(model, metadata, data, code, day, change=0.10):
    if not 0 <= change <= 0.10:
        raise ValueError("Maximum price change must be between 0 and 0.10")
    result = {"product": code, "date": str(day), "status": "unsupported",
              "recommended_price": None, "reasons": [], "grid": []}
    if code not in metadata["price_support"]:
        result["reasons"].append("Unknown product")
        return result
    if not date.fromisoformat(metadata["training_start"]) <= day <= date.fromisoformat(metadata["training_end"]):
        result["reasons"].append("Date outside training window")
        return result
    if day.isoweekday() not in metadata["weekdays"]:
        result["reasons"].append("Weekday not observed during fitting")
        return result
    history = data[(data.stock_code == code) & (data.sale_date.dt.date <= day)]
    if history.empty:
        result["reasons"].append("No reference price yet")
        return result
    reference = history.iloc[-1]
    price = float(reference[metadata["price"]])
    support = metadata["price_support"][code]
    if not support["min"] <= price <= support["max"]:
        result["reasons"].append("Reference price outside development support")
        return result
    lower = max(support["min"], price*(1-change))
    upper = min(support["max"], price*(1+change))
    prices = sorted(set(np.linspace(lower, upper, 101).tolist() + [price]))

    def project(candidate):
        inputs = pd.DataFrame({"stock_code": [code], "weekday": [day.isoweekday()],
            "trend_years": [(day-date(2010, 12, 1)).days/365],
            "log_price": [np.log(candidate)]})
        units = float(np.exp(model.predict(inputs).iloc[0])*metadata["smear"])
        return units, candidate*units

    reference_units, reference_revenue = project(price)
    beta = metadata["elasticity"]
    for candidate in prices:
        units, revenue = project(candidate)
        interval = [(candidate/price)**(1+b)-1 for b in [beta["low"], beta["high"]]]
        result["grid"].append({"price": candidate, "units": units, "revenue": revenue,
            "relative_change": revenue/reference_revenue-1,
            "slope_only_low": min(interval), "slope_only_high": max(interval)})
    if metadata["winner"] == "baseline":
        result["reasons"].append("Regression did not win validation")
    if beta["high"] >= 0:
        result["reasons"].append("Negative price association not established")
    result["status"] = "unsupported" if result["reasons"] else "scenario_only"
    result["reasons"].append("Observational association is not causal")
    result.update(reference_price=price, reference_date=str(reference.sale_date.date()),
        reference_units=reference_units, reference_revenue=reference_revenue,
        lower=lower, upper=upper, max_change=change,
        best=max(result["grid"], key=lambda row: row["revenue"]))
    assert all(lower <= row["price"] <= upper for row in result["grid"])
    return result


def save_scenario(metadata, result):
    scenario_id = str(uuid.uuid4())
    with connect() as connection:
        connection.execute("""
            CREATE TABLE IF NOT EXISTS learning_price_scenarios (
                scenario_id UUID PRIMARY KEY,
                model_run_id UUID NOT NULL REFERENCES learning_model_runs(run_id),
                created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
                product TEXT NOT NULL,
                scenario_date DATE NOT NULL,
                reference_price NUMERIC,
                status TEXT NOT NULL,
                recommended_price NUMERIC CHECK (recommended_price IS NULL),
                results JSONB NOT NULL
            )
        """)
        reference = result.get("reference_price")
        connection.execute("""
            INSERT INTO learning_price_scenarios
            (scenario_id, model_run_id, product, scenario_date, reference_price,
             status, recommended_price, results)
            VALUES (%s,%s,%s,%s,%s,%s,%s,%s)
        """, (scenario_id, metadata["run_id"], result["product"], result["date"],
              None if reference is None else Decimal(str(reference)),
              result["status"], None, Jsonb(result)))


if __name__ == "__main__":
    model, metadata, _ = load_run()
    data = daily_data()
    day = date.fromisoformat(metadata["training_end"])
    for code in metadata["price_support"]:
        result = make_scenario(model, metadata, data, code, day)
        save_scenario(metadata, result)
        print(code, result["status"], result.get("best"), result["reasons"])
```

Run:

```sh
.venv/bin/python learning/scenarios.py
```

Expected: six unsupported results and no recommended prices. `best` means
the mathematical maximum within an illustrative grid, not a recommendation.
`Decimal(str(reference))` preserves the same decimal text in PostgreSQL NUMERIC
and JSON; conversion through a database float can otherwise round endpoints.

The slope-only relative revenue interval is `(candidate/reference)^(1+beta)-1`
evaluated at the two coefficient interval endpoints. It holds other parameters
fixed and is **not** a full predictive interval. Min/max price support is only
a coarse check: some interior prices may be rarely observed. Check the reference
date too; an old reference can be stale.

This guided version assumes the modeling view has not changed since training.
Retrain after changing that view. Detecting changes using a data fingerprint is an extension exercise.

## 3. Build your dashboard

Create **`learning/app.py`**:

```python
from datetime import date
from pathlib import Path
import sys
import pandas as pd
import streamlit as st
sys.path.insert(0, str(Path(__file__).resolve().parent))
from db import WORK
from scenarios import load_run, daily_data, make_scenario

st.set_page_config(page_title="My Retail Pricing Explorer", layout="wide")
st.title("My Retail Pricing Explorer")
st.write("Historical sales analysis and conditional price scenarios")
model, metadata, metrics = load_run()
data = daily_data()
code = st.sidebar.selectbox("Product", sorted(data.stock_code.unique()))
product = data[data.stock_code == code].set_index("sale_date")
st.caption(f"Model {metadata['run_id']}; fitted through {metadata['training_end']}")
st.warning("Observed associations do not prove causal price effects. No validated revenue gain is claimed.")

history, evaluation, pricing, causal = st.tabs(["History", "Evaluation", "Scenarios", "Causal simulation"])
with history:
    st.subheader("Weekly observed units")
    st.line_chart(product[["units_sold"]].resample("W-SUN").sum())
    st.subheader("Daily transaction prices")
    st.line_chart(product[["weighted_unit_price", "median_line_price"]])
    st.caption("Missing product-days have unknown availability, not proven zero demand. Boundary weeks may be partial.")
with evaluation:
    st.write("Validation winner:", metadata["winner"])
    st.write("Elasticity and 95% interval:", metadata["elasticity"])
    st.dataframe(pd.DataFrame(metrics))
    st.caption("WAPE is a fraction. Evaluation is conditional on observed selling days and transaction prices.")
with pricing:
    day = st.date_input("Historical scenario date",
        value=date.fromisoformat(metadata["training_end"]),
        min_value=date.fromisoformat(metadata["training_start"]),
        max_value=date.fromisoformat(metadata["training_end"]))
    change = st.slider("Maximum price change (%)", 0, 10, 10)
    result = make_scenario(model, metadata, data, code, day, change/100)
    st.write("Status:", result["status"])
    st.write("Reasons:", result["reasons"])
    st.write("Recommended price: unavailable")
    if result["grid"]:
        st.write("Reference price/date:", result["reference_price"], result["reference_date"])
        curve = pd.DataFrame(result["grid"])
        st.line_chart(curve.set_index("price")[["revenue"]])
        st.write("Exploratory grid maximum:", result["best"])
        st.caption("Slope-only ranges are not full prediction intervals. Boundary maxima reflect the model assumptions.")
with causal:
    st.write("Synthetic experiment only; true elasticity = -1.5")
    path = WORK / "reports/causal_summary.csv"
    if path.exists():
        st.dataframe(pd.read_csv(path))
    else:
        st.info("Run learning/causal.py first")
```

Start the learning dashboard on port 8502:

```sh
.venv/bin/python -m streamlit run learning/app.py --server.port=8502 --server.address=127.0.0.1 --browser.gatherUsageStats=false
```

Open **http://localhost:8502**. Streamlit reruns the script when controls change.
The app calls the pure calculation function, not `save_scenario`, so moving a
slider does not write database rows or alter prices. The `if __name__ == '__main__'`
guard keeps the batch-saving block from running when the app imports the module.

The `sys.path` line makes the sibling learning modules importable when Streamlit
executes the app. It does not alter your Python installation.

## 4. Test behaviors, not just appearance

- Select all six products. Do units and price curves change?
- Set the slider to zero. The only unique candidate should be the reference price,
  with relative revenue change zero.
- Select a Saturday missing from fitting data. Expect an unsupported result with
  no curve, not a fabricated weekday effect.
- Confirm the CLI stores rows but UI interactions do not.
- Change the allowed range from 10% to 5%. Verify every candidate is inside both
  the new bound and development support.
- Add a check for an unknown product and a date outside the training window.

## Checkpoint

Demonstrate the dashboard to someone who has not read the code. Explain the
difference between an exploratory grid maximum and a recommendation, and show
where the model run, reference date, constraints, and reasons are recorded.

**Deliverable:** your scenario module, dashboard, and a screenshot with a short
caption describing one result accurately.
