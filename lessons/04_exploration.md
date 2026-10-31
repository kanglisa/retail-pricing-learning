# Lesson 4 — Explore before modeling

## Goal

Make plots that expose data behavior, not just decorate a report.

Start with one product (`20725`), then repeat for the other five. Before every
plot, write the question it should answer. Create **`learning/explore.py`**:

```python
import matplotlib
matplotlib.use("Agg")
import matplotlib.pyplot as plt
import pandas as pd
from db import WORK, fetch

data = fetch("SELECT * FROM modeling_daily_sales ORDER BY sale_date, stock_code")
data["sale_date"] = pd.to_datetime(data["sale_date"])
for column in ["units_sold", "revenue", "weighted_unit_price",
               "median_line_price", "identified_customer_price"]:
    data[column] = data[column].astype(float)

assert not data.duplicated(["stock_code", "country", "sale_date"]).any()
assert data.units_sold.gt(0).all()
data.to_csv(WORK / "reports/daily.csv", index=False)

transactions = fetch("SELECT * FROM pricing_transactions ORDER BY invoice_date")
transactions["invoice_date"] = pd.to_datetime(transactions["invoice_date"])
transactions["unit_price"] = transactions["unit_price"].astype(float)

for code, product in data.groupby("stock_code"):
    product = product.set_index("sale_date")
    weekly = product[["units_sold", "revenue", "invoice_count"]].resample("W-SUN").sum()
    figure, axes = plt.subplots(3, 1, figsize=(10, 8), constrained_layout=True)
    for axis, column in zip(axes, weekly.columns):
        axis.plot(weekly.index, weekly[column])
        axis.set_ylabel(column)
    figure.suptitle(f"{code}: observed weekly sales")
    figure.savefig(WORK / f"reports/{code}_weekly.png", dpi=120)
    plt.close(figure)

    prices = product[["weighted_unit_price", "median_line_price",
                      "identified_customer_price"]]
    axis = prices.plot(figsize=(10, 4), title=f"{code}: daily transaction prices")
    axis.set_ylabel("Price in workbook units")
    axis.figure.tight_layout()
    axis.figure.savefig(WORK / f"reports/{code}_prices.png", dpi=120)
    plt.close(axis.figure)

    lines = transactions[transactions.stock_code == code]
    figure, axes = plt.subplots(1, 2, figsize=(10, 4), constrained_layout=True)
    axes[0].hist(lines.unit_price, bins=30)
    axes[0].set(xlabel="Transaction price", ylabel="Line count")
    axes[1].scatter(lines.unit_price, lines.quantity, alpha=0.25, s=8)
    axes[1].set_yscale("log")
    axes[1].set(xlabel="Transaction price", ylabel="Quantity per line (log scale)")
    figure.suptitle(code)
    figure.savefig(WORK / f"reports/{code}_purchase_mix.png", dpi=120)
    plt.close(figure)

print(data.groupby("stock_code").agg(
    observed_days=("sale_date", "size"), units=("units_sold", "sum")
))
```

Run:

```sh
.venv/bin/python learning/explore.py
```

Open PNGs in VS Code. `Agg` generates image files without opening GUI windows.
SQL `NUMERIC` values may arrive as Python decimals; converting analytical columns
to floats makes them usable by numerical libraries. Keep exact money accounting
in SQL when required; these plots are approximate analytical calculations.

`set_index` puts dates on the index. `resample('W-SUN')` forms weeks ending Sunday.
Boundary weeks can be partial. A missing weekly bucket in this aggregation is
not evidence that inventory was available and demand was zero. Invoice counts
are for one product; summing across products would double-count some invoices.

## Work through these observations yourself

1. Do prices change gradually, in steps, or in spikes?
2. Is the identified-customer price steadier than the all-purchase price?
3. Do large orders occur at different prices than small orders?
4. Does one unusual line dominate a week?
5. Would a median line price remove all customer-mix effects? Why not?

For `20725`, full-window unit-weighted price should be about 1.931 across all
purchases and 1.594 for identified customers. Reproduce these yourself:

```python
lines = transactions[transactions.stock_code == "20725"]
identified = lines[lines.customer_id.notna()]
for name, group in [("all", lines), ("identified", identified)]:
    weighted = (group.quantity * group.unit_price).sum() / group.quantity.sum()
    print(name, weighted)
```

Add that snippet to your script. This is a ratio of whole-window totals, not an
unweighted average of daily prices. A lower identified-customer average does
not prove that anonymous purchases are bad data or reveal a specific channel.

## Independent exercises

- Calculate the fraction of days with more than one price for each product.
  Expected range: 57.6%–76.1% across the six products.
- For popcorn holders (`22197`), calculate what fraction of units come from
  transaction lines with at least 500 units. Expected: about 38.2%.
- Plot units with and without those lines, labeling both honestly. Do not delete
  raw records or choose the version that looks most impressive.
- Inspect possible return matches using customer, product, price, quantity, and
  later time. Explain why a match is a flag, not confirmed invoice linkage.
- Write a feature dictionary: which columns could be known before sales and
  which exist only after customers buy? Revenue and invoice counts are outcomes,
  not legitimate advance forecasting inputs.

## Checkpoint

Produce **`learning/reports/exploration.md`** with three plots, three concrete
observations, two possible explanations per observation, and two limitations.
Replace “price caused demand to decrease” with language your data supports.

**Before proceeding:** explain how a weighted price can change while every
available price tier stays fixed. Compare your results with the expected values in this lesson only after writing
your own interpretation.
