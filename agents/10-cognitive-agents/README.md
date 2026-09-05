# 10. Cognitive agents

Companion notes for **Chapter 10** of *AI Agents in Action* (2nd edition,
Micheal Lanham; Manning, 2026).

You already have reasoning primitives ([ch. 5](../5-reasoning-and-planning/)),
a three-layer loop ([ch. 9](../9-agentic-loop/)), and memory
([ch. 6](../6-memory-and-rag/)). Stacking those pieces does not produce a
mind. It produces a toolbox with no one deciding *which* instrument, *when*
to put it down, and *whether* the work is even going well. This chapter is
that missing craftsman — **cognition and metacognition as code**. Skip it
and you will ship an agent that looks smart in a demo and then answers from
a table of contents, retries the same search until the budget dies, or
invents a procedure because retrieval came back empty.

The Platform track does not replace this chapter. If you need shared
sessions, indexes, and judges as *services*, that is
[platform ch. 1](../../platform/1-why-a-platform/) onward. This folder stays
on **how one agent thinks about its own thinking**.

## The mental model

Reasoning patterns are skills. A cognitive architecture is a **shop floor**
that picks a skill, watches the work, and reroutes when the work stalls.

```
  user query
       |
       v
  +----------------------------------------------------------+
  |                 COGNITIVE WORKSPACE                       |
  |  task model · hypotheses · findings · confidence         |
  |  history of tries · attention flags                      |
  +----------------------------------------------------------+
       ^         ^         ^         ^         ^         ^
       |         |         |         |         |         |
  PERCEIVE   REMEMBER   PLAN    EXECUTE   EVALUATE   ATTEND
  (what is   (have we   (which   (do the   (is this   (who
   this?)     seen it?)  skill?)  step)     working?)  next?)
```

The one sentence to remember: **cognition is the quality of the agent's
internal model of the task; metacognition is the quality of the agent's
model of that model.** Architecture owns both. A bigger prompt owns
neither.

Two consequences. First, CoT / ReAct / ToT / Reflexion remain useful — they
are the **primitives** [ch. 5](../5-reasoning-and-planning/) taught. This
chapter is the **dispatcher** that chooses among them. Second, the
[ch. 9](../9-agentic-loop/) loops still run. Cognition sits *inside* an
iteration: L2 still decides "keep going or stop"; attention decides "which
module, right now."

## Cognition and metacognition as engineering

**Problem** — Teams treat "smarter" as "more patterns in the system
prompt."

**Solution** — Name a failure, map it to a missing *module*, then add that
module. Do not add another paragraph of "think carefully."

We are not arguing about machine consciousness. We are defining
**observable, measurable behaviors** you can log, gate, and regress-test.

### Five failure modes of capable-but-not-cognitive agents

These are the default of agents that can reason but cannot *monitor*
reasoning. Run ten diverse tickets through your current bot and tag the
logs. If you cannot tag them, you do not yet have this map.

```
  observed mess          missing skill              module that owns it
  ----------------       ----------------------     -------------------
  glossy miss            evidence quality           evaluation
  groove lock            progress sensing           attention + eval
  plan rigor mortis      strategy revision          planning + attention
  coverage bluff         knowledge-edge sensing     evaluation + memory
  tool salad             compositional use          planning + execution
```

**Glossy miss (confident wrong).** Retrieval returns a landing page, a
README heading, or an API table of contents. The agent quotes it as the
fix. The user hears certainty. The deficit is not "it didn't search." It
searched and **did not judge the evidence**. Evaluation that scores
relevance and refuses to treat metadata as content is the patch.

**Groove lock (broken record).** The same query, the same tool, the same
near-duplicate finding, until `max_iterations`. The deficit is progress
sensing. Without a signal that "this step did not move the workspace,"
the L2 loop is just a `while True` with extra tokens.

**Plan rigor mortis (rigid plan).** Perception classified the ticket as
"timeouts"; planning committed to "raise timeouts"; new evidence says the
timeouts were already raised. A non-cognitive agent finishes the original
checklist anyway. The deficit is **permission to throw the plan away**.

