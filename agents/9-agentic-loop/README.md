# 9. The agentic loop

Companion notes for **Chapter 9** of *AI Agents in Action* (2nd edition,
Micheal Lanham; Manning, 2026).

Chapter 1 gave you **SPAL** — sense, plan, act, learn — as the inner
beat of agency. Chapters 2–8 gave you personas, tools, MCP, multi-agent
shapes, reasoning patterns, memory, eval, and a runtime that can host
the result. None of that, by itself, is a *long-horizon* process. This
chapter is **three nested loops**: the inner SPAL cycle, an external
task loop that holds plan and state outside one model call, and a meta
loop in which an *agent* (not a `for` loop) decides strategy. Skip it
and you will either one-shot a research brief, or wrap `Runner.run()`
in `while True` until the bill explains the mistake.

The Platform track is a different book. How a graph becomes a
container with jobs and health checks lives in
[platform ch. 8](../../platform/8-workflow-service/). Control patterns,
handoffs, and hub-versus-team *inside* a graph live in
[ch. 4](../4-multi-agent-systems/). Mention those folders; do not merge
them. This folder stays on **iteration as a designed control system**.

## The mental model

Three layers. Mixing their names is how design reviews go nowhere and
how agents thrash.

```
  LAYER 3  META LOOP
           an AGENT holds decide / plan / state
           subtype: orchestration (hub)  |  collaboration (peers)
                |
                |  chooses the next long-horizon move
                v
  LAYER 2  TASK LOOP
           CODE holds plan + state across iterations
           research-until-done  |  queue-until-empty
                |
                |  each iteration is one (or a few) inner cycles
                v
  LAYER 1  INNER LOOP  (SPAL)
           the agent senses, plans, acts, learns
           until a *short* goal or the caller stops it
```

The one sentence to remember a year from now: **the inner loop thinks;
the task loop remembers the campaign; the meta loop changes the
campaign.**

Two consequences fall straight out of that diagram. First, "we added
a while loop" is not layer 2. Layer 2 externalizes **goal, plan,
state, and the stop decision** so they survive when the model's
context window does not. Second, layer 3 is not "more agents in the
YAML." It is an agent (or a team of peers) sitting on the decide
knob. If Python still owns every branch, you are still on layer 2 —
which is often what you wanted.

A third sketch, because horizon is the reason the layers exist:

```
  SHORT HORIZON                    LONG HORIZON
  -------------                    ------------
  "search this error code"         "brief me on vendor X for diligence"
  one SPAL cycle may suffice       plan has subtopics; state accumulates
  plan lives in this prompt        plan lives in ResearchPlan / a queue
  stop = answer returned           stop = layered termination gate
```

General-purpose models are good at the left column and sloppy at the
right unless you give them a file cabinet. The file cabinet is layer 2.

## Layer 1: the inner loop (sense–plan–act–learn)

Layer 1 is the loop you already met in
[ch. 1](../1-rise-of-ai-agents/). It is worth redrawing here so the
outer layers have something precise to wrap.

```
  goal (this call)
       |
       v
  +--> SENSE   current state, last observation, memory slice
  |      |
  |      v
  |    PLAN    next action that maps onto a tool (or a user reply)
  |      |
  |      v
  |    ACT     runtime executes the tool / speaks
  |      |
  |      v
  |    LEARN   observation: continue, replan, or stop
  |      |
  |      +-- goal met or caller budget --> EXIT
  |      |
  +------+  (carry forward this iteration's context)
```

What makes it **agentic**, rather than a `while` in your script, is
that the *model* chooses the next tool, judges whether the observation
is good enough, and (within the bounds you set) when to stop. You
still own the goal, the tool list, and the hard fences. You do not
click each step.

Internal state accumulates *inside this cycle*: tool results stuffed
back into the thread, a scratch plan in the prompt. That state is
**short-lived**. It dies when the call ends, the window fills, or the
process restarts. Fine for "look up the weather and summarize." Not
fine for "research this company for two hours across twenty searches."

Reasoning patterns from [ch. 5](../5-reasoning-and-planning/) (ReAct,
CoT, Reflexion) live *on this layer*. They make a single cycle
smarter. They do not, by themselves, persist a twelve-item plan
overnight. If you find yourself pasting the whole plan into the system
prompt every turn, you are paying layer-2 tax without getting layer-2
structure.

