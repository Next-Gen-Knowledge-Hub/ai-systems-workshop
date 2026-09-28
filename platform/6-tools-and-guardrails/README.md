# 6. Tools and guardrails

Companion notes for **Chapter 6** of *Designing AI Systems* (MEAP, Suhas
Suresha and Dewang Sultania; Manning). This MEAP's printed contents page is
**single-level** (nine chapter titles). The headings below follow the
chapter's actual topics.

This chapter is how the platform lets workflows **act** without turning
every team into a secrets-and-policy team. Sessions remember. Data
retrieves. Models generate. Tools change calendars, tickets, and ledgers.
Guardrails decide which of those changes are allowed. Skip this chapter
and "function calling" is a JSON blob in a prompt plus an API key in an
environment variable — until the sixth application copies both.

This folder stays on the **Tool Service**: registry, adapters,
credentials, MCP as an interoperability *bus*, execution limits, and
**policies the platform enforces**.

## The mental model

```
  workflow / SDK
        |
        v
  +------------------ Tool Service ------------------+
  |  Register / Discover / Validate / Execute        |
  |                                                  |
  |  REGISTRY     names, versions, capabilities      |
  |  ADAPTERS     translate platform call -> vendor  |
  |  CREDENTIALS  refs only; values never in app     |
  |  MCP CLIENT   one connector per server (cost M)  |
  |  LIMITS       CPU, RAM, time, payload, retries   |
  |  BREAKERS     closed / open / half-open          |
  |  POLICIES     input | behavioral | output        |
  +------------------------+-------------------------+
                           |
           +---------------+----------------+
           v               v                v
      HTTP APIs      MCP servers      async jobs
      (Epic, …)      (GitHub, …)      (slow verifiers)
```

The one sentence to remember: **a tool is a governed capability**, not a
function schema. The schema is what the *model* sees. Identity, version,
owner, credentials, rate limits, side effects, and audit are what the
*organization* sees. Guardrails are execution policy at every hop where
the system might do something irreversible. A swear filter on the reply
is only one slice of that policy surface.

Chapter 1's "safe action" bullet is this service. Sprawl here looks like
four copies of `check_status`, four Slack tokens in four `.env` files,
and no one who can answer "which bots can refund."

## Tools as platform-managed capabilities

Most teams start the way they start sessions and RAG: inline. A Python
function, a decorator, a key in the environment, a prompt that mentions
the function name. That ships a demo. It does not ship a catalog.

The inversion: applications **declare which capabilities they need**.
The platform owns registration, discovery, credential injection,
execution, and policy. A scheduling tool exists whether or not today's
assistant is the one calling it. Another team can discover it, request
access, and leave FHIR to the adapter authors.

```
  BEFORE                         AFTER
  app A: book() + key            Tool Service: healthcare.scheduling.book
  app B: book() + key     -->    apps A, B, C: discover + execute
  app C: book() + other key      credential store: one secret, rotation
```

Sharing is the point, but so is **governance**. A capability with a
name, a version, an owner, and an access policy can be reviewed. A
closure in a notebook cannot.

### The tool service contract

Same pattern as Model, Session, and Data: a small gRPC (or equivalent)
surface, with a shared set of verbs rather than twelve client libraries.

Four operations cover the lifecycle:

- **Register** — put a capability in the inventory: schema, docs,
  endpoint or adapter, metadata, credential *reference*, limits.
  Register once. Many apps consume.
- **Discover** — find tools by namespace, capability tags, read-only
  flag, version constraint. Return definitions the model can use as
  function calling JSON **and** the operational fields the platform
  needs. Callers do not pass "app id" to sneak around ACL; identity
  comes from auth context.
- **Execute** — run the tool with arguments. The service injects
  credentials, applies limits, records the span, returns a structured
  result or a job id.
- **Validate** — check arguments *before* side effects: schema, ranges,
  referential checks ("this patient id is the authenticated patient").
  Validate is how you fail cheap.

If your "tool service" is only Execute, you have a proxy. The other
three verbs are how you stop every workflow from hard-coding the Slack
client.

### Definitions beyond function schemas

Vendor function-calling docs care about **name, description, JSON
Schema**. Production cares about three layers that serve three
consumers:

