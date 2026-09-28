# 8. The Workflow Service

Companion notes for **Chapter 8** of *Designing AI Systems* (MEAP, Suhas
Suresha and Dewang Sultania; Manning). This MEAP's printed contents page is
**single-level** (nine chapter titles). The headings below follow the
chapter's actual topics.

This chapter is how a Python function becomes a **running service**:
HTTP, jobs, health, scale, a route on the gateway. Every earlier
chapter was a capability a workflow *calls*. This chapter is the
unit you *deploy*. Skip it and you still have notebooks, one-off
FastAPI files, and four teams inventing four ways to stream tokens.

The folder stays on the **Workflow Service** (control plane) and the
**runtime server** (data plane in the container). Control graphs,
handoffs, and agentic loops are a different book; here the workflow
is the **process** those graphs run in.

## The mental model

```
  genai-platform deploy foo.py
           |
           v
  register spec --> build image --> DeployWorkflow
           |                              |
           v                              v
  registry (name, api_path,         k8s Deployment + HPA
            image, scale, mode)           |
                                          v
                               +---- container ----+
                               | runtime server    |
                               |  (SDK entrypoint) |
                               |  finds @workflow  |
                               |  HTTP / SSE / job |
                               |  health probes    |
                               +---------+---------+
                                         |
                    gateway route /api_path  --> replicas
                                         |
                                         v
                          platform.*  (Model, Session, ...)
```

Two halves, one name. **Control plane:** registry, deploy, rollback,
replica counts, gateway routes, async **job records**. **Data plane:**
the SDK runtime inside each container, which unwraps JSON, runs your
function, and talks to other services. For sync and stream, the
control plane is **not** on the request path after routing. For
async, job status lives in the control plane because the HTTP
request already returned.

The one sentence to remember: **the decorator is the deploy
contract**; Kubernetes YAML is an implementation detail you should
not type.

The platform SDK promised one workflow / one service and three
response modes. This chapter builds the server that honors them.

## Workflow runtime server

The deploy pipeline stuffs your code, the SDK, and dependencies into
an image. Something has to listen. That something is **library code
in the SDK**, not a fifth platform microservice. The image
entrypoint *is* the runtime server. From outside: HTTP. From inside:
your function plus enough wrapping to enforce timeout, map modes, and
propagate traces.

### The @workflow decorator

What you write is a function plus metadata. The decorator does not
run the assistant at import time. It **stamps** configuration the
runtime and the deploy CLI both read (`_workflow_metadata` in the
teaching implementation).

Three buckets of knobs:

- **Identity and mode** — `name`, `api_path`, `response_mode`
  (`sync` | `stream` | `async`).
- **Scale and resources** — min/max replicas, CPU/memory targets,
  CPU/memory limits, optional GPU. These flow to the control plane
  at deploy time.
- **Reliability inside the container** — `timeout_seconds`,
  `max_retries` (request-level). The runtime enforces these; the
  orchestrator never sees your Python timeout.

A patient-intake teaching workflow then looks like intent:
`get_or_create` session, `guardrails.validate_input`, `data.search`,
`models.chat`, return a dict. `GenAIPlatform()` inside the function
(or a module-level client) is the SDK client, not a global god
object you reimplement.

If a knob cannot be expressed on the decorator, it will reappear as
a snowflake Dockerfile. Resist until you have evidence.

### From decorated function to HTTP server

On container start the runtime:

1. Reads `WORKFLOW_NAME` (and the module path) from the
   environment.
2. Imports the module — which runs the decorator, attaching
   metadata.
3. Finds the function whose metadata name matches.
4. Builds a small HTTP app (FastAPI in the book) with:
   - `POST {api_path}` bound to a handler for the mode
   - job GET if async
   - `/health/live` and `/health/ready`

Discovery-by-import is why "two decorated functions in one file"
is two **workflows** to the deploy tool, and why a missing
`WORKFLOW_NAME` is a boot failure, not a random function serving
traffic.

The handler's job is mechanical: JSON body → kwargs (check the
signature), trace headers → context var, run function under
timeout, dict → JSON (or SSE, or job id). Your assistant should
not parse `Request` objects unless you are doing something the
platform forgot.

