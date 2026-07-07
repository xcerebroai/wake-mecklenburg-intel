# Phase 0 — Automated Recon Report
Wake County + Mecklenburg County, NC — Universal County Intelligence Framework v5.5.0

**Method:** fetch-based recon only (`curl` with a desktop Chrome User-Agent — the AI-tool default UA got a 403 from Mecklenburg's WAF on a prior pass, confirming the Cloudflare lesson from GHL). No scraper/parser/pipeline code written. No CAPTCHA bypass attempted. Every conclusion below is either a directly observed HTTP/HTML fact or explicitly marked as needing a human-in-Chrome pass.

**Status:** framework repo `xcerebroai/xcerebro-county-intel` confirmed to exist (public, Python, private→public license "Proprietary VIP-only", last push 2026-06-27) — operating within it. **Repo-location flag:** no `wake-mecklenburg-intel` repo exists yet in the `xcerebroai` org or locally, but a separate standalone `xcerebroai/mecklenburg-intel` repo (public, created 2026-05-05, "Mecklenburg County NC motivated seller intelligence") already exists and this task wasn't told about it. Raised separately below — **not resolved in this report.**

---

## Primary six — architecture-deciding portals

| # | Portal | Rendering | Blockers | Stack | API endpoint found? | Scrape approach | Confidence | Human check needed? |
|---|---|---|---|---|---|---|---|---|
| 1 | NC eCourts Portal — `portal-nc.tylertech.cloud/Portal/` | **JS shell.** Base HTML is ~375 lines of scaffold (`navbar`, `main`, `timeout-wrapper`, `sessionTimeoutOverlay` divs); explicit `<noscript>JavaScript must be enabled...` block; loads `ePortal.js`, `main.js`, `TylerUiJs` bundles (AngularJS-era Tyler Odyssey Portal) | Hard cookie requirement (`nocookies-content` block present); 60-second session-timeout overlay; captcha referenced in portal help text (unconfirmed whether it fires on search itself vs. only payment/registration flows) | ASP.NET MVC 4.0 behind AWS ALB (`AWSALB` cookies), Tyler Odyssey "ePortal" product, AngularJS front end | No — case data loads via session-scoped AJAX calls not enumerable from static HTML | **Playwright.** JS execution + session/cookie handling + timeout management all required | Medium (stack certain, data-fetch endpoints unconfirmed) | **YES — highest priority.** This is the single portal that decides court-record strategy for BOTH counties. Do a real search, watch Network tab for the underlying XHR/JSON calls, and specifically test Special Proceedings, Estates, and a lis-pendens-adjacent civil filing to confirm doc-type classification |
| 2 | Mecklenburg Delinquent Bill Search — `taxbill.co.mecklenburg.nc.us/.../BillDelinquentSearch.aspx` | **Raw HTML.** Classic ASP.NET WebForms postback (`__VIEWSTATE`, `__EVENTVALIDATION`, `form action="./BillDelinquentSearch.aspx"`) | **Cloudflare** (`server: cloudflare`, `cf-ray`, `__cf_bm` bot-management cookie, embedded `/cdn-cgi/challenge-platform/scripts/jsd/main.js` fingerprint beacon). Notably: this exact URL returned **403 to the AI fetch tool's UA** but **200 to curl with a plain desktop Chrome UA** — Cloudflare here appears to filter on basic fingerprint/UA rather than issue a hard interactive challenge, at least on a single GET | ASP.NET WebForms behind Cloudflare, IIS | No separate API — WebForms postback only | **requests + BeautifulSoup**, real browser UA, full viewstate/postback handling. Keep Playwright in reserve — Cloudflare bot-management scoring can escalate under sustained automated volume in ways a single fetch won't reveal | Medium-High for a single request; **Low-Medium for sustained scrape behavior** (untestable from static recon) | **YES.** Submit an actual search in real Chrome, then repeat several searches back-to-back and watch for an interactive Cloudflare challenge appearing under volume |
| 3 | Wake Tax Bill Search — `services.wake.gov/ptax/main/billing/` | **Raw HTML.** ASP.NET WebForms postback (`txtFirst`/`txtLast`/`txtMiddle`, `hidSearchBy`, standard `__VIEWSTATE` fields) | **None detected** — no Cloudflare, no captcha, no login wall | ASP.NET WebForms, internal load-balanced app pool (`proxy-response-server: lrappsp01`) | No | **requests + BeautifulSoup**, no JS execution needed | High | Low priority — light spot-check that results render server-side post-search |
| 4 | Wake ROD (new system) — `rodrecords.wake.gov/web/` | Server-rendered app enhanced with jQuery/jQuery Mobile (`jquery-1.11.0`, `jquery.mobile-1.4.5`), not a full SPA. Redirects (302) to a disclaimer click-through gate first (`/web/user/disclaimer`, title "Self-Service") | **Live Google reCAPTCHA script loaded** (`https://www.google.com/recaptcha/api.js` present in page source) — the clearest hard blocker found in this recon. Placement in the flow (disclaimer step vs. guest search vs. only account/registration) is unconfirmed from a static fetch | Java backend (`JSESSIONID`), Tyler "Self-Service" Recorder product (v2026.1.5) | No | **Playwright required** — disclaimer gate + reCAPTCHA both need a real browser session | Medium (stack solid, captcha placement unconfirmed) | **YES — high priority.** Walk disclaimer → search manually to determine whether reCAPTCHA gates guest search or only account actions |
| 5 | Mecklenburg ROD (Manatron) — `meckrod.manatron.com/` | Raw HTML, ASP.NET WebForms + **Telerik RadControls** (`RangeContextMenu`, `dlgOptionWindow` artifacts); CSP allows `unsafe-inline`/`unsafe-eval` broadly | Login form present (`LoginForm1_txtLogonName`/`txtPassword`, "Public Computer" vs "Private Computer" radio) — but the username/password fields are `readonly="readonly"` and only unlock via an `onkeyup` JS handler, a legacy anti-bot/anti-autofill pattern. **Unconfirmed whether anonymous/guest search exists at all**, or if login (even a public/shared credential) is mandatory for every real-estate search | ASP.NET WebForms + Telerik AJAX, IIS 10, behind a Citrix NetScaler LB (`NSC_wt_...` cookie) | No | **Undetermined pending human check.** If guest search exists → requests+BeautifulSoup with postback handling; if login is mandatory → Playwright, given the JS-gated fields and Telerik AJAX behavior | **Low — least-understood portal of the six** | **YES — high priority.** Determine if a public/guest search path exists without credentials, and what "Public Computer" actually changes |
| 6 | Charlotte code enforcement open data — `data.charlottenc.gov/datasets/charlotte::code-enforcement-cases-all/about` | The Hub "about" page itself is a JS shell (Ember.js "opendata-ui" engine, `hubcdn.arcgis.com` chunk bundles) — **irrelevant**, see API column | None | Esri ArcGIS Server (GeoServices REST) behind an ArcGIS Hub SPA presentation layer | **YES — found and verified live:**<br>`https://gis.charlottenc.gov/arcgis/rest/services/HNS/CodeEnforcementCasesAll/MapServer/0`<br>+ direct download API: `https://data.charlottenc.gov/api/download/v1/items/f1c8670d7b6346ecbe17197a7316fff4/{geojson\|csv\|shapefile\|kml}?layers=0`<br>Confirmed schema: `CaseNumber, ParcelId, CaseType, FullAddress, CaseOrigin, CouncilDistrict, Inspector, DateCreated, DateClosed, CaseStatus` (`maxRecordCount: 2000`, capabilities `Map,Query,Data`). Live count query returned **430,098 records** | **Pure API pull.** No scraping needed at all — paginated `query` calls or scheduled bulk GeoJSON/CSV download | High | Low priority — just confirm actual data refresh cadence (compare `DateCreated`/`DateClosed` max values day-over-day) |

---

## Secondary sources — lighter pass

| Source | Finding |
|---|---|
| Kania Law Firm Mecklenburg foreclosure listings | WordPress site (`wp-content`), server-rendered `<table>` markup. Straightforward if ever used, but this is a **law-firm marketing page — enrichment/cross-reference only, never primary**, per framework invariant |
| RBCW Mecklenburg foreclosure listings | Same profile as Kania — WordPress, raw `<table>`, enrichment-only |
| Mecklenburg tax-foreclosure ArcGIS Experience app (`experience.arcgis.com/experience/640b8534...`) | Page loads (200), but a direct guess at the sharing-API `data` endpoint did not resolve a usable feature-service URL automatically. **Likely another pure-API win** (same pattern as the Charlotte success) but needs a human DevTools Network-tab pass to find its actual operational-layer FeatureServer, the same technique that cracked #6 |
| Wake delinquent tax PDF (`2023_WakeCounty_DelqREAccounts.pdf`, the file public directories link to) | **Confirmed stale** — `Last-Modified: Thu, 07 Mar 2024`. This is a one-time historical snapshot, **not** a daily-refreshed source. Do not build the taxdel pipeline around it |
| Wake "Real Estate Tax Bill Files" landing page | Contains only schema/layout reference PDFs (Tax Bill Layout, Payment File Layout, LenderCodes) — **no bulk data download link is exposed as a plain `<a href>` on this page.** This directly contradicts the earlier assumption that Wake publishes a daily CSV/zip here. **Unresolved — needs human recon**, since taxdel:30 is the single highest-weighted score input and Wake's primary source for it is not yet confirmed |
| ncnotices.com | Raw HTML, ASP.NET WebForms postback with checkbox-based city/category filters (`aspnetForm`). No blockers detected |
| Raleigh assessment liens PDF | Confirmed live and recently updated — `Last-Modified: Wed, 01 Jul 2026` (5 days before this recon). Static document, would need periodic re-download + PDF parsing, not a scrape target |

---

## Proposed architecture summary

**Pure API pull (no scraping):**
- Charlotte code enforcement (ArcGIS MapServer / GeoJSON download) — no rate limit observed in a single query; confirm actual refresh cadence before committing to "daily"
- *(likely)* Mecklenburg tax-foreclosure ArcGIS Experience app, pending human discovery of its FeatureServer

**requests + BeautifulSoup (server-rendered WebForms, no JS execution):**
- Wake Tax Bill Search
- Mecklenburg Delinquent Bill Search (flagged risk: Cloudflare bot-management behavior under sustained volume is unconfirmed — keep Playwright as fallback)
- ncnotices.com (statewide public notices)

**Playwright required (JS execution and/or interactive blockers):**
- NC eCourts Portal (AngularJS SPA, session-timeout handling — shared by both counties)
- Wake ROD new system (reCAPTCHA present, disclaimer gate)
- Mecklenburg ROD/Manatron (JS-gated login fields, Telerik AJAX, login requirement unresolved)

**Static document pulls (periodic fetch + parse, not scraping):**
- Raleigh assessment liens PDF

**Undetermined — resolve in the human recon session before any Phase 1 code:**
- Wake's actual current delinquent-tax bulk-file location (if a daily one exists at all)
- Mecklenburg ROD guest-vs-login requirement
- Wake ROD reCAPTCHA placement (guest search vs. account actions only)
- eCourts Portal's actual AJAX/API surface, and specifically **where lis pendens surfaces** — framework note says Clerk of Superior Court (eCourts), not ROD; this recon pass could not confirm or refute that since it requires an interactive case-type search. When you're in there, also check both ROD systems' recordable-document-type lists to confirm lis pendens is *absent* from ROD (negative confirmation matters here, not just positive)

**Cadence:**
- Charlotte code enforcement API: architecturally capable of daily/on-demand pulls; confirm actual server-side refresh frequency
- Wake / Mecklenburg tax-bill WebForms search: no inherent cadence restriction found — safe to poll daily against a known test parcel
- eCourts Portal, Wake ROD, Meck ROD: daily is fine once auth/JS/captcha handling is solved, but session-timeout and interactive constraints argue for **scheduled headless-browser runs**, not tight polling loops
- Kania / RBCW / Mecklenburg Times / ncnotices: weekly is likely sufficient — these are legal-notice/marketing layers with lower freshness requirements than the official tax/court primary sources

---

## Open item not addressed in this report

A standalone `xcerebroai/mecklenburg-intel` repo already exists (created 2026-05-05, predates this task) and this recon was never told about it. No `wake-mecklenburg-intel` repo exists yet anywhere. This report is staged locally at `~/repos/wake-mecklenburg-intel/recon-report.md` and has **not** been pushed or committed anywhere pending a decision on how the new combined project relates to the existing Mecklenburg-only repo.
