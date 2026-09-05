# 1. The rise of AI agents

Companion notes for **Chapter 1** of *AI Agents in Action* (2nd edition,
Micheal Lanham; Manning, 2026).

This chapter is the map for the whole Agents track. By themselves, LLM apps
generate text. Agents **perceive, decide, and act** toward a goal — book the
flight, not list the flights. Skip this chapter and you will do what thousands
of demos have done: call a chat API, sprinkle a tool or two, and then be
surprised when the thing loops, spends, or cannot explain *which layer*
failed.

The Platform track is a different book. If you need "why every team rebuilt
session storage," that is [platform ch. 1](../../platform/1-why-a-platform/).
This folder stays on **what an agent is**.

## The mental model

Three interaction patterns sit on one spectrum. Mixing their names is how
design reviews go nowhere.

```
  user prompt
       |
       v
  [ RAW LLM ]     prompt in, tokens out. No tools. No loop.
       |
       |  + tool calls, usually with a human still in the loop
       v
  [ ASSISTANT ]   can search / code / draw, often one task at a time
       |
       |  + a goal, a plan, multiple steps, learning from tool results
       v
  [ AGENT ]       sense -> plan -> act -> learn, until the goal is done
                  (or a budget / guardrail stops it)
```

The one sentence to remember a year from now: **an agent is software with
agency** — it can choose tools and next steps without you clicking each one —
and that agency is engineered as **five layers**, not as a bigger prompt.

Two consequences fall straight out of that diagram. First, ChatGPT-as-you-
use-it-today is mostly an *assistant*; your production bot that files tickets
overnight is trying to be an *agent*. Second, "we added function calling"
does not make an agent. Function calling without a loop, memory policy, and
evaluation is still an assistant with extra JSON.

## Defining agents and agentic thinking

The word *agent* is older than LLMs. Reinforcement learning already meant
"entity that acts in an environment and gets a signal back." Product language
uses it more loosely: "something that does tasks for a user." For this
workshop, keep the engineering definition:

**An agent perceives its environment, decides what to do, and takes action to
achieve a goal.**

*Agentic* is the adjective for systems that actually run that loop with some
autonomy. Philosophical debates about "intention" can wait. If it cannot
choose an action without a new human prompt per step, it is not agentic
enough to need the rest of this book.

### Agent, assistant, and LLM patterns

**Problem** — Teams say "we shipped an agent" when they shipped a system
prompt.

**Solution** — Name the pattern you actually run.

- **Direct LLM:** you talk to the model. Early ChatGPT was this. Fine for
  drafting. Useless for "update Jira and email the customer."
- **Assistant:** the model may call tools (search, images, code). A person
  still ratifies the important steps, or the tool surface is tiny and
  reversible.
- **Agent:** the model is wrapped in a loop that can chain tools, revise the
  plan from observations, and stop when a termination condition fires.

Assistants often *write a better prompt for another model* (image, search).
That is still not a multi-step goal loop. When you draw your system, label
the box honestly.

### Sense-plan-act-learn

Agency, internally, is a four-beat loop. The book abbreviates it as
**SPAL**:

1. **Sense** — read the user goal, the current state, tool results, memory.
2. **Plan** — decompose the goal into tasks that map onto tools.
3. **Act** — call a tool (or produce a user-visible answer).
4. **Learn** — look at the observation: continue, replan, or stop.

A goal such as "travel to Calgary" is not one tool. It is search flights,
book flights, hotels, transport — each a tool, each an observation that
changes the plan. **Tasks should be sized to tools.** If a "task" cannot be
expressed as a tool call plus a check, it is still a wish.

The inner SPAL loop is **layer 1** of the agentic loop in
[ch. 9](../9-agentic-loop/). Do not skip ahead until this four-beat is
boring.

### Agents act with tools

A tool is a function with a **schema** the model can see: name, description,
typed arguments. The model does not execute Python. It emits a structured
call; *your* runtime runs the function and feeds the result back.

```
  register tool  -->  JSON definition in the prompt / API
       |
       v
  model chooses a call  -->  runtime executes  -->  observation back
```

Tools wrap APIs, databases, files, browsers. They fail: timeouts, 429s,
unexpected payloads. **Failure handling is part of the agent**, not an
afterthought in the HTTP client. A demo that only shows the happy JSON path
is not an agent you can page.

