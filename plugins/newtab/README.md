# Firefox Newtab Plugin

Skills and tools for Firefox newtab development.

## Skills

### `nova-cleanup-comments`

Adds `@nova-cleanup` comments to Firefox newtab code for Project Nova parallel implementation tracking.

Use this skill when adding cleanup comments to Nova-related changes. See [`skills/nova-cleanup-comments/SKILL.md`](skills/nova-cleanup-comments/SKILL.md) for details.

### `hnt-backend-investigation`

Diagnoses Home New Tab **backend** errors, outages, and data-quality defects — Merino, the curated corpus, admin-api, the article crawler, and the ML section pipeline.

Use this skill when a Sentry alert fires, an editor reports something broken, or recommendations look empty, wrong, or stale. It confirms the symptom independently, probes competing hypotheses in parallel, stratifies metrics, tries to break its own conclusion, and writes an evidence-backed `FINDINGS.md` naming the check that would have caught the problem earlier. See [`skills/hnt-backend-investigation/SKILL.md`](skills/hnt-backend-investigation/SKILL.md) for details.

**Needs:** `gh`; `gcloud`/`bq` with a billing project; the `aws` CLI with a read-only SSO profile; a MySQL client plus Mozilla VPN; `uv` or `python3`; the Sentry MCP server if you have it. The skill runs on whatever subset you have — anything unreachable is raised as a task and reported under "Could not measure".