### Synchronous mode

Default. The HTTP request **is** the unit of work. Fits classifiers,
short Q&A, cached knowledge, anything that finishes in a few
seconds and fits in a reverse-proxy timeout.

Handler sketch: parse JSON, bind kwargs (422 if the client sent
garbage), `wait_for` the function on a worker thread if it is
sync Python, 504 on timeout, 500 on uncaught exception, 200 with
the returned dict.

Trace context comes off gateway headers (`x-trace-id` and friends)
**before** the first `platform.*` call. If you forget, every span
in this request is an orphan.

Sync is the wrong mode for "summarize these 400 PDFs." You will
fight idle timeouts and hold a worker for minutes. That is async.

### Streaming mode

`response_mode="stream"` and the function **yields**. Each yield
becomes a Server-Sent Event. Conversational UIs want tokens as
the Model Service produces them, not a JSON blob at the end.

Inside: `platform.models.chat_stream(...)` instead of `chat`.
Loop, yield `{"token": ...}`, then a terminal event (`done`,
`session_id`). The runtime does not guess tokens from a blocking
`return`.

Errors mid-stream are uglier than sync 500s: the client already
drew half an answer. Decide whether to send an SSE error event
and close, or to finish a sentinel. Document it. Timeouts still
apply to the **whole** generator, or you will stream forever.

Do not pretend stream is async. The connection stays open. A
research job that takes twenty minutes wants a job id, not an
SSE the laptop closed.

### Asynchronous mode

The request does **not** run the function inline. Runtime creates
a **job** via the Workflow Service, returns **202 + job_id**
immediately, and runs the function in a background task. Client
polls.

Developer code still looks like a function that returns a dict
(deep research, batch analysis). The mode flag changes the HTTP
contract, not the Python shape. That is the point: one decorator,
three ways to wait.

Idempotency and retries get sharper here. A client that never
saw the 202 may POST again. Job creation should be safe if you
pass an idempotency key, or you will research twice.

## Async job lifecycle

Two open questions after "return a job id": where does the row
live, and how does the user see progress.

### Job storage and the polling endpoint

`create_job` stores enough for **three consumers**:

```
  CONSUMER           ACCESS              NEEDS
  ----------------   -----------------   ---------------------------
  polling client     GET /jobs/{id}      status, progress, result
  runtime server     write by id         progress, complete, fail
  crash recovery     query orphans       original body, assignment
```

Status values you can explain to a frontend: queued, running,
succeeded, failed, cancelled (and maybe timed_out). Store the
request body (size-capped) so recovery can re-dispatch. Store the
result or error. Store which replica owns the run if you need
sticky cancellation.

The **Workflow Service** owns the table, not a local SQLite in
the container. Containers die. The poller should not.

`GET /jobs/{job_id}` can be served by the gateway → Workflow
Service, or by the same runtime that created the job. Teaching
design: control plane holds truth; runtime writes through the
SDK. Do not make the browser talk to a pod IP.

Crash recovery: a replica exits mid-job. Orphan finder sees
`running` with a dead endpoint, re-queues or fails. Without
checkpoints (next subsection) you restart from zero.

### Progress reporting

Pollers hate a boolean. `update_job_progress("Retrieved 80
documents")` is how a research UI stays honest. Call it between
expensive steps, not every token.

Checkpoints are the sibling: `save_checkpoint` / `load_checkpoint`
so a retry after crash does not re-embed the corpus. Progress is
for humans. Checkpoints are for the function. Do not confuse
them. A progress string is not restorable state.

If you never report progress, clients poll until a cliff. If you
report on a hot loop, you DDoS your own job table. Once per
stage is the teaching default.

Cancel: client asks, control plane marks cancelled, runtime
should notice between stages. LLM calls in flight may still
finish; you bill those tokens. Document that.

## Production readiness: concurrency, retries, health

Serving one request on your laptop is not production. AI
workloads stress the usual web patterns: a "request" may wait
**seconds** on a provider, so a blocked event loop is a self-DoS.
Transient gRPC blips to the gateway are normal during rolls.
The orchestrator needs probes that mean something.

### Concurrency inside a single container

