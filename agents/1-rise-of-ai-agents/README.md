# 1. The rise of AI agents

Companion notes for **Chapter 1** of *AI Agents in Action* (2nd edition, Micheal Lanham; Manning, 2026).

This chapter is the map. It establishes the definitions, the loop, and the building blocks of an agent. A chatbot generates text. An agent perceives a situation, plans, uses tools, and keeps going until a goal is done or a limit stops it. If you cannot name which pattern you shipped, you will be surprised when it loops, spends money, or cannot explain a wrong refund.

## 1. Defining agents vs assistants

**An agent perceives its environment, decides what to do, and takes action to achieve a goal.** *Agentic* means the system actually runs that loop with some autonomy. If every step needs a fresh human prompt, it is not agentic enough to need the rest of this track.

* **The autonomy spectrum**: A direct chat answers the question in front of it. A tool-using assistant still waits for a person to approve the next step, or it does one reversible call. An **agent** is handed a higher-level goal and chooses tools and next steps on its own. High-stakes actions can still require a human gate. Autonomy is about who schedules the steps, not about whether a human can ever say no.

Four interaction patterns sit on that spectrum. Name the one you actually run.

1. **Direct LLM chat**: Prompt in, tokens out. No tools and no loop. Drafting and Q&A live here. Useless for "update the ticket and email the customer," because nothing in the loop can change the world.
2. **Tool-augmented LLM**: One tool call per user prompt, such as a single image generation or a single search. The model emits one structured call. The person is still the scheduler for anything that comes next.
3. **Assistant**: Several tools are available, and a person approves each important step. Pair-programming assistants live here. The tool list can be rich. The goal is still one task at a time.
4. **Autonomous agent**: You approve the goal, not every step. The agent plans, picks tools, and executes multi-step work. A human gate stays on irreversible actions (send the email, issue the refund, change production).

Adding function calling does not by itself create pattern 4. Function calling without a loop, a stop condition, and a way to read the tool result is still an assistant with extra JSON.

## 2. The sense–plan–act–learn loop

Agency, inside the process, is a four-beat loop. The short name is **SPAL**.

* **Sense**: Read the goal, the user message, tool results, and anything already in memory. This beat is inputs, not a decision.
* **Plan**: Break the goal into tasks that a tool can do. A task that cannot be stated as a tool call plus a check is still a wish.
* **Act**: Call the tool, or produce an answer the user can see. The model does not execute the tool. It names the call. Your runtime runs it.
* **Learn**: Read the observation. Continue, revise the plan, or stop. A sold-out flight belongs here: the plan changes because the world answered.

A goal such as "travel to Calgary" is not one tool. Search flights, book a flight, book a hotel, arrange transport. Each result can change the next plan. Walk one real goal through the four beats until the mapping is boring. Everything later in this track is scaffolding around this rhythm.

## 3. Agents act with tools

A **tool** is a function the model can see as a schema: a name, a description, and typed arguments.

```
register the tool  -->  schema the model can read
        |
        v
model chooses a call  -->  your runtime executes  -->  observation back
```

* **Schema vs execution**: Registering JSON does not run anything. The runtime owns timeouts, HTTP 429s, unexpected payloads, and what gets fed back as the observation.
* **What the learn beat must decide**: On a 429 from a booking API, wait, switch carrier, or stop and tell the user. That choice lives in the loop. A retry inside the HTTP client is a different decision.
* **Two packagings, one contract**: A Python decorator that builds a schema from a function, and an MCP server that exposes the same kind of schema, are the same idea. The model still emits a call. Something you run still returns an observation.

Tools also differ by what they do to state. The schema can look the same. The blast radius does not.

1. **Context retrieval**: Read-only search, files, or APIs. They ground the next thought. They do not write long-term memory by themselves.
2. **Task completion**: Change the world. Send, book, mutate a ticket.
3. **Knowledge and memory tools**: Read or write the agent's own stores.
4. **Reasoning tools**: An inspectable scratchpad, such as a sequential-thinking server.
5. **Evaluation tools**: Score, ground, or critique an artifact.

If you cannot name the bucket, you cannot write the guardrail. A search tool and a send-mail tool do not share an approval policy.

## 4. Model Context Protocol (MCP)

**MCP** is an open protocol, introduced by Anthropic in 2024, so that any host (a desktop app, an IDE, your agent runtime) can talk to any server that exposes tools, resources, and prompts. People call it the **USB-C for LLMs**: one socket, many devices.

* **The problem it solves**: Before a shared protocol, every model vendor and every SaaS product grew a private plugin format. N hosts times M tools is a combinatorial tax. MCP wraps the tool once as a server. Hosts speak one client. The tax becomes addition.
* **What the socket standardizes**: Discovery of capabilities, the call shape, and a shared way to format responses. A new host does not re-integrate every tool. A new tool does not grow a wrapper per host.
* **What the socket does not decide**: Who may call the tool, where the API key lives, whether the call was a good idea, or how an organization versions the capability. Those stay product and platform work. The protocol makes the capability portable.

This chapter only introduces the socket so the layers below have a realistic place to plug in.

## 5. The five functional layers

Capability is not "add more prompt." It is five layers you can add, thin, or skip on purpose. They are not a waterfall. Reasoning consults memory while tools run. Evaluation can sit on every hop.

