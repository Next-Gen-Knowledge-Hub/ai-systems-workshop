# 2. SDK and API design

Companion notes for **Chapter 2** of *Designing AI Systems* (MEAP, Suhas
Suresha and Dewang Sultania; Manning). This MEAP's printed contents page is
**single-level** (nine chapter titles). The headings below follow the
chapter's actual topics.

Chapter 1 named the iceberg. This chapter is how a developer is supposed
to *touch* it without drowning. Services that nobody can call from a
function are a diagram. Services that every team wraps differently are
sprawl with extra YAML. Skip this chapter and later "Model Service"
notes look like gRPC homework. They are not. They are the other side of
a **developer contract**: one object, one decorator, one public HTTP
shape.

The Agents track is a different book. Dockerfiles, Compose, edge vs
API vs queues, threat models for a *single* agent process live in
[agents ch. 8](../../agents/8-deploying-agents/). Mention that folder.
Do not rewrite it. This folder stays on **how the organization exposes
workflows** — SDK, containers as the unit of isolation, a gateway,
sync / stream / async as *platform* modes.

## The mental model

```
  browser / mobile / partner / CLI
              |
              |  HTTPS
              |  POST /support-assistant   (or SSE, or job id)
              v
        +-----------+
        | GATEWAY   |  TLS, identity, quotas, health, path -> replica
        +-----+-----+
              |
              |  HTTP to *this* workflow's containers
              v
        +---------------------------+
        |  WORKFLOW CONTAINER       |
        |  @workflow handle(...)    |
        |  platform = GenAIPlatform |
        +-------------+-------------+
                      |
                      |  gRPC + routing metadata
                      |  (still via the gateway)
          +-----------+-----------+-----------+
          v           v           v           v
       models      sessions      data       tools / ...
```

The one sentence to remember: **the SDK is a lie that stays honest.**
`platform.sessions.get_or_create(user_id)` *feels* local. It is a
network call through a trust boundary. The decorator *feels* like
syntax. It is how the deploy tool learns name, path, replicas, and
mode. If either lie leaks (raw URLs in app code, three auth schemes,
workflows sharing a process), chapter 1's sprawl is back.

Interaction modes, isolation, and "HTTP at the edge / RPC inside"
sit in [`TRADEOFFS.md`](../../TRADEOFFS.md) (APIs table). Use this
folder for the *contract* those rows implement.

## Designing the ideal developer experience

Design the SDK backwards from a working assistant, not forwards from
Kubernetes. Chapter 1's Sam still has to ship. The question is what
he types *before* he understands the iceberg.

### Immediate productivity through effortless setup

**Problem** — First hour of an AI project is env files, client
constructors, and an argument about "which model wrapper we use."
Nobody has a conversation yet. The demo dies in setup.

**Solution** — A platform object with defaults. One import, one
constructor, a registered system prompt, a `chat` call. No replica
counts, no protobuf, no provider SDK. Hard things stay possible.
They are not *required* to get a reply.

```
  from genai_platform import GenAIPlatform

  platform = GenAIPlatform()                    # defaults, not a cluster
  platform.models.register_prompt(
      name="support_assistant",
      content="You help with orders and returns. No legal advice."
  )
  reply = platform.models.chat(system_prompt_name="support_assistant",
                               query=user_text)
```

That snippet is a **teaching shape**, not a vendor. Extra kwargs
(`gateway_url=`, timeouts) must not be required for the first green
run. If a new hire needs a wiki titled "local platform" to get a
completion, the DX contract is already broken.

### The conversation memory challenge

**Problem** — Follow-ups ("the laptop I mentioned," "those documents")
land as isolated prompts. Sam either stuffs a list in the handler or
becomes a DBA: tables, serialization, "what if the window fills."

**Solution** — Session as a service the SDK already knows.
`get_or_create` is one call. The workflow passes an id around. It does
not open Redis. Context-window policy is [chapter 4](../4-session-service/),
not a homework problem in the handler.

