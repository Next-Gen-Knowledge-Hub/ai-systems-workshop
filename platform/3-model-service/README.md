# 3. The Model Service

Companion notes for **Chapter 3** of *Designing AI Systems* (Suhas Suresha and Dewang
Sultania; Manning MEAP).

This chapter establishes the Model Service. The provider call is about 2% of the system.
The other 98% shares one contract: many vendors, streaming, retries, routing, a cache, and
numbers you can join to a trace. Afterward you can name the RPCs, the five adapter jobs,
when a fallback is legal, and which routing filter runs first. The workflow names a job.
The service names a provider. Without that split, twelve handlers hardcode `gpt-4o`, keys
live in six `.env` files, a self-hosted endpoint needs a flag day, and a $2,000 invoice
cannot name a feature.

## 1. The model service contract

Internal calls are gRPC. The **.proto is the promise.** SDK methods, gateway routing, the
Session Service (the same message shape), and Data (embeddings later) all hang off it. Get
the verbs wrong and you will version the SDK twice a quarter.

```
  service ModelService {
    rpc Chat(ChatRequest)          returns (ChatResponse);
    rpc ChatStream(ChatRequest)    returns (stream ChatChunk);
    rpc ListModels(...)            returns (...);
    rpc GetModelCapabilities(...)  returns (...);
    rpc RegisterPrompt / GetPrompt / ListPrompts ...
    rpc RegisterModel / ...
  }
```

`Chat` and `ChatStream` share `ChatRequest`. Streaming is a delivery mode of that request.
The gateway maps the stream to SSE for browsers.

* **Chat**: One RPC for "send roles, get text and token counts," the call every app was
  reinventing. One team forgets usage. Another cannot pass tools. The request carries
  `model`, `messages` (`role` + `content`), `ChatConfig`, optional `tools` (name and JSON
  schema), and optional `response_format`. The response carries `content` or `tool_calls`,
  the model that actually ran, and `usage` (prompt / completion). Use it for every
  generation, including the ones that only sometimes call a tool. It breaks when a handler
  grows a private body and drops usage or tools.
* **Empty model**: An empty `model` means the service picks, under routing config. After a
  fallback, `ChatResponse.model` is the model that ran. Use an empty name in production when
  combined routing is the policy. It breaks when you log only the requested name and the
  invoice names a model that never ran.
* **Roles**: `system`, `user`, `assistant`, `tool`. Session stores history in these roles,
  and adapters translate them. Use this set on every message. A private role enum becomes a
  second translation on every turn.
* **ChatConfig**: Temperature, max tokens, `top_p` when you expose it, `seed` when a
  provider has it. The service forwards the fields. A knob the vendor lacks dies in the
  adapter. Use one config object in the workflow. It breaks when an unknown knob is
  forwarded and a valid platform request 400s at the vendor.
* **response_format**: JSON schema and the other structured-output modes are a request
  field. Use it when the next step parses the reply. A schema that lives only as a sentence
  in the prompt is a footnote the model can ignore.
* **ChatMessage**: `role`, `content`, optional `tool_calls`, optional `tool_call_id` on
  result rows. Use the tool fields so the next turn can replay the call. It breaks when the
  call is stuffed into free-text `content` and the following generate invents a result.
* **ToolDefinition**: Type `function`, name, description, parameters as JSON. The service
  forwards schemas and returns calls. The workflow runtime executes the tool. Use the
  definition when the model must choose a call. It breaks when someone treats a returned
  `tool_calls` payload as already executed.
* **TokenUsage**: Prompt, completion, and cache-read / cache-write when the vendor reports
  them. Use the vendor's numbers. Inventing a tokenizer count here will disagree with the
  bill.
* **ListModels and GetModelCapabilities**: Name, provider, context window, vision, tools,
  JSON mode, and maybe price class. `GetCapabilities` is the same catalog. Workflows that
  care query at runtime. Routing uses the same list. Use it when a hardcoded `"gpt-4o"`
  might be stale, when a 50-page PDF needs a window that name does not have, or when a
  vision hop might hit a text-only endpoint. The list is a platform catalog: a model on it
  has an adapter and keys. A missing name fails fast. A scrape of marketing pages, or a
  silent default, sends the call to something the platform cannot run.
