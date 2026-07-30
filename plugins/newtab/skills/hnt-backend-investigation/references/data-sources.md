# New Tab (HNT) backend — where to look

Reference for the `hnt-backend-investigation` skill.

## Contents

- The pipeline, and which repo owns each stage
- Vocabulary: surface, locale, section, and the id to trace items by
- Sentry — projects and the traps in reading them
- Merino — live requests and GCP projects
- BigQuery — the tables worth knowing and the traps in reading them
- Curated corpus MySQL — access and schema shape
- The editor plane — admin-api and curated-corpus-api
- AWS — profiles, and what logs give you that Sentry cannot
- Zyte — extraction API and the vendor's own stats
- **Access requests** — the one-line ask for each gated source

Write your own queries: this file gives you the shape of the data and the traps, not canned SQL.
Access varies by developer. Verify a source is reachable before building a plan around it, and if it
is not, follow "Don't stall on the developer" in SKILL.md — raise it once as a task and keep
investigating, rather than stopping or silently substituting a weaker source.

## The pipeline

**crawl / discovery → hydration (Zyte) → curated corpus → ML section assembly → curated-corpus-api →
client-api → Merino → Firefox New Tab.** Editors act on the corpus through curation-admin-tools →
admin-api → curated-corpus-api. Telemetry lands in BigQuery.

`client-api` is the Apollo federated router (`Pocket/pocket-monorepo`, `servers/client-api`) and it
is easy to miss: it has **no Sentry project of its own in either organisation**, so a failure there
surfaces as a Merino symptom. Merino also reaches it at the **prod** endpoint unconditionally —
`CorpusApiGraphConfig.endpoint` returns `CORPUS_API_PROD_ENDPOINT` regardless of Merino's own
environment, and the dev constant beside it is unused — so stage Merino reads prod corpus data.

| Repo | GitHub | Role |
|---|---|---|
| `merino-py` | mozilla-services/merino-py | Serves New Tab recommendations and Firefox Suggest |
| `content-monorepo` | Pocket/content-monorepo | Curated corpus, recommendations, section manager |
| `content-ml-services` | mozilla/content-ml-services | Crawl, classification, section assembly (Metaflow, Cloud Functions) |
| `pocket-monorepo` | Pocket/pocket-monorepo | client-api federated router, shared infrastructure |
| `curation-admin-tools` | Pocket/curation-admin-tools | Editor-facing web app |
| `admin-api` | Pocket/admin-api | Federated GraphQL gateway for the admin tools |
| `bigquery-etl` | mozilla/bigquery-etl | New Tab engagement and Merino recommendation ETL |
| `serverless-image-cache` | Pocket/serverless-image-cache | Thumbor image resize and cache |
| `firefox` | mozilla-firefox/firefox | Client side of the contract (`browser/extensions/newtab`) |

Locate clones rather than assuming paths:
`find ~ -maxdepth 3 -type d -name .git -print0 2>/dev/null | xargs -0 -n1 dirname`. For a repo that
is not cloned, read it through `gh api` or `gh search code`.

## Vocabulary: surface, locale, section

A **surface** is one locale/market feed of the corpus, written `NEW_TAB_EN_US`. A **section** is a
topic row inside a surface. Merino's request `locale` is hyphenated (`en-US`) and the surface is
**derived** from language plus region — the mapping table is `SurfaceId` in
`merino/curated_recommendations/localization.py`, not a reformatting of the locale string. Some
surfaces are reachable only through experiment enrolment. Stratifying by locale and stratifying by
surface are therefore not the same slice.

Merino appends `utm_source=firefox-newtab-<surface>` to item URLs (`get_utm_source` and
`update_url_utm_source` in `curated_recommendations/corpus_backends/utils.py`), so a URL taken from a
response will never match a stored `url`. Trace items by the stable id instead: the response's
`corpusItemId` is the corpus GraphQL item `id`, which lines up with `ApprovedItem.externalId` in
corpus MySQL and `approved_corpus_item_external_id` in BigQuery. Confirm that last hop on the first
item you trace rather than assuming it.

## Sentry

