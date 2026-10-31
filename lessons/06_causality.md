# Lesson 6 — Create a world where you know the true effect

## Goal

See a confounded regression get the wrong answer, then understand why adjustment
or randomization can recover the effect in a carefully constructed simulation.

## 1. Start with the causal story

Suppose strong demand leads a retailer to charge more. Strong demand also leads
people to buy more:

```text
demand driver → price → units
       └────────────→ units
```

The second path mixes the demand driver's effect into the price association.
Product and weekday controls do not automatically close that path.

We will set the true elasticity to **-1.5**. The observational retailer increases
price when the simulated demand driver is high. In the randomized world, prices
are assigned independently of that driver.

## 2. Write a transparent simulator

Create **`learning/causal.py`**:

```python
import numpy as np
import pandas as pd
import statsmodels.formula.api as smf
from db import WORK

TRUE_BETA = -1.5


def generate(seed, n=730, noise_sd=0.25):
    rng = np.random.default_rng(seed)
    day = np.arange(n)
    driver = rng.normal(size=n)
    price_noise = rng.normal(size=n)
    randomized_noise = rng.normal(size=n)
    noise = rng.normal(scale=noise_sd, size=n)
    weekday = day % 7 + 1
    trend = day / 365
    base = 4.5 + 0.8*driver + 0.15*(weekday >= 6) + 0.1*trend + noise
    observed_price = np.log(2) + 0.25*driver + 0.15*price_noise
    assigned_price = np.log(2) + np.sqrt(0.25**2+0.15**2)*randomized_noise

    def frame(log_price):
        return pd.DataFrame({"log_price": log_price,
            "log_units": base + TRUE_BETA*log_price,
            "demand_driver": driver, "weekday": weekday, "trend": trend})
    return frame(observed_price), frame(assigned_price)


def estimate(data, adjusted=False):
    formula = "log_units ~ log_price + C(weekday) + trend"
    if adjusted:
        formula += " + demand_driver"
    model = smf.ols(formula, data).fit(cov_type="HC3", use_t=False)
    low, high = model.conf_int().loc["log_price"]
    return {"estimate": float(model.params["log_price"]),
            "low": float(low), "high": float(high),
            "covered": bool(low <= TRUE_BETA <= high)}


if __name__ == "__main__":
    results = []
    for seed in range(20261030, 20261030+200):
        observed, randomized = generate(seed)
        for name, sample, adjusted in [
            ("naive", observed, False),
            ("adjusted", observed, True),
            ("randomized", randomized, False),
        ]:
            results.append({"seed": seed, "method": name,
                            **estimate(sample, adjusted)})
    results = pd.DataFrame(results)
    results.to_csv(WORK / "reports/causal_estimates.csv", index=False)
    summary = []
    for method, group in results.groupby("method"):
        error = group.estimate - TRUE_BETA
        summary.append({"method": method, "mean_estimate": group.estimate.mean(),
            "bias": error.mean(), "rmse": np.sqrt((error**2).mean()),
            "coverage": group.covered.mean()})
    summary = pd.DataFrame(summary)
    summary.to_csv(WORK / "reports/causal_summary.csv", index=False)
    print(summary)
```

Run it:

```sh
.venv/bin/python learning/causal.py
```

Expected approximate averages: naive **+0.855**, adjusted **-1.499**, randomized
**-1.508**. Coverage is about 0%, 93%, and 95%, respectively, over these seeds.
Coverage itself is a noisy estimate; 200 repetitions do not guarantee exactly
95% for a valid method. These are repeated independently generated datasets,
not 200 regressions on the real workbook.

## 3. Explain the wrong sign

Without the demand driver in the model, price partly acts as a proxy for strong
demand. The positive confounding contribution is large enough to reverse the sign.
More data can make the biased estimate more precise; it does not remove the bias.

Adjusted regression works here because we deliberately observed the full driver
without error and generated the outcome using the specified equation. Real data
does not promise that every confounder is available or correctly modeled.

Randomization breaks the price-driver relationship in expectation. Finite samples
can still have imbalance. This simplified experiment assumes no carryover, no
spillovers between units, and no noncompliance with assigned prices.

The synthetic units are positive continuous quantities, not rounded retail counts.
Daily shocks are independent, so HC3 covariance is appropriate for this exercise;
the retail panel needed a temporal-dependence treatment.

## 4. Experiments you must do yourself

1. Change the confounding coefficient `0.25` to zero. Predict the naive result
   before rerunning, then explain the result.
2. Increase outcome noise. What changes: bias, interval width, or both?
3. Increase repetitions from 200 to 500. Distinguish more simulated experiments
   from more observations within each experiment.
4. Add the driver to the randomized regression. Does precision improve?
5. Set `noise_sd=0` and use adjusted regression. Verify beta is recovered within
   numerical tolerance: `abs(estimate - TRUE_BETA) < 1e-10`.
6. Plot a boxplot of estimates by method with a horizontal line at -1.5.

Restore the original parameters before the final rebuild.

## Checkpoint

Write **`learning/reports/causality.md`** explaining correlation, intervention,
confounding, bias, variance, interval coverage, and why the simulation does not
turn the retail coefficient into a causal estimate. Describe what measurements
a real experiment would need: assigned offered prices, exposure, availability,
and outcomes including opportunities without a purchase.
