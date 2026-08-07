# How New Tab (HNT) backend failures behave

Reference for the `hnt-backend-investigation` skill: the shapes these failures take, the invariants
to check, and the moves that kill a wrong hypothesis. SKILL.md names the moments to read it; the
alert audit below applies as soon as a report comes from an alert.

If you are editing this file later: it holds generalised mechanisms and techniques. It is deliberately
not a symptom-to-cause lookup, and it deliberately carries no worked incidents, issue ids, or canned
queries. Adding any of those changes what the file is for.

## Contents

- Failure classes — the vocabulary, for checking you have not narrowed too early
- If it came from an alert, audit the alert
- Why these failures are silent — the mechanisms that turn a failure into a valid-looking result
- Falsification moves that work — what to reach for when a story feels too clean
- Structural gaps to report rather than infer past
- Invariants to check — the hard ones, the ones relative to a stratum's own history, and the class each points to

## Failure classes

Vocabulary, not a lookup. Do not use this list to guess a cause from a symptom; derive the layer from
what you have measured. It is here so that when your hypothesis list has narrowed to one or two, you
can check it against the range of things that actually break in this stack:

vendor extraction-quality defect · upstream unavailability · silent validation rejection · resource
exhaustion or capacity cliff · ML model or routing failure · data-model or identity defect · filter
or threshold misconfiguration · unscoped bulk database operation · cascading failure or retry
amplification · observability defect · analytics correctness defect · dedup failure · authorization
defect · client or product defect · capacity near-miss

Observability defects are expanded in the next section, and the invariants at the end of this file
name the class each one points to.

## If it came from an alert, audit the alert

Alert quality is its own failure class, and worth a few minutes before chasing the system. This is an
addition to confirming the symptom independently, not a substitute for it.

- **Wrong layer** — the alert watches a gateway while the loss happens a hop upstream.
- **Umbrella fingerprint** — one issue collapses several unrelated failures; decompose by error
  message before trusting the title or the trend, and anchor on stable issue identifiers rather than
  numeric ids in the alert body.
- **Mis-centred or flapping band** — repeated transitions at a fixed offset within the hour, and
  margins under a percent, mean the band, not the system.
- **Counting artefact** — a panel that counts per open period rather than per calendar day can be off
  by an order of magnitude.
- **Unroutable** — if you cannot tell prod from staging from the alert alone, establish which one it
  is before reading anything else into it.

## Why these failures are silent

Not under-monitored — structurally unobservable. The recurring mechanism is that an upstream failure
gets converted into a valid-looking empty or partial result. Expect these:

- A pipeline that keeps only an allowlist of status codes and swallows request errors, so a total
  block is indistinguishable from "nothing found today".
- A vendor non-2xx arriving as a partially-filled success object, so a total outage raises nothing.
- A job reporting invocation-level success while silently dropping every item it received.
- Input-validation rejections that are logged and never surfaced.
- An `INNER JOIN` against a frozen upstream catalog shedding rows quietly instead of failing.
- A process that dies during startup, which emits no error event at all.
- A bulk write with no row-count guard.
- A write path disabled by a config flag, so the job runs, returns normally, and writes nothing.
- A queued handoff backing up or dead-lettering, which looks identical to the producer sending nothing.
- A cache serving stale content on upstream failure and extending its own expiry, so an outage
  upstream reaches clients as a successful response with old data.
- An aggregate quality gate passing while one class collapses beneath it.
- An alarm that treats missing data as "missing" rather than as a breach, sitting against an emitter
  that publishes nothing at zero — so it goes *quiet* during a total stop.

The consequence for every one of these: **liveness is not health.** Count what came out and compare
it to what went in.

## Falsification moves that work

Each of these has killed a plausible, confidently-held hypothesis. Reach for them when a story feels
too clean.

- **Read the code path.** Assumed behaviour of a resolver, filter, or job is the single most common
  source of a wrong mechanism.
- **Enumerate the whole population** rather than sampling, when it is free to count — in BigQuery or
  MySQL, "0 of N" is a different claim from "none in my sample". Against a metered API, sample, say
  what the sample is, and put any larger enumeration on the task list with its call count.
- **Repeat the observation N times** before calling anything non-deterministic, then vary one input
  at a time.
- **Read the actual limit** — quota, timeout, payload cap, row cap — instead of assuming which one
  binds. Adjacent limits on the same resource are easy to confuse.
