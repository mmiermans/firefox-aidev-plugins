---
name: hnt-backend-investigation
description: Investigates Home New Tab backend errors, outages, and data-quality problems in whichever New Tab feature the symptom lands in: content recommendations (Merino, the curated corpus, admin-api, the article crawler, the ML section pipeline, the New Tab data pipelines), Picture of the Day, the daily crossword, or a feature not named here. Confirms the symptom independently, probes competing hypotheses in parallel, stratifies metrics, tries to break its own conclusion, then writes an evidence-backed FINDINGS doc with a root cause and quantified impact. Use when a Sentry alert fires, an editor reports something broken, or a New Tab feature looks empty, wrong, or stale — recommendations, the picture of the day, the puzzle. For backend investigation, not in-tree browser/extensions/newtab frontend work.
---

# New Tab (HNT) Backend Investigation

Diagnose a problem in the backend services behind Firefox Home New Tab. The page is assembled from
features that are served separately and fail separately — content recommendations (Merino, the
curated corpus, admin-api, the article crawler, the ML section pipeline, the data pipelines),
Picture of the Day, the daily crossword — so establish which one the symptom belongs to before
probing. A New Tab feature with no notes in this skill is still yours to diagnose: work it the same
way, from the Merino provider and config that serve it. Not for in-tree `browser/extensions/newtab`
frontend bugs, and not for Merino's non-New-Tab consumers such as Firefox Suggest.

Your product is a **diagnosis**: a root cause, the evidence for it, and the impact quantified. Two
populations can be affected, and every finding should say which: **Firefox New Tab clients**
(everyone the affected feature reaches) and, where the feature has one, the **editors/curators**
working through curation-admin-tools and admin-api.

Access and traps, per system: [references/data-sources.md](references/data-sources.md)
Failure mechanisms, invariants, falsification moves: [references/failure-modes.md](references/failure-modes.md)

## How to think about this

**Confirm the symptom before explaining it.** Independently reproduce or measure the reported
problem first, whatever its source. An alert can be miscalibrated, mis-scoped, or watching the
wrong layer, and a description passed through two people can drift. Until you have seen the
symptom yourself in data you pulled, you do not know what you are diagnosing.

**Form the hypothesis before you run the check.** Write down what you expect to see, and what you
would expect to see if the hypothesis were false. A query designed to confirm and a query designed
to discriminate look similar and are not. The characteristic failure in this domain is a correct
query paired with a wrong inference, stated confidently.

**Work several lines at once.** When more than one explanation is plausible, probe them in parallel
rather than following the most attractive one to its end. Breadth is cheap in the unmetered planes —
Sentry, repo reads, CloudWatch alarm history, one live Merino request — and tunnel vision is the
expensive failure. It is not free in BigQuery, Zyte, or CloudWatch Logs Insights: dry-run, narrow the
columns, and narrow the window before you fan out. Keep a list of every line you considered and record which you dropped and why.

**Prefer measuring to reasoning.** Reproduce the request. Run the model on real inputs. Count the
actual rows. Read the code path instead of assuming its behaviour. A plausible mechanism becomes a
number, and only numbers survive review.

Write like the evidence is asymmetric, because it usually is. Avoid "always", "never", "in every
case"; say what each count is a count of, and whether it is a floor, a ceiling, or a sample.

## Don't stall on the developer

Every pause costs wall-clock, because the developer may not be watching. Default to proceeding:
pick the sensible option, say which one you picked, and keep going. Ask a blocking question only
when proceeding on any assumption would waste the whole investigation.

When a source you cannot reach looks promising, raise it **once** as a pending task carrying the
whole errand in its own text, prefixed `User action:` so it reads as theirs rather than yours — e.g.
`User action: connect to VPN so corpus MySQL is reachable — settles whether the DE items exist at
all`. Give the exact command, not a description of the problem; the "Access requests" section of
`references/data-sources.md` has the command for each gated source and how to word the ask. If the
answer lives in a dashboard or console you have no credential for, do not ask for access at all —
name the view, the filter, and the single number or shape you need, and ask them to read it back.

**Then keep investigating the other hypotheses immediately. Do not wait.**

Restate it exactly once more — whichever comes first: the unblocked lines run out, or you are three
probes into a line you had already judged weaker than the blocked one. That is the second and last
ask, and it does not apply if they have already answered. If there is still no response, finish steps 7 and 8 with `Status: blocked`,
name the one unblock under "Could not measure", and leave the access task pending.