Deployment from [ch. 8](../8-deploying-agents/) still applies: even
layer 1 needs timeouts. An inner loop with no wall-clock budget is a
runaway worker with extra poetry.

## Layer 2: the task loop

Layer 2 **externalizes** the pieces that must outlive one inner cycle:

- the **goal** (still one sentence the user cares about),
- the **plan** (subtopics, statuses, strategy notes),
- the **state** (findings, sources, queue of items, errors),
- the **decision** of whether to iterate again.

Those objects live in *your* process (Pydantic models, a DB row, a
Redis key) — not only in the transformer's residual stream. Each
iteration the inner agent still runs SPAL, but it is handed a
**compressed view** of the campaign so far, and it must return a
**typed delta** the controller can merge.

```
  ResearchPlan / TaskQueue     <---+  persist
  ResearchState / results      <---+
           |                       |
           v                       |
  pack to_context()                |
           |                       |
           v                       |
  inner agent (layer 1)            |
           |                       |
           v                       |
  typed iteration output ----------+
           |
           v
  termination gate --> continue or synthesize
```

Why bother, besides the context window? Because a general model asked
to "keep going" without an external plan will re-search the first
query, contradict itself, and declare victory. External state is how
you make progress **visible** to code: `3/7 subtopics complete` is a
number you can gate on. A vibe in the chat log is not.

Two major shapes share this layer and must not be collapsed:

| Shape | State looks like | Plan looks like |
|---|---|---|
| Deep research | growing findings + sources | subtopics that get marked done |
| Repetitive tasks | a queue + per-item results | often unnecessary; the queue *is* the work |

Research explores. The task loop consumes a list you already have.
Same controller pattern, different objects.

### Deep research: what "deep" is paying for

Every frontier lab ships a "deep research" button. The word **deep**
here is engineering, not marketing: more search, more reading, a
longer horizon than one RAG hop from
[ch. 6](../6-memory-and-rag/). The difference between a reliable
researcher and a token incinerator is almost entirely:

1. how you **manage state** between iterations,
2. how you **terminate**,
3. how you **attach tools** (search, fetch, files) so the agent can
   actually look at the world.

Get those right and the loop converges. Get them wrong and it
circles. Eval from [ch. 7](../7-evaluation-and-feedback/) still
scores the *report*; the loop controller scores *whether to stop
writing it*.

```
  init plan + empty state
        |
        v
  +--> run inner researcher with context(plan, state)
  |         |
  |         v
  |    parse ResearchIteration
  |         |
  |         v
  |    merge findings, update subtopics
  |         |
  |         v
  |    gate: stop? --no--+
  |         |            |
  |        yes           +--> (maybe compress context)
  |         v
  |    SYNTHESIS AGENT --> report
  +----(only if no)
```

The synthesis box is a different persona on purpose. Exploring and
editing are two jobs. Merging them is how the last search result
overwrites the outline.

### Initial state and plan

Before the first `Runner.run()`, construct two objects the rest of
the loop will pass around. Names from the chapter's shape (you can
rename them; do not skip the fields):

**Plan** — strategic, coarse, revisable.

- A list of **subtopics**, each with `name`, `status` (`pending` /
  `in_progress` / `complete`), and `notes`.
- A `strategy_notes` string for "search academic sources first" or
  "ignore marketing blogs."
- Helpers: `to_context()` (what the model is allowed to see) and a
  **progress summary** (`2/5 complete, 1 in progress`) for logs and
  for the gate.

**State** — the accumulating evidence, not the strategy.

- Goal text.
- Findings (claim + source + which subtopic).
- Iteration index, token/cost counters, last error.
- Anything you refuse to trust the model to remember (URLs already
  visited, so it does not loop the same hit).

```
  ResearchPlan                      ResearchState
  ------------                      -------------
  subtopics[]                       goal
    name / status / notes           findings[]
  strategy_notes                    sources_seen[]
  to_context()                      iteration, spend
  progress_summary                  errors[]
```

**Problem** — Stuffing both objects wholesale into every prompt.

