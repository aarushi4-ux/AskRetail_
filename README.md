# 🧠 AskRetail: An AI Analytics Copilot with a Measured Accuracy Score

> Ask business questions in plain English. Get the SQL, a chart, and a written explanation, plus a benchmark that tells you how often the AI is actually right.

![Python](https://img.shields.io/badge/Python-3.11-blue) ![DuckDB](https://img.shields.io/badge/DuckDB-warehouse-yellow) ![dbt](https://img.shields.io/badge/dbt-modelling-orange) ![Streamlit](https://img.shields.io/badge/Streamlit-UI-red) ![Status](https://img.shields.io/badge/status-in%20progress-lightgrey)

**Live demo:** `<add Streamlit Community Cloud link>` · **Eval report:** `docs/eval_report.md`

---

## 1. Overview

Most "AI over data" projects are chatbot wrappers that nobody measures. **AskRetail** is a text-to-SQL copilot built on a clean star schema, with three things that make it trustworthy:

1. **Guardrails**: every generated query is parsed, validated, read-only, and row-limited before it runs.
2. **Grounded answers**: the narrative is written only from the returned result set, not from the model's imagination.
3. **An evaluation harness**: 50 benchmark questions with gold-standard SQL and answers, scored automatically. Accuracy is tracked across experiments (baseline → schema pruning → few-shot → self-repair).

**Example**

> **Q:** Why did revenue drop in the UAE last month?
> **A:** UAE revenue fell 23% month over month (from X to Y). Order count was flat, but average order value dropped because the top product category lost its largest customer... *(chart + SQL shown below the answer)*

## 2. Features

- Natural-language → SQL using schema context (table/column descriptions, sample values, join paths)
- Read-only execution with SQL validation (allow-list of statements, table allow-list, forced `LIMIT`)
- Auto-chart selection (line for time series, bar for categories, table otherwise)
- Plain-English summary grounded in the result rows
- "Show SQL" panel and one-click re-run
- Self-repair loop: if the query errors, the error is fed back to the model for one retry
- Evaluation harness with execution accuracy, per-category breakdown, and experiment history
- Query log (question, SQL, latency, tokens, success) for debugging and cost tracking

## 3. Architecture

```
            ┌────────────┐   question   ┌──────────────────────────┐
  User ───▶ │ Streamlit  │ ───────────▶ │  Orchestrator (Python)   │
            │    UI      │ ◀─────────── │                          │
            └────────────┘ chart+text   │ 1. Schema pruning        │
                                        │ 2. Prompt build (+few-shot)│
                                        │ 3. LLM → SQL             │
                                        │ 4. Validate (sqlglot)    │
                                        │ 5. Execute (read-only)   │
                                        │ 6. Repair on error       │
                                        │ 7. LLM → grounded summary│
                                        └───────────┬──────────────┘
                                                    │
                                      ┌─────────────▼─────────────┐
                                      │  DuckDB (star schema)     │
                                      │  built by dbt             │
                                      └─────────────▲─────────────┘
                                                    │
                                      Raw CSV (Online Retail II / Olist)

  Eval harness: 50 questions + gold SQL ──▶ run pipeline ──▶ compare result sets ──▶ accuracy report
```

## 4. Tech stack

| Layer | Tools |
|---|---|
| Languages | Python, SQL, (Jinja for dbt models) |
| Warehouse | DuckDB (local, free); optional BigQuery free tier |
| Modelling | dbt-duckdb (staging → marts, tests, docs) |
| LLM | Claude API (or any LLM via a thin adapter class) |
| SQL safety | `sqlglot` (parse, validate, rewrite), read-only DuckDB connection |
| UI | Streamlit + Plotly |
| Eval | pytest + custom harness, results stored as CSV/Parquet |
| Deploy | Streamlit Community Cloud (public link), GitHub Actions for eval on PRs |

## 5. Dataset

**Recommended: Online Retail II** (UCI), a UK online retailer's transactions with customers in many countries **including the United Arab Emirates**, so the "UAE revenue" example works with real data. Alternative: **Olist** (Brazil), which has richer tables (orders, items, payments, reviews, sellers) but no UAE; use state or category in your questions instead.

## 6. Data model (star schema)

```
            dim_date                dim_customer
               │                        │
               └────────┐   ┌───────────┘
                         ▼   ▼
                      fct_sales  ◀──── dim_product
                (invoice_id, date_key, customer_key,
                 product_key, quantity, unit_price,
                 revenue, is_return)
```

dbt layers: `stg_*` (clean types, drop nulls/cancellations handling) → `dim_*` / `fct_*` (marts). Add dbt tests: `unique`, `not_null`, `relationships`, and a custom test that revenue = quantity × price.

## 7. Repository structure

```
askretail/
├── data/raw/                  # not committed (download script provided)
├── dbt_project/
│   ├── models/staging/
│   ├── models/marts/
│   └── tests/
├── src/
│   ├── llm/                   # adapter, prompts
│   ├── schema/                # schema docs, pruning, sample values
│   ├── sql_guard.py           # validation + LIMIT enforcement
│   ├── executor.py            # read-only DuckDB runner
│   ├── charts.py              # auto chart selection
│   ├── summarize.py           # grounded narrative
│   └── pipeline.py            # orchestrates the flow
├── app/streamlit_app.py
├── eval/
│   ├── benchmark.yaml         # 50 Qs: question, gold_sql, category, difficulty
│   ├── run_eval.py
│   └── results/               # per-experiment scores
├── docs/                      # architecture, eval_report.md, screenshots
├── tests/
├── requirements.txt
└── README.md
```

## 8. Getting started (Windows / PowerShell)

```powershell
git clone https://github.com/<you>/askretail.git
cd askretail
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt

# 1. Get data
python scripts/download_data.py

# 2. Build the warehouse
cd dbt_project
dbt deps
dbt build
cd ..

# 3. Set your key
$env:ANTHROPIC_API_KEY = "your-key"

# 4. Run the app
streamlit run app/streamlit_app.py

# 5. Run the benchmark
python eval/run_eval.py --config baseline
```

## 9. Guardrails

| Risk | Mitigation |
|---|---|
| Destructive SQL | Read-only connection + parse with `sqlglot`; only `SELECT` allowed |
| Huge scans/results | Enforced `LIMIT`, query timeout |
| Unknown tables/columns | Allow-list check against the schema before execution |
| Hallucinated answers | Summary prompt receives only result rows and must cite figures from them |
| Prompt injection via data values | Sample values are truncated/escaped; model told to treat them as data |
| Ambiguous questions | Model may ask one clarifying question instead of guessing |

## 10. Evaluation (the differentiator)

**Benchmark:** 50 questions across categories: simple aggregation (10), filtering & time (10), joins (10), ranking/top-N (8), period comparison (7), "why" questions needing multi-step queries (5). Each has gold SQL and difficulty tag.

**Metric:** *Execution accuracy*: the generated query's result set matches the gold result set (order-insensitive, numeric tolerance). Also track validity rate (runs without error), latency, and tokens per question.

**Experiments to run (log each):**

| # | Config | Execution accuracy |
|---|---|---|
| 0 | Baseline: full schema, zero-shot | _measure_ |
| 1 | + column descriptions & sample values | _measure_ |
| 2 | + schema pruning (only relevant tables) | _measure_ |
| 3 | + few-shot examples (retrieved by similarity) | _measure_ |
| 4 | + self-repair on error | _measure_ |

> Fill the table with your real numbers. The story you want: "Raised execution accuracy from X% to Y% through schema pruning and retrieved few-shot examples." Keep a held-out set so few-shot examples never come from test questions.

## 11. Build plan (≈ 4 weeks)

**Week 1: Data & warehouse**
- Download dataset, explore, document quirks (cancellations, null customers, returns)
- dbt staging and star schema, tests, `dbt docs generate`
- Write 15 gold questions + SQL while you learn the data

**Week 2: Copilot core**
- Schema doc generator (descriptions, sample values)
- LLM adapter, prompt, SQL guard, read-only executor
- CLI version end to end; log every query

**Week 3: Evaluation & improvement**
- Finish 50 benchmark questions; build `run_eval.py`
- Run baseline, then iterate experiments 1-4; record results
- Error analysis: categorise failures (wrong join, wrong filter, ambiguity)

**Week 4: UI, deploy, polish**
- Streamlit app with chart + SQL + narrative
- Deploy, add screenshots/GIF, write `docs/eval_report.md`
- README final pass, 2-minute demo video

## 12. Limitations & future work

- Single dataset; schema generalisation untested
- Gold-set questions written by the author (risk of bias); mention this honestly
- Ideas: semantic layer (metric definitions), caching, multi-turn follow-ups, role-based table access, BigQuery backend

## 13. Resume bullets (fill in after measuring)

- Built a text-to-SQL analytics copilot (Python, DuckDB, dbt, Claude API, Streamlit) over a star-schema retail warehouse with read-only SQL validation and grounded narrative summaries.
- Designed a 50-question evaluation harness; improved execution accuracy from **X% to Y%** via schema pruning, retrieved few-shot examples and error self-repair.

## 14. License

MIT
