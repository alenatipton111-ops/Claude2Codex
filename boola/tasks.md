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