Reachable through the Sentry MCP server if one is connected (its tools are prefixed `mcp__sentry__`).
Prefer the event-search tool over issue-search when you need a volume breakdown by error message —
issue search alone will not decompose an umbrella fingerprint. If no Sentry tools are present, treat
it as a blocked source.

The HNT services are in the **`mozilla`** organisation, prefixed `hnt-`; issue short-ids look like
`HNT-CRAWL-9`.

`hnt-crawl` · `hnt-metaflow` · `hnt-curated-corpus-api` · `hnt-admin-api` ·
`hnt-curation-admin-tools` · `hnt-section-manager-lambda` · `hnt-corpus-scheduler-lambda` ·
`hnt-prospect-api` · `hnt-prospect-translation-lambda` · `hnt-serverless-image-cache` ·
`hnt-braze-content-proxy` · `hnt-feature-flags`

Merino is separate: project `merino-py`, also in `mozilla`. Some older projects exist in the `pocket`
org; prefer the `hnt-` ones for anything current. Enumerate the current projects before relying on
this list — a project that does not resolve is a naming change, not evidence of zero errors.

Traps:

- **`first seen` is bounded by retention**, so it is not onset. Nearly every long-running issue
  reports a first-seen date at the edge of the retention window.
- Issue volume is dominated by long-standing error floors. Establish what was already there before
  attributing anything to today, and check whether a floor *changed* rather than whether it exists.
- Decompose an issue by error message before trusting its title or trending it.
- Absence of events is weak evidence: a process that dies at startup, or a job that drops items
  while reporting success, emits nothing.

## Merino

Reproducing the client call is often the fastest confirmation of a client-visible symptom:
`POST https://merino.services.mozilla.com/api/v1/curated-recommendations` with a JSON body carrying
`locale`, `region`, `topics`, `sections`, `feeds` (e.g. `["sections"]`), `enableInterestPicker`, and
optionally `experimentName` / `experimentBranch` to land on an experiment branch. Send a realistic
Firefox `User-Agent`. Save every response. First things to check on the payload: section count,
items per section, presence of `followable` / `allowAds`, and the age of the newest item.

Keep it to the requests you need. If you find yourself writing a loop over locales or branches
against prod, query the telemetry instead. Stage answers payload-shape questions — but remember it
reads prod corpus data, so it cannot answer questions about prod corpus content.

GCP projects, for log reads, metrics, and GCS listings — confirm against
`merino/configs/production.toml` **and `merino/configs/stage.toml`** rather than trusting this table:

| Environment | Project |
|---|---|
| Merino prod | `moz-fx-merino-prod-5de4`; some buckets in `moz-fx-merino-prod-1c2f` |
| Merino stage | `moz-fx-merino-nonprod-ee93`; also `moz-fx-merino-nonprod-db57` |

When the logging API returns nothing useful, the BigQuery log sink for the same project usually does.
That sink table is far larger than anything in the BigQuery list below — check whether it has a
partition column and filter on it, aggregate inside the query rather than pulling rows, and
`--dry_run` first.

## BigQuery

Confirm the billing project with a `--dry_run` before the first real query; a personal sandbox is
usually `moz-fx-dev-<ldap>-sandbox`. If you cannot confirm one, raise it as an access task.

| Table | What it is for |
|---|---|
| `moz-fx-mozsoc-ml-prod.prod_rss_news.rss_feed_items` | Articles discovered by crawl — per-surface, per-source (`PAGE` vs `RSS`) volume over time |
| `moz-fx-mozsoc-ml-prod.prod_articles.zyte_cache` | Hydrated article metadata keyed on `canonical_url` — what the pipeline believes an article says |
| `moz-fx-data-shared-prod.snowflake_migration_derived.sections_v1` | Section existence, enable/disable state, and surface, as an event stream |
| `moz-fx-data-shared-prod.snowflake_migration_derived.section_items_v1` | Which items sit in which section, and when each was last touched — the freshness check for a stalled section |
| `moz-fx-data-shared-prod.snowflake_migration_derived.corpus_items_current_v1` | Current-state view of corpus items — prefer this to reducing the event table yourself |
| `moz-fx-data-shared-prod.snowflake_migration_derived.approved_corpus_items` / `scheduled_corpus_items` | Maintained current-state views for approved and scheduled items |