**Solution** — `to_context()` is a *view*. As the campaign grows you
truncate, summarize, or drop completed subtopics' raw notes. Layer 2
exists so you can **control the window** on purpose. If iteration 12
still contains iteration 1's raw HTML, you did not externalize state;
you cloned the transcript into a Pydantic field.

On iteration zero, the plan may be empty. The agent's first job is
then: **write the plan** (subtopics) before pretending to search. Put
that in the instructions. An agent that searches without a plan will
search whatever the temperature likes.

### Tools for a research loop

After the objects exist, attach tools. The book's lab uses an MCP
search server over STDIO (Brave as the example; any search MCP is
the same shape). Appendix B is why Node/`npx` shows up. You can also
register native `@function_tool` search against a local corpus — the
loop does not care *which* packaging, only that **observations are
grounded in a tool result**.

```
  create_search_server()     # MCP STDIO, env has the vendor key
        |
        v
  agent.tools / mcp_servers = [search, maybe fetch_url, maybe files]
```

Multiple sources are normal: web search, an internal wiki MCP, a SQL
tool. Do not hand a research agent a shell "so it can curl anything."
That is a chapter 8 sandbox failure wearing a lab coat.

Lifecycle: connect the MCP server, run the loop, disconnect. Async
context managers exist so you do not leak Node processes. A loop that
forgets to close STDIO servers will look "fine" until the next
exercise cannot bind a port.

The inner agent still needs a model that can **call** those tools
reliably. A cheap model that cannot emit valid tool JSON will spin
the layer-2 controller without ever filling `findings`. That is a
routing problem from [ch. 8](../8-deploying-agents/), not a reason to
delete the gate.

### Iteration body output

Each cycle the inner agent produces work. The controller must
**parse** that work. Free-form markdown is a blog post, not a control
signal.

Define a typed iteration model and force the agent to fill it
(structured outputs, as in [ch. 2](../2-llms-prompting-agents/)):

```
  ResearchIteration
    summary_of_what_I_did
    findings[]            # claim, source, subtopic tag
    subtopic_updates[]    # name, status, notes
    follow_up_questions[] # what to chase next
    goal_satisfied: bool
    confidence: float     # 0-1, treated as a claim, not a fact
    notes_for_next_iteration
```

The controller's job after `Runner.run()`:

1. Validate the schema (reject extras; chapter 8's injection lesson
   applies to tool-shaped output too).
2. Merge findings into `ResearchState` (dedupe URLs).
3. Apply `subtopic_updates` to `ResearchPlan` with rules *you* own
   (do not let status jump to `complete` without a source if that is
   your bar).
4. Persist. Then ask the gate.

```
  Runner.run(agent, context)
        |
        v
  parse ResearchIteration   --fail--> retry / count as error iter
        |
        v
  merge --> plan + state
        |
        v
  termination gate
```

**Problem** — Trusting `goal_satisfied=True` because the model is
confident.

**Solution** — Treat self-assessment as one vote. Layered gates
below. A second "goal/quality" agent is allowed if your eval budget
can stand it; it is still not a proof.

### The termination gate

Stop conditions are **layered**. One predicate is how loops either
never start or never end.

Priority-shaped stack (always keep the first two):

```
  1. HARD MAX ITERATIONS     safety net; always present
  2. BUDGET                  tokens or dollars; production-critical
  3. GOAL SATISFIED          ideal exit; biased if self-reported
  4. QUALITY THRESHOLD       confidence / external evaluator
  5. STAGNATION              N iterations with no new information
```

**Hard max** is the fuse. Pick it from how long a *good* report
actually takes in your domain, then add a little, not from "what is
the biggest number that still feels lucky."

**Budget** is the fuse that finance understands. Shrink per-iteration
context if you keep blowing it; do not only raise the cap.

**Goal satisfied** is the happy path. If only the researching agent
votes, it will eventually vote yes to escape the homework. Mitigations:
require `complete` on all subtopics, require N independent sources,
or ask a separate judge (chapter 7 critic energy).

**Quality threshold** (e.g. confidence ≥ 0.85) is the same trap unless
an external scorer owns the number. Use it as a *shortcut exit* when
eval is real, not as a vibe.

**Stagnation** catches the loop that paraphrases the same finding.
Practical tests: URL overlap, embedding similarity of new findings vs
the last batch, or "follow-up questions unchanged." Two iterations of
near-duplicate text should stop even if the agent wants another
search.

