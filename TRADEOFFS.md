# Trade-offs cheat sheet

Every agent and platform decision buys you something and charges you
something else. This page is the one-screen version of choices the two books
spend whole chapters on. Agents track = *AI Agents in Action* 2e. Platform
track = *Designing AI Systems* MEAP.

Nothing here is a rule you should apply without measuring. That is itself
the first production lesson.

When a row lists both tracks, read **both** folders; do not fuse them into
one design note. Details: [`INDEX.md`](./INDEX.md).

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
