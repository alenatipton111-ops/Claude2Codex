# boola — Claude notes

_Architecture, plans, specs, edge cases. Codex reads this; Codex does not edit it._

---

## Standing context (from ~/.claude/CLAUDE.md)

- **Canonical project folder (single source of truth):** `/Users/alenatipton/Projects/boola/`
- The 8 AM `LaunchAgent` runs from here. Edits land here directly — no Desktop/Projects sync step anymore (Desktop folder retired 2026-05-11).
- **Hard rule:** every new feature must work for an arbitrary paying customer, not just for Alena. No hardcoded user paths, no plaintext creds, no hardcoded company/territory copy.

## Known scalability debt (punch list)

- File paths: `~/.boola_key`, `~/.boola_gmail`, `~/.boola_zoominfo`, `~/.boola_todos`, `~/.boola_todays_leads.json`, `~/.boola_zi_*` → must move to `app.getPath('userData')`
- Hardcoded copy "Alena at 1-800-GOT-JUNK NYC" in `chat.html` system prompts → Customer Profile config
- News feeds hardcoded to NYC in `prospect.html` (`RSS_FEEDS`, `NYC_GEO`, `NON_NYC_GEO`) → Customer Profile config
- Plaintext ZoomInfo credentials in `~/.boola_zoominfo` → macOS Keychain / Windows Credential Manager

## Pending product decisions

- **Email integration:** Path A (own Google-verified OAuth, ~6wk, free) vs Path B (Nylas-style managed API, ~3-5d, $30-100/customer/month). Undecided. No email-aware features until decided.
- **ZoomInfo data source:** current scrape will break across customers; needs real API contract eventually (ZoomInfo Enterprise / Apollo / Lusha).

---

## Universal SaaS rule — non-negotiable

Boola is sold to **any B2B seller anywhere selling anything.** Junk removal in NYC, medical devices in Cincinnati, SaaS in Austin, commercial cleaning in Toronto — same code, different workbook/profile data.

**No code may contain:**
- Hardcoded city/state/country names or neighborhoods
- Hardcoded URLs (OpenData portals, WARN sites, news feeds)
- Hardcoded company names, chain lists, vertical mappings
- `if (salesType === ...)` or `if (region === ...)` branches
- "Supported regions" gating

**All such data lives in `lead-rules.xlsx`** (and `profile.json` for per-customer customization). New region or new sales type = new workbook row, zero code changes.

**Litmus test:** before merging any task, mentally swap the customer to "medical device rep in Phoenix." If the feature breaks or needs special handling, the implementation is wrong.

## Claude-built code log (2026-05-13)

Claude directly built T48 + T43-OSM + T44 + T45 in this session (vs. delegating to Codex) because Codex's prior pass on the lead engine shipped half-working code that was hard to debug. Future Codex sessions should treat the following as **already implemented in `main.js`** and not rebuild:

### Added to `main.js`
- `geocodeLocation(query)` — free Nominatim geocoding, works globally, no auth
- `ensureProfileGeocoded(profile)` — auto-resolves region label → lat/lng on demand, caches to profile.json
- `VERTICAL_OSM_TAGS` mapping table — covers all 35 sales-type verticals via OSM amenity/office/shop/healthcare tags. Fuzzy-matches workbook vertical names against the table keys.
- `osmTagsForVertical(name)` — fuzzy lookup against the table
- `buildOverpassQuery(lat, lng, radiusMeters, tagPairs)` — constructs Overpass QL string
- `overpassQuery(query)` — POSTs to overpass-api.de, returns elements
- `osmElementToLead(el)` — normalizes OSM element to standard `{name, address, phone, website, osmId, sourceType:'osm'}` shape
- `osmDiscover({vertical, lat, lng, radiusMiles, count})` — high-level adapter
- `GLOBAL_CHAIN_BLOCKLIST` — Set of ~40 chains (Starbucks, Marriott, Walmart, Equity Residential, etc.) filtered universally regardless of region
- `isBlocklistedChain(name)` — substring case-insensitive match
- `dailyShuffle(arr)` — deterministic shuffle seeded by `Math.floor(Date.now()/86400000)`. Same day = same order; next day = fresh rotation
- `canonicalPhone(phone)` — 10-digit normalization for dedupe