Uvicorn (ASGI) multiplexes: while `models.chat` waits on gRPC,
the loop can start another request. One process still has a
ceiling (threads for blocking SDK calls, RAM, open connections).

Production often starts **several workers** in the container
(separate processes, each with a loop and a thread pool). A
starting point: a few workers per CPU for wait-on-I/O workflows.
GPU workflows are a different shape: one replica may already be
the unit of isolation.

Sync handlers that call blocking SDKs should not pin the event
loop. The book's `asyncio.to_thread` is that lesson. If you
forget, one slow generation stalls health checks on that worker.

`max_replicas` on the decorator is **horizontal**. Workers are
**vertical** inside a replica. Tune both; do not crank workers
until the pod OOMs.

### Retry at the network layer

Every `platform.*` call is a hop: container → gateway → service.
The hop can drop (brief partition, gateway bounce, service
scale-from-zero). SDK clients retry **transient** failures with
backoff. The workflow function sees success or a final
exception.

This is **not** the Model Service's provider retry. Two layers:

```
  workflow  --SDK retry-->  Model Service  --provider retry-->  vendor
```

SDK retry: get the request *to* Model. Model retry: get a
completion *from* OpenAI. Independent. A 429 from the vendor is
not cured by retrying the gRPC to your own Model Service in a
tight loop — that is how you amplify a quota problem.

Honor `is_idempotent` on tools before the SDK retries Execute.
POST-that-charges is not "transient" just because the socket died.

`max_retries` on the **workflow decorator** is about the
*incoming* HTTP request (gateway → this container), another
layer again. Draw the three on a whiteboard once or you will
debug the wrong one.

### Health endpoints

Two probes, because Kubernetes uses them differently.

**Liveness** (`/health/live`) — process can still serve. Failure
means restart me. Deadlock, wedged loop, unrecoverable leak.
A healthy replica should almost never fail live. If live fails
because a **dependency** is down, you restart-loop during a
Model Service outage. Live should be "I am not dead," not "the
world is perfect."

**Readiness** (`/health/ready`) — ready for **traffic**. During
startup: import finished, SDK constructed, maybe a cheap
connectivity check. During shutdown: fail ready first, drain,
then die. If Data is down, you *might* fail ready (stop sending
work) while staying live (do not thrash restarts). Product
choice: a workflow that can still answer from cache might stay
ready with degraded mode. Document it.

The runtime exposes both. The Deployment points probes at them.
If you only implement `/health`, you have collapsed two meanings
and will restart pods that were merely waiting on a dependency.

## Workflow composition

A self-contained intake assistant is the tutorial. The org-scale
pattern is **call the workflow another team already shipped**.
Search exists. Summarization exists. Product wants both. Copying
code copies none of the improvements and **all** of the
dependencies (including the GPU box summarization needed).

Composition is how the platform becomes a **network of
capabilities**, not a folder of duplicated Python.

### Calling other workflows

One SDK method: `platform.workflows.call(api_path, body)`. HTTP
POST through the **gateway**, same as an external client. You get
a dict. Parent does not import child's module. That import would
smash isolation (one container, one image, one resource envelope).

Ticket-responder teaching shape: classify with a small model,
`call("/knowledge/search", ...)`, draft with a larger model.
Search stays owned by the team that tunes chunking.

Auth and trace: the call must forward identity and `trace_id` or
you get a child trace that cannot join the parent waterfall and a
confused-deputy risk. Treat child workflows as you treat tools:
least privilege, not "internal so skip auth."

Timeouts nest. Parent timeout 30s and child timeout 30s means
the child can eat the whole budget. Set parent > sum of sync
children, or use parallel + a tighter child timeout.

Cycles: A calls B calls A. Detect or forbid. The registry can
store a graph; a runtime DFS on every call is a last resort.

### Parallel workflow calls

If children do not depend on each other, sequential `call` wastes
the clock. `call_parallel([...])` fires several HTTP requests,
waits for all (or documented fail-fast), returns a list aligned
with the inputs.

Research assistant: papers, news, patents, then one model
synthesis. Wall time ≈ max(children), not sum, plus synthesis.

Partial failure: decide all-or-nothing vs "best effort with
errors in the payload." Silent omit is how you hallucinate a
source you never searched.

