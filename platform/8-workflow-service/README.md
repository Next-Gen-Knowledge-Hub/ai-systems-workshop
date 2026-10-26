# 8. The Workflow Service

Companion notes for **Chapter 8** of *Designing AI Systems* (Suhas Suresha and Dewang Sultania; Manning MEAP).

This chapter is how a Python function becomes a running service: an HTTP route, a job when the work
is long, health probes, a scale range, and a path on the gateway. The function calls Model, Session,
Data, Tools, and Guardrails. The Workflow Service is the unit you deploy around that function.
Afterward you can name the control plane and the data plane, pick sync, stream, or async on purpose,
and say which retry layer owns a failure.

## 1. Control plane and data plane

Two halves share one name. The decorator is the deploy contract. Kubernetes YAML is an
implementation detail you should not have to type. The platform SDK promised one workflow per
service and three ways to answer. This chapter is the server that honors that promise.

* **Control plane**: The Workflow Service. It holds the registry, deploy and rollback, replica
  counts, gateway routes, and async job records. After the gateway has routed a sync or streaming
  request, this plane is not on the path. For async, job status lives here because the HTTP request
  already returned.
* **Data plane**: The runtime server inside each container. It is library code in the SDK, not
  another platform microservice. The image entrypoint is that server. From outside it is HTTP. From
  inside it is your function, plus enough wrapping to enforce a timeout, map the response mode, and
  pass a trace.
* **What you deploy**: `genai-platform deploy` reads the decorated function, registers a spec,
  builds an image, and calls deploy. The registry stores name, `api_path`, image, scale, and mode.
  The orchestrator runs a Deployment and a horizontal autoscaler. Replicas serve the gateway route
  and call the other platform services.

```
  genai-platform deploy foo.py
           |
           v
  register spec --> build image --> deploy
           |                            |
           v                            v
  registry (name, api_path,       replicas + autoscaler
            image, scale, mode)         |
                                        v
                              runtime server in the container
                              finds @workflow
                              HTTP / SSE / job, health probes
                                        |
                    gateway route /api_path --> replicas
                                        |
                                        v
                              platform.* (Model, Session, ...)
```

## 2. The @workflow decorator

What you write is a function plus metadata. The decorator does not run the assistant at import time.
It stamps configuration that both the runtime and the deploy CLI read. In the teaching
implementation that stamp is `_workflow_metadata`.

A patient-intake function then reads as intent: `get_or_create` a session,
`guardrails.validate_input`, `data.search`, `models.chat`, return a dict. `GenAIPlatform()` inside
the function, or a client at module scope, is the SDK client. A knob that cannot be said on the
decorator comes back as a snowflake Dockerfile. Wait until you have evidence before you special-case
the image.

1. **Identity and mode**: `name`, `api_path`, and `response_mode` of `sync`, `stream`, or `async`.
   These decide how callers wait and which URL the gateway publishes. A wrong mode is a product bug:
   chat tokens stuffed into one JSON blob, or a twenty-minute job holding a browser socket.
2. **Scale and resources**: Minimum and maximum replicas, CPU and memory targets, CPU and memory
   limits, optional GPU. These flow to the control plane at deploy time and become the Deployment
   and the autoscaler. They do nothing for a single call that is already running.
3. **In-container reliability**: `timeout_seconds` and `max_retries` at the request. The runtime
   enforces them. The orchestrator never sees your Python timeout. Putting `timeout_seconds` only in
   a Deployment spec leaves the function free to run past the only limit that understands the work.

## 3. From a decorated function to an HTTP server

The image contains your code, the SDK, and dependencies. Something still has to listen. On container
start the runtime does four mechanical steps. Your assistant should not parse `Request` objects
unless you are doing something the platform forgot.

1. Read `WORKFLOW_NAME` and the module path from the environment.
2. Import the module. Import runs the decorator and attaches metadata.
3. Find the function whose metadata name matches.
4. Build a small HTTP app (FastAPI in the teaching server) with `POST {api_path}` bound to the
   handler for that mode, a job GET when the mode is async, and `/health/live` plus `/health/ready`.

* **Discovery by import**: Two decorated functions in one file are two workflows to the deploy tool.
  A missing `WORKFLOW_NAME` is a boot failure. It is not a license for a random function to serve
  traffic.
