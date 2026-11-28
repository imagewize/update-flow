# Update Flow

Notes on how a WordPress core update round actually goes for a one-person agency that maintains its own themes and plugins on [Roots](https://roots.io/) [Trellis](https://roots.io/trellis/) and [Bedrock](https://roots.io/bedrock/) sites: what gets tested, where the time goes, what breaks, and what is worth automating.

The flow assumes a Bedrock/Trellis setup, where core, themes and plugins are Composer dependencies and deploys run through Trellis. On a classic wp-admin-updated install, the RC setup and deploy steps differ; the testing and logging parts carry over.

Not a tool. A running record, one file per round, so the next round starts from numbers instead of memory.

## The flow

1. **Set up** a local test copy on the release candidate (backup the DB first; on Bedrock, core is moved with `wp core update --version=<RC>`, not Beta Tester, and a `composer install` resets it; see [docs/bedrock-composer.md](docs/bedrock-composer.md)).
2. **Grep** each repo for the APIs the release changes (field guide and dev notes).
3. **Test per repo:** lint and unit tests, pattern validation, editor pass (blocks valid after save and reload), front end, admin, console and `debug.log`.
4. **Fix** what breaks, one branch per repo.
5. **Release:** changelog, version bump, `Tested up to`, PR with test results in the body, merge, tag after merge, publish.
6. **Deploy:** bump the Composer pins in the main-site and demo repos, run `trellis deploy`, then verify in a real browser.
7. **Log** results and gaps (what was not tested), and re-run on the final RC.

## What is automated and what is manual

The 7.2 round was driven by an AI coding assistant, mostly [Claude Code](https://claude.com/claude-code), with Mistral Vibe on Ixian. The assistant proposed and ran the commands: DB backups, moving core to the RC, the local Sentinel login, syncing packages to the test sites with wp-ops, the grep pass, the Playwright scripts, PR bodies and release steps. The person approved each command, steered, and supplied the WordPress field guide and dev notes. The checks themselves are our own tools and standard ones (WP Pattern Sentinel, pt-cli, wp-ops, Playwright, PHPCS, PHPUnit). There is no scheduler and no unattended pipeline.

| Step | Who does what |
|---|---|
| Set up RC copy, DB backup | Claude Code proposes and runs the WP-CLI commands in the Trellis VM; the person approves |
| Local test login for Sentinel | Claude Code proposed a helper script that creates a local-only `admin` user after each production DB pull |
| Sync packages to the test sites | Claude Code runs `wp-ops rsync-package-to-site` (wp-ops is our own CLI) |
| Grep for changed APIs | Claude Code searches each repo against the field guide and dev notes the person supplies; the person reviews the hits |
| Static pattern check | [pt-cli](https://github.com/imagewize/pt-cli) via `composer check`, seconds per repo |
| Pattern validation in the editor | [WP Pattern Sentinel](https://github.com/imagewize/wp-pattern-sentinel) opens each pattern in a real block editor and flags invalid blocks (Elayne: ~20 min for 161 patterns), run by Claude Code |
| Editor and front-end passes | Claude Code writes and runs Playwright scripts per repo (mostly throwaway; `tests/browser-check.js` in Aviendha and Elayne Legal is kept); the person looks at the results |
| Lint and unit tests | PHPCS, PHPUnit, Plugin Check through `composer lint` / `composer check`, run by Claude Code |
| Fixes | Claude Code proposes, the person reviews, one PR per repo |
| Release | Claude Code drafts changelog, PR body and release steps; the person approves, merges and tags after merge. A candidate for templating (see the round notes) |
| Deploy | Composer pin bump, then `trellis deploy`, run by Claude Code on approval; the result is checked in a browser |

Rounds follow WordPress core releases, not client requests, and the person starts each one. The exact commands, step by step, are in [docs/automation-details.md](docs/automation-details.md). Ideas for automating more (parallel Sentinel, shared scripts, release templating) are in [docs/improvements.md](docs/improvements.md).

## Bedrock and Composer

Both sites are [Bedrock](https://roots.io/bedrock/) installs deployed with [Trellis](https://roots.io/trellis/), so core, themes and plugins are Composer dependencies, not wp-admin updates. That changes how the RC is installed (`wp core update`, which a `composer install` resets), how packages reach production (release, bump the pin, `trellis deploy`) and which checks apply to hybrid versus FSE themes. Details in [docs/bedrock-composer.md](docs/bedrock-composer.md).

## Tools used

- [WP Pattern Sentinel](https://github.com/imagewize/wp-pattern-sentinel): validates block patterns in a real editor.
- [pt-cli](https://github.com/imagewize/pt-cli): static block-markup check of theme patterns and templates, no WordPress install needed (`composer check`). Fast first pass before Sentinel.
- Playwright scripts for editor and front-end passes (per-repo `tests/browser-check.js` where it exists).
- [Claude Code](https://claude.com/claude-code) and Mistral Vibe: AI coding assistants that drive the round (commands, scripts, grep, PR drafts). A person approves and steers; Claude Code did the 7.2 round except Ixian, which was done with Vibe.
- [wp-ops](https://github.com/imagewize/wp-ops): our own CLI and MCP server for WordPress operations (DB backup and pull, package sync, audits).
- Plugin Check, PHPCS, PHPUnit.
- [Composer](https://getcomposer.org/): core, themes and plugins are dependencies on Bedrock; pins and lockfiles decide what ships, and `composer lint` / `composer check` scripts run the checks in each repo.
- [Packagist](https://packagist.org/): where released versions of our own themes and plugins are published (tags are immutable).
- [WP Packages](https://wp-packages.org/): Composer repository for third-party WordPress.org plugins and themes (`wp-plugin/*`, `wp-theme/*`).
- [Trellis](https://roots.io/trellis/) and [Bedrock](https://roots.io/bedrock/) (both from Roots): server provisioning and deploys, and the Composer-based WordPress project structure.

## Rounds

| Round | Scope | Notes |
|---|---|---|
| [WP 7.2 (Nov 26, 2026)](rounds/2026-11-wp-7-2.md) | 10 repos, 4 plugins and 6 themes | ~11.5 h wall-clock, ~7.5 h on the round (minus lunch, dinner, cambridge-practice, social), mostly verification; 3 code bugs found |

## License

[MIT](LICENSE)
