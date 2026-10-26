# 10. Cognitive agents

Companion notes for **Chapter 10** of *AI Agents in Action* (2nd edition, Micheal Lanham; Manning, 2026).

This chapter establishes cognition and metacognition as code. You can stack reasoning patterns, a loop, and memory and still lack a shop floor: something that picks a skill, watches the work, and reroutes when the work stalls. Afterward you can name the five failure modes, the workspace slots, the attention route, and the gates on confidence, stagnation, and coverage.

## 1. A shop floor for skills

Reasoning patterns are skills. A cognitive architecture is the floor that picks a skill, watches the work, and changes skill when the work stalls.

```
  user query
       |
       v
  COGNITIVE WORKSPACE
  task · hypotheses · findings · confidence · history · flags
       ^
       |
  PERCEIVE  REMEMBER  PLAN  EXECUTE  EVALUATE  ATTEND
```

* **Cognition**: The quality of the agent's internal model of the task. What is being asked, what would count as done, what is still unknown.
* **Metacognition**: The quality of the agent's model of that model. Confidence, progress, and the edge of what it knows. Architecture owns both. A longer prompt owns neither.
* **Primitives stay**: Chain-of-thought, ReAct, tree-of-thought, and Reflexion remain useful. They are the tools on the bench. A factual lookup does not need a thought tree. An ambiguous, contradictory ticket should not be a single chain-of-thought pass. None of those patterns knows it is the wrong pattern.
* **Run-time dispatch**: Perception estimates complexity and type. Planning picks a primitive. Evaluation can veto and force a different one. Attention grants the next module control. Choosing ReAct once in the constructor, and hoping every query fits, is the design-time habit this floor replaces.
* **Inside the iteration, not instead of it**: An outer task loop still decides keep-going versus stop: goal, budget, hard cap. Attention decides which module runs right now. One outer iteration may contain several inner module steps.
* **Name the failure, then the module**: A glossy miss is unjudged evidence, not a missing search. Groove lock is a missing progress signal. Another paragraph of "think carefully" installs neither. Chain-of-thought, ReAct, tree-of-thought, and Reflexion stay on the bench. The dispatcher is the new piece.

We are not arguing about machine consciousness. The target is observable behavior you can log, gate, and regress.

## 2. Five failures you can tag

These are the default of an agent that can reason and cannot monitor the reasoning. Run ten diverse tickets through the current bot and tag the logs. A production incident you cannot tag this way is still a vibe.

1. **Glossy miss**: Retrieval returns a landing page, a README heading, or an API table of contents. The agent quotes it as the fix, with certainty. The search happened. The evidence was not judged. Evaluation that scores relevance, and refuses to treat metadata as content, is the patch. "Be thorough" does not add that score.
2. **Groove lock**: The same query, the same tool, the same near-duplicate finding, until `max_iterations`. The deficit is progress sensing. Without a signal that this step did not move the workspace, the outer loop is a `while` with extra tokens.
3. **Plan rigor mortis**: Perception classifies the ticket as timeouts. Planning commits to "raise timeouts." New evidence says the timeouts were already raised. The agent finishes the original checklist anyway. The deficit is permission to throw the plan away.
4. **Coverage bluff**: Tools and memory both miss. Parametric knowledge fills the hole with a fluent procedure. The deficit is knowing the edge of coverage, and having an off-ramp that is a behavior, not the word "maybe" in a paragraph.
5. **Tool salad**: The agent has search, a filesystem, and a calculator, and uses each once. It never runs them as a pipeline: search, open the file the search named, compute from the file. The deficit is dependency-aware composition. The tools were already there.

Each one is a module you can point at in a trace. Glossy miss and coverage bluff open evaluation. Groove lock opens attention plus evaluation. Rigor mortis opens planning plus attention. Tool salad opens planning plus execution. Print the field that module owns.

* **Glossy miss in the dump**: Relevance is high on a table-of-contents URL, and the quality note is empty. The agent quoted navigation as a procedure.
* **Groove lock in the dump**: Execution history shows the same query and the same URLs. The outer counter is the only thing moving.
* **Rigor mortis in the dump**: Active hypotheses still contain only "raise timeouts," and nothing is marked rejected after the user said that fix already shipped.
* **Coverage bluff in the dump**: Confidence is high, average relevance is zero, and memory had no entity for this problem type. The fluent steps came from parametric memory.
* **Tool salad in the dump**: The plan lists search, a file read, and a calculator, with no edge saying the read needs the path the search returned.

