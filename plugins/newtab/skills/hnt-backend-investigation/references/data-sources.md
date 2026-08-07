# New Tab (HNT) backend — where to look

Reference for the `hnt-backend-investigation` skill.

## Contents

- The pipeline, and which repo owns each stage
- Vocabulary: surface, locale, section, assembly, and the id to trace items by
- Sentry — projects and the traps in reading them
- Slack — `#hnt-dev-be-alerts`, as alert history and as where updates go
- Merino — live requests, deployed revision, GCP projects, ranking inputs, the serve-stale cache
- BigQuery — corpus and crawl state, client telemetry, and the traps in reading them
- Curated corpus MySQL — access and schema shape
- The editor plane — admin-api and curated-corpus-api
- AWS — profiles, the SQS handoff, and what logs give you that Sentry cannot
- Assembly and cadence — what runs, where, and how often
- Experiment enrolment
- Zyte — extraction API and the vendor's own stats
- **Access requests** — the one-line ask for each gated source

Write your own queries: this file gives you the shape of the data and the traps, not canned SQL.
Access varies by developer. Verify a source is reachable before building a plan around it, and if it
is not, follow "Don't stall on the developer" in SKILL.md — raise it once as a task and keep
investigating, rather than stopping or silently substituting a weaker source.

If you are editing this file later: it holds non-derivable access facts and traps that silently
produce wrong answers. Worked incidents, current issue ids, canned queries, and symptom-to-cause
lookups belong nowhere in this skill.

## The pipeline

**crawl / discovery → hydration (Zyte) → curated corpus → ML section assembly → SQS →
corpus-scheduler / section-manager lambda → curated-corpus-api → client-api → Merino → Firefox New
Tab.** Editors act on the corpus through curation-admin-tools → admin-api → curated-corpus-api.
Telemetry lands in BigQuery.

`client-api` is the Apollo federated router (`Pocket/pocket-monorepo`, `servers/client-api`) and it
is easy to miss: it has **no Sentry project of its own in either organisation**, so a failure there
surfaces as a Merino symptom. Merino also reaches it at the **prod** endpoint unconditionally —
`CorpusApiGraphConfig.endpoint` returns `CORPUS_API_PROD_ENDPOINT` regardless of Merino's own
environment, and the dev constant beside it is unused — so stage Merino reads prod corpus data.

| Repo | GitHub | Role |
|---|---|---|
| `merino-py` | mozilla-services/merino-py | Serves New Tab recommendations and Firefox Suggest |
| `content-monorepo` | Pocket/content-monorepo | Curated corpus, recommendations, section manager, the SQS lambdas |
| `content-ml-services` | mozilla/content-ml-services | Crawl, classification, section assembly (Metaflow, Cloud Functions) |
| `pocket-monorepo` | Pocket/pocket-monorepo | client-api federated router, shared infrastructure |
| `curation-admin-tools` | Pocket/curation-admin-tools | Editor-facing web app |
| `admin-api` | Pocket/admin-api | Federated GraphQL gateway for the admin tools |
| `bigquery-etl` | mozilla/bigquery-etl | New Tab engagement and Merino export ETL, and the Airflow DAGs behind it |
| `serverless-image-cache` | Pocket/serverless-image-cache | Thumbor image resize and cache |
| `firefox` | mozilla-firefox/firefox | Client side of the contract (`browser/extensions/newtab`) |

Locate clones rather than assuming paths:
`find ~ -maxdepth 3 -type d -name .git -print0 2>/dev/null | xargs -0 -n1 dirname`. For a repo that
is not cloned, read it through `gh api` or `gh search code`.

## Vocabulary: surface, locale, section, assembly

A **surface** is one locale/market feed of the corpus, written `NEW_TAB_EN_US`. A **section** is a
topic row inside a surface. **Assembly** is the ML stage that decides which corpus items sit in which
section for a surface; it runs as Metaflow flows in `content-ml-services` and reaches the corpus
through SQS, and the stage is called several things across these repos, so pin down which one a claim
refers to.

