# 5. Reasoning and planning

Companion notes for **Chapter 5** of *AI Agents in Action* (2nd edition, Micheal Lanham; Manning, 2026).

This chapter is where an agent spends tokens before it acts. An LLM predicts the next token. It does not hold a project plan between calls. Decomposition and planning fail on different traces, and a thought can live in a forward pass, a visible chain, a tool loop, a stored plan, a search, or a critique cycle. After this chapter you can name which of those you shipped, and what that choice cannot do.

## 1. Decomposition and planning

**Reasoning**, for someone building an agent, is two operations. They fail separately, so the fix is different.

* **Decomposition**: What are the pieces of this goal? "Onboard a vendor" can become five sensible pieces, and the split can still be the wrong split.
* **Planning**: In what order, with which tools, and how do the pieces join? Emailing legal before security has signed is a planning failure on a decent decomposition. A beautiful sequence over the wrong pieces is the other failure.
* **No persistent world**: If the plan is not in the prompt, a tool result, or a store you control, it does not exist on the next call. The model will not "still be thinking about it."

```
GOAL
  |-- DECOMPOSE    pieces
  |-- PLAN         order, tools, joins
  v
WHERE THE THOUGHT LIVES
  forward pass      hidden deliberation inside one call
  completion        CoT: a linear chain of text
  tool loop         ReAct: thought, act, observe
  external state    plan object, scratchpad, blackboard
  search            ToT: branch, score, prune
  critique cycle    Reflexion: try, judge, retry
```

Name which of those six the trace is actually using. They are not synonyms.

* **Forward pass**: Deliberation inside one call, often hidden. Use it when the hop is short and a wrong act is cheap. It breaks when a later step had to see a tool result.
* **Completion**: Chain-of-thought. A linear chain of text you can read. Use it for a middle step that is logic, not a lookup. It breaks as soon as the world has to answer.
* **Tool loop**: ReAct. Thought, action, observation, and the observation may change the next thought. Use it for live data. It breaks when the job needed a join you never wrote down.
* **External state**: A plan object, a scratchpad, or a blackboard slot. This is how a plan exists on the next call. It breaks if nothing writes it.
* **Search**: Tree-of-thought. Branches, scores, pruning. Use it when you can judge a candidate. It breaks when you cannot, because the fan-out is just spend.
* **Critique cycle**: Reflexion. Try, judge, retry, with the critique in the next prompt. Weights do not change. It breaks if the attempt has a side effect.

* **Four products**: A reasoning model, a chain-of-thought prompt, a ReAct loop, and a sequential-thinking server spend different tokens. Mixing the names in an incident write-up hides which tokens you paid for.
* **Two different traces**: A correct split in the wrong order looks like emails sent too early. A lovely order over the wrong pieces looks like a plan that never mentioned security. You can tell them apart in the log. You cannot tell them apart if you only write "the model cannot reason."
* **Structure is a cost**: A single-tool lookup does not want a tree. A long research job does not want one forward pass. Adding a chain, a tree, and a scratchpad in the same deploy teaches you nothing, because you cannot see which operation moved.

Add an explicit method when one of these is true. Skip it when none of them are.

1. **Long horizon**: Many steps, and a later step depends on an earlier result. A single completion will skip the middle. It breaks the budget if you also turn it on for a one-line lookup.
2. **Many tools compete**: The model can narrate a plausible tool instead of calling the right one. Name the order, or split the tools. A tree on top of twelve overlapping descriptions does not fix the descriptions.
3. **Expensive mistakes**: The act is hard to undo. You want a plan you can read before the act, and a trace after it. Extra structure on a reversible read is latency.
4. **An audit trail**: Someone later has to see why a step ran. A hidden deliberation you do not store is not that trail.
5. **Outside the model's comfort**: The domain is full of local constraints the weights do not reliably keep. A short instruction that already succeeds should stay a short instruction. A strong model on a reversible act can stay on the forward pass.

