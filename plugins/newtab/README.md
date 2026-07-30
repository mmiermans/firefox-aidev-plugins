# Firefox Newtab Plugin

Skills and tools for Firefox newtab development.

## Skills

### `nova-cleanup-comments`

Adds `@nova-cleanup` comments to Firefox newtab code for Project Nova parallel implementation tracking.

Use this skill when adding cleanup comments to Nova-related changes. See [`skills/nova-cleanup-comments/SKILL.md`](skills/nova-cleanup-comments/SKILL.md) for details.

### `hnt-backend-investigation`

Diagnostic workflow for Home New Tab **backend** incidents, covering Merino, the curated corpus, admin-api, the article crawler, and the ML section pipeline. Its output is an evidence-backed `FINDINGS.md` carrying a root cause, quantified impact, and the check that would have caught the problem earlier.

Use this skill when a Sentry alert fires, an editor reports something broken, or recommendations look empty, wrong, or stale. See [`skills/hnt-backend-investigation/SKILL.md`](skills/hnt-backend-investigation/SKILL.md) for details.

**Optional tools:**

- `gh`
- `gcloud` and `bq`
- the `aws` CLI, with a read-only SSO profile
- a MySQL client, plus Mozilla VPN
- `uv` or `python3`
- the Sentry MCP server

Anything unreachable is raised as a task and reported under "Could not measure".
