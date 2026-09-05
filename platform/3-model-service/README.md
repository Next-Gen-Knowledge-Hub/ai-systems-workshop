# 3. The Model Service

Companion notes for **Chapter 3** of *Designing AI Systems* (MEAP, Suhas
Suresha and Dewang Sultania; Manning). This MEAP's printed contents page is
**single-level** (nine chapter titles). The headings below follow the
chapter's actual topics.

Chapter 1 called the provider call ~2% of the system. This chapter is
how that 2% becomes a **service** the other 98% can share: one contract,
many vendors, streaming, retries, routing, cache, numbers you can join
to a trace. Skip it and every workflow imports a different SDK, keys
live in six `.env` files, and the invoice cannot name a feature.

The Agents track is a different book. Tokens, temperature, persona, *one
agent's* model call: [agents ch. 2](../../agents/2-llms-prompting-agents/).
Timeouts, fallbacks, and routing from a *deployed agent* runtime:
[agents ch. 8](../../agents/8-deploying-agents/). Mention those folders.
This folder stays on the **Model Service** — adapters, a stable message
shape, org-wide policy for cost and failure.

## The mental model

```
  workflow:  platform.models.chat(messages, model=?, stream=?)
                      |
                      v
                 ModelClient  (gRPC, x-target-service: models)
                      |
                      v
  +------------------- Model Service ----------------------+
  |  CONTRACT   Chat / ChatStream / ListModels / prompts / |
  |             RegisterModel                              |
  |  GATE       rate limit, response cache lookup          |
  |  ROUTE      cost | load | features | explicit name     |
  |  RESILIENCE retry -> fallback chain                    |
  |  ADAPTERS   OpenAI | Anthropic | Google | vLLM / ...   |
  |  EMIT       tokens, $ , latency, cache path, provider  |
  +------------------------+-------------------------------+
                           |
                           v
                    Observability Service
```

The one sentence to remember: **the workflow names a job; the service
names a provider.** Hardcoding `gpt-4o` in twelve handlers is how you
cannot fail over, cannot see spend, and cannot add a self-hosted
endpoint without a flag day.

Routing, cache, and fallbacks as one-screen rows:
[`TRADEOFFS.md`](../../TRADEOFFS.md). Use this folder for the *service*
that executes those rows.

## The model service contract

[Chapter 2](../2-sdk-and-api/) said internal calls are gRPC. The
**.proto is the promise.** SDK methods, gateway routing, Session (same
message shape), Data (embeddings later) all hang off it. Get the verbs
wrong and you will version the SDK twice a quarter.

### Generating responses

**Problem** — Every app reinvents "send some roles, get text and token
counts." One team forgets usage. Another cannot pass tools.

**Solution** — One **Chat** RPC: model (or empty, for routing),
messages, sampling/config, optional tool definitions, optional
response format (JSON schema, etc.). Response: text (or tool calls),
the model that *actually* ran, usage.

```
  ChatRequest                     ChatResponse
  -----------                     ------------
  model                           content | tool_calls
  messages[]  role + content      model     (after fallback, this
  config      temp, max_tokens              is not always what you
  tools[]     name, json schema             asked for)
  response_format                 usage     prompt / completion
```

Roles match the industry default: `system`, `user`, `assistant`,
`tool`. That is not fashion. It is how Session stores history and how
adapters translate. Sampling knobs as *agent craft* are Agents ch. 2.
Here they are fields on `ChatConfig` the service forwards.

### Discovering available models

**Problem** — `"gpt-4o"` in source goes stale. A 50-page PDF needs a
window the hardcoded name does not have. A vision hop hits a text-only
endpoint and fails in production.

**Solution** — **ListModels** / **GetCapabilities**: name, provider,
context window, vision, tools, JSON mode, maybe price class. Workflows
that care query at runtime. Workflows that do not still benefit because
**routing** (later) uses the same catalog.

Discovery is a platform catalog, not a scrape of marketing pages. If a
model is in the list, adapters and keys exist. If it is not, fail
fast — do not send a prayer to a default.

### Managing system prompts