```
  after merge:
       if iterations >= MAX:           STOP_MAX
       elif spend >= BUDGET:           STOP_BUDGET
       elif stagnant(N):               STOP_STAGNATION
       elif plan.done and quality_ok:  STOP_GOAL
       else:                           CONTINUE
```

Order matters. A goal-yes that arrives after you are already over
budget should still be `STOP_BUDGET` if you care about bills. Log the
reason. Phoenix spans should show *why* the loop ended, or you will
debug ghosts.

### Coding the loop

The controller is deliberately boring. Pseudocode you should be able
to recite:

```
  plan, state = init(goal)
  mcp = await connect_search()
  agent = build_researcher(mcp)   # instructions KNOW it is in a loop

  for i in 1..MAX:
      state.iteration = i
      raw = await Runner.run(agent, plan.to_context() + state.view())
      iter = parse(raw)
      merge(plan, state, iter)
      reason = gate(plan, state, iter)
      if reason: break

  report = await Runner.run(synthesizer, plan + state)
  await mcp.cleanup()
  return report
```

Instructions for the inner agent must say, in words:

- You are running **inside an external loop**.
- Iteration 1 with an empty plan → **emit the plan** (subtopics).
- Later iterations → pick incomplete subtopics, search, update
  statuses, do not re-open completed ones without a reason.
- Fill the **typed** iteration schema. Do not invent a parallel
  essay the controller will ignore.
- `goal_satisfied` is a proposal. The controller may ignore you.

If the prompt pretends this is a one-shot chat, the model will try to
write the final report on iteration 1 and then stall.

Keep the researcher off formatting. Headings, executive summary, and
"gaps and limitations" belong to synthesis. Mixing them produces a
half-report at iteration 3 that the loop is afraid to invalidate.

Deploy-shaped extras (chapter 8) that belong in this function even in
a lab: per-iteration timeout, cost counter, structured logs with
`iteration` and `gate_reason`.

### Synthesizing the final output

When the gate fires you have a bag of findings. They overlap, they
disagree, they have holes. A **separate synthesis agent** turns that
bag into a report.

Typed report (shape, not a mandate of field names):

```
  ResearchReport
    title
    executive_summary
    key_findings[]
    sources[]
    confidence_assessment
    gaps_and_limitations
```

Instructions: use the **plan's subtopics as the outline**. Surface
contradictions instead of averaging them into mush. List sources.
State what you did *not* find. That last field is how you keep the
system honest when the gate was `STOP_BUDGET` rather than
`STOP_GOAL`.

```
  researcher (many iterations)     synthesizer (once)
  --------------------------       -------------------
  explores, updates plan           does not search (or searches little)
  typed deltas                     typed report
  allowed to be messy              required to be structured
```

Do not let the synthesizer silently become a second research loop
without a gate. If it must search, it is another layer-2 cycle with
a tiny MAX, not an unbounded encore.

### When to use an agentic loop

Not every agent needs layer 2. Adding a loop to a one-shot task is
how you buy latency and cost with no quality.

Use a loop when **at least one** of these is true:

- **Iterative discovery** — you cannot know the sources in advance
  (research, investigation, "what's going on in this codebase").
- **Progressive refinement** — drafts get better (writing, a design
  that you critique and revise).
- **Batch processing** — a queue of similar items (see the next
  section).
- **Conditional branching** — the next tool depends on the last
  observation (diagnostics, troubleshooting).
- **Multisource aggregation** — relevant sources appear *during*
  execution, not in the original prompt.

Avoid a loop when a single pass is the product: classification,
short Q&A against a known index, format conversion, "extract the
order id." Those want one tool call and a schema, not a campaign.

```
  known answer shape, known tools, cheap mistake?
      yes --> single inner call (maybe ReAct once)
      no  --> is the missing piece *time and evidence*?
                 yes --> layer 2
                 no  --> maybe you need a better tool, not a loop
```

If the failure mode is "wrong tool schema" or "bad retrieval," a loop
will repeat the failure. Fix layer 2 tools / layer 4 knowledge first.

### A repetitive task loop agent

The second layer-2 pattern is **not** exploration. You already have
the work: invoices to post, documents to transform, endpoints to
test, records to migrate. The external object is a **queue**, not a
research plan.

