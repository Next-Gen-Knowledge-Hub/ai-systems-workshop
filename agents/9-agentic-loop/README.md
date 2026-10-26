# 9. The agentic loop

Companion notes for **Chapter 9** of *AI Agents in Action* (2nd edition, Micheal Lanham; Manning, 2026).

This chapter establishes iteration as a control system with three nested loops. One sense–plan–act–learn call can finish a short job. It cannot keep a diligence brief across twenty searches, or drain a queue of invoices overnight, once the context window fills. Afterward you can name which loop thinks, which loop stores the campaign, which loop may change the campaign, and which predicate actually stopped the run.

## 1. Three loops, three jobs

Mixing the names is how a design review thrashes. Each layer has a different owner and a different horizon.

```
  META LOOP     an agent holds decide / plan / state
                orchestration (one hub) or collaboration (peers)
                chooses the next long-horizon move
                     |
                     v
  TASK LOOP     code holds plan + state across iterations
                research-until-done, or queue-until-empty
                     |
                     v
  INNER LOOP    sense, plan, act, learn
                until a short goal, or until the caller stops it
```

* **Inner loop**: The model thinks. It senses, plans, acts, and learns toward a short goal. One cycle can answer "search this error code."
* **Task loop**: Your code remembers the campaign. Goal, plan, state, and the stop decision live outside the model so they survive a full window and a process restart. A diligence brief lives here: subtopics, findings, and a gate.
* **Meta loop**: An agent changes the campaign. It sits on the decide knob. More processes, more MCP servers, or a fancier Compose file do not promote you. If Python still owns every branch, you are on the task loop, which is often what you wanted.

A `while` around `Runner.run()` is not a task loop by itself. The task loop externalizes **goal, plan, state, and the stop decision**. The meta loop is an agent, or a team of peers, on that same knob.

Short work can live in one prompt. "Search this error code" may finish in one cycle. The plan is this turn, and stop means the answer came back. Long work keeps the plan in a `ResearchPlan` or a queue. "Brief me on vendor X for diligence" has subtopics, and state accumulates across searches. Stop means a layered gate. General-purpose models handle the short column. They re-search the first query and declare victory on the long column unless the cabinet is outside the model.

## 2. The inner loop

The inner loop is the short-horizon engine. Outer layers wrap it. They do not replace it.

```
  goal for this call
       |
       v
  SENSE  -->  PLAN  -->  ACT  -->  LEARN
    ^                              |
    |         continue             |
    +------------------------------+
                  |
                  +--> goal met, or the caller budget ends
```

* **Sense**: Read the goal for this call, the last observation, and the memory slice you were handed.
* **Plan**: Choose the next action that maps onto a tool, or onto a reply the user can see.
* **Act**: The runtime executes the tool, or speaks. The model names the call.
* **Learn**: Read the observation. Continue, replan, or stop. A sold-out result belongs here because the world answered.
* **What makes the cycle agentic**: The model chooses the next tool, judges whether the observation is good enough, and, inside bounds you set, when to stop. You still own the goal, the tool list, and the hard fences. You do not click each step.
* **State inside the cycle is short-lived**: Tool results land back in the thread. A scratch plan lives in the prompt. That state dies when the call ends, the window fills, or the process restarts. Fine for "look up the weather and summarize." Useless as the only memory for two hours and twenty searches.
* **Reasoning patterns live here**: ReAct, chain-of-thought, and Reflexion make one cycle smarter. They do not, by themselves, persist a twelve-item plan overnight. Pasting the whole plan into the system prompt every turn pays the task-loop cost without the task-loop structure.
* **A wall-clock budget**: An inner loop with no timeout is a runaway worker.

## 3. The task loop

The task loop externalizes whatever must outlive one inner cycle.

* **Goal**: Still one sentence the user cares about.
* **Plan**: Subtopics, statuses, strategy notes. Or, for a batch, the queue itself.
* **State**: Findings, sources, per-item results, errors.
* **Stop decision**: Whether to iterate again. A number your code can read. "3/7 subtopics complete" can gate. A vibe in the chat log cannot.

Those objects live in your process: a Pydantic model, a database row, a Redis key. Each iteration still runs the inner loop, but you hand it a **compressed view** of the campaign, and it must return a **typed delta** the controller merges.

```
  ResearchPlan / TaskQueue  ---- persist
  ResearchState / results   ---- persist
           |
           v
  pack to_context()
           |
           v
  inner agent (one short SPAL run)
           |
           v
  typed iteration output --> merge --> termination gate
                                         |
                                         +--> continue, or synthesize
```

