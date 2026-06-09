# Codex kickoff — Boola Update 6 + Update 7 (2026-06-06)

Paste the block below into Codex.app. Everything above the triple-backticks is reference for Alena; don't paste it.

---

## ⬇️ Paste this into Codex

```
You are working on Boola, Alena's AI sales-companion Electron app. The canonical
project folder on this Mac is /Users/alenatipton/BOOLA/ — edit files there
directly. There is no separate sync step; the 8 AM LaunchAgent runs from the
same folder.

STEP 1 — read these end-to-end before touching code:
  1. /Users/alenatipton/ai-notes/boola/claude-notes.md
       — architecture, the Universal SaaS rule, the canonical 13-emote mascot
         sheet, the Mascot Is The Product rule (NO idle/random animations),
         the Path A1 OAuth status (Gmail send + read scopes, sensitive-only),
         the date-keeper architecture (lib/time-keeper + lib/heartbeat).
  2. /Users/alenatipton/ai-notes/boola/tasks.md — find the "Update 6 — Rep
         Customization Pass" section and the "Update 7 — Personality +
         Workflow Pass" section. Those are your queue.
  3. /Users/alenatipton/ai-notes/boola/codex-notes.md — your prior session
         logs. T26-T46 done in May; T33/T37/T38/T39/T40/T41 marked complete
         on 2026-05-19; T42 QA pass logged. Read so you don't rebuild what
         already ships.
  4. /Users/alenatipton/BOOLA/personality.md — content source for T55.
         ~120 speech bubble quips across 18 action categories. Load this
         file at Boola startup, parse by "## category_name" headers, pick
         a random line per fire.

STEP 2 — run Update 6 first, then Update 7. Exact order:

   T50  →  T51  →  T52  →  T54  →  T53
   then
   T62  →  T57  →  T56  →  T55  →  T58  →  T59  →  T60  →  T61

   Reasoning:
   - Update 6: build the rep customization core (T50 custom email types,
     T51 layout editor, T52 settings re-org, T54 documents tab upgrades),
     QA at T53.
   - T62 (draggable popups) is small and orthogonal — knock it out first
     in Update 7 so the rest of the work happens against a less-annoying app.
   - T57 (re-add gmail.readonly + re-auth) must precede T58 (email triage)
     because triage needs the scope.
   - T56 (reminder mascot fix) before T55 (speech bubbles) so the mascot
     code path is uniform before you add the bubble system.
   - T59 (call list) before T60 (CSV export) — export needs the list.
   - T58 needs Alena to do a Google Console step + re-auth in between.
     See "BLOCKING DEPENDENCIES" below.

CRITICAL ARCHITECTURE CONSTRAINTS — read before touching anything:

  (a) Universal SaaS rule — NO hardcoded regions, products, company names,
      or "if salesType === X" branches. All sales-type/region behavior comes
      from lead-rules.xlsx (workbook) and profile.json. New customizations
      Alena introduces (custom email types, UI layout) must live in
      profile.json so they follow the rep across reinstall and SaaS launch.

  (b) Canonical mascot emote sheet (13 names) — happy, excited, thinking,
      focused, surprised, frustrated, decision, celebrating, tired,
      cooking-money, thinking-headset, cool, relaxed. Reference these by
      exact filename. Legacy PNGs (waving, angry, sleeping, etc.) DO NOT
      match canonical and should not be referenced in new code. If a
      canonical PNG file is missing on disk, fall back per the rule
      documented in claude-notes.md and log a one-line console message.

  (c) Mascot is the product — NO idle/random animations. Speech bubbles
      and emote changes in T55 fire ONLY on real user actions (send email,
      mark todo done, etc.). NO setInterval-based "Boola randomly says
      something" code. The bubble frequency dampener (1-in-2 fire rate) is
      to prevent fatigue ON action fires, not to enable idle fires. This
      is non-negotiable per claude-notes.md.

  (d) Path A1 OAuth — Boola sends via Gmail API (not nodemailer SMTP) using
      gmail.send scope only. T57 RE-ADDS gmail.readonly so email triage
      (T58) can function. Both scopes are "sensitive" tier — verification
      stays on the standard track, no CASA audit. Do NOT add
      mail.google.com (full Gmail) — that's restricted scope and breaks
      the SaaS unit economics later.

  (e) State persistence — per-customer/per-rep data lives in
      app.getPath('userData')/ files OR profile.json (also in userData).
      NEVER write to ~/.boola_* anymore (that's pre-2026-05 scalability
      debt being paid down). New files in this update:
        - app.getPath('userData')/call-lists.json (T59)
        - app.getPath('userData')/popup-positions.json (T62)
        - profile.json gets new keys: customEmailTypes (T50), uiLayout (T51)

  (f) Date-aware code — use lib/time-keeper.js helpers (todayISO,
      localDateISO, daysBetween, addDaysISO). NEVER use
      new Date().toISOString().slice(0,10) directly — that's UTC and gives
      wrong dates in the evening Eastern timezone. See main.js for examples.

  (g) Email send fallback chain in main.js send-email handler:
      app-password (creds.appPassword set) → OAuth Gmail API
        (creds.refreshToken + clientId + clientSecret) → error.
      Don't reintroduce SMTP. Don't break the app-password stopgap path —
      it's a dev fallback for when OAuth isn't set up yet.

BLOCKING DEPENDENCIES — sequence so Alena isn't surprised:

  Before starting T57 implementation, post a [codex→claude] note in
  tasks.md asking Alena to:
    1. Open https://console.cloud.google.com/auth/scopes (UseBoola project)
    2. Add https://www.googleapis.com/auth/gmail.readonly to Data Access
       sensitive scopes. Save.
    3. Open https://myaccount.google.com/permissions → revoke Boola
  THEN you update scripts/connect-gmail.js SCOPES, Alena re-runs it, you
  test gmail-search + gmail-read IPC handlers work, then proceed with T58.

  Without this, T58 will silently fail with permission errors. Don't ship
  T58 against a broken scope state — block, ask, wait.

WORKFLOW EXPECTATIONS:

  - Edit files in /Users/alenatipton/BOOLA/ directly.
  - After EACH task: (a) test the acceptance criteria as documented in
    tasks.md, (b) mark [x] next to the task, (c) append a session log to
    codex-notes.md (what was implemented / what was tested / what's
    deferred), (d) git commit + push to ai-notes.
  - Relaunch boola only when a task needs a manual smoke test:
       pkill -9 -f "BOOLA/node_modules/electron/dist" 2>/dev/null; sleep 2;
       cd /Users/alenatipton/BOOLA && nohup /opt/homebrew/bin/npx electron .
         > /tmp/boola.log 2>&1 &
  - Tests with mocked deps are fine for harness coverage, but mark anything
    that needs human eyes (visual layout, animation timing, mascot feel,
    sound) as "manual smoke needed" in the codex-notes session log.
  - Stop on any blocker. Write a [codex→claude] task at the top of tasks.md
    describing what you need from Claude or Alena. Do not improvise around
    architecture rules.

DELIVERABLES PER TASK:

  T50 (custom email types) — modal in Email pane, schema in profile.json,
       dropdown integration, allowVariance toggle behavior, T37 follow-up
       cadence wiring.
  T51 (UI layout editor) — iPhone-style drag-to-reorder + hide-tabs,
       overflow menu for hidden items, persistence in profile.json.
  T52 (Settings re-org) — 8 collapsible sections in the order specified,
       atomic save, deep-linkable.
  T54 (Documents tab upgrades) — in-tab upload button, ~/Desktop save with
       smart naming, region + full-screen capture, natural-language
       placeholder syntax (insert NAME here) alongside {{name}}, lookup-table
       field type for discount-driven price columns. Alena's Royal Blue
       Template is on her Desktop as a real-world test target — exercise
       against it.
  T53 (Update 6 QA) — full smoke test, mark each scenario ✅/⚠️/❌.

  T62 (draggable popups) — add app-region:drag handles + × close buttons to
       reminder.html, prospect.html, followup.html, setup.html. Persist
       positions per-window in popup-positions.json. Preserve swim-up
       animation on prospect window.
  T57 (gmail.readonly re-auth) — update SCOPES, walk Alena through console
       step, verify search + read IPC handlers work.
  T56 (reminder mascot canonical) — replace legacy SVG/PNG refs in
       reminder.html with canonical emote system; emote varies by reminder
       time (10am=excited, 1pm=focused, 3pm=cool).
  T55 (speech bubbles) — lib/speech-bubble.js component, mascot.html
       container + IPC listener, main.js trigger wiring at all action
       points, personality.md loader, 1-in-2 dampening, NO idle fires.
  T58 (email triage) — Triage Inbox button in chat, Claude search+read
       flow, result-card rendering with Open in Gmail + Draft Reply.
  T59 (call list) — Call List tab, schema in userData, lead-card "Add to
       call list" button, bulk "Make list from selected" on Leads tab,
       status pills, click-to-call via tel: links.
  T60 (CSV export) — lib/csv-export.js (RFC 4180 compliant), per-list
       Export CSV button, file lands at ~/Desktop with smart naming.
  T61 (Update 7 QA) — full smoke test, mark each scenario.

BUDGET:

  Update 6 is roughly 12-16 hours of focused work. Update 7 is roughly
  10-14 hours. Aim for high quality, not speed. Mark partial completions
  honestly — Alena will see through "code complete" claims that haven't
  been tested.

  Don't claim a task done without running its acceptance criteria.

When you finish T61, post a summary to codex-notes.md and stop. Don't
start a new update without Alena's go-ahead.
```

