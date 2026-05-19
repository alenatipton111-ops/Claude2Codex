# boola — Codex notes

_Implementation notes, file-level changes, test output, debugging. Claude reads this; Claude does not edit it._

---

## Standing context

- Run/relaunch boola:
  ```bash
  pkill -9 -f "boola-new" 2>/dev/null; sleep 2; cd /Users/alenatipton/Desktop/boola-new && nohup npx electron . > /tmp/boola.log 2>&1 &
  ```
- After meaningful edits, relaunch so Alena sees results immediately.

## Recent implementations
_(Codex fills this in after each task — newest at top)_

## 2026-05-19T00:00:00-05:00 - T33 lead card UI upgrade

### Implemented
- Updated `chat.html` lead ingestion to preserve workbook metadata instead of discarding it and re-guessing verticals only from regexes.
- Lead cards now show workbook vertical, confidence chip with green/yellow/red classes, why-now text, suggested buyer, and source chip.
- Expanded lead details now show why-now, suggested buyer, source link, phone/address validation badges, and the existing LinkedIn/ZoomInfo actions.
- Added a `📧 Cold email` quick action on each lead. It pre-fills the Email pane with context, builds a subject/body from workbook `Cold_Email_Rules`, stores the raw subject/body on the result element, and shows the draft in the Email pane.

### Test output
- `node --check main.js` passed.
- Extracted script blocks from `prospect.html`, `setup.html`, `chat.html`, and `mascot.html` compiled with `vm.Script`.
- Hidden Electron `chat.html` harness injected a sample workbook lead, expanded it, and verified vertical chip, confidence chip, why-now, suggested buyer, source chip, and Cold email button render.
- Same harness clicked Cold email and verified the Email pane became active with raw subject/body plus workbook context populated.

## 2026-05-18T16:06:00-05:00 - T46 verification gate for T26-T32

### Verified
- T26: `npm run build:lead-rules` passed in `/Users/alenatipton/BOOLA` and rebuilt `lead-rules.json` from `lead-rules.xlsx` with all 8 workbook tabs. Row counts: Master_B2B_Lead_Rules 105, Bot_Instructions 18, Sources 42, Category_Summary 35, Cold_Email_Rules 46, Regional_Source_Rules 55, Source_Routing_Logic 12, Source_Scoring_Rules 9.
- T27: Hidden Electron setup harness loaded `setup.html`; Sales type dropdown rendered 35 options from `lead-rules.json`. Changing to `tech_sales_saas_it` dynamically redrew target vertical checkboxes to healthcare & life sciences, financial services & insurance, and professional services / legal / accounting.
- T28: `rg -n "fallbackPool" prospect.html` returned zero matches.
- T29-T31: Live Electron harness triggered the actual prospect window `start-prospect` flow. After fixes below, the pipeline produced 10 callable leads with phone numbers, verticals, confidence scores, and why-now text in about 73 seconds. The list came from the workbook/OSM vertical path and did not use any static fallback.
- T32: `/Users/alenatipton/Library/Application Support/boola/.boola_lead_history.json` exists and is appended by `leads-ready`. Consecutive forced-regeneration runs skipped prior names and appended different OSM vertical leads, confirming the rolling exclusion is enforced.

### Fixes made during verification
- `prospect.html`: allowed Bucket 2 vertical leads to backfill all the way to 10 when news/seasonal buckets are dry; capped weak news/structured validation batches so bad candidates do not consume the whole refresh budget.
- `main.js`: stopped doing slow lookup enrichment for OSM vertical candidates that only lacked address. Vertical leads require a phone; candidates without a phone are skipped instead of blocking the queue.

### Test output
- `node --check main.js` passed.
- `node --check scripts/build-lead-rules.js` passed.
- Extracted script blocks from `prospect.html`, `setup.html`, `chat.html`, and `mascot.html` compiled with `vm.Script`.
- Live T46 lead harness report: 10 leads, 10 callable, 10 with vertical, 10 with confidence, 10 with why-now.