* **Named prompts**: Prompts are named, versioned documents the service stores. Workflows
  pass `system_prompt_name` and an optional version pin. Chat assembly prepends the current
  body. Eval swaps names without rewriting handlers. Use this when the same persona would
  otherwise be copied into four workflows. It breaks when legal edits one copy and the other
  three drift, or when a wording change deploys unrelated code. Crafting the paragraph is
  product work. Hosting the document is this service.
* **RegisterModel**: Name, base URL, protocol (often OpenAI-compatible), auth ref, and
  advertised capabilities. After registration the same Chat RPC applies. Use it for
  fine-tunes and vLLM boxes that will never appear on a vendor's list. You self-host for
  privacy, unit cost, or domain vocab. Ops cost is real. The on-ramp is an RPC with an ACL.
  A ticket to the platform team with a week SLA produces shadow endpoints that skip cache,
  quotas, and traces.

## 2. Provider adapters

Vendors disagree on the envelope. A **canonical platform message** plus one adapter class
per vendor keeps that table out of product code. An `if provider == "anthropic"` ladder
starts in Sam's handler, then marketing's, then a notebook.

```
  aspect        OpenAI               Anthropic            Gemini
  system        role in messages     top-level system     parts / roles
  body          {role, content}      {role, content}      {role, parts}
  text path     choices[0].message   content[0].text      candidates[0]
  token names   prompt/completion    input/output         prompt/candidates
  stream        SSE delta.content    text_stream ctx      chunks / parts
```

The platform shape is the de facto array of `{role, content}` plus `tool_calls`. Docs,
OpenAI-compatible servers such as vLLM, and Session storage already orbit that array. A
fourth neutral schema would translate twice for every vendor, including the one everyone
already speaks. Anthropic adapters extract `system`. Gemini adapters pack `parts`. OpenAI
adapters are nearly identity. Session persists this shape, so a turn does not convert on
the way through.

Applications see `ModelProvider.chat` and `chat_stream`. The path is `ChatRequest`, then
router or fallback, then `OpenAIAdapter`, `AnthropicAdapter`, or `VLLMAdapter` (an
OpenAI-compatible POST). Adding Google is a new class. The service may retry and fail over.
The adapter stays boring: translate, call, normalize, map errors. Which chain, and how many
retries, is configuration on the request or the app. Execution stays on the platform. Sam
declares the chain. The service runs the loop.

Five jobs. Miss one and "we have adapters" is a slogan.

1. **Message translation**: System-in-array, a top-level system parameter, or parts. Use it
   on every call, because the platform shape is one array. It breaks when an Anthropic body
   still has `system` inside `messages` and the vendor rejects it.
2. **Parameter mapping**: `max_tokens` versus `max_output_tokens`, and dropping knobs a
   vendor lacks. Use it so workflows keep one `ChatConfig`. It breaks when a knob the
   vendor lacks is forwarded and the call fails at the provider.
3. **Response normalization**: Find the text, the usage, and the finish reason, and emit
   `ChatResponse`. Use it so callers read one shape. It breaks when a workflow has to know
   that Gemini text lives at `candidates[0]`.
4. **Error translation**: Vendor exceptions become `RATE_LIMIT`, `TIMEOUT`,
   `INVALID_REQUEST`, `CONTENT_POLICY`, or `UNAVAILABLE`. Use it so retries key off these
   types. It breaks when policy matches the string `429` and misses the same condition
   under another spelling. This job has to exist before a retry policy can be correct.
5. **Streaming normalization**: A vendor chunk becomes a `ChatChunk`. Use it so the gateway
   and the browser see one fragment. It breaks when the frontend branches on SSE
   `delta.content`, a text stream, and parts.

* **Auth and base URL**: Both come from config and credential refs. Use refs so a workflow
  image has no provider key. A key in source is how six `.env` files appear.
