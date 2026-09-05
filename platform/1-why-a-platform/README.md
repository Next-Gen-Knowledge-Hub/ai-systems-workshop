# 1. Why your AI projects need a platform

Companion notes for **Chapter 1** of *Designing AI Systems* (MEAP, Suhas
Suresha and Dewang Sultania; Manning). This MEAP's printed contents page is
**single-level** (nine chapter titles). The headings below follow the
chapter's actual topics.

This chapter is the map for the whole Platform track. Prototypes win demos.
Production needs sessions, budgets, guardrails, retrieval, traces, and a way
to ship the *next* AI feature without copying last quarter's Redis glue.
Skip this chapter and every later "service" looks like enterprise theater.

The Agents track is a different book. If you need "what an agent is, five
layers, SPAL," that is [agents ch. 1](../../agents/1-rise-of-ai-agents/).
This folder stays on **why the surrounding platform exists**.

## The mental model

```
                         +------------------ 98% ------------------+
                         | gateway, identity, quotas, deploy       |
                         | model adapters, routing, cache          |
                         | sessions + long-term memories           |
                         | indexes, chunk, embed, hybrid search    |
                         | tools, credentials, MCP, guardrails     |
                         | traces, scores, experiments             |
                         | workflows as services                   |
                         +-----------------------------------------+
                                           ^
                                           |  ~2%
                                      provider.chat()
```

The one sentence to remember: **the model call is a leaf, not the tree.**
Sculley et al.'s 2015 "Hidden Technical Debt in Machine Learning Systems"
said model code was a sliver of an ML system. GenAI made the leaf even
smaller and grew new rings: conversational state, token cost, tool
isolation, retrieval, judges.

## The AI Wild West: why winging it does not scale

The chapter's running story is Sam: two weeks to a dazzling support-bot
demo, then production.

### The production wake-up call

What breaks is boring and expensive, and it is always the same list:

- **Latency becomes a distribution**, not a demo average. Without traces you
  cannot say whether the model, the retrieval, or a lock in *your* code
  stalled.
- **Concurrency.** A marketing blast is a load test you did not schedule.
  Scripts that "work on my laptop" have no queue, no limit, no degradation.
- **Cost without attribution.** The invoice jumps; nobody can say which
  feature or tenant did it.
- **Safety as a prompt paragraph.** Users will ask for financial advice,
  jailbreaks, or PII. Prompt-only policy will lose.
- **Stale knowledge.** FAQs hardcoded for the demo rot the week legal
  updates the PDF.
- **No experiment loop.** The next prompt change is a hope.

### Prototype vs production

Write the gap as a table for *your* system, not Sam's. Prototype reality is
"tolerable for five people." Production reality is SLOs, concurrent users,
audit, cost alerts, and a knowledge base that updates without a deploy.

If you cannot fill the production column, you are not ready to promise a
date. You are ready to list **services** you would have to invent.

### What if the infrastructure already existed?

The rhetorical move of the book: every fire Sam hit is a **named service**
later chapters build — observability, rate limits, cost, guardrails, data,
evaluation. You do not need to implement them this week. You need to stop
calling them "the app."

### AI sprawl

Sam's mess **multiplies**. Four teams, four session stores (Redis, Postgres,
files), four key-handling stories, four log formats. Security cannot answer
"which bots see customer data?" Locally smart choices sum to a brittle
estate. New features take months because the maze is the work.

Sprawl is not caused by stupid engineers. It is caused by **uncoordinated
progress**. A platform is how you make the secure, observable path the
default.

## The model is about 2% of the story

Traditional ML (2015-shaped support classifier): load weights, encode
ticket, `argmax`. Maybe 5–10% of the repo is "the model." The rest is data
pipelines, training cadence, serving, monitoring.

Modern GenAI support: three lines to `chat.completions.create`. The **98%**
is not optional decoration:

- knowledge ingestion and retrieval,
- safety and compliance,
- multi-turn sessions and memories,
- tool execution and credentials,
- tracing, metrics, cost,
- multi-provider routing,
- deploy, scale, configuration.

If your architecture diagram is a single box labeled "LLM," you have drawn
the demo, not the system.

## First principles: what applications actually need

LangChain, LlamaIndex, Bedrock, Azure OpenAI, and friends are real. This
book still wants you to **see the patterns**, so you can debug, avoid
accidental lock-in, and fill gaps. The platform it builds is a teaching
implementation *and* a blueprint you could subset.