## 2026-04-30T13:06:11.521Z - Phase 1 / Phase 1.5 implementation batch

### Implemented
- T1/T13: Added bootstrap CUSTOMER_PROFILE blocks and targetSignals defaults. Main process now loads/saves profile.json from app.getPath('userData'); prospect/chat refresh profile via IPC.
- T2: Expanded RSS feed config to 15 verified XML feeds. Verified valid: NY Post Business, NY YIMBY, Commercial Observer, amNewYork, Brownstoner, Brooklyn Eagle, PIX11 Business, City Limits, Real Estate Weekly, 6sqft, Bronx Times, QNS, Brooklyn Paper, Construction Dive, Restaurant Dive. Rejected/substituted: Crain's/news.xml and The Real Deal returned HTML; Bisnow, Bizjournals, Patch, City & State, NYREJ, Curbed were blocked/404/410.
- T3/T4/T5: Added extractAddress(html), extended lookup-company-info result with address, and added validate-news-lead requiring phone + address + headline/name token cross-check with 20s timeout.
- T6/T7/T9: Reworked prospect generation to async validation with 3-at-a-time concurrency, progress text, 90s budget, phone dedupe, daily validated cache, and PHASE-1-TEMP static fallback with phone/address schema.
- T8: Lead cards now show green Phone and Address validation chips when those fields exist.
- T14/T15/T16/T17: Removed active use of license-based opening/warehouse/DCWP inactive signals; added action-based Socrata handlers for construction permits, renovation permits, job filings, illegal dumping, plus WARN-only closings; wired these through the same validation gate and updated labels/reasons.
- T18: Added setup.html first-run wizard, profile-load/profile-save/profile-exists IPC, startup gating when profile.json is absent, and Settings gear in chat to reopen setup. API key is written to ~/.boola_key and stripped before profile.json persistence.
- T11/T20: Synced main.js, prospect.html, chat.html, setup.html from ~/Desktop/boola-new to ~/Projects/boola/.

### Test output
- RSS curl validation: 15 configured feeds returned HTTP 200 with <rss> root after substitutions.
- Socrata smoke checks: ipu4-2q9a, ic3t-wcy2, and erm2-nwe9 each returned HTTP 200 JSON for $limit=1.
- Syntax: node -c main.js passed. Extracted <script> blocks from prospect.html, chat.html, setup.html compiled with vm.Script.
- Relaunch: ran standing relaunch command from this file. /tmp/boola.log had no crash output immediately after launch.

### Not fully verified here
- T10/T19 UI click-through was not fully possible from this sandbox. I could not inspect the live Electron windows or process list (ps/pgrep blocked), so Claude should manually verify setup window, Leads refresh, validation duration, badges, and targetSignals toggling in the app UI.
- T12 acceptance is partially satisfied by static checks, but final acceptance should wait for the manual UI smoke above and inspection of ~/.boola_todays_leads.json after a real refresh.

## 2026-04-30T14:42:13.555Z - Validator hardening follow-up

### Implemented
- Tightened validate-news-lead headline cross-check: it now compares headline/name tokens against the discovered website, avoiding the previous self-match against the input company name.
- Removed the overly aggressive prospect-side abort after three ordinary validation misses; generation now uses the 90s budget and 3-at-a-time concurrency without bailing just because several candidates lack phone/address.
- Synced updated main.js, prospect.html, chat.html, and setup.html to ~/Projects/boola/.

### Test output
- node -c ~/Desktop/boola-new/main.js passed.
- Extracted scripts from prospect.html, chat.html, and setup.html compiled with vm.Script.
- Relaunched boola with the standing command; /tmp/boola.log had no crash output immediately after launch.

### Remaining
- T10/T19 still need live UI click-through: setup save, Leads refresh timing, badge rendering, and targetSignals toggle verification.

## 2026-04-30T19:07:47.444Z - Call Mode follow-up workflow