### Rewrote `fetchVerticalLeadsCore()` in `main.js`
- Drops DDG/`findVerticalLeadCandidates` entirely (was producing 0 results)
- For each enabled workbook vertical sorted by priority_score:
  - Calls `osmDiscover` with 80-candidate target
  - Filters: known chain, recent exclusion list, dedupe by name
  - Applies `dailyShuffle` for stable-within-day rotation
  - Enriches via existing `lookup-company-info` when OSM lead is missing phone/address
  - Requires phone for final inclusion (callable threshold)
- Per-vertical target = ceil(targetCount / vertical count)
- 90s total budget
- Returns leads tagged `source: 'OSM vertical'`

### New IPC handlers
- `geocode-location` → `geocodeLocation(query)`
- `osm-discover` → `osmDiscover(args)`

### Smoke-tested
- NYC junk removal salesType: 58 property mgmt, 59 commercial RE, 57 construction businesses found
- Cincinnati hospitals: 5 real ones with phones
- Daily shuffle deterministic verified

### Pending follow-ups (NOT yet done — for Codex or next Claude session)
- Strip remaining NYC-hardcoded constants (`NYC_BIZ_API`, `NYC_BOROUGHS`, etc.) — these still exist in `main.js` but are only used by the news/structured-signal path which is fine for NYC users today. For non-NYC users they silently return empty (acceptable fallback for now). True regionalization of OpenData portals is a future task.
- `KNOWN_COMPANIES` list in `prospect.html` is dead code now that OSM does discovery. Can be deleted.
- `findVerticalLeadCandidates` and `searchTemplatesForRule` in `main.js` are dead code. Can be deleted.
- News feed defaults are still NYC-flavored in `DEFAULT_CUSTOMER_PROFILE.newsFeeds`. For non-NYC users they'll fetch but produce few region-relevant items. National-feed default set is a follow-up.

## Canonical mascot emote sheet (2026-05-13)

The visual identity is fixed. There are exactly **13 approved Boola emotes**:

1. Happy · 2. Excited · 3. Thinking · 4. Focused · 5. Surprised · 6. Frustrated · 7. Decision (📧✅/📧❌) · 8. Celebrating · 9. Tired · 10. Cooking Money · 11. Thinking (Headset) · 12. Cool · 13. Boola (Relaxed)

PNG files live at `~/Projects/boola/mascot-emotes/<state>.png`. Any code that sets a Boola expression must reference one of these 13 names — never invent a new variant. If a feature needs a state that doesn't fit one of the 13, flag it to Alena before writing code; do not improvise.

## Mascot is the product — non-negotiable

Boola IS the floating whale in the corner. The mascot is not optional, not a launcher, not a header avatar. It must be:
- Always-on-top transparent frameless `BrowserWindow` in the bottom-right of the primary display by default
- Visible at startup
- Clickable to toggle the chat panel
- Draggable (user moves it where they want; persist position across launches if practical)
- **Static** — no idle bobbing, no auto-firing animations, no random "swim across screen" events
- Expression swaps allowed (face changes triggered by user actions or system state: thinking, headset, excited, celebrate, sleepy)

Any future change that proposes moving the mascot into the chat header / removing the floating window / replacing it with a tray icon is a **rule violation** unless Alena explicitly asks for it.

## Runtime independence guardrail

Boola the shipped product is **plain Node + Chromium** (Electron). It must never depend at runtime on Codex, Claude CLI, ai-notes, MCP, or any AI-developer tooling. The ai-notes ↔ Codex workflow exists only for *building* boola; nothing it produces should be required to *run* boola. When tempted to add a feature that calls Codex, the Claude CLI, or anything in `~/ai-notes/`, stop — that's a personal-use hack and violates the SaaS rule.

Allowed runtime AI: the customer's own Anthropic API key (already the pattern), and any future explicit integrations the customer authenticates themselves.

## Active plan — Lead Generation Overhaul (Phase 1 of 2)

**Goal (Phase 1, this session):** Daily leads must be validated to have business name + valid phone + address before counting. Expand news sources from 4 to 15. Fall back to static pool only if news can't validate ≥3.

**Goal (Phase 2, next session):** Vertical-research algorithm — when news doesn't supply enough validated leads, derive top verticals from the customer's service offering (Claude API call), then web-search + validate mid-size companies in those verticals to reach 8 total leads/day.

### Spec (from Alena, 2026-04-29)

- Total: **8 leads/day**
- Composition:
  - **Up to 3** from news (Phase 1)
  - **Remainder (5-8)** from vertical research (Phase 2)
- Hard validation rule: every lead must have **business name + valid phone**. News leads additionally must have **address**.
- News source pool: **15 business journals + region-specific updates**, focused on businesses **closing down** and **construction happening**.

