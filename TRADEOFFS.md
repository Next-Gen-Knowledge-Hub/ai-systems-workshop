# Trade-offs cheat sheet

Every agent, platform, and classical-ML decision buys you something and
charges you something else. This page is the one-screen version of choices
the three books spend whole chapters on. Agents track = *AI Agents in
Action* 2e. Platform track = *Designing AI Systems* MEAP. ML track =
*Hands-On Machine Learning* 2e.

Nothing here is a rule you should apply without measuring. That is itself
the first production lesson.

When a row lists more than one track, read **each** folder; do not fuse
them into one design note. Details: [`INDEX.md`](./INDEX.md).

---

## The method, before any of the tables

1. **Name the loop you are in.** Inner sense–plan–act–learn, a task loop
   (research until done), or a meta loop (change the strategy). Mixing them
   is how agents thrash. → [A9](./agents/9-agentic-loop/)
2. **The model call is not the product.** Treat provider APIs as ~2% of the
   system; design the other 98% on purpose. → [P1](./platform/1-why-a-platform/)
3. **Measure quality and cost on the same trace.** Latency without tokens,
   or tokens without a score, will optimize the wrong thing.
   → [P7](./platform/7-observability/), [A7](./agents/7-evaluation-and-feedback/)
4. **Change one thing, then measure again.** Prompt, model, retrieval, and
   tools moved in the same deploy give you one data point and no attribution.
   On the ML track the same rule is "do not peek at the test set, then
   retune." → [M2](./ml/2-end-to-end-project/)

---

## Agent vs assistant vs raw LLM (A1)

| Shape | You gain | You pay |
|---|---|---|
| Raw LLM chat | Simplest loop; great for drafting | No tools, no memory policy, no agency |
| Assistant (tools with a human in the loop) | Safer irreversible actions | Throughput; the user is the scheduler |
| Agent (plans and acts across steps) | Multi-step goals without a click per tool | Runaway loops, cost, harder eval |

## Five layers: what to add when (A1, A11)

| Layer | Add it when | Skip or thin it when |
|---|---|---|
| Persona | You need a stable role, tone, constraints | Throwaway scripts |
| Tools / actions | The model must change the world or fetch live state | Pure generation |
| Reasoning / planning | Long horizon, many tools, expensive mistakes | One-shot Q&A with a strong model |
| Knowledge / memory | Private docs or multi-session users | Public facts the model already knows |
| Evaluation / feedback | You will ship and iterate | A demo you will throw away |

---

## Multi-agent shapes (A1, A4) vs workflows (P8)

| Choice | You gain | You pay |
|---|---|---|
| Single agent, many tools | Simple ops, one context | Tool overload, worse specialization |
| Flow (assembly line) | Clear stages, easy to test each hop | Rigid; a bad plan early poisons the line |
| Hub-and-spoke | One mouth to the user; workers as tools | Hub is a bottleneck and a single context hog |
| Collaboration (peers) | Debate, critique, parallelism | Coordination cost, loops, harder traces |
| Platform workflow (decorated service) | Deploy, scale, sync/stream/async as HTTP | You are now operating a fleet, not a notebook |

**See also, do not merge:** [A4](./agents/4-multi-agent-systems/) is control
and handoffs inside the agent graph. [P8](./platform/8-workflow-service/) is
how that graph becomes a container with health checks.

---

## MCP vs native tools

| Choice | You gain | You pay |
|---|---|---|
| Native SDK tools (decorators) | Fast to write; tight types | Every app reimplements Slack, GitHub, filesystem |
| MCP server (agent-local) | Swap hosts; reuse community servers | Process lifecycle, transports, another failure domain |
| MCP *on the platform* (tool service) | N+M integrations, credentials and policy in one place | Platform must own registry, isolation, what MCP does not cover |

Agents angle: [A3](./agents/3-mcp/). Platform angle: [P6](./platform/6-tools-and-guardrails/).

### MCP transports (A3)

| Transport | Fits | Hurts |
|---|---|---|
| STDIO | Local desktop / one agent process | Does not survive across machines |
| SSE / HTTP | Remote servers, multiple clients | More moving parts; auth and timeouts |

---

## Reasoning patterns (A5)

| Pattern | You gain | You pay |
|---|---|---|
| Model-native reasoning | Cheap, short horizon | Weak audit trail; long tasks drift |
| Chain-of-thought | Better multi-step arithmetic/logic | Tokens; leaking chain if you show it |
| ReAct | Grounds plans in tool observations | Extra round trips; noisy traces |
| Tree-of-thought | Explores alternatives | Cost explodes with branching |
| Reflexion | Learns from a failed attempt in-session | Needs a decent evaluator; can reinforce a bad critique |
| Sequential-thinking MCP | Externalized working memory for the plan | Another server; still not a substitute for tools |

