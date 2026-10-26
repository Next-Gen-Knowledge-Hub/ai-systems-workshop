# 7. Observability and experimentation

Companion notes for **Chapter 7** of *Designing AI Systems* (Suhas Suresha and Dewang Sultania; Manning MEAP).

This chapter is how the platform answers whether an answer was any good, not only whether the
process stayed up. Model, Session, Data, Tools, and Guardrails already emit signals. Observability
joins those signals on one trace: what happened, what it cost, and a score that may arrive after the
user already has the reply. Experimentation turns the weak traces into a versioned change, an
offline comparison, and a sticky A/B you can promote or drop. Afterward you can name a trace, a
span, a generation, a score, and which clock you are on: offline, online, human, or an experiment.

## 1. A green dashboard can still be a bad answer

Uptime, latency, error rate, and saturation still page someone. An AI workflow that returns HTTP 500
is still an incident. The extra job is to judge whether the output was useful, grounded, safe, and
worth the tokens, on the same request that touched Session, Data, Guardrails, and Model.

* **Sarah's intake prompt**: She ships a new patient-intake prompt. p95 is unchanged, the error rate
  is unchanged, and the Model Service is up. Patients say the assistant is less helpful. It asks
  them to repeat facts already in the session, and it takes more turns. Nothing there is an HTTP
  error. The system is running. It is working poorly.
* **The deterministic lie**: A normal service that is up and returns 200 almost certainly returned
  the correct bytes. Same code, same input, same output. A generation does not give you that
  bargain.
* **Unstable outputs**: The same inputs can yield different text. Temperature, routing, a cache
  miss, and provider drift all move the words. A successful generation can still be useless.
* **Assembled inputs**: The model sees a pipeline, not a stored request. Which chunks Data returned,
  how much history Session could fit, which model survived routing and fallback, and which guardrail
  rewrote a sentence all change the answer. A healthy Model Service can still answer from a bad
  retrieval.
* **Two rings**: Keep latency and errors as the inner ring. Wrap them with tokens, model identity,
  retrieval hit quality, safety actions, response quality scores, and cost per workflow. A slow
  request can be a great answer, such as a long tool loop that did the research. A fast 200 can be a
  hallucination. An on-call page that only fires on 5xx will never fire because patients are
  repeating themselves.

## 2. One user request is one trace

One patient question is several service calls. Quality and cost have to share a `trace_id`, or you
will tune whichever dashboard is easiest to graph. Four log formats, token counts that never leave
the Model Service, and a prompt "v7" nobody can A/B are the sprawl this join exists to stop.

* **Where the id starts**: The gateway authenticates and starts the trace. Every later hop must
  carry the same id.
* **A typical hop**: Session loads history and maybe memories. Data searches an index. Guardrails
  inspect the input. Model generates, more than once if tools loop. Guardrails inspect tool
  arguments and the output. Session stores the exchange.
* **Why local timings lie**: A request that took four seconds instead of one leaves a timing in each
  service. Without a shared `trace_id`, debugging is lining up timestamps and hoping the clocks
  agree. The waterfall is the honest answer to where the time and the tokens went.
* **False blame**: "The model is slow" dies when the waterfall shows Data at 3.2s and Model at
  400ms. "The model hallucinated" dies when the generation span shows empty retrieval and a
  truncated session.
* **Propagation**: Put the context on every internal call. The teaching transport is gRPC metadata:
  trace id, span id, workflow id. A service that cannot join the tree is an observability hole. A
  raw HTTP call that skips the SDK splits the tree. Treat an unparented span as a bug.

```
  one user request
        |
        v
  TRACE
    span: sessions.get
    span: data.search
    span: guardrails.input
    GENERATION: models.chat    model, tokens, $, TTFT
    span: tools.execute
    span: guardrails.output
    span: sessions.append
    scores .... attached later (heuristic / judge / human)
        |
        +-- Observability:   what happened, what it cost, was it good
        +-- Experimentation: what to change, did the change help
```

## 3. Sessions, traces, spans, generations, and scores

Start from the question you must answer. The primitive is whatever you need in order to reconstruct
that question. Vendor product names come later.