```
  TaskState
    queue[]           pending items
    in_progress
    done[]            {item, result, attempts}
    failed[]          {item, error, attempts}

  each iteration:
    pull next item (or retry a failed one under a cap)
    inner agent + tools process THAT item
    record result
    stop when queue empty OR max attempts OR budget
```

No strategic plan is required: the list *is* the plan. The inner
agent still runs SPAL per item (maybe a tool, maybe a model
transform). The controller still owns retries, idempotency keys
(chapter 8), and "do not hide a failed item inside a cheerful
paragraph."

```
  +--> dequeue item
  |       |
  |       v
  |     process with tools
  |       |
  |       +-- ok --> append done
  |       +-- fail --> retry count++ or failed[]
  |       |
  +------- queue remaining? 
              |
             no --> summary (counts, failures)
```

Retries belong in the **controller**, not in a hope that the model
will remember it already tried `invoice-441`. At-least-once queues
make this mandatory.

When the items are heterogeneous and the *choice of which item* is
itself a reasoning problem, you have slid toward layer 3: an
orchestrator picking the next worker. Keep this section for
homogeneous batches.

## Layer 3: the meta loop

Layer 2's controller is code. Layer 3 puts an **agent** on decide /
plan / state. That is the only distinction that matters. More
processes, more MCP servers, or a fancier Compose file do not
promote you.

Because an agent sits on the knob, layer 3 splits by **who** that
agent is:

```
  ORCHESTRATION META LOOP          COLLABORATION META LOOP
  -----------------------          -----------------------
  one hub agent                    peers (optional turn manager)
  workers are tools                shared state / blackboard
  hub writes the plan              plan may be jointly owned
  hub votes stop                   stop may be a vote or a chair
```

This is the same fork as [ch. 4](../4-multi-agent-systems/)
(hub-and-spoke vs collaboration), now drawn as a **loop**: the hub
or the team may iterate until a meta-level gate fires. Chapter 4
taught handoffs and guardrails. This chapter teaches **when the
outer iteration is itself agentic**.

Platform workflows that *host* either subtype as HTTP jobs:
[platform ch. 8](../../platform/8-workflow-service/) — mention only.

### Multi-agent orchestration loops

A single orchestrator agent keeps the global plan and state. Worker
agents (researcher, coder, writer, critic) are **tools** from the
hub's point of view. Each delegation is an inner (layer 1) or even
a nested layer-2 research loop. The hub's SPAL is: sense the
campaign, plan which worker to call, act by delegating, learn from
the worker's typed result.

```
              +------------------------+
              |  ORCHESTRATOR AGENT    |
              |  plan + state + gate   |
              +-----------+------------+
                          |
            +-------------+-------------+
            |             |             |
            v             v             v
       [ worker A ]  [ worker B ]  [ worker C ]
       as a tool     as a tool     as a tool
            |             |             |
            +-------------+------------>+
                          |
                          v
                    merge, then
                    hub decides:
                    loop or finalize
```