## 3. Cognition you can test

Cognition here is how good the working model of the task is. A chain of thought can look poetic and still be the wrong model. A cognitive agent builds a representation, chooses an approach, and revises the representation when tools talk back.

1. **Novel decomposition**: Break a problem the prompt never templated into subtasks that map onto tools. Following a runbook is not this. It breaks when every ticket is forced through the same checklist.
2. **Dependency reasoning**: B needs A's output. C can run beside A. A flat bullet list of steps, with no edges, is a wish. It breaks when parallel calls assume a file that search has not named yet.
3. **Compositional tool use**: Combine tools in an order nobody enumerated. Search that returns a path, then a filesystem read of that path, then a computation on the file, is composition. Three unrelated calls are not.
4. **Representation update**: When the observation contradicts the task model, change the model. "The user already tried the standard fix" demotes that fix from plan to ruled out. Skip this and the first three capabilities tour the wrong neighborhood with confidence.

## 4. Metacognition you can gate

Metacognition is thinking about that thinking in a way you can gate on. It is what makes cognition safe to run unattended. Three operational meanings.

1. **Confidence calibration**: Internal signals that predict correctness. Token likelihoods, retrieval scores, agreement across findings. The string "I'm not sure" is verbal uncertainty. Models can be prompted to say it about anything, and it correlates poorly with being wrong. Structural confidence on the workspace is what you gate execution on. Plot workspace confidence against later correctness. A calibrated agent sits near the diagonal. High confidence on wrong answers is the glossy miss in aggregate. Low confidence on easy wins is a boundary detector with an itch.
2. **Stagnation detection**: After two hops, are the findings near-duplicates? Has confidence flatlined? A non-metacognitive agent waits for the iteration cap. A metacognitive one raises a signal and forces a strategy change inside the current outer iteration.
3. **Knowledge-boundary awareness**: Distinguish "we have coverage," "we are on the edge," and "we are outside." Outside is not a prompt to try harder in the same store. It is a different behavior: say so, gather from a different source, or hand off. Without this distinction you have a capable intern and no manager: the work continues, fluently, past the point where the tools went empty.

A test you can run this week for cognition: hand the agent a ticket whose standard fix the user already tried, and see whether the task model demotes that fix. A test for metacognition: plot workspace confidence against whether the answer was actually right. The sentence "I'm not sure" fails the second test even when the first representation was good.

## 5. Three design rules

The floor borrows three design rules from systems that already produce intelligent-looking behavior. It implements each rule as a component. It does not copy a human brain.

1. **Society of specialists** (Minsky): Intelligence as a committee of narrow skills that compete and cooperate, not one prompt that does everything. The modules are perceive, remember, plan, execute, evaluate, attend. Product shapes that split personas — a flow, a hub, a team — are cousins. Here the specialists are faculties sharing one workspace, not personas arguing in a chat. Skip the specialists and you are back to one blob with a longer prompt.
2. **A small shared stage** (Baars): Only a few items are on stage at once. Modules do not keep private side channels the others cannot see. If evaluation cannot read what execution just found, it cannot metacognize. The workspace is that stage, closer to a blackboard than to a chat log. Skip it and every module reasons from a different transcript.
3. **Cheap recognition versus expensive deliberation** (Kahneman): Easy work should not pay for six modules. Hard, contradictory work should not skip evaluation to save latency. Attention is that depth router. It is an engineering policy. It is not a claim that the model "has System 1." Skip the router and either every ticket tours the full cycle, or every ticket takes the fast path.

Steal the separation of concerns. Leave the neuroscience. Build two of the three and the third hole is specific. Specialists plus a workspace, with no router, send every ticket through six modules or skip evaluation to save a hop. A workspace plus a router, with one blob instead of modules, has nothing narrow to route to. Modules plus a router, with no shared stage, let evaluation metacognize a transcript execution never wrote down.

## 6. The workspace

Seven pieces: one workspace, five cycle modules, and memory as the long-lived graph they consult. Around a structured workspace, not a transcript, each slot has a job.