```
  session = platform.sessions.get_or_create(user_id, session_id)
  reply = platform.models.chat(..., session_id=session.id)
  platform.sessions.add_messages(session.id, [user_turn, assistant_turn])
```

Memory that "just works" is still *someone's* database. The DX move is
to make that someone the Session Service. If Sam copies a transcript
into a global `dict`, he has rebuilt the prototype that chapter 1
killed.

Two services, not one blob: **conversation state** here;
**organizational documents** in Data (next bullet). Mixing them is how
"as I mentioned yesterday" disappears into a PDF index.

### The organizational knowledge problem

**Problem** — The model never trained on this week's return window.
Hardcoding FAQs in the system prompt rot the day Legal ships a PDF.
Each team then invents a chunker.

**Solution** — Point the workflow at an **index** and search at
request time. Ingest is a platform pipeline. Sam chooses *which*
index and *which* filters. He does not choose embedding dimensions on
day one.

```
  hits = platform.data.search(
      index="support.policies",
      query=user_text,
      filters={"audience": "customer"},
  )
```

The Data Service is [chapter 5](../5-data-service/). This chapter only
needs the **feeling**: knowledge is a call, not a paste. When the PDF
changes, ingest changes the index; the workflow does not redeploy to
edit a string.

If search is optional sugar on a 4k prompt dump, you do not have
organizational knowledge. You have a slower prompt.

### The safety imperative

