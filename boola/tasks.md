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

- [x] [claude→codex] **T11: Sync to project folder** — after T10 passes, copy edited files (`prospect.html`, `main.js`, `chat.html`) from `~/Desktop/boola-new/` to `~/Projects/boola/`.

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

- [ ] [claude→codex] **T19: Smoke test the new pipeline.**
  1. Delete `app.getPath('userData')/profile.json` if present
  2. Relaunch boola — confirm setup wizard appears
  3. Save with defaults — confirm profile.json is written and main app launches
  4. Open Leads tab → tap ↻ — confirm leads come from action-based sources (no "newly licensed" reasons appear)
  5. Confirm at least one lead from each enabled signal type appears in `~/.boola_todays_leads.json` (or note in codex-notes if any signal yields zero on the test day)
  6. Open Settings → uncheck `illegal-dumping` → save → tap ↻ — confirm no dumping leads appear
  7. Record findings in `codex-notes.md`

- [x] [claude→codex] **T20: Sync to project folder** — `~/Desktop/boola-new/` → `~/Projects/boola/` for all edited/added files.

## Completed

_(none yet)_
