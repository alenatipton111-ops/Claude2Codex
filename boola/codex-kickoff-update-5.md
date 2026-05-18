# Codex kickoff — Boola Update 5 (2026-05-18)

Paste the block below into Codex.app. Everything above this line is just a reference for Alena; don't paste it.

---

## ⬇️ Paste this into Codex

```
You are working on Boola, a desktop AI sales companion (Electron app). Canonical project folder on this Mac is `/Users/alenatipton/BOOLA/` — edit files there directly, no separate sync step. The 8 AM LaunchAgent launches from the same folder.

STEP 1 — read these end-to-end before touching code:
  1. /Users/alenatipton/ai-notes/boola/claude-notes.md (architecture, rules, canonical emote sheet, what's already shipped)
  2. /Users/alenatipton/ai-notes/boola/tasks.md (full task queue with current checkbox state)
  3. /Users/alenatipton/ai-notes/boola/codex-notes.md (your prior session logs — last entry was 2026-05-12)

Pay special attention in claude-notes.md to:
  (a) "Universal SaaS rule" — no hardcoded regions, products, or company names
  (b) "Canonical mascot emote sheet (2026-05-13)" — the 13 canonical emote names with exact filenames. Use these names in code, NEVER invent variants. Legacy PNGs in mascot-emotes/ (waving, angry, sleeping, confused, etc.) DO NOT match canonical and will be replaced. Code must reference canonical filenames; the in-code fallback handles missing files.
  (c) "Claude-built code log (2026-05-13)" + "single-instance lock (2026-05-15)" — DO NOT rebuild what Claude shipped. OSM lead discovery, geocoding, daily shuffle, chain blocklist, single-instance lock, fallbackPool deletion are all done.

STEP 2 — run Update 5 in this exact order. Do not skip ahead.

  T46 (verification, do this FIRST):
    Manually exercise T26-T32 acceptance criteria on the current code. T26-T32 were either patched in a 2026-05-12 fire-drill or implemented without checkboxes being marked. Confirm each is actually working:
      - T26: `npm run build:lead-rules` regenerates `lead-rules.json` from `lead-rules.xlsx`
      - T27: setup wizard shows 35 sales types and dynamic target-verticals checkboxes
      - T28: `fallbackPool` is fully deleted from prospect.html (grep returns zero matches)
      - T29-T31: leads pipeline runs the 3-bucket flow (news → vertical OSM → seasonal) and produces 10 callable leads
      - T32: `.boola_lead_history.json` exists (under app.getPath('userData')) and prevents repeats
    Mark `[x]` on T26-T32 only after you've verified them, not on Codex's claim. If any fail, fix and document in codex-notes.md before moving on.

  Then in order: T33 → T37 → T38 → T39 → T41 → T40 → T42.
    (T40 is intentionally last-before-QA — it's the largest task and benefits from a stable base.)

CRITICAL CONSTRAINTS — read before T37/T39/T41/T42:

  Gmail OAuth is NOT set up on this Mac (Alena's parent brand blocks the OAuth client; Path A vs Path B is still undecided per claude-notes.md § Pending product decisions). This means:
    - T37 acceptance: build the auto-task creation + sent-emails.json ledger + toast UI. Mark "code complete, send-test deferred."
    - T39 acceptance: build the subject parsing + paragraph fidelity + visible-subject-row UI. Mark "code complete, send-test deferred."
    - T41 acceptance: build the listening → extraction → draft → auto-task flow. The "Send Now" path is code-complete only — do NOT claim end-to-end pass.
    - T42 scenario #5 (email send round-trip): mark partial. Verify subject/body parsing + copy-to-clipboard preservation. Skip the "send + recipient receives" steps.
    Do not work around this by setting up Gmail OAuth — that's a separate decision Alena is making.

  T48 (full regionalization) is NOT in Update 5 scope. T42 scenario #1 (fresh Cincinnati profile, no errors anywhere) will likely surface NYC-flavored residue from `DEFAULT_CUSTOMER_PROFILE.newsFeeds` and other constants. Document those findings in codex-notes.md under "T48 prerequisites surfaced during T42 QA" — don't fix them in this update. Alena will decide whether T48 ships in Update 6.

  T22 (mascot brand redesign) is DEFERRED. Original brief is missing from this Mac. Mascot art is handled separately by Alena. Do NOT work T22.

EMOTE FILENAMES — exact canonical strings (claude-notes.md § Canonical mascot emote sheet):
  happy.png, excited.png, thinking.png, focused.png, surprised.png, frustrated.png,
  decision.png, celebrating.png, tired.png, cooking-money.png, thinking-headset.png,
  cool.png, relaxed.png

  Note for T41: canonical is `thinking-headset` (NOT `headset-thinking`) and `decision` (NOT `holding-email`). The original tasks.md draft had these reversed; tasks.md was corrected 2026-05-18.

  If a canonical PNG is missing on disk (most are, as of 2026-05-18), fall back per claude-notes.md: thinking-headset → thinking.png, decision → happy.png, others → happy.png. Always log a one-line console message; never silently swap. Code references the canonical name regardless of which file is on disk, so swapping art in later is drag-and-drop.

WORKFLOW EXPECTATIONS:
  - After each task: (a) manually test the task's acceptance criteria as documented in tasks.md, (b) mark `[x]` next to the task in tasks.md, (c) append a session log to codex-notes.md with what was implemented, what was tested, what's deferred, (d) git-commit ai-notes if you're in the ai-notes repo.
  - Edit files in /Users/alenatipton/BOOLA/ directly.
  - npm dependencies are already installed (xlsx, docxtemplater, pizzip, googleapis, nodemailer, electron) — T40 step-zero install has already been run, you can proceed directly to coding.
  - Relaunch boola only when explicitly needed for a manual smoke test:
      pkill -9 -f "BOOLA" 2>/dev/null; sleep 2; cd /Users/alenatipton/BOOLA && nohup /opt/homebrew/bin/npx electron . > /tmp/boola.log 2>&1 &
  - Stop on any blocker. Write `[codex→claude]` task at the top of tasks.md describing what you need from Claude/Alena. Do not improvise around architectural rules.

BUDGET: Update 5 is ~7 tasks (T46, T33, T37, T38, T39, T41, T40, T42). Aim to land them with high quality, not speed. Mark partial completions honestly — Alena will see through "code complete" claims that haven't been tested.
```

