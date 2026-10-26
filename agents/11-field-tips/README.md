# 11. Field tips

Companion notes for **Chapter 11** of *AI Agents in Action* (2nd edition, Micheal Lanham; Manning, 2026).

These notes are the checklists that survive a real queue. Hang every tip on the same five layers: persona, tools, reasoning, knowledge and memory, evaluation. Optimize one layer, then integrate it. A witty persona will not save a blunt tool, and a perfect index will not save a loop with no stop. Support, RAG, and deep research are how you wire this stack for one job. If an incident cannot be tagged to a layer, the map is not in use yet.

## 1. Persona

```
+--------------------------------------------------------------+
| 5  Evaluation / feedback    traces, judges, humans, gates    |
+--------------------------------------------------------------+
| 4  Knowledge / memory       retrieval tool, session vs long  |
+--------------------------------------------------------------+
| 3  Reasoning / planning     ReAct, plan-then-do, caps        |
+--------------------------------------------------------------+
| 2  Tools / actions          one job each, schemas, failures  |
+--------------------------------------------------------------+
| 1  Persona                  contract, scope, I-don't-know    |
+--------------------------------------------------------------+
```

Treat the instructions as an **API contract**. Tools, memory, and plans inherit whatever you leave vague. Keep a short role, hard boundaries, and an explicit off-ramp. Debug from this shelf upward: did the instructions ask for the right job, do the hands work, is the thought the right size, are the stores honest, and can you prove the run?

Two failures show up in the transcript before any tool is wrong.

1. **Buried rule**: The lethal constraint sits in the middle of a prompt that reads like a novel. Recency drift hides it under the latest tool result.
2. **Missing no**: Nobody named the jobs that are out of scope, so the bot takes them.

* **Role and fences**: State what the agent does, what it refuses, and who it defers to. Repeat the lethal rules at the **top and the bottom**. Models drift across a long tool trace. "Never refund without lookup" has to still be in view after that trace, or the model will refund from the conversation alone.
* **I-don't-know**: Make "I don't know" a named action with three exits: ask, retrieve, or escalate. Completing a policy from parametric memory skips that action. A fluent answer about a return window the store never returned is this failure.
* **Narrow charter**: One job beats a "digital employee." A specialized agent is easier to evaluate and cheaper to route. Add a domain later, on purpose, after the first job has a score. A bot that also writes marketing copy will invent a coupon while it is "helping."
* **Structured outputs**: When another program or agent consumes the result, require JSON that matches a schema (Pydantic or a dataclass). That removes a class of parse bugs and the token padding models add to look helpful. Save prose for a human reader.
* **Dynamic instructions**: Inject facts that rot — the date, the tenant, the product version, the user locale — with a function at run time. A hardcoded Tuesday promo sits in a prompt you will not reopen, and it will still be Tuesday in June. Injecting *everything* on every turn is another dumping ground. Pass the facts this turn needs.

A constraint that is a **fact about the company** belongs in knowledge you retrieve. A constraint that is **how to behave** stays in the persona. Mixing them puts the legal policy in a system prompt nobody owns, next to the tone rules, where neither can be updated alone.

## 2. Tools and actions

Tools are the hands and the senses. A sloppy schema makes a smart model look drunk. A good schema makes a smaller model look competent. Prefer one responsibility, typed parameters, and failure as part of the contract.

Four failures cover most bad tool lists.

1. **God-tool**: A function named `do_anything`. The model calls it as a fidget because the description never says when to refuse.
2. **Mushy docstring**: The name and the arguments do not tell the model what the call is for, so it guesses.
3. **Missing timeout**: The call hangs, and the agent fills the silence with a success story.
4. **Starred MCP server**: The server was added because a blog post starred it. Nobody read the package or asked what filesystem it can reach.

