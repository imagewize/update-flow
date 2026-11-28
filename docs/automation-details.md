# Automation details: the commands behind a round

What is actually run during a round, per step, taken from the WP 7.2 round (Nov 26, 2026): the shell history of the Claude Code sessions and the commit history of the ten repos. The README has the short version ([What is automated and what is manual](../README.md#what-is-automated-and-what-is-manual)); this file has the commands.

Read this as a log of what was done, not a one-command pipeline.

**Who did what.** Claude Code drove the round (Ixian was done with Mistral Vibe). It proposed and ran almost every command below: the DB backup, the core update to the RC, the local Sentinel login helper, the `wp-ops rsync-package-to-site` syncs, the grep loop, the Playwright scripts, PR bodies and release steps. The person approved each command, steered, and fed it the WordPress field guide and dev notes. Nothing runs unattended or on a schedule. Paths are the local layout (`~/code/<repo>`); the demo and main sites are Bedrock/Trellis installs run in a local Lima VM.

Legend for the Kind column: **script** = a tool or command that does the check or the work, run by Claude Code on approval; **AI** = Claude Code wrote the command, script or text and the person reviewed it; **by hand** = the person did it themselves (merging, tagging, reading results). `...` means a value left out for brevity (a URL, a PR title or body), `<name>` is a placeholder to fill in, and "Command (shortened)" means the real command had more flags or longer arguments.

## 1. Set up the RC copy

The rows below use a small shell helper, `wpvm`, that runs WP-CLI inside the demo's Trellis VM (core lives in `web/wp` on Bedrock):

```sh
cd ~/code/imagewize.com/trellis
wpvm() { trellis vm shell --workdir /srv/www/demo.imagewize.com/current -- wp "$@" --path=web/wp; }
```

So `wpvm core version` runs `wp core version --path=web/wp` in the demo's current release.

| Step | Command (shortened) | Kind |
|---|---|---|
| DB backup of the demo before touching core | `wpvm db export /tmp/demo-pre-7.2-rc3.sql` | script |
| Move core to the RC (Bedrock pins 7.1.5 in Composer, so a later `composer install` reverts it) | `wpvm core update --version=7.2-RC3 --force` | script |
| Run DB upgrade on the multisite | `wpvm core update-db --network` | script |
| Confirm version | `wpvm core version` | script |
| Create a local-only `admin` so Sentinel can log in (needed after every production DB pull) | `tools/scripts/sentinel-local-admin.sh demo.imagewize.com` | script (in the seo-strategy repo) |
| Sync the working copy of a package into the site copy without releasing it | `SITE_ROOT=~/code/imagewize.com/demo/web/app wp-ops rsync-package-to-site plugin aludra ~/code/aludra` (same for `theme <name>`; `--dry-run` first) | script (wp-ops CLI) |

The `admin` script sets a random password through stdin, stores it in `~/.config/wp-sentinel/<site>-local.pass` and sets `admin_email_lifespan` so WordPress's "confirm admin email" screen doesn't block the login. It only touches the local VM.

## 2. Grep pass

One loop over all ten repos, counting files that match each API the release touches. The pattern list comes from the field guide and dev notes, so it changes per release. Hits are then read by hand or by the assistant.

```sh
for r in aludra warder-cookie-consent waygate min elayne aviendha nynaeve winespring ixian elayne-legal; do
  echo "=== $r"
  for p in '__experimentalGroup' 'InspectorControls group=' 'wp_register_ability|wp-abilities' \
           'rest_pre_dispatch|rest_send_cors_headers|register_rest_route' \
           'add_menu_page|add_submenu_page|admin_bar_menu' 'add_editor_style|editor-style' \
           'data-wp-|@wordpress/interactivity' 'JQMIGRATE|jQuery\(|\$\(document\)'; do
    n=$(grep -rEl --include=*.php --include=*.js --include=*.jsx --include=*.json --include=*.css \
        --exclude-dir=node_modules --exclude-dir=vendor --exclude-dir=build --exclude-dir=.git -e "$p" ~/code/$r | wc -l)
    [ "$n" != 0 ] && echo "$n files: $p"
  done
done
```

Follow-up greps are one-offs: the `template` prop of `InnerBlocks` (`grep -rnE 'template[=:]|useInnerBlocksProps|<InnerBlocks'`, found about 24 Aludra blocks), and numeric `width`/`height` on `core/image` in templates.

Kind: **script** for the counting, **AI** for reading the hits against the dev notes.

## 3. Test per repo

| Check | Command | Kind |
|---|---|---|
| PHP lint | `composer lint` | script |
| Coding standards | `composer wpcs:scan` (Aludra: `vendor/bin/phpcs -d memory_limit=512M .`) | script |
| Unit tests | `composer test` (Aludra: 39 tests, plus a source test added in 7.2) | script |
| Static pattern and template check | `composer check`, which runs `pt-cli check patterns/ --theme=<name>`; templates: `pt-cli check:templates --theme=<name>` | script |
| Block validation in a real editor | `sentinel --trellis --trellis-dir=$HOME/code/imagewize.com/trellis --site=demo.imagewize.com --user=admin --pass="$(cat ~/.config/wp-sentinel/demo.imagewize.com-local.pass)" --log ~/code/elayne/patterns/` | script |
| Same, per repo | `npm run validate` (Aludra), `npm run test:patterns` (Min), `tests/sentinel/` (Nynaeve) | script |
| Plugin Check | `wp plugin activate plugin-check --url=...`, then run it on the plugin in the demo | script |
| Editor pass (insert every block, save, reload, check validity and Inspector) | Playwright script written per repo, e.g. Aludra inserted 28 of 34 block types | AI wrote it, script runs it |
| Front-end and admin pass | Playwright scripts, e.g. a Warder script for banner, accept/reject, modal, persistence after reload; one for the settings page save | AI wrote it, script runs it |
| Dedicated browser check | `tests/browser-check.js` in Aviendha and Elayne Legal (`npm run test:browser`; Playwright comes with the Sentinel dependency, `npm run setup` installs Chromium; needs the local-only admin from `sentinel-local-admin.sh`) | script |
| Site Editor and server-side registration | `wp` commands in the VM: count templates and parts, check the Site Editor canvas for invalid blocks | script |
| Look at the result | read console output and `debug.log`, look at the pages | by hand |

Sentinel facts from the round: Elayne's full run took about 20 minutes for 161 patterns (149 first run, 160 after fixes). It logs to `sentinel-<timestamp>.log.json` with `--log`. With `--trellis` it reads the vault only and ignores `.env`, so Elayne Legal's run used a scratch directory with a `.env` and no `--trellis` (documented in that repo's `AGENTS.md`). The Playwright scripts for the editor and front-end passes were mostly throwaway files in a scratch directory, not committed, except `tests/browser-check.js`, which does not add Playwright itself but reuses the copy Sentinel installs and is tied to the local demo (URL, subsite, home ID and page list default to Aviendha's, overridable by env vars).

## 4. Fix

By hand or with the assistant, one branch per repo, named `chore/wp-7-2-compat`. Each fix gets a commit, and bug fixes that can be tested get a test (Aludra's `tests/php/BlockSourceTest.php`).

## 5. Release

| Step | Command | Kind |
|---|---|---|
| Branch, changelog, version bump, `Tested up to` | edit `CHANGELOG.md`, `readme.txt`, `style.css` / plugin header, `package.json` | by hand or AI |
| PR with test results in the body | `gh pr create --base main --title ... --body ...` | AI drafts, person reviews |
| Merge | `gh pr merge <n> --merge` | by hand |
| Tag after merge, never re-tag | `git fetch --tags`, `git tag --sort=-v:refname \| head -3`, then tag and `gh release create vX.Y.Z --verify-tag` | by hand |
| Theme zip | GitHub Action `create-release.yml` runs on a published release | script |
| wp.org plugins (Aludra, Warder) | `svn-wporg-deploy --dry-run`, then `--yes` (Aludra); `bin/prepare-svn.sh` then `svn ci` (Warder) | script |
| Packagist | new tag becomes a Composer version for the pin bump (tags are immutable) | not checked in this round's history |

## 6. Deploy

| Step | Command | Kind |
|---|---|---|
| Bump the pin in the main-site or demo repo | `composer update imagewize/<package> --no-interaction` in `site/` or `demo/` | script |
| Commit the lock file with versions in the message | `git commit -qam "Update Nynaeve to 4.8.0" && git push` | by hand |
| Deploy | `cd trellis && trellis deploy production imagewize.com` (check `PLAY RECAP` for `failed=0`) | script |
| Verify in a real browser | Playwright script or by hand; demo subsites are rebuilt and checked one by one | script plus by hand |

## 7. Log

The test plan in `seo-strategy/docs/wp-7-2-compatibility-test-plan-2026-11.md` is edited throughout the day (a small Python script replaces text per repo as each check finishes), and the day's entry goes into the monthly log. Round notes here are written from those two.

## Other tools and scripts the assistant used

Besides Sentinel, pt-cli and wp-ops, the round ran on plain command-line tools and short scripts the assistant wrote:

| Tool | Used for |
|---|---|
| `grep`, `sed` | The grep pass; bulk edits such as version and `Tested up to` lines, and one-line code fixes (e.g. numeric image `width`/`height` in Nynaeve Feature Cards) |
| `python3` heredocs | Text edits that need exact matches: updating the test plan after each check, rewriting PR bodies, changelog entries |
| `node` Playwright scripts | Throwaway editor and front-end passes (Aludra block insert, Warder front end and admin, Nynaeve front end, a padding check); written into a scratch directory, not committed |
| `wp` (WP-CLI) in the VM, `trellis`, `limactl` | Core and plugin state, template and part counts, deploys, VM access |
| `composer`, `npm`, `phpcs` | Pin bumps, builds, lint and coding standards |
| `git`, `gh` | Branches, PRs, PR body edits, releases |
| `svn`, `svn-wporg-deploy`, `bin/prepare-svn.sh` | wp.org plugin releases |
| `curl` | Checking the WordPress beta channel and HTTP status of pages |
| Claude in Chrome (browser MCP) | A few looks at pages in a real browser |

## Not automated yet

- Running Sentinel in parallel; the 7.2 round ran it as one long job per repo.
- Sharing the Playwright scripts across repos.
- A single command for RC setup, release or deploy.
- Before/after screenshots of key pages. `wp-ops update-snapshot` and `update-compare` exist for this and are planned for the production move to 7.2, not used in the 7.2 round.

See [improvements.md](improvements.md) for the plan.
