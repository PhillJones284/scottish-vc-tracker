# Scottish Venture News

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

The system is a deterministic pipeline that converts unstructured news into structured investment intelligence and publishes it as a weekly report and live website.

### Pipeline

```
News Sources
  → 1a. Fetcher (Python)
  → 1b. Scraper (Claude agent)
  → 1c. Firecrawl Scraper (Python)
  → 2.  Parser (Python)
  → 3.  Deduplicator (Python)
  → 3.5 Report Stats (Python)
  → 3.6 Chart Generator (Python)
  → 4.  Reporter (Claude agent)
  → 5.  VC Profiler (Python + Claude agent)
  → 6.  Deal Table Generator (Python)
  → 7.  Investor Page Generator (Python)
  → 8.  Sources + Landing Page Generator (Python)
  → 9.  Git commit + push (Python)
  → 10. Buttondown newsletter draft (Python)
```

Each stage performs a single transformation with clearly defined inputs and outputs.

---

## Architecture

### 1a. Fetcher (Python)

Fetches content from configured sources and filters it down to investment-relevant candidates.

* Input: `config/sources.json`
* Output: `data/raw/YYYY-MM-DD_candidates.json`, `data/raw/YYYY-MM-DD_fetch_log.json`
* Function: HTTP fetching, RSS/Atom parsing, text extraction (trafilatura), keyword filtering
* Implementation: `pipeline/fetcher.py`

Skips sources with `type: "firecrawl"` (handled by Stage 1c) and `type: "vc_newsrooms"` sources without an `rss_url` and without `direct_fetch_confirmed: true` (handled by Stage 1b via WebFetch, since plain HTTP is frequently blocked by bot protection or JS-rendered content on VC websites). `direct_fetch_confirmed` marks sources individually verified to extract cleanly via plain httpx — see the 2026-07-01 vc_newsrooms audit.

---

### 1b. Scraper (Claude agent)

Reads pre-fetched candidates and extracts structured investment records. Falls back to direct web fetching if the fetcher produced no results.

* Input: `data/raw/YYYY-MM-DD_candidates.json` (or `config/sources.json` in fallback mode)
* Output: `data/raw/YYYY-MM-DD_<source-slug>.json` per source
* Function: extraction of structured investment data from unstructured text

---

### 1c. Firecrawl Scraper (Python)

Handles sources that require JavaScript rendering or structured scraping (currently: Crunchbase).

* Input: `config/sources.json` (sources with `type: "firecrawl"`)
* Output: `data/raw/YYYY-MM-DD_<slug>.json` per source
* Implementation: `pipeline/firecrawl_scraper.py`

---

### 2. Parser (Python)

Transforms raw scraper output into structured, normalised investment records.

* Extracts: companies, investors, deal stage, amount, sector, date
* Resolves: entity aliases and inconsistent naming
* Normalises sectors to a taxonomy; a company may belong to multiple sectors
* Output: `data/processed/investments.json`
* Implementation: `pipeline/parser.py`

---

### 3. Deduplicator (Python)

Maintains a canonical historical record of all observed investments.

* Matches new records against existing ledger entries using three-tier confidence scoring (`definite` / `probable` / `possible`)
* Auto-merges only `definite` matches; stages `probable` and `possible` pairs in `merge_candidates.json` for human review
* Output:
  * `data/processed/investments_deduped.json`
  * `data/processed/ledger.json` (system of record)
  * `data/processed/merge_candidates.json` (persistent duplicate review queue)
* Implementation: `pipeline/deduplicator.py`

---

### 3.5. Report Stats (Python)

Computes all deterministic figures (deal counts, totals, deltas) that the reporter will narrate. Separating this from the LLM agent prevents arithmetic errors in the report.

* Output: `data/processed/report_stats.json`
* Implementation: `pipeline/report_stats.py`
* Hard gate: refuses to run if `merge_candidates.json` contains any unresolved `pending` pair

---

### 3.6. Chart Generator (Python)

Renders two chart PNGs — investment stage distribution and sector distribution — for embedding in the weekly report.

* Output: `data/reports/charts/YYYY-MM-DD_stage.png`, `data/reports/charts/YYYY-MM-DD_sector.png`
* Implementation: `pipeline/chart_generator.py`, `pipeline/chart_style.py`