| Slot | What it holds |
|---|---|
| Task representation | Type, entities, ambiguities, complexity |
| Active hypotheses | Candidate strategies, including rejected ones |
| Intermediate results | Findings with source, relevance, and a quality note |
| Confidence state | Current value plus a short trend |
| Execution history | What was tried, in order |
| Attention signals | Flags that break the default cycle |

* **Who writes**: Perception writes the task model. Planning writes a strategy. Execution writes findings. Evaluation writes confidence and signals. Attention reads the signals and chooses the next writer. Memory reads and writes a graph that outlives the cycle.
* **A pipeline that can interrupt itself**: Default order is perceive, plan, execute, evaluate. Signals break that order. This is not "one agent with more tools." The outer task loop still owns iteration and exit. This floor owns the quality of processing inside an iteration.
* **Sibling of research state**: A research loop accumulates findings. The workspace also accumulates self-assessment: confidence, contradictions among findings, whether the current strategy is earning progress, which attention flag is live.

The workspace is not conversation history, not leftover tokens, and not the findings alone. Typical fields, whatever you name them:

* **Task type**: Simple lookup, multi-step, contradictory, ambiguous, or compositional. Perception's first job is to pick one honestly.
* **Strategy type**: The primitive or plan family in play. Direct retrieve, a ReAct-style hop, hypothesis test, explore-then-plan.
* **Attention signal**: `NONE`, stagnation, contradiction, low confidence, or knowledge gap. The default cycle runs only when the signal is `NONE`.
* **Finding**: Content, source, relevance score, quality note. Raw tool JSON is not a finding until it is annotated.
* **History**: Enough of the last N steps to see overlap and to see "we already ruled this out."

Hidden state in prompt prose ("remember we tried X") gets dropped, drowned, or hallucinated back. If a later module must act on it, it is a field. If you cannot explain the next route from a dump of the workspace, the architecture is theater.

Read a bad run in slot order, before you reread the prompt.

* **Task representation**: What perception thought this was. A wrong type explains a wrong menu.
* **Active hypotheses**: Whether the failed strategy is still the plan. Rigor mortis sits here when nothing is marked rejected.
* **Intermediate results**: Whether each finding has a source and a quality note. A glossy miss is a heading with high relevance and an empty note.
* **Confidence state**: The level and the trend. A gate that reads only the latest number misses a decline across hops.
* **Execution history**: The last N steps. Groove lock is the same query copied down the list.
* **Attention signals**: Who should write next. An empty flag means the route was never given a reason to break the default cycle.

## 7. Perception and planning

Perception is first contact. It turns a raw utterance into a task model before anyone calls a tool. Classify the problem, pull entities, list ambiguities, estimate complexity. The complexity number is the dual-process switch.

* **Fast path**: Low complexity, under about 0.3, plus a memory hit. Attention may skip plan and evaluate and go to a cheap execute, or to a direct reply. You pay for that skip when you score a hard ticket as easy. A contradictory webhook ticket filed as 0.2 because the word "timeout" looked familiar is the misfire. Log the score and the type together.
* **Full cycle**: High complexity, over about 0.7. Usually a memory consult before planning, so past scars inform the first strategy. The band between 0.3 and 0.7 is where you decide with the type, not with a default of 0.5.

Task types worth logging, because the route depends on the name:

1. **Simple lookup**: One fact, one likely source. A fast path is honest. It breaks when the "one fact" is a contradiction the user already described.
2. **Multi-step**: A sequence with dependencies. Plan the edges. It breaks when you flatten the sequence into parallel calls.
3. **Contradictory**: The user already named a fix that failed, or two sources will disagree. Hypothesis-test, not the popular runbook. It breaks when perception files it as a lookup because "timeout" was in the text.
4. **Ambiguous**: A slot you must not guess. Which environment. Which customer. Clarify or explore before you commit. It breaks when the model fills the slot from the most common tenant.
5. **Compositional**: Tools in combination, not in parallel isolation. It breaks into tool salad when each tool is called once and nobody names an order.

If perception always emits `multi_step` at 0.5, you have a default, not a module. Calibrate it on a handful of tickets whose type you already know.

Planning reads that task model and selects a strategy, optionally colored by memory. This is where the floor stops being an agent that always ReActs.

