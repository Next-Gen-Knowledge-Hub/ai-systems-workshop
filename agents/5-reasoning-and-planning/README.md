# 5. Reasoning and planning

Companion notes for **Chapter 5** of *AI Agents in Action* (2nd edition,
Micheal Lanham; Manning, 2026).

## The mental model

An LLM does not wake up with a project plan. It predicts tokens.
**Reasoning**, for an agent author, is two operations that fail
separately:

- **Decomposition** — what are the pieces of this goal?
- **Planning** — in what order, with which tools, and how do the
  pieces join?

You can split the work correctly and sequence it badly. You can
sequence beautifully over the wrong pieces. The traces look
different. The fix is different. If you only say "the model cannot
reason," you will add CoT, trees, *and* a scratchpad in one deploy
and learn nothing.

```
  GOAL
    |
    +-- DECOMPOSE   pieces (may be a CoT list, a plan object, a tree)
    +-- PLAN        order, tools, joins (must live *somewhere*)
    |
    v
  WHERE THE THOUGHT LIVES

    [ IN THE FORWARD PASS ]     model-native / hidden "thinking"
    [ IN THE COMPLETION ]       CoT: a linear chain of text
    [ IN THE TOOL LOOP ]        ReAct: thought <-> act <-> observe
    [ IN EXTERNAL STATE ]       plan object, ST server, blackboard
    [ IN A SEARCH ]             ToT: branches, scores, prune
    [ IN A CRITIQUE CYCLE ]     Reflexion: try -> judge -> retry
```

The one sentence to remember a year from now: **the model has no
persistent world** — if the plan is not in the prompt, a tool
result, or a store you control, it does not exist between calls.

Two consequences fall straight out of that diagram. First,
"reasoning model" vs "we prompted CoT" vs "we run ReAct" vs "we
attached sequential thinking" are **four products**. Mixing the
names in an incident doc is how nobody knows which tokens you
paid for. Second, more structure is not more intelligence. A
single-tool lookup does not want a tree. A long-horizon research
job does not want a single forward pass and a prayer.

Add layer 3 when the job has a long horizon, many competing tools,
expensive mistakes, a need for an audit trail, or a domain outside
the model's comfort. Skip it for one-shot Q&A, a strong model on
a reversible act, and hops that already succeed with a short
instruction.

## Understanding LLM reasoning and planning

Humans "leave the house" without writing a four-step plan. We have
routines and a body. The model has neither. Agents therefore need
**explicit** decomposition and planning for work a person would
do on autopilot — not because the prose is hard, but because
there is no inner simulator that already knows the toaster is
off.

**Non-reasoning (single-pass) models** emit the answer in one
forward pass. Without scaffolding, they short-circuit: a
plausible sentence that skipped the nasty middle step. The
limit is **quality of multi-step structure**, not how many
tasks the model will *attempt*. It will happily attempt a
twelve-step goal in one breath. That is not planning. That is
a long completion.

**Reasoning models** spend extra compute on a deliberation
phase (sometimes hidden, sometimes a visible chain) before the
user-facing tokens. You still own whether that deliberation is
**grounded** (tools, retrieval) and **stored** (a plan object).
A thinking model that never calls `search` is still guessing.
A thinking model that dumps a 4k chain into the next agent still
leaks private working text across hops.

Teams sometimes switch to a reasoning model and delete the
planner. Native deliberation is **layer 3 inside one call**. It
does not give you a typed plan, a tool loop, or a scratchpad that
survives the next hop. Keep the architecture. Turn native effort
**down** on routine tool hops and **up** on the few steps that
are actually hard.

### Chain-of-thought reasoning

**Chain-of-thought (CoT)** is: emit intermediate steps *in the
completion*, then the answer. You are asking the sampler to
spend tokens on a visible working.

It is still **one pass** (or one pass plus hidden thinking).
There is no environment. No tool result can contradict step 3
unless you *also* have a loop. CoT is a way to **not skip
arithmetic and local logic**. Finding whether a wiki page exists
needs an observe step from a tool.

CoT is often treated as a personality ("our agent is thoughtful")
and bolted onto every node, including "format this JSON." Use CoT
when the **error is in the middle of a chain of dependencies**
(dates, constraints, "if A then not B"). Skip it when the hop is
a retrieve, a classify, or a typed extract. You pay tokens either
way. You only buy quality on the first class.

