---
description: Post-run maintenance checklist — works through merge candidates, staged VCs, staged sources, filter/extraction issues, source errors, and soft gate failures one at a time.
---

# /maintain — Post-run maintenance checklist

Work through each of the five checks below in order. For each check:
- Run the relevant code or read the relevant files to find what needs attention
- If nothing needs action, say so clearly and move on
- If there are items requiring a decision, present them **one at a time** and wait for Phill's response before proceeding to the next item or the next check

---

## Setup: find the most recent run date

```python
import glob, os
from pathlib import Path
files = glob.glob('data/raw/*_candidates.json')
if files:
    dates = sorted(Path(f).stem.replace('_candidates','') for f in files)
    print('Most recent run date:', dates[-1])
else:
    print('No run dates found')
```

Use this date as `RUN_DATE` throughout the checks below.

---

## Check 1 — Pending merge candidates

Read `data/processed/merge_candidates.json`. Find all entries where `status == "pending"`.

If none: say "No pending merge candidates." and move to Check 2.

If any: for each pending pair, show both records side by side from `data/processed/ledger.json`:
- company name, round type, amount, date, lead investor, source URLs, first_seen
- the `match_type` and `note` from the merge candidate entry

Ask: **merge or dismiss?**

On merge: apply the merge exactly as described in CLAUDE.md's "Reviewing merge candidates" section — newer extraction's fields win, `first_seen` keeps the earliest, `last_seen` keeps the latest, `source_urls` union, `confidence` reassessed, `merge_confidence: "definite"`. Apply to both `ledger.json` and `investments_deduped.json`. Set `status: "merged"` in `merge_candidates.json`.

On dismiss: set `status: "dismissed"` in `merge_candidates.json`. Leave both ledger records as-is.

Handle one pair at a time. Do not move to Check 2 until all pending pairs are resolved.

---

## Check 2 — Staged VCs

Read `config/suggested_vcs.json`. 

If empty: say "No staged VCs." and move to Check 3.

If any: for each entry, show:
- canonical name, aliases, HQ, stage focus, notes (including the source article)

Ask: **add to known_vcs.json, or discard?**

On add: append the entry to `config/known_vcs.json` (match the schema of existing entries — `canonical_name`, `aliases`, `hq`, `stage_focus`, `scotland_active`, `notes`). Remove the entry from `suggested_vcs.json`.

On discard: remove the entry from `suggested_vcs.json`.

Handle one entry at a time. Do not move to Check 3 until `suggested_vcs.json` is empty.

---

## Check 3 — Staged sources

Read `config/suggested_sources.json`.

If empty: say "No staged sources." and move to Check 4.

If any: for each entry, show:
- slug, name, type, URL, notes

Ask: **add to sources.json, or discard?**

On add: append the entry to `config/sources.json` (match the schema — include `slug`, `name`, `type`, `url`, `search_path`, `rss_url`, `queries`, `best_effort`, `notes`; use `null` for fields not known). Remove the entry from `suggested_sources.json`.

On discard: remove the entry from `suggested_sources.json`.

Handle one entry at a time. Do not move to Check 4 until `suggested_sources.json` is empty.

---

## Check 4 — Filter and extraction issues

Read `data/raw/RUN_DATE_fetch_log.json` (if it exists).

Report any entries where:
- `items_found > 0` AND `candidates_added == 0` — keyword filter may be too aggressive for that source
- `text_extract_failures > 0` — extraction silently losing content

For each flagged entry show: source slug, items_found, items_passed_filter, candidates_added, text_extract_failures, error (if any).

Group into two buckets: "filter concerns" and "extraction failures". Present both buckets together as a summary — these are informational, not decision points. Note any that look particularly suspicious (e.g. a high-signal source like `sifted-uk` or `bgf-news` with many items but zero candidates).

Ask Phill if any source warrants follow-up investigation before the next run. If yes, handle the discussion; if no, move to Check 5.

---

## Check 5 — Source errors

From `data/raw/RUN_DATE_fetch_log.json`, find entries where `http_status` is 403, `error` contains "timed out", or `error` contains "nodename nor servname" (DNS failure).

Also read `data/raw/errors.json` if it exists and report any entries from this run's date.

Present a grouped summary:
- **403 Forbidden** — may be bot protection (often transient) or a dead endpoint
- **Timeouts** — likely transient; worth noting if persistent across runs
- **DNS errors** — site may be gone; candidate for removal from `sources.json`

For each DNS error source, ask: **remove from sources.json, or leave for now?**

For 403s and timeouts: present as informational only (no action required unless Phill wants to investigate).

---

## Check 6 — Soft gate failures

Check whether the following files exist and were written on or after today's date (use file modification time):

- `docs/index.html` — Stage 8 (landing page)
- `docs/deals/index.html` — Stage 6 (deal table)
- `docs/investors/index.html` — Stage 7 (investor page)

For VC profiles: read `data/processed/vc_stats.json` (if it exists from this run). For each VC listed, check that `data/vc-profiles/<slug>.md` exists and has today's date in `last_updated`. Report any missing or stale profiles.

Present a summary of what passed and what didn't. For anything that failed: ask Phill whether to re-run the relevant stage now or note it for later.

---

## Wrap-up

Once all six checks are complete, print a brief summary:
- How many merge candidates were resolved (and how — merged vs dismissed)
- How many VCs were added / discarded
- How many sources were added / discarded
- Any filter/extraction issues flagged for follow-up
- Any source errors worth watching
- Any soft gate failures outstanding

Then stop.