### Implemented
- Added name + email intake fields to Call Mode.
- Added profile-backed follow-up email template and manual attachment document names to setup.html/profile defaults.
- Updated Call Mode to analyze seller-side transcript for email follow-up cues versus callback cues.
- Email cue: Boola asks through the follow-up popup before drafting, then previews the email and exposes Send only after preview.
- Callback cue: Boola asks to add a high-priority task like "Name Business follow up call date/time" to the Tasks tab.
- Drafted call follow-up emails use the rep template, call transcript, contact name/email, optional notes, and profile docs.
- Added red attachment reminder block in the preview: ATTACH <DOC> DOC TO EMAIL AND DELETE THIS TEXT. The same reminder is inserted at the top of the Gmail body; no attachments are sent automatically.
- Synced main.js, chat.html, setup.html, followup.html, and prospect.html to ~/Projects/boola/ and relaunched Boola.

### Test output
- node -c ~/Desktop/boola-new/main.js passed.
- Extracted scripts from chat.html, setup.html, followup.html, and prospect.html compiled with vm.Script.
- Relaunched Boola with the standing command; /tmp/boola.log had no crash output immediately after launch.

### Manual smoke needed
- In Call tab, enter name/email, start listening, say an email cue such as "I'll send you a follow-up email with our service offerings", stop listening, confirm the popup asks to draft, confirm preview has red attachment reminder, then click Send and verify Gmail compose opens to the entered email.
- Repeat with a callback cue such as "I'll give you a call back tomorrow at 2 pm", stop listening, confirm Boola asks to add the callback task and that it appears in Tasks.

## 2026-04-30T19:45:15.188Z - Embedded sales knowledge base

### Implemented
- Added main-process IPC handler sales-knowledge-load that reads bundled sales-knowledge.md from the app folder with a 24k character cap.
- Added renderer-side SALES_KNOWLEDGE cache plus ensureSalesKnowledge() so first chat/email/call prompt waits for the knowledge file if it has not loaded yet.
- Injected the validated sales knowledge base into buildSystemPrompt() for general sales chat and call coaching.
- Injected the same knowledge base into getEmailExpertSystem() for email drafting.
- Injected the knowledge base into call follow-up drafting so post-call emails use the validated frameworks, email rules, objection handling, CTAs, and vertical playbooks.
- Synced main.js, chat.html, and sales-knowledge.md to ~/Projects/boola/ and relaunched Boola.

### Test output
- node -c ~/Desktop/boola-new/main.js passed.
- Extracted scripts from chat.html, setup.html, followup.html, and prospect.html compiled with vm.Script.
- Relaunched Boola with the standing command; /tmp/boola.log had no crash output immediately after launch.

### Manual smoke needed
- Ask Boola for a cold call opener, objection response, or voicemail script and confirm it cites/uses the sales-knowledge frameworks in behavior.
- Generate a cold email and a post-call follow-up and confirm the output follows the markdown rules: concise, specific, one CTA, no em dashes, no sign-off.

## 2026-05-06T17:57:49Z - Mascot/icon-only update from approved shark reference

### Implemented
- Updated the floating mascot SVG in `mascot.html` to better match the approved Boola shark reference: shorter/chubbier body, small dorsal fin, compact curved tail, rounded side fins, large navy eyes, blush, open smile, friendly teeth, and red/pink tongue.
- Kept the existing mascot IPC/expression hooks intact (`happy`, `thinking`, `excited`, `headset`, `celebrate`, `sleepy`, etc.) so call mode and chat mood changes still route through the same code.
- Updated the small shark SVG used in the chat header and the swimming shark SVG in the Leads popup to use the same approved-shark proportions.
- Reverted the earlier wordmark/header/layout work from this pass; this update is intentionally mascot/icon-only.
- Regenerated `icon.icns` from the shark-only mascot and kept the existing `app.dock.setIcon()` hook in `main.js`.
- Removed the temporary `boola-logo.svg` wordmark asset from both `~/Desktop/boola-new/` and `~/Projects/boola/`.
- Synced `mascot.html`, `chat.html`, `prospect.html`, `setup.html`, `main.js`, and `icon.icns` to `~/Projects/boola/`.

