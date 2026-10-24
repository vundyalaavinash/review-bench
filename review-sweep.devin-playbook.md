# Review Sweep

Macro: `!review-sweep`

Run a probe-by-probe review of a pull request, incident postmortem, design doc/RFC, or one-pager. Verify claims by executing against the repository instead of reading and guessing. Produce one review memo in a fixed format. Never change the code under review.

## Required from User

- The artifact: a GitHub PR link (default), or a link/pasted text for an incident, design doc, RFC, or one-pager.
- Artifact type, only if it is not a PR: `incident`, `design`, or `onepager`. Infer it when obvious.
- Depth: `core` or `full`. Default `full` for any change touching persistence, Kafka, contracts, or concurrency; `core` otherwise.
- Whether to post the memo as a PR review comment. Default: do not post; return it in the session.

## Procedure

1. Read only the PR title and description (or the doc's title and first paragraph). Write three expected risks and the one-way door (schema change, published event contract, deletion, identifier choice) before opening the diff.
2. Clone the repository and check out the PR head. Build it and run the existing test suite once to establish a baseline. Record pass/fail counts.
3. Read the full diff, then read every file the diff calls into one level deep.
4. Select the probe set for the artifact type and depth from Specifications. Run every probe in order.
5. For each probe, run the verification listed with it where one exists. Record exactly one result: `finding`, `cleared` (with evidence as file:line, command output, or doc section), or `skipped` (not applicable, with a five-word reason).
6. For the "tests that lie" probe, pick the most important changed logic line, comment it out locally, run only the relevant test class, record whether anything failed, then revert. Confirm `git status` is clean afterwards.
7. Write each finding in four parts: observation stated as fact, concrete consequence, severity (`blocking`, `should-fix`, `nit`), and a path forward or "no suggestion".
8. Rank blockers. If there are more than three, merge findings that share a cause or demote one to should-fix. If more than three remain, set the verdict to "not reviewable as one change; split it".
9. Draft the "What I am accepting" line naming the largest risk you are letting through and why. Mark it `PROPOSED — reviewer to confirm`.
10. Compare the step 1 predictions against the findings: list hits, misses, and risks found only by reading.
11. Assemble the memo in the exact format under Specifications, followed by the JSON record.
12. Post the memo as a single PR review comment only if the user asked for it, using the "Comment" review type. Otherwise return it in the session.

## Specifications

Postconditions:

- One memo exists in the format below, and every probe in the selected set appears in exactly one of: WHAT I CHECKED, a finding, or the skipped list.
- No more than three items appear under WHAT MUST CHANGE.
- Every `cleared` reason cites evidence. No reason reads "looks fine", "LGTM", or similar.
- The working tree is unchanged; no commits, pushes, or branch changes were made.
- The JSON record at the end parses and uses the probe IDs below exactly.

### Memo format

```
Review — <PR title or doc name> (<Code|Incident|Design|One-pager>, <core|full> sweep)

WHAT I CHECKED
- <probe heading> — <evidence>

WHAT MUST CHANGE
1. <observation>
   Consequence: <consequence>
   [<probe heading>]

SHOULD FIX — can land after merge
- <what> — <consequence>

NITS — ignore all of these if you like
- <what>

WHAT I AM ACCEPTING
- PROPOSED — reviewer to confirm: <risk and why>

SKIPPED
- <probe heading> — <reason>

--- pre-read prediction ---
<three risks>
One-way door: <door>
Predictions: <n> hit, <n> missed, <n> found only by reading.
Baseline tests: <pass>/<total>. Mutation check: <test class> <failed|did not fail> when <line> was removed.
```

### JSON record

Append after the memo, inside a ```json fence, so the review can be logged to Review Bench:

```json
{
  "type": "code|incident|design|onepager",
  "depth": "core|full",
  "title": "<title>",
  "status": "done",
  "probeIds": ["<every probe id run, in order>"],
  "results": { "<probeId>": { "state": "cleared|finding|skipped", "reason": "<text>" } },
  "findings": [ { "probeId": "<id>", "probe": "<heading>", "what": "<observation>", "why": "<consequence>", "severity": "blocking|should-fix|nit" } ],
  "prediction": { "risks": "<text>", "door": "<text>" },
  "accepting": "<proposed line>"
}
```

### Probe sets

`core` probes run at both depths. `full` probes run only at full depth. IDs must not change.

**code** — services are Java 21 / Spring Boot 3.x / Kafka.

| ID | Depth | Probe | Verification |
|---|---|---|---|
| c-pred | core | Predict before you read | Step 1 output |
| c-inv | core | Name the invariant as a query that returns rows only when broken; does anything in prod run it? | Search repo for a reconciliation job, alert rule, or scheduled check covering it |
| c-kill | core | Kill points: every DB write, outbound call, published event, cache write; crash between each pair | List them in order with file:line |
| c-idem | core | Idempotency key, retention window, and whether the key is written in the same transaction as the effect | Trace the key to its storage; check `@Transactional` boundaries |
| c-lost | full | Read whose result decides a later write of the same row; is the write conditional? | Look for `WHERE version =` / `@Version` / `SELECT ... FOR UPDATE` |
| c-iso | full | Does correctness need nothing to change between statements; does the isolation level guarantee it? | `grep -rn "isolation" src/` and datasource config; default Postgres is READ COMMITTED |
| c-dual | core | Two durable systems mutated in one operation; where is the outbox or reconciliation? | Find DB write and `KafkaTemplate.send` in the same method or call chain |
| c-deliv | core | Offset commit position relative to the side effect; does it cover exactly the processed records? | `grep -rn "enable.auto.commit\|AckMode\|acknowledge()"` |
| c-part | full | Business ordering requirement; does the partition key deliver it; hottest key share? | Find the key passed to `send(topic, key, value)` |
| c-rebal | full | p99 of every call in the poll loop × `max.poll.records` vs `max.poll.interval.ms` | `grep -rn "max.poll"`; list downstream calls inside the listener |
| c-poison | full | Where a never-succeeding message goes; DLQ owner; effect of skipping on per-key ordering | Find `DefaultErrorHandler`, `DeadLetterPublishingRecoverer`, retry config |
| c-schema | core | Full compatibility for independent deploys; any field whose meaning changes while its type stays | Diff Avro/JSON schema and DTOs |
| c-err | core | Retryable / terminal / ambiguous classification; is ambiguous resolved, not assumed? | Read every changed `catch` block |
| c-time | core | Every hop's timeout incl. pool acquisition; sum under caller's; any inversion | `grep -rn "timeout\|connection-timeout\|connectTimeout\|readTimeout"` in config and clients |
| c-retry | full | Retry multiplication across layers; jitter, budget, breaker | Find `@Retryable`, Resilience4j config, client retry settings |
| c-bound | full | Pools, queues, caches, per-request collections with no maximum | Executor, Hikari, Caffeine/Redis TTL, in-memory lists |
| c-cases | core | null, empty, zero, one, max, duplicate, out of order, concurrent, replayed | Check each against changed inputs |
| c-obs | core | Can you tell if it is happening, how often, and to which entity? Was observability removed? | Look for metrics, log lines with entity IDs, swallowed exceptions |
| c-test | full | Does deleting one line of changed logic fail any test? Is any failure path tested? | Procedure step 6 |

**incident**

| ID | Depth | Probe |
|---|---|---|
| i-time | core | Timeline is sourced, separates observation from inference, and starts at the change rather than the alert |
| i-int | core | Compute begin, detect, engage, mitigate intervals; name the largest |
| i-found | core | Found by a signal or a person? A person means the monitoring gap outranks the fault |
| i-fact | core | Trigger, latent weakness, missing control; which factor generalises |
| i-count | core | Replace every "should have" with why the action was reasonable at the time |
| i-ctrl | core | Each control worked slowly, failed, was bypassed, or never existed |
| i-blast | full | What limited the damage; which boundary was missing |
| i-imp | full | Impact from primary data, in business units, including degradation and downstream repair |
| i-act | core | Action items specific, owned by a named person, dated, permanent rather than procedural |
| i-gen | core | Two other places with the same uncontrolled risk right now (search the repos) |

**design**

| ID | Depth | Probe |
|---|---|---|
| d-dec | core | Load-bearing decisions and the condition that would reverse each |
| d-prob | core | Problem has evidence with a number, an owner, problem framing, and a reason for now |
| d-num | core | Peak throughput, p99, availability + window, durability, consistency, retention, growth |
| d-10x | core | Which numbers break the design at 10x |
| d-alt | core | Alternatives on one shared axis, including do-nothing and buy/adopt |
| d-fail | core | Kill each component; find the undrawn shared dependency |
| d-data | core | One-way doors in the data model |
| d-cons | full | Consistency promised vs provided; who notices the gap |
| d-cap | core | Capacity recomputed independently; first binding constraint |
| d-cost | full | Cost per business unit now and at 10x, incl. egress, storage growth, engineer time |
| d-mig | core | Migration incremental, reversible, observable, bounded; point of no return named |
| d-back | full | Backfill validation, interference, resumability, continuous divergence rate |
| d-ops | full | New alerts, runbooks, manual steps; total-failure rebuild time |
| d-sec | full | Trust boundaries; credential blast radius; where sensitive data comes to rest |
| d-dep | full | Each dependency's availability, p99, owner, exit cost |
| d-seq | core | Phase one stands alone; riskiest unknown first; scope to cut |

**onepager**

| ID | Depth | Probe |
|---|---|---|
| o-ask | core | Decision, ask, and date findable in thirty seconds; can this reader grant it |
| o-open | core | Opening paragraph: situation with a number, complication, proposal, ask |
| o-read | core | The reader's three objections answered before they arrive |
| o-num | core | Central number's provenance, 2x sensitivity, honest precision |
| o-dec | core | Ask is specific, addressed, bounded, with a smaller alternative |
| o-risk | core | Biggest risk named first with mitigation and trigger |
| o-opt | full | Minimal, recommended, ambitious options, all choosable, with a recommendation |
| o-time | full | Timeline decomposed, dependencies confirmed, realistic availability, cut list |
| o-met | core | How to hit the metric without the goal; counter-metric; baseline |
| o-struct | full | First sentences form the argument; conclusion first |
| o-cut | full | Sections that don't change the decision; anything load-bearing buried |

## Advice

- Verification beats reading. If a config value decides a probe, find the value; do not assume framework defaults without saying so.
- When the PR does not contain what a probe needs (the isolation level, a timeout, a consumer config in another repo), record a should-fix finding: "Unstated: <what>". Do not guess.
- A blocking finding is one with wide blast radius or an irreversible effect: data corruption, lost messages, a published contract change, a deletion. Style is never blocking.
- Kafka defaults worth knowing: Spring Kafka's default AckMode is BATCH; `max.poll.interval.ms` defaults to 300000; `max.poll.records` defaults to 500. A dual write between Postgres and Kafka with no outbox is always at least should-fix.
- Keep the memo short. Findings get one or two lines each. The WHAT I CHECKED list carries the evidence.
- For non-code artifacts, still use the repositories: search them to verify claims in an incident or design doc when the doc names services.

## Forbidden Actions

- Do not push commits, create branches on the remote, or modify the PR branch.
- Do not approve, request changes on, or merge the PR. Post only a "Comment" review, and only when asked.
- Do not leave the local mutation from step 6 in place.
- Do not list more than three items under WHAT MUST CHANGE.
- Do not write a `cleared` result without evidence.
- Do not change probe IDs or drop probes from the selected set; skipped probes go in SKIPPED.
- Do not finalize the WHAT I AM ACCEPTING line; it is always marked PROPOSED.
