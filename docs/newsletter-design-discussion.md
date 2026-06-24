# Newsletter design discussion — 2026-06-23

Saved verbatim so it can be reread after clearing context.

## Prompt

If this becomes a weekly newsletter, how should it be structured. I'm concerned about what happens when an old investment is found via a new news source, how to handle things like running totals of investment for the quarter or year. What the content should be and how it's structured to be readable to people, without the AI prompt smell that sometimes creeps into reports.

## Response

Four separate problems here — worth treating separately since they need different fixes.

**Newsletter structure**
The current report (8 sections, full VC-profile rebuild every week, big appendix) reads like an internal analyst memo, not something you'd want landing in an inbox weekly. I'd restructure around a stable "issue" skeleton: a short *This Week* (3–5 bullets, what's new), *The Numbers* (running totals + revisions, see below), and then 1–2 deal write-ups in depth rather than profiling every VC every week. Move the full historical VC-profile section out of the weekly cadence entirely — that's reference data, not news, and rebuilding it every week is exactly the kind of bulk that makes a report feel AI-generated. Tradeoff: each issue is less exhaustive, but far more readable; you'd want a separate "VC database" view (could just be a standing page, not a new artefact) for the deep-reference use case.

**Old investment surfacing via a new source**
You already have the machinery for this — `is_new_this_run`, and `first_seen`/`last_seen` on the ledger entry, both distinct from `announcement_date`. The newsletter's "what's new" section should drive off `is_new_this_run` (discovery), not `announcement_date` — but when the two dates are far apart, say so explicitly ("a £700k round from March, surfaced this week via Daily Business Group") rather than presenting it as fresh news. Silently folding a backdated deal into "this week's deals" is the thing that'll erode reader trust fastest.

**Running totals (quarter/year)**
This is the real trap. If you compute the quarter total from discovery date, it'll jump around for reasons unrelated to actual investment activity. Compute it from `announcement_date` against the ledger instead — that's the only definition that means the same thing every time someone asks "what's the Q2 total." The cost is that the number will get revised upward in later issues as backfill comes in. Handle that like official stats revisions, not a bug: print the total plus an explicit delta versus the last issue's stated figure for the same period — "Q2 total: £143m (+£12m since last issue — a March Lentitek round surfaced via Daily Business Group)." That turns backfill into a visible feature of the system rather than an inconsistency.

**Avoiding AI-prompt smell**
Looking at the sample report (`data/reports/2026-06-08_vc-report.md`), the tells are: the repeated "**X is the [superlative]**" headline pattern three times in a row, editorializing after the fact ("this is a marquee deal for the Scottish ecosystem," "signals that... at scale"), and restating caveats in nearly every section. Fixes I'd put directly into `reporter.md`: ban that bolded-superlative-headline template, cut any sentence that explains *why* a fact is significant rather than just stating a sharper fact, put caveats in one place only, and vary section opening structure so it doesn't read like a filled-in form. Worth also feeding the agent 1–2 real newsletter examples (Sifted's briefing, Tech.eu) as a tone anchor rather than describing tone abstractly.

Want me to turn this into actual changes — updated `reporter.md` content sections, plus the quarter/year revision logic in the deduplicator or a new aggregation step?
