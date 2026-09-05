# AI Systems Workshop

A workshop for learning **how to build agents and how to platform them**:
the difference between a chatbot that answers and an agent that acts, the five
layers that make an agent more than a prompt, and the production services
(model, session, data, tools, guardrails, observability, workflows) that stop
every team from reinventing the same 98% of the stack.

We are guiding this workshop with two books. They are kept on **separate
tracks**. Where a topic appears in both, the notes do not merge the chapters —
they **mention** the other track and send you there.

| Track | Folder | Book |
|---|---|---|
| **Agents** | [`agents/`](./agents/) | Micheal Lanham, *AI Agents in Action*, 2nd edition (Manning, 2026). Subtitle: *Intelligent workflows with LLMs, MCP, A2A, and more*. |
| **Platform** | [`platform/`](./platform/) | Suhas Suresha and Dewang Sultania, *Designing AI Systems* (Manning MEAP, 9 chapters). Subtitle: *A guide to production-ready platforms*. |

Each numbered folder is one chapter of that book: the topics from the chapter,
rewritten as a human-friendly companion you can read after (or alongside) the
book.

The notes here are **original study material**. They do not reproduce the
books' text. Read the chapter, then use the matching folder to lock in the
vocabulary, the trade-offs, the failure mode, and one production example.
Keep the PDFs next to your other books; they are not committed in this git
repo.

The topic index for both books lives in [`INDEX.md`](./INDEX.md). Cross-cutting
choices (agent vs flow, MCP vs native tools, truncation vs RAG memory, sync vs
stream) live in [`TRADEOFFS.md`](./TRADEOFFS.md).

## What this workshop assumes

You can write Python and you have called an LLM API once. You do not need
prior agent-framework experience, MCP, RAG internals, or platform design —
that is what the folders are for. Where a chapter needs a concept from an
earlier one, it says so. Where the *other book* covers the same idea from a
different angle, you get a one-line **See also** and a link, not a rewrite.

## How the two books divide the work

Lanham's book is **how an agent is put together**: persona, tools, MCP,
multi-agent patterns, reasoning, memory/RAG, evaluation, deployment, the
agentic loop, and cognition. You finish it able to *build* an agent.

Suresha and Sultania's book is **how an organization stops rebuilding that
agent's plumbing**: a Model Service, Session Service, Data Service, Tool and
Guardrails services, observability and experiments, and a Workflow Service.
You finish it able to *platform* agents so the next team does not copy-paste
the same Redis session store.

Read the Agents track first if you have never built an agent. Read the
Platform track first if you already ship LLM features and keep hitting cost,
memory, and sprawl. Either order works; the index is the map.

## This repository contains the following topics

### Track A — Agents (*AI Agents in Action*, 2e)

**Part A1 — What an agent is**

1. [The rise of AI agents](./agents/1-rise-of-ai-agents/) — AIA ch. 1
    - Agent vs assistant vs raw LLM; sense–plan–act–learn
    - MCP as the standard tool socket; five functional layers
    - Multi-agent shapes: flow, hub-and-spoke, collaboration
2. [LLMs, prompting, and agents](./agents/2-llms-prompting-agents/) — AIA ch. 2
    - Tokens, temperature, top-p; persona as the system prompt
    - A minimal OpenAI Agents SDK agent; tools and tracing
3. [Actions with MCP](./agents/3-mcp/) — AIA ch. 3
    - Clients, servers, tools/resources/prompts; STDIO vs SSE
    - Using and building MCP servers for agents
    - **See also:** [platform tools & MCP](./platform/6-tools-and-guardrails/)

**Part A2 — Multi-agent, reasoning, memory**

4. [Multi-agent systems](./agents/4-multi-agent-systems/) — AIA ch. 4
    - Control patterns, shared memory vs messages vs MCP
    - Flows, handoffs, input/output guardrails
    - **See also:** [platform workflows](./platform/8-workflow-service/)
5. [Reasoning and planning](./agents/5-reasoning-and-planning/) — AIA ch. 5
    - Chain-of-thought, ReAct, tree-of-thought, Reflexion
    - Sequential thinking as an MCP server
6. [Memory and RAG](./agents/6-memory-and-rag/) — AIA ch. 6
    - Embeddings, vector search, hybrid RAG agents
    - Graph/hybrid memory over MCP; compression and forgetting
    - **See also:** [session service](./platform/4-session-service/), [data service](./platform/5-data-service/)

**Part A3 — Hardening and shipping**

7. [Evaluation and feedback](./agents/7-evaluation-and-feedback/) — AIA ch. 7
    - Test-driven agent development; grounding and critic agents
    - Phoenix traces, evaluators, annotations
    - **See also:** [platform observability](./platform/7-observability/)
8. [Deploying agents](./agents/8-deploying-agents/) — AIA ch. 8
    - Voice, API, Docker Compose; edge vs API vs event-driven
    - Reliability, cost routing, threat model, prompt injection
    - **See also:** [platform SDK/API](./platform/2-sdk-and-api/), [model service](./platform/3-model-service/)
