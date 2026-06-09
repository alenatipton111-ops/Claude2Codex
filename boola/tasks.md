# boola — tasks

## Active

- [x] [claude→codex] **T1: Add `CUSTOMER_PROFILE` config block** at the top of `prospect.html` per spec in `claude-notes.md` § Active plan. Migrate existing `RSS_FEEDS`, `NYC_GEO`, `NON_NYC_GEO`, and any `1-800-GOT-JUNK` / NYC-specific strings to reference fields on this object. Mark each migration site with `// SCALABILITY: source from CUSTOMER_PROFILE`. Do not move data to disk yet — that's Phase 3.

- [x] [claude→codex] **T2: Expand news feed list to 15 NYC sources.** Use the candidate list in `claude-notes.md` § Active plan #2. Before adding each, hit the URL once with curl and confirm it returns valid RSS XML (200 + `<rss>` or `<feed>` root). Substitute any that 404 or block bots. Final feed count must be ≥ 15. Update `CUSTOMER_PROFILE.newsFeeds`.

- [x] [claude→codex] **T3: Add `extractAddress(html)` helper** in `main.js` per spec in `claude-notes.md` § Active plan #4. Returns first match or `''`. Reject addresses outside `CUSTOMER_PROFILE.region.state` (use the regex's state-code group).

- [x] [claude→codex] **T4: Extend `lookup-company-info` IPC** in `main.js` to also populate `result.address` using `extractAddress` on each fetched site page (homepage, /contact, /about). Stop early once a non-empty address is found. Existing return shape stays compatible — just adds one field.

- [x] [claude→codex] **T5: Add `validate-news-lead` IPC handler** in `main.js`. Input: `{ name, headline }`. Calls `lookup-company-info(name)` internally. Returns `{ valid: bool, name, phone, website, address, reason: ?string }`. `valid: true` requires non-empty `phone` AND `address` AND that the looked-up `website` or `name` contains at least one ≥3-char token from `headline` (cross-check from § Edge cases). Bake in a 20s timeout per validation.

- [x] [claude→codex] **T6: Refactor `buildProspectsFromNews` in `prospect.html`** to be async. After extracting candidates, run them through `validate-news-lead` via IPC with a **3-at-a-time semaphore**. Keep validated leads only. Show progress strip per § Active plan #6: `"Validating leads… N / M checked, K valid"`. Total budget: 90s; abort remaining validations if exceeded and use what's validated so far. Dedupe by canonicalized phone digits at the end.

- [x] [claude→codex] **T7: Phase-1 fallback** — if validated count < 3 after T6, pad with the existing `fallbackPool` to reach 3. Mark with `// PHASE-1-TEMP:` comment block (Phase 2 will replace). Show banner `"News validation found nothing today; showing curated NYC list."` when fallback is used.

- [x] [claude→codex] **T8: Update lead card in `chat.html`** — when a lead has `address` and `phone` populated by validation, show `✓ Phone` and `✓ Address` badges (small green chips) above the existing detail rows. Don't duplicate the data; the badges just signal "this is callable today."

- [x] [claude→codex] **T9: Persist validated schema in `.boola_todays_leads.json`** — store `{name, phone, website, address, validatedAt}` per lead so app restart skips re-validation. On load, if a lead has `validatedAt` for today's date, render directly without re-checking.

- [ ] [claude→codex] **T10: End-to-end smoke test.**
  1. `rm ~/.boola_todays_leads.json` to force regeneration
  2. Relaunch boola (use the relaunch command in the codex-notes standing context)
  3. Open chat → Leads tab → tap ↻
  4. Confirm leads finish generating in ≤90s
  5. Confirm each lead has phone + address (inspect `~/.boola_todays_leads.json`)
  6. Confirm UI shows ✓ badges
  7. Record findings in `codex-notes.md`

- [x] [claude→codex] **T11: Sync to project folder** — _obsolete as of 2026-05-18: canonical folder is now `~/BOOLA/`, no separate sync step. Leave as `[x]`._

- [ ] [claude→codex] **T12: Acceptance check** — verify each acceptance criterion from `claude-notes.md` § Acceptance criteria. Note any that fail in `codex-notes.md` so Claude can address in the next planning round.

---

## Phase 1.5 — Lead source overhaul + setup wizard
_(Added 2026-04-29 — drop license-based signals, add action-based fetchers, add first-run setup wizard.)_

- [x] [claude→codex] **T13: Add `targetSignals` to `CUSTOMER_PROFILE`** per `claude-notes.md` § Customer Profile schema. Default = all 8 signals enabled. No UI yet — that comes in T18.

- [x] [claude→codex] **T14: Remove license-based fetchers.** Delete (or comment out with `// REMOVED 2026-04-29: license-based, replaced by action-based fetchers`) the IPC handlers `fetch-openings`, `fetch-warehouse`, and the DCWP-inactive-license branch of `fetch-closings` in `main.js`. Keep `parseWARN` and the WARN branch. Remove `mapOpening` from `prospect.html` and the DCWP branch of `mapClosing`.

- [x] [claude→codex] **T15: Add action-based fetchers in `main.js`.** One IPC handler per signal, each calling NYC OpenData (Socrata) endpoints listed in `claude-notes.md` § Lead source overhaul. Each returns the standard `{ name, address, borough, industry, source, date }` shape. Handlers must be **gated by `CUSTOMER_PROFILE.targetSignals`** — only enabled signals trigger their fetcher. Test each endpoint with curl before wiring (some Socrata datasets gate or rate-limit anonymous access — note any that need an app token).
  - `fetch-construction-permits` → DOB ipu4-2q9a, NB/DM
  - `fetch-renovation-permits` → DOB ipu4-2q9a, A1/A2
  - `fetch-job-filings` → DOB ic3t-wcy2 (covers move-in/move-out signal)
  - `fetch-illegal-dumping` → 311 erm2-nwe9, complaint_type='Illegal Dumping'

- [x] [claude→codex] **T16: Wire fetchers into the lead pipeline** in `prospect.html`. Replace the old `fetchOpenings`/`fetchClosings`/`fetchWarehouse` calls with the new fetchers, again gated by `CUSTOMER_PROFILE.targetSignals`. Each fetched lead still goes through `validate-news-lead` (T5) — same name+phone+address gate applies regardless of source. Update the source label so the UI can show "🏗️ Construction permit" / "🚮 Illegal dumping" / "📋 WARN closure" / "📰 News" instead of "Opening" / "Closing".

- [x] [claude→codex] **T17: Update `OPPORTUNITY_SIGNALS` and `LEAD_KEYWORDS`** in `prospect.html` to remove license/permit-issuance noise and tighten focus on the 8 `targetSignals`. Specifically remove anything referencing "licensed" as a positive signal. Keep the existing renovation/construction/closing entries.