Merino's request `locale` is hyphenated (`en-US`) and the surface is **derived** from language plus
region by `get_recommendation_surface_id` in `merino/curated_recommendations/utils.py`, which also
branches on experiment enrolment — it is not a reformatting of the locale string. The `SurfaceId` enum
itself lives in `merino/curated_recommendations/corpus_backends/protocol.py`. Stratifying by locale and
stratifying by surface are therefore not the same slice, and some surfaces are reachable only through
enrolment. The cheapest resolution is a live response: it echoes the surface it resolved to.

Merino rewrites item URLs with `utm_source=firefox-newtab-<surface-in-lower-kebab>` (`get_utm_source`
and `update_url_utm_source` in `curated_recommendations/corpus_backends/utils.py`), so a URL from a
response generally will not match a stored `url`. Do not reconstruct the parameter yourself: the
lookup covers only some surfaces, and where it has no entry the URL comes back unmodified. Trace items
by the stable id instead: the response's `corpusItemId` is the corpus GraphQL item `id`, which lines
up with `ApprovedItem.externalId` in corpus MySQL and `approved_corpus_item_external_id` in BigQuery.
Confirm that last hop on the first item you trace rather than assuming it.

## Sentry

Reachable through the Sentry MCP server (its tools are prefixed `mcp__sentry__`). Prefer the
event-search tool over issue-search when you need a volume breakdown by error message — issue search
alone will not decompose an umbrella fingerprint.

**If no `mcp__sentry__` tools are available, the server is not set up in this session, and you should
say so immediately.** Do not weigh this one against your other lines first; most symptoms here are
error-shaped, and steps 1, 2 and 6 all read from Sentry, so an investigation without it is working
half-blind. Raise it as a `User action:` task with the command:

```
! claude mcp add --scope user --transport http sentry https://mcp.sentry.dev/mcp
```

followed by `/mcp` to authenticate in the browser. Servers load when a session starts, so the tools
may not appear until Claude Code is restarted. Keep going meanwhile: service logs and the BigQuery log
sink cover part of the same ground, and the developer can read event counts off the Sentry web UI for
you in the interim.

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
- Absence of events is weak evidence: a process that dies at startup, a job that drops items while
  reporting success, and a write path disabled by a config flag all emit nothing.

## Slack

`#hnt-dev-be-alerts` carries the backend alerts for this stack, which makes it two things at once: a
record of what has already fired, worth searching in step 2 before you conclude something is new, and
the place investigation updates go. Its tools are prefixed `mcp__slack__`. Install it through the
official plugin rather than a bare `mcp add`, because the server does not support dynamic client
registration and a plain add will connect-fail:

```
/plugin install slack@claude-plugins-official
```

The OAuth round trip needs a fixed local callback port, so it can collide with another session doing
the same thing at the same moment. Reading the channel is free. Posting needs the developer's approval
for **each** message, and prefers a reply in the existing alert thread over a new one: the rule is at
the end of step 8 in SKILL.md. With no Slack tools, alert history is **unchecked**, not empty — do not
record "nothing similar has fired" on the strength of a source you could not read.

## Merino

Reproducing the client call is often the fastest confirmation of a client-visible symptom:
`POST https://merino.services.mozilla.com/api/v1/curated-recommendations` with a JSON body carrying
`locale`, `region`, `topics`, `sections`, `feeds` (e.g. `["sections"]`), `enableInterestPicker`, and
optionally `experimentName` / `experimentBranch` to land on an experiment branch. Send a realistic
Firefox `User-Agent`. Save every response. First things to check on the payload: section count,
items per section, presence of `followable` / `allowAds`, and the age of the newest item.

`GET /__version__` returns the running commit and build URL. Use it before trusting a repo log:
a merged commit is not a deployed one, and a rollback is the change that most cleanly explains a
symptom that started or stopped on its own. The lambdas carry the same information as a `GIT_SHA`
environment variable.

Keep live calls to the ones you need. If you find yourself looping over locales or branches against
prod, query the telemetry instead. Stage answers payload-shape questions only — it reads prod corpus
data — and its host is internal-only, so a request from outside the network returns 404 rather than an
auth error: `https://stage.merino.nonprod.webservices.mozgcp.net`.

