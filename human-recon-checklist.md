# Human Recon Checklist — Wake + Mecklenburg, NC

Prioritized because the highest-weighted score input just lost its assumed source. Do these in order. Each item lists why it matters, what's already known, and what a "done" answer looks like. **No scraper/pipeline code until this session is complete and reviewed.**

---

## 1. TAXDEL SOURCE — blocks everything

The assumed Wake daily delinquent CSV does not exist as a download link; the commonly-linked PDF is stale (`2023_WakeCounty_DelqREAccounts.pdf`, Last-Modified 2024-03-07). Wake's own "Real Estate Tax Bill Files" landing page exposes only schema/layout reference PDFs, no bulk data link, in raw HTML.

**Do:**
- In real Chrome, open `services.wake.gov/ptax/main/billing/` and try searching without an owner/account name — determine if there's any bulk/all-delinquent view, or if it's strictly per-bill lookup.
- Check `wake.gov/departments-government/tax-administration/real-estate/foreclosures` for the actual foreclosure-advertisement mechanism (News & Observer legal notice) as a possible parallel signal if no bulk delinquent file exists.
- For Mecklenburg: the published delinquent-taxpayer list reads as **annual** (NCGS 105-369) in county documentation. Open `taxbill.co.mecklenburg.nc.us/publicwebaccess/BillDelinquentSearch.aspx` in Chrome and determine whether it exposes a more-current view than the annual list, or if annual really is the only cadence available — if so, that's a hard cadence conflict with the framework's daily-pull requirement and needs to be flagged, not assumed away.

**Done looks like:** a confirmed URL (or explicit "does not exist publicly") for each county's authoritative, refreshable delinquent-tax source, with actual refresh cadence observed (not assumed).

---

## 2. LIS PENDENS CLASSIFICATION

Framework note says lis pendens files with Clerk of Superior Court (eCourts), not Register of Deeds, in NC — static fetches couldn't confirm this for either county, and the framework's own `canonical_doc_types.json` marks `LIS_PENDENS`'s `lead_pattern` as `"by_state_profile"` — a sentinel resolved at runtime from a `state_rule_family` value. **No NC state_rule_family currently exists in the framework** (the only concrete example found was `TX_non_judicial_foreclosure`) — this needs to be defined as part of this session's findings, not just the lis pendens location.

**Do:**
- Run a live interactive search on `portal-nc.tylertech.cloud/Portal/` for Wake and for Mecklenburg. Search Civil Actions and Special Proceedings case types for a live example; confirm whether "lis pendens" appears as a distinct filing/case-type or as an attribute of a civil case.
- Separately, open both ROD systems (`rodrecords.wake.gov`, `meckrod.manatron.com` — if item #4 below finds a way in) and check their recordable-document-type dropdowns/lists for "LIS PENDENS." A **negative** result (confirming it's absent from ROD) matters here, not just a positive one.

**Done looks like:** a confirmed answer to "where does lis pendens actually live for Wake and for Mecklenburg," plus enough detail (is NC's foreclosure process judicial, non-judicial, or hybrid from a lis-pendens-filing standpoint) to define a `NC_power_of_sale` (or similar) `state_rule_family` value.

---

## 3. WAKE ROD BLOCKER

`rodrecords.wake.gov/web/` loads a live Google reCAPTCHA script (`google.com/recaptcha/api.js`) — confirmed present in page source, but its placement in the flow is unconfirmed from a static fetch.

**Do:** walk the disclaimer → search flow manually. Does the captcha fire before you can search at all, only when you try to view/order a document, or only on account registration? Try a real search for a known recent recording without creating an account.

**Done looks like:** a clear statement of what the captcha gates, and whether guest search is actually usable at all.

---

## 4. MECK ROD BLOCKER

Manatron (`meckrod.manatron.com`) has a JS-gated, anti-autofill login form. **Strong prior evidence already exists**: the previous `xcerebroai/mecklenburg-intel` build's Phase 3 recon (2026-05-05) directly tested this and concluded there is **no public/anonymous search** — every unauthenticated search path returned a "Session state is not available" stub, and default `guest`/`guest` credentials failed validation. This is carried forward into `config/counties/mecklenburg_nc.json`.

**Do:** a quick re-confirmation only (not a fresh investigation) — open the site in Chrome, confirm the login modal is still the homepage, and try once more whether any guest/public tier has been added in the ~2 months since that finding. If nothing's changed, this item closes fast.

**Done looks like:** confirmed still true (fast path), or a new finding if Mecklenburg has changed its access model.

---

## 5. MECK TAXBILL — Cloudflare sustained-volume behavior

Cloudflare is present (`server: cloudflare`, `cf-ray`, `__cf_bm` bot-management cookie). A plain desktop-Chrome UA got HTTP 200 on a single request where an automated tool's UA got 403 — confirms the UA-fingerprint lesson from GHL, but says nothing about what happens under repeated/sustained automated-looking traffic.

**Do:** in real Chrome, submit several searches back-to-back (a handful, not a stress test) and watch for an interactive Cloudflare challenge appearing under volume, versus it staying clean.

**Done looks like:** a confirmed behavior profile — does Cloudflare escalate under normal-paced repeated use, or is single-request UA spoofing sufficient?

---

## Also worth doing in the same session (lower priority, lower cost)

- **Wake code enforcement** — no Wake-side code-enforcement portal has been fetch-tested with the same rigor as the Phase 0 six. Open `permitportal.raleighnc.gov`, search a code case, check rendering/blockers.
- **Wake parcel master REST endpoint** — `services.wake.gov/realestate/` / iMAPS likely has an ArcGIS REST endpoint like Charlotte's. Use DevTools Network tab while performing a search to find it — the same technique that found Charlotte's `MapServer` URL directly from a page fetch (no DevTools needed there since it was in a JSON feed; iMAPS may need the manual Network-tab approach).
- **Mecklenburg tax-foreclosure ArcGIS Experience app** — page loads, but a direct guess at its sharing-API `data` endpoint didn't resolve a usable FeatureServer URL. DevTools Network tab while the map loads should find it — likely another pure-API win like Charlotte's.
- **POLARIS REST endpoint re-verification** — the prior Mecklenburg build found `https://polaris3g.mecklenburgcountync.gov/polarisv/rest/services` (service `TaxParcel_camadata`, ~446K parcels). Confirm this is still live and the schema hasn't changed (~2 months old).

## Framework recon-completeness note

The framework's `knowledge_base/protocols/01_county_recon.md` (v5.5.0) formally requires, per county: 8 named recon artifacts (`source_discovery.md` through `recon_summary.md`, stubbed under `runs/<county>/recon/` in this repo), a Source-of-Record Matrix, ≥3 sample documents inspected per source before any source is classified as deferred, an explicit documented-API-discovery answer (Y/N + paths checked) per source, and a bulk-vs-per-record availability classification per source. This checklist covers the load-bearing gaps found so far — a full pass against all 8 artifacts is still needed to formally close recon for either county's `build_verdict`.