* **The handler**: JSON body to keyword arguments, checked against the signature. A body that does
  not match is HTTP 422. Trace headers become a context variable before the first `platform.*` call.
  The function runs under the timeout. A dict comes back as JSON, as server-sent events, or as a job
  id. Forget the trace headers and every span in this request is an orphan.
* **Import side effects**: Deploy imports the module to discover metadata. An import that talks to
  production turns deploy into a load test. Keep import light.

## 4. Sync, stream, and async

The mode flag changes the HTTP contract. It does not change the fact that you wrote a function. One
decorator, three ways for the caller to wait. Pick from the product moment, not from habit.

1. **Sync**: The HTTP request is the unit of work. Use it for a classifier, a short answer, or
   anything cached that finishes in a few seconds and fits a reverse-proxy timeout. The handler
   parses JSON, binds arguments, and waits on the function. Blocking Python runs on a worker thread
   so the event loop stays free. Timeout is HTTP 504. An uncaught exception is HTTP 500. Success is
   HTTP 200 and the returned dict. Trace context comes off gateway headers such as `x-trace-id`
   before the first platform call. Sync is the wrong mode for "summarize these 400 PDFs." You will
   fight idle timeouts and hold a worker for minutes.
2. **Stream**: `response_mode="stream"` and the function yields. Each yield becomes a server-sent
   event. A conversational UI wants tokens as the Model Service produces them. Inside, call
   `platform.models.chat_stream(...)`, yield `{"token": ...}`, and finish with a terminal event such
   as `done` plus `session_id`. The runtime will not invent tokens from a blocking `return`. An
   error mid-stream is uglier than a sync 500, because the client already drew half an answer. Send
   an SSE error event and close, or finish a sentinel, and document which. The timeout covers the
   whole generator. A stream is not a job. The connection stays open. A twenty-minute research task
   wants a job id, not an SSE the laptop already closed.
3. **Async**: The request does not run the function inline. The runtime creates a job through the
   Workflow Service, returns HTTP 202 and a `job_id`, and runs the function in a background task.
   The client polls. Developer code still looks like a function that returns a dict: deep research,
   a batch of charts. A client that never saw the 202 may POST again. Job creation has to be safe
   when the caller passes an idempotency key, or you will research the corpus twice.

## 5. Jobs, progress, and checkpoints

Returning a job id leaves two questions: where the row lives, and how a person sees progress. The
Workflow Service owns the table. A local database on the replica's disk dies with the replica, and
the poller should not.

`create_job` stores enough for three consumers.

| Consumer | How it gets in | What it needs |
|---|---|---|
| Polling client | `GET /jobs/{id}` | Status, progress, result |
| Runtime server | Write by id | Progress, complete, fail |
| Crash recovery | Query orphans | Original body, which replica owned the run |

* **Status**: Values a frontend can explain: queued, running, succeeded, failed, cancelled, and
  maybe timed out. Store the request body, size-capped, so recovery can dispatch again. Store the
  result or the error. Store which replica owns the run if cancellation has to hit a particular
  process.
* **Who serves the GET**: The gateway can route to the Workflow Service, or the same runtime that
  created the job can serve it. The teaching design keeps truth in the control plane and has the
  runtime write through the SDK. The browser does not talk to a pod IP.
* **Orphans**: A replica that exits mid-job leaves a row stuck in `running` with a dead endpoint. An
  orphan finder re-queues or fails that row. Without a checkpoint you restart from zero.
* **Progress**: `update_job_progress("Retrieved 80 documents")` is how a research UI stays honest.
  Call it between expensive stages, not on every token. A poller with only a boolean waits for a
  cliff. A progress write on a hot loop is a denial of service against your own job table. Once per
  stage is the teaching default.
* **Checkpoint**: `save_checkpoint` and `load_checkpoint` are for the function. A retry after a
  crash should not re-embed the corpus. A progress string is not restorable state. People read
  progress. The function reads checkpoints.
* **Cancel**: The client asks, the control plane marks the job cancelled, and the runtime notices
  between stages. An LLM call already in flight may still finish. You pay for those tokens. Say so
  in the contract the client reads.

## 6. Concurrency, retries, and health

One request on a laptop is not production. An AI "request" may wait seconds on a provider, so a
blocked event loop is a self-inflicted outage. Transient hops during a roll are normal. The
orchestrator needs probes that mean two different things.

