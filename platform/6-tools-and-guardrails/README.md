# 6. Tools and guardrails

Companion notes for **Chapter 6** of *Designing AI Systems* (Suhas Suresha and Dewang
Sultania; Manning MEAP).

This chapter establishes the Tool Service. A tool is a governed capability: a registry, an
adapter, a credential reference, and policies on the path that actually runs. Afterward you
can name the four verbs, the three layers of a definition, what MCP still leaves to the
platform, and which guardrail fires on the user's text, on the proposed action, and on the
reply. The schema is what the model sees. Identity, version, owner, credentials, limits,
side effects, and audit are what the organization sees. Without the service, function
calling is a JSON blob plus an API key in an environment variable, until the sixth app
copies both. Sprawl looks like four `check_status` functions, four Slack tokens, and nobody
who can say
which bots can refund.

## 1. A governed capability

Teams start the way they start a demo: a Python function, a decorator, a key in the
environment, a prompt that mentions the name. Applications instead declare which
capabilities they need. The platform registers, discovers, injects credentials, executes,
and applies policy. `healthcare.scheduling.book` exists whether or not today's assistant is
the caller. The next team
discovers it and leaves the vendor protocol to the adapter.

```
  BEFORE                          AFTER
  app A: book() + key             Tool Service: healthcare.scheduling.book
  app B: book() + key             apps A, B, C: discover + execute
  app C: book() + another key     one secret in the store, rotated once
```

* **Register**: Put the capability in the inventory once. Schema, docs, endpoint or adapter,
  metadata, a credential *reference*, limits. Many apps consume that one row.
* **Discover**: Find tools by namespace, capability tag, read-only flag, or version
  constraint. Return the function-calling JSON the model may see, and the operational fields
  the platform needs. Identity comes from the auth context. An app id in the request is not
  a way around the access check.
* **Validate**: Check arguments before any side effect. Schema, ranges, and referential
  checks, such as "this patient id is the authenticated patient." Fail cheap.
* **Execute**: Inject credentials, apply limits, record the span, return a structured result
  or a job id.

A service that only executes is a proxy. The other three verbs are how workflows stop
hard-coding the Slack client.

Three layers serve three consumers. Ship all three or the catalog is a docstring.

1. **Identity**: Name, version, owner, namespace. Registry, access control, and billing read
   this layer. It does not belong in the prompt.
2. **Schema**: Description, parameters, returns. This is what the model is allowed to see.
   The description has to keep `book_appointment` from being used as `cancel_appointment`.
   The credential name and the internal endpoint stay out.
3. **Operational**: Read-only or not, idempotent or not, typical latency, rate limits, cost,
   side-effect notes, `credential_ref`, execution limits. Guardrails, circuit breakers, and
   finance read this layer. "This call emails the patient" has to be a field. A wiki page is
   invisible to the policy engine.

* **The SDK test**: Register a dotted name, a schema, an adapter id, `is_read_only`,
  `is_idempotent`, rate limits, and a credential ref. Discover, pass the schemas into the
  model call, and on a tool call run `platform.tools.execute(...)`. The platform injects the
  secret. A new hire can find `healthcare.scheduling.*`, pin a version, and book a slot
  without the EHR vendor's OAuth guide. Endpoint URLs copied into workflow code mean the SDK
  has failed.

## 2. The registry

The registry is the catalog: authoritative and searchable. Global tools and per-app tools
both exist. "Send transactional email" can be global, with a brutal access list. An
experimental scraper stays per-app so the billing bot never discovers it.

* **Hierarchical names**: Uncoordinated teams all register `check_status`. One means the
  appointment, one the payment, one the prescription. The model calls whichever schema
  landed in the prompt last. Dotted names make the collision fail at register time:
  `healthcare.scheduling.check_availability`, `healthcare.scheduling.book_appointment`,
  `healthcare.billing.verify_insurance`, `healthcare.billing.check_payment_status`,
  `healthcare.clinical.lookup_prescription`.
* **Prefix discovery**: `healthcare.scheduling.*` loads one coherent kit. Four hundred tools
  in one prompt is an overload problem. Scoped discovery is how callers avoid it. A
  namespace access list is a security control. Scheduling is not granted to every
  application id.
