# ClaudeCockpit

A design spec for a local web dashboard that wraps Claude Code, written for one user on one machine.

## What it does

Nothing yet. There is no code in this repo. The spec in `docs/superpowers/specs/2026-07-26-claude-cockpit-design.md` (dated 2026-07-26) describes the plan:

- Buttons that launch Claude Code in headless mode (`claude -p`) for research runs, code sessions and notes-vault maintenance.
- A live view of running sessions, streamed to the browser over SSE.
- Usage metrics (tokens, cost, duration, tool calls) read from the session logs in `~/.claude/projects`.
- Per-button permission allowlists instead of skipping permissions.
- Run summaries and daily rollups written to an Obsidian vault.

## Status

Design only. The spec is marked approved. No server, UI or button definitions have been built.

## Stack

Planned: Bun and Hono for the local server, a browser UI, localhost only.

## License

All rights reserved.
