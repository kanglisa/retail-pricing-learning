# Lesson 8 — Prove you can rebuild and explain it

## 1. Rebuild from your own files

Stop the learning app with Ctrl+C. From the project root:

```sh
docker compose up -d --wait
.venv/bin/python learning/db.py
.venv/bin/python learning/ingest.py
docker compose exec -T db psql -U retail -d retail_learning -v ON_ERROR_STOP=1 -f - < learning/views.sql
.venv/bin/python learning/explore.py
.venv/bin/python learning/train.py
.venv/bin/python learning/causal.py
.venv/bin/python learning/scenarios.py
.venv/bin/python -m streamlit run learning/app.py --server.port=8502 --server.address=127.0.0.1 --browser.gatherUsageStats=false
```

Expected checkpoints:

| Layer | Check |
|---|---|
| Database | Connected to `retail_learning`, not `retail` |
| Raw data | 541,909 rows even after a loader rerun |
| Sales rules | 530,104 included and 11,805 excluded across the full extract |
| Modeling scope | 9,394 lines and 1,729 product-days |
| Split | 1,027 train, 347 validation, 355 holdout rows |
| Validation | Product mean wins; weighted-price regression beats median-price regression |
| Holdout | Both methods around 103.29 MAE / 80.68% WAPE |
| Elasticity | About -1.85, not a causal estimate |
| Simulation | Naive wrong sign; adjusted/randomized near -1.5 |
| Scenario | No recommended price; valid grids stay within constraints |
| Dashboard | Product/date/range controls work without saving rows |

UUIDs and timestamps should change; substantive results should not. The guided
schema is specific to this course; explain the meaning of every table and field.

## 2. Add an independent verification script

Create **`learning/check.py`** from your own understanding of the checks:

1. Assert the raw row count and unique `(source_file, source_row)` count match.
2. Assert there are no duplicate modeling keys.
3. Reconcile transaction units and revenue with daily units and revenue.
4. Read your model metadata and predictions CSV; assert every holdout date is
   later than the training cutoff.
5. Recalculate MAE and WAPE by hand in Python and compare with the saved CSV.
6. Load the latest model. Generate a zero-change scenario; assert one unique
   candidate, zero relative revenue change, and no recommended price.
7. For a regular scenario, assert every grid price is in both allowed ranges.
8. Ask for an unseen product and an unsupported weekday; assert no curve.

Use `assert` for learning checks. Turn these into `unittest` tests when you can
explain each assertion. Check outcomes and invariants rather than reproducing
the same implementation in test code.

Do not import your training script for a helper while it executes training at
module scope. First refactor it into functions and a guarded `main()`, or put
shared metric functions in a separate module. Explain why that refactor matters.

## 3. Debug systematically

| Symptom | First things to inspect |
|---|---|
| `ModuleNotFoundError` | Use `.venv/bin/python`; install requirements in that environment |
| Workbook not found | Check filename, capitalization, project root, and `ROOT` |
| Connection refused | Start Docker; check `docker compose ps` and port 5434 |
| Password authentication failed | Compare root `.env` with the existing Docker database configuration; do not delete the volume to guess a fix |
| Database does not exist | Create `retail_learning` once as instructed |
| Relation does not exist | Did ingestion create the raw table? Did you apply `learning/views.sql` in the learning database? |
| Counts doubled | Examine source identifiers and loader replacement transaction |
| Pandas/math type error | Inspect `type(value)` and numeric conversion from SQL decimals |
| Category prediction error | Was the product or weekday absent during fitting? Reject unsupported inputs |
| Log calculation is invalid | Inspect zero/nonpositive inputs; do not hide the problem with arbitrary constants |
| Streamlit port occupied | Use 8502 for the learning app or stop your previous learning process |
| Curve changes after modifying the SQL view | Retrain; old model and new data may no longer match |
| Model does worse than baseline | Report it; inspect assumptions before changing anything |

For every bug, record the exact error, the failing line, one inspected input,
and why the fix addresses the cause. Random changes make debugging harder.

## 4. Explain the project in five minutes

