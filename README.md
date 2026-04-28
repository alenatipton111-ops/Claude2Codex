# AI Notes — Cross-Assistant Handoff Repo

Shared, persistent memory between **Claude Code** and **Codex CLI**.
Every change is git-committed so we have permanent history of plans, decisions, and implementations across all of Alena's projects.

## Layout

```
~/ai-notes/
├── README.md              # this file — the protocol
├── <project>/
│   ├── tasks.md           # active + completed tasks
│   ├── claude-notes.md    # Claude's plans, architecture, edge cases
│   └── codex-notes.md     # Codex's implementation notes, test results
```

One subdir per project. Project names are short, lowercase, and match the project folder name (e.g. `boola`).

## Protocol

1. **Claude** writes plans / architecture / specs to `claude-notes.md` and adds concrete tasks to `tasks.md`.
2. **Codex** reads both, implements, writes results + test output to `codex-notes.md`, marks tasks done in `tasks.md`.
3. Each assistant commits after meaningful changes (see commit format below).
4. **Neither assistant edits the other's notes file.** `claude-notes.md` is Claude's; `codex-notes.md` is Codex's. `tasks.md` is shared but each side appends, doesn't rewrite the other's entries.

## Division of labor

- **Claude**: architecture, product thinking, edge cases, review, system-prompt design
- **Codex**: code edits, running the app, test runs, debugging, repo hygiene
- Both must avoid editing the same project source files at the same time. If overlap is needed, claim it in `tasks.md` first.

## Commit format

```
claude: <project>: <summary>
codex:  <project>: <summary>
```

Examples:
- `claude: boola: plan for migrating ~/.boola_* to userData`
- `codex: boola: implemented userData migration, all tests pass`

## Tasks file format

```markdown
## Active
- [ ] [claude→codex] Migrate file paths to app.getPath('userData')
- [ ] [codex→claude] Need review on IPC boundary in main.js:142

## Completed
- [x] 2026-04-28 [claude] First boola task plan written
```

The `[from→to]` tag makes it obvious who's waiting on whom.