---

## Reference for Alena (don't paste this part)

**Pre-handoff checklist for you to do BEFORE pasting:**

1. ✅ Confirm `~/Desktop/Royal-Blue-Template.docx` exists — that's Codex's T54 real-world test target. Open it once, paste in your terms-of-sale text, save. Don't move/rename.
2. ✅ Confirm `~/BOOLA/personality.md` exists — Codex needs it for T55.
3. 🟡 OPTIONAL preflight for T57: add `https://www.googleapis.com/auth/gmail.readonly` to your consent screen scopes now ([link](https://console.cloud.google.com/auth/scopes)). Codex will guide you through revoke + re-auth when it gets there. Doing the scope-add now means less context-switching mid-task.

**Sequencing reminders:**

- Codex will pause at T57 and write a `[codex→claude]` note in tasks.md telling you to revoke + re-auth. Watch for that — without you doing the Google-side steps, T58 will fail silently.
- Update 6 ships standalone. If you want to launch + use Update 6 features for a week before Update 7 lands, that's fine — tell Codex to stop after T53.

**What you'll see when Codex runs through:**

- 13 commits to your ai-notes git (one per task + per-update QA pass + kickoff doc)
- Each commit messages prefixed `codex: boola: T## ...`
- New files in `~/BOOLA/`:
  - `lib/speech-bubble.js`, `lib/csv-export.js`
  - Possibly `lib/region-picker.js` (for T54 screen capture)
- Modified files: `chat.html`, `main.js`, `setup.html`, `mascot.html`, `reminder.html`, `prospect.html`, `followup.html`, `profile.json` schema, `personality.md` consumed (not modified)
- New userData files appearing in `~/Library/Application Support/boola/`: `call-lists.json`, `popup-positions.json`

**If you want to monitor progress:**

```bash
# in a separate terminal window, run:
watch -n 30 'cd ~/ai-notes && git log --oneline -10'
```

That'll show new commits as Codex lands them. Or just check the Claude2Codex GitHub repo periodically.

**When you're ready, paste the block above into Codex.app and let it run.**