Needs you can test a design against:

- **Context-aware intelligence** — org knowledge *and* conversational state
  (two services, not one blob).
- **Orchestration** — more than one model/tool hop, with a lifecycle.
- **Safe action** — tools with isolation and policy, not raw keys in the
  prompt.
- **Visible quality and cost** — traces you can join to scores and dollars.
- **A developer path** — SDK + one deploy story so the fifth team does not
  invent a fifth HTTP wrapper.

## Platform blueprint

The rest of DAS is one service per concern. Independent deployability,
shared patterns (contracts, adapters, telemetry).

```
  App / SDK  -->  API gateway  -->  Workflow (the actual assistant)
                      |                    |
                      |         +----------+-----------+
                      |         v          v           v
                      |     Model      Session        Data
                      |     Service    Service        Service
                      |         |          |            |
                      |         +------+---+------------+
                      |                v
                      |         Tools + Guardrails
                      |                v
                      +----->  Observability + Experimentation
```

- **Model Service** — [ch. 3](../3-model-service/): vendors behind one
  contract, streaming, retries, routing, cache.
- **Session Service** — [ch. 4](../4-session-service/): history and
  memories, token budget.
- **Data Service** — [ch. 5](../5-data-service/): indexes and hybrid
  search.
- **Tools and guardrails** — [ch. 6](../6-tools-and-guardrails/).
- **Observability / experiments** — [ch. 7](../7-observability/).
- **Workflow Service** — [ch. 8](../8-workflow-service/): the unit you
  deploy.
- **SDK and API** — [ch. 2](../2-sdk-and-api/): how developers *feel* all
  of the above.

Agent graphs you might *run inside* a workflow:
[agents ch. 4](../../agents/4-multi-agent-systems/) and
[agents ch. 9](../../agents/9-agentic-loop/). Mention, do not rewrite.

## Platform in action

Trace one user question — "return policy for electronics?" — without
collapsing it into `openai.chat()`:

1. Gateway authenticates, starts a **trace**.
2. Workflow loads **session** (Maria already asked about a laptop).
3. **Data** retrieves the current policy chunk, not last year's demo FAQ.
4. **Model** generates with that context; maybe **tools** if she asks to
   start a return.
5. **Guardrails** check input and output (and tool args if any).
6. **Observability** records tokens, latency, a quality score; cost
   attributed to the support product, not "misc AI."

That dance is the book. Chapter 9 rebuilds it as a named assistant.

## Why platform thinking changes everything

Shared session and cost code is not DRY-for-its-own-sake. It is how
security policy and learning (a good prompt, a good chunk size) **escape
the team that found them**. The default path becomes the audited path.

You can still buy a vendor platform. You cannot skip naming these
capabilities, or Sam's story repeats with better slide decks.

## Check yourself

1. List six production failures from Sam's wake-up call. For each, name the
   *service* in this book's blueprint that would have owned it.
2. Why is "we use LangChain" not an answer to sprawl by itself?
3. Explain the 2% / 98% split to a manager who only sees the OpenAI
   invoice. What on the invoice is *not* model quality?
4. Data Service vs Session Service: a user says "as I mentioned yesterday,
   our enterprise discount." Which service should answer, and what goes
   wrong if you stuff that fact only into a vector index of PDFs?
5. Four teams, four session stores: pick one security question leadership
   cannot answer, and one learning that cannot spread. How does a platform
   change both without blocking the teams?
6. Draw the path of Maria's question through at least five components.
   Where would you put a rate limit, and why not only at the provider SDK?
7. A colleague wants to "just productionize the notebook." What is the
   first non-model component you would demand, and what failure does it
   prevent?
8. Hidden Technical Debt (2015) vs GenAI: name two infrastructure rings
   that *did not exist* for a sklearn classifier and now dominate the
   design.
9. When would you *not* build an internal platform (buy, or stay on one
   script)? Give a criterion from this chapter, not from vendor marketing.
10. This workshop keeps Agents and Platform tracks separate. In one
    sentence, what would you still have to learn from
    [agents ch. 1](../../agents/1-rise-of-ai-agents/) after finishing this
    folder?

Continue to [SDK and API design](../2-sdk-and-api/).
