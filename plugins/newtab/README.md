# Firefox Newtab Plugin

Skills and tools for Firefox newtab development.

## Skills

### `nova-cleanup-comments`

Adds `@nova-cleanup` comments to Firefox newtab code for Project Nova parallel implementation tracking.

Use this skill when adding cleanup comments to Nova-related changes. See [`skills/nova-cleanup-comments/SKILL.md`](skills/nova-cleanup-comments/SKILL.md) for details.

### `hnt-backend-investigation`

Diagnostic workflow for Home New Tab **backend** incidents, feature by feature: content recommendations (Merino, the curated corpus, admin-api, the article crawler, the ML section pipeline, the New Tab data pipelines), Picture of the Day, the daily crossword, and other New Tab features served the same way. Its output is an evidence-backed `FINDINGS.md` carrying a root cause and quantified impact.

Use this skill when a Sentry alert fires, an editor reports something broken, or a New Tab feature looks empty, wrong, or stale. See [`skills/hnt-backend-investigation/SKILL.md`](skills/hnt-backend-investigation/SKILL.md) for details.

**Optional tools.** Anything unreachable is raised as a task and reported under "Could not measure":

- `gh`
- `gcloud` and `bq`
- the `aws` CLI, with a read-only SSO profile
- a MySQL client, plus Mozilla VPN
- `uv` or `python3`
- the Sentry MCP server
- the Slack MCP server, for `#hnt-dev-be-alerts`
- Zyte API keys, for reproducing an extraction or reading the vendor's stats
