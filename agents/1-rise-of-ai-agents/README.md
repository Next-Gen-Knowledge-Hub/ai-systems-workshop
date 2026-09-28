# 1. The rise of AI agents

Companion notes for **Chapter 1** of *AI Agents in Action* (2nd edition,
Micheal Lanham; Manning, 2026).

This chapter is the map for the whole Agents track. By themselves, LLM apps
generate text. Agents perceive, decide, and act toward a goal — book the
flight, list the flights only as a step along the way. Skip this chapter and
you will do what thousands of demos have done: call a chat API, sprinkle a
tool or two, and then be surprised when the thing loops, spends, or cannot
explain which layer failed.

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

The one sentence to remember a year from now: an agent is software with
agency — it can choose tools and next steps without you clicking each one —
and that agency is engineered as five layers. A bigger prompt is one piece of
layer 1, nothing more.

Two consequences fall straight out of that diagram. First, ChatGPT as you use
it today is mostly an assistant; your production bot that files tickets
overnight is trying to be an agent. Second, adding function calling does not
by itself make an agent. Function calling without a loop, a memory policy, and
evaluation is still an assistant with extra JSON.

## Defining agents and agentic thinking

The word *agent* is older than LLMs. Reinforcement learning already meant
"entity that acts in an environment and gets a signal back." Product language
uses it more loosely: "something that does tasks for a user." For this
workshop, keep the engineering definition:

**An agent perceives its environment, decides what to do, and takes action to
achieve a goal.**

*Agentic* is the adjective for systems that actually run that loop with some
autonomy. Philosophical debates about "intention" can wait. If the system
cannot choose an action without a new human prompt per step, it is not
agentic enough to need the rest of this book.

### Agent, assistant, and LLM patterns

Teams often say "we shipped an agent" when they shipped a system prompt. Name
the pattern you actually run.

A **direct LLM** is you talking to the model. Early ChatGPT was this. Fine
for drafting. Useless for "update Jira and email the customer," because nothing
in the loop can change the world.

An **assistant** may call tools (search, images, code). A person still
ratifies the important steps, or the tool surface is tiny and reversible.
Assistants often write a better prompt for another model (image, search). That
is still one task at a time, with a human as the scheduler for the next goal.

An **agent** wraps the model in a loop that can chain tools, revise the plan
from observations, and stop when a termination condition fires. When you draw
your system, label the box honestly. The label decides which failure modes you
are on the hook for: a chatty assistant wastes tokens; a looping agent wastes
money and may mutate production.

### Sense-plan-act-learn

Agency, internally, is a four-beat loop. The book abbreviates it as
**SPAL**:

1. **Sense** — read the user goal, the current state, tool results, memory.
2. **Plan** — decompose the goal into tasks that map onto tools.
3. **Act** — call a tool (or produce a user-visible answer).
4. **Learn** — look at the observation: continue, replan, or stop.

A goal such as "travel to Calgary" is not one tool. It is search flights,
book flights, hotels, transport — each a tool, each an observation that
changes the plan. Tasks should be sized to tools. If a "task" cannot be
expressed as a tool call plus a check, it is still a wish.

Walk a goal you care about through those four beats until the mapping is
boring. Sense is the inputs. Plan is the decomposition. Act is each call.
Learn is where a sold-out flight forces a replan. That four-beat rhythm is
the inner core of every agentic loop you will build later; everything else in
this chapter is scaffolding around it.

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
unexpected payloads. Failure handling is part of the agent. A demo that only
shows the happy JSON path is not an agent you can page. When a booking API
returns 429, the learn beat has to decide whether to wait, switch carriers, or
stop and tell the user — and that decision lives in your loop, not only in the
HTTP client's retry policy.

You will implement tools two ways in this track: framework decorators that
build a schema from a Python function, and MCP servers that expose the same
kind of schema over a standard protocol. Same idea, different packaging. The
schema-and-observation contract stays identical either way.

## Introducing the Model Context Protocol