**Coverage bluff (overcommitted guess).** Tools and memory both miss. The
model's parametric knowledge fills the hole with a fluent procedure. The
deficit is knowing the **edge of coverage** and having a structural off-ramp
that is not the word "maybe" in a paragraph.

**Tool salad (shallow composition).** The agent has search, a filesystem,
and a calculator, and uses each once, never as a pipeline (search → open
the file the search named → compute from the file). The deficit is
**dependency-aware composition**, not missing tools.

None of these is fixed by "be thorough." Each one is a **module you can
point at in a trace**. If a production incident cannot be tagged this way,
you are still debugging vibes.

### From reasoning primitives to cognitive architecture

[Chapter 5](../5-reasoning-and-planning/) taught you when *you* pick CoT,
ReAct, a tree, Reflexion, or a sequential-thinking MCP server. The
developer still made that choice at design time. A factual lookup does not
need a thought tree. An ambiguous, contradictory ticket should not be a
single CoT pass. **None of those patterns know they are the wrong
pattern.**

```
  design-time pick (ch. 5)          run-time dispatch (this chapter)
  ------------------------          --------------------------------
  you choose ReAct in the           perception estimates complexity
  agent constructor and             and type; planning picks a
  hope every query fits             primitive; evaluation can veto
                                    and force a different primitive
```

Think of ch. 5 as teaching the agent to swing a hammer, a driver, and a
saw. This chapter teaches it to **read the drawing** and put the hammer
down when the job is a cut.

The architectural move is small to say and large to implement: primitives
stay. A **workspace** plus an **attention policy** decide which primitive
is in play, and whether it is earning its keep.

### Defining cognition for agents

Cognition here is **how good the agent's working model of the task is** —
not how poetic the chain of thought looks.

A cognitive agent does not "answer a query." It **builds a representation**
(what is being asked, what would count as done, what is still unknown),
**chooses an approach**, and **revises the representation** when tools
talk back.

Four capabilities you can actually test:

1. **Novel decomposition.** Break a problem the prompt never templated
   into subtasks that map onto tools. Following a runbook is not this.
2. **Dependency reasoning.** Know that B needs A's output, and that C can
   run beside A. A flat bullet list of "steps" with no edges is a wish.
3. **Compositional tool use.** Combine tools in an order nobody enumerated.
   Search that returns a path, then a filesystem read of that path, then a
   computation on the file, is composition. Three unrelated calls are not.
4. **Representation update.** When the observation contradicts the task
   model, *change the model*. "The user already tried the standard fix"
   must demote that fix from "plan" to "ruled out."

Skip (4) and (1)–(3) still tour the wrong neighborhood with confidence.

### Defining metacognition for agents

If cognition is the quality of the thinking, metacognition is **thinking
about that thinking** in a way you can gate on. Three operational meanings:

**Confidence calibration.** Not the string "I'm not sure" — models can be
prompted to say that about anything. Real calibration means **internal
signals predict correctness**: token likelihoods, retrieval scores,
agreement across findings. Verbal uncertainty is cheap and poorly
correlated. Structural confidence on the workspace is what you gate
execution on.

**Stagnation detection.** After two hops, are the findings near-duplicates?
Has confidence flatlined? A non-metacognitive agent hits the iteration
cap. A metacognitive one **raises a signal** and forces a strategy change
*inside* the current L2 iteration.

**Knowledge-boundary awareness.** Distinguish "we have coverage," "we are
on the edge," and "we are outside." Outside is not a prompt to try harder
in the same store. It is a different behavior: say so, gather more from a
*different* source, or hand off.

Metacognition is the layer that makes cognition **safe to run unattended**.
Without it you have a capable intern with no manager.

### Three theoretical foundations (high level)

The architecture is not a copy of a human brain. It **borrows three design
rules** from systems that already produce intelligent-looking behavior,
then implements each rule as a software component.

