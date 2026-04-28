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

## Active plan
_(Claude fills this in when a task is started)_