```
  lookup, low complexity --> direct retrieve, or one tool
  ambiguous               --> clarify, or explore, before you commit
  contradictory           --> hypothesis test; the failed fix is a negative
  compositional           --> an explicit dependency graph of tools
  hard and novel          --> a heavier primitive (a tree, sequential thinking)
```

* **A menu, and a reason**: Hypothesis-test fires because perception marked contradiction, not because the name is attractive.
* **Memory changes the menu**: If the graph already knows this class of incident was connection-pool exhaustion last quarter, the first plan is not "increase timeouts" again. Planning read an observation attached to a problem-type entity.
* **Rejected strategies are fields**: Write them down. Otherwise the next plan rediscovers the failure you just paid for.

## 8. Execution and evaluation

Execution takes the current plan step, calls tools (function tools or MCP), and writes annotated findings. If you have shipped ReAct, the call-and-observe rhythm is familiar. The difference is where the result goes.

* **Findings, not a user reply**: Content, source, relevance, quality note land on the workspace. A raw blob with no scores forces evaluation to guess. That is how a glossy miss sneaks through.
* **Keep the executor thin**: Planning and evaluation do not hide inside it "to save a hop." You would lose the ability to interrupt.
* **Two kinds of MCP**: Domain hands — filesystem, search, internal APIs — belong on execution. The memory server is a different process. One tool description that both "searches the repo" and "remembers pool exhaustion" rots the schema.
* **Interrupt requires a seam**: Hide planning inside the executor and a contradiction found mid-call has nowhere to go except the final paragraph. The user sees the runbook. Any flag is written after the reply. Keep the executor on one plan step, then let evaluation write, then let attention choose.

Evaluation is the metacognitive core. It does not find facts and it does not execute the plan. After each execution step it asks whether this advanced the task, whether findings agree, and whether confidence moved. Then it recommends continue, replan, gather more, or escalate and present with uncertainty.

* **Progress**: Moved, stalled, or reversed. Stalled plus a high relevance score on a navigation page is still stalled. The page was about the topic. It was not the fix.
* **Consistency**: A boolean plus the conflicting pair. "Timeouts" versus "the payload field" is the pair. A single blended sentence deletes it.
* **Confidence delta**: Up, down, or flat. The trend matters as much as the level. Three flat hops are a plateau even if the absolute number looks acceptable.
* **Recommendation**: Continue, replan, gather more, or escalate and present with uncertainty. This is the field attention reads. A recommendation that lives only inside a paragraph never becomes a route.
* **Three failures die here if you let them**: Evidence-quality scoring catches the glossy miss. Overlap checks catch groove lock. A contradiction flag catches rigor mortis, once attention routes back to planning. Coverage bluff dies here only if low relevance is allowed to pull confidence down. A cheerful confidence write on empty retrieval keeps the bluff alive.

This is cousin to an offline judge and a grounding score. It runs in the loop, on the workspace, every hop, not only as a score after the user saw the answer. Offline eval tells you the system is regressing. This module is how this run refuses to lie.

## 9. Attention grants the next module

Attention is what makes the floor an architecture rather than a fixed pipeline. It reads signals and grants the next module control. Default order is perceive, then plan, then execute, then evaluate. Signals break that order.

```
  complexity low AND memory hit     --> fast path: execute or reply
  signal == CONTRADICTION           --> planning (old strategy stays in history)
  signal == STAGNATION              --> planning, with "do not repeat"
  signal == KNOWLEDGE_GAP           --> broader memory, then maybe exploratory goals
  signal == LOW_CONFIDENCE          --> a different source, or do not present
  else                              --> next step of the default cycle
```

* **The fast path is the latency win**: Familiar, cheap tickets should not tour every module. Log every fast-path decision. Sample them in the eval set. The failure is a contradictory ticket routed onto that path.
* **Where to debug**: Start in this decision and in the workspace flags. Wrong flags mean perception or evaluation is lying. Right flags and a wrong route mean attention is the bug. "Why did it paste the runbook?" is an attention question once evaluation has already written `CONTRADICTION`. If that flag is absent, evaluation never saw the conflict, and the route was never given a chance.

## 10. A graph of experience

