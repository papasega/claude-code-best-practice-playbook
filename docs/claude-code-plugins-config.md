# Claude Code Plugins Config

Pre-declared plugin configuration for Claude Code, offered to local, cloud
(claude.ai/code), and mobile sessions via `.claude/settings.json`.

## What's included

Plugins reviewed at the date noted below — no lifecycle hooks, no background
services, no network calls beyond fetching the plugin content itself:

- **andrej-karpathy-skills** — a single `CLAUDE.md` encoding four engineering
  principles: think before coding, simplicity first, surgical changes,
  goal-driven execution.
- **document-skills** from Anthropic's skills marketplace (`anthropics/skills`).

> **Review date: 2026-09-13.** "Reviewed" describes the content as it stood on
> that date, at an unpinned ref. Both sources can change under you:
>
> - `forrestchang/andrej-karpathy-skills` now **redirects to
>   `multica-ai/andrej-karpathy-skills`** — the repository changed hands since this
>   config was written. A redirect means the content you install is no longer the
>   content that was reviewed.
> - Anthropic presents `anthropics/skills` as **example** skills to evaluate before
>   relying on them, not a production-supported suite.
>
> Neither entry is pinned to a tag or commit SHA, so this file cannot claim to be
> "vetted" indefinitely. Re-audit before trusting it in a new project, or pin the
> sources and record the SHA here.

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
2. Commit it. On the next Claude Code session (local, cloud, or mobile), the
   marketplaces and plugins listed are **offered** rather than silently enabled:
   after you trust the project, Claude Code asks for consent before adding a
   marketplace or installing a plugin from project config, and you can decline
   either. You will not need to type `/plugin` yourself, but you will be asked.
3. To add more plugins later, extend `enabledPlugins` and
   `extraKnownMarketplaces` following the same pattern, after reviewing the
   plugin's marketplace manifest and hooks yourself.