### Test output
- Inline mascot SVG passed `xmllint`.
- Leads popup swimmer SVG passed `xmllint`.
- `node --check ~/Desktop/boola-new/main.js` passed.
- Extracted scripts from `mascot.html`, `prospect.html`, `setup.html`, and `chat.html` compiled with `vm.Script`.
- Rendered a local Quick Look preview of the updated mascot SVG for visual QA.
- Regenerated `icon.icns` with `iconutil` outside the sandbox.
- Relaunched Boola; `/tmp/boola.log` had no crash output immediately after launch.

### Manual smoke needed
- Confirm the floating on-screen mascot now matches the approved reference closely enough at app size.
- Confirm the dock icon shows the shark-only mascot after relaunch.

## 2026-05-07T02:18:06Z - Approved shark asset correction

### Implemented
- Replaced the visible floating mascot with `boola-shark.png`, generated directly from the approved Boola reference image instead of continuing to hand-build the shark in SVG.
- Kept the old SVG hidden only as a compatibility layer for existing expression/accessory DOM calls; the on-screen shark now comes from the approved asset.
- Updated the Leads popup swimmer and chat header avatar to use the same approved shark PNG.
- Added `boola-shark-icon.png` as a square icon source and changed `main.js` to set the macOS dock icon from the PNG via `nativeImage` instead of the `.icns` path Electron rejected.
- Synced updated files/assets to `~/Projects/boola/`.

### Test output
- `node --check ~/Desktop/boola-new/main.js` passed.
- Extracted scripts from `mascot.html`, `prospect.html`, and `chat.html` compiled with `vm.Script`.
- Relaunched Boola after the PNG dock icon fix. `/tmp/boola.log` showed normal Chromium/camera warnings only; no dock icon load failure after the fix.

### Manual smoke needed
- Visually confirm the floating mascot no longer shows the bad multi-part SVG mouth and now matches the approved reference asset.

## 2026-05-07T13:11:51Z - Removed floating mascot window

### Implemented
- Stopped creating the always-on-top floating `mascotWindow` during normal app startup.
- Disabled the random floating mascot expression loop by no longer starting it from `createMainWindows()`.
- Replaced chat positioning that depended on the floating mascot with a stable default chat position near the bottom-right of the primary display.
- Updated call-mode hotkey, lead lookup, follow-up flow, and chat toggle handling to work without a mascot window anchor.
- Added a macOS `activate` handler so clicking the dock icon brings the chat window back.
- Synced `main.js` to `~/Projects/boola/` and relaunched Boola.

### Test output
- `node --check ~/Desktop/boola-new/main.js` passed.
- Relaunched Boola with the standing command; `/tmp/boola.log` had no crash output immediately after launch.

### Manual smoke needed
- Confirm the standalone floating shark no longer appears on the desktop after relaunch.
- Confirm Cmd+Shift+C still opens Call Mode and dock activation can bring back the chat window.

## 2026-05-07T16:11:35Z - Approved Boola mascot emote system

### Implemented
- Extracted a new transparent `mascot-emotes/` asset set from the approved base mascot and approved emote sheet.
- Added reusable mascot states for base, open-mouth, throwing-money, swimming, thinking, confused, cooking-money, email-reject, angry, waving, excited, and sleeping.
- Mapped legacy expression names like `happy`, `money`, `sleepy`, `celebrate`, and `headset` to the approved new assets so existing app flows keep working.
- Updated the dormant mascot window to swap approved emote PNGs through one reusable expression map instead of trying to redraw the character with SVG shapes.
- Updated the visible chat header avatar so expression IPC events now show the new Boola emote states even though the floating mascot window is disabled.
- Updated the Leads/prospect popup to use the approved swimming emote.
- Updated the macOS dock icon source to use the approved base mascot asset.
- Synced updated assets and files to `~/Projects/boola/`.