Memory is the seventh piece. It is persistent structure, not the session transcript. The official MCP memory server (`@modelcontextprotocol/server-memory`) is a local knowledge graph: entities, directed relations in the active voice, and observations hanging off entities. This pattern does not need an extra vector database. It persists as local JSON. Here the graph is the agent's institutional scar tissue.

* **Problem types** are entities.
* **Strategies that worked or failed** are observations on those entities.
* **"X co-occurs with Y" and "tool combo Z"** are relations. Write the relation in the active voice ("VAT-inclusive carts co-occur with this webhook failure"), so a later retrieval reads like a fact a planner can use.

```
  entity: "VAT-inclusive webhook"
    observation: timeout increase already failed
    observation: candidate, unconfirmed — amount field vs exclusive field
  relation: co-occurs with "VAT-inclusive carts, two EU countries"
```

Over time this is a model of experience, not a chat dump. Write the observation evaluation actually supported. On this ticket that is "the timeout fix already failed," not a finished theory of the tax field. The next session can retrieve that scar before anyone plans "increase timeouts" again. A chat log of the session would not hand planning that fact. You would reread it and hope the model notices. The memory module is a proactive layer around those tools, not a call you hope the prompt remembers.

1. **Retrieve before planning** on hard work, or on a problem type that looks familiar. The first strategy should be allowed to see last quarter's scar. Skip this and every incident starts naive, including the ones you have already paid to understand. Connection-pool exhaustion retrieved as an observation is how "increase timeouts" loses the menu.
2. **Write after evaluation**, not after every noisy hop. Store the strategy and the outcome once they have been judged. A write after every hop stores tool noise as gospel. The next session retrieves the noise and repeats it.
3. **Broaden on a knowledge-gap signal** before you declare the topic outside coverage. A wider slice of the graph, or a second store, comes first. Declaring "outside" on the first empty lookup is how the boundary detector flags easy questions.

An organization-wide session store and a document index do a different job. They keep transcripts and documents alive. This graph is the agent's learned problem-solving memory. Same word. Different write path.

## 11. Several modules inside one outer iteration

Layer the cognitive cycle inside an outer task loop. The outer loop still owns the macro decision: keep going or stop, under a goal check, a budget, and a hard cap. The cycle owns the micro decision: what to do next inside this iteration.

```
  outer task loop          keep going / stop
        |
        |  each iteration
        v
  attention  <---- workspace signals
        |
        v
  perceive / remember / plan / execute / evaluate
  (evaluation may raise a signal and attention may
   run another module before the outer loop resumes)
```

* **Spend the signal inside the iteration**: Evaluation raises it. Attention routes to another module without returning to the outer loop yet. Only a clean step, with no live signal, hands control back for the convergence check.
* **One billed outer iteration can contain a pivot**: Perceive, plan, execute, evaluate, replan, execute again. You are paying so a shallow hop does not end as a glossy miss.
* **Termination still layers**: Hard cap, budget, goal, quality. Stagnation here is richer than an overlap test that only kills the run. It can fire inside the iteration and cause a pivot.
* **Two MCP servers in a working composition**: The memory server for the graph, and domain hands for the ticket (filesystem, search, internal APIs). Initialize an empty workspace, run a real query, and name each span as a module. If you cannot tell perception from evaluation in the trace, you assembled a blob.
* **Boring constructors**: One typed module each, one workspace, attention as control flow. Node and `npx` launch the servers. Python holds the agent and the keys. Clever recursion without a printed workspace is a deadlock with extra tokens.
* **What one outer iteration is allowed to cost**: Perceive, remember, plan, execute, evaluate, then a replan and a second execute, before the outer loop checks the budget again. That is the bill for refusing a glossy miss. It is not permission to hide an unbounded search inside "one" iteration. The outer hard cap and the dollar budget still wrap the whole thing. Name the spans `perceive`, `plan`, `execute`, `evaluate`, `attention`, so a pivot is visible without reading the prompt.

## 12. A ticket a flat loop gets plausibly wrong

Need a query a flat ReAct agent gets wrong in a way that sounds helpful.

> Checkout conversion dropped about 18% after we "fixed" the payment webhook. On-call already doubled retries and lengthened timeouts. It still fails, and only for VAT-inclusive carts in two EU countries.

