# Lesson 3 — Audit the data and build a daily dataset

## Goal

Decide which rows answer your question and explain every exclusion.

Open `psql` against **retail_learning** as in the previous lesson. Execute the
queries below one at a time. Save your queries as `learning/audit.sql` and your
interpretations as `learning/reports/audit.md`.

## 1. Count first, then inspect

```sql
SELECT country, COUNT(*) AS rows
FROM retail_transactions
GROUP BY country ORDER BY rows DESC LIMIT 10;

SELECT COUNT(*) AS raw_rows,
       COUNT(*) FILTER (WHERE customer_id IS NULL) AS missing_customer,
       COUNT(*) FILTER (WHERE invoice_no ILIKE 'C%') AS cancellations,
       COUNT(*) FILTER (WHERE quantity <= 0) AS nonpositive_quantity,
       COUNT(*) FILTER (WHERE unit_price <= 0) AS nonpositive_price
FROM retail_transactions;
```

`GROUP BY` defines which rows are summarized together. `FILTER` lets each count
have a different condition. `ILIKE 'C%'` matches an invoice beginning with C,
case-insensitively. In SQL, test missingness with `IS NULL`, not `= NULL`.

Expected full-extract counts: cancellations=9,288; nonpositive quantity=10,624;
nonpositive price=2,517. These overlap. A canceled line may have negative quantity,
so adding those counts would double count some records.

## 2. Reconcile included and excluded rows

```sql
SELECT
    COUNT(*) FILTER (WHERE quantity > 0 AND unit_price > 0
                    AND invoice_no NOT ILIKE 'C%') AS included,
    COUNT(*) FILTER (WHERE NOT (quantity > 0 AND unit_price > 0
                    AND invoice_no NOT ILIKE 'C%')) AS excluded
FROM retail_transactions;
```

Expected: **530,104 + 11,805 = 541,909**. This reconciliation is valid because
these fields are non-null in the schema. Included sales are gross purchases,
not net sales after returns. Do not call the total profit: product costs are absent.

## 3. Investigate duplicates and unusual codes

```sql
SELECT invoice_no, stock_code, COUNT(*) AS lines,
       COUNT(DISTINCT unit_price) AS prices
FROM retail_transactions
GROUP BY invoice_no, stock_code
HAVING COUNT(*) > 1
ORDER BY lines DESC LIMIT 10;

SELECT stock_code, MIN(description) AS example_description, COUNT(*) AS rows
FROM retail_transactions
WHERE stock_code !~ '^[0-9]{5}[A-Za-z]*$'
GROUP BY stock_code ORDER BY rows DESC;

SELECT stock_code, description, quantity, unit_price, invoice_no
FROM retail_transactions
ORDER BY ABS(quantity) DESC LIMIT 10;
```

`HAVING` filters groups after aggregation; `WHERE` filters input rows before it.
The regular expression is a screening heuristic, not an authoritative product
dictionary. It identifies codes worth reviewing; some unusual codes are real
merchandise. Do not delete them blindly. The final scope explicitly selects six
reviewed products instead.

**Exercise:** find exact duplicates by grouping on all eight business fields.
Then sum `count - 1` over groups with more than one row. Expected: 5,268 extra
identical rows. Explain why this still does not prove they should be deleted.

## 4. Choose the date window

```sql
SELECT date_trunc('month', invoice_date)::date AS month,
       MIN(invoice_date)::date AS first_day,
       MAX(invoice_date)::date AS last_day,
       COUNT(*) AS rows
FROM retail_transactions GROUP BY 1 ORDER BY 1;
```

December 2011 ends on December 9. Use `>= '2010-12-01'` and `< '2011-12-01'`
to avoid comparing that partial month with full calendar months. Earlier windows
may still include closures, holidays, and missing availability; complete calendar
boundaries do not prove every product was available every day.

## 5. Understand weighted price by hand

For two lines with quantities 1 and 9, and prices 4 and 2:

```text
Units = 1 + 9 = 10
Revenue = 1×4 + 9×2 = 22
Unit-weighted price = 22/10 = 2.20
Mean line price = (4+2)/2 = 3.00
```

Both calculations are valid; they answer different questions. A changing mix of
bulk and small orders changes weighted price even if neither transaction price
changes. This matters when interpreting a regression.

## 6. Build your scoped views

Create **`learning/views.sql`**:

```sql
CREATE OR REPLACE VIEW pricing_transactions AS
SELECT * FROM retail_transactions
WHERE country = 'United Kingdom'
  AND invoice_date >= DATE '2010-12-01'
  AND invoice_date < DATE '2011-12-01'
  AND quantity > 0 AND unit_price > 0
  AND invoice_no NOT ILIKE 'C%'
  AND stock_code IN ('85099B','85123A','22197','21212','20725','47566');

CREATE OR REPLACE VIEW modeling_daily_sales AS
SELECT stock_code, country, invoice_date::date AS sale_date,
       EXTRACT(ISODOW FROM invoice_date::date)::integer AS weekday,
       invoice_date::date - DATE '2010-12-01' AS time_index,
       SUM(quantity) AS units_sold,
       SUM(quantity * unit_price) AS revenue,
       SUM(quantity * unit_price) / SUM(quantity) AS weighted_unit_price,
       percentile_cont(0.5) WITHIN GROUP (ORDER BY unit_price) AS median_line_price,
       COUNT(DISTINCT unit_price) AS distinct_prices,
       COUNT(*) AS sales_lines,
       COUNT(DISTINCT invoice_no) AS invoice_count,
       COUNT(*) FILTER (WHERE customer_id IS NULL) AS missing_customer_lines,
       SUM(quantity * unit_price) FILTER (WHERE customer_id IS NOT NULL)
         / NULLIF(SUM(quantity) FILTER (WHERE customer_id IS NOT NULL), 0)
         AS identified_customer_price,
       MAX(quantity) AS max_line_quantity
FROM pricing_transactions
GROUP BY stock_code, country, invoice_date::date;
```

Exit `psql`, then run from the terminal:

```sh
docker compose exec -T db psql -U retail -d retail_learning -v ON_ERROR_STOP=1 -f - < learning/views.sql
```

`::date` removes time of day. ISO weekday is Monday=1 through Sunday=7.
`NULLIF(..., 0)` avoids division by zero and returns NULL for an unavailable
identified-customer price. A view recomputes its query from current tables.

## 7. Verify the grain and totals

Inside `psql`:

```sql
SELECT COUNT(*) FROM pricing_transactions;
SELECT COUNT(*) FROM modeling_daily_sales;

SELECT stock_code, country, sale_date, COUNT(*)
FROM modeling_daily_sales
GROUP BY stock_code, country, sale_date HAVING COUNT(*) > 1;

SELECT (SELECT SUM(quantity) FROM pricing_transactions) AS raw_units,
       (SELECT SUM(units_sold) FROM modeling_daily_sales) AS daily_units;
```

Expected: **9,394 transaction lines**, **1,729 product-days**, no duplicate daily
keys, and equal unit totals. Grain means exactly what one row represents.

The six products were chosen for observed sales coverage and recurring price
variation. Document your own inclusion rationale in the audit. Selecting
them with full-period information makes later holdout results exploratory.

## Checkpoint

Write a one-page data contract: key, date window, market, inclusion rules,
target, price measure, known missing data, and exclusions. Explain why an absent
product/day is **unknown**, not automatically zero demand. Explain the difference
between raw rows, invoices, and product-days.

**Deliverable:** `audit.sql`, `views.sql`, and `reports/audit.md` under `learning/`.
