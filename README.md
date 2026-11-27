# update-flow

Notes on how a WordPress core update round actually goes for a one-person agency that maintains its own themes and plugins: what gets tested, where the time goes, what breaks, and what is worth automating.

Not a tool. A running record, one file per round, so the next round starts from numbers instead of memory.

## The flow

1. **Set up** a local test copy on the release candidate (backup the DB first; on Bedrock, core is moved with `wp core update --version=<RC>`, not Beta Tester, and a `composer install` resets it; see below).
2. **Grep** each repo for the APIs the release changes (field guide and dev notes).
3. **Test per repo:** lint and unit tests, pattern validation, editor pass (blocks valid after save and reload), front end, admin, console and `debug.log`.
4. **Fix** what breaks, one branch per repo.
5. **Release:** changelog, version bump, `Tested up to`, PR with test results in the body, merge, tag after merge, publish.
6. **Deploy:** bump the Composer pins in the main-site and demo repos, run `trellis deploy`, then verify in a real browser.
7. **Log** results and gaps (what was not tested), and re-run on the final RC.

## Bedrock and Composer: how updates reach the sites

Both sites are Bedrock installs deployed with Trellis, so WordPress core, themes and plugins are Composer dependencies, not wp-admin updates. Packages come from three places:

- **Own themes and plugins** (`imagewize/*`): released on [Packagist](https://packagist.org/). A few that are not on Packagist, such as private ones, are pulled from their GitHub repo through a `vcs` repository entry.
- **Third-party WordPress.org plugins and themes** (`wp-plugin/*`, `wp-theme/*`, e.g. WooCommerce, Contact Form 7, The SEO Framework): installed from [WP Packages](https://wp-packages.org/) via the `https://repo.wp-packages.org` repository entry. These move on their own release schedule, so check their `Tested up to` against the RC too.
- **Premium plugins** such as ACF: from the vendor's own Composer repository.

| Site | Install | Theme type | Packages pinned in Composer |
|---|---|---|---|
| Main site (imagewize.com) | Bedrock, `site/composer.json` | [Nynaeve](https://github.com/imagewize/nynaeve): hybrid (Sage/Blade templates plus block editor, custom blocks) | Nynaeve, Min (private), [Warder](https://github.com/imagewize/warder-cookie-consent) |
| Demo (demo.imagewize.com) | Separate Bedrock multisite, `demo/composer.json` | FSE block themes: [Elayne](https://github.com/imagewize/elayne) (all subsites), [Elayne Legal](https://github.com/imagewize/elayne-legal), [Aviendha](https://github.com/imagewize/aviendha), [Ixian](https://github.com/imagewize/ixian), [Winespring](https://github.com/imagewize/winespring) | those themes, plus [Aludra](https://github.com/imagewize/aludra), Warder, [Waygate](https://github.com/imagewize/waygate) |

What that changes in a round:

- **Core is pinned, so the RC is a local-only override.** Beta Tester can't upgrade a Bedrock install, because core is the Composer package `roots/wordpress`, pinned in `composer.json` and the lock (7.1.5 in the 7.2 round). To test an RC:
  1. Work on the local dev copies (Trellis/Lima VM), never production.
  2. Back up the DB inside the VM (e.g. `wp db export /tmp/<site>-pre-7.2-rc3.sql`).
  3. Run `wp core update --version=<RC>` (e.g. `7.2-RC3`) on each site. This replaces the core files on disk and runs the DB upgrade; `composer.json` and the lock are untouched.
  4. Sync the working copy of each theme or plugin into the site with `wp-ops rsync-package-to-site` (no release needed to test), then run the checks.
- **`composer install` resets core to the pinned version.** Composer reinstalls `roots/wordpress` at the locked version (7.1.5), so any `composer install` or `composer update` on that copy silently puts core back on the old release, while the DB stays on the RC schema. Check `wp core version` after every install and re-run the `wp core update` if needed. Production never takes the RC: it stays on the pinned release until release day, when the pin is raised and deployed.
- **A package only reaches production through three steps:** release (tag after merge) -> bump the pin in the site's `composer.json` and lock -> `trellis deploy`. This applies to own packages; WP Packages versions just follow WordPress.org releases. Tag only after merge: Packagist treats published versions as immutable, so a moved tag can't be fixed, only superseded by a new version.
- **Production stays on the old core** until the release day; the new `Tested up to` ships first, so nothing shows a compatibility warning.
- **Hybrid vs FSE needs different checks.** FSE themes (Elayne and the rest) are mostly patterns, templates and `theme.json`, so pattern validation (pt-cli, Sentinel) and Site Editor passes carry the test. Nynaeve has Blade templates, so pt-cli's template check doesn't apply; the test is custom blocks in the editor, front-end pages and the build (`npm run build`).
- **FSE patterns are rebuilt on the demo after a deploy.** The demo subsites are rebuilt from the new Elayne patterns, then browser-checked.

## Tools used

- [WP Pattern Sentinel](https://github.com/imagewize/wp-pattern-sentinel): validates block patterns in a real editor.
- [pt-cli](https://github.com/imagewize/pt-cli): static block-markup check of theme patterns and templates, no WordPress install needed (`composer check`). Fast first pass before Sentinel.
- Playwright scripts for editor and front-end passes.
- Plugin Check, PHPCS, PHPUnit.
- [Composer](https://getcomposer.org/): core, themes and plugins are dependencies on Bedrock; pins and lockfiles decide what ships, and `composer lint` / `composer check` scripts run the checks in each repo.
- [Packagist](https://packagist.org/): where released versions of our own themes and plugins are published (tags are immutable).
- [WP Packages](https://wp-packages.org/): Composer repository for third-party WordPress.org plugins and themes (`wp-plugin/*`, `wp-theme/*`).
- Trellis/Bedrock for deploys.

## Rounds

| Round | Scope | Notes |
|---|---|---|
| [WP 7.2 (Nov 26, 2026)](rounds/2026-11-wp-7-2.md) | 10 repos, 4 plugins and 6 themes | ~11.5 h wall-clock, ~7.5 h on the round (minus lunch, dinner, cambridge-practice, social), mostly verification; 3 code bugs found |

## License

[MIT](LICENSE)