**Problem** — The persona paragraph is copied into four workflows.
Legal edits one. Three drift. A prompt change requires a deploy of
unrelated code.

**Solution** — Prompts as **named, versioned documents** the Model
Service stores. Workflows pass `system_prompt_name` (and maybe a
version pin). Chat assembly prepends the current body. Eval in
[chapter 7](../7-observability/) can swap names without rewriting
handlers.

This is not "prompt engineering." Crafting the paragraph is Agents
ch. 2 / the product. **Hosting** the paragraph so it is not sprawl is
this service.

### Registering custom models

**Problem** — Fine-tunes and vLLM boxes do not appear in the vendor
list. Teams then call them with a one-off client, skipping cache,
quotas, and traces.

**Solution** — **RegisterModel**: name, base URL, protocol (often
OpenAI-compatible), auth ref, advertised capabilities. Same Chat RPC
after that. Privacy, unit cost, and domain vocab are why you self-host
([`TRADEOFFS.md`](../../TRADEOFFS.md) build-vs-buy is adjacent; ops
cost is real). The platform still wants one on-ramp.

If registration is a ticket to the platform team with a week SLA, you
will get shadow endpoints. Make it an RPC with ACL, not a wiki.

### The gRPC contract

Four groups, one service:

```
  service ModelService {
    rpc Chat(ChatRequest)       returns (ChatResponse);
    rpc ChatStream(ChatRequest) returns (stream ChatChunk);

    rpc ListModels(...)         returns (...);
    rpc GetModelCapabilities(...) returns (...);

    rpc RegisterPrompt / GetPrompt / ListPrompts ...
    rpc RegisterModel / ...
  }
```

**Chat** vs **ChatStream** share the request message. That is
deliberate: streaming is a delivery mode, not a different product.
Gateway maps stream to SSE for browsers
([chapter 2](../2-sdk-and-api/)).

### Request and response structures

Protobuf forces you to say what moves. Minimum honesty:

- **ChatMessage** — role, content, optional `tool_calls`, optional
  `tool_call_id` (result rows).
- **ChatConfig** — temperature, max tokens, top_p if you expose it,
  seed if a provider has it. Unknown knobs die in the adapter, not in
  the workflow.
- **ToolDefinition** — type `function`, name, description, parameters
  JSON. The Model Service does not *execute* tools
  ([chapter 6](../6-tools-and-guardrails/)); it only forwards schemas
  and returns calls.
- **TokenUsage** — prompt, completion, cache-read, cache-write if the
  vendor reports them.

Optional `model` empty means "you pick" under routing config. Optional
`response_format` is how structured output becomes a contract field
instead of a prompt footnote.

## Provider abstraction

Behind the contract sits the ugly fact: vendors did not agree.

### How providers differ

**Problem** — Same job, incompatible envelopes.

```
  aspect          OpenAI              Anthropic           Gemini
  ------          ------              ---------           ------
  system          role in messages    top-level system    parts / roles
  body            {role, content}     {role, content}     {role, parts}
  text path       choices[0].message  content[0].text     candidates[0]
  token names     prompt/completion   input/output        prompt/candidates
  stream          SSE delta.content   text_stream ctx     chunks / parts
```

These are not bugs. They are product histories. Your workflows cannot
absorb them. If they do, swapping a provider is a rewrite.

**Solution** — A **canonical platform message** plus one adapter class
per vendor (the next subsections). Product code never sees this table.

### The unified provider interface

**Problem** — `if provider == "anthropic"` in Sam's handler. Then
again in marketing's. Then a third time in a notebook.

**Solution** — **Adapter pattern.** Applications see `ModelProvider.chat`
/ `chat_stream`. Each vendor class translates. Adding Google is a new
class, not a new if-ladder in product code.

```
  ChatRequest (platform)
        |
        v
  router / fallback
        |
        +--> OpenAIAdapter -----> OpenAI HTTP
        +--> AnthropicAdapter --> Anthropic HTTP
        +--> VLLMAdapter -------> OpenAI-compatible POST
```

The service may retry and fail over. The adapter should stay **boring**:
translate, call, normalize, map errors. Policy (which chain, how many
retries) is configuration *on the request or app*, execution is
platform-side — Sam does not write the loop.