* **Single-responsibility tools**: The name, the description, and the arguments are the model's user manual. If you cannot say when *not* to call the tool, the model will call it as a fidget. Split "look up the order" from "refund the order." One description cannot tell the truth about both.
* **Function tools for your code**: Prefer function tools for your own code and APIs. A decorator that builds the schema from the types and the docstring stays tied to the function you test. A hand-written JSON blob drifts, and the model then calls a parameter the function no longer accepts.
* **When tools run**: Force a tool when the turn is worthless without live data. Forbid tools when the turn is a rewrite of text you already have. Stop after the first sufficient call when one API result is the answer. Put that rule in the persona and in the tool settings. A sentence in the prompt, with no runtime knob, does not control the call.
* **Hosted tools and MCP**: Use a hosted or prebuilt tool, including MCP, for code, the web, and files when someone actually maintains the server. Due diligence is reading the package, checking who publishes it, and threat-modeling what it can reach. A random `npx` package is a supply chain you just gave your filesystem.
* **Failure as a payload**: Plan a timeout, retries with a budget, and an error payload the model can read: the status, what was tried, and what to do next. A silent exception is how an agent invents success. It will tell the customer the refund went through because the tool returned nothing and the persona is helpful.

Bucket every tool by what it does to the world: retrieve, mutate, memory, plan, evaluate. A guardrail attaches to the bucket. A search tool and a refund tool do not share an approval policy just because both are "tools."

## 3. Reasoning and planning

Frontier models already think inside the forward pass. Add structure when the horizon is long, when several tools compete, or when a wrong act is expensive. Match the primitive to the difficulty, cap the loop, and glance back.

Four mismatches show up constantly.

1. **ReAct on a FAQ**: The answer is one retrieval. A thought-act loop adds hops and no new evidence.
2. **A tree on order status**: "What's my order status?" is a lookup, not a branching search.
3. **No cap**: The loop runs until a person kills the tab.
4. **No last look**: The agent sends the email, or the refund, with nobody reading the draft.

* **ReAct or an honest chain-of-thought**: Use ReAct, or an honest chain-of-thought, for nontrivial work. The rhythm is thought, act, observation, then the next thought. A single chain-of-thought with a tool call stuffed into the same pass is still one pass. It cannot change its mind when the observation contradicts the thought.
* **Plan-then-execute**: For a big goal, write a short checklist into structured state, or hand the checklist to a sequential-thinking server. That beats improvising twenty hops in the transcript. The plan is data. You can print it, diff it, and see which step is stuck.
* **Iteration cap**: Put a hard cap on iterations. When the cap hits, return a useful failure: "I hit the budget; here is what I have; here is the missing slot." That is a termination story. The failure mode you are avoiding is a tab that spins until someone kills the process.
* **Self-review before final**: On anything irreversible, or anything the user will read at length, run a cheap second pass before you send. It catches "we cited the table of contents" and "we emailed the wrong tenant." Skip that pass on one-shot classification, where the second look costs more than the mistake.
* **Turn reasoning down**: When the task is one shot and the model is strong, turn the extra reasoning off. Paying for a chain on "format this JSON" grows the bill while the output stays flat. Order status, after the lookup has already returned, is the same kind of job.

Caps and a last look still apply on a single ReAct agent. A separate step that picks the reasoning primitive for you does not make the cap optional, and it does not make the last look optional.

## 4. Knowledge and memory

Retrieval is how knowledge and memory stay honest. Stuffing a PDF into the prompt is a demo that dies on the second document.

Four traps account for most of the damage.

1. **Docs in the system prompt**: The policy is pasted into every turn, so the window fills up and nobody can tell a miss from a hit.
2. **Transcript as the wiki**: The session is treated as the corporate record. A one-time exception in chat becomes the rule next week.
3. **One giant index**: Every tenant and every product line shares a vector index. A query for product A returns product B's policy.
4. **Default embedding**: The model was whatever the tutorial installed. SKUs, languages, and cost were never chosen.

