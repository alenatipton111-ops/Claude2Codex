# boola — Claude notes

_Architecture, plans, specs, edge cases. Codex reads this; Codex does not edit it._

---

## Standing context (from ~/.claude/CLAUDE.md)

- **Canonical project folder:** `/Users/alenatipton/Projects/boola/`
- **Active working copy (8 AM auto-launch):** `/Users/alenatipton/Desktop/boola-new/`
- Edits should land in `~/Desktop/boola-new/` first so the running app picks them up, then sync to `~/Projects/boola/`.
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

### Phase 2 (next session — not yet for Codex)

- Vertical-research algorithm: derive verticals from `CUSTOMER_PROFILE.service` via one Claude API call/day → search "[vertical] [region]" → extract company names → validate via same IPC.
- Replace the Phase-1 static fallback with vertical research output.
- Spec to be written when Phase 1 is verified working.