```
  Minsky: many specialists,     -->  modules (perceive, plan, ...)
  not one genius blob

  Baars: a small shared stage   -->  cognitive workspace
  that specialists read/write

  Kahneman: cheap recognition    -->  attention routes "fast path"
  vs expensive deliberation         vs full cycle by complexity
```

**Society of specialists.** Intelligence as a **committee of narrow
skills** that compete and cooperate, not as one prompt that does
everything. You already met a cousin in
[ch. 4](../4-multi-agent-systems/) (flow / hub / team). Here the
"agents" are **cognitive faculties** sharing one workspace, not product
personas arguing in Slack.

**Shared workspace.** Only a few items are "on stage" at once. Modules do
not whisper in private side channels that the others cannot see. If
evaluation cannot read what execution just found, it cannot metacognize.
The workspace is that stage — closer to a blackboard than to a chat log.

**Dual process.** Easy work should not pay for six modules. Hard,
contradictory work should not skip evaluation to save latency. Attention
is that **depth router** — an engineering policy, not a claim the model
"has System 1." Steal the separation of concerns, not the neuroscience.

## Mapping the mind

Now the shop floor. Seven pieces: a workspace, five cycle modules, and
memory as the long-lived graph those modules consult and update.

### Architecture overview

```
                    +-------- attention (router) --------+
                    |  fast path | full cycle | interrupt |
                    +-------------------------------------+
                                      |
        perceive --> [maybe memory] --> plan --> execute --> evaluate
                                      |
                                      v
                               present / replan /
                               gather / escalate
```

Around a **structured workspace** (not a transcript):

| Slot | What it holds |
|---|---|
| Task representation | Type, entities, ambiguities, complexity |
| Active hypotheses | Candidate strategies, including rejected ones |
| Intermediate results | Findings with source, relevance, quality notes |
| Confidence state | Current value plus a short trend |
| Execution history | What was tried, in order |
| Attention signals | Flags that break the default cycle |

Perception writes the task model. Planning writes a strategy. Execution
writes findings. Evaluation writes confidence and signals. Attention
**reads signals and chooses the next writer**. Memory reads and writes a
graph that outlives the cycle.

This is a **pipeline that can interrupt itself**, not "one agent with more
tools." L2 from [ch. 9](../9-agentic-loop/) still owns iteration and exit;
this architecture owns **quality of processing inside an iteration**.

### The cognitive workspace

The workspace is the beating shared object. Every module reads it. Every
module writes a **typed slice**. It is not conversation history, not a
scratchpad of leftover tokens, and not the `ResearchState` from ch. 9
alone — though if you built that state, this will feel like a sibling that
also tracks *how the reasoning is going*.

Ch. 9's research state accumulated **findings**. The cognitive workspace
also accumulates **self-assessment**: confidence, contradictions among
findings, whether the current strategy is earning progress, which
attention flag is live.

Typical typed pieces (names are yours; the *slots* are the point):

- **Task type** — simple lookup, multi-step, contradictory, ambiguous,
  compositional. Perception's first job is to pick one honestly.
- **Strategy type** — the primitive or plan family in play (direct
  retrieve, ReAct-style hop, hypothesis test, explore-then-plan).
- **Attention signal** — `NONE`, stagnation, contradiction, low
  confidence, knowledge gap. Default cycle only when `NONE`.
- **Finding** — content, source, relevance score, quality note. Raw tool
  JSON is not a finding until it is annotated.
- **History** — enough of the last N steps to detect overlap and
  "we already ruled this out."

**Problem** — Hidden state in prompt prose ("remember we tried X").

**Solution** — If a later module must act on it, it is a **field**. If it
is only in the transcript, it will be dropped, drowned, or hallucinated
back.

Treat the workspace like SPAL in [ch. 1](../1-rise-of-ai-agents/): boring
and printable. If you cannot explain the next route from a dump, the
architecture is theater.

### Perception

