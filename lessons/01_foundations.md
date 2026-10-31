# Lesson 1 — Understand the question and prepare your workspace

## Goal

Explain what the project measures, then make a separate place to build it.

New to the tools? Complete [Docker setup](00_docker.md) and
[Git setup](00_git.md) first. Existing installations only need verification.

## 1. Work through the business question on paper

Imagine a product sells 10 units at a price of 2.00.

```text
Revenue = price × units = 2.00 × 10 = 20.00
```

Now imagine a price of 2.20 and 8 units: revenue is 17.60. A higher price does
not guarantee higher revenue. But observing those two sales situations does not
prove the price increase caused the drop. Perhaps one was a holiday or a stockout.

Write three explanations for lower units besides higher price.

Our real question is initially descriptive: **How do observed prices and sales
relate for selected UK products?** The causal question—what would happen if we
changed an offered price—requires stronger evidence.

## 2. Understand the system

```text
Excel transaction rows
    → Python loader
    → PostgreSQL raw table
    → SQL daily modeling view
    → pandas exploration
    → baseline + regression
    → historical pricing scenarios
    → Streamlit dashboard

Separate synthetic data → controlled causal experiment
```

- **Python** performs tasks and calculations.
- **Docker** runs PostgreSQL in a container.
- **PostgreSQL** stores tables and executes SQL.
- **A table** stores records; **a view** is a saved query, not a copied workbook.
- **pandas** holds an in-memory table for Python analysis.
- **Streamlit** turns Python analysis into a local interactive app.

Docker is not the database language. SQL is not a Python function. The dashboard
does not train a new model every time you move a slider.

## 3. Open the right terminal location

In VS Code choose **Terminal → New Terminal**. Run:

```sh
pwd
ls
```

The folder should be your standalone repository root; its contents should include
`compose.yaml`, `requirements.txt`, and the `Online Retail.xlsx` you supply separately.

If necessary:

```sh
cd ~/projects/retail-pricing-learning
```

If you have not completed the root README setup, create the environment first
with `python3 -m venv .venv`, configure `.env` from `.env.example`, then run:

```sh
mkdir -p learning/reports learning/artifacts
.venv/bin/python -m pip install -r requirements.txt
docker compose up -d --wait
```

Always use `.venv/bin/python` in this guide. That selects the environment with
the project packages instead of whichever Python happens to be on your PATH.

## 4. Create a separate learning database

Run this once in the terminal:

```sh
docker compose exec -T db psql -U retail -d postgres -v ON_ERROR_STOP=1 -c "CREATE DATABASE retail_learning;"
```

If it says the database already exists, keep it and continue. Do not drop it.
Compose initializes a database named `retail`; your lessons use
`retail_learning`.

Connect interactively:

```sh
docker compose exec db psql -U retail -d retail_learning
```

Inside the `psql` prompt, type SQL:

```sql
SELECT current_database();
SELECT 2.20 * 8 AS revenue;

SELECT product, SUM(quantity) AS units
FROM (VALUES ('bag', 2), ('bag', 3), ('cup', 1)) AS purchases(product, quantity)
GROUP BY product
ORDER BY units DESC;
```

Expected: database `retail_learning`; revenue 17.60; bag=5 and cup=1.

Type `\q` to exit. Terminal commands will not work at the SQL prompt, and SQL
will not work directly at the terminal prompt.

## 5. Write your database helper

Create **`learning/db.py`** inside this repository:

```python
from pathlib import Path
import os

import pandas as pd
import psycopg

ROOT = Path(__file__).resolve().parents[1]
WORK = ROOT / "learning"


def connect():
    config = {}
    for line in (ROOT / ".env").read_text().splitlines():
        if line.strip() and not line.lstrip().startswith("#"):
            key, value = line.split("=", 1)
            config[key.strip()] = value.strip()
    config.update(os.environ)
    return psycopg.connect(
        host="127.0.0.1",
        port=config.get("POSTGRES_PORT", "5434"),
        dbname="retail_learning",
        user="retail",
        password=config["POSTGRES_PASSWORD"],
    )


def fetch(sql):
    with connect() as connection:
        cursor = connection.execute(sql)
        return pd.DataFrame(
            cursor.fetchall(),
            columns=[column.name for column in cursor.description],
        )


if __name__ == "__main__":
    print(fetch("SELECT current_database() AS database"))
```

Run:

```sh
.venv/bin/python learning/db.py
```

Expected: one row containing `retail_learning`. Do not proceed if it says `retail`.

`Path(__file__)` locates this script regardless of terminal location.
`config` is a dictionary. Environment settings override the `.env` values.
`with connect()` commits on success, rolls back on failure, and closes the
connection. `fetch()` turns returned database rows into a pandas DataFrame.
Never print the configuration dictionary: it contains a password.

## Checkpoint

Without looking back, explain:

1. Where does persistent data live?
2. Why use a different database for practice?
3. How do terminal commands differ from SQL statements?
4. Why does `learning/db.py` use a fixed database name?
5. Why does correlation between price and sales not settle the business question?

**Deliverable:** `learning/db.py` and your first entry in `learning/notes.md`.