GCP projects, for log reads, metrics and GCS listings. Confirm these against the deployed values files
in the deployment repo rather than the in-repo TOMLs, which lag:

| Environment | Project |
|---|---|
| Merino prod | `moz-fx-merino-prod-5de4`; some buckets in `moz-fx-merino-prod-1c2f` |
| Merino stage | `moz-fx-merino-nonprod-db57` (deployed); `moz-fx-merino-nonprod-ee93` is legacy and still named in `stage.toml` |
| Crawl and ML | `moz-fx-mozsoc-ml-prod`, with gen2 Cloud Functions `prod-crawl-handler`, `prod-page-crawler`, `prod-ingester`, `prod-parser` |

When the logging API returns nothing useful, the BigQuery log sink for the same project usually does.
That sink table is far larger than anything in the BigQuery list below — check whether it has a
partition column and filter on it, aggregate inside the query rather than pulling rows, and
`--dry_run` first.

**Ranking inputs arrive as GCS blobs on a timer, and a stalled producer is invisible everywhere else.**
Merino pulls `engagement/latest.json` and `priors/latest.json` from its exports bucket on a short
interval and feeds them to the Thompson-sampling ranker (see `[default.curated_recommendations.gcs]`
in `merino/configs/default.toml`). If the producing Airflow DAG in `bigquery-etl` stops, Merino keeps
serving with frozen ranking: it logs at INFO, raises nothing, and no row in any table below changes.
Compare the blob's update time against the producing DAG's current `schedule_interval` rather than
against a remembered cadence.

**Merino serves stale on upstream failure, per pod.** When the corpus fetch fails and a warm cache
entry exists, it returns the stale entry *and* pushes the expiry out again, so the stale window has no
fixed bound while the upstream stays down. A corpus-api or client-api outage therefore reaches clients
as a successful 200 with old content, and only cold pods raise. One live request cannot refute an
upstream problem, and cannot establish its scope either.

## BigQuery

`bq` ships with the Google Cloud SDK and runs on your `gcloud` credentials, so a missing binary or an
unauthenticated session is the first thing to rule out, ahead of any dataset permission: `gcloud auth
list` shows whether there is an active account. The same credentials cover the `gcloud logging` and
GCS reads above. Confirm the billing project with a `--dry_run` before the first real query; a
personal sandbox is usually `moz-fx-dev-<ldap>-sandbox`. Anything missing there is an access task, and
authenticating is interactive so it belongs to the developer.

Corpus, section and crawl state:

| Table | What it is for |
|---|---|
| `moz-fx-mozsoc-ml-prod.prod_rss_news.rss_feed_items` | Articles discovered by crawl — per-surface, per-source (`PAGE` vs `RSS`) volume over time |
| `moz-fx-mozsoc-ml-prod.prod_articles.zyte_cache` | One row per hydration attempt, not per article — see the traps below before counting or joining it |
| `moz-fx-data-shared-prod.snowflake_migration_derived.sections_v1` | Section existence, enable/disable state, and surface, as an event stream |
| `moz-fx-data-shared-prod.snowflake_migration_derived.section_items_v1` | Which items sit in which section, and when each was last touched — the freshness check for a stalled section |
| `moz-fx-data-shared-prod.snowflake_migration_derived.corpus_items_current_v1` | One row per corpus item, deduped — but it keeps the latest row even when that row is a removal, so filter status yourself |
| `moz-fx-data-shared-prod.snowflake_migration_derived.scheduled_corpus_items` | Scheduled items; one row per item in practice |

Client-side telemetry, which is the only plane that answers "how many users" and the only one carrying
experiment branch or browser version:

| Table | What it is for |
|---|---|
| `moz-fx-data-shared-prod.telemetry_derived.newtab_visits_v1` | The experiment and browser-version stratum (`experiments`, `browser_version`, locale, country). Very large and `requirePartitionFilter` is on |
| `moz-fx-data-shared-prod.firefox_desktop.newtab_content_live` | The live plane, minutes behind rather than a day — the only one that can confirm a symptom that is happening now |
| `moz-fx-data-shared-prod.firefox_desktop_derived.newtab_content_items_daily_v1` | Daily item-level impressions and clicks. Carries no locale and no experiment branch, and its version column is null for most volume, so do not use it for those cuts |

The derived tables are T+1, so they cannot confirm a live symptom; reach for the live table when the
question is "is this happening right now". Confirm each object still exists before drawing a conclusion
from an empty result, and use `bq ls <dataset>` to settle whether something is a table or a view and
whether it is partitioned — a view has no partition column, so ordering a partition filter on one is a
query error.

Traps that will silently give you a wrong answer:

- **`zyte_cache` is not keyed on `canonical_url`.** It holds one row per hydration attempt; a single
  url can have hundreds. `rss_feed_items` is not unique on `canonical_url` either. So the natural
  discovered-to-hydrated funnel join fans out several-fold and inflates every count in it. Dedupe
  **both** sides to one row per url before joining, and note that the event-log warning below is about
  a different set of tables — it does not make this one safe to count.
- **There is no domain column on `rss_feed_items`.** Derive the domain from `canonical_url` (or
  `origin_url` for the page it was found on). A per-domain funnel is a computed grouping, not a
  lookup.
- **`source` is null for everything crawled before 2025-08-11**, several million rows. A long
  per-source series will show `PAGE` and `RSS` springing into existence on that date; that is the
  column being introduced, not the crawl changing.
- **`crawled_date` and `published_date` are STRING; `crawled_at`, `published_at` and `loaded_at` are
  TIMESTAMP.** Use the timestamps for any time arithmetic.
- **Cost does not work the way you expect.** `rss_feed_items` (tens of GB) and `zyte_cache` (over a
  hundred GB) are **unpartitioned and unclustered**, so a date predicate reduces nothing — only column
  selection does. Never `SELECT *` on them and always `--dry_run` first. The
  `snowflake_migration_derived` tables are day-partitioned on `happened_at` but do not require a
  partition filter, so supply one yourself.
- **The `*_v1` event tables are event logs, not current state.** Rows accumulate per change, so a
  plain `COUNT(*)` over-counts. Where you must reduce the log yourself, take the latest row per id and
  check the inflation ratio rather than assuming it.
- **`approved_corpus_items` is a filter, not a deduplication.** It excludes removed items but still
  carries multiple rows per item, so it is not a current-state view despite the name.
- **`sections_v1.source` is unusable for current sections** — null on about four fifths of all
  sections and on **every** currently-active one. Section ownership (`MANUAL` vs `ML`) comes from
  `Section.createSource` / `updateSource` / `deactivateSource` in corpus MySQL. A null `source` means
  unknown, never `MANUAL`.
- **Surface identifiers differ by table.** Crawl data uses `en_US`; section data uses
  `NEW_TAB_EN_US`. Joining or comparing them naively produces empty results that look like an outage.
- **The crawl surface set, the served surface set, and the set of surfaces ML actually runs for are
  three different sets**, so a surface missing from one of them is not automatically a bug.
- **Source mix varies by surface** — some have no RSS-sourced content, some no page-crawled content.
  Judge each surface against its own history.
- **Volume is strongly day-of-week seasonal.** Compare against a trailing median, never against
  yesterday.
- Row counts and byte sizes drift; treat any figure here as an order of magnitude and re-measure with
  `bq show` when the number matters.
- A zero in the crawl table is ambiguous between "found nothing" and "every request failed"; there is
  no attempt/error record to disambiguate it.

## Curated corpus MySQL

Production database behind curated-corpus-api, reached through a preconfigured read-only login path.
Check what exists with `mysql_config_editor print --all`, and expect to need VPN — a hang rather than
an auth error is the usual symptom of being off it.