- [x] [claude→codex] **T18: Build first-run setup wizard.**
  - Add `setup.html` (new file) — single-page form with the 6 fields listed in `claude-notes.md` § First-run setup wizard.
  - Add `createSetupWindow()` in `main.js`, opens at app start when `profile.json` doesn't exist at `app.getPath('userData')/profile.json`. Mascot/chat windows do **not** load until profile is saved.
  - Add IPC handlers `profile-load`, `profile-save`, `profile-exists`.
  - On save, write profile.json, close setup window, then call the normal app-start sequence (createMascotWindow, createChatWindow, etc.).
  - Add a "⚙️ Settings" button in the chat-pane sidebar that reopens setup to edit.
  - **Bootstrap default:** if no profile.json, the wizard pre-fills with the existing hardcoded values (Alena's profile) so she doesn't have to re-enter on first run after the upgrade.

## Phase 2 — Workbook-driven lead generation
_(Added 2026-05-08 — fixes the repeating-leads bug and rebuilds lead engine on `lead-rules.xlsx`. Spec in `claude-notes.md` § Phase 2.)_

- [x] [claude→codex] **T26: Build `scripts/build-lead-rules.js`.** Reads `~/BOOLA/lead-rules.xlsx`, emits `~/BOOLA/lead-rules.json` with all 8 tabs as keyed objects/arrays. Use the `xlsx` or `node-xlsx` npm package (or existing if already in deps). Run once and commit the resulting JSON. Add an `npm run build:lead-rules` script in `package.json`. Goal: zero xlsx parsing at runtime.

- [x] [claude→codex] **T27: Add sales-type + target-verticals to setup wizard.**
  In `setup.html`:
  1. Add **Sales type** single-select dropdown after Region. Options sourced from `lead-rules.json` → `Category_Summary` (35 options). Field: `profile.salesType` (string code like `facilities_commercial_services_junk_sales`).
  2. Add **Target verticals** multi-checkbox section, populated dynamically from `lead-rules.json` → `Master_B2B_Lead_Rules` filtered by `sales_type`. All checked by default. Field: `profile.targetVerticals` (array of vertical strings).
  3. Add **Territory radius** number input (miles). Default 25. Field: `profile.region.radiusMiles`.
  4. Bootstrap defaults for Alena's existing profile: `salesType = "facilities_commercial_services_junk_sales"`, all matching verticals checked, radius 25.

- [x] [claude→codex] **T28: Delete the static `fallbackPool` from `prospect.html`.** Not comment out — fully delete. Remove all references and the `// PHASE-1-TEMP:` block. This will break lead generation until T29-T31 land — that's expected; do these tasks together.

- [x] [claude→codex] **T29: Implement Bucket 2 — vertical fetcher.**
  In `main.js`: new IPC handler `fetch-vertical-leads({salesType, targetVerticals, region, currentMonth})`. For each enabled vertical (sorted by `priority_score` from workbook):
  1. Build search queries from the workbook's search-template field for that row
  2. Run through existing `searchCompanyWebsite` + `lookup-company-info`
  3. Validate name + phone (address optional)
  4. Tag each result with `{vertical, sourceKey, sourceUrl, priorityScore, suggestedBuyer (from buyer_titles)}`
  5. Stop when target count is reached
  Apply a per-call concurrency cap of 3, total budget 60s, return whatever validated by then.

- [x] [claude→codex] **T30: Implement Bucket 3 — seasonal toppers.**
  In `main.js`: new IPC handler `fetch-seasonal-leads({salesType, region, currentMonth})`. Find rows in workbook where `currentMonth ∈ best_buy_months`. If different from already-pulled verticals, run vertical fetcher for those. Used to fill remaining slots after Bucket 1+2.

- [x] [claude→codex] **T31: Rewrite `generateProspects` in `prospect.html`** as the workbook-driven 3-bucket pipeline per `claude-notes.md` § Phase 2 lead-gen pipeline. Acceptance: 10 leads/day, no static fallbackPool, never repeats yesterday's leads (rolling 14-day exclusion).

- [x] [claude→codex] **T32: Rolling exclusion list.**
  In `main.js`: maintain `~/BOOLA/.boola_lead_history.json` (or under `app.getPath('userData')`) with `[{name, date, source}]` entries for last 14 days. Drop entries older than 14 days on each write. `validate-news-lead` and the new vertical/seasonal fetchers must check this file and skip any name already on it. Persist new leads to it after generation.

- [x] [claude→codex] **T33: Lead card UI upgrade.**
  In `chat.html` lead rendering: each card gains
  - Vertical chip (purple pill)
  - Confidence score (1-100, computed via workbook `Source_Scoring_Rules`, color: ≥85 green / 70-84 yellow / <70 red)
  - "Why now" one-line (from buying-trigger logic)
  - Suggested buyer title
  - Source chip (workbook `source_name`)
  - "📧 Cold email" quick-action button that pre-fills Email pane with subject+opener built from workbook `Cold_Email_Rules` + lead vertical
  Lead schema expands accordingly; persisted in `.boola_todays_leads.json`.

- [ ] [claude→codex] **T34: Confidence scoring helper.**
  Add `computeConfidence(lead)` function consuming `Source_Scoring_Rules` from workbook JSON. Apply: base_score + recency_bonus + official_bonus + named_company_bonus + trigger_bonus, weighted by `verification_weight`. Store final score on each lead.

- [ ] [claude→codex] **T35: Smoke test the new pipeline.**
  1. Run boola for 3 consecutive days
  2. Compare `.boola_todays_leads.json` across days — confirm 10 distinct leads/day with ≤1 carryover
  3. Confirm each lead has name+phone, vertical, confidence, why-now, suggested buyer
  4. Confirm setup wizard shows 35 sales types and dynamic target-verticals list
  5. Change `salesType` to "Tech / SaaS / IT" → confirm leads now favor healthcare/financial verticals (per workbook priorities) instead of construction/property
  6. Record findings in `codex-notes.md`

- [ ] [claude→codex] **T36: Sync to `~/BOOLA/`** after T35 passes.

- [x] [claude→codex] **T39: Email header + body spacing fidelity end-to-end.** _Code complete 2026-05-19; real Gmail send/recipient verification deferred because OAuth is not set up on this Mac._
  Audit every code path that takes a Claude-generated email and exposes it to the user:
  1. `getEmailExpertSystem` already specifies "Subject line first. Then blank line. Then body." Verify the regex/split that parses subject from body in `generateEmail` is robust (handle missing blank line, multi-line subjects, leading/trailing whitespace).
  2. **Copy button** (`copyResult`) — currently copies rendered HTML innerText. Must preserve actual `\n` so pasting into Gmail/Outlook keeps paragraphs intact. Use `navigator.clipboard.writeText` on the raw text, not the rendered DOM text.
  3. **Send button** (`sendFromResult` → `send-email` IPC → nodemailer) — confirm `text:` field gets raw body with `\n` and `html:` field uses `<p>` paragraphs (not just `<br>`) so the email client renders proper paragraph spacing.
  4. Generated subject line is auto-routed to the email's actual Subject header — not buried in the body. Currently the email-pane result shows both subject+body in one text block. Add a small read-only subject field above the body when result renders, so it's visually clear which line becomes the Subject header.
  5. Acceptance: draft an after-call email, click Copy, paste into a fresh Gmail compose → paragraphs preserved, subject lands in subject row. Send via boola → recipient receives well-formatted email with paragraphs (verify in own inbox).

- [x] [claude→codex] **T40: Document templates with screenshot-based fill.** _Code complete 2026-05-19; live Gmail attachment send deferred because OAuth is not set up on this Mac._
  New feature: user uploads `.docx` templates with `{{placeholder}}` markers; boola fills them on demand using known lead data, manual input, or screenshot-extracted data.

  **STEP ZERO (do this first, before any code):**
  ```bash
  cd ~/BOOLA && npm install docxtemplater pizzip --save
  ```
  Verify `package.json` records both deps and `node_modules/docxtemplater` exists. If the install fails (network, lockfile, etc.), stop and report — do NOT proceed without these packages.

  **Dependencies:**
  - `docxtemplater` npm package + `pizzip` (its required peer) — these are the standard Word-mail-merge stack. Preserve fonts, tables, headers/footers, images. Install via `npm install docxtemplater pizzip --save`.

  **Storage:**
  - Templates live at `app.getPath('userData')/templates/<uuid>.docx`
  - Metadata at `app.getPath('userData')/templates.json`: `[{id, displayName, originalFilename, placeholders:[], createdAt}]`
  - `placeholders` populated at upload time by scanning the doc XML for `{{...}}` tokens.

  **Placeholder convention (final):**

  *Auto-fill from current lead context (set when user opens fill dialog from a lead card or selects a lead in dialog):*
  - `{{name}}` or `{{client_name}}` — lead's contact name
  - `{{email}}` or `{{client_email}}` — lead's email
  - `{{phone}}` or `{{client_phone}}` — lead's phone
  - `{{company}}` or `{{client_company}}` — lead's company name
  - `{{address}}` or `{{client_address}}` — lead's mailing address
  - `{{job_address}}` — defaults to lead address, but is **editable** in the fill dialog (job site often differs from billing/contact address)
  - `{{title}}` — lead's job title

  *Auto-fill from `profile.json` (captured once at setup, never re-asked):*
  - `{{rep_name}}`, `{{rep_email}}`, `{{rep_phone}}`, `{{rep_title}}`, `{{rep_company}}` — always populated from profile, never editable in fill dialog
  - Setup wizard must capture rep info if not already present: name, email, phone, title (use existing company field for rep_company)

  *Typed widgets (deterministic by placeholder name):*
  - `{{discount}}` or `{{discount_pct}}` — preset buttons **10% / 15% / 20% / Custom**. Custom opens number input. Stored as string like "15%".
  - `{{price}}`, `{{total}}`, `{{deposit}}`, `{{amount}}`, `{{subtotal}}` — currency input (USD-formatted, e.g. "$1,250.00").
  - `{{date}}`, `{{start_date}}`, `{{due_date}}`, `{{end_date}}` — date picker, defaults to today.
  - `{{quantity}}`, `{{qty}}`, `{{units}}` — integer input.

  *Custom user-defined fields:*
  - Any placeholder that doesn't match the above is **unknown** until configured.
  - At template upload time, after auto-detection, boola walks the user through each unknown placeholder one by one and asks: "How should `{{XYZ}}` be filled?" with options:
    - **Free text** (default — single-line or multi-line)
    - **Dropdown with options** — user supplies comma-separated options ("Standard, Premium, Enterprise")
    - **Number**
    - **Date**
    - **Currency**
    - **Map to a lead field** — dropdown showing existing auto-fill fields, so the user can alias their custom placeholder to (e.g.) `{{client_name}}` without renaming the template
  - This configuration is stored per-template in `templates.json` under a `fieldConfig` object: `{ "<placeholder>": { type: "dropdown" | "freeText" | "number" | "date" | "currency" | "alias", options?: string[], aliasOf?: string } }`
  - User can re-configure later from the template's settings menu (gear icon on each template in the Documents tab).

  **Setup wizard (extend `setup.html`):**
  - Add a **"Rep info"** section before the Document templates section if not already present. Captures: `rep_name`, `rep_email`, `rep_phone`, `rep_title`. Auto-saves to profile. These power the `{{rep_*}}` auto-fills.
  - Add a **"Document templates"** section after the existing fields.
  - Drag-drop or click-to-upload `.docx` files.
  - For each uploaded file:
    1. Scan placeholders.
    2. Show: filename, editable display name, detected placeholders as chips.
    3. For each placeholder that is NOT in the known-auto-fill list AND NOT in the deterministic-widget list (i.e. unknown custom field), present the field-type configuration UI inline — user picks type, supplies options if dropdown, or aliases to a known field.
    4. Save button only enables once all unknown placeholders have a type chosen.
  - Remove button per template.
  - On wizard save, copy files into templates directory, persist metadata + fieldConfig per template.
  - User can also reach the template management from a new "📄 Templates" section inside the Settings reopen flow — same upload + configure UI.

  **New "📄 Documents" tab in chat:**
  - Tab added to the main tab strip after "Tasks".
  - Shows a list of all saved templates with display name + placeholder count.
  - Click a template → opens the **Fill dialog**.

  **Fill dialog:**
  1. Header: template name + "Fill from" toggle: **🧠 Lead data** (default) / **📸 Screenshot** / **✍️ Manual**.
  2. Body: one input row per placeholder, pre-filled where boola knows the answer:
     - From the currently active lead in the Leads tab (if one is selected) — name, company, phone, address etc.
     - From `profile.json` — your name, your company, your contact info (for placeholders like `{{rep_name}}` or `{{rep_email}}`).
  3. Discount/price/date placeholders render their special widgets (preset buttons / currency / picker).
  4. **Screenshot mode:** drag-drop image OR Cmd+V paste from clipboard. Boola sends the image to Claude's vision API (model `claude-sonnet-4-5` or current) with a prompt asking it to extract specifically the missing placeholders → JSON response → auto-fill the form. User reviews + edits before confirming.
  5. **Manual mode:** form pre-fills only from profile.json — user types the rest.
  6. Footer: **Preview** (opens the filled doc in Quick Look) / **Download .docx** / **Send as email attachment** (uses existing Gmail send).

  **Claude vision wiring:**
  - `callClaude` currently only handles text. Add a new code path that accepts `{ system, userMsg, history, images:[{base64, mediaType}] }`.
  - Send to `https://api.anthropic.com/v1/messages` with content blocks: `[{type:'image', source:{type:'base64', media_type:'image/png', data:...}}, {type:'text', text: extractionPrompt}]`.
  - Extraction prompt: `"Look at this screenshot and extract the following fields. Return ONLY valid JSON with these keys: {placeholder list}. If a field is not present in the image, set its value to null."`
  - Add `// SCALABILITY: image API calls will route through boola backend in production` comment.

  **Docxtemplater integration in `main.js`:**
  - IPC `template-list` — returns metadata
  - IPC `template-fill` — input: `{templateId, data}` — loads template, runs through docxtemplater, returns either a temp file path (for download) or a base64 buffer (for email attachment)
  - IPC `template-add` — input: `{sourcePath or buffer, displayName}` — copies into templates dir, scans placeholders, persists metadata
  - IPC `template-remove` — input: `{templateId}` — deletes file + metadata entry
  - IPC `template-scan-placeholders` — input: file path or buffer — returns array of unique `{{...}}` tokens found in the doc text

  **Acceptance:**
  - Upload a real Word doc with 3+ placeholders via setup wizard → confirm placeholders are auto-detected (visible as chips).
  - Open Documents tab → click template → fill dialog opens with lead data pre-filled, manual fields empty.
  - Switch to Screenshot mode → paste a screenshot containing a name + email → boola fills those two fields automatically.
  - Click Download → `.docx` opens in Word with placeholders replaced and original formatting intact.
  - Discount field shows 10/15/20/Custom buttons; selecting one writes "10%" (or whatever) into the doc.

  Sync to `~/BOOLA/` when done.

## Phase 2.4 — Full regionalization (must happen before Phase 2.5)
_(Added 2026-05-12 — current code still hardcodes NYC OpenData endpoints, news feeds, chain lists, neighborhood regex. Cincinnati/Boston/SF customers would silently get empty leads. Fix before building Google Places so Places is region-aware from line 1.)_

- [ ] [claude→codex] **T48: Eliminate ALL hardcoded region/customer data. Zero special-casing for any city or product.**
  Boola must work identically for a rep in any U.S. city (and ideally any city globally) selling any B2B product/service. The code knows about nothing specific. The workbook + profile drive everything.

  **Step 1 — Delete hardcoded regional data:**
  - `main.js`: Delete `NYC_BIZ_API`, `DOB_PERMITS_API`, `DOB_JOBS_API`, `NYC_311_API`, `dol.ny.gov` constants and any handler that calls them directly. Delete `NYC_BOROUGHS` and `NYC_GEO_MAIN` regexes. Delete any hardcoded company names or borough lists.
  - `prospect.html`: Delete `KNOWN_COMPANIES` list (Vornado, Brookfield, etc.) — discovery is Google Places' job (T43). Delete or generalize `FILLER` regex (remove NYC neighborhood tokens). Delete or generalize `BROAD_KEYWORDS` (replace with content-based keywords like "deal/closed/construction" with no city tokens). Delete `NYC_GEO` and `NON_NYC_GEO` references that aren't sourced from profile.
  - Setup wizard: region is **free-text city + state** input. No dropdown of "supported regions."

  **Step 2 — Universal region resolution via geocoding:**
  - On profile save, if `region.latitude` / `region.longitude` not present, call **Nominatim** (free, no key, global): `https://nominatim.openstreetmap.org/search?q={city},{state}&format=json&limit=1`. Cache lat/lng/bbox onto `profile.region`. Works for any city on Earth.
  - If geocoding fails, show error: "Couldn't locate {city}, {state} — check spelling." Don't fall back to NYC.
  - All location-based searches (Google Places, etc.) use `profile.region.latitude`/`.longitude`/`.radiusMiles`. No region-specific code paths.

  **Step 3 — Default news feeds = national, not regional:**
  - Add a `Default_News_Feeds` workbook tab with ~10 high-quality national U.S. business feeds (Bloomberg Business, WSJ Business, Inc., Forbes Business, Axios Business, Reuters Business, Business Insider, CNBC Business, Crain's, BizJournals national).
  - Default profile.newsFeeds = the Default_News_Feeds list (loaded at profile creation).
  - News filter logic uses **content keywords** (closing/opening/permit/expansion/relocation/construction/renovation/dumping/damage) — NOT city/state names. A national feed mentioning Cincinnati construction passes the same way an NYC feed mentioning Manhattan construction does.

  **Step 4 — Regional enhancement layer (optional, data-driven only):**
  - Add a new workbook tab `Regional_Sources` with one row per metro the operator (you) wants to curate local data for. Schema:
    `region_match (city or state lowercase) | local_news_feeds_json | opendata_construction_url | opendata_311_url | warn_url`
  - At profile load, after geocoding, check if `Regional_Sources` has a row matching the user's city or state. If yes: merge those local feeds into `profile.newsFeeds` and store the optional OpenData URLs in `profile.regionalEnhancements`. If no row matches: just use national defaults. No errors, no special handling, no "unsupported region" warnings.
  - Customer can manually paste additional RSS URLs into setup wizard later.

  **Step 5 — OpenData fetchers become generic and optional:**
  - `fetch-construction-permits`, `fetch-renovation-permits`, `fetch-illegal-dumping`, `fetch-job-filings`, `fetch-warn-closures` all become parameterized: each reads its URL from `profile.regionalEnhancements` for the relevant key. If no URL set, the handler returns `[]` silently — no error, no fallback to NYC URLs.
  - The lead pipeline doesn't care whether OpenData returned data or not — it falls through to news + Google Places.

  **Step 6 — Chain blocklist universal + extensible:**
  - Add workbook tab `Chain_Blocklist`: a single column of business names that are chains anywhere in the world (Starbucks, McDonald's, Marriott, Walmart, Target, CVS, Walgreens, Hilton, Hyatt, IHG, Best Buy, Home Depot, Lowe's, etc.) — about 100 entries. Curate from publicly-known major chains, no NYC bias.
  - Customer can extend their personal blocklist via Settings → "Always exclude these companies" (free-text list, persisted to `profile.personalBlocklist`).
  - Filter logic merges global + personal blocklist. No regional-specific lists.

  **Step 7 — Sales type / vertical / product is fully workbook-driven (verify, don't rebuild):**
  - 35 sales types already in workbook ✅
  - Each vertical row already has buyer titles, buying triggers, priority score ✅
  - Verify the code reads ALL of this from workbook with no hardcoded fallbacks for any specific sales type.
  - Codex must not write `if (salesType === 'facilities_commercial_services_junk_sales') { ... }` anywhere. If you see this pattern, remove it.

  **Acceptance:**
  - Set up a fresh profile with city=Cincinnati, state=OH, salesType=medical_device_sales → lead pipeline runs successfully, returns 5-10 leads in Cincinnati for medical verticals (hospitals, clinics, etc.), uses national news feeds, skips OpenData silently (no Cincinnati URLs configured), no errors anywhere.
  - Set up another profile with city=Phoenix, state=AZ, salesType=janitorial_sanitation_sales → also works without any code changes.
  - Set up city=Toronto, state=ON (or province) → geocoding via Nominatim resolves it, Google Places returns Toronto businesses, national U.S. feeds run (or fail gracefully on relevance — that's fine, Places carries the load).
  - Grep all code: zero matches for `nyc`, `manhattan`, `brooklyn`, `1-800-GOT-JUNK`, hardcoded data.cityofnewyork.us URLs (the strings exist only as workbook data, never in source code).

- [ ] [claude→codex] **T49: Sales-type-driven Google Places type mapping in workbook.**
  Add a new column `places_types_csv` to the workbook `Master_B2B_Lead_Rules` tab. Each row's CSV lists Google Places `includedTypes` for that vertical:
  - "property management / multifamily" → `real_estate_agency,property_management_company,apartment_complex`
  - "hospitals" → `hospital`
  - "ambulatory surgery centers" → `medical_lab,doctor`
  - "general contractor / restoration" → `general_contractor,roofing_contractor,plumber`
  - "restaurants" → `restaurant`
  - "law offices" → `lawyer`
  - "schools / universities" → `school,primary_school,secondary_school,university`
  - "warehouses / logistics" → `warehouse`
  - "hotels" → `lodging`
  - "retail stores" → `clothing_store,shoe_store,store,furniture_store`

  Pre-populate for all 35 sales types in workbook. Rebuild lead-rules.json. Code in `main.js` reads `places_types_csv` per vertical instead of having a hardcoded mapping table. New sales type or vertical = just add a workbook row, no code changes.

  Acceptance: changing profile salesType from "facilities" to "medical_device_sales" → Google Places searches hospital/clinic types automatically, no code touched.

## Phase 2.5 — Google Places business discovery
_(Added 2026-05-12 — replaces DDG-based vertical search which produces 0 results. Spec in `claude-notes.md` § Phase 2.5.)_

- [ ] [claude→codex] **T43: Wire Google Places API (New) into `main.js`.**
  - Add `googlePlacesApiKey` field to setup wizard with helper tooltip linking to https://console.cloud.google.com/apis/library/places.googleapis.com — also add `region.latitude` + `region.longitude` fields (auto-resolved via Geocoding API on save).
  - New IPC handler `places-discover({vertical, region, radiusMiles, count})`.
  - Implement vertical → `includedTypes` mapping table per `claude-notes.md` § Phase 2.5 architecture #2.
  - Call `places:searchNearby` endpoint with proper FieldMask; return normalized `{name, address, phone, website, googlePlaceId, userRatingCount, primaryType, types[]}`.
  - Geocode `region.label` → lat/lng on profile save if missing, cache to profile.
  - Acceptance: with valid key, `places-discover({vertical:'property manager', region:{label:'New York City'}, radiusMiles:25, count:10})` returns 10 callable property management firms with phone + address.

- [ ] [claude→codex] **T44: Replace DDG vertical search with Google Places.**
  - In `fetchVerticalLeadsCore` (`main.js`), remove the `findVerticalLeadCandidates(query)` DDG path entirely for vertical-bucket leads.
  - Wire `places-discover` per workbook-selected vertical. For each enabled vertical (sorted by priority_score), call Places once with `count: 50`, filter through chain blocklist + 14-day exclusion + mid-size heuristic, take top `targetCount / verticalCount` from each.
  - Daily rotation: shuffle the post-filter pool using `Math.floor(Date.now() / 86400000)` as seed. Stable within a day, fresh next day.
  - News bucket (RSS+DOB+311+WARN) unchanged.
  - Seasonal bucket uses same `places-discover` infrastructure with vertical list derived from current month.
  - Fallback chain: if no API key OR HTTP 429, show banner + degrade gracefully per `claude-notes.md` § Phase 2.5 #8.
  - Acceptance: run leads pipeline → 7-10 vertical leads, no chains, no repeats from last 14 days.

- [ ] [claude→codex] **T45: Chain/franchise blocklist + mid-size filter.**
  - Implement `CHAIN_BLOCKLIST` in `main.js` per `claude-notes.md` § Phase 2.5 #3 — 30-40 known chains across verticals.
  - Reject candidates whose name matches blocklist (case-insensitive, partial match OK — e.g. "Marriott Marquis" matches "Marriott").
  - Add `sizeBucket` field per `claude-notes.md` § Phase 2.5 #4 — derived from `userRatingCount`. Boost mid-size in sort order.
  - Auto-learn: when the same business name appears 3+ times across customers in 30 days (future SaaS infra) add to dynamic blocklist. For now, mark `// SCALABILITY:` and stub.

- [x] [claude→codex] **T46: Re-verify and fix remaining Codex Update 5 tasks.**
  Codex reported done on T26-T32 but failed to mark them complete in `tasks.md`. Confirm each is actually working (especially T32 rolling exclusion — `lead-history.json` should exist and be enforced). Mark them with `[x]` after verifying. Then proceed with the previously-queued T33-T42 in order. **Do not claim completion on a task until you've manually tested its acceptance criteria.**

- [x] [claude→codex] **T42: Full QA pass — user functionality + bug hunt across Update 5.** _QA pass completed 2026-05-19; scenarios with OAuth/T48/product-scope limits are marked partial in `codex-notes.md`._
  After T26-T41 are individually marked done, run a full top-to-bottom user smoke test as if you were Alena using boola for a workday. Don't just check that code compiles — actually exercise every flow and capture what breaks. Report findings in `codex-notes.md` under a `## T42 QA pass` section.

  **Run these scenarios in order, document each result:**

  1. **Cold install path:** delete `app.getPath('userData')/profile.json`, relaunch. Setup wizard appears. Walk through every field including the new sales type / target verticals / territory radius / rep info / document templates section. Save. Confirm main app launches with mascot in the corner.

  2. **Lead generation:** open Leads tab → tap ↻. Confirm:
     - Progress strip shows "Validating leads… N/M checked, K valid"
     - Generation completes within ~90s
     - 10 distinct leads appear (count them)
     - Each has phone, vertical chip, confidence score, why-now line, suggested buyer
     - No "Hudson Yards / Marriott Marquis / NYU Langone" anywhere — fallbackPool is gone
     - Inspect `.boola_todays_leads.json` to verify schema

  3. **Lead repeat check:** delete `.boola_todays_leads.json` (force regen), tap ↻ again. Confirm at least 8 of the 10 leads are different from the previous run (rolling exclusion working).

  4. **Sales type swap:** open Settings, change sales type from "Facilities / Junk Removal" to "Tech / SaaS / IT", save. Tap ↻. Confirm leads now skew healthcare/financial verticals (per workbook priority scores), NOT property/construction.

  5. **Email pane:** generate a cold email using a real lead from the list. Verify:
     - Subject appears in a clearly-labeled subject row, body below in a separate area
     - Click Copy → paste into a fresh Gmail compose → paragraphs preserved, no `<br>` artifacts
     - Click Send → recipient gets a properly-formatted email with paragraph spacing (test by sending to your own address)
     - Verify auto-task appears in Tasks tab with correct due-date per email type

  6. **Todo flow:** add 3 todos. Check one off. Confirm:
     - Mascot celebrates (sparkles + happy face)
     - Item slides out of Open view
     - Appears in Done tab with completion timestamp
     - Un-check from Done → returns to Open, no celebrate

  7. **Documents tab:** upload a sample .docx with `{{name}}`, `{{job_address}}`, `{{discount}}`, plus one custom field like `{{service_tier}}`. During upload, confirm the field-type configuration step appears for the unknown placeholder (lets you pick dropdown/text/etc.). After save, open Documents tab, click the template:
     - Lead-data fields auto-fill from the active lead
     - Discount field shows 10/15/20/Custom buttons
     - Job address pre-fills from lead address but is editable
     - Click Download → opens in Word, all placeholders replaced, formatting intact

  8. **Document screenshot fill:** in the same fill dialog, switch to Screenshot mode. Paste a screenshot containing a name + email. Confirm:
     - Boola shows a brief "extracting…" state
     - Fields populate from the image
     - You can edit before downloading

  9. **Call mode:**
     - Click Start Listening. Mascot wears headphones + thinking face (or `thinking.png` fallback if asset missing — verify console message says so).
     - Speak: "Hey Mike, I'll send the quote to mike at acme dot com tomorrow morning. Talk to you later."
     - Within ~3 seconds of "talk to you later", boola auto-stops listening.
     - Confirmation card appears: "📧 Drafted follow-up email for mike@acme.com / 📋 Task added: Follow up in 2 days"
     - Mascot holds-email gesture (or fallback to `excited.png` with console message)
     - Click Send Now → email goes out, mascot celebrates, auto-task is marked replaced when T37's new task gets created
     - Verify the **"Pause auto-stop"** toggle appears in the call banner during listening. Default state: toggle OFF, which means auto-stop IS active. Flipping the toggle ON disables auto-stop so the user can talk through end-of-call phrases without boola cutting off.

  10. **Mic permission edge case:** revoke mic in System Settings → Privacy → Microphone, click Start Listening. Boola surfaces an explicit error message instead of silently failing.

  11. **Mascot rules:** confirm whale sits still in corner — no idle bounce, no random swims after 30+ minutes idle. Click whale → chat opens. Drag whale → it moves, position persists across relaunch.

  12. **Setup re-entry:** click Settings gear from chat sidebar. Setup wizard reopens with all fields pre-populated from current profile. Edit a field, save, confirm change persists.

  **Bugs to specifically watch for** (these have bitten before):
  - Multiple Electron instances running simultaneously after a relaunch (use `pgrep -fl boola` to verify only one)
  - Cache files not regenerating after profile changes
  - Mascot off-screen on multi-monitor setups
  - Network calls (DDG, Socrata) silently timing out and producing empty leads
  - Modal/window position bugs on different display sizes

  **For each scenario:** mark ✅ pass, ⚠️ partial (what's off), or ❌ fail (what broke). For any ⚠️ or ❌, include the exact reproduction steps and the file/line you suspect.

  Sync any final fixes to `~/BOOLA/`. When this task is done, Update 5 is complete.

- [x] [claude→codex] **T41: Autonomous call mode — mic listening + email extraction + auto follow-up.** _Code complete 2026-05-19; real Gmail Send Now round-trip deferred because OAuth is not set up on this Mac._

  Current state (audited 2026-05-11): basic Web Speech transcription works; on Start it tries to set `headset` expression but there's no `mascot-emotes/headset.png` so it falls back to `base.png`. No auto-stop, no email extraction from transcript, no auto-task creation. T41 builds the autonomous flow.

  **Mascot states needed (assets):** _Updated 2026-05-18 to match the canonical 13-emote sheet in `claude-notes.md` § Canonical mascot emote sheet._
  - **`thinking-headset.png`** (canonical name — NOT `headset-thinking`) — boola wearing black over-ear headphones with thinking face. Used during active listening. Maps to canonical emote #11 "Thinking (Headset)".
  - **`decision.png`** (canonical name — NOT `holding-email`) — boola holding 📧✅ and 📧❌ envelopes. Used after call ends to show the "send follow-up?" prompt. Maps to canonical emote #7 "Decision".
  - As of 2026-05-18, canonical PNGs have not yet been generated at their canonical filenames. If `thinking-headset.png` is missing, fall back to `thinking.png` with a one-line console log. If `decision.png` is missing, fall back to `happy.png` with a one-line console log. No silent failures.
  - Update the expression-mapping table in `chat.html` (lines ~385-400) to include both canonical keys (`thinking-headset` and `decision`).

  **Workflow (replace existing `startCallMode` / `stopCallMode`):**

  1. **User clicks Start Listening** → mascot fires `set-expression: thinking-headset`. Boola is silent and listening.

  2. **Continuous transcription** via Web Speech API (already implemented; verify it actually captures by speaking a known phrase and checking the live transcript element renders it. If it doesn't, surface the macOS mic-permission error explicitly instead of just `console.log`).

  3. **Auto-stop on call-end phrases.** On every final transcription chunk, check the last ~6 seconds of text against this regex:
     ```js
     const END_CALL_PHRASES = /\b(bye|goodbye|talk to you (later|soon)|have a (good|great) (one|day|night|weekend)|catch you later|appreciate (it|your time)|take care|thanks (for your time|so much)|talk soon|alright thanks)\b/i
     ```
     When matched, wait 2 seconds (in case more speech follows), then trigger `stopCallMode()` automatically.

  4. **User can also click Stop Listening manually** (existing button stays).

  5. **On stop (auto or manual):**
     - Mascot fires `set-expression: thinking` while boola processes the transcript.
     - Send the full transcript to Claude with this extraction prompt (new function `extractCallContext(transcript)`):
       ```
       The following is a one-sided transcript of a sales call — only the sales rep's voice. Extract structured info from it as JSON:

       {
         "recipient_email": "string or null — parse spelled-out emails like 'john dot doe at gmail dot com' into 'john.doe@gmail.com'",
         "recipient_phone": "string or null",
         "recipient_name": "string or null — anyone the rep addressed by name",
         "company_mentioned": "string or null",
         "commitment_made": "string or null — what the rep promised to do (send email, call back, etc.)",
         "followup_date_mentioned": "ISO date string or null — explicit date/time the rep promised",
         "summary": "2-sentence summary of what was discussed",
         "rep_pain_points_heard": ["string array — pain points the prospect mentioned (rep would have repeated/acknowledged them)"],
         "suggested_email_type": "one of: cold | followup1 | aftercall | proposal | won — best fit for the conversation"
       }

       Return ONLY valid JSON.
       ```
     - If `recipient_email` is non-null, auto-draft a follow-up email using `getEmailExpertSystem` + the existing `EMAIL_TYPE_GUIDE[suggested_email_type]`, with the body referencing `commitment_made` and `rep_pain_points_heard`.
     - Auto-create a follow-up task for **2 days from now** using the existing todo system. Task text: `"Follow up with {recipient_name or recipient_email} — {summary}"`, priority `medium`. Tag the task with `source:'call-auto'` so it's distinguishable from manual todos.
     - Mascot fires `set-expression: decision`. Hold this expression for 4 seconds, then return to `happy`.

  6. **Post-call UI in the call pane:**
     - Replace the existing "Callback follow-up detected / Email follow-up detected / Call notes captured" branches with a single result card:
       ```
       📧 Drafted follow-up email for {recipient}
       📋 Task added: "Follow up in 2 days"
       [Preview Email] [Send Now] [Edit]  [Cancel both]
       ```
     - **Preview Email** opens the existing email-result UI populated with the drafted message.
     - **Send Now** fires `send-email` IPC immediately (no edit step). Mascot fires `celebrate`.
     - **Edit** routes to the Email pane with the draft pre-filled.
     - **Cancel both** removes the auto-task and discards the draft.

  7. **If `recipient_email` is null** (no email captured during call):
     - Skip the email draft. Still create the task.
     - Result card: `"📋 Task added: 'Follow up — {summary}'  [View in Tasks] [Cancel]"`.

  **Integration with T37 (auto-task on email send):**
  - When the auto-drafted email from T41 is actually sent, T37's logic still fires — but the original 2-day task should be marked as "satisfied" (struck through, moved to Done) since sending the email IS the follow-up. T37 then creates the NEXT follow-up at the appropriate cadence.
  - Implementation: tag T41's auto-task with `replacedBy: emailId` when its email is sent.

  **Acceptance:**
  - Click Start Listening → mascot wears headphones + thinking face (or `thinking.png` fallback if asset missing).
  - Speak a test sentence like "I'll send the quote to mike at acme dot com later today" → on stop, draft appears addressed to `mike@acme.com`.
  - Say "alright, talk to you later, bye" at the end of a test session → boola auto-stops within 3 seconds.
  - Verify auto-task exists in Tasks tab with due-date 2 days out.
  - Click Send Now → mascot celebrates, email goes out, task moves to Done.
  - Test mic-permission error path: revoke mic in System Settings → click Start Listening → boola surfaces "Microphone access denied — open System Settings → Privacy → Microphone" instead of silently failing.

  Sync to `~/BOOLA/` when done.

- [x] [claude→codex] **T38: Todo completion = celebrate + hide from active view.**
  In `chat.html` `toggleTodo(i)`:
  1. When a todo transitions from `done:false` → `done:true`, fire `ipcRenderer.send('celebrate')` (existing IPC — triggers mascot celebrate expression + sparkles). Do not fire on un-toggle.
  2. Filter behavior:
     - **All** and **Active** filters: hide any todo where `done:true`. Currently "All" shows everything; change so "All" shows active + high-priority but not done.
     - **Done** filter: shows ONLY completed todos (already works).
     - **High** filter: still hides done items.
  3. Animate the line item out before it disappears — opacity 0 + slide-right over 350ms — so it doesn't just pop out abruptly. Then re-render the list.
  4. Updated filter labels for clarity:
     - "All" → "Open" (since done is now hidden by default)
     - keep "Active" / "🔴 Urgent" / "Done"
  5. Done view shows completion timestamp under each task. Add `completedAt` to the todo schema when marking done. Show as "✓ completed {relative time}" — e.g. "✓ completed 2h ago".
  Acceptance: check off a todo → mascot celebrates → item slides out → reappears in Done tab with timestamp. Un-toggle from Done tab → item returns to Open tab, no celebrate.

- [x] [claude→codex] **T37: Auto-task on email send.** _Code complete 2026-05-19; real Gmail send-test deferred because OAuth is not set up on this Mac._
  After `send-email` IPC succeeds, automatically:
  1. Append entry to `~/Library/Application Support/boola/sent-emails.json`: `{to, subject, type, sentAt, leadId?, taskId?}`
  2. Compute follow-up date based on email `type`:
     - `cold` → +3 days
     - `followup1` → +6 days
     - `followup2` → +10 days
     - `proposal` → +5 days
     - `aftercall` → +2 days
     - `social-proof` → +4 days
     - `breakup` → +60 days
     - `won` → no task
  3. Auto-create a todo via the existing todo system with text `"Follow up with {recipient name or email} re: {subject}"`, priority `medium`, due-date set per the table.
  4. Store the created `taskId` back on the sent-email ledger so we can cancel it later (Phase 3 with Gmail read).
  5. Add a small toast/notification in chat: "📋 Auto-added follow-up task for {recipient} on {date}". Dismissable.
  6. Add a "Sent emails" history view (small) reading from `sent-emails.json` — sortable by date, shows which ones still have open follow-up tasks. Place it in chat sidebar under "Tasks".
  Files affected: `main.js` (ledger persistence in send-email handler), `chat.html` (auto-task creation, toast UI, sent-emails view).
  Acceptance: send a test email, verify a new task appears in Tasks tab with correct due-date, verify entry appears in sent-emails.json, verify toast shows.

---

- [ ] [claude→codex] **T25: Restore the floating desktop mascot.**
  T23's previous implementation went too far — it removed the mascot window entirely and shoved the whale into the chat header. That violates the new core rule (`claude-notes.md` § Mascot is the product). Bring back the floating mascot window with full original functionality, MINUS the idle bouncing and random swim moments.

  **Restore:**
  1. **`main.js`** — re-enable `createMascotWindow()` and add it back to `createMainWindows()`. Position bottom-right of primary display (existing code is correct: `x: width-160, y: height-160`).
  2. Re-enable click-to-toggle-chat IPC handler (`toggle-chat` from mascot to main, then show/hide chatWindow).
  3. Re-enable dragging (`move-mascot` IPC handler that sets mascot bounds).
  4. Restore the chat-positioning-relative-to-mascot behavior (`chatWindow.setPosition(b.x - 290, b.y - 700)` where `b` is the mascot bounds) — that was the original anchor.
  5. Mascot uses the new emote PNG system (`./mascot-emotes/base.png` as default) — keep what Codex built for expressions, just put it back in the floating window.

  **Keep removed (what Alena actually asked to remove):**
  - Idle bobbing animation (`@keyframes bob` and any `animation: bob …` defaults). Mascot sits perfectly still.
  - Auto-firing `startRandomExpressions()` loop. No timer-driven silly moments. The function and `doSillyMoment()` can stay callable from the manual "🎭 Test Mood" button — just no scheduled fires.

  **Keep working:**
  - Manual `set-expression` IPC events still swap the avatar PNG (thinking/excited/headset/celebrate/sleepy/etc.)
  - Sparkles/celebrate effects on user-triggered events (close button success, manual celebrate)
  - Task reminder pop-up (9am/12pm/3pm) and 9:30am leads pop-up — both keep working untouched

  **Acceptance:**
  - After relaunch, the floating Boola whale appears in the bottom-right corner of the primary display
  - Whale sits still — no bobbing, no drift
  - Clicking the whale toggles the chat panel
  - Dragging the whale moves it; chat re-positions relative to the new whale location when next opened
  - 45+ min later, no auto-fired animations occur
  - Test Mood button still triggers expressions
  - Chat header avatar (added by Codex's last pass) can stay or be removed — Alena's call. Default: keep it; it's nice having the avatar visible inside the chat too.

  Sync to `~/BOOLA/` when done. Update `codex-notes.md` confirming the rule from `claude-notes.md` § Mascot is the product is now respected.

- [ ] [claude→codex] **T23: Calm down the mascot — kill random animations + idle bouncing.**
  Alena finds the constant float-bouncing distracting and the random "swim to a stage spot, do a silly face, swim back" too disruptive. Make boola stagnant by default. Keep timed task reminder + leads pop-ups (those are window pop-ins, not mascot motion — leave alone).

  **Specific changes:**

  1. **`main.js`** — disable the `startRandomExpressions()` auto-fire loop:
     - Find `function startRandomExpressions()` (~line 260) and either delete the call site (`createMainWindows()` invokes it) or no-op the function body. Keep `doSillyMoment(expr)` as a function so it's still callable by the "🎭 Test Mood" button — just stop the timer.
     - The `RANDOM_EXPRS` array, `swimTo()`, `getRandomStage()`, and `doSillyMoment()` themselves can stay (they're useful for the manual Test Mood button).

  2. **`mascot.html`** — kill the idle bouncing:
     - Find `@keyframes bob` (~line 374). Either delete it or make it a no-op (`0%,100%{transform:none}`).
     - In every `set-expression` handler that sets `boola.style.animation = 'bob …'`, change to `boola.style.animation = 'none'`.
     - The `swim` keyframe can stay — it's only used during transitions (manual Test Mood) and the leads-pop-up swim, which Alena wants kept.
     - The `swimming` IPC handler (line 444) — when called with `false`, set animation to `'none'` instead of `'bob 2s ease-in-out infinite'`.

  3. **Do NOT touch:**
     - `fireReminder()` / `animateReminderIn` / `animateReminderOut` (todo pop-up — keep)
     - `fireProspect()` (leads pop-up — keep)
     - `prospect.html` (leads pop-up has its own swim animation — keep)
     - The sparkles/money-throw effects on `celebrate` IPC events (those are intentional, user-triggered)

  **Acceptance:**
  - After relaunch, boola the whale sits perfectly still in the corner. No bouncing, no drift.
  - 45-75 min later, no random "swim across screen" event fires.
  - Tasks tab still pops the reminder window at scheduled times.
  - Leads tab still pops at 9:30 AM.
  - Clicking "🎭 Test Mood" in chat still triggers a one-off expression change (mascot face changes, no movement to a stage).

  Sync to `~/BOOLA/` when done.

- [ ] **T22: Mascot + brand identity redesign.** _DEFERRED 2026-05-18 — superseded by the canonical 13-emote sheet (see `claude-notes.md` § Canonical mascot emote sheet). The original brief at `~/Downloads/boola_mascot_redesign_brief_for_codex.md` is not on this Mac and is no longer the source of truth. Do NOT work T22 in Update 5. Mascot art is handled separately by Alena generating 13 transparent-bg PNGs at canonical filenames._
  Read the design brief at **`~/Downloads/boola_mascot_redesign_brief_for_codex.md`** — that's the canonical visual spec. Then read `claude-notes.md` § Mascot redesign for boola-specific implementation requirements (existing expression system, animation hooks, wordmark application, dock icon, etc.).
  Files to touch:
  - `mascot.html` — replace existing SVG with new baby-shark design; preserve `swim` keyframe + `set-expression` IPC handler with expression variants (`happy`, `thinking`, `excited`, `headset`, `celebrate`, `sleepy` minimum).
  - `prospect.html` — replace the "swimming whale holding scroll" SVG with new design; keep the swim animation.
  - `setup.html`, `chat.html` — replace plain-text "Boola" headers with the wordmark + wave underline (use SVG/Pacifico/Lobster/Fredoka per brief).
  - `main.js` — generate `icon.icns` (1024×1024 → iconutil) and apply via `app.dock.setIcon()` at startup. If iconutil generation needs to happen outside the app, document the build step in `codex-notes.md`.
  Acceptance:
  - Whale on screen matches the brief at first glance (baby shark, blush, big eyes, fin-not-arms, bubbles).
  - All expression states still trigger from chat ("test mood" button cycles through them).
  - Leads pop-up still shows the whale swimming up holding the scroll, in new style.
  - Setup wizard shows the branded wordmark.
  - macOS dock icon shows the new mark when boola is running.
  Sync to `~/BOOLA/` when done.

- [ ] [claude→codex] **T21: Remove API key field from setup wizard** per `claude-notes.md` § Anthropic API key. Specifically:
  1. Delete the "Anthropic API key" label, input, and any related form code from `setup.html`.
  2. Drop `apiKey` from the saved `profile.json` schema (don't write it; ignore on read).
  3. Continue loading the key from `~/.boola_key` silently in `chat.html` and anywhere else it's used. If `~/.boola_key` is missing, log to console and degrade gracefully (chat shows "Boola is offline — admin setup required" instead of asking the customer for a key).
  4. Add `// SCALABILITY: route through boola backend in production` comment at every `fetch('https://api.anthropic.com/v1/...')` call site in `chat.html` and any other file.
  5. Sync to `~/BOOLA/`.

- [ ] [claude→codex] **T19: Smoke test the new pipeline.**
  1. Delete `app.getPath('userData')/profile.json` if present
  2. Relaunch boola — confirm setup wizard appears
  3. Save with defaults — confirm profile.json is written and main app launches
  4. Open Leads tab → tap ↻ — confirm leads come from action-based sources (no "newly licensed" reasons appear)
  5. Confirm at least one lead from each enabled signal type appears in `~/.boola_todays_leads.json` (or note in codex-notes if any signal yields zero on the test day)
  6. Open Settings → uncheck `illegal-dumping` → save → tap ↻ — confirm no dumping leads appear
  7. Record findings in `codex-notes.md`

- [x] [claude→codex] **T20: Sync to project folder** — _obsolete as of 2026-05-18: canonical folder is now `~/BOOLA/`, no separate sync step._

## Update 6 — Rep Customization Pass (2026-06-06)

_(Added after Update 5 landed. Goal: let every rep make Boola theirs — own email templates, own UI layout, own button choices — with zero code changes. Universal-SaaS compliant: works for any customer's preferences, persisted to `profile.json` so customization survives reinstall and follows the rep.)_

**Why this is one package, not many small updates:** the rep-customization vision needs the storage schema, settings UI, and the per-tab edit affordances to ship together — partial rollout is confusing UX. Build all of T50-T53, QA as a unit, ship once.

- [ ] [claude→codex] **T50: Custom email types — create, label, save, edit, delete.**
  Reps create their own email templates beyond the built-in 7 (cold, followup1, followup2, social-proof, in-area, breakup, aftercall, won). Each rep's templates live in `profile.json` under `customEmailTypes` and survive reinstall.

  **Schema** (`profile.json` → `customEmailTypes`):
  ```js
  customEmailTypes: [
    {
      id: 'cet-1717689600000-abc',     // unique, generated at create time
      label: 'Quick Pic Quote',         // shown in dropdown as "★ Quick Pic Quote"
      systemPrompt: 'You write...',     // the guidance / template body
      allowVariance: true,              // see modes below
      followUpDays: 2,                   // for T37 auto-task cadence; default 3
      createdAt: '2026-06-06T...',
      lastEditedAt: '2026-06-06T...'
    }
  ]
  ```

  **Two modes — toggle on create:**
  1. **`allowVariance: true` (default)** — the `systemPrompt` field becomes Claude's guidance. Claude adapts wording per-lead using context (lead name, company, address, vertical, etc.). Best for tactics that need personalization.
  2. **`allowVariance: false`** — the `systemPrompt` field is treated as a literal template with `{{placeholder}}` tokens. Boola fills placeholders from lead + profile data and outputs verbatim. NO AI rewriting. Best for high-volume reps who have a script that works and don't want it altered.

  **Placeholder support (allowVariance: false mode)** — same token set as T40 document templates:
  - `{{lead_name}}`, `{{lead_company}}`, `{{lead_address}}`, `{{lead_phone}}`, `{{lead_vertical}}`
  - `{{rep_name}}`, `{{rep_email}}`, `{{rep_phone}}`, `{{rep_company}}`, `{{rep_title}}`
  - `{{job_address}}` (editable in fill dialog), `{{discount}}`, `{{date}}`, `{{week_days}}` (e.g. "Monday through Thursday")
  - Any unknown `{{xyz}}` token prompts the user at fill time.

  **UI — Email pane:**
  - Add `+ New Type` button next to the Email Type dropdown.
  - Click opens a modal:
    - Label (text, required, max 30 chars)
    - Template body (textarea, required)
    - Toggle: "Allow Boola to adapt this for each lead?" (default ON) — label changes to "Boola adapts wording per lead" / "Send exactly this template (placeholders only)"
    - Number input: "Follow-up task in N days after send" (default 3)
    - Live preview pane on the right showing how the type renders against the currently-selected lead (pulls real lead context, shows what placeholders would resolve to)
    - Save / Cancel
  - On save: appends to `customEmailTypes`, dropdown re-renders, new type is auto-selected.
  - Dropdown rendering: built-in types in one group, custom types prefixed with `★` in a second group separator.
  - Each custom type in dropdown gets an inline `✏️` icon (visible on hover) that re-opens the modal in edit mode.
  - Delete affordance lives in the edit modal (with a confirm dialog).

  **Send path (`generateEmail` in `chat.html`):**
  - If selected emailType key starts with `cet-`, look up the custom type in `profile.customEmailTypes`.
  - If `allowVariance: true` → use the custom type's `systemPrompt` as the email-type guidance block, call `callClaude(getEmailExpertSystem(), prompt, [], false, true)` — same flow as built-in types.
  - If `allowVariance: false` → skip the Claude call entirely. Run the template body through `fillPlaceholders(template, leadCtx, profileCtx)`. Parse the result with the existing `parseEmailDraft()`. Render directly.
  - Either way, T37 auto-task creation uses the custom type's `followUpDays`.

  **Migration:** existing profiles with no `customEmailTypes` field treat it as `[]`. No breaking changes.

  **Acceptance:**
  - Create a custom type "Quick Pic Quote" with allowVariance:false using Alena's exact gold-standard copy as the template body. Send against any lead → email body matches verbatim except placeholders filled.
  - Create a second type "Holiday Push" with allowVariance:true and a 1-paragraph guidance. Send against two different leads → wording differs but both follow the spirit.
  - Edit the label of a custom type → dropdown updates immediately.
  - Delete a custom type → it disappears from dropdown, profile.json no longer contains it.
  - Restart Boola → custom types persist.

- [ ] [claude→codex] **T51: UI Layout Editor — tabs + action buttons (iPhone-style edit mode).**
  Reps choose which tabs to show in the chat strip, in what order, and which action buttons within each tab to show. All persisted to `profile.json` under `uiLayout`. Hidden items aren't deleted — they sit behind a "..." overflow menu so the rep can re-enable later.

  **Schema** (`profile.json` → `uiLayout`):
  ```js
  uiLayout: {
    tabs: ['leads', 'chat', 'tasks', 'documents', 'email', 'calls'],  // ordered, visible
    hiddenTabs: [],                                                     // moved-out, not deleted
    actionButtons: {                                                    // per-context button order
      leadCard:   ['cold-email', 'in-area', 'linkedin', 'zoominfo'],
      emailType:  ['cold', 'followup1', 'followup2', 'in-area', 'aftercall', 'won', 'breakup', 'social-proof'],
      // ... extensible per surface
    },
    hiddenButtons: { leadCard: [], emailType: [], ... }
  }
  ```

  **Edit mode UX (iPhone home-screen analogy):**
  - In **Settings → Layout** there's a big `✏️ Edit Layout` button.
  - Click → top of the screen overlays an edit-mode banner: "Drag to reorder. Tap × to hide. Tap + to restore."
  - All tab chips become draggable (long-press or click+hold on desktop). Each gets a small `×` in the top-right corner.
  - Same affordance applies to button rows inside the Leads tab (per-card action buttons) and the Email Type dropdown (re-shown as a horizontal chip row in edit mode for easier editing).
  - Hidden items collapse to a `+ Show hidden` strip at the bottom.
  - `Done` button in the banner exits edit mode and persists changes to `profile.json`.

  **Built-in vs custom:**
  - Built-in tabs (Chat, Leads, Tasks, Documents, Email, Calls) can be hidden but not deleted (data preservation).
  - Custom email types (from T50) can be hidden from the dropdown too.

  **Rendering:**
  - On chat.html load: `renderTabs()` reads `uiLayout.tabs` and renders only those, in that order. If `uiLayout` is missing or empty, fall back to the default tab order.
  - Same for action button surfaces: each render site consults `uiLayout.actionButtons[contextKey]` for ordering, falls back to the default order if not present.
  - Hidden items live behind a small `⋯` overflow menu at the end of each strip — clicking it lists hidden items with "Show" buttons. So the rep never gets locked out.

  **Default order (used when `uiLayout` is missing):**
  - Tabs: `chat, leads, tasks, documents, email, calls`
  - Lead card actions: `cold-email, in-area, linkedin, zoominfo`
  - Email types: `cold, followup1, followup2, in-area, aftercall, social-proof, breakup, won`

  **Acceptance:**
  - Open Settings → Edit Layout. Drag Tasks above Leads → reorder visible immediately. Restart Boola → order persists.
  - Hide the Calls tab → it disappears from the strip, shows under "⋯" overflow.
  - Re-show Calls from the overflow → it returns to the end of the strip.
  - On the Leads tab, hide the LinkedIn button → it disappears from every lead card. Restart → still hidden.
  - On the Email Type dropdown, hide "Break-Up Email" → no longer in the picker. Re-show → returns.
  - Wipe `profile.json` `uiLayout` → app falls back to default order cleanly, no errors.

- [ ] [claude→codex] **T52: Settings re-organization (central customization hub).**
  Settings becomes the rep's one place to make Boola theirs. Currently the wizard mixes one-time setup (sales type, region, rep info) with ongoing config (templates). Restructure into clearly-labeled sections.

  **New Settings layout (top to bottom):**
  1. **You** — rep_name, rep_email, rep_phone, rep_title, rep_company (T40 fields)
  2. **Sales setup** — sales type dropdown, target verticals checkboxes, territory radius
  3. **Email templates** — list of all email types (built-in + custom). Each row: label, "Edit"/"Hide"/"Delete" actions (Delete only on custom). `+ New Type` button at bottom calls T50 modal.
  4. **Document templates** — existing T40 template list, unchanged behavior
  5. **Layout** — `✏️ Edit Layout` button → T51 edit mode
  6. **Mascot** — placeholder for future preferences (idle behavior, etc.) — leave empty for now
  7. **Integrations** — Gmail OAuth status ("Connected as alenatipton111@gmail.com" / "Connect" button), ZoomInfo status, etc.
  8. **Advanced** — Anthropic API key field (visible only if `~/.boola_key` is missing), reset settings button

  **Implementation notes:**
  - Each section collapsible (chevron toggle)
  - Section anchors so Boola can deep-link ("open Settings → Email templates")
  - "Save" button at the bottom commits all changes atomically (one `profile.json` write)
  - "Cancel" reverts unsaved changes
  - If the user closes Settings with unsaved changes → confirm prompt

  **Acceptance:**
  - Open Settings → see 8 clearly-labeled sections in order
  - Each section collapses/expands smoothly
  - Editing in one section + cancel → no changes persist
  - Editing in multiple sections + save → all changes persist atomically

- [ ] [claude→codex] **T54: Documents tab upgrades — in-tab upload, Desktop save, screen capture, friendly placeholder syntax.**
  T40 (Update 5) shipped 90% of the document templates feature. T54 closes the remaining gaps Alena flagged.

  **A) Upload button in Documents tab itself** (currently upload is only in Settings — drives reps out of flow):
  - In `chat.html` pane-documents at line ~328, add a primary action at the top:
    ```
    [📎 Upload Template]  [⚙️ Manage Templates]
    ```
  - "📎 Upload Template" opens a file picker (same as Settings flow): drag-drop or click, .docx only.
  - On upload, run the existing `template-add` IPC → placeholder scan → field-config wizard (existing T40 modal — show inline below the template list, not in setup wizard).
  - On save, the new template card appears immediately in the list.
  - "⚙️ Manage Templates" deep-links to Settings → Document templates section (T52's new Settings re-org).

  **B) Filled docs save to ~/Desktop with smart naming** (currently lands in `/var/folders/...` system tmpdir — invisible):
  - In `main.js`, replace `outputDocxPath(templateId)` (line ~1682) so it returns a path under `app.getPath('desktop')` instead of `os.tmpdir()`.
  - Filename pattern: `Boola - {TemplateName} - {LeadCompany} - {YYYY-MM-DD}.docx`
    - Strip illegal characters (`:/\?*"<>|`) to spaces.
    - If lead is unknown, omit the lead segment.
    - If a file with the same name exists, suffix `-2`, `-3`, etc.
  - On successful fill, surface a small toast in chat: "📄 Saved to Desktop: {filename}" with a `[Open]` button (calls existing `template-open-file` IPC).
  - Keep the `[Send as email attachment]` flow working — that one still uses a buffer in memory, doesn't need the Desktop file.

  **C) Live on-screen capture** (currently only Cmd+V from clipboard works for Screenshot mode):
  - Add two new buttons in the Screenshot mode panel:
    - **`📸 Capture region`** — opens an Electron `desktopCapturer` overlay. Screen dims, user drags a rectangle. On release, that region is captured as a PNG, base64-encoded, sent through `extractFromImage()` (existing Claude vision path), fills fields.
    - **`🖥️ Capture full screen`** — one-click full-display screenshot (using `desktopCapturer.getSources({types:['screen']})`), same downstream path.
  - The existing "Paste from clipboard (Cmd+V)" affordance stays as a third option.
  - Implementation note: region selection requires a transparent always-on-top BrowserWindow overlay with a custom canvas selector. New file `lib/screen-region-picker.js` is a clean place. Wire via new IPC `desktop-capture-region` (returns base64 PNG of selected region) and `desktop-capture-full` (returns base64 PNG of primary display).
  - macOS permission: app must request "Screen Recording" permission via `systemPreferences.askForMediaAccess` or graceful fallback. If denied, show "Boola needs Screen Recording permission in System Settings → Privacy → Screen Recording" and abort.

  **D) Natural-language placeholder syntax** (in addition to `{{name}}`):
  - Update `template-scan-placeholders` IPC handler in `main.js` (line ~1700) and the renderTemplateDocx fill logic to ALSO accept `(insert NAME here)` style — case-insensitive `(insert ([a-z_]+) here)` regex.
  - Both formats coexist in the same template; both extract to the same fieldConfig.
  - Internally, normalize to the `{{tokens}}` form before passing to docxtemplater (which only speaks the curly-brace syntax).
  - Document this in the upload modal's helper text: "Use `{{name}}` or `(insert name here)` — both work."

  **E) Lookup-table field type (for discount-driven rate tables):**
  Real-world templates like Alena's Royal Blue Agreement have a price table where the rep picks a discount level and an entire column of prices auto-populates. Currently T54 would force the rep to fill each rate cell manually. Add a new fieldConfig type:

  ```js
  fieldConfig: {
    rates_column: {
      type: 'lookup',
      label: 'Discounted rate column',
      indexBy: 'discount',          // name of the placeholder whose value drives the lookup
      table: {                       // keyed by indexBy value, list of strings
        '10% Off': ['$259', '$493', '$709', '$880', '$1,051'],
        '15% Off': ['$245', '$466', '$670', '$831', '$993'],
        '20% Off': ['$230', '$438', '$630', '$782', '$934']
      },
      outputFormat: 'lines'         // 'lines' = join with newlines for multi-row cell; 'csv' = comma-separated
    }
  }
  ```

  At fill time, if `discount` resolves to "15% Off", `rates_column` fills with the joined string `"$245\n$466\n$670\n$831\n$993"`. Word's table cell renders each on its own line.

  **Configuration UI:**
  - In the upload field-config wizard, when the user picks "Lookup table" as the type, present a grid editor: column = lookup key (auto-populated from the dropdown values of the `indexBy` field), rows = expandable.
  - User pastes in values from a spreadsheet, hits Save.
  - Once configured, the rep never sees the table during fill — only the discount picker. Magic.

  This is template-specific data — lives in `templates.json` per-template. NOT in Boola's source code. Universal-SaaS compliant: every customer can configure their own lookup tables.

  **Test template (synthetic, Codex generates during implementation):**
  - Codex creates a generic estimate.docx with these placeholders for testing:
    - `(insert CLIENT_NAME here)` / `(insert COMPANY here)` / `(insert ADDRESS here)` / `(insert JOB_ADDRESS here)`
    - `{{discount}}` / `{{total}}` / `{{date}}`
    - One custom field: `{{service_type}}` (dropdown: Standard / Premium / Same-Day)
  - Mixed syntax intentionally to validate both parsers.

  **Acceptance:**
  - Click 📎 Upload Template inside Documents tab → file picker → pick the test estimate.docx → scan finds 8 placeholders (4 natural-language, 4 curly-brace). Field-config wizard appears inline. Save.
  - Open the template → fill mode "Lead data" → placeholders pre-fill. Click Download → file lands at `~/Desktop/Boola - Estimate - {LeadCompany} - {today}.docx`. Open in Word → all placeholders replaced, formatting intact.
  - Switch to Screenshot mode → click 📸 Capture region → drag a rectangle over a webpage showing prospect details. Boola extracts the fields and fills the form (mocked Claude response acceptable in harness).
  - Same again with 🖥️ Capture full screen.
  - Paste image via Cmd+V still works.
  - Toast appears after each successful download with [Open] button that opens the file in Word.

  Sync ai-notes git commit.

## Update 7 — Personality + Workflow Pass (2026-06-06)

_(Goal: make Boola feel like a coworker rep would miss. Funny mascot reactions on real actions, sharper reminder UX, inbox triage, exportable call lists. Builds on Update 6 — should ship after T50-T54 land.)_

- [ ] [claude→codex] **T55: Funny mascot animations + speech bubbles (action-triggered only).**
  Boola gets a personality layer that fires on real user actions — NEVER on idle timers (the "Mascot is the product" rule in claude-notes.md bans idle animations).

  **Triggers (and their canonical emote + bubble category):**
  | User action | Mascot emote | Bubble category |
  |---|---|---|
  | Email Send succeeds | `cooking-money` for 2s → `happy` | `sent_email` |
  | Cold email send (first contact) | `excited` for 2s → `happy` | `cold_first_contact` |
  | Todo marked done | `celebrating` for 2s → `happy` | `task_complete` |
  | Generate Leads completes (≥8 callable) | `excited` + sparkles | `leads_loaded` |
  | Won email type sent | `cooking-money` for 4s + rain-money | `deal_won` |
  | Lead added to call list | `focused` for 1.5s → `happy` | `call_list_add` |
  | Call Mode start | `thinking-headset` (stays during call) | `call_start` |
  | Call Mode end | `decision` for 4s → `happy` | `call_end` |
  | Hit 5+ todos done in a day | `cool` for 3s | `streak_5_todos` |
  | 9:30 AM prospect fire | existing celebrate behavior + bubble | `morning_greeting` |
  | App relaunch / wake from sleep | `tired` for 1s → `happy` + bubble (only first time per day) | `welcome_back` |

  **Frequency dampening:** to avoid bubble fatigue, fire bubbles with **1-in-2 probability per qualifying event**. Always fire the emote. The bubble is the cherry on top — should feel like Boola is sometimes commenting, sometimes silent. Track in `app-state.json` so the rate persists across launches.

  **Speech bubble component** (`lib/speech-bubble.js`, new module — packageable):
  - Floating absolutely-positioned `<div>` above the mascot window
  - Tail pointing toward mascot
  - Fade in 200ms, hold 3 seconds, fade out 400ms
  - Style: rounded white card with soft shadow, dark text, max 3 lines
  - Stacks if multiple bubbles fire in succession (max 2 visible, newer pushes older out)
  - Click bubble to dismiss early

  **Content library** lives at `~/BOOLA/personality.md` (Claude is writing this now — Codex consumes it during implementation). Each category has 10-20 quips. Boola loads the file at startup, parses into a map keyed by category, picks a random quip when a bubble fires.

  **API:**
  - `mascotWindow.webContents.send('speech-bubble', { category: 'sent_email' })` from main.js
  - mascot.html reads the bubble, picks a random quip from the loaded library, renders the bubble

  **Files affected:**
  - `lib/speech-bubble.js` (new)
  - `mascot.html` — add bubble container + IPC listener + load personality.md on init
  - `main.js` — fire `speech-bubble` IPC at trigger points (after appendSentEmailLedger, after toggle-todo done, after leads-ready, etc.)
  - `personality.md` (new content file at BOOLA root)

  **Acceptance:**
  - Send a real email → mascot fires cooking-money emote + ~50% of the time a bubble appears like "💸 Filled the truck."
  - Mark a todo done → celebrating emote + ~50% chance of "One down. Who's next?"
  - Win-type email send → 4s cooking-money + sparkles + bubble like "🍾 That's how it's done."
  - 30 minutes of idle laptop → no random bubbles fire (idle rule preserved)
  - Bubbles never repeat the same quip twice in a row within a category (track last-fired quip per category in-memory)

- [ ] [claude→codex] **T56: Replace legacy mascot in reminder popup with canonical emote system.**
  `reminder.html` currently shows the old SVG mascot or a legacy emote token (audit at impl time). Replace with the canonical 13-emote PNG system used everywhere else.

  **Change:**
  - `reminder.html` mascot element → use `./mascot-emotes/excited.png` as default for reminder fires
  - On render, listen for the `set-expression` IPC the way mascot.html does (use the same fallback rules from claude-notes.md if the emote PNG isn't present)
  - If reminder is "morning" (10am) → use `excited`. Afternoon (1pm) → `focused`. Evening (3pm) → `cool`.

  **Acceptance:**
  - Fire a manual reminder (`fireReminder()` test) → mascot shown is canonical PNG, not legacy
  - Reminders at 10/13/15 show different emotes per the table above
  - Missing PNG falls back to `happy.png` with one console log line, not silent failure

- [ ] [claude→codex] **T57: Re-add `gmail.readonly` scope + OAuth re-auth flow.**
  Email triage (T58) needs inbox read access. Path A1 dropped `mail.google.com` in favor of `gmail.send` — adding `gmail.readonly` alongside keeps verification on the "sensitive scope" track (no CASA audit needed).

  **Code changes:**
  - `scripts/connect-gmail.js` SCOPES → `['gmail.send', 'gmail.readonly', 'userinfo.email']`
  - Update the comment block explaining why we're back to two send + read scopes
  - Boola's `send-email` IPC path is unchanged (already uses Gmail API send, no SMTP)
  - In main.js, add new IPC handlers `gmail-search` and `gmail-read` (they already exist as stubs from chat.html — wire them through to `gmail.users.messages.list` and `gmail.users.messages.get` via the existing oauth2Client)

  **User-side steps Boola must guide:**
  - In Settings → Integrations, add a "Email triage" status row. If readonly scope missing, show: "📥 Inbox triage offline. [Re-authorize Gmail]"
  - Clicking that button: opens a modal explaining the steps (add gmail.readonly to consent screen, revoke at myaccount.google.com/permissions, re-run connect-gmail.js)
  - After re-auth, status row shows "📥 Inbox triage active. Reading messages from alenatipton111@gmail.com"

  **Acceptance:**
  - Run connect-gmail.js with new scope list → tokens include readonly
  - `gmail-search` IPC returns real messages from authorized inbox
  - `gmail-read` returns full body of a specific message
  - Send still works (no regression)

- [ ] [claude→codex] **T58: Email triage feature.**
  When the rep asks Boola about their inbox in chat, Boola can actually search + read messages (now that T57 unlocked it).

  **Re-enable existing chat.html tools:**
  - The `search_gmail` and `read_email` tool definitions at chat.html lines ~682-700 already exist
  - They've been silently returning errors since Path A1
  - With T57 done, they Just Work — no chat.html code change needed beyond removing the "// dormant" comments if added
  - Verify `callClaude` includes them when `disableGmailTools: false` (default for chat tab)

  **New triage panel** in the chat tab:
  - Add "📥 Triage Inbox" quick-action button at top of chat
  - Click → calls Claude with prompt: "Use search_gmail to find emails from the last 7 days. Categorize: needs reply / FYI / spam-ish. List the top 5 that need a reply with sender, subject, and a 1-sentence summary."
  - Renders the result as a card list with [Open in Gmail] / [Draft reply with Boola] buttons per item
  - "Draft reply with Boola" pre-fills the Email pane with the original message context and `aftercall` email type as default

  **Acceptance:**
  - Click 📥 Triage Inbox → returns a real list of recent unreplied messages within ~10 seconds
  - Each item has a working [Open in Gmail] link
  - "Draft reply with Boola" opens email pane with context pre-filled

- [ ] [claude→codex] **T59: Call List feature — dedicated tab + add-to-list button + status tracking.**
  Reps build a daily call list from generated leads (or any leads), work through it, mark called/not-reached/connected, and track outcomes.

  **Schema** (`~/Library/Application Support/boola/call-lists.json` — userData per scalability rule):
  ```js
  {
    callLists: [
      {
        id: 'cl-{timestamp}',
        name: 'Mon 6/9 outreach',
        createdAt: '...',
        items: [
          {
            leadId: '...',
            name: '...', company: '...', phone: '...', address: '...',
            vertical: '...', confidence: 92,
            status: 'pending' | 'called' | 'no_answer' | 'connected' | 'not_interested' | 'callback_scheduled',
            calledAt: '',
            outcomeNotes: '',
            callbackDate: ''
          }
        ]
      }
    ]
  }
  ```

  **New "Call List" tab** in chat strip (between Tasks and Documents):
  - List of all call lists (most recent at top)
  - Each list shows: name, item count, called/pending breakdown, [Open] [Export CSV] [Delete]
  - Click [Open] → expands to show items
  - Each item row: name + company + phone + vertical chip + status pill + [📞 Call] [✏️ Status] [🗑️ Remove]
  - [📞 Call] = `<a href="tel:+15551234567">` — opens default phone app (FaceTime/system phone)
  - [✏️ Status] = inline dropdown to change status; setting `connected` or `not_interested` adds an outcome notes field
  - Status counter at top: "12 called / 18 pending / 3 connected"

  **Add to list — two entry points:**
  1. On each Lead card, add a "📋 Add to call list" button. Click → modal asks "Which call list?" (existing lists + "+ New list"). Adds the lead. Mascot fires `focused` emote per T55.
  2. On the Leads tab toolbar, add a "📋 Make call list from these" bulk button. Click → checkbox mode on every lead → "Add 7 selected to list" button → modal.

  **Acceptance:**
  - Click "Make call list from these" → select 5 leads → save as "Tuesday morning" → list appears in Call List tab
  - Click 📞 Call on an item → system phone app opens with the number pre-dialed
  - Change status to "no answer" → status pill updates, counter updates
  - Persist across relaunch

- [ ] [claude→codex] **T60: Call list CSV export (compatible with Sheets / Excel / Numbers).**
  Per-call-list "Export CSV" button generates a `.csv` file with columns the rep can open in any spreadsheet tool.

  **CSV columns:**
  ```
  Company, Contact Name, Phone, Email, Address, Vertical, Confidence,
  Status, Called At, Outcome Notes, Callback Date, Lead Source, Why Now
  ```

  **Implementation:**
  - New `lib/csv-export.js` module — pure CSV generation, no external deps. Handles escaping (quotes, commas, newlines) per RFC 4180.
  - IPC `call-list-export` in main.js → builds CSV string, writes to `~/Desktop/Boola - {ListName} - {YYYY-MM-DD}.csv` (matches T54 Desktop save pattern)
  - On success → toast: "📄 Saved to Desktop: Boola - Tuesday morning - 2026-06-09.csv [Open]"

  **Why CSV not Sheets API:**
  - CSV opens in Sheets, Excel, Numbers, Airtable, anything — no integration overhead
  - Sheets API would require new OAuth scope + verification path
  - User drags CSV into Sheets → done. ~5 seconds vs hours of integration code.

  **Acceptance:**
  - Click Export CSV on a list → file lands at ~/Desktop with correct name format
  - Open the file in Google Sheets → all columns parse correctly, no quote-escaping bugs even with commas/newlines in notes fields
  - Open in Numbers → same

- [ ] [claude→codex] **T62: Make all popup windows freely draggable + closable.**
  Current annoyance: when the reminder popup or daily-leads popup or after-call followup popup fires, the user can't reposition it. They're frameless windows with no drag affordance, so if they fire over something you're working on, you have to dismiss to keep working.

  **Affected windows:**
  - `reminder.html` (todo reminder popup, fires at 10/13/15)
  - `prospect.html` (daily leads popup, fires at 9:30)
  - `followup.html` (after-call follow-up popup)
  - `setup.html` (less of a popup but same issue — gets stuck when opened)

  **Fix per window:**

  1. **Drag handle** — add a top bar in each window's HTML with CSS `-webkit-app-region: drag` so the user can grab anywhere on that bar and move the window. Style it subtly: 28-32px tall, slightly tinted, no border. Within the drag bar, any interactive elements (close button, etc.) get `-webkit-app-region: no-drag` so they still receive clicks.

  2. **Always-visible close button** — top-right of every popup. White `×` icon on a circular subtle background. Hover state. Click → window.close() (or hide for the mascot's case).

  3. **Movability flag** — in `main.js`, ensure each BrowserWindow is created with `movable: true` (likely default but verify). If frameless, the drag handle (above) is what unlocks the move.

  4. **Position persistence (nice-to-have)** — when the user moves a popup, remember its position so the next fire opens at the same spot. Store under `app.getPath('userData')/popup-positions.json` keyed by window type. If the saved position is off-screen (multi-monitor case), fall back to default position.

  5. **Don't break existing animations** — the prospect window's swim-up animation should still work. The drag handle becomes interactive only AFTER the swim animation completes.

  **Acceptance:**
  - Fire each popup → drag from the top bar → window moves smoothly across screen
  - Each popup has a visible × close button that works
  - Move the prospect popup to a new location → close → next 9:30 fire opens at the new location
  - Multi-monitor: drag popup to second display, close, reopen → reopens on second display
  - Swim-up animation on the leads popup still plays correctly on first fire

- [ ] [claude→codex] **T61: Full QA pass for Update 7.**
  Same shape as T42/T53. Cold install, exercise every new feature end-to-end, document findings in codex-notes.md.

  Key scenarios:
  1. Send 5 emails → ~2-3 bubbles appear (50% rate), each from `sent_email` category, no repeats in a row
  2. Mark 5 todos done in a row → 5th one triggers `streak_5_todos` `cool` emote
  3. Sit idle for 30 min → no bubbles, no random animations
  4. Trigger 10/13/15 reminder → canonical emote appears, not legacy mascot
  5. Run T57 OAuth re-auth → readonly scope present, search_gmail returns real results
  6. Click 📥 Triage Inbox → real inbox summary appears
  7. Build a call list, mark statuses, export CSV → file opens cleanly in Sheets
  8. Click 📞 Call on a list item → system phone dialer opens with number

  Sync to ~/BOOLA/ (no-op), push ai-notes commit.

- [ ] [claude→codex] **T53: Full QA pass for Update 6.**
  After T50-T52 land, run the same kind of end-to-end smoke test as T42:

  1. **Cold install** — delete profile.json, relaunch. Walk through setup wizard. Customize layout in edit mode. Restart → all customization persists.
  2. **Custom type create + send (allowVariance:true)** — write a "Quick Pic Quote" type using Alena's gold-standard copy. Send against a junk-removal lead → email matches expected tone, length ≤5 sentences.
  3. **Custom type create + send (allowVariance:false)** — same content but allowVariance off. Send against same lead → output is verbatim template with placeholders filled, no AI rewriting.
  4. **Custom type edit + delete** — modify label, save. Delete custom type, confirm gone from dropdown + profile.json.
  5. **Tab reorder** — move Tasks to position 1. Restart. Tasks is still position 1.
  6. **Tab hide + restore** — hide Calls tab. Confirm in "⋯" overflow menu. Restore. Order preserved.
  7. **Action button hide** — hide LinkedIn button on lead cards. Restart. Still hidden across all leads. Restore.
  8. **Cross-rep simulation** — switch sales type from junk to "medical_device_sales" → all customization preserved, but EMAIL_TYPE_GUIDE adaptations work for the new vertical (in-area says "rep at hospital across town" instead of "trucks in the area").
  9. **Bug hunt** — anything that breaks should land in codex-notes.md.

  Mark `[x]` only after each scenario individually passes. No "code complete" — full validation.

  Sync to `~/BOOLA/` (no-op now since canonical = working copy). Push ai-notes git commit.

## Completed

_(none yet)_

## Codex Handoff To Claude

- [ ] [codex→claude] **Review latest Boola mascot/emote work in Claude Code.**
  Open the live app folder in Claude Code:
  `cd /Users/alenatipton/BOOLA && claude`

  Context:
  - Codex removed the floating always-on-top mascot window from normal startup so Boola no longer floats around the desktop.
  - Codex launched the updated app from `~/BOOLA/`.
  - Codex replaced the old hand-drawn/SVG mascot rendering with approved image-backed assets from:
    - `~/Downloads/Boola Mas.png`
    - `~/Downloads/Boola Expressions.png`
  - New assets live in:
    - `~/BOOLA/mascot-emotes/`
    - synced copy: `~/BOOLA/mascot-emotes/`
  - Implemented emote files:
    `base.png`, `open-mouth.png`, `throwing-money.png`, `swimming.png`, `thinking.png`, `confused.png`, `cooking-money.png`, `email-reject.png`, `angry.png`, `waving.png`, `excited.png`, `sleeping.png`
  - Updated `main.js` so `set-expression`, thinking, and celebrate events update the visible chat header avatar now that the floating mascot is off.
  - Updated `chat.html`, `mascot.html`, and `prospect.html` to use the new `mascot-emotes/` assets.
  - Updated dock icon source to `mascot-emotes/base.png`.
  - Test output is in `codex-notes.md` under `2026-05-07T16:11:35Z - Approved Boola mascot emote system`.

  Claude review asks:
  - Verify the new mascot/emotes visually match the approved references closely enough.
  - Confirm the header avatar expression swaps are acceptable UX now that the floating mascot is disabled.
  - Confirm whether T22 and T23 should be marked complete, partially complete, or rewritten around the new asset-backed mascot system.
  - If Claude wants to run the app manually, use:
    `cd /Users/alenatipton/BOOLA && npx electron .`