---

## Reference for Alena (don't paste this part)

**What I changed in the bundle before this kickoff:**
1. Unpacked bundle into `/Users/alenatipton/BOOLA/` (preserving your existing CLAUDE.md, which is more current than the bundle's copy) and `~/ai-notes/`
2. `claude-notes.md` — canonical path now `~/BOOLA/`, emote filename block now explicit + lists legacy PNGs as not-canonical
3. `tasks.md` — all `~/Projects/boola/` → `~/BOOLA/` (17 occurrences); T11/T20 sync steps marked obsolete; T22 marked deferred; T40 has explicit step-zero npm install; T41 emote names corrected to `thinking-headset` + `decision`; T42 "Pause auto-stop" toggle wording disambiguated
4. `npm install` run — xlsx, docxtemplater, pizzip, googleapis, nodemailer all resolve
5. Apr 25 BOOLA/ backed up to `~/BOOLA.apr25-backup.tgz` (91KB)

**Still on you:**
- Generate 13 transparent-bg PNGs at canonical filenames (sheet you sent → individual files). Codex won't block on this; in-code fallback covers gaps.
- Path A vs Path B email decision (blocks full T37/T39/T41/T42 send testing)
- Decide if T48 (full regionalization) goes in Update 6 or sooner

**What's working today (post-unpack):**
- 13 files in BOOLA/: chat.html, main.js, mascot.html, prospect.html, setup.html, reminder.html, followup.html, lead-rules.xlsx, lead-rules.json, sales-knowledge.md, package.json, scripts/build-lead-rules.js, mascot-emotes/ (12 legacy PNGs), boola-shark icons, icon.icns
- Single-instance lock, OSM lead discovery, geocoding, daily shuffle, chain blocklist all live in main.js per claude-notes.md
- Setup wizard with 35 sales types + dynamic target verticals (per codex-notes.md T27 entry)

**Open question Codex won't catch:**
The Codex Handoff item at the bottom of tasks.md (lines 591+) refers to a *prior* review request from Codex to Claude about the 2026-05-07 mascot/emote work. That request is stale now that the canonical 13-emote sheet supersedes those PNGs. You can ignore or delete that section before handing off — your call.