### Test output
- `node --check ~/Desktop/boola-new/main.js` passed.
- Extracted scripts from `chat.html`, `mascot.html`, and `prospect.html` compiled with `vm.Script`.
- Verified all 12 emote PNG assets exist in `~/Desktop/boola-new/mascot-emotes/`.
- Relaunched Boola; `/tmp/boola.log` showed only the existing Chromium `kern.hv_vmm_present` warning and no app crash.

### Manual smoke needed
- Open chat and trigger Thinking/Email/Tasks flows to confirm the header Boola swaps to the matching approved emotes.
- Confirm the prospect popup now shows the swimming Boola emote.
- Note: the approved sheet labels #8 as Angry and #10 as Bored, while the task text requested Waving and Sleeping. I kept the sheet's Angry asset available, mapped Waving to the approved raised-fin happy pose, and mapped Sleeping/Sleepy/Bored to the approved sleepy/bored pose.

## 2026-05-07T21:45:12Z - T25 restored floating desktop mascot

### Implemented
- Read `claude-notes.md` § Mascot is the product and implemented T25 only.
- Restored `createMascotWindow()` to normal startup so Boola is again an always-on-top transparent frameless desktop mascot.
- Restored click-to-toggle chat behavior through the existing `toggle-chat` IPC route.
- Restored mascot dragging through `move-mascot`, including lightweight persistence to `app.getPath('userData')/mascot-position.json`.
- Restored chat positioning relative to the mascot using the original anchor behavior: `chatWindow.setPosition(b.x - 290, b.y - 700)`.
- Kept the new approved `mascot-emotes/` PNG system for the floating mascot.
- Kept idle/random animation behavior disabled: `startRandomExpressions()` is a no-op, Test Mood swaps an expression without swimming to a random stage, and normal expression swaps set no animation.
- Left task reminder, lead popup, sparkles, money effects, and the chat header avatar intact.
- Synced `main.js` and `mascot.html` to `~/Projects/boola/`.

### Test output
- `node --check ~/Desktop/boola-new/main.js` passed.
- Extracted scripts from `mascot.html`, `chat.html`, and `prospect.html` compiled with `vm.Script`.
- Relaunched Boola from `~/Desktop/boola-new/`; `/tmp/boola.log` had no crash output immediately after launch.

### Manual smoke needed
- After relaunch, confirm floating Boola appears bottom-right, sits still, toggles chat on click, and can be dragged.
- Confirm Test Mood changes the mascot expression without moving Boola across the screen.
- Confirm no random mascot animation fires after 45+ minutes.

## 2026-05-11T23:33:05Z - T26 lead-rules workbook build

### Implemented
- Read `claude-notes.md` § Phase 2 and `tasks.md`; started the Update 5 sequence at T26 in the canonical `/Users/alenatipton/Projects/boola/` app folder.
- Installed the `xlsx` dependency locally for workbook conversion.
- Added `scripts/build-lead-rules.js`.
- Added `npm run build:lead-rules` to `package.json`.
- Built `lead-rules.json` from `lead-rules.xlsx` with all 8 required tabs:
  - `Master_B2B_Lead_Rules`
  - `Bot_Instructions`
  - `Sources`
  - `Category_Summary`
  - `Cold_Email_Rules`
  - `Regional_Source_Rules`
  - `Source_Routing_Logic`
  - `Source_Scoring_Rules`
- The generated JSON includes raw sheet rows plus indexes for category summary, master rules, sources, regional source rules, routing logic, scoring rules, and cold email rules.

### Test output
- `npm install xlsx --save` completed; npm reported 3 existing high-severity audit findings.
- `npm run build:lead-rules` passed and emitted `/Users/alenatipton/Projects/boola/lead-rules.json`.
- Row counts from the build:
  - Master_B2B_Lead_Rules: 105
  - Bot_Instructions: 18
  - Sources: 42
  - Category_Summary: 35
  - Cold_Email_Rules: 46
  - Regional_Source_Rules: 55
  - Source_Routing_Logic: 12
  - Source_Scoring_Rules: 9
