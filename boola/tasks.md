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

- [ ] [claude→codex] **T39: Email header + body spacing fidelity end-to-end.**
  Audit every code path that takes a Claude-generated email and exposes it to the user:
  1. `getEmailExpertSystem` already specifies "Subject line first. Then blank line. Then body." Verify the regex/split that parses subject from body in `generateEmail` is robust (handle missing blank line, multi-line subjects, leading/trailing whitespace).
  2. **Copy button** (`copyResult`) — currently copies rendered HTML innerText. Must preserve actual `\n` so pasting into Gmail/Outlook keeps paragraphs intact. Use `navigator.clipboard.writeText` on the raw text, not the rendered DOM text.
  3. **Send button** (`sendFromResult` → `send-email` IPC → nodemailer) — confirm `text:` field gets raw body with `\n` and `html:` field uses `<p>` paragraphs (not just `<br>`) so the email client renders proper paragraph spacing.
  4. Generated subject line is auto-routed to the email's actual Subject header — not buried in the body. Currently the email-pane result shows both subject+body in one text block. Add a small read-only subject field above the body when result renders, so it's visually clear which line becomes the Subject header.
  5. Acceptance: draft an after-call email, click Copy, paste into a fresh Gmail compose → paragraphs preserved, subject lands in subject row. Send via boola → recipient receives well-formatted email with paragraphs (verify in own inbox).

- [ ] [claude→codex] **T40: Document templates with screenshot-based fill.**
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

- [ ] [claude→codex] **T42: Full QA pass — user functionality + bug hunt across Update 5.**
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

- [ ] [claude→codex] **T41: Autonomous call mode — mic listening + email extraction + auto follow-up.**

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

- [ ] [claude→codex] **T38: Todo completion = celebrate + hide from active view.**
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

- [ ] [claude→codex] **T37: Auto-task on email send.**
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