How you invoke it, in practice:

- **Instruction.** "Work step by step, then give the answer."
  Cheap. Sometimes enough. Easy to forget on the next persona
  edit.
- **Few-shot CoT.** Show a *short* worked example of the *same
  shape*. Bad examples become the distribution (witty proofs,
  skipped units).
- **Forced structure.** Numbered steps, a scratch section, then
  a final line the runtime can parse. Better for eval. Still
  not a tool loop.

What CoT does **not** do:

- Recover from a wrong tool observation (there is none).
- Explore alternatives (that is ToT, or sampling several
  completions).
- Remember the plan on the next `Runner` call (unless you put
  the chain in the next prompt — usually a bad idea; store a
  *plan object* instead).

Teams sometimes show the chain to the user because "we value
transparency." The chain is a **debug artifact**. It leaks
policy, private tool traces, and sometimes a wrong path the
model then "helpfully" includes. Log it. Eval it. Default to
showing the **answer plus citations**, not the inner monologue.
If a regulator needs the trail, that is a store and a redaction
story, not a chat bubble.

CoT and model-native thinking can both be linear deliberation.
Native thinking may be hidden and billed as extra tokens you
do not see in the answer. CoT is visible in the completion.
For agents, **visible and parseable** beats hidden unless you
have a vendor trace that you actually retain.

### Reasoning, acting, observing: the ReAct paradigm

**ReAct** interleaves three beats:

```
  THOUGHT   what I believe / what I need
     |
     v
  ACTION    a tool call (or "finish")
     |
     v
  OBSERVE   the runtime result, stuffed back as tokens
     |
     +--> thought again  until stop
```

You can CoT and then emit one tool call at the end. That is still
a single plan. **ReAct is a closed loop**: each observation is
allowed to change the next thought. Wrong query? Reformulate.
Empty list? Try another source. Surprise error? Replan.

