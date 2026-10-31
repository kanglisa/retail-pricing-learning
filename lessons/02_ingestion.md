# Lesson 2 — Load Excel into PostgreSQL yourself

## Goal

Understand the workbook's grain, choose column types, and load it repeatably.

## 1. Inspect a few rows before designing anything

Create **`learning/inspect_workbook.py`**:

```python
from itertools import islice
import openpyxl
from db import ROOT

book = openpyxl.load_workbook(
    ROOT / "Online Retail.xlsx", read_only=True, data_only=True
)
try:
    sheet = book.active
    for row in islice(sheet.iter_rows(values_only=True), 6):
        print(row)
finally:
    book.close()
```

```sh
.venv/bin/python learning/inspect_workbook.py
```

The header should be:

```text
InvoiceNo, StockCode, Description, Quantity, InvoiceDate, UnitPrice, CustomerID, Country
```

One row is an **invoice line**, not one invoice, one customer, or one day.
A customer can have many invoices; an invoice can contain many products; a
product can appear on many invoice lines. Product codes and invoice identifiers
are labels, so store them as text even when they look numeric.

`read_only=True` streams worksheet rows instead of loading the entire workbook
into an editable structure. `data_only=True` reads cached cell values rather
than formula text. This does not calculate new Excel formulas.

## 2. Design the raw table

Create **`learning/schema.sql`**:

```sql
CREATE TABLE IF NOT EXISTS retail_transactions (
    source_file TEXT NOT NULL,
    source_row INTEGER NOT NULL,
    invoice_no TEXT NOT NULL,
    stock_code TEXT NOT NULL,
    description TEXT,
    quantity INTEGER NOT NULL,
    invoice_date TIMESTAMP NOT NULL,
    unit_price NUMERIC NOT NULL,
    customer_id TEXT,
    country TEXT NOT NULL,
    PRIMARY KEY (source_file, source_row)
);
```

Why these decisions?

- Source filename plus row number identifies an imported record without guessing
  whether repeated business fields are accidental duplicates.
- `NUMERIC` stores decimal monetary values; `INTEGER` stores unit counts.
- Customer ID and description can be missing, so they are nullable.
- Negative quantities/prices are preserved for auditing, not rejected at ingestion.
- The timestamp has no timezone because the workbook provides no timezone metadata.

Do not make `(invoice_no, stock_code)` unique: repeated product lines can exist.

## 3. Write the loader in small pieces

Create **`learning/ingest.py`**:

```python
import openpyxl
from db import ROOT, WORK, connect

EXPECTED = (
    "InvoiceNo", "StockCode", "Description", "Quantity", "InvoiceDate",
    "UnitPrice", "CustomerID", "Country",
)


def identifier(value):
    if isinstance(value, float) and value.is_integer():
        return str(int(value))
    return str(value)


def main():
    filename = "Online Retail.xlsx"
    book = openpyxl.load_workbook(
        ROOT / filename, read_only=True, data_only=True
    )
    try:
        rows = book.active.iter_rows(values_only=True)
        if tuple(next(rows)) != EXPECTED:
            raise ValueError("Unexpected workbook columns")

        with connect() as connection:
            connection.execute((WORK / "schema.sql").read_text())
            connection.execute(
                "DELETE FROM retail_transactions WHERE source_file = %s",
                (filename,),
            )
            count = 0
            with connection.cursor().copy(
                "COPY retail_transactions FROM STDIN"
            ) as copy:
                for row_number, row in enumerate(rows, start=2):
                    invoice, stock, description, quantity, when, price, customer, country = row
                    if any(value is None for value in (invoice, stock, quantity, when, price, country)):
                        raise ValueError(f"Missing required value on row {row_number}")
                    if int(quantity) != quantity:
                        raise ValueError(f"Noninteger quantity on row {row_number}")
                    copy.write_row((
                        filename, row_number, identifier(invoice), identifier(stock),
                        description, int(quantity), when, price,
                        None if customer is None else identifier(customer), country,
                    ))
                    count += 1
        print(f"Committed {count:,} rows")
    finally:
        book.close()


if __name__ == "__main__":
    main()
```

Before running, explain the `enumerate`, tuple unpacking, `None`, `with`, and
`try/finally` statements. The header is Excel row 1, so data starts at row 2.
`%s` is a database parameter placeholder, not string formatting. `(filename,)`
is a one-item tuple. Parameters let the driver handle values safely and correctly.

`COPY` streams rows efficiently. The delete and load share one transaction:
if loading fails, the prior dataset is restored by rollback. Reusing the same
filename replaces that input's rows. Renaming the same workbook and importing
it as a new source would require a separate policy to avoid double counting.

## 4. Run it, verify, then run it again

```sh
.venv/bin/python learning/ingest.py
docker compose exec -T db psql -U retail -d retail_learning -c "SELECT COUNT(*) FROM retail_transactions;"
.venv/bin/python learning/ingest.py
docker compose exec -T db psql -U retail -d retail_learning -c "SELECT COUNT(*) FROM retail_transactions;"
```

Expected: **541,909** after each run, not 1,083,818. The first full workbook
scan may take a little time.

Inspect records:

```sh
docker compose exec db psql -U retail -d retail_learning
```

```sql
SELECT * FROM retail_transactions ORDER BY source_row LIMIT 5;
SELECT MIN(invoice_date), MAX(invoice_date) FROM retail_transactions;
SELECT COUNT(*) FROM retail_transactions WHERE customer_id IS NULL;
```

Expected: December 1, 2010 through December 9, 2011; 135,080 missing customer IDs.
Exit with `\q`.

## Exercise

On paper, trace a transaction that loads three rows and fails on row four.
Which rows should remain afterward? Why is committing after every row a bad
choice for this replacement strategy?

## Checkpoint

Explain how the loader handles a cancellation, missing customer ID, repeated
product line, and unexpected column header. Explain why successful execution
alone does not prove the data is correct.

**Deliverable:** your schema, loader, and recorded rerun counts. Review your work against the checkpoint before moving on; this repository does
not ship a prebuilt solution that you need to run.
