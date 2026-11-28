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

The round is a human-run checklist with scripted and AI-assisted steps. There is no scheduler and no unattended pipeline: a person starts each round, reads the results and decides what ships.

| Step | How it is done |
|---|---|
| Set up RC copy, DB backup | Manual, with WP-CLI and Composer commands |
| Grep for changed APIs | AI-assisted: Claude Code or Vibe searches each repo against the field guide and dev notes; a person reviews the hits |
| Static pattern check | Scripted: [pt-cli](https://github.com/imagewize/pt-cli) via `composer check`, seconds per repo |
| Pattern validation in the editor | Scripted: [WP Pattern Sentinel](https://github.com/imagewize/wp-pattern-sentinel) opens each pattern in a real block editor and flags invalid blocks (Elayne: ~20 min for 161 patterns) |
| Editor and front-end passes | Part scripted: Playwright scripts (e.g. `tests/browser-check.js` in Aviendha and Elayne Legal), the rest by hand in a browser |
| Lint and unit tests | Scripted: PHPCS, PHPUnit, Plugin Check through `composer lint` / `composer check` |
| Fixes | Written by hand or with an AI assistant, reviewed in a PR per repo |
| Release | Manual: changelog, bump, PR, tag after merge. A candidate for templating (see the round notes) |
| Deploy | Composer pin bump, then `trellis deploy`; a person triggers it and checks the result in a browser |

The scripts are run per round, not on a schedule. Rounds follow WordPress core releases, not client requests.

## Bedrock and Composer

Both sites are [Bedrock](https://roots.io/bedrock/) installs deployed with [Trellis](https://roots.io/trellis/), so core, themes and plugins are Composer dependencies, not wp-admin updates. That changes how the RC is installed (`wp core update`, which a `composer install` resets), how packages reach production (release, bump the pin, `trellis deploy`) and which checks apply to hybrid versus FSE themes. Details in [docs/bedrock-composer.md](docs/bedrock-composer.md).

## Tools used

- [WP Pattern Sentinel](https://github.com/imagewize/wp-pattern-sentinel): validates block patterns in a real editor.
- [pt-cli](https://github.com/imagewize/pt-cli): static block-markup check of theme patterns and templates, no WordPress install needed (`composer check`). Fast first pass before Sentinel.
- Playwright scripts for editor and front-end passes (per-repo `tests/browser-check.js` where it exists).
- [Claude Code](https://claude.com/claude-code) and Vibe: AI assistants for the grep pass, reading release notes against code, and parallel checks on a repo. They propose; tests and a person decide.
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
