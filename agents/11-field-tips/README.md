### **Chapter 11 Summary: Tips for Building Agentic Systems**

Chapter 11 of *AI Agents in Action, Second Edition* distills **production lessons** into layer-by-layer guidance and three reference workflows—support, retrieval-augmented answering, and deep research—so design choices stay testable, grounded, and measurable.

---

#### **1. Tips by the Five Layers**

* **Persona layer**: Treat instructions as an **API contract** with a crisp role, explicit exclusions, repeated guardrails, a structured “I don’t know” path, and runtime facts injected sparingly to avoid token bloat.
* **Tools and actions**: Ship **single-purpose tools** with clear schemas and docstrings, control when tools may run, cap exposed MCP surface area, and return structured errors so the model can recover.
* **Reasoning and planning**: Use ReAct or plan-then-execute for multistep work, cap turns, add a brief **self-review** before final output, and dial reasoning down for one-shot tasks.
* **Knowledge and memory**: Make **RAG a first-class tool**, separate session thread context from long-term vector memory, hybridize search, chunk semantically, ground answers strictly in retrieval, and partition corpora by role or tenant.
* **Evaluation and feedback**: Trace prompts, tools, latency, and outcomes; automate eval suites and guardrails; capture **human feedback**; and iterate with an AIOps loop across indexes and APIs the agent touches.

---

#### **2. Customer Support Agent**

* **Narrow charter**: Begin with bounded intents such as orders, returns, and status, expanding domains only after grounding and escalation paths prove reliable.
* **Grounding and safety**: Route answers through **RAG over policies and manuals**, add grounding checks before users see text, and verify identity before account or refund actions under least privilege.
* **Human escalation**: Expose an escalate tool for angry users, ambiguous policy, or high-impact refunds, and collect per-conversation feedback for later eval review.
* **Resilience and cost**: Apply retries, fallbacks, timeouts, caching of frequent policy answers, rate limits on external APIs, and **transparent traces** of searches, tools, and citations for audit.
* **Multi-agent workflow**: Production support is usually a chain—**triage, retrieval, grounding, scoped API action behind a guardrail, final answer**—not a single monolithic bot.

---

#### **3. RAG Agent System**

* **Orchestrate only when needed**: A one-shot retrieve-and-answer path is enough for many cases; add agents when routing, critique, or post-retrieval actions justify the overhead.
* **Modular roles**: Split **router or triage, retriever, answerer**, and optionally a critic that triggers corrective retrieval when passages fail to support the question.
* **Retrieval quality**: Tune ANN indexes, metadata filters, hybrid search, reranking, and embeddings; log recall pain points and cases that should have ended in “I don’t know.”
* **Grounding discipline**: Instructions must **use only retrieved context**, require citations or excerpts, and prefer honest uncertainty over confident wrong answers.
* **Refinement loops**: When ranking shows weak coverage, a refinement agent may rewrite the query and search again behind a **bounded retry counter** to prevent infinite retrieval cycles.

---

#### **4. Deep Research Agent System**

* **Two-tier orchestration**: A **research planner** breaks goals and delegates to stateless workers—web search, extraction, analysis, writing—while keeping long memory out of tool-like hands.
* **Tool policy**: Force factual claims through retrieval or web tools; forbid unsourced assertions in the planner’s instructions so synthesis stays tied to evidence.
* **Critique before delivery**: A critic pass checks coverage, bias, and contradictions; insufficient material returns control to the planner rather than shipping a thin brief.
* **Streaming experience**: Stream partial findings or status updates so users perceive progress and can approve checkpoints on long runs.
* **Budgets and cache**: Deduplicate repeated queries, cap tokens and external calls, and **refresh caches** on a schedule so speed does not trade away freshness.