* **AnthropicProvider**: `split_system` pulls system text out of the array.
  `client.messages.create(system=..., messages=to_anthropic(rest))` sends the rest. The
  return is `ChatResponse(content=raw.content[0].text, usage=map_tokens(raw.usage))`. Use
  this class for the vendor whose system text is a top-level parameter.
* **One OpenAI-compatible class**: Self-hosted endpoints share an adapter with a
  registerable base URL. Use `RegisterModel` plus that class for a Llama box. A new class
  per file forks the translator.
* **Golden tests**: Messages in, vendor payloads out, and the reverse for responses. Use
  fixtures. A suite that only hits live keys is hope, and it cannot run when the key or the
  vendor is down.

## 3. Streaming responses

The latency users feel is **time-to-first-token**. A 4-second JSON blob feels broken. The
same tokens from 200 ms feel alive. You may also abort after a bad first sentence.
Streaming changes failure and UX together.

```
  provider SSE / iterator
       |
       v
  adapter --> ChatChunk
       |
       v
  gRPC ChatStream (Model Service -> gateway)
       |
       v
  SSE to the browser
```

* **Debug by layer**: A slow stream is a span question: vendor, adapter buffer, gateway, or
  the workflow still assembling context. Use a span on each hop. "Streaming is slow" names
  no layer, so the fix lands on the wrong one.
* **ChatChunk**: `token` is a text piece. Tokenizers cut oddly, so it is a fragment, and the
  client appends it. `index` keeps order when the network reorders. `finish_reason` is
  `stop`, `length`, `tool_calls`, `error`, or unset. `usage` often arrives only on the last
  chunk. Use `finish_reason` as the EOF. Treating silence as the end hangs the spinner.
* **Mid-stream death**: The vendor dies after half a paragraph is already on screen. Emit a
  final chunk with `finish_reason=error`, keep the partial text, and show a recovery
  affordance. Use this when bytes have already reached the client. A clean `ChatResponse`
  error cannot replace text the user has read. **Fallbacks apply before the first byte.** A
  hop to another model mid-sentence changes voice. Restarting the whole answer is a product
  choice: clear the UI and say "retrying." Streaming buys TTFT and spends recoverability.
  Write that down. A retry after a dropped stream is a new request. The bytes do not splice.

## 4. Retries

Outages, 429s, and blips will happen. Whose loop runs is the design choice. Every
workflow's `try/except` with `sleep(1)` grows a private policy. Some retry
`INVALID_REQUEST` forever. Some never retry a 429. Sam declares the policy. The Model
Service runs it, for transient types only.

```
  RetryConfig
    max_retries
    initial_delay
    exponential_backoff
    max_delay
    retry_on    [RATE_LIMIT, TIMEOUT, UNAVAILABLE, ...]
```

* **retry_on**: `RATE_LIMIT`, `TIMEOUT`, and `UNAVAILABLE`. A 429 and a timeout are
  transient. Use the typed errors from the adapter. It breaks when the list is "any
  exception," because a malformed body then retries until `max_retries` is spent.
* **Fail-fast types**: Malformed JSON (`INVALID_REQUEST`) and a content-policy refusal
  (`CONTENT_POLICY`) stay off `retry_on`. The same payload fails again, and the extra
  attempts burn budget. Use these as terminal for the retry loop.
* **Backoff and the cap**: `initial_delay` grows by `exponential_backoff`, up to `max_delay`,
  for at most `max_retries`. The cap stops a dying dependency from holding a replica
  forever. The workflow still carries a deadline from the gateway and the SDK. A retry loop
  with no cap, and no outer deadline, pins a worker to a vendor that is not coming back.

## 5. Fallbacks

A fallback runs when a provider is exhausted. It calls the next adapter. A retry hits the
same provider again. The two configs compose: each provider in the chain gets its own
retry loop, and some errors skip the rest of the chain.

```
  FallbackConfig
    enabled
    providers[]      ordered alternatives
    retry_config     applied per provider
    fail_on[]        skip the rest of the chain
```