* **Capability tags**: Sometimes the caller knows the job, not the path. "Verify insurance"
  matches a curated tag such as `insurance_verification`, `scheduling`, or `phi_read`,
  including across namespaces. Filter by permission first, then rank. A high score on a tool
  the caller cannot invoke is a gift to prompt injection: the model sees a name it cannot
  call and improvises. If every tool is tagged `important`, the tags are not a catalog. A
  research workflow can ask for `is_read_only=true` even when that namespace also contains
  cancel.
* **Semver that coexists**: An in-place overwrite turns every pinned prompt and every cached
  schema into an incident. `2.0.0` adds a required `appointment_type` and lives beside
  `1.4.2`. Discovery takes a constraint (`^1.4`, or `>=2.0.0 <3`) and the app migrates on
  purpose. A breaking JSON Schema is a major. A behavior change with the same schema is a
  major too: "now this emails the patient" would otherwise skip the confirmation policy.
  Deprecate, alert consumers, and delete when the breaker is quiet. The catalog has a
  retirement path.

## 3. Adapters and credentials

Inventory says the tool exists. Execute is I/O: auth, failure, and latency from hundreds of
milliseconds to hours. Workflows still speak arguments in, result out.

```
  Execute(book_appointment, {patient_id, slot_id})
        |
        v
  SchedulingAdapter
        |-- token from the credential store
        |-- args mapped to a FHIR Appointment
        |-- vendor call, under their rate limits
        |-- response mapped to the platform result
```

* **Adapter**: The EHR speaks FHIR, odd OAuth scopes, nested JSON, and per-endpoint quotas.
  Putting that in every app makes every developer a vendor specialist. The adapter is where
  vendor-safe retries live, including an idempotency key when a blind POST would
  double-book. Log the vendor request id. Keep the bearer token out of the log line. If you
  cannot name the adapter, HTTP is inlined in the workflow, and a second auth story will
  grow beside it. The platform result stays stable when the vendor adds a field.
* **credential_ref**: Register stores `credential_ref="scheduling-api-prod"`, never the
  value. The value sits in a store with encryption, an access list, and rotation. Retrieve
  happens at Execute, inside the platform, and is injected into the outbound call.
  Application code, SDK traces, and model prompts do not contain the secret. Errors do not
  echo `Authorization`. Rotation updates the store and maybe bounces a long-lived adapter
  connection. Workflows do not redeploy. The person who registers the shape does not have to
  see the production key. Provisioning stays out of band: a vault pipeline, an admin CLI, or
  sealed automation. Docs that still say `export EPIC_TOKEN=...` for app developers are a
  brochure.
* **The store wraps what you already run**: Vault or a cloud secrets manager remains the
  source of truth. **Store** keeps name, type, value, rotation policy, and the tool names
  allowed to use it. **Retrieve** takes the name and the requesting tool, and denies the
  call when that tool is absent from the allow list. The booking tool must not pull the
  warehouse admin token. That check is the confused-deputy control. **Rotate** installs the
  new value and invalidates the old one. In-flight calls fail or retry with backoff. They do
  not log the old secret. Retrieve is privileged and audited. A retrieve with no matching
  Execute is a smell.

### Watch for

* The secret in source, in a prompt, or in an error body. Any one of those is the isolation
  failure.
* A credential allow list of "any tool." Rotation then becomes a broader leak.

## 4. Sync, async, and execution limits

Backends do not share a latency budget. Slot lookup is hundreds of milliseconds. Insurance
verification can take tens of seconds. Prior authorization waits on a person and can take
hours.

* **Sync**: Bounded work. The caller waits inside a wall. "Sync" still has a hard kill.
  Typical latency on the definition is a hint. The timeout is the wall.
* **Async**: Work that would stall the conversation returns a job id immediately (`queued`,
  then a worker, then `completed` or `failed`, by poll or callback). The workflow can tell
  the user it is checking coverage and resume when the job finishes. Do not block the user
  on a vendor SLA. Idempotency matters more here. If `is_idempotent` is false, a second
  Execute after a timeout can double-book. The adapter sends an idempotency key the vendor
  understands, or Validate rejects a second book for the same slot.