* **One process**: An ASGI server such as Uvicorn multiplexes. While `models.chat` waits on gRPC,
  the loop can accept another request. The ceiling is still real: threads for blocking SDK calls,
  RAM, and open connections.
* **Workers**: Production often starts several worker processes in the container, each with a loop
  and a thread pool. A few workers per CPU is a starting point when the work is wait-on-I/O. A GPU
  workflow is a different shape: one replica may already be the unit of isolation. `max_replicas` on
  the decorator is horizontal, across pods. Workers are vertical, inside one replica. Raising
  workers until the pod runs out of memory is not a scale strategy.
* **Do not pin the loop**: A sync handler that calls a blocking SDK should hop to a thread. The
  teaching call is `asyncio.to_thread`. Forget it and one slow generation stalls health checks on
  that worker.

Three retry layers. Draw them once. Debugging the wrong one wastes the incident.

```
  caller --incoming retry--> this workflow
  workflow --SDK retry--> Model Service --provider retry--> vendor
```

1. **Incoming HTTP**: `max_retries` on the workflow decorator. This is the gateway, or the caller,
   trying this container again. It knows nothing about OpenAI. Use it for a lost connection to your
   own replica. Retrying a non-idempotent body here duplicates the work.
2. **SDK to a platform service**: Every `platform.*` call is container, then gateway, then service.
   The hop drops on a brief partition, a gateway bounce, or a service scaling from zero. The SDK
   retries transient failures with backoff. The workflow function sees success or a final exception.
   This layer's job is to get the request *to* the Model Service. Honor `is_idempotent` on a tool
   before the SDK retries Execute. A POST that charges the customer is not "transient" because the
   socket died.
3. **Model Service to the vendor**: A separate retry, inside the Model Service, whose job is to get
   a completion *from* the provider. A 429 from the vendor is not cured by immediately retrying the
   gRPC call to your own Model Service. That loop amplifies a quota problem.

* **Liveness**, `GET /health/live`: The process can still serve. Failure means restart me: deadlock,
  a wedged loop, an unrecoverable leak. A healthy replica almost never fails this probe. Failing
  live because a dependency is down restart-loops the fleet during a Model Service outage. Live
  means "this process is not dead."
* **Readiness**, `GET /health/ready`: This replica should receive traffic. During startup: import
  finished, SDK constructed, maybe a cheap connectivity check. During shutdown: fail ready first,
  drain, then exit. If Data is down you may fail ready, to stop new work, and stay live, to avoid a
  restart storm. A workflow that can still answer from cache may stay ready in a degraded mode.
  Write that choice down. One `/health` route collapses the two meanings, and you will restart pods
  that were only waiting on a dependency.

## 7. Calling another workflow

A self-contained intake assistant is the tutorial. The organization-scale pattern is to call the
workflow another team already shipped. Search exists. Summarization exists. Product wants both.
Copying the code copies none of the later improvements and all of the dependencies, including a GPU
box the parent never needed. Composition is how the platform becomes a network of capabilities.

* **`platform.workflows.call(api_path, body)`**: An HTTP POST through the gateway, the same door an
  external client uses. You get a dict back. The parent does not import the child's module. That
  import would put two teams in one container, one image, and one resource envelope. A
  ticket-responder shape: classify with a small model, `call("/knowledge/search", ...)`, draft with
  a larger model. Search stays with the team that tunes chunking.
* **Identity and trace**: The call forwards the caller's identity and `trace_id`. Drop them and the
  child trace cannot join the parent waterfall, and you have a confused deputy. Treat a child
  workflow as you treat a tool: least privilege. "Internal" is not an authentication policy.
* **Nested timeouts**: A parent timeout of 30s and a child timeout of 30s lets the child spend the
  entire budget. Set the parent above the sum of its synchronous children, or call them in parallel
  and give each child a tighter limit.
* **Cycles**: A calls B calls A. Detect or forbid the cycle. The registry can store the graph. A
  depth-first search on every call is a last resort.

