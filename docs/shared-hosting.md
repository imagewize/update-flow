# Shared-hosting client sites

How updates are tested and applied for client sites on plain shared hosting: classic WordPress, no Composer, often no SSH and no WP-CLI on the host. The Bedrock/Trellis flow is in [bedrock-composer.md](bedrock-composer.md); this is the other half. Client names, hosts and credentials are left out on purpose.

## The shape of a round

1. **Reconcile.** Compare what runs on the host with the local copy (versions of core, theme, every plugin). Do not trust the local copy.
2. **Local copy in Valet,** pinned to the host's PHP.
3. **Baseline screenshots,** apply updates in batches, compare after each.
4. **Human gate.** Look at the compare page and the functional checks, then decide per batch.
5. **Production:** full backup first, apply the same versions, purge caches, verify.

Production is not the tested artifact, since the update is repeated on the host. That is the main weakness compared with Trellis, where the tested commit is what ships and a bad update is a symlink flip back. There is no atomic rollback on shared hosting, so the backup is its own step.

## Debugging locally in Laravel Valet

The site is linked in [Laravel Valet](https://laravel.com/docs/valet) and served at `https://<site>.test`.

| Need | How |
|---|---|
| Match the host's PHP | `valet isolate php@8.x` in the site root. PHP is what has to match, sometimes MySQL (`SELECT VERSION();` on the host, write it down). |
| `wp` on the right PHP | The `wp` on PATH uses Homebrew's default PHP, not the isolated one. Call a pinned binary (a small shell function per PHP version). |
| PHP errors | `WP_DEBUG` and `WP_DEBUG_LOG` on, `WP_DEBUG_DISPLAY` off; read `wp-content/debug.log`. |
| Server errors (500, 502) | `valet log`, `valet log nginx`, `valet log php-fpm`. |
| Page cache hiding changes | Purge it (`wp rocket flush` or `wp cache flush`) after DB or content changes, and after a plugin round on production. |
| Outgoing mail | A local-only must-use plugin routes `wp_mail()` to MailHog (`localhost:1025`, UI on `8025`), so test orders and form submits cannot mail real people. Never deploy it to production. |
| Local URLs | Search-replace the production URL to the `.test` URL after the DB import. Pull the DB fresh before every round; the copy drifts as soon as the client edits content. |
| TLS | `valet secure <site>` if the cert warns. |

The web server differs (Nginx locally, Apache on the host) but that has not made an update fail. The exception is a plugin that writes its own `.htaccess` rules (page cache, WebP rewrites, security plugins): when one of those updates, check that feature on production.

Licences bound to the production domain refuse the `.test` domain (page cache and builder plugins are typical). Those can only be tested on production, behind the backup. Expect a vendor zip install locally for the ones that refuse an update.

## Checks

- Before/after screenshots of the key pages with `wp-ops update-snapshot` and `wp-ops update-compare` (mask the cookie banner and anything random, such as related-product rotation).
- Zero new fatals in `debug.log` after each batch.
- On shops: `wp-ops checkout-check --place-order` with MailHog to assert the confirmation email arrives, plus the contact form. Anything not automatable (a configurator product, for example) stays a manual gate.
- After production: repeat the screenshots against the local "after" set, purge caches, and rerun the version probe below.

## Working without SSH or WP-CLI: FTP and PHP

When the host gives FTP and wp-admin only, `wp plugin list` and `wp db export` are gone. Two small scripts cover the gap.

### An FTP wrapper

A curl wrapper that forces explicit FTP over TLS (`--ssl-reqd`, the same as the FileZilla GUI) and takes the password from FileZilla's Site Manager at runtime, so it never lands on a command line, in a file or in a log. The Site Manager stores the password base64-encoded; if FileZilla has a master password set, decoding fails and you re-enter the password in FileZilla instead of working around it.

```bash
#!/usr/bin/env bash
# ftp.sh: curl wrapper for one production FTP account. Any curl options, then an ftp:// URL.
set -euo pipefail
HOST="ftp.example.com"; FTP_USER="user@example.com"; SITE_NAME="example-site"
SITEMANAGER="$HOME/.config/filezilla/sitemanager.xml"

PASS=$(python3 - "$SITEMANAGER" "$SITE_NAME" <<'PY'
import base64, sys, xml.etree.ElementTree as ET
path, site = sys.argv[1], sys.argv[2]
for s in ET.parse(path).getroot().iter("Server"):
    if s.findtext("Name") == site:
        print(base64.b64decode(s.findtext("Pass") or "").decode()); break
else:
    sys.exit(1)
PY
) || { echo "could not read password" >&2; exit 1; }

exec curl --ssl-reqd \
  --netrc-file <(printf 'machine %s\nlogin %s\npassword %s\n' "$HOST" "$FTP_USER" "$PASS") "$@"
```

```bash
scripts/ftp.sh "ftp://ftp.example.com/public_html/"                      # list
scripts/ftp.sh -o wp-config.php "ftp://ftp.example.com/public_html/wp-config.php"   # download
scripts/ftp.sh -T local.php "ftp://ftp.example.com/public_html/wp-content/mu-plugins/"  # upload
```

Uploads change production and are hard to undo. Confirm first, and never bulk-upload `wp-content/` or overwrite files without a backup of what you replace. Plain `curl --user` also works but falls back to unencrypted FTP unless `--ssl-reqd` is set.

### A read-only version probe

Reconciling versions is the first step of a round and is the part `wp plugin list` normally does. The probe takes each plugin that exists locally, downloads only its entry file from production (a few KB), reads the `Version:` header, and prints one line per plugin:

`slug | local version | update available | production version | status`

Status is `OK`, `PROD-AHEAD`, `PROD-ABSENT` or `NO-ENTRY-FILE`. It does the same for core (`wp-includes/version.php`) and the active theme (`style.css`). It never writes to production. Plugins that exist only in the sandbox (the mail catcher, local work plugins) are excluded by a regex at the top.

What it found in practice: a count mismatch between production's update screen and the sandbox was fully explained by two sandbox-only plugins and one plugin whose licence kept it ahead on production. Without the probe that would have been a guess.

### Database and file backups

- **With SSH but no `exec()`** (common hardening): `wp db export` fails because it shells out to `mysqldump`. Call `mysqldump` over SSH directly and stream the dump to your machine; do not put the DB password on the command line (it shows in `ps` on a shared host).
- **FTP only:** upload a small PHP script that loads `wp-load.php`, dumps every table with `$wpdb`, and writes the SQL file; trigger it in the browser, download the file over FTP, then delete both from the server. It covers the database only; files come over FTP or from the host's own backup tool.
- **FTP is slow for many small files.** A full WordPress tree is thousands of files and FTP pays a round trip per file. Prefer the host's backup tool, a tarball, or parallel connections; check whether the limit is per connection before assuming it is bandwidth.
- **Download the backup off the host** and check sizes and archive listings before touching anything. A backup stored only on the server goes down with the server.

### Applying updates on production

Preferred: one-click updates in wp-admin (batch by batch, in the tested order), then purge caches. Fallback when a one-click refuses (licence or connectivity): download the exact version that passed local testing and upload it over FTP, deleting the old plugin directory first.

Rollback is a restore of the DB backup plus re-uploading the previous plugin directories. Take them from the pre-update file backup or the local copy before you start.

## Business angle

These rounds are on request, not on a schedule, billed by the hour, and not a subscription.