A standard agent searches "payment webhook timeout conversion," finds the runbook already followed, and pastes it. The user said that path is dead. The answer is wrong by construction. A cycle that earns the extra modules looks like this.

1. **Perception**: Type is contradictory, because the standard fix was already applied. Complexity is high, the kind of score that refuses the fast path. Entities: webhook, retries, VAT-inclusive, country codes. Ambiguity stays open: "fails" might mean a decline or a client-side abort. Do not collapse that yet. Signal is still `NONE`.
2. **Memory first**: Attention sees high complexity and consults the graph before planning. The graph may already know "VAT-inclusive + webhook" from a prior incident, such as a tax-inclusive amount versus the processor's exclusive field. A miss is itself information: coverage is weak, and the first plan should not pretend otherwise.
3. **Planning**: The menu must not offer "increase timeout" as if it were new. Hypothesis-test. The failed fix is a negative constraint, written into rejected strategies. The candidate is payload shape or a tax field, not transport.
4. **Execution**: Search, and open the webhook payload schema, and compare a failing cart to a succeeding one. That order is composition: the schema read depends on what the search named. Findings are annotated. The runbook scores high relevance for "timeouts," with a quality note that the user already applied it. The schema finding scores high relevance for VAT. Raw JSON does not enter the workspace without those notes.
5. **Evaluation**: Findings contradict the popular answer. Confidence in "timeouts" drops. Confidence in "amount field" rises only as far as the evidence goes. Progress on the timeout hypothesis is reversed. Signal: `CONTRADICTION`.
6. **Attention**: Routes back to planning, not to the user. The new plan confirms with a second source, a recent deploy diff or the tax config. Still no present. This second plan is inside the same outer iteration. The outer loop has not been asked to stop.
7. **Gate**: Present only when confidence and the contradiction policy allow. If you are still on the edge, say so and show sources. Sounding sure is the glossy miss this cycle was built to refuse.

The win is refusing the runbook because the workspace recorded that it had already failed. There is no VAT oracle hiding in the prompt.

## 13. Three gates on structured state

Ship confidence gating first. Before anything user-visible, inspect workspace confidence, the number evaluation has been updating.

* **Below a hard floor (example: 0.3)**: Do not present. Signal uncertainty or escalate. Gathering more on a collapsed confidence often throws tokens at a knowledge boundary.
* **Between the floor and a soft bar (example: 0.6)**: Gather more if iteration budget remains. Otherwise uncertainty, not a fluent guess.
* **A live `CONTRADICTION`**: Gather or replan, even when the number looks acceptable. An average hides a fight.
* **A declining trend across hops**: A red flag. More of the same strategy is not the remedy.

This is a gate on structured state. "I think…" in the output is not a gate. You can unit-test it without a model: feed workspaces, assert the decision. Pair it with grounding when the answer must cite retrieved text. The gate answers "are we allowed to speak?" Grounding answers "did we speak only from sources?"

Three workspaces, three decisions, using the example thresholds:

* **Confidence 0.2**: Under the 0.3 floor. Do not present. Escalate or say you do not know. Another search on a collapsed score usually spends tokens at a boundary.
* **Confidence 0.45, budget left**: Between the floor and the 0.6 bar. Gather more. If the budget is already spent, uncertainty, not a fluent guess.
* **Confidence 0.8 with `CONTRADICTION` still open**: The number looks presentable. The signal overrides it. Gather or replan. An average of "timeouts" and "tax field" is how a fight becomes a smooth wrong answer.

Stagnation is an overlap and plateau problem. Groove lock lives here.

* **Content overlap**: Token-set Jaccard, or a cheaper overlap ratio, between the last two findings. High overlap means you did not move. Identical queries in the history are the same fact, cheaper to see.
* **Confidence plateau**: The last few confidence values sit in a tiny band. The agent is busy and unchanged. A flat 0.62 across three hops is a plateau, not a mild success.
* **On fire**: Set `STAGNATION`. Record why, overlap versus plateau. Route to planning with an explicit constraint: do not repeat the failed strategy. Write the failure into memory so the next session does not buy the same groove. Planning must receive that constraint. A flag nobody reads is a metric.
* **The outer cap stays**: Stagnation spends fewer of those iterations on the same wall. It does not replace the hard limit. A pivot every hop, with no overlap test, is thrashing under a new name.

