# How New Tab (HNT) backend failures behave

Reference for the `hnt-backend-investigation` skill: the shapes these failures take, the invariants
worth asserting, and the moves that kill a wrong hypothesis. Read this before you take any
hypothesis seriously.

## Contents

- Classify the shape first — failure classes and what each looks like on first contact
- If it came from an alert, audit the alert
- Why these failures are silent — the mechanisms that turn a failure into a valid-looking result
- Falsification moves that work — what to reach for when a story feels too clean
- Structural gaps to report rather than infer past
- Invariants worth asserting — hard checks, and the class each one detects

## Classify the shape first

The tells below are heuristics for orienting, not proof. Use them to widen the hypothesis list at
step 3, not to shortcut it.

| Failure class | What you tend to see first |
|---|---|
| Vendor extraction-quality defect | Status codes fine and error rate flat, but a content-shape ratio per domain drifts — empty-body share, title length, headline churn |
| Upstream unavailability | Your own logs contain *nothing*; a per-domain count goes to zero with no errors logged. Visible only in the vendor's own status data |
| Silent validation rejection | `items_written < items_received`, exit status success, no error raised |
| Resource exhaustion / capacity cliff | Output stops abruptly and completely, tracking input size or scale rather than time of day |
| ML model or routing failure | Job succeeded and the aggregate gate passed, while one class's output is zero and its siblings are current |
| Data-model / identity defect | Counts look normal but the rows are wrong — the same content living under two keys (URL, canonical, slug, external id) |
| Filter / threshold misconfiguration | The affected population clusters exactly on a boundary value; or a global aggregate breaches while every stratum is fine |
| Unscoped bulk database operation | An entire entity's row count changes at one timestamp, with nothing thrown |
| Cascading failure / retry amplification | Outbound attempts per key far above the design rate; error volume scales with fleet size rather than user traffic |
| Observability defect | The alert's own numbers disagree with an independent measurement of the same thing; the alarm fires while service metrics are healthy |
| Analytics correctness defect | A derived dimension disagrees with the entity it claims to describe |
| Dedup failure | Distinct identifiers, identical assets — hash the bytes, not the URL |
| Authorization defect | Symptom is both user-specific and surface-specific; peers on the same surface are unaffected |
| Client / product defect | Backend telemetry is normal and the symptom is banded by client version or build |
| Capacity near-miss | Nothing is broken; the only signal is consumption against a documented quota |

The Observability defect row is expanded in the next section.

A calibration note: symptoms that originate from a monitor's output or from a prevailing worry are
the ones that most often turn out not to be real, while content a human actually observed usually is.
This is a reason to confirm the signal independently in both directions, not a reason to dismiss
either.

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
- **Unroutable** — if you cannot tell prod from staging at a glance, fix that first.

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
- No queryable deployment history to line a timeline up against.
- No status code or extraction-probability stored alongside cached hydration results, so a stored row
  does not imply a successful extraction.

## Invariants worth asserting

Hard equalities and floors rather than statistical thresholds, so they fire with near-zero false
positives. When you finish an investigation, propose the one that fits — or add a new one shaped like
these.

| Invariant | Class it detects |
|---|---|
| Articles/hour > 0 per surface **per source**, judged against that surface's own mix | Upstream unavailability; capacity cliff |
| Active items per fixed-size section == configured N | Silent validation rejection; model failure |
| `items_written == items_received` per job invocation | Silent validation rejection |
| Input-validation reject rate == 0 | Silent validation rejection |
| Newest active item age per section within that surface's assembly interval | Model or routing failure; capacity cliff |
| Vendor non-2xx share per domain below a floor | Upstream unavailability; extraction defect |
| Content-shape ratio per domain within bounds (empty-body %, title length) | Extraction-quality defect |
| Per-domain funnel positive at every stage: discovered → hydrated → approved | Upstream unavailability; filter misconfiguration |
| Canonical domain ∈ approved list | Data-model / identity defect |
| Per-class model recall above a floor before publish | Model or routing failure |
| Row-count delta on key tables within bounds | Unscoped bulk operation |
| Every bound parameter in shared SQL supplied by every caller | Analytics correctness |
| Consumption below documented quota, alerting on the ratio | Capacity near-miss |
| The alarm itself emits a value at zero | Observability defect |

Stratify anything you assert. A rule that holds globally can be false for one surface, because the
surfaces do not all draw content the same way.