* **Trace**: The whole journey of one user request. Input, output, duration, total cost, workflow
  id, user id, tags. "What happened on this request?" is a trace.
* **Span**: One unit of work inside the trace: a session load, a search, a guardrail pass. Spans
  nest through `parent_span_id`. The waterfall is that nesting plus duration. The root span is the
  request as the user felt it, usually opened at the gateway.
* **Generation**: A span with LLM fields the rest of the platform should not invent: model name,
  prompt tokens, completion tokens, cost, time to first token, cache hit, provider, finish reason.
  Index generations on their own so "GPT-4o calls over $0.10" is a query, not a JSON grep.
* **Session**: A group of traces, the same way the Session Service groups messages. A session view
  answers "this user had a bad afternoon." One trace cannot.
* **Score**: A named judgment (helpfulness, completeness, retrieval relevance, safety) attached to a
  trace, a span, or a generation, with a source: heuristic, judge, or human. Scores often land after
  the user already has the reply. That lag is the design. Do not hold the HTTP response for a second
  model call.

Walk one question through the tree: "Can I bring medical records digitally, or do I need paper
copies?"

* **Gateway**: Creates `trace_id` and binds it to the patient's session.
* **Session span**: Twelve prior messages, about 45ms, about 2,100 tokens of history charged against
  the window.
* **Data span**: Hybrid search, k hits, latency, index name. Child logs may list document ids. The
  span holds the aggregate.
* **Guardrails input span**: Allowed, blocked, or transformed.
* **Generation**: The Model Service call. Tokens, cost, time to first token, and the model that
  routing actually used.
* **Close**: Guardrails output span, then a session append span.
* **Scores, later**: An automated key-element check in about 20ms, an LLM judge on a 10% sample, or
  a human next Tuesday.

Six spans and one generation is a teaching size. Tool loops grow the tree. The shape stays one tree,
one `trace_id`, and scores as decorations that may lag.

```
  SESSION  (many requests, one conversation)
     |
     +-- TRACE  (this user request)
            +-- SPAN   session.get
            +-- SPAN   data.search
            +-- GENERATION  models.chat
            |      +-- SCORE  grounding
            +-- SPAN   guardrails.output
            +-- SCORE  helpfulness   (often on the trace, often late)
```

Logs and metrics sit beside those five primitives. The primitives are per request. Logs explain one
span. Metrics describe a fleet.

* **Structured log**: An event during a span. The guardrail span says "35ms, success." The logs say
  "rule A 0.91 pass, rule B 0.48 near-miss, rule C transform." A cache lookup may emit nothing. A
  messy policy may emit a dozen events. Every event carries `trace_id` and `span_id`. The span is
  what happened and how long. The log is why it went that way.
* **Metric**: A counter or a histogram you refuse to recompute from raw logs when a dashboard loads.
  Counters cover requests, tokens, and dollars. Histograms cover latency and relevance. Labels such
  as provider, model, workflow id, and team let one metric name answer many cuts. Metrics say
  Tuesday's p95 moved. They will not show Maria's four-second trace. Keep all three, or you can see
  a fleet and still be unable to debug one patient.

Logs alone cannot draw the waterfall. Spans alone cannot explain why a guardrail almost fired.
Metrics alone cannot open one patient's request.

## 4. The Observability Service contract

The surface is a small gRPC API with an SDK in front, the same shape as the other platform services.
The twist is direction. Most platform services are called because the workflow wants an answer.
Observability is mostly a receiver. The teaching count is fourteen operations. An implementation may
split further. Ingest paths are cheap and numerous. Query paths are richer and rarer.

1. **Logs**: `IngestLogs` and `QueryLogs`. Individual records, searched by fields. Use them when the
   question is what happened inside one span. Folding logs into the metrics call forces one store to
   do two jobs badly.
2. **Metrics**: `RecordMetrics` and `QueryMetrics`. Numeric series, queried by time window and
   labels. Use them for rates and percentiles. A trace query will not draw Tuesday's p95 cheaply.