* **Retrieval as a tool**: Give the model a retrieval tool over a store with metadata filters: product, version, locale, tenant. The model has to *call* it. Pre-stuffing the paragraph skips that call, so you cannot tell a miss from a hit, and you cannot filter the tenant.
* **Session versus long-term**: The thread is short-term working memory. "The user said their order id two turns ago" lives there. Facts that should survive a new chat — "this user prefers email," "this account is on Plan Enterprise" — live in a store you retrieve on purpose. Dumping the full graph into every prompt spends the window on facts this turn does not need.
* **Prune and shard**: Use an approximate-nearest-neighbor index (HNSW or IVF), shard by domain, and filter. Scale shows up as a retrieval problem before it shows up as a model problem. One index for every product line is how a query for product A returns product B's policy.
* **Hybrid search**: Dense similarity misses SKUs and error codes, because those strings are not "similar," they are exact. Keyword search (BM25) misses paraphrase. Graph or hierarchical search earns its keep on "related to this entity." Combine them. A single method will be wrong on a corpus that has both error codes and prose.
* **Embeddings on purpose**: Choose the dimensions, the quality, and a multilingual or domain-tuned variant. Price the cost per million tokens at ingest and again at query. Changing embedding families later is a re-index of the corpus, not a config flip you do on a Friday.

Org-wide indexes and session token budgets can live in shared platform services. The agent still chooses when to retrieve. A platform index does not fire itself.

## 5. Evaluation and feedback

You will not improve what you do not measure. Day one is logs, a tiny eval set, and a guardrail that can say no. A finished Phoenix deployment can wait.

The usual starting state has three holes.

1. **Demo energy, no traces**: The run looked good on stage. Nothing was logged, so Tuesday's failure has no prompt, no tool arguments, and no outcome.
2. **Edits with no score**: The prompt changed, the model changed, or the index changed, and nobody ran a set that could move a number.
3. **Slack as the dataset**: Humans yell, and the yell has nowhere to land. By Monday the label is gone.

* **Traces for 2 a.m.**: Log the prompt, the tool arguments, the tool results, the token counts, the latency, and the outcome. SDK tracing and Phoenix exist so you are not grepping stdout at 2 a.m. Ship those traces to the observability stack you already run. A trace that lives only on a laptop does not help the person on call.
* **Automated evals**: Keep a curated set and score accuracy, grounding, safety, and resolution. Run it when the prompt changes, when the model changes, and when the index changes. An LLM-as-judge is a tool. Pin the judge's model version, and spot-check its scores with a human. An unpinned judge moves the goalposts between runs.
* **Human in the loop**: Thumbs, a reason, and a queue beat a feedback link buried in the footer. On a support agent, the human who took the escalation is the label. Write that label down next to the trace, or the yell in Slack evaporates by Monday.
* **Guardrails and moderation**: This layer is schema validation, a content policy, and a grounding check: did this claim appear in a retrieved span? Put gates inside the loop, on workspace confidence, on the schema, or on a critic. Put judges around the loop for the runs you sample later. The prompt sentence "be safe" does not do any of that work.

Org-wide scores, shared datasets, and A/B experiments are a platform concern. This layer is the eval loop on the agent you are shipping. Stay here until that loop exists.

## 6. A customer support agent

Support is the shape most teams ship first and regret first. It talks to angry humans, it can mutate tickets, and "almost right" is a compliance event. An internal IT helpdesk is the same shape with a different persona. The wiring is the five layers, aimed at one queue:

```
persona    role + policy fences
tools      lookup, ticket APIs, escalate
reason     triage, then act or hand off
knowledge  manuals, via retrieval
eval       logs + the human who caught the escalation
```

| Layer | In this product |
|---|---|
| Persona | Support role, tone, policy fences, escalation manners |
| Tools | Order lookup, subscription, refund *or* "create ticket" — never both mutations as one god-tool |
| Reasoning | Triage for intent, risk, and missing slots, then act or hand off |
| Knowledge | Retrieval over manuals, policies, and versioned SKUs |
| Evaluation | Traces, resolution and CSAT, human review of escalations |

* **Narrow charter**: Start with orders, returns, and status. Add billing next. Add device troubleshooting after that. A bot whose charter also includes marketing copy will hallucinate a coupon. A light charter is how you keep latency and the eval set small enough to read.
* **Ground policy and facts**: For any answer that states a policy or a fact, retrieve first, then cite or quote the span. When the store is empty, ask or escalate. The default failure on this product is a coverage bluff: fluent invention on an empty retrieval.
* **Safe actions**: Read-only lookups can run eagerly. A mutation — refund, cancel, a change to personal data — needs confirmation, an identity check, and often a human. Hide the refund tool until triage says the intent is a refund *and* the order lookup succeeded. The model should not see a tool it is not allowed to call on this turn.
* **Triage is the plan**: Classify before you act.

  1. Can the docs answer this?
  2. Does it need an API?
  3. Is this rage, legal, or safety?
  4. Is a slot missing, such as the order id?

  A missing slot is a question, not a guess you fill from a previous ticket. A high-risk intent skips the happy path and goes to a human with a packet: what was tried, and what was retrieved.
