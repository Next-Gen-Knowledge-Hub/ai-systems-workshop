# 2. SDK and API design

Companion notes for **Chapter 2** of *Designing AI Systems* (Suhas Suresha and Dewang
Sultania; Manning MEAP).

This chapter establishes how a developer calls the platform. One object, one decorator, one
public HTTP shape. Afterward you can name the day-one calls, why each workflow is its own
container, which of sync, stream, or async the path declares, what the gateway terminates,
and the three jobs inside `GenAIPlatform`. Without that contract, every team wraps the same
services differently, workflows share a process, and a dashboard keeps three base URLs,
three auth schemes, and three error dictionaries.

## 1. The developer experience contract

Design the SDK backward from a working assistant. Sam still has to ship. The question is
what he types before he can draw the rest of the stack. `platform.sessions.get_or_create(user_id)`
feels local and is a network call across a trust boundary. The decorator feels like syntax
and is how the deploy tool learns the name, the path, the replicas, and the mode. The
contract leaks when app code holds raw URLs, when three auth schemes appear, or when
workflows share a process.

Read the seven rows as a contract you can test in onboarding. If a row needs a second
framework the SDK does not know, fill the hole in the platform.

| Need | SDK should feel like | Service that owns it |
|---|---|---|
| First reply today | constructor + `chat` | Model |
| Follow-ups | `get_or_create` + ids | Session |
| This week's policy | `search` / `hybrid_search` | Data |
| Domain policy | `validate_input` / `validate_output` | Guardrails |
| Calendars and ledgers | `register` / `execute` | Tools |
| Which prompt actually works | evaluate + compare | Experiments |
| All of it in one path | `@workflow` function | Workflow + gateway |

* **First reply**: One import, one `GenAIPlatform()` with defaults, a registered system
  prompt, and `models.chat`. The teaching prompt is `name="support_assistant"` with content
  "You help with orders and returns. No legal advice." The call passes `system_prompt_name`
  and `query`. Replica counts, protobuf, and a provider SDK stay available and stay off the
  first green run. `gateway_url=` and timeouts are extra kwargs. Use this shape for the
  first hour, before anyone has a conversation. It breaks when a new hire needs a wiki
  titled "local platform" to get a completion.
* **Follow-ups**: "The laptop I mentioned" and "those documents" are session state.
  `get_or_create(user_id, session_id)` returns an id, `chat` takes `session_id`, and
  `add_messages(session.id, [user_turn, assistant_turn])` appends the turn. Context-window
  policy lives in the Session Service. Use it whenever a later turn depends on an earlier
  one. It breaks when Sam opens Redis himself, or copies the transcript into a global
  `dict`, which rebuilds the prototype production already outgrew. Conversation state and
  organizational documents stay in different services. Mix them and "as I mentioned
  yesterday" disappears into a PDF index of policies.
* **Organizational knowledge**: The model did not train on this week's return window.
  Hardcoded FAQs in the system prompt rot the day Legal ships a PDF, and each team then
  invents a chunker. Point the workflow at an index and search at request time. Ingest is a
  platform pipeline. Sam chooses the index and the filters. Embedding dimensions wait.
  `data.search(index="support.policies", query=user_text, filters={"audience": "customer"})`,
  and `hybrid_search` when the index needs both. Use it for facts that change on a document
  schedule. It breaks when search is optional sugar on a 4k prompt dump: that is a slower
  prompt. When the PDF changes, ingest updates the index. The workflow does not redeploy to
  edit a string.
* **Guardrails**: A paragraph that says "never give medical or legal advice" meets users who
  ask anyway. Provider-side filters enforce the provider's rules. Hand review of every path
  does not scale. The workflow calls Guardrails with named policies before generation, after
  generation, and around tool arguments.
  `validate_input(user_text, policies=["no_legal_advice", "pii_minimize"])`, and
  `validate_output` on the way out. Configure once and enforce every request. Fail closed:
  `inbound.allowed` is false, so return `inbound.user_message`, a redirect the developer can
  log. Use named policies wherever a domain rule can outrun a prompt. It breaks when the
  only signal is a mysterious empty string, or when the only control is "the model is
  usually nice."