3. **Tracing**: `RecordSpan`, `RecordGeneration`, `GetTrace`, and `QueryTraces`. Generation is its
   own call so LLM columns stay off ordinary spans. Get is "this id." Query is "slow traces on
   workflow X yesterday."
4. **Scores**: `RecordScore`, plus a query. It arrives late and must attach to an existing trace,
   span, or generation. It must not rewrite the history of the request.
5. **Cost**: Reports and budgets derived from generation records plus labels such as team and
   workflow. A drill-down, not a second ledger. The Model Service remains the source of the leaf
   dollars.
6. **Health**: The service's own live and ready checks. Skipping them on the telemetry service is
   how you lose the recorder and notice last.

* **Ingest stays off the user path**: When Model finishes a call, it publishes asynchronously. If
  Observability is down, the user's answer still ships. Buffer locally and retry. Waiting inside
  `models.chat` for `RecordGeneration` turns telemetry into a single point of failure, and it fails
  while looking like care.
* **Query is the other shape**: Operators, dashboards, experiment pipelines, and cost reports. A
  second of latency is acceptable. Scores may appear eventually. The ingest API is not the query API
  with extra fields.
* **Who may read**: Every replica may ingest. Few principals may call `QueryTraces` and see raw
  prompts. A generation contains user text and sometimes retrieved documents. Treat it as sensitive.

### Watch for

* A flush that raises into `models.chat` pages users for a telemetry outage.
* When the local buffer hits its cap, drop or shed and emit a metric that you dropped. A silent drop
  looks like a quiet day.
* Designing `IngestLogs` as if an analyst will use it interactively makes the hot path slow and the
  query path weak.

## 5. What a log line, a metric, and a waterfall contain

The contract is verbs. Operators need one event shape so a slow trace does not end in four clusters
over SSH.

Every log event carries an id, a timestamp, a service, a severity, an event type, a message,
`trace_id`, `span_id`, a workflow id, and a user id or a hashed stand-in. Then two maps: string
attributes and numeric attributes. The split keeps `tokens=2100` queryable as a number.

* **Model keys**: Provider, model, cache hit, fallback reason, token counts, estimated cost.
* **Data keys**: Index, hit count, top score, filter keys. Raw chunk text only when policy allows
  it.
* **Guardrail keys**: Policy name, action, confidence, which rule.
* **Tool keys**: Tool name, version, breaker state, retry count.

Same envelope, different keys. Open the bad span, then query logs for that `span_id`.

Name each metric once. The teaching pattern is a constants class prefixed like
`ai.platform.{service}.{metric}`. `llm_latency_ms` in one repo and `model.duration` in another is
how the same latency gets two dashboards and zero trust.

1. **Counters**: Requests, tokens, cost, cache hits, guardrail blocks. A process only increments
   them. The time-series store computes the rate. Use a counter for a total. A counter cannot tell
   you a percentile.
2. **Histograms**: Latency, time to first token, retrieval scores. The numbers you will page on are
   p50, p95, and p99. The mean of a heavy tail hides the incident. Use a histogram when the tail is
   the outage.

Slice on `provider`, `model`, `workflow_id`, `team`, and maybe `cache`. Cardinality is a budget. A
user id label will melt the time-series store. High-cardinality truth belongs on the trace.

Each service already has domain structs. `RequestMetrics` comes from the Model Service. Tools and
Guardrails already emit evaluation records. Observability publishes those structs under
`PlatformMetrics` names through `record_counter` and `record_histogram`. The Model Service does not
learn a dashboard product. Observability does not learn a vendor usage object.

Guardrail block rate is a first-class metric. Block rate alone is a vanity number. Block rate plus
the policy name plus the trace is how a bad threshold gets caught.

Debug in this order. Skipping to the last step is Sam shipping v8 of a prompt.

1. Start from an alert or a complaint: intake is slow, or answers got worse.
2. Read metrics. Decide whether it is the whole fleet or a slice: one model, one workflow.
3. Query traces in that slice for duration, cost, and score.
4. Open one waterfall. Name the slow span or the empty span.
5. Pull logs for that `span_id`.
6. If the complaint is quality, read the attached scores and the retrieval or generation payloads.
   Access control still applies.