MCP (Anthropic, 2024) is an open JSON-RPC 2.0 convention so that any host
(Claude Desktop, your agent runtime, an IDE) can talk to any server that
exposes tools, resources, and prompts.

Before a shared protocol, every model vendor and every SaaS grew a private
plugin format. N agents times M tools is a combinatorial tax: every new host
pays M integrations, every new service pays N wrappers. MCP wraps the tool
once as a server; hosts speak one client. The tax becomes addition.

This chapter only introduces MCP so the five layers have a realistic socket.
A protocol solves discoverability and a shared call shape. It does not, by
itself, decide who may call a tool, where the API key lives, whether the call
was a good idea, or how teams version capabilities across an org. Those gaps
still need product and platform work; the socket just makes the capability
portable.

## Five functional layers

Capability is not "add more prompt." It is five layers you can add, thin, or
skip on purpose. They are not a waterfall. Reasoning consults memory while
tools run; evaluation can sit on every hop.

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

Core agents almost always need layers 1–3. Layers 4–5 are how you stop
hallucinating policy and shipping untested loops. If a production incident
cannot be tagged with a layer, you do not yet have this map in your bones.
"The bot is witty but refunds the wrong order" is a layer question: persona
may explain the wit; tools, reasoning, knowledge, or evaluation explain the
wrong refund — and you inspect the layer that owns the broken behavior first.

### Persona

The persona is the **system prompt as a product**: role (coder, support,
researcher), expertise, tone, operating constraints. It can be hand-written,
drafted by another model, or even searched (research has used evolutionary
tricks to mutate personas against a metric).

Everything you wish the agent "just knew" about *how to behave*, and that is
not a retrieved fact, belongs here. Facts about *your company* belong in
layer 4. Mixing them is how a prompt becomes a dumping ground and a
compliance nightmare: style and refusal rules drift next to SKUs and last
week's incident notes, and nobody can say what is policy versus what is data.

### Tools and actions

Tools are not only "book the flight." The book splits them by **what they do
to state**:

- **Context retrieval** — read-only: search, files, APIs. Ground the next
  thought. These do not write long-term memory by themselves.
- **Task completion** — change the world: send, book, mutate a ticket.
- **Knowledge/memory tools** — read and write the agent's own stores.
- **Reasoning/planning tools** — for example a sequential-thinking server that
  gives the agent an inspectable scratchpad.
- **Evaluation tools** — score, ground, critique an artifact.

If you cannot say which bucket a tool is in, you cannot write a guardrail for
it later. A read-only search tool and a send-mail tool may share a schema
shape; they do not share a blast radius. Classification is how you decide
which calls need human approval, budgets, or dry-run modes.

### Reasoning and planning

Frontier models already "think" in the forward pass. That is enough when the
horizon is short, the tool list is small, and a wrong action is cheap. You
add **structured** reasoning (CoT, ReAct, trees, Reflexion) when:

- the task is long-horizon or branching,
- many tools compete,
- errors are expensive or irreversible (money, email, prod data),
- you need an auditable trace,
- the domain is outside the model's comfort zone.

Model-native reasoning on a five-minute FAQ lookup is usually enough. The
same model booking a refund across three systems needs an explicit plan and
observations you can audit. Use the criteria above as a checklist against a
system you know: if several boxes light up, structured reasoning earns its
token cost; if none do, a sharper persona and fewer tools usually beat a
full ReAct scaffold.

### Knowledge and memory

Context is finite. Knowledge and memory are how you annotate the next prompt
with the right tokens instead of stuffing everything.

**Knowledge** is usually documents and indexes (RAG) — the agent's library.
**Memory** is usually interaction: this session (short-term, in the window)
and across sessions (long-term, retrieved).

Stores range from a list, to SQL/JSON, to graphs, to dense vectors. Hybrid
systems are normal. Conversational memory is the one you will ship first and
the one that will blow the token budget first.