Parallelism is **across containers**, not threads inside one
image. That is why composition beats stuffing three libraries
into the parent image: each child scales on its own HPA.

### Response mode handling

Parent authors should not branch on whether the child is sync,
stream, or async. `call()` inspects the HTTP response:

- **200 + JSON** — sync child; return the object.
- **200 + `text/event-stream`** — consume the stream to
  completion (or a documented truncated join), return the
  assembled dict (or concatenated text).
- **202 + job_id** — poll until terminal; return the result.
- anything else — error with status and body.

```
  call(path, body)
        |
        v
     HTTP POST
        |
        +-- 200 JSON  ----------------> dict
        +-- 200 SSE   --> consume ----> dict
        +-- 202       --> poll job ---> dict
        +-- other     --> raise
```

Streaming child into a sync parent means the parent **waits for
the whole stream**. You gained composition, not a streaming UX.
If the parent must stream to *its* caller, it should yield as it
reads the child's SSE, not buffer then return. That is a
different helper (`call_stream`) if you need it; the teaching
`call()` is "give me the answer."

Async child: parent's `call()` blocks on the poll loop (with
timeout). Nested async is easy to turn into unbounded wait.
Cap it.

## The Workflow Service contract

Other services had one consumer: workflow code via SDK. Workflow
Service has **three**, with different trust:

```
  deploy CLI          registry + deploy/rollback/status
  runtime in pods     create job, progress, complete, fail
  external clients    get status, cancel  (+ maybe status)
```

Figure that in your notes. Do not give the public poller
`DeleteWorkflow`.

### Registry operations

How a workflow **enters** the platform. `RegisterWorkflow` carries
a spec: name, `api_path`, image, `response_mode`, scaling,
resources, version. CLI fills this from decorator metadata plus
the image reference it just pushed.

Update vs new version: treat spec changes like tool semver. A
new `api_path` is a breaking gateway change. Bump version.
`DeleteWorkflow` is an explicit act, not garbage collection of
the last replica.

List/Get are how a portal shows what exists without SSH to the
cluster. The registry is the catalog. Kubernetes is not.

### Deployment operations

Registry is intent. **Deploy** is actuation: take `workflow_id` +
version, create replicas, wait for ready, **register the route**.
Response: `deployment_id`, status.

`WorkflowDeployment` view: version, status, current vs desired
replicas, healthy endpoints. `GetDeploymentStatus` is what CI
polls. `RollbackWorkflow` points traffic at a previous image
and spec — your prompt-adjacent analog of reverting a bad
container, not an experiment.

Register without deploy is a saved spec. Deploy without register
is how you get snowflake images the next CLI run cannot find.
Keep the order.

## Deployment pipeline

The one-liner is `genai-platform deploy`. Here, the steps and the
gateway mapping.

### What genai-platform deploy does

Developer has `patient_intake.py` with `@workflow`. They run one
command. Teaching sequence:

1. **Import and discover.** Decorator runs; scan for metadata.
   Multiple decorated functions → multiple workflows.
2. **Read metadata.** Name, path, mode, scale — same dict the
   runtime will read.
3. **Dockerfile.** Generate a default if none: base image, copy
   code, install deps, `ENTRYPOINT` = runtime server, `CMD` =
   module. If they supplied a Dockerfile, use it (GPU, system
   libs).
4. **Build and push** the image to the org registry. Tag with
   version / git sha. The spec stores that reference.
5. **Register** with Workflow Service.
6. **Deploy** and wait for healthy + route.

Boundaries: CLI is a client. It should not kubectl as the
developer user if the platform is supposed to own isolation.
The Workflow Service talks to the orchestrator.

If step 1 imports a module that talks to production at import
time, deploy is a load test. Keep imports side-effect light.

### Route registration with the API gateway

When replicas pass **readiness**, Workflow Service tells the
gateway: `api_path` → these endpoints. The routing table is
**dynamic**:

- new replica ready → add address
- replica fails probe / drain → remove address
- new version → swap addresses as the roll proceeds

The gateway should not be a static YAML committed on Friday.
If you "just" put an Ingress in front of a Service and never
tell the platform's gateway about `api_path`, SDK clients and
auth middleware that live on the **platform** gateway will not
see the workflow. Two doors, two security stories — sprawl.