Children that do not depend on each other should not run in series. `call_parallel([...])` fires
several HTTP requests, waits for all of them (or for a documented fail-fast), and returns a list
aligned with the inputs. A research assistant can fetch papers, news, and patents, then synthesize
once. Wall time is about the slowest child, plus synthesis, rather than the sum. Decide
all-or-nothing versus best effort with errors in the payload. Omitting a failed child in silence is
how you invent a source you never searched. This parallelism is across containers. Each child scales
on its own autoscaler, which is the point of refusing to stuff three libraries into the parent
image.

`call()` hides the child's mode so the parent author does not branch on it.

1. **HTTP 200 and JSON**: A sync child. Return the object.
2. **HTTP 200 and `text/event-stream`**: Consume the stream to completion, or to a documented
   truncated join, and return the assembled dict or the concatenated text. The parent waited for the
   whole stream. You gained composition. You did not gain a streaming user experience. If the parent
   must stream to its own caller, it yields while reading the child. That helper is `call_stream`.
   The teaching `call()` means "give me the answer."
3. **HTTP 202 and a job id**: Poll until the job is terminal, then return the result. The parent's
   `call()` blocks on that poll, with a timeout. Nested async without a cap becomes an unbounded
   wait.
4. **Anything else**: Raise, with the status and the body.

```
  call(path, body)
        |
        v
     HTTP POST
        +-- 200 JSON --> dict
        +-- 200 SSE  --> consume --> dict
        +-- 202      --> poll job --> dict
        +-- other    --> raise
```

## 8. Registry, deploy, and the gateway route

Other platform services have one consumer: workflow code, through the SDK. The Workflow Service has
three, and they are not equally trusted. The deploy CLI may register, deploy, roll back, and read
status. The runtime in a pod may create a job, report progress, complete, and fail. An external
client may read status and cancel. The public poller does not get `DeleteWorkflow`.

* **RegisterWorkflow**: How a workflow enters the platform. The spec carries name, `api_path`,
  image, `response_mode`, scaling, resources, and version. The CLI fills that spec from decorator
  metadata plus the image reference it just pushed. A new `api_path` is a breaking change for every
  caller and for the gateway. Bump the version. `DeleteWorkflow` is an explicit act, not garbage
  collection of the last replica. List and Get are how a portal shows the catalog. The registry is
  the catalog. The cluster is not.
* **Deploy**: Actuation. Take a workflow id and a version, create replicas, wait until they are
  ready, and register the route. The response includes a `deployment_id` and a status.
  `WorkflowDeployment` shows version, status, current versus desired replicas, and healthy
  endpoints. `GetDeploymentStatus` is what CI polls. `RollbackWorkflow` points traffic at a previous
  image and spec. Register without deploy is a saved spec. Deploy without register is how the next
  CLI run cannot find the image. Keep that order.

`genai-platform deploy` is the one command in front of six steps. The developer has something like
`patient_intake.py` with `@workflow` and runs it.

1. **Discover**: Import the module so the decorator runs, then scan for metadata. Several decorated
   functions become several workflows.
2. **Read metadata**: Name, path, mode, and scale. The same dict the runtime will read.
3. **Image recipe**: Generate a Dockerfile when the repo has none: a base image, the code, the
   dependencies, an `ENTRYPOINT` of the runtime server, and a `CMD` of the module. A Dockerfile the
   team supplied is used instead, for a GPU base or a system library.
4. **Build and push**: Tag the image with a version or a git sha and push it to the organization
   registry. The spec stores that reference.
5. **Register**: Send the spec to the Workflow Service.
6. **Deploy**: Wait until replicas are healthy and the route exists.

The CLI is a client. It should not drive the cluster as the developer. The Workflow Service talks to
the orchestrator. That split is what keeps one team's credentials from becoming cluster admin.

When replicas pass readiness, the Workflow Service tells the gateway: this `api_path` maps to these
endpoints. The table is dynamic. A new replica that is ready gets an address. A replica that fails a
probe, or that is draining, loses its address. A new version swaps addresses as the roll proceeds. A
gateway whose routes are a static file committed on Friday will not track that. An Ingress in front
of a Service, with the platform gateway never told about `api_path`, means SDK clients and the auth
middleware on the platform gateway never see the workflow. Two doors are two security stories.
Authentication and authorization at the gateway still apply. A deployed workflow is not
automatically public.

## 9. Keeping replicas alive

Deploy creates the process. Runtime management keeps it. Developers do not write manifests. The
decorator maps onto two orchestrator objects, plus the rules for a roll.