7. Change one thing — a prompt, a chunk `k`, or a model — and measure again.

## 6. Telemetry without the author wrapping every call

A workflow author calls `platform.models.chat` and `platform.data.search` and does not open spans.
If they must, the platform missed the developer promise. Three layers cover platform calls. A fourth
covers glue the platform cannot see.

* **TracedService**: Every platform service extends a base that holds an `ObservabilityClient` and
  two context managers. `trace_operation` wraps session get, search, validate, tool execute, and any
  unit that is not an LLM completion. `trace_generation` wraps the LLM call and records model,
  tokens, cost, and time to first token. They start the primitive, time it, record errors, and yield
  a child context so nested calls hang off the right parent. The workflow never imports them. The
  Model Service enters `trace_generation` itself. A service that skips the base class is a hole in
  every waterfall.
* **Domain publishers**: One generation record is one call. Operations also need rates: request
  rate, latency histograms, error ratio, cost rate, cache hit ratio, guardrail block rate, and the
  distribution of retrieval `k`. Publish the dataclass you already have. Do not invent a parallel
  metrics schema inside the domain service.
* **ObservabilityClient**: The path out of the process for spans, metrics, logs, generations, and
  scores. Buffer them. Flush on batch size or on an interval. Cap the buffer so a down Observability
  Service cannot exhaust memory inside the Model Service. Retry the flush on its own, independent of
  the user request. Teaching starting points, not physics: batch 100, flush every 5 seconds, cap
  10,000. Tune them to your rate. The invariant is no extra RPC on the hot path per span. A failed
  flush stays in the buffer. A tripped cap drops and emits a metric. It never raises into
  `models.chat`.
* **Custom span**: A homemade rerank, a rules engine, a third-party HTTP call, or a post-processor
  shows up as a gap between two platform spans until the author opens one. Use the same
  `trace_operation` the services use. Pass the current `trace_context`, a name such as
  `custom_rerank`, and a few attributes such as `num_candidates`. A second tracing library splits
  the tree again. If the step is actually a tool, register it with the Tools service and the execute
  span comes free. Custom spans are for glue that is not a capability.

## 7. Scores in one column, dollars in the next

A 1.2 second reply with zero errors and a $0.03 price can still be wrong. A score is how quality
becomes a column next to latency. Without scores, the only thing an experiment can maximize is a
playground impression.

Three sources. They differ by cost, by coverage, and by what they are blind to.

1. **Heuristic**: Code, milliseconds, deterministic. Structural checks: required JSON, length
   bounds, whether the string contained "insurance card," "photo ID," or "records." Run it on 100%
   of traffic. It misses tone, and it misses a paraphrase that is still correct. Use it as the
   always-on gate. It will not tell you the answer was kind.
2. **LLM-as-judge**: One model scores another's output against a criterion such as helpfulness,
   tone, or flow. It is flexible, expensive, noisy, and biased, especially when the judge is the
   same model family. Sample it. Treat the number as an instrument with error bars, not as a
   verdict. The platform stores the score and which judge produced it. Writing the rubric is
   separate work. Use a judge when the criterion is linguistic. Running it on every request will
   dominate the bill.
3. **Human**: Ground truth. Slow and expensive. Use people on a sample, on disagreements between the
   other two, and on traces that already look bad. Humans calibrate the heuristic and the judge.
   They will not scale as the only clock.

Attach each score with a name, a numeric or categorical value, a source, a target id (trace, span,
or generation), and a timestamp. Several scores on one trace are normal: completeness 0.8, toxicity
pass, human still pending.

Per-request dollars are a Model Service fact. Classical app cost tracks CPU and RAM and looks
similar across requests. Token cost swings with history length, retrieved context, model choice,
retries, and cache hits. The leaf record is how you find the one call that cost fifty cents instead
of two. Attribution is an Observability query: sum those dollars where the labels match.

1. Month by **team**.
2. That team's expensive workflows.
3. That workflow by **model**, including cache hit rate.
4. Still confused: open the generations.

