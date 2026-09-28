# 7. Observability and experimentation

Companion notes for **Chapter 7** of *Designing AI Systems* (MEAP, Suhas
Suresha and Dewang Sultania; Manning). This MEAP's printed contents page is
**single-level** (nine chapter titles). The headings below follow the
chapter's actual topics.

This chapter is how the platform answers **was the answer any good**,
not only **did the box stay up**. Model, Session, Data, Tools, and
Guardrails already emit signals. Without a place to join them — traces,
scores, dollars, experiments — you get six log streams and a finance
ticket. Skip this chapter and every prompt change is a hope, every
invoice is a mystery, and "quality" lives in Slack anecdotes.

The folder stays on the **Observability Service** and the
**Experimentation Service**: the fleet-wide data model, contracts, and
improvement loop. Agent-local eval loops, critics, and lab notebooks
belong elsewhere; here the platform stores scores and joins them to
cost.

## The mental model

```
  one user request
        |
        v
  +---------------- TRACE (one hop through the platform) -------------+
  |  span: sessions.get          history, tokens in window            |
  |  span: data.search           hits, relevance                      |
  |  span: guardrails.input      policy, action                       |
  |  GENERATION: models.chat     model, tokens, $, TTFT               |
  |  span: tools.execute         (if the loop acted)                  |
  |  span: guardrails.output                                          |
  |  span: sessions.append                                            |
  |  scores ........ attached later (heuristic / judge / human)       |
  +-------------------------------------------------------------------+
        |                                      ^
        v                                      |
  Observability Service                  Experimentation Service
  "what happened, what did              "what should we change,
   it cost, was it good"                 did the change help"
        |                                      ^
        +---- low-scoring traces, datasets ----+
              offline eval, A/B, annotate
```

The one sentence to remember: **quality and cost must share a
trace_id**, or you will optimize the dashboard that is easiest to
graph.

Sprawl here looks like four log formats, token counts that never leave
the Model Service, and a prompt "v7" that nobody can A/B. Offline vs
online eval, human queues, and A/B hygiene are product choices; this
chapter is the **services** that implement them.

## Why AI systems need specialized observability

Traditional monitoring is not wrong. Uptime, latency, error rate, and
saturation still matter. An AI workflow that 500s is still a
production incident. The gap is what "healthy" *means*.

Sarah ships a new patient-intake prompt. Dashboards stay green:
p95 is the same, error rate is the same, the Model Service is up.
Patients start saying the assistant is less helpful. It asks them to
repeat facts already in the session. It takes more turns. None of
that is an HTTP error. The system is running. It is **working
poorly**.

That is the specialized job: measure whether the *output* is useful,
grounded, safe, and worth the tokens — and join that measurement to
the same request that hit Session, Data, Guardrails, and Model.

### From infrastructure health to output quality

Deterministic services have a convenient lie: if the process is up
and the status is 200, the output is almost certainly correct. Same
code, same input, same bytes.

Generative systems break the lie in two directions.

- **The same inputs can yield different outputs.** Temperature,
  routing, cache misses, and provider drift all move the text. A
  "successful" generation can be useless.
- **The inputs themselves are assembled, not stored.** Which chunks
  Data returned, how much history Session could fit, which model
  survived routing and fallback, which guardrail rewrote a sentence:
  quality is a *pipeline* property. A healthy Model Service can still
  answer from a bad retrieval.

So you still keep SLOs for latency and errors. You add dimensions
that a web monolith never needed: tokens, model identity, retrieval
hit quality, safety actions, **response quality scores**, and **cost
per workflow**. Figure the chapter wants you to hold: traditional
metrics are the inner ring; AI-specific signals wrap them. Do not
throw away the inner ring. Do not stop there.

A slow request can be a *great* answer (long tool loop, good
research). A fast 200 can be a hallucination. If your on-call only
pages on 5xx, you will never page on "patients repeating themselves."

### Cross-service correlation

One patient question is not one service call. A typical hop:

1. Gateway authenticates and **starts a trace**.
2. Session loads history (and maybe memories).
3. Data searches an index.
4. Guardrails inspect the input.
5. Model generates (maybe several times if tools loop).
6. Guardrails inspect the output (and tool args in between).
7. Session stores the exchange.

If that request took four seconds instead of one, each service has a
local timing. Without a **shared trace_id** on every hop, debugging
is "align timestamps and pray the clocks agree." Distributed tracing
is not a nice-to-have visualization. It is the only honest answer to
"where did the time and the tokens go."

Correlation is also how you stop false blame. "The model is slow"
dies when the waterfall shows Data at 3.2s and Model at 400ms. "The
model hallucinated" dies when the generation span shows empty
retrieval and a truncated session.

Propagate context on every internal call (the book uses gRPC
metadata: trace id, span id, workflow id). If a new service cannot
join the tree, it is an observability hole, not a "lean
microservice."

### Cost as the bridge to experimentation

Classical app cost tracks CPU and RAM. It is dull and similar across
requests. GenAI cost tracks **tokens**, which swing with history
length, retrieved context, model choice, retries, and cache hits.
The Model Service already records per-request dollars. That is how
you investigate the one call that cost fifty cents instead of two.

Finance and engineering also need the layer *above* the request:
spend by **team**, by **workflow**, by **model**, trends, and
projections. Observability is the aggregator. Model Service remains
the source of the leaf numbers.

Knowing spend is not knowing value. A workflow at $1,400/month might
be excellent. Another at $400 might be burning tokens on padded,
unhelpful answers. **Cost without a score cannot tell you which.**
Scores without cost cannot tell you whether a quality win is
affordable. The bridge to experimentation is exactly that join: a
trace that carries dollars *and* quality, so a prompt change can be
judged on both.

The mystery invoice becomes a drill-down: total → team → workflow →
model → the generation that actually happened.

## The observability data model

Do not start from vendor product names. Start from **questions**.
The primitives fall out of what you must reconstruct.

### Derive the model from questions

**What happened on this user request?** You need a **trace**: the
whole journey, with input, output, duration, total cost, workflow
id, user id, tags. Inside it, **spans**: units of work (session
load, search, a guardrail pass). Spans nest. The waterfall is the
nesting plus duration.

**Which of those units was an LLM call?** A generation is a span
with extra fields the rest of the platform should not fake: model
name, prompt/completion tokens, cost, time to first token, cache
hit, provider, finish reason. Index generations as first-class
objects so "show me GPT-4o calls over $0.10" is not a JSON grep.

**Which conversation was this turn part of?** A **session** groups
traces the way the Session Service groups messages. Session-level
views answer "this user had a bad afternoon," which a single trace
cannot.

**Was the result any good?** A **score** attaches a named metric
(helpfulness, completeness, retrieval relevance, safety) to a
trace, a span, or a generation, with a source (heuristic, judge,
human). Scores often arrive **after** the user already has the
reply. That asynchrony is a feature: do not hold the HTTP response
for a second model call.

```
  SESSION  (conversation, many requests)
     |
     +-- TRACE  (one user request)
            |
            +-- SPAN  (session.get)
            +-- SPAN  (data.search)
            |      +-- logs: per-hit scores, filters
            +-- GENERATION  (models.chat)   <-- specialized span
            |      +-- SCORE  (grounding)     may attach here
            +-- SPAN  (guardrails.output)
            +-- SCORE  (helpfulness)        may attach on the trace
```

If you only store logs, you cannot draw the waterfall. If you only
store spans, you cannot explain *why* a guardrail almost fired. If
you only store metrics, you cannot debug one patient.

### How the primitives connect: one request

Walk a concrete question: "Can I bring medical records digitally or
do I need paper copies?"

- Gateway creates `trace_id`, binds it to the patient's session.
- Session span: twelve prior messages, ~45ms, ~2,100 tokens of
  history charged against the window.