## 2. Single-pass models and reasoning models

People leave the house without writing a four-step plan. They have routines and a body. The model has neither, so work a person would do on autopilot still needs an explicit split.

* **No inner simulator**: Nothing in the model already knows the toaster is off. A fluent sentence about leaving the house can skip the step a body would not skip.

1. **Non-reasoning model**: One forward pass, then the answer. Without scaffolding it short-circuits into a plausible sentence that skipped the middle step. The limit is the quality of multi-step structure, not willingness. It will attempt a twelve-step goal in one breath. That is a long completion, not a plan. Use it when the hop is one fact and a wrong act is cheap. It breaks when the nasty step sits in the middle and nothing forces the sampler to spend tokens there.
2. **Reasoning model**: Extra compute on a deliberation phase, sometimes hidden and sometimes a visible chain, before the user-facing tokens. You still own grounding (did a tool get called?) and storage (is there a plan object?). Use the extra deliberation on the few steps that are actually hard. It breaks if you treat the hidden chain as a typed plan, or as an audit trail you did not retain.

* **Still a guess without a tool**: A thinking model that never calls search is deliberating over the weights. The trace you want is a tool span, not a longer private chain.
* **A 4k chain is a leak**: Dumping that working text into the next hop ships policy and half-formed paths to a persona that did not need them. Store a short plan object instead.

### Watch for

* Deleting the planner because a thinking model is on. Native deliberation is reasoning inside one call. It does not hand you a typed plan, a tool loop, or a scratchpad that survives the next hop. Keep the architecture.
* Effort left high on every hop. Turn native deliberation down on routine tool calls and up on the few steps that are actually hard.
* A hidden chain you cannot parse. Visible and parseable beats hidden, unless you actually keep the vendor trace.

## 3. Chain-of-thought

**Chain-of-thought (CoT)** asks the model to emit intermediate steps in the completion, then the answer. You are buying visible working. It is still one pass, or one pass plus hidden thinking. Nothing in the environment can contradict step 3 unless a loop also exists.

* **What it buys**: The sampler spends tokens on arithmetic and local logic instead of skipping to a fluent ending. Use it when the error sits in the middle of a dependency chain: dates, unit conversions, "if A then not B."
* **A middle step you can name**: The outage started on Friday, the credit is two business days later, and Monday is a holiday. The wrong date is produced in the middle. CoT is for that chain. "Format this JSON" has no middle step to get wrong, and the chain is only spend.
* **What it cannot recover**: A wrong tool observation. There is no observation. Finding whether a page exists needs a tool. CoT will narrate the lookup.
* **What it cannot explore**: Alternative branches. That is tree-of-thought, or several sampled completions. One chain is one path.
* **What it cannot remember**: The plan on the next runner call. Pasting the chain into the next prompt leaks working text. Store a plan object.
* **When it is a tax**: Retrieve, classify, format JSON, or a typed extract. A thoughtful personality on every node pays tokens and does not buy quality.

Three ways to ask for the chain:

1. **Instruction**: "Work step by step, then give the answer." Cheap, and sometimes enough. The next persona edit drops the sentence, and the hop goes back to a short completion.
2. **Few-shot CoT**: A short worked example of the same shape. Use it when the shape is easy to imitate and hard to specify. A bad example becomes the distribution: witty proofs, skipped units.
3. **Forced structure**: Numbered steps, a scratch section, then a final line the runtime can split. Use it when you will score the answer apart from the scratch. It is still not a tool loop. If the hop needed a tool, the structure only makes the invention easier to parse.

* **Debug artifact**: The chain leaks policy, private tool traces, and sometimes a wrong path the model then includes. Log it. Score it.
* **What the user sees**: The answer, plus citations. A chat bubble is the wrong place for the inner monologue.
* **What a regulator gets**: A stored copy, redacted. That is a retention decision, not a UI default.
* **Visible versus hidden**: Native thinking and CoT can both be a linear deliberation. Native thinking may be hidden and billed as tokens that never appear in the answer. CoT sits in the completion. For an agent, a chain you can parse beats a deliberation you cannot read, unless you actually retain the vendor trace.

