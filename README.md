# Retail Pricing Learning Lab

A hands-on course for building a retail data science project yourself:
Docker, Git, Python ingestion, PostgreSQL, exploratory analysis, elasticity,
causal simulation, constrained price scenarios, and a Streamlit dashboard.

The lessons include runnable code to type and understand, expected outputs,
exercises, debugging guidance, and parent/mentor checkpoints. Start with
[the course guide](lessons/README.md). Allow 12–16 learning sessions plus two
setup sessions if Docker and Git are new.

## What is included

```text
lessons/             Docker + Git setup and eight project lessons
learning/            Workspace where you write your implementation
compose.yaml         PostgreSQL service with persistent storage
.env.example         Local configuration template
requirements.txt     Python packages for the course
PUBLISHING.md        Instructions for creating your own GitHub repository
```

There is no prebuilt application to launch on checkout. You create the scripts,
SQL, and dashboard through the lessons. This folder is self-contained and can
be copied into its own repository; it does not depend on the parent project.

## Prerequisites

- macOS and a terminal; the course uses VS Code for editing.
- Python; Python 3.10–3.12 is a practical choice for these pinned dependencies.
  The core examples were also tested with Python 3.9 in the original environment.
- Docker Desktop with Compose. See [Docker setup](lessons/00_docker.md).
- Git for version history. See [Git setup](lessons/00_git.md).
- Your own `Online Retail.xlsx`, supplied separately.

## First-time setup

Open this repository folder in VS Code and run commands from its root. The
example checkout path is `~/projects/retail-pricing-learning`; substitute yours.

```sh
python3 --version
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements.txt
cp .env.example .env
```

Only copy `.env.example` when `.env` does not already exist. Edit `.env` to set
your own local password before initializing PostgreSQL. It is ignored by Git.

```sh
docker compose up -d --wait
docker compose ps
```

PostgreSQL is available on `127.0.0.1:5434`, with initial user/database `retail`.
Lesson 1 creates the separate `retail_learning` database used by your code.
This Compose project has its own name and volume; it does not use a parent
project's database. Copies of this lab use the same default Compose name, so
run one lab stack at a time unless you deliberately change project name/port.

## Add the workbook locally

Place `Online Retail.xlsx` in the repository root, alongside `compose.yaml`.
It should contain the columns:

```text
InvoiceNo, StockCode, Description, Quantity, InvoiceDate, UnitPrice, CustomerID, Country
```

Expected results refer to the workbook with 541,909 invoice-line records from
December 2010 through December 2011. Another dataset version may yield different
counts. The workbook is not bundled or automatically downloaded. Obtain it
separately and follow its provider's terms; it stays ignored by Git.

## Work through the course

1. [Docker setup](lessons/00_docker.md)
2. [Git setup](lessons/00_git.md)
3. [Foundations and database connection](lessons/01_foundations.md)
4. [Excel ingestion](lessons/02_ingestion.md)
5. [SQL audit and daily views](lessons/03_sql_analysis.md)
6. [Exploration](lessons/04_exploration.md)
7. [Modeling and evaluation](lessons/05_modeling.md)
8. [Causal simulation](lessons/06_causality.md)
9. [Scenarios and dashboard](lessons/07_dashboard.md)
10. [Independent assessment](lessons/08_assessment.md)

Write your files under `learning/`. The lessons supply names and commands in
order. Once you have built the dashboard, start it with:

```sh
.venv/bin/python -m streamlit run learning/app.py --server.port=8502 --server.address=127.0.0.1 --browser.gatherUsageStats=false
```

Open http://localhost:8502. Optional notebook work requires installing `ipykernel`
in `.venv`; the course itself can be completed entirely with scripts and SQL.

## Learning principles

Predict before running, explain each line, inspect outputs, and reconstruct
small pieces without the guide. A passing script is only one checkpoint.
The real-data model estimates observational associations. The separate synthetic
experiment teaches causal identification; it does not validate retail price
effects. The baseline's validation win is a useful result, and recommendations
must remain unsupported when evidence is insufficient.

## Sharing your work

See [PUBLISHING.md](PUBLISHING.md) to copy this folder into a standalone local
repository and push it to your GitHub account. The ignore rules exclude local
credentials, workbooks, virtual environments, generated reports, model artifacts,
and database files. Source files you write in `learning/` can be committed.