Perception is first contact. It turns a raw utterance into a **task
model** before anyone calls a tool.

That is more than parse. Classify the problem, pull entities, list
ambiguities, **estimate complexity**. The complexity number is not
decoration. It is the dual-process switch:

- Low complexity (say, under ~0.3) plus a memory hit → attention may
  **skip** plan/evaluate and go to a cheap execute or even a direct
  reply. That is the fast path. You pay for it when you mis-score a
  hard ticket as easy.
- High complexity (say, over ~0.7) → full cycle, and usually a **memory
  consult before planning**, so past scars inform the first strategy.

Task types that earn their keep in logs:

- **Simple lookup** — one fact, one likely source.
- **Multi-step** — a sequence with dependencies.
- **Contradictory** — the user already named a fix that failed, or two
  sources will disagree.
- **Ambiguous** — missing a slot you must not guess (which env? which
  customer?).
- **Compositional** — will need tools in combination, not in parallel
  isolation.

If perception always emits `multi_step` at 0.5, you have a default, not a
module. Calibrate it on a handful of tickets you already know.

### Planning

Planning reads the task model and **selects a strategy**, optionally
colored by memory. This is where the architecture stops being "an agent
that always ReActs."

```
  type + complexity + memory hits
            |
            v
     +------+------+
     | lookup, low | --> direct retrieve / one tool
     | ambiguous   | --> clarify or explore before commit
     | contradict  | --> hypothesis test (the failed fix is a negative)
     | compose     | --> explicit dependency graph of tools
     | hard + novel| --> heavier primitive (tree, sequential thinking)
     +-------------+
```

A standard agent has one approach: receive, maybe tool, reply. A planning
module has a **menu** and a reason to pick. Hypothesis-test is the
clearest example: it should fire when perception marked **contradiction**,
not because you like the name.

Memory changes the menu. If the graph already knows "this class of
incident was connection-pool exhaustion last quarter," the first plan
should not be "increase timeouts" again. That is not magic. It is
**planning that reads observations attached to a problem-type entity**.

Write rejected strategies into the workspace. Otherwise the next plan
will rediscover the failure you just paid for.

### Execution

Execution is the simplest module, and the one people over-train because it
looks like "the agent." It takes the **current plan step**, calls tools
(function tools or MCP), and writes **annotated findings** back.

If you have shipped ReAct, the call/observe rhythm is familiar. The
difference is **context and wrapping**. Results do not go straight to the
user. They go to the workspace as findings: content, source, relevance,
quality note. Evaluation reads those fields. A raw blob with no scores
forces evaluation to guess, which is how glossy misses sneak through.

Keep execution **thin**. Do not hide planning or evaluation inside the
executor "to save a hop." You will lose the ability to interrupt.

MCP belongs here as **hands**: filesystem, search, internal APIs. Memory
MCP is a different server (below). Mixing "search the repo" and "remember
that pool exhaustion" in one tool description is how schemas rot.

### Evaluation

Evaluation is the **metacognitive core**. It does not find facts. It does
not execute the plan. After each execution step it asks: did this advance
the task? Are findings consistent? Is confidence moving? Continue, replan,
or stop?

Three of the five failure modes die here if you let them:

- Evidence-quality assessment → glossy miss
- Stagnation / overlap checks → groove lock
- Contradiction flags → plan rigor mortis (when paired with attention
  routing back to planning)

Typical structured output (again, names are local):

- Progress assessment (moved / stalled / reversed)
- Consistency check (bool plus the conflicting pair)
- Confidence delta (up, down, flat)
- Recommendation: continue, replan, gather more, escalate / present with
  uncertainty

This is cousin to [ch. 7](../7-evaluation-and-feedback/) judges and
grounding — **in the loop**, on the workspace, every hop, not only a
Phoenix score after the user already saw the answer. Use both. Ch. 7 is
how you know the *system* is regressing. This module is how *this run*
refuses to lie.

### Attention