Confirm each table still exists before drawing a conclusion from an empty result.

Traps that will silently give you a wrong answer:

- **There is no domain column on `rss_feed_items`.** Derive the domain from `canonical_url` (or
  `origin_url` for the page it was found on). A per-domain funnel is a computed grouping, not a
  lookup.
- **`source` is null for everything crawled before 2025-08-11** — 6.8M of 22.6M rows. A long
  per-source series will show `PAGE` and `RSS` springing into existence on that date; that is the
  column being introduced, not the crawl changing.
- **`crawled_date` and `published_date` are STRING; `crawled_at`, `published_at` and `loaded_at` are
  TIMESTAMP.** Use the timestamps for any time arithmetic.
- **Cost does not work the way you expect.** `rss_feed_items` (~23.6 GB) and `zyte_cache` (~128 GB)
  are **unpartitioned and unclustered**, so a date predicate reduces nothing — only column selection
  does. Never `SELECT *` on them and always `--dry_run` first. The `snowflake_migration_derived`
  tables are day-partitioned on `happened_at` but do not require a partition filter, so supply one
  yourself.
- **The `*_v1` event tables are event logs, not current state.** Rows accumulate per change, so a
  plain `COUNT(*)` over-counts. Prefer the current-state views above; where you must reduce the log
  yourself, take the latest row per id and check the inflation ratio rather than assuming it.
- **`sections_v1.source` is unusable for current sections** — null on 81% of all sections and on
  **every** currently-active one. Section ownership (`MANUAL` vs `ML`) comes from
  `Section.createSource` / `updateSource` / `deactivateSource` in corpus MySQL. A null `source` means
  unknown, never `MANUAL`.
- **Surface identifiers differ by table.** Crawl data uses `en_US`; section data uses
  `NEW_TAB_EN_US`. Joining or comparing them naively produces empty results that look like an outage.
- **The crawl surface set and the serving surface set are not identical**, so a crawled surface with
  no sections is not automatically a bug.
- **Source mix varies by surface** — some have no RSS-sourced content, some no page-crawled content.
  Judge each surface against its own history.
- **Volume is strongly day-of-week seasonal.** Compare against a trailing median, never against
  yesterday.
- A zero in the crawl table is ambiguous between "found nothing" and "every request failed"; there is
  no attempt/error record to disambiguate it.

## Curated corpus MySQL

Production database behind curated-corpus-api, reached through a preconfigured read-only login path.
Check what exists with `mysql_config_editor print --all`, and expect to need VPN — a hang rather than
an auth error is the usual symptom of being off it.

Useful invocation guards: `--safe-updates` (caps returned rows at 1000 and aborts queries estimated
to examine over a million) and `SET SESSION max_execution_time=10000` (10s server-side cap). The
1000-row cap **truncates silently**, so add an explicit `LIMIT` or raise the cap when you need a
complete set.

Schema `curation_corpus`. The tables that matter: `ApprovedItem` (the corpus itself, keyed by
`externalId` and `url`), `SectionItem` and `Section` (placement and section config — `Section` also
carries `createSource` / `updateSource` / `deactivateSource`, the authoritative ML-vs-manual
ownership), `ScheduledItem` (surface scheduling), `RejectedCuratedCorpusItem`, and the
`PublisherDomain` / `TrustedDomain` / `ExcludedDomain` domain lists. `SectionItem` runs into the
millions of rows and `ApprovedItem` into the hundreds of thousands — check indexes with `SHOW INDEX`
before filtering, since several obvious filter columns are unindexed.

## The editor plane

When the symptom is editor-facing, reproducing it means an authenticated editor session, which is an
interactive login and therefore the developer's errand rather than yours. What you can do alone:
`hnt-admin-api` and `hnt-curated-corpus-api` in Sentry for the failure window, the corpus rows the
mutation would have touched, and the resolver itself in `Pocket/admin-api` and
`Pocket/content-monorepo`. That is usually enough to name the mechanism without a session.

## AWS

content-monorepo infrastructure runs in AWS. Discover profiles with
`grep -E '^\[profile' ~/.aws/config` and confirm one works with `sts get-caller-identity`; match prod
against dev to the environment you are investigating.