* **Order and the log**: `providers[]` is preference order. `retry_config` applies per
  provider. Log which provider answered. Use the log whenever `enabled` is true. A silent
  fallback keeps dashboards green while quality dies.
* **fail_on**: Include `INVALID_REQUEST` and, usually, `CONTENT_POLICY`. Those types skip
  the rest of the chain. Another vendor refuses the same payload, and the hop burns budget.
  A fallback is the wrong tool for shopping a refusal across vendors.
* **User-facing chat**: Availability first. Short retries, then a second cloud, then a local
  model, so a vendor holiday leaves the product up. The local model is worse. Score
  that hop so you know how often you paid the tax. Use this when being up matters more than
  being identical. It breaks when you never score the local answers and the product quietly
  gets dumber.
* **Batch analytics**: Correctness first. Longer retries on one provider, and `enabled`
  false. Fail the item, dead-letter it, inspect it. Use this when a wrong answer is more
  expensive than a delay. A quick hop hides a poison payload or a key that is wrong.
* **The SLO sentence**: Write the policy next to the sentence you mean: "we would rather be
  down than quietly dumber," or the reverse. Copying either blob without that sentence is
  how the wrong chain ships.

## 6. Routing

Fallbacks run after failure. **Routing** runs before the first attempt, when `model` is
empty or when the service is asked to choose. Twelve workflows each implementing "cheap
unless hard" keep spend and in-flight counts local, and wrong. Strategies live where the
budget and the in-flight counts live. An explicit `model=` bypasses routing. That escape
hatch is how eval pins a name.

```
  capable   = feature_filter(request)
  preferred = cost_rank(capable, spend, complexity)
  pick least_loaded(preferred) or default_provider
```

* **RoutingConfig**: `strategy` is `cost`, `load`, `feature`, or `combined`.
  `default_provider` is the pick when the tree is indifferent. `cost_config`,
  `load_config`, and `feature_config` hold the knobs. The application supplies the config.
  The platform executes it. Use `combined` when Sam should call `chat(messages)` without
  naming a vendor. If every request names a model, routing never runs, and the month finance
  asks why you still need the counters.
* **Cost-aware**: Track spend against a window. Near the cap, prefer the cheapest model,
  often local. Under the cap, branch on **task complexity**: a heuristic, a classifier, or
  an explicit tag from the workflow. A simple task goes to a small API model. A hard task
  goes to the frontier. Use it so "what is your return window?" does not burn the month and
  leave the hard cases with no budget. Complexity guesses mis-route. Eval exists for that.
  Cost routing without scores makes the assistant worse at month-end while the bill looks
  controlled.
* **Load-based**: Track in-flight requests per provider and send the next call to the least
  loaded. Increment before send. Decrement on any completion, including failure, or the
  count leaks. Use it when latencies are comparable and you hold limits on several accounts.
  Sending everything to one vendor until a 429, then falling back, discovers capacity as an
  incident. It fails when one provider is a slow local GPU and the other is a fast API:
  least-outstanding-requests fills the slow one. Pair load with features, or with a latency
  EWMA, when that is the estate.
* **Feature-based**: Required features (vision, tools, JSON, window ≥ N) **intersect** the
  capability matrix from discovery. Vision might leave `{gpt-4o, claude, ...}`. A 100k
  window might leave `{claude, gpt-4o, ...}`. Route inside the intersection. An empty
  intersection fails fast. Use it for a PDF-with-figures hop. A text-only model returns a
  confusing error or a hallucinated "I can't see." There is no "try it anyway."
* **Combined order**: Features are the hard filter. Cost is a soft preference on what
  remains. Load is the tie-break. Image turns still land on a vision model. Month-end still
  shifts an easy FAQ to local. Cost first sends a figure-heavy PDF to a cheap text model
  that cannot see the page. Explicit `model=` still skips the whole tree.

## 7. Rate limiting

A loop, a load test aimed at production, or a bot can spend more than you can afford.
Provider quotas protect the provider. They can still accept enough traffic to ruin you. A
429 from OpenAI then trips fallbacks, so the runaway spends Anthropic next. Resilience
becomes a multiplier.

