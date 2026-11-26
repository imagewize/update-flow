# update-flow

Notes on how a WordPress core update round actually goes for a one-person agency that maintains its own themes and plugins: what gets tested, where the time goes, what breaks, and what is worth automating.

Not a tool. A running record, one file per round, so the next round starts from numbers instead of memory.

## The flow

1. **Set up** a test copy on the release candidate (backup the DB first).
2. **Grep** each repo for the APIs the release changes (field guide and dev notes).
3. **Test per repo:** lint and unit tests, pattern validation, editor pass (blocks valid after save and reload), front end, admin, console and `debug.log`.
4. **Fix** what breaks, one branch per repo.
5. **Release:** changelog, version bump, `Tested up to`, PR with test results in the body, merge, tag after merge, publish.
6. **Deploy** the released packages to the demo and production sites, then verify in a real browser.
7. **Log** results and gaps (what was not tested), and re-run on the final RC.

## Tools used

- [WP Pattern Sentinel](https://github.com/imagewize/wp-pattern-sentinel): validates block patterns in a real editor.
- [pt-cli](https://github.com/imagewize/pt-cli): static block-markup check of theme patterns and templates, no WordPress install needed (`composer check`). Fast first pass before Sentinel.
- Playwright scripts for editor and front-end passes.
- Plugin Check, PHPCS, PHPUnit.
- Trellis/Bedrock for deploys.

## Rounds

| Round | Scope | Notes |
|---|---|---|
| [WP 7.2 (Nov 26, 2026)](rounds/2026-11-wp-7-2.md) | 10 repos, 4 plugins and 6 themes | ~9-10 h, mostly verification; 3 small bugs found |

## License

[MIT](LICENSE)