CloudWatch Logs Insights is frequently the source that cracks a case — it will give you total
operation counts and per-error-type breakdowns that Sentry structurally cannot, because Sentry only
sees what was raised. It bills per GB scanned per query, so pass the narrowest
`--start-time`/`--end-time` that could answer the question and `stats`-aggregate rather than dumping
`fields`. Also useful: alarm history (transition timestamps and state-reason margins), and pulling an
anomaly band itself as a metric-math series to compare its predicted centre against reality.

SSO sessions expire and the login is interactive, so it has to be the developer: see the table below.

## Zyte

Two separate APIs, both **metered — every call costs money**. Save all responses; never bulk-crawl to
satisfy curiosity. Check for a key with `[ -n "$ZYTE_API_KEY" ] && echo present`, and reference it as
`$ZYTE_API_KEY` in anything you save, so no key is written into a query file or the transcript.

**Extraction API** (`ZYTE_API_KEY`) reproduces what the crawler saw for a URL. Request `article` for
a single page or `articleList` for an index page — not both — and put `extractFrom:
"httpResponseBody"` inside `articleOptions` / `articleListOptions` to skip the browser. Check
**`statusCode` and the extraction probability before anything else**: a non-2xx, a bot wall, or a
"JavaScript is disabled" page still returns a populated object that reads like success. Probability
lives at `article.metadata.probability` for a single page and per item at
`articleList.articles[].metadata.probability` — `articleList.metadata` carries only
`dateDownloaded`. Compare `canonicalUrl` against the URL you requested; cross-domain canonicals pull
unapproved domains into the corpus.

**Stats API** (`ZYTE_SECRET_KEY`, `https://zyte-api-stats.zyte.com/api/stats`) is the vendor's own
view of your traffic: per-domain response-code distribution over time. `organization_id` is
**required** and auth is HTTP basic with the secret key as the username and an empty password, so a
bare GET on the endpoint fails validation — take the org id from the production caller
(`scripts/fetch/zyte_stats.py` in `content-ml-services`) rather than guessing. Only `groupby_time`
and `groupby_domain` group; `response_codes` is a *filter*, not a grouping, and
`include_domain_health=true` is rejected without `groupby_domain=true`. This is the right source for
"did this domain start failing, and when" — your own logs will not show it if the pipeline discards
non-allowlisted status codes.

Freshness thresholds are a common thing to want here and a common thing to invent. Derive the
intended cadence from the assembly schedule in `content-ml-services` and the corpus-scheduler-lambda
trigger config rather than guessing, and record the interval you used in FINDINGS.md.

## Access requests

Raise one of these only when that source is one of your strongest current lines, then keep working.
Each is a short errand: give the developer the command, not a description of the problem.

| Blocked source | Ask them to |
|---|---|
| Corpus MySQL hangs rather than erroring | Connect to VPN. If it still hangs, ask whether the read-only login path is still valid |
| No read-only MySQL login path configured | Set one up, or give you the host and read-only user to configure |
| AWS SSO session expired | Run `! aws --profile <profile> sso login` in-session, so the output lands here |
| No AWS profile at all | Say which read-only profile they have, or request one for the account you need |
| BigQuery permission denied on a dataset | Confirm which billing project to use, or request read access to the dataset |
| `gcloud` lacks access to a Merino project | Request viewer on the project; say meanwhile whether the question is about payload shape, which stage can answer |
| Zyte key missing from the environment | Export it in the shell they launched from and restart the session, or run the one extraction you need and paste back the JSON — the JSON, not the key |
| Sentry MCP unavailable or scoped too narrowly | Authenticate the Sentry MCP server for the `mozilla` org, or paste the issue's event counts broken down by error message |
| Editor-facing symptom needs an authenticated session | Reproduce the click themselves and report the exact error text and time |
| The answer is in a dashboard you cannot reach | Open it, apply the specific filter you name, and read back the one number or shape you asked for |

Notice the last row: when a dashboard, explore, or console holds the answer, asking for *access* is
usually the slower path. Asking a precise question about what it shows is faster for both of you.