- Data span: hybrid search, k hits, latency, index name. Child logs
  can list document ids; the span holds the aggregate.
- Guardrails input span: allowed / blocked / transformed.
- **Generation:** the Model Service call — the thick box in the
  book's figure. Tokens, cost, TTFT, model after routing.
- Guardrails output span.
- Session append span.
- **Scores** land later: maybe an automated key-element check in
  20ms, maybe an LLM judge on a 10% sample, maybe a human next
  Tuesday.

Six spans and one generation is a teaching size. Production traces
grow when tools loop. The shape does not change: one tree, one
`trace_id`, scores as decorations that may lag.

### Logs and metrics complement the request-level model

The five primitives (session, trace, span, generation, score) are
**per request**. Two other kinds of data sit beside them.

**Structured logs** are events *during* a span. The guardrail span
says "35ms, success." The logs say "rule A 0.91 pass, rule B 0.48
near-miss, rule C transform." A cache lookup might emit zero logs.
A messy policy evaluation might emit a dozen. Always include
`trace_id` and `span_id` so an operator can open the span and pull
the events. The span is *what* and *how long*. The logs are *why
this way*.

**Metrics** are aggregates you must not recompute from raw logs at
dashboard time: counters (requests, tokens, dollars) and histograms
(latency, relevance). Labels (provider, model, workflow id, team)
make one metric name answer many cuts. Metrics tell you Tuesday's
p95 moved. They will not show you Maria's four-second trace. That
is why you keep all three.

## The Observability Service contract

Same pattern as Model, Session, Data, Tools: a small gRPC (or
equivalent) surface, SDK in front. The twist is **traffic
direction**. Most platform services are *called for an answer*.
Observability is mostly a **receiver**.

### Walking through the contract

Group operations by storage physics, not by taste.

**Logs** — `IngestLogs` / `QueryLogs`. Individual records, search by
fields.

**Metrics** — `RecordMetrics` / `QueryMetrics`. Numeric time series,
query by window and labels. Do not merge logs and metrics into one
RPC. You would force one store to do two jobs badly.

**Tracing** — `RecordSpan`, `RecordGeneration`, `GetTrace`,
`QueryTraces`. Generation is separate so you can index LLM fields
without stuffing them onto every span. Get is "this id." Query is
"slow traces on workflow X yesterday."

**Scores** — `RecordScore` (and a query). Arrives late. Must attach
to an existing trace/span/generation without rewriting history.

**Cost** — reports and budgets derived from generation records plus
labels (team, workflow). Drill-down, not a second ledger.

**Health** — the service's own live/ready, like every other
platform process. Ironic to skip.

Fourteen operations across those concerns is the book's teaching
count. Your implementation can split further. The invariant is:
ingest paths are cheap and numerous; query paths are richer and
rarer.

### Ingestion vs query

Ingestion is high-throughput and **must not sit on the user path**.
When Model finishes, it publishes asynchronously. If Observability
is down, the user's answer still ships. Buffer locally and retry.
If you make `models.chat` wait on `RecordGeneration`, you have
turned telemetry into a single point of failure — the worst kind,
because it fails *while looking like diligence*.

Corollary: the client in each service batches and flushes on a
timer or size threshold. Flush failure stays in the buffer (up to a
cap). Drop-oldest or shed traces if the cap hits; emit a metric
that you dropped. Never block the assistant.

Query is the opposite shape: operators, dashboards, experiment
pipelines, cost reports. Latency of a second is fine. Consistency
can be "eventual enough that scores appear." Do not design the
ingest API as if it were the query API with extra fields.

Authz differs too. Every replica may ingest. Few principals may
`QueryTraces` with raw prompts. Treat generations as sensitive:
they contain user text and sometimes retrieved documents.

## Structured logging, metrics, and distributed tracing

The contract is verbs. This section is what the records *contain*
and how you debug with them together.

### Structured logging for AI-specific debugging

JSON-with-a-message is not the win. The win is a **shared event
shape** plus AI fields that grep can actually find.

