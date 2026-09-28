# 11. Field tips

Companion notes for **Chapter 11** of *AI Agents in Action* (2nd edition,
Micheal Lanham; Manning, 2026).

Architecture is behind you. This chapter is **what survives contact with
a queue**. Skip it and you will rebuild the same support bot, the same
RAG blob, and the same "research" crawler that never stops — each time
surprised that the failure was a *layer*, not a model.

These notes are checklists and shapes, not a third framework. When an
incident cannot be tagged to a layer, the five-layer map is not yet in
your bones.

## The mental model

Tips are cheap if they are a pile. They stick if they hang on the same
five shelves you have used since the start of the Agents track.

```
  +--------------------------------------------------------------+
  | 5  Evaluation / feedback    traces, judges, HITL, guardrails |
  +--------------------------------------------------------------+
  | 4  Knowledge / memory       RAG tool, session vs long-term   |
  +--------------------------------------------------------------+
  | 3  Reasoning / planning     ReAct, plan-then-do, caps, review|
  +--------------------------------------------------------------+
  | 2  Tools / actions          one job each, schemas, failures  |
  +--------------------------------------------------------------+
  | 1  Persona                  contract, scope, I-don't-know    |
  +--------------------------------------------------------------+
```

The one sentence to remember: **optimize a layer, then integrate** — a
witty persona will not save a blunt tool; a perfect index will not save a
loop with no stop.

Three product shapes in this chapter reuse the stack. They are not new
layers. They are **how the stack is wired for a job**:

```
  support     persona = role + policy
              tools   = lookup, ticket APIs, escalate
              reason  = triage + HITL
              knowledge = manuals (RAG)
              eval    = logs + humans

  RAG system  often several *small* agents around retrieval
              (router, retriever, answerer, optional critic)

  research    outer task loop: planner + stateless workers
              + critic + a termination story
```

If your design review cannot point at this diagram, you are still arguing
about models.

## Field tips by the five layers

Walk the stack bottom-up the way you debug: persona first (did we even
ask for the right job?), then hands, then thought, then stores, then
proof.

### Persona

Treat instructions as an **API contract**, not as brand copy. Downstream
tools, memory, and plans inherit whatever you leave vague.

A prompt that reads like a novel buries the one rule that mattered under
recency drift. The bot does extra jobs because nobody said no. Keep a
short role, hard boundaries, and an explicit off-ramp.

- **Role and fences.** What it does, what it refuses, who it defers to.
  Repeat the lethal rules at the **top and bottom**. Models drift; your
  "never refund without lookup" should survive a long tool trace.
- **I-don't-know as a first-class action.** Not a vibe. A named behavior:
  ask, retrieve, or escalate — do not complete a policy from parametric
  memory.
- **Narrow charter.** One job beats a "digital employee." Specialized
  agents are easier to eval and cheaper to route. Add domains later, on
  purpose.
- **Structured outputs** when another program (or agent) consumes the
  result. JSON that matches a schema (Pydantic / dataclass) kills a class
  of parse bugs and token padding. Prose is for humans.
- **Dynamic instructions** for facts that rot (date, tenant, product
  version, user locale). Inject with a function at run time. Do not
  hardcode Tuesday's promo in a prompt you will not reopen. Do not inject
  *everything* — over-injection is another dumping ground.

Field rule: if a constraint is a **fact about the company**, it wants
layer 4 (knowledge you retrieve). If it is **how to behave**, it stays
in the persona. Mixing them is how legal ends up in a system prompt no
one owns.

### Tools and actions

Tools are hands and senses. A sloppy schema makes a smart model look
drunk. A good schema makes a smaller model look competent.

God-tools ("do_anything"), mushy docstrings, missing timeouts, and MCP
servers pulled in because a blog post starred them are the usual
failures. Prefer one responsibility, typed params, and failure as part
of the contract.

- **Single-responsibility tools.** Name, description, and arguments are
  the model's user manual. If you cannot say when *not* to call it, the
  model will call it as a fidget.
- **Prefer function tools** for *your* code and APIs. Decorators that
  build schemas from types and docstrings are easier to test than a
  hand-written JSON blob that drifted from the function.
- **Control when tools run.** Force a tool when the turn is worthless
  without live data. Forbid tools when the turn is a rewrite. Stop after
  the first sufficient call when one API result is the answer. Say this
  in the persona *and* in tool settings — prompts without runtime knobs
  are wishes.