```
  +------------------------------------------------------+
  | IDENTITY     name, version, owner, namespace         |
  |              (registry, ACL, billing)                |
  +------------------------------------------------------+
  | SCHEMA       description, parameters, returns        |
  |              (what the model is allowed to see)      |
  +------------------------------------------------------+
  | OPERATIONAL  read-only? idempotent? typical latency  |
  |              rate limits, cost, side-effect notes    |
  |              credential_ref, execution limits        |
  |              (guardrails, breakers, finance)         |
  +------------------------------------------------------+
```

The model should see a description that prevents `book_appointment`
from being used as `cancel_appointment`. Keep the credential name and
the internal endpoint out of that schema layer. Operational metadata is
how policy decides "this is not idempotent, do not retry blindly"
without parsing English in the docstring.

Side-effect documentation is not comments for humans only. Behavioral
guardrails later need to know whether a call mutates, notifies, or
charges. If that fact lives only in a wiki, the policy engine cannot
read it.

### The SDK interface

From an application author's chair the rest is supposed to vanish.
Register with a dotted name, a schema, an endpoint or adapter id, a
behavior block (`is_read_only`, `is_idempotent`), rate limits, a
credential ref. Discover by namespace or capability. Pass the returned
schemas into the Model Service's generate call. When the model emits a
tool call, `platform.tools.execute(...)` — with credentials injected
by the platform rather than loaded from disk in the workflow.

If developers still copy endpoint URLs into workflow code, the SDK has
failed even if the registry is full. The test: a new hire can find
`healthcare.scheduling.*`, pin a version, and book a slot without
opening the EHR vendor's OAuth guide.

## Registry: namespacing, discovery, versions

The registry is the catalog. Authoritative. Searchable. A searchable
inventory rather than a folder of Python files that import each other.

The org-scale question: global tools vs per-app tools? Usually **both**,
with namespacing as the structure. Global: "send transactional email"
with a brutal ACL. Per-app: experimental scrapers that should never be
discoverable by the billing bot.

### Namespacing and discovery

Uncoordinated teams all name a tool `check_status`. One means
appointment, one payment, one prescription. A model asked to
`check_status` will pick whichever schema was dumped into the prompt
last. Hierarchical names make the collision embarrassing *at register
time*:

```
  healthcare.scheduling.check_availability
  healthcare.scheduling.book_appointment
  healthcare.billing.verify_insurance
  healthcare.billing.check_payment_status
  healthcare.clinical.lookup_prescription
```

Discovery by prefix (`healthcare.scheduling.*`) is how a workflow loads
a *coherent* kit. Dumping the union of 400 tools into one prompt is an
agent-quality problem; the platform should offer scoped discovery so
callers can avoid that overload.

ACL on namespaces is a security control. Scheduling tools are not a
participation trophy for every application id.

### Capability-based discovery

Sometimes the caller knows the *job*, not the dotted path. "I need
something that verifies insurance." Capability tags on registration
(`insurance_verification`, `scheduling`, `phi_read`) let Discover search
across namespaces.

Return relevance scores if you must rank; still **filter by
permission** first. A high score on a tool you cannot invoke is a
prompt-injection gift: the model sees a name it cannot call and
improvises.

Tags are curated, not free prose. If every tool is tagged `important`,
you have built a folksonomy, not a catalog. Review tags the way you
review names.

Read-only filters belong here. A research workflow should be able to
discover only `is_read_only=true` tools even if the same namespace
contains a cancel API.

### Version control

Tools evolve. Parameters appear. Return shapes change. Deprecations
happen. If Register overwrites in place, every pinned prompt and every
cached schema is a production incident.

Keep **semver** on the definition. `2.0.0` that adds required
`appointment_type` can coexist with `1.4.2`. Applications discover with
a constraint (`^1.4`, `>=2.0.0 <3`) and migrate on purpose.

Major bumps are for breaking schema. Behavior changes that alter side
effects ("now this emails the patient") are breaking even if JSON
Schema is unchanged — treat them as majors or you will skip a
confirmation policy.

Do not leave infinite versions forever. Deprecate, alert consumers,
delete when the breaker is quiet. The registry is a living catalog with
a retirement path; treat it that way rather than as an endless history.

## Execution: adapters, credentials, sync vs async

Knowing a tool exists is inventory. Calling it is I/O: auth, failure,
latency that might be 200ms or four hours. The platform wraps that so
workflows speak **arguments in, result out**.

### Adapters

External systems are rude. An EHR might speak FHIR, OAuth with odd
scopes, nested JSON, and per-endpoint quotas. Exposing that to every
AI app means every developer becomes a vendor specialist. That is
sprawl with extra XML.

