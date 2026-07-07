# Scoring Model Reconciliation

**Status: unresolved — needs an operator decision before Phase 1 scoring code is written.**

## The finding

The scoring weight schema stated as a framework invariant throughout this project's planning (`taxdel:30, probate:28, fc:22, lis pendens:15`) **does not exist anywhere in the actual `xcerebroai/xcerebro-county-intel` framework repository.** Neither of the two artifacts that *do* define scoring in that repo uses these names or numbers:

1. **`knowledge_base/domain/03_scoring_and_stacking.md`** (design doc) — per-doc-type-subtype base scores (e.g. "Notice of Trustee Sale" = 45, "Lis Pendens" = 35, "Tax Delinquent (1+ year)" = 35, "Probate Case Opened" = 35), a stack bonus by *count of distinct patterns* (+15/+25/+35), attribute bonuses (capped +40), tiers Hot 80-100 / Strong 60-79 / Workable 40-59 / Low 20-39 / Archive 0-19.
2. **`scaffold/pipeline/score.py`** (the code that actually runs) — a `BASE_SCORE` dict keyed by canonical doc type, using the **strongest single signal only** (`max()`, not a sum) — e.g. `NOTICE_OF_SUBSTITUTE_TRUSTEE_SALE: 60`, `LIS_PENDENS: 35`, `AFFIDAVIT_OF_HEIRSHIP: 55`, `CODE_VIOLATION_NOTICE: 25` — plus `STACK_BONUS = {0:0, 1:0, 2:12, 3:24}` keyed by `stack_depth`, a flat `RECENCY_BONUS = 5`, and `ATTRIBUTE_BONUS` values of 2-5 each (capped at 12). Tiers: Hot ≥80 / Strong 65-79 / Workable 50-64 / Low 35-49 / Archive <35.

**These two artifacts disagree with each other** on every number. Per direct research of the repo, the code (`score.py`) is the one that actually executes and should be treated as ground truth over the design doc where they conflict.

Neither artifact expresses score as a fixed per-*lead-pattern* weight summed across independent categories the way `taxdel:30 / probate:28 / fc:22 / lis pendens:15` implies. The real mechanism is: **base score = the single strongest doc-type signal present (not a sum across signal types) + a capped stack-depth bonus + a flat recency bonus + a capped attribute bonus, clamped 0-100, then tiered by threshold.**

## What this means for Wake/Mecklenburg

- Any Phase 1 scoring code should be written against `scaffold/pipeline/score.py`'s actual mechanism, not the `taxdel:30`-style schema — unless the operator explicitly wants a new/different scoring mechanism for this build.
- The prior `xcerebroai/mecklenburg-intel` build used a **third, different model entirely** — tier assigned purely by count of distinct pattern categories firing (`jfc/tax/estate/code/lien/transfer`), with no per-doc-type base score at all. This does not conform to `score.py`'s real mechanism either (see `/docs/mecklenburg-intel-legacy-reconciliation.md`).
- `document_priority` (a field on every canonical doc type, 5-95) is a red herring for scoring purposes — it's carried through to the signal dict for display/priority but is **never consumed in `compute_score`**.

## Open decision — needs the operator's call

1. Adopt `score.py`'s real mechanism as-is for Wake/Mecklenburg (recommended default — it's what the shared, reusable pipeline engine actually runs).
2. Adopt the `03_scoring_and_stacking.md` design-doc numbers instead (would require patching `score.py` for this build, or all builds).
3. Design a new mechanism specific to this build (would diverge from the shared framework engine, increasing long-term maintenance cost).

No scoring code has been written for this project. This document exists so the conflict is visible before that decision gets made silently by whichever number happens to get typed into a config file first.