Per-call bounds are declared at Register and enforced at Execute. A limit that exists only
in a wiki is decoration. The platform may cap an owner's numbers. Nobody gets a 30-minute
sync timeout because they typed it.

* **Timeout**: Hard kill of one call.
* **Memory and CPU**: Required when an adapter runs user-shaped code or parses hostile JSON.
* **Max response size**: Truncate or reject. A dump does not enter the model's context.
* **Retry budget**: Retries only when `is_idempotent` is true, or when the adapter holds a
  real idempotency key. Otherwise you double-charge.

A session cap on mutating calls, such as three booking attempts, is behavioral policy. The
enforcement point is still Execute.

When the vendor is already failing, retries from every session are a self-inflicted load
test. A breaker per tool, or per vendor, watches recent failures.

```
  CLOSED --(failures >= threshold)--> OPEN
     ^                                 |
     |        recovery timeout         |
     +----------- HALF-OPEN <----------+
```

1. **Closed**: Traffic flows and failures are counted. This is the steady state. Opening on
   a single blip fails healthy calls.
2. **Open**: Fail fast, with no outbound call, when the failure count crosses the threshold.
   The model receives a structured error it can explain. Use this when the vendor is down
   across sessions. A single hung call is the timeout's job, not the breaker's.
3. **Half-open**: After the recovery timeout, a few probes go out. Success returns to
   closed. Failure returns to open. Unlimited probes mean you are closed again.

Thresholds and recovery timeouts are operations knobs, visible as state changes before
someone reconstructs them after an incident. A healthy vendor booking the wrong patient is a
policy miss. The breaker only sees vendor health.

## 5. MCP as the organization's one client

Outside the org, every SaaS has its own auth, schema, and errors. **MCP** (Model Context
Protocol, an open standard from 2024) standardizes that wire. Providers wrap a system once.
A host that speaks the client side can call a server that speaks the server side. This
section is the arithmetic, and the gaps the protocol leaves you.

1. **N times M**: No shared protocol. N applications each integrate M providers. Five apps
   and ten providers are fifty glue layers, and the sixth app pays for ten again.
2. **N plus M**: Each provider builds one MCP server. Each application builds one client.
3. **M**: The Tool Service is the company's single MCP client. App teams do not implement
   MCP. The service connects to each server once, and workflows discover those tools like
   native ones. You still pay for credentials, policy, and audit. The protocol does not do
   that work.

```
  1 Tool Service client + M servers
       ^
       |  N apps use Register, Discover, Execute
       +  credentials, policy, and audit stay here
```

* **Wire versus organization**: MCP is the shared socket, so you do not invent a private
  plugin format per host. The platform still decides who paid, who was allowed, which
  version, and which secret. Keep the Tool Service. The protocol does not replace it.
* **Server, client, host**: A **server** wraps a system (a repo, a database, a filesystem)
  and serves capabilities. It may be a local subprocess or a remote process. The word is a
  role, not a rack. A **client** is the peer inside a host that talks to one server. A
  **host** owns those clients: Claude Desktop, an IDE, or this Tool Service. You verify
  community servers. A "calendar" server that also dumps `.env` is a supply-chain incident.
  Application authors do not pick a transport. The Tool Service does.
* **Three primitives**: **Tools** are invokable actions (`tools/list`, `tools/call`) with
  JSON Schema arguments. They map onto Execute. **Resources** are read-only context, such as
  files or records. Keep `resources/read` on its own path. Do not fold a file read into
  Execute. **Prompts** are reusable templates, such as "review this PR with this rubric."
  The first platform hook is tools. Prompts need a single owner so they do not fork the
  Model Service's system-prompt store.
* **Credentials stay yours**: The spec does not say how a server authenticates to GitHub.
  Public servers often expect a static token in the environment. That is fine on a laptop.
  It is not rotation, scope, or audit.
* **Policy stays yours**: There is no standard field for human confirmation,
  production-only, or three calls per session. No namespace governance, no org rate limit,
  no cost attribution.
* **The threat model stays yours**: Prompt injection through tool results, confused
  deputies, and over-broad servers are still yours. MCP moves bytes in a standard shape.
  Sandbox and egress limits are still platform work.