Attention is what makes this a **cognitive architecture** rather than a
fixed multi-agent pipeline. It reads signals and **grants the next
module control**. Default order is perceive → plan → execute → evaluate.
Signals break that order.

```
  if complexity low AND memory hit:
       fast path --> execute or reply
  elif signal == CONTRADICTION:
       planning (new strategy; old one is in history)
  elif signal == STAGNATION:
       planning with "do not repeat" constraint
  elif signal == KNOWLEDGE_GAP:
       memory (broader) then maybe planning for exploratory goals
  elif signal == LOW_CONFIDENCE:
       execute (different source) or gate (do not present)
  else:
       next step in the default cycle
```

The fast path is the dual-process win: familiar, cheap tickets should not
tour every module. In production that is latency and tokens. The failure
mode is **mis-routing a contradictory ticket onto the fast path**. Log
every fast-path decision. Sample them in
[ch. 7](../7-evaluation-and-feedback/).

When debugging "why did it do that?", do not start in the LLM. Start in
this decision tree and the workspace flags. If the flags are wrong,
perception or evaluation is lying. If the flags are right and the route
is wrong, attention is the bug.

### Memory module and the MCP memory server

Memory is the seventh piece: **persistent long-term structure**, not the
session transcript ([ch. 6](../6-memory-and-rag/) already separated those).
The book wires this through the official MCP memory server
(`@modelcontextprotocol/server-memory`): a **local knowledge graph**
with entities, directed relations (active voice), and observations hanging
off entities. No extra vector DB required for this pattern; it persists as
local JSON. You met the server in the memory chapter; here it is the
agent's **institutional scar tissue**.

Map the graph onto how the architecture thinks:

- Problem types → **entities**
- Strategies that worked or failed → **observations** on those entities
- "X co-occurs with Y" / "tool combo Z" → **relations**

Over time this is not a chat dump. It is a **model of experience**.

The memory *module* is not "call the server when the prompt says so." It
is a **proactive cognitive layer** around those tools:

1. **Retrieve before planning** on hard or familiar-looking work, so the
   first strategy is not naive.
2. **Write after evaluation**, not after every noisy hop — store
   strategies and outcomes, not every tool blob.
3. **Broaden on knowledge-gap signals** before you declare outside
   coverage.

Platform-shaped session stores and org indexes remain
[platform ch. 4](../../platform/4-session-service/) and
[platform ch. 5](../../platform/5-data-service/). Do not merge them into
this graph. This graph is **the agent's learned problem-solving memory**.
Those services are **how the organization keeps transcripts and documents
alive**. Same word, different job.

## Building and running

Pieces in hand. Assemble a loop, run a ticket a flat ReAct agent will
flub, then add the metacognitive gates that make the architecture worth
its hops.

### The cognitive loop

Layer this on [ch. 9](../9-agentic-loop/):

```
  L2  task loop     keep going / stop (goal, budget, hard cap)
        |
        |  each iteration
        v
  attention  <---- workspace signals
        |
        v
  inner cycle      perceive / remember / plan / execute / evaluate
                   (may run several times in ONE L2 iteration)
```

L2 still owns **macro** iteration, convergence, and exit. The cognitive
cycle owns **micro** "what to do next *inside* this iteration." Attention
sits on the boundary: evaluation raises a signal; attention spends it on
another module **without** returning to L2 yet. Only a clean step (no
live signal) hands control back for a convergence check.

That means one billed L2 iteration can contain: perceive, plan, execute,
evaluate, replan, execute again. You are not paying for architecture
cosplay. You are paying to **not** emit a glossy miss at the end of a
shallow hop.

Termination still layers as in ch. 9 (hard cap, budget, goal, quality).
Stagnation here is **richer**: it can fire *inside* the iteration and
cause a pivot rather than only killing the whole run.

### A complete cognitive agent with MCP

A working composition typically attaches **two** MCP servers:

- Memory server — the graph (experience)
- Domain hands — often filesystem, plus whatever your ticket actually
  needs (search, internal APIs)