## 4. ReAct

**ReAct** interleaves thought, action, and observation so that each observation can change the next thought. A chain of thought that ends in one tool call is still a single plan.

```
THOUGHT    what I believe, and what I need
   |
   v
ACTION     a tool call, or finish
   |
   v
OBSERVE    the runtime result, appended as tokens
   |
   +--> next thought, until a stop
```

* **Closed loop**: A wrong query gets reformulated. An empty list tries another source. A surprise error replans. Use it when the next act depends on live data. It breaks when the runtime never comes back after the tool.
* **One observation, a new thought**: Thought: book the hotel now. Action: search flights. Observation: the only flight arrives after check-in. Next thought: move the hotel. A plan frozen before that search cannot make the edit. CoT followed by a single tool call at the end is still one plan.
* **Reactive on purpose**: ReAct does not require a global plan up front. That is a strength for search, APIs, and "what is on this page?" It is a weakness when the job needs a join, such as "do not book the hotel until the flight meets this constraint." The agent optimizes the latest observation and paints itself into a corner. Store a plan, and let ReAct execute the current step.
* **Three parts, or the name is decorative**: Honest tool descriptions, instructions that name the loop, and a runner that iterates. A persona that says "use tools if needed" often emits a long thought and a final answer with zero calls.
* **The trace that proves the loop**: At least two model calls, a tool span between them, and the observation text present in the later call. If the first model call already answers, the loop never started.
* **The runner is the product**: Call the model, execute the tool, append the observation, call the model again. The paper's name is not a vendor switch. If the runtime cannot return after a tool, you have an assistant.

### Watch for

* Trace tax. Every thought, act, and observe costs tokens and latency. Cap `max_turns`. Cap observation size. An 80k HTML dump will reason about ads.
* Untrusted observations. Tool text is data, not a new system prompt. Prompt injection still applies inside one agent.
* Twelve overlapping tools and no rule for which comes first. That is a description problem, or too many tools in one persona. Fix those before you add a tree.

## 5. An explicit plan

A **plan** is the global view, written outside a single forward pass: subtasks, dependencies, and done-checks. CoT answers one question in a line. ReAct picks the next act from the latest observation. Later hops have to be able to re-read the plan.

1. **The next prompt**: You paste the plan forward. Use it for a one-hop handoff you will throw away. It breaks as soon as a second agent must rely on step 4 still being true: the window fills, and private working text leaks.
2. **A typed plan object**: You pass a `Plan` from hop to hop. The planner speaks in your tool names. Use it when code will read status. It breaks when a task cannot be mapped to a tool plus a check. That task is still a wish.
3. **A blackboard slot**: Named state other hops can patch. Use it when several roles share artifacts. It breaks when a replan leaves the old slot in place and the next hop trusts it.
4. **A scratchpad tool**: A sequential-thinking server, or a similar thought log. Use it when the plan is revised across many tool calls. It breaks if the agent never writes. A silent tool stores nothing.

```
PLAN (stored)
  1. collect sources     owner: researcher     status
  2. outline claims      owner: planner
  3. draft               owner: writer
  4. done-check          code or a critic
        ^
        each hop reads or patches this object
        CoT or ReAct inside a hop executes one line
```

* **Serialize, or the search drifts**: If the plan is only the first paragraph of a ReAct thought, the agent searches off-plan because a snippet looked interesting. Replanning is a tool, a hop, or a typed patch. A change of mind that lives only in the thought channel will not be what the next hop reads.
* **Tool-shaped tasks**: "Research the vendor" is still a wish until it names a tool you actually have, plus a check for done. The planner should speak in those tool names. A step for a tool that is not registered will be narrated, not executed.
* **Order is part of the plan**: Three lookups with no dependency, forced into a chain, waste latency. "Book the hotel" run in parallel with "confirm the flight still meets the constraint" wastes retries. Encode "search before plan" in the flow. Keep about five lines. A strategy memo drifts, and nobody updates step 12.
* **CoT can serve the plan**: A planner hop may think step by step in order to emit the plan object. That is chain-of-thought in service of planning, not a third method. The object is working memory for this job. A plan that must survive a later session needs a memory write and a later read.

