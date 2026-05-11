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

## Phase 2 — Workbook-driven lead generation
_(Added 2026-05-08 — fixes the repeating-leads bug and rebuilds lead engine on `lead-rules.xlsx`. Spec in `claude-notes.md` § Phase 2.)_

- [ ] [claude→codex] **T26: Build `scripts/build-lead-rules.js`.** Reads `~/Desktop/boola-new/lead-rules.xlsx`, emits `~/Desktop/boola-new/lead-rules.json` with all 8 tabs as keyed objects/arrays. Use the `xlsx` or `node-xlsx` npm package (or existing if already in deps). Run once and commit the resulting JSON. Add an `npm run build:lead-rules` script in `package.json`. Goal: zero xlsx parsing at runtime.

- [ ] [claude→codex] **T27: Add sales-type + target-verticals to setup wizard.**
  In `setup.html`:
  1. Add **Sales type** single-select dropdown after Region. Options sourced from `lead-rules.json` → `Category_Summary` (35 options). Field: `profile.salesType` (string code like `facilities_commercial_services_junk_sales`).
  2. Add **Target verticals** multi-checkbox section, populated dynamically from `lead-rules.json` → `Master_B2B_Lead_Rules` filtered by `sales_type`. All checked by default. Field: `profile.targetVerticals` (array of vertical strings).
  3. Add **Territory radius** number input (miles). Default 25. Field: `profile.region.radiusMiles`.
  4. Bootstrap defaults for Alena's existing profile: `salesType = "facilities_commercial_services_junk_sales"`, all matching verticals checked, radius 25.

- [ ] [claude→codex] **T28: Delete the static `fallbackPool` from `prospect.html`.** Not comment out — fully delete. Remove all references and the `// PHASE-1-TEMP:` block. This will break lead generation until T29-T31 land — that's expected; do these tasks together.

- [ ] [claude→codex] **T29: Implement Bucket 2 — vertical fetcher.**
  In `main.js`: new IPC handler `fetch-vertical-leads({salesType, targetVerticals, region, currentMonth})`. For each enabled vertical (sorted by `priority_score` from workbook):
  1. Build search queries from the workbook's search-template field for that row
  2. Run through existing `searchCompanyWebsite` + `lookup-company-info`
  3. Validate name + phone (address optional)
  4. Tag each result with `{vertical, sourceKey, sourceUrl, priorityScore, suggestedBuyer (from buyer_titles)}`
  5. Stop when target count is reached
  Apply a per-call concurrency cap of 3, total budget 60s, return whatever validated by then.

- [ ] [claude→codex] **T30: Implement Bucket 3 — seasonal toppers.**
  In `main.js`: new IPC handler `fetch-seasonal-leads({salesType, region, currentMonth})`. Find rows in workbook where `currentMonth ∈ best_buy_months`. If different from already-pulled verticals, run vertical fetcher for those. Used to fill remaining slots after Bucket 1+2.

- [ ] [claude→codex] **T31: Rewrite `generateProspects` in `prospect.html`** as the workbook-driven 3-bucket pipeline per `claude-notes.md` § Phase 2 lead-gen pipeline. Acceptance: 10 leads/day, no static fallbackPool, never repeats yesterday's leads (rolling 14-day exclusion).

- [ ] [claude→codex] **T32: Rolling exclusion list.**
  In `main.js`: maintain `~/Desktop/boola-new/.boola_lead_history.json` (or under `app.getPath('userData')`) with `[{name, date, source}]` entries for last 14 days. Drop entries older than 14 days on each write. `validate-news-lead` and the new vertical/seasonal fetchers must check this file and skip any name already on it. Persist new leads to it after generation.

- [ ] [claude→codex] **T33: Lead card UI upgrade.**
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

- [ ] [claude→codex] **T36: Sync to `~/Projects/boola/`** after T35 passes.

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

  Sync to `~/Projects/boola/` when done. Update `codex-notes.md` confirming the rule from `claude-notes.md` § Mascot is the product is now respected.

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

  Sync to `~/Projects/boola/` when done.

- [ ] [claude→codex] **T22: Mascot + brand identity redesign.**
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
  Sync to `~/Projects/boola/` when done.

- [ ] [claude→codex] **T21: Remove API key field from setup wizard** per `claude-notes.md` § Anthropic API key. Specifically:
  1. Delete the "Anthropic API key" label, input, and any related form code from `setup.html`.
  2. Drop `apiKey` from the saved `profile.json` schema (don't write it; ignore on read).
  3. Continue loading the key from `~/.boola_key` silently in `chat.html` and anywhere else it's used. If `~/.boola_key` is missing, log to console and degrade gracefully (chat shows "Boola is offline — admin setup required" instead of asking the customer for a key).
  4. Add `// SCALABILITY: route through boola backend in production` comment at every `fetch('https://api.anthropic.com/v1/...')` call site in `chat.html` and any other file.
  5. Sync to `~/Projects/boola/`.

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

## Codex Handoff To Claude

- [ ] [codex→claude] **Review latest Boola mascot/emote work in Claude Code.**
  Open the live app folder in Claude Code:
  `cd /Users/alenatipton/Desktop/boola-new && claude`

  Context:
  - Codex removed the floating always-on-top mascot window from normal startup so Boola no longer floats around the desktop.
  - Codex launched the updated app from `~/Desktop/boola-new/`.
  - Codex replaced the old hand-drawn/SVG mascot rendering with approved image-backed assets from:
    - `~/Downloads/Boola Mas.png`
    - `~/Downloads/Boola Expressions.png`
  - New assets live in:
    - `~/Desktop/boola-new/mascot-emotes/`
    - synced copy: `~/Projects/boola/mascot-emotes/`
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
    `cd /Users/alenatipton/Desktop/boola-new && npx electron .`