Initialize an empty workspace, drop in a real query, run the loop. Then
open the SDK trace (OpenAI dashboard or whatever you wired in
[ch. 2](../2-llms-prompting-agents/)) and **name each span as a module**.
If you cannot tell perception from evaluation in the trace, you assembled
a blob.

Node/`npx` for those servers is [appendix B](../appendix-b-nodejs-mcp/).
Python env and keys are [appendix A](../appendix-a-sample-code/). MCP
shapes and transports remain [ch. 3](../3-mcp/). This chapter assumes
those sockets work so it can talk about **routing**, not about STDIO.

Keep constructors boring: one typed module each, one workspace, attention
as control flow. Clever recursion without a printed workspace is a
distributed deadlock with extra tokens.

### Walkthrough: one cycle that earns the complexity

Need a query a flat ReAct agent gets **wrong in a plausible way**.

> Checkout conversion dropped about 18% after we "fixed" the payment
> webhook. On-call already doubled retries and lengthened timeouts. It
> still fails, and only for VAT-inclusive carts in two EU countries.

A standard agent searches "payment webhook timeout conversion," finds the
runbook you already followed, and pastes it. The user told you that path
is dead. The answer is wrong **by construction**.

A cognitive cycle that deserves the extra modules:

1. **Perception.** Type: contradictory (standard fix already applied).
   Complexity high. Entities: webhook, retries, VAT-inclusive, country
   codes. Ambiguity noted: "fails" might mean decline vs client-side
   abort — do not collapse it yet. Signal still `NONE`.