A model told to "keep going," with no external plan, re-searches the first query, contradicts itself, and declares victory. External state is how progress becomes visible to code.

Two shapes share the controller and hide different objects if you collapse them.

| Shape | State | Plan |
|---|---|---|
| Deep research | Growing findings and sources | Subtopics marked pending, in progress, or complete |
| Repetitive tasks | A queue plus per-item results | Often unnecessary. The queue *is* the work |

Research explores. The task loop consumes a list you already have. Same controller pattern. Different objects.

**Deep**, on a research button, is an engineering word. It means more search, more reading, and a longer horizon than one retrieval hop. A reliable researcher and a token incinerator diverge on three choices: how you manage state between iterations, how you terminate, and how you attach tools so the agent can look at the world. Eval still scores the report. The controller scores whether to stop writing it.

Between iterations, compress on purpose. Completed subtopics can drop their raw notes from the next view. If you only ever append, iteration 12 is a transcript with a schema wrapped around it, and the gate is guessing from a flooded window.

## 4. Plan, state, and the view

Before the first `Runner.run()`, build two objects the loop will pass around. You can rename them. Do not skip the fields.

```
  ResearchPlan                      ResearchState
  subtopics[]                       goal
    name / status / notes           findings[]  claim, source, subtopic
  strategy_notes                    sources_seen[]
  to_context()                      iteration, spend, errors[]
  progress_summary
```

**Plan** is strategic, coarse, and revisable.

* **Subtopics**: Each has `name`, `status` (`pending`, `in_progress`, or `complete`), and `notes`.
* **strategy_notes**: "Search academic sources first." "Ignore marketing blogs."
* **to_context()**: The view the model is allowed to see this iteration.
* **Progress summary**: A line such as `2/5 complete, 1 in progress`, for logs and for the gate.

**State** is the accumulating evidence, not the strategy.

* **Goal text**, plus findings as claim + source + subtopic.
* **Iteration index**, token or dollar counters, and the last error.
* **sources_seen**: URLs already visited, so the same hit does not become the next plan. Anything you refuse to trust the model to remember belongs here.

Stuffing both objects wholesale into every prompt is the usual trap. `to_context()` is a view. As the campaign grows you truncate, summarize, or drop the raw notes of completed subtopics. The task loop exists so you control the window on purpose. If iteration 12 still contains iteration 1's raw HTML, you cloned the transcript into a Pydantic field.

On iteration zero the plan may be empty. The agent's first job is then to **write the subtopics** before it pretends to search. Put that in the instructions. An agent that searches with no plan searches whatever the temperature likes.

The synthesis step is a different persona on purpose. Exploring and editing are two jobs. One persona does both, and the last search result overwrites the outline. The controller runs the researcher until the gate fires, then hands plan plus state to the synthesizer. That handoff is below. The objects above are what make the handoff possible.

## 5. Tools that ground the iteration

After the objects exist, attach tools. Observations have to come from a tool result. A fluent paragraph is not a source.

* **MCP search over STDIO**: The lab shape is `create_search_server()`, with the vendor key in the environment. Brave is the example. Any search MCP server is the same shape. Node and `npx` show up because those servers are often Node packages.
* **A native tool**: `@function_tool` search against a local corpus is the same contract. The loop does not care which packaging you used.
* **Several sources**: Web search, an internal wiki MCP server, a SQL tool. Normal. A shell "so it can curl anything" is a sandbox failure wearing a research excuse. The observation that grounds a finding is the tool result, not a paragraph the model wrote about what it thinks the page said.
* **Lifecycle**: Connect the MCP server, run the loop, disconnect. An async context manager exists so you do not leak Node processes. A loop that forgets to close STDIO servers looks fine until the next exercise cannot bind a port.
* **A model that can call tools**: A cheap model that cannot emit valid tool JSON spins the controller and never fills `findings`. Pick a model that can tool-call, then keep the gate. That is routing, not a reason to delete the controller.

## 6. A typed delta

Each cycle the inner agent produces work. The controller must parse it. Free-form markdown is a blog post, not a control signal. Force a structured output:

```
  ResearchIteration
    summary_of_what_I_did
    findings[]                 claim, source, subtopic tag
    subtopic_updates[]         name, status, notes
    follow_up_questions[]
    goal_satisfied: bool
    confidence: float          0-1, a claim, not a measurement
    notes_for_next_iteration
```

After `Runner.run()` the controller does four things, in order.

1. **Validate**: Reject extra fields. A tool-shaped blob from the model is still untrusted structure.
2. **Merge findings** into `ResearchState`, and dedupe URLs.
3. **Apply subtopic updates** with rules you own. Status does not jump to `complete` without a source if that is your bar.
4. **Persist**, then ask the gate.