* **register_mcp_server**: Connect to an approved server, list its tools, and map them under
  a platform namespace such as `devtools.github.create_issue`. Execute rides the MCP
  transport. The SDK call is
  `register_mcp_server(url, namespace, credential_ref, policy_overrides)`, then ordinary
  Discover. Credentials come from the store, so a server that wanted `GITHUB_TOKEN` in its
  environment gets a reference instead, without a fork. Overrides such as
  `requires_confirmation` and `max_calls_per_minute` attach at registration. Each call is
  traced and billed like a native adapter. Workflows should not be able to tell they are
  speaking MCP. Pin server versions the way you pin semver. A surprise schema on an unpinned
  community server is a production break on either wire.

### Watch for

* MCP in name only. The registry lists the server, and the token still lives in a developer
  environment variable.
* A workflow that imports an MCP client itself. The transport has leaked into product code.

## 6. Where a guardrail fires

A profanity filter and a vendor safety classifier are real. They are one slice of the
surface. A patient-intake assistant that never swears can still book a specialist with no
referral, or schedule someone who has not finished the intake forms.

* **Content filters**: Often the same for every app, and often sitting beside the prompt. No
  generic moderation API knows the clinic's rules.
* **Execution policies**: Yours. Domain, workflow, and environment. They encode a rule you
  would fire a human for skipping. Each decision is deny, allow, confirm, or transform, and
  each one writes an audit row.
* **Prompt text**: A paragraph in the system prompt is a rule a careful user can walk past.
  The Tool Service is where the rule becomes one of those four outcomes.

```
  (1) user input
  (2) proposed tool choice       behavioral
  (3) tool arguments             Validate + constraints
      Execute                    limits, breaker, credentials
  (4) tool result                injection inside retrieved text
  (5) model output to the user   output policy
```

Prompt-only safety occupies none of these points reliably. A critic loop around (5) is
optional. Argument checks and the policy at (2) and (3) still have to exist when you never
train a judge.

## 7. Input, output, and behavioral policy

Three failure classes. A toxicity score catches one of them.