* **Small memory**: Session memory is the ticket so far. Long-term memory is "this user prefers email" and "this account is on plan X," fetched for this turn. Memorizing every past chat as if it were policy poisons the next ticket. Someone's one-time exception becomes the new rule.
* **Eval that matches the job**: Track resolution rate, escalation rate, grounding failures, and wrong-order refunds. A wrong-order refund is a tools incident and an evaluation incident. It is not a tone incident. Sample traces every week. A witty persona attached to a wrong refund is a failed refund.

Design the handoff payload with the same care as the greeting. A support agent that cannot escalate cleanly is stuck holding a ticket it should have given away. Autonomy here includes the ability to stop.

## 7. Design patterns for a RAG agent

Retrieval-augmented generation is usually the infrastructure under another agent: support, research, or internal Q&A. It is rarely a product whose only box says "RAG." Retrieval is the engine. The agentic part is routing, critique, and the decision to retrieve again. Two overbuilds show up in design reviews. One wraps an orchestrator around a one-shot retrieve that already worked. The other is a single mega-agent that retrieves, answers, critiques, and files a Jira. Add an agent only for a job that one retrieve-and-generate cannot do, and keep that agent small.

```
query
  |
  v
[ triage / router ]  -- which index, which rewrite, skip retrieval?
  |
  v
[ retriever ]        -- hybrid + filters + optional rerank
  |
  v
[ answerer ]         -- use only the context; citations required
  |
  +---- optional [ critic / CRAG ]
            low grounding --> rewrite the query or change shard
            else           --> release
```

* **Agents only when needed**: Add a role only when one retrieve-and-generate cannot do the job.

  1. **One-shot**: Embed the query, take top-k, generate. That is the right product for many FAQ corpora.
  2. **Router**: Add one when indexes or tools branch, so a billing question does not search the HR corpus.
  3. **Diagnostic agent**: Add one when "no hits" should mean something other than a shrug, such as a bad filter or the wrong product line.
  4. **Action tool**: Add one only when the answer must do a thing, not only say a thing.
* **Modular roles**: Split triage, retrieve, and answer. Add a critic when you want corrective retrieval. Each role gets a tiny persona and a tiny tool list. That assembly is a hub with workers more often than a debate among peers. A debate club on a shared thread will drown the answerer in the router's scratchpad.
* **Retrieval is the product**: Treat the index as the product. Approximate nearest neighbors, metadata filters, domain embeddings, a reranker, and hybrid search each earn a place. Isolate indexes per tenant or per product line. Searching the wrong corpus is a safety bug, not a quality nit, because the answer can be fluent and forbidden. Stale documents on ingest will hurt you more than the choice of chat model. The retrieval tool still has to expose the filters, or the model cannot ask for "version 3.2, this tenant."
* **Grounding discipline**: Instruct the answerer to use the provided context or to say it cannot answer. Require a citation or an excerpt. A grounding check used as a gate is cheaper than a scandal. If you already keep structured workspace state, pair that gate with a confidence check on the state. Low confidence goes back to retrieval or to a human, not out to the user.
* **Corrective retrieval**: When the critic says the context does not support the draft, reformulate the query, change the filters, try keyword against dense, and only then answer again. Cap how many times the critic may send the query back. Unbounded CRAG is groove lock: the loop keeps rewriting under a scholarly name and never releases an answer.
* **Eval the pipeline**: Measure hit rate, citation validity, the rate of answers that actually come from the context, and p95 latency. Change one of chunking, the embedding, the retriever, or the prompt. Then measure. Changing all four in one experiment produces a story and no attribution. You will not know which edit moved the number.

## 8. Blueprint for a deep research agent