- **Hosted / prebuilt tools and MCP** for code, web, files, when the
  server is actually maintained. Due diligence means reading the package,
  checking who publishes it, and threat-modeling what it can reach. A
  random npx package is a supply chain, not a feature.
- **Plan for failure.** Timeouts, retries with a budget, and **error
  payloads the model can read** (status, what was tried, what to do
  next). Silent exceptions are how agents invent success.

Bucket tools by what they do: retrieve, mutate, memory, plan, eval.
Guardrails attach to buckets, not to "tools" as a pile.

### Reasoning and planning

Frontier models already "think" in the forward pass. You add *structure*
when the horizon is long, tools compete, or a wrong act is expensive.

ReAct on a FAQ, a tree on "what's my order status," a loop with no cap,
or no last look before send are mismatches. Match the primitive to
difficulty; cap; glance back.

- **ReAct (or an honest CoT) for nontrivial work.** Thought, act,
  observation, *then* the next thought. A single CoT with a tool call
  stuffed in is still one pass.
- **Plan-then-execute for big goals.** A short checklist in structured
  state (or a sequential-thinking server) beats improvising twenty hops.
  The plan is data. You can print it.
- **Limit iterations.** Hard cap, then a useful failure ("I hit the
  budget; here is what I have; here is the missing slot") — a
  termination story, not a hung tab.
- **Self-review before final** on anything irreversible or user-visible
  at length. A cheap second pass catches "we cited the TOC" and "we
  emailed the wrong tenant." Skip it on one-shot classification.
- **Turn reasoning down** when the task is one-shot and the model is
  strong. Paying for a chain on "format this JSON" is how bills grow
  while quality stays flat.

If you already run a cognitive shop floor with attention picking the
primitive, the field tip still applies on a single ReAct agent: caps and
a last look are not optional because you skipped the full architecture.

### Knowledge and memory

Retrieval is how knowledge and memory stay honest. Stuffing PDFs into
the prompt is not a strategy; it is a demo that dies on the second
document.

Docs in the system prompt, the session transcript treated as the
corporate wiki, one giant vector index, and embeddings chosen because
they were the default are the usual traps. Prefer RAG as a **tool**;
keep session short-term and long-term separate; prune; hybrid; pick
embeddings on purpose.

- **RAG first-class.** A retrieval tool over a store with **metadata
  filters** (product, version, locale, tenant). The model should have to
  *call* it, not hope the right paragraph was pre-stuffed.
- **Session vs long-term.** The thread is short-term working memory.
  User facts and preferences that should survive a new chat belong in a
  store you retrieve *selectively*. Do not dump the full graph into
  every prompt.
- **Prune and shard.** ANN indexes (HNSW / IVF), **domain shards**,
  filters. Scale is a retrieval problem before it is a model problem.
- **Hybrid search.** Dense similarity misses SKUs and error codes.
  Keyword / BM25 misses paraphrase. Graph or hierarchical search earns
  its keep on "related to this entity." Combine; do not pick a religion.
- **Embeddings on purpose.** Dimensions, quality, multilingual or
  domain-tuned variants, **cost per million tokens at ingest and query**.
  Changing embedding families later is a re-index, not a config flip.

Org-wide indexes and session token budgets live in platform services.
Mention that only as a reminder: the agent still has to *choose* when to
retrieve.

### Evaluation and feedback

You will not improve what you do not measure. You also will not ship a
perfect Phoenix cathedral on day one. Start with **logs, a tiny eval set,
and a guardrail that can say no**. Grow.

Demo energy, no traces, prompt edits with no score, and humans yelling in
Slack with nowhere to put the yell are the usual starting state. Trace,
automate a slice, listen to humans, encode policy as code.

- **Trace everything you will debug at 2 a.m.** Prompts, tool args and
  results, tokens, latency, outcome. SDK tracing and Phoenix exist so
  you are not grepping stdout. Ship those traces to the observability
  stack you already have.
- **Automate evals** on a curated set: accuracy, grounding, safety,
  resolution. Run on prompt / model / index changes. LLM-as-judge is a
  tool, not a priest — pin versions and spot-check with humans.
- **HITL.** Thumbs, reasons, and queues beat a buried "feedback" link.
  Support agents especially: the human who took the escalation *is* the
  label.
- **Guardrails and moderation.** Schema validation, content policy,
  grounding ("claim in retrieved span?"). Put gates in the loop (on
  workspace confidence, schema, or a critic) and judges around the loop.
  Prompt-only "be safe" is not this layer.

Org-wide scores, datasets, and A/B experiments are a platform concern.
Stay here when you need the **agent's** eval loop.

## Tips for a customer support agent

Support is the shape most teams ship first and regret first: it talks to
angry humans, it can mutate tickets, and "almost right" is a compliance
event. Internal IT helpdesks are the same shape with a different persona.

Map the layers without inventing new ones:

| Layer | In this product |
|---|---|
| Persona | Support role, tone, **policy fences**, escalation manners |
| Tools | Order lookup, subscription, refund *or* "create ticket," never both as one god-tool |
| Reasoning | Triage (intent, risk, missing slots), then act or HITL |
| Knowledge | RAG over manuals, policies, versioned SKUs |
| Eval | Traces, CSAT / resolution, human review of escalations |

**Narrow the charter.** Orders, returns, status — then add billing, then
add device troubleshooting. A bot that also writes marketing copy will
hallucinate a coupon. Light and focused is not a MVP cop-out; it is how
you keep latency and eval tractable.

**Ground every answer that is a policy or a fact.** Retrieval first; cite
or quote the span; **I-don't-know / escalate** when the store is empty.
Coverage bluff — fluent invention when retrieval is empty — is this
product's default failure.

**Safe actions.** Read-only lookups can be eager. Mutations (refund,
cancel, PII change) want **confirmation, identity, and often a human**.
Tool policy is the guardrail: the model should not even *see* a refund
tool until triage says the intent is refund *and* the order lookup
succeeded.

**Triage is the plan.** Classify: can we answer from docs, do we need an
API, is this rage / legal / safety, is a slot missing (order id)? Missing
slots are questions, not guesses. High-risk intents skip the happy path
and go to HITL with a packet (what was tried, what was retrieved).

**Memory is small and purposeful.** Session: the ticket so far. Long-term:
"this user prefers email, this account is on plan X" — retrieved, not
stuffed. Do not memorize every chat as policy.

**Eval that matches the job.** Resolution rate, escalation rate, grounding
fails, **wrong-order refunds** (that is a tools-layer plus eval-layer
incident, not a "tone" incident). Sample traces weekly. A witty persona
with a wrong refund is not a persona win.

A support agent that cannot escalate cleanly is not autonomous. It is
stuck. Design the handoff payload as carefully as the greeting.

## Design patterns for a RAG agent system

RAG is usually **infrastructure for another agent** (support, research,
internal Q&A), not a product with one box labeled "RAG." Retrieval is the
engine. Agentic control is routing, critique, and *when* to retrieve
again.

Teams either wrap an orchestrator around a one-shot retrieve that already
worked, or build a single mega-agent that retrieves, answers, critiques,
and files Jira. Prefer agents only for jobs one retrieve-and-generate
cannot do. Keep them small.

```
  query
    |
    v
  [ triage / router ] -- which index, which rewrite, skip RAG?
    |
    v
  [ retriever ]  -- hybrid + filters + optional rerank
    |
    v
  [ answerer ]   -- "use only context"; citations required
    |
    +---- optional [ critic / CRAG ]
              low grounding --> rewrite query or different shard
              else --> release
```

**Use agents only when needed.** One-shot RAG (embed query, top-k,
generate) is the correct product for many FAQ corpora. Add a router when
indexes or tools **branch**. Add a diagnostic agent when "no hits" should
mean something other than a shrug. Add action tools only when the answer
must *do* a thing, not just say a thing.

**Modular roles.** Triage, retrieve, answer; optionally a critic for
corrective retrieval. Each role has a tiny persona and a tiny tool list.
That is hub-and-spoke more often than a debate club.

**Optimize retrieval as if it were the product** (because it is). ANN,
metadata filters, domain embeddings, rerankers, **hybrid**. Isolated
indexes per tenant or product line — searching the wrong corpus is a
safety bug, not a quality nit. Ingest lifecycle (stale docs) will hurt
you more than the choice of chat model. The agent still needs a retrieval
tool that exposes filters.

**Grounding discipline.** Instructions: use the provided context or say
you cannot. Require citations or excerpts. A **grounding agent** as a
gate is cheaper than a scandal. Pair with a confidence gate on structured
workspace state if you already have one.

**Corrective retrieval.** Critic says "this context does not support the
draft" → reformulate, change filters, try keyword vs dense, *then*
answer again. Cap the correctives. Unbounded CRAG is groove lock with a
scholarly name.

**Eval the pipeline, not the vibes.** Hit rate, citation validity,
answer-from-context rate, p95 latency. Change one of: chunking, embedding,
retriever, prompt. Then measure. Changing all four is how you get a story
and no attribution.

## Blueprint for a deep research agent

Deep research is **open-ended gathering plus synthesis**: web or internal
stores, many sources, an outline that changes as evidence arrives.
Frontier vendors ship a consumer version. You build an internal one when
the sources are **yours** (wikis, tickets, warehouses) or the policy for
"what counts as a source" is yours.

This is the outer task-loop shape with field constraints: a planner that
owns plan and state, workers that return typed deltas, a critic, and
layered stop rules. Do not start here if a RAG system plus a critic
answers the question. Complexity is expensive.

```
  goal
    |
    v
  [ planner / brain ]   owns plan + state; never answers a claim
         |                without a worker result
         | delegates
         v
  [ workers, stateless ]  searcher | extractor | analyst | summarizer
         |                  tools only; no long memory
         v
  [ critic ]            coverage, contradictions, missing views
         |
         v
  [ synthesizer ]       report + gaps + sources
         |
         +-- stream outline and partials to the UI
```

**Two-tier orchestration.** A planner holds the long-horizon plan and
state (subtopics, status, notes). Workers are **tool-like**: one skill,
fresh context, result back. Giving every worker the full memoir is how
you pay for noise and get role collapse.

**Tool policy on the planner.** Facts come from retrieval and web (or
internal search) tools. Direct answers to empirical claims without a
source are forbidden in the contract. This is persona + tool_choice, not
a hope.

**Self-critique before freeze.** Coverage vs the plan, contradictions
across sources, missing stakeholder views. The critic should be allowed
to send the planner back into the loop. Termination still layers: hard
cap, budget, quality, stagnation.

**Streaming UX.** Users will not watch a silent ten-minute job. Stream
the outline as it grows, the latest finding, and "still open" subtopics.
The synthesizer at the end is a different agent on purpose: explorers
make a mess; editors make a report.

**When to refuse the shape.** Classification, a single-doc question, a
known FAQ — no loop. If you cannot name the **state object** and the
**stop rule**, you do not have research. You have a crawler on a credit
card.

Internal research often swaps Brave/web MCP for **your** document store
and ticket search. The blueprint does not change. The hands do.

## What you can ship from these tips

A support bot, a RAG graph, and a research loop are buildable from the
blueprints above. Write down the layer that owns each piece — persona,
tools, plan, memory, eval — before you add another server. When two
products copy session stores, keys, and trace configs, that copy is the
signal to pull those pieces into shared services. Agent vs flow, native
tools vs MCP, one-shot RAG vs a loop: decide them in the design review
and write the choice next to the blueprint.

## Check yourself

1. A bug: the bot is charming and refunds the wrong SKU. Name the
   **layer** you inspect first and the **second** layer you inspect if
   the first looks clean. Why are they not the same?
2. Rewrite a fluffy persona ("you are a helpful expert") as a contract:
   role, two fences, one I-don't-know action, one structured output.
3. You added six MCP servers in a week. Give two rules from the tools
   layer that would have rejected at least three of them.
4. When do you *turn reasoning down*? Give a task from your world where
   ReAct would be malpractice.
5. Session memory vs long-term vs RAG: which store holds "the user said
   their order id two turns ago," which holds "this user is on Plan
   Enterprise," and which holds "return policy v3.2"? What goes wrong if
   you merge all three into the system prompt?
6. What is the smallest eval you would ship with a support agent on
   Friday, and what would you add after the first week of HITL?
7. Sketch a support mutation (refund) as a tool policy: when is the tool
   invisible, when is it callable, when is a human required?
8. For RAG: when is a one-shot retrieve-and-generate enough, and what
   symptom tells you you need a router or a critic — not a bigger
   context window?
9. Deep research: why are workers stateless, and what happens to the
   planner if you let it answer a factual claim with no tool result?
10. Take the support blueprint and the research blueprint. Which pieces
    are safe to share (a tool schema, a rubric), and which pieces have
    to stay product-specific (persona fences, stop rules)?