### OpenAI message format as platform standard

**Problem** — You need *one* in-memory shape. Inventing a fourth
"neutral" schema means two translations for every vendor, including
the one everyone already speaks.

**Solution** — Use the **de facto** array of `{role, content}` (plus
tool_calls) as the platform lingua franca. Not because one company is
owed worship. Because docs, OSS servers (vLLM), and Session storage
already orbit it. Anthropic adapters *extract* `system`. Gemini
adapters *pack* `parts`. OpenAI adapters are nearly identity.

Session Service will persist this shape. If Model used a different
one, every turn would convert. Do not.

### What adapters do

Five jobs. Miss one and "we have adapters" is a slogan.

1. **Message translation** — system-in-array vs system parameter vs
   parts.
2. **Parameter mapping** — `max_tokens` vs `max_output_tokens`; drop
   knobs a vendor lacks.
3. **Response normalization** — find the text, usage, finish reason;
   emit `ChatResponse`.
4. **Error translation** — vendor exceptions → `RATE_LIMIT`,
   `TIMEOUT`, `INVALID_REQUEST`, `CONTENT_POLICY`, `UNAVAILABLE`.
   Retries key off *these* types, not string matching `429`.
5. **Streaming normalization** — vendor chunk → `ChatChunk` (next
   section).

Auth and base URL come from config / credential refs, not from
workflow source.

### The adapter implementation pattern

One class per vendor, same interface. Anthropic as the teaching
example (the interesting split):

```
  class AnthropicProvider:
      def chat(self, messages, config):
          system, rest = split_system(messages)
          raw = client.messages.create(
              system=system, messages=to_anthropic(rest), ...
          )
          return ChatResponse(content=raw.content[0].text,
                              usage=map_tokens(raw.usage), ...)
```

OpenAI-compatible endpoints (many self-hosted) share an adapter with
a **registerable base URL**. Do not fork a new class for every Llama
file. Register the endpoint; reuse the translator.

Tests: golden messages in, golden vendor payloads out, and the reverse
for responses. If you only test against live keys, you do not have
adapters; you have hope.

## Streaming responses

Streaming is not a pretty-print of Chat. It changes failure and UX.

### The streaming architecture

**Problem** — Users feel **time-to-first-token**, not time-to-complete.
A 4s JSON blob feels broken. The same tokens arriving from 200ms feel
alive. Also: you may want to abort after a bad first sentence.

**Solution** — A pipeline with a typed fragment at each hop:

```
  provider SSE/iterator
       |
       v
  adapter --> ChatChunk
       |
       v
  gRPC ChatStream (Model Service -> gateway)
       |
       v
  SSE to the browser  (chapter 2)
```

Debug latency by **layer**. If TTFT is bad, is it the vendor, the
adapter buffer, the gateway, or the workflow waiting to assemble
context? Traces should have spans for each. "Streaming is slow" is not
a diagnosis.

### The ChatChunk message

**Problem** — OpenAI `delta.content`, Anthropic events, local servers
with third spellings. Frontends should not care.

**Solution** — One fragment:

```
  ChatChunk
    token          text piece (not a "word"; tokenizers cut oddly)
    index          order, in case the network lies
    finish_reason  stop | length | tool_calls | error | unset
    usage          often only on the last chunk
```

The client appends `token`. It treats `finish_reason` as the only
honest EOF. Do not infer EOF from silence — that is how you hang a
spinner.

### Streaming and error handling

**Problem** — Mid-stream, the vendor dies. Half a paragraph is already
on screen. You cannot replace it with a clean `ChatResponse` error.
Falling over to another model mid-sentence changes voice in a way
users read as possession.

**Solution** — Emit a final chunk with `finish_reason=error`. Keep the
partial text; show a recovery affordance. **Fallbacks apply before
the first byte**, not in the middle, unless you have a product reason
to restart the whole answer (clear the UI, say "retrying"). That is a
trade: streaming buys TTFT and spends recoverability. Document it.

Retries of the *same* stream after a drop are a new request. Do not
pretend they splice.

## Resilience: fallbacks and retries