An **adapter** sits between Execute and the vendor:

```
  Execute(book_appointment, {patient_id, slot_id})
        |
        v
  SchedulingAdapter
        |-- fetch token from credential store
        |-- map args -> FHIR Appointment
        |-- call vendor, honor their rate limits
        |-- map response -> platform result / errors
        v
  ExecuteToolResponse
```

The Tool Service contract stays stable when the vendor adds a field.
Adapters are where retries that are *vendor-safe* live (idempotency
keys, with a clear story when a blind POST would double-book). They are
also where you strip secrets from logs: log the vendor request id, and
keep the bearer token out of those lines.

If you cannot name the adapter for a tool, you probably inlined HTTP
in the workflow. That will grow a second auth story.

### Credential isolation

The most dangerous line in a naive tool is the key in source. Isolation
is **indirection**: at Register you attach `credential_ref="scheduling-
api-prod"`, never the value. Values live in a store with encryption,
ACL, rotation. Retrieve happens at Execute, inside the platform
boundary, injected into the outbound call.

Consequences you should be able to swear to:

- Application code, SDK traces, and model prompts never contain the
  secret.
- Error messages never echo `Authorization` headers.
- Rotation does not require redeploying workflows — update the store,
  maybe bounce long-lived adapter connections.
- Separation of duties: the person who registers the tool shape is not
  required to see the production key. Provisioning is out of band
  (vault pipeline, admin CLI, sealed IaC).

If a "platform" still documents `export EPIC_TOKEN=...` for app
developers, you have a brochure.

### Credential store

Most orgs already have Vault or a cloud secrets manager. The platform
interface should **wrap**, leaving the existing store as the source of
truth:

- **Store** — name, type, value, rotation policy, **allowed tool
  names**.
- **Retrieve** — by name *and* requesting tool; deny if the tool is
  not on the allow list (confused deputy: the booking tool should not
  pull the warehouse admin token).
- **Rotate** — new value, old value invalidated; in-flight calls fail
  or retry with backoff, they do not log the old secret.

Retrieve is a privileged operation. Audit it. Observability will want
those records next to the tool span. Even here: a retrieve without a
corresponding Execute is a smell.

### Synchronous vs asynchronous execution

Backends do not share a latency budget. Slot lookup: hundreds of
milliseconds. Insurance verification: tens of seconds. Prior
authorization: human in the loop, hours.

```
  SYNC                              ASYNC
  app -> Execute ---------------->  app -> Execute
           | wait                          |
           v                               v
        adapter                         job id (queued)
           |                               |
           v                               v
        result                          worker + vendor
                                           |
                                           v
                                        completed | failed
                                        (poll / callback)
```

Sync is for bounded, predictable work. The Tool Service still applies
timeouts; "sync" still means "wait within a wall," with a hard kill.

Async is for work that would stall the conversation. Return a task id
immediately. The workflow continues (tell the user you are checking
coverage) and resumes on completion. The product lesson matches async
ingest elsewhere on the platform: **do not block the user on someone
else's SLA**.

Idempotency matters more on async retries. If `is_idempotent` is false,
duplicate Execute after a timeout might double-book. The adapter should
use idempotency keys the vendor understands, or Validate should reject
a second book for the same slot.

## MCP interoperability

The inventory you built still sits inside one organization. Outside:
every SaaS has its own auth, schema, errors. Five apps times ten
providers is **fifty** glue layers. The sixth app pays ten again.

MCP (Model Context Protocol, open standard, 2024) attacks the glue, not
your org chart. Providers wrap once. Applications speak one protocol.
Any host that implements the client side can call any server that
implements the server side.

Stay here for the **platform arithmetic** and the gaps you must still
fill. Protocol transports, Inspector, and writing a server are a
different teaching track.

### From N×M to N+M to M

Without a standard: **N applications × M providers**. Custom connectors
everywhere.

With MCP, each provider builds one server, each application (or host)
builds one client: **N + M**.

With a platform Tool Service as the **single MCP client** for the
company, application teams do not implement MCP at all. The service
connects to each server once. Every workflow discovers those tools like
native ones. Integration cost inside the org collapses toward **M** —
one connection per server — plus the platform work MCP refuses to do
(next subsections).

```
  N apps x M APIs     -->  spaghetti

  N MCP clients + M servers  -->  protocol win

  1 Tool Service client + M servers
       ^
       |  N apps use Register/Discover/Execute
       +  credentials, policy, audit still here
```