* **Tools**: Real answers need calendars, billing, and tickets. Sam does not have to become
  the EHR or ERP expert, and those API keys stay out of the workflow image. Register once:
  `name="commerce.scheduling.check_availability"`, a description, and
  `credential_ref="sched_prod"` (a reference; the key itself stays in the store). The
  workflow discovers schemas, passes them into generate, and `execute`s what the model
  asked for. The Tool Service retrieves the credential. The handler never sees the secret.
  Use this for a capability the organization already governs. It breaks when the handler
  does `requests.post` with a token from `.env`. Which tool to pick, and whether to retry
  the plan, sits with agent control. This contract's claim is narrower: a governed
  capability feels like a method.
* **Experiments**: Prompt v3 "felt better." There is no dataset and no score, so v4 can hurt
  the 5% of users who ask about restocking fees. Evaluation is a service: a set of
  questions with ideal answers or key elements, criteria Sam can run offline, and a
  comparison across prompt names or models before production traffic. The workflow keeps
  `system_prompt_name`. The winner is a config change. Use it when you can say "run this
  prompt against this set" without a notebook that talks to prod. It breaks when you A/B
  before you can name the examples you would lose sleep over.
* **Workflow**: A real question needs the rows in order: validate, load session, retrieve,
  generate, maybe execute tools, validate output, persist. Each hop can fail. The handler
  then becomes a state machine Sam did not want: retries, partial writes, "did we already
  charge." An `@workflow` function keeps the product sequence in Sam's hands. The platform
  owns how that function becomes a service, how errors look on the wire, and how traces
  stitch the hops. Sam writes the sequence. The platform writes the process manager. Use
  the function as the unit you deploy.

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

That function is still the product logic. It is also the unit the next section deploys.

## 2. Code to container

A decorated function still has to run somewhere, survive marketing's Friday deploy, and
answer the frontend on Monday. Where it runs decides whether the platform is a product or
a shared laptop.

* **Shared process**: Import every `@workflow` into one process: one deploy, one port, one
  "AI service." That shape is a lab convenience. In production it is one failure domain
  wearing a decorator. Marketing's memory leak becomes Sam's p99. A tight loop in a batch
  summarizer starves the latency-sensitive assistant. A GPU workflow and a tiny-CPU workflow
  fight inside the same cgroup. A public content generator and a workflow that sees customer
  PII share an address space. Independent teams need independent failure domains. If you
  cannot crash one workflow without paging the other, the platform is a monolith with extra
  prompts.

Three isolation mechanisms. Pick a default you can operate. Dozens of assistants make the
fat option expensive and the leaky option unsafe.

1. **Processes**: Start in milliseconds. Isolation is weak and overhead is tiny. They share
   a kernel and leak CPU, files, and network, so a noisy neighbor is easy. Use them for a
   local loop. They break as the production default.
2. **Virtual machines**: Start in minutes. Isolation is the strongest of the three, with a
   hardware-backed boundary, and the overhead is a whole OS. Use them when that boundary is
   the requirement. They break when you have dozens of assistants and boot time plus image
   size dominate.
3. **Containers**: Start in seconds. Isolation is strong: kernel namespaces plus cgroups,
   and the overhead is an image plus a namespaced view. Package the workflow with its
   dependencies, cap CPU and memory, attach a network policy, and start fast enough to
   autoscale. They are the teaching default. Every workflow image is a unit the Workflow
   Service can schedule. They still share a kernel. Mitigate with images you control, a
   non-root user, and egress rules. A container shares that kernel with its neighbors; a VM
   does not, so "a container is a VM" is the wrong sentence for security. A plain process
   has no image, no namespace boundary, and no cgroup cap. From the agent author's chair,
   Docker means compose a loop and expose a port. The organizational question is the
   schedule unit.

* **One workflow, one service**: "All bots in one chart" and "the support bundle" as one
  Deployment couple releases, scaling, and versioning. **One workflow = one independently
  deployable service.** The decorator's `name` is the unit the registry tracks. Sam ships
  v2 of the assistant while marketing stays on v1. Traffic shift is per workflow. Scale
  follows the traffic shape, chatty versus batch. `@workflow("support_assistant")` maps to
  Deployment `support-assistant`, `returns_classifier` to `returns-classifier`, and
  `promo_copy` to `promo-copy`. Use the split when teams release on different clocks. The
  bill is more services. The refund is no release train across unrelated product logic. The
  Workflow Service exists because of this mapping. Grouping workflows to keep the cluster
  small is a pod-count decision. The number that tells you whether it worked is
  coordination cost.
