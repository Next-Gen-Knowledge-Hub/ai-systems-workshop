# 1. Why your AI projects need a platform

Companion notes for **Chapter 1** of *Designing AI Systems* (MEAP, Suhas
Suresha and Dewang Sultania; Manning). This MEAP's printed contents page is
**single-level** (nine chapter titles). The headings below follow the
chapter's actual topics.

This chapter is the map for the whole Platform track. Prototypes win demos.
Production needs sessions, budgets, guardrails, retrieval, traces, and a way
to ship the *next* AI feature without copying last quarter's Redis glue.
Skip this chapter and every later "service" looks like enterprise theater.

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

What breaks is boring and expensive, and it is always the same list.
Latency becomes a distribution. Without traces Sam cannot say whether
the model, the retrieval, or a lock in his own code stalled a request.
Concurrency arrives as a marketing blast: a load test nobody scheduled.
Scripts that worked on a laptop have no queue, no limit, no degradation
path. Cost arrives without attribution. The invoice jumps and nobody
can say which feature or tenant did it. Safety that lived as a prompt
paragraph meets users who ask for financial advice, jailbreaks, or PII.
Prompt-only policy loses. Knowledge goes stale: FAQs hardcoded for the
demo rot the week legal updates the PDF. There is no experiment loop,
so the next prompt change is a hope dressed as a release.

Each of those failures already points at a named capability: traces,
limits, cost meters, guardrails, retrieval, evaluation. Sam did not hit
"AI problems." He hit missing platform services.

### Prototype vs production

Write the gap as a table for *your* system. Prototype reality is
"tolerable for five people." Production reality is SLOs, concurrent
users, audit, cost alerts, and a knowledge base that updates without a
deploy.

If you cannot fill the production column, you are not ready to promise a
date. You are ready to list **services** you would have to invent.

### What if the infrastructure already existed?

The rhetorical move of the book is simple. Every fire Sam hit is a
named service — observability, rate limits, cost, guardrails, data,
evaluation. You do not need to implement them this week. You need to
stop calling them "the app." Once those capabilities have names, "we
will glue Redis again" stops sounding like pragmatism and starts
sounding like sprawl.

### AI sprawl

Sam's mess **multiplies**. Four teams, four session stores (Redis,
Postgres, files), four key-handling stories, four log formats. Security
cannot answer "which bots see customer data?" Locally smart choices sum
to a brittle estate. New features take months because navigating the
maze *is* the work.

Sprawl is not caused by stupid engineers. It is caused by uncoordinated
progress. Each team solved yesterday's demo. None of them shared a
default path. A platform is how you make the secure, observable path
the one people take without a ticket.

## The model is about 2% of the story

Traditional ML (2015-shaped support classifier): load weights, encode
a ticket, `argmax`. Maybe 5–10% of the repo is "the model." The rest is
data pipelines, training cadence, serving, monitoring.

Modern GenAI support: three lines to `chat.completions.create`. The
**98%** is not optional decoration. It is knowledge ingestion and
retrieval, safety and compliance, multi-turn sessions and memories,
tool execution and credentials, tracing and metrics and cost,
multi-provider routing, deploy and scale and configuration.

If your architecture diagram is a single box labeled "LLM," you have
drawn the demo. The leaf is real. The tree is what keeps it useful.

## First principles: what applications actually need

LangChain, LlamaIndex, Bedrock, Azure OpenAI, and friends are real.
This book still wants you to **see the patterns**, so you can debug,
avoid accidental lock-in, and fill gaps. The platform it builds is a
teaching implementation and a blueprint you could subset.

Needs you can test a design against:

- **Context-aware intelligence** — org knowledge *and* conversational
  state as two services. Stuffing both into one blob is how "as I
  mentioned yesterday" disappears into a PDF index.
- **Orchestration** — more than one model or tool hop, with a lifecycle
  you can observe and interrupt.
- **Safe action** — tools with isolation and policy. Raw keys in the
  prompt are a credential leak waiting for a jailbreak.
- **Visible quality and cost** — traces you can join to scores and
  dollars.
- **A developer path** — an SDK and one deploy story so the fifth team
  does not invent a fifth HTTP wrapper.

## Platform blueprint

The rest of *Designing AI Systems* is one service per concern.
Independent deployability, shared patterns: contracts, adapters,
telemetry.

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

Name the pieces so Sam's fires have owners. The **Model Service** puts
vendors behind one contract: streaming, retries, routing, cache. The
**Session Service** owns history and memories under a token budget.
The **Data Service** owns indexes and hybrid search. **Tools and
guardrails** own credentials, isolation, and policy checks. **Observability
and experimentation** own traces, scores, and comparisons. The
**Workflow Service** is the unit you deploy — the actual assistant as a
service. The **SDK and API** are how developers *feel* all of the
above without drowning in it.

## Platform in action

Trace one user question — "return policy for electronics?" — without
collapsing it into `openai.chat()`.

Maria opens the support app. The gateway authenticates her request and
starts a **trace**. The workflow loads her **session**: she already
asked about a laptop yesterday. **Data** retrieves the current policy
chunk, not last year's demo FAQ. The **model** generates with that
context. If she asks to start a return, **tools** run under
credentials the workflow never sees. **Guardrails** check input and
output, and tool arguments if any. **Observability** records tokens,
latency, a quality score. Cost is attributed to the support product,
not to a bucket labeled "misc AI."

That dance is the platform. Every hop is a service with a job, and the
model call is one hop among many.

## Why platform thinking changes everything

Shared session and cost code is not DRY-for-its-own-sake. It is how
security policy and learning — a good prompt, a good chunk size —
escape the team that found them. The default path becomes the audited
path. The next team inherits the win instead of rediscovering Redis.

You can still buy a vendor platform. You cannot skip naming these
capabilities. If you skip the names, Sam's story repeats with better
slide decks: another demo, another production wake-up, another estate
of four session stores.

## Check yourself

1. List six production failures from Sam's wake-up call. For each, name
   the *service* in this book's blueprint that would have owned it.
2. Why is "we use LangChain" not an answer to sprawl by itself?
3. Explain the 2% / 98% split to a manager who only sees the OpenAI
   invoice. What on the invoice is *not* model quality?
4. Data Service vs Session Service: a user says "as I mentioned
   yesterday, our enterprise discount." Which service should answer,
   and what goes wrong if you stuff that fact only into a vector index
   of PDFs?
5. Four teams, four session stores: pick one security question
   leadership cannot answer, and one learning that cannot spread. How
   does a platform change both without blocking the teams?
6. Draw the path of Maria's question through at least five components.
   Where would you put a rate limit, and why not only at the provider
   SDK?
7. A colleague wants to "just productionize the notebook." What is the
   first non-model component you would demand, and what failure does it
   prevent?
8. Hidden Technical Debt (2015) vs GenAI: name two infrastructure rings
   that *did not exist* for a sklearn classifier and now dominate the
   design.
9. When would you *not* build an internal platform (buy, or stay on one
   script)? Give a criterion from this chapter, not from vendor
   marketing.
10. In one sentence: what does this chapter teach that a single
    `provider.chat()` call cannot?