Outages, 429s, blips. The question is not whether. It is **whose
loop**.

**Problem** — Every workflow's `try/except` with `sleep(1)`. Different
codes. Some retry `INVALID_REQUEST` forever. Some never retry 429.

**Solution** — Sam **declares** policy. The Model Service **runs** it.
Retries for transient types; fallbacks to another adapter when a
provider is exhausted; fail-fast types that must not hop.

### Retry configuration

```
  RetryConfig
    max_retries
    initial_delay
    exponential_backoff
    max_delay
    retry_on   [RATE_LIMIT, TIMEOUT, UNAVAILABLE, ...]
```

Retry 429 and timeouts. Do not retry malformed JSON or a policy
refusal — another vendor will refuse too, and you burn budget. Cap
delay so a dying dependency cannot hold a replica forever (the
workflow still has a deadline from chapter 2).

### Fallback configuration

```
  FallbackConfig
    enabled
    providers[]     ordered alternatives
    retry_config    applied *per* provider
    fail_on[]       skip the rest of the chain
```

Order is preference. Log **which** provider answered. Silent fallback
is how quality dies while dashboards stay green. `fail_on` should
include invalid request and (usually) content policy — hopping is not
a jailbreak tool.

### Configuration examples

**User-facing chat (availability first).** Short retries, then a
second cloud, then a local model so a vendor holiday is not an
outage. Accept that the local model is worse; **score** it
([chapter 7](../7-observability/)) so you know how often you paid
that tax.

**Batch analytics (correctness first).** Longer retries on one
provider, fallbacks **off**. Fail the item, dead-letter, inspect.
Quick hops here hide a poison payload or a key that is wrong.

Do not copy either blob blindly. Write the policy next to the SLO:
"we would rather be down than quietly dumber" vs the reverse.

## Routing strategies

Fallbacks are after failure. **Routing** is before the first attempt,
when `model` is empty or when you asked the service to choose.

**Problem** — Twelve workflows each implement "cheap unless hard."
Spend and load counters are local and wrong.

**Solution** — Strategies in the Model Service, where **budget and
in-flight counts** actually live. Explicit `model=` still bypasses
routing. That is the escape hatch for eval pins.

### Routing configuration

```
  RoutingConfig
    strategy          cost | load | feature | combined
    default_provider  when the tree is indifferent
    cost_config / load_config / feature_config
```

Application supplies the config; platform executes. If every request
names a model, routing never runs — and you will still want it the
month finance asks why.

### Cost-aware routing

**Problem** — Frontier model for "what is your return window?" burns
the month. Then the hard cases have no budget.

**Solution** — Track spend against a window. Near the cap, prefer
local / cheap. Otherwise branch on **task complexity** (heuristic,
classifier, or explicit tag from the workflow).

```
  spend ~ limit? --yes--> cheapest (often local)
        |
        no
        v
  task simple? --yes--> small API model
        |
        no --> frontier
```

Complexity guesses mis-route. That is why eval exists. Cost routing
without scores is how you silently make the assistant worse at
month-end.

### Load-based routing

**Problem** — All traffic on one vendor until 429, *then* fallback.
You discover capacity as an incident.

**Solution** — Track in-flight per provider. Send the next call to
the least loaded. Increment before send, decrement on any completion
(including failure) or you leak counts.

Works when latency is comparable and you hold limits on several
accounts. Poor when one "provider" is a slow local GPU and the other
is a fast API — least-outstanding-requests will fill the slow one.
Pair with features or a latency EWMA if that is your estate.

### Feature-based routing

**Problem** — A PDF-with-figures hop hits a text-only model. You get
a confusing error or a hallucinated "I can't see."

**Solution** — Capability matrix from discovery. Required features
(vision, tools, JSON, window ≥ N) **intersect**. Empty intersection
fails fast. No "try it anyway."

```
  need vision  --> {gpt-4o, claude, ...}
  need 100k    --> {claude, gpt-4o, ...}
  intersection --> route inside that set
```

### Combining patterns

Real apps stack them:

1. **Features** (hard filter).
2. **Cost** (soft preference on the remainder).
3. **Load** (tie-break).