You will implement tools two ways in this track: framework decorators
([ch. 2](../2-llms-prompting-agents/)) and **MCP servers**
([ch. 3](../3-mcp/)). Same idea, different packaging.

## Introducing the Model Context Protocol

MCP (Anthropic, 2024) is an open JSON-RPC 2.0 convention so that **any host**
(Claude Desktop, your agent runtime, an IDE) can talk to **any server** that
exposes tools, resources, and prompts.

**Problem** — Every model vendor and every SaaS grew a private plugin format.
N agents times M tools is a combinatorial tax.

**Solution** — Wrap the tool once as an MCP server. Hosts speak one protocol.

This chapter only *introduces* MCP so the five layers have a realistic socket.
The Agents-track deep dive is [ch. 3](../3-mcp/). How a **platform** uses MCP
as a shared integration bus — registry, credentials, what the protocol does
*not* cover — is [platform ch. 6](../../platform/6-tools-and-guardrails/).
Do not merge those two chapters.

## Five functional layers

Capability is not "add more prompt." It is five layers you can add, thin, or
skip on purpose. They are **not** a waterfall. Reasoning consults memory
while tools run; evaluation can sit on every hop.

```
  +--------------------------------------------------------------+
  | 5  Evaluation and feedback   (judges, rubrics, grounding)    |
  +--------------------------------------------------------------+
  | 4  Knowledge and memory      (docs, vectors, graphs, chat)   |
  +--------------------------------------------------------------+
  | 3  Reasoning and planning    (CoT, ReAct, explicit plans)    |
  +--------------------------------------------------------------+
  | 2  Tools and actions         (the only way to touch the world)|
  +--------------------------------------------------------------+
  | 1  Persona                   (role, constraints, style)      |
  +--------------------------------------------------------------+
```

Core agents almost always need 1–3. Layers 4–5 are how you stop hallucinating
policy and shipping untested loops. [Chapter 11](../11-field-tips/) is tips
**by these layers**. If a production incident cannot be tagged with a layer,
you do not yet have this map in your bones.

### Persona

The persona is the **system prompt as a product**: role (coder, support,
researcher), expertise, tone, operating constraints. It can be hand-written,
drafted by another model, or even searched (research has used evolutionary
tricks to mutate personas against a metric).

Everything you wish the agent "just knew" about *how to behave* and that is
not a retrieved fact belongs here. Facts about *your company* belong in
layer 4. Mixing them is how a prompt becomes a dumping ground and a
compliance nightmare.

### Tools and actions

Tools are not only "book the flight." The book splits them by **what they do
to state**:

- **Context retrieval** — read-only: search, files, APIs. Ground the next
  thought. Do not confuse with long-term memory writes.
- **Task completion** — change the world: send, book, mutate a ticket.
- **Knowledge/memory tools** — read and write the agent's own stores.
- **Reasoning/planning tools** — e.g. a sequential-thinking server
  ([ch. 5](../5-reasoning-and-planning/)).
- **Evaluation tools** — score, ground, critique ([ch. 7](../7-evaluation-and-feedback/)).

If you cannot say which bucket a tool is in, you cannot write a guardrail for
it later.

### Reasoning and planning

Frontier models already "think" in the forward pass. That is enough when the
horizon is short, the tool list is small, and a wrong action is cheap. You
add **structured** reasoning (CoT, ReAct, trees, Reflexion) when:

- the task is long-horizon or branching,
- many tools compete,
- errors are expensive or irreversible (money, email, prod data),
- you need an auditable trace,
- the domain is outside the model's comfort zone.

Details and when-to-choose: [ch. 5](../5-reasoning-and-planning/).

### Knowledge and memory

Context is finite. Knowledge/memory is how you **annotate the next prompt
with the right tokens** instead of stuffing everything.

- **Knowledge** — usually documents and indexes (RAG). The agent's "library."
- **Memory** — usually interaction: this session (short-term, in the window)
  and across sessions (long-term, retrieved).

Stores range from a list, to SQL/JSON, to graphs, to dense vectors. Hybrid
systems are normal. Conversational memory is the one you will ship first and
the one that will blow the token budget first.