2. **Memory first** (attention sees high complexity). Graph may already
   know "VAT-inclusive + webhook" from a prior incident (tax-inclusive
   amount vs processor's exclusive field). If not, that miss is itself
   information (weak coverage).

3. **Planning.** Not "increase timeout." Hypothesis-test: the failed fix
   is a **negative constraint**. Candidate: payload shape / tax field
   mismatch, not transport.

4. **Execution.** Search *and* open the webhook payload schema *and*
   compare a failing cart to a succeeding one. Findings are annotated:
   runbook relevance high for "timeouts," quality note "user already
   applied"; schema finding relevance high for VAT.

5. **Evaluation.** Findings contradict the first popular answer.
   Confidence in "timeouts" drops. Confidence in "amount field" rises
   only as far as the evidence goes. Signal: `CONTRADICTION` (popular
   runbook vs payload evidence).

6. **Attention.** Routes back to planning, not to the user. New plan:
   confirm with a second source (recent deploy diff, tax config), still
   no present.

7. **Gate.** Only present when confidence and contradiction policy
   allow. If still on the edge, say so and show sources — do not "sound
   sure."

The win is **refusing the runbook because the workspace recorded that it
had already failed**, not a magic VAT oracle.

### Confidence-gated execution

The simplest metacognitive pattern, and the one you should ship first.

Before anything user-visible, **inspect workspace confidence** (the
number evaluation has been updating), not the model's manners.

A workable policy, as control flow not as poetry:

- Below a hard floor (example: 0.3) → **do not present**. Signal
  uncertainty or escalate. Gathering more on a collapsed confidence is
  often throwing tokens at a knowledge boundary.
- Between floor and a soft bar (example: 0.6) → **gather more** if you
  still have iteration budget; otherwise uncertainty, not a fluent
  guess.
- Live `CONTRADICTION` signal → gather / replan, even if the number
  looks "ok." Averages hide fights.
- A **declining** confidence trend across several hops → red flag. More
  of the same strategy is not medicine.

This is a **gate on structured state**. "I think…" in the output text is
not a gate. You can unit-test the gate without calling a model: feed
workspaces, assert decisions.

Pair this with [ch. 7](../7-evaluation-and-feedback/) grounding when the
answer must cite retrieved text. The gate answers "are we allowed to
speak?" Grounding answers "did we speak only from sources?"

### Stagnation detection and strategy pivot

Groove lock is an **overlap and plateau** problem.

Detectors that stay honest:

- **Content overlap** between the last two findings (token-set Jaccard
  or a cheaper overlap ratio). High overlap → you did not move.
- **Confidence plateau** — last few confidence values sit in a tiny
  band. The agent is busy and unchanged.

On fire: set `STAGNATION`, record *why* (overlap vs plateau), route to
planning with an explicit constraint: **do not repeat the failed
strategy**. Write the failure into memory so the next *session* does not
buy the same groove.

Pivot without detection is thrashing (new strategy every hop). Detection
without pivot is a metric you ignore until `max_iterations`. You need
both.

Cap L2 iterations anyway ([ch. 9](../9-agentic-loop/) hard limit).
Stagnation is how you spend fewer of those iterations on the same brick
wall.

### Knowledge boundary awareness

This is the coverage-bluff patch: **know when you are outside**.

Blend a few cheap signals rather than one magic score:

- Average relevance of findings (or 0 if there are none)
- Whether memory had hits for this problem type
- Current workspace confidence

Map the blend onto **within / edge / outside**. Outside should raise
`LOW_CONFIDENCE` (or a dedicated gap flag) and change behavior:

- **Within** — normal present path, still gated.
- **Edge** — present with sources and explicit limits, or one more
  gather from a *different* class of tool.
- **Outside** — graceful degradation: say the store and graph do not
  cover this; ask a clarifying question; escalate. Do not complete the
  procedure from parametric memory while wearing a source-shaped hat.

False positives (flagging easy questions as outside) annoy users. False
negatives (fluent hallucination on empty retrieval) create incidents.
Measure both. That pair *is* the metric later in this chapter.

### Emergent behaviors

Four behaviors you should not special-case as extra if-statements if the
modules and graph are doing their jobs. They **show up from interaction**.
That is the society-of-specialists bet in code.

**Curiosity.** Low confidence plus no memory hits → knowledge-gap signal
→ memory broadens → if still empty, planning adds **exploratory subgoals**
the user did not name (adjacent config, last deploy, a second store).
From the outside it looks like the agent "wondered." Inside it is a
flag and a route.

**Adaptive persistence.** Execution fails, stagnation fires, attention
sends planning a "do not repeat" constraint, memory records the miss.
Next time the same problem type appears, planning starts somewhere else.
That is learning from failure without a training job.

**Selective depth.** Fast path on easy+familiar; full cycle on
contradictory+novel. Same binary, different token bill. If everything
takes the full cycle, attention is shy. If everything takes the fast
path, attention is lying.

**Composition under constraints.** Evaluation refuses shallow one-tool
answers on compositional types; planning is forced to name a dependency
order; execution actually runs that order. You did not hard-code "always
search then open then compute." You made **shallow answers fail a
check**.

If they do not show up, do not add a "curiosity prompt." Inspect signals,
routes, and whether memory writes actually happen.

## Measuring cognitive capability

Not a leaderboard. A **diagnostic** you can run on *your* agent, aimed at
the same five messes.

| Failure | Probe | Pass looks like | If you fail, open |
|---|---|---|---|
| Glossy miss | Query whose top hit is a TOC / index page | Agent flags junk evidence; does not cite nav as procedure | Evaluation scores |
| Groove lock | Query with no good hits | Pivot within a few hops, not identical retries | Attention + overlap |
| Rigor mortis | Mid-run, user says the first plan failed | Strategy changes; old plan is in history as rejected | Planning + signals |
| Coverage bluff | Empty store on purpose | Uncertainty / escalate, not a fluent invention | Boundary + gate |
| Tool salad | Task that needs search *then* file *then* compute | Ordered composition, not three unrelated calls | Planning graph |

Steal the probes. Write them as fixtures. This is TDAD-shaped
([ch. 7](../7-evaluation-and-feedback/)): the test is the requirement.

### Cognitive efficiency metrics

Four production numbers. Track them on traces, not on anecdotes.

**Cognitive efficiency.** Steps taken versus a sane minimum for that
*task type*. A lookup that tours six modules is a fast-path bug. A
contradictory incident that exits in one hop is an evaluation bug.

**Metacognitive calibration.** Plot confidence vs later correctness (human
or judge). A well-calibrated agent is roughly diagonal. High confidence
on wrong answers is the glossy miss in aggregate. Low confidence on easy
wins is a boundary detector with an itch.

**Adaptation rate.** Hops between stagnation-detected and a strategy that
actually moves findings. Lower is better. Infinite means you detect and
do not pivot.

**Knowledge-boundary accuracy.** False positive rate (uncertainty theater
on easy questions) vs false negative rate (hallucinated procedure with a
straight face). Optimize the pair, not one side.

Those four numbers are honesty and efficiency, not a leaderboard: they
tell you whether the extra modules pay rent.

### Before and after

Run the same probe set on:

1. A flat ReAct agent (ch. 5 shape, one loop).
2. The cognitive composition, **empty** graph (first day).
3. The same composition after a few dozen completed tickets (graph has
   scars).

Compare correctness, calibration, graceful-degradation rate (how often it
*should* signal uncertainty and does), and **tokens per solved ticket**.
The interesting plot is (2) vs (3): experience should raise calibration,
shorten familiar types, and cut stagnation events. If (3) is worse, you
are writing junk into the graph (noisy hops stored as gospel). Memory
hygiene from [ch. 6](../6-memory-and-rag/) still applies: observations
should be outcomes, not transcripts.

Celebrate **fewer incidents per thousand tickets** at similar spend, not
a token spike you can excuse with "but calibration."

### The road toward more general agents

Lab modules are still **narrow** (task types, servers, strategy names).
The **skeleton** is not: workspace, attention, evaluation, gates, graph
memory. Wider agents come from **swapping specialists and keeping the
shop floor** — perception's taxonomy, execution's MCP hands, planning's
menu — not from hoping a bigger model grows a manager. Generalization
here is architectural: compose monitorable skills. You will not ace
compositional benchmarks with this lab agent. You will be pointed at
composition and monitoring instead of at a longer system prompt.

## Next steps in this book

Cognition is the last *architecture* chapter in Lanham. What remains is
**field craft**: five layers from [ch. 1](../1-rise-of-ai-agents/) as they
show up in support, RAG, and research products. The Platform hole (shared
traces, judges, stores that are not a laptop JSON file) is still
[platform ch. 1](../../platform/1-why-a-platform/).

## Check yourself

1. A stakeholder says "we added CoT, ReAct, *and* Reflexion, so the
   agent is cognitive now." What dispatcher is still missing, and which
   failure mode stays likely?
2. Pick a real incident from a system you know. Tag it as glossy miss,
   groove lock, rigor mortis, coverage bluff, or tool salad. Which
   module would you inspect first, and what field on the workspace
   would you print?
3. In one sentence each, define cognition vs metacognition as
   *engineering* terms. Give one test you could run this week for each.
4. Why is "I'm not sure" in the output not confidence calibration? What
   would you plot instead?
5. Map Minsky / Baars / Kahneman onto **module, workspace, router**
   without retelling a psychology textbook. What bug appears if you
   implement two of the three and skip one?
6. When should attention take the fast path, and what log would convince
   you it misfired?
7. How does the cognitive cycle sit *inside* an L2 iteration from
   [ch. 9](../9-agentic-loop/) without replacing L2's termination gate?
8. Sketch confidence-gate decisions for: confidence 0.2; confidence 0.45
   with budget left; confidence 0.8 with an open contradiction signal.
9. Stagnation fires. What must planning receive that it did not have on
   the first attempt, and what should memory write before the session
   ends?
10. You swap this architecture from incident response to internal policy
    Q&A. Which pieces do you replace, and which do you keep? Why is that
    an argument about generalization?

Continue to [Field tips](../11-field-tips/).