* **Platform limit, before any adapter**: Per key, per workflow, per tenant, and per model
  class. Fail with a typed `RATE_LIMIT` the UX can explain. Use it in front of the adapter
  call, on the same path as the response-cache lookup: both are the gate. Waiting for the
  vendor's 429 is how the fallback multiplier starts.
* **Org policy**: A limit Sam can `sleep` past is only a delay. Align the numbers with the
  gateway's external quotas so the two layers double-count only when you mean them to. The
  gateway protects ingress. The Model Service protects **token spend**. Two accidental
  counters reject traffic the budget still allowed.

## 8. Caching

Two caches. A team that says "we cache" and means one of them will wonder why the bill
moved 4%.

```
  request in
     |
     v
  RESPONSE CACHE (platform)    exact messages + config?
     | hit --> return, $0, sub-ms
     miss
     v
  adapter call, with PROMPT / PREFIX CACHE (vendor)
     |
     v
  store in the response cache when it is enabled
```

* **Response cache**: Identical inputs return the stored output and skip the HTTP call. FAQ
  bots are the fit. The key is `hash(messages, config)`, so a different temperature is a
  different key. `CacheConfig` is `enabled`, `ttl_seconds`, and `max_size`, with
  `ResponseCache.get(messages, config)` and `set(messages, config, response)`, LRU plus TTL.
  Use it for low-temperature answers you are willing to repeat. High `temperature` should
  turn it off, or the bot repeats a joke. A key that contains raw PII you would not
  otherwise store makes the cache a privacy system. Secrets in the messages become secrets
  in the store.
* **Prompt / prefix cache**: The vendor reuses a long system prompt or a document prefix.
  You still pay for a call. The prefix tokens are cheaper. Anthropic wants markers. OpenAI
  may do this automatically. vLLM prefix-caches by default. Adapters hide the flags. The
  workflow can still opt in or out. Those options pass through the adapter, on the vendor
  call. They are a different store from `ResponseCache`. Use the prefix cache for a stable
  preamble in front of a unique user turn. It breaks when you expect it to skip the call, or
  when a marker the vendor requires is never set, and you pay list price for a prefix you
  thought was discounted.
* **What you measure**: Hit rate, saved tokens, and saved dollars, split by response cache
  versus prefix cache. A cache that is "on" at a 5% hit rate usually has a TTL that is too
  short, queries that are too unique, or a temperature that randomizes the key. Cost
  accounting: a response hit is 0, a prefix hit is a discount, a miss is list price. If
  Observability never sees the cache path, finance still has a mystery.

## 9. Cost and metrics

The failure this section exists to stop is a $2,000 invoice with no feature name. Every
call emits tokens, dollars, latency, the cache path, and the provider.

* **Usage**: Prompt tokens, completion tokens, and cache-read / cache-write when the vendor
  reports them. Use the adapter's normalized `TokenUsage`. A missing usage field means the
  bill cannot be joined to the call.
* **Identity**: The requested model, and the **actual** model and provider after a fallback.
  Use both. Logging only the request hides which cheap model answered.
* **Time**: TTFT, total duration, and adapter wait versus vendor wait when you can split
  them. Use the split when "streaming is slow" has to name a layer. One number cannot.
* **Disposition**: Ok, retry count, fallback hop, rate-limited, cache-hit kind, and error
  type. Use the hop and the error type together. A green success rate during a fallback
  storm means the primary is returning 429. Load-balance that primary. Retries during the
  storm multiply the same failure.
* **Attribution**: Workflow id, tenant, and maybe a user hash. Use these on every call.
  Without them you have a sum, and the sum is a story that only says "AI."
* **Generation span**: Quality scores live with experimentation. This service emits the span
  those scores hang on. Use the span id so a bad score can find the model, the prompt name,
  and the token counts. A score with no span is a number you cannot debug.
* **Producer**: Publish each call to the Observability Service. Aggregate by provider,
  model, workflow, tenant, and time. A sidecar CSV nobody joins teaches nothing. A 40% spend
  jump should read "the new summarizer, frontier model, mid-month." Dashboards and alerts
  live in Observability. The Model Service stays a producer. If it is also the only UI, you
  rebuild experimentation in the wrong service.