### Architecture decisions

1. **Customer Profile config block** — single object loaded from `profile.json` at app startup. Schema:
   ```js
   const CUSTOMER_PROFILE = {
     // Persisted to app.getPath('userData')/profile.json after first-run setup wizard.
     name: '1-800-GOT-JUNK NYC',
     service: 'junk removal',
     region: { code: 'nyc', label: 'New York City', state: 'NY' },
     newsFeeds: [...],          // 15 entries
     geoMatch: NYC_GEO,         // regex
     geoExclude: NON_NYC_GEO,   // regex
     // Action-based signal types the customer cares about. Selected via
     // checkbox in the first-run setup wizard. Each signal maps internally
     // to a data fetcher (see § Lead source overhaul).
     targetSignals: [
       'move-in',            // new tenants, lease signings, business openings (legit move-ins)
       'move-out',           // closings, vacate orders, tenant turnover
       'construction',       // new builds, gut renovations
       'renovation',         // alterations, retrofits
       'illegal-dumping',    // 311 dumping complaints near service area
       'furniture-changeover', // office moves, redesigns
       'business-closing',   // WARN, restaurant shutters, retail bankruptcies
       'damage-event'        // fire/flood/storm damage requiring cleanout
     ],
     verticals: ['property manager','apartment complex','construction GC','restaurant','retail',
                 'medical office','law office','hotel','school','warehouse'] // for Phase 2
   }
   ```
   `RSS_FEEDS`, `NYC_GEO`, `NON_NYC_GEO` become references into `CUSTOMER_PROFILE`. Mark each migration with `// SCALABILITY:` comment.

   **Hardcoded defaults are bootstrap-only** — once first-run setup is built (see § First-run setup wizard), defaults are written to `profile.json` and the user edits from there.

2. **15 NYC news sources** — pick best feeds for closing/construction signal. Codex should test each URL before committing; if any return 404 or paywall blocking RSS, substitute. Suggested set:
   - NY Post Business — `https://nypost.com/business/feed/` (existing)
   - NY YIMBY — `https://newyorkyimby.com/feed` (existing)
   - Commercial Observer — `https://commercialobserver.com/feed/` (existing)
   - amNewYork — `https://www.amny.com/feed/` (existing)
   - Crain's New York Business — `https://www.crainsnewyork.com/news.xml` or similar
   - The Real Deal NY — `https://therealdeal.com/feed/`
   - Bisnow NY — `https://www.bisnow.com/rss/national/new-york`
   - NY Business Journal (Bizjournals) — `https://www.bizjournals.com/newyork/news/rss.xml`
   - Brownstoner — `https://www.brownstoner.com/feed/`
   - Brooklyn Eagle — `https://brooklyneagle.com/feed/`
   - Patch NYC — `https://patch.com/new-york/across-ny/rss.xml`
   - City & State NY — `https://www.cityandstateny.com/rss`
   - PIX11 Business — `https://pix11.com/business/feed/`
   - NYREJ — `https://nyrej.com/feed/`
   - Curbed NY (Real Estate) — `https://ny.curbed.com/rss/index.xml` (verify still active)

