# Build the Retail Pricing Explorer yourself

This course teaches you to build a retail pricing project from the ground up. You will write Python and SQL, check your own results, fit models, and
explain why an attractive-looking result may not support a business decision.

You will work in a new `learning/` folder and a separate `retail_learning`
database. You will create the code yourself from the guided examples. Docker
configuration and dependencies are included at the repository root; a separate
finished application is not required.

## What you will be able to do

- Explain a transaction row, a daily aggregate, and a model feature.
- Move an Excel workbook into PostgreSQL without duplicating it on reruns.
- Write SQL that audits messy data and builds a modeling dataset.
- Make plots that reveal customer mix and bulk-purchase effects.
- Evaluate a baseline and regression using chronological splits.
- Explain elasticity, confounding, uncertainty, and the limits of a model.
- Demonstrate causal identification with a controlled simulation.
- Build a constrained scenario tool and a small Streamlit app.
- Rebuild the workflow and defend your decisions without following a script blindly.

## Suggested pace

Use 12–16 sessions of 60–90 minutes for the core course, plus two setup sessions
if Docker and Git are new, with extra time for Python/SQL practice.
There is no deadline. If a checkpoint is unclear, repeat it before moving on.

| Lesson | What you build | Suggested sessions |
|---|---|---:|
| [Setup A. Docker](00_docker.md) | Docker Desktop, PostgreSQL, ports and persistent storage | 1 |
| [Setup B. Git](00_git.md) | Installation, commits, ignore rules, branches and merges | 1 |
| [1. Foundations and setup](01_foundations.md) | Workspace, database, tiny Python/SQL exercises | 2 |
| [2. Ingest the workbook](02_ingestion.md) | Your own schema and repeatable loader | 2 |
| [3. Audit and aggregate](03_sql_analysis.md) | Audit notes and modeling views | 2 |
| [4. Explore visually](04_exploration.md) | Plots, summary, feature dictionary | 2 |
| [5. Train and evaluate](05_modeling.md) | Baseline, regression, artifacts, run records | 2–3 |
| [6. Learn causality](06_causality.md) | Repeated synthetic experiment | 1–2 |
| [7. Scenarios and dashboard](07_dashboard.md) | Bounded projections, stored scenarios, app | 2 |
| [8. Rebuild and explain](08_assessment.md) | Final checks, debugging, independent presentation | 1–2 |

## How to use each lesson

1. Read the goal and predict the result before executing anything.
2. Type the short examples yourself. Explain each line aloud or in your notes.
3. Run one section at a time and inspect its output.
4. Complete the checkpoint without looking at the explanation.
5. Change one thing and predict what will happen before rerunning.
6. Only then compare your work with expected outputs and the checkpoints.

Typing is useful only if you understand the code. You should be able to close
the guide and reconstruct a small piece from memory.

Create `learning/notes.md` and use this template after each session:

```text
Today’s question:
What I predicted:
What I ran:
What I observed:
Why it happened:
A mistake I fixed:
What I can now explain without notes:
```

## Before starting

If Docker or Git is unfamiliar, start with [Docker setup](00_docker.md), then
[Git setup](00_git.md). Each has exercises, expected results, and checkpoints.

Follow [the repository setup instructions](../README.md) first. These lessons
use macOS, VS Code, Docker, and a local `Online Retail.xlsx` supplied separately.
Run commands from this repository's root unless labeled SQL. Paths such as
`~/projects/retail-pricing-learning` are examples; use your actual checkout path.
The workbook, credentials, generated models, and customer-derived outputs are
excluded from the repository by default.

If you are new to Python, first practice variables, lists, dictionaries, loops,
functions, imports, and exceptions. If you are new to SQL, practice `SELECT`,
`WHERE`, `GROUP BY`, and `ORDER BY` on the tiny examples in lesson 1. The project
will teach their application, but learning all syntax at once is unnecessarily hard.

## What success looks like

The goal is not a dramatic revenue increase. The existing baseline beat the
regression on validation. A successful student explains that result honestly,
keeps projections separate from causal claims, and can reproduce the analysis.

The examples are a guided implementation. Counts should match the supplied
workbook version; UUIDs and timestamps will differ between runs. Exact model results can also vary
slightly across numerical-library versions.

The core code examples were tested in the original course environment on
October 30, 2026 in a fresh disposable database: workbook inspection, ingestion
twice, SQL views, exploration, training, model reload, simulation, saved scenarios,
and Streamlit product/slider interactions passed. Test resources were removed
afterward. These checks establish that the examples work together; the exercises
and explanations are still yours to complete.

## Guidance for a parent or mentor

Ask “What do you expect?” before “Did it run?” Let her read the error and propose
a fix before helping. Ask for a drawing or a two-row worked example if the code
is confusing. Have her explain one result to a nontechnical person each session.

When stuck, give hints in this order: identify the failing line; identify the
type of each input; inspect one row; construct a tiny example. Reveal a complete
solution only after she has tried to reason through the problem.

Useful assessment questions and suggested answers appear in lesson 8. Passing
them matters more than finishing quickly.
