# Wake + Mecklenburg Intel

Combined lead-intelligence build for **Wake County, NC** and **Mecklenburg County, NC**, on the Universal County Intelligence Framework (`xcerebroai/xcerebro-county-intel`).

**"Combined" is organizational, not a data merge.** Wake and Mecklenburg are two fully separate parcel-identity namespaces (Wake: PIN/REID; Mecklenburg: Parcel ID) that live in one repo for operational convenience. They are never joined, fuzzy-matched, or unified against each other — see the Camden/Burlington MOD-IV lesson and the framework's explicit cross-county-merge prohibition (`knowledge_base/architecture/12_entity_resolution.md`).

## Status

**Hard gate: CLOSED.** No scraper, parser, pipeline, or ingest code exists in this repo yet. What's here is recon output, framework-conformant config scaffolding, and a checklist for the next human-in-Chrome recon session. Phase 1 code starts only after that session and an explicit greenlight.

## What's in this repo

| Path | What it is |
|---|---|
| `recon-report.md` | Phase 0 automated (fetch-based) recon on the six architecture-deciding portals + secondary sources |
| `human-recon-checklist.md` | Prioritized checklist for the next human-in-Chrome session — what's still unconfirmed and why it matters |
| `doc-type-taxonomy.md` | Classification map for the classify pipeline stage, cross-checked against the framework's actual `canonical_doc_types.json` and `score.py` |
| `config/counties/wake_nc.json` | Wake County config, scaffolded from the framework's real `_template.json` schema |
| `config/counties/mecklenburg_nc.json` | Mecklenburg County config, same |
| `runs/wake_nc/recon/`, `runs/mecklenburg_nc/recon/` | Stubs for the 8 recon artifacts the framework's `01_county_recon.md` protocol requires per county, pending the human session |
| `docs/scoring-model-reconciliation.md` | The scoring-weight schema stated as a project invariant does not match the framework's actual scoring code — documented here, unresolved, needs an operator decision |
| `docs/mecklenburg-intel-legacy-reconciliation.md` | What's in the pre-existing `xcerebroai/mecklenburg-intel` repo, what conflicts with this framework, and a disposition recommendation (not yet actioned) |

## Framework conformance notes — read before writing any Phase 1 code

This build surfaced real gaps between what was assumed going in and what the framework (`xcerebroai/xcerebro-county-intel`) actually specifies:

- **Scoring weights.** The `taxdel:30 / probate:28 / fc:22 / lis pendens:15` schema referenced throughout this project's planning does not exist in the framework's actual code or design docs. See `docs/scoring-model-reconciliation.md`.
- **`stable_id`.** No field by this exact name exists in the framework. What exists is a chain of stage-specific IDs (`raw_event_id → base_record_id → lead_id → scored_lead_id`, plus `parcel_id`/`entity_id` for identity) — all per-county scoped, none globally unique across counties by design.
- **GHL export format.** No single authoritative column list exists. `knowledge_base/architecture/09_output_schemas.md` §7 documents a CRM-export schema that's never wired to any writer; `dashboard/dashboard.js` implements a real 32-column CSV export that's shaped for the operator dashboard, not GHL import, and doesn't match §7. These two disagree with each other — a GHL-specific export writer needs to be designed fresh.
- **NC canonical doc types.** The framework's `canonical_doc_types.json` (74 entries) is shaped around Texas non-judicial-foreclosure practice (`NOTICE_OF_SUBSTITUTE_TRUSTEE_SALE`, `AFFIDAVIT_OF_HEIRSHIP`). NC's judicial/quasi-judicial process (SP special proceedings, E-case estate administration, in-rem tax foreclosure) has no direct canonical-type equivalent yet — see `doc-type-taxonomy.md` for what's confirmed reusable vs. what needs new canonical types added upstream.
- **Code enforcement IS a scored signal.** Contrary to an open question raised earlier in this project, `CODE_VIOLATION_NOTICE` (base score 25), `DEMOLITION_ORDER` (50), and `CONDEMNATION_NOTICE` (45) already exist as first-class, weighted, lead-generating canonical types in the framework. This is resolved, not open — see `doc-type-taxonomy.md`.
- **Reseller/law-firm sites are never primary sources.** Confirmed via `knowledge_base/protocols/01_county_recon.md` §01.6 and the source-of-record matrix (`16_source_of_record_matrix.md`): a law-firm site republishing court filings is at best `SUPPORTING_EVENT_SOURCE` with reliability grade `C` ("vendor mirror"), never `PRIMARY_EVENT_SOURCE`. Relevant to Kania Law Firm / RBCWB, which the prior Mecklenburg build used as its only tax-foreclosure source.

## Confirmed ready for Phase 1 (once greenlit)

**Charlotte code enforcement** — verified live ArcGIS GeoServices REST endpoint (`gis.charlottenc.gov/.../CodeEnforcementCasesAll/MapServer/0`) plus a direct GeoJSON/CSV/Shapefile/KML download API, tested against 430,098 real records. Pure API pull, no scraping required. Logged in `config/counties/mecklenburg_nc.json` under `sources.code_enforcement_charlotte`.

## Next step

Complete the human-in-Chrome recon session described in `human-recon-checklist.md`, then bring the findings back for a build-verdict decision before any Phase 1 code is written.