## 6. Instructing a chain-of-thought hop

The model reasons. The agent is the model plus a persona, tools, orchestration, and an environment. Instructions only spend tokens inside a loop the runtime actually runs, and only keep a plan you actually store.

The smallest chain-of-thought agent is a persona that says to work step by step, a question that has a middle, and a runner. No tools yet. You are testing whether the instruction changes the completion.

* **Say this**: Produce numbered steps. Put the final answer after a delimiter the runtime can split. Do not skip unit conversions. If a constraint conflicts, name the conflict instead of smoothing it over.
* **Leave the other patterns out**: A full ReAct spec, when there is no tool loop. "Consider many alternatives," when you did not build a search. "Remember last week's plan," when nothing was stored. Those sentences describe products you have not built.
* **Schema owns the shape**: When `output_type` already defines the answer, prose that repeats it will drift. Put the chain in a `reasoning` field. Put the answer in `answer`.
* **Yield to tools**: A hop that has a calculator, and instructions that only narrate, will invent the number. Facts that are not in the prompt come from tools. Otherwise you have trained a narrator.
* **Low temperature**: You want step 4 to follow from steps 1 through 3. You are not brainstorming. A creative chain is a different graph every run, and the suite cannot separate a regression from a mood.
* **Score the answer**: Eloquence is not the product, unless you are building a tutor and the chain is the product. A beautiful wrong chain is a regression. At least look at the split between the scratch section and the answer.
* **Pass the conclusion**: The next agent in a flow receives the typed result. It has its own reasoning budget. The chain stays in the log.

## 7. Implementing ReAct

ReAct is three constructor choices. Miss one, and the trace is a single completion with a persuasive persona.

1. **Tools**: Descriptions that say when to call and what empty means. Toys are enough to learn the rhythm: a date jump, a calculator, `get_status`, `travel_back(year, n) -> year'`, `travel_forward(year, n) -> year'`. A vague description breaks the loop because the model narrates the action instead of calling it.
2. **Instructions that name the beats**: Think. If you need a fact or an act, call a tool. After the result, update the thought. Then another call, or a final answer. A travel persona says: maybe call `travel_back` or `travel_forward`, revise after each result, then answer. "Be helpful" never starts the loop.
3. **A runner with `max_turns` greater than 1**: Call the model, run the tool, append the observation, repeat until a final output or the cap. `max_turns` of 1 is an assistant that was described as ReAct.

```
persona: Think, then maybe call travel_back or travel_forward.
         After each result, revise. Then answer.
tools:   travel_back(year, n) -> year'
         travel_forward(year, n) -> year'
runner:  until a final output, or max_turns
```

* **Observation quality is the production failure**: Stack traces as strings, raw HTML, and an HTTP 429 the model reads as page content. Wrap the tool so the observation is small and structured. How failure is phrased is part of the agent.
* **Your schema is the action space**: A paper's syntax (`Search[...]`) is not your tool schema. The model emits "Search: cats" and never calls the function. Instruct in the runtime's language. A second dialect is how a notebook demo fails once tools are real.
Stop on a check you could write down. Do not stop because the thought sounds finished.

1. **The persona's done-check is true**: The user-facing answer is actually ready. The instructions named what "ready" means.
2. **A coded gate**: Tests pass, a required field is present, or a schema validates. Prefer this when the check can be code.
3. **`max_turns`**: A budget, not a suggestion. The loop stops when the cap hits, whether or not the thought sounded finished.
4. **A guardrail trips**: The run is blocked. That is a stop even if the model wanted another hop.