Stopping outright is a last resort: only when the blocked source is the only thing that can settle
the question **and** it cannot be reduced to a question they can answer for you — a dashboard,
console, or explore is almost never a real stop. A request that quietly disappears is worse than one
that was never made.

## Workspace

Each investigation gets its own subdirectory inside one root directory that the developer keeps their
investigations in. Do not assume where that root is. Establish it in this order:

1. **Look in user memory.** `~/.claude/CLAUDE.md` is where the root is recorded, because it is the
   only memory scope that applies across every repository you might be launched from. If it names a
   root, use it and do not ask.
2. **Otherwise ask, once, before your first write.** Where the developer keeps investigations cannot
   be derived, and guessing puts files somewhere they did not choose. Ask it as a single question at
   the start, and run the step-1 probes while you wait rather than idling.
3. **Then record it.** Append one line to `~/.claude/CLAUDE.md`, creating that file if it does not
   exist: `Investigations live in <path> (one subdirectory per investigation).` Tell the developer
   you saved it, so they know they will not be asked again. Auto memory is the wrong home for this:
   it is keyed to the repository you were launched from, and investigations get started from
   whichever service repo is to hand.

Then create this investigation's subdirectory inside that root. If the root already holds
investigations, follow the naming they use; otherwise `<mon><DD>-<slug>`, e.g.
`jul29-empty-de-sections`. Say which directory you created. Reuse an existing one only when its
FINDINGS.md is about the same symptom.

- `FINDINGS.md` — one living document. Create it from the step-7 skeleton the moment the directory
  exists, and in any case before you write a conclusion down; steps 2 through 6 write into it as they
  go, so step 7 is a final pass over a document that already exists. If you are still waiting on the
  root, run the step-1 probes anyway and hold their output until you have somewhere to put it. Rewrite
  it in place, not as `FINDINGS-v2.md`.
- `queries/` and `results/` — every query as a file, its output alongside under the same basename.
- `api_responses/` — raw JSON from live calls. Save them even when they look boring; metered APIs
  cost money to re-hit and the data may be gone tomorrow.
- Aggregate before saving. Counts, rates, and shapes answer nearly every question these
  investigations ask; this directory sits outside any repo and gets linked into tickets, so prefer a
  distribution over a dump of rows carrying editor identities or user data.
- Rewrite only the FINDINGS.md you wrote and delete only files this session created — anything else
  in the directory belongs to another investigation.
- Python work goes in an isolated venv inside this directory: `uv venv` if `uv` is installed,
  otherwise `python3 -m venv`.

## Step 1 — pin down the report

Before your first probe, read [references/data-sources.md](references/data-sources.md). It carries the
traps that silently return wrong answers, and several of them look like an outage.

Establish these five facts from the data. The alert or issue carries the symptom and its volume,
stratifying gives you the surface and locale, and a live request tells you whether it is still
happening.

| Fact | Why it matters |
|---|---|
| Exact symptom, verbatim, and which feature | "Empty section" and "wrong items in section" have disjoint causes |
| Who noticed, and how | Alert / editor report / spotted by hand — sets what evidence exists |
| First and last seen, **with timezone** | Anchors the timeline; reports usually arrive late |
| The stratum — surface, locale, section, client version, or for a daily artifact the date | The stratum is very often the diagnosis |
| Still happening right now? | Live reproduction vs. historical forensics |

**If it is still happening, capture the perishable evidence first** — a live request for the feature,
current logs, and whatever counts or artifact it publishes — and save the raw responses under
`api_responses/`. Build the step-2 prior while those probes are in flight; history
keeps, live signal does not.

**An error-rate signal is not yet a symptom.** Measure the erroring stage's *output* to fix the
blast radius — items per section and per-surface freshness for assembly, the newest object under a
dated prefix for a publish job, a live Merino request for anything client-facing — so you can say
whether Firefox clients are affected at all. An
error floor that never reaches the output is a not-incident; errors flat with output at zero is
worse than the alert says. When the report is a Sentry alert, resolve it to a concrete error and a
volume, and decompose the issue by error message before trusting its title or its trend — the
alert-quality checks are in `references/failure-modes.md`.

Sentry is the plane most of this reads from, so if the `mcp__sentry__` tools are absent, say so and
raise it like any other blocked source, then carry on with the planes you do have.

**If you cannot reproduce it, that is a result, not a blocker.** Record the exact attempt — surface,
locale, time, request, what you saw instead — then switch the question to why the reporter saw it
and you do not: a stratum you did not hit, a window that has closed, a cache, a client version, or a
monitor measuring something other than what it claims. Classify as `not-incident` only after that
second question has an answer; unconfirmed is not refuted.