- **Check the history of the constant or config** you are relying on; a value that used to be true is
  the classic stale-assumption trap.
- **Dump the attribute across the entire population** when you suspect truncation or a per-record
  limit.
- **Go to the other side's telemetry.** A vendor's or upstream's own status data will show what your
  logs structurally cannot.
- **Measure the same thing in a different plane.** Each plane here is blind in a particular way, and
  the pairs are what break structurally-invisible cases: Sentry sees only what was raised, while
  CloudWatch Logs Insights sees every operation including the ones that failed quietly; the BigQuery
  log sink sees requests the logging API will not surface; the vendor's stats see the responses your
  own pipeline discarded before logging them; client telemetry sees what users got when every backend
  table looks healthy.
- **Compare against sibling strata** at the same layer — the healthy peers localise the fault faster
  than reading code does.
- **Design a canary that would falsify** the hypothesis rather than a test that would confirm it.
- **Refute it yourself, in writing.** Before a conclusion leaves FINDINGS.md, draft the strongest
  case against it under "Hypotheses considered and dropped" and answer that case with a measurement.
  A second agent handed the evidence and told to break the conclusion is cheaper than waiting for a
  human reviewer.

## Structural gaps to report rather than infer past

These make certain questions unanswerable from stored data alone. When one blocks you, say so.

- No crawl attempt/success/error record, so a zero row count is ambiguous between "found nothing" and
  "every request failed".
- No lifecycle metadata or tombstones on the crawl target list, so an intentional retirement is
  indistinguishable from an outage.
- No persisted ML filter funnel, so per-stage drop-off cannot be reconstructed afterwards.
- No queryable deployment history. The running revision is available per service — Merino's
  `/__version__`, the lambdas' `GIT_SHA` — but there is no log of past deploys to line a timeline up
  against, so correlate against the current revision and the repo history instead.
- No status code or extraction-probability stored alongside cached hydration results, so a stored row
  does not imply a successful extraction.

## Invariants to check

Checks worth asserting against the data when they bear on your hypotheses: a broken one narrows the
search, and a holding one eliminates a line cheaply. Read the two groups differently. The first group
is a hard equality or a structural fact, so a violation means something is wrong. The second group is
relative to the stratum's own history, so a violation means *look here*, not *this is the fault* —
healthy production violates a naive absolute version of every one of them.

Hard:

| Invariant | Class it points to |
|---|---|
| Canonical domain ∈ approved list | Data-model or identity defect |
| Every bound parameter in shared SQL supplied by every caller | Analytics correctness defect |
| A derived dimension agrees with the entity it describes | Analytics correctness defect |
| The alarm itself emits a value at zero rather than nothing | Observability defect |
| Consumption against documented quota | Capacity near-miss |

Relative to the stratum's own trailing behaviour:

| Check | Class it points to |
|---|---|
| Articles per **day** > 0 per surface per source, plus the hourly count inside that stratum's own trailing range. An hour at zero is normal for low-volume strata and for a newly launched surface | Upstream unavailability; resource exhaustion or capacity cliff |
| Items served per section against that section's trailing count. There is no single configured size — the served count varies by layout and by request, so do not assert an equality | Silent validation rejection; ML model or routing failure |
| `items_written` against the job's own trailing ratio, not against `items_received`. Pipelines here dedupe by design, so a healthy ratio is far below one | Silent validation rejection |
| Input-validation reject count against its trailing floor. A steady non-zero floor is normal for third-party content; a step change is not | Silent validation rejection |
| Newest active item age per section against that surface's assembly interval | ML model or routing failure; resource exhaustion or capacity cliff |
| Vendor non-2xx share per domain below a **ceiling** | Upstream unavailability; vendor extraction-quality defect |
| Content-shape ratio per domain within bounds (empty-body share, title length) | Vendor extraction-quality defect |
| Per-domain funnel positive at every stage: discovered → hydrated → approved | Upstream unavailability; filter or threshold misconfiguration |
| Row-count delta on key tables within bounds | Unscoped bulk database operation |
| Queue depth, oldest-message age and dead-letter depth against their trailing values | Resource exhaustion or capacity cliff; silent validation rejection |

For the relative group, name the plane you measured in: Cloud Logging or the BigQuery log sink for the
crawl and the GCP jobs, CloudWatch Logs Insights for the AWS lambdas, and client telemetry for anything
expressed in users.

Stratify anything you assert. A rule that holds globally can be false for one surface, because the
surfaces do not all draw content the same way.