1. **Input policy**: Runs at (1), and again on tool arguments at (3). Use it when the harm
   is in what arrived. It fails when you only filter tone and the arguments still name the
   wrong patient. Injection and jailbreaks ("ignore previous instructions," roleplay
   smuggling, encoded payloads) are the first check. Patterns catch sloppy attempts.
   Classifiers catch some of the rest. Neither is complete, so output policy and
   least-privilege tools stay in place. A topic check keeps a scheduling assistant from
   becoming a diagnosis bot. Out of scope can redirect ("I can help with intake; a clinician
   handles clinical questions") instead of dead-ending. Warn or block a social security
   number or a card number when this chat is not supposed to collect them. The check itself
   is config: type, action (`block`, `redirect`, `warn`), the sentence the user sees,
   thresholds. A new jailbreak pattern does not require a new container. Argument checks are
   the same class aimed at the vendor: types, required fields, a date that is not in the
   past, a `patient_id` that matches the authenticated user. A past-dated booking never
   reaches the adapter.
2. **Output policy**: Runs at (5), the last look before a person sees tokens, and before
   those tokens are stored as fact in the session. Use it when the harm is what the user
   would read or what you would persist. It fails when the bad action already committed at
   Execute. PII leakage is redact or review. A clinic phone number and a patient social
   security number are different leaks. Blind redact can be as wrong as a blind send.
   Grounding cross-checks a claim about coverage or policy against retrieved hits or a
   structured source. An unverifiable claim is hedged, flagged, or blocked. Brand and
   regulatory tone carry real weight in health and finance.

| Severity | Example | Action |
|---|---|---|
| Low | Slightly informal tone | Log |
| Medium | An unverifiable claim | Transform (hedge) |
| High | An exposed card number (PAN), or an invented legal right | Block, plus a fallback sentence |

A block with no fallback is the ticket that says the bot stopped. Always have a safe
sentence.

3. **Behavioral policy**: Runs at (2) and (3). It chooses which tools may run, in which
   order, with which authority. Use it when the harm is an action. It fails when you only
   score the final paragraph. Content filters do not see these cases. A specialist booked
   with no referral on file. Cancel and rebook of the same slot in one turn: stop, ask, and
   leave both calls unexecuted until someone confirms. An address update, then a notice sent
   to the old address: reorder, or wait until the update finishes. Insurance verification
   failed, then a book anyway: deny, and say why. Three booking attempts in the session,
   then stop. Human confirmation is a real UI step. Irreversible or expensive tools
   (`is_read_only=false`, or high cost on the operational layer) default to confirm in
   production until a risk review says otherwise. A cross-tool rule needs the whole proposed
   set for the session, not one schema at a time, which is why the check sits next to
   Execute. The prompt stays descriptive. Discover should already have hidden refund from
   the marketing bot. This policy is the second line when the model asks anyway.

## 8. Policy as configuration

Application authors write flows. Compliance writes what must never happen. Security writes
injection rules. Operations writes rate limits. Python in the assistant repo makes Sam or
Sarah the bottleneck for every change, and makes audit a git blame.

* **The document**: Something a non-engineer can read. Application id, version of the policy
  document, input checks (topic, injection, PII), output handlers by severity, behavioral
  rules (allow lists, confirmation, cross-tool order), and actions: block, redirect, warn,
  transform, confirm, log_only. It lives in version control. Compliance and security review
  it, not only the assistant author. Merge hot-loads the platform. Adding a blocked topic
  does not redeploy the assistant. The policy version is on every audit row, so "what was
  live on Tuesday" has an answer. An `if topic == diagnosis` branch in the assistant is a
  second copy, and it will drift.
* **Three readers of the same record**: Developers need the inspection point, the policy
  name, and the action, or a refusal is a hang. Operations need block rate, confirm rate,
  breaker opens, p95 of Execute, and cost per tool. Compliance needs what was proposed, what
  ran, what was denied, and who confirmed, retained for the regulation rather than until the
  disk fills. Record allows too. Allows are how you see false negatives. Classifier policies
  store scores so a threshold change is a measurement. When raw PII cannot sit in the trace
  lake, store a hash and join it to the session id and the trace id. This service emits that
  safety span. A graph of "booking denials because the referral was missing" is how you tell
  a product bug from a control that worked.
* **The socket, not the loop**: Choosing the next tool, and whether to try again, is the
  caller's loop. This service is the socket that loop uses.
* **A tool changes the world**: Reading a policy PDF is retrieval from the document library.
  Filing a ticket is a tool. A search tool that also emails Legal mixes the two permissions
  until both rot.
* **Same trace as generation**: Execute, breaker state, policy decisions, and credential
  retrieves belong on the trace beside the model call. This service emits that span.

### Watch for

* A silent block. You cannot tell enforcement from a false positive from a bad threshold.
* A rule that ships only as prompt text. The config document never sees it, and Tuesday's
  audit row cannot name it.

## Check yourself

1. A team says their tools are Python functions in the workflow. What are they missing
   across Register, Discover, Execute, and Validate, and which incident does each verb
   shorten?
2. Draw the three layers of a tool definition. Which layer may go to the model, and which
   layer must stay out of the prompt?
3. Two squads both want `check_status`. How does a namespace stop the collision at register
   time, and how does a capability tag still find payment status without the dotted name?
4. Why do `1.4.2` and `2.0.0` coexist? Give one schema break and one behavior break that
   JSON Schema would not show.
5. Name three places a secret must never appear. Which credential-store check stops the
   booking tool from loading the warehouse admin token?
6. For an org with N apps and M providers, what do N×M, N+M, and M each buy? Which work is
   still yours at M because MCP does not specify it?
7. A community MCP server expects `GITHUB_TOKEN` in the environment. How does
   `register_mcp_server` change that without forking the server?
8. The vendor is failing every call, a single call hangs, and `is_idempotent` is false.
   Which of breaker, timeout, and retry budget do you use for each?
9. Name a clinic failure a toxicity classifier will not catch. At which inspection point,
   from (1) through (5), do you enforce it?
10. Compliance wants a new blocked topic tomorrow, without a deploy. What do they change,
    who reviews it, and what must the audit row contain so Sarah can debug a false positive?