---

### 4. Reporter (Claude agent)

Generates the weekly analyst-quality intelligence report. Narrates numbers from `report_stats.json` and embeds charts from Stage 3.6 — does not compute figures independently.

* Output: `data/reports/YYYY-MM-DD_vc-report.md`

---

### 5. VC Profiler (Python + Claude agent)

Refreshes standing per-VC reference pages for all firms active in the current run.

* Stats step: `pipeline/vc_profile_stats.py` → `data/processed/vc_stats.json`
* Agent step: rewrites `data/vc-profiles/<slug>.md` for each active VC

---

### 6. Deal Table Generator (Python)

Generates a static HTML deal table for the website.

* Output: `docs/deals/index.html`
* Implementation: `pipeline/deal_table_generator.py`

---

### 7. Investor Page Generator (Python)

Generates a static HTML investor directory for the website.

* Output: `docs/investors/index.html`
* Implementation: `pipeline/investor_page_generator.py`

---

### 8. Sources + Landing Page Generator (Python)

Generates the sources reference page and the site landing page.

* Output: `docs/sources/index.html`, `docs/index.html`
* Implementation: `pipeline/sources_page_generator.py`, `pipeline/landing_page_generator.py`

---

### 9. Git commit + push

Commits and pushes `docs/` to trigger a GitHub Pages rebuild. Data files (ledger, reports, vc-profiles) are left for manual review and commit.

---

### 10. Newsletter draft (Python)

Creates a Buttondown draft with the week's report content. Nothing goes to subscribers until sent from the Buttondown dashboard.

* Implementation: `pipeline/newsletter_publish.py`
* Requires: `BUTTONDOWN_API_KEY`, `IMGBB_API_KEY` in `.env`

---

## Configuration

| File                            | Purpose                                                        |
| ------------------------------- | -------------------------------------------------------------- |
| `config/sources.json`           | Curated news sources and query targets (do not edit directly)  |
| `config/known_vcs.json`         | VC firm identity resolution and alias mapping (do not edit directly) |
| `config/suggested_sources.json` | Staging area — unknown sources found by scraper, pending review |
| `config/suggested_vcs.json`     | Staging area — unknown VCs found by scraper, pending review    |
| `config/sectors.json`           | Sector classification taxonomy                                 |
| `config/fx_rates.json`          | Currency normalisation (mid-market rates)                      |

---

## Data Model

### Raw data — `data/raw/`

Per-source JSON files written by Stages 1a, 1b, and 1c each run.

### Processed data — `data/processed/`

| File                       | Persistence  | Description                                          |
| -------------------------- | ------------ | ---------------------------------------------------- |
| `investments.json`         | Transient    | Normalised extraction output (this run)              |
| `investments_deduped.json` | Transient    | Deduplicated output (this run)                       |
| `ledger.json`              | **Persistent** | All-time historical record — the system of record  |
| `merge_candidates.json`    | **Persistent** | Audit trail of duplicate pairs and their resolutions |
| `report_stats.json`        | Transient    | Deterministic figures for the reporter (this run)    |
| `report_history.json`      | **Persistent** | Each run's stated totals, for computing week-on-week deltas |
| `vc_stats.json`            | Transient    | Per-VC stats aggregation for the profiler (this run) |

### Reports — `data/reports/`

| Path                              | Description                            |
| --------------------------------- | -------------------------------------- |
| `YYYY-MM-DD_vc-report.md`         | Weekly intelligence report             |
| `charts/YYYY-MM-DD_stage.png`     | Investment stage distribution chart    |
| `charts/YYYY-MM-DD_sector.png`    | Sector distribution chart              |

### VC profiles — `data/vc-profiles/`

One standing Markdown reference page per VC firm (`<slug>.md`), accumulating full Scottish deal history. Refreshed by Stage 5 whenever a firm appears in a run.

### Website — `docs/`

Static HTML site served via GitHub Pages. Overwritten each run.

| Path                    | Description             |
| ----------------------- | ----------------------- |
| `index.html`            | Landing page            |
| `deals/index.html`      | Deal table              |
| `investors/index.html`  | Investor directory      |
| `sources/index.html`    | Intelligence sources    |
| `charts/`               | Chart PNGs (copied from `data/reports/charts/`) |

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