How an *agent* wires RAG and MCP memory: [ch. 6](../6-memory-and-rag/). How
a *platform* persists sessions and org indexes:
[platform ch. 4](../../platform/4-session-service/) and
[platform ch. 5](../../platform/5-data-service/). Same words, different
jobs — stay in this folder for the agent-shaped version.

### Evaluation and feedback

Two timescales:

- **In the loop (learn):** after a tool result, decide whether the plan still
  holds. This is SPAL's learn beat.
- **Around the loop:** LLM-as-judge, rubrics, grounding ("did this claim
  appear in retrieved docs?"), critic agents, traces in Phoenix.

Without this layer you cannot tell a prompt regression from a model
provider's bad Thursday. Deep dive: [ch. 7](../7-evaluation-and-feedback/).
Platform-shaped eval and A/B:
[platform ch. 7](../../platform/7-observability/).

## Multi-agent systems

A single agent hits walls: too many tools in one persona, no parallelism, a
context window that cannot hold the whole problem, or a domain that is
*inherently* multi-party (markets, debate, simulation).

Reasons to split, which are not the same reason:

| Reason | What you are buying |
|---|---|
| Specialization | A billing agent plus a tech agent beats one god-prompt |
| Parallelism | Ten companies researched concurrently |
| Context | Each agent holds a slice |
| Inherent multi-agent | The problem *is* several roles |

Three assembly patterns. Pick one per product; mixing them without a diagram
is how handoffs go missing.

### Flow (assembly line)

Planner → researcher → writer, in order. Coordination options:

- **Shared thread** — everyone sees the full chat. Context explodes; later
  agents drown in early chatter.
- **Blackboard** — named slots for artifacts. More structure, more design.
- **Message passing** — explicit payloads between stages. Testable, easy to
  drop fields by accident.

Flows are easy to test stage-by-stage and brittle when the first stage's plan
is wrong. More in [ch. 4](../4-multi-agent-systems/).

### Orchestration (hub-and-spoke)

A hub talks to the user and treats specialists as tools. You keep **one
mouth**. The hub's context and latency become the bottleneck. This is the
natural upgrade when a single assistant's tool list got ridiculous.

### Collaboration (teams)

Peers talk to each other (optionally with a manager/user-proxy). QA can
critique code while product checks requirements. You gain debate and lose
a single-threaded story. Traces get harder; you will want
[ch. 7](../7-evaluation-and-feedback/) sooner.

Platform workflows that *host* these graphs as HTTP services:
[platform ch. 8](../../platform/8-workflow-service/). Mention only.

## Next steps in this book

The first half of Lanham is the five layers, with MCP and multi-agent
introduced early so later chapters are not toys. Then: deployment, the
three-layer agentic loop, cognition/metacognition, and field tips.

You do not need the Platform book to finish this track. You will *feel* the
hole when you try to share memory, keys, and eval across two teams — that
hole is [platform ch. 1](../../platform/1-why-a-platform/).

## Check yourself

1. A stakeholder says "our ChatGPT wrapper is an agent because it has web
   search." Which of the three patterns is it, and what extra loop would
   have to exist before you would agree?
2. Walk a goal you have actually automated (or wanted to) through SPAL.
   Name one tool per *act* beat. Where would *learn* change the plan?
3. Why does registering a JSON schema not by itself give you production
   tool use? Name two failure modes the runtime must own.
4. MCP is introduced here and taught in chapter 3. In one sentence, what
   problem does a *protocol* solve that a Python decorator does not?
5. Map a bug to a layer: the bot is witty but refunds the wrong order.
   Persona, tools, reasoning, knowledge, or evaluation — and what would
   you inspect first?
6. When is model-native reasoning enough, and when would you add ReAct
   anyway? Steal the criteria from this chapter, then give a counterexample
   from a system you know.
7. Short-term vs long-term memory: which one lives in the context window,
   and what goes wrong if you treat a PDF knowledge base as "memory"?
8. Specialization vs parallelism vs context slicing: pick one multi-agent
   *reason* and one *pattern* (flow / hub / team). They are not the same
   axis — show a combination that would be a mistake.
9. Shared thread vs blackboard vs messages in a three-stage flow: which
   one fails by drowning the writer in research chatter, and which fails
   by dropping a field the writer needed?
10. Why are the five layers not a top-to-bottom pipeline you run once?

Continue to [LLMs, prompting, and agents](../2-llms-prompting-agents/).