```
+--------------------------------------------------------------+
| 5  Evaluation and feedback   judges, rubrics, grounding      |
+--------------------------------------------------------------+
| 4  Knowledge and memory      docs, vectors, graphs, chat     |
+--------------------------------------------------------------+
| 3  Reasoning and planning    CoT, ReAct, explicit plans      |
+--------------------------------------------------------------+
| 2  Tools and actions         the only way to touch the world |
+--------------------------------------------------------------+
| 1  Persona                   role, constraints, style        |
+--------------------------------------------------------------+
```

Core agents almost always need layers 1–3. Layers 4 and 5 are how you stop inventing policy and shipping untested loops. If a production incident cannot be tagged with a layer, the map is not in use yet.

1. **Persona**: The system prompt as a product. Role, expertise, tone, and operating constraints. Hand-written, drafted by another model, or searched. How the agent should *behave*, when that is not a retrieved fact, belongs here. Company facts belong in layer 4. Mixing them makes a prompt that nobody can audit: style rules drift next to SKUs.
2. **Tools and actions**: The functions and endpoints that touch the world: APIs, databases, files. Use the five buckets in section 3. A witty persona does not explain a wrong refund. The tool that issued it might.
3. **Reasoning and planning**: How the agent breaks a task down and adapts. Chain-of-thought, ReAct, and tree-of-thought are the usual names. The model's own forward pass is enough when the horizon is short, the tool list is small, and a wrong action is cheap. Add an explicit method when the task is long or branching, many tools compete, the action is irreversible, you need an auditable trace, or the domain is outside the model's comfort zone. A five-minute FAQ lookup usually fails that test. A refund across three systems usually passes it.
4. **Knowledge and memory**: **Knowledge** is the library: documents and indexes, usually retrieved. **Memory** is the interaction: this session (short-term, inside the context window) and across sessions (long-term, retrieved back in). Stores range from a list, to SQL, to a graph, to vectors. Stuffing a PDF into every turn blows the token budget. Calling that PDF "memory" also hides the retrieval step and its own failures. Keep the words separate even when one vector store holds both kinds of embedding.
5. **Evaluation and feedback**: Two clocks. **In the loop**, the learn beat decides whether the plan still holds. **Around the loop**, judges, rubrics, grounding ("did this claim appear in the retrieved docs?"), critic agents, and traces score the run after the fact. Without the second clock you cannot tell a prompt regression from a bad day at the model provider. Skip it and every incident becomes folklore that "the model got worse."

"The bot is witty but refunds the wrong order" is a layer question. Persona may explain the wit. Tools, reasoning, knowledge, or evaluation explain the refund. Inspect the layer that owns the broken behavior first.

## 6. Multi-agent system patterns

A single agent hits a wall: too many tools in one persona, no parallelism, a context window that cannot hold the problem, or a job that is inherently several roles.

Split for a named reason, then pick an assembly pattern. The reason and the pattern are different axes.

| Reason | What you are buying |
|---|---|
| **Specialization** | A billing agent and a tech agent, instead of one god-prompt |
| **Parallelism** | Ten companies researched at once |
| **Context** | Each agent holds a slice of the problem |
| **The problem is multi-party** | Markets, debate, simulation |

Three assembly patterns cover most products. Pick one and draw it.

* **Agent flow (assembly line)**: Specialists run in order. Planner, then researcher, then writer. Easy to test stage by stage. Brittle when the first stage's plan is wrong. Three ways to pass work:
  1. **Shared thread**: Everyone sees the full chat. Later agents drown in early chatter.
  2. **Blackboard**: Named slots hold artifacts. More structure, more design. Slots go stale after a replan.
  3. **Message passing**: An explicit payload between stages. Testable. Easy to drop a field the next stage needed.
* **Orchestration (hub-and-spoke)**: One orchestrator talks to the user and treats specialists as tools. You keep one mouth. The hub's context and latency become the bottleneck. This is the natural upgrade when one assistant's tool list got ridiculous and you still want one conversation. Summaries into the hub lie. Structured worker outputs keep control honest.
* **Collaboration (peer-to-peer)**: Agents talk to each other, optionally with a manager. QA critiques code while product checks requirements. You gain debate and lose a single-threaded story. Traces get harder, so evaluation shows up sooner.

A combination that is a mistake: you needed a shorter tool list (specialization or context), and you built a peer debate team on a shared thread. You paid for coordination and a flooded context. A three-stage flow with typed messages would have matched the reason.

## Check yourself

1. A stakeholder says "our ChatGPT wrapper is an agent because it has web search." Which of the four patterns is it, and what extra loop would have to exist before you would call it an autonomous agent?
2. Walk a goal you have actually wanted to automate through sense, plan, act, and learn. Name one tool for each act. Where would learn change the plan?
3. Why does registering a JSON schema not give you production tool use? Name two failures the runtime must own.
4. A booking API returns HTTP 429. What three choices belong to the learn beat, and why is an HTTP-client retry a different decision?
5. In one sentence, what problem does a shared tool protocol solve that a Python decorator inside one app does not? Name one decision MCP still leaves to you.
6. The bot is witty and refunds the wrong order. Which layer explains the wit, which layers could explain the refund, and what would you inspect first?
7. When is the model's own reasoning enough, and when would you add an explicit method such as ReAct? Give one example of each from a system you know.
8. Which memory lives in the context window? What goes wrong if you treat a PDF knowledge base as memory and paste it into every turn?
9. Pick one multi-agent *reason* and one *pattern* (flow, hub, or team). Show a combination that would be a mistake, and say what you would have built instead.
10. In a three-stage flow, which handoff drowns the writer in research chatter, which drops a field the writer needed, and which goes stale after a replan? Why are the five layers not a pipeline you run once from top to bottom?