9. [The agentic loop](./agents/9-agentic-loop/) — AIA ch. 9
    - Inner SPAL loop, task loop, meta loop
    - Deep research agent; orchestration and collaboration loops
10. [Cognitive agents](./agents/10-cognitive-agents/) — AIA ch. 10
    - Cognition and metacognition as engineering, not metaphor
    - Workspace, attention, confidence gates, stagnation
11. [Field tips](./agents/11-field-tips/) — AIA ch. 11
    - Tips by the five layers; support, RAG, and research blueprints

Appendices: [sample code setup](./agents/appendix-a-sample-code/) (AIA A),
[Node.js for local MCP](./agents/appendix-b-nodejs-mcp/) (AIA B).

### Track B — Platform (*Designing AI Systems*, MEAP)

**Part B1 — Why a platform, and how you talk to it**

1. [Why your AI projects need a platform](./platform/1-why-a-platform/) — DAS ch. 1
    - Prototype vs production; AI sprawl; the model is ~2% of the system
    - Platform blueprint: the services the rest of the book builds
2. [SDK and API design](./platform/2-sdk-and-api/) — DAS ch. 2
    - Developer experience contract; code-to-container
    - Sync / stream / async APIs; gateway and SDK
    - **See also:** [deploying agents](./agents/8-deploying-agents/)

**Part B2 — The core services**

3. [The Model Service](./platform/3-model-service/) — DAS ch. 3
    - Provider adapters, streaming, retries, fallbacks, routing, cache
    - **See also:** [LLMs and prompting](./agents/2-llms-prompting-agents/)
4. [The Session Service](./platform/4-session-service/) — DAS ch. 4
    - Conversation history, storage backends, token-budget strategies
    - **See also:** [agent memory](./agents/6-memory-and-rag/)
5. [The Data Service](./platform/5-data-service/) — DAS ch. 5
    - Indexes, ingestion, embeddings, hybrid search, RRF
    - **See also:** [agent RAG](./agents/6-memory-and-rag/)
6. [Tools and guardrails](./platform/6-tools-and-guardrails/) — DAS ch. 6
    - Tool registry, credentials, MCP on the platform, execution policies
    - **See also:** [agent MCP](./agents/3-mcp/), [agent guardrails](./agents/4-multi-agent-systems/)

**Part B3 — Seeing, shipping, assembling**

7. [Observability and experimentation](./platform/7-observability/) — DAS ch. 7
    - Traces, scores, cost; offline/online eval; A/B
    - **See also:** [agent evaluation](./agents/7-evaluation-and-feedback/)
8. [The Workflow Service](./platform/8-workflow-service/) — DAS ch. 8
    - Decorated functions as HTTP; jobs, health, composition, deploy
    - **See also:** [multi-agent flows](./agents/4-multi-agent-systems/), [agentic loop](./agents/9-agentic-loop/)
9. [Building an AI assistant](./platform/9-building-an-assistant/) — DAS ch. 9
    - Putting every service into one assistant (memory, RAG, tools, safety)
    - **See also:** [field tips](./agents/11-field-tips/)

Cross-cutting: [topic index](./INDEX.md) · [trade-offs cheat sheet](./TRADEOFFS.md).

## How to use this workshop

1. Read the book chapter. The book is the source of truth.
2. Read the matching folder here. Headings follow the book's topics, so you
   can move between the two without losing your place.
3. When a **See also** points at the other track, follow it only if you need
   that angle (how to *code* the agent vs how to *serve* it). Do not merge
   the two chapters in your notes.
4. Answer the **Check yourself** questions in your own words. A good answer
   has three parts: the takeaway, the failure mode it prevents, and one
   example from a system you have actually worked on.

Work through each track in order the first time. MCP (Agents 3) without the
five layers (Agents 1) is a protocol with nowhere to plug in. The Model
Service (Platform 3) without "why a platform" (Platform 1) looks like
needless abstraction. The assistant chapter (Platform 9) assumes every
service in 3–8 exists.

## Try it locally

You do not need the full platform to learn the notes. For the Agents track,
the book's samples live at
[cxbxmxcx/AI-Agent-Workflows](https://github.com/cxbxmxcx/AI-Agent-Workflows)
and need Python, an OpenAI (or compatible) API key, and Node.js/`npx` for
local MCP servers — see [appendix A](./agents/appendix-a-sample-code/) and
[appendix B](./agents/appendix-b-nodejs-mcp/).

For the Platform track, think in services, not notebooks: one process per
workflow, a gateway in front, Postgres (or similar) for sessions and
pgvector for indexes. A scratch stack that matches the mental model:

```bash
docker run --name aisys-pg \
  -e POSTGRES_PASSWORD=secret \
  -p 5432:5432 -d pgvector/pgvector:pg16
```

Use that for session rows and hybrid search labs. Point model calls at
whatever provider you already have; the Model Service chapter is about the
*adapter*, not a particular vendor.

Feel free to use and make any change ;)
