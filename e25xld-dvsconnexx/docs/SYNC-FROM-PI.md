# Safe Production Source Sync Runbook

Purpose: copy the live DVSConnexx source from the DVSwitch Pi into GitHub without publishing secrets or stale backup files.

## Source of truth

Live application root:

`/opt/dvsquick/`

Expected important files may include:
- `app.py`
- `audio_relay.py`
- `activity_monitor.py`
- `static/index.html`
- `static/app.js`
- `static/style.css`

Always inspect the live tree first. Do not assume this list is complete or current.

## Do not copy blindly

Exclude or inspect carefully:
- backup files (`*.backup-*`)
- credentials
- PIN/config secrets
- generated runtime state
- browser/session tokens
- logs
- caches
- temporary test files
- mock fixtures used only during development
- files containing Cloudflare or DVSwitch credentials

## Recommended workflow for Claude/AI agent on the Pi

1. Read `CLAUDE.md` and `SECURITY.md` from GitHub first.
2. Run a read-only tree/listing of `/opt/dvsquick/`.
3. Identify the production files actually imported/served by `dvsquick.service`.
4. Search candidate files for secret-like strings before staging:
   - token
   - password
   - secret
   - PIN
   - bearer
   - api key
   - private key markers
5. Create sanitized copies in a staging directory, never modify production solely for GitHub export.
6. Remove machine-specific secrets while preserving safe operational defaults.
7. Add an `.example` suffix for config templates where appropriate.
8. Diff staged copies against production and explicitly report every redaction.
9. Only then commit/push to the GitHub project path.

## Suggested repository destination

```text
e25xld-dvsconnexx/
  src/
    dvsquick/
      app.py
      audio_relay.py
      activity_monitor.py
      static/
        index.html
        app.js
        style.css
```

## Documentation expected with each sync

Record:
- production timestamp
- relevant application version/commit if one exists
- files synchronized
- files intentionally excluded
- secrets/redactions performed
- syntax/tests executed
- whether production itself was changed (normally: no)

## Important

A GitHub sync is not authorization to deploy GitHub back onto production.

Deployment and source archival are separate operations.
