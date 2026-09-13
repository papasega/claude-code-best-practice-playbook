# Claude Code Plugins Config

Pre-declared, vetted plugin configuration for Claude Code, loaded automatically
in local, cloud (claude.ai/code), and mobile sessions via `.claude/settings.json`.

## What's included

Only plugins that were manually reviewed (no lifecycle hooks, no background
services, no network calls beyond fetching the plugin content itself):

- **andrej-karpathy-skills** (`forrestchang/andrej-karpathy-skills`) — a single
  `CLAUDE.md` encoding four engineering principles: think before coding,
  simplicity first, surgical changes, goal-driven execution.
- **document-skills** from the official Anthropic skills marketplace
  (`anthropics/skills`).

## Deliberately excluded

`claude-mem` (`thedotmack/claude-mem`) and `Superpowers` (`obra/superpowers`)
install lifecycle hooks and a background worker service. They are installed
manually, locally, one at a time, with an audit of their
`.claude-plugin/marketplace.json` and hook scripts before each install —
never auto-loaded from committed config.

## How to use this file

1. Copy `.claude/settings.json` into the root of any repository (or into
   `~/.claude/settings.json` for a user-wide default) where you want these
   plugins available.
2. Commit it. On the next Claude Code session (local, cloud, or mobile),
   the marketplaces and plugins listed are loaded automatically — no
   `/plugin` command needed.
3. To add more plugins later, extend `enabledPlugins` and
   `extraKnownMarketplaces` following the same pattern, after reviewing the
   plugin's marketplace manifest and hooks yourself.