### Full pipeline (headless)

```bash
export ANTHROPIC_API_KEY=sk-...
python pipeline/run.py
# or for a specific date:
python pipeline/run.py --date 2026-05-26
```

`pipeline/run.py` orchestrates all stages in sequence, runs gate checks between stages, and exits with a non-zero code on failure. Requires `ANTHROPIC_API_KEY` and the `claude` CLI on `$PATH`.

### Interactive run (via Claude Code)

Open the project in Claude Code and say "run the agent". Claude will invoke each stage in sequence and report gate results interactively.

### Individual stages

Stages 1b, 4, and 5 are Claude agents — run them interactively via Claude Code.

The remaining stages are Python and can be run directly:

```bash
source .venv/bin/activate

# Stage 1a — Fetcher
python pipeline/fetcher.py [--date YYYY-MM-DD]

# Stage 1c — Firecrawl Scraper
python pipeline/firecrawl_scraper.py [--date YYYY-MM-DD]

# Stage 2 — Parser
python pipeline/parser.py [--date YYYY-MM-DD]

# Stage 3 — Deduplicator
python pipeline/deduplicator.py [--date YYYY-MM-DD]

# Stage 3.5 — Report Stats
python pipeline/report_stats.py [--date YYYY-MM-DD]

# Stage 3.6 — Chart Generator
python pipeline/chart_generator.py [--date YYYY-MM-DD]

# Stage 6 — Deal Table
python pipeline/deal_table_generator.py

# Stage 7 — Investor Page
python pipeline/investor_page_generator.py

# Stage 8 — Sources + Landing Page
python pipeline/sources_page_generator.py
python pipeline/landing_page_generator.py

# Stage 10 — Newsletter draft
python pipeline/newsletter_publish.py [--date YYYY-MM-DD]
```

---

## Outputs

Each run produces:

### Weekly report

A structured Markdown report containing:

* Total investment activity summary with week-on-week comparisons
* Active VC firms in the period
* Sector-level capital distribution (with chart)
* Investment stage breakdown (with chart)
* Deal-level structured listings
* Historical comparisons of VC activity

### Website

A static HTML site published to GitHub Pages, updated each run:

* Deal table — browsable, filterable record of all tracked investments
* Investor directory — per-VC activity summaries
* Sources page — the intelligence sources feeding the pipeline

### VC profiles

Standing per-firm reference pages in `data/vc-profiles/`, accumulating full Scottish deal history across all runs.

### Newsletter

A Buttondown draft ready to send, containing the week's report content.

---

## Limitations

* Only publicly reported deals are captured
* Deal values are often undisclosed or estimated
* Coverage depends on source availability and access constraints
* The system reflects reported activity, not private deal flow
* Outputs are for analysis only and are not financial advice

---

## Design Principles

* Deterministic transformation of inputs into structured outputs
* Append-only ledger as the system of record
* Arithmetic and deduplication-aware counts computed in Python — never delegated to an LLM agent
* Clear separation between raw, processed, and output layers
* Configuration-driven extraction and classification
* Modular pipeline stages with single responsibilities
* Conservative deduplication: flag for human review rather than auto-merge on ambiguous matches

---

## Operational Notes

### Data completeness

This system only processes publicly available information. Coverage gaps are expected due to:

- Paywalled or inaccessible sources
- Incomplete or delayed reporting of investment events
- Variability in source structure and formatting

---

### Scraper failures

If a source returns no data:

- Check `data/raw/YYYY-MM-DD_fetch_log.json` for per-source diagnostics
- Inspect `data/raw/` for partial outputs or error markers
- Validate `config/sources.json` endpoints

---

### Deduplication conflicts

The deduplicator uses fuzzy matching and stages ambiguous pairs in `data/processed/merge_candidates.json` for human review. Only `definite` confidence matches are auto-merged; `probable` and `possible` pairs always require a decision before the pipeline continues.

---

### Report sparsity

Low activity in reports may indicate either:

- A genuinely quiet funding period
- Source degradation or ingestion failure

Always validate against `data/raw/` before interpreting output quality.