Authn/n at the gateway still applies. A deployed workflow is
not automatically public.

## Runtime management

Deploy creates. Runtime management **keeps**.

Developers do not write manifests. The decorator maps to:

**Deployment** — image, `min_replicas` as baseline, CPU/memory
(GPU) limits, probes on `/health/live` and `/health/ready`,
restart policy. Crash → reschedule. Live fail → replace.

**HorizontalPodAutoscaler** — watch CPU (and optionally memory)
against `target_cpu_percent`, clamp `[min_replicas, max_replicas]`.
Asymmetric windows: scale up faster than down so a traffic spike
does not flap.

```
  @workflow(min_replicas=1, max_replicas=10, target_cpu_percent=70)
          |
          v
  Deployment replicas=1..10
  HPA: CPU 70% --> add pods (cap 10)
                 --> remove pods (floor 1)
```

GPU and scale-to-zero are product choices. A GPU assistant with
`min_replicas=0` means cold starts measured in minutes. Sync
user-facing chat rarely wants that. Async research might.

Rolling updates: new version, new pods become ready, route
flips, old pods drain. In-flight sync requests need drain
timeout ≥ workflow timeout or you 502 successful work.

The control plane watches probe results to fill
`healthy_endpoints`. If your HPA scales on CPU but the workflow
is **token-bound and idle on gRPC**, CPU stays low while latency
explodes. Then you need concurrency-based or queue-based
signals — a later refinement. Know the failure: CPU is a proxy,
not a user SLO.

## Putting the halves together

A request to `/patient-assistant`:

1. Gateway authenticates, starts trace, routes via the table
   Workflow Service maintains.
2. A ready replica's runtime binds JSON to the function.
3. Function calls Session, Data, Model, Guardrails; SDK retries
   blips; TracedService records spans.
4. Sync: JSON back. Stream: SSE. Async: 202, job row, poll.

Nobody wrote a Deployment by hand. Nobody opened a provider SDK
inside the handler if they followed the platform services. The
graph you *might* run **inside** the function (handoffs, ReAct,
research loop) is agent work. The container, the route, and the
job table are this chapter.

### What this service is not

It is not the agent graph. Flows, hubs, teams, layered loops
belong with agent design. The workflow is the **process** that
graph runs in.

It is not Observability. You emit traces; you do not store them
here. Job progress is not a substitute for a generation span.

It is not the gateway. It **feeds** the gateway routes. Auth,
quotas, and external HTTP still live with the gateway.

It is not a replacement for a general job bus for *non-AI*
batch. You *can* run a slow workflow async; you should not
rebuild payroll on `@workflow` because the HPA looks handy.

## Check yourself

1. Name the two halves of "Workflow Service." For a sync POST,
   which half is *not* on the path after the gateway routes?
2. Map decorator fields to: (a) registry spec, (b) K8s
   resources, (c) enforced only inside the runtime. Where does
   `timeout_seconds` belong, and why not in the Deployment spec?
3. Sync vs stream vs async: pick one product moment (intake
   chat token UX, ICD-10 classify, overnight chart review) for
   each, and one failure if you chose wrong.
4. Three job-record consumers: what breaks if the job row lives
   only on the replica's disk?
5. Live vs ready: during a Model Service outage, which probe
   should fail if you want to stop traffic without a restart
   storm? Why is "dependency down ⇒ live fail" dangerous?
6. Draw the three retry layers (incoming HTTP, SDK to platform
   services, Model Service to vendor). Which one retries a 429
   from OpenAI, and which should *not*?
7. Why must `workflows.call` go through the gateway instead of
   importing the child's Python? Give a resource reason and a
   security reason.
8. A parent `call()`s a child with `response_mode="async"`. What
   does the parent actually wait on, and what timeout bug appears
   if nobody caps the poll?
9. List the six deploy-CLI steps. At which step does `api_path`
   become a gateway route, and what happens to that table when
   a replica fails ready?
10. In one sentence each: what problem does composition solve
    that copying a search team's Python into your image does
    not, and what does the HPA fail to see when the workflow is
    token-bound and idle on gRPC?