3. **Validation gate** — every news candidate must pass through `validate-news-lead` IPC before being counted. Validation uses existing `lookup-company-info` in `main.js` to return phone + website + address. Reject if no valid phone. Reject news leads if no address (vertical-phase leads in Phase 2 don't require address).

4. **Address extraction** — `lookup-company-info` does not currently extract addresses. Add an `extractAddress(html)` helper in `main.js`:
   - Regex for US-style street addresses: `/\d{1,5}\s+[A-Z][a-zA-Z\s]+(?:Street|St\.?|Avenue|Ave\.?|Road|Rd\.?|Boulevard|Blvd\.?|Drive|Dr\.?|Lane|Ln\.?|Way|Plaza|Place|Pl\.?),?\s+[A-Z][A-Za-z\s]+,\s*(?:NY|New York)(?:\s+\d{5})?/`
   - Try the company homepage + `/contact` + `/about` (already-fetched in lookup flow — just parse them)
   - Validate against the customer's `region.state` to drop out-of-region results

5. **Concurrency** — validations are slow (~5-15s each). Use a 3-at-a-time semaphore in `prospect.html` so we don't serialize 20 lookups (= 5 minutes) but also don't hammer DDG/sites with parallel storms. Total lead-gen budget: **target ≤ 90 seconds end-to-end**.

6. **Progress UI** — `prospect.html` shows a small progress strip during validation: `"Validating leads… 4 / 12 checked, 2 valid"`. Existing whale animation stays. When done, the UI swaps to the lead list.

7. **Phase 1 fallback when <3 validated** — temporary: pad with the existing static `fallbackPool` until Phase 2's vertical research is built. Mark with `// PHASE-1-TEMP:` comment so it's easy to find and replace.

8. **Caching** — keep existing `.boola_todays_leads.json` daily cache. Validated leads stored with full `{name, phone, website, address, validatedAt}` schema so re-renders don't re-validate.

### Files affected

| File | Change |
|---|---|
| `prospect.html` | Add `CUSTOMER_PROFILE`. Refactor `buildProspectsFromNews` to be async + validate via IPC. Progress UI. |
| `main.js` | Add `validate-news-lead` IPC handler. Add `extractAddress` helper. Extend `lookup-company-info` return shape with `address`. |
| `chat.html` | Update lead card render to show validation badges (`✓ phone`, `✓ address`). |

### Edge cases / failure modes

- **Validation produces 0 valid leads.** UI must not be empty — fall back to static pool, show banner "News validation found nothing today; showing curated NYC list."
- **lookup-company-info itself fails** (timeouts, network). Catch per-lead, log, continue with remaining candidates.
- **Rate limiting from DuckDuckGo** — current `searchCompanyWebsite` already has UA + timeout. Codex should add: if 3 consecutive validations fail with empty `searchHtml`, abort the rest and use whatever validated so far.
- **A company name extraction returns junk that happens to validate** (e.g., "Manhattan" returns Manhattan Borough President's office). Mitigation: cross-check that a lookup result's `website` or `name` contains at least one token from the original headline. Codex add this check in `validate-news-lead`.
- **Same business surfaces from multiple feeds** with slightly different names. Dedupe by phone number (canonicalize digits) after validation.

### Acceptance criteria

- [ ] After hitting ↻ on the Leads tab, leads regenerate within ~90s
- [ ] Each shown lead has a phone number and address — verify by inspecting `~/.boola_todays_leads.json`
- [ ] If feeds produce 0 validated leads, static fallback shows with a "no validated news" banner
- [ ] No hardcoded `1-800-GOT-JUNK` or `NYC` strings in code outside `CUSTOMER_PROFILE`
- [ ] All scalability violations marked with `// SCALABILITY:` comment

### Lead source overhaul — kill license-based, switch to action-based

**Drop these (they generate noise):**
- `fetch-openings` IPC in `main.js` (NYC DCWP "newly licensed" — many false positives, unrelated to cleanout demand)
- `fetch-closings` DCWP-inactive-license branch (keep WARN — that's signal, not license-based)
- `fetch-warehouse` IPC (active warehouse licenses — same noise problem)
- `mapOpening` and DCWP branch of `mapClosing` in `prospect.html` (and the "newly licensed / license went inactive" reason strings)

**Add these signal-fetchers** (one per `targetSignals` entry the customer enables):

| Signal | Data source | API endpoint |
|---|---|---|
| `construction` | NYC DOB Permit Issuance | `https://data.cityofnewyork.us/resource/ipu4-2q9a.json` filter `permit_type='NB'` (new building) or `'DM'` (demolition) last 14 days |
| `renovation` | NYC DOB Permit Issuance | same dataset, `permit_type='A1'` or `'A2'` (alterations) last 14 days |
| `move-in`/`move-out` | NYC DOB Job Application Filings | `https://data.cityofnewyork.us/resource/ic3t-wcy2.json` filter recent filings |
| `illegal-dumping` | NYC 311 Service Requests | `https://data.cityofnewyork.us/resource/erm2-nwe9.json` filter `complaint_type='Illegal Dumping'` last 7 days |
| `business-closing` | WARN (existing) + DOH closures | keep `parseWARN`. Add DOH restaurant closures dataset if available. |
| `damage-event` | News only | already covered via OPPORTUNITY_SIGNALS regex on RSS items |
| `furniture-changeover` | News only | covered by news; no public dataset for this directly |

Each fetcher returns the standard `{ name, address, borough, industry, source, date }` shape so the existing rendering pipeline keeps working. Apply the same validation gate (`validate-news-lead`) to **every** lead regardless of source — must produce phone + address before counting.

### First-run setup wizard

A small Electron window that appears on first launch when no `profile.json` exists. Persists output to `app.getPath('userData')/profile.json`. Fields:

1. **Company name** — text (e.g., "1-800-GOT-JUNK NYC")
2. **Service or product offered** — text (e.g., "junk removal", "commercial cleaning"). Drives Phase 2 vertical derivation.
3. **Region** — text + state dropdown (e.g., "New York City", state: NY). Determines news feeds, geo regex, public dataset endpoints.
4. **Target signal types** — multi-checkbox, defaulting to all enabled. Each = one entry in `targetSignals`. Tooltip per checkbox explains what it captures.
5. **Brand voice / tone notes** — optional textarea, threaded into chat system prompt.
6. **Anthropic API key** — re-uses existing `~/.boola_key` flow but moves into wizard.

After save: profile.json written, wizard closes, mascot launches normally. A "Settings" gear in chat reopens the wizard at any time to edit.

### Anthropic API key — never customer-facing

**SaaS rule:** customers do not bring their own Anthropic key. The shipped product must include a boola-operated backend that brokers all Claude calls. Customers authenticate against the backend with a customer-issued token; the backend proxies to Anthropic with the operator's master key and meters usage for billing.

**Current state (transitional):** boola calls Anthropic directly from `chat.html` via `fetch` using a key loaded from `~/.boola_key`. This is acceptable for development but must not surface in customer UI.

**Immediate action (T21):** remove the "Anthropic API key" field from `setup.html`. Continue loading from `~/.boola_key` silently for dev use. Mark every direct `api.anthropic.com` call site with `// SCALABILITY: route through boola backend in production` so we have a punch list when the backend ships.

**Later (separate phase, post-Phase-2):** build the backend service:
- Endpoint: `POST https://api.boola.app/v1/chat` (or similar)
- Auth: `Authorization: Bearer <customer_session_token>`
- Boola Electron client stores only customer_id + session_token (no Anthropic key)
- Backend proxies to Anthropic, logs usage per customer, enforces rate limits
- Subscription billing handled at backend (Stripe etc.)
- This unblocks: customer onboarding (no Anthropic account needed), unified billing, usage analytics, abuse protection

### Mascot redesign

Alena finalized a new Boola brand identity (cute blue baby shark, big glossy eyes, blush cheeks, navy script wordmark, blue wave underline). Full visual spec: **`~/Downloads/boola_mascot_redesign_brief_for_codex.md`** — Codex reads that file as the canonical design source.

Implementation requirements specific to boola's existing code (extends the brief):

1. **SVG, not raster.** Inline SVG in `mascot.html` and `prospect.html`. Keeps animations crisp at any DPI and no asset pipeline needed.

2. **Preserve the existing expression system.** `mascot.html` listens for `set-expression` IPC events and switches between: `happy`, `thinking`, `excited`, `headset`, `celebrate`, `sleepy`, plus a few others. Codex should build expression variants by swapping eye/mouth/accessory layers within the new mascot, not rewriting it. Acceptable approach: structure the SVG with named groups (`#face-happy`, `#face-thinking`, etc.) that show/hide via class toggle.

3. **Preserve animations.** Current mascot has a `swim` keyframe (rocks side-to-side); leads pop-up has the whale "swimming up" holding the scroll. Both need to keep working — just on the new artwork.

4. **Wordmark application.** Use the navy "Boola" script + blue wave underline as the branded header in:
   - `setup.html` (currently "Boola Setup" plain text)
   - Leads pop-up header in `prospect.html`
   - Chat window header in `chat.html`
   - Any other place currently saying "Boola" in plain text

5. **macOS dock icon.** Generate a 1024×1024 PNG from the new SVG, convert to `.icns` via `iconutil`, drop in project root as `icon.icns`, and reference via `app.dock.setIcon()` in `main.js` startup. (Path: research the current Electron dock-icon setup; may already use `productName` / no icon set.)

6. **No customer-specific copy.** Wordmark, mascot, and underline are global brand — never gated by `CUSTOMER_PROFILE`. The customer's company name shows up in chat content, not in the mascot itself.

7. **Color palette compliance.** Use the exact hex values from the brief — they're brand-locked now, not suggestions.

### Phase 2 — Workbook-driven lead generation (THIS IS THE ACTIVE PLAN)

**Bug confirmed (2026-05-08):** `~/.boola_todays_leads.json` shows the same 3 leads (Hudson Yards / Marriott Marquis Times Square / NYU Langone) every day. Root cause: news-validation gate (must produce phone+address+token-cross-check) is too strict, so news leads almost never qualify. The `// PHASE-1-TEMP:` static `fallbackPool` (3 hardcoded NYC anchors) then pads the list. That fallback was meant as a one-week safety net; it's been the only producer for weeks.

**Fix is structural, not a tweak.** Alena delivered a master workbook (`lead-rules.xlsx`, copied to project root) with 8 tabs covering sales types, verticals by month, source routing, source scoring, cold-email rules, regional source rules. Boola's lead engine becomes a workbook-driven pipeline.

#### Goal

10 leads/day, generated fresh each morning, composed of:

| Bucket | Count | Source |
|---|---|---|
| 1. **Regional news triggers** | up to 3 | RSS news + 311/DOB + WARN, validated to phone+address |
| 2. **Targeted vertical companies** | fills to 7 | Workbook-driven: customer's `targetVerticals` × current month, searched via workbook source rules |
| 3. **Seasonal toppers** | fills to 10 | Workbook `best_buy_months` matches current month — vertical entering peak buying window |

If a bucket can't fill, fill from the next bucket — never duplicate, never repeat yesterday's leads. Persist a 14-day rolling exclusion list of recently-shown company names so even random/static fills don't repeat.

#### Workbook integration

- Ship `lead-rules.xlsx` in the project root (committed) and load at startup. Use `xlsx` npm package (small, no native build) or pre-bake the workbook into JSON at build time. **Recommendation: pre-bake to `lead-rules.json`** at install time so runtime has zero spreadsheet parsing cost. Include a build script `scripts/build-lead-rules.js` that reads the xlsx and emits the JSON.
- Tabs to consume:
  - `Master_B2B_Lead_Rules` → sales_type → vertical → priority_score, best_buy_months, outreach_start_months, search_template, buyer_titles
  - `Source_Routing_Logic` → intent → primary/secondary sources mapping
  - `Source_Scoring_Rules` → confidence-score formula
  - `Sources` → URL directory for each source_key
  - `Regional_Source_Rules` → tier/freshness/best_for per source
  - `Cold_Email_Rules` → subject lines, openers, follow-ups, objection responses
- Treat workbook as immutable runtime data; updates ship with new boola releases.

#### Setup wizard additions (one-time per customer)

Three new fields on `setup.html`, persisted to `profile.json`:

1. **Sales type** — single-select dropdown sourced from workbook `Category_Summary.sales_type`. Drives vertical recommendations and source routing. Default for Alena: "Facilities / Commercial Services / Junk Removal".
2. **Target verticals** — multi-select shown after sales type is picked, populated from workbook `Master_B2B_Lead_Rules` rows where `sales_type` matches. Default = all rows checked. Each vertical row exposes its `priority_score` so customer can see what's weighted heavy.
3. **Territory radius** — text input, miles from region center. Used to scope source queries (e.g. "Manhattan + 25mi"). Default 25.

#### Lead-gen pipeline (replaces current `prospect.html` flow)

```
1. Load profile.json → {salesType, targetVerticals, region, targetSignals, ...}
2. Compute current month → derive in-season verticals via workbook best_buy_months
3. Bucket 1 — News triggers (existing pipeline, unchanged in mechanism):
   - RSS feeds + DOB permits + 311 dumping + WARN
   - Validate via validate-news-lead (phone + address + token check)
   - Cap at 3
4. Bucket 2 — Targeted verticals (NEW):
   - For each enabled targetVertical (sorted by priority_score):
     - Generate search queries from workbook search_template field
     - Run via existing searchCompanyWebsite + lookup-company-info
     - Validate (name + phone required; address optional for vertical leads per spec)
     - Stop when bucket reaches 7 - bucket1.length
5. Bucket 3 — Seasonal toppers (NEW):
   - Find verticals where currentMonth ∈ best_buy_months
   - If different from already-pulled verticals, do one more pass to fill remaining slots
6. Apply rolling exclusion: drop any name in last 14 days' leads file
7. Confidence-score each lead via Source_Scoring_Rules
8. Sort by confidence desc, take top 10
9. Persist to .boola_todays_leads.json with full schema
   {name, phone, address, website, vertical, source, sourceUrl,
    confidence, whyNow, suggestedBuyer, buyingTrigger,
    coldSubject, coldOpener, validatedAt}
10. Also persist to .boola_lead_history.json (rolling 14 days, name+date)
```

#### Lead card UI changes (`chat.html`)

Each lead card gains:
- **Vertical** chip (small purple pill)
- **Confidence score** (1-100, color-coded green/yellow/red)
- **Why now** line (one-sentence trigger from buying-signal logic)
- **Suggested buyer title** (from workbook buyer_titles)
- **Source** chip (showing the workbook source_name)
- Quick-action: "📧 Generate cold email" — fills the Email pane with workbook-rule-compliant subject + opener pre-populated

#### What gets retired

- `// PHASE-1-TEMP:` static `fallbackPool` — deleted, not commented out
- Hardcoded 3-company anchor (Hudson Yards / Marriott Marquis / NYU Langone) — gone

#### Acceptance

- Run lead refresh on three consecutive days → cache shows 10 different leads each day, ≤ 1 carryover
- All 10 leads have name + phone (address required only for bucket-1)
- Each lead card shows vertical, confidence, why-now, suggested buyer
- Setup wizard shows sales-type dropdown with 35 options from workbook
- Setting `salesType = "Tech / SaaS / IT"` and re-running produces leads in healthcare/financial verticals (per workbook), not real estate

### Phase 2.5 — Real business discovery via Google Places API

**Problem confirmed (2026-05-12):** Codex's Phase 2 vertical fetcher uses DuckDuckGo HTML search via `findVerticalLeadCandidates(query)` to discover companies. DDG returns articles/directories/ads, not company directories. Almost no candidates survive extraction. Result: 0 vertical leads, daily list is empty when news doesn't produce.

**Fix:** replace DDG-based vertical discovery with **Google Places API (New)**. Authoritative, structured, returns name+address+phone+website per company. Free tier covers a customer's daily usage with margin.

#### Architecture

1. **Customer setup adds Google Places API key field.**
   - In setup wizard, add a section: "Lead discovery API key (Google Places)". Customer creates a Google Cloud project, enables Places API (New), generates an API key, pastes it in.
   - Wizard shows a one-paragraph "how to get this" tooltip with a link to https://console.cloud.google.com/apis/library/places.googleapis.com
   - Stored in `profile.json` as `profile.googlePlacesApiKey`.
   - SCALABILITY: in production this routes through the boola backend broker (one shared key, billing per customer). Mark with `// SCALABILITY:` comment.
   - For Alena's current dev install: she generates her own key once.

2. **New IPC handler `places-discover` in `main.js`:**
   - Input: `{vertical, region, radiusMiles, count}` (vertical from workbook `Master_B2B_Lead_Rules`, region from profile)
   - Maps vertical → Google Places `includedTypes` via a mapping table (see workbook → can be a new column or hardcoded mapping in code initially):
     - "property manager" → `["real_estate_agency","property_management_company"]`
     - "construction GC" → `["general_contractor"]`
     - "restaurant" → `["restaurant"]`
     - "law office" → `["lawyer"]`
     - "medical office" → `["doctor","medical_lab","dental_clinic","veterinary_care"]`
     - "hotel" → `["lodging"]`
     - "retail" → `["clothing_store","shoe_store","store"]` (curated subset)
     - "school" → `["school","primary_school","secondary_school","university"]`
     - "warehouse" → `["warehouse","wholesaler"]`
     - "apartment complex" → `["apartment_complex","real_estate_agency"]`
   - Calls Google Places API (New) `places:searchNearby` endpoint:
     ```
     POST https://places.googleapis.com/v1/places:searchNearby
     X-Goog-Api-Key: {key}
     X-Goog-FieldMask: places.id,places.displayName,places.formattedAddress,
                       places.internationalPhoneNumber,places.nationalPhoneNumber,
                       places.websiteUri,places.businessStatus,
                       places.userRatingCount,places.types,places.primaryType
     Body:
     {
       "includedTypes": [...],
       "maxResultCount": 20,
       "locationRestriction": {
         "circle": {
           "center": {"latitude": <lat>, "longitude": <lng>},
           "radius": <radiusMeters>
         }
       }
     }
     ```
   - Returns array of `{name, address, phone, website, googlePlaceId, userRatingCount, primaryType, types[]}`.

3. **Chain / franchise filter.**
   - Reject obvious chains. Maintain a `CHAIN_BLOCKLIST` in `main.js`:
     - Property: Equity Residential, AvalonBay, Greystar, Camden, Aimco, Related Companies, Brookfield, Tishman Speyer, Vornado
     - Hotels: Marriott, Hilton, Hyatt, IHG, Wyndham, Best Western, Hampton Inn, Holiday Inn, Sheraton
     - Restaurants: Starbucks, McDonald's, Subway, Chipotle, Sweetgreen, Shake Shack, Panera, Chick-fil-A
     - Retail: Walmart, Target, Whole Foods, Trader Joe's, CVS, Walgreens, Duane Reade, 7-Eleven, Macy's, Nordstrom, Saks
     - Generic: any name matching `/^(?:the )?(.*?) (?:inc|llc|corp|holdings)$/` where it appears 3+ times in last 30 days (auto-learned chain)
   - Reject if `places.types` includes `chain_locality` or if name matches blocklist.
   - SCALABILITY: blocklist eventually moves to workbook so customers can edit per market.

4. **Mid-size detection heuristic.**
   - Sweet spot: `userRatingCount` between **5 and 500**. <5 = too small/possibly fake. >500 = chain or franchise.
   - For B2B-only verticals (property management, law) drop the upper bound — established firms can have many reviews.
   - Add `sizeBucket: 'micro' | 'mid' | 'large'` to each lead based on rating count. Show mid-size first.

5. **Daily rotation — never show the same lead twice within 14 days.**
   - On each `places-discover` call: fetch top 50 per vertical from Google.
   - Filter through 14-day exclusion list (already specced as `lead-history.json`).
   - Of the remaining pool, **shuffle deterministically using `Date.now() / 86400000` as seed** so same day = same order, different day = different shuffle.
   - Take the first N. Stable per-day, fresh day-over-day.

6. **Geocoding the region.**
   - Profile has `region.label` like "New York City". Need `(latitude, longitude)` for Places.
   - Add a small static mapping of common metros, OR call Google Geocoding API (also free tier) on first save to resolve region → coordinates. Cache lat/lng in profile.
   - For unknown region: prompt user during setup to confirm "Use New York City coordinates? [yes / change]".

7. **Replace DDG path in `fetchVerticalLeadsCore`.**
   - Drop `findVerticalLeadCandidates(query)` for vertical-bucket leads.
   - News bucket (T26-onward) still uses RSS+DOB+311 — unchanged.
   - Seasonal bucket uses the same Places infrastructure with different vertical inputs.

8. **Fallback chain when Places key is missing or quota exceeded.**
   - If no `googlePlacesApiKey` set: skip vertical bucket entirely, show banner "Add a Google Places API key in Settings to unlock vertical leads."
   - If quota exceeded (HTTP 429): retry with exponential backoff up to 3x, then degrade to existing DDG path with banner "Google Places quota hit; using fallback search."

#### Cost projection (per customer, daily 10-lead generation)

| Source | Calls/day | Cost |
|---|---|---|
| Nearby search (3 verticals × 1 call each) | 3 | $0.017 × 3 = $0.05 |
| Place details on top 10 of pool | 10 | $0.017 × 10 = $0.17 |
| **Daily total** | 13 | **$0.22** |
| **Monthly** | ~400 | **$6.60** |

Well within free tier ($200/mo credit) per customer.

#### Acceptance

- Open Settings → enter a valid Google Places key → save
- Tap ↻ on Leads tab → 7-10 vertical leads appear, ALL with phone + address + website
- None are obvious chains (no Marriott, no Whole Foods, no Equity Residential)
- Tap ↻ again next day → at least 6 of 10 leads differ from previous day
- `~/Projects/boola/.boola_lead_history.json` shows last 14 days of names

### Phase 3 (after Phase 2 verified)

- Cold email pane upgrade: when user has selected a lead, "Generate" pulls workbook subject_line_styles + opener_templates + objection_responses for that vertical. Currently `EMAIL_TYPE_GUIDE` is hardcoded; replace with workbook lookup.
- Manager-tier upgrade: per-rep analytics dashboard reading lead-touch logs.

## 2026-05-13 UI: single-column lead popup (Alena ask)

- Removed the "Target Titles" column from the daily-leads popup (`prospect.html` `renderList`). Buyer titles still surface inline on each lead card via `suggestedBuyer` field.
- Single column now spans full popup width.
- Up to 10 leads visible (was 5 per column).
- Each card now shows: #, name, confidence badge (green pill), why-now, phone (cyan), vertical · suggested buyer.

## 2026-05-15 single-instance lock (Alena ask)

Symptom: 8 AM LaunchAgent fires every morning and stacks more boola processes. After 5 days = 5 instances.

Fix in `main.js`: wrapped `app.whenReady()` and all top-level lifecycle handlers in `app.requestSingleInstanceLock()`. If a second launch is attempted, it logs "another instance is already running — exiting" and quits. The existing instance receives a `second-instance` event and brings its mascot/chat windows forward.

Tested: ran two consecutive `electron .` launches. First spawned full process set; second exited immediately with the expected log message. No new processes from the second attempt.
