# Legacy Repo Reconciliation — `xcerebroai/mecklenburg-intel`

**Task 1 deliverable. This is a report and recommendation only — no action has been taken against the old repo. It has not been touched, migrated, archived, or deleted.**

## What's actually in it

Contrary to a "stub" assumption, `xcerebroai/mecklenburg-intel` (created 2026-05-05) contains a **complete, working, one-day build**:

- Five live scrapers (`scrapers/polaris.py`, `code_violations.py`, `tax_delinquent.py`, `foreclosures.py`, `estates.py`) plus one CSV-import shim (`scrapers/rod.py`)
- A real pipeline (`pipeline/build_leads.py`, ~50KB) that joins on parcel ID, runs a 6-pattern stack, computes derived fields, and writes `data/leads.json`
- Real output data: `data/leads.json` is **8.2MB** — not synthetic/placeholder
- A single-file vanilla-JS dashboard (`index.html`) with filters, row-expand, CSV export
- `RECON.md` (Phase 1 recon) and `methodology.html` documenting the design
- All 12 commits landed within about 3 hours on 2026-05-05 — a rapid, complete build, not an abandoned scaffold

## Genuinely reusable findings (carried forward into this recon)

The prior build's Phase 3 recon **already answers, with direct verification, two items this project's Phase 0 recon could only flag as open**:

- **Mecklenburg ROD (Manatron) has no public/anonymous search at all.** Every unauthenticated search path returns a 238-byte "Session state is not available" stub; default `guest`/`guest` credentials fail validation; the platform is subscriber-only. This is stronger evidence than this project's own recon produced and has been folded into `config/counties/mecklenburg_nc.json` as a known limitation, flagged for a quick re-confirmation rather than a fresh investigation.
- **POLARIS's ArcGIS REST root** (`https://polaris3g.mecklenburgcountync.gov/polarisv/rest/services`, service `TaxParcel_camadata`, ~446K parcels, ArcGIS Server 10.81) — found and scraped successfully by the old build. Carried forward into `parcel_master_polaris` in the new config, flagged for re-verification since it's ~2 months old.
- **mecktimes.com's Cloudflare behavior**: the old build's operator notes record that `urllib`, `requests`, `curl`, and even `curl_cffi` impersonating Chrome 120 all got 403 on the first request from their build machine — only Anthropic's own WebFetch service got through. This is a stronger, more specific finding than this project's own recon (which only tested via WebFetch and one curl pass) and suggests IP-reputation-based blocking, not just UA filtering.

## Real conflicts with this project's framework (not just a style difference)

1. **Scoring model does not conform to the framework's actual engine.** The old repo tiers leads by *count of distinct pattern categories firing* (`jfc/tax/estate/code/lien/transfer` — Hot=3+, Warm=2, Active=1), with no per-doc-type base score anywhere. Per direct research of `scaffold/pipeline/score.py` (the framework's real, executing scoring code — see `/docs/scoring-model-reconciliation.md`), every score requires a per-canonical-doc-type `BASE_SCORE` as its foundation; stack-depth only ever contributes a bonus on top of that base, never a substitute for it. The old repo's mechanism is a third model, distinct from both the framework's design doc and its code.
2. **Primary-source rule violation.** The old repo's `tax_delinquent.py` scrapes **Kania Law Firm and RBCWB** — two law-firm marketing/case-tracking sites — as its only tax-foreclosure source; there is no scraper hitting Mecklenburg's own tax collector list directly. Per `knowledge_base/protocols/01_county_recon.md` §01.6: *"Paid-data aggregators and reseller portals are NOT primary recon targets... they are reseller layers over official data, not the official record authority."* This is an explicit framework rule the old build did not follow (it predates this project's framework-invariant statement and was likely built under a looser or earlier standard).
3. **No `stable_id` field, no GHL export format.** Neither concept as stated in this project's invariants exists in the old repo. (Note, per separate research: neither concept exists in the framework's own code either, in the exact form stated — see the open item on `stable_id`/GHL below. This is not unique to the old repo.)
4. **Fuzzy name-based estate matching** — the old repo's own methodology docs admit the decedent-to-parcel name match attaches an average decedent to ~3.4 parcels, with some false positives expected. This is a different join type than the parcel-key fuzzy-matching the Camden/Burlington MOD-IV lesson warns against (name→parcel, not parcel-key→parcel-key), but it's still a known accuracy limitation worth carrying forward as a documented caveat, not a silent assumption, if estate matching is rebuilt for Wake/Mecklenburg.

## Recommended disposition (awaiting your confirmation — nothing has been done)

**Do not migrate the old repo's code or `data/leads.json` wholesale into `wake-mecklenburg-intel`.** Its scoring model and primary-source choices don't conform to the framework as it actually runs, and its output data was generated under that non-conforming model — importing it would mean either re-scoring it later (redundant work) or shipping data that doesn't match this build's stated invariants.

**Do treat it as prior-art recon intelligence.** Its three genuinely reusable findings (above) have already been folded into `config/counties/mecklenburg_nc.json`'s `known_limitations`/`notes` fields with explicit "carried forward, needs re-verification" framing, not treated as settled fact.

**Leave the old repo exactly as it is** — untouched, not archived, not deleted — until you confirm a disposition. Options, for your decision:
- Leave it running independently as-is (it's a working POC on its own terms, even if it predates this project's framework conformance).
- Archive it (GitHub "archive" flag — read-only, not deleted) once Wake/Mecklenburg's combined build supersedes it functionally.
- Cherry-pick specific reusable snippets (e.g. the POLARIS objectid-cursor resume pattern, or the mecktimes Cloudflare workaround notes) into the new build's scrapers once Phase 1 starts.

No action taken. Awaiting your call.