A parse failure retries or counts as an error iteration. It does not become a silent success.

`goal_satisfied=True` is one vote. Self-assessment is how a loop escapes the homework: the model eventually votes yes to get out. A second goal-or-quality agent may cast another vote if the eval budget can stand it. That vote is still not a proof.

## 7. A stack of stop conditions

One predicate is how loops either never start or never end. Keep a stack. Always keep the first two.

```
  after merge:
    iterations >= MAX              --> STOP_MAX
    spend >= BUDGET                --> STOP_BUDGET
    stagnant for N iterations      --> STOP_STAGNATION
    plan done and quality ok       --> STOP_GOAL
    else                           --> CONTINUE
```

1. **Hard max iterations**: The fuse. Set it from how long a *good* report takes in your domain, then add a little. Do not set it from the largest number that still feels lucky. Checked first.
2. **Budget**: Tokens or dollars. The fuse finance understands. If you keep blowing it, shrink per-iteration context. Raising the cap is the other move, and it hides the window problem. A goal-yes that arrives after you are over budget is still `STOP_BUDGET`.
3. **Stagnation**: N iterations with no new information. Checked before anyone may claim the goal, so a paraphrasing loop cannot vote itself done. Practical tests: URL overlap against `sources_seen`, embedding similarity of the new findings against the last batch, or `follow_up_questions` that did not change. Two iterations of near-duplicate text should stop even if the agent wants another search. This predicate ends the run. It does not, by itself, switch strategy and keep going.
4. **Goal satisfied**: The happy path, and only after the fuses. If only the researching agent votes, it will vote yes to finish. Require `complete` on every subtopic, require N independent sources, or ask a separate judge.
5. **Quality threshold**: For example, confidence at or above 0.85, owned by an external scorer. In the sketch this is the `quality_ok` half of `STOP_GOAL` (`plan.done and quality_ok`), not an earlier exit that beats max, budget, or stagnation. The researching agent's own float is the same trap as a self-reported goal.

Log the reason. A Phoenix span that shows *why* the loop ended is the difference between a gate and a ghost.

### Watch for

* Treating `confidence` as a measured metric because the field is a float.
* A goal-yes logged as `STOP_GOAL` after the budget already tripped.
* Raising `MAX` every time the loop feels unfinished, until the fuse is decorative.

## 8. The controller is boring

The controller is a for-loop you should be able to recite. Cleverness belongs in the gate, not in hidden branches.

```
  plan, state = init(goal)
  mcp = await connect_search()
  agent = build_researcher(mcp)

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

The inner agent's instructions have to say the situation in words.

* **You are inside an external loop.** This call is one iteration, not the whole campaign.
* **Empty plan on iteration 1**: Emit the subtopics. Do not search yet.
* **Later iterations**: Pick incomplete subtopics, search, update statuses. Do not reopen a completed subtopic without a reason.
* **Fill the typed schema.** A parallel essay is ignored by the controller.
* **`goal_satisfied` is a proposal.** The controller may ignore it.

A prompt that pretends this is a one-shot chat makes the model write the final report on iteration 1 and then stall. Keep the researcher off formatting. Headings, the executive summary, and "gaps and limitations" belong to synthesis. A half-report at iteration 3 is a document the loop becomes afraid to throw away.

Even a lab version of this function carries a per-iteration timeout, a cost counter, and a structured log line with `iteration` and `gate_reason`.

## 9. Synthesis is a second persona

When the gate fires you have a bag of findings. They overlap, they disagree, and they have holes. A separate synthesis agent turns the bag into a report. It does not explore.

```
  ResearchReport
    title
    executive_summary
    key_findings[]
    sources[]
    confidence_assessment
    gaps_and_limitations
