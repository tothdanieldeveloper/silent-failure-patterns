# Silent failure patterns

Failures that don't crash or alert. Notes from running self-hosted, multi-tenant systems.

## Test guard misses a second DB connection

Tests redirect the main database to a test DB, but a second connection (admin DB, replica)
is not redirected, so tests write to production.

Fix: assert at runtime that every engine points to a test database.

```python
for engine in all_engines():
    assert engine.url.database.endswith("_test")
```

## Tests depend on the production environment

Tests run in the production image and pick up its feature flags. When a flag changes,
tests fail, or keep passing while testing the wrong code path.

Fix: each test sets the mode it needs.

## Backup step lost in a migration

The user-file backup was a step on the old server, not part of the app. After the move,
only the database gets backed up.

Fix: after a migration, list the files on the backup target. Make sure the backup finishes
before the offsite copy runs.

## Certificate renewed, old one still served

Certbot renews the cert on disk, nginx keeps serving the old one until it reloads.

Check:

```bash
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | openssl x509 -noout -enddate
openssl x509 -in /etc/letsencrypt/live/example.com/cert.pem -noout -enddate
```

Fix: deploy hook `nginx -t && systemctl reload nginx`, tested with
`certbot renew --dry-run --run-deploy-hooks`.

## fail2ban reading a dead log file

fail2ban resolves glob log paths only when the jail loads. A recreated container gets a new
log path, the jail keeps reading the old file and finds nothing.

Fix: a cron job compares the file list from `fail2ban-client status <jail>` with
`docker inspect -f '{{.LogPath}}'` and reloads the jail when they differ.

## Cache prune that frees nothing

An age filter based on last use never removes layers that are used on every build.
The job succeeds and frees 0 bytes.

Fix: log the freed size on every run. Test flags on throwaway data before relying on them.

## Empty input wipes a crontab

`crontab -l | sed ... | crontab -` installs an empty crontab if sed fails, for example
when the delimiter also appears in the pattern.

Fix:

```bash
crontab -l > /tmp/cron.bak
crontab -l | sed 's#old#new#' > /tmp/cron.new
test -s /tmp/cron.new \
  && [ "$(wc -l < /tmp/cron.new)" -eq "$(wc -l < /tmp/cron.bak)" ] \
  && crontab /tmp/cron.new
```

## Fallback to the first item

In a multi-tenant app, a missing setting falls back to the first page or account the user
can access, which may belong to another customer. Often paired with marking an action as
done when nothing was sent.

Fix: no target means skip with an error. Store the target and the external ID with each
action.

## Verify command that writes

A verify or sync command saves its result. Run at the wrong time, it downgrades a working
state.

Fix: add a read-only mode for checks and monitoring.

## Security check that fails open

When the store behind a CSRF or rate-limit check returns an error, the code skips the check.

Fix: reject the request if the check can't run. Test the error path.