Use one diagram, one data-quality example, one plot, one metric table, and one
causal simulation result. Answer:

1. What question did you answer, and what question remains unanswered?
2. What does a raw row represent? What does a model row represent?
3. What was excluded, and why were raw records retained?
4. Why did you choose a time split and a simple baseline?
5. What did the regression improve, if anything?
6. Why is -1.85 not a proven causal elasticity?
7. What does randomization fix in the synthetic experiment?
8. Why does the scenario tool refuse to recommend a price?
9. What additional data would you ask the retailer for?
10. How would another person reproduce your result?

## 5. Mentor answer notes

Try answering before reading these.

- A raw row is an invoice line; a model row is one selected UK product on one
  observed selling day. Aggregation changes the question and information content.
- The loader transaction replaces one input atomically. Primary-key provenance
  does not prove identical business-field rows are mistakes.
- Time splits approximate applying earlier knowledge to later observations.
  Random splits can mix nearby periods. Neither fixes causal confounding.
- A baseline establishes whether extra model complexity adds predictive value.
  Here the baseline wins validation and overall holdout performance is similar.
- A coefficient interval concerns sampling uncertainty under assumptions.
  It does not remove omitted confounders, selection bias, or endogeneity.
- Large purchases and missing-customer mix can move weighted price mechanically.
  Fewer observed units are not necessarily fewer opportunities to buy.
- Randomization makes assigned price independent of potential-outcome drivers
  in expectation. The simulator's adjustment works because its complete driver
  is known; this is not promised for the real workbook.
- An optimizer maximizes the equation supplied to it. Constant elasticity often
  yields a boundary optimum. A business recommendation requires evidence beyond
  a constrained arithmetic maximum.
- Useful additional data includes offered prices, exposure, stock availability,
  promotions, costs, and credible experiment assignments/outcomes.

## 6. Review your implementation critically

Explain how you would improve each component without looking at the code blocks:

| Component | Improvement to consider |
|---|---|
| Ingestion | More explicit validation and source summaries |
| SQL | Clear grain, inclusion rules, and reconciliation |
| Exploration | Repeatable artifacts and sensitivity checks |
| Training | Portable coefficients, per-run evaluation, data fingerprints |
| Simulation | Monte Carlo uncertainty and saved configuration |
| Scenarios | Freshness checks and explicit unsupported inputs |
| Dashboard | Read-only database transactions and helpful error handling |

This repository contains the lessons and infrastructure, not a separately
maintained production solution. Use expected outputs and your independent
checks to review your implementation.

## 7. Final independent tasks

Choose at least two:

- Add a seventh product with a written inclusion rationale. Rebuild every affected
  layer and explain why the earlier holdout is no longer untouched.
- Add a product/weekday baseline using fitting-period information only.
- Implement a data fingerprint that rejects scenarios after data changes.
- Replace the pickle artifact with JSON coefficients and reproduce predictions
  independently within numerical tolerance.
- Add a bootstrap or longer-lag sensitivity analysis and explain what uncertainty
  it addresses and what it cannot fix.
- Rebuild the simulator from a blank file with different confounding strength.

**Completion standard:** with the guide closed, write a SQL aggregation, explain
a metric calculation, rebuild one script from an outline, diagnose an error,
and defend the project's main limitation. That demonstrates understanding rather
than memorization.

## 8. A simple mastery rubric

For each lesson, assess four skills: explain the purpose, implement a small part,
check the output, and diagnose a deliberately introduced mistake.

| Level | Evidence |
|---|---|
| 0 — Following | Can run copied code but cannot explain it yet |
| 1 — Guided | Explains it and makes changes with hints |
| 2 — Independent | Recreates the main idea and checks it without the guide |
| 3 — Transfer | Applies it to a different product/example and explains limitations |

Aim for level 2 on the main workflow and level 3 on at least two lessons. Advanced
covariance theory can remain at level 1 initially, provided the student can
explain why an ordinary interval is insufficient and what the chosen method
does not solve. Revisit a lesson rather than treating copied output as mastery.