```

* **Outline from the plan**: The subtopics are the section structure. The synthesizer does not invent a new table of contents that hides an unfinished subtopic.
* **Contradictions stay visible**: Surface them. Averaging two sources into one smooth sentence deletes the disagreement the researcher paid to find.
* **Sources and gaps**: List sources. State what you did not find. `gaps_and_limitations` is how the report stays honest when the gate was `STOP_BUDGET` rather than `STOP_GOAL`.
* **A tiny second loop, if it must search**: Another task-loop cycle with a small `MAX`. An unbounded encore that "fills gaps" is a nested research loop with no fuse. You see it in the trace as search spans after the research gate has already logged a `gate_reason`, with no `iteration` cap of their own. When the synthesizer was itself delegated, `parent_turn_id` ties that encore to the outer turn.

| | Researcher | Synthesizer |
|---|---|---|
| How often | Many iterations | Once |
| Job | Explore, update the plan | Write the report |
| Output | Typed deltas, allowed to be messy | Typed report, required to be structured |
| Search | The point of the loop | Little or none, and gated if any |

## 10. When a loop earns its cost

A loop on a one-shot task buys latency and cost. Use a task loop when at least one of these is true.

1. **Iterative discovery**: You cannot know the sources in advance. Research, investigation, "what is going on in this codebase."
2. **Progressive refinement**: Drafts get better. Writing, or a design you critique and revise.
3. **Batch processing**: A queue of similar items. The next section.
4. **Conditional branching**: The next tool depends on the last observation. Diagnostics and troubleshooting.
5. **Multisource aggregation**: The relevant sources appear during execution, not in the original prompt.

Refuse a loop when a single pass is the product. Classification. Short Q&A against a known index. Format conversion. "Extract the order id." Those want one tool call and a schema.

```
  known answer shape, known tools, a cheap mistake?
      yes --> one inner call (ReAct once, maybe)
      no  --> is the missing piece time and evidence?
                yes --> task loop
                no  --> a better tool or a better index
```

If the failure is a wrong tool schema or bad retrieval, a loop repeats the failure. Fix the tool and the knowledge path first.

## 11. A queue of similar items

The second task-loop shape is not exploration. You already have the work: invoices to post, documents to transform, endpoints to test, records to migrate. The external object is a queue, not a research plan.

```
  TaskState
    queue[]          pending items
    in_progress
    done[]           item, result, attempts
    failed[]         item, error, attempts
```

* **Pull**: The next pending item, or a failed item whose attempt count is under the cap. The controller chooses. The model does not get to skip `invoice-441` because the paragraph would read better without it.
* **Process**: The inner agent runs sense–plan–act–learn on that item only. Tools post, transform, or test. One item's context is not the whole queue pasted into the prompt.
* **Record**: Success appends `{item, result, attempts}` to `done`. Failure increments the count or lands `{item, error, attempts}` in `failed`.
* **Stop**: Queue empty, max attempts, or budget. The summary is those counts and the failed rows.
* **The list is the plan**: Subtopics on a homogeneous invoice queue add a strategy object the work does not need. The controller's progress line is counts: done, failed, remaining.
* **Retries live in the controller**: Do not hope the model remembers it already tried `invoice-441`. At-least-once queues make that hope a double post. Pair the retry with the idempotency key on the side effect.
* **Failed items stay visible**: A cheerful summary paragraph is not a place to hide `failed[]`.
* **When this slides into the meta loop**: The items are heterogeneous, and choosing which item is itself a reasoning problem. An orchestrator is then picking the next worker. Keep this shape for homogeneous batches.

## 12. An agent on the decide knob

The task loop's controller is code. The meta loop puts an **agent** on decide, plan, and state. That is the promotion. It forks by who that agent is.

1. **Orchestration**: One hub agent. Workers are tools. The hub writes the plan and votes stop. You keep one mouth. You pay a bottleneck and a context hog. Use it when the next subtask can change because a worker returned a surprise, and you still want a single owner of the campaign.
2. **Collaboration**: Peers, with an optional turn manager. They share state. The plan may be jointly owned. Stop may be a vote or a chair. Use it when the product is multiple viewpoints: debate, a red team beside a builder, style beside factuality. It breaks on a queue of invoices, which wanted the task loop.

A flow is the ordered cousin, and it is not either of these loops. Stages run in order (planner, then researcher, then writer). Easy to test stage by stage. Brittle when stage 1 is wrong, because later stages do not get a round to revisit it. A collaboration loop can revisit, and you pay in coordination and in traces. A hub that assigns turns while workers cannot see each other is orchestration. Call it that.

## 13. The hub delegates

A single orchestrator keeps the global plan and state. Worker agents — researcher, coder, writer, critic — are tools from the hub's point of view. Each delegation is an inner loop, or a nested task loop. The hub's own cycle is: sense the campaign, plan which worker to call, act by delegating, learn from the worker's typed result.

```
  ORCHESTRATOR (plan + state + gate)
        |
        +--> worker A as a tool
        +--> worker B as a tool
        +--> worker C as a tool
        |
        v
  merge typed results
  hub decides: another round, or finalize