* **Confidence is not a stop**: "The thought looks sure" is a token pattern. The loop keeps going until one of the four checks above is true.
* **MCP is the same loop**: The action can be a native function or an MCP tool. Extra failures are transport and tool listing. If the tools never appear in the trace, debug the wiring before you debug the reasoning.
* **One horizon at a time**: Thinking through all ten steps, executing step 1, then thinking through all ten again on a stale plan pays for a global essay and a local loop. Shorten the thought to the next act, or store the plan and only ReAct inside the current step.

## 8. Tree-of-thought

Most hops should stay plain. A scoped extract, one lookup, or a fixed rewrite wastes the latency of a named pattern. Add structure when the hop has several decisions, intermediate results, or a single pass that keeps failing a score you trust.

**Tree-of-thought (ToT)** is a search over candidate thoughts. Chain-of-thought is one path. ToT branches, scores, prunes, and sometimes backtracks. If you cannot score a candidate, you cannot prune, and the fan-out is just spend.

```
thought 0
  |-- A    score 0.2    prune
  |-- B    score 0.8    expand
  |      |-- B1
  |      |-- B2         pick or backtrack
  |-- C
```

1. **Lookahead**: Puzzles, move-based games, and mutually exclusive plans ("if the API quota fails, change the product or the vendor?"). Use it when a wrong early branch is expensive and a judge is cheap. It breaks the moment "score" is another frontier call on every node: the bill is the product. It is also a poor fit for a chat that has to answer immediately.
2. **A controller, not a mood**: Generate k candidates, score them, expand the winners, stop at a budget. Those four pieces are the search. Miss the score and you cannot prune. Miss the budget and the fan-out is open-ended.
3. **One sentence that says "consider three options"**: That is chain-of-thought with a list, inside one completion. Several samples plus a judge is a usable approximation when the branches are obvious and a controller is more machinery than the decision. Reasoning models cover some of this with longer deliberation. Explicit search still wins when the branches and the scores are clear enough to store.
4. **Search plans, then act**: Run the tree over candidate plans or moves. Execute the winner with ReAct. Searching tool dumps forks the bill and not the idea.

### Watch for

* Cost is branching factor times depth times judge calls. Write that product down before you code. Store which node you are on and what budget remains. A tree that lives only in prose collapses into a list.
* Pruned paths leak into the user answer. Return the chosen path's conclusion, not the graveyard.

## 9. Reflexion

**Reflexion** is an attempt, a critique, and another attempt with that critique in the context. The next prompt is richer. The weights are the same weights.

```
attempt n  -->  artifact
     |
     v
critic: code, a rubric, or another model
     |
     +-- pass --> done
     +-- fail --> feedback becomes part of attempt n+1
```

* **The weights do not learn**: Older write-ups call this learning. Nothing in the model is updated. You conditioned a prompt. If you do not persist the critique, the next session has amnesia. If you persist a bad critique, you reinforce it. That is memory hygiene.
* **Depth along one path**: Tree-of-thought spends on a wide frontier. Reflexion spends on retries along one path. The same shape shows up whenever a critic sends comments back.
* **Code before a model critic**: Use `tests_pass()` or any check you can write when the failure is detectable that way. A model critic is for judgments you cannot write as code. Code is stable. The model critic drifts.
* **Falsifiable feedback**: "Step 3 used 2015 instead of 2016. Recompute from the tool." Use it on messy generation, code, or a plan that almost works, when the first attempt is cheap and the failure is detectable. A critic that says "be more detailed" makes the solver write novels. Tokens rise. The answer does not. That critic has a persona bug.
* **Idempotent attempts**: Retrying `charge_card` or `send_invoice` because a memo was rude is an incident. The attempt runs in a sandbox, or the side effect is impossible to repeat. A critique must not be able to fire the charge again.
* **A missing fact is a tool call**: If the failure is "needed an observation," call the tool. Critiquing a guess does not create the fact. Reflexion is not a stand-in for ReAct.