* **Workflow Service**: Containers do not babysit themselves. The service is the operator.
  The registry stores the image, resources, min and max replicas, and the health path.
  Lifecycle is create, replace, and revert. Scaling takes metrics in and emits a replica
  count out, including adding replicas at 70% CPU. Discovery means the gateway is not a
  hardcoded IP list. The service starts replicas, notices the unhealthy ones, drains them
  on deploy, and points the gateway at healthy ones. Internal calls such as
  `platform.models.chat` travel that same path. A localhost shortcut is the lab story.
  Resource limits belong in the registry. Without them, one workflow is the noisy neighbor
  again. `uvicorn main:app` on a box is a lab process. The production end state is this
  registry.

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

## 3. Sync, stream, and async APIs

Containers nobody can call are expensive unit tests. Browsers, mobile apps, partners, and
CLIs need a contract: one way in, and three timing shapes. The public call is HTTPS, a
POST such as `/support-assistant`, or SSE, or a job id.

* **API sprawl**: Each team exposes "their" bot. `support.internal/api/v1/ask` takes an API
  key, `copy.marketing.io/generate/v2` takes OAuth2, and `ops-docs.company.net/analyze`
  takes a homemade JWT. The dashboard client then keeps 3 base URLs, 3 auth schemes, and 3
  error dictionaries. The frontend becomes an integration team, an auth change becomes a
  program, and docs live in three wikis. A unified portal turns into a political project.
  The platform owns exposure: one hostname, one identity story, one error envelope, one
  observability header. Teams still choose the path, the public name of their assistant.
  They share one protocol. A raw ALB "just this once" becomes the estate.
* **`api_path`**: A path generated from a function name churns when Sam renames a symbol.
  An implicit path lets two workflows collide. A free-form path with no owner recreates
  sprawl under one host. The decorator declares `api_path`, a public contract versioned
  like any API. `name="support_assistant"` is the registry identity.
  `api_path="/support-assistant"` is the public contract. Internal image names can change.
  The path stays. The gateway maps `POST https://api.example.com/support-assistant` to
  Sam's replicas. Authn and authz wrap that map. OpenAPI, or an equivalent catalog, can be
  generated from the registry so the dashboard learns one catalog. Changing `api_path` is a
  breaking release. Changing Python locals stays a local edit.
* **HTTP at the trust boundary**: Mobile clients and a partner PHP app already speak HTTP.
  A custom binary protocol needs a custom client, which is another wrapper. POST means "do
  the workflow." GET reads job status. Standard status codes and `Authorization` headers
  give the gateway something to hang policy on. `curl` is the compatibility test. Some
  calls finish in about two seconds (classify this image). Some should paint tokens as they
  exist (a paragraph of help). Some run for minutes (summarize a pack, research a ticket).
  HTTP is the transport. The interaction pattern is a separate choice. HTTP can carry all
  three patterns. gRPC stays inside, where you control both ends.

Declare one pattern per workflow as `response_mode`, so the gateway and the SDK agree. A
yielding workflow is a stream, and a sync mode that says JSON will disagree with it.
Measure timeouts against your proxy chain.

```
  SYNC                         STREAM                         ASYNC
  POST, wait for JSON          POST + SSE                     POST -> job_id
  timeout ~ seconds            tokens as events               GET job / GET result
  classify, extract            timeout ~ tens of seconds      timeout ~ hours
                               chat UX                        research, batch

  event: token   data: {"token": "Please"}
  event: done    data: {"session_id": "..."}
```

1. **Synchronous**: The client waits for JSON. Use it when p95 sits comfortably under the
   gateway and load-balancer idle timeouts, and a spinner is an honest wait. Image tagging,
   short classification, and "is this message toxic" live here. A 90-second agent loop on
   this pattern shows up as HTTP 504s. Those 504s are proxy timeouts, and the model may be
   fine.
2. **Streaming**: The same POST, with the connection held open and fragments pushed out.
   SSE is the usual web-native choice: unidirectional, simple, and enough when the server
   talks and the client listens. The UX metric is time-to-first-token. Use it for chat that
   should paint a paragraph as tokens exist, on a timeout of tens of seconds. An error
   after tokens have landed is a stream failure. The client will not receive a clean JSON
   error for bytes it already rendered.
3. **Asynchronous**: Accept the work, return `job_id`, and let the client poll. Offer a
   webhook when completion has to be pushed. Progress is a first-class field on the job
   record, which is what the client polls. Use it for research and batch work whose timeout
   is measured in hours, so a multi-minute job survives ordinary HTTP. Exactly-once delivery
   is unreliable here. Idempotent job keys are the practical stand-in. The platform holds a
   job record. A thread you hope is still running is an absent job record.

## 4. The gateway