Every event: id, timestamp, service, severity, event type, message,
`trace_id`, `span_id`, workflow id, user id (or a hashed stand-in).
Then two maps: string attributes and numeric attributes. That last
split keeps "tokens=2100" queryable as a number, not a string you
parse in panic.

What goes in the maps is domain-shaped:

- Model: provider, model, cache hit, fallback reason, token counts,
  estimated cost.
- Data: index, hit count, top score, filter keys (not raw chunk
  text unless policy allows).
- Guardrails: policy name, action, confidence, which rule.
- Tools: tool name, version, breaker state, retry count.

Same envelope, different keys. An operator investigating a slow
trace opens the bad span, then queries logs where `span_id` matches.
They should not SSH to four clusters.

Do not log secrets, full credential values, or needless PII. Hash
or drop. Observability that cannot go to the vendor SOC2 review is
not production observability.

### Metrics collection

Name metrics once. The book uses a constants class with a prefix
like `ai.platform.{service}.{metric}`. Chaos is `llm_latency_ms` in
one repo and `model.duration` in another.

Two families:

- **Counters** — requests, tokens, cost, cache hits, guardrail
  blocks. They only go up (per process); the TSDB rates them.
- **Histograms** (or summaries) — latency, TTFT, retrieval scores.
  You care about p50/p95/p99, not the mean of a heavy tail.

Dimensions you will actually slice: `provider`, `model`,
`workflow_id`, `team`, maybe `cache`. Cardinality is a budget. User
id as a label will melt Prometheus. High-cardinality truth belongs
on traces.

Each service already *has* domain structs (`RequestMetrics` from the
Model Service, evaluation records from Tools and Guardrails).
Observability does not reinvent them. It **publishes** them into
the standard names.

### Distributed tracing and the debug workflow

Logs: events in a service. Metrics: shapes over thousands of
requests. Tracing: **this** request, across services.

A trace is a tree. Root span = the request as the user experienced
it (often started at the gateway). Children nest via
`parent_span_id`. Generations are children with extra columns.

Debug workflow you should be able to run without folklore:

1. Alert or complaint ("intake is slow" / "answers got worse").
2. Metrics: is it a fleet problem or a slice (one model, one
   workflow)?
3. Query traces for that slice: duration, cost, score.
4. Open one representative waterfall. Name the slow or empty span.
5. Pull logs for that `span_id`.
6. If quality: look at attached scores and at retrieval/generation
   payloads (with ACL).
7. Only then change **one** thing (prompt, chunk k, model) and
   measure again — which is the Experimentation half.

If you skip to step 7, you are Sam shipping v8 of a prompt.

Context propagation is mechanical: inbound headers → context var →
outbound headers. Lose it once (a raw HTTP call that is not the
SDK) and the tree splits. Treat "unparented spans" as a bug.

## How platform services report telemetry

The platform promised observability **by default**. A workflow
author calls `platform.models.chat` and `platform.data.search` and
does not wrap spans. If they must, the platform failed the DX test.

Three layers deliver the promise. A fourth exists for logic the
platform cannot see.

### TracedService: automatic instrumentation

Every platform service extends a base that holds an
`ObservabilityClient` and two context managers:

- `trace_operation` — session get, search, validate, tool execute,
  anything that is not an LLM completion.
- `trace_generation` — LLM calls, recording model, tokens, cost,
  TTFT.

They start the primitive, time it, record errors, and yield a child
context so nested calls hang off the right parent. The workflow
developer never imports them. The Model Service implementation
enters `trace_generation` internally.

If a service skips the base class "because we were in a hurry," it
will be the span-shaped hole in every waterfall.

### Domain-specific telemetry

A generation record is one call. Ops also needs **aggregates**:
request rate, latency histograms, error ratio, cost rate, cache
hit ratio, guardrail block rate, retrieval k distribution.

That is a thin publisher: take the dataclass you already built in
the domain, call `record_counter` / `record_histogram` with
`PlatformMetrics` names and the standard labels. Do not make Model
Service know about Grafana. Do not make Observability know about
OpenAI usage objects.