A report relayed from an editor is ambiguous between what editors see in curation-admin-tools and
what clients see on the surface. Probe both in parallel and state which population you confirmed.

The moment you confirm client-visible impact that is still happening, state it in one line — what,
how big, since when — and keep investigating. If that deserves a heads-up in `#hnt-dev-be-alerts`,
follow the posting rule at the end of step 8.

## Step 2 — establish what "normal" is

You do not know what is currently normal in this system, so you have no basis on which to reject
your own inference. Build that prior yourself before asking for it — most of it is in reach:

- **What the serving path returns now** — a live request for the feature; for recommendations, the
  surfaces present and not disabled in the section data.
- **What the producer last wrote** — per-surface, per-source crawl volume over the last day for
  recommendations, the newest object under the dated bucket prefix for a publish job. Producer and
  serving sets differ, and a crawled surface with no sections is not automatically a bug.
- **What shipped recently** — recent commits in the service repos, releases in Sentry, timestamps on
  model or config artifacts.
- **Whether this is normally noisy or seasonal** — a trailing profile of the same metric, by day of
  week. Do not ask; compute it.
- **Whether it has happened before** — search Sentry for the same signature and the service repos'
  GitHub issues. Where one of those is unreachable, record it as unchecked rather than empty; "nothing
  similar has fired" is a claim about a source you actually read.

What is left is genuinely in the developer's head: anything in flight that a repo or a dashboard
would not show, and whether a postmortem or incident register already covers this area. Raise that
as one short note and carry on without waiting; an existing write-up can answer in a paragraph what
costs hours to reconstruct.

Record what you derive and what they tell you in FINDINGS.md, keeping the two apart. When a later
query contradicts the prior, treat your own inference as the suspect first, and say so.

## Step 3 — form hypotheses from the data, then probe them in parallel

Hypotheses come from what you have already measured, not from a catalogue. Start where step 1 put
you: the stage whose output is wrong, the stratum that is affected, the moment it changed. Ask what
could produce exactly that, follow the data one hop upstream, and let each result generate the next
question. A hypothesis you cannot tie to something you have observed is a guess competing for the
same probe budget as one you can.

Work the live ones in parallel rather than serially. Before firing, write beside each hypothesis the
result that would kill it. Then issue the probes as independent tool calls in a single batch,
backgrounding anything slow and handing a line that needs several dependent steps to a subagent so
the batch still returns together. Cap the first wave at four or five cheap, independent probes, and
hold metered calls and large scans for the second wave. Name each `queries/` and `results/` pair
after its hypothesis, so a result cannot be attributed to the wrong line.

Then look at what came back and do it again. Each round should either kill a line or sharpen the next
question; when a round kills everything, the data has moved you upstream rather than left you stuck.
If you are down to a single surviving hypothesis, that is the moment to check the failure classes and
silent-failure mechanisms in `references/failure-modes.md` against what you have measured, as a guard
against having narrowed too early.

Two moves consistently break cases open: **leave the tool you started in** — when a source stops
yielding, measure the same thing somewhere else, since the failure is often invisible in the plane
you began with — and **compare against sibling strata**, which localises a fault faster than reading
code. Both are expanded in `references/failure-modes.md`.

## Step 4 — build the timeline

An explicit `timestamp (UTC) | observation | source` table.

- Normalise every timestamp to UTC and label it. Timezone mismatch is a common source of phantom
  gaps, and a subtraction error can manufacture an outage that never happened.
- Extend the window to weeks, not hours — these failures are frequently much older than the report.
  Widen freely on the cheap planes; on a billed plane, widen only after a narrow window has shown
  you something.
- Distinguish *first occurrence* from *first noticed*, and state both.
- Line the window up against the deploys and config changes from step 2.
- Treat retention limits as limits: a "first seen" date can be the edge of a retention window rather
  than onset. See `references/data-sources.md`.
- **The scheduling layer is not in UTC.** Scheduled dates and the assembly crons run in each surface's
  own timezone, so a UTC comparison invents a one-day gap for part of every day on any surface offset
  from UTC — worst for the Americas, and `en-US` is the largest surface. Convert per surface, and say
  which timezone you used.

## Step 5 — stratify before concluding anything