MCP is why you should adopt a shared wire format instead of inventing a
private plugin format. The protocol standardizes the *wire*. The
platform still standardizes *who paid, who was allowed, which version,
which secret*. Keep the Tool Service; MCP does not replace it.

### Hosts, clients, and servers

MCP vocabulary is easy to scramble with HTTP:

- **Server** — wraps a system (repo, DB, filesystem) and *serves
  capabilities*. May run as a local subprocess or as a remote service.
  "Server" is a role, not a rack.
- **Client** — the protocol peer that sits inside a host and talks to
  one server.
- **Host** — the application that owns clients: Claude Desktop, an IDE,
  **or your Tool Service**.

You do not build most community servers. You **verify** they do what
they claim (a "calendar" server that also dumps `.env` is a supply-chain
incident). On a platform, application authors should not even choose a
transport. The Tool Service does.

### Three primitives

Servers expose:

- **Tools** — invokable actions (`tools/list`, `tools/call`). This maps
  onto the Tool Service Execute path. JSON Schema for arguments.
- **Resources** — read-only context (files, records). Pull, with
  mutation left to tools. A codebase server might expose files as
  resources.
- **Prompts** — reusable templates (review this PR with this rubric).

The platform's first integration point is **tools**. Resources can feed
document indexes or session context if you have a story; keep
`resources/read` on its own path rather than folding it into Execute.
Prompts collide with Model Service system-prompt management — decide an
owner, and keep one store.

### What MCP does not cover

Say this louder than the announcement blog:

- **Credential management.** The spec does not tell you how the server
  authenticates to GitHub. Surveys of public servers have found heavy
  reliance on static tokens in environment variables. Fine on a laptop.
  Not a rotation, scope, or audit story.
- **Policy.** No first-class "needs human confirmation," "prod only,"
  "max three calls per session." No namespace governance, no org rate
  limits, no cost attribution.
- **Security as a complete model.** Prompt injection via tool results,
  confused deputies, over-broad servers — still *your* threat model.
  Sandbox and egress hardening are real work; assume MCP moves bytes in
  a standard shape, and keep the threat model yours.

MCP moves bytes from A to B in a standard shape. Before and after the
call remain platform work. That is the whole justification for
Register-plus-policy on top of `tools/call`.

### Integrating MCP with the platform

The Tool Service **is** an MCP client. It connects to approved servers,
lists tools, maps them into the registry under a **platform namespace**
(`devtools.github.create_issue`), and routes Execute through the MCP
transport.

Layer what the protocol omitted:

- Credentials from the store, with values kept out of the server's env
  on a laptop.
- Policy overrides at registration (`requires_confirmation`,
  `max_calls_per_minute`).
- Every invocation traced and billed like a native adapter.

SDK shape: `register_mcp_server(url, namespace, credential_ref,
policy_overrides)` then Discover as usual. Workflows should not know
they are speaking MCP. If they do, you leaked a transport into product
code.

Pin server versions the same way you pin tool semver. Community servers
move. A surprise schema change is a production break whether the wire
is HTTP or MCP.

## Execution safeguards

Execute is you acting on the world's APIs **on behalf of a model**. A
runaway adapter loop, a multi-gigabyte payload, or a vendor outage
should not take down the Tool Service or adjacent sessions.

Isolation here is the same lesson as process isolation for workflows,
applied per invocation.

### Resource limits

Per-call bounds, declared at Register, enforced at Execute:

- **Timeout** — hard kill. "Typical latency" is a hint; timeout is a
  wall.
- **Memory / CPU** — especially if adapters run user-shaped code or
  parse hostile JSON.
- **Max response size** — truncate or reject; do not ingest a dump into
  the model's context.
- **Retry budget** — retries only if `is_idempotent` (or if the adapter
  has a real idempotency key). Otherwise you double-charge.

Defaults come from the tool owner. The platform may cap them (no tool
gets a 30-minute sync timeout just because someone typed it). Session-
level budgets (max mutating calls per conversation) are **behavioral
policy**, but they share the same enforcement point: Execute.

Limits that exist only in a wiki are decorative.

### Circuit breakers

When a vendor is already failing, retrying from every session is a
self-inflicted load test. A breaker per tool (or per vendor) tracks
recent failures:

```
  CLOSED  --(failures >= threshold)-->  OPEN
    ^                                    |
    |         recovery timeout           v
    +------------ HALF-OPEN <------------+
         (few probes; success -> CLOSED
          fail -> OPEN)
```