Useful invocation guards: `--safe-updates` (caps returned rows at 1000 and aborts queries estimated
to examine over a million) and `SET SESSION max_execution_time=10000` (10s server-side cap). The
1000-row cap **truncates silently**, so add an explicit `LIMIT` or raise the cap when you need a
complete set. The million-row abort surfaces as `ERROR 1104`, and it is your own guard rather than
access or VPN — it fires on the first aggregate over `SectionItem`. Lift the examined-rows ceiling with
`SET SESSION sql_big_selects=1` or `--max-join-size`, which is a different control from the row cap.

Schema `curation_corpus`. The tables that matter: `ApprovedItem` (the corpus itself, keyed by
`externalId` and `url`), `SectionItem` and `Section` (placement and section config — `Section` also
carries `createSource` / `updateSource` / `deactivateSource`, the authoritative ML-vs-manual
ownership), `ScheduledItem` (surface scheduling, and its `scheduledDate` is a zoneless calendar date in
the surface's own timezone), `RejectedCuratedCorpusItem`, and the `PublisherDomain` / `TrustedDomain` /
`ExcludedDomain` domain lists. `SectionItem` runs into the millions of rows and `ApprovedItem` into the
hundreds of thousands — check indexes with `SHOW INDEX` before filtering, since several obvious filter
columns are unindexed.

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

**The ML-to-corpus handoff is a queue, and it is where a day's candidates go missing quietly.** The
corpus-scheduler and section-manager lambdas are each fed by SQS with concurrency of one and a batch
size of one, and the shared construct gives each queue a dead-letter queue named after it. Check queue
depth, in-flight count, oldest-message age, and DLQ depth before concluding that ML produced nothing:
a backlog and a drained DLQ look identical in every corpus table. Two mechanics worth knowing: a
message whose handler outlives the queue's visibility timeout is redelivered while still in flight and
can exhaust its receive count without any error being raised, and the write path is gated by an
`ALLOWED_TO_SCHEDULE` flag that makes the lambda return normally while writing nothing. ML also has one
path that bypasses the queue and calls admin-api directly, so do not describe the handoff as
queue-only.

CloudWatch Logs Insights is frequently the source that cracks a case for these lambdas — it will give
you total operation counts and per-error-type breakdowns that Sentry structurally cannot, because
Sentry only sees what was raised. It bills per GB scanned per query, so pass the narrowest
`--start-time`/`--end-time` that could answer the question and `stats`-aggregate rather than dumping
`fields`. Also useful: alarm history (transition timestamps and state-reason margins), and pulling an
anomaly band itself as a metric-math series to compare its predicted centre against reality. The crawl
runs in GCP, not AWS, so its equivalent plane is Cloud Logging or the BigQuery log sink.

SSO sessions expire and the login is interactive, so it has to be the developer: see the table below.

## Assembly and cadence

Freshness thresholds are a common thing to want and a common thing to invent. Derive the intended
cadence from the `@schedule` decorators on the Metaflow flows in `content-ml-services`, and record the
interval you used in FINDINGS.md. Two traps sit in that derivation:

- The cron is wrapped so that it **only takes effect when a deploy-time environment variable is set**;
  otherwise the decorator receives a deliberately impossible date that never fires. Reading the literal
  cron out of the source and reporting "runs daily at 11:00" can be wrong twice over.
- The decorator also passes a per-surface timezone, so the schedule is surface-local, not UTC.

Which flows exist at all is per-locale, listed in the deployed-locale and deployed-flow JSON manifests
in the same repo. A surface with no deployed flow is a third possibility alongside a crawl gap and a
serving gap.

## Experiment enrolment

Step 5 makes experiment branch one of the first cuts, and the branch names are not in any of the
tables above. The Experimenter API lists live and recent experiments without authentication:
`https://experimenter.services.mozilla.com/api/v6/experiments/`. Use it to get the real slug and branch
names before slicing telemetry, rather than inventing them or asking.

## Zyte

Two separate APIs, both **metered — every call costs money**. Save all responses; never bulk-crawl to
satisfy curiosity. Check for a key with `[ -n "$ZYTE_API_KEY" ] && echo present`. Reference keys by
variable name in anything you save, so no value lands in a query file or the transcript.

**Extraction API** (`ZYTE_API_KEY`, created at https://app.zyte.com/o/612928/zyte-api/api-access)
reproduces what the crawler saw for a URL. Request `article` for a single page or `articleList` for an
index page — not both — and put `extractFrom: "httpResponseBody"` inside `articleOptions` /
`articleListOptions` to skip the browser. Check **`statusCode` and the extraction probability before
anything else**: a non-2xx, a bot wall, or a "JavaScript is disabled" page still returns a populated
object that reads like success. Probability lives at `article.metadata.probability` for a single page
and per item at `articleList.articles[].metadata.probability` — `articleList.metadata` carries only
`dateDownloaded`. Compare `canonicalUrl` against the URL you requested; cross-domain canonicals pull
unapproved domains into the corpus.

**Stats API** (`https://zyte-api-stats.zyte.com/api/stats`) is the vendor's own view of your traffic:
per-domain response-code distribution over time. It takes a **different credential** — the Zyte
dashboard API key from the organisation's settings page, explicitly not the Zyte API key above — as the
HTTP basic username with an empty password, so the extraction key will be rejected here.
`organization_id` is **required** and is `612928`. Only `groupby_time` and `groupby_domain` group;
`response_codes` is a *filter*, not a grouping, and `include_domain_health=true` is rejected without
`groupby_domain=true`. This is the right source for "did this domain start failing, and when" — your
own logs will not show it if the pipeline discards non-allowlisted status codes.

## Access requests

Raise one of these when the source looks promising, then keep working. The two Sentry rows are the
exception: raise those on sight, per step 1. Each is a short errand, so give the developer the command
rather than a description of the problem.

| Blocked source | Ask them to |
|---|---|
| Corpus MySQL hangs rather than erroring | Connect to Mozilla VPN, then say so; if it still hangs the login path itself is stale |
| No read-only MySQL login path configured | Create one: `mysql_config_editor set --login-path=prod-curated-corpus-api-readonly --host=<host> --user=<user> --password`, and tell you the path name |
| AWS SSO session expired | Run `! aws --profile <profile> sso login`, so the output lands here |
| No AWS profile at all | Run `! aws configure sso` for a read-only role, or name a profile already in their `~/.aws/config` |
| `gcloud` or `bq` not installed | Install the Google Cloud SDK, which provides both |
| `gcloud` installed but not authenticated | Run `! gcloud auth login`, and `! gcloud auth application-default login` as well if you need the Python client libraries |
| No billing project configured | Name one they can bill, usually `moz-fx-dev-<ldap>-sandbox`, or set it with `gcloud config set project <id>` |
| Permission denied on a dataset or a Merino project | Request read access, or viewer on the project; say meanwhile whether the question is about payload shape, which stage can answer |
| Zyte extraction key missing | Create one at https://app.zyte.com/o/612928/zyte-api/api-access, `export ZYTE_API_KEY=<key>` in the shell they launch from, and restart the session. Or have them run the single extraction and paste back the JSON, not the key |
| Zyte Stats key missing | Issue a **dashboard** API key from the Zyte organisation settings page and export it; the extraction key will not authenticate against the Stats API |
| No `mcp__sentry__` tools at all | Run `! claude mcp add --scope user --transport http sentry https://mcp.sentry.dev/mcp`, then `/mcp` to authenticate; the tools appear after a session restart |
| Sentry connected but unauthenticated or scoped too narrowly | Run `/mcp` and authenticate for the `mozilla` org, or read back the issue's event counts broken down by error message |
| No `mcp__slack__` tools | Run `/plugin install slack@claude-plugins-official`, then `/mcp` to authenticate; or post the drafted message to `#hnt-dev-be-alerts` themselves |
| Editor-facing symptom needs an authenticated session | Reproduce the click themselves and report the exact error text and time |
| The answer is in a dashboard you cannot reach | Open it, apply the specific filter you name, and read back the one number or shape you asked for |

Notice the last row: when a dashboard, explore, or console holds the answer, asking for *access* is
usually the slower path. Asking a precise question about what it shows is faster for both of you.
