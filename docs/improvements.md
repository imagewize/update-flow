# Possible improvements

Ideas for cutting the time of the next round. None of this is built yet unless marked. The 7.2 round spent most of its time proving nothing broke (see [rounds/2026-11-wp-7-2.md](../rounds/2026-11-wp-7-2.md)), so the list targets verification and release work first.

## Scale scrutiny to the risk of the update

The 7.2 round took about 7.5 hours of round work across 10 repos, and almost all of it went on proving nothing broke. Every repo got roughly the same full pass whether or not the release touched anything it uses. Client rounds are much faster: one site, no release pipeline, no pattern validation, no tagging. A 13-plugin round on a single shared-host shop took about 2 hours (see [shared-hosting.md](shared-hosting.md)). Most of the gap is not the checks themselves but the number of repos, and the release work on our own themes and plugins.

Proposal, not tried yet: tier the pass by what changed, and keep the full pass for the top tier.

| Tier | Example | Pass |
|---|---|---|
| Patch or minor update, changelog shows no API change, grep finds no hits in the repo | A bug-fix plugin release, a core point release | Light: read the changelog, `composer check` or `wp plugin list` clean, `debug.log` empty after a front-end and editor smoke test, one before/after screenshot set. No block-by-block editor pass. |
| Minor release that changes behaviour the repo uses (grep hits), or touches the editor, blocks or checkout | A core minor with block editor changes, a WooCommerce minor | Full pass for the affected repos only: editor pass, pattern validation, browser check. Skip the repos with no hits. |
| Major update, release candidate, or a plugin with its own data migrations | A core major, a WooCommerce major, a page-builder update | Full pass, its own batch, DB backup first, human gate before production. |

What would make this work:

- **Decide from the grep, not from the version number.** A patch can break something and a major can touch nothing you use, so the grep pass over each repo is what puts it in a tier. That ties into the per-release pattern file under Grep pass below.
- **Skip repos with zero hits** for the full editor pass, but still run the cheap static checks (`composer check`, pt-cli) everywhere, since they take seconds.
- **Use screenshot compare as the default light check** (`wp-ops update-snapshot` / `update-compare`) instead of hand-driven editor passes for plugin-only rounds. The one flagged page in a recent client round was a false positive, so mask rotating content up front.
- **Batch patch updates** together and only split out the ones in the top tier, as is already done for WooCommerce and page builders.
- **Log what a light pass skipped** in the round file, the same way the gaps are logged now, so a missed regression can be traced back to the tier decision.

Risk to watch: a lighter pass is a bet. Pattern drift and invalid blocks after a save and reload (the main 7.2 findings) only show in a real editor, so a patch that changes block markup or `save()` output belongs in the second tier even though it looks small.

## Verification

| Idea | Why | Notes |
|---|---|---|
| Run [WP Pattern Sentinel](https://github.com/imagewize/wp-pattern-sentinel) in parallel, per repo or per pattern batch | The Elayne full run (~20 min, 161 patterns) was the longest single wait | Needs several editor sessions or sites at once; check that logins and fixtures don't collide |
| Share the browser-check script across repos | Aviendha and Elayne Legal have `tests/browser-check.js`; the other repos repeat the pass by hand | Extract to one shared package or repo with per-repo config (URLs, blocks to insert, expected console state) |
| Run pt-cli in CI on every PR | It is static and takes seconds, so drift shows up before the round | Already `composer check` locally |
| Fix Sentinel login and setup problems once | Login trouble ate part of the setup time in 7.2 | Document a known-good setup, or script it |
| Build Sentinel fixtures from `block.json` defaults | Min's fixture used live text and failed | Generate fixtures instead of hand-writing them |

## Grep pass

- Keep the grep patterns per release (removed APIs, deprecated props) in a file, so the next round reuses and extends them instead of starting from the field guide.
- Run the same patterns across all repos with one command and collect the hits in one list. The AI-assisted pass then reviews hits instead of searching.

## Release and deploy

- Template the release steps (changelog, version bump, `Tested up to`, PR body with test results, tag after merge) into one command.
- Script the Composer pin bump in the main-site and demo repos, so a release leads to a ready-to-review PR.
- Script the post-deploy browser check for both sites.
- Guard against re-tagging: a pre-tag check that the tag doesn't exist (Packagist tags are immutable; see the Aviendha 1.18.13 mishap).

## Sharing the scripts

The scripts live inside separate repos. Moving the reusable ones (browser check, grep patterns, release helpers) into a shared location would let a new theme or plugin join a round without copying code. Open question: one public tools repo, or per-tool repos like pt-cli and Sentinel today.

## Measuring

Nothing logged time per activity in 7.2; the split is rebuilt from commits. Logging start and stop per activity (even a plain text file) would turn the next round's estimates into numbers.