- **Closed** — traffic flows; failures counted.
- **Open** — fail fast, no outbound call. The model gets a structured
  error it can explain, with a hang replaced by an immediate response.
- **Half-open** — limited probes. Success closes. Failure re-opens.

Thresholds and recovery timeouts are ops knobs. They belong in
observability (error rate, state changes), with state changes visible
before an engineer reconstructs them after an incident.

Breakers are a companion to Validate. A tool that is "healthy" and
booking the wrong patient is a policy miss; the circuit only sees
vendor health.

## Guardrails as execution policies

"Guardrails" in casual speech means profanity filters and vendor safety
classifiers. Those are real. They are a **subset**.

A patient intake assistant that never swears can still book a specialist
without a referral, or schedule someone who has not finished intake
forms. Those are **policy** failures. No generic moderation API knows
your clinic's rules.

That distinction moves the control plane:

- Content filters are often vendor-generic, prompt-adjacent, the same
  for every app.
- Execution policies are **yours**: domain, workflow, environment.
  They encode rules you would fire a human for skipping.

If policy lives only as a paragraph in the system prompt, a user who
asks the right way will walk past it. The Tool Service (with a
guardrail evaluator on the path) is where rules become **deny/allow/
confirm/transform** with an audit row.

Treat inspection as a chain, not a single gate. A useful map of *when*
policy can fire:

```
  (1) user input
        |
        v
  (2) proposed tool choice          behavioral
        |
        v
  (3) tool arguments                Validate + constraints
        |
        v
      Execute (limits, breaker, credentials)
        |
        v
  (4) tool result                   (injection via retrieved text)
        |
        v
  (5) model output to user          output policy
```

Prompt-only safety occupies none of these boxes reliably. Extra critic
loops around (5) are optional. Argument checks and execution policy
(steps 2 and 3) still have to exist even when you never train a judge.

## In practice: input, output, behavioral

Three categories, three failure classes.

### Input policies

Operate at (1), and often again on tool arguments at (3).