Two questions stay separate. What receives Maria's phone request at a stable URL? What
does Sam's container use to reach Session and Model? External traffic crosses untrusted
networks. Internal traffic is service-to-service on your fabric. The public listener is
the gateway: TLS, identity, quotas, health, and a path mapped to a replica. Behind it, the
workflow container runs `@workflow` and a `GenAIPlatform`. Calls onward to models,
sessions, data, and tools are gRPC plus routing metadata, and they still pass through the
gateway.

* **One API gateway**: Replicas come and go. TLS, JWT or API keys, per-tenant quotas, load
  balancing, canary weights, and health probes belong in one place. If every workflow
  implements them, you get N half-broken gateways and nowhere to revoke a key. One gateway,
  a fleet of stateless instances, is the only public listener. Maria's app sends
  `HTTPS POST /support-assistant`. The gateway runs authn, then authz, then a rate limit,
  then picks a healthy replica. The gateway is allowed to be boring. Workflow authors do
  not terminate TLS and do not parse JWTs. They see an already-authenticated request, or
  the request fails closed. Another workflow, or a batch job, enters through this edge when
  you want one policy plane. A call straight to a pod IP is a path production and the test
  no longer share.
* **gRPC inside**: One click fans out to a session load, a search, a generate, and maybe
  tools. HTTP and JSON on every hop pay for parsing and invite a private schema on the hot
  path. Hand-built URLs and headers mean the SDK missed its job. Service-to-service uses
  gRPC and Protocol Buffers: typed binary, generated stubs, deadlines, and status codes.
  The SDK method still looks like a function.
  `platform.sessions.get_or_create(user_id)` hits an SDK stub, then gRPC, then the gateway
  routes on metadata `x-target-service: sessions`, then the Session Service. The gateway
  proxies those bytes through, with no JSON round trip. The phone stays on HTTP because the
  phone is outside your proto build. JSON on the inside is possible, and the cost is the
  parse plus a second schema. The teaching architecture is HTTP/JSON or SSE west of the
  gateway, and gRPC east of it. Mixing the two at random produces two tracers and a missing
  span.

Maria asks, "What is the return window for the laptop I already asked about?"

1. The app sends HTTPS POST `/support-assistant` with her token and
   `{question, user_id, session_id}`.
2. The gateway authenticates, attaches a trace id, rate-limits the key, and picks a healthy
   support-assistant replica. The trace id is born here.
3. `get_or_create` goes gRPC to the gateway, then to the Session Service, then to Postgres
   or whichever backend Session chose. History returns.
4. `data.search` follows the same pattern to the Data Service. Chunks come from the current
   policy index that ingest maintains. Last year's prompt FAQ is a stale string left in the
   workflow.
5. `models.chat` hits the Model Service, and may stream. Tokens can return as a gRPC
   stream, then gateway SSE, then the app.
6. Guardrails run on the way in and the way out. `add_messages` persists the turn.
   Observability already holds the trace id from step 2.

```
  app --HTTPS--> GW --HTTP--> workflow
                    ^              |
                    |     gRPC     |
                    +-- GW --------+--> session
                    +-- GW --------+--> data
                    +-- GW --------+--> models
```

The model call is one hop. A diagram whose middle box is `openai.chat()` is still the 2%
leaf. The path is the system.

* **Scale the planes separately**: A design that holds at 5 requests per second often needs
  a rewrite at 500, and the rewrite gets blamed on microservices. Scale the stateless edge
  horizontally: N gateway instances behind a load balancer. Scale each platform service on
  its bottleneck. Session reads, token generation, and ingest are different bottlenecks.
  Scale each workflow on its own traffic. More Maria QPS means more support-assistant
  replicas for chat, more model-service replicas for GPU or API fanout, and more
  session-service replicas because that path is read-heavy. Sam can ship on day one without
  a service mesh. State stays out of the gateway. Routing is data in a registry the gateway
  reads, so a hardcoded host list is what goes stale when replicas move. Canary and revert
  stay cheap because one workflow is one service. A bad model bump in marketing rolls
  marketing's replicas and leaves Sam's in place. That is the payoff of the isolation
  decision.

## 5. Building the SDK

Later platform work assumes this object exists. Build it as three jobs.

1. **Gateway address**: Where is the gateway? An environment variable, or the constructor.
2. **Lazy clients**: `.models`, `.sessions`, `.data`, `.tools`, and the rest, created on
   first use.
3. **`@workflow`**: The decorator captures deploy metadata for the ship tool.