Deep research is open-ended gathering plus synthesis. The sources may be the web or your own stores. There are many of them. The outline changes as evidence arrives. Frontier vendors ship a consumer version of this. Build an internal one when the sources are yours — wikis, tickets, warehouses — or when the policy for what counts as a source is yours. The shape is an outer task loop. A planner owns the plan and the state. Workers return typed deltas. A critic reads the pile. Stop rules are layered, not a single "are we done?" If a retrieval system plus a critic already answers the question, this blueprint is the expensive option. Leave it for that question.

```
goal
  |
  v
[ planner ]            owns plan + state; a claim needs a worker result
       | delegates
       v
[ workers, stateless ] searcher | extractor | analyst | summarizer
       |               tools only; fresh context; no long memory
       v
[ critic ]             coverage, contradictions, missing views
       |
       v
[ synthesizer ]        report + gaps + sources
       |
       +-- stream the outline and partials to the UI
```

* **Two-tier orchestration**: The planner holds the long-horizon plan and the state: subtopics, status, and notes. Workers are tool-like. One skill, a fresh context, a typed result back. Giving every worker the full memoir pays for noise, and the roles collapse into one voice that both searches and concludes. The planner never answers an empirical claim itself.
* **Tool policy on the planner**: Facts come from a retrieval tool and from web or internal search. The contract forbids a direct answer to an empirical claim that has no source. Enforce that in the persona and with `tool_choice`, so the runtime refuses the unsourced answer instead of hoping the model behaves.
* **Self-critique before freeze**: Before the report freezes, check three things: coverage against the plan, contradictions across sources, and missing stakeholder views. The critic is allowed to send the planner back into the loop. Four stop rules still stack on top of that permission.

  1. **Hard cap**: A maximum number of iterations.
  2. **Budget**: A spend or token ceiling.
  3. **Quality bar**: A score the critic is willing to freeze.
  4. **Stagnation**: The notes stopped changing, so another hop will not help.
* **Streaming**: People will not watch a silent ten-minute job. Stream the outline as it grows, the latest finding, and the subtopics that are still open. The synthesizer at the end is a different agent on purpose. Explorers make a mess of notes. The editor writes the report, with gaps and sources still attached.
* **Refuse the shape**: Three jobs do not need this loop.

  1. Classification.
  2. A question about a single document.
  3. A known FAQ.

  If you cannot name the state object — the subtopics, their status, the notes — and you cannot name the stop rule, you do not have research. You have a crawler spending money.
* **Swap the hands**: Internal research often replaces a Brave or other web MCP server with your document store and your ticket search. The planner, the stateless workers, the critic, the synthesizer, and the stop rules stay. The hands change. The blueprint does not.

## Check yourself

1. The bot is charming and refunds the wrong SKU. Which layer do you inspect first, which layer do you inspect if that one is clean, and why are those two layers different jobs?
2. Rewrite "you are a helpful expert" as a contract: a role, two fences, one I-don't-know action, and one structured output. Which of those four is a company fact that belongs in retrieval instead?
3. You added six MCP servers in a week. Which two rules from the tools layer would have rejected at least three of them, and what would you have asked about the package before installing it?
4. Name a task where you would turn reasoning down, and a task where ReAct is the wrong primitive. What sentence should the agent return when it hits the iteration cap?
5. Which store holds "the user said their order id two turns ago," which holds "this user is on Plan Enterprise," and which holds "return policy v3.2"? What breaks if all three are pasted into the system prompt?
6. What is the smallest evaluation you would ship with a support agent on Friday? What would you add after the first week of labels from the humans who took escalations?
7. For a refund tool, when is the tool invisible, when is it callable, and when is a human required? What two facts go in the handoff packet?
8. When is one-shot retrieve-and-generate enough, and what symptom tells you to add a router or a critic? Why is changing chunking, embedding, retriever, and prompt in one experiment a bad measurement?
9. Why are research workers stateless? What happens if the planner answers a factual claim with no tool result? Name the four stop rules that still apply after the critic sends the planner back.
10. A support bot and a research loop both need a retrieval tool. Which pieces are safe to share (a tool schema, a rubric), and which stay product-specific (persona fences, stop rules)?
