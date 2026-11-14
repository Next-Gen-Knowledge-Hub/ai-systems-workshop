### **Chapter 10 Summary: The Cognitive Agent That Thinks, Monitors, and Adapts**

Chapter 10 of *AI Agents in Action, Second Edition* moves from isolated reasoning patterns to a **cognitive architecture** in which specialized modules share state, monitor progress, and adapt when evidence or confidence shifts.

---

#### **1. Cognition and Metacognition**

* **Five failure modes**: Production agents often emit **confident wrong answers**, repeat failed searches, cling to obsolete plans, hallucinate past missing data, or flatten compositional questions into one lookup; each maps to a structural module rather than a prompt fix.
* **Primitives versus architecture**: Chain-of-thought, ReAct, trees, and reflection are **skills**; architecture chooses which skill to deploy, watches whether it works, and reroutes when evaluation signals stagnation or contradiction.
* **Cognition defined**: Agent cognition is the fidelity of the **internal task model**—decomposition, dependency ordering, novel tool chaining, and mid-run plan revision when new facts contradict earlier assumptions.
* **Metacognition defined**: Metacognition covers **confidence that tracks correctness**, stagnation detection before iteration limits burn out, and behavior that changes when retrieval and memory no longer cover the question.
* **Theoretical foundations**: **Minsky** favors a society of narrow modules; **Baars** supplies a global workspace where signals broadcast; **Kahneman** motivates a fast path for trivial queries and a full cognitive cycle for ambiguous or contradictory work.

---

#### **2. Cognitive Architecture**

* **Cognitive workspace**: A typed shared state holds task representation, strategy, findings, confidence, execution history, and **attention signals** such as contradiction, stagnation, or knowledge gaps.
* **Perception and planning**: Perception classifies task type and **complexity** before any tool call; planning picks direct, decompose, explore, or hypothesis-test strategies and can lean on memory hits from prior cases.
* **Execution and evaluation**: Execution wraps tool output in scored findings; evaluation is **real-time metacognition** that judges evidence quality, consistency, progress, and whether to continue, replan, escalate, or stop.
* **Attention routing**: Deterministic routing implements **system 1 fast respond**, default perceive-plan-execute-evaluate cycles, meta-plan after stagnation, replan on contradiction, and escalate when confidence collapses.
* **Memory via MCP**: The official **MCP memory server** stores entities, relations, and observations; proactive retrieval before planning and experience recording after evaluation turn sessions into cumulative institutional knowledge.

---

#### **3. Building and Running the Cognitive Agent**

* **Cognitive loop**: The outer **agentic loop** still governs iteration and exit; inside each iteration, attention may run multiple cognitive steps until signals clear or the task completes.
* **Confidence gate**: Presentation to the user is gated on workspace confidence, unresolved contradictions, and **declining confidence trends**, favoring more gathering or explicit uncertainty over polished guesses.
* **Stagnation pivot**: Overlapping findings or flat confidence trends raise a stagnation signal, record the failed strategy, and route to **meta-planning** so the planner cannot replay the same approach.
* **Knowledge boundary**: Combined retrieval quality, memory coverage, and confidence classify whether the agent is **within, at the edge of, or outside** known territory, triggering low-confidence handling when coverage is thin.
* **Emergent behaviors**: Module interaction produces curiosity after knowledge gaps, adaptive persistence after failure, selective depth on simple familiar queries, and **graceful degradation** when budgets or confidence run out.

---

#### **4. Measuring Cognitive Capability**

* **Diagnostic tests**: Five practical probes mirror the production failure modes—metadata mistaken for content, repeated queries, plans that ignore new facts, hallucination without coverage, and flat single-step answers to compositional questions.
* **Efficiency and calibration**: Track **cognitive efficiency** (steps versus optimal), **metacognitive calibration** (confidence versus correctness), adaptation rate after stagnation, and knowledge-boundary accuracy for false certainty versus false caution.
* **Before-and-after studies**: Compare a flat ReAct agent with the cognitive stack on the same queries, then contrast an empty knowledge graph against a graph populated after many tasks to see memory compound.
* **Generalization path**: Swapping perception labels, execution MCP servers, and planning strategies reuses the same workspace, attention, evaluation, and gating patterns across domains without rescaling the base model alone.