* **One address**: Workflow code runs in its container. Session runs in another. Calling
  localhost works until the second replica, so it is a trap in dev and a false story in
  production. The platform object holds the gateway address, TLS, and credentials for the
  mesh. Production uses stable internal DNS. A laptop uses `localhost` or a port-forward.
  The constructor is the same either way. Clients are created against that one address. A
  list of `SESSION_HOST`, `MODEL_HOST`, and `DATA_HOST` is the host sprawl this object
  replaces. `GenAIPlatform()` reads `GATEWAY_URL` from the environment.
  `GenAIPlatform(gateway_url="...")` is the explicit form for tests and dev. No sockets
  open until first use. Constructor failures are config failures. A Session Service that is
  still booting stays out of the constructor, or local startup becomes miserable.
* **Lazy `SessionClient`**: Hand-written HTTP means every team invents another Session
  client, and the fourth one is how the contract splits. Creating every client at import
  time means importing the SDK requires the whole cluster. The first access to
  `platform.sessions` builds a `SessionClient`: a channel to the gateway, metadata
  `x-target-service: sessions`, and a proto stub. Methods convert Python objects to
  protobuf, call the stub, and convert the result back to Python. Sam sees wire types only
  when he asks. `handle` calls `get_or_create`; the first call constructs
  `SessionClient(channel, metadata)`; `stub.GetOrCreateSession(request)` runs; the gateway
  routes the binary to the Session Service. Model, Data, and Tools use the same skeleton.
  Learn it once. A later `ModelClient` is this paragraph with a different stub. Deadlines
  and retries in this layer cover transport. A product retry, such as trying Claude when
  OpenAI returns 503, belongs in the Model Service, inside the service that chooses
  vendors, and stays out of every workflow's `try/except`.
* **Deploy struct**: The ship tool cannot guess `api_path`, replica policy, or sync versus
  stream from a bare `def handle`. A comment in Slack is not a schema. `@workflow(...)`
  attaches `name`, `api_path`, `min_replicas`, `max_replicas`, `target_cpu_percent`, and
  `response_mode`. The deploy CLI imports the module, reads the struct, builds the image,
  and registers with the Workflow Service. The happy path needs no second YAML. An overlay
  remains available for prod-only limits. If the function yields and `response_mode` says
  sync, fail at deploy, before Maria's first token.
* **One pattern, new methods**: Later services hang on one gateway connection story, one
  client pattern (metadata, stub, and typed methods), and one decorator that is the deploy
  API. When Model adds `chat_stream`, it is a new method on `ModelClient`, and host
  discovery stays the one you already have. When Session adds memories, they are new RPCs
  on `SessionClient`. That consistency is how the fifth team avoids a fifth wrapper.

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

## Check yourself

1. A new hire needs a first reply, a follow-up about "the laptop," this week's return
   policy, a ban on legal advice, a calendar check, a scored prompt, and one path that does
   all of it. For each of the seven rows, name the service that owns it and one smell that
   Sam is writing infrastructure again.
2. Why is "we wrapped OpenAI in a Python class" short of an SDK in this chapter's sense?
   What third job, besides calling a model, is missing?
3. Shared process versus one workflow per service: name one security failure and one
   scaling failure of the shared process. What do you pay for the split?
4. For processes, virtual machines, and containers, name the startup time and the isolation
   boundary. What risk remains with containers, and why should you refuse to tell security
   that a container is a VM?
5. Three workflows publish `support.internal/api/v1/ask` (API key),
   `copy.marketing.io/generate/v2` (OAuth2), and `ops-docs.company.net/analyze` (homemade
   JWT). What does the frontend team maintain? Which `@workflow` fields remove that sprawl
   and still leave path design with the team?
6. Assign sync, SSE, or async to a classify-image call, a 40-second chat, and a 20-minute
   research job. What breaks if the research job is synchronous behind a 60-second load
   balancer?
7. Why does the phone stay on HTTP while a session load uses gRPC? Name a client you should
   refuse to force onto protobuf, and a cost you should refuse to pay on that hot path.
8. Trace Maria's return-window question from HTTPS to a model token. Where is the trace id
   born, and why does `platform.sessions` still go through the gateway on the way to a pod?
9. Lazy clients versus connect-on-import: which local-dev failure does lazy construction
   avoid? When do you still want a readiness check before serving traffic?
10. Name the three jobs of `GenAIPlatform`. Which job remains if there is no Model Service,
    and which job disappears if the decorator disappears?