* **What the numbers are for**: Cut cost by naming the workflow, the model, and what should
  be a small model or a cache hit. Cut latency from TTFT regressions and from fallback
  storms. Catch anomalies: error spikes, and a sudden rise in tokens per request from prompt
  bloat or runaway tools. A monthly review on the same dashboards finance sees is how
  routing is allowed to change. Numbers nobody may act on are a museum.

## 10. SDK integration

Sam still types `platform.models.chat`. This is the flesh on the SDK skeleton. The workflow
image can omit every vendor SDK. Keys stay on the Model Service. That is half the security
win.

```
  1. GenAIPlatform()             # no sockets yet
  2. platform.models             # channel to the gateway
  3. chat(messages, model=...)   # or chat_stream
  4. gateway routes to Model Service (x-target-service: models)
  5. response cache?  rate limit?  route?  retry / fallback?
  6. adapter --> vendor
  7. ChatResponse back; metrics emitted (tokens, $, latency, cache path, provider)
  8. workflow reads .content and .usage
```

* **ModelClient**: A lazy `BaseClient` with `service_name="models"` and a stub from the
  generated proto. Three jobs: open the channel, protobuf in, Python out. Methods cover
  Chat, ChatStream, ListModels, GetModelCapabilities, and the prompt and register calls.
  Use the same lazy channel as the other platform clients. A client that constructs a
  vendor SDK inside the workflow puts the key back in the image.
* **chat**: Convert `ChatMessage` lists and configs to protobuf, attach routing metadata,
  call `stub.Chat`, and map `ChatResponse`. Pass `fallback_config` and `routing_config`
  through, so policy can be per call, and default them from app config when the call does
  not care. Keep `usage` on the returned object. If the SDK drops it, Observability never
  sees the tokens, and Sam cannot log cost on the handler either.
* **chat_stream**: Returns an iterator. Pull gRPC chunks and yield `ChatChunk`. The workflow
  yields those to the gateway. Use the iterator when the user is watching the reply.
  Buffering the whole stream inside the client destroys TTFT. The 200 ms first token
  becomes the 4-second blob from section 3.
* **When to pin**: Pin `model` in eval. Leave it empty in production when combined routing
  is the policy. The direct-to-OpenAI path on the SDK skips the service: no shared cache, no
  org rate limit, no fallback, and no workflow id on the invoice. That path is how the $2,000
  line comes back with no feature name.

## Check yourself

1. Why do `Chat` and `ChatStream` share one `ChatRequest`? What breaks if streaming is a
   boolean a JSON API only half-implements?
2. Legal edits the persona in one of four copied workflows. Name two incidents that hosting
   a named, versioned prompt prevents, and that a better paragraph still cannot.
3. Where does system text sit for OpenAI, Anthropic, and Gemini? Why is the platform
   message the `{role, content}` array plus `tool_calls`, and what does the Anthropic
   adapter extract on every call?
4. List the five adapter jobs. Which one has to land before retries can key off
   `RATE_LIMIT` instead of the string `429`?
5. The vendor dies after half a paragraph is on screen. What `finish_reason` does the
   client see, what happens to the partial text, and why is a mid-sentence hop to another
   model the wrong default?
6. Sketch one user-facing `FallbackConfig` and one batch `FallbackConfig`. What do you
   score so the availability chain cannot lie about quality?
7. Combined routing filters by features, then cost, then load. Name a request that breaks
   if cost runs first, and say where that request lands.
8. A runaway loop gets an OpenAI 429 while fallbacks are on. Why can the bill rise, and
   where does a platform rate limit sit relative to the adapters?
9. Which cache returns at $0 in sub-millisecond time, and which still makes the HTTP call?
   When must the first be disabled, and what does a 5% hit rate usually mean?
10. Spend jumps 40%, or the invoice is $2,000 with no feature name. Which fields must each
    call emit so the line can read "the new summarizer, frontier model, mid-month"?
