# AROL Analytics — Industrial Telemetry Analysis & AI Agent

[![Python](https://img.shields.io/badge/Python-3.12%2B-blue.svg)](https://www.python.org/)
[![Tests](https://img.shields.io/badge/Tests-139%20passed-brightgreen.svg)](tests/)
[![Institution](https://img.shields.io/badge/Politecnico%20di%20Torino-SDP%20Project-003366.svg)](https://www.polito.it/)
[![License](https://img.shields.io/badge/Academic-Project-orange.svg)]()

An end-to-end industrial data engineering and agentic AI pipeline for telemetry analysis on a **36-head AROL rotary capping machine**.

Raw high-frequency telemetry exports (~4.6 GB, 77.5M rows across 89 daily CSV files) are reconstructed in a streaming pipeline into per-closure events, analyzed by 14 deterministic analytical tools, exposed to natural-language querying via an LLM-powered agent (with grounded synthesis and deterministic fallback), and made accessible through an interactive terminal interface.

> Developed for the **System and Device Programming** course (Prof. Stefano Quer)  
> **Politecnico di Torino** — M.Sc. in Computer Engineering.

---

## 👥 Authors & Roles

- **[Matteo Zylfo](https://github.com/mzylfo)** — **Layer 1 Lead**: Industrial context analysis, streaming ingestion architecture, cross-day `CarryState` event boundary handling, closure event reconstruction ($\Delta\text{Count} > 0$), flicker filtering, and data quality validation.
- **[Christian Priska](https://github.com/christianpriska)** — **Layer 4 Lead**: Terminal bot interface, deterministic result formatters, end-to-end demo workflow, and system integration.
- **Irene** — **Layers 2 & 3 Lead**: 14 deterministic analytics functions (torque statistics, anomaly detection, statistical tests, KPIs), LLM semantic routing, and grounded query synthesis.

📄 **Slides & Presentation:** see [`presentation/Q3 - Project presentation.pdf`](presentation/Q3%20-%20Project%20presentation.pdf).

---

## 🚀 Key Highlights

- **Streaming Big-Data Ingestion:** Processed 89 daily CSV files (~4.6 GB, 77.5M rows) into compressed Parquet (`closure_events.parquet`, `idle_periods.parquet`) with constant memory consumption, ensuring cross-day state continuity with `CarryState`.
- **Event Reconstruction:** Accurately isolated 55.9M inferred closures from discrete 1 Hz PLC sampling, distinguishing true machine stops/resets from electrical bus flickers.
- **14 Deterministic Analytics Tools:** Outlier detection (IQR / Z-score), torque variability, Welch's t-test and Mann-Whitney U for head comparisons, failure code categorization, and idle gap analysis.
- **Zero-Hallucination Agent:** LLM (Ollama / local or cloud) used strictly for semantic tool routing and grounded natural-language summarization; all numerical calculations remain 100% deterministic.
- **Resilient Fallback:** Fully operational even without an active LLM backend via keyword-based routing.
- **Robust Test Coverage:** 139 automated tests covering all 4 layers with synthetic datasets and zero external API dependencies.

---

## Architecture

Four layers, each building on the last:

1. **Ingestion** (`src/arol_analytics/ingestion/`) - 89 raw wide-format CSVs → two clean Parquet datasets (`closure_events.parquet`, `idle_periods.parquet`) plus a JSON/Markdown data-quality report.
2. **Analytics** (`src/arol_analytics/analytics/`) - 14 deterministic tools (success rate, torque statistics/trend/correlation, anomaly detection, head comparison, failure analysis, production speed, idle analysis, KPI dashboard, event listing, chart rendering) over the Layer-1 output. `src/arol_analytics/reports/` renders three of these into polished Markdown report templates (KPI dashboard, anomaly report, head comparison report).
3. **Agent** (`src/arol_analytics/agent/`) - routes a natural-language question to one or more Layer-2 tools via an LLM (Ollama, local or cloud), with a deterministic keyword-based fallback when no LLM is reachable. Plain numeric questions are answered by deterministic formatters; "why"/"explain"/multi-tool questions get a prose write-up the LLM composes strictly from the tool results (no recomputed or invented numbers).
4. **Terminal bot** (`src/arol_analytics/bot/`) - a local terminal chat interface (guided menu + free-text questions) over Layers 2 and 3. Reuses `python-telegram-bot`'s keyboard classes purely as an in-memory data container (no network service required).

Full detail, including a data-flow diagram, real dataset statistics, the complete status-code reference, every tool's method/parameters/caveats, and a traced agent-routing example, is in [`docs/`](docs/) - see the links at the bottom of this file.

---

## Prerequisites

- **Python 3.12+** (developed and tested with Python 3.14). The pinned NumPy and SciPy versions require Python 3.12 or newer.
- **An LLM backend for Layer 3/4's free-question mode (Optional).** Supports [Ollama](https://ollama.com) (`src/arol_analytics/agent/llm.py`), either local (`ollama serve`) or Ollama Cloud. Without either, the agent and bot still run fully via deterministic keyword routing.
- **No external network service required** — Layer 4 is a local terminal interface.

---

## Installation

```bash
git clone https://github.com/mzylfo/arol-telemetry-analysis.git
cd arol-telemetry-analysis
python3 --version                  # must be 3.12 or newer
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Place the raw telemetry CSVs under `src/data/` (or point the ingestion CLI at any other directory containing them - see below).
**NOTE! We have ALREADY INCLUDED the data in our repository.**

### Optional: enabling the LLM-backed agent

```bash
# Option A: local Ollama
ollama serve                      # in a separate terminal
ollama pull mistral                # or any other local model
export AROL_LLM_MODEL=mistral      # optional, "mistral" is the default

# Option B: Ollama Cloud (no local install)
export OLLAMA_API_KEY=<your-key-from-ollama.com/settings/keys>
export AROL_LLM_MODEL=<a-model-name-from-ollama.com/models>   # no ":cloud" suffix
```

`AROL_OLLAMA_HOST` overrides the host explicitly if neither default applies.

## Running each component

All commands assume the virtualenv is active. `PYTHONPATH=src` is required for the module CLIs below (pytest picks it up automatically from `pyproject.toml`).

```bash
# Layer 1: ingestion -- incrementally reads src/data/ and writes data/processed/
PYTHONPATH=src python -m arol_analytics.ingestion src/data --output-dir data/processed

# Layer 2: analytics CLI -- runs the full tool suite, writes reports/ (~40-90s)
PYTHONPATH=src python -m arol_analytics.analytics data/processed --output reports

# Report templates: renders three report types as separate Markdown files (~45s)
PYTHONPATH=src python -m arol_analytics.reports data/processed --output reports/samples
# --types kpi_dashboard anomaly head_comparison   (pass a subset to skip the rest)

# Layer 3: agent, interactive CLI (loads data/processed/ once, then a query loop)
PYTHONPATH=src python -m arol_analytics.agent data/processed

# Layer 4: terminal bot -- guided menu + free-question mode
PYTHONPATH=src python -m arol_analytics.bot data/processed
```

The ingestion CLI writes each processed CSV directly into the output Parquet file. It therefore keeps roughly one daily CSV in memory instead of collecting all 55 million closure events before writing them. `data/processed/` and `reports/` are gitignored (generated artifacts) - regenerate them with the commands above after cloning, or whenever the raw data changes.

## Demo

```bash
./scripts/demo.sh            # reuses data/processed/ if it already exists (~1 minute)
./scripts/demo.sh --fresh    # re-ingests from src/data/ first (~2-3 minutes)
```

One end-to-end run: loads the dataset, generates three report types (KPI
dashboard, anomaly report, head comparison report), then runs a few
natural-language questions through the Layer-3 agent. See
[`docs/demo.md`](docs/demo.md) for the stage-by-stage runbook.

## Running tests

The full suite is synthetic-data-only (no real dataset needed) and mocks the LLM entirely (no live Ollama needed):

```bash
PYTHONPATH=src pytest tests/ -v
```

(`pyproject.toml` already sets `pythonpath = ["src"]` for pytest, so plain `pytest tests/ -v` from the repo root works too.) 139 tests across ingestion, analytics, agent, reports, and end-to-end integration.

## Project structure

```
├── docs/                          # Architecture, data schema, analytics methods, agent flow, demo runbook
├── scripts/
│   ├── demo.sh                    # End-to-end demo (see "Demo" above)
│   ├── demo_agent_queries.py      # Canned Layer-3 Q&A used by demo.sh
│   └── verify_spec_queries.py     # Runs every specification example query through the agent
├── reports/
│   └── samples/                   # Sample reports generated from the real dataset
├── src/arol_analytics/
│   ├── ingestion/                 # Layer 1: raw CSV -> clean Parquet
│   ├── analytics/                 # Layer 2: 14 deterministic analytics tools
│   ├── reports/                   # Report templates: Layer-2 output -> Markdown reports
│   ├── agent/                     # Layer 3: LLM routing + deterministic result composition
│   └── bot/                       # Layer 4: terminal chat interface
├── tests/                         # pytest suite (conftest.py + 5 test modules)
├── pyproject.toml                 # pytest configuration
└── requirements.txt
```

## Documentation

- [`docs/architecture.md`](docs/architecture.md) - system overview, data-flow diagram, and per-layer implementation detail.
- [`docs/data_schema.md`](docs/data_schema.md) - raw and processed data formats, the complete closure status-code reference, key dataset statistics.
- [`docs/analytics_methods.md`](docs/analytics_methods.md) - every analytics tool's method, parameters, interpretation notes, and known caveats.
- [`docs/agent_flow.md`](docs/agent_flow.md) - the full query → routing → execution → grounded composition pipeline, the routing prompt, and three traced example queries.
- [`docs/demo.md`](docs/demo.md) - stage-by-stage runbook for `scripts/demo.sh`.
- further info on objective and development of each of the layers
  - [`docs/1layer_data_prep.md`](docs/1layer_data_prep.md) - layer 1
  - [`docs/2layer_analytic_function.md`](docs/2layer_analytic_function.md) - layer 2
  - [`docs/3layer_ai_agent.md`](docs/3layer_ai_agent.md) - layer 3
  - [`docs/4layer_bot_interface.md`](docs/4layer_bot_interface.md) - layer 4

## Sample reports

Generated from the real dataset in `data/processed/` - see
[`reports/samples/`](reports/samples/):
- [`kpi_dashboard.md`](reports/samples/kpi_dashboard.md) - KPI summary + full 36-head performance table.
- [`anomaly_report.md`](reports/samples/anomaly_report.md) - anomaly detection + failure breakdown, bursts, and monitoring recommendations.
- [`head_comparison_report.md`](reports/samples/head_comparison_report.md) - full 36-head ranking, statistical test results, and torque variability analysis.
