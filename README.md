# Scottish VC Investment Tracker

A multi-stage data pipeline that monitors publicly available news sources to track venture capital investment activity in Scottish scale-up companies.

It transforms unstructured reporting into a structured, deduplicated intelligence ledger of VC activity across Scotland.

---

## Purpose

The system tracks and structures venture capital investment activity in Scottish companies to enable longitudinal analysis of funding flows, investor behaviour, and sector trends.

It is designed to support:

* Fundraising and investor targeting
* Competitive intelligence and market mapping
* Sector-level capital allocation analysis
* VC firm activity and cadence tracking over time

---

## System Overview

The system is a deterministic pipeline that converts unstructured news into structured investment intelligence.

### Pipeline

News Sources → Scraper → Parser → Deduplicator → Ledger → Reporter → Weekly Report

Each stage performs a single transformation with clearly defined inputs and outputs.

---

## Architecture

The system is composed of four stages:

### 1. Scraper

Responsible for collecting raw mentions of venture capital activity from configured sources.

* Input: `config/sources.json`
* Output: `data/raw/`
* Function: extraction of unstructured investment-related content

---

### 2. Parser

Transforms raw text into structured investment records.

* Extracts: companies, investors, deal stage, amount, sector, date
* Resolves: entity aliases and inconsistent naming
* Output: `data/processed/investments.json`

---

### 3. Deduplicator

Maintains a canonical historical record of all observed investments.

* Matches new records against existing ledger entries
* Prevents duplicates and resolves near-duplicates
* Output:

  * `data/processed/investments_deduped.json`
  * `data/processed/ledger.json` (system of record)

---

### 4. Reporter

Generates structured analytical outputs for human consumption.

* Aggregates weekly investment activity
* Produces sector and geography breakdowns
* Builds VC firm activity profiles
* Outputs: `data/reports/YYYY-MM-DD_vc-report.md`

---

## Configuration

The system is fully configuration-driven.

| File                    | Purpose                                       |
| ----------------------- | --------------------------------------------- |
| `config/sources.json`   | Defines news sources and query targets        |
| `config/known_vcs.json` | VC firm identity resolution and alias mapping |
| `config/sectors.json`   | Sector classification taxonomy                |
| `config/fx_rates.json`  | Currency normalisation rules                  |

---

## Data Model

### Raw Data

`data/raw/`

Unstructured source material captured per ingestion cycle.

---

### Processed Data

`data/processed/`

* `investments.json`: normalised extraction output
* `investments_deduped.json`: cleaned output for the current run
* `ledger.json`: append-only historical record of all investments

---

### Outputs

`data/reports/`

Human-readable analytical reports generated per run.

---

## Setup

### Requirements

* Python 3.14+ (recommended via pyenv)
* Claude Code CLI authenticated environment

Install Claude Code:

```bash
brew install --cask claude-code
```

---

### Installation

```bash
git clone <repository-url>
cd scottish-vc-tracker

pyenv install 3.14.5
pyenv local 3.14.5

python -m venv .venv
source .venv/bin/activate

pip install -e .
```

---

## Execution

### Full pipeline

```bash
claude --print "Run the full VC tracking pipeline as defined in CLAUDE.md"
```

### Single stage execution

```bash
cd agents/scraper
claude --print "Run scraper as defined in CLAUDE.md"
```

Each stage can be executed independently via its agent definition.

---

## Outputs

Each run produces:

### Weekly report

A structured Markdown report containing:

* Total investment activity summary
* Active VC firms in the period
* Sector-level capital distribution
* Geographic breakdown across Scotland
* Deal-level structured listings
* Historical comparisons of VC activity

### Data artifacts

| File                                      | Description                  |
| ----------------------------------------- | ---------------------------- |
| `data/raw/*`                              | Raw extracted source data    |
| `data/processed/investments.json`         | Normalised dataset           |
| `data/processed/investments_deduped.json` | Deduplicated dataset         |
| `data/processed/ledger.json`              | Persistent historical record |
| `data/reports/*`                          | Generated analytical reports |

---

## Limitations

* Only publicly reported deals are captured
* Deal values are often undisclosed or estimated
* Coverage depends on source availability and access constraints
* The system reflects reported activity, not private deal flow
* Outputs are for analysis only and are not financial advice

---

## Extensibility

The pipeline is designed for incremental extension through additional agents or data sources.

Possible extensions include:

* Alternative data sources (e.g. Companies House filings)
* Founder and sentiment analysis from public discourse
* VC hiring and recruitment signal tracking
* Early signal detection models for investment activity

---

## Design Principles

* Deterministic transformation of inputs into structured outputs
* Append-only ledger as the system of record
* Clear separation between raw, processed, and output layers
* Configuration-driven extraction and classification
* Modular pipeline stages with single responsibilities

## Operational Notes

### Data completeness

This system only processes publicly available information. Coverage gaps are expected due to:

- Paywalled or inaccessible sources
- Incomplete or delayed reporting of investment events
- Variability in source structure and formatting

---

### Scraper failures

If a source returns no data:

- Check whether the source HTML structure has changed
- Inspect `data/raw/` for partial outputs or error markers
- Validate `config/sources.json` endpoints

---

### Deduplication conflicts

The deduplication layer relies on fuzzy matching. In rare cases:

- Distinct companies with similar names may be merged incorrectly
- The same deal reported with inconsistent metadata may be split

These cases are surfaced in `investments_deduped.json` for review.

---

### Report sparsity

Low activity in reports may indicate either:

- A genuinely quiet funding period
- Source degradation or ingestion failure

Always validate against `data/raw/` before interpreting output quality.