```

* **What you buy**: The next subtask can change when a worker returns a surprise. "This vendor is a subsidiary; research the parent." A hardcoded `for subtopic in plan` cannot notice that unless you encoded the rule.
* **Typed, small returns**: The hub is a bottleneck and a context hog. Novel-length dumps blow the window. Timeouts and budgets apply per delegation and to the meta loop as a whole. Two fuses.
* **Hub instructions**: Name the workers as tools. Forbid the hub from doing the specialist work itself. Require an explicit field you parse, in the spirit of `need_another_round` or `final_output`. A hub that both researches and orchestrates starves the workers and fills the window with its own notes.
* **Nested cost**: A hub that delegates to a research worker, which runs its own task loop, multiplies the hub's `MAX` by the worker's `MAX`. Set both. Trace both, with `parent_turn_id` on the inner spans.

## 14. Peers on a blackboard

Collaboration is the other meta subtype. Agents share a state object and take turns contributing, critiquing, and revising. A lightweight turn manager may pick who speaks. It should not secretly be an orchestrator with extra adjectives.

* **Blackboard**: Named slots for the draft, findings, critiques, and votes. Modules read and write that object. Isolated agents that cannot read it are a slow flow: stages that happen to be called in a loop.
* **A common triad**: The researcher adds evidence. The critic challenges and asks for sources. The synthesizer updates the narrative. Each sees the others' last writes, so the round is cumulative.
* **Stop conditions still layer**: Max rounds. Budget. Unanimous, or chair, "good enough." Stagnation, when the critic repeats the same complaint and the researcher cannot answer. A collaboration with no fuse is a meeting that invoices by the token.
* **Score the artifact**: Traces get harder because speech is interleaved. Score the shared draft and the critiques, not only each speaker's prose.
* **Slot rights are code**: The critic cannot silently mark a subtopic complete. That is behavioral policy on the blackboard, enforced where the write happens.
* **Wrong product for this subtype**: A due-diligence memo can use the triad, because the product is a draft that survives critique. An invoice batch cannot. Peers debating each invoice buy a meeting. The queue already knew the next item. A hub that assigns turns, while workers cannot read each other's slots, is orchestration with extra steps. Draw it as a hub.

## 15. What these loops do not monitor

The three layers answer who thinks, who stores the campaign, and who may change the campaign. They do not build a working model of how the reasoning itself is going.

* **A stop is not a mid-course monitor**: Stagnation here compares URLs, embeddings, or follow-up questions, then ends the run. Two near-duplicate iterations become `STOP_STAGNATION`. Nothing in this stack then picks a different tool strategy and continues the same iteration. The overlap test is a fuse.
* **A confidence field is a vote you chose to read**: The schema carries a float from 0 to 1. A threshold such as 0.85 can shortcut the exit when an external scorer owns the number. The researching agent's own float does not become a measurement because you stored it. "I am confident" in the summary is the same kind of claim.
* **Findings are not rejected strategies**: `ResearchState` remembers claims, sources, and visited URLs. It does not require a slot for "raise the timeout; already failed." The controller does not route back to planning because two findings contradict each other. If you want that route, it is extra machinery these three layers do not include.

Name the layer you shipped. A `while` that cannot say whether code or an agent owns the next campaign move will thrash. The overlap test, the budget fuse, and the typed delta are the machinery this chapter actually gives you.

## Check yourself

1. A stakeholder says the system has an agentic loop because `Runner.run()` sits in `while True`. Which layer is that, and which objects would have to live outside the model before you would call it a task loop?
2. Walk a goal you actually want, research or a batch of tickets, through the three layers. Which layer's stop condition is a number in your code, and which is a model's opinion?
3. Why does `to_context()` exist if you already stored the full `ResearchState`? What goes wrong on iteration 15 if you skip the view?
4. List the five gate predicates in an order you would ship. Where does a self-reported `goal_satisfied=True` sit, and how can that vote cheat?
5. Typed `ResearchIteration` versus a markdown essay: name two fields the controller cannot live without, and one failure if `confidence` is treated as a measured metric.
6. When would you refuse a loop entirely? Give a one-shot task from a system you know, and name the failure a loop would only repeat.
7. Research loop versus repetitive task loop: which external object is the plan, and why would subtopics on a homogeneous invoice queue be a mistake?
8. Code as controller, versus a hub: a worker discovers the company is a subsidiary. Which layer can change the plan without a human editing the `for` loop, and which budget fuse did you just multiply?
9. Orchestration versus collaboration: pick support triage, a due-diligence memo, or an invoice batch. Which subtype fits, and what goes wrong with the other?
10. A synthesizer starts calling search "just to fill gaps," with no `MAX`. Which loop did you accidentally nest, and which trace fields would have shown the encore?