A workflow at $1,400 a month can be excellent. A workflow at $400 can be burning tokens on padded,
unhelpful answers. Cost without a score cannot tell those apart. A score without cost cannot tell
you whether a quality win is affordable. Put dollars per successful task on the same report as
helpfulness. A quality win that doubles tokens can still be a product loss. Budgets are thresholds
on the same cuts, with a projection, so the alert fires before the quarter ends. Once the report
names the model, routing and cache in the Model Service are the usual levers.

## 8. The Experimentation Service

Observability says what is happening. Experimentation says what to change, and whether the change
helped. The two services are one cycle. Traffic becomes traces, traces receive scores, low scores
become dataset cases, the same dataset compares targets offline, a candidate meets people in an A/B,
and you promote or deprecate. Change one class of thing at a time. Two changes in one run leave you
unable to say which one moved the number.

```
  traffic --> traces --> scores --> low-scoring cases
       --> datasets --> offline compare --> A/B
       --> promote or deprecate --> traffic
```

The teaching surface is about twenty RPCs, grouped by lifecycle. The SDK entry is
`platform.experiments`, the same lazy client pattern as the rest of the platform SDK.

1. **Targets**: Register, read history, compare. A target is a change you might ship: a prompt name
   plus version, a model config, or retrieval knobs. Prompts are one kind of target. States run
   draft, then evaluated, then production, then deprecated. A prompt pasted into a handler has no
   state and no history, so it cannot be promoted with numbers.
2. **Datasets**: Create a dataset, add cases, add cases from production.
3. **Evaluations**: Create a run, execute it (often as a stream of progress), and fetch results.
4. **Scoring rules**: Sample live traces and attach scores in the background.
5. **A/B experiments**: Create, assign a variant, record the outcome, stop.
6. **Annotation queues**: Create a queue, route traces, record human labels.

Experimentation does not clone the objects it points at. Prompts already live in the Model Service
registry. Retrieval config already lives with Data and with the workflow. Experimentation stores a
pointer — name and version — plus lifecycle metadata, eval summaries, and a human description of the
change. A copied prompt is how "v3" in the experiment diverges from "v3" at inference time. Compare
targets by running the same dataset through each pointer. Promotion is that comparison, not a
reaction emoji.

* **Critics stay outside**: Call `RecordScore` and the platform will store a critic's number,
  including which judge wrote it. The service does not write the critic, the rubric, or an in-loop
  grounding check. A lab session that produced the score is not this database.
* **Prices stay in Model**: Token prices and per-call usage originate in the Model Service. This
  chapter aggregates them and joins them to quality.
* **Deploys stay in the workflow runtime**: Observability watches a workflow. It does not deploy
  one. A job row and a health probe are not a generation span.
* **Prompts are user data**: A generation has a person in it. Query access is a short allow list. A
  shared bucket of raw prompts is a different system, and a worse one.

## 9. Datasets and the three evaluation clocks

A target is promoted through evaluation. Three clocks, on purpose. The best teams run all three.
Offline-only rots as soon as real phrasing drifts. Online-only ships the bug to patients. Human-only
does not scale.

* **Dataset**: Cases. Each case has an input, an optional ideal output, tags, and extra fields such
  as key elements or required citations. Curate the first set yourself: the happy path and the
  obvious traps. Then call `AddFromProduction`, filtered by score, tag, or workflow, and review what
  comes back. Production traces include junk and injection attempts. A case that becomes gospel
  without review trains the eval on an attack. Datasets rot when policy, product names, or the right
  answer changes. Version them and retire cases. A stale golden answer is a regression detector for
  the wrong behavior.
* **Offline**: Cross targets with cases, generate, score, then return an aggregate and the per-case
  diffs. If the change can interact with retrieval, run the full stack. A harness that scores a
  prompt against a frozen context misses "the new prompt ignores the PDF." Outputs are stochastic,
  so repeat each case and average. A one-shot winner is a coin flip that spent CI time. Report the
  headline, such as helpfulness +0.06, and the cases that got worse. An average hides the failure
  patients will quote. Offline is required, and it is not enough. Patients do not talk like the
  file.