---

## Memory and context (A6, P4)

| Choice | You gain | You pay |
|---|---|---|
| Stuff the whole transcript in context | Perfect recall of this session | Hits the window; cost scales with chatter |
| Truncate oldest turns | Simple | Loses the constraint the user stated in turn 1 |
| Summarize older turns | Bounded tokens | Summary error becomes "false memory" |
| Hierarchical memory (recent raw + older summaries) | Balance | Two stores to keep honest |
| Retrieval-augmented memory | Pulls only relevant past | Retrieval miss = the agent "forgets" on purpose |
| Graph / episodic / procedural stores | Structure beyond a chat log | Design and hygiene (forgetting) work |

Agents angle: forms of memory and MCP. Platform angle: Session Service and
token-budget algorithms. Knowledge that is **documents**, not **chat**, is
[P5](./platform/5-data-service/) / [A6 RAG](./agents/6-memory-and-rag/).

---

## RAG (A6, P5)

| Choice | You gain | You pay |
|---|---|---|
| Long context, no retrieval | Simple | Cost, noise, stale private facts still missing |
| Vector search only | Semantic matches | Exact identifiers (SKUs, error codes) miss |
| Keyword only | Exact tokens | Paraphrase misses |
| Hybrid + RRF | Best of both on mixed corpora | Tuning; two indexes |
| Isolated indexes per team/tenant | Safety and independent knobs | Ops; you must not search the wrong index |
| Agentic RAG (retrieve as a tool, maybe more than once) | Can reformulate the query | Latency and runaway retrieval loops |

---

## Guardrails (A4, P6, A8)

| Layer | You gain | You pay |
|---|---|---|
| Prompt-only "please be safe" | Free | Users will jailbreak it |
| Input policy (PII, injection, allow-listed intents) | Stops bad requests early | False positives block real work |
| Output policy (toxicity, leakage, schema) | Last line before the user | Extra latency; still not a proof of safety |
| Behavioral policy (which tools, which args) | Stops *actions*, not just text | Needs a real tool graph and credentials model |
| Agent-as-guardrail / grounding critic | Flexible, rubric-aware | Cost; the judge can be wrong |
| Sandbox + egress control | Limits blast radius of tools | Engineering; some tools need the network |

---

## Models: routing, cache, resilience (P3, A8)

| Choice | You gain | You pay |
|---|---|---|
| One frontier model for everything | Quality ceiling | Bill; latency |
| Cost-aware routing (easy → small model) | Spend | Mis-route a hard task; eval must catch it |
| Feature-based routing (vision / JSON mode) | Use the model that can actually do it | Catalog of capabilities to maintain |
| Fallback chain | Survive an outage | Silent quality drop if you do not score outputs |
| Response cache | Repeat prompts become cheap | Stale answers; privacy if keys include user data |
| Rate limits | Protect budget and upstream quotas | Timeouts your UX must explain |

---

## APIs: sync, stream, async (P2, P8, A8)

| Mode | You gain | You pay |
|---|---|---|
| Synchronous HTTP | Simple clients | User waits; timeouts on long agents |
| Streaming tokens | Perceived speed | Harder error handling mid-stream |
| Async job + poll | Long research, batch | Job store, progress, exactly-once-ish semantics |
| Event-driven / queue | Fan-out, decoupling | At-least-once, idempotency (A8) |
| Edge runtime | Low latency near the user | Tooling and secret constraints |

---

## Evaluation (A7, P7)

| Choice | You gain | You pay |
|---|---|---|
| No eval, ship the demo | Speed | You cannot tell if the next prompt helped |
| TDAD (tests around traces and tools) | Regressions get caught in CI | Tests are stubs unless you pin models or mock tools |
| Offline dataset + scores | Compare variants before prod | Dataset rot; not the real traffic |
| Online scoring of production | Reality | Noise; need sampling and cost of the judge |
| Human annotation queues | Calibrates automatic scores | Slow, expensive |
| A/B on the platform | Causal claims | Traffic split, experiment hygiene |

---

## Build vs buy vs platform (P1)