```
  capable = feature_filter(request)
  preferred = cost_rank(capable, spend, complexity)
  pick least_loaded(preferred) or default
```

Sam's assistant can then `chat(messages)` without naming a vendor.
Image turns still land on vision. Month-end still shifts easy FAQ to
local. Do not invert the order: cost must not override "this request
needs eyes."

## Rate limiting

**Problem** — A loop, a load test aimed at prod, a bot. Provider
quotas protect *them*. They may happily take enough traffic to ruin
*you*. Worse: 429 on OpenAI trips fallbacks, so the runaway spends
Anthropic next. Resilience becomes a multiplier.

**Solution** — **Platform-side** limits before any adapter: per key,
per workflow, per tenant, per model class. Fail with a typed
`RATE_LIMIT` the UX can explain. Do not wait for the vendor.

This is org policy, not an app hobby. If Sam can `sleep` around it,
it is not a limit. Align with the gateway's external quotas
([chapter 2](../2-sdk-and-api/)) so you do not double-count without
meaning to — gateway protects ingress; Model Service protects **token
spend**.

## Caching for cost and performance

Two different caches. People say "we cache" and mean one, then wonder
why the bill moved 4%.

### Two levels of caching

```
  request in
     |
     v
  RESPONSE CACHE  (platform)  exact messages+config hit?
     | yes --> return, $0, sub-ms
     no
     v
  adapter call, with PROMPT/PREFIX CACHE (vendor)
     |
     v
  store in response cache (if enabled)
```

**Response cache.** Identical inputs → identical output. FAQ bots
love it. Creative sampling (`temperature` high) should **disable**
it or you will repeat a joke. Keys must not accidentally include
raw PII you would not store; if messages contain secrets, cache is
a privacy system.

**Prompt / prefix cache.** Vendor reuses the long system prompt or
document prefix. You still pay a call, cheaper on the prefix tokens.
Anthropic wants markers; OpenAI may automatic; vLLM prefix-caches by
default. Adapters hide the flags; the workflow can still opt in/out.

### The cache interface

```
  CacheConfig: enabled, ttl_seconds, max_size
  ResponseCache.get(messages, config) -> ChatResponse | None
  ResponseCache.set(messages, config, response)
```

Key = hash(messages, config). Different temperature → different key.
LRU + TTL. Prompt-cache options pass through the adapter, not through
this store.

### Monitoring cache effectiveness

**Problem** — Cache "on" with 5% hit rate. TTL too short, queries too
unique, or temperature randomizing the key.

**Solution** — Hit rate, saved tokens, saved dollars, split by
response-cache vs prefix-cache. Cost accounting: hit = 0; prefix =
discount; miss = list price. If Observability does not know the path,
finance will still see a mystery.

## Observability: cost tracking and metrics

Chapter 1's $2,000 invoice with no feature name is this section's
villain.

### What the model service tracks

Every request, at least:

- **Usage** — prompt, completion, cache-read/write tokens.
- **Identity** — requested model, **actual** model/provider (fallback).
- **Time** — TTFT, total duration, adapter wait vs vendor wait if you
  can split them.
- **Disposition** — ok, retry count, fallback hop, rate-limited,
  cache hit kind, error type.
- **Attribution** — workflow id, tenant, maybe user hash. Without
  this, you have a sum, not a story.

Quality scores are [chapter 7](../7-observability/). This service
must emit the **generation span** those scores hang on.

### Feeding the observability service

**Problem** — Metrics in a sidecar CSV nobody joins.

**Solution** — Publish per call into the Observability Service.
Aggregate by provider, model, workflow, tenant, time. When spend
jumps 40%, you should be able to say "the new summarizer, frontier
model, mid-month" — not "AI."

Dashboards and alerts live there. Model Service stays a **producer**.
If it also becomes the only UI, you will rebuild chapter 7 badly.

### Enabling informed decisions

What the numbers are *for*:

- **Cut cost** — which workflows, which models, which should be
  small-model or cached.
- **Cut latency** — TTFT regressions, fallback storms that mean the
  primary is 429ing (load-balance, do not only retry).