* **Online**: A scoring rule names the workflow, the sample rate, the scorers, and the alert
  thresholds. A teaching mix: 10% of intake traces, a key-element check on everyone in that sample,
  and an LLM helpfulness judge on a nested sample when the judge is too expensive to run wide.
  Online is how you discover that the file says 0.85 and production says 0.72, because real phrasing
  was never in the dataset. It is also how quality can page someone on a day when latency did not
  move. Score from the completed trace, in the background. If the judge is down, serving continues.
  You lose a metric. You do not lose the patient's reply.
* **Annotation queue**: A queue routes traces to a reviewer who has a rubric: categorical
  correctness, a numeric helpfulness score, and whether to add the case to a dataset. Keep the
  routes boring and explicit. Helpfulness under 0.5 from the judge, a `guardrail_triggered` flag, or
  a random 1% held out for calibration. The queue does three jobs: ground-truth labels, edge cases
  the automation missed, and labeled rows back into datasets. Humans do not replace the online
  sample. They keep two models from scoring each other into a mutual admiration society. Measure
  whether annotators agree. A rubric that two reviewers apply differently is theater.

## 10. An A/B test compares the change to people

Offline compares a target to a file. An A/B compares a target to people using the product. A prompt
can win the benchmark and lose in production.

An experiment has two or more variants — a prompt, a model, a retrieval setting, or a deliberate
combination — a traffic split, success metrics with a direction for "better," and a duration or a
stopping rule. The workflow runs as usual. The trace is still recorded. Scores, cost, and task
success are attributed to the variant.

* **Sticky assignment**: Hash `(experiment_id, user_id)` so the same patient stays on one variant
  for the life of the experiment. The teaching split is control on prompt v4 at 50% and treatment on
  prompt v5 at 50%. Hashing on `trace_id` gives one person the warm v3 and the terse v2 in a single
  sitting. Hashing on whichever replica accepted the request is worse.
* **One knob**: Five knobs in one experiment leave you unable to say what won. A "temporary" 5%
  experiment that nobody stops becomes an undocumented shadow prompt.
* **Eligibility**: Some users must stay on control: clinical, legal, or anyone else the policy
  names. Encode that as an assignment constraint. A hallway conversation about who is excluded will
  not survive the next on-call.

```
  request + user_id
        |
        v
  consistent_hash(experiment, user) --> bucket
        +-- control     prompt v4, 50%
        +-- treatment   prompt v5, 50%
        |
        v
  workflow runs; outcomes attach to the variant
```

## Check yourself

1. Sarah's new intake prompt keeps p95 and error rate flat, while patients repeat facts already in
   the session. Which traditional signal missed that, and which of span, generation, and score would
   you open first?
2. Why is HTTP 200 from the Model Service weak evidence that the assistant worked? Name two
   assembled inputs that can make a healthy model produce a bad answer.
3. Walk one intake question through at least five hops. Where is `trace_id` created, and what
   failure appears in the waterfall if Data omits it?
4. Which question does a per-request Model cost answer, and which question does the Observability
   rollup answer? Why might the $400 workflow be the one you shut off, rather than the one at
   $1,400?
5. A guardrail near-miss: which object holds the duration, which holds the per-rule confidence,
   which holds the week-over-week block rate, and which holds whether the final reply was helpful?
6. Why does `RecordGeneration` stay off the user path? What does the client buffer do when
   Observability is down, and what does it do when the cap trips?
7. When is `TracedService` enough, and when do you open a custom `trace_operation`? Name a gap in
   the waterfall that the defaults leave empty.
8. You need JSON validity on every request and a tone check. Which source runs at 100%, which do you
   sample, and why does the judge stay off the path that serves the patient?
9. A prompt scores 0.85 offline and 0.72 online. Name two dataset problems that can open that gap,
   and the assignment rule you need before you treat the online number as caused by the variant.
10. What does Observability store that Experimentation does not copy, and what lifecycle does
    Experimentation manage that a trace store does not? What breaks if the experiment holds its own
    copy of prompt v3?