**Problem** — Policy lives in a paragraph ("never give medical or
legal advice"). Users ask anyway. Provider-side filters are not *your*
regulation. Reviewing every path by hand does not scale.

**Solution** — Guardrails as a service the workflow **calls**, with
named policies, before and after generation (and around tool args).
Configure once; enforce every request. Fail closed with a reason the
developer can log, not a mysterious empty string.

```
  inbound = platform.guardrails.validate_input(
      user_text, policies=["no_legal_advice", "pii_minimize"]
  )
  if not inbound.allowed:
      return inbound.user_message          # graceful redirect, not hang
```

Tools and policy engines are [chapter 6](../6-tools-and-guardrails/).
The DX claim here is: safety is not a prompt appendix. If the only
control is "the model is usually nice," you have chapter 1's
wake-up call on a timer.

### The external system integration challenge

**Problem** — Real answers need calendars, billing, tickets. Sam does
not want to become the EHR or ERP expert, and he must not put those
API keys in the workflow image.

**Solution** — Register tools once on the platform. The workflow
discovers schemas, passes them into generate, and `execute`s what the
model asked for. Credentials stay in a store the Tool Service
retrieves. The handler never sees the secret.

```
  platform.tools.register(
      name="commerce.scheduling.check_availability",
      description="...",
      credential_ref="sched_prod",         # reference, not the key
  )
  result = platform.tools.execute(name=..., arguments=...)
```

Agent graphs (which tool, whether to retry the *plan*) are the Agents
track. This chapter only needs: **calling a governed capability feels
like a method**, not like `requests.post` with a token from `.env`.

### The optimization dilemma

**Problem** — Prompt v3 "felt better." There is no dataset, no score,
no way to know if v4 will hurt the 5% of users who ask about
restocking fees. Shipping is a vibe.

**Solution** — Evaluation as a service: a set of questions with
ideal answers or key elements, criteria you can run offline, a
comparison across prompt names (or models) *before* the assistant
meets production traffic. The workflow keeps `system_prompt_name`;
the winner is a config change, not a rewrite of business logic.

Experimentation's full data model is [chapter 7](../7-observability/).
The DX requirement is: Sam can say "run this prompt against this
set" without standing up a notebook that talks to prod.

If you cannot name the examples you would lose sleep over, you are
not ready to A/B. You are ready to write the set.

### The multi-step coordination challenge

**Problem** — A real question needs *all* of the above, in order:
validate, load session, retrieve, generate, maybe execute tools,
validate output, persist. Each hop can fail. Sam's handler becomes
a state machine he did not want to own — retries, partial writes,
"did we already charge."

**Solution** — A **workflow** function that reads like the business
steps. The platform owns the boring coordination: how this function
becomes a service, how errors look on the wire, how traces stitch
the hops. Sam still writes the *sequence*. He does not write the
process manager.

```
  @workflow(name="support_assistant", api_path="/support-assistant")
  def handle(question, user_id, session_id=None):
      session = platform.sessions.get_or_create(user_id, session_id)
      if not platform.guardrails.validate_input(question, ...).allowed:
          return redirect
      hits = platform.data.search(index="support.policies", query=question)
      reply = platform.models.chat(..., session_id=session.id)
      platform.sessions.add_messages(...)
      return {"response": reply.content, "session_id": session.id}
```

That is still *your* product logic. Agent control planes stay in
[agents ch. 4](../../agents/4-multi-agent-systems/) and
[agents ch. 9](../../agents/9-agentic-loop/). Mention them. The
platform move: this function is the unit you will **deploy**.

### Synthesis: the developer experience contract

Read the bullets as a contract you can test in onboarding:

| Need                         | SDK should feel like              | Service that owns it   |
|------------------------------|-----------------------------------|------------------------|
| First reply today            | constructor + `chat`              | Model                  |
| Follow-ups                   | `get_or_create` + ids             | Session                |
| This week's policy           | `search` / `hybrid_search`        | Data                   |
| Domain policy                | `validate_input` / `_output`      | Guardrails             |
| Calendars and ledgers        | `register` / `execute`            | Tools                  |
| Which prompt actually works  | dataset + compare                 | Experiments            |
| All of it in one path        | `@workflow` function              | Workflow + gateway     |

If any row requires a second framework the SDK does not know, the
contract has a hole. Fill it in the platform, not in Sam's README.

## Code to container: the deployment story

A decorated function still has to **run**. Where? With whom? What
happens when marketing ships on Friday? How does the frontend call
it on Monday? Those answers decide whether the platform is a product
or a shared laptop.

### The shared process trap

**Problem** — Fastest path: import every `@workflow` into one
process. One deploy, one port, one "AI service."

That process is chapter 1's sprawl wearing a decorator:

- Marketing's memory leak is Sam's p99.
- A tight loop in a batch summarizer starves the latency-sensitive
  assistant.
- GPU for one workflow, tiny CPU for another, same cgroup fight.
- A public content generator and a workflow that sees customer PII
  share an address space. Isolation is a comment.

**Solution** — Treat "same process" as a lab convenience, never as
the production default. Independent teams need **independent
failure domains**. If you cannot crash one workflow without paging
the other, you do not have a platform. You have a monolith with
extra prompts.

### Choosing the right isolation mechanism

Three honest options. None is free.

```
  PLAIN PROCESSES          VIRTUAL MACHINES         CONTAINERS
  ----------------         ----------------         ----------
  start: milliseconds      start: minutes           start: seconds
  isolation: weak          isolation: strongest     isolation: strong
  overhead: tiny           overhead: whole OS       overhead: image +
                                                      namespaced view
  security: shared         security: hardware-      security: kernel
    kernel, easy noisy       backed boundary          namespaces +
    neighbor                                          cgroups
```

**Problem** — Processes are cheap and leaky (CPU, files, network).
VMs are sealed and too fat when you have dozens of assistants.
You need a default you can operate.

**Solution** — **Containers** as the teaching default: package the
workflow with its deps, cap CPU/memory, give it a network policy,
start fast enough to autoscale. You still share a kernel — that is
the remaining risk; mitigate with images you control, non-root,
egress rules. Do not pretend a container is a VM. Do not pretend a
process is a container.

Agents ch. 8 will talk Docker from the *agent author's* chair
(compose a loop, expose a port). This chapter's question is
organizational: **every workflow image is a unit the Workflow
Service can schedule.** Same noun (container), different job.

### One workflow, one service

**Problem** — "All bots in one chart" and "the support bundle" as one
Deployment. Releases couple. Scaling couples. Versioning couples.
You are back in the shared process, one YAML later.

**Solution** — **One workflow = one independently deployable
service.** The decorator's `name` is the unit the registry tracks.
Sam ships v2 of the assistant while marketing stays on v1. Traffic
shift is per workflow. Scale is per traffic shape (chatty vs batch).

```
  @workflow("support_assistant")     -->  Deployment support-assistant
  @workflow("returns_classifier")    -->  Deployment returns-classifier
  @workflow("promo_copy")            -->  Deployment promo-copy
```

You will have more services. That is the bill. The refund is: no
release train across unrelated product logic. The Workflow Service
([chapter 8](../8-workflow-service/)) exists *because* of this
mapping. This chapter only needs the principle. If you group "to
keep the cluster small," measure coordination cost, not pod count.

### Managing execution

**Problem** — Containers do not babysit themselves. Something has
to start them, notice they are sad, add replicas at 70% CPU, drain
them on deploy, point the gateway at healthy ones.

**Solution** — A **Workflow Service** as operator: registry
(image, resources, min/max replicas, health path), lifecycle
(create, replace, revert), scaling (metrics in, replica count
out), discovery (so the gateway is not a hardcoded IP list).

```
  Sam's function
       |
       v
  build image (deps + entrypoint)
       |
       v
  Workflow Service: register + desired replicas
       |
       +-- replica A  (healthy)
       +-- replica B  (healthy)
       +-- replica C  (starting)
              ^
              |  gateway load-balances external HTTP here
```

Internal calls (`platform.models.chat`) do not go "to localhost
magic." They go through the same communication architecture you
will pin below. Resource limits are not optional decoration;
without them one workflow *is* the noisy neighbor again.

Stop imagining `uvicorn main:app` on a box as the end state.

## Exposing workflows as APIs

Containers that nobody can call are expensive unit tests. Frontends,
mobile, and partners need a **contract**. That contract is not "pick
JSON-RPC vs gRPC vs GraphQL in Slack." It is: one way in, three
**timing** shapes AI actually needs.

### The API sprawl problem

**Problem** — Each team exposes "their" bot.

```
  support.internal/api/v1/ask     API key
  copy.marketing.io/generate/v2   OAuth2
  ops-docs.company.net/analyze    homemade JWT

  dashboard client: 3 base URLs, 3 auth, 3 error dictionaries
```

This is chapter 1 sprawl on the **wire**. Frontend becomes an
integration team. Auth changes become a program. Docs live in three
wikis. A unified "AI portal" is a political project.

**Solution** — The platform **owns exposure**: one hostname, one
identity story, one error envelope, one observability header. Teams
still choose the **path** (the public name of *their* assistant).
They do not choose a second protocol religion. A raw ALB "just this
once" becomes the estate.

### Unified workflow exposure

**Problem** — If paths are generated from function names, they
churn when Sam renames a symbol. If paths are implicit, two
workflows collide. If paths are free-form *and* ungoverned, you
recreate sprawl under one host.

**Solution** — The decorator declares `api_path`. That string is a
**public contract**, versioned like any API. Internal image names
can change. The path should not.

```
  @workflow(
      name="support_assistant",          # registry identity
      api_path="/support-assistant",     # public contract
  )
```

Gateway maps `POST https://api.example.com/support-assistant` to
Sam's replicas. Authn/authz wrap that map. OpenAPI (or equivalent)
can be generated from the registry so the dashboard team learns
*one* catalog.

Stability rule: changing `api_path` is a breaking release. Changing
Python locals is not. Keep them separate on purpose.

### The communication protocol

**Problem** — Internal fashion says "gRPC all the way to the
browser." Mobile and a partner PHP app disagree. Custom binary
protocols need custom clients. You wanted fewer wrappers.

**Solution** — **HTTP** at the trust boundary. Everyone already
speaks it. POST for "do the workflow," GET for job status, standard
status codes, Authorization headers you can hang a gateway on.
`curl` is the compatibility test.

AI still does not fit naive request/response:

- Some calls finish in two seconds (classify this image).
- Some should paint tokens as they exist (a paragraph of help).
- Some run for minutes (summarize a pack, research a ticket).

Protocol (HTTP) is the **transport**. Interaction **pattern** is
the next section. Do not collapse them. HTTP can carry all three
patterns. gRPC belongs *inside*, where you control both ends.

### Patterns of AI interaction

Three shapes. Pick per workflow, declare it, do not make every
client invent a fourth.

```
  SYNC                         STREAM                      ASYNC
  ----                         ------                      -----
  POST                         POST + SSE                  POST -> job_id
  wait for JSON                tokens as events            GET job / GET result
  timeout ~ seconds            timeout ~ tens of seconds   timeout ~ hours
  classify, extract            chat UX                     research, batch
```

**Synchronous.** Client waits. Fits when p95 is comfortably under
your gateway and load-balancer idle timeouts, and a spinner is
honest. Image tagging, short classifications, "is this message
toxic." If you put a 90-second agent loop here, you will debug
504s that are not model bugs.

**Streaming.** Same POST, different body: keep the connection
open, push fragments (SSE is the usual web-native choice:
unidirectional, simple, good enough when the server talks and the
client listens). Time-to-first-token is the UX metric. Errors
*after* tokens have landed are a different species — handle them
in [chapter 3](../3-model-service/) as stream failures, not as a
clean JSON error the client never sees.

```
  event: token   data: {"token": "Please"}
  event: done    data: {"session_id": "..."}
```

**Asynchronous.** Accept work, return `job_id`, let the client
poll (or webhook if you must). Progress is a first-class field,
not a log line. This is how you survive multi-minute jobs without
lying about HTTP. Exactly-once is a lie; **idempotent job keys**
are the practical cousin — Agents ch. 8 talks that language for
queues. Here: the platform must *have* a job record, not a thread
you hope is still running.

Declare `response_mode` on the decorator so gateway and SDK agree.
A yielding workflow is not sync JSON. Measure timeouts against
*your* proxy chain ([`TRADEOFFS.md`](../../TRADEOFFS.md)).

## The communication architecture

Two different questions people mash together:

1. What receives Maria's phone request at a stable URL?
2. What does Sam's container use to talk to Session and Model?

External traffic crosses **untrusted** networks. Internal traffic
is service-to-service on your fabric. Same HTTP everywhere is a
comfort choice, not a physics result.

### The entry point problem

**Problem** — Replicas come and go. TLS, JWT/API keys, per-tenant
quotas, load balancing, canary weights, health probes: if every
workflow implements them, you have N half-broken gateways and no
place to revoke a key.

**Solution** — One **API gateway** (fleet of stateless instances)
as the only public listener.

```
  Maria's app
       |  HTTPS POST /support-assistant
       v
  GATEWAY  -- authn --> authz --> rate limit --> choose healthy replica
       |
       v
  workflow container
```

The gateway is allowed to be boring. Boring is the feature. Workflow
authors do not terminate TLS. They do not parse JWTs. They see an
already-authenticated request (or they fail closed).

Internal callers (another workflow, a batch job) still enter through
this edge if you want one policy plane. "Skip the gateway, hit the
pod IP" is how prod and "the path we tested" diverge.

### The internal communication challenge

**Problem** — Each user click fans out: session load, search,
generate, maybe tools. If every hop is HTTP+JSON, you pay parse and
schema chaos on the hot path. If Sam must build URLs and headers by
hand, the SDK failed.

**Solution** — **gRPC** (Protocol Buffers) for service-to-service.
The SDK method *looks* like a function. Underneath: typed binary,
generated stubs, deadlines, status codes. The gateway can **proxy**
those bytes to the right service using metadata (`x-target-service:
sessions`) without turning them into JSON and back.

```
  platform.sessions.get_or_create(user_id)
        |
        v
  SDK stub  -- gRPC -->  GATEWAY (route by metadata)  -->  Session Service
```

Why not gRPC to the phone? Because the phone is not in your proto
build. Why not JSON inside? You can, at a cost. The teaching
architecture is: **HTTP/JSON (or SSE) west of the gateway, gRPC
east of it.** Mixing randomly is how you debug two tracers and
still miss the span.

### Tracing a complete flow

Maria: "What is the return window for the laptop I already asked
about?"

1. App sends HTTPS POST `/support-assistant` with her token and
   `{question, user_id, session_id}`.
2. Gateway authenticates, attaches a **trace id**, rate-limits the
   key, picks a healthy support-assistant replica.
3. Workflow: `get_or_create` — gRPC to gateway → Session Service →
   Postgres (or whatever backend chapter 4 chose). History returns.
4. `data.search` — same pattern → Data Service → chunks from the
   current policy index, not last year's prompt FAQ.
5. `models.chat` — Model Service (maybe stream). Tokens may flow
   back as gRPC stream → gateway SSE → app.
6. Guardrails on the way in and out. `add_messages` persists the
   turn. Observability already has the trace id from step 2.

```
  app --HTTPS--> GW --HTTP--> workflow
                    ^              |
                    |     gRPC     |
                    +-- GW --------+--> session
                    +-- GW --------+--> data
                    +-- GW --------+--> models
```

If you cannot draw this without writing `openai.chat()` in the
middle, you are still in the 2% leaf from
[chapter 1](../1-why-a-platform/).

### Why these decisions scale

**Problem** — Designs that work at 5 rps often require a rewrite
at 500. People then blame "microservices."

**Solution** — Scale the **stateless** edge horizontally (more
gateway pods behind a load balancer). Scale **each** platform
service on *its* bottleneck (session reads vs token generation vs
ingest). Workflows scale on *their* traffic. None of that requires
Sam to learn a service mesh on day one; it requires that state
not live in the gateway and that routing be data, not a
hardcoded host list.

```
  more Maria QPS
       |
       v
  N gateway instances (stateless)
       |
       +--> more support-assistant replicas     (chat)
       +--> more model-service replicas         (GPU/API fanout)
       +--> more session-service replicas       (read-heavy)
```

Canary and revert are cheap *because* one workflow is one service.
A bad model bump in marketing does not require Sam's replicas to
roll. That is the payoff of the isolation fight, not a slogan.

## Building the SDK

The rest of the book assumes this object exists. Build it as three
jobs, not as a kitchen-sink "AI class."

```
  GenAIPlatform
  -------------
  (1) Where is the gateway?  env or ctor
  (2) Lazy clients: .models .sessions .data .tools ...
  (3) @workflow captures deploy metadata for the ship tool
```

### Connection to the API Gateway

**Problem** — Workflow code runs in *its* container. Session runs
in another. "Just call localhost" is a lie in prod and a trap in
dev (it works until you add a second replica).

**Solution** — The platform object holds **gateway address, TLS,
and credentials for the mesh.** Production: stable internal DNS.
Laptop: `localhost` or a port-forward. Same constructor. Clients
are created against that address, not against `SESSION_HOST`,
`MODEL_HOST`, `DATA_HOST` (that list is sprawl).

```
  platform = GenAIPlatform()                    # GATEWAY_URL from env
  platform = GenAIPlatform(gateway_url="...")   # explicit, tests/dev
```

No sockets until first use (next section). Constructor failures
should be about *config*, not about a Session Service that is
still booting — or you will make local startup miserable.

### Providing access to services

**Problem** — If Sam writes HTTP by hand, every team invents a
fourth Session client. If every client is created at import time,
importing the SDK needs the whole cluster.

**Solution** — **Lazy properties.** First access to
`platform.sessions` builds a `SessionClient`: channel to the
gateway, metadata `x-target-service: sessions`, proto stub.
Methods convert Python objects → protobuf → stub → Python
objects. Sam never sees the wire types unless he wants to.

```
  handle() calls platform.sessions.get_or_create
        |
        v
  first time: SessionClient(channel, metadata)
        |
        v
  stub.GetOrCreateSession(request)
        |
        v
  gateway routes binary to Session Service
```

Same skeleton for Model, Data, Tools. **Learn once.** Chapter 3's
`ModelClient` is not a new religion; it is this paragraph with a
different stub.

Deadlines and retries at *this* layer are for transport. Product
retries (try Claude if OpenAI 503s) belong in the Model Service,
not in every workflow's `try/except`.

### Capturing deployment information

**Problem** — The ship tool cannot guess `api_path`, replica
policy, or sync vs stream from a bare `def handle`. Comments in
Slack are not a schema.

**Solution** — `@workflow(...)` attaches a struct to the function:
`name`, `api_path`, `min_replicas`, `max_replicas`,
`target_cpu_percent`, `response_mode`. The deploy CLI imports the
module, reads the struct, builds the image, registers with the
Workflow Service. No second YAML *required* for the happy path
(you can still overlay for prod-only limits).

```
  @workflow(
      name="support_assistant",
      api_path="/support-assistant",
      min_replicas=2,
      max_replicas=20,
      target_cpu_percent=70,
      response_mode="stream",
  )
  def handle(...):
      ...
```

If the decorator and the gateway disagree (function yields, mode
says sync), fail **at deploy**, not at Maria's first token.

### The SDK foundation

Everything later hangs on this:

- One gateway connection story.
- One client pattern (metadata + stub + typed methods).
- One decorator that is the deploy API.

When [chapter 3](../3-model-service/) adds `chat_stream`, it is a
new method on `ModelClient`, not a new way to find a host. When
[chapter 4](../4-session-service/) adds memories, it is new RPCs
on `SessionClient`. Consistency is how the fifth team does not
invent a fifth wrapper.

## What this chapter is not

Not Workflow Service internals ([chapter 8](../8-workflow-service/)).
Not agent loops or a personal Docker runtime
([agents ch. 8](../../agents/8-deploying-agents/)). Not the
observability store — the gateway **starts** a trace;
[chapter 7](../7-observability/) keeps it.

## See also

Same words, different job — do not paste those chapters here.

- **[agents ch. 8 — Deploying agents](../../agents/8-deploying-agents/)**
  Runtimes, Compose, queues, threat model. This folder is org-shaped
  exposure of workflows.
- Sync vs stream vs async: [`TRADEOFFS.md`](../../TRADEOFFS.md).
- Blueprint: [chapter 1](../1-why-a-platform/).

## Check yourself

1. List the seven rows of the DX contract. For each, name the
   *service* that should own it and one smell that the SDK leaked
   (Sam is writing infrastructure again).
2. Why is "we wrapped OpenAI in a Python class" not an SDK in this
   chapter's sense? What third job (besides calling a model) is
   missing?
3. Shared process vs one-workflow-one-service: give one *security*
   failure and one *scaling* failure of the shared process. What
   do you pay for the fix?
4. Processes, VMs, containers: pick isolation, startup, and one
   remaining risk of your choice. Why is "containers are VMs" a
   lie you should not tell security?
5. Draw API sprawl with three workflows. What does the frontend
   team maintain? Which fields of `@workflow` prevent the sprawl
   *without* taking away path design?
6. A classify-image call, a 40-second chat, a 20-minute research
   job: assign sync / SSE / async. What breaks if you put the
   research job in sync behind a 60s load balancer?
7. Why is HTTP the *external* protocol and gRPC the *internal*
   one? Give a client you refuse to force onto protobuf, and a
   cost you refuse to pay on the session-load hot path.
8. Trace Maria's question from HTTPS to a model token. Where is
   the trace id born, and why should `platform.sessions` still
   go *through* the gateway instead of to a pod IP?
9. Lazy clients vs eager connect-on-import: which local-dev
   failure does lazy avoid? When would you still want a
   readiness check before serving traffic?
10. This workshop keeps Agents and Platform separate. In one
    sentence, what would you still open
    [agents ch. 8](../../agents/8-deploying-agents/) for after
    this folder — and what would you *not* copy into a workflow
    service design?

Continue to [The Model Service](../3-model-service/).
