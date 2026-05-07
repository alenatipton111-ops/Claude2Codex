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