Coverage is the bluff patch: know when you are outside. Blend a few cheap signals rather than one magic score. Average relevance of findings, or 0 when there are none. Whether memory had hits for this problem type. Current workspace confidence. Map the blend onto within, edge, or outside. Outside raises `LOW_CONFIDENCE`, or a dedicated gap flag, and changes behavior.

1. **Within**: The normal present path, still gated.
2. **Edge**: Present with sources and explicit limits, or one more gather from a different class of tool.
3. **Outside**: Say the store and the graph do not cover this. Ask a clarifying question, or escalate. Do not complete the procedure from parametric memory while wearing a source-shaped hat.

False positives, flagging an easy question as outside, annoy users. False negatives, a fluent procedure on empty retrieval, create incidents. Measure the pair. That pair is the knowledge-boundary accuracy number below.

### Watch for

* A new strategy every hop, with no overlap test. That is thrashing.
* `STAGNATION` written and the same query issued again. Detection without a pivot waits for `max_iterations`.
* A sourced-looking procedure whose findings have relevance 0. Parametric memory is wearing a citation.

## 14. Behaviors that fall out of the routing

Four behaviors you should not special-case as extra `if` statements, if the modules and the graph are doing their jobs. They show up from interaction. That is the society-of-specialists bet.

1. **Curiosity**: Low confidence plus no memory hits raises a knowledge-gap signal. Memory broadens. If it is still empty, planning adds exploratory subgoals the user did not name: adjacent config, the last deploy, a second store. From the outside it looks like the agent wondered. Inside it is a flag and a route.
2. **Adaptive persistence**: Execution fails. Stagnation fires. Attention sends planning a "do not repeat" constraint. Memory records the miss. Next time that problem type appears, planning starts somewhere else. That is learning from failure without a training job.
3. **Selective depth**: Fast path on easy and familiar. Full cycle on contradictory and novel. The same binary, a different token bill. If everything takes the full cycle, attention is shy. If everything takes the fast path, attention is lying.
4. **Composition under constraints**: Evaluation refuses a shallow one-tool answer on a compositional type. Planning is forced to name a dependency order. Execution runs that order. You did not hard-code "always search, then open, then compute." You made a shallow answer fail a check.

If they do not show up, do not add a curiosity prompt. Inspect three things. Did evaluation raise the signal? Did attention take the route that signal names? Did memory actually write an outcome after evaluation, or is the graph still empty after dozens of tickets? A prompt that says "be curious" imitates the behavior and leaves those three questions unanswerable.

## 15. Probes and four numbers

Not a leaderboard. A diagnostic on your agent, aimed at the same five messes. Steal the probes. Write them as fixtures. The test is the requirement.

| Failure | Probe | Pass looks like | Open |
|---|---|---|---|
| Glossy miss | Top hit is a table of contents or an index page | Junk evidence is flagged. Navigation is not cited as procedure | Evaluation scores |
| Groove lock | A query with no good hits | A pivot within a few hops, not identical retries | Attention and overlap |
| Rigor mortis | Mid-run, the user says the first plan failed | Strategy changes. The old plan is in history as rejected | Planning and signals |
| Coverage bluff | An empty store, on purpose | Uncertainty or escalate, not a fluent invention | Boundary and the gate |
| Tool salad | Search, then the file it named, then a computation | An ordered composition, not three unrelated calls | The planning graph |

Walk the probes as fixtures, not as slogans.

* **Glossy miss**: The top hit is an index page or a table of contents. Pass when the quality note calls it navigation and the reply does not cite it as the procedure. Fail when the heading is quoted with high confidence.
* **Groove lock**: A query you already know has no good hit. Pass when the strategy changes within a few hops and history records the repeated query. Fail when the tool calls stay identical until the cap.
* **Rigor mortis**: Mid-run, the user says the first plan already failed. Pass when that strategy is stored as rejected and the next action is a different one. Fail when the checklist finishes anyway.
* **Coverage bluff**: The store is empty on purpose. Pass when the agent signals uncertainty or escalates. Fail when it invents a procedure and sounds sourced.
* **Tool salad**: The task needs search, then the file that search named, then a computation on the file. Pass when the plan has those edges and execution follows them. Fail when the three calls are unrelated.

