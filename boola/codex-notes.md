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