What you buy versus layer 2: the *next subtask* can change when a
worker returns a surprise ("this vendor is a subsidiary; research
the parent"). A hardcoded `for subtopic in plan` cannot notice that
without you encoding the rule.

What you pay: the hub is a bottleneck and a context hog (chapter 4's
warning, still true). The hub must get **typed, small** worker
returns, not novel-length dumps. Timeouts and budgets apply **per
delegation** and to the meta loop as a whole — two fuses.

Instructions for the hub should name the workers as tools, forbid
doing the specialist work itself, and require an explicit
`need_another_round` / `final_output` style field you parse. A hub
that both researches *and* orchestrates will starve the workers and
blow the window.

Nested loops are allowed and dangerous:

```
  meta (hub) --delegates--> research worker
                                |
                                +--> its own layer-2 search loop
```

Multiply MAX_ITERATIONS of the hub by MAX of the worker and you have
a cost surface. Set both. Trace both (`parent_turn_id`).

### Collaborative agentic loops

Collaboration is the other meta subtype: **peers**, not a deputy
list. Agents share a state object (blackboard) and take turns
contributing, critiquing, and revising. A lightweight **turn
manager** may pick who speaks; it should not secretly be an
orchestrator with extra adjectives.

```
  shared state (blackboard)
    draft / findings / critiques / votes
           ^
           |  read-write each turn
           |
  +--------+--------+--------+
  |        |        |        |
  v        v        v        v
  researcher  critic  synthesizer  (example triad)
           |
           v
  turn manager: whose move, and did we stop?
```

A common triad: **researcher** adds evidence, **critic** challenges
and asks for sources, **synthesizer** updates the narrative. Each
sees the others' last writes, so the round is cumulative. That is
the point. Isolated agents that cannot read the blackboard are just
a slow flow.

Stop conditions still layer: max rounds, budget, unanimous (or
chair) "good enough," stagnation (critic repeats the same complaint,
researcher cannot answer). Collaboration without a fuse is a meeting
that invoices by the token.

When this subtype wins: the product *is* multiple viewpoints
(debate, red-team plus builder, style plus factuality). When it
loses: you needed a queue of invoices. Use the task loop.

Chapter 4's costs still apply: traces get harder; you will want
[ch. 7](../7-evaluation-and-feedback/) on the shared artifacts, not
only on each speaker's prose. Guardrails on who may write which
slot (critic cannot silently mark a subtopic complete) are
behavioral policy from the deploy chapter, not a vibe in the
persona.

```
  FLOW (ch. 4)          COLLAB LOOP (this chapter)
  ------------          --------------------------
  stages in order       rounds over shared state
  brittle if stage 1    can revisit, at the cost of
  is wrong              coordination and traces
```

Do not mix "we take turns" with "the hub assigned turns but workers
cannot see each other" without drawing it. The second is
orchestration. Call it that.

## Where this chapter stops

You now have the loop map:

- layer 1 SPAL as the inner, short-horizon engine,
- layer 2 as external plan/state/gate (research and task-queue
  flavors),
- typed iteration output and layered termination,
- a synthesizer that is not the explorer,
- layer 3 as agent-controlled meta: hub orchestration vs peer
  collaboration.

What you do **not** have yet: **cognition and metacognition** as an
architecture — attention, confidence gates, stagnation as a *mind*
problem rather than a `if overlap` problem. That is
[ch. 10](../10-cognitive-agents/).

What this folder will not become: a workflow engine or a full
multi-agent operating manual. When you need handoffs, A2A flows, and
guardrail agents as *graph design*, or HTTP job lifecycle as
*platform design*, leave this directory:

- [ch. 4 — Multi-agent systems](../4-multi-agent-systems/)
- [platform ch. 8 — Workflow Service](../../platform/8-workflow-service/)

Keep the layers named. A `while` that cannot say whether it is layer
2 or 3 is how campaigns thrash.

## Check yourself

1. A stakeholder says "we have an agentic loop" because
   `Runner.run()` sits in `while True`. Which layer do they have,
   and which objects would have to live *outside* the model before
   you would agree they have layer 2?
2. Walk a goal you actually want (research, or a batch of tickets)
   through the three layers. Which layer's stop condition is a
   number in your code, and which is a model's opinion?
3. Why does `to_context()` exist if you already stored the full
   `ResearchState`? What goes wrong on iteration 15 if you skip
   the view?
4. List the five gate predicates in an order you would ship. Where
   does a self-reported `goal_satisfied=True` sit, and how can that
   vote cheat?
5. Typed `ResearchIteration` vs a markdown essay: name two fields
   the controller cannot live without, and one failure mode if
   `confidence` is treated as a measured metric.
6. When would you refuse a loop entirely? Give a one-shot task from
   a system you know, and name the failure a loop would only
   repeat.
7. Research loop vs repetitive task loop: which external object is
   the plan, and why would adding subtopics to a homogeneous
   invoice queue be a mistake?
8. Layer 2 code-as-controller vs layer 3 hub: a worker discovers
   the company is a subsidiary. Which layer can change the plan
   without a human editing the `for` loop, and what budget fuse
   did you just need to multiply?
9. Orchestration vs collaboration: pick one product (support
   triage, due-diligence memo, invoice batch). Which subtype fits,
   and what goes wrong if you use the other?
10. A synthesizer starts calling search "just to fill gaps" with no
    MAX. Which layer did you accidentally nest, and what trace
    fields would have shown the encore?

Continue to [Cognitive agents](../10-cognitive-agents/).
