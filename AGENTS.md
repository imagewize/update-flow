# AGENTS.md

Guidance for coding agents (Claude Code, Vibe and others) working in this repository.

## What this repo is

A Markdown-only notes repo, not a tool: no build, lint, or test commands. It records how each WordPress core update round goes for a one-person agency (imagewize) that maintains its own themes and plugins. The point is that the next round starts from numbers instead of memory.

- `README.md`: the general flow (setup → grep → test per repo → fix → release → deploy → log), how updates reach the Bedrock/Trellis sites via Composer, tools used, and the **Rounds** index table.
- `rounds/YYYY-MM-wp-X-Y.md`: one file per update round (e.g. `rounds/2026-11-wp-7-2.md`).

## Adding or editing a round

- New round: create `rounds/<year>-<month>-wp-<major>-<minor>.md` and add a row to the Rounds table in `README.md` (Round link with date, Scope, Notes). Keep the README row consistent with the round file's headline numbers; past commits fixed exactly this kind of mismatch (time framing, bug count).
- Follow the existing round structure: Repos covered, Where the time went (table with time and share), Timeline (from commit times), What the checklist found, What actually broke, Ideas to cut the time.
- Be honest about precision. Time splits are reconstructed from commit timestamps and test plans, so label them as estimates; separate wall-clock time from dedicated time; distinguish real regressions from pattern drift or pre-existing bugs.
- Source material lives outside this repo (commit history of the theme/plugin repos, `seo-strategy/docs/...` test plans, monthly logs). Don't invent numbers that aren't in those sources or the user's input.

## Context worth knowing

- Main site (imagewize.com) is a Bedrock install with Nynaeve, a hybrid Sage/Blade theme; the demo (demo.imagewize.com) is a separate Bedrock multisite with FSE block themes (Elayne and others). Hybrid and FSE themes need different checks (see README).
- Min is a private repo; don't link to it.
- Packagist tags are immutable: tag only after merge, never re-tag.

## Style

Plain, concise prose and tables. Short imperative commit messages (e.g. "Add pt-cli to tools", "Fix round title"). Work on a branch such as `docs/<topic>`, not directly on `main`.