* **Deployment**: The image, `min_replicas` as the baseline, CPU and memory limits, GPU when you
  asked for it, probes on `/health/live` and `/health/ready`, and a restart policy. A crash
  reschedules the pod. A failed live probe replaces it.
* **HorizontalPodAutoscaler**: Watch CPU, and optionally memory, against `target_cpu_percent`, and
  clamp the count to `[min_replicas, max_replicas]`. Scale up faster than you scale down so a spike
  does not flap. The teaching decorator is `@workflow(min_replicas=1, max_replicas=10,
  target_cpu_percent=70)`: the Deployment may run from 1 to 10 pods, and the autoscaler adds pods as
  CPU approaches 70%, never above 10 and never below 1.
* **Scale to zero and GPU**: Product choices. A GPU assistant with `min_replicas=0` cold-starts in
  minutes. Sync chat in front of a person rarely wants that wait. An async research job might.
* **Rolling update**: New pods become ready, the route flips, old pods drain. In-flight sync
  requests need a drain timeout at least as long as the workflow timeout, or you return 502 for work
  that was going to succeed.
* **What CPU cannot see**: The control plane fills `healthy_endpoints` from probe results. An
  autoscaler that watches CPU misses a workflow that is token-bound and idle on gRPC. CPU stays low
  while latency explodes. Concurrency or queue depth is the later signal. CPU is a proxy for load.
  It is not the user's latency objective.

```
  @workflow(min_replicas=1, max_replicas=10, target_cpu_percent=70)
          |
          v
  Deployment: 1..10 pods, live + ready probes
  Autoscaler: CPU near 70% --> add pods (cap 10)
                             --> remove pods (floor 1)
```

A request to `/patient-assistant` then has a short life. The gateway authenticates, starts a trace,
and routes with the table the Workflow Service maintains. A ready replica binds JSON to the
function. The function calls Session, Data, Model, and Guardrails. The SDK retries a blip. Spans get
recorded because the platform services are traced. Sync returns JSON. Stream returns events. Async
returns 202, a job row, and a poll. Nobody wrote a Deployment by hand. The graph you might run
inside the function is application logic. The container, the route, and the job table are this
service.

* **The process, not the graph**: Flows, hubs, and handoffs can run inside the function. They are
  not the registry, and they are not a reason to skip the deploy contract.
* **Traces leave**: The runtime emits them. It does not store them. Job progress is not a substitute
  for a generation span.
* **The gateway stays the door**: This service feeds it routes. Authentication, quotas, and external
  HTTP stay at the gateway.
* **Not a general job bus**: You can run a slow AI workflow async. Rebuilding payroll on `@workflow`
  because the autoscaler looks handy copies an AI runtime into a problem that never needed one.

## Check yourself

1. Name the control plane and the data plane. For a sync POST, which one is off the path once the
   gateway has routed?
2. Which decorator fields become registry spec, which become replica and autoscaler settings, and
   which does only the runtime enforce? Where does `timeout_seconds` belong, and why is a Deployment
   spec the wrong home for it?
3. Pair one product moment with sync, one with stream, and one with async: an intake chat that shows
   tokens, an ICD-10 classify, an overnight chart review. For each, what breaks if you pick a
   different mode?
4. Name the three consumers of a job row. What breaks if that row lives only on the replica's disk?
5. During a Model Service outage, which probe should fail if you want to stop traffic without a
   restart storm? Why does "dependency down, so fail liveness" take the fleet with it?
6. Draw the three retry layers: incoming HTTP, SDK to a platform service, Model Service to the
   vendor. Which layer retries an OpenAI 429, and which layer makes that 429 worse if it retries
   blindly?
7. Why does `workflows.call` go through the gateway instead of importing the child's Python? Give
   one resource reason and one security reason.
8. A parent `call()`s a child whose `response_mode` is `async`. What does the parent actually wait
   on, and what timeout bug appears if nobody caps the poll?
9. List the six steps inside `genai-platform deploy`. At which step does `api_path` become a gateway
   route, and what happens to that table when a replica fails readiness?
10. What does calling a search team's workflow keep, that copying their Python into your image
    loses? What does a CPU autoscaler fail to see when the workflow is waiting on tokens and the
    process looks idle?