## 10. Pick a pattern and pin it

Pick from the shape of the hop and from what you can measure. Each node gets one pattern. An instruction block that says "think step by step, explore branches, reflect, use tools, and keep a global plan" lets the sampler pick a different vibe on each run.

| If the hop is... | Start with | What you pay if you pick wrong |
|---|---|---|
| One fact, and the model is strong | The forward pass | A chain-of-thought essay |
| Multi-step logic, no tools | CoT | A tree, unless the hop is a search puzzle |
| Live data, or an action | ReAct | A chain that was supposed to "just know" |
| Mutually exclusive plans you can score | ToT with a controller | One prompt that says "consider options" |
| A detectable failure, a cheap retry, no side effect | Reflexion | Another call to an irreversible tool |
| A long job across many hops | A stored plan, then ReAct on the current step | One reactive soup with no join |

* **A mix is a graph of contracts**: A planner hop writes a plan with chain-of-thought. Executor hops run ReAct. A critic runs Reflexion. If one persona lists every pattern, Monday's trace and Tuesday's trace will not match. Pin the pattern the way you pin temperature. Two patterns means two nodes, or a coded controller that chooses.
* **SKU lookup**: Start with the forward pass. You pay one completion. A chain-of-thought essay is the wrong bill.
* **Logic puzzle, no API**: Start with chain-of-thought. You pay for the visible chain. A tree is extra unless you can score the branches.
* **Three vendors you can score**: Start with tree-of-thought and a controller. You pay judge calls. That cost is the reason the controller exists.
* **Latency is part of the product**: Tree-of-thought, and Reflexion stacked on a tree, are slow and expensive. A chat reply with an 800 ms budget does not contain that product. Use a smaller pattern, or a smaller model, on that hop.
* **A stronger model thins the pattern**: You may drop a search you no longer need. The tools stay. Longer deliberation is still not a place you can read a plan back from, and it still does not remember a plan you failed to store.

## 11. Sequential thinking is a scratchpad

The sequential-thinking server stores thought records the agent writes and later reads. It does not think. A plan can survive across tool calls without stuffing the whole chain into every completion. The protocol only packages the tool. The agent still has to call it. A server nobody calls is a process you are paying to spawn.

```
agent --MCP--> sequential-thinking server (thought log, revisions)
          \--> world tools (search, a calculator, files)
```

* **Working memory with a schema**: Instructions have to say when to write, when to revise, and when to stop. Otherwise the model ignores the tool, or it journals every token. Description, persona, and traces, the same as any other tool.
* **Prove the handshake before the puzzle**: Start the server. Attach it to a trivial agent. Print `list_tools`. Ask the agent what it can do. The first persona should be bored: a planning assistant with a sequential-thinking tool. If that run cannot see the tool, stop. Do not pile a tree-of-thought instruction on top of a failed handshake.
* **Inspector versus the agent**: If the MCP Inspector can call the thought tool and the agent never does, the bug is the persona or the schema. You do not have a planning paper to reread. You have a tool the model is not choosing.
* **Transport is the lifetime of the pad**: STDIO versus SSE is a transport choice. The usual local setup is `npx` plus STDIO. A new subprocess is a new empty scratchpad. Thoughts that must survive two runner processes need SSE or some other store. An empty plan across processes is not a reasoning failure.
* **Facts stay in world tools**: A persona that calls the pad "your brain," while search is also available, will record URLs that were never fetched. The pad holds the plan and the current step. A claimed source has to appear in the sources the next hop can see, not only inside a thought record.

## 12. Hard problems on the scratchpad

Pick a job that punishes a skipped step: nested constraints, several state updates, a done-check that prose can fake. Time-travel puzzles are one genre, local arithmetic that is easy to narrate wrongly. An SLA calculation, a migration order, or a policy with exceptions has the same shape.