ReAct is **reactive**. It does not require a global plan
up front. That is a strength on lookup-heavy work (search,
APIs, "what's on this page?"). It is a weakness on work that
needs a **join** ("do not book the hotel until the flight is
a specific constraint"). Reactive agents locally optimize and
paint themselves into a corner. If you need a horizon, add an
**explicit plan** (next subsection) and make ReAct *execute*
the plan, not replace it.

A persona that says "use tools if needed" often produces a
CoT-shaped trace: a long thought, then a final answer, zero
calls. ReAct is a **runtime loop plus instructions plus
tools**. Missing any one: you do not have ReAct. Check the
trace spans. If the first LLM call already answers, the loop
never started.

The loop that *runs* ReAct is your `Runner` (or equivalent):
call model → execute tool → append observation → call model.
The paper-name is not a vendor feature you toggle. If the
runtime cannot come back after a tool, you have an assistant.

Noisy traces are the tax. Every thought+act+observe is tokens
and latency. Cap `max_turns`. Cap observation size (a 80k HTML
dump will "reason" about ads). Treat observations as
**untrusted data**, not as new system prompts — injection still
applies inside one agent.

ReAct with twelve overlapping tools and no persona rule for
*which* tool first is usually a tool-description and agent-
splitting bug. Fix descriptions and split agents before you add
a tree.

### Planning with LLMs

Planning is the **global view**: an outline of subtasks,
dependencies, and done-checks that lives **outside** a single
forward pass. CoT handles one question linearly. ReAct decides
the next act from the latest observation. Planning writes a
document the next steps can **re-read**.

Because the model has no persistent inner state, that document
must sit in:

- the next prompt (fragile, expensive),
- a typed `Plan` object you pass hop to hop,
- a blackboard slot,
- or a scratchpad tool (sequential thinking, below).

```
  PLAN (stored)
    1. collect sources     [owner: researcher]  status: ...
    2. outline claims      [owner: planner]
    3. draft               [owner: writer]
    4. done-check          [coded or critic]
         ^
         |  each hop reads / patches this
         |
  ReAct or CoT *inside* a hop executes *one* line
```

When the "plan" is only the first paragraph of a ReAct thought,
never written to a store, the agent will search for something
off-plan because a snippet looked interesting. If the plan
matters, **serialize it**. Make replanning a deliberate act (a
tool, a hop, a typed patch), not a vibe in the thought channel.

Planning with LLMs is also **wrong often**. The model will
invent a step for a tool you do not have. It will order
independent steps as a chain (wasted latency) or parallelize
a true dependency (wasted retries). Mitigations:

- **Tool-shaped tasks.** A task you cannot map to a tool plus a
  check is still a wish. The planner should speak in **your tool
  names**.
- **Coded constraints.** "Must call search before plan" is a
  flow, not a hope.
- **Short plans.** Five lines beat a strategy memo. Long
  plans drift; nobody updates step 12.

A planner hop can *use* CoT internally to produce the `Plan`
object. That is CoT in service of planning, not a third
religion.

When a plan must survive **sessions**, you need a memory write
and a later retrieval. This chapter's plan is **working memory
for the job**.

## Instructing agents to reason and plan

The LLM is the reasoning component. The agent is the LLM plus
persona, tools, orchestration, and an environment. Instructing
"reasoning" is mostly **persona + loop + what you store**. You
cannot prompt your way into a loop the runtime does not run.
You cannot prompt your way into a plan you throw away.

Keep role, task, constraints, and tools clear. Layer 3 adds
**how to spend tokens before the act**.

### Applying CoT to an agent

The smallest CoT agent is a persona that says to work step by
step, a question that actually has a middle, and a runner.
No tools. You are testing whether the **instruction** changes
the completion, not whether the product is an agent yet.

What to put in instructions (and what not to):

- Do say: produce numbered steps; put the final answer after
  a delimiter the runtime can split; do not skip unit
  conversions; if a constraint conflicts, name it.
- Do not say: a full ReAct spec; "consider many alternatives"
  (that is ToT, and you did not build a search); "remember
  last week's plan" (no memory layer).
- Do not duplicate a typed schema in prose if `output_type`
  already owns the answer shape. CoT can live in a `reasoning`
  field; the answer lives in `answer`.

CoT instructions on a tool-using agent with no mention of tools
will invent numbers that a calculator tool would have given.
If the hop has tools, CoT must **yield to tools** for facts that
are not in the prompt. Otherwise you trained a narrator.

Sampling: CoT planners usually want **low temperature**. You
are not brainstorming; you are trying to keep step 4 entailed
by steps 1–3. Creative CoT is how you get a different graph
every run and cannot eval.

Eval: score the **answer**, not the eloquence of the chain —
unless the chain is the product (a tutor). A beautiful wrong
chain is a regression. At least look at the split between
scratch and answer.

A CoT hop inside a **flow** should **not** pass the chain
downstream by default. Pass the typed conclusion. The next
agent has its own layer 3.

### Implementing ReAct with agents

ReAct needs three things in the constructor, not in a blog
post:

1. **Tools** with honest descriptions (when to call, what
   "empty" means).
2. **Instructions** that name the loop: think; if you need a
   fact or an act, call a tool; after the result, update the
   thought; then either another call or a final answer.
3. **A runner that actually iterates** (`max_turns` > 1).

Toy tools (date jump, calculator, `get_status`) are valid for
learning the rhythm. Production ReAct fails on **observation
quality**: stack traces dumped as strings, HTML, 429 messages
the model interprets as content. Wrap tools so the observation
is small and structured. Failure handling is part of the agent.

```
  persona: Think, then maybe call travel_back / travel_forward.
           After each result, revise. Then answer.

  tools:   travel_back(year, n) -> year'
           travel_forward(year, n) -> year'

  runner:  until final output or max_turns
```

Copying a ReAct prompt from a paper into an agent that already
has MCP servers is a common trap. The paper's action syntax
(`Search[...]`) is not your tool schema. The model emits
prose "Search: cats" and never calls the function. Instruct in
**your runtime's language**. The schema in context is the action
space. A second dialect is how ReAct demos work in notebooks and
fail in the SDK.

When ReAct should **stop**:

- The persona's done-check is true (answer the user).
- A coded gate says the artifact is complete.
- `max_turns` (a budget, not a suggestion).
- A guardrail trips.

When it should **not** stop: "the thought looks confident."
Confidence is a token pattern.

ReAct + MCP: the act beat can be a native function or an MCP
tool. Same loop. The extra failure domain is transport and
listing. If tools never appear in the trace, you are debugging
MCP wiring, not ReAct.

A ReAct agent that "plans" by thinking about all ten steps,
then executes step 1, then thinks about all ten steps again with
a stale plan pays for both a global essay and a local loop and
gets neither. Either shorten the thought (one next act) or
**externalize** the plan and only ReAct inside the current step.

## Advanced reasoning patterns with agents

Most hops should not get a named research pattern. A
well-scoped extract, a single lookup, a fixed-format rewrite:
the cost of CoT/ReAct is wasted latency. Add structure when
there are **multiple decisions**, **intermediate results**,
or a single pass that keeps failing eval.

CoT and ReAct cover most of what you will ship. ToT and
Reflexion are for when you have accepted **search cost** or
**retry cost** and you can **score** a candidate. If you
cannot score, you cannot prune or critique. You will fan out
spend.

### Tree-of-thought

**Tree-of-thought (ToT)** is a **search procedure** over
candidate thoughts, not a prompt synonym for "think harder."
CoT is one path. ToT **branches**, **evaluates**, **prunes**,
and sometimes **backtracks**.

```
  thought 0
     |-- candidate A   score 0.2   prune
     |-- candidate B   score 0.8   expand
     |      |-- B1
     |      |-- B2     score ...   pick / backtrack
     |-- candidate C   ...
```

It fits when **lookahead** matters more than depth on a
single story: puzzles, move-based games, plans with mutually
exclusive branches ("if we cannot get the API quota, do we
change the product or the vendor?"). It is a poor fit for
time-sensitive UX and for problems with no cheap evaluator
(if "score" is another frontier call on every node, the bill
is the product).

"We enabled ToT" by asking the model, in one completion, to
"consider three options and pick" is **CoT with a list**. Real
ToT needs a controller: generate k candidates, score, expand,
budget. You can approximate with several samples and a judge.
You cannot fully invoke the search with one magic sentence.
Reasoning models eat *some* of this space with longer
deliberation. Explicit search still wins when branches and
scores are **clear**.

Operational reality:

- Branching factor × depth × judge calls = the cost model.
  Write it down before you code.
- State must be stored (which node, which leftover budget).
  A tree that lives only in prose will collapse into a list.
- Partial paths can leak into the user answer. Return the
  **chosen** path's conclusion, not the graveyard.

Do not ToT a ReAct observation. Search over **plans or
moves**, then execute the winner with ReAct. Searching over
tool dumps is how you fork the bill and not the idea.

### Reflexion

**Reflexion** is try → **critique** → try again with the
critique in context. The "learning" language in old write-ups
is misleading. **Weights do not change.** You are conditioning
on a richer prompt. If you do not persist the critique, the
next session is amnesia. If you persist a *bad* critique, you
reinforce it. That is memory hygiene, not magic.

```
  attempt n  -->  artifact
       |
       v
  critic (agent, rubric, or code)
       |
       +-- pass --> done
       +-- fail --> feedback into next attempt   (budget n+1)
```

Compared to ToT: you explore **one path at a time** and spend
on depth + retries, not on a wide frontier. The same *shape*
as a multi-agent critique loop, now named as a reasoning
strategy. The critic can be code (`tests_pass()`) or an LLM.
Code first when you can.

Reflexion with a critic that says "be more detailed" every
time makes the solver write novels. Tokens go up. The answer
does not. Feedback must be **specific and falsifiable**:
"step 3 used 2015 instead of 2016; recompute from the tool."
Vague aesthetic critique is a persona bug on the judge.

Fits: messy generation (code, a plan that almost works),
tasks where the first attempt is cheap and the failure is
detectable. Does not fit: you have no evaluator; each attempt
has a side effect (Reflexion that retries `charge_card` is an
incident). Use **idempotent attempts** or a sandbox.

Reflexion is not a substitute for ReAct. If the failure is
"needed a tool observation," go get it. Critiquing a guess
does not create a fact.

### Selecting the right pattern for your agents

Do not collect patterns. Pick from **task shape** and **what
you can measure**.

| If the hop is... | Start with | Avoid |
|---|---|---|
| One fact, strong model | Native / no extra pattern | CoT essays |
| Multi-step logic, no tools | CoT | ToT (unless it is a search puzzle) |
| Needs live data or acts | ReAct | CoT-only ("it should just know") |
| Mutually exclusive plans you can score | ToT (controller) | One-prompt "consider options" |
| Detectable failure, cheap retry, no side effects | Reflexion | Retrying irreversible tools |
| Long job, many hops | Explicit plan + ReAct per step | One ReAct soup |

**Mix on purpose.** A planner hop (CoT → `Plan`) feeding
executor hops (ReAct) feeding a critic (Reflexion) is a
**graph**, not a smoothie. Each node has one pattern in its
persona. Combining all adjectives in one instruction block
("think step by step, explore branches, reflect, use tools,
keep a global plan") is how the sampler picks a vibe per
run.

When the same agent is ToT on Monday and ReAct on Tuesday
because the persona listed both, the pattern was never part of
the **contract**. Pin it the way you pin temperature. If you
need both, it is two nodes or a coded controller.

Cost is a product constraint. ToT and Reflexion+ToT together
are "expect this to be slow and expensive." If the SLA is a
chat bubble in 800 ms, you do not have that product. Use a
smaller pattern or a smaller model on the hop.

When the **model** gets better, you may thin the pattern, not
the tools. Frontier deliberation is still not a filesystem.

## Utilizing the sequential thinking MCP server

The sequential-thinking (ST) server is a **scratchpad**. The
name oversells. It does not think. It stores **thought
records** the agent writes and later **reads**, so a plan can
survive across tool calls without stuffing the entire chain
into every completion by accident.

That is layer 3 as a **tool**: a reasoning/planning tool in
the taxonomy. MCP is the packaging. The agent still has to
choose to call it. A silent ST server is a process you are
paying to spawn.

```
  agent  --MCP-->  ST server (one tool, thought log / revisions)
              \-->  world tools (search, calc, fs, ...)
```

"We added sequential thinking, so the agent plans" is a
category error. You added **external working memory**. Without
instructions that say *when* to write, *when* to revise, and
*when* to stop, the model may ignore the tool (persona) or
journal every token (cost). Treat ST like any other tool:
descriptions, persona, traces.

### Unchaining the sequential thinking server

Before you solve a puzzle with it, **list the tools** the
server actually exposes. Unchain means: start the server,
attach it to a trivial agent, print `list_tools`, maybe ask
the agent to report what it sees. You are testing **wiring**,
not intelligence.

STDIO vs SSE is a transport choice. The common local story is
`npx` plus STDIO. A new subprocess is a **new empty
scratchpad**. If you expected thoughts to survive across
`Runner` processes, you wanted SSE or some other store. Do
not debug "forgetting" as a reasoning failure.

When the agent lists ST in the persona as "your brain" and
also has search, it may write thoughts about URLs it never
fetched. ST is not retrieval. Persona: use world tools for
facts; use ST to **record the plan and the current step**.
If a thought claims a source, a later hop should see it in
**sources**, not only in the scratchpad.

Inspect with the MCP Inspector the way you would any server.
If Inspector can call the thought tool and the agent never
does, you have a persona/schema problem, not a planning paper
to reread.

Keep the first agent **bored**: "you are a planning
assistant; you have a sequential-thinking tool; list what you
can do." If that run cannot see the tool, stop. Do not pile
on ToT instructions to hide a handshake bug.

### Revisiting hard problems with sequential thinking

Use a problem that **punishes skipped steps** (nested
constraints, several state updates, a done-check that is easy
to fake in prose). The book's time-travel puzzles are one
genre: lots of local arithmetic, easy to narrate wrongly.
Your genre might be an SLA calculation, a migration order, a
policy with exceptions.

Wire:

- **World tools** that make the state change *real* (even if
  they are toys: `travel_back`, `apply_discount`). The agent
  should not do the arithmetic only in ST.
- **ST** for the plan and the running notes.
- **Instructions** that force: draft a plan in ST; execute
  with world tools; after each observation, **revise** the
  plan if it broke; then answer.

When ST holds a beautiful plan, world tools are never called,
and the answer matches the plan and not reality, the scratchpad
became CoT with extra latency. Eval must require **tool
spans**. A plan-only success is a fail if the product is
supposed to act.

Pass observations **into** the next thought (ReAct). Do not
hope the model will remember an observation that you did not
put in the window or in ST. If ST is the store, the agent
must **write** the observation summary. Agents forget to
write. Persona should make the write part of "done" for the
beat.

Cap thought size. A scratchpad that accretes every discarded
branch is a second context-dilution machine. Revise means
**replace**, not only append, when the plan changed.

### Advanced use: combining patterns on the scratchpad

ST is a good **place** to combine patterns because the
combination needs state:

- CoT (or a planner hop) **writes** a plan into ST.
- ReAct **executes** the current step with world tools.
- ToT **branches** in ST (if the server/tooling lets you
  mark alternatives) or you store branch ids yourself.
- Reflexion **writes a critique** and a patched plan.

That paragraph is a **system design**, not an instruction
dump. If you paste all of it into one persona, you will
get a random subset per run. Prefer **coded phases** or
**split agents**: a planner that may only write ST + a typed
plan; an executor that may only run world tools and patch
status; a critic that may only write feedback.

```
  [ Planner: CoT ] --plan--> ST
  [ Executor: ReAct ]  reads ST, calls world tools, patches ST
  [ Critic: Reflexion ]  reads artifact + ST, verdict / patch
  [ optional ToT controller ]  scores alternative plans in ST
```

One listing that enables ToT *and* Reflexion *and* ST *and*
two world tools will surprise you at the bill. You built the
expensive corner of the pattern table on purpose. Time it.
Budget it. Pin the model. Do not use that agent for "what's
the office Wi-Fi." Advanced combination is for **ambiguous,
high-value** jobs. Low-latency paths stay on CoT-or-less.

Results will **vary by model**. A strong reasoning model may
need less ToT. A weak model may fill ST with noise. That is
not an argument against the store. It is an argument for
**eval per model** and for not treating ST content as ground
truth.

A few-shot **plan** in the persona (a worked example of the
*strategy*, not a copied puzzle answer) can stabilize CoT
planners. Keep it short. A full solved novel in the
instructions is how every new problem becomes a remix of the
example (wrong numbers, right vibe).

When ST is overkill: the hop is one tool call; the plan is
three bullets you could put in a `Plan` pydantic model and
pass in memory. **Do not start an `npx` process to store
three bullets.** Externalize when the plan is revised across
many tool calls or across agents that should not share a
thread.

## Check yourself

1. An agent decomposes "onboard a vendor" into five sensible
   pieces, then emails legal before security has signed.
   Which operation failed (decomposition vs planning), and
   what would you store so the next hop cannot "forget" the
   dependency?
2. You enable a vendor "thinking" model and delete CoT
   instructions. The hop still never calls search. What did
   native deliberation not give you, and which trace span
   would prove it?
3. When is CoT the right layer-3 move, and when is it a tax?
   Give one job from your work for each. What should happen
   to the chain before the next agent in a flow sees the
   hop?
4. A persona describes ReAct. `max_turns=1`. Tools are
   registered. What pattern do you actually have, and what
   would the trace of a real ReAct run have to show that
   this one cannot?
5. Why is ReAct a bad only-pattern for "book flight then
   hotel under one constraint"? What object would you add,
   and which beats stay reactive?
6. A teammate "turns on ToT" with one prompt: "consider
   three options." What search pieces are missing
   (generate / score / expand / budget), and when would
   you *still* accept the cheap approximation?
7. Reflexion retries a `send_invoice` tool after a critic
   says the memo was rude. What invariant did you break,
   and how would you reshape attempts so critique cannot
   create a side effect?
8. Using the pattern table in this chapter, pick a pattern for
   (a) a SKU lookup, (b) a logic puzzle with no APIs, (c)
   a plan with three mutually exclusive vendors you can
   score. Name what you **pay** in each.
9. Sequential-thinking MCP: the Inspector shows the tool;
   the agent never writes thoughts; across two `Runner`
   processes the "plan" is empty anyway. Separate a
   **persona** miss from a **transport/lifetime** miss.
   What experiment distinguishes them?
10. You want CoT to draft a plan, ReAct to execute, and
    Reflexion to patch. Why is that three contracts (or
    three agents) rather than one adjective-rich persona,
    and where does ST help vs a typed `Plan` passed in
    code?