- **Catch anomalies** — error spikes, sudden tokens-per-request
  (prompt bloat, runaway tools).

If nobody is allowed to change routing after looking, you built a
museum. Tie a monthly review to the same dashboards finance sees.

## Integrating with the SDK

Sam still types `platform.models.chat`. [Chapter 2](../2-sdk-and-api/)
gave the skeleton; this is the flesh.

### The ModelClient

Lazy client, `service_name="models"`, stub from generated proto.
Three jobs: channel, protobuf in, Python out.

```
  platform.models  -->  ModelClient(BaseClient)
                            |
                            +-- Chat / ChatStream / ListModels / ...
```

No vendor SDK in the workflow image *required*. Keys stay on the
Model Service. That is half the security win.

### Method implementation pattern

`chat(...)` converts `ChatMessage` lists and configs to protobuf,
attaches routing metadata, calls `stub.Chat`, maps `ChatResponse`.
Pass through `fallback_config` and `routing_config` so policy is
per-call when it must be, defaulted from app config when it need
not be.

Do not hide usage. If the SDK drops `usage`, Observability never
sees what the handler already threw away — and Sam cannot log cost
even in a pinch.

### Streaming support

`chat_stream` returns an iterator. Pull gRPC chunks, yield
`ChatChunk`. The workflow `yield`s to the gateway (chapter 2). Do
not buffer the whole stream in the client "to make it easier." That
destroys TTFT.

### The complete picture

```
  1. GenAIPlatform()           # no sockets yet
  2. platform.models           # channel to gateway
  3. chat(messages, model=...)
  4. GW routes to Model Service
  5. cache? rate limit? route? retry?
  6. adapter --> vendor
  7. ChatResponse back; metrics emitted
  8. workflow uses .content
```

Pin a model in eval. Leave it empty in prod if combined routing is
the policy. Never skip the service "because the SDK can call OpenAI
directly" — that path is how chapter 1 returns.

## What this service is not

Not persona design or sampling craft (Agents ch. 2). Not the agent
runtime's own retry wrapper (Agents ch. 8) — those notes are how
*one process* survives; this is how the **org** does.

Not Session (history) or Data (PDFs). The Model Service will *embed*
for Data later; it does not store Maria's transcript.

Not Experimentation. It must **emit** so experiments can choose
models. It does not own A/B assignment.

## See also

Same words, different job — do not merge the folders.

- **[agents ch. 2](../../agents/2-llms-prompting-agents/)** — tokens,
  temperature, persona, one agent SDK.
- **[agents ch. 8](../../agents/8-deploying-agents/)** — deploy-time
  budgets and routing in an agent process.
- [`TRADEOFFS.md`](../../TRADEOFFS.md) — cost vs load vs features vs
  cache vs fallback.

## Check yourself

1. Why is `Chat` vs `ChatStream` two RPCs that share `ChatRequest`?
   What goes wrong if streaming is a boolean the JSON API half-
   implements?
2. A system prompt is copied in four workflows. Name two incidents
   prompt *hosting* prevents that prompt *wording* (Agents ch. 2)
   cannot.
3. Draw OpenAI vs Anthropic vs Gemini for *system* placement. Why is
   the platform shape OpenAI-like, and what does the Anthropic
   adapter have to do on every call?
4. List the five adapter jobs. Which one must exist before retry
   policy can be correct?
5. Mid-stream vendor death: what does the client see, and why is
   failing over to Claude *mid-sentence* usually the wrong default?
6. User-facing vs batch: sketch two `FallbackConfig`s. What metric
   tells you the user-facing chain is lying about quality?
7. Combined routing: order feature, cost, load. Give a request that
   breaks if you put cost first.
8. Why can provider 429 + fallbacks *increase* spend during a
   runaway loop? Where does a platform rate limit sit relative to
   adapters?
9. Response cache vs prefix cache: which skip the HTTP call, which
   only cheapen it, and when must you disable the first?
10. An invoice jumps. Which dimensions must Model Service emit so
    you can name a workflow, not "AI"? What would you still open
    [agents ch. 2](../../agents/2-llms-prompting-agents/) for?

Continue to [The Session Service](../4-session-service/).
