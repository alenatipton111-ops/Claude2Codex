# boola — tasks

## Active

- [ ] [claude→codex] **T1: Add `CUSTOMER_PROFILE` config block** at the top of `prospect.html` per spec in `claude-notes.md` § Active plan. Migrate existing `RSS_FEEDS`, `NYC_GEO`, `NON_NYC_GEO`, and any `1-800-GOT-JUNK` / NYC-specific strings to reference fields on this object. Mark each migration site with `// SCALABILITY: source from CUSTOMER_PROFILE`. Do not move data to disk yet — that's Phase 3.

- [ ] [claude→codex] **T2: Expand news feed list to 15 NYC sources.** Use the candidate list in `claude-notes.md` § Active plan #2. Before adding each, hit the URL once with curl and confirm it returns valid RSS XML (200 + `<rss>` or `<feed>` root). Substitute any that 404 or block bots. Final feed count must be ≥ 15. Update `CUSTOMER_PROFILE.newsFeeds`.

- [ ] [claude→codex] **T3: Add `extractAddress(html)` helper** in `main.js` per spec in `claude-notes.md` § Active plan #4. Returns first match or `''`. Reject addresses outside `CUSTOMER_PROFILE.region.state` (use the regex's state-code group).

- [ ] [claude→codex] **T4: Extend `lookup-company-info` IPC** in `main.js` to also populate `result.address` using `extractAddress` on each fetched site page (homepage, /contact, /about). Stop early once a non-empty address is found. Existing return shape stays compatible — just adds one field.

- [ ] [claude→codex] **T5: Add `validate-news-lead` IPC handler** in `main.js`. Input: `{ name, headline }`. Calls `lookup-company-info(name)` internally. Returns `{ valid: bool, name, phone, website, address, reason: ?string }`. `valid: true` requires non-empty `phone` AND `address` AND that the looked-up `website` or `name` contains at least one ≥3-char token from `headline` (cross-check from § Edge cases). Bake in a 20s timeout per validation.

- [ ] [claude→codex] **T6: Refactor `buildProspectsFromNews` in `prospect.html`** to be async. After extracting candidates, run them through `validate-news-lead` via IPC with a **3-at-a-time semaphore**. Keep validated leads only. Show progress strip per § Active plan #6: `"Validating leads… N / M checked, K valid"`. Total budget: 90s; abort remaining validations if exceeded and use what's validated so far. Dedupe by canonicalized phone digits at the end.

- [ ] [claude→codex] **T7: Phase-1 fallback** — if validated count < 3 after T6, pad with the existing `fallbackPool` to reach 3. Mark with `// PHASE-1-TEMP:` comment block (Phase 2 will replace). Show banner `"News validation found nothing today; showing curated NYC list."` when fallback is used.

- [ ] [claude→codex] **T8: Update lead card in `chat.html`** — when a lead has `address` and `phone` populated by validation, show `✓ Phone` and `✓ Address` badges (small green chips) above the existing detail rows. Don't duplicate the data; the badges just signal "this is callable today."

- [ ] [claude→codex] **T9: Persist validated schema in `.boola_todays_leads.json`** — store `{name, phone, website, address, validatedAt}` per lead so app restart skips re-validation. On load, if a lead has `validatedAt` for today's date, render directly without re-checking.

- [ ] [claude→codex] **T10: End-to-end smoke test.**
  1. `rm ~/.boola_todays_leads.json` to force regeneration
  2. Relaunch boola (use the relaunch command in the codex-notes standing context)
  3. Open chat → Leads tab → tap ↻
  4. Confirm leads finish generating in ≤90s
  5. Confirm each lead has phone + address (inspect `~/.boola_todays_leads.json`)
  6. Confirm UI shows ✓ badges
  7. Record findings in `codex-notes.md`

- [ ] [claude→codex] **T11: Sync to project folder** — after T10 passes, copy edited files (`prospect.html`, `main.js`, `chat.html`) from `~/Desktop/boola-new/` to `~/Projects/boola/`.

- [ ] [claude→codex] **T12: Acceptance check** — verify each acceptance criterion from `claude-notes.md` § Acceptance criteria. Note any that fail in `codex-notes.md` so Claude can address in the next planning round.

## Completed

_(none yet)_