- `node --check scripts/build-lead-rules.js` passed.
- Parsed `package.json` and `lead-rules.json` successfully with `JSON.parse`.
- Spot check: default junk-removal sales type resolves to property management / multifamily, commercial real estate / office buildings, and construction / restoration.

### Notes
- `/Users/alenatipton/Projects/boola/` is not currently a git repo, so the per-task commit is recorded in `~/ai-notes` only.

## 2026-05-11T23:37:13Z - T27 setup wizard sales type and verticals

### Implemented
- Added workbook-backed `salesType` to `DEFAULT_CUSTOMER_PROFILE` with Alena's default `facilities_commercial_services_junk_sales`.
- Added workbook-backed `targetVerticals` defaults for the junk-removal sales type:
  - property management / multifamily
  - commercial real estate / office buildings
  - construction / restoration
- Added `region.radiusMiles` default of 25.
- Updated profile merge logic so `targetVerticals` persists and falls back safely.
- Rebuilt `setup.html` with:
  - Sales type dropdown sourced from `lead-rules.json` `Category_Summary` rows.
  - Dynamic target-vertical checkboxes sourced from `lead-rules.json` `Master_B2B_Lead_Rules` rows for the selected sales type.
  - Priority-score display on each target vertical.
  - Territory radius number input.
  - Save path for `profile.salesType`, `profile.targetVerticals`, `profile.verticals`, and `profile.region.radiusMiles`.

### Test output
- `node --check ~/Projects/boola/main.js` passed.
- Extracted scripts from `setup.html`, `chat.html`, `prospect.html`, and `mascot.html` compiled with `vm.Script`.
- Confirmed `lead-rules.json` exposes 35 sales types.
- Confirmed default junk-removal sales type exposes 3 target verticals.

### Manual smoke needed
- Open Settings/setup and confirm the dropdown renders all 35 sales types.
- Change sales type and confirm target vertical checkboxes update dynamically.
- Save and inspect `profile.json` to confirm `salesType`, `targetVerticals`, and `region.radiusMiles` persist.

## 2026-05-12T13:14:20Z - Fixed old repeating lead fallback

### Implemented
- Responded to Alena's report that Leads was still producing the old pre-update companies.
- Found the root cause in `prospect.html`: the old `fallbackPool` still existed and `generateProspects()` still padded with Hudson Yards, Marriott Marquis Times Square, and NYU Langone whenever validation was thin.
- Deleted the static `fallbackPool` and all `// PHASE-1-TEMP` fallback usage from `prospect.html`.
- Updated today's lead cache gate so old cached fallback leads are not treated as valid:
  - cache must contain at least 10 leads
  - each lead must be validated today
  - `source:'Backup'` is rejected
- Wired `generateProspects()` to the workbook-driven buckets:
  - Bucket 1: regional news/structured action signals capped at 3
  - Bucket 2: `fetch-vertical-leads` fills toward 7 using selected workbook target verticals
  - Bucket 3: `fetch-seasonal-leads` fills remaining spots from seasonal workbook rows
- Added lead dedupe, confidence decoration, default vertical/buyer metadata, and a no-static-fallback empty state.
- Relaunched Boola from `/Users/alenatipton/Projects/boola/`.

### Test output
- `node --check ~/Projects/boola/main.js` passed.
- Extracted scripts from `prospect.html`, `setup.html`, `chat.html`, and `mascot.html` compiled with `vm.Script`.
- `rg` confirmed no `fallbackPool`, `PHASE-1-TEMP`, Hudson Yards Development, Marriott Marquis Times Square, or NYU Langone Medical Center remain in `main.js`/`prospect.html`.
- Relaunch log showed only the existing Chromium `kern.hv_vmm_present` warning and no app crash.

### Manual smoke needed
- Open Leads and hit refresh. It should ignore the old 3-lead cache and attempt the workbook pipeline instead of showing the static anchors.
- If it returns fewer than 10, that is now a real validation/search issue to tune, not the old static fallback masking the problem.