Guardrails belong here as first-class metrics, not only logs.
Block rate without a trace is a vanity number; block rate *plus*
the policy name *plus* the trace is how you catch a bad threshold.

### The observability client: batching and buffering

`TracedService` creates objects. Publishers create metric records.
`ObservabilityClient` is how they **leave the process**:

- buffers for spans, metrics, logs (and generations/scores)
- flush when batch size or interval hits
- cap the buffer so a down Observability Service cannot OOM the
  Model Service
- retry flush independently of user requests

Numbers in the book (batch 100, flush 5s, cap 10k) are starting
points, not physics. Tune to your QPS. The invariant is: **no extra
RPC on the hot path per span**.

If flush fails, keep the buffer. If the cap trips, drop with a
metric. Never raise into `models.chat`.

### Custom observability beyond the defaults

Platform calls are covered. Application-only steps are not: a
custom rerank, a rules engine, a third-party HTTP API, a
post-processor. Those appear as a **gap** between two
platform-generated spans unless the author opens a span.

The SDK exposes the same `trace_operation` (and maybe a log helper)
the services use. Pass the current `trace_context`, a name
(`custom_rerank`), and a few attributes (`num_candidates`). Do not
invent a second tracing library.

If the custom step is actually a tool, register it with the Tools
service and you get Execute spans for free. Custom spans are for
glue that is not a capability.

## Quality scores and cost attribution

Telemetry answers what / how long / where it broke. It does not
answer **was it good**. A 1.2s, zero-error, $0.03 answer can still
be wrong. Scores are how quality becomes a column next to latency.

### Scores: measuring response quality

Three sources, three cost/quality curves:

**Automated / heuristic.** Code. Milliseconds. Deterministic.
Structural checks: required JSON, length bounds, "did the string
contain insurance card / photo ID / records." Cheap enough to run
on 100% of traffic. Blind to tone and to paraphrases that are
still correct.

**LLM-as-judge.** A model scores another model's output against a
criterion. Flexible: helpfulness, tone, flow. Expensive, noisy,
biased (especially same-family judges). Sample. Treat as
instrumentation with error bars, not a court. The platform
**stores the score** and which judge produced it; writing the
rubric is separate work.

**Human.** Ground truth. Slow. Pricey. Use on a sample, on
disagreements, on traces that already look bad. Calibration fuel
for the other two, not a replacement for them.

Attach with: name, value (or categorical), source, target
(trace/span/generation id), timestamp. Multiple scores per trace
are normal (completeness 0.8, toxicity pass, human pending).

Without scores, experimentation has nothing to maximize except
"looks nicer in the playground."

### Cost attribution and budget tracking

Per-request cost is a Model Service fact. Attribution is an
Observability query: sum dollars where labels match.

Drill-down, not a flat invoice:

1. Month by **team**.
2. Expensive team by **workflow**.
3. Expensive workflow by **model** (and cache hit rate).
4. Still confused: open generations.

Budgets are thresholds on those same cuts, with projections. Alert
before the quarter ends, not when procurement forwards the PDF.

A quality win that doubles tokens may still be a product loss. Put
`$ / successful task` on the same report as helpfulness, or you
will "improve" the assistant into insolvency. Routing and cache
in the Model Service are the usual levers once the report names
the model.

## The Experimentation Service

Observability asks what is happening. Experimentation asks **what
we should change**. Versioned targets, datasets, offline runs,
online scoring, A/B, annotation queues. This is the improvement
loop, not a second dashboard product.

Insight to keep: the two services are halves of one cycle.

```
  traffic --> traces --> scores --> low-scoring cases
                                         |
                                         v
                                   eval datasets
                                         |
                                         v
                              offline compare targets
                                         |
                                         v
                                   A/B in production
                                         |
                                         v
                              promote / deprecate --> traffic
```

Change **one** class of thing at a time or the loop cannot
attribute.