| Choice | You gain | You pay |
|---|---|---|
| Script + vendor API | Demo this week | Sprawl by month six |
| Framework only (chains, indexes) | Fast composition | You still own sessions, keys, eval, cost |
| Managed cloud AI suite | Ops off your plate | Lock-in, gaps vs your compliance story |
| Internal platform (this book's shape) | One way to do memory, RAG, tools, traces | You are now a platform team |

The books do not require you to rewrite Bedrock. They require you to **see
the services** even if a vendor implements them.

---

## Train a model vs call a model vs wrap a loop (M1, A1, P3)

These three are the most common collision in this workshop. They share
vocabulary and do not share a stack.

| Job | You gain | You pay |
|---|---|---|
| Fit sklearn / Keras weights on *your* table or pixels (ML track) | A model that knows *your* distribution | Labels, leakage discipline, retraining, serving the graph |
| Call a provider LLM (Platform Model Service / Agents runtime) | No training job; frontier quality | Tokens, routing, eval of generations, you do not own the weights |
| Run an agent loop around that call (Agents track) | Multi-step tools and plans | Runaway cost, tool blast radius, traces |

**See also, do not merge:** [M1](./ml/1-ml-landscape/) is "the machine
changes when the data changes." [A1](./agents/1-rise-of-ai-agents/) is
"the runtime chooses the next tool." [P3](./platform/3-model-service/) is
"one adapter so every workflow does not grow its own SDK."

## Batch vs online learning (M1) vs session memory (P4)

| Choice | You gain | You pay |
|---|---|---|
| Batch training (full dataset, then freeze) | Simple; reproducible snapshots | Stale the day the distribution moves |
| Online / incremental learning | Adapts as examples arrive | Catastrophic forgetting; harder eval; poisoning |
| Session memory (chat transcript) | The *conversation* is available to the next LLM call | Not a trained weight update; hits the context window |

Online learning in Géron is **updating parameters**. Session Service is
**storing tokens**. Do not call a Redis chat log "online learning."

## Hold-out metrics vs judges vs platform scores (M2–M3, A7, P7)

| Choice | You gain | You pay |
|---|---|---|
| Held-out test set + CV (sklearn) | Unbiased estimate if the split is honest | One number; silent if production drift |
| Precision/recall/ROC on labels | You can pick an operating point | Needs labels; accuracy on imbalanced data will lie |
| LLM-as-judge / critic agent | Scores generations that have no gold label | The judge can be wrong; cost |
| Platform traces + A/B | Causal claims on live traffic | Experiment hygiene; not a confusion matrix |

A confusion matrix does not tell you whether the *agent* refunded the
wrong order. A Phoenix trace does not tell you whether the *classifier*
is calibrated. Read both folders if you have both jobs.

## Embeddings as features vs embeddings as retrieval (M13, M16, A6, P5)

| Choice | You gain | You pay |
|---|---|---|
| Embedding table inside the net (Géron) | The task loss trains the vectors | Tied to that model; not a document index |
| Frozen pretrained word vectors (2019-shaped) | Transfer on small text sets | Domain mismatch; no retrieval loop |
| Retrieval embeddings + vector index (Agents/Platform) | Private docs enter the prompt | Chunking, index isolation, miss/latency |

Same word. The ML folder trains (or loads) a lookup. The Data Service
indexes *chunks* so an agent can retrieve them. Do not store Géron's
`Embedding` layer in pgvector and call it RAG.

## Attention you train vs an LLM you sample (M16, A2)

| Choice | You gain | You pay |
|---|---|---|
| Train a Transformer (or even a char-RNN) | You own the task and the weights | Data, GPUs, 2019 NLP is not 2026 chat |
| Prompt a hosted LLM | No training loop | You do not own the weights; persona and tools are the lever |

M16 is how attention *works*. A2 is how you *steer* a model that already
has it. Keep the notes in their folders.

## RL agent vs LLM agent (M18, A1, A9)

| Choice | You gain | You pay |
|---|---|---|
| RL (reward, MDP, Q-learning, policy gradient) | Behaviour that maximises a scalar you defined | Credit assignment, exploration, sample hunger |
| LLM agent (SPAL, tools, termination gate) | Language, tools, plans without a gym | Not a Bellman update; eval is traces and judges |

Gym + DQN is not MCP + ReAct. The word "agent" is the overlap. The
algorithms are not.

## Feature scaling and pipelines (M2) vs ingestion (P5)

| Choice | You gain | You pay |
|---|---|---|
| sklearn `Pipeline` (impute → encode → scale → estimate) | Fit only on train; no leakage | Tabular; not a PDF corpus |
| Data Service ingestion (extract → chunk → embed → index) | Documents become searchable | Different grain; different failure (miss, not MSE) |

## Serving weights vs a Model Service vs deploying an agent (M19, P3, A8)

| Choice | You gain | You pay |
|---|---|---|
| TensorFlow Serving / a frozen SavedModel | Low-latency *your* net | You operate GPUs and versions of *that* graph |
| Platform Model Service | One gateway to many *providers* | You still do not train those providers |
| Agent deploy (API, Compose, edge, events) | The *loop* is the product | Tools, secrets, prompt injection, budgets |

TF Serving is how Géron ships chapter-14 weights. The Model Service is
how the platform ships "call whoever." Agent deploy is how Lanham ships
the loop. Pick the folder that matches the artifact you are shipping.
