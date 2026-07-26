# Claude Cockpit — Design Spec

Date: 2026-07-26
Status: Approved (brainstorm session)

## Purpose

Local web dashboard that wraps Claude Code: one-click buttons for daily workflows
(research runs, code sessions, vault maintenance), live view of running sessions,
and usage metrics the terminal can't show — all wired into the Obsidian vault at
`C:\Users\rishi\Documents\Vault`.

Differentiator: metrics + session history accumulate in the vault as wikilinked
notes. Launcher UIs are commodity; the compounding Obsidian work-history is the moat.

## Decisions (locked)

- Scope: all-in thin shell — minimal slice of launcher + runs view + metrics, together.
- Form factor: local web app (Bun + Hono server, browser UI, localhost only).
- V1 button categories: research runs, code sessions, vault maintenance. Video workflows excluded.
- Permission strategy: per-button allowlists via `--settings` files. No
  `--dangerously-skip-permissions`. Blocked action fails visibly with fix hint.
- Thin-shell rule: never reimplement Claude Code logic. Spawn the CLI, parse its
  outputs. CLI version churn must only touch two adapter files (spawn args, log schema).

## Architecture

```
Browser UI  <--SSE/HTTP-->  Local server (Bun + Hono)  <--spawn-->  claude -p (headless)
                                   |
                                   +-- reads ~/.claude/projects/**/*.jsonl  (metrics)
                                   +-- writes Documents/Vault/...           (rollups, summaries)
```

## Components

### Runner
- `POST /runs` spawns:
  `claude -p "<prompt>" --output-format stream-json --settings <button-settings.json>`
  with `cwd` from the button def.
- Streams stream-json events to the browser over SSE.
- Tracks run state: running / done / failed. Keeps transcript per run.

### Button registry
- Folder of JSON button definitions, one file per button:

```json
{
  "id": "vault-inbox-sweep",
  "label": "Ingest _raw inbox",
  "prompt": "Process all files in _raw/, file into _wiki with frontmatter...",
  "cwd": "C:/Users/rishi/Documents/Vault",
  "settings": "./allowlists/vault-write.json",
  "promptVars": []
}
```

- `promptVars` = optional named inputs (research question, project path). Nonempty
  -> button opens a small form before launch.
- Starter buttons (~6): research-question, code-review-diff, resume-project,
  fix-tests, vault-inbox-sweep, vault-orphan-scan.
- Per-button allowlist settings files live in `allowlists/`, scoped to what each
  workflow needs (vault writes, WebSearch, git, etc.).

### Metrics reader
- Parses `~/.claude/projects/**/*.jsonl` session logs on demand.
- Extracts: tokens in/out, cost, duration, tool-call counts, models, per-project totals.
- No database. Parse + in-memory cache, invalidated on file mtime.
- Version-tolerant: unknown fields skipped, never crash on schema drift.

### Vault bridge
- Per-run summary note: `_brain/sessions/YYYY-MM-DD-<slug>.md`, wikilinked to the
  project note.
- Daily rollup: `_brain/metrics/YYYY-MM-DD.md` with tokens/cost/runs table.
- All writes UTF-8, YAML frontmatter, kebab-case filenames (vault standards).

## UI — one page, three panes

1. Launch grid — buttons grouped by category; click -> run (or var form -> run).
2. Runs strip — live cards: streaming output tail, status, elapsed, cost-so-far;
   click -> full transcript view.
3. Metrics pane — today/week: cost, tokens, runs by button, sparkline trend.

Design constraints (from user global rules): DESIGN.md written before any frontend
code; OKLCH color tokens; no Inter/Roboto/Arial; no purple-blue gradients; one
justified aesthetic risk. Visual detail deferred to implementation (frontend-design
+ dataviz skills at build time).

## Error handling

- Blocked permission -> run fails; card shows the exact denied tool and which
  allowlist file to edit.
- Log schema drift -> parser skips unknowns; banner shows "N sessions unparsed
  (CLI version X)".
- Vault write failure -> run still succeeds; sync marked pending with retry button.

## Testing

- Unit: jsonl parser against fixture files captured from real `~/.claude/projects` logs.
- Unit: vault note writer (frontmatter, wikilinks, UTF-8 on Windows).
- Smoke: spawn trivial `claude -p "say ok"` end-to-end through SSE.
- No browser E2E in v1.

## Traded away (explicit)

- No live permission relay — fail-visible instead (accepted with allowlist choice).
- No historical metrics DB — recomputed from logs each load; revisit if logs get huge.
- No auth on the localhost server — single-user machine assumption.