Four production numbers, tracked on traces.

* **Cognitive efficiency**: Steps taken versus a sane minimum for that task type. A lookup that tours six modules is a fast-path bug. Attention was shy. A contradictory incident that exits in one hop is an evaluation bug. The runbook was allowed to win.
* **Metacognitive calibration**: Plot workspace confidence against later correctness, from a human or a judge. A well-calibrated agent is roughly diagonal. High confidence on wrong answers is the glossy miss in aggregate. Low confidence on easy wins means the boundary detector is flagging questions that were inside coverage.
* **Adaptation rate**: Hops from the stagnation signal to a strategy that actually moves the findings. Lower is better. Infinite means the flag fired and planning repeated the failed strategy anyway.
* **Knowledge-boundary accuracy**: False positives are uncertainty theater on easy questions. False negatives are a hallucinated procedure delivered with a straight face. Optimize the pair. Driving one rate to zero inflates the other.

Run the same probe set on three compositions.

1. **Flat ReAct**: One loop, one primitive, no workspace. The baseline the probes should beat. On the checkout ticket it pastes the runbook.
2. **Cognitive, empty graph**: The modules run, and memory has no scars yet. You are paying for routing without experience. Familiar types are not short yet. This is the first day.
3. **Cognitive, scarred graph**: The same composition after a few dozen completed tickets. Observations are outcomes. Experience should raise calibration, shorten familiar types, and cut stagnation events.

Compare correctness, calibration, graceful degradation (how often uncertainty was required and was actually signaled), and tokens per solved ticket. The plot that matters is empty versus scarred. If the scarred graph is worse, noisy hops were stored as gospel. Hygiene still applies: write outcomes, not transcripts.

The number worth celebrating is fewer incidents per thousand tickets at similar spend. A token spike excused with "but calibration" is not that number.

## 16. Keep the floor, swap the specialists

The lab modules are narrow: a task-type list, a couple of servers, a menu of strategy names. The skeleton is not narrow. Workspace, attention, evaluation, gates, and graph memory stay. A wider agent comes from swapping specialists and keeping the floor: perception's taxonomy, execution's MCP hands, planning's menu. A bigger model does not grow a manager by itself.

Move the same floor from incident response to internal policy Q&A. Replace perception's types: a policy question is ambiguous in a different way than a webhook. Replace execution's hands: the policy index, not the payload schema. Replace planning's menu: cite the clause, rather than hypothesis-testing a timeout. Keep the workspace slots, the attention router, the evaluation write, the confidence gate, the contradiction override, and the graph of which question-types the agent has already botched. The example floor of 0.3 is a policy you retune. The fact of a floor is what you keep.

Generalization here is architectural. You compose skills you can monitor. This lab agent will not ace a compositional benchmark. It points you at composition and at monitoring, which is the requirement a longer system prompt does not meet. The four numbers in section 15 tell you whether the extra modules pay rent. They are not a leaderboard. Fewer incidents per thousand tickets, at a similar spend, is the rent. A token spike excused with "but calibration" means the floor is not paying it.

## Check yourself

1. A stakeholder says the agent is cognitive now because it has chain-of-thought, ReAct, and Reflexion. Which dispatcher is still missing, and which failure mode stays likely?
2. Pick a real incident from a system you know. Tag it as glossy miss, groove lock, rigor mortis, coverage bluff, or tool salad. Which module would you inspect first, and which workspace field would you print?
3. In one sentence each, define cognition and metacognition as engineering terms. Name one test you could run this week for each.
4. Why is "I'm not sure" in the output not confidence calibration? What would you plot instead?
5. Map Minsky, Baars, and Kahneman onto module, workspace, and router. What bug appears if you build two of the three and skip one?
6. When should attention take the fast path, and which log would convince you it misfired?
7. How does the cognitive cycle sit inside one outer task-loop iteration without replacing that loop's termination gate?
8. Sketch the confidence-gate decision for confidence 0.2, for confidence 0.45 with budget left, and for confidence 0.8 with an open contradiction signal.
9. Stagnation fires. What must planning receive that it did not have on the first attempt, and what should memory write before the session ends?
10. You move this architecture from incident response to internal policy Q&A. Which pieces do you replace, and which do you keep?