### Service contract

A teaching gRPC surface groups ~twenty RPCs into the lifecycle:

- **Targets** — register, history, compare. A target is a *change
  you might ship*: prompt name+version, model config, retrieval
  knobs. Not only prompts.
- **Datasets** — create, add cases, add-from-production.
- **Evaluations** — create, run (often a **stream** of progress),
  get results.
- **Scoring rules** — online sampling on live traces.
- **A/B experiments** — create, assign, record outcome, stop.
- **Annotation queues** — create, route, record labels.

SDK: `platform.experiments`, same lazy client pattern as the rest
of the platform SDK.

### Target lifecycle and evaluation

Teams improve by changing something measurable: rewrite a prompt,
swap a model or adapter, change chunk size or `k`, change a
reranker. All of those are **targets**. They need the same states:
draft → evaluated → production → deprecated.

The Experimentation Service **does not clone storage**. Prompts
already live in the Model Service registry. Retrieval config already
lives with Data / the workflow. Experimentation stores a *pointer*
(name, version) plus lifecycle metadata, eval summaries, and the
human description of the change. Duplicate storage is how v3 in
experiments diverges from v3 at inference.

Compare targets by running the same dataset through each pointer.
Promotion is a decision with numbers, not a Slack emoji.

## Evaluation: datasets, offline, online, humans

Evaluation is the gate on promotion. Three modes, on purpose:

- **Offline** — catch regressions before traffic.
- **Online** — catch what the dataset never imagined.
- **Human** — keep the other two honest.

Best teams use all three. Offline-only rots. Online-only ships
bugs to patients. Human-only does not scale.

### Datasets: the foundation

A dataset is cases: input, optional ideal output, tags, extra
fields (key elements, required citations). Traditional tests are
hand-written. AI eval's best cases often **come from production**:
the question that confused the model, the trace with helpfulness
< 0.5, the guardrail near-miss.

Start curated (the happy path and the obvious traps). Augment with
`AddFromProduction` filtered by score, tag, or workflow. Review
before they become gospel: production also contains junk and
injection.

Datasets rot. Policies change, products rename, last year's "ideal
response" is wrong. Version them. Retire cases. A stale golden
answer is a regression detector for the *wrong* behavior.

### Offline evaluation: compare before deploy

Pipeline: targets × cases → generate (full stack if you are testing
RAG, not only the prompt against a frozen context) → score →
aggregate and per-case diffs.

Because outputs are stochastic, **repeat** each case and average.
A one-shot "winner" is a coin flip with extra CI time. Include
retrieval when the change could interact with chunks; prompt-only
harnesses miss "the new prompt ignores the PDF."

Return both a headline (helpfulness +0.06) and the cases that
got worse. Averages hide the failure you will see on Twitter.

Offline is necessary and insufficient. Patients do not talk like
your YAML.

### Online evaluation: score production

Scoring rules: workflow, **sample rate**, scorers (mix heuristic
and judge), alert thresholds. Example: 10% of intake traces, key
elements on everyone in the sample, LLM helpfulness on a nested
sample if cost hurts.

Online is how you learn that the benchmark is 0.85 and production
is 0.72 because real phrasing was never in the file. It is also
how a silent quality drop pages someone when latency did not move.

Keep the judge **off** the user path. Score from the completed
trace in the background. If the judge is down, serving continues;
you lose a metric, not a patient.

### Annotation queues: calibrate with humans

Queues route traces to reviewers with a **rubric** (categorical
correctness, numeric helpfulness, "add this to the dataset?").
Routing rules should be boring and explicit: helpfulness < 0.5
from the judge, or `guardrail_triggered`, or random 1% for
calibration.

Three jobs: ground-truth labels, edge cases automation missed,
labeled data back into datasets. Humans do not replace online
scoring. They keep it from drifting into a mutual admiration
society of models.

Measure annotator agreement. A rubric nobody applies the same way
is theater.

## A/B testing infrastructure

