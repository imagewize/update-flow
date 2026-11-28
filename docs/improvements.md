# Possible improvements

Ideas for cutting the time of the next round. None of this is built yet unless marked. The 7.2 round spent most of its time proving nothing broke (see [rounds/2026-11-wp-7-2.md](../rounds/2026-11-wp-7-2.md)), so the list targets verification and release work first.

## Scale scrutiny to the risk of the update

The 7.2 round took about 7.5 hours of round work across 10 repos, and almost all of it went on proving nothing broke. Every repo got roughly the same full pass whether or not the release touched anything it uses. Client rounds are much faster: one site, no release pipeline, no pattern validation, no tagging. A 13-plugin round on a single shared-host shop took about 2 hours (user estimate, not logged; the flow is in [shared-hosting.md](shared-hosting.md)). Most of the gap is not the checks themselves but the number of repos, and the release work on our own themes and plugins.

Proposal, not tried yet: tier the pass per repo by what changed, and keep the full pass for the repos that need it.

The tier is set in two steps. The release sets the ceiling, that is, how heavy the pass can get and what guards apply. The grep over each repo then decides which repos actually get that pass. A core RC therefore no longer means a full pass for all ten repos, only for the ones with hits.

| Tier | When (per repo) | Pass |
|---|---|---|
| Light | Grep finds no hits in the repo, and the changelog shows no change to block markup, `save()` output, the editor or checkout | Read the changelog, `composer check` or `wp plugin list` clean, no new `debug.log` entries after a front-end and editor smoke test, one before/after screenshot set. No block-by-block editor pass. |
| Full | Grep hits in the repo, or the release touches block markup, `save()` output, the editor, blocks or checkout | Editor pass, pattern validation, browser check for this repo. |
| Full and guarded | A Full repo on a release-candidate or major ceiling, or any plugin with its own data migrations (WooCommerce major, page builder) | Full pass, its own batch, DB backup first, human gate before production. |

What would make this work:

- **Decide from the grep, not from the version number.** A patch can break something and a major can touch nothing you use, so the grep pass over each repo is what puts it in a tier. The version only decides whether a Full repo also gets the guards. A core point release is not automatically light: point releases sometimes patch block `save()` output, which puts the affected repos in Full. The grep relies on the per-release pattern file under Grep pass below.
- **Give zero-hit repos a bump-only release.** In 7.2 the release pipeline (~1.5 h) and deploys (~1 h) were about a third of the round, and tiering the checks alone doesn't touch that. A Light repo only needs a `Tested up to` bump and a patch release. Batch those bumps into one PR per repo, run in one go, and skip the per-repo PR write-ups. A Light theme or plugin that the sites don't need yet can wait for its next real release before the Composer pin is bumped.
- **Backtest on 7.2 before trusting it.** Take the grep hits per repo from the 7.2 round, sort the repos into tiers, and check whether a Light pass would have missed a real finding. The Elayne pattern drift came from the editor rewriting saved markup, not from a removed API, so a grep may not have flagged it. If the backtest shows a miss, add that signal (for example "theme ships patterns", for a core release that changes the editor) to the Full criteria.
- **Skip repos with zero hits** for the full editor pass, but still run the cheap static checks (`composer check`, pt-cli) everywhere, since they take seconds.
- **Use screenshot compare as the default light check** (`wp-ops update-snapshot` / `update-compare`) instead of hand-driven editor passes for plugin-only rounds. In a recent client round the one flagged page was a false positive (not logged in this repo), so mask rotating content up front.
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

## Estimated saving

A rough guess at what all of the above would save on the next comparable round, mapped onto the 7.2 split. Nothing here is measured: the 7.2 split is itself rebuilt from commits, and none of the fixes has been tried.

| 7.2 activity | 7.2 time | Fixes that apply | Rough saving |
|---|---|---|---|
| Browser and editor passes | ~2.25 h | Light pass for zero-hit repos, shared browser-check script | 0.75–1.5 h |
| Release pipeline | ~1.5 h | One release command, bump-only releases for Light repos, re-tag guard | 0.5–1 h |
| Pattern validation | ~1.25 h | Sentinel in parallel, generated fixtures, pt-cli in CI so drift is fixed before the round | 0.25–0.5 h |
| Deploys and checks | ~1 h | Scripted pin bump and post-deploy check, fewer pin bumps when Light repos wait | 0.25–0.5 h |
| Setup and grep | ~0.75 h | Sentinel login fixed once, stored grep patterns run with one command | 0.25–0.5 h |
| Bug fixes | ~0.5–1 h | None; this is the real work | 0 |
| **Total** | **~7.5 h** | | **~2–4 h**, so a round of ~3.5–5.5 h |

Caveats:

- **The range depends on how many repos have no grep hits.** With 6 of 10 repos Light, expect the top of the range; with 2 or 3, closer to 2 h. The 7.2 backtest above answers this.
- **Some of the saving is waiting time, not your own time.** Running Sentinel in parallel shortens how long the round takes overall more than your hands-on hours.
- **Building the tools costs time,** which this table doesn't include. The break-even point in rounds is unknown. The bump-only path and the grep pattern file are probably the cheapest to set up for what they save.
- **The Light pass is a bet.** A missed regression in production costs more than it saved.

## Measuring

Nothing logged time per activity in 7.2; the split is rebuilt from commits. Logging start and stop per activity (even a plain text file) would turn the next round's estimates into numbers. Log time per repo and per tier too, since that is the only way to tell whether the Light pass actually saves time (see Scale scrutiny above).