Short-term memory lives in the context window. Long-term memory is retrieved
into that window when needed. Treating a PDF knowledge base as "memory" mixes
the library with the conversation: you either dump the whole PDF every turn
(token blow-up) or forget that knowledge needs a retrieval step with its own
failure modes. Keep the words separate even when the same vector store holds
both kinds of embedding.

### Evaluation and feedback

Two timescales share this layer.

**In the loop (learn):** after a tool result, decide whether the plan still
holds. This is SPAL's learn beat — continue, replan, or stop.

**Around the loop:** LLM-as-judge, rubrics, grounding ("did this claim appear
in retrieved docs?"), critic agents, traces in tools such as Phoenix.

Without this layer you cannot tell a prompt regression from a model
provider's bad Thursday. Evaluation is how you pin a persona change, a tool
schema tweak, or a sampling knob and see which one moved the score. Skip it
and every incident becomes folklore about "the model got worse."

## Multi-agent systems

A single agent hits walls: too many tools in one persona, no parallelism, a
context window that cannot hold the whole problem, or a domain that is
inherently multi-party (markets, debate, simulation).

Reasons to split, which are not the same reason:

| Reason | What you are buying |
|---|---|
| Specialization | A billing agent plus a tech agent beats one god-prompt |
| Parallelism | Ten companies researched concurrently |
| Context | Each agent holds a slice |
| Inherent multi-agent | The problem *is* several roles |

Pick a reason, then pick an assembly pattern. Mixing "we needed parallelism"
with "so we built a debating team" buys coordination cost you did not ask for.
Three assembly patterns cover most products. Pick one per product; mixing them
without a diagram is how handoffs go missing.

### Flow (assembly line)

Planner → researcher → writer, in order. Coordination options:

- **Shared thread** — everyone sees the full chat. Context explodes; later
  agents drown in early chatter.
- **Blackboard** — named slots for artifacts. More structure, more design.
- **Message passing** — explicit payloads between stages. Testable; easy to
  drop fields by accident.

Flows are easy to test stage-by-stage and brittle when the first stage's plan
is wrong. Shared thread fails by drowning the writer in research chatter.
Message passing fails by omitting a field the writer needed. Blackboard fails
when slots go stale after a replan. Those are the production stories under the
cute diagram.

### Orchestration (hub-and-spoke)

A hub talks to the user and treats specialists as tools. You keep one mouth.
The hub's context and latency become the bottleneck. This is the natural
upgrade when a single assistant's tool list got ridiculous and you still want
one conversation with the human. Summaries into the hub lie; structured worker
outputs keep control honest.

### Collaboration (teams)

Peers talk to each other (optionally with a manager or user-proxy). QA can
critique code while product checks requirements. You gain debate and lose a
single-threaded story. Traces get harder; you will want evaluation sooner,
because "who said what to whom" is no longer one span.

Specialization vs parallelism vs context slicing is one axis. Flow vs hub vs
team is another. A combination that is a mistake: you needed a shorter tool
list (specialization / context), and you built a peer debate team (team shape)
with a shared thread. You paid for coordination and context explosion when a
three-stage flow with typed messages would have been enough.

## Next steps in this book

The first half of Lanham is the five layers, with MCP and multi-agent
introduced early so later chapters are not toys. Then: deployment, the
three-layer agentic loop, cognition and metacognition, and field tips.

You can finish this track as an agent author without an org-wide platform
book. You will feel the hole when you try to share memory, keys, and eval
across two teams — that hole is session storage, registries, and shared
observability, which sit outside this chapter's map.

## Check yourself

1. A stakeholder says "our ChatGPT wrapper is an agent because it has web
   search." Which of the three patterns is it, and what extra loop would
   have to exist before you would agree?
2. Walk a goal you have actually automated (or wanted to) through SPAL.
   Name one tool per *act* beat. Where would *learn* change the plan?
3. Why does registering a JSON schema not by itself give you production
   tool use? Name two failure modes the runtime must own.
4. In one sentence, what problem does a shared tool protocol solve that a
   Python decorator on one app does not?
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