Offline compares against files. A/B compares against **people**.
A prompt that wins the benchmark can lose in production.

An experiment: two or more **variants** (prompt / model / retrieval
/ combination), a **traffic split**, **success metrics** (and the
direction of "better"), a duration or a stopping rule.

**Assignment must be sticky.** Consistent hash of
`(experiment_id, user_id)` so the same patient does not get warm
v3 and terse v2 in one sitting. Hashing on `trace_id` is a UX bug.
Hashing on "whoever hit the replica" is worse.

```
  request + user_id
        |
        v
  consistent_hash(experiment, user) --> bucket
        |
        +-- control  (prompt v4, 50%)
        +-- treatment (prompt v5, 50%)
        |
        v
  workflow runs as usual (trace still recorded)
        |
        v
  outcomes (scores, cost, task success) attributed to variant
```

Do not A/B five knobs at once. You will not know what won. Do not
leave a "temporary" 5% experiment forever; it becomes an
undocumented shadow prompt.

Guardrails and eligibility: some users must never leave control
(clinical, legal). Encode that as an assignment constraint, not as
tribal knowledge.

## Putting it together

Two services. Observability stores and queries the truth of
production. Experimentation manages the lifecycle of change.
`TracedService` plus publishers plus a buffering client make
telemetry the default. Scores attach. Cost rolls up. Online rules
watch a sample. Low scores become dataset cases. Offline eval
picks a candidate. A/B confirms. You promote a **target**, not a
vibes-based prompt paste.

The workflow author's code should look like ordinary
`platform.sessions` / `models` / `data` / `guardrails` calls with
`trace_context` passing through, optional `assign_experiment`, and
an outcome record at the end. No span wallpaper. Listing-shaped
intent, infrastructure underneath.

If traces are empty when you hang a whole assistant on this stack,
that assistant is a black box with better branding.

### What these services are not

They are not the agent-local eval loop. Critics, grounding as a
guardrail, Phoenix sessions, test-driven agent development: the
platform will happily store a critic's score if you `RecordScore`.
It will not teach you how to write the critic.

They are not the Model Service. Token prices and usage originate
there; this chapter **aggregates and joins**.

They are not the Workflow Service. Observability *watches*
workflows; it does not deploy them. Jobs and health probes live
with workflow runtime management.

They are not an excuse to log every prompt in a shared bucket
without ACL. Generations are data with a user in them.

## Check yourself

1. Sarah's new prompt keeps p95 and error rate flat, but patients
   repeat themselves. Which traditional metric missed this, and
   which primitives (span, generation, score) would you inspect
   first?
2. Why is a 200 OK from the Model Service insufficient evidence
   that the assistant "worked"? Name two assembled inputs that can
   make a healthy model produce a bad answer.
3. Draw the path of one intake question through at least five
   services. Where does `trace_id` get created, and what debugging
   failure appears if Data omits it?
4. Cost report vs per-request Model metrics: which question does
   each answer, and why can a cheap workflow still be the one you
   kill?
5. Span vs log vs metric vs generation vs score: pick a guardrail
   near-miss and say which object holds duration, which holds the
   per-rule confidence, which holds the week-over-week block rate,
   and which holds "was the final reply helpful."
6. Why must ingest be fire-and-forget? Describe what goes wrong if
   `RecordGeneration` sits on the user path, and what the client
   buffer should do when Observability is down.
7. TracedService vs a custom `trace_operation` in workflow code:
   when is each appropriate? Give an example of a "gap" in the
   waterfall that defaults will not fill.
8. Automated scorer vs LLM-as-judge vs human queue: you need JSON
   validity on 100% of traffic *and* a tone check. How do you
   combine them without doubling the token bill?
9. A prompt scores 0.85 offline and 0.72 online. Give two dataset
   problems that could cause that gap, and what A/B assignment
   rule you would insist on before calling the online number
   causal.
10. In one sentence each: what does Observability store that
    Experimentation does not, and what does Experimentation manage
    that Observability does not?
