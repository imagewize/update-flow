# Possible improvements

Ideas for cutting the time of the next round. None of this is built yet unless marked. The 7.2 round spent most of its time proving nothing broke (see [rounds/2026-11-wp-7-2.md](../rounds/2026-11-wp-7-2.md)), so the list targets verification and release work first.

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