* **World tools make the state real**: Even a toy such as `travel_back` or `apply_discount`. Arithmetic that happens only in the pad is a story about the numbers.
* **Write, act, revise, answer**: Draft the plan in the pad. Execute with world tools. After each observation, revise the plan if it broke. Then answer.
* **The observation has to be written**: Pass it into the next thought. If the pad is the store, the agent must write a short summary of what the tool returned. Agents skip that write unless "done" for the beat includes it. Hoping the model remembers an observation you did not place in the window, or in the pad, drops the fact.
* **A plan-only success is a fail**: When the product is supposed to act, the eval requires tool spans. A beautiful plan, zero world-tool calls, and an answer that matches the plan did not do the job. The scratchpad became chain-of-thought with extra latency.
* **Revise means replace**: Cap the size of a thought. Appending every discarded branch builds a second place for context to get diluted. When the plan changed, the old lines should not remain as if they were still current.

Combining patterns on the pad is a design, because the combination needs state. One persona that contains every pattern yields a random subset each run. Split the contracts.

1. **Planner**: Chain-of-thought writes a plan into the pad, and may also emit a typed plan. It does not call world tools.
2. **Executor**: ReAct reads the current step, calls world tools, and patches status.
3. **Critic**: Reflexion reads the artifact and the pad, then writes a verdict or a patch. It does not generate the artifact.
4. **Optional search**: A controller scores alternative plans in the pad when you can mark alternatives or store branch ids. The executor then runs the winner.

```
[Planner: CoT] --> plan in the pad
[Executor: ReAct] reads the pad, calls world tools, patches status
[Critic: Reflexion] reads the artifact and the pad
[optional ToT] scores alternative plans, then the executor runs one
```

* **Budget the expensive corner**: One agent with a tree, Reflexion, the scratchpad, and two world tools will show up on the bill. Time it, cap it, and pin the model.
* **Save it for an ambiguous, high-value job**: Office Wi-Fi does not need that stack. A migration with exceptions might.
* **Score each model**: A strong reasoning model may need less search. A weak model may fill the pad with noise. Text in the pad is not ground truth either way.
* **A short worked strategy**: A few-shot plan in the persona can steady a chain-of-thought planner. A solved novel in the instructions makes the next problem a remix: wrong numbers, the right vibe.
* **Three bullets can ride along in code**: One tool call, or a plan you can put in a typed `Plan` and pass in memory, does not need an `npx` process. Externalize when the plan is revised across many tool calls, or across agents that should not share a thread.

## Check yourself

1. An agent splits "onboard a vendor" into five sensible pieces, then emails legal before security has signed. Which operation failed, and what would you store so the next hop cannot forget the dependency?
2. You turn on a thinking model and delete the chain-of-thought instructions. The hop still never calls search. What did the extra deliberation not give you, and which trace would prove it?
3. When is chain-of-thought the right spend, and when is it a tax on a hop such as "format this JSON"? What should the next agent in a flow receive instead of the chain?
4. The persona describes ReAct, tools are registered, and `max_turns` is 1. What pattern is actually running, and what would a real ReAct trace have to show?
5. Why is ReAct a weak only-pattern for booking a flight and then a hotel under one constraint? What object would you add, and which beats stay reactive?
6. Someone enables tree-of-thought with the sentence "consider three options." Which pieces of the search are missing, and when is several samples plus a judge still acceptable?
7. Reflexion retries `send_invoice` after a critic says the memo was rude. What invariant broke, and how do you reshape attempts so a critique cannot create a side effect?
8. Using the pattern table, what do you start with for a SKU lookup, a logic puzzle with no APIs, and three mutually exclusive vendors you can score? What do you pay in each case?
9. The MCP Inspector shows the sequential-thinking tool. The agent never writes a thought. Across two runner processes the plan is empty anyway. Which experiment separates a persona miss from a transport or lifetime miss?
10. You want chain-of-thought to draft a plan, ReAct to execute it, and Reflexion to patch it. Why is that three contracts rather than one persona, and when does the scratchpad earn a process compared with a typed plan passed in code?