**Defend the system:** prompt injection and jailbreaks ("ignore previous
instructions," roleplay smuggling, encoded payloads). Pattern matching
catches sloppy attempts. Classifiers catch some of the rest. Neither is
complete — hence defense in depth with output policy and least-privilege
tools.

**Scope the product:** topic classifiers so a scheduling assistant does
not become a diagnosis bot. Out-of-scope is a product boundary. Action
might be *redirect* ("I can help with intake; for clinical questions, a
clinician") rather than a dead end.

**PII on the way in:** warn or block SSNs and card numbers if the
workflow is not supposed to collect them in chat.

Declarative checks belong in config: type, action (`block`, `redirect`,
`warn`), user-visible message, thresholds. Engineers should not ship a
new container to add a jailbreak pattern.

Tool-argument validation is input policy for the vendor: types, required
fields, "date is not in the past," "patient_id matches the authenticated
user." This is Validate with teeth. A past-dated booking should never
reach the adapter.

### Output policies

Operate at (5): last look before a human sees tokens (and before you
persist them as gospel in Session).

- **PII leakage** — redact or review. Clinic phone numbers vs patient
  SSNs are different; blind redact can be as wrong as blind send.
- **Grounding** — claims about coverage or policy cross-checked against
  retrieved hits or structured sources. Unverifiable claims: hedge,
  flag, or block. This is a **platform check** you can attach to many
  workflows.
- **Brand / regulatory tone** — material in health and finance, with
  real compliance weight.

Severity tiers keep you honest. After you know which failure classes
matter, this table is a starting map:

| Severity | Example | Action |
|---|---|---|
| Low | slightly informal | log |
| Medium | unverifiable claim | transform (hedge) |
| High | exposed PAN, invented legal right | block + fallback |

Blocking without a fallback is how you generate support tickets that
say "the bot just stopped." Always have a safe sentence.

### Behavioral policies

Operate at (2) and (3): **which tools, in which order, with which
authority.** This is the layer that makes guardrails *execution*
policies.

Examples that content filters will never catch:

- Book specialist without referral on file.
- Cancel and rebook the same slot in one turn (confused model) —
  intervene, ask, and leave both calls unexecuted until confirmed.
- Update address then notify the old address — reorder or gate on
  completion.
- Failed insurance verify then book anyway — deny with an explanation.
- Session budget: three booking attempts, then stop.

Human confirmation is a behavioral action, with a real UI step behind
it. Irreversible or expensive tools (`is_read_only=false`, high cost
metadata) default to confirm in production until a risk committee says
otherwise.

Cross-tool rules need a **session-shaped view** of proposed actions,
with the full proposed set visible rather than one schema at a time.
That is why this lives next to Execute, with room for the prompt to
stay descriptive rather than authoritative.

Least privilege: Discover should have already hidden the refund tool
from the marketing bot. Behavioral policy is defense in depth when the
model still asks.

## Declarative policy and observability

Application authors write flows. Compliance writes what must never
happen. Security writes injection rules. Ops writes rate limits. If all
of that is Python in the assistant repo, you have made Sarah (or Sam)
the bottleneck for every rule change, and you have made audit a git
blame exercise.

### Policy as configuration

Policies specify **what** to check, **when**, and **what to do**. YAML
(or equivalent) that a non-engineer can read is the point:

- application id
- version of the *policy document*
- input checks (topic, injection, PII)
- output handlers by severity
- behavioral rules (tool allow lists, confirmation, cross-tool)
- actions: block, redirect, warn, transform, confirm, log_only

Store in version control. Review like code, with **different
reviewers** (compliance, security). On merge, the platform hot-loads.
No assistant redeploy to add a blocked topic.

Policy version appears on every audit row so you can answer "what rule
was live on Tuesday."

Application code should leave topic checks in the policy document. An
`if topic == diagnosis` branch in the assistant is a second, drifting
copy of the rule.

### Making safety visible

A silent block is how you get "the assistant refused and hung up" with
no way to tell legitimate enforcement from a false positive from a
buggy threshold.

Three audiences:

- **Developers** — why this request died (inspection point, policy
  name, action).
- **Operations** — rates: block rate, confirm rate, breaker opens,
  p95 Execute, cost per tool.
- **Compliance** — what was proposed, what ran, what was denied, who
  confirmed. Retention that matches your regulation, sized for the
  retention policy rather than the log disk alone.

Every evaluation produces a record, including **allows**. Allows are
how you measure false-negative risk. For classifier policies, store
scores so you can retune thresholds without folklore.

Hash content when you cannot retain raw PII in the same lake as traces.
Join hashes to Session and observability ids. This chapter's job is to
**emit** the safety span; the store-and-experiment layer lives with
observability.

If you cannot graph "booking denials because missing referral" you
cannot tell a product bug from a successful control.

## What this service is not

It is not the agent loop. ReAct, which tool to pick, whether to try
again: that decision layer calls this service. This service is the
**safe socket** that loop uses.

It is not Data. Reading a policy PDF is retrieval. Filing a ticket is a
tool. Mixing them ("the search tool that also emails Legal") is how
permissions rot.

It is not Observability, but it is a noisy citizen of it. Execute,
breaker state, policy decisions, credential retrieves belong on the
same trace as the model generation.

## Check yourself

1. A team says "we have tools; they're Python functions in the
   workflow." Map Register, Discover, Execute, and Validate to what
   they are missing, and name one incident each verb would have
   shortened.
2. Draw the three layers of a tool definition. Which layer is safe to
   send to the model, and which layer should never appear in a prompt?
3. Two squads register `check_status`. How does namespacing prevent the
   collision, and how does capability discovery still find "payment
   status" without guessing the dotted name?
4. Why do major versions coexist instead of overwrite? Give a schema
   break and a *behavior* break that JSON Schema would not catch.
5. Credential_ref vs the secret in `.env`: list three places the secret
   must never appear, and which CredentialStore retrieve check stops a
   confused-deputy tool from loading the wrong token.
6. MCP: explain N×M → N+M → M for *your* org's app count and SaaS
   count. What work remains at M because the protocol does not specify
   it?
7. A community MCP server needs `GITHUB_TOKEN` in the environment.
   How does `register_mcp_server` on the platform change that story
   without forking the server?
8. Circuit breaker vs timeout vs retry budget: which one you turn when
   the vendor is 100% failing, which when a single call hangs, and
   which you *disable* when `is_idempotent` is false?
9. Content filter vs execution policy: give one clinic (or your domain)
   failure that a toxicity classifier will not catch. At which
   inspection point (1–5) would you enforce it?
10. Compliance wants to add a blocked topic tomorrow without a
    deploy. What artifact do they change, who reviews it, and what
    must appear in the audit row so Sarah can debug a false positive?