An aggregate that looks fine is the normal way these problems hide. Slice every metric by the
dimensions the feature actually has — for recommendations **domain, locale, region, surface, section,
experiment branch, and client/addon version**, for a once-a-day global artifact barely more than the
date — and look for a single stratum at zero or down sharply against its own trailing median rather
than against yesterday. When the report says "some users" with no stratum attached, treat experiment branch,
region, and rollout state as the first cuts — some surfaces are reachable only through experiment
enrolment, so an enrolled/unenrolled split is invisible to a locale slice.

Strata are not interchangeable, and the volumes are strongly seasonal; both traps, and the
per-surface source mix, are detailed in `references/data-sources.md`. Compare each stratum against
its own history.

**Liveness is not health.** "The job ran successfully" is compatible with total data loss in every
one of these pipelines. Count what came out, compare it against what went in, and never accept a
green run as evidence.

If the measurements exonerate the backend, stop at the service boundary. Read far enough to name the
owning team and the contract that is being broken, then hand it over rather than continuing into a
system this skill does not cover. A second, unrelated anomaly you trip over on the way is a one-line
note plus a task, not a second investigation.

## Step 6 — try to break your own conclusion

Before writing anything down as fact, attack it:

- Check the boring explanations first: timezone; a filter in your own query; a `LIMIT` silently
  truncating; an unrepresentative code path; sampling; a column that does not mean what its name
  says; a partially-launched feature.
- Then reach for the falsification moves in `references/failure-modes.md` — the code path, exact
  enumeration, repetition, the real limit, the config's history — picking the ones that would break
  your specific mechanism.
- **Measure the effect, not just the mechanism.** A guard that provably runs is not evidence that
  the outcome is correct; verify the outcome separately.
- Cross-check against a second, independent source. One source is a hypothesis.
- Re-read the step-2 prior. If your conclusion implies that something the prior says is working is
  broken, re-check your own measurement first — a filter in your query, a stratum mismatch, or a
  wrong surface identifier is the likelier explanation. If the measurement survives that re-check,
  the measurement wins: state the contradiction explicitly in FINDINGS.md, note it to the developer
  in one line, and keep going.

Mark every claim **verified**, **inferred**, or **refuted**. Keep the refuted ones in the document;
they stop the next person re-running them.

## Step 7 — write FINDINGS.md

```markdown
# <feature>: <symptom> — investigation

**Status:** investigating | root cause identified | blocked
**Incident type:** outage | degradation | data-quality defect | user-facing bug | near-miss | not-incident
**Severity:** critical | high | medium | low — anchored to the magnitude below, not chosen by feel
**Affected:** Firefox clients | editors | both — with a magnitude, not an adjective

## Summary
Three sentences: what broke, since when, what the user-visible effect is.

## Timeline (UTC)
| Time | Observation | Source |

## Prior
**Derived** — the feature's own volumes or published artifacts, recent deploys and model artifacts,
trailing profile, prior occurrences. Each with the query or link.
**Reported** — verbatim answers from the developer, or `asked <time UTC>, no answer received`.

## Evidence
Numbered findings. Each: the claim, the query or command, the result, and
verified/inferred/refuted. Reference saved files by path — every number reproducible from this
directory.

## Root cause
The mechanism, with the code path or config that produces it. "Not established" when it is not.

## Hypotheses considered and dropped
Every line you enumerated, and what ruled each one out — or why it was not pursued.

## Impact quantified
Rows, requests, users, hours, locales. A number, or an explicit "unquantified because X".

## Proposed fix
The change as a diff in this document, at file-and-line where you can get there, produced by
reading the clones.

## Could not measure
Any telemetry plane you could not reach, and the access that would unblock it.
```

## Step 8 — close the loop

- **Report what you could not reach.** Missing dashboard, IAM, token, or VPN access is a finding,
  not an inconvenience to route around. Leave any unresolved access task on the list rather than
  clearing it.
- **Link prior art** — older investigation, ticket, postmortem, or a related error already known.
- **Leave the follow-up as a task**, owned by whoever will act: confirm the fix shipped and the
  metric actually recovered. State that you have done so rather than asking whether to.
- Hand back a two-line verdict plus the FINDINGS.md path. Do not paste the document into chat.

### Posting to `#hnt-dev-be-alerts`

Updates worth sharing go to `#hnt-dev-be-alerts`. If the investigation started from an alert or message
posted there in the last day or two, find that one message and reply **in its thread**, so the
diagnosis stays attached to the alert people already saw. Otherwise start a new thread.

**Ask before every single message, showing the exact text and where it will go.** Approval for one
post is not approval for the next, and this is a shared channel colleagues act on. If approval does
not come, keep the draft in FINDINGS.md and carry on; an unposted update is not a reason to stop.
